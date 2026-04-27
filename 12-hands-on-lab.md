# 12 — Hands-On Lab: Real Commands from This Machine

Every command you need to explore networking — run live on this machine (10.12.24.141) with real output and line-by-line explanations.

**This file covers:** IP addresses (concept 03), subnets (04), ports (05), data travel (06), DNS (07), firewalls (08), NAT (09), SSL/TLS (10).

---

## Table of Contents

- [About This Machine](#about-this-machine)
- [Lab 1 — Your IP Address and Interfaces](#lab-1-your-ip-address-and-interfaces)
- [Lab 2 — Your Subnet and Neighbors (ARP)](#lab-2-your-subnet-and-neighbors-arp)
- [Lab 3 — Route Table and Default Gateway](#lab-3-route-table-and-default-gateway)
- [Lab 4 — Listening Ports and Active Connections](#lab-4-listening-ports-and-active-connections)
- [Lab 5 — Ping and Connectivity](#lab-5-ping-and-connectivity)
- [Lab 6 — Traceroute: See Every Hop](#lab-6-traceroute-see-every-hop)
- [Lab 7 — DNS Lookups](#lab-7-dns-lookups)
- [Lab 8 — Firewall Rules (iptables)](#lab-8-firewall-rules-iptables)
- [Lab 9 — TLS Handshake and Certificates](#lab-9-tls-handshake-and-certificates)
- [Lab 10 — Full HTTP Request with curl](#lab-10-full-http-request-with-curl)
- [Lab 11 — Port Checking with nc (netcat)](#lab-11-port-checking-with-nc-netcat)
- [Lab 12 — Established Connections: What's Talking Right Now](#lab-12-established-connections-whats-talking-right-now)
- [Quick Reference: All Commands on One Page](#quick-reference-all-commands-on-one-page)

---

## About This Machine

```
Hostname:  ubuntu-VMware-Virtual-Platform
IP:        10.12.24.141/23
MAC:       00:50:56:a6:ab:1b
Interface: ens192 (VMware virtual NIC)
Gateway:   10.12.24.1
DNS:       10.56.56.56, 10.56.56.156 (corporate DNS)
Network:   GE HealthCare corporate network
```

---

## Lab 1 — Your IP Address and Interfaces

**Concept covered:** 03 — IP Addresses

### Command: `ip addr show`

```bash
ip -br addr show
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens192           UP             10.12.24.141/23 fe80::1dc2:ddd5:1556:fef8/64
docker0          UP             172.17.0.1/16 fe80::ccee:dbff:feeb:65d8/64
vetha5e096e@if2  UP             fe80::c022:3fff:fed2:f4b6/64
veth764e743@if2  UP             fe80::1467:d4ff:feca:bb63/64
```

### Reading Every Line

```
lo — UNKNOWN — 127.0.0.1/8
│      │            │
│      │            └── Loopback address (concept 03)
│      │                Always 127.0.0.1. "Talking to myself."
│      │                /8 means 127.0.0.0 – 127.255.255.255
│      └── UNKNOWN = loopback isn't really "up" or "down" — it always exists
└── lo = loopback interface (virtual, not a real network card)

ens192 — UP — 10.12.24.141/23
│          │        │
│          │        └── THIS machine's real IP address
│          │            /23 means subnet: 10.12.24.0 – 10.12.25.255
│          │            That's 512 addresses (concept 04)
│          └── UP = interface is active and connected
└── ens192 = the physical network interface (VMware virtual NIC)
    "en" = ethernet, "s192" = slot/bus number assigned by VMware

docker0 — UP — 172.17.0.1/16
│                    │
│                    └── Docker's bridge network IP
│                        /16 = 172.17.0.0 – 172.17.255.255
│                        Docker containers get IPs in this range
└── docker0 = Docker creates this virtual bridge interface
    Containers on this host talk to each other through this

vetha5e096e@if2 / veth764e743@if2
│
└── "veth" = Virtual Ethernet — Docker creates one per running container
    These are the cable endpoints connecting containers to docker0
    You can ignore these for networking fundamentals
```

### Getting Just Your IP

```bash
ip addr show ens192 | grep "inet "
```

```
    inet 10.12.24.141/23 brd 10.12.25.255 scope global noprefixroute ens192
```

```
Reading this:
  inet 10.12.24.141/23  ← your IP and subnet mask
  brd 10.12.25.255      ← broadcast address (last IP in the subnet)
  scope global           ← reachable from the network (not just localhost)
  noprefixroute          ← OS won't auto-add prefix routes (set statically)
  ens192                 ← which interface
```

### Your MAC Address

```bash
ip addr show ens192 | grep "link/ether"
```

```
    link/ether 00:50:56:a6:ab:1b brd ff:ff:ff:ff:ff:ff
```

```
00:50:56:a6:ab:1b = your MAC address (concept 01)
  00:50:56 = VMware's OUI (vendor prefix — all VMware NICs start with this)
  a6:ab:1b = unique to this virtual NIC

ff:ff:ff:ff:ff:ff = broadcast MAC (send to ALL devices on the LAN)
```

---

## Lab 2 — Your Subnet and Neighbors (ARP)

**Concepts covered:** 03 — IP Addresses, 04 — Subnets, 06 — ARP

### Your Subnet Range

```
Your IP: 10.12.24.141/23

/23 means:
  Network: 10.12.24.0
  First usable: 10.12.24.1
  Last usable: 10.12.25.254
  Broadcast: 10.12.25.255
  Total hosts: 510

Any IP in 10.12.24.0 – 10.12.25.255 is on YOUR LAN.
Anything outside needs to go through the gateway (10.12.24.1).
```

### Command: `ip neigh show` (ARP Table)

```bash
ip neigh show | grep ens192
```

```
10.12.24.174 dev ens192 lladdr 00:50:56:a6:f1:20 STALE
```

### Reading This

```
10.12.24.174   dev ens192   lladdr 00:50:56:a6:f1:20   STALE
│              │             │                           │
│              │             │                           └── STALE: we talked to this
│              │             │                               device before but not recently
│              │             └── Link Layer Address = MAC address
│              │                 00:50:56 = VMware (another VM on the same network)
│              └── which interface we see this neighbor on
└── IP address of the neighbor device
```

```
ARP states:
  REACHABLE — recently confirmed alive (we talked to it in the last few seconds)
  STALE     — known but not confirmed recently (might still be there)
  FAILED    — tried to reach it, got no response (device is gone or down)
  DELAY     — waiting for confirmation

The Docker entries show FAILED because those containers have stopped:
  172.17.0.18 dev docker0 FAILED  ← container doesn't exist anymore
  172.17.0.3 dev docker0 lladdr 7a:17:14:34:c7:bf STALE  ← container exists but idle
```

---

## Lab 3 — Route Table and Default Gateway

**Concepts covered:** 04 — Subnets, 06 — Routing, Default Gateway

### Command: `ip route show`

```bash
ip route show
```

```
default via 10.12.24.1 dev ens192 proto static metric 100
10.12.24.0/23 dev ens192 proto kernel scope link src 10.12.24.141 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

### Reading Every Route

```
Route 1: default via 10.12.24.1 dev ens192 proto static metric 100

  default        = 0.0.0.0/0 — matches EVERYTHING
  via 10.12.24.1 = send to this gateway (the router)
  dev ens192     = out through this interface
  proto static   = manually configured (not learned automatically)
  metric 100     = priority (lower = preferred if multiple defaults exist)

  Translation: "For ANY destination not in the other routes,
                send the packet to the router at 10.12.24.1."

  This is THE most important route. It's your EXIT DOOR to the internet.


Route 2: 10.12.24.0/23 dev ens192 proto kernel scope link src 10.12.24.141

  10.12.24.0/23  = your local subnet (10.12.24.0 – 10.12.25.255)
  dev ens192     = directly connected (no router needed)
  proto kernel   = automatically created by the OS when the IP was assigned
  scope link     = only reachable on this directly-connected link
  src 10.12.24.141 = use this as the source IP when sending

  Translation: "For anything in 10.12.24.0 – 10.12.25.255,
                send DIRECTLY — it's on my LAN, no gateway needed."


Route 3: 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1

  172.17.0.0/16  = Docker's internal network
  dev docker0    = through the Docker bridge
  src 172.17.0.1 = use the Docker bridge IP as source

  Translation: "For anything in 172.17.x.x, send via Docker bridge."
```

### How the Route Table Decides (Longest Prefix Match)

```
Packet to 10.12.24.174 (another VM on your LAN):
  10.12.24.0/23 — matches ✓ (/23)
  default       — matches ✓ (/0)
  Winner: /23 is more specific → send directly on ens192

Packet to 172.17.0.3 (a Docker container):
  172.17.0.0/16 — matches ✓ (/16)
  default       — matches ✓ (/0)
  Winner: /16 is more specific → send via docker0

Packet to 172.253.118.102 (Google):
  10.12.24.0/23 — no match ✗
  172.17.0.0/16 — no match ✗
  default       — matches ✓ (/0)
  Winner: default → send to gateway 10.12.24.1
```

---

## Lab 4 — Listening Ports and Active Connections

**Concept covered:** 05 — Ports & Protocols

### Command: `ss -tlnp` (TCP Listening Ports)

```bash
ss -tlnp
```

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      4096   127.0.0.53%lo:53    0.0.0.0:*         systemd-resolve
LISTEN 0      4096       127.0.0.1:40513 0.0.0.0:*         code-10c8e557c8
LISTEN 0      4096      127.0.0.54:53    0.0.0.0:*         systemd-resolve
LISTEN 0      4096         0.0.0.0:111   0.0.0.0:*         rpcbind
LISTEN 0      4096         0.0.0.0:22    0.0.0.0:*         sshd
LISTEN 0      4096       127.0.0.1:631   0.0.0.0:*         cupsd
```

### Reading Every Line

```
ss -tlnp flags:
  -t = TCP only
  -l = listening (waiting for connections)
  -n = numeric (show port numbers, not names)
  -p = show process name
```

```
LISTEN 0 4096  127.0.0.53%lo:53  0.0.0.0:*  systemd-resolve
│      │  │    │              │   │           │
│      │  │    │              │   │           └── process: DNS resolver
│      │  │    │              │   └── accepts from: any IP, any port
│      │  │    │              └── port 53 (DNS — concept 07!)
│      │  │    └── bound to 127.0.0.53 on loopback
│      │  │        This means ONLY this machine can query it
│      │  │        (127.x.x.x = localhost only)
│      │  └── backlog: max 4096 queued connections
│      └── Recv-Q: 0 pending connections (healthy)
└── LISTEN = waiting for incoming connections

LISTEN 0 4096  0.0.0.0:22  0.0.0.0:*  sshd
                │       │
                │       └── port 22 (SSH — concept 05!)
                └── 0.0.0.0 = listening on ALL interfaces
                    Unlike 127.0.0.1, this accepts connections from
                    ANY network (remote SSH access is enabled)

LISTEN 0 4096  127.0.0.1:631  0.0.0.0:*  cupsd
                │          │
                │          └── port 631 (CUPS — printing service)
                └── 127.0.0.1 = localhost only
                    Nobody from the network can reach the print service
```

### What's Exposed to the Network vs Localhost

```
Exposed to the network (0.0.0.0):
  Port 22  (SSH)  — remote access works ✓
  Port 111 (RPC)  — remote procedure calls

Localhost only (127.0.0.1 or 127.0.0.53):
  Port 53   (DNS resolver)  — only this machine can query
  Port 631  (CUPS printing) — only local printing
  Port 40513 (VS Code)      — VS Code server internal port
```

### Command: `ss -ulnp` (UDP Listening Ports)

```bash
ss -ulnp
```

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
UNCONN 0      0            0.0.0.0:5353  0.0.0.0:*         avahi-daemon
UNCONN 0      0         127.0.0.54:53    0.0.0.0:*         systemd-resolve
UNCONN 0      0      127.0.0.53%lo:53    0.0.0.0:*         systemd-resolve
UNCONN 0      0            0.0.0.0:111   0.0.0.0:*         rpcbind
```

```
UNCONN = UDP socket (UDP is connectionless, so it's "unconnected")

Port 5353 (mDNS — multicast DNS for local network discovery)
Port 53   (DNS — same resolver, but UDP this time — concept 07)
Port 111  (RPC — UDP version)
```

### Connection Summary

```bash
ss -s
```

```
Total: 824
TCP:   50 (estab 10, closed 30, orphaned 0, timewait 21)

Transport Total     IP        IPv6
RAW       1         0         1
UDP       8         5         3
TCP       20        17        3
INET      29        22        7
```

```
Reading this:
  824 total sockets (includes Unix domain sockets, not just network)
  50 TCP sockets:
    10 established — actively exchanging data right now
    30 closed      — recently closed, being cleaned up
    21 timewait    — TCP connections that are closed but waiting
                     (TCP waits ~60 seconds before fully releasing a connection
                      to handle any delayed packets — concept 05 "four-way close")
```

---

## Lab 5 — Ping and Connectivity

**Concept covered:** 06 — How Data Travels

### Command: `ping`

```bash
ping -c 3 google.com
```

```
PING google.com (172.253.118.102) 56(84) bytes of data.
64 bytes from 172.253.118.102: icmp_seq=1 ttl=109 time=80.4 ms
64 bytes from 172.253.118.102: icmp_seq=2 ttl=109 time=80.7 ms
64 bytes from 172.253.118.102: icmp_seq=3 ttl=109 time=87.1 ms

--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 80.440/82.756/87.099/3.073 ms
```

### Reading Every Field

```
PING google.com (172.253.118.102) 56(84) bytes of data.
│                │                 │  │
│                │                 │  └── 84 = 56 data + 8 ICMP header + 20 IP header
│                │                 └── 56 bytes of payload data
│                └── DNS resolved to this IP (concept 07)
└── domain name you pinged

64 bytes from 172.253.118.102: icmp_seq=1 ttl=109 time=80.4 ms
│                               │          │        │
│                               │          │        └── Round-trip time: 80.4 milliseconds
│                               │          │            This machine → Google → back = 80ms
│                               │          │
│                               │          └── TTL=109 (concept 06)
│                               │              Google likely sent with TTL=128
│                               │              128 - 109 = 19 routers between us
│                               │
│                               └── Sequence 1 (first ping)
│                                   seq=1, seq=2, seq=3 — if one is missing = packet loss
│
└── 64 bytes received back

⚠ Sometimes you'll see a DIFFERENT TTL for the same domain:

  ping google.com → 172.253.118.139 → ttl=105  (128 - 105 = 23 hops)
  ping google.com → 172.253.118.102 → ttl=109  (128 - 109 = 19 hops)

  Why? Google has 6 different IPs (we saw them in dig — Lab 7).
  Each IP is a different server, possibly in a different data center.
  Different servers = different network paths = different hop counts.

  172.253.118.102 → 19 hops (shorter path, maybe closer data center)
  172.253.118.139 → 23 hops (longer path, routed through more networks)

  This is NORMAL. DNS round-robin means you might hit a different
  server each time you ping, and each path has a different length.

3 packets transmitted, 3 received, 0% packet loss
│                      │             │
│                      │             └── Perfect. If >0% there's a problem.
│                      └── 3 came back
└── 3 were sent

rtt min/avg/max/mdev = 80.440/82.756/87.099/3.073 ms
│   │    │    │    │
│   │    │    │    └── standard deviation: 3ms (variation between pings)
│   │    │    └── slowest ping: 87ms
│   │    └── average: 82ms
│   └── fastest ping: 80ms
└── rtt = round-trip time
```

### Quick Ping Diagnostic

```bash
# Can I reach the internet at all?
ping -c 1 8.8.8.8

# If YES → network works, DNS might be broken if names don't resolve
# If NO  → network issue: check cable, gateway, firewall
```

---

## Lab 6 — Traceroute: See Every Hop

**Concept covered:** 06 — How Data Travels (routing, hops, TTL)

### Command: `traceroute`

```bash
traceroute -I -n google.com
```

(`-I` = use ICMP like ping, `-n` = show IPs not hostnames)

```
traceroute to google.com (172.253.118.138), 30 hops max, 60 byte packets
 1  10.213.50.5      0.671 ms  0.771 ms  0.888 ms
 2  10.213.200.5     0.313 ms  0.425 ms  0.542 ms
 3  172.16.1.5       0.385 ms  0.404 ms  0.545 ms
 4  * * *
 5  * * *
 ...
23  * * *
24  172.253.118.138  85.269 ms  85.240 ms  85.253 ms
```

### Reading Every Hop

```
YOUR MACHINE: 10.12.24.141
Default gateway: 10.12.24.1 (from route table)

Hop 1: 10.213.50.5  (0.7 ms)
  │
  │  This is NOT 10.12.24.1 (your configured default gateway).
  │  Why? 10.12.24.1 is likely a virtual IP (VRRP/HSRP — a redundancy
  │  protocol where multiple routers share one IP). The actual device
  │  handling your traffic is 10.213.50.5, or 10.12.24.1 forwards
  │  without decrementing TTL and the first device that DOES
  │  decrement TTL and responds is 10.213.50.5.
  │
  │  10.213.x.x = private IP, still inside the corporate network.
  │  0.7ms = very fast, same building/data center.
  │
Hop 2: 10.213.200.5  (0.3 ms)
  │
  │  Another corporate router. Different subnet (10.213.200.x).
  │  Still private IP space. Even faster than hop 1.
  │
Hop 3: 172.16.1.5  (0.4 ms)
  │
  │  172.16.x.x = private IP (172.16.0.0/12 range — concept 03).
  │  This might be the edge of the corporate network or a firewall
  │  that connects to the ISP/internet.
  │
Hops 4–23: * * *
  │
  │  ALL SILENT. Every one of these routers exists and forwards
  │  your packet, but they refuse to send "TTL expired" messages.
  │
  │  This is NORMAL in corporate networks. Reasons:
  │    1. Corporate firewalls block ICMP TTL-expired messages
  │    2. ISP routers configured to not reveal their IPs (security)
  │    3. Some network devices (load balancers, proxies) don't
  │       participate in traceroute at all
  │
  │  The packet IS moving through them — it reaches Google.
  │
Hop 24: 172.253.118.138  (85 ms)
  │
  │  GOOGLE! The final destination.
  │  172.253.x.x = Google's IP range.
  │  85ms round-trip = total latency from you to Google and back.
```

### The Network Path Diagram

```
Your Machine (10.12.24.141)
     │
     │ default via 10.12.24.1
     ▼
Hop 1: 10.213.50.5  ← corporate router/firewall (0.7ms)
     │
     ▼
Hop 2: 10.213.200.5 ← corporate backbone (0.3ms)
     │
     ▼
Hop 3: 172.16.1.5   ← corporate edge / ISP handoff (0.4ms)
     │
     ▼
Hops 4–23: * * *    ← ~20 invisible routers (ISP + internet backbone)
     │                  firewalls hiding them
     ▼
Hop 24: 172.253.118.138 ← Google's server (85ms total)
```

---

## Lab 7 — DNS Lookups

**Concept covered:** 07 — DNS

### Your DNS Configuration

```bash
resolvectl status | head -12
```

```
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub

Link 2 (ens192)
    Current Scopes: DNS
         Protocols: +DefaultRoute
Current DNS Server: 10.56.56.56
       DNS Servers: 10.56.56.56 10.56.56.156
```

```
Reading this:
  resolv.conf mode: stub
    → Your machine uses systemd-resolved as a local DNS proxy
    → /etc/resolv.conf points to 127.0.0.53 (the local stub)
    → The stub forwards queries to the REAL DNS servers below

  Current DNS Server: 10.56.56.56
    → This is the GEHC corporate DNS server
    → 10.56.56.156 is the backup

  Your DNS path:
    Application → 127.0.0.53 (local stub) → 10.56.56.56 (corporate DNS) → answer
```

### Command: `dig` (DNS Lookup)

```bash
dig google.com
```

```
google.com.    273    IN    A    172.253.118.100
google.com.    273    IN    A    172.253.118.101
google.com.    273    IN    A    172.253.118.102
google.com.    273    IN    A    172.253.118.113
google.com.    273    IN    A    172.253.118.138
google.com.    273    IN    A    172.253.118.139

;; Query time: 45 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
```

### Reading Every Field

```
google.com.   273   IN   A   172.253.118.100
│             │     │    │   │
│             │     │    │   └── The IP address (the answer!)
│             │     │    └── A = Address record (IPv4) — concept 07
│             │     └── IN = Internet class (always IN)
│             │
│             └── TTL = 273 seconds (about 4.5 minutes)
│                 "Cache this answer for 273 more seconds.
│                  After that, ask DNS again — IP might change."
│
└── The domain name (trailing dot = FQDN — concept 07)

Why 6 different IPs?
  Google runs thousands of servers. DNS returns multiple IPs
  so your browser can pick one (load balancing).
  If 172.253.118.100 is slow, try .101, etc.

;; Query time: 45 msec
  → DNS lookup took 45ms (not instant — had to ask the corporate DNS server)

;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
  → Query went to 127.0.0.53 port 53 (the local systemd-resolved stub)
  → Over UDP (DNS uses UDP for speed — concept 05)
```

### Quick One-Liner: Just the IPs

```bash
dig +short google.com
```

```
172.253.118.113
172.253.118.100
172.253.118.138
172.253.118.101
172.253.118.139
172.253.118.102
```

### Lookup Mail Servers (MX Record)

```bash
dig google.com MX
```

```
google.com.    300    IN    MX    10 smtp.google.com.
```

```
MX = Mail Exchange record
10 = priority (lower = tried first)
smtp.google.com = the server that handles email for @google.com

This is why when you send email to someone@google.com,
your mail server knows to deliver it to smtp.google.com.
```

### Lookup Name Servers (NS Record)

```bash
dig google.com NS
```

```
google.com.    345600    IN    NS    ns1.google.com.
google.com.    345600    IN    NS    ns2.google.com.
google.com.    345600    IN    NS    ns3.google.com.
google.com.    345600    IN    NS    ns4.google.com.
```

```
NS = Name Server records
These are the AUTHORITATIVE servers for google.com (concept 07).
TTL = 345600 seconds = 4 DAYS (these rarely change).

If you want the definitive answer for any google.com record,
ask ns1.google.com directly:

  dig @ns1.google.com google.com
```

---

## Lab 8 — Firewall Rules (iptables)

**Concept covered:** 08 — Firewalls

### Command: `iptables -L -n`

```bash
sudo iptables -L -n --line-numbers
```

```
Chain INPUT (policy ACCEPT)
num  target  prot opt source      destination

Chain FORWARD (policy DROP)
num  target          prot opt source      destination
1    DOCKER-USER     0    --  0.0.0.0/0   0.0.0.0/0
2    DOCKER-FORWARD  0    --  0.0.0.0/0   0.0.0.0/0

Chain OUTPUT (policy ACCEPT)
num  target  prot opt source      destination
```

### Reading Every Chain

```
iptables has 3 main chains (concept 08):

Chain INPUT (policy ACCEPT)
  │                    │
  │                    └── DEFAULT policy: ACCEPT everything
  │                        No rules = all inbound traffic is allowed
  │
  └── INPUT = packets coming IN to this machine

  ⚠ No rules here! This machine has NO host-level inbound firewall.
  SSH (22), any port — all accepted.
  Security relies on the CORPORATE firewall (those invisible hops
  in traceroute), not on this machine's iptables.


Chain FORWARD (policy DROP)
  │                     │
  │                     └── DEFAULT: DROP everything being forwarded
  │
  └── FORWARD = packets passing THROUGH this machine (routing)
      The DOCKER rules handle container traffic forwarding.

  Rule 1: DOCKER-USER — custom user rules for Docker (usually empty)
  Rule 2: DOCKER-FORWARD — Docker's own forwarding logic


Chain OUTPUT (policy ACCEPT)
  │                     │
  │                     └── DEFAULT: ACCEPT all outbound
  │
  └── OUTPUT = packets going OUT from this machine
      No rules = this machine can send anything anywhere.
```

### What This Means for Security

```
This machine's iptables is essentially WIDE OPEN:
  INPUT:   policy ACCEPT, no rules → everything allowed in
  OUTPUT:  policy ACCEPT, no rules → everything allowed out
  FORWARD: policy DROP + Docker rules → only Docker traffic forwarded

This is common in corporate environments where:
  1. The NETWORK firewall (Palo Alto, etc.) handles security
  2. Individual servers rely on the network perimeter
  3. Host firewalls are not configured

In AWS, this would be like having an EC2 instance with a
Security Group that allows all inbound — NOT recommended.
The equivalent of the corporate network firewall = AWS Security Groups.
```

---

## Lab 9 — TLS Handshake and Certificates

**Concept covered:** 10 — SSL/TLS

### Command: `openssl s_client` (See the Certificate)

```bash
echo | openssl s_client -connect google.com:443 -servername google.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

```
subject=CN = *.google.com
issuer=O = GE HealthCare, OU = GE HealthCare Proxy, CN = GEHC Proxy RN ECC 202601197000
notBefore=Mar 30 08:35:08 2026 GMT
notAfter=Jun 22 08:35:07 2026 GMT
```

### Reading This — And Why It's Surprising

```
subject=CN = *.google.com
  → The certificate says it's for "*.google.com" (wildcard — covers
    any subdomain like www.google.com, mail.google.com, etc.)

issuer=O = GE HealthCare, OU = GE HealthCare Proxy
  │
  └── WAIT. Google's certificate is issued by... GE HealthCare?!
      
      This is because you're behind a CORPORATE TLS PROXY.
      
      Here's what's happening (concept 10 — chain of trust):
      
      Normal internet:
        google.com cert → signed by Google Trust Services → trusted

      YOUR corporate network:
        Your browser → GEHC Proxy (Palo Alto/Zscaler) → Google
        
        The proxy INTERCEPTS the TLS connection:
        1. You connect to google.com
        2. The GEHC proxy intercepts, connects to Google on your behalf
        3. The proxy creates a NEW certificate for *.google.com
           signed by "GEHC Proxy RN ECC" (the corporate CA)
        4. Your machine trusts "GE HealthCare Root CA 2" (installed
           in your OS trust store by IT)
        5. You see a valid certificate — but it's NOT Google's real cert
        
      WHY? So the corporate firewall can INSPECT encrypted traffic
      for malware, data loss prevention, and security policy.

notBefore=Mar 30 08:35:08 2026 GMT
notAfter=Jun 22 08:35:07 2026 GMT
  → Valid for ~3 months. The proxy auto-generates short-lived certs.
```

### The Full Certificate Chain

```bash
echo | openssl s_client -connect google.com:443 -servername google.com \
  -showcerts 2>/dev/null | grep -E "s:|i:"
```

```
 0 s:CN = *.google.com
   i:O = GE HealthCare, OU = GE HealthCare Proxy, CN = GEHC Proxy RN ECC 202601197000
 1 s:O = GE HealthCare, OU = GE HealthCare Proxy, CN = GEHC Proxy RN ECC 202601197000
   i:C = US, O = GE HealthCare, OU = Enterprise, CN = GE HealthCare Int CA 2 G2
 2 s:C = US, O = GE HealthCare, OU = Enterprise, CN = GE HealthCare Int CA 2 G2
   i:C = US, O = GE HealthCare, OU = Enterprise, CN = GE HealthCare Root CA 2
 3 s:C = US, O = GE HealthCare, OU = Enterprise, CN = GE HealthCare Root CA 2
   i:C = US, O = GE HealthCare, OU = Enterprise, CN = GE HealthCare Root CA 2
```

```
Reading the chain (s = subject, i = issuer):

Cert 0: *.google.com
  │     Signed by: GEHC Proxy RN ECC (the TLS inspection proxy)
  │
Cert 1: GEHC Proxy RN ECC
  │     Signed by: GE HealthCare Int CA 2 G2 (intermediate CA)
  │
Cert 2: GE HealthCare Int CA 2 G2
  │     Signed by: GE HealthCare Root CA 2 (root CA)
  │
Cert 3: GE HealthCare Root CA 2
        Signed by: itself (self-signed — this is the ROOT)
        This root cert is installed in your OS trust store.

Chain of trust (concept 10):
  *.google.com → Proxy CA → Intermediate CA → Root CA (TRUSTED ✓)
```

### TLS Version and Cipher

```bash
echo | openssl s_client -connect google.com:443 -servername google.com 2>/dev/null \
  | grep -E "Protocol|Cipher"
```

```
    Protocol  : TLSv1.2
    Cipher    : ECDHE-ECDSA-AES128-GCM-SHA256
```

```
Protocol: TLSv1.2
  → Current standard. TLS 1.3 would be better but the proxy
    likely supports 1.2 for compatibility.

Cipher: ECDHE-ECDSA-AES128-GCM-SHA256
  │      │      │       │     │
  │      │      │       │     └── SHA256: hash algorithm for integrity
  │      │      │       └── GCM: Galois/Counter Mode (authenticated encryption)
  │      │      └── AES128: symmetric encryption (128-bit key)
  │      └── ECDSA: server authenticated with Elliptic Curve signature
  └── ECDHE: Elliptic Curve Diffie-Hellman Ephemeral (key exchange)

  In plain English:
    "We used Diffie-Hellman to agree on a key (concept 10),
     then AES-128 to encrypt all data."
```

---

## Lab 10 — Full HTTP Request with curl

**Concepts covered:** 05 — HTTP, 07 — DNS, 10 — TLS

### Command: `curl -v` (Verbose — See Everything)

```bash
curl -sv -o /dev/null https://httpbin.org/get 2>&1 | grep -E "^\*|^>|^<"
```

This shows DNS resolution → TLS handshake → HTTP request → HTTP response:

```
* Host httpbin.org:443 was resolved.
* IPv6: (none)
* IPv4: 3.234.199.80, 54.198.84.224, 98.84.87.4, 3.217.55.179
```

```
Step 1 — DNS (concept 07):
  curl asks: "What's the IP for httpbin.org?"
  DNS returns 6 IPv4 addresses (all AWS IPs — httpbin runs on AWS)
  No IPv6 available on this network
```

```
*   Trying 3.234.199.80:443...
* Connected to httpbin.org (3.234.199.80) port 443
```

```
Step 2 — TCP Connection (concept 05):
  curl picks the first IP (3.234.199.80) and connects on port 443
  This is the TCP three-way handshake: SYN → SYN-ACK → ACK
```

```
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
```

```
Step 3 — TLS Handshake (concept 10):
  Client hello  → "I support TLS 1.3 and these ciphers"
  Server hello  → "Let's use TLS 1.2" (proxy downgraded from 1.3 to 1.2)
  Certificate   → server sends its certificate
  Key exchange  → both sides compute shared secret
  Finished      → handshake complete, encrypted channel established

  curl offered TLS 1.3, but the GEHC proxy only supports 1.2.
  The actual cipher negotiated: ECDHE-RSA-AES128-GCM-SHA256
```

```
* Server certificate:
*  subject: CN=httpbin.org
*  issuer: O=GE HealthCare; OU=GE HealthCare Proxy
*  SSL certificate verify ok.
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits)
*   Certificate level 1: Public key type RSA (2048/112 Bits/secBits)
*   Certificate level 2: Public key type RSA (4096/152 Bits/secBits)
*   Certificate level 3: Public key type RSA (4096/152 Bits/secBits)
```

```
Step 4 — Certificate Verification (concept 10):
  Certificate for httpbin.org — also issued by GEHC Proxy (same TLS inspection)
  4 certificates in the chain (levels 0-3)
  "SSL certificate verify ok" — chain trusted ✓
```

```
> GET /get HTTP/2
> Host: httpbin.org
> User-Agent: curl/8.5.0
> Accept: */*
```

```
Step 5 — HTTP Request (concept 05):
  > = what curl SENT (request)
  
  GET /get HTTP/2     ← method=GET, path=/get, protocol=HTTP/2
  Host: httpbin.org   ← which website (required for virtual hosting)
  User-Agent: curl/8.5.0  ← "I'm curl" (browsers say "Mozilla/5.0...")
  Accept: */*         ← "I'll accept any content type"
```

```
< HTTP/2 200
< date: Fri, 24 Apr 2026 05:37:58 GMT
< content-type: application/json
< content-length: 254
< server: gunicorn/19.9.0
< access-control-allow-origin: *
< access-control-allow-credentials: true
```

```
Step 6 — HTTP Response (concept 05):
  < = what the server SENT BACK (response)
  
  HTTP/2 200           ← status code 200 = OK (success!)
  content-type: application/json  ← response body is JSON
  content-length: 254  ← 254 bytes of data
  server: gunicorn     ← the web server software (Python WSGI server)
  access-control-allow-origin: *  ← CORS header — any website can call this API
```

---

## Lab 11 — Port Checking with nc (netcat)

**Concept covered:** 05 — Ports, 08 — Firewalls

### Command: `nc -zv` (Check if a Port Is Open)

```bash
nc -zv -w 3 google.com 443
```

```
Connection to google.com (172.253.118.100) 443 port [tcp/https] succeeded!
```

```
nc flags:
  -z = scan mode (don't send data, just check if port is open)
  -v = verbose
  -w 3 = timeout after 3 seconds

"succeeded" = port 443 is OPEN and accepting connections.
The TCP three-way handshake completed successfully.
```

```bash
nc -zv -w 3 google.com 22
```

```
nc: connect to google.com (172.253.118.101) port 22 (tcp) timed out: Operation now in progress
nc: connect to google.com (172.253.118.138) port 22 (tcp) timed out: Operation now in progress
...
```

```
"timed out" = port 22 is NOT reachable.
nc tried every IP that DNS returned (Google has 6) — ALL timed out.

Why? Google doesn't run SSH on port 22 for public access.
Their firewall DROPs traffic to port 22 (concept 08 — default deny).

Three possible responses when checking a port:
  "succeeded"              → port is OPEN (service is listening, firewall allows)
  "timed out"              → port is FILTERED (firewall silently drops — no response)
  "Connection refused"     → port is CLOSED (nothing listening, but network is reachable)
```

### Quick Port Check Cheat Sheet

```bash
# Is SSH open on a server?
nc -zv -w 3 10.0.1.50 22

# Is PostgreSQL reachable?
nc -zv -w 3 10.0.2.30 5432

# Is HTTPS working?
nc -zv -w 3 api.example.com 443

# Check multiple ports at once:
for port in 22 80 443 8080; do
  nc -zv -w 2 google.com $port 2>&1
done
```

---

## Lab 12 — Established Connections: What's Talking Right Now

**Concepts covered:** 05 — Sockets, 06 — How Data Travels

### Command: `ss -tnp state established`

```bash
ss -tnp state established
```

```
Recv-Q Send-Q Local Address:Port    Peer Address:Port  Process
0      0      10.12.24.141:1001   10.177.213.232:2049
0      0      10.12.24.141:60458     99.86.20.55:443   node
0      0      10.12.24.141:9090    10.66.49.170:55893  mkdocs
0      480    10.12.24.141:22      10.66.49.170:64680  sshd
0      0         127.0.0.1:39902      127.0.0.1:40513  sshd
0      0      10.12.24.141:58124    20.189.173.1:443   node
0      0      10.12.24.141:57054   140.82.113.22:443   node
0      0         127.0.0.1:40513      127.0.0.1:39902  code-10c8e557c8
```

### Reading Every Connection

```
10.12.24.141:22 ←→ 10.66.49.170:64680  (sshd)
│            │      │             │       │
│            │      │             │       └── Process: SSH daemon
│            │      │             └── Ephemeral port (concept 05)
│            │      └── YOUR laptop (VS Code SSH remote)
│            └── Port 22 = SSH
└── This machine

  This is YOUR SSH session! You're connected from 10.66.49.170 (your laptop)
  to this machine on port 22. The Send-Q of 480 means data is being
  sent to you right now (probably terminal output).


10.12.24.141:9090 ←→ 10.66.49.170:55893  (mkdocs)
  MkDocs dev server is running on port 9090.
  Your laptop (10.66.49.170) is viewing the MkDocs site.


10.12.24.141:60458 ←→ 99.86.20.55:443  (node)
  VS Code extension making HTTPS call to 99.86.20.55 (AWS CloudFront IP).
  Port 443 = HTTPS.
  60458 = ephemeral port picked by the OS (concept 05).


10.12.24.141:57054 ←→ 140.82.113.22:443  (node)
  VS Code connecting to 140.82.113.22 port 443.
  140.82.113.x = GitHub's IP range.
  This is VS Code's GitHub Copilot or extension updates.


10.12.24.141:58124 ←→ 20.189.173.1:443  (node)
  VS Code connecting to 20.189.173.x = Microsoft Azure IP.
  This is VS Code telemetry or marketplace.


10.12.24.141:1001 ←→ 10.177.213.232:2049
  Port 2049 = NFS (Network File System).
  This machine has an NFS mount from 10.177.213.232.
  Port 1001 = local ephemeral port for the NFS client.


127.0.0.1:39902 ←→ 127.0.0.1:40513  (sshd ↔ code)
  Localhost-to-localhost connection.
  SSH daemon forwarding data to VS Code server.
  This is how VS Code Remote SSH works internally.
```

### The Diagram of Active Connections

```
Your Laptop (10.66.49.170)
  ├── :64680 ──SSH──→ 10.12.24.141:22       (your terminal)
  └── :55893 ──HTTP──→ 10.12.24.141:9090    (mkdocs preview)

This Machine (10.12.24.141)
  ├── :60458 ──HTTPS──→ 99.86.20.55:443     (AWS CloudFront)
  ├── :57054 ──HTTPS──→ 140.82.113.22:443   (GitHub)
  ├── :58124 ──HTTPS──→ 20.189.173.1:443    (Microsoft Azure)
  └── :1001  ──NFS────→ 10.177.213.232:2049 (file server)

Internal (127.0.0.1)
  └── :39902 ←→ :40513  (sshd ↔ VS Code server)
```

---

## Quick Reference: All Commands on One Page

```bash
# ─── YOUR IDENTITY ────────────────────────────────────
ip -br addr show              # All interfaces and IPs
ip addr show ens192            # Detailed info for main interface
hostname                       # Machine name

# ─── SUBNET AND NEIGHBORS ────────────────────────────
ip neigh show                  # ARP table (who's on your LAN)
ip neigh show dev ens192       # ARP for main interface only

# ─── ROUTING ─────────────────────────────────────────
ip route show                  # Full route table
ip route get 172.253.118.102   # "How would I reach this IP?"

# ─── PORTS AND CONNECTIONS ───────────────────────────
ss -tlnp                       # TCP listening ports
ss -ulnp                       # UDP listening ports
ss -tnp state established      # Active connections
ss -s                          # Connection summary/stats

# ─── CONNECTIVITY ────────────────────────────────────
ping -c 3 google.com           # Basic reachability test
traceroute -I -n google.com    # Show every hop (ICMP mode)
nc -zv -w 3 HOST PORT          # Check if a specific port is open

# ─── DNS ─────────────────────────────────────────────
dig google.com                 # Full DNS lookup
dig +short google.com          # Just the IPs
dig google.com MX              # Mail servers
dig google.com NS              # Name servers
dig @8.8.8.8 google.com       # Use specific DNS server
resolvectl status              # Your DNS resolver config

# ─── FIREWALL ────────────────────────────────────────
sudo iptables -L -n            # View firewall rules
sudo iptables -L -n -v         # With packet/byte counters

# ─── TLS / CERTIFICATES ─────────────────────────────
# View certificate details:
echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# View certificate chain:
echo | openssl s_client -connect HOST:443 -servername HOST \
  -showcerts 2>/dev/null | grep -E "s:|i:"

# View TLS version and cipher:
echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null \
  | grep -E "Protocol|Cipher"

# ─── HTTP ────────────────────────────────────────────
curl -sv -o /dev/null https://URL   # Full verbose (TLS + HTTP)
curl -I https://URL                  # Response headers only
curl -w "%{http_code}" -o /dev/null -s https://URL  # Just status code
```

---

> This is a companion lab file for the networking fundamentals series (concepts 01–11).
> Run these commands on any Linux machine to explore networking yourself.
