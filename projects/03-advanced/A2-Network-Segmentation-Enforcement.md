---
title: A2 - Network Segmentation Enforcement
aliases:
  - a2-network-segmentation
  - vlan-segmentation-advanced
tags:
  - advanced
  - networking
  - segmentation
  - vlan
  - proxmox
  - evidence
  - net412
  - net377
date: 2026-05-10
---

# A2 — Network Segmentation Enforcement

> [!warning] Operating Assumptions & Threat Model
> **Context:** Proxmox VE hypervisor lab. Network segmentation is implemented using Linux bridges and optional VLAN tagging at the Proxmox level. All changes affect lab VMs only — the physical host network (internet access, management interface) is not modified.
> **Risk level:** High — incorrect network configuration can isolate VMs from each other or from the internet, or create unintended routing paths. Snapper and VM snapshots are mandatory before any changes.
> **Threat model:** A flat lab network where all VMs can freely communicate with each other is an attacker's dream — lateral movement is unrestricted. Network segmentation enforces the principle of least connectivity: each VM can only reach the other hosts it explicitly needs to.
> **Hard rules:** (1) Do not modify the Proxmox management interface (typically `vmbr0` connected to the physical NIC). (2) Test all firewall and routing changes with a non-destructive validation step before considering the change complete.
> **Out of scope:** Physical switch VLAN trunking, OSPF/BGP routing, SD-WAN. This project covers intra-Proxmox segmentation using Linux bridges and iptables/nftables.

## 1) Mission

- **Problem statement:** A default Proxmox lab has all VMs sharing a single bridge (`vmbr0`), giving them unrestricted Layer 2 access to each other and potentially to the host management plane. Real infrastructure separates workloads into network segments (DMZ, management, storage, application) using VLANs or separate bridges. This project implements a basic two-segment architecture and validates it.
- **Why it matters:** Network segmentation is a foundational security architecture skill tested in NET 412 and NET 377. The segmentation built here is the prerequisite for A3 (Threat-Informed Detection Pack) — you need isolated segments to meaningfully detect east-west traffic.

## 2) Difficulty

- Advanced (estimated 8–14 focused hours)

## 3) Execution Context

- **VM** / **Proxmox host** — Configuration is applied to the Proxmox node (bridge/VLAN setup) and to guest VMs (network interface, IP, routing). All Proxmox CLI commands run on the node shell; VM network commands run inside the VMs.

## 4) Prerequisites

- **Skills:** Completion of I4 (Proxmox Backup Validation). Understanding of IP subnets, default gateways, and VLAN concepts. Familiarity with `ip` (iproute2) commands from Phase 4.
- **Tools:** `ip`, `bridge`, `nft` or `iptables`, `ping`, `ss`, `tcpdump`, `qm`, Proxmox web UI or `pvesh`
- **Dependencies:**
  - Proxmox VE ≥ 7 with at least two test VMs.
  - Physical host has at least one free NIC or VLAN-capable switch port (for VLAN sub-interfaces).
  - `nftables` or `iptables` installed on the Proxmox host for inter-segment routing policy.

## 5) Rollback Plan

- **Host Snapper pre:** N/A (Proxmox host filesystem is typically ext4, not Btrfs — use Proxmox node config backup instead).
- **VM snapshot (pre-change):** `qm snapshot <VMID> pre-a2-segmentation` for all participating VMs.
- **VM snapshot rollback:** `qm rollback <VMID> pre-a2-segmentation`
- **Proxmox network rollback:** Keep a backup of `/etc/network/interfaces` on the Proxmox node: `sudo cp /etc/network/interfaces /etc/network/interfaces.bak-a2`. If networking breaks, restore and run `ifreload -a`.
- **Container rebuild command:** N/A
- **Rollback trigger:** If any VM loses connectivity and a rollback of VM snapshots does not restore it, restore the Proxmox network config backup and reboot the affected VMs.

## 6) Project Plan

- **Phase A — Baseline Network Map:** Document the current bridge/VLAN layout, VM network assignments, and routing table.
- **Phase B — Segment Creation:** Create a second bridge (`vmbr2`) for an isolated lab segment. Assign VMs to appropriate segments.
- **Phase C — Inter-Segment Policy:** Configure nftables on the Proxmox host to enforce which segments can communicate, block unauthorized lateral movement, and log inter-segment traffic.
- **Phase D — Validation:** Prove that VMs in the same segment can communicate, and VMs in different segments cannot (without explicit firewall allow rules).

## 7) Walkthrough

### Step 1 — Baseline network documentation

```bash
mkdir -p evidence/a2

# On the Proxmox node: document current bridge configuration
cat /etc/network/interfaces | tee evidence/a2/network_interfaces_before.txt

# Show all bridges and their attached interfaces
bridge link show | tee evidence/a2/bridge_links_before.txt
brctl show 2>/dev/null | tee evidence/a2/brctl_before.txt || bridge link show | tee evidence/a2/bridge_info_before.txt

# Show routing table
ip route show | tee evidence/a2/routing_table_before.txt

# Show iptables/nftables current rules
sudo nft list ruleset 2>/dev/null | tee evidence/a2/nft_ruleset_before.txt
sudo iptables -L -n -v 2>/dev/null | tee evidence/a2/iptables_before.txt

# Document VM network assignments from Proxmox
for vmid in $(qm list | awk 'NR>1 {print $1}'); do
  echo "=== VMID $vmid ===" | tee -a evidence/a2/vm_network_assignments.txt
  qm config $vmid | grep -E "net[0-9]|name" | tee -a evidence/a2/vm_network_assignments.txt
done
```

Expected:
- `network_interfaces_before.txt` shows current bridges (likely just `vmbr0` and the physical NIC).
- `vm_network_assignments.txt` shows which bridge each VM NIC is connected to.

### Step 2 — Create VM snapshots and backup network config

```bash
# Backup network interfaces config
sudo cp /etc/network/interfaces /etc/network/interfaces.bak-a2
sha256sum /etc/network/interfaces | tee evidence/a2/network_interfaces_hash_before.txt

# Snapshot all participating VMs
for vmid in 101 102; do  # Replace with your actual VMIDs
  qm snapshot $vmid pre-a2-segmentation \
    --description "pre-a2 network segmentation - $(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    && echo "Snapshot created for VMID $vmid" | tee -a evidence/a2/vm_snapshots_created.txt
done
```

### Step 3 — Create isolated lab segment bridge (vmbr2)

```bash
# Design:
# vmbr0 = external/internet segment (existing, do not modify)
# vmbr1 = management segment (192.168.100.0/24) — if already exists
# vmbr2 = isolated lab segment (10.10.20.0/24) — NEW

# Add vmbr2 to /etc/network/interfaces
cat | sudo tee -a /etc/network/interfaces << 'EOF'

auto vmbr2
iface vmbr2 inet static
    address 10.10.20.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    # Isolated lab segment — no external uplink
    comment A2-lab-segment-isolated
EOF

# Apply the new configuration without reboot
sudo ifreload -a 2>&1 | tee evidence/a2/ifreload_output.txt

# Verify vmbr2 is up
ip addr show vmbr2 | tee evidence/a2/vmbr2_interface.txt
bridge link show | tee evidence/a2/bridge_links_after.txt
```

Expected:
- `vmbr2_interface.txt` shows `vmbr2` with `state UP` and IP `10.10.20.1/24`.

### Step 4 — Assign VMs to segments

Using the Proxmox web UI or `qm` CLI, update VM network assignments:

```bash
# VM 101: place on the external segment (vmbr0)
# VM 102: move to the isolated segment (vmbr2) — requires VM shutdown
VMID_ISOLATED=102

qm stop $VMID_ISOLATED 2>&1 | tee evidence/a2/vm_stop_for_reconfigure.txt
sleep 10

# Update net0 to use vmbr2
qm set $VMID_ISOLATED --net0 virtio,bridge=vmbr2 2>&1 | tee evidence/a2/vm_net_reconfigure.txt

# Start VM again
qm start $VMID_ISOLATED 2>&1 | tee evidence/a2/vm_start_after_reconfigure.txt
sleep 30

# Verify VM network from Proxmox
qm config $VMID_ISOLATED | grep net | tee evidence/a2/vm102_net_config.txt
```

Expected:
- `vm102_net_config.txt` shows `net0: virtio=...,bridge=vmbr2`.
- VM 102 boots successfully on the new segment.

### Step 5 — Configure IP on isolated VM

```bash
# SSH into VM 102 and configure its IP on the 10.10.20.0/24 network
ssh root@<VM102_CONSOLE_IP> << 'VMEOF'
# Set static IP on the VM's network interface
ip addr add 10.10.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 10.10.20.1

# Confirm
ip addr show eth0
ip route show
VMEOF

# Record VM 102 IP and route config
ssh root@10.10.20.10 "ip addr; ip route" 2>/dev/null \
  | tee evidence/a2/vm102_network_config.txt \
  || echo "Configure VM IP manually and capture: ip addr show; ip route show" \
  | tee evidence/a2/vm102_network_config.txt
```

### Step 6 — Implement nftables inter-segment policy

```bash
# Create nftables policy on the Proxmox host
# Policy: vmbr2 (isolated) cannot reach vmbr0 (external)
# But vmbr0 hosts CAN reach vmbr2 for management

cat | sudo tee /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f
# A2 - Network Segmentation Enforcement

flush ruleset

table inet filter {
    chain forward {
        type filter hook forward priority 0; policy drop;
        
        # Allow established/related connections
        ct state established,related accept
        
        # Allow vmbr0 segment to reach vmbr2 (management access)
        iifname "vmbr0" oifname "vmbr2" accept
        
        # Block vmbr2 from reaching vmbr0 (isolation enforced)
        iifname "vmbr2" oifname "vmbr0" drop
        
        # Allow intra-segment communication (vmbr2 to vmbr2)
        iifname "vmbr2" oifname "vmbr2" accept
        
        # Log inter-segment drops for evidence
        iifname "vmbr2" oifname "vmbr0" log prefix "A2-SEGMENTATION-DROP: " drop
    }
    
    chain input {
        type filter hook input priority 0; policy accept;
    }
    
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
EOF

# Apply nftables rules
sudo nft -f /etc/nftables.conf 2>&1 | tee evidence/a2/nft_apply_output.txt

# Enable nftables on boot
sudo systemctl enable --now nftables.service

# Show applied ruleset
sudo nft list ruleset | tee evidence/a2/nft_ruleset_after.txt
```

Expected:
- `nft_ruleset_after.txt` shows the table and chain configuration.
- `nftables.service` is active.

### Step 7 — Validate segmentation policy

```bash
# Test 1: VM 102 (isolated) cannot reach the internet
echo "=== Test 1: vmbr2 to internet ===" | tee evidence/a2/segmentation_tests.txt
ssh root@10.10.20.10 "ping -c 3 8.8.8.8 2>&1; echo exit=$?" \
  | tee -a evidence/a2/segmentation_tests.txt \
  && echo "TEST RESULT: Check if ping succeeded or failed" | tee -a evidence/a2/segmentation_tests.txt

# Test 2: VM 102 can reach the Proxmox host (management allowed)
echo "=== Test 2: vmbr2 to host gateway ===" | tee -a evidence/a2/segmentation_tests.txt
ssh root@10.10.20.10 "ping -c 3 10.10.20.1 2>&1; echo exit=$?" \
  | tee -a evidence/a2/segmentation_tests.txt

# Test 3: tcpdump on Proxmox to capture inter-segment traffic and drops
echo "=== Test 3: Packet capture during segmentation test ===" | tee -a evidence/a2/segmentation_tests.txt
sudo timeout 20 tcpdump -i vmbr2 -n -c 30 2>&1 | tee evidence/a2/tcpdump_vmbr2.txt &

# Generate traffic from VM 102 to vmbr0 network
ssh root@10.10.20.10 "ping -c 5 192.168.1.1 2>&1" | tee -a evidence/a2/segmentation_tests.txt
wait

# Test 4: Check nftables counters for DROP rule
sudo nft list ruleset -a | grep -A5 "SEGMENTATION-DROP" | tee evidence/a2/nft_drop_counters.txt

# Test 5: Verify dropped packets appear in kernel log
sudo journalctl -b -k --no-pager | grep "A2-SEGMENTATION-DROP" | tee evidence/a2/segmentation_drop_log.txt
echo "Dropped packet count: $(wc -l < evidence/a2/segmentation_drop_log.txt)"
```

Expected:
- Test 1: `ping 8.8.8.8` from VM 102 fails (100% packet loss or connection timeout) — isolation confirmed.
- Test 2: `ping 10.10.20.1` (host gateway) succeeds — management access maintained.
- Test 4/5: Kernel log shows `A2-SEGMENTATION-DROP` entries for the blocked traffic.

## 8) Validation

```bash
# Verify vmbr2 interface is up with correct IP
ip addr show vmbr2 | grep "10.10.20.1" \
  && echo "PASS: vmbr2 configured with correct IP" \
  || echo "FAIL: vmbr2 IP not correct"

# Verify nftables is active
sudo nft list ruleset | grep -q "A2 - Network Segmentation" \
  && echo "PASS: nftables segmentation ruleset active" \
  || echo "FAIL: nftables ruleset not active"

# Confirm drops were logged
[ -s evidence/a2/segmentation_drop_log.txt ] \
  && echo "PASS: Inter-segment drops logged ($(wc -l < evidence/a2/segmentation_drop_log.txt) entries)" \
  || echo "WARN: No drop log entries — generate traffic from isolated segment"

# Final routing table
ip route show | tee evidence/a2/routing_table_after.txt
diff evidence/a2/routing_table_before.txt evidence/a2/routing_table_after.txt \
  | tee evidence/a2/routing_table_diff.txt
```

## 9) Evidence

- **Output files:**
  - `evidence/a2/network_interfaces_before.txt` — pre-change network config
  - `evidence/a2/bridge_links_before.txt` — pre-change bridge layout
  - `evidence/a2/routing_table_before.txt` — pre-change routes
  - `evidence/a2/nft_ruleset_before.txt` — pre-change firewall rules
  - `evidence/a2/vm_network_assignments.txt` — VM-to-bridge mapping
  - `evidence/a2/network_interfaces_hash_before.txt` — config file hash
  - `evidence/a2/vm_snapshots_created.txt` — snapshot confirmation
  - `evidence/a2/ifreload_output.txt` — bridge creation output
  - `evidence/a2/vmbr2_interface.txt` — new bridge interface state
  - `evidence/a2/bridge_links_after.txt` — post-change bridge layout
  - `evidence/a2/vm_net_reconfigure.txt` — VM bridge reassignment
  - `evidence/a2/vm102_net_config.txt` — VM 102 Proxmox config
  - `evidence/a2/vm102_network_config.txt` — VM 102 in-guest network
  - `evidence/a2/nft_apply_output.txt` — ruleset apply output
  - `evidence/a2/nft_ruleset_after.txt` — applied firewall ruleset
  - `evidence/a2/segmentation_tests.txt` — all validation test results
  - `evidence/a2/tcpdump_vmbr2.txt` — packet capture on isolated bridge
  - `evidence/a2/nft_drop_counters.txt` — nftables drop counters
  - `evidence/a2/segmentation_drop_log.txt` — kernel log drop entries
  - `evidence/a2/routing_table_after.txt` — post-change routes
  - `evidence/a2/routing_table_diff.txt` — route table diff
- **Hash manifest:**

```bash
find evidence/a2 -type f ! -name "SHA256SUMS.txt" -print0 \
  | xargs -0 sha256sum | sort > evidence/a2/SHA256SUMS.txt
cat evidence/a2/SHA256SUMS.txt
```

## 10) Failure Modes & Recovery

- **Symptom:** `ifreload -a` breaks connectivity to the Proxmox web UI.
  - **Cause:** The `/etc/network/interfaces` edit introduced a syntax error or conflicting IP.
  - **Fix (console access required):** `sudo cp /etc/network/interfaces.bak-a2 /etc/network/interfaces && sudo ifreload -a`
  - **Rollback trigger:** Immediate — restore backup config and reload.

- **Symptom:** VM 102 has no IP after moving to vmbr2.
  - **Cause:** DHCP is not running on the vmbr2 segment (no DHCP server configured there) and the VM has no static IP set.
  - **Fix:** Configure a static IP inside the VM via console access: `ip addr add 10.10.20.10/24 dev eth0`.
  - **Rollback trigger:** `qm rollback 102 pre-a2-segmentation` if VM cannot be accessed.

- **Symptom:** nftables forward chain drops all traffic including management.
  - **Cause:** The `policy drop` on the forward chain drops all traffic including allowed rules if rules are evaluated after.
  - **Fix:** Ensure the `accept` rules come before the `drop` rules. Check rule order with `sudo nft list ruleset`.
  - **Rollback trigger:** `sudo nft flush ruleset` to clear all rules immediately (allows all forwarding temporarily).

## 11) Sources

- [iproute2 documentation — ip route, ip addr](https://man7.org/linux/man-pages/man8/ip.8.html)
- [Arch Wiki — Network configuration](https://wiki.archlinux.org/title/Network_configuration)
- [Proxmox VE Administration Guide — Network Configuration](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_network_configuration)
- [nftables wiki — Getting started](https://wiki.nftables.org/wiki-nftables/index.php/Getting_started_with_nftables)
- [Linux Bridge documentation](https://wiki.linuxfoundation.org/networking/bridge)
- [NIST SP 800-41r1 — Guidelines on Firewalls and Firewall Policy](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-41r1.pdf)

## 12) Stretch Goals

- Add a VLAN-aware bridge and configure VLAN tagging (802.1Q) between the Proxmox host and a VLAN-capable switch. Assign VMs to specific VLANs using the Proxmox `tag=` option in VM network config.
- Deploy a pfSense or OPNsense VM as a virtual router between segments. Route specific ports (e.g., HTTP/HTTPS only) from the isolated segment to the internet through the router VM.
- Implement Proxmox SDN (Software Defined Networking) with VXLAN to segment VMs using overlay networking rather than bridge-level isolation.
- Add IPFIX/NetFlow export from the Proxmox host to capture per-flow statistics for all inter-segment traffic. Feed the flow data into a Grafana dashboard.
