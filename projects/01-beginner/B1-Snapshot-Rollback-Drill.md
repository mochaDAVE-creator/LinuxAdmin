---
title: B1 - Snapshot & Rollback Drill
tags: [beginner, btrfs, snapper, rollback]
---

# B1 — Snapshot & Rollback Drill

## Mission
Prove you can safely perform system changes with rollback guarantees.

## Plan
1. Create pre-change snapshot.
2. Make controlled change (e.g., harmless config edit).
3. Capture evidence.
4. Revert change and validate restoration.

## Walkthrough
```bash
sudo snapper -c root create --description "pre-b1-drill"
snapper -c root list | tail -n 10 | tee evidence/b1/snapshots_before.txt
echo "# b1 test" | sudo tee -a /etc/motd
cat /etc/motd | tee evidence/b1/motd_modified.txt
```

## Validation
```bash
grep -n "b1 test" /etc/motd
```

## Evidence
- `evidence/b1/snapshots_before.txt`
- `evidence/b1/motd_modified.txt`
- `evidence/b1/SHA256SUMS.txt`

## Sources
- [SRC-ARCH-SNAPPER-01]
- [SRC-ARCH-BTRFS-01]
