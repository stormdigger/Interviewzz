# 🗓️ Study Plans

> Pick one track. Ignore everything else in the library until it's done.

---

## Choosing Your Track

```
                    What's your situation?
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
  Interview in        Interview in        No deadline,
   ~1 month            ~3 months          building depth
        │                   │                   │
        ▼                   ▼                   ▼
   TRACK A            TRACK B             TRACK C
   Sprint             Standard            Mastery
   (30 days)          (90 days)           (180 days)
```

Cross-cut by role:

| Role target | Weight DSA | Weight SysDesign | Weight Domain |
|---|---|---|---|
| Junior / New grad | 60% | 10% | 30% |
| Mid-level (3-5y) | 40% | 25% | 35% |
| Senior (6y+) | 25% | 40% | 35% |
| Staff+ | 15% | 45% | 40% (+ architecture, influence) |
| Frontend specialist | 30% | 20% | 50% (React, browser, perf) |
| Backend specialist | 35% | 30% | 35% (DB, API, queues) |
| DevOps / SRE | 20% | 35% | 45% (K8s, cloud, observability) |
| Security | 20% | 25% | 55% (appsec, crypto, network) |

---

## 🏃 Track A — 30-Day Sprint

**Assumption:** you already have working experience; this is *recall activation*, not first learning.

```
Week 1: DSA Core          Week 2: DSA Advanced + SysDesign Basics
├─ Patterns               ├─ Trees, graphs
├─ Arrays/strings         ├─ DP intro
├─ Hashing                ├─ SysDesign fundamentals
└─ Two pointers           └─ Building blocks

Week 3: SysDesign + Domain    Week 4: Simulation
├─ Case studies 1-2           ├─ 6 mock interviews
├─ Your stack deep dive       ├─ Behavioral stories
├─ Databases + caching        ├─ Weak-area patching
└─ Framework                  └─ Rest before onsite
```

### Daily structure (Track A)

| Time | Activity |
|---|---|
| 60 min | 3 DSA problems (1 easy warm-up, 2 medium/hard) |
| 45 min | One book section, active recall |
| 30 min | Review yesterday's misses |
| 15 min | Behavioral story polish (1 story/day) |

### Week-by-week detail

**Week 1 — Foundations**
| Day | DSA | Reading |
|---|---|---|
| 1 | Patterns chapter, complexity | [Patterns](../04-dsa/00-patterns.md) |
| 2 | Arrays: 3 problems | [Arrays & Strings](../04-dsa/01-arrays-strings.md) §1-2 |
| 3 | Arrays: 3 problems | Arrays §3-4 |
| 4 | Hashing: 3 problems | [Hashing](../04-dsa/02-hashing.md) |
| 5 | Two pointers: 3 problems | [Two Pointers](../04-dsa/03-two-pointers-sliding-window.md) §1 |
| 6 | Sliding window: 3 problems | Two Pointers §2 |
| 7 | **Mixed review — 5 random problems** | Blank-page recall of week |

**Week 2 — Structures**
| Day | DSA | Reading |
|---|---|---|
| 8 | Linked lists: 3 | [Linked Lists](../04-dsa/04-linked-lists.md) |
| 9 | Stacks/queues: 3 | [Stacks & Queues](../04-dsa/05-stacks-queues.md) |
| 10 | Trees: 3 | [Trees](../04-dsa/06-trees.md) §1-2 |
| 11 | Trees: 3 | Trees §3-4 |
| 12 | Heaps/intervals: 3 | [Heaps & Intervals](../04-dsa/07-heaps-intervals.md) |
| 13 | Graphs: 3 | [Graphs](../04-dsa/08-graphs.md) §1-2 |
| 14 | **Mixed review** | [SysDesign Fundamentals](../05-system-design/00-fundamentals.md) |

**Week 3 — Design + Domain**
| Day | DSA | Reading |
|---|---|---|
| 15 | Graphs: 3 | [Building Blocks](../05-system-design/01-building-blocks.md) |
| 16 | DP: 3 | [Framework](../05-system-design/02-framework.md) |
| 17 | DP: 3 | [Case Studies 1](../05-system-design/03-case-studies-1.md) — Uber, Twitter |
| 18 | DP: 3 | Case Studies 1 — Netflix, WhatsApp |
| 19 | Greedy/backtrack: 3 | [Databases](../03-backend/databases.md) |
| 20 | Mixed: 3 | [Caching](../03-backend/caching.md) |
| 21 | **Mock: 1 DSA + 1 design** | Your framework book |

**Week 4 — Simulation**
| Day | Focus |
|---|---|
| 22 | Full mock loop (2 coding + 1 design) |
| 23 | Patch weakest area from day 22 |
| 24 | Full mock loop |
| 25 | Patch |
| 26 | Behavioral mock, 8 stories tight |
| 27 | Light review, cheat sheets only |
| 28-30 | Taper: 1 easy problem/day, sleep, logistics |

---

## 🚶 Track B — 90-Day Standard

**Assumption:** you want real depth plus interview readiness.

```
Phase 1 (Days 1-30): FOUNDATIONS
  Language depth + DSA patterns + backend basics
  ────────────────────────────────────────────────
Phase 2 (Days 31-60): DEPTH
  System design + your specialty + cloud/devops
  ────────────────────────────────────────────────
Phase 3 (Days 61-90): INTEGRATION
  Hard DSA + case studies + projects + mocks
```

### Phase 1 — Foundations (Days 1-30)

| Week | Mon-Fri (weekday, ~2h) | Weekend (~4h) |
|---|---|---|
| 1 | [JS](../01-languages/javascript.md) or [Python](../01-languages/python.md) or [Java](../01-languages/java.md) — one section/day | DSA patterns + 6 array problems |
| 2 | Same language, finish book | Hashing + two pointers, 8 problems |
| 3 | [API Design](../03-backend/api-design.md) + [Databases](../03-backend/databases.md) | Linked lists + stacks, 8 problems |
| 4 | [SQL Mastery](../08-data-ai/sql.md) | Trees, 8 problems + **build a small CRUD API** |

### Phase 2 — Depth (Days 31-60)

| Week | Weekday | Weekend |
|---|---|---|
| 5 | [SysDesign Fundamentals](../05-system-design/00-fundamentals.md) + [Building Blocks](../05-system-design/01-building-blocks.md) | Graphs, 8 problems |
| 6 | [Caching](../03-backend/caching.md) + [Queues](../03-backend/queues-streaming.md) | Graphs advanced + **add caching+queue to your project** |
| 7 | [Docker](../06-cloud-devops/docker.md) + [Kubernetes](../06-cloud-devops/kubernetes.md) | DP intro, 8 problems |
| 8 | [AWS](../06-cloud-devops/aws.md) | DP continued + **deploy your project** |

### Phase 3 — Integration (Days 61-90)

| Week | Weekday | Weekend |
|---|---|---|
| 9 | [Case Studies 1](../05-system-design/03-case-studies-1.md) — one/day, whiteboard each | Hard DSA mixed, 8 problems |
| 10 | [Case Studies 2](../05-system-design/04-case-studies-2.md) | Mixed + [Observability](../06-cloud-devops/observability-sre.md) |
| 11 | [Case Studies 3](../05-system-design/05-case-studies-3.md) + [AppSec](../07-security/appsec.md) | Mocks begin: 2 coding, 1 design |
| 12 | [Behavioral](../09-interview/behavioral.md) + weak-area patching | Full loop simulation ×2 |
| 13 | Taper | Rest |

---

## 🧗 Track C — 180-Day Mastery

Two passes over the library. Pass 1 is breadth (L2), Pass 2 is depth (L4) on your chosen specialty.

```
Month 1  ██ Languages: primary L4, secondary L2
Month 2  ██ Frontend OR Backend (your side) to L4
Month 3  ██ DSA full sweep, all patterns, 200 problems
Month 4  ██ System design + distributed theory to L4
Month 5  ██ Cloud, DevOps, Security to L3
Month 6  ██ Data/AI to L2, projects, mocks, polish
```

### Monthly themes with deliverables

| Month | Theme | Deliverable (non-negotiable) |
|---|---|---|
| 1 | Language mastery | Write a small interpreter / async scheduler from scratch |
| 2 | Your stack | Ship a non-trivial app publicly |
| 3 | Algorithms | 200 problems logged, pattern-tagged |
| 4 | Distributed systems | Design doc for a real system, reviewed by someone |
| 5 | Infrastructure | Your app on K8s with CI/CD, monitoring, and a runbook |
| 6 | Synthesis | 10 mock interviews + a written technical blog post |

The deliverable column is the actual curriculum. The reading supports it.

---

## 🎯 Role-Specific Overlays

Apply on top of any track.

### Frontend Engineer
Priority order:
1. [React](../02-frontend/react.md) → L4
2. [JavaScript](../01-languages/javascript.md) → L4
3. [Browser & Performance](../02-frontend/browser-performance.md) → L4
4. [TypeScript](../01-languages/typescript.md) → L3
5. [Next.js](../02-frontend/nextjs.md) → L3
6. [CSS](../02-frontend/css.md) → L3
7. Frontend system design (see [Framework](../05-system-design/02-framework.md) §Frontend) → L3

### Backend Engineer
1. Your language → L4
2. [Databases](../03-backend/databases.md) → L4
3. [API Design](../03-backend/api-design.md) → L4
4. [Caching](../03-backend/caching.md) + [Queues](../03-backend/queues-streaming.md) → L4
5. [System Design](../05-system-design/00-fundamentals.md) → L4
6. Your framework ([FastAPI](../03-backend/fastapi.md) / [Spring](../03-backend/spring-boot.md) / [Node](../03-backend/nodejs.md)) → L4

### DevOps / SRE
1. [Linux & Networking](../06-cloud-devops/linux-networking.md) → L4
2. [Kubernetes](../06-cloud-devops/kubernetes.md) → L4
3. [AWS](../06-cloud-devops/aws.md) → L4
4. [Observability & SRE](../06-cloud-devops/observability-sre.md) → L4
5. [Terraform](../06-cloud-devops/terraform.md) + [CI/CD](../06-cloud-devops/cicd.md) → L3
6. [Docker](../06-cloud-devops/docker.md) → L3

### Security Engineer
1. [AppSec](../07-security/appsec.md) → L4
2. [Cryptography](../07-security/cryptography.md) → L4
3. [Network Security](../06-cloud-devops/linux-networking.md) + [Network Security](../07-security/network-security.md) → L4
4. [Offensive & Defensive](../07-security/offensive-defensive.md) → L4
5. Backend + cloud fundamentals → L3

---

## 📏 Weekly Review Ritual

Every Sunday, 20 minutes, answer in writing:

```
1. What did I learn this week that I could teach?
2. What did I "read" but couldn't reproduce?     ← next week's priority
3. Which problems did I fail, and what was the
   *pattern* I missed (not the specific trick)?
4. What am I avoiding, and why?                  ← usually the real gap
5. One adjustment for next week.
```

Question 4 is the one people skip. The topic you keep postponing is almost always the one with the highest marginal return.

---

**Next:** [Progress Tracker →](progress-tracker.md)
