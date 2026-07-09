# Security

## CrowdSec

An intrusion detection **and prevention** system - but the two halves are separate, and it's easy to end up with only the first.

### The detection half
The CrowdSec container parses logs (in this setup, NPM's access logs, tailed live via Docker's log API rather than a file mount) against community-maintained scenarios, producing "decisions" (bans) when it spots malicious patterns.

```yaml
# /etc/crowdsec/acquis.d/npm.yaml
source: docker
container_name:
  - npm
labels:
  type: nginx
```

```bash
docker exec crowdsec cscli collections install crowdsecurity/nginx
```

### The enforcement half (the part that's easy to skip)
CrowdSec alone **does not block anything** - it only produces decisions. A **bouncer** is required to actually enforce them. This stack uses the firewall bouncer, installed on the host (not in Docker) so it can manipulate iptables directly:

```bash
sudo apt install crowdsec-firewall-bouncer-iptables -y
docker exec crowdsec cscli bouncers add firewall-bouncer-1
```

The bouncer's `api_url` needs to point somewhere it can actually reach - `127.0.0.1` only works if CrowdSec's API port is published to the host. If it's only reachable inside the Docker network, use the container's bridge-network IP instead, or (cleaner, more stable across container recreates) publish the port bound to localhost only.

**Verifying enforcement actually works:** CrowdSec's bouncer typically uses an `ipset` rather than one iptables rule per banned IP, so checking `iptables -L` directly for a specific IP won't show anything even when it's working correctly:

```bash
docker exec crowdsec cscli decisions add --ip 1.2.3.4 --duration 5m --reason "test"
sudo ipset list crowdsec-blacklists | grep 1.2.3.4
```

## Vaultwarden

Self-hosted Bitwarden-compatible password manager. Requires a genuine HTTPS secure context - browsers disable the clipboard API and WebAuthn over plain HTTP, which Vaultwarden depends on for core functionality. See [Reverse Proxy & HTTPS](reverse-proxy.md) for how this is achieved without exposing anything to the internet.

```yaml
vaultwarden:
  image: vaultwarden/server:latest
  environment:
    - WEBSOCKET_ENABLED=true    # needed for live sync to Bitwarden apps
    - SIGNUPS_ALLOWED=false     # flip true briefly for initial account creation only
```
