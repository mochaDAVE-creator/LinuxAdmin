---
aliases:
  - journalctl
  - Log Analysis
  - systemd Journal
  - Forensic Log Parsing
tags:
  - linux
  - journalctl
  - logging
  - forensics
  - net412
  - net179
  - phase3
date: 2026-05-10
---

# 03 — journalctl & Log Forensics

> [!info] Why This Matters
> journald aggregates logs from every process, kernel, and systemd unit into a structured binary journal. journalctl is the lens. A forensic analyst who can't read logs quickly is not an analyst. A sysadmin who doesn't check logs after incidents is leaving the door open.

## The Journal Architecture

```
Kernel            → kmsg → journald
systemd units     → stdout/stderr → journald
syslog (legacy)   → /dev/log → journald (forwarded)
Applications      → /dev/log or journal API → journald

Storage:
  /run/log/journal/   → volatile (lost on reboot)
  /var/log/journal/   → persistent (if configured)
```

```bash
# Enable persistent journal
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald

# Verify
ls /var/log/journal/
journalctl --disk-usage
```

---

## journalctl Query Fundamentals

### Time-Based Filtering

```bash
# Current boot only (most commonly needed)
journalctl -b

# Previous boot (boot number, -1 = one boot ago)
journalctl -b -1
journalctl -b -2

# List all recorded boots
journalctl --list-boots

# Time range
journalctl --since "2026-05-10 00:00:00" --until "2026-05-10 06:00:00"
journalctl --since "1 hour ago"
journalctl --since "yesterday"
journalctl --since "2026-05-09" --until "2026-05-10"
```

### Unit-Based Filtering

```bash
# Follow a specific service (tail -f equivalent)
journalctl -u sshd.service -f

# Multiple units
journalctl -u sshd.service -u nginx.service

# Kernel messages only
journalctl -k

# All messages from current boot for one unit
journalctl -b -u apex-predator.service --no-pager
```

### Priority (Severity) Filtering

```bash
# Syslog priority levels:
# 0=emerg  1=alert  2=crit  3=err  4=warning  5=notice  6=info  7=debug

# Show only errors and above
journalctl -p err

# Show warnings and above
journalctl -p warning

# Show a range
journalctl -p 3..6    # err through info

# Combine with unit
journalctl -u nginx -p err -b
```

### Output Format Control

```bash
# Default (human-readable)
journalctl -b -u sshd

# Short (one line per entry)
journalctl -o short

# Verbose (all metadata fields)
journalctl -o verbose

# JSON (machine-parseable — pipe to jq)
journalctl -o json-pretty | head -50
journalctl -o json | jq '.MESSAGE' | head -20

# Export for offline analysis
journalctl -b -o export > /tmp/boot_journal.export
journalctl --since "2026-05-09" -o json > /tmp/daily_audit.json

# Tail mode (continuous follow)
journalctl -f
journalctl -f -u apex-predator -p warning
```

---

## Structured Journal Fields

The journal stores structured key-value metadata. You can filter on any field.

```bash
# Common fields:
# _SYSTEMD_UNIT — which unit logged this
# _PID — process ID
# _UID — user ID
# _HOSTNAME — system hostname
# _COMM — process name (short)
# _EXE — full executable path
# PRIORITY — syslog priority number
# MESSAGE — the actual log message
# _TRANSPORT — kernel/syslog/journal/stdout/stderr

# Filter by field (exact match)
journalctl _SYSTEMD_UNIT=sshd.service
journalctl _UID=1000         # logs from UID 1000 (your user)
journalctl _COMM=python3     # logs from python3 processes
journalctl _EXE=/usr/bin/sudo   # all sudo invocations

# Combine filters (AND logic within same field, OR across invocations)
journalctl _SYSTEMD_UNIT=sshd.service _PID=1234

# List all available fields
man systemd.journal-fields
```

---

## Forensic Investigation Workflows

### Workflow 1: Boot Timeline Reconstruction (NET 179)

```bash
# Full boot sequence with timestamps
journalctl -b --no-pager -o short-iso | head -100

# Time to reach each target
systemd-analyze blame

# What crashed on last boot?
journalctl -b -1 -p err --no-pager | head -50

# Find unexpected service restarts
journalctl -b | grep "Starting\|Started\|Failed\|Stopped" | head -50
```

### Workflow 2: Authentication Forensics

```bash
# All authentication events (current boot)
journalctl -b -u sshd.service --no-pager | grep -E "Failed|Accepted|Invalid|session"

# Failed logins with timestamps
journalctl --since "24 hours ago" | grep "Failed password" | \
  awk '{print $1, $2, $3, $(NF-3)}' | sort | uniq -c | sort -rn

# Successful sudo usage
journalctl | grep "sudo" | grep "COMMAND" | \
  awk '{print $1, $2, $3, $NF}' | tail -20

# su escalations
journalctl | grep "pam_unix.*su:" | tail -20

# New user creations (forensics: detect persistence)
journalctl | grep "useradd\|adduser" | tail -20
```

### Workflow 3: Service Failure Root Cause Analysis

```bash
# What failed since last boot?
systemctl list-units --state=failed --no-pager

# Deep dive on a failed service
journalctl -u apex-predator.service -b --no-pager -p err

# Find the exact moment it started failing
journalctl -u apex-predator.service --no-pager | \
  grep -E "failed|error|panic|killed" | tail -30

# Check resource limit hits
journalctl | grep -i "oom\|killed process\|out of memory" | tail -10

# Check for kernel module issues
journalctl -k | grep -iE "error|warn|fail" | tail -20
```

### Workflow 4: Privilege Escalation Detection

```bash
# SUID usage
journalctl | grep "uid=0" | grep -v "root" | tail -20

# Sudo to root
journalctl | grep "sudo.*COMMAND=/bin/bash\|sudo.*COMMAND=/bin/sh" | tail -20

# Cron job executions (scheduled persistence)
journalctl | grep -i "cron\|CRON\|crond" | tail -20

# New service installations (attacker persistence)
journalctl | grep "systemd.*enabled\|systemd.*started" | \
  grep -v "^$(date +%b\ \ 1 2>/dev/null)" | tail -20
```

### Workflow 5: Your Rust Engine Forensics

```bash
# All logs from apex-predator across all boots
journalctl -u apex-predator.service --no-pager --since "7 days ago"

# Panics (Rust panics emit to stderr → journald)
journalctl -u apex-predator.service | grep -i "panic\|thread.*panicked\|SIGILL\|SIGSEGV"

# Performance events
journalctl -u apex-predator.service | grep -i "slow\|timeout\|latency"

# Startup and shutdown events only
journalctl -u apex-predator.service | grep "Started\|Stopped\|Deactivated"
```

---

## journalctl for CTF Log Analysis

When you receive log dumps in a CTF (digital forensics challenges), replicate journald's filtering:

```bash
# Simulate journalctl filtering on a plain log file
grep "May 10" auth.log | grep "Failed" | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn

# Build timeline from multiple log files
cat /var/log/auth.log /var/log/syslog | \
  awk '{print $1, $2, $3, $0}' | \
  sort -k1,3 | head -50

# Find the gap (periods of inactivity — possible evidence deletion)
journalctl --since "2026-05-09" --until "2026-05-10" -o json | \
  jq -r '.__REALTIME_TIMESTAMP' | \
  awk 'NR>1 {diff=$1-prev; if(diff>60000000) print "GAP:", diff/1000000, "seconds at", prev} {prev=$1}'
```

---

## Log Integrity and Sealing

> [!warning] Journal Tampering
> The binary journal format makes casual editing harder but doesn't prevent it. For forensic-grade log integrity, use the journal sealing feature or export to a remote syslog server immediately.

```bash
# Check journal sealing status
journalctl --verify

# Seal the journal with FSS (Forward-Secure Sealing)
# Configure in /etc/systemd/journald.conf:
# Seal=yes
# ForwardToSyslog=yes  # also send to syslog for remote logging

# Remote logging with systemd-journal-remote (send to Proxmox log host)
sudo pacman -S systemd-journal-remote

# Client side (/etc/systemd/journal-upload.conf)
[Upload]
URL=https://proxmox-log-host:19532
ServerKeyFile=/etc/ssl/private/journal-upload.pem
ServerCertificateFile=/etc/ssl/certs/journal-upload.pem
TrustedCertificateFile=/etc/ssl/certs/journal-remote.pem
```

---

## Practical Lab: Build a Forensic Timeline

```bash
# Complete forensic timeline for the last 24 hours
{
  echo "=== FORENSIC TIMELINE: $(hostname) ==="
  echo "Generated: $(date -u)"
  echo "Analyst: $(whoami)"
  echo ""
  
  echo "--- Authentication Events ---"
  journalctl --since "24 hours ago" \
    | grep -E "session opened|session closed|Failed password|Accepted|sudo.*COMMAND" \
    | awk '{print $1, $2, $3, $0}' \
    | head -50
  
  echo ""
  echo "--- Service State Changes ---"
  journalctl --since "24 hours ago" \
    | grep -E "systemd.*Started|systemd.*Stopped|systemd.*Failed" \
    | head -30
  
  echo ""
  echo "--- Privilege Events ---"
  journalctl --since "24 hours ago" \
    | grep -E "sudo|su\[" \
    | grep -v "pam_unix" \
    | head -20
  
  echo ""
  echo "--- Kernel Errors ---"
  journalctl -k --since "24 hours ago" -p err --no-pager | head -20
  
  echo ""
  echo "--- Package Changes (pacman) ---"
  grep "$(date +%Y-%m-%d)" /var/log/pacman.log | head -20

} | tee /tmp/forensic_timeline.txt

sha256sum /tmp/forensic_timeline.txt
```

---

## Troubleshooting Scenario: Broken Service — Parse to Fix

**Symptom:** `systemctl status nginx.service` → `Active: failed`

**Systematic Log Investigation:**
```bash
# Step 1: Get the status message
systemctl status nginx.service --no-pager -l
# Look at the exit code in: Main PID: XXXX (code=exited, status=N/FAILURE)

# Step 2: Check the journal for specifics
journalctl -u nginx.service -b --no-pager -p err
# Common nginx errors:
# "Address already in use" → another process on port 80
# "invalid number of arguments" → config syntax error
# "failed (13: Permission denied)" → can't bind to port <1024 as non-root

# Step 3: Test config manually
sudo nginx -t    # test nginx config syntax
# nginx: [emerg] unexpected ";" ... → syntax error at that line

# Step 4: Check port conflicts
ss -tlnp | grep :80
# If occupied:
sudo fuser 80/tcp    # which PID is on port 80?
ls -la /proc/$(sudo fuser 80/tcp 2>/dev/null | tr -d ' ')/exe

# Step 5: Fix and restart
sudo systemctl start nginx
journalctl -u nginx -f   # watch for success
```

---

## 🏁 Proof of Work — Phase 3.3 Mini-CTF

> [!example] Challenge: Reconstruct an Attack Timeline from Journal
>
> ```bash
> # Simulate attack events in a test service log
> sudo systemd-cat -t "simulated-attack" -p info << 'EOF'
> Failed password for root from 10.0.0.99 port 44231 ssh2
> Failed password for root from 10.0.0.99 port 44232 ssh2
> Failed password for admin from 10.0.0.99 port 44233 ssh2
> Accepted publickey for dave from 192.168.1.10 port 22001 ssh2
> session opened for user dave by (uid=0)
> sudo: dave : COMMAND=/usr/bin/cat /etc/shadow
> EOF
>
> # Retrieve and analyze
> journalctl -t "simulated-attack" --no-pager -o json | \
>   jq -r '[.__REALTIME_TIMESTAMP, .MESSAGE] | @tsv' | \
>   awk -F'\t' '{
>     ts=$1/1000000;
>     cmd = "date -d @"ts" +\"%Y-%m-%d %H:%M:%S\"";
>     cmd | getline dt;
>     close(cmd);
>     print dt, $2
>   }'
>
> # Count failed attempts
> journalctl -t "simulated-attack" --no-pager | grep -c "Failed"
>
> # Identify the authenticated user
> journalctl -t "simulated-attack" --no-pager | grep "Accepted" | awk '{print $9}'
>
> # Find the suspicious sudo command
> journalctl -t "simulated-attack" --no-pager | grep "COMMAND" | awk -F'COMMAND=' '{print $2}'
>
> # Save and hash the timeline
> journalctl -t "simulated-attack" --no-pager > /tmp/phase3_3_pow.txt
> sha256sum /tmp/phase3_3_pow.txt
> ```
>
> **Submit:** The hash of your timeline file. The three key findings are: (1) attacking IP, (2) authenticated username, (3) the command run via sudo.

---

← [[02-Systemd-Deep-Dive]] | [[../Phase-4-Communicator/00-Phase4-Overview|Next: Phase 4 →]]
