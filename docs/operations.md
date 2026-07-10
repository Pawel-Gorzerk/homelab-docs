# Operations

Tools and configuration are the visible part of this project. This page is about the operating discipline underneath them - the part that doesn't show up in a `docker-compose.yml` but determines whether an environment stays reliable over time.

## Monitoring is only useful if someone reacts to it

A dashboard nobody looks at and an alert nobody acts on are functionally equivalent to having no monitoring at all. Every monitoring integration in this environment (see [Monitoring](monitoring.md)) is deliberately paired with a notification path - the design goal was never "have metrics," it was "know within minutes when something needs attention, without having to go looking for it."

## Backups are unproven until they're restored

A backup job reporting success confirms the job ran - it says nothing about whether the resulting archive can actually be restored from. This environment's backup process includes an explicit restore-verification step for exactly this reason (see [Backup & Disaster Recovery](backup.md)). Treating "backup completed" and "recovery works" as the same fact is a common and avoidable failure mode.

## Automation must be tested, not just deployed

The Watchtower crash loop (see [Lessons Learned](lessons-learned.md#the-watchtower-crash-loop)) happened because two automated systems' interaction was never actually verified - each individually worked as designed, but their combination didn't. Automating a process removes manual effort; it doesn't remove the need to verify the automation itself behaves correctly under real conditions.

## Every incident should improve the architecture, not just get patched

The pattern followed throughout this project's [Lessons Learned](lessons-learned.md) log isn't "find the broken setting, change it, move on." Each incident fed back into a structural change: the IPv6 DNS outage led to a standing rule about upstream DNS configuration, not just a one-time fix; the orphaned Docker volumes incident led to routinely checking `docker system df -v` after any stack rename, not just cleaning up that one instance. Root cause analysis that doesn't change future behavior is incomplete.

## Documentation is part of the system, not separate from it

This entire site exists because undocumented infrastructure is infrastructure only one person can operate, troubleshoot, or hand off - and even that one person will eventually forget the reasoning behind a decision made months earlier. Every setup guide and every incident on this site was written close to when it happened, not reconstructed afterward from memory.

## Change is expected to be iterative

Nothing on this site represents a finished state. The backup system alone has gone through several distinct redesigns as new failure modes surfaced (see [Backup & Disaster Recovery](backup.md)); the security layer was built incrementally, with CrowdSec initially deployed in a detection-only state before the enforcement gap was caught and closed. Good infrastructure isn't built once - it's continuously operated, monitored, and revised. See [Roadmap](roadmap.md) for what's currently planned next.
