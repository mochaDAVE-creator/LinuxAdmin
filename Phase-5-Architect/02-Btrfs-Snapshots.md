---
aliases:
  - Btrfs
  - Snapper
  - Snapshots
  - Btrfs Rollback
  - Subvolumes
tags:
  - linux
  - btrfs
  - snapper
  - arch
  - net412
  - phase5
date: 2026-05-10
---

# 02 — Btrfs & Snapper Rollbacks

> [!info] Why This Matters
> Your Arch system lives on Btrfs with Snapper. A bad update, a misconfigured service, or an accidental `rm -rf` in the wrong directory can be reversed in under 60 seconds — if you set this up correctly. This module teaches you how to take snapshots, when to take them, and how to roll back from GRUB without booting the broken system.

## Btrfs Concepts

### Subvolumes

A Btrfs subvolume is an independent filesystem tree within a Btrfs pool. Subvolumes can be snapshotted instantly (copy-on-write — actual data is shared until modified).

```bash
# List all subvolumes on the Btrfs filesystem
sudo btrfs subvolume list /

# Typical Arch Btrfs layout:
# ID 256  top level 5  path @          → mounted as /
# ID 257  top level 5  path @home      → mounted as /home
# ID 258  top level 5  path @log       → mounted as /var/log
# ID 259  top level 5  path @pkg       → mounted as /var/cache/pacman/pkg
# ID 260  top level 5  path @snapshots → Snapper snapshot storage

# Check current Btrfs filesystem info
sudo btrfs filesystem show /
sudo btrfs filesystem usage /
df -h /    # disk usage (Btrfs reports differently than actual usage)
```

### Copy-on-Write (CoW)

The key Btrfs feature: when you write to a file, Btrfs writes the new data to a new location and updates the pointer — it never overwrites existing data. This makes snapshots instant (just copy the pointer, not the data).

```bash
# Create a manual snapshot
sudo btrfs subvolume snapshot / /mnt/btrfs-root/@snapshots/manual-2026-05-10

# Create a read-only snapshot (recommended for backups)
sudo btrfs subvolume snapshot -r / /mnt/btrfs-root/@snapshots/ro-2026-05-10

# Delete a subvolume/snapshot
sudo btrfs subvolume delete /mnt/btrfs-root/@snapshots/old-snapshot
```

---

## Snapper — Automated Snapshot Management

Snapper automates snapshot creation, retention, and comparison. It's the layer on top of raw Btrfs commands.

### Installation and Setup

```bash
# Install
sudo pacman -S snapper snap-pac grub-btrfs

# Create a Snapper config for root (/)
# Snapper requires the subvolume to be writable
sudo snapper -c root create-config /

# List configs
sudo snapper list-configs

# The config file is at:
cat /etc/snapper/configs/root
```

### Snapper Configuration

```bash
sudo nvim /etc/snapper/configs/root
```

```bash
# Key settings to tune:
TIMELINE_CREATE="yes"           # create hourly timeline snapshots
TIMELINE_CLEANUP="yes"          # auto-cleanup old snapshots

# Retention policy (how many to keep)
TIMELINE_LIMIT_HOURLY="5"       # keep 5 hourly snapshots
TIMELINE_LIMIT_DAILY="7"        # keep 7 daily snapshots
TIMELINE_LIMIT_WEEKLY="4"       # keep 4 weekly snapshots
TIMELINE_LIMIT_MONTHLY="6"      # keep 6 monthly snapshots
TIMELINE_LIMIT_YEARLY="1"       # keep 1 yearly snapshot

# Space limits
SPACE_LIMIT="0.5"              # don't exceed 50% of filesystem for snapshots
FREE_LIMIT="0.2"               # ensure 20% free space always
```

```bash
# Enable automatic timeline snapshots
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer

# snap-pac: automatically takes snapshots before/after pacman operations
# It's already configured if you installed snap-pac — no setup needed
```

---

## Working with Snapshots

### Creating Snapshots

```bash
# Create a single snapshot (with description)
sudo snapper -c root create --description "Before major config change"
sudo snapper -c root create --description "Before pacman -Syu"

# Create a pre/post pair (for tracking changes)
sudo snapper -c root create --type pre --print-number
# → 42
sudo pacman -Syu   # or whatever you're doing
sudo snapper -c root create --type post --pre-number 42 --description "Post pacman upgrade"
```

### Listing Snapshots

```bash
# List all snapshots
sudo snapper -c root list

# Output:
#  #   | Type   | Pre # | Date                        | User | Cleanup  | Description
# ------+--------+-------+-----------------------------+------+----------+------------------
#    0  | single |       |                             | root |          | current
#    1  | single |       | 2026-05-09 22:00:00 +0000   | root | timeline |
#   42  | pre    |       | 2026-05-10 01:00:00 +0000   | root | number   | Before upgrade
#   43  | post   |    42 | 2026-05-10 01:05:00 +0000   | root | number   | Post pacman upgrade
```

### Comparing Snapshots (Change Detection)

```bash
# Show what files changed between two snapshots
sudo snapper -c root diff 42 43

# Detailed diff (content changes)
sudo snapper -c root diff --diff-cmd="diff -u" 42 43 /etc/ssh/sshd_config

# Status (C=changed, + added, - deleted, T=type changed)
sudo snapper -c root status 42..43

# Forensic: what changed in /etc in the last 24 hours?
PRE=$(sudo snapper -c root list | awk '/yesterday/{print $1}' | head -1)
CURRENT=0
sudo snapper -c root status ${PRE}..${CURRENT} | grep "/etc/"
```

---

## Rolling Back with Snapper + grub-btrfs

### Setup grub-btrfs (Boot into Any Snapshot from GRUB)

```bash
# Install grub-btrfs (already done above)
sudo pacman -S grub-btrfs

# Configure grub-btrfs to automatically update GRUB menu
sudo systemctl enable --now grub-btrfsd

# Or update GRUB manually after taking snapshots
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Verify snapshots appear in GRUB
grep -i "btrfs\|snapshot" /boot/grub/grub.cfg | head -10
```

### Method 1: Rollback via GRUB (No Boot Required — GUI-Accessible)

1. Reboot your system
2. In GRUB menu, select "Arch Linux snapshots"
3. Select the snapshot number you want to boot (read-only boot — test it first)
4. If it works, make it permanent (see Method 2)

### Method 2: Permanent Rollback (from Working System)

```bash
# Identify the snapshot number to roll back to
sudo snapper -c root list

# Option A: snapper rollback (Arch method)
sudo snapper -c root rollback 42    # rolls back to snapshot #42
# This creates a new writable snapshot from #42 and marks it as default subvolume
# Requires reboot to take effect
sudo reboot

# Option B: Manual rollback (full control)
# Mount the Btrfs top-level subvolume
sudo mount /dev/sda2 /mnt/btrfs-root -o subvol=/

# View available snapshots
ls /mnt/btrfs-root/.snapshots/

# Backup current system (paranoia is good)
sudo btrfs subvolume snapshot /mnt/btrfs-root/@ /mnt/btrfs-root/@.broken

# Replace current root with the snapshot
sudo btrfs subvolume delete /mnt/btrfs-root/@
sudo btrfs subvolume snapshot /mnt/btrfs-root/.snapshots/42/snapshot /mnt/btrfs-root/@

# Reboot
sudo umount /mnt/btrfs-root
sudo reboot
```

> [!danger] Rollback is Irreversible Without Another Snapshot
> Before rolling back, take a snapshot of the CURRENT (broken) state. You may need it to extract specific files later.
> ```bash
> sudo snapper -c root create --description "broken-state-before-rollback-$(date +%Y%m%d)"
> ```

---

## Btrfs Maintenance and Health

```bash
# Scrub — verify all data against checksums (detect silent corruption)
sudo btrfs scrub start /
sudo btrfs scrub status /

# Schedule regular scrubs (monthly is typical)
sudo systemctl enable --now btrfs-scrub@-.timer     # for /
sudo systemctl enable --now btrfs-scrub@home.timer  # for /home

# Balance — redistribute data across drives (for multi-device pools)
sudo btrfs balance start -dusage=85 /
sudo btrfs balance status /

# Check filesystem health
sudo btrfs check --readonly /dev/sda2   # DO NOT run without --readonly on mounted FS

# Find snapshot disk usage
sudo btrfs subvolume show /
sudo btrfs qgroup show /   # if quotas enabled

# Enable space quotas (required for accurate snapshot disk usage)
sudo btrfs quota enable /
sudo btrfs qgroup show /
```

---

## Using Snapshots as Forensic Baselines (NET 179)

```bash
# Take a "clean state" snapshot as forensic baseline
sudo snapper -c root create --description "forensic-baseline-clean-$(date +%Y%m%d)"
BASELINE=$(sudo snapper -c root list | grep "forensic-baseline" | awk '{print $1}' | tail -1)
echo "Baseline snapshot: $BASELINE"

# After an incident, compare current state to baseline
sudo snapper -c root status ${BASELINE}..0 | tee /tmp/incident_changes.txt

# Focus on high-value directories
sudo snapper -c root status ${BASELINE}..0 | grep -E "^(c|\.\.)(.*\/etc\/|.*\/root\/|.*\/home\/|.*\/tmp\/)"

# Extract a file from a snapshot without rolling back
SNAPSHOT_PATH="/.snapshots/${BASELINE}/snapshot"
sudo ls "${SNAPSHOT_PATH}/etc/ssh/sshd_config"   # verify it exists
sudo diff "${SNAPSHOT_PATH}/etc/ssh/sshd_config" /etc/ssh/sshd_config
```

---

## Practical Lab: Snapshot Workflow

```bash
# 1. Verify Snapper is configured and running
sudo snapper list-configs
sudo systemctl status snapper-timeline.timer
sudo systemctl status snapper-cleanup.timer

# 2. Take a manual snapshot before this lab
sudo snapper -c root create --description "phase5-lab-before-$(date +%Y%m%d_%H%M%S)"

# 3. Make a deliberate change
echo "# Lab test comment - $(date)" | sudo tee -a /etc/hosts

# 4. Take post-change snapshot
sudo snapper -c root create --description "phase5-lab-after-$(date +%Y%m%d_%H%M%S)"

# 5. Show the diff
SNAPSHOTS=$(sudo snapper -c root list | tail -2 | awk '{print $1}')
PRE=$(echo "$SNAPSHOTS" | head -1)
POST=$(echo "$SNAPSHOTS" | tail -1)
echo "Comparing snapshots $PRE and $POST:"
sudo snapper -c root diff $PRE $POST

# 6. Roll back the change (file-level, no reboot)
sudo snapper -c root undochange $PRE..$POST /etc/hosts

# 7. Verify rollback
grep "Lab test comment" /etc/hosts && echo "FAIL: Still present" || echo "PASS: Rolled back"

# 8. List all snapshots
sudo snapper -c root list
```

---

## Troubleshooting Scenario: Pacman Upgrade Broke the System

**Symptom:** After `pacman -Syu`, a critical service won't start.

**Recovery Without Reboot:**
```bash
# Check if snap-pac captured pre/post snapshots
sudo snapper -c root list | tail -5
# Look for "pre" and "post" types with description matching the upgrade

# Identify what changed
sudo snapper -c root status PRE..POST | grep -E "^c.*\/(etc|usr\/lib)"

# Roll back only the affected config files (no full rollback needed)
PRE=<pre_snapshot_number>
POST=<post_snapshot_number>
sudo snapper -c root undochange $PRE..$POST /etc/nginx/nginx.conf

# Restart the service
sudo systemctl restart nginx
```

**Recovery via GRUB (if boot is broken):**
1. Reboot → GRUB → "Arch Linux snapshots" → select pre-upgrade snapshot
2. Boot into it (read-only) — verify system works
3. Log in and run: `sudo snapper -c root rollback <snapshot_number>`
4. Reboot into the rolled-back system

---

## 🏁 Proof of Work — Phase 5.2 Mini-CTF

> [!example] Challenge: Snapshot-Based Change Detection
>
> ```bash
> # Create a baseline snapshot
> sudo snapper -c root create --description "ctf-baseline" --print-number
> BASE_SNAP=$(sudo snapper -c root list | grep "ctf-baseline" | awk '{print $1}')
>
> # Simulate "attacker activity"
> echo "evil_binary ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/99-evil
> sudo useradd -m -s /bin/bash evil_user 2>/dev/null
>
> # Create post-incident snapshot
> sudo snapper -c root create --description "ctf-post-incident" --print-number
> POST_SNAP=$(sudo snapper -c root list | grep "ctf-post-incident" | awk '{print $1}')
>
> # Detect the changes (your forensic analysis)
> echo "=== CHANGES DETECTED ==="
> sudo snapper -c root status ${BASE_SNAP}..${POST_SNAP} | tee /tmp/phase5_2_changes.txt
>
> # Identify specific high-value changes
> echo ""
> echo "=== SUDOERS CHANGES ==="
> sudo snapper -c root status ${BASE_SNAP}..${POST_SNAP} | grep "sudoers"
>
> echo ""
> echo "=== NEW USER ACCOUNTS ==="
> sudo snapper -c root status ${BASE_SNAP}..${POST_SNAP} | grep "passwd\|shadow"
>
> # Roll back BOTH changes
> sudo snapper -c root undochange ${BASE_SNAP}..${POST_SNAP} /etc/sudoers.d/99-evil /etc/passwd /etc/shadow /etc/group
> sudo rm -f /etc/sudoers.d/99-evil
> sudo userdel -r evil_user 2>/dev/null
>
> # Verify clean
> grep "evil" /etc/sudoers.d/99-evil 2>/dev/null && echo "FAIL" || echo "PASS: Sudoers clean"
> grep "evil_user" /etc/passwd && echo "FAIL" || echo "PASS: User removed"
>
> sha256sum /tmp/phase5_2_changes.txt
> ```
>
> **Submit:** The hash of the changes file. The key findings: what two persistence mechanisms were installed, and how were they detected?

---

← [[01-Pacman-Database]] | Next: [[03-Container-Isolation]] →
