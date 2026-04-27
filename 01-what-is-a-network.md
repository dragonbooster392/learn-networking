# 01 — What Is a Network

Everything starts here. Before AWS, before the internet, before anything — you need to understand what a network actually is.

---

## Table of Contents

- [The Simplest Network in the World](#the-simplest-network-in-the-world)
- [Why Do We Need Networks](#why-do-we-need-networks)
- [What Connects Computers Together](#what-connects-computers-together)
- [LAN — Local Area Network](#lan-local-area-network)
- [WAN — Wide Area Network](#wan-wide-area-network)
- [The Internet — A Network of Networks](#the-internet-a-network-of-networks)
- [Client and Server](#client-and-server)
- [What Is a Protocol](#what-is-a-protocol)
- [MAC Address — The Hardware Address](#mac-address-the-hardware-address)
- [Network Devices — Switch, Router, Modem](#network-devices-switch-router-modem)
- [Wired vs Wireless](#wired-vs-wireless)
- [Bandwidth and Latency](#bandwidth-and-latency)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## The Simplest Network in the World

Take two computers. Connect them with a cable. That's a network.

```
  Computer A ────── cable ────── Computer B
```

That's it. A **network** is two or more devices connected together so they can share information.

When you send a file from your phone to your friend's phone over Wi-Fi, that's networking. When you open a website, your computer is talking to another computer somewhere far away — that's also networking.

---

## Why Do We Need Networks

Before networks existed, if you wanted to give a file to someone, you had to copy it to a floppy disk (or USB drive) and physically walk it to them. People actually called this "sneakernet" — because you used your sneakers to transfer data.

```
Before networks:
  Computer A → copy to disk → walk to desk → insert disk → Computer B
  (slow, painful, doesn't scale)

With networks:
  Computer A → send over network → Computer B
  (instant, works across the world)
```

Networks let us:

| Need | How a Network Solves It |
|------|------------------------|
| **Share files** | Send documents, photos, videos between computers |
| **Share devices** | One printer shared by 50 people in an office |
| **Communicate** | Email, video calls, chat |
| **Access information** | Websites, databases, APIs |
| **Work together** | Multiple people editing the same document |

---

## What Connects Computers Together

Computers need **two things** to talk to each other:

### 1. A Physical Connection (or Wireless)

Something that carries the data from one device to another:

```
Wired:
  Computer ──── Ethernet Cable ──── Computer
  (electrical signals travel through copper wire)

  Computer ──── Fiber Optic Cable ──── Computer
  (light pulses travel through glass fiber — very fast, long distances)

Wireless:
  Computer ~~~~ Wi-Fi ~~~~ Computer
  (radio waves travel through the air)
```

### 2. A Set of Rules (Protocol)

Both computers need to agree on HOW to talk. Just like two people need to speak the same language:

```
English speaker: "Hello, how are you?"
English speaker: "I'm fine, thanks!"
(communication works — same language)

English speaker: "Hello, how are you?"
Japanese speaker: "すみません、英語がわかりません"
(communication fails — different language)
```

In networking, these agreed-upon rules are called **protocols**. We'll cover them in detail, but for now: a protocol is just a set of rules that both sides follow so they can understand each other.

---

## LAN — Local Area Network

A **LAN** is a network in a small area — your home, an office, a building, a school.

### Your Home Network

```
                        ┌────────────────┐
                        │   The Internet │
                        └───────┬────────┘
                                │
                        ┌───────┴────────┐
                        │   Your Router  │
                        │ (the box from  │
                        │  your ISP)     │
                        └───────┬────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                  │
        ┌─────┴─────┐   ┌──────┴──────┐   ┌──────┴──────┐
        │  Your     │   │  Your       │   │  Your       │
        │  Laptop   │   │  Phone      │   │  Smart TV   │
        └───────────┘   └─────────────┘   └─────────────┘

        ←──────────── Your Home LAN ──────────────→
```

Everything behind your router is your LAN. Your laptop, phone, smart TV, smart speaker — they're all on the same local network. They can talk to each other directly:

- Your phone can cast a video to your TV (both on the LAN)
- Your laptop can print to the wireless printer (both on the LAN)
- Your phone can send a file to your laptop via AirDrop/nearby share (both on the LAN)

### An Office LAN

```
                                ┌────────────────┐
                                │   The Internet │
                                └───────┬────────┘
                                        │
                                ┌───────┴────────┐
                                │   Company      │
                                │   Router       │
                                └───────┬────────┘
                                        │
                    ┌───────────────────┤
                    │                   │
             ┌──────┴──────┐     ┌──────┴──────┐
             │  Switch     │     │  Switch     │
             │  (Floor 1)  │     │  (Floor 2)  │
             └──────┬──────┘     └──────┬──────┘
                    │                   │
         ┌─────┬───┴───┬─────┐  ┌─────┬┴──────┬─────┐
         │     │       │     │  │     │       │     │
        PC1   PC2    PC3  Printer PC4  PC5   PC6  Printer
```

An office LAN might have hundreds of computers, printers, and phones — all connected by cables and switches within the building.

### Key Facts About LANs

| Property | LAN |
|----------|-----|
| **Size** | Small — one room, one floor, one building |
| **Speed** | Very fast (1 Gbps or 10 Gbps is common) |
| **Who owns it** | You own and control it |
| **Cost** | Just the cost of cables and a switch/router |
| **Example** | Your home Wi-Fi, office network |

---

## WAN — Wide Area Network

A **WAN** connects LANs that are far apart — different cities, different countries, different continents.

```
┌───────────────────┐                          ┌───────────────────┐
│  New York Office  │                          │  London Office    │
│  (LAN)            │                          │  (LAN)            │
│                   │                          │                   │
│  PC1  PC2  PC3    │ ──── WAN Connection ──── │  PC4  PC5  PC6    │
│                   │     (leased line,        │                   │
│                   │      VPN, etc.)          │                   │
└───────────────────┘                          └───────────────────┘
```

A company with offices in New York and London needs a WAN to connect them. Without it, New York employees can't access files stored in London.

### Key Facts About WANs

| Property | WAN |
|----------|-----|
| **Size** | Large — cities, countries, continents |
| **Speed** | Slower than LAN (depends on connection) |
| **Who owns it** | Usually a telecom provider (AT&T, BT, etc.) |
| **Cost** | Expensive (you pay for the long-distance connection) |
| **Example** | A company connecting its offices worldwide |

---

## The Internet — A Network of Networks

The internet is NOT one giant network. It's **thousands of networks connected together**.

```
┌──────────────────────────────────────────────────────────────────┐
│                         THE INTERNET                              │
│                                                                   │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│   │ Google's │    │ Amazon's │    │ Netflix  │                  │
│   │ Network  │    │ Network  │    │ Network  │                  │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘                  │
│        │               │               │                         │
│        └───────────────┼───────────────┘                         │
│                        │                                          │
│              ┌─────────┴─────────┐                               │
│              │  Internet Backbone │                               │
│              │  (high-speed fiber │                               │
│              │   across oceans)   │                               │
│              └─────────┬─────────┘                               │
│                        │                                          │
│        ┌───────────────┼───────────────┐                         │
│        │               │               │                         │
│   ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐                  │
│   │  ISP 1   │    │  ISP 2   │    │  ISP 3   │                  │
│   │(Comcast) │    │ (AT&T)   │    │ (Jio)    │                  │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘                  │
│        │               │               │                         │
│   Your Home       Office LAN     Someone's Phone                 │
└──────────────────────────────────────────────────────────────────┘
```

### How Your Home Connects to the Internet

1. Your devices connect to your **home router** (that's your LAN)
2. Your router connects to your **ISP** (Internet Service Provider — Comcast, AT&T, Jio, etc.)
3. Your ISP connects to the **internet backbone** (massive fiber optic cables)
4. The backbone connects to **other ISPs and networks** (Google, Amazon, etc.)

When you type `google.com` in your browser:

```
Your laptop
  → your router
    → your ISP's network
      → internet backbone
        → Google's network
          → Google's server
            → sends the webpage back through the same path
```

### ISP — Internet Service Provider

Your ISP is the company that gives you internet access. You pay them monthly, and they connect your home to the global internet.

```
You ←── cable/fiber/DSL ──→ ISP ←── backbone ──→ Rest of Internet
```

Without an ISP, your home LAN is just an island — your devices can talk to each other, but they can't reach the outside world.

---

## Client and Server

These two words come up constantly in networking. They're simple:

- **Client**: The device that **asks** for something
- **Server**: The device that **provides** something

```
Your Browser (CLIENT)                   Google (SERVER)
     │                                       │
     │  "Give me the google.com page"        │
     │ ─────────────────────────────────────→ │
     │                                       │
     │  "Here's the page"                    │
     │ ←───────────────────────────────────── │
     │                                       │
```

### Important: It's About the Role, Not the Machine

A "server" is not a special magical computer. It's just a regular computer that is **serving** something to others.

- Your laptop can be a server (if you share a folder on it, other devices are clients accessing it)
- The same machine can be both a client AND a server at the same time

```
Your Computer:
  - CLIENT when you open google.com (asking Google for a page)
  - SERVER when your phone accesses a shared folder on it (providing files)
```

### Common Client-Server Examples

| Client | Server | What's Being Shared |
|--------|--------|-------------------|
| Your browser | Google's web server | Web pages |
| Your email app | Gmail's mail server | Emails |
| Your phone's YouTube app | YouTube's video server | Videos |
| Your game | Game company's server | Game data, multiplayer |
| kubectl on your laptop | Kubernetes API server | Cluster state |

---

## What Is a Protocol

A **protocol** is an agreed set of rules for communication. Both sides must follow the same rules or they can't understand each other.

### Real-World Example: Ordering Food at a Restaurant

There's an unspoken protocol:

```
1. You sit down
2. Waiter brings menu
3. You choose items
4. You tell the waiter your order
5. Kitchen prepares food
6. Waiter brings food
7. You eat
8. You ask for the bill
9. You pay
10. You leave
```

If you walked into a restaurant and started with "Here's my money, give me the bill" — the waiter would be confused. You broke the protocol.

### Network Protocols Work the Same Way

```
HTTP protocol (loading a web page):
1. Client connects to server
2. Client sends request: "GET /index.html"
3. Server processes request
4. Server sends response: "200 OK" + the page content
5. Connection closes

If the client sent "GIVE ME PAGE" instead of "GET /index.html"
  → the server wouldn't understand → protocol broken
```

### Common Network Protocols

| Protocol | What It's For | Everyday Example |
|----------|--------------|-----------------|
| **HTTP** | Loading web pages (unencrypted) | Typing http://example.com |
| **HTTPS** | Loading web pages (encrypted, secure) | Typing https://google.com (the lock icon) |
| **TCP** | Reliable data delivery | Makes sure every piece of data arrives |
| **UDP** | Fast data delivery (some loss okay) | Video calls, online gaming |
| **DNS** | Translating names to IP addresses | Turning "google.com" into 142.250.80.46 |
| **SSH** | Secure remote access to a computer | Logging into a Linux server |
| **FTP** | Transferring files | Uploading files to a web server |
| **SMTP** | Sending emails | Your email app sending a message |

Don't worry about memorizing these. We'll cover each one in detail in later files.

---

## MAC Address — The Hardware Address

Every network device (your laptop's Wi-Fi chip, your phone's Wi-Fi chip, your Ethernet port) has a **MAC address** burned into it at the factory. It's a unique ID — like a serial number.

```
Format: six groups of two hex characters, separated by colons
Example: A4:83:E7:2B:5F:01

Your Laptop's Wi-Fi:     A4:83:E7:2B:5F:01   (unique — no other device has this)
Your Phone's Wi-Fi:      B8:27:EB:3A:44:9C   (unique)
Your Smart TV:           DC:A6:32:11:22:33   (unique)
```

### MAC vs IP Address (Preview)

You'll learn about IP addresses in the next concept, but here's the difference in simple terms:

| | MAC Address | IP Address |
|---|-----------|-----------|
| **What** | Hardware ID — burned in at factory | Network address — assigned by the network |
| **Changes?** | Never (permanent) | Yes (changes when you connect to a different network) |
| **Analogy** | Your birth name | Your mailing address |
| **Scope** | Only used within your local network (LAN) | Used across the entire internet |
| **Example** | A4:83:E7:2B:5F:01 | 192.168.1.5 |

Your MAC address is like your **name** — it never changes.
Your IP address is like your **home address** — it changes if you move.

When you move from home Wi-Fi to a coffee shop Wi-Fi:
- Your MAC address stays the same (your device didn't change)
- Your IP address changes (you're on a different network now)

---

## Network Devices — Switch, Router, Modem

### Switch — Connects Devices Within a LAN

A switch is a box with many Ethernet ports. It connects multiple devices on the same LAN and knows which device is on which port (using MAC addresses).

```
                    ┌──────────────────────┐
  Computer A ─────── │ Port 1               │
  Computer B ─────── │ Port 2    SWITCH     │
  Computer C ─────── │ Port 3               │
  Printer    ─────── │ Port 4               │
                    └──────────────────────┘

When Computer A sends data to the Printer:
  Switch looks at the destination MAC address
  → "That's the Printer, which is on Port 4"
  → sends data ONLY to Port 4 (not to B or C)
```

**Analogy:** A switch is like a **post office inside a building**. It knows which apartment (port) each person (device) lives in, and delivers mail only to the right apartment.

### Router — Connects Different Networks Together

A router connects your LAN to other networks (like the internet). It makes decisions about where to send data based on IP addresses.

```
  Your Home LAN                    The Internet
  ┌─────────────┐                  ┌─────────────┐
  │ Laptop      │                  │ Google      │
  │ Phone       │ ── Router ────── │ Netflix     │
  │ TV          │                  │ YouTube     │
  └─────────────┘                  └─────────────┘
       LAN                              WAN
  (private network)             (public network)
```

**Analogy:** A router is like a **border checkpoint**. It decides which traffic can leave your country (LAN) and enter another country (internet), and where incoming traffic should go.

### Key Difference

| Device | Connects | Uses | Analogy |
|--------|----------|------|---------|
| **Switch** | Devices within ONE network | MAC addresses | Post office inside a building |
| **Router** | DIFFERENT networks together | IP addresses | Border checkpoint between countries |

### Modem — Translates Signals

A modem converts your digital data into the format your ISP's line uses (cable, DSL, fiber) and vice versa.

```
Your devices → Router → Modem → ISP cable → ISP → Internet

Many home routers today have the modem built in (one box does both).
```

### Your Home Network — Complete Picture

```
                            Internet
                               │
                        ┌──────┴──────┐
                        │    Modem    │  ← converts signal format
                        └──────┬──────┘
                               │
                        ┌──────┴──────┐
                        │   Router    │  ← connects your LAN to internet
                        │             │  ← also has Wi-Fi built in
                        └──┬──┬──┬──┬─┘
                           │  │  │  │
  Ethernet cable ──────────┘  │  │  └────── Ethernet cable
  to Desktop PC               │  │          to Smart TV
                              │  │
                     Wi-Fi ~~~┘  └~~~ Wi-Fi
                     to Phone        to Laptop
```

In most homes today, the modem, router, switch, and Wi-Fi access point are **all in one box** — the thing your ISP gave you.

---

## Wired vs Wireless

### Wired (Ethernet)

```
Computer ────── Ethernet Cable ────── Switch/Router
```

| Property | Wired |
|----------|-------|
| **Speed** | Very fast (1 Gbps, 10 Gbps, 25 Gbps) |
| **Reliability** | Very reliable — cable doesn't drop |
| **Latency** | Very low — nearly instant |
| **Mobility** | None — you're tied to the cable |
| **Use case** | Servers, desktops, data centers, offices |

### Wireless (Wi-Fi)

```
Computer ~~~~ radio waves ~~~~ Wi-Fi Access Point/Router
```

| Property | Wireless |
|----------|----------|
| **Speed** | Slower than wired (but getting faster) |
| **Reliability** | Can drop — walls, distance, interference |
| **Latency** | Higher than wired |
| **Mobility** | Full — move anywhere in range |
| **Use case** | Laptops, phones, tablets, IoT devices |

### In Data Centers and AWS

AWS data centers use **wired connections** — Ethernet cables and fiber optic cables connecting thousands of servers at extremely high speeds. Nobody uses Wi-Fi in a data center.

```
Data Center:
  Server ── 25 Gbps cable ── Switch ── 100 Gbps fiber ── Router ── Internet

Your Home:
  Laptop ~~~~ Wi-Fi (300 Mbps) ~~~~ Router ── cable ── ISP
```

---

## Bandwidth and Latency

These two words describe how fast a network feels. They mean different things:

### Bandwidth — How Much Data Per Second

Bandwidth is the **width of the pipe** — how much data can flow through at once.

```
Low Bandwidth (dial-up, old):     ──────>  (thin pipe, little water)
High Bandwidth (fiber, modern):   ══════>  (fat pipe, lots of water)
```

Measured in: **bits per second** (bps)
- 1 Kbps = 1,000 bits per second (ancient dial-up)
- 1 Mbps = 1,000,000 bits per second (basic home internet)
- 1 Gbps = 1,000,000,000 bits per second (fast home/office internet)
- 10 Gbps = data center speed
- 100 Gbps = backbone / AWS internal network

### Latency — How Long Before Data Arrives

Latency is the **delay** — how long it takes for data to travel from A to B.

```
Low latency:   A ──→ B  (5ms — feels instant)
High latency:  A ─────────────→ B  (200ms — noticeable delay)
```

Measured in: **milliseconds (ms)**

| Latency | Feels Like |
|---------|-----------|
| 1-5ms | Same data center — instant |
| 10-30ms | Same city — still instant |
| 50-100ms | Same continent — barely noticeable |
| 150-300ms | Across the ocean — slight delay |
| 500ms+ | Satellite — very noticeable lag |

### The Highway Analogy

```
Bandwidth = number of lanes on the highway
  (more lanes → more cars at the same time → more data per second)

Latency = length of the highway
  (longer highway → takes longer for each car to arrive)
```

You can have **high bandwidth but high latency** (satellite internet: fat pipe, but data takes long to arrive because it goes to space and back).

You can have **low bandwidth but low latency** (old DSL in the same city: thin pipe, but data arrives quickly because the distance is short).

---

## Summary

| Term | What It Means |
|------|--------------|
| **Network** | Two or more devices connected to share information |
| **LAN** | Local Area Network — small area (home, office) |
| **WAN** | Wide Area Network — connects LANs far apart |
| **Internet** | A network of networks, connected globally |
| **Client** | Device that asks for something |
| **Server** | Device that provides something |
| **Protocol** | Rules both sides agree on for communication |
| **MAC Address** | Permanent hardware ID on every network device |
| **Switch** | Connects devices within a LAN (uses MAC addresses) |
| **Router** | Connects different networks (uses IP addresses) |
| **Modem** | Converts signals between your network and ISP |
| **Bandwidth** | How much data per second (pipe width) |
| **Latency** | How long data takes to arrive (delay) |

---

## How This Connects to AWS

Everything in AWS is networking:

| This Concept | In AWS |
|-------------|--------|
| LAN | A **VPC** (Virtual Private Cloud) is your private LAN in the cloud |
| Router | AWS manages routers for your VPC — you configure **route tables** |
| Switch | AWS handles switching — you don't see physical switches |
| Internet connection | An **Internet Gateway** connects your VPC to the internet |
| Client-Server | Your app pods (server) and users (client) |
| Bandwidth | AWS network: 25 Gbps per instance, 100 Gbps backbone |
| Latency | Why AWS has **regions** — put servers near your users |

You don't need to understand these AWS terms yet. Just know that every AWS networking concept maps directly back to these fundamentals.

---

> **Next:** [02 — Networking Layers (OSI & TCP/IP)](02-networking-layers.md)
