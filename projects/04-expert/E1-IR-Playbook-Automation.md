---
title: E1 - Incident Response Playbook Automation
aliases:
  - e1-ir-playbook
  - ir-automation-expert
tags:
  - expert
  - incident-response
  - automation
  - forensics
  - evidence
  - net179
  - net412
date: 2026-05-10
---

# E1 — Incident Response Playbook Automation

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host and Proxmox VMs. The automation script collects system state, running processes, network connections, persistence locations, and generates a hashed evidence bundle — all in a single execution with no interactive input required.
> **Risk level:** High — the IR triage script reads from every major system data source. On a production system, running an IR triage generates audit noise that could alert a sophisticated attacker. In the lab, all data collected is local and no external connections are made.
> **Threat model:** The primary threat this playbook defends against is delayed or incomplete triage. Without automation, an analyst under pressure will miss evidence sources. With automation, the first 15 minutes of an incident produce a reproducible, hash-verified evidence bundle regardless of who runs it.
> **Court-defensibility note:** This is a lab exercise. Real forensic investigations require chain-of-custody documentation, write-blocker use for disk imaging, and qualified forensic examiners. The methodology learned here is a starting point, not a complete IR framework.
> **Data sensitivity:** The evidence bundle produced may contain sensitive data (process args, network connections, loaded kernel modules). Handle output as confidential. Do not commit real triage outputs to public repositories.

## 1) Mission

- **Problem statement:** When an incident occurs, the first responder has minutes to capture volatile data before it is lost (process list, memory-mapped files, network connections, active sessions). A scripted playbook ensures consistent, complete, and hash-verified triage every time — regardless of time pressure or analyst experience level.
- **Why it matters:** IR automation is a senior-level NET 179 and NET 412 competency. This playbook directly implements NIST SP 800-61r2 Section 3.3 (Detection and Analysis) and produces outputs mapped to MITRE ATT&CK tactic evidence for each major tactic category.

## 2) Difficulty

- Expert (estimated 12–20 focused hours to write, test, and refine the full playbook)

## 3) Execution Context

- **Host and VM** — The IR script can run on any Linux system with systemd. Design it to be portable — it should work on the Arch bare-metal host AND be deployable to Proxmox VMs via SSH.

## 4) Prerequisites

- **Skills:** Completion of all beginner, intermediate, and A1/A3 projects. Strong Bash scripting. Understanding of all major evidence sources from B2, B4, I2. Familiarity with ATT&CK tactics.
- **Tools:** `ps`, `ss`, `journalctl`, `find`, `sha256sum`, `lsof`, `lsmod`, `netstat`, `auditctl`, `tee`, `tar`, `gpg` (optional)
- **Dependencies:** `sudo` access or root access on the target system. `lsof` package (`pacman -S lsof`).

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-e1-ir-playbook"`
- **Host Snapper post:** `sudo snapper -c root create --description "post-e1-ir-playbook"`
- **VM snapshot (pre):** `qm snapshot <VMID> pre-e1-ir` for any VMs where the playbook is tested.
- **Container rebuild command:** N/A
- **Rollback trigger:** The IR playbook is read-only and does not modify the target system. Rollback is only needed if the `auditd` rule additions (if any) cause issues. `sudo auditctl -D` clears all temporary audit rules.

## 6) Project Plan

- **Phase A — Evidence Collection Script:** Write the main `ir-triage.sh` script covering all required evidence categories.
- **Phase B — Hash and Package:** Add integrity hashing, timestamped output directory, and optional encryption.
- **Phase C — ATT&CK Annotation:** Map collected artifacts to ATT&CK tactic categories and produce an annotated summary.
- **Phase D — Remote Deployment:** Test the playbook over SSH against a Proxmox lab VM.

## 7) Walkthrough

### Step 1 — Pre-snapshot and environment setup

```bash
mkdir -p evidence/e1

sudo snapper -c root create --description "pre-e1-ir-playbook" --print-number \
  | tee evidence/e1/snapshot_pre_num.txt

# Install lsof if not present
pacman -Q lsof 2>/dev/null || sudo pacman -S --noconfirm lsof
```

### Step 2 — Write the IR triage script

```bash
cat | tee evidence/e1/ir-triage.sh << 'SCRIPT_EOF'
#!/usr/bin/env bash
# =============================================================================
# ir-triage.sh — E1 Incident Response Triage Script
# =============================================================================
# Collects volatile and non-volatile system data for incident analysis.
# Output: timestamped directory with hashed evidence bundle.
# Usage: sudo bash ir-triage.sh [output_dir]
# =============================================================================

set -euo pipefail

TRIAGE_ID="$(hostname -s)-$(date -u +%Y%m%dT%H%M%SZ)"
OUT="${1:-/tmp/ir-triage-$TRIAGE_ID}"
mkdir -p "$OUT"

log() { echo "[$(date -u +%H:%M:%SZ)] $*" | tee -a "$OUT/triage.log"; }
capture() {
  local label="$1"; shift
  log "Collecting: $label"
  { "$@" 2>&1; } | tee "$OUT/${label}.txt" > /dev/null
}

log "=== IR Triage Started: $TRIAGE_ID ==="
log "Output directory: $OUT"
log "Operator: $(id)"

# ---- Phase 1: System Identity ----
capture "01-uname"           uname -a
capture "01-hostname"        hostname -f
capture "01-os-release"      cat /etc/os-release
capture "01-date-utc"        date -u
capture "01-uptime"          uptime
capture "01-who"             who
capture "01-last"            last -n 50

# ---- Phase 2: Process Snapshot (TA0002 Execution) ----
capture "02-ps-full"         ps auxwwef
capture "02-pstree"          pstree -p 2>/dev/null || ps -ejH
capture "02-proc-cmdlines"   sh -c 'for pid in /proc/[0-9]*/cmdline; do printf "%s: " "${pid%/cmdline}"; cat "$pid" | tr "\0" " "; echo; done'

# Map /proc/<PID>/exe for all processes
log "Collecting: 02-proc-exe-map"
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  exe=$(readlink /proc/$pid/exe 2>/dev/null || echo "[no-exe]")
  printf "%s\t%s\n" "$pid" "$exe"
done > "$OUT/02-proc-exe-map.txt"

# ---- Phase 3: Network State (TA0011 Command and Control) ----
capture "03-ss-all"          ss -tulpen
capture "03-ss-raw"          ss -apen
capture "03-ip-addr"         ip addr
capture "03-ip-route"        ip route
capture "03-arp"             ip neigh
capture "03-lsof-network"    lsof -nP -i 2>/dev/null || echo "lsof not available"
capture "03-netstat"         netstat -tulpen 2>/dev/null || echo "netstat not available"

# ---- Phase 4: Authentication (TA0006 Credential Access) ----
capture "04-logged-in"       w
capture "04-last-logins"     last -n 100
capture "04-auth-journal"    journalctl _SYSTEMD_UNIT=sshd.service -n 500 --no-pager
capture "04-sudo-journal"    journalctl --no-pager --since "24 hours ago" | grep -i "sudo\|su\[" | head -100
capture "04-shadow-stat"     stat /etc/shadow
capture "04-passwd-accounts" getent passwd

# List all authorized_keys files
log "Collecting: 04-authorized-keys"
find /home /root -name "authorized_keys" 2>/dev/null \
  | while read -r f; do echo "=== $f ==="; cat "$f"; echo; done \
  > "$OUT/04-authorized-keys.txt"

# ---- Phase 5: Persistence (TA0003 Persistence) ----
capture "05-systemd-units"   find /etc/systemd/system /lib/systemd/system -maxdepth 2 -name "*.service" -o -name "*.timer" 2>/dev/null
capture "05-enabled-units"   systemctl list-unit-files --state=enabled --no-pager
capture "05-active-timers"   systemctl list-timers --all --no-pager
capture "05-cron-system"     sh -c 'cat /etc/crontab 2>/dev/null; ls -la /etc/cron.d/ 2>/dev/null; cat /etc/cron.d/* 2>/dev/null'
capture "05-cron-spool"      find /var/spool/cron -type f 2>/dev/null | xargs cat 2>/dev/null
capture "05-profile-d"       cat /etc/profile.d/*.sh 2>/dev/null
capture "05-shell-init"      find /home /root -maxdepth 2 -name ".bashrc" -o -name ".profile" -o -name ".bash_profile" 2>/dev/null

# ---- Phase 6: Kernel and Modules (TA0005 Defense Evasion) ----
capture "06-lsmod"           lsmod
capture "06-dmesg-err"       dmesg --level=err,crit,alert,emerg --ctime 2>/dev/null || dmesg | grep -iE "error|crit|alert|emerg"
capture "06-suid-binaries"   find / -xdev -perm -4000 -type f 2>/dev/null

# ---- Phase 7: Recent File Activity (TA0009 Collection) ----
capture "07-recently-modified-etc"   find /etc -newer /etc/os-release -type f -maxdepth 3 2>/dev/null
capture "07-recently-modified-tmp"   find /tmp /var/tmp -newer /etc/os-release -type f 2>/dev/null
capture "07-recently-modified-usr-local" find /usr/local/bin /usr/local/sbin -newer /etc/os-release -type f 2>/dev/null
capture "07-var-tmp-contents"        find /var/tmp -type f 2>/dev/null

# ---- Phase 8: Log Summary (TA0005 Defense Evasion / TA0010 Exfiltration) ----
capture "08-journal-errors"  journalctl -b -p err --no-pager
capture "08-journal-boot"    journalctl -b --no-pager -n 500
capture "08-journal-prev"    journalctl -b -1 --no-pager -n 200 2>/dev/null || echo "No previous boot journal"

# ---- Phase 9: Integrity Manifest ----
log "Generating SHA-256 integrity manifest"
find "$OUT" -type f ! -name "SHA256SUMS.txt" ! -name "triage.log" -print0 \
  | xargs -0 sha256sum | sort > "$OUT/SHA256SUMS.txt"

log "=== IR Triage Complete ==="
log "Evidence directory: $OUT"
log "File count: $(find "$OUT" -type f | wc -l)"
log "Total size: $(du -sh "$OUT" | cut -f1)"

echo ""
echo "Evidence bundle: $OUT"
echo "Integrity manifest: $OUT/SHA256SUMS.txt"
SCRIPT_EOF

chmod +x evidence/e1/ir-triage.sh
```

Expected:
- `evidence/e1/ir-triage.sh` is created and executable.
- Script is ~120 lines of Bash, readable and well-commented.

### Step 3 — Execute the triage script and capture output

```bash
# Run the IR triage (as root for full data access)
IR_OUT="/tmp/ir-e1-$(date -u +%Y%m%dT%H%M%SZ)"
sudo bash evidence/e1/ir-triage.sh "$IR_OUT" 2>&1 | tee evidence/e1/ir_triage_run.txt

# Verify output
ls -lah "$IR_OUT/" | tee evidence/e1/ir_output_listing.txt
echo "Evidence files: $(find "$IR_OUT" -type f | wc -l)" | tee evidence/e1/ir_output_counts.txt
echo "Total size: $(du -sh "$IR_OUT" | cut -f1)" | tee -a evidence/e1/ir_output_counts.txt

# Verify hash manifest
echo "Verifying SHA-256 manifest..." | tee evidence/e1/ir_hash_verify.txt
cd "$IR_OUT" && sha256sum -c SHA256SUMS.txt 2>&1 | head -30 | tee -a /tmp/ir_hash_verify_result.txt
cd -
cat /tmp/ir_hash_verify_result.txt | tee -a evidence/e1/ir_hash_verify.txt
```

Expected:
- All phase output files present (01- through 09-).
- `SHA256SUMS.txt` present with hash entries.
- Hash verification shows `OK` for all files.

### Step 4 — ATT&CK-annotated summary report

```bash
cat | tee evidence/e1/ir_attack_mapping.md << 'EOF'
# E1 — IR Triage ATT&CK Coverage Map

| Evidence File | ATT&CK Tactic | Technique |
|---|---|---|
| 02-ps-full.txt | TA0002 Execution | T1059 Command/Script Interpreter |
| 02-proc-exe-map.txt | TA0005 Defense Evasion | T1036 Masquerading |
| 03-ss-all.txt | TA0011 C2 | T1071 App Layer Protocol |
| 03-lsof-network.txt | TA0011 C2 | T1095 Non-Std Port |
| 04-auth-journal.txt | TA0006 Credential Access | T1110 Brute Force |
| 04-authorized-keys.txt | TA0003 Persistence | T1098.004 SSH Authorized Keys |
| 05-systemd-units.txt | TA0003 Persistence | T1543.002 Systemd Service |
| 05-cron-spool.txt | TA0003 Persistence | T1053.003 Cron |
| 05-profile-d.txt | TA0003 Persistence | T1546.004 Unix Shell Profile |
| 06-lsmod.txt | TA0005 Defense Evasion | T1014 Rootkit |
| 06-suid-binaries.txt | TA0004 Privilege Escalation | T1548.001 Setuid/Setgid |
| 07-recently-modified-*.txt | TA0009 Collection | T1005 Local Data |
| 08-journal-errors.txt | TA0005 Defense Evasion | T1070 Log Clearing |

## Investigation Workflow

1. Start with `04-auth-journal.txt` — identify any failed/successful logins
2. Review `05-*` files — look for unpackaged persistence mechanisms
3. Review `02-proc-exe-map.txt` — identify processes running from unusual paths (e.g., /tmp, /dev/shm)
4. Review `03-ss-all.txt` — look for unexpected listening ports or outbound connections
5. Review `06-suid-binaries.txt` — identify any SUID binaries not owned by package manager
6. Cross-reference `SHA256SUMS.txt` timestamps against `01-date-utc.txt` for timeline integrity
EOF

cat evidence/e1/ir_attack_mapping.md
```

### Step 5 — Test remote deployment (SSH to a Proxmox VM)

```bash
# Deploy and run the IR triage on a remote Proxmox VM
VM_IP="<VM_IP>"  # Replace with actual VM IP

if [ -n "$VM_IP" ] && [ "$VM_IP" != "<VM_IP>" ]; then
  # Copy triage script to remote VM
  scp evidence/e1/ir-triage.sh root@${VM_IP}:/tmp/
  
  # Execute remotely and retrieve results
  ssh root@${VM_IP} "bash /tmp/ir-triage.sh /tmp/ir-remote-triage" 2>&1 \
    | tee evidence/e1/ir_remote_run.txt
  
  # Pull evidence bundle back
  scp -r root@${VM_IP}:/tmp/ir-remote-triage/ evidence/e1/remote-evidence/ 2>&1 \
    | tee evidence/e1/ir_remote_copy.txt
  
  echo "Remote triage files: $(find evidence/e1/remote-evidence -type f | wc -l)" \
    | tee evidence/e1/ir_remote_summary.txt
else
  echo "Remote deployment: set VM_IP variable to test against a Proxmox VM" \
    | tee evidence/e1/ir_remote_run.txt
fi
```

## 8) Validation

```bash
# Triage script is executable
[ -x evidence/e1/ir-triage.sh ] \
  && echo "PASS: ir-triage.sh is executable" \
  || echo "FAIL: ir-triage.sh not executable"

# All 8 phase categories are covered
for phase in 01 02 03 04 05 06 07 08; do
  found=$(ls "${IR_OUT}/${phase}-"*.txt 2>/dev/null | wc -l)
  [ "$found" -gt 0 ] \
    && echo "PASS: Phase $phase covered ($found files)" \
    || echo "FAIL: Phase $phase missing"
done

# SHA256SUMS.txt exists and is non-empty
[ -s "${IR_OUT}/SHA256SUMS.txt" ] \
  && echo "PASS: SHA256 manifest present" \
  || echo "FAIL: SHA256 manifest missing"

# ATT&CK mapping exists
[ -s evidence/e1/ir_attack_mapping.md ] \
  && echo "PASS: ATT&CK mapping written" \
  || echo "FAIL: ATT&CK mapping missing"
```

## 9) Evidence

- **Output files:**
  - `evidence/e1/snapshot_pre_num.txt` — Snapper pre-snapshot
  - `evidence/e1/ir-triage.sh` — the IR triage automation script
  - `evidence/e1/ir_triage_run.txt` — script execution log
  - `evidence/e1/ir_output_listing.txt` — evidence directory listing
  - `evidence/e1/ir_output_counts.txt` — file count and total size
  - `evidence/e1/ir_hash_verify.txt` — hash verification results
  - `evidence/e1/ir_attack_mapping.md` — ATT&CK coverage map
  - `evidence/e1/ir_remote_run.txt` — remote deployment output (if tested)
  - `/tmp/ir-e1-<timestamp>/` — full triage evidence bundle (keep locally, do not commit)
- **Hash manifest:**

```bash
find evidence/e1 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/e1/SHA256SUMS.txt
cat evidence/e1/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `lsof -nP -i` takes extremely long or hangs.
  - **Cause:** `lsof` can hang on certain NFS or FUSE mounts. The script should have a timeout.
  - **Fix:** Add `timeout 30` prefix: `timeout 30 lsof -nP -i 2>/dev/null`. Add this to the capture function.
  - **Rollback trigger:** Kill the hung script: `kill <pid>`. Evidence collected up to that point is still valid.

- **Symptom:** SHA-256 verification fails for some files.
  - **Cause:** Evidence files were modified after manifest generation (another process writing to the system while triage runs).
  - **Fix:** Note which files failed verification in the report. These may indicate a noisy environment or actual tampering. Re-run the hash manifest: `find "$IR_OUT" -type f ! -name "SHA256SUMS.txt" -print0 | xargs -0 sha256sum | sort > "$IR_OUT/SHA256SUMS.txt"`
  - **Rollback trigger:** N/A.

- **Symptom:** Remote SSH deployment fails with "Connection refused."
  - **Cause:** SSH is not running on the VM or the VM IP has changed.
  - **Fix:** Check VM status: `qm status <VMID>`. Get VM IP: `qm agent <VMID> network-get-interfaces`.
  - **Rollback trigger:** N/A.

## 11) Sources

- [NIST SP 800-61r2 — Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)
- [MITRE ATT&CK — Enterprise Tactics](https://attack.mitre.org/tactics/enterprise/)
- [SANS Institute — Linux IR cheat sheet](https://www.sans.org/posters/linux-shell-survival-guide/)
- [CISA — Incident Response Best Practices](https://www.cisa.gov/sites/default/files/publications/CISA_MS-ISAC_Ransomware%20Guide_S508C_.pdf)
- [man lsof — list open files](https://man7.org/linux/man-pages/man8/lsof.8.html)

## 12) Stretch Goals

- Add encryption to the evidence bundle using `gpg --symmetric` with a passphrase. Document key management and verify that the bundle can be decrypted and verified on a different machine.
- Extend the script to collect memory artifacts: `/proc/<PID>/maps`, `/proc/<PID>/smaps` for suspicious processes (high memory usage, mapped from `/tmp`).
- Integrate with the A1 centralized logging pipeline: after triage completes, `logger` a JSON summary to the local syslog so it appears in the remote collector.
- Write a parallel version that runs multiple evidence collection phases simultaneously using Bash `&` and `wait`, measuring the speedup vs sequential execution.

