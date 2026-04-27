# 10 — SSL/TLS (Encryption in Transit)

How HTTPS encrypts your data — the TLS handshake, certificates, public/private keys, symmetric vs asymmetric encryption, certificate authorities, and the chain of trust.

**Prerequisites:** Concept 05 (HTTPS port 443, TCP), Concept 06 (how data travels), Concept 07 (DNS).

> **Deep dive:** The [SSL.md](SSL.md) reference file in this folder covers certificates, installation, algorithms, and troubleshooting in exhaustive detail. This file teaches you the **concepts** so that reference makes sense.

---

## Table of Contents

- [Why Encryption Matters](#why-encryption-matters)
- [What Happens Without Encryption](#what-happens-without-encryption)
- [SSL vs TLS — The Names](#ssl-vs-tls-the-names)
- [What TLS Actually Does](#what-tls-actually-does)
- [The Three Problems TLS Solves](#the-three-problems-tls-solves)
- [Symmetric Encryption — One Key](#symmetric-encryption-one-key)
- [Asymmetric Encryption — Two Keys](#asymmetric-encryption-two-keys)
- [Why TLS Uses Both](#why-tls-uses-both)
- [Certificates — Proving Identity](#certificates-proving-identity)
- [What's Inside a Certificate](#whats-inside-a-certificate)
- [Certificate Authorities (CAs)](#certificate-authorities-cas)
- [The Chain of Trust](#the-chain-of-trust)
- [The TLS Handshake — Step by Step](#the-tls-handshake-step-by-step)
- [After the Handshake — Encrypted Communication](#after-the-handshake-encrypted-communication)
- [The Lock Icon in Your Browser](#the-lock-icon-in-your-browser)
- [Certificate Types — DV, OV, EV](#certificate-types-dv-ov-ev)
- [Self-Signed Certificates](#self-signed-certificates)
- [Certificate Expiration and Renewal](#certificate-expiration-and-renewal)
- [mTLS — Mutual TLS](#mtls-mutual-tls)
- [TLS Versions](#tls-versions)
- [Common TLS Problems](#common-tls-problems)
- [Summary](#summary)
- [How This Connects to AWS](#how-this-connects-to-aws)

---

## Why Encryption Matters

When data travels across a network (concept 06), it passes through many routers, switches, and networks. **Anyone along that path could read or modify the data** unless it's encrypted.

```
WITHOUT encryption (HTTP — port 80):

  You: "My password is hunter42"
       ↓
  Your Wi-Fi → router → ISP → internet → server
       ↑
  Anyone on this path can read: "My password is hunter42"
  Hackers on public Wi-Fi, ISP employees, compromised routers — all can see it.


WITH encryption (HTTPS — port 443):

  You: "My password is hunter42"
  TLS encrypts it → "a7$kL!m9xQ#2pR..."
       ↓
  Your Wi-Fi → router → ISP → internet → server
       ↑
  Anyone on this path sees: "a7$kL!m9xQ#2pR..." (gibberish)
  Only the server can decrypt it back to "My password is hunter42"
```

---

## What Happens Without Encryption

Real attacks that happen on unencrypted connections:

```
1. EAVESDROPPING (passive)
   Attacker sits on the same Wi-Fi (coffee shop, hotel, airport)
   and reads everything you send:
     - Passwords
     - Credit card numbers
     - Emails
     - Private messages

2. MAN-IN-THE-MIDDLE (active)
   Attacker intercepts AND MODIFIES data:
     You → "Transfer $100 to Alice" → Attacker → "Transfer $10,000 to Hacker" → Bank
     Neither you nor the bank knows the message was changed.

3. IMPERSONATION
   Attacker pretends to BE the website:
     You think you're talking to your bank.
     You're actually talking to an attacker's server.
     You enter your username/password → attacker steals them.
```

TLS prevents all three of these attacks.

---

## SSL vs TLS — The Names

```
SSL (Secure Sockets Layer):
  - Created by Netscape in the 1990s
  - SSL 1.0 (never released), SSL 2.0 (1995), SSL 3.0 (1996)
  - ALL versions of SSL are now BROKEN and deprecated
  - DO NOT USE SSL

TLS (Transport Layer Security):
  - The replacement for SSL
  - TLS 1.0 (1999), TLS 1.1 (2006), TLS 1.2 (2008), TLS 1.3 (2018)
  - TLS 1.2 and TLS 1.3 are the current standards

Everyone says "SSL" out of habit, but they mean TLS.
"SSL certificate" = TLS certificate. Same thing.
```

---

## What TLS Actually Does

TLS is a protocol that sits **between TCP and HTTP** (or any application protocol):

```
Without TLS:
  Application (HTTP) → TCP → IP → Network

With TLS:
  Application (HTTP) → TLS → TCP → IP → Network

  HTTP sends data to TLS.
  TLS encrypts it.
  TLS hands encrypted data to TCP.
  TCP sends it over the network.

On the receiving end:
  TCP receives encrypted data → TLS decrypts → HTTP gets plain text
```

TLS works at **Layer 5-6** (between transport and application). When HTTP uses TLS, we call it **HTTPS**.

---

## The Three Problems TLS Solves

| Problem | What It Means | How TLS Solves It |
|---------|--------------|-------------------|
| **Confidentiality** | Nobody can READ the data | Encryption — data is scrambled |
| **Integrity** | Nobody can MODIFY the data | Message authentication codes (MACs) detect changes |
| **Authentication** | You're talking to the RIGHT server | Certificates prove the server's identity |

```
Without TLS:
  ✗ Anyone can read your data (no confidentiality)
  ✗ Anyone can change your data (no integrity)
  ✗ You can't be sure who you're talking to (no authentication)

With TLS:
  ✓ Data is encrypted — only you and the server can read it
  ✓ Data is integrity-checked — any modification is detected
  ✓ Certificate proves the server is really google.com, not an impersonator
```

---

## Symmetric Encryption — One Key

**Symmetric encryption** uses the **same key** to encrypt and decrypt:

```
Key: "supersecret123"

Encrypt:
  "Hello World" + key → "a7$kL!m9xQ#2"

Decrypt:
  "a7$kL!m9xQ#2" + key → "Hello World"

Same key for both. Like a door lock — same key locks and unlocks.
```

```
Alice and Bob both have the SAME key:

Alice: encrypt("Hello", key) → "a7$kL" → sends to Bob
Bob:   decrypt("a7$kL", key) → "Hello"

✓ Very fast (used for bulk data)
✗ Problem: How do Alice and Bob share the key safely?
  If they send the key over the network, an attacker can steal it!
```

Algorithms: **AES-128, AES-256** (the standard — very fast, very secure).

---

## Asymmetric Encryption — Two Keys

**Asymmetric encryption** uses a **pair of keys**: one public, one private.

```
Key pair:
  Public key:  can be shared with everyone
  Private key: kept SECRET, never shared

Rule:
  What one key encrypts, ONLY the other key can decrypt.

Encrypt with public key → only private key can decrypt
Encrypt with private key → only public key can decrypt
```

```
The Mailbox Analogy:

  Public key = the mailbox slot (anyone can put a letter in)
  Private key = the mailbox key (only you can open and read letters)

  Anyone can encrypt a message WITH your public key.
  Only you can decrypt it WITH your private key.
```

```
Alice wants to send a secret message to Bob:

1. Bob publishes his PUBLIC key (everyone can see it)
2. Alice encrypts: encrypt("Hello", Bob's public key) → "x9#mK"
3. Alice sends "x9#mK" to Bob
4. Bob decrypts: decrypt("x9#mK", Bob's private key) → "Hello"

Even if an attacker intercepts "x9#mK", they CAN'T decrypt it
because they don't have Bob's PRIVATE key.
```

Algorithms: **RSA, ECDSA, Ed25519**.

```
✓ Solves the key-sharing problem (public key is... public)
✗ SLOW — 100-1000x slower than symmetric encryption
✗ Not practical for encrypting bulk data
```

---

## Why TLS Uses Both

TLS uses asymmetric encryption to **safely exchange a symmetric key**, then uses that symmetric key for all the actual data:

```
Step 1: Asymmetric encryption (slow but secure)
  Client and server use public/private keys to negotiate
  a shared "session key" — a symmetric key.
  
  An attacker can't steal this session key because they
  don't have the server's private key.

Step 2: Symmetric encryption (fast)
  Both sides now have the same session key.
  All data is encrypted/decrypted with this fast symmetric key.

This gives you:
  ✓ Security of asymmetric (safe key exchange)
  ✓ Speed of symmetric (fast data transfer)
```

```
Analogy:
  You meet someone new. You exchange phone numbers using business cards (slow, formal).
  Now you have each other's numbers.
  All future communication is by phone call (fast, easy).

  Asymmetric = exchanging business cards (one-time setup)
  Symmetric = phone calls (ongoing fast communication)
```

---

## Certificates — Proving Identity

A certificate is a **digital document** that says: "This public key belongs to this website."

```
Without certificates:
  You connect to "google.com"
  A server responds with a public key
  But HOW DO YOU KNOW it's actually Google?
  An attacker could intercept and give you THEIR public key!
  
  You: "Hello google.com, give me your public key"
  Attacker (pretending to be Google): "Here's MY public key"
  You encrypt with attacker's key → attacker can read everything

With certificates:
  Google's server sends a CERTIFICATE containing:
    - Google's public key
    - Signed by a trusted Certificate Authority (CA)
  
  Your browser checks: "Is this certificate signed by a CA I trust? YES"
  Now you KNOW the public key really belongs to Google.
```

---

## What's Inside a Certificate

```
Certificate for google.com:
  ├── Subject: CN=google.com         ← who this certificate is for
  ├── Issuer: Let's Encrypt           ← who issued it (the CA)
  ├── Valid From: 2024-01-01           ← start date
  ├── Valid Until: 2024-03-31          ← expiration date
  ├── Public Key: 04:a5:3b:7f:...     ← the server's public key
  ├── Subject Alternative Names:       ← other domains covered
  │     google.com
  │     www.google.com
  │     *.google.com
  └── Digital Signature: 3a:8b:1c:... ← CA's signature (proof of authenticity)
```

The **digital signature** is the critical part. It's created using the CA's private key. Your browser verifies it using the CA's public key. If it checks out, the certificate is legit.

---

## Certificate Authorities (CAs)

A **CA** is a trusted organization that issues certificates. They verify that you actually own the domain before issuing a certificate.

```
The Passport Office Analogy:
  You want a passport (certificate) for your domain.
  You go to the passport office (CA).
  They verify your identity (prove you own the domain).
  They issue a passport (signed certificate).
  Anyone who trusts the passport office trusts your passport.
```

### Well-Known CAs

| CA | Notes |
|----|-------|
| **Let's Encrypt** | Free, automated, most popular |
| **DigiCert** | Enterprise, paid |
| **Sectigo (Comodo)** | Paid, widely used |
| **Google Trust Services** | Google's own CA |
| **Amazon (ACM)** | Free certificates for AWS resources |

### How Your Browser Trusts CAs

Your browser (and your OS) comes with a pre-installed list of **trusted root CAs** — about 100-150 of them. Any certificate signed by one of these CAs is automatically trusted.

```bash
# See trusted CAs on Linux:
ls /etc/ssl/certs/

# See trusted CAs on Mac:
# Open Keychain Access → System Roots → Certificates
```

---

## The Chain of Trust

Certificates form a **chain** from your website up to a trusted root CA:

```
Your Browser's Trust Store:
  "I trust these Root CAs: DigiCert, Let's Encrypt, ..."

Certificate Chain:
  ┌─────────────────────────┐
  │  Root CA Certificate    │  ← Pre-installed in your browser/OS
  │  (DigiCert Root)        │     TRUSTED by default
  └───────────┬─────────────┘
              │ signed
              ▼
  ┌─────────────────────────┐
  │  Intermediate CA Cert   │  ← Signed by the Root CA
  │  (DigiCert SHA2)        │
  └───────────┬─────────────┘
              │ signed
              ▼
  ┌─────────────────────────┐
  │  Server Certificate     │  ← Signed by the Intermediate CA
  │  (google.com)           │     This is what the server sends you
  └─────────────────────────┘

Verification:
  1. Browser receives server cert (google.com)
  2. "Who signed this? DigiCert SHA2 Intermediate"
  3. "Who signed the intermediate? DigiCert Root"
  4. "Do I trust DigiCert Root? YES (it's in my trust store)"
  5. → Entire chain verified → Certificate TRUSTED ✓
```

### Why Intermediate CAs?

Root CAs keep their private keys **offline** in secure vaults. If a root key were compromised, ALL certificates under it would be untrustworthy. Intermediate CAs handle day-to-day signing so the root key stays safe.

---

## The TLS Handshake — Step by Step

When your browser connects to `https://google.com`, this happens before any data is sent:

```
Client (Browser)                           Server (google.com)
       │                                          │
       │ 1. ClientHello                            │
       │    "I support TLS 1.3, these ciphers..."  │
       │ ─────────────────────────────────────────→ │
       │                                          │
       │ 2. ServerHello                            │
       │    "Let's use TLS 1.3 with AES-256"       │
       │    + Server Certificate                   │
       │    + Server's public key                  │
       │ ←───────────────────────────────────────── │
       │                                          │
       │ 3. Client verifies certificate:           │
       │    - Is it expired? No ✓                  │
       │    - Is the domain correct? Yes ✓         │
       │    - Is the CA trusted? Yes ✓             │
       │    - Is the signature valid? Yes ✓        │
       │                                          │
       │ 4. Key Exchange                           │
       │    Client and server compute a shared     │
       │    session key using Diffie-Hellman       │
       │    (even an eavesdropper can't derive it) │
       │ ←──────────────────────────────────────→  │
       │                                          │
       │ 5. Finished                               │
       │    Both sides confirm: "We have the       │
       │    same session key. Handshake complete."  │
       │ ←──────────────────────────────────────→  │
       │                                          │
       │ ═══ ALL DATA NOW ENCRYPTED ═══            │
       │                                          │
       │ 6. GET /search?q=cats HTTP/1.1            │
       │    (encrypted with session key)           │
       │ ─────────────────────────────────────────→ │
       │                                          │
       │ 7. HTTP/1.1 200 OK                        │
       │    <html>...results...</html>             │
       │    (encrypted with session key)           │
       │ ←───────────────────────────────────────── │
```

The entire handshake happens in **1-2 round trips** (TLS 1.3 does it in just 1). Total time: 20-100ms.

---

## After the Handshake — Encrypted Communication

Once the handshake is done, **every byte** between client and server is encrypted with the session key:

```
What you send: "My password is hunter42"
What goes on the wire: "a7$kL!m9xQ#2pR8vN3..."

What the server sends: "<html>Your account balance: $5,000</html>"
What goes on the wire: "mK9#x2Lp$qR7vN1..."

Anyone intercepting traffic sees ONLY encrypted gibberish.
The session key exists only in memory on the client and server.
When the connection closes, the key is discarded.
```

---

## The Lock Icon in Your Browser

```
🔒 Lock icon = HTTPS connection with a valid certificate

What it means:
  ✓ Connection is encrypted
  ✓ Certificate is valid (not expired, correct domain, trusted CA)
  ✓ Nobody between you and the server can read your data

What it does NOT mean:
  ✗ The website is safe or trustworthy
  ✗ The website won't scam you
  ✗ The website is free of malware

  A phishing site can ALSO have a lock icon!
  Lock = encrypted connection, NOT "this site is safe"
```

---

## Certificate Types — DV, OV, EV

| Type | Verification | Cost | Use Case |
|------|-------------|------|----------|
| **DV (Domain Validation)** | Prove you own the domain | Free-$50/yr | Personal sites, blogs, most websites |
| **OV (Organization Validation)** | Prove domain ownership + company identity | $100-500/yr | Business websites |
| **EV (Extended Validation)** | Extensive verification of company | $500-2000/yr | Banks, large enterprises (shows company name in some browsers) |

```
DV (most common):
  CA checks: "Do you control this domain?" → YES → certificate issued
  Method: Add a DNS TXT record, upload a file, or respond to email

OV:
  CA checks: "Do you control the domain AND is your company real?"
  → Verify business registration, phone call, etc.

EV:
  CA checks: "Extensive verification — legal entity, physical address, etc."
  → Takes days/weeks. Shows company name in browser.
```

For most use cases, **DV is sufficient.** Let's Encrypt provides free DV certificates.

---

## Self-Signed Certificates

A **self-signed certificate** is one where you are your own CA — you sign it yourself:

```
Normal certificate:
  google.com → signed by DigiCert → browser trusts DigiCert → TRUSTED ✓

Self-signed certificate:
  myserver.local → signed by myserver.local → browser doesn't trust myserver.local → NOT TRUSTED ✗

Browser shows: "Your connection is not private" / "Certificate not trusted"
```

### When Self-Signed Is OK

```
✓ Development/testing (localhost)
✓ Internal tools where you control all the clients
✓ Learning and experimentation

✗ NEVER for public-facing websites
✗ NEVER for production APIs that external clients connect to
```

---

## Certificate Expiration and Renewal

Certificates have an **expiration date** (typically 90 days for Let's Encrypt, 1 year for paid CAs).

```
Expired certificate:
  Your browser checks: "Valid until 2024-03-31. Today is 2024-04-01."
  → "NET::ERR_CERT_DATE_INVALID"
  → Users see a scary warning page
  → They leave your site

Prevention:
  → Automate renewal (Let's Encrypt + certbot auto-renews)
  → Monitor certificate expiration dates
  → Set up alerts 30 days before expiry
```

---

## mTLS — Mutual TLS

Normal TLS: **only the server** proves its identity with a certificate. The client is anonymous.

**mTLS** (Mutual TLS): **both sides** present certificates. The server verifies the client's certificate too.

```
Normal TLS:
  Client: "Prove you're google.com" → Server shows certificate ✓
  Server: "Who are you?" → Client: "Just a browser, no cert" → OK

mTLS:
  Client: "Prove you're api.company.com" → Server shows certificate ✓
  Server: "Prove you're an authorized client" → Client shows certificate ✓
  Both verified → connection established

Use cases:
  - Service-to-service communication (microservices)
  - Zero trust networks
  - Istio service mesh (every pod has a certificate)
  - API clients that need strong authentication
```

---

## TLS Versions

| Version | Year | Status |
|---------|------|--------|
| SSL 2.0 | 1995 | **BROKEN** — do not use |
| SSL 3.0 | 1996 | **BROKEN** (POODLE attack) — do not use |
| TLS 1.0 | 1999 | **Deprecated** — being phased out |
| TLS 1.1 | 2006 | **Deprecated** — being phased out |
| TLS 1.2 | 2008 | **Current standard** — widely supported |
| TLS 1.3 | 2018 | **Best** — faster handshake, stronger security |

```
TLS 1.3 improvements over 1.2:
  ✓ Faster: 1 round trip instead of 2 (0-RTT resumption possible)
  ✓ Removed old/weak cipher suites
  ✓ Simpler (less room for misconfiguration)
  ✓ Forward secrecy is mandatory
```

---

## Common TLS Problems

### Certificate expired

```
Error: NET::ERR_CERT_DATE_INVALID
Fix: Renew the certificate. Set up auto-renewal with certbot.
```

### Certificate name mismatch

```
Error: NET::ERR_CERT_COMMON_NAME_INVALID
Cause: Certificate is for "example.com" but you connected to "www.example.com"
Fix: Certificate must include all domain names (use SAN — Subject Alternative Names)
```

### Self-signed certificate

```
Error: NET::ERR_CERT_AUTHORITY_INVALID
Cause: Certificate not signed by a trusted CA
Fix: Get a certificate from a real CA (Let's Encrypt is free)
```

### Certificate chain incomplete

```
Error: Unable to verify certificate
Cause: Server only sends its own cert, not the intermediate CA cert
Fix: Configure server to send the full certificate chain
```

### Mixed content

```
Warning: "This page includes resources from insecure sources"
Cause: HTTPS page loads images/scripts over HTTP
Fix: Change all resource URLs to HTTPS
```

---

## Summary

| Term | What It Means |
|------|--------------|
| **TLS** | Protocol that encrypts data between client and server |
| **HTTPS** | HTTP + TLS = encrypted web traffic (port 443) |
| **Certificate** | Digital document proving a server's identity + containing its public key |
| **CA** | Certificate Authority — trusted organization that issues certificates |
| **Public key** | Can be shared — used to encrypt data or verify signatures |
| **Private key** | NEVER shared — used to decrypt data or create signatures |
| **Symmetric encryption** | One key for both encrypt and decrypt (fast — AES) |
| **Asymmetric encryption** | Key pair — public encrypts, private decrypts (slow — RSA) |
| **Chain of trust** | Root CA → Intermediate CA → Server cert |
| **TLS handshake** | Initial negotiation that establishes the encrypted connection |
| **Session key** | Symmetric key created during handshake — used for actual data encryption |
| **mTLS** | Both client AND server present certificates |
| **Self-signed** | Certificate you sign yourself — not trusted by browsers |

---

## How This Connects to AWS

| This Concept | In AWS |
|-------------|--------|
| **Certificates** | **AWS Certificate Manager (ACM)** — free public certificates for ALB, NLB, CloudFront, API Gateway |
| **TLS termination** | ALB terminates TLS — decrypts traffic, forwards HTTP to targets |
| **End-to-end TLS** | ALB → re-encrypts → target (backend also has a certificate) |
| **Private certificates** | **ACM Private CA** — issue certificates for internal services |
| **mTLS** | API Gateway supports mTLS. Istio on EKS does mTLS between pods |
| **Certificate renewal** | ACM auto-renews certificates for AWS resources |
| **TLS version** | ALB/NLB/CloudFront let you set minimum TLS version (use 1.2+) |
| **Cipher suites** | ALB security policies control which ciphers are allowed |

```
AWS ALB TLS setup:

  Client ──── HTTPS (TLS 1.3) ────→ ALB (has ACM certificate)
                                      │
                                      │ TLS terminated here
                                      │ ALB decrypts the traffic
                                      │
                                      │ ── HTTP (port 80) ──→ EC2 instances
                                      │                       (private subnet)
                                      │
                                      │ OR for end-to-end encryption:
                                      │ ── HTTPS (port 443) ──→ EC2 instances
                                      │                         (backend cert)

ACM certificate:
  - Free for ALB, NLB, CloudFront, API Gateway
  - Auto-renewed (no expiry worries)
  - Cannot be downloaded (stays within AWS)
  - Validates via DNS (add a CNAME record to Route 53)
```

```
Istio mTLS on EKS:

  Pod A ←── mTLS (auto-certificates) ──→ Pod B
  
  Istio automatically:
  1. Issues a certificate to every pod (via its CA)
  2. Rotates certificates regularly
  3. Enforces mTLS between all services
  4. No application code changes needed
```

---

> **Previous:** [09 — NAT](09-nat.md)
> **Next:** [11 — VPN](11-vpn.md)
