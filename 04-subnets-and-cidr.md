# 04 — Subnets & CIDR

How networks are divided into smaller pieces, what /24 actually means, subnet masks explained step by step, and why this is the single most important concept for understanding AWS VPCs.

**Prerequisites:** Concept 03 (IP addresses, public vs private, binary basics).

---

## Table of Contents

- [Why Split a Network Into Pieces](#why-split-a-network-into-pieces)
- [What Is a Subnet](#what-is-a-subnet)
- [The Apartment Building Analogy](#the-apartment-building-analogy)
- [Subnet Mask — The Dividing Line](#subnet-mask-the-dividing-line)
- [CIDR Notation — The Slash Number](#cidr-notation-the-slash-number)
- [Common CIDR Blocks — The Ones You'll Actually Use](#common-cidr-blocks-the-ones-youll-actually-use)
- [How to Calculate a CIDR Block](#how-to-calculate-a-cidr-block)
- [Worked Examples](#worked-examples)
- [Subnetting — Splitting a Network Into Smaller Subnets](#subnetting-splitting-a-network-into-smaller-subnets)
- [Network Address and Broadcast Address](#network-address-and-broadcast-address)
- [Can These Two Devices Talk Directly](#can-these-two-devices-talk-directly)
- [CIDR Cheat Sheet](#cidr-cheat-sheet)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## Why Split a Network Into Pieces

Imagine a company with 1,000 computers all on one big flat network:

```
One giant network:
  PC1, PC2, PC3, ... PC1000 — all sharing the same network

Problems:
  1. SLOW — every broadcast reaches all 1,000 devices
  2. INSECURE — any device can see traffic from any other device
  3. MESSY — no organization, no separation between departments
```

Now split it into smaller networks (subnets):

```
Subnet 1 (Engineering):   PC1-PC200
Subnet 2 (Sales):         PC201-PC400
Subnet 3 (HR):            PC401-PC500
Subnet 4 (Servers):       Server1-Server50

Benefits:
  1. FASTER — broadcasts stay within each subnet
  2. SECURE — you can put firewall rules between subnets
  3. ORGANIZED — each department has its own address range
```

---

## What Is a Subnet

A **subnet** (sub-network) is a portion of a larger network. It's a group of IP addresses that belong together.

```
Full network: 10.0.0.0 (big — millions of addresses)

Split into subnets:
  Subnet A: 10.0.1.0   → IPs from 10.0.1.0 to 10.0.1.255    (256 addresses)
  Subnet B: 10.0.2.0   → IPs from 10.0.2.0 to 10.0.2.255    (256 addresses)
  Subnet C: 10.0.3.0   → IPs from 10.0.3.0 to 10.0.3.255    (256 addresses)
```

Devices within the same subnet can talk to each other directly (like being on the same floor of a building). Devices in different subnets need a **router** to communicate (like going through the lobby to reach a different floor).

---

## The Apartment Building Analogy

```
Building Address: 10 Main Street         ← This is the "network"

  Floor 1 (Subnet 1): Apartments 101-110  ← Engineering
  Floor 2 (Subnet 2): Apartments 201-210  ← Sales
  Floor 3 (Subnet 3): Apartments 301-310  ← Servers

Rules:
  - People on the SAME floor can visit each other directly (same subnet)
  - To visit a DIFFERENT floor, you must take the elevator (router)
  - The building manager can block elevator access between floors (firewall)
```

---

## Subnet Mask — The Dividing Line

A **subnet mask** tells a device: "which part of the IP address is the network, and which part is the host?"

```
IP Address:   192.168.1.5
Subnet Mask:  255.255.255.0

The mask is like a ruler laid over the IP:
  Where the mask has 255 → that part is the NETWORK
  Where the mask has 0   → that part is the HOST

  IP:   192 . 168 . 1  .  5
  Mask: 255 . 255 . 255 .  0
        ─────────────────  ───
        NETWORK part       HOST part

Network: 192.168.1
Host:    5
```

### In Binary (How It Really Works)

```
IP:     11000000.10101000.00000001.00000101    (192.168.1.5)
Mask:   11111111.11111111.11111111.00000000    (255.255.255.0)
        ────────────────────────── ────────
        1s = Network part          0s = Host part
        (24 bits)                  (8 bits)
```

The subnet mask is a string of **1s followed by 0s** in binary:
- The **1s** mark the network portion
- The **0s** mark the host portion

This is the dividing line between "which network" and "which device on that network."

---

## CIDR Notation — The Slash Number

Instead of writing `255.255.255.0`, we use **CIDR notation**: a slash followed by how many bits are the network part.

```
255.255.255.0    = /24   (24 bits are 1s → 24 bits for network)
255.255.0.0      = /16   (16 bits are 1s → 16 bits for network)
255.0.0.0        = /8    (8 bits are 1s  → 8 bits for network)
```

### Reading CIDR

```
10.0.1.0/24

This means:
  Network:  10.0.1    (first 24 bits are the network address)
  Hosts:    .0 to .255 (last 8 bits are for devices)
  Number of addresses: 2⁸ = 256

192.168.0.0/16

This means:
  Network:  192.168   (first 16 bits are the network address)
  Hosts:    .0.0 to .255.255 (last 16 bits are for devices)
  Number of addresses: 2¹⁶ = 65,536

10.0.0.0/8

This means:
  Network:  10        (first 8 bits are the network address)
  Hosts:    .0.0.0 to .255.255.255 (last 24 bits are for devices)
  Number of addresses: 2²⁴ = 16,777,216
```

### The Rule

```
/X means:
  - X bits are for the network (locked, can't change)
  - (32 - X) bits are for hosts (these vary)
  - Number of addresses = 2^(32-X)

The BIGGER the number after /, the FEWER addresses:
  /8  → 16,777,216 addresses (huge network)
  /16 → 65,536 addresses     (large network)
  /24 → 256 addresses        (small network)
  /28 → 16 addresses         (tiny network)
  /32 → 1 address            (single host)
```

---

## Common CIDR Blocks — The Ones You'll Actually Use

| CIDR | Subnet Mask | Number of Addresses | Use Case |
|------|------------|-------------------|----------|
| **/8** | 255.0.0.0 | 16,777,216 | Massive network (entire 10.x.x.x range) |
| **/16** | 255.255.0.0 | 65,536 | Large VPC in AWS |
| **/20** | 255.255.240.0 | 4,096 | Large subnet |
| **/24** | 255.255.255.0 | 256 | Most common subnet size |
| **/25** | 255.255.255.128 | 128 | Medium subnet |
| **/26** | 255.255.255.192 | 64 | Small subnet |
| **/27** | 255.255.255.224 | 32 | Small subnet |
| **/28** | 255.255.255.240 | 16 | Tiny subnet (AWS minimum) |
| **/32** | 255.255.255.255 | 1 | Single IP address |

### The Most Important Ones to Remember

```
/16 → 65,536 IPs — used for VPC total range
/24 → 256 IPs    — used for most subnets
/28 → 16 IPs     — smallest AWS subnet
/32 → 1 IP       — single host (used in security rules: "allow only this one IP")
```

---

## How to Calculate a CIDR Block

### Step 1: How Many Addresses?

```
Formula: 2^(32 - CIDR number) = number of addresses

/24: 2^(32-24) = 2^8  = 256
/20: 2^(32-20) = 2^12 = 4,096
/16: 2^(32-16) = 2^16 = 65,536
/28: 2^(32-28) = 2^4  = 16
```

### Step 2: What's the Range?

```
10.0.1.0/24:
  Start: 10.0.1.0
  End:   10.0.1.255
  (last octet varies: 0-255 = 256 addresses)

10.0.0.0/16:
  Start: 10.0.0.0
  End:   10.0.255.255
  (last TWO octets vary: 0.0-255.255 = 65,536 addresses)

10.0.0.0/20:
  Start: 10.0.0.0
  End:   10.0.15.255
  (trickier — 4,096 addresses, spanning 16 blocks of .0-.255)
```

---

## Worked Examples

### Example 1: Home Network

```
Your router assigns: 192.168.1.0/24

Network:     192.168.1.0
First usable: 192.168.1.1  (usually the router)
Last usable:  192.168.1.254
Broadcast:    192.168.1.255
Total:        256 addresses (254 usable for devices)

Your devices: 192.168.1.2, 192.168.1.3, ... 192.168.1.254
```

### Example 2: AWS VPC

```
You create a VPC with CIDR: 10.0.0.0/16

That gives you: 10.0.0.0 to 10.0.255.255 (65,536 addresses)

Now you create subnets within this VPC:
  Public subnet:    10.0.1.0/24  → 10.0.1.0 to 10.0.1.255   (256 IPs)
  Private subnet A: 10.0.10.0/24 → 10.0.10.0 to 10.0.10.255 (256 IPs)
  Private subnet B: 10.0.11.0/24 → 10.0.11.0 to 10.0.11.255 (256 IPs)
  Data subnet:      10.0.20.0/24 → 10.0.20.0 to 10.0.20.255 (256 IPs)
```

### Example 3: Security Group Rule

```
"Allow SSH from only my office IP"

Your office public IP: 203.0.113.50
Rule: Allow TCP port 22 from 203.0.113.50/32

/32 means "this ONE specific IP address" — nobody else
```

### Example 4: Allow an Entire Subnet

```
"Allow traffic from the entire 10.0.1.x subnet"

Rule: Allow from 10.0.1.0/24

This allows: 10.0.1.0 through 10.0.1.255 (all 256 addresses)
```

---

## Subnetting — Splitting a Network Into Smaller Subnets

You have a /16 VPC and want to split it into subnets. Think of it as cutting a cake:

```
VPC: 10.0.0.0/16 (65,536 addresses — the whole cake)

Cut into /24 subnets (256 addresses each — slices):
  10.0.0.0/24    → 10.0.0.0 - 10.0.0.255
  10.0.1.0/24    → 10.0.1.0 - 10.0.1.255
  10.0.2.0/24    → 10.0.2.0 - 10.0.2.255
  ...
  10.0.255.0/24  → 10.0.255.0 - 10.0.255.255

Total: 256 subnets × 256 addresses = 65,536 ✓ (fills the whole VPC)
```

You can also make subnets of different sizes:

```
VPC: 10.0.0.0/16

  TGW Subnet:   10.0.0.0/28    → 16 addresses (tiny — just needs a few)
  App Subnet:   10.0.1.0/24    → 256 addresses (needs room for pods)
  Data Subnet:  10.0.2.0/24    → 256 addresses (databases)
  Spare:        10.0.3.0/24 through 10.0.255.0/24 (room to grow)
```

### Rules for Subnetting

1. Subnets **cannot overlap** (no IP can belong to two subnets)
2. Subnets must **fit within** the parent network
3. Subnet sizes must be **powers of 2** (16, 32, 64, 128, 256...)

```
VALID:
  VPC: 10.0.0.0/16
    Subnet A: 10.0.1.0/24  ✓ (fits within 10.0.0.0/16)
    Subnet B: 10.0.2.0/24  ✓ (doesn't overlap with A)

INVALID:
  VPC: 10.0.0.0/16
    Subnet A: 10.0.1.0/24     (10.0.1.0 - 10.0.1.255)
    Subnet B: 10.0.1.128/25   (10.0.1.128 - 10.0.1.255) ✗ OVERLAPS with A!
```

---

## Network Address and Broadcast Address

Every subnet has two special addresses that **cannot be assigned to devices**:

```
Subnet: 10.0.1.0/24

  10.0.1.0      ← NETWORK ADDRESS (identifies the subnet itself)
  10.0.1.1      ← first usable address
  10.0.1.2      ← usable
  ...
  10.0.1.254    ← last usable address
  10.0.1.255    ← BROADCAST ADDRESS (sends to all devices in subnet)

  Total: 256 addresses
  Usable: 254 (256 - 2)
```

In AWS, **5 addresses** are reserved per subnet (not 2):

```
AWS subnet: 10.0.1.0/24

  10.0.1.0    ← Network address
  10.0.1.1    ← Reserved by AWS (VPC router)
  10.0.1.2    ← Reserved by AWS (DNS)
  10.0.1.3    ← Reserved by AWS (future use)
  10.0.1.4    ← first usable
  ...
  10.0.1.254  ← last usable
  10.0.1.255  ← Broadcast address

  Usable in AWS: 256 - 5 = 251 addresses
```

---

## Can These Two Devices Talk Directly?

Devices in the **same subnet** can communicate directly (Layer 2, using MAC addresses).
Devices in **different subnets** need a router (Layer 3).

```
Same subnet? → Direct communication (fast)
Different subnet? → Must go through a router (extra hop)

Example:
  Device A: 10.0.1.50 /24  (subnet: 10.0.1.0/24)
  Device B: 10.0.1.100/24  (subnet: 10.0.1.0/24)
  → SAME subnet → talk directly ✓

  Device A: 10.0.1.50 /24  (subnet: 10.0.1.0/24)
  Device C: 10.0.2.50 /24  (subnet: 10.0.2.0/24)
  → DIFFERENT subnets → need router ✓
```

---

## CIDR Cheat Sheet

Quick reference for the most common sizes:

```
/8   → 16,777,216 addresses  │ 255.0.0.0       │ x.0.0.0 - x.255.255.255
/12  →  1,048,576 addresses  │ 255.240.0.0     │ 
/16  →     65,536 addresses  │ 255.255.0.0     │ x.x.0.0 - x.x.255.255
/17  →     32,768 addresses  │ 255.255.128.0   │
/18  →     16,384 addresses  │ 255.255.192.0   │
/19  →      8,192 addresses  │ 255.255.224.0   │
/20  →      4,096 addresses  │ 255.255.240.0   │
/21  →      2,048 addresses  │ 255.255.248.0   │
/22  →      1,024 addresses  │ 255.255.252.0   │
/23  →        512 addresses  │ 255.255.254.0   │
/24  →        256 addresses  │ 255.255.255.0   │ x.x.x.0 - x.x.x.255
/25  →        128 addresses  │ 255.255.255.128 │
/26  →         64 addresses  │ 255.255.255.192 │
/27  →         32 addresses  │ 255.255.255.224 │
/28  →         16 addresses  │ 255.255.255.240 │ AWS minimum subnet
/29  →          8 addresses  │ 255.255.255.248 │
/30  →          4 addresses  │ 255.255.255.252 │ point-to-point links
/31  →          2 addresses  │ 255.255.255.254 │
/32  →          1 address    │ 255.255.255.255 │ single host
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **Subnet** | A portion of a larger network — a group of IP addresses |
| **Subnet mask** | Divides an IP into network part and host part |
| **CIDR** | Slash notation (/24, /16) — how many bits are the network part |
| **/24** | 256 addresses — most common subnet size |
| **/16** | 65,536 addresses — typical VPC size |
| **/32** | 1 address — single host |
| **Network address** | First IP in a subnet — identifies the subnet (not usable) |
| **Broadcast address** | Last IP in a subnet — sends to everyone (not usable) |
| **Same subnet** | Devices can talk directly |
| **Different subnet** | Devices need a router to talk |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| VPC CIDR | When you create a VPC, you pick a CIDR block (e.g., 10.0.0.0/16) |
| Subnets | You split the VPC CIDR into subnets (10.0.1.0/24, 10.0.2.0/24, etc.) |
| Public subnet | A subnet with a route to the Internet Gateway |
| Private subnet | A subnet with NO route to the Internet Gateway |
| 3-tier pattern | TGW subnet (/28) + App subnet (/24) + Data subnet (/24) |
| /28 minimum | AWS requires at least /28 (16 IPs) for a subnet |
| 5 reserved IPs | AWS reserves 5 IPs per subnet (.0, .1, .2, .3, and broadcast) |
| Security rules | Use CIDR in security groups: "allow 10.0.1.0/24" means "allow entire subnet" |
| /32 in rules | "allow 10.0.1.50/32" means "allow only this one IP" |

---

> **Previous:** [03 — IP Addresses](03-ip-addresses.md)
> **Next:** [05 — Ports & Protocols](05-ports-and-protocols.md)
