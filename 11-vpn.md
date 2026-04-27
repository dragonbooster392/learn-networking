# 11 — VPN (Virtual Private Network)

How VPNs create encrypted tunnels across the internet — remote access VPN, site-to-site VPN, tunneling protocols, and why every company uses them.

**Prerequisites:** Concept 03 (IP addresses, public vs private), Concept 04 (subnets), Concept 06 (how data travels, routing), Concept 08 (firewalls), Concept 10 (encryption, TLS).

---

## Table of Contents

- [What Is a VPN](#what-is-a-vpn)
- [The Secret Tunnel Analogy](#the-secret-tunnel-analogy)
- [Why VPNs Exist](#why-vpns-exist)
- [How a VPN Works](#how-a-vpn-works)
- [Tunneling — Packet Inside a Packet](#tunneling-packet-inside-a-packet)
- [Types of VPN](#types-of-vpn)
- [Remote Access VPN](#remote-access-vpn)
- [Site-to-Site VPN](#site-to-site-vpn)
- [How Remote Access VPN Works — Step by Step](#how-remote-access-vpn-works-step-by-step)
- [How Site-to-Site VPN Works — Step by Step](#how-site-to-site-vpn-works-step-by-step)
- [Split Tunneling vs Full Tunneling](#split-tunneling-vs-full-tunneling)
- [VPN Protocols](#vpn-protocols)
- [IPSec — The Standard for Site-to-Site](#ipsec-the-standard-for-site-to-site)
- [What Happens to Your IP Address](#what-happens-to-your-ip-address)
- [VPN vs TLS — Different Layers](#vpn-vs-tls-different-layers)
- [When to Use a VPN vs When Not To](#when-to-use-a-vpn-vs-when-not-to)
- [Common VPN Problems](#common-vpn-problems)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## What Is a VPN

A **VPN** (Virtual Private Network) creates an **encrypted tunnel** between two points across an untrusted network (like the internet). All traffic inside the tunnel is private — no one on the internet can see it.

```
WITHOUT VPN:
  Your laptop ──── internet (anyone can see traffic) ────→ company server
  
WITH VPN:
  Your laptop ════ encrypted tunnel through internet ════→ company network
                   (nobody can see what's inside)
```

"Virtual" because it creates a private network **virtually** over the public internet, instead of needing a dedicated physical cable.

---

## The Secret Tunnel Analogy

```
Imagine two buildings (your home and your office) connected by a public street.
Anyone on the street can see you walking and hear what you're saying.

A VPN is like building a secret underground tunnel between the two buildings.
  - You enter the tunnel at home
  - You walk through it (invisible to everyone on the street)
  - You emerge inside the office building

To everyone on the street, you just... disappeared.
They don't know where you went or what you're carrying.
```

---

## Why VPNs Exist

### Problem 1: Accessing Private Resources Remotely

```
Company servers are on a PRIVATE network: 10.0.0.0/8
  Database: 10.0.2.50
  Internal tools: 10.0.1.100
  Intranet: 10.0.1.10

You're working from home. Your IP is 192.168.1.5 (different network).
You CAN'T reach 10.0.x.x — those are private, not routable on the internet.

Solution: VPN gives your laptop a VIRTUAL IP on the company network.
  Your laptop gets 10.0.5.50 (a company IP, through the VPN tunnel)
  Now you can reach 10.0.2.50, 10.0.1.100, etc. as if you're in the office.
```

### Problem 2: Connecting Two Office Networks

```
New York office: 10.1.0.0/16
London office:   10.2.0.0/16

These are private networks in different cities.
They can't reach each other over the internet (private IPs aren't routable).

Solution: Site-to-site VPN between the two offices.
  Traffic between 10.1.x.x and 10.2.x.x flows through an encrypted tunnel.
  Both offices can communicate as if on the same network.
```

### Problem 3: Security on Untrusted Networks

```
You're at a coffee shop on public Wi-Fi.
Anyone on that Wi-Fi can sniff your traffic.

Solution: VPN encrypts ALL your traffic before it leaves your laptop.
  Other Wi-Fi users see encrypted gibberish.
  Your traffic goes through the tunnel to the VPN server, then out to the internet.
```

---

## How a VPN Works

```
1. Your laptop connects to a VPN server (authentication)
2. An encrypted tunnel is established
3. Your laptop gets a VIRTUAL IP address on the VPN network
4. All traffic (or selected traffic) is sent through the tunnel
5. The VPN server decrypts it and forwards it to the destination
6. Responses come back through the tunnel the same way

Your Laptop                  VPN Server               Destination
(192.168.1.5)               (52.10.20.30)            (10.0.1.100)
     │                            │                        │
     │  Encrypted tunnel          │                        │
     │ ═══════════════════════════│                        │
     │  (outer: 192.168.1.5 →    │                        │
     │   52.10.20.30)            │                        │
     │  (inner: 10.0.5.50 →     │  Decrypted:            │
     │   10.0.1.100)            │  10.0.5.50 → 10.0.1.100│
     │                            │───────────────────────→│
     │                            │                        │
     │                            │←───────────────────────│
     │ ═══════════════════════════│                        │
     │  (encrypted response)      │                        │
```

---

## Tunneling — Packet Inside a Packet

The key concept behind VPN is **tunneling** — wrapping an original packet inside an encrypted outer packet:

```
Original packet (what you actually want to send):
  ┌────────────────────────────────────────┐
  │ Source: 10.0.5.50  Dest: 10.0.1.100   │  ← private IPs
  │ Data: "SELECT * FROM users"            │
  └────────────────────────────────────────┘

This packet can't travel over the internet (private IPs).

VPN wraps it:
  ┌──────────────────────────────────────────────────────────┐
  │ Outer Header:                                            │
  │   Source: 192.168.1.5  Dest: 52.10.20.30 (VPN server)  │  ← public IPs
  │   Protocol: UDP/4500 (IPSec) or TCP/443 (SSL VPN)      │
  │                                                          │
  │ ┌────────────────────────────────────────────────────┐  │
  │ │ ENCRYPTED PAYLOAD:                                  │  │
  │ │  Source: 10.0.5.50  Dest: 10.0.1.100               │  │
  │ │  Data: "SELECT * FROM users"                        │  │
  │ └────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘

The outer packet travels over the internet using public IPs.
Nobody can see the inner packet — it's encrypted.

VPN server receives it:
  1. Strips outer header
  2. Decrypts payload
  3. Forwards original packet (10.0.5.50 → 10.0.1.100) on the private network
```

---

## Types of VPN

### Remote Access VPN

One user connects from anywhere to a company network.

```
                  Internet
Home/Coffee Shop ══════════ VPN Server ──── Company Network
(your laptop)               (gateway)        (10.0.0.0/8)
192.168.1.5                 52.10.20.30

Your laptop gets virtual IP: 10.0.5.50
Now you can access:
  10.0.1.100 (intranet)
  10.0.2.50 (database)
  10.0.3.30 (internal API)
```

**Tools:** Cisco AnyConnect, GlobalProtect (Palo Alto), OpenVPN, WireGuard, AWS Client VPN.

### Site-to-Site VPN

Two entire networks connected permanently through a VPN tunnel.

```
New York Office              Tunnel              London Office
10.1.0.0/16    ════════════════════════════════   10.2.0.0/16

VPN Router ←────── encrypted tunnel ──────→ VPN Router
10.1.0.1                                    10.2.0.1

Any device in NY can reach any device in London:
  10.1.0.50 → 10.2.0.100 (goes through tunnel automatically)

Users don't need VPN software — the ROUTERS handle it.
```

**Tools:** IPSec (standard), AWS Site-to-Site VPN, direct on firewalls (Palo Alto, Cisco).

---

## How Remote Access VPN Works — Step by Step

```
Step 1: You open VPN client and enter credentials
  Username: john@company.com
  Password: ****
  (or certificate-based authentication)

Step 2: VPN client connects to VPN server (52.10.20.30)
  → TLS or IPSec handshake
  → Server authenticates you (username/password, MFA, certificate)
  → Encrypted tunnel established

Step 3: VPN server assigns you a virtual IP
  "Your VPN IP is 10.0.5.50"
  Your laptop now has TWO IPs:
    192.168.1.5 (real, home network)
    10.0.5.50 (virtual, company network)

Step 4: VPN server pushes routes to your laptop
  "Route 10.0.0.0/8 through the VPN tunnel"
  Now when you access 10.0.1.100, traffic goes through the tunnel.

Step 5: You work as if you're in the office
  Browse intranet: 10.0.1.10 → tunnel → company network → intranet
  SSH to server: 10.0.2.50 → tunnel → company network → server
```

---

## How Site-to-Site VPN Works — Step by Step

```
Step 1: VPN routers at both sites are configured
  NY router: "Tunnel to 203.0.113.50 (London's public IP)"
  London router: "Tunnel to 198.51.100.10 (NY's public IP)"

Step 2: IPSec tunnel established between routers
  → IKE handshake (Internet Key Exchange — negotiates encryption)
  → Encryption keys exchanged
  → Tunnel is UP

Step 3: Route tables configured
  NY router: "10.2.0.0/16 → send through VPN tunnel"
  London router: "10.1.0.0/16 → send through VPN tunnel"

Step 4: Traffic flows automatically
  NY device (10.1.0.50) sends packet to London (10.2.0.100):
    → NY router sees dest 10.2.0.0/16 → VPN tunnel
    → Encrypts packet → sends to London's public IP
    → London router decrypts → forwards to 10.2.0.100
    → Response takes the reverse path

Users don't even know a VPN is involved. It just works.
```

---

## Split Tunneling vs Full Tunneling

### Full Tunneling

ALL traffic goes through the VPN, even internet traffic:

```
Full tunnel:
  google.com → VPN tunnel → company network → internet → Google
  10.0.1.100 → VPN tunnel → company network → server

✓ Company can inspect/filter ALL your traffic (security)
✓ Your real IP is hidden from all websites
✗ Slower (everything goes through the tunnel, even YouTube)
✗ Uses company bandwidth for personal traffic
```

### Split Tunneling

Only company traffic goes through the VPN. Internet traffic goes directly:

```
Split tunnel:
  google.com → directly to internet (no VPN)
  10.0.1.100 → VPN tunnel → company network → server

✓ Faster (internet traffic doesn't go through VPN)
✓ Less load on company VPN
✗ Company can't inspect internet traffic
✗ Your real IP is visible to websites
```

```
Route-based split tunneling:
  10.0.0.0/8 → through VPN tunnel    (company traffic)
  0.0.0.0/0  → direct internet        (everything else)
```

---

## VPN Protocols

| Protocol | Layer | Speed | Security | Use Case |
|----------|-------|-------|----------|----------|
| **IPSec** | Layer 3 | Fast | Strong | Site-to-site (industry standard) |
| **OpenVPN** | Layer 3-4 | Medium | Strong | Remote access (open source) |
| **WireGuard** | Layer 3 | Very fast | Strong | Remote access (modern, simple) |
| **SSL/TLS VPN** | Layer 4-7 | Medium | Strong | Remote access through browsers |
| **L2TP/IPSec** | Layer 2 | Medium | Strong | Legacy remote access |
| **PPTP** | Layer 2 | Fast | **BROKEN** | **Never use** — easily cracked |

---

## IPSec — The Standard for Site-to-Site

**IPSec** (Internet Protocol Security) is the most widely used VPN protocol for site-to-site connections:

```
IPSec has two phases:

Phase 1 — IKE (Internet Key Exchange):
  Both sides authenticate each other (pre-shared key or certificates)
  Negotiate encryption settings (AES-256, SHA-256, etc.)
  Create a secure channel for Phase 2

Phase 2 — IPSec Tunnel:
  Negotiate which traffic to encrypt (e.g., 10.1.0.0/16 ↔ 10.2.0.0/16)
  Create encryption keys for actual data
  Tunnel is UP — data flows encrypted
```

### IPSec Modes

```
Tunnel Mode (site-to-site):
  Entire IP packet is encrypted and wrapped in a new IP header
  Original source/dest IPs hidden
  This is what site-to-site VPNs use

Transport Mode (host-to-host):
  Only the payload is encrypted, IP header stays visible
  Used for direct server-to-server encryption (less common)
```

---

## What Happens to Your IP Address

### Remote Access VPN

```
Without VPN:
  Your public IP: 73.162.45.201 (your ISP)
  Websites see: 73.162.45.201

With VPN (full tunnel):
  Your traffic exits through the VPN server
  VPN server's IP: 52.10.20.30
  Websites see: 52.10.20.30

  → Your real IP is hidden
  → Websites think you're at the VPN server's location
```

### Site-to-Site VPN

```
Packet from NY (10.1.0.50) to London (10.2.0.100):

On the internet (outer packet):
  Source: 198.51.100.10 (NY router's public IP)
  Dest:   203.0.113.50 (London router's public IP)

Inside the tunnel (inner packet, encrypted):
  Source: 10.1.0.50
  Dest:   10.2.0.100

Internet routers only see the outer public IPs.
The private IPs are encrypted inside the tunnel.
```

---

## VPN vs TLS — Different Layers

```
TLS (concept 10):
  Encrypts ONE application's traffic (e.g., your browser to one website)
  Works at the application layer
  Per-connection encryption

VPN:
  Encrypts ALL traffic (or all traffic to certain destinations)
  Works at the network layer
  Tunnel-level encryption

Example:
  You connect to VPN → access company intranet (10.0.1.10)
  The intranet uses HTTP (not HTTPS) internally.
  
  Is the traffic encrypted? YES — the VPN tunnel encrypts it.
  Is there TLS? NO — the website itself is HTTP.

  VPN encrypts the tunnel.
  TLS encrypts the application.
  Using BOTH is ideal (TLS inside VPN = double protection).
```

---

## When to Use a VPN vs When Not To

| Situation | VPN Needed? | Why |
|-----------|-------------|-----|
| Access company internal tools from home | **Yes** | Private IPs only reachable via VPN |
| Browse the web securely at a coffee shop | **Maybe** | HTTPS already encrypts. VPN adds IP hiding |
| Connect two office networks | **Yes** (site-to-site) | Only way to bridge private networks |
| Access a public website (google.com) | **No** | HTTPS is sufficient |
| SSH to a private EC2 instance | **Yes** | Instance has no public IP |
| Bypass geo-restrictions | **Consumer VPN** | Not the same as corporate VPN |

---

## Common VPN Problems

### "VPN connected but can't reach internal resources"

```
Check:
  1. Routes: does your laptop know to send 10.0.x.x through the VPN?
     ip route show | grep 10.0
  2. DNS: can you resolve internal hostnames?
     dig server.internal.company.com
  3. Firewall: is the destination server allowing your VPN IP?
  4. Split tunnel: is the traffic going through the VPN or direct?
```

### "VPN is slow"

```
Causes:
  - Full tunneling (all traffic through VPN, including YouTube)
  - VPN server is far away (high latency)
  - VPN server is overloaded (too many users)
  - Encryption overhead (minimal usually)

Fix:
  - Use split tunneling
  - Connect to a closer VPN server
  - Use WireGuard (faster than OpenVPN)
```

### "VPN keeps disconnecting"

```
Causes:
  - NAT timeout (NAT table entry expires — concept 09)
  - Firewall blocking VPN traffic
  - Unstable network connection

Fix:
  - Enable keep-alive/dead peer detection
  - Check firewall allows UDP/4500 and UDP/500 (IPSec)
  - Check firewall allows TCP/443 (SSL VPN)
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **VPN** | Encrypted tunnel between two points over the internet |
| **Remote access VPN** | One user connects to a company network from anywhere |
| **Site-to-site VPN** | Two entire networks connected via permanent tunnel |
| **Tunneling** | Wrapping a packet inside another encrypted packet |
| **IPSec** | Standard protocol for site-to-site VPNs |
| **Split tunneling** | Only company traffic goes through VPN; internet traffic goes direct |
| **Full tunneling** | ALL traffic goes through VPN |
| **Virtual IP** | IP address assigned to you on the VPN network |
| **IKE** | Key exchange protocol used by IPSec |
| **WireGuard** | Modern, fast VPN protocol |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| **Site-to-site VPN** | **AWS Site-to-Site VPN** — connects your data center to AWS VPC via IPSec tunnels |
| **Remote access VPN** | **AWS Client VPN** — employees connect to AWS VPCs from laptops |
| **VPN gateway** | **Virtual Private Gateway (VGW)** — the AWS side of a site-to-site VPN |
| **Customer gateway** | **Customer Gateway** — your on-premises router/firewall side of the VPN |
| **Transit Gateway** | **TGW** can terminate VPN connections from multiple sites |
| **IPSec** | AWS Site-to-Site VPN uses IPSec with two tunnels for redundancy |
| **Split tunneling** | AWS Client VPN supports split tunneling (routes only specified CIDRs through VPN) |

### AWS Site-to-Site VPN

```
Your Data Center                         AWS VPC (10.0.0.0/16)
10.1.0.0/16                              
                                          ┌──────────────────────┐
┌──────────┐    IPSec Tunnel 1           │                      │
│ Customer │ ════════════════════════════ │  Virtual Private     │
│ Gateway  │                             │  Gateway (VGW)       │
│ (router) │ ════════════════════════════ │                      │
└──────────┘    IPSec Tunnel 2           └──────────────────────┘
                (redundancy)              
                                          Route table:
Route table:                               10.1.0.0/16 → VGW
10.0.0.0/16 → VPN tunnel                  10.0.0.0/16 → local

Two tunnels for high availability.
If tunnel 1 goes down, traffic uses tunnel 2.
```

### AWS Client VPN

```
Employee laptop                          AWS VPC
     │                                    
     │ OpenVPN connection                 ┌──────────────────────┐
     │ ══════════════════════════════════ │  Client VPN          │
     │                                   │  Endpoint             │
     │ Gets virtual IP: 10.100.0.50      │  (10.100.0.0/16)     │
     │                                   └──────┬───────────────┘
     │                                          │
     │ Can now access:                          │
     │   10.0.1.50 (EC2 instance)              │ Routes to VPC
     │   10.0.2.30 (RDS database)              │ subnets
```

### VPN vs Direct Connect

```
VPN (over internet):
  ✓ Quick to set up (minutes/hours)
  ✓ Cheap (~$36/month per tunnel in AWS)
  ✗ Bandwidth limited by internet connection
  ✗ Latency varies (internet is unpredictable)
  ✗ Encrypted (overhead)

Direct Connect (dedicated physical cable):
  ✓ Consistent high bandwidth (1Gbps, 10Gbps, 100Gbps)
  ✓ Consistent low latency
  ✓ No internet involved — private connection
  ✗ Expensive ($$$)
  ✗ Takes weeks/months to provision
  ✗ Need colocation or partner facility

Many companies use BOTH:
  Direct Connect for primary (high bandwidth, low latency)
  VPN as backup (if Direct Connect goes down)
```

### Your Architecture — Transit Gateway + VPN

```
In your NGPS architecture:
  Connectivity Plane VPC → Transit Gateway → Site-to-Site VPN → Data Centers

  External traffic path:
    Data Center ── VPN ──→ TGW ──→ Connectivity VPC ──→ Palo Alto ──→ App VPC

  The Palo Alto firewalls (concept 08) inspect traffic AFTER it arrives
  through the VPN tunnel. The VPN provides encryption in transit;
  the Palo Alto provides deep packet inspection and security policy.
```

---

> **Previous:** [10 — SSL/TLS](10-ssl-tls.md)
> **Next:** This is the final concept in the networking fundamentals series. You're ready for the AWS course!
