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

## Diagnosis

```bash
# Confirm it's DNS not connectivity
ping -c2 1.1.1.1
getent ahosts google.com

# Check what DNS servers are actually being used
resolvectl status

# Look for eth0 showing Hetzner DNS (2a01:4ff:...) instead of public DNS
# Look for Global showing public DNS but eth0 overriding it
```

## Fix

### Quick fix (current session only)

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
```

If any are missing, see the post-mortem for the full setup:
[`post-mortems/2026-05-20-dns-cascade.md`](../post-mortems/2026-05-20-dns-cascade.md)

## Verification

```bash
resolvectl status                        # eth0 and Global should show 1.1.1.1 8.8.8.8 9.9.9.9
getent ahosts api.anthropic.com          # should return IPs
getent ahosts google.com                 # should return IPs
curl -4Iv --connect-timeout 10 https://api.anthropic.com/v1/messages 2>&1 | grep -E "Connected|resolve"
```

## What should prevent this from recurring

As of 2026-05-20, the server has three permanent safeguards:

1. `/etc/systemd/resolved.conf.d/dns.conf` — sets Cloudflare/Google/Quad9 as global DNS and fallback, independent of what DHCP injects
2. `/etc/netplan/99-custom-dns.yaml` — prevents Hetzner's DHCP from overwriting `eth0` DNS on reboot
3. `dns-watchdog.timer` — runs every 5 minutes, restarts `systemd-resolved` if `google.com` and `cloudflare.com` both fail to resolve, logs to journal

## Notes

- **Do not remove Tailscale before diagnosing.** Tailscale uses its own `tailscale0` interface and doesn't share DNS config with `eth0`. It is never the cause of this failure mode.
- The watchdog logs to `journalctl -u dns-watchdog.service` — check there if you suspect it fired.
- `DNSSEC` and `DNSOverTLS` are intentionally disabled in the resolved config for reliability. Re-enabling them is a security improvement but adds failure modes.
