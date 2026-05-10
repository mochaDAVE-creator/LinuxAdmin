---
aliases:
  - Network Tools
  - ss netstat nmap tcpdump
  - Network Enumeration
  - Packet Capture
tags:
  - linux
  - networking
  - nmap
  - tcpdump
  - wireshark
  - net377
  - net210
  - phase4
date: 2026-05-10
---

# 02 — ss, netstat, nmap, tcpdump, wireshark

> [!info] Why This Matters
> Reconnaissance starts with enumeration. Whether you're a forensics analyst checking for C2 connections or a penetration tester mapping a target, these tools are your eyes on the network. Know them at the level where you can interpret raw output without a GUI.

## ss — Socket Statistics (Modern netstat)

`ss` queries the kernel's socket table directly — faster and more detailed than netstat.

```bash
# All listening ports (TCP + UDP)
ss -tlnup
# -t = TCP  -u = UDP  -l = listening  -n = numeric (no DNS)  -p = show process

# All established connections
ss -tnp state established

# Connection summary
ss -s

# Find what's listening on a specific port
ss -tlnp sport = :22     # what's on port 22?
ss -tlnp dport = :443    # what's connecting to port 443?

# Show all connections to a specific host
ss -tnp dst 192.168.1.1

# Unix domain sockets
ss -xlnp

# UDP connections (stateless — shows UNCONN)
ss -ulnp

# Full output with timer information
ss -tnop state established
```

### Reading ss Output

```
Netid  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
tcp    LISTEN  0       128     0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=500,fd=3))
tcp    ESTAB   0       0       192.168.1.100:22     192.168.1.50:55123 users:(("sshd",pid=1200,fd=4))
```

| Column | Meaning |
|---|---|
| State | LISTEN / ESTAB / TIME-WAIT / CLOSE-WAIT |
| Recv-Q | Bytes received but not read by app (growing = app overloaded) |
| Send-Q | Bytes queued to send (growing = network congestion) |
| 0.0.0.0:22 | Listening on all interfaces, port 22 |
| 127.0.0.1:5432 | Listening only on localhost |

> [!tip] Forensic Connection Audit
> ```bash
> ss -tnp state established | awk 'NR>1 {print $5, $6}' | column -t
> ```
> This shows all live TCP connections with the remote IP. Anything connecting to an unexpected external IP is suspicious.

---

## nmap — Network Mapper (NET 377)

nmap is the industry-standard network scanner. Know it from the CTF attacker side and the defender side.

> [!warning] Legal Notice
> Only scan networks and systems you own or have explicit written permission to scan. Unauthorized scanning is illegal. Your Proxmox lab = fair game. Your college network = requires permission.

### Host Discovery

```bash
# Ping scan — is it alive?
nmap -sn 192.168.1.0/24              # no port scan, just ICMP + ARP
nmap -sn 192.168.1.0/24 --open      # only report alive hosts

# Disable ping (scan even if ICMP blocked)
nmap -Pn 192.168.1.100

# ARP scan (most reliable on local network)
sudo nmap -PR -sn 192.168.1.0/24
```

### Port Scanning

```bash
# TCP SYN scan (requires root — sends raw packets, stealthy)
sudo nmap -sS 192.168.1.100

# TCP Connect scan (no root needed — uses full TCP handshake)
nmap -sT 192.168.1.100

# UDP scan (slow — each port needs a timeout)
sudo nmap -sU --top-ports 100 192.168.1.100

# Specific port ranges
nmap -p 22,80,443,8080 192.168.1.100    # specific ports
nmap -p 1-1024 192.168.1.100             # range
nmap -p- 192.168.1.100                   # all 65535 ports (slow)
nmap --top-ports 1000 192.168.1.100      # top 1000 most common

# Combine host + port discovery
sudo nmap -sS -sU --top-ports 100 192.168.1.0/24
```

### Service and Version Detection

```bash
# Service version detection
nmap -sV 192.168.1.100
# Reports: 22/tcp open  ssh  OpenSSH 9.6 (protocol 2.0)

# OS detection (requires root)
sudo nmap -O 192.168.1.100

# Aggressive scan (OS + version + scripts + traceroute)
sudo nmap -A 192.168.1.100

# Version intensity (0=light, 9=try every probe)
nmap -sV --version-intensity 5 192.168.1.100
```

### NSE — Nmap Scripting Engine

```bash
# Run default safe scripts
nmap -sC 192.168.1.100

# Run specific script
nmap --script=ssh-auth-methods 192.168.1.100
nmap --script=http-title 192.168.1.0/24

# Vulnerability scanning scripts
nmap --script=vuln 192.168.1.100
nmap --script=smb-vuln-ms17-010 192.168.1.100   # EternalBlue

# Banner grabbing
nmap --script=banner 192.168.1.100

# List available scripts
ls /usr/share/nmap/scripts/ | grep ssh
```

### Output Formats

```bash
# Normal (human-readable)
nmap -sV 192.168.1.100 -oN scan_results.txt

# XML (machine-parseable)
nmap -sV 192.168.1.100 -oX scan_results.xml

# Grepable (one host per line)
nmap -sV 192.168.1.100 -oG scan_results.gnmap

# All formats simultaneously
nmap -sV 192.168.1.100 -oA scan_results   # creates .nmap, .xml, .gnmap

# Parse grepable output
grep "22/open" scan_results.gnmap | awk '{print $2}'   # IPs with port 22 open
```

---

## tcpdump — Packet Capture (NET 210, NET 179)

tcpdump captures raw packets at the kernel level. Essential for forensics and traffic analysis.

```bash
# List available interfaces
sudo tcpdump -D

# Capture on an interface
sudo tcpdump -i eth0

# Capture to file (PCAP format — open in Wireshark)
sudo tcpdump -i eth0 -w /tmp/capture.pcap

# Read from file
tcpdump -r /tmp/capture.pcap

# Limit capture to N packets
sudo tcpdump -i eth0 -c 100

# Verbose output (show packet contents)
sudo tcpdump -i eth0 -v       # verbose
sudo tcpdump -i eth0 -vvv     # very verbose

# Print as hex + ASCII
sudo tcpdump -i eth0 -X

# Don't resolve hostnames/ports (faster, forensic)
sudo tcpdump -i eth0 -n -nn
```

### BPF Filters — Berkeley Packet Filter

```bash
# Filter by host
sudo tcpdump -i eth0 host 192.168.1.50

# Filter by source/destination
sudo tcpdump -i eth0 src host 192.168.1.50
sudo tcpdump -i eth0 dst host 192.168.1.50

# Filter by port
sudo tcpdump -i eth0 port 22
sudo tcpdump -i eth0 port 80 or port 443

# Filter by protocol
sudo tcpdump -i eth0 icmp
sudo tcpdump -i eth0 udp
sudo tcpdump -i eth0 tcp

# Compound filters
sudo tcpdump -i eth0 host 192.168.1.50 and port 22
sudo tcpdump -i eth0 "src net 192.168.1.0/24 and not port 443"

# Capture SSH traffic to/from Proxmox host
sudo tcpdump -i eth0 host <proxmox-ip> and port 22 -w /tmp/proxmox_ssh.pcap

# Capture all non-SSH traffic (detect unexpected connections)
sudo tcpdump -i eth0 not port 22 -w /tmp/non_ssh.pcap
```

### Real-Time String Extraction

```bash
# Extract printable strings from live traffic (credentials in cleartext)
sudo tcpdump -i eth0 -A -n port 80 | grep -i "password\|login\|user\|pass"

# HTTP GET requests
sudo tcpdump -i eth0 -A -n port 80 | grep "GET\|POST\|Host:"

# FTP credentials (cleartext)
sudo tcpdump -i eth0 port 21 -A | grep -i "user\|pass"
```

> [!warning] Cleartext Credentials
> If `tcpdump -A` on port 80/21/23/110 shows usernames and passwords, you have unencrypted services. This is a critical finding in both CTF and real audits. Document and remediate.

---

## Wireshark — GUI Packet Analysis

For deep packet inspection, Wireshark's dissectors are unmatched. Use tcpdump for capture, Wireshark for analysis.

```bash
# Install
sudo pacman -S wireshark-qt

# Add yourself to wireshark group (capture without root)
sudo usermod -aG wireshark dave
newgrp wireshark    # activate in current session

# Open a capture file
wireshark /tmp/capture.pcap &

# CLI version (tshark)
tshark -r /tmp/capture.pcap
tshark -r /tmp/capture.pcap -Y "tcp.port == 22"   # display filter
tshark -r /tmp/capture.pcap -T fields -e ip.src -e ip.dst -e tcp.dstport \
  | sort | uniq -c | sort -rn | head -20    # connection frequency
```

### Essential Wireshark Display Filters

```
tcp.port == 22                   # SSH traffic
http.request.method == "POST"    # HTTP POST requests
ip.addr == 192.168.1.50          # traffic to/from host
!arp && !dns                     # exclude noise
tcp.flags.syn == 1               # TCP SYN packets (connection attempts)
ssl || tls                       # encrypted traffic
icmp                             # ping traffic
tcp.analysis.retransmission      # retransmissions (connection problems)
```

---

## Practical Lab: Full Network Reconnaissance of Your Proxmox Lab

```bash
# 1. Baseline your own machine's network posture
echo "=== Listening Services ==="
ss -tlnup
echo ""
echo "=== Established Connections ==="
ss -tnp state established

# 2. Scan your local network (Arch host on 192.168.x.x)
LOCAL_NET=$(ip route | grep proto kernel | head -1 | awk '{print $1}')
echo "Scanning: $LOCAL_NET"
sudo nmap -sS -sV --top-ports 20 "$LOCAL_NET" -oA /tmp/local_scan

# 3. Identify all live hosts
grep "Nmap scan report" /tmp/local_scan.nmap | awk '{print $5}'

# 4. Capture 30 seconds of your own traffic
sudo tcpdump -i any -c 500 -w /tmp/self_capture.pcap &
TCPID=$!
sleep 5
ping -c 3 8.8.8.8 > /dev/null
curl -s https://archlinux.org > /dev/null 2>&1
wait $TCPID

# 5. Analyze the capture
tshark -r /tmp/self_capture.pcap -T fields -e ip.dst 2>/dev/null \
  | grep -v "^$\|^[0-9]$" | sort | uniq -c | sort -rn | head -10
```

---

## Troubleshooting Scenario: tcpdump Permission Denied

**Symptom:** `tcpdump: eth0: You don't have permission to capture on that device`

**See:** [[../Phase-2-Blueprint/01-Permissions-ACLs|Phase 2.1 — Permissions]] for the capability-based fix.

**Quick Reference:**
```bash
# Check if capabilities are set
getcap $(which tcpdump)

# Fix: grant raw socket capability (no SUID needed)
sudo setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)

# Verify fix
tcpdump -i eth0 -c 5    # should work as regular user now
```

**Symptom:** `No such device: eth0` — wrong interface name.
```bash
# List your actual interface names
ip link show | grep "^[0-9]"
# Modern naming: eno1, enp3s0, wlp2s0, etc.
tcpdump -D   # list capture-capable interfaces
```

---

## 🏁 Proof of Work — Phase 4.2 Mini-CTF

> [!example] Challenge: Detect and Enumerate a Service
>
> ```bash
> # Part 1: Internal port audit (what services are YOU exposing?)
> echo "=== Services I Am Exposing ==="
> ss -tlnup | awk 'NR>1 {print $1, $5, $NF}' | \
>   grep -v "127.0.0.1\|::1" | \
>   sort -k2
>
> # Part 2: Scan your localhost (safe — you own it)
> nmap -sV -p 1-1024 127.0.0.1 -oN /tmp/localhost_scan.txt
>
> # Part 3: Capture 10 packets and count by protocol
> sudo tcpdump -i lo -c 10 -w /tmp/lo_capture.pcap 2>/dev/null &
> sleep 1
> curl -s http://localhost/ > /dev/null 2>&1
> ping -c 3 localhost > /dev/null
> wait
>
> tshark -r /tmp/lo_capture.pcap -T fields -e frame.protocols 2>/dev/null \
>   | sort | uniq -c | sort -rn | head -10
>
> # Save proof
> cat /tmp/localhost_scan.txt > /tmp/phase4_2_pow.txt
> sha256sum /tmp/phase4_2_pow.txt
> ```
>
> **Submit:** The hash. Bonus: identify any unexpected listening service discovered in Part 1.

---

← [[01-IP-Routing]] | Next: [[03-SSH-Hardening]] →
