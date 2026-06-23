# Infra Initiative — Handoff / Living Status

> The current snapshot of the "make the server rebuildable + trustworthy" initiative. **Rewrite the
> relevant parts each working session.** For the *why* behind choices, see [`DECISIONS.md`](DECISIONS.md).
> Last updated: **2026-06-22**.

## The goal (one paragraph)

Turn a hand-built Hetzner server (Coolify + ~20 app containers, hundreds of undocumented manual
changes) into something **rebuildable from code** with **trustworthy backups**, so a lost server or a
bad Hetzner backup no longer means ~a month of downtime. Approach: Infrastructure as Code (Ansible
for the host, Coolify keeps the apps) + 3-2-1 data backups + change-through-git discipline.
Driver is a "vibe-coder" (not a sysadmin) who approves diffs and holds credentials; AI authors and
maintains.

## ✅ Done so far

- **Backup outage fixed + hardened.** DB dumps had silently broken for 4 days; root-caused (stale
  docker.sock after a host Docker restart) and fixed. `pre-backup.sh` is now resilient/validating/
  loud; a host `systemd` watchdog self-heals it. Reclaimed 35 GB → ~640 MB. Added
  `BACKUP-MANIFEST.md`. (All in `backrest-wiz` → `hetzner-producer/`.)
- **Ansible plan hardened after expert review** — added phased execution with hard gates (§9),
  role-safety rules for firewall/docker/cloudflared (§4a), a seeded secrets inventory (§5.2), and
  the `cron_glue` ownership decision. See `DECISIONS.md`.
- **Twenty fully removed; Directus deprecated** (→ hosted supabase.com), removed from backups; live
  Directus app left running for the PopPIM migration.
- **Docs on GitHub:** backup system in `backrest-wiz`; host-Ansible plan in
  `albert-standards/ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`; this file + `DECISIONS.md` here.
- **Leaked GitHub token contained.** Removed from 4 git remotes (now use the `gh` credential
  helper); new PAT stored in 1Password (`vibe_coding/github-pat`) and swapped into `~/.netrc` and
  `~/.claude.json`.
- **Directus decommission reminder** scheduled (cloud routine `trig_01Crwkmo2gGLhTNqnNbNu4Qu`,
  fires 2026-07-22 — verifies the Supabase migration first).

## ⏭️ What's left (next time)

**Security / token (finish before revoking the old PAT):**
- Update the old token in the `/worksp/designflow-*` repos (couldn't reach them as user `ai`).
- Sweep each environment: `grep -rl '<the-old-token-you-are-revoking>' ~ /worksp` until empty, then **revoke** the old
  token on GitHub. Delete `~/.netrc.bak-20260622` + `~/.claude.json.bak-20260622` (hold the old token).
- **Rotate the DO Spaces key pair** — the access key *id* was briefly in `backrest-wiz` history.

**Backups (see the backup `HANDOFF.md` in `backrest-wiz`):**
- Add a **second, independent backup destination** (not yet 3-2-1; only DO Spaces today).
- **Rehearse a restore** — no backup has ever been test-restored.
- Restic history purge: Twenty can be purged as a monitored job anytime; Directus purge rides the
  2026-07-22 decommission reminder.

**The main build (the big one):**
- Execute `ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`: host-layer Ansible + a GitHub Actions apply
  pipeline (the serialized "one apply point" that also solves the 7-concurrent-AI-sessions problem).
- Migrate app secrets into 1Password (`op inject` at deploy time — Path A in `DECISIONS.md`),
  app-by-app, reviewed.

## Pointers

| What | Where |
|------|-------|
| The *why* behind decisions | [`DECISIONS.md`](DECISIONS.md) |
| Full host-Ansible + CI build plan | `albert-standards/ansible/ANSIBLE-IMPLEMENTATION-PLAN.md` |
| Live server reference (tunnels, DNS, "do not touch") | [`CLAUDE.md`](CLAUDE.md) |
| Backup system (producer config, incident, manifest, gaps) | `backrest-wiz` → `hetzner-producer/` |
| Backup monitor / restore UI | `backrest-wiz` (runs on a DigitalOcean droplet) |
| Past incidents | [`post-mortems/`](post-mortems/) · [`runbooks/`](runbooks/) |
