# Network Engineering Master Notes — Detailed Edition

---

# Part 1: DNS Architecture & Packet Mechanics

---

## 1.1 How Domain Names Map to IP Addresses — The Complete Picture

When you type `api.example.com` into a browser or an application makes an API call, the very first thing that must happen is converting that human-readable hostname into a machine-routable IP address. This is the job of the Domain Name System (DNS). Think of DNS as the **phonebook of the internet** — it translates names into numbers.

But unlike a simple local phonebook, DNS is a **distributed, hierarchical, caching system** spanning thousands of servers worldwide. Understanding exactly how a query travels through this system is fundamental to debugging latency, migrations, and outages.

### The Two Types of Queries: Recursive vs Iterative

Before diving into the flow, you must understand the two ways DNS queries work. This distinction is the most commonly tested concept in networking interviews and is the foundation of everything that follows.

---

### Recursive Query

**Definition**: The client makes **one single request** to a resolver and waits patiently until the resolver returns the final answer (or an error like NXDOMAIN). The client does zero work.

**Analogy**: You call a customer service helpline and say "Find me John's phone number." The agent does all the looking up — checks the directory, calls other departments if needed — and then calls you back with the number. You never leave your chair.

**Who makes recursive queries**:
- Your web browser (Chrome, Safari, Firefox)
- Your operating system's resolver (`systemd-resolved` on Linux, `mDNSResponder` on macOS)
- Any application that calls `getaddrinfo()` or `gethostbyname()` in C/Python/Go

**The resolver's obligation**: Once a resolver accepts a recursive query (`RD=1` in the DNS header), it **must** either return the final IP address or return a definitive error code (SERVFAIL, NXDOMAIN, REFUSED). It cannot say "I don't know, try someone else."

**Example**:
```
Client asks:  "What is the IP of api.example.com?"
Resolver:     "It is 93.184.216.34"    ← Final answer in ONE response
```

---

### Iterative Query

**Definition**: The client (or resolver acting on behalf of the client) asks each DNS server "Do you know the answer?" Each server either gives the final answer **or** provides a referral pointing to the next best server to ask. The client must follow the referrals step by step.

**Analogy**: You ask a receptionist "Where is John's office?" She says "I don't know, but he's on the 3rd floor in the Engineering wing." You go to the 3rd floor. You ask someone there "Where is John?" They say "He's in room 304." You walk to room 304. Each step gives you a better answer, but you must physically move yourself.

**Who makes iterative queries**: DNS servers talking to other DNS servers (resolver → root → TLD → authoritative)

**The key insight**: No single server ever has to know the entire internet's namespace. Each server only needs to know a small piece and can point to the next server that knows more. This is why the system scales.

**Example**:
```
Resolver asks Root:        "What is api.example.com?"
Root answers:              "I don't know, but ask .com TLD server at 192.5.6.30"

Resolver asks TLD (.com):  "What is api.example.com?"
TLD answers:               "I don't know, but ask ns1.awsdns.com at 205.251.192.36"

Resolver asks Authoritative: "What is api.example.com?"
Authoritative answers:      "93.184.216.34 (TTL=300s)"    ← Final answer
```

### Why Both Exist

- **Recursive** exists because end devices (browsers, apps) are simple — they should never have to implement the entire DNS hierarchy logic
- **Iterative** exists because DNS servers must be scalable — no single server can store all records, so they delegate

**In a real resolution**, the recursive resolver on your behalf performs **iterative queries** against the hierarchy. The client sees one recursive query; behind the scenes, the resolver executes multiple iterative queries.

---

## 1.2 The Complete Recursive Resolution Flow — Step by Step with Real IPs

Let's trace what happens when `curl https://api.github.com` is executed from a laptop. This is not a simplified diagram — this is what actually happens at the packet level.

### Step 0: Browser / OS Cache Check

```bash
# Check if the OS resolver cache already has this answer
# Linux (systemd-resolved):
resolvectl statistics

# Check /etc/hosts first (always checked before any network query):
cat /etc/hosts
# Possible entry: 127.0.0.1   api.github.dev  ← Local override, no DNS needed
```

The browser maintains its own DNS cache (Chrome caches ~4000 entries). The OS maintains a resolver cache (`/etc/hosts` + systemd-resolved cache). If the answer is found here **and TTL hasn't expired**, the query never leaves the machine. This is called a **cache hit** and takes microseconds.

**If not found** → **Cache Miss** → Query proceeds to the recursive resolver.

### Step 1: Query the Recursive Resolver

The OS sends a **recursive query** (`RD=1`) to the configured resolver — typically:
- Your ISP's resolver (e.g., Comcast: 75.75.75.75)
- Google DNS: 8.8.8.8
- Cloudflare: 1.1.1.1
- OpenDNS: 208.67.222.222

**UDP packet sent** (port 53, 512 bytes or less, ID assigned by resolver):
```
DNS Header:
  Transaction ID: 0x3A2F
  Flags: 0x0100 (Standard query, Recursion Desired)
  Questions: 1
  Answer RRs: 0
  Authority RRs: 0
  Additional RRs: 0

Question Section:
  api.github.com.  IN  A
```

The resolver receives this and now must perform the iterative lookup on behalf of the client.

### Step 2: Query the Root Server (`.`)

The resolver picks a root server from the hardcoded root hint file (`/etc/bind/db.root` or `named.cache`). There are 13 root server IP addresses (a.root-servers.net through m.root-servers.net), each announced via BGP anycast from 1700+ physical locations globally.

**Resolver sends** to 198.41.0.4 (a.root-servers.net):
```
Question: api.github.com.  IN  A
```

**Root server parses** the domain name **right-to-left**:
- It sees `com` as the rightmost label before the root dot
- It searches its in-memory **Root Zone File** for `com.`

**Root Zone File snippet** (this entire file is ~2-3 MB, containing every TLD):
```dns
; --- Root Zone File ---
com.                 IN   NS   a.gtld-servers.net.
com.                 IN   NS   b.gtld-servers.net.
com.                 IN   NS   c.gtld-servers.net.
...
a.gtld-servers.net.  IN   A    192.5.6.30
a.gtld-servers.net.  IN   AAAA 2001:503:a83e::2:30
b.gtld-servers.net.  IN   A    192.33.14.30
...
in.                  IN   NS   ns1.registry.in.
in.                  IN   NS   ns2.registry.in.
ns1.registry.in.     IN   A    37.209.192.9
...
```

**Root server finds** `com.` in its table. It returns a **REFERRAL** (not an answer — it doesn't have the final IP):

```
DNS Header:
  Flags: 0x8100 (Response, Recursion Available, AA=0 for non-authoritative)
  Questions: 1
  Answer RRs: 0          ← No A record here
  Authority RRs: 1       ← The NS referral
  Additional RRs: 3      ← Glue records (A/AAAA for the nameservers)

Authority Section:
  com.  IN  NS  a.gtld-servers.net.
  com.  IN  NS  b.gtld-servers.net.
  com.  IN  NS  c.gtld-servers.net.

Additional Section (Glue Records):
  a.gtld-servers.net.  IN  A     192.5.6.30
  b.gtld-servers.net.  IN  A     192.33.14.30
  c.gtld-servers.net.  IN  A     192.26.92.30
```

**Critical detail — Glue Records**: Without glue records, there would be a circular dependency. If `a.gtld-servers.net` were the nameserver for `.com`, you'd need to know its IP to query it — but its IP is only known by querying `.com`. Glue records break this circle by providing the IP addresses alongside the NS records in the Additional section.

**Why the root server doesn't know about `api.github.com`**: The root server's job is solely to answer "Who manages .com?" It has zero information about domains under .com. It only maps TLD strings to their registry nameservers.

### Step 3: Query the TLD Server (.com)

The resolver now queries one of the returned glue IPs — say 192.5.6.30 (a.gtld-servers.net, operated by Verisign).

**Resolver sends** to 192.5.6.30:
```
Question: api.github.com.  IN  A
```

**TLD server parses** the query:
- Strips the `com.` label
- What remains is `api.github.com.` — but wait, the TLD server for .com only knows the authoritative nameservers for `.com` domains, not individual subdomains. It doesn't have records for `api.github.com`. What it does know is: "The nameservers responsible for `github.com` are..."

The `.com` TLD server returns another **REFERRAL**:

```
Authority Section:
  github.com.  IN  NS  ns-1234.awsdns-27.org.
  github.com.  IN  NS  ns-567.awsdns-15.com.

Additional Section (Glue):
  ns-1234.awsdns-27.org.  IN  A    205.251.198.123
  ns-567.awsdns-15.com.    IN  A    205.251.193.56
```

**The TLD server also doesn't know `api.github.com`.** It just says "The company that manages github.com uses these nameservers at AWS Route 53."

### Step 4: Query the Authoritative Nameserver (Route 53)

The resolver now queries one of the returned authoritative nameservers — say 205.251.198.123.

**Resolver sends** to 205.251.198.123:
```
Question: api.github.com.  IN  A
```

**Authoritative server responds** with the actual data. This server holds the zone file for `github.com`:

```
DNS Header:
  Flags: 0x8180 (Response, AA=1 Authoritative Answer)
  Questions: 1
  Answer RRs: 1
  Authority RRs: 0
  Additional RRs: 0

Answer Section:
  api.github.com.  300  IN  A  140.82.113.4
```

**The `AA` (Authoritative Answer) flag is set** — this tells the resolver "I am the authoritative source for this domain and this answer is definitive."

The resolver caches this result for **300 seconds** (the TTL value in the answer) and returns it to the client.

### Step 5: Client Receives the IP

The client (`curl`) now has `140.82.113.4` and initiates a TCP connection on port 443 for HTTPS.

### Complete Flow Summary

```
YOUR LAPTOP                          RECURSIVE RESOLVER              ROOT                TLD (.com)         AUTHORITATIVE
    │                                      │                              │                    │                    │
    │──(1) Recursive Query──►              │                              │                    │                    │
    │   "api.github.com A?"                │                              │                    │                    │
    │                                      │──(2) Iterative──────────────►│                    │                    │
    │                                      │   "api.github.com A?"        │                    │                    │
    │                                      │                              │                    │                    │
    │                                      │◄─(3) Referral────────────────│                    │                    │
    │                                      │   "Go ask .com TLD at        │                    │                    │
    │                                      │    192.5.6.30"               │                    │                    │
    │                                      │                              │                    │                    │
    │                                      │──(4) Iterative─────────────────────────────────────►│                    │
    │                                      │   "api.github.com A?"        │                    │                    │
    │                                      │                              │                    │                    │
    │                                      │◄─(5) Referral────────────────────────────────────────│                    │
    │                                      │   "Go ask awsdns at          │                    │                    │
    │                                      │    205.251.198.123"          │                    │                    │
    │                                      │                              │                    │                    │
    │                                      │──(6) Iterative─────────────────────────────────────────────────────►│
    │                                      │   "api.github.com A?"        │                    │                    │
    │                                      │                              │                    │                    │
    │                                      │◄─(7) Answer────────────────────────────────────────────────────────│
    │                                      │   "api.github.com A          │                    │                    │
    │                                      │    140.82.113.4 (TTL=300)"    │                    │                    │
    │                                      │                              │                    │                    │
    │◄─(8) Cached Answer───────────────────│   "140.82.113.4"             │                    │                    │
    │                                      │                              │                    │                    │
    │──(9) TCP SYN──►                      │                              │                    │                    │
    │   to 140.82.113.4:443                 │                              │                    │                    │
```

**Total DNS queries**: 3 iterative (root, TLD, authoritative) + 1 recursive (client to resolver). The client only made 1 DNS call; everything else happened behind the scenes.

---

## 1.3 The 13 Root Servers — Deep Dive

### Why exactly 13 letters?

The number 13 comes from DNS protocol limitations, not physical infrastructure. The original DNS specification used a 1-octet label length field, limiting the number of root server names that could fit in a single UDP response packet to 13 (each name like `a.root-servers.net.` fits in the packet without fragmentation when considering the 512-byte UDP limit).

### BGP Anycast — How 13 addresses become 1700+ nodes

Each root server IP address (a through m) is announced via **BGP (Border Gateway Protocol)** from hundreds of physical server locations (Points of Presence) around the world. When your resolver sends a query to `a.root-servers.net` at `198.41.0.4`, BGP automatically routes your packet to the **geographically nearest physical server** hosting that anycast address.

**This means**:
- There are NOT just 13 root servers — there are 1700+ physical machines
- All 13 root server operators use **anycast** to distribute traffic globally
- An operator like Verisign (running both a and j) runs root server infrastructure in 150+ locations worldwide

### Root Server Operators

| Letter | Operator | Locations |
|--------|----------|-----------|
| a | Verisign | 150+ |
| b | USC-ISI (Information Sciences Institute) | 10+ |
| c | Cogent Communications | 50+ |
| d | University of Maryland | 15+ |
| e | NASA (Ames Research Center) | 15+ |
| f | Internet Systems Consortium (ISC) | 100+ |
| g | US Department of Defense | 10+ |
| h | US Army (Research Lab) | 15+ |
| i | Netnod (Sweden) | 50+ |
| j | Verisign | 150+ |
| k | RIPE NCC (Netherlands) | 100+ |
| l | ICANN (Los Angeles) | 30+ |
| m | WIDE Project (Japan) | 50+ |

**Key fact**: The root zone file is updated 2-3 times per day by IANA. All operators synchronize from the same source. If a new TLD is added (e.g., `.app`, `.tech`, `.ai`), it appears in the zone file and all 13 operators load the updated file within hours.

---

## 1.4 How the Root Server Parses Queries — Right-to-Left Logic

DNS was designed with a hierarchical namespace where each domain label is separated by dots. The hierarchy reads from **right to left**:

```
api.example.com.
     ▲       ▲       ▲
     │       │       │
   (root)  (TLD)  (domain)
    ^       ^       ^
    │       │       └── Level 3: Subdomain or hostname
    │       └────────── Level 2: Second-Level Domain (what you register)
    └────────────────── Level 1: Top-Level Domain
```

**The DNS protocol** encodes domain names as a sequence of labels, each prefixed by its length. So `api.example.com.` is encoded as:
```
03api 07example 03com 00
```
(3 bytes for "api", 7 bytes for "example", 3 bytes for "com", 00 = root dot terminator)

When a root server receives this:
1. It starts reading from the **rightmost label** (`com`)
2. It looks up `com.` in its root zone table
3. It finds the delegation to Verisign's `.com` TLD nameservers
4. It returns the NS records + glue A/AAAA records

**The root server completely ignores** labels to its left (`api`, `example`). It never looks at them. It is not designed to store or process individual domain records.

### Example: Query for `service.gov.in`

1. Root server reads rightmost label: `in`
2. Looks up `in.` in Root Zone File
3. Finds delegation to NIXI (National Internet Exchange of India)
4. Returns: `in. IN NS ns1.registry.in.` + glue `ns1.registry.in. IN A 37.209.192.9`

### Example: Query for `mycomputer.companyinternal`

1. Root server reads rightmost label: `companyinternal`
2. Searches root zone table for `companyinternal.`
3. **No match found**
4. Returns: **NXDOMAIN** (Non-Existent Domain) — query stops immediately

---

## 1.5 Core DNS Record Types — Detailed Mechanics

### A Record (IPv4 Address)

The most fundamental record. Maps a hostname to a 32-bit IPv4 address.

```dns
api.example.com.  300  IN  A  192.0.2.1
```
- **TTL**: 300 seconds (how long resolvers cache this)
- **Class**: IN (Internet)
- **Type**: A
- **Value**: 192.0.2.1

**Multiple A records = basic load balancing** (DNS round-robin):
```dns
api.example.com.  300  IN  A  192.0.2.1
api.example.com.  300  IN  A  192.0.2.2
api.example.com.  300  IN  A  192.0.2.3
```
When a client queries, the authoritative server rotates the order of records. Most client libraries pick the first returned IP. This gives a naive ~33/33/33 split.

### AAAA Record (IPv6 Address)

Maps a hostname to a 128-bit IPv6 address. Format uses hexadecimal colons.

```dns
api.example.com.  300  IN  AAAA  2606:2800:220:1:248:1893:25c8:1946
```

### CNAME Record (Canonical Name / Alias)

Creates an alias: "This name is really that other name."

```dns
app.example.com.  300  IN  CNAME  lb-123.aws.amazon.com.
```

**Critical rule**: When a resolver sees a CNAME, it **must** follow it and query the canonical name. The final answer is the A/AAAA record of the canonical name, NOT the CNAME itself.

**Chaining**:
```
app.example.com.  → CNAME  → lb-123.aws.amazon.com.  → A  → 52.0.0.1
```
The resolver performs TWO queries: one for the CNAME, then one for the canonical name's A record.

**The Zone Apex Problem**: A CNAME cannot coexist with SOA, NS, MX, or any other record at the same name. The zone apex (`example.com`) **must** have SOA and NS records. Placing a CNAME at `example.com` would destroy the SOA and NS records, breaking email delivery and delegation entirely.

**Cloud solutions**:
- **AWS Route 53 Alias**: Route 53 internally resolves the target and returns a synthetic A/AAAA record at the apex — it's not a real CNAME, it's a Route 53 proprietary feature
- **Cloudflare ANAME (CNAME Flattening)**: Cloudflare resolves the CNAME target and returns the resulting A/AAAA records, appearing as if there's no CNAME

### TXT Record (Text)

Holds arbitrary text strings. Originally for human-readable notes; now used for machine-readable security policies.

```dns
example.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"
example.com.  3600  IN  TXT  "google-site-verification=abc123def456"
example.com.  3600  IN  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

**SPF (Sender Policy Framework)**: Specifies which mail servers are authorized to send email on behalf of a domain. The `~all` means "soft fail — mark but don't reject" for unauthorized servers. `-all` would reject.

**DKIM (DomainKeys Identified Mail)**: Published as a TXT record at a selector subdomain (e.g., `default._domainkey.example.com`).

**DMARC (Domain-based Message Authentication)**: Published at `_dmarc.example.com`. Tells receiving mail servers what to do with emails that fail SPF/DKIM checks.

### PTR Record (Pointer / Reverse DNS)

Maps an IP address back to a domain name — the inverse of A records.

```dns
1.2.0.192.in-addr.arpa.  3600  IN  PTR  server.example.com.
```

The format reverses the IP octets and appends `in-addr.arpa.` (for IPv4) or `ip6.arpa.` (for IPv6).

**Why reverse DNS matters**:
- Email servers require reverse DNS (PTR) to accept incoming mail — many mail servers reject connections from IPs without PTR records to prevent spam
- Used in `traceroute`, network diagnostics, and security logging (showing hostnames instead of raw IPs)

### MX Record (Mail Exchange)

Directs email to mail servers with priority values.

```dns
example.com.  3600  IN  MX  10  mail.example.com.
example.com.  3600  IN  MX  20  mail2.example.com.
```

Lower priority number = higher preference. Email from other servers will try `mail.example.com` first (priority 10), then `mail2.example.com` (priority 20) if the first is unreachable.

**Important**: MX records point to **hostnames**, not IP addresses. The hostname must resolve to an A/AAAA record.

### SRV Record (Service Location)

More complex — specifies hostname, port, priority, and weight for a specific service.

```dns
_sip._tcp.example.com.  3600  IN  SRV  10  60  5060  bigbox.example.com.
```

Format: `_service._protocol.name. TTL SRV priority weight port target`

- **Priority**: Lower = preferred (like MX)
- **Weight**: Used for load distribution among servers with equal priority — higher weight gets more traffic proportionally
- **Port**: The specific port the service runs on
- **Target**: The hostname providing the service

**Common uses**: SIP (VoIP), LDAP, Kerberos, XMPP, etc.

### SOA Record (Start of Authority)

The most important record in any zone — it defines the zone's authoritative parameters.

```dns
example.com.  3600  IN  SOA  ns1.example.com. admin.example.com. (
    2024010101    ; Serial (YYYYMMDDNN — increment on every change)
    7200          ; Refresh (how often secondary nameservers check for updates)
    3600          ; Retry (how often to retry if refresh fails)
    1209600       ; Expire (how long to keep zone data if refresh fails completely)
    3600          ; Negative Cache TTL (how long to cache NXDOMAIN responses)
)
```

**The Serial number** is the heartbeat of DNS migrations. Every time you change a zone file, you **must** increment the serial (or use a date-based scheme). Secondary/ slave nameservers compare serials to decide if they need to pull a zone transfer (AXFR/IXFR).

---

## 1.6 Transport Protocols: UDP vs TCP Port 53

### UDP Port 53 — The Default

- **Stateless**: Each query is independent — no connection is established
- **1-RTT**: Single request → single response (fastest possible)
- **No congestion control**: No backoff, no retransmission logic at the transport layer
- **512-byte limit**: The original DNS specification limited UDP responses to 512 bytes

**Why UDP?** DNS queries are tiny (typically 50-100 bytes) and the response pattern is request-response. Establishing a TCP connection for a single query would add unnecessary overhead. For high-volume resolvers handling millions of queries per second, UDP's statelessness is critical.

### EDNS0 (Extension Mechanisms for DNS)

EDNS0 extended the 512-byte UDP limit. Instead of the fixed 512-byte maximum, EDNS0 allows the client to advertise a larger UDP payload size (typically 1232 bytes or 4096 bytes) in the OPT pseudo-section of the query.

```dns
; EDNS0 OPT Record (in Additional section)
; UDP payload size: 4096
; Version: 0
; Do (DNSSEC OK): 1
```

Without EDNS0, DNSSEC-signed responses (which include RRSIG records) would always exceed 512 bytes and trigger TCP fallback. With EDNS0, most DNSSEC responses fit in UDP.

### TCP Port 53 — When UDP Isn't Enough

**Triggers for TCP fallback**:
1. **Response truncation (TC=1)**: When the response exceeds the negotiated UDP payload size
2. **Zone transfers (AXFR/IXFR)**: Full zone file transfers between primary and secondary nameservers — these can be megabytes of data
3. **DNS-over-TLS (DoT)**: Encrypted DNS on TCP port 853 (or 443 for DoH)
4. **Large TXT records**: SPF records with many include statements can exceed UDP limits

**TCP 3-way handshake for DNS**:
```
Client                          Server
  │                               │
  │──SYN─────────────────────────►│
  │◄──SYN-ACK────────────────────│
  │──ACK─────────────────────────►│
  │                               │
  │──DNS Query (TCP, no size limit)────────────────►│
  │◄────────DNS Response─────────│
  │                               │
```

**Practical implication**: If TCP 53 is blocked by a firewall, DNSSEC, large responses, and zone transfers will fail — but normal A/AAAA queries usually still work over UDP.

### Practical Testing

```bash
# Check if a response was truncated (forced TCP)
dig @8.8.8.8 +bufsize=512 org. SOA | grep flags
# Look for "; (2 bytes) -> buf 512B" and flags: trunc

# Force a query to use UDP with a specific buffer size
dig @8.8.8.8 +bufsize=512 google.com A    # May truncate
dig @8.8.8.8 +bufsize=4096 google.com A   # Likely succeeds over UDP with EDNS0

# Capture UDP vs TCP on port 53
sudo tcpdump -i eth0 'port 53 and udp' -nn   # Watch UDP DNS
sudo tcpdump -i eth0 'port 53 and tcp' -nn   # Watch TCP DNS
```

---

## 1.7 Glue Records — Breaking the Circular Dependency

### The Problem

When a nameserver for a domain (e.g., `ns1.provider.com`) is itself a subdomain of that same domain (e.g., `ns1.example.com` managing `example.com`), there is a circular dependency:

```
example.com uses ns1.example.com as authoritative nameserver
To find ns1.example.com, you need to resolve example.com
To resolve example.com, you need ns1.example.com
```

### The Solution: Glue Records

The parent TLD (.com) provides the IP addresses of `ns1.example.com` directly in the Additional section of the referral response. These are called **glue records** — they are "glued" into the response by the parent TLD.

```
TLD Server (.com) Response:
  Authority Section:
    example.com.  IN  NS  ns1.example.com.

  Additional Section (Glue):
    ns1.example.com.  IN  A  203.0.113.5
```

**The resolver can now contact `ns1.example.com` at `203.0.113.5` without having to resolve `example.com` first.** The glue breaks the circular dependency.

**Without glue**: The resolver would get the NS record but have no IP for the nameserver. It would have to perform an additional recursive resolution just to find the nameserver's IP, creating an infinite recursion.

### In-Bailiwick vs Out-of-Bailiwick
- **In-bailiwick glue**: The nameserver hostname is a subdomain of the zone being delegated (e.g., `ns1.example.com` for `example.com`) — glue is REQUIRED
- **Out-of-bailiwick glue**: The nameserver hostname is in a different domain (e.g., `ns1.awsdns.com` for `example.com`) — no glue needed, the resolver can resolve the nameserver independently

---

# Part 2: DNS Migration — Zero-Downtime Strategy

---

## 2.1 The TTL Problem — Why DNS Changes Don't Propagate Instantly

### How Resolvers Actually Honor TTL

When an authoritative nameserver returns a DNS record, it includes a **TTL (Time To Live)** value in seconds. Every recursive resolver (Google 8.8.8.8, Cloudflare 1.1.1.1, your ISP's resolver) that fetches this record stores it in memory and starts a local countdown timer — completely independent of the authoritative server's clock.

**Timeline of a cached record**:

```
T = 0s:   Authoritative server returns A record for api.example.com → 192.0.2.1 with TTL=300
T = 0s:   Resolver answers client from cache → TTL shows 300s
T = 120s: Resolver answers client from cache → TTL shows 180s (countdown in progress)
T = 299s: Resolver answers client from cache → TTL shows 1s
T = 301s: Cache EXPIRED. Resolver MUST query upstream authoritative server again
T = 301s: Resolver gets fresh IP (could be the same or new if record was updated)
```

**The fundamental problem**: There is **no universal push mechanism** in DNS to tell resolvers worldwide to clear a cached record before its TTL expires. Unlike HTTP where you can send `Cache-Control: no-cache` or a CDN purge API, DNS is designed as a pull system with distributed caches.

### Real-World Migration Example

**Scenario**: Migrate `api.example.com` from Server A (`1.1.1.1`) to Server B (`2.2.2.2`) without dropping any active connections.

**If TTL = 86400 (24 hours)** and you change the A record at noon:
- Resolvers that cached the record at 11:59 AM still serve `1.1.1.1` until tomorrow at 11:59 AM
- Resolvers that cached it at 12:01 PM will serve `2.2.2.2` until tomorrow at 12:01 PM
- Some resolvers might still serve `1.1.1.1` up to 48 hours later if their TTL was set before your change

**The math is unforgiving**: You must wait at least the **original TTL duration** before changing the IP.

### ISP Non-Compliance (The Hidden Trap)

Many mobile carriers and regional ISPs override TTL values to reduce their DNS infrastructure load. Common clamping behaviors:

| ISP Behavior | Effect |
|-------------|--------|
| Clamp TTL minimum to 300s | Any TTL < 300s is treated as 300s |
| Clamp TTL to 3600s | Even TTL=60s gets treated as 1 hour |
| Some ignore TTL entirely | Cache for fixed duration regardless of TTL |

**Practical implication**: When planning migrations, add a **buffer** to your pre-migration wait time. If you set TTL=60s but your mobile carrier clamps to 3600s, you need to wait **1 hour minimum** (not 60 seconds) before executing the cutover.

**How to test for ISP TTL overriding**:
```bash
# Set a very low TTL (e.g., 10s) and monitor how long resolvers actually cache it
# If they cache for much longer than 10s, you know they're overriding TTLs
```

---

## 2.2 Zero-Downtime Migration — Complete 4-Step Walkthrough

### Scenario

Migrate `api.example.com` from:
- **Server A** (old): `192.168.1.2`
- **Server B** (new): `192.168.1.3`
- Current TTL: 86400s (24 hours)

### Step 1: Drop the TTL (T - 24 Hours)

**Action**: Edit the zone file to change TTL from 86400s to 60s. Increment the SOA serial.

**Zone file before**:
```dns
$TTL 86400
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum
)
@   IN  NS   ns1.example.com.
@   IN  A    192.168.1.2
api IN  A    192.168.1.2
```

**Zone file after**:
```dns
$TTL 60
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024010102  ; Serial incremented!
        3600
        1800
        604800
        60
)
@   IN  NS   ns1.example.com.
@   IN  A    192.168.1.2
api IN  A    192.168.1.2
```

**Reload**: `sudo systemctl reload named` or `sudo rndc reload example.com`

**CRITICAL RULE**: You must now **wait at least 86400 seconds (24 hours)** before proceeding. Why?

- Any resolver that cached the record **before** you changed the TTL still has the old 86400s countdown running
- Those resolvers will not query again until their cache expires (which could be up to 24 hours from when they last fetched)
- Only resolvers that fetched the record **after** you dropped the TTL will be on the 60-second countdown
- Waiting 24 hours ensures the **last batch** of resolvers on the old TTL have also expired

**What happens during this wait**:
```
Resolvers that cached at T-1hr:  Still on 86400s countdown → will re-query at T+23hrs
Resolvers that cached at T-1min: Still on 86400s countdown → will re-query at T+23h59m
Resolvers that cached at T+1min: On 60s countdown → will re-query at T+61min
...
```

### Step 2: Execute the Cutover (T = 0)

**Action**: Update the A record to point to Server B's IP. Increment the SOA serial again.

**Zone file after cutover**:
```dns
$TTL 60
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024010103  ; Serial incremented again!
        3600
        1800
        604800
        60
)
@   IN  NS   ns1.example.com.
@   IN  A    192.168.1.3       ← CHANGED TO SERVER B
api IN  A    192.168.1.3       ← CHANGED TO SERVER B
```

**Reload** and verify:
```bash
sudo named-checkzone example.com /etc/bind/zones/db.example.com
sudo systemctl reload named

# Verify authoritative server returns new IP
dig @ns1.example.com api.example.com A +short
# Output: 192.168.1.3

# Check TTL in recursive resolver response
dig @your-resolver api.example.com A
# Answer section should show: api.example.com.  60  IN  A  192.168.1.3
```

**What happens globally over the next 60 seconds**:
- Resolvers with TTL=60s that have expired → fetch the new IP immediately
- Resolvers with TTL still counting down → continue serving old IP until their timer hits 0
- Traffic gradually shifts from A to B over ~60-120 seconds

### Step 3: Traffic Drain Window (T + 60s to T + several minutes)

**Do NOT turn off Server A immediately.** Keep it running.

**Why**:
- **In-flight TCP connections**: Users who connected to Server A before the cutover may still be in the middle of an HTTP request. TCP sessions don't die instantly when DNS changes.
- **Keep-alive connections**: Clients with persistent HTTP connections to Server A will continue using those connections.
- **ISP TTL clamping**: As discussed above, some resolvers may serve the old IP for longer than 60s.
- **Application-level caching**: JVMs, browsers, and CDNs may cache DNS results independently of DNS TTLs.

**Verification commands**:
```bash
# On Server A — check if any traffic is still arriving
sudo tail -f /var/log/nginx/access.log
# OR
sudo tcpdump -i eth0 host 192.168.1.2 and port 80

# On Server B — confirm it's receiving traffic
sudo tail -f /var/log/nginx/access.log

# From client — confirm which server responds
curl http://api.example.com
# Should return response from Server B
```

**Safe to decommission Server A when**: Access logs on Server A show zero requests for several minutes AND you have confirmed all in-flight sessions have completed.

### Step 4: Restore Standard TTL (T + several minutes)

**Action**: Change TTL back to standard value (3600s or 86400s). Increment SOA serial.

```dns
$TTL 3600                    ← Back to standard
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024010104  ; Serial
        ...
        3600         ← Standard TTL restored
)
@   IN  A    192.168.1.3
api IN  A    192.168.1.3
```

**Why restore**:
- Protects infrastructure from unnecessary DNS query load (a TTL of 60s means resolvers query 86400/60 = 1440x more often than with 86400s TTL)
- Reduces client latency (higher cache-hit ratio)
- Reduces nameserver bandwidth costs

### Complete Timeline Visualization

```
TIME ─────────────────────────────────────────────────────────────►

T - 24h        T - 23h        T - 1hr        T = 0          T + 60s     T + 10min    T + 24h
  │              │              │              │              │           │            │
  ▼              ▼              ▼              ▼              ▼           ▼            ▼
[TTL=86400]   [TTL=86400]   [TTL=86400]  [TTL=60]     [TTL=60]    [TTL=60]    [TTL=3600]
  │              │              │              │              │           │            │
  │         (all resolvers     │         Resolvers    Resolvers   Resolvers    Resolvers
  │          still on 86400s   │          refreshing  still       on 60s       on 3600s
  │          countdown)        │          on 60s     serving     serving     serving
  │              │              │         new IP     old/new     new IP      new IP
  │              │              │              │              │           │            │
  │         "Keep waiting..."  │         "Cutover     "Traffic    "TTL     "Migration
  │                              "complete"    "draining"  "restored" complete!"
```

---

## 2.3 Global Traffic Management (GTM) Routing Policies

Modern cloud DNS providers (AWS Route 53, Cloudflare, NS1, Azure DNS) evaluate routing rules **dynamically at query time** based on real-time telemetry, geography, and server health.

### 1. Latency-Based Routing (LBR)

**How it works**: The DNS provider continuously measures network Round Trip Time (RTT) between its global network and major recursive resolver networks (Google DNS, Cloudflare, ISP resolvers).

**Example**: User in Frankfurt queries `api.example.com`. Route 53's latency table shows the lowest RTT from Frankfurt resolver networks to `eu-central-1` (Frankfurt AWS region). It returns `10.0.0.1` (Frankfurt server) instead of `10.1.1.1` (us-east-1 Virginia server).

**The ECS (EDNS0 Client Subnet) catch**:
- Standard DNS sees the IP of the **recursive resolver** (e.g., Google DNS `8.8.8.8` in the US), not the actual end user
- Without ECS, a user in Frankfurt behind Google DNS would get the US server's IP
- **ECS extension**: The resolver forwards the client's `/24` subnet prefix in the EDNS0 option field:
```dns
; Client sends via resolver:
api.example.com.  IN  A  (with EDNS0 ECS: 192.168.1.0/24)
```
This allows the DNS provider to route based on the **user's actual subnet**, not the resolver's location.

### 2. Weighted Routing (Canary Deployments)

**How it works**: Assign numeric weights to multiple IPs for the same record name. Traffic splits proportionally.

**Formula**: `Traffic % to Server B = Weight_B / (Weight_A + Weight_B) × 100`

**Example — safe rollout**:
```
Phase 1:  Weight_A=90, Weight_B=10  →  10% to new v2, 90% to old v1
Phase 2:  Weight_A=50, Weight_B=50  →  50/50 split after monitoring v2 for 1 hour
Phase 3:  Weight_A=0,  Weight_B=100 →  100% to v2 (old decommissioned)
```

**Route 53 implementation**:
```bash
aws route53 change-resource-record-sets --hosted-zone-id Z1234567 \
--change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "api.example.com",
      "Type": "A",
      "SetIdentifier": "v2-canary",
      "Weight": 10,
      "ResourceRecords": [{"Value": "10.0.0.2"}]
    }
  }, {
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "api.example.com",
      "Type": "A",
      "SetIdentifier": "v1-production",
      "Weight": 90,
      "ResourceRecords": [{"Value": "10.0.0.1"}]
    }
  }]
}'
```

### 3. Health Checks & Automated DNS Failover

**How it works**: Route 53 (or equivalent) runs active health checks every 10-30 seconds against your endpoints using HTTP, HTTPS, or TCP probes.

**Active-Passive Setup**:
```
Primary Record:   api.example.com → 1.1.1.1  (Health Check A: /health on 443)
Secondary Record: api.example.com → 2.2.2.2  (DR site, no traffic normally)
```

**Failover sequence**:
```
T=0:     All traffic → 1.1.1.1 (primary healthy)
T=30s:   Health check fails on 1.1.1.1 (1st consecutive failure)
T=60s:   Health check fails on 1.1.1.1 (2nd consecutive failure)
T=90s:   Health check fails on 1.1.1.1 (3rd consecutive failure)
T=90s:   Route 53 marks 1.1.1.1 UNHEALTHY
T=90s:   All subsequent DNS responses return 2.2.2.2
T=90s+:  Resolvers update caches → traffic shifts to DR site
```

**Key detail**: The failover time = max(3 × health check interval, TTL). If health checks run every 30s and TTL is 60s, total failover time is ~90-120 seconds.

---

## 2.4 Scenario Walkthrough: System Outage During Migration

### The Interview Scenario

> "You are executing a cloud migration from on-prem to AWS using DNS. You updated your record from on-prem to AWS, but 10 minutes later, the AWS database crashes under load. You immediately change the DNS record back to on-prem, but users report they are still hitting the broken AWS site. Why did this happen, and how would you structure the architecture to prevent it?"

### Root Cause Analysis

**The TTL Trap**: The team did **not** lower the TTL 24 hours prior to the migration. Resolvers cached the broken AWS IP (`10.0.0.2`) for the default duration (e.g., 86400s / 24 hours). When the team changed the record back to on-prem (`192.168.1.2`), resolvers that had cached the AWS IP **continued serving it** until their TTL expired — ignoring the rollback update entirely.

**Hardcoded caching at the application layer**: Some clients (Java with JVM DNS caching, Go with cgo resolver, browsers with DNS caching) may cache the IP indefinitely, ignoring DNS TTLs entirely.

**The timeline of failure**:
```
T=0:   Change DNS from on-prem (192.168.1.2) → AWS (10.0.0.2)
T=5m:  AWS database crashes under load (5xx errors)
T=5m:  Engineer changes DNS back to on-prem (192.168.1.2)
T=5m:  Engineers see it working locally (their resolver had TTL=60s or less)
T=5m:  But 90% of users' resolvers still have 10.0.0.2 cached (TTL was 86400s)
T=5m:  Users still hit broken AWS site
T+24h: Last cached resolvers expire → users finally reach on-prem
```

### Prevention Architecture

**Pre-Migration**: Always drop TTL to 60s at T-24h (the standard zero-downtime procedure).

**Active-Active During Migration**: Instead of a hard 0→100 cutover, use **Weighted Routing with Health Checks** at all times:

```
Phase 1 (T-24h): Lower TTL to 60s
Phase 2 (T=0): Set Weighted Routing:
  - 90% weight → AWS (10.0.0.2)
  - 10% weight → On-prem (192.168.1.2) — keep both running
Phase 3 (Monitor): Watch CloudWatch metrics, error rates, latency on AWS
Phase 4 (Rollback if needed):
  - If AWS fails: change weight to 0% AWS, 100% on-prem
  - Route 53 Health Checks auto-detect 5xx errors and can auto-failover
Phase 5 (Stable): Gradually increase AWS weight to 100%
```

**The key architectural insight**: Never rely solely on a manual DNS rollback. Combine **low TTL** + **weighted routing** + **automated health checks** so that rollback is either automatic or near-instant.

---

# Part 3: Kubernetes Internal DNS

---

## 3.1 Pod DNS Configuration — `/etc/resolv.conf` Deep Dive

Every pod in Kubernetes gets its `/etc/resolv.conf` automatically injected by the **kubelet** (the agent running on each node). The file is not configurable by default — it's generated from the cluster's DNS policy.

### Default `/etc/resolv.conf` Inside a Pod

```bash
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Each line has a specific purpose:

### `nameserver 10.96.0.10`

This is the **ClusterIP** of the `kube-dns` Service (in the `kube-system` namespace). It's a virtual IP managed by Kubernetes — it doesn't point to a single machine. Behind this IP, there are multiple CoreDNS pod replicas. Traffic to this ClusterIP gets load-balanced (via iptables/IPVS rules set up by kube-proxy) to one of the CoreDNS pod IPs.

**How the ClusterIP works**:
1. When a pod sends DNS to `10.96.0.10:53`, the packet hits the node's `kube-proxy` daemon
2. `kube-proxy` has iptables rules that match destination `10.96.0.10:53`
3. These rules perform **DNAT** (Destination NAT) — rewriting the destination IP from `10.96.0.10` to the actual IP of a CoreDNS pod (e.g., `10.244.1.15`)
4. The packet is forwarded to the CoreDNS pod which responds directly

### `search` Domains

The `search` directive tells the resolver to **append these domain suffixes** to any hostname that is not fully qualified. This enables shorthand service discovery within Kubernetes.

```bash
search default.svc.cluster.local svc.cluster.local cluster.local
```

**What this means in practice**: If an application tries to connect to `payments`, the resolver tries:
1. `payments.default.svc.cluster.local`
2. `payments.svc.cluster.local`
3. `payments.cluster.local`
4. `payments.` (the bare domain — if none of the above worked, this fails)

This allows cross-namespace service discovery: a pod in namespace `default` can reach `payments` in namespace `production` by using `payments.production` (which becomes `payments.production.default.svc.cluster.local`).

### `options ndots:5`

The `ndots` option tells the resolver how many dots a hostname must contain before it's treated as **fully qualified** (absolute) vs. **relative** (needs search suffixes appended).

**The rule**: If a hostname contains **fewer than 5 dots**, the resolver first tries appending each search domain before trying the raw domain.

---

## 3.2 The `ndots:5` Query Amplification Problem

### The Problem in Detail

Consider an application making an API call to `api.stripe.com`. Let's trace exactly what the resolver does:

```
Step 1: Check for exact name "api.stripe.com" — not fully qualified (only 2 dots)
        2 < 5 (ndots threshold), so treat as relative
        
Step 2: Try appending search domains:
        a) api.stripe.com.default.svc.cluster.local.  → NXDOMAIN (CoreDNS)
        b) api.stripe.com.svc.cluster.local.          → NXDOMAIN (CoreDNS)
        c) api.stripe.com.cluster.local.              → NXDOMAIN (CoreDNS)
        
Step 3: Finally try the raw domain:
        d) api.stripe.com.                             → SUCCESS (returns public IP)
```

That's **4 DNS queries** for a single API call.

### The Dual-Stack Amplification

Modern applications and glibc resolver typically query **both A (IPv4) and AAAA (IPv6)** records simultaneously over the **same socket** (same source port):

```
Query 1: api.stripe.com.default.svc.cluster.local.  → A  → NXDOMAIN
Query 2: api.stripe.com.default.svc.cluster.local.  → AAAA → NXDOMAIN
Query 3: api.stripe.com.svc.cluster.local.          → A  → NXDOMAIN
Query 4: api.stripe.com.svc.cluster.local.          → AAAA → NXDOMAIN
Query 5: api.stripe.com.cluster.local.              → A  → NXDOMAIN
Query 6: api.stripe.com.cluster.local.              → AAAA → NXDOMAIN
Query 7: api.stripe.com.                            → A  → SUCCESS
Query 8: api.stripe.com.                            → AAAA → SUCCESS
```

**4 search domains × 2 record types = 8 DNS queries** for a single external API call to `api.stripe.com`.

**Impact at scale**: If you have 100 pods each making 10 API calls per second to external services, that's 100 × 10 × 8 = **8,000 unnecessary DNS queries per second** hitting your CoreDNS cluster — plus 7 out of every 8 queries return NXDOMAIN, wasting resources.

### Fix 1: Append a Trailing Dot in Application Code

```python
# Instead of:
url = "https://api.stripe.com"

# Use:
url = "https://api.stripe.com."  # Trailing dot = absolute FQDN
```

The trailing dot tells the resolver: "This is a Fully Qualified Domain Name, do NOT append search suffixes." This bypasses the search domain lookup entirely, reducing 8 queries to 2 (just A and AAAA for `api.stripe.com.`).

### Fix 2: Pod-Level `dnsConfig` Override

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
  containers:
  - name: app
    image: myapp:latest
```

With `ndots:2`, a hostname like `api.stripe.com` (2 dots) is treated as fully qualified immediately. No search suffixes are appended. Only 2 queries (A + AAAA) instead of 8.

**Tradeoff of lowering ndots cluster-wide**: Some cross-namespace service lookups rely on the search domains. For example, a pod in namespace `sales` calling `payments.billing` needs `ndots >= 2` to work. Setting `ndots:2` cluster-wide breaks shorthand lookups like `service.other-namespace` unless developers always use the full FQDN.

---

## 3.3 The Linux Kernel conntrack UDP Race Condition

This is one of the most subtle and impactful bugs in Kubernetes networking. It causes **exactly 5-second latency spikes** that are notoriously difficult to diagnose.

### The Problem: What Happens in the Linux Kernel

When a pod sends DNS queries, here is the exact sequence of events in the kernel:

```
[ Application Thread (glibc resolver) ]
        │
        ├──► UDP Socket Send: Query A (IPv4)
        │    Source: (10.244.1.5, ephemeral_port_12345)
        │    Dest:   (10.96.0.10, 53)
        │    ┌─────────────────────────────────────────┐
        │    │ DNS Header: ID=0xABCD, QNAME=api.stripe.com, QTYPE=A │
        │    └─────────────────────────────────────────┘
        │
        └──► UDP Socket Send: Query AAAA (IPv6)
             Source: (10.244.1.5, ephemeral_port_12345)  ← SAME PORT!
             Dest:   (10.96.0.10, 53)
             ┌─────────────────────────────────────────┐
             │ DNS Header: ID=0xABCD, QNAME=api.stripe.com, QTYPE=AAAA │
             └─────────────────────────────────────────┘
```

**Key observation**: Both queries use the **same source IP, same source port, same destination**. They arrive at the kernel within microseconds of each other over the same socket.

Now, the packet enters the Linux kernel's networking stack. In standard Kubernetes clusters using **iptables mode**, kube-proxy has configured DNAT rules that intercept all traffic destined for `10.96.0.10:53` and rewrite it to a CoreDNS pod IP.

```
[ Linux Kernel netfilter / conntrack ]

For each packet, netfilter must create a conntrack entry:
  Tuple = (SrcIP, SrcPort, DstIP, DstPort, Protocol)
  = (10.244.1.5, 12345, 10.96.0.10, 53, UDP)

For Packet 1 (Query A):
  └► conntrack_allocate() → creates entry: (10.244.1.5, 12345, 10.96.0.10, 53, UDP)
     Also performs DNAT: 10.96.0.10 → 10.244.1.15 (CoreDNS pod IP)
     Marks as "confirmed" → entry now in conntrack table

For Packet 2 (Query AAAA):
  └► conntrack_allocate() → SAME TUPLE as Packet 1!
     └► ⚠️ RACE CONDITION:
         conntrack hash table lock is already held by Packet 1's thread
         Packet 2's __nf_conntrack_confirm() tries to insert duplicate unconfirmed tuple
         → conntrack_spin_lock() blocks briefly
         → Packet 2 is rejected as conflicting tuple
         → NF_DROP (silently dropped, NO ICMP error, NO RST)
```

**The result**: Query AAAA is silently dropped by the kernel. The application (glibc resolver) sends Query A and waits. After 5 seconds (glibc's hardcoded retransmit timer), it retransmits Query AAAA. By that time, the conntrack slot from Query A has expired, so Query AAAA goes through successfully.

**Total latency for Query AAAA = 5000ms** (exactly the glibc 5-second retransmit timeout).

### Why 5 Seconds Specifically

The glibc resolver (`getaddrinfo()` / `gethostbyname()`) has a hardcoded retransmission timeout:
- First query sent
- Wait **5 seconds** for response
- If no response, retransmit once
- Wait **5 seconds** more
- Give up

The 5-second timer is baked into glibc's `_res.retrans` default value (defined in `/usr/include/resolv.h`). You cannot easily change this without recompiling glibc.

### The Fix: NodeLocal DNSCache

```yaml
# NodeLocal DNSCache is deployed as a DaemonSet - one pod per node
# Each pod listens on link-local IP: 169.254.20.10
```

**Architecture**:
```
[ Pod ] → queries 169.254.20.10 (local dummy interface on same node)
         ↓
[ NodeLocal DNSCache DaemonSet Pod ] (on same node)
         ↓ Cache hit? → Return cached result immediately (microseconds)
         ↓ Cache miss? → Forward to upstream CoreDNS over TCP port 53
                          (TCP is connection-oriented, no conntrack race!)
         ↓
[ CoreDNS Pod(s) ] → Resolve and return answer
```

**Why this works**:
1. **Local queries bypass NAT entirely**: Traffic to `169.254.20.10` goes through the node's loopback/dummy interface. No iptables DNAT rules are triggered. No conntrack entries are created.
2. **TCP upstream**: When NodeLocal DNSCache needs to query upstream CoreDNS, it uses a **persistent TCP connection**. TCP connections don't have the duplicate-tuple problem because the kernel creates a unique connection for each TCP session (different source port).
3. **Caching**: NodeLocal DNSCache caches responses on the node. 80%+ of queries never leave the node, reducing CoreDNS load and latency to microseconds.

---

## 3.4 CoreDNS Pod Identification & Routing Architecture

### Case 1: Standard Cluster (Without NodeLocal DNSCache)

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES SERVICE ABSTRACTIONS            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Service: kube-dns                                          │
│  Namespace: kube-system                                     │
│  ClusterIP: 10.96.0.10                                      │
│  Selector: k8s-app: kube-dns                                │
│  Port: 53/UDP                                               │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ CoreDNS Pod 1 │    │ CoreDNS Pod 2 │    │ CoreDNS Pod 3 │  │
│  │ Labels:       │    │ Labels:       │    │ Labels:       │  │
│  │ k8s-app:      │    │ k8s-app:      │    │ k8s-app:      │  │
│  │ kube-dns      │    │ kube-dns      │    │ kube-dns      │  │
│  │ IP: 10.244.1.15│   │ IP: 10.244.2.20│   │ IP: 10.244.3.25│  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│           │                   │                   │        │
│           └───────────────────┴───────────────────┘        │
│                     Endpoints / EndpointSlices              │
│                     (tracks all pod IPs automatically)      │
└─────────────────────────────────────────────────────────────┘
```

**How pods find CoreDNS**:
1. `kubelet` injects `nameserver 10.96.0.10` into every pod's `/etc/resolv.conf`
2. Pod sends UDP query to `10.96.0.10:53`
3. `kube-proxy`'s iptables rules intercept the packet
4. iptables DNAT rule matches `10.96.0.10:53` → rewrites destination to one of the CoreDNS pod IPs (10.244.1.15, 10.244.2.20, or 10.244.3.25)
5. DNAT selection uses `probability` or `random` matching → essentially random load balancing
6. CoreDNS pod responds directly back to the pod

**The problem**: Every single DNS query from every pod hits iptables DNAT rules. With N services in the cluster, iptables has O(N) rules. Each packet traverses all of them sequentially. For clusters with thousands of services, this adds significant per-packet latency.

### Case 2: Advanced Cluster (With NodeLocal DNSCache)

```
┌─────────────────────────────────────────────────────────────┐
│                 NODELOCAL DNS CACHE DEPLOYMENT                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  DaemonSet: node-local-dns                                  │
│  Runs on EVERY worker node                                  │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ Node 1    │  │ Node 2    │  │ Node 3    │                │
│  │ 169.254.20.10│ 169.254.20.10│ 169.254.20.10│             │
│  │ (Daemon) │  │ (Daemon) │  │ (Daemon) │                 │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                 │
│       │              │              │                       │
│       ▼              ▼              ▼                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │Reads CoreDNS│ │Reads CoreDNS│ │Reads CoreDNS│            │
│  │Endpoints   │ │Endpoints   │ │Endpoints   │              │
│  │from API    │ │from API    │ │from API    │              │
│  │directly    │ │directly    │ │directly    │              │
│  └──────────┘  └──────────┘  └──────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

**How traffic flows differently with NodeLocal DNSCache**:

1. `kubelet` injects `nameserver 169.254.20.10` (instead of `10.96.0.10`) into pod `/etc/resolv.conf`
2. Pod sends UDP query to `169.254.20.10:53` — this address lives on a **dummy network interface** on the same physical node
3. The packet stays on the node — **no network crossing, no overlay, no NAT**
4. NodeLocal DNSCache DaemonSet pod on that node receives the query
5. If cached → returns immediately (sub-millisecond)
6. If not cached → NodeLocal DNSCache opens a **direct TCP connection** to a CoreDNS pod IP (read directly from Endpoints API)
7. TCP connection is established peer-to-peer — **no kube-proxy, no iptables, no conntrack**

---

# Part 4: Cilium eBPF — Replacing kube-proxy

---

## 4.1 The Problem with kube-proxy and iptables

Traditional Kubernetes networking uses **kube-proxy** which generates massive chains of **iptables** (or **IPVS**) rules to handle Service routing via Destination Network Address Translation (DNAT).

### O(N) Sequential Rule Processing

```
[ Packet destined for ClusterIP 10.96.0.10:53 ]
        │
        ▼
[ iptables chains ]
        │
        ▼
┌──────────────────────────────────────────┐
│ KUBE-SERVICES chain                      │
│  └► KUBE-DNS cluster IP match            │
│       └► KUBE-DNS-YYYYYY rules (random)  │ ← Must traverse
│            └► DNAT to 10.244.1.15:53      ← one of N rules
└──────────────────────────────────────────┘
```

For a cluster with **N services**, kube-proxy generates iptables rules for every service endpoint. A cluster with 1000 services × 10 replicas = 10,000 DNAT rules. Every packet must traverse this chain **sequentially** until it finds the matching rule. This is **O(N) complexity**.

### nf_conntrack Table Locks — The 5-Second DNS Killer

For every NAT translation, the kernel's `nf_conntrack` subsystem must create an entry in a hash table tracking the connection tuple: `(SrcIP, SrcPort, DstIP, DstPort)`.

```
Packet 1: (10.244.1.5:12345 → 10.96.0.10:53)
  conntrack_allocate(hash) → slot X → insert tuple → lock slot X

Packet 2: (10.244.1.5:12345 → 10.96.0.10:53)  ← SAME TUPLE!
  conntrack_allocate(hash) → slot X → lock slot X → BLOCKED
  └► ⚠️ RACE: Packet 2's thread spins waiting for the lock held by Packet 1
  └► Packet 2 is DROPPED (NF_DROP) when the tuple fails to confirm
```

Because glibc sends A and AAAA queries simultaneously from the **same socket** (same source port), both packets arrive at `__nf_conntrack_confirm()` at the exact same microsecond. One wins the spinlock, the other is rejected. The application waits 5 seconds for the timeout.

---

## 4.2 Cilium eBPF Replacement Model

Cilium replaces the entire iptables/netfilter pipeline with **eBPF (Extended Berkeley Packet Filter)** programs attached directly to kernel hooks. eBPF programs run in a sandboxed VM inside the kernel — they're JIT-compiled to native machine code, verified for safety by the kernel, and execute at line rate.

### Mechanism A: Socket-Level Translation (sockops)

Instead of waiting for a network packet to be constructed and traversing down through the entire network stack, Cilium intercepts the **syscall** itself:

```
[ Application calls connect(10.96.0.10:53) ]
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  eBPF Program attached to cgroup2 / sock_ops hook            │
│                                                             │
│  1. Intercepts connect() syscall                             │
│  2. Extracts destination: 10.96.0.10:53                      │
│  3. BPF Map O(1) lookup: 10.96.0.10:53 → 10.244.1.15:53   │
│     (Service IP → Backend Pod IP, via pre-loaded BPF Map)    │
│  4. Rewrites sockaddr_in in the kernel's socket buffer       │
│     dest_ip = 10.244.1.15, dest_port = 53                    │
│  5. Returns to application: connect() "succeeds"             │
│                                                             │
│  APPLICATION BELIEVES it connected to 10.96.0.10            │
│  ACTUAL packet is born destined for 10.244.1.15:53           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
[ TCP/IP Stack ]
  • Packet is born with REAL target Pod IP (10.244.1.15:53)
  • ❌ ZERO iptables rules touched
  • ❌ ZERO netfilter conntrack NAT locks
        │
        ▼
[ Network ]
  Direct pod-to-pod communication (no NAT, no proxy)
```

**Key advantage**: The destination IP is rewritten **before the packet is even created**. The kernel's TCP/IP stack generates a packet destined directly for the backend Pod IP. No NAT translation is needed at the network layer. No conntrack entry is created. No 5-second race condition.

### Mechanism B: Fast-Path Packet Forwarding (TC & XDP)

For packets arriving from other nodes or the outside world:

```
[ Incoming packet on physical NIC ]
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  eBPF Program attached to TC (Traffic Control) hook          │
│                                                             │
│  1. eBPF checks connection tracking map: cilium_ct_4        │
│     (native BPF hash map, O(1) lookup)                      │
│  2. If existing connection: use cached backend              │
│  3. If new connection: select backend via BPF Map           │
│  4. bpf_redirect() → direct memory redirect to target veth  │
│                                                             │
│  ❌ No netfilter traversal                                   │
│  ❌ No conntrack                                             │
│  ❌ No iptables rules                                        │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
[ Target Pod veth pair ]
```

---

## 4.3 kube-proxy (iptables) vs. Cilium (eBPF) Comparison

| Feature | kube-proxy (iptables) | Cilium (eBPF) |
|---------|----------------------|---------------|
| **Lookup algorithm** | O(N) sequential rule traversal | O(1) hash table lookup in BPF Maps |
| **Translation point** | Layer 3/4 network stack (netfilter) | Layer 4 Socket syscall (`connect`, `sendmsg`) |
| **Conntrack dependency** | Required — locks on every NAT | Bypassed — uses native BPF Maps |
| **5-second UDP DNS drops** | Vulnerable (conntrack race) | Eliminated (no NAT, no conntrack) |
| **CPU overhead** | Scales linearly with Service/Endpoint count | Flat, constant regardless of cluster size |
| **Data plane** | Kernel iptables / IPVS tables | In-kernel eBPF bytecode programs |
| **Per-service latency** | Increases with N services | Constant (O(1) BPF Map lookup) |

---

# Part 5: Control Plane Components (Kubernetes)

---

## 5.1 Component Deep Dive

### API Server

The API Server is the **single entry point** for all Kubernetes operations. Every `kubectl` command, every controller action, every scheduler decision — all go through the API Server.

**Responsibilities**:
- Authenticate requests (x509 certs, service accounts, OIDC tokens, bearer tokens)
- Authorize requests (RBAC, ABAC, webhook policies)
- Validate request payloads (admission controllers)
- Persist objects to etcd
- Watch/notify components of changes via the watch API

**When the cluster is unresponsive**:
- Check API Server metrics: `apiserver_request_total`, `apiserver_current_inflight_requests`
- High inflight requests → scheduler/controller manager may be blocked
- etcd latency directly impacts API Server responsiveness

### Scheduler

The Scheduler assigns pods to nodes through a three-stage pipeline:

**Stage 1 — Filtering (Predicates)**: Eliminates nodes that cannot run the pod.
```
Filtering checks:
  • Has the node got the required labels? (e.g., disk-type=ssd)
  • Are the required taints tolerable by the pod's tolerations?
  • Is there sufficient CPU/Memory on the node?
  • Does the node match nodeAffinity rules?
  • Are the required ports available?
  • Is the node in the correct topology domain?
```

**Stage 2 — Scoring (Priorities)**: Ranks the surviving nodes.
```
Scoring functions:
  • LeastRequestedPriority: prefer nodes with lowest CPU/Memory utilization
  • BalancedResourceAllocation: prefer nodes where CPU and Memory are balanced
  • ImageLocalityPriority: prefer nodes that already have container images
  • InterPodAffinityPriority: prefer nodes where pod affinity/anti-affinity rules match
  • Custom scoring via Score plugins (e.g., topology spread)
```

**Stage 3 — Binding**: The scheduler selects the highest-scoring node and creates a Binding object.
```
Binding action:
  1. Creates Binding object: {pod: "my-pod", target: {node: "worker-3"}}
  2. Writes to etcd → API Server
  3. kubelet on worker-3 picks up the Binding from etcd
  4. kubelet pulls container images and starts the pod
```

### etcd

etcd is the **single source of truth** for all Kubernetes state. It stores:
- All resource objects (Pods, Services, Deployments, ConfigMaps, Secrets)
- Status information
- Node information
- Lease information (for leader election)

**Raft Consensus Protocol**:
- etcd uses the **Raft consensus algorithm** to ensure consistency across replicas
- Standard configuration: **3 replicas** (1 leader + 2 followers)
- **Quorum = 2** (majority of 3). The cluster can tolerate 1 replica failure.
- If the leader fails, the remaining 2 followers hold an election. The one with the most up-to-date log becomes the new leader.

**Lag causes**:
- Insufficient healthy replicas (e.g., only 2 of 3 up → but they ARE the quorum, so it works; however if only 1 of 3 is up → no quorum → writes blocked)
- Disk I/O saturation on etcd leader → cannot write entries fast enough
- Network partitions isolating the leader from followers
- Heavy write load (e.g., thousands of pod creations per second)

**Critical rule**: Never run etcd with 2 replicas. With 2 replicas, a single failure = no quorum = cluster unavailable. Always use odd numbers: 3 or 5.

### Controller Manager

A collection of controllers that continuously reconcile the desired state with the actual state:

| Controller | Role |
|-----------|------|
| **ReplicaSet Controller** | Ensures exact number of pod replicas are running |
| **Deployment Controller** | Manages rolling updates, rollbacks |
| **Job Controller** | Creates pods to complete a finite task |
| **Node Controller** | Monitors node health, marks nodes unschedulable if unresponsive |
| **EndpointSlice Controller** | Updates EndpointSlice objects when pods change |
| **Service Account Controller** | Creates default service accounts in new namespaces |
| **PersistentVolume Controller** | Attaches/detaches volumes based to PVC claims |
| **Admission Controller** | Validates and mutates requests before persistence (e.g., enforce resource limits, block privileged containers) |

### Kubelet

The **node agent** running on every worker node. It:
- Registers the node with the API Server
- Watches for pods scheduled to its node
- Creates and starts containers (via container runtime: containerd, CRI-O)
- Reports node status (CPU, memory, pod count)
- Performs liveness/readiness/startup probes
- Manages volume mounts

### Kube-proxy

Responsible for **Service networking** — mapping virtual ClusterIPs to real Pod IPs.

```bash
# iptables mode (legacy):
# Creates massive iptables chains with DNAT rules
iptables -t nat -L KUBE-SERVICES | grep ClusterIP

# IPVS mode (better for large clusters):
# Uses Linux IPVS (IP Virtual Server) for O(1) hash-based load balancing
ipvsadm -ln
```

---

# Part 6: Cryptographic Fundamentals — TLS 1.3 & SSL Mechanics

---

## 6.1 Asymmetric vs Symmetric Encryption — The Hybrid Model

### Why Both Are Needed

| Cryptography Type | Speed | Use Case | Problem if Used Alone |
|-------------------|-------|----------|----------------------|
| **Asymmetric** (RSA, ECDSA, ECDHE) | ~1000x slower than symmetric | Handshake: identity verification, key exchange | Cannot encrypt gigabytes of data without melting CPUs |
| **Symmetric** (AES-GCM, ChaCha20) | Hardware-accelerated (AES-NI) | Bulk payload encryption after handshake | Key distribution problem — how to share the secret securely? |

**The hybrid solution**: Use asymmetric crypto only for the brief handshake to establish identity and exchange a shared secret. Then use symmetric crypto with that shared secret for all data transfer.

### Real-World Analogy

Imagine Alice wants to send Bob a confidential letter:
1. **Asymmetric**: Alice and Bob meet in person (expensive, slow). Bob gives Alice an open padlock (public key). Alice locks her letter in a box with Bob's padlock.
2. **Symmetric**: Once the locked box arrives, Bob has the key (shared secret derived during the "meeting"). Now Alice and Bob can exchange many locked boxes quickly using that key.

---

## 6.2 TLS 1.3 Handshake (1-RTT) — Complete Breakdown

TLS 1.3 reduced handshake latency from 2 Round Trip Times (TLS 1.2) to just 1 RTT by having the client send its key share and cipher preferences in the very first packet.

```
CLIENT                                                    SERVER
  │                                                          │
  │─── 1. ClientHello ──────────────────────────────────►│
  │      • Protocol Version: TLS 1.3                       │
  │      • Client Random (32 bytes)                        │
  │      • Supported Cipher Suites:                        │
  │        TLS_AES_256_GCM_SHA384                          │
  │        TLS_CHACHA20_POLY1305_SHA256                   │
  │      • Key Share: Client's ephemeral ECDHE public      │
  │        key (e.g., X25519 curve, 32 bytes)             │
  │      • SNI Extension: "api.example.com"               │
  │      • Supported Versions: [TLS 1.3]                  │
  │                                                          │
  │◄── 2. ServerHello ──────────────────────────────────│
  │      • Selected Cipher Suite: TLS_AES_256_GCM_SHA384  │
  │      • Server Key Share: Server's ephemeral DH public  │
  │        key (X25519)                                    │
  │    ═══════════════════════════════════════════════════  │
  │    🔒 FROM THIS POINT, ALL TRAFFIC IS ENCRYPTED 🔒    │
  │    ═══════════════════════════════════════════════════  │
  │                                                          │
  │◄── 3. EncryptedExtensions ───────────────────────────│
  │      • Server supports additional extensions           │
  │      • (none critical for basic setup)                  │
  │                                                          │
  │◄── 4. Certificate (X.509 Chain) ─────────────────────│
  │      • Leaf Cert: api.example.com                       │
  │      • Intermediate CA: Let's Encrypt R3                │
  │      • (Root CA not sent — pre-installed in trust store)│
  │                                                          │
  │◄── 5. CertificateVerify ─────────────────────────────│
  │      • Digital signature over all handshake messages   │
  │      • Proves server owns the private key               │
  │                                                          │
  │◄── 6. Finished ──────────────────────────────────────│
  │      • HMAC/Binder value confirming handshake integrity│
  │                                                          │
  │─── 7. Finished ──────────────────────────────────────►│
  │                                                          │
  │─── 8. Encrypted Application Data ────────────────────►│
  │      • HTTP/2, HTTP/3, gRPC traffic                     │
```

### Key Derivation

Both sides independently compute the shared secret using ECDHE:
```
Shared_Secret = ECDHE(ClientPrivate, ServerPublic) = ECDHE(ServerPrivate, ClientPublic)
```

This shared secret is then fed into **HKDF** (HMAC-based Extract-and-Expand Key Derivation Function) to derive:
- Client write key
- Server write key  
- Client write IV
- Server write IV

Both sides derive **identical keys** without ever transmitting them over the wire.

### Forward Secrecy

Because ephemeral ECDHE keys are used and discarded after the session, compromising the server's long-term private key **cannot** decrypt past recorded sessions. This is called **Perfect Forward Secrecy (PFS)**. TLS 1.3 mandates PFS — static RSA key exchange is not allowed.

---

## 6.3 SNI (Server Name Indication)

### The Problem

In HTTP/1.1, the `Host:` header tells the server which domain the client wants:
```
GET / HTTP/1.1
Host: api.company.com
```

But in HTTPS, the **TLS handshake happens before any HTTP headers are sent**. The server must present a certificate **before** it can read the HTTP request. If a single IP hosts 50 domains, how does it know which certificate to present?

```
WITHOUT SNI (HISTORIC DEADLOCK):

Client → IP 198.51.100.1
  └► [Nginx / ALB / Ingress Controller]
       ❌ "I host 50 websites on this IP.
          I don't know who you want because
          the HTTP Host header is inside
          the ENCRYPTED payload!"
```

```
WITH SNI (RFC 6066):

Client → IP 198.51.100.1
  └► ClientHello includes Extension: server_name = "api.company.com"
       └► [Nginx / ALB / Ingress Controller]
            ✅ "I see 'api.company.com'!
               Returning certificate for api.company.com immediately.
               Proceeding with TLS handshake."
```

SNI places the target hostname in **plaintext** inside the ClientHello extensions field, allowing the server to select the correct certificate before the handshake is completed.

### ECH (Encrypted Client Hello)

Legacy SNI exposes visited domains to network observers (ISPs, firewalls, Wi-Fi providers). **ECH** solves this by encrypting the SNI payload:
```
ClientHello.extensions:
  └► encrypted_sni: "ENC(api.company.com)" [encrypted with public key from DNS]
```

The public key for decryption is published in the domain's DNS HTTPS/SVCB resource records. Only the target server can decrypt the SNI.

---

## 6.4 mTLS (Mutual TLS)

### One-Way TLS vs mTLS

| Aspect | Standard TLS | mTLS |
|--------|-------------|------|
| Server verifies client | ❌ No | ✅ Yes (CertificateRequest) |
| Client verifies server | ✅ Yes | ✅ Yes |
| Authentication direction | One-way | Two-way |
| Use case | Public websites | Service-to-service communication |

### mTLS Handshake

```
[ Client Pod: Billing Service ]              [ Server Pod: Payment Service ]
        │                                             │
        │─── 1. ClientHello ──────────────────────►│
        │◄── 2. ServerHello + Server Certificate ─│
        │◄── 3. CertificateRequest ──────────────│  ← Server demands client identity
        │                                             │
        │─── 4. Client Certificate (X.509) ──────►│
        │      + CertificateVerify (signed with  │
        │        client's private key)            │
        │                                             │
        │      🔒 Both identities verified against │
        │         trusted root CA ────────────────►│
        │◄──── Mutual Authenticated Stream ──────│
```

### Service Mesh Implementation (Istio/Linkerd)

**SPIFFE IDs**: Each service gets a cryptographic identity as a URI embedded in the X.509 SAN:
```
spiffe://cluster.local/ns/prod/sa/billing-service-sa
```

**Ephemeral certificates**: Sidecar proxies (Envoy) receive short-lived certificates rotated every 12–24 hours from a central control-plane CA (istiod or HashiCorp Vault).

**Transparent proxying**: The application sends plain HTTP to `localhost`. The local Envoy sidecar intercepts the connection, performs mTLS with the destination Pod's sidecar, validates SPIFFE IDs, and forwards decrypted traffic to the backend container. The application is completely unaware of TLS.

---

# Part 7: Certificate Management

---

## 7.1 Certificate Chain & PKI

```
Leaf Certificate (api.example.com)
       │  signed by
       ▼
Intermediate CA (Let's Encrypt R3)
       │  signed by
       ▼
Root CA (ISRG Root X1) ← stored in OS trust stores
                            /etc/ssl/certs/ (Linux)
                            /etc/pki/tls/certs/ (RHEL)
                            Keychain (macOS)
                            Windows Certificate Store
```

The Root CA is pre-installed in operating systems and browsers. The chain of trust flows from Root → Intermediate → Leaf. Each certificate contains the public key of the next level signed by the previous level's private key.

## 7.2 ACME Protocol & Cert-Manager

### HTTP-01 Challenge (Cannot issue wildcards)
```
1. Cert-Manager creates temporary Pod + Ingress at:
   http://example.com/.well-known/acme-challenge/<token>
2. Let's Encrypt HTTP client hits port 80, fetches the token
3. Token must match the account key fingerprint
4. Proves domain control over HTTP endpoint
```

### DNS-01 Challenge (Required for wildcards)
```
1. Cert-Manager uses cloud API credentials (Route 53, Cloudflare)
2. Creates DNS TXT record: _acme-challenge.example.com → <token>
3. Let's Encrypt DNS resolver queries public DNS for the TXT record
4. Token matches → domain ownership verified
```

## 7.3 TLS Termination vs Passthrough

| Mode | Description | Security Implication |
|------|-------------|---------------------|
| **Termination** | LB/Ingress decrypts HTTPS, forwards plain HTTP to backend over private network | Backend traffic is unencrypted within the VPC (acceptable if VPC is private) |
| **Passthrough** | LB forwards encrypted packets directly to pod without decrypting | End-to-end encryption preserved; LB cannot inspect/policy traffic |

---

# Part 8: Edge Architecture & Load Balancing

---

## 8.1 Anycast Routing Explained

**Anycast** is a network addressing and routing methodology where multiple servers share the same IP address, and BGP routes traffic to the **topologically closest** server.

When Cloudflare, AWS Global Accelerator, or CloudFront advertise an anycast IP to the world:
- BGP announces the same IP prefix from hundreds of edge Points of Presence (PoPs)
- A client's packet is routed by the internet's BGP fabric to the nearest PoP
- The client is unaware they're hitting anycast — it looks like a normal unicast connection

**Example**: Cloudflare's 1.1.1.1 is advertised from 300+ locations worldwide. A user in Tokyo gets routed to the Tokyo PoP, a user in London to the London PoP, a user in São Paulo to the São Paulo PoP — all to the same IP address.

**Edge functions**:
- SSL/TLS termination at the PoP (saves backend compute)
- DDoS mitigation (volumetric attacks absorbed at the edge)
- WAF inspection (L7 rules applied before traffic reaches origin)
- CDN caching (static content served from edge cache)

## 8.2 GWLB vs NLB vs ALB

See the comparison table in Part 8 above.

---

# Part 9: Practical Labs & Verification

---

## Lab 1: Trace the Real DNS Hierarchy

```bash
# Trace the full resolution chain from root to authoritative
dig +trace api.github.com

# Look for the 3-stage referral: Root → TLD → Authoritative
# Each section shows:
#   QUESTION: what we asked
#   ANSWER: the final A record (only at the end)
#   AUTHORITY: NS referral to next level
#   ADDITIONAL: glue records (A/AAAA for nameservers)

# Inspect glue records at root level
dig +norecurse @a.root-servers.net com.
# Look in ADDITIONAL SECTION for IP addresses of .com TLD servers
```

## Lab 2: Capture Raw DNS Packets & Force TCP Fallback

```bash
# Terminal 1: Sniff all DNS traffic
sudo tcpdump -i any port 53 -nnvv

# Terminal 2: Normal small query (uses UDP)
dig @8.8.8.8 google.com A

# Large payload query forcing TCP fallback
dig @8.8.8.8 +bufsize=512 +dnssec org. SOA
# Watch tcpdump for [TC] (Truncated) flag in UDP → then TCP 3-way handshake
```

## Lab 3: Trigger 502 via Keep-Alive Mismatch

```javascript
// app.js
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World\n');
});
server.keepAliveTimeout = 2000;  // Intentionally 2 seconds
server.headersTimeout = 2500;
server.listen(3000);
```

```bash
node app.js &
curl -v --keepalive-time 10 http://localhost:3000 http://localhost:3000
# Second request after 2s timeout → observe TCP RST → HTTP 502
```

## Lab 4: Inspect TLS 1.3 Handshake

```bash
openssl s_client -connect google.com:443 -servername google.com -tls1_3
# Look for: TLSv1.3, ECDHE key share, certificate chain, SNI
```

---

# Part 10: Systematic Failure Mapping

## Failure Scenarios & Resolutions

| Scenario | Layer | Manifestation | Root Cause | Resolution |
|----------|-------|---------------|------------|------------|
| Apex CNAME Breakage | DNS | Email fails (SERVFAIL), broken delegation | RFC 1034 collision at zone apex | Use Route 53 Alias or Cloudflare CNAME Flattening |
| 5-Second K8s DNS Lag | Kernel/Netfilter | Intermittent 5.002s latency spikes | conntrack tuple collision on parallel UDP lookups | Deploy NodeLocal DNSCache |
| Random HTTP 502 | L7/Proxy | Intermittent 502 Bad Gateway | Keep-Alive timeout mismatch | App Keep-Alive (65s) > Ingress (65s) > ALB (60s) |
| Connection Timed Out | L4/Transport | ETIMEDOUT / Handshake fail | TCP SYN backlog queue full | Increase `somaxconn`, `tcp_max_syn_backlog` |
| TTL Expired in Transit | L3/Network | ICMP Type 11 Code 0 | Routing loop or saturated hop exceeding TTL | Fix routing, increase path MTU |

---

# Part 11: The Downward-Tracing Diagnostic Blueprint

For technical interviews and incident response, apply this 4-tier model:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   THE DOWNWARD-TRACING RESPONSE                      │
├───────────────────┬─────────────────────────────────────────────────┤
│ 1. EDGE LAYER     │ "Check Route 53 DNS TTLs, Anycast PoPs, ALB  │
│                   │  Keep-Alive timeouts, and TLS SNI              │
│                   │  negotiation..."                                │
├───────────────────┼─────────────────────────────────────────────────┤
│ 2. PLATFORM       │ "Check K8s Service Endpoints, CoreDNS         │
│   LAYER           │  metrics, Readiness Probes, and ArgoCD        │
│                   │  sync drift..."                               │
├───────────────────┼─────────────────────────────────────────────────┤
│ 3. CONTAINER      │ "Inspect containerd CRI execution and Cgroups │
│   LAYER           │  v2 memory.high CPU throttling vs             │
│                   │  memory.max OOM..."                           │
├───────────────────┼─────────────────────────────────────────────────┤
│ 4. OS / KERNEL    │ "Verify netfilter conntrack UDP race          │
│   LAYER           │  conditions, ss -tulpn socket backlogs,       │
│                   │  and disk I/O wait..."                        │
└───────────────────┴─────────────────────────────────────────────────┘
```

---

# Part 12: CI/CD & Internal Systems

## CI/CD Pipeline

```
CI: Git → Code Quality (NIST CVE, Pentesting, Compliance) → Docker Build → Artifactory Signing → ECR
CD: Git → DB Update → Argo Sync
```

## Internal Applications

| App | Purpose |
|-----|---------|
| CTIP | Coaching |
| CRM | 3rd party application |
| ReedyMix | Internal food coupons |
| Blood Donation | Internal voluntary service |
| Mfit | New joinee training & records |
| Abhayahastam | NGO application |

---

*Compiled from study sessions: 2026-08-23 through 2026-08-28*
