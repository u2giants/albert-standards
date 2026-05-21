# Infrastructure

Runbooks and post-mortems for the server infrastructure running Albert's applications.

## Structure

| Directory | Purpose |
|-----------|---------|
| [`runbooks/`](runbooks/) | Step-by-step diagnosis and fix guides for known failure modes |
| [`post-mortems/`](post-mortems/) | Incident records: what happened, root cause, what was fixed |

## Runbooks

| Runbook | When to use |
|---------|-------------|
| [DNS resolution broken](runbooks/dns-resolution-broken.md) | Apps can't reach external APIs; hostnames don't resolve but IP pings work |

## Post-Mortems

| Date | Incident |
|------|---------|
| [2026-05-20](post-mortems/2026-05-20-dns-cascade.md) | hiclaw→Twenty cascade caused DNS failure; Hetzner DNS servers unreachable |

## Server

Hetzner dedicated server running Ubuntu with:
- Docker (multiple app containers)
- Tailscale (remote access)
- Coolify (deployment orchestration)
- `systemd-resolved` for DNS (configured to use Cloudflare/Google/Quad9)
