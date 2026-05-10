---
title: I3 - ACL-Backed Secret Store
aliases:
  - i3-acl-secret-store
  - acl-permissions-intermediate
tags:
  - intermediate
  - acl
  - setfacl
  - permissions
  - least-privilege
  - evidence
  - net412
  - net210
date: 2026-05-10
---

# I3 — ACL-Backed Secret Store

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host. A dedicated non-privileged service user account is created. ACLs are applied to a secrets directory to enforce fine-grained access control beyond traditional DAC (Discretionary Access Control).
> **Risk level:** Medium — user accounts and filesystem ACLs are modified. A misconfigured ACL could grant unintended access. Snapper pre-snapshot is mandatory.
> **Threat model:** Applications that store secrets (API keys, database passwords, TLS private keys) in world-readable or group-readable files are vulnerable to credential theft by any process or user on the system. This project enforces the principle of least privilege: only the designated service identity can read the secret, and all other access is explicitly denied. The pattern also creates an auditable access log.
> **Out of scope:** Secret management platforms (Vault, SOPS, age). Kernel keyring storage. Those provide additional capabilities beyond what DAC+ACL offers, but ACL-level isolation is the prerequisite understanding.
> **Sensitive data note:** The "secrets" used in this project are dummy placeholder values only. Never store real API keys, passwords, or private keys in a git-tracked directory.

## 1) Mission

- **Problem statement:** Most Linux file permissions are managed with the traditional Unix DAC model (owner/group/world). But when multiple services need different levels of access to the same file, or when you need to grant access to a specific user without adding them to a group, POSIX ACLs (Access Control Lists) are the correct tool. This project builds a minimal, auditable secret store using `setfacl`/`getfacl` with real service identity isolation.
- **Why it matters:** ACL-backed access control is a NET 412 and NET 210 competency. The pattern is used in every real-world service deployment where multiple applications share a server. Understanding it is also necessary before studying SELinux/AppArmor (MLS labels build on top of DAC+ACL).

## 2) Difficulty

- Intermediate (estimated 3–5 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal. All changes are to the local filesystem and user account database.

## 4) Prerequisites

- **Skills:** Completion of B2 (FHS Artifact Hunt) — familiarity with filesystem paths. Understanding of Linux user/group model (UID/GID, `/etc/passwd`, `/etc/shadow`, `sudo`).
- **Tools:** `setfacl`, `getfacl`, `chmod`, `chown`, `useradd`, `su`, `id`, `stat`, `acl` (package), `tee`
- **Dependencies:**
  - `acl` package installed: `pacman -Q acl`
  - Filesystem mounted with `acl` option (Btrfs supports ACLs natively; verify with `mount | grep acl` or check that `getfacl` works on a test file)
  - `sudo` access.

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-i3-acl-secret-store"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-i3-acl-secret-store"`
- **VM snapshot:** N/A
- **Container rebuild command:** N/A
- **Rollback trigger:** If the secret store directory or service user causes issues: `sudo snapper -c root undochange <PRE>..<POST>`. Then manually delete the test user: `sudo userdel -r svc-labsecret 2>/dev/null`.

## 6) Project Plan

- **Phase A — Service User Creation:** Create a dedicated non-login service account for the secret consumer. Confirm its identity isolation.
- **Phase B — Secret Store Setup:** Create the secret directory, apply restrictive base permissions, then layer ACL entries for the service user.
- **Phase C — Access Validation:** Prove that only the service user can read the secrets, and all other non-root users are denied. Capture ACL state as evidence.

## 7) Walkthrough

### Step 1 — Setup and pre-snapshot

```bash
mkdir -p evidence/i3

# Capture environment
id | tee evidence/i3/env_invoking_user.txt
date -u | tee evidence/i3/env_datetime.txt

# Verify acl package is available
pacman -Q acl | tee evidence/i3/env_acl_version.txt

# Verify filesystem supports ACLs (Btrfs does natively)
touch /tmp/i3-acl-test && setfacl -m u:$(id -un):r /tmp/i3-acl-test \
  && echo "PASS: ACLs supported on this filesystem" \
  || echo "FAIL: ACLs not supported — check filesystem mount options"
rm -f /tmp/i3-acl-test

# Snapper pre-snapshot
sudo snapper -c root create --description "pre-i3-acl-secret-store" --print-number \
  | tee evidence/i3/snapshot_pre_num.txt
```

Expected:
- `acl` version output confirms package is installed.
- ACL test passes with "PASS: ACLs supported."

### Step 2 — Create dedicated service user

```bash
# Create a non-login, no-home service user for the secret consumer
# System users have UIDs below 1000 on most Linux systems
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  --comment "Lab Secret Store Consumer Service" \
  svc-labsecret

# Verify user was created correctly
getent passwd svc-labsecret | tee evidence/i3/svc_user_entry.txt
id svc-labsecret | tee evidence/i3/svc_user_id.txt

# Confirm no login shell and no home directory
echo "Shell: $(getent passwd svc-labsecret | cut -d: -f7)" | tee -a evidence/i3/svc_user_id.txt
echo "Home: $(getent passwd svc-labsecret | cut -d: -f6)" | tee -a evidence/i3/svc_user_id.txt
```

Expected:
- `svc_user_entry.txt` shows the user with shell `/usr/sbin/nologin` and UID in system range (typically < 1000).
- No home directory is created.

### Step 3 — Create and restrict the secret store directory

```bash
SECRET_DIR="/etc/lab-secrets"

# Create the directory
sudo mkdir -p "$SECRET_DIR"

# Set restrictive base permissions: root-owned, no group/world access
sudo chown root:root "$SECRET_DIR"
sudo chmod 700 "$SECRET_DIR"

# Capture base state
stat "$SECRET_DIR" | tee evidence/i3/secret_dir_stat_before_acl.txt
getfacl "$SECRET_DIR" | tee evidence/i3/secret_dir_acl_before.txt

# Create a dummy secret file (placeholder only — never use real secrets here)
echo "dummy-api-key-placeholder-$(date -u +%Y%m%d)" | sudo tee "$SECRET_DIR/api-key.txt" > /dev/null
echo "dummy-db-password-placeholder" | sudo tee "$SECRET_DIR/db-password.txt" > /dev/null

# Set restrictive permissions on secret files
sudo chmod 600 "$SECRET_DIR/api-key.txt"
sudo chmod 600 "$SECRET_DIR/db-password.txt"
sudo chown root:root "$SECRET_DIR/api-key.txt" "$SECRET_DIR/db-password.txt"

# Confirm base permissions
ls -lah "$SECRET_DIR" | tee evidence/i3/secret_dir_listing_before_acl.txt
```

Expected:
- `SECRET_DIR` has permissions `drwx------` (700) owned by `root:root`.
- Secret files have permissions `-rw-------` (600) owned by `root:root`.
- `getfacl` shows only the base owner/group/other entries — no ACL entries yet.

### Step 4 — Apply ACL entries for the service user

```bash
SECRET_DIR="/etc/lab-secrets"

# Grant svc-labsecret read-only access to the directory (for traversal)
sudo setfacl -m u:svc-labsecret:rx "$SECRET_DIR"

# Grant svc-labsecret read-only access to each secret file
sudo setfacl -m u:svc-labsecret:r "$SECRET_DIR/api-key.txt"
sudo setfacl -m u:svc-labsecret:r "$SECRET_DIR/db-password.txt"

# Set the default ACL so new files in the directory inherit the restriction
sudo setfacl -d -m u:svc-labsecret:r "$SECRET_DIR"
sudo setfacl -d -m o::--- "$SECRET_DIR"

# Verify the ACLs were applied
getfacl "$SECRET_DIR" | tee evidence/i3/secret_dir_acl_after.txt
getfacl "$SECRET_DIR/api-key.txt" | tee evidence/i3/secret_apikey_acl.txt
getfacl "$SECRET_DIR/db-password.txt" | tee evidence/i3/secret_dbpassword_acl.txt

# The effective permissions mask — verify it does not override our ACL
ls -lah "$SECRET_DIR" | tee evidence/i3/secret_dir_listing_after_acl.txt
```

Expected:
- `getfacl` shows `user:svc-labsecret:r-x` on the directory and `user:svc-labsecret:r--` on files.
- Traditional `ls -lah` shows a `+` at the end of the permission string, indicating ACL presence.

### Step 5 — Validate: service user can read, other users cannot

```bash
SECRET_DIR="/etc/lab-secrets"

# Test 1: svc-labsecret can read the secret file
echo "--- Test 1: svc-labsecret reads api-key.txt ---" | tee evidence/i3/access_tests.txt
sudo -u svc-labsecret cat "$SECRET_DIR/api-key.txt" | tee -a evidence/i3/access_tests.txt \
  && echo "PASS: svc-labsecret can read api-key.txt" | tee -a evidence/i3/access_tests.txt \
  || echo "FAIL: svc-labsecret cannot read api-key.txt" | tee -a evidence/i3/access_tests.txt

# Test 2: svc-labsecret cannot write to the secret file (read-only ACL)
echo "--- Test 2: svc-labsecret cannot write ---" | tee -a evidence/i3/access_tests.txt
sudo -u svc-labsecret bash -c "echo test >> $SECRET_DIR/api-key.txt" 2>&1 \
  | tee -a evidence/i3/access_tests.txt \
  && echo "FAIL: svc-labsecret can write (ACL too permissive)" | tee -a evidence/i3/access_tests.txt \
  || echo "PASS: write correctly denied" | tee -a evidence/i3/access_tests.txt

# Test 3: A second non-root user (nobody) cannot access the secret directory
echo "--- Test 3: nobody cannot access secrets ---" | tee -a evidence/i3/access_tests.txt
sudo -u nobody cat "$SECRET_DIR/api-key.txt" 2>&1 \
  | tee -a evidence/i3/access_tests.txt \
  && echo "FAIL: nobody can read secrets" | tee -a evidence/i3/access_tests.txt \
  || echo "PASS: nobody correctly denied" | tee -a evidence/i3/access_tests.txt

# Test 4: Verify new files inherit restrictive ACL
echo "--- Test 4: new file inherits default ACL ---" | tee -a evidence/i3/access_tests.txt
echo "dummy-new-secret" | sudo tee "$SECRET_DIR/new-secret.txt" > /dev/null
sudo chmod 600 "$SECRET_DIR/new-secret.txt"
getfacl "$SECRET_DIR/new-secret.txt" | tee -a evidence/i3/access_tests.txt
```

Expected:
- Test 1 PASS: `api-key.txt` content is read successfully as `svc-labsecret`.
- Test 2 PASS: Write attempt exits non-zero with "Permission denied."
- Test 3 PASS: `nobody` gets "Permission denied" accessing the directory.
- Test 4: `getfacl` shows the inherited default ACL on the new file.

### Step 6 — Capture final state and produce audit report

```bash
echo "=== ACL Audit Report ===" | tee evidence/i3/acl_audit_report.txt
echo "Date: $(date -u)" | tee -a evidence/i3/acl_audit_report.txt
echo "" | tee -a evidence/i3/acl_audit_report.txt
echo "Secret directory: /etc/lab-secrets" | tee -a evidence/i3/acl_audit_report.txt
echo "" | tee -a evidence/i3/acl_audit_report.txt
echo "Directory ACL:" | tee -a evidence/i3/acl_audit_report.txt
getfacl /etc/lab-secrets | tee -a evidence/i3/acl_audit_report.txt
echo "" | tee -a evidence/i3/acl_audit_report.txt
echo "File ACLs:" | tee -a evidence/i3/acl_audit_report.txt
for f in /etc/lab-secrets/*; do
  echo "--- $f ---" | tee -a evidence/i3/acl_audit_report.txt
  getfacl "$f" | tee -a evidence/i3/acl_audit_report.txt
done

# Create Snapper post-snapshot
sudo snapper -c root create --description "post-i3-acl-secret-store" --print-number \
  | tee evidence/i3/snapshot_post_num.txt
```

## 8) Validation

```bash
SECRET_DIR="/etc/lab-secrets"

# Check svc-labsecret user exists
getent passwd svc-labsecret > /dev/null \
  && echo "PASS: svc-labsecret user exists" \
  || echo "FAIL: svc-labsecret user not found"

# Check directory permissions
perms=$(stat -c "%a" "$SECRET_DIR")
[ "$perms" = "770" ] || [ "$perms" = "700" ] \
  && echo "PASS: Directory permissions restrictive ($perms)" \
  || echo "WARN: Directory permissions: $perms (expected 700 or 770)"

# Check ACL entry exists
getfacl "$SECRET_DIR" | grep -q "svc-labsecret" \
  && echo "PASS: ACL entry for svc-labsecret present" \
  || echo "FAIL: ACL entry for svc-labsecret missing"

# Check access test results
grep -q "PASS: svc-labsecret can read" evidence/i3/access_tests.txt \
  && echo "PASS: Read access test confirmed" \
  || echo "FAIL: Read access test not passed"

grep -q "PASS: write correctly denied" evidence/i3/access_tests.txt \
  && echo "PASS: Write denial confirmed" \
  || echo "FAIL: Write denial not confirmed"

grep -q "PASS: nobody correctly denied" evidence/i3/access_tests.txt \
  && echo "PASS: Third-party denial confirmed" \
  || echo "FAIL: Third-party denial not confirmed"
```

## 9) Evidence

- **Output files:**
  - `evidence/i3/env_invoking_user.txt` — operator identity
  - `evidence/i3/env_datetime.txt` — investigation timestamp
  - `evidence/i3/env_acl_version.txt` — acl package version
  - `evidence/i3/snapshot_pre_num.txt` — Snapper pre-snapshot number
  - `evidence/i3/svc_user_entry.txt` — /etc/passwd entry for service user
  - `evidence/i3/svc_user_id.txt` — id output for service user
  - `evidence/i3/secret_dir_stat_before_acl.txt` — directory stat before ACLs
  - `evidence/i3/secret_dir_acl_before.txt` — getfacl before ACLs
  - `evidence/i3/secret_dir_listing_before_acl.txt` — ls before ACLs
  - `evidence/i3/secret_dir_acl_after.txt` — getfacl after ACLs applied
  - `evidence/i3/secret_apikey_acl.txt` — getfacl for api-key.txt
  - `evidence/i3/secret_dbpassword_acl.txt` — getfacl for db-password.txt
  - `evidence/i3/secret_dir_listing_after_acl.txt` — ls after ACLs (shows + flag)
  - `evidence/i3/access_tests.txt` — all access validation test results
  - `evidence/i3/acl_audit_report.txt` — full ACL audit summary
  - `evidence/i3/snapshot_post_num.txt` — Snapper post-snapshot number
- **Hash manifest:**

```bash
find evidence/i3 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/i3/SHA256SUMS.txt
cat evidence/i3/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `setfacl` returns "Operation not supported."
  - **Cause:** Filesystem does not support ACLs (e.g., FAT32, older ext2 without `acl` mount option).
  - **Fix:** On Btrfs this should not occur. On ext4, verify `acl` mount option is present: `mount | grep acl`. Edit `/etc/fstab` to add `,acl` to the mount options and remount.
  - **Rollback trigger:** N/A — no changes made to the secret store yet.

- **Symptom:** `sudo -u svc-labsecret cat /etc/lab-secrets/api-key.txt` returns "Permission denied" even after ACL is applied.
  - **Cause:** The ACL effective mask may be restricting the granted permissions. Check: `getfacl /etc/lab-secrets/api-key.txt` — the `mask` entry must include read.
  - **Fix:** Set the mask explicitly: `sudo setfacl -m m::r-- /etc/lab-secrets/api-key.txt`
  - **Rollback trigger:** N/A — ACL adjustment only.

- **Symptom:** `useradd` fails with "user 'svc-labsecret' already exists."
  - **Cause:** The project was run previously.
  - **Fix:** Skip Step 2, or delete and recreate: `sudo userdel svc-labsecret && sudo useradd ...`
  - **Rollback trigger:** Snapper rollback will restore `/etc/passwd` to its pre-run state.

## 11) Sources

- [man 1 setfacl — ACL manipulation](https://man7.org/linux/man-pages/man1/setfacl.1.html)
- [man 1 getfacl — ACL display](https://man7.org/linux/man-pages/man1/getfacl.1.html)
- [Arch Wiki — Access Control Lists](https://wiki.archlinux.org/title/Access_Control_Lists)
- [CIS Benchmark — Linux file permissions section](https://www.cisecurity.org/benchmark/distribution_independent_linux)
- [NIST SP 800-53 — AC-3 Access Enforcement](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [POSIX.1e — ACL specification background](https://www.usenix.org/legacy/publications/library/proceedings/usenix03/tech/freenix03/full_papers/gruenbacher/gruenbacher_html/main.html)

## 12) Stretch Goals

- Extend the ACL store to support a second service user (`svc-labsecret-ro`) with read-only access and a writer service (`svc-labsecret-rw`) with write access to a separate secrets path. Document the ACL matrix.
- Set up `auditd` to log every access to `/etc/lab-secrets/` and generate an access report after running the validation tests. Rules: `sudo auditctl -w /etc/lab-secrets/ -p rwa -k lab-secret-access`
- Explore using the Linux kernel keyring (`keyctl`) to store a secret and compare it with the ACL-backed approach: which is more appropriate for ephemeral credentials vs persistent ones?
- Combine with I1 (Systemd Service Hardening): configure the service unit to run as `svc-labsecret` using `User=svc-labsecret` and confirm it can read its secrets while being sandboxed.
