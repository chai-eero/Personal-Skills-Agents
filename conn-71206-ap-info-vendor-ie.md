---
name: conn-71206-ap-info-vendor-ie
description: "Verify the advertise_ap_info feature (CONN-71206) on stage network 1242756. Sets the feature flag, SSHes into one node to capture beacon frames from the other node's AP, and validates vendor IEs (OUI 14:22:DB) are present over-the-air. Requires firmware >= v7.16.0."
---

# CONN-71206: Advertise AP Info Vendor IE — Live Verification

## Target Network

- **Network**: Chai's LG Office — https://admin.stage.e2ro.com/networks/1242756
- **Environment**: Stage (do NOT use `--prod`)
- **Network ID**: `1242756`

### Nodes (both Merci, v7.17.0)

| Name | Serial (DSN) | Session | Role |
|------|-------------|---------|------|
| Hallway | `GGC54M015352006L` | `SES0013232053` | Node 1 |
| Hallway | `GGC54M0153520010` | `SES0013232056` | Node 2 |

## Overview

Verify the `advertise_ap_info` feature end-to-end:
1. Check firmware >= v7.16.0 (currently v7.17.0 — OK)
2. Set the `advertise_ap_info` feature flag
3. SSH into **one node** and capture beacon frames from the **other node's** AP to confirm the vendor IE is broadcast over-the-air
4. Cross-reference with system logs

## Process

### Step 0: Authenticate (REQUIRED FIRST)

The admin API uses an Okta-backed token that expires. Before anything else, the **user must** run this themselves in the session (it requires interactive Okta login, which the agent cannot perform):

```
! eero api admin auth
```

Then verify credentials are valid:

```bash
eero api admin whoami
```

**STOP CONDITION:** If `whoami` returns `401 Unauthorized` / `Authentication Required` (or any auth error), **do not continue the test**. The credentials have failed. Instruct the user to run `eero api admin auth` and re-run. Do not attempt to set flags, SSH, or capture beacons without valid admin credentials.

> Note: `eero nimble auth` only creates local device SSH credentials — it does NOT authenticate the admin API. The admin API requires `eero api admin auth` (Okta).

### Step 1: Verify Firmware

```bash
eero api admin get-network 1242756 | grep firmware
```

Confirm both nodes are >= v7.16.0. Currently `v7.17.0-2937+test-inet-debug.stage.merci` — proceed.

### Step 2: Set the Feature Flag

```bash
eero api admin set-network-flag 1242756 advertise_ap_info '{"ap-name": true, "network-state": true}'
```

Verify:
```bash
eero api admin get-network-flag 1242756 advertise_ap_info
```

Expected: `{"ap-name": true, "network-state": true}`

### Step 3: Confirm via System Logs (Both Nodes)

Wait 15-20 seconds for cloud config propagation, then check both nodes:

```bash
eero ssh GGC54M015352006L sh -c 'grep -E "ap_info_ie|Vendor IE changed" /var/log/messages | tail -20'
```

```bash
eero ssh GGC54M0153520010 sh -c 'grep -E "ap_info_ie|Vendor IE changed" /var/log/messages | tail -20'
```

**Success indicators:**

| Log Pattern | Meaning |
|-------------|---------|
| `ap_info_ie: feature active, enabled fields: ['ap-name', 'network-state']` | Flag received and parsed |
| `Vendor IE changed for subnet X: (empty) -> <hex>` | IEs built and pushed to hostapd |
| `ap_info_ie: internet_up changed to True` | Internet bit set |
| `ap_info_ie: wan_metric changed to X, limited=False` | Backhaul resolved |

### Step 4: Capture Beacon Frames — Filter for eero Vendor IEs Only

The key verification: confirm the vendor IE (OUI `14:22:DB`, types `05`/`06`) is in the actual beacon frames over-the-air.

**Important constraints discovered on this device:**
- `tcpdump` on AP interfaces (`ap_G0`, `ap_tt0`, etc.) does NOT work — they are bridged interfaces without 802.11 link-layer headers
- `hostapd_cli get vendor_elements` is not supported in eero's hostapd build (returns FAIL)
- The correct approach is **`iw scan dump`** from a station or mesh interface on the **other node**, which shows beacons received over-the-air including all vendor IEs

#### Method: iw scan dump from Node 2 → sees Node 1's beacons (and vice versa)

SSH into Node 2, trigger a scan on a station interface, then dump cached beacons and filter for eero OUI:

```bash
# SSH into Node 2, trigger scan then dump — filter for eero vendor IEs only
eero ssh GGC54M0153520010 sh -c "iw dev ec_sta1 scan trigger 2>/dev/null; sleep 2; iw dev ec_sta1 scan dump 2>/dev/null | grep -A1 'Vendor specific: OUI 14:22:db'"
```

If `ec_sta1` doesn't have scan results, try other interfaces:
```bash
eero ssh GGC54M0153520010 sh -c "for iface in ec_sta0 ec_sta1 ec_sta2 mesh0 mesh1 mesh2; do echo '--- '\$iface' ---'; iw dev \$iface scan dump 2>/dev/null | grep -A1 'Vendor specific: OUI 14:22:db'; done"
```

#### Same from Node 1, to verify Node 2's beacons:

```bash
eero ssh GGC54M015352006L sh -c "iw dev ec_sta1 scan trigger 2>/dev/null; sleep 2; iw dev ec_sta1 scan dump 2>/dev/null | grep -A1 'Vendor specific: OUI 14:22:db'"
```

#### Expected output (filtered — only our IEs)

`iw scan dump` formats vendor IEs as:
```
	Vendor specific: OUI 14:22:db, data: 06 05
	Vendor specific: OUI 14:22:db, data: 05 43 68 61 69 ...
```

Where:
- `data: 06 05` → type 06 (NETWORK_STATE) + bitmask 0x05
- `data: 05 43 68 61 69...` → type 05 (AP_NAME) + UTF-8 bytes

The `grep -A1 'Vendor specific: OUI 14:22:db'` filters out ALL other vendor IEs (Qualcomm, Microsoft, etc.) and shows only the two eero AP info IEs.

#### If scan dump is empty or shows no eero IEs

1. The scan cache may be stale — trigger a fresh scan:
   ```bash
   eero ssh GGC54M0153520010 sh -c "iw dev ec_sta1 scan trigger freq 5580 2>/dev/null; sleep 3; iw dev ec_sta1 scan dump 2>/dev/null | grep -A1 '14:22:db'"
   ```

2. Try mesh interfaces which passively hear beacons:
   ```bash
   eero ssh GGC54M0153520010 sh -c "iw dev mesh1 scan dump 2>/dev/null | grep -A1 '14:22:db'"
   ```

3. If NIMBLE auth expires, re-auth: `eero nimble auth`

### Step 5: Decode and Validate

**NETWORK_STATE bitmask (1 byte):**

| Bit(s) | Field | Values |
|--------|-------|--------|
| 0 | Internet reachable | 0=no, 1=yes |
| 1-2 | Captive portal state | 00=unknown, 01=not behind, 11=behind |
| 3 | Backhaul limited | 0=no, 1=yes |
| 4-7 | Reserved | 0 |

**Expected for a healthy stage network with internet:**
- Bitmask = `0x05` (binary `00000101`) = internet up + CP not behind
- Or `0x01` (binary `00000001`) = internet up + CP unknown

**AP_NAME:** Only on enterprise networks. This network ("Chai's LG Office") is likely residential, so expect NETWORK_STATE only.

### Step 6: Report Results

Summarize:
- Both nodes firmware v7.17.0 >= v7.16.0 ✓
- Feature flag set with ap-name + network-state ✓
- System logs confirm IE pushed to hostapd (include hex from logs)
- **Beacon capture confirms** IE bytes `dd XX 14 22 db 06 YY` present over-the-air
- Decoded bitmask value and meaning

## IE Frame Reference

```
NETWORK_STATE example (internet up, not behind CP):
  dd 05 14 22 db 06 05
  │  │  └──────┘  │  └─ bitmask 0x05
  │  │    OUI     └─ type 0x06
  │  └─ length 5
  └─ element ID 0xDD

AP_NAME example ("Office"):
  dd 0a 14 22 db 05 4f 66 66 69 63 65
  │  │  └──────┘  │  └────────────────┘
  │  │    OUI     │   "Office" UTF-8
  │  └─ length    └─ type 0x05
  └─ element ID
```

## Cleanup

```bash
eero api admin delete-network-flag 1242756 advertise_ap_info
```

## Troubleshooting

**No vendor IE in beacon capture:**
- Confirm logs show "Vendor IE changed" (nodelib pushed it)
- Try different interfaces: `ap_G0`, `ap_G1`, `ap_tt0`, `ap_tt1`
- eeroConnect BSSes (`eero_connect*`) are intentionally skipped — don't capture there
- Make sure tcpdump is capturing beacons (not just data frames)

**tcpdump fails:**
- Interface might not support it; check `iw dev` for active interfaces
- Try: `tcpdump -i <iface> -c 1 type mgt subtype beacon` to verify beacons exist

**AP_NAME not in beacon:**
- AP_NAME only emitted on enterprise networks
- This residential network will only have NETWORK_STATE

**No ap_info_ie logs after 30s:**
- Re-check flag: `eero api admin get-network-flag 1242756 advertise_ap_info`
- Confirm APs active: `eero ssh GGC54M015352006L hostapd_cli -i ap_G0 status`
- Cloud config may need a reboot to pick up: `eero api user reboot-eero --session SES0013232053`
