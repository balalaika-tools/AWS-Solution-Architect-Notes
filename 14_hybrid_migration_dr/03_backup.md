# AWS Backup: Centralized Data Protection

> **Who this is for**: An engineer who has used EBS or RDS snapshots ad-hoc but never run a
> centralized, policy-driven backup program — and isn't sure how backup differs from
> replication. Builds on [02_migration_services.md](02_migration_services.md). The RPO idea
> here feeds directly into [04_disaster_recovery.md](04_disaster_recovery.md).

---

## 1. Prerequisite: Backup vs Snapshot vs Replication

These three words get used loosely, but they protect against **different failures**. Knowing
the distinction is half the battle on the exam.

| Concept | What it is | Protects against | Point-in-time? |
|---------|-----------|------------------|----------------|
| **Snapshot** | A point-in-time copy of a volume/disk (e.g., EBS snapshot to S3) | Data loss, corruption, accidental delete | ✅ Yes — frozen at time *T* |
| **Backup** | A managed, retained, policy-governed *set* of snapshots/copies | Same as snapshot, but with **retention, lifecycle, and recovery management** | ✅ Yes |
| **Replication** | A **continuous, near-real-time** copy to another target (read replica, S3 CRR, Aurora replica) | Hardware/AZ/Region failure, read scaling | ❌ No — it copies the *current* state, including mistakes |

> **Key insight**: **Replication is not a backup.** If you accidentally `DELETE FROM users`,
> replication faithfully deletes the rows on the replica too. Only a **point-in-time
> backup/snapshot** lets you go back to *before* the mistake. The exam loves this trap.

```
 Backup / Snapshot          Replication
   T1   T2   T3              source ──live──▶ replica
    •    •    •                       (mirrors everything,
  (frozen restore points)              including mistakes)
```

---

## 2. Prerequisite: RPO — How Much Data Can You Lose?

**Recovery Point Objective (RPO)** = the **maximum amount of data, measured in time, you can
afford to lose**. It answers: *"If we restore from our last backup, how far back is it?"*

- Backups **every 24 h** → RPO = 24 h → you could lose up to a day of changes.
- Backups **every 1 h** → RPO = 1 h.
- Continuous replication → RPO ≈ seconds.

```
 last backup                  failure
     │◀────── RPO window ──────▶│
     │   (data created here     │
     │    is lost on restore)   │
─────●──────────────────────────✗──────▶ time
```

> **Rule**: **Shorter RPO costs more** (more frequent backups, more storage, replication).
> RPO drives *backup frequency*. (Its sibling, **RTO** — how fast you recover — is covered in
> [04_disaster_recovery.md](04_disaster_recovery.md).)

---

## 3. AWS Backup — One Policy Across Many Services

Manually scripting EBS snapshots here, RDS snapshots there, and EFS backups somewhere else
doesn't scale and is easy to get wrong. **AWS Backup** is a **centralized, policy-based**
service that manages backups for many AWS services from one place.

Supported services (exam-relevant set):

- **EBS** volumes
- **EFS** file systems
- **RDS** and **Aurora**
- **DynamoDB**
- **FSx** file systems
- **Storage Gateway** volumes
- (and EC2 instances, S3, and more)

```
                ┌──────────────────────┐
   Backup plan  │      AWS Backup      │
   (schedule +  │  policy engine       │
   retention)   └─────────┬────────────┘
                          │ runs jobs against
        ┌───────┬─────────┼─────────┬─────────┐
        ▼       ▼         ▼         ▼         ▼
      EBS     RDS       EFS     DynamoDB    FSx
        └──────────── stored in ───────────┘
                          ▼
                   Backup Vault (encrypted)
```

---

## 4. Core Building Blocks

| Concept | What it does |
|---------|--------------|
| **Backup plan** | A policy: **schedule** (e.g., daily 02:00), **retention** (keep 35 days), lifecycle (move to cold storage after N days), and copy rules (cross-Region/account). |
| **Backup vault** | The encrypted container where recovery points are stored; KMS-encrypted; access controlled by resource policy. |
| **Resource assignment** | Which resources the plan applies to — by **resource ID** or, better, by **tag**. |
| **Recovery point** | An individual backup (a point-in-time copy you can restore). |

---

## 5. Tag-Based Backup Selection

Instead of naming each resource, you assign resources to a backup plan **by tag** (e.g.,
`Backup = daily`). Any new resource that gets that tag is **automatically** included — no
plan edits needed.

✅ Tag a new RDS instance `Backup=daily` and AWS Backup starts protecting it on the next run.

💡 Tag-based selection is the scalable, exam-preferred answer for *"ensure all production
resources are backed up consistently."*

---

## 6. Vault Lock — WORM / Compliance

**AWS Backup Vault Lock** enforces **WORM (Write-Once-Read-Many)** on a vault: once locked,
**backups cannot be deleted or their retention shortened** — not even by the root user — until
the retention period expires.

| Mode | Behavior |
|------|----------|
| **Governance mode** | Users with special IAM permissions can still alter/delete. |
| **Compliance mode** | Immutable. After the cooling-off (grace) period, **no one** can delete or shorten retention — full stop. |

✅ Use Compliance-mode Vault Lock for regulatory requirements (financial, healthcare) and as
**protection against ransomware or a malicious admin** deleting backups.

⚠️ Compliance mode is irreversible after the grace period. You cannot "undo" it, so test in
governance mode first.

---

## 7. Cross-Region & Cross-Account Backup

A backup that sits only in the **same Region and same account** as the source is exposed to a
Region-wide event or a compromised account.

- **Cross-Region copy** — replicate recovery points to a second Region. Foundation of the
  **Backup & Restore** DR strategy (file 04).
- **Cross-account copy** — copy to a separate, isolated AWS account (often via AWS
  Organizations). Protects against account compromise / accidental deletion in the source
  account.

```
 Source account / Region A          Isolated account / Region B
   Backup vault  ──cross-Region copy──▶  Backup vault
                 ──cross-account copy──▶  (blast-radius isolation)
```

---

## 8. Backup vs DR — Where the Line Is

Backup gives you **durable, restorable copies** of data. It is the **cheapest, slowest** form
of disaster recovery — you still have to **provision infrastructure and restore** before the
app runs again.

> **Mental model**: Backup answers *"can I get the data back?"* (RPO). DR answers *"how fast is
> the whole application running again?"* (RTO). A solid backup program is the **floor** of DR;
> faster recovery (pilot light, warm standby) layers infrastructure on top — see file 04.

---

## 9. Snapshot Recap: EBS & RDS

You'll still see direct snapshots, with or without AWS Backup:

- **EBS snapshots** — incremental, stored in S3 (AWS-managed), point-in-time. Can be copied
  cross-Region and shared. Automate with Data Lifecycle Manager or AWS Backup.
- **RDS snapshots** — **automated** (enabled by default, retention up to 35 days, support
  point-in-time restore) vs **manual** (kept until you delete them). Copyable cross-Region.

💡 When deleting a supported RDS DB instance, you can **retain automated backups** for their
configured retention period, and you can take a final manual snapshot. Manual snapshots persist
until explicitly deleted. Make the retention/final-snapshot choice part of the deletion approval;
do not assume the console default is the recovery policy.

---

## 10. Organization-Wide Backup Governance

At scale, backups should be controlled outside the workload account that might be compromised.
With AWS Organizations, designate a **backup administrator account**, enable trusted access and
apply **backup policies** through the organization hierarchy. Policies can combine inherited
settings, so test the effective policy at each OU before assuming a resource is protected.

A common separation-of-duties design is:

```
management account: organization policy and delegated administration
        │
        ├── workload accounts: source resources and local operational restore
        │          └── scheduled copy
        ▼
backup account: isolated vault + restricted operators + monitoring
        └── cross-Region copy: regional recovery and sovereignty requirement
```

- Use tag/resource assignments plus Organizations backup policies for coverage, but use AWS
  Config, Backup Audit Manager or equivalent controls to detect unprotected resources and failed
  jobs. A policy attachment is intent, not evidence of a recoverable backup.
- Place critical copies in a separate account with a different administrator path. Scope vault
  access policies, service roles and KMS key policies so the copy job and restore team have only
  the permissions they need.
- A **logically air-gapped vault** adds stronger isolation and compliance-mode Vault Lock. Sharing
  through AWS RAM can support controlled recovery by another account without making the workload
  account the backup owner. Confirm service and Region support for the protected resource.
- Use legal holds when recovery points must be retained for an investigation independently of
  normal lifecycle rules. Treat removal authority as a high-risk compliance permission.
- For cross-account encrypted backups, verify both source and destination KMS policy/grants and
  use customer-managed keys when the service requires them. Test restores with the recovery role;
  a successful copy does not prove that the destination can decrypt and rebuild the resource.

### Ransomware recovery is more than immutability

Vault Lock prevents deletion or shortened retention, but an immutable backup can still contain
malware or corrupted data. Maintain recovery points from before the dwell time, isolate the
recovery environment, scan restored hosts/data with appropriate security tooling, rotate exposed
credentials and rebuild infrastructure from trusted definitions. Reconnect to production only
after data-integrity and application validation.

Run scheduled restore tests into isolated accounts/VPCs. Measure discovery, authorization,
provisioning, data restore and application validation separately. Record actual recovery time and
the newest usable recovery point against RTO/RPO; “backup job completed” is not a DR test.

---

## Key Exam Points

- **Replication ≠ backup.** Replication propagates mistakes; only point-in-time
  backups/snapshots let you roll back.
- **RPO = how much data you can lose** → drives backup frequency; shorter RPO = higher cost.
- **AWS Backup** centralizes backups across **EBS, EFS, RDS/Aurora, DynamoDB, FSx, Storage
  Gateway**, using **backup plans** + **vaults**.
- **Tag-based selection** auto-includes new resources — the scalable answer.
- **Vault Lock (Compliance mode) = immutable WORM**, even against root — for compliance and
  ransomware protection.
- **Cross-Region** and **cross-account** copies isolate blast radius and enable DR.
- **Manual RDS snapshots persist** until deleted. Supported RDS instances can retain automated
  backups for their remaining retention period when the instance is deleted.
- Organization policies, isolated accounts and locked vaults reduce administrative blast radius,
  but only repeated restore tests demonstrate recoverability.

---

## Common Mistakes

- ❌ Treating a **read replica** or **S3 cross-Region replication** as a backup — it isn't, it
  mirrors deletions.
- ❌ Leaving backups in the **same Region/account** as the source and calling it DR.
- ❌ Believing AWS Backup alone gives a fast recovery — it gives RPO, not low RTO.
- ❌ Forgetting that **Compliance-mode Vault Lock is irreversible** after the grace period.
- ❌ Deleting an RDS instance without explicitly choosing retained automated backups and/or a
  final manual snapshot that meets the recovery and retention requirement.

---

**Next**: [04_disaster_recovery.md — Disaster Recovery Strategies](04_disaster_recovery.md)
