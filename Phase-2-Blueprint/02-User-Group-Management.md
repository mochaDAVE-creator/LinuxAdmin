---
aliases:
  - User Group Management
  - useradd usermod
  - /etc/shadow
  - sudo hardening
tags:
  - linux
  - users
  - groups
  - sudo
  - net412
  - net377
  - phase2
date: 2026-05-10
---

# 02 — User & Group Management

> [!info] Why This Matters
> User and group management is the identity layer of your system. In a CTF, understanding this layer reveals privilege escalation paths. In production, mismanaged accounts are the most common initial compromise vector. Know every account on your system — and why it exists.

## The Identity Files

```bash
/etc/passwd     # Account info: username:x:UID:GID:GECOS:home:shell
/etc/shadow     # Hashed passwords (root-readable only)
/etc/group      # Group definitions: groupname:x:GID:member1,member2
/etc/gshadow    # Group password hashes (rarely used)
/etc/sudoers    # Sudo privilege configuration
/etc/sudoers.d/ # Modular sudo rules (preferred over editing sudoers directly)
```

### Decoding /etc/passwd

```
root:x:0:0:root:/root:/bin/bash
│    │ │ │ │    │     └── login shell
│    │ │ │ │    └── home directory
│    │ │ │ └── GECOS (comment/full name)
│    │ │ └── primary GID
│    │ └── UID
│    └── password placeholder (x = look in /etc/shadow)
└── username
```

```bash
# Parse useful information
awk -F: '{print $1, $3, $6, $7}' /etc/passwd     # user, UID, home, shell

# UID ranges (Arch Linux defaults)
# 0       = root
# 1-999   = system accounts (services, daemons)
# 1000+   = regular human users (created by useradd)

# Forensic checks
awk -F: '$3 == 0 {print "UID-0:", $1}' /etc/passwd          # root-level accounts
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd             # human accounts
awk -F: '$7 !~ /nologin|false/ {print $1}' /etc/passwd      # accounts with shells
```

### Decoding /etc/shadow

```
dave:$6$rounds=5000$salt$hash:19121:0:99999:7:::
│    │                        │     │ │     └── inactive days after expiry
│    │                        │     │ └── max password age (days)
│    │                        │     └── min days between changes
│    │                        └── last password change (days since epoch)
│    └── hash: $id$salt$hash ($6 = SHA-512)
└── username
```

Hash ID prefixes:
- `$1$` = MD5 (old, weak)
- `$5$` = SHA-256
- `$6$` = SHA-512 (current standard)
- `$y$` = yescrypt (Arch default on newer systems)
- `!` or `*` = account locked (no password login)
- `!!` = account was never given a password

```bash
# Check for empty or weak-hash accounts
sudo awk -F: '$2 == "" || $2 == "!" {print "NO PASSWORD:", $1}' /etc/shadow
sudo awk -F: '$2 ~ /^\$1\$/ {print "MD5 HASH (WEAK):", $1}' /etc/shadow
```

---

## User Management

### Creating Users

```bash
# Create a standard user (Arch: home dir created by default)
useradd -m -s /bin/bash -G wheel dave
# -m = create home directory
# -s = specify login shell
# -G = supplementary groups (wheel = sudo on Arch)

# Create a system account (for services — no home, no shell)
useradd --system --no-create-home --shell /usr/sbin/nologin apex_engine
# This is the correct way to create service accounts (NET 412, CIS benchmark)

# Create user with specific UID and primary group
useradd -u 1050 -g forensics -m analyst2

# Set password
passwd dave              # interactive
echo "dave:MyP@ss123" | chpasswd   # non-interactive (avoid in scripts — use env vars)
```

### Modifying Users

```bash
# Change primary group
usermod -g forensics dave

# Add to supplementary groups (append, don't replace)
usermod -aG docker dave
usermod -aG wheel dave        # sudo access
usermod -aG wireshark dave    # Wireshark capture

# Change home directory and move contents
usermod -d /new/home -m dave

# Lock/unlock account
usermod -L dave               # lock (prepends ! to shadow hash)
usermod -U dave               # unlock

# Change shell
usermod -s /bin/zsh dave
chsh -s /bin/fish             # change your own shell

# Expire account (force password reset on next login)
chage -d 0 dave
```

### Deleting Users

```bash
# Remove user but keep home directory
userdel dave

# Remove user AND home directory (caution)
userdel -r dave

# Check for orphaned files after deletion
find / -nouser -o -nogroup 2>/dev/null | head -20
```

---

## Group Management

```bash
# Create group
groupadd forensics
groupadd -g 1500 ctf_team    # specify GID

# Add user to group
usermod -aG forensics dave
gpasswd -a dave forensics    # alternative

# Remove user from group
gpasswd -d dave forensics

# Change primary group temporarily (within current session)
newgrp forensics             # opens new shell with forensics as primary group

# List group memberships
groups dave
id dave
# Output: uid=1000(dave) gid=1000(dave) groups=1000(dave),998(wheel),969(wireshark),...
```

---

## sudo — Privilege Elevation

### Understanding /etc/sudoers Syntax

```bash
# Format: who   where=(as_whom)   what
root    ALL=(ALL:ALL) ALL          # root can do everything
%wheel  ALL=(ALL:ALL) ALL          # wheel group can do everything
dave    ALL=(ALL) /usr/bin/tcpdump # dave can only run tcpdump as any user
%forensics ALL=(ALL) NOPASSWD: /usr/bin/journalctl   # forensics group, no password prompt
```

> [!warning] NEVER Edit sudoers Directly
> Always use `visudo` which validates syntax before saving. A corrupt `/etc/sudoers` locks you out of sudo entirely.
> ```bash
> visudo                           # edit main sudoers
> visudo -f /etc/sudoers.d/dave   # edit/create a drop-in file
> ```

### CIS-Hardened sudo Configuration

```bash
# Check current sudo rules
sudo -l                          # your own rules
sudo -l -U dave                  # another user's rules (as root)

# Recommended CIS settings in /etc/sudoers or sudoers.d:
Defaults    env_reset
Defaults    mail_badpass
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    use_pty              # prevent sudo from being used in scripts/pipes
Defaults    logfile="/var/log/sudo.log"  # audit trail
Defaults    log_input            # log all stdin to sudoreplay
Defaults    log_output           # log all output to sudoreplay
Defaults    !visiblepw           # don't show password in prompt
Defaults    timestamp_timeout=5  # re-authenticate after 5 minutes

# Restrict root SSH login (sudo is the right way to escalate)
PermitRootLogin no              # in /etc/ssh/sshd_config
```

### Auditing sudo Usage

```bash
# Check sudo log (journald)
journalctl -u sudo --no-pager | tail -20

# Parse sudo events
journalctl | grep "sudo" | grep -v "pam_unix" | tail -20

# Check /var/log/sudo.log if configured
tail -f /var/log/sudo.log 2>/dev/null
```

---

## PAM — Pluggable Authentication Modules

PAM is the authentication framework behind su, sudo, SSH, and login. Understanding PAM = understanding how authentication actually works.

```bash
/etc/pam.d/              # PAM configuration directory
/etc/pam.d/sshd          # SSH authentication stack
/etc/pam.d/sudo          # sudo authentication stack
/etc/pam.d/login         # console login
/etc/pam.d/system-auth   # Arch: common auth settings
```

```bash
# Enforce password complexity with pam_pwquality
# /etc/security/pwquality.conf
minlen = 14
dcredit = -1    # at least 1 digit
ucredit = -1    # at least 1 uppercase
lcredit = -1    # at least 1 lowercase
ocredit = -1    # at least 1 special char

# Account lockout after failed attempts (pam_faillock)
# /etc/security/faillock.conf
deny = 5                 # lock after 5 failures
unlock_time = 900        # unlock after 15 minutes
```

---

## Practical Lab: Arch Live System

```bash
# 1. Audit all accounts on your system
echo "=== All Accounts ==="
awk -F: '{printf "%-20s UID:%-6s Shell:%s\n", $1, $3, $7}' /etc/passwd | column -t

echo ""
echo "=== Human Accounts (UID >= 1000) ==="
awk -F: '$3 >= 1000 && $1 != "nobody" {print $1, $3, $7}' /etc/passwd

echo ""
echo "=== Sudo-Capable Users ==="
grep -E "^%wheel|^%sudo" /etc/sudoers 2>/dev/null
grep -r "ALL\s*=\s*(ALL" /etc/sudoers.d/ 2>/dev/null

# 2. Check your own sudo access
sudo -l

# 3. Create a test service account (safe — uses nologin)
sudo useradd --system --no-create-home --shell /usr/sbin/nologin test_service
id test_service
grep "test_service" /etc/passwd

# 4. Lock it (already locked by nologin, but demonstrate)
sudo passwd -l test_service
sudo grep "test_service" /etc/shadow | cut -d: -f1,2

# 5. Clean up
sudo userdel test_service
```

---

## Troubleshooting Scenario: Locked Out of sudo

**Symptom:** `sudo: PAM authentication failure` or `sudo: no valid sudoers sources found`

**Recovery (if you have root console access):**
```bash
# Boot to recovery mode or single-user mode
# On GRUB: press 'e', add 'single' or 'init=/bin/bash' to kernel line

# Once at root shell, remount filesystem read-write
mount -o remount,rw /

# Fix sudoers
visudo

# Or restore from backup
cp /etc/sudoers.bak /etc/sudoers
chmod 440 /etc/sudoers
chown root:root /etc/sudoers
```

**If sudoers syntax is corrupted:**
```bash
# Use pkexec as an alternative to sudo (if PolicyKit is installed)
pkexec visudo

# Or use su to switch to root (if root password is set)
su -
visudo
```

> [!tip] Btrfs Rollback as Recovery
> If your Arch system has Snapper configured, you can roll back to a pre-corruption snapshot before the sudoers edit broke things. See [[../Phase-5-Architect/02-Btrfs-Snapshots|Phase 5 — Btrfs Snapshots]].

---

## 🏁 Proof of Work — Phase 2.2 Mini-CTF

> [!example] Challenge: Least-Privilege Service Account
> Set up a proper least-privilege account for your Apex Predator PMO Rust engine.
>
> **Requirements:**
> 1. Create system account `apex_engine` with no home, no shell
> 2. Create group `apex_group`
> 3. Add `apex_engine` to `apex_group`
> 4. Give `dave` sudo permission to restart only `apex-predator.service` — no password required
> 5. Verify the account cannot be used to log in interactively
>
> ```bash
> # Your solution:
> sudo groupadd apex_group 2>/dev/null
> sudo useradd --system --no-create-home --shell /usr/sbin/nologin --gid apex_group apex_engine
>
> # Create sudoers rule (validate with visudo)
> echo "dave ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart apex-predator.service" \
>   | sudo tee /etc/sudoers.d/dave_apex
> sudo chmod 440 /etc/sudoers.d/dave_apex
> sudo visudo -c -f /etc/sudoers.d/dave_apex   # syntax check
>
> # Validation
> id apex_engine
> sudo -l -U dave | grep apex
> su - apex_engine 2>&1 || echo "Cannot log in interactively — CORRECT"
>
> # Cleanup
> sudo userdel apex_engine
> sudo groupdel apex_group
> sudo rm /etc/sudoers.d/dave_apex
> ```
>
> **Success:** `id apex_engine` shows a system UID. `su - apex_engine` fails. `sudo -l` shows restricted systemctl rule.

---

← [[01-Permissions-ACLs]] | Next: [[03-Securing-Python-SQLite]] →
