# DNS Fundamentals (No AWS Yet)

> **Who this is for**: Engineers with a weak networking/DNS background who are about to learn Route 53. This file is **pure prerequisites** — no AWS. By the end you should be able to explain how a name like `www.example.com` becomes an IP address, what a record is, what TTL does, and why you can't put a `CNAME` at a zone apex. Everything in [02_route_53.md](02_route_53.md) assumes you know this.

---

## 1. What DNS Is and Why It Exists

Computers talk to each other using **IP addresses** — numbers like `93.184.216.34` (IPv4) or `2606:2800:220:1:248:1893:25c8:1946` (IPv6). Humans are terrible at remembering numbers, and worse, the number behind a service changes over time (you redeploy, you scale, you fail over to another region).

**DNS (Domain Name System)** is the internet's phone book: it maps human-friendly **names** to machine-usable **IP addresses** (and other data). It lets you type `www.example.com` instead of an IP, and it lets the owner of that name change the underlying IP without you ever knowing.

> **Mental model**: DNS is a giant, distributed, cached key→value lookup. The key is a name + record type (`www.example.com`, type `A`). The value is whatever that record holds (an IP, another name, mail server info, etc.).

Two properties make DNS work at internet scale:

- **Distributed** — no single server holds all of the internet's records. Responsibility is delegated down a hierarchy (explained in §3).
- **Cached** — answers are stored temporarily at many layers so the same question doesn't hit the authoritative source every time. This is what TTL controls (§6).

💡 DNS primarily uses **UDP port 53** (fast, connectionless) for normal queries, falling back to **TCP port 53** for large responses (e.g. zone transfers or DNSSEC). For the exam, just remember "DNS = port 53."

---

## 2. Anatomy of a Domain Name (FQDN & Hierarchy)

Read a domain name **right to left** — it's a hierarchy, and the rightmost label is the most significant.

```
        www  .  example  .  com  .
         │        │         │     │
   subdomain   second-    top-   root
   (host)      level     level   (the trailing dot,
              domain    domain    usually implied)
              (SLD)     (TLD)
```

- **Root** — the top of the tree, written as a single trailing dot (`.`). It's almost always omitted, but it's there.
- **TLD (Top-Level Domain)** — `.com`, `.org`, `.net`, `.io`, `.uk`. Managed by registry operators.
- **SLD (Second-Level Domain)** — `example`. This is the part you register and own.
- **Subdomain / host** — `www`, `api`, `mail`. You create these freely under a domain you own.

A name written out fully, all the way to the root, is a **Fully Qualified Domain Name (FQDN)** — e.g. `www.example.com.` (note the trailing dot). An FQDN is unambiguous; a partial name like `www` is relative and needs context.

> **Key term — zone apex (a.k.a. naked domain / root domain)**: the domain itself with no subdomain, e.g. `example.com`. This sounds trivial but it causes a real technical limitation you'll hit in §8.

---

## 3. The Resolution Flow: How a Name Becomes an IP

This is the single most important diagram in this file. When your browser needs the IP for `www.example.com`, a chain of servers cooperates. Four kinds of player are involved:

| Player | Role |
|--------|------|
| **Stub resolver** | The tiny DNS client built into your OS. It doesn't do the legwork; it just asks the recursive resolver and waits for an answer. |
| **Recursive resolver** | The workhorse (run by your ISP, your company, or a public one like `8.8.8.8`). It does *all* the chasing on your behalf and **caches** results. |
| **Root / TLD servers** | Directory servers that don't know the final answer but know *who to ask next* (a referral). |
| **Authoritative name server** | The server that holds the actual records for a domain. Its answer is the truth. |

```
  ┌──────────┐   "what is www.example.com?"
  │  Browser │──────────────┐
  │  / app   │              │
  └──────────┘              ▼
                    ┌─────────────────┐
                    │  Stub resolver  │  (in your OS)
                    │                 │
                    └────────┬────────┘
                             │ asks (recursive query)
                             ▼
                    ┌─────────────────────┐
                    │ Recursive resolver  │◄── caches every answer
                    │ (ISP / 8.8.8.8)     │    for its TTL
                    └──────┬──────────────┘
       ┌───────────────────┼────────────────────────┐
       │ 1. ask ROOT       │                        │
       ▼                   │                        │
  ┌──────────┐  "I don't know, but for .com         │
  │  Root    │   ask the TLD servers here" ─────────┤
  │  servers │   (a referral, not the answer)       │
  └──────────┘                                      │
       │ 2. ask .com TLD                            │
       ▼                                            │
  ┌──────────┐  "I don't know the IP, but           │
  │ .com TLD │   example.com's authoritative NS     │
  │  servers │   is ns1.example-dns.com" ───────────┤
  └──────────┘   (another referral)                 │
       │ 3. ask the authoritative NS                │
       ▼                                            │
  ┌────────────────────┐  "www.example.com is       │
  │ Authoritative NS   │   A 93.184.216.34,         │
  │ for example.com    │   TTL 300" ────────────────┘
  └────────────────────┘   (the actual answer)
                             │
                             ▼
              Recursive resolver caches it,
              returns it to the stub resolver,
              which hands the IP to the browser.
```

Two query *styles* appear here:

- **Recursive query** — "Give me the final answer, do whatever it takes." The stub resolver makes this to the recursive resolver.
- **Iterative query** — "Give me the best you've got (an answer or a referral)." The recursive resolver makes these to root → TLD → authoritative, walking down the hierarchy one referral at a time.

> **Key insight**: Root and TLD servers never know the IP of `www.example.com`. They only know *where to delegate next*. The authoritative name server is the only place the real record lives. DNS scales because each level only has to know about the level directly below it.

⚠️ Common confusion: the recursive resolver is *not* authoritative for anything — it's a caching middleman. The authoritative server *is* the source of truth and serves answers from the zone's owner.

---

## 4. Zones, Authority, and Delegation

A **zone** is the slice of the DNS namespace that one administrator controls, stored in a **zone file** (a list of records). When you register `example.com`, you get authority over the `example.com` zone — you decide what `www`, `api`, `mail`, etc. resolve to.

**Delegation** is how authority is handed down the tree. The `.com` TLD doesn't store your records; it stores `NS` records (§5) pointing at *your* authoritative name servers. That's the referral the TLD gave back in §3. You can further delegate a subdomain (e.g. hand `dev.example.com` to a different team's name servers) by adding `NS` records in your zone.

> **Rule**: Whoever holds the `NS` records that the parent points to is authoritative for that zone. Change those `NS` records at the registrar and you've moved your DNS hosting.

---

## 5. DNS Record Types

A zone is just a collection of **records**. Each record has a **name**, a **type**, a **TTL**, and a **value** (the "RDATA"). The type tells you what kind of answer it is. These are the ones the exam cares about:

| Type | Stands for | Maps a name to… | Example value | Notes |
|------|-----------|-----------------|---------------|-------|
| **A** | Address | an **IPv4** address | `93.184.216.34` | The most common record. |
| **AAAA** | "quad-A" | an **IPv6** address | `2606:2800:220::1` | IPv6 equivalent of `A`. |
| **CNAME** | Canonical Name | **another name** (an alias) | `www → example.com` | Resolver re-queries the target. Cannot coexist with other records on the same name. **Cannot be used at the zone apex** (§8). |
| **ALIAS** | *(concept, not standard DNS)* | another name, **but resolved by the DNS provider** | `example.com → loadbalancer.aws` | A provider-specific feature (e.g. Route 53 Alias) that *acts like* a CNAME but works at the apex and returns A/AAAA directly. Covered in [02_route_53.md](02_route_53.md). |
| **MX** | Mail eXchange | a **mail server** name + priority | `10 mail.example.com` | Lower priority number = preferred. Points to a name, not an IP. |
| **TXT** | Text | arbitrary **text** | `"v=spf1 include:..."` | Used for domain verification, SPF/DKIM/DMARC email auth. |
| **NS** | Name Server | the **authoritative servers** for a zone | `ns1.example-dns.com` | This is what delegation (§4) is built on. |
| **SOA** | Start Of Authority | zone **metadata** | primary NS, admin email, serial, timers | Exactly one per zone; defines refresh/retry/expire and the default negative-cache TTL. |
| **PTR** | Pointer | an **IP back to a name** (reverse DNS) | `34.216.184.93.in-addr.arpa → example.com` | Reverse lookup; mail servers check it for anti-spam. |
| **SRV** | Service | a **service location** (host + port + priority + weight) | `10 5 5060 sip.example.com` | Used by protocols like SIP, LDAP, XMPP. |

💡 The exam loves the distinction between records that point to an **IP** (`A`, `AAAA`) and records that point to a **name** (`CNAME`, `MX`, `NS`). And it loves `TXT` for "verify you own the domain" / SPF-DKIM-DMARC email scenarios.

---

## 6. TTL and Caching

Every record carries a **TTL (Time To Live)** — a number of seconds telling resolvers "you may cache this answer for this long before asking me again."

```
  Authoritative says: www.example.com  A 93.184.216.34  TTL 300
                                                          │
        Recursive resolver caches it ────────────────────►│ for 300s
        During those 300s, every client asking gets the
        cached answer instantly — the authoritative server
        is NOT contacted again until the TTL expires.
```

The TTL is a fundamental trade-off:

| Lower TTL (e.g. 60s) | Higher TTL (e.g. 86400s / 1 day) |
|----------------------|----------------------------------|
| ✅ Changes (failover, new IP) propagate fast | ❌ Old IP can be cached for up to a day |
| ❌ More queries hit authoritative servers (cost/load) | ✅ Fewer queries, cheaper, more resilient |
| Used when records change often / for failover | Used for stable records |

> **Key insight**: When you change a DNS record, the world does **not** update instantly. Existing cached copies live until their TTL expires. This is why "DNS propagation" feels slow. **Plan ahead of a migration by lowering the TTL days in advance**, so caches are short by the time you cut over.

⚠️ There is also **negative caching** — a "this name doesn't exist" (NXDOMAIN) answer is cached too, governed by the `SOA` record's minimum TTL. A typo can therefore "stick" in caches for a while even after you fix it.

---

## 7. Registrar vs DNS Hosting (Two Different Jobs)

People conflate these constantly. They are separate roles and can be done by different companies:

| Role | What it does | Example |
|------|-------------|---------|
| **Domain registrar** | *Sells/registers* the domain name and records it in the TLD registry. This is who you pay yearly for `example.com`. | GoDaddy, Namecheap, Route 53 (registrar) |
| **DNS hosting (authoritative DNS)** | *Runs the name servers* that answer queries for your zone — i.e. holds your records. | Cloudflare DNS, Route 53 (hosted zone) |

The link between them: at the registrar, you set the **NS records** to point at whoever hosts your DNS. You can register at GoDaddy but host DNS at Route 53 — totally normal. AWS Route 53 happens to do *both* jobs (and more), which is why it can feel like one thing. It isn't.

---

## 8. CNAME vs A — and Why You Can't CNAME a Zone Apex

This is a classic exam trap, so understand the *why*.

- An **A record** maps a name directly to an IP. Final answer, one hop.
- A **CNAME** maps a name to *another name*. The resolver then has to look up that target. It's an alias/redirect at the DNS layer.

```
  A record:      api.example.com ──────────────► 93.184.216.34  (done)

  CNAME record:  www.example.com ──► example-app.herokudns.com ──► 93.184.216.34
                 (resolver re-queries the target name, then gets the IP)
```

**The apex problem.** The DNS standard (RFC 1034) says a `CNAME` cannot coexist with any other record on the same name. But your zone apex (`example.com`) **must** already have `SOA` and `NS` records — that's mandatory for the zone to exist. Therefore:

> **Rule**: You **cannot** put a `CNAME` at the zone apex (`example.com`). You *can* put a `CNAME` on a subdomain (`www.example.com`), because a subdomain has no mandatory `SOA`/`NS` to collide with.

❌ Wrong: `example.com  CNAME  myapp.elb.amazonaws.com` — illegal, conflicts with the apex `SOA`/`NS`.
✅ OK: `www.example.com  CNAME  myapp.elb.amazonaws.com` — fine, it's a subdomain.

So how do you point the *naked* `example.com` at a load balancer or CDN whose IP you don't control and that changes? You can't use a plain `A` record (no fixed IP) and you can't use `CNAME` (apex). This exact gap is what **Alias records** (a provider feature, e.g. Route 53 Alias) were invented to solve — see [02_route_53.md](02_route_53.md).

---

## Key Exam Points

- DNS resolves **names → IPs** (and other data) using a **distributed, cached hierarchy**; runs on **port 53** (UDP, TCP for large).
- Resolution order: **stub resolver → recursive resolver → root → TLD → authoritative**. Root/TLD give **referrals**, only the **authoritative** server gives the real answer.
- **A** = IPv4, **AAAA** = IPv6, **CNAME** = name→name, **MX** = mail, **NS** = delegation, **TXT** = verification/SPF, **SOA** = zone metadata, **PTR** = reverse, **SRV** = service+port.
- **TTL** controls caching: low TTL = fast propagation + more load; high TTL = fewer queries + slow propagation. **Lower TTL before a migration.**
- **Registrar ≠ DNS host.** The registrar's `NS` records point at whoever hosts the zone.
- **You cannot CNAME the zone apex** because it conflicts with the mandatory `SOA`/`NS` records. Subdomains can use CNAME. Apex → use an **Alias** (provider feature).

---

## Common Mistakes

- ❌ Thinking the recursive resolver is authoritative. It's a caching middleman; the authoritative server owns the truth.
- ❌ Expecting record changes to take effect instantly. Cached copies live until their TTL expires — including negative (NXDOMAIN) answers.
- ❌ Trying to put a `CNAME` on `example.com`. Illegal at the apex; this is a deliberate exam trap.
- ❌ Pointing a `CNAME` or `MX` at an IP address. Those record types point at **names**, not IPs.
- ❌ Confusing the registrar (who you pay for the name) with the DNS host (who answers queries). Moving DNS = changing `NS` records, not re-buying the domain.

---

**Next**: [02_route_53.md — Route 53: AWS DNS, Registrar & Health Checks](02_route_53.md)
