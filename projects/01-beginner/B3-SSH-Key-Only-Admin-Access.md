---
title: B3 - SSH Key-Only Admin Access
aliases:
  - b3-ssh-key-only
  - ssh-hardening-beginner
tags:
  - beginner
  - ssh
  - hardening
  - authentication
  - evidence
  - net412
  - net210
date: 2026-05-10
---

# B3 — SSH Key-Only Admin Access

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host acting as SSH server. A second machine (or a Proxmox VM on the same host) acts as the SSH client for testing.
> **Risk level:** Medium — disabling password authentication locks out anyone without a valid key. **Always confirm key-based login works in a separate terminal session before closing the current session.**
> **Threat model:** Password-based SSH authentication is vulnerable to brute-force and credential-stuffing attacks. Key-only access eliminates that entire attack surface. This project also documents the admin access path so it can be audited and recovered if keys are lost.
> **Hard rule:** Do NOT close your existing SSH session or terminal until you have verified key-based login from a separate terminal. Opening two side-by-side terminals is mandatory for Step 6.
> **Out of scope:** Certificate-based SSH (SSHCA), multi-factor SSH, jump hosts. Those are covered in advanced networking modules.

## 1) Mission

- **Problem statement:** Default SSH installations on many systems permit password authentication, making the SSH port a high-value brute-force target. Additionally, administrators who have not explicitly documented their key setup often find themselves locked out after a configuration error.
- **Why it matters:** SSH key hygiene is a fundamental NET 412 and NET 210 competency. A hardened SSH config with key-only access is also the entry point for all remote Proxmox administration. Getting this right creates a secure, auditable admin access channel.

## 2) Difficulty

- Beginner (estimated 2–4 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal as SSH server (`sshd`). SSH client can be the same machine (loopback), a Proxmox VM, or any client machine on the lab network.

## 4) Prerequisites

- **Skills:** Basic terminal usage, understanding of public/private key cryptography concepts (you generate a key pair; public key goes on server, private key stays on client).
- **Tools:** `ssh-keygen`, `ssh-copy-id`, `ssh`, `sshd`, `systemctl`, `journalctl`, `grep`, `tee`
- **Dependencies:**
  - `openssh` package installed and `sshd` running: `pacman -Q openssh && systemctl status sshd`
  - At least two terminal windows (or tmux panes) available simultaneously.
  - A non-root user account to test with (do not test root SSH login directly).

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-b3-ssh-hardening"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-b3-ssh-hardening"`
- **VM snapshot:** If testing against a VM, take a Proxmox snapshot before modifying `sshd_config` inside the VM.
- **Container rebuild command:** N/A
- **Emergency rollback:** If locked out of SSH, access the machine physically (console) and run:
  ```bash
  sudo snapper -c root undochange <PRE_NUM>..<POST_NUM>
  sudo systemctl restart sshd
  ```
  Alternatively, manually restore: `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && sudo systemctl restart sshd`

## 6) Project Plan

- **Phase A — Baseline Capture:** Document current SSH config and authentication methods before any changes.
- **Phase B — Key Generation and Deployment:** Generate an Ed25519 key pair and install the public key on the server. Verify key-based login works.
- **Phase C — Configuration Hardening:** Disable password authentication and other weak options in `sshd_config`. Validate with `ssh-audit` or manual connection tests.

## 7) Walkthrough

### Step 1 — Capture pre-change SSH configuration baseline

```bash
mkdir -p evidence/b3

# Record current sshd_config (all active, non-comment lines)
sudo grep -Ev "^#|^$" /etc/ssh/sshd_config | tee evidence/b3/sshd_config_before.txt

# Backup the original config before any changes
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sha256sum /etc/ssh/sshd_config | tee evidence/b3/sshd_config_hash_before.txt

# Record sshd service status
systemctl status sshd --no-pager | tee evidence/b3/sshd_status_before.txt

# Record sshd version and compiled options
sshd -V 2>&1 | tee evidence/b3/sshd_version.txt

# List existing authorized_keys files across all users
sudo find /home /root -name "authorized_keys" 2>/dev/null | tee evidence/b3/existing_authorized_keys.txt

# Record current listening ports and sockets for sshd
ss -tlnp | grep sshd | tee evidence/b3/sshd_listening_before.txt
```

Expected:
- `sshd_config_before.txt` shows current active SSH configuration — note whether `PasswordAuthentication` is set.
- `sshd_status_before.txt` shows `active (running)`.
- `sshd_listening_before.txt` shows `sshd` listening on port 22 (or custom port).

### Step 2 — Create Snapper pre-change snapshot

```bash
sudo snapper -c root create --description "pre-b3-ssh-hardening" --print-number \
  | tee evidence/b3/snapshot_pre_num.txt

echo "Pre-change snapshot: $(cat evidence/b3/snapshot_pre_num.txt)"
```

Expected:
- A snapshot number is printed and saved. Record this number — you will need it if rollback is required.

### Step 3 — Generate Ed25519 SSH key pair (on the client machine)

> [!tip] Run this step on the machine you will SSH *from* (the client). If testing loopback (client = server), run this as your normal user.

```bash
# Generate Ed25519 key pair with a strong passphrase
# Replace "b3-admin-key" with a descriptive name for this key
ssh-keygen -t ed25519 -C "b3-admin-$(hostname)-$(date -u +%Y%m%d)" \
  -f ~/.ssh/b3_admin_ed25519

# Verify key files were created
ls -lah ~/.ssh/b3_admin_ed25519 ~/.ssh/b3_admin_ed25519.pub

# Display the public key — this is what goes on the server
cat ~/.ssh/b3_admin_ed25519.pub | tee evidence/b3/client_pubkey_deployed.txt
```

Expected:
- `~/.ssh/b3_admin_ed25519` — private key (permissions should be `600`).
- `~/.ssh/b3_admin_ed25519.pub` — public key (safe to share / place on server).
- Public key format: `ssh-ed25519 AAAA... b3-admin-<hostname>-<date>`

> [!danger] Never share or commit the private key file (`b3_admin_ed25519`). It must remain on the client only. The evidence file `client_pubkey_deployed.txt` contains only the public key and is safe.

### Step 4 — Install public key on the server

```bash
# Method A: ssh-copy-id (preferred — handles permissions automatically)
# Replace <SERVER_IP> and <USERNAME> with your values
ssh-copy-id -i ~/.ssh/b3_admin_ed25519.pub <USERNAME>@<SERVER_IP>

# Method B: Manual install (if ssh-copy-id is unavailable)
# Run this on the server as the target user
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat >> ~/.ssh/authorized_keys << 'EOF'
<paste public key here>
EOF
chmod 600 ~/.ssh/authorized_keys
```

After installing, capture evidence:

```bash
# On the server: confirm authorized_keys contents and permissions
ls -lah ~/.ssh/authorized_keys | tee evidence/b3/authorized_keys_stat.txt
cat ~/.ssh/authorized_keys | tee evidence/b3/authorized_keys_contents.txt
```

Expected:
- `authorized_keys` permissions: `600` (`-rw-------`).
- `~/.ssh/` directory permissions: `700` (`drwx------`).
- `authorized_keys_contents.txt` shows the public key you just deployed.

### Step 5 — Test key-based authentication (BEFORE disabling passwords)

> [!danger] Do NOT proceed to Step 6 until key-based login is confirmed working here. This is your safety gate.

```bash
# Test login with explicit key file — should NOT prompt for account password
ssh -i ~/.ssh/b3_admin_ed25519 -o PasswordAuthentication=no \
  <USERNAME>@<SERVER_IP> "echo KEY_AUTH_SUCCESS && id && hostname"

# Save test result
ssh -i ~/.ssh/b3_admin_ed25519 -o PasswordAuthentication=no \
  <USERNAME>@<SERVER_IP> "echo KEY_AUTH_SUCCESS && id && hostname" \
  | tee evidence/b3/key_auth_test_before_hardening.txt 2>&1
```

Expected:
- Output: `KEY_AUTH_SUCCESS` followed by `uid=...` and hostname.
- No password prompt appears.
- Exit code 0: `echo $?` returns `0`.

If this step fails, debug before proceeding:
```bash
# Verbose SSH for debugging
ssh -vvv -i ~/.ssh/b3_admin_ed25519 <USERNAME>@<SERVER_IP> 2>&1 | tee evidence/b3/ssh_debug_output.txt
```

### Step 6 — Harden sshd_config (in a second terminal — leave first terminal open)

> [!warning] Open a second terminal window before making any changes. Keep your existing session alive as a fallback.

```bash
# On the server — create hardened sshd_config entries
# Edit /etc/ssh/sshd_config using your preferred editor (nano, vim)
sudo nano /etc/ssh/sshd_config
```

Apply these hardening settings (add or modify existing lines):

```ini
# Disable password and keyboard-interactive authentication
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

# Disable root login
PermitRootLogin no

# Limit authentication attempts
MaxAuthTries 3

# Limit login grace period
LoginGraceTime 30

# Disable empty passwords
PermitEmptyPasswords no

# Allow only specific users (replace with your actual username)
AllowUsers <USERNAME>

# Use only modern key exchange algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

# Use only strong MACs
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,umac-128-etm@openssh.com

# Use only strong ciphers
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
```

```bash
# Validate configuration syntax before restarting
sudo sshd -t && echo "CONFIG SYNTAX OK" | tee evidence/b3/sshd_config_test.txt

# Capture the new config for evidence
sudo grep -Ev "^#|^$" /etc/ssh/sshd_config | tee evidence/b3/sshd_config_after.txt
sha256sum /etc/ssh/sshd_config | tee evidence/b3/sshd_config_hash_after.txt

# Restart sshd to apply changes
sudo systemctl restart sshd
systemctl status sshd --no-pager | tee evidence/b3/sshd_status_after.txt
```

Expected:
- `sshd -t` exits 0 and prints `CONFIG SYNTAX OK`.
- `sshd_config_after.txt` shows `PasswordAuthentication no` and `PermitRootLogin no`.
- `sshd_status_after.txt` shows `active (running)`.

### Step 7 — Validate: key login succeeds, password login fails

```bash
# Test 1: Key-based login still works
ssh -i ~/.ssh/b3_admin_ed25519 -o PasswordAuthentication=no \
  <USERNAME>@<SERVER_IP> "echo POST_HARDENING_KEY_SUCCESS && id" \
  | tee evidence/b3/key_auth_test_after_hardening.txt

# Test 2: Attempt password login — should be rejected
# (The -o PasswordAuthentication=yes forces the client to try password)
ssh -o PasswordAuthentication=yes -o PubkeyAuthentication=no \
  <USERNAME>@<SERVER_IP> "echo SHOULD_NOT_APPEAR" 2>&1 \
  | tee evidence/b3/password_auth_rejection_test.txt || echo "Password auth correctly rejected"

# Test 3: Capture failed auth attempt in sshd journal
sudo journalctl -u sshd --no-pager -n 30 | tee evidence/b3/sshd_journal_post_test.txt
```

Expected:
- Test 1: `POST_HARDENING_KEY_SUCCESS` printed — key auth works.
- Test 2: Connection is rejected (exit code non-zero). Journal shows `Authentication refused: not listed in AllowUsers` or `Permission denied (publickey)`.
- Test 3: Journal confirms the rejection event with client IP and username.

## 8) Validation

```bash
# Check PasswordAuthentication is explicitly disabled
grep "PasswordAuthentication no" /etc/ssh/sshd_config \
  && echo "PASS: PasswordAuthentication disabled" \
  || echo "FAIL: PasswordAuthentication not disabled"

# Check PermitRootLogin is disabled
grep "PermitRootLogin no" /etc/ssh/sshd_config \
  && echo "PASS: PermitRootLogin disabled" \
  || echo "FAIL: PermitRootLogin not disabled"

# Check sshd is still running
systemctl is-active sshd \
  && echo "PASS: sshd is active" \
  || echo "FAIL: sshd is not running"

# Check key auth test succeeded
grep "POST_HARDENING_KEY_SUCCESS" evidence/b3/key_auth_test_after_hardening.txt \
  && echo "PASS: Key auth works post-hardening" \
  || echo "FAIL: Key auth failed post-hardening"

# Check password auth was rejected
grep -i "denied\|refused\|not allowed" evidence/b3/password_auth_rejection_test.txt \
  && echo "PASS: Password auth rejected" \
  || echo "REVIEW: Password rejection not confirmed — check evidence/b3/password_auth_rejection_test.txt"
```

## 9) Evidence

- **Output files:**
  - `evidence/b3/sshd_config_before.txt` — original active SSH config
  - `evidence/b3/sshd_config_hash_before.txt` — SHA-256 of original config
  - `evidence/b3/sshd_status_before.txt` — sshd service status pre-change
  - `evidence/b3/sshd_version.txt` — OpenSSH version
  - `evidence/b3/existing_authorized_keys.txt` — pre-existing key locations
  - `evidence/b3/sshd_listening_before.txt` — listening port pre-change
  - `evidence/b3/snapshot_pre_num.txt` — Snapper pre-change snapshot number
  - `evidence/b3/client_pubkey_deployed.txt` — public key deployed to server
  - `evidence/b3/authorized_keys_stat.txt` — authorized_keys permissions
  - `evidence/b3/authorized_keys_contents.txt` — authorized_keys contents
  - `evidence/b3/key_auth_test_before_hardening.txt` — pre-hardening key auth test
  - `evidence/b3/sshd_config_test.txt` — syntax validation result
  - `evidence/b3/sshd_config_after.txt` — hardened active SSH config
  - `evidence/b3/sshd_config_hash_after.txt` — SHA-256 of hardened config
  - `evidence/b3/sshd_status_after.txt` — sshd service status post-change
  - `evidence/b3/key_auth_test_after_hardening.txt` — post-hardening key auth test
  - `evidence/b3/password_auth_rejection_test.txt` — password rejection proof
  - `evidence/b3/sshd_journal_post_test.txt` — journal entries showing rejection event
- **Hash manifest:**

```bash
find evidence/b3 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/b3/SHA256SUMS.txt
cat evidence/b3/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** SSH connection hangs or times out after restarting sshd.
  - **Cause:** Firewall rule blocking port 22, or sshd crashed due to config error.
  - **Fix (physical console access):** Check sshd status: `sudo systemctl status sshd`. If failed, restore config: `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && sudo systemctl restart sshd`
  - **Rollback trigger:** Config syntax error detected by `sshd -t` — do not restart sshd if syntax check fails.

- **Symptom:** Key auth test fails with "Permission denied (publickey)."
  - **Cause:** Public key not in `authorized_keys`, wrong key file specified, or `authorized_keys` has wrong permissions.
  - **Fix:** Verify permissions: `ls -lah ~/.ssh/ ~/.ssh/authorized_keys` — must be `700` and `600` respectively. Check key: `ssh-keygen -lf ~/.ssh/b3_admin_ed25519.pub` and compare fingerprint with `cat ~/.ssh/authorized_keys`.
  - **Rollback trigger:** N/A — password auth is still enabled at this stage.

- **Symptom:** Locked out — password auth disabled but key auth not working.
  - **Cause:** Premature `PasswordAuthentication no` before key auth was verified.
  - **Fix:** Physical console → `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && sudo systemctl restart sshd`
  - **Rollback trigger:** Immediate — restore original config and re-test key auth before re-applying hardening.

## 11) Sources

- [OpenSSH project — sshd_config man page](https://man.openbsd.org/sshd_config)
- [Arch Wiki — SSH keys](https://wiki.archlinux.org/title/SSH_keys)
- [Mozilla InfoSec — Modern SSH recommendations](https://infosec.mozilla.org/guidelines/openssh.html)
- [NIST SP 800-53 — AC-17 Remote Access](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [CIS Benchmark — Linux SSH hardening section](https://www.cisecurity.org/benchmark/distribution_independent_linux)
- [ssh-audit tool — SSH server auditing](https://github.com/jtesta/ssh-audit)

## 12) Stretch Goals

- Run `ssh-audit <SERVER_IP>` and document which algorithm recommendations it flags. Apply its recommended `sshd_config` settings and re-run until all high-severity findings are resolved.
- Configure `fail2ban` or `sshguard` to ban IPs after repeated failed SSH auth attempts. Simulate repeated failures from a second terminal and verify the ban takes effect.
- Set up SSH certificate authority (SSHCA) for the Proxmox lab: generate a CA key, sign a user certificate, configure the server to trust the CA. This replaces per-host `authorized_keys` management.
- Write a compliance check script (`b3-ssh-audit.sh`) that reads `sshd_config` and reports pass/fail for each hardening requirement.
