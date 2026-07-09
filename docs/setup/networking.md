# Networking & DNS

## Overview

The Pi runs Pi-hole as the network's DNS server (not DHCP - the router handles that). This gives ad-blocking plus the ability to create local hostnames like `grafana.home.lan` that resolve only on the LAN.

## Static IP

Set on the Pi itself (not via router reservation) so it survives router changes:

```bash
ip a        # confirm current config
ip route    # confirm gateway
```

Configured via NetworkManager/dhcpcd depending on OS version, pinned to `192.168.33.55`.

## Pi-hole deployment

Pi-hole runs in **bridge mode**, not host mode - host mode was only ever needed when the Pi also handled DHCP (DHCP broadcasts don't traverse a Docker bridge cleanly). Since DHCP lives on the router, Pi-hole only needs its DNS and web ports published:

```yaml
pihole:
  image: pihole/pihole:latest
  networks:
    - proxy_net
  ports:
    - "53:53/tcp"
    - "53:53/udp"
    - "8082:80/tcp"   # admin UI moved off :80 to leave room for the reverse proxy
  environment:
    - TZ=Europe/Warsaw
    - FTLCONF_dns_listeningMode=all
```

!!! warning "IPv6 upstream DNS caution"
    An IPv6-only upstream DNS entry (e.g. Cloudflare's IPv6 address) will silently break *all* resolution if the network ever loses its IPv6 route - Pi-hole gets stuck endlessly retrying an unreachable server. Keep upstream DNS servers IPv4-only unless IPv6 connectivity is confirmed stable. See the [Troubleshooting Log](../troubleshooting.md#the-ipv6-dns-outage) for the incident this caused.

## Local DNS records

Every internal service gets a `.home.lan` entry in **Pi-hole → Local DNS → DNS Records**, pointing back to the Pi's own IP (`192.168.33.55`). Nginx Proxy Manager then routes by hostname to the correct container.

## Custom Docker network

All containers join a custom bridge network (not Docker's default `bridge`), since only custom networks get automatic container-name DNS resolution:

```yaml
networks:
  proxy_net:
    name: proxy_net
    driver: bridge
```
