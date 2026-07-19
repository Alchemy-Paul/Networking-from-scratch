# networking-from-scratch

> Hands-on networking lab built with KVM and Linux. No GUIs, no shortcuts — just real config, real errors, and real fixes documented step by step.

Built on **Pop!_OS** using **KVM/virt-manager** and **Ubuntu Server 22.04** VMs. The goal is to go from zero networking knowledge to being able to build, segment, secure, and monitor a small network — entirely from the command line.

---

## Stack

| Tool | Purpose |
|---|---|
| KVM / QEMU | Hypervisor |
| virt-manager | VM management |
| Ubuntu Server 22.04 | Guest OS on node VMs |
| OPNsense 26.1.6 | Firewall / Router |
| netplan | Network configuration |
| tshark | Packet capture and analysis |
| virsh | CLI VM management |

---

## Lab Topology (current)

```
host (Pop!_OS)
      │
      │ SSH tunnel (localhost:8443 → 192.168.50.2:443)
      │
   OPNsense
   ├── WAN (vtnet1) → 192.168.122.134/24 (simulated internet via NAT)
   ├── LAN (vtnet0) → 192.168.50.2/24
   └── OPT1 (vtnet2) → 192.168.51.2/24
          │                      │
   labnet (192.168.50.0/24)    labnet2 (192.168.51.0/24)
   └── node-A  192.168.50.10   └── node-B  192.168.51.10
```

- **labnet** (`virbr1`) — isolated virtual network, no DHCP, no internet
- **labnet2** (`virbr2`) — isolated virtual network, no DHCP, no internet
- **OPNsense** — firewall + router, all inter-subnet traffic passes through it
- Linux router VM still exists but is no longer in the active traffic path

---

## Progress

### ✅ Project 1 — Basic VM Lab
- Created two isolated virtual networks (`labnet`, `labnet2`) in virt-manager
- Installed Ubuntu Server 22.04 on `node-A`, cloned to `node-B`
- Configured static IPs manually via netplan
- Verified connectivity with `ping` and `ssh` between the two nodes

**Bugs hit and fixed:**
- cloud-init silently reverting static IP on every reboot — fixed by creating `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`
- Cloned VM sharing hostname, machine-id, and SSH host keys with the original — regenerated all three
- "Destination host unreachable" traced back to VM being powered off (ARP-level failure)
- NIC link state toggled off at the hypervisor level — invisible from inside the guest OS
- Netplan YAML rejected for inconsistent indentation (tabs vs spaces mixed by nano)

---

### ✅ Project 2 — Linux Router
- Created `router` VM with two NICs — one on each subnet
- Configured `enp1s0` as `192.168.50.254/24` (gateway for labnet)
- Configured `enp7s0` as `192.168.51.254/24` (gateway for labnet2)
- Enabled IP forwarding: `net.ipv4.ip_forward=1` via `/etc/sysctl.d/99-ipforward.conf`
- Moved `node-B` from `labnet` to `labnet2`, updated its IP and default gateway
- Added persistent static routes on `node-A` via netplan
- Verified cross-subnet traffic with `ping` and `ssh` from node-A to node-B
- Confirmed with `tcpdump` on both router NICs that packets traverse the router

**Bugs hit and fixed:**
- Host bridge interfaces (`virbr1`/`virbr2`) owned `192.168.50.1` and `192.168.51.1` — same IPs originally assigned to the router VM. Host intercepted packets and returned "Destination Port Unreachable." Fixed by using `.254` for router addresses
- Conflicting default routes on node-B caused replies to go out the wrong interface
- Temporary NAT NIC needed on each VM for SSH/paste access during config

---

### ✅ Project 3 — Packet Capture and Traffic Analysis
- Installed tshark on node-A and router
- Captured and analyzed ICMP (ping) traffic — TTL, sequence numbers, request/reply pairs
- Captured ARP resolution — broadcast MAC, who asks, who answers, timing relative to ping
- Captured TCP three-way handshake — SYN, SYN-ACK, ACK before SSH encryption kicks in
- Captured cross-subnet forwarding on both router interfaces simultaneously — TTL decrement proves routing

**Bugs hit and fixed:**
- tshark feedback loop over SSH (CPU spike, 80,000+ packets) — fixed with `-f` capture filter
- Route disappearing after reboot — cloud-init disable file had gone missing, recreated it

---

### ✅ Project 4 — Firewall with OPNsense
- Installed OPNsense 26.1.6 as a VM with three NICs (WAN, LAN, OPT1)
- Configured LAN on labnet (`192.168.50.2`) and OPT1 on labnet2 (`192.168.51.2`)
- Routed all inter-subnet traffic through OPNsense, retiring the Linux router
- Accessed web GUI via SSH tunnel through node-A (`ssh -L 8443:192.168.50.2:443 node@192.168.50.10`)
- Wrote and tested three custom firewall rules with logging enabled:
  - Block node-B ICMP → node-A (OPT1 interface)
  - Allow node-A ICMP → node-B (LAN interface)
  - Block node-B SSH → node-A (OPT1 interface, port 22)
- Verified all rules in Firewall → Log Files → Live View in real time
- Proved direction matters — same two hosts, opposite directions, different outcomes

**Bugs hit and fixed:**
- NICs assigned backwards by OPNsense — WAN ended up on labnet, LAN on NAT. Fixed by swapping network sources in virt-manager hardware details
- Host bridge (`virbr1`) owns `192.168.50.1` — GUI inaccessible at that IP. Fixed by using `192.168.50.2` for OPNsense LAN
- GUI inaccessible from WAN side even with firewall rule — OPNsense web server only listens on LAN. Fixed with SSH tunnel
- Block rule on LAN had no effect on node-B traffic — node-B enters on OPT1, not LAN. Moved rule to correct interface
- Traffic bypassing OPNsense entirely — node routes still pointed to Linux router. Updated netplan on both nodes

---

### 📋 Upcoming Projects
- **Project 5** — WireGuard VPN tunnel between VMs
- **Project 6** — VLANs and network segmentation
- **Project 7** — IDS with Suricata
- **Project 8** — Centralized logging with ELK stack or Security Onion

---

## Key Lessons So Far

**cloud-init owns the network config by default.** On Ubuntu Server, cloud-init overwrites your netplan file on every boot unless you explicitly tell it not to. Fix: `network: {config: disabled}` in `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`.

**Cloning a VM copies more than the disk.** hostname, machine-id, and SSH host keys all come along. Always regenerate all three after cloning.

**"Destination host unreachable" is ARP-level.** The packet never left the local network — the OS couldn't resolve the MAC address of the target. Usually means the target is offline or on the wrong network.

**Subnets can't talk without a router.** Moving a device to a different /24 and watching pings immediately fail is the fastest way to make subnetting concrete.

**The host bridge owns .1.** libvirt assigns itself the `.1` address on every virtual bridge it creates. If your firewall or router VM uses `.1`, the host intercepts packets destined for it — use `.2` or `.254` instead.

**Two default routes break routing.** When a VM has two NICs on different networks, Linux may install two default routes. Only one can win — the other silently drops traffic. Always check `ip route show` after adding a NIC.

**tcpdump is the fastest way to find where packets die.** Run it on each interface in the path simultaneously — the first interface where packets stop appearing is where the problem is.

**Firewall rules are per interface, not per network.** A rule on LAN won't affect traffic arriving on OPT1, even if the destination is the same. Always think about which interface traffic enters the firewall on.

**Rule order matters.** Firewalls evaluate rules top to bottom, first match wins. A block rule below an allow-all rule will never fire.

**Two routers on the same network means one is being bypassed.** If you have a Linux router and OPNsense both on labnet, traffic will use whichever one the routes point to — the other's rules are invisible. Always verify the actual traffic path with tcpdump or firewall logs.

---

## Repo Structure

```
/01-basic-vm-lab/       notes and configs for Project 1
/02-router-setup/       notes and configs for Project 2
/03-packet-capture/     notes and configs for Project 3
/04-opnsense-firewall/  notes and configs for Project 4
/configs/               reusable netplan, sysctl, cloud-init snippets
```

---

## Why this exists

Most networking tutorials either use pre-built appliances (pfSense, GNS3 with Cisco images) or skip straight to the interesting security stuff without explaining why the network underneath works. This repo documents building everything from scratch — including every error and why it happened — so the concepts actually stick.