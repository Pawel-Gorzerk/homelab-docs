# Tech Stack Reference

| Container | Purpose | Why this one |
|---|---|---|
| **Pi-hole** | DNS + ad-blocking, local hostnames | Network-wide, no per-device config needed |
| **Nginx Proxy Manager** | Reverse proxy, HTTPS termination | GUI-based, simpler than Traefik/Caddy for this scale |
| **Portainer** | Docker/stack management | Visual management on top of Compose |
| **Jellyfin** | Media server | Fully open-source, no licensing telemetry |
| **Jellyseerr** | Media request front-end | Turns "can you download X" into self-service |
| **Sonarr / Radarr / Prowlarr** | TV/movie/indexer automation | Industry-standard *arr stack |
| **Bazarr** | Automated subtitles | Integrates directly with Sonarr/Radarr's library |
| **qBittorrent** | Download client | Lightweight, well-supported WebUI |
| **Grafana + Prometheus** | Metrics visualization + storage | Standard, well-documented combo |
| **node-exporter / cAdvisor** | Host / per-container metrics | Prometheus-native exporters |
| **Uptime Kuma** | Uptime monitoring | Lightweight, built-in notification providers |
| **Homepage** | Unified dashboard | No database, native widgets for most of this stack |
| **Vaultwarden** | Password manager | Bitwarden-compatible, tiny footprint |
| **ntfy** | Push notifications | Self-hosted, no third-party dependency |
| **n8n** | Workflow automation | General-purpose glue between services |
| **CrowdSec + firewall bouncer** | Intrusion detection & prevention | Community threat intelligence, actively enforced |
| **Watchtower** | Automatic container updates | Scheduled, not continuous - avoids surprise mid-day updates |
| **Piper** | Text-to-speech (Wyoming protocol) | Local TTS for voice-assistant integrations |
| **Tailscale** | Remote access | Zero exposed ports, subnet router mode |
| **restic + rclone** | Encrypted, deduplicated backups | SQLite-consistent, cloud-agnostic backend |

## Resource-conscious choices for an 8GB Pi

- **Homepage over Homarr**: no database requirement, near-zero footprint
- **Headless boot** (no desktop GUI): freed a genuinely significant chunk of RAM that was otherwise going to Chromium/desktop processes running alongside the actual workload
- **Watchtower on a schedule, not continuous polling**: avoids constant background overhead
- Swap increased from the Pi OS default (200MB - too small for this workload) to 2GB as a safety margin
