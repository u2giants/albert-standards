# Runbook: Seafile NAS Sync Inotify Health

**Last updated:** 2026-08-03
**Applies to:** `edgesynology1` running POP Creations Seafile NAS sync

## When to use

Use this runbook when Seafile NAS sync looks stale, a library reports `synchronized` despite missing recent NAS changes, daemon logs contain watch errors, or `@eaDir` / Synology metadata paths appear in Seafile.

The canonical project docs are in [`u2giants/seafile`](https://github.com/u2giants/seafile), especially `AGENTS.md`, `docs/deployment.md`, `docs/configuration.md`, and `synology-seaf-cli/README.md`.

## Current live model

One Docker container named `seaf-cli` runs on `edgesynology1`. It syncs multiple live NAS folders directly:

| NAS path | Container path | Library |
|---|---|---|
| `/volume1/mac/Decor/Character Licensed` | `/library/char` | Character Licensed |
| `/volume1/mac/Decor/Generic Decor` | `/library/decor` | Generic Decor |
| `/volume1/mac/Art Library` | `/library/art` | ArtLibrary |
| `/volume1/files/shared` | `/library/shared` | shared |
| `/volume1/styleguides` | `/library/guides` | Styleguides |

Deploy/recreate work should be done over SSH from the Seafile VPS with `ssh edge1`, not through the NAS MCP. Docker on Synology is at:

```bash
/var/packages/ContainerManager/target/usr/bin/docker
```

The container's memory target is `8g`. The NAS Settings panel can queue a
runtime memory-limit update and the wrapper applies it through the mounted
`/var/run/docker.sock` on the next heartbeat. If the dashboard reports that the
Docker socket is unavailable, update the compose file and recreate the container
instead of assuming the runtime command took effect.

## Root cause pattern: false synchronized state

On 2026-06-19, Generic Decor drift was traced to host inotify exhaustion. `seaf-daemon` logged `fail to add watch` / `No space left on device` while the host limit was still the Synology default `fs.inotify.max_user_watches=8192`. When directories are not watched, local edits there may not trigger events, so `seaf-cli status` can report `synchronized` because the client sees no new changes.

Restarting `seaf-daemon` is not the repair. It can trigger a one-time scan and temporarily hide the problem while leaving the host watch limit too low.

## Diagnosis

Check the host limits:

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
```

Check the running container:

```bash
DOCKER=/var/packages/ContainerManager/target/usr/bin/docker
sudo -n $DOCKER ps --format 'table {{.Names}}	{{.Image}}	{{.Status}}'
sudo -n $DOCKER logs --tail 200 seaf-cli
```

Watch-limit symptoms include:

```text
fail to add watch
No space left on device
```

The Seafile wrapper also reports inotify headroom and daemon watch errors to the NAS Settings dashboard. A red/anomaly dashboard state should be treated as evidence to inspect host limits and logs before restarting anything.

## Fix

Raise the Synology host limits. Current target for the live Seafile workload:

```bash
sysctl -w fs.inotify.max_user_watches=2097152
sysctl -w fs.inotify.max_user_instances=1024
```

Make the change persistent in DSM Task Scheduler:

1. Open DSM Task Scheduler.
2. Create a triggered task that runs at boot-up.
3. Run as `root`.
4. Command:

```bash
sysctl -w fs.inotify.max_user_watches=2097152
sysctl -w fs.inotify.max_user_instances=1024
```

DSM does not reliably preserve `/etc/sysctl.conf` across updates, so Task Scheduler is the operating standard.

After applying the boot task, reboot or run the task manually, then verify the values with `sysctl`.

## Ignore-file rules

The ignore file must be named exactly:

```text
seafile-ignore.txt
```

It must live at each synced library root. `.seafile-ignore` is not recognized by Seafile.

Current expected contents:

```text
@eaDir
#recycle
#snapshot
@tmp
.DS_Store
Thumbs.db
*.tmp
```

Important caveats:

- Synology continuously regenerates local `@eaDir` thumbnail/metadata folders. Seeing them reappear on the NAS is expected.
- Ignore rules prevent future uploads or changes for ignored paths. They do not delete server-side `@eaDir` trees that already reached Seafile.
- Ignore rules are cleanup/hygiene. They may not reduce inotify watch usage because the monitor may still watch ignored directories.

## Recovery sequence

1. Verify host inotify limits are raised and persistent.
2. Verify each library root has `seafile-ignore.txt`, not `.seafile-ignore`.
3. Deploy the current `ghcr.io/u2giants/seafile:seaf-cli-latest` image after CI passes in `u2giants/seafile`.
4. Recreate the single `seaf-cli` container from the current repo compose file.
5. Confirm the container is running with the expected memory target (`8g` unless an operator intentionally changed it).
6. Confirm no new watch-limit errors appear in logs.
7. Use the NAS Settings dashboard `verify_now` / canary controls to verify each library.
8. Delete existing server-side `@eaDir` trees only after a human confirms they are junk.

## What not to do

- Do not treat a daemon restart as the fix.
- Do not delete `seaf-cli-data` for watch-limit drift; it discards client state and does not fix the host limit.
- Do not deploy stale `/tmp/seaf-cli-compose.yml` files without checking services; old two-service compose files have existed. The current service list should be only `seaf-cli`.
- Do not run two sync hosts for the same library. The Windows workstation deployment is an alternative, not an additional active syncer.

## Incident: 2026-08-03 Generic Decor PSD missing in Seafile

### Impact

A 237,388,126-byte PSD written at 13:00 EDT under
`/_New structure/AAW_/24x24/AAW2FMSREG01` was present on `edgesynology1`
and inside `/library/decor`, but the corresponding Seafile server directory was
empty. Brazil users could not see the file for more than two hours.

### Confirmed observations

- `seaf-cli status` reported Generic Decor as `synchronized` while the server API
  returned HTTP 404 for the file.
- `seaf-daemon` had exited at 15:18 UTC and the container had restarted twice.
- The live container memory limit and live Compose file had drifted to `2g`; the
  documented target was `8g`.
- The live host limit was `fs.inotify.max_user_watches=1048576`, below the
  current operating target of `2097152`.
- The PSD was visible through the container bind mount and its size matched the
  source NAS file. No data loss occurred.

These observations show a missed or stale client change index during resource
and daemon instability. They do not prove that either the memory limit or watch
ceiling alone caused this specific missed event.

### Recovery

1. Raised live inotify limits to `2097152` watches and `1024` instances.
2. Added the same values to `/etc/sysctl.conf`, with a dated backup. A DSM
   Task Scheduler boot task remains the preferred persistence mechanism because
   DSM updates may replace `/etc/sysctl.conf`.
3. Raised the running container memory limit to `8g` and restored the canonical
   Compose file at `/volume1/docker/seaf-cli/docker-compose.yml`.
4. Restarted `seaf-cli` cleanly and touched the unchanged source PSD to emit a
   fresh filesystem event.
5. While the large client reconciliation was still running, uploaded the same
   source file through Seafile's authenticated upload API. The server then
   reported the exact expected name and size.
6. Verified `seaf-cli` healthy with zero post-restart resource/watch errors and
   confirmed Generic Decor remained internally shared read-only to all logged-in
   users.

### Temporary maintenance containers

Diagnostic host changes may use commands such as `docker run --rm` with the
current `seaf-cli` image. When `--name` is omitted, Docker assigns a random name,
for example `silly_tu`. These containers are temporary operator activity, not a
new Seafile service, and `--rm` removes them when the command exits. Confirm with
`docker ps -a --filter name=<name>` before investigating further. For clearer
audit trails, future maintenance commands should set an explicit name such as
`--name seafile-maintenance-<purpose>` when Docker permits it.
