# Infra Initiative — Handoff / Living Status

> The current snapshot of the "make the server rebuildable + trustworthy" initiative. **Rewrite the
> relevant parts each working session.** For the *why* behind choices, see [`DECISIONS.md`](DECISIONS.md).
> Last updated: **2026-07-22**.

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
- **Legacy backends retired** in favor of hosted supabase.com and removed from routine backups; one
  retired app was temporarily left running for the PopPIM migration.
- **Docs on GitHub:** backup system in `backrest-wiz`; host-Ansible plan in
  `albert-standards/ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`; this file + `DECISIONS.md` here.
- **Leaked GitHub token contained.** Removed from 4 git remotes (now use the `gh` credential
  helper); new PAT stored in 1Password (`vibe_coding/github-pat`) and swapped into `~/.netrc` and
  `~/.claude.json`.
- **Legacy-backend decommission reminder** scheduled (cloud routine `trig_01Crwkmo2gGLhTNqkNbNu4Qu`,
  fires 2026-07-22 — verifies the Supabase migration first).
- **Directus migration confirmed complete (2026-07-22).** Supabase `popdam` project
  (`qsllyeztdwjgirsysgai`) has 132K+ assets, 286K+ style-guide files, active users and daily
  scans — fully live on Supabase. Decommission runbook issued; containers/volumes/restic purge
  require manual execution (see ACTION-REQUIRED note below and the notification sent 2026-07-22).

## 🔴 ACTION REQUIRED — Directus Decommission (2026-07-22)

Migration confirmed complete. All steps below require human execution on the Hetzner VPS
(`178.156.180.212`) or via Coolify. Do in order.

### (a) Remove containers + volumes via Coolify

1. Log in to `https://coolify.designflow.app`
2. Find and delete the **`directus-app`** service (stop → delete service)
3. Find and delete the **`directus-db`** service (stop → delete service)
4. Delete the two Docker volumes — via Coolify UI or SSH:
   ```bash
   docker volume rm nzli85mk3luzb6u7cnq5fidu_directus-pgdata
   docker volume rm nzli85mk3luzb6u7cnq5fidu_directus-extensions
   ```
   Combined ~3.7 GB freed.

### (b) Purge Directus from restic backup history (DO Spaces)

Directus was already removed from the backup *schedule*; this purges old snapshot data.
Run from inside the `backrest` container on the host (HEAVY — plan ~30–60 min, monitor it):

```bash
# SSH to the host, then:
docker exec -it backrest sh

# Read repo creds from config:
cat /opt/backrest/config/config.json
# Extract: repo URI, password, and DO Spaces key/secret

export RESTIC_REPOSITORY="s3:..."     # from config
export RESTIC_PASSWORD="..."           # from config
export AWS_ACCESS_KEY_ID="..."         # from config
export AWS_SECRET_ACCESS_KEY="..."     # from config

# Rewrite snapshots to exclude Directus paths:
restic rewrite --exclude '/worksp/directus' \
               --exclude '/home/ai/backups/*directus*' \
               --exclude '*/directus-pgdata*'

# Then prune unreferenced data (run monitored — this is the heavy step):
restic prune --max-repack-size 10G

# Verify cleanup:
restic snapshots
restic stats
```

### (c) Delete Directus dump files on host

```bash
# Preview first:
ls -lh /worksp/directus/ 2>/dev/null
ls -lh /home/ai/backups/ | grep -i directus

# Delete once satisfied nothing is needed:
rm -rf /worksp/directus/
ls /home/ai/backups/ | grep -i directus | xargs -I{} rm /home/ai/backups/{}
```

### (d) Update server-side documentation

After all of the above is done, on the host:

```bash
# Mark Directus fully removed in the backup manifest:
nano /home/ai/backrest-hetzner/BACKUP-MANIFEST.md
# Add: "Directus: FULLY DECOMMISSIONED 2026-07-22. Containers, volumes, and restic history removed."

# Mark in infra memory:
nano /home/ai/.claude/projects/-/memory/infra-as-code-initiative.md
# Update Directus status to: removed 2026-07-22
```

Then update this file (`infrastructure/HANDOFF.md`) in `albert-standards`: move the
Directus decommission item from the ACTION REQUIRED section to ✅ Done.

---

## ⏭️ What's left (next time)

**Security / token (finish before revoking the old PAT):**
- Update the old token in the `/worksp/designflow-*` repos (couldn't reach them as user `ai`).
- Sweep each environment: `grep -rl '<the-old-token-you-are-revoking>' ~ /worksp` until empty, then **revoke** the old
  token on GitHub. Delete `~/.netrc.bak-20260622` + `~/.claude.json.bak-20260622` (hold the old token).
- **Rotate the DO Spaces key pair** — the access key *id* was briefly in `backrest-wiz` history.

**Backups (see the backup `HANDOFF.md` in `backrest-wiz`):**
- Add a **second, independent backup destination** (not yet 3-2-1; only DO Spaces today).
- **Rehearse a restore** — no backup has ever been test-restored.
- Restic history purge: see ACTION REQUIRED section above — Directus purge is the immediate item.

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
