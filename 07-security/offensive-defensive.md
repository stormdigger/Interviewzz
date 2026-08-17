# ⚔️ Offensive & Defensive Security

> Understanding how attacks work is what makes defenses meaningful rather than cargo-cult. This book covers both sides — methodology, detection, and response.

**Prerequisite:** [AppSec](appsec.md) · [Network Security](network-security.md)

> ⚠️ **Scope note:** everything here is framed for authorized testing, defensive work, and interview preparation. Offensive techniques are described at the conceptual level needed to defend against them. Testing systems you don't own or have written permission to test is a crime in most jurisdictions.

---

## 📑 Contents

1. [The Attacker's Mental Model](#1-the-attackers-mental-model)
2. [MITRE ATT&CK](#2-mitre-attck)
3. [Reconnaissance](#3-reconnaissance)
4. [Vulnerability Assessment vs Pentest vs Red Team](#4-assessment-types)
5. [The Pentest Methodology](#5-the-pentest-methodology)
6. [Web Application Testing](#6-web-application-testing)
7. [Blue Team — Detection Engineering](#7-blue-team--detection-engineering)
8. [Incident Response](#8-incident-response)
9. [Digital Forensics](#9-digital-forensics)
10. [Purple Teaming](#10-purple-teaming)
11. [Bug Bounty & Disclosure](#11-bug-bounty--disclosure)
12. [Building a Security Program](#12-building-a-security-program)
13. [Interview Section](#13-interview-section)
14. [Cheat Sheet](#14-cheat-sheet)

---

## 1. The Attacker's Mental Model

```
   ⭐ ATTACKERS DON'T THINK IN VULNERABILITIES. THEY THINK IN
     PATHS.

   A defender asks: "is this input validated?"
   An attacker asks: "how do I get from HERE to the data?"

   ⭐ THE CONSEQUENCE: a chain of three individually "low
     severity" issues is often a critical compromise, while an
     isolated "high severity" finding with no path to anything
     valuable may not matter at all.

   EXAMPLE OF A REAL CHAIN
     ① An outdated JS library → low severity XSS
     ② XSS steals a session token → medium
     ③ That user is a support agent with impersonation rights
     ④ Impersonate an admin → export the customer database

     ⭐ Each step is unremarkable. The CHAIN is a breach.
```

```
   ⭐ THE ECONOMICS FRAMING

   Attackers optimize for cost per outcome. Your goal is
   usually not to be impregnable — it's to be expensive enough
   that they move on, and to detect them fast enough that the
   attempt fails.

   ⭐ THIS IS WHY LAYERED DEFENSE WORKS. Each layer adds cost
     and adds a detection opportunity. You don't need every
     layer to be perfect.
```

---

## 2. MITRE ATT&CK

```
   ⭐ A CATALOGUE OF REAL, OBSERVED ADVERSARY BEHAVIOUR,
     organized by tactic (the goal) and technique (the method).

   ┌──────────────────────────────────────────────────────────────┐
   │ Reconnaissance        gathering information                  │
   │ Resource Development  building infrastructure                │
   │ Initial Access        ⭐ phishing · exploit public app ·      │
   │                       valid accounts · supply chain          │
   │ Execution             running code                           │
   │ Persistence           ⭐ surviving reboots and remediation    │
   │ Privilege Escalation  gaining higher permissions             │
   │ Defense Evasion       ⭐ avoiding detection                   │
   │ Credential Access     dumping/stealing credentials           │
   │ Discovery             mapping the environment                │
   │ Lateral Movement      ⭐ moving to other systems              │
   │ Collection            staging the data                       │
   │ Command & Control     maintaining a channel                  │
   │ Exfiltration          getting the data out                   │
   │ Impact                ransomware · destruction · extortion   │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHY IT'S ACTUALLY USEFUL — a common shared language

   1. ⭐ COVERAGE MAPPING: for each technique relevant to your
      environment, ask "would we detect this?" That turns
      "are we secure?" — unanswerable — into a concrete gap list.
   2. Threat intelligence uses ATT&CK IDs, so you can translate
      "this group uses T1078" directly into a detection question.
   3. Purple team exercises are scoped by technique.
   4. It reframes detection around BEHAVIOUR rather than
      indicators. IPs and hashes change constantly; the
      technique of "dump credentials from LSASS" persists.

   ⭐ THE DETECTION MATURITY LADDER (the Pyramid of Pain)
     hash values      → trivially changed by the attacker
     IP addresses     → easily changed
     domain names     → cheap to change
     network artifacts→ annoying to change
     tools            → costly to change
     ⭐ TTPs           → genuinely hard to change — this is where
                        detection investment pays off
```

---

## 3. Reconnaissance

```
   ⭐ PASSIVE — no interaction with the target at all

   • Certificate Transparency logs → ⭐ every subdomain for which
     a certificate was ever issued, including internal-sounding
     ones nobody meant to expose
   • DNS records, historical DNS data
   • Search engines and code search (⭐ GitHub for leaked
     credentials, internal hostnames, and API keys)
   • Job postings → ⭐ your exact tech stack, described in detail
   • LinkedIn → org structure, naming conventions, who's new
     (new employees are prime phishing targets)
   • Breach databases → credentials for reuse
   • Shodan / Censys → internet-exposed services by fingerprint

   ⭐ THE DEFENSIVE INSIGHT: your attack surface is far larger
     than your asset inventory. Certificate Transparency alone
     routinely reveals forgotten staging environments,
     abandoned subdomains, and internal tools that were never
     supposed to be reachable.
```

```
   ACTIVE — touching the target (⚠️ authorization required)

   • Port and service scanning
   • Web crawling and directory enumeration
   • Technology fingerprinting
   • API endpoint discovery from JS bundles
     (⭐ frontend bundles frequently reveal internal API routes,
      feature flags, and occasionally credentials)
```

```
   ⭐ DEFENSIVE ACTIONS THAT ACTUALLY REDUCE SURFACE

   □ ⭐ Monitor Certificate Transparency for your domains —
     free, and it catches both mis-issuance and forgotten hosts
   □ Continuous external attack surface discovery
   □ ⭐ Decommission aggressively. Abandoned subdomains pointing
     at deprovisioned cloud resources enable SUBDOMAIN TAKEOVER.
   □ Secret scanning across public repos, including personal
     accounts of employees
   □ Review what job postings and public docs disclose
   □ ⚠️ Assume all of this is already known to an attacker —
     it costs them almost nothing
```

---

## 4. Assessment Types

```
   ┌──────────────────────────────────────────────────────────────┐
   │ VULNERABILITY SCAN                                           │
   │   Automated, broad, shallow. Finds known CVEs and            │
   │   misconfigurations.                                         │
   │   ✅ Cheap, continuous, good coverage                         │
   │   ⚠️ False positives; ⭐ finds NO logic or authorization flaws │
   ├──────────────────────────────────────────────────────────────┤
   │ PENETRATION TEST                                             │
   │   Human-driven, scoped, time-boxed. Attempts to EXPLOIT      │
   │   and to CHAIN findings.                                     │
   │   ✅ Finds logic flaws and realistic attack paths             │
   │   ⚠️ Point-in-time; scope-limited                             │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ RED TEAM                                                   │
   │   Objective-based, adversarial, stealthy, usually            │
   │   unannounced to the defenders.                              │
   │   ⭐ Tests DETECTION AND RESPONSE, not just vulnerabilities.  │
   │     "Can we get to the crown jewels without being caught?"   │
   │   ⚠️ Expensive; only valuable once basic hygiene exists —     │
   │     a red team against an immature program just confirms     │
   │     what a scanner would have told you                       │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ PURPLE TEAM                                                │
   │   Red and blue working TOGETHER, openly. Execute a           │
   │   technique, check whether it was detected, tune, repeat.    │
   │   ⭐ Usually the highest learning-per-dollar of any of these. │
   ├──────────────────────────────────────────────────────────────┤
   │ BUG BOUNTY                                                   │
   │   Continuous, crowd-sourced, pay-per-finding.                │
   │   ✅ Broad coverage, real-world creativity, pay only for      │
   │     results                                                  │
   │   ⚠️ Needs triage capacity and a functioning remediation      │
   │     pipeline first — otherwise you accumulate unfixed reports│
   └──────────────────────────────────────────────────────────────┘

   ⭐ SEQUENCING MATTERS
     Scanning → pentest → purple team → bug bounty → red team.
     Running a red team before you can patch reliably wastes
     money and demoralizes everyone.
```

---

## 5. The Pentest Methodology

```
   ⭐ ① SCOPING AND AUTHORIZATION — never skip this

   □ ⭐ WRITTEN authorization from someone who can grant it
     ("get out of jail free" letter)
   □ Explicit in-scope and out-of-scope targets
   □ Testing window and rate limits
   □ Emergency contacts on both sides
   □ Rules of engagement: is social engineering allowed?
     DoS testing? Data exfiltration (usually simulated)?
   □ Data handling: what happens to anything sensitive found
   □ ⚠️ Cloud provider notification if required

   ⭐ Without written authorization, this is a crime rather
     than a service. This is not a formality.
```

```
   ② RECONNAISSANCE      passive, then active mapping
   ③ VULNERABILITY ID    automated scanning + manual review
   ④ EXPLOITATION        ⭐ prove impact, don't just theorize it
   ⑤ POST-EXPLOITATION   ⭐ what does this ACCESS actually enable?
                         escalate, pivot, demonstrate reach
   ⑥ ⭐ REPORTING         the actual deliverable
   ⑦ RETEST              verify the fixes work
```

```
   ⭐ THE REPORT IS THE PRODUCT

   A findings dump is not a pentest report. What makes it
   useful:

   ┌──────────────────────────────────────────────────────────────┐
   │ EXECUTIVE SUMMARY   ⭐ business risk in plain language, for   │
   │                     people who will fund the fixes           │
   │ METHODOLOGY         what was and wasn't tested — ⭐ so a      │
   │                     clean report isn't misread as "secure"   │
   │ FINDINGS, each with:                                         │
   │   • ⭐ severity based on REAL impact in THIS environment,     │
   │     not a generic CVSS score                                 │
   │   • reproduction steps a developer can actually follow       │
   │   • evidence                                                 │
   │   • ⭐ SPECIFIC remediation, not "sanitize input"             │
   │   • ⭐ ATTACK CHAINS — how findings combine. This is where    │
   │     the real value is, and where automation cannot compete.  │
   │ STRATEGIC RECOMMENDATIONS  ⭐ the systemic causes, not just   │
   │                     the instances                            │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE BEST FINDING IN ANY REPORT IS USUALLY NOT A BUG —
     it's "you have no way to detect any of this," or "these
     twelve findings all stem from one missing control."
```

---

## 6. Web Application Testing

```
   ⭐ THE MANUAL TESTING CHECKLIST — what scanners miss

   AUTHENTICATION
   □ User enumeration via responses OR ⭐ timing
   □ Password reset: token predictability, expiry, ⭐ whether
     it invalidates existing sessions, host header poisoning
   □ MFA bypass: skip the step, race conditions, ⭐ backup codes
   □ Session fixation — is the ID regenerated on login?
   □ Concurrent session handling
   □ ⭐ Does logout actually invalidate server-side?

   ⭐ AUTHORIZATION — the highest-yield area
   □ ⭐ BOLA/IDOR: change every ID and see what happens
   □ Horizontal: access another user's resources
   □ Vertical: access admin functions as a normal user
   □ ⭐ Forced browsing to endpoints not linked in the UI
   □ ⭐ Mass assignment: add role/isAdmin/credits to a request
   □ Does the API enforce what the UI hides?
   □ ⭐ Multi-tenant: can tenant A reach tenant B's data?

   INPUT HANDLING
   □ Injection in every parameter, header, and cookie
   □ ⭐ Second-order injection: stored now, executed later
     elsewhere (scanners almost never find these)
   □ File upload: content vs extension, path traversal
   □ SSRF in any URL-accepting field
   □ XXE in XML parsers
   □ Deserialization

   BUSINESS LOGIC  ⭐ scanners find NONE of these
   □ ⭐ Negative quantities, integer overflow in prices
   □ ⭐ Race conditions: apply a coupon twice concurrently
   □ Skipping steps in a multi-step flow
   □ Replaying or tampering with a payment callback
   □ Currency and rounding manipulation
   □ Abusing rate limits or quotas

   ⭐ BUSINESS LOGIC AND AUTHORIZATION ARE WHERE HUMAN TESTING
     EARNS ITS COST. Everything else is increasingly automated.
```

```
   ⚠️ RACE CONDITIONS ARE UNDER-TESTED AND HIGH-IMPACT

   Send N concurrent requests to a state-changing endpoint:
     • redeem a single-use coupon many times
     • withdraw funds twice before the balance updates
     • ⭐ accept the same invitation repeatedly to gain
       multiple permission grants

   The root cause is always a check-then-act sequence without
   atomicity. ⭐ The fix is a database-level constraint or a
   conditional update — the same pattern as preventing
   double-booking in [Airbnb](../05-system-design/04-case-studies-2.md#4-deep-dive--preventing-double-booking).
```

---

## 7. Blue Team — Detection Engineering

```
   ⭐ DETECTION AS A SOFTWARE DISCIPLINE

   ┌──────────────────────────────────────────────────────────────┐
   │ ① HYPOTHESIZE  "an attacker would do X" (from ATT&CK, from   │
   │                threat intel, or from a real incident)        │
   │ ② IDENTIFY     what telemetry would reveal X?                │
   │ ③ WRITE        the detection logic                           │
   │ ④ ⭐ TEST       ACTUALLY EXECUTE the technique and confirm    │
   │                the detection fires (this is purple teaming)  │
   │ ⑤ TUNE         reduce false positives to a sustainable rate  │
   │ ⑥ ⭐ DOCUMENT   a runbook — a detection with no response      │
   │                procedure is just an alarm                    │
   │ ⑦ MAINTAIN     ⚠️ detections ROT as the environment changes   │
   └──────────────────────────────────────────────────────────────┘

   ⭐ Detections belong in version control, get code review, and
     ideally have automated tests. Treating them as ad-hoc SIEM
     queries is why detection programs decay.
```

```
   ⭐ WHAT MAKES A GOOD DETECTION

   HIGH SIGNAL      Rare in normal operation. ⚠️ A rule that
                    fires 500 times a day will be muted, and
                    then it protects nothing.
   ACTIONABLE       There's a defined response
   ⭐ BEHAVIOURAL    Targets the TECHNIQUE, not an indicator.
                    Hashes and IPs change hourly; "credentials
                    dumped from process memory" persists.
   RESILIENT        Not trivially evaded by a minor variation
   DOCUMENTED       What it means, what to do, who owns it
```

```
   ⭐ DETECTION TIERS BY DIFFICULTY OF EVASION

   ┌──────────────────────────────────────────────────────────────┐
   │ EASY TO EVADE                                                │
   │   known-bad hashes · known-bad IPs · specific filenames      │
   ├──────────────────────────────────────────────────────────────┤
   │ MODERATE                                                     │
   │   command-line patterns · registry keys · specific tools     │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ HARD TO EVADE — invest here                                │
   │   • a process spawning an unexpected child                   │
   │   • authentication from an impossible location               │
   │   • ⭐ data volume anomalies on egress                        │
   │   • a service account behaving like a human                  │
   │   • lateral movement patterns across hosts                   │
   │   • ⭐ any deviation from a well-established baseline          │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ BEHAVIOURAL DETECTION REQUIRES A BASELINE, which requires
     time and clean data. Start collecting before you need it.
```

---

## 8. Incident Response

```
   ⭐ THE SIX PHASES (NIST)

   ┌──────────────────────────────────────────────────────────────┐
   │ ① PREPARATION    ⭐ the phase that determines everything else │
   │    plan · roles · tooling · access · runbooks · retainers ·  │
   │    ⭐ EXERCISES                                               │
   ├──────────────────────────────────────────────────────────────┤
   │ ② DETECTION & ANALYSIS                                       │
   │    Is it real? What's the scope? What's the impact?          │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ CONTAINMENT    stop the bleeding                           │
   │    ⭐ SHORT-TERM: isolate the host, disable the account,      │
   │      block the C2 domain                                     │
   │    LONG-TERM: patch, rotate credentials, rebuild             │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ ERADICATION    remove the attacker's access completely     │
   ├──────────────────────────────────────────────────────────────┤
   │ ⑤ RECOVERY       restore service, ⭐ monitor intensely for    │
   │                  reinfection                                 │
   ├──────────────────────────────────────────────────────────────┤
   │ ⑥ LESSONS LEARNED  ⭐ blameless postmortem with tracked       │
   │                  action items                                │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️⚠️ THE CONTAINMENT DILEMMA — the hardest real-time decision

   CONTAIN IMMEDIATELY          OBSERVE FIRST
   ✅ Stops further damage       ✅ Learn the full scope and all
   ✅ Simple to justify            footholds before acting
   ⚠️ Tips off the attacker      ✅ Identify every backdoor
   ⚠️ ⭐ They may have OTHER      ⚠️ Damage continues meanwhile
      footholds you haven't      ⚠️ Legally and ethically fraught
      found — you contain one       if data is actively being
      and they persist through      exfiltrated
      the rest

   ⭐ THE GENERAL GUIDANCE
     • Active data exfiltration or ransomware deployment →
       CONTAIN NOW, without hesitation
     • Quiet reconnaissance, and you have good visibility →
       a brief observation window may be worth it
     • ⭐ When you do contain, contain EVERYTHING SIMULTANEOUSLY.
       Partial containment is the worst outcome — it alerts the
       attacker while leaving them access.
```

```
   ⭐ THE PRACTICAL FIRST HOUR

   □ ⭐ Declare an incident and assign an INCIDENT COMMANDER
     (who coordinates and does NOT debug)
   □ Open a dedicated channel; ⚠️ consider out-of-band comms if
     the corporate systems may be compromised
   □ ⭐ Start a timestamped timeline immediately — it cannot be
     reconstructed later
   □ ⭐ PRESERVE EVIDENCE BEFORE remediating. Memory first,
     then disk. Rebooting destroys memory-resident artifacts.
   □ Determine scope: what accounts, hosts, and data?
   □ Assess legal and regulatory obligations early (⭐ GDPR is
     72 hours from awareness)
   □ Decide on containment
   □ ⚠️ Rotate credentials that may be compromised — including
     the ones you use to investigate
```

```
   ⚠️ MISTAKES THAT MAKE INCIDENTS WORSE

   • ⭐ Destroying evidence by rebooting or reimaging first
   • Partial containment that alerts without stopping
   • Investigating with potentially-compromised credentials
   • Discussing the incident on potentially-compromised systems
   • ⚠️ Declaring victory too early — attackers frequently
     maintain multiple footholds specifically to survive
     remediation
   • No timeline, so the postmortem is guesswork
   • Not involving legal and communications until too late
```

---

## 9. Digital Forensics

```
   ⭐ ORDER OF VOLATILITY — collect most-volatile first

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. CPU registers, cache                    seconds           │
   │ 2. ⭐ RAM — running processes, network       minutes           │
   │    connections, encryption keys, injected                    │
   │    code that never touched disk                              │
   │ 3. Network state, ARP, routing tables       minutes          │
   │ 4. Running processes and open files         minutes          │
   │ 5. Disk                                     persists         │
   │ 6. Remote logs, backups                     persists         │
   │ 7. Physical configuration, archives         long-term        │
   └──────────────────────────────────────────────────────────────┘

   ⚠️⚠️ REBOOTING DESTROYS RAM — and modern malware is often
     memory-resident specifically to avoid disk forensics.
     ⭐ Capture memory BEFORE anything else, including before
     "just restarting it to see if that fixes it."
```

```
   ⭐ CHAIN OF CUSTODY — required if this may ever be legal
     evidence

   □ Who collected it, when, and using what method
   □ Cryptographic hash at collection, verified at each transfer
   □ ⭐ Work on COPIES; the original is never touched
   □ Documented handoffs
   □ Secure storage with access logging

   ⭐ Even if litigation seems unlikely, following this costs
     little and preserves the option. Deciding retroactively
     that evidence should have been preserved properly is not
     possible.
```

```
   KEY ARTIFACT SOURCES
     Linux    auth.log · bash_history · /tmp · cron · systemd
              units · SSH authorized_keys · ⭐ package manager logs
     Windows  Event Logs · registry · prefetch · ⭐ Amcache ·
              scheduled tasks · WMI subscriptions
     Cloud    ⭐ CloudTrail · flow logs · config history ·
              access logs
     Network  full packet capture (if available) · NetFlow · DNS
     App      application logs · database audit logs
```

---

## 10. Purple Teaming

```
   ⭐ THE HIGHEST LEARNING-PER-DOLLAR SECURITY ACTIVITY.

   THE LOOP
   ┌──────────────────────────────────────────────────────────────┐
   │ 1. Pick a technique (ATT&CK ID) relevant to your threat model│
   │ 2. RED executes it, openly, in a controlled way              │
   │ 3. BLUE checks: did we log it? did we alert? would we have   │
   │    noticed?                                                  │
   │ 4. ⭐ If not — is the telemetry missing, or just the rule?    │
   │ 5. Build or tune the detection                               │
   │ 6. ⭐ RE-RUN to verify it now fires                           │
   │ 7. Document and move to the next technique                   │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY IT BEATS A TRADITIONAL RED TEAM FOR MOST ORGANIZATIONS
     A red team produces a report saying "we got in and you
     didn't notice." Purple teaming produces WORKING DETECTIONS,
     verified by execution, plus a measurable coverage map —
     and it builds the blue team's skills in the process.

   ⭐ Tools: Atomic Red Team (small, safe, per-technique tests),
     Caldera, Prelude — all let you run this without a
     specialist offensive team.
```

---

## 11. Bug Bounty & Disclosure

```
   ⭐ PREREQUISITES BEFORE LAUNCHING A BOUNTY

   ⚠️ A bug bounty without these produces a backlog of unfixed
     reports and frustrated researchers who eventually publish.

   □ ⭐ Basic hygiene done — scanning, pentests, obvious issues
     fixed. Otherwise you pay bounty rates for findings a
     scanner would have given you free.
   □ Triage capacity — someone who responds within days
   □ ⭐ A functioning remediation pipeline. Receiving reports
     you can't fix is worse than not receiving them.
   □ A clear scope and clear rules
   □ ⭐ SAFE HARBOR language — legal protection for good-faith
     research. Without it, serious researchers won't participate.
   □ Reward table set against real severity
```

```
   ⭐ EVEN WITHOUT A BOUNTY, HAVE A DISCLOSURE PATH

   /.well-known/security.txt
     Contact: security@example.com
     Policy: https://example.com/security-policy
     Preferred-Languages: en

   ⚠️ Researchers WILL find issues. The only question is whether
     they can reach you easily, or whether frustration leads to
     public disclosure. A security.txt file costs nothing.

   ⭐ AND: never threaten a good-faith researcher. It's
     counterproductive, it becomes public, and it guarantees
     the next finder publishes instead of reporting.
```

---

## 12. Building a Security Program

```
   ⭐ MATURITY STAGES — in the order that actually works

   ┌──────────────────────────────────────────────────────────────┐
   │ ① FOUNDATIONS  ⭐ do these before anything sophisticated      │
   │    • Asset inventory — ⭐ you cannot protect what you don't   │
   │      know exists, and this is always harder than expected    │
   │    • MFA everywhere, phishing-resistant for privileged users │
   │    • Patch management with a tracked SLA                     │
   │    • Backups with ⭐ TESTED restores and immutable copies     │
   │    • Centralized logging                                     │
   │    • ⭐ An incident response plan that has been EXERCISED     │
   ├──────────────────────────────────────────────────────────────┤
   │ ② HARDENING                                                  │
   │    Least privilege · segmentation · secure SDLC ·            │
   │    vulnerability management · security training              │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ DETECTION                                                  │
   │    SIEM · detection engineering · ⭐ purple teaming ·         │
   │    threat intelligence                                       │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ MATURITY                                                   │
   │    Threat hunting · red team · bug bounty · automation ·     │
   │    metrics that drive decisions                              │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ THE COMMON MISTAKE: buying a sophisticated detection
     platform while lacking an asset inventory, MFA, and tested
     backups. Skipping stage one to reach stage three produces
     expensive alerts about assets you can't identify.
```

```
   ⭐ METRICS THAT ACTUALLY DRIVE BEHAVIOUR

   ✅ Mean time to DETECT · mean time to RESPOND
   ✅ ⭐ Patch latency for critical vulnerabilities (p50 and p95)
   ✅ Percentage of assets with EDR/logging coverage
   ✅ ⭐ ATT&CK technique coverage — measured by purple team,
     not by vendor claims
   ✅ Phishing simulation click AND report rates
     (⭐ the REPORT rate matters more than the click rate —
      it measures whether people help you)
   ✅ Percentage of privileged access that is just-in-time

   ⚠️ VANITY METRICS
     Number of alerts · number of vulnerabilities found ·
     training completion percentage · number of tools deployed
     — none of these correlate with being harder to compromise.
```

---

## 13. Interview Section

<details>
<summary><b>Q1. How do attackers actually think, and why does that matter?</b></summary>

They think in paths, not vulnerabilities. A defender asks whether an input is validated; an attacker asks how to get from here to the data.

That reframing has real consequences. A chain of three individually low-severity issues is frequently a critical compromise, while an isolated high-severity finding with no path to anything valuable may not matter much.

A concrete example: an outdated JavaScript library gives you a low-severity XSS. The XSS steals a session token — medium. That user happens to be a support agent with impersonation rights. Now you impersonate an admin and export the customer database. Every individual step is unremarkable; the chain is a breach.

The second framing is economic. Attackers optimize for cost per outcome. Your goal usually isn't to be impregnable — it's to be expensive enough that they move on, and to detect them fast enough that the attempt fails.

That's precisely why layered defense works even when each layer is imperfect. Each one adds cost and adds a detection opportunity.
</details>

<details>
<summary><b>Q2. Pentest vs red team vs purple team — when do you use each?</b></summary>

A penetration test is scoped and time-boxed, with the goal of finding and demonstrating vulnerabilities, including chains an automated scanner would miss. The defenders usually know it's happening.

A red team is objective-based and adversarial — "can we reach the crown jewels without being caught?" — typically unannounced. Crucially, it tests detection and response, not just vulnerability existence.

A purple team is red and blue working together openly: execute a technique, check whether it was detected, tune, re-run to verify.

The sequencing matters more than the definitions. Scanning first, then pentesting, then purple teaming, then bug bounty, and red team last. Running a red team against an immature program just confirms what a scanner would have told you, at ten times the cost and with demoralizing results.

For most organizations, purple teaming is the highest learning-per-dollar activity. A red team produces a report saying "we got in and you didn't notice." Purple teaming produces working detections, verified by actual execution, plus a measurable coverage map — and it builds the defenders' skills rather than just testing them.
</details>

<details>
<summary><b>Q3. What is MITRE ATT&CK and how do you use it?</b></summary>

A catalogue of observed adversary behaviour, organized by tactic — the goal — and technique — the method. It runs from reconnaissance through initial access, persistence, privilege escalation, lateral movement, exfiltration, and impact.

Its practical value is as a coverage map. For each technique relevant to your environment, you ask "would we detect this?" That converts "are we secure?" — which is unanswerable — into a concrete, prioritized gap list.

It also gives a shared language. Threat intelligence reports reference technique IDs, so "this group uses valid accounts for initial access" translates directly into a detection question rather than staying abstract.

The deeper reason it matters is the Pyramid of Pain. Detections built on hashes and IP addresses are trivially evaded — attackers change those hourly. Detections built on techniques are genuinely hard to evade, because the technique is what the attacker actually needs to accomplish.

So ATT&CK pushes detection investment toward behaviour rather than indicators, which is where it pays off.
</details>

<details>
<summary><b>Q4. Walk me through responding to a suspected breach.</b></summary>

Declare an incident and name an incident commander who coordinates and does not debug. Open a dedicated channel — and consider out-of-band communication if corporate systems may be compromised, because attackers read incident channels.

Start a timestamped timeline immediately. It cannot be reconstructed afterward and it's the foundation of the postmortem.

Then a critical ordering decision: preserve evidence before remediating. Memory first, then disk, because modern malware is often memory-resident specifically to avoid disk forensics, and rebooting destroys it. "Let's just restart it" is the single most common way evidence is lost.

Determine scope — which accounts, hosts, and data — and assess legal obligations early, since GDPR is 72 hours from awareness.

Then containment, which is the hardest real-time judgment. Containing immediately stops damage but tips off the attacker, who may have other footholds you haven't found. Observing longer reveals full scope but lets damage continue. My rule: active exfiltration or ransomware means contain now without hesitation. Quiet reconnaissance with good visibility may justify a brief observation window.

And when you do contain, contain everything simultaneously. Partial containment is the worst outcome — it alerts the attacker while leaving them access.

Then eradication, recovery with intensive monitoring for reinfection, and a blameless postmortem with tracked action items.

The mistake I'd flag most: declaring victory early. Attackers routinely maintain multiple footholds specifically to survive remediation.
</details>

<details>
<summary><b>Q5. What makes a good detection rule?</b></summary>

High signal above everything. A rule firing five hundred times a day will be muted, and a muted rule protects nothing. Alert fatigue is the primary failure mode of detection programs.

Behavioural rather than indicator-based. Hashes and IP addresses change hourly, so detections built on them are trivially evaded. "Credentials dumped from process memory" or "a service account authenticating from a new country" describes what the attacker needs to do, and that's much harder to change.

Actionable, with a documented response. A detection without a runbook is just an alarm.

And tested by actually executing the technique — which is purple teaming. An untested detection is a hypothesis, and detections rot as environments change, so this has to be ongoing.

The tiers worth investing in are things like a process spawning an unexpected child, impossible travel, data volume anomalies on egress, a service account behaving like a human, and lateral movement patterns. Those require a baseline, which requires collecting telemetry before you need it — so starting collection early matters even before you can analyze it.

I'd also treat detections as code: version controlled, code reviewed, ideally with automated tests. Ad-hoc SIEM queries are why detection programs decay.
</details>

<details>
<summary><b>Q6. Where does automated scanning fall short?</b></summary>

Two categories, and they happen to be the two highest-severity ones.

Authorization flaws. A scanner has no idea that user five shouldn't be able to read order nine — it doesn't know your data model or your permission semantics. Broken object level authorization is the number one cause of real API breaches and is essentially invisible to automation.

Business logic. Negative quantities, integer overflow in prices, race conditions where a coupon is applied twice concurrently, skipping steps in a multi-step flow, replaying a payment callback. None of these are malformed input; they're valid requests in an invalid sequence or combination.

Race conditions in particular are under-tested and high-impact. Send N concurrent requests to a state-changing endpoint and you frequently find single-use things being used multiple times. The root cause is always check-then-act without atomicity, and the fix is a database constraint or a conditional update.

Scanners are genuinely good at known CVEs, missing headers, and obvious injection — which is real value, cheaply and continuously.

So the right split is automation for breadth and continuity, humans for authorization and logic. And increasingly, automated authorization testing in CI — authenticate as user A, attempt to access user B's resources, assert failure — which catches the highest-severity class without a human.
</details>

<details>
<summary><b>Q7. How would you build a security program from scratch?</b></summary>

In the order that actually works, which is not the order that's most interesting.

Foundations first. An asset inventory, because you cannot protect what you don't know exists — and this is always harder and more revealing than people expect. MFA everywhere, phishing-resistant for privileged accounts. Patch management with a tracked SLA. Backups with tested restores and immutable copies, since ransomware deletes reachable backups first. Centralized logging. And an incident response plan that has actually been exercised.

Then hardening: least privilege, segmentation, secure SDLC practices, vulnerability management.

Then detection: SIEM, detection engineering, purple teaming.

Then maturity: threat hunting, red team, bug bounty.

The common mistake is buying a sophisticated detection platform while lacking an asset inventory and MFA. That produces expensive alerts about assets nobody can identify.

For metrics I'd track mean time to detect and respond, patch latency at p50 and p95, coverage percentage for logging and EDR, ATT&CK technique coverage measured by purple team rather than vendor claims, and phishing report rate — which matters more than click rate because it measures whether people help you.

And I'd explicitly avoid vanity metrics: alert counts, vulnerabilities found, training completion. None of them correlate with being harder to compromise.
</details>

<details>
<summary><b>Q8. Should we run a bug bounty?</b></summary>

Only with prerequisites in place, or it backfires.

You need basic hygiene done first — scanning and pentesting, obvious issues fixed. Otherwise you're paying bounty rates for findings a scanner would have given you free.

You need triage capacity, meaning someone who responds within days rather than months. And critically, a functioning remediation pipeline. Receiving reports you can't fix is genuinely worse than not receiving them, because you accumulate a backlog of known unfixed vulnerabilities and frustrated researchers who eventually publish.

You need clear scope, clear rules, and safe harbor language giving legal protection for good-faith research. Without safe harbor, serious researchers won't participate.

But regardless of whether you run a bounty, you should have a disclosure path — a security.txt file with a contact address costs nothing. Researchers will find issues whether or not you invited them. The only question is whether they can reach you easily.

And never threaten a good-faith researcher. It's counterproductive, it becomes public, and it guarantees the next person publishes rather than reporting.
</details>

---

## 14. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              OFFENSIVE & DEFENSIVE — ONE PAGE                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ ATTACKERS THINK IN PATHS, NOT VULNERABILITIES                      ║
║   3 "low" findings chained = critical. Isolated "high" with no       ║
║   path may not matter. ⭐ Economics: make it expensive + detectable.  ║
╠══════════════════════════════════════════════════════════════════════╣
║ ATT&CK = shared language + ⭐ COVERAGE MAP ("would we detect this?")  ║
║ ⭐ PYRAMID OF PAIN: hashes/IPs are trivially changed; TTPs are not.   ║
║   Invest detection in BEHAVIOUR, not indicators.                     ║
╠══════════════════════════════════════════════════════════════════════╣
║ RECON: ⭐ Certificate Transparency reveals forgotten subdomains ·     ║
║   job posts leak your stack · JS bundles leak internal APIs          ║
║   ⭐ your attack surface > your asset inventory                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ SEQUENCE: scan → pentest → ⭐ PURPLE → bounty → red team              ║
║   red team before basic hygiene just confirms what a scanner said    ║
║ ⭐ PURPLE TEAM = highest learning per dollar. Produces WORKING,       ║
║   VERIFIED detections instead of a report saying "we got in."        ║
║ ⭐ THE PENTEST REPORT IS THE PRODUCT — attack CHAINS + systemic causes║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ SCANNERS MISS: authorization (BOLA) and BUSINESS LOGIC — the two   ║
║   highest-severity classes. Race conditions especially under-tested. ║
║   (check-then-act without atomicity → DB constraint/conditional update)║
╠══════════════════════════════════════════════════════════════════════╣
║ INCIDENT: IC coordinates (doesn't debug) · timestamped timeline NOW  ║
║   ⭐⭐ PRESERVE MEMORY BEFORE REBOOTING (malware is memory-resident)   ║
║   ⚠️ containment dilemma: exfil/ransomware → contain NOW.             ║
║     ⭐ Contain EVERYTHING AT ONCE — partial containment is the worst. ║
║   ⚠️ don't declare victory early — attackers keep multiple footholds  ║
║   ⭐ GDPR = 72 hours from awareness                                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ FORENSICS: ⭐ order of volatility — RAM before disk. Chain of custody.║
║ DETECTION: high signal (⚠️ 500 alerts/day = muted = worthless) ·      ║
║   behavioural · runbook attached · ⭐ TESTED by execution · in git    ║
╠══════════════════════════════════════════════════════════════════════╣
║ PROGRAM ORDER: ⭐ inventory + MFA + patching + TESTED BACKUPS +       ║
║   logging + exercised IR plan  →  hardening → detection → maturity   ║
║   ⚠️ buying a SIEM before an asset inventory is the classic mistake   ║
║ METRICS: MTTD/MTTR · patch latency · coverage % · ⭐ phishing REPORT  ║
║   rate.  ⚠️ Vanity: alert counts, vulns found, training completion.   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [AppSec](appsec.md) · [Network Security](network-security.md) · [Cryptography](cryptography.md) · [Observability & SRE](../06-cloud-devops/observability-sre.md)
