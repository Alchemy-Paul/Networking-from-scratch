# Project 1 — Basic VM Lab

## Goal
Create two isolated virtual networks and two Ubuntu Server VMs with static IPs,
get them communicating via `ping` and `ssh`, and understand why devices on
different subnets cannot talk without a router.

## Final State
| VM | Network | IP | Interface |
|---|---|---|---|
| node-A | labnet | 192.168.50.10/24 | enp1s0 |
| node-B | labnet | 192.168.50.20/24 | enp1s0 |

---

## Steps

### 1. Create the isolated virtual network (labnet)
- In virt-manager: **Edit → Connection Details → Virtual Networks → +**
- Name: `labnet`
- Network: `192.168.50.0/24`
- DHCP: **disabled** (we assign IPs manually)
- Forwarding: **Isolated network** (no NAT, no internet)
- Confirm it shows as **Active** and **Autostart: On Boot**

Verify on host:
```bash
sudo virsh net-list --all
```

### 2. Install Ubuntu Server on node-A
- New VM → Ubuntu Server 22.04 ISO
- 1 vCPU, 1-2GB RAM, 10GB disk
- Network: `Virtual network 'labnet': Isolated network`
- During install, network config will show "autoconfiguration failed" — expected, DHCP is disabled
- Select **Continue without network**
- Install OpenSSH server when prompted
- Skip all optional snaps

### 3. Set static IP on node-A
After install, log in and check the netplan filename:
```bash
ls /etc/netplan/
```

Edit the file (usually `50-cloud-init.yaml`):
```bash
sudo bash -c 'cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.50.10/24]
EOF'
sudo netplan apply
```

Disable cloud-init network management (prevents IP from reverting on reboot):
```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'
```

### 4. Clone node-A to create node-B
- Right-click node-A in virt-manager → **Clone**
- virt-manager assigns a new MAC address automatically

After booting node-B, fix all clone artifacts:

**Fix hostname:**
```bash
sudo hostnamectl set-hostname node-b
sudo nano /etc/hosts   # change 127.0.1.1 node-a → 127.0.1.1 node-b
```

**Fix IP address:**
```bash
sudo bash -c 'cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.50.20/24]
EOF'
sudo netplan apply
```

**Regenerate machine-id:**
```bash
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
```

**Regenerate SSH host keys:**
```bash
sudo rm /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

**Reboot:**
```bash
sudo reboot
```

### 5. Test connectivity
From node-A:
```bash
ping -c 4 192.168.50.20
ssh node@192.168.50.20
```

### 6. Break it on purpose (subnet lesson)
Change node-B's IP to a different subnet:
```bash
# Temporarily change to 192.168.51.20/24
sudo ip addr add 192.168.51.20/24 dev enp1s0
sudo ip addr del 192.168.50.20/24 dev enp1s0
```
Then try pinging from node-A — it fails immediately.
This proves devices on different subnets can't communicate without a router.
Revert when done.

---

## Bugs Hit and Fixed

### cloud-init reverting static IP on every reboot
**Symptom:** IP reverts to DHCP/unconfigured after reboot despite netplan edit.
**Cause:** cloud-init runs early at boot and overwrites `/etc/netplan/50-cloud-init.yaml`
with its cached install-time config.
**Fix:**
```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'
```

### YAML indentation error in netplan
**Symptom:** `sudo netplan apply` returns `Error in network definition: unknown key`
**Cause:** nano mixed tabs and spaces invisibly — YAML only allows spaces.
**Fix:** Use heredoc to write the file instead of nano:
```bash
sudo bash -c 'cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.50.10/24]
EOF'
```
Check for hidden tabs:
```bash
cat -A /etc/netplan/50-cloud-init.yaml | grep -P '\t'
```

### Cloned VM sharing identity with original
**Symptom:** node-B shows hostname `node-a`, same machine-id, same SSH fingerprint.
**Cause:** VM clone copies everything including `/etc/hostname`, `/etc/machine-id`,
and `/etc/ssh/ssh_host_*` keys.
**Fix:** Regenerate hostname, machine-id, and SSH host keys (see steps above).

### NIC in DOWN state with no IP (qdisc noop)
**Symptom:** `ip addr show` shows interface as DOWN with no IP despite netplan config.
**Cause:** netplan failed to parse the config file (YAML error) so it applied nothing,
leaving the interface unconfigured.
**Fix:** Fix the YAML syntax error, then:
```bash
sudo ip link set enp1s0 up
sudo netplan apply
```

### "Destination host unreachable" on ping
**Symptom:** ping returns "Destination host unreachable" from own IP.
**Cause:** ARP failed — couldn't resolve the target's MAC address. Usually means
the target VM is powered off or on a different virtual network.
**Fix:** Confirm target VM is running and both VMs are on the same virtual network
in virt-manager hardware details.

---

## Key Commands Reference
```bash
ip addr show                          # show all interfaces and IPs
ip addr show enp1s0                   # show specific interface
ip link show enp1s0                   # show link state (UP/DOWN)
ip route show                         # show routing table
ip neigh show                         # show ARP table
sudo netplan apply                    # apply netplan config
sudo netplan try                      # apply with auto-rollback on error
hostnamectl                           # show/set hostname
ping -c 4 <ip>                        # send 4 ICMP echo requests
traceroute <ip>                       # trace route to target
sudo systemctl status ssh             # check SSH server status
sudo ufw status                       # check firewall status
```