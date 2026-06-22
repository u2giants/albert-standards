# Implementation Plan — Host-Layer Ansible + GitHub Actions Apply Pipeline

> **You are a fresh AI session with no prior context. Read this entire document before doing
> anything.** It is the complete brief for turning a hand-built production server into a
> reproducible, code-managed system. It was written by a previous session that audited the server
> in depth; every non-obvious detail here was learned the hard way. Do not skip sections because
> they "look like background" — the background is where the landmines are.

---

## 0. TL;DR of what you are building

1. An **Ansible** project (in this repo, `u2giants/albert-standards (this repo, ansible/ folder)`) that declaratively manages the
   **host/OS layer** of one Hetzner VPS — packages, users, firewall, systemd units, cron,
   Docker, and the "glue" scripts — so the box can be rebuilt from code.
2. A **GitHub Actions workflow** that is the **single, serialized apply point**: changes land as
   pull requests; merging to `main` runs `ansible-playbook` against the server. No human and no
   other AI session applies host changes by hand, ever.
3. **Secrets sourced from 1Password** (`op` CLI) at apply time — never committed, never in plaintext
   `.env` on disk where avoidable.

The end state: if the server is destroyed, a new Ubuntu box + this repo + the 1Password vault +
the data backups = full recovery, with no tribal knowledge required.

---

## 1. Who this is for and why (do not lose this nuance)

The owner is a **"vibe-coder": not a sysadmin or devops engineer.** They drive everything through
AI (Claude Code, Codex) and act as approver/operator, not implementer. Consequences that must
shape every decision you make:

- **Favor boring, well-documented tooling.** Ansible (plain YAML, huge corpus) over NixOS/Salt.
  The owner cannot debug a clever solution when it breaks; neither can the next AI session reliably.
- **Explain, don't just emit.** Every role and workflow needs a README a non-engineer can follow.
- **Optimize for "AI authors, human approves."** The human reads diffs and clicks merge. The
  pipeline must make the safe path the easy path.
- **The owner runs ~7 concurrent AI sessions across ~5 apps.** The single most important property
  of this system is preventing those sessions from making conflicting, drift-inducing changes to
  one shared server. See §6.

The mental model the owner has bought into: **"pets vs. cattle."** The server is currently a
hand-raised *pet* (a snowflake nobody fully understands). The goal is *cattle*: rebuildable from
code, where every change that ever made it special is written down as runnable code, not done by
hand. **A descriptive `.md` of the server is NOT the goal — runnable code that rebuilds it is.**

---

## 2. The server you are managing

- **Provider/OS:** Hetzner VPS, **Ubuntu 24.04.4 LTS** ("noble"), 16 GB. Hostname `hetz`.
- **Public IP:** `178.156.180.212`. **Tailscale IP:** `100.66.37.58`. Domain: `designflow.app`
  (DNS on Cloudflare).
- **Primary user:** `ai` (has **passwordless sudo**). Also `root`. SSH in as `ai` or `root`.
- **This is a single server today.** Design the Ansible inventory to scale to N hosts (there is
  also a DigitalOcean droplet running the backup monitor — see §2.3 — and a "compshop" server
  referenced in backup configs), but only this one host is in scope for the first iteration.

### 2.1 The two layers (critical mental model)

**Do not try to capture everything equally.** The server has two layers in very different shape:

- **App layer = Coolify.** Coolify (`coolify`, `coolify-db`, `coolify-redis`, `coolify-realtime`,
  `coolify-sentinel`, `coolify-proxy`) manages ~20 application containers (the Pop apps, HiClaw,
  DevOps MCP/ContextForge, Synology Monitor, EmailCleanup, etc.). **This layer is already codified**
  — Coolify deploys from GitHub repos/images and stores its state in `coolify-db` + `/data/coolify`.
  **Ansible must NOT try to manage application containers.** That is Coolify's job; fighting it
  causes reconcile loops.
- **Host/glue layer = your job.** Everything Coolify does *not* manage: the OS, packages, Docker
  engine install, the firewall, system cron jobs, manually-managed systemd units (tunnels, the
  backup watchdog), `/etc` config, users/SSH. **This is what your Ansible encodes.**

So the scope boundary is: **Ansible owns the host and the glue; Coolify owns the apps.** Where they
meet (e.g. the Docker daemon Coolify runs on, the Traefik proxy config Coolify generates) is
documented in §3 as "manage the install, leave the runtime to Coolify."

### 2.2 What actually runs at the host level (encode these)

Discover the live truth on the box (commands in §7), but here is what the audit found so you know
what to expect:

- **Docker** (engine + compose plugin), **containerd**. Coolify and all apps run on this.
- **Enabled systemd services** (non-default highlights): `docker`, `containerd`, `fail2ban`,
  `netfilter-persistent` (persists iptables), `cloudflared-coolify.service` (Cloudflare tunnel,
  systemd-managed, NOT a container), `coolify-autostart.service`, `snapd`, and
  `backrest-dump-watchdog.timer` (added 2026-06-22, self-heals the backup agent).
- **Disabled but present:** `socks5-home-tunnel.service` (SSH SOCKS proxy to a home machine via
  Tailscale; intentionally disabled — do NOT enable).
- **Cron (root):** several `/worksp/hiclaw/*.sh` "keeper" scripts run every minute (config
  reconciliation for HiClaw/OpenClaw); `/home/ai/bin/sync-infra-docs.sh` every 15 min
  (auto-commits infra docs to git). `/etc/cron.d`: `e2scrub_all`, `sysstat`.
- **Firewall:** iptables rules persisted at `/etc/iptables/rules.v4` and `rules.v6` via
  `netfilter-persistent`. `fail2ban` enabled.
- **DNS hardening (already applied, see §3.3):** `/etc/systemd/resolved.conf.d/fallback-dns.conf`,
  `/etc/docker/daemon.json` with `"live-restore": true`.
- **Glue scripts:** `/usr/local/bin/backrest-dump-watchdog.sh`,
  historically a `docker-rename-containers.sh` (a rename service no longer present — verify).

### 2.3 Related machines (context, not first-iteration scope)

- **DigitalOcean droplet** running `restore-wizard` / `backrest-wiz` (the backup monitor UI at
  `backup.designflow.app`). It already has an **Ansible playbook** authored in the `restore-wizard`
  repo under `ansible/` (idempotent: Docker, the compose stack, `.env` from vaulted vars). Fold
  this droplet into the same inventory/pipeline eventually so both servers are managed from one
  place. **This Hetzner box has no SSH path to the droplet** (the droplet deploys via its own
  GitHub Actions using `DEPLOY_HOST`/`DEPLOY_SSH_KEY` secrets) — so manage it from the CI runner.

---

## 3. Server-specific landmines (this is the section that prevents outages)

These are real incidents and gotchas from this exact server. Your Ansible must respect every one.
The authoritative live reference is `/worksp/infra/CLAUDE.md` and `/home/ai/CLAUDE.md` on the box —
**read both before writing roles.** Summary of what must not be broken:

### 3.1 Cloudflare Tunnels — three independent tunnels, do not consolidate
- **Tunnel 1** `coolify.designflow.app`: **systemd-managed** `cloudflared-coolify.service`, token in
  `/etc/cloudflared/coolify-tunnel.env` (root-only, not in git). Routes by **anchored regex** paths
  (`^/app(/|$)` → Soketi `:6001`, `^/terminal/ws(/|$)` → `:6002`, else Coolify `:8000`).
  **The regexes MUST stay anchored** — an unanchored `/app` matches asset filenames like
  `app-C9Z.js` and misroutes them (404s, then Cloudflare caches the 404 for 4h). Config lives
  remotely in Cloudflare, not in a local file.
- **Tunnels 2 & 3** (`mcp.designflow.app`, `mcpgw.designflow.app`): **Coolify-managed containers**
  (`cloudflared-vj5...`, `cf-cloudflared-vj5...`). **Coolify owns their lifecycle — never
  start/stop/recreate them by hand.**
- **Ansible implication:** you may manage Tunnel 1's systemd unit + its env file (root-only secret
  via 1Password), but **must not** touch Tunnels 2/3. Do not create a generic "cloudflared" role
  that would fight Coolify.

### 3.2 Traefik (`coolify-proxy`) — Coolify overwrites your changes
- Config in `/data/coolify/proxy/`. Two cert resolvers: `letsencrypt` (HTTP-01, direct subdomains)
  and `letsencrypt-dns` (DNS-01 via Cloudflare API token `CF_DNS_API_TOKEN`, for tunnel subdomains).
- **Coolify reverts `certresolver: letsencrypt-dns` back to `letsencrypt` whenever the proxy is
  restarted from the UI**, which breaks `coolify.designflow.app` certs. There is a manual `sed`
  fix documented. **Do not have Ansible manage Traefik's dynamic config** — it's Coolify-owned and
  will fight you. At most, document the gotcha.

### 3.3 DNS cascade (the May 2026 outage) — keep the fixes intact
- The host uses `systemd-resolved` (`127.0.0.53`), which is **unreachable from inside containers.**
  When host DNS broke, Docker's DNS relay (`127.0.0.11`) broke, and **every tunnel/oauth2-proxy
  container crashed simultaneously.** Fixes that are now in place and your Ansible must preserve
  (encode them as managed files so a rebuild reapplies them):
  - `/etc/systemd/resolved.conf.d/fallback-dns.conf` → `FallbackDNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4`
  - `/etc/docker/daemon.json` → `{"live-restore": true}` and **NO `dns` key** (Docker 27+ reads the
    real upstream from resolved automatically; pointing it at `127.0.0.53` breaks all containers).
  - `oauth2-proxy` compose sets per-container `dns: [1.1.1.1, 8.8.8.8]` (it does OIDC discovery on
    every startup and exits code 1 if DNS fails). That file is at
    `/worksp/hiclaw/oauth2-proxy/docker-compose.yml` — manually managed, not Coolify.

### 3.4 The docker.sock staleness lesson (the June 2026 outage)
- Long-running containers that bind-mount `/var/run/docker.sock` (e.g. the `backrest` backup agent)
  **lose access when the host Docker daemon restarts** (the socket is recreated; the container holds
  a stale handle). This silently broke database backups for 4 days. The same event broke
  `coolify-proxy` route discovery. **Fix already deployed:** `backrest-dump-watchdog.timer` restarts
  the affected container when it detects the wedged state. **Lesson for your roles:** anything that
  depends on docker.sock from inside a container needs either a host-side execution path or a
  watchdog. Prefer running host-level scripts (cron/systemd on the host) over in-container
  docker.sock when you have the choice.

### 3.5 Things you must NOT do
- Do **not** manage application containers or Coolify-owned resources with Ansible.
- Do **not** set `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY` anywhere (the SOCKS tunnel on `:1080` is
  intentionally disabled; proxy env vars silently break tools/containers).
- Do **not** enable `socks5-home-tunnel.service`.
- Do **not** add a `dns` key to `daemon.json`.
- Do **not** commit secrets. Tunnel tokens, the Cloudflare API token, restic/S3 creds, the GHCR
  PAT, OAuth secrets all live in **1Password** (vault `vibe_coding`) and are injected at apply time.
- Do **not** create/push new external repos or relocate infra content without the owner's explicit
  approval (the permission classifier blocks this by design; surface it, don't work around it).

---

## 4. Target repository layout (build this under ansible/ in this repo)

```
server-infra/
├── README.md                     # what this is, how to apply, how to add a change (non-engineer level)
├── ansible.cfg                   # inventory path, ssh settings, no host key prompt in CI
├── inventory/
│   ├── hosts.ini                 # [hetzner] and (later) [do_backup_wiz] groups
│   └── group_vars/
│       └── all.yml               # non-secret vars; secrets pulled from 1Password at runtime
├── playbooks/
│   └── site.yml                  # the entrypoint: applies all roles to hetzner
├── roles/
│   ├── base/                     # apt packages, timezone, unattended-upgrades, journald limits
│   ├── users/                    # the `ai` user, sudo, authorized_keys (public keys only)
│   ├── firewall/                 # iptables rules.v4/v6 + netfilter-persistent + fail2ban
│   ├── docker/                   # engine + compose plugin + daemon.json (live-restore) ONLY
│   ├── dns_hardening/            # resolved.conf.d/fallback-dns.conf  (see §3.3)
│   ├── cloudflared_coolify/      # Tunnel 1 systemd unit + env file (token from 1Password)
│   ├── cron_glue/               # /worksp/hiclaw keepers, sync-infra-docs (managed copies)
│   └── backrest_watchdog/        # the backup self-heal timer (already authored in backrest-hetzner)
├── files/ , templates/           # static files + jinja templates referenced by roles
└── .github/workflows/
    └── apply.yml                 # the serialized apply pipeline (see §5)
```

Notes:
- Keep roles **small and single-purpose**; each maps to one landmine-free unit of the host.
- Use `ansible.builtin` modules only where possible (no exotic collection dependencies) so any AI
  session can run it with a stock Ansible install.
- Make **every task idempotent** and safe to re-run. Use `--check` (dry-run) in CI on PRs.

---

## 5. The GitHub Actions apply pipeline (the serialization point)

This is the heart of the "7 AI sessions can't collide" guarantee. Design:

### 5.1 Flow
1. Any change to the host is made by **editing this repo and opening a PR** — never by SSHing in.
2. **On PR:** CI runs `ansible-lint` + `ansible-playbook --check --diff` against the server (read-only
   dry run) and posts the diff. The human reviews what *would* change.
3. **On merge to `main`:** CI runs `ansible-playbook` for real. **Concurrency is serialized** so two
   merges never apply at once:
   ```yaml
   concurrency:
     group: apply-hetzner
     cancel-in-progress: false   # queue, never run two applies simultaneously
   ```
4. The runner reaches the server over **SSH via Tailscale** (preferred) or public IP. Use a
   dedicated CI SSH key stored in GitHub Actions secrets (or better, a Tailscale ephemeral auth
   key + `tailscale/github-action`).

### 5.2 Secrets (1Password)
- The owner wants secrets in **1Password** (`op` CLI; vault `vibe_coding`). In CI, use the official
  **`1password/load-secrets-action`** with a **1Password Service Account token** (the only secret
  stored directly in GitHub). All other secrets (SSH key, tunnel tokens, CF API token, restic/S3
  creds) are referenced as `op://vibe_coding/<item>/<field>` and injected as env vars at apply time,
  then passed to Ansible via `--extra-vars` or `lookup('env', ...)`.
- **Prerequisite task:** the secrets must first be *put into* 1Password. Today they are NOT — they
  live in plaintext `.env` files on the server (e.g. `restore-wizard/.env` has 25 plaintext values)
  and one was even embedded in a git remote. Part of this project is migrating them into the
  `vibe_coding` vault. Do this carefully, one secret at a time, verifying the app still works; this
  is the riskiest part and should be its own reviewed phase.

### 5.3 Why CI and not on-box Ansible
The owner explicitly chose a CI runner over installing Ansible on the server because: the control
node is **ephemeral/external** (survives the server dying — critical for rebuild), it **naturally
serializes** applies, it **scales to N servers** via inventory, and it keeps the **truth in git**.
On-box Ansible was rejected (chicken-and-egg on rebuild, no serialization). Honor this choice.

### 5.4 First-boot bootstrap
A brand-new server can't be reached by CI until it has an SSH user + key + Tailscale. Provide a
small `bootstrap.sh` (or a separate minimal playbook run once from a laptop) that: creates the `ai`
user, installs the CI public key, installs Tailscale and joins the tailnet, installs Python (for
Ansible). After bootstrap, everything else is CI-driven. Document this clearly — it's the one
manual step in a rebuild.

---

## 6. The 7-concurrent-AI-sessions problem (design requirement, not optional)

The owner runs many AI sessions against shared infrastructure. Without discipline they re-create
the original "pet server" drift. The system must enforce:

- **One apply path.** Host changes happen ONLY through this repo's PR→merge→CI pipeline. The CI
  `concurrency` group serializes them. No session runs `ansible-playbook` locally against prod.
- **App changes stay in app repos** and deploy through Coolify (already the case). Ansible is for
  the host only, so app sessions and infra sessions don't touch the same files.
- **The filesystem/git is the source of truth, not any chat's memory.** A coordinator pattern, if
  built later, must read state from git/files and dispatch scoped workers — never hold everything in
  one context window. (The owner asked about a "single coordinator"; the honest answer is there is
  no cross-tool always-on coordinator, and the real guarantee comes from the serialized pipeline +
  per-project state, not a chat UI.)
- **Document the rule loudly** in the repo README: "To change the server, change this repo. Never
  `apt install` / `crontab -e` / edit `/etc` on the box directly. Manual changes will be silently
  reverted by the next apply."

---

## 7. How to discover the current state to encode (run these first)

Do not guess what's on the box — inventory it, then write roles that reproduce it. Useful commands
(run on the server, you have sudo):

```bash
# Packages explicitly installed (not pulled in as deps):
comm -23 <(apt-mark showmanual | sort) <(gzip -dc /var/log/installer/initial-status.gz 2>/dev/null | sed -n 's/^Package: //p' | sort)
# Enabled services:
systemctl list-unit-files --state=enabled --type=service
systemctl list-timers --all
# Cron:
crontab -l; sudo ls -la /etc/cron.d /etc/cron.daily; sudo cat /var/spool/cron/crontabs/* 2>/dev/null
# Firewall:
sudo iptables-save; sudo cat /etc/iptables/rules.v4 /etc/iptables/rules.v6
# Users with login + sudo:
getent passwd | awk -F: '$7!~/nologin|false/'; sudo cat /etc/sudoers.d/*
# Managed /etc files of interest:
ls -la /etc/systemd/resolved.conf.d/ /etc/docker/daemon.json /etc/cloudflared/
# Glue scripts:
ls -la /usr/local/bin/ /home/ai/bin/ /worksp/hiclaw/*.sh
# Docker engine version + compose:
docker version; docker compose version
```

Encode the **host** outputs of these into roles. Skip anything Coolify-owned (app containers,
Traefik dynamic config, Coolify's own systemd autostart beyond noting it exists).

---

## 8. Verification — definition of done (do NOT skip; this is the whole point)

A backup/rebuild plan you have never tested is a hope, not a plan. Prove it:

1. **Idempotency:** running `ansible-playbook playbooks/site.yml` twice yields **zero changes** on
   the second run. If not, the role isn't truly declarative — fix it.
2. **`--check` cleanliness:** on the real server, `--check --diff` shows no surprise drift after a
   successful apply.
3. **Rebuild-and-diff (the real test):** provision a **throwaway** Hetzner/DO box, run bootstrap +
   the full pipeline against it, and **diff it against the real server**: package lists, enabled
   services, listening ports, firewall rules, key `/etc` files. Every difference is either a bug in
   your roles or an undocumented manual change on prod — chase each to zero. This is how you *prove*
   completeness instead of hoping.
4. **No secrets in git:** `git log -p | grep -iE 'BEGIN.*PRIVATE|ghp_|gho_|aws_secret|password='`
   returns nothing. Confirm `.gitignore` covers vault files, inventories with IPs if sensitive, etc.
5. **Pipeline self-test:** open a trivial PR (e.g. add a managed motd line), confirm the `--check`
   diff posts, merge, confirm it applies and the concurrency group serializes.

---

## 9. Sequencing (suggested order of work)

1. **Scaffold** the repo (§4), `ansible.cfg`, inventory with the one host, an empty `site.yml`.
2. **Inventory the box** (§7); write the README scope/rules.
3. **Roles, safest first:** `base` → `users` → `dns_hardening` → `firewall` → `docker` (install +
   daemon.json only) → `cron_glue` → `backrest_watchdog` → `cloudflared_coolify` (last; touches a
   live tunnel). Apply each with `--check` first, then for real, verifying nothing breaks.
4. **Secrets → 1Password** (§5.2): migrate one secret at a time; verify each app still works.
5. **CI pipeline** (§5): start with `--check` on PRs only (no auto-apply) until you trust it, then
   enable apply-on-merge with the concurrency guard.
6. **Fold in the DO droplet** (§2.3) as a second inventory group using the existing `restore-wizard`
   Ansible.
7. **Rebuild-and-diff test** (§8.3). Only after this passes is the project "done."

---

## 10. Things to confirm with the owner before/while building (open questions)

- **Secrets migration scope & timing** — moving ~25+ plaintext secrets into 1Password is risky;
  confirm the owner wants this now and do it in a reviewed, app-by-app way.
- **CI runner network path** — Tailscale (recommended) vs public-IP SSH. Needs a CI key/auth method
  the owner provisions.
- **Repo creation/push approval** — creating `server-infra` content and pushing requires the owner's
  explicit OK (classifier blocks agent-initiated repo creation/bulk push). This repo already exists
  (`u2giants/albert-standards (this repo, ansible/ folder)`) so prefer committing here over creating anything new.
- **How aggressive to be with `cron_glue`** — the `/worksp/hiclaw/*` keepers are app-adjacent;
  confirm whether they belong in host Ansible or should stay with HiClaw.
- **Directus teardown** — Directus is being deprecated (PopPIM migrating to hosted supabase.com).
  If/when the `directus-app`/`directus-db` containers are fully retired, remove them via Coolify
  (not Ansible) and update docs.

---

## 11. Reference material already on the box (read these)

- `/worksp/infra/CLAUDE.md` and `/home/ai/CLAUDE.md` — the authoritative live server reference
  (traffic routing, tunnels, DNS, ports, Cloudflare IDs, "things you must not do"). **Read fully.**
- `/home/ai/backrest-hetzner/` — the backup system: `README.md` (incident record + how it works),
  `BACKUP-MANIFEST.md` (what's backed up), `HANDOFF.md` (3-2-1 + restore-test gaps),
  `bin/` + `systemd/` (the watchdog you'll wrap in a role).
- `/home/ai/restore-wizard/ansible/` — the existing droplet playbook to model the DO host on and
  eventually fold into this inventory.
- `/worksp/albert-standards/infrastructure/` — prior incident post-mortems and runbooks (DNS
  cascade, cloudflared/oauth2 recovery) — good context, but it is **docs only, not runnable IaC**.

---

*Written 2026-06-22 by an audit session. If anything here conflicts with what you observe live on
the server, trust the live server, update this document, and tell the owner what changed.*
