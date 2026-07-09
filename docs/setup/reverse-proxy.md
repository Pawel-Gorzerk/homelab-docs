# Reverse Proxy & HTTPS

## Nginx Proxy Manager

Fronts every web UI in the stack, terminating both plain HTTP (`.home.lan` hostnames) and real HTTPS (for the two services that specifically require a secure browser context).

```yaml
npm:
  image: jc21/nginx-proxy-manager:latest
  networks:
    - proxy_net
  ports:
    - "80:80"
    - "443:443"
    - "81:81"     # admin GUI
```

!!! tip "Backend scheme matters"
    Every proxy host's **Forward Scheme** must be `http`, not `https` - containers serve plain HTTP internally; SSL termination happens at NPM. Setting this to `https` by mistake produces a 502 with no obvious cause.

## Getting a real, trusted certificate without exposing anything

Since the whole stack is LAN-only, there's no port 80/443 exposed to the internet - which normally rules out Let's Encrypt's standard HTTP challenge. The fix: a **DNS-01 challenge** via a free DuckDNS subdomain.

1. Create a free subdomain at [duckdns.org](https://duckdns.org) (e.g. `yourlab.duckdns.org`)
2. In NPM: **SSL Certificates → Add → Let's Encrypt → DNS Challenge → DuckDNS provider**, paste the DuckDNS token
3. Request it as a **wildcard**: `*.yourlab.duckdns.org` - one cert covers every subdomain
4. In Pi-hole, override resolution of those subdomains back to `192.168.33.55` - the domain is real (so the cert is trusted) but never resolves publicly

### Why not just use `.home.lan` for everything?

Local TLDs like `.lan`/`.home`/`.lab` aren't real, publicly-recognized domains - no certificate authority can ever issue a trusted cert for them. Only the services that genuinely need a secure context (Vaultwarden's clipboard/WebAuthn APIs, ntfy's push notifications) were moved to the DuckDNS domain; everything else stays on `.home.lan` over plain HTTP, since mixing schemes across *different* services causes no issues.

## Homepage dashboard host validation

Homepage validates the `Host` header against an allowlist - reaching it through NPM's proxied hostname requires:

```yaml
environment:
  - HOMEPAGE_ALLOWED_HOSTS=dashboard.home.lan,192.168.33.55,localhost
```
