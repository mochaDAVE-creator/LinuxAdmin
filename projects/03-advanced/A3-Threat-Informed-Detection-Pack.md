---
title: A3 - Threat-Informed Detection Pack
aliases:
  - a3-detection-pack
  - threat-detection-advanced
tags:
  - advanced
  - detection
  - mitre-attack
  - journald
  - forensics
  - evidence
  - net179
  - net377
date: 2026-05-10
---

# A3 — Threat-Informed Detection Pack

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host and Proxmox lab VMs. Detection rules are built to alert on attacker behaviors in a lab environment using available tooling (journald queries, auditd rules, simple shell watchers). No offensive tools are deployed against live systems.
> **Risk level:** High — auditd rules and filesystem watchers generate significant log volume and CPU overhead on busy systems. Test all rules in a VM before applying to the bare-metal host.
> **Threat model:** Detections are mapped to MITRE ATT&CK techniques: T1059 (Command and Scripting Interpreter), T1053 (Scheduled Task/Job), T1547 (Boot/Logon Autostart), T1078 (Valid Accounts), T1110 (Brute Force). Each detection rule is paired with a safe lab simulation to prove it fires correctly.
> **Out of scope:** Host-based IDS platforms (Wazuh, OSSEC, Falco). This project uses native Linux tooling only. Falco and Wazuh are covered in stretch goals.
> **Important:** Simulations in this project generate benign signals only (e.g., failed logins, harmless cron entries). No real attack payloads, exploit code, or actual malicious activity is used or simulated.

## 1) Mission

- **Problem statement:** Building detections without testing them is theater. This project follows a Detect→Simulate→Verify loop: write a detection rule, generate a safe benign signal that mimics attacker behavior, and confirm the detection fires. The result is a small but validated detection pack.
- **Why it matters:** Detection engineering is a core NET 179 and NET 377 skill. The detection rules built here feed directly into E4 (Purple-Team Validation Cycle) and complement the centralized logging pipeline built in A1.

## 2) Difficulty

- Advanced (estimated 10–16 focused hours)

## 3) Execution Context

- **Host and VM** — Detection rules are deployed on the Arch host. Simulations can be run on a test VM (isolated segment from A2) to avoid generating false positives on the host.

## 4) Prerequisites

- **Skills:** Completion of B4 (Log Triage Starter), I2 (Persistence Detection Sweep), A1 (Centralized Logging Pipeline). Understanding of auditd rules and journalctl filters.
- **Tools:** `auditd`, `auditctl`, `ausearch`, `aureport`, `journalctl`, `systemd-analyze`, `logger`, `crontab`, `tee`
- **Dependencies:**
  - `audit` package: `pacman -Q audit`
  - `auditd` service enabled: `systemctl status auditd`
  - A1 logging pipeline active (detections should also appear in the centralized log).

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-a3-detection-pack"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-a3-detection-pack"`
- **VM snapshot (pre):** `qm snapshot <VMID> pre-a3-detection`
- **Container rebuild command:** N/A
- **Rollback trigger:** If auditd rules cause performance issues or log flooding: `sudo auditctl -D` clears all rules immediately. Snapper rollback removes the auditd config file changes.

## 6) Project Plan

- **Phase A — Auditd Setup:** Install and configure auditd with a baseline set of rules.
- **Phase B — Detection Rules:** Write rules for five ATT&CK-mapped behaviors.
- **Phase C — Simulations:** Safely simulate each attacker behavior and verify the detection fires.
- **Phase D — Reporting:** Build an `aureport` and `journalctl` summary for each detection event.

## 7) Walkthrough

### Step 1 — Install and configure auditd

```bash
mkdir -p evidence/a3

# Install audit package
sudo pacman -S --noconfirm audit 2>/dev/null || pacman -Q audit | tee evidence/a3/audit_pkg_version.txt

# Enable and start auditd
sudo systemctl enable --now auditd.service
systemctl status auditd --no-pager | tee evidence/a3/auditd_status.txt

# Capture pre-snapshot
sudo snapper -c root create --description "pre-a3-detection-pack" --print-number \
  | tee evidence/a3/snapshot_pre_num.txt

# Show current auditd rules (baseline)
sudo auditctl -l | tee evidence/a3/auditd_rules_before.txt

# Show audit configuration
cat /etc/audit/auditd.conf | grep -Ev "^#|^$" | tee evidence/a3/auditd_config.txt
```

Expected:
- `auditd_status.txt` shows `active (running)`.
- `auditd_rules_before.txt` shows existing rules (may be empty on fresh install).

### Step 2 — Write Detection Rule 1: Brute Force SSH (T1110.001)

```bash
# Detection: Multiple failed SSH auth attempts in a short time window
# ATT&CK: T1110.001 — Password Guessing

# Auditd rule: watch auth log for login failures
# (On systems using sshd+journald, we use journalctl query instead)
cat | sudo tee /etc/audit/rules.d/10-a3-ssh-brute.rules << 'EOF'
# A3-DETECT-01: Brute Force SSH — T1110.001
# Watch for execve of su/ssh commands (supplemental — main detection via journalctl)
-w /usr/bin/ssh -p x -k a3-ssh-exec
-w /usr/bin/su -p x -k a3-su-exec
EOF

# Journalctl query for the detection
cat | tee evidence/a3/detect01_query.sh << 'EOF'
#!/bin/bash
# A3-DETECT-01: SSH Brute Force detection query
WINDOW="30 minutes ago"
THRESHOLD=3
COUNT=$(journalctl _SYSTEMD_UNIT=sshd.service --since "$WINDOW" --no-pager \
  | grep -cE "authentication failure|Failed password|Invalid user")
echo "Failed SSH auth count in last 30 min: $COUNT"
[ "$COUNT" -ge "$THRESHOLD" ] \
  && echo "ALERT: Possible brute force ($COUNT failures >= threshold $THRESHOLD)" \
  || echo "INFO: Below threshold"
journalctl _SYSTEMD_UNIT=sshd.service --since "$WINDOW" --no-pager \
  | grep -E "authentication failure|Failed password|Invalid user"
EOF
chmod +x evidence/a3/detect01_query.sh
```

Simulation and verification:

```bash
# Simulation: generate 5 failed SSH logins (safe — using localhost and bad credentials)
for i in $(seq 1 5); do
  ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 \
    -o PasswordAuthentication=no \
    baduser_a3test@127.0.0.1 2>/dev/null || true
done
sleep 5

# Run detection query
bash evidence/a3/detect01_query.sh | tee evidence/a3/detect01_result.txt
journalctl _SYSTEMD_UNIT=sshd.service --no-pager -n 20 | grep -i "invalid user" \
  | tee evidence/a3/detect01_events.txt
```

Expected:
- `detect01_result.txt` shows `ALERT: Possible brute force` with count ≥ 3.

### Step 3 — Write Detection Rule 2: New SUID Binary (T1548.001)

```bash
# Detection: New file with SUID bit set outside expected locations
# ATT&CK: T1548.001 — Setuid and Setgid

# Auditd rule: alert on chmod/chown setting SUID
cat | sudo tee /etc/audit/rules.d/20-a3-suid.rules << 'EOF'
# A3-DETECT-02: New SUID binary — T1548.001
-a always,exit -F arch=b64 -S chmod -F a1=0x800 -k a3-suid-set
-a always,exit -F arch=b64 -S fchmod -F a1=0x800 -k a3-suid-set
-a always,exit -F arch=b32 -S chmod -F a1=0x800 -k a3-suid-set
EOF

sudo auditctl -R /etc/audit/rules.d/20-a3-suid.rules 2>&1

# Detection query script
cat | tee evidence/a3/detect02_query.sh << 'EOF'
#!/bin/bash
# A3-DETECT-02: SUID bit set detection query
echo "=== Recent SUID-set audit events ==="
sudo ausearch -k a3-suid-set --interpret 2>/dev/null | tail -30
echo "=== Current SUID binary count ==="
sudo find / -xdev -perm -4000 -type f 2>/dev/null | wc -l
EOF
chmod +x evidence/a3/detect02_query.sh

# Simulation: create a temp binary and set SUID on it
cp /bin/echo /tmp/a3-suid-test-binary
sudo chmod u+s /tmp/a3-suid-test-binary
sleep 2

# Run detection query
bash evidence/a3/detect02_query.sh | tee evidence/a3/detect02_result.txt

# Cleanup simulation artifact
sudo chmod u-s /tmp/a3-suid-test-binary
rm -f /tmp/a3-suid-test-binary
```

### Step 4 — Write Detection Rule 3: New Systemd Unit File (T1543.002)

```bash
# Detection: New file written to systemd unit directories
# ATT&CK: T1543.002 — Systemd Service persistence

cat | sudo tee /etc/audit/rules.d/30-a3-systemd-persist.rules << 'EOF'
# A3-DETECT-03: New systemd unit file — T1543.002
-w /etc/systemd/system -p w -k a3-systemd-unit-write
-w /usr/lib/systemd/system -p w -k a3-systemd-unit-write
-w /home -p w -k a3-home-write
EOF

sudo auditctl -R /etc/audit/rules.d/30-a3-systemd-persist.rules 2>&1

# Detection query
cat | tee evidence/a3/detect03_query.sh << 'EOF'
#!/bin/bash
# A3-DETECT-03: New systemd unit detection query
echo "=== Recent systemd unit directory writes ==="
sudo ausearch -k a3-systemd-unit-write --interpret 2>/dev/null | tail -30
echo "=== Unpackaged systemd unit files ==="
find /etc/systemd/system -maxdepth 2 -type f -name "*.service" 2>/dev/null | while read f; do
  pacman -Qo "$f" 2>/dev/null || echo "UNPACKAGED: $f"
done
EOF
chmod +x evidence/a3/detect03_query.sh

# Simulation: create a temporary fake unit file
echo "[Unit]" | sudo tee /etc/systemd/system/a3-test-detect.service > /dev/null
sleep 2

# Run detection
bash evidence/a3/detect03_query.sh | tee evidence/a3/detect03_result.txt

# Cleanup
sudo rm -f /etc/systemd/system/a3-test-detect.service
sudo systemctl daemon-reload
```

### Step 5 — Write Detection Rule 4: Unauthorized sudo/su Usage (T1078)

```bash
# Detection: Unexpected user executing sudo or su
# ATT&CK: T1078 — Valid Accounts

# Auditd rule: watch sudo binary execution
cat | sudo tee /etc/audit/rules.d/40-a3-sudo-use.rules << 'EOF'
# A3-DETECT-04: Unexpected sudo/su execution — T1078
-w /usr/bin/sudo -p x -k a3-sudo-exec
-w /usr/bin/su -p x -k a3-su-exec
-w /etc/sudoers -p rwa -k a3-sudoers-mod
-w /etc/sudoers.d -p rwa -k a3-sudoers-mod
EOF

sudo auditctl -R /etc/audit/rules.d/40-a3-sudo-use.rules 2>&1

# Detection query
cat | tee evidence/a3/detect04_query.sh << 'EOF'
#!/bin/bash
# A3-DETECT-04: sudo/su usage detection
echo "=== Recent sudo executions ==="
sudo ausearch -k a3-sudo-exec --interpret 2>/dev/null | tail -20
echo "=== Recent sudoers modifications ==="
sudo ausearch -k a3-sudoers-mod --interpret 2>/dev/null | tail -20
echo "=== journalctl sudo events ==="
journalctl --no-pager --since "1 hour ago" \
  | grep -i "sudo\|su\[" | head -20
EOF
chmod +x evidence/a3/detect04_query.sh

# Simulation: run a benign sudo command (already legitimately needed)
sudo id > /dev/null 2>&1
sleep 2

# Run detection
bash evidence/a3/detect04_query.sh | tee evidence/a3/detect04_result.txt
```

### Step 6 — Write Detection Rule 5: Cron Persistence Attempt (T1053.003)

```bash
# Detection: New crontab entry created
# ATT&CK: T1053.003 — Cron

cat | sudo tee /etc/audit/rules.d/50-a3-cron-persist.rules << 'EOF'
# A3-DETECT-05: Cron persistence — T1053.003
-w /var/spool/cron -p wa -k a3-cron-write
-w /etc/cron.d -p wa -k a3-cron-write
-w /etc/crontab -p wa -k a3-cron-write
EOF

sudo auditctl -R /etc/audit/rules.d/50-a3-cron-persist.rules 2>&1

# Detection query
cat | tee evidence/a3/detect05_query.sh << 'EOF'
#!/bin/bash
# A3-DETECT-05: Cron persistence detection
echo "=== Recent cron directory writes ==="
sudo ausearch -k a3-cron-write --interpret 2>/dev/null | tail -20
echo "=== Current user crontabs ==="
for user in $(getent passwd | awk -F: '$7~/bash|zsh/ {print $1}'); do
  ct=$(sudo crontab -l -u "$user" 2>/dev/null)
  [ -n "$ct" ] && echo "User $user has crontab:" && echo "$ct"
done
EOF
chmod +x evidence/a3/detect05_query.sh

# Simulation: add a benign cron entry then immediately remove it
(crontab -l 2>/dev/null; echo "# A3-TEST-$(date -u +%s) harmless test entry") | crontab -
sleep 2
# Remove the test entry
crontab -l | grep -v "A3-TEST" | crontab -

# Run detection
bash evidence/a3/detect05_query.sh | tee evidence/a3/detect05_result.txt
```

### Step 7 — Load all rules and generate aureport summary

```bash
# Reload all audit rules
sudo auditctl -D  # Clear temporary rules
sudo augenrules --load 2>&1 | tee evidence/a3/augenrules_output.txt

# Show all loaded rules
sudo auditctl -l | tee evidence/a3/auditd_rules_final.txt

# Generate comprehensive audit report
sudo aureport --summary | tee evidence/a3/aureport_summary.txt
sudo aureport --auth --summary | tee evidence/a3/aureport_auth.txt
sudo aureport --key --summary | tee evidence/a3/aureport_keys.txt
```

## 8) Validation

```bash
# Verify auditd is running
systemctl is-active auditd \
  && echo "PASS: auditd active" \
  || echo "FAIL: auditd not running"

# Verify rules are loaded
RULE_COUNT=$(sudo auditctl -l | wc -l)
echo "Loaded audit rules: $RULE_COUNT"
[ "$RULE_COUNT" -ge 5 ] \
  && echo "PASS: Rules loaded" \
  || echo "FAIL: Fewer rules than expected"

# Confirm each detection result file exists and shows ALERT or events
for detect in 01 02 03 04 05; do
  [ -s "evidence/a3/detect${detect}_result.txt" ] \
    && echo "PASS: Detection $detect result captured" \
    || echo "WARN: Detection $detect result missing or empty"
done

# Final detection mapping to ATT&CK
echo "=== Detection Pack ATT&CK Mapping ===" | tee evidence/a3/attack_mapping.txt
echo "DETECT-01: T1110.001 - Brute Force SSH" | tee -a evidence/a3/attack_mapping.txt
echo "DETECT-02: T1548.001 - SUID bit set on file" | tee -a evidence/a3/attack_mapping.txt
echo "DETECT-03: T1543.002 - New systemd unit file" | tee -a evidence/a3/attack_mapping.txt
echo "DETECT-04: T1078     - Unexpected sudo/su usage" | tee -a evidence/a3/attack_mapping.txt
echo "DETECT-05: T1053.003 - Cron persistence write" | tee -a evidence/a3/attack_mapping.txt
cat evidence/a3/attack_mapping.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/a3/audit_pkg_version.txt` — audit package version
  - `evidence/a3/auditd_status.txt` — auditd service status
  - `evidence/a3/snapshot_pre_num.txt` — Snapper pre-snapshot
  - `evidence/a3/auditd_rules_before.txt` — rules before project
  - `evidence/a3/auditd_config.txt` — auditd configuration
  - `evidence/a3/detect01_query.sh` — detection 1 query script
  - `evidence/a3/detect01_result.txt` — detection 1 result (SSH brute force)
  - `evidence/a3/detect01_events.txt` — detection 1 event log entries
  - `evidence/a3/detect02_query.sh` — detection 2 query script
  - `evidence/a3/detect02_result.txt` — detection 2 result (SUID)
  - `evidence/a3/detect03_query.sh` — detection 3 query script
  - `evidence/a3/detect03_result.txt` — detection 3 result (systemd unit)
  - `evidence/a3/detect04_query.sh` — detection 4 query script
  - `evidence/a3/detect04_result.txt` — detection 4 result (sudo/su)
  - `evidence/a3/detect05_query.sh` — detection 5 query script
  - `evidence/a3/detect05_result.txt` — detection 5 result (cron)
  - `evidence/a3/augenrules_output.txt` — rule load output
  - `evidence/a3/auditd_rules_final.txt` — final loaded rules
  - `evidence/a3/aureport_summary.txt` — audit summary report
  - `evidence/a3/aureport_auth.txt` — authentication report
  - `evidence/a3/aureport_keys.txt` — audit key report
  - `evidence/a3/attack_mapping.txt` — ATT&CK technique mapping
- **Hash manifest:**

```bash
find evidence/a3 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/a3/SHA256SUMS.txt
cat evidence/a3/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `auditd` fails to start with "audit socket already in use."
  - **Cause:** `kauditd` is running with `auditd` not cleanly stopped.
  - **Fix:** `sudo systemctl stop auditd && sudo auditd -f` for debugging. Check `dmesg | grep audit`.
  - **Rollback trigger:** Snapper rollback removes auditd rule files.

- **Symptom:** `ausearch -k a3-suid-set` returns no results after simulation.
  - **Cause:** The rule key was not loaded, or the auditd buffer was full.
  - **Fix:** `sudo auditctl -l | grep a3-suid-set` to confirm rule is loaded. Increase buffer size: `sudo auditctl -b 8192`.
  - **Rollback trigger:** N/A.

- **Symptom:** auditd generates so many events that disk fills rapidly.
  - **Cause:** A noisy rule (e.g., watching `/tmp` for writes) generates events for every tmpfs operation.
  - **Fix:** `sudo auditctl -D` clears all rules. Remove the offending rule file from `/etc/audit/rules.d/`.
  - **Rollback trigger:** Snapper if rule files were committed to `/etc/audit/`.

## 11) Sources

- [MITRE ATT&CK — T1110 Brute Force](https://attack.mitre.org/techniques/T1110/)
- [MITRE ATT&CK — T1543.002 Systemd Service](https://attack.mitre.org/techniques/T1543/002/)
- [MITRE ATT&CK — T1053.003 Cron](https://attack.mitre.org/techniques/T1053/003/)
- [MITRE ATT&CK — T1059 Command and Scripting Interpreter](https://attack.mitre.org/techniques/T1059/)
- [Linux Audit System — auditctl man page](https://man7.org/linux/man-pages/man8/auditctl.8.html)
- [CISA — Logging Made Easy](https://www.cisa.gov/sites/default/files/2023-03/CISA_Logging_Made_Easy.pdf)
- [NIST SP 800-61r2 — Computer Security Incident Handling](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)

## 12) Stretch Goals

- Port the five detection rules to Falco (container-native security monitoring) using Falco's YAML rule syntax. Compare Falco's rule expressiveness to auditd.
- Add a sixth detection rule for T1021.004 (Remote Services: SSH) — detect when SSH forwarding is used to tunnel traffic through the host.
- Build a daily cron job that runs all five detection queries and emails or writes a markdown report summarizing the previous 24 hours of detection events.
- Cross-reference all detection events with the A1 centralized log pipeline to confirm that alerts generated on the host appear in the remote journal on the collector within the expected latency window.
