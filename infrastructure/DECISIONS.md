# Infrastructure Decisions

An append-only log of the *why* behind how this infrastructure is built. New decisions go at the
top. Each entry: what we decided, why, and what we rejected. If you're about to change one of these,
read it first — the rejected options were usually rejected for a real reason.

---

## 2026-06-22 — Documentation lives in GitHub, not on the server or in AI memory

**Decision:** Cross-cutting infra docs (this file + `HANDOFF.md`) live in `albert-standards`
(GitHub). Backup-system docs live in `backrest-wiz`. The Ansible plan lives in
`albert-standards/ansible/`.
**Why:** The owner works through GitHub and can't reliably use local repos on the box. Work trapped
in local clones or an AI's private memory is effectively lost. GitHub is the durable, browsable home.
**Rejected:** Local repos on the server (stale, diverged, inaccessible to the owner); a new
dedicated repo (agent repo-creation is blocked, and more repos = more fragmentation).

## 2026-06-22 — Secrets: 1Password, injected at DEPLOY time (`op inject`/`op run`)

**Decision:** 1Password (vault `vibe_coding`, via the `op` CLI / a service account scoped to that
one vault) is the source of truth for secrets. Apps receive them via `op` **at deploy time** — the
deploy pipeline resolves `op://…` references into env vars and hands them to Coolify.
**Why:** Simple, no per-app changes, fits Coolify, keeps secrets out of git. An AI can set it up and
maintain it.
**Rejected:** `op run` as a **container entrypoint** (secrets fetched fresh on every container
start) — stronger at-rest security but far more moving parts (every container needs the `op` CLI +
a service-account token) and a new failure mode (no 1Password at boot = container won't start).
Too fragile for a non-sysadmin to debug. Revisit only if at-rest secret storage becomes a hard
requirement.

## 2026-06-22 — Git auth via the `gh` credential helper, not embedded tokens

**Decision:** Git remotes use no embedded tokens; auth flows through `gh auth setup-git`. The
canonical GitHub PAT is stored in 1Password (`vibe_coding/github-pat`).
**Why:** A PAT was found leaked in plaintext across multiple git remotes (and `~/.netrc`,
`~/.claude.json`). Embedding tokens in URLs leaks them into `.git/config`, logs, and transcripts.
**Rejected:** Keeping tokens in remote URLs (the status quo that caused the leak).

## 2026-06-22 — Backups: make them resilient, loud, and self-healing (not just "running")

**Decision:** The Hetzner backup producer (`pre-backup.sh`) is resilient (one DB failing doesn't
abort the rest), self-validating (size-check before overwriting a good dump), and loud (non-zero
exit on failure). A host `systemd` watchdog restarts the backup container if it loses docker.sock.
A `BACKUP-MANIFEST.md` is the reviewed source of truth for what gets backed up.
**Why:** DB dumps had silently broken for 4 days (a stale docker.sock mount after a host Docker
restart wrote an error *into* the dump file; `set -e` then froze every dump). "A backup that runs
is not a backup that works." See the backup README incident record in `backrest-wiz`.
**Rejected:** The original fail-silent script; relying on the in-container docker.sock without a
host-side safety net.

## 2026-06-22 — Twenty retired; Directus deprecated (→ hosted Supabase)

**Decision:** Twenty CRM fully removed. Directus deprecated in favor of hosted supabase.com; removed
from backups, app left running ~1 month for the PopPIM migration (scheduled decommission reminder
for 2026-07-22).
**Why:** Apps no longer in use shouldn't consume backup space or clutter configs; but a live app
mid-migration shouldn't be torn down (PopPIM still read through Directus).

## 2026-06-22 — The model: "pets → cattle" via Infrastructure as Code

**Decision:** Treat the server as reproducible "cattle," not a hand-raised "pet." The goal is
runnable code that rebuilds the host, plus data backups — not a server nobody can recreate.
**Why:** The server had hundreds of undocumented manual changes; losing it (or a bad Hetzner backup)
meant ~a month of downtime. IaC + 3-2-1 backups + change-via-code removes that single point of
failure.
**Rejected:** Documentation-only runbooks (drift out of date, can't rebuild); golden-image snapshots
as the *sole* strategy (opaque, can carry malware, go stale — fine as a *complement*).

## 2026-06-22 — Host IaC with Ansible, applied from GitHub Actions (not on-box, not NixOS)

**Decision:** Manage the **host/glue layer** with **Ansible**, applied by a **GitHub Actions**
pipeline. Coolify keeps owning the **app layer**. See `ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`.
**Why:**
- *Ansible over NixOS:* plain YAML, huge corpus → AI is reliable at it and the owner can follow a
  diff. NixOS is powerful but AI tooling on it is rougher and a non-sysadmin can't debug the dead
  ends.
- *GitHub Actions over on-box Ansible:* the control node is ephemeral/external (survives the server
  dying — critical for rebuild), it **serializes** applies (one run at a time), scales to N servers,
  and keeps the truth in git. On-box Ansible can't bootstrap a dead box and doesn't serialize.
- *Why serialization matters:* ~7 concurrent AI sessions touch shared infra; a single apply pipeline
  + per-project state is what stops them from re-creating drift. The guarantee comes from the
  pipeline, **not** from a "single coordinator chat" (which doesn't exist across tools and would
  blow its context window anyway).
**Rejected:** On-box Ansible; NixOS; a persistent cross-tool coordinator agent.
