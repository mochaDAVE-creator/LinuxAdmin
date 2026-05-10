---
aliases:
  - Permissions and ACLs
  - chmod chown
  - Linux Access Control
  - DAC
tags:
  - linux
  - permissions
  - acl
  - chmod
  - net412
  - net210
  - net377
  - phase2
date: 2026-05-10
---

# 01 — Permissions & ACLs

> [!info] Why This Matters
> Discretionary Access Control (DAC) is the primary gate in Linux. But standard `rwx` permissions have a blind spot: they only model one owner and one group. ACLs extend this to granular multi-party access. Getting this wrong is the root cause of most privilege escalation paths in CTFs and real networks.

## The Permission Model

Every file and directory has three permission triplets: **owner (u)**, **group (g)**, **others (o)**.

```
-rwxr-xr--  1  dave  forensics  4096  May 10 02:00  toolkit.py
│└──┴──┴──      │     │
│  u  g  o      │     └── group name
│               └── owner name
└── file type: - (file), d (dir), l (symlink), b (block), c (char), p (pipe), s (socket)
```

| Symbol | Octal | File Meaning | Directory Meaning |
|---|---|---|---|
| `r` | 4 | Read file contents | List directory contents |
| `w` | 2 | Modify file | Create/delete files inside |
| `x` | 1 | Execute file | Traverse (cd into) directory |
| `-` | 0 | Permission denied | Permission denied |

**Octal notation:** each triplet is 3 bits → one digit.
- `rwx` = 4+2+1 = **7**
- `r-x` = 4+0+1 = **5**
- `r--` = 4+0+0 = **4**
- `---` = 0+0+0 = **0**

```
chmod 750 script.sh
# owner: 7 = rwx  (can do everything)
# group: 5 = r-x  (can read and execute)
# other: 0 = ---  (locked out)
```

---

## chmod — Change Mode

```bash
# Symbolic (easier to read)
chmod u+x script.sh         # add execute for owner
chmod g-w sensitive.db      # remove write for group
chmod o-r private.txt       # remove read for others
chmod a-x notascript.txt    # remove execute for all
chmod u=rwx,g=rx,o= file   # set explicit permissions

# Octal (faster, precise)
chmod 755 script.sh         # rwxr-xr-x
chmod 644 readme.md         # rw-r--r--
chmod 600 ~/.ssh/id_ed25519 # rw------- (SSH key must be this)
chmod 700 ~/.ssh/            # rwx------ (SSH dir must be this)
chmod 000 blackhole.txt      # --------- (nobody can do anything)

# Recursive
chmod -R 750 /opt/myapp/    # all files and dirs recursively

# Apply to directories only (use find)
find /opt/myapp -type d -exec chmod 750 {} \;
find /opt/myapp -type f -exec chmod 640 {} \;
```

> [!warning] Recursive chmod is Dangerous
> `chmod -R 755 /some/dir` will set execute bits on regular files, not just directories. That's a security issue. Use `find` + `-exec` to separate file and directory permission setting.

---

## chown — Change Ownership

```bash
# Change owner
chown dave file.txt

# Change owner and group
chown dave:forensics file.txt

# Change group only
chown :forensics file.txt
# or
chgrp forensics file.txt

# Recursive
chown -R dave:forensics /opt/myapp/

# Change ownership to match another file
chown --reference=/etc/passwd newfile.txt
```

---

## Special Permission Bits

Beyond rwx, three special bits control elevated behaviors.

### SUID (Set User ID) — Bit 4000

When set on an **executable**, it runs as the file's **owner**, not the invoking user.

```bash
chmod u+s /usr/bin/someutil   # symbolic
chmod 4755 /usr/bin/someutil  # octal (4 prefix = SUID)

# Identify SUID binaries (NET 377 — privilege escalation audit)
find / -perm -4000 -type f 2>/dev/null
```

> [!danger] SUID + World-Writable = Instant Privesc
> A SUID binary that is also world-writable can be replaced by an attacker. This is a critical misconfiguration:
> ```bash
> find / -perm -4000 -perm -002 -type f 2>/dev/null
> ```
> Zero results = good. Any result = investigate immediately.

### SGID (Set Group ID) — Bit 2000

On an **executable**: runs with the file's **group** privileges.
On a **directory**: new files inherit the directory's group (essential for team shares).

```bash
chmod g+s /shared/team/       # new files inherit 'team' group
chmod 2770 /shared/team/      # owner: rwx, group: rwx, SGID set
```

### Sticky Bit — Bit 1000

On a **directory**: only the file **owner** (or root) can delete files inside. Classic use: `/tmp`.

```bash
chmod +t /shared/public/
chmod 1777 /tmp              # world-writable + sticky (the /tmp standard)

# Verify
ls -la /tmp | head -3
# drwxrwxrwt   # 't' at the end = sticky bit set
```

---

## umask — Default Permission Mask

umask defines which permissions are **removed** from newly created files/directories.

```bash
# View current umask
umask           # shows e.g. 0022
umask -S        # symbolic: u=rwx,g=rx,o=rx

# New file permissions = 0666 - umask (files never get execute by default)
# New dir permissions  = 0777 - umask

# umask 0022:
# Files: 0666 - 0022 = 0644 (rw-r--r--)
# Dirs:  0777 - 0022 = 0755 (rwxr-xr-x)

# Set restrictive umask for sensitive work
umask 0077      # Files: 0600, Dirs: 0700 (owner only)

# Set in shell profile for persistence
echo 'umask 0027' >> ~/.bashrc   # Files: 640, Dirs: 750
```

> [!tip] CIS Benchmark — umask
> CIS Benchmark NET 412 requires umask 027 or more restrictive for all users. Set it in `/etc/profile` and `/etc/bash.bashrc` for system-wide enforcement.

---

## stat — Inspect Permissions in Detail

```bash
stat /etc/shadow
# Output includes:
#   File: /etc/shadow
#   Size: 1245   Blocks: 8   IO Block: 4096
#   Access: (0640/-rw-r-----)  Uid: (0/root)  Gid: (42/shadow)
#   Access: 2026-05-09 22:14:33.000000000
#   Modify: 2026-05-09 22:14:33.000000000
#   Change: 2026-05-09 22:14:33.000000000
```

The `Change` time (ctime) updates when permissions change — this is how you detect permission tampering even if the attacker modified the timestamp.

---

## ACLs — Access Control Lists

Standard DAC only supports one owner and one group. ACLs let you grant specific permissions to **any user or group**.

```bash
# Install ACL tools (Arch: usually pre-installed)
pacman -S acl

# Check if filesystem supports ACLs (Btrfs does by default)
tune2fs -l /dev/sda1 | grep "Default mount options"
# Or check mount options:
findmnt -o TARGET,OPTIONS | grep acl

# Get current ACLs
getfacl /etc/passwd
getfacl /opt/myapp/database.db

# Set ACL — give user 'ctf_analyst' read access to a file only root can read
setfacl -m u:ctf_analyst:r /var/log/auth.log

# Give group 'forensics' read+write on a directory
setfacl -m g:forensics:rw /mnt/evidence/

# Default ACL (applied to newly created files in directory)
setfacl -d -m g:forensics:rw /mnt/evidence/

# Remove a specific ACL entry
setfacl -x u:ctf_analyst /var/log/auth.log

# Remove ALL ACLs
setfacl -b /var/log/auth.log

# Recursive ACL set
setfacl -R -m g:forensics:rx /opt/myapp/
```

> [!info] ACL Indicator
> When a file has ACLs set, `ls -la` shows a `+` at the end of the permission string:
> ```
> -rw-r-----+  1 root shadow 1245 May 10 02:00 /etc/shadow
> ```
> The `+` means "additional ACL entries exist — use getfacl to see them."

### Real-World: Shared Evidence Directory

```bash
# Create a shared forensics evidence directory
# Multiple analysts can write, but only owners can delete their own files

groupadd forensics
usermod -aG forensics dave
usermod -aG forensics analyst2

mkdir -p /mnt/evidence/case01
chown root:forensics /mnt/evidence/case01
chmod 2770 /mnt/evidence/case01    # SGID + group rwx

# Set default ACL so new files are group-accessible
setfacl -d -m g:forensics:rw /mnt/evidence/case01

# Verify
getfacl /mnt/evidence/case01
```

---

## Practical Lab: Permission Hardening Audit

```bash
# 1. Find world-writable files (not directories, not symlinks)
find / -xdev -perm -002 -type f 2>/dev/null | grep -v "/proc\|/sys"

# 2. Find SUID/SGID binaries and compare against known-good list
find / -perm -4000 -o -perm -2000 2>/dev/null | sort > /tmp/suid_sgid_current.txt
# On Arch, expected SUID binaries include: sudo, ping, newgrp, pkexec, etc.
cat /tmp/suid_sgid_current.txt

# 3. Audit SSH directory permissions (must be exact or SSH refuses to work)
stat -c "%a %n" ~/.ssh ~/.ssh/*

# 4. Check /etc/shadow permissions (should be 640, owned root:shadow)
stat /etc/shadow

# 5. Find files without a valid owner (orphaned — forensics red flag)
find / -nouser -o -nogroup 2>/dev/null | grep -v "/proc\|/sys"

# 6. Test ACL setup
getfacl /etc/passwd
getfacl /home
```

---

## Troubleshooting Scenario: "Permission Denied" on tcpdump

**Symptom:** `tcpdump -i eth0` returns `tcpdump: eth0: You don't have permission to capture on that device`

**Root Cause Analysis:**

```bash
# Check what capability tcpdump needs
getcap $(which tcpdump)
# Expected: /usr/bin/tcpdump = cap_net_admin,cap_net_raw+eip

# If it has no capability:
ls -la $(which tcpdump)
# Check if it's SUID root (older method)
```

**Fix Option 1: Linux Capabilities (preferred, least privilege)**
```bash
# Grant tcpdump raw network capture capability without SUID
setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)

# Verify
getcap $(which tcpdump)

# Now run as regular user
tcpdump -i eth0 -c 10
```

**Fix Option 2: Add user to wireshark/pcap group**
```bash
groupadd pcap
chgrp pcap $(which tcpdump)
chmod 750 $(which tcpdump)
setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)
usermod -aG pcap dave
```

> [!tip] Capabilities vs SUID
> Linux capabilities are the CIS/STIG-approved method. SUID root on tcpdump gives it full root privileges. Capabilities give it only what it needs. Use `getcap -r / 2>/dev/null` to audit all capability-enabled binaries.

---

## 🏁 Proof of Work — Phase 2.1 Mini-CTF

> [!example] Challenge: Secure a Team Workspace with ACLs
>
> Set up a shared analysis directory that satisfies these requirements:
> 1. Root owns the directory
> 2. Group `forensics` has read/write/execute
> 3. Others have no access
> 4. New files created inside inherit group ownership
> 5. User `dave` has explicit write access even if not in the `forensics` group
>
> ```bash
> # Setup
> sudo groupadd forensics 2>/dev/null
> sudo mkdir -p /tmp/ctf_workspace
>
> # Your solution here:
> sudo chown root:forensics /tmp/ctf_workspace
> sudo chmod 2770 /tmp/ctf_workspace
> sudo setfacl -m u:dave:rwx /tmp/ctf_workspace
> sudo setfacl -d -m g:forensics:rwx /tmp/ctf_workspace
> sudo setfacl -d -m u:dave:rwx /tmp/ctf_workspace
>
> # Validation
> getfacl /tmp/ctf_workspace
> stat -c "%a %U %G" /tmp/ctf_workspace
> touch /tmp/ctf_workspace/test_file
> getfacl /tmp/ctf_workspace/test_file    # should show inherited ACLs
> ```
>
> **Success criteria:** `getfacl` shows your ACLs. `stat` shows `2770`. New files inherit the group ACL.

---

← [[00-Phase2-Overview|Phase 2 Overview]] | Next: [[02-User-Group-Management]] →
