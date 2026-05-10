---
aliases:
  - Pacman
  - AUR
  - Arch Package Management
  - paru yay
tags:
  - linux
  - arch
  - pacman
  - aur
  - net412
  - phase5
date: 2026-05-10
---

# 01 — Pacman & AUR Management

> [!info] Why This Matters
> Pacman is your system's lifeline. Corrupt the database, break a GPG key, or install conflicting packages and your system goes from "runs" to "doesn't boot". Know pacman deeply enough to fix it blind, from recovery mode, without internet if needed.

## pacman Fundamentals

### The Four Operations

```bash
# -S  = Sync (install from repos)
# -R  = Remove
# -U  = Upgrade (install local package file)
# -Q  = Query (interrogate local database)
# -D  = Database (manipulate package metadata)
# -F  = Files (search file database)
```

### Installing and Removing

```bash
# Install package(s)
sudo pacman -S git vim htop

# Install from multiple repos
sudo pacman -S extra/htop core/filesystem

# Remove package only
sudo pacman -R packagename

# Remove package + unused dependencies
sudo pacman -Rs packagename

# Remove package + deps + config files
sudo pacman -Rns packagename

# Remove package but keep explicitly installed deps
sudo pacman -Rdd packagename   # dangerous — removes without dep check

# Remove orphans (packages installed as deps, no longer needed)
sudo pacman -Qtdq | sudo pacman -Rns -   # pipe orphan list to remove
```

### Searching and Querying

```bash
# Search repos (by name or description)
pacman -Ss "keyword"
pacman -Ss "^vim$"     # exact name match

# Search local installed packages
pacman -Qs "keyword"

# Get package info (remote)
pacman -Si packagename

# Get package info (local/installed)
pacman -Qi packagename

# List all installed packages
pacman -Q
pacman -Qe             # explicitly installed (not pulled in as dep)
pacman -Qd             # installed as dependency
pacman -Qn             # installed from main repos (not AUR)
pacman -Qm             # foreign packages (AUR/manual installs)
pacman -Qt             # unrequired packages (orphans)

# Which package owns a file?
pacman -Qo /usr/bin/vim
pacman -F /usr/bin/vim   # search file database (even for uninstalled packages)

# List files installed by a package
pacman -Ql vim | head -20

# Check for broken/missing files in installed packages
sudo pacman -Qkk 2>&1 | grep -v "0 missing"   # shows files that are missing
```

### Updating the System

```bash
# Full system update
sudo pacman -Syu
# -y = sync package database  -u = upgrade all packages

# Update database only (don't upgrade)
sudo pacman -Sy

# Force refresh database even if up-to-date
sudo pacman -Syy

# Upgrade without database refresh (not recommended)
sudo pacman -Su
```

> [!warning] Never Run pacman -Sy packagename
> Running `-Sy` (sync database) without `-u` (upgrade) and then installing a new package can break your system. The new package may depend on updated libraries that aren't installed because you skipped the upgrade. **Always use `-Syu`.**

---

## Package Signing and GPG Keys

```bash
# Initialize the keyring (first time or after corruption)
sudo pacman-key --init

# Populate with Arch Linux default keys
sudo pacman-key --populate archlinux

# Refresh all keys from keyserver
sudo pacman-key --refresh-keys

# Add a specific key (e.g., for an AUR package's maintainer)
sudo pacman-key --recv-keys KEY_FINGERPRINT
sudo pacman-key --lsign-key KEY_FINGERPRINT    # locally sign the key

# List all trusted keys
pacman-key --list-keys

# Check key status
pacman-key --list-sigs KEY_FINGERPRINT
```

> [!tip] Key Refresh Behind Firewall
> If `--refresh-keys` hangs (firewall blocking HKP port 11371):
> ```bash
> sudo pacman-key --keyserver hkps://keys.openpgp.org --refresh-keys
> # or
> sudo pacman-key --keyserver hkps://keyserver.ubuntu.com --refresh-keys
> ```

---

## AUR — Arch User Repository

The AUR contains community-maintained packages not in the official repos. **Always inspect PKGBUILDs before installing.**

> [!danger] AUR Security Model
> AUR packages are user-submitted. The PKGBUILD (build script) runs with your user's permissions during build, but `makepkg` itself runs as you. A malicious PKGBUILD could exfiltrate data, add SSH keys, or install backdoors. **Read every PKGBUILD before installing from the AUR.**

### Manual AUR Installation

```bash
# Clone the AUR package
git clone https://aur.archlinux.org/paru-bin.git
cd paru-bin

# READ THE PKGBUILD — this is not optional
cat PKGBUILD
# Verify: source URLs, checksums, install steps

# Build and install
makepkg -si
# -s = install missing deps  -i = install after build

# View generated package before installing
makepkg -s    # build only
pacman -U paru-bin-*.pkg.tar.zst   # install manually
```

### paru — AUR Helper

```bash
# Install paru (bootstrapped from AUR — see manual method above)
git clone https://aur.archlinux.org/paru-bin.git && cd paru-bin && makepkg -si

# Usage mirrors pacman
paru -Syu              # update system + AUR packages
paru -S package-name   # install from AUR (shows PKGBUILD diff by default)
paru -Ss "keyword"     # search AUR

# paru security feature: review PKGBUILDs before install
# /etc/paru.conf
# [options]
# BottomUp          # newest results at bottom
# SudoLoop          # keep sudo alive during long builds
# NewsOnUpgrade     # show Arch news before upgrading
# CombinedUpgrade   # handle AUR + official together
```

---

## pacman Database Management and Repair

### Backup and Restore

```bash
# Location of local package database
ls /var/lib/pacman/local/

# Location of sync databases (repo data)
ls /var/lib/pacman/sync/

# Backup package list (for reinstall after catastrophic failure)
pacman -Qqe > /backup/explicit_packages.txt       # explicitly installed
pacman -Qqm > /backup/aur_packages.txt            # AUR/foreign packages
pacman -Qqn > /backup/native_packages.txt         # from official repos

# Restore from backup
sudo pacman -S --needed $(cat /backup/explicit_packages.txt)
```

### Database Repair

```bash
# Remove stale/locked database lock
sudo rm /var/lib/pacman/db.lck
# Only do this if pacman crashed and you're sure no other pacman is running!
pgrep pacman || sudo rm /var/lib/pacman/db.lck

# Force database refresh
sudo pacman -Syy

# Check for database corruption
sudo pacman -Dk    # check database consistency

# Rebuild broken database from installed packages
# Nuclear option — only if database is severely corrupted
sudo mv /var/lib/pacman/local /var/lib/pacman/local.bak
sudo pacman-db-upgrade   # may not exist on all versions
# Better: restore from Btrfs snapshot (see Phase 5.2)

# Re-register a package that was installed outside pacman
sudo pacman -U /var/cache/pacman/pkg/packagename-version.pkg.tar.zst

# Reinstall all explicitly installed packages (rebuilds database)
sudo pacman -Qqe | sudo pacman -S --needed -
```

### Package Cache Management

```bash
# View cache size
du -sh /var/cache/pacman/pkg/

# Remove packages from cache that aren't installed (keep all versions)
sudo pacman -Sc

# Remove all cached packages except the 3 most recent versions
sudo paccache -r

# Remove all cached versions of uninstalled packages
sudo paccache -ruk0

# Keep only N versions in cache
sudo paccache -rk 2   # keep 2 versions

# Enable automatic cache cleanup
sudo systemctl enable --now paccache.timer
```

---

## Downgrading Packages

```bash
# Downgrade from cache (if previous version is cached)
sudo pacman -U /var/cache/pacman/pkg/packagename-oldversion.pkg.tar.zst

# Downgrade using downgrade tool (AUR)
sudo downgrade packagename

# Prevent a package from being upgraded (pin it)
# In /etc/pacman.conf:
IgnorePkg = badpackage anotherpackage
IgnoreGroup = gnome    # ignore an entire group
```

---

## Practical Lab: Arch Package Audit

```bash
# 1. System state snapshot
echo "=== Package Audit: $(date) ===" | tee /tmp/pkg_audit.txt
echo "Total packages: $(pacman -Qq | wc -l)" | tee -a /tmp/pkg_audit.txt
echo "Explicitly installed: $(pacman -Qqe | wc -l)" | tee -a /tmp/pkg_audit.txt
echo "Orphans: $(pacman -Qtdq | wc -l)" | tee -a /tmp/pkg_audit.txt
echo "AUR packages: $(pacman -Qqm | wc -l)" | tee -a /tmp/pkg_audit.txt

# 2. List orphans (consider removing)
echo ""
echo "=== Orphaned Packages (possible removal) ==="
pacman -Qtdq

# 3. Find recently installed packages (last 10)
echo ""
echo "=== Recently Installed ==="
grep "installed" /var/log/pacman.log | tail -10 | awk '{print $1, $2, $4}'

# 4. Check for failed installs
echo ""
echo "=== Recent Errors ==="
grep -i "error\|warning\|failed" /var/log/pacman.log | tail -10

# 5. Verify key packages are authentic
echo ""
echo "=== File Integrity Check (pacman) ==="
sudo pacman -Qkk 2>&1 | grep -v "0 missing\|0 altered" | head -10
```

---

## Troubleshooting Scenario: Invalid or Corrupted Package Signature

**Symptom:**
```
error: package: signature from "Developer Name" is invalid
error: failed to commit transaction (invalid or corrupted package (PGP signature))
```

**Fix:**
```bash
# Step 1: Refresh keys
sudo pacman-key --refresh-keys

# Step 2: Re-populate keyring
sudo pacman-key --populate archlinux

# Step 3: Clear stale package cache entry
sudo pacman -Sc    # clear uninstalled packages from cache

# Step 4: Retry the installation
sudo pacman -Syu

# If specific key is the issue:
# Find the key ID from the error message (8 hex chars after "from")
sudo pacman-key --recv-keys KEYID
sudo pacman-key --lsign-key KEYID
sudo pacman -Syu

# Nuclear: delete and rebuild entire keyring
sudo rm -rf /etc/pacman.d/gnupg
sudo pacman-key --init
sudo pacman-key --populate archlinux
sudo pacman -Syu
```

> [!warning] Disabling Signature Checking
> You may find suggestions to set `SigLevel = Never` in pacman.conf. **Do not do this.** Package signing is your protection against supply chain attacks. Fix the key issue properly.

---

## 🏁 Proof of Work — Phase 5.1 Mini-CTF

> [!example] Challenge: Package Provenance Audit
>
> ```bash
> # Identify all foreign (AUR/unverified) packages and audit their sources
> {
>   echo "=== PACKAGE PROVENANCE AUDIT ==="
>   echo "Date: $(date -u)"
>   echo "Host: $(hostname)"
>   echo ""
>
>   echo "--- Official Repo Packages ---"
>   pacman -Qqn | wc -l
>
>   echo ""
>   echo "--- Foreign/AUR Packages (requires manual review) ---"
>   pacman -Qqm | while read pkg; do
>     install_date=$(grep -m1 "installed $pkg" /var/log/pacman.log | awk '{print $1, $2}')
>     echo "$pkg — installed: $install_date"
>   done
>
>   echo ""
>   echo "--- Security-Critical Packages and Versions ---"
>   for pkg in openssl openssh linux sudo systemd; do
>     ver=$(pacman -Qi "$pkg" 2>/dev/null | grep "^Version" | awk '{print $3}')
>     echo "$pkg: $ver"
>   done
>
>   echo ""
>   echo "--- Orphaned Packages ---"
>   pacman -Qtdq || echo "None"
>
> } | tee /tmp/phase5_1_pow.txt
>
> sha256sum /tmp/phase5_1_pow.txt
> ```
>
> **Task:** If you find orphaned packages, determine if they're safe to remove. If you find AUR packages you don't recognize, check their AUR page. Document your findings.

---

← [[00-Phase5-Overview|Phase 5 Overview]] | Next: [[02-Btrfs-Snapshots]] →
