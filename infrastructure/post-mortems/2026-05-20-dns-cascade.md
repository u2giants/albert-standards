# Post-Mortem: DNS Cascade Failure — 2026-05-20

## Summary

A problem in the hiclaw application cascaded to the Twenty CRM application, which ultimately caused DNS resolution to break on the Hetzner server. External API calls (including Claude CLI) failed completely. SSH remained functional because the connecting client already had the server's IP. The root cause was Hetzner's IPv6 DNS servers being used as the primary resolver on `eth0` via DHCP/cloud-init injection — when those became unreachable, all hostname resolution stopped.

## Timeline

| Time | Event |
|------|-------|
| Unknown | hiclaw application encounters a problem |
| Unknown | Problem cascades to Twenty CRM |
| Unknown | Cascading failures disrupt system networking/DNS |
| ~2026-05-20 evening | DNS resolution confirmed broken; Claude CLI, curl failing on all hostnames |
| 2026-05-20 ~21:00 EDT | Temporary fix applied in prior session (public DNS forced via `resolvectl`) |
| 2026-05-20 21:22 EDT | Permanent fix applied and verified |

## Root Cause

`/etc/netplan/50-cloud-init.yaml` had Hetzner's IPv6 DNS servers (`2a01:4ff:ff00::add:1`, `2a01:4ff:ff00::add:2`) hardcoded as the nameservers for `eth0`. When the cascade disrupted the path to those servers, `systemd-resolved` had no working resolver. The global fallback DNS in `/etc/systemd/resolved.conf.d/dns.conf` was insufficient because `eth0`'s link-specific DNS (with a `~.` catch-all routing domain) took priority.

## Contributing Factors

- Cloud-init Netplan config used Hetzner DNS with no fallback to public resolvers
- No DNS health monitoring to detect the failure automatically
- SSH working correctly masked the severity — the server appeared reachable when it was not fully functional

## What Was Fixed

Three permanent safeguards now in place:

### 1. `/etc/systemd/resolved.conf.d/dns.conf`
Sets Cloudflare, Google, and Quad9 as global DNS with full fallbacks. Ensures a working baseline regardless of what link-specific DNS is configured.

```ini
[Resolve]
DNS=1.1.1.1 8.8.8.8 9.9.9.9
FallbackDNS=1.0.0.1 8.8.4.4 149.112.112.112
DNSSEC=no
DNSOverTLS=no
DNSStubListener=yes
Cache=yes
```

### 2. `/etc/netplan/99-custom-dns.yaml`
Overrides the cloud-init Netplan config so DHCP no longer injects Hetzner DNS on `eth0`. Public DNS is used directly and survives reboots.

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
      dhcp6: false
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
          - 9.9.9.9
```

### 3. `dns-watchdog.timer` (systemd)
Runs every 5 minutes. If both `google.com` and `cloudflare.com` fail to resolve, restarts `systemd-resolved` and logs the event. Script at `/usr/local/sbin/fix-dns-if-broken.sh`.

```bash
# Check watchdog logs
journalctl -u dns-watchdog.service --no-pager -n 50

# Check timer schedule
systemctl list-timers | grep dns-watchdog
```

## What Was Deliberately Not Changed

- **Tailscale** — uses `tailscale0` independently; none of the above affects it
- **Hetzner private networking** — if internal Hetzner hostnames are ever needed, the fallback config would need revisiting
- **DNSSEC / DNSOverTLS** — kept disabled for reliability; re-enabling is a future security improvement

## Open Questions

- What specifically in the hiclaw→Twenty cascade disrupted DNS? This wasn't fully diagnosed. Worth investigating whether those apps have any capability to modify `/etc/resolv.conf`, restart networking services, or alter systemd-resolved configuration.
- Is there an application-level circuit breaker that would have contained the hiclaw failure before it reached Twenty?

## Runbook

For diagnosis and quick-fix steps if this recurs:
[`runbooks/dns-resolution-broken.md`](../runbooks/dns-resolution-broken.md)
