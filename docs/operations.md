# Operations

How this system is run day-to-day: deployment procedure, update ritual,
monitoring, and logging. The recurring design principle: **every operation
gets a cheap verification step** -- trust is earned per layer, and each layer
has one command that proves it's healthy before the next layer gets blamed.

## Deploying a new service

Every service follows the same checklist, derived from the failure modes in
the [troubleshooting log](troubleshooting.md):

0. Create the app directory and data directories, `chown 1000:1000` them
   **before** first start -- Docker auto-creates missing bind-mount dirs as
   root, which blocks non-root containers (#5).
1. Write `docker-compose.yml` (diffed against the project's stock compose
   line by line -- env defaults are load-bearing, #8) and add the reverse
   proxy block, validating the proxy config before reload.
2. `docker compose up -d`, wait, then `docker ps` for health.
3. Verify the publish actually took: `docker port <name>` (declared ≠ live,
   #3), then `curl -I` against the bound address -- loopback for
   proxy-fronted apps, LAN address for admin tools.

## Update ritual

The server sits directly on a public IP with no NAT buffer, so the firewall
layer is load-bearing -- updates don't end at "packages upgraded":

0. Check distribution news and manual-intervention notices before upgrading
   (rolling-release discipline).
1. Upgrade the host (`pacman -Syu`), merging pacnew files.
2. Update container images per project: `docker compose pull && up -d`.
3. Refresh the Docker-aware firewall rules (`ufw-docker update`) -- new
   containers don't inherit rules.
4. External port scan from an outside network (e.g. phone hotspot):
   only the intended ports open, everything else filtered -- verified on
   both IPv4 and IPv6.
5. Check systemd timers are still green.
6. Check `docker compose ps` for containers stuck restarting, and
   `findmnt` on data volumes if Docker restarted after any storage
   operation (#7).

## Monitoring & alerting

- **Uptime Kuma**: service health for the full app stack plus one
  end-to-end check of the public chain (DNS → TLS → proxy → app).
- **Glances + Speedtest Tracker**: host metrics (disk, CPU, network)
  feeding a Homarr dashboard.
- **Tautulli**: media-server activity -- who's streaming, whether streams
  transcode (which matters, given CPU-only transcoding), and history.
- **Dozzle** (on probation): live container log tailing.

The design intent: one glance at the dashboard answers "is anything broken,
and where in the chain?"

## Logging

- **journald**: persistent, size-capped (500 MB).
- **Docker**: `local` log driver with rotation (10 MB × 3) via
  `daemon.json` -- applies to newly created containers; existing ones are
  recreated to adopt it.

## Remote administration

- SSH only, key-based; password authentication disabled.
- LAN-only admin UIs are reached via SSH port forwarding when away:
  `ssh -L <port>:localhost:<port> user@host`.

## Known gaps (tracked)

- **Backups**: config/database snapshots exist for some services but there
  is no automated, tested backup routine yet -- the largest open
  operational risk, and the top of the infrastructure roadmap.
- Secrets live in a shared password manager; the manager itself has no
  independent backup.
