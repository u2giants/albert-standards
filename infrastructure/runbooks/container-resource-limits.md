# Runbook: Container Resource Limits

**Last updated:** 2026-05-21
**Applies to:** Docker-based services on the production Hetzner VPS. For
Designflow PLM Cloud Run services, use Cloud Run CPU/memory/concurrency/max
instance settings from each repo's `cloudbuild.yaml` and trigger substitutions;
see [`../designflow-cloud-run.md`](../designflow-cloud-run.md).

---

## Policy

Every containerized service on this server **must** have explicit memory and CPU limits. No service should run uncapped.

Designflow Cloud Run services follow the same capacity-control principle, but
the controls are Cloud Run service settings rather than Docker/Coolify limits.

---

## Why This Matters

Without resource limits, a single misbehaving job, query, or sync process can consume all available RAM on the host. When RAM is exhausted, the OS begins swapping aggressively. Heavy swap I/O degrades every other service on the machine. If the pressure is sustained long enough, or if the kernel encounters swap exhaustion, the host can crash with an unclean reboot.

This is exactly what happened on **2026-05-20**: HiClaw recursively mirrored its own storage directory into MinIO, generating extreme disk I/O and memory pressure over 10 days of uptime. With no container limits in place, nothing bounded the damage. The result was an unclean reboot with journal corruption and hours of post-reboot swap drainage.

See the full post-mortem: [2026-05-20 Hetzner Crash](../post-mortems/2026-05-20-hetzner-crash.md)

---

## Current Limits (as of 2026-05-21)

| Service | Memory Limit | Memory Reservation | CPU Limit | Where Set | Status |
|---|---|---|---|---|---|
| twenty (server + worker) | 1536 MiB | 512 MiB | 2 CPUs | Coolify UI / API | ✅ Set |
| twenty-postgres | 4096 MiB | 2048 MiB | 2 CPUs | Coolify UI / API | ✅ Set |
| twenty-redis | 256 MiB | 64 MiB | 0.5 CPUs | Coolify UI / API | ✅ Set |
| hiclaw-controller | — | — | — | — | ❌ Uncapped (~1.84 GiB observed) |
| hiclaw-manager | — | — | — | — | ❌ Uncapped (~588 MB observed) |
| minio | — | — | — | — | ❌ Uncapped (~680 MB RSS observed) |
| All other services | — | — | — | — | ❌ Audit required |

---

## How to Set Limits for Coolify-Managed Services

### Using the Coolify UI

1. Open the service in the Coolify dashboard.
2. Go to **Resources** (or equivalent tab).
3. Set memory limit, memory swap, memory reservation, and CPU limit.
4. Save and redeploy.

### Using the Coolify API

For applications:

```bash
curl -X PATCH \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "limits_cpus": "2",
    "limits_memory": "1536",
    "limits_memory_swap": "1536",
    "limits_memory_swappiness": 0,
    "limits_memory_reservation": "512"
  }' \
  http://localhost:8000/api/v1/applications/<app-uuid>
```

For databases (postgres, redis, etc.):

```bash
curl -X PATCH \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "limits_cpus": "2",
    "limits_memory": "4096",
    "limits_memory_swap": "4096",
    "limits_memory_swappiness": 0,
    "limits_memory_reservation": "2048"
  }' \
  http://localhost:8000/api/v1/databases/<db-uuid>
```

Memory values are in **MiB integers** (not strings with suffixes).

**Why this persists:** Coolify stores these limits in its own database and writes them into the compose file it generates on every deploy. Limits set this way survive redeployments. Limits added manually to the compose file on disk do not — Coolify overwrites them.

---

## How to Set Limits in Docker Compose (non-Coolify services)

Use the `deploy.resources.limits` key (Docker Compose v2+). Example:

```yaml
services:
  my-service:
    image: my-image:latest
    deploy:
      resources:
        limits:
          memory: 1536m
          cpus: "2.0"
        reservations:
          memory: 256m
          cpus: "0.25"
```

### Memory unit reference

| Suffix | Meaning |
|---|---|
| `m` | Mebibytes (e.g. `1536m`) |
| `g` | Gibibytes (e.g. `4g`) |

---

## How to Verify Limits Are Applied

After deploying or redeploying a service, verify that Docker actually picked up the limits:

```bash
# Check memory limit (in bytes). 0 = no limit.
docker inspect <container_name> --format '{{.HostConfig.Memory}}'

# Check CPU limit (in nanoseconds per second). 0 = no limit.
docker inspect <container_name> --format '{{.HostConfig.NanoCpus}}'

# Quick sanity check across all running containers
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.CPUPerc}}"
```

A container with `Memory = 0` from `docker inspect` has no memory limit and must be fixed.

---

## Coolify Caveat

Coolify generates its own Docker Compose files and **regenerates them on every redeploy**.

- **Do not** add limits by hand-editing the Coolify-generated compose file on disk (`/data/coolify/...`). They will be overwritten.
- **Do** use the Coolify UI or API (see above). Those values are stored in Coolify's database and written into every generated compose file.
- After any Coolify-triggered redeploy, re-verify limits are still in place using `docker inspect` as shown above.

---

## Warning Thresholds

Monitor `docker stats` regularly. If any container's `MemPerc` consistently exceeds **80% of its configured limit**, take action:

1. Investigate the application for memory leaks or runaway workloads.
2. If usage is legitimate and growing, raise the limit — but understand why before doing so.
3. Never silently raise limits.

Consider setting up automated alerting: host swap usage > 50% should trigger a notification.

---

## Related

- [2026-05-20 Hetzner Crash Post-Mortem](../post-mortems/2026-05-20-hetzner-crash.md)
