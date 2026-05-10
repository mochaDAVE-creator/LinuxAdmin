---
title: I2 - Persistence Detection Sweep
aliases:
  - i2-persistence-detection
  - persistence-sweep-intermediate
tags:
  - intermediate
  - persistence
  - detection
  - forensics
  - mitre-attack
  - evidence
  - net179
  - net377
date: 2026-05-10
---

# I2 — Persistence Detection Sweep

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host. All commands are read-only (observation only). The sweep is designed to *detect* persistence mechanisms, not to install or simulate them. This is a defensive audit — no attacker tooling is used.
> **Risk level:** Medium — some queries require root to inspect protected paths. No files are modified.
> **Threat model:** An attacker who gains initial access will typically install at least one persistence mechanism to survive reboots. MITRE ATT&CK TA0003 (Persistence) documents the most common techniques. This sweep maps to the following sub-techniques: T1547.001 (Registry Run Keys/autostart), T1543.002 (Systemd Service), T1053.003 (Cron), T1098.004 (SSH Authorized Keys), T1546.004 (Unix Shell Profile).
> **Out of scope:** Memory-resident persistence (rootkits, kernel implants), firmware persistence. Those require specialized forensics tools beyond this project scope.

## 1) Mission

- **Problem statement:** After any suspected compromise or during routine security audits, an administrator must know how to rapidly survey all common persistence locations on a Linux system and identify unauthorized entries. Missing a single persistence mechanism means an attacker survives remediation.
- **Why it matters:** This skill is tested in NET 179 (Digital Forensics) and NET 377 (Ethical Hacking). The sweep methodology built here feeds directly into E1 (IR Playbook Automation) and E4 (Purple-Team Validation Cycle).

## 2) Difficulty

- Intermediate (estimated 5–8 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal. All inspection is local. No network-based scanning.

## 4) Prerequisites

- **Skills:** Completion of B2 (FHS Artifact Hunt) and B4 (Log Triage Starter). Understanding of systemd unit types and cron syntax. Familiarity with SSH authorized_keys.
- **Tools:** `find`, `ls`, `stat`, `grep`, `awk`, `systemctl`, `journalctl`, `crontab`, `getent`, `sha256sum`, `tee`
- **Dependencies:** `sudo` access; `pacman` for package verification queries.

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-i2-persistence-sweep"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-i2-persistence-sweep"`
- **VM snapshot:** If sweeping a Proxmox VM, take a VM snapshot before any remediation actions.
- **Container rebuild command:** N/A
- **Rollback trigger:** This project is read-only. Rollback is only needed if you act on sweep findings. If you remove a suspected persistence mechanism and it breaks something legitimate, use `undochange` to restore.

## 6) Project Plan

- **Phase A — Systemd Persistence:** Survey all systemd unit files in user and system locations, identify non-package-owned units.
- **Phase B — Scheduled Task Persistence:** Check all cron and timer locations for unauthorized scheduled tasks.
- **Phase C — Shell Profile and Startup Persistence:** Inspect all shell initialization files for injected commands.
- **Phase D — SSH Authorized Keys:** Audit all `authorized_keys` files across users.
- **Phase E — SUID/SGID and PATH Hijacking:** Find setuid binaries and unexpected PATH entries.

## 7) Walkthrough

### Step 1 — Setup and Snapper pre-snapshot

```bash
mkdir -p evidence/i2

# Record sweep start time
date -u | tee evidence/i2/sweep_start_time.txt

# Capture host inventory for context
uname -a | tee evidence/i2/env_uname.txt
hostname | tee evidence/i2/env_hostname.txt
getent passwd | awk -F: '$7 ~ /bash|zsh|sh/ {print $1, $3, $6, $7}' \
  | tee evidence/i2/shell_users.txt

# Snapper pre-snapshot
sudo snapper -c root create --description "pre-i2-persistence-sweep" --print-number \
  | tee evidence/i2/snapshot_pre_num.txt
```

Expected:
- `shell_users.txt` lists all users with interactive shells — these are the accounts that could have persistence installed in their home directories.

### Step 2 — Phase A: Systemd persistence locations

```bash
# Survey system-level unit locations
echo "=== /etc/systemd/system/ ===" | tee evidence/i2/systemd_units_system.txt
find /etc/systemd/system -maxdepth 2 -type f -name "*.service" -o -name "*.timer" \
  | sort | tee -a evidence/i2/systemd_units_system.txt

# Survey user-level unit locations for all users
echo "=== User-level unit files ===" | tee evidence/i2/systemd_units_user.txt
find /home -name "*.service" -o -name "*.timer" 2>/dev/null \
  | sort | tee -a evidence/i2/systemd_units_user.txt
find /root/.config/systemd/user -maxdepth 2 -type f 2>/dev/null \
  | sort | tee -a evidence/i2/systemd_units_user.txt

# For each system unit, check whether it is owned by a package
echo "=== Package ownership of system units ===" | tee evidence/i2/systemd_unit_ownership.txt
for f in $(find /etc/systemd/system -maxdepth 2 -type f -name "*.service" 2>/dev/null); do
  owner=$(pacman -Qo "$f" 2>/dev/null || echo "NOT OWNED BY PACKAGE")
  echo "$f -> $owner" | tee -a evidence/i2/systemd_unit_ownership.txt
done

# List enabled units and their state
systemctl list-unit-files --state=enabled --no-pager | tee evidence/i2/systemd_enabled_units.txt

# List active timers (scheduled execution)
systemctl list-timers --all --no-pager | tee evidence/i2/systemd_active_timers.txt
```

Expected:
- Any `.service` or `.timer` file in `/etc/systemd/system/` that is NOT owned by a package (`NOT OWNED BY PACKAGE`) is a candidate for review.
- Note any unexpected enabled services.

### Step 3 — Phase B: Cron persistence locations

```bash
# System cron directories
echo "=== /etc/cron.d ===" | tee evidence/i2/cron_sweep.txt
ls -lah /etc/cron.d/ 2>/dev/null | tee -a evidence/i2/cron_sweep.txt
cat /etc/cron.d/* 2>/dev/null | tee -a evidence/i2/cron_sweep.txt

echo "=== /etc/crontab ===" | tee -a evidence/i2/cron_sweep.txt
cat /etc/crontab 2>/dev/null | tee -a evidence/i2/cron_sweep.txt

echo "=== /etc/cron.daily /etc/cron.weekly /etc/cron.hourly ===" | tee -a evidence/i2/cron_sweep.txt
ls -lah /etc/cron.daily /etc/cron.weekly /etc/cron.hourly 2>/dev/null | tee -a evidence/i2/cron_sweep.txt

# Per-user crontabs (/var/spool/cron/crontabs/ or /var/spool/cron/)
echo "=== User crontabs ===" | tee evidence/i2/cron_user_crontabs.txt
sudo ls -lah /var/spool/cron/ 2>/dev/null | tee -a evidence/i2/cron_user_crontabs.txt
for user in $(getent passwd | awk -F: '$7 ~ /bash|zsh|sh/ {print $1}'); do
  echo "--- Crontab for $user ---" | tee -a evidence/i2/cron_user_crontabs.txt
  sudo crontab -l -u "$user" 2>/dev/null | tee -a evidence/i2/cron_user_crontabs.txt
done

# at / batch queues
echo "=== at/batch queue ===" | tee evidence/i2/at_queue.txt
sudo atq 2>/dev/null | tee -a evidence/i2/at_queue.txt \
  || echo "at command not available or queue empty" | tee -a evidence/i2/at_queue.txt
```

Expected:
- On a clean Arch system, `/etc/cron.d/` and `/etc/crontab` may be empty or absent (Arch uses systemd timers by default).
- Any user crontab entry should be correlated with known administrative tasks.

### Step 4 — Phase C: Shell profile and startup file persistence

```bash
# Files that execute on shell login or startup
PROFILE_FILES=(
  /etc/profile
  /etc/bash.bashrc
  /etc/zsh/zshrc
  /etc/zsh/zprofile
)

echo "=== System-wide shell init files ===" | tee evidence/i2/shell_profiles_system.txt
for f in "${PROFILE_FILES[@]}"; do
  [ -f "$f" ] && echo "=== $f ===" | tee -a evidence/i2/shell_profiles_system.txt \
    && cat "$f" | tee -a evidence/i2/shell_profiles_system.txt
done

# /etc/profile.d/ scripts (sourced on login)
echo "=== /etc/profile.d/ ===" | tee -a evidence/i2/shell_profiles_system.txt
ls -lah /etc/profile.d/ | tee -a evidence/i2/shell_profiles_system.txt
cat /etc/profile.d/*.sh 2>/dev/null | tee -a evidence/i2/shell_profiles_system.txt

# Per-user startup files
echo "=== User shell init files ===" | tee evidence/i2/shell_profiles_user.txt
for user_home in /home/*/; do
  for init_file in .bashrc .bash_profile .bash_login .profile .zshrc .zprofile; do
    full_path="${user_home}${init_file}"
    if sudo test -f "$full_path"; then
      echo "=== $full_path ===" | tee -a evidence/i2/shell_profiles_user.txt
      sudo cat "$full_path" | tee -a evidence/i2/shell_profiles_user.txt
    fi
  done
done

# Root user profile
for init_file in .bashrc .bash_profile .profile .zshrc; do
  [ -f "/root/$init_file" ] && echo "=== /root/$init_file ===" \
    | tee -a evidence/i2/shell_profiles_user.txt \
    && sudo cat "/root/$init_file" | tee -a evidence/i2/shell_profiles_user.txt
done
```

Expected:
- Shell init files on a clean system contain only distribution-provided settings and admin customizations.
- Red flags: `curl | bash` pipelines, base64-encoded strings, reverse shell commands, unexpected environment variable modifications (especially `PATH` prepending).

### Step 5 — Phase D: SSH authorized_keys audit

```bash
# Find all authorized_keys files
sudo find /home /root /etc/ssh -name "authorized_keys" -type f 2>/dev/null \
  | tee evidence/i2/authorized_keys_paths.txt

# Dump contents of each authorized_keys file
echo "=== authorized_keys contents ===" | tee evidence/i2/authorized_keys_contents.txt
while IFS= read -r keyfile; do
  echo "--- $keyfile ---" | tee -a evidence/i2/authorized_keys_contents.txt
  sudo cat "$keyfile" | tee -a evidence/i2/authorized_keys_contents.txt
  echo "" | tee -a evidence/i2/authorized_keys_contents.txt
done < evidence/i2/authorized_keys_paths.txt

# Count and fingerprint all deployed keys
echo "=== Key fingerprints ===" | tee evidence/i2/authorized_keys_fingerprints.txt
while IFS= read -r keyfile; do
  echo "--- $keyfile ---" | tee -a evidence/i2/authorized_keys_fingerprints.txt
  while IFS= read -r key_line; do
    [ -z "$key_line" ] || [[ "$key_line" == \#* ]] && continue
    echo "$key_line" | ssh-keygen -lf - 2>/dev/null | tee -a evidence/i2/authorized_keys_fingerprints.txt
  done < <(sudo cat "$keyfile" 2>/dev/null)
done < evidence/i2/authorized_keys_paths.txt

echo "Total authorized_keys files found: $(wc -l < evidence/i2/authorized_keys_paths.txt)"
```

Expected:
- Every public key fingerprint should be traceable to a known administrator or account.
- Any `command=` prefix option on a key is noteworthy (restricts what the key can run — could be used by attacker for targeted persistence).

### Step 6 — Phase E: SUID/SGID binary audit

```bash
# Find all SUID binaries
sudo find / -xdev -perm -4000 -type f 2>/dev/null \
  | sort | tee evidence/i2/suid_binaries.txt
echo "SUID binary count: $(wc -l < evidence/i2/suid_binaries.txt)"

# Find all SGID binaries
sudo find / -xdev -perm -2000 -type f 2>/dev/null \
  | sort | tee evidence/i2/sgid_binaries.txt
echo "SGID binary count: $(wc -l < evidence/i2/sgid_binaries.txt)"

# Check package ownership of each SUID binary
echo "=== SUID ownership check ===" | tee evidence/i2/suid_ownership.txt
while IFS= read -r f; do
  owner=$(pacman -Qo "$f" 2>/dev/null || echo "NOT OWNED BY PACKAGE")
  echo "$f -> $owner" | tee -a evidence/i2/suid_ownership.txt
done < evidence/i2/suid_binaries.txt

# World-writable directories (drop zones)
sudo find / -xdev -type d -perm -0002 -not -name "tmp" 2>/dev/null \
  | sort | tee evidence/i2/world_writable_dirs.txt
echo "World-writable directories: $(wc -l < evidence/i2/world_writable_dirs.txt)"
```

Expected:
- All SUID binaries on a clean Arch system are owned by packages (`pacman -Qo` returns a result).
- Any SUID/SGID binary NOT owned by a package is a critical finding.
- World-writable directories beyond `/tmp`, `/var/tmp`, and expected share paths are potential attacker staging areas.

## 8) Validation

```bash
# Summary report: flag any anomalies for review
echo "=== PERSISTENCE SWEEP SUMMARY ===" | tee evidence/i2/sweep_summary.txt
echo "" | tee -a evidence/i2/sweep_summary.txt

echo "1. Unpackaged systemd units:" | tee -a evidence/i2/sweep_summary.txt
grep "NOT OWNED BY PACKAGE" evidence/i2/systemd_unit_ownership.txt \
  | tee -a evidence/i2/sweep_summary.txt || echo "   None found (CLEAN)" | tee -a evidence/i2/sweep_summary.txt

echo "" | tee -a evidence/i2/sweep_summary.txt
echo "2. User crontab entries:" | tee -a evidence/i2/sweep_summary.txt
grep -v "^#\|^$\|^---" evidence/i2/cron_user_crontabs.txt \
  | tee -a evidence/i2/sweep_summary.txt || echo "   None found (CLEAN)" | tee -a evidence/i2/sweep_summary.txt

echo "" | tee -a evidence/i2/sweep_summary.txt
echo "3. Unpackaged SUID binaries:" | tee -a evidence/i2/sweep_summary.txt
grep "NOT OWNED BY PACKAGE" evidence/i2/suid_ownership.txt \
  | tee -a evidence/i2/sweep_summary.txt || echo "   None found (CLEAN)" | tee -a evidence/i2/sweep_summary.txt

echo "" | tee -a evidence/i2/sweep_summary.txt
echo "4. authorized_keys file count:" | tee -a evidence/i2/sweep_summary.txt
wc -l < evidence/i2/authorized_keys_paths.txt | tee -a evidence/i2/sweep_summary.txt

cat evidence/i2/sweep_summary.txt
date -u | tee evidence/i2/sweep_end_time.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/i2/sweep_start_time.txt` — investigation timestamp
  - `evidence/i2/env_uname.txt` — host identity
  - `evidence/i2/shell_users.txt` — interactive shell users
  - `evidence/i2/snapshot_pre_num.txt` — Snapper snapshot number
  - `evidence/i2/systemd_units_system.txt` — system unit file inventory
  - `evidence/i2/systemd_units_user.txt` — user unit file inventory
  - `evidence/i2/systemd_unit_ownership.txt` — package ownership of units
  - `evidence/i2/systemd_enabled_units.txt` — enabled unit list
  - `evidence/i2/systemd_active_timers.txt` — active timer list
  - `evidence/i2/cron_sweep.txt` — system cron locations
  - `evidence/i2/cron_user_crontabs.txt` — per-user crontabs
  - `evidence/i2/at_queue.txt` — scheduled at jobs
  - `evidence/i2/shell_profiles_system.txt` — system shell init files
  - `evidence/i2/shell_profiles_user.txt` — user shell init files
  - `evidence/i2/authorized_keys_paths.txt` — authorized_keys locations
  - `evidence/i2/authorized_keys_contents.txt` — authorized_keys data
  - `evidence/i2/authorized_keys_fingerprints.txt` — SSH key fingerprints
  - `evidence/i2/suid_binaries.txt` — SUID binary list
  - `evidence/i2/sgid_binaries.txt` — SGID binary list
  - `evidence/i2/suid_ownership.txt` — SUID package ownership
  - `evidence/i2/world_writable_dirs.txt` — world-writable directories
  - `evidence/i2/sweep_summary.txt` — anomaly summary report
  - `evidence/i2/sweep_end_time.txt` — investigation end timestamp
- **Hash manifest:**

```bash
find evidence/i2 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/i2/SHA256SUMS.txt
cat evidence/i2/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `find / -perm -4000` hangs or runs very slowly.
  - **Cause:** The `-xdev` flag was omitted, causing `find` to cross into network or virtual filesystems.
  - **Fix:** Always include `-xdev` to restrict search to the same filesystem. Kill the hung find: `pkill find`.
  - **Rollback trigger:** N/A — read-only operation.

- **Symptom:** `pacman -Qo <file>` returns errors for many binaries.
  - **Cause:** Package database may be out of sync, or you are checking files outside the package manager's scope.
  - **Fix:** `sudo pacman -Fy && sudo pacman -F <file>` to search by filename. If file is in `/usr/local/` it will never be owned by pacman — that is expected.
  - **Rollback trigger:** N/A.

- **Symptom:** Shell profile analysis reveals suspicious base64 command.
  - **Cause:** Potential malware or leftover CTF artifact.
  - **Fix:** Decode and inspect before taking action: `echo "<base64_string>" | base64 -d`. Take a Snapper snapshot before removing anything. Document all findings.
  - **Rollback trigger:** Snapper rollback if a legitimate script was accidentally removed.

## 11) Sources

- [MITRE ATT&CK — TA0003 Persistence](https://attack.mitre.org/tactics/TA0003/)
- [MITRE ATT&CK — T1543.002 Systemd Service](https://attack.mitre.org/techniques/T1543/002/)
- [MITRE ATT&CK — T1053.003 Cron](https://attack.mitre.org/techniques/T1053/003/)
- [MITRE ATT&CK — T1098.004 SSH Authorized Keys](https://attack.mitre.org/techniques/T1098/004/)
- [man 5 proc — /proc filesystem](https://man7.org/linux/man-pages/man5/proc.5.html)
- [Arch Wiki — Security — Restricting root](https://wiki.archlinux.org/title/Security)
- [NIST SP 800-61r2 — Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)

## 12) Stretch Goals

- Automate the sweep into a single script (`i2-persistence-sweep.sh`) that outputs a timestamped evidence directory and a markdown anomaly summary report.
- Extend to cover LD_PRELOAD persistence: `sudo grep -r LD_PRELOAD /etc/profile.d/ /etc/ld.so.conf.d/ /home /root` and `/etc/ld.so.preload`.
- Add `/etc/rc.local` inspection and check for SysVinit scripts: `ls /etc/init.d/ 2>/dev/null`.
- Cross-reference findings against a known-good baseline generated immediately after a fresh install. Use `diff` to identify what changed since the baseline capture.
