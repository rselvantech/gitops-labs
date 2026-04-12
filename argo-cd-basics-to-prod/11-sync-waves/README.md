# Demo-11: Sync Waves — Controlling Resource Creation Order

## Overview

Demo-10 introduced sync hooks — lifecycle control that runs logic before,
during, and after sync. This demo builds on that foundation and solves a
different but related problem: **ordering**.

When ArgoCD syncs an application with many resources, it applies them in a
deterministic sequence based on resource kind and name. This default ordering
handles explicit dependencies well — ConfigMaps before Deployments, Namespaces
before namespaced resources. But it ignores **implicit dependencies** — things
like NetworkPolicy, ResourceQuota, and LimitRange that your Deployment relies
on but does not reference by name in its YAML.

Sync waves solve this by letting you assign an integer wave number to any
resource. ArgoCD applies all resources in wave N completely before moving to
wave N+1. Lower numbers are applied first. Negative numbers are valid.

In this demo we deploy an nginx application with a realistic set of supporting
resources and use sync waves to enforce a production-grade creation order —
ensuring governance resources (quota, network policy) are in place before any
workload pod starts.

**What you'll learn:**
- How ArgoCD orders resources by default — and why that is often not enough
- The difference between explicit and implicit dependencies
- What sync waves are and how they work within a sync lifecycle
- How wave numbering works — lower first, gaps are intentional
- How sync waves relate to sync phases (hooks always take precedence)
- Why leaving gaps in wave numbers is a production best practice
- The 2-second delay between waves and why it exists
- How pruning reverses wave order — highest wave deleted first
- What happens when a wave resource never becomes Healthy
- How App-of-Apps enables cross-application ordering via waves
- How to verify wave-based ordering in the ArgoCD UI and CLI

**What you'll do:**
- Deploy an nginx application with Namespace, ResourceQuota, LimitRange,
  ServiceAccount, Secret, ConfigMap, NetworkPolicy, Service and Deployment
- Assign sync wave annotations to control creation order
- Observe ArgoCD applying resources wave by wave in the UI
- Prove that NetworkPolicy and ResourceQuota exist before the Deployment pod starts
- Add a new resource mid-sequence without renumbering existing waves

---

## Prerequisites

- ✅ Completed Demo-10 — sync hooks and lifecycle phases understood
- ✅ ArgoCD running on minikube
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `gitops-apps-config` and `argocd-config`
- ✅ `gitops-apps-config` already registered with ArgoCD from Demo-10

**Verify Prerequisites:**

### 1. ArgoCD pods are running
```bash
kubectl get pods -n argocd
```

**Expected:** All pods `Running` and `1/1` Ready.

### 2. ArgoCD UI is accessible
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `http://localhost:8080` — you should see the ArgoCD login page.

### 3. ArgoCD CLI is installed and logged in
```bash
argocd version --client
argocd login localhost:8080 --username admin --insecure
```

**Expected:**
```text
argocd: v3.x.x
'admin:login' logged in successfully
```

### 4. No leftover namespace from a previous run
```bash
kubectl get namespace sync-waves-demo 2>&1
```

**Expected:** `Error from server (NotFound)` — namespace does not exist.

If it exists from a previous run:
```bash
kubectl delete namespace sync-waves-demo
```

---

## Concepts

### How kubectl Applies Resources — No Ordering Intelligence

When you run `kubectl apply -f manifests/`, kubectl applies resources in the
order they appear — either in the file or by filename if split across multiple
files. It has no awareness of logical dependencies.

```bash
# If deployment.yaml comes before namespace.yaml alphabetically:
kubectl apply -f manifests/
# → tries to create Deployment first
# → fails: namespace "sync-waves-demo" not found
# → then creates Namespace
# → Deployment was never created
```

The only workaround is manual file ordering or naming conventions (01-namespace,
02-deployment). This does not scale.

---

### How ArgoCD Applies Resources — Default Ordering

ArgoCD does better. When a sync starts, ArgoCD evaluates all manifests upfront
and computes a deterministic execution plan. It orders resources by:

```
1. Phase       (PreSync hooks first, then Sync phase, then PostSync)
2. Sync Wave   (lower wave number first — covered in this demo)
3. Resource Kind (Namespace → CRD → ClusterScoped → Namespaced)
4. Resource Name (alphabetical within the same kind and wave)
```

The default kind ordering means ArgoCD creates Namespaces before Deployments,
ConfigMaps before Deployments, Secrets before Deployments — automatically.
These are **explicit dependencies** — resources that a Deployment references
by name in its YAML (`configMapRef`, `secretRef`, `serviceAccountName`).

ArgoCD enforces explicit dependencies without any annotation from you.

---

### Explicit vs Implicit Dependencies

**Explicit dependency** — defined in the manifest YAML:
```yaml
# The Deployment explicitly references these — ArgoCD handles ordering
spec:
  serviceAccountName: app-service-account    # explicit
  containers:
    - envFrom:
        - configMapRef:
            name: app-config                  # explicit
        - secretRef:
            name: app-secret                  # explicit
```
ArgoCD reads these references and ensures ServiceAccount, ConfigMap, and Secret
exist before applying the Deployment. No wave annotation needed.

**Implicit dependency** — NOT referenced in the manifest but still required:
```yaml
# NetworkPolicy — Deployment has no field referencing it
# but pods must not start without it (security requirement)
# ResourceQuota — Deployment has no field referencing it
# but quota must exist before pods start (governance requirement)
# LimitRange — Deployment has no field referencing it
# but default limits must be set before pods start
```
ArgoCD has no way to know about these. It may apply the Deployment before
the NetworkPolicy — leaving pods briefly unprotected. Sync waves solve this.

---

### What Sync Waves Are

A sync wave is an integer annotation on a Kubernetes resource that tells
ArgoCD when to apply it relative to other resources in the same sync.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"    # apply in wave 0
```

**Rules:**
- Lower number = applied first
- Negative numbers are valid (applied before wave 0)
- Resources with no annotation default to wave 0
- All resources in wave N must be `Healthy` before wave N+1 begins
- Wave ordering operates within a single Application — not across Applications

> **Watch for unannotated resources defaulting to wave 0:**
> Any resource without a sync-wave annotation is treated as wave 0.
> If you annotate your ConfigMap at wave 0 but leave your Service
> unannotated, both will be in wave 0 and applied together — which may
> not be what you intended. Always annotate every resource explicitly
> when using sync waves to avoid unexpected ordering from defaults.

**Wave numbering — leave gaps intentionally:**
```
Wave -1  →  Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret
Wave  0  →  ConfigMap, NetworkPolicy
Wave  1  →  Service
Wave  5  →  Deployment
```
The gap between wave 1 and wave 5 is intentional. If you need to insert a
new resource between Service and Deployment later (e.g. a PodDisruptionBudget),
you assign it wave 3 without renumbering anything.

---

### Sync Waves vs Sync Phases (Hooks)

These are often confused. The key distinction:

| | Sync Phases (Hooks) | Sync Waves |
|---|---|---|
| What they control | When to run one-off logic (PreSync, PostSync) | Order of regular manifest application |
| Resource types | Jobs (most common) | Any Kubernetes resource |
| Scope | Runs once, then deleted | Reconciled continuously |
| Precedence | Always higher than waves | Applied within the Sync phase |

**Phases always take precedence over waves:**
```
PreSync hooks (any wave)
  → Sync phase, wave -1
  → Sync phase, wave 0
  → Sync phase, wave 1
  → Sync phase, wave 5
PostSync hooks (any wave)
```

Within the PreSync phase, waves determine the order of any hook resources.
Within the Sync phase, waves determine the order of regular manifests.

---

### The 2-Second Delay Between Waves

ArgoCD intentionally waits **2 seconds** between completing one wave and
starting the next. This is not a bug — it is by design.

The delay gives other Kubernetes controllers (such as the endpoints controller,
admission webhooks, or external operators) time to react to resources applied
in the previous wave before ArgoCD assesses their health. Without this delay,
ArgoCD might read a stale object and declare a resource unhealthy prematurely.

```
Wave -1 applied → ArgoCD waits 2s → health check → wave 0 applied →
ArgoCD waits 2s → health check → wave 1 applied → ...
```

**In practice:** A 4-wave application adds ~8 seconds of overhead. This is
negligible. Do not create more waves than necessary just to avoid this delay —
the grouping of independent resources in the same wave is the correct pattern.

The delay is configurable via the `ARGOCD_SYNC_WAVE_DELAY` environment variable
on the ArgoCD Application Controller pod. The default is `2s`.

---

### Pruning Order Is Reversed

During pruning (when resources are removed from Git), ArgoCD processes waves
in **reverse order** — highest wave first, working down to the lowest.

```
Creation order:   wave -1 → wave 0 → wave 1 → wave 5
Pruning order:    wave  5 → wave 1 → wave 0 → wave -1
```

This ensures dependent resources are removed before their dependencies. The
Deployment is deleted before the NetworkPolicy it depends on. If a resource
fails to prune in a higher wave, ArgoCD stops — resources in lower waves are
not processed. This prevents orphaned resources from being left in an
inconsistent state.

---

### What Happens When a Wave Resource Never Becomes Healthy

ArgoCD waits for all resources in a wave to reach `Healthy` status before
starting the next wave. If a resource never becomes healthy — due to a bad
image, missing dependency, or misconfiguration — the sync times out and fails.
Subsequent waves are never started.

```
Wave 0: ConfigMap applied → Healthy ✅
Wave 1: Service applied → Healthy ✅
Wave 5: Deployment applied → pods in ImagePullBackOff → never Healthy ❌
         → sync timeout → sync fails
         → PostSync hooks never run
```

**Production implication:** Resources that legitimately take time to become
ready (e.g. PersistentVolumeClaims with `WaitForFirstConsumer` storage class)
should not be placed in a wave before the resource that will claim them. A PVC
with `WaitForFirstConsumer` only becomes `Bound` after a pod references it —
placing it alone in an early wave will cause the sync to hang waiting for a
health status that can only be reached in a later wave.

---

### Using sync waves in Helm chart templates:

When your application is a Helm chart, add the wave annotation under the
resource's `metadata.annotations` in the template file — not in `values.yaml`:
```yaml
# templates/deployment.yaml
metadata:
  name: {{ .Release.Name }}-deployment
  annotations:
    argocd.argoproj.io/sync-wave: "20"
```

ArgoCD reads the annotation after Helm renders the template to plain YAML.

---

### Sync Waves Across Multiple Applications

Sync waves only work within a single Application. You cannot use wave
annotations to order resources across two separate ArgoCD Applications.

**However, App-of-Apps enables cross-application ordering:**
When a parent Application manages child Application CRDs, you can add sync
wave annotations to the child Application CRDs themselves. ArgoCD will apply
child Application A (wave 0) and wait for it to become Healthy before applying
child Application B (wave 1).

```yaml
# In argocd-config — child Application CRDs managed by parent App-of-Apps
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: database-app
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # database deployed first
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-app
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # backend after database is Healthy
```

This is covered in Demo-12 (App-of-Apps).


---

### Why Gaps Matter in Production

```
# Bad — no gaps
Wave 0: Namespace
Wave 1: ResourceQuota
Wave 2: ServiceAccount
Wave 3: ConfigMap
Wave 4: Service
Wave 5: Deployment

# Six months later: need to add NetworkPolicy between ConfigMap and Service
# Must renumber Service (4→5) and Deployment (5→6)
# Every YAML that references these resources must also be updated

# Good — gaps
Wave -1: Namespace, ResourceQuota, ServiceAccount, Secret
Wave  0: ConfigMap, NetworkPolicy
Wave  1: Service
Wave  5: Deployment

# Six months later: need to add PodDisruptionBudget before Deployment
# Assign wave 3 — nothing else changes
```

**Use multiples of a base number — not arbitrary gaps:**

The recommendation is use multiples of 10 (or 5,or 100 depending on expected growth):
```
# Arbitrary gaps — hard to reason about
Wave -1, 0, 1, 5    ← unclear spacing

# Multiples of 10 — clear and predictable
Wave -10  →  Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret
Wave   0  →  ConfigMap, NetworkPolicy
Wave  10  →  Service
Wave  20  →  Deployment
```

With multiples of 10, you have 9 available slots between each wave for future
insertions. If you foresee many additions, use multiples of 100. The magnitude
of the number does not affect timing — only the ordering.

---

## The Application — What We Deploy

A production-realistic nginx web application serving a custom page from a
ConfigMap. The app is simple enough to focus entirely on ordering but has
the full set of supporting resources you would see in a real environment.

```
sync-waves-demo namespace
├── ResourceQuota        ← limits total resource usage in namespace
├── LimitRange           ← sets default requests/limits on pods
├── ServiceAccount       ← identity for the nginx pod
├── Secret               ← simulated API key (nginx reads it as env var)
├── ConfigMap            ← custom index.html content
├── NetworkPolicy        ← restricts pod ingress/egress
├── Service              ← ClusterIP exposing nginx on port 80
└── Deployment           ← nginx pod using all of the above
```

**Why each resource exists:**

- **ResourceQuota** — governance ceiling. Max 2 pods, defined CPU/memory limits
- **LimitRange** — default resource limits applied automatically to pods
- **ServiceAccount** — explicit identity instead of the default service account
- **Secret** — simulated API key (`DEMO_API_KEY`) injected as environment variable
- **ConfigMap** — custom `index.html` mounted as volume into nginx
- **NetworkPolicy** — restricts traffic. Pods unprotected until this is applied
- **Service** — exposes nginx ClusterIP on port 80
- **Deployment** — nginx pod using all resources above

---

## Folder Structure

```
11-sync-waves/src/
├── gitops-apps-config/              ← git init → remote: rselvantech/gitops-apps-config (new private repo)
│   └── demo-11-sync-waves-nginx-app/
│       ├── namespace.yaml
│       ├── resourcequota.yaml
│       ├── limitrange.yaml
│       ├── serviceaccount.yaml
│       ├── secret.yaml
│       ├── configmap.yaml
│       ├── networkpolicy.yaml
│       ├── service.yaml
│       ├── deployment.yaml
│       └── poddisruptionbudget.yaml    ← added in Step 6
└── argocd-config/           ← git init → remote: rselvantech/argocd-config
    └── demo-11-sync-waves/
        └── sync-waves-app.yaml
```

**What happens on GitHub after all pushes:**

```
rselvantech/gitops-apps-config (GitHub) — existing repo from Demo-10
├── demo-10-sync-hooks-goals-app/       ← Demo-10 (untouched)
└── demo-11-sync-waves-nginx-app/       ← Demo-11 adds this
    ├── namespace.yaml          ← wave -1
    ├── resourcequota.yaml      ← wave -1
    ├── limitrange.yaml         ← wave -1
    ├── serviceaccount.yaml     ← wave -1
    ├── secret.yaml             ← wave -1
    ├── configmap.yaml          ← wave  0
    ├── networkpolicy.yaml      ← wave  0
    ├── service.yaml            ← wave  1
    └── deployment.yaml         ← wave  5

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml            ← Demo-05 root (untouched)
├── demo-06-sync-pruning/       ← Demo-06 (untouched)
├── demo-09-argocd-projects/    ← Demo-09 (untouched)
├── demo-10-sync-hooks/         ← Demo-10 (untouched)
└── demo-11-sync-waves/         ← Demo-11 adds this
    └── sync-waves-app.yaml
```

> `gitops-apps-config` is already registered with ArgoCD from Demo-10. No
> new repo registration needed. `argocd-config` is applied manually with
> `kubectl apply` and does not need ArgoCD registration.

---

## Step 1: Setup — Repo Registration with ArgoCD

### Step 1a: Check `gitops-apps-config` registered

From Demo-10 onward, all application manifests are maintained in the `gitops-apps-config` repository. This repository was created and a separate GitHub Personal Access Token (PAT) was generated for it. The repository was also registered with ArgoCD as part of the setup documented in Demo-10’s `README-goals-app-setup.md`.

**Verify Repo `gitops-apps-config` registration:**
```bash
argocd repo list | grep gitops-apps-config
```

**Expected:** `gitops-apps-config` listed with `Successful` status.
```text
git   gitops-apps-config   https://github.com/rselvantech/gitops-apps-config.git   Successful
```

If repo is **not registered**, it means Demo-10's `README-goals-app-setup.md` was not completed.

**Create PAT and Register Repo Now:** 
 - Create a new PAT on GitHub and add `gitops-apps-config` with `Contents: Read-only` permission.
 - Register the repo with argocd
    ```bash
    argocd repo add https://github.com/rselvantech/gitops-apps-config.git \
      --username rselvantech \
      --password <GITHUB_PAT>
    ```
 - Verify Repo registration
    ```bash
    argocd repo list
    ```



### Step 1b: Initialise local repo structure

```bash
cd 11-sync-waves/src
mkdir gitops-apps-config && cd gitops-apps-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```
---

## Step 2: Add All Manifests with Wave Annotations

Create all files under `demo-11-sync-waves-nginx-app/`. The `argocd.argoproj.io/sync-wave`
annotation is the only addition beyond standard Kubernetes manifests.

**Verify:**
```bash
cd 11-sync-waves/src/gitops-apps-config

mkdir demo-11-sync-waves-nginx-app

touch demo-11-sync-waves-nginx-app/namespace.yaml
touch demo-11-sync-waves-nginx-app/resourcequota.yaml
touch demo-11-sync-waves-nginx-app/limitrange.yaml
touch demo-11-sync-waves-nginx-app/serviceaccount.yaml
touch demo-11-sync-waves-nginx-app/secret.yaml
touch demo-11-sync-waves-nginx-app/configmap.yaml
touch demo-11-sync-waves-nginx-app/networkpolicy.yaml
touch demo-11-sync-waves-nginx-app/service.yaml
touch demo-11-sync-waves-nginx-app/deployment.yaml
```

**`demo-11-sync-waves-nginx-app/namespace.yaml`** — Wave -1
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

**`demo-11-sync-waves-nginx-app/resourcequota.yaml`** — Wave -1
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: sync-waves-quota
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  hard:
    pods: "2"
    requests.cpu: "500m"
    requests.memory: "256Mi"
    limits.cpu: "1"
    limits.memory: "512Mi"
```

**`demo-11-sync-waves-nginx-app/limitrange.yaml`** — Wave -1
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: sync-waves-limits
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "128Mi"
      defaultRequest:
        cpu: "100m"
        memory: "64Mi"
```

**`demo-11-sync-waves-nginx-app/serviceaccount.yaml`** — Wave -1
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-service-account
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

**`demo-11-sync-waves-nginx-app/secret.yaml`** — Wave -1
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
type: Opaque
stringData:
  DEMO_API_KEY: "demo-key-sync-waves-2026"
```

**`demo-11-sync-waves-nginx-app/configmap.yaml`** — Wave 0
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "0"
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>ArgoCD Sync Waves Demo</title></head>
    <body>
      <h1>ArgoCD Sync Waves — Demo-11</h1>
      <p>Resources were created in this order:</p>
      <ol>
        <li>Wave -1: Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret</li>
        <li>Wave  0: ConfigMap, NetworkPolicy</li>
        <li>Wave  1: Service</li>
        <li>Wave  5: Deployment (this pod)</li>
      </ol>
      <p>Governance resources (quota, network policy) existed before this pod started.</p>
    </body>
    </html>
```

**`demo-11-sync-waves-nginx-app/networkpolicy.yaml`** — Wave 0
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-network-policy
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  podSelector:
    matchLabels:
      app: nginx-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports:
        - port: 80
  egress:
    - {}
```

**`demo-11-sync-waves-nginx-app/service.yaml`** — Wave 1
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  selector:
    app: nginx-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-11-sync-waves-nginx-app/deployment.yaml`** — Wave 5
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "5"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      serviceAccountName: nginx-service-account
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          env:
            - name: DEMO_API_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DEMO_API_KEY
          volumeMounts:
            - name: html-content
              mountPath: /usr/share/nginx/html
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
      volumes:
        - name: html-content
          configMap:
            name: nginx-config
```

**Push all manifests to `gitops-apps-config`:**
```bash
git add demo-11-sync-waves-nginx-app/
git commit -m "feat: add demo-11 sync waves nginx app manifests with wave annotations"
git push origin main
```

---

## Step 3: Create the Application CRD

**Set up `argocd-config` local repo:**

```bash
cd 11-sync-waves/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase

mkdir demo-11-sync-waves
touch demo-11-sync-waves/sync-waves-app.yaml

```

**Create `demo-11-sync-waves/sync-waves-app.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sync-waves-demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    path: demo-11-sync-waves-nginx-app
  destination:
    server: https://kubernetes.default.svc
    namespace: sync-waves-demo
```

> **`targetRevision: main`** — explicit branch name from Demo-10 onwards.
> `HEAD` is not used in this course.


No `syncPolicy` block — manual sync only. This lets us watch the wave-by-wave
progression in real time.

> **Why push to Git AND `kubectl apply`?** ArgoCD watches `gitops-apps-config` (the
> source repo) — not `argocd-config` where the Application CRD lives. Pushing
> to `argocd-config` does not trigger ArgoCD to pick up the change automatically
> — `kubectl apply` is still required. We push to Git to version-control the
> Application CRD. This manual apply step is eliminated in Demo-12
> (App-of-Apps). See Demo-05 Concepts for the full explanation.

**Push and apply:**
```bash
git add demo-11-sync-waves/sync-waves-app.yaml
git commit -m "feat: add demo-11 Application CRD for sync waves demo"
git push origin main

kubectl apply -f demo-11-sync-waves/sync-waves-app.yaml
```

**Verify:**
```bash
argocd app get sync-waves-demo-app
```

**Expected:**
```text
Sync Status:   OutOfSync
Health Status: Missing
```

---

## Step 4: Sync and Observe Wave-by-Wave Ordering

Before syncing, open two terminals:

**Terminal 1 — watch resources being created in real time:**
```bash
watch kubectl get all,resourcequota,limitrange,networkpolicy,configmap,serviceaccount,secret \
  -n sync-waves-demo 
```

**Terminal 2 — trigger the sync:**
```bash
argocd app sync sync-waves-demo-app
```

**Observe in the ArgoCD UI:**

Go to `http://localhost:8080` → click `sync-waves-demo-app`.

Watch the resource tree — resources appear in wave order:

```
Wave -1 (all applied together):
  ✅ Namespace/sync-waves-demo
  ✅ ResourceQuota/sync-waves-quota
  ✅ LimitRange/sync-waves-limits
  ✅ ServiceAccount/nginx-service-account
  ✅ Secret/app-secret

Wave 0 (after wave -1 is Healthy):
  ✅ ConfigMap/nginx-config
  ✅ NetworkPolicy/nginx-network-policy

Wave 1 (after wave 0 is Healthy):
  ✅ Service/nginx-service

Wave 5 (after wave 1 is Healthy):
  ✅ Deployment/nginx-app
```

**Expected — all resources created, app Synced and Healthy:**
```bash
argocd app get sync-waves-demo-app
```

```text
Name:               argocd/sync-waves-demo-app
Sync Policy:        Manual
Sync Status:        Synced to HEAD
Health Status:      Healthy

GROUP                    KIND            NAMESPACE         NAME                    STATUS  HEALTH
                         Namespace                         sync-waves-demo         Synced  Healthy
                         ResourceQuota   sync-waves-demo   sync-waves-quota        Synced  Healthy
                         LimitRange      sync-waves-demo   sync-waves-limits       Synced  Healthy
                         ServiceAccount  sync-waves-demo   nginx-service-account   Synced  Healthy
                         Secret          sync-waves-demo   app-secret              Synced  Healthy
                         ConfigMap       sync-waves-demo   nginx-config            Synced  Healthy
networking.k8s.io        NetworkPolicy   sync-waves-demo   nginx-network-policy    Synced  Healthy
                         Service         sync-waves-demo   nginx-service           Synced  Healthy
apps                     Deployment      sync-waves-demo   nginx-app               Synced  Healthy
```

**Key observation — governance before workload:**
The NetworkPolicy and ResourceQuota were created in waves -1 and 0 respectively.
The Deployment was created in wave 5. The nginx pod **never started without
NetworkPolicy in place** — which is exactly the production requirement.

---

## Step 5: Verify the Application

**Check all resources created:**
```bash
kubectl get all -n sync-waves-demo
kubectl get resourcequota,limitrange,networkpolicy,configmap -n sync-waves-demo
```

**Expected:**
```text
NAME                                  READY   STATUS    RESTARTS
pod/nginx-app-xxxxxxxxx-xxxxx         1/1     Running   0

NAME                    TYPE        CLUSTER-IP    PORT(S)
service/nginx-service   ClusterIP   10.96.x.x     80/TCP

NAME                              READY   UP-TO-DATE   AVAILABLE
deployment.apps/nginx-app         1/1     1            1

NAME                                          AGE
resourcequota.v1/sync-waves-quota             2m
limitrange.v1/sync-waves-limits               2m
networkpolicy.networking.k8s.io/nginx...      2m
configmap/nginx-config                        2m
```


**Verify DEMO_API_KEY was injected from Secret:**
```bash
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- env | grep DEMO_API_KEY
```

**Expected:**
```text
DEMO_API_KEY=demo-key-sync-waves-2026
```

**Access the application:**
```bash
kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80
```

Open `http://localhost:8081` — you should see the custom HTML page showing
the wave ordering that was used to create the resources.


**Prove wave ordering — verify resources exist before later waves started:**

The most reliable proof is checking the `creationTimestamp` of resources across
waves. Wave -1 resources must have an earlier timestamp than wave 5 resources.

```bash
# Show creation timestamps for key resources across all waves
kubectl get namespace,resourcequota,configmap,networkpolicy,service,deployment \
  -n sync-waves-demo \
  -o custom-columns=\
"KIND:.kind,NAME:.metadata.name,CREATED:.metadata.creationTimestamp,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave" \
  2>/dev/null | sort -k4
```

**Expected — timestamps increase as wave number increases:**
```text
KIND            NAME                   CREATED                WAVE
Namespace       sync-waves-demo        2026-04-08T10:00:00Z   -1
ResourceQuota   sync-waves-quota       2026-04-08T10:00:00Z   -1
ConfigMap       nginx-config           2026-04-08T10:00:02Z   0
NetworkPolicy   nginx-network-policy   2026-04-08T10:00:02Z   0
Service         nginx-service          2026-04-08T10:00:04Z   1
Deployment      nginx-app              2026-04-08T10:00:06Z   5
```

The timestamps show wave -1 resources were created ~2 seconds before wave 0,
wave 0 before wave 1, and wave 1 before wave 5 — exactly the 2-second inter-wave
delay ArgoCD enforces.

> **The ArgoCD UI is the clearest visual proof.** During sync, go to
> `http://localhost:8080` → click `sync-waves-demo-app` → **Sync** button →
> watch the resource tree. Resources appear in batches — all wave -1 together,
> then wave 0 together after a brief pause, and so on. The UI shows each
> resource's health check completing before the next wave begins.

**Also verify using ArgoCD sync operation detail:**
```bash
argocd app get sync-waves-demo-app -o json \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
resources = data.get('status', {}).get('resources', [])
for r in sorted(resources, key=lambda x: x.get('syncWave', 0)):
    print(f\"Wave {r.get('syncWave','?'):>3}  {r['kind']:<20} {r['name']}\")"
```

**Expected — resources listed in wave order:**
```text
Wave  -1  Namespace            sync-waves-demo
Wave  -1  ResourceQuota        sync-waves-quota
Wave  -1  LimitRange           sync-waves-limits
Wave  -1  ServiceAccount       nginx-service-account
Wave  -1  Secret               app-secret
Wave   0  ConfigMap            nginx-config
Wave   0  NetworkPolicy        nginx-network-policy
Wave   1  Service              nginx-service
Wave   5  Deployment           nginx-app
```

---

## Step 6: Prove Wave Ordering — Add a Resource Mid-Sequence

This step demonstrates why gaps in wave numbers matter. We add a
`PodDisruptionBudget` between the Service (wave 1) and the Deployment (wave 5)
without renumbering anything.

```bash
cd 11-sync-waves/src/gitops-apps-config

touch demo-11-sync-waves-nginx-app/poddisruptionbudget.yaml
```

**Create `demo-11-sync-waves-nginx-app/poddisruptionbudget.yaml`** — Wave 3:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
  namespace: sync-waves-demo
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx-app
```

**Push:**
```bash
git add demo-11-sync-waves-nginx-app/poddisruptionbudget.yaml
git commit -m "feat: add PodDisruptionBudget at wave 3 — no renumbering needed"
git push origin main
```

**Sync:**
```bash
argocd app sync sync-waves-demo-app
```

**Expected — PDB inserted between Service and Deployment:**
```
Wave 1: Service/nginx-service         ✅
Wave 3: PodDisruptionBudget/nginx-pdb ✅  ← inserted with no renumbering
Wave 5: Deployment/nginx-app          ✅
```

**Verify:**
```bash
kubectl get pdb -n sync-waves-demo
```

**Expected:**
```text
NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
nginx-pdb   1               N/A               0
```

---

## Optional: What the Application Does — Full Functional Walkthrough

> **This section is optional.** Skip it if you only want to understand sync
> wave ordering. Read it if you want to understand what each manifest actually
> does and how the resources interact.

The nginx application in this demo is not just a placeholder — every resource
has a real function. Together they form a complete, production-patterned nginx
deployment that you can browse, inspect, and verify.

### The Full Architecture

```
Browser (on your laptop)
    │
    │  kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80
    ▼
nginx-service (ClusterIP, port 80)
    │  governed by: NetworkPolicy (allows ingress on port 80)
    │                ResourceQuota (limits pods in namespace)
    │                LimitRange (sets default CPU/memory)
    ▼
nginx-app pod
    │  identity: nginx-service-account (ServiceAccount)
    │  config:   nginx-config (ConfigMap → /usr/share/nginx/html/index.html)
    │  secret:   app-secret (Secret → env var DEMO_API_KEY)
    ▼
Serves: custom index.html showing the wave ordering that created this pod
```

### What Each Resource Does

**Namespace (`sync-waves-demo`)** — Isolates all demo resources. Deleted in
cleanup to remove everything at once.

**ResourceQuota** — Hard ceiling on the namespace:
```bash
kubectl describe resourcequota sync-waves-quota -n sync-waves-demo
```
```text
Resource          Used   Hard
--------          ----   ----
limits.cpu        200m   1
limits.memory     128Mi  512Mi
pods              1      2
requests.cpu      100m   500m
requests.memory   64Mi   256Mi
```
The Deployment uses 100m CPU and 64Mi memory — within quota. If you tried to
scale to 3 replicas, the third pod would be rejected by quota.

**LimitRange** — Applies default resource requests/limits to any pod that
doesn't specify them. Without this, a pod without `resources:` would have no
limits and could consume the entire node.
```bash
kubectl describe limitrange sync-waves-limits -n sync-waves-demo
```
```text
Type        Resource  Default  DefaultRequest
----        --------  -------  --------------
Container   cpu       200m     100m
Container   memory    128Mi    64Mi
```

**ServiceAccount** — The Deployment uses `nginx-service-account` instead of
the `default` service account. This follows least-privilege — the nginx pod
has only the permissions explicitly granted to this service account (none, in
this demo). In production, this is where you bind IAM roles or RBAC rules.

**Secret** — The `DEMO_API_KEY` is injected as an environment variable into
the nginx container:
```bash
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- env | grep DEMO_API_KEY
```
```text
DEMO_API_KEY=demo-key-sync-waves-2026
```
In production this would be a real API token, DB password, or certificate.

**ConfigMap** — The `index.html` is mounted directly into nginx's serving
directory. Any change to the ConfigMap triggers a sync — nginx serves the
updated page without rebuilding the image.
```bash
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- cat /usr/share/nginx/html/index.html
```
```text
<h1>ArgoCD Sync Waves — Demo-11</h1>
<p>Resources were created in this order:</p>
<ol>
  <li>Wave -1: Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret</li>
  ...
```

**NetworkPolicy** — Restricts which traffic can reach the nginx pod:

```yaml
spec:
  podSelector:
    matchLabels:
      app: nginx-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports:
        - port: 80     # Only HTTP ingress allowed
  egress:
    - {}               # All egress allowed (nginx needs no outbound restrictions)
```

> **Important — minikube limitation:** Minikube's default CNI (Container
> Network Interface) does not enforce NetworkPolicy rules. The NetworkPolicy
> object is created and ArgoCD shows it as Healthy, but traffic blocking
> is not actually enforced on minikube. This demo proves the *ordering*
> guarantee (NetworkPolicy exists before the Deployment pod starts), not
> enforcement. In production with Calico or Cilium, this policy would
> actively block traffic on any port other than 80.

**Service** — ClusterIP exposing the nginx pod on port 80. Only accessible
within the cluster — accessible from your laptop via `kubectl port-forward`.

**Deployment** — The nginx pod itself. References ServiceAccount, ConfigMap
(volume), and Secret (env var) explicitly. ArgoCD's default ordering ensures
all three are created before the Deployment — no wave annotation needed for
these explicit dependencies. Wave 5 is needed only for the *implicit*
dependencies (NetworkPolicy, ResourceQuota, LimitRange).

### Test the Running Application

```bash
kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80
```

Open `http://localhost:8081` — you should see:

```
ArgoCD Sync Waves — Demo-11

Resources were created in this order:
1. Wave -1: Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret
2. Wave  0: ConfigMap, NetworkPolicy
3. Wave  1: Service
4. Wave  5: Deployment (this pod)

Governance resources (quota, network policy) existed before this pod started.
```

**Verify ResourceQuota is enforced — try to exceed it:**
```bash
kubectl scale deployment nginx-app --replicas=3 -n sync-waves-demo
```

```bash
kubectl get pods -n sync-waves-demo
```

**Expected — only 2 pods start, third is blocked by ResourceQuota (max pods: 2):**
```text
NAME                         READY   STATUS    RESTARTS
nginx-app-xxxxxxxxx-aaaaa    1/1     Running   0
nginx-app-xxxxxxxxx-bbbbb    1/1     Running   0
```

```bash
kubectl describe replicaset -n sync-waves-demo | grep -A3 "Warning\|quota"
```

**Expected — quota exceeded event:**
```text
Warning  FailedCreate  pods "nginx-app-xxxxxxxxx-ccccc" is forbidden:
         exceeded quota: sync-waves-quota, requested: pods=1, used: pods=2, limited: pods=2
```

This confirms the ResourceQuota from wave -1 is actively enforced on the
Deployment from wave 5. Scale back to 1 replica:

```bash
kubectl scale deployment nginx-app --replicas=1 -n sync-waves-demo
```

---

## Verify Final State

```bash
# Application synced and healthy
argocd app get sync-waves-demo-app

# All resources in correct namespace
kubectl get all,resourcequota,limitrange,networkpolicy,pdb -n sync-waves-demo

# gitops-apps-config registered with ArgoCD
argocd repo list

# Wave annotations visible on resources
kubectl get deployment nginx-app -n sync-waves-demo \
  -o jsonpath='{.metadata.annotations}' | python3 -m json.tool
```

---

## Cleanup

```bash
# Delete the Application
kubectl delete app sync-waves-demo-app -n argocd

# Delete the namespace (removes all resources)
kubectl delete namespace sync-waves-demo
```

> **Do NOT deregister `gitops-apps-config`** — it is used future demos Leave it registered with ArgoCD.


**Verify:**
```bash
kubectl get namespace sync-waves-demo          # Error: not found
argocd app list | grep sync-waves-demo-app     # No output
argocd repo list | grep gitops-apps-config     # Still listed — correct
```


---

## Key Concepts Summary

**Default ArgoCD ordering handles explicit dependencies — not implicit ones**
ArgoCD automatically applies ConfigMaps, Secrets, and ServiceAccounts before
Deployments that reference them. But NetworkPolicy, ResourceQuota, and
LimitRange are not referenced by Deployments — they are implicit dependencies.
Without sync waves, ArgoCD may apply the Deployment before these governance
resources exist.

**Sync waves assign explicit ordering to implicit dependencies**
`argocd.argoproj.io/sync-wave: "N"` on any resource tells ArgoCD when to apply
it. Lower number = applied first. All resources in wave N must be `Healthy`
before wave N+1 begins. The annotation value must always be a **quoted string**
— `"1"` not `1`.

**Negative wave numbers for bootstrapping resources**
Wave -1 (or lower) is the natural home for resources that must exist before
everything else — Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret.
These run before wave 0 which is the default for unannotated resources.

**Gaps in wave numbers are intentional**
Always leave room between wave numbers (-1, 0, 1, 5 not -1, 0, 1, 2). When
you need to insert a resource between existing waves, you assign it a number
in the gap — no renumbering of existing resources needed.

**ArgoCD waits 2 seconds between each wave**
This gives other controllers time to react to changes before health checks run.
Configurable via `ARGOCD_SYNC_WAVE_DELAY`. Group independent resources in the
same wave rather than creating unnecessary waves.

**Pruning reverses wave order**
Resources are deleted highest wave first, lowest wave last — mirroring creation
order in reverse. If pruning fails in a higher wave, lower waves are not
processed.

**A wave that never becomes Healthy blocks the entire sync**
If a resource in wave N never reaches Healthy status, subsequent waves never
start. Be careful with PVCs using `WaitForFirstConsumer` storage class — they
only become Bound when a pod claims them, so placing them alone in an early
wave causes the sync to hang.

**Waves operate within a single Application — cross-app ordering via App-of-Apps**
Sync waves cannot order resources across two separate Applications. But in
App-of-Apps, wave annotations on child Application CRDs control the order in
which child applications are deployed. Covered in Demo-12.

**Phases always take precedence over waves**
PreSync hooks run before any wave. PostSync hooks run after all waves. Within
the Sync phase, waves determine order.

---

## Commands Reference

```bash

# Manual sync
argocd app sync sync-waves-demo-app

# Refresh without applying
argocd app get sync-waves-demo-app --refresh

# Check app status
argocd app get sync-waves-demo-app

# Prove wave ordering via creation timestamps
kubectl get namespace,resourcequota,configmap,networkpolicy,service,deployment \
  -n sync-waves-demo \
  -o custom-columns=\
"KIND:.kind,NAME:.metadata.name,CREATED:.metadata.creationTimestamp,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave" \
  2>/dev/null | sort -k4

# View resources with wave annotations
kubectl get all -n sync-waves-demo \
  -o custom-columns=NAME:.metadata.name,WAVE:.metadata.annotations."argocd\.argoproj\.io/sync-wave"

# Access the nginx application
kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80

# Verify Secret injection
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- env | grep DEMO_API_KEY
```

---

## Lessons Learned

**1. Default ArgoCD ordering is not enough for production**
ArgoCD's default kind-based ordering handles explicit dependencies correctly.
But implicit dependencies — NetworkPolicy, ResourceQuota, LimitRange — are
not referenced in Deployment manifests and ArgoCD cannot infer them. Sync
waves exist specifically for this gap.


**2. The annotation value must be a quoted string**
`argocd.argoproj.io/sync-wave: "1"` is correct.
`argocd.argoproj.io/sync-wave: 1` (unquoted integer) is wrong and will not
work as expected. Always quote the value.


**3. Watch for PVC WaitForFirstConsumer in early waves**
A PVC with `WaitForFirstConsumer` storage class will never become `Bound` until
a pod claims it. Placing it in an earlier wave than its Deployment will cause
the sync to hang waiting for a Healthy status that cannot be reached.


---

## What's Next

**Demo-12: App-of-Apps Pattern**
Manage multiple ArgoCD Applications declaratively using a parent Application
that watches `argocd-config` and auto-syncs child Applications from Git.
Eliminate the manual `kubectl apply` for every Application CRD.
