# NIC Reference — DS1621xs+

Both units (edgesynology1, edgesynology2) are identical DS1621xs+ hardware with the same NIC configuration.

## NIC inventory

| Interface | PCI ID | Chip | Vendor | Driver | Max speed | Stability |
|---|---|---|---|---|---|---|
| eth0 | 0x1d6a:0x07b1 | Aquantia AQC107 | Marvell/Aquantia | `atlantic` | 10GbE | ⚠️ Unreliable |
| eth1 | 0x8086:0x1533 | Intel i210 | Intel | `igb` | 1GbE | ✅ Reliable |
| eth2 | 0x8086:0x1533 | Intel i210 | Intel | `igb` | 1GbE | ✅ Reliable |

## Aquantia AQC107 (eth0) — known issues

- Driver: `atlantic` on DSM
- Symptom: spontaneous `link change old 1000 new 0` events in dmesg — link drops without any physical cause
- Confirmed on both DS1621xs+ units
- Four brand-new cables tested — physical cables ruled out as cause
- **Decision (2026-05-14): exclude eth0 from bond on both units permanently**
- Port may be left cabled for future use (e.g. dedicated 10GbE connection) but should never be a bond slave

## Intel i210 (eth1, eth2)

- Driver: `igb`
- No link instability observed on either unit
- These are the only ports used in the bond (target state)

## MAC addresses

| Unit | eth0 | eth1 | eth2 |
|---|---|---|---|
| edgesynology1 | 90:09:d0:27:a3:42 | 90:09:d0:27:a3:43 | 90:09:d0:27:a3:44 |
| edgesynology2 | 00:11:32:ea:fa:fa | 00:11:32:ea:fa:f9 | 00:11:32:ea:fa:fb |

## Verifying NIC chip from command line

```bash
cat /sys/class/net/eth0/device/vendor   # 0x1d6a = Aquantia, 0x8086 = Intel
cat /sys/class/net/eth0/device/device   # 0x07b1 = AQC107, 0x1533 = i210
```
