---
title: B1 - Snapshot & Rollback Drill
aliases:
  - b1-snapshot-rollback
  - btrfs-snapper-drill
tags:
  - beginner
  - btrfs
  - snapper
  - rollback
  - evidence
  - net412
date: 2026-05-10
---

# B1 — Snapshot & Rollback Drill

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host with Btrfs filesystem and Snapper ≥ 0.10 installed and configured on the `root` config.
> **Risk level:** Low — all changes are made to `/etc/motd` (non-critical file). No services are restarted. No network changes occur.
> **Threat model:** Operator error during system configuration. Goal is to prove the rollback loop works *before* it is needed under pressure.
> **Out of scope:** Kernel updates, bootloader changes, subvolume restructuring. Those require advanced rollback procedures not covered here.
> **Prerequisite check:** Run `sudo snapper -c root list` — if it errors, configure Snapper first per the [Arch Wiki Snapper article](https://wiki.archlinux.org/title/Snapper) before proceeding.

## 1) Mission

- **Problem statement:** Operators who have never tested their rollback path discover it is broken exactly when they need it most — after a bad upgrade or misconfiguration. This project forces a dry run of the complete snapshot → change → validate → rollback loop using a harmless target file.
- **Why it matters:** Btrfs + Snapper is the core safety net for all other projects in this runbook. If you cannot execute a reliable rollback here, every subsequent high-risk project is dangerous. This drill also teaches the evidence-capture habit that all other projects depend on.

## 2) Difficulty

- Beginner (estimated 2–3 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal only. Do not attempt on a VM guest running Btrfs unless the VM itself has Snapper configured.

## 4) Prerequisites

- **Skills:** Basic terminal navigation (`cd`, `ls`, `cat`, `grep`), understanding of what a filesystem snapshot is conceptually.
- **Tools:** `snapper`, `btrfs-progs`, `sudo` access, `sha256sum`, `tee`
- **Dependencies:**
  - Btrfs filesystem mounted at `/` (verify with `findmnt -t btrfs /`)
  - Snapper `root` config created (`sudo snapper -c root list` succeeds)
  - `snapper` package installed (`pacman -Q snapper`)

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-b1-drill"` — capture snapshot number from output.
- **Host Snapper post:** `sudo snapper -c root create --description "post-b1-drill"`
- **VM snapshot:** N/A (this project runs on host, not in a VM)
- **Container rebuild command:** N/A
- **Manual rollback trigger:** If any step produces unexpected output, run:
  ```bash
  sudo snapper -c root undochange <PRE_NUM>..<POST_NUM>
  ```
  Replace `<PRE_NUM>` and `<POST_NUM>` with the snapshot numbers recorded during execution.

## 6) Project Plan

- **Phase A — Environment Verification:** Confirm Btrfs mount layout, Snapper config, and disk space headroom before touching anything.
- **Phase B — Snapshot + Controlled Change:** Create pre-snapshot, make reversible edit to `/etc/motd`, capture state.
- **Phase C — Rollback + Validation:** Use Snapper to undo the change, confirm file is restored byte-for-byte, hash the evidence bundle.

## 7) Walkthrough

### Step 1 — Verify Btrfs and Snapper environment

```bash
# Confirm Btrfs is the root filesystem
findmnt -t btrfs / | tee evidence/b1/env_btrfs_mount.txt

# Show subvolume layout
sudo btrfs subvolume list / | tee evidence/b1/env_subvol_list.txt

# Show existing Snapper snapshots (baseline)
sudo snapper -c root list | tee evidence/b1/env_snapshots_baseline.txt

# Show current disk usage (ensure >10 % free for snapshot COW overhead)
df -h / | tee evidence/b1/env_disk_usage.txt

# Record Snapper config details
sudo snapper -c root get-config | tee evidence/b1/env_snapper_config.txt
```

Expected:
- `findmnt` output shows `TYPE=btrfs` and `SOURCE` pointing to your disk device.
- `btrfs subvolume list` shows at least a `@` or `@root` subvolume and a `.snapshots` subvolume.
- `snapper list` shows existing snapshots (or empty if this is a fresh config — that is fine).
- Disk usage shows filesystem is not near capacity.

### Step 2 — Capture pre-change baseline of target file

```bash
mkdir -p evidence/b1

# Record the exact current state of /etc/motd
cat /etc/motd | tee evidence/b1/motd_before.txt

# Record file metadata (permissions, ownership, size, modification time)
stat /etc/motd | tee evidence/b1/motd_stat_before.txt

# Hash the file before any change
sha256sum /etc/motd | tee evidence/b1/motd_hash_before.txt
```

Expected:
- `motd_before.txt` contains the current contents of `/etc/motd` (may be empty — that is valid).
- `stat` output shows `Uid`, `Gid`, and `Access` mode — record these for comparison post-rollback.

### Step 3 — Create pre-change Snapper snapshot

```bash
# Create the pre-change snapshot and capture its number
sudo snapper -c root create --description "pre-b1-drill" --print-number | tee evidence/b1/snapshot_pre_num.txt

PRE_NUM=$(cat evidence/b1/snapshot_pre_num.txt)
echo "Pre-change snapshot number: $PRE_NUM"

# Confirm snapshot was created
sudo snapper -c root list | tee evidence/b1/snapshots_after_pre.txt

# Show snapshot metadata
sudo snapper -c root info "$PRE_NUM" | tee evidence/b1/snapshot_pre_info.txt
```

Expected:
- `snapshot_pre_num.txt` contains a single integer (e.g., `5`).
- `snapper list` now shows the new snapshot with description `pre-b1-drill`.

### Step 4 — Make controlled, reversible change

```bash
# Append a clearly marked test line to /etc/motd
echo "# B1-DRILL-TEST: $(date -u +%Y-%m-%dT%H:%M:%SZ)" | sudo tee -a /etc/motd

# Capture modified state
cat /etc/motd | tee evidence/b1/motd_after_change.txt
sha256sum /etc/motd | tee evidence/b1/motd_hash_after_change.txt

# Confirm the line is present
grep "B1-DRILL-TEST" /etc/motd | tee evidence/b1/motd_grep_change.txt
```

Expected:
- `/etc/motd` now ends with the `B1-DRILL-TEST` marker line.
- `sha256sum` output is different from `motd_hash_before.txt`.
- `grep` output shows exactly one matching line.

### Step 5 — Create post-change snapshot

```bash
sudo snapper -c root create --description "post-b1-drill" --print-number | tee evidence/b1/snapshot_post_num.txt

POST_NUM=$(cat evidence/b1/snapshot_post_num.txt)
echo "Post-change snapshot number: $POST_NUM"

sudo snapper -c root list | tee evidence/b1/snapshots_after_post.txt
```

Expected:
- `snapshot_post_num.txt` contains an integer greater than `PRE_NUM`.

### Step 6 — Execute rollback via snapper undochange

```bash
PRE_NUM=$(cat evidence/b1/snapshot_pre_num.txt)
POST_NUM=$(cat evidence/b1/snapshot_post_num.txt)

# Show what will be changed before committing
sudo snapper -c root undochange --dry-run "$PRE_NUM".."$POST_NUM" | tee evidence/b1/rollback_dry_run.txt

# Execute the actual rollback
sudo snapper -c root undochange "$PRE_NUM".."$POST_NUM" | tee evidence/b1/rollback_output.txt
```

Expected:
- Dry run shows `/etc/motd` as the file that will be modified.
- Actual rollback completes without errors.

### Step 7 — Validate restoration

```bash
# Check /etc/motd is restored
cat /etc/motd | tee evidence/b1/motd_after_rollback.txt
sha256sum /etc/motd | tee evidence/b1/motd_hash_after_rollback.txt

# Confirm B1 test line is gone
grep "B1-DRILL-TEST" /etc/motd && echo "FAIL: line still present" || echo "PASS: line removed"

# Compare hashes: before == after rollback
echo "Hash before change:"
cat evidence/b1/motd_hash_before.txt
echo "Hash after rollback:"
cat evidence/b1/motd_hash_after_rollback.txt
```

Expected:
- `grep` returns exit code 1 (no match) — you should see `PASS: line removed`.
- Both hash files show identical SHA-256 values, proving byte-for-byte restoration.

## 8) Validation

```bash
# Objective proof: pre-change and post-rollback hashes must match
PRE_HASH=$(awk '{print $1}' evidence/b1/motd_hash_before.txt)
POST_HASH=$(awk '{print $1}' evidence/b1/motd_hash_after_rollback.txt)

if [ "$PRE_HASH" = "$POST_HASH" ]; then
  echo "VALIDATION PASSED: rollback restored /etc/motd byte-for-byte"
else
  echo "VALIDATION FAILED: hashes differ — investigate evidence/b1/"
fi

# Confirm Snapper still lists both snapshots
sudo snapper -c root list | grep -E "pre-b1-drill|post-b1-drill"
```

## 9) Evidence

- **Output files:**
  - `evidence/b1/env_btrfs_mount.txt` — filesystem type confirmation
  - `evidence/b1/env_subvol_list.txt` — Btrfs subvolume layout
  - `evidence/b1/env_snapshots_baseline.txt` — pre-drill snapshot inventory
  - `evidence/b1/env_disk_usage.txt` — disk space check
  - `evidence/b1/env_snapper_config.txt` — Snapper config dump
  - `evidence/b1/motd_before.txt` — file contents before change
  - `evidence/b1/motd_stat_before.txt` — file metadata before change
  - `evidence/b1/motd_hash_before.txt` — SHA-256 before change
  - `evidence/b1/snapshot_pre_num.txt` — pre-change snapshot number
  - `evidence/b1/snapshot_pre_info.txt` — pre snapshot metadata
  - `evidence/b1/snapshots_after_pre.txt` — snapshot list after pre
  - `evidence/b1/motd_after_change.txt` — file after modification
  - `evidence/b1/motd_hash_after_change.txt` — SHA-256 after change
  - `evidence/b1/motd_grep_change.txt` — grep proof of modification
  - `evidence/b1/snapshot_post_num.txt` — post-change snapshot number
  - `evidence/b1/snapshots_after_post.txt` — snapshot list after post
  - `evidence/b1/rollback_dry_run.txt` — dry-run rollback output
  - `evidence/b1/rollback_output.txt` — actual rollback output
  - `evidence/b1/motd_after_rollback.txt` — file contents post-rollback
  - `evidence/b1/motd_hash_after_rollback.txt` — SHA-256 post-rollback
- **Hash manifest:**

```bash
find evidence/b1 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/b1/SHA256SUMS.txt
cat evidence/b1/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `snapper -c root list` fails with "No config 'root' found."
  - **Cause:** Snapper root config has not been created.
  - **Fix:** `sudo snapper -c root create-config /` then re-verify with `snapper -c root list`.
  - **Rollback trigger:** N/A — no changes made yet.

- **Symptom:** `btrfs subvolume list /` shows no `.snapshots` subvolume.
  - **Cause:** Btrfs subvolume layout does not have a dedicated snapshots subvolume or it is not mounted.
  - **Fix:** Consult the [Arch Wiki Snapper article](https://wiki.archlinux.org/title/Snapper#Configuration_of_snapper_and_mount_point) for proper `.snapshots` mount setup.
  - **Rollback trigger:** N/A — no changes made yet.

- **Symptom:** `undochange` fails with "snapshot not found" or path errors.
  - **Cause:** Snapshot numbers were not captured correctly, or snapshots were deleted.
  - **Fix:** Run `sudo snapper -c root list` to find valid snapshot numbers. If both snapshots exist, re-run the undochange command with correct numbers.
  - **Rollback trigger:** Manual file restoration: `sudo cp evidence/b1/motd_before.txt /etc/motd`

- **Symptom:** Hashes differ after rollback.
  - **Cause:** Another process modified `/etc/motd` between steps, or rollback was incomplete.
  - **Fix:** Inspect `evidence/b1/motd_after_rollback.txt` vs `motd_before.txt` using `diff`. Manually restore from backup copy.
  - **Rollback trigger:** `sudo cp evidence/b1/motd_before.txt /etc/motd`

## 11) Sources

- [Arch Wiki — Snapper](https://wiki.archlinux.org/title/Snapper)
- [Arch Wiki — Btrfs](https://wiki.archlinux.org/title/Btrfs)
- [Snapper project homepage](http://snapper.io/)
- [btrfs-progs documentation](https://btrfs.readthedocs.io/en/latest/)
- [NIST SP 800-61r2 — Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)

## 12) Stretch Goals

- Configure Snapper timeline cleanup (`TIMELINE_CLEANUP=yes`) and verify that old snapshots are automatically pruned.
- Attempt a more impactful but still reversible change: install a test package, snapshot before/after, rollback via `undochange`, confirm package files are removed.
- Write a 10-line shell script (`b1-drill.sh`) that automates all steps and saves outputs to a timestamped evidence directory.
- Read the [Btrfs send/receive documentation](https://btrfs.readthedocs.io/en/latest/Send-receive.html) and export one snapshot as a backup stream: `sudo btrfs send /.snapshots/<NUM>/snapshot | gzip > /tmp/b1-snapshot.btrfs.gz`
