---
aliases:
  - Phase 4 Overview
  - The Communicator
  - Networking Phase
tags:
  - linux
  - networking
  - net412
  - net377
  - net210
  - phase4
date: 2026-05-10
---

# Phase 4 — The Communicator

> [!info] Phase Objective
> Every exploit, every pivot, every exfil, and every legitimate service depends on the network stack. This phase gives you command of Linux networking from the interface level to the application layer — and the ability to see every packet moving through your Proxmox lab.

## Modules

1. [[01-IP-Routing|01 — IP Routing & Interfaces]] — ip, nmcli, routing tables, VLAN configuration
2. [[02-Network-Tools|02 — ss, netstat, nmap, tcpdump, wireshark]] — Port enumeration, traffic analysis, footprinting
3. [[03-SSH-Hardening|03 — SSH Hardening & Proxmox Key Auth]] — Key-based auth, sshd_config hardening, jump hosts

## Phase Competencies

By the end of Phase 4 you will be able to:

- Configure network interfaces and static routes without a GUI
- Map every open port and connection on any system you have access to
- Capture and analyze raw packet traffic for forensics or penetration testing
- Lock down SSH to key-only, rate-limited, non-default-port configuration
- Set up secure Proxmox node-to-node communication via SSH tunnels

## Cross-Phase Links

- Prerequisites: [[../Phase-1-Core-Alphabet/01-CLI-Survival|Phase 1 — CLI]], [[../Phase-3-Engine-Room/02-Systemd-Deep-Dive|Phase 3 — systemd]] (networking is a systemd service)
- Feeds into: [[../Phase-5-Architect/03-Container-Isolation|Phase 5 — Containers]] (container networking namespaces)

## Degree Alignment

| Course | Competency Covered |
|---|---|
| NET 412 | Network interfaces, routing, SSH, firewall basics |
| NET 377 | Network footprinting, enumeration, traffic interception |
| NET 210 | Network traffic analysis, IDS, security monitoring |

---

← [[../README|Back to MoC]] | Next: [[01-IP-Routing]] →
