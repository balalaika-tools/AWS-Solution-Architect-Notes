# Networking Primer — IP, CIDR, Subnets, Routing, Ports, NAT, Firewalls

> **Who this is for**: Engineers with little or no networking background who are about to learn
> AWS VPC. This file contains **almost no AWS** — it builds the mental model you need first.
> Read it slowly. Everything in the rest of section 3 assumes you understand it.

---

## 1️⃣ What an IP Address Is

Every device on a network needs a unique number so that data can be addressed to it — exactly like a postal address. That number is the **IP address**. The version you must know for the exam is **IPv4**.

An IPv4 address is **32 bits** long. We write it as four numbers (each 0–255) separated by dots, called **dotted-decimal** notation:

```
       10  .  0  .  1  .  25
    ┌──────┬──────┬──────┬──────┐
    │00001010│00000000│00000001│00011001│   ← the same 32 bits in binary
    └──────┴──────┴──────┴──────┘
      8 bits   8 bits   8 bits   8 bits     ← each group is one "octet"
```

- Each of the four numbers is one **octet** (8 bits).
- 8 bits can represent 0–255, which is why no octet ever exceeds 255.
- Total addresses possible: 2³² ≈ 4.3 billion. (This shortage is why IPv6 exists — see section 6.)

> **Key insight**: An IP address has two parts — a **network portion** (which network you're on) and a **host portion** (which specific device on that network). The split between them is *not* fixed; it's defined by the CIDR prefix, covered next. This single idea unlocks subnetting.

---

## 2️⃣ CIDR Blocks — How We Describe a Range of IPs

You rarely talk about one IP; you talk about a **range** of them — "this network owns these addresses." We describe a range with **CIDR notation** (Classless Inter-Domain Routing):

```
   10.0.0.0 / 16
   └──────┘  └┘
   base IP   prefix length (how many leading bits are the NETWORK portion)
```

The number after the slash (the **prefix length**) says how many of the 32 bits are fixed as the network. The remaining bits are free to vary — those are your usable host addresses.

```
/16  →  16 network bits fixed,  16 host bits free
        10.0.?.?   →   10.0.0.0  through  10.0.255.255
```

A **smaller prefix number = bigger network** (fewer bits locked down, more addresses). A **bigger prefix number = smaller network**. This trips people up constantly:

⚠️ `/16` is **larger** than `/24`, even though 24 is the bigger number. Think "how many bits are *locked*": more locked bits → fewer free addresses → smaller network.

### Reading the two most common prefixes

```
10.0.0.0/16              10.0.5.0/24
─────────                ─────────
network = 10.0           network = 10.0.5
host    = last two       host    = last octet only
          octets         (0–255)
65,536 addresses         256 addresses
```

### Common prefix sizes → address counts

| CIDR  | Network bits | Host bits | Total addresses | Typical use |
|-------|-------------|-----------|-----------------|-------------|
| `/8`  | 8  | 24 | 16,777,216 | Entire `10.x.x.x` block |
| `/16` | 16 | 16 | 65,536 | A whole VPC (common default) |
| `/20` | 20 | 12 | 4,096 | Large subnet |
| `/24` | 24 | 8  | 256 | A typical subnet |
| `/26` | 26 | 6  | 64 | Small subnet |
| `/28` | 28 | 4  | 16 | Smallest practical subnet |
| `/32` | 32 | 0  | 1 | A single host (one exact IP) |

> **Rule of thumb**: each step the prefix *decreases by 1*, the address count **doubles**. `/24` = 256, `/23` = 512, `/22` = 1024. Each step it *increases* by 1, the count halves.

💡 You do **not** need to do binary subnet math under exam pressure. You need to (a) read a prefix, (b) know roughly how big it is, and (c) remember `/16`–`/28` is the legal VPC range in AWS.

---

## 3️⃣ Private vs Public IP Ranges (RFC1918)

Not every IP is reachable from the public internet. Certain ranges are reserved as **private** — they are reused inside millions of separate networks (your home Wi-Fi, every corporate LAN, every AWS VPC) and are **never routed on the public internet**. These are defined by a standard called **RFC 1918**:

| Private range (RFC 1918) | CIDR | Size |
|--------------------------|------|------|
| `10.0.0.0` – `10.255.255.255`     | `10.0.0.0/8`     | ~16.7M addresses |
| `172.16.0.0` – `172.31.255.255`   | `172.16.0.0/12`  | ~1M addresses |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | 65,536 addresses |

- **Private IPs** are for internal communication. Two different companies can both use `10.0.0.0/16` without conflict because the ranges never leave their networks.
- **Public IPs** are globally unique and routable on the internet. They are allocated by registries (and, in AWS, leased to you).

✅ You should build your AWS VPC using one of these private ranges (almost everyone uses `10.0.0.0/16`).

⚠️ If you ever want two private networks to talk *directly* (e.g., VPC peering), their ranges **must not overlap** — there'd be no way to tell which `10.0.1.5` you meant. This rule comes back repeatedly in AWS.

---

## 4️⃣ Subnets — Why We Subdivide a Network

A **subnet** ("sub-network") is a slice of a larger network's address range. We take a big block like `10.0.0.0/16` and carve it into smaller pieces like `10.0.1.0/24`, `10.0.2.0/24`, and so on.

```
            VPC: 10.0.0.0/16   (65,536 addresses)
            ┌───────────────────────────────────────┐
            │                                         │
   Subnet A │ 10.0.1.0/24   Subnet B │ 10.0.2.0/24   │
   ┌────────┴────────┐      ┌─────────┴───────┐       │
   │ 256 addresses   │      │ 256 addresses   │       │
   └─────────────────┘      └─────────────────┘       │
            │                                         │
            └───────────────────────────────────────┘
```

Why bother subdividing?

1. **Isolation / security** — put public-facing servers in one subnet and databases in another, then apply different firewall rules to each.
2. **Routing control** — different subnets can have different paths to the outside world (this is exactly how "public" and "private" subnets are made in AWS — section 2).
3. **Fault domains** — in AWS, each subnet lives in one Availability Zone, so spreading subnets across AZs spreads risk.

> **Key insight**: A subnet's CIDR must fit *inside* its parent network's CIDR and must not overlap any sibling subnet. `10.0.1.0/24` and `10.0.2.0/24` both fit inside `10.0.0.0/16` and don't collide.

---

## 5️⃣ Routing and Route Tables

Having addresses isn't enough — devices need to know **where to send a packet** to reach a given destination. That decision is made by a **route table**: a list of rules mapping *destination ranges* to *next hops*.

A route table entry answers one question: *"For a packet headed to destination X, where do I send it next?"*

```
┌──────────────────────────┬────────────────────────┐
│ Destination (CIDR)        │ Target (next hop)       │
├──────────────────────────┼────────────────────────┤
│ 10.0.0.0/16               │ local (stay inside)     │
│ 0.0.0.0/0                 │ internet gateway        │
└──────────────────────────┴────────────────────────┘
```

How a router reads it: it checks the packet's destination IP against every rule and picks the **most specific match** — the one with the longest prefix (more locked bits = more specific). This is called **longest-prefix match**.

```
Packet to 10.0.5.20:
   matches 10.0.0.0/16  (specific)   ✅  ← chosen, it's the longer prefix
   matches 0.0.0.0/0    (everything) ❌  ← only used if nothing better matches
```

### The default route: `0.0.0.0/0`

`0.0.0.0/0` means **"every possible IP address"** — a prefix of 0 locks zero bits, so it matches literally everything. Because it's the *least* specific route, it acts as the **catch-all / default route**: "if no more specific rule matched, send the packet here." In practice this is the path to the internet.

> **Rule**: `0.0.0.0/0` = "the whole internet" = the default route. Memorize this — it appears in nearly every AWS networking question.

💡 A subnet becomes "public" or "private" in AWS purely based on whether its route table has a `0.0.0.0/0` route pointing at an internet gateway. That's the whole secret (section 2 and 3).

---

## 6️⃣ Ports and Protocols

An IP address gets a packet to the right *machine*. A **port number** gets it to the right *application* on that machine. A server might run a web server, a database, and an SSH daemon simultaneously — ports keep their traffic separate.

A connection is therefore identified by the combination: `source IP : source port → destination IP : destination port`.

### TCP vs UDP

These are the two transport **protocols** you must know:

| | TCP | UDP |
|---|-----|-----|
| Connection | Connection-oriented (handshake first) | Connectionless (fire and forget) |
| Reliability | Guaranteed, ordered delivery, retransmits | No guarantee, no ordering |
| Overhead | Higher | Lower |
| Typical use | Web, SSH, databases, email | DNS, video/voice streaming, gaming |

### Common ports for the exam

| Port | Protocol | Service |
|------|----------|---------|
| `22`   | TCP | SSH (secure shell — Linux admin) |
| `80`   | TCP | HTTP (unencrypted web) |
| `443`  | TCP | HTTPS (encrypted web / TLS) |
| `3389` | TCP | RDP (Windows remote desktop) |
| `3306` | TCP | MySQL / Aurora MySQL |
| `5432` | TCP | PostgreSQL |
| `53`   | TCP/UDP | DNS |

### Ephemeral ports (this matters for firewalls later)

When *your* machine initiates a connection (e.g., to a web server on port 443), your machine picks a random high-numbered **ephemeral port** as its *source* port — typically in the range `1024–65535`. The server's reply comes *back* to that ephemeral port.

> **Key insight**: Outbound requests use a well-known destination port (443) but a random source ephemeral port. The **return traffic** therefore arrives on that high ephemeral port, not on 443. A firewall that doesn't account for return traffic will silently drop replies — this is the single biggest gotcha with stateless firewalls (section 4).

---

## 7️⃣ NAT — Network Address Translation

Private IPs can't be routed on the internet. So how does a machine with a private address (`10.0.5.20`) browse the web? Through **NAT**.

A NAT device sits at the network's edge with a **public** IP. When a private host sends a packet out:

1. NAT **rewrites the source** from the private IP to its own public IP, remembering the mapping.
2. The internet server replies to the public IP.
3. NAT looks up the mapping and **rewrites the destination** back to the original private IP, delivering the reply.

```
Private host          NAT device              Internet
10.0.5.20             public: 52.1.2.3
   │                       │                      │
   │ src=10.0.5.20  ──────>│                       │
   │                       │ src=52.1.2.3 ────────>│   (private IP hidden)
   │                       │<──── dst=52.1.2.3 ────│
   │<── dst=10.0.5.20 ─────│   (mapping reversed)  │
```

- NAT lets many private hosts share **one** public IP for **outbound** access.
- Crucially, NAT is **outbound-initiated only**: the internet cannot start a connection *to* a private host through NAT, because there's no pre-existing mapping. This is a security benefit — private servers reach out but can't be reached.

💡 In AWS this exact behavior is provided by a **NAT Gateway** (section 3): it lets private-subnet instances download updates and call APIs while remaining unreachable from the internet.

---

## 8️⃣ Firewalls — Stateful vs Stateless

A **firewall** filters traffic, allowing or blocking packets based on rules (source/destination IP, port, protocol). The one distinction the exam cares about deeply is **stateful** vs **stateless**.

### Stateful firewall

Remembers connections it has seen. If it **allows an outbound request**, it **automatically allows the matching reply** — you don't write a rule for return traffic.

```
You allow:  OUT  to 443
Reply on the ephemeral port comes back  →  ✅ allowed automatically
                                            (firewall remembers the connection)
```

### Stateless firewall

Treats every packet independently with **no memory**. If you allow the outbound request, the **inbound reply is still evaluated separately** — and will be blocked unless you *also* added an inbound rule covering the ephemeral return ports.

```
You allow:  OUT  to 443
Reply on ephemeral port 51000 comes back  →  ❌ DROPPED
                                              unless you ALSO allow
                                              IN on ports 1024–65535
```

| | Stateful | Stateless |
|---|----------|-----------|
| Connection memory | Yes — tracks established flows | No — each packet judged alone |
| Return traffic | Auto-allowed | Must be explicitly allowed |
| Rule complexity | Simpler | Must handle ephemeral ports both directions |
| AWS equivalent | **Security Group** | **Network ACL** |

> **Rule**: Stateful = "allow the request, the reply is free." Stateless = "you must allow both directions yourself, including the ephemeral return ports." This *is* the Security Group vs NACL distinction in section 4.

⚠️ The classic mistake with stateless firewalls is allowing inbound 443 but forgetting the outbound rule for the ephemeral ports the reply leaves on — connections appear to hang.

---

## 9️⃣ DNS — One Sentence For Now

**DNS** (Domain Name System) translates human-friendly names like `example.com` into IP addresses. It's the "phone book" of the internet. Networking and DNS are tightly linked, but DNS is large enough to deserve its own treatment.

For the full mental model — resolution flow, record types, TTL — see **[DNS Fundamentals](../08_dns_edge/01_dns_fundamentals.md)**.

---

## 🔟 Key Exam Points

- IPv4 = 32 bits, four octets, each 0–255.
- CIDR prefix = number of **network** bits. **Smaller prefix number = larger network.** `/16` > `/24`.
- Each `-1` to the prefix **doubles** the address count.
- RFC1918 private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Build VPCs from these.
- Overlapping CIDRs **cannot** be directly connected (peering, VPN). No exceptions.
- `0.0.0.0/0` = the entire internet = the **default route**.
- Routers pick the **longest-prefix (most specific)** matching route.
- TCP = reliable/ordered; UDP = fast/unreliable. Know ports 22, 80, 443, 3389, 3306, 5432, 53.
- **Return traffic uses ephemeral ports (1024–65535)**, not the original destination port.
- **Stateful** firewall auto-allows replies; **stateless** does not — you must allow both directions.

---

## Common Mistakes

- ❌ Thinking `/24` is bigger than `/16` because 24 > 16. It's the opposite.
- ❌ Assuming a private IP can be reached from the internet. It can't — that's the point of RFC1918 and NAT.
- ❌ Forgetting return traffic on a stateless firewall. The reply lands on an ephemeral port you didn't allow.
- ❌ Designing two networks with overlapping CIDRs and expecting to connect them later.
- ❌ Believing NAT lets the internet initiate connections inward. NAT is outbound-initiated only.

---

**Next**: [02_vpc_subnets_route_tables.md — Building the Virtual Network in AWS](02_vpc_subnets_route_tables.md)
