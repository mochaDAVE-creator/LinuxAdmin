---
title: B2 - FHS Artifact Hunt
aliases:
  - b2-fhs-artifact-hunt
tags:
  - beginner
  - fhs
  - forensics
  - triage
  - evidence
date: 2026-05-10
---

# B2 — FHS Artifact Hunt

## 1) Mission

- **Problem statement:** Operators who do not know where Linux stores critical files waste time during incidents searching for logs, configs, and process state. This project builds a mental map of the Filesystem Hierarchy Standard (FHS) by locating real artifacts on your lab system.
- **Why it matters:** Rapid artifact location is the first step in any triage. Knowing that auth logs are in `/var/log/auth.log` (or in `journalctl` on systemd hosts) and that process state lives in `/proc` reduces incident response time significantly.

---

## 2) Difficulty

**Beginner** — Low risk. This project is entirely read-only. No files are modified.

---

## 3) Execution Context

- **Primary path:** Host — Arch Linux bare metal
- **Alternate path:** VM — any Linux VM where you have read access to `/proc`, `/var/log`, and `/etc`
- All commands are non-destructive (`find`, `ls`, `stat`, `cat`, `sha256sum`).

---

## 4) Prerequisites

### Skills
- Basic shell navigation
- Understanding of file ownership and permissions
- Ability to read `man` pages

### Tools
```bash
# All tools should be present on any standard Arch/Linux system
find --version
stat --version
sha256sum --version
```

### Lab Environment
- `sudo` or read access to `/var/log`, `/proc`, and `/etc`
- Arch Linux host or any systemd-based Linux VM

---

## 5) Rollback Plan

> [!info] No rollback needed
> This project is entirely read-only. No system files are created, modified, or deleted.
> The only writes are to your local `evidence/b2/<TS>/` directory.
>
> If you are running on a host with Snapper configured, you may optionally create a snapshot before starting as a general habit drill — but it is not required for this project.

---

## 6) Project Plan

- **Phase A:** Initialise evidence directory
- **Phase B:** Map key FHS locations and collect artifact metadata
- **Phase C:** Capture `/proc` process and socket state
- **Phase D:** Identify log sources and capture recent entries
- **Phase E:** Generate hash manifest

---

## 7) Walkthrough

### Step 1 — Initialise evidence directory

```bash
TS="$(date -u +%Y%m%d_%H%M%SZ)"
OUT="evidence/b2/${TS}"
mkdir -p "$OUT"
echo "Evidence path: $OUT"
export OUT
```

Expected:
- Directory created without errors
- `$OUT` is set and non-empty

---

### Step 2 — Map key FHS directories

Capture the top-level layout and annotate what each directory is for:

```bash
# Top-level FHS layout
ls -la / | tee "$OUT/fhs_root_listing.txt"

# Key directories — existence and ownership
stat /bin /sbin /usr /etc /var /tmp /home /root /proc /sys /run \
  | tee "$OUT/fhs_key_dirs_stat.txt"
```

Expected:
- Standard FHS directories visible
- Stat output shows owner, permissions, and inode numbers

---

### Step 3 — Locate configuration artifacts in /etc

```bash
# Find all files modified in the last 7 days (recent config changes)
sudo find /etc -type f -newer /etc/hostname \
  -printf '%TY-%Tm-%Td %TH:%TM  %p\n' 2>/dev/null \
  | sort | tee "$OUT/etc_recent_files.txt"

# Capture sshd config (if present) — no secrets, just structure
if [ -f /etc/ssh/sshd_config ]; then
  sudo grep -v '^\s*#' /etc/ssh/sshd_config | grep -v '^\s*$' \
    | tee "$OUT/sshd_config_effective.txt"
fi

# Capture sudoers structure (group membership, not credentials)
sudo cat /etc/sudoers 2>/dev/null | grep -v '^\s*#' | grep -v '^\s*$' \
  | tee "$OUT/sudoers_effective.txt" || echo "No read access to /etc/sudoers" \
  | tee "$OUT/sudoers_access.txt"
```

Expected:
- Recently modified `/etc` files listed
- `sshd_config_effective.txt` shows active (non-comment) directives
- `sudoers_effective.txt` captures group-level sudo grants

---

### Step 4 — Explore /proc for live process artifacts

```bash
# Current process list with parent relationships
ps auxf | tee "$OUT/proc_ps_auxf.txt"

# Open network sockets
ss -tulpen | tee "$OUT/proc_sockets.txt"

# Kernel version and uptime
uname -a | tee "$OUT/kernel_version.txt"
uptime | tee "$OUT/uptime.txt"

# Loaded kernel modules
lsmod | tee "$OUT/lsmod.txt"

# Mounted filesystems
mount | tee "$OUT/mounts.txt"
cat /proc/mounts | tee "$OUT/proc_mounts.txt"
```

Expected:
- Process tree visible in `ps_auxf.txt`
- Listening ports captured in `proc_sockets.txt`
- All files saved to `$OUT`

---

### Step 5 — Locate log artifacts in /var/log and journald

```bash
# List log files with sizes and modification times
sudo find /var/log -type f -printf '%s\t%TY-%Tm-%Td %TH:%TM\t%p\n' 2>/dev/null \
  | sort -rn | head -30 | tee "$OUT/varlog_file_list.txt"

# Recent authentication events (systemd journal)
sudo journalctl -u sshd --since "24 hours ago" --no-pager -o short-iso \
  | tee "$OUT/journal_sshd_24h.txt"

# Recent failed logins (if auth.log exists — Debian-style systems)
if [ -f /var/log/auth.log ]; then
  grep -i "failed\|invalid\|error" /var/log/auth.log | tail -50 \
    | tee "$OUT/auth_log_failures.txt"
fi

# systemd journal errors from current boot
sudo journalctl -p err..emerg -b --no-pager -o short-iso \
  | tee "$OUT/journal_errors_boot.txt"
```

Expected:
- Log file list shows sizes, helping prioritise which logs to review
- SSH daemon events captured for the last 24 hours
- Journal errors from current boot captured

---

### Step 6 — Locate user home directories and check shell history paths

```bash
# List home directories and their ownership
ls -la /home/ | tee "$OUT/home_dir_listing.txt"

# Check current user's shell history location (not contents — just path and size)
HISTFILE_PATH="${HISTFILE:-$HOME/.bash_history}"
if [ -f "$HISTFILE_PATH" ]; then
  stat "$HISTFILE_PATH" | tee "$OUT/shell_history_stat.txt"
else
  echo "No history file at $HISTFILE_PATH" | tee "$OUT/shell_history_stat.txt"
fi

# Check for other user accounts with login shells
grep -v 'nologin\|false' /etc/passwd | tee "$OUT/users_with_shells.txt"
```

Expected:
- Home directory listing shows owners and permissions
- Shell history stat shows file size (useful for anomaly hunting, not reading content)
- User list filtered to accounts with interactive shells

---

### Step 7 — Capture /tmp and /run state

```bash
# /tmp — world-writable, often used for staging
find /tmp -maxdepth 2 -printf '%M %u %g %s %TY-%Tm-%Td %p\n' 2>/dev/null \
  | tee "$OUT/tmp_listing.txt"

# /run — runtime state (sockets, PIDs, lock files)
find /run -maxdepth 2 -type f -printf '%M %u %g %s %p\n' 2>/dev/null \
  | head -50 | tee "$OUT/run_listing.txt"
```

Expected:
- `/tmp` listing shows any files staged there (typically empty on a clean system)
- `/run` listing shows active runtime files

---

## 8) Validation

```bash
# Verify all expected evidence files exist
for f in \
  fhs_root_listing.txt \
  fhs_key_dirs_stat.txt \
  etc_recent_files.txt \
  proc_ps_auxf.txt \
  proc_sockets.txt \
  kernel_version.txt \
  varlog_file_list.txt \
  journal_sshd_24h.txt \
  journal_errors_boot.txt \
  home_dir_listing.txt \
  users_with_shells.txt \
  tmp_listing.txt; do
  [ -f "$OUT/$f" ] && echo "PASS: $f" || echo "FAIL: $f missing"
done
```

Expected:
- All lines output `PASS`

---

## 9) Evidence

### Files to capture
- `evidence/b2/<TS>/fhs_root_listing.txt`
- `evidence/b2/<TS>/fhs_key_dirs_stat.txt`
- `evidence/b2/<TS>/etc_recent_files.txt`
- `evidence/b2/<TS>/sshd_config_effective.txt` (if sshd present)
- `evidence/b2/<TS>/sudoers_effective.txt` or `sudoers_access.txt`
- `evidence/b2/<TS>/proc_ps_auxf.txt`
- `evidence/b2/<TS>/proc_sockets.txt`
- `evidence/b2/<TS>/kernel_version.txt`
- `evidence/b2/<TS>/uptime.txt`
- `evidence/b2/<TS>/lsmod.txt`
- `evidence/b2/<TS>/mounts.txt`
- `evidence/b2/<TS>/varlog_file_list.txt`
- `evidence/b2/<TS>/journal_sshd_24h.txt`
- `evidence/b2/<TS>/journal_errors_boot.txt`
- `evidence/b2/<TS>/home_dir_listing.txt`
- `evidence/b2/<TS>/shell_history_stat.txt`
- `evidence/b2/<TS>/users_with_shells.txt`
- `evidence/b2/<TS>/tmp_listing.txt`
- `evidence/b2/<TS>/run_listing.txt`

> [!warning] Before committing to git
> Review all evidence files for private IP addresses, hostnames, real usernames, or any sensitive data before committing. Replace with `<REDACTED>` or use a `.gitignore` rule to exclude the `evidence/` directory from the repository.

### Generate hash manifest
```bash
find "$OUT" -type f ! -name 'SHA256SUMS.txt' -print0 \
  | xargs -0 sha256sum | tee "$OUT/SHA256SUMS.txt"

# Verify the manifest
sha256sum --check "$OUT/SHA256SUMS.txt" && echo "Manifest verified OK"
```

---

## 10) Failure Modes & Recovery

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `find /etc` returns many "Permission denied" errors | Missing `sudo` | Re-run with `sudo find /etc ...` |
| `journalctl` returns no output | Journal not persistent or service not logging | Check `journalctl --disk-usage`; verify `Storage=persistent` in `/etc/systemd/journald.conf` |
| `/var/log/auth.log` not found | Arch uses systemd journal, not syslog | Use `journalctl` commands instead (already covered in Step 5) |
| `ss` command not found | `iproute2` not installed | `sudo pacman -S iproute2` |
| `stat` on `/proc` files returns unusual sizes | `/proc` is a virtual filesystem — sizes are always 0 | Expected behaviour; use `cat` to read content |

---

## 11) Sources

- [SRC-FHS-3.0-01] — Filesystem Hierarchy Standard 3.0 — https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html
- [SRC-MAN7-HIER-01] — hier(7) Linux man page — https://man7.org/linux/man-pages/man7/hier.7.html
- [SRC-PROCFS-01] — proc(5) Linux man page — https://man7.org/linux/man-pages/man5/proc.5.html
- [SRC-SYSTEMD-JOURNALCTL-01] — journalctl(1) man page — https://www.freedesktop.org/software/systemd/man/journalctl.html

---

## 12) Stretch Goals

- Extend the artifact hunt to `/sys` (sysfs): capture USB devices, block devices, and network interface state
- Automate the hunt as a shell script that takes `$OUT` as an argument and runs all steps non-interactively
- Compare your artifact list against the SANS Linux triage checklist or a CIS benchmark artifact inventory
- Map each artifact location to a MITRE ATT&CK technique (e.g., `/tmp` staging → T1074.001)

---

## Report Checklist

Fill in before marking B2 complete:

- [ ] Execution context declared: _____________________ (Host / VM)
- [ ] Evidence directory created: `evidence/b2/<TS>/`
- [ ] FHS root listing captured
- [ ] `/etc` recent-files list captured
- [ ] `/proc` process and socket state captured
- [ ] Log sources identified and sampled (journal + `/var/log`)
- [ ] User accounts with login shells identified
- [ ] All validation checks output `PASS`
- [ ] `SHA256SUMS.txt` generated and verified
- [ ] Evidence files reviewed for sensitive data before any potential commit
- [ ] UTC start time: _____________________ — UTC end time: _____________________
