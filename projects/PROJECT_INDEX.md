---
title: Project Index Dashboard
aliases:
  - project-dashboard
  - project-index
tags:
  - projects
  - dashboard
  - roadmap
date: 2026-05-10
---

# Project Index Dashboard

> [!info]
> Single-pane view of all projects with skill prerequisites, estimated effort, risk, and source IDs.

## Legend
- Difficulty: Beginner / Intermediate / Advanced / Expert
- Risk: Low / Medium / High (operational impact)
- Effort: estimated focused hours

| ID | Project | Difficulty | Effort (hrs) | Risk | Core Skills | Primary Sources |
|---|---|---|---:|---|---|---|
| B1 | Snapshot & Rollback Drill | Beginner | 2–3 | Low | Btrfs, Snapper, safe change mgmt | [SRC-ARCH-SNAPPER-01], [SRC-ARCH-BTRFS-01] |
| B2 | FHS Artifact Hunt | Beginner | 3–5 | Low | FHS, log paths, triage | [SRC-FHS-3.0-01], [SRC-MAN7-HIER-01], [SRC-PROCFS-01] |
| B3 | SSH Key-Only Admin Access | Beginner | 2–4 | Medium | SSH keys, sshd config, auth testing | [SRC-OPENSSH-01], [SRC-ARCH-SSH-01], [SRC-MOZ-SSH-01] |
| B4 | Log Triage Starter | Beginner | 3–4 | Low | journalctl, grep/awk/sed | [SRC-SYSTEMD-JOURNALCTL-01], [SRC-GNU-GREP-01], [SRC-GNU-AWK-01] |
| I1 | Systemd Service Hardening | Intermediate | 4–6 | Medium | unit files, sandboxing, reliability | [SRC-SYSTEMD-SERVICE-01], [SRC-SYSTEMD-SECURITY-01], [SRC-ARCH-SYSTEMD-01] |
| I2 | Persistence Detection Sweep | Intermediate | 5–8 | Medium | cron/systemd/profile/SSH persistence checks | [SRC-MITRE-T1547-01], [SRC-SYSTEMD-UNIT-01], [SRC-PROCFS-01] |
| I3 | ACL-Backed Secret Store | Intermediate | 3–5 | Medium | DAC/ACL, service identity isolation | [SRC-MAN-SETFACL-01], [SRC-MAN-GETFACL-01], [SRC-CIS-LINUX-01] |
| I4 | Proxmox Backup Validation | Intermediate | 4–7 | Medium | backup/restore testing, evidence handling | [SRC-PVE-ADMIN-01], [SRC-PVE-DOCS-01], [SRC-NIST-80061-01] |
| A1 | Centralized Logging Pipeline | Advanced | 8–12 | High | log forwarding, parsing, retention | [SRC-SYSTEMD-JOURNALD-01], [SRC-CISA-LOG-01], [SRC-NIST-80061-01] |
| A2 | Network Segmentation Enforcement | Advanced | 8–14 | High | VLANs, routing policy, verification | [SRC-IPROUTE2-01], [SRC-ARCH-NETCONF-01], [SRC-PVE-ADMIN-01] |
| A3 | Threat-Informed Detection Pack | Advanced | 10–16 | High | ATT&CK mapping, rule design, simulation | [SRC-MITRE-ATTACK-01], [SRC-MITRE-T1059-01], [SRC-CISA-LOG-01] |
| A4 | Container Isolation Benchmark | Advanced | 8–12 | Medium | rootless containers, hardening baseline | [SRC-PODMAN-01], [SRC-DISTROBOX-01], [SRC-CIS-CONTAINERS-01] |
| E1 | IR Playbook Automation | Expert | 12–20 | High | automation, evidence packaging, IR workflow | [SRC-NIST-80061-01], [SRC-MITRE-ATTACK-01], [SRC-CISA-LOG-01] |
| E2 | Zero-Trust Homelab Blueprint | Expert | 16–30 | High | policy architecture, segmentation, identity | [SRC-CIS-LINUX-01], [SRC-NIST-80061-01], [SRC-PVE-DOCS-01] |
| E3 | Secure Platform Baseline | Expert | 20–40 | High | hardening profile + exception process | [SRC-CIS-LINUX-01], [SRC-DISA-STIG-01], [SRC-OPENSCAP-01] |
| E4 | Purple-Team Validation Cycle | Expert | 20–40 | High | adversary emulation + detection tuning | [SRC-MITRE-ATTACK-01], [SRC-MITRE-T1059-01], [SRC-CISA-LOG-01] |

---

## Suggested Learning Sequence (90-Day)

### Days 1–30
- Complete: B1, B2, B4
- Milestone: produce first reproducible evidence bundle with hash manifest

### Days 31–60
- Complete: B3, I1, I3
- Milestone: hardened service + ACL-protected data + rollback proof

### Days 61–90
- Complete: I2, I4, A1
- Milestone: persistence hunting and centralized logging with validated detections

Then progress into A2/A3/A4 and expert projects.

---

## Per-Project Quality Gate (must pass)

1. Execution Context declared
2. Rollback plan recorded
3. Validation commands successful
4. Evidence captured and hashed
5. Report includes source citations + control mapping

---

## Quick Start Commands

```bash
# create project evidence directory
TS="$(date -u +%Y%m%d_%H%M%SZ)"
mkdir -p "evidence/project-$TS"

# create hash manifest
find "evidence/project-$TS" -type f -print0 | xargs -0 sha256sum > "evidence/project-$TS/SHA256SUMS.txt"
```
