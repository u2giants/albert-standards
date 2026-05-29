# Runbook: Network Bond Troubleshooting

## Quick diagnosis

```bash
cat /proc/net/bonding/bond0
```

Look for:
- `MII Status: down` on any slave → that slave has no physical link
- `Link Failure Count` incrementing → link is dropping and recovering
- `Currently Active Slave` → which port is carrying traffic

```bash
cat /sys/class/net/eth0/operstate    # up / down
cat /sys/class/net/eth0/carrier      # 1 = signal, 0 = no signal
cat /sys/class/net/eth0/carrier_changes   # total state changes since boot
```

## Isolating cable vs NIC vs switch port vs driver

Work through in this order:

1. **Check dmesg for driver messages**
   ```bash
   dmesg | grep -i "atlantic\|igb.*eth\|link change\|bond.*down\|bond.*up" | tail -30
   ```
   - `atlantic: link change old 1000 new 0` with no physical event → Aquantia driver bug, not cable
   - `igb: eth1 NIC Link is Down` → check cable and switch port

2. **Try a new cable** — if dmesg shows `atlantic` drops without any physical disturbance, new cables won't help (driver issue)

3. **Try a different switch port** — move the cable to a known-good port

4. **Short patch cable test** — plug a 1m patch cable directly from the NAS port into the nearest switch port, bypassing any long cable run. If the port comes up, the long cable run is faulty.

5. **If all above fail** — the NIC port itself may be dead

## Known issue: Aquantia eth0 on DS1621xs+

**Do not waste time on eth0 cable replacement.** The Aquantia AQC107 `atlantic` driver drops link spontaneously regardless of cable quality. The fix is to exclude eth0 from the bond and use eth1+eth2 only. See `hardware/nic-reference.md`.

## Reconfiguring the bond in DSM

1. Control Panel → Network → Network Interface
2. Select Bond 1 → Edit
3. Remove eth0, add eth2 (so bond = eth1 + eth2)
4. Save — DSM keeps the bond IP on bond0, no connectivity loss if at least one slave is up
5. Physically move the cable from eth0 to eth2

**Do not assign a static IP to eth2 before adding it to the bond.** If eth2 already has a DHCP IP, that's fine — DSM strips it automatically when eth2 becomes a bond slave.

## Checking bond health after changes

```bash
cat /proc/net/bonding/bond0
```

Healthy state (both slaves up):
```
MII Status: up
Slave Interface: eth1
MII Status: up
Speed: 1000 Mbps
Link Failure Count: N   ← only increments if link drops
Slave Interface: eth2
MII Status: up
Speed: 1000 Mbps
Link Failure Count: N
```
