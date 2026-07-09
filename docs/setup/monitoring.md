# Monitoring & Dashboard

## Stack

- **Prometheus** - metrics collection and storage
- **Grafana** - dashboards/visualization
- **node-exporter** - host-level metrics (runs in `network_mode: host`, needed for accurate host stats)
- **cAdvisor** - per-container resource metrics
- **Uptime Kuma** - external-style up/down monitoring with its own notification system

## Homepage dashboard

A single-pane view of every service, using native widgets rather than plain links wherever available:

| Service | Widget support |
|---|---|
| Jellyfin, Pi-hole, Portainer, Grafana, Prometheus, Uptime Kuma, ntfy, NPM, CrowdSec, Watchtower, Sonarr, Radarr, Prowlarr, qBittorrent | ✅ Native live-data widget |
| n8n, Vaultwarden, Bazarr, Piper | Link only (no native widget) |

Configured via a single `services.yaml`, grouped by category, with API keys pulled from each service's own settings page.

## Notifications via ntfy

Rather than checking dashboards manually, key events push directly to a phone:

- **Uptime Kuma → ntfy**: native notification type, alerts on monitor down/up
- **Sonarr/Radarr → ntfy**: native connection type (or n8n as a translator on older versions lacking it) - alerts on grab/import
- **CrowdSec → ntfy**: via CrowdSec's generic HTTP notification plugin, wired through a profile - alerts on ban decisions

Full wiring details in the [Troubleshooting Log](../troubleshooting.md) and the project's `ntfy-integrations.md`.

## Reconstructing compose files from running containers

Not every container in this stack started life in a compose file - some were deployed ad hoc early on. `docker-autocompose` reconstructs a compose definition from any running container's actual configuration:

```bash
python3 -m docker_autocompose <container-names> > reconstructed.yml
```

Useful for bringing legacy containers into version control retroactively, though the output benefits from manual cleanup (stale `hostname`/`mac_address` fields, verbose auto-generated labels).
