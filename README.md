# ☁️ Nextcloud Docker Setup with NGINX Proxy Manager

This repository documents the setup of a secure self-hosted [Nextcloud](https://nextcloud.com) server using Docker Compose and HTTPS via **NGINX Proxy Manager**. This is ideal for Raspberry Pi, home servers, or small business environments.

## ✅ Features

- Nextcloud with persistent storage
- NGINX Proxy Manager for HTTPS with Let's Encrypt
- MariaDB as backend database
- Redis cache for improved performance
- Optional monitoring tools (Grafana/Promtail)

## 📖 Documentation

- [Setup Instructions](setup-instructions)
- [Troubleshooting Guide](troubleshooting)

## 🖥 Screenshots

### 📁 Nextcloud Web Interface
![Nextcloud UI](nextcloud-ui.png)
![Nextcloud dashboard](nextcloud-dashboard.png)
### 🔐 NGINX Proxy Manager Setup
![NPM Proxy Host](nignx-pm.png)

### 🐳 Portainer Container Dashboard
![Portainer View](portainer-dashboard.png)

## 🔄 Raspberry Pi → Ubuntu migration (November 2026)

I migrated my running Nextcloud instance to an Ubuntu Linux host after diagnosing Raspberry Pi storage exhaustion and a failing Docker CLI. The migration included an SSH/rsync file transfer, MariaDB backup and restore, Docker Compose volume corrections, and validation of the restored environment. The existing Nginx Proxy Manager remains on the Raspberry Pi and forwards traffic to Ubuntu.

- **Result:** 32 GB of user data migrated; final SQL restore validated with 197 tables; 383 GB free on the Ubuntu filesystem at the last check.
- **Documentation:** [Read the technical migration, recovery, and validation walkthrough](docs/raspberry-pi-to-ubuntu-migration.md).
- **Still planned:** encrypted automated off-device backups and an isolated full restore drill.
