# Post-Mortem: Hetzner VPS Crash — 2026-05-20

**Date:** 2026-05-20
**Severity:** High (unclean reboot, journal corruption, extended swap pressure)
**Status:** Partially resolved; HiClaw action items outstanding

---

## Summary

At approximately 20:32 on 2026-05-20, the production Hetzner VPS suffered an unclean reboot after 10 days of uptime (previous boot: 2026-05-10 13:51). The root cause was HiClaw/OpenClaw recursively mirroring its own storage directory into MinIO, generating extreme disk I/O and/or memory pressure that destabilized the host. A contributing factor was the `EmailRerouterCronJob` in Twenty CRM, which fired on container startup and issued 50,000+ Postgres queries in the first few minutes, amplifying post-reboot stress.

---

## Server Profile

| Property | Value |
|---|---|
| Provider | Hetzner VPS |
| OS | Ubuntu 24.04 |
| Runtime | Docker-based |
| RAM | 15 GiB |
| CPUs | 8 |
| Disk | 226 GiB |

---

## Timeline

| Time | Event |
|---|---|
| 2026-05-10 13:51 | Server last cleanly booted |
| 2026-05-20 ~20:32 | Unclean reboot; journal corruption observed |
| 2026-05-20 (post-reboot) | 3 GiB of 4 GiB swap in use; active swap I/O still draining hours later |
| 2026-05-20–21 | Mitigations applied (see below) |

---

## Post-Reboot State

- **Swap:** 3 GiB of 4 GiB in use; swap I/O actively draining for hours after reboot
- **Disk:** 68% used — not full; disk pressure was from I/O throughput, not capacity
- **OOM killer:** No evidence in available journal fragments (journal was corrupted)
- **Journal:** Partially corrupted due to unclean shutdown; pre-crash logs unavailable

---

## Root Cause

**HiClaw/OpenClaw recursive MinIO mirror**

HiClaw (or its OpenClaw component) recursively mirrored its own storage directory into MinIO, creating a massive nested directory tree of the form:

```
hiclaw-storage/manager/hiclaw/hiclaw-storage/manager/hiclaw/hiclaw-storage/...
```

This recursive self-mirroring likely generated extreme disk I/O (potentially filling the OS page cache and thrashing swap) and/or sustained memory pressure over the 10-day uptime period, ultimately destabilizing the host kernel enough to trigger an unclean reboot.

No container resource limits were set on any service, meaning no container could be bounded when misbehaving.

---

## Contributing Factor

**`EmailRerouterCronJob` query storm on startup**

The Twenty CRM `EmailRerouterCronJob` fires on container startup. With no in-memory caching of ignore rules or customer company lists, it executed a full table scan against `_emailMessage` for each batch page. With 9,833 emails to process, this generated approximately 50,000+ Postgres queries in the first few minutes after the reboot.

This did not cause the crash — it amplified post-reboot system stress at a time when the server was already under memory and I/O pressure from swap draining.

---

## Fixes Applied (2026-05-20/21)

### Twenty CRM

| Fix | Detail |
|---|---|
| Partial index on `_emailMessage` | Index on `(createdAt, id)` WHERE `routingStatus` is reroutable; eliminates full table scans in the cron job |
| Postgres `shared_buffers` | Raised to 3 GiB |
| Postgres `work_mem` | Raised to 32 MB |
| Postgres `maintenance_work_mem` | Raised to 512 MB |
| `pg_stat_statements` | Enabled for query performance visibility |
| Slow query logging | Enabled for queries ≥ 1 s |
| `statement_timeout` | Set to 300 s on the `twenty` role |
| `idle_in_transaction_session_timeout` | Set to 60 s on the `twenty` role |
| `EmailRouterService` caching | Ignore rules and customer company list now cached with 5-minute TTL |
| Container resource limits | Twenty server: 1536 MiB / 2 CPUs; Twenty worker: 1536 MiB / 2 CPUs |

### HiClaw

- **Outstanding** — see action items below.

---

## Outstanding Action Items

| Owner | Item | Priority |
|---|---|---|
| HiClaw developer | Investigate and fix recursive MinIO storage mirror | Critical |
| HiClaw developer | Add container resource limits to hiclaw-controller (currently ~1.84 GiB, uncapped) | High |
| HiClaw developer | Add container resource limits to hiclaw-manager (currently ~588 MB, uncapped) | High |
| Infrastructure | Audit and add resource limits to all remaining uncapped services (MinIO, etc.) | High |
| Infrastructure | Add memory pressure alerting: swap > 50% → PagerDuty or equivalent alert | Medium |

---

## Lessons Learned

1. **No container resource limits = no blast radius control.** A single misbehaving service can consume all host RAM, drive the entire system into swap, and crash the kernel. Every service must have explicit limits.
2. **Recursive sync bugs can be catastrophic at scale.** A mirroring job that mirrors its own output can grow exponentially. Input/output paths must be validated to be disjoint.
3. **Startup cron jobs need caching.** Jobs that fire immediately on container start and hit the database without caching can create severe query storms during the most vulnerable window (post-crash reboot).
4. **Journal corruption makes forensics hard.** Enabling persistent journaling and/or shipping logs off-host before a crash would improve root cause analysis.

---

## References

- Runbook: [Container Resource Limits](../runbooks/container-resource-limits.md)
