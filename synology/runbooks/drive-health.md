# Runbook: Drive Health

## Checking SMART stats

Note: `smartctl` via the MCP returns limited output on DSM. Use DSM Storage Manager → HDD/SSD for full SMART data.

Via MCP (basic):
```bash
smartctl -A /dev/sda
```

Key attributes to watch:
- `Reallocated_Sector_Ct` (ID 5) — reallocated sectors. Any non-zero value is a warning.
- `Current_Pending_Sector` (ID 197) — sectors pending reallocation. Non-zero = drive struggling.
- `Offline_Uncorrectable` (ID 198) — sectors that couldn't be corrected. Non-zero = serious.
- `Power_On_Hours` — drive age

## Known issues

Two 18TB drives on edgesynology2 (sdd: WUH721818ALE6L4, sde: WUH721818ALE604) showed elevated write latencies. Long SMART tests were initiated. Results to be recorded here once completed.

## SMART test caution

**Do NOT trigger SMART tests via the MCP `start_smart_test` tool.** It deadlocks the MCP server. Use DSM Storage Manager → HDD/SSD → Health Info → SMART Test instead.
