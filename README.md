# networking-from-scratch

> Hands-on networking lab built with KVM and Linux. No GUIs, no shortcuts — just real config, real errors, and real fixes documented step by step.

Built on **Pop!_OS** using **KVM/virt-manager** and **Ubuntu Server 22.04** VMs. The goal is to go from zero networking knowledge to being able to build, segment, secure, and monitor a small network — entirely from the command line.

---

## Stack

| Tool | Purpose |
|---|---|
| KVM / QEMU | Hypervisor |
| virt-manager | VM management |
| Ubuntu Server 22.04 | Guest OS on all VMs |
| netplan | Network configuration |
| virsh | CLI VM management |

---

## Lab Topology (current)

```
                        host — Pop!_OS
                              │
                           router
                          /       \
               enp1s0              enp7s0
          192.168.50.254          192.168.51.254
                │                       │
    labnet (192.168.50.0/24)    labnet2 (192.168.51.0/24)
    └── node-A  192.168.50.10   └── node-B  192.168.51.10
```

- **labnet** (`virbr1`) — isolated virtual network, no DHCP, no internet
- **labnet2** (`virbr2`) — isolated virtual network, no DHCP, no internet
- **router** — Ubuntu Server VM with two NICs, IP forwarding enabled, no NAT
- Traffic between node-A and node-B flows exclusively through the router

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
- Host bridge interfaces (`virbr1`/`virbr2`) owned `192.168.50.1` and `192.168.51.1` — same IPs originally assigned to the router VM. The host was intercepting packets before they reached the router and returning "Destination Port Unreachable." Fixed by using `.254` for router addresses instead of `.1`
- Conflicting default routes on node-B caused replies to go out the wrong interface — removed the extra default route pointing to the NAT NIC
- `domifaddr` returning blank — qemu-guest-agent not installed, used `ip addr show` inside the VM instead
- Temporary NAT NIC needed on each VM for SSH/paste access during config — removed after setup was complete

---

### 📋 Upcoming Projects
- **Project 3** — Wireshark packet capture: analyze DNS, TCP handshake, and ICMP traffic
- **Project 4** — pfSense/OPNsense firewall with custom rules
- **Project 5** — WireGuard VPN tunnel between VMs
- **Project 6** — VLANs and network segmentation
- **Project 7** — IDS with Snort or Suricata
- **Project 8** — Centralized logging with ELK stack or Security Onion

---

## Key Lessons So Far

**cloud-init owns the network config by default.** On Ubuntu Server, cloud-init overwrites your netplan file on every boot unless you explicitly tell it not to. Fix: `network: {config: disabled}` in `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`.

**Cloning a VM copies more than the disk.** hostname, machine-id, and SSH host keys all come along. Always regenerate all three after cloning.

**"Destination host unreachable" is ARP-level.** The packet never left the local network — the OS couldn't resolve the MAC address of the target. Usually means the target is offline or on the wrong network.

**Subnets can't talk without a router.** Moving a device to a different /24 and watching pings immediately fail is the fastest way to make subnetting concrete.

**The host bridge owns .1.** libvirt assigns itself the `.1` address on every virtual bridge it creates. If your router VM uses `.1`, the host intercepts packets destined for it — use `.254` or any other address instead.

**Two default routes break routing.** When a VM has two NICs on different networks, Linux may install two default routes. Only one can win — the other silently drops traffic going the wrong way. Always check `ip route show` after adding a NIC.

**tcpdump is the fastest way to find where packets die.** Run it on each interface in the path simultaneously — the first interface where packets stop appearing is where the problem is.

**virsh console gives you paste support.** The virt-manager graphical console doesn't support clipboard paste for server VMs. Use `sudo virsh -c qemu:///system console <vm>` from your host terminal, or add a temporary NAT NIC for SSH access.

---

## Repo Structure

```
/01-basic-vm-lab/       notes and configs for Project 1
/02-router-setup/       notes and configs for Project 2
/configs/               reusable netplan, sysctl, cloud-init snippets
```

---

## Why this exists

Most networking tutorials either use pre-built appliances (pfSense, GNS3 with Cisco images) or skip straight to the interesting security stuff without explaining why the network underneath works. This repo documents building everything from scratch — including every error and why it happened — so the concepts actually stick.
