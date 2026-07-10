# Infrastructure Engineering Portfolio

This site documents a self-hosted infrastructure environment built and operated on a single Raspberry Pi 5, used as a continuous learning platform for the kind of work involved in infrastructure, cloud, and support engineering roles.

It is not a media server project. Media automation is one workload running on this infrastructure - a useful one, because it generates real operational problems to solve, but it is not the point. The point is everything underneath it: DNS design, reverse proxy and certificate management, container networking, monitoring, security hardening, backup and disaster recovery, and the troubleshooting process behind each of them.

!!! note "How to read this site"
    Every page here reflects something actually built and operated, not a tutorial being followed. Where something broke, it's documented as it happened in [Lessons Learned](lessons-learned.md) - the debugging process is usually more informative than the final working config.

## What this demonstrates

- **I design infrastructure** - network topology, service segmentation, and certificate strategy were deliberate decisions with tradeoffs, not defaults accepted as-is.
- **I operate infrastructure** - this environment runs continuously and gets maintained, not just set up once and left alone.
- **I troubleshoot failures methodically** - see [Lessons Learned](lessons-learned.md) for real incidents worked through from symptom to root cause.
- **I monitor and act on what I monitor** - metrics and alerting exist because they get looked at and acted on, not because a dashboard looks good.
- **I automate repetitive work** - both the workload automation (media pipeline) and the operational automation (backups, notifications).
- **I validate what I build** - backups are tested by restoring them, not just assumed to work.
- **I document as I go** - this entire site is that documentation.

## At a glance

| | |
|---|---|
| **Host** | Raspberry Pi 5 (8GB RAM), Raspberry Pi OS (Debian 12 "bookworm"), SSD-backed storage |
| **Orchestration** | Docker + Docker Compose, managed via Portainer |
| **Services running** | 20+ containers across networking, monitoring, security, automation, and workload layers |
| **Network design** | Custom Docker bridge network for container-name DNS resolution; internal-only hostnames via Pi-hole |
| **Remote access** | Tailscale subnet router - zero ports forwarded on the router |
| **Backup strategy** | Restic + rclone, SQLite-consistent snapshots, encrypted, deduplicated, off-site |

## Where to start

- [**Architecture**](architecture.md) - system design and how the pieces connect
- [**Infrastructure**](infrastructure.md) - the core services the rest of the stack depends on
- [**Backup & Disaster Recovery**](backup.md) - the section I'd point a hiring manager to first
- [**Lessons Learned**](lessons-learned.md) - real incidents, root cause analysis, and what changed as a result
- [**About**](about.md) - background and why this project exists
