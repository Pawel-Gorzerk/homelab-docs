# Networking

## DNS and DHCP split

The router handles DHCP; the Pi handles DNS via Pi-hole. This split was a deliberate correction, not the original design - see the [DNS migration incident](lessons-learned.md#dns-migration-after-isp-change) for how this environment actually arrived here after an ISP change forced a full network reconfiguration.

Keeping DHCP on the router avoids a real operational risk: if the Pi ever goes down, devices still get an IP address and can still reach the internet - only internal `.home.lan` hostnames and ad-blocking are affected, not basic connectivity for the whole household.

## Static addressing

The Pi's IP is set on the host itself (`192.168.33.55`), not via a router-side DHCP reservation, so the address survives a router replacement or factory reset rather than depending on router-side configuration persisting.

## Custom Docker bridge network

Every container joins a custom bridge network (`proxy_net`) rather than Docker's default `bridge` network:

```yaml
networks:
  proxy_net:
    name: proxy_net
    driver: bridge
```

This matters because Docker's default bridge network provides no embedded DNS - containers can only address each other by IP, which changes on every recreate. A custom network gets automatic container-name resolution, so the reverse proxy can route to `grafana:3000` by name instead of a fragile hardcoded IP.

## Local hostname resolution

Every internal service gets a DNS record in Pi-hole (**Local DNS → DNS Records**) pointing back at the Pi's own IP. Nginx Proxy Manager then does the actual routing based on which hostname the request came in on. This two-layer design (DNS resolution, then reverse proxy routing) keeps hostname management and traffic routing as separate concerns, rather than baking IP addresses into every service's configuration directly.

!!! warning "IPv6 upstream DNS risk"
    A single unreachable IPv6 upstream DNS entry can silently break resolution entirely, not fail over gracefully - full writeup in [Lessons Learned](lessons-learned.md#the-ipv6-dns-outage).

## Remote access - Tailscale as a subnet router

Installed at the host OS level, not in a container - this is the detail that makes it work as intended. A host-level install can modify routing tables to advertise the **entire LAN subnet** (`192.168.33.0/24`), giving remote devices access to every host on the network through a single connection, rather than needing Tailscale installed on every service individually.

```bash
sudo tailscale up --advertise-routes=192.168.33.0/24 --accept-dns=false
```

The advertised route requires manual approval in the Tailscale admin console before it takes effect - a deliberate safety gate against a route being silently trusted.

### Making local hostnames resolve remotely

Without additional configuration, `.home.lan` hostnames only resolve through Pi-hole - meaning IP-based access works over Tailscale, but friendly hostnames don't. Fixed via Tailscale's **Split DNS**: Pi-hole added as a custom nameserver, scoped specifically to the `.home.lan` suffix.

!!! warning "Exact suffix match required"
    The domain suffix in Tailscale's DNS settings must match the suffix used in Pi-hole's records character-for-character. A mismatch (e.g. `.home.lab` vs `.home.lan`) causes DNS queries to silently fall through to public resolvers instead of reaching Pi-hole - the symptom looks identical to a broken Pi-hole, but the actual fault is entirely in the Tailscale configuration, not the DNS server.

Result: ad-blocking and local hostname resolution both follow the device on mobile data, not just at home - a small detail, but one that reflects treating remote access as a first-class part of the network design rather than a bolt-on.
