# Runbook: DNS Resolution Broken on Hetzner Server

## Symptoms

- Claude CLI fails: `Could not resolve host: api.anthropic.com`
- `curl https://any-domain.com` fails with "Could not resolve host"
- Apps that call external APIs stop working
- SSH still works (client already has the IP or uses Tailscale)
- **Pinging by IP succeeds** — this is the tell that it's DNS, not connectivity:

```bash
ping 1.1.1.1     # works
ping 8.8.8.8     # works
getent ahosts api.anthropic.com  # fails
```

## Likely Cause

The server uses `systemd-resolved` for DNS. On Hetzner, DHCP or cloud-init can inject Hetzner's IPv6 DNS servers as link-specific resolvers on `eth0`. If those servers become unreachable (IPv6 flap, Hetzner DNS outage), all hostname resolution fails — even if public DNS servers are configured globally.

A second cause: Tailscale DNS being re-enabled. When `tailscale set --accept-dns=true` is run, Tailscale intercepts all system DNS and routes it through its internal resolver at `100.100.100.100`. If that resolver has no upstream configured (a persistent bug triggered by the May 2026 incident), all DNS returns SERVFAIL. Check this first — it's silent and easy to miss.

## Diagnosis

```bash
# Confirm it's DNS not connectivity
ping -c2 1.1.1.1
getent ahosts google.com

# Check Tailscale DNS status — must be disabled
tailscale dns status | head -3
# Should say: "Tailscale DNS: disabled."
# If enabled, that is almost certainly the cause — see fix below.

# Check what DNS servers are actually being used
resolvectl status

# Look for eth0 showing Hetzner DNS (2a01:4ff:...) instead of public DNS
# Look for Global showing public DNS but eth0 overriding it
```

## Fix

### If Tailscale DNS is enabled (most likely cause post-May 2026)

```bash
tailscale set --accept-dns=false
# Then restart all containers to flush the stale 100.100.100.100 ExtServer config:
docker restart $(docker ps -q)
```

Verify:
```bash
tailscale dns status | head -3   # must say "Tailscale DNS: disabled."
docker exec <any-container> nslookup google.com   # must succeed
```

### If Tailscale DNS is already disabled

```bash
sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches
resolvectl status
getent ahosts google.com
```

If that doesn't work, force the interface DNS:

```bash
IFACE="$(ip route | awk '/default/ {print $5; exit}')"
sudo resolvectl dns "$IFACE" 1.1.1.1 8.8.8.8 9.9.9.9
sudo resolvectl domain "$IFACE" '~.'
sudo resolvectl flush-caches
getent ahosts google.com
```

### Persistent fix (if the above reverts after reboot)

Check whether the permanent config files are in place:

```bash
cat /etc/systemd/resolved.conf.d/dns.conf
cat /etc/netplan/99-custom-dns.yaml
systemctl list-timers | grep dns-watchdog
tailscale dns status | head -3   # should say "disabled"
```

If any are missing, see the post-mortem for the full setup:
[`post-mortems/2026-05-20-dns-cascade.md`](../post-mortems/2026-05-20-dns-cascade.md)

## Verification

```bash
resolvectl status                        # eth0 and Global should show 1.1.1.1 8.8.8.8 9.9.9.9
tailscale dns status | head -3           # must say "Tailscale DNS: disabled."
getent ahosts api.anthropic.com          # should return IPs
getent ahosts google.com                 # should return IPs
curl -4Iv --connect-timeout 10 https://api.anthropic.com/v1/messages 2>&1 | grep -E "Connected|resolve"
```

## What should prevent this from recurring

As of 2026-05-26, the server has four permanent safeguards:

1. `/etc/systemd/resolved.conf.d/dns.conf` — sets Cloudflare/Google/Quad9 as global DNS and fallback, independent of what DHCP injects (applied 2026-05-20)
2. `/etc/netplan/99-custom-dns.yaml` — prevents Hetzner's DHCP from overwriting `eth0` DNS on reboot (applied 2026-05-20)
3. `dns-watchdog.timer` — runs every 5 minutes, restarts `systemd-resolved` if `google.com` and `cloudflare.com` both fail to resolve, logs to journal (applied 2026-05-20)
4. `tailscale set --accept-dns=false` — Tailscale DNS is disabled so it cannot override system DNS or inject `100.100.100.100` into Docker container DNS configuration. Do not re-enable. (applied 2026-05-26)

## Notes

- **Do not remove Tailscale before diagnosing.** Tailscale uses its own `tailscale0` interface and doesn't share DNS config with `eth0`. It is never the cause of this failure mode — but Tailscale *DNS management* (separate from Tailscale networking) can be. Check `tailscale dns status`, not `tailscale status`.
- The watchdog logs to `journalctl -u dns-watchdog.service` — check there if you suspect it fired.
- `DNSSEC` and `DNSOverTLS` are intentionally disabled in the resolved config for reliability. Re-enabling them is a security improvement but adds failure modes.
- MagicDNS (`.tail769aaf.ts.net` hostname resolution) is intentionally disabled. Use Tailscale IPs directly.
