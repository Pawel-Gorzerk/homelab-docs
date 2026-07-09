# Remote Access

## Tailscale as a subnet router

Installed directly on the Pi's host OS (not in a container) - this is the detail that matters, since only a host-level install can modify routing/iptables to advertise the **entire LAN subnet**, giving remote access to every device on the network through a single connection rather than exposing services one at a time.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=192.168.33.0/24 --accept-dns=false
```

The advertised route must be manually approved in the Tailscale admin console before it takes effect.

## Making local hostnames resolve remotely

Without extra config, `.home.lan` hostnames only resolve via Pi-hole - meaning IP-based access works over Tailscale, but the friendly hostnames don't. Fixed via **Tailscale's Split DNS**: add Pi-hole (`192.168.33.55`) as a custom nameserver, restricted to the exact `.home.lan` domain suffix.

!!! warning "Exact suffix match required"
    The domain suffix configured in Tailscale's DNS settings must match **character-for-character** what's actually used in Pi-hole's local DNS records. A typo here (e.g. `.home.lab` vs `.home.lan`) causes DNS queries for that suffix to silently fall through to public DNS instead of being forwarded to Pi-hole - the symptom looks identical to a broken Pi-hole, but the fix is purely in the Tailscale config.

Result: Pi-hole's ad-blocking and local hostname resolution both follow the device on mobile data, not just at home.
