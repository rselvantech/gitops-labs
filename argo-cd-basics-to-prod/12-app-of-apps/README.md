# Demo-12: App-of-Apps — Product-Level GitOps Orchestration

## Overview

Demo-11 established sync waves — controlling the order of resources within a
single Application. This demo steps up one level: what happens when your
product is not one Application but many?

Every demo so far has treated each component (frontend, backend) as a separate
ArgoCD Application that you apply manually with `kubectl apply`. At small scale
this is manageable. At production scale — 10, 20, 50 microservices across dev,
staging, and prod — it breaks down. You have no single view of product health,
no automated Application lifecycle, and every new component requires a manual
`kubectl apply` to register it with ArgoCD.

App-of-Apps solves this. It is a **GitOps pattern** — not a new ArgoCD feature
— where one parent Application manages a set of child Applications. The parent
watches a Git path containing child Application CRDs. When you push a new child
Application YAML to that path, the parent auto-syncs it. The child then manages
its own Kubernetes resources.

```
Git push → parent detects → child Application created → child syncs → pods running
```

No `kubectl apply` needed after the parent is created. The entire product is
managed declaratively from Git.

**What you'll learn:**
- Why managing multiple Applications manually does not scale
- What App-of-Apps is — pattern not feature, uses the Application CRD you already know
- The three-tier sync order: parent → child Applications → Kubernetes resources
- How the parent Application uses `argocd-config` as its source
- The role of sync options: `CreateNamespace=true`, `PruneLast`, `ApplyOutOfSyncOnly`
- How self-healing at the Application level works — deleted child is auto-recreated
- How to view product health from a single parent Application

**What you'll do:**
- Add frontend and backend manifests to `podinfo-config` under `demo-12-app-of-apps/`
- Create child Application CRDs in `argocd-config` under `demo-12-app-of-apps/`
- Create the parent Application CRD pointing to `argocd-config/demo-12-app-of-apps/`
- Apply the parent — observe child Applications and pods created automatically
- Delete a child Application manually — observe self-healing recreating it
- Verify product health from the parent Application view

---

## Prerequisites

- ✅ Completed Demo-11 — sync waves understood
- ✅ `podinfo-config` exists and is registered with ArgoCD
- ✅ `argocd-config` exists and is committed to GitHub
- ✅ ArgoCD running on minikube
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `podinfo-config` and `argocd-config`

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

### 4. Required repos registered and available
```bash
argocd repo list
```

**Expected:** `podinfo-config` listed with `Successful` status.

If `podinfo-config` is not listed, re-register it:
```bash
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

---

## Concepts

### Why Managing Multiple Applications Manually Does Not Scale

With everything learned so far, deploying a two-component product requires:

```bash
kubectl apply -f argocd-config/podinfo-frontend-app.yaml
kubectl apply -f argocd-config/podinfo-backend-app.yaml
```

At two components this is fine. At ten components across three environments:

```
10 components × 3 environments = 30 Application CRDs
→ 30 kubectl apply commands
→ 30 applications to monitor individually
→ No single health view for the product
→ Every new microservice = another manual kubectl apply
→ Every new environment = repeat all 30 commands
```

You also lose the GitOps property for the Applications themselves — ArgoCD
watches the source repos (podinfo-config) but nobody watches argocd-config.
Changes to Application CRDs require manual intervention.

---

### What App-of-Apps Is

App-of-Apps is a pattern where one ArgoCD Application (the **parent**) has
`argocd-config` as its source and manages a set of child Application CRDs
stored there.

```
Parent Application
  source: argocd-config/demo-12-app-of-apps/
  destination: argocd namespace

  ↓ creates and manages

Child Application: product-demo-frontend
  source: podinfo-config/demo-12-app-of-apps/frontend/
  destination: product-demo namespace

Child Application: product-demo-backend
  source: podinfo-config/demo-12-app-of-apps/backend/
  destination: product-demo namespace

  ↓ each child creates and manages

Kubernetes resources: Deployment, Service (frontend)
Kubernetes resources: Deployment, Service (backend)
```

**It uses the same Application CRD you already know.** No new resource types.
The only difference is the parent Application's source contains Application
YAMLs instead of Deployment/Service YAMLs.

---

### The Three-Tier Sync Order

```
1. Parent Application syncs
   → reads argocd-config/demo-12-app-of-apps/
   → finds frontend-app.yaml and backend-app.yaml
   → creates Application resources in argocd namespace

2. Child Applications sync (automatically, with automated sync)
   → frontend app reads podinfo-config/demo-12-app-of-apps/frontend/
   → backend app reads podinfo-config/demo-12-app-of-apps/backend/
   → each creates its Kubernetes resources

3. Kubernetes resources become Running
   → pods start, services created
   → children report Healthy
   → parent reports Healthy
```

**Product health = parent Application health.**
If all child Applications are Synced and Healthy, the parent is Healthy.
One dashboard — one check — full product status.

---

### Sync Options Explained

The transcript introduces three sync options used in production App-of-Apps
deployments. These go in the child Application's `syncPolicy.syncOptions`:

**`CreateNamespace=true`**
ArgoCD creates the destination namespace if it does not exist. Without this,
the sync fails with `namespace not found`. Eliminates the manual
`kubectl create namespace` step from every demo so far.

```yaml
syncOptions:
  - CreateNamespace=true
```

**`PruneLast=true`**
During a sync, ArgoCD creates and updates resources first, then deletes orphaned
resources last. Without this, ArgoCD may delete a resource before its replacement
is ready — causing a brief outage. With `PruneLast`, the old resource stays until
the new one is healthy.

```yaml
syncOptions:
  - PruneLast=true
```

**`ApplyOutOfSyncOnly=true`**
ArgoCD only sends changed resources to the Kubernetes API server. Without this,
every sync sends all 50 manifests to the API server — even if 49 are unchanged.
This reduces API server load significantly in large applications.

```yaml
syncOptions:
  - ApplyOutOfSyncOnly=true
```

---

### Self-Healing at the Application Level

Self-healing in previous demos reverted manual `kubectl` changes to pods and
deployments. In App-of-Apps, self-healing works at the Application level too.

When `selfHeal: true` is set in the parent Application:
- If someone manually deletes a child Application (`kubectl delete app product-demo-frontend`)
- The parent detects the drift — a child Application that should exist no longer does
- The parent immediately re-creates the child Application
- The child then syncs and recreates its Kubernetes resources

```
kubectl delete app product-demo-frontend
  → parent detects: desired state has frontend Application, live state does not
  → parent self-heals: re-creates product-demo-frontend Application
  → frontend app syncs: Deployment and Service re-created
```

This means ArgoCD protects not just your application resources but also the
Application CRDs themselves.

---

### argocd-config Now Has a Source — The Critical Shift

Until Demo-12, `argocd-config` was a repo you pushed to and ran `kubectl apply`
from. ArgoCD never watched it.

From Demo-12 onwards, the parent Application watches `argocd-config`. This is
the critical shift:

```
Before Demo-12:
  argocd-config → you run kubectl apply → ArgoCD learns about Application

From Demo-12:
  argocd-config → parent Application watches → ArgoCD auto-syncs Application
```

**Now `argocd-config` needs to be registered with ArgoCD** — because the parent
Application references it as its source. ArgoCD must be able to clone it.

> **PAT scope:** Your PAT must include read access to both `podinfo-config`
> (source for child Applications) and `argocd-config` (source for parent
> Application). `argocd-config` now genuinely needs ArgoCD registration —
> not just for your local git push but because ArgoCD actively clones it.

---

## Folder Structure

```
12-app-of-apps/src/
├── podinfo-config/          ← git init → remote: rselvantech/podinfo-config
│   └── demo-12-app-of-apps/
│       ├── frontend/
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       └── backend/
│           ├── deployment.yaml
│           └── service.yaml
└── argocd-config/           ← git init → remote: rselvantech/argocd-config
    └── demo-12-app-of-apps/
        ├── frontend-app.yaml    ← child Application CRD
        ├── backend-app.yaml     ← child Application CRD
        └── parent-app.yaml      ← parent Application CRD (applied once manually)
```

**What happens on GitHub after all pushes:**

```
rselvantech/podinfo-config (GitHub)
├── deployment.yaml                      ← Demo-05 root (untouched)
├── service.yaml                         ← Demo-05 root (untouched)
├── demo-06-sync-pruning/                ← Demo-06 (untouched)
├── demo-09-argocd-projects/             ← Demo-09 (untouched)
└── demo-12-app-of-apps/                 ← Demo-12 adds this
    ├── frontend/
    │   ├── deployment.yaml
    │   └── service.yaml
    └── backend/
        ├── deployment.yaml
        └── service.yaml

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml                     ← Demo-05 root (untouched)
├── demo-06-sync-pruning/                ← Demo-06 (untouched)
├── demo-09-argocd-projects/             ← Demo-09 (untouched)
├── demo-10-sync-hooks/                  ← Demo-10 (untouched)
├── demo-11-sync-waves/                  ← Demo-11 (untouched)
└── demo-12-app-of-apps/                 ← Demo-12 adds this
    ├── frontend-app.yaml
    ├── backend-app.yaml
    └── parent-app.yaml
```

---

## Step 1: Register `argocd-config` with ArgoCD

`argocd-config` has always been applied manually with `kubectl apply`. In this
demo the parent Application watches `argocd-config` as its source — so ArgoCD
now needs credentials to clone it.

```bash
argocd repo add https://github.com/rselvantech/argocd-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

**Verify:**
```bash
argocd repo list
```

**Expected:** Both `podinfo-config` and `argocd-config` listed with `Successful` status.

```text
TYPE  NAME  REPO                                                    STATUS
git         https://github.com/rselvantech/argocd-config.git       Successful
git         https://github.com/rselvantech/podinfo-config.git      Successful
```

> This is the one permanent change in the repo setup. `argocd-config` is now
> registered with ArgoCD and stays registered for all future demos. See the
> Concepts section — "argocd-config Now Has a Source — The Critical Shift."

---

## Step 2: Add Manifests to `podinfo-config`

Initialise the local repo:

```bash
cd gitops-labs/argo-cd-basics-to-prod/12-app-of-apps/src
mkdir podinfo-config && cd podinfo-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

Create the manifest files. Both frontend and backend use `rselvantech/podinfo:v1.0.0`
— the same image from Demo-05. No image build needed.

**`demo-12-app-of-apps/frontend/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-demo-frontend
  namespace: product-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-demo-frontend
  template:
    metadata:
      labels:
        app: product-demo-frontend
    spec:
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#4CAF50"
            - name: PODINFO_UI_MESSAGE
              value: "Demo-12: App-of-Apps — Frontend"
```

**`demo-12-app-of-apps/frontend/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-demo-frontend-svc
  namespace: product-demo
spec:
  selector:
    app: product-demo-frontend
  ports:
    - port: 9898
      targetPort: 9898
```

**`demo-12-app-of-apps/backend/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-demo-backend
  namespace: product-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-demo-backend
  template:
    metadata:
      labels:
        app: product-demo-backend
    spec:
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#2196F3"
            - name: PODINFO_UI_MESSAGE
              value: "Demo-12: App-of-Apps — Backend"
```

**`demo-12-app-of-apps/backend/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-demo-backend-svc
  namespace: product-demo
spec:
  selector:
    app: product-demo-backend
  ports:
    - port: 9898
      targetPort: 9898
```

**Push to `podinfo-config`:**
```bash
git add demo-12-app-of-apps/
git commit -m "feat: add demo-12 frontend and backend manifests for app-of-apps"
git push origin main
```

---

## Step 3: Create Child Application CRDs in `argocd-config`

Set up `argocd-config` local repo:

```bash
cd gitops-labs/argo-cd-basics-to-prod/12-app-of-apps/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

Create the child Application CRDs. These are the Application resources the
parent will create and manage — not you.

**`demo-12-app-of-apps/frontend-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: product-demo-frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-12-app-of-apps/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: product-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
```

**`demo-12-app-of-apps/backend-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: product-demo-backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-12-app-of-apps/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: product-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
```

**Push to `argocd-config`:**
```bash
git add demo-12-app-of-apps/frontend-app.yaml demo-12-app-of-apps/backend-app.yaml
git commit -m "feat: add demo-12 child Application CRDs for frontend and backend"
git push origin main
```

> **Do NOT `kubectl apply` the child Application CRDs.** The parent will create
> them. Applying them manually defeats the purpose of App-of-Apps.

---

## Step 4: Create and Apply the Parent Application

The parent Application is the only resource you `kubectl apply` manually in this
demo. After this, everything is managed by ArgoCD from Git.

**Create `demo-12-app-of-apps/parent-app.yaml` in `argocd-config`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: product-demo-parent
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/argocd-config.git
    targetRevision: HEAD
    path: demo-12-app-of-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Key points in the parent Application:**

- `source.repoURL` points to `argocd-config` — the repo containing child Application CRDs
- `source.path` is `demo-12-app-of-apps` — the folder containing `frontend-app.yaml` and `backend-app.yaml`
- `destination.namespace` is `argocd` — child Application CRDs are created in the ArgoCD namespace
- No `syncOptions` — the parent manages Application CRDs, not namespaced Kubernetes resources
- `selfHeal: true` — if a child Application is manually deleted, parent re-creates it

**Push and apply:**

```bash
git add demo-12-app-of-apps/parent-app.yaml
git commit -m "feat: add demo-12 parent Application CRD for app-of-apps"
git push origin main

kubectl apply -f demo-12-app-of-apps/parent-app.yaml
```

> **Why push AND `kubectl apply`?** The parent Application needs to be applied
> once to bootstrap the pattern. After this, pushing new child Application YAMLs
> to `argocd-config` is all that is needed — the parent auto-syncs them.
> This is the last `kubectl apply` you need for this product's Applications.

---

## Step 5: Observe the Three-Tier Sync

Watch in a second terminal:
```bash
watch -n 2 'argocd app list'
```

**Expected progression — three tiers in sequence:**

```text
# Immediately after kubectl apply:
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     OutOfSync     Missing

# Seconds later — parent syncs, creates child Applications:
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     Synced        Healthy
product-demo-frontend   OutOfSync     Missing
product-demo-backend    OutOfSync     Missing

# Seconds later — children sync, create Kubernetes resources:
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     Synced        Healthy
product-demo-frontend   Synced        Healthy
product-demo-backend    Synced        Healthy
```

**Verify in ArgoCD UI:**

Go to `http://localhost:8080` → click `product-demo-parent`.

You will see the resource tree showing two child Application resources.
Click into each child to see their own resource trees (Deployment, Service, Pod).

**Verify Kubernetes resources:**
```bash
kubectl get all -n product-demo
```

**Expected:**
```text
NAME                                        READY   STATUS    RESTARTS
pod/product-demo-backend-xxxxxxxxx-xxxxx    1/1     Running   0
pod/product-demo-frontend-xxxxxxxxx-xxxxx   1/1     Running   0

NAME                               TYPE        PORT(S)
service/product-demo-backend-svc   ClusterIP   9898/TCP
service/product-demo-frontend-svc  ClusterIP   9898/TCP

NAME                                   READY   UP-TO-DATE   AVAILABLE
deployment.apps/product-demo-backend   1/1     1            1
deployment.apps/product-demo-frontend  1/1     1            1
```

**Key observation:**
The `product-demo` namespace was created automatically by `CreateNamespace=true`
in the child Applications — no manual `kubectl create namespace` needed.

---

## Step 6: Access the Product

**Access frontend:**
```bash
kubectl port-forward svc/product-demo-frontend-svc -n product-demo 9898:9898
```

Open `http://localhost:9898` — green UI with "Demo-12: App-of-Apps — Frontend".

**Access backend:**
```bash
kubectl port-forward svc/product-demo-backend-svc -n product-demo 9899:9898
```

Open `http://localhost:9899` — blue UI with "Demo-12: App-of-Apps — Backend".

The two different colours confirm the correct manifests were applied to each
component despite both using the same podinfo image.

---

## Step 7: Prove Self-Healing at Application Level

This step demonstrates that `selfHeal: true` in the parent Application protects
child Applications — not just Kubernetes resources.

**Open a watch terminal:**
```bash
watch -n 1 'argocd app list'
```

**Delete the frontend child Application manually:**
```bash
kubectl delete app product-demo-frontend -n argocd
```

**Expected in watch terminal — frontend disappears then reappears:**
```text
# Immediately after delete:
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     OutOfSync     Degraded
product-demo-backend    Synced        Healthy
# product-demo-frontend is gone

# Seconds later — parent self-heals:
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     Synced        Healthy
product-demo-frontend   Synced        Healthy      ← recreated
product-demo-backend    Synced        Healthy
```

**Verify in ArgoCD UI:**
```
product-demo-parent → App Details → History
```

You will see a sync event triggered automatically immediately after the deletion
with no manual intervention.

**Key observation:** The parent Application detected that the desired state
(two child Applications defined in Git) did not match the live state (one child
Application in the cluster). It self-healed by re-creating the missing child.
The frontend pods and service were also automatically restored by the child
Application's own automated sync.

---

## Step 8: Add a New Component Without kubectl apply

This step proves the core value of App-of-Apps — adding a new component only
requires a Git push.

**Create `demo-12-app-of-apps/cache-app.yaml` in `argocd-config`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: product-demo-cache
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-12-app-of-apps/cache
  destination:
    server: https://kubernetes.default.svc
    namespace: product-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
```

**Also create the manifests in `podinfo-config`:**

```bash
# In podinfo-config local repo:
mkdir -p demo-12-app-of-apps/cache
```

**`demo-12-app-of-apps/cache/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-demo-cache
  namespace: product-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-demo-cache
  template:
    metadata:
      labels:
        app: product-demo-cache
    spec:
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#FF9800"
            - name: PODINFO_UI_MESSAGE
              value: "Demo-12: App-of-Apps — Cache (added without kubectl apply)"
```

**`demo-12-app-of-apps/cache/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-demo-cache-svc
  namespace: product-demo
spec:
  selector:
    app: product-demo-cache
  ports:
    - port: 9898
      targetPort: 9898
```

**Push both repos:**
```bash
# In podinfo-config:
git add demo-12-app-of-apps/cache/
git commit -m "feat: add cache component manifests for demo-12"
git push origin main

# In argocd-config:
git add demo-12-app-of-apps/cache-app.yaml
git commit -m "feat: add cache child Application for demo-12"
git push origin main
```

**Watch — no kubectl apply needed:**
```bash
watch -n 2 'argocd app list'
```

**Expected — cache Application appears automatically:**
```text
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     Synced        Healthy
product-demo-frontend   Synced        Healthy
product-demo-backend    Synced        Healthy
product-demo-cache      Synced        Healthy      ← appeared with no kubectl apply
```

---

## Verify Final State

```bash
# All three Applications plus parent
argocd app list

# All pods running
kubectl get pods -n product-demo

# Parent Application detail
argocd app get product-demo-parent

# Both repos registered
argocd repo list

# Namespace auto-created
kubectl get namespace product-demo
```

**Expected — four Applications, four pods:**
```text
NAME                    SYNC STATUS   HEALTH STATUS
product-demo-parent     Synced        Healthy
product-demo-frontend   Synced        Healthy
product-demo-backend    Synced        Healthy
product-demo-cache      Synced        Healthy
```

---

## Cleanup

```bash
# Deleting the parent Application with prune cascades to children
kubectl delete app product-demo-parent -n argocd

# Verify children are also deleted (pruning from parent)
argocd app list
# → only non-demo-12 applications remain

# Delete the namespace and all resources
kubectl delete namespace product-demo

# argocd-config stays registered — it is used by all future demos
# podinfo-config stays registered — used since Demo-05
```

> **Do NOT deregister `argocd-config`** — it is now permanently registered
> and watched by the parent Application in Demo-12 and all future demos.

---

## Key Concepts Summary

**App-of-Apps is a pattern, not a feature**
It uses the standard Application CRD you have used since Demo-03. The only
difference is the parent Application's source contains Application YAMLs
instead of Deployment/Service YAMLs.

**Three-tier sync order**
Parent Application syncs → child Application CRDs created in argocd namespace →
each child syncs its source repo → Kubernetes resources created. This order
always holds regardless of how many components the product has.

**One Application = one product health view**
When the parent Application is Healthy, all child Applications and their
resources are Healthy. One dashboard — full product status.

**Self-healing works at the Application level**
With `selfHeal: true` in the parent, manually deleting a child Application
triggers the parent to re-create it. ArgoCD protects Application CRDs just
as it protects Deployment and Service resources.

**Adding a component = Git push, no kubectl**
After the parent is created, all new components are added by pushing a new
child Application YAML to `argocd-config`. The parent auto-syncs it. No
`kubectl apply` needed for any new component.

**`argocd-config` is now registered with ArgoCD**
Previously `argocd-config` was only used for manual `kubectl apply`. From this
demo onwards it is the parent Application's source — ArgoCD actively clones it.
This is the GitOps closure: even Application CRDs are managed declaratively.

**Sync options for production child Applications**

| Option | Purpose |
|---|---|
| `CreateNamespace=true` | Auto-creates destination namespace |
| `PruneLast=true` | Creates new resources before deleting old ones |
| `ApplyOutOfSyncOnly=true` | Reduces API server load — only sends changed resources |

---

## Commands Reference

```bash
# Register argocd-config with ArgoCD (one time)
argocd repo add https://github.com/rselvantech/argocd-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>

# Apply parent Application (one time — last kubectl apply for this product)
kubectl apply -f demo-12-app-of-apps/parent-app.yaml

# Watch all Applications
watch -n 2 'argocd app list'

# Get parent Application status
argocd app get product-demo-parent

# Get child Application status
argocd app get product-demo-frontend
argocd app get product-demo-backend

# List all Applications in argocd namespace
kubectl get apps -n argocd

# Delete child Application (to prove self-healing)
kubectl delete app product-demo-frontend -n argocd

# Access frontend
kubectl port-forward svc/product-demo-frontend-svc -n product-demo 9898:9898

# Access backend
kubectl port-forward svc/product-demo-backend-svc -n product-demo 9899:9898

# Delete parent (cascades to children with prune)
kubectl delete app product-demo-parent -n argocd
```

---

## Lessons Learned

**1. App-of-Apps requires registering `argocd-config` with ArgoCD**
Previous demos used `argocd-config` only for manual `kubectl apply` — ArgoCD
never needed to clone it. App-of-Apps makes the parent Application watch
`argocd-config` as its source, so ArgoCD now needs credentials to clone it.
Register it with `argocd repo add` before creating the parent Application.

**2. Never `kubectl apply` child Application CRDs**
The parent creates and manages child Application CRDs. Manually applying them
defeats the App-of-Apps pattern — you lose the audit trail, the self-healing
protection, and the declarative control. Only apply the parent.

**3. Parent destination namespace is `argocd` — not the app namespace**
The parent creates Application CRDs, which live in the `argocd` namespace.
Child Applications define their own destination namespace. Getting this wrong
is the most common App-of-Apps setup mistake.

**4. Deleting the parent with prune cascades to children**
When the parent Application is deleted with pruning enabled, ArgoCD deletes the
child Application CRDs it manages. Those children then lose their Application
CRDs and their Kubernetes resources are also deleted. This is the correct
cascade behaviour — but be intentional about it in production.

**5. `CreateNamespace=true` eliminates all manual namespace creation**
Every demo before Demo-12 required `kubectl create namespace` before applying
the Application CRD. With `CreateNamespace=true` in the child's `syncOptions`,
the namespace is created automatically on first sync. This is the production
pattern — namespaces are part of the declarative desired state.

**6. Adding a new component after the parent exists = one Git push**
No `kubectl apply`, no ArgoCD UI interaction, no CLI command. Push the child
Application YAML to `argocd-config` and the manifest to `podinfo-config`.
The parent detects the new child Application YAML and creates it automatically.
This is the value of App-of-Apps at scale.

---

## What's Next

**Demo-13: ApplicationSets**
App-of-Apps requires you to write one Application CRD per component and one
per environment. For 10 components across 3 environments that is 30 Application
CRDs. ApplicationSets generate Application CRDs dynamically from templates —
one ApplicationSet replaces dozens of manually written Application CRDs.
The App-of-Apps foundation from this demo carries directly into ApplicationSets.
