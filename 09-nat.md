# 09 — NAT (Network Address Translation)

How your home router lets 10 devices share one public IP — SNAT, DNAT, PAT, and why private IPs need translation to reach the internet.

**Prerequisites:** Concept 03 (public vs private IPs), Concept 04 (subnets), Concept 05 (ports, ephemeral ports), Concept 06 (how data travels, routing).

---

## Table of Contents

- [The Problem NAT Solves](#the-problem-nat-solves)
- [What Is NAT](#what-is-nat)
- [The Front Desk Analogy](#the-front-desk-analogy)
- [Why Private IPs Can't Reach the Internet](#why-private-ips-cant-reach-the-internet)
- [How NAT Works — Step by Step](#how-nat-works-step-by-step)
- [The NAT Translation Table](#the-nat-translation-table)
- [Types of NAT](#types-of-nat)
- [SNAT — Source NAT](#snat-source-nat)
- [DNAT — Destination NAT](#dnat-destination-nat)
- [PAT — Port Address Translation (NAT Overload)](#pat-port-address-translation-nat-overload)
- [Static NAT vs Dynamic NAT](#static-nat-vs-dynamic-nat)
- [Your Home Router — NAT in Action](#your-home-router-nat-in-action)
- [NAT and Connection Tracking](#nat-and-connection-tracking)
- [NAT Limitations and Problems](#nat-limitations-and-problems)
- [NAT Traversal](#nat-traversal)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## The Problem NAT Solves

There are only ~4.3 billion IPv4 addresses. There are 15+ billion internet-connected devices. The numbers don't work.

```
IPv4 addresses available: ~4,300,000,000
Devices on the internet:  ~15,000,000,000+

Solution: don't give every device a public IP.
  - Give your HOME one public IP (from your ISP)
  - Give each DEVICE a private IP (from your router)
  - Use NAT to translate between them
```

Your home has ONE public IP. Your laptop, phone, tablet, smart TV, smart speaker — they all share it using NAT.

---

## What Is NAT

**NAT** (Network Address Translation) changes the source or destination IP address in a packet as it passes through a router.

```
BEFORE NAT (leaving your laptop):
  Source IP:   192.168.1.5 (private)
  Dest IP:     142.250.80.46 (Google)

AFTER NAT (leaving your router):
  Source IP:   73.162.45.201 (public — your router's IP)
  Dest IP:     142.250.80.46 (Google — unchanged)

Google sees the request coming from 73.162.45.201 (your public IP).
Google has NO IDEA that 192.168.1.5 exists.
```

---

## The Front Desk Analogy

```
Company office building:
  Employees have internal extension numbers: 101, 102, 103...
  The building has ONE public phone number: 555-1000

  Employee 101 calls a customer:
    Customer's caller ID shows: 555-1000 (the building's number, not 101)
    Customer calls back 555-1000
    Front desk (NAT) routes the return call to extension 101

  Employee 103 calls the same customer:
    Customer's caller ID shows: 555-1000 (same building number)
    But front desk knows to route this return call to extension 103

Front desk = NAT router
Extension numbers = private IPs
Building phone number = public IP
```

---

## Why Private IPs Can't Reach the Internet

Private IP ranges (concept 03) are **not routable** on the internet:

```
Private ranges (not routable on the internet):
  10.0.0.0/8        (10.x.x.x)
  172.16.0.0/12     (172.16.x.x - 172.31.x.x)
  192.168.0.0/16    (192.168.x.x)

If your laptop (192.168.1.5) sends a packet to Google:
  Source IP: 192.168.1.5
  
Google receives it. Google tries to reply:
  Destination IP: 192.168.1.5
  
Internet routers see 192.168.1.5 and say:
  "That's a private address. I don't know where that is. DROP."

The response NEVER comes back. → Connection fails.
```

**NAT fixes this** by replacing 192.168.1.5 with a public IP before the packet leaves your network.

---

## How NAT Works — Step by Step

Let's trace a complete request from your laptop to Google and back:

### Outbound (your laptop → Google)

```
Step 1: Your laptop creates a packet
  Source: 192.168.1.5:52431
  Dest:   142.250.80.46:443

Step 2: Packet reaches your router (default gateway)
  Router sees: source is a private IP
  Router does NAT:
    - Replaces source IP: 192.168.1.5 → 73.162.45.201 (public IP)
    - Replaces source port: 52431 → 12001 (new tracking port)
    - Records this in the NAT table

Step 3: Packet leaves your router
  Source: 73.162.45.201:12001
  Dest:   142.250.80.46:443
```

### Inbound (Google → your laptop)

```
Step 4: Google sends response
  Source: 142.250.80.46:443
  Dest:   73.162.45.201:12001

Step 5: Packet reaches your router
  Router checks NAT table:
    "Port 12001 → that's 192.168.1.5:52431"
  Router does reverse NAT:
    - Replaces dest IP: 73.162.45.201 → 192.168.1.5
    - Replaces dest port: 12001 → 52431

Step 6: Packet delivered to your laptop
  Source: 142.250.80.46:443
  Dest:   192.168.1.5:52431
  → Your browser receives the response
```

### The Complete Picture

```
Your Laptop              Router (NAT)            Google
192.168.1.5:52431        73.162.45.201           142.250.80.46:443
     │                        │                        │
     │  src: 192.168.1.5     │                        │
     │  dst: 142.250.80.46   │                        │
     │ ──────────────────────→│                        │
     │                        │  src: 73.162.45.201   │
     │                        │  dst: 142.250.80.46   │
     │                        │───────────────────────→│
     │                        │                        │
     │                        │  src: 142.250.80.46   │
     │                        │  dst: 73.162.45.201   │
     │                        │←───────────────────────│
     │  src: 142.250.80.46   │                        │
     │  dst: 192.168.1.5     │                        │
     │←──────────────────────│                        │
```

---

## The NAT Translation Table

The router maintains a table to track which internal device owns which connection:

```
NAT Translation Table:
  Internal IP:Port       External IP:Port      Destination         Protocol
  ─────────────────────────────────────────────────────────────────────────
  192.168.1.5:52431      73.162.45.201:12001   142.250.80.46:443   TCP
  192.168.1.5:52432      73.162.45.201:12002   142.250.80.46:443   TCP
  192.168.1.8:49100      73.162.45.201:12003   151.101.1.69:443    TCP
  192.168.1.12:60200     73.162.45.201:12004   142.250.80.46:443   TCP

When a response comes in to port 12003:
  → look up in table: that's 192.168.1.8:49100
  → translate and forward to 192.168.1.8:49100
```

This is how **multiple devices share one public IP**. Each connection gets a unique external port number.

---

## Types of NAT

### SNAT — Source NAT

Changes the **source** IP address. Used when internal devices access the internet.

```
Outbound traffic:
  Before: Source=192.168.1.5 (private)
  After:  Source=73.162.45.201 (public)

This is what your home router does.
This is what AWS NAT Gateway does.
```

### DNAT — Destination NAT

Changes the **destination** IP address. Used to make internal servers accessible from outside.

```
Scenario: You run a web server at 192.168.1.50 behind your router.
You configure "port forwarding": external port 443 → 192.168.1.50:443

Inbound traffic:
  Before: Dest=73.162.45.201:443 (your public IP)
  After:  Dest=192.168.1.50:443 (your internal server)

People on the internet connect to 73.162.45.201:443
→ Router translates destination to 192.168.1.50:443
→ Your server receives the request
```

### PAT — Port Address Translation (NAT Overload)

**PAT** is the most common form of NAT. Multiple internal IPs share **one public IP** by using different port numbers. This is what your home router does.

```
PAT — many-to-one:
  192.168.1.5:52431  → 73.162.45.201:12001
  192.168.1.8:49100  → 73.162.45.201:12002
  192.168.1.12:60200 → 73.162.45.201:12003

All three devices share 73.162.45.201.
The PORT number distinguishes which device gets which response.

Maximum connections with one public IP: ~65,000 (limited by port numbers)
```

When people say "NAT" they usually mean PAT.

---

## Static NAT vs Dynamic NAT

### Static NAT

One-to-one permanent mapping. One private IP always maps to one specific public IP.

```
Static NAT:
  10.0.1.50 ←→ 52.10.20.30 (always, permanently)
  10.0.1.51 ←→ 52.10.20.31 (always, permanently)

Use case: when a server needs a FIXED public IP
  (e.g., whitelisted by a partner's firewall)
```

### Dynamic NAT

The router picks a public IP from a **pool** for each connection. The mapping is temporary.

```
Dynamic NAT:
  Public IP pool: 52.10.20.30 - 52.10.20.35 (6 IPs)

  10.0.1.50 connects → assigned 52.10.20.30 (temporary)
  10.0.1.51 connects → assigned 52.10.20.31 (temporary)
  ...
  10.0.1.56 connects → pool exhausted! Connection fails.

Problem: limited by pool size. If pool has 6 IPs, only 6 devices at a time.
```

### PAT (most common)

Many-to-one. Hundreds of devices share one public IP using port numbers.

```
PAT:
  10.0.1.50:* → 52.10.20.30:12001
  10.0.1.51:* → 52.10.20.30:12002
  10.0.1.52:* → 52.10.20.30:12003

All share 52.10.20.30. Port number differentiates them.
```

---

## Your Home Router — NAT in Action

Every home router does NAT. Here's what's happening:

```
Your Home Network:
  Router public IP: 73.162.45.201 (from ISP)
  Router private IP: 192.168.1.1

  Laptop:    192.168.1.5
  Phone:     192.168.1.8
  Smart TV:  192.168.1.12
  Tablet:    192.168.1.15

All four devices browsing the internet simultaneously:
  Laptop → YouTube:   73.162.45.201:12001 → YouTube
  Phone → Instagram:  73.162.45.201:12002 → Instagram
  TV → Netflix:       73.162.45.201:12003 → Netflix
  Tablet → Google:    73.162.45.201:12004 → Google

YouTube, Instagram, Netflix, Google — they all see traffic from 73.162.45.201.
They have NO IDEA four different devices are behind that one IP.
```

### Port Forwarding on Home Routers

If you want to run a server at home (e.g., Minecraft server):

```
Router config: Forward external port 25565 to 192.168.1.50:25565

Someone connects to 73.162.45.201:25565
→ Router: "Port 25565? That goes to 192.168.1.50:25565" (DNAT)
→ Your Minecraft server receives the connection
```

---

## NAT and Connection Tracking

NAT requires **connection tracking** (similar to stateful firewalls):

```
1. Internal device sends outbound packet → NAT creates table entry
2. Response comes back → NAT looks up table → forwards to correct device
3. Connection closes (TCP FIN) or times out → NAT removes table entry

If the NAT table entry is gone:
  → Response packets can't be matched
  → They're DROPPED
  → Connection fails
```

This is why long-idle connections can break through NAT. The NAT table entry times out, and the response gets dropped.

---

## NAT Limitations and Problems

### 1. Breaks End-to-End Connectivity

```
Without NAT: any device can directly connect to any other device
With NAT:    devices behind NAT can't receive incoming connections
             (unless port forwarding is configured)

This is why you can't run a public server at home without port forwarding.
```

### 2. Performance Overhead

```
Every packet through NAT:
  1. Look up in NAT table
  2. Modify IP header
  3. Recalculate checksum
  4. Forward

At high traffic volumes (thousands of connections per second),
NAT can become a bottleneck.
```

### 3. Port Exhaustion

```
One public IP = ~65,000 ports available

If 65,000 connections are active simultaneously through one public IP:
  → Port exhaustion
  → New connections fail
  → "Cannot allocate port" errors

Solution: use multiple public IPs (larger NAT pool)
```

### 4. Protocols That Embed IP Addresses

```
Some protocols put IP addresses inside the DATA (not just headers):
  FTP, SIP (VoIP), some gaming protocols

NAT translates headers but doesn't always fix the data.
These protocols may break through NAT without special handling.
```

---

## NAT Traversal

How do services like video calls work if both people are behind NAT?

```
Problem:
  Person A (behind NAT): 192.168.1.5 → public 73.162.45.201
  Person B (behind NAT): 192.168.1.8 → public 104.28.55.60

  Neither can receive incoming connections.
  How do they connect to each other?

Solutions:
  1. STUN: A server helps both discover their public IP:port,
     then they connect directly (UDP hole punching)

  2. TURN: A relay server in the middle forwards traffic
     A → Relay → B (works always, but adds latency)

  3. ICE: Tries STUN first, falls back to TURN if needed
     (used by WebRTC — video calls in browsers)
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **NAT** | Changes IP addresses in packets as they pass through a router |
| **SNAT** | Changes the SOURCE IP (internal → public for outbound traffic) |
| **DNAT** | Changes the DESTINATION IP (public → internal for inbound traffic) |
| **PAT** | Multiple devices share one public IP using different port numbers |
| **Static NAT** | Permanent one-to-one mapping (private IP ↔ public IP) |
| **NAT table** | Router's record of which internal device owns which connection |
| **Port forwarding** | DNAT rule — route incoming traffic on a specific port to an internal server |
| **Port exhaustion** | All ~65,000 ports on a public IP are in use — new connections fail |
| **NAT traversal** | Techniques (STUN/TURN) to connect two devices both behind NAT |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| **SNAT** | **NAT Gateway** — lets private subnet instances reach the internet |
| **DNAT** | **Load Balancers (ALB/NLB)** — forward public traffic to private instances |
| **PAT** | NAT Gateway does PAT — many instances share the NAT Gateway's Elastic IP |
| **Static NAT** | **Elastic IP** on an EC2 instance — one-to-one public ↔ private mapping |
| **Port exhaustion** | NAT Gateway supports ~55,000 connections per destination per Elastic IP. Need more? Add more Elastic IPs |
| **Port forwarding** | Not a concept in AWS — use load balancers or Elastic IPs instead |
| **No NAT needed** | Public subnet with Internet Gateway — instances get public IPs directly |

### AWS NAT Gateway — The Most Common NAT in AWS

```
Private Subnet (10.0.2.0/24):
  EC2 instance 10.0.2.50 needs to download packages from the internet

  Route table:
    10.0.0.0/16 → local
    0.0.0.0/0   → nat-gw-abc123    ← send internet traffic to NAT Gateway

NAT Gateway (in public subnet, 10.0.1.0/24):
  Has Elastic IP: 52.10.20.30
  Does SNAT: 10.0.2.50 → 52.10.20.30

  Route table:
    10.0.0.0/16 → local
    0.0.0.0/0   → igw-xyz789       ← NAT Gateway sends to Internet Gateway

Traffic flow:
  EC2 (10.0.2.50) → NAT GW (SNAT to 52.10.20.30) → IGW → Internet
  Internet → IGW → NAT GW (reverse SNAT to 10.0.2.50) → EC2
```

### Why Not Just Put Everything in Public Subnets?

```
Security:
  Public subnet: instance has a public IP, directly reachable from internet
  Private subnet: instance has NO public IP, not directly reachable

  Database servers → private subnet (no internet access to them)
  App servers → private subnet (access internet via NAT Gateway for updates)
  Load balancers → public subnet (need to be reachable from internet)
```

### NAT Gateway vs NAT Instance

```
NAT Gateway (managed by AWS):
  ✓ Highly available, scales automatically
  ✓ No patching/maintenance
  ✗ Costs ~$32/month + data processing charges

NAT Instance (EC2 instance running NAT):
  ✓ Cheaper for low traffic
  ✗ You manage it (patching, scaling, HA)
  ✗ Single point of failure unless you set up redundancy
```

---

> **Previous:** [08 — Firewalls](08-firewalls.md)
> **Next:** [10 — SSL/TLS](10-ssl-tls.md)
