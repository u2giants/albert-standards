# AI Agent Entry Point — Synology NAS Systems

**This is the single file to give any AI agent at the start of a Synology session.**
It is AI-agnostic — written for Claude, GPT, Gemini, or any other model.

---

## What you are working on

Two identical Synology DS1621xs+ NAS units used for business file storage and sync at POP Creations.

| Hostname | IP | Role |
|---|---|---|
| edgesynology1 | 192.168.3.100 | Primary |
| edgesynology2 | 192.168.3.101 (port 1904) | Secondary |

They are connected to each other via Synology Drive ShareSync.
Both are joined to Active Directory. Both have Tailscale installed for remote access.
Both are accessible via a custom MCP server (`synology-monitor`) that exposes a `run_command` tool.

---

## Step 1 — Read the hardware and system reference

Fetch and read this file in full before doing anything else:

```
https://raw.githubusercontent.com/u2giants/albert-standards/main/synology/CLAUDE.md
```

It contains:
- NIC chip map and known hardware defects
- Current bond configuration (which ports are in the bond and why)
- Key file paths on the NASes
- MCP tool rules (which tools work, which ones hang or crash)
- Known DSM quirks that will waste your time if you don't know them
- Current health baselines (BTRFS, SMART, ShareSync)

Do not skip this. It will prevent you from repeating mistakes that have already been made.

---

## Step 2 — Check if this problem has been seen before

Fetch but DO NOT fully ingest this file — it will grow very large over time:

```
https://raw.githubusercontent.com/u2giants/albert-standards/main/synology/issue-history.md
```

Search it by keyword (error message, component name, symptom) or date range.
Load only the relevant entries into your context. Discard the rest.

If you find a matching entry, read the root cause and fix before touching anything.

---

## Step 3 — Find the right runbook

Use this index to find the specific runbook for the current problem type.
Fetch only the runbook(s) relevant to the current session.

| Problem | Runbook |
|---|---|
| Network / NIC / bond failure / link down / NO-CARRIER | `runbooks/network-bond-troubleshooting.md` |
| ShareSync bio errors | `runbooks/sharesync-bio-errors.md` |
| BTRFS corruption / scrub | `runbooks/btrfs-health.md` |
| Drive health / SMART errors | `runbooks/drive-health.md` |
| MCP tool timeouts or crashes | Read CLAUDE.md MCP section — no separate runbook |

All runbook URLs follow the pattern:
```
https://raw.githubusercontent.com/u2giants/albert-standards/main/synology/runbooks/<filename>
```

---

## Step 4 — Do the work

Connect to the NASes via the `synology-monitor` MCP server.
The only reliable tool is `run_command`. Read CLAUDE.md for the full list of tools that hang or crash.

**Critical MCP rules (also in CLAUDE.md):**
- Never chain commands with `&&` or `;` — causes hangs
- Never use: `check_system_info`, `get_resource_snapshot`, `disk_latency_test`, `start_smart_test`
- Session degrades after ~10-15 tool calls — full application restart is the only recovery

---

## Step 5 — Update the knowledgebase after the session

This is mandatory. The knowledgebase is only useful if it stays current.

### Always update after every session:

**`issue-history.md`** — add a new entry at the TOP (file is newest-first):

```markdown
## YYYY-MM-DD — [Short title]

**Units affected:** edgesynology1 / edgesynology2 / both
**Status:** Resolved / Ongoing / Investigating

### Symptom
What was observed. Include exact error messages and log snippets.

### Root cause
What actually caused it.

### Fix applied
Exactly what was done. Commands, settings changed, cables swapped.

### Outcome
Did it work? Any caveats or remaining concerns?

### Related
Links to any runbooks or post-mortems created.
```

**`CLAUDE.md`** — update any baseline table that changed:
- Bond state table (which ports are in the bond)
- BTRFS baseline (last scrub date, corruption_errs count)
- ShareSync bio error baseline (last error date)
- NIC/hardware notes if anything new was learned

### Create when appropriate:

**New runbook** in `runbooks/` — when a fix is proven and repeatable and no runbook exists yet.

**Post-mortem** in `post-mortems/` — when an issue is fully resolved. Name it `YYYY-MM-DD-short-title.md`. Write it once and never edit it again.

---

## GitHub access for updates

All updates go to:
- Repo: `u2giants/albert-standards`
- Token: `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`
- Base path: `synology/`

Use the GitHub Contents API (PUT) to update files.
Always GET the file first to retrieve its current SHA — the PUT will fail without it.

```
GET  https://api.github.com/repos/u2giants/albert-standards/contents/synology/<file>
PUT  https://api.github.com/repos/u2giants/albert-standards/contents/synology/<file>
     Body: { "message": "...", "content": "<base64>", "sha": "<current sha>" }
     Header: Authorization: token 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
```

---

## Full file map

```
synology/
├── agents.md                          ← YOU ARE HERE — give this to every AI session
├── CLAUDE.md                          ← hardware reference, MCP rules, health baselines
├── README.md                          ← human-readable overview and quick-index
├── issue-history.md                   ← full issue log (scan only, newest first)
├── hardware/
│   ├── nic-reference.md               ← NIC chips, MAC addresses, known defects
│   └── drive-inventory.md             ← drive models, RAID layout
├── runbooks/
│   ├── network-bond-troubleshooting.md
│   ├── sharesync-bio-errors.md
│   ├── btrfs-health.md
│   └── drive-health.md
└── post-mortems/
    └── 2026-05-14-aquantia-eth0-instability.md
```

---

## What not to do

- Do not read all files at once — fetch only what is relevant to the current problem
- Do not skip Step 2 (issue history check) — many issues are recurrences
- Do not edit post-mortems after they are written — they are historical records
- Do not make changes to the NASes without first checking CLAUDE.md for known quirks
- Do not run SMART tests via MCP (`start_smart_test` deadlocks the server) — use DSM UI
