# Runbook: Container Resource Limits

**Last updated:** 2026-05-21
**Applies to:** All Docker-based services on the production Hetzner VPS

---

## Policy

Every containerized service on this server **must** have explicit memory and CPU limits defined in its Docker Compose service definition under `deploy.resources.limits`.

No service should run uncapped.

---

## Why This Matters

Without resource limits, a single misbehaving job, query, or sync process can consume all available RAM on the host. When RAM is exhausted, the OS begins swapping aggressively. Heavy swap I/O degrades every other service on the machine. If the pressure is sustained long enough, or if the kernel encounters swap exhaustion, the host can crash with an unclean reboot.

This is exactly what happened on **2026-05-20**: HiClaw recursively mirrored its own storage directory into MinIO, generating extreme disk I/O and memory pressure over 10 days of uptime. With no container limits in place, nothing bounded the damage. The result was an unclean reboot with journal corruption and hours of post-reboot swap drainage.

See the full post-mortem: [2026-05-20 Hetzner Crash](../post-mortems/2026-05-20-hetzner-crash.md)

---

## Current Limits (as of 2026-05-21)

| Service | Memory Limit | CPU Limit | Where Set | Status |
|---|---|---|---|---|
| twenty (server) | 1536 MiB | 2 CPUs | Docker Compose | ✅ Set |
| twenty (worker) | 1536 MiB | 2 CPUs | Docker Compose | ✅ Set |
| twenty-postgres | 4 GiB | 2 CPUs | Coolify UI | ✅ Set |
| twenty-redis | 256 MiB | 0.5 CPUs | Coolify UI | ⚠️ TODO: verify still in place |
| hiclaw-controller | — | — | — | ❌ Uncapped (~1.84 GiB observed) |
| hiclaw-manager | — | — | — | ❌ Uncapped (~588 MB observed) |
| minio | — | — | — | ❌ Uncapped (~680 MB RSS observed) |
| All other services | — | — | — | ❌ Audit required |

---

## How to Set Limits in Docker Compose

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

### Notes

- `limits` is a hard cap. The container is OOM-killed if it exceeds the memory limit.
- `reservations` is a soft hint to the scheduler — it does not impose a cap.
- CPU limits are in fractional CPUs (e.g. `"0.5"` = half a core).
- Limits under `deploy` require Docker Compose v2 (the `docker compose` plugin, not the legacy `docker-compose` binary). Confirm with `docker compose version`.

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

Coolify manages its own Docker Compose files and **may regenerate them on redeploy**, overwriting any manual edits including resource limits.

**Rules for Coolify-managed services:**

1. Set resource limits in **Coolify's UI** (Service → Resources tab), not directly in the compose file.
2. After every Coolify-triggered redeploy, re-verify that limits are still in place using `docker inspect` as shown above.
3. If Coolify does not expose a resource limit field for a given service, document this and track it manually.

---

## Warning Thresholds

Monitor `docker stats` regularly. If any container's `MemPerc` consistently exceeds **80% of its configured limit**, take action:

1. Investigate the application for memory leaks or runaway workloads.
2. If usage is legitimate and growing, raise the limit — but also raise an alert so you know headroom is shrinking.
3. Never silently raise limits without understanding why usage increased.

Consider setting up automated alerting: swap usage > 50% on the host should trigger a PagerDuty alert or equivalent.

---

## Related

- [2026-05-20 Hetzner Crash Post-Mortem](../post-mortems/2026-05-20-hetzner-crash.md)
