---
title: "A Practical Docker Home Server on Raspberry Pi 4"
description: "Build a useful Raspberry Pi 4 Docker home server with Nextcloud, AdGuard Home, secure remote access, monitoring, and reliable backups."
author: "Sebastian Insausti"
date: "2026-08-31"
tags: ["Infrastructure", "Linux"]
canonical_url: "https://insaustis.com/blog/raspberry-pi-4-docker-home-server.html"
---

# A Practical Docker Home Server on Raspberry Pi 4

A Raspberry Pi 4 makes a quiet, low-power home server, but its value comes from running a few dependable services—not every interesting container on Docker Hub. A focused stack can provide private file storage, network-wide DNS filtering, secure remote access, and basic monitoring.

## 1. Start with Reliable Hardware

Use a 64-bit operating system, wired Ethernet, proper cooling, and preferably a 4 GB or 8 GB Pi. Keep the boot card simple and place Docker data on a USB 3 SSD; databases and Nextcloud generate enough writes that an SD card is a poor long-term storage device.

```
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
df -h
docker info --format '{{.Architecture}}'
```

Give the Pi a DHCP reservation or static address. DNS, storage, and remote-access services should not move around the network.

## 2. Run Nextcloud with a Database and Redis

For private file storage and synchronization, deploy Nextcloud with a separate MariaDB or PostgreSQL container and Redis. Redis improves locking and caching, while a dedicated database is preferable to SQLite for regular multi-user use. The [official Nextcloud Docker project](https://github.com/nextcloud/docker) provides Compose examples for this layout.

Persist the Nextcloud files, database data, and configuration separately. Do not publish the database or Redis ports to the LAN. Before updating an installed stack, read the release notes and take both a database dump and a copy of the Nextcloud data and configuration.

```
# Replace service names to match your Compose project.
docker compose exec -u33 app ./occ maintenance:mode --on

docker compose exec db \
  mariadb-dump -u root -p --single-transaction nextcloud \
  > nextcloud-$(date +%F).sql

# Copy Nextcloud data and configuration to separate backup storage here.

docker compose exec -u33 app ./occ maintenance:mode --off

docker compose pull
docker compose up -d
docker compose ps
```

## 3. Run AdGuard Home for Network-Wide DNS Filtering

AdGuard Home is one of the most useful additions: it provides DNS-based blocking for every device on the home network with very little CPU or memory usage.

```
sudo install -d -o "$USER" -g "$USER" \
  /srv/adguardhome /srv/adguardhome/work /srv/adguardhome/conf
cd /srv/adguardhome
```

Create `compose.yaml`:

```
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "3000:3000/tcp"
      - "8081:80/tcp"
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
```

```
docker compose up -d
docker compose logs --tail=50 adguardhome
```

Open `http://PI-IP:3000` and complete the wizard, then configure your router to advertise the Pi as its DNS server. Port 53 must be free; `systemd-resolved` or another DNS service may already be using it. Bridge networking may show Docker's gateway instead of each client's original IP; use host networking only after resolving all port conflicts. After testing, pin the image to a specific release instead of tracking `latest`.

## 4. Choose Secure Remote Access

For remote access, use WireGuard or Tailscale rather than exposing Nextcloud and administration panels directly to the Internet. Tailscale is particularly convenient behind carrier-grade NAT and can make the Pi a subnet router for the rest of the LAN.

Although an official Tailscale container exists, installing the VPN directly on the Pi is often simpler for a home gateway: routing, kernel networking, startup order, and troubleshooting remain outside Docker. Containerize applications; let the host handle access to the host.

## 5. Add Lightweight Monitoring When Needed

A small uptime dashboard or a few scheduled health checks are enough for most home labs. Monitor free disk space, temperature, container health, database backups, and whether DNS still answers.

```
docker stats --no-stream
vcgencmd measure_temp
df -h /srv
dig @127.0.0.1 example.com
docker compose ps
```

Send failures to a Telegram bot or another external channel. An alert stored only on the failed Pi is not very useful.

## 6. Back Up What Cannot Be Recreated

Compose files and images are easy to recreate; user data is not. Back up database dumps, Nextcloud data and config, AdGuard configuration, and any secrets. Keep at least one copy on another device and periodically test a restore.

> RAID, mirrored disks, and Docker volumes are not backups. They do not protect against accidental deletion, corruption, theft, or a bad application update.

## What Not to Run

Avoid CPU-heavy video transcoding, large analytical databases, compilation farms, and too many overlapping dashboards. Home Assistant can run in a container, but some appliance-style features are easier with its dedicated operating system. Leave memory and storage headroom for the services that matter.

---

For a Raspberry Pi 4, the practical sweet spot is Nextcloud with its database and Redis, AdGuard Home, secure VPN access, and lightweight monitoring. It is small enough to understand, back up, and recover.
