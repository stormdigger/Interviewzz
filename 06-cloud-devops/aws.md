# ☁️ AWS

> AWS has 200+ services. You need about 20. This book covers those, the architecture patterns that combine them, and the cost traps that surprise people.

**Prerequisite:** [Linux & Networking](linux-networking.md)

---

## 📑 Contents

1. [The Mental Model](#1-the-mental-model)
2. [The Global Infrastructure](#2-the-global-infrastructure)
3. [IAM](#3-iam)
4. [Networking — VPC](#4-networking--vpc)
5. [Compute](#5-compute)
6. [Storage](#6-storage)
7. [Databases](#7-databases)
8. [Messaging & Events](#8-messaging--events)
9. [Content Delivery & DNS](#9-content-delivery--dns)
10. [Observability](#10-observability)
11. [Architecture Patterns](#11-architecture-patterns)
12. [Cost Engineering](#12-cost-engineering)
13. [The Well-Architected Framework](#13-the-well-architected-framework)
14. [Interview Section](#14-interview-section)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. The Mental Model

#### 💬 The organizing idea

```
   ⭐ AWS SELLS PRIMITIVES, NOT SOLUTIONS.

   Every service is a building block with a well-defined
   contract. Architecture is composition. There is no "AWS way"
   to build an app — there are twelve ways, with different
   cost, operational, and scaling profiles.

   THE THREE QUESTIONS FOR EVERY SERVICE CHOICE:

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. WHO OPERATES IT?                                          │
   │    Self-managed on EC2 → you patch, scale, back up           │
   │    Managed (RDS, MSK)  → AWS patches, you tune               │
   │    Serverless (Lambda) → AWS does everything, you write code │
   │    ⭐ Moving right trades control and cost-at-scale for       │
   │      operational burden.                                     │
   ├──────────────────────────────────────────────────────────────┤
   │ 2. WHAT'S THE FAILURE DOMAIN?                                │
   │    AZ-scoped? Region-scoped? Global?                         │
   │    ⭐ This determines your real availability, not the SLA     │
   │      number on the marketing page.                           │
   ├──────────────────────────────────────────────────────────────┤
   │ 3. HOW IS IT PRICED?                                         │
   │    Per hour? Per request? Per GB stored? Per GB TRANSFERRED? │
   │    ⭐ Data transfer is the cost that surprises everyone.      │
   └──────────────────────────────────────────────────────────────┘
```

### The shared responsibility model

```
   ┌──────────────────────────────────────────────────────────────┐
   │  YOU are responsible for security IN the cloud               │
   │    • Your data, and its encryption                           │
   │    • IAM policies and who can do what                        │
   │    • OS patching (EC2) · application code · dependencies     │
   │    • Network configuration: security groups, NACLs           │
   │    • ⭐ S3 bucket policies — the #1 source of AWS breaches    │
   ├──────────────────────────────────────────────────────────────┤
   │  AWS is responsible for security OF the cloud                │
   │    • Physical datacenters · hypervisor · network fabric      │
   │    • Managed service patching (RDS engine, Lambda runtime)   │
   └──────────────────────────────────────────────────────────────┘

   ⭐ The boundary MOVES depending on the service. On EC2 you
     patch the OS; on Lambda you don't even see one. Knowing
     where the line sits for each service is the whole point
     of the model.
```

---

## 2. The Global Infrastructure

```
   ┌──────────────────────────────────────────────────────────────┐
   │  REGION  (us-east-1, eu-west-1, ap-south-1)                  │
   │  ⭐ Geographically separate. Data does NOT leave a region     │
   │    unless you explicitly move it.                            │
   │                                                              │
   │  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
   │  │    AZ-a    │  │    AZ-b    │  │    AZ-c    │              │
   │  │            │  │            │  │            │              │
   │  │ ⭐ separate │  │            │  │            │              │
   │  │ buildings, │  │            │  │            │              │
   │  │ power, and │  │            │  │            │              │
   │  │ cooling    │  │            │  │            │              │
   │  └────────────┘  └────────────┘  └────────────┘              │
   │        └──── low-latency private links (<2ms) ────┘          │
   └──────────────────────────────────────────────────────────────┘

           EDGE LOCATIONS (400+) — CloudFront, Route 53, WAF
```

```
   ⭐ THE AVAILABILITY LADDER — know what each level buys you

   Single AZ         ⚠️ an AZ failure takes you down entirely
   Multi-AZ          ⭐ the baseline for production. Survives a
                     datacenter loss. Usually cheap to add.
   Multi-Region      Survives a regional failure. ⚠️ Expensive
                     and genuinely complex — cross-region data
                     consistency is a hard problem.
   Multi-Cloud       ⚠️ Usually a mistake. You get the lowest
                     common denominator of both platforms and
                     double the operational surface.
```

```
   ⚠️ us-east-1 IS SPECIAL AND IT MATTERS

   • Several global services have their control plane there:
     IAM, CloudFront, Route 53, ACM certificates for CloudFront
   • It's the oldest and largest region, and historically the
     one with the most incidents
   • ⭐ A us-east-1 outage can affect global service control
     planes even if your workload runs elsewhere
```

---

## 3. IAM

#### 💬 The evaluation logic — get this right and IAM stops being mysterious

```
   ⭐ HOW A REQUEST IS AUTHORIZED

   ┌──────────────────────────────────────────────────────────────┐
   │  1. Is there an EXPLICIT DENY anywhere?                      │
   │       (identity policy, resource policy, SCP, permissions    │
   │        boundary, session policy)                             │
   │     → YES: ⭐ DENIED. Nothing can override an explicit deny.  │
   │                                                              │
   │  2. Is there an explicit ALLOW?                              │
   │     → NO: DENIED (⭐ default deny — everything is forbidden   │
   │            unless granted)                                   │
   │                                                              │
   │  3. Do ALL applicable boundaries also allow it?              │
   │       SCPs · permissions boundaries · session policies       │
   │     → NO: DENIED                                             │
   │                                                              │
   │  4. → ALLOWED                                                │
   └──────────────────────────────────────────────────────────────┘

   ⭐ TWO RULES EXPLAIN ALMOST EVERY IAM PUZZLE:
     • Explicit deny always wins
     • Default is deny — an allow must exist somewhere
```

### The core entities

```
   USER      A long-lived human or app identity with credentials
             ⚠️ Access keys are the most commonly leaked AWS
               credential. Avoid creating them.

   ROLE  ⭐   An identity with no permanent credentials that is
             ASSUMED temporarily. This is the correct primitive
             for almost everything:
               • EC2 instance profiles
               • Lambda execution roles
               • EKS pods via IRSA
               • Cross-account access
               • Federated human access via SSO

   POLICY    A JSON document granting or denying actions
             Identity-based → attached to a user/group/role
             Resource-based → attached to the resource (S3
               bucket policy, KMS key policy) ⭐ these can grant
               cross-account access without the other side
               having a role

   SCP       Service Control Policy — an Organizations-level
             GUARDRAIL. ⭐ It does not grant anything; it caps
             what any principal in the account can be granted.
```

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/uploads/${aws:username}/*",
    "Condition": {
      "Bool": { "aws:SecureTransport": "true" },
      "StringEquals": { "s3:x-amz-server-side-encryption": "aws:kms" }
    }
  }]
}
```

```
   ⭐ IAM BEST PRACTICES, IN ORDER OF IMPACT

   1. ⭐ NO LONG-LIVED ACCESS KEYS. Use roles, IAM Identity
      Center for humans, and IRSA/workload identity for pods.
      A leaked key in a git repo is the single most common
      AWS compromise.
   2. Least privilege — start from nothing and add. Use Access
      Analyzer to generate policies from actual CloudTrail usage.
   3. MFA on all human access, enforced via SCP.
   4. ⭐ Separate ACCOUNTS as the strongest blast-radius
      boundary — prod, staging, dev, security, logging.
      Account boundaries are far stronger than IAM boundaries.
   5. SCPs to make dangerous actions impossible org-wide
      (e.g. deny disabling CloudTrail, deny leaving the org).
   6. Permissions boundaries so teams can create roles without
      escalating beyond a cap.
```

---

## 4. Networking — VPC

```
   ┌──────── VPC 10.0.0.0/16 ─────────────────────────────────────┐
   │                                                              │
   │  ┌─── AZ-a ──────────────┐   ┌─── AZ-b ──────────────┐       │
   │  │                       │   │                       │       │
   │  │ PUBLIC 10.0.1.0/24    │   │ PUBLIC 10.0.2.0/24    │       │
   │  │  ┌─────┐ ┌────────┐   │   │  ┌────────┐           │       │
   │  │  │ ALB │ │NAT GW  │   │   │  │NAT GW  │           │       │
   │  │  └──┬──┘ └───┬────┘   │   │  └───┬────┘           │       │
   │  │     │        │        │   │      │                │       │
   │  │ PRIVATE 10.0.11.0/24  │   │ PRIVATE 10.0.12.0/24  │       │
   │  │  ┌─────────┐          │   │  ┌─────────┐          │       │
   │  │  │ App/EKS │──────────┼───┼──│ App/EKS │          │       │
   │  │  └─────────┘          │   │  └─────────┘          │       │
   │  │                       │   │                       │       │
   │  │ DATA 10.0.21.0/24     │   │ DATA 10.0.22.0/24     │       │
   │  │  ┌─────────┐          │   │  ┌─────────┐          │       │
   │  │  │ RDS     │◀─────────┼───┼─▶│ RDS     │ standby  │       │
   │  │  └─────────┘          │   │  └─────────┘          │       │
   │  └───────────────────────┘   └───────────────────────┘       │
   │                                                              │
   │  ┌────────────────┐                                          │
   │  │Internet Gateway│  ⭐ a subnet is PUBLIC only because its   │
   │  └────────────────┘    route table points 0.0.0.0/0 at an IGW│
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHAT MAKES A SUBNET "PUBLIC"
     Not a flag. A subnet is public because its ROUTE TABLE has
     a route for 0.0.0.0/0 pointing at an Internet Gateway.
     That's the entire distinction.

   PRIVATE subnets route 0.0.0.0/0 to a NAT Gateway instead —
   outbound internet works, inbound does not.
```

### Security Groups vs NACLs

```
   ┌──────────────────────┬───────────────────────────────────────┐
   │ SECURITY GROUP       │ NETWORK ACL                           │
   ├──────────────────────┼───────────────────────────────────────┤
   │ Attached to an ENI   │ Attached to a SUBNET                  │
   │ ⭐ STATEFUL — return  │ ⚠️ STATELESS — you must allow return   │
   │   traffic is         │   traffic explicitly (ephemeral       │
   │   automatically      │   ports 1024-65535)                   │
   │   allowed            │                                       │
   │ ALLOW rules only     │ ALLOW and DENY rules                  │
   │ All rules evaluated  │ ⭐ Numbered, evaluated in ORDER,       │
   │                      │   first match wins                    │
   │ ⭐ Can reference      │ CIDR blocks only                      │
   │   another SG as the  │                                       │
   │   source             │                                       │
   ├──────────────────────┼───────────────────────────────────────┤
   │ ⭐ Your primary tool  │ Rarely needed. Use for coarse         │
   │                      │ subnet-level deny rules.              │
   └──────────────────────┴───────────────────────────────────────┘

   ⭐ SG-REFERENCING IS THE PATTERN TO USE
     Instead of "allow 10.0.11.0/24 on port 5432", write
     "allow sg-app on port 5432". Now the rule follows the
     application wherever it runs, and there are no CIDR
     blocks to maintain.
```

```
   ⚠️⚠️ NAT GATEWAY IS A TOP-3 SURPRISE COST

   • ~$0.045/hour per NAT GW  → ~$32/month each
   • ⭐ PLUS ~$0.045 per GB PROCESSED
   • You need one per AZ for high availability
   • Three AZs = ~$100/month before a single byte moves

   ⚠️ THE TRAP: pulling container images, OS packages, or
     talking to S3 from private subnets all flows through NAT
     and is billed per GB. Teams routinely see four-figure
     monthly NAT bills for traffic that never leaves AWS.

   ⭐ THE FIX: VPC ENDPOINTS
     Gateway endpoints (S3, DynamoDB) are FREE and route
     traffic over the AWS backbone instead of NAT.
     Interface endpoints (most other services) cost ~$7/month
     each plus data, but are still far cheaper than NAT for
     any meaningful volume.
     ⭐ Adding an S3 gateway endpoint is often the single
       highest-ROI change in an AWS account.
```

```
   CONNECTIVITY OPTIONS

   VPC PEERING         1:1, non-transitive. ⚠️ Doesn't scale —
                       N VPCs need N(N-1)/2 peerings.
   TRANSIT GATEWAY ⭐   Hub-and-spoke. The right answer for more
                       than a handful of VPCs.
   PRIVATELINK         Expose a service privately to other VPCs
                       or accounts without any network routing.
                       ⭐ The most secure option — no route,
                         no CIDR overlap concerns.
   SITE-TO-SITE VPN    Encrypted over the internet. Cheap, but
                       variable latency.
   DIRECT CONNECT      Dedicated physical link. Consistent
                       latency, lower egress cost, weeks to
                       provision.
```

---

## 5. Compute

```
   ┌──────────────────────────────────────────────────────────────┐
   │ EC2          Virtual machines. Maximum control, maximum      │
   │              operational burden. You patch, scale, monitor.  │
   ├──────────────────────────────────────────────────────────────┤
   │ ECS          AWS-native container orchestration. Simpler     │
   │              than Kubernetes, deeply integrated with IAM,    │
   │              ALB, and CloudWatch. ⭐ Underrated when you      │
   │              don't need K8s portability.                     │
   ├──────────────────────────────────────────────────────────────┤
   │ EKS          Managed Kubernetes. ⭐ Choose it for the         │
   │              ecosystem and portability, not because it's     │
   │              easier — it isn't.                              │
   ├──────────────────────────────────────────────────────────────┤
   │ FARGATE      Serverless containers (works with ECS or EKS).  │
   │              No nodes to manage or patch.                    │
   │              ⚠️ ~2-4× the raw compute cost of EC2 — you're   │
   │                paying for the operational savings.           │
   ├──────────────────────────────────────────────────────────────┤
   │ LAMBDA       Function-as-a-service. Per-millisecond billing, │
   │              scales to zero. ⭐ Best for spiky, event-driven, │
   │              short-lived work.                               │
   ├──────────────────────────────────────────────────────────────┤
   │ APP RUNNER   Container → URL, fully managed. Simplest path   │
   │              for a straightforward web service.              │
   └──────────────────────────────────────────────────────────────┘
```

### Choosing compute

```
                    What are you running?
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
   Event-driven,      Long-running          Needs specific
   spiky, short       service                OS/kernel/GPU
        │                   │                    │
        ▼                   ▼                    ▼
     LAMBDA          Containers?               EC2
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
            Want K8s ecosystem?   Just want it to run?
                  │                   │
                  ▼                   ▼
                 EKS            ECS + Fargate
                                (⭐ often the right
                                 answer and rarely
                                 the chosen one)
```

### Lambda in practice

```
   ⭐ THE EXECUTION MODEL

   COLD START    First invocation, or a scale-out event:
                 provision an environment, download code,
                 initialize the runtime, run your init code
                 ~100ms (Go/Rust) to ~2s+ (JVM/.NET with a
                 large dependency graph)

   WARM START    Reuses the environment: only the handler runs
                 ~1-10ms

   ⭐ CODE OUTSIDE THE HANDLER RUNS ONCE PER ENVIRONMENT.
     That's where you create database clients and load config —
     doing it inside the handler pays the cost on every call.
```

```python
# ⭐ Module scope — runs ONCE per execution environment
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def handler(event, context):
    # ⭐ Handler scope — runs per invocation. Keep it minimal.
    return table.get_item(Key={'id': event['id']})
```

```
   ⚠️ LAMBDA'S REAL CONSTRAINTS
   • 15-minute max execution
   • 10 GB memory max · ⭐ CPU scales WITH memory, so more memory
     can be CHEAPER for CPU-bound work (it finishes proportionally
     faster). Use AWS Lambda Power Tuning to find the optimum —
     this is genuinely counterintuitive and often saves 40%+.
   • 6 MB synchronous payload
   • ⚠️ Lambda in a VPC needs ENIs; historically this caused huge
     cold starts. Now much improved with shared ENIs, but still
     adds latency.
   • ⭐ CONNECTION EXHAUSTION: 1,000 concurrent Lambdas each
     opening a database connection will kill an RDS instance.
     → Use RDS Proxy, or prefer DynamoDB (HTTP-based, no
       persistent connections).
```

```
   ⭐ WHEN LAMBDA GETS EXPENSIVE
     Lambda is cheap for spiky workloads and expensive for
     sustained high throughput. A service handling constant
     traffic 24/7 is usually far cheaper on Fargate or EC2.
     The crossover is roughly when you'd need more than a
     couple of always-warm containers.
```

---

## 6. Storage

### S3

```
   ⭐ S3 IS AN OBJECT STORE, NOT A FILESYSTEM.
     No partial writes. No rename (it's a copy + delete).
     Listing is expensive at scale. Eventually consistent for
     some operations historically — now strongly consistent
     for reads after writes.

   STORAGE CLASSES
   ┌────────────────────┬──────────────────────────────────────┐
   │ Standard           │ Frequent access, highest cost        │
   │ Intelligent-Tiering│ ⭐ Auto-moves between tiers. Best     │
   │                    │ default when access is unpredictable │
   │ Standard-IA        │ Infrequent; retrieval fee            │
   │ One Zone-IA        │ Cheaper, ⚠️ single AZ (recreatable    │
   │                    │ data only)                           │
   │ Glacier Instant    │ Archive, millisecond retrieval       │
   │ Glacier Flexible   │ Minutes to hours                     │
   │ Deep Archive       │ ⭐ Cheapest. 12+ hour retrieval.      │
   └────────────────────┴──────────────────────────────────────┘

   ⭐ LIFECYCLE POLICIES move objects automatically.
     "30 days → IA, 90 → Glacier, 365 → Deep Archive, 7y → delete"
     Routinely cuts storage bills 60-80% with no code change.
```

```
   ⚠️⚠️ S3 SECURITY — THE #1 SOURCE OF AWS DATA BREACHES

   □ ⭐ Block Public Access ON at the ACCOUNT level (not just
     per bucket) — this overrides any bucket policy mistake
   □ Default encryption enabled (SSE-S3 or SSE-KMS)
   □ Bucket policies that deny non-TLS requests
   □ Versioning + MFA delete for critical buckets
   □ Access logging or CloudTrail data events
   □ ⭐ Use PRESIGNED URLS for user uploads/downloads — never
     make a bucket public and never proxy bytes through your
     application servers
   □ Object Lock (WORM) for compliance and ⭐ ransomware
     protection — immutable backups that even a compromised
     admin cannot delete
```

```
   ⭐ S3 PERFORMANCE
   • 3,500 PUT/s and 5,500 GET/s per PREFIX, and prefixes
     scale horizontally without limit
   • ⚠️ Sequential key prefixes (like a timestamp) concentrate
     load on one partition. Randomize the prefix, or use a
     hash, for very high write rates.
   • Multipart upload for objects over ~100 MB (and required
     above 5 GB) — also gives parallelism and resumability
   • S3 Transfer Acceleration routes uploads via CloudFront
     edges for distant clients
```

### EBS vs EFS vs S3

```
   ┌────────┬─────────────────────────────────────────────────────┐
   │ EBS    │ Block storage, ⚠️ ONE instance at a time (except     │
   │        │ io2 Multi-Attach). ⭐ AZ-scoped — an EBS volume      │
   │        │ cannot cross AZs. Snapshot to move it.              │
   │        │ gp3 ⭐ — set IOPS and throughput INDEPENDENTLY of    │
   │        │ size, and it's cheaper than gp2. Migrate.           │
   ├────────┼─────────────────────────────────────────────────────┤
   │ EFS    │ NFS. Multi-AZ, many instances read-write.          │
   │        │ ⚠️ Much slower and pricier than EBS. Use only when  │
   │        │ you genuinely need shared POSIX access.            │
   ├────────┼─────────────────────────────────────────────────────┤
   │ S3     │ Object storage. ⭐ The default for anything that    │
   │        │ isn't a filesystem: media, backups, data lake,     │
   │        │ static assets, logs.                               │
   └────────┴─────────────────────────────────────────────────────┘
```

---

## 7. Databases

```
   ┌──────────────────────────────────────────────────────────────┐
   │ RDS          Managed PostgreSQL/MySQL/MariaDB/Oracle/SQLServer│
   │              Multi-AZ = a synchronous standby ⭐ for          │
   │              availability, NOT for read scaling              │
   │              Read replicas = asynchronous, for read scaling  │
   ├──────────────────────────────────────────────────────────────┤
   │ AURORA   ⭐   AWS's reimplementation of Postgres/MySQL with   │
   │              a distributed storage layer: 6 copies across    │
   │              3 AZs, log-structured, ~15 read replicas,       │
   │              much faster failover (~30s vs minutes)          │
   │              Aurora Serverless v2 scales capacity in seconds │
   ├──────────────────────────────────────────────────────────────┤
   │ DYNAMODB     Serverless key-value/document. ⭐ Single-digit   │
   │              ms at any scale, but you MUST know your access  │
   │              patterns before designing the table.            │
   ├──────────────────────────────────────────────────────────────┤
   │ ELASTICACHE  Managed Redis or Memcached                      │
   │ OPENSEARCH   Managed Elasticsearch — search and log analytics│
   │ TIMESTREAM   Time-series · NEPTUNE  Graph · DOCUMENTDB  Mongo│
   │ REDSHIFT     Data warehouse (columnar, MPP) — OLAP not OLTP  │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ MULTI-AZ vs READ REPLICA — the distinction people confuse

   MULTI-AZ        Synchronous standby in another AZ.
                   ⚠️ You CANNOT read from it. It exists purely
                   for automatic failover. Doubles your cost
                   and buys availability, not performance.

   READ REPLICA    Asynchronous copy you CAN read from.
                   ⚠️ Lags behind — expect the read-your-writes
                   problem. Can be promoted manually.

   ⭐ You often want BOTH: Multi-AZ for availability, replicas
     for read scaling.
```

### DynamoDB design

```
   ⭐ SINGLE-TABLE DESIGN — the pattern that confuses everyone

   PK              SK                     Attributes
   ─────────────── ────────────────────── ──────────────────
   USER#123        PROFILE                name, email
   USER#123        ORDER#2026-08-01#456   total, status
   USER#123        ORDER#2026-08-05#789   total, status
   ORDER#456       METADATA               user_id, total

   ⭐ ONE query gets a user AND all their orders, sorted:
     PK = USER#123 AND SK begins_with "ORDER#"

   GSI inverts the access pattern:
     GSI1PK = ORDER#456 → look up an order without the user
```

```
   ⚠️ DYNAMODB PITFALLS
   • ⭐ HOT PARTITIONS — a partition key with skewed access
     throttles. Add a random suffix ("write sharding") for
     very hot keys.
   • Scan is O(table) and expensive. If you're scanning, the
     table design is wrong.
   • 400 KB item limit
   • Eventually consistent reads by default (strongly
     consistent costs 2× and only works on the base table,
     not GSIs)
   • ⭐ On-demand vs provisioned: on-demand is ~7× the per-request
     price but zero capacity planning. Provisioned with
     auto-scaling is much cheaper for steady load.
```

---

## 8. Messaging & Events

```
   ┌──────────────────────────────────────────────────────────────┐
   │ SQS          Queue. At-least-once. Standard (unlimited       │
   │              throughput, best-effort order) or FIFO          │
   │              (ordered per MessageGroupId, 300 TPS)           │
   ├──────────────────────────────────────────────────────────────┤
   │ SNS          Pub/sub fan-out to SQS, Lambda, HTTP, SMS, email│
   ├──────────────────────────────────────────────────────────────┤
   │ EVENTBRIDGE  ⭐ Event bus with content-based routing rules,   │
   │              schema registry, and 100+ SaaS integrations.    │
   │              The modern choice for event-driven architecture.│
   ├──────────────────────────────────────────────────────────────┤
   │ KINESIS      Ordered, replayable stream (Kafka-like) with    │
   │              shards. Multiple consumers, 24h-365d retention. │
   ├──────────────────────────────────────────────────────────────┤
   │ MSK          Managed Kafka — when you need actual Kafka      │
   ├──────────────────────────────────────────────────────────────┤
   │ STEP FUNCTIONS ⭐ Visual state machine for orchestration.     │
   │              Built-in retries, error handling, parallel      │
   │              branches, and human approval steps. The right   │
   │              tool for sagas.                                 │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE STANDARD FAN-OUT PATTERN — worth knowing by name

                          ┌──▶ SQS(billing)   ──▶ Lambda
   Event ──▶ SNS/EventBridge ─▶ SQS(email)    ──▶ ECS worker
                          └──▶ SQS(analytics) ──▶ Firehose → S3

   ⭐ WHY SQS BETWEEN SNS AND THE CONSUMER, rather than SNS →
     Lambda directly?
     • The queue absorbs consumer downtime — messages wait
       instead of being lost
     • Each consumer gets its own retry policy and DLQ
     • Consumers can be scaled and deployed independently
     • You can replay from the DLQ

   ⚠️ SQS VISIBILITY TIMEOUT must EXCEED your processing time,
     or the message is redelivered while you're still working
     on it and the same job runs twice concurrently. This is
     the #1 SQS bug.
```

---

## 9. Content Delivery & DNS

```
   CLOUDFRONT
   • 400+ edge locations, ⭐ terminates TLS at the edge (which
     alone cuts latency substantially even for uncacheable content)
   • Origin can be S3, ALB, or any HTTP endpoint
   • ⭐ Origin Access Control keeps the S3 bucket fully private —
     only CloudFront can read it
   • Lambda@Edge / CloudFront Functions for edge logic
   • ⚠️ Cache key design decides your hit rate. Strip tracking
     query params; ⚠️ never Vary on Cookie.

   ROUTE 53
   ⭐ Routing policies are a real architecture lever:
     Simple · Weighted (⭐ canary and gradual migration) ·
     Latency-based · Geolocation (data residency) ·
     Failover (⭐ DR) · Multivalue
   • Health checks drive automatic failover
   • ⭐ ALIAS records work at the zone apex where CNAME can't,
     and they're free for AWS targets
```

---

## 10. Observability

```
   CLOUDWATCH        Metrics, logs, alarms, dashboards
                     ⚠️ Log ingestion is ~$0.50/GB — this becomes
                       a top-5 line item surprisingly fast.
                       ⭐ Set retention policies; default is
                       "never expire."
                     ⭐ Metric filters extract metrics from logs
                     Logs Insights for ad-hoc querying

   X-RAY             Distributed tracing across AWS services

   CLOUDTRAIL   ⭐    Every API call — WHO did WHAT, WHEN, from
                     WHERE. This is your audit log and your
                     incident forensics.
                     ⭐ Enable org-wide, send to a separate
                       LOGGING ACCOUNT that prod cannot write to,
                       with S3 Object Lock. An attacker who
                       compromises prod must not be able to
                       erase the evidence.

   CONFIG            Resource configuration history + compliance
   GUARDDUTY         ML-based threat detection from VPC flow
                     logs, DNS logs, and CloudTrail
   SECURITY HUB      Aggregates findings across security services
```

---

## 11. Architecture Patterns

### Classic three-tier web app

```
   Route 53 → CloudFront → ALB → ECS/EKS (private subnets)
                                     │
                        ┌────────────┼────────────┐
                        ▼            ▼            ▼
                  RDS Multi-AZ  ElastiCache   S3 (via VPC endpoint)
                        │
                  Read replicas

   ⭐ Everything except the ALB in private subnets.
   ⭐ S3 and DynamoDB reached via free GATEWAY ENDPOINTS,
     not through the NAT Gateway.
```

### Serverless API

```
   CloudFront → API Gateway → Lambda → DynamoDB
                    │             │
                    │             └──▶ EventBridge ──▶ async Lambdas
                    └── Cognito for auth

   ⭐ Scales to zero, no servers to patch.
   ⚠️ Watch: cold starts on latency-sensitive paths, per-request
     cost at sustained high volume, and Lambda→RDS connection
     exhaustion (prefer DynamoDB, or use RDS Proxy).
```

### Event-driven processing

```
   S3 upload ──▶ EventBridge ──▶ Step Functions
                                      │
                    ┌─────────────────┼──────────────────┐
                    ▼                 ▼                  ▼
              Lambda (validate)  Lambda (transform)  Lambda (index)
                    │                 │                  │
                    └────────▶ DynamoDB / OpenSearch ◀───┘

   ⭐ Step Functions gives retries, error handling, and a visual
     execution history for free — far better than chaining
     Lambdas manually.
```

### Disaster recovery tiers

```
   ┌──────────────────┬──────────┬──────────┬────────────────────┐
   │ Strategy         │ RTO      │ RPO      │ Cost               │
   ├──────────────────┼──────────┼──────────┼────────────────────┤
   │ Backup & Restore │ hours    │ hours    │ $                  │
   │ Pilot Light      │ ~10s min │ minutes  │ $$  (data          │
   │                  │          │          │ replicating, no    │
   │                  │          │          │ compute running)   │
   │ Warm Standby     │ minutes  │ seconds  │ $$$ (scaled-down   │
   │                  │          │          │ but running)       │
   │ Multi-Site Active│ ~zero    │ ~zero    │ $$$$               │
   └──────────────────┴──────────┴──────────┴────────────────────┘

   ⭐ RTO = how long until you're back. RPO = how much data you
     can lose. Pick from the BUSINESS requirement, then price it.
   ⚠️ ⭐ AN UNTESTED DR PLAN IS A HYPOTHESIS, NOT A PLAN.
     Run a real failover exercise on a schedule.
```

---

## 12. Cost Engineering

```
   ⭐ THE COSTS THAT SURPRISE PEOPLE, RANKED

   1. ⭐ DATA TRANSFER
      • Internet egress: ~$0.09/GB (⚠️ ingress is free)
      • Cross-AZ: ~$0.01/GB EACH WAY — ⭐ a chatty microservice
        architecture spread across AZs can generate enormous
        cross-AZ charges invisibly
      • Cross-region: more expensive again
      • ⭐ NAT Gateway processing: ~$0.045/GB on top

   2. ⭐ NAT GATEWAY  — see §4. VPC endpoints fix most of it.

   3. IDLE RESOURCES
      Unattached EBS volumes · old snapshots · unused Elastic
      IPs · load balancers with no targets · dev environments
      running nights and weekends

   4. ⭐ CLOUDWATCH LOGS
      ~$0.50/GB ingestion, and the default retention is FOREVER.
      Set retention on every log group.

   5. OVER-PROVISIONING
      Instances sized for peak that run at 5% average.

   6. ⭐ S3 REQUEST COSTS AND INCOMPLETE MULTIPART UPLOADS
      Failed multipart uploads persist and are billed forever
      unless a lifecycle rule aborts them. Add one.
```

```
   ⭐ SAVINGS LEVERS, IN ORDER OF EFFORT/REWARD

   1. Add VPC endpoints for S3 and DynamoDB              (free, instant)
   2. Set CloudWatch log retention                       (5 minutes)
   3. Lifecycle policies on S3 + abort incomplete uploads
   4. ⭐ Delete unattached EBS volumes and old snapshots
   5. Right-size instances using Compute Optimizer
   6. gp2 → gp3 for EBS (cheaper AND more configurable)
   7. Schedule non-prod environments off outside work hours
      (⭐ ~65% saving on those resources for near-zero effort)
   8. Savings Plans / Reserved Instances for steady baseline
      (up to ~72% — ⭐ but commit only to your genuine floor)
   9. Spot for fault-tolerant work (up to 90% off)
   10. Graviton (ARM) instances — ~20% better price/performance
       and usually a one-line change
```

```
   ⭐ TAGGING IS THE PREREQUISITE FOR ALL COST WORK
     Without cost-allocation tags (Environment, Team, Service,
     CostCenter) you cannot attribute spend, which means you
     cannot make anyone accountable for it. Enforce tags via
     SCP or Config rules — voluntary tagging never happens.
```

---

## 13. The Well-Architected Framework

```
   ⭐ SIX PILLARS — a useful checklist, and a common interview prompt

   ① OPERATIONAL EXCELLENCE
     Infrastructure as code · small reversible changes ·
     runbooks · learn from every failure · game days

   ② SECURITY
     Identity foundation (roles not keys) · traceability
     (CloudTrail) · defense in depth · encrypt everywhere ·
     automate security · ⭐ prepare for incidents before them

   ③ RELIABILITY
     Automatic recovery · ⭐ TEST recovery procedures ·
     horizontal scaling · stop guessing capacity ·
     manage change through automation

   ④ PERFORMANCE EFFICIENCY
     Use managed services · go global easily · experiment
     often · use the right resource for the job

   ⑤ COST OPTIMIZATION
     Cloud financial management · pay only for what you use ·
     measure efficiency · stop spending on undifferentiated
     heavy lifting

   ⑥ SUSTAINABILITY
     Right-size · use managed services (higher utilization) ·
     efficient regions · reduce data movement
```

---

## 14. Interview Section

<details>
<summary><b>Q1. Design a highly available web application on AWS.</b></summary>

Route 53 with health checks in front, CloudFront for static assets and TLS termination at the edge, then an Application Load Balancer spanning at least three availability zones.

The application tier runs in private subnets across those AZs — ECS on Fargate or EKS — behind the ALB, with autoscaling on a meaningful metric rather than just CPU.

Data tier is RDS Multi-AZ, or Aurora for faster failover, plus read replicas for read scaling. It's worth being precise here: Multi-AZ gives you a synchronous standby you cannot read from — it's purely for failover. Read replicas are asynchronous and are what actually scales reads. People conflate them constantly.

ElastiCache for session and hot data, S3 for uploads and static content, reached through a VPC gateway endpoint rather than the NAT Gateway.

The details that make it genuinely highly available rather than nominally so: nothing in public subnets except the load balancer, security groups referencing other security groups rather than CIDR blocks, at least three AZs so losing one leaves a quorum, and a tested failover procedure. An untested DR plan is a hypothesis.
</details>

<details>
<summary><b>Q2. Explain IAM policy evaluation.</b></summary>

Four steps. First, is there an explicit deny anywhere — identity policy, resource policy, service control policy, permissions boundary, or session policy? If so, denied, and nothing can override it. Second, is there an explicit allow? If not, denied, because the default is deny. Third, do all applicable boundaries also permit it — SCPs and permissions boundaries cap what can be granted. Fourth, allowed.

Two rules explain nearly every IAM puzzle: explicit deny always wins, and everything is forbidden unless something grants it.

The practical implications matter more than the mechanics. Roles rather than long-lived access keys, because a leaked key in a git repository is the most common AWS compromise. Separate accounts as the strongest blast-radius boundary — an account boundary is far stronger than any IAM policy. SCPs to make dangerous actions impossible organization-wide, like denying the ability to disable CloudTrail. And Access Analyzer to generate least-privilege policies from actual CloudTrail usage rather than guessing.
</details>

<details>
<summary><b>Q3. Security groups vs NACLs?</b></summary>

Security groups attach to network interfaces and are stateful — return traffic is automatically allowed. They only support allow rules, all rules are evaluated together, and crucially they can reference other security groups as a source.

NACLs attach to subnets and are stateless, so you must explicitly allow return traffic on ephemeral ports. They support both allow and deny, and rules are numbered and evaluated in order with first match winning.

In practice, security groups are the primary tool. The pattern worth using is referencing security groups instead of CIDR blocks — "allow the app security group on 5432" rather than "allow 10.0.11.0/24." That rule then follows the application wherever it runs, with no CIDR maintenance.

NACLs are for coarse subnet-level deny rules, like blocking a known-bad IP range. Their statelessness makes them easy to misconfigure, and the usual symptom is traffic that works one direction and silently times out on the return.
</details>

<details>
<summary><b>Q4. What's the biggest hidden cost in AWS?</b></summary>

Data transfer, and specifically NAT Gateway.

NAT Gateway costs about thirty-two dollars a month just to exist, plus roughly four and a half cents per gigabyte processed. You need one per AZ for availability, so three AZs is a hundred dollars before any traffic. Then every container image pull, OS package update, and S3 call from a private subnet flows through it and is billed per gigabyte. Teams routinely discover four-figure monthly NAT bills for traffic that never leaves AWS.

The fix is VPC endpoints. Gateway endpoints for S3 and DynamoDB are completely free and route over the AWS backbone instead. Adding an S3 gateway endpoint is often the single highest-return change you can make in an account.

Close behind: cross-AZ transfer at a cent per gigabyte each way, which a chatty microservice architecture spread across AZs generates invisibly. And CloudWatch Logs at fifty cents per gigabyte ingested, where the default retention is forever — setting retention on log groups takes five minutes and often cuts a meaningful line item.
</details>

<details>
<summary><b>Q5. When would you choose Lambda over containers?</b></summary>

Lambda for spiky, event-driven, short-lived work. Scheduled jobs, S3 upload processing, queue consumers with variable volume, webhook handlers, glue between services. The scale-to-zero property is the real value — you pay nothing when idle, and it absorbs a hundred-fold spike without any capacity planning.

Containers for sustained traffic, anything needing more than fifteen minutes, workloads with heavy dependencies where cold starts hurt, or where you need specific runtime control.

The economics matter and people get them backwards. Lambda is cheap for bursty work and expensive at sustained high throughput. Somewhere around the point where you'd need a couple of always-warm containers, Fargate becomes cheaper.

The operational traps: cold starts on latency-sensitive paths, especially for JVM runtimes. And connection exhaustion — a thousand concurrent Lambdas each opening a database connection will overwhelm an RDS instance, which is why RDS Proxy exists and why DynamoDB pairs better with Lambda, being HTTP-based with no persistent connections.

One counterintuitive tuning note: since CPU scales with memory, allocating more memory to a CPU-bound Lambda often reduces total cost because it finishes proportionally faster.
</details>

<details>
<summary><b>Q6. How do you secure an S3 bucket?</b></summary>

Start with Block Public Access enabled at the account level, not just per bucket. That overrides any bucket policy mistake and is the single control that would have prevented most publicized S3 breaches.

Then default encryption with SSE-KMS if you need key control and audit trails, a bucket policy denying any request where `aws:SecureTransport` is false, and versioning with MFA delete on anything critical.

For access, use presigned URLs for user uploads and downloads. Never make a bucket public, and never proxy the bytes through your application servers — that costs double bandwidth and ties up capacity for the whole transfer.

If CloudFront serves the content, use Origin Access Control so the bucket stays entirely private and only CloudFront can read it.

For ransomware and insider risk, Object Lock in compliance mode gives you write-once-read-many backups that even a compromised administrator cannot delete. That's increasingly the control that matters most.

And enable CloudTrail data events for the bucket, so you actually have forensics if something goes wrong.
</details>

<details>
<summary><b>Q7. RDS Multi-AZ vs read replicas vs Aurora?</b></summary>

Multi-AZ maintains a synchronous standby in another availability zone. You cannot read from it — it exists solely for automatic failover. It roughly doubles cost and buys availability, not performance.

Read replicas are asynchronous copies you can read from, which is what actually scales reads. They lag, so you get the read-your-writes problem: a user writes, gets redirected to a read, and sees stale data or a 404. You handle that by routing a user's reads to the primary for a short window after they write.

Most production systems want both.

Aurora is a reimplementation of the Postgres and MySQL engines on a distributed storage layer — six copies across three AZs, log-structured writes, up to fifteen read replicas sharing the same storage, and failover in around thirty seconds rather than a minute or more. Replicas don't need their own copy of the data, which makes them cheaper and faster to add.

I'd default to Aurora for new production workloads unless there's a specific reason for stock RDS, like needing an engine version or extension Aurora doesn't support. Aurora Serverless v2 is also worth considering for variable workloads since it scales capacity in seconds.
</details>

<details>
<summary><b>Q8. Design a disaster recovery strategy.</b></summary>

Start from the business requirement, not the technology. Two numbers: RTO, how long until we're back, and RPO, how much data we can afford to lose. Those determine which tier you can justify.

Backup and restore gives hours of RTO and RPO, and costs almost nothing. Pilot light replicates data continuously with compute stopped, giving tens of minutes RTO. Warm standby runs a scaled-down copy for minutes of RTO. Multi-site active-active gives near-zero for both, at multiples of the cost.

Whatever tier, the essentials are the same: infrastructure as code so the environment can be recreated rather than manually rebuilt, cross-region replication for S3 and database snapshots, and Route 53 health checks with failover routing.

The part that actually matters is testing. An untested DR plan is a hypothesis. I'd run scheduled failover exercises — game days — with the actual on-call team, because the failures are always in the parts nobody documented: a hardcoded region, a certificate that only exists in one place, an IAM role that wasn't replicated.

I'd also be clear about what DR does not cover. Cross-region failover doesn't protect against a bad deploy or a data corruption bug, because you'd replicate both. That needs point-in-time recovery and immutable backups, which is a different control.
</details>

---

## 15. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                         AWS — ONE PAGE                               ║
╠══════════════════════════════════════════════════════════════════════╣
║ FOR EVERY SERVICE ASK: who operates it? what's the failure domain?   ║
║   how is it priced (⭐ especially DATA TRANSFER)?                     ║
║ Multi-AZ is the production baseline. Multi-region is expensive       ║
║   and hard. Multi-cloud is usually a mistake.                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ IAM: explicit DENY always wins · default is DENY                     ║
║   ⭐ ROLES not access keys · separate ACCOUNTS = strongest boundary   ║
║   SCPs as org-wide guardrails · Access Analyzer for least privilege  ║
╠══════════════════════════════════════════════════════════════════════╣
║ VPC: a subnet is PUBLIC only because its route table → IGW           ║
║   SG = stateful, allow-only, ⭐ can reference other SGs               ║
║   NACL = stateless (must allow return traffic), ordered, allow+deny  ║
║   ⚠️⚠️ NAT GATEWAY: $32/mo each + $0.045/GB → ⭐ VPC ENDPOINTS         ║
║      (S3/DynamoDB gateway endpoints are FREE — biggest easy win)     ║
╠══════════════════════════════════════════════════════════════════════╣
║ COMPUTE: Lambda(spiky/event) · Fargate(containers, no nodes) ·       ║
║   ECS(simpler than K8s) · EKS(ecosystem) · EC2(control)              ║
║   ⭐ Lambda: init OUTSIDE the handler · CPU scales WITH memory        ║
║     ⚠️ 1000 concurrent Lambdas will exhaust RDS connections           ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ RDS Multi-AZ = FAILOVER ONLY (can't read from it)                  ║
║   Read replicas = read scaling, async, lag → read-your-writes bug    ║
║   Aurora = 6 copies/3 AZs, ~30s failover, 15 replicas                ║
║ DynamoDB: know access patterns FIRST · ⚠️ hot partitions · no Scan    ║
╠══════════════════════════════════════════════════════════════════════╣
║ S3: ⭐ Block Public Access at the ACCOUNT level (prevents most        ║
║   breaches) · presigned URLs, never proxy bytes · lifecycle policies ║
║   (60-80% savings) · Object Lock for ransomware protection           ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ FAN-OUT: SNS/EventBridge → SQS per consumer → workers              ║
║   (the queue gives each consumer its own retry + DLQ + buffering)    ║
║   ⚠️ SQS visibility timeout MUST exceed processing time              ║
║ Step Functions for sagas — retries and error handling built in       ║
╠══════════════════════════════════════════════════════════════════════╣
║ COST: data transfer > NAT > idle resources > CloudWatch Logs         ║
║   quick wins: VPC endpoints · log retention · delete orphan EBS ·    ║
║   gp2→gp3 · schedule non-prod off · Graviton · Savings Plans         ║
║   ⭐ tag enforcement is the prerequisite for all cost work            ║
╠══════════════════════════════════════════════════════════════════════╣
║ DR: pick from RTO/RPO business needs, then ⭐ ACTUALLY TEST IT        ║
║   CloudTrail → a separate logging account with Object Lock, so a     ║
║   prod compromise can't erase the evidence                           ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Kubernetes](kubernetes.md) · [Terraform](terraform.md) · [Observability & SRE](observability-sre.md) · [Network Security](../07-security/network-security.md)
