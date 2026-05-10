---
title: I1 - Systemd Service Hardening
tags: [intermediate, systemd, hardening]
---

# I1 — Systemd Service Hardening

## Mission
Deploy a custom service with measurable hardening controls and reliability.

## Walkthrough (example checks)
```bash
systemctl cat your-service.service | tee evidence/i1/unit_contents.txt
systemd-analyze security your-service.service | tee evidence/i1/security_score.txt
journalctl -u your-service.service -n 200 --no-pager > evidence/i1/service_logs.txt
```

## Validation
- Service starts successfully after reboot.
- `systemd-analyze security` score improves after hardening edits.

## Sources
- [SRC-SYSTEMD-SERVICE-01]
- [SRC-SYSTEMD-SECURITY-01]
- [SRC-ARCH-SYSTEMD-01]
