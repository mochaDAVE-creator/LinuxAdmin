---
title: A1 - Centralized Logging Pipeline
aliases:
  - a1-centralized-logging
  - log-pipeline-advanced
tags:
  - advanced
  - logging
  - detection
  - journald
  - evidence
  - net412
  - net179
date: 2026-05-10
---

# A1 — Centralized Logging Pipeline

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host and one or more Proxmox VMs. The goal is to forward logs from guest VMs to the Arch host acting as a log collector, using systemd-journal-remote or a compatible transport.
> **Risk level:** High — network services are exposed and log storage configuration is modified. A misconfigured pipeline can silently drop logs or expose sensitive log data over the network.
> **Threat model:** Without centralized logging, an attacker who compromises a VM can clear local logs without leaving a trace on the host. A centralized log pipeline means the attacker must also compromise the collector to fully cover their tracks. This project implements a minimal but hardened log collection architecture.
> **Out of scope:** SIEM platforms (Splunk, Elastic, Graylog), encrypted TLS log transport (requires PKI setup). This project uses cleartext or self-signed TLS on a trusted lab network only.
> **Network note:** All log forwarding in this project occurs on an isolated lab network (Proxmox internal bridge), not exposed to the internet.

## 1) Mission

- **Problem statement:** Individual host logs are easy to tamper with after a compromise. A centralized log pipeline creates a second, harder-to-tamper copy of all log events. Building one also forces you to understand the full systemd journal architecture: how logs are generated, stored, forwarded, and queried.
- **Why it matters:** Centralized logging is a NET 179 (Digital Forensics) and NET 412 (Linux Admin) requirement. It is also the prerequisite infrastructure for A3 (Threat-Informed Detection Pack) and E4 (Purple-Team Validation Cycle).

## 2) Difficulty

- Advanced (estimated 8–12 focused hours)

## 3) Execution Context

- **Host** (log collector) — Arch Linux bare metal running `systemd-journal-remote` in passive/receive mode.
- **VM** (log forwarder) — One or more Proxmox VMs running `systemd-journal-upload` or `systemd-journal-gateway` to forward logs to the host.

## 4) Prerequisites

- **Skills:** Completion of B4 (Log Triage Starter), I1 (Systemd Service Hardening). Understanding of TCP/IP networking and systemd unit configuration.
- **Tools:** `systemd-journal-remote`, `systemd-journal-upload`, `journalctl`, `curl`, `openssl`, `tee`, `ss`, `firewall-cmd` or `nftables`
- **Dependencies:**
  - `systemd-journal-remote` package on the Arch host: `pacman -S systemd-journal-remote`
  - At least one Proxmox VM with `systemd` running and network connectivity to the host.
  - Proxmox internal network bridge configured (e.g., `vmbr1`) for lab traffic.

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-a1-logging-pipeline"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-a1-logging-pipeline"`
- **VM snapshot (pre):** Take a Proxmox snapshot of all participating VMs before installing journal-upload.
- **VM snapshot rollback:** `qm rollback <VMID> <snapshot-name>`
- **Container rebuild command:** N/A
- **Rollback trigger:** If log forwarding causes unexpected behavior or disk fill: `sudo snapper -c root undochange <PRE>..<POST>` on the host; `qm rollback` on VMs. Disable the pipeline services: `sudo systemctl disable --now systemd-journal-remote.socket systemd-journal-remote.service`

## 6) Project Plan

- **Phase A — Collector Setup:** Install and configure `systemd-journal-remote` on the Arch host to receive logs.
- **Phase B — Forwarder Setup:** Install and configure `systemd-journal-upload` on the test VM to push logs.
- **Phase C — Validation Queries:** Verify logs from the VM appear on the collector; build 5 investigation queries against the collected journal.

## 7) Walkthrough

### Step 1 — Pre-snapshot and environment check

```bash
mkdir -p evidence/a1

# Record environment
uname -a | tee evidence/a1/env_uname.txt
ip addr show | tee evidence/a1/env_network.txt
date -u | tee evidence/a1/env_datetime.txt

# Snapper pre-snapshot
sudo snapper -c root create --description "pre-a1-logging-pipeline" --print-number \
  | tee evidence/a1/snapshot_pre_num.txt

# Install systemd-journal-remote if not present
pacman -Q systemd-journal-remote 2>/dev/null \
  || sudo pacman -S --noconfirm systemd-journal-remote
pacman -Q systemd-journal-remote | tee evidence/a1/pkg_journal_remote_version.txt
```

Expected:
- `systemd-journal-remote` package installed and version recorded.
- Network interfaces visible — note the IP on the lab bridge (e.g., `192.168.100.1` on `vmbr1`).

### Step 2 — Configure systemd-journal-remote on the host (collector)

```bash
# Show default configuration
cat /lib/systemd/system/systemd-journal-remote.service | tee evidence/a1/journal_remote_unit_default.txt

# Create configuration directory for journal-remote
sudo mkdir -p /etc/systemd/journal-remote.conf.d/

# Configure journal-remote to listen on HTTP (lab-only — use HTTPS in production)
cat | sudo tee /etc/systemd/journal-remote.conf.d/lab.conf << 'EOF'
[Remote]
# Passive mode: listen for incoming connections
ServerKeyFile=
ServerCertificateFile=
TrustedCertificateFile=
# Where to store received journals
Output=/var/log/journal/remote/
EOF

# Create the remote journal directory
sudo mkdir -p /var/log/journal/remote
sudo chown systemd-journal-remote:systemd-journal /var/log/journal/remote
sudo chmod 750 /var/log/journal/remote

# Enable and start the socket-activated receiver
sudo systemctl daemon-reload
sudo systemctl enable --now systemd-journal-remote.socket
sudo systemctl enable --now systemd-journal-remote.service

# Check status
systemctl status systemd-journal-remote.socket --no-pager | tee evidence/a1/journal_remote_socket_status.txt
systemctl status systemd-journal-remote.service --no-pager | tee evidence/a1/journal_remote_service_status.txt

# Verify listening port (default 19532)
ss -tlnp | grep 19532 | tee evidence/a1/journal_remote_listening_port.txt
```

Expected:
- `systemd-journal-remote.socket` shows `active (listening)` on port 19532.
- `/var/log/journal/remote/` directory exists and is owned by `systemd-journal-remote`.

### Step 3 — Configure systemd-journal-upload on the VM (forwarder)

> [!info] Run this step inside the Proxmox test VM. SSH to the VM first: `ssh root@<VM_IP>`

```bash
# On the VM: install systemd-journal-upload (may need package name adjustment per distro)
# Debian/Ubuntu: apt install systemd-journal-remote
# Arch: pacman -S systemd-journal-remote
systemctl --version | head -1  # Confirm systemd version

# Configure journal-upload to point at the host collector
# Replace HOST_IP with the Arch host IP on the lab network
HOST_IP="192.168.100.1"  # Update to your actual host IP

sudo mkdir -p /etc/systemd/journal-upload.conf.d/
cat | sudo tee /etc/systemd/journal-upload.conf.d/lab.conf << EOF
[Upload]
URL=http://${HOST_IP}:19532
EOF

# Enable and start journal-upload
sudo systemctl daemon-reload
sudo systemctl enable --now systemd-journal-upload.service

# Check status
systemctl status systemd-journal-upload.service --no-pager

# Confirm upload is active
journalctl -u systemd-journal-upload --no-pager -n 20
```

Expected:
- `systemd-journal-upload.service` shows `active (running)`.
- Journal shows connection attempts and successes to the host IP:19532.

### Step 4 — Verify logs appear on the collector

```bash
# On the host: wait 30 seconds then check for received journals
sleep 30

# List received journal files
ls -lah /var/log/journal/remote/ | tee evidence/a1/remote_journal_files.txt

# Read received journals from the VM
VM_JOURNAL=$(ls /var/log/journal/remote/remote-*.journal 2>/dev/null | head -1)
if [ -n "$VM_JOURNAL" ]; then
  journalctl --file="$VM_JOURNAL" --no-pager -n 50 | tee evidence/a1/remote_journal_sample.txt
  echo "Remote journal file: $VM_JOURNAL" | tee evidence/a1/remote_journal_path.txt
else
  echo "No remote journal files yet — check VM upload service and network connectivity" \
    | tee evidence/a1/remote_journal_path.txt
fi
```

Expected:
- At least one `remote-*.journal` file appears in `/var/log/journal/remote/`.
- `remote_journal_sample.txt` contains recent entries from the VM's journal.

### Step 5 — Build five investigation queries against the collected journal

```bash
VM_JOURNAL=$(ls /var/log/journal/remote/remote-*.journal 2>/dev/null | head -1)
[ -z "$VM_JOURNAL" ] && echo "ERROR: No remote journal found" && exit 1

# Query 1: All errors from the VM in the last 24 hours
echo "=== Query 1: VM errors in last 24h ===" | tee evidence/a1/investigation_queries.txt
journalctl --file="$VM_JOURNAL" -p err --since "24 hours ago" --no-pager \
  | tee -a evidence/a1/investigation_queries.txt

# Query 2: Authentication events from the VM
echo "=== Query 2: VM auth events ===" | tee -a evidence/a1/investigation_queries.txt
journalctl --file="$VM_JOURNAL" _SYSTEMD_UNIT=sshd.service --no-pager -n 30 \
  | tee -a evidence/a1/investigation_queries.txt

# Query 3: Failed units on the VM
echo "=== Query 3: VM failed systemd units ===" | tee -a evidence/a1/investigation_queries.txt
journalctl --file="$VM_JOURNAL" --no-pager \
  | grep -i "failed\|start-limit-hit" | head -20 \
  | tee -a evidence/a1/investigation_queries.txt

# Query 4: Kernel messages from the VM
echo "=== Query 4: VM kernel messages ===" | tee -a evidence/a1/investigation_queries.txt
journalctl --file="$VM_JOURNAL" _TRANSPORT=kernel --no-pager -n 20 \
  | tee -a evidence/a1/investigation_queries.txt

# Query 5: Simulate a failed SSH login on VM and verify it appears on collector
echo "=== Query 5: Simulated auth failure detection ===" | tee -a evidence/a1/investigation_queries.txt
# On the VM (via SSH), generate a failed login:
# ssh baduser@localhost (will fail)
# Then on the host, check if it appears in the remote journal within 60s
journalctl --file="$VM_JOURNAL" --since "5 minutes ago" --no-pager \
  | grep -i "authentication failure\|invalid user\|Failed password" \
  | tee -a evidence/a1/investigation_queries.txt \
  || echo "No recent auth failures found (generate a test failure on the VM)" \
  | tee -a evidence/a1/investigation_queries.txt
```

### Step 6 — Measure log delivery latency

```bash
# Generate a uniquely tagged log entry on the VM
MARKER="A1-LATENCY-TEST-$(date -u +%s)"
# On the VM: logger -p user.info "$MARKER"
# (Run this command over SSH on the VM)
ssh root@<VM_IP> "logger -p user.info '$MARKER'" 2>/dev/null \
  || echo "Run manually on VM: logger -p user.info '$MARKER'"

# On the host: poll for the marker in the remote journal
START=$(date +%s)
for i in $(seq 1 30); do
  VM_JOURNAL=$(ls /var/log/journal/remote/remote-*.journal 2>/dev/null | head -1)
  journalctl --file="$VM_JOURNAL" --no-pager -n 100 2>/dev/null \
    | grep -q "$MARKER" && break
  sleep 2
done
END=$(date +%s)
LATENCY=$((END - START))
echo "Log delivery latency for marker $MARKER: ~${LATENCY}s" | tee evidence/a1/log_latency_test.txt
```

Expected:
- Latency should be under 60 seconds for a default journal-upload configuration.

## 8) Validation

```bash
# 1. Remote socket is listening
ss -tlnp | grep 19532 \
  && echo "PASS: journal-remote listening on 19532" \
  || echo "FAIL: journal-remote not listening"

# 2. Remote journal files exist
ls /var/log/journal/remote/remote-*.journal &>/dev/null \
  && echo "PASS: Remote journal files present" \
  || echo "FAIL: No remote journal files"

# 3. Journal-upload on VM is active (check remotely)
ssh root@<VM_IP> "systemctl is-active systemd-journal-upload" 2>/dev/null \
  && echo "PASS: journal-upload active on VM" \
  || echo "MANUAL CHECK: verify journal-upload on VM"

# 4. Investigation queries ran (file non-empty)
[ -s evidence/a1/investigation_queries.txt ] \
  && echo "PASS: Investigation queries file populated" \
  || echo "FAIL: Investigation queries file empty"

# 5. Log delivery latency measured
[ -f evidence/a1/log_latency_test.txt ] \
  && cat evidence/a1/log_latency_test.txt \
  || echo "FAIL: Latency test not completed"
```

## 9) Evidence

- **Output files:**
  - `evidence/a1/env_uname.txt`, `env_network.txt`, `env_datetime.txt` — host environment
  - `evidence/a1/snapshot_pre_num.txt` — Snapper pre-snapshot number
  - `evidence/a1/pkg_journal_remote_version.txt` — package version
  - `evidence/a1/journal_remote_unit_default.txt` — default unit file
  - `evidence/a1/journal_remote_socket_status.txt` — socket service status
  - `evidence/a1/journal_remote_service_status.txt` — main service status
  - `evidence/a1/journal_remote_listening_port.txt` — port 19532 listener
  - `evidence/a1/remote_journal_files.txt` — received journal file listing
  - `evidence/a1/remote_journal_sample.txt` — sample of received log entries
  - `evidence/a1/remote_journal_path.txt` — path to received journal
  - `evidence/a1/investigation_queries.txt` — five investigation query results
  - `evidence/a1/log_latency_test.txt` — delivery latency measurement
- **Hash manifest:**

```bash
find evidence/a1 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/a1/SHA256SUMS.txt
cat evidence/a1/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `systemd-journal-remote.socket` fails to start with "address already in use."
  - **Cause:** Port 19532 is in use by another process.
  - **Fix:** `ss -tlnp | grep 19532` to identify the process. Change the port in the socket unit via a drop-in if needed.
  - **Rollback trigger:** Snapper rollback removes all configuration changes.

- **Symptom:** VM journal-upload shows connection refused.
  - **Cause:** Host firewall is blocking port 19532, or journal-remote socket is not active.
  - **Fix:** On host: `sudo systemctl start systemd-journal-remote.socket`. Check firewall: `sudo nft list ruleset | grep 19532`.
  - **Rollback trigger:** N/A.

- **Symptom:** Remote journal files appear but are empty or contain no entries.
  - **Cause:** Journal-upload is connecting but not yet sending historical entries (it starts from the cursor).
  - **Fix:** Generate new log events on the VM: `logger -p user.info "test message"`. Wait 30s and re-check.
  - **Rollback trigger:** N/A.

## 11) Sources

- [systemd-journal-remote documentation](https://man7.org/linux/man-pages/man8/systemd-journal-remote.8.html)
- [systemd-journal-upload documentation](https://man7.org/linux/man-pages/man8/systemd-journal-upload.8.html)
- [Arch Wiki — systemd/Journal](https://wiki.archlinux.org/title/Systemd/Journal)
- [CISA — Logging and Monitoring Best Practices](https://www.cisa.gov/sites/default/files/2023-03/CISA_Logging_Made_Easy.pdf)
- [NIST SP 800-92 — Guide to Computer Security Log Management](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-92.pdf)
- [NIST SP 800-61r2 — Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)

## 12) Stretch Goals

- Add TLS encryption to the journal-remote pipeline using self-signed certificates generated with `openssl`. Update both the host listener and VM uploader configurations.
- Add a second VM forwarder and demonstrate that both VM journals appear as separate files in `/var/log/journal/remote/`, queryable independently with `journalctl --file=`.
- Write a `journalctl` wrapper query (`a1-search-remote.sh`) that searches all remote journal files simultaneously for a given pattern.
- Configure log retention limits in `journald.conf` on both the host and the remote directory to prevent disk exhaustion: `SystemMaxUse=`, `SystemKeepFree=`.

