---
aliases:
  - Phase 3 Overview
  - Engine Room
  - Process and Service Management
tags:
  - linux
  - systemd
  - processes
  - net412
  - net179
  - phase3
date: 2026-05-10
---

# Phase 3 — The Engine Room

> [!info] Phase Objective
> systemd is the init system, the service supervisor, the socket activator, the log aggregator, and the timer daemon. If you don't understand systemd, you're operating blind. This phase gives you full situational awareness of everything running on your machine.

## Modules

1. [[01-Process-Management|01 — Process Management]] — ps, top, htop, nice, kill, /proc, signals
2. [[02-Systemd-Deep-Dive|02 — systemd Deep Dive]] — Units, targets, timers, writing service files for Apex Predator
3. [[03-Journalctl-Troubleshooting|03 — journalctl & Log Forensics]] — Parsing logs, boot analysis, forensic timeline reconstruction

## Phase Competencies

By the end of Phase 3 you will be able to:

- Identify any process by name, PID, parent, user, or resource consumption
- Send the right signal to the right process without bringing down the system
- Write a production-quality systemd unit file with security hardening
- Use journalctl to reconstruct a forensic timeline of system events
- Diagnose broken services from logs alone — no trial and error

## Cross-Phase Links

- Prerequisites: [[../Phase-1-Core-Alphabet/01-CLI-Survival|Phase 1 — CLI]] (pipes, find, grep), [[../Phase-2-Blueprint/02-User-Group-Management|Phase 2 — Users]] (service accounts)
- Feeds into: [[../Phase-4-Communicator/03-SSH-Hardening|Phase 4 — SSH]] (sshd as a systemd service)
- Feeds into: [[../Phase-5-Architect/03-Container-Isolation|Phase 5 — Containers]] (container runtime management)

## Degree Alignment

| Course | Competency Covered |
|---|---|
| NET 412 | Process management, systemd, service configuration |
| NET 179 | Log-based forensic timeline reconstruction |
| NET 377 | Process injection detection, privilege via process manipulation |

---

← [[../README|Back to MoC]] | Next: [[01-Process-Management]] →
