# 🔭 Observability & SRE

> Monitoring tells you *that* something is wrong. Observability lets you ask *why* — including questions you didn't anticipate when you instrumented the system. SRE is the discipline of deciding how reliable is reliable enough, and spending accordingly.

**Prerequisite:** [Linux & Networking](linux-networking.md)

---

## 📑 Contents

1. [Monitoring vs Observability](#1-monitoring-vs-observability)
2. [The Three Pillars](#2-the-three-pillars)
3. [Metrics](#3-metrics)
4. [Logging](#4-logging)
5. [Distributed Tracing](#5-distributed-tracing)
6. [What to Measure](#6-what-to-measure)
7. [SLIs, SLOs, and Error Budgets](#7-slis-slos-and-error-budgets)
8. [Alerting](#8-alerting)
9. [Incident Response](#9-incident-response)
10. [Postmortems](#10-postmortems)
11. [Chaos Engineering](#11-chaos-engineering)
12. [Toil and Automation](#12-toil-and-automation)
13. [Interview Section](#13-interview-section)
14. [Cheat Sheet](#14-cheat-sheet)

---

## 1. Monitoring vs Observability

#### 💬 The distinction that actually matters

```
   MONITORING                      OBSERVABILITY
   ──────────                      ─────────────
   "Is CPU above 80%?"             "Why are checkout requests
   "Is the service up?"             from Android users in Brazil
   "Did errors exceed 1%?"          on the new app version slow?"

   ⭐ Answers questions you        ⭐ Answers questions you
     ANTICIPATED                     DIDN'T anticipate

   Dashboards you built            Ad-hoc exploration of
   in advance                      high-cardinality data
```

```
   ⭐ THE KEY CONCEPT: CARDINALITY

   Cardinality = the number of unique values a dimension can take.

   LOW cardinality:   status_code (~10 values)
                      environment (~3 values)
   HIGH cardinality:  user_id (millions)
                      request_id (unbounded)
                      trace_id (unbounded)

   ⚠️ Metrics systems EXPLODE on high cardinality — every unique
     label combination is a separate time series stored forever.
     Adding user_id as a Prometheus label will take down your
     Prometheus.

   ⭐ Observability requires high-cardinality data, which is why
     it lives in logs, traces, and wide events — not metrics.
     This is the single most important operational distinction
     between the pillars.
```

```
   ⭐ THE UNKNOWN-UNKNOWNS FRAMING

   Monitoring handles KNOWN failure modes: you predicted disk
   could fill, so you alert on disk usage.

   Modern distributed systems fail in ways nobody predicted —
   an interaction between three services under a specific
   traffic pattern at a specific time. You cannot pre-build a
   dashboard for a failure you've never imagined.

   → You need the ability to SLICE arbitrary dimensions after
     the fact. That's observability.
```

---

## 2. The Three Pillars

```
   ┌──────────────────────────────────────────────────────────────┐
   │ METRICS — numbers over time, pre-aggregated                  │
   │   "p99 latency is 800ms; error rate is 2%"                   │
   │   ✅ Cheap, always-on, ideal for alerting and trends          │
   │   ✅ Constant cost regardless of traffic volume               │
   │   ❌ ⚠️ Aggregation DESTROYS detail — you can't ask which      │
   │      requests were slow                                      │
   │   ❌ Low cardinality only                                     │
   ├──────────────────────────────────────────────────────────────┤
   │ LOGS — discrete events with full detail                      │
   │   "request abc123 failed: connection refused to payment-svc" │
   │   ✅ High cardinality, arbitrary context                      │
   │   ✅ The detail you need once you know where to look          │
   │   ❌ ⚠️ Expensive at volume — cost scales with traffic         │
   │   ❌ Hard to aggregate or compute trends from                 │
   ├──────────────────────────────────────────────────────────────┤
   │ TRACES — one request's full journey across services          │
   │   "gateway 5ms → auth 20ms → orders 400ms → db 380ms"        │
   │   ✅ ⭐ The ONLY thing that finds latency in a distributed     │
   │      system. Shows causality and where time actually went.   │
   │   ❌ Requires instrumentation and context propagation         │
   │   ❌ Usually sampled, so a specific request may be missing    │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE GLUE: correlation IDs propagated through EVERYTHING

   Client ──[trace_id: abc]──▶ Gateway ──▶ Service A ──▶ Service B
                                 │            │            │
                                 └────────────┴────────────┘
                              every log line tagged trace_id=abc
                              every metric exemplar links to it

   ⭐ THE WORKFLOW THIS ENABLES:
     Metric alert fires (p99 latency spike)
       → click through to an exemplar TRACE from that period
       → see which service and span consumed the time
       → jump to that service's LOGS filtered by trace_id
       → read the actual error

   Without correlation IDs, each pillar is an island and
   debugging is archaeology.
```

```
   ⭐ THE EMERGING FOURTH: WIDE STRUCTURED EVENTS
     Rather than three separate systems, emit ONE very wide
     event per request containing everything — duration, all
     the IDs, the user tier, the feature flags, the DB timings.
     High cardinality by design, queryable on any dimension.
     This is the model Honeycomb popularized, and it addresses
     the unknown-unknowns problem more directly than metrics
     plus logs plus traces bolted together.
```

---

## 3. Metrics

### The four metric types

```
   ┌──────────────────────────────────────────────────────────────┐
   │ COUNTER    ⭐ Monotonically increasing. Only goes up (or      │
   │            resets to 0 on restart).                          │
   │            http_requests_total, errors_total                 │
   │            ⚠️ NEVER graph a counter directly — always rate()  │
   ├──────────────────────────────────────────────────────────────┤
   │ GAUGE      Goes up and down. A point-in-time value.          │
   │            queue_depth, memory_bytes, active_connections     │
   ├──────────────────────────────────────────────────────────────┤
   │ HISTOGRAM  ⭐ Bucketed distribution. Enables percentiles.      │
   │            request_duration_seconds                          │
   │            ⭐ AGGREGATABLE across instances — this is why it's │
   │              the right choice for latency                    │
   ├──────────────────────────────────────────────────────────────┤
   │ SUMMARY    Client-side computed quantiles.                    │
   │            ⚠️ NOT aggregatable across instances. Avoid for     │
   │              anything you'll sum across a fleet.             │
   └──────────────────────────────────────────────────────────────┘
```

```promql
# ⭐ Request rate over 5 minutes
rate(http_requests_total[5m])

# ⭐ Error rate as a RATIO (the form SLOs need)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# ⭐ p99 latency from a histogram
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Saturation
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
  / sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod)

# ⭐ CPU throttling — the metric people miss entirely
rate(container_cpu_cfs_throttled_seconds_total[5m])
```

```
   ⚠️⚠️ AVERAGES LIE. USE PERCENTILES.

   Request latencies: 10ms ×99, and 5000ms ×1
     mean   = 60ms      ← "we're fine!"
     p99    = 5000ms    ← ⭐ one in a hundred users waits 5 seconds

   ⭐ AND PERCENTILES DON'T AVERAGE.
     avg(p99 across instances) is a MEANINGLESS NUMBER.
     You must aggregate the HISTOGRAM BUCKETS, then compute the
     quantile. That's exactly what histogram_quantile(sum(rate(
     ..._bucket))) does, and why histograms beat summaries.

   ⭐ AND: p99 of a single request path understates user pain.
     A page making 20 backend calls hits p99 on at least one
     of them ~18% of the time. ⭐ Measure the USER-FACING
     journey, not just individual services.
```

```
   ⚠️ CARDINALITY IS THE #1 WAY TO KILL A METRICS SYSTEM

   Each unique label combination = one time series, stored forever.

   labels: {method, status, endpoint}
     5 methods × 10 statuses × 50 endpoints = 2,500 series ✅

   ADD user_id:
     × 1,000,000 users = 2.5 BILLION series ⚠️⚠️

   ⭐ RULES
     • Never use unbounded values as labels: user_id, request_id,
       trace_id, full URL paths, error messages
     • ⭐ Normalize URL paths: /users/123 → /users/{id}
     • If you need per-user analysis, that belongs in
       logs/traces/wide events, NOT metrics
```

---

## 4. Logging

```
   ⭐ STRUCTURED LOGS, NOT PROSE

   ❌ log.info(f"User {uid} failed login from {ip}")
      → unparseable, ungreppable at scale, no fields to filter on

   ✅ log.info("login_failed", user_id=uid, ip=ip,
                reason="bad_password", attempt=3)
      → queryable: "all login_failed where attempt > 5 grouped by ip"
```

```python
import structlog

logger = structlog.get_logger()

logger.info("order_created",
    order_id=order.id,
    user_id=user.id,
    total_cents=order.total,
    item_count=len(order.items),
    payment_method="card",
    duration_ms=elapsed,
)
# → {"event":"order_created","order_id":"ord_123",...,
#    "trace_id":"abc","level":"info","timestamp":"..."}
```

```
   ⭐ LOG LEVELS — and what they should MEAN operationally

   ERROR  ⭐ Something a human must eventually look at.
          If your ERROR log has 10,000 entries/hour that nobody
          reads, they aren't errors — they're noise, and you've
          trained everyone to ignore the level that matters.
   WARN   Unexpected but handled. Retries, fallbacks, degradation.
   INFO   Significant business events. Keep the volume sane.
   DEBUG  Detailed flow. ⚠️ Off in production by default, but
          ⭐ toggleable at runtime per-service or per-request
   TRACE  Extremely verbose. Local development only.
```

```
   ⚠️ NEVER LOG THESE
     passwords · tokens · API keys · session IDs · full credit
     card numbers · government IDs · full request bodies that
     might contain any of the above

   ⭐ Redact at the LOGGING LIBRARY level, not by remembering
     at each call site. Configure a redaction list once:
       redact=["password", "token", "authorization", "*.secret"]
     Relying on discipline at every call site guarantees a leak.

   ⚠️ GDPR/PII: logs are data storage. Retention policies and
     deletion requests apply to them.
```

```
   ⭐ SAMPLING — how to control log cost without losing signal

   • Log 100% of ERRORS — always
   • Sample successful requests at 1-10%
   • ⭐ Log 100% for a specific user/trace when debugging
     (dynamic sampling triggered by a header or flag)
   • ⭐ TAIL-BASED SAMPLING: buffer, then decide AFTER the
     request completes — keep everything for slow or failed
     requests, sample the fast successful ones. Far better
     than head-based sampling, which decides before it knows
     whether the request was interesting.
```

---

## 5. Distributed Tracing

```
   ⭐ A TRACE IS A TREE OF SPANS

   Trace: abc123  (total 450ms)
   ┌────────────────────────────────────────────────────────┐
   │ gateway                                        450ms   │
   │  ├── auth-service                    20ms              │
   │  ├── order-service                          400ms      │
   │  │    ├── db: SELECT orders                  30ms      │
   │  │    ├── ⚠️ payment-service                 350ms  ⭐   │
   │  │    │    └── ⚠️ external: stripe API       340ms      │
   │  │    └── db: UPDATE orders                  15ms      │
   │  └── notification-service (async)     5ms              │
   └────────────────────────────────────────────────────────┘

   ⭐ INSTANTLY VISIBLE: the 450ms is 340ms of waiting on an
     external API. No amount of metrics or log-reading would
     have shown you that as directly.
```

```
   ⭐ CONTEXT PROPAGATION IS THE WHOLE MECHANISM

   W3C Trace Context standard header:
     traceparent: 00-{trace_id}-{parent_span_id}-{flags}

   Each service must:
     1. EXTRACT the context from incoming headers
     2. Create a child span
     3. ⭐ INJECT the context into every outgoing call

   ⚠️ Break the chain anywhere — a service that doesn't
     propagate, a queue that drops headers — and the trace
     splits into disconnected fragments. This is the most
     common tracing failure.

   ⭐ ASYNC BOUNDARIES need explicit handling: put trace context
     in message headers when publishing to a queue, and extract
     it in the consumer. Otherwise producer and consumer traces
     are unlinked.
```

```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order.id)
    span.set_attribute("order.total_cents", order.total)
    span.set_attribute("user.tier", user.tier)      # ⭐ high-cardinality
                                                    #   attributes are FINE
                                                    #   here, unlike metrics
    try:
        result = charge_payment(order)
    except PaymentError as e:
        span.record_exception(e)
        span.set_status(trace.Status(trace.StatusCode.ERROR))
        raise
```

```
   ⭐ AUTO-INSTRUMENTATION GETS YOU 80% FREE
     OpenTelemetry auto-instrumentation covers HTTP servers and
     clients, database drivers, message queues, and caches
     without code changes.

     ⭐ Then add MANUAL spans and attributes around your BUSINESS
       logic — that's where the auto-instrumentation can't help
       and where the interesting questions live.
```

---

## 6. What to Measure

```
   ⭐ THE GOLDEN SIGNALS (Google SRE) — for user-facing services

   ┌──────────────────────────────────────────────────────────────┐
   │ LATENCY      How long requests take.                         │
   │              ⭐ Measure successful and FAILED separately —     │
   │                fast failures would otherwise flatter your p99│
   │ TRAFFIC      Demand on the system (RPS, concurrent users)    │
   │ ERRORS       Rate of failed requests.                        │
   │              ⭐ Include "successful" responses that are wrong │
   │                — a 200 with an empty body is an error        │
   │ SATURATION   ⭐ How full the system is. The most predictive   │
   │              signal — it tells you about problems BEFORE     │
   │              they become errors.                             │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE USE METHOD — for RESOURCES (CPU, disk, network, pools)
     Utilization · Saturation · Errors
     ⭐ Saturation is again the one people skip. A disk at 100%
       utilization with queue depth 1 is fine; at 70% with
       queue depth 50 it's in serious trouble.

   ⭐ THE RED METHOD — for SERVICES
     Rate · Errors · Duration
     (essentially the golden signals minus saturation)
```

```
   ⭐ AND THE ONES PEOPLE FORGET

   BUSINESS METRICS   checkout completion rate · signups ·
                      revenue per minute
                      ⭐ These detect failures that infrastructure
                        metrics MISS — a bug making the buy button
                        invisible produces perfect latency and
                        zero errors while revenue collapses.

   DEPENDENCY HEALTH  latency and error rate per downstream,
                      circuit breaker state

   QUEUE DEPTH        ⭐ measured as TIME TO DRAIN, not message
                      count. A million messages behind a fast
                      consumer is seconds; a thousand behind a
                      slow one is minutes.
```

---

## 7. SLIs, SLOs, and Error Budgets

#### 💬 The framework that makes reliability a decision rather than a vibe

```
   SLI   Service Level INDICATOR — the measurement
         "proportion of requests served in <300ms"

   SLO   Service Level OBJECTIVE — your internal target
         "99.9% of requests in <300ms over 30 days"

   SLA   Service Level AGREEMENT — the contractual promise,
         with financial penalties
         ⭐ Always set the SLA looser than the SLO, so you
           breach your internal target well before you breach
           a contract.

   ⭐ ERROR BUDGET = 100% − SLO
     99.9% over 30 days → 43 minutes of allowed failure
```

```
   ⭐⭐ THE ERROR BUDGET IS THE POINT OF THE WHOLE FRAMEWORK

   It converts "should we ship this risky feature?" from an
   argument into a data question.

   ┌──────────────────────────────────────────────────────────────┐
   │ BUDGET REMAINING  → ship freely. Reliability is adequate;    │
   │                     spending budget on velocity is correct.  │
   │ BUDGET EXHAUSTED  → ⭐ feature freeze. All engineering effort │
   │                     goes to reliability until the budget     │
   │                     recovers.                                │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THIS ALIGNS INCENTIVES STRUCTURALLY.
     Product wants features; SRE wants stability. The error
     budget makes that tradeoff explicit, measured, and
     pre-agreed — so it's resolved by policy rather than by
     whoever argues loudest during an incident.

   ⚠️ IT ONLY WORKS IF THE FREEZE IS REAL. An error budget that
     leadership overrides whenever it's inconvenient is a
     dashboard, not a policy.
```

```
   THE AVAILABILITY TABLE — worth memorizing

   99%      "two nines"     7.3 hours/month    ⚠️ not viable
   99.9%    "three nines"   43 minutes/month   ← typical SaaS
   99.95%                   22 minutes/month
   99.99%   "four nines"    4.4 minutes/month  ← serious product
   99.999%  "five nines"    26 seconds/month   ← telco/payments

   ⭐ EACH ADDITIONAL NINE COSTS ROUGHLY 10× MORE.
     And beyond about three nines, your users' networks and
     devices are less reliable than your service — so the extra
     investment buys nothing they can perceive.

   ⭐ THE RIGHT SLO IS NOT 100%. A 100% target means never
     shipping anything, and it wastes budget that could buy
     features users actually want.
```

```promql
# ⭐ Error budget burn rate — the alerting primitive
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  / sum(rate(http_requests_total[1h]))
) / (1 - 0.999)

# Result interpretation:
#   1  = burning budget exactly at the sustainable rate
#   14 = ⚠️ will exhaust the entire 30-day budget in ~2 days
```

---

## 8. Alerting

```
   ⭐⭐ THE CARDINAL RULE:
     ALERT ON SYMPTOMS THE USER FEELS, NOT ON CAUSES.

   ❌ CAUSE-BASED           ✅ SYMPTOM-BASED
   "CPU is above 80%"       "p99 latency exceeds the SLO"
   "A pod restarted"        "Error rate exceeds the SLO"
   "Disk is 85% full"       "Requests are failing"

   ⭐ WHY: high CPU with fine latency needs no human at 3am.
     And you cannot enumerate every cause in advance — but
     you CAN enumerate the handful of symptoms users care about.

   ⚠️ THE EXCEPTION: leading indicators of unavoidable future
     failure, where you need lead time to act. "Disk will be
     full in 4 hours" is legitimately actionable. "Certificate
     expires in 14 days" likewise.
```

### Multi-window burn-rate alerting

```
   ⚠️ THE PROBLEM WITH NAIVE THRESHOLD ALERTS
     "Alert if error rate > 1% for 5 minutes"
     → too noisy for brief blips
     → too slow for a catastrophic outage
     → completely blind to a slow burn that quietly consumes
       the entire month's budget

   ⭐ THE GOOGLE SRE ANSWER: alert on BURN RATE across two
     windows simultaneously — a long window for significance
     and a short window for "is it still happening right now?"

   ┌──────────────────────────────────────────────────────────────┐
   │ Severity  │ Burn rate │ Long win │ Short win │ Budget used   │
   ├───────────┼───────────┼──────────┼───────────┼───────────────┤
   │ ⚠️ PAGE    │    14×    │   1 hour │  5 min    │ 2% in 1h      │
   │ ⚠️ PAGE    │     6×    │   6 hours│  30 min   │ 5% in 6h      │
   │ 🎫 TICKET  │     3×    │   1 day  │  2 hours  │ 10% in 1 day  │
   │ 🎫 TICKET  │     1×    │   3 days │  6 hours  │ 10% in 3 days │
   └───────────┴───────────┴──────────┴───────────┴───────────────┘

   ⭐ THE SHORT WINDOW PREVENTS ALERTING ON AN ALREADY-RESOLVED
     PROBLEM. Without it, a 10-minute outage keeps paging for
     an hour after it's fixed, because the long window still
     shows the damage.
```

```
   ⭐ EVERY PAGE MUST BE: URGENT · ACTIONABLE · IMPORTANT

   If a human can't do something meaningful right now, it is
   not a page. Make it a ticket.

   ⚠️ ALERT FATIGUE IS THE REAL FAILURE MODE.
     A team that gets 50 pages a night stops reading them.
     Then the one that mattered is missed. ⭐ FEWER, BETTER
     ALERTS IS ALWAYS THE RIGHT DIRECTION.

   THE ALERT REVIEW RITUAL (weekly):
     • Which alerts fired?
     • Which required NO action? → delete or downgrade them
     • Which incidents had NO alert? → add one
     • ⭐ What's the ratio of actionable to total? Track it.
```

```yaml
# Every alert needs a runbook link. Non-negotiable.
- alert: HighErrorBudgetBurn
  expr: |
    (sum(rate(http_requests_total{status=~"5.."}[1h]))
     / sum(rate(http_requests_total[1h]))) > (14 * 0.001)
    and
    (sum(rate(http_requests_total{status=~"5.."}[5m]))
     / sum(rate(http_requests_total[5m]))) > (14 * 0.001)
  for: 2m
  labels: { severity: page }
  annotations:
    summary: "Burning error budget 14× — will exhaust in ~2 days"
    runbook: "https://wiki/runbooks/high-error-rate"    # ⭐ REQUIRED
    dashboard: "https://grafana/d/service-overview"
```

---

## 9. Incident Response

```
   ⭐ THE INCIDENT COMMAND STRUCTURE

   ┌──────────────────────────────────────────────────────────────┐
   │ INCIDENT COMMANDER (IC)                                      │
   │   ⭐ COORDINATES. Does NOT debug.                             │
   │   Makes decisions, assigns tasks, decides on escalation      │
   │   and rollback. The single most important role, and the      │
   │   one most often missing.                                    │
   ├──────────────────────────────────────────────────────────────┤
   │ OPERATIONS LEAD    Actually makes changes to the system      │
   ├──────────────────────────────────────────────────────────────┤
   │ COMMUNICATIONS     Updates the status page, stakeholders,    │
   │                    and support. ⭐ Shields responders from    │
   │                    "any update?" interruptions.              │
   ├──────────────────────────────────────────────────────────────┤
   │ SCRIBE             Timestamped timeline of actions and       │
   │                    findings. ⭐ Invaluable for the postmortem │
   │                    and impossible to reconstruct later.      │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ In a small incident one person may hold several roles —
     but the ROLES must be explicitly named and claimed, or
     five people debug in parallel, nobody communicates, and
     two people make conflicting changes simultaneously.
```

```
   ⭐⭐ MITIGATE FIRST, DIAGNOSE SECOND.

   These are separate activities and conflating them extends
   outages dramatically.

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. DETECT      alert fires, or a customer reports it         │
   │ 2. TRIAGE      how bad? who's affected? declare severity     │
   │ 3. ⭐ MITIGATE  RESTORE SERVICE. Roll back, fail over, shed   │
   │                load, flip a feature flag. Do NOT wait to     │
   │                understand the root cause.                    │
   │ 4. DIAGNOSE    now find out why                              │
   │ 5. RESOLVE     permanent fix                                 │
   │ 6. LEARN       blameless postmortem                          │
   └──────────────────────────────────────────────────────────────┘

   ⭐ "Did anything change recently?" is the highest-yield first
     question. Most incidents are caused by a change — a deploy,
     a config update, a feature flag, a certificate expiry, or
     a dependency's change.
```

```
   SEVERITY LEVELS — define these BEFORE you need them

   SEV1  ⚠️ Complete outage or data loss. All hands, page
         leadership, public status page. Response: immediate.
   SEV2  Major degradation, significant user impact.
         Response: within minutes.
   SEV3  Minor degradation, workaround exists.
         Response: business hours.
   SEV4  Cosmetic or internal only. Normal ticket.

   ⭐ Err toward declaring HIGHER severity. Downgrading is easy
     and free; realizing an hour in that you under-triaged has
     already cost you an hour.
```

```
   ⭐ RUNBOOKS THAT ACTUALLY HELP

   A good runbook, for the specific alert:
     • What this alert means, in one sentence
     • ⭐ User impact — what is actually broken for whom
     • First diagnostic commands, copy-pasteable
     • ⭐ Known causes and their specific fixes
     • Mitigation steps (rollback command, feature flag name,
       scaling command) — exact, not described
     • Escalation path with names
     • Link to the relevant dashboard

   ⚠️ A runbook that says "investigate the issue" is worthless.
     Write them at 2pm assuming the reader is exhausted,
     unfamiliar with this service, and it's 3am.
```

---

## 10. Postmortems

```
   ⭐⭐ BLAMELESS IS NOT POLITENESS — IT'S EPISTEMOLOGY.

   If people fear blame, they hide information. You then learn
   nothing and the same failure recurs. The goal is maximum
   information, which requires psychological safety as a
   precondition.

   ⭐ THE REFRAME:
     ❌ "Alice deployed a bad config"
     ✅ "A config change reached production without validation
        that would have caught the error. Why was that path
        possible?"

   ⭐ SECOND STORY THINKING: it always made sense to the person
     at the time, given what they could see. Ask what made the
     wrong action look reasonable — that's the system flaw.
```

```
   THE POSTMORTEM TEMPLATE

   ┌──────────────────────────────────────────────────────────────┐
   │ SUMMARY        What happened, in two sentences               │
   │ IMPACT         ⭐ QUANTIFIED: duration, users affected,       │
   │                requests failed, revenue impact, SLO burn     │
   │ TIMELINE       Timestamped, including DETECTION time and     │
   │                each mitigation attempt                       │
   │ ROOT CAUSE     ⭐ Usually plural. "Five whys" is a starting   │
   │                point, not a stopping point — complex         │
   │                failures have contributing factors, not one   │
   │                cause.                                        │
   │ WHAT WENT WELL ⭐ Genuinely important. Reinforce what worked. │
   │ WHAT WENT POORLY                                             │
   │ WHERE WE GOT LUCKY  ⭐ Often the most valuable section —      │
   │                the near-misses reveal fragility that didn't  │
   │                happen to fire this time                      │
   │ ACTION ITEMS   ⭐ Each with an OWNER, a DUE DATE, and a       │
   │                priority. Tracked in the normal backlog.      │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ THE FAILURE MODE: postmortems written, filed, and never
     acted on. Action items must enter the real backlog with
     real owners, and completion must be reviewed. Otherwise
     you're documenting your outages rather than preventing them.
```

```
   ⭐ ACTION ITEMS RANKED BY DURABILITY

   1. ⭐ Make the failure IMPOSSIBLE (a type, a constraint,
      a policy check that blocks it)
   2. Make it automatically DETECTED and mitigated
   3. Make it detected and alerted
   4. Add a runbook so response is faster
   5. ⚠️ "Be more careful" / "add training" — the weakest
      possible action item, and the most commonly written one

   ⭐ Push every action item as far up this list as you can.
```

---

## 11. Chaos Engineering

```
   ⭐ THE PREMISE: you don't know how your system fails until
     you make it fail. Every untested failover is a hypothesis.

   THE METHOD — this is a scientific experiment, not vandalism

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. Define STEADY STATE — the measurable normal (e.g.         │
   │    "checkout success rate ≥ 99%")                            │
   │ 2. HYPOTHESIZE that it holds under a specific failure        │
   │ 3. Inject the failure in the SMALLEST possible blast radius  │
   │ 4. ⭐ Measure. Did steady state hold?                         │
   │ 5. If not, you found a real weakness BEFORE a customer did   │
   │ 6. Fix it, then widen the blast radius                       │
   └──────────────────────────────────────────────────────────────┘

   EXPERIMENTS WORTH RUNNING
     • Kill a random instance (Chaos Monkey)
     • Inject latency into a dependency
     • Return errors from a dependency
     • ⭐ Fill a disk · exhaust a connection pool
     • Partition the network between services
     • ⭐ Take out an entire availability zone
     • Expire a certificate in staging

   ⚠️ PREREQUISITES — do NOT start here
     Solid observability (you must be able to SEE the impact),
     a tested rollback, and an agreed abort criterion. Chaos
     engineering on an unobservable system just causes outages.
```

```
   ⭐ GAME DAYS — the higher-leverage version

   A scheduled exercise where the team responds to a simulated
   incident using real tooling and real runbooks.

   ⭐ What they reliably reveal:
     • The runbook references a dashboard that no longer exists
     • Nobody knows who can approve a rollback
     • The on-call person lacks the required permissions
     • The escalation contact left the company
     • The failover procedure has an undocumented manual step

   These are exactly the things that turn a 10-minute incident
   into a 3-hour one, and you cannot find them any other way.
```

---

## 12. Toil and Automation

```
   ⭐ TOIL is work that is:
     manual · repetitive · automatable · tactical (no enduring
     value) · scales linearly with the service

   ⚠️ Toil is not simply "work I dislike." Debugging a novel
     problem is hard but NOT toil — it produces understanding.
     Manually restarting a service every night IS toil.

   ⭐ THE GOOGLE SRE TARGET: keep toil below 50% of time.
     Above that, the team becomes an operations team, has no
     capacity to improve the system, and the toil compounds
     because nothing gets fixed.
```

```
   THE AUTOMATION LADDER

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. Do it manually, and ⭐ DOCUMENT it while doing it          │
   │ 2. Write a script; run it manually                           │
   │ 3. Trigger the script automatically, human confirms          │
   │ 4. Fully automated with a human notified                     │
   │ 5. ⭐ ELIMINATE THE NEED ENTIRELY — fix the underlying cause  │
   └──────────────────────────────────────────────────────────────┘

   ⭐ STEP 5 IS THE ONE PEOPLE SKIP.
     Automating a nightly restart is better than doing it
     manually — but fixing the memory leak means the task
     ceases to exist. Automation can entrench a problem by
     making it tolerable.
```

```
   ⭐ ON-CALL HEALTH — the leading indicator for team health

   TRACK:
     • Pages per shift  (⭐ target: fewer than 2, so sleep is
       possible)
     • Pages outside business hours
     • ⭐ Percentage of pages that were actionable
     • Time to acknowledge, time to mitigate

   ⚠️ An unhealthy on-call rotation causes attrition, and the
     people who leave take the operational knowledge with them.
     ⭐ On-call load is a system-quality metric, not an
       individual-endurance metric.
```

---

## 13. Interview Section

<details>
<summary><b>Q1. Monitoring vs observability — is that just marketing?</b></summary>

There's a real distinction, though the term is heavily marketed.

Monitoring answers questions you anticipated: you predicted disk could fill, so you built a dashboard and an alert for it. Observability is the ability to answer questions you didn't anticipate — "why are checkout requests from Android users in Brazil on the new app version slow?" — without shipping new instrumentation.

The technical difference is cardinality. Metrics systems pre-aggregate, and every unique label combination is a separate time series stored forever. Adding user ID as a label takes down your Prometheus. So metrics fundamentally cannot answer high-cardinality questions.

Observability needs high-cardinality data you can slice arbitrarily after the fact, which is why it lives in traces, structured logs, and wide events rather than metrics.

The reason it matters is unknown-unknowns. In a distributed system, failures emerge from interactions nobody predicted. You cannot pre-build a dashboard for a failure you've never imagined, which is exactly what monitoring requires.

Practically, you need both: metrics for cheap always-on alerting and trends, and high-cardinality data for investigation.
</details>

<details>
<summary><b>Q2. Explain SLOs and error budgets.</b></summary>

An SLI is a measurement — the proportion of requests served under 300 milliseconds. An SLO is your target for it, say 99.9% over 30 days. An SLA is the contractual version with penalties, and it should always be looser than your SLO so you breach the internal target well before the contract.

The error budget is one minus the SLO. At 99.9% over 30 days, that's 43 minutes of allowed failure.

The budget is the actual point of the framework. It converts "should we ship this risky feature" from an argument into a data question. Budget remaining means ship freely — reliability is adequate and spending budget on velocity is the correct trade. Budget exhausted means feature freeze, with engineering effort going to reliability until it recovers.

That aligns incentives structurally. Product wants features, SRE wants stability, and the budget makes the tradeoff explicit and pre-agreed rather than decided by whoever argues loudest during an incident.

Two things worth adding. The right SLO is not 100% — that means never shipping, and beyond about three nines your users' own networks are less reliable than your service, so the extra investment buys nothing they can perceive. And the framework only works if the freeze is real. A budget leadership overrides whenever inconvenient is a dashboard, not a policy.
</details>

<details>
<summary><b>Q3. What should you alert on?</b></summary>

Symptoms the user feels, not causes.

Alerting on high CPU pages someone for a condition that may be entirely fine — high CPU with healthy latency needs no human at 3am. And you can't enumerate every possible cause in advance, but you can enumerate the handful of symptoms users actually care about: latency, errors, and availability.

For the mechanism, I'd use multi-window burn-rate alerting rather than static thresholds. A naive "error rate above 1% for 5 minutes" is simultaneously too noisy for blips, too slow for a catastrophic outage, and blind to a slow burn quietly consuming the whole month's budget.

Burn rate fixes this by paging at 14× burn measured over both a one-hour and a five-minute window. The long window establishes significance; the short window confirms it's still happening — without it, a ten-minute outage keeps paging for an hour after it's fixed.

The overriding constraint is that every page must be urgent, actionable, and important. If a human can't do something meaningful right now, it's a ticket. Alert fatigue is the real failure mode: a team getting fifty pages a night stops reading them, and then misses the one that mattered.

The one legitimate exception to symptom-based alerting is leading indicators of unavoidable future failure — "disk full in four hours" or "certificate expires in fourteen days" — where you need lead time.
</details>

<details>
<summary><b>Q4. Walk me through responding to a production incident.</b></summary>

First, establish roles explicitly, even if one person holds several. An incident commander who coordinates and does not debug, an operations lead making changes, someone on communications, and a scribe keeping a timestamped timeline. Without named roles, five people debug in parallel, nobody communicates, and two people make conflicting changes.

Then triage: how bad, who's affected, declare a severity. I'd err toward declaring higher — downgrading is free, while realizing an hour in that you under-triaged has already cost an hour.

Then mitigate before diagnosing. Roll back, fail over, shed load, flip a feature flag. Restoring service and understanding the cause are separate activities, and conflating them extends outages significantly. The instinct to understand first is strong and usually wrong.

The highest-yield first question is "did anything change recently?" — most incidents are caused by a deploy, a config change, a feature flag, a certificate expiry, or a dependency's change.

Once service is restored, diagnose properly, ship a permanent fix, and run a blameless postmortem.

Throughout, communications matter more than people expect. A dedicated comms role shields responders from constant "any update?" interruptions, which are a genuine drag on resolution time.
</details>

<details>
<summary><b>Q5. Why blameless postmortems?</b></summary>

It's epistemology rather than politeness. If people fear blame they withhold information, so you learn nothing and the same failure recurs. The goal is maximum information, and psychological safety is a precondition for that.

The reframe is from "Alice deployed a bad config" to "a config change reached production without validation that would have caught it — why was that path possible?" The second question produces a fix; the first produces a more careful Alice and an identical outage next quarter with someone else.

Second-story thinking helps: it always made sense to the person at the time, given what they could see. Asking what made the wrong action look reasonable identifies the actual system flaw.

For the document itself, I'd emphasize quantified impact, a timeline including detection time, and a "where we got lucky" section — the near-misses are often the most valuable content, because they reveal fragility that happened not to fire this time.

And action items ranked by durability. Making the failure impossible through a constraint or policy check beats automatic detection, which beats alerting, which beats a runbook, which beats "be more careful" — the weakest and most commonly written action item.

The real failure mode is postmortems that are written, filed, and never acted on. Action items need real owners, real due dates, and review.
</details>

<details>
<summary><b>Q6. How would you debug a latency spike in a microservices system?</b></summary>

Start with traces, because that's the only tool that shows where time actually went across service boundaries. Metrics tell me p99 rose; a trace tells me 340 of 450 milliseconds were spent waiting on an external payment API.

Concretely: find an exemplar trace from the affected period — good metrics systems link exemplars directly from the histogram — and read the span waterfall. That immediately localizes the problem to a service and often to a specific span.

Then jump to that service's logs filtered by trace ID for the actual error or the specific query.

If tracing isn't in place, I'd work down the golden signals per service, looking for the one where latency rose without its own downstream latency rising — that's where the time is being added.

I'd also check the usual suspects in parallel: was there a deploy, did traffic change shape, is a dependency degraded, is a connection pool saturated, is a specific instance the problem rather than the service as a whole.

That last one matters and aggregate metrics hide it. A single bad instance behind a load balancer poisons a fraction of requests, which looks like a mild p99 increase in aggregate but is a complete failure for the users who hit it. Per-instance latency percentiles reveal it immediately.
</details>

<details>
<summary><b>Q7. What's wrong with monitoring average latency?</b></summary>

Averages hide the tail, and the tail is where users suffer. Ninety-nine requests at 10 milliseconds and one at 5 seconds averages to 60 milliseconds, which looks healthy while one in a hundred users waits five seconds.

So percentiles — p50, p95, p99, and often p99.9.

Two subtleties matter beyond that. Percentiles don't average. Taking the mean of p99 across instances is a meaningless number. You have to aggregate the histogram buckets and then compute the quantile, which is exactly why histograms are the right metric type and summaries aren't — summaries compute quantiles client-side and can't be combined.

And single-service percentiles understate user pain. A page making twenty backend calls hits the p99 on at least one of them roughly eighteen percent of the time. So you have to measure the user-facing journey, not just individual service latencies.

I'd also separate successful and failed request latency, because fast failures flatter your percentiles — a service returning 500s in 5 milliseconds can look like it improved.
</details>

<details>
<summary><b>Q8. What is toil and why does it matter?</b></summary>

Toil is work that's manual, repetitive, automatable, tactical in the sense of producing no enduring value, and scales linearly with the service.

The distinction people miss is that toil isn't just "work I dislike." Debugging a novel problem is hard but produces understanding, so it's not toil. Manually restarting a service every night is toil.

Google's SRE target is keeping it under fifty percent of time. Above that a team becomes an operations team with no capacity to improve the system, and toil compounds because nothing gets fixed.

The automation ladder runs from doing it manually while documenting, to scripting it, to triggering automatically with human confirmation, to full automation — and then to the step people skip, which is eliminating the need entirely.

That last step matters most. Automating a nightly restart is better than doing it by hand, but fixing the memory leak means the task ceases to exist. Automation can entrench a problem by making it tolerable enough that nobody fixes the cause.

I'd track on-call health as the leading indicator: pages per shift, out-of-hours pages, and the percentage that were actionable. An unhealthy rotation causes attrition, and the people who leave take the operational knowledge with them. On-call load is a system-quality metric, not a measure of individual endurance.
</details>

---

## 14. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║               OBSERVABILITY & SRE — ONE PAGE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ MONITORING = questions you ANTICIPATED (low cardinality, cheap)      ║
║ OBSERVABILITY = questions you DIDN'T (high cardinality, sliceable)   ║
║ ⭐ CARDINALITY is the dividing line. NEVER put user_id/request_id     ║
║   in a metric label — it will kill your metrics system.              ║
╠══════════════════════════════════════════════════════════════════════╣
║ METRICS(trends+alerting) · LOGS(detail) · TRACES(⭐ where time went)  ║
║ ⭐ THE GLUE: propagate a trace_id through EVERYTHING                  ║
║   metric alert → exemplar trace → filtered logs → the actual error   ║
║ counter(rate() it!) · gauge · ⭐ HISTOGRAM (aggregatable) · summary   ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ AVERAGES LIE → percentiles. ⭐ AND PERCENTILES DON'T AVERAGE —      ║
║   aggregate BUCKETS then compute the quantile                        ║
║ ⭐ 20 backend calls → you hit p99 on one ~18% of the time. Measure    ║
║   the USER JOURNEY, not just services.                               ║
╠══════════════════════════════════════════════════════════════════════╣
║ GOLDEN SIGNALS: latency · traffic · errors · ⭐ SATURATION (predicts) ║
║ USE(resources): utilization · saturation · errors                    ║
║ RED(services): rate · errors · duration                              ║
║ ⭐ + BUSINESS METRICS — they catch failures infra metrics miss        ║
║ ⭐ queue depth as TIME TO DRAIN, not message count                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ SLI(measure) → SLO(target) → SLA(contract, looser than SLO)          ║
║ ⭐ ERROR BUDGET = 100% − SLO.  99.9% = 43 min/month                   ║
║   budget left → ship · budget gone → ⭐ FREEZE                        ║
║   makes the velocity/stability tradeoff a DATA question              ║
║   ⚠️ only works if the freeze is actually honored                     ║
║ ⭐ each extra nine ≈ 10× cost; past 3 nines users can't perceive it   ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ ALERT ON SYMPTOMS USERS FEEL, NOT CAUSES                          ║
║   multi-window BURN RATE (14× over 1h AND 5m) — the short window     ║
║   stops you paging about an already-fixed problem                    ║
║   every page: urgent + actionable + important, ⭐ with a RUNBOOK link ║
║   ⚠️ alert fatigue is the real failure — FEWER, BETTER alerts         ║
╠══════════════════════════════════════════════════════════════════════╣
║ INCIDENT: name roles (IC coordinates, does NOT debug) →              ║
║   ⭐ MITIGATE FIRST, diagnose second → "what changed recently?"       ║
║ POSTMORTEM: blameless because FEAR HIDES INFORMATION                 ║
║   quantify impact · ⭐ "where we got lucky" · action items ranked     ║
║   impossible > auto-detected > alerted > runbook > "be careful"      ║
╠══════════════════════════════════════════════════════════════════════╣
║ CHAOS: steady state → hypothesis → smallest blast radius → measure   ║
║   ⭐ GAME DAYS find the stale runbook, the missing permission, the    ║
║   escalation contact who left — the things that turn 10 min into 3h  ║
║ TOIL < 50%. ⭐ The final step is ELIMINATING the need, not            ║
║   automating it — automation can entrench a problem.                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Linux & Networking](linux-networking.md) · [Kubernetes](kubernetes.md) · [CI/CD](cicd.md) · [System Design Fundamentals](../05-system-design/00-fundamentals.md)
