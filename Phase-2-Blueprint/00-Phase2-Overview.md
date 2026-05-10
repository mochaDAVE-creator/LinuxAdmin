---
aliases:
  - Phase 2 Overview
  - Blueprint
  - Permissions Phase
tags:
  - linux
  - permissions
  - acl
  - net412
  - net210
  - phase2
date: 2026-05-10
---

# Phase 2 — The Blueprint

> [!info] Phase Objective
> Permissions are the contract between the OS and everything running on it. A misunderstood permission is an open door. This phase gives you precise control over who can read, write, and execute every resource on your system — including your Python SQLite database.

## Modules

1. [[01-Permissions-ACLs|01 — Permissions & ACLs]] — DAC, chmod, chown, setfacl, getfacl, umask
2. [[02-User-Group-Management|02 — User & Group Management]] — useradd, passwd, /etc/shadow, sudo hardening
3. [[03-Securing-Python-SQLite|03 — Securing Your Python SQLite DB]] — File permissions, encryption at rest, AppArmor profile

## Phase Competencies

By the end of Phase 2 you will be able to:

- Read and set any permission mode (octal and symbolic) without a cheat sheet
- Apply fine-grained ACLs beyond the traditional owner/group/other model
- Create isolated users for services with least-privilege principles
- Audit and harden sudo configuration to CIS benchmarks
- Lock down your Python cryptography database so only the application can read it

## Cross-Phase Links

- Prerequisites: [[../Phase-1-Core-Alphabet/01-CLI-Survival|Phase 1 — CLI Survival]] (you need `find`, `ls`, `stat`)
- Feeds into: [[../Phase-3-Engine-Room/02-Systemd-Deep-Dive|Phase 3 — systemd]] (service users need correct permissions)
- Feeds into: [[../Phase-5-Architect/03-Container-Isolation|Phase 5 — Containers]] (container rootless security model)

## Degree Alignment

| Course | Competency Covered |
|---|---|
| NET 412 | DAC, ACLs, user management, sudo, CIS baselines |
| NET 210 | Privilege escalation prevention, security controls |
| NET 377 | Identifying privilege escalation paths (attacker lens) |

---

← [[../README|Back to MoC]] | Next: [[01-Permissions-ACLs]] →
