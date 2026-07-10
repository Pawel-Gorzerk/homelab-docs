# Roadmap

This project is treated as continuously evolving, not a finished deliverable. Current priorities, in rough order:

## In progress / recently completed

- **CrowdSec enforcement gap closed** - the firewall bouncer was initially missing entirely, meaning CrowdSec had been detecting threats without ever blocking any of them. Caught during a deliberate end-to-end test, not by accident.
- **Bazarr** added to the automation pipeline for subtitle handling.
- **ntfy notification wiring** across monitoring, automation, and security layers - closing the gap between "something is being logged" and "someone finds out promptly."

## Planned

- **Authelia** - SSO/2FA layer in front of admin panels (Portainer, Grafana, NPM), reducing reliance on individually strong passwords per service as the number of exposed admin interfaces grows.
- **Disk health monitoring** - evaluated (Scrutiny) and deliberately deferred: the current SSD is behind a USB-to-SATA bridge that doesn't reliably pass through SMART data, so this needs either different hardware or an accepted gap, not a tool misconfigured against a hardware limitation it can't overcome.
- **Infrastructure-as-code coverage** - not every container in this environment started life in a tracked compose file. Bringing 100% of the stack under version control (using `docker-autocompose` as a starting point for legacy containers) is an ongoing cleanup effort, not a one-time task.

## Explicitly not planned

- **Kubernetes** - would be resume-driven complexity for a single-host environment with no actual scaling requirement. Docker Compose is the right tool for this scale; using something else just to have used it would work against the project's actual goal of demonstrating good judgment, not tool breadth.
- **Public internet exposure** - the LAN + Tailscale design is deliberate, not a limitation waiting to be removed. Nothing about the current workload requires public reachability, and the certificate/DNS strategy exists specifically to avoid needing it.
