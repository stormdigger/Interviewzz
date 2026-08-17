# 🔄 CI/CD

> CI/CD is a feedback-loop optimization problem. Every practice here exists to shorten the time between "I wrote a bug" and "I know about it" — or to reduce the damage when that loop fails.

**Prerequisite:** [Docker](docker.md)

---

## 📑 Contents

1. [The Mental Model](#1-the-mental-model)
2. [Continuous Integration](#2-continuous-integration)
3. [The Pipeline](#3-the-pipeline)
4. [Deployment Strategies](#4-deployment-strategies)
5. [Feature Flags](#5-feature-flags)
6. [Database Migrations](#6-database-migrations)
7. [GitOps](#7-gitops)
8. [Supply Chain Security](#8-supply-chain-security)
9. [Pipeline Performance](#9-pipeline-performance)
10. [Branching Strategies](#10-branching-strategies)
11. [Metrics That Matter](#11-metrics-that-matter)
12. [Interview Section](#12-interview-section)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. The Mental Model

#### 💬 What CI/CD is actually optimizing

```
   ⭐ THE FEEDBACK LOOP IS THE PRODUCT.

   bug written ──────────────────────────▶ bug discovered
                        │
                   ⭐ MINIMIZE THIS

   The cost of a defect grows roughly an order of magnitude
   at each stage it escapes:

   ┌──────────────────────────────────────────────────────────────┐
   │ Caught by a type checker in your editor        ~seconds      │
   │ Caught by a unit test locally                  ~minutes      │
   │ Caught by CI on the PR                         ~10 minutes   │
   │ Caught in staging                              ~hours        │
   │ Caught by a canary in production               ~minutes but  │
   │                                                 real users   │
   │ ⚠️ Caught by a customer                         ~days, and    │
   │                                                 reputational │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE SECOND LOOP MATTERS TOO: time-to-recover. Fast, safe,
     reversible deploys mean a bug that escapes costs minutes,
     not a weekend.
```

### CI vs CD vs CD

```
   CONTINUOUS INTEGRATION
     Everyone merges to trunk frequently (⭐ at least daily),
     and every merge is automatically built and tested.
     ⚠️ CI is a PRACTICE, not a tool. Running GitHub Actions on
       a six-week-old feature branch is not continuous
       integration.

   CONTINUOUS DELIVERY
     Every commit that passes is DEPLOYABLE. Release is a
     business decision, executed with one button.

   CONTINUOUS DEPLOYMENT
     Every commit that passes IS deployed, automatically.
     ⭐ Requires very high confidence in your test suite and
       your ability to detect and roll back problems.
```

---

## 2. Continuous Integration

```
   ⭐ THE TEST PYRAMID — and why the shape matters

              ▲  few · slow · expensive · brittle
         ┌────┴────┐
         │   E2E   │     browser/API tests through the real stack
        ┌┴─────────┴┐
        │INTEGRATION│    service + real database (Testcontainers)
       ┌┴───────────┴┐
       │    UNIT     │   pure logic, no I/O, milliseconds
       └─────────────┘
              ▼  many · fast · cheap · stable

   ⚠️ THE INVERTED PYRAMID (mostly E2E) is the most common
     failure mode. Symptoms: a 45-minute pipeline, flaky
     failures nobody trusts, and developers re-running builds
     until they pass — which destroys the entire value of CI.
```

### What runs on every PR

```
   FAST FEEDBACK (⭐ target under 10 minutes total)
   □ Lint + format check          seconds
   □ Type check                   seconds
   □ Unit tests                   ⭐ parallelized, 1-3 minutes
   □ Build                        cached
   □ Security: secret scanning, dependency audit, SAST
   □ Integration tests            with Testcontainers
   □ Container build + image scan

   SLOWER, POST-MERGE OR NIGHTLY
   □ Full E2E suite
   □ Performance regression tests
   □ Load tests
   □ Full dependency/license audit
```

```yaml
# GitHub Actions — the shape that matters
name: CI
on: [pull_request, push]

jobs:
  # ⭐ Fast checks run in PARALLEL and fail early
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint && npm run typecheck

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]        # ⭐ parallel sharding
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        options: >-
          --health-cmd pg_isready --health-interval 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm test -- --shard=${{ matrix.shard }}/4

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # ⭐ full history for secret scanning
      - uses: gitleaks/gitleaks-action@v2
      - run: npm audit --audit-level=high

  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write              # ⭐ OIDC — no long-lived AWS keys
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: us-east-1
      - uses: docker/build-push-action@v6
        with:
          push: true
          # ⭐ tag by commit SHA — never :latest
          tags: ${{ env.REGISTRY }}/app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

```
   ⭐ OIDC FEDERATION IS THE SINGLE BIGGEST CI SECURITY WIN

   Instead of storing long-lived AWS access keys as repository
   secrets, the CI provider presents a signed OIDC token and
   assumes a role scoped to that specific repo and branch.

   ✅ No credentials to leak, rotate, or find in a log
   ✅ Scoped per repository and per branch
   ✅ Short-lived by construction
```

### Flaky tests

```
   ⚠️ FLAKY TESTS DESTROY CI. Once developers learn that a red
     build might be noise, they stop reading failures — and CI
     becomes theatre.

   ⭐ THE POLICY THAT WORKS
     1. Detect flakes automatically (rerun failures; if a rerun
        passes, tag it)
     2. QUARANTINE immediately — move it out of the blocking
        suite the same day
     3. File a ticket with an owner and a deadline
     4. ⭐ Fix or DELETE. A permanently quarantined test is a
        lie about your coverage.

   COMMON CAUSES: shared mutable state between tests · real
   time and sleeps instead of injected clocks · test ordering
   dependence · unmocked network · animation and timing races
   in browser tests · leaked database state between tests
```

---

## 3. The Pipeline

```
   ┌──────────────────────────────────────────────────────────────┐
   │  COMMIT                                                      │
   │     ▼                                                        │
   │  BUILD  ⭐ ONCE. The artifact is immutable from here on.      │
   │     │   Environment differences come from CONFIG, never      │
   │     │   from rebuilding.                                     │
   │     ▼                                                        │
   │  TEST   unit → integration → contract                        │
   │     ▼                                                        │
   │  SCAN   image CVEs · SAST · dependency audit · secrets       │
   │     ▼                                                        │
   │  PUBLISH  to registry, tagged by commit SHA, signed          │
   │     ▼                                                        │
   │  DEPLOY → staging                                            │
   │     ▼                                                        │
   │  VERIFY  smoke tests · E2E · ⭐ automated gates               │
   │     ▼                                                        │
   │  DEPLOY → production (progressive)                           │
   │     ▼                                                        │
   │  OBSERVE  ⭐ auto-rollback on SLO breach                      │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐⭐ BUILD ONCE, DEPLOY MANY — the most important pipeline rule

   ❌ Rebuilding per environment means staging and production
      run DIFFERENT binaries. Every test you ran in staging
      tested something that isn't what you shipped.

   ✅ One build produces one immutable artifact, promoted
      through environments. Configuration is injected at
      runtime.

   → If you can't say with certainty that the exact bytes
     tested in staging are the bytes running in production,
     your pipeline has a correctness hole.
```

---

## 4. Deployment Strategies

```
   ┌──────────────────────────────────────────────────────────────┐
   │ RECREATE           stop all old, start all new               │
   │                    ⚠️ downtime. Only for dev, or when         │
   │                      versions genuinely can't coexist.       │
   ├──────────────────────────────────────────────────────────────┤
   │ ROLLING            replace instances gradually               │
   │                    ✅ no downtime, no extra capacity needed   │
   │                    ⚠️ both versions run simultaneously →      │
   │                      requires backward compatibility         │
   │                    ⚠️ rollback is another slow rolling update │
   ├──────────────────────────────────────────────────────────────┤
   │ BLUE-GREEN         two full environments, flip the router    │
   │                    ✅ ⭐ INSTANT rollback (flip back)          │
   │                    ✅ test green fully before any traffic     │
   │                    ⚠️ 2× infrastructure during the switch     │
   │                    ⚠️ database schema must serve BOTH         │
   ├──────────────────────────────────────────────────────────────┤
   │ CANARY        ⭐    small % of real traffic → observe → widen │
   │                    ✅ limits blast radius to a few %          │
   │                    ✅ ⭐ catches what staging cannot: real     │
   │                      traffic patterns, real data, real scale │
   │                    ⚠️ needs solid metrics and automation      │
   ├──────────────────────────────────────────────────────────────┤
   │ SHADOW / MIRROR    duplicate real traffic to the new version,│
   │                    discard its responses                     │
   │                    ✅ zero user risk, real production load    │
   │                    ⚠️ ⚠️ side effects must be suppressed —     │
   │                      a shadowed payment must not charge      │
   └──────────────────────────────────────────────────────────────┘
```

### Blue-green

```
   ┌──────────┐
   │  ROUTER  │────────────▶ ┌────────────┐
   └──────────┘   100%       │ BLUE (v1)  │  ← live
        │                    └────────────┘
        │  after verification
        └────────────────────▶ ┌────────────┐
                     100%      │ GREEN (v2) │  ← now live
                               └────────────┘
                               ⭐ BLUE stays running for a
                                 while → rollback is a router
                                 flip, measured in seconds
```

### Canary — the recommended default

```
   ⭐ PROGRESSIVE DELIVERY WITH AUTOMATED GATES

   ┌──────────────────────────────────────────────────────────────┐
   │  1%  ─── 5 min ───▶ check SLOs ───▶ pass? continue           │
   │                          │                                   │
   │                        fail                                  │
   │                          ▼                                   │
   │                    ⭐ AUTO-ROLLBACK                           │
   │  5%  ─── 5 min ───▶ check ───▶ ...                           │
   │ 25%  ─── 10 min ──▶ check ───▶ ...                           │
   │ 50%  ─── 10 min ──▶ check ───▶ ...                           │
   │100%                                                          │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE GATES ARE THE WHOLE POINT
     error rate · p99 latency · saturation · business metrics
     (checkout completion, signup rate)

     ⚠️ A canary a human watches is not a canary — it's a slow
       deploy. The value comes from AUTOMATED analysis and
       AUTOMATIC rollback, because humans miss subtle
       regressions and aren't watching at 3am.
```

```yaml
# Argo Rollouts — canary with automated analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis:                          # ⭐ the automated gate
            templates: [{ templateName: success-rate }]
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99    # ⭐ fail → auto-rollback
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status!~"5.."}[2m]))
            / sum(rate(http_requests_total[2m]))
```

---

## 5. Feature Flags

#### 💬 Decoupling deploy from release

```
   ⭐ THE CORE IDEA: DEPLOY ≠ RELEASE.

   Ship the code to production with the feature OFF, then turn
   it on separately. This changes what a deploy means:

   ✅ Merge to trunk continuously, even for unfinished work
     (⭐ this is what makes trunk-based development possible)
   ✅ "Rollback" becomes a config change in SECONDS, with no
     redeploy and no rebuild
   ✅ Gradual rollout by user segment, percentage, or geography
   ✅ A/B testing and experimentation use the same machinery
   ✅ Kill switches for degrading expensive features under load
```

```python
if flags.is_enabled("new-checkout", user=current_user):
    return new_checkout_flow()
return legacy_checkout_flow()
```

```
   ⚠️ THE FLAG DEBT PROBLEM — this is what goes wrong

   Flags accumulate. Each one doubles the theoretical number
   of code paths. Twenty flags is a million combinations you
   cannot possibly test.

   ⭐ THE DISCIPLINE THAT KEEPS IT SANE
     • Every flag has an OWNER and an EXPIRY DATE at creation
     • CI fails the build on flags past their expiry
     • Separate SHORT-LIVED release flags (delete within weeks)
       from LONG-LIVED operational flags (kill switches,
       permission gates) — they have different lifecycles
     • Track flag count as a metric; a growing count is debt
     • ⭐ Removing a flag is part of shipping the feature, not
       a follow-up ticket that never happens
```

---

## 6. Database Migrations

```
   ⚠️ THE CORE CONSTRAINT: DURING A ROLLING DEPLOY, OLD AND NEW
     CODE RUN SIMULTANEOUSLY AGAINST THE SAME DATABASE.

   → Every migration must be backward compatible with the
     currently-deployed code.
   → ⭐ And note: you can roll back CODE instantly, but you
     usually cannot roll back DATA. Migrations are the least
     reversible part of any deploy.
```

### Expand-contract — the only safe pattern

```
   Renaming a column from `name` to `full_name`:

   ┌─ ① EXPAND ──────────────────────────────────────────────────┐
   │  Add `full_name` as NULLABLE. Deploy nothing else.          │
   │  Old code ignores it. Nothing breaks.                       │
   ├─ ② DUAL WRITE ──────────────────────────────────────────────┤
   │  Deploy code that writes BOTH columns, reads the OLD one.   │
   ├─ ③ BACKFILL ────────────────────────────────────────────────┤
   │  Copy existing rows in ⭐ THROTTLED BATCHES with sleeps —    │
   │  never one giant UPDATE (it locks and blows up replica lag).│
   ├─ ④ SWITCH READS ────────────────────────────────────────────┤
   │  Deploy code that reads the NEW column. Still writes both.  │
   │  ⭐ Rollback is still safe at this point.                    │
   ├─ ⑤ STOP WRITING OLD ────────────────────────────────────────┤
   │  Deploy code that only uses the new column.                 │
   ├─ ⑥ CONTRACT ────────────────────────────────────────────────┤
   │  ⭐ VERIFY nothing reads the old column (monitor first!),    │
   │  then drop it. Often days or weeks later.                   │
   └─────────────────────────────────────────────────────────────┘

   ⭐ Six deploys to rename a column. That's the actual cost of
     zero-downtime, and it's why you think carefully about
     schema before shipping it.
```

```
   ⚠️ MIGRATION SAFETY RULES

   □ ⭐ ALWAYS set lock_timeout before DDL. Without it,
     ALTER TABLE waits behind a long query while every new
     query queues behind IT → a "quick migration" becomes a
     full outage.
   □ CREATE INDEX CONCURRENTLY (Postgres) — no write lock
   □ Never add a NOT NULL column with a default to a large
     table in one step on older engines (table rewrite)
   □ Backfill in batches with sleeps; watch replica lag
   □ Migrations run as a separate step BEFORE the app deploy,
     not on application startup
   □ ⚠️ Never run migrations from multiple app replicas
     simultaneously — use a job with a lock
   □ Test against a production-SIZED dataset. A migration that
     takes 200ms on dev data can take 40 minutes on 500M rows.
```

---

## 7. GitOps

```
   ⭐ THE PRINCIPLE: GIT IS THE SINGLE SOURCE OF TRUTH FOR
     DESIRED STATE, AND AN AGENT IN THE CLUSTER CONTINUOUSLY
     RECONCILES REALITY TOWARD IT.

   ┌──────────────────────────────────────────────────────────────┐
   │  PUSH-BASED (traditional)      PULL-BASED (GitOps) ⭐         │
   │                                                              │
   │  CI ──kubectl apply──▶ cluster  Git ◀──watches── Agent        │
   │                                              (in cluster)    │
   │  ⚠️ CI needs cluster credentials ✅ ⭐ NO cluster credentials  │
   │  ⚠️ Drift goes undetected           in CI at all              │
   │  ⚠️ No record of actual state    ✅ ⭐ Drift auto-corrected    │
   │                                  ✅ Git log = deploy history  │
   │                                  ✅ Revert commit = rollback  │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHY "NO CLUSTER CREDENTIALS IN CI" IS THE KILLER FEATURE

   In push-based CI/CD, your CI system holds credentials that
   can modify production. Compromise the CI system — or a
   third-party Action, or a dependency — and you own production.

   With GitOps the agent pulls from git. CI only needs write
   access to a repository. The attack surface shrinks
   dramatically.
```

```
   REPOSITORY STRUCTURE

   app-repo/            source code + Dockerfile + CI
   config-repo/  ⭐      Kubernetes manifests, one dir per env
     ├── base/
     └── overlays/
         ├── staging/
         └── production/

   ⭐ SEPARATE REPOS so a config change doesn't rebuild the app,
     and so config access can be governed differently (e.g.
     production overlay changes require two approvals).

   FLOW: CI builds and pushes an image → updates the image tag
         in config-repo → ArgoCD/Flux notices and syncs.
```

```
   ⚠️ GITOPS GOTCHAS
   • Secrets can't live in git as plaintext → Sealed Secrets,
     SOPS, or External Secrets Operator
   • ⭐ Anything that mutates cluster state outside git (an HPA
     changing replicas, a mutating webhook) causes false drift
     → configure ignoreDifferences
   • The image-tag update commit creates a noisy git history →
     use an image updater with its own commit convention
```

---

## 8. Supply Chain Security

```
   ⚠️ THE ATTACK SURFACE IS THE WHOLE PIPELINE, NOT JUST YOUR CODE

   ┌──────────────────────────────────────────────────────────────┐
   │  Your code            ← the part you actually review         │
   │  Direct dependencies  ← you chose these                      │
   │  ⭐ TRANSITIVE deps    ← hundreds you've never heard of       │
   │  Base images          ← someone else's OS                    │
   │  ⭐ CI plugins/actions ← RUN WITH YOUR CREDENTIALS            │
   │  Build infrastructure ← if compromised, everything is        │
   │  Registry             ← image substitution                   │
   └──────────────────────────────────────────────────────────────┘

   Real incidents to know: SolarWinds (build system compromise),
   Codecov (a modified bash uploader exfiltrating CI secrets),
   event-stream and xz-utils (malicious maintainer takeover),
   dependency confusion (internal package names claimed publicly).
```

```
   ⭐ DEFENSES, ROUGHLY BY IMPACT

   1. ⭐ PIN EVERYTHING BY DIGEST, not by tag
      uses: actions/checkout@8f4b7f8... (not @v4)
      FROM node:22-alpine@sha256:...
      → tags are mutable; digests are not

   2. LOCKFILES committed, and `npm ci` / `pip install
      --require-hashes` in CI — never `npm install`

   3. ⭐ MINIMAL CI PERMISSIONS
      permissions: { contents: read }   as the default
      → grant write only in the specific job that needs it

   4. ⭐ OIDC instead of long-lived cloud credentials

   5. SBOM generation (Syft) + continuous vulnerability
      scanning against it (Grype/Trivy) — so when a new CVE
      lands you know within minutes whether you're affected

   6. SIGN images (cosign) and VERIFY at admission
      → an unsigned image cannot be deployed

   7. SLSA provenance — attest how the artifact was built

   8. ⚠️ Never run untrusted PR code with secrets available
      (`pull_request_target` is the classic GitHub Actions
      footgun — it runs with write permissions and secrets
      against a fork's code)
```

---

## 9. Pipeline Performance

```
   ⭐ TARGET: PR FEEDBACK IN UNDER 10 MINUTES.

   Beyond that, developers context-switch, and the feedback
   loop that CI exists to shorten is broken.

   ┌──────────────────────────────────────────────────────────────┐
   │ CACHE AGGRESSIVELY   dependencies · Docker layers · build     │
   │                      outputs · test fixtures                 │
   ├──────────────────────────────────────────────────────────────┤
   │ PARALLELIZE          ⭐ shard tests across N runners.         │
   │                      Split by historical timing, not         │
   │                      alphabetically, or one shard dominates. │
   ├──────────────────────────────────────────────────────────────┤
   │ FAIL FAST            lint and typecheck before the slow       │
   │                      test suite                              │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ ONLY BUILD WHAT CHANGED                                    │
   │                      Monorepo tooling (Nx, Turborepo, Bazel) │
   │                      computes the affected graph. A change   │
   │                      to one package shouldn't test all forty.│
   ├──────────────────────────────────────────────────────────────┤
   │ RIGHT-SIZE RUNNERS   bigger runners are often cheaper in      │
   │                      total, because they finish sooner       │
   ├──────────────────────────────────────────────────────────────┤
   │ MOVE SLOW THINGS     E2E and performance tests run           │
   │                      post-merge or nightly, not on every PR  │
   └──────────────────────────────────────────────────────────────┘
```

```dockerfile
# ⭐ BuildKit cache mounts — dependency cache persists between
#    builds without ever entering an image layer
RUN --mount=type=cache,target=/root/.npm npm ci
```

```yaml
# Registry-backed layer cache — survives runner recycling
cache-from: type=registry,ref=myrepo/app:buildcache
cache-to:   type=registry,ref=myrepo/app:buildcache,mode=max
```

---

## 10. Branching Strategies

```
   ┌──────────────────────────────────────────────────────────────┐
   │ TRUNK-BASED  ⭐ what high-performing teams do                 │
   │   Short-lived branches (< 1 day), merge to main constantly.  │
   │   Unfinished work hidden behind feature flags.               │
   │   ✅ Minimal merge conflicts · genuine CI · fast flow         │
   │   ⚠️ Requires: strong test suite, feature flags, and the      │
   │     discipline to keep changes small                         │
   ├──────────────────────────────────────────────────────────────┤
   │ GITHUB FLOW                                                  │
   │   main + feature branches + PR review + deploy from main.    │
   │   ⭐ The sensible default for most teams.                     │
   ├──────────────────────────────────────────────────────────────┤
   │ GITFLOW                                                      │
   │   develop, release, hotfix, feature branches.                │
   │   ⚠️ Heavy. Long-lived branches mean painful merges and       │
   │     defeat continuous integration. Its own author has        │
   │     since said it's not right for continuously delivered     │
   │     web applications.                                        │
   │   Reasonable only for versioned software with supported      │
   │   parallel releases.                                         │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE INSIGHT MOST TEAMS MISS
     Branch strategy is downstream of BATCH SIZE. Long-lived
     branches exist because changes are large. Make changes
     small — behind flags where necessary — and the merge
     problem mostly disappears on its own.
```

---

## 11. Metrics That Matter

```
   ⭐ THE FOUR DORA METRICS — the research-backed measures

   ┌──────────────────────┬───────────────────────────────────────┐
   │ DEPLOYMENT FREQUENCY │ Elite: on demand, multiple per day    │
   │                      │ Low: less than monthly                │
   ├──────────────────────┼───────────────────────────────────────┤
   │ LEAD TIME FOR CHANGE │ Elite: under 1 hour (commit → prod)   │
   │                      │ Low: over 1 month                     │
   ├──────────────────────┼───────────────────────────────────────┤
   │ CHANGE FAILURE RATE  │ Elite: 0-15%                          │
   │                      │ Low: 45%+                             │
   ├──────────────────────┼───────────────────────────────────────┤
   │ ⭐ MTTR              │ Elite: under 1 hour                   │
   │ (time to restore)    │ Low: over 1 week                      │
   └──────────────────────┴───────────────────────────────────────┘

   ⭐ THE COUNTERINTUITIVE FINDING
     Speed and stability are NOT a tradeoff. Teams that deploy
     more frequently also have LOWER failure rates and faster
     recovery. Small, frequent, well-tested changes are safer
     than large, rare ones — because each carries less risk
     and is far easier to diagnose when it fails.
```

```
   ALSO WORTH TRACKING
   • Pipeline duration (p50 and ⭐ p95 — the tail is what hurts)
   • Flaky test rate
   • Time from PR open to merge
   • ⭐ Rollback rate and rollback DURATION
   • Failed deploys per week
```

---

## 12. Interview Section

<details>
<summary><b>Q1. Walk me through a production-grade CI/CD pipeline.</b></summary>

On every pull request: lint, typecheck, unit tests sharded across parallel runners, security scanning for secrets and vulnerable dependencies, then integration tests against a real database via Testcontainers. Target is under ten minutes of feedback, because beyond that developers context-switch and the loop CI exists to shorten is broken.

On merge to main: build the container image once, tagged by commit SHA, scan it, sign it, and push. That artifact is then immutable — the same bytes get promoted through every environment.

Deploy to staging automatically, run smoke tests, then progressive deployment to production: a canary at a small percentage with automated analysis of error rate, latency, and business metrics, expanding on success and automatically rolling back on failure.

The two decisions I'd emphasize. Build once, deploy many — if you rebuild per environment, staging tested different bytes than production runs, and your testing has a correctness hole. And automated canary gates rather than human watching, because humans miss subtle regressions and aren't awake at 3am.

I'd also use OIDC federation rather than storing cloud credentials in CI, since that removes the most valuable thing an attacker could steal from the pipeline.
</details>

<details>
<summary><b>Q2. Blue-green vs canary — which and why?</b></summary>

Blue-green runs two complete environments and flips a router. Its strength is instant rollback — flip back, seconds not minutes. You can also fully test green before it takes any traffic. The costs are double infrastructure during the switch, and the database schema has to serve both versions simultaneously.

Canary shifts a small percentage of real traffic to the new version, watches metrics, and expands gradually. Its strength is limiting blast radius to a few percent of users, and — more importantly — catching problems that staging structurally cannot: real traffic patterns, real data shapes, real concurrency, real scale.

I'd default to canary for most services because the blast-radius limiting is the more valuable property. Staging never reproduces production faithfully enough.

Blue-green is better when versions genuinely can't coexist, when you need a single atomic cutover, or when the rollback speed matters more than gradual exposure.

They combine well, too — blue-green at the infrastructure level with canary traffic shifting between them gives both instant rollback and gradual exposure.

The critical detail either way: the gates must be automated. A canary a human watches is just a slow deploy.
</details>

<details>
<summary><b>Q3. How do you do zero-downtime database migrations?</b></summary>

Expand-contract, because during a rolling deploy old and new code run simultaneously against the same database, so every migration has to be backward compatible with currently-deployed code.

Renaming a column takes six steps. Add the new column as nullable. Deploy code writing both, reading old. Backfill existing rows in throttled batches. Deploy code reading new, still writing both. Deploy code using only the new. Then, after verifying nothing reads the old column, drop it — often weeks later.

Six deploys to rename a column. That's the real cost of zero downtime, and it's why you think carefully about schema before shipping.

The operational details that prevent outages: always set a lock timeout before DDL, because ALTER TABLE waits behind a long-running query while every new query queues behind it — that's how a "quick migration" becomes a full outage. Use CREATE INDEX CONCURRENTLY. Backfill in batches with sleeps while watching replica lag. Run migrations as a separate step before the app deploy, never from application startup and never from multiple replicas concurrently.

And the asymmetry worth stating: you can roll back code instantly, but usually not data. Migrations are the least reversible part of any deploy, which is why they need the most care.
</details>

<details>
<summary><b>Q4. What is GitOps and what does it actually buy you?</b></summary>

Git holds the desired state, and an agent inside the cluster continuously reconciles reality toward it. That's a pull model rather than CI pushing changes.

The biggest benefit is security. In push-based CI/CD, your CI system holds credentials that can modify production — so compromising CI, or a third-party Action, or a transitive dependency, gets you production. With GitOps, CI only needs write access to a repository; the agent pulls. The attack surface shrinks substantially.

Second is drift correction. If someone kubectl-edits production at 2am during an incident, the agent detects the divergence and either reverts it or alerts. Push-based deployment has no idea that happened.

Third, git history becomes deployment history with full audit trail, and rollback is reverting a commit.

The gotchas: secrets can't sit in git as plaintext, so you need Sealed Secrets, SOPS, or External Secrets Operator. And anything that legitimately mutates cluster state outside git — an HPA adjusting replicas, a mutating webhook — creates false drift and needs to be explicitly ignored.
</details>

<details>
<summary><b>Q5. How do you handle flaky tests?</b></summary>

Treat them as a production incident for the pipeline, because the damage is systemic. Once developers learn that a red build might be noise, they stop reading failures and start re-running until green — which destroys the entire value of CI.

The policy: detect automatically by rerunning failures and tagging tests that pass on retry. Quarantine the same day — move it out of the blocking suite immediately rather than letting it degrade trust. File a ticket with a named owner and a deadline. Then fix or delete, because a permanently quarantined test is a lie about your coverage.

Root causes cluster predictably: shared mutable state between tests, real time and sleeps instead of injected clocks, ordering dependence, unmocked network calls, animation and timing races in browser tests, and leaked database state.

The structural fix is usually the test pyramid. Most flakiness lives in end-to-end tests, and teams with inverted pyramids — mostly E2E — have chronic flakiness by construction. Pushing coverage down into fast, deterministic unit and integration tests removes the category rather than fighting individual instances.
</details>

<details>
<summary><b>Q6. How do you secure the software supply chain?</b></summary>

The attack surface is the whole pipeline, not just your code — transitive dependencies you've never heard of, base images, CI plugins that run with your credentials, and the build infrastructure itself. SolarWinds compromised the build system; Codecov modified a bash uploader to exfiltrate CI secrets.

The highest-impact controls, roughly in order. Pin everything by digest rather than tag, for both GitHub Actions and base images, since tags are mutable and digests aren't. Commit lockfiles and use `npm ci` rather than `npm install`. Minimize CI permissions to read-only by default, granting write only in the specific job that needs it. Use OIDC federation instead of long-lived cloud credentials.

Then generate an SBOM and continuously scan it, so when a new CVE lands you know within minutes whether you're affected rather than auditing manually. Sign images with cosign and verify at admission, so an unsigned image simply cannot be deployed.

And the specific footgun worth naming: never run untrusted pull request code with secrets available. `pull_request_target` in GitHub Actions runs with write permissions and repository secrets against a fork's code, which has been exploited repeatedly.
</details>

<details>
<summary><b>Q7. Feature flags — value and cost?</b></summary>

The value is decoupling deploy from release. Ship code with the feature off, turn it on separately. That enables trunk-based development, because unfinished work can merge safely behind a flag. It makes "rollback" a config change in seconds with no redeploy. And it gives gradual rollout by segment, A/B testing, and kill switches for shedding expensive features under load — all from the same machinery.

The cost is flag debt, and it's real. Each flag doubles the theoretical number of code paths; twenty flags is a million combinations you cannot test. Old flags become permanent hidden conditionals that nobody understands or dares remove.

The discipline that keeps it manageable: every flag gets an owner and an expiry date at creation, CI fails the build on expired flags, and removing the flag is part of shipping the feature rather than a follow-up ticket that never happens.

I'd also separate short-lived release flags from long-lived operational flags like kill switches and permission gates — they have genuinely different lifecycles and shouldn't be governed by the same policy.
</details>

<details>
<summary><b>Q8. Your deploy broke production. Walk me through it.</b></summary>

Restore service first, diagnose second. Those are separate activities and conflating them extends outages.

So: roll back immediately. With canary or blue-green that's automated or a single action. Don't try to fix forward under pressure unless rollback is genuinely impossible — for instance if a migration already ran destructively, which is exactly why expand-contract matters.

Then verify recovery with actual metrics, not assumption, and communicate status.

Only then diagnose, and the first question is why the pipeline didn't catch it. That's more valuable than the specific bug. Was it a scenario staging can't reproduce, which argues for better canary gates? A missing test, which is a concrete gap to fill? A load-dependent issue, arguing for shadow traffic? A configuration difference between environments, which shouldn't exist if you're building once and deploying many?

The follow-up is a blameless postmortem focused on system improvements rather than individual error. And I'd measure it: change failure rate and MTTR are two of the four DORA metrics, and the research finding worth citing is that teams deploying more frequently also have lower failure rates — small frequent changes are safer, not riskier, because each carries less risk and is far easier to diagnose.
</details>

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                        CI/CD — ONE PAGE                              ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ CI/CD OPTIMIZES THE FEEDBACK LOOP. Cost of a defect grows ~10×     ║
║   at every stage it escapes. Second loop: time-to-RECOVER.           ║
║ CI is a PRACTICE (merge daily), not a tool.                          ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ BUILD ONCE, DEPLOY MANY. Immutable artifact + injected config.    ║
║   Rebuilding per env means staging tested DIFFERENT BYTES.           ║
║ Tag by commit SHA. Deploy by digest. Never :latest.                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ PYRAMID: many unit → some integration → few E2E                      ║
║   ⚠️ inverted pyramid = 45-min flaky pipeline nobody trusts           ║
║ ⭐ FLAKY TESTS: detect → QUARANTINE same day → fix or DELETE          ║
║   (a red build that might be noise destroys CI entirely)             ║
║ Target: PR feedback < 10 min. Shard tests. Cache. Fail fast.         ║
╠══════════════════════════════════════════════════════════════════════╣
║ DEPLOY STRATEGIES                                                    ║
║   rolling(no extra capacity, slow rollback) ·                        ║
║   blue-green(⭐ INSTANT rollback, 2× infra) ·                         ║
║   canary(⭐ limits blast radius, catches what staging CANNOT) ·       ║
║   shadow(zero risk, ⚠️ must suppress side effects)                    ║
║ ⭐ GATES MUST BE AUTOMATED — a canary a human watches is just a       ║
║   slow deploy                                                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ MIGRATIONS: old + new code run SIMULTANEOUSLY                      ║
║   expand → dual-write → backfill(batched!) → switch reads →          ║
║   stop old writes → contract.  6 deploys to rename a column.         ║
║   ⚠️ ALWAYS set lock_timeout · CREATE INDEX CONCURRENTLY              ║
║   ⭐ code rolls back instantly; DATA usually cannot                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ GITOPS: git = desired state, agent PULLS and reconciles              ║
║   ⭐ killer feature: NO cluster credentials in CI                     ║
║   + automatic drift correction + git log = deploy history            ║
╠══════════════════════════════════════════════════════════════════════╣
║ SUPPLY CHAIN: ⭐ pin by DIGEST (actions AND base images) ·            ║
║   lockfiles + npm ci · minimal CI permissions · ⭐ OIDC not keys ·     ║
║   SBOM + continuous scan · sign & verify · ⚠️ never pull_request_target║
╠══════════════════════════════════════════════════════════════════════╣
║ FLAGS decouple DEPLOY from RELEASE → rollback in seconds             ║
║   ⚠️ flag debt: owner + expiry at creation, CI fails on expired       ║
╠══════════════════════════════════════════════════════════════════════╣
║ DORA: deploy frequency · lead time · change failure rate · MTTR      ║
║   ⭐ speed and stability are NOT a tradeoff — frequent small changes  ║
║     are SAFER than rare large ones                                   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Terraform →](terraform.md) · **Related:** [Docker](docker.md) · [Kubernetes](kubernetes.md) · [Observability & SRE](observability-sre.md)
