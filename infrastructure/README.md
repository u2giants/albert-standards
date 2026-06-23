# Infrastructure

Runbooks and post-mortems for the server infrastructure running Albert's applications.

## Structure

| Path | Purpose |
|------|---------|
| [`CLAUDE.md`](CLAUDE.md) | **Full server reference** — traffic routing, tunnel inventory, Docker DNS architecture, runbooks, "do not touch" rules. Auto-loaded into every Claude Code session on the server. |
| [`HANDOFF.md`](HANDOFF.md) | **Start here for the IaC/backup initiative** — current status, what's done, what's left for next time. Rewritten each session. |
| [`DECISIONS.md`](DECISIONS.md) | **The *why* log** — every architecture decision, when, and what was rejected. Append-only. |
| [`runbooks/`](runbooks/) | Step-by-step diagnosis and fix guides for known failure modes |
| [`post-mortems/`](post-mortems/) | Incident records: what happened, root cause, what was fixed |

> The full host-Ansible + CI build plan lives in [`../ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`](../ansible/ANSIBLE-IMPLEMENTATION-PLAN.md). Backup-system docs live in the `backrest-wiz` repo.

## Runbooks

| Runbook | When to use |
|---------|-------------|
| [DNS resolution broken](runbooks/dns-resolution-broken.md) | Apps can't reach external APIs; hostnames don't resolve but IP pings work |
| [Container resource limits](runbooks/container-resource-limits.md) | Every service must have memory/CPU limits; how to set and verify them |

## Applications on this infrastructure

| Application | Public hostnames | Runtime owner | Notes |
|-------------|------------------|---------------|-------|
| PopDAM / PopSG | `dam.designflow.app`, `sg.designflow.app` | Coolify app `qxj8a0j3tpa9lq4q5rs6pezy` on the Hetzner VPS | One `ghcr.io/u2giants/popdam-frontend:latest` image serves both hostnames; `dam` is routed by Coolify Docker labels and `sg` by `/data/coolify/proxy/dynamic/popdam-sg.yml` using the same Docker service. |

When a project-specific infrastructure decision for PopDAM changes VPS, Coolify,
Traefik, GHCR, DNS, Railway, or Synology operating assumptions, update this
knowledgebase and the PopDAM repo docs together.

## Post-Mortems

| Date | Incident |
|------|---------|
| [2026-05-21](post-mortems/2026-05-21-cloudflared-oauth2-recovery.md) | Docker DNS cascade; cloudflared + oauth2-proxy containers recovered; Docker hardened |
| [2026-05-20](post-mortems/2026-05-20-hetzner-crash.md) | Hetzner VPS unclean reboot; HiClaw recursive MinIO mirror (primary) + Twenty email rerouter query storm (contributing) |
| [2026-05-20](post-mortems/2026-05-20-dns-cascade.md) | hiclaw→Twenty cascade caused DNS failure; Hetzner DNS servers unreachable |

## Server

Hetzner VPS (`178.156.180.212`) running Ubuntu with:
- Docker (all apps containerized)
- Coolify (deployment orchestration, owns most containers)
- Traefik (`coolify-proxy` container, reverse proxies all traffic)
- Cloudflare Tunnels (3 active: `coolify`, `mcp`, `mcpgw` subdomains)
- Authentik (`auth.designflow.app`, identity provider)
- `systemd-resolved` with Cloudflare/Google/Quad9 DNS + fallback
