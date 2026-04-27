# 07 — DNS (Domain Name System)

How your computer translates "google.com" into an IP address — the complete DNS lookup process, record types, caching, and why it matters.

**Prerequisites:** Concept 03 (IP addresses), Concept 05 (UDP port 53), Concept 06 (how data travels, route tables).

---

## Table of Contents

- [Why DNS Exists](#why-dns-exists)
- [The Phone Book Analogy](#the-phone-book-analogy)
- [DNS in One Sentence](#dns-in-one-sentence)
- [What Happens When You Type google.com](#what-happens-when-you-type-googlecom)
- [Step-by-Step DNS Lookup](#step-by-step-dns-lookup)
- [The DNS Hierarchy](#the-dns-hierarchy)
- [Root Name Servers](#root-name-servers)
- [TLD Name Servers](#tld-name-servers)
- [Authoritative Name Servers](#authoritative-name-servers)
- [Recursive Resolver — The One Doing the Work](#recursive-resolver-the-one-doing-the-work)
- [DNS Caching — Why Lookups Are Usually Fast](#dns-caching-why-lookups-are-usually-fast)
- [TTL in DNS — How Long to Cache](#ttl-in-dns-how-long-to-cache)
- [DNS Record Types](#dns-record-types)
- [A Record](#a-record)
- [AAAA Record](#aaaa-record)
- [CNAME Record](#cname-record)
- [MX Record](#mx-record)
- [NS Record](#ns-record)
- [TXT Record](#txt-record)
- [SOA Record](#soa-record)
- [PTR Record](#ptr-record)
- [SRV Record](#srv-record)
- [How to Query DNS Yourself](#how-to-query-dns-yourself)
- [FQDN — Fully Qualified Domain Name](#fqdn-fully-qualified-domain-name)
- [Zones and Zone Files](#zones-and-zone-files)
- [Public DNS vs Private DNS](#public-dns-vs-private-dns)
- [Common DNS Problems](#common-dns-problems)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## Why DNS Exists

Computers communicate using **IP addresses** (like 142.250.80.46). Humans remember **names** (like google.com). DNS is the system that translates between the two.

```
Without DNS:
  You'd type: https://142.250.80.46 in your browser
  And memorize IPs for every website you visit.
  Gmail? 142.250.189.5. YouTube? 142.250.191.46. Impossible.

With DNS:
  You type: https://google.com
  DNS automatically finds: 142.250.80.46
  Your browser connects using that IP.
```

---

## The Phone Book Analogy

```
Phone book:
  "John Smith" → 555-0123

DNS:
  "google.com" → 142.250.80.46

You look up the NAME, you get back the NUMBER.
```

But DNS is a **distributed** phone book — not one giant book, but millions of servers around the world, each responsible for a piece.

---

## DNS in One Sentence

DNS takes a domain name (google.com) and returns an IP address (142.250.80.46).

---

## What Happens When You Type google.com

Here's the complete flow — we'll break down each step after:

```
You type: google.com

1. Browser cache:    "Do I already know this IP?" → No
2. OS cache:         "Does the OS know?" → No
3. Router cache:     "Does the router know?" → No
4. ISP DNS Resolver: "Let me look it up..."
   a. Ask Root server:         "Where is .com?"
   b. Ask .com TLD server:     "Where is google.com?"
   c. Ask Google's server:     "What's the IP for google.com?"
   d. Answer:                  "142.250.80.46"
5. Result cached at every level
6. Browser connects to 142.250.80.46
```

---

## Step-by-Step DNS Lookup

Let's trace what happens in detail:

### Step 1 — Browser Cache

```
Browser: "Have I looked up google.com recently?"
  → YES: use cached IP (done, no network request needed)
  → NO:  ask the operating system
```

### Step 2 — OS Cache

```
OS: "Is google.com in my local DNS cache?"
  → YES: return cached IP to browser (done)
  → NO:  ask the configured DNS server
```

On Linux, check `/etc/resolv.conf` to see your DNS server:
```bash
cat /etc/resolv.conf
# nameserver 192.168.1.1     ← your router
# or
# nameserver 8.8.8.8         ← Google's public DNS
```

### Step 3 — Recursive Resolver

Your OS sends a DNS query (UDP port 53) to the configured DNS server. This is usually:
- Your **router** (which forwards to your ISP's DNS)
- Your **ISP's DNS server** directly
- A **public DNS** like Google (8.8.8.8) or Cloudflare (1.1.1.1)

This server is called the **recursive resolver** because it does the hard work of chasing down the answer.

```
Your computer → Recursive Resolver:
  "What's the IP for google.com?"
  
Resolver: "Let me check my cache..."
  → If cached: return answer immediately
  → If not cached: start asking the hierarchy
```

### Step 4 — Asking the Hierarchy

```
Recursive Resolver                 Root Server (one of 13)
       │                                  │
       │ "Where can I find .com?"          │
       │ ────────────────────────────────→ │
       │                                  │
       │ ←── "Ask the .com TLD servers:   │
       │      a.gtld-servers.net"         │
       │                                  │

Recursive Resolver                 .com TLD Server
       │                                  │
       │ "Where can I find google.com?"    │
       │ ────────────────────────────────→ │
       │                                  │
       │ ←── "Ask Google's nameservers:   │
       │      ns1.google.com"             │
       │                                  │

Recursive Resolver                 Google's Nameserver
       │                                  │
       │ "What's the IP for google.com?"   │
       │ ────────────────────────────────→ │
       │                                  │
       │ ←── "142.250.80.46"              │
       │                                  │

Recursive Resolver → Your Computer:
  "google.com = 142.250.80.46"
```

---

## The DNS Hierarchy

DNS is organized as a **tree**, from general to specific:

```
                         . (root)
                        / | \
                    .com  .org  .net  .io  .uk  .edu  ...
                   / | \
          google.com  amazon.com  github.com
          /    |
   mail.google.com  drive.google.com  maps.google.com
```

Each level is managed by different organizations:
- **Root** (.) — managed by ICANN, 13 root server clusters worldwide
- **TLDs** (.com, .org, etc.) — managed by registries (Verisign for .com)
- **Domain** (google.com) — managed by the domain owner (Google)
- **Subdomains** (mail.google.com) — managed by the domain owner

---

## Root Name Servers

The **root servers** are the starting point for every DNS lookup. There are **13 root server clusters** (named A through M), but each is actually hundreds of servers distributed globally.

```
Root servers don't know the IP of google.com.
They DO know where to find information about .com, .org, .net, etc.

Root server → "You're looking for something.com? 
               Ask the .com TLD servers."
```

You never interact with root servers directly — the recursive resolver does this for you.

---

## TLD Name Servers

**TLD** = Top-Level Domain (.com, .org, .net, .io, .uk, etc.)

TLD servers don't know individual IPs either. They know which **nameservers** are responsible for each domain:

```
.com TLD server:
  "google.com? Ask ns1.google.com (216.239.32.10)"
  "amazon.com? Ask ns1.p31.dynect.net"
  "github.com? Ask dns1.p08.nsone.net"
```

---

## Authoritative Name Servers

The **authoritative nameserver** has the actual, definitive answer. It's the final authority for a domain:

```
Google's authoritative nameserver (ns1.google.com):
  "google.com = 142.250.80.46"       ← THE answer
  "mail.google.com = 142.250.102.17"
  "drive.google.com = 142.250.80.46"
```

This is where the real records live. When you buy a domain and set up DNS records, they go on the authoritative nameserver.

---

## Recursive Resolver — The One Doing the Work

The recursive resolver is the server that does all the chasing. Your computer sends **one query**, and the resolver does all the work:

```
Your Computer: "What's google.com?"  (one question)

Resolver does:
  1. Ask root → get TLD referral
  2. Ask TLD → get authoritative referral
  3. Ask authoritative → get answer

Resolver to Your Computer: "142.250.80.46"  (one answer)
```

Your computer doesn't talk to root/TLD/authoritative servers directly. The recursive resolver handles the entire chain.

---

## DNS Caching — Why Lookups Are Usually Fast

The full lookup (root → TLD → authoritative) takes 50-200ms. But most lookups are instant because of **caching** at every level:

```
Level 1: Browser cache
  "I looked this up 2 minutes ago, use the cached answer"

Level 2: OS cache
  Every DNS answer is cached by your operating system

Level 3: Router cache
  Your home router caches answers for all devices

Level 4: Recursive resolver cache
  Your ISP's DNS or 8.8.8.8 caches answers for millions of users

Because of caching:
  First lookup:   50-200ms (full hierarchy walk)
  Second lookup:  0-5ms (cached — instant)
```

---

## TTL in DNS — How Long to Cache

Every DNS record has a **TTL** (Time To Live) — how many seconds it can be cached before it must be refreshed:

```
google.com.   300   IN   A   142.250.80.46

TTL = 300 seconds (5 minutes)

This means:
  "Cache this answer for up to 300 seconds.
   After that, ask again — the IP might have changed."
```

| TTL Value | Duration | Use Case |
|-----------|----------|----------|
| **60** | 1 minute | Frequently changing (failover, blue-green deployments) |
| **300** | 5 minutes | Normal websites |
| **3600** | 1 hour | Stable services |
| **86400** | 1 day | Very stable (rarely changes) |

### Low TTL vs High TTL

```
Low TTL (60 seconds):
  ✓ Changes propagate quickly (within 1 minute)
  ✗ More DNS queries (more load on DNS servers)

High TTL (86400 seconds):
  ✓ Fewer DNS queries (cached all day)
  ✗ Changes take up to 24 hours to propagate
```

---

## DNS Record Types

DNS doesn't just map names to IPs. There are many types of records:

### A Record

Maps a domain to an **IPv4 address**. The most common record type.

```
google.com.       A     142.250.80.46
myserver.com.     A     10.0.1.50
```

"When someone asks for google.com, give them this IPv4 address."

### AAAA Record

Maps a domain to an **IPv6 address** (concept 03). Called "quad-A."

```
google.com.       AAAA    2607:f8b0:4004:800::200e
```

### CNAME Record

An **alias** — points one name to another name (not an IP):

```
www.google.com.      CNAME   google.com.
blog.example.com.    CNAME   example.com.

When someone looks up www.google.com:
  → CNAME says "this is really google.com"
  → DNS then looks up google.com → gets the A record → 142.250.80.46
```

Think of CNAME as a redirect: "Don't look here, look THERE instead."

```
Rules for CNAME:
  ✓ Can point to another domain name
  ✗ CANNOT exist alongside other records for the same name
  ✗ CANNOT be at the zone apex (google.com itself — only subdomains)
```

### MX Record

**Mail Exchange** — tells email servers where to deliver email for a domain:

```
google.com.    MX    10  smtp.google.com.
google.com.    MX    20  smtp2.google.com.

"Email for @google.com should be sent to smtp.google.com.
 If that's down, try smtp2.google.com."

The number (10, 20) is PRIORITY — lower number = tried first.
```

### NS Record

**Name Server** — says which DNS servers are authoritative for a domain:

```
google.com.    NS    ns1.google.com.
google.com.    NS    ns2.google.com.
google.com.    NS    ns3.google.com.
google.com.    NS    ns4.google.com.

"If you want to know anything about google.com,
 ask ns1.google.com, ns2.google.com, etc."
```

### TXT Record

Holds **arbitrary text**. Used for verification and security:

```
google.com.    TXT    "v=spf1 include:_spf.google.com ~all"
example.com.   TXT    "google-site-verification=abc123..."

Common uses:
  - SPF (email authentication — who can send email for this domain)
  - DKIM (email signing)
  - Domain ownership verification (Google, AWS, etc.)
```

### SOA Record

**Start of Authority** — metadata about the DNS zone:

```
google.com.    SOA    ns1.google.com. dns-admin.google.com. (
                      2024010101  ; serial number
                      3600        ; refresh interval
                      900         ; retry interval
                      604800      ; expire time
                      300         ; minimum TTL
                      )
```

You rarely need to create SOA records manually — they're managed by your DNS provider.

### PTR Record

**Pointer** — the reverse of an A record. Maps an IP address back to a domain name:

```
46.80.250.142.in-addr.arpa.    PTR    google.com.

"If someone asks 'what domain is 142.250.80.46?' → google.com"
```

Used for **reverse DNS lookups** — common for email servers to verify sender identity.

### SRV Record

**Service** — tells clients where to find a specific service:

```
_sip._tcp.example.com.    SRV    10 60 5060 sipserver.example.com.

"The SIP service over TCP is at sipserver.example.com on port 5060"
```

---

## How to Query DNS Yourself

```bash
# Simple lookup — get the IP for a domain:
dig google.com

# Output (simplified):
# google.com.    300    IN    A    142.250.80.46

# Lookup specific record type:
dig google.com MX        # mail servers
dig google.com NS        # nameservers
dig google.com TXT       # text records
dig google.com AAAA      # IPv6 address

# Trace the full lookup path (root → TLD → authoritative):
dig +trace google.com

# Use a specific DNS server:
dig @8.8.8.8 google.com         # use Google's DNS
dig @1.1.1.1 google.com         # use Cloudflare's DNS

# Short answer only:
dig +short google.com
# 142.250.80.46

# Reverse lookup (IP → domain):
dig -x 142.250.80.46

# Alternative tool — nslookup:
nslookup google.com
nslookup google.com 8.8.8.8     # use specific DNS server

# Check your DNS cache (Linux):
resolvectl statistics
```

### Reading dig Output

```bash
dig google.com

;; QUESTION SECTION:
;google.com.                    IN      A        ← "What's the A record for google.com?"

;; ANSWER SECTION:
google.com.             300     IN      A       142.250.80.46
│                       │       │       │       │
│                       │       │       │       └── the IP address
│                       │       │       └── record type (A = IPv4)
│                       │       └── IN = Internet class
│                       └── TTL (300 seconds)
└── the domain name

;; Query time: 15 msec   ← how long the lookup took
;; SERVER: 8.8.8.8       ← which DNS server answered
```

---

## FQDN — Fully Qualified Domain Name

An **FQDN** is the complete domain name including all levels, ending with a dot:

```
FQDN:                Not FQDN:
google.com.           google.com   (missing trailing dot)
mail.google.com.      mail.google  (incomplete)
www.example.com.      www          (relative name)

The trailing dot (.) represents the ROOT of DNS.

Technically: www.google.com. = www (host) . google (domain) . com (TLD) . (root)
```

In practice, you usually omit the trailing dot — tools add it automatically.

---

## Zones and Zone Files

A **DNS zone** is a portion of the DNS namespace managed by a specific administrator. A **zone file** contains all the DNS records for that zone:

```
Zone: example.com
Zone File Contents:

$TTL 300
example.com.       SOA   ns1.example.com. admin.example.com. (
                         2024010101 3600 900 604800 300)
example.com.       NS    ns1.example.com.
example.com.       NS    ns2.example.com.
example.com.       A     93.184.216.34
www.example.com.   CNAME example.com.
mail.example.com.  A     93.184.216.50
example.com.       MX    10 mail.example.com.
```

---

## Public DNS vs Private DNS

### Public DNS

Resolves names on the **public internet** — anyone can query:

```
Public DNS servers:
  8.8.8.8 / 8.8.4.4      — Google Public DNS
  1.1.1.1 / 1.0.0.1      — Cloudflare DNS (fast)
  9.9.9.9                 — Quad9 (security-focused)
  208.67.222.222          — OpenDNS
```

### Private DNS

Resolves names only within a **private network** — not accessible from the internet:

```
Company internal DNS:
  server1.internal.company.com → 10.0.1.50
  database.internal.company.com → 10.0.2.30
  api.prod.internal.company.com → 10.0.3.100

These names only work inside the company network.
From outside (home, coffee shop), they don't resolve.
```

---

## Common DNS Problems

### "DNS not resolving" — Can't reach websites by name

```bash
# Diagnosis:
ping 8.8.8.8          # works? → network is fine, DNS is broken
ping google.com        # fails? → DNS can't resolve the name

# Fix: Try a different DNS server
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

### "DNS propagation delay" — Changed a record but old IP still shows

```
You changed:  example.com A → 10.0.1.100 (new server)
But users still see the old IP (10.0.1.50)

Why: Cached copies haven't expired yet (TTL hasn't passed)
Fix: Wait for TTL to expire, or lower TTL BEFORE making the change
```

### "NXDOMAIN" — Domain doesn't exist

```bash
dig doesntexist12345.com
# status: NXDOMAIN
# This domain is not registered or has no DNS records
```

### "SERVFAIL" — DNS server can't reach authoritative server

```
The recursive resolver tried to look up the domain
but couldn't reach the authoritative nameserver.
Usually a temporary issue — retry or try a different resolver.
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **DNS** | Translates domain names to IP addresses |
| **Recursive resolver** | The server that does the full lookup for you |
| **Root server** | Top of the hierarchy — points to TLD servers |
| **TLD server** | Knows which nameservers handle .com, .org, etc. |
| **Authoritative server** | Has the actual answer — the definitive record |
| **A record** | Maps name → IPv4 address |
| **AAAA record** | Maps name → IPv6 address |
| **CNAME** | Alias — points one name to another name |
| **MX record** | Where to deliver email for a domain |
| **NS record** | Which servers are authoritative for a domain |
| **TTL** | How long a DNS answer can be cached |
| **FQDN** | Complete domain name (www.google.com.) |
| **Zone** | Portion of DNS managed by one administrator |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| DNS service | **Route 53** — AWS's DNS service (named after UDP port 53) |
| A record | Route 53 A records point domains to EC2, ALB, NLB IPs |
| CNAME | Route 53 CNAME records — but you can also use **Alias records** (AWS-specific, works at zone apex) |
| Alias record | AWS extension — like CNAME but works at the zone apex (example.com → ALB) and is free |
| Private DNS | **Route 53 Private Hosted Zones** — DNS that only works inside your VPC |
| MX record | Route 53 MX records for email routing |
| TTL | You set TTL on Route 53 records — low for failover, high for stability |
| Health checks | Route 53 can check if a server is healthy and route traffic away from unhealthy ones |
| DNS resolution | VPC setting: "Enable DNS resolution" must be ON for private hosted zones |
| DNS hostnames | VPC setting: "Enable DNS hostnames" gives EC2 instances automatic DNS names |
| Split-horizon DNS | Same domain name resolves differently inside vs outside the VPC (public + private hosted zones) |

```
Route 53 example:

Public Hosted Zone (api.myapp.com):
  api.myapp.com    A    ALIAS → my-alb-1234.us-east-1.elb.amazonaws.com

Private Hosted Zone (internal.myapp.com):
  db.internal.myapp.com    A    10.0.2.50
  cache.internal.myapp.com A    10.0.3.30

  → These only resolve from INSIDE the VPC
  → From the internet: NXDOMAIN (doesn't exist)
```

---

> **Previous:** [06 — How Data Travels](06-how-data-travels.md)
> **Next:** [08 — Firewalls](08-firewalls.md)
