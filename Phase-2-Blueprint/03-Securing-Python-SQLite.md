---
aliases:
  - Securing Python SQLite
  - Database Security
  - AppArmor
  - Encryption at Rest
tags:
  - linux
  - security
  - sqlite
  - python
  - apparmor
  - net412
  - net210
  - phase2
date: 2026-05-10
---

# 03 — Securing Your Python SQLite Cryptography Database

> [!info] Why This Matters
> Your Python cryptography toolkit stores key material, hashes, and potentially plaintext secrets in an SQLite database. If that file is world-readable, you've done all the cryptographic work and handed the result to anyone with `cat`. This module locks it down at every layer.

## The Threat Model

Your database lives at (example): `~/crypto_toolkit/keys.db`

| Threat | Vector | Control |
|---|---|---|
| Local user reads DB | File permissions | `chmod 600 keys.db` |
| Process escapes sandbox | Missing AppArmor | AppArmor profile |
| Backup exposes plaintext | Unencrypted archive | SQLCipher or SQLite encryption |
| App runs as root | Bad practice | Dedicated service account |
| Temp files expose data | Python tmpfiles | `tempfile.mkstemp()` in `/run/user/$(id -u)` |
| Memory dump exposes keys | Key in memory too long | Key handling patterns in code |

---

## Layer 1: File System Permissions

```bash
# Locate your database
DB_PATH="$HOME/crypto_toolkit/keys.db"

# Set strict ownership — only your account can read/write
chmod 600 "$DB_PATH"
chown dave:dave "$DB_PATH"

# The parent directory should also be restricted
chmod 700 "$HOME/crypto_toolkit/"

# Verify
stat "$DB_PATH"
# Access: (0600/-rw-------)  Uid: (1000/dave)  Gid: (1000/dave)

# If running as a service, use the service account
sudo chown apex_engine:apex_group "$DB_PATH"
sudo chmod 640 "$DB_PATH"  # owner rw, group r (service account in apex_group)
```

> [!warning] Never Run Your App as Root
> If your Python toolkit runs as root and gets exploited (e.g., via a malicious crypto input), the attacker has full system access. Run it as a dedicated service account instead. See [[02-User-Group-Management|User & Group Management]] for creating service accounts.

---

## Layer 2: Filesystem ACLs for Fine-Grained Access

```bash
# Allow only your application's process user to access the DB
# Assuming application runs as 'apex_engine'
sudo setfacl -m u:apex_engine:rw "$DB_PATH"
sudo setfacl -m u:dave:rw "$DB_PATH"           # developer access
sudo setfacl -m o::--- "$DB_PATH"              # others: no access

# Set ACL on the directory too
sudo setfacl -m u:apex_engine:rx "$HOME/crypto_toolkit/"
sudo setfacl -m u:dave:rwx "$HOME/crypto_toolkit/"

# Verify
getfacl "$DB_PATH"
```

---

## Layer 3: AppArmor Profile for Your Python Script

AppArmor confines what your Python script can access, even if the code is exploited. It operates on **mandatory access control (MAC)** — enforced by the kernel, bypasses DAC.

> [!info] AppArmor vs. SELinux
> Arch Linux uses AppArmor (easier to write profiles). RHEL/CentOS use SELinux. Both provide MAC. For NET 412/NET 210: know that MAC enforces policy regardless of what root wants.

### Installing AppArmor on Arch

```bash
# Install AppArmor
sudo pacman -S apparmor

# Enable kernel parameter (add to GRUB_CMDLINE_LINUX in /etc/default/grub)
# apparmor=1 security=apparmor
sudo nvim /etc/default/grub
# Add to GRUB_CMDLINE_LINUX_DEFAULT: "apparmor=1 security=apparmor"

# Update GRUB
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Enable AppArmor service
sudo systemctl enable --now apparmor

# Verify (after reboot)
sudo aa-status
```

### Writing an AppArmor Profile

```bash
# Profile location
sudo nvim /etc/apparmor.d/usr.bin.crypto_toolkit
```

```
# /etc/apparmor.d/usr.bin.crypto_toolkit
# AppArmor profile for Python cryptography toolkit

#include <tunables/global>

/home/dave/crypto_toolkit/main.py {
  #include <abstractions/base>
  #include <abstractions/python>
  #include <abstractions/nameservice>

  # Binary and interpreter
  /usr/bin/python3*  rix,    # read, inherit, execute

  # Script files (read only — can't modify its own source)
  /home/dave/crypto_toolkit/  r,
  /home/dave/crypto_toolkit/** r,

  # Database — read and write only, no exec
  /home/dave/crypto_toolkit/keys.db  rw,
  /home/dave/crypto_toolkit/keys.db-shm rw,
  /home/dave/crypto_toolkit/keys.db-wal rw,

  # Python libraries
  /usr/lib/python3*/**  r,
  /usr/lib/python3*/lib-dynload/*.so m,    # allow memory-mapping .so files

  # Temp files — constrained to user runtime dir
  owner /run/user/*/crypto_toolkit.tmp rw,

  # Log output
  /home/dave/.local/share/crypto_toolkit/  rw,
  /home/dave/.local/share/crypto_toolkit/** rw,

  # Deny everything else
  deny /etc/shadow r,
  deny /etc/passwd w,
  deny /root/** rw,
  deny /home/*/  rw,   # deny other users' homes
  deny network,        # no network access (cryptography only)
}
```

```bash
# Load the profile in complain mode first (log violations, don't block)
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.crypto_toolkit
sudo aa-complain /home/dave/crypto_toolkit/main.py

# Run your app and observe violations
sudo journalctl -f | grep "apparmor" &
python3 /home/dave/crypto_toolkit/main.py

# After testing, switch to enforce mode
sudo aa-enforce /home/dave/crypto_toolkit/main.py

# Check status
sudo aa-status | grep crypto
```

---

## Layer 4: Encryption at Rest with SQLCipher

Standard SQLite has no encryption. SQLCipher is a drop-in AES-256-CBC encrypted fork.

### Python Integration

```bash
# Install pysqlcipher3
pip install pysqlcipher3
# Or on Arch:
sudo pacman -S sqlcipher
pip install sqlcipher3
```

```python
# In your crypto_toolkit/db.py
from sqlcipher3 import dbapi2 as sqlite

# DO NOT hardcode the key — read from environment or keyring
import os
import keyring  # pip install keyring

def get_db_connection(db_path: str):
    """Open encrypted SQLite connection using key from system keyring."""
    key = keyring.get_password("crypto_toolkit", "db_key")
    if not key:
        raise ValueError("Database key not found in keyring. Run setup first.")
    
    conn = sqlite.connect(db_path)
    # Pragma must be first statement before any DB operation
    conn.execute(f"PRAGMA key='{key}'")
    conn.execute("PRAGMA cipher_page_size = 4096")
    conn.execute("PRAGMA kdf_iter = 256000")   # key derivation iterations
    conn.execute("PRAGMA cipher_hmac_algorithm = HMAC_SHA512")
    return conn

def setup_db_key():
    """One-time key generation — store securely in system keyring."""
    import secrets
    key = secrets.token_hex(32)   # 256-bit key
    keyring.set_password("crypto_toolkit", "db_key", key)
    print("Key stored in system keyring. Protect your keyring password.")
```

> [!tip] System Keyring on Arch
> Arch uses `libsecret` (GNOME Keyring or KWallet backend). For headless/server use, use `pass` (password-store) or HashiCorp Vault. Never put the key in a `.env` file in the same directory as the database.

---

## Layer 5: Secure Temp File Handling in Python

```python
import tempfile
import os

# BAD — creates predictable temp file in /tmp (race condition / symlink attack)
# f = open("/tmp/crypto_temp.dat", "w")

# GOOD — secure temp file in user-private runtime directory
runtime_dir = f"/run/user/{os.getuid()}"  # mode 700, owned by user, in RAM
with tempfile.NamedTemporaryFile(
    dir=runtime_dir,
    prefix="crypto_",
    suffix=".tmp",
    delete=True   # auto-delete on close
) as tmp:
    tmp.write(b"sensitive intermediate data")
    tmp.flush()
    # ... process data ...
# File is automatically deleted when context exits
```

---

## Layer 6: Systemd Service Sandboxing

If your toolkit runs as a systemd service, harden the unit file:

```ini
# /etc/systemd/system/crypto-toolkit.service
[Unit]
Description=Python Cryptography Toolkit
After=network.target

[Service]
Type=simple
User=apex_engine
Group=apex_group
WorkingDirectory=/home/dave/crypto_toolkit
ExecStart=/usr/bin/python3 main.py

# Filesystem isolation
PrivateTmp=true           # private /tmp (not shared with system)
PrivateDevices=true       # no device access
ProtectHome=read-only     # can't write to /home
ProtectSystem=strict      # /usr, /boot, /etc read-only
ReadWritePaths=/home/dave/crypto_toolkit/   # explicit write access

# Network isolation (cryptography toolkit doesn't need network)
PrivateNetwork=true       # or use: RestrictAddressFamilies=

# Capability drops
CapabilityBoundingSet=    # drop ALL capabilities
NoNewPrivileges=true      # can't escalate privileges

# System call filtering
SystemCallFilter=@system-service
SystemCallFilter=~@mount @privileged @resources

[Install]
WantedBy=multi-user.target
```

```bash
# Load and analyze the hardening
sudo systemctl daemon-reload
sudo systemctl start crypto-toolkit
# Check the security score
systemd-analyze security crypto-toolkit.service
```

> [!tip] systemd-analyze security
> Run `systemd-analyze security <service>` on any service unit to get a security exposure score. Below 4.0 is well-hardened. The goal is to lock it down until systemd runs out of things to complain about.

---

## Practical Lab: Harden Your Database Right Now

```bash
# 1. Find your SQLite databases
find ~ -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" 2>/dev/null | head -20

# 2. Check their current permissions
find ~ -name "*.db" 2>/dev/null | xargs ls -la 2>/dev/null

# 3. Fix any that are too permissive
find ~ -name "*.db" -perm /044 2>/dev/null   # group/other readable — these need fixing
find ~ -name "*.db" -perm /044 2>/dev/null -exec chmod 600 {} \;

# 4. Verify
find ~ -name "*.db" 2>/dev/null | xargs stat -c "%a %n" 2>/dev/null

# 5. Check if AppArmor is loaded
sudo aa-status 2>/dev/null | head -10 || echo "AppArmor not active — consider enabling"
```

---

## Troubleshooting Scenario: Python App Can't Open Database After chmod

**Symptom:** After running `chmod 600 keys.db`, your Python app (running as a service account) gets `sqlite3.OperationalError: unable to open database file`.

**Diagnosis:**
```bash
# Check what user the service is running as
systemctl status crypto-toolkit | grep "Main PID"
ps aux | grep "main.py"

# Check if that user can access the file
sudo -u apex_engine ls -la /home/dave/crypto_toolkit/keys.db
# → Permission denied

# Check the directory permissions too
ls -la /home/dave/ | grep crypto_toolkit
# The directory itself may not be traversable by apex_engine
```

**Fix:**
```bash
# Option A: Change file ownership to the service account
sudo chown apex_engine:apex_group /home/dave/crypto_toolkit/keys.db
sudo chown apex_engine:apex_group /home/dave/crypto_toolkit/

# Option B: Use ACLs to grant access without changing ownership
sudo setfacl -m u:apex_engine:rx /home/dave/
sudo setfacl -m u:apex_engine:rx /home/dave/crypto_toolkit/
sudo setfacl -m u:apex_engine:rw /home/dave/crypto_toolkit/keys.db

# Verify
sudo -u apex_engine stat /home/dave/crypto_toolkit/keys.db
```

---

## 🏁 Proof of Work — Phase 2.3 Mini-CTF

> [!example] Challenge: Security Audit Your Own Database
>
> Run this audit script against your Python toolkit database:
>
> ```bash
> #!/bin/bash
> # Phase 2.3 Proof of Work: Database Security Audit
>
> DB_PATH="${1:-$HOME/crypto_toolkit/keys.db}"
>
> echo "=== DATABASE SECURITY AUDIT ==="
> echo "Target: $DB_PATH"
> echo "Date: $(date -u)"
> echo ""
>
> # Check 1: File exists
> if [ ! -f "$DB_PATH" ]; then
>   echo "[SKIP] Database not found at $DB_PATH"
>   echo "  Creating a test database for audit purposes..."
>   mkdir -p "$(dirname "$DB_PATH")"
>   sqlite3 "$DB_PATH" "CREATE TABLE test (id INTEGER PRIMARY KEY);"
> fi
>
> # Check 2: Permissions
> PERM=$(stat -c "%a" "$DB_PATH")
> OWNER=$(stat -c "%U:%G" "$DB_PATH")
> echo "[CHECK] Permissions: $PERM (owner: $OWNER)"
> if [ "$PERM" -le 640 ]; then
>   echo "  [PASS] Permissions are sufficiently restrictive"
> else
>   echo "  [FAIL] Permissions too open — run: chmod 600 $DB_PATH"
> fi
>
> # Check 3: World-readable?
> if [ $((8#$PERM & 8#004)) -ne 0 ]; then
>   echo "  [CRITICAL] Database is world-readable!"
> else
>   echo "  [PASS] Not world-readable"
> fi
>
> # Check 4: ACLs
> echo ""
> echo "[CHECK] ACL Status:"
> getfacl "$DB_PATH" 2>/dev/null
>
> # Check 5: AppArmor
> echo ""
> echo "[CHECK] AppArmor Status:"
> sudo aa-status 2>/dev/null | grep -E "enforce|complain|profile" | head -5 \
>   || echo "  AppArmor not active"
>
> echo ""
> echo "=== AUDIT COMPLETE ==="
> ```
>
> **Save, run, and capture output:**
> ```bash
> bash audit.sh 2>&1 | tee /tmp/phase2_3_pow.txt
> sha256sum /tmp/phase2_3_pow.txt
> ```

---

← [[02-User-Group-Management]] | [[../Phase-3-Engine-Room/00-Phase3-Overview|Next: Phase 3 →]]
