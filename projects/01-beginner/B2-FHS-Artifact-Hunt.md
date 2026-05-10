---
title: B2 - FHS Artifact Hunt
aliases:
  - b2-fhs-artifact-hunt
  - filesystem-triage-beginner
tags:
  - beginner
  - fhs
  - forensics
  - triage
  - evidence
  - net179
  - net412
date: 2026-05-10
---

# B2 — FHS Artifact Hunt

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host. All commands are read-only except the evidence directory creation. No files are modified or deleted.
> **Risk level:** Low — purely forensic/observational. No services are touched.
> **Threat model:** An analyst who does not know the Linux Filesystem Hierarchy Standard (FHS) cannot reliably locate artifacts during an investigation or triage session. This project builds systematic artifact-location muscle memory.
> **Out of scope:** Live-memory acquisition, kernel module inspection, raw disk imaging. Those require advanced forensics tooling not covered here.
> **Data sensitivity:** Some paths visited (e.g., `/etc/shadow`, `/root/`) require root. Never export shadow hashes or private key material to the evidence bundle — capture only metadata and redacted excerpts where required.

## 1) Mission

- **Problem statement:** During incident response and forensic investigations, knowing exactly which filesystem paths to check — and in what order — is the difference between finding an attacker's artifact in minutes versus hours. The FHS defines where binaries, configs, logs, runtime data, and user homes live. Mastering these paths is prerequisite knowledge for every subsequent project.
- **Why it matters:** NET 179 (Digital Forensics) exam scenarios and real investigations both test whether you can systematically locate persistence mechanisms, dropped files, modified configs, and log evidence across the standard Linux directory tree.

## 2) Difficulty

- Beginner (estimated 3–5 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal. Read-only investigation of the local filesystem. Evidence output directory is created under `./evidence/b2/`.

## 4) Prerequisites

- **Skills:** Basic terminal navigation, understanding of absolute vs relative paths, ability to read `man` pages.
- **Tools:** `find`, `ls`, `stat`, `file`, `cat`, `grep`, `awk`, `sha256sum`, `journalctl`, `tee`
- **Dependencies:** `sudo` access for protected paths; `procfs` mounted at `/proc` (standard on all Linux systems).

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-b2-fhs-hunt"` — record snapshot number.
- **Host Snapper post:** Not required — no changes are made to the host. Create a post snapshot as good practice: `sudo snapper -c root create --description "post-b2-fhs-hunt"`
- **VM snapshot:** N/A
- **Container rebuild command:** N/A
- **Rollback trigger:** If any command accidentally modifies a file (e.g., accidental redirect with `>`), use `sudo snapper -c root undochange <PRE>..<POST>` to restore.

## 6) Project Plan

- **Phase A — FHS Map:** Document the purpose and key artifact types for each major FHS directory. Record directory tree and ownership.
- **Phase B — Artifact Locations Sweep:** Systematically visit each major FHS location and collect metadata about interesting files.
- **Phase C — Log and Runtime Paths:** Inspect `/var/log/`, `/run/`, and `/proc/` for volatile artifacts and running-process data.

## 7) Walkthrough

### Step 1 — Setup evidence directory and environment check

```bash
mkdir -p evidence/b2

# Record hostname, kernel, and date for evidence context
uname -a | tee evidence/b2/env_uname.txt
hostname | tee evidence/b2/env_hostname.txt
date -u | tee evidence/b2/env_datetime.txt
id | tee evidence/b2/env_whoami.txt

# Record mount table — know what filesystems are active
findmnt --output TARGET,SOURCE,FSTYPE,OPTIONS | tee evidence/b2/env_mounts.txt
```

Expected:
- `env_uname.txt` shows Linux kernel version and architecture.
- `env_mounts.txt` shows at least `/` (Btrfs), `/proc`, `/sys`, `/dev`, `/run`, and any additional mounts.

### Step 2 — Document the FHS top-level layout

```bash
# Capture directory listing with types
ls -lah --color=never / | tee evidence/b2/fhs_root_listing.txt

# Show size of each top-level directory (non-recursive, quick)
du -shx --exclude=/proc --exclude=/sys /* 2>/dev/null | sort -h | tee evidence/b2/fhs_root_sizes.txt

# Show ownership and permissions of critical directories
stat /bin /sbin /usr /etc /var /home /root /tmp /opt /srv /run | tee evidence/b2/fhs_critical_stat.txt
```

Expected:
- Root listing shows standard FHS directories: `bin`, `boot`, `dev`, `etc`, `home`, `lib`, `opt`, `proc`, `root`, `run`, `srv`, `sys`, `tmp`, `usr`, `var`.
- Symlinks (e.g., `/bin -> usr/bin` on modern Arch) are visible in `ls -lah` output.

### Step 3 — Investigate /etc (configuration artifacts)

```bash
# List files modified in the last 7 days inside /etc
find /etc -maxdepth 2 -newer /etc/os-release -type f 2>/dev/null \
  | tee evidence/b2/etc_recently_modified.txt

# Record OS release info
cat /etc/os-release | tee evidence/b2/etc_os_release.txt

# Record active shell configuration files
ls -lah /etc/profile /etc/profile.d/ /etc/bash.bashrc 2>/dev/null | tee evidence/b2/etc_shell_configs.txt

# List /etc/cron.d and crontab directories for scheduled tasks
ls -lah /etc/cron.d /etc/cron.daily /etc/cron.weekly /etc/cron.hourly 2>/dev/null \
  | tee evidence/b2/etc_cron_dirs.txt

# Record sudoers configuration (redacted — do NOT capture sensitive values)
sudo ls -lah /etc/sudoers /etc/sudoers.d/ 2>/dev/null | tee evidence/b2/etc_sudoers_listing.txt

# Record SSH server config (sensitive values will be visible — keep evidence bundle local)
sudo grep -Ev "^#|^$" /etc/ssh/sshd_config 2>/dev/null | tee evidence/b2/etc_sshd_config_active.txt
```

Expected:
- `etc_recently_modified.txt` may show recently changed config files — any unexpected entries are worth noting.
- `etc_cron_dirs.txt` reveals any scheduled tasks in standard cron directories.

### Step 4 — Investigate /var (variable data and logs)

```bash
# List /var/log contents with sizes
ls -lahR --color=never /var/log/ 2>/dev/null | tee evidence/b2/var_log_listing.txt

# Capture last 50 lines of key system logs
sudo journalctl -b --no-pager -n 50 | tee evidence/b2/var_journal_recent.txt

# Capture auth log entries (if present as flat file)
sudo cat /var/log/auth.log 2>/dev/null | tail -n 50 | tee evidence/b2/var_auth_log_tail.txt || echo "auth.log not present (using journald)" | tee evidence/b2/var_auth_log_tail.txt

# Check /var/spool/cron for user crontabs
sudo ls -lahR /var/spool/cron/ 2>/dev/null | tee evidence/b2/var_spool_cron.txt

# Record /var/tmp for persistent temp files (survives reboots unlike /tmp)
find /var/tmp -maxdepth 2 -type f 2>/dev/null | tee evidence/b2/var_tmp_files.txt
```

Expected:
- `var_log_listing.txt` shows journal files plus any flat log files present.
- `var_tmp_files.txt` may reveal temp files left by processes or installers.

### Step 5 — Investigate /proc (live process and kernel data)

```bash
# List running process IDs
ls /proc | grep -E '^[0-9]+$' | tee evidence/b2/proc_pids.txt | wc -l

# For each running process, capture its executable path (non-root processes only)
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  exe=$(readlink /proc/$pid/exe 2>/dev/null)
  [ -n "$exe" ] && echo "$pid $exe"
done | tee evidence/b2/proc_exe_paths.txt

# Capture kernel parameters
cat /proc/version | tee evidence/b2/proc_kernel_version.txt
cat /proc/cmdline | tee evidence/b2/proc_cmdline.txt
cat /proc/meminfo | tee evidence/b2/proc_meminfo.txt

# Show loaded kernel modules
lsmod | tee evidence/b2/proc_lsmod.txt
```

Expected:
- `proc_exe_paths.txt` shows PID-to-executable mappings for all accessible processes.
- `proc_lsmod.txt` lists all loaded kernel modules — note any unusual or unexpected entries.

### Step 6 — Investigate /tmp and /run (volatile runtime paths)

```bash
# List /tmp contents (world-writable — common attacker drop zone)
find /tmp -maxdepth 2 2>/dev/null | tee evidence/b2/tmp_listing.txt
ls -lahR --color=never /tmp 2>/dev/null | tee evidence/b2/tmp_detailed.txt

# List /run contents (runtime state — cleared on reboot)
ls -lah /run/ | tee evidence/b2/run_listing.txt
find /run -maxdepth 2 -type s 2>/dev/null | tee evidence/b2/run_sockets.txt
```

Expected:
- `/tmp` may contain socket files, lock files, or session data from running applications.
- `/run/sockets.txt` lists UNIX domain sockets currently in use.

### Step 7 — Investigate /home and user paths

```bash
# List all home directories
ls -lah /home/ | tee evidence/b2/home_listing.txt

# For each home directory, list top-level contents (as root)
for user_home in /home/*/; do
  echo "=== $user_home ===" | tee -a evidence/b2/home_contents.txt
  sudo ls -lah "$user_home" 2>/dev/null | tee -a evidence/b2/home_contents.txt
done

# Check for SSH authorized_keys in all home directories and root
sudo find /home /root -name "authorized_keys" -type f 2>/dev/null | tee evidence/b2/ssh_authorized_keys_paths.txt

# List .bash_history paths (do NOT cat — may contain sensitive commands)
sudo find /home /root -name ".bash_history" -type f 2>/dev/null \
  | tee evidence/b2/bash_history_paths.txt
```

Expected:
- `ssh_authorized_keys_paths.txt` lists all `authorized_keys` files — verify that no unexpected public keys have been added.
- `bash_history_paths.txt` documents which users have shell history files.

### Step 8 — Investigate /usr/local (admin-installed artifacts)

```bash
# List custom binaries and scripts installed outside package manager
find /usr/local/bin /usr/local/sbin /usr/local/lib /usr/local/share \
  -maxdepth 2 -type f 2>/dev/null | tee evidence/b2/usr_local_files.txt

# Check file types for anything unexpected
find /usr/local/bin /usr/local/sbin -maxdepth 1 -type f 2>/dev/null \
  -exec file {} \; | tee evidence/b2/usr_local_bin_filetypes.txt
```

Expected:
- On a fresh Arch install, `/usr/local/bin` and `/usr/local/sbin` should be empty or contain only admin-placed tools.
- Any ELF binaries present that were not deliberately installed are a potential indicator of compromise.

## 8) Validation

```bash
# Confirm all expected evidence files exist
expected_files=(
  "env_uname.txt" "env_mounts.txt" "fhs_root_listing.txt"
  "etc_os_release.txt" "etc_recently_modified.txt"
  "var_log_listing.txt" "proc_exe_paths.txt" "tmp_listing.txt"
  "home_listing.txt" "ssh_authorized_keys_paths.txt"
  "usr_local_files.txt"
)

for f in "${expected_files[@]}"; do
  if [ -f "evidence/b2/$f" ]; then
    echo "PRESENT: evidence/b2/$f"
  else
    echo "MISSING: evidence/b2/$f"
  fi
done

# Count total files captured
find evidence/b2 -type f | wc -l
```

## 9) Evidence

- **Output files:**
  - `evidence/b2/env_*.txt` — host environment context
  - `evidence/b2/fhs_root_listing.txt` — root directory layout
  - `evidence/b2/fhs_root_sizes.txt` — directory size inventory
  - `evidence/b2/fhs_critical_stat.txt` — key directory metadata
  - `evidence/b2/etc_*.txt` — /etc artifact captures
  - `evidence/b2/var_*.txt` — /var artifact captures
  - `evidence/b2/proc_*.txt` — /proc kernel and process data
  - `evidence/b2/tmp_*.txt` — /tmp volatile data
  - `evidence/b2/run_*.txt` — /run socket and runtime state
  - `evidence/b2/home_*.txt` — user home directory survey
  - `evidence/b2/ssh_authorized_keys_paths.txt` — SSH key locations
  - `evidence/b2/usr_local_*.txt` — admin-installed binary inventory
- **Hash manifest:**

```bash
find evidence/b2 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/b2/SHA256SUMS.txt
cat evidence/b2/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `Permission denied` on protected paths like `/root/` or `/etc/shadow`.
  - **Cause:** Running commands as a non-root user without `sudo`.
  - **Fix:** Prefix commands with `sudo`. For bulk operations, run a dedicated sub-shell: `sudo bash -c 'find /root -type f'`
  - **Rollback trigger:** N/A — read-only operation.

- **Symptom:** `/proc/<PID>/exe` shows `(deleted)` for some processes.
  - **Cause:** The process was started from a binary that has since been replaced or deleted (common after package upgrades without restart).
  - **Fix:** Note the process in evidence. Use `ls -la /proc/<PID>/exe` and `cat /proc/<PID>/cmdline` to get more context. This is an expected condition — not necessarily malicious.
  - **Rollback trigger:** N/A.

- **Symptom:** `find /var/tmp` returns an unexpected number of files.
  - **Cause:** Application wrote temp data that was not cleaned up.
  - **Fix:** Identify which package owns the files using `pacman -Qo <path>`. Do not delete without understanding the purpose.
  - **Rollback trigger:** N/A.

## 11) Sources

- [Linux Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html)
- [man 7 hier — Linux filesystem hierarchy description](https://man7.org/linux/man-pages/man7/hier.7.html)
- [man 5 proc — /proc filesystem documentation](https://man7.org/linux/man-pages/man5/proc.5.html)
- [Arch Wiki — Security](https://wiki.archlinux.org/title/Security)
- [SANS Digital Forensics — Linux IR cheat sheet](https://www.sans.org/posters/linux-shell-survival-guide/)
- [NIST SP 800-86 — Integration of Forensic Techniques into Incident Response](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-86.pdf)

## 12) Stretch Goals

- Write a single Bash script (`b2-fhs-sweep.sh`) that runs all investigation steps non-interactively and outputs to a timestamped evidence directory.
- Use `auditd` or `inotifywait` to monitor `/etc/` for write events during a 60-second observation window and capture the output.
- Extend the sweep to a Proxmox VM guest: SSH in, run the same sweep, and compare the FHS layout of a Debian/Ubuntu VM vs Arch bare metal.
- Cross-reference findings with the [MITRE ATT&CK Linux persistence techniques](https://attack.mitre.org/tactics/TA0003/) and annotate which FHS locations are relevant to each technique.
