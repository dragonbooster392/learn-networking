# 05 — Ports & Protocols

What ports are, why they exist, TCP vs UDP in detail, the three-way handshake, and every common port number you need to know.

**Prerequisites:** Concept 02 (Layer 4 — Transport), Concept 03 (IP addresses).

---

## Table of Contents

- [What Is a Port](#what-is-a-port)
- [The Apartment Building Analogy](#the-apartment-building-analogy)
- [Port Number Ranges](#port-number-ranges)
- [Common Port Numbers — The Ones to Know](#common-port-numbers-the-ones-to-know)
- [Socket — IP + Port Together](#socket-ip-port-together)
- [TCP — The Reliable Protocol](#tcp-the-reliable-protocol)
- [The Three-Way Handshake](#the-three-way-handshake)
- [TCP Data Transfer — How Reliability Works](#tcp-data-transfer-how-reliability-works)
- [TCP Connection Termination](#tcp-connection-termination)
- [UDP — The Fast Protocol](#udp-the-fast-protocol)
- [TCP vs UDP — Complete Comparison](#tcp-vs-udp-complete-comparison)
- [When to Use TCP vs UDP](#when-to-use-tcp-vs-udp)
- [Common Application Protocols](#common-application-protocols)
- [HTTP and HTTPS — The Web Protocols](#http-and-https-the-web-protocols)
- [SSH — Secure Shell](#ssh-secure-shell)
- [How to Check Open Ports](#how-to-check-open-ports)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## What Is a Port

An **IP address** gets data to the right **computer**. A **port** gets data to the right **application** on that computer.

Your computer runs many applications at the same time — browser, email, Spotify, SSH terminal. Each one needs to receive its own data without mixing up with the others. Ports solve this.

```
Data arrives at your computer (192.168.1.5):
  "Destination port: 443" → goes to your BROWSER (HTTPS)
  "Destination port: 22"  → goes to your SSH SESSION
  "Destination port: 993" → goes to your EMAIL APP (IMAP)

Without ports, your browser would get email data,
your email app would get web pages — chaos.
```

A port is just a **number** from 0 to 65,535.

---

## The Apartment Building Analogy

```
IP address = building address (123 Main Street)
Port       = apartment number (Apt 443)

Letter addressed to: 123 Main Street, Apt 443
  → delivery person finds the building (IP)
  → then finds apartment 443 (port)
  → delivers to the right person (application)

Letter addressed to: 123 Main Street, Apt 22
  → same building
  → different apartment
  → different person receives it
```

---

## Port Number Ranges

| Range | Name | Who Uses It |
|-------|------|------------|
| **0 – 1,023** | Well-known ports | Standard services (HTTP=80, HTTPS=443, SSH=22) |
| **1,024 – 49,151** | Registered ports | Applications (MySQL=3306, PostgreSQL=5432) |
| **49,152 – 65,535** | Dynamic/ephemeral ports | Your computer picks these temporarily for outgoing connections |

### Ephemeral Ports — The Return Address

When your browser connects to Google:

```
Your Browser                          Google's Server
192.168.1.5:52431  ──────────────→   142.250.80.46:443

Source port: 52431    ← random ephemeral port (your computer picked it)
Destination port: 443 ← well-known HTTPS port

Google sends the response BACK to:
142.250.80.46:443  ──────────────→   192.168.1.5:52431
```

Your computer picks a random high port (like 52431) as the "return address." When Google replies, it sends data back to that port, and your OS delivers it to the correct browser tab.

---

## Common Port Numbers — The Ones to Know

These are the ports you'll encounter constantly:

| Port | Protocol | What It's For |
|------|----------|--------------|
| **22** | SSH | Remote access to Linux servers |
| **53** | DNS | Domain name lookups (google.com → IP) |
| **80** | HTTP | Web pages (unencrypted) |
| **443** | HTTPS | Web pages (encrypted — the standard now) |
| **25** | SMTP | Sending email |
| **110** | POP3 | Retrieving email (old) |
| **143** | IMAP | Retrieving email (modern) |
| **993** | IMAPS | Retrieving email (encrypted) |
| **3306** | MySQL | MySQL database |
| **5432** | PostgreSQL | PostgreSQL database |
| **6379** | Redis | Redis cache |
| **27017** | MongoDB | MongoDB database |
| **9092** | Kafka | Apache Kafka brokers |
| **9098** | Kafka (IAM) | Kafka with IAM authentication (AWS MSK) |
| **8080** | HTTP (alt) | Alternate HTTP — common for dev servers |
| **8443** | HTTPS (alt) | Alternate HTTPS |
| **3389** | RDP | Remote Desktop to Windows servers |
| **10250** | Kubelet | Kubernetes node communication |
| **6443** | K8s API | Kubernetes API server |
| **15017** | Istio | Istio webhook |

### The Big Four to Memorize

```
22  = SSH     (remote access)
53  = DNS     (name resolution)
80  = HTTP    (web — unencrypted)
443 = HTTPS   (web — encrypted)
```

---

## Socket — IP + Port Together

A **socket** is an IP address + port number together. It uniquely identifies one end of a connection:

```
Socket = IP : Port

Your browser:  192.168.1.5:52431
Google:        142.250.80.46:443

A connection is defined by BOTH sockets:
  (192.168.1.5:52431) ←──→ (142.250.80.46:443)
```

Your computer can have many connections to Google at the same time (multiple tabs). Each uses a different source port:

```
Tab 1: 192.168.1.5:52431 ←→ 142.250.80.46:443
Tab 2: 192.168.1.5:52432 ←→ 142.250.80.46:443
Tab 3: 192.168.1.5:52433 ←→ 142.250.80.46:443

Different source ports → different connections → no mix-up
```

---

## TCP — The Reliable Protocol

**TCP** (Transmission Control Protocol) guarantees that data arrives **completely, in order, and without errors**.

```
TCP promises:
  ✓ Data will arrive (or you'll be told it failed)
  ✓ Data will be in the correct order
  ✓ Duplicate data will be discarded
  ✓ Corrupted data will be detected and resent
  ✓ Both sides agree before sending (connection established first)
```

How TCP achieves this:
1. **Connection setup** — three-way handshake (both sides agree to talk)
2. **Sequence numbers** — every piece of data is numbered
3. **Acknowledgments** — receiver confirms what it received
4. **Retransmission** — if something is lost, sender resends it
5. **Flow control** — sender doesn't overwhelm a slow receiver
6. **Ordered delivery** — even if packets arrive out of order, TCP reassembles them

---

## The Three-Way Handshake

Before any data is sent via TCP, both sides do a **three-way handshake** to establish the connection:

```
Step 1: SYN (Synchronize)
  Client: "Hey, I want to connect. Here's my sequence number: 100"

Step 2: SYN-ACK (Synchronize + Acknowledge)
  Server: "I got your SYN. Here's MY sequence number: 300.
           I acknowledge your sequence 100 (expecting 101 next)."

Step 3: ACK (Acknowledge)
  Client: "I got your SYN-ACK. I acknowledge your sequence 300
           (expecting 301 next). We're connected!"
```

```
Client                                 Server
  │                                       │
  │ ──── SYN (seq=100) ─────────────────→ │
  │                                       │
  │ ←──── SYN-ACK (seq=300, ack=101) ──── │
  │                                       │
  │ ──── ACK (ack=301) ─────────────────→ │
  │                                       │
  │         CONNECTION ESTABLISHED         │
  │ ←─────── data flows both ways ──────→ │
```

### Why Three Steps?

- Step 1: Client proves it can **send**
- Step 2: Server proves it can **send and receive**
- Step 3: Client proves it can **receive**

After these three steps, both sides know the other is alive, listening, and ready.

### Real-World Analogy

```
You:      "Hey, can you hear me?" (SYN)
Friend:   "Yes I can hear you! Can you hear me?" (SYN-ACK)
You:      "Yes I can hear you too! Let's talk." (ACK)
```

---

## TCP Data Transfer — How Reliability Works

After the handshake, data is sent in **segments**, each with a sequence number:

```
Sender sends:
  Segment 1: seq=101, data="Hello "     (6 bytes)
  Segment 2: seq=107, data="World!"     (6 bytes)

Receiver acknowledges:
  ACK: ack=107   ("I got everything up to byte 107, send me 107 next")
  ACK: ack=113   ("I got everything up to byte 113, we're good")
```

### What If a Packet Gets Lost?

```
Sender sends:
  Segment 1: seq=101 → ✓ received
  Segment 2: seq=107 → ✗ LOST in the network
  Segment 3: seq=113 → ✓ received

Receiver sends:
  ACK: ack=107  ("I got up to 107, but I'm still waiting for 107")
  ACK: ack=107  ("Still waiting for 107...")

Sender sees: "They keep asking for 107 — it must have been lost"
Sender RESENDS segment 2: seq=107
Receiver: ACK=119  ("Got it now! All caught up.")
```

This is why TCP is reliable — lost data is automatically detected and resent.

---

## TCP Connection Termination

When done, both sides do a **four-way close**:

```
Client                                 Server
  │                                       │
  │ ──── FIN ────────────────────────────→ │  "I'm done sending"
  │                                       │
  │ ←──── ACK ─────────────────────────── │  "Got it"
  │                                       │
  │ ←──── FIN ─────────────────────────── │  "I'm done sending too"
  │                                       │
  │ ──── ACK ────────────────────────────→ │  "Got it. Connection closed."
  │                                       │
```

---

## UDP — The Fast Protocol

**UDP** (User Datagram Protocol) is the opposite of TCP. It's simple and fast, but makes no promises.

```
UDP says:
  "Here's your data. Good luck."

  ✗ No connection setup (no handshake)
  ✗ No guarantee it arrives
  ✗ No guarantee of order
  ✗ No retransmission if lost
  ✗ No flow control

  ✓ Very fast (no overhead)
  ✓ Very simple
  ✓ Good for real-time data
```

### How UDP Sends Data

```
Sender:
  "Here's a datagram for 142.250.80.46:53"  → sends it → done

  No handshake. No waiting for acknowledgment. Just fire and forget.
```

### Why Would Anyone Use UDP?

Because sometimes **speed matters more than perfection**:

```
Video call:
  If frame #47 is lost... you DON'T want to pause and wait for retransmission.
  By the time it arrives, the conversation has moved on.
  Better to skip it and show frame #48.

  TCP: "Wait... retransmitting frame 47... got it... displaying... too late, lag."
  UDP: "Frame 47 lost? Whatever, here's frame 48." (smooth, real-time)

DNS lookup:
  "What's the IP for google.com?"
  If the response is lost, the application just asks again.
  No need for TCP's complexity for a single small question.
```

---

## TCP vs UDP — Complete Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented (handshake first) | Connectionless (just send) |
| **Reliability** | Guaranteed delivery | Best-effort (might get lost) |
| **Ordering** | Data arrives in order | Data might arrive out of order |
| **Error detection** | Yes + retransmission | Error detection only (no resend) |
| **Speed** | Slower (overhead from reliability) | Faster (no overhead) |
| **Header size** | 20 bytes | 8 bytes |
| **Flow control** | Yes (adjusts speed) | No |
| **Use case** | Web, email, file transfer, APIs, databases | Video, voice, gaming, DNS |

---

## When to Use TCP vs UDP

| Application | Protocol | Why |
|------------|----------|-----|
| **Web browsing (HTTP/HTTPS)** | TCP | Every byte of the page matters |
| **Email (SMTP, IMAP)** | TCP | Can't lose parts of an email |
| **File transfer (FTP, SCP)** | TCP | Can't have missing bytes in a file |
| **SSH** | TCP | Every keystroke must arrive |
| **Database queries** | TCP | Query results must be complete |
| **API calls (REST, gRPC)** | TCP | Responses must be complete |
| **DNS lookups** | UDP | Small, fast, retries at app level |
| **Video streaming (live)** | UDP | Real-time, skip lost frames |
| **Voice over IP** | UDP | Real-time, skip lost audio |
| **Online gaming** | UDP | Real-time positions matter more than old data |

---

## Common Application Protocols

These are **Layer 7** protocols (concept 02) that run **on top of** TCP or UDP:

### HTTP (HyperText Transfer Protocol)

```
How the web works. Your browser speaks HTTP to web servers.

Request:
  GET /search?q=cats HTTP/1.1
  Host: google.com

Response:
  HTTP/1.1 200 OK
  Content-Type: text/html
  <html>...search results...</html>
```

- Runs over **TCP port 80** (unencrypted)
- Being replaced by HTTPS everywhere

### HTTPS (HTTP Secure)

```
Same as HTTP, but encrypted with TLS (concept 10 — SSL/TLS).

Your browser ←── encrypted tunnel (TLS) ──→ server
  Inside the tunnel: regular HTTP

Nobody between you and the server can read the data.
```

- Runs over **TCP port 443**
- The lock icon in your browser
- Standard for ALL websites now

### DNS (Domain Name System)

```
Translates domain names to IP addresses.

Request: "What's the IP for google.com?"
Response: "142.250.80.46"
```

- Runs over **UDP port 53** (fast, small queries)
- Also supports TCP port 53 (for large responses)
- Covered in detail in concept 07

### SSH (Secure Shell)

```
Remote access to Linux servers, encrypted.

You:     ssh user@server.example.com
         → connects to port 22
         → encrypted terminal session
         → you can type commands on the remote server
```

- Runs over **TCP port 22**
- Used to manage cloud servers, deploy code

---

## HTTP and HTTPS — The Web Protocols

Since HTTP/HTTPS is the most common protocol you'll work with:

### HTTP Methods

| Method | Purpose | Example |
|--------|---------|---------|
| **GET** | Read data | Load a web page, fetch API data |
| **POST** | Create data | Submit a form, create a new record |
| **PUT** | Replace data | Update an entire record |
| **PATCH** | Partially update | Update one field of a record |
| **DELETE** | Remove data | Delete a record |

### HTTP Status Codes

| Code | Meaning | Plain English |
|------|---------|--------------|
| **200** | OK | Success — here's your data |
| **201** | Created | Success — new resource was created |
| **301** | Moved Permanently | This page has moved, go here instead |
| **400** | Bad Request | Your request was malformed |
| **401** | Unauthorized | You need to log in |
| **403** | Forbidden | You're logged in but don't have permission |
| **404** | Not Found | This page/resource doesn't exist |
| **500** | Internal Server Error | Something broke on the server |
| **502** | Bad Gateway | Server behind the load balancer is down |
| **503** | Service Unavailable | Server is overloaded or under maintenance |

```
Quick rule:
  2xx → success
  3xx → redirect
  4xx → client made a mistake
  5xx → server made a mistake
```

---

## SSH — Secure Shell

SSH is how you remotely access and manage servers:

```bash
# Connect to a server:
ssh username@10.0.1.50

# Connect with a key file (common in AWS):
ssh -i my-key.pem ec2-user@10.0.1.50

# Copy a file to a server:
scp file.txt username@10.0.1.50:/home/username/
```

SSH encrypts everything — your password, your commands, the output. Nobody watching the network can see what you're typing.

---

## How to Check Open Ports

```bash
# See what ports are listening on YOUR machine (Linux/Mac):
ss -tlnp
# or: netstat -tlnp

# Output:
# LISTEN  0  128  0.0.0.0:22     0.0.0.0:*    users:(("sshd",pid=1234))
# LISTEN  0  128  0.0.0.0:443    0.0.0.0:*    users:(("nginx",pid=5678))
# This means SSH (22) and HTTPS (443) are open and listening

# Check if you can reach a specific port on ANOTHER machine:
nc -zv 10.0.1.50 22
# Connection to 10.0.1.50 22 port [tcp/ssh] succeeded!

nc -zv 10.0.1.50 443
# Connection to 10.0.1.50 443 port [tcp/https] succeeded!

# Check from outside using telnet:
telnet google.com 443
# Connected → port is open and reachable
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **Port** | A number (0-65535) that identifies an application on a computer |
| **Socket** | IP address + port number together (192.168.1.5:443) |
| **TCP** | Reliable, ordered, error-checked delivery (most traffic) |
| **UDP** | Fast, unreliable, fire-and-forget delivery (real-time traffic) |
| **Three-way handshake** | SYN → SYN-ACK → ACK (TCP connection setup) |
| **Ephemeral port** | Random high port your OS picks for outgoing connections |
| **HTTP (80)** | Web traffic, unencrypted |
| **HTTPS (443)** | Web traffic, encrypted |
| **SSH (22)** | Secure remote access |
| **DNS (53)** | Name resolution |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| Port 443 | What ALBs and NLBs listen on for HTTPS |
| Port 22 | SSH to EC2 instances (restricted in security groups) |
| Security Groups | Rules are: protocol (TCP/UDP) + port + source CIDR |
| NLB (Layer 4) | Routes based on TCP/UDP + port — doesn't look at HTTP |
| ALB (Layer 7) | Routes based on HTTP path/headers — understands port 80/443 |
| Kafka port 9098 | MSK IAM authentication port — allowed in security groups |
| Ephemeral ports | NACLs must allow ephemeral port range (1024-65535) for return traffic |
| HTTP status codes | ALB health checks — a "200" means the target is healthy |

```
Security Group example:
  Allow inbound TCP port 443 from 0.0.0.0/0     → HTTPS from anywhere
  Allow inbound TCP port 22 from 10.0.0.0/8      → SSH from private network only
  Allow inbound TCP port 5432 from 10.0.1.0/24   → PostgreSQL from app subnet only
```

---

> **Previous:** [04 — Subnets & CIDR](04-subnets-and-cidr.md)
> **Next:** [06 — How Data Travels](06-how-data-travels.md)
