# Lessons Learned

Real incidents encountered while building and operating this environment, documented in the same structure used for production incident reports: problem, investigation, root cause, solution, and what changed afterward. Nothing here is sanitized or reconstructed after the fact - these are written close to when each incident actually happened.

---

## DNS migration after ISP change

**Problem**
A router and ISP change broke every container's network connectivity. The Pi's previous static IP no longer existed on the new subnet, and the network's default gateway had changed - Docker's bridge networking itself was unaffected, but the host could no longer reach the LAN or internet correctly.

**Investigation**
Checked the host's actual network state first (`ip a`, `ip route`) rather than assuming the failure was in Docker - confirmed the interface was still configured for the old subnet entirely, which meant nothing above the network layer had a chance of working until this was corrected.

**Root cause**
The new router used a different LAN subnet and gateway than the old one, and the Pi's static IP configuration was still pinned to the old addressing scheme. Separately, the new ISP setup removed a constraint the original design was built around (the router's own DNS server option being ISP-blocked, which was why Pi-hole had originally been deployed in `network_mode: host` to also handle DHCP) - the network change was also an opportunity to correct that design decision, not just restore the old one.

**Solution**
Reconfigured the Pi's static IP for the new subnet and gateway, then used the opportunity to move Pi-hole out of host networking entirely - since DHCP could now be left to the router, Pi-hole only needed to handle DNS, removing the host-networking requirement altogether and freeing ports 80/443 on the host for a reverse proxy. This is the origin of the current DNS/DHCP split described in [Networking](networking.md).

**What changed afterward**
An infrastructure assumption (host networking required for Pi-hole) that was true under one set of constraints stopped being true once those constraints changed - and the original design hadn't been revisited until forced to by an unrelated external change. This is now a standing question whenever any upstream dependency changes (ISP, router firmware, cloud provider defaults): not just "what broke," but "which of my existing design decisions were built around a constraint that may no longer apply."

---

## The IPv6 DNS outage

**Problem**
Following a power outage, mobile devices on the network returned `DNS_PROBE_FINISHED_NXDOMAIN` for every internal hostname. Other devices appeared unaffected at first glance, which initially pointed investigation in the wrong direction (a device-specific DHCP/DNS assignment issue) rather than the server itself.

**Investigation**
Checked container health (`docker ps`, `docker logs`) before assuming a network-layer fault. Pi-hole's logs showed a continuous stream of `Network unreachable` errors against a specific upstream address, rather than generic timeouts - which was the actual signal that mattered, missed on first read-through.

**Root cause**
Pi-hole had an IPv6 upstream DNS server configured (Cloudflare, `2606:4700:4700::1111`). The router lost its IPv6 route after the outage, and Pi-hole had no fallback behavior for an upstream it could reach at the protocol level but never get a response from - it retried endlessly, and that retry loop degraded its ability to resolve *anything*, not just queries destined for that specific upstream.

**Solution**
Removed the IPv6 upstream entry, standardized on IPv4-only upstreams (`1.1.1.1`, `8.8.8.8`).

**What changed afterward**
IPv6 upstream DNS is now treated as something that requires confirmed-stable IPv6 connectivity before being reintroduced, not a default to leave enabled. A single unreachable upstream turned out to be capable of taking down DNS resolution entirely, rather than degrading gracefully - worth checking upstream-specific logs early whenever "everything" seems broken after any network change.

---

## The Watchtower crash loop

**Problem**
While investigating the DNS outage above, a routine network health check revealed something unrelated but more serious: a new virtual network interface (`veth`) being created almost exactly every 60 seconds - the signature of a container stuck in a continuous restart loop, actively churning Docker's networking layer in the background.

**Investigation**
`docker ps -a` and a restart-count check across every container (`docker inspect --format '{{.RestartCount}}'`) isolated it immediately to a single container with over 1,000 restarts, versus zero for everything else.

**Root cause**
An external cron job independently started and stopped Watchtower on a nightly schedule, while Watchtower's own `restart: unless-stopped` Docker policy was still active. Every time the cron-managed run exited, Docker's restart policy treated that as an unexpected stop and relaunched it immediately - which then failed to start cleanly in that context, restarted again, and repeated indefinitely.

**Solution**
`docker update --restart=no watchtower` - removed Docker's restart policy entirely, leaving the cron job as the sole authority over the container's lifecycle.

**What changed afterward**
Any container with an external lifecycle controller (cron, a separate scheduler) now gets its Docker restart policy explicitly reviewed at setup time, rather than left at whatever default was used when the container was first deployed. Two independent systems both assuming ownership of the same resource's lifecycle is a structural problem, not a one-off misconfiguration - it will recur with any container set up the same way.

---

## The bypassed orchestration layer

**Problem**
A movie downloaded successfully and completed in qBittorrent, but never appeared in the Jellyfin library. Radarr showed the title as "Missing" despite the file existing on disk.

**Investigation**
Confirmed the file existed at the expected path, confirmed the download client's connection test passed from within Radarr, then walked the actual request history - Radarr's queue and history showed no record of the download ever happening, which was the key clue: not a failed import, but an import Radarr never knew it needed to perform.

**Root cause**
The download had been initiated directly through Prowlarr's own search interface rather than through Radarr. Prowlarr exists purely as shared indexer configuration for Sonarr/Radarr - triggering a grab from Prowlarr's own UI sends the download straight to qBittorrent with Radarr never informed, so nothing ever watches for its completion or performs the library import.

**Solution**
Re-triggered the search and grab from within Radarr itself; the existing file was recovered via manual import rather than re-downloaded.

**What changed afterward**
This is a class of failure worth generalizing: any architecture with a shared configuration layer (Prowlarr) sitting in front of multiple orchestrators (Sonarr, Radarr) needs a clear operational rule about which interface actually initiates action versus which one only configures shared resources. The tooling doesn't prevent the wrong entry point from being used - only the operator's understanding of the architecture does.

---

## The category-mapping dead end

**Problem**
Automatic and manual search in Radarr both returned zero results for a specific private tracker, despite the connection test passing and the tracker clearly having relevant content when checked independently.

**Investigation**
Ruled out authentication and connectivity first (both fine). The actual cause wasn't visible in any error message - both search paths simply returned an empty result set with no failure indication, which made this a genuinely harder diagnosis than a typical error-driven investigation.

**Root cause**
The tracker used non-standard, region-specific category IDs (e.g. `100072 Filmy x264/h264/1080p`) instead of the standard Torznab category scheme Radarr expects. Without explicit mapping between the two, Radarr silently filtered out every result as unclassifiable - correct behavior from its perspective, but with no visible signal that this filtering was happening.

**Solution**
Explicit category mapping configured in Prowlarr, between the tracker's custom category IDs and Radarr's standard categories.

**What changed afterward**
Confirmed this as a standing checklist item whenever a new, non-mainstream indexer is added: verify not just that a connection works, but that the data it returns is actually classifiable by whatever's consuming it. A passing connection test and a functionally working integration are not the same thing.

---

## The Google Drive rate-limit chase

**Problem**
Scheduled backups to a cloud storage backend failed intermittently with `403 RATE_LIMIT_EXCEEDED`. The failures initially appeared to correlate with specific database files, leading to an early (incorrect) hypothesis of a file-locking issue.

**Investigation**
Ruled out file locking directly by testing SQLite's backup mechanism against a live database in isolation - it worked cleanly outside the backup pipeline, which eliminated that hypothesis and redirected investigation toward the upload layer itself. From there, reading the actual API error response (rather than just the surface-level failure message) revealed it was a quota/rate-limit response, not a data-access error.

**Root cause, in three layers**
1. The first full backup attempt was significantly larger than expected, due to reproducible/disposable data (dependency directories, raw metrics storage, regenerable cache) being included unnecessarily - more upload volume than needed, though not the actual cause of the rate limiting itself.
2. The real cause: the cloud storage client's default OAuth credentials are **shared globally** across every user who doesn't configure their own - meaning the quota being exhausted wasn't specific to this account's actual usage at all.
3. A secondary, unrelated permission issue (a configuration file's ownership silently changing after a privileged process modified it) surfaced around the same time and briefly appeared connected to the rate-limiting problem, but was independently diagnosed and fixed.

**Solution**
Created dedicated OAuth credentials for this specific use instead of relying on shared defaults, increased the backup tool's batch/pack size to reduce total API call count, and tuned upload concurrency down to stay comfortably under quota even during a full initial backup.

**What changed afterward**
When a cloud backend throttles at a scale that shouldn't reasonably trigger a rate limit, checking whether default/shared credentials are in play is now an early step, not a late one - it changes the entire direction of the investigation. This also reinforced not diagnosing multiple concurrent issues as if they were one: the file size problem, the credential-sharing problem, and the file-permission problem were three separate root causes that happened to surface in the same debugging session.

---

## The orphaned Docker volumes

**Problem**
Disk usage review revealed several Docker-managed volumes consuming meaningful space with **zero containers referencing them** - one alone accounting for nearly a gigabyte.

**Investigation**
`docker system df -v` surfaces per-volume size alongside a link count - a link count of zero is Docker's own signal that nothing currently depends on that volume, which made this a low-risk cleanup rather than one requiring manual cross-referencing against every running container.

**Root cause**
Leftovers from an earlier project naming convention, before a stack was renamed - the old named volumes were never explicitly removed when the rename happened, they simply stopped being referenced. A second contributor: containers that had since been redeployed using explicit bind mounts instead of Docker-managed volumes, leaving their original anonymous volumes orphaned with no name to even identify their origin.

**Solution**
`docker volume prune`, after confirming via the link-count check that nothing currently in use would be affected - Docker itself refuses to delete a volume still attached to a running container, which made this safe to run without a more manual audit.

**What changed afterward**
Renaming a stack or migrating a service from named volumes to bind mounts now includes an explicit cleanup step for whatever it leaves behind, rather than treating the rename/migration as complete once the new configuration is running. Orphaned state doesn't announce itself - it just quietly accumulates until something like a disk-usage review surfaces it.
