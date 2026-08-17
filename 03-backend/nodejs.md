# 🟢 Node.js Backend

> Node is a single-threaded event loop wrapped around libuv. Every performance characteristic, every failure mode, and every best practice follows from that one fact.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Event Loop in Node](#2-event-loop)
3. [Blocking the Loop](#3-blocking-the-loop)
4. [Streams](#4-streams)
5. [Buffers and Binary Data](#5-buffers)
6. [Modules](#6-modules)
7. [Error Handling](#7-error-handling)
8. [Scaling: Cluster and Worker Threads](#8-scaling)
9. [HTTP Servers and Frameworks](#9-http-frameworks)
10. [Database Access](#10-database-access)
11. [Security](#11-security)
12. [Performance and Profiling](#12-performance)
13. [Observability](#13-observability)
14. [Testing](#14-testing)
15. [Production](#15-production)
16. [Interview Section](#16-interview-section)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Mental Model

```
   ┌──────────────────────────────────────────────────────────────────┐
   │                          NODE.JS                                 │
   │                                                                  │
   │   ┌──────────────────┐          ┌──────────────────────────┐     │
   │   │       V8         │          │         libuv            │     │
   │   │  JS execution    │◀────────▶│  Event loop              │     │
   │   │  Heap, GC        │          │  Thread pool (default 4) │     │
   │   │  JIT             │          │  epoll/kqueue/IOCP       │     │
   │   └──────────────────┘          └──────────────────────────┘     │
   │            │                                 │                   │
   │   ┌────────▼─────────────────────────────────▼───────────────┐   │
   │   │              Node C++ bindings + core modules            │   │
   │   │   fs · net · http · crypto · stream · worker_threads     │   │
   │   └──────────────────────────────────────────────────────────┘   │
   │                                                                  │
   │   ONE JavaScript thread. Concurrency comes from NOT WAITING.     │
   └──────────────────────────────────────────────────────────────────┘
```

🧠 **Node is not fast because it's multi-threaded. It's fast because it never waits.** While one request waits on a database, the thread serves a thousand others. The corollary is brutal: any CPU-bound work stalls *every* concurrent request.

**What runs where:**

```
   Your JavaScript          → the single main thread
   Network I/O              → OS kernel (epoll/kqueue), truly async
   File system I/O          → libuv THREAD POOL (default 4 threads)
   DNS lookup (getaddrinfo) → libuv thread pool  ⚠️ surprising
   crypto.pbkdf2 / scrypt   → libuv thread pool
   zlib compression         → libuv thread pool
```

⚠️ The thread pool defaults to **4 threads**. Heavy file I/O, DNS, or crypto will queue behind each other. `UV_THREADPOOL_SIZE=16` is a common production tuning.

---

## 2. Event Loop

```
   ┌───────────────────────────┐
┌─▶│         timers            │  setTimeout / setInterval callbacks
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
│  │    pending callbacks      │  deferred I/O errors (e.g. TCP ECONNREFUSED)
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
│  │     idle, prepare         │  internal
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐      ┌─────────────────┐
│  │          poll             │◀─────│  incoming I/O   │
│  │  execute I/O callbacks;   │      └─────────────────┘
│  │  MAY BLOCK here waiting   │
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
│  │          check            │  setImmediate callbacks
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
└──│      close callbacks      │  socket.on('close')
   └───────────────────────────┘

   ⭐ Between EVERY phase (and between each callback):
      1. drain process.nextTick queue    (HIGHEST priority)
      2. drain microtask queue           (promises)
```

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
console.log('sync');

// sync, nextTick, promise, then timeout/immediate in NONDETERMINISTIC order
```

The last part surprises people: from the main module, whether `setTimeout(0)` or `setImmediate` fires first depends on how long process startup took relative to the 1ms timer threshold. **Inside an I/O callback**, `setImmediate` always wins, because you're already past the poll phase.

⚠️ **`process.nextTick` starves the loop if recursive.** It runs before promises and before the loop advances, so an infinite `nextTick` chain freezes everything permanently. `setImmediate` yields properly.

---

## 3. Blocking the Loop

```
   ❌ THE #1 NODE PRODUCTION BUG

   app.get('/report', (req, res) => {
     const data = fs.readFileSync('huge.json');       // blocks
     const parsed = JSON.parse(data);                 // blocks (CPU)
     const result = heavyTransform(parsed);           // blocks (CPU)
     res.json(result);
   });

   500ms of blocking × 100 concurrent requests
   = the 100th request waits 50 SECONDS
```

**Common blockers:**

| Operation | Cost | Alternative |
|---|---|---|
| `JSON.parse` on large payloads | O(n), CPU | Stream parse, or a worker thread |
| `fs.readFileSync` | Blocks | `fs.promises.readFile` |
| Synchronous `crypto` (`pbkdf2Sync`) | Very slow | Async variant → thread pool |
| Regex with catastrophic backtracking | Can hang forever | RE2, or validate the pattern |
| `Array.sort` on huge arrays | O(n log n) | Chunk it, or a worker |
| Deep object cloning | O(n) | `structuredClone`, or avoid |
| Template rendering of huge datasets | CPU | Stream, or paginate |

**Chunking CPU work:**

```js
async function processLargeArray(items, fn) {
  const CHUNK = 100;
  const results = [];
  for (let i = 0; i < items.length; i += CHUNK) {
    results.push(...items.slice(i, i + CHUNK).map(fn));
    await new Promise(resolve => setImmediate(resolve));   // ⭐ yield the loop
  }
  return results;
}
```

**Detecting a blocked loop:**

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  const p99 = h.percentile(99) / 1e6;      // ns → ms
  if (p99 > 100) logger.warn({ p99 }, 'event loop lag');
  h.reset();
}, 10_000);
```

🏭 **Event loop lag is the single most important Node metric.** p99 above ~50ms means users are experiencing queuing delay that has nothing to do with your database.

---

## 4. Streams

Streams process data incrementally, in bounded memory. For a Node backend, this is the difference between handling a 2 GB file and crashing.

```
   ┌──────────┐   ┌───────────┐   ┌──────────┐
   │ Readable │──▶│ Transform │──▶│ Writable │
   └──────────┘   └───────────┘   └──────────┘
       source        modify          sink

   Types: Readable · Writable · Duplex · Transform
```

```js
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';
import { Transform } from 'node:stream';

// ❌ Loads the entire file into memory — OOM on large files
const data = await fs.readFile('10gb.csv');

// ✅ Constant memory regardless of file size
await pipeline(
  createReadStream('10gb.csv'),
  parseCsv(),
  new Transform({
    objectMode: true,
    transform(row, _enc, cb) {
      cb(null, JSON.stringify(transform(row)) + '\n');
    },
  }),
  createGzip(),
  createWriteStream('out.jsonl.gz'),
);
```

⚠️ **Always use `pipeline`, never `.pipe()` chains.** `pipe` does not propagate errors or clean up on failure, leaking file descriptors. `pipeline` handles error propagation and destroys every stream in the chain.

### Backpressure

```
   Fast producer ──▶ [ buffer ] ──▶ Slow consumer

   write() returns FALSE when the buffer exceeds highWaterMark
   → the producer MUST pause until 'drain'

   pipeline() handles this automatically. Manual writing does not.
```

```js
// Manual backpressure handling
async function writeAll(stream, items) {
  for (const item of items) {
    if (!stream.write(item)) {
      await once(stream, 'drain');        // ⭐ wait, don't buffer unboundedly
    }
  }
  stream.end();
}
```

```js
// Async iteration — the most readable stream API
for await (const chunk of readable) {
  await process(chunk);        // backpressure is automatic
}
```

### Streaming HTTP responses

```js
app.get('/export', async (req, res) => {
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename=export.csv');
  await pipeline(
    db.queryStream('SELECT * FROM large_table'),   // cursor, not full result
    toCsv(),
    res,
  );
});
```

---

## 5. Buffers

```js
Buffer.alloc(10);                    // ✅ zero-filled
Buffer.allocUnsafe(10);              // ⚠️ FASTER but contains old memory —
                                     //    never send it without fully overwriting
Buffer.from('hello', 'utf8');
Buffer.from([0x68, 0x69]);

buf.toString('hex');
buf.toString('base64url');
Buffer.concat([a, b]);
buf.readUInt32BE(0);

// ⭐ Timing-safe comparison for secrets — prevents timing attacks
import { timingSafeEqual } from 'node:crypto';
const equal = a.length === b.length && timingSafeEqual(a, b);
```

⚠️ Buffers live **outside the V8 heap**, so a buffer leak won't show in heap snapshots. Watch RSS versus heap size — a growing gap points at buffers or native memory.

---

## 6. Modules

| | ESM (`import`) | CJS (`require`) |
|---|---|---|
| Loading | Static, hoisted | Dynamic, runtime |
| Top-level `await` | ✅ | ❌ |
| Tree-shakeable | ✅ | ❌ |
| `__dirname` | ❌ (use `import.meta.dirname`) | ✅ |
| Can import the other | ✅ CJS via default import | ⚠️ only `await import()` |

```json
{
  "type": "module",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
    "./utils": "./dist/utils.js"
  },
  "engines": { "node": ">=20" }
}
```

```js
// ESM equivalents of CJS globals
import.meta.url;
import.meta.dirname;        // Node 20.11+
import.meta.filename;

// Node: prefix built-ins — unambiguous, and blocks a class of
// dependency-confusion attacks where an npm package shadows a core module
import fs from 'node:fs/promises';
import { createServer } from 'node:http';
```

---

## 7. Error Handling

```js
// ── Custom error hierarchy ────────────────────────────────
class AppError extends Error {
  constructor(message, { status = 500, code = 'internal_error', cause, ...meta } = {}) {
    super(message, { cause });                 // ⭐ preserve the original
    this.name = this.constructor.name;
    this.status = status;
    this.code = code;
    this.meta = meta;
    this.isOperational = true;                 // ⭐ expected vs programmer bug
    Error.captureStackTrace(this, this.constructor);
  }
}
class NotFoundError extends AppError {
  constructor(resource, id) {
    super(`${resource} ${id} not found`, { status: 404, code: 'not_found' });
  }
}
```

```
   OPERATIONAL ERRORS          PROGRAMMER ERRORS
   ──────────────────          ─────────────────
   Expected, handled           Bugs
   404, validation, timeout    undefined is not a function
   → return an error response  → CRASH and restart
                                  (state is now unknown)
```

```js
// ── Process-level handlers ────────────────────────────────
process.on('uncaughtException', (err, origin) => {
  logger.fatal({ err, origin }, 'uncaught exception');
  // ⭐ The process is in an UNKNOWN state. Log and exit.
  //    Do NOT attempt to continue serving.
  gracefulShutdown().finally(() => process.exit(1));
});

process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'unhandled rejection');
  process.exit(1);              // this is the default in Node 15+
});
```

⚠️ **Do not "recover" from `uncaughtException`.** After an uncaught throw, module state, open transactions, and half-written buffers are all undefined. Log, drain, exit, and let the process manager restart you. A clean restart beats a corrupted process serving wrong data.

```js
// ── Async error handling ──────────────────────────────────
// Express 4 does NOT catch async errors — wrap handlers
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await db.getUser(req.params.id);
  if (!user) throw new NotFoundError('User', req.params.id);
  res.json(user);
}));

// Express 5 and Fastify handle this natively.
```

---

## 8. Scaling

```
   CLUSTER — N processes, one per core, sharing a port

   ┌────────────────────────────────────┐
   │           PRIMARY                  │
   │  (fork workers, restart on death)  │
   └───┬────────┬────────┬────────┬─────┘
       ▼        ▼        ▼        ▼
   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
   │ W1   │ │ W2   │ │ W3   │ │ W4   │  each with its OWN
   │ :3000│ │ :3000│ │ :3000│ │ :3000│  event loop, heap, and
   └──────┘ └──────┘ └──────┘ └──────┘  connection pool

   ⚠️ No shared memory. In-process caches are PER WORKER.
      Session state must live in Redis, not memory.
```

```js
import cluster from 'node:cluster';
import { availableParallelism } from 'node:os';

if (cluster.isPrimary) {
  const n = availableParallelism();
  for (let i = 0; i < n; i++) cluster.fork();

  cluster.on('exit', (worker, code, signal) => {
    logger.warn({ pid: worker.process.pid, code, signal }, 'worker died');
    if (!worker.exitedAfterDisconnect) cluster.fork();   // restart unless intentional
  });
} else {
  startServer();
}
```

🏭 **In containers, prefer one process per container** and let Kubernetes handle replication. Cluster adds a supervision layer that duplicates what the orchestrator already does, and makes memory limits harder to reason about.

### Worker threads — for CPU

```js
// worker.js
import { parentPort, workerData } from 'node:worker_threads';
parentPort.postMessage(heavyComputation(workerData));

// main.js — a reusable pool, not one worker per task
import { Worker } from 'node:worker_threads';

class WorkerPool {
  #workers = []; #queue = []; #free = [];
  constructor(file, size = availableParallelism()) {
    for (let i = 0; i < size; i++) this.#spawn(file);
  }
  #spawn(file) {
    const w = new Worker(file);
    w.on('message', (result) => { w.resolve?.(result); this.#release(w); });
    w.on('error', (err) => { w.reject?.(err); this.#release(w); });
    this.#workers.push(w); this.#free.push(w);
  }
  #release(w) {
    const next = this.#queue.shift();
    if (next) this.#run(w, next); else this.#free.push(w);
  }
  #run(w, { data, resolve, reject }) {
    w.resolve = resolve; w.reject = reject; w.postMessage(data);
  }
  exec(data) {
    return new Promise((resolve, reject) => {
      const w = this.#free.pop();
      if (w) this.#run(w, { data, resolve, reject });
      else this.#queue.push({ data, resolve, reject });
    });
  }
}
```

| | Cluster | Worker Threads |
|---|---|---|
| Isolation | Separate processes | Same process, separate loops |
| Memory | Full copy each | Shared via `SharedArrayBuffer` |
| Startup | ~50 ms | ~10 ms |
| Use | Scale HTTP across cores | Offload CPU-bound tasks |

---

## 9. HTTP Frameworks

| | Express | Fastify | Hono | NestJS |
|---|---|---|---|---|
| Throughput | Baseline | ~2-3× | ~3-4× | ~Express (built on it) |
| Schema validation | Manual | Built-in (JSON Schema) | With Zod | class-validator |
| TypeScript | Add-on types | First class | First class | First class |
| Structure | None | Plugins | Minimal | Opinionated (DI, modules) |
| Runtime | Node | Node | Node/Bun/Deno/Workers | Node |
| Ecosystem | Enormous | Good | Growing | Good |

```js
// ── Fastify — schema-driven, fast serialization ──────────
import Fastify from 'fastify';

const app = Fastify({
  logger: { level: 'info' },
  requestIdHeader: 'x-request-id',
  bodyLimit: 1_048_576,                 // ⭐ 1 MB cap
  trustProxy: true,
});

await app.register(import('@fastify/helmet'));
await app.register(import('@fastify/cors'), { origin: allowedOrigins });
await app.register(import('@fastify/rate-limit'), { max: 100, timeWindow: '1 minute' });

app.post('/users', {
  schema: {
    body: {
      type: 'object',
      required: ['email', 'password'],
      additionalProperties: false,       // ⭐ reject unknown fields
      properties: {
        email: { type: 'string', format: 'email' },
        password: { type: 'string', minLength: 12 },
      },
    },
    response: {
      201: {                             // ⭐ response schema = 2-3× faster
        type: 'object',                  //    serialization AND field filtering
        properties: { id: { type: 'string' }, email: { type: 'string' } },
      },
    },
  },
}, async (req, reply) => {
  const user = await createUser(req.body);
  reply.code(201);
  return user;                           // password can't leak — not in the schema
});
```

Fastify's response schema does double duty: it makes serialization dramatically faster by pre-compiling a stringifier, and it strips fields not in the schema, which is the same defense-in-depth property FastAPI's `response_model` provides.

---

## 10. Database Access

```js
// ── Connection pooling — always, never a connection per request ──
import pg from 'pg';

const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                          // ⭐ per process — multiply by replicas
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 3_000,   // fail fast instead of queueing forever
  statement_timeout: 10_000,        // ⭐ kill runaway queries server-side
});

pool.on('error', (err) => logger.error({ err }, 'idle client error'));

// ── Parameterized queries ONLY ──────────────────────────
// ❌ SQL injection
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);
// ✅
await pool.query('SELECT * FROM users WHERE email = $1', [email]);

// ── Transactions ────────────────────────────────────────
async function withTransaction(fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();                 // ⭐ ALWAYS release, or the pool leaks
  }
}

// ── Streaming large result sets ─────────────────────────
import QueryStream from 'pg-query-stream';
const stream = client.query(new QueryStream('SELECT * FROM huge_table'));
for await (const row of stream) { /* constant memory */ }
```

⚠️ **Forgetting `client.release()`** exhausts the pool. Every request then hangs at `pool.connect()` until timeout. Always release in a `finally`.

```js
// ── Drizzle: typed SQL without an ORM's magic ────────────
const result = await db.select({ id: users.id, email: users.email })
  .from(users)
  .leftJoin(orders, eq(orders.userId, users.id))
  .where(and(eq(users.active, true), gt(orders.total, 100)))
  .limit(20);
```

---

## 11. Security

```js
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

app.disable('x-powered-by');
app.use(helmet());                                 // security headers
app.use(express.json({ limit: '1mb' }));           // ⭐ cap body size
app.use(rateLimit({ windowMs: 60_000, max: 100, standardHeaders: true }));

// CORS — never reflect arbitrary origins with credentials
app.use(cors({
  origin: (origin, cb) => cb(null, ALLOWED.includes(origin)),
  credentials: true,
}));
```

**The Node-specific security list:**

```
   □ npm audit / Snyk in CI, with a patch SLA
   □ package-lock.json committed; `npm ci` in CI (never `npm install`)
   □ Pin dependency versions; review transitive additions
   □ ⚠️ Prototype pollution: never merge untrusted JSON into objects
        Object.create(null) or Map for user-keyed data
   □ ⚠️ ReDoS: validate regex complexity, or use RE2
   □ ⚠️ Command injection: execFile with an array, NEVER exec with a string
   □ ⚠️ Path traversal: path.resolve + verify the result is inside the base dir
   □ ⚠️ SSRF: allowlist outbound hosts; block link-local (169.254.169.254)
   □ Secrets from env/vault, never in code or the image
   □ Run as a non-root user in the container
   □ timingSafeEqual for all secret comparisons
```

```js
// Prototype pollution — the Node-specific classic
const merge = (target, source) => {
  for (const key of Object.keys(source)) {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') continue; // ⭐
    if (typeof source[key] === 'object' && source[key] !== null) {
      target[key] ??= {};
      merge(target[key], source[key]);
    } else target[key] = source[key];
  }
  return target;
};

// Command injection
import { execFile } from 'node:child_process';
execFile('convert', [input, '-resize', '50%', output]);   // ✅ array args
// exec(`convert ${input} ...`)                            // ❌ shell injection

// Path traversal
const safe = path.resolve(BASE, userPath);
if (!safe.startsWith(BASE + path.sep)) throw new Error('Invalid path');
```

---

## 12. Performance

### Diagnostic order

```
   1. Event loop lag        → are we blocking?     ⭐ check this first
   2. CPU profile           → where does time go?
   3. Heap snapshot         → memory growth?
   4. Database query times  → usually the real cause
   5. External call latency → downstream problem?
```

```bash
# CPU profiling
node --cpu-prof --cpu-prof-dir=./profiles app.js     # → .cpuprofile, open in DevTools
node --inspect app.js                                 # attach chrome://inspect

# Continuous production profiling
clinic doctor -- node app.js        # diagnoses the category of problem
clinic flame  -- node app.js        # flamegraph
clinic bubbleprof -- node app.js    # async operation delays

# Heap
node --heapsnapshot-signal=SIGUSR2 app.js
kill -USR2 <pid>                     # dump on demand, no restart
```

### Common wins

| Fix | Impact |
|---|---|
| Remove blocking calls from the hot path | Largest |
| Connection pooling (DB, HTTP with keep-alive) | Large |
| `Promise.all` for independent I/O | Latency ÷ n |
| Response schema (Fastify) / avoid huge `JSON.stringify` | Medium |
| Compression (gzip/brotli) | Bandwidth |
| Caching layer | Depends |
| `UV_THREADPOOL_SIZE` for fs/crypto-heavy apps | Medium |
| Avoid `async_hooks` in hot paths | Medium (it has real overhead) |

```js
// ⭐ HTTP keep-alive — without it every outbound call does a full TCP+TLS handshake
import { Agent } from 'undici';
const agent = new Agent({ connections: 100, keepAliveTimeout: 10_000 });
// Node 18+: global fetch uses undici; set a global dispatcher
setGlobalDispatcher(agent);
```

### Memory leaks

```
   Common Node leaks:
   1. Event listeners never removed         → MaxListenersExceededWarning
   2. Unbounded Map/array caches            → use LRU with a size cap
   3. Closures capturing large objects
   4. Timers never cleared
   5. Global arrays accumulating requests
   6. Buffers (invisible in heap snapshots — watch RSS vs heap)
```

```js
// Diagnose: two snapshots around a repeated action, compare deltas
process.memoryUsage();
// { rss, heapTotal, heapUsed, external, arrayBuffers }
//   rss growing while heapUsed is flat → buffers or native memory
```

---

## 13. Observability

```js
// ── Structured logging with request correlation ──────────
import pino from 'pino';
import { AsyncLocalStorage } from 'node:async_hooks';

const als = new AsyncLocalStorage();
const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'req.headers.cookie', '*.password'],  // ⭐
  mixin: () => ({ requestId: als.getStore()?.requestId }),
});

app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] ?? randomUUID();
  res.setHeader('x-request-id', requestId);
  als.run({ requestId }, next);            // ⭐ propagates through async calls
});
```

`AsyncLocalStorage` is Node's answer to thread-local storage — it survives across `await` boundaries, so every log line in a request automatically carries the request ID without threading it through every function.

```js
// ── OpenTelemetry ────────────────────────────────────────
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

new NodeSDK({
  instrumentations: [getNodeAutoInstrumentations()],
}).start();      // auto-instruments http, pg, redis, and more
```

```js
// ── Metrics that matter ──────────────────────────────────
//   • event loop lag p99          ⭐ the Node-specific one
//   • http request duration by route + status
//   • active handles / requests
//   • heap used, RSS
//   • DB pool: total, idle, waiting
//   • external call latency + error rate
```

---

## 14. Testing

```js
// Node's built-in test runner — no dependencies needed
import { test, describe, before, after, mock } from 'node:test';
import assert from 'node:assert/strict';

describe('UserService', () => {
  let container, db;

  before(async () => {
    container = await new PostgreSqlContainer('postgres:16').start();
    db = createPool(container.getConnectionUri());
    await migrate(db);
  });

  after(async () => { await db.end(); await container.stop(); });

  test('creates a user without exposing the password', async () => {
    const res = await request(app).post('/users').send({
      email: 'a@b.com', password: 'Str0ngPassword!',
    });
    assert.equal(res.status, 201);
    assert.equal(res.body.password, undefined);      // ⭐ the security assertion
  });

  test('rejects duplicate email with 409', async () => { ... });
});
```

```js
// Mock the network at the boundary, not by stubbing fetch
import { MockAgent, setGlobalDispatcher } from 'undici';
const agent = new MockAgent();
setGlobalDispatcher(agent);
agent.get('https://api.stripe.com')
     .intercept({ path: '/v1/charges', method: 'POST' })
     .reply(200, { id: 'ch_1' });
```

---

## 15. Production

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
RUN apk add --no-cache dumb-init          # ⭐ proper PID 1, forwards signals
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps  --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
USER node                                  # ⭐ never root
EXPOSE 3000
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

```js
// ── Graceful shutdown — required for zero-downtime deploys ──
let shuttingDown = false;

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info({ signal }, 'shutting down');

  server.close();                          // stop accepting NEW connections
  await new Promise(r => setTimeout(r, 5_000));  // let the LB notice (preStop)

  await Promise.race([
    Promise.all([pool.end(), redis.quit(), queue.close()]),
    new Promise(r => setTimeout(r, 10_000)),     // hard cap
  ]);
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

// Readiness flips false immediately so the LB stops routing
app.get('/health/ready', (req, res) =>
  shuttingDown ? res.status(503).end() : res.json({ ok: true }));
```

⚠️ **Node does not handle signals if it isn't PID 1's direct child properly.** Without `dumb-init` or `--init`, `SIGTERM` may not reach your process, so Kubernetes waits the full grace period then `SIGKILL`s you mid-request.

```
   ⭐ Container memory: Node doesn't automatically respect cgroup limits
      for old-space sizing in every version. Set it explicitly:
      NODE_OPTIONS="--max-old-space-size=768"   for a 1 GB container
      (leave headroom for buffers, native memory, and the stack)
```

---

## 16. Interview Section

<details>
<summary><b>Q1. Explain Node's event loop and its phases.</b></summary>

Node runs your JavaScript on a single thread, with libuv providing an event loop that cycles through phases: timers, pending callbacks, idle/prepare, poll, check, and close callbacks.

Timers run `setTimeout` and `setInterval` callbacks. Poll is where most work happens — it retrieves I/O events and can block waiting for them. Check runs `setImmediate`.

The critical detail is that between every phase, and between each individual callback, Node drains two queues: `process.nextTick` first, then promise microtasks. That's why `nextTick` can starve the loop entirely if it recurses — it never yields to let the loop advance.

The practical consequence of all this is that `setTimeout(0)` versus `setImmediate` from the main module is nondeterministic, since it depends on how long startup took relative to the timer threshold. Inside an I/O callback, `setImmediate` always wins because you're already past poll.
</details>

<details>
<summary><b>Q2. How does Node handle concurrency with one thread?</b></summary>

By never waiting. When you make a network request, Node registers interest with the OS through epoll or kqueue and immediately moves on to other work. When the data arrives, the OS notifies the loop and the callback runs.

So a thousand concurrent requests aren't a thousand threads — they're a thousand registered callbacks with almost no memory cost each. That's why Node handles high-concurrency I/O well with a fraction of the memory of thread-per-request models.

The important nuance is what's *not* truly async. Network I/O goes to the kernel, but file system operations, DNS lookups via `getaddrinfo`, some crypto, and zlib all run on libuv's thread pool, which defaults to just four threads. So heavy file or crypto work queues internally even though it looks async — `UV_THREADPOOL_SIZE` is a real production tuning knob.

And the fundamental limit: none of this helps with CPU work. A 500ms computation blocks every concurrent request for 500ms.
</details>

<details>
<summary><b>Q3. What happens if you block the event loop?</b></summary>

Everything stops. Not just the current request — every concurrent connection, every pending timer, every queued callback. If a handler spends 500 milliseconds on a synchronous computation and you have a hundred concurrent requests, the last one waits fifty seconds.

Common causes: `JSON.parse` on a large payload, synchronous file reads, catastrophic regex backtracking, sorting a huge array, synchronous crypto, or deep object cloning.

The fixes depend on the type. For CPU-bound work, offload to a worker thread pool. For long loops, chunk the work and `await setImmediate` between chunks to yield. For file and crypto, use the async variants so they go to the thread pool.

Detection matters as much as prevention. I'd monitor event loop lag with `monitorEventLoopDelay` and alert when p99 exceeds around 50 milliseconds. That single metric distinguishes "our database is slow" from "we're blocking," which are completely different problems with completely different fixes.
</details>

<details>
<summary><b>Q4. Explain streams and backpressure.</b></summary>

Streams process data in chunks rather than loading it all into memory, so you can handle a 10 GB file in a few megabytes of RAM. There are four types: readable, writable, duplex, and transform.

Backpressure is the flow control mechanism. When you write to a stream faster than it can drain, the internal buffer grows past its `highWaterMark` and `write()` returns false. That's the signal to stop and wait for the `drain` event. Ignoring it means unbounded buffering, which is how you turn a slow-consumer problem into an out-of-memory crash.

The practical rule is to always use `pipeline` rather than chaining `.pipe()`. `pipe` doesn't propagate errors and doesn't clean up on failure, so a mid-stream error leaks file descriptors and leaves streams open. `pipeline` handles error propagation and destroys every stream in the chain.

`for await...of` over a readable is the most readable form and handles backpressure automatically.
</details>

<details>
<summary><b>Q5. Cluster vs worker threads.</b></summary>

Cluster forks multiple processes that share a listening port, so you use all CPU cores for HTTP handling. Each has its own event loop, heap, and connection pool, with no shared memory — so in-process caches are per worker and session state must live somewhere external like Redis.

Worker threads run inside one process with separate event loops, and they can share memory through `SharedArrayBuffer`. They're for offloading CPU-bound work off the main thread, not for scaling HTTP.

So: cluster to use all cores for request handling; worker threads to keep a heavy computation from blocking the loop.

In containerized deployments I'd usually skip cluster entirely and run one process per container, letting Kubernetes handle replication. Cluster adds a supervision layer duplicating what the orchestrator already does, and it makes memory limits harder to reason about since one container now holds N heaps.
</details>

<details>
<summary><b>Q6. How do you handle errors in Node?</b></summary>

The key distinction is operational errors versus programmer errors. Operational errors are expected — a 404, a validation failure, a timeout — and you handle them and return a proper response. Programmer errors are bugs, and the right response is to crash.

That sounds drastic, but after an uncaught exception the process state is unknown: transactions may be half-open, module state may be inconsistent, buffers may be partially written. Continuing to serve traffic from a corrupted process produces wrong answers silently, which is worse than a restart. So `uncaughtException` should log, attempt a graceful drain, and exit — never attempt to recover.

For async errors: `unhandledRejection` terminates the process by default since Node 15, which is correct. In Express 4, async handler errors aren't caught by the framework, so you need a wrapper that catches and forwards to `next`. Express 5 and Fastify handle it natively.

I'd also always preserve the original error when wrapping, using the `cause` option, so the stack trace chain survives.
</details>

<details>
<summary><b>Q7. How do you find a memory leak in Node?</b></summary>

Start with `process.memoryUsage()` over time. The relationship between RSS and heapUsed tells you where to look — if heap is growing, it's JavaScript objects; if RSS grows while heap stays flat, it's buffers or native memory, which won't appear in heap snapshots at all.

For heap growth, take two snapshots around a repeated action, force GC between them, and use the comparison view sorted by delta. Objects that grow every cycle are the leak, and the retainer chain shows what's holding them.

The common Node causes: event listeners never removed, which shows up as a `MaxListenersExceededWarning`; unbounded Maps or arrays used as caches; timers never cleared; closures capturing large objects; and request data accumulating in a module-level array.

In production I'd use `--heapsnapshot-signal=SIGUSR2` so you can dump a snapshot from a live process without restarting it, and `clinic doctor` to categorize the problem before digging in.
</details>

<details>
<summary><b>Q8. What are the Node-specific security concerns?</b></summary>

Prototype pollution is the signature one. If you merge untrusted JSON into an object without filtering keys, an attacker sends `__proto__` and pollutes `Object.prototype`, so every object in the process appears to have their property. That can become privilege escalation or RCE depending on what reads it. The defense is filtering `__proto__`, `constructor`, and `prototype` in any merge, and using `Object.create(null)` or `Map` for user-keyed data.

ReDoS matters more in Node than most runtimes because one blocked event loop is a full denial of service, not one slow request. A regex with catastrophic backtracking on attacker-controlled input hangs the entire server.

Then the dependency surface. npm's transitive dependency trees are enormous, so `npm ci` with a committed lockfile, audit in CI, and review of new transitive additions are baseline.

Beyond that it's the usual list applied carefully: `execFile` with array arguments rather than `exec` with a string, path resolution verified against a base directory, SSRF allowlists that block link-local addresses, and `timingSafeEqual` for secret comparison.
</details>

<details>
<summary><b>Q9. How do you deploy Node for zero-downtime?</b></summary>

Graceful shutdown is the core of it. On SIGTERM, immediately flip readiness to false so the load balancer stops routing new requests, stop accepting new connections with `server.close()`, wait a few seconds for the load balancer to actually notice, then drain in-flight requests and close database and Redis connections with a hard timeout.

The container detail people miss: Node needs to actually receive SIGTERM. Without `dumb-init` or Docker's `--init`, signals may not reach your process, so Kubernetes waits the full grace period and then SIGKILLs you mid-request. And the termination grace period must exceed your longest request plus the drain delay.

For memory, set `--max-old-space-size` explicitly rather than relying on the default, since Node's automatic sizing against cgroup limits isn't reliable across versions. Leave headroom below the container limit for buffers and native memory, otherwise the OOM killer takes you instead of V8 running GC.

And run as a non-root user with a multi-stage build so dev dependencies never reach the runtime image.
</details>

<details>
<summary><b>Q10. Express vs Fastify vs NestJS?</b></summary>

Express is the default by inertia — enormous ecosystem, everyone knows it, minimal structure. The downsides are that it's slower, it has no built-in validation or serialization, and Express 4 doesn't catch async errors, which is a footgun people hit constantly.

Fastify is roughly two to three times faster, and the reason is interesting: it compiles JSON Schema into a specialized serializer at startup. That schema does double duty — faster serialization plus automatic filtering of fields not declared in the response schema, so sensitive fields can't leak. It also has first-class TypeScript support and a good plugin/encapsulation model.

NestJS is opinionated architecture on top of Express or Fastify — dependency injection, modules, decorators, very Spring-like. That's valuable for large teams who want enforced structure, and costly if you want something small, because the abstraction layers are substantial.

For a new service I'd default to Fastify: the performance is free, the schema-first approach improves both speed and security, and it doesn't impose architecture. NestJS if the team is large and wants convention enforced.
</details>

---

## 17. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                         NODE.JS — ONE PAGE                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ ONE JS THREAD. Fast because it never WAITS, not because it's parallel║
║   network I/O → kernel (true async)                                  ║
║   fs · DNS · crypto · zlib → libuv THREAD POOL (default 4!)          ║
║   → UV_THREADPOOL_SIZE=16 for fs/crypto-heavy apps                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ LOOP: timers → pending → poll → check(setImmediate) → close          ║
║   between EVERY callback: nextTick queue, THEN microtasks            ║
║   ⚠️ recursive nextTick STARVES the loop; setImmediate yields         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ #1 BUG: blocking the loop → ALL concurrent requests freeze         ║
║   JSON.parse(huge) · readFileSync · ReDoS · big sort · sync crypto   ║
║   fix: worker threads (CPU) · await setImmediate to chunk            ║
║   ⭐ MONITOR EVENT LOOP LAG p99 — the key Node metric                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ STREAMS: always pipeline() NEVER .pipe() chains (errors + cleanup)   ║
║   backpressure: write() returns false → wait for 'drain'             ║
║   for await...of handles it automatically                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ ERRORS: operational(handle) vs programmer(CRASH + restart)           ║
║   uncaughtException → log, drain, exit. NEVER "recover"              ║
║   Express 4 doesn't catch async errors → wrap handlers               ║
╠══════════════════════════════════════════════════════════════════════╣
║ SCALE: cluster = processes for HTTP across cores (no shared memory)  ║
║   worker_threads = CPU offload (use a POOL, not one per task)        ║
║   in containers: 1 process/container, let K8s replicate              ║
╠══════════════════════════════════════════════════════════════════════╣
║ DB: pool always · ALWAYS client.release() in finally · $1 params     ║
║   statement_timeout server-side · stream large result sets           ║
╠══════════════════════════════════════════════════════════════════════╣
║ SECURITY: prototype pollution (filter __proto__/constructor) ·       ║
║   ReDoS = full DoS here · execFile array not exec string ·           ║
║   npm ci + lockfile + audit · timingSafeEqual · non-root             ║
╠══════════════════════════════════════════════════════════════════════╣
║ PROD: dumb-init for signals · graceful shutdown (readiness false     ║
║   FIRST) · --max-old-space-size explicitly · AsyncLocalStorage for   ║
║   request correlation · undici keep-alive agent for outbound calls   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [JavaScript](../01-languages/javascript.md) · [TypeScript](../01-languages/typescript.md) · [API Design](api-design.md) · [Observability](../06-cloud-devops/observability-sre.md)
