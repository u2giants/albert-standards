# Drive Inventory

## edgesynology1

Drive models to be documented on next session (run `smartctl -i /dev/sdX` for each drive).

## edgesynology2

Two drives of note (elevated write latencies observed, long SMART tests initiated 2026-05-xx):

| Slot | Device | Model | Notes |
|---|---|---|---|
| ? | sdd | WUH721818ALE6L4 | 18TB, elevated write latency observed |
| ? | sde | WUH721818ALE604 | 18TB, elevated write latency observed |

## RAID layout (edgesynology2)

- md0: RAID1 across sda1, sdb1, sdc1, sdd1, sde1 (5 drives — OS/boot partition)
- md1: RAID1 across sda2, sdb2, sdc2, sdd2, sde2 (5 drives — swap)
- Data volume: SHR across remaining partitions

**Note:** md0/md1 kernel reporting shows "5 out of 6 mirrors" — this is a DSM artifact for a 5-drive unit, not a degraded array. All 5 slots show `in_sync`.
