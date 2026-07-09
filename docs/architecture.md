# Architecture

## Network topology

Everything runs on a single Raspberry Pi 5 at a static LAN IP, with Pi-hole handling DNS and Nginx Proxy Manager (NPM) terminating HTTPS for internal hostnames. All containers share a custom Docker bridge network (`proxy_net`) rather than the default `bridge` network, since only custom networks get Docker's embedded container-name DNS resolution.

```mermaid
graph TB
    subgraph Internet
        LE[Let's Encrypt]
        GD[Google Drive]
    end

    subgraph Router["Home Router"]
        DHCP[DHCP Server]
    end

    subgraph Pi["Raspberry Pi 5 - 192.168.33.55"]
        subgraph proxy_net["Docker network: proxy_net"]
            PH[Pi-hole<br/>DNS :53]
            NPM[Nginx Proxy Manager<br/>:80 :443 :81]
            JF[Jellyfin]
            JS[Jellyseerr]
            SO[Sonarr]
            RA[Radarr]
            PR[Prowlarr]
            BZ[Bazarr]
            QB[qBittorrent]
            GR[Grafana]
            PM[Prometheus]
            UK[Uptime Kuma]
            VW[Vaultwarden]
            NT[ntfy]
            N8[n8n]
            HP[Homepage]
            POR[Portainer]
            CS[CrowdSec]
        end
        TS[Tailscale<br/>subnet router]
        FB[CrowdSec<br/>Firewall Bouncer<br/>host-level]
    end

    subgraph Client["Client Devices"]
        Phone[Phone]
        Laptop[Laptop]
    end

    DHCP -->|assigns IPs| Client
    Client -->|DNS queries| PH
    PH -->|local records| NPM
    Client -->|HTTPS| NPM
    NPM --> JF
    NPM --> JS
    NPM --> GR
    NPM --> UK
    NPM --> VW
    NPM --> NT
    NPM --> HP
    NPM --> POR
    NPM -->|DNS-01 challenge| LE
    CS -->|decisions| FB
    FB -->|enforces bans| Pi
    TS -.->|remote access,<br/>no exposed ports| Client
    Pi -->|encrypted backups| GD

    style Pi fill:#1a1a2e,stroke:#4a4a6a,color:#fff
    style proxy_net fill:#16213e,stroke:#4a4a6a,color:#fff
```

## Media automation pipeline

A request flows from a friend/family member all the way to a playable file in Jellyfin without any manual intervention.

```mermaid
sequenceDiagram
    actor User
    participant Jellyseerr
    participant Radarr
    participant Prowlarr
    participant qBittorrent
    participant Bazarr
    participant Jellyfin

    User->>Jellyseerr: Request a movie
    Jellyseerr->>Radarr: Add + monitor (via API)
    Radarr->>Prowlarr: Search indexers
    Prowlarr-->>Radarr: Matching releases
    Radarr->>qBittorrent: Send best release
    qBittorrent-->>Radarr: Download complete
    Radarr->>Radarr: Rename + hardlink into /media/movies
    Radarr->>Jellyfin: Notify (Connect webhook)
    Bazarr->>Radarr: Poll for new library items
    Bazarr->>Bazarr: Fetch matching subtitles
    Jellyfin-->>User: Movie appears, ready to watch
```

## Backup flow

SQLite databases are snapshotted consistently *before* the main backup runs, so nothing is ever backed up mid-write.

```mermaid
graph LR
    A[Scan for every<br/>.db / .sqlite file] --> B["sqlite3 .backup<br/>(consistent copy)"]
    B --> C[Staging directory]
    D[Config directories] --> F[restic backup]
    E[Docker named volumes] --> F
    C --> F
    F -->|encrypted, deduplicated| G[rclone]
    G --> H[Google Drive]
    F --> I[restic forget<br/>retention policy]
    F --> J[restic check<br/>integrity]

    style F fill:#16213e,stroke:#4a4a6a,color:#fff
```

## Design decisions worth calling out

**Why a custom bridge network instead of Docker's default `bridge`?**
Docker's default `bridge` network has no embedded DNS - containers can only reach each other by IP. A custom network (`proxy_net`) gets automatic container-name resolution, so NPM can proxy to `grafana:3000` instead of a hardcoded IP that changes on every recreate.

**Why DNS-01 challenge instead of exposing port 80/443 publicly?**
The entire stack is LAN-only by design. Using a DuckDNS subdomain purely for the DNS-01 Let's Encrypt challenge gets a *real*, browser-trusted certificate (required for Vaultwarden's clipboard API and WebAuthn, and for ntfy's push notifications - both need a secure browser context) without ever exposing a port to the internet. Pi-hole then overrides that public domain to resolve locally.

**Why Tailscale as a host-level subnet router instead of per-device or in-container?**
Running it directly on the Pi and advertising the whole `/24` subnet means every device on the tailnet gets access to the entire LAN through one connection - no need to install Tailscale on every service individually, and no ports ever opened on the router.

**Why SQLite `.backup` instead of stopping containers for backups?**
Nearly every service in this stack (Sonarr, Radarr, Prowlarr, Jellyseerr, Jellyfin, NPM, CrowdSec, Grafana, Pi-hole, Vaultwarden, n8n, Uptime Kuma) uses embedded SQLite. Stopping all of them nightly would mean real downtime. SQLite's Online Backup API produces a fully consistent copy of a live database with zero downtime, so the backup script scans for every `.db`/`.sqlite` file generically rather than hardcoding a per-service stop list.
