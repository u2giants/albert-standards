# Seafile Infrastructure — designflow.app

**Last synced from `u2giants/seafile`:** 2026-06-26

This file is the shared infrastructure reference for POP Creations' Seafile Pro deployment at `seafile.designflow.app`. The canonical implementation and detailed runbooks live in [`u2giants/seafile`](https://github.com/u2giants/seafile); this standards repo records the cross-project operating facts future infrastructure sessions must not miss.

## Deployment identity

| Item | Current value |
|---|---|
| Public URL | `https://seafile.designflow.app` |
| VPS | Linode `172.233.14.233`, hostname `seafile-br`, Ubuntu 24.04 |
| Repo | `https://github.com/u2giants/seafile` |
| Branch model | `main` only |
| Deployment model | CI publishes images to GHCR; operators manually pull/recreate on target hosts |
| Reverse proxy | Caddy via `lucaslorentz/caddy-docker-proxy:2.12-alpine` |
| Not used | Coolify, Traefik, deploy webhooks, CI SSH deploys |

Before deploying or debugging production, verify the host is really the live VPS:

```bash
hostname
ls -la /opt/seafile
docker ps --format 'table {{.Names}}	{{.Image}}	{{.Status}}' | grep -E 'seafile|caddy|nas-settings'
```

Expected containers include `seafile`, `seafile-mysql`, `seafile-redis`, `seafile-caddy`, and `nas-settings`. If they are absent, stop; a stale clone has existed on another host.

## Runtime stack

| Component | Container / host | Notes |
|---|---|---|
| Seafile Pro | `seafile` | `seafileltd/seafile-pro-mc:13.0-latest` from Docker Hub |
| MariaDB | `seafile-mysql` | Metadata only: users, libraries, shares, audit/history references |
| Redis | `seafile-redis` | Cache/session support, no persistence required |
| Caddy | `seafile-caddy` | Handles HTTP/HTTPS and Let's Encrypt; only container exposing host ports 80/443 |
| NAS Settings panel | `nas-settings` | Flask app image `ghcr.io/u2giants/seafile:nas-settings-latest` |
| NAS sync agent | `seaf-cli` on `edgesynology1` | Image `ghcr.io/u2giants/seafile:seaf-cli-latest`; one container syncs multiple NAS folders; memory target `8g` |

Core compose files are under `/opt/seafile` on the VPS. The `nas-settings` compose file is deployed separately from the repo path `/home/ai/seafile-repo/seafile-server/nas-settings.yml` and is not part of `COMPOSE_FILE`.

## Storage and data durability

Seafile file data is not stored on the VPS disk. It is stored in Linode Object Storage region `br-gru-1` using three distinct buckets:

| Seafile store | Bucket |
|---|---|
| blocks | `seafile-s3` |
| commits | `seafile-s3-commits` |
| fs metadata | `seafile-s3-fs` |

The VPS disk holds MariaDB, Caddy state, Seafile config, logs, and SQL backups. Daily root cron dumps all MariaDB databases to `/opt/backups/seafile-db-YYYYMMDD.sql` and prunes dumps older than 30 days. The SQL backups do not contain file blocks.

## DNS and TLS

Cloudflare zone: `designflow.app`.

`seafile.designflow.app` must remain a DNS-only A record pointing to `172.233.14.233`. Do not enable the Cloudflare orange-cloud proxy: Seafile desktop sync traffic is not reliable through Cloudflare's HTTP proxy layer. Caddy handles TLS with Let's Encrypt using HTTP challenge on ports 80/443.

## Authentication

Primary staff login is Microsoft Entra OAuth for POP Creations. The current tenant-locked Entra tenant is `1caeb1c0-a087-4cb9-b046-a5e22404f971`. Seafile internal SSO usernames use `@auth.local`; do not rename them casually because ownership, OAuth bindings, and profiles are separate records.

Current human admin identity:

| Human email | Internal Seafile username |
|---|---|
| `albert@popcre.com` | `4cba3f5721f7436fbe06a2b154ee296a@auth.local` |

The NAS sync machine account uses `nas-sync@popcre.com` credentials stored outside git.

## Release and rollback

GitHub Actions publish two project images:

| Image | Built from | Tags |
|---|---|---|
| `ghcr.io/u2giants/seafile:seaf-cli-latest` | `synology-seaf-cli/` | `seaf-cli-latest`, immutable `sha-<commit>` |
| `ghcr.io/u2giants/seafile:nas-settings-latest` | `seafile-server/nas-settings/` | `nas-settings-latest`, immutable `nas-settings-sha-<commit>` |

CI must not SSH into production. Deployment is manual after CI succeeds:

- VPS: pull/recreate from `/opt/seafile` and the repo compose files.
- NAS: SSH from the VPS with `ssh edge1`, use Synology Docker at `/var/packages/ContainerManager/target/usr/bin/docker`, stage the current compose file, then pull/recreate.

Rollback by pinning the affected image to a prior immutable tag and recreating. Do not rebuild production images on the VPS or NAS.

## Synology coupling

The live NAS sync agent runs on `edgesynology1` and bind-mounts these source paths into a single `seaf-cli` container:

| NAS path | Container path | Seafile library |
|---|---|---|
| `/volume1/mac/Decor/Character Licensed` | `/library/char` | Character Licensed |
| `/volume1/mac/Decor/Generic Decor` | `/library/decor` | Generic Decor |
| `/volume1/styleguides` | `/library/guides` | Styleguides |
| `/volume1/mac/Art Library` | `/library/art` | ArtLibrary |
| `/volume1/files/shared` | `/library/shared` | shared |

Only one sync host may be active for a library at a time. The Windows workstation deployment is an optional alternative: it uses CIFS `/source` mounts plus disposable Docker `/library` staging volumes, and the compose must keep staging cleanup enabled so Docker Desktop's WSL disk does not retain copied library data after containers stop.

The `seaf-cli` container is intentionally memory-limited to `8g`. The NAS
Settings panel can queue runtime memory-limit changes; the wrapper applies them
through `/var/run/docker.sock` on its next heartbeat. If the Docker socket is not
mounted, the configured value remains documented in the panel but only takes
effect on the next compose recreate. Treat the Docker socket mount as privileged
access and keep it scoped to this trusted operations container.

Synology host inotify limits are part of this system's operating environment. The current target is:

```bash
fs.inotify.max_user_watches=2097152
fs.inotify.max_user_instances=1024
```

Persist these with a DSM Task Scheduler boot-up task running as root; DSM does not reliably preserve `/etc/sysctl.conf` across updates. See [`../synology/runbooks/seafile-sync-inotify.md`](../synology/runbooks/seafile-sync-inotify.md).

## Update rule

When infrastructure, deployment, DNS, authentication, storage, backup, or NAS operating-environment decisions change in `u2giants/seafile`, update this file in the same work session. The Seafile repo should also keep links back here in `README.md`, `AGENTS.md`, and deployment/configuration docs so future AI sessions know the shared standards exist.
