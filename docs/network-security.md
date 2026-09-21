# Network & Security

This server sits directly on a public IP -- no NAT, no consumer-router
firewall in front of it. That constraint shaped the entire security design:
**the host's firewall is the only boundary, so it's treated as load-bearing
infrastructure** rather than a defaults-and-forget layer.

## Threat model

- The host is internet-scannable, so port hygiene matters more than in a
  NAT'd setup: anything accidentally published is immediately reachable.
- Services split into three audiences: household users (from anywhere),
  admin tools (LAN/VPN only), and one P2P application (intentionally
  exposed on a single port).
- Untrusted internet traffic should only ever reach the TLS proxy.

## Layer 1: firewall (ufw + ufw-docker)

A plain Docker installation punches holes in ufw via its own iptables
rules -- published container ports are reachable even when ufw "denies"
them. That's the classic footgun for a Docker host on a public IP.

**ufw-docker** closes that gap: it manages per-container rules so Docker's
port publishing and the firewall agree with each other. Rules are
re-generated whenever containers change:

    sudo ufw-docker allow <container> <port>/tcp && sudo ufw-docker update

This step is part of the [update ritual](operations.md) -- a new container
without refreshed rules is either unreachable (rules not added) or, worse,
over-exposed (Docker's own rule bypassed the policy).

## Layer 2: bind-to-the-smallest-ring

Every service is bound to the smallest network audience it serves:

| Ring | Binding | Who reaches it |
|---|---|---|
| loopback | `127.0.0.1:PORT` | the proxy only -- all public apps |
| LAN | `10.21.0.1:PORT` | admin/monitoring tools, filtered |
| public | via proxy | nothing binds the public interface directly |

An accidentally published port is a *second* line of defense, not the only
one: services aren't listening on the public interface to begin with.

## Layer 3: TLS proxy (Caddy)

Caddy is the sole public entry point: automatic certificate provisioning
and renewal per subdomain, reverse-proxying to loopback-bound containers.
Public exposure is a deliberate, per-service decision expressed as
"add a proxy block," not an accident of port publishing.

Where an app has no built-in authentication, proxy-level basic auth gates
it (bcrypt-hashed credentials -- validated config before every reload,
because a plaintext password in the config fails reload with a confusing
error). One service uses an `Origin`-header match at the proxy to restrict
a cross-origin API to the app's own frontend (see the troubleshooting log
for why proxy auth and cross-origin APIs don't mix).

## Layer 4: host access

- SSH: key-based only, password authentication disabled (verified).
- IPv6: exposure audited; AAAA records removed so the stack is
  single-stack and the audit surface stays small.
- LAN-only admin UIs reachable remotely only via SSH port forwarding.

## Layer 5: verification

Posture is verified empirically, not assumed -- a periodic external port
scan from an outside network (phone hotspot) confirms that only the
intended ports answer and everything else is filtered. Both IPv4 and IPv6
were audited when IPv6 was still in play.

## Residual risks (known & accepted)

- A determined non-browser client could probe intentionally public ports;
  mitigation is rate-limiting at the app layer plus the small attack
  surface (two applications + the proxy).
- Single host: no segmentation between services beyond Docker's bridge
  isolation -- acceptable at household scale.
- See [operations](operations.md) for the open backup gap.
