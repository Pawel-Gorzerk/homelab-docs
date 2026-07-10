# Infrastructure

The services on this page are the foundation everything else depends on. If any of these fail, the whole environment degrades or becomes unmanageable - which is why they're treated differently from workload containers in terms of monitoring priority and change caution.

## Pi-hole - DNS

Acts as the authoritative DNS server for the LAN (DHCP is deliberately left to the router - see [Networking](networking.md) for the reasoning). Beyond ad-blocking, it provides local hostname resolution (`service.home.lan`) for every internal service, which is what makes the reverse proxy layer usable by name instead of by port number.

Runs in Docker bridge mode, not host mode. Host networking was only ever a requirement when this Pi also handled DHCP - DHCP broadcasts don't traverse a Docker bridge cleanly, but DNS has no such constraint.

## Nginx Proxy Manager - reverse proxy & TLS termination

Sits in front of every web UI in the stack. Two jobs: routing requests by hostname to the correct container, and terminating TLS so individual services don't each need their own certificate handling.

The certificate strategy here is deliberate, not default - see [Security](security.md) for why a DNS-01 challenge was used instead of the more common HTTP-01 approach, given this environment has no public-facing ports.

## Portainer - container management

GUI layer over the Docker Engine API. Used for stack deployment, log inspection, and container lifecycle management rather than reaching for raw `docker` CLI commands for every routine operation - though the CLI is still where actual diagnosis happens (see [Lessons Learned](lessons-learned.md) for examples).

## Homepage - service dashboard

A single-pane operational view: every service in the stack, with live-data widgets (queue depth, uptime status, container health) rather than static links, wherever the underlying service exposes an API for it. Functions as the first place to check system state before diving into individual service UIs or logs.

## Why these four, specifically

DNS, reverse proxy, container orchestration, and a unified view of system state are the categories that show up in almost every infrastructure environment, self-hosted or enterprise. Building and operating this layer - not just the workloads running on top of it - is the actual substance of this project.
