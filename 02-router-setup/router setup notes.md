# Project 2 — Linux Router Setup

## Goal
Connect two isolated subnets (`labnet` and `labnet2`) using a Linux VM as a router.
Traffic between `node-A` and `node-B` must pass through the router — no direct path
exists between the subnets. Verify with `tcpdump` that packets actually traverse
both router interfaces.

## Final State
| VM | Network | IP | Interface |
|---|---|---|---|
| node-A | labnet | 192.168.50.10/24 | enp1s0 |
| node-B | labnet2 | 192.168.51.10/24 | enp1s0 |
| router | labnet | 192.168.50.254/24 | enp1s0 |
| router | labnet2 | 192.168.51.254/24 | enp7s0 |

Traffic path:
```
node-A → router (enp1s0 → enp7s0) → node-B
```

---

## Steps

### 1. Create labnet2 virtual network
- Edit → Connection Details → Virtual Networks → +
- Name: `labnet2`
- Network: `192.168.51.0/24`
- DHCP: disabled
- Forwarding: Isolated network
- Confirm Active and Autostart: On Boot

### 2. Create router VM
- New VM from Ubuntu Server ISO
- 1 vCPU, 1GB RAM, 10GB disk
- Attach to `labnet` during install
- Set static IP `192.168.50.254/24` during installer network config step
- Install OpenSSH server
- Skip all optional snaps

Disable cloud-init network management:
```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'
```

### 3. Add second NIC to router
- Shut down router completely (not just close console)
- Hardware Details → Add Hardware → Network → `labnet2`
- Boot back up, confirm second interface appears:
```bash
ip addr show
```
Should see `enp1s0` (labnet) and a new interface e.g. `enp7s0` (labnet2, no IP yet).

### 4. Configure both router interfaces
```bash
sudo bash -c 'cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.50.254/24]
    enp7s0:
      addresses: [192.168.51.254/24]
EOF'
sudo netplan apply
```

Verify:
```bash
ip addr show enp1s0    # should show 192.168.50.254/24
ip addr show enp7s0    # should show 192.168.51.254/24
```

### 5. Enable IP forwarding
```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl -p /etc/sysctl.d/99-ipforward.conf
cat /proc/sys/net/ipv4/ip_forward    # should print 1
```

### 6. Move node-B to labnet2
- Shut down node-B
- Hardware Details → NIC → change Network source to `labnet2`
- Boot back up, update netplan:
```bash
sudo bash -c 'cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.51.10/24]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.51.254
EOF'
sudo netplan apply
```

### 7. Add persistent static route on node-A
```bash
sudo bash -c 'cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.50.10/24]
      routes:
        - to: 192.168.51.0/24
          via: 192.168.50.254
EOF'
sudo netplan apply
```

### 8. Verify end-to-end with ping and tcpdump
From node-A:
```bash
ping -c 4 192.168.51.10
ssh node@192.168.51.10
```

On router (two terminals simultaneously):
```bash
# Terminal 1 — watch packets arrive from node-A
sudo tcpdump -i enp1s0 icmp

# Terminal 2 — watch packets leave toward node-B
sudo tcpdump -i enp7s0 icmp
```

Expected tcpdump output on both interfaces:
```
192.168.50.10 > 192.168.51.10: ICMP echo request
192.168.51.10 > 192.168.50.10: ICMP echo reply
```

---

## Bugs Hit and Fixed

### Host bridge IP conflict (main bug)
**Symptom:** ping from node-A returns "Destination Port Unreachable" from `192.168.50.1`
even though the router VM is running and has IP forwarding enabled.
**Cause:** libvirt assigns itself the `.1` address on every virtual bridge it creates.
`virbr1` owns `192.168.50.1` and `virbr2` owns `192.168.51.1`. When the router VM
was also configured with these IPs, the host intercepted packets before they reached
the VM and returned "Destination Port Unreachable" because the host had no route to
the other subnet.
**Diagnosis:**
```bash
ip addr show virbr1    # shows 192.168.50.1/24 on host
ip addr show virbr2    # shows 192.168.51.1/24 on host
```
**Fix:** Use `.254` for router interfaces instead of `.1`:
- `enp1s0` → `192.168.50.254/24`
- `enp7s0` → `192.168.51.254/24`

### Conflicting default routes on node-B
**Symptom:** Router can reach node-B but cross-subnet ping from node-A still fails.
**Cause:** node-B had two default routes — one via the router (`192.168.51.254`)
and one via the temporary NAT NIC (`192.168.122.1`). Linux picked the wrong one
for replies, so responses went out the wrong interface and disappeared.
**Diagnosis:**
```bash
ip route show    # look for two "default via" lines
```
**Fix:**
```bash
sudo ip route del default via 192.168.122.1 dev enp7s0
```
Long term: remove the temporary NAT NIC entirely.

### Second NIC not appearing after adding in virt-manager
**Symptom:** `ip addr show` only shows one interface after adding a second NIC.
**Cause:** VM was not fully shut down before adding hardware — virt-manager
requires the VM to be completely off (not just console closed) to add NICs.
**Fix:** Use `sudo shutdown now` inside the VM, wait for it to show as Shutoff
in virt-manager's main window, then add the NIC and boot.

### Temporary NAT NIC losing DHCP lease on reboot
**Symptom:** After reboot, the NAT NIC (`enp8s0`) has no IP.
**Cause:** DHCP lease was grabbed manually with `dhclient` — not managed by netplan,
so it doesn't survive reboots.
**Fix (temporary):** Run `sudo dhclient enp8s0` each time.
**Fix (permanent):** Add to netplan with `dhcp4: true` or remove the NIC entirely.

### `virsh domifaddr` returning blank
**Symptom:** `sudo virsh -c qemu:///system domifaddr Router` returns empty table.
**Cause:** qemu-guest-agent is not installed inside the VM — virsh can't query
guest IPs without it.
**Workaround:** Open VM console and run `ip addr show` directly.

---

## Key Commands Reference
```bash
# Router
ip addr show                              # show all interfaces
ip route show                             # show routing table
cat /proc/sys/net/ipv4/ip_forward         # check if forwarding is on (1=yes)
sudo sysctl -p /etc/sysctl.d/99-ipforward.conf  # apply forwarding config
sudo tcpdump -i enp1s0 icmp              # capture ICMP on interface
sudo tcpdump -i enp7s0 icmp              # capture ICMP on second interface
sudo iptables -L FORWARD                  # check iptables forward chain
sudo nft list ruleset                     # check nftables rules

# General networking
ip neigh show                             # show ARP table
ip route add 192.168.51.0/24 via 192.168.50.254  # add static route
ip route del default via 192.168.122.1    # remove conflicting default route
sudo dhclient enp8s0                      # manually request DHCP lease
```