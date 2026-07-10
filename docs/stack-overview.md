# Tech Stack Reference

Grouped by function. Infrastructure and operational tooling are covered in depth on their own pages ([Infrastructure](infrastructure.md), [Networking](networking.md), [Monitoring](monitoring.md), [Security](security.md), [Backup & Disaster Recovery](backup.md)) - this page is a flat reference table for quick lookup, not the primary explanation of any of these.

## Infrastructure & networking

| Container | Purpose | Why this one |
|---|---|---|
| **Pi-hole** | DNS + ad-blocking, local hostnames | Network-wide filtering with zero per-device configuration |
| **Nginx Proxy Manager** | Reverse proxy, TLS termination | GUI-managed, appropriate complexity for this scale versus Traefik/Caddy |
| **Portainer** | Docker/stack management | Visual layer over Compose for day-to-day operations |
| **Tailscale** | Remote access | Host-level subnet router - zero ports exposed on the router |

## Monitoring & observability

| Container | Purpose | Why this one |
|---|---|---|
| **Prometheus** | Metrics collection & storage | Industry-standard time-series backend |
| **Grafana** | Dashboards | Standard visualization layer over Prometheus |
| **node-exporter / cAdvisor** | Host / per-container metrics | Prometheus-native exporters |
| **Uptime Kuma** | Independent availability monitoring | Deliberately separate from the Prometheus stack, so one outage doesn't blind both |
| **Homepage** | Unified status dashboard | No database dependency, native live-data widgets across most of the stack |

## Security

| Container | Purpose | Why this one |
|---|---|---|
| **CrowdSec + firewall bouncer** | Intrusion detection & active prevention | Community threat intelligence plus local log-based detection, actually enforced via iptables |
| **Vaultwarden** | Self-hosted credential management | Bitwarden-protocol-compatible, small footprint |

## Operational automation

| Container | Purpose | Why this one |
|---|---|---|
| **n8n** | General-purpose workflow automation | Connective tissue between services without a native integration path |
| **ntfy** | Push notifications | Self-hosted, closes the loop between monitoring and actually noticing |
| **Watchtower** | Scheduled container updates | Runs on a schedule rather than continuous polling - deliberate, see [Lessons Learned](lessons-learned.md#the-watchtower-crash-loop) for what happens when its lifecycle isn't managed carefully |
| **restic + rclone** | Encrypted, deduplicated, off-site backups | Full design rationale in [Backup & Disaster Recovery](backup.md) |

## Workload: media automation

The orchestration pattern here (request → search → fetch → import) is discussed in depth in [Automation](automation.md); this is a flat reference only.

| Container | Purpose |
|---|---|
| Jellyfin | Media server / playback |
| Jellyseerr | Request front-end |
| Sonarr / Radarr / Prowlarr | TV / movie / indexer orchestration |
| Bazarr | Automated subtitle fetching |
| qBittorrent | Download client |
| Piper | Local text-to-speech (Wyoming protocol), used for voice-assistant integration experiments |

## Resource-conscious choices for an 8GB Pi

- **Homepage over Homarr** - no database requirement, near-zero footprint
- **Headless boot** (no desktop GUI) - freed a genuinely significant chunk of RAM that was otherwise going to a desktop session running alongside the actual workload; see [Lessons Learned](lessons-learned.md) for how this surfaced during a memory-pressure investigation
- **Watchtower on a schedule, not continuous polling** - avoids constant background overhead
- Swap increased from the Pi OS default (200MB - too small for this workload) to 2GB as a safety margin
