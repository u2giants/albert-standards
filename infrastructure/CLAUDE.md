# Server Infrastructure — designflow.app VPS

Designflow PLM itself is not hosted by this VPS/Coolify stack. The Designflow
application runtime is Google Cloud Run with Cloud Build/Artifact Registry and a
public BFF in front of private services. For Designflow app infrastructure,
deployment, auth, and secrets rules, read
[`designflow-cloud-run.md`](designflow-cloud-run.md). This file remains the
reference for the Hetzner VPS, Coolify, Traefik, Cloudflare tunnels, and related
server runbooks.

## Overview

This VPS (178.156.180.212) runs Coolify, which manages all deployed applications.
All apps are reverse-proxied through Traefik (`coolify-proxy` container).
The domain `designflow.app` uses Cloudflare for DNS, with most subdomains pointing directly at this server and a few using Cloudflare Tunnels.

---

## PopDAM / PopSG production app

PopDAM and PopSG are served from the same Coolify-managed frontend container.

| Field | Value |
|---|---|
| Public hostnames | `dam.designflow.app`, `sg.designflow.app` |
| Coolify app UUID | `qxj8a0j3tpa9lq4q5rs6pezy` |
| Frontend image | `ghcr.io/u2giants/popdam-frontend:latest` |
| Traefik Docker service | `https-0-qxj8a0j3tpa9lq4q5rs6pezy@docker` |
| PopDAM repo | `u2giants/popdam3` |
| Supabase project | `qsllyeztdwjgirsysgai` (Virginia, current production) |
| Old Supabase project | `ryltkzzernhwnojzouyb` (Ohio, decommissioned/frozen; do not use for live data) |
| Railway worker | `apps/worker/`, auto-deploys from every push to `main` |

### Routing model

- `dam.designflow.app` is routed by Coolify's Docker labels on the
  `popdam-frontend` app.
- `sg.designflow.app` is routed by the Traefik file provider at
  `/data/coolify/proxy/dynamic/popdam-sg.yml`, using the `@docker` service
  reference above.
- The frontend container is a static nginx server. It has no runtime env vars;
  Supabase URL/anon key are baked into the bundle by source code.
- The app switches PopDAM/PopSG mode by hostname at runtime. Do not split this
  into two Coolify apps unless the PopDAM repo explicitly changes that design.

### Deployment ownership

Normal frontend path:

1. Push to `main` in `u2giants/popdam3`.
2. GitHub Actions `publish-frontend.yml` builds the Vite app and Docker image.
3. The workflow pushes `latest`, `sha-<short-sha>`, and `<short-sha>` tags to
   GHCR.
4. The workflow calls the Coolify deploy API.
5. Coolify pulls `ghcr.io/u2giants/popdam-frontend:latest` and recreates the
   managed container.

Do not use SSH or manual `docker run` for routine PopDAM frontend deploys.
Coolify owns the app container, labels, health checks, and restart policy.

### PopDAM-specific deployment traps

- GitHub's green `popdam / production` deployment badge can be Railway worker
  status, not frontend status. Verify the `Publish Frontend Image` workflow,
  GHCR tags, Coolify deployment record, and live headers/assets for frontend
  freshness.
- The frontend GHCR package is user-scoped. If GitHub Actions fails with
  `permission_denied: write_package`, check the package's "Manage Actions
  access" for `u2giants/popdam3` or the repo secret `GHCR_PAT`.
- Coolify pulls private GHCR images using the VPS Docker credential file at
  `/root/.docker/config.json`. If Coolify logs `unauthorized` during image
  pull, refresh the VPS GHCR login without recording token values.
- If both public domains return 502 while the app container is healthy, check
  `docker logs coolify-proxy` for Docker provider errors. A stale
  `/var/run/docker.sock` bind mount inside `coolify-proxy` has broken Docker
  provider discovery before; restarting only `coolify-proxy` refreshed it.
- The nginx config must listen on IPv4 and IPv6 (`listen 80; listen [::]:80;`)
  because Coolify's health check may resolve `localhost` to `::1`.

### Break-glass rule

If GHCR/Coolify publishing is blocked during an incident and the shell is
already on the VPS, a local image rebuild may be used only as break glass:
build/tag `ghcr.io/u2giants/popdam-frontend:latest`, then recreate only the
Coolify-managed service from
`/data/coolify/applications/qxj8a0j3tpa9lq4q5rs6pezy/docker-compose.yaml`.
Afterward, document the action in `u2giants/popdam3` and repair the normal
GitHub Actions -> GHCR -> Coolify path.

---

## How traffic reaches the server

There are two different paths depending on the subdomain:

### Direct subdomains (DNS only, no Cloudflare proxy)
`crm`, `mon`, `manus`, `vnc`, `nas-mcp`, etc. point directly to `178.156.180.212`.
Traffic hits Traefik on ports 80/443. Traefik routes by hostname and issues Let's Encrypt certs via **HTTP-01 challenge**.

### Cloudflare Tunnel subdomains

There are three active Cloudflare Tunnels. Each is independent — do not confuse them or consolidate them.

#### Tunnel 1 — `coolify.designflow.app` (systemd-managed)

Service: `cloudflared-coolify.service`
Tunnel ID: `1bd58f0b-1eb1-4cc7-8d9a-74790db459b3`
Token: stored in `/etc/cloudflared/coolify-tunnel.env` (root-only, not in source control)
Owner: manually managed systemd unit — **not a Coolify container**

The tunnel routes by path:

| Path prefix          | Routes to               | Service                     |
|----------------------|-------------------------|-----------------------------|
| `^/app(/|$)`         | `http://127.0.0.1:6001` | Soketi (realtime WebSocket) |
| `^/terminal/ws(/|$)` | `http://127.0.0.1:6002` | Coolify terminal WebSocket  |
| everything else      | `http://127.0.0.1:8000` | Coolify PHP app             |

**Critical:** Path rules are regexes and MUST be anchored (`^/app(/|$)` not `/app`). An unanchored rule matches asset filenames like `app-C9Z-drIo.js` and misroutes them to Soketi (returns 404). If the tunnel config is ever recreated via the Zero Trust dashboard, reapply the anchored regexes via the API (see runbook below).

Tunnel config is stored remotely in Cloudflare (no local config file).
Inspect live config: `curl -s http://127.0.0.1:20241/config | python3 -m json.tool`
Restart after config changes: `sudo systemctl restart cloudflared-coolify`

#### Tunnel 2 — `mcp.designflow.app` (Coolify-managed Docker container)

Container: `cloudflared-vj5f76xet05bxwdq4utw1kho`
Coolify service UUID: `vj5f76xet05bxwdq4utw1kho`
Tunnel ID: `aa2bbb47-3907-485d-a0fa-61f57af478d8`
Token env var: `TUNNEL_TOKEN` / `CLOUDFLARE_TUNNEL_TOKEN`
Routes to: `http://devops-mcp:8765`

**Owner: Coolify.** Do not manually start/stop/recreate this container — Coolify owns its lifecycle. Only restart via Coolify UI or `docker restart` for emergencies.

#### Tunnel 3 — `mcpgw.designflow.app` (Coolify-managed Docker container)

Container: `cf-cloudflared-vj5f76xet05bxwdq4utw1kho`
Coolify service UUID: `vj5f76xet05bxwdq4utw1kho` (same service stack as Tunnel 2)
Tunnel ID: `04aab164-ce2c-424c-98f1-a3c8402e5c06`
Token env var: `CF_GW_TUNNEL_TOKEN`
Routes to: `http://contextforge:4444`

**Owner: Coolify.** Same lifecycle rules as Tunnel 2.

---

## Traefik (reverse proxy)

Container: `coolify-proxy`
Config files: `/data/coolify/proxy/`
- `docker-compose.yml` — Traefik startup config (ports, resolvers, env vars). **Do not let Coolify overwrite this without re-applying the changes below.**
- `dynamic/coolify.yaml` — Auto-generated by Coolify. Contains routing rules for the Coolify dashboard and its realtime/terminal WebSocket routes.
- `acme.json` — Let's Encrypt certs issued via HTTP-01 (direct subdomains)
- `acme-dns.json` — Let's Encrypt certs issued via DNS-01 (tunnel subdomains like `coolify.designflow.app`)

### Two cert resolvers

**`letsencrypt`** (HTTP-01): Used for direct subdomains. Works because Let's Encrypt can reach port 80 on the server directly.

**`letsencrypt-dns`** (DNS-01 via Cloudflare): Used for `coolify.designflow.app` and any other subdomain behind a Cloudflare Tunnel. Works by creating a temporary DNS TXT record via the Cloudflare API. Requires `CF_DNS_API_TOKEN` env var on the Traefik container.

The Cloudflare API token is set in `/data/coolify/proxy/docker-compose.yml` under `environment.CF_DNS_API_TOKEN`. It needs these Cloudflare permissions: Zone Settings Edit, Zone DNS Edit, Cloudflare Tunnel Edit, Cache Purge.

### DNS fix for Traefik container

The host uses `systemd-resolved` (`127.0.0.53`) which is unreachable inside Docker containers. The Traefik `docker-compose.yml` explicitly sets `dns: [1.1.1.1, 8.8.8.8]` to work around this. If you ever recreate the Traefik container and DNS stops working inside it, this is why.

---

## Key ports (host-level)

| Port  | Service                        |
|-------|--------------------------------|
| 80    | Traefik HTTP                   |
| 443   | Traefik HTTPS                  |
| 6001  | Soketi (Coolify realtime)      |
| 6002  | Soketi (Coolify terminal WS)   |
| 8000  | Coolify PHP app (via Docker)   |
| 8080  | Traefik dashboard (internal)   |
| 20241 | cloudflared metrics/config API |

---

## Cloudflare settings that matter

- **WebSockets**: Must be ON (Network → WebSockets). Required for the Coolify realtime service to work through Cloudflare.
- DNS records for tunnel subdomains are CNAMEs to `*.cfargotunnel.com` with proxy enabled.
- DNS records for direct subdomains are A records with proxy **disabled** (grey cloud).

---

## If Coolify assets (CSS/JS) return 404 or the login page renders unstyled

1. Check the tunnel path routing: `curl -s http://127.0.0.1:20241/config | python3 -m json.tool`
   — Confirm `/app` and `/terminal/ws` rules are anchored (`^/app(/|$)`, not `/app`).
2. If misrouted, update via API:
   ```bash
   curl -X PUT -H "Authorization: Bearer $CF_TOKEN" \
     -H "Content-Type: application/json" \
     "https://api.cloudflare.com/client/v4/accounts/8303d11002766bf1cc36bf2f07ba6f20/cfd_tunnel/1bd58f0b-1eb1-4cc7-8d9a-74790db459b3/configurations" \
     -d '{"config":{"ingress":[{"hostname":"coolify.designflow.app","path":"^/app(/|$)","service":"http://127.0.0.1:6001"},{"hostname":"coolify.designflow.app","path":"^/terminal/ws(/|$)","service":"http://127.0.0.1:6002"},{"hostname":"coolify.designflow.app","service":"http://127.0.0.1:8000"},{"service":"http_status:404"}],"warp-routing":{"enabled":false}}}'
   sudo systemctl restart cloudflared-coolify.service
   ```
3. Purge the Cloudflare cache for the affected paths (Soketi returns `cache-control: max-age=14400` on 404s, so bad responses get cached for 4 hours):
   ```bash
   ZONE_ID=921eb133a3f7d5802780445b283f84ce
   curl -X POST -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
     "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache" \
     -d '{"prefixes":["coolify.designflow.app/build/"]}'
   ```
4. Users may still see unstyled pages after cache purge if their browser cached the 404s — hard refresh (`Cmd+Shift+R`) or incognito window fixes it.

## If the Coolify "Cannot connect to real-time service" warning appears

1. Check WebSockets is enabled in Cloudflare dashboard (Network → WebSockets).
2. Check the tunnel path routing: `curl -s http://127.0.0.1:20241/config | python3 -m json.tool`
   — `/app` must route to `127.0.0.1:6001`, not to the Coolify app.
3. Check the `coolify-realtime` container is healthy: `docker ps | grep realtime`

---

---

## Docker DNS architecture — why containers need direct DNS

### The problem

The host's `/etc/resolv.conf` points to `127.0.0.53` (systemd-resolved stub). That address is on the host's loopback and unreachable from inside Docker containers. Docker has a built-in DNS relay at `127.0.0.11` that forwards to the host's resolv.conf, but when the host's primary DNS is broken or systemd-resolved itself is unhealthy, `127.0.0.11` also fails — and every container that needs a DNS lookup crashes simultaneously.

This is what happened in the May 2026 incident: host DNS broke → Docker DNS relay broke → all Cloudflare tunnel containers and oauth2-proxy crashed in a cascade.

### The fix (four layers, fully applied as of 2026-05-26)

1. **systemd-resolved global DNS** (`/etc/systemd/resolved.conf.d/dns.conf`):
   ```ini
   [Resolve]
   DNS=1.1.1.1 8.8.8.8 9.9.9.9
   FallbackDNS=1.0.0.1 8.8.4.4 149.112.112.112
   ```
   Sets Cloudflare/Google/Quad9 as global resolvers regardless of what DHCP injects. Applied 2026-05-20.

2. **systemd-resolved fallback DNS** (`/etc/systemd/resolved.conf.d/fallback-dns.conf`):
   ```
   FallbackDNS=1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4
   ```
   Belt-and-suspenders fallback. Docker 27+ detects that `127.0.0.53` is a loopback address and reads the real upstream servers from resolved, injecting them as `ExtServers` in container DNS config. As of 2026-05-26, containers show `ExtServers: [host(127.0.0.53)]` and resolve via systemd-resolved → 1.1.1.1. Applied 2026-05-21.

3. **oauth2-proxy compose file** (`/worksp/hiclaw/oauth2-proxy/docker-compose.yml`):
   ```yaml
   dns:
     - 1.1.1.1
     - 8.8.8.8
   ```
   Per-container DNS bypasses the Docker relay entirely. Critical because oauth2-proxy must resolve `auth.designflow.app` at every startup to perform OIDC discovery — if this fails, the container exits immediately with code 1. Applied 2026-05-21.

4. **Tailscale DNS disabled** (`tailscale set --accept-dns=false`, applied 2026-05-26):
   When Tailscale DNS is enabled, it intercepts all system DNS and routes it through its internal resolver at `100.100.100.100` (Tailscale's magic DNS address). Docker reads this as the `ExtServer` for all containers — overriding the fallback-dns fix from layer 2. After the May 2026 DNS incident, Tailscale's resolver had no upstream configured and was returning SERVFAIL for every query at 130+ errors/second. This broke all container DNS silently for 6 days despite the May 21 remediation. Disabling Tailscale DNS management means systemd-resolved handles DNS normally, and Docker reads the real upstream (1.1.1.1) instead of `100.100.100.100`. All containers were restarted on 2026-05-26 to flush the stale ExtServer config.
   **MagicDNS is intentionally disabled on this server.** Tailscale peer hostnames (`.tail769aaf.ts.net`) are not resolvable by name. Use Tailscale IPs directly.

**Docker daemon live-restore** (`/etc/docker/daemon.json`): `"live-restore": true` — containers survive Docker daemon restarts/upgrades without going down. Applied 2026-05-21.

### What to watch for

If you ever see this error in any container log:
```
lookup <hostname> on 127.0.0.11:53: server misbehaving
```
The Docker DNS relay is broken. First check host DNS:
```bash
resolvectl status
dig api.anthropic.com @1.1.1.1   # bypass local resolver to confirm network is up
```
If the network is up but systemd-resolved is broken: `sudo systemctl restart systemd-resolved`

Also check Tailscale DNS status — if someone re-enabled it, it will be intercepting DNS again:
```bash
tailscale dns status | head -3
# Should say: "Tailscale DNS: disabled."
# If it says "enabled", run: tailscale set --accept-dns=false && docker restart $(docker ps -q)
```

---

## OAuth2 Proxy

**Container:** `oauth2-proxy`
**Source:** `/worksp/hiclaw/oauth2-proxy/docker-compose.yml` + `.env`
**Owner:** manually managed docker-compose (not Coolify)

Provides authentication in front of `claw.designflow.app` and `control.claw.designflow.app`. Configured as a forward-auth middleware — Traefik sends every request through it before routing to the app. The proxy validates the user's session and sets `X-Auth-Request-*` headers.

**OIDC provider:** Authentik at `https://auth.designflow.app/application/o/hiclaw/`
**Redirect URL:** `https://control.claw.designflow.app/oauth2/callback`
**Cookie domain:** `.claw.designflow.app` (covers all subdomains)
**Allowed users:** `/worksp/hiclaw/oauth2-proxy/allowed-emails.txt` (one email per line, mounted read-only)
**Upstream:** `static://202` — the proxy itself doesn't serve the app, it just returns 202 to tell Traefik the user is authenticated

**Important:** oauth2-proxy performs OIDC discovery on every startup by fetching:
`https://auth.designflow.app/application/o/hiclaw/.well-known/openid-configuration`
If this fetch fails (network issue, DNS issue, Authentik down), the container exits with code 1. It will not start degraded. Fix the network/DNS first, then start the container.

To add an allowed user: add their email to `/worksp/hiclaw/oauth2-proxy/allowed-emails.txt` — the container watches the file and reloads automatically without restart.

---

## SOCKS5 tunnel (port 1080)

**Service:** `socks5-home-tunnel.service` (systemd, currently **disabled**)
**What it does:** SSH dynamic forward (`-D 0.0.0.0:1080`) to a home machine via Tailscale
**Target:** Tailscale peer at `100.110.219.31`
**Run as:** user `ai`

This is an SSH SOCKS5 proxy used to route traffic through a home machine. Nothing on the server currently depends on it. It was disabled in May 2026 when the Tailscale peer became unreachable.

**To re-enable** (once Tailscale connectivity to the home machine is confirmed):
```bash
ssh home-tailscale -p 22   # verify the target is reachable first
sudo systemctl enable --now socks5-home-tunnel
```

**No apps should route through this proxy.** There are no `ALL_PROXY`, `HTTP_PROXY`, or `HTTPS_PROXY` env vars set anywhere on the server. If you find any, remove them — they are either leftover from debugging or misconfiguration.

---

## Cloudflare account IDs

Cloudflare Account ID: `8303d11002766bf1cc36bf2f07ba6f20`
Zone ID (designflow.app): `921eb133a3f7d5802780445b283f84ce`
CF API token: stored as `CF_DNS_API_TOKEN` env var on the `coolify-proxy` (Traefik) container. Retrieve with: `docker exec coolify-proxy printenv CF_DNS_API_TOKEN`

Permissions required on that token: Zone Settings Edit, Zone DNS Edit, Cloudflare Tunnel Edit, Cache Purge.

---

## If Traefik stops issuing certs for coolify.designflow.app

The `letsencrypt-dns` resolver uses the Cloudflare API. Check:
1. The API token is still valid and has DNS Edit + Zone Settings Edit permissions.
2. The `dynamic/coolify.yaml` routes use `certresolver: letsencrypt-dns` (not `letsencrypt`). Coolify may revert this to `letsencrypt` if you click "Restart Proxy" in the UI — reapply with: `sudo sed -i 's/certresolver: letsencrypt$/certresolver: letsencrypt-dns/' /data/coolify/proxy/dynamic/coolify.yaml`

---

## If all Docker containers suddenly lose DNS (mass crash)

Symptom: Many containers simultaneously logging `server misbehaving` or `lookup <host> on 127.0.0.11:53`.

1. Check whether the host network is up: `ping -c2 1.1.1.1` (IP, not hostname)
2. Check systemd-resolved: `resolvectl status` — look for "DNS Servers" showing real IPs
3. If resolved is broken: `sudo systemctl restart systemd-resolved`
4. **Check Tailscale DNS is still disabled**: `tailscale dns status | head -3` — must say "Tailscale DNS: disabled." If not: `tailscale set --accept-dns=false`
5. Containers that exited will need to be restarted:
   ```bash
   # restore restart policies if they were changed to 'no' during troubleshooting:
   docker update --restart=unless-stopped <container-id> ...
   docker start <container-id> ...
   ```
6. For `cloudflared-coolify.service` (systemd, not Docker): `sudo systemctl start cloudflared-coolify`
7. Do NOT blindly re-enable all disabled systemd services — only `cloudflared-coolify`. The `socks5-home-tunnel` is intentionally disabled.

---

## Things you must not do

**Do not run two cloudflared mechanisms for the same tunnel.**
There used to be an orphan systemd unit `cloudflared.service` running tunnel `f6096da8` (`coolify-ssh`) alongside the real `cloudflared-coolify.service`. It was removed in May 2026. If you see a plain `cloudflared.service` reappear, it is stale — disable and delete it.

**Do not let Coolify "Restart Proxy" without checking `dynamic/coolify.yaml` afterward.**
Coolify overwrites `certresolver: letsencrypt-dns` back to `letsencrypt`, which breaks cert issuance for `coolify.designflow.app`. Always re-run the sed command above after any Coolify proxy restart.

**Do not manually start/stop the Coolify-managed cloudflared containers.**
`cloudflared-vj5f76xet05bxwdq4utw1kho` and `cf-cloudflared-vj5f76xet05bxwdq4utw1kho` are owned by Coolify service `vj5f76xet05bxwdq4utw1kho`. If Coolify reconciles, it will fight any manual changes. Emergency restart: `docker restart <container>` is fine. Changing config: do it in Coolify UI.

**Do not set HTTP_PROXY / HTTPS_PROXY / ALL_PROXY environment variables** pointing to `socks5://127.0.0.1:1080` or any other proxy. The SOCKS tunnel is disabled and nothing depends on it. Proxy env vars will silently break any tool or container that inherits them.

**Do not add a `dns` key to daemon.json pointing at `127.0.0.53`.**
That's the systemd-resolved stub and unreachable from containers. The daemon.json intentionally has no `dns` key; Docker 27+ reads the real upstream from resolved automatically. The fallback DNS is configured in `/etc/systemd/resolved.conf.d/fallback-dns.conf`.

**Do not re-enable Tailscale DNS** (`tailscale set --accept-dns=true`).
When Tailscale DNS is enabled, all system DNS is routed through Tailscale's internal resolver at `100.100.100.100`, and Docker injects `100.100.100.100` as the `ExtServer` for every container. If Tailscale's resolver loses its upstream configuration — which happened after the May 2026 DNS incident and is not self-healing — all container DNS breaks silently with SERVFAIL. This exact failure mode caused 6 days of broken container DNS (2026-05-20 to 2026-05-26) despite the May 21 remediation work. Tailscale DNS is deliberately disabled on this server. MagicDNS hostname resolution is the only thing lost; use Tailscale IPs directly.

---

## Incident record

See [`post-mortems/`](post-mortems/) in this repository for full incident records.

Relevant post-mortems for this server:
- [2026-05-20: DNS cascade failure](post-mortems/2026-05-20-dns-cascade.md) — Hetzner DNS servers unreachable; root DNS fix applied
- [2026-05-21: Cloudflared/OAuth2 container recovery](post-mortems/2026-05-21-cloudflared-oauth2-recovery.md) — cascading Docker DNS failure; tunnels and auth proxy restored; Docker hardened
