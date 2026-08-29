# Networking & Security Fundamentals Syllabus

A comprehensive 6-stage curriculum covering DNS, TLS, Certificate Automation, Kubernetes Networking, and AWS Load Balancing.

---

## Stage 1: DNS Fundamentals & Packet Mechanics

**Goal:** Understand how domain names map to IP addresses at the protocol level.

### Recursive Lookup Flow (Step-by-Step)
```
Client → OS DNS Cache → Recursive Resolver (8.8.8.8) → Root (.) → TLD (.com) → Authoritative (example.com)
```

### Iterative vs Recursive Queries
| Query Type | Description |
|------------|-------------|
| **Recursive** | Resolver does all the work, returns final answer to client |
| **Iterative** | Resolver returns referral, client continues querying |

### Core DNS Record Types

| Type | Purpose | Notes |
|------|---------|-------|
| **A** | IPv4 address | |
| **AAAA** | IPv6 address | |
| **CNAME** | Canonical alias | Cannot exist at zone apex (collides with SOA/NS per RFC 1034) |
| **TXT** | Arbitrary text | SPF, DKIM, DMARC verification |
| **PTR** | Reverse DNS | IP → domain mapping |
| **MX** | Mail exchange | Priority-based routing |
| **SRV** | Service location | Specifies port, weight, priority |

### Transport Protocols
- **UDP Port 53** — Default (speed, low overhead)
- **TCP Port 53** — Fallback for responses >512 bytes (no EDNS0), zone transfers (AXFR/IXFR)

---

## Stage 2: Enterprise Traffic Management & DNS Migrations

**Goal:** Learn how high-traffic infrastructure uses DNS for routing and zero-downtime failovers.

### TTL & Cache Invalidation
- **High TTL** (86400s): Low query load, slow failover
- **Low TTL** (60s): High query load, fast failover

### Zero-Downtime Migration Strategy
```
Scenario: Migrate Server A (1.1.1.1) → Server B (2.2.2.2)

Step 1: Reduce TTL 86400s → 60s (24h before migration)
Step 2: Update A record to 2.2.2.2
Step 3: Keep Server A alive 60s while traffic drains
Step 4: Increase TTL back to standard values
```

### Global Traffic Management (GTM)
- **Latency-Based Routing** — Return IPs closest to user geography
- **Weighted Routing / Canary** — Split traffic (e.g., 90% old, 10% new)
- **Health Checks & Failover** — Auto-remove unhealthy endpoints (Route 53, Cloudflare)

---

## Stage 3: Kubernetes Internal DNS (CoreDNS & NodeLocal DNSCache)

**Goal:** Master cluster-internal service discovery and troubleshoot networking bottlenecks.

### Inside the Pod (`/etc/resolv.conf`)
```bash
nameserver 10.96.0.10          # kube-dns service IP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### The `ndots:5` Latency Overhead
External query `api.stripe.com` triggers **4 failed internal lookups** before external resolution:
1. `api.stripe.com.default.svc.cluster.local`
2. `api.stripe.com.svc.cluster.local`
3. `api.stripe.com.cluster.local`
4. `api.stripe.com` → finally resolves

### Linux Kernel conntrack UDP Race Condition
**Problem:** High-concurrency UDP DNS queries (parallel A + AAAA) from pods using same socket  
**Mechanism:** netfilter race in SNAT/DNAT tuple allocation locks in conntrack table → drops UDP packet → 5-second timeout fallback

### NodeLocal DNSCache Architecture
- Runs CoreDNS as DaemonSet on every node (IP: `169.254.20.10`)
- Converts UDP → TCP locally on node
- Bypasses conntrack tuple locks entirely
- **Eliminates 5-second DNS timeouts**

---

## Stage 4: Cryptographic Fundamentals, TLS & SSL Mechanics

**Goal:** Understand how data in transit is encrypted and verified.

### Asymmetric vs Symmetric Encryption
| Type | Use Case | Performance |
|------|----------|-------------|
| **Asymmetric** (Public/Private) | Handshake only — identity, key exchange | Slow, heavy computation |
| **Symmetric** (Session Key) | Bulk payload encryption | Fast, hardware-accelerated |

### TLS 1.3 Handshake (1-RTT)
```
1. ClientHello: Cipher suites, key shares (DH params), SNI
2. ServerHello: Selected cipher, server key share, X.509 Certificate
3. Certificate Verification: Chain validation against OS trust stores
4. Session Key Generation: ECDHE → shared symmetric key → encrypted channel
```

### SNI (Server Name Indication)
- Sends target hostname in **plaintext** inside `ClientHello`
- Enables single IP / Load Balancer to host multiple TLS certificates
- Required for multi-tenant HTTPS on shared infrastructure

### mTLS (Mutual TLS)
- Two-way authentication: Server verifies client cert, client verifies server cert
- Used in Service Meshes (Istio/Linkerd) for zero-trust microservice security
- Transparent via Envoy sidecars + SPIFFE/SPIRE identities

---

## Stage 5: Certificate Management, Automation & Security

**Goal:** Automate certificate lifecycles and manage trust boundaries.

### Certificate Chain & PKI
```
Leaf Certificate → Intermediate CA → Root CA
```
- Root CAs stored in OS trust stores: `/etc/ssl/certs/` or `/etc/pki/tls/certs/`

### Certificate Automation (ACME / Let's Encrypt)

| Challenge | Method | Wildcard Support |
|-----------|--------|------------------|
| **HTTP-01** | Temporary pod + Ingress at `/.well-known/acme-challenge/<token>` | ❌ No |
| **DNS-01** | Cloud API (Route 53, Cloudflare) creates `_acme-challenge` TXT record | ✅ Yes |

### Cert-Manager in Kubernetes
- **ClusterIssuer** — CA configuration (cluster-scoped)
- **Certificate** — Requests domains, links to target K8s Secret
- Auto-renews before 90-day expiry threshold

### TLS Termination vs Passthrough
| Mode | Description |
|------|-------------|
| **Termination** | Decrypt at LB/Ingress, pass plain HTTP to backend (private network) |
| **Passthrough** | Forward encrypted packets directly to pod without edge decryption |

---

## Stage 6: Practical CLI Toolkit & Hands-on Labs

### 1. Trace DNS Resolution Chain
```bash
# Full recursive chain from Root → Authoritative
dig +trace example.com

# Specific record types
dig example.com CNAME
dig example.com TXT
dig example.com MX
```

### 2. Test Port Connectivity & Network Path
```bash
# TCP connection test
nc -zv api.example.com 443

# HTTP/HTTPS with headers
curl -iv https://api.example.com
```

### 3. Inspect TLS Certificates & Handshakes
```bash
# Full certificate details, expiration, chain
openssl s_client -connect example.com:443 -servername example.com

# Local cert expiration
openssl x509 -in /path/to/cert.crt -noout -dates
```

### 4. Monitor Live System Sockets & Traffic
```bash
# Active listening ports
ss -tulpn

# Capture DNS UDP/TCP packets
sudo tcpdump -i eth0 port 53 -n
```

---

## Knowledge Verification Checklist

Confirm you can clearly answer these 4 scenarios:

1. **Zero-downtime domain IP migration** across cloud providers?
2. **5-second DNS delays** in Kubernetes — root cause and NodeLocal DNSCache fix?
3. **TLS 1.3 handshake** mechanics and why **SNI** enables multi-domain on single IP?
4. **HTTP-01 vs DNS-01 ACME challenges** for wildcard certs via Cert-Manager?

---

## References
- [Istio Security Concepts](https://istio.io/latest/docs/concepts/security/)
- [TLS 1.3 Explained](https://tls13.xargs.org/)
- [Cloudflare DNS Learning](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [DevOpsBeast Networking Fundamentals](https://devopsbeast.com/courses/networking-fundamentals)
- [KodeKloud CoreDNS Mastery](https://kodekloud.com/courses/learn-by-doing-core-dns-mastery)

---

## Appendix: Layer 4 vs Layer 7 Traffic (AWS NLB vs ALB / Nginx / Envoy)

**Required Depth:** 100% (High Priority for System Design)

### Must Explain
- **OSI Boundary**: L4 routes by IP/port/sequence; L7 parses HTTP payload/headers/cookies/paths
- **AWS NLB (L4)**: Hyperplane hardware, ultra-low latency, static IPs/AZ, millions RPS, gRPC/TCP/UDP
- **ALB/Envoy/Nginx (L7)**: Path routing, header rewrites, cookie stickiness, rate-limiting, higher CPU
- **Packet Journey**: NLB → Nginx Ingress (TLS termination, header inspection) → Pod

### Can Skip
- Advanced BGP, custom raw socket C code (unless Core Edge Network Engineer)

---

## Appendix: TLS 1.3, SNI & mTLS

**Required Depth:** High (Security & Service Mesh)

### Must Explain
- **TLS 1.3 vs 1.2**: 1-RTT vs 2-RTT; key shares in ClientHello; ECDHE forward secrecy
- **SNI**: Hostname in plaintext `ClientHello` → LB selects correct cert before encryption
- **mTLS**: Two-way auth; Istio/Linkerd use Envoy + SPIFFE/SPIRE without app code changes

### Can Skip
- ECDHE mathematical proofs (know it provides Forward Secrecy)

---

## Appendix: Certificate Lifecycle Automation (Cert-Manager / ACME)

**Required Depth:** Medium-High (K8s & DevSecOps Standard)

### Must Explain
- **ACME Challenges**: HTTP-01 (port 80, no wildcards) vs DNS-01 (TXT records, wildcards OK)
- **Cert-Manager CRs**: `ClusterIssuer` vs `Certificate` → K8s Secret
- **Renewal**: Automatic before 90-day expiry

### Can Skip
- Writing ACME clients from scratch

---

## Appendix: Keep-Alive Timeout Mismatch (Preventing 502 Errors)

**Required Depth:** 100% (#1 Practical Scenario)

### The Race Condition
```
ALB Idle Timeout: 60s (default)
Backend App Keep-Alive: < 60s (e.g., Node.js 5s default)

→ App closes socket first
→ Client request hits ALB at exact moment of close
→ ALB routes to dead connection → TCP RST → HTTP 502 Bad Gateway
```

### Golden Rule
```
App Keep-Alive > Ingress Proxy Timeout > ALB Idle Timeout (60s)
```

### Exact Config Values
| Platform | Config |
|----------|--------|
| **ALB** | 60 seconds |
| **Node.js/Express** | `server.keepAliveTimeout = 65000`, `server.headersTimeout = 66000` |
| **Python/Gunicorn** | `gunicorn --keep-alive 65` |

### Interview Scenarios

| Scenario | Target Answer |
|----------|---------------|
| **Random 502s on Node.js behind ALB, normal CPU/mem** | Keep-Alive race condition; set `keepAliveTimeout > 60s` |
| **Wildcard cert (*.dev.company.com) in EKS via Cert-Manager** | Use **DNS-01**; HTTP-01 cannot prove wildcard control; DNS-01 writes TXT to Route 53 |
| **SNI for 50 domains on single IP with Nginx Ingress** | SNI puts hostname in plaintext `ClientHello`; Nginx inspects and selects matching TLS secret |

---

## Appendix: AWS Gateway Load Balancer (GWLB)

**Specialized L3 Load Balancer** for third-party virtual network appliances (firewalls, DPI, IDS/IPS).

### Key Concepts
| Aspect | Detail |
|--------|--------|
| **OSI Layer** | Layer 3 (Network) |
| **Protocol** | GENEVE encapsulation (UDP port 6081) |
| **Encapsulation** | Wraps original L3 packet in GENEVE header, preserves metadata |
| **VPC Integration** | GWLB Endpoints (GWLBE) via PrivateLink; central Security VPC |

### Traffic Flow
```
[Internet] → [IGW] → [GWLBE] → [GWLB Security VPC] → [GENEVE:6081] → [Firewalls]
                                                              ↓
[App VPC Subnet] ← [Original packet returned unchanged] ←────┘
```

### GWLB vs NLB vs ALB

| Feature | GWLB | NLB | ALB |
|---------|------|-----|-----|
| **OSI Layer** | Layer 3 | Layer 4 | Layer 7 |
| **Protocol** | GENEVE (6081) | TCP/UDP/TLS | HTTP/HTTPS/gRPC |
| **Use Case** | Inline firewalls, IDS/IPS | Low-latency APIs, gaming | Microservices, path routing |
| **Traffic Mod** | Transparent | Header mods if TLS/NAT | SSL term, X-Forwarded-For |
| **Targets** | Appliance EC2/IP | EC2, IP, ALB | EC2, Containers, Lambda |

### Core Interview Q&A

**Q1: Why GWLB instead of firewalls in public subnets?**
> GWLB decouples appliances from app network. Auto load-balances, health-checks, scales firewall fleet.

**Q2: How does GWLB preserve original source/dest IP?**
> GENEVE encapsulation (UDP 6081) wraps original L3 packet; appliance inspects inner packet, returns original untouched.

**Q3: Sticky connections for stateful firewalls?**
> 2/3/5-tuple flow hashing ensures bidirectional traffic hits same appliance instance for state table consistency.

---
