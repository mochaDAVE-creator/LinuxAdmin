---
title: E4 - Purple-Team Validation Cycle
aliases:
  - e4-purple-team
  - adversary-emulation-cycle
tags:
  - expert
  - purple-team
  - adversary-emulation
  - detection
  - validation
  - mitre-attack
  - evidence
  - net179
  - net377
  - net412
date: 2026-05-10
---

# E4 — Purple-Team Validation Cycle

> [!warning] Operating Assumptions & Threat Model
> **Context:** Isolated Proxmox lab environment. All emulation activities are performed inside the lab — never against production systems, external hosts, or systems without explicit written authorization. The lab network is fully air-gapped from production for this exercise.
> **Risk level:** High — adversary emulation generates intentional security events. If detection rules are misconfigured, emulated activities could be mistaken for real attacks by monitoring systems. All activity is documented and scoped.
> **Safety hard rules:**
> 1. ALL emulation runs in the Proxmox lab only — NEVER on production or external systems.
> 2. Every emulation step is preceded by a VM snapshot.
> 3. No actual exploit code, malware, or real attack tools are used. Only benign simulation commands that produce the same telemetry signature as the mapped technique.
> 4. All network emulation is isolated to the lab bridge (vmbr2 from A2).
> **Legal/ethical context:** Unauthorized penetration testing or adversary emulation against systems you do not own is illegal. This project is strictly educational and uses benign simulations in your own lab. NET 377 (Ethical Hacking) requires explicit written authorization for any testing beyond your own equipment.
> **Out of scope:** Real exploit frameworks (Metasploit against live targets), actual malware deployment, privilege escalation via 0-day. This project uses controlled, documented simulation commands that generate detectable telemetry without causing system harm.

## 1) Mission

- **Problem statement:** Detection rules built without adversary validation are unverified. A purple team exercise closes this loop: simulate attacker techniques, verify detections fire, tune rules that produce false positives or miss real activity, and document the validated detection state. This is the capstone project that integrates all previous work.
- **Why it matters:** Purple-team methodology is the gold standard for detection validation (NET 377, NET 179). It requires skills from every previous project: log triage (B4), persistence detection (I2), detection rules (A3), logging pipeline (A1), network segmentation (A2), and IR playbook (E1).

## 2) Difficulty

- Expert (estimated 20–40 focused hours)

## 3) Execution Context

- **VM environment** — All emulation activities run inside Proxmox lab VMs. The Arch bare-metal host is the "analyst" position that monitors detections from outside the emulated environment.

## 4) Prerequisites

- **Skills:** Completion of ALL previous projects (B1–B4, I1–I4, A1–A4, E1–E3). Strong understanding of MITRE ATT&CK framework. Bash and Python scripting.
- **Tools:** `logger`, `at`, `crontab`, `ssh`, `auditctl`, `journalctl`, `ausearch`, `tcpdump`, `nmap` (for lab-internal scanning only), `tee`
- **Dependencies:**
  - A1 centralized logging pipeline active.
  - A3 detection rules loaded (auditd).
  - A2 network segmentation in place.
  - E1 IR triage script available.
  - Two Proxmox VMs: one "target" (emulation runs here) and one "analyst" (detection monitoring here).

## 5) Rollback Plan

- **Target VM snapshot (pre):** `qm snapshot <TARGET_VMID> pre-e4-purple-team` — MANDATORY before each emulation phase.
- **Target VM snapshot rollback:** `qm rollback <TARGET_VMID> pre-e4-purple-team`
- **Analyst VM snapshot (pre):** `qm snapshot <ANALYST_VMID> pre-e4-analyst`
- **Host Snapper pre:** `sudo snapper -c root create --description "pre-e4-purple-team"`
- **Container rebuild command:** N/A
- **Rollback trigger:** After each emulation phase completes AND detections are validated, roll back the target VM to its clean state before the next phase. This ensures each phase starts from a known-clean baseline.

## 6) Project Plan

- **Phase A — Environment Preparation:** Confirm all prerequisite systems (A1, A3, E1) are operational. Take baseline snapshots.
- **Phase B — Emulation Scenarios (5 techniques):** Execute benign simulations of 5 ATT&CK techniques, capturing telemetry at each step.
- **Phase C — Detection Validation:** Verify each emulation produced a detectable event in the A3 rules and/or A1 log pipeline.
- **Phase D — Tune and Remediate:** Adjust detection rules based on findings. Document false positives and missed detections.
- **Phase E — Final Report:** Produce a purple-team exercise report with ATT&CK heat map and detection coverage summary.

## 7) Walkthrough

### Step 1 — Pre-exercise preparation

```bash
mkdir -p evidence/e4

# Record environment
date -u | tee evidence/e4/exercise_start_time.txt
qm list | tee evidence/e4/vm_inventory.txt

# Verify prerequisites
echo "=== Prerequisite Check ===" | tee evidence/e4/prereq_check.txt
systemctl is-active auditd && echo "PASS: auditd running" || echo "FAIL: auditd not running" | tee -a evidence/e4/prereq_check.txt
sudo auditctl -l | grep -q "a3-" && echo "PASS: A3 detection rules loaded" || echo "FAIL: A3 rules not loaded" | tee -a evidence/e4/prereq_check.txt
ls /var/log/journal/remote/remote-*.journal &>/dev/null && echo "PASS: A1 log pipeline active" || echo "WARN: A1 pipeline may not be active" | tee -a evidence/e4/prereq_check.txt
[ -x evidence/e1/ir-triage.sh ] && echo "PASS: E1 IR script available" || echo "FAIL: E1 IR script missing" | tee -a evidence/e4/prereq_check.txt
cat evidence/e4/prereq_check.txt

# Take mandatory VM snapshots
TARGET_VMID=101  # Replace with your target VM ID
ANALYST_VMID=102  # Replace with your analyst VM ID

qm snapshot $TARGET_VMID pre-e4-purple-team \
  --description "pre-e4-purple-team $(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  && echo "Target VM snapshot created" | tee evidence/e4/vm_snapshots.txt
sudo snapper -c root create --description "pre-e4-purple-team" --print-number \
  | tee evidence/e4/snapshot_pre_num.txt
```

### Step 2 — Emulation Phase 1: Scheduled Task Persistence (T1053.003)

```bash
TARGET_IP="10.10.20.10"  # Replace with your target VM IP

echo "=== Phase 1: T1053.003 Scheduled Task Persistence ===" | tee evidence/e4/emulation_log.txt
echo "Emulation started: $(date -u)" | tee -a evidence/e4/emulation_log.txt

# --- EMULATION: Add a benign crontab entry ---
# This simulates an attacker adding a persistence mechanism via cron
MARKER="E4-T1053-$(date -u +%s)"
ssh root@$TARGET_IP "(crontab -l 2>/dev/null; echo \"# $MARKER: @reboot echo 'benign persistence test'\") | crontab -"
sleep 5

# --- DETECTION: Query A3 rule ---
echo "--- Detection query ---" | tee -a evidence/e4/emulation_log.txt
sudo ausearch -k a3-cron-write --interpret 2>/dev/null | tail -10 | tee evidence/e4/phase1_detection.txt
journalctl --no-pager --since "2 minutes ago" | grep -i "cron" | tee -a evidence/e4/phase1_detection.txt

# --- RESULT ---
grep -q "a3-cron-write\|crontab" evidence/e4/phase1_detection.txt \
  && echo "DETECTED: T1053.003 cron persistence detected" | tee evidence/e4/phase1_result.txt \
  || echo "MISSED: T1053.003 cron persistence not detected — tune A3 rule" | tee evidence/e4/phase1_result.txt

# --- CLEANUP: Remove the test crontab entry ---
ssh root@$TARGET_IP "crontab -l | grep -v '$MARKER' | crontab -"

echo "Phase 1 complete. VM rollback recommended before Phase 2." | tee -a evidence/e4/emulation_log.txt

# Snapshot target VM after emulation (for forensic reference)
qm snapshot $TARGET_VMID post-e4-phase1 --description "post-e4-T1053-emulation"
```

### Step 3 — Emulation Phase 2: Systemd Service Persistence (T1543.002)

```bash
TARGET_IP="10.10.20.10"

echo "=== Phase 2: T1543.002 Systemd Service Persistence ===" | tee -a evidence/e4/emulation_log.txt

# --- EMULATION: Create a benign systemd service file ---
MARKER="e4-t1543-test-$(date -u +%s)"
ssh root@$TARGET_IP "cat > /etc/systemd/system/${MARKER}.service << 'EOF'
[Unit]
Description=E4 Emulation Test - T1543.002

[Service]
Type=oneshot
ExecStart=/bin/echo 'benign-emulation-test'
EOF
systemctl daemon-reload"
sleep 5

# --- DETECTION ---
echo "--- Detection query ---" | tee -a evidence/e4/emulation_log.txt
sudo ausearch -k a3-systemd-unit-write --interpret 2>/dev/null | tail -10 | tee evidence/e4/phase2_detection.txt
journalctl --since "2 minutes ago" --no-pager | grep -i "systemd.*${MARKER}\|daemon-reload" \
  | tee -a evidence/e4/phase2_detection.txt

# Remote: check from centralized log collector
VM_JOURNAL=$(ls /var/log/journal/remote/remote-*.journal 2>/dev/null | head -1)
[ -n "$VM_JOURNAL" ] && journalctl --file="$VM_JOURNAL" --since "2 minutes ago" --no-pager \
  | grep -i "$MARKER" | tee -a evidence/e4/phase2_detection.txt

grep -q "a3-systemd-unit-write\|$MARKER" evidence/e4/phase2_detection.txt \
  && echo "DETECTED: T1543.002 systemd persistence detected" | tee evidence/e4/phase2_result.txt \
  || echo "MISSED: T1543.002 systemd persistence not detected" | tee evidence/e4/phase2_result.txt

# --- CLEANUP ---
ssh root@$TARGET_IP "systemctl stop ${MARKER}.service 2>/dev/null; rm -f /etc/systemd/system/${MARKER}.service; systemctl daemon-reload"
```

### Step 4 — Emulation Phase 3: SSH Authorized Keys (T1098.004)

```bash
TARGET_IP="10.10.20.10"

echo "=== Phase 3: T1098.004 SSH Authorized Keys ===" | tee -a evidence/e4/emulation_log.txt

# --- EMULATION: Add a test key to authorized_keys (benign, harmless key format) ---
BOGUS_KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIE4-TEST-EMULATION-KEY-E4-T1098-DO-NOT-USE e4-emulation-test"
ssh root@$TARGET_IP "echo '$BOGUS_KEY' >> /root/.ssh/authorized_keys"
sleep 5

# --- DETECTION ---
echo "--- Detection query ---" | tee -a evidence/e4/emulation_log.txt
sudo ausearch -k a3-home-write --interpret 2>/dev/null | tail -10 | tee evidence/e4/phase3_detection.txt

# Sweep authorized_keys for the test marker
ssh root@$TARGET_IP "cat /root/.ssh/authorized_keys | grep e4-emulation-test" \
  | tee -a evidence/e4/phase3_detection.txt

grep -q "authorized_keys\|e4-emulation-test\|a3-home-write" evidence/e4/phase3_detection.txt \
  && echo "DETECTED: T1098.004 authorized_keys modification detected" | tee evidence/e4/phase3_result.txt \
  || echo "MISSED: T1098.004 authorized_keys modification not detected" | tee evidence/e4/phase3_result.txt

# --- CLEANUP ---
ssh root@$TARGET_IP "sed -i '/e4-emulation-test/d' /root/.ssh/authorized_keys"
```

### Step 5 — Emulation Phase 4: Brute Force Simulation (T1110.001)

```bash
TARGET_IP="10.10.20.10"

echo "=== Phase 4: T1110.001 SSH Brute Force Simulation ===" | tee -a evidence/e4/emulation_log.txt

# --- EMULATION: Generate multiple failed SSH auth attempts ---
for i in $(seq 1 8); do
  ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 \
    -o PasswordAuthentication=no \
    baduser_e4_bruteforce@$TARGET_IP 2>/dev/null || true
done
sleep 5

# --- DETECTION ---
echo "--- Detection query ---" | tee -a evidence/e4/emulation_log.txt
# Check analyst position (host) via centralized log
VM_JOURNAL=$(ls /var/log/journal/remote/remote-*.journal 2>/dev/null | head -1)
if [ -n "$VM_JOURNAL" ]; then
  journalctl --file="$VM_JOURNAL" --since "3 minutes ago" --no-pager \
    | grep -i "invalid user\|authentication failure\|Failed password" \
    | tee evidence/e4/phase4_detection.txt
fi

# Run the A3 brute force detection query
bash evidence/a3/detect01_query.sh 2>/dev/null | tee -a evidence/e4/phase4_detection.txt

FAIL_COUNT=$(grep -c "invalid user\|authentication failure\|Failed password" evidence/e4/phase4_detection.txt 2>/dev/null || echo 0)
[ "$FAIL_COUNT" -ge 3 ] \
  && echo "DETECTED: T1110.001 brute force detected ($FAIL_COUNT events)" | tee evidence/e4/phase4_result.txt \
  || echo "MISSED or BELOW THRESHOLD: $FAIL_COUNT events detected" | tee evidence/e4/phase4_result.txt
```

### Step 6 — Emulation Phase 5: Network Discovery (T1046)

```bash
TARGET_IP="10.10.20.10"
SCAN_TARGET="10.10.20.0/24"

echo "=== Phase 5: T1046 Network Service Discovery ===" | tee -a evidence/e4/emulation_log.txt
echo "NOTE: Scanning only the isolated lab segment (vmbr2 from A2)" | tee -a evidence/e4/emulation_log.txt

# --- EMULATION: Run nmap against the isolated lab segment ONLY ---
# This is legitimate on your own lab network — never scan external networks
ssh root@$TARGET_IP "which nmap && nmap -sn $SCAN_TARGET 2>&1 | head -30" \
  | tee evidence/e4/phase5_nmap_output.txt \
  || echo "nmap not available on target — install with: apt install nmap / pacman -S nmap" \
  | tee evidence/e4/phase5_nmap_output.txt

# --- DETECTION ---
# Capture network flows during scan
sudo timeout 30 tcpdump -i vmbr2 -n -c 50 2>/dev/null | tee evidence/e4/phase5_tcpdump.txt &
sleep 20 && wait

# Check for scan signature in logs
journalctl -b --since "5 minutes ago" --no-pager | grep -i "nmap\|scan\|probe" \
  | tee evidence/e4/phase5_detection.txt

[ -s evidence/e4/phase5_tcpdump.txt ] \
  && echo "CAPTURED: Network discovery traffic captured in tcpdump" | tee evidence/e4/phase5_result.txt \
  || echo "MISSED: Network discovery not captured" | tee evidence/e4/phase5_result.txt
```

### Step 7 — Generate purple-team exercise report

```bash
# Run E1 IR triage on the target VM as a post-exercise capture
ssh root@$TARGET_IP "bash /tmp/ir-triage.sh /tmp/e4-post-exercise-triage" 2>/dev/null \
  | tee evidence/e4/ir_triage_run.txt || true

# Generate final report
cat | tee evidence/e4/purple_team_report.md << 'EOF'
# E4 — Purple-Team Validation Cycle: Exercise Report

## Exercise Metadata
- Lab: Arch/Proxmox Homelab
- Framework: MITRE ATT&CK Enterprise
- Scope: Isolated lab segment (vmbr2 / 10.10.20.0/24)
- Duration: [fill in actual duration]

## Detection Coverage Summary

| Phase | Technique | ID | Result | Notes |
|---|---|---|---|---|
| 1 | Scheduled Task Persistence | T1053.003 | See phase1_result.txt | Cron write audit rule |
| 2 | Systemd Service | T1543.002 | See phase2_result.txt | systemd unit write audit rule |
| 3 | SSH Authorized Keys | T1098.004 | See phase3_result.txt | home dir write audit rule |
| 4 | SSH Brute Force | T1110.001 | See phase4_result.txt | journald SSH query |
| 5 | Network Discovery | T1046 | See phase5_result.txt | tcpdump capture |

## Findings

### Detections Confirmed
[Fill in from phase results — techniques where detection fired]

### Detection Gaps
[Fill in from phase results — techniques that were missed or had low confidence]

### False Positive Risk
[Document any rules that might fire on legitimate activity]

## Tuning Actions Taken
[Document any A3 rule changes made based on exercise findings]

## ATT&CK Techniques Exercised
- T1053.003 (Persistence)
- T1543.002 (Persistence)  
- T1098.004 (Persistence)
- T1110.001 (Credential Access)
- T1046 (Discovery)

## References
- MITRE ATT&CK: https://attack.mitre.org/
- Purple Team Framework: https://www.attackiq.com/lp/purple-team-framework/
EOF

cat evidence/e4/purple_team_report.md

# Final evidence summary
date -u | tee evidence/e4/exercise_end_time.txt
echo "Total evidence files: $(find evidence/e4 -type f | wc -l)" | tee evidence/e4/exercise_summary.txt
```

## 8) Validation

```bash
# Check all 5 phases completed
for phase in 1 2 3 4 5; do
  [ -f "evidence/e4/phase${phase}_result.txt" ] \
    && echo "COMPLETE: Phase $phase" \
    || echo "INCOMPLETE: Phase $phase result missing"
done

# Count detections vs misses
DETECTED=$(grep -l "DETECTED" evidence/e4/phase*.result.txt 2>/dev/null | wc -l)
MISSED=$(grep -l "MISSED" evidence/e4/phase*.result.txt 2>/dev/null | wc -l)
echo "Detection results: $DETECTED DETECTED, $MISSED MISSED" | tee evidence/e4/detection_summary.txt

# Confirm report exists
[ -s evidence/e4/purple_team_report.md ] \
  && echo "PASS: Purple team report written" \
  || echo "FAIL: Report missing"
```

## 9) Evidence

- **Output files:**
  - `evidence/e4/exercise_start_time.txt`, `exercise_end_time.txt` — timing
  - `evidence/e4/vm_inventory.txt` — VM state at exercise start
  - `evidence/e4/prereq_check.txt` — prerequisite validation
  - `evidence/e4/vm_snapshots.txt`, `snapshot_pre_num.txt` — rollback checkpoints
  - `evidence/e4/emulation_log.txt` — all emulation step log
  - `evidence/e4/phase1_detection.txt`, `phase1_result.txt` — T1053.003 evidence
  - `evidence/e4/phase2_detection.txt`, `phase2_result.txt` — T1543.002 evidence
  - `evidence/e4/phase3_detection.txt`, `phase3_result.txt` — T1098.004 evidence
  - `evidence/e4/phase4_detection.txt`, `phase4_result.txt` — T1110.001 evidence
  - `evidence/e4/phase5_nmap_output.txt`, `phase5_tcpdump.txt`, `phase5_result.txt` — T1046 evidence
  - `evidence/e4/ir_triage_run.txt` — post-exercise IR triage
  - `evidence/e4/purple_team_report.md` — final exercise report
  - `evidence/e4/detection_summary.txt` — detection count summary
- **Hash manifest:**

```bash
find evidence/e4 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/e4/SHA256SUMS.txt
cat evidence/e4/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** Phase 1 or 2 emulation commands fail because the target VM is not reachable.
  - **Cause:** VM is not running, SSH key not deployed to target VM, or lab network routing issue from A2.
  - **Fix:** Verify VM is running (`qm status <VMID>`). Test connectivity: `ping 10.10.20.10`. Check A2 nftables rules allow SSH from the management segment.
  - **Rollback trigger:** `qm rollback <TARGET_VMID> pre-e4-purple-team` if VM state is inconsistent.

- **Symptom:** Detection query finds no events after emulation.
  - **Cause:** Audit rule key may be different from what's loaded, or auditd buffer dropped events.
  - **Fix:** `sudo auditctl -l` to confirm rule keys. Increase audit buffer: `sudo auditctl -b 16384`. Re-run emulation.
  - **Rollback trigger:** N/A — emulation was cleanup'd. Roll back VM if residue remains.

- **Symptom:** nmap scan in Phase 5 produces no output on the analyst position.
  - **Cause:** vmbr2 does not route traffic through the analyst position, or nmap is not installed on target.
  - **Fix:** Run tcpdump on the Proxmox host's vmbr2 interface directly: `sudo tcpdump -i vmbr2 -n`. Alternatively install nmap on the analyst VM and scan from there.
  - **Rollback trigger:** N/A.

## 11) Sources

- [MITRE ATT&CK — Enterprise Matrix](https://attack.mitre.org/matrices/enterprise/)
- [MITRE ATT&CK — T1053.003 Cron](https://attack.mitre.org/techniques/T1053/003/)
- [MITRE ATT&CK — T1543.002 Systemd Service](https://attack.mitre.org/techniques/T1543/002/)
- [MITRE ATT&CK — T1098.004 SSH Authorized Keys](https://attack.mitre.org/techniques/T1098/004/)
- [MITRE ATT&CK — T1110.001 Password Guessing](https://attack.mitre.org/techniques/T1110/001/)
- [CISA — Purple Team Best Practices](https://www.cisa.gov/sites/default/files/publications/Purple_Team_Guidance.pdf)
- [NIST SP 800-61r2 — Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)

## 12) Stretch Goals

- Extend the purple-team to cover T1059.004 (Unix Shell execution — script execution from suspicious paths) by placing a test script in `/tmp`, executing it, and verifying auditd captures the `execve` call.
- Build an ATT&CK Navigator heat map layer file (JSON) showing which techniques are covered by your current detection ruleset. Upload to the ATT&CK Navigator at https://mitre-attack.github.io/attack-navigator/.
- Automate the entire 5-phase exercise as a single Bash script (`e4-purple-cycle.sh`) that runs all emulation steps, collects evidence, generates the report, and rolls back the target VM automatically.
- Run the same emulation scenarios against a different OS guest (Debian, Ubuntu, or Alpine in Proxmox) and compare whether the same auditd rules detect the same techniques across different distributions.
