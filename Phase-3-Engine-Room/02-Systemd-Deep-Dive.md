---
aliases:
  - systemd
  - systemctl
  - Unit Files
  - Systemd Timers
  - Service Files
tags:
  - linux
  - systemd
  - net412
  - phase3
date: 2026-05-10
---

# 02 — systemd Deep Dive

> [!info] Why This Matters
> systemd is PID 1 — it owns everything. It starts your services, manages dependencies, handles logging, and controls the boot sequence. If you fight systemd instead of understanding it, you're wasting time. If you can write a unit file, you can make any program behave like a first-class system service.

## systemd Architecture

```
systemd (PID 1)
├── Units (.service, .socket, .timer, .mount, .target, .path, .device)
├── Journal (unified logging — journald)
├── Login management (logind)
├── Network configuration (networkd)
└── Hostname/locale/time management
```

**Unit file search path (order matters):**
```
/etc/systemd/system/        # Administrator-defined (highest priority)
/run/systemd/system/        # Runtime (not persistent)
/usr/lib/systemd/system/    # Package-installed (lowest priority)
```

---

## systemctl — The Control Interface

```bash
# Service lifecycle
systemctl start   myservice       # start now
systemctl stop    myservice       # stop now
systemctl restart myservice       # stop + start
systemctl reload  myservice       # reload config (if supported — no restart)
systemctl status  myservice       # detailed status with recent logs

# Enable/disable at boot
systemctl enable  myservice       # create symlink → start at boot
systemctl enable --now myservice  # enable + start immediately
systemctl disable myservice       # remove symlink → don't start at boot
systemctl mask    myservice       # prevent starting entirely (symlink to /dev/null)
systemctl unmask  myservice       # undo mask

# Query state
systemctl is-active  myservice    # active/inactive
systemctl is-enabled myservice    # enabled/disabled
systemctl is-failed  myservice    # true/false

# List units
systemctl list-units                             # all active units
systemctl list-units --type=service             # only services
systemctl list-units --state=failed             # failed services (your priority)
systemctl list-units --state=running --no-pager # running services
systemctl list-unit-files                       # all installed units and their state

# System control
systemctl reboot
systemctl poweroff
systemctl suspend
systemctl hibernate

# Reload systemd after editing unit files
systemctl daemon-reload
```

---

## Anatomy of a Unit File

### [Unit] Section — Metadata & Dependencies

```ini
[Unit]
Description=Apex Predator PMO Execution Engine
Documentation=https://github.com/mochaDAVE-creator/apex-predator
After=network.target    # start after networking is up
Requires=database.service   # hard dependency (fail if this fails)
Wants=optional.service      # soft dependency (continue even if this fails)
```

### [Service] Section — Behavior

```ini
[Service]
Type=simple          # process starts and stays running (most common)
# Type=forking       # process forks and parent exits (traditional daemons)
# Type=oneshot       # runs once and exits
# Type=notify        # process sends readiness notification (sd_notify)
# Type=idle          # like simple but waits until job queue is empty

ExecStart=/usr/bin/apex-predator --config /etc/apex/config.toml
ExecReload=/bin/kill -SIGHUP $MAINPID    # what 'systemctl reload' does
ExecStop=/bin/kill -SIGTERM $MAINPID     # what 'systemctl stop' does

# Environment
Environment="RUST_LOG=info"
EnvironmentFile=/etc/apex/environment    # key=value file (one per line)

# Restart policy
Restart=on-failure           # restart if exits with non-zero code
RestartSec=5                 # wait 5 seconds before restarting
StartLimitIntervalSec=60     # within 60 seconds...
StartLimitBurst=3            # ...allow max 3 restarts before giving up

# Process identity
User=apex_engine
Group=apex_group
WorkingDirectory=/opt/apex-predator
```

### [Install] Section — Boot Integration

```ini
[Install]
WantedBy=multi-user.target   # standard for most services (runlevel 3/5 equivalent)
# WantedBy=graphical.target  # for services that need a display
# WantedBy=default.target    # for user services
```

---

## Writing a Unit File for Apex Predator PMO

```bash
# Create the unit file
sudo nvim /etc/systemd/system/apex-predator.service
```

```ini
[Unit]
Description=Apex Predator PMO — Rust/Solidity Blockchain Execution Engine
Documentation=https://github.com/mochaDAVE-creator/apex-predator
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=apex_engine
Group=apex_group
WorkingDirectory=/opt/apex-predator
ExecStart=/opt/apex-predator/bin/apex-predator --config /etc/apex/config.toml
ExecReload=/bin/kill -SIGHUP $MAINPID
Restart=on-failure
RestartSec=10
StartLimitIntervalSec=120
StartLimitBurst=3

# Environment
Environment="RUST_LOG=info"
Environment="APEX_ENV=production"
EnvironmentFile=-/etc/apex/environment    # '-' prefix: don't fail if file missing

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/apex-predator /var/log/apex-predator
CapabilityBoundingSet=

# Logging to specific syslog identifier
StandardOutput=journal
StandardError=journal
SyslogIdentifier=apex-predator

[Install]
WantedBy=multi-user.target
```

```bash
# Deploy the service
sudo systemctl daemon-reload
sudo systemctl enable --now apex-predator
sudo systemctl status apex-predator

# Check security score
systemd-analyze security apex-predator.service
```

---

## systemd Targets — The Runlevel Replacement

Targets are collections of units that group system states.

```bash
# View current target
systemctl get-default

# List all targets
systemctl list-units --type=target

# Common targets
# graphical.target   = multi-user + graphical (like runlevel 5)
# multi-user.target  = multi-user, no graphical (like runlevel 3)
# rescue.target      = single-user mode
# emergency.target   = minimal shell, read-only filesystem

# Switch target (persistent)
systemctl set-default multi-user.target

# Switch target immediately (temporary)
systemctl isolate rescue.target

# Boot into a specific target (add to GRUB kernel line)
# systemd.unit=rescue.target
```

---

## systemd Timers — Cron Replacement

systemd timers are more powerful than cron: they have logging, dependency management, and accurate timing.

### One-Shot Timer (runs at a specific time)

```bash
sudo nvim /etc/systemd/system/db-backup.service
```

```ini
[Unit]
Description=Crypto Toolkit Database Backup

[Service]
Type=oneshot
User=dave
ExecStart=/usr/local/bin/backup_crypto_db.sh
```

```bash
sudo nvim /etc/systemd/system/db-backup.timer
```

```ini
[Unit]
Description=Daily Crypto DB Backup Timer
Requires=db-backup.service

[Timer]
OnCalendar=*-*-* 02:00:00    # daily at 2 AM
RandomizedDelaySec=300       # randomize by up to 5 min (avoid thundering herd)
Persistent=true              # run missed timers on next boot

[Install]
WantedBy=timers.target
```

```bash
# Enable the timer (not the service — timer triggers the service)
sudo systemctl enable --now db-backup.timer
sudo systemctl list-timers    # show all timers with next trigger time
```

### Monotonic Timer (relative to system events)

```ini
[Timer]
OnBootSec=5min          # 5 minutes after boot
OnUnitActiveSec=1h      # 1 hour after last activation
```

---

## Analyzing Boot Performance

```bash
# Overall boot time breakdown
systemd-analyze

# Per-unit timing (find slow services)
systemd-analyze blame | head -20

# Critical path visualization
systemd-analyze critical-chain

# Generate SVG boot graph
systemd-analyze plot > /tmp/boot.svg

# Check why a unit is enabled
systemctl cat sshd.service          # show the unit file
systemctl show sshd.service         # show all unit properties
systemd-analyze verify sshd.service # check for errors without starting
```

---

## systemd Overrides — Modifying Package Units

Never edit `/usr/lib/systemd/system/` directly — package updates overwrite it. Use drop-ins instead.

```bash
# Method 1: Create an override file
systemctl edit sshd.service
# Creates /etc/systemd/system/sshd.service.d/override.conf

# Method 2: Full unit replacement (copies original to /etc/systemd/system/)
systemctl edit --full sshd.service

# Example: Limit sshd's memory usage
# /etc/systemd/system/sshd.service.d/limits.conf
[Service]
MemoryMax=256M
CPUQuota=50%
```

---

## Practical Lab: Arch Live System

```bash
# 1. Audit currently running services
systemctl list-units --type=service --state=running --no-pager | tee /tmp/running_services.txt
wc -l /tmp/running_services.txt

# 2. Find failed services
systemctl list-units --state=failed --no-pager

# 3. Boot time analysis
systemd-analyze
systemd-analyze blame | head -15

# 4. List all timers (your automated tasks)
systemctl list-timers --all --no-pager

# 5. Inspect a service unit
systemctl cat sshd.service

# 6. Check security exposure of sshd
systemd-analyze security sshd.service | head -20

# 7. Check service startup dependencies
systemctl list-dependencies sshd.service
```

---

## Troubleshooting Scenario: Service Starts Then Immediately Dies

**Symptom:** `systemctl start apex-predator` shows Active: activating, then transitions to `failed`.

**Step 1: Read the status**
```bash
systemctl status apex-predator.service --no-pager -l
# Look for:
# Main PID: 12345 (code=exited, status=1/FAILURE)
# And the last few log lines
```

**Step 2: Check the journal**
```bash
journalctl -u apex-predator.service -n 50 --no-pager
# OR since last boot:
journalctl -u apex-predator.service -b --no-pager
```

**Step 3: Common failure causes and fixes**

| Error in Journal | Likely Cause | Fix |
|---|---|---|
| `exec format error` | Wrong architecture binary | Recompile with `cargo build --release` |
| `No such file or directory` | Wrong ExecStart path | Use `which apex-predator` to find real path |
| `Permission denied` | User can't access binary/config | `chmod +x` or fix ownership |
| `Failed to bind` | Port already in use | `ss -tlnp \| grep <port>` — kill the occupier |
| `core dumped` | Segfault (Rust panic) | Check `journalctl` for panic backtrace |

```bash
# Manual execution test (run as service user to replicate environment)
sudo -u apex_engine /opt/apex-predator/bin/apex-predator --config /etc/apex/config.toml
# This immediately shows the error without systemd noise
```

---

## 🏁 Proof of Work — Phase 3.2 Mini-CTF

> [!example] Challenge: Write and Deploy a systemd Timer
>
> Create a systemd timer that runs a security check every hour and logs the result:
>
> ```bash
> # Create the checker script
> sudo tee /usr/local/bin/security_check.sh << 'EOF'
> #!/bin/bash
> echo "=== Security Check: $(date -u) ==="
> echo "SUID count: $(find / -perm -4000 -type f 2>/dev/null | wc -l)"
> echo "Failed services: $(systemctl list-units --state=failed --no-pager | grep -c failed)"
> echo "Listening ports: $(ss -tlnp | grep -c LISTEN)"
> EOF
> sudo chmod +x /usr/local/bin/security_check.sh
>
> # Create service unit
> sudo tee /etc/systemd/system/security-check.service << 'EOF'
> [Unit]
> Description=Hourly Security Check
>
> [Service]
> Type=oneshot
> ExecStart=/usr/local/bin/security_check.sh
> StandardOutput=journal
> SyslogIdentifier=security-check
> EOF
>
> # Create timer unit
> sudo tee /etc/systemd/system/security-check.timer << 'EOF'
> [Unit]
> Description=Hourly Security Check Timer
>
> [Timer]
> OnBootSec=1min
> OnUnitActiveSec=1h
>
> [Install]
> WantedBy=timers.target
> EOF
>
> # Deploy
> sudo systemctl daemon-reload
> sudo systemctl enable --now security-check.timer
>
> # Trigger manually to test
> sudo systemctl start security-check.service
>
> # Verify output
> journalctl -u security-check.service -n 20 --no-pager
> systemctl list-timers security-check.timer
> ```
>
> **Proof:** Run `systemctl list-timers security-check.timer` and capture output. Next trigger time should be within 1 hour.

---

← [[01-Process-Management]] | Next: [[03-Journalctl-Troubleshooting]] →
