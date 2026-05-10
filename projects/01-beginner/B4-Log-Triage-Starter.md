---
title: B4 - Log Triage Starter
aliases:
  - b4-log-triage
tags:
  - beginner
  - journalctl
  - logs
  - triage
  - grep
  - awk
  - evidence
date: 2026-05-10
---

# B4 — Log Triage Starter

## 1) Mission

- **Problem statement:** Operators who cannot efficiently query system logs cannot detect incidents, validate changes, or produce defensible evidence. Raw log files contain noise; this project teaches you to filter signal from noise using `journalctl`, `grep`, `awk`, and `sed`.
- **Why it matters:** Log triage is the entry point for every security investigation. Building repeatable query patterns now means faster, more consistent analysis later — whether you are responding to a failed login, a crashed service, or a suspicious process.

---

## 2) Difficulty

**Beginner** — Low risk. This project is entirely read-only against existing logs.

---

## 3) Execution Context

- **Primary path:** Host — Arch Linux bare metal (systemd journal available)
- **Alternate path:** VM — any systemd-based Linux VM
- All commands are read-only (`journalctl`, `grep`, `awk`, `sed` with no in-place edits).

---

## 4) Prerequisites

### Skills
- Basic shell usage
- Familiarity with pipes (`|`) and output redirection (`>`, `tee`)
- Basic `grep` pattern matching

### Tools
```bash
# All tools are part of core systemd and GNU coreutils
journalctl --version
grep --version | head -1
awk --version | head -1
sed --version | head -1
```

### Lab Environment
- `sudo` or `systemd-journal` group membership to read full journal
- Active sshd or other network service generating log entries (for meaningful examples)

Check journal access:
```bash
# If this shows entries, you have sufficient access
journalctl -n 5 --no-pager
```

If you see `No entries`, add your user to the journal group:
```bash
sudo usermod -aG systemd-journal "$USER"
# Log out and back in for the group change to take effect
```

---

## 5) Rollback Plan

> [!info] No rollback needed
> This project is entirely read-only. No system files are created, modified, or deleted.
> The only writes are to your local `evidence/b4/<TS>/` directory.

---

## 6) Project Plan

- **Phase A:** Initialise evidence directory
- **Phase B:** Basic journal navigation and filtering
- **Phase C:** Failed login detection
- **Phase D:** Service failure and error detection
- **Phase E:** Suspicious sudo and privilege escalation events
- **Phase F:** Build a repeatable triage summary
- **Phase G:** Generate hash manifest

---

## 7) Walkthrough

### Step 1 — Initialise evidence directory

```bash
TS="$(date -u +%Y%m%d_%H%M%SZ)"
OUT="evidence/b4/${TS}"
mkdir -p "$OUT"
echo "Evidence path: $OUT"
export OUT
```

Expected:
- Directory created; `$OUT` set

---

### Step 2 — Basic journal navigation

```bash
# How large is the journal?
journalctl --disk-usage | tee "$OUT/journal_disk_usage.txt"

# List all journal boots (system restarts)
journalctl --list-boots --no-pager | tee "$OUT/journal_boots.txt"

# Current boot — last 50 lines overview
journalctl -b --no-pager -n 50 -o short-iso | tee "$OUT/journal_current_boot_tail.txt"

# Kernel messages from current boot
journalctl -k -b --no-pager -o short-iso | tee "$OUT/journal_kernel_current_boot.txt"
```

Expected:
- Disk usage shows total journal size
- Boot list shows one or more boots with timestamps
- Kernel messages captured

---

### Step 3 — Detect failed SSH login attempts

```bash
# All SSH authentication failures in the current boot
sudo journalctl -u sshd -b --no-pager -o short-iso \
  | grep -iE "failed|invalid|error|refused" \
  | tee "$OUT/ssh_failures.txt"

# Count failed login attempts per source (if IP visible in log)
sudo journalctl -u sshd -b --no-pager -o short-iso \
  | grep -i "Failed password\|Invalid user" \
  | awk '{print $NF}' \
  | sort | uniq -c | sort -rn \
  | tee "$OUT/ssh_failure_source_count.txt"

# Successful authentications for contrast
sudo journalctl -u sshd -b --no-pager -o short-iso \
  | grep -iE "accepted|session opened" \
  | tee "$OUT/ssh_successes.txt"
```

Expected:
- `ssh_failures.txt` contains failed login events (may be empty on a fresh or isolated lab VM)
- `ssh_failure_source_count.txt` shows count-by-source sorted by frequency
- `ssh_successes.txt` shows successful logins

---

### Step 4 — Detect service failures and errors

```bash
# All errors and above from current boot
sudo journalctl -p err..emerg -b --no-pager -o short-iso \
  | tee "$OUT/journal_errors_current_boot.txt"

# Failed systemd units (services that did not start or crashed)
systemctl --state=failed --no-legend --no-pager \
  | tee "$OUT/systemd_failed_units.txt"

# For each failed unit, capture last 20 journal lines
while IFS= read -r unit; do
  unit_name=$(echo "$unit" | awk '{print $1}')
  [ -z "$unit_name" ] && continue
  journalctl -u "$unit_name" -b -n 20 --no-pager -o short-iso \
    > "$OUT/failed_unit_${unit_name}.txt" 2>&1
  echo "Captured: $unit_name"
done < "$OUT/systemd_failed_units.txt"

# Recent kernel OOM events (if any)
sudo journalctl -k -b --no-pager -o short-iso \
  | grep -i "oom\|out of memory\|killed process" \
  | tee "$OUT/kernel_oom_events.txt"
```

Expected:
- Error-level journal entries captured
- Failed unit list (may be empty on healthy system — that is a valid finding)
- Per-unit logs for any failed services
- OOM events (likely empty on a healthy lab system)

---

### Step 5 — Detect suspicious sudo and privilege escalation

```bash
# All sudo events (successful and failed) from current boot
sudo journalctl -b --no-pager -o short-iso \
  | grep -i "sudo\|su\[" \
  | tee "$OUT/sudo_events.txt"

# Failed sudo attempts specifically
sudo journalctl -b --no-pager -o short-iso \
  | grep -i "authentication failure\|incorrect password attempts\|not in sudoers" \
  | tee "$OUT/sudo_failures.txt"

# PAM authentication events (broader privilege/auth picture)
sudo journalctl -b --no-pager -o short-iso _TRANSPORT=syslog \
  | grep -i "pam\|auth" | head -50 \
  | tee "$OUT/pam_auth_events.txt"
```

Expected:
- `sudo_events.txt` shows all privilege escalation attempts
- `sudo_failures.txt` shows any unauthorised escalation attempts
- PAM events captured

---

### Step 6 — Detect unusual process and cron activity

```bash
# Cron job execution events
sudo journalctl -u cron -u crond -u anacron -b --no-pager -o short-iso 2>/dev/null \
  | tee "$OUT/cron_events.txt"

# Systemd timer activations (common cron replacement on Arch/systemd hosts)
sudo journalctl -b --no-pager -o short-iso \
  | grep -i "systemd\[1\].*timer\|started\|triggered" \
  | grep -i "timer" \
  | tee "$OUT/systemd_timer_events.txt"

# Boot messages — who logged in at startup
sudo journalctl -b --no-pager -o short-iso \
  | grep -iE "session opened|logged in|new session" \
  | tee "$OUT/session_open_events.txt"
```

Expected:
- Cron events captured (may be empty if no cron daemon is running)
- Systemd timer activations captured
- Session open events listed

---

### Step 7 — Build a triage summary report

Produce a concise single-file summary of key findings:

```bash
{
  echo "=== B4 LOG TRIAGE SUMMARY ==="
  echo "Run at: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
  echo "Host: $(hostname)"
  echo "Kernel: $(uname -r)"
  echo ""

  echo "--- Journal Size ---"
  cat "$OUT/journal_disk_usage.txt"
  echo ""

  echo "--- System Boots ---"
  cat "$OUT/journal_boots.txt"
  echo ""

  echo "--- Failed SSH Logins (count) ---"
  wc -l < "$OUT/ssh_failures.txt" | xargs echo "Total events:"
  cat "$OUT/ssh_failure_source_count.txt"
  echo ""

  echo "--- Successful SSH Logins (count) ---"
  wc -l < "$OUT/ssh_successes.txt" | xargs echo "Total events:"
  echo ""

  echo "--- Current-Boot Errors (count) ---"
  wc -l < "$OUT/journal_errors_current_boot.txt" | xargs echo "Total error lines:"
  echo ""

  echo "--- Failed systemd Units ---"
  cat "$OUT/systemd_failed_units.txt"
  echo ""

  echo "--- Sudo Events (count) ---"
  wc -l < "$OUT/sudo_events.txt" | xargs echo "Total sudo events:"
  echo ""

  echo "--- Sudo Failures ---"
  cat "$OUT/sudo_failures.txt"
  echo ""

  echo "--- OOM Events ---"
  cat "$OUT/kernel_oom_events.txt"
  echo ""

} | tee "$OUT/b4_triage_summary.txt"
```

Expected:
- `b4_triage_summary.txt` provides a human-readable overview
- Each section shows counts and notable events

---

## 8) Validation

```bash
# Confirm all key evidence files exist
for f in \
  journal_disk_usage.txt \
  journal_boots.txt \
  journal_current_boot_tail.txt \
  ssh_failures.txt \
  ssh_successes.txt \
  journal_errors_current_boot.txt \
  systemd_failed_units.txt \
  sudo_events.txt \
  sudo_failures.txt \
  b4_triage_summary.txt; do
  [ -f "$OUT/$f" ] && echo "PASS: $f" || echo "FAIL: $f missing"
done

# Confirm summary file is non-empty
[ -s "$OUT/b4_triage_summary.txt" ] \
  && echo "PASS: triage summary is non-empty" \
  || echo "FAIL: triage summary is empty"
```

Expected:
- All lines output `PASS`

---

## 9) Evidence

### Files to capture
- `evidence/b4/<TS>/journal_disk_usage.txt`
- `evidence/b4/<TS>/journal_boots.txt`
- `evidence/b4/<TS>/journal_current_boot_tail.txt`
- `evidence/b4/<TS>/journal_kernel_current_boot.txt`
- `evidence/b4/<TS>/ssh_failures.txt`
- `evidence/b4/<TS>/ssh_failure_source_count.txt`
- `evidence/b4/<TS>/ssh_successes.txt`
- `evidence/b4/<TS>/journal_errors_current_boot.txt`
- `evidence/b4/<TS>/systemd_failed_units.txt`
- `evidence/b4/<TS>/sudo_events.txt`
- `evidence/b4/<TS>/sudo_failures.txt`
- `evidence/b4/<TS>/pam_auth_events.txt`
- `evidence/b4/<TS>/cron_events.txt`
- `evidence/b4/<TS>/session_open_events.txt`
- `evidence/b4/<TS>/b4_triage_summary.txt`

> [!warning] Before committing to git
> Review all evidence files for real IP addresses, hostnames, usernames, or other identifying information. Replace sensitive values with placeholders before committing. Consider excluding the `evidence/` directory from git entirely.

### Generate hash manifest
```bash
find "$OUT" -type f ! -name 'SHA256SUMS.txt' -print0 \
  | xargs -0 sha256sum | tee "$OUT/SHA256SUMS.txt"

sha256sum --check "$OUT/SHA256SUMS.txt" && echo "Manifest verified OK"
```

---

## 10) Failure Modes & Recovery

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `journalctl` shows `No entries` | User not in `systemd-journal` group, or journal not persistent | Add user to group; or use `sudo journalctl`; check `journalctl --disk-usage` |
| `journalctl -u sshd` returns nothing | `sshd` is not running or uses a different service name | Check `systemctl status sshd ssh openssh` to find the correct unit name |
| `grep` pattern returns no matches | No events matching the pattern exist (valid finding) | Record "no matching events found" as a finding in the summary — absence of alerts is itself evidence |
| `awk` output is empty | Different log format (e.g., no source IP in log line) | Adjust the `awk` field number (`$NF`, `$(NF-1)`, etc.) to match the actual log format |
| Journal is very large (>1 GB) | Persistent journal accumulating over long uptime | Add `--since "48 hours ago"` to narrow queries; consider configuring `SystemMaxUse` in `journald.conf` |
| `systemd_failed_units.txt` is empty | No services have failed (healthy system) | This is a valid result — document it in the report as "no failed units detected" |

---

## 11) Sources

- [SRC-SYSTEMD-JOURNALCTL-01] — journalctl(1) man page — https://www.freedesktop.org/software/systemd/man/journalctl.html
- [SRC-GNU-GREP-01] — GNU grep manual — https://www.gnu.org/software/grep/manual/grep.html
- [SRC-GNU-AWK-01] — GNU awk (gawk) manual — https://www.gnu.org/software/gawk/manual/gawk.html
- [SRC-ARCH-SYSTEMD-01] — Arch Wiki: systemd — https://wiki.archlinux.org/title/Systemd/Journal

---

## 12) Stretch Goals

- Write a reusable `triage.sh` script that accepts a `--since` argument and runs all Phase C–F queries automatically
- Add detection for `su` escalation events alongside `sudo`
- Extend the triage to cover `/var/log/audit/audit.log` if `auditd` is installed
- Map detected event types to MITRE ATT&CK techniques (e.g., failed SSH → T1110 Brute Force, sudo failures → T1548.003 Sudo and Sudo Caching)
- Set up a persistent journal (`Storage=persistent` in `/etc/systemd/journald.conf`) and compare cross-boot patterns

---

## Report Checklist

Fill in before marking B4 complete:

- [ ] Execution context declared: _____________________ (Host / VM)
- [ ] Evidence directory created: `evidence/b4/<TS>/`
- [ ] Journal size and boot history captured
- [ ] SSH failure and success events queried and captured
- [ ] Service error and failed-unit events captured
- [ ] Sudo and PAM events captured
- [ ] Cron and systemd timer events captured
- [ ] Triage summary report generated: `b4_triage_summary.txt`
- [ ] Key findings (or confirmed absence of findings) noted: _____________________
- [ ] All validation checks output `PASS`
- [ ] `SHA256SUMS.txt` generated and verified
- [ ] Evidence reviewed for sensitive data before any potential commit
- [ ] UTC start time: _____________________ — UTC end time: _____________________
