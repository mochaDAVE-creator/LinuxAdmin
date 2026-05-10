---
aliases:
  - Phase 1 Overview
  - Core Alphabet
tags:
  - linux
  - net412
  - net179
  - phase1
  - cli
date: 2026-05-10
---

# Phase 1 — The Core Alphabet

> [!info] Phase Objective
> You cannot secure, forensicate, or administer what you cannot navigate. This phase gives you fluent, muscle-memory command of the Linux terminal. Every subsequent phase assumes you can do this in your sleep.

## Modules

1. [[01-CLI-Survival|01 — CLI Survival]] — Filesystem navigation, file operations, I/O redirection, pipes
2. [[02-Text-Manipulation|02 — Text Manipulation]] — grep, awk, sed, cut for log forensics and data extraction
3. [[03-FHS-Forensics|03 — FHS & Forensics Artifact Hunting]] — Where Linux hides its secrets (and attackers hide too)

## Phase Competencies

By the end of Phase 1 you will be able to:

- Navigate any Linux filesystem blind, using only the terminal
- Extract specific fields from multi-gigabyte log files using pipelines
- Identify the canonical location of forensic artifacts (auth logs, cron jobs, shell history, SUID binaries)
- Answer: *"Was this system compromised?"* using only CLI tools

## Cross-Phase Links

This phase feeds directly into:
- [[../Phase-2-Blueprint/01-Permissions-ACLs|Phase 2 — Permissions]] (you'll `ls -la` everything)
- [[../Phase-3-Engine-Room/03-Journalctl-Troubleshooting|Phase 3 — Log Forensics]] (text manipulation at scale)
- [[../Phase-4-Communicator/02-Network-Tools|Phase 4 — Network Tools]] (parsing tcpdump/nmap output)

## Degree Alignment

| Course | Competency Covered |
|---|---|
| NET 412 | FHS directory structure, CLI proficiency |
| NET 179 | Artifact hunting, sterile evidence identification |
| NET 377 | Footprinting via filesystem enumeration |

---

← [[../README|Back to MoC]] | Next: [[01-CLI-Survival]] →
