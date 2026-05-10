---
aliases:
  - Text Manipulation
  - grep awk sed
  - Log Parsing
  - Log Forensics CLI
tags:
  - linux
  - cli
  - grep
  - awk
  - sed
  - net412
  - net179
  - phase1
date: 2026-05-10
---

# 02 — Text Manipulation: grep, awk, sed, cut

> [!info] Why This Matters
> System logs, network captures, and database dumps are text. The attacker's tracks, the malware's persistence, the pivot point — all text. grep/awk/sed are your scalpel. Blunt analysts use GUIs. You use pipes.

## grep — Global Regular Expression Print

grep filters lines. Master it first.

```bash
# Basic pattern match
grep "Failed password" /var/log/auth.log

# Case-insensitive
grep -i "error" /var/log/syslog

# Invert match (lines NOT containing pattern)
grep -v "^#" /etc/ssh/sshd_config       # strip comments from config files

# Count matches
grep -c "Failed password" /var/log/auth.log

# Show line numbers
grep -n "PermitRootLogin" /etc/ssh/sshd_config

# Show context (lines before/after)
grep -A 3 -B 2 "segfault" /var/log/syslog    # 3 after, 2 before

# Recursive search through directories
grep -r "password" /etc/ 2>/dev/null

# Extended regex (no need to escape +, ?, |, etc.)
grep -E "Failed|Invalid|error" /var/log/auth.log

# Only print the matching part (not the whole line)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' /var/log/auth.log   # extract IPs

# Quiet mode — exit code only (for scripting)
grep -q "PermitRootLogin yes" /etc/ssh/sshd_config && echo "CRITICAL: Root login enabled"
```

> [!warning] Regex vs. Extended Regex
> Basic grep (`grep`) uses BRE (Basic Regular Expressions) — `+` and `?` need backslashes. Use `grep -E` or `egrep` for ERE (Extended), which is cleaner. In scripts, always specify explicitly.

### Real-World: Parse SSH Brute Force Attack

```bash
# Extract attacker IPs from auth log
grep "Failed password" /var/log/auth.log \
  | grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' \
  | sort | uniq -c | sort -rn \
  | head -20
```

Output: Top 20 IPs by brute-force attempt count. Feed this to your firewall.

---

## awk — Columnar Data Extraction & Processing

awk processes structured text field-by-field. Think of it as a lightweight scripting language built for columns.

```bash
# Print specific fields ($1=first, $NF=last, $0=entire line)
awk '{print $1}' /var/log/auth.log          # timestamps only
awk '{print $1, $2, $3}' /var/log/auth.log  # date + time
awk '{print $NF}' file.txt                  # last field of each line

# Custom field separator
awk -F: '{print $1, $3}' /etc/passwd        # username and UID
awk -F: '{print $1}' /etc/passwd            # just usernames
awk -F: '$3 == 0 {print $1}' /etc/passwd   # UID 0 users (should only be root)

# Conditional printing
awk '$3 > 1000 {print $1, $3}' /etc/passwd  # regular users (UID > 1000)

# Arithmetic and aggregation
awk '{sum += $5} END {print "Total bytes:", sum}' access.log

# Pattern + action
awk '/Failed password/ {print $11}' /var/log/auth.log   # IPs from failed logins

# BEGIN and END blocks
awk 'BEGIN {print "=== Failed Logins ==="} \
     /Failed password/ {count++} \
     END {print count, "failed attempts"}' /var/log/auth.log
```

> [!tip] UID 0 Audit
> `awk -F: '$3 == 0 {print $1}' /etc/passwd` — if this outputs anything other than `root`, you have a rootkit or misconfiguration. This is a one-liner NET 179 forensic check.

### Real-World: Parse /etc/passwd for Privilege Escalation Surface

```bash
# Users with shells (not /sbin/nologin or /bin/false)
awk -F: '$7 !~ /nologin|false/ {print $1, $3, $6, $7}' /etc/passwd

# Service accounts with valid shells (should not exist)
awk -F: '$3 < 1000 && $7 !~ /nologin|false|sync/ {print "SUSPICIOUS:", $1, $7}' /etc/passwd
```

### Real-World: Analyze Apache/Nginx Access Log

```bash
# Top 10 requesting IPs
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# All POST requests (potential data exfil or injection)
awk '$6 ~ /POST/ {print $1, $7, $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 4xx and 5xx error counts
awk '$9 >= 400 {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

---

## sed — Stream Editor for In-Place Text Transformation

sed modifies text as it flows through — substitution, deletion, insertion.

```bash
# Basic substitution (first occurrence per line)
sed 's/foo/bar/' file.txt

# Global substitution (all occurrences)
sed 's/foo/bar/g' file.txt

# Case-insensitive substitution
sed 's/error/ERROR/gi' file.txt

# In-place edit (modify the file directly)
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# In-place edit with backup
sed -i.bak 's/old/new/g' config.conf       # creates config.conf.bak

# Delete lines matching pattern
sed '/^#/d' /etc/ssh/sshd_config           # remove comment lines
sed '/^$/d' file.txt                       # remove blank lines

# Print specific line numbers
sed -n '10,20p' /var/log/syslog            # lines 10-20

# Print lines matching pattern (like grep but sed)
sed -n '/Failed password/p' /var/log/auth.log

# Delete lines in range
sed '5,10d' file.txt
```

> [!tip] Hardening Configs with sed
> ```bash
> # Disable root SSH login non-interactively
> sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
> sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
> systemctl reload sshd
> ```
> This pattern is used in CIS/STIG baseline automation scripts (NET 412 / NET 210).

---

## cut — Column Extraction by Delimiter

cut is simpler than awk — use it when you just need specific columns.

```bash
# Cut by delimiter
cut -d: -f1 /etc/passwd             # field 1 (username), colon delimiter
cut -d: -f1,3,7 /etc/passwd         # fields 1, 3, and 7
cut -d',' -f2 data.csv              # CSV second column

# Cut by character position
cut -c1-10 /var/log/syslog          # first 10 characters (timestamps)
cut -c-5 file.txt                   # characters 1 through 5
cut -c10- file.txt                  # character 10 to end of line
```

---

## sort & uniq — De-duplication and Ranking

```bash
# Sort alphabetically
sort /etc/passwd

# Sort numerically on field 3 (UID)
sort -t: -k3 -n /etc/passwd

# Reverse sort
sort -rn numbers.txt

# Unique lines only
sort file.txt | uniq

# Count occurrences (must be sorted first)
sort ips.txt | uniq -c

# Sort by count descending (top frequency analysis)
sort ips.txt | uniq -c | sort -rn

# Show only duplicates
sort file.txt | uniq -d
```

---

## tr — Character Translation

```bash
# Lowercase to uppercase
echo "hello world" | tr 'a-z' 'A-Z'

# Delete characters
echo "remove-dashes-here" | tr -d '-'

# Squeeze repeated characters
echo "aaaaabbbbb" | tr -s 'ab'

# Replace newlines (join lines)
cat file.txt | tr '\n' ','
```

---

## Building Investigation Pipelines

### Pipeline 1: SSH Brute Force Attribution

```bash
grep "Failed password" /var/log/auth.log \
  | awk '{print $(NF-3)}' \
  | sort \
  | uniq -c \
  | sort -rn \
  | head -20 \
  | awk '{printf "%-6s %s\n", $1, $2}'
```

### Pipeline 2: Find Privilege Escalation Events

```bash
journalctl --no-pager \
  | grep -E "sudo|su\[|COMMAND" \
  | awk '{print $1, $2, $3, $0}' \
  | grep -v "pam_unix" \
  | tail -50
```

### Pipeline 3: CTF Log Analysis (Parse Flag Patterns)

```bash
# Find potential flags in captured files (common CTF formats)
strings suspicious_binary | grep -E "flag\{[^}]+\}|CTF\{[^}]+\}|NCL\{[^}]+\}" 

# Hex dump and grep for strings
xxd /tmp/suspicious.bin | grep -A1 "flag"
```

### Pipeline 4: Parse Your Python SQLite DB Queries from Logs

```bash
# If your toolkit logs SQL operations
grep "SELECT\|INSERT\|DELETE\|UPDATE" /var/log/crypto_toolkit.log \
  | awk -F'"' '{print $2}' \
  | sort | uniq -c | sort -rn
```

---

## Practical Lab: Arch Live System

```bash
# 1. Audit all users with interactive shells
awk -F: '$7 !~ /nologin|false/ {print $1, "UID:"$3, "Shell:"$7}' /etc/passwd

# 2. Find the top 5 largest log files
find /var/log -type f -name "*.log" 2>/dev/null \
  | xargs ls -la 2>/dev/null \
  | awk '{print $5, $NF}' \
  | sort -rn \
  | head -5

# 3. Extract all unique processes from journalctl (last boot)
journalctl -b --no-pager \
  | awk '{print $5}' \
  | sed 's/\[.*//' \
  | sort | uniq -c | sort -rn \
  | head -20

# 4. Parse pacman.log for recent installations
grep "installed" /var/log/pacman.log \
  | tail -20 \
  | awk '{print $1, $2, $4, $5}'

# 5. Simulate a credential leak search in /etc
grep -rE "password\s*=\s*['\"]?[^'\"\s]+" /etc/ 2>/dev/null \
  | grep -v "^Binary"
```

---

## Troubleshooting Scenario: awk Produces No Output

**Symptom:** `awk -F: '$3 == 0 {print $1}' /etc/passwd` outputs nothing.

**Diagnosis:**
```bash
# Verify the file has the right format
head -3 /etc/passwd
# Expected: root:x:0:0:root:/root:/bin/bash

# Check if field separator is correct
awk 'NR==1 {print NF}' /etc/passwd   # should print 7

# Try without the condition first
awk -F: '{print $3}' /etc/passwd | head -5   # should show UIDs
```

**Root Cause:** Often a whitespace issue — some files use spaces, not colons. Or the file has Windows line endings (`\r\n`).

```bash
# Fix Windows line endings
sed -i 's/\r//' /etc/passwd           # strip carriage returns

# Or use file to diagnose
file /etc/passwd
```

---

## 🏁 Proof of Work — Phase 1.2 Mini-CTF

> [!example] Challenge: Extract the Intruder's IP
> A simulated auth log is embedded below. Use grep + awk + sort to identify the top attacking IP and the targeted username.
>
> ```bash
> # Create the simulated log
> cat << 'EOF' > /tmp/fake_auth.log
> May 10 02:01:15 archbox sshd[1234]: Failed password for root from 10.0.0.99 port 44231 ssh2
> May 10 02:01:17 archbox sshd[1234]: Failed password for admin from 10.0.0.99 port 44232 ssh2
> May 10 02:01:19 archbox sshd[1234]: Failed password for root from 10.0.0.99 port 44233 ssh2
> May 10 02:01:21 archbox sshd[1235]: Failed password for root from 192.168.1.50 port 55100 ssh2
> May 10 02:01:23 archbox sshd[1234]: Failed password for root from 10.0.0.99 port 44234 ssh2
> May 10 02:01:25 archbox sshd[1234]: Accepted publickey for dave from 192.168.1.10 port 22001 ssh2
> EOF
>
> # Task 1: Top attacker IP (most failed attempts)
> grep "Failed password" /tmp/fake_auth.log \
>   | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' \
>   | sort | uniq -c | sort -rn | head -1
>
> # Task 2: Most targeted username
> grep "Failed password" /tmp/fake_auth.log \
>   | awk '{print $9}' \
>   | sort | uniq -c | sort -rn | head -1
>
> # Task 3: Who successfully authenticated?
> grep "Accepted" /tmp/fake_auth.log | awk '{print $9}'
> ```
>
> **Expected Answers:** Top IP = `10.0.0.99` (4 attempts), Top Target = `root` (4 attempts), Accepted = `dave`

---

← [[01-CLI-Survival]] | Next: [[03-FHS-Forensics]] →
