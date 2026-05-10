---
aliases:
  - FHS
  - Filesystem Hierarchy Standard
  - Forensic Artifact Hunting
  - Linux Evidence Locations
tags:
  - linux
  - fhs
  - forensics
  - net412
  - net179
  - phase1
date: 2026-05-10
---

# 03 — FHS & Forensics Artifact Hunting

> [!info] Why This Matters
> The Linux Filesystem Hierarchy Standard (FHS) is the map. Every attacker leaves traces in predictable locations. Every misconfiguration lives in a documented directory. If you know the FHS cold, you can triage any system in minutes.

## The FHS at a Glance

```
/
├── bin/        → Essential user binaries (symlinked to /usr/bin/ on modern systems)
├── boot/       → Bootloader, kernel, initramfs
├── dev/        → Device files (disks, ttys, null, random)
├── etc/        → System-wide configuration files
├── home/       → User home directories (/home/username)
├── lib/        → Shared libraries (symlinked to /usr/lib/ on Arch)
├── media/      → Removable media mount points
├── mnt/        → Temporary manual mount points
├── opt/        → Third-party/self-contained software
├── proc/       → Virtual FS: live kernel and process data
├── root/       → root user's home directory
├── run/        → Runtime data (PIDs, sockets) — cleared on boot
├── srv/        → Service data (web roots, FTP data)
├── sys/        → Virtual FS: kernel hardware interface
├── tmp/        → Temporary files — cleared on boot (usually)
├── usr/        → User-land programs, libraries, documentation
│   ├── bin/    → User binaries
│   ├── lib/    → Libraries
│   ├── local/  → Locally compiled/installed software
│   └── share/  → Architecture-independent data
└── var/        → Variable data: logs, databases, mail, spools
    ├── log/    → System and application logs
    ├── spool/  → Print/mail queues
    └── tmp/    → Persistent temp (not cleared on boot)
```

> [!tip] Arch Linux Note
> On Arch, `/bin`, `/lib`, `/lib64`, and `/sbin` are symlinks to their `/usr/` equivalents. This is the "UsrMerge" change. Don't be confused when `ls -la /bin` shows `lrwxrwxrwx ... /bin -> usr/bin`.

---

## Critical Directories for Forensic Triage

### 🔴 /etc — Configuration Ground Truth

```bash
# Most-examined files in a forensic investigation:
/etc/passwd             # User accounts (no passwords since ~1990)
/etc/shadow             # Hashed passwords (root-readable only)
/etc/group              # Group memberships
/etc/sudoers            # Who can run what as root
/etc/sudoers.d/         # Modular sudo rules (check for rogue entries)
/etc/hosts              # Local DNS overrides (attackers poison this)
/etc/crontab            # System-wide cron jobs
/etc/cron.d/            # Cron job fragments
/etc/cron.{hourly,daily,weekly,monthly}/  # Time-based script drops
/etc/profile            # Login shell environment
/etc/profile.d/         # Shell profile fragments (persistence vector)
/etc/bash.bashrc        # Global bash config
/etc/ld.so.preload      # Library preload — CRITICAL rootkit indicator
/etc/ssh/sshd_config    # SSH daemon configuration
/etc/systemd/system/    # Custom systemd units (persistence)
/etc/modules-load.d/    # Kernel modules loaded at boot
```

> [!danger] /etc/ld.so.preload — Rootkit Red Flag
> If this file exists and contains anything, investigate immediately. Attackers use LD_PRELOAD to hook libc functions and hide processes, files, and network connections. On a clean Arch system, this file should not exist.
> ```bash
> ls -la /etc/ld.so.preload 2>/dev/null && cat /etc/ld.so.preload || echo "CLEAN: File does not exist"
> ```

---

### 🔴 /var/log — The Paper Trail

```bash
# System and security logs (systemd-based systems use journald)
/var/log/journal/       # Binary systemd journal (query with journalctl)
/var/log/auth.log       # SSH, sudo, PAM authentication (Debian/Ubuntu)
/var/log/secure         # Same as auth.log on RHEL/Arch
/var/log/syslog         # General system log (Debian)
/var/log/messages       # General system log (RHEL)
/var/log/kern.log       # Kernel messages
/var/log/dmesg          # Boot-time hardware detection
/var/log/pacman.log     # Arch: package install/remove/upgrade history
/var/log/Xorg.0.log     # X display server log
/var/log/nginx/         # Web server access and error logs
/var/log/apache2/       # Apache access and error logs
/var/log/wtmp           # Binary: successful logins (read with 'last')
/var/log/btmp           # Binary: failed logins (read with 'lastb')
/var/log/lastlog        # Binary: last login per user (read with 'lastlog')
/var/log/faillog        # Failed login attempts
```

```bash
# Reading binary log files
last -F -20             # last 20 successful logins with timestamps
lastb -20               # last 20 failed logins (requires root)
lastlog | grep -v "Never"   # users who have logged in recently
who                     # currently logged in users
w                       # who + what they're doing
```

> [!tip] Log Rotation Awareness
> Logs rotate. `/var/log/auth.log` → `/var/log/auth.log.1` → `.gz`. During an investigation, check rotated archives:
> ```bash
> zcat /var/log/auth.log.2.gz | grep "Failed password"
> ```

---

### 🔴 /proc — Live System Intelligence (NET 377)

`/proc` is a virtual filesystem populated entirely by the kernel at runtime. Nothing here persists after reboot.

```bash
/proc/1/                # PID 1 = systemd/init
/proc/<PID>/cmdline     # Full command line of that process
/proc/<PID>/exe         # Symlink to the actual executable
/proc/<PID>/fd/         # Open file descriptors
/proc/<PID>/net/        # Network connections for that process namespace
/proc/<PID>/maps        # Memory maps (loaded libraries)
/proc/<PID>/environ     # Environment variables
/proc/version           # Kernel version
/proc/cpuinfo           # CPU details
/proc/meminfo           # Memory statistics
/proc/mounts            # Currently mounted filesystems
/proc/net/tcp           # TCP connections (hex encoded)
/proc/net/tcp6          # IPv6 TCP connections
/proc/sys/              # Runtime kernel parameters (sysctl interface)
```

```bash
# Find the executable for a suspicious process
ls -la /proc/$(pgrep suspicious_proc)/exe

# What files does a process have open?
ls -la /proc/$(pgrep sshd)/fd/

# Read a process's environment (is it running with secrets in env vars?)
strings /proc/$(pgrep suspicious)/environ

# Check all TCP connections with owning process (raw)
cat /proc/net/tcp | awk '{print $2, $3, $4}' | head -20
```

---

### 🔴 /home & /root — User Activity Artifacts

```bash
# Shell history files — first stop in any investigation
~/.bash_history         # Bash command history
~/.zsh_history          # Zsh history
~/.fish/fish_history    # Fish shell history
/root/.bash_history     # Root's history

# SSH artifacts
~/.ssh/authorized_keys  # Public keys allowed to log in (backdoor vector)
~/.ssh/known_hosts      # Systems this user has connected to
~/.ssh/id_*             # Private keys (gold for lateral movement)
~/.ssh/config           # SSH client configuration

# Application artifacts
~/.gnupg/               # GPG keys and keyring
~/.local/share/         # XDG data (browser data, application state)
~/.config/              # Application configuration files
~/.bashrc / ~/.bash_profile   # Persistence via aliases, functions, PATH modification

# Your project artifacts (Apex Predator PMO)
~/apex-predator/        # Rust engine source — check for hardcoded secrets
~/.config/apex/         # Configuration files — check permissions
```

> [!warning] .ssh/authorized_keys Backdoor
> Attackers who compromise a system often append their public key to `/root/.ssh/authorized_keys`. This survives password resets and persists across reboots.
> ```bash
> # Check for unauthorized authorized_keys entries
> cat /root/.ssh/authorized_keys 2>/dev/null
> for home in /home/*/; do
>   echo "=== $home ==="
>   cat "${home}.ssh/authorized_keys" 2>/dev/null || echo "None"
> done
> ```

---

### 🔴 /tmp & /var/tmp — Attacker Staging Areas

```bash
# Attackers love /tmp: world-writable, often no-exec but tricks exist
find /tmp /var/tmp -type f 2>/dev/null | xargs ls -la 2>/dev/null
find /tmp /var/tmp -name "*.sh" -o -name "*.py" -o -name "*.pl" 2>/dev/null
find /tmp /var/tmp -perm /111 -type f 2>/dev/null    # executable files in /tmp
```

> [!danger] SUID/SGID in /tmp
> ```bash
> find /tmp /var/tmp -perm -4000 -o -perm -2000 2>/dev/null
> ```
> If this returns anything, your system is compromised. No legitimate software installs SUID binaries in /tmp.

---

### 🔴 Persistence Mechanisms by Location

| Persistence Vector | Location | Detection Command |
|---|---|---|
| Cron job | `/etc/cron*`, `/var/spool/cron/` | `cat /etc/crontab; ls /etc/cron.d/` |
| Systemd unit | `/etc/systemd/system/`, `~/.config/systemd/user/` | `systemctl list-units --state=enabled` |
| Shell profile | `~/.bashrc`, `~/.bash_profile`, `/etc/profile.d/` | `grep -r "curl\|wget\|nc\|bash" /etc/profile.d/` |
| SUID binary | Anywhere | `find / -perm -4000 2>/dev/null` |
| SSH key | `~/.ssh/authorized_keys` | `cat /root/.ssh/authorized_keys` |
| LD_PRELOAD | `/etc/ld.so.preload` | `cat /etc/ld.so.preload` |
| Kernel module | `/lib/modules/`, `lsmod` | `lsmod | grep -v "$(modinfo $mod | grep name)"` |
| PAM module | `/etc/pam.d/` | `diff /etc/pam.d/sshd /etc/pam.d/sshd.bak` |

---

## NET 179: Sterile Evidence Acquisition at the FHS Level

When you're doing forensics on a live system (or a mounted image), acquisition must be sterile — don't write to the evidence source.

```bash
# Hash a file BEFORE touching it
sha256sum /var/log/auth.log > /tmp/auth.log.sha256

# Acquire a file to evidence directory (never work on originals)
cp -p /var/log/auth.log /mnt/evidence/case01/auth.log
sha256sum /mnt/evidence/case01/auth.log >> /tmp/auth.log.sha256

# Verify integrity
sha256sum -c /tmp/auth.log.sha256

# Acquire all shell histories (with path preservation)
find /home /root -name ".*_history" 2>/dev/null \
  | while read f; do
      dest="/mnt/evidence/case01${f}"
      mkdir -p "$(dirname "$dest")"
      cp -p "$f" "$dest"
      sha256sum "$f" >> /tmp/evidence_hashes.txt
    done
```

> [!tip] Btrfs Snapshot as Evidence Anchor
> On your Arch system with Snapper, a snapshot is a read-only, point-in-time copy that can serve as forensic baseline:
> ```bash
> snapper create --description "forensic-baseline-$(date +%Y%m%d_%H%M%S)" --cleanup-algorithm=number
> ```
> See [[../Phase-5-Architect/02-Btrfs-Snapshots|Phase 5 — Btrfs Snapshots]] for full Snapper workflow.

---

## Practical Lab: Arch Live System Triage

```bash
# === COMPLETE SYSTEM TRIAGE SCRIPT ===

echo "=== 1. Rootkit Indicators ==="
ls -la /etc/ld.so.preload 2>/dev/null || echo "CLEAN: No ld.so.preload"
find / -perm -4000 -type f 2>/dev/null | tee /tmp/suid_list.txt
echo "SUID binary count: $(wc -l < /tmp/suid_list.txt)"

echo ""
echo "=== 2. Authentication Audit ==="
awk -F: '$3 == 0 {print "UID-0 ACCOUNT:", $1}' /etc/passwd
awk -F: '$7 !~ /nologin|false/ {print "SHELL ACCOUNT:", $1, $7}' /etc/passwd

echo ""
echo "=== 3. Persistence Check ==="
ls /etc/cron.d/ 2>/dev/null
ls /etc/systemd/system/*.service 2>/dev/null | head -20
for home in /home/* /root; do
  echo "--- $home ---"
  cat "${home}/.ssh/authorized_keys" 2>/dev/null || echo "  No authorized_keys"
done

echo ""
echo "=== 4. Recent /etc Modifications (7 days) ==="
find /etc -mtime -7 -type f 2>/dev/null | grep -v ".pyc" | head -20

echo ""
echo "=== 5. Suspicious /tmp Content ==="
find /tmp /var/tmp -type f -perm /111 2>/dev/null | xargs ls -la 2>/dev/null
```

---

## Troubleshooting Scenario: "Where Is the Log for X?"

**Symptom:** You need to find why a service failed but don't know where it logs.

**Approach:**
```bash
# Step 1: Check if it's a systemd service (most things are)
systemctl status myservice.service
# The "Loaded" and "Active" lines tell you where the unit file is

# Step 2: journalctl for systemd-managed services
journalctl -u myservice.service -n 50 --no-pager

# Step 3: Check if it has a custom log location in its config
grep -i "log\|Log" /etc/myservice/config.conf 2>/dev/null

# Step 4: Check for syslog-style logs
ls /var/log/ | grep -i myservice

# Step 5: strace the process to see what files it opens
strace -e trace=openat -p $(pgrep myservice) 2>&1 | grep "log\|\.conf"

# Step 6: lsof for open files
lsof -p $(pgrep myservice) | grep -i log
```

---

## 🏁 Proof of Work — Phase 1.3 Mini-CTF

> [!example] Challenge: FHS Artifact Hunt
> You are performing triage on your own Arch system. Generate a "System State Report" proving you can locate all key forensic artifacts.
>
> ```bash
> # Run this complete triage and save output
> {
>   echo "=== FORENSIC TRIAGE REPORT ==="
>   echo "Date: $(date -u)"
>   echo "Hostname: $(hostname)"
>   echo "Kernel: $(uname -r)"
>   echo ""
>   echo "--- UID 0 Accounts ---"
>   awk -F: '$3 == 0 {print $1}' /etc/passwd
>   echo ""
>   echo "--- Accounts with Interactive Shells ---"
>   awk -F: '$3 >= 1000 && $7 !~ /nologin|false/ {print $1, $7}' /etc/passwd
>   echo ""
>   echo "--- ld.so.preload Status ---"
>   ls -la /etc/ld.so.preload 2>/dev/null || echo "CLEAN: Not present"
>   echo ""
>   echo "--- SUID Binary Count ---"
>   find / -perm -4000 -type f 2>/dev/null | wc -l
>   echo ""
>   echo "--- Last 5 Logins ---"
>   last -F -5
>   echo ""
>   echo "--- Systemd Enabled Services ---"
>   systemctl list-units --type=service --state=running --no-pager | grep -c "running"
>   echo "running services"
> } | tee /tmp/phase1_triage.txt
>
> # Hash and store as your proof
> sha256sum /tmp/phase1_triage.txt
> ```
>
> **Submit:** The SHA-256 hash of `/tmp/phase1_triage.txt`. The content of this file is your NET 179 deliverable.

---

← [[02-Text-Manipulation]] | [[../Phase-2-Blueprint/00-Phase2-Overview|Next: Phase 2 →]]
