# Monitoring & Observability

## Why monitoring exists here

Not because a Grafana dashboard looks good. Monitoring in this environment serves four concrete purposes:

- **Availability** - knowing a service is down before a user (or a friend using Jellyseerr) reports it
- **Resource usage** - an 8GB Raspberry Pi running 20+ containers has real, visible limits; memory pressure and swap usage are things that have actually caused incidents here (see [Lessons Learned](lessons-learned.md))
- **Early detection** - catching a problem (a crash-looping container, a filling disk) while it's still a log entry, not yet an outage
- **Root cause analysis** - when something does break, historical metrics are what turns "it's broken" into "it broke at 00:47, right when X happened"

Monitoring that isn't looked at, or isn't wired to actually notify someone, doesn't serve any of these purposes - which is why the notification layer (below) is treated as part of the monitoring design, not an optional extra.

## Stack and what each piece is actually for

| Component | Role |
|---|---|
| **Prometheus** | Time-series metrics collection and storage - the source of truth for "what happened, when" |
| **Grafana** | Visualization layer over Prometheus - dashboards for trends over time, not just current state |
| **node-exporter** | Host-level metrics (CPU, memory, disk, swap). Runs in `network_mode: host` deliberately - accurate host-level stats require visibility into the host's own network namespace, which a normal bridge-networked container doesn't have |
| **cAdvisor** | Per-container resource metrics - which specific container is consuming what, not just aggregate host usage |
| **Uptime Kuma** | External-style up/down monitoring with its own independent notification path - deliberately separate from the Prometheus/Grafana stack, so a Prometheus outage doesn't also blind availability monitoring |

## Homepage - unified status view

A single-pane dashboard using live-data widgets rather than static links wherever a service exposes an API for it (Jellyfin, Pi-hole, Portainer, Grafana, Prometheus, Uptime Kuma, ntfy, NPM, CrowdSec, and the automation stack all have native widget support; a few services without an API-exposed status are link-only). Configured through a single `services.yaml`, grouped by function.

## Closing the loop: notifications

Dashboards require someone to be looking at them. The actual operational value comes from push notifications closing that gap:

- **Uptime Kuma → ntfy** - native notification integration, alerts on monitor state changes
- **Sonarr/Radarr → ntfy** - alerts on grab/import events in the automation pipeline
- **CrowdSec → ntfy** - alerts on ban decisions, via CrowdSec's generic HTTP notification plugin wired through a notification profile

This is the difference between "monitoring exists" and "monitoring is operationally useful" - see [Operations](operations.md) for more on this distinction.

## Bringing legacy containers under version control

Not every container here started life in a tracked compose file - several were deployed ad hoc early in the project, before configuration-as-code discipline was fully in place. `docker-autocompose` reconstructs a compose definition from a running container's actual live configuration:

```bash
python3 -m docker_autocompose <container-names> > reconstructed.yml
```

The output needs manual cleanup (stale `hostname`/`mac_address` fields tied to the specific running instance, verbose auto-generated labels) before it's genuinely reusable - but it's a fast way to retroactively bring infrastructure into version control instead of leaving it undocumented.
