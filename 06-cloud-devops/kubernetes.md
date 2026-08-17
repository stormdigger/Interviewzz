# ☸️ Kubernetes

> Kubernetes is a control loop. You declare desired state; controllers continuously work to make reality match. Every feature is an instance of that one idea.

**Prerequisite:** [Docker](docker.md) · [Linux & Networking](linux-networking.md)

---

## 📑 Contents

1. [The Mental Model](#1-the-mental-model)
2. [Architecture](#2-architecture)
3. [Pods](#3-pods)
4. [Workload Controllers](#4-workload-controllers)
5. [Networking](#5-networking)
6. [Configuration & Secrets](#6-configuration--secrets)
7. [Storage](#7-storage)
8. [Resources & Scheduling](#8-resources--scheduling)
9. [Health Probes](#9-health-probes)
10. [Autoscaling](#10-autoscaling)
11. [Security](#11-security)
12. [Debugging Playbook](#12-debugging-playbook)
13. [Production Checklist](#13-production-checklist)
14. [Interview Section](#14-interview-section)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. The Mental Model

#### 💬 The one idea

```
   ⭐ KUBERNETES IS A SET OF CONTROL LOOPS.

   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │    DESIRED STATE  ──────┐                                    │
   │    (what you declared)  │                                    │
   │                         ▼                                    │
   │                    ┌─────────┐                               │
   │                    │ COMPARE │                               │
   │                    └────┬────┘                               │
   │                         │  they differ?                      │
   │                         ▼                                    │
   │                    ┌─────────┐                               │
   │                    │   ACT   │  create/delete/update         │
   │                    └────┬────┘                               │
   │                         │                                    │
   │    ACTUAL STATE  ◀──────┘                                    │
   │    (what exists)        │                                    │
   │         └───────────────┘  repeat forever                    │
   └──────────────────────────────────────────────────────────────┘

   You NEVER tell Kubernetes to "start a container." You declare
   "I want 3 replicas of this," and a controller notices there
   are 2 and creates one.

   ⭐ THIS EXPLAINS EVERYTHING:
     • Why deleting a pod recreates it (the ReplicaSet controller notices)
     • Why kubectl apply is idempotent
     • Why "it's stuck in Pending" means a controller CAN'T act
     • Why you edit the Deployment, not the Pod
```

### Declarative vs imperative

```
   IMPERATIVE (how)                DECLARATIVE (what)  ⭐
   docker run -d nginx             kind: Deployment
   docker stop abc123                replicas: 3
   docker run -d nginx             (Kubernetes figures out how)

   ✅ Self-healing — a node dies, pods are rescheduled automatically
   ✅ Version-controllable — the YAML IS the system state
   ✅ Idempotent — apply the same file a hundred times safely
```

---

## 2. Architecture

```
   ┌───────────────── CONTROL PLANE ──────────────────────────────┐
   │                                                              │
   │  ┌────────────────┐   ⭐ The ONLY component that talks to     │
   │  │  API SERVER    │      etcd. Everything else goes through   │
   │  │  (kube-apiserver)     the API server.                      │
   │  │  auth · admission · validation · the front door            │
   │  └───────┬────────┘                                          │
   │          │                                                   │
   │  ┌───────▼────────┐  ┌──────────────┐  ┌──────────────────┐  │
   │  │      etcd      │  │  SCHEDULER   │  │   CONTROLLER     │  │
   │  │  ⭐ the ONLY    │  │              │  │    MANAGER       │  │
   │  │  stateful part │  │ assigns pods │  │ runs the control │  │
   │  │  (Raft, odd #) │  │ to nodes     │  │ loops            │  │
   │  └────────────────┘  └──────────────┘  └──────────────────┘  │
   │                                        ┌──────────────────┐  │
   │                                        │ CLOUD CONTROLLER │  │
   │                                        │ (LBs, volumes)   │  │
   │                                        └──────────────────┘  │
   └──────────────────────────────────────────────────────────────┘
                              │
   ┌──────────────────────────┼───────────────────────────────────┐
   │                    WORKER NODES                              │
   │  ┌────────────────────────────────────────────────────────┐  │
   │  │  KUBELET     ⭐ the node agent. Watches the API server   │  │
   │  │              for pods assigned to it, then makes them   │  │
   │  │              real via the container runtime.            │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  KUBE-PROXY  programs iptables/IPVS for Service routing │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  CONTAINER RUNTIME  (containerd / CRI-O)               │  │
   │  ├────────────────────────────────────────────────────────┤  │
   │  │  PODS  ──  PODS  ──  PODS                              │  │
   │  └────────────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHAT HAPPENS WHEN YOU `kubectl apply -f deploy.yaml`

   ① kubectl → API server (authenticate, authorize via RBAC)
   ② Admission controllers mutate then validate the object
      (defaults injected, policies enforced, sidecars added)
   ③ The object is persisted to etcd. ⭐ At this point kubectl
      returns — nothing is running yet.
   ④ Deployment controller notices a Deployment with no ReplicaSet
      → creates a ReplicaSet
   ⑤ ReplicaSet controller notices 0 pods but wants 3
      → creates 3 Pod objects (⚠️ still unscheduled, no node)
   ⑥ Scheduler notices pods with no nodeName
      → filters feasible nodes, scores them, binds each pod
   ⑦ Kubelet on each node notices a pod assigned to it
      → pulls the image, creates containers, reports status
   ⑧ Status flows back up to etcd and becomes visible to kubectl

   ⭐ EVERY STEP IS AN INDEPENDENT CONTROLLER WATCHING THE API.
     Nothing calls anything directly. That's why the system is
     resilient — any component can restart and resume from
     observed state.
```

```
   ⚠️ etcd IS THE WHOLE CLUSTER
     • Odd number of members (3 or 5) for Raft quorum
     • Very sensitive to disk latency — use fast SSDs
     • ⭐ BACK IT UP. An etcd backup is your cluster backup.
     • Size limits matter: large ConfigMaps/Secrets bloat it
```

---

## 3. Pods

#### 💬 What a pod actually is

```
   ⭐ A POD IS A SHARED NAMESPACE ENVELOPE, not "a container."

   Containers in a pod SHARE:
     • the network namespace ⭐ → same IP, reach each other on
       localhost, and they cannot bind the same port
     • the IPC namespace
     • volumes you mount into both

   They do NOT share:
     • the filesystem (each has its own image)
     • the PID namespace by default (opt in with shareProcessNamespace)

   ⭐ THE PAUSE CONTAINER
     Every pod has a hidden "pause" container that does nothing
     but hold the namespaces open. That's why your app container
     can crash and restart while the pod keeps its IP.
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  # ⭐ INIT CONTAINERS run to completion, in order, BEFORE app containers
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 1; done']

  containers:
    - name: app
      image: myapp:1.2.3
      ports:
        - containerPort: 8080
      resources:
        requests: { memory: "256Mi", cpu: "250m" }
        limits:   { memory: "512Mi" }        # ⭐ note: no CPU limit
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef: { fieldPath: metadata.name }
      volumeMounts:
        - name: config
          mountPath: /etc/config

    # SIDECAR — runs alongside the app for the pod's lifetime
    - name: log-shipper
      image: fluent-bit:3.0

  volumes:
    - name: config
      configMap: { name: app-config }
```

### Multi-container patterns

```
   ┌──────────────────────────────────────────────────────────────┐
   │ SIDECAR      An auxiliary container running alongside the    │
   │              app: log shipping, a service-mesh proxy,        │
   │              metric exporters, config reloaders.             │
   │              ⭐ K8s 1.29+ has native sidecars (init           │
   │                containers with restartPolicy: Always) which  │
   │                start before and stop after the app —         │
   │                fixing the old "sidecar dies first and the    │
   │                app can't send its final logs" problem.       │
   ├──────────────────────────────────────────────────────────────┤
   │ AMBASSADOR   Proxies outbound connections, so the app just   │
   │              talks to localhost and the ambassador handles   │
   │              discovery, retries, TLS.                        │
   ├──────────────────────────────────────────────────────────────┤
   │ ADAPTER      Normalizes the app's output into a standard     │
   │              format (e.g. converts custom metrics to         │
   │              Prometheus format).                             │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ DON'T put multiple independent services in one pod. They'd
     scale together, restart together, and share a lifecycle
     they don't want. One pod = one deployable unit.
```

### Pod lifecycle

```
   Pending    ⭐ accepted but not running — usually UNSCHEDULABLE
              (insufficient resources, no matching node, PVC
              not bound) or still pulling the image
   Running    at least one container is running
   Succeeded  all containers exited 0 (Jobs)
   Failed     all terminated, at least one failed
   Unknown    the node stopped reporting

   ⭐ "Pending" almost always means the SCHEDULER can't place it.
     kubectl describe pod → look at Events for the reason.
```

---

## 4. Workload Controllers

```
   ┌──────────────────────────────────────────────────────────────┐
   │ DEPLOYMENT     ⭐ stateless apps. Manages ReplicaSets to give │
   │                rolling updates and rollback.                 │
   │                Pods are interchangeable, get random names.   │
   ├──────────────────────────────────────────────────────────────┤
   │ STATEFULSET    Stateful apps needing STABLE IDENTITY:        │
   │                • ordinal names: db-0, db-1, db-2             │
   │                • stable DNS: db-0.db-headless.ns.svc         │
   │                • ⭐ its OWN persistent volume per replica     │
   │                • ordered, sequential rollout and scale-down  │
   │                Use for: databases, Kafka, ZooKeeper          │
   ├──────────────────────────────────────────────────────────────┤
   │ DAEMONSET      Exactly one pod per node (or per matching     │
   │                node). Use for: log collectors, CNI agents,   │
   │                node exporters, security agents               │
   ├──────────────────────────────────────────────────────────────┤
   │ JOB            Run to completion, with retries               │
   │ CRONJOB        Job on a schedule                             │
   └──────────────────────────────────────────────────────────────┘
```

### Deployment and rolling updates

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: web }     # ⚠️ IMMUTABLE after creation
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                 # extra pods allowed above replicas
      maxUnavailable: 0           # ⭐ 0 = never lose capacity
  template:
    metadata:
      labels: { app: web }        # must match the selector
    spec:
      terminationGracePeriodSeconds: 60   # ⭐ > your longest request
      containers:
        - name: web
          image: web:1.2.3
```

```
   ⭐ ROLLING UPDATE with maxSurge=1, maxUnavailable=0

   start:   [v1][v1][v1]
   step 1:  [v1][v1][v1][v2]        ← surge: create the new one FIRST
   step 2:  [v1][v1]    [v2]        ← only then remove an old one
   step 3:  [v1][v1][v2][v2]
   ...
   end:     [v2][v2][v2]

   ⭐ maxUnavailable: 0 guarantees you never drop below the
     desired capacity — the safest setting, at the cost of
     temporarily needing extra resources.
```

```bash
kubectl set image deployment/web web=web:1.2.4
kubectl rollout status deployment/web           # ⭐ blocks until done
kubectl rollout history deployment/web
kubectl rollout undo deployment/web             # ⭐ instant rollback
kubectl rollout undo deployment/web --to-revision=3
kubectl rollout restart deployment/web          # recreate all pods
kubectl rollout pause/resume deployment/web     # for manual canary
```

### StatefulSet — when identity matters

```
   ┌──────────────────────────────────────────────────────────────┐
   │  DEPLOYMENT              STATEFULSET                         │
   │  web-7d4b8c-x9k2p        db-0  ⭐ stable, predictable         │
   │  web-7d4b8c-m4n1q        db-1                                │
   │  web-7d4b8c-p2j8r        db-2                                │
   │                                                              │
   │  random names            ordinal names                       │
   │  shared/no volume        ⭐ own PVC per pod, retained across  │
   │                            restarts AND rescheduling         │
   │  any order               ⭐ ordered: 0, then 1, then 2         │
   │                            scale down in REVERSE order       │
   │  no stable DNS           ⭐ db-0.db-headless.ns.svc.cluster.local│
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY ORDERING MATTERS: a database cluster often needs its
     first node to become the primary before replicas join.
     Ordered startup makes that deterministic.

   ⚠️ StatefulSets do NOT make your app distributed. They give
     it stable identity and storage. The clustering logic is
     still your application's problem.
```

---

## 5. Networking

### The four fundamental rules

```
   ⭐ THE KUBERNETES NETWORK MODEL

   1. Every POD gets its own unique IP
   2. Pods can reach every other pod DIRECTLY — no NAT
   3. Agents on a node can reach all pods on that node
   4. A pod sees its own IP as the same one others use to reach it

   → ⭐ NO PORT MAPPING. Unlike Docker, you don't map ports.
     Every pod is a first-class network citizen.
     This is what makes service discovery simple.
```

### Services

```
   ┌──────────────────────────────────────────────────────────────┐
   │ ClusterIP (default)  Internal-only stable virtual IP + DNS   │
   │                      ⭐ Pods come and go; the Service IP      │
   │                        never changes. This is the point.     │
   ├──────────────────────────────────────────────────────────────┤
   │ NodePort             Opens a port (30000-32767) on EVERY     │
   │                      node. ⚠️ Rarely right in production.     │
   ├──────────────────────────────────────────────────────────────┤
   │ LoadBalancer         Provisions a cloud load balancer.       │
   │                      ⚠️ One LB (and one bill) per Service.    │
   ├──────────────────────────────────────────────────────────────┤
   │ ExternalName         DNS CNAME to an external host. No proxy.│
   ├──────────────────────────────────────────────────────────────┤
   │ Headless (None)      ⭐ No virtual IP. DNS returns the pod    │
   │                      IPs directly. Required for StatefulSets │
   │                      and for client-side load balancing.     │
   └──────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: web }        # ⭐ matches POD labels — this is the
  ports:                        #   entire mechanism of pod selection
    - port: 80                  # the Service's port
      targetPort: 8080          # the container's port
```

```
   ⭐ HOW A SERVICE ACTUALLY WORKS

   Service (ClusterIP 10.96.0.10)
        │
        │  ⭐ The Service controller watches pods matching the
        │    selector and maintains an EndpointSlice listing
        │    their IPs.
        ▼
   EndpointSlice: [10.244.1.5:8080, 10.244.2.7:8080, ...]
        │
        ▼
   kube-proxy programs iptables/IPVS rules on EVERY node:
     "traffic to 10.96.0.10:80 → DNAT to a random endpoint"

   ⚠️ THE SERVICE IP IS VIRTUAL. Nothing listens on it. You
     cannot ping it. It exists only as an iptables rule.
```

```
   ⭐ SERVICE DNS

   <service>.<namespace>.svc.cluster.local

   Within the same namespace:  http://web
   Cross-namespace:            http://web.production
   Fully qualified:            http://web.production.svc.cluster.local

   ⚠️ A SUBTLE PERFORMANCE TRAP: `ndots:5` in the pod's
     resolv.conf means any name with fewer than 5 dots is
     tried against every search domain first. Looking up
     "api.example.com" generates 4 failed queries before the
     right one. ⭐ Use a trailing dot ("api.example.com.") for
     external names in hot paths, or tune dnsConfig.
```

### Ingress and Gateway API

```
   ⚠️ ONE LoadBalancer Service PER APP = one cloud LB per app.
     Expensive and unmanageable at scale.

   ⭐ INGRESS: one load balancer, many routes.

   ┌───────────────────────────────────────────────────────────┐
   │              INGRESS CONTROLLER (nginx / Traefik)         │
   │              ← ONE cloud LoadBalancer                     │
   │                                                           │
   │   api.example.com/*      → api-service                    │
   │   app.example.com/*      → web-service                    │
   │   app.example.com/admin  → admin-service                  │
   │   + TLS termination, cert-manager for automatic certs     │
   └───────────────────────────────────────────────────────────┘

   ⚠️ Ingress needs a CONTROLLER installed. The Ingress resource
     alone does nothing — a very common beginner confusion.

   ⭐ GATEWAY API is the successor: role-oriented (infra team owns
     the Gateway, app teams own HTTPRoutes), protocol-aware
     beyond HTTP, and standardizes what used to require
     vendor-specific annotations.
```

### NetworkPolicy

```
   ⚠️ BY DEFAULT, ALL PODS CAN TALK TO ALL PODS.
     A compromised frontend can reach your database directly.

   ⭐ NetworkPolicy is a per-namespace firewall — but it needs
     a CNI that implements it (Calico, Cilium; NOT flannel).
```

```yaml
# ⭐ Start with default-deny, then allow explicitly
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-ingress, namespace: prod }
spec:
  podSelector: {}                 # all pods in the namespace
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-allows-web, namespace: prod }
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: web }
      ports:
        - { protocol: TCP, port: 8080 }
```

---

## 6. Configuration & Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
```

```yaml
containers:
  - name: app
    envFrom:
      - configMapRef: { name: app-config }    # all keys as env vars
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef: { name: db-secret, key: password }
    volumeMounts:
      - name: config
        mountPath: /etc/config                # ⭐ mounted files can
                                              #   hot-reload; env vars
                                              #   CANNOT
```

```
   ⚠️⚠️ KUBERNETES SECRETS ARE ONLY BASE64-ENCODED, NOT ENCRYPTED

   $ kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
   → the plaintext password

   ⭐ MAKE THEM ACTUALLY SECRET:
     • Enable encryption at rest for etcd (EncryptionConfiguration)
     • RBAC — very few principals should read secrets
     • ⭐ External secret stores: External Secrets Operator with
       Vault / AWS Secrets Manager / GCP Secret Manager
     • ⭐ Better still: workload identity (IRSA, GKE Workload
       Identity) so short-lived credentials are issued
       automatically and there's no static secret to steal
     • Sealed Secrets or SOPS if you must commit them to git
     • ⚠️ Never `kubectl create secret` with a literal in shell
       history, and never commit plain Secret YAML
```

```
   ⭐ CONFIG CHANGES DON'T RESTART PODS

   Updating a ConfigMap does NOT restart pods using it.
     • Mounted as a volume → the file updates within ~60s, but
       your app must WATCH the file to notice
     • As env vars → ⚠️ NEVER updates; requires a pod restart

   ⭐ THE STANDARD FIX: put a hash of the config in the pod
     template annotations, so any config change alters the
     template and triggers a rolling update automatically.
     (Helm: `checksum/config: {{ include (print $.Template.BasePath
     "/configmap.yaml") . | sha256sum }}`)
```

---

## 7. Storage

```
   ┌──────────────────────────────────────────────────────────────┐
   │  StorageClass    ⭐ defines HOW to provision (disk type,      │
   │                  IOPS, replication, reclaim policy)          │
   │        │  dynamic provisioning                               │
   │        ▼                                                     │
   │  PersistentVolume (PV)     the actual storage resource       │
   │        ▲                                                     │
   │        │  bound 1:1                                          │
   │  PersistentVolumeClaim (PVC)  ⭐ the pod's REQUEST for storage│
   │        ▲                                                     │
   │        │                                                     │
   │      POD                                                     │
   └──────────────────────────────────────────────────────────────┘
```

```
   ACCESS MODES
   ReadWriteOnce (RWO)    ⭐ one NODE can mount read-write.
                          Most block storage (EBS, PD). This is
                          why an RWO PVC pins pods to one node.
   ReadOnlyMany (ROX)     many nodes, read-only
   ReadWriteMany (RWX)    many nodes read-write — needs a shared
                          filesystem (EFS, NFS, CephFS)
   ReadWriteOncePod       exactly one POD (not just one node)

   ⚠️ RWO IS THE #1 STORAGE GOTCHA. A Deployment with replicas: 3
     and an RWO PVC will have 2 pods stuck Pending forever,
     because only one node can attach the volume.
     ⭐ Use a StatefulSet with volumeClaimTemplates so each
       replica gets its own volume.
```

```yaml
# StatefulSet: each replica gets its OWN PVC automatically
volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast-ssd
      resources: { requests: { storage: 100Gi } }
```

```
   ⚠️ RECLAIM POLICY
     Delete   (often the default) — ⚠️ deleting the PVC DESTROYS
              the data. This has caused real data loss.
     Retain   ⭐ the PV survives; you clean it up manually.
              Use this for anything you care about.
```

---

## 8. Resources & Scheduling

#### 💬 Requests vs limits — the most consequential setting

```
   ┌──────────────────────────────────────────────────────────────┐
   │ REQUESTS   ⭐ What the SCHEDULER uses to place the pod.       │
   │            A guarantee: this much is reserved for you.       │
   │            → Set this based on typical usage.                │
   ├──────────────────────────────────────────────────────────────┤
   │ LIMITS     The hard ceiling enforced by cgroups.             │
   │            memory over limit → ⚠️ OOM KILLED (exit 137)       │
   │            CPU over limit    → ⚠️ THROTTLED (latency spikes)  │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐⭐ THE MOST IMPORTANT PRACTICAL ADVICE

   ALWAYS set memory REQUESTS and LIMITS, and set them EQUAL.
     → Guaranteed QoS class, predictable behaviour, and the
       kernel kills you rather than degrading the whole node.

   ⭐ CONSIDER NOT SETTING A CPU LIMIT AT ALL.
     CPU is compressible — the scheduler already prevents
     oversubscription using requests. A CPU limit only adds
     throttling, which produces mysterious latency spikes with
     no errors. This is genuinely counterintuitive and widely
     recommended by SRE practitioners.

     Set a CPU limit only when you must protect noisy neighbours
     in a multi-tenant cluster.
```

```
   QoS CLASSES (determine eviction order under node pressure)

   Guaranteed  requests == limits for ALL containers
               ⭐ evicted LAST. Use for critical workloads.
   Burstable   requests < limits (or only requests set)
               evicted second
   BestEffort  no requests or limits at all
               ⚠️ evicted FIRST. Never use in production.
```

```
   ⚠️ CPU THROTTLING IS INVISIBLE WITHOUT THE RIGHT METRIC

   A pod at 40% average CPU can still be throttled badly, because
   the quota is enforced per 100ms period. A GC pause or a burst
   consumes the whole quota and the app stalls for the rest of
   the period.

   ⭐ Monitor: container_cpu_cfs_throttled_seconds_total
     not just CPU utilization.
```

### Scheduling controls

```yaml
# Node selection
nodeSelector: { disktype: ssd }

# Richer expression + soft preference
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - { key: zone, operator: In, values: [us-east-1a, us-east-1b] }

  # ⭐ Spread replicas across nodes for availability
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels: { app: web }
          topologyKey: kubernetes.io/hostname

# ⭐ Modern, simpler alternative to anti-affinity
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels: { app: web }
```

```
   ⭐ TAINTS AND TOLERATIONS — the inverse of affinity

   TAINT on a node:     "keep pods away unless they tolerate this"
   TOLERATION on a pod: "I accept this taint"

   kubectl taint nodes gpu-1 gpu=true:NoSchedule

   Use for: dedicating nodes to specific workloads (GPU, licensed
   software), or draining nodes for maintenance.

   ⚠️ A toleration does NOT attract a pod to the node — it only
     permits it. Pair with nodeAffinity to actually target it.
```

```
   ⭐ PodDisruptionBudget — protect availability during
     VOLUNTARY disruptions (node drains, cluster upgrades)

   apiVersion: policy/v1
   kind: PodDisruptionBudget
   spec:
     minAvailable: 2          # or maxUnavailable: 1
     selector:
       matchLabels: { app: web }

   → `kubectl drain` will BLOCK rather than take you below this.
   ⚠️ Without a PDB, a cluster upgrade can evict every replica
     at once. This is a very common cause of self-inflicted
     outages during maintenance.
```

---

## 9. Health Probes

```
   ┌──────────────────────────────────────────────────────────────┐
   │ LIVENESS    "is this process broken?"                        │
   │             FAIL → the container is RESTARTED                │
   │             ⚠️ MUST NOT check dependencies. If your DB is     │
   │               down and liveness checks it, every pod         │
   │               restarts simultaneously and you turn a         │
   │               database problem into a total outage.          │
   ├──────────────────────────────────────────────────────────────┤
   │ READINESS   "can I serve traffic right now?"                 │
   │             FAIL → removed from Service endpoints,           │
   │             but NOT restarted                                │
   │             ⭐ This is the one that matters most. It's how    │
   │               you avoid sending traffic to a pod that's      │
   │               still warming up or is temporarily degraded.   │
   ├──────────────────────────────────────────────────────────────┤
   │ STARTUP     "has it finished booting?"                       │
   │             Disables the other two until it passes.          │
   │             ⭐ For slow-starting apps (JVM). Prevents a       │
   │               liveness probe from killing a pod that's       │
   │               simply still starting.                         │
   └──────────────────────────────────────────────────────────────┘
```

```yaml
startupProbe:                       # ⭐ allows up to 5 minutes to boot
  httpGet: { path: /health/live, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10

livenessProbe:
  httpGet: { path: /health/live, port: 8080 }   # ⭐ process only
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }  # ⭐ deps included
  periodSeconds: 5
  failureThreshold: 2
```

### Zero-downtime shutdown

```
   ⚠️ THE RACE THAT DROPS REQUESTS ON EVERY DEPLOY

   When a pod is deleted, TWO things happen IN PARALLEL:
     ① kubelet sends SIGTERM to the container
     ② the endpoint is removed from the Service

   ⭐ ② PROPAGATES SLOWLY — kube-proxy on every node must update
     its iptables rules. Meanwhile traffic is still arriving at
     a pod that has already begun shutting down.

   THE FIX:
```

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]   # ⭐ absorb in-flight routing
terminationGracePeriodSeconds: 60         # ⭐ > preStop + longest request
```

```
   ⭐ THE CORRECT SHUTDOWN SEQUENCE
   1. Pod marked Terminating; endpoint removal begins
   2. preStop hook runs (sleep) — traffic still arrives and is
      served normally while kube-proxy catches up
   3. SIGTERM sent to the container
   4. App flips readiness false, stops accepting new connections,
      drains in-flight requests
   5. App closes DB/cache connections and exits 0
   6. If still running after terminationGracePeriodSeconds → SIGKILL

   ⚠️ And the container must actually RECEIVE SIGTERM — use exec-form
     CMD and an init process. See [Docker §3](docker.md#3-writing-a-good-dockerfile).
```

---

## 10. Autoscaling

```
   ┌──────────────────────────────────────────────────────────────┐
   │ HPA   Horizontal Pod Autoscaler — MORE PODS                  │
   │       Scales on CPU, memory, or custom/external metrics      │
   ├──────────────────────────────────────────────────────────────┤
   │ VPA   Vertical Pod Autoscaler — BIGGER PODS                  │
   │       Adjusts requests/limits. ⚠️ Usually requires a restart; │
   │       ⚠️ conflicts with HPA on the same metric.               │
   ├──────────────────────────────────────────────────────────────┤
   │ CLUSTER AUTOSCALER / KARPENTER — MORE NODES                  │
   │       Adds nodes when pods are Pending due to capacity.      │
   │       ⭐ Karpenter is faster and picks instance types to fit  │
   │         the pending pods, rather than scaling fixed groups.  │
   ├──────────────────────────────────────────────────────────────┤
   │ KEDA  ⭐ Event-driven autoscaling — scales on QUEUE DEPTH,    │
   │       Kafka lag, Redis list length, cron, and 60+ sources.   │
   │       Can scale to ZERO. The right tool for workers.         │
   └──────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
  behavior:                        # ⭐ prevent flapping
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
        - { type: Percent, value: 50, periodSeconds: 60 }
    scaleUp:
      stabilizationWindowSeconds: 0     # ⭐ scale up IMMEDIATELY
      policies:
        - { type: Percent, value: 100, periodSeconds: 30 }
```

```
   ⭐ ASYMMETRIC SCALING IS THE KEY INSIGHT
     Scale UP fast (a few seconds) — being under-provisioned
     costs you availability right now.
     Scale DOWN slowly (several minutes) — being briefly
     over-provisioned only costs a little money, and rapid
     scale-down causes flapping.

   ⚠️ HPA on CPU utilization is a PERCENTAGE OF REQUESTS.
     If requests are set too low, the pod shows 300% and
     scales constantly. Get requests right first.

   ⭐ For queue workers, CPU is the wrong signal entirely.
     Scale on queue depth or consumer lag with KEDA — that's
     the actual measure of whether you're keeping up.
```

---

## 11. Security

```
   ⭐ THE LAYERED CHECKLIST

   ── RBAC ────────────────────────────────────────────────────
   □ Least privilege — Roles over ClusterRoles
   □ ⚠️ NEVER bind cluster-admin to a ServiceAccount
   □ Separate ServiceAccount per workload
   □ automountServiceAccountToken: false unless the pod
     genuinely calls the API server
   □ Audit with: kubectl auth can-i --list --as=system:serviceaccount:...

   ── POD SECURITY ────────────────────────────────────────────
   □ Pod Security Admission: enforce the `restricted` profile
   □ runAsNonRoot: true, runAsUser: 1000
   □ readOnlyRootFilesystem: true (+ emptyDir for writable paths)
   □ allowPrivilegeEscalation: false
   □ capabilities: drop [ALL]
   □ seccompProfile: RuntimeDefault
   □ ⚠️ NEVER privileged: true or hostPID/hostNetwork/hostPath
     unless it's a deliberate infrastructure DaemonSet

   ── NETWORK ─────────────────────────────────────────────────
   □ ⭐ Default-deny NetworkPolicy per namespace, then allow
   □ mTLS between services (service mesh) if the threat model
     warrants it

   ── SUPPLY CHAIN ────────────────────────────────────────────
   □ Image scanning in CI with a real patch SLA
   □ Signed images, verified by an admission policy
   □ Pin by digest, not tag
   □ Admission policies (Kyverno / OPA Gatekeeper) to enforce
     all of the above automatically
```

```yaml
securityContext:                    # pod level
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  seccompProfile: { type: RuntimeDefault }
containers:
  - name: app
    securityContext:                # container level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities: { drop: [ALL] }
```

---

## 12. Debugging Playbook

```bash
# ⭐ START HERE, ALWAYS
kubectl describe pod <pod>          # Events at the bottom explain 90%
kubectl logs <pod> -f
kubectl logs <pod> --previous       # ⭐ logs from the CRASHED container
kubectl logs <pod> -c <container>   # a specific container
kubectl get events --sort-by=.lastTimestamp -A

# Inspect
kubectl get pod <pod> -o yaml
kubectl top pods / kubectl top nodes
kubectl get pods -o wide            # shows NODE and IP

# Get inside
kubectl exec -it <pod> -- sh
kubectl debug <pod> -it --image=nicolaka/netshoot --target=<container>
                                    # ⭐ ephemeral container — works
                                    #   even on distroless images
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash
kubectl port-forward svc/web 8080:80

# ⭐ RBAC troubleshooting
kubectl auth can-i create pods --as=system:serviceaccount:prod:api
kubectl auth can-i --list --as=system:serviceaccount:prod:api
```

```
   ⭐ SYMPTOM → CAUSE TABLE

   ┌─────────────────────┬────────────────────────────────────────┐
   │ Pending             │ Unschedulable. describe → Events:      │
   │                     │ insufficient CPU/memory · no matching  │
   │                     │ node · unbound PVC · taint not         │
   │                     │ tolerated · ⚠️ RWO volume already       │
   │                     │ attached to another node               │
   ├─────────────────────┼────────────────────────────────────────┤
   │ ImagePullBackOff    │ Bad image name/tag · missing           │
   │                     │ imagePullSecret · private registry ·   │
   │                     │ ⚠️ wrong architecture (arm64 on amd64)  │
   ├─────────────────────┼────────────────────────────────────────┤
   │ CrashLoopBackOff    │ ⭐ App exits on startup. ALWAYS check    │
   │                     │ `logs --previous`. Usual causes: bad   │
   │                     │ config, missing env var, unreachable   │
   │                     │ dependency, failed migration, or a     │
   │                     │ liveness probe that's too aggressive   │
   ├─────────────────────┼────────────────────────────────────────┤
   │ OOMKilled (137)     │ Memory limit exceeded. Check whether   │
   │                     │ the runtime respects the cgroup limit  │
   │                     │ (JVM/Node heap sizing) before raising  │
   ├─────────────────────┼────────────────────────────────────────┤
   │ Terminating forever │ Finalizer stuck · a volume that won't  │
   │                     │ detach · the node is gone              │
   ├─────────────────────┼────────────────────────────────────────┤
   │ Service returns     │ ⭐ Selector doesn't match pod labels.    │
   │ nothing             │ CHECK: kubectl get endpoints <svc>     │
   │                     │ If it's empty, the selector is wrong   │
   │                     │ or no pod is READY.                    │
   ├─────────────────────┼────────────────────────────────────────┤
   │ Intermittent 5xx    │ A pod is Ready but broken (gray        │
   │ from a Service      │ failure) · readiness probe too lax ·   │
   │                     │ ⚠️ missing preStop → requests routed to │
   │                     │ terminating pods on every deploy       │
   └─────────────────────┴────────────────────────────────────────┘
```

```
   ⭐ THE SERVICE DEBUGGING SEQUENCE — memorize this order

   1. kubectl get endpoints <svc>
      EMPTY? → the selector doesn't match, or no pod is Ready.
      This single check resolves most "my service doesn't work."

   2. Are the pods actually Ready?  kubectl get pods
   3. Is the app listening on targetPort? (exec in, check ss -tlnp)
   4. Does DNS resolve?  kubectl run tmp --rm -it --image=busybox \
                          -- nslookup web.prod
   5. Is a NetworkPolicy blocking it?
   6. Only then look at kube-proxy / CNI
```

---

## 13. Production Checklist

```
   WORKLOAD
   □ Resource requests AND limits (memory equal; consider no CPU limit)
   □ Liveness (process only) + Readiness (deps) + Startup for slow boots
   □ ⭐ preStop sleep + terminationGracePeriodSeconds > longest request
   □ PodDisruptionBudget
   □ topologySpreadConstraints across zones
   □ ⭐ replicas ≥ 2 (⭐ 3 across zones for real HA)
   □ Image pinned by digest, not :latest
   □ Non-root, read-only filesystem, capabilities dropped

   PLATFORM
   □ Default-deny NetworkPolicy
   □ RBAC least privilege, automountServiceAccountToken false
   □ Pod Security Admission `restricted`
   □ ⭐ etcd encryption at rest + tested BACKUPS
   □ Cluster autoscaler or Karpenter
   □ Metrics, logs, traces with pod/namespace labels
   □ Admission policies (Kyverno/Gatekeeper) enforcing the above
   □ ⭐ Cluster upgrade plan and a tested rollback
```

---

## 14. Interview Section

<details>
<summary><b>Q1. Explain what happens when you run `kubectl apply`.</b></summary>

kubectl sends the manifest to the API server, which authenticates the caller and authorizes via RBAC. Admission controllers then run — mutating ones inject defaults and sidecars, validating ones enforce policy. The object is persisted to etcd, and at that point kubectl returns. Nothing is running yet.

From there it's a chain of independent controllers watching the API. The Deployment controller sees a Deployment with no ReplicaSet and creates one. The ReplicaSet controller sees zero pods against a desired three and creates three Pod objects — still unscheduled, with no node assigned. The scheduler notices pods without a nodeName, filters nodes for feasibility, scores the feasible ones, and binds each pod. Finally the kubelet on each node sees a pod assigned to it, pulls images, and starts containers, reporting status back up.

The important part is that no component calls another directly. Each watches the API server and reconciles. That's why the system self-heals — any controller can restart and resume purely from observed state, and why deleting a pod immediately recreates it.
</details>

<details>
<summary><b>Q2. Requests vs limits — how do you set them?</b></summary>

Requests are what the scheduler uses to place a pod and represent a guaranteed reservation. Limits are the hard cgroup ceiling.

The behaviours differ crucially by resource. Exceeding a memory limit gets you OOM-killed with exit code 137. Exceeding a CPU limit gets you throttled, which shows up as latency spikes with no errors and no obvious cause.

My guidance is to always set memory requests and limits, and set them equal. That gives Guaranteed QoS, so you're evicted last under node pressure, and the behaviour is predictable.

For CPU, I'd often argue for setting a request but no limit. CPU is compressible, and the scheduler already prevents oversubscription using requests. Adding a limit only introduces throttling. It's counterintuitive but widely recommended, and the exception is multi-tenant clusters where you need to protect against noisy neighbours.

The trap people hit is monitoring CPU utilization instead of throttling. A pod averaging 40% CPU can be badly throttled, because quota is enforced per 100-millisecond period — a GC pause consumes the whole quota and the app stalls for the remainder. You have to watch `container_cpu_cfs_throttled_seconds_total`.
</details>

<details>
<summary><b>Q3. Liveness vs readiness, and what goes wrong?</b></summary>

Liveness answers "is this process broken" — failing it restarts the container. Readiness answers "can I serve traffic right now" — failing it removes the pod from Service endpoints without restarting.

The dangerous mistake is checking dependencies in the liveness probe. If your liveness endpoint queries the database and the database has a brief blip, every pod fails liveness simultaneously and Kubernetes restarts your entire fleet. You've converted a recoverable database hiccup into a full outage with a cold-start stampede on top.

So liveness should check only that the process itself is functional. Readiness is where dependency checks belong, because failing it just stops traffic temporarily.

Startup probes solve a related problem: a slow-starting JVM can be killed by liveness before it finishes booting. A startup probe disables the other two until the app is up, letting you allow several minutes for startup without weakening liveness afterward.
</details>

<details>
<summary><b>Q4. How do you achieve zero-downtime deployments?</b></summary>

There's a race most people miss. When a pod is deleted, SIGTERM and endpoint removal happen in parallel — and endpoint removal propagates slowly, because kube-proxy on every node has to update iptables. So traffic keeps arriving at a pod that's already shutting down.

The fix has several parts. A preStop hook that sleeps around fifteen seconds, so the pod keeps serving normally while routing catches up. A terminationGracePeriodSeconds longer than the preStop plus your longest request. The application handling SIGTERM properly: flip readiness false, stop accepting new connections, drain in-flight requests, close connections, exit zero.

And the container must actually receive SIGTERM, which means exec-form CMD and an init process as PID 1 — otherwise the shell swallows the signal and Kubernetes SIGKILLs you mid-request after the grace period.

On the rollout side, maxUnavailable zero with maxSurge one means you never dip below desired capacity. Plus a PodDisruptionBudget so cluster upgrades and node drains can't evict everything at once — that's a very common cause of self-inflicted outages during maintenance.
</details>

<details>
<summary><b>Q5. A pod is stuck in CrashLoopBackOff. Walk me through it.</b></summary>

First, `kubectl logs --previous`, because the current container may not have started yet and the crashed one has the actual error. That alone resolves most cases.

Then `kubectl describe pod` and read the Events section, which shows exit codes, image pull problems, and probe failures.

The exit code narrows it fast. 137 is OOM-killed, so I'd check whether the runtime respects the cgroup limit before assuming I need more memory — a JVM or Node process sizing its heap from host memory will die immediately regardless of the limit. 127 means command not found, usually a typo or a binary missing from a minimal base image. A generic 1 means the application itself failed on startup.

Common causes beyond those: a missing environment variable or ConfigMap key, a dependency that isn't reachable yet, a failed database migration on boot, or a liveness probe firing before the app finishes starting — which a startup probe fixes.

If logs are empty, I'd override the entrypoint to get a shell and run the command manually, or use `kubectl debug` with an ephemeral container to inspect the environment.
</details>

<details>
<summary><b>Q6. My Service isn't routing traffic. How do you debug it?</b></summary>

`kubectl get endpoints <service>` first. If it's empty, that's the answer — either the Service selector doesn't match the pod labels, or no pod is passing its readiness probe. Those two account for the large majority of cases.

If endpoints exist, I'd check that the pods are actually Ready, then exec into one and confirm the app is listening on the targetPort and on all interfaces rather than 127.0.0.1 — binding to localhost inside a container is a classic mistake.

Then DNS: run a temporary busybox pod and nslookup the service name. That distinguishes a DNS problem from a routing problem.

Then NetworkPolicy — if a default-deny policy is in place and there's no matching allow rule, connections fail silently with a timeout rather than a refusal.

Only after all of that would I look at kube-proxy or the CNI, because those are rarely the cause and much harder to investigate.

The mental model that makes this fast is remembering the Service IP is virtual — nothing listens on it, it's purely an iptables rule pointing at the EndpointSlice. So the EndpointSlice being wrong or empty is where problems concentrate.
</details>

<details>
<summary><b>Q7. Deployment vs StatefulSet?</b></summary>

Deployment for stateless workloads. Pods are interchangeable, get random names, can start and stop in any order, and typically share nothing.

StatefulSet when identity matters. Pods get ordinal names — db-0, db-1 — with stable DNS entries, each gets its own persistent volume via volumeClaimTemplates, and they start in order and scale down in reverse.

That ordering matters for clustered systems where the first node needs to become primary before replicas join, and the stable identity matters because a database replica needs to reattach to *its* data, not just any volume.

The trap people hit is using a Deployment with a ReadWriteOnce persistent volume and multiple replicas. Only one node can attach an RWO volume, so the other pods sit Pending forever. That's what volumeClaimTemplates in a StatefulSet solves.

The important caveat is that a StatefulSet doesn't make your application distributed. It provides stable identity and storage; the clustering, replication, and failover logic remain your application's responsibility. Which is why operators exist for complex stateful systems.
</details>

<details>
<summary><b>Q8. Are Kubernetes Secrets actually secret?</b></summary>

Not by default. They're base64-encoded, which is encoding, not encryption. Anyone who can read the Secret object — or read etcd directly — gets the plaintext with one command.

To make them genuinely protected: enable encryption at rest for etcd via EncryptionConfiguration, ideally with a KMS provider rather than a local key. Lock down RBAC so very few principals can read secrets, and turn off automountServiceAccountToken for pods that don't call the API.

Better still is not storing long-lived secrets in the cluster at all. External Secrets Operator syncs from Vault or a cloud secrets manager. And workload identity — IRSA on EKS, Workload Identity on GKE — issues short-lived credentials automatically, so there's no static secret to steal.

If you must commit secrets to git, Sealed Secrets or SOPS encrypt them so only the cluster can decrypt.

I'd also prefer mounting secrets as files over environment variables, because environment variables leak through /proc, child processes, crash dumps, and error reporting tools.
</details>

---

## 15. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                     KUBERNETES — ONE PAGE                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ EVERYTHING IS A CONTROL LOOP: desired vs actual → act → repeat     ║
║   nothing calls anything; controllers WATCH the API server           ║
║   etcd is the only stateful part → back it up, keep it fast          ║
╠══════════════════════════════════════════════════════════════════════╣
║ POD = shared network + IPC namespace. Same IP, localhost between     ║
║   containers. Pause container holds the namespaces open.             ║
║ Deployment(stateless) · StatefulSet(identity+own PVC+ordered) ·      ║
║ DaemonSet(one per node) · Job/CronJob                                ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ RESOURCES                                                         ║
║   memory: requests == limits (Guaranteed QoS, evicted last)          ║
║   CPU: set requests, ⭐ consider NO limit (limits only THROTTLE)      ║
║   memory over → OOMKilled 137 · CPU over → throttled (latency spikes)║
║   monitor container_cpu_cfs_throttled_seconds_total, not just %CPU   ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ LIVENESS MUST NOT CHECK DEPENDENCIES — a DB blip would restart     ║
║   your whole fleet. Deps go in READINESS. Startup probe for slow boot║
║ ⭐ ZERO DOWNTIME: preStop sleep 15 + grace > longest request +        ║
║   app handles SIGTERM (readiness false FIRST) + PDB + maxUnavailable 0║
╠══════════════════════════════════════════════════════════════════════╣
║ NETWORK: every pod gets a real IP, NO port mapping                   ║
║   Service IP is VIRTUAL (iptables only — you cannot ping it)         ║
║   ⭐ DEBUG SEQUENCE: kubectl get endpoints FIRST. Empty = selector    ║
║     mismatch or no Ready pod. That's most "service doesn't work."    ║
║   ⚠️ default: ALL pods can reach ALL pods → default-deny NetworkPolicy║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ SECRETS ARE BASE64, NOT ENCRYPTED                                  ║
║   → etcd encryption at rest + RBAC + External Secrets / workload ID  ║
║ ⚠️ ConfigMap changes DON'T restart pods → hash config into the        ║
║   pod template annotation to force a rollout                         ║
║ ⚠️ RWO volume + Deployment replicas>1 = pods stuck Pending forever    ║
║   → StatefulSet with volumeClaimTemplates                            ║
║ ⚠️ reclaimPolicy: Delete destroys data with the PVC → use Retain      ║
╠══════════════════════════════════════════════════════════════════════╣
║ DEBUG: describe pod (Events!) → logs --previous → exit code →        ║
║   kubectl debug (ephemeral container, works on distroless)           ║
║ Pending=unschedulable · 137=OOM · 127=not found ·                    ║
║ CrashLoop→logs --previous · no traffic→get endpoints                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [CI/CD →](cicd.md) · **Related:** [Docker](docker.md) · [Linux & Networking](linux-networking.md) · [Observability & SRE](observability-sre.md)
