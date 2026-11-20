# Uptime Kuma Monitoring on AWS EC2 (DevOps Capstone)

A self-hosted monitoring stack built with Uptime Kuma, Docker and AWS EC2.
It monitors three containerised services plus one external endpoint, sends
Discord alerts on DOWN/UP events, and publishes a public status page.

## 1. Project Overview

The goal was to deploy a working monitoring system, prove it detects
failures and recoveries, and document backup and restore procedures.

## 2. Architecture

```
Internet --> AWS EC2 (Ubuntu) --> Docker
                |-- uptime-kuma  (port 3001)  monitoring + status page
                |-- nexusdeploy  (port 3000)  Nexus Dashboard
                |-- flask app    (port 5000)  Flask application

Uptime Kuma --> HTTP check every 60 s --> Discord webhook (#alerts)
```

Uptime Kuma reaches the other containers through the Docker bridge
gateway (`172.17.0.1`).

## 3. Tools and Technologies

AWS EC2 (Ubuntu), Docker, Uptime Kuma 2.5.11, Flask, Discord webhooks,
Linux shell (curl, tar, sha256sum).

## 4. Deployment Steps

1. Launched an Ubuntu EC2 instance and installed Docker.
2. Ran Uptime Kuma with persistent storage: host `/opt/uptime-kuma/data`
   mounted to `/app/data` in the container.
3. Ran the Nexus Dashboard and the Flask app as separate containers.
4. Created four HTTP monitors in Uptime Kuma.
5. Created a Discord webhook and linked it as a notification.
6. Created a public status page at
   `http://[YOUR-EC2-IP]:3001/status/devops-capstone`.

## 5. Monitoring Configuration

| Monitor | Type | Target | Interval |
|---|---|---|---|
| Flask-Application | HTTP(s) | http://172.17.0.1:5000 | 60 s |
| Nexus-Dashboard | HTTP(s) | http://172.17.0.1:3000 | 60 s |
| Uptime-kuma-server | HTTP(s) | http://172.17.0.1:3001 | 60 s |
| CloudPulse - Production Website | HTTP(s) | https://example.com (public test endpoint, not my own site) | 60 s |

Retries are set to 0, so a failed check marks the monitor DOWN immediately.

## 6. Downtime and Recovery Testing

The Flask container was stopped on purpose with `docker stop` and later
started again with `docker start`.

- DOWN detected: 2026-11-19 20:50:44 (`connect ECONNREFUSED 172.17.0.1:5000`)
- UP recovered: 2026-11-20 07:02:34 (`200 - OK`)
- The same test was repeated on Nexus-Dashboard.

<img width="1366" height="731" alt="goj" src="https://github.com/user-attachments/assets/12a7a0a0-d927-4ee7-935e-e4271430614e" />

<img width="1366" height="689" alt="goj2" src="https://github.com/user-attachments/assets/7aaa0ae4-8a7a-4bd9-b28c-7a4f9c4e98db" />


## 7. Notification Testing

Discord alerts were tested with the Test button and with real DOWN/UP
events caused by stopping and starting containers.


## 8. Public Status Page

The status page is public to anyone with the link and needs no login.
The dashboard and settings stay behind the login. It has two groups:
Application Services (Flask, Nexus, Uptime Kuma) and External Endpoint
(CloudPulse).

<img width="1366" height="768" alt="dev proffesional page" src="https://github.com/user-attachments/assets/bb0d0299-0c77-4be1-984f-e61d43cedb54" />


## 9. Backup and Recovery

Data is a SQLite database (`kuma.db`) in `/opt/uptime-kuma/data`.

Backup (Uptime Kuma is stopped for a few seconds so the database is
consistent):

```bash
sudo mkdir -p /opt/uptime-kuma-backups
sudo docker stop uptime-kuma
sudo tar -czf /opt/uptime-kuma-backups/uptime-kuma-data-$(date +%Y%m%d-%H%M%S).tar.gz -C /opt/uptime-kuma data
sudo docker start uptime-kuma
```

Verification:

```bash
sudo tar -tzvf <backup>.tar.gz
sudo sha256sum <backup>.tar.gz | sudo tee <backup>.tar.gz.sha256
sudo sha256sum -c <backup>.tar.gz.sha256
```

Results: the archive listed correctly, the SHA-256 check returned OK, and a
test restore into `/tmp/restore-test` produced a config file identical to
the live one. Production data was not touched during the test.
Limitation: the restored `kuma.db` was not opened inside a running
Uptime Kuma instance.

Production restore (only if data is lost; never delete the old data):

```bash
sudo docker stop uptime-kuma
sudo mv /opt/uptime-kuma/data /opt/uptime-kuma/data.before-restore-$(date +%Y%m%d-%H%M%S)
sudo tar -xzf <backup>.tar.gz -C /opt/uptime-kuma
sudo docker start uptime-kuma
```

## 10. SSL Monitoring

No own domain or SSL certificate is configured, so my own services run on
plain HTTP and certificate-expiry monitoring was not set up for them.
The CloudPulse monitor targets `https://example.com` only as a public
HTTPS test endpoint.

## 11. Limitations and Security Notes

- The status page uses HTTP, so browsers show "Not secure".
- The EC2 public IP can change after a stop/start. An Elastic IP fixes this.
- Port 3001 is open to the internet, which also exposes the login page.
  Use a strong password and enable 2FA.
- Backups contain the database, login hash and Discord webhook URL, so they
  are never stored in this repository.

## 12. Future Improvements

- HTTPS with a domain and a reverse proxy (Caddy or Nginx)
- Elastic IP
- Scheduled backups with cron and off-server copies (S3)
- Restrict Security Group access

Note: this repository is a fork of louislam/uptime-kuma. The CI checks (CodeQL, auto-tests, linters) are inherited from the upstream project. The autofix.ci check is an upstream bot that does not run successfully on forks and is unrelated to this capstone.
