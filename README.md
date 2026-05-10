---
aliases:
  - MoC
  - Map of Content
  - Linux Admin Vault Index
tags:
  - linux
  - moc
  - net412
  - net377
  - net210
  - net179
  - obsidian
date: 2026-05-10
---

# 🗺️ Linux Administration — Zero to Hero Vault (MoC)

> [!tip] How to Use This Vault
> This is a living curriculum designed for an **Arch Linux / Proxmox / CTF** workflow. Every module is wired to real commands you can run right now. Start at Phase 1 and work forward — or jump to the phase that matches your current blocker.

## Identity & Context

| Attribute | Detail |
|---|---|
| **Primary OS** | Arch Linux (bare metal, Btrfs + Snapper rollback) |
| **Lab** | Proxmox hypervisor (local), Distrobox for CTF tool isolation |
| **CTF Role** | Forensics Lead — NCL competitive team |
| **Active Projects** | Python cryptography toolkit (SQLite/Tkinter), Rust/Solidity blockchain engine (Apex Predator PMO) |
| **Degree Alignment** | NET 412 (Linux Admin), NET 377 (Ethical Hacking), NET 210 (Security Analyst), NET 179 (Digital Forensics) |

---

## Curriculum Phases

```
Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4 ──► Phase 5
 CLI/FHS    Perms/ACLs  Processes   Networking  Arch/Containers
```

---

## [[Phase-1-Core-Alphabet/00-Phase1-Overview|Phase 1 — The Core Alphabet]]

> Master the terminal before you touch anything else. CLI fluency is the prerequisite for every other phase.

| Module | Topic | Degree Alignment |
|---|---|---|
| [[Phase-1-Core-Alphabet/01-CLI-Survival|01 — CLI Survival]] | Navigation, file ops, pipes, redirection | NET 412, NET 377 |
| [[Phase-1-Core-Alphabet/02-Text-Manipulation|02 — Text Manipulation]] | grep, awk, sed, cut — log parsing for forensics | NET 179, NET 412 |
| [[Phase-1-Core-Alphabet/03-FHS-Forensics|03 — FHS & Forensics Artifact Hunting]] | Filesystem Hierarchy, artifact locations, evidence trails | NET 179, NET 412 |

---

## [[Phase-2-Blueprint/00-Phase2-Overview|Phase 2 — The Blueprint]]

> Permissions, identities, and ownership. Get this wrong and every project (including your Python SQLite DB) is a security liability.

| Module | Topic | Degree Alignment |
|---|---|---|
| [[Phase-2-Blueprint/01-Permissions-ACLs|01 — Permissions & ACLs]] | DAC, chmod/chown, setfacl, getfacl | NET 412, NET 210 |
| [[Phase-2-Blueprint/02-User-Group-Management|02 — User & Group Management]] | useradd, passwd, /etc/shadow, sudo hardening | NET 412, NET 377 |
| [[Phase-2-Blueprint/03-Securing-Python-SQLite|03 — Securing Your Python SQLite DB]] | File permissions, encryption at rest, AppArmor profile | NET 412, NET 210 |

---

## [[Phase-3-Engine-Room/00-Phase3-Overview|Phase 3 — The Engine Room]]

> systemd is the init system, service supervisor, and log aggregator. Learn it or be blind to your own infrastructure.

| Module | Topic | Degree Alignment |
|---|---|---|
| [[Phase-3-Engine-Room/01-Process-Management|01 — Process Management]] | ps, top, htop, nice, kill, /proc | NET 412, NET 377 |
| [[Phase-3-Engine-Room/02-Systemd-Deep-Dive|02 — systemd Deep Dive]] | Units, targets, timers, writing service files | NET 412 |
| [[Phase-3-Engine-Room/03-Journalctl-Troubleshooting|03 — journalctl & Log Forensics]] | Parsing logs, boot analysis, Rust engine service triage | NET 412, NET 179 |

---

## [[Phase-4-Communicator/00-Phase4-Overview|Phase 4 — The Communicator]]

> Every packet matters. Understand routing, sniff traffic, and lock down Proxmox node communication.

| Module | Topic | Degree Alignment |
|---|---|---|
| [[Phase-4-Communicator/01-IP-Routing|01 — IP Routing & Interfaces]] | ip, nmcli, routing tables, VLAN basics | NET 412, NET 377 |
| [[Phase-4-Communicator/02-Network-Tools|02 — ss, netstat, nmap, tcpdump]] | Port enumeration, traffic capture, footprinting | NET 377, NET 210 |
| [[Phase-4-Communicator/03-SSH-Hardening|03 — SSH Hardening & Proxmox Key Auth]] | Key-based auth, sshd_config, agent forwarding, jump hosts | NET 412, NET 210 |

---

## [[Phase-5-Architect/00-Phase5-Overview|Phase 5 — The Architect]]

> Manage packages, take atomic snapshots, and spin up isolated workloads. This is your Arch/Proxmox superpower stack.

| Module | Topic | Degree Alignment |
|---|---|---|
| [[Phase-5-Architect/01-Pacman-Database|01 — Pacman & AUR Management]] | pacman, paru/yay, package signing, database repair | NET 412 |
| [[Phase-5-Architect/02-Btrfs-Snapshots|02 — Btrfs & Snapper Rollbacks]] | Subvolumes, snapshots, rollback after bad upgrade | NET 412 |
| [[Phase-5-Architect/03-Container-Isolation|03 — Docker/Podman for Isolated Workloads]] | Rootless Podman, Distrobox CTF containers, security namespaces | NET 412, NET 210 |

---

## Capstone Projects — Lab Runbook Matrix

> [!info]
> The **Projects** track is a parallel evidence-first, rollback-aware lab curriculum mapped to the five curriculum phases above. Each project produces a hashed evidence bundle. See [[projects/PROJECT_INDEX|Project Index]] for the full matrix.

### Beginner Track

| ID | Project | Key Skills | Rollback |
|---|---|---|---|
| **B1** | [[projects/01-beginner/B1-Snapshot-Rollback-Drill\|Snapshot & Rollback Drill]] | Btrfs, Snapper, `snapper undochange` | Snapper |
| **B2** | [[projects/01-beginner/B2-FHS-Artifact-Hunt\|FHS Artifact Hunt]] | FHS, `find`, `stat`, forensic locations | None (read-only) |
| **B3** | [[projects/01-beginner/B3-SSH-Key-Only-Admin-Access\|SSH Key-Only Admin Access]] | `ssh-keygen`, `sshd_config`, `ssh-copy-id` | Snapper + config backup |
| **B4** | [[projects/01-beginner/B4-Log-Triage-Starter\|Log Triage Starter]] | `journalctl`, `grep`, `awk`, `sed` | None (read-only) |

### Intermediate Track

| ID | Project | Key Skills | Rollback |
|---|---|---|---|
| **I1** | [[projects/02-intermediate/I1-Systemd-Service-Hardening\|Systemd Service Hardening]] | `systemd-analyze security`, drop-in units | Snapper |
| **I2** | [[projects/02-intermediate/I2-Persistence-Detection-Sweep\|Persistence Detection Sweep]] | `find`, `pacman -Qo`, cron, MITRE TA0003 | None (read-only) |
| **I3** | [[projects/02-intermediate/I3-ACL-Backed-Secret-Store\|ACL-Backed Secret Store]] | `setfacl`, `getfacl`, service users | Snapper |
| **I4** | [[projects/02-intermediate/I4-Proxmox-Backup-Validation\|Proxmox Backup Validation]] | `vzdump`, `qmrestore`, backup→restore cycle | Proxmox VM snapshot |

### Advanced Track

| ID | Project | Key Skills | Rollback |
|---|---|---|---|
| **A1** | [[projects/03-advanced/A1-Centralized-Logging-Pipeline\|Centralized Logging Pipeline]] | `systemd-journal-remote`, `journal-upload` | Snapper |
| **A2** | [[projects/03-advanced/A2-Network-Segmentation-Enforcement\|Network Segmentation Enforcement]] | Proxmox bridges, `nftables`, vmbr isolation | Proxmox VM snapshot |
| **A3** | [[projects/03-advanced/A3-Threat-Informed-Detection-Pack\|Threat-Informed Detection Pack]] | `auditd`, MITRE ATT&CK, detection→simulate→verify | Snapper |
| **A4** | [[projects/03-advanced/A4-Container-Isolation-Benchmark\|Container Isolation Benchmark]] | rootless Podman, Distrobox, seccomp, capabilities | Snapper |

### Expert Track

| ID | Project | Key Skills | Rollback |
|---|---|---|---|
| **E1** | [[projects/04-expert/E1-IR-Playbook-Automation\|IR Playbook Automation]] | `ir-triage.sh`, ATT&CK mapping, SHA-256 manifest | None (read-only) |
| **E2** | [[projects/04-expert/E2-Zero-Trust-Homelab-Blueprint\|Zero-Trust Homelab Blueprint]] | SSHCA, short-lived certs, per-segment nftables | Snapper + VM snapshot |
| **E3** | [[projects/04-expert/E3-Secure-Platform-Baseline\|Secure Platform Baseline]] | `lynis`, sysctl hardening, CIS Benchmark | Snapper |
| **E4** | [[projects/04-expert/E4-Purple-Team-Validation-Cycle\|Purple-Team Validation Cycle]] | Adversary emulation, detection validation, ATT&CK | VM snapshot (mandatory) |

---

## Quick-Reference Index

### By Tool
| Tool | Module |
|---|---|
| `grep` / `awk` / `sed` | [[Phase-1-Core-Alphabet/02-Text-Manipulation]] |
| `chmod` / `setfacl` | [[Phase-2-Blueprint/01-Permissions-ACLs]] |
| `useradd` / `sudo` | [[Phase-2-Blueprint/02-User-Group-Management]] |
| `ps` / `kill` / `htop` | [[Phase-3-Engine-Room/01-Process-Management]] |
| `systemctl` / `journalctl` | [[Phase-3-Engine-Room/02-Systemd-Deep-Dive]], [[Phase-3-Engine-Room/03-Journalctl-Troubleshooting]] |
| `ip` / `nmcli` | [[Phase-4-Communicator/01-IP-Routing]] |
| `ss` / `nmap` / `tcpdump` | [[Phase-4-Communicator/02-Network-Tools]] |
| `ssh` / `sshd` | [[Phase-4-Communicator/03-SSH-Hardening]] |
| `pacman` / `paru` | [[Phase-5-Architect/01-Pacman-Database]] |
| `btrfs` / `snapper` | [[Phase-5-Architect/02-Btrfs-Snapshots]] |
| `podman` / `docker` | [[Phase-5-Architect/03-Container-Isolation]] |

### By Degree Course
| Course | Relevant Modules |
|---|---|
| **NET 412** (Linux Admin) | All modules |
| **NET 377** (Ethical Hacking) | [[Phase-1-Core-Alphabet/01-CLI-Survival]], [[Phase-2-Blueprint/02-User-Group-Management]], [[Phase-3-Engine-Room/01-Process-Management]], [[Phase-4-Communicator/02-Network-Tools]] |
| **NET 210** (Security Analyst) | [[Phase-2-Blueprint/01-Permissions-ACLs]], [[Phase-4-Communicator/02-Network-Tools]], [[Phase-4-Communicator/03-SSH-Hardening]], [[Phase-5-Architect/03-Container-Isolation]] |
| **NET 179** (Digital Forensics) | [[Phase-1-Core-Alphabet/02-Text-Manipulation]], [[Phase-1-Core-Alphabet/03-FHS-Forensics]], [[Phase-3-Engine-Room/03-Journalctl-Troubleshooting]] |

---

## Vault Conventions

> [!note] Callout Legend
> - `> [!tip]` — Best practice or pro technique
> - `> [!warning]` — Common failure mode or security trap
> - `> [!danger]` — Destructive command — verify before running
> - `> [!info]` — Contextual background
> - `> [!example]` — CTF/forensics applied scenario

**Proof of Work** challenges end every module. Complete them in order — they chain together across phases.

---

*Vault initialized: 2026-05-10 | Arch Linux / Proxmox / NCL-CTF edition*
