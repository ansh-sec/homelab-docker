# Homelab

Personal homeserver running on Ubuntu Server, managed via Docker Compose, accessed remotely over Tailscale.

> ⚠️ **Before pushing this repo to GitHub:** scrub any real Wi-Fi passwords, MAC addresses, static IPs, or `.env` secrets from every file in this repo. Once something is in git history it's there permanently, even if you delete it in a later commit (`git log -p` or a clone from before the delete will still show it). Use `git-secrets` or just `grep -ri password .` before every push as a habit.

## Stack

| Service | Purpose |
|---|---|
| [Portainer](https://www.portainer.io/) | Web UI to manage Docker containers |
| [Homepage](https://gethomepage.dev/) | Dashboard / landing page for all services |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Uptime / service health monitoring |
| [Nextcloud](https://nextcloud.com/) | Self-hosted cloud storage |
| [Tailscale](https://tailscale.com/) | Zero-config VPN mesh for remote access |
| Jellyfin | Media server — **not yet set up**, planned |
| Watchtower | Auto-updates containers — **skipped for now**, doing manual updates until We actually understand `docker compose` update flow |

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Base OS Setup](#1-base-os-setup)
- [2. Networking — Static IP](#2-networking--static-ip)
- [3. Hostname](#3-hostname)
- [4. SSH](#4-ssh)
- [5. Firewall (UFW)](#5-firewall-ufw)
- [6. Fail2ban](#6-fail2ban)
- [7. Docker](#7-docker)
- [8. Remote Access — Tailscale](#8-remote-access--tailscale)
- [9. Portainer](#9-portainer)
- [10. Homepage](#10-homepage)
- [11. Uptime Kuma](#11-uptime-kuma)
- [12. Nextcloud](#12-nextcloud)
- [Roadmap](#roadmap)

## Prerequisites

- A machine to act as the server (bare metal or VM) with Ubuntu Server installed.
- Another machine on the same LAN to SSH in from.
- Basic comfort with a terminal. This doc assumes you know how to edit a file with `nano` and read a `docker compose` file, but explains *why* each step exists.

---

## 1. Base OS Setup

Install the Ubuntu Server ISO on the target machine. Standard install, no GUI needed — this is a headless server.

Once installed, log in locally or find its IP to SSH in:

```bash
ip addr        # find your IP
whoami         # confirm your username
```

From another machine on the same network:

```bash
ssh username@<server-ip>
```

## 2. Networking — Static IP

A server needs a fixed address — if your router hands it a new DHCP lease after a reboot, every bookmark, Tailscale config, and container `HOMEPAGE_ALLOWED_HOSTS` entry pointing at the old IP breaks.

Ubuntu Server uses **netplan** for network config.

1. Find your interface name:

```bash
ip addr
```

Wireless interfaces are usually named `wlan0` / `wlo1`, wired interfaces `eth0` / `eno1`.

2. Edit the netplan config (the exact filename varies by install — check `/etc/netplan/`):
, not just the command
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

3. Example config — **replace every placeholder below with your own values**:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      match:
        macaddress: <YOUR_INTERFACE_MAC_ADDRESS>
      set-name: eno1
  wifis:
    wlo1:
      access-points:
        "<YOUR_WIFI_SSID>":
          password: "<YOUR_WIFI_PASSWORD>"
      dhcp4: no
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

Your actual addresses, gateway, and subnet depend on your router — check with `ip addr` and `ip route` first, don't copy these values blindly.

Apply it:

```bash
sudo netplan apply
```

## 3. Hostname

```bash
sudo hostnamectl set-hostname homeserver
```

## 4. SSH

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

## 5. Firewall (UFW)

[UFW](https://help.ubuntu.com/community/UFW) (Uncomplicated Firewall) is a friendlier frontend over `iptables`. Default policy should be deny-incoming, allow only what you explicitly open.

```bash
sudo apt install ufw -y
sudo ufw allow 22/tcp     # SSH — do this BEFORE enabling, or you'll lock yourself out
sudo ufw enable
sudo ufw status           # should show "active"
```

Every time you expose a new service on a new port later in this doc, you'll need a matching `sudo ufw allow <port>/tcp`.

## 6. Fail2ban

[Fail2ban](https://github.com/fail2ban/fail2ban) watches auth logs and temporarily bans IPs after repeated failed login attempts. mitigates SSH brute-forcing.

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

## 7. Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Add yourself to the `docker` group so you don't need `sudo` for every docker command (log out and back in for this to take effect):

```bash
sudo usermod -aG docker $USER
```

Confirm Compose is available (ships with Docker as a plugin now, no separate install needed):

```bash
docker compose version
```

Set up the directory structure — one subdirectory per service, each holding its own `compose.yml`:

```bash
sudo mkdir -p /opt/docker/{portainer,watchtower,nextcloud,jellyfin,homepage,uptime-kuma}
sudo chown -R $USER:$USER /opt/docker
```

`/opt` is root-owned by default; the `chown` gives your user write access without needing `sudo` for every edit.

## 8. Remote Access — Tailscale

[Tailscale](https://tailscale.com/) creates a private mesh VPN between your devices using WireGuard, so you can reach your server from anywhere without port-forwarding or exposing it to the raw internet.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Authenticate via Google, GitHub, or another supported identity provider. Repeat `tailscale up` on your main PC (or any device you want connected) using the same account. The Tailscale admin dashboard lists every connected device's Tailscale IP — use that IP, not your LAN IP, to reach the server from outside your home network.

**Quick test** before setting up real services:

```bash
sudo python3 -m http.server 9999 --bind 0.0.0.0
sudo ufw allow 9999/tcp
```

Then hit `http://<server-tailscale-ip>:9999` from another device on your tailnet. If that loads, networking + firewall + Tailscale are all working. Kill the test server (`Ctrl+C`) and remove the UFW rule once confirmed:

```bash
sudo ufw delete allow 9999/tcp
```

## 9. Portainer

[Portainer](https://www.portainer.io/) gives you a web UI to manage all your other containers instead of doing everything from the CLI.

```bash
cd /opt/docker/portainer
nano compose.yml
```

```yaml
services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9443:9443"
      - "8000:8000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

```bash
docker compose up -d
sudo ufw allow 9443/tcp
```

Access at `https://<server-ip>:9443`.

Your browser will show a certificate warning — that's expected, it's because Portainer generates a self-signed cert and your browser doesn't trust it by default. Click through (Advanced → Proceed). This is safe on your own LAN/Tailscale network; it would **not** be safe to click through blindly on a random public site.

On first load, set your admin username/password. If it asks for a setup token, retrieve it with:

```bash
docker logs portainer
```

Look for a line like `Setup_token = <token>`. Skip the "Edge compute" prompt — you don't need it for this setup.

Confirm it's reachable over Tailscale too: `https://<tailscale-ip>:9443`.

## 10. Homepage

[Homepage](https://gethomepage.dev/) is a dashboard that links out to all your other services.

```bash
sudo mkdir -p /opt/docker/homepage
sudo chown -R $USER:$USER /opt/docker/homepage
cd /opt/docker/homepage
nano compose.yml
```

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      HOMEPAGE_ALLOWED_HOSTS: <SERVER_LAN_IP>:3000,<SERVER_TAILSCALE_IP>:3000
    volumes:
      - /opt/docker/homepage/config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

```bash
docker compose up -d
sudo ufw allow 3000/tcp
```

Access at `http://<server-ip>:3000`. Widget/service config to come in a later commit.

## 11. Uptime Kuma

[Uptime Kuma](https://github.com/louislam/uptime-kuma) monitors whether your other services are online and responding, and alerts you when they're not.

```bash
sudo mkdir -p /opt/docker/uptime-kuma
sudo chown -R $USER:$USER /opt/docker/uptime-kuma
cd /opt/docker/uptime-kuma
nano compose.yml
```

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
```

```bash
docker compose up -d
sudo ufw allow 3001/tcp
```

Access at `http://<server-ip>:3001`. Choose SQLite as the database on first run and create your account.

Add each service you've set up so far as a monitor — URL as `<server-ip>:<port>`, name it after the container. If checking an HTTPS service with a self-signed cert (like Portainer), tick **"Ignore TLS/SSL errors"** or the check will always show as down. Double-check `http` vs `https` per service — Portainer is https, everything else here is currently http.

## 12. Nextcloud

[Nextcloud](https://nextcloud.com/) is self-hosted cloud storage — your own Google Drive equivalent.

```bash
sudo mkdir -p /opt/docker/nextcloud
sudo chown -R $USER:$USER /opt/docker/nextcloud
cd /opt/docker/nextcloud
```

Secrets go in a `.env` file, not directly in `compose.yml` — keeps them out of version control if you `.gitignore` the file (see the note at the bottom of this section).

```bash
nano .env
```

```env
MYSQL_ROOT_PASSWORD=CHANGE_THIS_TO_A_LONG_RANDOM_PASSWORD
MYSQL_PASSWORD=CHANGE_THIS_TO_ANOTHER_LONG_RANDOM_PASSWORD
```

Lock the file down so only your user can read it:

```bash
chmod 600 .env
```

```bash
nano compose.yml
```

```yaml
services:
  db:
    image: mariadb:lts
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED
    volumes:
      - db:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MARIADB_AUTO_UPGRADE: "1"

  redis:
    image: redis:alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  nextcloud:
    image: nextcloud:apache
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - nextcloud:/var/www/html
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      REDIS_HOST: redis
    depends_on:
      - db
      - redis

volumes:
  db:
  nextcloud:
```

```bash
docker compose up -d
sudo ufw allow 8080/tcp
```

Access at `http://<server-ip>:8080`. All your data lives in `/opt/docker/nextcloud` on the server (in the named Docker volumes `db` and `nextcloud`).

This is currently LAN/Tailscale-only over plain HTTP — fine for now since it's not exposed to the internet. If it's ever exposed publicly, it needs to sit behind **Caddy** (or another reverse proxy) with a real TLS certificate first — Nextcloud's own documentation requires HTTPS for internet-facing instances. Nextcloud auth over plain HTTP on an open network is a credential-sniffing risk.

> **`.env` files and git:** if this directory is a git repo, add `.env` to `.gitignore` right now, before your first commit. `git rm --cached .env` if you already committed it, then rotate both passwords — they're compromised the moment they hit a public repo.

---

## Roadmap

- [ ] Jellyfin media server
- [ ] Watchtower (auto-updates) — deliberately deferred until I understand the manual `docker compose pull && docker compose up -d` update flow
- [ ] Caddy reverse proxy + real TLS certs for any service that might go internet-facing
- [ ] Backups (currently none — `/opt/docker` is a single point of failure)

## Notes

- Everything here is reachable over LAN or Tailscale only. Nothing is exposed directly to the internet, which is why plain HTTP is currently tolerable for most services — it stops being tolerable the second anything is port-forwarded externally.
- Every new container needs a matching `sudo ufw allow <port>/tcp` or it won't be reachable, even on LAN.
