# Backups

## Design

Two categories of data, treated deliberately differently:

- **Config directories & Docker named volumes** - small, critical, irreplaceable. Backed up.
- **Media library files** - large, but fully replaceable via the automation pipeline itself. Not backed up directly; Sonarr/Radarr's own databases (which *are* backed up) remember what existed, so the library can be rebuilt.

## SQLite consistency without downtime

Nearly every container in this stack embeds SQLite (Sonarr, Radarr, Prowlarr, Jellyseerr, Jellyfin, NPM, CrowdSec, Grafana, Pi-hole, Vaultwarden, n8n, Uptime Kuma - more services than it first appears). Rather than stopping each one before backup, every `.db`/`.sqlite`/`.sqlite3` file is discovered generically and snapshotted live via SQLite's own **Online Backup API**, which safely produces a consistent copy of an actively-written database with zero downtime:

```bash
sqlite3 "$db" ".backup '$STAGING/${rel}.backup'"
```

## What gets excluded, and why

| Excluded | Reason |
|---|---|
| `node_modules` (inside n8n's data dir) | Reproducible via reinstalling community nodes - not unique data |
| `prometheus_data` volume | Disposable time-series metrics, not configuration |
| Jellyfin's `metadata/library` cache | Auto-regenerated from TMDb/TVDb on next scan |
| Orphaned/anonymous Docker volumes | Leftovers from renamed stacks - `docker volume prune` first |

Getting these exclusions right took an early backup run from **28GB down to a few GB** - worth doing before wiring up any cloud backend with a storage quota.

## Restic + rclone + Google Drive

```bash
export RCLONE_CONFIG=/home/gorlab/.config/rclone/rclone.conf
export RCLONE_TPSLIMIT=3
export RCLONE_TRANSFERS=2
export RCLONE_CHECKERS=4
export RESTIC_REPOSITORY="rclone:gdrive:homelab-restic-backup"
```

!!! tip "Use your own Google Cloud OAuth credentials"
    Leaving `client_id`/`client_secret` blank during `rclone config` uses rclone's **shared** default application credentials - meaning API rate limits are shared across every rclone user globally using default settings. Creating a free personal Google Cloud project with its own OAuth client eliminates most throttling entirely, since traffic then counts against a quota that's genuinely yours.

## Retention

```bash
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
restic check
```

## Restore verification

The step most commonly skipped, and the one that actually matters:

```bash
restic restore latest --target /tmp/restore-test --path /mnt/ssd/data --path /mnt/ssd/docker/volumes
```

A backup that's never been restored from is unproven.
