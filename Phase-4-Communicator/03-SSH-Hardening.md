---
aliases:
  - SSH Hardening
  - SSH Key Auth
  - sshd_config
  - Jump Hosts
  - Proxmox SSH
tags:
  - linux
  - ssh
  - security
  - proxmox
  - net412
  - net210
  - phase4
date: 2026-05-10
---

# 03 — SSH Hardening & Proxmox Key Auth

> [!info] Why This Matters
> SSH is the primary administrative channel for every Linux system. A misconfigured SSH daemon is an open invitation to brute force attacks, credential theft, and unauthorized access. Key-based authentication + proper hardening makes SSH effectively impenetrable to automated attacks.

## The SSH Key Pair Model

```
Your Arch Machine (client)          Proxmox/Remote Host (server)
┌─────────────────────────┐         ┌──────────────────────────────┐
│  ~/.ssh/id_ed25519      │         │  ~/.ssh/authorized_keys      │
│  (private key — secret) │─────────│  (your public key stored here│
│  ~/.ssh/id_ed25519.pub  │         │                              │
│  (public key — shareable│         │  /etc/ssh/sshd_config        │
└─────────────────────────┘         │  (daemon configuration)      │
                                    └──────────────────────────────┘
Authentication: Client proves it has the private key without ever sending it.
```

---

## Generating Keys

```bash
# ED25519 — current best practice (fast, secure, small key size)
ssh-keygen -t ed25519 -C "dave@archbox-$(date +%Y%m%d)"

# RSA 4096 — for compatibility with older servers
ssh-keygen -t rsa -b 4096 -C "dave@archbox-$(date +%Y%m%d)"

# Options:
# -t = key type
# -b = bits (RSA only)
# -C = comment (helps identify which key this is)
# -f = output file (don't use default if you have multiple keys)
# -N = passphrase (empty string = no passphrase — risky)

# Always use a passphrase — use ssh-agent so you only type it once
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_proxmox -C "proxmox-access-$(date +%Y%m%d)"

# Set correct permissions immediately
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/id_ed25519_proxmox
chmod 644 ~/.ssh/id_ed25519_proxmox.pub
```

> [!warning] Private Key Permissions are Enforced
> SSH refuses to use a private key if it's readable by anyone else:
> ```
> WARNING: UNPROTECTED PRIVATE KEY FILE!
> Permissions 0644 for 'id_ed25519' are too open.
> ```
> Fix: `chmod 600 ~/.ssh/id_ed25519`

---

## Distributing Keys to Remote Hosts

### Method 1: ssh-copy-id (Easiest)

```bash
# Copy your public key to a remote host
ssh-copy-id -i ~/.ssh/id_ed25519_proxmox.pub dave@192.168.1.254

# To a specific port
ssh-copy-id -i ~/.ssh/id_ed25519_proxmox.pub -p 2222 dave@192.168.1.254
```

### Method 2: Manual (When ssh-copy-id Isn't Available)

```bash
# On remote host:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat >> ~/.ssh/authorized_keys << 'EOF'
<paste your public key here — one line>
EOF
chmod 600 ~/.ssh/authorized_keys
```

### Method 3: Remote Command (If You Have Temporary Password Access)

```bash
# Copy via pipe (no ssh-copy-id needed)
cat ~/.ssh/id_ed25519_proxmox.pub | \
  ssh dave@192.168.1.254 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

## ssh-agent — Key Management Without Repeated Passphrase Entry

```bash
# Start ssh-agent (usually auto-started by your desktop/shell)
eval "$(ssh-agent -s)"

# Add your key (prompts for passphrase once)
ssh-add ~/.ssh/id_ed25519_proxmox

# List loaded keys
ssh-add -l

# Remove a key
ssh-add -d ~/.ssh/id_ed25519_proxmox

# Remove all keys
ssh-add -D

# Keyring integration on Arch (GNOME Keyring or KWallet handles this automatically)
# For systemd-based auto-start, add to ~/.config/systemd/user/ssh-agent.service:
```

```ini
[Unit]
Description=SSH key agent

[Service]
Type=simple
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now ssh-agent
echo 'export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"' >> ~/.bashrc
```

---

## ~/.ssh/config — Client-Side Configuration

Stop typing long ssh commands. The config file gives you shortcuts and per-host settings.

```bash
nvim ~/.ssh/config
chmod 600 ~/.ssh/config
```

```
# Global defaults
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentitiesOnly yes

# Proxmox main host
Host proxmox
    HostName 192.168.1.254
    User root
    IdentityFile ~/.ssh/id_ed25519_proxmox
    Port 22

# Proxmox VM via jump host (agent forwarding)
Host pve-vm-*
    User dave
    ProxyJump proxmox
    IdentityFile ~/.ssh/id_ed25519_proxmox

# Specific VM
Host pve-kali
    HostName 192.168.100.10
    User kali
    ProxyJump proxmox
    IdentityFile ~/.ssh/id_ed25519_proxmox

# CTF challenge host (short TTL, weak ciphers disabled)
Host ctf-*
    User root
    StrictHostKeyChecking accept-new
    IdentityFile ~/.ssh/id_ed25519
```

```bash
# Now connect with:
ssh proxmox               # instead of: ssh -i ~/.ssh/id_ed25519_proxmox root@192.168.1.254
ssh pve-kali              # jumps through Proxmox automatically

# Test config parsing
ssh -G proxmox            # print resolved config for 'proxmox' host
```

---

## Hardening /etc/ssh/sshd_config (CIS/STIG Baseline)

```bash
sudo nvim /etc/ssh/sshd_config
```

### Critical Settings

```bash
# Port — change from default 22 (reduces automated scan noise)
Port 2222                           # or any non-standard port

# Protocol — must be 2
Protocol 2

# Authentication — key only, no passwords
PermitRootLogin no                  # CRITICAL: never allow root SSH
PubkeyAuthentication yes
PasswordAuthentication no           # CRITICAL: disable password auth
ChallengeResponseAuthentication no
UsePAM yes                          # keep PAM for account/session management

# Restrict which users/groups can SSH
AllowUsers dave analyst             # whitelist specific users
AllowGroups ssh-users               # or restrict by group

# Timeouts and limits
LoginGraceTime 30                   # 30s to authenticate before disconnect
MaxAuthTries 3                      # 3 failed attempts before disconnect
MaxSessions 5                       # max concurrent sessions per connection

# Session hardening
ClientAliveInterval 300             # send keepalive every 5 min
ClientAliveCountMax 2               # disconnect after 2 missed keepalives
TCPKeepAlive no                     # disable TCP-level keepalive (use ssh-level)
Compression delayed                 # delay compression until after auth

# Forwarding restrictions
AllowAgentForwarding no             # disable by default (enable per-user if needed)
AllowTcpForwarding no               # disable if tunneling not needed
X11Forwarding no                    # no GUI forwarding (unless you need it)
PermitTunnel no

# Cryptography (modern ciphers only)
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group14-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Banner (legal warning — important for forensics)
Banner /etc/ssh/banner

# Logging
LogLevel VERBOSE                    # log fingerprints and key types
SyslogFacility AUTH

# SFTP subsystem (confined)
Subsystem sftp /usr/lib/ssh/sftp-server
```

```bash
# Create SSH banner
sudo tee /etc/ssh/banner << 'EOF'
***  AUTHORIZED ACCESS ONLY  ***
This system is monitored. Unauthorized access is prohibited and will be prosecuted.
EOF

# Test config before restarting
sudo sshd -t       # exits 0 = valid, prints errors otherwise

# Apply changes
sudo systemctl reload sshd

# Verify settings are active
ssh -v localhost 2>&1 | grep -i "cipher\|auth\|kex\|server version"
```

---

## SSH Jump Hosts for Proxmox Segmentation

```
Arch Laptop → [Internet/LAN] → Proxmox Host (jump) → VM inside Proxmox
```

```bash
# Direct jump host usage (ad-hoc)
ssh -J dave@proxmox-ip:2222 dave@vm-192.168.100.10

# ProxyJump in config (see ~/.ssh/config above)
ssh pve-kali

# SSH tunnel — forward a remote port to localhost
# Example: access Proxmox web UI locally (port 8006)
ssh -L 8006:localhost:8006 proxmox
# Now browse to https://localhost:8006/

# Dynamic SOCKS proxy (route all traffic through Proxmox)
ssh -D 1080 proxmox
# Configure browser/tool to use SOCKS5 at 127.0.0.1:1080

# Reverse tunnel (from Proxmox VM back to your Arch machine)
# Useful when VM is behind NAT
ssh -R 2222:localhost:22 dave@your-arch-ip
# Now from your Arch machine: ssh -p 2222 vm-user@localhost
```

---

## fail2ban — Brute Force Protection

```bash
sudo pacman -S fail2ban

# Basic configuration
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nvim /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 3600        # ban for 1 hour
findtime = 600         # within 10 minutes
maxretry = 3           # allow 3 failures

[sshd]
enabled  = true
port     = 2222        # match your custom SSH port
logpath  = %(syslog_authpriv)s
backend  = systemd
```

```bash
sudo systemctl enable --now fail2ban

# Check ban status
sudo fail2ban-client status sshd
sudo fail2ban-client banned     # list all banned IPs
sudo fail2ban-client set sshd unbanip 192.168.1.50   # unban
```

---

## Practical Lab: Harden Your SSH Right Now

```bash
# 1. Generate a new ED25519 key for Proxmox (don't overwrite existing)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_proxmox -C "proxmox-$(date +%Y%m%d)" -N ""
# Use a passphrase in production! The -N "" skips it for lab testing only

# 2. Audit current sshd_config
grep -E "^[^#]" /etc/ssh/sshd_config | sort

# 3. Check what authentication methods your sshd accepts
nmap --script=ssh-auth-methods -p 22 localhost

# 4. Verify no password auth is accepted
ssh -o PasswordAuthentication=no -o PubkeyAuthentication=no localhost 2>&1 | \
  grep -i "permission\|auth"
# Should fail with "Permission denied (publickey)"

# 5. Check active SSH sessions
who
w
ss -tnp sport = :22
```

---

## Troubleshooting Scenario: SSH Key Authentication Fails

**Symptom:** You copied the key with `ssh-copy-id` but still get password prompt.

**Step 1: Verbose client output**
```bash
ssh -vvv dave@192.168.1.254 2>&1 | grep -A2 "Offering\|Trying\|authentication"
# Look for:
# "Offering public key: ~/.ssh/id_ed25519_proxmox"
# "Server accepted public key" OR "Server rejected public key"
```

**Step 2: Check server-side logs**
```bash
# On the remote host:
journalctl -u sshd -f    # watch for authentication attempts in real time
# Look for:
# "Authentication refused: bad ownership or modes for directory /home/dave"
# "Authentication refused: bad ownership or modes for file .ssh/authorized_keys"
```

**Step 3: Permission fix (most common cause)**
```bash
# On the remote host:
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/authorized_keys
chown dave:dave ~/.ssh/ ~/.ssh/authorized_keys

# If home directory is group/world writable — SSH refuses keys
stat -c "%a %U %G" ~/
# Must be 755 or 700, owned by the user
chmod 755 ~/
```

**Step 4: Check authorized_keys format**
```bash
cat ~/.ssh/authorized_keys
# Must be one key per line, no extra whitespace, correct format:
# ssh-ed25519 AAAA... comment

# If corrupted, re-copy
ssh-copy-id -i ~/.ssh/id_ed25519_proxmox.pub dave@192.168.1.254
```

---

## 🏁 Proof of Work — Phase 4.3 Mini-CTF

> [!example] Challenge: SSH Security Audit
>
> ```bash
> # Audit your SSH configuration against CIS benchmarks
> {
>   echo "=== SSH SECURITY AUDIT ==="
>   echo "Date: $(date -u)"
>   echo "Host: $(hostname)"
>   echo ""
>
>   echo "--- sshd Configuration (active settings) ---"
>   sudo sshd -T 2>/dev/null | grep -E \
>     "permitrootlogin|passwordauthentication|pubkeyauthentication|\
> x11forwarding|allowtcpforwarding|logLevel|port|maxauthtries"
>
>   echo ""
>   echo "--- CIS Benchmark Checks ---"
>
>   check() {
>     local setting="$1" expected="$2"
>     actual=$(sudo sshd -T 2>/dev/null | grep "^${setting}" | awk '{print $2}')
>     if [ "$actual" = "$expected" ]; then
>       echo "[PASS] $setting = $actual"
>     else
>       echo "[FAIL] $setting = $actual (expected: $expected)"
>     fi
>   }
>
>   check "permitrootlogin" "no"
>   check "passwordauthentication" "no"
>   check "pubkeyauthentication" "yes"
>   check "x11forwarding" "no"
>   check "maxauthtries" "3"
>
>   echo ""
>   echo "--- Authorized Keys Audit ---"
>   for home in /home/* /root; do
>     keyfile="${home}/.ssh/authorized_keys"
>     if [ -f "$keyfile" ]; then
>       count=$(grep -c "^ssh-" "$keyfile" 2>/dev/null || echo 0)
>       echo "$home: $count key(s)"
>     fi
>   done
>
>   echo ""
>   echo "--- Key Inventory ---"
>   ls -la ~/.ssh/*.pub 2>/dev/null | awk '{print $NF}'
>   for pubkey in ~/.ssh/*.pub; do
>     echo "Fingerprint: $(ssh-keygen -lf "$pubkey" 2>/dev/null)"
>   done
>
> } | tee /tmp/phase4_3_pow.txt
>
> sha256sum /tmp/phase4_3_pow.txt
> ```
>
> **Target:** All CIS checks should show `[PASS]`. Fix any failures before submitting.

---

← [[02-Network-Tools]] | [[../Phase-5-Architect/00-Phase5-Overview|Next: Phase 5 →]]
