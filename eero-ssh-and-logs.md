---
name: eero-ssh-and-logs
description: "Guide for SSH into eero devices and inspecting on-device logs. Covers all authentication cases (NIMBLE auth, re-auth, expired creds), remote command execution patterns, and flexible grep/tail/follow workflows for any log file, duration, or pattern the user needs."
---

# eero SSH & Log Inspection Skill

SSH into eero devices and inspect logs on-device in real time. Handles every auth scenario, remote command patterns, and flexible log searching.

---

## Authentication (Every Case)

SSH requires NIMBLE credentials. Always verify auth state before attempting SSH.

### Fresh Auth (No Existing Credentials)

```bash
eero nimble auth
```

This creates short-lived SSH credentials. Must be on VPN.

### Check If Already Authenticated

```bash
eero nimble list-trusted-nodes
```

If this returns nodes, credentials are active. If empty or errors, re-auth.

### Re-Auth (Expired Credentials)

NIMBLE creds expire after ~24 hours. Symptoms:
- SSH hangs or times out
- "Permission denied" or "Connection refused"
- `list-trusted-nodes` returns empty

Fix:
```bash
eero nimble auth
```

### Revoke and Re-Create (Stale/Corrupt Credentials)

If `eero nimble auth` itself fails or SSH still doesn't work after re-auth:

```bash
eero nimble revoke-credentials
eero nimble auth
```

### Admin API Auth (Needed for Device Lookup Before SSH)

If you need to look up a device identifier before SSH:

```bash
eero api admin auth
```

### Full Auth Recovery Sequence

When nothing works, run the complete sequence:

```bash
# 1. Revoke stale NIMBLE creds
eero nimble revoke-credentials

# 2. Re-auth admin API (needed for identifier resolution)
eero api admin auth

# 3. Create fresh NIMBLE creds
eero nimble auth

# 4. Verify
eero nimble list-trusted-nodes
```

### Common Auth Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Connection timed out` | NIMBLE expired or VPN down | `eero nimble auth` + check VPN |
| `Permission denied (publickey)` | NIMBLE creds revoked/expired | `eero nimble revoke-credentials && eero nimble auth` |
| `Could not resolve hostname` | VPN disconnected | Reconnect VPN |
| `NIMBLE auth failed` | Admin API session expired | `eero api admin auth` then `eero nimble auth` |
| `No route to host` | Device offline or wrong env | Verify device is online, check stage vs prod |

---

## SSH Basics

### Identifiers

SSH accepts any of these as `<IDENTIFIER>`:
- **Serial (DSN)**: `GGC54M015352006L`
- **MAC address**: `14:22:DB:XX:XX:XX`
- **Session ID**: `SES0013232053`
- **eero ID**: numeric ID from admin API

```bash
eero ssh <IDENTIFIER>
```

### Interactive Shell

```bash
eero ssh GGC54M015352006L
```

Opens a shell on the device. `exit` or `Ctrl+D` to disconnect.

### Run a Single Command

```bash
eero ssh <IDENTIFIER> <COMMAND>
```

Example:
```bash
eero ssh GGC54M015352006L uptime
```

### Run Complex Commands (sh -c)

For pipes, redirects, semicolons, or any shell metacharacters, wrap in `sh -c`:

```bash
eero ssh <IDENTIFIER> sh -c '<COMMAND>'
```

Use single quotes around the command to prevent local shell expansion:
```bash
eero ssh GGC54M015352006L sh -c 'ps aux | grep hostapd'
```

### Multiple Commands in One SSH Session

```bash
eero ssh <IDENTIFIER> sh -c 'cmd1; cmd2; cmd3'
```

Or with `&&` for fail-fast:
```bash
eero ssh <IDENTIFIER> sh -c 'cmd1 && cmd2 && cmd3'
```

### Environment Variables in Remote Commands

Escape `$` to prevent local expansion:

```bash
eero ssh <IDENTIFIER> sh -c 'echo $HOSTNAME'        # remote's $HOSTNAME
eero ssh <IDENTIFIER> sh -c "echo \$HOSTNAME"       # also works (escaped)
```

### Quoting Nested Quotes

When the remote command itself needs quotes, alternate quote styles:

```bash
eero ssh <IDENTIFIER> sh -c 'grep "some pattern" /var/log/messages'
```

Or escape inner quotes:
```bash
eero ssh <IDENTIFIER> sh -c "grep 'some pattern' /var/log/messages"
```

---

## Log Files on eero Devices

### Key Log Locations

| Log File | Contents |
|----------|----------|
| `/var/log/messages` | Main system log (syslog) — most useful for debugging |
| `/var/log/kernel` | Kernel messages (driver issues, panics, wifi subsystem) |
| `/var/log/daemon.log` | Daemon-specific logs |
| `/tmp/log/rt_agent.log` | Real-time agent logs |
| `/tmp/log/foghorn.log` | Foghorn (metrics/telemetry) logs |
| `/var/log/analytics` | Analytics events |

### Quick Check — Is There Anything in the Log?

```bash
eero ssh <IDENTIFIER> sh -c 'wc -l /var/log/messages'
```

---

## Grep Patterns (Search for Anything)

### Basic grep

```bash
eero ssh <IDENTIFIER> sh -c 'grep "<PATTERN>" /var/log/messages'
```

### Case-Insensitive

```bash
eero ssh <IDENTIFIER> sh -c 'grep -i "<PATTERN>" /var/log/messages'
```

### Extended Regex (Multiple Patterns with OR)

```bash
eero ssh <IDENTIFIER> sh -c 'grep -E "pattern1|pattern2|pattern3" /var/log/messages'
```

### Invert Match (Exclude Noise)

```bash
eero ssh <IDENTIFIER> sh -c 'grep -v "noisy_pattern" /var/log/messages'
```

### Context Lines (Before/After Match)

```bash
# 3 lines before and after each match
eero ssh <IDENTIFIER> sh -c 'grep -B3 -A3 "<PATTERN>" /var/log/messages'

# 5 lines after only
eero ssh <IDENTIFIER> sh -c 'grep -A5 "<PATTERN>" /var/log/messages'
```

### Count Occurrences

```bash
eero ssh <IDENTIFIER> sh -c 'grep -c "<PATTERN>" /var/log/messages'
```

### Show Line Numbers

```bash
eero ssh <IDENTIFIER> sh -c 'grep -n "<PATTERN>" /var/log/messages'
```

### Search Multiple Log Files

```bash
eero ssh <IDENTIFIER> sh -c 'grep "<PATTERN>" /var/log/messages /var/log/kern.log'
```

### Recursive Search in a Directory

```bash
eero ssh <IDENTIFIER> sh -c 'grep -r "<PATTERN>" /var/log/'
```

---

## Tail Patterns (Recent Logs & Following)

### Last N Lines

```bash
# Last 20 lines
eero ssh <IDENTIFIER> sh -c 'tail -20 /var/log/messages'

# Last 100 lines
eero ssh <IDENTIFIER> sh -c 'tail -100 /var/log/messages'

# Last 500 lines (large window)
eero ssh <IDENTIFIER> sh -c 'tail -500 /var/log/messages'
```

### Tail + Grep (Recent Lines Matching a Pattern)

```bash
# Last 200 lines, filtered to pattern
eero ssh <IDENTIFIER> sh -c 'tail -200 /var/log/messages | grep "<PATTERN>"'

# Last 50 lines of kernel log matching wifi
eero ssh <IDENTIFIER> sh -c 'tail -50 /var/log/kern.log | grep -i wifi'
```

### Grep + Tail (All Matches, Show Last N)

```bash
# All matches of pattern, show most recent 20
eero ssh <IDENTIFIER> sh -c 'grep "<PATTERN>" /var/log/messages | tail -20'
```

### Head (First N Lines / Oldest)

```bash
# First 10 matches (earliest occurrences)
eero ssh <IDENTIFIER> sh -c 'grep "<PATTERN>" /var/log/messages | head -10'
```

---

## Time-Based Log Filtering

eero syslog lines are typically timestamped like: `Jun 24 14:30:01 ...`

### Grep by Date/Time

```bash
# All logs from a specific hour
eero ssh <IDENTIFIER> sh -c 'grep "Jun 24 14:" /var/log/messages'

# Specific minute range
eero ssh <IDENTIFIER> sh -c 'grep "Jun 24 14:3[0-5]" /var/log/messages'
```

### Logs from the Last N Minutes (Approximate)

Since devices don't always have `journalctl --since`, use line count as a proxy:
- ~1 line/sec under normal load → `tail -60` ≈ last minute
- Adjust based on log verbosity

```bash
# Approximate last 5 minutes (adjust -300 based on log rate)
eero ssh <IDENTIFIER> sh -c 'tail -300 /var/log/messages'
```

### Combining Time + Pattern

```bash
eero ssh <IDENTIFIER> sh -c 'grep "Jun 24 14:" /var/log/messages | grep "<PATTERN>"'
```

---

## Follow / Live Tail (Continuous Monitoring)

### Follow Logs in Real Time

For interactive monitoring (run this in your terminal, not automated):

```bash
eero ssh <IDENTIFIER> sh -c 'tail -f /var/log/messages'
```

### Follow with Filter

```bash
eero ssh <IDENTIFIER> sh -c 'tail -f /var/log/messages | grep --line-buffered "<PATTERN>"'
```

`--line-buffered` ensures grep outputs immediately instead of buffering.

### Follow Multiple Patterns

```bash
eero ssh <IDENTIFIER> sh -c 'tail -f /var/log/messages | grep --line-buffered -E "pattern1|pattern2"'
```

### Follow for a Fixed Duration

Use `timeout` to auto-stop after N seconds:

```bash
# Follow for 30 seconds
eero ssh <IDENTIFIER> sh -c 'timeout 30 tail -f /var/log/messages'

# Follow for 2 minutes, filtered
eero ssh <IDENTIFIER> sh -c 'timeout 120 tail -f /var/log/messages | grep --line-buffered "<PATTERN>"'

# Follow for 5 minutes
eero ssh <IDENTIFIER> sh -c 'timeout 300 tail -f /var/log/messages | grep --line-buffered "<PATTERN>"'
```

### Follow Until a Specific Pattern Appears

```bash
# Stop as soon as "success_marker" appears
eero ssh <IDENTIFIER> sh -c 'tail -f /var/log/messages | grep --line-buffered -m 1 "success_marker"'
```

`-m 1` exits grep after the first match, which closes the pipe and stops tail.

---

## Combined Workflows

### "Show me the last N lines matching X"

```bash
eero ssh <IDENTIFIER> sh -c 'grep "<PATTERN>" /var/log/messages | tail -<N>'
```

### "Search for X in the last N lines"

```bash
eero ssh <IDENTIFIER> sh -c 'tail -<N> /var/log/messages | grep "<PATTERN>"'
```

### "Follow logs for X seconds, only showing Y"

```bash
eero ssh <IDENTIFIER> sh -c 'timeout <SECONDS> tail -f /var/log/messages | grep --line-buffered "<PATTERN>"'
```

### "Count how many times X happened"

```bash
eero ssh <IDENTIFIER> sh -c 'grep -c "<PATTERN>" /var/log/messages'
```

### "Show me X with 5 lines of context, last 10 occurrences"

```bash
eero ssh <IDENTIFIER> sh -c 'grep -B5 -A5 "<PATTERN>" /var/log/messages | tail -70'
```

### "Grep across all log files for X"

```bash
eero ssh <IDENTIFIER> sh -c 'grep -r "<PATTERN>" /var/log/ /tmp/log/ 2>/dev/null'
```

### "Show unique log lines matching X (deduplicated)"

```bash
eero ssh <IDENTIFIER> sh -c 'grep "<PATTERN>" /var/log/messages | sort -u'
```

### "Show logs between two timestamps"

```bash
eero ssh <IDENTIFIER> sh -c 'sed -n "/Jun 24 14:00/,/Jun 24 14:30/p" /var/log/messages'
```

---

## Useful On-Device Commands (via SSH)

### System Info

```bash
eero ssh <IDENTIFIER> sh -c 'uname -a'                    # kernel version
eero ssh <IDENTIFIER> sh -c 'uptime'                       # uptime + load
eero ssh <IDENTIFIER> sh -c 'cat /etc/eero-release'        # firmware version
eero ssh <IDENTIFIER> sh -c 'free -m'                      # memory usage
eero ssh <IDENTIFIER> sh -c 'df -h'                        # disk usage
```

### Networking

```bash
eero ssh <IDENTIFIER> sh -c 'ip addr'                      # all interfaces
eero ssh <IDENTIFIER> sh -c 'iw dev'                       # wireless interfaces
eero ssh <IDENTIFIER> sh -c 'brctl show'                   # bridge config
eero ssh <IDENTIFIER> sh -c 'ip route'                     # routing table
eero ssh <IDENTIFIER> sh -c 'cat /proc/net/arp'            # ARP table
```

### Process Inspection

```bash
eero ssh <IDENTIFIER> sh -c 'ps aux | grep <PROCESS>'
eero ssh <IDENTIFIER> sh -c 'pidof <PROCESS>'
```

### Hostapd Status

```bash
eero ssh <IDENTIFIER> sh -c 'hostapd_cli -i ap_G0 status'
eero ssh <IDENTIFIER> sh -c 'hostapd_cli -i ap_G0 all_sta'
```

### Wireless Scan (iw)

```bash
eero ssh <IDENTIFIER> sh -c 'iw dev ec_sta1 scan trigger 2>/dev/null; sleep 2; iw dev ec_sta1 scan dump'
```

---

## Troubleshooting SSH Issues

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| SSH hangs indefinitely | NIMBLE expired or VPN down | `eero nimble auth` + verify VPN |
| "Permission denied" | Creds revoked or wrong device | `eero nimble revoke-credentials && eero nimble auth` |
| "No route to host" | Device offline | Verify via `eero mac <IDENTIFIER>` or admin panel |
| "Connection reset" | Device rebooting | Wait 2-3 min, retry |
| Command runs but no output | Wrong log file or pattern | Try `wc -l` to confirm file has content |
| `grep` returns nothing | Pattern not in logs | Broaden pattern, check `tail -50` for log format |
| `tail -f` hangs after connecting | No new log lines being written | Device is quiet — trigger an action and watch |
| SSH works but command fails | Binary not on device | Check with `which <cmd>` or `ls /usr/bin/` |

---

## Best Practices

1. **Always auth check first** — run `eero nimble list-trusted-nodes` before SSH attempts
2. **Use `sh -c` for anything complex** — pipes, redirects, semicolons all need it
3. **Single-quote remote commands** — prevents local shell from expanding variables
4. **Use `timeout` for follow commands** — prevents hanging in automated workflows
5. **Use `grep --line-buffered` with tail -f** — ensures real-time output
6. **Start broad, narrow down** — `tail -100` first, then add grep filters
7. **Check log file exists** — `ls -la /var/log/messages` before grepping
8. **Use `2>/dev/null`** — suppress stderr noise from interfaces that don't exist
9. **Stage only** — `eero ssh` only works on stage devices (prod requires different access)
10. **One command at a time in automation** — avoid long-running `tail -f` in scripts; use `timeout`
