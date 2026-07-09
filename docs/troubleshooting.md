# Troubleshooting Log

Real incidents encountered while building and operating this stack, kept here rather than sanitized out - the debugging process is often more instructive than the final config.

## The IPv6 DNS outage

**Symptom:** After a power outage, mobile devices got `DNS_PROBE_FINISHED_NXDOMAIN` for every local hostname, but other devices seemed fine.

**Root cause:** Pi-hole had an IPv6 upstream DNS server configured (Cloudflare's `2606:4700:4700::1111`). The router lost its IPv6 route after the outage, so Pi-hole was endlessly retrying an unreachable server - `Network unreachable` errors flooding its logs - which choked its ability to resolve *anything*, local or public.

**Fix:** Removed IPv6 upstream entries, kept IPv4-only (`1.1.1.1`, `8.8.8.8`).

**Lesson:** A single unreachable upstream can silently take down DNS entirely, not just fail over gracefully. Worth checking upstream DNS logs specifically when "everything seems broken" after any kind of network change.

---

## The watchtower crash loop

**Symptom:** Following the DNS incident above, a Docker network health check revealed a *new* veth network device being created almost exactly every 60 seconds - the signature of a container stuck in a restart loop.

**Root cause:** A cron job was independently starting/stopping Watchtower on a schedule, while Watchtower's own `restart: unless-stopped` policy fought against that - every time it exited after a scheduled run, Docker's restart policy immediately relaunched it, which then failed, restarted, failed again. **1,075 restarts** by the time it was caught.

**Fix:** `docker update --restart=no watchtower` - since the cron job now solely controls its lifecycle, the container's own restart policy needed to step aside entirely.

**Lesson:** Two independent lifecycle-management mechanisms on the same container (an external scheduler + Docker's built-in restart policy) will eventually conflict. Pick one.

---

## The Prowlarr vs. Radarr confusion

**Symptom:** A movie downloaded successfully via qBittorrent, but never appeared in Jellyfin - Radarr showed it as "Missing" despite the file existing on disk.

**Root cause:** The download was initiated directly from **Prowlarr's** own search interface, not Radarr's. Prowlarr exists purely as shared indexer configuration for Sonarr/Radarr - grabbing a release from Prowlarr's own UI sends it straight to the download client with Radarr never informed, so Radarr never watches for its completion or imports it.

**Fix:** Always initiate grabs from within Radarr/Sonarr (automatically, or via their own Interactive Search) - never from Prowlarr's UI directly.

---

## The category-mapping dead end

**Symptom:** Radarr's automatic *and* interactive search both returned zero results for a Polish-language private tracker, despite the tracker clearly having relevant content and the connection test passing.

**Root cause:** The tracker used custom, non-standard category IDs (`100072 Filmy x264/h264/1080p`, etc.) that weren't mapped to Radarr's expected standard Torznab categories. Radarr silently filters out anything it can't categorize as a movie - it never surfaces as an error, just empty results.

**Fix:** Explicit category mapping in **Prowlarr → Indexers → [indexer] → Categories**.

---

## The Google Drive rate-limit chase

**Symptom:** `restic backup` to a Google Drive rclone remote repeatedly failed with `403 RATE_LIMIT_EXCEEDED`, initially misdiagnosed as a SQLite file-locking issue since the backup appeared to "freeze" on database files.

**Root cause, in layers:**

1. First backup attempt was **28GB** - `node_modules` inside n8n's data directory, Prometheus's raw TSDB, and Jellyfin's regenerable artwork cache were all being backed up unnecessarily.
2. After trimming that down, throttling continued - traced to rclone's **default OAuth credentials being shared globally across every rclone user**, not a quota specific to this account.
3. A stale `rclone.conf` file ownership issue (flipped to `root` after a `sudo`-run backup triggered a background token refresh) caused a *separate*, unrelated permission error that looked related but wasn't.

**Fix:** Created a personal Google Cloud OAuth client instead of using rclone's shared defaults; increased restic's `--pack-size` to reduce total API call count; tuned `RCLONE_TPSLIMIT`/`RCLONE_TRANSFERS` down; excluded genuinely disposable data.

**Lesson:** When a cloud backend throttles unexpectedly at a scale that shouldn't trigger rate limits, check whether default/shared API credentials are in play before assuming the data volume itself is the problem.

---

## The orphaned Docker volumes

**Symptom:** `docker system df -v` revealed several volumes with **zero container links** - `piholenginx_npm_data`, `piholenginx_pihole_etc`, and multiple randomly-hashed anonymous volumes, one alone accounting for nearly 1GB.

**Root cause:** Leftovers from an earlier stack naming convention (`piholenginx_*`) before it was renamed to `npmpihole_*`, plus anonymous volumes from containers that had since been recreated with explicit bind mounts instead.

**Fix:** `docker volume prune` after confirming zero links via `docker system df -v` - Docker refuses to delete any volume still attached to a running container, making this safe to run without manually cross-checking each one.
