---
title: Lab Topology & Trust Boundaries
aliases:
  - homelab-topology
  - execution-context-map
tags:
  - lab
  - topology
  - proxmox
  - archlinux
  - security
  - net179
date: 2026-05-10
---

# Lab Topology & Trust Boundaries

> [!info] Purpose
> This document defines the authoritative topology for your Arch host + Proxmox environment.
> Every module and project must reference this file for execution context, network scope, and rollback safety.

## 1) Asset Inventory

## 1.1 Arch Host (Bare Metal)
- Hostname:
- Management IP:
- Kernel:
- Filesystem: Btrfs
- Critical subvolumes:
  - `@`
  - `@home`
  - `@var_log`
  - `@snapshots`
- Snapper configs:
  - `root`
  - `home` (optional)

## 1.2 Proxmox Node(s)
- Node name(s):
- Node management IP(s):
- Cluster: Yes/No
- Storage backends:
  - local-lvm:
  - ZFS/Btrfs/NFS/Ceph:
- Backup target(s):
  - PBS/NFS path:

## 1.3 Security Tooling Hosts
- SIEM/log collector VM:
- Forensics workstation VM:
- Scanner VM (OpenSCAP/Lynis/Nmap):
- Jump box:

---

## 2) Network Segmentation

## 2.1 VLANs / Zones
- VLAN 10 — Management
  - Purpose: Admin access (SSH/WebUI/API)
  - Allowed sources: trusted admin endpoints only
- VLAN 20 — Server/Lab Services
  - Purpose: App/service hosting
- VLAN 30 — Security/Monitoring
  - Purpose: Log aggregation, IDS, telemetry
- VLAN 40 — Adversary/CTF Range
  - Purpose: Isolated offensive testing
- VLAN 50 — Backup/Replication
  - Purpose: backup traffic only

## 2.2 Trust Boundaries
- Boundary A: Internet ↔ Edge Router/Firewall
- Boundary B: Management VLAN ↔ All other VLANs
- Boundary C: CTF VLAN ↔ Production-like lab services
- Boundary D: Host OS ↔ Containers/VMs

> [!warning]
> No direct SSH from CTF VLAN to management plane. Force jump-host pattern.

---

## 3) Execution Context Rules

Every lab/project must declare one of:

- **Host**: Bare metal Arch only
- **VM**: Proxmox VM only
- **Container**: Distrobox/Podman/LXC only

And must include:

- Rollback strategy:
  - Host: Snapper pre/post IDs
  - VM: Proxmox snapshot name
  - Container: immutable image tag + rebuild command
- Evidence path:
  - `evidence/<phase-or-project>/<timestamp>/`

---

## 4) Snapshot / Rollback SOP

## 4.1 Host (Snapper)
```bash
sudo snapper -c root create --description "pre-change-<ticket-or-project>"
snapper -c root list | tail -n 5
```

## 4.2 Proxmox VM Snapshot
- Snapshot naming convention:
  - `pre-<project>-YYYYMMDD-HHMMZ`
- Required before:
  - package upgrades
  - service reconfiguration
  - kernel/network changes

## 4.3 Recovery Drill Cadence
- Weekly: one VM snapshot restore test
- Monthly: one host rollback simulation (non-destructive if possible)

---

## 5) Logging & Evidence Architecture

## 5.1 Log Sources
- Arch host: `journalctl`, `/var/log/*`
- Proxmox: task logs, node logs
- Services: nginx/apache/app logs
- Security tools: scanner outputs, audit reports

## 5.2 Evidence Standards
- All evidence files hashed with SHA-256
- Manifest at:
  - `evidence/_hashes/manifest.sha256`
- Verification output:
  - `evidence/_hashes/verification.txt`

## 5.3 Chain-of-Custody Lite (for student labs)
Track:
- Operator
- UTC timestamp
- Command executed
- Output file path
- Hash before/after transfer

---

## 6) Access Control Baseline

- Admin access via SSH keys only (no passwords where feasible)
- Separate admin identities from daily user identities
- `sudo` restricted to admin group
- Root login policy documented and justified
- MFA enabled where supported

---

## 7) Naming Conventions

- VM names: `role-env-##` (e.g., `siem-lab-01`)
- Snapshots: `pre-<change>-<UTC>`
- Evidence folders: `evidence/<project>/<YYYYMMDD_HHMMSSZ>/`
- Reports: `<project>_report.md`

---

## 8) Diagrams

- Add exported diagrams under `docs/diagrams/`
  - `network-logical.drawio`
  - `trust-boundaries.drawio`
  - `data-flow-evidence.drawio`

---

## 9) Sign-off

- Last topology review date:
- Reviewed by:
- Next review due:
