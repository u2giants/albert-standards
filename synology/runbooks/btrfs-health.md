# Runbook: BTRFS Health

## Quick status check

```bash
btrfs device stats /volume1
```

Fields:
- `write_io_errs` / `read_io_errs` / `flush_io_errs` — IO errors at the block device layer
- `corruption_errs` — blocks that failed checksum verification
- `generation_errs` — tree generation mismatches (rare, serious)

**Important:** All counters are cumulative from filesystem mount. They never reset, even after a scrub repairs the blocks. A non-zero `corruption_errs` after a clean scrub is normal — the errors were repaired.

## Scrub status

```bash
btrfs scrub status /volume1
```

A clean result says `Error summary: no errors found`. This is the authoritative health indicator — more reliable than device stats.

## Interpreting results

| Scenario | Meaning |
|---|---|
| corruption_errs > 0, last scrub clean | Old errors, already repaired. Not a current concern. |
| corruption_errs > 0, no recent scrub | Run a scrub to find and repair |
| Scrub reports errors | Real data at risk — investigate immediately |
| generation_errs > 0 | Serious — escalate |

## Known baseline

- edgesynology1: `corruption_errs 0` — clean
- edgesynology2: `corruption_errs 25` — old accumulated errors, repaired by scrub on 2026-05-14

## Running a scrub via DSM

Storage Manager → Storage Pool → select pool → Action → Data Scrubbing.
Scrubs run in the background and take hours to days on large volumes (edgesynology2 has ~47TB).
Do NOT trigger via `btrfs scrub start` through the MCP — this can interfere with DSM's scrub scheduler.
