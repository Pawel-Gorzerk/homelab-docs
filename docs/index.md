# Pawel's Homelab

A fully self-hosted infrastructure stack running on a single **Raspberry Pi 5 (8GB)**, covering DNS, reverse proxying with real HTTPS, automated media management, monitoring, intrusion prevention, remote access, and encrypted off-site backups.

Built incrementally, with every design decision, misstep, and fix documented as it happened — see the [Troubleshooting Log](troubleshooting.md) for the honest version of how this actually got built.

## At a glance

| | |
|---|---|
| **Hardware** | Raspberry Pi 5, 8GB RAM, SSD-based storage |
| **Containers running** | 20+ |
| **Uptime approach** | Docker Compose stacks via Portainer, `restart: unless-stopped` |
| **Network** | Custom bridge network (`proxy_net`) for container-name DNS resolution |
| **Remote access** | Tailscale subnet router - zero exposed ports |
| **Backups** | Restic + rclone to Google Drive, SQLite-consistent, encrypted |

## What this stack does

- **Blocks ads and handles local DNS** for the whole network via Pi-hole
- **Terminates real, trusted HTTPS** for internal services via Nginx Proxy Manager + Let's Encrypt DNS challenge (no ports exposed to the internet)
- **Automatically fetches and organizes media** on request: Jellyseerr → Sonarr/Radarr → Prowlarr → qBittorrent → Bazarr for subtitles → Jellyfin
- **Monitors itself**: Prometheus + Grafana for metrics, Uptime Kuma for availability, cAdvisor + node-exporter for container/host stats
- **Defends itself**: CrowdSec parsing logs and enforcing bans via a firewall bouncer
- **Notifies me proactively**: ntfy push notifications for downtimes, completed downloads, and security events
- **Is reachable from anywhere** via Tailscale, without a single forwarded port on the router
- **Backs itself up nightly**: SQLite-consistent snapshots, encrypted, deduplicated, shipped off-site

## Explore

- [**Architecture**](architecture.md) — how everything connects, with diagrams
- [**Setup Guides**](setup/networking.md) — step-by-step for each subsystem
- [**Troubleshooting Log**](troubleshooting.md) — real incidents encountered while building this, and how they were actually diagnosed and fixed
- [**Tech Stack Reference**](stack-overview.md) — every container, what it does, and why it was chosen
