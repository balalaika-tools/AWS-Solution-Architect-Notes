# Security Groups vs NACLs — Worked Rule Examples

> **Who this is for**: Engineers who read [Security Groups vs NACLs](../03_networking/04_security_groups_vs_nacls.md)
> conceptually and want to see the *actual rule tables* side by side — including the ephemeral-port
> return rule a NACL needs but an SG doesn't, and the classic "NACL denies but SG allows" trap.
> Uses the [reference VPC](01_vpc_public_private_subnets.md) from file 01.

---

## 1. The One Difference That Drives Everything

| | Security Group | Network ACL |
|--|----------------|-------------|
| Level | **Instance / ENI** | **Subnet** |
| State | **Stateful** | **Stateless** |
| Rules | **Allow only** (implicit deny) | **Allow *and* Deny** |
| Evaluation | All rules; allow if any match | **Lowest rule number first**, stop at first match |
| Return traffic | **Automatic** (stateful) | **Must be allowed explicitly** (ephemeral ports) |
| Default | Deny all inbound, allow all outbound | Default NACL: allow all; custom NACL: deny all |

> **Key insight**: "Stateful" means the SG *remembers* the outbound request and automatically lets the matching reply back in (and vice-versa). "Stateless" means the NACL evaluates every packet on its own with no memory — so you must write a rule for the **return** direction yourself, on the **ephemeral ports**.

---

## 2. Worked Example A — Stateful SG (no return rule needed)

Goal: a web server accepts HTTPS from the internet.

**Web server security group:**

| Type | Protocol | Port | Source | Notes |
|------|----------|------|--------|-------|
| Inbound | TCP | 443 | `0.0.0.0/0` | Allow HTTPS in |
| Outbound | — | — | — | **Nothing added for the reply** |

That's it. A client connects from `203.0.113.5:51000` → server `:443`. The server's reply goes back to `203.0.113.5:51000`. Even though `51000` is an ephemeral port and you wrote **no outbound rule** for it, the SG allows the reply because it is **stateful** — it tracks the inbound flow and permits its return automatically.

✅ With SGs you reason in terms of "what connections do I want to accept / initiate", never "what about the reply".

⚠️ Don't be tricked by the exam's reminder that SGs have a default *allow-all outbound* rule. Even if you removed it, the **reply to an allowed inbound request still flows** — stateful tracking is separate from the explicit outbound rules. Explicit outbound rules govern connections the instance *initiates*.

---

## 3. Worked Example B — Stateless NACL (return rule REQUIRED)

Same goal — accept HTTPS — but now expressed at the subnet's NACL. Because the NACL is stateless, you need **two** rules: one for the inbound request, one for the **outbound reply on ephemeral ports**.

**Inbound NACL rules:**

| Rule # | Type | Protocol | Port range | Source | Allow/Deny |
|--------|------|----------|-----------|--------|------------|
| 100 | HTTPS | TCP | 443 | `0.0.0.0/0` | ALLOW — the request |
| * | All | All | All | All | DENY (implicit final) |

**Outbound NACL rules:**

| Rule # | Type | Protocol | Port range | Dest | Allow/Deny |
|--------|------|----------|-----------|------|------------|
| 100 | Custom TCP | TCP | **1024–65535** | `0.0.0.0/0` | ALLOW — **the reply** |
| * | All | All | All | All | DENY (implicit final) |

> **Rule**: The reply to an inbound `:443` request goes back out to the client's **ephemeral port** (Linux: `32768–60999`; AWS uses **`1024–65535`** as the safe catch-all). Without that outbound ephemeral ALLOW, the request arrives but the **response is dropped** — the connection appears to hang.

❌ The single most common NACL mistake: allowing inbound 443 but forgetting the outbound `1024–65535` rule. Symptom: connections time out even though "the port is open".

💡 If outbound traffic the instance *initiates* (e.g. an API call to `:443`) is also needed, you'd add outbound `443 ALLOW` **and** inbound `1024–65535 ALLOW` for *its* replies. Stateless means every direction of every flow is your responsibility.

---

## 4. Ordered NACL Evaluation + "Deny but Allow" Interaction

NACL rules are evaluated **in ascending rule-number order**, and the **first match wins** — no further rules are checked. Deny rules are possible (SGs can't deny), so a low-numbered deny overrides a higher-numbered allow.

**Inbound NACL — block one bad actor, allow everyone else on 443:**

| Rule # | Type | Port | Source | Allow/Deny | Effect |
|--------|------|------|--------|------------|--------|
| 90  | All | All | `198.51.100.7/32` | **DENY** | Evaluated first — this IP is blocked outright |
| 100 | TCP | 443 | `0.0.0.0/0` | ALLOW | Everyone else gets HTTPS |
| *   | All | All | All | DENY | Implicit final deny |

A packet from `198.51.100.7` matches rule **90** (deny) and is dropped — rule 100 is never reached. A packet from any other IP skips 90 (no match) and matches 100 (allow).

⚠️ Ordering matters: if you numbered the deny as **110** instead of **90**, the allow at 100 would match *first* and the bad IP would get through. **Put deny rules at lower numbers than the broad allow.**

### The classic "NACL deny but SG allow" trap

A packet must pass **both** layers. They are evaluated in this order for inbound:

```
Internet ──► [ Subnet NACL ] ──► [ Instance SG ] ──► Instance
                  │                    │
            stateless,           stateful,
            allow+deny           allow-only
```

> **Rule**: Traffic is allowed **only if the NACL allows it AND the SG allows it**. If the NACL denies, the SG never even sees the packet. A NACL deny *overrides* an SG allow.

| NACL says | SG says | Result |
|-----------|---------|--------|
| ALLOW | ALLOW | ✅ Traffic passes |
| **DENY** | ALLOW | ❌ Blocked at the NACL — SG allow is irrelevant |
| ALLOW | DENY (no matching allow) | ❌ Blocked at the SG |
| DENY | DENY | ❌ Blocked |

This is why "but my security group allows it!" debugging fails — a subnet NACL denying the port (or missing the ephemeral return rule) blocks traffic regardless of the SG.

---

## 5. Build It (CLI)

```bash
# --- Security Group: allow inbound 443, no return rule needed (stateful) ---
aws ec2 authorize-security-group-ingress \
  --group-id sg-0web \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# --- NACL: inbound 443 allow ---
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0web --rule-number 100 --ingress \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# --- NACL: outbound ephemeral allow (THE REPLY) — required, stateless ---
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0web --rule-number 100 --egress \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# --- NACL: block one IP, low rule number so it wins ---
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0web --rule-number 90 --ingress \
  --protocol -1 --cidr-block 198.51.100.7/32 --rule-action deny
```

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Connection hangs / times out, SG looks correct | NACL missing **outbound ephemeral** (`1024–65535`) allow for replies | Add the outbound ephemeral ALLOW rule |
| A blocked IP still gets through the NACL | Deny rule numbered **higher** than the broad allow | Renumber the deny **below** the allow |
| "SG allows it but traffic is blocked" | Subnet **NACL** denies the port or its return | Check the NACL on that subnet — deny overrides SG allow |
| Outbound API call from instance fails (NACL) | Inbound ephemeral allow missing for the call's **reply** | Add inbound `1024–65535` ALLOW |
| Custom NACL blocks everything | Custom NACLs **deny all by default**; you added only some allows | Add the rules you need; nothing is implied |
| SG change "doesn't take" for a long time | Expecting NACL-style ordering | SGs have no order; all rules apply at once — re-check the rule values |

---

## Key Exam Points

- **SG = stateful, instance-level, allow-only.** Return traffic is automatic — no ephemeral rule needed.
- **NACL = stateless, subnet-level, allow + deny.** You **must** add the outbound (or inbound) **ephemeral `1024–65535`** rule for return traffic.
- NACLs evaluate **lowest rule number first, first match wins.** Put **deny rules at low numbers** to override broader allows.
- A packet must pass **both** NACL and SG. A **NACL deny overrides an SG allow** — and is checked first.
- Default NACL **allows all**; a custom NACL **denies all** until you add rules.
- SGs cannot express **deny**; if a question needs to block a specific IP, the answer is a **NACL**.
- The #1 NACL bug: forgetting the **outbound ephemeral return rule** → connections hang.

---

**Next**: [05_vpc_peering.md — Connecting Two VPCs with VPC Peering](05_vpc_peering.md)
