---
aliases:
  - Docker Podman
  - Container Isolation
  - Distrobox
  - Rootless Containers
  - Container Security
tags:
  - linux
  - containers
  - docker
  - podman
  - distrobox
  - net412
  - net210
  - phase5
date: 2026-05-10
---

# 03 — Docker/Podman for Isolated Workloads

> [!info] Why This Matters
> Containers give you isolation without the overhead of full VMs. Your CTF tools in Distrobox, your Apex Predator service in Podman — these run in kernel namespaces that separate processes, filesystems, and network stacks. Understanding the security model means you can both harden containers and (in CTF/pentest scenarios) understand their escape vectors.

## Container vs. VM vs. Distrobox

| Feature | VM (QEMU/KVM) | Docker/Podman | Distrobox |
|---|---|---|---|
| Kernel | Separate | Shared (host) | Shared (host) |
| Isolation | Full hardware | Namespace-based | Namespace-based |
| Startup | 5-60 seconds | <1 second | <2 seconds |
| Use case | Full OS testing | Service deployment | Developer environments |
| CTF use | Binary exploitation | CTF challenge isolation | Full toolkit environments |
| Your stack | Proxmox guests | Apex Predator | CTF tool environments |

---

## Linux Namespaces — The Container Foundation

Containers are kernel namespaces. There is no "container technology" — it's all kernel primitives.

```
Namespace        | What it Isolates
-----------------+------------------------------------------------
pid              | Process IDs (container PID 1 ≠ host PID 1)
net              | Network interfaces, routes, iptables
mnt              | Filesystem mount points
uts              | Hostname and domain name
ipc              | System V IPC, POSIX message queues
user             | UIDs/GIDs (rootless containers — UID 0 in container ≠ root on host)
cgroup           | Resource limits (CPU, memory, I/O)
time             | System clock offsets (Linux 5.6+)
```

```bash
# See which namespaces a process is in
ls -la /proc/$$/ns/

# See container namespaces (for a running container)
CONTAINER_PID=$(podman inspect --format '{{.State.Pid}}' mycontainer)
ls -la /proc/${CONTAINER_PID}/ns/

# Enter a running container's namespaces manually (without podman exec)
sudo nsenter -t ${CONTAINER_PID} -m -u -i -n -p -- /bin/bash
```

---

## Podman — Rootless Container Runtime

Podman is Docker-compatible but daemonless and rootless. It's the Arch/Fedora/RHEL standard.

> [!tip] Podman vs Docker on Arch
> Podman's rootless mode maps container UIDs to a range in `/etc/subuid` and `/etc/subgid`. This means container root (UID 0) maps to an unprivileged UID on the host — a critical security improvement over Docker's daemon model.

### Installation and Setup

```bash
# Install Podman
sudo pacman -S podman podman-compose

# Set up user namespace mapping (required for rootless)
# These should already exist, but verify:
cat /etc/subuid     # dave:100000:65536
cat /etc/subgid     # dave:100000:65536

# If missing, add them:
sudo usermod --add-subuids 100000-165535 dave
sudo usermod --add-subgids 100000-165535 dave

# Enable lingering (allows rootless containers to run without active login)
loginctl enable-linger dave

# Verify rootless Podman works
podman run hello-world
podman info | grep -E "rootless|cgroupVersion"
```

### Basic Container Operations

```bash
# Pull an image
podman pull archlinux:latest
podman pull kalilinux/kali-rolling

# Run a container
podman run -it archlinux /bin/bash              # interactive
podman run -d --name web nginx                  # detached (background)
podman run --rm alpine echo "hello world"       # auto-remove after exit

# List containers
podman ps                  # running containers
podman ps -a               # all containers including stopped

# Execute in running container
podman exec -it web /bin/bash
podman exec web ls /etc/nginx/

# Stop and remove
podman stop web
podman rm web

# Remove image
podman rmi nginx

# View logs
podman logs web
podman logs -f web         # follow
```

### Volume Mounts

```bash
# Mount a host directory into a container (read-write)
podman run -it -v /home/dave/ctf/:/ctf:Z archlinux bash
# :Z = relabel with SELinux context (required on SELinux systems, harmless on AppArmor)

# Read-only mount (protect your host data)
podman run -it -v /etc/hosts:/etc/hosts:ro archlinux bash

# Named volume (Podman-managed, survives container deletion)
podman volume create mydata
podman run -it -v mydata:/data archlinux bash
```

### Port Mapping

```bash
# Map container port to host port
podman run -d -p 8080:80 nginx          # host:8080 → container:80
podman run -d -p 127.0.0.1:8080:80 nginx   # localhost only (more secure)

# Rootless: host ports < 1024 require privilege — use ports > 1024
podman run -d -p 8443:443 nginx         # use 8443, not 443

# Check port mapping
podman port web
```

---

## Distrobox — CTF Tool Environments on Arch

Distrobox wraps containers with deep host integration — you get Ubuntu/Kali inside Arch without VMs.

```bash
# Install Distrobox
sudo pacman -S distrobox
# Requires Podman (already installed) or Docker

# Create a Kali container for CTF work
distrobox create --name kali-ctf --image kalilinux/kali-rolling
distrobox create --name ubuntu-forensics --image ubuntu:22.04

# Enter the container
distrobox enter kali-ctf

# Inside kali-ctf: install tools (without affecting Arch host)
sudo apt update && sudo apt install -y \
  nmap metasploit-framework john hashcat \
  volatility3 binwalk foremost \
  gobuster ffuf

# Export an application to run from Arch host (GUI works too)
distrobox-export --app firefox    # makes Kali's Firefox available on Arch desktop
distrobox-export --bin /usr/bin/john --export-path ~/.local/bin/

# List containers
distrobox list

# Stop and remove
distrobox stop kali-ctf
distrobox rm kali-ctf
```

> [!tip] Distrobox Integration Points
> Distrobox containers share:
> - Your home directory (`~`)
> - X11/Wayland display
> - Your user's UID/GID (no permission issues between host and container)
> - Systemd D-Bus (if using systemd inside)
>
> They do NOT share:
> - Package databases (apt inside Kali is separate from pacman)
> - System libraries at `/usr/lib`
> - Running processes (separate PID namespace)

---

## Building Custom Container Images

### Containerfile (Dockerfile-compatible)

```dockerfile
# /home/dave/apex-predator/Containerfile
FROM rust:1.75-alpine AS builder

WORKDIR /build
COPY Cargo.toml Cargo.lock ./
COPY src/ ./src/

# Build the Rust binary
RUN cargo build --release --locked

# Production image — minimal base
FROM alpine:latest

# Create non-root user
RUN addgroup -g 1001 apex && \
    adduser -D -u 1001 -G apex apex_engine

WORKDIR /app

# Copy only the binary
COPY --from=builder /build/target/release/apex-predator /app/apex-predator
COPY config/default.toml /app/config.toml

# Set correct ownership
RUN chown -R apex_engine:apex /app

# Switch to non-root user
USER apex_engine

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD /app/apex-predator --health-check || exit 1

EXPOSE 8080
ENTRYPOINT ["/app/apex-predator"]
CMD ["--config", "/app/config.toml"]
```

```bash
# Build the image
podman build -t apex-predator:latest -f Containerfile .

# Inspect the image
podman inspect apex-predator:latest
podman history apex-predator:latest

# Check for vulnerabilities (install trivy)
paru -S trivy
trivy image apex-predator:latest

# Run the production container
podman run -d \
  --name apex-predator \
  --user apex_engine \
  -p 127.0.0.1:8080:8080 \
  -v /etc/apex/config.toml:/app/config.toml:ro \
  -v /var/lib/apex:/app/data:Z \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  apex-predator:latest
```

---

## Generating systemd Units from Podman

Podman can generate systemd service files for your containers, making them first-class system services.

```bash
# Generate systemd unit for a running container
podman generate systemd --name apex-predator --files --new

# This creates: container-apex-predator.service

# Install as user service (rootless)
mkdir -p ~/.config/systemd/user/
mv container-apex-predator.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now container-apex-predator.service

# Or as system service (requires root container)
sudo mv container-apex-predator.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-apex-predator.service

# Check status
systemctl --user status container-apex-predator.service
journalctl --user -u container-apex-predator.service -f
```

---

## Container Security — Hardening and Escape Prevention

### Security Options

```bash
# Run with all capabilities dropped
podman run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Read-only root filesystem (prevent persistence)
podman run --read-only --tmpfs /tmp --tmpfs /run nginx

# Disable privilege escalation (setuid/setgid)
podman run --security-opt=no-new-privileges nginx

# Drop ambient capabilities
podman run --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --security-opt=seccomp=/etc/containers/seccomp.json \
  nginx

# Limit resources
podman run --memory=512m --cpus=0.5 --pids-limit=100 nginx

# Run as specific UID (don't run as root in containers)
podman run --user 1001:1001 nginx
```

### Common Container Escape Vectors (NET 377 Knowledge)

```bash
# 1. Privileged mode — gives container near-host-root access
podman run --privileged ...   # NEVER use in production

# 2. Host PID namespace — container can see all host processes
podman run --pid=host ...     # avoid unless debugging

# 3. Host network namespace — no network isolation
podman run --network=host ... # avoid unless performance-critical

# 4. Mounted Docker socket — full host control
podman run -v /var/run/docker.sock:/var/run/docker.sock ...  # CRITICAL RISK

# 5. Writable /proc or /sys mounts (container breakout)
# Avoid mounting /proc, /sys from host unless read-only for monitoring

# Audit running containers for dangerous options
podman inspect $(podman ps -q) | \
  jq '.[] | {Name: .Name, Privileged: .HostConfig.Privileged, \
             PidMode: .HostConfig.PidMode, Mounts: [.Mounts[].Source]}'
```

---

## podman-compose — Multi-Container Orchestration

```bash
# Install
sudo pacman -S podman-compose

# Example compose file for CTF forensics lab
# /home/dave/ctf-lab/docker-compose.yml
```

```yaml
version: '3.8'

services:
  kali:
    image: kalilinux/kali-rolling
    container_name: ctf-kali
    stdin_open: true
    tty: true
    volumes:
      - ./shared:/shared:Z
    networks:
      - ctf-net
    cap_drop:
      - ALL
    cap_add:
      - NET_RAW          # for tcpdump in container
      - NET_ADMIN
    security_opt:
      - no-new-privileges

  target:
    image: vulhub/weblogic:12.2.1.3-2019-2725
    container_name: ctf-target
    networks:
      - ctf-net
    ports:
      - "7001:7001"     # WebLogic port

networks:
  ctf-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

```bash
# Start the lab
podman-compose -f docker-compose.yml up -d

# Connect to Kali and start attacking target
podman exec -it ctf-kali bash

# Stop lab
podman-compose down
```

---

## Practical Lab: Deploy Apex Predator in a Container

```bash
# 1. Check Podman setup
podman info | grep -E "rootless|version|cgroupVersion"
cat /etc/subuid | grep dave
cat /etc/subgid | grep dave

# 2. Pull and run a test container
podman run --rm --read-only --cap-drop=ALL --security-opt=no-new-privileges \
  alpine echo "Secure container test passed"

# 3. Create a CTF Distrobox environment
distrobox create --name forensics-lab --image ubuntu:22.04
distrobox enter forensics-lab

# Inside: install forensics tools
# sudo apt update && sudo apt install -y volatility3 foremost binwalk sleuthkit

# 4. List all containers and images
podman ps -a
podman images

# 5. Check container security posture
podman inspect $(podman ps -q 2>/dev/null) 2>/dev/null | \
  jq -r '.[] | "\(.Name): privileged=\(.HostConfig.Privileged // false)"' 2>/dev/null \
  || echo "No running containers to inspect"

# 6. Check generated systemd units
ls ~/.config/systemd/user/ 2>/dev/null
```

---

## Troubleshooting Scenario: Rootless Podman — Permission Denied

**Symptom:** `podman run -v /etc/hosts:/etc/hosts alpine cat /etc/hosts` fails with permission denied.

**Root Cause:** In rootless mode, container processes run as a mapped UID (e.g., host UID 100000), which has no permission to read host root-owned files.

```bash
# Diagnose: what UID is the container running as?
podman run alpine id
# → uid=0(root) gid=0(root) ...  (inside container)

# But on host, that "root" is actually:
cat /proc/$(podman inspect --format '{{.State.Pid}}' <container_name>)/status | grep "^Uid"
# → Uid: 100000  100000  100000  100000

# Fix 1: Set correct permissions on the file
# (only if you control the file)
sudo chmod 644 /etc/hosts   # it already is, so this is fine

# Fix 2: Use :z or :Z volume option for SELinux contexts
podman run -v /etc/hosts:/etc/hosts:ro,z alpine cat /etc/hosts

# Fix 3: Use --userns=keep-id (map container UID to your actual UID)
podman run --userns=keep-id -v /home/dave/data:/data alpine ls -la /data
# Now container UID matches your actual UID — no permission issues
```

---

## 🏁 Proof of Work — Phase 5.3 Mini-CTF

> [!example] Challenge: Secure Service Containerization
>
> ```bash
> # Deploy a hardened container and audit its security posture
>
> # 1. Run a hardened nginx container
> podman run -d \
>   --name hardened-web \
>   --read-only \
>   --tmpfs /tmp:rw,noexec,nosuid,size=50m \
>   --tmpfs /var/run:rw,noexec,nosuid \
>   --tmpfs /var/cache/nginx:rw,noexec,nosuid \
>   --cap-drop=ALL \
>   --cap-add=NET_BIND_SERVICE \
>   --cap-add=CHOWN \
>   --cap-add=SETUID \
>   --cap-add=SETGID \
>   --security-opt=no-new-privileges \
>   --memory=128m \
>   --cpus=0.25 \
>   -p 127.0.0.1:8888:80 \
>   nginx:alpine
>
> # 2. Test it works
> curl -s http://localhost:8888/ | head -5
>
> # 3. Audit its security posture
> {
>   echo "=== CONTAINER SECURITY AUDIT ==="
>   echo "Container: hardened-web"
>   echo "Date: $(date -u)"
>   echo ""
>
>   echo "--- Security Options ---"
>   podman inspect hardened-web | jq '.[0].HostConfig | {
>     Privileged: .Privileged,
>     ReadonlyRootfs: .ReadonlyRootfs,
>     CapAdd: .CapAdd,
>     CapDrop: .CapDrop,
>     SecurityOpt: .SecurityOpt,
>     Memory: .Memory,
>     NanoCpus: .NanoCpus
>   }'
>
>   echo ""
>   echo "--- Running Process in Container ---"
>   podman exec hardened-web ps aux
>
>   echo ""
>   echo "--- Network Configuration ---"
>   podman port hardened-web
>
> } | tee /tmp/phase5_3_pow.txt
>
> # 4. Generate systemd unit
> podman generate systemd --name hardened-web > ~/.config/systemd/user/hardened-web.service
>
> # 5. Clean up
> podman stop hardened-web
> podman rm hardened-web
>
> sha256sum /tmp/phase5_3_pow.txt
> ```
>
> **Submit:** Hash of audit file. Verify: `ReadonlyRootfs=true`, `Privileged=false`, all capabilities dropped except the 4 required ones.

---

← [[02-Btrfs-Snapshots]] | [[../README|Back to MoC]] →
