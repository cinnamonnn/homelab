# Troubleshooting Log

Every issue below was hit, diagnosed, and fixed on this system in production
(no staging environment). Each entry records the symptom as it appeared, the
root cause, the fix, and — where it generalized — the lesson that now shapes
routine procedure. Two patterns recur across all of them and are worth
stating up front:

> **Containers don't share your identity or your network position.**
> UID mismatches and network-position confusion (Docker bridge subnets vs.
> the host's interfaces; "localhost" inside a container is the container's
> own loopback) cause most bugs on this box. When something "can't see" or
> "can't write," check who the container is and where it's standing first.
>
> **Verify each layer with its own cheap command** — `findmnt`, `docker port`,
> `curl`, `caddy validate` — before blaming the layer above it.

## Configuration

### 1. YAML breaks despite being "copied perfectly"
A single space of misalignment in a pasted block makes the parser error point
at the wrong line, far from the fault. Fix: edit values in place rather than
pasting blocks, and validate before applying:
`python3 -c "import yaml; yaml.safe_load(open('file'))"`.

## Networking

### 2. Container can't reach a host-published port / host LAN IP times out
Docker blocks traffic between separate bridge networks by default (each
compose project gets its own). Fix: use *that container's own* gateway IP —
`docker inspect <name> --format '{{range .NetworkSettings.Networks}}{{.Gateway}}{{end}}'`
— or the public hostname.

### 3. Port binding declared but never activated
`HostConfig.PortBindings` showed a published port, but `docker port` (which
reads live state) was empty — nothing listened, and the reverse proxy 502'd.
Fix: `docker compose down && up -d` to force Docker to re-wire the network
namespace. **`docker port` is now a standard post-deploy check.**

### 4. Cross-origin API behind proxy auth returns 401/CORS errors
Browser fetches to a different subdomain don't carry cached basic-auth
credentials, and failed requests surfaced as CORS errors. Fix: authenticate
at the UI layer, and restrict the API endpoint by `Origin` header at the
proxy. Lesson: a proxy-level auth scheme can be structurally incompatible
with a cross-origin frontend that doesn't send credentials.

## Permissions & storage

### 5. Docker auto-creates missing bind-mount directories as root
Non-root containers then can't write to them. Fix: `mkdir` + `chown 1000:1000`
**before** the first `up -d` — now step zero of every deployment.

### 6. Bind mounts must use mount points, never `/dev/` paths
`/dev/sdb1/...` is a raw device, not a directory. `findmnt /dev/sdb1` gives
the real mount point.

### 7. A container's data "vanished" after storage maintenance
A volume-management operation unmounted a filesystem; Docker restarted
underneath and auto-created empty directories on the wrong volume, and every
app appeared to have lost its configuration. Nothing was deleted. Fixes now
ritualized: `findmnt <mount>` after any Docker restart that follows storage
work; remember that a directory existing ≠ a volume being mounted; and check
timestamps on "missing" data before assuming loss.

### 8. SQLite `SQLITE_CANTOPEN` crash-loop on a verified-writable mount
Mount, ownership, and environment all checked out — the real cause was a
hand-written compose file omitting two environment variables from the
project's stock compose (`DB_PATH`, `BACKUP_DIR`) whose defaults resolve to
a relative path inside the image. Third specimen of the same bug class
(a frontend's build-time API URL, an internal port, this): **when writing a
compose from a stock one, diff the environment section line by line —
innocuous-looking defaults are load-bearing.**

## Platform quirks

### 9. Managed-platform (Umbrel) container oddities
Prefixed container names (`sonarr_server_1`), per-app bridge networks, and
apps dying on middleware quirks. Fix pattern: `docker ps` before assuming
names; standalone compose fixed a health-monitoring app outright. This drove
the platform → standalone compose migration.

### 10. `ufw-docker allow` "could not find running instance"
The target container wasn't running. `docker ps -a` + `docker logs` —
usually permissions or placeholder volume paths.

### 11. Installer shipped an empty package-manager mirrorlist
Fixed by fetching a fresh regional mirrorlist, uncommenting servers, and
forcing a cache refresh. Lesson: validate installer defaults.

## Media pipeline

### 12. Importer reports "no files imported"
An app-level download-directory setting was overriding the volume mapping —
files were landing somewhere the importer never looked. Fix: align the
app-level setting with the container-internal mount path.

### 13. Imports with empty album-artist metadata collapse the library layout
Missing metadata collapsed the path template, leaving albums flat in the
library root where scans found zero artists. Music libraries, unlike
video, need Artist/Album/Track hierarchy. Fix with metadata modification +
a library move; check the album-artist field first when music "doesn't show up."

### 14. Media server "sees the folder but no files"
Three distinct root causes found: file permissions (container user couldn't
read host-written files), hierarchy problems (#13), and the folder simply
not being in the container's volume mounts. `docker exec <name> ls /path`
settles which one in seconds.

### 15. Audio/subtitle track selection depends on language flags
`und` (undefined) tracks break "preferred language" selection rules.
`ffprobe`/MediaInfo on the problem files confirms.

## Hardware constraints

### 16. CPU-only transcoding on pre-Skylake Intel
Hardware transcode support is deprecated on this generation of iGPU, so all
transcoding is software. This shapes operational decisions: client quality
caps, codec-aware media sourcing, and monitoring concurrent transcodes
(a single HEVC→H.264 HDR→SDR transcode is a significant fraction of this
CPU's capacity).

## Known-open

### 17. Third-party API refusing sessions (cobalt / YouTube)
Self-hosted instance works for all providers except YouTube, which rejects
the server's sessions (PO-token/visitor-data — an active upstream arms race;
fix exists as an unmerged PR upstream). Non-YouTube providers fully working.
Documented and pinned rather than patched: building from an unmerged branch
isn't worth the maintenance for this use case.
