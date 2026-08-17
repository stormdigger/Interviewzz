# 🔌 API Design

> An API is a contract. Once published, you cannot break it. Everything in this book is about designing a contract you can live with for years and evolve without breaking clients.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [REST — Done Properly](#2-rest)
3. [Resource Modeling](#3-resource-modeling)
4. [HTTP Semantics](#4-http-semantics)
5. [Pagination](#5-pagination)
6. [Filtering, Sorting, Sparse Fields](#6-filtering-sorting)
7. [Error Handling](#7-error-handling)
8. [Versioning](#8-versioning)
9. [Idempotency and Reliability](#9-idempotency)
10. [Authentication & Authorization](#10-auth)
11. [Rate Limiting](#11-rate-limiting)
12. [GraphQL](#12-graphql)
13. [gRPC](#13-grpc)
14. [Real-Time APIs](#14-real-time)
15. [Documentation and Contracts](#15-documentation)
16. [API Security Checklist](#16-security-checklist)
17. [Interview Section](#17-interview-section)
18. [Cheat Sheet](#18-cheat-sheet)

---

## 1. Mental Model

```
   An API design has four layers. Get them in this order.

   ┌─────────────────────────────────────────────────────┐
   │ 1. DOMAIN     What are the nouns? What can happen?  │
   │               (resources, state transitions)        │
   ├─────────────────────────────────────────────────────┤
   │ 2. CONTRACT   Shapes, types, required fields,       │
   │               errors, guarantees                    │
   ├─────────────────────────────────────────────────────┤
   │ 3. PROTOCOL   REST / GraphQL / gRPC / events        │
   │               (this is a LATE decision)             │
   ├─────────────────────────────────────────────────────┤
   │ 4. OPERATIONS Auth, rate limits, versioning,        │
   │               observability, SLAs                   │
   └─────────────────────────────────────────────────────┘
```

Most bad APIs come from starting at layer 3 — picking REST or GraphQL first and then bending the domain to fit.

### Choosing a protocol

```
                     Who is the consumer?
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
   Public / 3rd party   Your own frontend    Service-to-service
        │                   │                    │
        ▼                   ▼                    ▼
      REST              REST or GraphQL        gRPC
   (universal,        (GraphQL if many         (fast, typed,
    cacheable,         clients need             streaming,
    debuggable)        different shapes)        internal only)

   Plus, orthogonally:
   Real-time push?  → WebSocket (bidirectional) or SSE (server→client)
   Async workflow?  → Message queue + webhook/callback
```

---

## 2. REST

REST is an architectural style, not "JSON over HTTP." The constraints that actually matter in practice:

| Constraint | What it buys you |
|---|---|
| **Uniform interface** | Any client can talk to any server; tooling works |
| **Stateless** | Any server can handle any request → horizontal scaling |
| **Cacheable** | Responses declare cacheability → CDNs and proxies work |
| **Layered** | Insert proxies, gateways, caches transparently |
| Client-server | Independent evolution |
| Code on demand (optional) | Rarely used |

### The Richardson Maturity Model

```
   Level 0 ─ The swamp of POX
             POST /api  {"action": "getUser", "id": 1}
             One endpoint, HTTP as a tunnel. This is RPC in a trenchcoat.

   Level 1 ─ Resources
             POST /users/1   POST /orders/5
             Nouns exist, but everything is still POST.

   Level 2 ─ HTTP verbs + status codes        ⭐ where 95% of good APIs live
             GET /users/1 → 200
             DELETE /users/1 → 204
             POST /users → 201 + Location

   Level 3 ─ HATEOAS
             Responses include links to available next actions.
             Rarely worth the complexity except in long-lived
             public APIs (e.g. payment flows where the state
             machine is genuinely complex).
```

🏭 **Target Level 2.** Level 3 is theoretically pure and practically ignored by most clients, who hardcode URLs anyway.

---

## 3. Resource Modeling

### 3.1 Naming rules

```
   ✅  /users                  plural nouns for collections
   ✅  /users/42               singular item by ID
   ✅  /users/42/orders        nesting shows ownership
   ✅  /orders?user_id=42      flat alternative — prefer for deep nesting
   ✅  /users/me               well-known alias for the caller

   ❌  /getUsers               verbs — that's what the method is for
   ❌  /user                   inconsistent plurality
   ❌  /users/42/orders/7/items/3/reviews/9     too deep (>2 levels)
   ❌  /Users  /user_profiles  inconsistent casing (pick kebab-case)
```

**Nesting rule:** nest at most one level, and only when the child cannot exist without the parent. `/orders/7/items` is fine (items belong to an order). `/users/42/orders/7` is not — an order has its own ID, so `/orders/7` is enough.

### 3.2 Actions that aren't CRUD

Not every operation is create/read/update/delete. Three options:

```
   1. Model the action as a RESOURCE (best)
      POST /users/42/password-reset-requests
      POST /orders/7/refunds            → creates a refund resource
      POST /videos/9/transcode-jobs     → returns a job you can poll

   2. Sub-resource representing state
      PUT  /orders/7/status   {"status": "cancelled"}

   3. Controller pattern — an explicit verb, sparingly
      POST /orders/7/cancel
      POST /users/42/verify-email
```

Option 1 is preferable because the result is addressable: you can `GET /refunds/123` to check on it, and it's naturally idempotent-able and auditable.

### 3.3 Representation design

```jsonc
{
  "id": "usr_2Nx8kL",                    // prefixed opaque ID — not a raw int
  "type": "user",
  "email": "ada@example.com",
  "display_name": "Ada L.",              // consistent snake_case (or camel — pick one)
  "status": "active",                    // enum as a string, not a magic int
  "created_at": "2026-03-14T09:00:00Z",  // ISO 8601, ALWAYS UTC with Z
  "updated_at": "2026-08-01T11:30:00Z",
  "metadata": {},                        // extension point for clients
  "_links": {                            // optional
    "self": "/users/usr_2Nx8kL",
    "orders": "/users/usr_2Nx8kL/orders"
  }
}
```

**Rules learned the hard way:**

| Rule | Why |
|---|---|
| Prefixed string IDs (`usr_`, `ord_`) | Self-describing in logs; hides row counts; allows migration off integers |
| Never expose sequential integer IDs publicly | Leaks business volume; enables enumeration attacks |
| ISO 8601 UTC timestamps, always | Timezone bugs are endless otherwise |
| Money as integer minor units + currency | `{"amount": 1050, "currency": "USD"}` — never a float |
| Enums as strings | Readable, extensible, doesn't break when values are reordered |
| Wrap collections in an object | `{"data": [...], "meta": {...}}` — a bare array can't grow |
| Omit vs null: be consistent | Document which one means "absent" |
| Never return more than needed | Every field is a permanent commitment |

---

## 4. HTTP Semantics

### 4.1 Methods

| Method | Safe | Idempotent | Body | Use |
|---|---|---|---|---|
| `GET` | ✅ | ✅ | ❌ | Read. **Never** mutate. |
| `HEAD` | ✅ | ✅ | ❌ | Headers only — existence/size checks |
| `OPTIONS` | ✅ | ✅ | ❌ | CORS preflight, capability discovery |
| `POST` | ❌ | ❌ | ✅ | Create, or non-idempotent actions |
| `PUT` | ❌ | ✅ | ✅ | Full replace (or create at a known URI) |
| `PATCH` | ❌ | ❌* | ✅ | Partial update |
| `DELETE` | ❌ | ✅ | opt | Remove |

*A `PATCH` can be made idempotent by design (set absolute values, not increments).

```
   SAFE       = no observable state change (cacheable, prefetchable)
   IDEMPOTENT = doing it N times ≡ doing it once (safe to retry)
```

⚠️ **Safety matters operationally.** Browsers, proxies, and prefetchers will issue `GET` requests speculatively. A `GET /users/42/delete` endpoint *will* eventually be triggered by a crawler.

### 4.2 Status Codes

```
   2xx SUCCESS
     200 OK                 GET, PUT, PATCH with a body
     201 Created            POST that created a resource + Location header
     202 Accepted           Async — work queued, not done. Return a job URL.
     204 No Content         DELETE, or PUT with nothing to return
     206 Partial Content    Range requests

   3xx REDIRECTION
     301 Moved Permanently  Resource has a new canonical URL
     302 Found / 307 / 308  Temporary (307 preserves method; 302 historically didn't)
     304 Not Modified       Conditional GET hit — no body, saves bandwidth

   4xx CLIENT ERROR — the caller must change something
     400 Bad Request        Malformed syntax or invalid params
     401 Unauthorized       ⚠️ Actually means UNAUTHENTICATED — no/bad credentials
     403 Forbidden          Authenticated but not permitted
     404 Not Found          No such resource (also used to hide 403 for privacy)
     405 Method Not Allowed Must include an `Allow` header
     409 Conflict           State conflict — duplicate, version mismatch
     410 Gone               Deliberately removed, won't come back
     412 Precondition Failed  If-Match failed → optimistic locking
     413 Payload Too Large
     415 Unsupported Media Type
     422 Unprocessable Entity  Syntactically valid, semantically wrong ← validation
     428 Precondition Required  Force clients to use If-Match
     429 Too Many Requests  + Retry-After

   5xx SERVER ERROR — the caller did nothing wrong
     500 Internal Server Error   Unhandled — never leak stack traces
     502 Bad Gateway             Upstream returned garbage
     503 Service Unavailable     Overloaded/maintenance + Retry-After
     504 Gateway Timeout         Upstream too slow
```

**400 vs 422:** use 400 for malformed requests (bad JSON, missing required param, wrong type) and 422 for well-formed requests that fail business rules ("email already registered", "end date before start date"). Being consistent matters more than which convention you pick.

### 4.3 Essential Headers

```http
# Request
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
If-None-Match: "abc123"                 # conditional GET → 304
If-Match: "abc123"                      # optimistic lock → 412 on mismatch
Idempotency-Key: 550e8400-e29b-...      # safe retry for POST
X-Request-Id: <uuid>                    # trace correlation

# Response
Content-Type: application/json; charset=utf-8
ETag: "abc123"
Cache-Control: private, max-age=60
Location: /users/usr_2Nx8kL             # with 201
Retry-After: 30                         # with 429/503
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1723640400
Deprecation: true                       # RFC 8594
Sunset: Wed, 01 Jan 2027 00:00:00 GMT
```

---

## 5. Pagination

### 5.1 Offset pagination

```http
GET /users?page=3&per_page=20
```

```jsonc
{
  "data": [...],
  "meta": { "page": 3, "per_page": 20, "total": 4832, "total_pages": 242 }
}
```

| Pro | Con |
|---|---|
| Jump to any page | **O(offset)** — `OFFSET 100000` scans and discards 100k rows |
| Total count available | Items shift between pages when data changes |
| Trivial to implement | Duplicates and skips under concurrent writes |

### 5.2 Cursor (keyset) pagination ⭐

```http
GET /users?limit=20&after=eyJpZCI6MTIzLCJjcmVhdGVkIjoiMjAyNi0wMy0xNCJ9
```

```jsonc
{
  "data": [...],
  "page_info": {
    "has_next_page": true,
    "end_cursor": "eyJpZCI6MTQzfQ=="
  }
}
```

```sql
-- Constant time regardless of depth, given an index on (created_at, id)
SELECT * FROM users
WHERE (created_at, id) < ($cursor_created, $cursor_id)   -- row-value comparison
ORDER BY created_at DESC, id DESC
LIMIT 21;                                                -- fetch n+1 to know has_next
```

| Pro | Con |
|---|---|
| **O(1)** at any depth | No jumping to page N |
| Stable under concurrent inserts | Total count is expensive/omitted |
| Required for infinite scroll | Cursor must encode the full sort key |

🏭 **Rule:** cursor pagination for anything that can grow large or is sorted by time. Offset only for small, bounded, admin-facing lists where page jumping matters.

⚠️ Always include a **tiebreaker** in the sort key. Sorting by `created_at` alone with duplicate timestamps silently drops or repeats rows.

---

## 6. Filtering, Sorting

```http
GET /orders
  ?status=paid,shipped              # OR within a field (CSV)
  &created_at[gte]=2026-01-01       # operators in brackets
  &total[lt]=10000
  &q=laptop                         # full-text search
  &sort=-created_at,total           # `-` prefix = descending
  &fields=id,total,status           # sparse fieldsets — save bandwidth
  &include=customer,items           # related resources — avoid N+1 round trips
  &limit=20&after=<cursor>
```

⚠️ **Never build SQL from query params directly.** Whitelist every filterable field, every operator, and every sortable column. An unbounded `?sort=` is both an injection vector and a way to trigger a full table scan on an unindexed column.

```python
ALLOWED_SORTS = {"created_at", "total", "status"}
ALLOWED_FILTERS = {"status": str, "total": int, "created_at": date}
MAX_LIMIT = 100
```

**Include/expand** is how you avoid the client-side N+1 problem in REST:

```jsonc
// GET /orders/7?include=customer,items
{
  "data": { "id": "ord_7", "total": 4500, "customer_id": "usr_1" },
  "included": [
    { "type": "customer", "id": "usr_1", "name": "Ada" },
    { "type": "item", "id": "itm_9", "sku": "ABC" }
  ]
}
```

---

## 7. Error Handling

Use **RFC 9457 Problem Details** — a standard shape so clients can write one error handler.

```jsonc
// HTTP/1.1 422 Unprocessable Entity
// Content-Type: application/problem+json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "The request contained 2 invalid fields.",
  "instance": "/orders",
  "request_id": "req_01HXYZ",                  // ⭐ for support tickets
  "errors": [
    { "field": "email",    "code": "invalid_format", "message": "Must be a valid email" },
    { "field": "quantity", "code": "out_of_range",   "message": "Must be between 1 and 99" }
  ]
}
```

**Error design rules:**

| Rule | Reason |
|---|---|
| Machine-readable `code` **and** human `message` | Clients branch on codes; humans read messages |
| Return **all** validation errors, not just the first | Otherwise the user fixes one field per round trip |
| Include a `request_id` matching your logs | Turns a support ticket into a 30-second lookup |
| Never leak stack traces, SQL, or internal hostnames | Reconnaissance for attackers |
| Document every error code | Clients need to know what they must handle |
| Same shape for every error | One handler, not twenty |

```python
# Never do this
except Exception as e:
    return {"error": str(e)}, 500      # ❌ leaks internals

# Do this
except Exception as e:
    request_id = get_request_id()
    logger.exception("Unhandled error", extra={"request_id": request_id})
    return problem(500, "internal_error",
                   "An unexpected error occurred.", request_id=request_id)
```

---

## 8. Versioning

| Strategy | Example | Pro | Con |
|---|---|---|---|
| **URL path** | `/v1/users` | Obvious, cacheable, easy routing | "Not RESTful"; version sprawl |
| Header | `Accept: application/vnd.api.v2+json` | Clean URLs | Hard to test in a browser; cache key complexity |
| Query param | `/users?version=2` | Simple | Easily forgotten; pollutes params |
| **Date-based** | `Stripe-Version: 2026-03-14` | Fine-grained; per-account pinning | Complex server-side transform layer |

🏭 **Recommendation:** URL path versioning (`/v1/`) for most APIs — it's boring, visible, and works with every tool. Date-based versioning if you're Stripe-scale and need to ship breaking changes continuously without forcing migrations.

### What is a breaking change?

```
   ✅ NON-BREAKING (safe to ship anytime)
      • Adding a new endpoint
      • Adding an OPTIONAL request field
      • Adding a field to a response       ← clients must ignore unknown fields
      • Adding a new enum value ⚠️ only if clients handle unknowns gracefully
      • Relaxing a validation rule
      • Adding a new optional header

   ❌ BREAKING (needs a new version)
      • Removing or renaming any field
      • Changing a field's type or format
      • Making an optional field required
      • Changing default values
      • Changing status codes or error shapes
      • Tightening validation
      • Changing pagination semantics
      • Changing sort order
```

⚠️ The "adding an enum value is safe" claim depends entirely on your clients. Many deserializers throw on unknown enum values. Document from day one that clients must tolerate unknown enums and unknown fields — this is the **tolerant reader** pattern and it's what makes evolution possible.

### Deprecation process

```
   1. ANNOUNCE   Docs, changelog, email. Give a date.
   2. HEADERS    Deprecation: true
                 Sunset: Wed, 01 Jan 2027 00:00:00 GMT
                 Link: <https://docs/migrate-v2>; rel="deprecation"
   3. MEASURE    Log usage per client/version. Who is still on v1?
   4. NUDGE      Contact remaining heavy users directly.
   5. BROWNOUT   Return 410 for short windows to surface hidden dependencies.
   6. REMOVE     Only after usage is ~zero and the sunset date has passed.
```

---

## 9. Idempotency

Networks fail *after* the server has processed the request. The client cannot tell "never arrived" from "arrived, response lost." Without idempotency, retrying a payment charges twice.

```
   Client            Network            Server
     │  POST /charge  ──────────────────▶│
     │                                   │  charge created ✅
     │                 ✗ response lost   │
     │  (timeout)                        │
     │  retry POST ───────────────────── ▶│  charge created AGAIN ❌❌
```

### The Idempotency-Key pattern

```http
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{"amount": 5000, "currency": "USD", "source": "card_1"}
```

```python
async def create_payment(body: PaymentIn, idempotency_key: str = Header(...)):
    fingerprint = sha256(canonical_json(body))

    # Atomic claim — the unique index does the concurrency control
    try:
        async with db.transaction():
            await db.execute(
                "INSERT INTO idempotency_keys (key, fingerprint, status) "
                "VALUES ($1, $2, 'in_progress')", idempotency_key, fingerprint)
    except UniqueViolation:
        record = await db.fetchone(
            "SELECT * FROM idempotency_keys WHERE key = $1", idempotency_key)

        if record.fingerprint != fingerprint:
            raise Conflict("Idempotency-Key reused with a different payload")

        if record.status == 'in_progress':
            raise Conflict("Request in progress, retry shortly")   # 409

        return json.loads(record.response_body)     # ⭐ replay the stored response

    # First time through — do the real work
    async with db.transaction():
        payment = await charge(body)
        await db.execute(
            "UPDATE idempotency_keys SET status='completed', response_body=$2, "
            "response_status=201 WHERE key=$1", idempotency_key, json.dumps(payment))
    return payment
```

**Design points that matter:**

1. **Store the response**, not just a "seen" flag — the retry must get the same body and status.
2. **Fingerprint the payload.** Same key with a different body is a client bug; return 409 rather than silently returning the wrong result.
3. **Handle the in-flight case.** Two concurrent retries must not both execute.
4. **Expire keys** after ~24 hours.
5. Keys must be **client-generated** (UUID v4) — the client is the only one who knows two requests are the same intent.

### Optimistic concurrency

```http
GET /documents/7          → ETag: "v12"
PUT /documents/7          → If-Match: "v12"
                          → 200 if still v12, 412 Precondition Failed if someone else edited
```

This turns lost-update races into an explicit client-visible conflict.

---

## 10. Auth

```
   AUTHENTICATION — who are you?      AUTHORIZATION — what may you do?
   ───────────────────────────        ─────────────────────────────
   401 if it fails                    403 if it fails
```

### 10.1 Token comparison

| Scheme | Stateless | Revocable | Use |
|---|---|---|---|
| Session cookie | ❌ (server store) | ✅ instantly | Browser apps, same-site |
| JWT access token | ✅ | ❌ until expiry | Short-lived (5-15 min) access |
| Opaque token + introspection | ❌ | ✅ | When revocation matters more than scale |
| API key | ❌ | ✅ | Server-to-server, machine clients |
| mTLS | ✅ | ✅ (cert revocation) | Internal service mesh |

### 10.2 The JWT reality check

```
   header.payload.signature      — base64url, NOT encrypted. Anyone can read it.

   ⚠️ You cannot revoke a JWT before expiry without server state,
      which defeats the whole point of using one.
```

**The workable pattern:**

```
   Access token   JWT, 5-15 min, no revocation needed (expires fast)
   Refresh token  opaque, in the DB, long-lived, revocable, ROTATED on use
   
   Refresh rotation with reuse detection:
     refresh #1 used → issue new pair, mark #1 as used
     refresh #1 used AGAIN → someone stole it → revoke the entire family
```

**JWT security rules:**

```
   ✅ Verify the signature with a pinned algorithm (never trust the `alg` header)
   ✅ Reject `alg: none` explicitly
   ✅ Validate iss, aud, exp, nbf every time
   ✅ Keep them short-lived
   ✅ Store in an HttpOnly, Secure, SameSite cookie for browsers
   ❌ Never put secrets or PII in the payload — it's readable
   ❌ Never store in localStorage (XSS reads it instantly)
   ❌ Never use JWTs as sessions if you need instant logout
```

### 10.3 OAuth 2.1 / OIDC

```
   ┌────────┐  1. redirect + PKCE challenge  ┌──────────────┐
   │ Client │──────────────────────────────▶│ Authorization│
   │        │                                │    Server    │
   │        │  2. user authenticates         │              │
   │        │◀── 3. authorization code ──────│              │
   │        │                                │              │
   │        │  4. code + PKCE verifier ─────▶│              │
   │        │◀── 5. access + refresh + id ───│              │
   └───┬────┘                                └──────────────┘
       │  6. Authorization: Bearer <access>
       ▼
   ┌──────────────┐
   │Resource Server│ verifies signature/introspects, checks scopes
   └──────────────┘
```

**Authorization Code + PKCE** is the only flow you should use for new work. Implicit flow is dead. Password grant is dead. PKCE is mandatory even for confidential clients in OAuth 2.1.

OAuth is for **delegated authorization**. OIDC adds an `id_token` for **authentication**. Using a raw access token to identify a user is a common and dangerous mistake — access tokens aren't audience-bound to your app.

---

## 11. Rate Limiting

```
   ALGORITHM COMPARISON

   Fixed window     |████████|        |████████|
                    ⚠️ 2× burst at the boundary
                    
   Sliding log      exact, but stores every timestamp — memory heavy
   
   Sliding window   weighted blend of current + previous window — good approximation
   
   Token bucket     ┌─────┐  refill at rate r, capacity b
                    │ ▓▓▓ │  ⭐ allows controlled bursts, simple, standard
                    └──┬──┘
                       ▼ 1 token per request
                       
   Leaky bucket     smooths output to a constant rate (queue-based)
```

```lua
-- Redis token bucket, atomic via Lua
local key, rate, capacity, now, requested = KEYS[1], tonumber(ARGV[1]),
      tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])

local b = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(b[1]) or capacity
local ts = tonumber(b[2]) or now

tokens = math.min(capacity, tokens + (now - ts) * rate)   -- refill

local allowed = tokens >= requested
if allowed then tokens = tokens - requested end

redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
return { allowed and 1 or 0, tokens }
```

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 1000
RateLimit-Remaining: 0
RateLimit-Reset: 30
```

**What to key on, in order of preference:** authenticated user/API key > IP + user agent > IP alone. IP alone punishes everyone behind a corporate NAT.

Apply different limits per endpoint cost — a search endpoint costs far more than a health check. And always return `Retry-After` so well-behaved clients back off correctly instead of hammering.

---

## 12. GraphQL

```graphql
type Query {
  user(id: ID!): User
  orders(first: Int, after: String, filter: OrderFilter): OrderConnection!
}

type User {
  id: ID!
  email: String!
  orders(first: Int = 10): OrderConnection!
}

type OrderConnection {          # Relay cursor spec
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}
```

```graphql
query {
  user(id: "1") {
    email
    orders(first: 5) {
      edges { node { id total items { sku } } }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

### 12.1 When GraphQL wins

| Wins | Loses |
|---|---|
| Many clients needing different data shapes | HTTP caching (everything is POST /graphql) |
| Deeply nested, related data in one round trip | Simple CRUD (needless complexity) |
| Rapid frontend iteration without backend changes | File uploads (needs an extension) |
| Strong typed schema + introspection tooling | Rate limiting (query cost ≠ request count) |
| Federating multiple services | Observability (one endpoint hides everything) |

### 12.2 The three mandatory defenses

Every GraphQL API needs these or it's a DoS target:

```js
// 1. Depth limiting — stops recursive nesting bombs
depthLimit(7)

// 2. Cost analysis — assign complexity, reject expensive queries
costAnalysis({ maximumCost: 1000, defaultCost: 1,
               variables: { first: (n) => n } })

// 3. Persisted queries — only allow pre-registered query hashes in production
//    Also fixes HTTP caching: GET /graphql?sha256=...
```

The attack it prevents:

```graphql
# Without depth limiting, this is an exponential explosion
{ user { friends { friends { friends { friends { friends { id }}}}}} }
```

### 12.3 The N+1 problem and DataLoader

```js
// ❌ Resolving 100 orders each fetching its user = 101 queries
const resolvers = {
  Order: { user: (order) => db.user.findById(order.userId) },
};

// ✅ DataLoader batches within one tick and dedupes
const userLoader = new DataLoader(async (ids) => {
  const users = await db.user.findMany({ where: { id: { in: ids } } });
  const byId = new Map(users.map(u => [u.id, u]));
  return ids.map(id => byId.get(id) ?? null);      // MUST preserve order
});

const resolvers = {
  Order: { user: (order, _, ctx) => ctx.loaders.user.load(order.userId) },
};
```

⚠️ Create loaders **per request**, never globally — a global loader caches across users and leaks data between them.

---

## 13. gRPC

```protobuf
syntax = "proto3";
package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);        // server stream
  rpc UploadEvents(stream Event) returns (UploadSummary);          // client stream
  rpc Chat(stream Message) returns (stream Message);               // bidirectional
}

message Order {
  string id = 1;
  string customer_id = 2;
  int64 total_cents = 3;
  Status status = 4;
  google.protobuf.Timestamp created_at = 5;

  enum Status {
    STATUS_UNSPECIFIED = 0;      // ⭐ field 0 must be UNSPECIFIED
    STATUS_PENDING = 1;
    STATUS_PAID = 2;
  }
}
```

### Protobuf evolution rules

```
   ✅ Add new fields with NEW tag numbers
   ✅ Rename a field (the tag number is the identity, not the name)
   ✅ Add enum values
   ❌ NEVER reuse a tag number — old clients will misinterpret data
   ❌ NEVER change a field's type
   ❌ NEVER change a tag number

   When removing a field:
     reserved 4, 7 to 9;
     reserved "old_field_name";
```

| | REST/JSON | gRPC/Protobuf |
|---|---|---|
| Payload size | Baseline | ~30-50% smaller |
| Speed | Baseline | 2-10× faster (binary, HTTP/2) |
| Streaming | SSE/WebSocket bolt-on | Native, four modes |
| Browser support | ✅ | Needs grpc-web proxy |
| Debuggability | curl works | Needs grpcurl/tooling |
| Schema | OpenAPI (optional) | Mandatory, enforced |

🏭 **Use gRPC for internal service-to-service.** Use REST at the public edge. This is the standard architecture at most large companies.

---

## 14. Real-Time

```
   POLLING            client asks repeatedly           simple, wasteful
                      GET /messages?since=X

   LONG POLLING       server holds the request open    ok fallback, ties up connections
                      until data or timeout

   SSE                one-way server→client            ⭐ simplest for notifications
                      text/event-stream, auto-reconnect, works over plain HTTP

   WEBSOCKET          full duplex, persistent          chat, collaboration, games
                      needs its own auth, scaling, heartbeats

   WEBHOOK            server→server HTTP callback      integrations, async results
```

### SSE — underrated

```js
// Server
res.writeHead(200, {
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  'Connection': 'keep-alive',
  'X-Accel-Buffering': 'no',            // disable nginx buffering
});
res.write(`id: ${eventId}\nevent: update\ndata: ${JSON.stringify(payload)}\n\n`);

// Client — reconnects automatically, resumes with Last-Event-ID
const es = new EventSource('/stream');
es.addEventListener('update', (e) => handle(JSON.parse(e.data)));
```

SSE handles reconnection and event IDs for you, works through most proxies, and needs no special infrastructure. Choose it over WebSocket unless you genuinely need client→server streaming.

### Webhook design

Webhooks are an API you provide *to* someone else's server. They need:

```
   ✅ HMAC signature over the raw body + timestamp
        X-Signature: t=1723640400,v1=5257a8...
   ✅ Timestamp in the signed payload (replay protection, ±5 min tolerance)
   ✅ Retry with exponential backoff (e.g. 1m, 5m, 30m, 2h, 12h)
   ✅ At-least-once delivery → consumers MUST dedupe on event_id
   ✅ Fast ACK — respond 2xx immediately, process async
   ✅ A dead-letter view + manual replay in your dashboard
   ✅ Documented event catalog with versioned payloads
```

```python
def verify(raw_body: bytes, header: str, secret: str) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    ts, sig = parts["t"], parts["v1"]
    if abs(time.time() - int(ts)) > 300:            # replay window
        return False
    expected = hmac.new(secret.encode(), f"{ts}.".encode() + raw_body,
                        hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, sig)       # ⭐ constant-time compare
```

⚠️ Sign the **raw bytes**, before any JSON parsing. Re-serializing changes whitespace and key order, breaking the signature.

---

## 15. Documentation

```yaml
# OpenAPI 3.1 — generate this from code, or generate code from it. Never both by hand.
openapi: 3.1.0
info:
  title: Orders API
  version: 1.0.0
paths:
  /orders:
    post:
      operationId: createOrder
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          schema: { type: string, format: uuid }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateOrder' }
      responses:
        '201':
          description: Created
          headers:
            Location: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '422':
          $ref: '#/components/responses/ValidationError'
```

**Documentation that actually gets used:**

| Element | Why |
|---|---|
| Runnable examples (curl + 2-3 languages) | People copy-paste before reading |
| A "getting started in 5 minutes" path | First impression decides adoption |
| Every error code listed with a cause and fix | The most-visited page after launch |
| Changelog with dates | Trust |
| Sandbox environment with test data | Lets people try before signing up |
| Generated from the same source as the code | Docs that drift are worse than none |

**Contract testing** (Pact, Spring Cloud Contract) verifies consumer expectations against the provider in CI — this catches breaking changes before deploy, which OpenAPI alone does not.

---

## 16. Security Checklist

```
   TRANSPORT
   □ HTTPS only, HSTS with preload
   □ TLS 1.2 minimum, prefer 1.3

   AUTH
   □ Every endpoint authenticated unless deliberately public
   □ Authorization checked on the RESOURCE, not just the route
     (the #1 API vulnerability: BOLA / IDOR)
   □ Short-lived access tokens + rotating refresh tokens
   □ No secrets in URLs (they land in logs, referrers, browser history)

   INPUT
   □ Validate everything against a schema — allowlist, not denylist
   □ Reject unknown fields (or document that you ignore them)
   □ Enforce max body size, max array length, max string length
   □ Parameterized queries only — never string-concatenate SQL
   □ Validate content-type; don't guess

   OUTPUT
   □ No stack traces, SQL, or internal hostnames in errors
   □ Filter fields by the caller's permissions (mass assignment protection)
   □ Set X-Content-Type-Options: nosniff

   RATE / ABUSE
   □ Rate limits per user AND per IP
   □ Stricter limits on auth endpoints (credential stuffing)
   □ Pagination limits enforced server-side (cap `limit`)
   □ Query cost limits for GraphQL

   OPERATIONS
   □ Request IDs on everything, propagated to logs and traces
   □ Audit log for sensitive actions (who, what, when, from where)
   □ CORS allowlist — never reflect arbitrary Origin with credentials
   □ Dependency scanning + a documented patch SLA
```

⚠️ **BOLA (Broken Object Level Authorization)** is #1 on the OWASP API Top 10 and causes most real API breaches:

```python
# ❌ Authenticated, but not authorized for THIS object
@app.get("/orders/{order_id}")
async def get_order(order_id: str, user=Depends(current_user)):
    return await db.order.find(order_id)     # any user can read any order!

# ✅ Scope the query by ownership
@app.get("/orders/{order_id}")
async def get_order(order_id: str, user=Depends(current_user)):
    order = await db.order.find_one(id=order_id, customer_id=user.id)
    if not order:
        raise HTTPException(404)             # 404 not 403 — don't confirm existence
    return order
```

---

## 17. Interview Section

<details>
<summary><b>Q1. Design a REST API for a ride-hailing service.</b></summary>

I'd start with the domain: riders, drivers, ride requests, rides, and locations. The key insight is that a ride request and a ride are different resources — a request may never become a ride.

```
POST   /ride-requests          → 202 Accepted + a request resource to poll
GET    /ride-requests/{id}     → status: searching | matched | expired
DELETE /ride-requests/{id}     → cancel while searching

GET    /rides/{id}             → the actual trip
POST   /rides/{id}/cancel      → controller action, has business rules
PATCH  /rides/{id}             → driver updates status (arrived, started, completed)

PUT    /drivers/me/location    → high-frequency, idempotent, last-write-wins
GET    /drivers/me/rides?status=active
```

Matching is asynchronous, so `POST /ride-requests` returns 202 with a resource to poll — or better, the client subscribes over WebSocket for the match event and location updates, since polling for a ~10-second matching window at scale is wasteful.

Location updates use `PUT` because they're idempotent replacements, and they'd bypass the main API entirely at scale — a dedicated high-throughput ingestion path.

I'd add `Idempotency-Key` on ride creation so a network retry doesn't book two rides, and cursor pagination on ride history since it grows unboundedly.
</details>

<details>
<summary><b>Q2. PUT vs PATCH vs POST.</b></summary>

`POST` creates a subordinate resource or performs a non-idempotent action. The server usually assigns the URL, returned in `Location`.

`PUT` replaces the resource entirely at a known URL. It's idempotent — sending the same body twice leaves the same state. Any field you omit should be treated as cleared, which is why partial updates via PUT are a common bug.

`PATCH` applies a partial modification. It's not inherently idempotent — `{"op": "increment"}` isn't — but you can design it to be by using absolute values only. Two standard formats exist: JSON Merge Patch (RFC 7386, simple, but can't distinguish "set to null" from "remove") and JSON Patch (RFC 6902, an operation list, more expressive but verbose).

In practice most APIs use `PATCH` with merge semantics and document that `null` means "clear."
</details>

<details>
<summary><b>Q3. How do you handle API versioning without breaking clients?</b></summary>

First, minimize the need. Design responses as objects that can grow, document the tolerant reader contract — clients must ignore unknown fields and handle unknown enum values — and add optional fields rather than changing existing ones.

When a breaking change is unavoidable, I'd use URL path versioning for its visibility and tooling support. Run versions in parallel with a shared core so you're not maintaining two codebases — usually a translation layer that maps the old contract onto the new domain model.

Then run a real deprecation process: announce with a date, emit `Deprecation` and `Sunset` headers, instrument usage per client so you know who's affected, contact heavy users directly, do brownouts to surface forgotten integrations, and only then remove.

Stripe's date-based versioning is the gold standard for high-volume public APIs — clients pin a date and a transformation chain upgrades old requests — but the operational cost is significant and only worth it at that scale.
</details>

<details>
<summary><b>Q4. Explain idempotency and how you'd implement it for payments.</b></summary>

Idempotency means performing an operation N times has the same effect as performing it once. It matters because a client that times out cannot distinguish "request never arrived" from "request succeeded but the response was lost." Without idempotency, the safe retry charges the customer twice.

The implementation is a client-generated `Idempotency-Key` header, typically a UUID. Server side, I'd insert the key into a table with a unique constraint inside a transaction — the unique index does the concurrency control, so two simultaneous retries can't both proceed.

Three details people miss. First, you must store the full response and replay it, not just record that you've seen the key — the retry needs the same body and status. Second, fingerprint the request payload and return 409 if the same key arrives with different content, because that's a client bug you want surfaced. Third, handle the in-flight case: a retry arriving while the original is still processing should get a 409 telling it to retry shortly, not execute concurrently.

Keys expire after about 24 hours to bound storage.
</details>

<details>
<summary><b>Q5. Offset vs cursor pagination.</b></summary>

Offset uses `LIMIT/OFFSET`. It's simple and lets you jump to page N and show a total count, but it's O(offset) — the database scans and discards every skipped row, so deep pages get progressively slower. It's also unstable: if rows are inserted or deleted while a user pages, they'll see duplicates or miss items.

Cursor pagination encodes the last row's sort key and uses a `WHERE (sort_key) < cursor` comparison. With a matching index it's constant time at any depth and stable under concurrent writes, because the cursor is a position in the data rather than a count.

The tradeoffs: no page jumping, and total counts are expensive so most cursor APIs omit them.

The critical implementation detail is including a unique tiebreaker in the sort key. Sorting by `created_at` alone with duplicate timestamps will silently skip or repeat rows at page boundaries — use `(created_at, id)` as a row-value comparison.

I'd default to cursor pagination for anything unbounded or time-ordered, and use offset only for small admin lists where page numbers are a real requirement.
</details>

<details>
<summary><b>Q6. REST vs GraphQL vs gRPC.</b></summary>

REST for public APIs and anything a third party consumes. It's universally understood, works with every tool, caches at the HTTP layer for free, and is debuggable with curl.

GraphQL when you have many clients needing different shapes of the same data — a web app, iOS, Android, and partner integrations all with different needs. It eliminates over-fetching and lets frontend teams iterate without backend changes. The costs are real though: you lose HTTP caching, rate limiting becomes cost analysis rather than request counting, and you must add depth limits and persisted queries or you've built a DoS endpoint.

gRPC for internal service-to-service. Binary protobuf over HTTP/2 is substantially faster and smaller, the schema is mandatory and enforced, and streaming is native in all four directions. It needs a proxy for browsers, which is why the common architecture is gRPC internally and REST at the edge.

The decision is mostly about who the consumer is, not about which is technically superior.
</details>

<details>
<summary><b>Q7. What's the most common API security vulnerability?</b></summary>

Broken Object Level Authorization — BOLA, also called IDOR. It's #1 on the OWASP API Top 10 and behind most real API breaches.

It happens when an endpoint authenticates the caller but doesn't verify they're authorized for the specific object. `GET /orders/{id}` checks you have a valid token but fetches the order by ID alone, so any authenticated user can read anyone's order by changing the number.

The fix is to scope every query by ownership rather than checking after the fetch: `find_one(id=order_id, customer_id=current_user.id)`. Return 404 rather than 403 so you don't confirm the resource exists.

It's insidious because it's invisible in testing — everything works fine when you only test with your own data. It needs an architectural answer: a data access layer where you cannot fetch a resource without a subject, plus automated tests that attempt cross-tenant access.
</details>

<details>
<summary><b>Q8. How do you design error responses?</b></summary>

One consistent shape everywhere, so clients write one handler. RFC 9457 Problem Details is the standard: `type`, `title`, `status`, `detail`, `instance`, plus extensions.

Every error needs both a stable machine-readable code that clients branch on and a human-readable message. The code is part of your contract and can't change; the message can.

Return all validation errors at once, not just the first, or the user fixes one field per round trip. Include a request ID that matches your logs — that turns a support ticket from an investigation into a lookup.

And never leak internals. Stack traces, SQL fragments, and internal hostnames in a 500 response are free reconnaissance. Log the detail server-side with the request ID, return a generic message.
</details>

<details>
<summary><b>Q9. How would you rate limit an API?</b></summary>

Token bucket, implemented in Redis with a Lua script so the check-and-decrement is atomic. Token bucket is the right default because it allows controlled bursts — real traffic is bursty, and a strict rate limiter rejects legitimate usage patterns.

Key on the authenticated user or API key first, falling back to IP only for unauthenticated traffic, because IP alone punishes everyone behind a corporate NAT while barely inconveniencing an attacker with a proxy pool.

Different limits per endpoint by cost — search and report generation are far more expensive than a health check. And stricter limits on auth endpoints specifically, since that's where credential stuffing happens.

On the response side, always return `Retry-After` plus the `RateLimit-*` headers so well-behaved clients back off correctly instead of retrying immediately and making it worse.

At scale you'd also want this at the edge — a gateway or CDN — so rejected traffic never reaches your application servers.
</details>

<details>
<summary><b>Q10. Design a webhook system.</b></summary>

Webhooks are an API you're providing to someone else's server, which inverts most assumptions — you can't control their availability, their latency, or their correctness.

Delivery is at-least-once with exponential backoff over a long window, something like 1 minute to 12 hours across several attempts. Every event carries a unique `event_id` and consumers are documented as needing to dedupe on it, because at-least-once means duplicates will happen.

Security is HMAC-SHA256 over a timestamp concatenated with the raw request body, sent in a header. The timestamp is inside the signed payload with a five-minute tolerance window to prevent replay. Sign the raw bytes before parsing — re-serializing JSON changes whitespace and key order and breaks verification.

Operationally you need a dead-letter queue with a dashboard where customers can see failed deliveries and manually replay them, plus automatic disabling of endpoints that fail continuously so you're not spending resources on a dead URL for months.

For high volume the sender should be a queue consumer, not inline with the triggering request — a slow consumer must never slow down your own API.
</details>

---

## 18. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                        API DESIGN — ONE PAGE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ORDER: domain → contract → protocol → operations                     ║
║ REST(public) · GraphQL(many client shapes) · gRPC(internal)          ║
╠══════════════════════════════════════════════════════════════════════╣
║ METHODS  safe=no state change · idempotent=retry-safe                ║
║   GET(s,i) PUT(i) DELETE(i) | POST(neither) PATCH(design it in)      ║
╠══════════════════════════════════════════════════════════════════════╣
║ CODES 201+Location · 202 async · 204 delete                          ║
║   401=unauthenticated 403=unauthorized 404=hide existence            ║
║   409 conflict · 412 If-Match failed · 422 validation · 429+Retry-After║
╠══════════════════════════════════════════════════════════════════════╣
║ PAGINATION: cursor by default (O(1), stable)                         ║
║   ALWAYS include a unique tiebreaker in the sort key                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ IDEMPOTENCY: client UUID key · unique index claim · STORE the        ║
║   response and replay it · fingerprint payload → 409 on mismatch     ║
╠══════════════════════════════════════════════════════════════════════╣
║ VERSIONING: /v1/ path · tolerant reader (ignore unknown fields+enums)║
║   breaking = remove/rename/retype/require/reorder                    ║
║   Deprecation + Sunset headers → measure → brownout → remove         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ERRORS: RFC 9457 · machine code + human message · ALL validation     ║
║   errors at once · request_id · never leak internals                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ SECURITY: #1 is BOLA — authorize the OBJECT, not just the route      ║
║   scope every query by owner · 404 not 403 · allowlist validation    ║
║   short JWT + rotating refresh · never secrets in URLs               ║
╠══════════════════════════════════════════════════════════════════════╣
║ GRAPHQL MUST HAVE: depth limit + cost analysis + persisted queries   ║
║   DataLoader per REQUEST (never global)                              ║
║ WEBHOOKS: HMAC(timestamp + RAW body) · retry backoff · dedupe by id  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [FastAPI](fastapi.md) · [Spring Boot](spring-boot.md) · [Databases](databases.md) · [AppSec](../07-security/appsec.md) · [System Design](../05-system-design/00-fundamentals.md)
