# Runbook: ShareSync bio errors

## Log location

```bash
/volume1/@synologydrive/log/syncfolder.log
/volume1/@synologydrive/log/syncfolder.log_0   # rotated previous log
```

## What bio errors look like

```
2026-05-14T11:24:08 (23677:11520) [ERROR] channel.cpp(911): bio error is set to -1  (read_len: 0, buf_len: 1).
2026-05-14T11:24:08 (23677:11520) [ERROR] channel.cpp(911): bio error is set to -2  (read_len: 0, buf_len: 1).
2026-05-14T11:24:08 (23677:11520) [ERROR] channel.cpp(911): bio error is set to -3  (read_len: 0, buf_len: 2).
```

Error codes: -1 = connection reset, -2 = connection lost, -3 = timeout/read error. All mean the same thing: the sync channel dropped.

## Root cause

bio errors are always caused by a network interruption between the two NAS units. They are **not** a ShareSync software bug and do not indicate data corruption or sync failure — ShareSync recovers automatically once the network is stable.

## Diagnosis

1. Check when bio errors occurred:
   ```bash
   grep "bio error" /volume1/@synologydrive/log/syncfolder.log | tail -20
   ```

2. Correlate with bond link failures:
   ```bash
   cat /proc/net/bonding/bond0 | grep "Link Failure Count"
   ```
   High link failure counts = network was unstable = explains the bio errors.

3. Check dmesg for `atlantic` driver drops around the same timestamps:
   ```bash
   dmesg | grep "atlantic\|bond.*down" | tail -20
   ```

## Resolution

Fix the underlying network instability. See `runbooks/network-bond-troubleshooting.md`.

Once the bond is stable (eth1+eth2 only, no Aquantia eth0), bio errors should stop entirely.

## Monitoring after a fix

Check the log the morning after any network change:
```bash
grep "bio error" /volume1/@synologydrive/log/syncfolder.log | grep "$(date +%Y-%m-%d)"
```
No output = clean day.
