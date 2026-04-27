# 08 — Firewalls

How firewalls allow and block network traffic — inbound vs outbound rules, stateful vs stateless, packet filtering, and how firewalls secure every network.

**Prerequisites:** Concept 03 (IP addresses), Concept 04 (CIDR), Concept 05 (ports, TCP/UDP).

---

## Table of Contents

- [What Is a Firewall](#what-is-a-firewall)
- [The Security Guard Analogy](#the-security-guard-analogy)
- [Why Firewalls Exist](#why-firewalls-exist)
- [What Firewalls Look At](#what-firewalls-look-at)
- [Inbound vs Outbound Traffic](#inbound-vs-outbound-traffic)
- [Firewall Rules — Allow and Deny](#firewall-rules-allow-and-deny)
- [How Rules Are Evaluated](#how-rules-are-evaluated)
- [Default Deny vs Default Allow](#default-deny-vs-default-allow)
- [Stateful vs Stateless Firewalls](#stateful-vs-stateless-firewalls)
- [Stateless Firewall — Checks Every Packet Individually](#stateless-firewall-checks-every-packet-individually)
- [Stateful Firewall — Remembers Connections](#stateful-firewall-remembers-connections)
- [Stateful vs Stateless — Complete Comparison](#stateful-vs-stateless-complete-comparison)
- [Types of Firewalls](#types-of-firewalls)
- [Packet Filtering Firewall (Layer 3-4)](#packet-filtering-firewall-layer-3-4)
- [Application Firewall / WAF (Layer 7)](#application-firewall-waf-layer-7)
- [Network Firewall vs Host Firewall](#network-firewall-vs-host-firewall)
- [Next-Generation Firewalls (NGFW)](#next-generation-firewalls-ngfw)
- [Firewall Rule Examples](#firewall-rule-examples)
- [Common Firewall Patterns](#common-firewall-patterns)
- [Linux Firewall — iptables and nftables](#linux-firewall-iptables-and-nftables)
- [Troubleshooting Firewall Issues](#troubleshooting-firewall-issues)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## What Is a Firewall

A **firewall** is a system that controls which network traffic is **allowed** and which is **blocked**. It sits between two networks (or between a network and a device) and inspects every packet.

```
Internet (untrusted)         Firewall         Your Network (trusted)
                               │
Hacker scanning port 22 ─────→│──→ BLOCKED (rule: deny SSH from internet)
                               │
Customer on port 443 ─────────→│──→ ALLOWED (rule: allow HTTPS from anywhere)
                               │
Your server replying ──────────│←── ALLOWED (rule: allow outbound responses)
```

A firewall is just a **ruleset**: a list of conditions that decide "allow" or "deny" for each packet.

---

## The Security Guard Analogy

```
Building security guard:
  "Do you have a badge?" → YES → come in
  "Are you on the guest list?" → NO → you can't enter
  "Are you carrying prohibited items?" → YES → denied

Firewall:
  "Is this traffic on port 443?" → YES → allow
  "Is this coming from 10.0.0.0/8?" → YES → allow
  "Is this SSH from the internet?" → YES → deny
```

---

## Why Firewalls Exist

Without a firewall, **every port on every server is open** to the entire internet:

```
No firewall:
  Server 10.0.1.50:
    Port 22 (SSH) → OPEN to the entire internet → anyone can try to log in
    Port 3306 (MySQL) → OPEN → anyone can try to access your database
    Port 8080 (app) → OPEN → anyone can access internal tools
    Port 443 (HTTPS) → OPEN → anyone can access your website (this one is fine)

With firewall:
  Server 10.0.1.50:
    Port 22 (SSH) → only from 10.0.0.0/8 (internal network)
    Port 3306 (MySQL) → only from 10.0.1.0/24 (app subnet)
    Port 8080 (app) → BLOCKED from internet
    Port 443 (HTTPS) → OPEN to 0.0.0.0/0 (everyone — it's public)
```

---

## What Firewalls Look At

A firewall inspects packet headers to make decisions:

```
Every packet has:
  Source IP:        192.168.1.5       ← Where it came from
  Destination IP:   10.0.1.50        ← Where it's going
  Protocol:         TCP               ← TCP or UDP
  Source Port:      52431             ← Ephemeral port (return address)
  Destination Port: 443              ← What service it's targeting

The firewall checks these fields against its rules.
```

| Field | What It Tells the Firewall |
|-------|---------------------------|
| **Source IP** | Who is sending this traffic? |
| **Destination IP** | Who is it going to? |
| **Protocol** | TCP, UDP, or ICMP (ping)? |
| **Destination port** | What service is being accessed? |
| **Direction** | Inbound (coming in) or outbound (going out)? |

---

## Inbound vs Outbound Traffic

```
INBOUND (ingress):
  Traffic coming INTO your network/server FROM outside
  Example: A user accessing your website

  Internet ──────→ [Firewall] ──────→ Your Server
                   (inbound rules checked)

OUTBOUND (egress):
  Traffic going OUT of your network/server TO outside
  Example: Your server calling an external API

  Your Server ──────→ [Firewall] ──────→ Internet
                      (outbound rules checked)
```

### Why Control Outbound Too?

```
Scenario: An attacker compromises your server.
  Without outbound rules: malware phones home, downloads tools, exfiltrates data
  With outbound rules: malware can't reach the internet — attack contained

Good outbound rules:
  ✓ Allow HTTPS (443) outbound — needed for API calls, updates
  ✗ Block everything else outbound — no SSH out, no random ports
```

---

## Firewall Rules — Allow and Deny

Each rule specifies:

```
Rule format:
  [Direction] [Protocol] [Source IP/CIDR] [Dest Port] → [Allow/Deny]

Examples:
  Inbound  TCP  0.0.0.0/0      443  → ALLOW   (HTTPS from anywhere)
  Inbound  TCP  10.0.0.0/8     22   → ALLOW   (SSH from internal only)
  Inbound  TCP  0.0.0.0/0      22   → DENY    (SSH from internet — blocked)
  Outbound TCP  10.0.1.0/24    3306 → ALLOW   (app can reach MySQL)
  Inbound  ALL  0.0.0.0/0      ALL  → DENY    (block everything else)
```

---

## How Rules Are Evaluated

Rules are checked **in order, top to bottom**. The first matching rule wins:

```
Rule 1: Allow TCP 10.0.0.0/8 port 22    ← checked first
Rule 2: Deny TCP 0.0.0.0/0 port 22      ← checked second
Rule 3: Allow TCP 0.0.0.0/0 port 443    ← checked third
Rule *: Deny ALL (default — if nothing matches)  ← last resort

Packet from 10.0.1.5, TCP, port 22:
  Rule 1: Source is 10.0.1.5, in 10.0.0.0/8? YES → ALLOW → stop checking

Packet from 203.0.113.50, TCP, port 22:
  Rule 1: Source is 203.0.113.50, in 10.0.0.0/8? NO → next rule
  Rule 2: Source is 203.0.113.50, in 0.0.0.0/0? YES → DENY → stop checking

Packet from 203.0.113.50, TCP, port 443:
  Rule 1: Port 22? NO → next
  Rule 2: Port 22? NO → next
  Rule 3: Port 443 from 0.0.0.0/0? YES → ALLOW → stop checking
```

**Order matters!** If you put a broad "deny all" before specific "allow" rules, nothing gets through.

---

## Default Deny vs Default Allow

### Default Deny (Recommended — Whitelist approach)

```
Start: block EVERYTHING
Then: explicitly allow only what you need

Rule 1: Allow TCP port 443 from 0.0.0.0/0
Rule 2: Allow TCP port 22 from 10.0.0.0/8
Default: DENY everything else

Result: only HTTPS and internal SSH work. Everything else is blocked.
```

### Default Allow (Dangerous — Blacklist approach)

```
Start: allow EVERYTHING
Then: explicitly block what you don't want

Rule 1: Deny TCP port 23 (Telnet)
Rule 2: Deny TCP port 3389 from 0.0.0.0/0 (RDP)
Default: ALLOW everything else

Result: anything you forgot to block is open. Dangerous.
```

**Always use default deny.** You'd rather accidentally block something (easy to fix) than accidentally expose something (security breach).

---

## Stateful vs Stateless Firewalls

This is one of the most important firewall concepts.

---

### Stateless Firewall — Checks Every Packet Individually

A stateless firewall treats each packet **independently**. It doesn't remember previous packets or connections.

```
Problem with stateless:

You allow inbound port 443 (HTTPS).
A client connects and sends a request.
Your server generates a response.

The response goes OUTBOUND to the client's ephemeral port (e.g., 52431).

Stateless firewall at this point:
  "Outbound traffic to port 52431? I don't have a rule for that."
  → BLOCKED!

You'd need to manually allow the ENTIRE ephemeral port range outbound:
  Allow outbound TCP ports 1024-65535 to 0.0.0.0/0
  (That's a very broad rule — not great for security)
```

```
Stateless firewall requires EXPLICIT rules for BOTH directions:

  Inbound:   Allow TCP 0.0.0.0/0 → port 443       (the request)
  Outbound:  Allow TCP port 443 → 0.0.0.0/0:1024-65535  (the response)
```

---

### Stateful Firewall — Remembers Connections

A stateful firewall **tracks connections**. When it allows an inbound packet, it automatically allows the **response** back out — no extra rule needed.

```
Stateful firewall:

1. Client sends SYN to port 443 → firewall checks inbound rules → ALLOW
2. Firewall records: "Connection from 203.0.113.50:52431 to 10.0.1.50:443"
3. Server sends SYN-ACK back → firewall checks:
   "Is this a response to an existing connection? YES" → ALLOW automatically
4. All subsequent packets in this connection → ALLOW automatically

You only need ONE rule:
  Inbound: Allow TCP 0.0.0.0/0 → port 443

The response traffic is automatically allowed because the firewall
remembers the connection.
```

### The Connection Table

Stateful firewalls maintain a **connection table** (also called state table):

```
Connection Table:
  Protocol  Source IP:Port       Dest IP:Port        State
  TCP       203.0.113.50:52431   10.0.1.50:443      ESTABLISHED
  TCP       198.51.100.8:61234   10.0.1.50:443      ESTABLISHED
  TCP       10.0.1.50:44321      10.0.2.30:5432     ESTABLISHED

Any packet that matches an existing connection → automatically allowed
New inbound packet? → check inbound rules
New outbound packet? → check outbound rules
```

---

### Stateful vs Stateless — Complete Comparison

| Feature | Stateful | Stateless |
|---------|----------|-----------|
| **Remembers connections** | Yes | No |
| **Return traffic** | Automatically allowed | Must have explicit rule |
| **Number of rules needed** | Fewer (one direction enough) | More (both directions needed) |
| **Performance** | Slightly slower (tracks state) | Slightly faster (no state) |
| **Security** | Higher (only allows valid responses) | Lower (broad return rules needed) |
| **Complexity** | Simpler rules | More complex rules |

---

## Types of Firewalls

### Packet Filtering Firewall (Layer 3-4)

The most basic type. Looks at IP addresses, ports, and protocol:

```
What it sees:
  Source IP, Dest IP, Source Port, Dest Port, Protocol (TCP/UDP)

What it DOESN'T see:
  The actual data (HTTP requests, SQL queries, etc.)

Example:
  "Allow TCP to port 443" → it does this
  "Block SQL injection attacks" → it CANNOT do this
```

### Application Firewall / WAF (Layer 7)

**WAF** = Web Application Firewall. It understands the **content** of traffic:

```
What it sees (in addition to Layer 3-4):
  HTTP methods (GET, POST)
  URLs (/admin, /api/users)
  Headers (User-Agent, cookies)
  Request body (form data, JSON)
  SQL queries inside the data

What it can block:
  SQL injection: DROP TABLE users  → BLOCKED
  Cross-site scripting: <script>alert('xss')</script> → BLOCKED
  Path traversal: /../../../etc/passwd → BLOCKED
  Known attack patterns → BLOCKED
```

```
Layer 3-4 Firewall:
  "Is this going to port 443? Yes → ALLOW"
  (Doesn't care if it's a SQL injection attack)

Layer 7 WAF:
  "Is this going to port 443? Yes.
   What's in the HTTP request? 'DROP TABLE'?! → BLOCKED!"
```

### Network Firewall vs Host Firewall

```
Network Firewall:
  Sits at the NETWORK boundary
  Protects the entire network
  All traffic passes through it

  Internet → [Network Firewall] → All Servers

Host Firewall:
  Runs ON the server itself
  Protects just that one server
  Example: iptables on Linux, Windows Firewall

  Server: [Host Firewall (iptables)] → Application

Best practice: use BOTH (defense in depth)
```

### Next-Generation Firewalls (NGFW)

Combines everything:
- Packet filtering (Layer 3-4)
- Stateful inspection
- Deep packet inspection (Layer 7)
- Intrusion Prevention System (IPS)
- URL filtering
- Malware detection

Examples: Palo Alto, Fortinet FortiGate, Cisco Firepower, Check Point.

---

## Firewall Rule Examples

### Web Server

```
Inbound Rules:
  Allow TCP port 443 from 0.0.0.0/0     ← HTTPS from anyone
  Allow TCP port 80 from 0.0.0.0/0      ← HTTP (redirect to HTTPS)
  Allow TCP port 22 from 10.0.0.0/8     ← SSH from internal only
  Deny ALL from 0.0.0.0/0               ← block everything else

Outbound Rules:
  Allow TCP port 443 to 0.0.0.0/0       ← server can make HTTPS calls
  Allow TCP port 5432 to 10.0.2.0/24    ← server can reach database subnet
  Deny ALL to 0.0.0.0/0                 ← block all other outbound
```

### Database Server

```
Inbound Rules:
  Allow TCP port 5432 from 10.0.1.0/24  ← PostgreSQL from app subnet only
  Allow TCP port 22 from 10.0.0.5/32    ← SSH from bastion host only
  Deny ALL from 0.0.0.0/0               ← nothing else gets in

Outbound Rules:
  Allow TCP port 443 to 0.0.0.0/0       ← for software updates
  Deny ALL to 0.0.0.0/0                 ← no other outbound
```

### Bastion Host (Jump Box)

```
A bastion host is a hardened server that you SSH into FIRST,
then from there SSH to internal servers. It's the only server
with SSH exposed to the internet.

Inbound Rules:
  Allow TCP port 22 from 203.0.113.0/24   ← SSH from your office IP only
  Deny ALL from 0.0.0.0/0

Outbound Rules:
  Allow TCP port 22 to 10.0.0.0/8         ← SSH to internal servers
  Deny ALL to 0.0.0.0/0
```

---

## Common Firewall Patterns

### Least Privilege

```
Only allow exactly what's needed, nothing more:

BAD:   Allow ALL from 0.0.0.0/0       ← everything from everywhere
GOOD:  Allow TCP port 443 from 0.0.0.0/0  ← only HTTPS, only TCP
BEST:  Allow TCP port 443 from 198.51.100.0/24  ← only HTTPS from known IPs
```

### Defense in Depth

```
Multiple layers of firewalls:

Internet → [Network Firewall] → [Load Balancer] → [Security Group] → [Host Firewall] → App

Even if one layer is misconfigured, others still protect you.
```

### Microsegmentation

```
Separate firewall rules for every subnet/service:

Web tier:      Can talk to App tier on port 8080
App tier:      Can talk to DB tier on port 5432
DB tier:       Can talk to nobody except App tier

Web tier → DB tier?  BLOCKED (web servers don't need database access directly)
```

---

## Linux Firewall — iptables and nftables

Linux has a built-in firewall. Older systems use **iptables**, newer ones use **nftables**:

```bash
# View current firewall rules:
iptables -L -n -v

# Allow incoming SSH:
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow incoming HTTPS:
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow established connections (stateful — allow responses):
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block everything else:
iptables -A INPUT -j DROP

# Read the rules:
#   -A INPUT   = append to inbound chain
#   -p tcp     = protocol TCP
#   --dport 22 = destination port 22
#   -j ACCEPT  = action: allow
#   -j DROP    = action: silently discard
```

---

## Troubleshooting Firewall Issues

### "Connection timed out" — Usually a firewall blocking

```bash
# Test if a port is reachable:
nc -zv 10.0.1.50 443
# Connection timed out → firewall is probably blocking

nc -zv 10.0.1.50 443
# Connection refused → port is reachable but nothing is listening

nc -zv 10.0.1.50 443
# Connection succeeded → port is open and service is running
```

### Common mistakes

```
Problem: Can reach server on port 443 but not port 22
Cause:   Firewall allows 443 but not 22 from your IP

Problem: Server can't download packages / call external APIs
Cause:   Outbound rules too restrictive — need to allow port 443 outbound

Problem: New app on port 8080 not reachable
Cause:   Forgot to add a firewall rule for port 8080

Problem: Works from one IP but not another
Cause:   Firewall rule restricts source IP (e.g., only 10.0.0.0/8 allowed)
```

### Debugging checklist

```
1. Is the application running? (ss -tlnp | grep <port>)
2. Is the host firewall allowing it? (iptables -L -n)
3. Is the network firewall allowing it? (check cloud firewall rules)
4. Is the route correct? (traceroute to destination)
5. Is there a NAT or proxy in the way?
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **Firewall** | Controls which traffic is allowed/blocked based on rules |
| **Inbound (ingress)** | Traffic coming IN to your network/server |
| **Outbound (egress)** | Traffic going OUT from your network/server |
| **Stateful** | Remembers connections — automatically allows return traffic |
| **Stateless** | Treats every packet independently — needs rules for both directions |
| **Default deny** | Block everything, then allow specific traffic (recommended) |
| **Packet filter** | Layer 3-4 firewall — checks IPs and ports |
| **WAF** | Layer 7 firewall — inspects HTTP content (SQL injection, XSS) |
| **NGFW** | Next-gen firewall — combines packet filtering + deep inspection |
| **Least privilege** | Only allow exactly what's needed |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| **Stateful firewall** | **Security Groups** — stateful, no need for return rules |
| **Stateless firewall** | **NACLs** (Network ACLs) — stateless, need rules for both directions |
| **WAF (Layer 7)** | **AWS WAF** — protects ALBs, CloudFront, API Gateway from SQL injection, XSS |
| **Network firewall** | **AWS Network Firewall** — managed firewall for VPC |
| **NGFW** | **Palo Alto on AWS** — runs as EC2 instances or GWLB endpoints |
| **Inbound rules** | Security Group inbound rules, NACL inbound rules |
| **Outbound rules** | Security Group outbound rules (default: allow all), NACL outbound rules |
| **Default deny** | Security Groups default deny all inbound, NACLs have explicit deny |

### Security Groups vs NACLs — The AWS Comparison

```
Security Groups (STATEFUL):
  - Attached to EC2 instances / ENIs
  - ALLOW rules only (no deny rules)
  - Return traffic automatically allowed
  - All rules evaluated (no order)
  - Default: deny all inbound, allow all outbound

NACLs (STATELESS):
  - Attached to SUBNETS
  - ALLOW and DENY rules
  - Return traffic needs explicit rule (ephemeral ports!)
  - Rules evaluated in ORDER (number)
  - Default NACL: allow all. Custom NACL: deny all.
```

```
NACL example (stateless — need return traffic rules):

  Inbound Rules:
    100  Allow TCP port 443 from 0.0.0.0/0
    *    Deny ALL

  Outbound Rules:
    100  Allow TCP ports 1024-65535 to 0.0.0.0/0  ← RETURN traffic on ephemeral ports!
    *    Deny ALL

  If you forget the outbound ephemeral port rule → responses can't leave → broken
```

```
Security Group example (stateful — no return rules needed):

  Inbound:
    Allow TCP port 443 from 0.0.0.0/0

  Outbound:
    (default: allow all — or customize)

  Return traffic? Automatically allowed because stateful.
```

---

> **Previous:** [07 — DNS](07-dns.md)
> **Next:** [09 — NAT](09-nat.md)
