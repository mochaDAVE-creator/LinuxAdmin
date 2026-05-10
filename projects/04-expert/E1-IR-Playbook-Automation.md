---
title: E1 - Incident Response Playbook Automation
tags: [expert, incident-response, automation, forensics]
---

# E1 — Incident Response Playbook Automation

## Mission
Automate first-response triage to produce a court-defensible-style evidence bundle for lab incidents.

## Scope
- Process list
- Active sockets
- Recent auth events
- Persistence checks
- Hash manifest generation

## Walkthrough Skeleton
```bash
mkdir -p evidence/e1/$(date -u +%Y%m%d_%H%M%SZ)
OUT="evidence/e1/$(date -u +%Y%m%d_%H%M%SZ)"
ps aux > "$OUT/ps_aux.txt"
ss -tulpen > "$OUT/sockets.txt"
journalctl -p err -b --no-pager > "$OUT/journal_errors.txt"
find /etc/systemd/system /etc/cron.d -type f > "$OUT/persistence_paths.txt"
find "$OUT" -type f -print0 | xargs -0 sha256sum > "$OUT/SHA256SUMS.txt"
```

## Validation
- One-command execution completes without errors.
- Evidence package includes integrity manifest.
- Report maps findings to ATT&CK techniques.

## Sources
- [SRC-NIST-80061-01]
- [SRC-MITRE-ATTACK-01]
- [SRC-CISA-LOG-01]
