---
aliases:
  - Phase 5 Overview
  - The Architect
  - Arch Proxmox Containers
tags:
  - linux
  - arch
  - btrfs
  - docker
  - podman
  - net412
  - phase5
date: 2026-05-10
---

# Phase 5 — The Architect

> [!info] Phase Objective
> This is your infrastructure layer. You're running Arch Linux on Btrfs with Snapper rollbacks, managing a Proxmox hypervisor, and isolating CTF tools in Distrobox containers. This phase gives you mastery over all of it — package management, atomic snapshots, and container isolation.

## Modules

1. [[01-Pacman-Database|01 — Pacman & AUR Management]] — pacman, paru, package signing, database repair
2. [[02-Btrfs-Snapshots|02 — Btrfs & Snapper Rollbacks]] — Subvolumes, snapshots, grub-btrfs rollback
3. [[03-Container-Isolation|03 — Docker/Podman for Isolated Workloads]] — Rootless Podman, Distrobox CTF containers, security namespaces

## Phase Competencies

By the end of Phase 5 you will be able to:

- Manage Arch packages and the pacman database including repair from corruption
- Take, list, compare, and roll back Btrfs snapshots from GRUB without booting the system
- Run fully isolated CTF toolsets in Distrobox containers without root
- Build rootless Podman containers with proper security namespaces for your Apex Predator services

## Cross-Phase Links

- Prerequisites: All previous phases (this is infrastructure — it touches everything)
- Containers use [[../Phase-2-Blueprint/01-Permissions-ACLs|Phase 2 — Permissions]] (namespaces are permission isolation)
- Containers use [[../Phase-3-Engine-Room/02-Systemd-Deep-Dive|Phase 3 — systemd]] (podman generates systemd units)
- Containers use [[../Phase-4-Communicator/01-IP-Routing|Phase 4 — Networking]] (container network namespaces)

## Degree Alignment

| Course | Competency Covered |
|---|---|
| NET 412 | Package management, containerization, storage management |
| NET 210 | Container isolation, security namespaces, AppArmor with containers |

---

← [[../README|Back to MoC]] | Next: [[01-Pacman-Database]] →
