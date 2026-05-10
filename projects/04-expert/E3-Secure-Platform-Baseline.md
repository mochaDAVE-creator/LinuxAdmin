---
title: E3 - Secure Platform Baseline
aliases:
  - e3-platform-baseline
  - cis-hardening-expert
tags:
  - expert
  - hardening
  - cis-benchmark
  - baseline
  - compliance
  - evidence
  - net412
  - net210
date: 2026-05-10
---

# E3 — Secure Platform Baseline

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host. This project implements a systematic hardening baseline derived from CIS Benchmark and DISA STIG guidance, adapted for the Arch Linux and Proxmox lab context. Changes are applied incrementally with validation at each step.
> **Risk level:** High — system-level hardening changes include kernel parameter tuning, SSH daemon configuration, filesystem permissions, and PAM settings. Each change can break normal operation if applied incorrectly. Every step must be preceded by a Snapper snapshot.
> **Threat model:** A Linux system with default configuration exposes a large attack surface: kernel debugging interfaces, IPv4 forwarding, world-readable configuration files, default password complexity settings, and more. This project systematically closes these default-open exposures to match a defined security baseline.
> **Not a compliance audit:** This is a learning exercise implementing selected controls from CIS and STIG guidance. It is NOT a full CIS compliance scan and does NOT produce a compliance certificate. It teaches the methodology of structured hardening and evidence documentation.
> **Reversibility:** Every control applied should be documented with a corresponding `snapper undochange` command. Do not apply controls you cannot reverse in the lab context.

## 1) Mission

- **Problem statement:** Most Linux systems as shipped have reasonable defaults for general use, but not for security-sensitive deployments. A secure platform baseline is a defined set of configurations that reduce the attack surface to a documented, auditable state. This project builds and validates a lab baseline systematically.
- **Why it matters:** Baseline hardening is a NET 412 and NET 210 senior competency. CIS Benchmark and DISA STIG are industry-standard frameworks. Understanding how they work — and why each control exists — is the foundation for operating in enterprise environments.

## 2) Difficulty

- Expert (estimated 20–40 focused hours to implement and document completely)

## 3) Execution Context

- **Host** — Arch Linux bare metal. Apply controls to the host system. Use a Proxmox VM for testing control impact before applying to the bare-metal host.

## 4) Prerequisites

- **Skills:** Completion of I1 (Systemd Service Hardening), I3 (ACL-Backed Secret Store), B3 (SSH Key-Only Admin Access). Deep understanding of Linux kernel parameters and PAM.
- **Tools:** `sysctl`, `sysctl.d`, `pam`, `sshd`, `chmod`, `chown`, `auditd`, `openscap` (if available), `lynis`, `tee`
- **Dependencies:**
  - `lynis` security auditing tool: `pacman -S lynis` (or from AUR).
  - `audit` package installed (from A3): `pacman -Q audit`.

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-e3-platform-baseline"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-e3-platform-baseline"`
- **VM snapshot:** Test each control in a Proxmox VM before applying to bare metal: `qm snapshot <VMID> pre-e3-test`.
- **Container rebuild command:** N/A
- **Rollback trigger:** For any control that breaks normal operation: `sudo snapper -c root undochange <PRE>..<POST>`. For sysctl changes specifically: `sudo sysctl --system` re-applies `/etc/sysctl.d/` files. Remove the specific `.conf` file to revert a sysctl change without full snapshot rollback.

## 6) Project Plan

- **Phase A — Pre-Hardening Audit:** Run `lynis` to establish a baseline score before any changes.
- **Phase B — Kernel Hardening:** Apply sysctl parameters for network, memory, and kernel security.
- **Phase C — Service Hardening:** Disable unnecessary services and apply SSH hardening (extends B3).
- **Phase D — Filesystem and Account Hardening:** Apply file permission controls, PAM password policy, and account security settings.
- **Phase E — Post-Hardening Audit:** Re-run `lynis` and compare score improvement.

## 7) Walkthrough

### Step 1 — Pre-hardening lynis audit

```bash
mkdir -p evidence/e3

# Snapper pre-snapshot
sudo snapper -c root create --description "pre-e3-platform-baseline" --print-number \
  | tee evidence/e3/snapshot_pre_num.txt

# Install lynis
sudo pacman -S --noconfirm lynis 2>/dev/null || pacman -Q lynis | tee evidence/e3/lynis_version.txt

# Run pre-hardening audit
sudo lynis audit system --no-colors --quiet 2>&1 | tee evidence/e3/lynis_audit_before.txt

# Extract hardening index score
grep "Hardening index" evidence/e3/lynis_audit_before.txt | tee evidence/e3/lynis_score_before.txt

# Extract all warnings and suggestions
grep -E "^\[WARNING\]|\[SUGGESTION\]" evidence/e3/lynis_audit_before.txt \
  | tee evidence/e3/lynis_warnings_before.txt \
  || grep -E "^  !" evidence/e3/lynis_audit_before.txt | head -30 | tee evidence/e3/lynis_warnings_before.txt
```

Expected:
- `lynis_score_before.txt` shows a score (typically 40–65 on a standard Arch install).
- `lynis_warnings_before.txt` lists controls that need attention.

### Step 2 — Phase B: Kernel hardening via sysctl

```bash
cat | sudo tee /etc/sysctl.d/90-e3-security-baseline.conf << 'EOF'
# E3 Secure Platform Baseline — Kernel Parameters
# CIS Benchmark sections: 3.x Network Configuration, 1.x OS Configuration

# ===== Network: Disable IPv4 forwarding (unless this host is a router) =====
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# ===== Network: Ignore ICMP redirects (prevent routing table poisoning) =====
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# ===== Network: Ignore source-routed packets =====
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# ===== Network: Enable TCP SYN cookie protection =====
net.ipv4.tcp_syncookies = 1

# ===== Network: Ignore broadcast ICMP (prevent smurf attacks) =====
net.ipv4.icmp_echo_ignore_broadcasts = 1

# ===== Network: Ignore bogus ICMP error responses =====
net.ipv4.icmp_ignore_bogus_error_responses = 1

# ===== Network: Enable reverse path filtering (prevent IP spoofing) =====
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# ===== Network: Log martian packets =====
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# ===== Kernel: Restrict dmesg to root =====
kernel.dmesg_restrict = 1

# ===== Kernel: Restrict kernel pointer exposure =====
kernel.kptr_restrict = 2

# ===== Kernel: Disable magic sysrq (reduces risk of local console attacks) =====
kernel.sysrq = 0

# ===== Kernel: Disable core dumps for SUID programs =====
fs.suid_dumpable = 0

# ===== Kernel: Enable ASLR (Address Space Layout Randomization) =====
kernel.randomize_va_space = 2

# ===== Kernel: Restrict ptrace to root =====
kernel.yama.ptrace_scope = 1
EOF

# Apply sysctl settings
sudo sysctl --system 2>&1 | tee evidence/e3/sysctl_apply_output.txt

# Verify specific controls
echo "=== Kernel parameter verification ===" | tee evidence/e3/sysctl_verify.txt
for param in \
  net.ipv4.ip_forward \
  net.ipv4.conf.all.accept_redirects \
  net.ipv4.tcp_syncookies \
  kernel.dmesg_restrict \
  kernel.kptr_restrict \
  kernel.randomize_va_space \
  kernel.yama.ptrace_scope; do
  val=$(sysctl -n "$param" 2>/dev/null)
  echo "$param = $val" | tee -a evidence/e3/sysctl_verify.txt
done
```

Expected:
- `sysctl_apply_output.txt` shows settings applied without errors.
- `sysctl_verify.txt` shows each parameter at its target value.

### Step 3 — Phase C: Service hardening — disable unnecessary services

```bash
# List all enabled services before changes
systemctl list-unit-files --state=enabled --no-pager | tee evidence/e3/services_enabled_before.txt

# Disable services not needed in this lab (adjust based on actual use)
DISABLE_SERVICES=(
  "avahi-daemon"        # mDNS — not needed unless zero-conf networking is required
  "cups"                # Printing daemon — disable if no printer attached
  "bluetooth"           # Bluetooth — disable if no BT hardware
  "rpcbind"             # NFS portmapper — disable if no NFS
  "nfs-server"          # NFS server — disable if not serving NFS
)

for svc in "${DISABLE_SERVICES[@]}"; do
  if systemctl is-enabled "$svc" &>/dev/null; then
    sudo systemctl disable --now "$svc" 2>&1 \
      && echo "Disabled: $svc" | tee -a evidence/e3/services_disabled.txt
  else
    echo "Not present/already disabled: $svc" | tee -a evidence/e3/services_disabled.txt
  fi
done

# List enabled services after changes
systemctl list-unit-files --state=enabled --no-pager | tee evidence/e3/services_enabled_after.txt

# Generate diff
diff evidence/e3/services_enabled_before.txt evidence/e3/services_enabled_after.txt \
  | tee evidence/e3/services_diff.txt
```

### Step 4 — Phase D: Filesystem and account hardening

```bash
# 4a: Set restrictive permissions on sensitive files
echo "=== File permission hardening ===" | tee evidence/e3/file_perms_before.txt
for f in /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/sudoers; do
  [ -f "$f" ] && stat "$f" | grep -E "File:|Access:" | tee -a evidence/e3/file_perms_before.txt
done

# Apply CIS-recommended permissions
sudo chmod 644 /etc/passwd /etc/group 2>/dev/null
sudo chmod 640 /etc/shadow /etc/gshadow 2>/dev/null
sudo chown root:shadow /etc/shadow /etc/gshadow 2>/dev/null || sudo chown root:root /etc/shadow /etc/gshadow 2>/dev/null

# After
echo "=== File permissions after ===" | tee evidence/e3/file_perms_after.txt
for f in /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/sudoers; do
  [ -f "$f" ] && stat "$f" | grep -E "File:|Access:" | tee -a evidence/e3/file_perms_after.txt
done

# 4b: Ensure /tmp is mounted with noexec,nosuid
echo "=== /tmp mount options ===" | tee evidence/e3/tmp_mount_check.txt
findmnt /tmp | tee -a evidence/e3/tmp_mount_check.txt

# 4c: Find world-writable files outside of expected locations
echo "=== World-writable files ===" | tee evidence/e3/world_writable_check.txt
sudo find / -xdev -perm -0002 -type f ! -path "/tmp/*" ! -path "/var/tmp/*" 2>/dev/null \
  | tee -a evidence/e3/world_writable_check.txt | wc -l

# 4d: Find files with no owner
echo "=== Unowned files ===" | tee evidence/e3/unowned_files.txt
sudo find / -xdev -nouser -o -nogroup 2>/dev/null | tee -a evidence/e3/unowned_files.txt | wc -l
```

### Step 5 — Phase E: Post-hardening lynis audit

```bash
# Re-run lynis audit after all hardening
sudo lynis audit system --no-colors --quiet 2>&1 | tee evidence/e3/lynis_audit_after.txt

# Extract post-hardening score
grep "Hardening index" evidence/e3/lynis_audit_after.txt | tee evidence/e3/lynis_score_after.txt

# Generate comparison
echo "=== Hardening Score Comparison ===" | tee evidence/e3/lynis_score_comparison.txt
echo "Before:" | tee -a evidence/e3/lynis_score_comparison.txt
cat evidence/e3/lynis_score_before.txt | tee -a evidence/e3/lynis_score_comparison.txt
echo "After:" | tee -a evidence/e3/lynis_score_comparison.txt
cat evidence/e3/lynis_score_after.txt | tee -a evidence/e3/lynis_score_comparison.txt

# Post-snapshot
sudo snapper -c root create --description "post-e3-platform-baseline" --print-number \
  | tee evidence/e3/snapshot_post_num.txt
```

Expected:
- Post-hardening lynis score is measurably higher than the pre-hardening score.
- `lynis_score_comparison.txt` documents the improvement.

## 8) Validation

```bash
# Verify key kernel parameters
PASS_COUNT=0
FAIL_COUNT=0

check_sysctl() {
  local param="$1" expected="$2"
  val=$(sysctl -n "$param" 2>/dev/null)
  if [ "$val" = "$expected" ]; then
    echo "PASS: $param = $val"
    ((PASS_COUNT++))
  else
    echo "FAIL: $param = $val (expected $expected)"
    ((FAIL_COUNT++))
  fi
}

check_sysctl net.ipv4.ip_forward 0
check_sysctl net.ipv4.conf.all.accept_redirects 0
check_sysctl net.ipv4.tcp_syncookies 1
check_sysctl kernel.dmesg_restrict 1
check_sysctl kernel.randomize_va_space 2
check_sysctl kernel.yama.ptrace_scope 1

echo "Validation: $PASS_COUNT PASS, $FAIL_COUNT FAIL" | tee evidence/e3/sysctl_validation_summary.txt

# Verify lynis score improved
BEFORE_SCORE=$(grep -oP '\d+' evidence/e3/lynis_score_before.txt | tail -1)
AFTER_SCORE=$(grep -oP '\d+' evidence/e3/lynis_score_after.txt | tail -1)
echo "Lynis score before: $BEFORE_SCORE" | tee evidence/e3/final_validation.txt
echo "Lynis score after: $AFTER_SCORE" | tee -a evidence/e3/final_validation.txt
[ "$AFTER_SCORE" -gt "$BEFORE_SCORE" ] \
  && echo "PASS: Score improved ($BEFORE_SCORE → $AFTER_SCORE)" | tee -a evidence/e3/final_validation.txt \
  || echo "WARN: Score did not improve — review lynis output" | tee -a evidence/e3/final_validation.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/e3/snapshot_pre_num.txt`, `snapshot_post_num.txt` — Snapper snapshots
  - `evidence/e3/lynis_version.txt` — lynis version
  - `evidence/e3/lynis_audit_before.txt` — pre-hardening full lynis report
  - `evidence/e3/lynis_score_before.txt` — pre-hardening score
  - `evidence/e3/lynis_warnings_before.txt` — pre-hardening warnings
  - `evidence/e3/sysctl_apply_output.txt` — sysctl apply output
  - `evidence/e3/sysctl_verify.txt` — sysctl verification
  - `evidence/e3/services_enabled_before.txt` — service inventory before
  - `evidence/e3/services_disabled.txt` — list of disabled services
  - `evidence/e3/services_enabled_after.txt` — service inventory after
  - `evidence/e3/services_diff.txt` — service change diff
  - `evidence/e3/file_perms_before.txt` — file permissions before
  - `evidence/e3/file_perms_after.txt` — file permissions after
  - `evidence/e3/tmp_mount_check.txt` — /tmp mount options
  - `evidence/e3/world_writable_check.txt` — world-writable files
  - `evidence/e3/unowned_files.txt` — files without owners
  - `evidence/e3/lynis_audit_after.txt` — post-hardening full lynis report
  - `evidence/e3/lynis_score_after.txt` — post-hardening score
  - `evidence/e3/lynis_score_comparison.txt` — before vs after comparison
  - `evidence/e3/sysctl_validation_summary.txt` — sysctl pass/fail counts
  - `evidence/e3/final_validation.txt` — final validation results
- **Hash manifest:**

```bash
find evidence/e3 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/e3/SHA256SUMS.txt
cat evidence/e3/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** After applying `kernel.yama.ptrace_scope = 1`, a debugging workflow breaks (e.g., `gdb` cannot attach to running processes).
  - **Cause:** `ptrace_scope = 1` restricts ptrace to parent processes only. External debuggers need `ptrace_scope = 0`.
  - **Fix:** Temporarily: `sudo sysctl kernel.yama.ptrace_scope=0`. Permanently: change the value in the sysctl config file and re-apply.
  - **Rollback trigger:** Snapper rollback of the sysctl.d config file.

- **Symptom:** `fs.suid_dumpable = 0` prevents core dumps from suid binaries, breaking a debugging workflow.
  - **Cause:** This is the intended behavior — but may conflict with development workflows.
  - **Fix:** On the bare-metal host, accept this restriction. In a development VM, set `fs.suid_dumpable = 1`.
  - **Rollback trigger:** Remove the relevant line from the sysctl config and `sudo sysctl --system`.

- **Symptom:** `lynis` post-hardening score does not improve.
  - **Cause:** lynis may flag a different set of controls than what was applied, or the applied controls were already at their target value before the project.
  - **Fix:** Read `lynis_audit_after.txt` warnings section and address specific items. Not every lynis warning maps to a control in this project — that is expected.
  - **Rollback trigger:** N/A — read-only lynis audit.

## 11) Sources

- [CIS Benchmark — Distribution Independent Linux](https://www.cisecurity.org/benchmark/distribution_independent_linux)
- [DISA STIG — Red Hat Enterprise Linux (adapted for context)](https://www.stigviewer.com/stig/red_hat_enterprise_linux_9/)
- [OpenSCAP Security Guide](https://www.open-scap.org/security-policies/scap-security-guide/)
- [Lynis security auditing tool](https://cisofy.com/lynis/)
- [Arch Wiki — Security — Kernel parameters](https://wiki.archlinux.org/title/Security#Kernel_parameters)
- [NIST SP 800-123 — Guide to General Server Security](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-123.pdf)

## 12) Stretch Goals

- Run OpenSCAP with the `ssg-fedora-ds.xml` or equivalent content against the system and compare its findings with lynis. Document which tool catches controls the other misses.
- Create a second hardening config file (`91-e3-network-advanced.conf`) covering IPv6 privacy addresses, TCP timestamp disabling, and Explicit Congestion Notification settings. Document the rationale for each.
- Write a hardening remediation script (`e3-apply-baseline.sh`) that reads a hardening configuration YAML file and applies each control, logging pass/fail to a structured report. This is the beginning of a configuration-as-code hardening pipeline.
- Set up daily lynis cron job that alerts (via `logger` to syslog) if the hardening score drops below a defined threshold, indicating a configuration regression.
