---
title: B3 - SSH Key-Only Admin Access
aliases:
  - b3-ssh-key-only
tags:
  - beginner
  - ssh
  - hardening
  - authentication
  - evidence
date: 2026-05-10
---

# B3 — SSH Key-Only Admin Access

## 1) Mission

- **Problem statement:** Password-based SSH authentication is vulnerable to brute-force and credential-stuffing attacks. Many administrators know they should use key-only SSH but have never drilled the safe migration procedure, including how to verify it works and how to roll back if it breaks.
- **Why it matters:** Disabling password authentication in `sshd_config` without first confirming your key is accepted will lock you out of the system. This project teaches the correct order of operations and how to validate each step before making the change permanent.

---

## 2) Difficulty

**Beginner** — Medium risk. Misconfiguration of `sshd` can lock you out of a remote host. Follow the rollback plan and validate each step before proceeding to the next.

---

## 3) Execution Context

- **Primary path:** VM — Proxmox lab VM (recommended; easy Proxmox console fallback if locked out)
- **Alternate path:** Host — Arch Linux bare metal (requires physical console or KVM-over-IP access as fallback)
- Do **not** run this project on a shared or production SSH server without proper change management.

> [!warning] Lockout risk
> Always confirm key-based login works **before** disabling password authentication.
> Keep a separate terminal session open during the transition so you have a live fallback if the new session fails.

---

## 4) Prerequisites

### Skills
- Basic SSH usage (connect to a host, understand `~/.ssh/` directory)
- Ability to edit a file with a terminal text editor (`nano`, `vim`, or `micro`)
- Understanding of file permissions

### Tools
```bash
# Verify ssh-keygen and ssh-copy-id are available (openssh package)
ssh-keygen --help 2>&1 | head -3
ssh-copy-id 2>&1 | head -3

# Confirm sshd is installed and running on the target
systemctl status sshd || systemctl status ssh
```

### Lab Environment
- A lab VM or host running `sshd` that you fully own and control
- SSH network access from your workstation to the target
- `sudo` privileges on the target system
- A Proxmox console or physical console available as a lockout fallback

---

## 5) Rollback Plan

> [!warning] Create your snapshot BEFORE editing sshd_config

### VM path (Proxmox) — recommended
```bash
# Run on the Proxmox node; replace <VMID> with your VM's numeric ID
VMID=<VMID>
qm snapshot "$VMID" pre-b3-ssh --description "B3 pre-SSH-hardening snapshot"
qm listsnapshot "$VMID"
```

### Host path (Snapper)
```bash
sudo snapper -c root create --description "pre-b3-ssh"
snapper -c root list | tail -n 5
```

### Emergency rollback — if locked out
1. Use the Proxmox console (or physical console) to log in directly
2. Restore the snapshot: `qm rollback <VMID> pre-b3-ssh` (from Proxmox node), or
3. Restore `sshd_config` from backup:
   ```bash
   sudo cp /etc/ssh/sshd_config.b3.bak /etc/ssh/sshd_config
   sudo systemctl restart sshd
   ```

### Rollback trigger conditions
- Cannot authenticate with the new SSH key
- `sshd` fails to restart after config change
- Lost access to the terminal session

---

## 6) Project Plan

- **Phase A:** Initialise evidence directory and create pre-change snapshot
- **Phase B:** Generate SSH key pair (if one does not already exist)
- **Phase C:** Deploy the public key to the target host and verify key-based login
- **Phase D:** Harden `sshd_config` to disable password authentication
- **Phase E:** Validate the hardened configuration
- **Phase F:** Generate hash manifest and complete report checklist

---

## 7) Walkthrough

### Step 1 — Initialise evidence directory

Run on your **workstation** (the machine you will SSH _from_):

```bash
TS="$(date -u +%Y%m%d_%H%M%SZ)"
OUT="evidence/b3/${TS}"
mkdir -p "$OUT"
echo "Evidence path: $OUT"
export OUT
```

Expected:
- Directory created; `$OUT` set

---

### Step 2 — Generate an SSH key pair

> [!info] Skip if you already have a suitable key pair
> If you already have an ed25519 key at `~/.ssh/id_ed25519`, review it with `ssh-keygen -l -f ~/.ssh/id_ed25519` and skip to Step 3.

```bash
# Generate an Ed25519 key (preferred over RSA for new keys)
ssh-keygen -t ed25519 -C "b3-lab-$(date -u +%Y%m%d)" -f ~/.ssh/id_ed25519_b3lab

# Record the public key fingerprint as evidence
ssh-keygen -l -f ~/.ssh/id_ed25519_b3lab.pub | tee "$OUT/key_fingerprint.txt"

# Capture the public key content (safe to record — not the private key)
cat ~/.ssh/id_ed25519_b3lab.pub | tee "$OUT/public_key.txt"
```

Expected:
- Key pair generated at `~/.ssh/id_ed25519_b3lab` and `~/.ssh/id_ed25519_b3lab.pub`
- Fingerprint captured in evidence
- `public_key.txt` contains the single-line public key (begins with `ssh-ed25519`)

> [!warning]
> Never capture or commit the private key (`id_ed25519_b3lab`, no `.pub` extension). Only the public key (`.pub`) is safe to record.

---

### Step 3 — Deploy the public key to the target host

```bash
TARGET_HOST="<lab-vm-hostname-or-ip>"
TARGET_USER="<your-admin-username>"

# Copy the public key using ssh-copy-id
ssh-copy-id -i ~/.ssh/id_ed25519_b3lab.pub "${TARGET_USER}@${TARGET_HOST}"
```

Expected:
- `ssh-copy-id` reports "1 key(s) added"
- The key is appended to `~/.ssh/authorized_keys` on the target

Verify the key is present on the target:
```bash
ssh -i ~/.ssh/id_ed25519_b3lab "${TARGET_USER}@${TARGET_HOST}" \
  "cat ~/.ssh/authorized_keys" | tee "$OUT/authorized_keys_snapshot.txt"
```

Expected:
- Your new public key appears in the output

---

### Step 4 — Confirm key-based login works BEFORE changing sshd_config

> [!warning] Critical step — do not skip
> Open a **new** terminal and verify key-based login succeeds before proceeding.

```bash
# Test key-based login (should succeed without a password prompt)
ssh -i ~/.ssh/id_ed25519_b3lab -o PasswordAuthentication=no \
  "${TARGET_USER}@${TARGET_HOST}" "echo 'Key login: OK'" \
  | tee "$OUT/key_login_test.txt"
```

Expected:
- Output: `Key login: OK`
- No password prompt

If this step fails, **stop here**. Debug the key deployment before proceeding.

---

### Step 5 — Back up sshd_config and harden it

Run on the **target host**:

```bash
# Backup the original sshd_config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.b3.bak

# Capture original effective config as evidence
sudo grep -v '^\s*#' /etc/ssh/sshd_config | grep -v '^\s*$' \
  | tee /tmp/sshd_config_before.txt
# Copy to workstation evidence directory
scp -i ~/.ssh/id_ed25519_b3lab \
  "${TARGET_USER}@${TARGET_HOST}:/tmp/sshd_config_before.txt" \
  "$OUT/sshd_config_before.txt"
```

Apply the hardening changes on the target:

```bash
ssh -i ~/.ssh/id_ed25519_b3lab "${TARGET_USER}@${TARGET_HOST}" bash <<'ENDSSH'
  # Disable password authentication
  sudo sed -i 's/^#*\s*PasswordAuthentication.*/PasswordAuthentication no/' \
    /etc/ssh/sshd_config

  # Disable empty passwords
  sudo sed -i 's/^#*\s*PermitEmptyPasswords.*/PermitEmptyPasswords no/' \
    /etc/ssh/sshd_config

  # Disable root login (use a named admin account instead)
  sudo sed -i 's/^#*\s*PermitRootLogin.*/PermitRootLogin no/' \
    /etc/ssh/sshd_config

  # Verify the changes are in the file before restarting
  grep -E 'PasswordAuthentication|PermitEmptyPasswords|PermitRootLogin' \
    /etc/ssh/sshd_config
ENDSSH
```

Expected:
- Three lines showing `no` values for each directive

Capture the updated config:
```bash
ssh -i ~/.ssh/id_ed25519_b3lab "${TARGET_USER}@${TARGET_HOST}" \
  "sudo grep -v '^\s*#' /etc/ssh/sshd_config | grep -v '^\s*\$'" \
  | tee "$OUT/sshd_config_after.txt"
```

---

### Step 6 — Test the configuration and restart sshd

```bash
# Validate sshd config syntax BEFORE restarting
ssh -i ~/.ssh/id_ed25519_b3lab "${TARGET_USER}@${TARGET_HOST}" \
  "sudo sshd -t && echo 'Config syntax: OK'" | tee "$OUT/sshd_syntax_check.txt"
```

Expected:
- `Config syntax: OK`
- If errors are reported, fix them before proceeding

```bash
# Restart sshd
ssh -i ~/.ssh/id_ed25519_b3lab "${TARGET_USER}@${TARGET_HOST}" \
  "sudo systemctl restart sshd && systemctl is-active sshd" \
  | tee "$OUT/sshd_restart_result.txt"
```

Expected:
- Output: `active`

> [!tip] Keep your existing session open
> Do not close your current SSH session until you verify the new session works in Step 7.

---

### Step 7 — Validate: key login works, password login is rejected

Open a **new** terminal for each test:

```bash
# Test 1: Key-based login should succeed
ssh -i ~/.ssh/id_ed25519_b3lab -o PasswordAuthentication=no \
  "${TARGET_USER}@${TARGET_HOST}" "echo 'Key login post-hardening: OK'" \
  | tee "$OUT/validation_key_login.txt"

# Test 2: Password-based login should be rejected
ssh -o PubkeyAuthentication=no -o PasswordAuthentication=yes \
  "${TARGET_USER}@${TARGET_HOST}" "echo 'Should not reach here'" 2>&1 \
  | tee "$OUT/validation_password_rejected.txt"
```

Expected:
- Test 1: `Key login post-hardening: OK`
- Test 2: Connection refused or `Permission denied (publickey)` — password not accepted

Capture the `sshd` journal for evidence:
```bash
ssh -i ~/.ssh/id_ed25519_b3lab "${TARGET_USER}@${TARGET_HOST}" \
  "sudo journalctl -u sshd --since '10 minutes ago' --no-pager -o short-iso" \
  | tee "$OUT/sshd_journal_post_hardening.txt"
```

---

## 8) Validation

```bash
# Confirm key login works
grep -q "Key login post-hardening: OK" "$OUT/validation_key_login.txt" \
  && echo "PASS: Key login confirmed" || echo "FAIL: Key login not confirmed"

# Confirm password login is rejected
grep -qiE "denied|refused|keyboard" "$OUT/validation_password_rejected.txt" \
  && echo "PASS: Password login rejected" || echo "FAIL: Password login not rejected"

# Confirm sshd config has PasswordAuthentication no
grep -q "PasswordAuthentication no" "$OUT/sshd_config_after.txt" \
  && echo "PASS: PasswordAuthentication disabled" \
  || echo "FAIL: PasswordAuthentication not disabled"

# Confirm all evidence files exist
for f in \
  key_fingerprint.txt \
  public_key.txt \
  sshd_config_before.txt \
  sshd_config_after.txt \
  sshd_syntax_check.txt \
  sshd_restart_result.txt \
  validation_key_login.txt \
  validation_password_rejected.txt \
  sshd_journal_post_hardening.txt; do
  [ -f "$OUT/$f" ] && echo "PASS: $f" || echo "FAIL: $f missing"
done
```

Expected:
- All `PASS` lines

---

## 9) Evidence

### Files to capture
- `evidence/b3/<TS>/key_fingerprint.txt`
- `evidence/b3/<TS>/public_key.txt` (**public key only — never the private key**)
- `evidence/b3/<TS>/authorized_keys_snapshot.txt`
- `evidence/b3/<TS>/sshd_config_before.txt`
- `evidence/b3/<TS>/sshd_config_after.txt`
- `evidence/b3/<TS>/sshd_syntax_check.txt`
- `evidence/b3/<TS>/sshd_restart_result.txt`
- `evidence/b3/<TS>/validation_key_login.txt`
- `evidence/b3/<TS>/validation_password_rejected.txt`
- `evidence/b3/<TS>/sshd_journal_post_hardening.txt`

> [!warning] Before committing to git
> - Replace `TARGET_HOST` values with a placeholder (e.g., `lab-vm-01`)
> - Replace `TARGET_USER` with a placeholder (e.g., `admin`)
> - Do **not** include the private key file path or passphrase anywhere
> - Exclude the `evidence/` directory from git if it contains real hostnames or IPs

### Generate hash manifest
```bash
find "$OUT" -type f ! -name 'SHA256SUMS.txt' -print0 \
  | xargs -0 sha256sum | tee "$OUT/SHA256SUMS.txt"

sha256sum --check "$OUT/SHA256SUMS.txt" && echo "Manifest verified OK"
```

---

## 10) Failure Modes & Recovery

| Symptom | Likely Cause | Fix | Rollback Trigger |
|---------|-------------|-----|-----------------|
| `ssh-copy-id` fails with "Permission denied" | Password auth already disabled or wrong user | Use Proxmox console to manually append key to `authorized_keys` | — |
| Key login test (Step 4) fails | Wrong key file, wrong target user, or `authorized_keys` permissions | Check `~/.ssh/authorized_keys` permissions (`chmod 600`); check `~/.ssh/` is `700` | Stop here; do not proceed to Step 5 |
| `sshd -t` reports config errors | Syntax error in `sshd_config` | Review the error line; restore from `.b3.bak` backup | Restore backup and restart sshd |
| `systemctl restart sshd` fails | Invalid config or dependency issue | `journalctl -u sshd -n 50`; restore `.b3.bak` | Restore backup |
| Locked out after restart | Residual bug or untested edge case | Use Proxmox console or physical console; restore `.b3.bak`; or roll back VM snapshot | Roll back VM snapshot `pre-b3-ssh` |
| Password login still works after hardening | Multiple `PasswordAuthentication` lines in config | `grep -n PasswordAuthentication /etc/ssh/sshd_config`; remove duplicates | — |

---

## 11) Sources

- [SRC-OPENSSH-01] — OpenSSH manual — https://www.openssh.com/manual.html
- [SRC-ARCH-SSH-01] — Arch Wiki: SSH keys — https://wiki.archlinux.org/title/SSH_keys
- [SRC-MOZ-SSH-01] — Mozilla SSH Guidelines — https://infosec.mozilla.org/guidelines/openssh

---

## 12) Stretch Goals

- Add `AllowUsers` or `AllowGroups` directive to restrict which accounts can SSH in
- Configure `Match` blocks to allow password auth from localhost only (useful for emergency console access)
- Enable `Banner` to display a lab-use warning message to SSH clients
- Implement `fail2ban` or `sshguard` to rate-limit failed login attempts
- Map this hardening to CIS Benchmark controls for SSH (Section 5 in most Linux CIS profiles)

---

## Report Checklist

Fill in before marking B3 complete:

- [ ] Execution context declared: _____________________ (VM / Host)
- [ ] Pre-change snapshot created — Name: _____________________
- [ ] SSH key pair generated (or existing key confirmed): key type _____________________
- [ ] Public key deployed to target and `authorized_keys` verified
- [ ] Key-based login confirmed working BEFORE sshd_config change
- [ ] `sshd_config` backed up to `/etc/ssh/sshd_config.b3.bak`
- [ ] `sshd_config` hardened: `PasswordAuthentication no`, `PermitEmptyPasswords no`, `PermitRootLogin no`
- [ ] `sshd -t` syntax check passed
- [ ] `sshd` restarted successfully
- [ ] Post-hardening key login: PASS / FAIL
- [ ] Post-hardening password login rejected: PASS / FAIL
- [ ] All validation checks output `PASS`
- [ ] `SHA256SUMS.txt` generated and verified
- [ ] Evidence reviewed for sensitive data (real hostnames, IPs, usernames)
- [ ] UTC start time: _____________________ — UTC end time: _____________________
