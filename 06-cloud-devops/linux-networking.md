# 🐧 Linux & Networking

> The layer beneath everything else. When a container won't start, a service is slow, or a connection hangs, the answer is almost always here.

---

## 📑 Contents

1. [Linux Mental Model](#1-linux-mental-model)
2. [Processes](#2-processes)
3. [Memory](#3-memory)
4. [Filesystems & I/O](#4-filesystems--io)
5. [Signals](#5-signals)
6. [The Networking Stack](#6-the-networking-stack)
7. [TCP Deep Dive](#7-tcp-deep-dive)
8. [DNS](#8-dns)
9. [TLS](#9-tls)
10. [HTTP Evolution](#10-http-evolution)
11. [Diagnostic Playbook](#11-diagnostic-playbook)
12. [Interview Section](#12-interview-section)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. Linux Mental Model

```
   ┌──────────────────────────────────────────────────────────────┐
   │                      USER SPACE                              │
   │   your app · shell · systemd · nginx · databases             │
   │                          │                                   │
   │                    SYSTEM CALLS                              │
   │        (read, write, open, socket, fork, mmap...)            │
   ├──────────────────────────┼───────────────────────────────────┤
   │                          ▼            KERNEL SPACE           │
   │   ┌──────────┬──────────┬──────────┬──────────┬──────────┐   │
   │   │ Process  │  Memory  │   VFS    │ Network  │  Device  │   │
   │   │scheduler │  manager │(filesys) │  stack   │ drivers  │   │
   │   └──────────┴──────────┴──────────┴──────────┴──────────┘   │
   ├──────────────────────────────────────────────────────────────┤
   │                       HARDWARE                               │
   └──────────────────────────────────────────────────────────────┘
```

#### 💬 The two ideas that explain most of Linux

**Everything is a file.** Files, directories, devices, sockets, pipes, and even process information are exposed through the filesystem interface. That's why you can `cat /proc/self/status` to inspect a process, or write to `/dev/null`, or read from a socket with the same `read()` call you use on a file.

**A system call is a boundary crossing.** Your code cannot touch hardware directly. Every I/O operation traps into the kernel, which is why syscalls are relatively expensive and why batching them matters.

```
   ⭐ /proc AND /sys — the kernel exposed as files

   /proc/cpuinfo              CPU details
   /proc/meminfo              memory statistics
   /proc/<pid>/status         process state, memory, threads
   /proc/<pid>/fd/            open file descriptors ⭐ (find leaks here)
   /proc/<pid>/limits         ulimits actually in force
   /proc/net/tcp              open TCP sockets
   /proc/loadavg              load averages
   /sys/fs/cgroup/            ⭐ container limits live here
```

---

## 2. Processes

### The lifecycle

```
   fork()          create a copy of the current process
      │            ⭐ copy-on-write — pages are shared until written,
      │              so fork is cheap
      ▼
   exec()          replace the process image with a new program
      │
      ▼
   running ──────▶ exit()
                     │
                     ▼
                  ⭐ ZOMBIE — the process is dead but its exit
                    status is retained until the parent calls wait()
                     │
                     ▼ parent calls wait()
                   reaped
```

```
   ⚠️ ZOMBIES AND ORPHANS — the classic container problem

   ZOMBIE   dead child, parent never called wait()
            → the process table entry leaks
            → thousands of zombies exhaust the PID table

   ORPHAN   parent died first → adopted by PID 1

   ⭐ IN CONTAINERS: your app is often PID 1, and PID 1 has
     special responsibilities it usually doesn't implement:
       • reaping orphaned children
       • forwarding signals to children

     → this is why `docker run --init` and `dumb-init`/`tini`
       exist. Without them, SIGTERM may never reach your app
       and Kubernetes waits the full grace period then SIGKILLs
       you mid-request.
```

### Process states

```
   R  RUNNING          on CPU or ready to run
   S  SLEEPING         waiting for an event (interruptible)
   D  UNINTERRUPTIBLE  ⭐ waiting on I/O — CANNOT be killed,
                       not even with SIGKILL. Many D-state
                       processes means a storage problem.
   T  STOPPED          suspended (SIGSTOP / debugger)
   Z  ZOMBIE           dead, awaiting reaping
```

### Understanding load average

```
   $ uptime
   load average: 2.15, 1.80, 1.42
                  ▲     ▲     ▲
                 1min  5min  15min

   ⚠️ ON LINUX, LOAD INCLUDES D-STATE (I/O WAIT), NOT JUST CPU.
     This differs from most Unixes and confuses people constantly.

   Interpreting on an 8-core machine:
     load 8   → fully utilized
     load 16  → tasks are queueing, roughly 2× oversubscribed
     load 100 with LOW CPU  ⭐ → I/O bound, not CPU bound.
       Check disk latency, not CPU.

   ⭐ Load alone tells you almost nothing. Always pair it with
     %CPU and %iowait to know WHICH resource is saturated.
```

### Key commands

```bash
ps aux                          # snapshot of all processes
ps -eLf                         # include threads
top / htop                      # live view
pidstat -p <pid> 1              # per-process CPU/IO over time

strace -p <pid>                 # ⭐ trace syscalls — see exactly
                                #   what a hung process is waiting on
strace -c -p <pid>              # syscall summary (counts + time)
ltrace -p <pid>                 # library calls

lsof -p <pid>                   # open files/sockets ⭐ FD leak hunting
lsof -i :8080                   # what is listening on this port

nice -n 10 cmd                  # lower priority
renice -n 5 -p <pid>
taskset -c 0-3 cmd              # pin to specific CPUs
```

---

## 3. Memory

```
   ┌──────────────────────────────────────────────────────────────┐
   │  PROCESS VIRTUAL ADDRESS SPACE                               │
   │  ┌────────────────────────────────────────────────────────┐  │
   │  │  Stack        ↓ grows down (locals, call frames)        │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  (gap)                                                  │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  mmap region  shared libs, large allocations, files      │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  Heap         ↑ grows up (malloc/brk)                    │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  BSS          uninitialized globals                      │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  Data         initialized globals                        │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  Text         the program code (read-only, shareable)    │  │
   │  └────────────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────────────┘
```

### RSS vs VSZ — the distinction that matters

```
   VSZ   Virtual Size — everything MAPPED, including memory
         never touched, shared libraries, and reserved regions.
         ⚠️ Often wildly larger than actual usage. Mostly useless
           for capacity planning. A JVM with -Xmx4g may show
           tens of GB of VSZ.

   RSS   Resident Set Size — physical RAM actually in use.
         ⭐ This is the number that matters.
         ⚠️ But shared pages are counted in EVERY process that
           maps them, so summing RSS across processes
           over-counts.

   PSS   Proportional Set Size — shared pages divided among
         the processes sharing them.
         ⭐ The correct metric for "how much memory does this
           process really cost." Read from /proc/<pid>/smaps.
```

### Page cache — why "free memory" is misleading

```
   $ free -h
                 total   used   free   shared  buff/cache  available
   Mem:           31Gi   8Gi    1Gi    0.5Gi        22Gi       21Gi
                                 ▲                    ▲          ▲
                          "only 1GB free!"      page cache   ⭐ THE
                           ⚠️ WRONG PANIC       (reclaimable) REAL
                                                             NUMBER

   ⭐ Linux uses ALL free RAM as disk cache. That's correct
     behaviour — unused RAM is wasted RAM. The cache is
     instantly reclaimable when a process needs memory.

   → Always read the `available` column, never `free`.
```

### The OOM killer

```
   When memory is exhausted and nothing can be reclaimed, the
   kernel kills a process to survive.

   SELECTION: each process gets an oom_score based on memory
   usage, adjusted by /proc/<pid>/oom_score_adj (-1000 to 1000).

   ⚠️ THE OOM KILLER OFTEN PICKS YOUR MAIN APPLICATION,
     because it's the largest consumer.

   $ dmesg -T | grep -i "killed process"
   [Thu Aug 14 10:23:11] Out of memory: Killed process 12345 (java)
                          total-vm:8000000kB, anon-rss:4000000kB

   ⭐ IN CONTAINERS, cgroup limits trigger a per-container OOM
     kill even when the host has plenty of free memory. Exit
     code 137 (128 + SIGKILL 9) is the signature.
     → check: kubectl describe pod → "OOMKilled"
```

### Swap

```
   Swap trades latency for capacity: pages move to disk when
   RAM is short.

   vm.swappiness  0-100, how eagerly the kernel swaps
     • 60  default — reasonable for desktops
     • 1-10 ⭐ typical for servers: swap only under real pressure
     • 0   almost never swap (still swaps to avoid OOM)

   ⚠️ SWAP THRASHING is worse than an OOM kill. A process
     swapping constantly is alive but useless, and it drags
     down everything sharing the disk. Many production systems
     disable swap entirely so failures are fast and obvious.
     ⭐ Kubernetes historically required swap disabled for
       exactly this predictability reason.
```

---

## 4. Filesystems & I/O

### File descriptors

```
   Every open file, socket, or pipe is an integer FD.

   0  stdin    1  stdout    2  stderr

   ⚠️ FD EXHAUSTION IS A CLASSIC PRODUCTION FAILURE
     "Too many open files" / EMFILE

   $ ulimit -n                    # soft limit for this shell
   $ cat /proc/<pid>/limits       # ⭐ what the PROCESS actually has
   $ ls /proc/<pid>/fd | wc -l    # how many are open right now
   $ lsof -p <pid> | head         # what they are

   ⭐ Usual causes: unclosed sockets (missing finally/defer),
     a connection pool without a cap, or a leak in a library.
     Watch the FD count over time — a steady climb is a leak.
```

### The I/O path

```
   Application
      │  write()
      ▼
   ┌──────────────────┐
   │  PAGE CACHE      │  ⭐ the write returns HERE — it is NOT
   │  (dirty pages)   │    on disk yet
   └────────┬─────────┘
            │ background writeback, or fsync()
            ▼
   ┌──────────────────┐
   │  BLOCK LAYER     │  I/O scheduler, request merging
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │  DEVICE          │
   └──────────────────┘

   ⭐ THE DURABILITY BOUNDARY IS fsync(), NOT write().
     A successful write() only means "the kernel has it."
     A power loss before writeback loses that data.
     This is why databases fsync the WAL on commit, and why
     that fsync is the dominant cost of a transaction.
```

### Diagnosing I/O

```bash
iostat -xz 1                    # ⭐ per-device utilization and latency
#   %util  → how busy the device is
#   await  → average I/O latency in ms  ⭐ the number that matters
#   aqu-sz → average queue depth

iotop -o                        # which processes are doing I/O
df -h                           # disk space
df -i                           # ⭐ INODES — you can be "full" with
                                #   free space if inodes are exhausted
du -sh * | sort -h              # what's using space
ncdu                            # interactive disk usage
```

```
   ⚠️ THE INODE TRAP
     A filesystem full of tiny files (session files, cache
     entries, log fragments) can exhaust inodes while df -h
     shows plenty of free space. Writes fail with ENOSPC and
     the cause is invisible unless you check `df -i`.
```

---

## 5. Signals

```
   ┌──────────────────────────────────────────────────────────────┐
   │ SIGTERM  15  ⭐ "please shut down" — CATCHABLE                │
   │              This is what Docker/Kubernetes send first.      │
   │              Your app should drain and exit cleanly.         │
   ├──────────────────────────────────────────────────────────────┤
   │ SIGKILL   9  ⚠️ "die now" — CANNOT be caught or ignored.      │
   │              No cleanup runs. Sent after the grace period.   │
   ├──────────────────────────────────────────────────────────────┤
   │ SIGINT    2  Ctrl-C                                          │
   │ SIGHUP    1  terminal closed; conventionally "reload config" │
   │ SIGSTOP  19  suspend (uncatchable) · SIGCONT 18 resume       │
   │ SIGSEGV  11  invalid memory access                           │
   │ SIGPIPE  13  ⭐ wrote to a closed socket/pipe — default is    │
   │              to TERMINATE, which surprises people            │
   │ SIGUSR1/2    application-defined (often "dump state")        │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE GRACEFUL SHUTDOWN SEQUENCE — get this right or you
     drop requests on every deploy

   1. Orchestrator sends SIGTERM
   2. ⭐ App IMMEDIATELY marks itself NOT READY
      → the load balancer stops sending new traffic
   3. App waits a few seconds for the LB to actually notice
      (this is what a preStop hook is for)
   4. App stops accepting new connections
   5. App finishes in-flight requests
   6. App closes DB/cache connections, flushes buffers
   7. App exits 0
   8. If it hasn't exited within the grace period → SIGKILL

   ⚠️ Step 2 is the one people skip, and it's why deploys drop
     requests. Closing the listener before the LB stops routing
     means connections get refused.
```

---

## 6. The Networking Stack

```
   ┌──────────────────────────────────────────────────────────────┐
   │ L7  APPLICATION   HTTP · gRPC · DNS · TLS                    │
   ├──────────────────────────────────────────────────────────────┤
   │ L4  TRANSPORT     TCP (reliable, ordered) · UDP (fast, lossy)│
   │                   ⭐ ports live here                          │
   ├──────────────────────────────────────────────────────────────┤
   │ L3  NETWORK       IP · routing · ICMP                        │
   │                   ⭐ IP addresses live here                   │
   ├──────────────────────────────────────────────────────────────┤
   │ L2  DATA LINK     Ethernet · ARP · MAC addresses · VLANs     │
   ├──────────────────────────────────────────────────────────────┤
   │ L1  PHYSICAL      cables, radio, optics                      │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY THE LAYERS MATTER IN PRACTICE
     "L4 load balancer" = routes on IP:port, doesn't read the
       request → very fast, but can't do path-based routing
     "L7 load balancer" = parses HTTP → path routing, TLS
       termination, header rewriting, but more CPU per request
```

### The journey of a packet

```
   curl https://api.example.com/users

   ① DNS      api.example.com → 93.184.216.34
   ② ARP      who has the gateway's IP? → MAC address
   ③ TCP      SYN → SYN-ACK → ACK               (1 RTT)
   ④ TLS      ClientHello → ServerHello → keys  (1 RTT with TLS 1.3)
   ⑤ HTTP     GET /users
   ⑥ Response streams back
   ⑦ Connection kept alive for reuse (or closed)

   ⭐ EACH STEP CAN FAIL DIFFERENTLY, and the error tells you where:
     DNS fail        → "could not resolve host"
     TCP refused     → "connection refused"  (nothing listening)
     TCP timeout     → "connection timed out" (firewall drop, or
                        the host is unreachable)
     TLS fail        → certificate error, protocol mismatch
     HTTP 5xx        → the app is reachable but broken

   ⭐ "Connection refused" vs "timed out" is the single most
     useful diagnostic distinction in networking:
       REFUSED = the packet ARRIVED, nothing was listening
       TIMEOUT = the packet was DROPPED (firewall, routing, or
                 a dead host)
```

---

## 7. TCP Deep Dive

### The three-way handshake

```
   Client                              Server
     │  ── SYN (seq=x) ──────────────────▶│
     │  ◀── SYN-ACK (seq=y, ack=x+1) ─────│
     │  ── ACK (ack=y+1) ────────────────▶│
     │                                    │
     │  ⭐ 1 full RTT before any data      │
     │     moves. This is why connection  │
     │     reuse (keep-alive) matters     │
     │     so much.                       │
```

### Connection teardown and TIME_WAIT

```
     │  ── FIN ──────────────────────────▶│
     │  ◀── ACK ──────────────────────────│
     │  ◀── FIN ──────────────────────────│
     │  ── ACK ──────────────────────────▶│
     │                                    │
     │  ⭐ TIME_WAIT (2×MSL, typically 60s)│
     │     The side that closes FIRST      │
     │     holds the socket to absorb      │
     │     delayed duplicate packets.      │
```

```
   ⚠️ TIME_WAIT EXHAUSTION — a real production failure

   A busy client that opens and closes many short connections
   accumulates TIME_WAIT sockets, exhausting ephemeral ports
   (~28,000 by default). New connections then fail.

   $ ss -s                          # summary including TIME_WAIT count
   $ sysctl net.ipv4.ip_local_port_range

   ⭐ THE REAL FIX: connection pooling / keep-alive. Don't open
     a new connection per request.

   Tuning (secondary):
     net.ipv4.tcp_tw_reuse = 1      # reuse TIME_WAIT for outbound
     net.ipv4.ip_local_port_range   # widen the range
     ⚠️ tcp_tw_recycle was REMOVED — it broke badly behind NAT
```

### Socket states worth recognizing

```
   LISTEN        waiting for connections
   ESTABLISHED   active connection
   SYN_SENT      ⚠️ many of these = the server isn't responding
   SYN_RECV      ⚠️ many of these = possible SYN flood
   TIME_WAIT     normal after closing; excessive = pooling problem
   CLOSE_WAIT    ⚠️⚠️ THE PEER CLOSED, YOUR APP HASN'T
                 → this is an APPLICATION BUG: you're not
                   closing sockets. It will leak FDs until
                   the process dies.
```

### Flow control vs congestion control

```
   FLOW CONTROL       protects the RECEIVER from being overwhelmed
                      → the receive window (rwnd) advertised in
                        every ACK

   CONGESTION CONTROL protects the NETWORK from being overwhelmed
                      → the congestion window (cwnd), inferred
                        from loss and delay

   ⭐ Effective send rate = min(rwnd, cwnd)
```

```
   SLOW START AND CONGESTION AVOIDANCE

   cwnd
     │        ╱╲          ╱╲
     │       ╱  ╲        ╱  ╲      ← congestion avoidance
     │      ╱    ╲      ╱         (linear growth)
     │     ╱      ╲    ╱
     │    ╱        ╲__╱           ← loss detected, cwnd cut
     │   ╱  slow start
     │  ╱   (exponential)
     └─────────────────────────── time

   ⭐ WHY THIS MATTERS FOR YOU
     A new connection starts SLOW — it must ramp up. That's
     another reason connection reuse beats reconnecting, and
     why the first request on a fresh connection is slower.

   ALGORITHMS
     CUBIC   Linux default; loss-based
     BBR ⭐   Google's; models bandwidth and RTT instead of
             treating loss as the only signal. Much better on
             lossy links (mobile, long-distance) and now widely
             deployed.
```

### TCP vs UDP

```
   ┌────────────────────┬────────────────────────────────────────┐
   │ TCP                │ UDP                                    │
   ├────────────────────┼────────────────────────────────────────┤
   │ Connection-oriented│ Connectionless                         │
   │ Reliable, ordered  │ Best-effort, unordered                 │
   │ Flow + congestion  │ None (you implement it if you need it) │
   │   control          │                                        │
   │ Higher overhead    │ Minimal header, no handshake           │
   │ Head-of-line       │ ⭐ No head-of-line blocking             │
   │   blocking         │                                        │
   ├────────────────────┼────────────────────────────────────────┤
   │ HTTP/1.1, HTTP/2,  │ DNS, DHCP, NTP, video/voice streaming, │
   │ databases, SSH     │ gaming, ⭐ QUIC (and therefore HTTP/3)  │
   └────────────────────┴────────────────────────────────────────┘

   ⭐ QUIC is UDP-based specifically to escape TCP's
     head-of-line blocking and to allow protocol evolution
     without waiting for OS kernel updates.
```

---

## 8. DNS

```
   RESOLUTION ORDER (each step is cached)

   browser cache → OS cache → /etc/hosts → configured resolver
        │
        └──▶ Root (.) ──▶ TLD (.com) ──▶ Authoritative NS ──▶ IP

   Cold lookup: ~20-120ms. Warm: ~0.
```

### Record types

```
   A       name → IPv4
   AAAA    name → IPv6
   CNAME   alias → another name  ⚠️ cannot coexist with other
                                   records at the same name, and
                                   cannot be used at the zone apex
   ALIAS/  ⭐ provider-specific CNAME-like record that DOES work
   ANAME     at the apex (Route53 ALIAS, Cloudflare CNAME
             flattening)
   MX      mail servers (with priority)
   TXT     arbitrary text — SPF, DKIM, domain verification
   NS      delegation to authoritative nameservers
   SOA     zone metadata, serial number
   SRV     service location (host + port) — used by Kubernetes
   CAA     which CAs may issue certificates for this domain
   PTR     reverse lookup (IP → name)
```

### TTL — the deployment lever

```
   LOW TTL (60s)     fast failover, heavy query load
   HIGH TTL (86400)  cheap and fast, but changes take a day

   ⭐ STANDARD PRACTICE
     • Lower TTL to 60s a day BEFORE a planned migration
     • Migrate
     • Raise it back afterward

   ⚠️ CLIENTS LIE ABOUT TTL
     Java historically cached DNS forever by default.
     Browsers pin. Some resolvers enforce minimums.
     ⭐ Never rely on DNS alone for failover — pair it with a
       load balancer that can shift traffic instantly.
```

```bash
dig api.example.com             # full query detail
dig +short api.example.com
dig @8.8.8.8 example.com        # query a specific resolver
dig +trace example.com          # ⭐ follow the full delegation chain
dig -x 93.184.216.34            # reverse lookup
host example.com
nslookup example.com
```

---

## 9. TLS

### The handshake

```
   TLS 1.2 — 2 RTT                    TLS 1.3 — 1 RTT ⭐
   ─────────────────                  ─────────────────
   ClientHello        ──▶             ClientHello + key share ──▶
              ◀── ServerHello                    ◀── ServerHello
                   Certificate                       + key share
                   ServerKeyExchange                 + Certificate
                   ServerHelloDone                   + Finished
   ClientKeyExchange  ──▶             Finished ──▶
   ChangeCipherSpec                   ⭐ application data can be
   Finished           ──▶               sent immediately
              ◀── ChangeCipherSpec
                   Finished

   ⭐ TLS 1.3 also removes all legacy cipher suites, mandates
     forward secrecy, and supports 0-RTT resumption (with a
     replay-attack caveat, so 0-RTT should only carry
     idempotent requests).
```

### The certificate chain

```
   ┌────────────────────────────────────────────────────────────┐
   │  ROOT CA                (in the OS/browser trust store)    │
   │      │ signs                                               │
   │      ▼                                                     │
   │  INTERMEDIATE CA        ⭐ must be sent by YOUR server      │
   │      │ signs                                               │
   │      ▼                                                     │
   │  LEAF (your certificate for api.example.com)               │
   └────────────────────────────────────────────────────────────┘

   ⚠️ THE #1 TLS MISCONFIGURATION: forgetting the intermediate.
     Browsers often paper over it by fetching the intermediate
     themselves; curl, Java, and Go clients usually do NOT.
     → "works in Chrome, fails from the server" is almost
       always a missing intermediate.
```

### Key concepts

```
   SNI (Server Name Indication)
     The client sends the requested hostname in the ClientHello,
     so one IP can serve many certificates.
     ⚠️ Historically sent in PLAINTEXT — it leaks which site you
       are visiting. Encrypted Client Hello (ECH) fixes this.

   FORWARD SECRECY (ECDHE)
     A fresh ephemeral key per session, so compromising the
     server's long-term private key does NOT decrypt past
     recorded traffic. ⭐ Mandatory in TLS 1.3.

   mTLS (mutual TLS)
     Both sides present certificates. ⭐ The standard for
     service-to-service authentication in a service mesh —
     it replaces network-location trust with identity.

   CERTIFICATE PINNING
     The client hardcodes an expected certificate/public key.
     ⚠️ Powerful but dangerous — a rotation mistake bricks all
       clients until they're updated.
```

```bash
# ⭐ The essential TLS debugging command
openssl s_client -connect example.com:443 -servername example.com

openssl x509 -in cert.pem -text -noout        # inspect a cert
openssl x509 -in cert.pem -noout -dates       # ⭐ check expiry
openssl verify -CAfile chain.pem cert.pem     # verify the chain

# Check what a server actually presents, including the chain
echo | openssl s_client -connect example.com:443 -showcerts
```

---

## 10. HTTP Evolution

```
   ┌──────────────────────────────────────────────────────────────┐
   │ HTTP/1.1                                                     │
   │   One request at a time per connection.                      │
   │   ⚠️ Head-of-line blocking at the application layer.          │
   │   Browsers open ~6 connections per host to compensate        │
   │   → domain sharding was a legitimate optimization.           │
   ├──────────────────────────────────────────────────────────────┤
   │ HTTP/2                                                       │
   │   ⭐ Multiplexed streams over ONE TCP connection              │
   │   Binary framing · HPACK header compression · priorities     │
   │   ⚠️ STILL suffers TCP-level head-of-line blocking: one lost  │
   │     packet stalls EVERY stream, because TCP must deliver     │
   │     bytes in order.                                          │
   │   → domain sharding now HURTS (it defeats multiplexing)      │
   ├──────────────────────────────────────────────────────────────┤
   │ HTTP/3 (QUIC over UDP)                                       │
   │   ⭐ Per-stream flow control — packet loss on one stream      │
   │     does NOT stall the others. This is the whole point.      │
   │   0-1 RTT handshake (TLS is built in, not layered on)        │
   │   ⭐ CONNECTION MIGRATION — a connection ID rather than a     │
   │     4-tuple, so switching WiFi→cellular doesn't drop it      │
   │   Runs in USER SPACE → protocol changes ship without         │
   │     kernel updates                                           │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE HEAD-OF-LINE BLOCKING PROGRESSION — a great interview answer

   HTTP/1.1: blocking at the APPLICATION layer
             (one request per connection at a time)
   HTTP/2:   fixed at the application layer, but TCP still
             delivers in order → blocking at the TRANSPORT layer
   HTTP/3:   fixed at both, by replacing TCP with QUIC
```

---

## 11. Diagnostic Playbook

### The USE method

```
   ⭐ For every resource, check three things:

   UTILIZATION   how busy is it?         (%CPU, %util, memory used)
   SATURATION    is work QUEUEING?       (load, run queue, aqu-sz)
   ERRORS        are operations failing? (dropped packets, I/O errors)

   ⭐ SATURATION IS THE MOST IMPORTANT AND MOST IGNORED.
     A disk at 100% utilization with a queue depth of 1 is fine.
     A disk at 70% utilization with a queue depth of 50 is a
     serious problem. Utilization alone misleads.
```

### The 60-second triage

```bash
uptime                  # load trend — rising, falling, or steady?
dmesg -T | tail -20     # ⭐ OOM kills, disk errors, driver problems
vmstat 1 5              # r (run queue) · si/so (swapping) · wa (I/O wait)
mpstat -P ALL 1 3       # ⭐ per-CPU — is ONE core pinned?
pidstat 1 3             # which process is consuming what
iostat -xz 1 3          # ⭐ per-device await and %util
free -h                 # ⭐ read `available`, not `free`
sar -n DEV 1 3          # network throughput per interface
ss -s                   # socket summary (TIME_WAIT, CLOSE_WAIT counts)
top                     # the overall picture
```

```
   ⭐ WHAT THE PATTERNS MEAN

   High load + high %CPU              → genuinely CPU bound
   High load + LOW %CPU + high %wa    → I/O bound, check iostat
   High load + LOW %CPU + LOW %wa     → many D-state processes,
                                         or lock contention
   One core at 100%, others idle      → single-threaded bottleneck
   si/so nonzero in vmstat            → ⚠️ SWAPPING, expect misery
   Many CLOSE_WAIT sockets            → ⚠️ application FD leak
   Many TIME_WAIT sockets             → connection churn, add pooling
```

### Network diagnosis

```bash
# ⭐ Modern replacements for netstat
ss -tlnp                # listening TCP sockets with process names
ss -tnp state established
ss -tn state time-wait | wc -l

ping -c 4 host          # basic reachability (⚠️ ICMP is often blocked)
mtr host                # ⭐ traceroute + ping combined, live — the
                        #   best single tool for "where is the loss?"
traceroute -T -p 443 host   # TCP traceroute (works when ICMP is blocked)

curl -v https://api.example.com          # verbose: DNS, TLS, headers
curl -w "@curl-format.txt" -o /dev/null -s URL   # ⭐ timing breakdown

nc -zv host 443         # ⭐ is the port open? refused vs timeout
telnet host 443

tcpdump -i any -nn port 443 -w cap.pcap   # capture for Wireshark
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0'   # SYNs only
```

```
   ⭐ curl-format.txt — the timing breakdown that isolates the
     slow phase

   time_namelookup:    %{time_namelookup}\n
   time_connect:       %{time_connect}\n
   time_appconnect:    %{time_appconnect}\n     ← TLS
   time_starttransfer: %{time_starttransfer}\n  ← TTFB (server time)
   time_total:         %{time_total}\n

   Reading it:
     namelookup high     → DNS problem
     connect high        → network latency or a slow path
     appconnect high     → TLS handshake cost
     starttransfer high  → ⭐ the SERVER is slow, not the network
     total >> starttransfer → the RESPONSE BODY is large or slow
```

---

## 12. Interview Section

<details>
<summary><b>Q1. What happens when you type a URL and press Enter? (network layer)</b></summary>

DNS resolution first — browser cache, OS cache, hosts file, then the configured resolver, which walks root, TLD, and authoritative nameservers if nothing is cached.

Then ARP to find the gateway's MAC address, since IP packets need a layer-2 destination on the local network.

TCP handshake: SYN, SYN-ACK, ACK — one full round trip before any data moves. That's a big part of why connection reuse matters.

Then TLS. In 1.3 it's one round trip because the client sends its key share in the ClientHello; in 1.2 it was two. TLS 1.3 also mandates forward secrecy.

Then the HTTP request, the server processes it, and the response streams back.

The diagnostic value is that each stage fails distinctly. "Could not resolve" is DNS. "Connection refused" means the packet arrived and nothing was listening. "Connection timed out" means the packet was dropped — a firewall or routing problem. A certificate error is TLS. A 5xx means the app is reachable but broken. That refused-versus-timeout distinction is probably the single most useful thing to know in network debugging.
</details>

<details>
<summary><b>Q2. A server has load average 50 but CPU is 10%. What's happening?</b></summary>

On Linux, load average includes processes in uninterruptible sleep — D state, waiting on I/O — not just runnable ones. That differs from most Unixes and is exactly the situation that reveals it.

So load 50 with low CPU almost certainly means I/O saturation. I'd confirm with `iostat -xz 1`, looking at `await` for per-device latency and queue depth rather than just `%util`.

The usual causes are a slow or failing disk, a storage backend problem, or an NFS mount that's hung — NFS hangs produce a lot of unkillable D-state processes.

If I/O looks clean, the other possibility is lock contention producing many processes blocked on the same resource, which I'd investigate by sampling stacks with `perf` or checking application-level lock metrics.

I'd also run `mpstat -P ALL` to check whether it's actually one core pinned rather than aggregate CPU being low — a single-threaded bottleneck can hide behind a low average.
</details>

<details>
<summary><b>Q3. Explain TIME_WAIT and CLOSE_WAIT. What do lots of each mean?</b></summary>

TIME_WAIT is on the side that closes first. After the four-way teardown, that socket is held for twice the maximum segment lifetime — typically 60 seconds — so delayed duplicate packets from the old connection can't be misinterpreted by a new connection reusing the same port pair. It's correct, expected behaviour.

Lots of TIME_WAIT means connection churn — you're opening and closing many short-lived connections. At high rates this exhausts ephemeral ports and new connections start failing. The real fix is connection pooling and keep-alive, not kernel tuning. `tcp_tw_reuse` helps for outbound connections; `tcp_tw_recycle` was removed from the kernel because it broke badly behind NAT.

CLOSE_WAIT is different and much worse. It means the *peer* closed and your application hasn't called close on its side. That's an application bug — a missing close in a finally block, or a leaked connection. Those sockets never go away on their own, so file descriptors leak until the process hits its limit and starts failing with "too many open files."

So: TIME_WAIT is a design smell, CLOSE_WAIT is a bug.
</details>

<details>
<summary><b>Q4. Why do containers need an init process?</b></summary>

Because your application usually runs as PID 1, and PID 1 has responsibilities most applications don't implement.

First, reaping. When a child process exits, it stays as a zombie until its parent calls wait. Orphaned processes get reparented to PID 1, which is expected to reap them. If your app doesn't, zombies accumulate and eventually exhaust the process table.

Second, and more commonly damaging, signal handling. PID 1 doesn't get default signal handlers — signals are only delivered if the process explicitly handles them. So if your app doesn't install a SIGTERM handler, SIGTERM does nothing. Kubernetes sends SIGTERM, waits the entire termination grace period, then SIGKILLs you mid-request. Every deploy drops connections and nobody knows why.

The fix is a minimal init like tini or dumb-init as PID 1, or `docker run --init`, which forwards signals to your process and reaps orphans. Kubernetes doesn't inject one automatically, so it has to be in your image.
</details>

<details>
<summary><b>Q5. `free -h` shows almost no free memory. Is that a problem?</b></summary>

Almost certainly not. Linux uses all otherwise-idle RAM as page cache for disk data, because unused RAM is wasted RAM. That cache is instantly reclaimable the moment a process needs memory.

The column to read is `available`, not `free`. Available accounts for reclaimable cache and is the honest answer to "how much can a new process get."

The real warning signs are different: `si` and `so` non-zero in `vmstat` means you're actually swapping, which is far worse than being out of cache. And OOM kills in `dmesg` mean you genuinely ran out.

In containers there's an extra subtlety — cgroup limits trigger a per-container OOM kill even when the host has plenty free. Exit code 137 is the signature, and `kubectl describe pod` shows OOMKilled.
</details>

<details>
<summary><b>Q6. Walk me through diagnosing "the API is slow."</b></summary>

First, establish where the time goes, because "slow" could be network, server, or client.

`curl` with a write-out format gives the breakdown: namelookup, connect, appconnect for TLS, starttransfer for time-to-first-byte, and total. If starttransfer is high but connect is low, the network is fine and the server is slow. If total is much larger than starttransfer, the response body is large or streaming slowly.

If it's the server, I'd go to the USE method on the box: utilization, saturation, and errors for CPU, memory, disk, and network. `vmstat`, `mpstat -P ALL` to check for a single pinned core, `iostat -xz` for disk latency, and `ss -s` for socket state anomalies.

If system resources look fine, it's inside the application — at which point I'd move to application profiling and distributed tracing, since a slow downstream dependency looks identical to slow application code from the outside.

If it's intermittent rather than constant, I'd suspect a gray failure: one bad instance behind a load balancer poisoning a fraction of requests. Per-instance latency percentiles reveal that immediately, whereas aggregate metrics hide it.
</details>

<details>
<summary><b>Q7. TCP vs UDP, and why is HTTP/3 on UDP?</b></summary>

TCP gives reliable ordered delivery with flow and congestion control, at the cost of a handshake and head-of-line blocking. UDP is a thin wrapper over IP — no handshake, no ordering, no reliability.

HTTP/3 uses UDP because of head-of-line blocking specifically. HTTP/2 multiplexes many streams over one TCP connection, which solved application-layer blocking. But TCP guarantees in-order byte delivery, so a single lost packet stalls *every* multiplexed stream until it's retransmitted. The fix moved the problem down a layer rather than eliminating it.

QUIC implements reliability, ordering, and congestion control per stream, on top of UDP. Loss on one stream doesn't affect the others.

Two other wins fall out. TLS is integrated rather than layered, so the handshake is zero or one round trip instead of TCP's plus TLS's. And connections are identified by a connection ID rather than the IP-port four-tuple, so switching from WiFi to cellular doesn't drop the connection — which matters enormously on mobile.

Running in user space also means protocol improvements ship with the application rather than waiting years for OS kernel updates.
</details>

<details>
<summary><b>Q8. Why does "works in the browser, fails from curl" happen with TLS?</b></summary>

Nearly always a missing intermediate certificate.

The chain runs from a root CA in the trust store, through one or more intermediates, down to your leaf certificate. Your server must send the leaf *and* the intermediates — the client only has the root.

Browsers hide the mistake. They cache intermediates from previous sites and many will fetch a missing one via the Authority Information Access extension. curl, Java, Go, and most server-side HTTP clients do neither, so they fail validation.

I'd confirm with `openssl s_client -connect host:443 -showcerts` and check whether the full chain is presented. The fix is concatenating the intermediate into the certificate file your server serves.

The other candidates, less common: a hostname mismatch where SNI isn't being sent correctly, an expired certificate that a browser is caching past, or a protocol/cipher mismatch with an old client.
</details>

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                 LINUX & NETWORKING — ONE PAGE                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ LOAD on Linux INCLUDES I/O WAIT, not just CPU                      ║
║   high load + low CPU + high %wa → I/O bound, check iostat await     ║
║ ⭐ free -h: read `available`, NOT `free` — cache is reclaimable       ║
║ ⭐ RSS = real memory · VSZ = mapped (mostly meaningless) · PSS = fair ║
║ OOM: dmesg | grep -i killed · container exit code 137 = OOMKilled    ║
║ D state = uninterruptible I/O, can't even be SIGKILLed               ║
╠══════════════════════════════════════════════════════════════════════╣
║ SIGNALS: SIGTERM(15) catchable ← what orchestrators send             ║
║          SIGKILL(9) uncatchable, no cleanup                          ║
║ ⭐ GRACEFUL SHUTDOWN: readiness FALSE first, wait for the LB, THEN    ║
║   stop accepting, drain, close, exit                                 ║
║ ⭐ CONTAINERS NEED AN INIT (tini/dumb-init/--init) or SIGTERM may     ║
║   never reach your app and zombies accumulate                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ REFUSED vs TIMEOUT is the key diagnostic                           ║
║   refused = packet ARRIVED, nothing listening                        ║
║   timeout = packet DROPPED (firewall/routing/dead host)              ║
║ TIME_WAIT (many) → connection churn, add POOLING                     ║
║ ⚠️ CLOSE_WAIT (many) → YOUR APP isn't closing sockets = FD leak bug   ║
║ FD exhaustion: ls /proc/<pid>/fd | wc -l · cat /proc/<pid>/limits    ║
║ ⚠️ df -i for INODES — you can be full with free space                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ TCP: 1 RTT handshake → REUSE CONNECTIONS (also avoids slow start)    ║
║   flow control = protect RECEIVER · congestion = protect NETWORK     ║
║ HTTP/1.1 app-layer HOL → HTTP/2 fixes it but TCP-layer HOL remains   ║
║   → HTTP/3 (QUIC/UDP) fixes both + connection migration              ║
╠══════════════════════════════════════════════════════════════════════╣
║ DNS: lower TTL to 60s BEFORE a migration; clients ignore TTL anyway  ║
║ TLS: ⭐ missing INTERMEDIATE = "works in browser, fails from curl"    ║
║   openssl s_client -connect host:443 -servername host -showcerts     ║
╠══════════════════════════════════════════════════════════════════════╣
║ USE METHOD: Utilization · SATURATION (most ignored) · Errors         ║
║ 60-SEC TRIAGE: uptime · dmesg -T · vmstat 1 · mpstat -P ALL ·        ║
║   pidstat · iostat -xz · free -h · ss -s · top                       ║
║ ⭐ BEST TOOLS: mtr (where is the loss) · strace -p (what is it        ║
║   waiting on) · lsof -p (FD leaks) · curl -w (which phase is slow)   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Docker](docker.md) · [Kubernetes](kubernetes.md) · [Observability & SRE](observability-sre.md) · [Network Security](../07-security/network-security.md)
