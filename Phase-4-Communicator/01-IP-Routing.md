---
aliases:
  - IP Routing
  - Linux Networking
  - ip command
  - nmcli
tags:
  - linux
  - networking
  - routing
  - net412
  - net377
  - phase4
date: 2026-05-10
---

# 01 — IP Routing & Interfaces

> [!info] Why This Matters
> You cannot attack, defend, or administrate a network you don't understand. The `ip` command is your complete network stack interface. If you're still using `ifconfig` and `route`, you're working with deprecated tools.

## The `ip` Command — Modern Linux Networking

`ip` replaced `ifconfig`, `route`, `arp`, and `netstat` for interface/routing tasks.

### Interface Management

```bash
# List all interfaces with addresses
ip addr show
ip a                    # short form
ip a show eth0          # specific interface

# Typical output:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
#     link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
#     inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic eth0
#     inet6 fe80::1/64 scope link

# Interface state
ip link show
ip link show eth0

# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Set MTU (useful for Proxmox VXLAN tunnels)
sudo ip link set eth0 mtu 1450

# Assign IP address
sudo ip addr add 192.168.1.200/24 dev eth0

# Remove IP address
sudo ip addr del 192.168.1.200/24 dev eth0

# Flush all addresses from interface
sudo ip addr flush dev eth0
```

### Routing Table Management

```bash
# View routing table
ip route show
ip r                    # short

# Typical output:
# default via 192.168.1.1 dev eth0 proto dhcp metric 100
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
# 10.10.0.0/24 via 192.168.1.254 dev eth0

# Add static route
sudo ip route add 10.10.0.0/24 via 192.168.1.254 dev eth0

# Add default gateway
sudo ip route add default via 192.168.1.1

# Delete route
sudo ip route del 10.10.0.0/24

# Check which interface/gateway handles a destination
ip route get 8.8.8.8
# → 8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.100

# Add a route through your Proxmox interface (lab isolation)
sudo ip route add 172.16.0.0/16 via 192.168.1.254 dev eth0
```

> [!tip] Proxmox Lab Routing
> Your Proxmox VMs likely live on an internal bridge (`vmbr0`). To reach them from your Arch host:
> ```bash
> sudo ip route add 192.168.100.0/24 via <proxmox-host-ip> dev eth0
> ```
> Where `192.168.100.0/24` is your Proxmox VM network and `<proxmox-host-ip>` is your Proxmox server's main IP.

### Neighbor (ARP) Table

```bash
# View ARP cache
ip neigh show
ip n

# Example output:
# 192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE

# Add static ARP entry (prevent ARP spoofing for critical hosts)
sudo ip neigh add 192.168.1.1 lladdr aa:bb:cc:dd:ee:ff dev eth0 nud permanent

# Delete ARP entry
sudo ip neigh del 192.168.1.1 dev eth0

# Flush ARP cache
sudo ip neigh flush dev eth0
```

---

## Network Namespaces (Container Networking Foundation)

Network namespaces are the building block of container networking.

```bash
# List namespaces
ip netns list

# Create a namespace
sudo ip netns add lab_ns

# Run a command inside the namespace
sudo ip netns exec lab_ns ip addr show
# → only loopback exists (isolated)

# Create a veth pair (virtual ethernet pipe)
sudo ip link add veth0 type veth peer name veth1

# Move one end into the namespace
sudo ip link set veth1 netns lab_ns

# Configure the pair
sudo ip addr add 10.99.0.1/30 dev veth0
sudo ip link set veth0 up
sudo ip netns exec lab_ns ip addr add 10.99.0.2/30 dev veth1
sudo ip netns exec lab_ns ip link set veth1 up
sudo ip netns exec lab_ns ip link set lo up

# Test isolation
sudo ip netns exec lab_ns ping 10.99.0.1 -c 3

# Clean up
sudo ip netns del lab_ns
```

> [!info] Container Networking Insight
> When Podman or Docker creates a container, it creates a network namespace + veth pair + bridge exactly like the above. `ip netns list` and `ip link show` reveal the host-side of every container's network stack.

---

## nmcli — NetworkManager CLI (Persistent Config on Arch)

`ip` commands are ephemeral — they don't survive reboots. For persistent config, use NetworkManager.

```bash
# Show all connections
nmcli connection show
nmcli c s        # short form

# Show active connections
nmcli connection show --active

# Show device status
nmcli device status

# Create a static IP connection
nmcli connection add \
  type ethernet \
  ifname eth0 \
  con-name "static-home" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1 8.8.8.8"

# Activate a connection
nmcli connection up "static-home"

# Modify an existing connection
nmcli connection modify "static-home" ipv4.addresses "192.168.1.101/24"
nmcli connection up "static-home"

# Delete a connection
nmcli connection delete "static-home"

# Restart networking
sudo systemctl restart NetworkManager
```

---

## VLANs — Proxmox Lab Segmentation

VLANs let you segment your Proxmox lab traffic without separate physical switches.

```bash
# Create a VLAN interface (VLAN ID 100 on eth0)
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 192.168.100.1/24 dev eth0.100
sudo ip link set eth0.100 up

# Via nmcli (persistent)
nmcli connection add \
  type vlan \
  con-name "vlan100" \
  ifname eth0.100 \
  dev eth0 \
  id 100 \
  ipv4.method manual \
  ipv4.addresses 192.168.100.1/24

# Verify
ip link show | grep vlan
ip addr show eth0.100
```

---

## Practical Lab: Map Your Network Stack

```bash
# 1. Complete interface audit
echo "=== Network Interfaces ==="
ip addr show

echo ""
echo "=== Routing Table ==="
ip route show

echo ""
echo "=== ARP Cache ==="
ip neigh show

echo ""
echo "=== Active Connections Summary ==="
nmcli connection show --active 2>/dev/null || echo "NetworkManager not active"

# 2. Find your default gateway and trace it
GW=$(ip route | grep default | awk '{print $3}' | head -1)
echo "Default gateway: $GW"
ping -c 3 "$GW"

# 3. Find your external IP
curl -s ifconfig.me 2>/dev/null || curl -s icanhazip.com 2>/dev/null

# 4. Check if IP forwarding is enabled (required for routing/NAT)
cat /proc/sys/net/ipv4/ip_forward
# 0 = disabled, 1 = enabled (needed for Proxmox routing)

# Enable temporarily:
sudo sysctl net.ipv4.ip_forward=1
# Enable permanently:
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forwarding.conf
sudo sysctl -p /etc/sysctl.d/99-forwarding.conf
```

---

## Troubleshooting Scenario: Can't Reach Proxmox VM Network

**Symptom:** You can SSH to the Proxmox host but can't reach VMs on `192.168.100.0/24`.

**Step 1: Check routing**
```bash
ip route get 192.168.100.1
# If it says "unreachable" or routes to wrong interface → missing route

# Add the route
sudo ip route add 192.168.100.0/24 via <proxmox-host-ip>
ip route get 192.168.100.1    # re-verify
```

**Step 2: Check if Proxmox is forwarding packets**
```bash
# On the Proxmox host:
ssh proxmox-host "cat /proc/sys/net/ipv4/ip_forward"
# Must be 1

# Check iptables on Proxmox (default is restrictive)
ssh proxmox-host "iptables -L FORWARD -n | head -20"
```

**Step 3: Check VM's default gateway**
```bash
# On the VM:
ssh vm-user@192.168.100.10 "ip route show"
# VM must have a route back to your Arch machine, via Proxmox bridge
```

**Step 4: Packet trace**
```bash
# On Arch, trace if packets are leaving
sudo tcpdump -i eth0 dst 192.168.100.10 -n

# On Proxmox host, check if packets are arriving
ssh proxmox-host "sudo tcpdump -i vmbr0 host 192.168.100.10 -n -c 10"
```

---

## 🏁 Proof of Work — Phase 4.1 Mini-CTF

> [!example] Challenge: Network Stack Audit
>
> ```bash
> # Complete network state snapshot
> {
>   echo "=== NETWORK AUDIT: $(hostname) ==="
>   echo "Date: $(date -u)"
>   echo ""
>   echo "--- Interfaces ---"
>   ip addr show | grep -E "^[0-9]+:|inet "
>   echo ""
>   echo "--- Routes ---"
>   ip route show
>   echo ""
>   echo "--- ARP Table ---"
>   ip neigh show | grep -v "FAILED"
>   echo ""
>   echo "--- IP Forwarding ---"
>   echo "IPv4 forwarding: $(cat /proc/sys/net/ipv4/ip_forward)"
>   echo "IPv6 forwarding: $(cat /proc/sys/net/ipv6/conf/all/forwarding)"
>   echo ""
>   echo "--- Network Namespaces ---"
>   ip netns list 2>/dev/null || echo "None"
> } | tee /tmp/phase4_1_pow.txt
>
> sha256sum /tmp/phase4_1_pow.txt
> ```
>
> **Submit:** Hash of the audit file. If you have a Proxmox lab, include a route to your VM network.

---

← [[00-Phase4-Overview|Phase 4 Overview]] | Next: [[02-Network-Tools]] →
