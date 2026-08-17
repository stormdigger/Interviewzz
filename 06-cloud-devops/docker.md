# 🐳 Docker & Containers

> A container is not a lightweight VM. It's a normal Linux process with restricted visibility and restricted resources. Once that clicks, everything else about Docker becomes obvious.

**Prerequisite:** [Linux & Networking](linux-networking.md)

---

## 📑 Contents

1. [What a Container Actually Is](#1-what-a-container-actually-is)
2. [Images and Layers](#2-images-and-layers)
3. [Writing a Good Dockerfile](#3-writing-a-good-dockerfile)
4. [Multi-Stage Builds](#4-multi-stage-builds)
5. [Networking](#5-networking)
6. [Storage](#6-storage)
7. [Resource Limits](#7-resource-limits)
8. [Security](#8-security)
9. [Docker Compose](#9-docker-compose)
10. [Production Practices](#10-production-practices)
11. [Debugging](#11-debugging)
12. [Interview Section](#12-interview-section)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. What a Container Actually Is

#### 💬 The core realization

```
   ⭐ A CONTAINER IS JUST A LINUX PROCESS.

   Run `ps aux` on the HOST and you can see the container's
   processes listed alongside everything else. There's no
   virtualization, no guest kernel, no hypervisor.

   What makes it feel isolated is three kernel features:

   ┌──────────────────────────────────────────────────────────────┐
   │ ① NAMESPACES  — control what a process can SEE               │
   │ ② CGROUPS     — control what a process can USE               │
   │ ③ UNION FS    — layered, copy-on-write filesystem            │
   └──────────────────────────────────────────────────────────────┘
```

### VM vs Container

```
   VIRTUAL MACHINE                    CONTAINER
   ┌──────────┬──────────┐            ┌──────────┬──────────┐
   │  App A   │  App B   │            │  App A   │  App B   │
   ├──────────┼──────────┤            ├──────────┼──────────┤
   │  Bins    │  Bins    │            │  Bins    │  Bins    │
   ├──────────┼──────────┤            └──────────┴──────────┘
   │ GUEST OS │ GUEST OS │  ⚠️ each   ┌─────────────────────┐
   │ (kernel) │ (kernel) │    ~GBs    │  Container runtime  │
   ├──────────┴──────────┤            ├─────────────────────┤
   │     HYPERVISOR      │            │   ⭐ SHARED KERNEL   │
   ├─────────────────────┤            ├─────────────────────┤
   │      HOST OS        │            │      HOST OS        │
   ├─────────────────────┤            ├─────────────────────┤
   │      HARDWARE       │            │      HARDWARE       │
   └─────────────────────┘            └─────────────────────┘

   Boot: ~30-60 seconds                Boot: ~50-500 ms  ⭐
   Size: GBs                           Size: MBs
   Isolation: STRONG (hardware)        Isolation: weaker (kernel)
   Can run a different OS kernel       ⚠️ MUST share the host kernel
```

```
   ⭐ THE SECURITY IMPLICATION
     A container escape is a KERNEL exploit away from the host.
     A VM escape requires a HYPERVISOR exploit, which is a much
     smaller and more hardened surface.

     → For untrusted/multi-tenant workloads, use VMs, or
       hardened runtimes like gVisor (user-space kernel) or
       Kata Containers (lightweight VMs with a container API).
```

### Namespaces — what you can see

```
   ┌──────────────────────────────────────────────────────────────┐
   │ pid      own process tree; ⭐ your app sees itself as PID 1   │
   │ net      own network stack: interfaces, routes, ports        │
   │ mnt      own filesystem mounts                               │
   │ uts      own hostname and domain name                        │
   │ ipc      own shared memory and semaphores                    │
   │ user     ⭐ UID/GID mapping — root inside ≠ root outside      │
   │ cgroup   own view of the cgroup hierarchy                    │
   │ time     own boot time / clock offset (newer kernels)        │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THESE ARE COMPOSABLE. `--network host` means "share the
     host's net namespace but keep the others." Kubernetes pods
     share the net and ipc namespaces between containers, which
     is exactly why containers in a pod reach each other on
     localhost.
```

### Cgroups — what you can use

```
   Limit and account for: CPU · memory · block I/O · network ·
                          PIDs · devices

   ⚠️ EXCEEDING THE MEMORY LIMIT → the kernel OOM-kills your
     process, even when the HOST has plenty of free memory.
     Exit code 137 (128 + SIGKILL) is the signature.

   ⭐ CPU limits are enforced by THROTTLING, not killing.
     Your process is descheduled for part of each period, which
     shows up as mysterious latency spikes rather than errors.
```

---

## 2. Images and Layers

```
   AN IMAGE IS A STACK OF READ-ONLY LAYERS.
   A CONTAINER ADDS ONE THIN WRITABLE LAYER ON TOP.

   ┌──────────────────────────────────────┐
   │  Container writable layer  (thin)    │  ← all changes go here
   ├──────────────────────────────────────┤  ═══ IMAGE (read-only) ═══
   │  L4: COPY . .                        │
   ├──────────────────────────────────────┤
   │  L3: RUN npm ci                      │
   ├──────────────────────────────────────┤
   │  L2: COPY package.json .             │
   ├──────────────────────────────────────┤
   │  L1: FROM node:22-alpine             │
   └──────────────────────────────────────┘

   ⭐ Layers are shared across images and containers. Ten
     containers from the same image consume ONE copy of the
     image on disk.
```

### Copy-on-write

```
   Reading a file: found by searching DOWN through the layers.
   Writing a file: the whole file is COPIED UP into the
                   writable layer first, then modified.

   ⚠️ CONSEQUENCES
     • The first write to a large file is slow (full file copy)
     • ⭐ Writing lots of data into the container filesystem is
       a mistake — use a VOLUME instead
     • Deleting a file in a later layer does NOT shrink the image;
       it only adds a "whiteout" marker. The bytes are still there.
```

```
   ⚠️⚠️ THE SECRET-IN-A-LAYER TRAP — the most common serious
     Dockerfile mistake

   RUN echo "$AWS_SECRET" > /tmp/creds && \
       do-something && \
       rm /tmp/creds          ← ⚠️ THE SECRET IS STILL IN THE LAYER

   The rm creates a whiteout in a LATER layer. Anyone who pulls
   the image can extract the earlier layer and read the secret.

   ⭐ FIXES
     • BuildKit secret mounts: RUN --mount=type=secret,id=aws ...
       → never written to any layer
     • Multi-stage builds: use the secret in a build stage that
       is discarded and never copied forward
     • Runtime injection: environment variables or mounted files,
       never baked into the image
```

### Layer caching — the rule that determines build speed

```
   ⭐ CACHE INVALIDATION CASCADES. Once one layer's cache is
     busted, EVERY subsequent layer rebuilds.

   ❌ SLOW — every source change reinstalls all dependencies
   COPY . .                     ← any file change busts this...
   RUN npm ci                   ← ...so this reruns every time

   ✅ FAST — dependencies are cached until the manifest changes
   COPY package.json package-lock.json ./
   RUN npm ci                   ← cached unless deps actually change
   COPY . .                     ← only this rebuilds on code change

   ⭐ THE PRINCIPLE: order instructions from LEAST to MOST
     frequently changing.
       base image → system packages → dependency manifests →
       dependency install → application source
```

---

## 3. Writing a Good Dockerfile

```dockerfile
# ⭐ Pin the digest, not just the tag — tags are mutable and
#    "node:22-alpine" can silently change under you
FROM node:22-alpine@sha256:abc123...

# Install system deps early (changes rarely) and clean up in the
# SAME RUN, or the cleanup lands in a later layer and saves nothing
RUN apk add --no-cache dumb-init curl

WORKDIR /app

# ⭐ Dependencies before source — the key caching decision
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --chown=node:node . .

# ⭐ Non-root user — containers run as root by default
USER node

ENV NODE_ENV=production

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health/live || exit 1

# ⭐ dumb-init as PID 1: forwards signals, reaps zombies
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server.js"]
```

### `.dockerignore` — do not skip this

```
   ⭐ Without it, `COPY . .` sends your entire working directory
     to the daemon: node_modules, .git history, local .env files,
     build artifacts. That's slow, bloats the image, and can
     leak secrets.

   node_modules
   .git
   .env
   .env.*
   *.log
   dist
   coverage
   .vscode
   README.md
   Dockerfile
   docker-compose.yml
```

### ENTRYPOINT vs CMD

```
   ENTRYPOINT   the executable — hard to override
   CMD          the default arguments — easily overridden

   ENTRYPOINT ["node"]
   CMD ["server.js"]

   docker run img                → node server.js
   docker run img worker.js      → node worker.js       ⭐ CMD replaced
   docker run --entrypoint sh img → sh                  (ENTRYPOINT overridden)
```

```
   ⚠️ SHELL FORM vs EXEC FORM — this one causes real outages

   CMD node server.js             ← SHELL form
     → runs as: /bin/sh -c "node server.js"
     → ⚠️ SHELL is PID 1, not node
     → ⚠️ SIGTERM goes to the shell, which does NOT forward it
     → your app never shuts down gracefully

   CMD ["node", "server.js"]      ← ⭐ EXEC form (JSON array)
     → node is PID 1 and receives signals directly

   ⭐ ALWAYS USE EXEC FORM for CMD and ENTRYPOINT.
```

---

## 4. Multi-Stage Builds

#### 💬 The single biggest image-size win

```
   ⚠️ A NAIVE IMAGE SHIPS YOUR ENTIRE BUILD ENVIRONMENT:
     compilers, dev dependencies, test fixtures, source code,
     build caches. That's both bloat and attack surface.

   ⭐ MULTI-STAGE: build in a fat image, copy only the ARTIFACT
     into a minimal runtime image. Everything else is discarded.
```

```dockerfile
# ─── STAGE 1: build ────────────────────────────────────────
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/server

# ─── STAGE 2: runtime ──────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]

# ⭐ RESULT: ~900 MB build image → ~10 MB runtime image
#   No shell, no package manager, no compiler = tiny attack surface
```

### Base image comparison

```
   ┌────────────────┬─────────┬──────────────────────────────────┐
   │ Base           │ Size    │ Notes                            │
   ├────────────────┼─────────┼──────────────────────────────────┤
   │ ubuntu/debian  │ ~75 MB  │ Familiar, full toolchain, more   │
   │                │         │ CVEs to patch                    │
   │ alpine         │ ~7 MB   │ ⚠️ musl libc, not glibc — subtle  │
   │                │         │ breakage with some binaries;     │
   │                │         │ known DNS resolver quirks        │
   │ distroless ⭐   │ ~2-20MB │ No shell, no package manager.    │
   │                │         │ Best security. Harder to debug   │
   │                │         │ (use the :debug variant)         │
   │ scratch        │ 0 MB    │ Truly empty. Static binaries only│
   │ chainguard     │ ~5-20MB │ Distroless + continuous CVE      │
   │                │         │ patching, SBOMs included         │
   └────────────────┴─────────┴──────────────────────────────────┘
```

```
   ⚠️ THE ALPINE TRAP FOR PYTHON
     Alpine uses musl instead of glibc, so Python wheels compiled
     for manylinux don't work — pip falls back to compiling from
     source. Builds become dramatically slower and images can
     end up LARGER than the Debian equivalent.

     ⭐ For Python, prefer python:3.12-slim over alpine.
```

### BuildKit features worth using

```dockerfile
# syntax=docker/dockerfile:1

# ⭐ Cache mount — the package cache persists between builds
#    without ever entering a layer
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# ⭐ Secret mount — available during the RUN, never in any layer
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci

# Bind mount — read files without COPYing them into a layer
RUN --mount=type=bind,source=.,target=/src \
    go build -o /app /src/cmd/server
```

```bash
# Multi-architecture builds (Apple Silicon → x86 servers)
docker buildx build --platform linux/amd64,linux/arm64 -t img:tag --push .
```

---

## 5. Networking

```
   ┌──────────────────────────────────────────────────────────────┐
   │ bridge (default)   Private network with NAT. Containers      │
   │                    reach each other by IP; ⭐ on a USER-      │
   │                    DEFINED bridge they also resolve each     │
   │                    other by NAME via embedded DNS.           │
   ├──────────────────────────────────────────────────────────────┤
   │ host               Shares the host's network namespace.      │
   │                    No NAT overhead, no port mapping.         │
   │                    ⚠️ No port isolation; Linux only.          │
   ├──────────────────────────────────────────────────────────────┤
   │ none               No networking at all.                     │
   ├──────────────────────────────────────────────────────────────┤
   │ overlay            Multi-host networking (Swarm/K8s CNI).    │
   ├──────────────────────────────────────────────────────────────┤
   │ macvlan            Container gets its own MAC on the physical│
   │                    network — appears as a real device.       │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️ THE DEFAULT BRIDGE HAS NO DNS.
     Containers on the default bridge can only reach each other
     by IP. Always create a user-defined network:

   $ docker network create app-net
   $ docker run --network app-net --name db postgres
   $ docker run --network app-net --name api myapi
     → the api container can now connect to host "db"  ⭐
```

```
   PORT PUBLISHING

   -p 8080:3000        host 8080 → container 3000
   -p 127.0.0.1:8080:3000   ⭐ bind to localhost only — otherwise
                              you may expose a service to the
                              entire internet without realizing
   -P                  publish all EXPOSEd ports to random host ports

   ⚠️ Docker's port publishing writes iptables rules that can
     BYPASS a UFW firewall. A container published with -p is
     reachable even if UFW says the port is blocked.
```

```
   ⭐ REACHING THE HOST FROM A CONTAINER
     host.docker.internal          (Docker Desktop, and Linux
                                    with --add-host)
     ⚠️ localhost inside a container means the CONTAINER, not the
       host — one of the most common beginner mistakes.
```

---

## 6. Storage

```
   ┌──────────────────────────────────────────────────────────────┐
   │ VOLUMES  ⭐ the default choice for persistent data            │
   │   Managed by Docker in /var/lib/docker/volumes               │
   │   ✅ Best performance · portable · backup-able · driver       │
   │      plugins for cloud storage                               │
   │   docker volume create pgdata                                │
   │   -v pgdata:/var/lib/postgresql/data                         │
   ├──────────────────────────────────────────────────────────────┤
   │ BIND MOUNTS  — a host path mapped into the container         │
   │   ⭐ Great for DEVELOPMENT (live source reload)               │
   │   ⚠️ Host-path dependent, permission mismatches on Linux,     │
   │     ⚠️ slow on macOS/Windows (filesystem translation)         │
   │   -v $(pwd):/app                                             │
   ├──────────────────────────────────────────────────────────────┤
   │ tmpfs  — in memory only, never touches disk                  │
   │   ⭐ For secrets and scratch data you don't want persisted    │
   │   --tmpfs /tmp                                               │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️ NEVER WRITE APPLICATION DATA TO THE CONTAINER FILESYSTEM.
     • It's lost when the container is removed
     • Copy-on-write makes writes slow
     • The writable layer grows unbounded and fills the host disk

     ⭐ Anything that must survive → a volume. Logs → stdout.
```

---

## 7. Resource Limits

```bash
docker run \
  --memory="512m"           `# hard limit — OOM kill if exceeded` \
  --memory-reservation="256m" `# soft limit, reclaimed under pressure` \
  --cpus="1.5"              `# 1.5 cores worth of time` \
  --pids-limit=100          `# ⭐ prevents fork bombs` \
  --restart=unless-stopped \
  myapp
```

```
   ⚠️⚠️ RUNTIMES DON'T ALWAYS SEE CGROUP LIMITS

   A container limited to 512 MB on a 64 GB host: many runtimes
   historically read the HOST's memory and sized their heap
   accordingly — then got OOM-killed immediately.

   ⭐ FIXES
     Java 10+   -XX:+UseContainerSupport is on by default;
                use -XX:MaxRAMPercentage=75 instead of -Xmx
                → the same image works at any memory limit
     Node       NODE_OPTIONS="--max-old-space-size=384"
                (leave headroom below the cgroup limit for
                 buffers and native memory)
     Go         GOMEMLIMIT and automaxprocs
     Python     usually fine, but worker counts computed from
                os.cpu_count() will see HOST cores — use
                len(os.sched_getaffinity(0)) instead
```

```
   ⭐ CPU LIMITS THROTTLE, THEY DON'T KILL

   With --cpus=1, your process is descheduled once it uses its
   quota within each 100ms period. The symptom is latency spikes
   with no errors and no obvious cause — particularly brutal for
   garbage-collected runtimes, where a GC pause can consume the
   entire quota and stall the app for the rest of the period.

   → Watch cgroup throttling metrics
     (container_cpu_cfs_throttled_seconds_total), not just CPU usage.
```

---

## 8. Security

```
   ⭐ THE CONTAINER SECURITY CHECKLIST

   IMAGE
   □ Minimal base (distroless/alpine/slim), pinned by DIGEST
   □ Multi-stage — no compilers or build tools in the runtime
   □ Scan for CVEs in CI (Trivy, Grype, Snyk) with a patch SLA
   □ ⭐ No secrets in layers — BuildKit secret mounts or runtime
     injection only
   □ .dockerignore excludes .git, .env, node_modules
   □ Sign images (cosign) and verify at deploy time

   RUNTIME
   □ ⭐ USER non-root (containers run as root by DEFAULT)
   □ --read-only filesystem + tmpfs for writable paths
   □ --cap-drop=ALL, then add back only what's needed
   □ --security-opt=no-new-privileges
   □ Resource limits set (memory, CPU, PIDs)
   □ ⚠️ NEVER --privileged
   □ ⚠️ NEVER mount /var/run/docker.sock into a container

   HOST
   □ Rootless Docker, or user namespace remapping
   □ Keep the kernel patched (container isolation IS the kernel)
   □ seccomp and AppArmor/SELinux profiles enabled
```

```
   ⚠️⚠️ MOUNTING THE DOCKER SOCKET IS ROOT ON THE HOST

   -v /var/run/docker.sock:/var/run/docker.sock

   Anyone inside that container can start a new container with
   --privileged and the host root filesystem mounted. That is a
   complete host compromise, not a partial one.

   It's extremely common in CI setups and "container management
   UI" tools. ⭐ Use a socket proxy with a restricted API surface,
   or rootless/daemonless builders (Kaniko, Buildah) instead.
```

```bash
# ⭐ A hardened run
docker run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --user 1000:1000 \
  --memory=512m --cpus=1 --pids-limit=100 \
  myapp
```

---

## 9. Docker Compose

```yaml
services:
  api:
    build:
      context: .
      target: development           # ⭐ build a specific stage
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    volumes:
      - .:/app                      # live reload in dev
      - /app/node_modules           # ⭐ anonymous volume so the host's
                                    #   node_modules doesn't shadow the
                                    #   container's
    depends_on:
      db:
        condition: service_healthy  # ⭐ waits for the HEALTHCHECK,
                                    #   not just for the container to start
    develop:
      watch:                        # Compose Watch — sync/rebuild on change
        - action: sync
          path: ./src
          target: /app/src

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

```
   ⚠️ `depends_on` WITHOUT `condition: service_healthy` only
     waits for the container to START, not to be READY.
     Postgres accepts a TCP connection seconds before it can
     serve queries — which is why apps crash-loop on first
     `docker compose up`.
```

---

## 10. Production Practices

```
   ⭐ THE TWELVE-FACTOR ESSENTIALS FOR CONTAINERS

   ① CONFIG VIA ENVIRONMENT
      Never bake environment-specific config into the image.
      ⭐ The SAME image must run in dev, staging, and prod —
        that's what makes a build artifact trustworthy.

   ② LOGS TO STDOUT/STDERR
      Never write log files inside a container. The platform
      collects stdout. Log files fill the writable layer and
      disappear on restart.

   ③ ONE CONCERN PER CONTAINER
      Not strictly "one process" — a process may fork workers.
      But don't run nginx + your app + cron in one container;
      you can't scale or restart them independently.

   ④ STATELESS AND DISPOSABLE
      Any container can be killed at any moment. State goes to
      volumes or external services.

   ⑤ FAST STARTUP, GRACEFUL SHUTDOWN
      ⭐ Handle SIGTERM: flip readiness false, drain, exit.
        See [Linux §5](linux-networking.md#5-signals).
```

```
   TAGGING STRATEGY

   ❌ :latest              ⚠️ mutable, unreproducible, and you can
                            never tell what's actually deployed
   ✅ :1.2.3               semantic version
   ✅ :git-a1b2c3d         ⭐ commit SHA — exact traceability
   ✅ :2026-08-14T10-30    timestamp

   ⭐ Deploy by DIGEST (sha256:...) in production. Tags can be
     re-pointed; digests cannot.
```

```
   HEALTHCHECKS — separate liveness from readiness

   LIVENESS   "is the process alive?"
              ⚠️ Must NOT check dependencies. If your database
                is down, failing liveness restarts every
                container and makes everything worse.

   READINESS  "can I serve traffic right now?"
              Checks DB, cache, critical dependencies.
              Failing = remove from the load balancer,
              but DON'T restart.

   ⭐ Conflating these is a classic outage amplifier.
```

---

## 11. Debugging

```bash
# ── Inspecting ─────────────────────────────────────────────
docker ps -a                          # all containers, incl. exited
docker logs -f --tail 100 <c>         # follow logs
docker logs --since 10m <c>
docker inspect <c>                    # full JSON config + state
docker inspect -f '{{.State.ExitCode}}' <c>
docker stats                          # ⭐ live CPU/memory/IO/net
docker top <c>                        # processes inside
docker diff <c>                       # ⭐ what changed vs the image

# ── Getting inside ─────────────────────────────────────────
docker exec -it <c> sh                # shell in a RUNNING container
docker run -it --entrypoint sh <img>  # shell instead of the app
                                      # ⭐ for debugging a crash-loop

# ⭐ DEBUGGING A DISTROLESS CONTAINER (no shell inside)
docker debug <c>                      # Docker Desktop
# or attach a debug container sharing its namespaces:
docker run -it --pid=container:<c> --network=container:<c> \
  --cap-add SYS_PTRACE nicolaka/netshoot

# ── Build problems ─────────────────────────────────────────
docker build --progress=plain --no-cache .   # ⭐ see full output
docker history <img>                  # ⭐ layer sizes — find the bloat
dive <img>                            # interactive layer explorer

# ── Cleanup ────────────────────────────────────────────────
docker system df                      # ⭐ what's consuming disk
docker system prune -a --volumes      # ⚠️ removes everything unused
docker builder prune                  # build cache only
```

```
   ⭐ EXIT CODE DECODER

   0     clean exit
   1     application error
   125   the Docker daemon itself failed
   126   the command exists but isn't executable
   127   command not found  ⭐ usually a typo or a missing binary
                              in a minimal base image
   137   ⚠️ SIGKILL (128+9) — almost always OOM-KILLED
         → check `docker inspect` for "OOMKilled": true
   139   SIGSEGV (128+11) — segmentation fault
   143   SIGTERM (128+15) — graceful shutdown, normal on stop
```

```
   ⚠️ THE "CONTAINER EXITS IMMEDIATELY" CHECKLIST
     1. The main process finished (a container lives only as long
        as PID 1). Running a shell script that ends = exit.
     2. Wrong CMD/ENTRYPOINT — exit code 127 means not found
     3. OOM killed — exit 137
     4. It's writing to a path that doesn't exist or isn't writable
     5. Check `docker logs` FIRST — the answer is usually there
```

---

## 12. Interview Section

<details>
<summary><b>Q1. What's the difference between a container and a VM?</b></summary>

A VM virtualizes hardware and runs a complete guest operating system with its own kernel, managed by a hypervisor. A container is just a Linux process on the host, with restricted visibility through namespaces and restricted resources through cgroups. There's no guest kernel and no hypervisor.

That difference produces everything else. Containers start in milliseconds rather than tens of seconds, are measured in megabytes rather than gigabytes, and you can run far more of them per host because there's no duplicated OS overhead.

The tradeoff is isolation strength. Containers share the host kernel, so a container escape is one kernel vulnerability away from host compromise. A VM escape requires breaking the hypervisor, which is a much smaller and more hardened attack surface.

That's why untrusted or multi-tenant workloads still use VMs, or hybrid runtimes like gVisor, which interposes a user-space kernel, or Kata Containers, which runs lightweight VMs behind a container API.
</details>

<details>
<summary><b>Q2. Explain image layers and why Dockerfile order matters.</b></summary>

An image is a stack of read-only layers, one per instruction, unioned into a single filesystem view. A running container adds one thin writable layer on top, and all changes go there via copy-on-write.

Order matters because cache invalidation cascades. When one layer's inputs change, that layer and every layer after it rebuild. So you order instructions from least to most frequently changing: base image, system packages, dependency manifests, dependency install, then application source.

The concrete win is copying package.json and running the install *before* copying source. Now a code change doesn't reinstall dependencies, which can turn a three-minute build into fifteen seconds.

The other consequence people miss is that deleting a file in a later layer doesn't shrink the image — it just writes a whiteout marker. The original bytes are still in the earlier layer and anyone who pulls the image can extract them. That's why the classic pattern of writing a secret, using it, and removing it in separate steps leaks the secret permanently.
</details>

<details>
<summary><b>Q3. Why use multi-stage builds?</b></summary>

Because a naive image ships your entire build environment: compilers, dev dependencies, test files, source code, and build caches. That's both size and attack surface you don't need at runtime.

Multi-stage lets you build in a full-featured image and then copy only the resulting artifact into a minimal runtime image. A Go service can go from around 900 megabytes to about 10 with a distroless base.

The security benefit is arguably larger than the size benefit. A distroless image has no shell and no package manager, so an attacker who achieves code execution has dramatically fewer tools available — no curl to pull a payload, no sh to pivot with.

It also solves the build-secret problem cleanly. A secret used in a build stage that's never copied forward never appears in the final image's layers.
</details>

<details>
<summary><b>Q4. Container exits with code 137. What happened?</b></summary>

137 is 128 plus 9, meaning SIGKILL. In practice that's almost always an OOM kill from exceeding the cgroup memory limit.

I'd confirm with `docker inspect` and check the OOMKilled flag, or in Kubernetes look at the pod's last state. `dmesg` on the host shows the kernel's OOM messages.

The important subtlety is that this happens even when the *host* has plenty of free memory — the cgroup limit is what's enforced, not host capacity.

The most common root cause is a runtime that doesn't respect cgroup limits. A JVM or Node process that reads the host's total memory and sizes its heap accordingly will immediately exceed a container limit. The fix is `-XX:MaxRAMPercentage` for Java rather than a fixed `-Xmx`, and `--max-old-space-size` for Node, always leaving headroom below the cgroup limit for native memory and buffers.

If the limit genuinely isn't enough, that's a capacity question — but I'd first check for a leak by watching RSS over time rather than just raising the limit.
</details>

<details>
<summary><b>Q5. How do you handle secrets in containers?</b></summary>

Never in the image. Not as an ARG, not as a file written and deleted, not in an ENV baked at build time — all of those persist in layers where anyone with the image can read them.

At build time, if you genuinely need a credential to fetch private dependencies, use BuildKit secret mounts. The secret is available during that RUN instruction and never enters any layer.

At runtime, inject from the orchestrator: Kubernetes secrets mounted as files, or a secrets manager like Vault or AWS Secrets Manager fetched at startup. Mounted files are preferable to environment variables, because environment variables leak through `/proc`, crash dumps, child processes, and error reporting tools.

Best of all is short-lived credentials fetched with a workload identity — IRSA on EKS, workload identity on GKE — so there's no long-lived secret to steal in the first place.

And the operational piece: scan images in CI for accidentally committed secrets, since this is a mistake that gets made repeatedly.
</details>

<details>
<summary><b>Q6. Your container works locally but fails in production. How do you debug it?</b></summary>

First, is it the same image? If the tag is `:latest`, that's likely the whole answer — tags are mutable. Deploying by digest removes this class of problem entirely.

Then the usual differences. Architecture: an image built on Apple Silicon is arm64 and won't run on x86 nodes without a multi-arch build. Resource limits: it may work locally with unlimited memory and get OOM-killed under a production cgroup limit. Environment variables and secrets that exist locally in a `.env` file but weren't configured in production. Filesystem permissions, especially if production runs read-only or as a different user. And network policy — production often restricts egress in ways local Docker doesn't.

Practically I'd start with `docker logs` or `kubectl logs --previous` for a crashed container, then check the exit code, then `describe` the pod for events like OOMKilled or ImagePullBackOff.

If the container has no shell because it's distroless, I'd attach a debug container sharing its process and network namespaces — `nicolaka/netshoot` is the standard tool — which gives full diagnostic capability without adding anything to the production image.
</details>

<details>
<summary><b>Q7. Why is mounting the Docker socket dangerous?</b></summary>

Because access to the Docker socket is equivalent to root on the host, not a subset of it.

Anyone who can talk to that socket can start a new container with `--privileged` and the host's root filesystem bind-mounted. From there they read any file, write to any file, and escape entirely. There's no meaningful boundary — the Docker API is a root-level API.

It's dangerously common in CI runners that build images, and in container management dashboards.

The alternatives: a socket proxy that exposes only a whitelisted subset of API endpoints; daemonless build tools like Kaniko or Buildah that don't need a socket at all; or in Kubernetes, using the platform's own APIs with proper RBAC rather than reaching into the container runtime.
</details>

<details>
<summary><b>Q8. What makes a container image production-ready?</b></summary>

Working backwards from what fails.

Build: multi-stage with a minimal runtime base, pinned by digest rather than tag, with a `.dockerignore` that excludes `.git`, `.env`, and `node_modules`. Dependencies installed before source is copied so caching works.

Security: runs as a non-root user, read-only root filesystem with tmpfs for anything writable, all capabilities dropped and only necessary ones added back, no secrets in layers, and CVE scanning in CI with an actual patch commitment rather than an ignored report.

Runtime correctness: exec-form CMD so signals reach the process, an init like tini as PID 1 to forward signals and reap zombies, graceful SIGTERM handling that flips readiness false before draining, and resource limits with a runtime configured to respect them.

Operations: logs to stdout rather than files, separate liveness and readiness endpoints, config entirely from environment so the same image runs everywhere, and immutable tagging by commit SHA so you can always tell exactly what's deployed.

The one I'd emphasize is signal handling, because it's invisible until you notice every deploy drops requests.
</details>

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                    DOCKER — ONE PAGE                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ A CONTAINER IS A LINUX PROCESS with restricted view + resources    ║
║   namespaces = what you SEE · cgroups = what you USE · unionfs       ║
║   shared kernel → weaker isolation than a VM                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ LAYERS: order LEAST→MOST frequently changing                         ║
║   COPY package.json → RUN install → COPY .   ⭐ the caching win       ║
║ ⚠️ deleting a file in a later layer does NOT remove it — SECRETS      ║
║   WRITTEN THEN DELETED ARE STILL EXTRACTABLE                         ║
║   → BuildKit --mount=type=secret, or a discarded build stage         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ ALWAYS EXEC FORM: CMD ["node","server.js"] not CMD node server.js  ║
║   shell form makes sh PID 1 → SIGTERM never reaches your app         ║
║ ⭐ USE an init (tini/dumb-init/--init): forwards signals, reaps zombies║
║ ⭐ USER non-root — containers run as ROOT by default                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ MULTI-STAGE: build fat, ship thin. Go 900MB → 10MB distroless        ║
║   distroless = no shell/package manager = tiny attack surface        ║
║   ⚠️ alpine + Python = musl breaks wheels → use slim instead          ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ RUNTIMES MISREAD CGROUP LIMITS → immediate OOM                     ║
║   Java: -XX:MaxRAMPercentage=75 (not -Xmx)                           ║
║   Node: --max-old-space-size below the limit                         ║
║ ⭐ CPU limits THROTTLE (latency spikes), memory limits KILL (137)     ║
╠══════════════════════════════════════════════════════════════════════╣
║ NETWORK: default bridge has NO DNS → create a user-defined network   ║
║   -p 127.0.0.1:8080:3000 to avoid exposing publicly                  ║
║   ⚠️ docker -p writes iptables rules that BYPASS UFW                  ║
║ STORAGE: volumes for data · NEVER write app data to the container FS ║
║   logs → stdout, always                                              ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️⚠️ NEVER mount /var/run/docker.sock = ROOT ON THE HOST              ║
║ ⚠️ NEVER --privileged                                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ EXIT CODES: 137=OOM(128+9) · 143=SIGTERM · 127=not found · 125=daemon║
║ DEBUG: docker logs → inspect → stats → diff → history/dive           ║
║   distroless? attach netshoot sharing --pid/--network namespaces     ║
║ TAG BY COMMIT SHA, deploy by DIGEST. Never :latest in production.    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Kubernetes →](kubernetes.md) · **Related:** [Linux & Networking](linux-networking.md) · [CI/CD](cicd.md)
