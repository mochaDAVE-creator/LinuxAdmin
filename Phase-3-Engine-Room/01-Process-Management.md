---
aliases:
  - Process Management
  - ps top htop
  - kill signals
  - /proc filesystem
tags:
  - linux
  - processes
  - net412
  - net377
  - phase3
date: 2026-05-10
---

# 01 — Process Management

> [!info] Why This Matters
> Every running program is a process. Privilege escalation often targets vulnerable processes. Malware hides inside legitimate process trees. Understanding the process model lets you see what's actually executing — and what shouldn't be.

## The Process Model

```
PID 1 (systemd)
 └── PID 500 (sshd)
      └── PID 1200 (sshd: dave@pts/0)    ← your session
           └── PID 1201 (bash)
                └── PID 1350 (htop)      ← you started this
```

Every process has:
- **PID** — Process ID (unique)
- **PPID** — Parent Process ID (who spawned it)
- **UID/GID** — the identity it runs under
- **stdin/stdout/stderr** — I/O file descriptors
- **An open file descriptor table** — everything it can access

---

## ps — Process Status

```bash
# All processes, full format
ps aux
# a = all users, u = user-friendly format, x = include processes without terminal

# Column reference:
# USER   PID  %CPU  %MEM  VSZ   RSS  TTY  STAT  START  TIME  COMMAND
# dave   1350   2.0   1.1  ...   ...  pts/0  S+  02:00  0:01  htop

# Tree view (shows parent-child relationships)
ps auxf
ps -ejH             # hierarchy format

# Find a specific process
ps aux | grep sshd
ps -C sshd          # by name

# Show specific columns
ps -eo pid,ppid,user,stat,cmd --sort=-%cpu | head -20

# Long listing with all info
ps -elf
```

### STAT Column Codes

| Code | Meaning |
|---|---|
| `R` | Running or runnable |
| `S` | Sleeping (interruptible) |
| `D` | Uninterruptible sleep (disk I/O) |
| `Z` | Zombie — finished but parent hasn't collected it |
| `T` | Stopped (Ctrl+Z or SIGSTOP) |
| `<` | High priority (negative nice) |
| `N` | Low priority (positive nice) |
| `s` | Session leader |
| `+` | In foreground process group |
| `l` | Multi-threaded |

> [!warning] Zombie Processes
> A zombie (Z) means the parent process is failing to call `wait()`. It doesn't consume CPU but does consume a PID slot. If you have hundreds of zombies, the parent process is buggy. Kill the parent to clean them up.

---

## pgrep & pkill — Name-Based Process Control

```bash
# Find PID(s) by name
pgrep sshd
pgrep -a sshd        # show full command line
pgrep -u dave        # processes owned by dave
pgrep -P 1           # children of PID 1

# Kill by name (sends SIGTERM by default)
pkill firefox
pkill -9 frozen_app  # force kill
pkill -u dave        # kill all of dave's processes (careful!)
pkill -P 1200 -SIGTERM   # kill children of PID 1200
```

---

## Signals — The Process Communication Protocol

```bash
# List all signals
kill -l

# Common signals
kill -SIGTERM <pid>    # 15: graceful shutdown (default)
kill -SIGKILL <pid>    # 9:  force kill (cannot be caught or ignored)
kill -SIGHUP  <pid>    # 1:  hangup — most daemons reload config on HUP
kill -SIGSTOP <pid>    # 19: pause process (cannot be caught)
kill -SIGCONT <pid>    # 18: resume paused process
kill -SIGINT  <pid>    # 2:  interrupt (Ctrl+C equivalent)
kill -SIGQUIT <pid>    # 3:  quit with core dump (Ctrl+\)

# Send to process group
kill -SIGTERM -<PGID>  # negative PID targets the whole process group

# Kill all processes with the same name
killall firefox
killall -9 unresponsive_daemon
```

> [!tip] SIGTERM Before SIGKILL
> Always try SIGTERM first. It lets the process clean up (close DB connections, flush buffers, remove temp files). SIGKILL is instant but leaves mess. Pattern:
> ```bash
> kill -SIGTERM <pid>
> sleep 5
> kill -0 <pid> 2>/dev/null && kill -SIGKILL <pid>  # kill -0 checks if process exists
> ```

---

## top & htop — Real-Time Process Monitoring

```bash
# top — classic, always available
top
# Interactive commands:
# P = sort by CPU
# M = sort by Memory
# k = kill a process (enter PID)
# u = filter by user
# q = quit
# 1 = show all CPU cores

# top output example:
# PID  USER  PR  NI  VIRT  RES  SHR  S  %CPU  %MEM  TIME+  COMMAND
# 1350 dave  20   0  ...   ...  ...  S   2.0   1.1  0:01  htop

# htop — interactive, color-coded (install: pacman -S htop)
htop
# F5 = tree view
# F6 = sort by column
# F9 = send signal
# F4 = filter by name

# Non-interactive CPU snapshot (for scripts)
ps -eo pid,comm,%cpu --sort=-%cpu | head -11
```

---

## nice & renice — Process Priority

Priority range: **-20** (highest priority) to **+19** (lowest).
Only root can set negative nice values.

```bash
# Start process with low priority (leave resources for your CTF tools)
nice -n 10 ./heavy_computation

# Start with high priority (give your Rust engine priority)
sudo nice -n -10 ./apex-predator-engine

# Change priority of a running process
sudo renice -n -5 -p 1350     # change PID 1350 to priority -5
sudo renice -n 15 -u dave      # lower all of dave's processes to 15
```

---

## /proc — Process Intelligence (Forensics & Offensive)

Every running process has a directory in `/proc/<PID>/`.

```bash
# What is this process actually executing?
ls -la /proc/$(pgrep sshd)/exe
# → lrwxrwxrwx ... /proc/1200/exe -> /usr/bin/sshd

# What files does it have open? (look for unexpected connections)
ls -la /proc/$(pgrep sshd)/fd/
# → fd/3 -> socket:[12345] (network socket)
# → fd/4 -> /var/log/auth.log

# Full command line (null-byte delimited)
cat /proc/$(pgrep sshd)/cmdline | tr '\0' ' '

# Environment variables
strings /proc/$(pgrep suspicious)/environ
# Look for: AWS keys, passwords, API tokens in env

# Memory maps — what libraries is it using?
cat /proc/$(pgrep sshd)/maps | awk '{print $6}' | sort | uniq | grep -v "^$"

# Network connections for this process's namespace
cat /proc/$(pgrep sshd)/net/tcp   # hex encoded

# Working directory
ls -la /proc/$(pgrep sshd)/cwd
```

> [!tip] NET 377 — Identifying Process Injection
> Legitimate processes don't usually have anonymous memory-mapped regions with executable permissions. Look for:
> ```bash
> cat /proc/<PID>/maps | grep "rwxp" | grep -v ".so\|heap\|stack\|vdso"
> ```
> Anonymous `rwxp` regions (no file path) are a shellcode injection indicator.

---

## lsof — List Open Files

```bash
# All open files on system
lsof | head -50

# Files opened by a specific process
lsof -p $(pgrep sshd)

# Who has a specific file open?
lsof /var/log/auth.log

# All network connections
lsof -i
lsof -i TCP                 # TCP only
lsof -i :22                 # processes using port 22
lsof -i @192.168.1.50       # connections to specific IP

# Find deleted-but-still-open files (forensics: recover from running process)
lsof | grep "deleted"
# If a log file was deleted but a process still has it open, you can recover:
# cat /proc/<PID>/fd/<FD_NUM> > /tmp/recovered.log
```

---

## Practical Lab: Process Investigation

```bash
# 1. Show your full process tree
ps auxf | head -40

# 2. Find top 5 CPU consumers right now
ps -eo pid,comm,%cpu --sort=-%cpu | head -6

# 3. Find top 5 memory consumers
ps -eo pid,comm,%mem,rss --sort=-rss | head -6 | awk '{printf "%-8s %-20s %s MB\n", $1, $2, $4/1024}'

# 4. Inspect the init process
cat /proc/1/cmdline | tr '\0' ' '
ls /proc/1/fd/ | wc -l     # how many files does systemd have open?

# 5. Find processes with network connections
lsof -i -n -P | grep ESTABLISHED

# 6. Check for suspicious executable paths
ps aux | awk '{print $11}' | sort -u | grep -v "^\[" | head -30

# 7. Find zombie processes
ps aux | awk '$8 == "Z" {print "ZOMBIE:", $2, $11}'
```

---

## Troubleshooting Scenario: Process Won't Die

**Symptom:** `kill -SIGKILL <pid>` doesn't kill the process.

**Diagnosis:**
```bash
# Check process state
ps aux | grep <pid>
# If state is 'D' (uninterruptible sleep), it's blocked in kernel code

# D-state processes cannot be killed — they're waiting for I/O
# Common causes: NFS hang, dying disk, stuck kernel driver

# Check what system call it's blocked in
cat /proc/<PID>/wchan    # which kernel function it's waiting in
strace -p <PID>          # may time out if stuck in kernel

# Check dmesg for disk/filesystem errors
dmesg | tail -20 | grep -iE "error|ata|io|timeout"
```

**Fix:**
```bash
# For NFS: unmount the hanging NFS mount
sudo umount -f -l /mnt/nfs_share

# For disk I/O: wait for it to resolve, or reboot (last resort)
# For Btrfs scrub in progress:
sudo btrfs scrub status /

# If it's truly stuck and causing system issues, reboot cleanly
sudo systemctl reboot
```

---

## 🏁 Proof of Work — Phase 3.1 Mini-CTF

> [!example] Challenge: Hunt the Phantom Process
>
> ```bash
> # Start a background "phantom" process to find
> sleep 99999 &
> PHANTOM_PID=$!
> echo "A process with PID $PHANTOM_PID is running."
> echo "Find it using ONLY ps, pgrep, and /proc (no using the $PHANTOM_PID variable)"
>
> # Your investigation:
> # 1. Find all sleep processes
> pgrep -a sleep
>
> # 2. Verify via /proc
> cat /proc/$(pgrep -n sleep)/cmdline | tr '\0' ' '
>
> # 3. Who spawned it? (PPID)
> ps -o ppid= -p $(pgrep -n sleep)
>
> # 4. Terminate it gracefully
> pkill -SIGTERM sleep
>
> # 5. Verify it's gone
> pgrep sleep || echo "PASS: Process terminated"
>
> # Proof
> echo "Investigation complete: $(date -u)" | tee /tmp/phase3_1_pow.txt
> sha256sum /tmp/phase3_1_pow.txt
> ```

---

← [[00-Phase3-Overview|Phase 3 Overview]] | Next: [[02-Systemd-Deep-Dive]] →
