---
title: I4 - Proxmox Backup Validation
aliases:
  - i4-proxmox-backup
  - pve-backup-validation-intermediate
tags:
  - intermediate
  - proxmox
  - backup
  - restore
  - evidence
  - net412
date: 2026-05-10
---

# I4 — Proxmox Backup Validation

> [!warning] Operating Assumptions & Threat Model
> **Context:** Local Proxmox VE hypervisor lab (running on bare-metal Arch host or dedicated hardware). A test VM exists in Proxmox that is safe to backup, restore, and snapshot. **Do not use production VMs for this exercise** — use a dedicated lab VM.
> **Risk level:** Medium — VM snapshots and restores temporarily consume disk space and halt or pause the target VM. If disk space is insufficient, backup operations will fail and may leave the VM in a degraded state.
> **Threat model:** An untested backup is not a backup — it is a false sense of security. This project validates that the complete backup→restore cycle works and that restored VMs are functionally intact. It directly maps to the business continuity and DR requirements of any production Proxmox deployment.
> **Out of scope:** External backup targets (NFS, PBS — Proxmox Backup Server), tape/offsite backup. Those are advanced configurations. This project uses local `vzdump` and Proxmox built-in storage.
> **VM selection:** Use a VM with a small disk (≤20 GB) to keep backup times manageable. A Debian minimal install or Alpine Linux VM is ideal.

## 1) Mission

- **Problem statement:** Proxmox VE provides powerful snapshot and backup capabilities, but many administrators never test the restore path until a real disaster. This project forces a complete backup→restore cycle, validates the result, and produces evidence that the process works under controlled conditions.
- **Why it matters:** Backup validation is a core NET 412 and IT operations competency. The Proxmox backup toolchain (`vzdump`, `qmrestore`, `pvesm`) is the primary DR mechanism for the entire lab infrastructure. Knowing exactly how long backups take, how much space they consume, and how long restores take is essential for writing an SLA or incident response runbook.

## 2) Difficulty

- Intermediate (estimated 4–7 focused hours, including VM start/stop wait times)

## 3) Execution Context

- **VM** on Proxmox VE — backup and restore operations target a dedicated lab VM. The Proxmox host CLI (`qm`, `pvesm`, `vzdump`) is run from the Proxmox node shell (SSH or console).

## 4) Prerequisites

- **Skills:** Completion of B1 (Snapshot & Rollback Drill) for snapshot concepts. Basic Proxmox web UI navigation. SSH access to the Proxmox node.
- **Tools:** `vzdump`, `qm`, `pvesm`, `qmrestore`, `sha256sum`, `tee`, `journalctl` (on Proxmox node)
- **Dependencies:**
  - Proxmox VE ≥ 7.0 installed and licensed (community OK).
  - A test VM with a unique VMID (e.g., 101).
  - Local storage configured in Proxmox with `vzdump` content type enabled.
  - Sufficient free disk space: at least 2× the VM disk size.

## 5) Rollback Plan

- **Host Snapper pre:** N/A — Snapper is for the Arch host, not the Proxmox node.
- **VM snapshot (pre-backup test):** Create a Proxmox snapshot before starting: `qm snapshot <VMID> pre-i4-backup --description "pre-i4-backup-validation"`
- **VM snapshot rollback:** `qm rollback <VMID> pre-i4-backup`
- **Container rebuild command:** N/A
- **Restore rollback:** If the restore creates a corrupted VM state, delete the restored VM: `qm destroy <RESTORED_VMID> --purge`. The original VM (if kept) can be started from the pre-backup snapshot.

## 6) Project Plan

- **Phase A — Environment Assessment:** Document the Proxmox environment, storage configuration, and VM state.
- **Phase B — Backup Execution:** Run `vzdump` backup of the test VM, capture timing and size data.
- **Phase C — Restore and Validation:** Restore the backup to a new VMID, start the restored VM, and validate it is functionally intact.

## 7) Walkthrough

### Step 1 — Proxmox node environment assessment

> [!info] Run all commands in this project on the Proxmox VE node shell via SSH: `ssh root@<PROXMOX_IP>` or through the Proxmox web console (Node → Shell).

```bash
mkdir -p evidence/i4

# Record Proxmox version
pveversion -v | tee evidence/i4/pve_version.txt

# List available storage locations and their content types
pvesm status | tee evidence/i4/pve_storage_status.txt
pvesm list local 2>/dev/null | head -30 | tee evidence/i4/pve_storage_local_content.txt

# List all VMs and their current state
qm list | tee evidence/i4/pve_vm_list.txt

# Define the test VM ID (replace 101 with your actual VMID)
VMID=101

# Show VM configuration
qm config $VMID | tee evidence/i4/vm_config_before.txt

# Show VM current status
qm status $VMID | tee evidence/i4/vm_status_before.txt

# Show VM disk usage
qm df $VMID 2>/dev/null | tee evidence/i4/vm_disk_usage.txt || \
  pvesm list local --vmid $VMID | tee evidence/i4/vm_disk_usage.txt
```

Expected:
- `pve_version.txt` shows Proxmox VE version (7.x or 8.x).
- `pve_storage_status.txt` shows at least `local` storage with `vzdump` content enabled.
- `pve_vm_list.txt` shows your target VMID with its name and current state.

### Step 2 — Create pre-backup VM snapshot

```bash
VMID=101

# Create a Proxmox snapshot (RAM state included if VM is running)
qm snapshot $VMID pre-i4-backup-validation \
  --description "Pre-I4 backup validation drill - $(date -u +%Y-%m-%dT%H:%M:%SZ)"

# List snapshots to confirm creation
qm listsnapcon $VMID 2>/dev/null | tee evidence/i4/vm_snapshots_before.txt \
  || qm config $VMID | grep "\[" | tee evidence/i4/vm_snapshots_before.txt

echo "Pre-backup snapshot created at $(date -u)" | tee evidence/i4/snapshot_pre_timestamp.txt
```

Expected:
- Snapshot appears in `vm_snapshots_before.txt` with the description containing "pre-i4-backup-validation".

### Step 3 — Execute vzdump backup

```bash
VMID=101
BACKUP_STORAGE="local"  # Replace with your storage ID if different

# Record disk space before backup
df -h | tee evidence/i4/disk_space_before_backup.txt

# Start backup — use snapshot mode to avoid downtime (preferred)
# Or use suspend mode if snapshot mode fails: --mode suspend
echo "Backup started: $(date -u)" | tee evidence/i4/backup_timing.txt

vzdump $VMID \
  --storage "$BACKUP_STORAGE" \
  --mode snapshot \
  --compress zstd \
  --notes-template "i4-backup-validation-$(date -u +%Y%m%d)" \
  --remove 0 \
  2>&1 | tee evidence/i4/backup_output.txt

echo "Backup completed: $(date -u)" | tee -a evidence/i4/backup_timing.txt

# Record disk space after backup
df -h | tee evidence/i4/disk_space_after_backup.txt

# List the backup file created
pvesm list "$BACKUP_STORAGE" --vmid $VMID | tee evidence/i4/backup_file_listing.txt

# Find the backup file path and hash it
BACKUP_FILE=$(pvesm list "$BACKUP_STORAGE" --vmid $VMID | grep "vzdump" | awk '{print $1}')
echo "Backup file: $BACKUP_FILE" | tee evidence/i4/backup_file_info.txt
```

Expected:
- `backup_output.txt` shows `INFO: Backup job finished successfully.`
- `backup_timing.txt` shows start and end times — calculate duration.
- `backup_file_listing.txt` shows the `vzdump-qemu-<VMID>-*.vma.zst` file with its size.

### Step 4 — Locate and verify backup file integrity

```bash
BACKUP_STORAGE="local"
VMID=101

# Find the actual path to the backup file
BACKUP_PATH=$(find /var/lib/vz/dump/ /mnt/ -name "vzdump-qemu-${VMID}-*.vma*" 2>/dev/null | head -1)
echo "Backup file path: $BACKUP_PATH" | tee evidence/i4/backup_file_path.txt

# Hash the backup file
if [ -f "$BACKUP_PATH" ]; then
  sha256sum "$BACKUP_PATH" | tee evidence/i4/backup_file_hash.txt
  ls -lah "$BACKUP_PATH" | tee evidence/i4/backup_file_stat.txt
  
  # Also check the Proxmox log checksum if present
  LOG_FILE="${BACKUP_PATH%.vma.zst}.log"
  [ -f "$LOG_FILE" ] && tail -5 "$LOG_FILE" | tee evidence/i4/backup_log_tail.txt
else
  echo "ERROR: Could not locate backup file" | tee evidence/i4/backup_file_path.txt
fi
```

Expected:
- `backup_file_hash.txt` contains a SHA-256 hash of the backup archive.
- `backup_file_stat.txt` shows file size consistent with the VM's disk usage.

### Step 5 — Restore backup to a new VMID

```bash
VMID=101
RESTORED_VMID=199  # Use a VMID not in use
BACKUP_STORAGE="local"

BACKUP_PATH=$(cat evidence/i4/backup_file_path.txt | awk '{print $NF}')

# Record current VM list before restore
qm list | tee evidence/i4/vm_list_before_restore.txt

echo "Restore started: $(date -u)" | tee evidence/i4/restore_timing.txt

# Restore to new VMID
qmrestore "$BACKUP_PATH" $RESTORED_VMID \
  --storage "$BACKUP_STORAGE" \
  --force 1 \
  2>&1 | tee evidence/i4/restore_output.txt

echo "Restore completed: $(date -u)" | tee -a evidence/i4/restore_timing.txt

# Verify the restored VM appears
qm list | tee evidence/i4/vm_list_after_restore.txt
qm config $RESTORED_VMID | tee evidence/i4/restored_vm_config.txt
```

Expected:
- `restore_output.txt` shows the restore completing without errors.
- `vm_list_after_restore.txt` shows VMID 199 (or your chosen restored VMID) present.
- `restored_vm_config.txt` shows the same disk and hardware configuration as the original.

### Step 6 — Start restored VM and validate functional integrity

```bash
RESTORED_VMID=199

# Start the restored VM
qm start $RESTORED_VMID 2>&1 | tee evidence/i4/restored_vm_start.txt
sleep 30  # Allow VM to boot

# Check VM status
qm status $RESTORED_VMID | tee evidence/i4/restored_vm_status.txt

# Get the VM's IP address (if QEMU Guest Agent is installed)
qm agent $RESTORED_VMID network-get-interfaces 2>/dev/null \
  | tee evidence/i4/restored_vm_network.txt \
  || echo "Guest agent not available — check VM console manually" | tee evidence/i4/restored_vm_network.txt

# SSH into the restored VM and run integrity checks (if network is available)
RESTORED_IP=$(qm agent $RESTORED_VMID network-get-interfaces 2>/dev/null \
  | python3 -c "import sys,json; ifaces=json.load(sys.stdin); \
    [print(ip['ip-address']) for iface in ifaces['result'] \
    for ip in iface.get('ip-addresses',[]) \
    if ip['ip-address-type']=='ipv4' and not ip['ip-address'].startswith('127')]" 2>/dev/null)

if [ -n "$RESTORED_IP" ]; then
  ssh root@$RESTORED_IP "uname -a; df -h; systemctl --failed" 2>/dev/null \
    | tee evidence/i4/restored_vm_integrity_check.txt
else
  echo "Check VM console manually — record: uname -a, df -h, systemctl --failed" \
    | tee evidence/i4/restored_vm_integrity_check.txt
fi
```

Expected:
- `restored_vm_status.txt` shows `status: running`.
- `restored_vm_integrity_check.txt` shows the same kernel version and disk layout as the original VM backup.
- `systemctl --failed` inside the restored VM shows no failed units.

### Step 7 — Cleanup: stop and remove restored VM

```bash
RESTORED_VMID=199
VMID=101

# Stop the restored VM
qm stop $RESTORED_VMID 2>&1 | tee evidence/i4/restored_vm_stop.txt
sleep 10

# Destroy the restored VM (cleanup — the original backup file is retained)
qm destroy $RESTORED_VMID --purge 2>&1 | tee evidence/i4/restored_vm_destroy.txt

# Confirm cleanup
qm list | tee evidence/i4/vm_list_final.txt
grep $RESTORED_VMID evidence/i4/vm_list_final.txt \
  && echo "WARNING: Restored VM $RESTORED_VMID still present" \
  || echo "PASS: Restored VM $RESTORED_VMID successfully removed"

# Record final disk space
df -h | tee evidence/i4/disk_space_final.txt
```

Expected:
- `restored_vm_destroy.txt` shows successful deletion.
- `vm_list_final.txt` does not contain VMID 199.

## 8) Validation

```bash
# Confirm backup file exists and is hashed
[ -f evidence/i4/backup_file_hash.txt ] \
  && echo "PASS: Backup file hash captured" \
  || echo "FAIL: Backup file hash missing"

# Confirm backup was reported successful
grep -qi "finished successfully" evidence/i4/backup_output.txt \
  && echo "PASS: Backup completed successfully" \
  || echo "FAIL: Backup may not have completed cleanly"

# Confirm restore was reported successful
grep -qi "successfully restored" evidence/i4/restore_output.txt \
  || grep -qi "restore job finished" evidence/i4/restore_output.txt \
  && echo "PASS: Restore completed successfully" \
  || echo "REVIEW: Check restore_output.txt for errors"

# Confirm restored VM was running
grep -q "running" evidence/i4/restored_vm_status.txt \
  && echo "PASS: Restored VM reached running state" \
  || echo "FAIL: Restored VM did not reach running state"

# Calculate and report total elapsed time
echo "Backup timing:" && cat evidence/i4/backup_timing.txt
echo "Restore timing:" && cat evidence/i4/restore_timing.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/i4/pve_version.txt` — Proxmox VE version
  - `evidence/i4/pve_storage_status.txt` — storage configuration
  - `evidence/i4/pve_vm_list.txt` — VM inventory
  - `evidence/i4/vm_config_before.txt` — target VM configuration
  - `evidence/i4/vm_status_before.txt` — target VM state before backup
  - `evidence/i4/vm_disk_usage.txt` — VM disk size data
  - `evidence/i4/vm_snapshots_before.txt` — snapshot inventory
  - `evidence/i4/snapshot_pre_timestamp.txt` — pre-backup snapshot timestamp
  - `evidence/i4/disk_space_before_backup.txt` — disk space before
  - `evidence/i4/backup_output.txt` — vzdump backup output
  - `evidence/i4/backup_timing.txt` — backup start/end times
  - `evidence/i4/disk_space_after_backup.txt` — disk space after backup
  - `evidence/i4/backup_file_listing.txt` — backup file in storage
  - `evidence/i4/backup_file_path.txt` — absolute path to backup
  - `evidence/i4/backup_file_hash.txt` — SHA-256 of backup archive
  - `evidence/i4/backup_file_stat.txt` — backup file metadata
  - `evidence/i4/backup_log_tail.txt` — Proxmox backup log tail
  - `evidence/i4/vm_list_before_restore.txt` — VM list before restore
  - `evidence/i4/restore_timing.txt` — restore start/end times
  - `evidence/i4/restore_output.txt` — qmrestore output
  - `evidence/i4/vm_list_after_restore.txt` — VM list after restore
  - `evidence/i4/restored_vm_config.txt` — restored VM configuration
  - `evidence/i4/restored_vm_start.txt` — restored VM start output
  - `evidence/i4/restored_vm_status.txt` — restored VM status
  - `evidence/i4/restored_vm_network.txt` — restored VM IP info
  - `evidence/i4/restored_vm_integrity_check.txt` — in-VM functional check
  - `evidence/i4/restored_vm_stop.txt` — stop output
  - `evidence/i4/restored_vm_destroy.txt` — cleanup output
  - `evidence/i4/vm_list_final.txt` — final VM inventory
  - `evidence/i4/disk_space_final.txt` — final disk space
- **Hash manifest:**

```bash
find evidence/i4 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/i4/SHA256SUMS.txt
cat evidence/i4/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `vzdump` fails with "lvm snapshot failed" or "unable to create snapshot."
  - **Cause:** Insufficient disk space for the snapshot COW (Copy-on-Write) layer, or the storage backend does not support snapshots.
  - **Fix:** Use `--mode stop` instead of `--mode snapshot` (VM will be stopped briefly): `vzdump $VMID --mode stop ...`. Free up disk space before retrying.
  - **Rollback trigger:** Original VM is unaffected — no rollback needed.

- **Symptom:** `qmrestore` fails with "target storage has not enough space."
  - **Cause:** The restore target storage does not have enough free space for the VM disk.
  - **Fix:** `df -h` and `pvesm status` to check free space. Delete old backups or use a different target storage.
  - **Rollback trigger:** N/A — restore was not initiated.

- **Symptom:** Restored VM boots but shows filesystem errors inside.
  - **Cause:** Backup captured the VM in an inconsistent state (mid-write). This can happen with snapshot mode on some filesystems.
  - **Fix:** Boot the restored VM, run `fsck` on the affected partition from a rescue mode or live CD. For future backups, ensure applications are quiesced before backup (use QEMU Guest Agent quiescing: `vzdump --quiesce`).
  - **Rollback trigger:** Destroy the restored VM and re-try backup with `--quiesce` flag.

## 11) Sources

- [Proxmox VE Administration Guide — Backup and Restore](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_vzdump)
- [Proxmox VE documentation — vzdump man page](https://pve.proxmox.com/wiki/Backup_and_Restore)
- [Proxmox VE documentation — qm command reference](https://pve.proxmox.com/pve-docs/qm.1.html)
- [NIST SP 800-61r2 — Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)
- [Arch Wiki — Proxmox](https://wiki.archlinux.org/title/Proxmox)

## 12) Stretch Goals

- Set up Proxmox Backup Server (PBS) in a separate VM and configure daily incremental backups. Compare deduplication storage efficiency against the local `vzdump` approach.
- Write a backup validation script (`i4-pve-backup-verify.sh`) that automatically runs the backup→restore→integrity check→cleanup cycle and emails or writes a markdown report.
- Test backup of a container (LXC) instead of a VM using `vzdump` with `--type lxc`. Compare the backup format and restore procedure.
- Document your Proxmox lab's complete backup schedule as a runbook: which VMs are backed up, to which storage, on what schedule, with what retention policy, and what the restore time objective (RTO) is for each.
