---
aliases:
  - CLI Survival
  - Terminal Basics
  - Linux Navigation
tags:
  - linux
  - cli
  - net412
  - net377
  - phase1
date: 2026-05-10
---

# 01 — CLI Survival

> [!info] Why This Matters
> Every offensive tool, forensic technique, and administrative task in this vault runs through the terminal. If you're slow here, you're slow everywhere. Build the muscle memory now.

## Core Navigation

### The Mental Model

The Linux filesystem is a single tree rooted at `/`. There are no drive letters — only mount points. Everything (disks, devices, sockets, processes) is a file.

```bash
# Where am I?
pwd

# What's here?
ls -la          # long format, show hidden files
ls -lah         # add human-readable sizes
ls -laht        # sort by modification time (newest first) — forensics gold

# Move around
cd /var/log     # absolute path
cd ..           # up one level
cd -            # back to previous directory
cd ~            # home directory
```

### Efficient Navigation

```bash
# Jump deep fast
cd /etc/systemd/system

# Autocomplete (hit Tab)
cd /etc/sys<Tab>

# History — your command memory
history
!!              # repeat last command
!ssh            # repeat last command starting with 'ssh'
Ctrl+R          # reverse search through history
```

> [!tip] Forensics Angle
> On a compromised system, `~/.bash_history` and `/root/.bash_history` are the first artifacts to grab. The attacker's commands are in there (unless they ran `unset HISTFILE` or `history -c`).

---

## File Operations

```bash
# Create
touch file.txt                  # create empty file / update timestamp
mkdir -p /tmp/evidence/case01   # create nested directories

# Copy and Move
cp -av source dest              # archive mode (preserve timestamps), verbose
cp -r dir/ dest/                # recursive directory copy
mv oldname newname              # move or rename

# Delete
rm file.txt
rm -rf /tmp/evidence/case01     # recursive force — no recovery from here
```

> [!danger] rm -rf Has No Undo
> On Btrfs you can roll back with [[../Phase-5-Architect/02-Btrfs-Snapshots|Snapper]], but only if you took a snapshot first. Default assumption: deletion is permanent.

### Linking

```bash
# Hard link — same inode, survives original deletion
ln /etc/passwd /tmp/passwd-backup

# Symbolic link — pointer to path
ln -s /opt/apex-predator/engine /usr/local/bin/apex

# Identify links
ls -lai         # inode numbers reveal hard links
readlink -f /usr/local/bin/apex
```

---

## I/O Redirection & Pipes

The most powerful concept in Linux. Master this and you can build any data pipeline.

```
stdin  (0) — keyboard input
stdout (1) — normal output
stderr (2) — error messages
```

```bash
# Redirect stdout to file (overwrite)
nmap -sV 192.168.1.0/24 > scan_results.txt

# Redirect stdout to file (append)
nmap -sV 192.168.1.0/24 >> scan_results.txt

# Redirect stderr
./exploit 2> errors.txt

# Redirect both stdout and stderr
./exploit > output.txt 2>&1
./exploit &> output.txt         # bash shorthand

# Discard output entirely
./noisy_tool > /dev/null 2>&1

# Pipe — stdout of left becomes stdin of right
cat /var/log/auth.log | grep "Failed password" | tail -20

# tee — write to file AND keep it in the pipe
tcpdump -i eth0 -w - | tee capture.pcap | strings | grep "password"
```

> [!tip] Pipe Chains for Forensics
> Build analysis pipelines:
> ```bash
> journalctl -b | grep -i "sudo\|su\|root" | awk '{print $1,$2,$3,$5}' | sort | uniq -c | sort -rn
> ```
> This extracts privilege escalation activity from the current boot log.

---

## Essential File Viewing

```bash
# View entire file
cat /etc/passwd
less /var/log/syslog     # paginated, search with /pattern

# View extremes
head -20 /var/log/auth.log      # first 20 lines
tail -50 /var/log/auth.log      # last 50 lines
tail -f /var/log/auth.log       # follow in real time (live monitoring)
tail -F /var/log/auth.log       # follow, even if file is rotated

# Word/line/byte counts
wc -l /etc/passwd               # how many users?
wc -c capture.pcap              # file size in bytes
```

---

## Finding Things

```bash
# Find files by name
find / -name "*.conf" 2>/dev/null
find /home -name ".bash_history" 2>/dev/null

# Find by permission (forensics: SUID binaries)
find / -perm -4000 -type f 2>/dev/null     # SUID files
find / -perm -2000 -type f 2>/dev/null     # SGID files
find / -perm -o+w -type f 2>/dev/null      # world-writable files

# Find by modification time
find /etc -mtime -1 2>/dev/null            # modified in last 24 hours
find /tmp -newer /tmp/reference_file       # newer than reference

# Find by owner
find / -user root -writable 2>/dev/null    # root-owned but writable

# Locate (uses database — may be stale)
updatedb
locate passwd
```

> [!warning] find on / is Noisy
> Redirect stderr to `/dev/null` or you'll drown in "Permission denied" messages. In a forensic context, those permission errors are actually interesting — note what you can't access.

---

## Compression & Archiving

```bash
# tar — Tape Archive (the standard)
tar -czf backup.tar.gz /path/to/dir        # compress with gzip
tar -cjf backup.tar.bz2 /path/to/dir      # compress with bzip2
tar -xzf backup.tar.gz -C /tmp/restore/   # extract to directory
tar -tzf backup.tar.gz                    # list contents without extracting

# zip/unzip
zip -r archive.zip /path/to/dir
unzip archive.zip -d /tmp/extracted/

# Hash before and after to verify integrity (NET 179)
sha256sum backup.tar.gz > backup.tar.gz.sha256
sha256sum -c backup.tar.gz.sha256
```

> [!tip] NET 179 — Evidence Packaging
> When packaging forensic evidence, always hash before and after. The SHA-256 of your evidence container is your chain of custody anchor.

---

## Practical Lab: Arch Live System

Run this sequence on your Arch system to validate CLI fluency:

```bash
# 1. Map your system's critical directories
ls -la / | awk '{print $NF}' | sort

# 2. Find all SUID binaries (privilege escalation surface)
find / -perm -4000 -type f 2>/dev/null | tee /tmp/suid_audit.txt
wc -l /tmp/suid_audit.txt

# 3. Check recent file modifications in /etc
find /etc -mtime -7 -type f 2>/dev/null | head -20

# 4. Build a pipeline: top 10 processes by CPU via text manipulation
ps aux --sort=-%cpu | head -11 | awk '{printf "%-10s %-6s %s\n", $1, $3, $11}'

# 5. Check your bash history integrity
wc -l ~/.bash_history
head -5 ~/.bash_history
stat ~/.bash_history            # check modification time
```

---

## Troubleshooting Scenario: "Command Not Found" on PATH Issues

**Symptom:** You install a tool (e.g., `paru`) and get `-bash: paru: command not found` immediately after.

**Diagnosis:**
```bash
# Check what your PATH actually contains
echo $PATH

# Find where paru was installed
which paru 2>/dev/null || echo "not in PATH"
find /usr /opt /home -name "paru" 2>/dev/null

# Check if it's in a non-standard location
ls /usr/local/bin/
ls ~/.local/bin/
```

**Fix:**
```bash
# Add ~/.local/bin to PATH permanently
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Or for fish shell
fish_add_path ~/.local/bin
```

**Log Evidence:**
```bash
# Check pacman install log to see where files landed
grep "paru" /var/log/pacman.log | tail -5
```

---

## 🏁 Proof of Work — Phase 1.1 Mini-CTF

> [!example] Challenge: Forensic Triage on Your Own System
> A "suspicious script" was dropped somewhere on your system in the last 48 hours. Find it using only CLI tools.
>
> **Objective:** Identify all `.sh` files modified in the last 48 hours outside of standard package paths.
>
> ```bash
> # Your command here — build the pipeline
> find / -name "*.sh" -mtime -2 -not -path "/usr/share/*" -not -path "/usr/lib/*" 2>/dev/null
> ```
>
> **Validation:** Run the command. If you find zero results, that's a clean bill of health — document it:
> ```bash
> find / -name "*.sh" -mtime -2 -not -path "/usr/share/*" -not -path "/usr/lib/*" 2>/dev/null | tee /tmp/phase1_pow.txt
> echo "Files found: $(wc -l < /tmp/phase1_pow.txt)"
> sha256sum /tmp/phase1_pow.txt
> ```
> Submit the SHA-256 of your output file as proof. A hash of an empty output is still valid proof of a clean check.

---

← [[00-Phase1-Overview|Phase 1 Overview]] | Next: [[02-Text-Manipulation]] →
