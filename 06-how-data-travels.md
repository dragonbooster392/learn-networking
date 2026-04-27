# 06 — How Data Travels

What actually happens when you type a URL — packets, routing, hops, traceroute, ARP, and the complete journey of a request from your keyboard to a server and back.

**Prerequisites:** Concept 02 (layers, encapsulation), Concept 03 (IP addresses), Concept 04 (subnets), Concept 05 (ports, TCP, three-way handshake).

---

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Step 1 — You Type a URL](#step-1-you-type-a-url)
- [Step 2 — DNS Lookup](#step-2-dns-lookup)
- [Step 3 — ARP — Finding the MAC Address](#step-3-arp-finding-the-mac-address)
- [Step 4 — The Packet Is Built (Encapsulation)](#step-4-the-packet-is-built-encapsulation)
- [Step 5 — From Your Computer to Your Router](#step-5-from-your-computer-to-your-router)
- [Step 6 — Router to ISP to the Internet](#step-6-router-to-isp-to-the-internet)
- [Step 7 — Hop by Hop Across the Internet](#step-7-hop-by-hop-across-the-internet)
- [Step 8 — Arrival at the Server](#step-8-arrival-at-the-server)
- [Step 9 — The Response Travels Back](#step-9-the-response-travels-back)
- [Routing — How Routers Decide Where to Send Packets](#routing-how-routers-decide-where-to-send-packets)
- [Route Tables](#route-tables)
- [Default Gateway — The Exit Door](#default-gateway-the-exit-door)
- [Traceroute — See the Hops Yourself](#traceroute-see-the-hops-yourself)
- [Ping — Test Connectivity](#ping-test-connectivity)
- [Packet Loss and What Causes It](#packet-loss-and-what-causes-it)
- [Hands-On: Reading Your Own Traceroute (Real Output)](#hands-on-reading-your-own-traceroute-real-output)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## The Big Picture

When you open `https://google.com` in your browser, here's everything that happens:

```
Your Keyboard → Browser → OS → Network Card → Cable/Wi-Fi →
Your Router → ISP → Internet Backbone → Google's Network →
Google's Router → Google's Server → Application processes request →
Response travels the ENTIRE path back in reverse → Your Screen
```

Total time: usually 50-200 milliseconds. Let's trace every step.

---

## Step 1 — You Type a URL

```
You type: https://google.com/search?q=cats

Your browser breaks this down:
  Protocol:  https (use TLS encryption, port 443)
  Hostname:  google.com (need to convert to an IP address)
  Path:      /search?q=cats (what to request from the server)
```

Before anything can happen, the browser needs **Google's IP address**. It only has the name "google.com."

---

## Step 2 — DNS Lookup

Your browser asks: "What is the IP address for google.com?"

```
Browser → OS DNS cache: "Do I already know google.com's IP?"
  → If yes: use the cached IP (skip the lookup)
  → If no: ask the DNS server

OS → DNS Server (e.g., 8.8.8.8):
  "What's the IP for google.com?"
  → DNS server responds: "142.250.80.46"

Now the browser knows: connect to 142.250.80.46:443
```

DNS is covered fully in concept 07. For now, just know: name → IP translation happens first.

---

## Step 3 — ARP — Finding the MAC Address

Your computer knows the **destination IP** (142.250.80.46) but it needs to send the packet to your **router first** (since Google is not on your LAN). And to send data on the local network (Layer 2), it needs the router's **MAC address**.

**ARP** (Address Resolution Protocol) translates IP addresses to MAC addresses on the local network:

```
Your computer: "I need to send to 142.250.80.46, which is not on my subnet.
                I need to send to my default gateway: 192.168.1.1.
                What's the MAC address of 192.168.1.1?"

ARP broadcast: "Hey everyone on this LAN! Who has IP 192.168.1.1?"

Router responds: "That's me! My MAC address is AA:BB:CC:DD:EE:FF"

Your computer: "Got it. I'll address the Ethernet frame to AA:BB:CC:DD:EE:FF"
```

```
ARP cache (your computer remembers for next time):
  192.168.1.1 → AA:BB:CC:DD:EE:FF
  192.168.1.8 → 11:22:33:44:55:66  (your phone)

Check your ARP cache:
  Linux/Mac: arp -a
  Windows:   arp -a
```

### Why ARP Matters

- Layer 2 (Ethernet) uses **MAC addresses** to deliver frames
- Layer 3 (IP) uses **IP addresses** to route packets
- ARP bridges the gap: "I know the IP, give me the MAC"
- ARP only works **within one subnet** (local network)

---

## Step 4 — The Packet Is Built (Encapsulation)

Your OS builds the packet layer by layer (concept 02 — encapsulation):

```
Layer 7 (Application):
  HTTP Request: "GET /search?q=cats HTTP/1.1\r\nHost: google.com"

Layer 4 (Transport — TCP):
  ┌──────────────────────────────────────────────────────┐
  │ Source Port: 52431 │ Dest Port: 443 │ Seq: 101 │ ... │
  │ [HTTP request data]                                  │
  └──────────────────────────────────────────────────────┘

Layer 3 (Network — IP):
  ┌──────────────────────────────────────────────────────┐
  │ Source IP: 192.168.1.5 │ Dest IP: 142.250.80.46     │
  │ TTL: 64 │ Protocol: TCP                              │
  │ [TCP segment]                                        │
  └──────────────────────────────────────────────────────┘

Layer 2 (Data Link — Ethernet):
  ┌──────────────────────────────────────────────────────┐
  │ Dest MAC: AA:BB:CC:DD:EE:FF │ Source MAC: my-MAC    │
  │ Type: IPv4                                           │
  │ [IP packet]                                          │
  │ [CRC checksum]                                       │
  └──────────────────────────────────────────────────────┘

Layer 1 (Physical):
  01001010110100101101001... (electrical signals on the wire)
```

Notice: the **Destination MAC** is the **router's MAC** (not Google's). That's because the next hop is the router. But the **Destination IP** is **Google's IP** — that stays the same the whole journey.

---

## Step 5 — From Your Computer to Your Router

```
Your Laptop                              Your Router
 192.168.1.5                              192.168.1.1
     │                                         │
     │  Ethernet frame:                        │
     │  Dest MAC: router's MAC                 │
     │  Dest IP: 142.250.80.46                 │
     │ ────────────────────────────────────────→│
     │  (over Wi-Fi or Ethernet cable)         │
                                               │
                                    Router receives frame:
                                    1. Strips Ethernet header (Layer 2)
                                    2. Reads IP header (Layer 3)
                                    3. "Dest IP 142.250.80.46 — not on my LAN"
                                    4. Checks route table → send to ISP
                                    5. Does NAT (concept 09):
                                       replaces 192.168.1.5 with public IP
                                    6. Builds new Ethernet frame for ISP link
                                    7. Sends it out
```

---

## Step 6 — Router to ISP to the Internet

```
Your Router → ISP's Router → ISP's Core Network → Internet Exchange Point

Your Router:
  "I don't know how to reach 142.250.80.46.
   I'll send it to my ISP — they'll know."

ISP's Router:
  "142.250.80.46 is Google. I know Google's network is via
   the Internet Exchange Point. I'll forward it there."
```

---

## Step 7 — Hop by Hop Across the Internet

The packet doesn't teleport to Google. It passes through **multiple routers**, each one forwarding it one step closer:

```
Hop 1: Your Router (192.168.1.1)
Hop 2: ISP Local Router (10.100.1.1)
Hop 3: ISP Regional Router (10.200.1.1)
Hop 4: Internet Exchange (198.32.132.1)
Hop 5: Google's Edge Router (108.170.242.1)
Hop 6: Google's Internal Router (142.250.80.1)
Hop 7: Google's Server (142.250.80.46)

Each hop:
  1. Router receives packet
  2. Reads destination IP (142.250.80.46)
  3. Looks up in route table: "where should I send this?"
  4. Forwards to the next router
  5. Decrements TTL (Time To Live) by 1
```

### TTL (Time To Live)

Every IP packet has a **TTL** value (usually starts at 64 or 128). Each router decrements it by 1. If TTL reaches 0, the packet is **discarded**.

```
TTL prevents packets from circling forever if there's a routing loop:

Packet created: TTL=64
Hop 1: TTL=63
Hop 2: TTL=62
...
Hop 64: TTL=0 → DISCARDED (and sender gets an error message)
```

---

## Step 8 — Arrival at the Server

```
Google's Server (142.250.80.46):
  1. Receives the Ethernet frame (Layer 2)
  2. Strips Ethernet header → reads IP packet (Layer 3)
  3. Strips IP header → reads TCP segment (Layer 4)
  4. TCP checks sequence numbers, sends ACK back
  5. Strips TCP header → reads HTTP request (Layer 7)
  6. Web application processes: "GET /search?q=cats"
  7. Generates response: HTML page with cat search results
  8. Sends response back through all the layers
```

---

## Step 9 — The Response Travels Back

The response follows the **reverse path**. The routers don't remember the original packet — they make independent routing decisions. But the source and destination are swapped:

```
Response packet:
  Source IP: 142.250.80.46 (Google)
  Dest IP:   73.162.45.201 (your public IP — after NAT)
  Source Port: 443
  Dest Port:  52431 (your ephemeral port)

Path: Google → Internet → ISP → Your Router → NAT → Your Laptop:52431 → Browser
```

### Total Journey Summary

```
Forward (request):
  Keyboard → Browser → DNS → TCP handshake → HTTP request →
  Layers 7→4→3→2→1 → Wi-Fi → Router → NAT → ISP → Internet →
  Google → Layers 1→2→3→4→7 → Web server → Process request

Backward (response):
  Web server → Layers 7→4→3→2→1 → Internet → ISP →
  Router → NAT → Layers 1→2→3→4→7 → Browser → Screen

You see: cat search results 🐱
```

---

## Routing — How Routers Decide Where to Send Packets

A router's only job is: **look at the destination IP, decide the next hop.**

It uses a **route table** (also called a routing table) — a list of rules:

```
Router's Route Table:
  Destination        Next Hop          Interface
  ──────────────────────────────────────────────
  192.168.1.0/24     directly connected  eth0 (LAN)
  10.0.0.0/8         10.100.1.1          eth1 (ISP link)
  0.0.0.0/0          10.100.1.1          eth1 (default route)
```

### How the Router Uses the Route Table

```
Packet arrives with Dest IP: 142.250.80.46

Router checks each route:
  192.168.1.0/24 → Does 142.250.80.46 match? NO (not in 192.168.1.x range)
  10.0.0.0/8     → Does 142.250.80.46 match? NO (not in 10.x.x.x range)
  0.0.0.0/0      → MATCHES everything (default route)

Action: forward to 10.100.1.1 via eth1
```

### Longest Prefix Match

If multiple routes match, the router picks the **most specific** one (longest prefix = highest /number):

```
Route Table:
  10.0.0.0/8       → Router A
  10.0.1.0/24      → Router B
  10.0.1.128/25    → Router C

Packet for 10.0.1.200:
  10.0.0.0/8     matches ✓ (/8 — very broad)
  10.0.1.0/24    matches ✓ (/24 — more specific)
  10.0.1.128/25  matches ✓ (/25 — most specific)

Winner: 10.0.1.128/25 → send to Router C
(longest prefix = most specific = wins)
```

---

## Route Tables

```bash
# View your computer's route table (Linux):
ip route show

# Output:
default via 192.168.1.1 dev wlan0          ← default route (0.0.0.0/0)
192.168.1.0/24 dev wlan0 scope link        ← local subnet (direct)

# This means:
# - Anything on 192.168.1.0/24 → send directly (same LAN)
# - Everything else → send to 192.168.1.1 (your router)
```

---

## Default Gateway — The Exit Door

The **default gateway** is the router your device sends ALL non-local traffic to. It's the exit from your LAN.

```
Your computer (192.168.1.5):
  "I want to reach 142.250.80.46"
  "Is 142.250.80.46 on my subnet (192.168.1.0/24)? NO"
  "I'll send it to my default gateway: 192.168.1.1 (the router)"
```

DHCP (concept 03) automatically tells your device the default gateway. Without it, your device can talk to other devices on the LAN but **nothing outside**.

```
No default gateway? → Can ping 192.168.1.x devices ✓
                    → Cannot reach google.com ✗
                    → Cannot reach anything outside LAN ✗
```

---

## Traceroute — See the Hops Yourself

**traceroute** (Linux/Mac) or **tracert** (Windows) shows every router hop between you and a destination:

```bash
traceroute google.com

 1  192.168.1.1 (your router)        1ms
 2  10.100.1.1 (ISP local)           5ms
 3  10.200.1.1 (ISP regional)        8ms
 4  198.32.132.1 (internet exchange)  12ms
 5  108.170.242.1 (Google edge)       15ms
 6  142.250.80.46 (Google server)     18ms
```

Each line is a **hop** — a router the packet passes through. The time shows the **round-trip latency** to that hop.

### How Traceroute Works

It sends packets with increasing **TTL** values:

```
TTL=1 → dies at hop 1 → hop 1 sends back "TTL expired" → you see hop 1's IP
TTL=2 → dies at hop 2 → hop 2 sends back "TTL expired" → you see hop 2's IP
TTL=3 → dies at hop 3 → ...
...until the packet reaches the destination
```

### Reading Traceroute Output

```
 5  108.170.242.1  15ms  16ms  14ms

Hop number: 5
Router IP:  108.170.242.1
Round-trip times: 15ms, 16ms, 14ms (three probes)

If you see * * *: that router doesn't respond (firewalled) — packet still passes through
```

---

## Ping — Test Connectivity

**ping** sends a tiny packet to a destination and measures if it responds:

```bash
ping google.com

PING google.com (142.250.80.46): 56 data bytes
64 bytes from 142.250.80.46: seq=0 ttl=117 time=15.2 ms
64 bytes from 142.250.80.46: seq=1 ttl=117 time=14.8 ms
64 bytes from 142.250.80.46: seq=2 ttl=117 time=15.1 ms

--- google.com ping statistics ---
3 packets sent, 3 received, 0% packet loss
round-trip min/avg/max = 14.8/15.0/15.2 ms
```

| Field | Meaning |
|-------|---------|
| `ttl=117` | Time To Live — packet crossed ~(128-117)=11 routers |
| `time=15.2 ms` | Round-trip latency |
| `0% packet loss` | All packets arrived — connection is healthy |

```bash
# Ping a server to check if it's reachable:
ping 10.0.1.50

# If no response: server is down, firewall is blocking, or route is broken
```

---

## Packet Loss and What Causes It

When packets don't arrive, that's **packet loss**:

```
5 packets sent, 3 received, 40% packet loss ← problem!
```

| Cause | What's Happening |
|-------|-----------------|
| **Network congestion** | Too much traffic, routers drop packets |
| **Faulty cable** | Physical damage causing intermittent loss |
| **Wi-Fi interference** | Walls, distance, other devices |
| **Firewall blocking** | Firewall drops certain packets |
| **Routing problem** | Packet goes to wrong destination |
| **Server overloaded** | Server can't handle all requests |

---

## Hands-On: Reading Your Own Traceroute (Real Output)

Let's trace data from **this actual machine** (10.12.24.141) to Google and understand every hop.

### Step 1 — Know Your Starting Point

```bash
ip addr show ens192 | grep inet
```

```
inet 10.12.24.141/23 brd 10.12.25.255 scope global noprefixroute ens192
```

This machine's IP is **10.12.24.141** with a **/23 subnet** (10.12.24.0 – 10.12.25.255).

```bash
ip route show
```

```
default via 10.12.24.1 dev ens192 proto static metric 100
10.12.24.0/23 dev ens192 proto kernel scope link src 10.12.24.141 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

Reading this route table:

```
Line 1: default via 10.12.24.1
  → "For everything NOT on my local network, send to 10.12.24.1"
  → 10.12.24.1 is the DEFAULT GATEWAY (the exit door)

Line 2: 10.12.24.0/23 dev ens192
  → "For anything in 10.12.24.0 – 10.12.25.255, send directly (same LAN)"
  → No router needed — these machines are on your subnet

Line 3: 172.17.0.0/16 dev docker0
  → "For 172.17.x.x, use the Docker bridge"
  → This is Docker's internal network — ignore for now
```

### Step 2 — Run Traceroute

```bash
traceroute -I -n google.com
```

(`-I` = use ICMP like ping, `-n` = don't do DNS lookups — show raw IPs)

```
traceroute to google.com (172.253.118.102), 30 hops max, 60 byte packets
 1  10.12.24.3        22.160 ms  22.163 ms  22.183 ms
 2  172.16.1.5         1.181 ms   1.116 ms   1.212 ms
 3  * 172.23.0.3       0.644 ms *
 4  * * *
 5  * * *
 ...
20  * * *
21  172.253.118.102   81.437 ms  81.450 ms  81.468 ms
```

### Step 3 — Read Every Hop

```
YOUR MACHINE: 10.12.24.141

Hop 1: 10.12.24.3  (22 ms)
  │
  │  Wait — this is NOT 10.12.24.1 (the default gateway from route table).
  │
  │  Why? Your default gateway is 10.12.24.1, but that router FORWARDED
  │  the packet to 10.12.24.3 (or 10.12.24.1 didn't respond to TTL-expired
  │  and the next device that DID respond is 10.12.24.3).
  │
  │  10.12.24.3 is on YOUR subnet (10.12.24.0/23) — likely a firewall or
  │  gateway appliance sitting in the same VLAN as your machine.
  │
  │  22ms is high for a local hop — this suggests the device is doing
  │  inspection or processing (firewall, proxy, or security appliance).
  │
Hop 2: 172.16.1.5  (1 ms)
  │
  │  172.16.x.x is a PRIVATE IP (172.16.0.0/12 range from concept 03).
  │  This is an internal company router — the packet has left your
  │  subnet and entered the corporate backbone.
  │  1ms = fast, it's nearby (same data center or building).
  │
Hop 3: 172.23.0.3  (0.6 ms)
  │
  │  Another private IP. Still inside the corporate network.
  │  This might be the router that connects your company to the ISP.
  │  The * means one of the three probes didn't get a response
  │  (the router was busy or rate-limited ICMP).
  │
Hops 4–20: * * *
  │
  │  ALL SILENT. These routers exist but REFUSE to respond.
  │
  │  Why? Two common reasons:
  │  1. Corporate/ISP firewalls block ICMP TTL-expired messages
  │  2. Routers are configured to not reveal their IPs (security)
  │
  │  The packet IS passing through them (it reaches Google eventually).
  │  They just won't tell you who they are. Like driving through
  │  a tunnel — you're moving, but you can't see the road signs.
  │
Hop 21: 172.253.118.102  (81 ms)
  │
  │  GOOGLE'S SERVER! The packet arrived.
  │  172.253.x.x is Google's IP range.
  │  81ms round-trip = your total latency to Google.
```

### Step 4 — Verify with Ping

```bash
ping -c 3 google.com
```

```
64 bytes from 172.253.118.102: icmp_seq=1 ttl=109 time=82.1 ms
64 bytes from 172.253.118.102: icmp_seq=2 ttl=109 time=80.5 ms
64 bytes from 172.253.118.102: icmp_seq=3 ttl=109 time=80.7 ms

3 packets transmitted, 3 received, 0% packet loss
```

Reading this:

```
ttl=109:
  Google probably sent with TTL=128 (Windows/Linux default).
  128 - 109 = 19 routers between you and Google.

  But traceroute showed 21 hops? That's because some hops
  showed * * * (they exist but didn't respond to traceroute).
  The TTL count (19) is actually MORE accurate than traceroute
  for counting real hops.

time=80-82 ms:
  ~80ms round-trip to Google. Consistent with traceroute's 81ms.
  This is reasonable — your traffic goes through a corporate
  network before reaching the internet.

0% packet loss:
  Perfect connectivity. No packets dropped.
```

### The Complete Picture of YOUR Network Path

```
Your Machine (10.12.24.141)
     │
     │ Route table says: "default via 10.12.24.1"
     │ Packet sent to gateway
     │
     ▼
Hop 1: 10.12.24.3 (corporate firewall/gateway on your VLAN)
     │  22ms — doing inspection
     │
     ▼
Hop 2: 172.16.1.5 (corporate backbone router)
     │  1ms — fast, nearby
     │
     ▼
Hop 3: 172.23.0.3 (corporate edge / ISP handoff)
     │  0.6ms
     │
     ▼
Hops 4-20: ISP routers + internet backbone
     │  All silent (* * *) — firewalls hiding them
     │  ~17 invisible hops
     │
     ▼
Hop 21: 172.253.118.102 (Google's server)
     │  81ms total round-trip
     │
     ▼
Response travels back the same chain → your screen
```

### Key Takeaways from This Traceroute

```
1. Your traffic passes through a CORPORATE NETWORK first
   (10.12.24.x → 172.16.x → 172.23.x — all private IPs)
   before reaching the public internet.

2. Most of the internet hops are INVISIBLE (* * *)
   because firewalls and ISP routers hide themselves.
   This is NORMAL in corporate environments.

3. The 22ms on hop 1 suggests a security appliance
   (firewall/IDS) — it's inspecting your traffic.
   A simple router would be <1ms on the same LAN.

4. Total latency to Google: ~81ms
   About 22ms is your corporate network overhead.
   The remaining ~59ms is internet transit.

5. "Why isn't my default gateway (10.12.24.1) showing?"
   10.12.24.1 may be a virtual IP (VRRP/HSRP) that
   forwards to 10.12.24.3 without decrementing TTL,
   or it simply doesn't send ICMP TTL-expired replies.
   Either way, 10.12.24.3 is the first device that
   RESPONDED — it's effectively your first visible hop.
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **Packet** | A chunk of data with headers (IP, TCP, etc.) |
| **Hop** | Each router a packet passes through |
| **Route table** | List of rules a router uses to forward packets |
| **Default gateway** | The router that handles traffic leaving your LAN |
| **ARP** | Translates IP addresses to MAC addresses (local network only) |
| **TTL** | Counter that prevents packets from looping forever |
| **Traceroute** | Shows every hop between you and a destination |
| **Ping** | Tests if a destination is reachable |
| **Packet loss** | Data that never arrives |
| **NAT** | Translates private IPs to public (covered in concept 09) |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| Route tables | Every VPC **subnet** has a route table — you configure where traffic goes |
| Default gateway | Subnet route: `0.0.0.0/0 → Internet Gateway` (public subnet) or `0.0.0.0/0 → NAT Gateway` (private subnet) |
| Hops | Traffic in AWS: Subnet → Route Table → TGW → Route Table → Destination subnet |
| TTL | Packets crossing many VPCs still have TTL — usually not an issue |
| ARP | Handled automatically within AWS VPC subnets |
| Traceroute | Useful for debugging: `traceroute` from an EC2 instance to see the path |
| Ping | `ping` between EC2 instances to test connectivity (security groups must allow ICMP) |
| Longest prefix match | AWS route tables use longest prefix match — most specific route wins |

```
AWS Route Table example:
  10.0.0.0/16  → local           (within the VPC — direct)
  10.1.0.0/16  → tgw-abc123      (via Transit Gateway to another VPC)
  0.0.0.0/0    → igw-xyz789      (everything else → Internet Gateway)

Packet for 10.1.2.50:
  10.0.0.0/16 → no match (10.1 ≠ 10.0)
  10.1.0.0/16 → MATCH → send via Transit Gateway
```

---

> **Previous:** [05 — Ports & Protocols](05-ports-and-protocols.md)
> **Next:** [07 — DNS](07-dns.md)
