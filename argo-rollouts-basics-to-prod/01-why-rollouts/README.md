# Demo-01: Why Argo Rollouts — Limitations of Standard Deployments

## Overview

Before installing or writing a single rollout manifest, you need a clear
mental model of why Argo Rollouts exists. This demo covers that foundation.

There is no cluster work here. This is deliberate. Every architectural
decision in Argo Rollouts — the canary strategy, the pause step, the
analysis gate, the traffic weight — exists because standard Kubernetes
Deployments cannot do it. Understanding the limitation first makes the
solution obvious rather than arbitrary.

**What you'll learn:**
- What a standard Kubernetes rolling update actually does — step by step
- The four specific limitations that make rolling updates risky in production
- What "progressive delivery" means and why it is different from rolling updates
- What Argo Rollouts adds on top of Deployments — and what it does not change
- The four deployment strategies Argo Rollouts supports and when to use each
- How Argo Rollouts architecture differs from Argo CD — one component vs six
- Why Argo Rollouts is single-cluster only and what that means for multi-cluster setups
- How multi-tenancy differs between Argo CD (AppProjects) and Argo Rollouts (RBAC + namespace scope)
- Who accesses Argo Rollouts and how — compared to Argo CD's access model
- What additional features exist in Argo Rollouts beyond the core series
- The full series roadmap — what each demo covers

---

## Prerequisites

No cluster work in this demo. Nothing to install or configure.

To follow the hands-on demos starting from Demo-02 you will need:
- Minikube installed
- `kubectl` installed and configured
- A terminal and browser

If you are following this series in order, read this demo first, then
install everything in Demo-02 before proceeding to Demo-03.

---

## Concepts

### What a Standard Rolling Update Actually Does

When you change the image tag in a Deployment and run `kubectl apply`, here
is exactly what Kubernetes does:

```
Before update:
  ReplicaSet v1: 5 pods running image:v1 (stable)

kubectl apply → image:v2

Step 1: Create new ReplicaSet v2 with 0 replicas
Step 2: Scale v2 up by maxSurge (default 25% = 1 pod)
        → 1 pod running image:v2 starts, readiness probe runs
        → when v2 pod passes readiness → added to Service endpoints
        → v2 pod begins receiving live traffic
Step 3: Scale v1 down by maxUnavailable (default 25% = 1 pod)
        → 1 pod from v1 is terminated
        → v1 now has 4 pods, v2 has 1 pod
Step 4: Repeat until v2 has 5 pods and v1 has 0 pods
Step 5: v1 ReplicaSet kept at 0 replicas (for rollback history)

Total time: as fast as Kubernetes can create and pass readiness probes
```

This process is automatic, continuous, and — critically — has no concept
of pausing to observe what the new version is actually doing in production.

---

### Limitation 1 — All-or-Nothing with No Observation Window

Once a rolling update starts, it proceeds to completion as quickly as
it can create new pods and pass readiness probes. There is no built-in
concept of:

- "Deploy 20% of the new version, then wait and watch"
- "Stop here and let a human verify the new version looks healthy"
- "Run a metric query before continuing to the next batch"

```
Rolling update timeline:

  t=0s:   Update begins
  t=30s:  20% of pods on new version (1/5) — receiving live traffic
  t=60s:  40% on new version — still no visibility into error rate
  t=90s:  60% on new version — if errors are happening, 60% of users see them
  t=120s: 80% on new version
  t=150s: 100% on new version — full blast radius, no way back without another update

  At no point did anyone check: is the new version actually working?
```

In a production environment with thousands of requests per second, a bug
in the new version at t=60s means 40% of users are already hitting it —
and the deployment is still rolling forward with no automatic stop.

---

### Limitation 2 — Readiness Probes Are Not Business Logic

The rolling update's only safety check is the readiness probe. A pod
passes readiness and immediately starts receiving production traffic.

The problem is that readiness probes are binary process checks, not
application correctness checks:

```
What a readiness probe checks:
  httpGet: /health → 200 OK

What a readiness probe does NOT check:
  - Is the database query returning correct results?
  - Is the payment processing endpoint working?
  - Is the error rate above acceptable threshold?
  - Is p99 latency within SLA?
  - Did this version introduce a regression in a specific user flow?
```

A real example: you deploy a new version that has a bug in the database
query for the goals list endpoint. The `/health` endpoint returns 200 —
it is just a simple liveness check. The pod passes readiness. Live traffic
flows to it. Every user who loads their goals list gets an error. The
rolling update continues to completion because, as far as Kubernetes is
concerned, all pods are healthy.

```
Deployment health check:           What production actually needs:
  Pod readiness: 200 OK ✅          Error rate < 1% on /goals ✅
                                    p99 latency < 200ms ✅
                                    Database queries returning data ✅
                                    No 5xx responses in past 5 minutes ✅
```

Deployments have no mechanism to express or check the second column.

---

### Limitation 3 — No Real Traffic Control

As a rolling update proceeds, the percentage of traffic going to the new
version is determined entirely by the ratio of new pods to total pods.
Kubernetes Service load balancing is round-robin across all healthy pods
in the endpoint list — it has no concept of weights.

```
5 total pods during a rolling update:
  1 new pod  = ~20% of traffic to new version  (1/5)
  2 new pods = ~40% of traffic to new version  (2/5)
  3 new pods = ~60% of traffic to new version  (3/5)

To send only 5% of traffic to the new version:
  → you need 20 total pods (1 new out of 20)
  → at 3 replicas the minimum granularity is 33% (1 out of 3)
  → at 5 replicas the minimum is 20% (1 out of 5)

You cannot say: "send 5% to the new version, keep 3 pods total."
Pod count and traffic percentage are the same number.
```

**Why this matters in production:**

In a real production system, you want to expose a new version to a small,
controlled slice of users first — 1%, 5%, 10% — to observe its behaviour
before increasing exposure. With a Deployment, achieving 1% canary traffic
requires running 100 pods. Most teams cannot or will not do that, so they
skip fine-grained canary entirely and go straight to 20–50% — which is
a much larger blast radius if something goes wrong.

The deeper problem is that traffic percentage and pod count are coupled.
As you scale the Deployment for load (more pods under high traffic), the
traffic split changes. You cannot independently control "how many pods run"
and "what percentage of traffic goes to the new version." They are the same
knob.

```
Team running 10 pods, wants 10% canary:
  → 1 canary pod + 9 stable pods = 10% ✓

Traffic spikes, team scales to 20 pods for capacity:
  → still 1 canary pod + 19 stable pods = 5% (not 10%)
  → or scales canary to 2 pods = 10% again
  → now manually tracking: "always keep 10% of pods on new version"
  → this breaks the moment auto-scaling triggers

With Argo Rollouts + a traffic router:
  → traffic split is configured in the ingress/mesh layer: exactly 10%
  → pod count scales independently for capacity
  → traffic percentage stays at 10% regardless of how many pods run
```

The only way to achieve real percentage-based traffic control with a
standard Deployment is to manually configure weights in an ingress
controller or service mesh — separately, outside of the Deployment
rollout mechanism, with no coordination between the two systems.
Any pod scale event invalidates the manually configured weights.
Argo Rollouts coordinates traffic weight and pod management together
as a single controlled operation (covered in Demo-11).

---

### Limitation 4 — Rollback Is Just Another Rolling Update

When you trigger a rollback on a Deployment:

```
kubectl rollout undo deployment/goals-backend
```

Kubernetes does not do anything special. It starts another rolling update,
this time replacing the new pods with the previous revision's pods. This
means:

```
Rollback problems:

1. Speed: same rolling update speed — pods are replaced gradually,
   not immediately. Users continue hitting the broken version during rollback.

2. Previous version: "undo" goes to the immediately previous revision.
   If you deployed v3 (broken) on top of v2 (also broken), rolling back
   lands on v2, which is still broken.
     → you must specify --to-revision=N to go further back
     → finding the right revision number under production pressure is stressful

3. Cold-start on rollback: Kubernetes keeps the previous ReplicaSet at
   0 replicas (via `revisionHistoryLimit`), but the pods are not running.
   When you roll back, Kubernetes scales the old ReplicaSet back up —
   each pod goes through the full container pull, init, and startup
   sequence again. For an application with a 30-second startup time,
   rolling back 10 pods means the last pod is not serving traffic for
   at least 5 minutes. There is no "warm standby" — you are starting
   from zero every time. Blue-green deployments solve this by keeping
   the old version's pods actually running until the new version is
   confirmed stable (covered in Demo-04).

4. History limit: revisionHistoryLimit defaults to 10. Beyond that, older
   ReplicaSets are garbage collected. You cannot roll back to them.
```

---

### What Progressive Delivery Means

Progressive delivery is a deployment model where a new version is released
to a controlled subset of users or traffic first, observed against defined
success criteria, and then either promoted to full traffic or automatically
rolled back — all without human intervention for the happy path.

```
Progressive delivery vs rolling update:

Rolling update:
  v1 (100%) → v2 gradually replaces v1 → v2 (100%)
  No observation. No gate. No control.

Progressive delivery:
  v1 (100%)
    → v2 receives 5% of traffic (canary)
    → metrics observed for 10 minutes
    → error rate < 1%? latency OK? → promote
    → v2 receives 25% → observe again
    → v2 receives 50% → observe again
    → v2 receives 100% → v1 scaled down
    OR
    → error rate > 1% at any step → automatic rollback → v1 stays at 100%
```

The key properties of progressive delivery:
- **Controlled blast radius** — only a fraction of users see the new version initially
- **Observable** — metrics define what "good" looks like before the next step
- **Automated** — the promotion and rollback decisions are driven by data, not humans
- **Reversible at every step** — if something goes wrong at 25%, rolling back affects only 25% of users

---

### What Argo Rollouts Is

Argo Rollouts is a Kubernetes controller and set of CRDs that adds
progressive delivery capabilities to Kubernetes. It does this by
introducing a new resource type — the `Rollout` — that replaces the
standard `Deployment` for workloads that need advanced release control.

```
Standard Kubernetes:
  Deployment → manages ReplicaSets → manages Pods

Argo Rollouts:
  Rollout → manages ReplicaSets → manages Pods
  (same underlying mechanics — different release controller on top)
```

**What Argo Rollouts adds:**
- Canary strategy with configurable steps (`setWeight`, `pause`, `analysis`)
- Blue-green strategy with `activeService` and `previewService`
- Analysis integration — query Prometheus, run Jobs, call webhooks to
  determine whether a rollout should proceed or abort
- Traffic management integration — work with ingress controllers and
  service meshes for true percentage-based traffic splitting
- CLI and dashboard for visualising and managing rollout progress

**What Argo Rollouts does NOT change:**
- Pod spec — same fields, same probes, same resources, same volumes
- ReplicaSets — still used under the hood, managed automatically
- Services — standard Kubernetes Services, Rollouts just manages their selectors
- Kubernetes RBAC — same permissions model
- Container images — any image that works in a Deployment works in a Rollout

**Argo Rollouts does not require Argo CD.** It is a standalone controller.
You can use it with any CD tool or directly with `kubectl`. The two work
exceptionally well together (covered in proj-02) but neither requires the other.

---

### The Four Rollout Strategies

**1. Rolling Update (default, same as Deployment)**

Argo Rollouts supports the same rolling update behaviour as a standard
Deployment — same `maxSurge`, same `maxUnavailable`, same gradual pod
replacement. Functionally, the release process does not change at all.

**Why use a Rollout with rolling update instead of a plain Deployment?**

The answer is the surrounding tooling you get for free by switching to
the Rollout CRD:

- **CLI dashboard** — `kubectl argo rollouts get rollout <n> --watch`
  gives a live tree view of ReplicaSets, pods, and status that plain
  `kubectl rollout status` does not provide
- **Web UI** — visual rollout progress, pod status per revision, action
  buttons accessible to non-kubectl users
- **Analysis hooks** — you can add automated metric checks to a rolling
  update without changing the rollout strategy, giving you automated
  rollback even on a basic rolling deployment
- **Migration on-ramp** — the Rollout CRD accepts the same pod spec as
  a Deployment. Converting is two field changes. You get all the tooling
  immediately and can add canary or blue-green strategy later without
  touching the pod spec

```
Use when: migrating existing Deployments to Rollouts incrementally.
          Convert first, gain visibility and analysis capability,
          then layer in canary or blue-green strategy when ready.
          Teams that want analysis-driven rollback without changing
          their release process also use this pattern.
```

**2. Canary**

The new version receives a small percentage of traffic first. The rollout
proceeds through a defined sequence of steps — each step either sets a
traffic weight, pauses for a duration, pauses for manual approval, or
runs an analysis. If any analysis fails, the rollout aborts and traffic
returns to the stable version.

```
Example canary steps:
  - setWeight: 10    → 10% of traffic to new version
  - pause: {duration: 5m}   → wait 5 minutes, observe metrics
  - analysis: templates: [{templateName: success-rate}]
  - setWeight: 50    → 50% of traffic
  - pause: {}        → indefinite pause — wait for human approval
  - setWeight: 100   → full promotion

Use when: you want gradual traffic shifting with metric-based gates.
          Most production canary deployments use this strategy.
```

**3. Blue-Green**

Two complete environments run simultaneously. The stable version (`active`)
receives all production traffic. The new version (`preview`) receives no
production traffic but is fully deployed and can be tested. When ready,
a single promotion switches all traffic from active to preview instantly.

```
                     ┌─────────────────────┐
Production traffic → │  Active Service      │ → stable pods (blue)
                     └─────────────────────┘
No production traffic │  Preview Service     │ → new pods (green)
(testing only)        └─────────────────────┘

On promotion:
  Active Service selector → switches to green pods (instant cutover)
  Green is now production. Blue is kept for fast rollback.

Use when: you need zero downtime instant cutover, or want to run
          full integration tests on the new version before any
          production traffic touches it.
```

**4. Experiment**

An Experiment runs multiple versions of an application simultaneously
for a defined period and compares their behaviour against each other.
Unlike canary (which gradually shifts traffic from old to new) and
blue-green (which switches all traffic at once), an Experiment keeps
multiple versions alive side by side for the duration of the test,
then tears them all down. It is designed for A/B testing and hypothesis
validation — "does version B perform better than version A for users in
a specific region?" — rather than for a standard production release.

The Experiment CRD is an advanced pattern that requires metric
infrastructure and careful design to be useful. It is not covered in
this series because canary with analysis (Demo-09, Demo-10) covers the
vast majority of production progressive delivery needs. Experiments are
worth knowing about when your team moves beyond "is the new version
healthy?" to "is the new version measurably better than the current one?"

```
Use when: you want to validate a hypothesis about two versions under
          real production traffic before committing to either one.
          Requires: metric provider, defined success criteria, and a
          clear question the experiment is designed to answer.
```

---

### Argo Rollouts in the Argo Ecosystem

The Argo project consists of four tools that each solve a distinct problem
in the Kubernetes delivery pipeline. They are designed to work together
but are completely independent — you can use any one without the others.

```
Argo CD
  What: GitOps continuous delivery controller
  Job:  Watches Git repositories. When a manifest changes in Git,
        syncs the change to the cluster. Ensures cluster always matches
        Git. Detects and corrects drift.
  Does NOT: build images, run tests, manage deployment strategy

Argo Rollouts  ← this series
  What: Progressive delivery controller
  Job:  Manages how a new version is released. Controls traffic split,
        runs analysis, promotes or aborts based on metrics.
  Does NOT: manage Git sync, build images, orchestrate pipelines

Argo Workflows
  What: Kubernetes-native workflow engine (CI pipelines)
  Job:  Runs multi-step DAG pipelines as Kubernetes pods. Used for
        building images, running tests, scanning, publishing artifacts.
        Each step is a container — no external CI server needed.
  Does NOT: manage deployments, sync Git, control traffic

Argo Events
  What: Event-driven automation trigger system
  Job:  Listens for events (Git push, S3 upload, message queue, webhook,
        cron schedule) and triggers Argo Workflows or other actions in
        response. The "if this happens, do that" layer of the pipeline.
  Does NOT: run pipeline steps, manage deployments
```

**How all four work together in a complete GitOps pipeline:**

```
Developer pushes code to Git
  │
  ▼
Argo Events detects the Git push event
  → triggers an Argo Workflow
  │
  ▼
Argo Workflows runs the CI pipeline
  → build Docker image
  → run unit tests and security scan
  → push image to registry
  → update image tag in the config repo (Git commit)
  │
  ▼
Argo CD detects the image tag change in the config repo
  → syncs the updated Rollout manifest to the cluster
  │
  ▼
Argo Rollouts detects the Rollout spec change
  → begins progressive delivery (canary/blue-green)
  → runs analysis against metrics
  → promotes if healthy, aborts if not
  │
  ▼
Argo CD marks the Application as Healthy
  → pipeline complete
```

Each tool has one job and hands off to the next. None of them overlap.

The integration between Argo CD and Argo Rollouts is covered in proj-02.
For demos 01–11, Argo Rollouts runs standalone — no Argo CD required.

---

### The Rollout Custom Resource — First Look

The Rollout CRD is intentionally designed to be as close to a standard
Deployment as possible. The migration path is:

```yaml
# Standard Deployment (before)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goals-backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: goals-backend
  template:
    metadata:
      labels:
        app: goals-backend
    spec:
      containers:
        - name: backend
          image: rselvantech/goals-backend:v2.0.0
          ports:
            - containerPort: 80
```

```yaml
# Argo Rollout (after) — two fields changed, strategy added
apiVersion: argoproj.io/v1alpha1   # ← changed
kind: Rollout                       # ← changed
metadata:
  name: goals-backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: goals-backend
  template:                         # ← identical to Deployment
    metadata:
      labels:
        app: goals-backend
    spec:
      containers:
        - name: backend
          image: rselvantech/goals-backend:v2.0.0
          ports:
            - containerPort: 80
  strategy:                         # ← new: required, no default
    canary:
      steps:
        - setWeight: 20
        - pause: {}
```

The `strategy` field is **required**. A Rollout without a strategy field
is accepted by `kubectl apply` (no client-side validation error) but
immediately enters a `Degraded` state in the cluster. This is a common
first mistake — it is covered and demonstrated in Demo-03.

---

### Argo Rollouts Architecture — One Controller, One Job

Argo Rollouts has a deliberately simple architecture: a single controller
pod that does everything. This is fundamentally different from Argo CD,
which is composed of three core components with separate responsibilities.

Understanding this difference matters because it explains why Argo Rollouts
is easier to install, has no internal communication to debug, but also has
a hard limit on multi-cluster capability (covered in the next section).

```
Argo CD — three core components, each with one job:
  ┌──────────────┐   ┌───────────────────┐   ┌────────────────────┐
  │  API Server  │   │ Repository Server │   │ App Controller     │
  │              │   │                   │   │                    │
  │ Entry point  │   │ Fetches desired   │   │ Compares desired   │
  │ for UI, CLI  │   │ state from Git    │   │ vs live state      │
  │ and REST API │   │ Renders manifests │   │ Reconciles drift   │
  └──────────────┘   └───────────────────┘   └────────────────────┘
  Plus three supporting components: Redis, ApplicationSet Controller,
  Dex Server

Argo Rollouts — one component:
  ┌──────────────────────────────────────────────────────┐
  │  argo-rollouts controller pod                        │
  │                                                      │
  │  Watches Rollout resources cluster-wide              │
  │  Manages ReplicaSets (stable + canary/preview)       │
  │  Runs AnalysisRuns (queries metrics, executes Jobs)  │
  │  Updates Service selectors and ingress weights       │
  │  Promotes or aborts based on analysis results        │
  └──────────────────────────────────────────────────────┘
  No Git. No caching layer. No SSO server. No separate API server.
```

**What this means in practice:**

- **Install**: one `helm install` or one `kubectl apply` — nothing else
- **Debugging**: one pod's logs contain everything — no need to check
  which of three components handled a request
- **Dependencies**: none — no Redis, no Dex, no external Git credentials
  at the controller level
- **Scope**: operates entirely within the cluster it is installed in —
  it cannot reach out to external clusters (see next section)

**What the controller does NOT do** (sourced from official docs at
`argoproj.github.io/argo-rollouts/architecture`):

> "Note that Argo Rollouts will not tamper with or respond to any changes
> that happen on normal Deployment Resources."

The controller only watches resources of type `Rollout`. Standard
Deployments in the same cluster are completely unaffected. This means
you can install Argo Rollouts in a cluster that already uses standard
Deployments — the two coexist without interference.

---

### Multi-Cluster — A Fundamental Difference from Argo CD

This is one of the most important architectural differences between Argo
CD and Argo Rollouts, and it is confirmed explicitly in the official
Argo Rollouts documentation.

**Argo CD — multi-cluster by design:**

One Argo CD installation can manage Rollout and Deployment resources
across many Kubernetes clusters. Each cluster is registered with its
credentials. An Application CRD specifies `destination.server` pointing
to whichever cluster it targets.

```
Argo CD (installed in cluster-hub)
  ├── manages cluster-hub (local)       destination.server: https://kubernetes.default.svc
  ├── manages cluster-dev (remote)      destination.server: https://dev-cluster-api:6443
  ├── manages cluster-staging (remote)  destination.server: https://staging-cluster-api:6443
  └── manages cluster-prod (remote)     destination.server: https://prod-cluster-api:6443
```

**Argo Rollouts — single-cluster only:**

From the official Best Practices page
(`argoproj.github.io/argo-rollouts/best-practices`):

> "Currently, Argo Rollouts works with a single Kubernetes
> deployment/application and within a single cluster only. You also need
> to have the controller deployed on every cluster where a Rollout is
> running if you have more than one cluster using Rollout workloads."

And from the official FAQ
(`argoproj.github.io/argo-rollouts/FAQ`):

> "Can we install Argo Rollouts centrally in a cluster and manage Rollout
> resources in external clusters? No."

The controller only watches and manages resources in the cluster it is
deployed in. It uses the in-cluster Kubernetes service account — it has
no mechanism to reach external cluster APIs.

```
Multi-cluster setup with Argo Rollouts:

  cluster-dev:
    └── argo-rollouts controller    ← manages Rollouts in cluster-dev only

  cluster-staging:
    └── argo-rollouts controller    ← manages Rollouts in cluster-staging only

  cluster-prod:
    └── argo-rollouts controller    ← manages Rollouts in cluster-prod only

  Each cluster needs its own controller installation.
  There is no central control plane.
```

**What this means for your setup:**

In this series, all demos run on a single local Minikube cluster, so this
limitation does not affect anything you will do here. When you move to EKS
or multi-cluster production environments, each cluster needs its own Argo
Rollouts install. You can still use one Argo CD to manage all of them —
Argo CD manages the Rollout manifests and the Argo Rollouts controller in
each cluster manages the actual progressive delivery locally.

```
Production multi-cluster pattern (Argo CD + Argo Rollouts):

  Argo CD (hub cluster)
    ├── syncs Rollout manifests → cluster-dev
    │     └── argo-rollouts controller watches → executes canary locally
    ├── syncs Rollout manifests → cluster-staging
    │     └── argo-rollouts controller watches → executes canary locally
    └── syncs Rollout manifests → cluster-prod
          └── argo-rollouts controller watches → executes canary locally

  One Argo CD controls what is deployed to each cluster.
  Each cluster's Argo Rollouts controls how it is deployed.
```

---

### Multi-Tenancy — How Argo Rollouts and Argo CD Differ

Both tools support multi-tenancy but achieve it through completely
different mechanisms.

**Argo CD multi-tenancy — AppProjects:**

Argo CD has a dedicated governance resource called `AppProject` that
controls what each team can do:
- Which Git repositories they can deploy from
- Which destination clusters and namespaces they can target
- Which Kubernetes resource types they are allowed to sync
- Whether they can create Applications outside their permitted scope

One Argo CD installation serves many teams. Teams are isolated from each
other via AppProject rules enforced by the Argo CD API server. A team
cannot accidentally (or maliciously) deploy to another team's namespace
even if they have access to the Argo CD UI.

**Argo Rollouts multi-tenancy — Kubernetes RBAC + namespace scope:**

Argo Rollouts has no AppProject equivalent. There is no Argo
Rollouts-specific governance layer. Multi-tenancy is achieved through
two mechanisms:

*Mechanism 1 — Kubernetes RBAC:*
Who can create, modify, or delete Rollout objects in which namespace is
controlled entirely by standard Kubernetes Roles and RoleBindings. If
Team A has a Role allowing `create/update/delete` on `rollouts` in
`namespace-a`, they can manage rollouts there. They have no access to
`namespace-b` unless explicitly granted. This is pure Kubernetes — Argo
Rollouts adds nothing on top.

*Mechanism 2 — Namespace-scoped controller:*
As covered in the Demo-02 install section, Argo Rollouts supports a
namespace-scoped install where each team runs their own controller
instance scoped to their namespace. One team's controller cannot see or
interfere with another team's rollouts. This is the recommended pattern
for shared clusters where teams do not trust each other's releases.

```
Argo CD multi-tenancy:
  One Argo CD instance
    Team A → AppProject-A → permitted repos, clusters, namespaces, resources
    Team B → AppProject-B → permitted repos, clusters, namespaces, resources
  Central governance. One control plane. Teams isolated by policy.

Argo Rollouts multi-tenancy:
  Option 1 — Shared controller + Kubernetes RBAC:
    One controller (cluster-wide)
    Team A → RBAC Role → can manage Rollouts in namespace-a only
    Team B → RBAC Role → can manage Rollouts in namespace-b only
    Teams isolated by Kubernetes permissions. No Rollouts-level policy.

  Option 2 — Namespace-scoped controllers:
    Team A installs argo-rollouts scoped to namespace-a
    Team B installs argo-rollouts scoped to namespace-b
    Each team's controller sees only its namespace.
    Platform team installs CRDs once cluster-wide.
    Teams isolated by controller scope + RBAC.
```

**ClusterAnalysisTemplate — the one shared resource:**

Argo Rollouts does have one cross-namespace sharing mechanism:
`ClusterAnalysisTemplate`. A platform team can define analysis success
criteria once cluster-wide (e.g. "success rate must be above 95%"),
and every team's Rollout in any namespace can reference it without
copying the template into their own namespace.

```yaml
# ClusterAnalysisTemplate — defined once by platform team, used by all
apiVersion: argoproj.io/v1alpha1
kind: ClusterAnalysisTemplate
metadata:
  name: company-success-rate-threshold
spec:
  metrics:
    - name: success-rate
      successCondition: result[0] >= 0.95
      provider:
        prometheus:
          query: |
            sum(rate(http_requests_total{status=~"2.."}[5m]))
            / sum(rate(http_requests_total[5m]))

# Team A Rollout — references the cluster-wide template
strategy:
  canary:
    analysis:
      templates:
        - templateName: company-success-rate-threshold
          clusterScope: true   ← tells Argo Rollouts to look cluster-wide
```

This is the closest Argo Rollouts gets to the shared policy model that
AppProjects provide in Argo CD — but it is limited to analysis definitions
only, not to deployment permissions or target restrictions.

---

### Who Accesses Argo Rollouts — The Access Model

The access model for Argo Rollouts is simpler than Argo CD's because
Argo Rollouts has no separate API server, no UI login, and no SSO layer.
Everything goes through the standard Kubernetes API using your existing
kubeconfig credentials.

```
Argo CD access model:
  Developer → Argo CD UI / argocd CLI → Argo CD API Server → cluster
  Requires: Argo CD login (local account or SSO via Dex)
  Has: Argo CD RBAC (separate from Kubernetes RBAC)
       AppProject restrictions
       UI-level permissions (who can sync, who can delete)

Argo Rollouts access model:
  Developer → kubectl / kubectl argo rollouts plugin → Kubernetes API → cluster
  Requires: valid kubeconfig with Kubernetes RBAC permissions
  Has: standard Kubernetes RBAC only
       No separate Argo Rollouts login
       No Argo Rollouts-specific permission layer
```

**What this means in practice:**

| Action | What is required |
|---|---|
| View a rollout's status | `kubectl get rollout` permission in that namespace |
| Promote a paused rollout | `kubectl patch rollout` permission — the plugin patches the Rollout object |
| Abort a rollout | `kubectl patch rollout` permission |
| Create a new Rollout | `kubectl create rollout` permission |
| Access the web UI dashboard | Network access to the dashboard Service — no login required |

**The dashboard has no authentication.** The Argo Rollouts dashboard
serves its UI without a login screen. Access is controlled entirely by
whether you can reach the Service (via port-forward or network policy).
In production, access to the dashboard port should be restricted at the
network level.

> **Argo CD comparison:**
> Argo CD has its own authentication layer — you log in with a username
> and password or SSO. Inside Argo CD, RBAC controls what each user can
> do (sync, delete, override). Argo Rollouts has none of this — if you
> can reach the Kubernetes API with permissions to patch a Rollout, you
> can promote or abort it, regardless of who you are. Security is
> entirely delegated to Kubernetes RBAC and network access controls.

---

### Additional Features — Worth Being Aware Of

These features are part of Argo Rollouts but are not covered in this
series. They are listed here so you know they exist when you encounter
them in production environments.

**Notifications**

Argo Rollouts supports the same notification framework used by Argo CD —
you can configure alerts to Slack, PagerDuty, email, or any webhook when
a rollout is paused, promoted, aborted, or degraded. Notifications are
configured via a `ConfigMap` in the `argo-rollouts` namespace and use
the same trigger/template model as Argo CD notifications.

Official docs: `argoproj.github.io/argo-rollouts/features/notifications`

**HPA Integration (Horizontal Pod Autoscaler)**

Argo Rollouts exposes a `/scale` subresource identical to a standard
Deployment, which means HPA can target a Rollout object directly. The
HPA scales the total replica count; Argo Rollouts then distributes that
count between the stable and canary ReplicaSets according to the
configured traffic weight. Available since v0.3.0.

Official docs: `argoproj.github.io/argo-rollouts/features/hpa-support`

**VPA Integration (Vertical Pod Autoscaler)**

Argo Rollouts supports VPA for automatic right-sizing of pod resource
requests. VPA can target Rollout objects and recommend or apply CPU/memory
changes. Available in beta.

Official docs: `argoproj.github.io/argo-rollouts/features/vpa-support`

**Workload Reference (workloadRef)**

Instead of embedding the full pod template in a Rollout spec, you can
reference an existing Deployment using `workloadRef`. Argo Rollouts reads
the pod template from the referenced Deployment and manages the release.
This is useful for migrating existing Deployments to Rollouts without
duplicating the pod spec.

Official docs: `argoproj.github.io/argo-rollouts/features/specification`

**HA Mode**

The Argo Rollouts controller supports leader-election for high
availability. Running multiple controller replicas with `--leader-elect`
ensures the controller keeps working if one pod is evicted or crashes.

Official docs: `argoproj.github.io/argo-rollouts/FAQ`
(`Can we run the Argo Rollouts controller in HA mode?`)

---

### What Is Still Ahead — Series Roadmap

| Demo | Topic | What you will do |
|---|---|---|
| Demo-01 | Why Argo Rollouts | This demo — concepts only |
| Demo-02 | Installing Argo Rollouts | Install controller (Helm + raw manifest), plugin, dashboard |
| Demo-03 | Canary Basics | First Rollout, canary strategy, promote and abort |
| Demo-04 | Blue-Green Basics | Blue-green strategy, active/preview Services, instant cutover |
| Demo-05 | Canary + K8s Features | Probes, native sidecar, PDB, topologySpreadConstraints, preStop |
| Demo-06 | Blue-Green + K8s Features | Same K8s features applied to blue-green strategy |
| Demo-07 | Analysis — Job-based, Canary | AnalysisTemplate using Kubernetes Job, automated promote/abort |
| Demo-08 | Analysis — Job-based, Blue-Green | Pre-promotion analysis on preview service |
| Demo-09 | Analysis — Prometheus, Canary | kube-prometheus-stack, Goals App backend metrics, Prometheus analysis |
| Demo-10 | Analysis — Prometheus, Blue-Green | Prometheus pre-promotion analysis on Goals App |
| Demo-11 | Traffic Management | Traefik + Kubernetes Gateway API + Gateway API plugin |
| proj-02 | Argo CD + Argo Rollouts | ArgoCD managing a Rollout, health states, resource actions, GitOps rollback |

**Additional concepts introduced within demos as they become relevant:**

| Concept | Introduced in |
|---|---|
| `ClusterAnalysisTemplate` — shared analysis across namespaces | Demo-07 |
| `setCanaryScale` — decouple canary pod count from traffic weight | Demo-11 |
| ArgoCD `Suspended` state for paused Rollouts | proj-02 |
| ArgoCD resource actions (`resume`, `restart`) on Rollouts | proj-02 |
| GitOps rollback — `git revert` vs `kubectl argo rollouts undo` | proj-02 |
| `PruneLast=true` sync option for AnalysisTemplate safety | proj-02 |


## Key Concepts Summary

**Standard Deployments have four production limitations**
All-or-nothing speed with no observation window. Readiness probes are
process checks, not business logic. Traffic control is pod-count-based,
not percentage-based. Rollback is another slow rolling update.

**Progressive delivery solves all four**
Controlled blast radius. Metric-based gates. Percentage-based traffic
control. Instant abort with stable version preserved.

**Argo Rollouts implements progressive delivery without rewriting your manifests**
Change `apiVersion` and `kind`. Add `strategy`. Everything else — pod spec,
probes, resources, volumes — is identical to what you already have.

**Two strategies cover almost all production use cases**
Canary for gradual traffic shifting with automated analysis. Blue-green
for instant cutover with pre-production verification.

**Argo Rollouts is standalone**
It does not require Argo CD, a service mesh, or an ingress controller.
Traffic management integration is optional and progressive (covered in
Demo-11). Analysis is optional (covered in Demos 07–10).

**Argo Rollouts is single-cluster only**
Unlike Argo CD which manages multiple clusters from one installation,
Argo Rollouts works within a single cluster only. Each cluster where
Rollout workloads run needs its own controller. This is confirmed in
the official docs and FAQ.

**Multi-tenancy uses Kubernetes RBAC — not a dedicated governance layer**
Argo CD uses AppProjects to isolate teams. Argo Rollouts has no
equivalent — team isolation is achieved through Kubernetes RBAC and
optionally namespace-scoped controller installs. The one shared resource
across namespaces is `ClusterAnalysisTemplate`.

**No separate access control or login**
Argo Rollouts has no authentication layer. Access is pure Kubernetes
RBAC via kubeconfig. The dashboard has no login screen. In production,
access is controlled at the network and RBAC level.

---

## What's Next

**Demo-02: Installing Argo Rollouts**
Install the Argo Rollouts controller on a local Minikube cluster using
the official stable manifest. Install the `kubectl argo rollouts` plugin.
Access the Argo Rollouts dashboard via both the web UI and the CLI.
Verify all components are running before writing the first Rollout manifest.