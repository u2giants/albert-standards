# Infrastructure Decisions

An append-only log of the *why* behind how this infrastructure is built. New decisions go at the
top. Each entry: what we decided, why, and what we rejected. If you're about to change one of these,
read it first — the rejected options were usually rejected for a real reason.

---

## 2026-06-22 — Operational hardening of the apply pipeline (second review pass)

**Decision:** Four refinements to the Ansible/CI pipeline (details in the plan):
1. **Cloud-Init bootstrap** — on Hetzner *Cloud*, provision with a `user-data` script (create `ai`
   user + sudo, SSH key, Python, ephemeral Tailscale up) so the box is code-defined from first boot;
   no manual `bootstrap.sh`. (Dedicated/Robot servers use `installimage` instead — confirm product.)
2. **Scheduled drift detection** — a daily GitHub Actions job runs `ansible-playbook --check --diff`
   and fails+alerts on drift, catching out-of-band SSH hotfixes that merge-serialization can't.
3. **Ephemeral, tagged Tailscale CI auth** (`tailscale/github-action`, `tag:ci`) so runner nodes
   auto-remove from the tailnet instead of accumulating.
4. **1Password SA-token expiry handling** — the CI service-account token can expire silently and kill
   the whole pipeline; record its expiry and set a rotation reminder when it's created.
**Why:** Second expert-review pass. Each closes a real failure mode (a manual bootstrap step, silent
out-of-band drift, tailnet node sprawl, silent pipeline death on token expiry).
**Rejected:** a manual `bootstrap.sh` as the *only* bootstrap; relying solely on merge-time
serialization with no drift detection.

## 2026-06-22 — Ansible rollout is gated by phases; no auto-apply until proven safe

**Decision:** The host-Ansible build runs in five phases with **hard gates** between them, and CI
stays **check/PR-diff only** until idempotency is clean *and* the risky roles pass a rebuild-and-diff
on a throwaway host. Phases: (0) observe & scaffold → (1) non-disruptive roles → (2) risky
live-service roles (firewall/docker/cloudflared) → (3) secrets migration → (4) CI auto-apply.
Guardrails baked in: firewall changes keep a **Tailscale out-of-band lifeline** (`100.66.37.58`, by
IP — MagicDNS is off) + an **auto-revert timer**; the Docker role is **version-pinned and never
auto-restarts**; `--check` is **not trusted on prod** for the risky roles (prove on scratch first);
secrets migrate one at a time with per-secret validation + rollback (seeded inventory table in the plan).
**Why:** Incorporates an expert review of `ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`. The owner can't
quickly recover from a firewall lockout, a Docker restart that takes down Coolify, or a silently
broken backup — so on *this* server the extra rigor is worth it. "A plan you haven't proven on a
throwaway box is a hope."
**Rejected:** "safest-first, apply each role then move on" with no scratch-host proof and no explicit
auto-apply gate (the original §9) — too easy to push a lockout/outage straight to prod.

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

## 2026-06-22 — legacy application backends retired

**Decision:** The earlier CRM was fully removed. The later legacy CMS was deprecated in favor of hosted supabase.com; removed
from backups, app left running ~1 month for the PopPIM migration (scheduled decommission reminder
for 2026-07-22).
**Why:** Apps no longer in use shouldn't consume backup space or clutter configs; but a live app
mid-migration should not be torn down until all PopPIM reads are migrated.

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
