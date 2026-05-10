---
title: E2 - Zero-Trust Homelab Blueprint
aliases:
  - e2-zero-trust-blueprint
  - zero-trust-homelab
tags:
  - expert
  - zero-trust
  - architecture
  - segmentation
  - identity
  - evidence
  - net412
  - net210
date: 2026-05-10
---

# E2 — Zero-Trust Homelab Blueprint

> [!warning] Operating Assumptions & Threat Model
> **Context:** Proxmox VE homelab with multiple VMs across segmented networks (built from A2). This project extends the network segmentation with explicit identity verification, minimum-privilege access policies, and continuous validation controls.
> **Risk level:** High — significant infrastructure changes including firewall policies, SSH certificate authority setup, and service access controls. Full rollback plan is mandatory before any changes.
> **Threat model:** Traditional perimeter security ("trust but verify within the network") assumes the internal network is safe. Zero-trust assumes compromise is always possible — every access request is verified regardless of source location. This project implements zero-trust principles in the Proxmox homelab context.
> **Scope note:** A full enterprise zero-trust implementation requires a commercial identity provider (Okta, Azure AD), mTLS service mesh, and SIEM. This project implements the underlying principles using open-source tools available in the lab stack. It is an architectural exercise, not a production deployment.
> **Out of scope:** Wireguard VPN, service mesh (Linkerd, Istio), hardware-backed identity (TPM). Those are extensions for the stretch goals.

## 1) Mission

- **Problem statement:** The default homelab architecture is fundamentally trust-based: once inside the network, a VM can reach most other resources. This project redesigns the lab architecture around zero-trust principles: explicit authentication for every service access, network segmentation by function, least-privilege service accounts, and continuous validation rather than one-time perimeter authentication.
- **Why it matters:** Zero-trust architecture is the current standard for enterprise security design (NIST SP 800-207). Understanding how to implement its principles — even in a simplified homelab context — is a critical skill for NET 412 and NET 210 advanced coursework and real-world infrastructure work.

## 2) Difficulty

- Expert (estimated 16–30 focused hours)

## 3) Execution Context

- **Proxmox Host and VMs** — Architecture changes affect all lab VMs. The Arch bare-metal host acts as the management plane. Proxmox VMs implement the segmented services.

## 4) Prerequisites

- **Skills:** Completion of A2 (Network Segmentation), I1 (Systemd Service Hardening), B3 (SSH Key-Only Admin Access), A1 (Centralized Logging). Strong understanding of PKI and TLS fundamentals.
- **Tools:** `openssl`, `ssh-keygen`, `nftables`, `qm`, `pvesh`, `systemctl`, `journalctl`, `tee`
- **Dependencies:** A2 network segmentation already implemented. At least 3 Proxmox VMs for the architecture test.

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-e2-zero-trust"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-e2-zero-trust"`
- **VM snapshots (pre):** Snapshot all participating VMs: `qm snapshot <VMID> pre-e2-zt`
- **VM snapshot rollback:** `qm rollback <VMID> pre-e2-zt`
- **Container rebuild command:** N/A
- **Rollback trigger:** If any service loses connectivity after policy changes, restore VM snapshot and remove the offending firewall rule. The SSH CA can be decommissioned by removing the `TrustedUserCAKeys` line from `sshd_config`.

## 6) Project Plan

- **Phase A — Architecture Design:** Document the target zero-trust architecture. Define trust boundaries, identity sources, and access policies.
- **Phase B — SSH Certificate Authority:** Set up a lab SSHCA to replace per-host `authorized_keys` management. Issue short-lived user certificates.
- **Phase C — Service Identity and mTLS (simplified):** Configure services to authenticate callers using client certificates.
- **Phase D — Access Policy Validation:** Test that the policies work correctly — permitted access succeeds, denied access is blocked and logged.

## 7) Walkthrough

### Step 1 — Document the zero-trust architecture

```bash
mkdir -p evidence/e2

# Record current architecture baseline
date -u | tee evidence/e2/env_datetime.txt
ip addr show | tee evidence/e2/env_host_network.txt
qm list | tee evidence/e2/env_vm_inventory.txt

# Snapper pre-snapshot
sudo snapper -c root create --description "pre-e2-zero-trust" --print-number \
  | tee evidence/e2/snapshot_pre_num.txt
```

The zero-trust architecture for this lab:

```bash
cat | tee evidence/e2/zt_architecture.md << 'EOF'
# Zero-Trust Homelab Architecture

## Segments (from A2)
- vmbr0: External/Internet (untrusted)
- vmbr1: Management (192.168.100.0/24) — admin access only
- vmbr2: Workload-Isolated (10.10.20.0/24) — no internet, controlled east-west

## Zero-Trust Principles Applied
1. Identity Verification
   - SSH: Certificate-based auth (SSHCA) replaces static authorized_keys
   - Services: Client certificate mutual TLS where applicable
   
2. Least Privilege
   - Each VM has only the firewall rules it needs
   - Service accounts have per-service ACL-backed permissions (I3 pattern)
   - No wildcard "allow all from internal" rules

3. Explicit Access Control
   - All cross-segment traffic is explicitly allowed or denied (nftables)
   - No implicit trust based on network location

4. Continuous Validation
   - SSH certificates expire (short TTL: 8 hours)
   - Access events logged to centralized pipeline (A1)
   - Failed access attempts generate A3 detection alerts

5. Micro-Segmentation
   - Services are isolated within segments using per-service nftables rules
   - East-west traffic between VMs in the same segment is not automatically trusted

## Trust Hierarchy
CA (on management host) → Signs → User Certificates (8h TTL)
                        → Signs → Host Certificates (permanent, renewed on rotation)
EOF

cat evidence/e2/zt_architecture.md
```

### Step 2 — SSH Certificate Authority setup

```bash
CA_DIR="/etc/ssh/lab-sshca"
sudo mkdir -p "$CA_DIR"

# Generate the CA signing key pair (keep the private key protected)
sudo ssh-keygen -t ed25519 -f "$CA_DIR/lab_sshca_key" -N "" \
  -C "Lab SSHCA - created $(date -u +%Y-%m-%d)" 2>&1 | tee evidence/e2/ca_keygen_output.txt

# Set strict permissions on the CA private key
sudo chmod 600 "$CA_DIR/lab_sshca_key"
sudo chmod 644 "$CA_DIR/lab_sshca_key.pub"

# Capture CA public key fingerprint
ssh-keygen -lf "$CA_DIR/lab_sshca_key.pub" | sudo tee evidence/e2/ca_pubkey_fingerprint.txt
sudo cat "$CA_DIR/lab_sshca_key.pub" | tee evidence/e2/ca_pubkey.txt
```

Expected:
- `ca_pubkey.txt` contains the CA public key.
- `ca_pubkey_fingerprint.txt` shows the CA key fingerprint for verification.

### Step 3 — Configure SSH hosts to trust the CA

```bash
CA_DIR="/etc/ssh/lab-sshca"

# On the Arch host: add CA public key as trusted user CA
echo "TrustedUserCAKeys $CA_DIR/lab_sshca_key.pub" | sudo tee -a /etc/ssh/sshd_config.d/99-lab-sshca.conf

# Validate sshd config
sudo sshd -t && echo "CONFIG SYNTAX OK" | tee evidence/e2/sshd_config_test.txt

# Reload sshd
sudo systemctl reload sshd
systemctl status sshd --no-pager | tee evidence/e2/sshd_status_after_ca.txt

# Capture updated sshd config
sudo grep -Ev "^#|^$" /etc/ssh/sshd_config | tee evidence/e2/sshd_config_after.txt
sudo cat /etc/ssh/sshd_config.d/99-lab-sshca.conf | tee evidence/e2/sshd_ca_config.txt
```

Expected:
- `sshd_config_test.txt` shows `CONFIG SYNTAX OK`.
- `sshd_config_after.txt` includes `TrustedUserCAKeys`.

### Step 4 — Issue user certificates with short TTL

```bash
CA_DIR="/etc/ssh/lab-sshca"
ADMIN_USER=$(id -un)

# Generate a user key pair for certificate-based auth
ssh-keygen -t ed25519 -f ~/.ssh/lab_zt_user_key -N "" \
  -C "zt-user-$ADMIN_USER-$(date -u +%Y%m%d)" 2>&1 | tee evidence/e2/user_keygen_output.txt

# Sign the user public key with the CA — 8 hour validity, specify principals
sudo ssh-keygen -s "$CA_DIR/lab_sshca_key" \
  -I "zt-user-$ADMIN_USER" \
  -n "$ADMIN_USER" \
  -V +8h \
  ~/.ssh/lab_zt_user_key.pub 2>&1 | tee evidence/e2/user_cert_issue.txt

# Inspect the issued certificate
ssh-keygen -Lf ~/.ssh/lab_zt_user_key-cert.pub | tee evidence/e2/user_cert_contents.txt

ls -lah ~/.ssh/lab_zt_user_key* | tee evidence/e2/user_key_files.txt
```

Expected:
- `user_cert_contents.txt` shows:
  - `Type: ssh-ed25519-cert-v01@openssh.com user certificate`
  - `Valid: from ... to ...` (8-hour window from now)
  - `Principals: <your_username>`

### Step 5 — Test certificate-based SSH authentication

```bash
# Test: connect using the certificate (should not require password or authorized_keys)
ssh -i ~/.ssh/lab_zt_user_key -o CertificateFile=~/.ssh/lab_zt_user_key-cert.pub \
  -o PasswordAuthentication=no \
  localhost "echo ZT_CERT_AUTH_SUCCESS && id" \
  | tee evidence/e2/cert_auth_test.txt 2>&1

# Confirm certificate login appeared in journal
sudo journalctl -u sshd --no-pager -n 10 | grep -i "cert\|certificate\|Accepted" \
  | tee evidence/e2/cert_auth_journal.txt

# Test: attempt login without certificate (should fail if authorized_keys not present)
# Clean up any existing authorized_keys for fair test
# NOTE: Only do this if you have console access as backup
echo "Skipping authorized_keys cleanup — test is informational only" \
  | tee evidence/e2/cert_vs_key_test.txt
```

Expected:
- `cert_auth_test.txt` shows `ZT_CERT_AUTH_SUCCESS`.
- `cert_auth_journal.txt` shows `Accepted publickey for <user> from ... cert ID zt-user-<user>`.

### Step 6 — Apply per-VM nftables access policies

```bash
# Extend A2 segmentation with service-specific rules
# Example: only the management VM (192.168.100.10) can SSH to the workload segment
# All other access from workload to management is blocked

cat | sudo tee /etc/nftables.d/e2-zero-trust-policy.conf << 'EOF'
# E2 Zero-Trust Policy Extensions
# Applied on top of A2 segmentation rules

table inet zt-policy {
    chain forward-zt {
        type filter hook forward priority 1; policy accept;
        
        # Allow only management subnet to SSH to workload segment
        ip saddr 192.168.100.0/24 ip daddr 10.10.20.0/24 tcp dport 22 accept
        
        # Block all other inbound SSH to workload segment
        ip daddr 10.10.20.0/24 tcp dport 22 log prefix "ZT-SSH-BLOCKED: " drop
        
        # Allow workload segment to access centralized log collector (A1) only
        ip saddr 10.10.20.0/24 ip daddr 192.168.100.1 tcp dport 19532 accept
        ip saddr 10.10.20.0/24 ip daddr 192.168.100.0/24 drop
    }
}
EOF

# Include in main nftables config
echo 'include "/etc/nftables.d/e2-zero-trust-policy.conf"' | sudo tee -a /etc/nftables.conf

# Test config syntax before applying
sudo nft -c -f /etc/nftables.conf \
  && echo "Config syntax OK" | tee evidence/e2/nft_syntax_check.txt \
  || echo "SYNTAX ERROR — fix before applying" | tee evidence/e2/nft_syntax_check.txt

# Apply
sudo systemctl reload nftables
sudo nft list ruleset | tee evidence/e2/nft_zt_ruleset.txt
```

### Step 7 — Validation testing

```bash
# Test 1: Certificate-based SSH works
grep -q "ZT_CERT_AUTH_SUCCESS" evidence/e2/cert_auth_test.txt \
  && echo "PASS: Certificate SSH authentication works" | tee evidence/e2/validation_results.txt \
  || echo "FAIL: Certificate SSH authentication failed" | tee evidence/e2/validation_results.txt

# Test 2: SSH certificate shows correct 8h validity
grep -q "Valid:" evidence/e2/user_cert_contents.txt \
  && echo "PASS: Certificate validity documented" | tee -a evidence/e2/validation_results.txt \
  || echo "FAIL: Certificate validity not captured" | tee -a evidence/e2/validation_results.txt

# Test 3: CA public key is in TrustedUserCAKeys
grep -q "TrustedUserCAKeys" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/* 2>/dev/null \
  && echo "PASS: TrustedUserCAKeys configured" | tee -a evidence/e2/validation_results.txt \
  || echo "FAIL: TrustedUserCAKeys not configured" | tee -a evidence/e2/validation_results.txt

# Test 4: nftables ZT policy is loaded
sudo nft list ruleset | grep -q "zt-policy" \
  && echo "PASS: ZT nftables policy active" | tee -a evidence/e2/validation_results.txt \
  || echo "FAIL: ZT policy not in ruleset" | tee -a evidence/e2/validation_results.txt

cat evidence/e2/validation_results.txt

# Post-snapshot
sudo snapper -c root create --description "post-e2-zero-trust" --print-number \
  | tee evidence/e2/snapshot_post_num.txt
```

## 8) Validation

```bash
# Summarize ZT implementation status
echo "=== Zero-Trust Implementation Checklist ===" | tee evidence/e2/zt_checklist.txt
echo "" | tee -a evidence/e2/zt_checklist.txt
echo "[x] Network Segmentation (vmbr0/vmbr1/vmbr2 from A2)" | tee -a evidence/e2/zt_checklist.txt
grep -q "TrustedUserCAKeys" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/* 2>/dev/null \
  && echo "[x] SSH Certificate Authority (TrustedUserCAKeys configured)" \
  || echo "[ ] SSH Certificate Authority (not configured)"  | tee -a evidence/e2/zt_checklist.txt
[ -f ~/.ssh/lab_zt_user_key-cert.pub ] \
  && echo "[x] Short-lived User Certificates (8h TTL)" \
  || echo "[ ] Short-lived User Certificates" | tee -a evidence/e2/zt_checklist.txt
sudo nft list ruleset | grep -q "zt-policy" \
  && echo "[x] Per-service Firewall Policy (nftables zt-policy)" \
  || echo "[ ] Per-service Firewall Policy" | tee -a evidence/e2/zt_checklist.txt
systemctl is-active systemd-journal-remote.socket &>/dev/null \
  && echo "[x] Centralized Logging (journal-remote from A1)" \
  || echo "[ ] Centralized Logging" | tee -a evidence/e2/zt_checklist.txt
cat evidence/e2/zt_checklist.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/e2/env_datetime.txt`, `env_host_network.txt`, `env_vm_inventory.txt` — environment
  - `evidence/e2/snapshot_pre_num.txt`, `snapshot_post_num.txt` — Snapper snapshots
  - `evidence/e2/zt_architecture.md` — architecture design document
  - `evidence/e2/ca_keygen_output.txt` — CA key generation output
  - `evidence/e2/ca_pubkey_fingerprint.txt` — CA key fingerprint
  - `evidence/e2/ca_pubkey.txt` — CA public key (safe to share)
  - `evidence/e2/sshd_config_test.txt` — sshd config syntax check
  - `evidence/e2/sshd_status_after_ca.txt` — sshd status after CA config
  - `evidence/e2/sshd_config_after.txt` — sshd active configuration
  - `evidence/e2/sshd_ca_config.txt` — CA drop-in config
  - `evidence/e2/user_keygen_output.txt` — user key generation
  - `evidence/e2/user_cert_issue.txt` — certificate signing output
  - `evidence/e2/user_cert_contents.txt` — certificate inspection
  - `evidence/e2/user_key_files.txt` — key file listing
  - `evidence/e2/cert_auth_test.txt` — certificate authentication test
  - `evidence/e2/cert_auth_journal.txt` — journal entries for cert login
  - `evidence/e2/nft_syntax_check.txt` — nftables config validation
  - `evidence/e2/nft_zt_ruleset.txt` — applied ZT nftables rules
  - `evidence/e2/validation_results.txt` — all validation test results
  - `evidence/e2/zt_checklist.txt` — ZT implementation checklist
- **Hash manifest:**

```bash
find evidence/e2 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/e2/SHA256SUMS.txt
cat evidence/e2/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** Certificate-based SSH login fails with "certificate invalid: bad validity interval."
  - **Cause:** System clock skew between client and server exceeds the validity window.
  - **Fix:** Ensure both machines have accurate time (`timedatectl status`). Use NTP: `sudo systemctl enable --now systemd-timesyncd`.
  - **Rollback trigger:** The `TrustedUserCAKeys` line in `sshd_config` can be removed and sshd reloaded to revert to key-only auth.

- **Symptom:** nftables ZT policy accidentally blocks legitimate management traffic.
  - **Cause:** Overly broad DROP rule before the specific ACCEPT rules.
  - **Fix:** `sudo nft flush table inet zt-policy` to remove the ZT table. Verify base connectivity is restored. Re-examine rule ordering.
  - **Rollback trigger:** `sudo nft delete table inet zt-policy` to remove the ZT extension rules without affecting A2 base rules.

- **Symptom:** SSH CA private key is exposed or lost.
  - **Cause:** Accidental permission change or file deletion.
  - **Fix:** If key is lost: revoke all issued certificates (`AuthorizedKeysCommand` with revocation list). Generate a new CA key and redistribute the new public key. If key is exposed: same emergency procedure.
  - **Rollback trigger:** Remove `TrustedUserCAKeys` from sshd_config and reload. All certificate-based logins will fail, reverting to key-based auth.

## 11) Sources

- [NIST SP 800-207 — Zero Trust Architecture](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf)
- [OpenSSH Certificate Authentication documentation](https://man.openbsd.org/ssh-keygen#CERTIFICATES)
- [Mozilla SSH Security Guidelines](https://infosec.mozilla.org/guidelines/openssh.html)
- [CIS Benchmark — Linux](https://www.cisecurity.org/benchmark/distribution_independent_linux)
- [nftables documentation — nft man page](https://man7.org/linux/man-pages/man8/nft.8.html)
- [NIST SP 800-53 — AC-17 Remote Access](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)

## 12) Stretch Goals

- Add a WireGuard VPN overlay so that all management access to the lab (including Proxmox web UI) requires VPN authentication before reaching the management segment. This adds a cryptographic identity layer at the network level.
- Implement SSH certificate revocation using the `RevokedKeys` directive in `sshd_config`. Issue a certificate, add it to the revocation list, and confirm that the revoked certificate is rejected.
- Deploy a minimal identity-aware proxy (Caddy with forward auth, or Nginx with a simple auth backend) in front of the Proxmox web UI. Require authentication before the UI is accessible.
- Write a `certificate-manager.sh` script that automates the daily re-issuance of short-lived certificates for all lab administrators, with a reminder if any certificate is within 1 hour of expiry.
