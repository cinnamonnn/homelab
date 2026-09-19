# Architecture

## Host

- Dell OptiPlex 9020 (SFF) — Intel Core i5-4590S, 8 GB RAM
- Arch Linux, rolling release
- OS/apps on a 256 GB SSD (LVM); media on a 2 TB external HDD mounted via fstab
- Directly exposed public IP — no NAT/router buffer, which makes the
  firewall layer load-bearing (see [security](network-security.md))

## Entry point & TLS

Caddy (host-level) terminates TLS for all public subdomains and reverse-proxies
to containers bound on loopback. Only ports 80/443 are intentionally public.
DNS on Porkbun.

## Service model

Services run as standalone `docker compose` projects (one directory per app,
bind-mounted config/data), with a managed platform (Umbrel) being progressively
migrated away from. Each public service follows the same pattern:

| Ring     | Address        | Audience                          |
|----------|----------------|-----------------------------------|
| loopback | `127.0.0.1:PORT` | Caddy-fronted apps only           |
| LAN      | `10.21.0.1:PORT` | admin tools, filtered by firewall |
| public   | via Caddy      | household-facing apps             |

The rule: **bind every service to the smallest network ring that serves its
audience.** Nothing binds the public interface directly (with the sole exception of monitoring services, e.g. Uptime Kuma)

## Notable pipelines

- Automated music library: downloads → beets (two-pass import on a systemd
  timer: MusicBrainz match + as-is fallback) → media server library
- Monitoring: Uptime Kuma (service health), Glances/Speedtest (metrics) →
  Homarr dashboard; Tautulli for media-server activity
- Logging: journald persistence with a size cap; Docker `local` log driver
  with rotation
- Remote admin: SSH (key-only, password auth disabled), port forwarding for
  LAN-only UIs

## Hardware constraints & trade-offs

The Haswell-era CPU has no usable hardware transcode (deprecated iGPU support),
so all transcoding is CPU-only — this shapes operational decisions (client
quality caps, codec-aware sourcing, monitoring concurrent transcodes) and
motivates the planned hardware upgrade path.
