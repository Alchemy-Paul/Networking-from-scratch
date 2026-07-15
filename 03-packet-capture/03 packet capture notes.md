# Project 3 — Packet Capture and Traffic Analysis

## Goal
Stop treating the network as a black box. Use `tshark` (Wireshark's CLI engine)
to capture and read real traffic — ICMP, ARP, TCP handshakes, and cross-subnet
forwarding — and understand exactly what happens on the wire when two machines
communicate.

## Tools
| Tool | Purpose |
|---|---|
| tshark | CLI packet capture and analysis (Wireshark engine) |
| ping | Generate ICMP traffic |
| ssh | Generate TCP traffic |

---

## Setup

### Install tshark on node-A and router
Since labnet is isolated (no internet), temporarily add a NAT NIC to each VM if you need package installs or SSH access during setup:
- Shut down VM → Add Hardware → Network → `default` (NAT) → Boot up
- Run `sudo dhclient <new-interface>` to get a DHCP IP
- Install tshark:
```bash
sudo apt update
sudo apt install tshark -y
# Select "Yes" when asked about non-superusers capturing packets
```

Remove the NAT NIC after installation if desired (or keep for SSH access).

### Verify the interface names first
The examples below assume `enp1s0` and `enp7s0`, but interface names can differ on your host or VM image. Confirm them before starting a capture:
```bash
ip -br addr
```
Use the interface that carries the lab subnet for your current VM. On the router, one NIC is usually `enp1s0` for `labnet` and the second is `enp7s0` for `labnet2`.

### Key tshark flags
```bash
-i enp1s0       # which interface to capture on
-n              # don't resolve IPs to hostnames (keeps output clean)
-f "icmp"       # capture filter — only capture matching traffic
                # (applied at kernel level, prevents feedback loops)
```

### Avoiding the feedback loop
Running tshark over SSH without a filter captures the SSH session itself,
creating a feedback loop (tshark output → SSH packets → more tshark output).
Always use `-f` to filter to only the traffic you care about:
```bash
sudo tshark -i enp1s0 -n -f "icmp"           # only ICMP
sudo tshark -i enp1s0 -n -f "arp"            # only ARP
sudo tshark -i enp1s0 -n -f "tcp port 22"    # only SSH
sudo tshark -i enp1s0 -n -f "tcp and host 192.168.50.254"  # only traffic to/from specific host
```

---

## Capture 1: ICMP (ping)

**Setup:** Two terminals SSHed into node-A simultaneously.

**Terminal 1 — start capture:**
```bash
sudo tshark -i enp1s0 -n -f "icmp"
```

**Terminal 2 — generate traffic:**
```bash
ping -c 4 192.168.50.254
```

**Sample output:**
```
1  0.000000  192.168.50.10 → 192.168.50.254  ICMP 98 Echo (ping) request  id=0x0001, seq=1/256, ttl=64
2  0.000189  192.168.50.254 → 192.168.50.10  ICMP 98 Echo (ping) reply    id=0x0001, seq=1/256, ttl=64
3  1.000198  192.168.50.10 → 192.168.50.254  ICMP 98 Echo (ping) request  id=0x0001, seq=2/512, ttl=64
4  1.001066  192.168.50.254 → 192.168.50.10  ICMP 98 Echo (ping) reply    id=0x0001, seq=2/512, ttl=64
```

**Reading the columns:**
| Column | Meaning |
|---|---|
| `1` | Packet number |
| `0.000000` | Timestamp (seconds since capture started) |
| `192.168.50.10 → 192.168.50.254` | Source → destination IP |
| `ICMP` | Protocol |
| `98` | Packet size in bytes |
| `Echo (ping) request` | ICMP type |
| `id=0x0001` | Ping session ID |
| `seq=1/256` | Sequence number (1st ping) |
| `ttl=64` | Time To Live — decrements by 1 at each router hop |

**Key observations:**
- 4 request/reply pairs match `ping -c 4`
- `ttl=64` on both sides — router is directly connected, no TTL decrement
- Sequence numbers increment by 256 each time (`seq=1/256`, `seq=2/512`)

---

## Capture 2: ARP

ARP resolves IP addresses to MAC addresses before any IP communication can happen.

**Setup:** Two terminals SSHed into node-A.

**Terminal 1 — start ARP capture:**
```bash
sudo tshark -i enp1s0 -n -f "arp"
```

**Terminal 2 — flush ARP cache then ping:**
```bash
sudo ip neigh flush dev enp1s0
ping -c 1 192.168.50.254
```

**Sample output:**
```
1  0.000000  52:54:00:5a:4d:78 → ff:ff:ff:ff:ff:ff  ARP 42 Who has 192.168.50.1? Tell 192.168.50.10
2  0.000125  52:54:00:a4:58:e6 → 52:54:00:5a:4d:78  ARP 42 192.168.50.1 is at 52:54:00:a4:58:e6
3  13.081272 52:54:00:5a:4d:78 → ff:ff:ff:ff:ff:ff  ARP 42 Who has 192.168.50.254? Tell 192.168.50.10
4  13.081886 52:54:00:a2:9a:17 → 52:54:00:5a:4d:78  ARP 42 192.168.50.254 is at 52:54:00:a2:9a:17
5  13.081721 52:54:00:a2:9a:17 → ff:ff:ff:ff:ff:ff  ARP 42 Who has 192.168.50.10? Tell 192.168.50.254
6  18.321776 52:54:00:5a:4d:78 → 52:54:00:a2:9a:17  ARP 42 192.168.50.10 is at 52:54:00:5a:4d:78
```

**Reading the packets:**
- **Packets 1-2:** node-A looks for host bridge (`.1`) after cache flush
- **Packets 3-4:** node-A broadcasts "Who has `.254`?" — router replies with its MAC
- **Packets 5-6:** Router also needs node-A's MAC to send the ping reply back

**Key observations:**
- `ff:ff:ff:ff:ff:ff` is the **broadcast MAC** — ARP requests go to every device on the subnet
- Only the device that owns the IP responds — everyone else ignores the broadcast
- ARP always happens before any IP packet leaves the NIC
- Without a cached ARP entry, there's a brief delay before the first ping reply

---

## Capture 3: TCP Three-Way Handshake

Every TCP connection (SSH, HTTP, HTTPS) starts with SYN → SYN-ACK → ACK.

**Setup:** Two terminals SSHed into node-A. Make sure no existing SSH session
to the router is open (exit first) so a fresh connection is created.

**Terminal 1 — capture only traffic to router:**
```bash
sudo tshark -i enp1s0 -n -f "tcp and host 192.168.50.254"
```

**Terminal 2 — SSH into router (creates a new TCP connection):**
```bash
ssh router@192.168.50.254
```

**Sample output (first 9 packets):**
```
1  0.000000  192.168.50.10 → 192.168.50.254  TCP  74  35040 → 22 [SYN] Seq=0 Win=64240 MSS=1460
2  0.000162  192.168.50.254 → 192.168.50.10  TCP  74  22 → 35040 [SYN, ACK] Seq=0 Ack=1 Win=65160
3  0.001759  192.168.50.10 → 192.168.50.254  TCP  66  35040 → 22 [ACK] Seq=1 Ack=1 Win=64256
4  0.005997  192.168.50.10 → 192.168.50.254  SSH  108 Client: Protocol (SSH-2.0-OpenSSH_8.9p1)
5  0.007701  192.168.50.254 → 192.168.50.10  TCP  66  22 → 35040 [ACK]
6  0.068272  192.168.50.254 → 192.168.50.10  SSHv2 108 Server: Protocol (SSH-2.0-OpenSSH_8.9p1)
7  0.068299  192.168.50.10 → 192.168.50.254  TCP  66  [ACK]
8  0.069557  192.168.50.10 → 192.168.50.254  SSHv2 1602 Client: Key Exchange Init
9  0.069775  192.168.50.254 → 192.168.50.10  TCP  66  [ACK]
```

**The three-way handshake (packets 1-3):**
| Packet | Direction | Flag | Meaning |
|---|---|---|---|
| 1 | node-A → router | SYN | "I want to connect, here's my sequence number" |
| 2 | router → node-A | SYN-ACK | "Accepted, here's mine, I acknowledge yours" |
| 3 | node-A → router | ACK | "Acknowledged — connection is open" |

**After the handshake (packets 4-9):**
- Both sides announce SSH version
- Key exchange begins (Diffie-Hellman)
- All subsequent traffic is encrypted ("Encrypted packet")

**Key TCP fields:**
| Field | Meaning |
|---|---|
| `Win=64240` | TCP window size — how much data can be in flight before needing an ACK |
| `MSS=1460` | Maximum Segment Size — largest chunk per packet (standard for Ethernet) |
| `Seq=0` | Starting sequence number |
| `Ack=1` | Acknowledging up to byte 1 from the other side |
| `35040 → 22` | Random source port → SSH well-known port |

---

## Capture 4: Cross-Subnet Forwarding on the Router

The definitive proof that IP forwarding works — the same packet appears on
both router interfaces as it crosses from one subnet to the other.

**Setup:** Three terminals — two SSHed into the router, one into node-A.

**Router Terminal 1 — watch labnet side:**
```bash
sudo tshark -i enp1s0 -n -f "icmp"
```

**Router Terminal 2 — watch labnet2 side:**
```bash
sudo tshark -i enp7s0 -n -f "icmp"
```

**node-A Terminal — ping node-B:**
```bash
ping -c 4 192.168.51.10
```

**enp1s0 output (labnet side):**
```
1  0.000000  192.168.50.10 → 192.168.51.10  ICMP 98 Echo (ping) request  ttl=64
2  0.000069  192.168.51.10 → 192.168.50.10  ICMP 98 Echo (ping) reply    ttl=63
```

**enp7s0 output (labnet2 side):**
```
1  0.000000  192.168.50.10 → 192.168.51.10  ICMP 98 Echo (ping) request  ttl=63
2  0.000022  192.168.51.10 → 192.168.50.10  ICMP 98 Echo (ping) reply    ttl=64
```

**What the TTL decrement proves:**
- Request arrives on `enp1s0` with `ttl=64` → router decrements → leaves `enp7s0` with `ttl=63`
- Reply arrives on `enp7s0` with `ttl=64` → router decrements → leaves `enp1s0` with `ttl=63`

The TTL decrement is the router's fingerprint on every packet it forwards.
This is exactly how `traceroute` works — it sends packets with incrementing TTLs
(1, 2, 3...) and each router that drops an expired packet sends back an ICMP
"Time Exceeded" error, revealing its IP and mapping the full path.

---

## Bugs Hit and Fixed

### tshark feedback loop (CPU spike, 80,000+ packets)
**Symptom:** Running `tshark -i enp1s0` without a filter over SSH causes
packet count to explode and CPU to spike — tshark captures its own SSH output.
**Fix:** Always use `-f` capture filter:
```bash
sudo tshark -i enp1s0 -n -f "icmp"
sudo tshark -i enp1s0 -n -f "tcp and host 192.168.50.254"
```

### Route disappearing after reboot (cloud-init revert)
**Symptom:** `ping: connect: Network is unreachable` after reboot despite
working before.
**Cause:** `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` was missing,
so cloud-init restored the original install-time netplan on every reboot,
overwriting the static route.
**Fix:**
```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'
```
Always verify this file exists on every VM after any reinstall or clone.

---

## Key Commands Reference
```bash
# Install
sudo apt install tshark -y

# Basic capture
sudo tshark -i enp1s0 -n

# Filtered captures
sudo tshark -i enp1s0 -n -f "icmp"
sudo tshark -i enp1s0 -n -f "arp"
sudo tshark -i enp1s0 -n -f "tcp port 22"
sudo tshark -i enp1s0 -n -f "tcp and host 192.168.50.254"

# Flush ARP cache (forces ARP resolution to happen again)
sudo ip neigh flush dev enp1s0

# Check ARP table
ip neigh show

# Verify routing before any capture
ip route show
```