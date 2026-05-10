---
title: I1 - Systemd Service Hardening
aliases:
  - i1-systemd-hardening
  - systemd-service-security
tags:
  - intermediate
  - systemd
  - hardening
  - sandboxing
  - evidence
  - net412
date: 2026-05-10
---

# I1 — Systemd Service Hardening

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host. A custom or existing system service is selected as the hardening target. The service must be non-critical (i.e., losing it does not break the host) — use a test service or a low-criticality daemon.
> **Risk level:** Medium — service unit files are modified. A misconfigured unit may prevent the service from starting. Snapper snapshot is mandatory before any unit file edits.
> **Threat model:** Systemd services running with default settings have access to the full system: all filesystems, all capabilities, root if configured poorly. A compromised or vulnerable service can pivot to full host compromise. Sandboxing via systemd security directives limits the blast radius of such exploits to the service's declared resource access.
> **Out of scope:** AppArmor/SELinux LSM profiles (separate project). Kernel capability bounding sets beyond what systemd exposes. Network namespace isolation (covered in A2).

## 1) Mission

- **Problem statement:** Most systemd service units distributed by Linux packages use minimal or no sandboxing. `systemd-analyze security <unit>` scores most services as "UNSAFE" or "MEDIUM" out of the box. This project walks through systematically applying the available systemd security directives to a service and measurably improving its isolation score.
- **Why it matters:** Understanding systemd unit hardening is a core NET 412 competency. It directly maps to CIS Benchmark controls and NIST 800-53 least-privilege requirements. Every service in the lab that stores secrets, handles network traffic, or runs as a privileged user should have these controls applied.

## 2) Difficulty

- Intermediate (estimated 4–6 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal. Target a non-critical service such as a custom Python HTTP server unit, `syncthing`, or another user-space service. Do not target core system services (`sshd`, `NetworkManager`, `dbus`) until you are comfortable with rollback procedures.

## 4) Prerequisites

- **Skills:** Comfortable with `systemctl` start/stop/restart/status, reading unit file syntax, understanding of Linux process capabilities and filesystem namespaces.
- **Tools:** `systemctl`, `systemd-analyze`, `journalctl`, `systemd-run`, `capsh`, `tee`
- **Dependencies:**
  - A target service unit to harden (create a simple test service if needed — see Step 1).
  - `libcap` installed for `capsh`: `pacman -Q libcap`
  - `sudo` access.

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-i1-systemd-hardening"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-i1-systemd-hardening"`
- **VM snapshot:** N/A (host project)
- **Container rebuild command:** N/A
- **Rollback trigger:** If the service fails to start after hardening changes: `sudo snapper -c root undochange <PRE>..<POST>` then `sudo systemctl daemon-reload && sudo systemctl restart <service>`.

## 6) Project Plan

- **Phase A — Baseline Assessment:** Capture the pre-hardening security score, unit file contents, and runtime capabilities.
- **Phase B — Hardening Application:** Apply systemd security directives in order from least- to most-disruptive. Test after each batch.
- **Phase C — Post-Hardening Validation:** Re-run `systemd-analyze security`, confirm service remains functional, capture final evidence.

## 7) Walkthrough

### Step 1 — Create a test service (if no safe target exists)

```bash
mkdir -p evidence/i1

# Create a simple long-running test service
# This service runs a harmless infinite loop and writes to a temp file
cat | sudo tee /etc/systemd/system/b-test-hardening.service << 'EOF'
[Unit]
Description=B-Test Hardening Lab Service
After=network.target

[Service]
Type=simple
ExecStart=/bin/bash -c "while true; do date -u >> /tmp/b-test.log; sleep 30; done"
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the test service
sudo systemctl daemon-reload
sudo systemctl enable --now b-test-hardening.service
systemctl status b-test-hardening.service --no-pager | tee evidence/i1/service_status_initial.txt
```

Expected:
- `b-test-hardening.service` shows `active (running)`.

> [!info] If you have a real non-critical service (e.g., `syncthing.service`, a custom API server, a log collector), use that as your target. Replace `b-test-hardening` with its unit name throughout.

### Step 2 — Capture pre-hardening baseline

```bash
TARGET="b-test-hardening.service"

# Record full unit file as shipped
systemctl cat "$TARGET" | tee evidence/i1/unit_file_before.txt

# Run security assessment BEFORE any changes
systemd-analyze security "$TARGET" | tee evidence/i1/security_score_before.txt

# Show process capabilities at runtime
PID=$(systemctl show -p MainPID --value "$TARGET")
if [ -n "$PID" ] && [ "$PID" != "0" ]; then
  cat /proc/$PID/status | grep -E "^Cap" | tee evidence/i1/proc_capabilities_before.txt
  capsh --decode=$(grep CapPrm /proc/$PID/status | awk '{print $2}') | tee evidence/i1/capabilities_decoded_before.txt
fi

# Show namespace isolation status
ls -la /proc/$PID/ns/ 2>/dev/null | tee evidence/i1/namespaces_before.txt

# Capture journal for the service
journalctl -u "$TARGET" --no-pager -n 30 | tee evidence/i1/service_journal_before.txt
```

Expected:
- `security_score_before.txt` shows the current exposure score (often `UNSAFE` or a high numeric value — lower is better in systemd-analyze output).
- `capabilities_decoded_before.txt` shows which Linux capabilities the process currently holds.

### Step 3 — Create Snapper pre-change snapshot

```bash
sudo snapper -c root create --description "pre-i1-systemd-hardening" --print-number \
  | tee evidence/i1/snapshot_pre_num.txt
echo "Pre-change snapshot: $(cat evidence/i1/snapshot_pre_num.txt)"
```

### Step 4 — Apply hardening directives (Phase 1: filesystem restrictions)

Create a systemd drop-in override to avoid modifying the original unit file:

```bash
sudo mkdir -p /etc/systemd/system/b-test-hardening.service.d/

cat | sudo tee /etc/systemd/system/b-test-hardening.service.d/hardening.conf << 'EOF'
[Service]
# === Filesystem restrictions ===
# Mount entire filesystem read-only; selectively allow writes where needed
ProtectSystem=strict
# Make /home, /root, /run/user inaccessible
ProtectHome=true
# Provide private /tmp that is not shared with the host
PrivateTmp=true
# Prevent writing to /proc and /sys kernel tunables
ProtectKernelTunables=true
# Prevent loading kernel modules
ProtectKernelModules=true
# Prevent accessing kernel log buffer
ProtectKernelLogs=true
# Prevent modification to hostname, locale, machine-id
ProtectControlGroups=true
ProtectHostname=true
# Restrict to only the necessary paths
ReadWritePaths=/tmp/b-test.log
EOF

# Reload and restart to apply Phase 1
sudo systemctl daemon-reload
sudo systemctl restart b-test-hardening.service
systemctl status b-test-hardening.service --no-pager | tee evidence/i1/service_status_phase1.txt

# Check security score after Phase 1
systemd-analyze security b-test-hardening.service | tee evidence/i1/security_score_phase1.txt
```

Expected:
- Service remains `active (running)` after Phase 1 changes.
- Security score improves (lower UNSAFE score or upgrade from UNSAFE to MEDIUM).

### Step 5 — Apply hardening directives (Phase 2: privilege and capability restrictions)

```bash
cat | sudo tee -a /etc/systemd/system/b-test-hardening.service.d/hardening.conf << 'EOF'

# === Privilege restrictions ===
# Prevent privilege escalation via setuid/setgid
NoNewPrivileges=true
# Drop all ambient capabilities
AmbientCapabilities=
CapabilityBoundingSet=
# Run as a non-root user
DynamicUser=true

# === Syscall filtering ===
# Restrict to safe system call sets
SystemCallFilter=@system-service
SystemCallArchitectures=native

# === Network namespace (no network access needed for this service) ===
PrivateNetwork=true
EOF

sudo systemctl daemon-reload
sudo systemctl restart b-test-hardening.service
systemctl status b-test-hardening.service --no-pager | tee evidence/i1/service_status_phase2.txt
systemd-analyze security b-test-hardening.service | tee evidence/i1/security_score_phase2.txt
```

Expected:
- Service remains `active (running)`.
- Security score improves further — may reach `MEDIUM` or `OK`.

### Step 6 — Capture post-hardening state and diff

```bash
# Final security score
systemd-analyze security b-test-hardening.service | tee evidence/i1/security_score_final.txt

# Show effective unit configuration (merged with drop-in)
systemctl cat b-test-hardening.service | tee evidence/i1/unit_file_final.txt

# Diff: before vs final score
echo "=== BEFORE ===" > evidence/i1/security_score_diff.txt
grep -E "^b-test|Overall" evidence/i1/security_score_before.txt >> evidence/i1/security_score_diff.txt
echo "=== AFTER ===" >> evidence/i1/security_score_diff.txt
grep -E "^b-test|Overall" evidence/i1/security_score_final.txt >> evidence/i1/security_score_diff.txt
cat evidence/i1/security_score_diff.txt

# Capture journal to confirm service is running correctly
journalctl -u b-test-hardening.service --no-pager -n 20 | tee evidence/i1/service_journal_final.txt

# Verify /tmp isolation: the service's /tmp should be separate from host /tmp
ls /tmp/b-test.log 2>/dev/null && echo "WARNING: File visible in host /tmp (PrivateTmp may not be working)" \
  || echo "PASS: /tmp/b-test.log not visible in host /tmp (PrivateTmp working)"
```

Expected:
- `security_score_diff.txt` shows a clear improvement in the overall score.
- The file `/tmp/b-test.log` is not visible in the host's `/tmp` (PrivateTmp creates an isolated namespace).

## 8) Validation

```bash
TARGET="b-test-hardening.service"

# Service is still running
systemctl is-active "$TARGET" \
  && echo "PASS: service is active" \
  || echo "FAIL: service is not active"

# PrivateTmp working (file not in host /tmp)
ls /tmp/b-test.log 2>/dev/null \
  && echo "WARN: file visible in host /tmp" \
  || echo "PASS: PrivateTmp isolation confirmed"

# NoNewPrivileges is set
systemctl show -p NoNewPrivileges "$TARGET" \
  | grep "NoNewPrivileges=yes" \
  && echo "PASS: NoNewPrivileges=yes" \
  || echo "FAIL: NoNewPrivileges not set"

# ProtectSystem is set
systemctl show -p ProtectSystem "$TARGET" \
  | grep -v "ProtectSystem=$" \
  && echo "PASS: ProtectSystem configured" \
  || echo "FAIL: ProtectSystem not set"

# Confirm security score improved
echo "Security score before:"
grep "Overall" evidence/i1/security_score_before.txt || grep -m1 "UNSAFE\|MEDIUM\|OK" evidence/i1/security_score_before.txt
echo "Security score after:"
grep "Overall" evidence/i1/security_score_final.txt || grep -m1 "UNSAFE\|MEDIUM\|OK" evidence/i1/security_score_final.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/i1/service_status_initial.txt` — initial service status
  - `evidence/i1/unit_file_before.txt` — original unit file
  - `evidence/i1/security_score_before.txt` — pre-hardening security score
  - `evidence/i1/proc_capabilities_before.txt` — raw capability bitmask
  - `evidence/i1/capabilities_decoded_before.txt` — decoded capabilities
  - `evidence/i1/namespaces_before.txt` — namespace isolation before
  - `evidence/i1/service_journal_before.txt` — service log before
  - `evidence/i1/snapshot_pre_num.txt` — Snapper snapshot number
  - `evidence/i1/service_status_phase1.txt` — status after Phase 1
  - `evidence/i1/security_score_phase1.txt` — score after Phase 1
  - `evidence/i1/service_status_phase2.txt` — status after Phase 2
  - `evidence/i1/security_score_phase2.txt` — score after Phase 2
  - `evidence/i1/security_score_final.txt` — final security score
  - `evidence/i1/unit_file_final.txt` — final merged unit configuration
  - `evidence/i1/security_score_diff.txt` — before vs after comparison
  - `evidence/i1/service_journal_final.txt` — service log after hardening
- **Hash manifest:**

```bash
find evidence/i1 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/i1/SHA256SUMS.txt
cat evidence/i1/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** Service fails to start after applying hardening directives with `Permission denied` in journal.
  - **Cause:** `ReadWritePaths` or `ProtectSystem=strict` is blocking a path the service actually writes to.
  - **Fix:** Check `journalctl -u <service> -n 50` for the denied path. Add it to `ReadWritePaths=` in the drop-in. Restart service.
  - **Rollback trigger:** If service cannot be recovered by adjusting paths: `sudo snapper -c root undochange <PRE>..<POST>`

- **Symptom:** `systemd-analyze security` shows no improvement after changes.
  - **Cause:** Drop-in file has a syntax error and is silently ignored, or `daemon-reload` was not run.
  - **Fix:** `sudo systemd-analyze verify b-test-hardening.service` to check for syntax errors. Always run `sudo systemctl daemon-reload` before restarting.
  - **Rollback trigger:** N/A — service is still running with original settings.

- **Symptom:** `DynamicUser=true` causes the service to fail because it cannot write to paths owned by the original user.
  - **Cause:** Dynamic users get a fresh UID each start; they cannot own pre-created files.
  - **Fix:** Use `StateDirectory=`, `LogsDirectory=`, or `RuntimeDirectory=` directives so systemd creates and owns the directories automatically. Remove manual `ReadWritePaths` entries for those paths.
  - **Rollback trigger:** Remove `DynamicUser=true` from the drop-in and restart.

## 11) Sources

- [systemd.exec man page — security directives](https://man7.org/linux/man-pages/man5/systemd.exec.5.html)
- [systemd-analyze security documentation](https://man7.org/linux/man-pages/man1/systemd-analyze.1.html)
- [Arch Wiki — systemd](https://wiki.archlinux.org/title/Systemd)
- [Lennart Poettering — Systemd for Administrators: Service Sandboxing](http://0pointer.net/blog/projects/security.html)
- [CIS Benchmark — Linux process isolation](https://www.cisecurity.org/benchmark/distribution_independent_linux)
- [NIST SP 800-53 — SC-39 Process Isolation](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)

## 12) Stretch Goals

- Apply the same hardening process to a real service: `sshd.service`, `nginx.service`, or another package-provided service. Use drop-in overrides so the package manager can still update the base unit.
- Achieve an `OK` (green) rating from `systemd-analyze security` on your test service. Document which directives had the largest score impact.
- Write a Bash wrapper (`i1-harden-check.sh`) that runs `systemd-analyze security` on all active services and reports any scoring `UNSAFE`.
- Configure `systemd-coredump` and trigger a controlled crash of the test service inside its private namespace. Confirm the coredump is captured and that the crash was contained within the service's sandbox.

