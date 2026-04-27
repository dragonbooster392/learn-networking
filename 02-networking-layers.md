# 02 — Networking Layers (OSI & TCP/IP)

How networking is organized into layers — and why understanding this makes everything in AWS (NLB, ALB, GWLB, security groups, WAF) immediately make sense.

**Prerequisites:** Concept 01 (what a network, protocol, switch, router are).

---

## Table of Contents

- [Why Layers Exist](#why-layers-exist)
- [The Real-World Analogy — Sending a Package](#the-real-world-analogy-sending-a-package)
- [The OSI Model — 7 Layers](#the-osi-model-7-layers)
- [Layer 1 — Physical](#layer-1-physical)
- [Layer 2 — Data Link](#layer-2-data-link)
- [Layer 3 — Network](#layer-3-network)
- [Layer 4 — Transport](#layer-4-transport)
- [Layer 5 — Session](#layer-5-session)
- [Layer 6 — Presentation](#layer-6-presentation)
- [Layer 7 — Application](#layer-7-application)
- [The TCP/IP Model — The Practical Version](#the-tcpip-model-the-practical-version)
- [OSI vs TCP/IP — Quick Comparison](#osi-vs-tcpip-quick-comparison)
- [What "Layer X" Means When People Say It](#what-layer-x-means-when-people-say-it)
- [How Data Moves Through the Layers](#how-data-moves-through-the-layers)
- [Encapsulation — Wrapping Data in Envelopes](#encapsulation-wrapping-data-in-envelopes)
- [Summary Table — All 7 Layers at a Glance](#summary-table-all-7-layers-at-a-glance)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## Why Layers Exist

Imagine if one single protocol had to handle EVERYTHING:
- The electrical signals on the cable
- Finding the right device on the local network
- Finding the right device across the internet
- Making sure data arrives without errors
- Encrypting the data
- Displaying a web page

That would be impossibly complex. So networking is split into **layers**. Each layer does ONE job and hands off to the next layer.

```
WITHOUT layers:
  One giant, complex, impossible-to-debug system

WITH layers:
  Layer 7: "I want to load a web page"          (Application)
  Layer 6: "Let me format the data"               (Presentation)
  Layer 5: "Let me manage the conversation"        (Session)
  Layer 4: "Let me make sure it all arrives"       (Transport)
  Layer 3: "Let me figure out the route"           (Network)
  Layer 2: "Let me send it to the right device"    (Data Link)
  Layer 1: "Let me put electrical signals on wire" (Physical)

  Each layer only talks to the layer above and below it.
  Each layer can be changed independently.
```

---

## The Real-World Analogy — Sending a Package

You want to send a birthday gift to your friend in another city. This process has layers too:

```
YOU (sender):
  Layer 7 — Application:  You decide to send a gift (the intent)
  Layer 6 — Presentation: You wrap the gift in nice paper (formatting)
  Layer 5 — Session:      You choose to use FedEx (start the conversation)
  Layer 4 — Transport:    FedEx gives you a tracking number (reliable delivery)
  Layer 3 — Network:      FedEx figures out the route (which cities, which hubs)
  Layer 2 — Data Link:    The delivery truck drives to the next stop (local hop)
  Layer 1 — Physical:     The truck is on an actual road (physical medium)

YOUR FRIEND (receiver):
  Layer 1 — Physical:     Package arrives by truck
  Layer 2 — Data Link:    Delivered to correct house on the street
  Layer 3 — Network:      Package arrived in the right city
  Layer 4 — Transport:    Tracking confirms delivery (all pieces arrived)
  Layer 5 — Session:      FedEx delivery complete
  Layer 6 — Presentation: Unwrap the paper
  Layer 7 — Application:  Friend sees the gift! 🎁
```

Notice: **each layer doesn't care about the layers above or below it**. The truck driver (Layer 1) doesn't care what's in the package. FedEx routing (Layer 3) doesn't care what road the truck takes (Layer 1). The wrapping paper (Layer 6) doesn't care which city it goes to (Layer 3).

This independence is the whole point of layers.

---

## The OSI Model — 7 Layers

The **OSI model** (Open Systems Interconnection) is the **theoretical framework** — the textbook version. It has 7 layers:

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 7  │  Application   │  HTTP, HTTPS, DNS, SSH, FTP         │
├──────────┼────────────────┼─────────────────────────────────────┤
│ Layer 6  │  Presentation  │  Encryption (SSL/TLS), compression  │
├──────────┼────────────────┼─────────────────────────────────────┤
│ Layer 5  │  Session       │  Establish/manage connections        │
├──────────┼────────────────┼─────────────────────────────────────┤
│ Layer 4  │  Transport     │  TCP, UDP                           │
├──────────┼────────────────┼─────────────────────────────────────┤
│ Layer 3  │  Network       │  IP addresses, routing              │
├──────────┼────────────────┼─────────────────────────────────────┤
│ Layer 2  │  Data Link     │  MAC addresses, switches            │
├──────────┼────────────────┼─────────────────────────────────────┤
│ Layer 1  │  Physical      │  Cables, Wi-Fi, electrical signals  │
└──────────┴────────────────┴─────────────────────────────────────┘
         ▲                                                    ▲
      Closest                                             Closest
      to you                                              to the
     (human)                                             cable/wire
```

**Memory trick:** From bottom to top: **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way
(Physical, Data Link, Network, Transport, Session, Presentation, Application)

Let's go through each one.

---

## Layer 1 — Physical

**What it does:** Moves raw bits (0s and 1s) across a physical medium.

This layer is about the actual hardware — cables, signals, voltages, light pulses, radio waves.

```
Computer A                                    Computer B
  sends 0s and 1s                              receives 0s and 1s
     │                                              ▲
     └──── electrical signals on copper wire ────────┘
     └──── light pulses on fiber optic cable ────────┘
     └──── radio waves through air (Wi-Fi) ─────────┘
```

### What Layer 1 Includes

| Thing | How It Carries Data |
|-------|-------------------|
| **Ethernet cable (copper)** | Electrical voltage changes: high = 1, low = 0 |
| **Fiber optic cable** | Light pulses: light on = 1, light off = 0 |
| **Wi-Fi** | Radio wave patterns |
| **Network card (NIC)** | The hardware in your computer that plugs into the cable |

### What Layer 1 Does NOT Know

- It has no idea what the data means
- It doesn't know where the data should go
- It doesn't know if the data arrived correctly
- It just pushes bits onto the wire

**Analogy:** Layer 1 is the **road**. It doesn't know where the car is going. It just provides the surface for the car to drive on.

---

## Layer 2 — Data Link

**What it does:** Sends data between devices on the **same local network** (LAN). Uses **MAC addresses**.

Layer 2 takes the raw bits from Layer 1 and organizes them into **frames** (chunks of data with a source and destination MAC address).

```
Frame:
┌────────────────────────────────────────────────────────────────┐
│ Destination MAC │ Source MAC │ Type │    Data    │ Error Check │
│  A4:83:E7:2B:.. │ B8:27:EB:.. │ IPv4 │  (payload) │  (CRC)     │
└────────────────────────────────────────────────────────────────┘
```

### How a Switch Works (Layer 2)

```
Computer A (MAC: AA:AA:AA:AA:AA:AA) ─── Port 1 ─┐
Computer B (MAC: BB:BB:BB:BB:BB:BB) ─── Port 2 ─┤── SWITCH
Computer C (MAC: CC:CC:CC:CC:CC:CC) ─── Port 3 ─┘

Computer A wants to send data to Computer C:

1. A sends a frame with:
   Source: AA:AA:AA:AA:AA:AA
   Destination: CC:CC:CC:CC:CC:CC

2. Switch receives frame on Port 1
3. Switch looks at destination MAC: CC:CC:CC:CC:CC:CC
4. Switch knows CC:CC... is on Port 3
5. Switch sends frame ONLY to Port 3

Computer B never sees this data — the switch is smart.
```

### What Layer 2 Does NOT Know

- It has no idea about IP addresses or the internet
- It only works within ONE local network
- It can't route data to a different network

**Analogy:** Layer 2 is the **local mail delivery**. The mailman knows every house on your street (MAC addresses), but doesn't deliver to other cities.

---

## Layer 3 — Network

**What it does:** Routes data between **different networks** across the internet. Uses **IP addresses**.

This is the layer that makes the internet work. Layer 2 can only deliver within one LAN. Layer 3 figures out how to get data from your LAN, across the internet, to a server in another country.

```
Your Computer                                         Google's Server
IP: 192.168.1.5                                       IP: 142.250.80.46

Packet:
┌─────────────────────────────────────────────────────────────┐
│ Source IP: 192.168.1.5 │ Destination IP: 142.250.80.46 │ Data │
└─────────────────────────────────────────────────────────────┘

Layer 3 (routers) figure out: how do I get this packet from
192.168.1.5 to 142.250.80.46?

  Your Router → ISP Router → Backbone Router → Google's Router → Server
```

### Routers Are Layer 3 Devices

```
Your LAN                 Internet                Google's LAN
192.168.1.0/24           (many routers)          10.0.0.0/8
     │                        │                       │
  Router A ─────── Router B ────── Router C ────── Router D
     │                        │                       │
  "I need to                 "I'll pass               "This is
   send this to              it to the                 for my
   142.250.80.46.            next router               network.
   Not on my LAN.            closer to                 I'll deliver
   I'll send it to           Google."                  it to the
   my ISP router."                                     right server."
```

### IP Address (Preview)

You'll learn IP addresses fully in concept 03. For now:
- An IP address is a number like `192.168.1.5` or `142.250.80.46`
- Every device on a network gets one
- Routers use IP addresses to figure out where to send data

**Analogy:** Layer 3 is the **postal service routing system**. Your letter has a destination address (IP). The postal system figures out which trucks, planes, and sorting facilities it needs to pass through to get there.

---

## Layer 4 — Transport

**What it does:** Ensures data is delivered **reliably** (or quickly, depending on the protocol). Uses **ports** to identify which application the data is for.

Layer 3 got the data to the right computer. But a computer runs many applications at once — your browser, email, music streaming, a game. Layer 4 uses **port numbers** to deliver data to the right application.

```
Your Computer (192.168.1.5):
  - Browser      → Port 443 (HTTPS)
  - Email app    → Port 993 (IMAP)
  - Spotify      → Port 4070
  - SSH terminal → Port 22

Data arrives with:
  Destination IP: 192.168.1.5, Destination Port: 443
  → Layer 3 delivers to your computer
  → Layer 4 delivers to your browser (port 443)
```

### Two Transport Protocols: TCP and UDP

```
TCP (Transmission Control Protocol):
  ✓ Reliable — guarantees every piece arrives
  ✓ Ordered — data arrives in the right order
  ✓ Error-checked — detects and resends corrupted data
  ✗ Slower — because of all the checking
  Use for: Web pages, email, file transfers, APIs

  Like sending a REGISTERED LETTER:
    Tracked, guaranteed delivery, signature required

UDP (User Datagram Protocol):
  ✓ Fast — just sends, doesn't wait for confirmation
  ✗ Unreliable — data might get lost
  ✗ Unordered — data might arrive out of order
  ✗ No error recovery — if a piece is lost, it's gone
  Use for: Video calls, live streaming, online gaming, DNS lookups

  Like SHOUTING across a room:
    Fast, but some words might get lost. That's fine — you get the gist.
```

### Why Not Always Use TCP?

In a video call, if one frame of video gets lost, you don't want to pause and wait for it to be resent — by then the conversation has moved on. You'd rather skip it and keep going. That's UDP.

For loading a web page, you need every byte — a missing byte means a broken page. That's TCP.

**Analogy:** Layer 4 is the **apartment number** on a letter. Layer 3 (IP) gets the letter to the right building (computer). Layer 4 (port) gets it to the right apartment (application).

---

## Layer 5 — Session

**What it does:** Manages **connections** (sessions) between applications. Starts, maintains, and ends conversations.

```
Your Browser ←── session ──→ Gmail Server

Session lifecycle:
  1. Open session (you log in to Gmail)
  2. Exchange data (load emails, send emails)
  3. Keep session alive (you keep the tab open)
  4. Close session (you close the tab or log out)
```

This layer keeps track of who's talking to whom and makes sure a long conversation doesn't randomly break.

**In practice:** Layers 5, 6, and 7 are often merged together. Most people and most documentation don't distinguish between them. When someone says "Layer 7" they usually mean all three.

---

## Layer 6 — Presentation

**What it does:** Translates data into a format the application can understand. Handles **encryption**, **compression**, and **encoding**.

```
Data from the network: [encrypted binary blob]
  ↓ Layer 6 decrypts it (SSL/TLS)
  ↓ Layer 6 decompresses it (gzip)
  ↓ Layer 6 decodes it (UTF-8 text)
Data for the application: "Hello, welcome to Gmail!"
```

This is where **SSL/TLS encryption** technically lives — it encrypts data before sending and decrypts after receiving.

**In practice:** Just like Layer 5, most people lump this into "Layer 7."

---

## Layer 7 — Application

**What it does:** The layer closest to the **user**. This is where the actual applications and protocols you interact with live.

```
When you type https://google.com in your browser:

Layer 7 (Application):  HTTP protocol — "GET /search?q=cats"
Layer 6 (Presentation): SSL/TLS encrypts the request
Layer 5 (Session):      Maintains the HTTPS connection
Layer 4 (Transport):    TCP ensures reliable delivery, port 443
Layer 3 (Network):      IP routing to 142.250.80.46
Layer 2 (Data Link):    Frame sent to your router's MAC address
Layer 1 (Physical):     Electrical signals on your Ethernet cable
```

### Layer 7 Protocols

| Protocol | Port | What It Does |
|----------|------|-------------|
| **HTTP** | 80 | Web pages (unencrypted) |
| **HTTPS** | 443 | Web pages (encrypted) |
| **DNS** | 53 | Name resolution (google.com → IP) |
| **SSH** | 22 | Secure remote access |
| **FTP** | 21 | File transfer |
| **SMTP** | 25 | Sending email |
| **IMAP** | 993 | Reading email |

**Analogy:** Layer 7 is the **actual letter content**. All the other layers are about envelopes, addressing, trucks, and roads. Layer 7 is the message itself — "Happy Birthday!"

---

## The TCP/IP Model — The Practical Version

The OSI model has 7 layers. In reality, the internet uses the **TCP/IP model** which has **4 layers**. It's simpler and matches how things actually work:

```
TCP/IP Model:                          Maps to OSI:
┌───────────────────────────────┐
│ Layer 4: Application          │ ←── OSI Layers 5 + 6 + 7
│   HTTP, HTTPS, DNS, SSH, etc. │
├───────────────────────────────┤
│ Layer 3: Transport            │ ←── OSI Layer 4
│   TCP, UDP                    │
├───────────────────────────────┤
│ Layer 2: Internet             │ ←── OSI Layer 3
│   IP, ICMP, routing           │
├───────────────────────────────┤
│ Layer 1: Network Access       │ ←── OSI Layers 1 + 2
│   Ethernet, Wi-Fi, cables     │
└───────────────────────────────┘
```

The TCP/IP model is what the internet actually runs on. The OSI model is used as a **reference** to talk about layers.

---

## OSI vs TCP/IP — Quick Comparison

```
OSI (7 layers — reference model):       TCP/IP (4 layers — real internet):
┌──────────────────────┐                ┌──────────────────────┐
│ 7. Application       │                │                      │
├──────────────────────┤                │ 4. Application       │
│ 6. Presentation      │                │                      │
├──────────────────────┤                │                      │
│ 5. Session           │                │                      │
├──────────────────────┤                ├──────────────────────┤
│ 4. Transport         │                │ 3. Transport         │
├──────────────────────┤                ├──────────────────────┤
│ 3. Network           │                │ 2. Internet          │
├──────────────────────┤                ├──────────────────────┤
│ 2. Data Link         │                │                      │
├──────────────────────┤                │ 1. Network Access    │
│ 1. Physical          │                │                      │
└──────────────────────┘                └──────────────────────┘
```

**Which one should you use?** People **talk** using OSI layer numbers (because they're more specific), but the internet **runs** on TCP/IP. When someone says "Layer 4 load balancer," they mean OSI Layer 4 (Transport — TCP/UDP).

---

## What "Layer X" Means When People Say It

You'll hear these phrases constantly in AWS and networking. Here's what they actually mean:

| Phrase | OSI Layer | What They Mean |
|--------|-----------|---------------|
| **"Layer 2"** | Data Link | Working with MAC addresses, switches, VLANs |
| **"Layer 3"** | Network | Working with IP addresses, routing |
| **"Layer 4"** | Transport | Working with TCP/UDP, port numbers |
| **"Layer 7"** | Application | Working with HTTP, URLs, headers, request body |
| **"Layer 4 load balancer"** | Transport | Routes based on IP + port (doesn't look at HTTP content) |
| **"Layer 7 load balancer"** | Application | Routes based on URL path, HTTP headers, cookies |
| **"Layer 3 firewall"** | Network | Filters based on IP addresses |
| **"Layer 4 firewall"** | Transport | Filters based on IP + port |
| **"Layer 7 firewall"** | Application | Filters based on HTTP content (WAF) |
| **"Deep packet inspection"** | All layers | Looks at everything — from IP to actual content |

---

## How Data Moves Through the Layers

When you load `https://google.com`, data goes **down** the layers on your computer, **across** the network, and **up** the layers on Google's server:

```
YOUR COMPUTER                                    GOOGLE'S SERVER

Layer 7: Browser creates           Layer 7: Web server reads
         HTTP request                        HTTP request
         "GET /search"                       "GET /search"
              │                                   ▲
              ▼                                   │
Layer 4: TCP adds port info        Layer 4: TCP removes port info
         src port: 52431                    and delivers to
         dst port: 443                      port 443 application
              │                                   ▲
              ▼                                   │
Layer 3: IP adds addresses         Layer 3: IP reads addresses
         src: 192.168.1.5                   dst: 142.250.80.46
         dst: 142.250.80.46                 "this is for me!"
              │                                   ▲
              ▼                                   │
Layer 2: Ethernet frame            Layer 2: Ethernet frame
         src MAC → dst MAC                  received and
         (your router's MAC)                unwrapped
              │                                   ▲
              ▼                                   │
Layer 1: Electrical signals  ───────────→  Layer 1: Signals received
         on the cable           (network)          from cable

         GOING DOWN                        GOING UP
         (sending)                         (receiving)
```

---

## Encapsulation — Wrapping Data in Envelopes

Each layer **wraps** the data from the layer above with its own header. This is called **encapsulation**:

```
Layer 7: [HTTP Request: "GET /search"]

Layer 4: [TCP Header: port 443] [HTTP Request: "GET /search"]

Layer 3: [IP Header: src/dst IP] [TCP Header] [HTTP Request]

Layer 2: [Ethernet Header: MAC] [IP Header] [TCP Header] [HTTP Request] [Trailer]

Layer 1: 01001011101001011010010110100101... (raw bits on the wire)
```

It's like putting a letter inside envelopes:

```
1. Write the letter                    → "GET /search" (Layer 7)
2. Put in envelope, write port number  → TCP envelope (Layer 4)
3. Put in bigger envelope, write address → IP envelope (Layer 3)
4. Put in shipping package, write local address → Ethernet package (Layer 2)
5. Put on delivery truck               → Physical signal (Layer 1)
```

On the receiving end, each layer **unwraps** its envelope and passes the content up:

```
Layer 1: Receive bits → pass to Layer 2
Layer 2: Remove Ethernet header → pass to Layer 3
Layer 3: Remove IP header → pass to Layer 4
Layer 4: Remove TCP header → pass to Layer 7
Layer 7: "GET /search" → application processes it
```

### What Each Layer's Data Unit Is Called

| Layer | Data Unit Name |
|-------|---------------|
| Layer 7 (Application) | **Data** or **Message** |
| Layer 4 (Transport) | **Segment** (TCP) or **Datagram** (UDP) |
| Layer 3 (Network) | **Packet** |
| Layer 2 (Data Link) | **Frame** |
| Layer 1 (Physical) | **Bits** |

When someone says "packet" they usually mean Layer 3 (IP level). When they say "frame" they mean Layer 2 (Ethernet level).

---

## Summary Table — All 7 Layers at a Glance

| Layer | Name | What It Does | Key Protocols/Devices | Data Unit | Real-World Analogy |
|-------|------|-------------|----------------------|-----------|-------------------|
| **7** | Application | User-facing protocols | HTTP, HTTPS, DNS, SSH | Data | The letter content |
| **6** | Presentation | Encryption, formatting | SSL/TLS, JPEG, gzip | Data | Wrapping paper |
| **5** | Session | Connection management | Sessions, sockets | Data | The conversation |
| **4** | Transport | Reliable/fast delivery + ports | TCP, UDP | Segment | Apartment number |
| **3** | Network | Routing across networks | IP, routers | Packet | Postal address |
| **2** | Data Link | Local delivery | MAC, switches | Frame | Street delivery |
| **1** | Physical | Raw bits on wire | Cables, Wi-Fi, NIC | Bits | The road |

---

## How This Connects to AWS

This is why layers matter for your AWS learning:

| AWS Service | Layer | What It Means |
|------------|-------|--------------|
| **NLB** (Network Load Balancer) | **Layer 4** | Routes based on IP + TCP/UDP port. Doesn't look at HTTP content. Very fast. |
| **ALB** (Application Load Balancer) | **Layer 7** | Routes based on URL path (`/api` vs `/web`), HTTP headers, cookies. Slower but smarter. |
| **GWLB** (Gateway Load Balancer) | **Layer 3** | Passes entire IP packets to firewalls for inspection. Transparent — doesn't change anything. |
| **Security Groups** | **Layer 3 + 4** | Rules based on IP address (Layer 3) + port (Layer 4). "Allow TCP port 443 from 10.0.0.0/8" |
| **NACLs** | **Layer 3 + 4** | Same — IP + port rules, but at the subnet level |
| **WAF** (Web Application Firewall) | **Layer 7** | Inspects HTTP request body, headers, URLs. Blocks SQL injection, XSS. |
| **Palo Alto / GWLB firewalls** | **All layers** | Deep packet inspection — looks at everything from Layer 3 to Layer 7 |
| **VPC** | **Layer 3** | A virtual network with IP address ranges, subnets, route tables |
| **Transit Gateway** | **Layer 3** | Routes IP packets between VPCs |

When someone says "NLB is a Layer 4 load balancer" — you now know exactly what that means: it makes routing decisions based on IP addresses and TCP/UDP port numbers, without looking at the HTTP content inside.

When someone says "WAF is Layer 7" — you know it inspects the actual HTTP request (the URL, headers, body) to block attacks.

---

> **Previous:** [01 — What Is a Network](01-what-is-a-network.md)
> **Next:** [03 — IP Addresses](03-ip-addresses.md)
