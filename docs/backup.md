# Backup & Disaster Recovery

This started as a single `restic backup` command pointed at a folder. What's running now is the result of that first version failing in several genuinely instructive ways - each covered below not as a footnote, but because the failure and the fix are the actual substance of the design.

## What gets backed up, and what deliberately doesn't

Two categories, treated differently on purpose:

- **Configuration directories and Docker named volumes** - small, critical, irreplaceable. Backed up on every run.
- **Media library files** - large, but fully reproducible via the automation pipeline itself. Not backed up directly; Sonarr/Radarr's own databases (which *are* backed up) retain the record of what existed, so the library can be rebuilt rather than restored byte-for-byte.

This distinction is a disaster recovery decision, not a storage-saving shortcut: it's a deliberate choice about what "recovery" actually means for each category of data.

## Problem 1: SQLite consistency, without accepting downtime

Nearly every service in this environment embeds SQLite as its database - Sonarr, Radarr, Prowlarr, Jellyseerr, Jellyfin, Nginx Proxy Manager, CrowdSec, Grafana, Pi-hole, Vaultwarden, n8n, and Uptime Kuma. That's a far longer list than it first appears, and the initial backup design didn't account for most of it.

The first working approach was **stopping containers before backup** to guarantee a consistent file copy. It worked, but it meant real downtime across a growing list of services every night, and it didn't scale - every new container added to the stack meant another manual addition to a stop-list.

The better approach: SQLite's own **Online Backup API**, which produces a fully consistent snapshot of a live, actively-written database with zero downtime:

```bash
sqlite3 "$db" ".backup '$STAGING/${rel}.backup'"
```

Rather than hardcoding which services use SQLite (a list that was already wrong once), the backup script **scans generically** for every `.db`/`.sqlite`/`.sqlite3` file across the config and volume paths and snapshots each one the same way. This scales automatically as services are added, instead of silently missing new ones.

## Problem 2: backup size ballooned to 28GB on the first real run

Before any tuning, the first full backup attempt came in at 28GB - which immediately blew past the free tier of the cloud backend being used. Root causes, found by walking the directory tree with `du` rather than guessing:

| Cause | Fix |
|---|---|
| `node_modules` inside n8n's data directory (community node dependencies) | Excluded - fully reproducible by reinstalling the node, not unique data |
| Prometheus's raw time-series database | Excluded - disposable metrics history, not configuration |
| Jellyfin's artwork/metadata cache | Excluded - auto-regenerated from TMDb/TVDb on next library scan |
| Orphaned, zero-link Docker volumes from an earlier stack naming convention | Removed via `docker volume prune`, after confirming zero links via `docker system df -v` |

After these fixes, the real, meaningful backup size dropped to a few GB - the difference between a backup strategy that fits a free storage tier and one that doesn't.

## Problem 3: repository locks and Google Drive API throttling

Once size was under control, backups still failed intermittently with `403 RATE_LIMIT_EXCEEDED` - initially misdiagnosed as a SQLite file-locking issue, since the failures appeared to cluster around database files.

The actual root cause was layered:

1. **rclone's default OAuth credentials are shared globally** across every user who leaves `client_id`/`client_secret` blank during setup - meaning the rate limit being hit wasn't specific to this account's usage at all, it was a shared pool being exhausted by unrelated traffic.
2. Restic's default pack size meant a very high number of small API calls for the initial backup - each one counted against quota individually.
3. A **stale repository lock** from a previously interrupted run occasionally blocked new backup attempts entirely, requiring `restic unlock` before proceeding - a reminder that a failed run can leave state behind that needs explicit cleanup, not just a retry.

**Fix**: created a personal Google Cloud OAuth client instead of relying on rclone's shared defaults, increased restic's `--pack-size` to reduce total API call count, and tuned concurrency down (`RCLONE_TPSLIMIT`, `RCLONE_TRANSFERS`, `RCLONE_CHECKERS`) to stay well under quota even on a full initial run.

## Retention policy

```bash
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
restic check
```

Balances recovery granularity (7 daily snapshots for recent-mistake recovery) against long-term storage growth (monthly snapshots beyond that), with `restic check` run after every retention pass to catch repository corruption early rather than discovering it during an actual restore.

## Recovery validation

The step most commonly skipped in backup design, and the one that actually determines whether a backup strategy works:

```bash
restic restore latest --target /tmp/restore-test --path /mnt/ssd/data --path /mnt/ssd/docker/volumes
```

A backup that has never been restored from is a hypothesis, not a working recovery plan. This is run as an explicit verification step, not assumed to work because the backup job itself reports success.

## What this demonstrates

Backup and disaster recovery design isn't a single command - it's an iterative process of finding out what actually breaks (data consistency, storage growth, API limits, stale lock state) and designing around each of those failure modes deliberately. The current state is the fourth or fifth iteration of this system, not the first.
