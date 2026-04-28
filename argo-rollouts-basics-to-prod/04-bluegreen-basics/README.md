# Demo-04: Blue-Green Basics — Instant Cutover with Zero Production Exposure

## Overview

This demo introduces the blue-green deployment strategy. Unlike canary —
which gradually shifts live production traffic to the new version — blue-green
keeps the new version completely isolated from production traffic while it is
being tested, then switches all traffic in a single instant.

By the end of this demo you will understand exactly how the two-Service
model works, why Argo Rollouts injects its own selector hash into your
Services (and why you must not set those selectors yourself), how to test
the preview version without any users hitting it, how the instant cutover
works, and when to use blue-green vs canary.

**What you'll learn:**
- Why blue-green needs two Services and canary only needs one
- How `activeService` and `previewService` work — and who manages their selectors
- What `autoPromotionEnabled: false` does and how it compares to canary's `pause: {}`
- What `scaleDownDelaySeconds` is, why it exists, and what happens if it is too short
- How to test the preview version (zero production traffic) before promoting
- How the CLI dashboard fields differ from canary — no Steps, different status messages
- How the web UI shows blue-green state vs canary state
- What `kubectl argo rollouts undo` does and how it differs from `abort`
- When to use blue-green vs canary — the decision framework

**What you'll do:**
- Write and apply a broken Rollout (missing `activeService`) to see the error
- Write the correct Rollout with `autoPromotionEnabled: false`
- Write `active-service.yaml` and `preview-service.yaml` as separate files
- Deploy stable (`:red`), trigger update to `:blue`, verify preview isolation
- Promote via CLI (instant cutover), observe CLI and web UI post-promotion
- Rollback with `kubectl argo rollouts undo` and understand why it differs from abort
- Promote via web UI — repeat cycle

---

## Prerequisites

- ✅ Demo-03 completed — `argo-rollouts-demo` namespace exists, `dockerhub-secret` exists
- ✅ `rselvantech/podinfo:red` and `rselvantech/podinfo:blue` in Docker Hub (from Demo-03 prerequisite)
- ✅ `kubectl argo rollouts` plugin installed
- ✅ Argo Rollouts controller running in `argo-rollouts` namespace

> **Images are already built.** This demo uses the same `:red` and `:blue`
> images built in the `README-build-podinfo-images.md` prerequisite of
> Demo-03. No new image builds are needed.

> **`dockerhub-secret` already exists** in `argo-rollouts-demo` from
> Demo-03. If you have reset your cluster since then, recreate it —
> see Demo-03 Step 4 for the exact command and the ArgoCD Demo-05
> cross-reference for PAT rules.

**Verify:**
```bash
# Controller running
kubectl get pods -n argo-rollouts
# Expected: argo-rollouts-* Running 1/1

# Namespace and secret from Demo-03 exist
kubectl get namespace argo-rollouts-demo
kubectl get secret dockerhub-secret -n argo-rollouts-demo
# Expected: both exist

# Images available locally
docker images | grep podinfo
# Expected: rselvantech/podinfo:red and rselvantech/podinfo:blue

# Dashboard port-forward
kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts &
# Open: http://localhost:3100/rollouts
```

---

## Concepts

### Blue-Green — How It Works

A blue-green rollout maintains two complete, fully-deployed ReplicaSets:
the **active** stack (current production version, receiving all traffic)
and the **preview** stack (new version, deployed and ready but receiving
zero production traffic).

```
Steady state — stable version only:
  activeService  → stable ReplicaSet (5 pods, podinfo:red)
  previewService → stable ReplicaSet (same pods — both Services converge here)

Update triggered — new version deployed:
  activeService  → stable ReplicaSet (5 pods, podinfo:red)  ← all production traffic
  previewService → new    ReplicaSet (5 pods, podinfo:blue) ← zero production traffic

  ← New pods are fully deployed and passing readiness probes.
  ← Zero users are hitting them. You can test via previewService privately.
  ← Rollout is paused (autoPromotionEnabled: false).

On promotion:
  activeService  → new ReplicaSet (5 pods, podinfo:blue) ← instant switch
  previewService → new ReplicaSet (same pods — both converge again)
  old ReplicaSet → scaled down after scaleDownDelaySeconds (default: 30s)
```

The key difference from canary: **no user sees the new version until you
explicitly promote.** In canary, real production traffic hits the canary
pod the moment it passes readiness. In blue-green, production traffic
stays on the active Service pointing at old pods for the entire duration
of testing.

### Why Two Services — The Selector Mechanism

This is the most important technical detail to understand about blue-green.

Kubernetes Services route traffic to pods via label selectors. When you
write a Service, you set `spec.selector` to match pod labels. Argo Rollouts
blue-green works by **injecting an additional selector** into both Services
— specifically the ReplicaSet's unique pod template hash — so each Service
points at exactly one ReplicaSet and not the other.

```
Your Service manifest (what you write):
  selector:
    app: simple-color-app

What Argo Rollouts injects at runtime:
  selector:
    app: simple-color-app
    rollouts-pod-template-hash: cc7bbd5b7    ← added by controller
                                                changes with every new ReplicaSet
```

You define the base selector (`app: simple-color-app`). Argo Rollouts
adds the hash. This is how active and preview Services each point at
exactly one ReplicaSet even though both ReplicaSets share the same
`app` label.

**Critical consequence:** Do NOT set `rollouts-pod-template-hash` in your
Service manifests. You do not know the hash before deploying — the
controller calculates it from the pod template. If you hardcode it, the
hash will not match the newly created ReplicaSet and routing will break.
Write only the base `app` label in your Service selector and let Argo
Rollouts manage the rest.

**Why canary does not need this:** In canary, you want both ReplicaSets
behind the same Service simultaneously — the traffic mix is intentional.
A single Service with just the base `app` selector works because you want
all matching pods to receive requests. In blue-green, you need hard
isolation — each Service must point at exactly one ReplicaSet — so the
injected hash is necessary.

### `activeService` Is Mandatory — `previewService` Is Optional

From the official docs (`argoproj.github.io/argo-rollouts/features/bluegreen`):

> "The active Service is used to send regular application traffic to the
> old version, while the preview Service is used to funnel traffic to
> the new version."

`activeService` is **required**. A Rollout with a `blueGreen` strategy
but no `activeService` field is immediately `Degraded` with `InvalidSpec`.
This is demonstrated in Step 3.

`previewService` is **optional**. Without it, the new ReplicaSet is still
deployed and still tested via readiness probes, but there is no dedicated
Service to reach it privately. You lose the ability to test the preview
version before promoting. For this series, `previewService` is always
included — the whole point of blue-green is pre-promotion testing.

### `autoPromotionEnabled: false` — The Manual Gate

```yaml
strategy:
  blueGreen:
    autoPromotionEnabled: false
```

When set to `false`, the rollout pauses immediately before promotion —
after the new ReplicaSet is fully ready and the preview Service is pointing
at it, but before the active Service switches. It waits indefinitely until
you promote manually.

When set to `true` (the default if the field is omitted), the rollout
promotes automatically as soon as the new ReplicaSet is fully ready. This
behaves like a rolling update with instant cutover — useful in CD pipelines
where every commit is trusted but not appropriate when you want manual
verification.

```
autoPromotionEnabled: false
  → deploy new RS → wait for ready → PAUSE (BlueGreenPause)
  → wait for: kubectl argo rollouts promote <n>
  → active Service switches → scaleDownDelay starts → old RS scaled down

autoPromotionEnabled: true (default)
  → deploy new RS → wait for ready → IMMEDIATELY promote
  → active Service switches → scaleDownDelay starts → old RS scaled down
  (No pause. No human gate. Suitable only when all commits are auto-tested.)
```

`autoPromotionSeconds: 30` is a middle option — auto-promotes after N
seconds if not manually promoted. Used when you want a timed observation
window without requiring manual action every time.

> **Argo CD comparison:**
> Argo CD has no promotion gate within the sync process. It applies what
> is in Git and marks the Application healthy when resources are healthy.
> Promotion control in a GitOps pipeline is handled at the Git layer
> (branch protection, PR approvals). Argo Rollouts' `autoPromotionEnabled:
> false` is a cluster-side gate that holds the cutover regardless of how
> the manifest change arrived — GitOps or manual `kubectl apply`.

### `scaleDownDelaySeconds` — Why Old Pods Stay Running After Promotion

From the official docs:

> "When the rollout changes the selector on a service, there is a
> propagation delay before all the nodes update their IP tables to send
> traffic to the new pods instead of the old. During this delay, traffic
> will be directed to the old pods if the nodes have not been updated yet.
> In order to prevent the packets from being sent to a node that killed
> the old pod, the rollout uses the `scaleDownDelaySeconds` field to give
> nodes enough time to broadcast the IP table changes."

After the active Service selector is updated to point at the new
ReplicaSet, the old pods continue running for `scaleDownDelaySeconds`
(default: 30 seconds). This is not a grace period for in-flight requests
— it is specifically to allow iptables propagation across all nodes in
the cluster.

```
t=0s:   Promote triggered
        activeService selector → updated to new ReplicaSet hash
        iptables change broadcast begins across all nodes

t=0-30s: Old pods still running
          Nodes that have updated iptables → send traffic to new pods
          Nodes that have NOT yet updated → still send traffic to old pods
          Both old and new pods receive traffic during this window

t=30s:  scaleDownDelaySeconds elapsed
        Old ReplicaSet scaled down to 0 replicas
        All nodes have had time to propagate iptable changes
```

Setting `scaleDownDelaySeconds: 0` means old pods are killed immediately
after the selector switch. Nodes that have not yet updated their iptables
will send packets to pods that no longer exist, causing brief connection
errors on promotion.

In this demo we use the default of 30 seconds. You will observe the old
ReplicaSet staying in the `delay` state briefly after promotion before
becoming `ScaledDown`.

### `undo` vs `abort` — The Two Revert Paths

Two commands can revert a blue-green update. They apply at different
points in the lifecycle and have different effects:

```
kubectl argo rollouts abort simple-color-app -n argo-rollouts-demo

  When to use: rollout is PAUSED before promotion
               (preview deployed, active still on old version)
  What it does: scales down the preview ReplicaSet immediately
                leaves the active Service and its ReplicaSet unchanged
  Result: Status → ✖ Degraded (spec says new image, cluster runs old image)
  User impact: ZERO — production never saw the new version

kubectl argo rollouts undo simple-color-app -n argo-rollouts-demo

  When to use: rollout has ALREADY BEEN PROMOTED
               (active Service is on new version, users are seeing it)
  What it does: creates a new revision using the previous revision's
                pod template — as if you applied the old image yourself
                goes through the full blue-green cycle with the old image
  Result: new revision deployed to preview, paused, waits for promote
  User impact: users stay on new version UNTIL you promote the undo
```

The distinction matters because `abort` is a cancellation (no exposure)
and `undo` is a recovery (exposure already happened, creating a new
rollout to fix it). After `undo`, you must still manually promote to
complete the rollback to the old version.

> **Argo CD + Rollouts (proj-02) note:**
> In a GitOps setup, `undo` creates drift between Git and the cluster —
> Git says new image, cluster after undo rolls back to old image. The
> correct approach in GitOps is `git revert` so Git and cluster stay in
> sync. In standalone demos (03–11), `undo` is fine because there is no
> GitOps sync watching.

### Blue-Green vs Canary — Decision Framework

| Situation | Use Blue-Green | Use Canary |
|---|---|---|
| New version must not touch real users before testing | ✅ | ❌ (canary pod serves real traffic immediately) |
| Need to run integration tests on the new version privately | ✅ | ❌ |
| Breaking schema change — all-or-nothing cutover needed | ✅ | ❌ |
| Regulatory requirement: zero exposure before sign-off | ✅ | ❌ |
| Want gradual traffic shifting to limit blast radius | ❌ | ✅ |
| Want metric-driven automated promotion | ❌ (use with postPromotionAnalysis) | ✅ |
| Resource-constrained cluster (can't afford 2× replicas) | ❌ | ✅ |
| Want to test with a real sample of production traffic | ❌ | ✅ |

Both strategies support analysis (Demos 07–10). The fundamental difference
is isolation: blue-green gives you a private test environment before any
cutover; canary gives you a controlled production experiment.

---

## Folder Structure

```
04-bluegreen-basics/
├── README.md                                   ← This file
└── src/
      ├──argo-rollouts-config/                  ← already exists from Demo-03
      │   └── 04-bluegreen-basics/
      │       ├── rollout.yaml                  ← Rollout CRD with blueGreen strategy
      │       ├── active-service.yaml           ← Production traffic Service
      │       └── preview-service.yaml          ← Test/preview traffic Service
      └──podinfo/                               ← already exists from Demo-03 prerequisite
          └── (no changes in this demo — Dockerfile already modified)
```

> **`argo-rollouts-config` repo already exists** from Demo-03. This demo
> adds a new `04-bluegreen-basics/` subdirectory to the existing repo.
> No new GitHub repo creation needed — navigate into the existing clone.

> **`podinfo/` clone already exists** from the Demo-03 prerequisite
> (`README-build-podinfo-images.md`). No Dockerfile changes are needed
> in this demo — the `:red` and `:blue` images are already built and
> pushed to Docker Hub.

---

## Step 1: Check Podinfo Images

```bash
docker images | grep podinfo
```

**Expected:**
```
rselvantech/podinfo   blue   <digest>   Xd ago   ~50MB
rselvantech/podinfo   red    <digest>   Xd ago   ~50MB
```

If not present locally, pull them:
```bash
docker pull rselvantech/podinfo:red
docker pull rselvantech/podinfo:blue
```

---

## Step 2: Create the Demo Subdirectory

```bash
cd argo-rollouts-basics-to-prod/04-bluegreen-basics/src/argo-rollouts-config

mkdir -p 04-bluegreen-basics
```

---

## Step 3: Write the Rollout Manifest — Intentionally Broken First

Blue-green has its own required field: `activeService`. A Rollout with
`strategy.blueGreen` but no `activeService` is immediately `Degraded`.
Observe this before writing the correct manifest.

**Create `src/argo-rollouts-config/04-bluegreen-basics/rollout.yaml`
with an empty blueGreen strategy block:**

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
        - name: simple-color-app
          image: rselvantech/podinfo:red
          ports:
            - containerPort: 9898
  strategy:
    blueGreen: {}
```

**Apply it:**
```bash
kubectl apply -f 04-bluegreen-basics/rollout.yaml
```

**Expected output:**
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
Message:         InvalidSpec: blueGreen requires activeService to be set
```

`blueGreen: {}` passes syntax validation. The controller catches the
missing `activeService` on reconciliation. Fix in the next step.

---

## Step 4: Write the Correct Rollout Manifest

**Replace the contents of `04-bluegreen-basics/rollout.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: simple-color-app
  namespace: argo-rollouts-demo
spec:
  replicas: 5
  revisionHistoryLimit: 2
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
        - name: simple-color-app
          image: rselvantech/podinfo:red
          ports:
            - name: http
              containerPort: 9898
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
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              memory: 256Mi
  strategy:
    blueGreen:
      # activeService: mandatory — name of the Service receiving all production traffic.
      # Argo Rollouts injects rollouts-pod-template-hash into this Service's selector
      # at runtime. Do NOT add that selector in active-service.yaml — the controller
      # manages it. Write only the base app label in the Service.
      activeService: simple-color-app-active

      # previewService: optional — name of the Service pointing at the new ReplicaSet.
      # Zero production traffic flows here. Use this to test the new version before
      # promoting. If omitted, no private test endpoint exists.
      previewService: simple-color-app-preview

      # autoPromotionEnabled: false — pause before switching active Service.
      # The rollout waits indefinitely after preview is fully ready.
      # Promote with: kubectl argo rollouts promote simple-color-app -n argo-rollouts-demo
      # Or click Promote in the web UI.
      autoPromotionEnabled: false

      # scaleDownDelaySeconds: keep old pods running this long after active Service
      # selector switch. Allows iptables propagation across all cluster nodes.
      # Default is 30s. Never set to 0 — brief connection errors will occur.
      scaleDownDelaySeconds: 30
```

**Apply:**
```bash
kubectl apply -f 04-bluegreen-basics/rollout.yaml
```

---

## Step 5: Write the Service Manifests

Two separate files — one per Service. This makes the active vs preview
distinction explicit and mirrors real production practice where each
Service has its own lifecycle and possibly its own Ingress rule.

**Create `04-bluegreen-basics/active-service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-color-app-active
  namespace: argo-rollouts-demo
spec:
  # Write only the base selector. Argo Rollouts injects:
  #   rollouts-pod-template-hash: <hash>
  # at runtime so this Service points at exactly the active ReplicaSet.
  # The hash changes with every new ReplicaSet — never hardcode it here.
  selector:
    app: simple-color-app
  ports:
    - name: http
      port: 80
      targetPort: 9898
      protocol: TCP
```

**Create `04-bluegreen-basics/preview-service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-color-app-preview
  namespace: argo-rollouts-demo
spec:
  # Same base selector as active-service. Argo Rollouts injects a DIFFERENT
  # hash here — pointing at the preview (new) ReplicaSet instead of the active.
  # In steady state both Services share the same hash (same ReplicaSet).
  # During an update they diverge — each pointing at a different ReplicaSet.
  selector:
    app: simple-color-app
  ports:
    - name: http
      port: 80
      targetPort: 9898
      protocol: TCP
```

**Apply both Services:**
```bash
kubectl apply -f 04-bluegreen-basics/active-service.yaml
kubectl apply -f 04-bluegreen-basics/preview-service.yaml
```

**Watch the Rollout reach Healthy:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — initial deployment Healthy:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          rselvantech/podinfo:red (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS     AGE   INFO
⟳ simple-color-app                        Rollout     ✔ Healthy  60s
└──# revision:1
   └──⧉ simple-color-app-xxxxxxxxxx       ReplicaSet  ✔ Healthy  60s   stable,active
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  55s   ready:1/1
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  55s   ready:1/1
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  55s   ready:1/1
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  55s   ready:1/1
      └──□ simple-color-app-xxx-xxx        Pod         ✔ Running  55s   ready:1/1
```

**Key observations — initial healthy state:**
- `Strategy: BlueGreen` confirmed. No `Step:` field. No `SetWeight:`.
- `INFO: stable,active` — one ReplicaSet is both stable version and the
  active Service target. In steady state there is only one ReplicaSet.
- No `preview` label yet — appears only when an update is in progress.

**Verify the selector injection:**
```bash
kubectl get service simple-color-app-active -n argo-rollouts-demo -o yaml \
  | grep -A5 selector
```

**Expected:**
```yaml
selector:
  app: simple-color-app
  rollouts-pod-template-hash: xxxxxxxxxx   ← injected by Argo Rollouts controller
```

The controller added `rollouts-pod-template-hash` automatically. The hash
matches the ReplicaSet name suffix in the `get rollout` tree output. Verify
the preview Service has the same hash in steady state:

```bash
kubectl get service simple-color-app-preview -n argo-rollouts-demo -o yaml \
  | grep rollouts-pod-template-hash
# Expected: same hash as active — both point at the same (stable) ReplicaSet
```

---

## Step 6: Read the CLI Dashboard — Blue-Green Fields

With a healthy blue-green Rollout running, compare the output structure
to what you saw in Demo-03 canary. Key differences are annotated below.

```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
```

**Annotated output — Healthy blue-green state:**

```
Name:            simple-color-app       ← same as canary
Namespace:       argo-rollouts-demo     ← same as canary
Status:          ✔ Healthy             ← same symbols as canary
Message:                                ← empty when Healthy
Strategy:        BlueGreen             ← was "Canary" in Demo-03

                 ← NO "Step: N/N" field — blue-green has no steps list
                 ← NO "SetWeight:" / "ActualWeight:" — not applicable
                    traffic is binary: 0% or 100%, never a percentage weight

Images:          rselvantech/podinfo:red (stable)
                 ← same format as canary

Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5
  ← same five fields as canary, same meanings
```

**Blue-green INFO column labels explained:**

| Label | What it means |
|---|---|
| `stable` | This ReplicaSet carries the currently promoted (production) image |
| `active` | The `activeService` (`simple-color-app-active`) selector hash points here |
| `canary` | This ReplicaSet carries the new version (Argo Rollouts uses "canary" internally for the new revision in both strategies) |
| `preview` | The `previewService` (`simple-color-app-preview`) selector hash points here |
| `delay` | Active Service has switched here, but `scaleDownDelaySeconds` is counting for old RS |
| `ScaledDown` | ReplicaSet exists at 0 replicas (kept for `undo`) |

**Label progression through the blue-green lifecycle:**

```
Steady state (one RS):
  revision:1  →  stable,active

Update triggered (two RS):
  revision:2  →  canary,preview     ← new RS, preview Service points here
  revision:1  →  stable,active      ← old RS, active Service still here

After promotion (scaledown in progress):
  revision:2  →  stable,active      ← now production
  revision:1  →  delay              ← scaleDownDelaySeconds counting

After scaledown (stable state):
  revision:2  →  stable,active
  revision:1  →  ScaledDown         ← kept for undo
```

---

## Step 7: Walk Through the Web UI — Healthy Blue-Green State

Open `http://localhost:3100/rollouts` → select namespace `argo-rollouts-demo`.

### Main List View

The Rollout card shows:

| Field | Value | Comparison to Demo-03 Canary |
|---|---|---|
| Strategy badge | `BlueGreen` (blue) | Was yellow `Canary` |
| Weight | `100` | Same — 100% on stable |
| Pods | 5 ✅ | Same |
| Promote button | Greyed out | Same — nothing to promote |

### Rollout Detail View — Healthy State

Click the Rollout name.

**Steps panel:** **Not present.** This is the most immediate visual
difference from canary. Blue-green has no configurable steps list — the
only gate is `autoPromotionEnabled` and optionally analysis. Without a
Steps panel, the left side of the detail view is empty until an update is
triggered.

**Summary panel:** Shows `Strategy: BlueGreen` only. No Step counter,
no SetWeight, no ActualWeight — none of those fields apply to blue-green.

**Action buttons:**

| Button | State | Meaning |
|---|---|---|
| `Restart` | Active | Restarts all pods without image change |
| `Retry` | Greyed out | Nothing to retry |
| `Abort` | Greyed out | Nothing to abort — no update in progress |
| `Promote` | Greyed out | Nothing to promote — no preview deployed |
| `PromoteFull` | Greyed out | Same as Promote for blue-green |

**Revisions panel:**

| Revision | Image | Badges | Pods |
|---|---|---|---|
| Revision 1 | `podinfo:red` | `stable` + `active` | 5 ✅ |

Both `stable` and `active` badges on the same revision because the active
Service and the current production version point at the same ReplicaSet.

Keep this browser tab open — the next step triggers the update and
populates the preview revision.

---

## Step 8: Push Initial Manifests and Trigger the Update

Push the initial manifests first — repo state before applying the update:

```bash
cd argo-rollouts-basics-to-prod/04-bluegreen-basics/src/argo-rollouts-config
git add 04-bluegreen-basics/
git commit -m "feat: demo-04 initial blue-green rollout and services (stable: red)"
git push -u origin main
```

Now trigger the update — change only the image tag:

**Edit `04-bluegreen-basics/rollout.yaml`:**
```yaml
          image: rselvantech/podinfo:blue   # ← was: podinfo:red
```

**Apply:**
```bash
kubectl apply -f 04-bluegreen-basics/rollout.yaml
```

**Watch in a second terminal:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — new ReplicaSet deployed, Rollout paused:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ⏸ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          rselvantech/podinfo:blue (canary)
                 rselvantech/podinfo:red  (stable)
Replicas:
  Desired:       5
  Current:       10         ← both ReplicaSets fully running: 5 + 5
  Updated:       5
  Ready:         10
  Available:     10

NAME                                       KIND        STATUS     AGE    INFO
⟳ simple-color-app                        Rollout     ⏸ Paused   5m
├──# revision:2
│  └──⧉ simple-color-app-yyyyyyyyyy       ReplicaSet  ✔ Healthy  60s    canary,preview
│     ├──□ simple-color-app-yyy-yyy        Pod         ✔ Running  55s    ready:1/1
│     ├──□ simple-color-app-yyy-yyy        Pod         ✔ Running  55s    ready:1/1
│     ├──□ simple-color-app-yyy-yyy        Pod         ✔ Running  55s    ready:1/1
│     ├──□ simple-color-app-yyy-yyy        Pod         ✔ Running  55s    ready:1/1
│     └──□ simple-color-app-yyy-yyy        Pod         ✔ Running  55s    ready:1/1
└──# revision:1
   └──⧉ simple-color-app-xxxxxxxxxx       ReplicaSet  ✔ Healthy  5m     stable,active
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  5m     ready:1/1
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  5m     ready:1/1
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  5m     ready:1/1
      ├──□ simple-color-app-xxx-xxx        Pod         ✔ Running  5m     ready:1/1
      └──□ simple-color-app-xxx-xxx        Pod         ✔ Running  5m     ready:1/1
```

**Reading the paused blue-green output — what changed from healthy state:**

| Field | Healthy state | Paused blue-green | What it means |
|---|---|---|---|
| `Status` | `✔ Healthy` | `⏸ Paused` | Waiting at pre-promotion gate |
| `Message` | empty | `BlueGreenPause` | Specific to blue-green pause (vs `CanaryPauseStep` in canary) |
| `Images` | one line | two lines | Two complete ReplicaSets running |
| `Current` | `5` | `10` | Both full ReplicaSets running — double the pod count |
| `Updated` | `5` | `5` | 5 pods on new `:blue` image |
| revision:2 INFO | — | `canary,preview` | New RS; preview Service selector points here |
| revision:1 INFO | `stable,active` | `stable,active` | Old RS; active Service still here — unchanged |

**Verify the selector injection diverged:**
```bash
kubectl get service simple-color-app-active -n argo-rollouts-demo -o yaml \
  | grep rollouts-pod-template-hash
# Expected: hash matching revision:1 (xxxxxxxxxx)

kubectl get service simple-color-app-preview -n argo-rollouts-demo -o yaml \
  | grep rollouts-pod-template-hash
# Expected: hash matching revision:2 (yyyyyyyyyy) — different from active
```

The two Services now carry different hashes — each pointing at exactly
one ReplicaSet. This is the isolation mechanism in operation.

---

## Step 9: Test the Preview — Zero Production Exposure

This is the defining capability of blue-green. Test the new version
through the preview Service before any production user sees it.

**Get the active Service URL (production — still red):**
```bash
minikube service simple-color-app-active -n argo-rollouts-demo --url
```

Keep this terminal open. Open a new terminal for the curl test.

**Verify active serves only red:**
```bash
# Use the URL returned by minikube above
for i in $(seq 1 5); do
  curl -s <active-url> | grep -o 'v[12] — [a-z]*'
done
```

**Expected:**
```
v1 — stable
v1 — stable
v1 — stable
v1 — stable
v1 — stable
```

Zero blue responses. Production is completely isolated on red.

**Get the preview Service URL (new version — blue):**
```bash
minikube service simple-color-app-preview -n argo-rollouts-demo --url
```

Keep this terminal open too. Open another new terminal.

**Test the preview:**
```bash
# Use the preview URL returned above
for i in $(seq 1 5); do
  curl -s <preview-url> | grep -o 'v[12] — [a-z]*'
done
```

**Expected:**
```
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
```

All blue. The new version is running fully, passing readiness probes,
accessible for testing — and completely invisible to production users.

**Open both URLs in your browser simultaneously:**
- Active URL → red background, `v1 — stable (red)` — this is what users see
- Preview URL → blue background, `v2 — canary (blue)` — this is your test environment

Both versions side by side. This is the blue-green isolation model in its
most visible form.

---

## Step 10: Walk Through the Web UI — Paused Blue-Green State

Return to `http://localhost:3100/rollouts` → `simple-color-app`.

**Main List View:**
The card now shows an orange pause icon. Weight shows `0` — new version
has 0% of active Service traffic. This is correct — blue-green does not
use percentages. `0` means preview is deployed but not yet active.

**Rollout Detail View — Paused State:**

**Action buttons:**

| Button | State | Meaning |
|---|---|---|
| `Restart` | Active | Still available |
| `Retry` | Greyed out | Nothing to retry |
| `Abort` | **Active** | Scales down preview RS, active stays on red, no users affected |
| `Promote` | **Active** | Switches active Service selector to blue RS, begins 30s scaledown for red |
| `PromoteFull` | **Active** | Same effect as Promote for blue-green — no steps to skip |

> **Argo CD comparison:** These action buttons are the cluster-side
> equivalent of merging a PR or reverting a commit in a GitOps pipeline.
> In proj-02, ArgoCD's resource actions panel surfaces `Promote` and
> `Abort` for Rollout resources directly in the ArgoCD UI.

**Revisions panel — paused state:**

| Revision | Image | Badges | ReplicaSet | Pods |
|---|---|---|---|---|
| Revision 2 | `podinfo:blue` | `canary` + `preview` | `simple-color-app-yyyyyyyyyy` | 5 ✅ |
| Revision 1 | `podinfo:red` | `stable` + `active` | `simple-color-app-xxxxxxxxxx` | 5 ✅ |

Both revisions fully healthy. Revision 1 shows a `Rollback` button —
since active is still on red, clicking Rollback here is equivalent to
Abort: it scales down the preview and cancels the pending promotion.

---

## Step 11: Promote via CLI

```bash
kubectl argo rollouts promote simple-color-app -n argo-rollouts-demo
```

**Expected:**
```
rollout 'simple-color-app' promoted
```

**Watch:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — immediately after promote (active switched, scaledown pending):**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ⟳ Progressing
Message:         waiting for all steps to complete
Strategy:        BlueGreen
Images:          rselvantech/podinfo:blue (stable)
Replicas:
  Desired:       5
  Current:       10         ← old pods still running during scaleDownDelaySeconds
  Updated:       5
  Ready:         10
  Available:     10

NAME                                       KIND        STATUS        AGE    INFO
⟳ simple-color-app                        Rollout     ⟳ Progressing  6m
├──# revision:2
│  └──⧉ simple-color-app-yyyyyyyyyy       ReplicaSet  ✔ Healthy      2m    stable,active
│     └──□ ...  (5 pods Running)
└──# revision:1
   └──⧉ simple-color-app-xxxxxxxxxx       ReplicaSet  ✔ Healthy      6m    delay
      └──□ ...  (5 pods still running — scaleDownDelaySeconds: 30 counting)
```

**Key observation:** Revision 1 shows `delay` in the INFO column — the
controller is waiting for `scaleDownDelaySeconds: 30` before scaling
down the old pods. Revision 2 is already `stable,active` — the active
Service selector has already switched. Users are now on blue.

**Expected — after 30s scaledown completes (Healthy):**
```
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          rselvantech/podinfo:blue (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS        AGE    INFO
⟳ simple-color-app                        Rollout     ✔ Healthy     7m
├──# revision:2
│  └──⧉ simple-color-app-yyyyyyyyyy       ReplicaSet  ✔ Healthy     3m    stable,active
│     ├──□ ...  (5 pods Running)
└──# revision:1
   └──⧉ simple-color-app-xxxxxxxxxx       ReplicaSet  • ScaledDown  7m
```

**Verify all traffic is blue:**
```bash
for i in $(seq 1 10); do
  curl -s <active-url> | grep -o 'v[12] — [a-z]*'
done
```

**Expected:**
```
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
v2 — canary
```

All 10 responses are blue. Promotion complete.

### Web UI — Post-Promotion State (Screen 3)

Return to `http://localhost:3100/rollouts` → `simple-color-app`.

**Main List View:**
The card is back to a green tick. Weight shows `100`. Strategy badge shows
`BlueGreen`. Compare to Screen 1 (initial healthy state) — it looks
identical, but the image is now `:blue` not `:red`.

**Rollout Detail View — Post-Promotion:**

**Summary panel:**

| Field | Screen 1 (initial healthy) | Screen 3 (post-promotion) |
|---|---|---|
| Status | `✔ Healthy` | `✔ Healthy` |
| Strategy | `BlueGreen` | `BlueGreen` |
| Image | `podinfo:red` | `podinfo:blue` |

The summary looks the same. The only change is the image tag — blue
is now stable.

**Action buttons:**

| Button | State | Change from paused state |
|---|---|---|
| `Restart` | Active | Unchanged |
| `Abort` | **Greyed out** | Was active during pause — now nothing to abort |
| `Promote` | **Greyed out** | Was active during pause — now nothing to promote |
| `PromoteFull` | **Greyed out** | Same |

All promotion and abort buttons are greyed out — confirming no update
is in progress. This is the key visual signal that promotion completed
successfully.

**Revisions panel — post-promotion:**

| Revision | Image | Badges | Pods |
|---|---|---|---|
| Revision 2 | `podinfo:blue` | `stable` + `active` | 5 ✅ |
| Revision 1 | `podinfo:red` | `ScaledDown` | 0 |

Compare to Screen 2 (paused state):
- Revision 2 had `canary + preview` → now has `stable + active`
- Revision 1 had `stable + active` → now has `ScaledDown`
- Revision 1 pod count was 5 → now 0

The `ScaledDown` revision is kept in the list (not deleted) — it is
available for `undo` if needed. Clicking the `Rollback` button on
Revision 1 here triggers `undo` — deploying red to preview and pausing
for promotion, the same as `kubectl argo rollouts undo`.

---

## Step 12: Roll Back with `undo`

Promotion is complete — blue is active. Practice the rollback path.

```bash
kubectl argo rollouts undo simple-color-app -n argo-rollouts-demo
```

**Expected:**
```
rollout 'simple-color-app' rolled back
```

**Watch:**
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch
```

**Expected — revision:3 deploying (old red image), Rollout paused:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ⏸ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          rselvantech/podinfo:red  (canary)
                 rselvantech/podinfo:blue (stable)
Replicas:
  Desired:       5
  Current:       10
  Updated:       5
  Ready:         10
  Available:     10

NAME                                       KIND        STATUS     AGE    INFO
⟳ simple-color-app                        Rollout     ⏸ Paused   9m
├──# revision:3
│  └──⧉ simple-color-app-xxxxxxxxxx       ReplicaSet  ✔ Healthy  30s    canary,preview
│     └──□ ...  (5 pods, podinfo:red)
└──# revision:2
   └──⧉ simple-color-app-yyyyyyyyyy       ReplicaSet  ✔ Healthy  5m     stable,active
      └──□ ...  (5 pods, podinfo:blue — still serving production traffic)
```

**What `undo` did:**
- Created revision:3 using the pod template from revision:1 (`:red` image)
- Deployed 5 `:red` pods into the preview ReplicaSet
- Paused — waiting for you to promote
- **Production users are still on blue** (revision:2 active) until you promote

You must promote to complete the rollback:

```bash
kubectl argo rollouts promote simple-color-app -n argo-rollouts-demo
```

Wait for `Healthy` then verify:
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
# Expected: Images: podinfo:red (stable), revision:3 stable,active

for i in $(seq 1 10); do
  curl -s <active-url> | grep -o 'v[12] — [a-z]*'
done
# Expected: v1 — stable (all 10)
```

Production is back on red. `undo` + `promote` completed the two-step rollback.

---

## Step 13: Abort — Cancel Before Promotion

Trigger another update to blue, but abort it before promoting.

**Edit `rollout.yaml`** — change to blue:
```yaml
          image: rselvantech/podinfo:blue
```

```bash
kubectl apply -f 04-bluegreen-basics/rollout.yaml
```

Wait for `Status: ⏸ Paused` and `Message: BlueGreenPause`:
```bash
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
# Wait for: Status: ⏸ Paused
```

**Abort:**
```bash
kubectl argo rollouts abort simple-color-app -n argo-rollouts-demo
```

**Expected after abort:**
```
Name:            simple-color-app
Namespace:       argo-rollouts-demo
Status:          ✖ Degraded
Message:         RolloutAborted: Rollout aborted update to revision 5
Strategy:        BlueGreen
Images:          rselvantech/podinfo:red (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5
```

The preview ReplicaSet (`:blue`) was scaled down immediately. Active
Service never switched. Zero production users saw the new version.
Status is `Degraded` because spec says `:blue` but cluster runs `:red`.

**Recover:**
```bash
# Edit rollout.yaml back to red (matches running cluster)
# image: rselvantech/podinfo:red

kubectl apply -f 04-bluegreen-basics/rollout.yaml
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
# Expected: Status: ✔ Healthy immediately (spec and cluster agree)
```

---

## Step 14: Promote via Web UI — Repeat the Cycle

Trigger one final update and use the web UI to promote.

**Edit `rollout.yaml`** — change to blue:
```yaml
          image: rselvantech/podinfo:blue
```

```bash
kubectl apply -f 04-bluegreen-basics/rollout.yaml
```

Wait for `Status: ⏸ Paused` in the CLI dashboard.

Open `http://localhost:3100/rollouts` → click `simple-color-app` →
click **Promote** → confirm.

Watch the CLI simultaneously to observe the `delay` → `ScaledDown`
transition — the scaleDownDelaySeconds counting in the INFO column.

---

## Step 15: Push Final Manifests

End state: `rollout.yaml` has `image: rselvantech/podinfo:blue`.

```bash
cd argo-rollouts-basics-to-prod/04-bluegreen-basics/src/argo-rollouts-config
git add 04-bluegreen-basics/
git commit -m "feat: demo-04 final manifests — blue-green stable on blue"
git push origin main
```

---

## Verify Final State

```bash
# Rollout Healthy on blue
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo
# Expected: Status Healthy, Images: podinfo:blue (stable), revision:latest stable,active

# Active Service has 5 endpoints
kubectl get endpoints simple-color-app-active -n argo-rollouts-demo
# Expected: 5 pod IPs

# Preview Service points at same pods as active (converged in steady state)
kubectl get endpoints simple-color-app-preview -n argo-rollouts-demo
# Expected: same 5 pod IPs as active

# Both Service selectors now carry the same hash (same ReplicaSet)
kubectl get svc simple-color-app-active simple-color-app-preview \
  -n argo-rollouts-demo -o yaml | grep rollouts-pod-template-hash
# Expected: same hash value on both

# Old ReplicaSet is ScaledDown (not deleted)
kubectl get replicasets -n argo-rollouts-demo
# Expected: latest RS has READY=5, previous RS has READY=0
```

---

## Cleanup

```bash
kubectl delete -f 04-bluegreen-basics/rollout.yaml
kubectl delete -f 04-bluegreen-basics/active-service.yaml
kubectl delete -f 04-bluegreen-basics/preview-service.yaml

kubectl get all -n argo-rollouts-demo
# Expected: No resources found in argo-rollouts-demo namespace.
```

> **Namespace and `dockerhub-secret` are kept.** They are shared across
> all demos in the series. Never delete them between demos.

---

## Key Concepts Summary

**Blue-green deploys the full new version before any user sees it**
The new ReplicaSet is fully deployed — all 5 pods passing readiness — before
the active Service selector switches. In canary, the first user hits the
new version the moment its first pod passes readiness.

**`activeService` is mandatory — `previewService` is optional but always used**
Without `activeService`, the rollout immediately enters `Degraded` with
`InvalidSpec`. Without `previewService`, there is no private test endpoint
before promotion — defeating the purpose of blue-green.

**Argo Rollouts injects `rollouts-pod-template-hash` — never set it yourself**
The hash is calculated from the pod template and injected at runtime. It
changes with every new ReplicaSet. Write only the base `app` selector;
the controller manages the routing hash.

**`autoPromotionEnabled: false` is the manual gate — default is auto-promote**
If the field is omitted or set to `true`, the rollout promotes automatically
as soon as the preview is ready. Always set `false` when manual verification
is required.

**`scaleDownDelaySeconds` protects against iptables propagation lag**
Old pods stay running for 30 seconds (default) after the active Service
selector switches. Setting this to 0 causes brief connection errors as
packets arrive at killed pods from nodes with stale iptables.

**Blue-green temporarily doubles your pod count**
During an update: 5 active pods + 5 preview pods = 10 pods. Plan for 2×
resource headroom. `previewReplicaCount` (Demo-06) can reduce preview pod
count if running fewer pods for testing is acceptable.

**`undo` creates a new rollout — it is a two-step operation**
`undo` deploys the old image to preview and pauses. You must still
`promote` to complete the rollback. Production stays on the current
version until you promote the undo. In GitOps (proj-02), prefer `git
revert` over `undo` to avoid Git/cluster drift.

**Both Services point at the same ReplicaSet in steady state**
After promotion, the preview Service selector hash is updated to match
the active Service. They diverge again only when the next update is
triggered. Verify with `kubectl get svc -o yaml | grep rollouts-pod-template-hash`.

---

## Lessons Learned

**1. `blueGreen: {}` is accepted by `kubectl apply` but immediately Degraded**
An empty `blueGreen` block passes syntax validation. The controller
rejects it with `InvalidSpec: blueGreen requires activeService to be set`.
Always verify with `kubectl argo rollouts get rollout` after any apply —
`created` does not mean `healthy`.

**2. Do not set `rollouts-pod-template-hash` in your Service selector**
The controller calculates and injects this hash at runtime. It changes with
every new ReplicaSet. Hardcoding it means the Service points at the wrong
ReplicaSet the moment a new revision is deployed. Write only the base `app`
label and let the controller manage routing.

**3. Both Services have the same selector YAML — Argo Rollouts differentiates them**
`active-service.yaml` and `preview-service.yaml` look identical in the
manifest. The controller injects different hashes into each at runtime.
This is by design — you declare intent (active vs preview), the controller
handles the routing.

**4. `autoPromotionEnabled` defaults to `true` — always set `false` explicitly**
If the field is omitted from the manifest, the rollout auto-promotes the
moment the preview is ready. There is no pause, no manual gate. Set
`autoPromotionEnabled: false` explicitly in any rollout where manual
verification before promotion is required.

**5. Blue-green doubles your pod count during updates — plan for it**
10 pods running simultaneously during an update with `replicas: 5`. On a
local Minikube with limited resources this may cause scheduling pressure.
Reduce `replicas` for local demos if needed. In production, ensure your
cluster has headroom for 2× before enabling blue-green.

**6. `scaleDownDelaySeconds: 0` causes brief connection errors**
Old pods are killed immediately after the active Service selector switches.
Nodes with stale iptables continue routing to dead pods for a brief window.
The default of 30 seconds covers typical iptables propagation. Never set
this to 0 in production.

**7. `undo` does NOT immediately revert production traffic**
`undo` creates a new revision (revision:N) using the old image, deploys
it to preview, and pauses. Production stays on the current (new) version
until you explicitly promote. Always follow `undo` with `promote` to
complete the rollback. If you want to cancel before any user exposure,
use `abort` instead.

**8. In GitOps (proj-02), `git revert` is preferred over `undo`**
`undo` changes the cluster state without changing Git. This creates drift
— Git says new image, cluster runs old image after undo completes. In
proj-02 where ArgoCD watches the Git repo, use `git revert` so that Git
and the cluster stay in sync.

---

## Commands Reference

```bash
# Apply manifests
kubectl apply -f 04-bluegreen-basics/rollout.yaml
kubectl apply -f 04-bluegreen-basics/active-service.yaml
kubectl apply -f 04-bluegreen-basics/preview-service.yaml

# Watch rollout progress
kubectl argo rollouts get rollout simple-color-app -n argo-rollouts-demo --watch

# Promote (switch active Service to new version)
kubectl argo rollouts promote simple-color-app -n argo-rollouts-demo

# Abort (cancel before promotion — preview scaled down, active unchanged)
kubectl argo rollouts abort simple-color-app -n argo-rollouts-demo

# Undo (roll back after promotion — creates new rollout with previous image)
kubectl argo rollouts undo simple-color-app -n argo-rollouts-demo

# Verify selector injection
kubectl get svc simple-color-app-active -n argo-rollouts-demo -o yaml \
  | grep -A5 selector
kubectl get svc simple-color-app-preview -n argo-rollouts-demo -o yaml \
  | grep -A5 selector

# Traffic verification (use URL from minikube service command)
minikube service simple-color-app-active -n argo-rollouts-demo --url
minikube service simple-color-app-preview -n argo-rollouts-demo --url

for i in $(seq 1 10); do
  curl -s <url> | grep -o 'v[12] — [a-z]*'
done
```

---

## What's Next

**Demo-05: Canary + Kubernetes Features**
Take the canary strategy from Demo-03 and add production-grade Kubernetes
features on top: `startupProbe`, native sidecar containers (Kubernetes
1.29+ `restartPolicy: Always`), `PodDisruptionBudget`, `topologySpreadConstraints`,
`preStop` lifecycle hook, `minReadySeconds`, and `revisionHistoryLimit`.
Each feature is added to the same Rollout incrementally, with an explanation
of why it matters specifically during a progressive delivery release.