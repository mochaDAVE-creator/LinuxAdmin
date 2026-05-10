---
title: B1 - Snapshot & Rollback Drill
aliases:
  - b1-snapshot-rollback
tags:
  - beginner
  - btrfs
  - snapper
  - rollback
  - proxmox
  - evidence
date: 2026-05-10
---

# B1 — Snapshot & Rollback Drill

## 1) Mission

- **Problem statement:** Operators who make system changes without a rollback path risk unrecoverable downtime. This project drills the pre-change safety habit until it is automatic.
- **Why it matters:** Every subsequent project in this ladder depends on you being able to undo changes safely. Rollback discipline is the single most important operational skill for a Linux administrator.

---

## 2) Difficulty

**Beginner** — Low risk. All changes are reversible by design.

---

## 3) Execution Context

- **Primary path:** Host — Arch Linux bare metal with Btrfs + Snapper
- **Alternate path:** VM — Proxmox virtual machine snapshot (no Snapper required)
- Run on your **lab environment only**. Never run snapshot/rollback drills on shared or production systems without explicit change-management approval.

---

## 4) Prerequisites

### Skills
- Basic shell navigation and file editing
- Familiarity with `sudo`

### Tools
```bash
# Confirm Snapper is installed and a 'root' config exists (Host path)
snapper --version
snapper -c root list | head -5

# Confirm Proxmox qm tool is available (VM path — run on Proxmox node)
qm list
```

### Lab Environment
- **Host path:** Arch Linux with Btrfs root subvolume and Snapper `root` config
- **VM path:** Proxmox node with at least one VM you own and can safely snapshot
- `sudo` privileges on the target system

---

## 5) Rollback Plan

> [!warning] Safety first
> Create your rollback point **before** making any change. If you skip this step and something goes wrong, you have no recovery path.

### Host path (Snapper)
```bash
# Record the pre-change snapshot number after creation
sudo snapper -c root create --description "pre-b1-drill"
snapper -c root list | tail -n 5
# Note the snapshot NUMBER from the output — you will need it for rollback
```

### VM path (Proxmox)
```bash
# Run on the Proxmox node; replace <VMID> with your VM's numeric ID
VMID=<VMID>
qm snapshot "$VMID" pre-b1-drill --description "B1 pre-change snapshot"
qm listsnapshot "$VMID"
```

### Rollback trigger conditions
- Any unexpected file corruption
- Loss of shell access
- The `/etc/motd` change persists after the intended revert step

---

## 6) Project Plan

- **Phase A:** Set up evidence directory and create pre-change snapshot
- **Phase B:** Make a controlled, reversible change and capture evidence
- **Phase C:** Validate the change, then revert and validate restoration
- **Phase D:** Generate hash manifest and complete the report checklist

---

## 7) Walkthrough

### Step 1 — Initialise evidence directory

```bash
TS="$(date -u +%Y%m%d_%H%M%SZ)"
OUT="evidence/b1/${TS}"
mkdir -p "$OUT"
echo "Evidence path: $OUT"
# Export OUT so subsequent steps can reference it in the same shell session
export OUT
```

Expected:
- Directory created without errors
- `$OUT` is set and non-empty

---

### Step 2 — Create pre-change snapshot and record it

#### Host path (Snapper)
```bash
sudo snapper -c root create --description "pre-b1-drill"
snapper -c root list | tee "$OUT/snapshots_before.txt"
# Capture the new snapshot number
PRE_SNAP=$(snapper -c root list | awk '/pre-b1-drill/ {print $1}' | tail -n1)
echo "Pre-change snapshot: $PRE_SNAP" | tee "$OUT/pre_snap_id.txt"
```

Expected:
- New snapshot appears in the list
- `snapshots_before.txt` and `pre_snap_id.txt` saved to `$OUT`

#### VM path (Proxmox)
```bash
VMID=<VMID>
qm snapshot "$VMID" pre-b1-drill --description "B1 pre-change snapshot"
qm listsnapshot "$VMID" | tee "$OUT/vm_snapshots_before.txt"
```

Expected:
- Snapshot `pre-b1-drill` appears in the list
- `vm_snapshots_before.txt` saved to `$OUT`

---

### Step 3 — Make a controlled, reversible change

```bash
# Append a clearly-labelled test line to /etc/motd
echo "# b1-drill-test $(date -u +%Y-%m-%dT%H:%M:%SZ)" | sudo tee -a /etc/motd
# Capture the modified file as evidence
cat /etc/motd | tee "$OUT/motd_modified.txt"
```

Expected:
- The test line appears at the end of `/etc/motd`
- `motd_modified.txt` saved to `$OUT`

---

### Step 4 — Validate the change

```bash
grep "b1-drill-test" /etc/motd | tee "$OUT/motd_grep_result.txt"
```

Expected:
- The grep returns the test line with exit code `0`
- `motd_grep_result.txt` saved to `$OUT`

---

### Step 5 — Revert the change

#### Option A — Manual revert (both paths)
```bash
# Remove only the line added by the drill; preserve any pre-existing content
sudo sed -i '/b1-drill-test/d' /etc/motd
cat /etc/motd | tee "$OUT/motd_after_revert.txt"
```

#### Option B — Snapper rollback (Host path only, when Option A is not sufficient)
```bash
# This restores the subvolume to the pre-change state — use with care
sudo snapper -c root undochange "${PRE_SNAP}..0" /etc/motd
cat /etc/motd | tee "$OUT/motd_after_snapper_revert.txt"
```

> [!warning]
> `snapper undochange` operates on the live filesystem. For a full subvolume rollback, use `snapper rollback` and reboot — only do this if the manual revert fails and you cannot recover the file otherwise.

---

### Step 6 — Create post-change snapshot

#### Host path
```bash
sudo snapper -c root create --description "post-b1-drill"
snapper -c root list | tee "$OUT/snapshots_after.txt"
POST_SNAP=$(snapper -c root list | awk '/post-b1-drill/ {print $1}' | tail -n1)
echo "Post-change snapshot: $POST_SNAP" | tee "$OUT/post_snap_id.txt"
```

#### VM path
```bash
qm snapshot "$VMID" post-b1-drill --description "B1 post-change snapshot"
qm listsnapshot "$VMID" | tee "$OUT/vm_snapshots_after.txt"
```

---

## 8) Validation

Run these commands to confirm the drill completed successfully:

```bash
# 1. Confirm the test line is gone from /etc/motd
grep -c "b1-drill-test" /etc/motd && echo "FAIL: test line still present" \
  || echo "PASS: test line removed"

# 2. Confirm snapshot(s) exist — Host path
snapper -c root list | grep -E "pre-b1-drill|post-b1-drill"

# 3. Confirm evidence files exist
ls -lh "$OUT/"

# 4. Confirm SHA256SUMS.txt will cover all evidence files
find "$OUT" -type f ! -name 'SHA256SUMS.txt'
```

Expected:
- grep exits non-zero (no match) — revert succeeded
- Both snapshots visible in Snapper list
- All evidence files present

---

## 9) Evidence

### Files to capture
- `evidence/b1/<TS>/snapshots_before.txt`
- `evidence/b1/<TS>/pre_snap_id.txt`
- `evidence/b1/<TS>/motd_modified.txt`
- `evidence/b1/<TS>/motd_grep_result.txt`
- `evidence/b1/<TS>/motd_after_revert.txt`
- `evidence/b1/<TS>/snapshots_after.txt`
- `evidence/b1/<TS>/post_snap_id.txt`

### Generate hash manifest
```bash
find "$OUT" -type f ! -name 'SHA256SUMS.txt' -print0 \
  | xargs -0 sha256sum | tee "$OUT/SHA256SUMS.txt"

# Verify the manifest
sha256sum --check "$OUT/SHA256SUMS.txt" && echo "Manifest verified OK"
```

---

## 10) Failure Modes & Recovery

| Symptom | Likely Cause | Fix | Rollback Trigger |
|---------|-------------|-----|-----------------|
| `snapper -c root list` shows no configs | Snapper not configured for Btrfs root | Follow [SRC-ARCH-SNAPPER-01] setup guide | — |
| `sudo tee -a /etc/motd` fails with permission denied | Insufficient sudo rights | Verify sudo config; check `/etc/sudoers` | — |
| `motd` line not removed by `sed` | Pattern mismatch or file encoding issue | Run `grep "b1-drill" /etc/motd` to inspect, then use manual editor | Use Snapper `undochange` if motd is corrupted |
| `qm snapshot` fails | VM is running an operation or VMID is wrong | Wait for running tasks to complete; verify `qm list` | — |
| Snapper rollback changes boot target | Full `snapper rollback` modifies default subvolume | Reboot required; follow recovery in [SRC-ARCH-SNAPPER-01] | — |

---

## 11) Sources

- [SRC-ARCH-SNAPPER-01] — Arch Wiki: Snapper — https://wiki.archlinux.org/title/Snapper
- [SRC-ARCH-BTRFS-01] — Arch Wiki: Btrfs — https://wiki.archlinux.org/title/Btrfs
- [SRC-PVE-ADMIN-01] — Proxmox VE Administration Guide — https://pve.proxmox.com/pve-docs/pve-admin-guide.html

---

## 12) Stretch Goals

- Script the entire drill (snapshot → change → validate → revert → verify → hash) as `scripts/b1_drill.sh`
- Extend the drill to test rollback of a package install (`pacman -S <pkg>` → rollback via Snapper)
- Add a Proxmox PBS (Proxmox Backup Server) restore test as an alternate recovery path
- Map this drill to NIST SP 800-128 configuration management controls

---

## Report Checklist

Fill in before marking B1 complete:

- [ ] Execution context declared: _____________________ (Host / VM)
- [ ] Pre-change snapshot created — ID/Name: _____________________
- [ ] Controlled change made and evidence captured
- [ ] Change validated with grep — result: PASS / FAIL
- [ ] Change reverted — revert method used: _____________________
- [ ] Revert validated — result: PASS / FAIL
- [ ] Post-change snapshot created — ID/Name: _____________________
- [ ] Evidence directory: `evidence/b1/<TS>/`
- [ ] `SHA256SUMS.txt` generated and verified
- [ ] UTC start time: _____________________ — UTC end time: _____________________
