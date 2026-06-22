# Synology NAS — Session Reference

**Read this file at the start of every session involving these NASes.**
Then consult `issue-history.md` (scan only — do not ingest the whole file) to check if the current problem has been seen before.

---

## Units

| Hostname | IP | Model | Uptime pattern |
|---|---|---|---|
| edgesynology1 | 192.168.3.100 | DS1621xs+ | Stable |
| edgesynology2 | 192.168.3.101 port 1904 | DS1621xs+ | Rebooted more frequently |

Both NASes are joined to Active Directory. Tailscale is installed on both for remote access.

---

## PopDAM workload on the NAS

`edgesynology2` runs the PopDAM bridge agent container (`popdam-bridge`) from
`ghcr.io/u2giants/popdam-bridge:stable`.

What it does:
- scans the NAS for PopDAM and PopSG source files;
- generates or defers thumbnails/renders;
- uploads thumbnails to DigitalOcean Spaces;
- polls Supabase `agent-api` outward (the cloud does not connect inward to the NAS);
- verifies Seafile-sourced Helper check-ins after files land on the Synology.

Operational notes:
- The PopDAM source repo is `u2giants/popdam3`; its reference compose file is
  `deploy/synology/docker-compose.yml`.
- Bridge agent self-update is normally triggered from the PopDAM admin UI and
  pulls `ghcr.io/u2giants/popdam-bridge:stable`.
- PopDAM file identity uses a sampled `quick_hash`, not a content-unique hash.
  Do not treat it as globally unique in NAS investigations.
- Seafile/SeaDrive is a transport layer for remote designers, but Synology
  receipt is still verified by the bridge agent for Seafile check-ins.
- If a NAS decision changes bond/NIC/volume/container operating assumptions,
  update this knowledgebase and the PopDAM docs together.

---

## NIC map (identical on both units — DS1621xs+)

| Physical port | Interface | Chip | Driver | Speed | Stability |
|---|---|---|---|---|---|
| 10GbE port (rear) | eth0 | Aquantia AQC107 (0x1d6a:0x07b1) | `atlantic` | 10GbE | ⚠️ UNRELIABLE — drops link spontaneously |
| 1GbE port 1 (rear) | eth1 | Intel i210 (0x8086:0x1533) | `igb` | 1GbE | ✅ Reliable |
| 1GbE port 2 (rear) | eth2 | Intel i210 (0x8086:0x1533) | `igb` | 1GbE | ✅ Reliable |

### Bond configuration (target state — both units)

The bond should use **eth1 + eth2 only**. The Aquantia eth0 port must NOT be in the bond.
- eth0: excluded from bond, no IP, cable unplugged or unused
- eth1: bond slave
- eth2: bond slave
- bond0: carries the NAS IP (192.168.3.100 / .101)

**Why eth0 is excluded:** The Aquantia AQC107 `atlantic` driver on DSM spontaneously drops link. This has been observed on both units causing ShareSync `bio error` entries and network instability. Intel i210 ports are rock solid.

### Current bond state (last verified 2026-05-14)

| Unit | eth0 | eth1 | eth2 | Bond |
|---|---|---|---|---|
| edgesynology1 | Excluded (Aquantia, unreliable) | In bond ✅ | In bond ✅ | Healthy |
| edgesynology2 | Excluded (Aquantia, unreliable) | In bond ✅ | In bond ✅ | Healthy |

---

## Key file paths

| Purpose | Path |
|---|---|
| ShareSync logs | `/volume1/@synologydrive/log/syncfolder.log` |
| ShareSync log (rotated) | `/volume1/@synologydrive/log/syncfolder.log_0` |
| Bond status | `/proc/net/bonding/bond0` |
| BTRFS device stats | `btrfs device stats /volume1` |
| DSM package list | `/var/packages/` |

---

## Network topology

- Both NASes are on different floors of the building
- Each NAS has its two bond ports plugged into the **same switch** (one switch per floor)
- The two switches are on different floors — no direct NAS-to-NAS switch connection

---

## MCP usage rules (synology-monitor)

**Reliable tools:**
- `run_command` with `target: edgesynology1 / edgesynology2 / both` — always use this

**Never use (hang/freeze):**
- `check_system_info`
- `get_resource_snapshot`
- `disk_latency_test`
- `start_smart_test` (deadlocks the MCP server)

**Compound commands cause hangs.** Never use `&&` or `;` to chain commands. One command per call.

**Session degradation:** MCP sessions degrade after ~10–15 tool calls. Symptoms: timeouts, tool-not-found errors. Fix: full Claude Desktop restart (system tray → Quit, not just closing the window).

**Write commands are blocked** by the NAS API validator. `run_privileged_command` is whitelisted only for `synopkg` and `docker` commands.

---

## Known DSM quirks

- **DSM bond UI hides slave failures.** DSM shows bond as "Connected" as long as one port is up. Check `/proc/net/bonding/bond0` directly — never trust the DSM UI for individual slave status.
- **BTRFS corruption_errs are cumulative.** They never reset, even after a scrub repairs the blocks. A non-zero count after a clean scrub is normal and not a current concern.
- **md0/md1 show "5 out of 6 mirrors"** on edgesynology2 — this is a kernel reporting artifact for a 5-drive unit. Not a real degraded array.
- **eth2 may acquire a DHCP IP** if a cable is plugged into it while it's not a bond slave. This is harmless but should be cleared when adding eth2 to the bond.
- **Tailscale on DSM 7** intentionally disables TUN kernel mode and runs in userspace. This causes higher CPU overhead. DSM Task Scheduler is the correct persistence mechanism for Tailscale.

---

## BTRFS health baseline

| Unit | corruption_errs | Last scrub | Status |
|---|---|---|---|
| edgesynology1 | 0 | Unknown | ✅ Clean |
| edgesynology2 | 25 | 2026-05-14 (clean result) | ✅ Old accumulated errors, repaired |

---

## ShareSync bio error baseline

bio errors in `/volume1/@synologydrive/log/syncfolder.log` correlate directly with network instability (bond link drops). Once the bond is stable on eth1+eth2, these should stop.

Last bio errors observed:
- edgesynology1: 2026-05-14 (during eth0 cable swap — expected)
- edgesynology2: 2026-05-17 (during eth0/eth1 cable investigation)

---

## See also

- `issue-history.md` — full chronological issue log (scan, don't ingest)
- `runbooks/network-bond-troubleshooting.md` — bond/NIC diagnosis steps
- `runbooks/sharesync-bio-errors.md` — how to diagnose and trace bio errors
- `runbooks/btrfs-health.md` — scrub, check, and interpret BTRFS stats
- `hardware/nic-reference.md` — full NIC chip details
- `agents.md` — how to update these docs after a session
