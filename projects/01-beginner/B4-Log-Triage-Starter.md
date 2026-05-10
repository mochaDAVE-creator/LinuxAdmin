---
title: B4 - Log Triage Starter
aliases:
  - b4-log-triage
  - journalctl-triage-beginner
tags:
  - beginner
  - journalctl
  - logs
  - triage
  - forensics
  - evidence
  - net412
  - net179
date: 2026-05-10
---

# B4 — Log Triage Starter

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host running systemd. All commands are read-only against system logs and journal files. No services are modified.
> **Risk level:** Low — read-only log investigation. No configuration changes.
> **Threat model:** An administrator who cannot efficiently query and parse system logs is blind to both operational problems and security events. This project builds fluency with `journalctl`, `grep`, `awk`, and `sed` as the primary log triage toolkit.
> **Privacy note:** System logs may contain usernames, IP addresses, email addresses, and other PII. Keep the `evidence/b4/` directory on local storage only. Do not commit raw log exports to public repositories — only sanitized summaries are appropriate.
> **Out of scope:** Log forwarding, SIEM ingestion, persistent storage configuration. Those are covered in A1 (Centralized Logging Pipeline).

## 1) Mission

- **Problem statement:** When a system behaves unexpectedly — service crash, failed login attempt, unexpected reboot, filesystem error — the logs contain the answer. Most operators know `tail -f /var/log/syslog` but cannot efficiently navigate systemd journal, filter by priority, correlate events across units, or extract structured data with `awk`. This project closes that gap.
- **Why it matters:** Log triage is the first action in every incident response workflow. NET 179 (Digital Forensics) and NET 412 (Linux Admin) both test log analysis skills. The `journalctl` filters learned here are used directly in A1, I2, E1, and E4.

## 2) Difficulty

- Beginner (estimated 3–4 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal with systemd ≥ 245. All commands run against the local journal. Persistent journal storage should be enabled (see Step 1).

## 4) Prerequisites

- **Skills:** Basic terminal usage, understanding of what a daemon/service is, familiarity with reading timestamped output.
- **Tools:** `journalctl`, `grep`, `awk`, `sed`, `cut`, `sort`, `uniq`, `tee`, `systemctl`
- **Dependencies:**
  - systemd running and journal accessible: `systemctl status systemd-journald`
  - Persistent journal configured (check `Storage=` in `/etc/systemd/journald.conf`)

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-b4-log-triage"`
- **Host Snapper post:** Not required — no host changes. Create for consistency: `sudo snapper -c root create --description "post-b4-log-triage"`
- **VM snapshot:** N/A
- **Container rebuild command:** N/A
- **Rollback trigger:** If journald configuration is modified (Step 1), and causes unexpected behavior, run: `sudo snapper -c root undochange <PRE>..<POST>` and restart journald.

## 6) Project Plan

- **Phase A — Journal Setup Verification:** Confirm persistent logging is enabled and understand journal storage layout.
- **Phase B — Core journalctl Queries:** Learn and apply the most important filtering, output, and export options.
- **Phase C — Text Processing:** Apply `grep`, `awk`, and `sed` pipelines to extract structured data from log output.

## 7) Walkthrough

### Step 1 — Verify and enable persistent journal storage

```bash
mkdir -p evidence/b4

# Check current journal storage mode
grep -i "^Storage" /etc/systemd/journald.conf 2>/dev/null \
  | tee evidence/b4/journald_storage_setting.txt \
  || echo "Storage= not explicitly set (default: auto)" | tee evidence/b4/journald_storage_setting.txt

# Show journal disk usage
journalctl --disk-usage | tee evidence/b4/journal_disk_usage.txt

# List journal files on disk
ls -lah /var/log/journal/ 2>/dev/null | tee evidence/b4/journal_files_listing.txt \
  || echo "/var/log/journal not present — journal may be volatile" | tee evidence/b4/journal_files_listing.txt
```

If persistent storage is not enabled:

```bash
# Enable persistent journal storage
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal

# Optional: set explicit Storage=persistent in journald.conf
sudo sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf

# Flush current runtime journal to disk
sudo journalctl --flush

# Verify
journalctl --disk-usage | tee -a evidence/b4/journal_disk_usage.txt
```

Expected:
- `journal_disk_usage.txt` shows a non-zero disk usage value (e.g., `Archived and active journals take up X.XM in the file system.`).
- `/var/log/journal/` directory exists and contains subdirectories with `.journal` files.

### Step 2 — Basic journalctl navigation

```bash
# Show all journal entries from the current boot (most common starting point)
journalctl -b --no-pager | wc -l | tee evidence/b4/journal_current_boot_linecount.txt

# Show journal entries from previous boot (if available)
journalctl -b -1 --no-pager 2>/dev/null | wc -l \
  | tee evidence/b4/journal_previous_boot_linecount.txt \
  || echo "No previous boot journal available" | tee evidence/b4/journal_previous_boot_linecount.txt

# List available boot IDs
journalctl --list-boots | tee evidence/b4/journal_boot_list.txt

# Show only kernel messages from current boot (equivalent to dmesg)
journalctl -b -k --no-pager | tee evidence/b4/journal_kernel_messages.txt

# Show entries since a specific time
journalctl -b --since "today" --no-pager | tee evidence/b4/journal_today.txt
```

Expected:
- `journal_current_boot_linecount.txt` shows total log lines in current boot — could be thousands.
- `journal_boot_list.txt` lists all available boot records with their boot IDs and timestamps.

### Step 3 — Filter by priority (severity level)

```bash
# Emerg=0, Alert=1, Crit=2, Err=3, Warning=4, Notice=5, Info=6, Debug=7
# Show only errors and above from current boot
journalctl -b -p err --no-pager | tee evidence/b4/journal_errors_only.txt

# Show warnings and above
journalctl -b -p warning --no-pager | tee evidence/b4/journal_warnings_plus.txt

# Count errors by unit/service
journalctl -b -p err --no-pager -o json \
  | awk -F'"' '/_SYSTEMD_UNIT/{u=$4} /MESSAGE/{m=$4} u && m {print u": "m; u=""; m=""}' \
  | sort | uniq -c | sort -rn \
  | head -20 | tee evidence/b4/journal_error_summary_by_unit.txt

echo "Total error-level entries: $(wc -l < evidence/b4/journal_errors_only.txt)" \
  | tee -a evidence/b4/journal_errors_only.txt
```

Expected:
- `journal_errors_only.txt` contains only `err`, `crit`, `alert`, and `emerg` level entries.
- `journal_error_summary_by_unit.txt` ranks which units are generating the most errors.

### Step 4 — Filter by unit (service-specific logs)

```bash
# Query SSH daemon logs
sudo journalctl -u sshd -b --no-pager | tee evidence/b4/journal_sshd.txt

# Query NetworkManager logs
journalctl -u NetworkManager -b --no-pager | tee evidence/b4/journal_networkmanager.txt

# Query kernel firewall/audit messages
journalctl -b -k --grep="denied\|DROP\|REJECT" --no-pager | tee evidence/b4/journal_firewall_drops.txt

# Query Btrfs-related kernel messages
journalctl -b -k --grep="btrfs\|BTRFS" --no-pager | tee evidence/b4/journal_btrfs_messages.txt

# Query any systemd unit failures
journalctl -b --no-pager \
  | grep -i "failed\|failure\|error\|killed" \
  | tee evidence/b4/journal_failure_keywords.txt

echo "Lines with failure keywords: $(wc -l < evidence/b4/journal_failure_keywords.txt)" \
  | tee -a evidence/b4/journal_failure_keywords.txt
```

Expected:
- `journal_sshd.txt` shows SSH daemon startup, connection events, and authentication results.
- `journal_firewall_drops.txt` may be empty on a standard desktop install without explicit firewall rules — that is expected.

### Step 5 — Authentication and security event triage

```bash
# Extract failed authentication events
sudo journalctl -b --no-pager \
  | grep -iE "authentication failure|failed password|invalid user|connection closed by invalid user" \
  | tee evidence/b4/auth_failures.txt

# Extract successful su/sudo events
sudo journalctl -b --no-pager \
  | grep -iE "session opened|sudo:|su\[" \
  | tee evidence/b4/auth_successes.txt

# Extract SSH key fingerprints used for authentication
sudo journalctl -u sshd -b --no-pager \
  | grep -i "Accepted publickey" \
  | tee evidence/b4/ssh_key_logins.txt

# Count failed auth attempts by source IP (extracts IPs from sshd logs)
sudo journalctl -u sshd -b --no-pager \
  | grep -oP "(?<=from )\d+\.\d+\.\d+\.\d+" \
  | sort | uniq -c | sort -rn \
  | tee evidence/b4/ssh_source_ips.txt

echo "Unique source IPs in SSH log: $(wc -l < evidence/b4/ssh_source_ips.txt)"
```

Expected:
- `auth_failures.txt` may be empty on a freshly configured system — that is correct.
- `ssh_source_ips.txt` shows IP addresses and attempt counts for any SSH connections.

### Step 6 — Time-range queries and log export

```bash
# Query last 24 hours (useful for daily review)
journalctl --since "24 hours ago" --until "now" --no-pager \
  | tee evidence/b4/journal_last_24h.txt | wc -l

# Export in structured JSON format for tooling
sudo journalctl -b -p err --no-pager -o json \
  | head -20 | tee evidence/b4/journal_errors_json_sample.txt

# Export in short-precise format (includes full timestamps)
journalctl -b --no-pager -o short-precise | head -50 \
  | tee evidence/b4/journal_precise_timestamps_sample.txt

# Export all current boot logs to a flat file for offline analysis
journalctl -b --no-pager > evidence/b4/journal_full_boot_export.txt
echo "Full boot export size: $(du -h evidence/b4/journal_full_boot_export.txt)"
```

Expected:
- JSON output shows structured key-value pairs including `__REALTIME_TIMESTAMP`, `MESSAGE`, `_SYSTEMD_UNIT`, `_PID`, etc.
- `journal_precise_timestamps_sample.txt` shows microsecond-precision timestamps.

### Step 7 — awk and sed log parsing pipelines

```bash
# Extract only timestamps and messages from sshd logs using awk
sudo journalctl -u sshd -b --no-pager -o short-precise \
  | awk '{ts=$1" "$2; $1=""; $2=""; print ts $0}' \
  | tee evidence/b4/sshd_parsed_timestamps.txt

# Count log entries per hour for current boot using awk
journalctl -b --no-pager -o short-precise \
  | awk '{split($1, a, "T"); split(a[2], b, ":"); print b[1]}' \
  | sort | uniq -c \
  | tee evidence/b4/journal_entries_per_hour.txt

# Use sed to redact IP addresses from auth log (for sanitized report)
cat evidence/b4/auth_failures.txt \
  | sed 's/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/[REDACTED_IP]/g' \
  | tee evidence/b4/auth_failures_sanitized.txt

# Extract unique usernames from failed sudo attempts
sudo journalctl -b --no-pager \
  | grep "sudo:" \
  | grep -oP "(?<=sudo: )[a-z_][a-z0-9_-]*" \
  | sort -u \
  | tee evidence/b4/sudo_users.txt
```

Expected:
- `journal_entries_per_hour.txt` shows the log volume distribution across hours — useful for spotting anomalous bursts.
- `auth_failures_sanitized.txt` has all IP addresses replaced with `[REDACTED_IP]` — safe for sharing or committing.

## 8) Validation

```bash
# Confirm all key evidence files exist and are non-empty
evidence_files=(
  "journal_current_boot_linecount.txt"
  "journal_boot_list.txt"
  "journal_errors_only.txt"
  "journal_sshd.txt"
  "auth_failures.txt"
  "journal_errors_json_sample.txt"
  "journal_entries_per_hour.txt"
)

for f in "${evidence_files[@]}"; do
  if [ -s "evidence/b4/$f" ]; then
    echo "PASS (non-empty): evidence/b4/$f"
  else
    echo "WARN (empty or missing): evidence/b4/$f"
  fi
done

# Confirm journalctl is accessible and functional
journalctl -b --no-pager -n 1 > /dev/null && echo "PASS: journalctl accessible" || echo "FAIL: journalctl unavailable"

# Summary: count evidence files
echo "Total evidence files: $(find evidence/b4 -type f | wc -l)"
```

## 9) Evidence

- **Output files:**
  - `evidence/b4/journald_storage_setting.txt` — journal persistence configuration
  - `evidence/b4/journal_disk_usage.txt` — journal storage size
  - `evidence/b4/journal_files_listing.txt` — journal file inventory
  - `evidence/b4/journal_current_boot_linecount.txt` — total log line count
  - `evidence/b4/journal_boot_list.txt` — available boot records
  - `evidence/b4/journal_kernel_messages.txt` — kernel log for current boot
  - `evidence/b4/journal_today.txt` — today's log entries
  - `evidence/b4/journal_errors_only.txt` — error-priority entries
  - `evidence/b4/journal_warnings_plus.txt` — warning and above entries
  - `evidence/b4/journal_error_summary_by_unit.txt` — error count per unit
  - `evidence/b4/journal_sshd.txt` — SSH daemon log
  - `evidence/b4/journal_networkmanager.txt` — network manager log
  - `evidence/b4/journal_firewall_drops.txt` — firewall/drop messages
  - `evidence/b4/journal_btrfs_messages.txt` — Btrfs kernel messages
  - `evidence/b4/journal_failure_keywords.txt` — failure keyword hits
  - `evidence/b4/auth_failures.txt` — authentication failure events
  - `evidence/b4/auth_successes.txt` — authentication success events
  - `evidence/b4/ssh_key_logins.txt` — SSH public key login events
  - `evidence/b4/ssh_source_ips.txt` — SSH source IP count table
  - `evidence/b4/journal_last_24h.txt` — last 24 hour log export
  - `evidence/b4/journal_errors_json_sample.txt` — JSON format sample
  - `evidence/b4/journal_precise_timestamps_sample.txt` — precise timestamp sample
  - `evidence/b4/journal_full_boot_export.txt` — full boot log export
  - `evidence/b4/sshd_parsed_timestamps.txt` — awk-parsed sshd log
  - `evidence/b4/journal_entries_per_hour.txt` — hourly log volume
  - `evidence/b4/auth_failures_sanitized.txt` — IP-redacted auth failures (safe to share)
  - `evidence/b4/sudo_users.txt` — users who invoked sudo
- **Hash manifest:**

```bash
find evidence/b4 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/b4/SHA256SUMS.txt
cat evidence/b4/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `journalctl` shows `No journal files were found.`
  - **Cause:** systemd-journald is not running or journal storage is not initialized.
  - **Fix:** `sudo systemctl start systemd-journald && sudo systemd-tmpfiles --create --prefix /var/log/journal`
  - **Rollback trigger:** N/A — no host changes made.

- **Symptom:** Many entries show `Message from sysklogd... truncated` or similar artifacts.
  - **Cause:** Both rsyslog/syslog-ng and journald are running simultaneously, causing duplicate or malformed entries.
  - **Fix:** On Arch, prefer journald-only. If syslog daemon is installed: `sudo systemctl disable --now rsyslog syslog-ng 2>/dev/null` then restart journald.
  - **Rollback trigger:** Snapper if syslog config was modified.

- **Symptom:** `journalctl -o json` produces malformed output that breaks awk parsing.
  - **Cause:** Binary or non-UTF8 data in log messages breaks JSON parsing.
  - **Fix:** Add `--no-hostname` and use Python's `json.loads()` for robust parsing: `journalctl -o json --no-pager | python3 -c "import sys, json; [print(json.loads(l).get('MESSAGE','')) for l in sys.stdin]"`
  - **Rollback trigger:** N/A.

## 11) Sources

- [systemd journalctl man page](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [systemd journald.conf man page](https://man7.org/linux/man-pages/man5/journald.conf.5.html)
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU awk (gawk) manual](https://www.gnu.org/software/gawk/manual/gawk.html)
- [Arch Wiki — systemd/Journal](https://wiki.archlinux.org/title/Systemd/Journal)
- [CISA — Logging and Monitoring](https://www.cisa.gov/sites/default/files/2023-03/CISA_Logging_Made_Easy.pdf)
- [NIST SP 800-92 — Guide to Computer Security Log Management](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-92.pdf)

## 12) Stretch Goals

- Write a `b4-daily-triage.sh` script that runs each morning, queries the previous day's journal for errors, auth failures, and failed units, and outputs a concise markdown report to `evidence/b4/daily-<date>.md`.
- Configure `journald.conf` with `RateLimitBurst` and `RateLimitIntervalSec` tuning and document the impact on log completeness vs disk pressure.
- Export journal logs from a Proxmox VM using `journalctl --remote` or SSH log forwarding and compare the same triage workflow against a remote guest's journal.
- Install `lnav` (the Log Navigator) and use its interactive filtering to explore the journal export file (`journal_full_boot_export.txt`). Document three queries that would not be easy to write with raw `journalctl`.
