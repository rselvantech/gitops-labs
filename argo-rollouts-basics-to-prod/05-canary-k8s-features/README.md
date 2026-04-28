# Demo-05: Canary + Kubernetes Features

## Overview

Demo-03 introduced the minimum viable canary: a Rollout with `setWeight` and
`pause`. That is enough to understand the progressive delivery lifecycle, but it is
not production-grade. Real workloads need probes that handle slow starts, sidecars
that run alongside the main container, budgets that protect pods during node drains,
spread constraints that keep the canary distributed, and clean shutdown hooks that
prevent dropped requests during scale-down.

This demo enriches the same canary with seven Kubernetes features — each added one
concept at a time. The final `rollout.yaml` combines them all into a single manifest
that could serve as a production starting point.

The images, the strategy steps, the namespace, and the Docker Hub secret are all
unchanged from Demo-03 — this demo is purely about enriching the pod spec.

**What you'll learn:**
- What `startupProbe` is and why it exists alongside `livenessProbe` and `readinessProbe`
- How native sidecar containers work in Kubernetes 1.29+ and how they differ from
  regular multi-container pods
- What a `PodDisruptionBudget` protects against and how to size `minAvailable` correctly
- How `topologySpreadConstraints` distributes pods across nodes during a canary
- How `preStop` + `terminationGracePeriodSeconds` protect in-flight requests during scale-down
- How `minReadySeconds` affects the `Available` replica counter during a rollout
- What `revisionHistoryLimit` controls and why the default of 10 matters
- How each of these features is visible (or invisible) in the CLI dashboard and web UI

**What you'll do:**
- Create the `05-canary-k8s-features/` subdirectory in `argo-rollouts-config`
- Write and apply a broken Rollout with no strategy field — observe the error
- Incrementally understand each feature through the concept sections
- Write the final `rollout.yaml` with all seven features combined
- Write `service.yaml` (same pattern as Demo-03)
- Write `pdb.yaml` as a separate Kubernetes object
- Apply all three manifests and verify each feature is active in the cluster
- Trigger a canary update, observe how `minReadySeconds` delays `Available`,
  how the PDB `ALLOWED DISRUPTIONS` changes during the surge, and promote to completion
- Walk through the web UI for all three states

---

## Prerequisites

- ✅ Completed Demo-03 — `argo-rollouts-demo` namespace exists, `dockerhub-secret` exists
- ✅ `rselvantech/podinfo:red` and `rselvantech/podinfo:blue` pushed to Docker Hub
- ✅ Argo Rollouts controller running in `argo-rollouts` namespace
- ✅ `kubectl argo rollouts` plugin installed
- ✅ Argo Rollouts v1.7+ (required for native sidecar support — see Concepts section 2)

**Verify:**
```bash
# Controller is running
kubectl get pods -n argo-rollouts
# Expected: argo-rollouts-* pods Running

# Rollouts version — must be v1.7+ for native sidecar display
kubectl argo rollouts version
# Expected: v1.7.x or higher

# Namespace exists
kubectl get namespace argo-rollouts-demo
# Expected: argo-rollouts-demo   Active

# Docker Hub pull secret exists
kubectl get secret dockerhub-secret -n argo-rollouts-demo
# Expected: dockerhub-secret   kubernetes.io/dockerconfigjson   1

# Images are available
docker manifest inspect rselvantech/podinfo:red  | grep -m1 mediaType
docker manifest inspect rselvantech/podinfo:blue | grep -m1 mediaType
# Expected: both return a mediaType line — images exist on Docker Hub

# No leftover Rollout from a previous demo
kubectl argo rollouts list rollouts -n argo-rollouts-demo
# Expected: No resources found. (or clean it up before continuing)
```

> **Cleanup from Demo-04:** If Demo-04 left a Rollout running in `argo-rollouts-demo`,
> delete it before starting:
> ```bash
> kubectl delete rollout simple-color-app -n argo-rollouts-demo
> kubectl delete service simple-color-app-active simple-color-app-preview -n argo-rollouts-demo 2>/dev/null || true
> ```

---

## Concepts

### 1 · `startupProbe` — A Dedicated Startup Phase Probe

#### Why it exists

Kubernetes has two steady-state probes: `livenessProbe` and `readinessProbe`.
`livenessProbe` restarts a container if it stops responding. `readinessProbe`
removes a pod from Service endpoints if it is not ready to serve traffic.

The problem: both probes begin firing from the moment the container starts. Some
applications are legitimately slow to initialise — they might load a large model,
run a database migration, or warm a cache. If `livenessProbe` fires during that
window, it kills the container before it ever finishes starting. The result is a
crash loop — not because the app is broken, but because the probe fired too early.

The traditional workaround was to set a large `initialDelaySeconds` on the liveness
probe. That trades one problem for another: a genuinely crashed container now takes
the full `initialDelaySeconds` before Kubernetes notices and restarts it.

#### What it does

`startupProbe` is a third probe that acts as a **startup gate**. While the startup
probe is configured and has not yet succeeded, Kubernetes suspends both
`livenessProbe` and `readinessProbe` entirely. Only after the startup probe
succeeds once do the other two probes activate and take over.

The startup probe itself runs on its own `failureThreshold × periodSeconds` budget.
If the container does not pass the startup probe within that window, Kubernetes
restarts it. After the startup probe passes once, it never runs again for the
lifetime of that container instance.

```
Container starts
  │
  ├── startupProbe fires every periodSeconds
  │     livenessProbe: SUSPENDED
  │     readinessProbe: SUSPENDED
  │
  ├── startupProbe succeeds once
  │     → livenessProbe: ACTIVATED (fires every livenessProbe.periodSeconds)
  │     → readinessProbe: ACTIVATED (fires every readinessProbe.periodSeconds)
  │     → startupProbe: DONE (never fires again)
  │
  └── normal steady-state operation
```

#### When to use it

Use `startupProbe` whenever your container has a startup time that is longer than
what your liveness probe can tolerate. A value of `failureThreshold × periodSeconds`
that equals your maximum expected startup time is the right sizing:

```
failureThreshold: 10
periodSeconds: 3
→ allows up to 30 seconds for startup
→ after 30s without passing, restart the container
```

For applications with consistent short startups (< 5s), `initialDelaySeconds` on
the liveness probe is sufficient. Once startup time is variable or potentially
long, `startupProbe` is cleaner and more precise.

#### The three-probe picture

| Probe | Phase | Failure action | Suspends others? |
|---|---|---|---|
| `startupProbe` | Startup only — fires until first success | Restart container | Yes — liveness + readiness paused while startup runs |
| `livenessProbe` | Running — fires every `periodSeconds` forever | Restart container | No |
| `readinessProbe` | Running — fires every `periodSeconds` forever | Remove from Service endpoints | No |

podinfo exposes `/healthz` (liveness, health check) and `/readyz` (readiness, ready
to serve). For the startup probe we use `/healthz` — the same endpoint as liveness,
just fired in a different phase.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 9898
  failureThreshold: 10   # 10 × 3s = 30s maximum startup window
  periodSeconds: 3
```

---

### 2 · Native Sidecar Container — `initContainers` with `restartPolicy: Always`

#### Why it exists

A sidecar is a helper process that runs alongside the main container — a log
shipper, a metrics collector, a proxy agent. Before Kubernetes 1.29, sidecars were
declared as regular containers in `spec.containers`. This worked, but it had two
problems:

1. **No guaranteed startup order.** All `spec.containers` start in parallel.
   The sidecar might not be ready when the main container tries to use it.
2. **Blocks Job completion.** In a batch Job, the sidecar keeps running after
   the main container finishes because it does not know the job is done —
   the Job hangs waiting for all containers to exit.

Kubernetes 1.29 (beta, enabled by default; stable in 1.33) introduced a native
sidecar declaration: an entry in `initContainers` with `restartPolicy: Always`.
Because init containers run in declared order before the main container, the
sidecar is guaranteed to start first. Because it has `restartPolicy: Always` at
the container level, it continues running for the lifetime of the Pod — unlike a
regular init container which must exit before the next step begins.

#### What it does

A container in `initContainers` with `restartPolicy: Always` behaves as follows:

```
Pod starts
  │
  ├── initContainers executed in order
  │     ├── timestamp-writer (restartPolicy: Always) ← starts, does NOT exit
  │     │     → runs for lifetime of Pod
  │     │     → next step begins once its startupProbe succeeds (or immediately if none)
  │     └── (any other regular init containers — must exit 0 before proceeding)
  │
  ├── spec.containers start
  │     └── podinfo ← main container
  │
  ├── If timestamp-writer crashes:
  │     → restarted independently (exponential backoff: 10s, 20s, 40s… capped at 5m)
  │     → podinfo is NOT restarted
  │
  └── When podinfo exits:
        → Pod terminates
        → timestamp-writer is stopped by Kubernetes
        → Job sees Pod as complete — sidecar does NOT block job completion
```

**Key differences from a regular second container in `spec.containers`:**

| Aspect | Regular multi-container (`spec.containers`) | Native sidecar (`initContainers` + `restartPolicy: Always`) |
|---|---|---|
| Startup order | Parallel — no guarantee | Sidecar starts before main container |
| Lifecycle | Both run until Pod terminates | Sidecar stopped after main container exits |
| Job completion | Sidecar blocks Job completion | Does not block Job completion |
| Resource accounting | Resources summed with main containers | Resources added to main container sum (same for scheduling) |
| Restart | Follows Pod-level `restartPolicy` | Always restarts regardless of Pod `restartPolicy` |

#### When to use it

Use native sidecars when:
- The helper must be running before the main container starts (proxy agents, secrets fetchers, network proxies)
- You are running Jobs and the sidecar should not block completion
- You want the sidecar to restart independently without affecting the main container

The old pattern — multiple entries in `spec.containers` — still works. Use native
sidecars for new workloads on Kubernetes 1.29+. On older clusters, keep the
original multi-container approach.

#### Argo Rollouts compatibility

Argo Rollouts versions before v1.7 had a bug where native sidecar containers caused
pods to appear stuck on `Init` in the CLI dashboard pod status display (GitHub issue
#3366). The controller itself managed the Rollout correctly — only the status display
was wrong. The fix was merged and cherry-picked to the v1.7 release branch in
July 2024 (PR #3639).

The current Helm chart (2.40.9+, Argo Rollouts v1.9.0) is well past v1.7. Verify:
```bash
kubectl argo rollouts version
# Must show v1.7.0 or later
```

With v1.7+, pods with a native sidecar correctly show `ready:2/2` — two containers
ready: the sidecar and the main container. This is expected and correct.

#### Demo sidecar

The demo sidecar is a `busybox` container that writes timestamps to a shared
`emptyDir` volume every second. The main podinfo container mounts the same volume.
This is the simplest possible illustration of the shared-volume sidecar pattern.

```yaml
initContainers:
  - name: timestamp-writer
    image: busybox:1.36
    restartPolicy: Always          # ← this single line makes it a native sidecar
    command:
      - /bin/sh
      - -c
      - |
        while true; do
          echo "$(date -Iseconds) sidecar heartbeat" >> /shared/timestamps.log
          sleep 1
        done
    volumeMounts:
      - name: shared-logs
        mountPath: /shared
    resources:
      requests:
        cpu: 10m
        memory: 16Mi
      limits:
        cpu: 50m
        memory: 32Mi
```

---

### 3 · `PodDisruptionBudget` — Protecting Pods During Voluntary Evictions

#### Why it exists

A node drain, cluster upgrade, autoscaler scale-in, or Spot instance eviction can
remove all pods from a node at once. Without any constraint, Kubernetes is free to
evict every single pod simultaneously — taking the service completely down during
what should be a routine maintenance operation.

> **Compare with ArgoCD:** ArgoCD has no equivalent concept. ArgoCD manages
> Application sync state — not individual pod availability during disruptions.
> PDB is a plain Kubernetes resource; both ArgoCD and Argo Rollouts respect it
> automatically.

#### What it does

A `PodDisruptionBudget` tells the Kubernetes Eviction API the minimum number (or
percentage) of pods that must remain available during **voluntary disruptions**.
Voluntary disruptions are cluster-driven eviction events: `kubectl drain`, cluster
upgrades, autoscaler scale-downs, Spot instance preemptions. Involuntary disruptions
(node hardware failure, OOM kill, kernel panic) are not covered by PDB.

The Eviction API honours the PDB by refusing eviction requests that would drop
below the minimum. Tools that call the Eviction API — `kubectl drain`, cluster
upgrade controllers — respect PDBs automatically. Direct `kubectl delete pod` does
not go through the Eviction API and bypasses PDB.

```
Without PDB:
  Node drain → all 5 pods evicted simultaneously → service down

With PDB (minAvailable: 4):
  Node drain → evict pod 1 → 4 remaining → PDB: OK
             → evict pod 2 → 3 remaining → PDB: BLOCKED
             → wait for new pod to start elsewhere
             → pod started → 4 available again
             → evict pod 2 → 3 remaining → PDB: BLOCKED... (cycle continues)
  → drain completes, but only one pod evicted at a time
```

Two mutually exclusive fields — pick one:

| Field | Meaning | Best for |
|---|---|---|
| `minAvailable` | At least N pods must be available after eviction | Non-built-in controllers (Rollout is not a Deployment or StatefulSet) — integer values are safest |
| `maxUnavailable` | At most N pods may be unavailable after eviction | Built-in controllers (Deployment, StatefulSet) — adjusts automatically when replicas change |

For a Rollout (not a built-in controller type), use `minAvailable` with an integer.
The `maxUnavailable` field requires a supported owning resource type when used with
percentages, and behaves inconsistently with non-built-in controllers.

> **Critical constraint:** Never set `minAvailable` equal to `spec.replicas`. That
> means zero pods may ever be evicted — `kubectl drain` will hang indefinitely
> waiting for a pod slot that never opens. With `replicas: 5` and `minAvailable: 4`,
> exactly one pod may be evicted at a time. With `minAvailable: 5`, drain is
> permanently blocked.

#### How it works — a separate object

PDB is a separate Kubernetes resource. The Rollout controller does not create it.
You apply it once alongside the Rollout. It selects pods by label, independent of
what manages those pods.

**`pdb.yaml`:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: simple-color-app-pdb
  namespace: argo-rollouts-demo
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: simple-color-app
```

The `ALLOWED DISRUPTIONS` field in `kubectl get pdb` is live — it updates as pod
count changes during the canary surge. With 5 pods and `minAvailable: 4`:
`ALLOWED DISRUPTIONS: 1`. With 6 pods (5 stable + 1 canary surge):
`ALLOWED DISRUPTIONS: 2`.

---

### 4 · `topologySpreadConstraints` — Even Distribution During the Canary

#### Why it exists

Without spread constraints, the scheduler places pods wherever node capacity is
available. In a three-zone cluster, you could end up with all five canary pods on
nodes in zone-a and none in zone-b. If zone-a experiences a disruption during the
canary, the entire 20% canary weight concentrates in a degraded zone — giving you a
false signal about the canary's health.

#### What it does

`topologySpreadConstraints` instructs the scheduler to spread pods across a topology
domain so that no single domain is overloaded relative to others. The `maxSkew`
field is the maximum allowed pod count difference between the most-loaded and
least-loaded domain.

```
3 zones: a, b, c — 6 pods, maxSkew: 1

Allowed spread:   2 / 2 / 2  (difference: 0 — within maxSkew 1)
                  2 / 2 / 1  (difference: 1 — within maxSkew 1)
Not allowed:      3 / 2 / 1  (difference: 2 — violates maxSkew 1)
```

Key fields:

| Field | Meaning |
|---|---|
| `maxSkew` | Maximum allowed count difference between most and least loaded domain |
| `topologyKey` | Node label defining the domain — `topology.kubernetes.io/zone` for zones, `kubernetes.io/hostname` for individual nodes |
| `whenUnsatisfiable` | `DoNotSchedule` — hard constraint, pod stays Pending if unsatisfied. `ScheduleAnyway` — soft constraint, scheduler tries but proceeds anyway |
| `labelSelector` | Which pods this constraint applies to |

#### When to use it

Use zone-level spread (`topology.kubernetes.io/zone`) whenever availability across
failure domains matters — which is almost always for any stateless HTTP service.
Use node-level spread (`kubernetes.io/hostname`) to prevent hot-spotting on a single
node, similar to `podAntiAffinity` but more flexible.

In a Minikube single-node cluster, `topologySpreadConstraints` is trivially
satisfied — all pods are on the one node, skew is always 0. Set `whenUnsatisfiable:
ScheduleAnyway` so the single-node demo does not block scheduling. In production
multi-zone clusters, switch to `DoNotSchedule` for a hard guarantee.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway   # soft — works on single-node Minikube
    labelSelector:
      matchLabels:
        app: simple-color-app
```

---

### 5 · `preStop` Hook + `terminationGracePeriodSeconds` — Clean Shutdown

#### Why it exists

When the Rollout controller scales down the stable ReplicaSet during promotion (or
the canary ReplicaSet during abort), it triggers pod termination. Kubernetes sends
`SIGTERM` to the container. The problem is timing:

1. Kubernetes removes the pod from the Service `Endpoints` list at the same moment
   it sends `SIGTERM`. But `kube-proxy` propagates endpoint changes asynchronously.
   For a brief window (typically a few hundred milliseconds to a few seconds), traffic
   is still being routed to a pod that has already started shutting down.

2. If the application exits the instant it receives `SIGTERM`, any in-flight requests
   are dropped with a connection reset — visible to the user as a 503 or socket error.

#### What it does

`preStop` is a lifecycle hook that runs *before* the container receives `SIGTERM`.
It can run a shell command or send an HTTP request. The most common pattern is a
`sleep` command that gives the endpoint propagation time to complete before the
container begins handling the SIGTERM:

```
Pod termination sequence:

  1. Pod marked for deletion
  2. Pod removed from Service Endpoints list
  3. preStop hook executes (sleep 5)
     ← during this 5s, kube-proxy propagates the endpoint removal
     ← new traffic stops arriving at this pod
  4. SIGTERM sent to container's main process
  5. Application handles SIGTERM gracefully (drains in-flight requests)
  6. Container exits cleanly
  7. SIGKILL fires if terminationGracePeriodSeconds is exceeded
```

`terminationGracePeriodSeconds` (default: 30s) is the total time budget for the
entire sequence: preStop execution + application graceful shutdown. Both must fit
within this window. If the container is still running when the budget expires,
Kubernetes sends SIGKILL.

```yaml
# Inside spec.template.spec.containers[].lifecycle:
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]

# Inside spec.template.spec — NOT inside containers[]:
terminationGracePeriodSeconds: 30   # preStop (5s) + app shutdown must fit here
```

> **`terminationGracePeriodSeconds` is pod spec level**, not container level.
> It is a sibling of `containers`, not nested inside a container definition.
> A common mistake is placing it inside the container block — it is silently ignored
> there.

#### When to use it

Use `preStop` sleep for any HTTP service. The typical sleep value is 5–15 seconds.
podinfo handles SIGTERM gracefully with its own built-in shutdown hook, so the
`preStop` sleep covers the endpoint propagation gap and the app shutdown covers
the rest of the 30s budget.

---

### 6 · `minReadySeconds` — Requiring Sustained Readiness

#### Why it exists

A pod becomes `Ready` the instant all readiness probes pass. With `minReadySeconds:
0` (the default), a pod is immediately counted as `Available` the moment it becomes
Ready. The problem: a flaky pod might pass the readiness probe on first check, be
counted as Available, then fail on the very next check. The Rollout controller
could advance a step based on an Available count that immediately collapses.

#### What it does

`minReadySeconds` at the Rollout spec level (default: **0**) requires a pod to
remain continuously Ready for the specified number of seconds before it counts as
`Available`. Only `Available` pods count toward the Rollout controller's decision
to proceed to the next step.

```
Pod becomes Ready
  │
  ├── minReadySeconds: 0 (default)
  │     → immediately counted as Available
  │     → Rollout controller may advance a step
  │
  └── minReadySeconds: 10
        → pod must sustain readiness for 10 consecutive seconds
        → only then counted as Available
        → single readiness failure during those 10s resets the clock
```

This delay is visible in the CLI dashboard during a canary: `Ready: 6` while
`Available: 5` — the sixth (new canary) pod has passed readiness but has not yet
sustained it for 10 seconds.

> **ArgoCD comparison:** Kubernetes Deployment has the same `minReadySeconds` field
> at the same spec level. The Rollout spec mirrors it intentionally. In ArgoCD
> managed Deployments, `minReadySeconds` works the same way.

#### When to use it

Set `minReadySeconds` to a value slightly longer than your readiness probe
`periodSeconds` — enough for the pod to pass at least one full probe cycle before
counting. A value of 10–30 seconds is common for HTTP services.

```yaml
# spec level in the Rollout — not inside template
minReadySeconds: 10
```

---

### 7 · `revisionHistoryLimit` — Controlling Old ReplicaSet Retention

#### Why it exists

Every Rollout update creates a new ReplicaSet. Promoted ReplicaSets are scaled to
zero but not deleted — they are kept as rollback targets. Over many deployments,
dozens of zero-scale ReplicaSets accumulate. Each consumes a small amount of API
server memory and etcd storage. In large clusters with many Rollouts and frequent
deployments, this adds up.

#### What it does

`revisionHistoryLimit` at the Rollout spec level (default: **10**) is the number of
old ReplicaSets to retain after they are scaled to zero. The currently active stable
ReplicaSet and the currently active canary ReplicaSet do not count toward this limit
— only retired ones do.

```
After 8 updates (8 revisions total), revisionHistoryLimit: 3:
  revision:8 → stable (active, not counted)
  revision:7 → ScaledDown (counted: 1)
  revision:6 → ScaledDown (counted: 2)
  revision:5 → ScaledDown (counted: 3)
  revision:4 → DELETED (limit reached)
  revision:3 → DELETED
  revision:2 → DELETED
  revision:1 → DELETED
```

> **ArgoCD comparison:** In GitOps with ArgoCD, rollback means reverting the Git
> commit and letting ArgoCD re-sync. The history lives in Git — not in ReplicaSets.
> In standalone Argo Rollouts mode (these demos), history lives in ReplicaSets.
> `revisionHistoryLimit` matters here in a way it does not matter in the ArgoCD
> integration demos (proj-02) where you roll back by reverting a Git commit.

#### When to use it

The default of 10 is fine for most cases. If you deploy many times per day and
want to limit etcd bloat, 3 is a practical minimum that still gives you meaningful
rollback options. Setting it to 0 is valid but leaves no rollback targets at all.

```yaml
revisionHistoryLimit: 3
```

---

## Folder Structure

```
05-canary-k8s-features/
├── README.md                                     ← This file
└── src/
    └── argo-rollouts-config/                     ← git remote: rselvantech/argo-rollouts-config
        └── 05-canary-k8s-features/
            ├── rollout.yaml                      ← Rollout with all 7 K8s features
            ├── service.yaml                      ← ClusterIP Service (same pattern as Demo-03)
            └── pdb.yaml                          ← PodDisruptionBudget (separate K8s object)
```

> **No `podinfo/` subdirectory.** Demo-05 uses the same `rselvantech/podinfo:red`
> and `rselvantech/podinfo:blue` images built in Demo-03. No Dockerfile changes are
> needed — both images are already on Docker Hub.

> **`gitops-labs` vs `argo-rollouts-config`:** `gitops-labs` is your documentation
> and learning repo — READMEs only, never applied to the cluster.
> `argo-rollouts-config` is the working manifests repo — its contents are applied
> with `kubectl apply`. The manifests for this demo live in
> `src/argo-rollouts-config/05-canary-k8s-features/` inside `gitops-labs` but are
> committed to the `argo-rollouts-config` remote, following the same pattern
> established in Demo-03.

---

## Step 1: Set Up the Sub-directory

```bash
# Navigate to the argo-rollouts-config working directory inside gitops-labs
cd argo-rollouts-basics-to-prod/05-canary-k8s-features/src/argo-rollouts-config

# Confirm the remote is already pointing to argo-rollouts-config
git remote -v
# Expected: origin  https://rselvantech:...@github.com/rselvantech/argo-rollouts-config.git

# Create the demo subdirectory
mkdir -p 05-canary-k8s-features
```

---

## Step 2: Write and Apply the Broken Rollout

Before writing the final manifest, apply an intentionally incomplete Rollout —
missing the `strategy` field — to observe the error. This is the same pattern
established in Demo-03.

**Create `05-canary-k8s-features/rollout.yaml` — broken version:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: simple-color-app
  namespace: argo-rollouts-demo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: simple-color-app
  template:
    metadata:
      labels:
        app: simple-color-app
    spec:
      imagePullSecrets:
        - name: dockerhub-secret
      containers:
        - name: podinfo
          image: rselvantech/podinfo:red
```

**Apply:**
```bash
kubectl apply -f 05-canary-k8s-features/rollout.yaml
```

**Expected output from apply:**
```
rollout.argoproj.io/simple-color-app created
```

**Check status:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
```

**Expected:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ✖ Degraded
Message:         InvalidSpec: Rollout has missing field '.spec.strategy'
```

`kubectl apply` succeeds — the API server validates structure, not semantics. The
Argo Rollouts controller catches the semantic error immediately and marks the Rollout
Degraded. Same behaviour as Demo-03.

**Delete before proceeding:**
```bash
kubectl delete rollout simple-color-app -n argo-rollouts-demo
```

---

## Step 3: Write the Final Rollout Manifest

Now write the complete manifest with all seven features combined.

**Update `05-canary-k8s-features/rollout.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: simple-color-app
  namespace: argo-rollouts-demo
spec:
  replicas: 5
  minReadySeconds: 10          # pod must sustain readiness for 10s before counting as Available
  revisionHistoryLimit: 3      # keep 3 old ReplicaSets; Rollout default is 10
  selector:
    matchLabels:
      app: simple-color-app
  template:
    metadata:
      labels:
        app: simple-color-app
    spec:
      terminationGracePeriodSeconds: 30   # total budget: preStop (5s) + app shutdown
      imagePullSecrets:
        - name: dockerhub-secret

      # Native sidecar: restartPolicy: Always on an initContainer makes it
      # a native sidecar (Kubernetes 1.29+). It starts before the main
      # container and runs for the lifetime of the Pod. Requires Argo
      # Rollouts v1.7+ for correct pod-status display (PR #3639).
      initContainers:
        - name: timestamp-writer
          image: busybox:1.36
          restartPolicy: Always
          command:
            - /bin/sh
            - -c
            - |
              while true; do
                echo "$(date -Iseconds) sidecar heartbeat" >> /shared/timestamps.log
                sleep 1
              done
          volumeMounts:
            - name: shared-logs
              mountPath: /shared
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 50m
              memory: 32Mi

      containers:
        - name: podinfo
          image: rselvantech/podinfo:red
          imagePullPolicy: Always

          ports:
            - containerPort: 9898
              protocol: TCP

          # Volume mount to read sidecar-written timestamps
          volumeMounts:
            - name: shared-logs
              mountPath: /shared

          # startupProbe suspends liveness and readiness until it succeeds once.
          # 10 failures × 3s period = 30s maximum startup window.
          startupProbe:
            httpGet:
              path: /healthz
              port: 9898
            failureThreshold: 10
            periodSeconds: 3

          readinessProbe:
            httpGet:
              path: /readyz
              port: 9898
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /healthz
              port: 9898
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          # preStop sleep covers the endpoint propagation window before SIGTERM.
          # Must fit within terminationGracePeriodSeconds along with app shutdown.
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi

      # Soft topology spread across nodes.
      # ScheduleAnyway works on single-node Minikube — switch to DoNotSchedule
      # in production multi-zone clusters for a hard guarantee.
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: simple-color-app

      volumes:
        - name: shared-logs
          emptyDir: {}

  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {}
```

---

## Step 4: Write the Service Manifest

Same single-Service pattern as Demo-03 — one Service, selector matches all pods
regardless of which ReplicaSet they belong to.

**Create `05-canary-k8s-features/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-color-app
  namespace: argo-rollouts-demo
spec:
  selector:
    app: simple-color-app
  ports:
    - name: http
      port: 80
      targetPort: 9898
      protocol: TCP
  type: ClusterIP
```

---

## Step 5: Write the PodDisruptionBudget

PDB is a separate Kubernetes object — not part of the Rollout spec. You apply it
independently. The Rollout controller neither creates nor manages it.

**Create `05-canary-k8s-features/pdb.yaml`:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: simple-color-app-pdb
  namespace: argo-rollouts-demo
spec:
  # minAvailable: 4 with replicas: 5 allows at most 1 voluntary eviction at a time.
  # Using minAvailable with an integer is the safe choice for non-built-in
  # controller types (Rollout is not a Deployment or StatefulSet).
  # Never set minAvailable equal to replicas — that permanently blocks node drains.
  minAvailable: 4
  selector:
    matchLabels:
      app: simple-color-app
```

---

## Step 6: Apply All Manifests

```bash
# Apply Rollout and Service
kubectl apply -f 05-canary-k8s-features/rollout.yaml
kubectl apply -f 05-canary-k8s-features/service.yaml

# Apply PDB — separate object, separate apply
kubectl apply -f 05-canary-k8s-features/pdb.yaml
```

**Watch the initial deployment:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — pods starting:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ⟳ Progressing
Strategy:        Canary
  Step:          0/2
  SetWeight:     0
  ActualWeight:  0
Images:          rselvantech/podinfo:red
Replicas:
  Desired:       5
  Current:       0
  Updated:       0
  Ready:         0
  Available:     0
```

**Expected — all pods running (Healthy):**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          2/2
  SetWeight:     100
  ActualWeight:  100
Images:          rselvantech/podinfo:red (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                        KIND        STATUS     AGE   INFO
⟳ simple-color-app                          Rollout     ✔ Healthy  45s
└──# revision:1
   └──⧉ simple-color-app-<hash>             ReplicaSet  ✔ Healthy  45s   stable
      ├──□ simple-color-app-<hash>-<pod>    Pod         ✔ Running  40s   ready:2/2
      ├──□ simple-color-app-<hash>-<pod>    Pod         ✔ Running  40s   ready:2/2
      ├──□ simple-color-app-<hash>-<pod>    Pod         ✔ Running  40s   ready:2/2
      ├──□ simple-color-app-<hash>-<pod>    Pod         ✔ Running  40s   ready:2/2
      └──□ simple-color-app-<hash>-<pod>    Pod         ✔ Running  40s   ready:2/2
```

> **`ready:2/2` — new in this demo.** Each pod shows two containers ready: the
> native sidecar (`timestamp-writer`) and the main container (`podinfo`). Demo-03
> showed `ready:1/1`. This is correct and expected — not an error.

> **Why the initial deployment does not pause at `setWeight: 20`:** Canary steps
> only run during an *update* against an existing stable version. On the first
> deploy there is no stable to protect — Argo Rollouts deploys all replicas
> directly as fast as possible. Same behaviour as Demo-03.

---

## Step 7: Verify Each Feature Is Active

### Step 7-1: Verify the sidecar is running in every pod

```bash
# Check container count per pod
kubectl get pods -n argo-rollouts-demo -l app=simple-color-app \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.name}={.ready}{" "}{end}{"\n"}{end}'
```

**Expected (one line per pod — 2 containers each):**
```
simple-color-app-<hash>-<pod>    timestamp-writer=true podinfo=true
simple-color-app-<hash>-<pod>    timestamp-writer=true podinfo=true
simple-color-app-<hash>-<pod>    timestamp-writer=true podinfo=true
simple-color-app-<hash>-<pod>    timestamp-writer=true podinfo=true
simple-color-app-<hash>-<pod>    timestamp-writer=true podinfo=true
```

```bash
# Spot-check the shared log — sidecar is writing timestamps
kubectl exec -n argo-rollouts-demo \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -c timestamp-writer -- tail -3 /shared/timestamps.log
```

**Expected:**
```
2025-06-01T10:23:44+00:00 sidecar heartbeat
2025-06-01T10:23:45+00:00 sidecar heartbeat
2025-06-01T10:23:46+00:00 sidecar heartbeat
```

### Step 7-2: Verify all three probes are configured

```bash
kubectl describe pod \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -n argo-rollouts-demo | grep -A5 "Startup:\|Liveness:\|Readiness:"
```

**Expected:**
```
    Startup:      http-get http://:9898/healthz delay=0s timeout=1s period=3s #success=1 #failure=10
    Liveness:     http-get http://:9898/healthz delay=10s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get http://:9898/readyz delay=5s timeout=1s period=5s #success=1 #failure=3
```

### Step 7-3: Verify the PodDisruptionBudget

```bash
kubectl get pdb -n argo-rollouts-demo
```

**Expected:**
```
NAME                   MIN AVAILABLE   MAX AVAILABLE   ALLOWED DISRUPTIONS   AGE
simple-color-app-pdb   4               N/A             1                     60s
```

`ALLOWED DISRUPTIONS: 1` — with 5 pods running and `minAvailable: 4`, exactly one
pod may be voluntarily evicted at a time.

### Step 7-4: Verify topology spread constraint is set

```bash
kubectl get pod -n argo-rollouts-demo -l app=simple-color-app \
  -o wide --no-headers | awk '{print $7}' | sort | uniq -c
```

**Expected (Minikube — all pods on one node, trivially satisfied):**
```
5 minikube
```

On a multi-node cluster, pods would be spread. The soft constraint (`ScheduleAnyway`)
means scheduling succeeds regardless.

### Step 7-5: Verify `terminationGracePeriodSeconds` and `preStop`

```bash
kubectl get pod \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -n argo-rollouts-demo \
  -o jsonpath='{.spec.terminationGracePeriodSeconds}'
```

**Expected:**
```
30
```

```bash
kubectl get pod \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -n argo-rollouts-demo \
  -o jsonpath='{.spec.containers[0].lifecycle.preStop.exec.command}'
```

**Expected:**
```
[/bin/sh,-c,sleep 5]
```

---

## Step 8: Push to Git Before Triggering the Update

```bash
cd src/argo-rollouts-config
git add 05-canary-k8s-features/
git commit -m "feat: demo-05 canary with startupProbe, native sidecar, PDB, topology spread, preStop, minReadySeconds, revisionHistoryLimit"
git push origin main
```

---

## Step 9: Trigger a Canary Update

**Edit `05-canary-k8s-features/rollout.yaml`** — change the image from `:red` to
`:blue`:

```yaml
          image: rselvantech/podinfo:blue   # ← was: rselvantech/podinfo:red
```

**Apply the update:**
```bash
kubectl apply -f 05-canary-k8s-features/rollout.yaml
```

**Watch the rollout in a second terminal:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — canary executing, paused at `setWeight: 20`:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ⏸ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/2
  SetWeight:     20
  ActualWeight:  20
Images:          rselvantech/podinfo:blue (canary)
                 rselvantech/podinfo:red (stable)
Replicas:
  Desired:       5
  Current:       6
  Updated:       1
  Ready:         6
  Available:     5      ← may show 5 for ~10s while minReadySeconds counts down

NAME                                        KIND        STATUS     AGE   INFO
⟳ simple-color-app                          Rollout     ⏸ Paused   3m
├──# revision:2
│  └──⧉ simple-color-app-<hash>             ReplicaSet  ✔ Healthy  30s   canary
│     └──□ simple-color-app-<hash>-<pod>   Pod         ✔ Running  28s   ready:2/2
└──# revision:1
   └──⧉ simple-color-app-<hash>             ReplicaSet  ✔ Healthy  3m    stable
      ├──□ simple-color-app-<hash>-<pod>   Pod         ✔ Running  3m    ready:2/2
      ├──□ simple-color-app-<hash>-<pod>   Pod         ✔ Running  3m    ready:2/2
      ├──□ simple-color-app-<hash>-<pod>   Pod         ✔ Running  3m    ready:2/2
      └──□ simple-color-app-<hash>-<pod>   Pod         ✔ Running  3m    ready:2/2
```

**Reading this output — new observations vs Demo-03:**

| Field | Demo-03 | Demo-05 | What it means |
|---|---|---|---|
| `ready:X/X` | `ready:1/1` | `ready:2/2` | Two containers per pod: sidecar + podinfo |
| `Available` | Immediately `5` | `5` for ~10s, then `6` | `minReadySeconds: 10` delays the canary pod counting as Available |
| All other fields | Same | Same | Strategy steps, weights, revisions — unchanged |

> **`Available: 5` while `Ready: 6`** — this is `minReadySeconds` in action.
> The canary pod passed its readiness probe, so `Ready: 6`. But it has not yet
> been continuously Ready for 10 seconds, so it has not yet counted as `Available`.
> After 10 seconds of sustained readiness, `Available` rises to 6 and the Rollout
> settles into its paused state.

**Verify the PDB during the surge:**

```bash
kubectl get pdb -n argo-rollouts-demo
```

**Expected:**
```
NAME                   MIN AVAILABLE   MAX AVAILABLE   ALLOWED DISRUPTIONS   AGE
simple-color-app-pdb   4               N/A             2                     ...
```

`ALLOWED DISRUPTIONS: 2` — with 6 pods (5 stable + 1 canary) and `minAvailable: 4`,
two pods may be evicted. The PDB is dynamic — it recalculates as the pod count
changes.

**Verify the traffic split:**

```bash
# Terminal 1 — get the Service URL
minikube service simple-color-app -n argo-rollouts-demo --url
```

**Expected:**
```
😿  service argo-rollouts-demo/simple-color-app has no node port
❗  Services [argo-rollouts-demo/simple-color-app] have type "ClusterIP" not meant to be exposed, however for local development minikube allows you to access this ClusterIP service through a tunnel
http://127.0.0.1:<port>
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```

```bash
# Terminal 2 — sample 20 responses
for i in $(seq 1 20); do
  curl -s http://127.0.0.1:<port> | grep -o 'v[12] — [a-z]*'
done
```

**Expected (approximately 20% blue responses):**
```
v1 — stable
v1 — stable
v1 — stable
v2 — canary
v1 — stable
v1 — stable
v1 — stable
v1 — stable
v2 — canary
v1 — stable
...
```

---

## Step 10: Walk Through the Web UI

```bash
kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
```

Open: **http://localhost:3100/rollouts**

### State 1 — Healthy (initial stable deployment, before any update)

**Main list view:**

| Field | Value |
|---|---|
| Status badge | ✔ Healthy (green) |
| Strategy | Canary |
| Weight | 100 |
| Pods | 5 green ✅ |
| Promote button | Greyed out — nothing to promote |

**Rollout detail view:**

| Panel | Field | Value |
|---|---|---|
| Steps | Set Weight: 20% | Completed |
| Steps | Pause | Completed |
| Summary | Step | 2/2 |
| Summary | Set Weight | 100 |
| Summary | Actual Weight | 100 |
| Revisions | Revision 1 | `rselvantech/podinfo:red` — `stable` badge — 5 pods |
| Pods | Each pod | `2/2 Running` ← two containers: timestamp-writer + podinfo |

> **New in this demo:** Every pod shows `2/2` containers ready, not `1/1`. This
> is the native sidecar. The pod count per revision is unchanged from Demo-03.

### State 2 — Paused (20% canary, after `setWeight: 20` and `pause: {}`)

**Rollout detail view:**

| Panel | Field | Value |
|---|---|---|
| Header | Status icon | ⏸ orange pause icon |
| Buttons | Abort | Active |
| Buttons | Promote | Active |
| Buttons | PromoteFull | Active |
| Steps | Set Weight: 20% | Completed — green border |
| Steps | Pause | Active — blue border — waiting for promotion |
| Summary | Step | 1/2 |
| Summary | Set Weight | 20 |
| Summary | Actual Weight | 20 |
| Revisions | Revision 2 | `rselvantech/podinfo:blue` — `canary` badge — 1 pod, `2/2 Running` |
| Revisions | Revision 1 | `rselvantech/podinfo:red` — `stable` badge — 4 pods, `2/2 Running` |

> **`Available` counter:** In the Replicas summary during the first ~10 seconds after
> the canary pod starts, you may see `Available: 5` while `Ready: 6`. This is
> `minReadySeconds: 10` in action — the canary pod must sustain readiness for 10s
> before counting as Available. Once it crosses that threshold, `Available` rises to 6.

> **Revision 1 shows a `Rollback` button** — same as Demo-03. Clicking it is
> equivalent to Abort.

### State 3 — Post-promotion (fully promoted to blue)

**Rollout detail view:**

| Panel | Field | Value |
|---|---|---|
| Header | Status icon | ✔ green healthy |
| Buttons | All except Restart | Greyed out |
| Summary | Step | 2/2 |
| Summary | Set Weight | 100 |
| Summary | Actual Weight | 100 |
| Revisions | Revision 2 | `rselvantech/podinfo:blue` — `stable` badge — 5 pods |
| Revisions | Revision 1 | `rselvantech/podinfo:red` — `ScaledDown` — 0 pods |

> **`revisionHistoryLimit: 3`** — only the most recent 3 retired revisions appear
> in the revision list. On later demos with many updates, older revisions are pruned
> automatically. In State 3 (first update) only 2 revisions exist — one active, one
> retired — so no pruning has occurred yet.

---

## Step 11: Promote the Rollout

```bash
kubectl argo rollouts promote simple-color-app -n argo-rollouts-demo
```

**Expected output:**
```
rollout 'simple-color-app' promoted
```

**Watch completion:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — final state:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          2/2
  SetWeight:     100
  ActualWeight:  100
Images:          rselvantech/podinfo:blue (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                        KIND        STATUS       AGE   INFO
⟳ simple-color-app                          Rollout     ✔ Healthy    7m
├──# revision:2
│  └──⧉ simple-color-app-<hash>             ReplicaSet  ✔ Healthy    4m    stable
│     ├──□ simple-color-app-<hash>-<pod>   Pod         ✔ Running    3m    ready:2/2
│     ├──□ ...
│     ├──□ ...
│     ├──□ ...
│     └──□ ...                              Pod         ✔ Running    3m    ready:2/2
└──# revision:1
   └──⧉ simple-color-app-<hash>             ReplicaSet  • ScaledDown  7m
```

---

## Step 12: Abort — Reset to Red

To reset the cluster for future demos, trigger a new update and abort it.

```bash
# Trigger blue → red update
kubectl argo rollouts set image simple-color-app \
  podinfo=rselvantech/podinfo:red \
  -n argo-rollouts-demo

# Wait for pause at 20%
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
# Wait until: Status: ⏸ Paused, Step: 1/2

# Abort
kubectl argo rollouts abort simple-color-app -n argo-rollouts-demo
```

**Expected after abort:**
```
Status:   ✖ Degraded
Message:  RolloutAborted: Rollout aborted update to revision 3
```

**Recover — apply the red image to spec to match the running cluster:**
```bash
# Edit rollout.yaml: change image back to rselvantech/podinfo:red
kubectl apply -f 05-canary-k8s-features/rollout.yaml
```

**Expected:**
```
Status:   ✔ Healthy
Images:   rselvantech/podinfo:red (stable)
```

---

## Step 13: Push Final Manifests

The final rollout.yaml should have `image: rselvantech/podinfo:red` — matching
the running cluster state.

```bash
cd src/argo-rollouts-config
git add 05-canary-k8s-features/
git commit -m "feat: demo-05 final manifests — canary with k8s features stable on red"
git push origin main
```

---

## Verify Final State

```bash
# Rollout is Healthy on stable red image
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
# Expected: Status Healthy, 5 pods, rselvantech/podinfo:red (stable)

# PDB is active
kubectl get pdb -n argo-rollouts-demo
# Expected: simple-color-app-pdb   4   N/A   1

# Service has 5 endpoints
kubectl get endpoints simple-color-app -n argo-rollouts-demo
# Expected: 5 pod IPs listed

# All pods showing 2 containers ready (sidecar + podinfo)
kubectl get pods -n argo-rollouts-demo
# Expected: 5 pods, READY shows 2/2 for each

# Sidecar still writing
kubectl exec -n argo-rollouts-demo \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -c timestamp-writer -- tail -3 /shared/timestamps.log
# Expected: 3 timestamp lines showing current time
```

---

## Cleanup

```bash
kubectl delete -f 05-canary-k8s-features/rollout.yaml
kubectl delete -f 05-canary-k8s-features/service.yaml
kubectl delete -f 05-canary-k8s-features/pdb.yaml

# Verify
kubectl get all -n argo-rollouts-demo
# Expected: No resources found in argo-rollouts-demo namespace.

kubectl get pdb -n argo-rollouts-demo
# Expected: No resources found.
```

> **The namespace and `dockerhub-secret` are kept.** All demos share
> `argo-rollouts-demo` and the pull secret. Only the Rollout, Service, and PDB are
> deleted between demos.

---

## Key Concepts Summary

**`ready:2/2` is expected when a native sidecar is running.** Each pod now has two
containers: `timestamp-writer` (native sidecar) and `podinfo` (main container). The
CLI dashboard reflects this in the `ready:X/Y` column. This is correct behaviour
with Argo Rollouts v1.7+. Earlier versions showed these pods stuck on `Init` due
to a bug in the dashboard pod-status display (fixed in PR #3639).

**`startupProbe` suspends the other two probes.** While the startup probe is active,
neither liveness nor readiness fires. The startup probe runs once and is then done.
This prevents premature liveness restarts on slow-starting containers without
sacrificing liveness responsiveness during steady-state operation.

**`minReadySeconds` delays `Available`, not `Ready`.** The Rollout controller waits
for `Available` pods before advancing steps. `Available` lags `Ready` by up to
`minReadySeconds` seconds. During a canary, the `Available` counter momentarily
showing less than `Ready` is the visible proof that `minReadySeconds` is active.

**PDB `ALLOWED DISRUPTIONS` is live and dynamic.** It updates automatically as pod
count changes. During the canary surge (5 + 1 = 6 pods, `minAvailable: 4`) it shows
`ALLOWED DISRUPTIONS: 2`. After promotion (5 pods) it drops back to
`ALLOWED DISRUPTIONS: 1`. This means node drains are slightly more permissive
during a canary — understand this when sizing `minAvailable` for production.

**`terminationGracePeriodSeconds` is pod spec level — not container level.** Place it
at `spec.template.spec`, as a sibling of `containers`. Placing it inside a container
definition is valid YAML but silently ignored by Kubernetes.

**PDB is a separate object — the Rollout controller does not create or own it.** You
apply it once independently. The Eviction API consults it whenever a drain or
scale-in eviction is attempted. Direct `kubectl delete pod` bypasses the Eviction
API and the PDB entirely.

**`revisionHistoryLimit: 3` counts only retired ReplicaSets.** The active stable
ReplicaSet and any active canary ReplicaSet are not counted. Setting `3` means up
to 3 previous versions are available as rollback targets on top of the current ones.

---

## Commands Reference

```bash
# Apply manifests
kubectl apply -f 05-canary-k8s-features/rollout.yaml
kubectl apply -f 05-canary-k8s-features/service.yaml
kubectl apply -f 05-canary-k8s-features/pdb.yaml

# Watch rollout progress (primary observation command)
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch

# Trigger canary update via image change
kubectl argo rollouts set image simple-color-app \
  podinfo=rselvantech/podinfo:blue -n argo-rollouts-demo

# Promote (advance past pause step)
kubectl argo rollouts promote simple-color-app -n argo-rollouts-demo

# Abort (revert to stable)
kubectl argo rollouts abort simple-color-app -n argo-rollouts-demo

# Check PodDisruptionBudget status
kubectl get pdb -n argo-rollouts-demo

# Verify all three probes are configured on a pod
kubectl describe pod \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -n argo-rollouts-demo | grep -A5 "Startup:\|Liveness:\|Readiness:"

# Exec into sidecar container
kubectl exec -n argo-rollouts-demo \
  $(kubectl get pod -n argo-rollouts-demo -l app=simple-color-app -o name | head -1) \
  -c timestamp-writer -- tail -5 /shared/timestamps.log

# Get the Service URL for browser/curl testing
minikube service simple-color-app -n argo-rollouts-demo --url

# Web UI dashboard
kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
# Open: http://localhost:3100/rollouts
```

---

## Lessons Learned

**1. `ready:2/2` in the CLI dashboard is correct — not a stuck init container.**
With native sidecars, each pod has two containers. Argo Rollouts v1.7+ displays
this correctly. If you see `Init:0/1` or pods stuck on Init, check your controller
version — anything before v1.7 has the dashboard display bug (issue #3366).

**2. `minReadySeconds` delays `Available`, not `Ready` — these are different counts.**
During a canary, watch both fields in the dashboard. `Ready: 6` and `Available: 5`
for the first 10 seconds is expected behaviour, not a stuck rollout. The controller
waits for Available before it would advance to the next step.

**3. PDB `ALLOWED DISRUPTIONS` is dynamic and increases during canary surge.**
With `minAvailable: 4` and 5 replicas, you allow 1 disruption normally. During the
canary surge (6 pods), you allow 2. If your maintenance window runs during a canary,
this matters — more pods can be evicted than you may have intended.

**4. `terminationGracePeriodSeconds` goes at pod spec level — not inside `containers`.**
This is the most common placement mistake. Inside a container block it is a no-op.
It must be a sibling of `containers`, inside `spec.template.spec`.

**5. Separate files for separate concerns.**
PDB is a separate `kubectl apply` — not embedded in the Rollout. If you delete the
Rollout, the PDB stays (protecting whatever pods remain). This mirrors how production
teams manage PDBs — as independent objects with their own lifecycle, not coupled to
any single workload definition.

**6. `ScheduleAnyway` vs `DoNotSchedule` in topology spread.**
On Minikube (single node) `DoNotSchedule` would block canary pods from scheduling
if the constraint cannot be satisfied (it trivially can be on one node, but the
principle holds). Always use `ScheduleAnyway` in dev. Switch to `DoNotSchedule` in
production only after confirming your cluster has the required topology.

**7. Push to Git before triggering the update.**
The commit timestamp in Git is your record of what was intended. Triggering the
update before committing means your Git history does not reflect the exact state
that was deployed. Always: edit → commit → push → apply → trigger.

---

## What's Next

**Demo-06: Blue-Green + Kubernetes Features**

The same seven Kubernetes features applied to a blue-green strategy. One new
concept: `previewReplicaCount` — running fewer pods in the preview (inactive) colour
to save resources during the pre-promotion window. The key difference from this
demo: blue-green doubles the pod count during the update (active + preview both
running simultaneously), so the PDB must account for `2 × replicas` at peak rather
than `replicas + 1` as in canary. Understanding how `minAvailable` interacts with
the blue-green pod doubling is the central new concept in Demo-06.