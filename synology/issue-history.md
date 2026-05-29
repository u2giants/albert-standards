# Issue History — Synology NAS

**Newest entries at top. Scan by keyword or date — do NOT ingest the full file.**

---

## 2026-05-14 — Aquantia eth0 identified as root cause; bond migration to eth1+eth2

**Units affected:** both
**Status:** In progress (bond reconfiguration underway)

### Symptom
edgesynology2 eth1 showing NO-CARRIER. edgesynology1 eth0 had accumulated link failures. ShareSync bio errors on both units correlating with network instability.

### Root cause
The DS1621xs+ has three NICs: one Aquantia AQC107 (eth0, 10GbE, `atlantic` driver) and two Intel i210 (eth1/eth2, 1GbE, `igb` driver). The `atlantic` driver on DSM spontaneously drops link on eth0 on both units. dmesg confirmed repeated `atlantic: link change old 1000 new 0` cycles. Four brand-new cables ruled out physical cable failure. The Aquantia chip/driver combination is inherently unstable on this DSM version.

### Fix applied
Decision made to remove eth0 from the bond on both units and rebuild the bond using eth1+eth2 (both Intel i210 — rock solid). Configuration done via DSM Control Panel → Network → Bond edit.

### Outcome
Reconfiguration in progress at time of writing. Bond on both units should be eth1+eth2 when complete.

### Related
- `runbooks/network-bond-troubleshooting.md`
- `hardware/nic-reference.md`

---

## 2026-05-14 — edgesynology2 eth2 acquired unexpected DHCP IP (192.168.0.115)

**Units affected:** edgesynology2
**Status:** Resolved

### Symptom
`ip addr show eth2` on edgesynology2 showed `192.168.0.115/22` on a standalone (non-bonded) interface.

### Root cause
eth2 was set to DHCP in DSM and had been uncabled. When a cable was plugged into it during bond investigation work, it acquired a DHCP lease.

### Fix applied
eth2 added to the bond — DSM automatically stripped the DHCP IP when it became a bond slave.

### Outcome
Resolved. eth2 is now a bond slave with no standalone IP.

---

## 2026-05-14 — edgesynology1 eth0 link failure / ShareSync bio errors

**Units affected:** edgesynology1 (primary), edgesynology2 (downstream bio errors)
**Status:** Root cause identified (see Aquantia entry above)

### Symptom
`/proc/net/bonding/bond0` on edgesynology1 showed eth0 Link Failure Count > 0. ShareSync `bio error` entries in `/volume1/@synologydrive/log/syncfolder.log` on edgesynology2, clustered around network instability events. Errors had error codes -1, -2, -3.

### Root cause
Aquantia eth0 `atlantic` driver dropping link. Initially suspected bad cable (cable was swapped), but link failure count incremented during the swap and instability continued — confirming driver/chip issue, not cable.

### Fix applied
Cable swap on edgesynology1 eth0 (temporary — real fix is bond migration off eth0).
Monitoring ShareSync logs on edgesynology2 after cable swap.

### Outcome
Bio errors reduced after cable swap but not eliminated. Full resolution pending bond migration to eth1+eth2.

---

## 2026-05-14 — edgesynology2 BTRFS 25 corruption_errs

**Units affected:** edgesynology2
**Status:** Resolved (not a current concern)

### Symptom
`btrfs device stats /volume1` showed `corruption_errs 25` on edgesynology2.

### Root cause
Old accumulated errors. BTRFS corruption_errs counter is cumulative and never resets, even after repairs.

### Fix applied
None needed. A scrub completed ~12 hours prior with a clean result (no errors found). The 25 errors were already repaired by the scrub.

### Outcome
Not a current concern. Scrub result is the authoritative health indicator, not the cumulative counter.

---

## 2026-05-14 — /volume1/mac permissions (0000) causing ShareSync SynoEAStream errors

**Units affected:** edgesynology1
**Status:** Resolved

### Symptom
ShareSync logs showing `mac@SynoEAStream` errors. edgesynology1 `/volume1/mac` was inaccessible.

### Root cause
`/volume1/mac` on edgesynology1 had permissions `0000` and wrong ownership, preventing ShareSync from reading extended attributes.

### Fix applied
Permissions changed to `0777`, owner set to `1024:users` (matching edgesynology2).

### Outcome
`mac@SynoEAStream` errors resolved.
