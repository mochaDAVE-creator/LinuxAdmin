---
title: A4 - Container Isolation Benchmark
aliases:
  - a4-container-isolation
  - podman-distrobox-benchmark
tags:
  - advanced
  - containers
  - podman
  - distrobox
  - isolation
  - hardening
  - evidence
  - net412
  - net210
date: 2026-05-10
---

# A4 — Container Isolation Benchmark

> [!warning] Operating Assumptions & Threat Model
> **Context:** Arch Linux bare-metal host with rootless Podman and Distrobox installed. This project benchmarks the isolation properties of containers versus the host, and applies hardening controls to close identified gaps.
> **Risk level:** Medium — container configurations are modified. A misconfigured container security policy could weaken isolation. Snapper pre-snapshot is mandatory.
> **Threat model:** Containers provide process, filesystem, and namespace isolation — but not all containers are created equal. A rootless Podman container has different isolation properties than a root Docker container. An attacker who escapes container isolation gains access to the host. This project measures and improves the isolation gap.
> **Out of scope:** Kubernetes pod security policies, VM-level isolation (gVisor, Kata Containers). Distrobox integration testing with desktop apps. This project focuses on Podman container security primitives.
> **Note on Distrobox:** Distrobox intentionally shares home directory, display, and network with the host for usability. It is NOT designed for security isolation — it is a development/tooling convenience. This project documents that distinction explicitly.

## 1) Mission

- **Problem statement:** CTF workflows use Distrobox containers to isolate tool environments (e.g., Kali tools in a Distrobox without contaminating the Arch host). But Distrobox is not a security boundary — it intentionally shares the home directory and display. Understanding what IS and IS NOT isolated in each container type is critical for correct threat modeling.
- **Why it matters:** Container isolation knowledge is a NET 412 and NET 210 competency. The patterns learned here are foundational for cloud-native environments where containers are the primary workload unit. The benchmark methodology also feeds into E3 (Secure Platform Baseline).

## 2) Difficulty

- Advanced (estimated 8–12 focused hours)

## 3) Execution Context

- **Host** — Arch Linux bare metal. Rootless Podman containers are created and benchmarked. All container operations run as the non-root user (rootless).

## 4) Prerequisites

- **Skills:** Completion of I1 (Systemd Service Hardening) and B2 (FHS Artifact Hunt). Understanding of Linux namespaces (PID, network, mount, user). Familiarity with `podman run` syntax.
- **Tools:** `podman`, `distrobox`, `skopeo`, `buildah`, `podman inspect`, `nsenter`, `capsh`, `tee`
- **Dependencies:**
  - `podman` installed: `pacman -Q podman`
  - `distrobox` installed: `pacman -Q distrobox` or from AUR
  - Rootless Podman configured (user namespaces enabled): `cat /proc/sys/user/max_user_namespaces` — should be > 0
  - `slirp4netns` or `pasta` for rootless networking: `pacman -Q slirp4netns`

## 5) Rollback Plan

- **Host Snapper pre:** `sudo snapper -c root create --description "pre-a4-container-benchmark"` — record snapshot number.
- **Host Snapper post:** `sudo snapper -c root create --description "post-a4-container-benchmark"`
- **VM snapshot:** N/A
- **Container rebuild command:** Remove all test containers and images: `podman rm -af && podman rmi -af`
- **Rollback trigger:** If host networking or user namespaces are disrupted: Snapper rollback restores `/etc/sysctl.d/` and related config. Then `sudo sysctl --system` to re-apply kernel params.

## 6) Project Plan

- **Phase A — Baseline Container Audit:** Run a baseline Alpine container and document all isolation properties: namespaces, capabilities, mounts, network.
- **Phase B — Distrobox Isolation Assessment:** Run a Distrobox container and document what IS shared with the host (home, display, audio) vs what is NOT (PID, some mounts).
- **Phase C — Hardening Application:** Apply security options to a Podman container: drop capabilities, add seccomp, set read-only rootfs, no-new-privileges.
- **Phase D — Benchmark Comparison:** Compare isolation properties before and after hardening. Document the delta.

## 7) Walkthrough

### Step 1 — Environment setup and pre-snapshot

```bash
mkdir -p evidence/a4

# Record host environment
uname -a | tee evidence/a4/env_uname.txt
id | tee evidence/a4/env_user.txt

# Check Podman and rootless configuration
podman version | tee evidence/a4/podman_version.txt
podman info | tee evidence/a4/podman_info.txt

# Check user namespace support
cat /proc/sys/user/max_user_namespaces | tee evidence/a4/user_ns_max.txt
cat /proc/sys/kernel/unprivileged_userns_clone 2>/dev/null | tee evidence/a4/unprivileged_userns.txt || echo "sysctl not present" | tee evidence/a4/unprivileged_userns.txt

# Check subuid/subgid configuration
cat /etc/subuid | grep "$(id -un)" | tee evidence/a4/subuid_config.txt
cat /etc/subgid | grep "$(id -un)" | tee evidence/a4/subgid_config.txt

# Snapper pre-snapshot
sudo snapper -c root create --description "pre-a4-container-benchmark" --print-number \
  | tee evidence/a4/snapshot_pre_num.txt

# Pull Alpine image for testing
podman pull alpine:latest 2>&1 | tee evidence/a4/alpine_pull.txt
podman images | tee evidence/a4/podman_images.txt
```

Expected:
- `user_ns_max.txt` shows a large number (≥ 10000 for rootless to work).
- `subuid_config.txt` shows your username with a UID range (e.g., `user:100000:65536`).

### Step 2 — Phase A: Baseline container isolation audit

```bash
# Run a baseline container and capture its isolation properties
podman run --name a4-baseline --rm -d alpine sleep 3600
CTRPID=$(podman inspect a4-baseline --format '{{.State.Pid}}')

echo "Container PID on host: $CTRPID" | tee evidence/a4/baseline_container_pid.txt

# Namespace isolation
ls -la /proc/$CTRPID/ns/ | tee evidence/a4/baseline_namespaces.txt

# Capabilities inside the container
podman exec a4-baseline sh -c "cat /proc/1/status | grep -E '^Cap'" \
  | tee evidence/a4/baseline_capabilities_raw.txt

# Decode capabilities
CAP_EFF=$(podman exec a4-baseline sh -c "grep CapEff /proc/1/status | awk '{print \$2}'")
capsh --decode="$CAP_EFF" 2>/dev/null | tee evidence/a4/baseline_capabilities_decoded.txt \
  || echo "capsh not available in container — raw: $CAP_EFF" | tee evidence/a4/baseline_capabilities_decoded.txt

# Filesystem visibility from inside container
podman exec a4-baseline ls /proc/1/root/ | tee evidence/a4/baseline_fs_root.txt
podman exec a4-baseline ls / | tee evidence/a4/baseline_container_root.txt
podman exec a4-baseline mount | tee evidence/a4/baseline_mounts.txt

# Network isolation
podman exec a4-baseline ip addr | tee evidence/a4/baseline_network.txt

# Process isolation — can container see host PIDs?
HOST_PID_COUNT=$(ls /proc | grep -c '^[0-9]')
CTR_PID_COUNT=$(podman exec a4-baseline sh -c "ls /proc | grep -c '^[0-9]'")
echo "Host PID count: $HOST_PID_COUNT" | tee evidence/a4/baseline_pid_comparison.txt
echo "Container PID count: $CTR_PID_COUNT" | tee -a evidence/a4/baseline_pid_comparison.txt
[ "$CTR_PID_COUNT" -lt "$HOST_PID_COUNT" ] \
  && echo "PASS: PID namespace isolation active" | tee -a evidence/a4/baseline_pid_comparison.txt \
  || echo "WARN: Container sees same PIDs as host" | tee -a evidence/a4/baseline_pid_comparison.txt

podman stop a4-baseline 2>/dev/null || true
```

Expected:
- Container has its own PID namespace (few PIDs vs host's many).
- Container has a separate network namespace with a virtual interface.
- Container's `/` is isolated from host's root filesystem.

### Step 3 — Phase B: Distrobox isolation assessment

```bash
# Create a minimal Distrobox container for assessment
distrobox create --name a4-distrobox-assess --image alpine:latest --yes 2>&1 \
  | tee evidence/a4/distrobox_create.txt

# Start and enter non-interactively
distrobox enter a4-distrobox-assess -- sh -c "
  echo '=== Distrobox: HOME visibility ===' && ls \$HOME | head -10
  echo '=== Distrobox: Network ===' && ip addr 2>/dev/null || ifconfig 2>/dev/null
  echo '=== Distrobox: Hostname ===' && hostname
  echo '=== Distrobox: UID inside ===' && id
  echo '=== Distrobox: /etc/passwd ===' && wc -l /etc/passwd
" 2>/dev/null | tee evidence/a4/distrobox_isolation_report.txt

# Compare: what is shared vs isolated in Distrobox
cat | tee evidence/a4/distrobox_isolation_matrix.txt << 'EOF'
=== Distrobox Isolation Matrix ===

SHARED WITH HOST (by design):
- $HOME directory (full read/write access)
- Host network namespace (same IPs as host)
- /run/user/$UID (XDG runtime)
- /tmp/.X11-unix (display server socket)
- PulseAudio/PipeWire sockets (if present)
- Hostname (inherits host hostname by default)

ISOLATED FROM HOST:
- Root filesystem (/ is container image)
- /usr, /bin, /lib (container's package set)
- PID namespace (separate process tree)
- Most /etc files (container's own config)

SECURITY IMPLICATION:
- Distrobox is NOT a security boundary
- Any file the host user can write, Distrobox can too
- Use Distrobox for tooling isolation only
- Use hardened Podman containers for security isolation
EOF

cat evidence/a4/distrobox_isolation_matrix.txt

# Stop distrobox
distrobox stop a4-distrobox-assess 2>/dev/null || true
```

Expected:
- `distrobox_isolation_report.txt` shows the home directory is shared.
- `distrobox_isolation_matrix.txt` documents the sharing boundaries clearly.

### Step 4 — Phase C: Apply hardening to a Podman container

```bash
# Run a hardened container with all security controls applied
# Compare against the baseline container from Phase A

podman run --name a4-hardened --rm -d \
  --security-opt no-new-privileges \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --security-opt seccomp=/usr/share/containers/seccomp.json \
  --network slirp4netns:allow_host_loopback=false \
  --user 65534:65534 \
  --userns=keep-id \
  alpine sleep 3600 2>&1 | tee evidence/a4/hardened_run_output.txt

HCTRPID=$(podman inspect a4-hardened --format '{{.State.Pid}}' 2>/dev/null)
echo "Hardened container PID: $HCTRPID" | tee evidence/a4/hardened_container_pid.txt

# Capture hardened container isolation properties
podman inspect a4-hardened --format json \
  | python3 -c "import sys,json; d=json.load(sys.stdin)[0]; \
    print('SecurityOpt:', d['HostConfig']['SecurityOpt']); \
    print('CapAdd:', d['HostConfig']['CapAdd']); \
    print('CapDrop:', d['HostConfig']['CapDrop']); \
    print('ReadonlyRootfs:', d['HostConfig']['ReadonlyRootfs']); \
    print('UsernsMode:', d['HostConfig']['UsernsMode'])" \
  2>/dev/null | tee evidence/a4/hardened_inspect.txt

# Capture hardened capabilities
CAP_EFF_HARDENED=$(podman exec a4-hardened sh -c "grep CapEff /proc/1/status | awk '{print \$2}'" 2>/dev/null)
echo "Hardened CapEff (raw): $CAP_EFF_HARDENED" | tee evidence/a4/hardened_capabilities.txt
capsh --decode="$CAP_EFF_HARDENED" 2>/dev/null | tee -a evidence/a4/hardened_capabilities.txt

# Test: can the hardened container write to its own rootfs?
podman exec a4-hardened sh -c "touch /test-write 2>&1; echo exit=$?" \
  | tee evidence/a4/hardened_readonly_test.txt

podman stop a4-hardened 2>/dev/null || true
```

Expected:
- `hardened_inspect.txt` shows `ReadonlyRootfs: True`, `CapDrop: ALL`.
- `hardened_readonly_test.txt` shows "Read-only file system" — rootfs is immutable.
- Hardened capabilities should show empty or only `net_bind_service`.

### Step 5 — Phase D: Benchmark comparison

```bash
cat | tee evidence/a4/benchmark_comparison.txt << 'EOF'
=== A4 Container Isolation Benchmark Summary ===

                    | Baseline | Hardened | Distrobox
--------------------|----------|----------|----------
PID Namespace       | YES      | YES      | YES
Network Namespace   | YES      | YES      | NO (host)
Mount Namespace     | YES      | YES      | PARTIAL
User Namespace      | YES      | YES      | YES
Read-only rootfs    | NO       | YES      | NO
No new privs        | NO       | YES      | NO
Capabilities        | DEFAULT  | NONE+    | DEFAULT
Seccomp profile     | DEFAULT  | CUSTOM   | DEFAULT
Home dir shared     | NO       | NO       | YES
Security boundary   | WEAK     | STRONG   | NONE

+ = cap_net_bind_service only

Recommendation for CTF/lab use:
- Development tools (Kali, custom envs): Distrobox (convenience over isolation)
- Lab services needing isolation: Hardened Podman containers
- High-security workloads: VM isolation (not containers)
EOF

cat evidence/a4/benchmark_comparison.txt

# Record final Snapper post-snapshot
sudo snapper -c root create --description "post-a4-container-benchmark" --print-number \
  | tee evidence/a4/snapshot_post_num.txt

# Cleanup all test containers and images
podman rm -af 2>/dev/null
distrobox rm --force a4-distrobox-assess 2>/dev/null || true
```

## 8) Validation

```bash
# Confirm Podman is working
podman version > /dev/null 2>&1 \
  && echo "PASS: Podman accessible" \
  || echo "FAIL: Podman not working"

# Confirm baseline evidence captured
[ -s evidence/a4/baseline_namespaces.txt ] \
  && echo "PASS: Baseline namespaces captured" \
  || echo "FAIL: Baseline namespace evidence missing"

# Confirm hardened container evidence captured
[ -s evidence/a4/hardened_inspect.txt ] \
  && echo "PASS: Hardened container evidence captured" \
  || echo "FAIL: Hardened container evidence missing"

# Confirm read-only rootfs test ran
grep -q "Read-only\|Permission denied\|exit=1" evidence/a4/hardened_readonly_test.txt \
  && echo "PASS: Read-only rootfs confirmed" \
  || echo "WARN: Read-only rootfs test inconclusive — check evidence/a4/hardened_readonly_test.txt"

# Confirm benchmark comparison exists
[ -s evidence/a4/benchmark_comparison.txt ] \
  && echo "PASS: Benchmark comparison written" \
  || echo "FAIL: Benchmark comparison missing"
```

## 9) Evidence

- **Output files:**
  - `evidence/a4/env_uname.txt`, `env_user.txt` — host environment
  - `evidence/a4/podman_version.txt` — Podman version
  - `evidence/a4/podman_info.txt` — Podman system info
  - `evidence/a4/user_ns_max.txt` — user namespace limit
  - `evidence/a4/subuid_config.txt`, `subgid_config.txt` — rootless UID ranges
  - `evidence/a4/snapshot_pre_num.txt` — Snapper pre-snapshot
  - `evidence/a4/alpine_pull.txt` — image pull output
  - `evidence/a4/baseline_container_pid.txt` — baseline container PID
  - `evidence/a4/baseline_namespaces.txt` — namespace isolation state
  - `evidence/a4/baseline_capabilities_raw.txt` — raw capability bitmask
  - `evidence/a4/baseline_capabilities_decoded.txt` — decoded capabilities
  - `evidence/a4/baseline_mounts.txt` — container mount table
  - `evidence/a4/baseline_network.txt` — container network interfaces
  - `evidence/a4/baseline_pid_comparison.txt` — PID namespace test
  - `evidence/a4/distrobox_create.txt` — Distrobox creation output
  - `evidence/a4/distrobox_isolation_report.txt` — Distrobox sharing findings
  - `evidence/a4/distrobox_isolation_matrix.txt` — isolation boundary doc
  - `evidence/a4/hardened_run_output.txt` — hardened container start
  - `evidence/a4/hardened_inspect.txt` — hardened container config
  - `evidence/a4/hardened_capabilities.txt` — hardened capabilities
  - `evidence/a4/hardened_readonly_test.txt` — read-only rootfs test
  - `evidence/a4/benchmark_comparison.txt` — isolation benchmark matrix
  - `evidence/a4/snapshot_post_num.txt` — Snapper post-snapshot
- **Hash manifest:**

```bash
find evidence/a4 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/a4/SHA256SUMS.txt
cat evidence/a4/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `podman run` fails with "cannot create user namespace: unshare: operation not permitted."
  - **Cause:** User namespaces are disabled in the kernel.
  - **Fix:** `sudo sysctl -w user.max_user_namespaces=28633 && sudo sysctl -w kernel.unprivileged_userns_clone=1` (if applicable). Add to `/etc/sysctl.d/99-rootless-podman.conf`.
  - **Rollback trigger:** Snapper rollback if sysctl.conf was modified.

- **Symptom:** Hardened container fails to start with seccomp errors.
  - **Cause:** Custom seccomp profile blocks a syscall the container needs at startup.
  - **Fix:** Use `--security-opt seccomp=unconfined` temporarily to identify the blocking syscall. Then modify the seccomp profile to allow it. Or use the default profile: remove `--security-opt seccomp=...`.
  - **Rollback trigger:** N/A — container was not started.

- **Symptom:** Distrobox fails to enter with "container not found."
  - **Cause:** Distrobox create failed silently.
  - **Fix:** `distrobox list` to see all containers. `distrobox rm a4-distrobox-assess` to clean up and recreate.
  - **Rollback trigger:** `podman rm a4-distrobox-assess` if Distrobox does not clean up.

## 11) Sources

- [Podman documentation](https://docs.podman.io/en/latest/)
- [Distrobox project — GitHub](https://github.com/89luca89/distrobox)
- [Open Container Initiative — Runtime Security Specification](https://github.com/opencontainers/runtime-spec)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Arch Wiki — Podman](https://wiki.archlinux.org/title/Podman)
- [Red Hat — Understanding rootless containers](https://www.redhat.com/en/blog/understanding-root-inside-and-outside-container)
- [NIST SP 800-190 — Application Container Security Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)

## 12) Stretch Goals

- Create a custom seccomp profile using `podman run --security-opt seccomp=profile.json`. Start with the default profile and iteratively remove syscalls until the container breaks, then document the minimal required syscall set for Alpine `sleep`.
- Set up Podman systemd integration: generate a systemd unit for a hardened container using `podman generate systemd` and deploy it with the hardening options applied. Verify it starts on boot.
- Implement rootless network isolation using `--network none` for a batch-processing container that reads input files and writes output files, with no network access needed. Document the use case.
- Write a `a4-container-audit.sh` script that runs all existing containers through the `podman inspect` security checklist and reports which containers have `ReadonlyRootfs=false`, have extra capabilities beyond the default set, or run as root inside the container.
