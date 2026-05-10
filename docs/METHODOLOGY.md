---
title: Evidence-First Methodology
aliases:
  - lab-method
  - runbook-methodology
tags:
  - methodology
  - evidence
  - forensics
  - operations
date: 2026-05-10
---

# Evidence-First Methodology

> [!info]
> This is the standard workflow for every lab and project in this repository.

## 1) Pre-Change Safety

- Declare execution context: Host | VM | Container
- Record baseline system state
- Create rollback point:
  - Host: Snapper pre snapshot
  - VM: Proxmox snapshot
  - Container: immutable tag + rebuild command

## 2) Change Plan

Document before execution:
- objective
- exact commands
- expected outcomes
- blast radius
- abort conditions

## 3) Controlled Execution

- Run steps in smallest safe increments
- Capture outputs directly to timestamped evidence folders
- Avoid undocumented ad-hoc changes

## 4) Validation

Use objective checks:
- service health/status
- expected file permissions/ownership
- expected network state
- expected log events

## 5) Evidence Packaging

```bash
TS="$(date -u +%Y%m%d_%H%M%SZ)"
OUT="evidence/run-$TS"
mkdir -p "$OUT"
# store command outputs into $OUT
find "$OUT" -type f -print0 | xargs -0 sha256sum > "$OUT/SHA256SUMS.txt"
```

## 6) Reporting

Each report includes:
- scope and timeline (UTC)
- executed commands
- validation results
- failures + fixes
- source citations (IDs from [[docs/references|references]])

## 7) Rollback Drill

- Confirm rollback steps are known and tested
- Execute rollback if change objective fails or risk increases
- Capture rollback evidence and lessons learned

## 8) Quality Gate Checklist

A lab is complete only if all are true:
- [ ] Execution context declared
- [ ] Rollback point recorded
- [ ] Validation evidence captured
- [ ] SHA256 manifest generated
- [ ] Report written with citations

## 9) Anti-Patterns to Avoid

- Running risky commands without snapshots
- Mixing host and lab contexts without documentation
- No hash manifest for artifacts
- Relying on memory instead of timestamped evidence
- Citing non-authoritative sources when primary docs exist
