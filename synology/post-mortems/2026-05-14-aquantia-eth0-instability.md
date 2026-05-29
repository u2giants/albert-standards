# Post-Mortem: Aquantia eth0 Network Instability (2026-05-14)

**Status:** Root cause identified. Fix in progress.

## Summary

Both DS1621xs+ NAS units experienced periodic network instability traced to the Aquantia AQC107 10GbE NIC (eth0) and its `atlantic` driver on DSM. The instability caused ShareSync `bio error` entries and brief sync interruptions. Four new cables were tested; all failed to resolve the issue, confirming the problem is the driver/chip, not physical cabling.

## Timeline

| Date | Event |
|---|---|
| Unknown | eth1 on edgesynology2 silently went NO-CARRIER (unnoticed in DSM) |
| 2026-05-13 | bio errors in ShareSync logs on edgesynology2 identified |
| 2026-05-14 | `/proc/net/bonding/bond0` checked — eth0 on edgesynology1 showed link failures |
| 2026-05-14 | Cable swapped on edgesynology1 eth0 — link failure count incremented during swap, instability continued |
| 2026-05-14 | Four new cables tested across both units — no improvement |
| 2026-05-14 | dmesg analysis confirmed `atlantic` driver drops: `link change old 1000 new 0` on both units without physical cause |
| 2026-05-14 | Decision: migrate both bonds to eth1+eth2 (Intel i210 only), exclude eth0 permanently |

## Root cause

The Aquantia AQC107 chip with the `atlantic` driver on DSM spontaneously drops and recovers link. This is a known driver stability issue on this hardware/software combination. The drops cascade into ShareSync `bio error` entries because the sync channel loses its TCP connection during each drop.

## Fix

Remove eth0 from the bond on both units. Rebuild bond using eth1+eth2 (both Intel i210, driver `igb`, stable). Configuration via DSM Control Panel → Network → Bond edit.

## Lessons learned

1. **DSM's network UI is insufficient for bond health monitoring.** It shows the bond as "Connected" even when one slave is down. Always check `/proc/net/bonding/bond0` directly.
2. **Don't replace cables before checking dmesg.** The `atlantic` driver drop pattern in dmesg would have immediately identified the Aquantia chip as the culprit before any cables were touched.
3. **bio errors in ShareSync are always downstream of network instability**, never a ShareSync bug.
4. **eth2 should have been in the bond from day one.** The DS1621xs+ has two reliable Intel i210 ports; there was no reason to use the unreliable Aquantia port for bonding.

## Prevention

Both units now (or will) run bond on eth1+eth2 only. eth0 is excluded. This configuration should be checked at the start of any future network troubleshooting session before any other investigation.
