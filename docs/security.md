# Security

Four layers, each addressing a different threat surface rather than one tool trying to cover everything.

## DNS-level filtering - Pi-hole

The first layer, and the cheapest: blocking known-malicious and ad-serving domains before a device on the network ever resolves them. This is coarse-grained (domain-level, not traffic-inspection-level) but catches a meaningful class of traffic before it goes anywhere.

## TLS everywhere it's actually needed - Nginx Proxy Manager

Not every internal service gets a public-CA-trusted certificate - only the ones where it's a functional requirement, not just a nice-to-have. Vaultwarden needs it because browsers disable the clipboard API and WebAuthn outside a secure context; ntfy needs it for web push notifications to work at all. Everything else runs over plain HTTP internally, since the traffic never leaves the LAN/Tailscale boundary.

Getting a *real*, browser-trusted certificate for internal-only services without exposing any port to the internet required a DNS-01 challenge instead of the more common HTTP-01 approach - full reasoning in [Architecture](architecture.md#design-decisions-worth-calling-out).

## Intrusion detection and prevention - CrowdSec

The most involved layer, and the one most likely to be set up incorrectly - CrowdSec's detection and enforcement are genuinely separate systems, and it's easy to end up with only the first.

**Detection**: the CrowdSec container parses logs against community-maintained attack scenarios. In this environment, it tails Nginx Proxy Manager's access logs directly via Docker's log API rather than a file mount:

```yaml
# acquis.d/npm.yaml
source: docker
container_name:
  - npm
labels:
  type: nginx
```

**Enforcement**: CrowdSec alone does not block anything - it only produces "decisions." A separate **bouncer** component is required to actually act on them. This environment uses the firewall bouncer, installed at the host level (not in Docker) so it can manipulate `iptables` directly, using an `ipset` to enforce bans at the network layer regardless of which internal service was targeted.

This detection/enforcement split, and the debugging required to get the bouncer actually talking to CrowdSec's API across the Docker network boundary, is documented in [Lessons Learned](lessons-learned.md).

## Credential management - Vaultwarden

Self-hosted, Bitwarden-protocol-compatible password manager, replacing reliance on a third-party cloud password vault for infrastructure and personal credentials alike. Its HTTPS requirement (above) isn't incidental - it's the reason the certificate strategy for this whole environment exists in the form it does.

## Why layer it this way

Each layer catches a different failure mode: DNS filtering stops known-bad destinations before a connection is even attempted; TLS protects credentials and session data in transit; CrowdSec responds to active attack patterns against exposed services; Vaultwarden reduces the blast radius of any single credential compromise. No single layer is sufficient on its own, and that's the point.
