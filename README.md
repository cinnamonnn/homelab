# homelab

Architecture, security, and operations documentation for a self-hosted
Arch Linux server serving a small household -- media streaming,
file access/management, and automation, running on repurposed hardware
and directly exposed to the internet.

This repo documents my real, running system: how it's built, why it's
built that way, and every notable failure it has taught me along the way.

## The stack at a glance

- **Host**: Arch Linux on a Dell OptiPlex 9020 (i5-4590S, 8 GB) -- a 2014
  SFF box doing the work of a small server
- **Containers**: 20 services via Docker Compose, one project per app,
  progressively migrated off a managed platform onto standalone compose
- **Entry point**: Caddy reverse proxy with automatic TLS for all public
  subdomains; the only intentionally open ports on the host
- **Security**: direct public IP with no NAT buffer -- host firewall
  (ufw + ufw-docker) is load-bearing; every service binds to the smallest
  network ring that serves its audience
- **Automation**: systemd-timer music pipeline (download → two-pass
  metadata import → library), monitoring and alerting stack, Docker log
  rotation
- **Ops discipline**: post-update verification ritual including external
  port scans; key-only SSH; remote admin via SSH port forwarding

## Documentation

- [Architecture](docs/architecture.md) -- the host, service model, and
  network rings
- [Network & Security](docs/network-security.md) -- the layered posture
  behind a direct public IP, and the Docker-vs-firewall footgun it depends
  on managing
- [Operations](docs/operations.md) -- deployment checklist, update ritual,
  monitoring, logging, and known gaps
- [Troubleshooting](docs/troubleshooting.md) -- 17 documented production
  issues with root causes, fixes, and the lessons that became procedure.
  **The debugging log is the best part!**

## Why this repo exists

I'm transitioning into IT, and this server is where I practice the real
thing: designing a system, running it under load for actual users, and
fixing it when it breaks. Everything documented here happened in
production -- there is no staging environment, only me and an SSH session.

The most interesting constraints: **the hardware is old enough that
hardware transcoding is unsupported, and the IP is public enough that
nothing hides by accident.** Both forced engineering decisions I'd never
have made on newer gear behind a NAT -- and both are documented honestly,
including what's still unsolved.

## Known gaps

Backups are the biggest open item -- configuration snapshots exist, but
there's no automated, tested backup routine yet, due to storage constraints.
It's the top of the infrastructure roadmap, documented in
[operations](docs/operations.md#known-gaps-tracked).
