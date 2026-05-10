---
title: A1 - Centralized Logging Pipeline
tags: [advanced, logging, detection]
---

# A1 — Centralized Logging Pipeline

## Mission
Create a reliable multi-node log pipeline for investigations and troubleshooting.

## Plan
1. Choose collector architecture.
2. Configure forwarding from Arch + selected VMs.
3. Validate message integrity/timestamps.
4. Build 5 investigation queries.

## Validation
- Simulated failed SSH login appears at collector within expected delay.
- Collector retains logs across reboot window.

## Sources
- [SRC-SYSTEMD-JOURNALD-01]
- [SRC-CISA-LOG-01]
- [SRC-NIST-80061-01]
