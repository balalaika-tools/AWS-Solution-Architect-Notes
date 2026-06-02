# Cloud Computing Fundamentals

> **Who this is for**: Engineers starting the SAA-C03 path with no cloud background. This file
> builds the vocabulary — service models, deployment models, the cost shift, and how AWS bills —
> that every later topic assumes. No networking knowledge required.

---

## 1️⃣ What "The Cloud" Actually Is

Cloud computing is the **on-demand delivery of IT resources over the internet with pay-as-you-go
pricing**. Instead of buying and racking your own servers, you rent compute, storage, databases,
and networking from a provider (AWS) and turn them on or off through an API or console in minutes.

The shift is from *owning infrastructure* to *consuming it as a service*:

```
TRADITIONAL (on-premises)            CLOUD (AWS)
┌──────────────────────────┐        ┌──────────────────────────┐
│ Buy servers (weeks)      │        │ Launch a server (minutes)│
│ Rack, power, cool them   │   →    │ AWS runs the data center │
│ Pay up front, fixed size │        │ Pay per hour/second used │
│ Overprovision "just in   │        │ Scale up/down on demand  │
│ case" → idle capacity    │        │ Match capacity to load   │
└──────────────────────────┘        └──────────────────────────┘
```

> **Key insight**: The cloud does not remove servers — it removes *your responsibility for owning
> and operating the hardware*. You trade capital expense and lead time for elasticity and an API.

---

## 2️⃣ The Three Service Models: IaaS, PaaS, SaaS

The service models differ by **how much the provider manages versus how much you manage**. The
classic "pizza as a service" framing: the more managed the model, the less you touch.

| Model | You manage | Provider manages | AWS examples |
|-------|-----------|------------------|--------------|
| **IaaS** (Infrastructure as a Service) | OS, runtime, apps, data, scaling config | Hardware, virtualization, network, facilities | **EC2** (virtual servers), **EBS** (disks), **VPC** (networking) |
| **PaaS** (Platform as a Service) | Your code and data | OS, patching, runtime, scaling, hardware | **Elastic Beanstalk**, **RDS** (managed database), **Lambda** |
| **SaaS** (Software as a Service) | Just your usage and data | Everything — it's a finished application | **Amazon WorkMail**, **Amazon Quick Sight / QuickSight**, **Chime** |

```
Control / responsibility
  ▲
  │  IaaS  ████████████░░░░  you manage more, more flexible
  │  PaaS  █████░░░░░░░░░░░  AWS manages the platform
  │  SaaS  █░░░░░░░░░░░░░░░  AWS manages nearly everything
  └──────────────────────────────────────────────────────►  abstraction
```

💡 On the exam, a scenario that says *"minimize operational/management overhead"* is steering you
toward the more managed option (PaaS or serverless) over raw IaaS (EC2 you patch yourself).

---

## 3️⃣ Deployment Models: Public, Private, Hybrid

Where the infrastructure lives and who can use it:

| Model | Definition | Typical reason | AWS angle |
|-------|-----------|----------------|-----------|
| **Public cloud** | Resources owned and run by a provider, shared across many tenants over the internet | Speed, elasticity, no capital cost | Standard AWS usage |
| **Private cloud** | Infrastructure dedicated to a single organization (on-prem or hosted) | Strict compliance, legacy systems, full control | **AWS Outposts** brings AWS hardware into your data center |
| **Hybrid cloud** | A mix — some workloads on-prem, some in the public cloud, connected together | Gradual migration, data residency, burst capacity | VPN / Direct Connect linking on-prem to a VPC |

⚠️ "Hybrid" specifically means **on-premises + cloud connected together**, not "two AWS Regions."
Don't confuse multi-Region (still all public cloud) with hybrid.

> **Rule**: Compliance or "data must stay in our building" + still wants the AWS API → **Outposts**
> (private/hybrid). "Connect our existing data center to AWS" → **hybrid** via VPN or Direct Connect.

---

## 4️⃣ CapEx vs OpEx — The Cost Model Shift

This is the financial heart of the cloud value proposition and a recurring exam theme.

| | **CapEx** (Capital Expenditure) | **OpEx** (Operational Expenditure) |
|---|---|---|
| What it is | Large up-front purchase of assets you own | Ongoing pay-for-what-you-use spending |
| Example | Buying 50 servers before launch | Paying for EC2 hours actually consumed |
| Timing | Pay before you know real demand | Pay after, matched to actual usage |
| Risk | Over- or under-provisioning | Cost scales with usage (must be watched) |
| Cloud fit | Traditional data centers | The default AWS model |

> **Key insight**: The cloud **converts large CapEx into variable OpEx**. You stop guessing
> capacity months in advance and instead pay for what you actually use, when you use it.

---

## 5️⃣ The Six Advantages of Cloud (AWS Canon)

AWS frames the value of cloud as six benefits. Memorize these — they appear almost verbatim in
"why move to the cloud?" questions.

| # | Advantage | Plain-English meaning |
|---|-----------|-----------------------|
| 1 | **Trade CapEx for variable expense** | Pay only for what you consume instead of buying hardware up front |
| 2 | **Benefit from massive economies of scale** | AWS's huge aggregate usage drives lower pay-as-you-go prices than you could achieve alone |
| 3 | **Stop guessing capacity** | Scale up or down on demand; no over-provisioning "just in case" |
| 4 | **Increase speed and agility** | New resources are minutes away, so experimentation is cheap and fast |
| 5 | **Stop spending on running data centers** | Focus on your customers, not racking servers and managing power/cooling |
| 6 | **Go global in minutes** | Deploy to Regions worldwide to put apps close to users with low latency |

💡 Mnemonic: **C**apEx→variable, **E**conomies of scale, **C**apacity guessing gone, **A**gility,
no **D**ata-center ops, **G**lobal in minutes.

---

## 6️⃣ How AWS Bills You

AWS pricing follows three core principles, but the day-one mental model is simple: **you pay for
compute, storage, and data transfer (out), and most foundational services have a free tier.**

**The three pricing fundamentals:**

1. **Compute** — pay for processing time (e.g., EC2 per-second/per-hour, Lambda per request + duration).
2. **Storage** — pay for data stored (e.g., S3 per GB-month, EBS per provisioned GB-month).
3. **Data transfer** — **inbound is generally free; outbound to the internet is charged**, and
   so is transfer between Regions. Traffic within the same AZ is usually free.

```
        ┌─────────────┐
  IN ──►│             │ (data IN to AWS is typically FREE)
        │     AWS     │
 OUT ◄──│   Region    │ (data OUT to internet / cross-Region is CHARGED $$$)
        └─────────────┘
```

⚠️ **Data transfer out is the cost most beginners forget.** A "free" S3 bucket can generate a real
bill if a lot of data is downloaded from it. Same-AZ traffic free; cross-AZ and egress cost money.

✅ **AWS Free Tier** comes in three flavors: *always free* (e.g., 1M Lambda requests/month),
*12-months free* (e.g., 750 EC2 t2/t3.micro hours/month for the first year), and *short-term
trials*. Great for studying without spending.

> **Pay-as-you-go, no long-term commitment** is the default — but committing to usage (Reserved
> Instances, Savings Plans) trades flexibility for big discounts. Covered later in compute/cost.

---

## Key Exam Points

- **IaaS = EC2** (you patch the OS); **PaaS = RDS/Beanstalk/Lambda** (AWS manages the platform);
  **SaaS = a finished app** (WorkMail, Amazon Quick Sight / QuickSight).
- "Minimize operational overhead / undifferentiated heavy lifting" → choose the **more managed**
  (PaaS / serverless) option.
- The cloud **converts CapEx into OpEx** — variable, pay-as-you-go spending.
- **Hybrid = on-premises + cloud connected**; **Outposts = AWS hardware in your data center**
  (private/hybrid use case, keeps the AWS API).
- Know the **six advantages** — questions paraphrase them ("go global in minutes," "stop guessing
  capacity," "trade CapEx for variable expense").
- Billing: **inbound data transfer is free, outbound (egress) and cross-Region are charged.**

---

## Common Mistakes

- ❌ Calling EC2 "PaaS." EC2 is **IaaS** — you still own the OS and patching. Beanstalk/Lambda are the platform layer.
- ❌ Thinking multi-Region or multi-AZ counts as "hybrid." Hybrid requires an **on-premises** component.
- ❌ Assuming everything in the Free Tier is free forever — much of it is **12 months only** or capped.
- ❌ Forgetting **egress/data-transfer-out charges** when estimating cost; they often dominate a bill.
- ❌ Equating "private cloud" with "a VPC." A VPC is your private *network* inside the public cloud, not a private cloud deployment model.

---

**Next**: [02_aws_global_infrastructure.md — Regions, Availability Zones & the Edge](02_aws_global_infrastructure.md)
