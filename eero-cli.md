---
name: eero-cli
description: "Comprehensive eero CLI reference for network engineering tasks — device lookup, log fetching, ACS channel management, network diagnostics, OTA firmware updates, API queries, mesh inspection, and local device interaction. Use when working with eero devices, networks, debugging, firmware, or any eero CLI operation."
---

# eero CLI Skill

The `eero` CLI is the primary tool for eero engineering tasks. It provides commands to manage devices, networks, firmware, logs, and more across stage and prod environments.

## Prerequisites

- eero CLI installed: `brew install eero` or check `eero --version`
- Credentials configured: run `eero creds list` to verify, `eero creds set <domain>` to configure
- VPN connected for cloud API access

## Environment & Authentication

Most commands default to **stage**. Add `--prod` or `--env prod` to target production.

### Authentication

```bash
# Admin API auth
eero api admin auth

# User API auth (stage)
eero api user auth

# User API auth (prod) — uses SSO
eero api user --prod auth --sso

# Check current auth status
eero api admin whoami
eero api user whoami
```

### Credential Management

```bash
eero creds list                      # list stored credentials
eero creds get <DOMAIN>              # get credentials for a domain
eero creds set <DOMAIN>              # set credentials for a domain (interactive)
```

### NIMBLE (required for SSH/SCP)

```bash
eero nimble auth                     # create NIMBLE credentials
eero nimble list-trusted-nodes       # list trusted nodes
eero nimble revoke-credentials       # revoke credentials
```

---

## ACS (Automatic Channel Selection)

### Get ACS Info / Channel Status

```bash
# Get ACS and channel configuration for a network
eero acs info <NETWORK_ID>
eero acs info --prod <NETWORK_ID>

# List available channels
eero acs channels
eero acs channels --band 5GHz
eero acs channels --width 80 --model j
eero acs channels --country US --format json
eero acs channels --network <NETWORK_ID>

# Show info for a specific channel
eero acs channel <CHANNEL_NUMBER>
```

### Run ACS

```bash
# Run ACS on a network (triggers channel selection)
eero acs run <NETWORK_ID>
eero acs run --prod <NETWORK_ID>
eero acs run --band 5GHz <NETWORK_ID>
```

### Switch Channels Manually

```bash
# Switch by channel number (requires --band)
eero acs switch <NETWORK_ID> 36 --band 5GHz

# Switch by frequency
eero acs switch <NETWORK_ID> 5180

# Switch with specific width
eero acs switch <NETWORK_ID> 36 --band 5GHz --width 80

# Switch a specific eero
eero acs switch-eero <EERO_ID> 36 --band 5GHz
```

### Enable/Disable ACS

```bash
eero acs enable <NETWORK_ID>
eero acs disable <NETWORK_ID>
```

---

## Network Management

### List Nodes & Clients

```bash
eero network nodes <NETWORK_ID>
eero network nodes --prod <NETWORK_ID>
eero network clients <NETWORK_ID>
```

### Outages & Reboots

```bash
eero network outages <NETWORK_ID>    # show network outage history
eero network reboots <NETWORK_ID>    # show node reboot history
```

### Alerts & Utilization

```bash
eero network alerts <NETWORK_ID>         # show intelligence alerts
eero network utilization <NETWORK_ID>    # show channel utilization data
eero network utilization-events <NETWORK_ID>
```

### Feature Flags

```bash
eero network flags <NETWORK_ID>
eero network flags --prod <NETWORK_ID>
```

### Audit Logs

```bash
eero network audit-logs <NETWORK_ID>
```

### Network Time

```bash
eero network time <NETWORK_ID>     # print the local time of a network
```

---

## Network Diagnostics

### Health Check (14 automated checks)

```bash
eero network check <NETWORK_ID> --start "2026-02-20" --duration 2d
```

Available checks:
- `check_channel_utilization`
- `check_duplicate_names`
- `check_eero_outages`
- `check_intelligence_alerts`
- `check_mixed_firmware`
- `check_network_group`
- `check_network_outages`
- `check_network_wan`
- `check_offline_eeros`
- `check_reboots`
- `check_slow_ethernet_links`
- `check_status`
- `check_thermals`
- `check_weak_mesh_quality`

### Event History

```bash
# Network-level events
eero network events <NETWORK_ID> --start "..." --duration 1h --tz UTC

# Topology changes (node join/leave, mesh changes)
eero network topology-events <NETWORK_ID> --start "..." --duration 2d --tz UTC

# Client-specific events
eero network client-events <NETWORK_ID> <DEVICE_ID> --start "..." --duration 1h
```

---

## Device Lookup

### MAC Lookup

```bash
eero mac <MAC_ADDRESS>               # stage
eero mac --prod <MAC_ADDRESS>        # prod
eero mac --prod <SERIAL>             # also accepts serial
eero mac --prod <EERO_ID>            # also accepts eero ID
```

Output includes node info and full MAC-to-interface mapping table.

### Serial Lookup

```bash
eero serial <SERIAL>
```

### Session Lookup

```bash
eero session <SESSION_ID|MAC|SERIAL|LOCATION>
eero session --env prod <IDENTIFIER>
```

### Model Info

```bash
eero model                           # list all models
eero model <MODEL_CODE>              # check specific model (j, m, pa, s, etc.)
```

### SKU Lookup

```bash
eero sku <SKU>
```

---

## Log Downloading

Core workflow: **upload → fetch → tag**

### Upload Logs

```bash
eero logs upload --eero <EERO_ID|MAC|SERIAL|LOCATION>
eero logs upload --network <NETWORK_ID|SSID>
eero logs upload --eero <ID> --prod
```

### Fetch Logs

```bash
# Fetch all logs for a network
eero logs fetch --network <NETWORK_ID> --start <DATE> --duration 12h --tz UTC \
    --output-dir ~/logs/network-<NETWORK_ID>

# Fetch for a specific eero
eero logs fetch --eero <EERO_ID|MAC|SERIAL> --start <DATE> --duration 12h --tz UTC \
    --output-dir ~/logs/eero-<EERO_ID>

# Fetch specific log types only
eero logs fetch --network <NETWORK_ID> --start <DATE> --duration 12h --tz UTC \
    --output-dir <DIR> -t system -t kernel

# Prod
eero logs fetch --eero <ID> --start <DATE> --duration 24h --log-type system --output-dir <DIR> --prod
```

Log types: `system`, `kernel`, `openthread`, `samknows`, `rt_agent`, `qmdl`, `foghorn`, `qfirehose`

### Tag Logs with Node Names

```bash
eero logs tag <DIR>
```

### List Available Logs

```bash
eero logs list --network <NETWORK_ID> --start "2026-02-21" --duration 12h --tz UTC
```

### Other Log Commands

```bash
eero logs join <DIR>     # combine logs from same eero + same type
eero logs parse          # parse raw logs into human-readable format
eero logs view           # view logs directly from an eero
```

---

## OTA (Over-the-Air Updates)

> CRITICAL: OTA commands should ONLY be used on **stage** networks unless explicitly confirmed by the user for prod.

### Firmware Management

```bash
eero firmware list <VERSION_PREFIX>        # e.g., v7.16
eero firmware list -m j <VERSION_PREFIX>   # filter by model
eero firmware list --prod <VERSION_PREFIX>
eero firmware list --base <VERSION_PREFIX> # base firmwares only
eero firmware info <VERSION>
eero firmware info -m m <VERSION>
eero firmware info --prod <VERSION>
```

### Run OTA

```bash
eero ota run --session <SESSION_ID|MAC|SERIAL> --firmware <VERSION>
eero ota run --network <NETWORK_ID> --firmware <VERSION>
eero ota run --network <NETWORK_ID> --firmware <VERSION> --dry-run
```

### Monitor OTA

```bash
eero ota monitor <UPDATE_ID>
eero ota monitor --session <IDENTIFIER>
eero ota monitor --network <NETWORK_ID>
```

### OTA History

```bash
eero ota history --session <SESSION_ID|MAC|SERIAL>
eero ota history --network <NETWORK_ID>
eero ota history --prod --network <NETWORK_ID>
```

### OTA Summary

```bash
eero ota summary <OTA_ID>
eero ota summary --prod <OTA_ID>
```

---

## API Queries

### Admin API

```bash
eero api admin <COMMAND>
eero api admin --prod <COMMAND>
eero api admin curl <ENDPOINT>                   # raw HTTP request
eero api admin curl --method POST <ENDPOINT>     # POST request
eero api admin curl --json='{"key":"val"}' <ENDPOINT>

# Common admin commands
eero api admin get-network <NETWORK_ID>
eero api admin get-eero <EERO_ID>
eero api admin get-session <SESSION_ID>
eero api admin get-network-flags <NETWORK_ID>
eero api admin set-network-flag <NETWORK_ID> <FLAG> <VALUE>
eero api admin search <QUERY>
eero api admin enqueue-action --action <ACTION_NAME> <EERO_ID>
eero api admin get-actions <EERO_ID>
```

### User API

```bash
eero api user <COMMAND>
eero api user --prod <COMMAND>
eero api user curl <ENDPOINT>

# Common user commands
eero api user get-networks
eero api user get-eeros --network <NETWORK_ID>
eero api user get-devices --network <NETWORK_ID>
eero api user get-speedtests --network <NETWORK_ID>
eero api user run-speedtest --network <NETWORK_ID>
eero api user get-channel-utilization --network <NETWORK_ID>
```

### Node API

```bash
eero api node <COMMAND>
```

---

## Actions (Real-Time Diagnostics)

```bash
eero action speedtest <IDENTIFIER>
eero action ping <IDENTIFIER>
eero action traceroute <IDENTIFIER>
eero action tcpdump <IDENTIFIER>
eero action iperf-eero <IDENTIFIER1> <IDENTIFIER2>
eero action nslookup <IDENTIFIER>
eero action list <IDENTIFIER>        # view the action queue
```

### Enqueue Custom Actions via Admin API

```bash
eero api admin enqueue-action --action dynamic \
    --action-kwargs name=<action_name> \
    --action-kwargs args='[{"key":"<param>","value":"<value>"}]' <EERO_ID>
```

---

## Search

```bash
eero search <QUERY>                  # Admin Panel (stage)
eero search --prod <QUERY>           # Admin Panel (prod)
eero search --insight <QUERY>        # Insight
```

---

## Mesh Inspection

```bash
eero mesh links <IDENTIFIER>         # show mesh links
eero mesh graph <IDENTIFIER>         # graph ethernet links and mesh paths
eero mesh fdb <IDENTIFIER>           # show forwarding database
eero mesh ethers <IDENTIFIER>        # generate Wireshark ethers file
eero mesh hosts <IDENTIFIER>         # generate Wireshark hosts file
```

---

## Shapeshift (Environment Switching)

```bash
eero shapeshift --eero <EERO_ID> --to stage
eero shapeshift --network <NETWORK_ID> --to prod
```

---

## Local Device Interaction

### gRPC (local eeros on same network)

```bash
eero grpc get-network-status
eero grpc get-node-status
eero grpc get-network-topology
eero grpc get-scan-results
eero grpc reboot
eero grpc blink-led
```

### BLE

```bash
eero ble scan
eero ble read <IDENTIFIER>
```

### mDNS Discovery

```bash
eero mdns
eero mdns --service _eero._tcp.local.
```

---

## SSH/SCP (Stage Only)

```bash
eero ssh <IDENTIFIER>                # SSH to a stage eero
eero scp <LOCAL_PATH> <IDENTIFIER>:<REMOTE_PATH>   # copy files
```

Requires NIMBLE credentials (`eero nimble auth` first).

---

## MCP Server

```bash
eero mcp api                         # run MCP server for eero APIs
eero mcp ssh                         # run MCP server for SSH access
```

---

## Troubleshooting

- **Credential errors**: Run `eero creds list` to check. Use `eero creds set <domain>` to reconfigure.
- **`logs list` returns empty**: Try `--prod` flag, or use `eero logs fetch` with a wider time range.
- **Stage vs Prod**: Most commands default to stage. Use `--prod` or `--env prod` for production.
- **Command not found**: Verify with `which eero` and update with `brew upgrade eero`.
- **SSH fails**: Run `eero nimble auth` first to create credentials.
- **API 401/auth errors**: Re-auth with `eero api admin auth` or `eero api user --prod auth --sso`.
- **VPN required**: Cloud API commands need VPN. Local commands (gRPC, BLE, mDNS) do not.
- **Debug mode**: Add `-v` for verbose output or `--debug-info` to print environment details.
- **Interactive TUI**: Run `eero tui` for a visual command builder.
