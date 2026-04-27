# 03 — IP Addresses

What an IP address is, how it works, public vs private, IPv4 vs IPv6, and how every device on the internet gets identified — explained from zero.

**Prerequisites:** Concept 01 (networks, routers), Concept 02 (Layer 3 uses IP addresses for routing).

---

## Table of Contents

- [What Is an IP Address](#what-is-an-ip-address)
- [The Postal Address Analogy](#the-postal-address-analogy)
- [IPv4 — The Standard Format](#ipv4-the-standard-format)
- [How to Read an IPv4 Address](#how-to-read-an-ipv4-address)
- [Binary — How Computers Actually See IP Addresses](#binary-how-computers-actually-see-ip-addresses)
- [How Many IPv4 Addresses Exist](#how-many-ipv4-addresses-exist)
- [Public vs Private IP Addresses](#public-vs-private-ip-addresses)
- [Private IP Ranges — Memorize These Three](#private-ip-ranges-memorize-these-three)
- [How Your Home Network Uses Both](#how-your-home-network-uses-both)
- [Static vs Dynamic IP Addresses](#static-vs-dynamic-ip-addresses)
- [DHCP — How Devices Get IP Addresses Automatically](#dhcp-how-devices-get-ip-addresses-automatically)
- [Loopback Address — 127.0.0.1](#loopback-address-127001)
- [Special IP Addresses](#special-ip-addresses)
- [IPv6 — The Future (and Present)](#ipv6-the-future-and-present)
- [Finding Your Own IP Address](#finding-your-own-ip-address)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## What Is an IP Address

An **IP address** (Internet Protocol address) is a number assigned to every device on a network. It's how devices find each other — like a phone number for computers.

```
Your laptop:   192.168.1.5
Your phone:    192.168.1.8
Google server: 142.250.80.46
Netflix:       54.246.232.87
```

Without IP addresses, data has no way to know where to go. When your laptop sends a request to Google, it addresses the request to `142.250.80.46` — and every router along the way reads that address to forward it in the right direction.

---

## The Postal Address Analogy

An IP address works like a mailing address:

```
Mailing address:
  123 Main Street, Apartment 4, New York, NY 10001, USA

IP address:
  192.168.1.5

Both answer the same question: "Where should this be delivered?"
```

When you send a letter:
1. You write the destination address on the envelope
2. The post office reads the country → routes to the right country
3. Then reads the city → routes to the right city
4. Then reads the street → routes to the right street
5. Then reads the number → delivers to the right house

When your computer sends data:
1. It writes the destination IP on the packet
2. Your router reads the IP → decides which network to send to
3. Each router along the way reads the IP → forwards it closer
4. The final router delivers it to the right device

---

## IPv4 — The Standard Format

**IPv4** (Internet Protocol version 4) is the most common format. It looks like this:

```
192.168.1.5
```

### Structure

An IPv4 address is **four numbers separated by dots**. Each number ranges from **0 to 255**.

```
┌─────┐ . ┌─────┐ . ┌─────┐ . ┌─────┐
│ 192 │   │ 168 │   │  1  │   │  5  │
└─────┘   └─────┘   └─────┘   └─────┘
 0-255     0-255     0-255     0-255

Each number is called an "octet" (because it's 8 bits — more on that below)
```

### Valid IP Addresses

```
✓ 192.168.1.5      (typical home device)
✓ 10.0.0.1         (typical office/AWS)
✓ 172.16.0.100     (another private range)
✓ 142.250.80.46    (Google — public IP)
✓ 0.0.0.0          (special — means "all addresses")
✓ 255.255.255.255  (special — broadcast)

✗ 192.168.1.256    (INVALID — 256 > 255)
✗ 192.168.1        (INVALID — only 3 numbers)
✗ 192.168.1.5.6    (INVALID — 5 numbers)
```

---

## How to Read an IPv4 Address

Every IP address has two parts: the **network part** and the **host part**.

```
IP Address:  192.168.1.5

Network Part:  192.168.1   ← identifies WHICH NETWORK (like city + street)
Host Part:     5            ← identifies WHICH DEVICE on that network (like house number)
```

All devices on the same network share the same **network part** but have different **host parts**:

```
Your home network: 192.168.1.x
  Laptop:    192.168.1.5    ← same network (192.168.1), different host (5)
  Phone:     192.168.1.8    ← same network (192.168.1), different host (8)
  Smart TV:  192.168.1.12   ← same network (192.168.1), different host (12)
  Router:    192.168.1.1    ← same network (192.168.1), different host (1)
```

The exact split between network and host parts is determined by the **subnet mask** — you'll learn that in concept 04 (Subnets & CIDR).

---

## Binary — How Computers Actually See IP Addresses

You see `192.168.1.5`. Your computer sees:

```
11000000.10101000.00000001.00000101
```

Each of the four numbers (octets) is stored as **8 bits** (binary digits — 0s and 1s):

```
192     = 11000000    (in binary)
168     = 10101000
1       = 00000001
5       = 00000101

Total: 32 bits (that's why IPv4 is called a "32-bit address")
```

### How Binary Works (Quick Primer)

Binary is counting with only two digits: 0 and 1 (instead of 0-9 like normal decimal).

Each position is a **power of 2**:

```
Position:  128  64  32  16   8   4   2   1

Example: 192 in binary
  128 + 64 = 192
  128  64  32  16   8   4   2   1
   1    1   0   0   0   0   0   0  = 11000000

Example: 5 in binary
  4 + 1 = 5
  128  64  32  16   8   4   2   1
   0    0   0   0   0   1   0   1  = 00000101
```

### Why Binary Matters

You don't need to convert binary in your head daily. But understanding that IP addresses are 32-bit numbers explains:
- Why each octet goes from 0 to 255 (8 bits → 2⁸ = 256 values: 0 to 255)
- Why subnet masks look the way they do (concept 04)
- Why CIDR notation uses /24, /16, etc. (number of bits for the network part)

---

## How Many IPv4 Addresses Exist

Each address is 32 bits. That means:

```
2³² = 4,294,967,296 addresses (about 4.3 billion)
```

That sounds like a lot, but there are over 8 billion people on Earth, plus billions of devices (phones, laptops, servers, IoT devices). **We've run out of IPv4 addresses.** This is why private addresses (below) and IPv6 (later) exist.

---

## Public vs Private IP Addresses

This is one of the most important concepts in networking.

### Public IP Address

A **public IP** is unique across the **entire internet**. No two devices on the internet have the same public IP at the same time.

```
Google:     142.250.80.46    ← public, unique worldwide
Facebook:   157.240.1.35     ← public, unique worldwide
Your home:  73.162.45.201    ← public, assigned by your ISP
```

Public IPs are like **phone numbers** — each one is globally unique.

### Private IP Address

A **private IP** is only used within a **local network** (LAN). It's NOT unique globally — millions of people use the same private IPs at the same time.

```
Your home:   192.168.1.5     ← your laptop
Your neighbor: 192.168.1.5   ← their laptop (same IP, different network!)
An office:   192.168.1.5     ← some printer there

This is fine because private IPs never leave the local network.
They're reused in every home, office, and data center in the world.
```

Private IPs are like **apartment numbers** — "Apartment 5" exists in thousands of buildings, but within each building it's unique.

### Why We Need Private IPs

We ran out of public IPs (only 4.3 billion). But there are billions of devices. Solution:

```
Your Home (public IP: 73.162.45.201):
  Laptop:     192.168.1.5    ← private
  Phone:      192.168.1.8    ← private
  TV:         192.168.1.12   ← private
  Printer:    192.168.1.20   ← private

All 4 devices share ONE public IP (73.162.45.201).
Your router translates between private and public (using NAT — concept 09).
```

Instead of needing 4 public IPs, you need just 1. Multiply that by every home and office in the world — huge savings.

---

## Private IP Ranges — Memorize These Three

The internet standards body (IANA) reserved three ranges of IP addresses exclusively for private networks. These will NEVER be used as public IPs:

| Range | From | To | How Many |
|-------|------|----|----------|
| **10.0.0.0/8** | 10.0.0.0 | 10.255.255.255 | 16,777,216 addresses |
| **172.16.0.0/12** | 172.16.0.0 | 172.31.255.255 | 1,048,576 addresses |
| **192.168.0.0/16** | 192.168.0.0 | 192.168.255.255 | 65,536 addresses |

```
If you see an IP address starting with:
  10.x.x.x       → private (used in AWS VPCs, large offices)
  172.16.x.x to
  172.31.x.x     → private (used in AWS, Docker)
  192.168.x.x    → private (used in home networks)
  
  Anything else   → public (routable on the internet)
```

### Which Range to Use?

| Range | Typical Use |
|-------|------------|
| **192.168.x.x** | Home routers (your Wi-Fi is probably 192.168.1.x or 192.168.0.x) |
| **10.x.x.x** | Large networks — AWS VPCs, corporate offices (16 million addresses!) |
| **172.16-31.x.x** | Medium networks — Docker containers, some cloud setups |

---

## How Your Home Network Uses Both

```
                          The Internet
                              │
                      Public IP: 73.162.45.201
                              │
                       ┌──────┴──────┐
                       │   Router    │ ← has BOTH a public IP and a private IP
                       │             │
                       │ Public:  73.162.45.201 (internet side)
                       │ Private: 192.168.1.1   (home side)
                       └──┬──┬──┬───┘
                          │  │  │
              ┌───────────┘  │  └───────────┐
              │              │              │
         ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
         │ Laptop  │   │ Phone   │   │  TV     │
         │ 192.168.│   │ 192.168.│   │ 192.168.│
         │  1.5    │   │  1.8    │   │  1.12   │
         └─────────┘   └─────────┘   └─────────┘
              ↑              ↑              ↑
         Private IPs (only visible inside your home network)
```

When your laptop (192.168.1.5) opens google.com:
1. Laptop sends request to router: "from 192.168.1.5, going to 142.250.80.46"
2. Router does NAT (concept 09): replaces 192.168.1.5 with 73.162.45.201
3. Google sees: "request from 73.162.45.201" (your public IP — it doesn't know about 192.168.1.5)
4. Google sends response to 73.162.45.201
5. Your router receives it, remembers which device asked, sends it to 192.168.1.5

---

## Static vs Dynamic IP Addresses

### Dynamic IP (Most Common)

Your device gets an IP address **automatically** when it connects to a network, and that IP might change next time.

```
Monday:    Laptop connects to Wi-Fi → gets 192.168.1.5
Tuesday:   Laptop connects to Wi-Fi → gets 192.168.1.9 (different!)
Wednesday: Laptop connects to Wi-Fi → gets 192.168.1.5 (back to old one, maybe)
```

This is fine for normal devices — they don't need a fixed address.

### Static IP (For Servers)

A **static IP** is manually assigned and never changes.

```
Web server:     always 10.0.1.50
Database:       always 10.0.2.100
DNS server:     always 10.0.0.2
```

Servers need static IPs because other devices need to know where to find them. If a web server's IP changed every day, nobody could connect to it.

---

## DHCP — How Devices Get IP Addresses Automatically

**DHCP** (Dynamic Host Configuration Protocol) is what gives your device an IP address when you connect to a network.

```
1. Your laptop connects to Wi-Fi
2. Laptop broadcasts: "I'm new here! Can someone give me an IP address?"
3. DHCP server (usually your router) responds:
   "Here, use 192.168.1.5"
   "Your subnet mask is 255.255.255.0"
   "Your default gateway (router) is 192.168.1.1"
   "Your DNS server is 8.8.8.8"
   "This IP is yours for 24 hours (lease time)"

4. Laptop: "Thanks!" → now configured and can talk on the network
```

### What DHCP Provides

| Setting | What It Is | Example |
|---------|-----------|---------|
| **IP address** | Your address on this network | 192.168.1.5 |
| **Subnet mask** | Which part is network vs host | 255.255.255.0 |
| **Default gateway** | Router's IP (exit from the LAN) | 192.168.1.1 |
| **DNS server** | Where to look up domain names | 8.8.8.8 |
| **Lease time** | How long you keep this IP | 24 hours |

Without DHCP, you'd have to manually type in all these settings on every device. Imagine doing that for your phone, laptop, TV, and smart speaker every time you restart them.

---

## Loopback Address — 127.0.0.1

**127.0.0.1** is a special IP address that means **"this computer itself."**

```
When you type 127.0.0.1 or "localhost" in your browser:
  → your computer talks to ITSELF
  → the data never leaves your machine
  → no network involved
```

Developers use this constantly:

```
Running a web server on your laptop:
  Server starts on 127.0.0.1:8080
  Open browser → http://127.0.0.1:8080
  → you're accessing the server running on your own machine
  → also works with "localhost" (same thing)
```

---

## Special IP Addresses

| Address | Meaning | Use |
|---------|---------|-----|
| **127.0.0.1** | Loopback (yourself) | Testing, local development |
| **0.0.0.0** | "All interfaces" or "any address" | Server listening on all IPs |
| **255.255.255.255** | Broadcast | Send to all devices on the network |
| **169.254.x.x** | Link-local (APIPA) | Device couldn't get DHCP → assigned itself |
| **224.0.0.0 – 239.255.255.255** | Multicast | Send to a group of devices |

If you see `169.254.x.x` on your device, it means DHCP failed — your device couldn't get a proper IP address.

---

## IPv6 — The Future (and Present)

IPv4 gives us ~4.3 billion addresses. That's not enough. **IPv6** was created to solve this.

### IPv6 Format

```
IPv4:  192.168.1.5                              (32 bits)
IPv6:  2001:0db8:85a3:0000:0000:8a2e:0370:7334  (128 bits)
```

### How Many IPv6 Addresses?

```
IPv4: 2³²  = ~4.3 billion
IPv6: 2¹²⁸ = ~340 undecillion (340,000,000,000,000,000,000,000,000,000,000,000,000)

That's enough to give every atom on Earth multiple addresses.
```

### Why IPv6 Hasn't Fully Replaced IPv4

- Most of the internet still runs IPv4
- NAT (concept 09) and private IPs extended IPv4's life
- Migration is slow — both need to work during transition
- AWS supports both, but most VPCs use IPv4

### IPv6 Simplified Notation

```
Full:      2001:0db8:0000:0000:0000:0000:0000:0001
Shortened: 2001:db8::1     (leading zeros removed, consecutive zeros = ::)
```

For this course, we'll focus on IPv4 — it's what you'll work with in AWS daily.

---

## Finding Your Own IP Address

### Your Private IP (on your LAN)

```bash
# Linux / Mac:
ip addr show        # or: ifconfig
# Look for "inet 192.168.x.x" or "inet 10.x.x.x"

# Windows:
ipconfig
# Look for "IPv4 Address: 192.168.x.x"
```

### Your Public IP (what the internet sees)

```bash
# From terminal:
curl ifconfig.me

# Or just Google: "what is my IP"
```

Your private IP and public IP are different:
```
Private: 192.168.1.5     ← what your laptop uses on your home LAN
Public:  73.162.45.201   ← what Google, Netflix, etc. see
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **IP address** | A number that identifies a device on a network |
| **IPv4** | 32-bit address, 4 numbers separated by dots (192.168.1.5) |
| **Octet** | Each of the 4 numbers (0-255) in an IPv4 address |
| **Public IP** | Globally unique, used on the internet |
| **Private IP** | Only unique within a LAN, reused worldwide |
| **10.x.x.x** | Private range (large networks, AWS) |
| **172.16-31.x.x** | Private range (medium networks) |
| **192.168.x.x** | Private range (home networks) |
| **Static IP** | Manually assigned, never changes (servers) |
| **Dynamic IP** | Automatically assigned by DHCP, can change |
| **DHCP** | Protocol that automatically gives devices an IP |
| **127.0.0.1** | Loopback — "this machine" / localhost |
| **IPv6** | 128-bit address — the replacement for IPv4 |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| Private IPs (10.x.x.x) | Every AWS **VPC** uses the 10.0.0.0/8 range (or 172.16.x.x) |
| Public IPs | **Elastic IPs** — static public IPs you attach to instances |
| DHCP | AWS DHCP automatically assigns private IPs to EC2 instances |
| Static IPs | Servers in AWS get fixed private IPs within their subnet |
| IPv4 | Most VPCs and services use IPv4 |
| No public IP? | Instances in **private subnets** have only private IPs — they use **NAT** to reach internet |

---

> **Previous:** [02 — Networking Layers](02-networking-layers.md)
> **Next:** [04 — Subnets & CIDR](04-subnets-and-cidr.md)
