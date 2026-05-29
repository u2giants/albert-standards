# Synology NAS Knowledgebase

Documentation, troubleshooting history, and operational guides for the two Synology NAS units.

## Units

| Hostname | IP | Model | Notes |
|---|---|---|---|
| edgesynology1 | 192.168.3.100 | DS1621xs+ | Primary unit |
| edgesynology2 | 192.168.3.101 (port 1904) | DS1621xs+ | Secondary unit |

## Structure

| Path | Purpose |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | **Start here every session** — hardware reference, NIC map, known quirks, MCP usage rules |
| [`issue-history.md`](issue-history.md) | Chronological log of every issue ever encountered. Scan by date or keyword — do NOT ingest the whole file |
| [`agents.md`](agents.md) | Instructions for AI agents: how to start a session, diagnose, and update these docs |
| [`hardware/`](hardware/) | Static hardware facts: NIC chips, drive models, bond config |
| [`runbooks/`](runbooks/) | Step-by-step fix guides for known failure modes |
| [`post-mortems/`](post-mortems/) | Root cause analyses for resolved incidents |

## Quick index — what to read for common problems

| Problem type | Read first |
|---|---|
| Network / NIC / bond / link failure | `CLAUDE.md` → NIC section, then `runbooks/network-bond-troubleshooting.md` |
| ShareSync bio errors | `runbooks/sharesync-bio-errors.md` |
| BTRFS corruption / scrub | `runbooks/btrfs-health.md` |
| MCP tool timeouts / crashes | `CLAUDE.md` → MCP section |
| Drive health / SMART | `runbooks/drive-health.md` |
| Any issue — check history first | `issue-history.md` (scan only) |
