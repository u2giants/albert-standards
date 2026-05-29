# Agent Instructions — Synology Knowledgebase

Instructions for every AI session working on these NAS systems.

---

## Session start protocol

1. **Always read `CLAUDE.md` first.** It contains hardware facts, NIC map, MCP rules, and known quirks. This is your anchor.
2. **Check `issue-history.md` for the current problem** — scan by keyword or date range. Do NOT ingest the entire file. Use a targeted search: grep for the symptom, error message, or component name.
3. **Check the relevant runbook** using the quick index in `README.md`.
4. Then connect to the NASes via synology-monitor MCP (`run_command` only — see CLAUDE.md for rules).

## Session end protocol

After every session that produces new findings, fixes, or learnings:

1. **Update `issue-history.md`** — add a new entry at the TOP of the file (newest first). Use the format below.
2. **Update `CLAUDE.md`** — if any baseline facts changed (bond state, BTRFS stats, NIC config, etc.), update the relevant table.
3. **Update the relevant runbook** — if a fix worked or didn't work, note it. If a new runbook is needed, create it in `runbooks/`.
4. **Create a post-mortem** in `post-mortems/` if an issue was fully resolved.
5. **Update Claude memory** — update the memory entry for Synology to reflect any changed baseline state.

---

## issue-history.md format

Add entries at the TOP (file is newest-first). Each entry:

```
## YYYY-MM-DD — [Short title]

**Units affected:** edgesynology1 / edgesynology2 / both
**Status:** Resolved / Ongoing / Investigating

### Symptom
What was observed. Error messages, log entries, commands that showed the problem.

### Root cause
What actually caused it.

### Fix applied
Exactly what was done. Commands run, cables swapped, settings changed.

### Outcome
Did it work? Any remaining concerns?

### Related
Links to runbooks or post-mortems created from this issue.
```

---

## File ownership rules

| File | Who updates it | When |
|---|---|---|
| `CLAUDE.md` | AI agent | Any time a baseline fact changes |
| `issue-history.md` | AI agent | After every session with new findings |
| `agents.md` | AI agent | If the process itself needs to change |
| `hardware/*.md` | AI agent | Only if hardware physically changes |
| `runbooks/*.md` | AI agent | When a new fix is proven or a step is found wrong |
| `post-mortems/*.md` | AI agent | Once per resolved incident, never edited after |

---

## Writing rules

- **Be specific.** Include exact error messages, exact command output, exact file paths.
- **Be terse.** Future sessions have limited context windows. No prose padding.
- **Never ingest `issue-history.md` fully.** It will grow very large. Always scan/grep for the relevant section.
- **Silo your reads.** Only fetch the files relevant to the current problem. The README quick-index tells you which ones.
- **Update memory.** After every session, ensure Claude's persistent memory reflects the current bond state, any new known issues, and any resolved issues.

---

## GitHub access

Push updates via the GitHub API using token `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`.
Repo: `u2giants/albert-standards`
Base path: `synology/`

Always GET the file's current SHA before a PUT (required by GitHub API to avoid conflicts).
