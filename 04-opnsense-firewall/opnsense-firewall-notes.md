# Project 4 — Firewall with OPNsense

## Goal
Add a real firewall to the lab, route all inter-subnet traffic through it,
write custom rules to allow and block specific traffic, and verify rules work
using live logs. Understand stateful inspection, rule order, interface
assignment, and default deny behavior.

## Final Topology
```
host (Pop!_OS)
      │
      │ SSH tunnel (localhost:8443 → 192.168.50.2:443)
      │
   OPNsense 26.1.6
   ├── WAN (vtnet1) → 192.168.122.134/24 (default NAT, simulated internet)
   ├── LAN (vtnet0) → 192.168.50.2/24    (labnet)
   └── OPT1 (vtnet2) → 192.168.51.2/24  (labnet2)
          │                    │
   labnet (192.168.50.0/24)   labnet2 (192.168.51.0/24)
   └── node-A 192.168.50.10   └── node-B 192.168.51.10
```

All traffic between node-A and node-B passes through OPNsense.
The Linux router VM is no longer in the traffic path.

---

## Why OPNsense over pfSense
- Fully open source, no feature restrictions
- More actively maintained (weekly updates)
- Better built-in IDS/IPS integration (Suricata native)
- Modern UI
- Skills transfer 100% between the two

---

## Setup Steps

### 1. Download and install OPNsense
- Download AMD64 DVD ISO from https://opnsense.org/download/
- Extract: `bunzip2 OPNsense-*.iso.bz2`
- Create VM: 2 vCPU, 2GB RAM, 16GB disk
- Attach two NICs during creation: `default` (NAT) and `labnet`
- Boot ISO, log in as `installer` / `opnsense` to start installation
- Install to `vtbd0` (16GB disk), not `cd0`
- After install, disconnect ISO (Hardware Details → SATA CDROM → No media)
- Boot from disk, log in as `root` / `opnsense`

### 2. Fix NIC assignment (NICs were swapped)
OPNsense assigned WAN to labnet and LAN to NAT — backwards. Fixed by:
- Shutting down OPNsense
- Swapping network sources in virt-manager hardware details
- NIC :2b:20:cf → labnet (LAN)
- NIC :69:f8:01 → default/NAT (WAN)

### 3. Set LAN IP
Console option 2 → LAN → static → 192.168.50.2/24 → no gateway → no DHCP

**Why not 192.168.50.1?**
libvirt assigns itself .1 on every virtual bridge (virbr1 owns 192.168.50.1).
The host intercepts packets to .1 before they reach the VM. Use .2 or .254.

### 4. Access the web GUI via SSH tunnel
The OPNsense web GUI only listens on the LAN interface by default. Because the host cannot reach the LAN subnet directly when libvirt's bridge owns the .1 address, use an SSH tunnel through node-A:

```bash
# Run on host machine (not inside any VM)
ssh -L 8443:192.168.50.2:443 node@192.168.50.10
```

Then open: https://localhost:8443

This must be rerun every session — the tunnel does not persist.

### 5. Add OPT1 interface for labnet2
- Shut down OPNsense
- Add Hardware → Network → labnet2 in virt-manager
- Boot back up
- Console option 1 → assign interfaces:
  - WAN → vtnet1
  - LAN → vtnet0
  - OPT1 → vtnet2
- Console option 2 → OPT1 → static → 192.168.51.2/24 → no gateway → no DHCP

### 6. Update node routes to go through OPNsense

**node-A netplan** (/etc/netplan/50-cloud-init.yaml):
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.50.10/24]
      routes:
        - to: 192.168.51.0/24
          via: 192.168.50.2
```

**node-B netplan** (/etc/netplan/50-cloud-init.yaml):
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.51.10/24]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.51.2
```

Apply on both: sudo netplan apply

---

## Firewall Rules

### How OPNsense rules work
- Rules are evaluated top to bottom; the first match wins
- Everything not explicitly matched is blocked by default
- Rules are applied per interface — traffic from labnet2 hits OPT1 rules, not LAN rules
- Direction matters — the same two hosts can have different outcomes depending on which way traffic flows

### Rule 1: Block node-B ping to node-A
Interface: OPT1 | Action: Block | Protocol: ICMP
Source: 192.168.51.10 | Destination: 192.168.50.10 | Log: enabled
Description: Block node-B ping to node-A

Test:
```bash
# From node-B — should fail
ping -c 4 192.168.50.10
# 4 packets transmitted, 0 received, 100% packet loss
```

### Rule 2: Allow node-A ping to node-B
Interface: LAN | Action: Pass | Protocol: ICMP
Source: 192.168.50.10 | Destination: 192.168.51.10 | Log: enabled
Description: Allow node-A ping to node-B

Test:
```bash
# From node-A — should succeed
ping -c 4 192.168.51.10
# 4 packets transmitted, 4 received, 0% packet loss
```

### Rule 3: Block node-B SSH to node-A
Interface: OPT1 | Action: Block | Protocol: TCP
Source: 192.168.51.10 | Destination: 192.168.50.10
Destination port: SSH (22) | Log: enabled
Description: Block node-B SSH to node-A

Test:
```bash
# From node-B — should timeout
ssh node@192.168.50.10

# From node-A — should succeed
ssh node@192.168.51.10
```

### Viewing blocked traffic
Firewall → Log Files → Live View shows every packet hitting a rule in real time.
Enable Auto-refresh to watch traffic as it happens.

---

## Key Concepts

### Stateful inspection
OPNsense tracks connection state, not just individual packets. When node-A
initiates a connection to node-B, the firewall remembers the session and
automatically allows reply packets back without needing a separate rule.

### Interface assignment matters
Traffic from node-B (on labnet2) arrives on OPT1.
Writing a block rule on LAN will not affect it.
Rules must match the interface where traffic enters the firewall.

### Default deny
Everything not explicitly allowed is blocked. This is the correct
security posture — allowlist, not blocklist.

### Linux router vs OPNsense
| Feature | Linux Router | OPNsense |
|---|---|---|
| Routing | yes | yes |
| Firewall rules | manual iptables | GUI + stateful |
| Logging | manual | built-in live view |
| IDS/IPS | manual | Suricata built-in |
| VPN | manual | WireGuard/OpenVPN built-in |
| Web GUI | no | yes |

---

## Bugs Encountered and Fixed

### NIC assignment backwards
Symptom: WAN had no IP, LAN was on NAT network
Cause: virt-manager presents NICs in a different order than OPNsense assigns them.
Fix: Shut down VM, swap network sources in virt-manager, reassign in console.

### Host bridge intercepting LAN IP
Symptom: Browser shows Connection refused on 192.168.50.1
Cause: libvirt virbr1 bridge owns 192.168.50.1 on the host.
Fix: Set OPNsense LAN IP to 192.168.50.2 instead.

### GUI inaccessible after firewall enables
Symptom: WAN IP times out after pfctl -e
Cause: OPNsense blocks GUI access from WAN by default.
Fix: Use SSH tunnel through node-A.

### Block rule not matching (wrong interface)
Symptom: Rule on LAN did not block node-B traffic
Cause: node-B traffic arrives on OPT1, not LAN.
Fix: Move the rule to OPT1 interface.

### Traffic bypassing OPNsense
Symptom: Firewall rules had no effect on inter-subnet traffic
Cause: node-A route still pointed to Linux router (192.168.50.254).
Fix: Update routes on both nodes to use OPNsense as gateway.

---

## Key Commands Reference
```bash
# SSH tunnel to access OPNsense GUI (run on host)
ssh -L 8443:192.168.50.2:443 node@192.168.50.10
# Then open: https://localhost:8443

# Temporarily disable firewall (emergency access)
pfctl -d

# Re-enable firewall
pfctl -e

# Check interface IPs (OPNsense shell)
ifconfig vtnet0
ifconfig vtnet1
ifconfig vtnet2

# Check what is listening on port 443
sockstat -l | grep 443

# Test connectivity from node-A
ping -c 4 192.168.51.10
ssh node@192.168.51.10
curl -k https://192.168.50.2
```