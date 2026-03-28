# Eyevinn Open Intercom – Self-Hosted Setup

This repository contains the configuration files to self-host the [Eyevinn Open Intercom](https://github.com/Eyevinn/intercom-manager) solution on an Ubuntu server using Docker.

Eyevinn Open Intercom is a low-latency, web-based, open-source voice-over-IP intercom solution built on WebRTC, designed for broadcast and media production environments.

---

## Architecture

| Component | Runtime | Purpose |
|---|---|---|
| **SMB** | Native on Ubuntu | Symphony Media Bridge – WebRTC media server |
| **MongoDB** | Docker container | Persistent database (named volume) |
| **intercom-manager** | Docker container (patched) | API microservice |
| **intercom-frontend** | Docker container | Web UI |
| **Nginx** | Docker container | HTTPS reverse proxy |

> **Why SMB native?** The official SMB binary v2.1.0-266 was compiled with Intel CET and requires kernel >= 6.x with CET support. On Ubuntu 22.04 with the HWE kernel it runs natively without issues.

> **Why a patched intercom-manager?** intercom-manager v3.5.x sends `PUT` requests to SMB for endpoint configuration, but SMB only accepts `POST`. The `Dockerfile` in this repo patches this automatically at build time.

---

## Repository Structure

```
intercom-server-config/
├── README.md
├── docker-compose.yml          # Main compose file, reads from .env
├── .gitignore                  # Excludes .env and secrets
├── nginx/
│   └── nginx.conf              # HTTPS reverse proxy config
└── intercom-manager/
    └── Dockerfile              # Patches PUT -> POST for SMB compatibility
```

> **Note:** The `.env` file is **not** in this repository. It is generated automatically on the server at startup and contains only the current server IP.

---

## Requirements

- Ubuntu Server 22.04 LTS or newer
- Kernel 6.x (HWE): verify with `uname -r`
- A user with sudo privileges
- Ports **22**, **80**, **443**, **4443** and **10000/udp** open on the firewall
- Internet access to pull Docker images and the SMB package from GitHub

---

## Installation

### Step 1 – Prepare the system

```bash
sudo apt update && sudo apt upgrade -y

# Install HWE kernel if not already present
sudo apt install -y linux-generic-hwe-22.04
sudo reboot

# Verify kernel version after reboot (should show 6.x)
uname -r
```

### Step 2 – Install Docker

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Step 3 – Install SMB (Symphony Media Bridge)

```bash
sudo apt install -y wget libcap2-bin

# libssl1.1 is required by SMB but not available in Ubuntu 22.04 by default
wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2_amd64.deb \
  -O /tmp/libssl1.1.deb
sudo apt install -y /tmp/libssl1.1.deb

wget https://github.com/finos/SymphonyMediaBridge/releases/download/2.1.0-266/finos-rtc-smb_2.1.0-266.deb \
  -O /tmp/smb.deb
sudo apt install -y /tmp/smb.deb
```

Create the SMB config (the startup script will fill in the IP automatically):

```bash
sudo nano /etc/finos-rtc-smb/config.json
```

```json
{
    "port": 4443,
    "ice": {
        "publicIpv4": "PLACEHOLDER",
        "singlePort": 10000,
        "udpPortRangeLow": 10006,
        "udpPortRangeHigh": 26000
    }
}
```

Create the SMB systemd service:

```bash
sudo nano /etc/systemd/system/smb.service
```

```ini
[Unit]
Description=Symphony Media Bridge
After=network.target intercom-startup.service

[Service]
ExecStart=/usr/bin/smb /etc/finos-rtc-smb/config.json
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable smb
```

### Step 4 – Clone this repository

```bash
cd ~
git clone https://github.com/<YOUR-USERNAME>/intercom-server-config.git intercom
cd intercom
```

### Step 5 – Create the startup script

This script runs at every system boot. It detects the current IP, writes the `.env` file, updates the SMB config, pulls the latest configuration from GitHub, and starts the containers.

```bash
sudo nano /usr/local/bin/intercom-startup.sh
```

```bash
#!/bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
set -e

INTERCOM_DIR="/home/<YOUR-USERNAME>/intercom"
SMB_CONFIG="/etc/finos-rtc-smb/config.json"
LOG="/var/log/intercom-startup.log"

echo "=== Intercom Startup $(date) ===" >> "$LOG"

# Detect current IP (most reliable method)
SERVER_IP=$(ip route get 1.1.1.1 | grep -oP 'src \K[\d.]+')
echo "Server IP: $SERVER_IP" >> "$LOG"

# Write .env file (automatically read by Docker Compose)
cat > "$INTERCOM_DIR/.env" << EOF
SERVER_IP=${SERVER_IP}
EOF

# Update SMB config with current IP and ensure correct port
sudo sed -i "s/\"publicIpv4\": \".*\"/\"publicIpv4\": \"${SERVER_IP}\"/" "$SMB_CONFIG"
sudo sed -i "s/\"port\": .*/\"port\": 4443,/" "$SMB_CONFIG"

# Pull latest configuration from GitHub
cd "$INTERCOM_DIR"
git pull origin main >> "$LOG" 2>&1

# Restart containers with latest config
docker compose down >> "$LOG" 2>&1
docker compose up -d --build >> "$LOG" 2>&1

echo "=== Startup complete ===" >> "$LOG"
```

```bash
sudo chmod +x /usr/local/bin/intercom-startup.sh

# Create log file with correct ownership
sudo touch /var/log/intercom-startup.log
sudo chown <YOUR-USERNAME>:<YOUR-USERNAME> /var/log/intercom-startup.log

# Fix repository ownership (if cloned with sudo)
sudo chown -R <YOUR-USERNAME>:<YOUR-USERNAME> ~/intercom
```

Allow the startup script to update the SMB config without a password prompt:

```bash
sudo visudo
```

Add this line at the end:
```
<YOUR-USERNAME> ALL=(ALL) NOPASSWD: /usr/bin/sed
```

### Step 6 – Create the startup systemd service

```bash
sudo nano /etc/systemd/system/intercom-startup.service
```

```ini
[Unit]
Description=Intercom Startup (detect IP, git pull, start Docker)
After=network-online.target docker.service
Wants=network-online.target
Before=smb.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/intercom-startup.sh
RemainAfterExit=yes
User=<YOUR-USERNAME>

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable intercom-startup.service
sudo systemctl enable smb.service
```

### Step 7 – Configure the firewall

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 4443/tcp
sudo ufw allow 10000/udp
sudo ufw enable
sudo ufw status
```

> **Important:** Always open port 22 first to avoid locking yourself out.

### Step 8 – First start

```bash
sudo systemctl start intercom-startup.service
sudo systemctl start smb.service

# Check status
sudo systemctl status intercom-startup.service
sudo systemctl status smb.service

# View startup log
cat /var/log/intercom-startup.log

# Check containers
docker compose -f ~/intercom/docker-compose.yml ps
```

### Step 9 – Verify

```bash
# SMB should respond with an empty array
curl -s http://localhost:4443/conferences/
# Expected: []

# Check which IP was detected
cat ~/intercom/.env
```

Open in browser (accept the self-signed certificate warning on first visit):
```
https://<SERVER-IP>
```

---

## How it works: Dynamic IP

Since this server may run on different networks, the startup script automatically detects the current IP using `ip route get 1.1.1.1`. This selects the IP of the interface that would be used to reach the internet – always the right one regardless of network configuration.

On every boot:
1. Current IP is detected
2. `.env` is written with `SERVER_IP=<detected-ip>`
3. SMB config is updated with the new IP
4. `git pull` fetches the latest configuration
5. Docker Compose rebuilds and starts the containers

---

## How it works: GitOps

All configuration lives in this repository. To update the server configuration:

```bash
# On your development machine
git add .
git commit -m "describe your change"
git push
```

The server picks up changes automatically on next reboot, or immediately by running:

```bash
# On the server
cd ~/intercom
git pull
docker compose up -d --build
```

The `.env` file (containing the server IP) is never committed – it is always generated locally on the server.

---

## Database persistence

MongoDB data is stored in the named volume `mongodb-data` and survives container restarts and updates:

```bash
# Safe restart – data is preserved
docker compose down
docker compose up -d

# WARNING: this deletes all data
docker compose down -v

# Create a backup
docker exec intercom-mongodb-1 mongodump --out /tmp/backup
docker cp intercom-mongodb-1:/tmp/backup ~/mongodb-backup-$(date +%Y%m%d)
```

---

## Known issues and fixes

### MongoDB AVX requirement
MongoDB 5.0+ requires AVX CPU support. If your server does not support AVX, use MongoDB 4.4 in `docker-compose.yml`:

```yaml
mongodb:
  image: mongo:4.4
```

### intercom-manager PUT vs POST
intercom-manager v3.5.x uses `PUT` for SMB endpoint configuration, but SMB only accepts `POST`. The `intercom-manager/Dockerfile` in this repo patches this automatically:

```dockerfile
FROM eyevinntechnology/intercom-manager:latest
USER root
RUN sed -i "s/method: 'PUT'/method: 'POST'/" /app/src/smb.ts
USER node
```

### SMB Intel CET requirement
SMB v2.1.0-266 was compiled with Intel CET support, which requires kernel >= 6.x. On Ubuntu 22.04 install the HWE kernel:

```bash
sudo apt install -y linux-generic-hwe-22.04
sudo reboot
```

---

## Troubleshooting

**Which IP was detected at startup?**
```bash
cat ~/intercom/.env
cat /var/log/intercom-startup.log
```

**Update after IP change without rebooting:**
```bash
sudo systemctl restart intercom-startup.service
sudo systemctl restart smb.service
```

**SMB not running:**
```bash
sudo systemctl status smb
sudo journalctl -u smb -n 50
curl -s http://localhost:4443/conferences/
```

**Containers not starting:**
```bash
cd ~/intercom
docker compose logs
docker compose ps
```

**Microphone not working:**
The browser blocks `getUserMedia` over plain HTTP. Make sure you access the app via `https://` and accept the self-signed certificate warning.

**No audio despite connected:**
```bash
# Check UDP port
ss -ulnp | grep 10000
# Check firewall
sudo ufw status
```

---

## Credits

- [Eyevinn Open Intercom](https://github.com/Eyevinn/intercom-manager) by Eyevinn Technology
- [Symphony Media Bridge](https://github.com/finos/SymphonyMediaBridge) by FINOS
