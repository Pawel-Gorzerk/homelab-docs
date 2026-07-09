# Media Automation

## The pipeline

See the [sequence diagram](../architecture.md#media-automation-pipeline) for the full flow. In short: **Jellyseerr is the front door** - requests made there flow automatically through Sonarr/Radarr, Prowlarr, and qBittorrent, landing in Jellyfin without manual intervention.

!!! note "A common point of confusion"
    Searching or downloading directly from **Prowlarr's own interface** bypasses Sonarr/Radarr entirely - the file lands in the downloads folder, but Sonarr/Radarr never learn it exists, so it never gets imported into the library. Always let Sonarr/Radarr initiate the grab (automatically, or via their own Interactive Search) - Prowlarr exists purely as their shared indexer configuration layer.

## Shared filesystem paths matter

Sonarr, Radarr, and qBittorrent all mount the **same parent path** for downloads and media:

```yaml
volumes:
  - /mnt/ssd/shared/media:/media
  - /mnt/ssd/shared/downloads:/downloads
```

This lets Sonarr/Radarr **hardlink** finished downloads into the media library instead of copying them - instant, no duplicated disk usage, and the torrent keeps seeding from its original location while Jellyfin also sees it in the library. Hardlinks only work within the same filesystem/mount, which is why this path structure matters.

## Root folders

- Sonarr: `/media/tv`
- Radarr: `/media/movies`

Both must match subfolders under the same path Jellyfin scans (`/mnt/ssd/shared/media` on the host).

## Category mapping (private/regional trackers)

Non-standard indexers (e.g. region-specific trackers with custom category schemes) need their categories explicitly mapped to Radarr/Sonarr's expected standard categories (`2000` Movies, `5000` TV, etc.) inside **Prowlarr → Indexers → [indexer] → Categories**. An unmapped category means Radarr's automatic search silently returns zero results even though the indexer itself works fine - the fix is mapping, not touching Radarr's search logic.

## Bazarr - automated subtitles

Sits alongside Sonarr/Radarr, polling them for library changes and fetching matching subtitles automatically:

```yaml
bazarr:
  image: lscr.io/linuxserver/bazarr:latest
  volumes:
    - /mnt/ssd/data/bazarr/config:/config
    - /mnt/ssd/shared/media:/media
```

Configured with Sonarr/Radarr as sources (same API-key pattern as Prowlarr's connections) and at least one subtitle provider.

## Jellyfin auto-refresh

Without this, Jellyfin waits for its own scheduled scan to notice new files. Wiring Sonarr/Radarr's **Connect** setting to Jellyfin triggers an instant library refresh on import instead:

```
Settings > Connect > Add Connection > Jellyfin
  Host: jellyfin, Port: 8096
  API Key: from Jellyfin's own Advanced > API Keys
  Triggers: On Import, On Upgrade
```
