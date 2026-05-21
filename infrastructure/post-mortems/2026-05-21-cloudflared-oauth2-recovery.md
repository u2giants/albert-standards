# Post-Mortem: Cloudflared + OAuth2 Proxy Container Recovery — 2026-05-21

## Summary

Following the 2026-05-20 DNS cascade (see [2026-05-20-dns-cascade.md](2026-05-20-dns-cascade.md)), three Docker containers that required DNS at startup were stuck in crash loops or manually stopped: both Cloudflare tunnel containers for `mcp.designflow.app` / `mcpgw.designflow.app`, and the OAuth2 Proxy for `claw.designflow.app`. The `cloudflared-coolify` systemd service had also been left disabled. Additionally, an orphan systemd `cloudflared.service` and its corresponding defunct Cloudflare tunnel were discovered and cleaned up, and structural DNS hardening was applied to Docker itself.

## Timeline

| Time (EDT) | Event |
|------------|-------|
| 2026-05-20 ~12:14 | oauth2-proxy last healthy log entry before DNS disruption |
| 2026-05-21 00:33 | oauth2-proxy begins restart loop (OIDC discovery failing: `lookup auth.designflow.app on 127.0.0.11:53: server misbehaving`) |
| 2026-05-21 00:34 | Both cloudflared Docker containers enter restart loop (same DNS error on `region1.v2.argotunnel.com`) |
| 2026-05-21 ~00:44 | Containers manually stopped; restart policy set to `no` to halt crash loops |
| 2026-05-21 21:54 | All containers restored, DNS hardening applied, orphan tunnel cleaned up |

## Root Cause

Docker's embedded DNS resolver (`127.0.0.11`) forwards to the host's systemd-resolved stub (`127.0.0.53`). When the host's upstream DNS broke (see prior post-mortem), `127.0.0.11` also failed. None of the three affected containers had custom DNS configured — they all relied on the Docker DNS relay.

The three containers fail differently on startup:
- **cloudflared containers**: require DNS to connect to `region1.v2.argotunnel.com` (Cloudflare's tunnel edge). No DNS → immediate exit(1).
- **oauth2-proxy**: requires DNS to fetch `https://auth.designflow.app/application/o/hiclaw/.well-known/openid-configuration` (OIDC discovery). No DNS → immediate exit(1). Will not start degraded.

## What Was Found During Investigation

**Orphan tunnel:** A systemd unit `cloudflared.service` existed pointing to tunnel `f6096da8-26d6-4405-8661-e5fcb4584dc0` (named `coolify-ssh` in Cloudflare). This tunnel was:
- Different from the Coolify tunnel (`1bd58f0b`), the mcp tunnel (`aa2bbb47`), and the mcpgw tunnel (`04aab164`)
- Status: `down` in Cloudflare (no active connectors)
- Not referenced in any current runbook or documentation
- Already disabled before the incident — was left over from a previous setup

**Two Docker cloudflared containers:** Both are legitimate and intentional, serving different hostnames from the same Coolify service stack (`vj5f76xet05bxwdq4utw1kho`):
- `cloudflared-vj5f76xet05bxwdq4utw1kho` → `mcp.designflow.app` → `http://devops-mcp:8765`
- `cf-cloudflared-vj5f76xet05bxwdq4utw1kho` → `mcpgw.designflow.app` → `http://contextforge:4444`

**No proxy environment variables:** Confirmed no `ALL_PROXY`, `HTTP_PROXY`, or `HTTPS_PROXY` variables in any shell profile, container, or system config. The SOCKS5 tunnel on port 1080 (`socks5-home-tunnel.service`) was already disabled and nothing depended on it.

## What Was Fixed

### Immediate recovery
1. Restored restart policy to `unless-stopped` on three containers
2. Started `cloudflared-vj5f76xet05bxwdq4utw1kho`, `cf-cloudflared-vj5f76xet05bxwdq4utw1kho`, `oauth2-proxy`
3. Enabled and started `cloudflared-coolify.service`
4. Verified all tunnel connections registered (4 connections each to Cloudflare edge)

### DNS hardening
5. Added `/etc/systemd/resolved.conf.d/fallback-dns.conf`:
   ```ini
   [Resolve]
   FallbackDNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
   ```
   Docker 27+ detects that `127.0.0.53` is a loopback address and injects the real upstream DNS servers directly into container `/etc/resolv.conf`, bypassing the stub entirely. New containers now get `8.8.8.8 8.8.4.4` directly.

6. Added `dns: [1.1.1.1, 8.8.8.8]` to `/worksp/hiclaw/oauth2-proxy/docker-compose.yml`. Recreated the container to apply immediately. This is belt-and-suspenders: even if Docker's DNS resolver behavior changes, oauth2-proxy has explicit DNS.

7. Added `"live-restore": true` to `/etc/docker/daemon.json`. Containers now survive Docker daemon restarts/upgrades without going down.

### Cleanup
8. Deleted Cloudflare tunnel `f6096da8` (`coolify-ssh`) via API — confirmed `deleted_at` null, `status: down`, safe to remove
9. Removed `/etc/systemd/system/cloudflared.service` and ran `systemctl daemon-reload`

## What Was Deliberately Left Alone

- **`socks5-home-tunnel.service`** — remains disabled. Tailscale peer at `100.110.219.31` was unreachable as of 2026-05-20 20:36. Nothing depends on it. Re-enable only after confirming Tailscale connectivity.
- **Coolify-managed cloudflared containers** — DNS hardening was NOT applied inside Coolify's service config because Coolify reconciliation would fight manual container changes. Mitigated by the resolved-level fix which applies to all containers automatically.

## Prevention

The structural fix (Docker 27+ reading resolved upstreams directly + fallback DNS in resolved) means a single DNS server going down will no longer cascade into container crashes. The prior post-mortem's netplan fix (removing Hetzner DNS dependency) prevents the primary DNS from failing in the first place.

If containers are ever found in crash loops with `127.0.0.11:53: server misbehaving`:
```bash
resolvectl status                          # check if upstream DNS is healthy
sudo systemctl restart systemd-resolved   # if resolved is broken
docker start <stopped-containers>         # after DNS is confirmed working
```

## Runbook

[`runbooks/dns-resolution-broken.md`](../runbooks/dns-resolution-broken.md) — covers the general DNS diagnosis flow.
