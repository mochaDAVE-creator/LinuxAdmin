---
title: Beginner Projects
aliases:
  - beginner-landing
tags:
  - beginner
  - projects
  - archlinux
  - proxmox
date: 2026-05-10
---

# Beginner Projects

> [!info] Who this is for
> These projects are designed for operators who are new to Arch Linux and/or Proxmox lab environments.
> Each project builds safe-change habits, evidence discipline, and core Linux administration skills.
> Complete them in the recommended order below.

---

## Recommended Order

| Order | ID | Project | Effort | Risk |
|------:|----|-----------------------------------------|--------|------|
| 1 | B1 | [Snapshot & Rollback Drill](B1-Snapshot-Rollback-Drill.md) | 2–3 hrs | Low |
| 2 | B2 | [FHS Artifact Hunt](B2-FHS-Artifact-Hunt.md) | 3–5 hrs | Low |
| 3 | B4 | [Log Triage Starter](B4-Log-Triage-Starter.md) | 3–4 hrs | Low |
| 4 | B3 | [SSH Key-Only Admin Access](B3-SSH-Key-Only-Admin-Access.md) | 2–4 hrs | Medium |

> [!tip] Why this order?
> B1 establishes your safety net (snapshots/rollback) before you touch anything else.
> B2 and B4 are read-only exploration drills — safe to run while you build confidence.
> B3 changes authentication configuration and carries slightly more operational risk, so it comes last.

---

## Prerequisites

Before starting any beginner project, confirm the following:

### Skills
- Basic Linux command-line navigation (`cd`, `ls`, `cat`, `less`, `grep`)
- Understanding of file ownership and permissions (`ls -la`, `chmod`, `chown`)
- Ability to read `man` pages and use `--help` flags

### Lab Environment
- Arch Linux host with Btrfs filesystem and Snapper configured (for B1)
- At least one Proxmox VM you can safely snapshot and restore (for B1 VM path)
- SSH access to your lab host (for B3)
- `sudo` privileges on the target system

### Tools (install if missing)
```bash
# Verify snapper is installed (needed for B1)
snapper --version

# Verify ssh-keygen is available (needed for B3)
ssh-keygen --help 2>&1 | head -3

# journalctl is part of systemd — should already be present
journalctl --version
```

---

## Safety Reminders

> [!warning] Read before starting
> 1. **Always create a snapshot before making changes.** B1 teaches this — apply it to every subsequent project.
> 2. **Declare your execution context** (Host / VM / Container) at the top of every lab log. Never mix contexts without documenting it.
> 3. **Never run lab commands on production systems** or systems you do not own/control.
> 4. **No real credentials, IPs, or private keys** should be committed to this repository. Use placeholder values in evidence files before committing.
> 5. **Rollback first, diagnose second.** If something breaks and you are unsure, roll back to your snapshot before investigating.

---

## Expected Deliverables

Every beginner project must produce the following before it is considered complete:

- [ ] Evidence directory with timestamped outputs: `evidence/b<N>/<YYYYMMDD_HHMMSSZ>/`
- [ ] `SHA256SUMS.txt` hash manifest inside the evidence directory
- [ ] Completed report checklist (found at the end of each project file)
- [ ] Rollback point recorded (Snapper snapshot ID or Proxmox snapshot name)

Use the quick-start snippet below to initialise your evidence directory at the start of each project:

```bash
TS="$(date -u +%Y%m%d_%H%M%SZ)"
PROJECT="b1"   # change to b2, b3, or b4 as appropriate
OUT="evidence/${PROJECT}/${TS}"
mkdir -p "$OUT"
echo "Evidence directory: $OUT"
```

At the end of each project, generate the hash manifest:

```bash
find "$OUT" -type f ! -name 'SHA256SUMS.txt' -print0 \
  | xargs -0 sha256sum > "$OUT/SHA256SUMS.txt"
cat "$OUT/SHA256SUMS.txt"
```

---

## Project Summaries

### B1 — Snapshot & Rollback Drill
**File:** [B1-Snapshot-Rollback-Drill.md](B1-Snapshot-Rollback-Drill.md)

Prove you can perform system changes with a guaranteed rollback path. Covers Snapper (Btrfs host) and Proxmox VM snapshots. This is the safety foundation for all future projects.

**Key skills:** Snapper, Btrfs subvolumes, Proxmox snapshot API, evidence collection, hash manifest generation.

---

### B2 — FHS Artifact Hunt
**File:** [B2-FHS-Artifact-Hunt.md](B2-FHS-Artifact-Hunt.md)

Navigate the Linux Filesystem Hierarchy Standard (FHS) to locate core forensic artifacts: logs, configuration files, process information, and user home directories. Generate a hashed triage report of findings.

**Key skills:** FHS layout, `/proc`, `/var/log`, `/etc`, `find`, `stat`, `sha256sum`.

---

### B3 — SSH Key-Only Admin Access
**File:** [B3-SSH-Key-Only-Admin-Access.md](B3-SSH-Key-Only-Admin-Access.md)

Harden your lab SSH workflow by generating a key pair, configuring `sshd` to disable password authentication, and validating the change with explicit verification tests. Includes a rollback procedure.

**Key skills:** `ssh-keygen`, `ssh-copy-id`, `sshd_config`, `systemctl`, rollback planning, auth log review.

---

### B4 — Log Triage Starter
**File:** [B4-Log-Triage-Starter.md](B4-Log-Triage-Starter.md)

Use `journalctl` and standard text tools (`grep`, `awk`, `sed`) to detect failed logins, service failures, and suspicious sudo usage. Build a repeatable triage pattern you can apply to any incident.

**Key skills:** `journalctl`, failed login detection, log filtering, output redirection, evidence packaging.

---

## Quality Gate Checklist

A beginner project is **complete** only when all of the following are true:

- [ ] Execution context declared (Host / VM / Container)
- [ ] Pre-change snapshot or rollback point recorded
- [ ] All validation commands returned expected output
- [ ] Evidence files captured and saved to timestamped directory
- [ ] `SHA256SUMS.txt` hash manifest generated and verified
- [ ] Report checklist at the end of the project file is filled in

---

## Next Steps After Beginner

Once you have completed B1–B4, proceed to:

- `projects/02-intermediate/README.md` — service hardening, persistence detection, ACL management
- Review `projects/PROJECT_INDEX.md` for the full 90-day learning sequence
- Consult `docs/METHODOLOGY.md` any time you are unsure about evidence or rollback procedures
