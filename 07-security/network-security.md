# 🛡️ Network & Infrastructure Security

> The perimeter model is dead — not because it was wrong, but because there is no longer a perimeter. This book covers what replaced it, and how to defend infrastructure you don't physically own.

**Prerequisite:** [Linux & Networking](../06-cloud-devops/linux-networking.md) · [AppSec](appsec.md)

---

## 📑 Contents

1. [The Death of the Perimeter](#1-the-death-of-the-perimeter)
2. [Zero Trust](#2-zero-trust)
3. [Network Segmentation](#3-network-segmentation)
4. [Firewalls and Filtering](#4-firewalls-and-filtering)
5. [DDoS Defense](#5-ddos-defense)
6. [Cloud Security](#6-cloud-security)
7. [Identity as the Perimeter](#7-identity-as-the-perimeter)
8. [Container & Kubernetes Security](#8-container--kubernetes-security)
9. [Detection & Monitoring](#9-detection--monitoring)
10. [Common Infrastructure Attacks](#10-common-infrastructure-attacks)
11. [Hardening Checklists](#11-hardening-checklists)
12. [Interview Section](#12-interview-section)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. The Death of the Perimeter

```
   ⭐ THE OLD MODEL — "castle and moat"

   ┌──────────────────────────────────────────────────────────────┐
   │                    ⚠️ TRUSTED INTERNAL NETWORK                │
   │                                                              │
   │    [App] ──── [DB] ──── [File server] ──── [Admin box]       │
   │       everything here trusts everything else                 │
   │                                                              │
   └────────────────────────┬─────────────────────────────────────┘
                       ┌────┴────┐
                       │ FIREWALL│  ← the single control point
                       └────┬────┘
                     UNTRUSTED INTERNET

   ⚠️ THE FATAL ASSUMPTION: "inside the network" == "trustworthy"

   ⭐ WHY IT COLLAPSED
     • Cloud — there is no inside
     • SaaS — your data lives on someone else's network
     • Remote work — employees are outside by default
     • Mobile devices, contractors, partners
     • ⭐ AND MOST IMPORTANTLY: once an attacker phishes ONE
       employee, they are "inside" — and the flat trusted
       network hands them everything.

   ⭐ Nearly every large breach follows this shape: modest
     initial access, then unimpeded LATERAL MOVEMENT because
     the internal network trusted itself.
```

---

## 2. Zero Trust

```
   ⭐ THE PRINCIPLE: "NEVER TRUST, ALWAYS VERIFY."
     Network location grants NOTHING. Every request is
     authenticated and authorized on its own merits.
```

```
   ┌──────────────────────────────────────────────────────────────┐
   │ ① VERIFY EXPLICITLY                                          │
   │   Authenticate and authorize on every request, using all     │
   │   available signals: identity, device posture, location,     │
   │   behaviour, and the sensitivity of what's being accessed.   │
   ├──────────────────────────────────────────────────────────────┤
   │ ② LEAST PRIVILEGE                                            │
   │   Just-in-time and just-enough access. ⭐ Short-lived         │
   │   credentials rather than standing permissions.              │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ ⭐ ASSUME BREACH                                            │
   │   Design as though the attacker is already inside. Segment   │
   │   to limit blast radius, encrypt end to end, and instrument  │
   │   everything for detection.                                  │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHAT IT LOOKS LIKE IN PRACTICE

   ┌──────────────────────────────────────────────────────────────┐
   │ USERS      SSO + MFA (⭐ phishing-resistant), device posture  │
   │            checks, conditional access policies, and          │
   │            per-application authorization — no flat VPN into  │
   │            "the network"                                     │
   ├──────────────────────────────────────────────────────────────┤
   │ SERVICES   ⭐ mTLS with SPIFFE-style workload identity.       │
   │            A service proves WHO IT IS cryptographically,     │
   │            not by which subnet it sits in.                   │
   ├──────────────────────────────────────────────────────────────┤
   │ DEVICES    Managed, patched, encrypted, attested — device    │
   │            health as an access signal                        │
   ├──────────────────────────────────────────────────────────────┤
   │ DATA       Classified, encrypted, access logged              │
   ├──────────────────────────────────────────────────────────────┤
   │ NETWORK    ⭐ Microsegmentation — default-deny between        │
   │            workloads, not just at the edge                   │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️ WHAT ZERO TRUST IS NOT
     • A product you can buy (vendors will disagree)
     • "No VPN" — it's about what access a VPN grants
     • ⭐ A reason to remove network controls. Defense in depth
       still applies; zero trust ADDS identity-based controls,
       it doesn't replace segmentation.
     • Achievable in one project. It's a multi-year direction.

   ⭐ THE HIGHEST-VALUE FIRST STEPS
     1. Phishing-resistant MFA everywhere (passkeys/FIDO2)
     2. Eliminate standing admin privileges → just-in-time
     3. Inventory and segment the crown jewels
     4. mTLS between services
```

---

## 3. Network Segmentation

```
   ⭐ SEGMENTATION EXISTS TO LIMIT BLAST RADIUS.
     Not to prevent initial access — to prevent the compromise
     of one thing from becoming the compromise of everything.

   ┌──────────────────────────────────────────────────────────────┐
   │  DMZ / PUBLIC     load balancers, WAF, bastion                │
   │       │ ⭐ only specific ports, one direction                  │
   │  APPLICATION      app servers — ⚠️ NO direct internet inbound  │
   │       │ ⭐ only the DB port, only from app SGs                 │
   │  DATA             databases, caches — ⭐ NO internet at all,   │
   │                   inbound OR outbound                        │
   ├──────────────────────────────────────────────────────────────┤
   │  MANAGEMENT       ⭐ separate: monitoring, CI/CD, admin        │
   │                   Compromising the app tier must NOT reach    │
   │                   the deployment pipeline.                   │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ MICROSEGMENTATION — the modern form

   Instead of coarse subnets, policy is per-WORKLOAD and
   identity-based:

     "the checkout service may talk to the payments service
      on port 8443, and to nothing else"

   Implemented with: Kubernetes NetworkPolicy, service mesh
   authorization policies, cloud security groups referencing
   other security groups, or host-based agents.

   ⭐ THE KEY SHIFT: policy follows the WORKLOAD, not the IP
     address. In an autoscaling environment IPs are ephemeral,
     so CIDR-based rules are unmaintainable and quietly rot.
```

```
   ⚠️ EGRESS FILTERING IS THE MOST NEGLECTED CONTROL

   Almost everyone restricts inbound traffic. Very few restrict
   OUTBOUND.

   ⭐ But outbound is how attacks actually complete:
     • Command-and-control callbacks
     • Data exfiltration
     • Downloading the second-stage payload
     • ⭐ SSRF reaching the cloud metadata endpoint

   Even a modest egress allowlist — package registries, your
   own APIs, known SaaS — breaks a large fraction of real
   attack chains. It's high-value and unglamorous.
```

---

## 4. Firewalls and Filtering

```
   ┌──────────────────────────────────────────────────────────────┐
   │ PACKET FILTER (L3/L4)   IP, port, protocol. Stateless.       │
   │ STATEFUL FIREWALL       Tracks connections → return traffic  │
   │                         is allowed automatically             │
   │ ⭐ NGFW                  + application awareness, IPS, TLS    │
   │                         inspection                           │
   │ ⭐ WAF (L7)              HTTP-aware: blocks SQLi/XSS patterns,│
   │                         rate limits, bot management          │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️ A WAF IS NOT A FIX — IT'S A SPEED BUMP AND A SENSOR.

   ✅ GENUINELY VALUABLE FOR:
     • ⭐ VIRTUAL PATCHING — buying time between a CVE
       disclosure and your deploy. This is its best use.
     • Blocking automated scanners and commodity attacks
     • Rate limiting and bot management
     • ⭐ Visibility — telling you that you're being probed

   ⚠️ NOT A SUBSTITUTE FOR:
     • Fixing the actual vulnerability. WAF bypasses are a
       whole research field, and signature evasion is routine.
     • Authorization logic. ⭐ A WAF cannot possibly know that
       user 5 shouldn't read order 9 — the BOLA class is
       completely invisible to it.

   ⭐ RUN IT IN DETECTION MODE FIRST. Blocking mode without
     tuning generates false positives that break legitimate
     traffic, and teams then disable it entirely.
```

```
   ⭐ FIREWALL RULE HYGIENE

   □ DEFAULT DENY, both directions
   □ ⭐ Reference identities/groups, not CIDR blocks —
     "allow sg-app" survives autoscaling; "allow 10.0.1.0/24"
     rots
   □ Document WHY each rule exists and who owns it
   □ ⚠️ Review and prune regularly — rule sets accumulate
     permanently because nobody dares delete an unexplained rule
   □ Prefer identity-based policy over IP-based where possible
   □ ⚠️ Any rule allowing 0.0.0.0/0 inbound needs explicit
     justification
```

---

## 5. DDoS Defense

```
   ⭐ THREE LAYERS OF ATTACK, THREE DIFFERENT DEFENSES

   ┌──────────────────────────────────────────────────────────────┐
   │ VOLUMETRIC (L3/L4)   Saturate the pipe.                      │
   │   ⭐ AMPLIFICATION: send a small spoofed query to a DNS/NTP/  │
   │     memcached server, which sends a huge reply to the victim.│
   │     Amplification factors reach 50,000× with memcached.      │
   │   ⭐ DEFENSE: you CANNOT absorb this on your own bandwidth.   │
   │     You need an upstream scrubbing provider or an anycast    │
   │     CDN. This is a capacity problem, not a code problem.     │
   ├──────────────────────────────────────────────────────────────┤
   │ PROTOCOL (L4)        Exhaust connection state.               │
   │   SYN flood, ⭐ Slowloris (open many connections, send        │
   │   headers one byte at a time to hold them open).             │
   │   DEFENSE: SYN cookies, connection limits per source,        │
   │   aggressive timeouts, a reverse proxy that buffers          │
   │   complete requests.                                         │
   ├──────────────────────────────────────────────────────────────┤
   │ APPLICATION (L7)  ⭐ The hard one.                            │
   │   Requests that LOOK legitimate but are expensive — search   │
   │   queries, report generation, password hashing endpoints.    │
   │   ⚠️ Low volume, so bandwidth defenses don't see it.          │
   │   DEFENSE: rate limiting, ⭐ COST-BASED limits (charge more   │
   │   for expensive endpoints), caching, CAPTCHA/proof-of-work,  │
   │   and behavioural bot detection.                             │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ PREPARATION MATTERS MORE THAN REACTION

   □ CDN / anycast in front — absorbs volumetric attacks and
     distributes them geographically by design
   □ ⚠️ HIDE YOUR ORIGIN IP. If attackers can reach the origin
     directly, the CDN is irrelevant. Firewall the origin to
     accept only CDN ranges.
   □ Autoscaling with sane caps (⚠️ or a DDoS becomes a very
     large bill instead of an outage — "economic denial of
     sustainability")
   □ Rate limiting at the edge, so rejected traffic never
     reaches your servers
   □ ⭐ Graceful degradation: shed expensive features and serve
     cached content rather than failing entirely
   □ A runbook and a pre-established contact with your provider
```

---

## 6. Cloud Security

```
   ⭐ THE SHARED RESPONSIBILITY LINE MOVES BY SERVICE

   EC2       you patch the OS, the runtime, and the app
   RDS       AWS patches the engine; you handle access, network,
             encryption, and backups
   Lambda    AWS handles everything below your code
   S3        AWS handles durability; ⭐ YOU handle access policy —
             which is where nearly every S3 breach comes from

   ⚠️ The failure is almost never AWS's infrastructure. It's a
     misconfiguration on the customer side.
```

```
   ⭐ THE TOP CLOUD MISCONFIGURATIONS, BY REAL-WORLD IMPACT

   1. ⭐ PUBLIC STORAGE BUCKETS
      → Block Public Access at the ACCOUNT level, not per bucket
   2. ⭐ OVER-PRIVILEGED IAM
      Wildcards in policies, roles that accumulate permissions
      and never lose them
      → Access Analyzer to generate least privilege from actual
        CloudTrail usage
   3. ⭐ EXPOSED MANAGEMENT PORTS
      SSH, RDP, database ports open to 0.0.0.0/0
      → SSM Session Manager instead of SSH; no bastion at all
   4. UNENCRYPTED DATA
      → Enforce encryption via SCP so it can't be turned off
   5. ⭐ DISABLED OR DELETABLE LOGGING
      → CloudTrail org-wide, to a SEPARATE logging account with
        S3 Object Lock, so a prod compromise can't erase evidence
   6. ⚠️ LONG-LIVED ACCESS KEYS
      → Roles and OIDC federation. A leaked key in a public repo
        is the single most common cloud compromise.
   7. NO MFA on privileged accounts
   8. ⚠️ Overly permissive security groups between tiers
```

```
   ⭐ THE INSTANCE METADATA ENDPOINT — the highest-value target

   169.254.169.254 returns temporary IAM credentials for the
   instance role. An SSRF that reaches it is a full cloud
   compromise, not an information leak. ⭐ This is how the
   Capital One breach happened.

   DEFENSES
     • ⭐ IMDSv2 — requires a session token obtained via PUT
       with a hop limit, which a simple SSRF cannot perform.
       ⭐ ENFORCE IT (disable IMDSv1) via SCP or launch template.
     • Minimal instance role permissions
     • Egress filtering and application-level SSRF protection
     • Set the hop limit to 1 so containers can't reach it
```

```
   ⭐ ACCOUNT SEPARATION IS THE STRONGEST BOUNDARY

   Stronger than any IAM policy, because it's a different
   trust domain entirely.

   prod · staging · dev · ⭐ security · ⭐ logging · shared services

   With SCPs at the organization level enforcing invariants
   that no one in the account can override — such as denying
   the ability to disable CloudTrail or leave the organization.
```

---

## 7. Identity as the Perimeter

```
   ⭐ IF NETWORK LOCATION GRANTS NOTHING, IDENTITY IS THE
     CONTROL PLANE — which makes the identity provider the
     highest-value target in the organization.

   ┌──────────────────────────────────────────────────────────────┐
   │ HUMANS                                                       │
   │   SSO with ⭐ phishing-resistant MFA (passkeys/FIDO2)         │
   │   ⭐ Just-in-time privilege elevation, not standing admin     │
   │   Conditional access: device posture, location, risk score   │
   │   Automated deprovisioning on offboarding (⚠️ the step most   │
   │     commonly missed — orphaned accounts are a real vector)   │
   ├──────────────────────────────────────────────────────────────┤
   │ WORKLOADS                                                    │
   │   ⭐ Workload identity: IRSA, GKE Workload Identity,          │
   │     SPIFFE/SPIRE — short-lived credentials issued            │
   │     automatically, so there is NO static secret to steal     │
   │   mTLS for service-to-service                                │
   │   ⚠️ NEVER long-lived static keys                             │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ PRIVILEGED ACCESS MANAGEMENT

   Standing admin access is the thing attackers hunt for.

   ✅ Break-glass accounts, sealed and monitored
   ✅ ⭐ Time-bound elevation with approval and a business reason
   ✅ Session recording for privileged sessions
   ✅ Separate admin identities from daily-use identities
   ⭐ THE GOAL: at any given moment, almost nobody holds admin.
     Phishing a developer then yields a developer's access,
     not the keys to production.
```

---

## 8. Container & Kubernetes Security

```
   ⭐ THE LAYERS — an attacker moves upward through these

   ┌──────────────────────────────────────────────────────────────┐
   │ ① IMAGE      minimal base · pinned by DIGEST · scanned ·     │
   │              signed · ⚠️ no secrets in layers                 │
   ├──────────────────────────────────────────────────────────────┤
   │ ② RUNTIME    non-root · read-only rootfs · drop ALL caps ·   │
   │              seccomp · ⚠️ NEVER privileged                    │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ ORCHESTRATOR  RBAC least privilege · Pod Security          │
   │              Admission `restricted` · admission policies     │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ NETWORK    ⭐ default-deny NetworkPolicy · mTLS · egress    │
   │              restrictions                                    │
   ├──────────────────────────────────────────────────────────────┤
   │ ⑤ HOST       patched kernel (⭐ container isolation IS the    │
   │              kernel) · minimal OS · CIS benchmarks           │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️⚠️ THE CONTAINER ESCAPE PATHS — know these by name

   • ⭐ MOUNTED DOCKER SOCKET
     /var/run/docker.sock in a container = ROOT ON THE HOST.
     The attacker starts a new privileged container with the
     host filesystem mounted. Complete compromise.
   • privileged: true — effectively no isolation at all
   • hostPID / hostNetwork / hostPath mounts
   • Excessive capabilities: CAP_SYS_ADMIN is nearly root
   • ⭐ Kernel vulnerabilities — containers share the kernel,
     so a kernel exploit escapes by definition
   • Writable /proc or /sys mounts

   ⭐ FOR GENUINELY UNTRUSTED WORKLOADS, containers are the wrong
     boundary. Use gVisor (user-space kernel), Kata Containers
     (lightweight VMs), or Firecracker microVMs.
```

```
   ⭐ THE KUBERNETES SPECIFICS THAT MATTER MOST

   □ ⚠️ Secrets are BASE64, not encrypted → enable etcd
     encryption at rest, and prefer External Secrets or
     workload identity
   □ ⚠️ By default ALL pods can reach ALL pods → default-deny
     NetworkPolicy per namespace
   □ automountServiceAccountToken: false unless the pod
     genuinely calls the API server
   □ ⭐ RBAC: never bind cluster-admin to a ServiceAccount.
     Audit with `kubectl auth can-i --list --as=...`
   □ ⭐ etcd IS the cluster — encrypt it, restrict access,
     back it up
   □ Admission control (Kyverno/Gatekeeper) to ENFORCE the
     above automatically rather than relying on review
```

---

## 9. Detection & Monitoring

```
   ⭐ PREVENTION FAILS. DETECTION IS WHAT BOUNDS THE DAMAGE.

   ⚠️ Industry median dwell time — the gap between compromise
     and detection — has historically been measured in WEEKS.
     That's the window in which an attacker moves laterally,
     escalates, and exfiltrates.
```

```
   ⭐ WHAT TO COLLECT, BY VALUE

   ┌──────────────────────────────────────────────────────────────┐
   │ ⭐ CLOUD AUDIT LOGS (CloudTrail)  who did what, when, from    │
   │    where — the single most valuable source for investigation │
   │ AUTHENTICATION LOGS  ⭐ especially FAILURES and impossible    │
   │    travel                                                    │
   │ ⭐ AUTHORIZATION FAILURES  a strong attack signal — normal    │
   │    users rarely trigger these                                │
   │ DNS QUERIES  ⭐ reveals C2 domains and exfiltration           │
   │ VPC FLOW LOGS  unexpected internal connections               │
   │ PROCESS EXECUTION on hosts (eBPF/Falco)                      │
   │ FILE INTEGRITY on critical paths                             │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ HIGH-SIGNAL DETECTIONS — worth building first

   • ⭐ Impossible travel (login from two distant locations
     within an implausible interval)
   • New IAM user, key, or role created
   • ⭐ Privilege escalation — a policy granting * on *
   • CloudTrail or logging DISABLED (⭐ a near-certain sign of
     an active intrusion)
   • Security group opened to 0.0.0.0/0
   • Large or unusual data egress volume
   • ⭐ Access to the metadata endpoint from a container
   • Execution from /tmp or a writable directory
   • ⭐ Outbound connections to newly registered domains
   • Login from a Tor exit node or a known-bad ASN
   • A service account authenticating from an unusual location
```

```
   ⚠️⚠️ PROTECT THE LOGS THEMSELVES
     An attacker's first move after gaining access is often to
     disable or delete logging.

   ⭐ Ship logs to a SEPARATE ACCOUNT that production
     credentials cannot write to or delete from, with S3
     Object Lock for immutability. Alert loudly if logging
     stops.

   ⭐ "The logs went quiet" is itself a critical detection.
```

---

## 10. Common Infrastructure Attacks

```
   ⭐ THE ATTACK CHAIN — most breaches follow this shape

   ┌──────────────────────────────────────────────────────────────┐
   │ ① INITIAL ACCESS                                             │
   │    phishing · exposed service · leaked credential ·          │
   │    unpatched CVE · supply chain                              │
   │ ② EXECUTION / PERSISTENCE                                    │
   │    backdoor · scheduled task · new IAM user · added SSH key  │
   │ ③ ⭐ PRIVILEGE ESCALATION                                     │
   │    kernel exploit · misconfigured sudo · over-broad IAM role │
   │ ④ ⭐ LATERAL MOVEMENT                                         │
   │    stolen credentials · trust relationships · flat network   │
   │ ⑤ COLLECTION & EXFILTRATION                                  │
   │    stage data, then move it out over DNS, HTTPS, or cloud    │
   │    storage                                                   │
   │ ⑥ IMPACT                                                     │
   │    ransomware · destruction · extortion                      │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE DEFENSIVE INSIGHT: you don't need to block every step.
     Breaking ANY link stops the chain. Segmentation breaks (4),
     egress filtering breaks (5), least privilege breaks (3).
     Layered defenses give you multiple chances.
```

```
   ⭐ SPECIFIC ATTACKS WORTH KNOWING

   CREDENTIAL STUFFING   Reusing breached passwords at scale.
     ⭐ Defense: MFA, breached-password checks, rate limiting
       per account AND per IP, device fingerprinting.

   PASSWORD SPRAYING     One common password against many
     accounts — evades per-account lockout entirely.
     ⭐ Defense: detect by looking at failures ACROSS accounts,
       not per account.

   KERBEROASTING (AD)    Request service tickets and crack them
     offline for service account passwords.
     Defense: long random passwords for service accounts,
     managed service accounts.

   ⚠️ DNS HIJACKING       Compromise the registrar or DNS provider
     and redirect traffic — ⭐ including issuing valid
     certificates for your domain.
     Defense: registrar lock, MFA on the registrar account,
     DNSSEC, CAA records, ⭐ Certificate Transparency monitoring.

   BGP HIJACKING         Announce someone else's IP ranges.
     Defense: RPKI/ROA. Largely an ISP-level concern, but it
     has been used to steal cryptocurrency at scale.

   ⭐ SUPPLY CHAIN        Compromise a dependency, a build system,
     or an update mechanism. SolarWinds, Codecov, xz-utils.
     Defense: pin by digest, SBOM, signed artifacts, minimal
     CI permissions, and reproducible builds where feasible.
```

---

## 11. Hardening Checklists

```
   ── HOST ───────────────────────────────────────────────────────
   □ Minimal OS install; remove unused packages and services
   □ Automatic security updates, and a patch SLA that's tracked
   □ ⭐ SSH: keys only, no root login, no password auth
     ⭐ Better: no SSH at all — SSM Session Manager or equivalent
   □ Host firewall default-deny, both directions
   □ ⭐ Disk encryption
   □ Audit logging (auditd/eBPF) shipped off-host
   □ File integrity monitoring on critical paths
   □ ⭐ Time synchronization (⚠️ log correlation and certificate
     validation both break with clock skew)
   □ CIS Benchmark as a baseline

   ── NETWORK ────────────────────────────────────────────────────
   □ Default-deny in BOTH directions
   □ ⭐ EGRESS FILTERING (the most neglected high-value control)
   □ Segmentation by trust level and by workload
   □ No management ports exposed to the internet
   □ ⭐ TLS everywhere, including internal traffic
   □ DNS filtering for known-malicious domains
   □ Flow logs enabled and retained

   ── IDENTITY ───────────────────────────────────────────────────
   □ SSO with ⭐ phishing-resistant MFA
   □ ⭐ No standing privileged access — JIT elevation only
   □ Service accounts use workload identity, not static keys
   □ Automated deprovisioning on offboarding
   □ Regular access reviews
   □ Break-glass accounts sealed and alerted on

   ── DATA ───────────────────────────────────────────────────────
   □ Encrypted at rest and in transit
   □ ⭐ Classified — you cannot protect what you haven't inventoried
   □ Backups: ⭐ tested restores, and immutable/offline copies
     (ransomware deletes reachable backups first)
   □ Retention and deletion policies enforced
   □ DLP on egress paths for sensitive data
```

---

## 12. Interview Section

<details>
<summary><b>Q1. What is zero trust, and how is it different from a VPN?</b></summary>

Zero trust means network location grants nothing. Every request is authenticated and authorized on its own merits, using identity, device posture, and context — regardless of whether it originates inside or outside your network.

A traditional VPN is the opposite model. It authenticates you once at the boundary and then places you *inside* a trusted network, where you often have broad reachability. That's the castle-and-moat design, and its failure mode is that phishing one employee gives an attacker the same broad internal access.

The practical difference is granularity and continuity. Zero trust grants access to a specific application, re-evaluated per request, with signals like device health factored in — rather than granting network reachability once and trusting it indefinitely.

For services it means mTLS with workload identity, so a service proves cryptographically who it is rather than being trusted because of its subnet.

Two clarifications I'd add. Zero trust doesn't mean removing network controls — defense in depth still applies, and microsegmentation is part of it. And it's a multi-year direction, not a product. The highest-value first steps are phishing-resistant MFA, eliminating standing admin privileges, and mTLS between services.
</details>

<details>
<summary><b>Q2. Why does egress filtering matter?</b></summary>

Because nearly every attack requires outbound connectivity to complete, and almost nobody restricts it.

Command-and-control callbacks, second-stage payload downloads, and data exfiltration all need to reach out. And SSRF reaching the cloud metadata endpoint is an outbound request too.

So even a modest egress allowlist — package registries, your own APIs, known SaaS dependencies — breaks a large fraction of real attack chains. An attacker who achieves code execution but cannot call home has dramatically less capability.

The reason it's neglected is friction. Inbound rules are obvious and easy to reason about; outbound rules break things in ways that are hard to predict, and every new dependency needs a rule.

I'd approach it by starting in log-only mode to discover what's actually needed, then moving to enforcement tier by tier — beginning with the data tier, which legitimately needs almost no internet access at all.

It also pairs well with DNS filtering, since observing DNS queries reveals both C2 domains and the destinations of exfiltration attempts.
</details>

<details>
<summary><b>Q3. A WAF blocks SQL injection. Do we still need parameterized queries?</b></summary>

Yes, and the WAF is not a substitute in any meaningful sense.

WAFs work on pattern matching, and bypass techniques are an entire research field — encoding tricks, comment insertion, alternate syntax, chunked requests. Signature evasion is routine, not exotic.

More fundamentally, a WAF can only detect attack *patterns*. It cannot understand your application's logic. It has no way of knowing that user 5 shouldn't be able to read order 9 — the broken object level authorization class, which is the number one cause of real API breaches, is completely invisible to it.

Where a WAF genuinely earns its place is virtual patching — buying time between a CVE disclosure and your deploy — plus blocking commodity scanners, rate limiting, and visibility into being probed.

So I'd frame it as a sensor and a speed bump rather than a control. And I'd run it in detection mode first, because blocking mode without tuning generates false positives that break legitimate traffic, and teams respond by disabling it entirely — which is worse than never having deployed it.
</details>

<details>
<summary><b>Q4. How would you defend against DDoS?</b></summary>

It depends on the layer, and they need different answers.

Volumetric attacks saturate your bandwidth, often via amplification — a small spoofed query to a DNS or NTP or memcached server produces a huge reply directed at the victim, with amplification factors reaching tens of thousands. You cannot absorb that on your own capacity. You need an anycast CDN or a scrubbing provider. It's a capacity problem, not a code problem.

Protocol attacks exhaust connection state — SYN floods, or Slowloris holding connections open by sending headers one byte at a time. SYN cookies, per-source connection limits, aggressive timeouts, and a buffering reverse proxy handle these.

Application-layer attacks are the hard case: requests that look legitimate but are expensive, like search queries or report generation. Volume is low so bandwidth defenses don't see it. Defenses are rate limiting, cost-based limits that charge more for expensive endpoints, caching, and behavioural bot detection.

The most commonly missed preparation is hiding the origin IP. If attackers can reach your origin directly, the CDN is irrelevant — so the origin must be firewalled to accept only CDN ranges.

And I'd cap autoscaling, because otherwise a DDoS converts an outage into a very large bill. That's sometimes called economic denial of sustainability, and it's a real outcome.
</details>

<details>
<summary><b>Q5. What are the most impactful cloud misconfigurations?</b></summary>

Public storage buckets remain the most publicized, and the fix is Block Public Access at the account level rather than per bucket, so a policy mistake can't expose anything.

Over-privileged IAM is the most pervasive. Wildcards in policies, and roles that accumulate permissions over time and never lose them. Access Analyzer generating least-privilege policies from actual CloudTrail usage is the practical remedy.

Exposed management ports — SSH, RDP, database ports open to the world — are still surprisingly common. The modern answer is not a hardened bastion but eliminating SSH access entirely in favour of something like SSM Session Manager.

Long-lived access keys deserve specific mention, because a key committed to a public repository is the single most common cloud compromise. Roles and OIDC federation eliminate the credential entirely.

And the one that determines whether you can investigate anything: logging that's disabled or deletable. CloudTrail should be organization-wide, delivered to a separate logging account that production credentials cannot write to, with Object Lock for immutability. An attacker's first move is often to disable logging, so "the logs went quiet" should itself be a high-severity alert.

The unifying point is that the failure is almost never the cloud provider's infrastructure — it's customer-side configuration.
</details>

<details>
<summary><b>Q6. How do you secure a Kubernetes cluster?</b></summary>

In layers, because an attacker moves upward through them.

Image layer: minimal base, pinned by digest rather than tag, scanned in CI with an actual patch SLA, signed and verified at admission, and no secrets baked into layers.

Runtime: non-root, read-only root filesystem, all capabilities dropped, seccomp enabled, and never privileged. Pod Security Admission enforcing the restricted profile.

Orchestrator: RBAC with least privilege, never binding cluster-admin to a service account, and `automountServiceAccountToken` disabled for pods that don't call the API. Audited with `kubectl auth can-i --list`.

Network: default-deny NetworkPolicy per namespace, because by default every pod can reach every pod — which means a compromised frontend reaches your database directly. Plus mTLS between services and egress restrictions.

Host: a patched kernel above all, because container isolation *is* the kernel — a kernel exploit escapes by definition.

Two specifics I'd call out. Kubernetes secrets are base64-encoded, not encrypted, so etcd encryption at rest is necessary, and workload identity is better than storing secrets at all. And etcd *is* the cluster — encrypt it, restrict access, back it up.

Finally, admission policies with Kyverno or Gatekeeper to enforce all of this automatically. Relying on code review to catch a privileged pod spec doesn't scale.
</details>

<details>
<summary><b>Q7. What would you monitor to detect a breach?</b></summary>

I'd prioritize by signal-to-noise, because the failure mode of detection programs is drowning in alerts nobody reads.

The highest-value source is cloud audit logs — who did what, when, from where. Then authentication logs, especially failures and impossible travel. Then authorization failures, which are a strong signal because normal users rarely trigger them.

DNS queries are underrated: they reveal command-and-control domains and exfiltration destinations that other telemetry misses. VPC flow logs surface unexpected internal connections, which is how you detect lateral movement.

For specific detections I'd build first: impossible travel, new IAM users or access keys, any policy granting wildcard permissions, security groups opened to the world, unusual egress volume, access to the metadata endpoint from a container, and outbound connections to newly registered domains.

And critically, CloudTrail or logging being disabled — that's a near-certain sign of an active intrusion rather than a configuration accident.

Which leads to the most important architectural point: protect the logs themselves. Ship them to a separate account that production credentials cannot write to or delete from, with Object Lock. And alert if logging stops, because silence is itself the detection.

The context worth stating is that median dwell time between compromise and detection has historically been weeks. That window is where lateral movement and exfiltration happen, so compressing it is where detection investment pays off.
</details>

<details>
<summary><b>Q8. How do attackers move laterally, and how do you stop it?</b></summary>

Lateral movement is what turns a modest initial compromise into a breach, and it's the step where defenses have the most leverage.

The mechanisms: stolen or reused credentials, especially service accounts with broad access; trust relationships between systems, like a CI system with production credentials; flat internal networks where everything can reach everything; and cached credentials on compromised hosts.

Defenses, roughly by impact. Microsegmentation with default-deny between workloads, so a compromised frontend can't reach the database directly or scan the internal network. Eliminating standing privileges through just-in-time elevation, so phishing a developer yields a developer's access rather than production. Workload identity with short-lived credentials, so there's no static secret to harvest from a compromised host. And separate accounts or trust domains for production, CI/CD, and management, so compromising the application tier doesn't reach the deployment pipeline.

For detection: unexpected internal connections in flow logs, authentication from unusual sources, service accounts authenticating from new locations, and authorization failures clustering.

The framing I'd emphasize is that most breaches follow the same chain — initial access, persistence, escalation, lateral movement, exfiltration. You don't need to block every step. Breaking any link stops the chain, which is what makes layered defenses valuable even when each layer is individually imperfect.
</details>

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              NETWORK & INFRA SECURITY — ONE PAGE                     ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ THE PERIMETER IS DEAD. "Inside the network" ≠ trustworthy.         ║
║   Most breaches = modest initial access + unimpeded LATERAL MOVEMENT ║
╠══════════════════════════════════════════════════════════════════════╣
║ ZERO TRUST: verify explicitly · least privilege · ⭐ ASSUME BREACH    ║
║   network location grants NOTHING; identity is the control plane     ║
║   first steps: phishing-resistant MFA · kill standing admin ·        ║
║   segment crown jewels · mTLS between services                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ SEGMENTATION limits BLAST RADIUS (not initial access)                ║
║   ⭐ microsegmentation: policy follows the WORKLOAD, not the IP       ║
║   ⭐⭐ EGRESS FILTERING is the most neglected high-value control       ║
║     C2 · exfiltration · payload download · SSRF→metadata all need it ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ WAF = speed bump + SENSOR, not a fix. Best use: VIRTUAL PATCHING.  ║
║   ⭐ It CANNOT see authorization bugs (BOLA) at all.                  ║
║   Run in DETECTION mode first or false positives get it disabled.    ║
╠══════════════════════════════════════════════════════════════════════╣
║ DDoS: volumetric(⭐ need a CDN/scrubber — capacity problem) ·         ║
║   protocol(SYN cookies, timeouts) · L7(⭐ hard: cost-based limits)    ║
║   ⚠️ HIDE THE ORIGIN IP or the CDN is pointless                       ║
║   ⚠️ cap autoscaling or DDoS = a huge bill instead of an outage       ║
╠══════════════════════════════════════════════════════════════════════╣
║ CLOUD MISCONFIGS: public buckets · over-privileged IAM · exposed     ║
║   SSH/RDP · ⚠️ long-lived access keys · disabled logging             ║
║ ⭐ 169.254.169.254 → IAM CREDENTIALS. ENFORCE IMDSv2, hop limit 1.    ║
║ ⭐ SEPARATE ACCOUNTS are a stronger boundary than any IAM policy      ║
╠══════════════════════════════════════════════════════════════════════╣
║ K8s: image→runtime→RBAC→network→host. ⚠️ secrets are BASE64 ·         ║
║   ⚠️ all pods reach all pods by default · ⚠️⚠️ docker.sock = host root ║
║   untrusted workloads → gVisor/Kata, containers are the wrong boundary║
╠══════════════════════════════════════════════════════════════════════╣
║ DETECT: cloud audit logs · auth failures · ⭐ authz failures ·        ║
║   DNS queries · flow logs · impossible travel · new IAM keys ·       ║
║   ⭐ LOGGING DISABLED (near-certain intrusion)                        ║
║ ⭐⭐ SHIP LOGS TO A SEPARATE ACCOUNT prod can't delete from            ║
║   (Object Lock). "The logs went quiet" IS a detection.               ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ ATTACK CHAIN: access→persist→escalate→LATERAL→exfil→impact         ║
║   You don't need to block every step — BREAKING ANY LINK STOPS IT.   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Offensive & Defensive →](offensive-defensive.md) · **Related:** [AppSec](appsec.md) · [AWS](../06-cloud-devops/aws.md) · [Kubernetes](../06-cloud-devops/kubernetes.md)
