# Demo-09: ArgoCD Projects — Governance, Guardrails & RBAC

## Overview

Until now every demo has used the **default project** — fully permissive by design,
with no restrictions on source repositories, deployment destinations, or resource kinds.
That approach does not scale to production.

In this demo we introduce **ArgoCD Projects (`AppProject`)** — the mechanism that
enforces governance boundaries before a sync ever begins. We will understand what a
project actually controls, create one for a two-tier application, deploy both tiers
within its boundaries, and intentionally violate its rules to prove the guardrails
are real and enforced.

**What you'll learn:**
- Why the default project is unsafe for production
- What an `AppProject` controls: source repos, destinations, resource kinds, and RBAC
- The difference between cluster-scoped and namespace-scoped resource controls
- How ArgoCD project roles work and how they map to local users
- How to prove guardrails are enforced by intentionally triggering violations

**What you'll do:**
- Create a new GitHub repo (`argocd-platform-config`) for the AppProject YAML
- Restructure `podinfo-config` into `frontend/` and `backend/` subdirectories
- Create an `AppProject` named `podinfo-project` with strict source, destination, and resource controls
- Deploy two ArgoCD Applications (frontend + backend podinfo) within the project
- Trigger a guardrail violation by adding a disallowed resource to the config repo
- Create a local ArgoCD user and verify project-level RBAC is enforced

## Prerequisites

- ✅ Completed Demo-05 — `podinfo-config` and `argocd-config` repos exist, podinfo image in Docker Hub
- ✅ ArgoCD running on minikube (`kubectl get pods -n argocd`)
- ✅ ArgoCD CLI installed and logged in (`argocd login localhost:8080 --username admin --insecure`)
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT available (same one from Demo-05)

> **PAT Scope — include all repos upfront:** The fine-grained PAT must include all
> repositories it will be used with at creation time. In this demo three repos are
> needed: `podinfo-config`, `argocd-config`, and `argocd-platform-config`. If your
> existing PAT from Demo-05 only covers `podinfo-config` and `argocd-config`, either
> edit it to add `argocd-platform-config` or generate a new PAT selecting all three
> repos. A PAT missing a repo returns a 403 error on push — not an expiry message.

**Verify prerequisites:**
```bash
kubectl get pods -n argocd
argocd app list
argocd repo list        # should show only podinfo-config from Demo-05 — 1 repo expected
```

---

## Background: Why the Default Project Is Not Enough

Since Demo-01, every Application CRD has had this line:

```yaml
spec:
  project: default
```

The default project exists automatically in every ArgoCD installation. It is fully
permissive — any source repo, any destination cluster, any destination namespace, any
Kubernetes resource kind is allowed. There are no guardrails.

This is fine for learning. It is unsafe for production for four reasons:

**1. Unrestricted source repositories**
Any Application can point to any Git repository. A typo or a deliberate change could
deploy manifests from an unauthorised or personal repository into your cluster.

**2. Uncontrolled deployment destinations**
An Application can deploy to any namespace in any cluster. With a single ArgoCD
instance managing 10+ clusters this creates serious blast radius risk.

**3. No resource kind restrictions**
ArgoCD can create any Kubernetes resource — PersistentVolumes, ClusterRoleBindings,
CRDs — unless explicitly restricted. Applications should only be allowed to create
the resource types they actually need.

**4. No application-level RBAC**
Kubernetes RBAC controls who can create `Application` objects. But it cannot control
who can *sync*, *prune*, or *delete* within ArgoCD. That requires ArgoCD project roles.

An `AppProject` addresses all four. Every Application references a project and must
satisfy all of its constraints — **before sync begins**.

---

## Background: What an AppProject Controls

```
AppProject
├── sourceRepos                  → which Git repos are allowed as sources
├── destinations                 → which clusters + namespaces are allowed as targets
├── clusterResourceWhitelist     → allowlist for cluster-scoped resource kinds
├── namespaceResourceWhitelist   → allowlist for namespace-scoped resource kinds
├── namespaceResourceBlacklist   → denylist for namespace-scoped resource kinds
└── roles                        → ArgoCD-level RBAC (sync, view, delete permissions)
```

**Important: namespace vs cluster scoped resources**

Kubernetes resources fall into two categories:

| Category | Examples | Kubernetes behaviour |
|---|---|---|
| Namespace-scoped | Deployment, Service, ConfigMap, Secret, Pod | Live inside a namespace |
| Cluster-scoped | Namespace, PersistentVolume, ClusterRole, CRD | Span the entire cluster |

AppProject controls these two categories differently — the defaults are asymmetric:

```
Cluster-scoped resources   → default DENY  (must explicitly allow what you need)
Namespace-scoped resources → default ALLOW (optionally restrict what you want)
```

**Cluster-scoped resources — `clusterResourceWhitelist` only**

Omit the field entirely to deny all cluster-scoped resources. List only what you need:

```yaml
# Omit field → all cluster-scoped resources denied (no need to write anything)

# Allow only Namespace creation:
clusterResourceWhitelist:
  - group: ''
    kind: Namespace
```

There is no `clusterResourceBlacklist` — since the default is already deny, a
denylist would be redundant.

> **Note:** `clusterResourceWhitelist: []` (empty list) is meaningless — it behaves
> identically to omitting the field. Never write it. Either omit the field or list
> what you want to allow.

**Namespace-scoped resources — whitelist or blacklist depending on intent**

Since the default is allow-all, you have two options to restrict:

```yaml
# Option 1 — Whitelist: allow ONLY these kinds, deny everything else
# Use when your app needs only a few specific resource kinds
namespaceResourceWhitelist:
  - group: apps
    kind: Deployment
  - group: ''
    kind: Service
# Result: only Deployment and Service allowed, all others denied

# Option 2 — Blacklist: allow everything EXCEPT these kinds
# Use when your app needs most resource kinds but wants to block a few
namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
# Result: all namespace-scoped resources allowed except ResourceQuota and LimitRange

# Option 3 — Omit both fields: allow all namespace-scoped resources (no restriction)
```

**Which to use:**

| Situation | Use |
|---|---|
| App needs only 2–4 resource kinds | `namespaceResourceWhitelist` — short allow list |
| App needs most kinds, block a few dangerous ones | `namespaceResourceBlacklist` — short deny list |
| No namespace-scoped restrictions needed | Omit both fields |

In this demo we use `namespaceResourceWhitelist` — podinfo only needs `Deployment`,
`Service`, `ConfigMap`, and `Secret`, so an allowlist is the cleaner choice.

---

## Background: Which Repos Need ArgoCD Registration — and Why

This is one of the most important concepts to get right before building the repo
structure for any demo.

**ArgoCD registration (`argocd repo add`) is only needed for repos that ArgoCD
actively pulls from** — specifically, repos referenced as `source` in an Application
CRD. ArgoCD needs credentials to clone those repos during sync.

For all other repos — ones you only interact with yourself using `kubectl apply` —
registration adds nothing. ArgoCD never touches them.

```
podinfo-config        → ArgoCD pulls this (it is source in Application CRD) → REGISTER ✅
argocd-config         → You apply this with kubectl apply → DO NOT REGISTER ❌
argocd-platform-config → You apply this with kubectl apply → DO NOT REGISTER ❌
```

**Auto-sync scope — what ArgoCD actually watches:**

The `syncPolicy.automated` block in an Application CRD controls what ArgoCD watches
and syncs continuously. It applies to the `source` repo only — not to the repo where
the Application CRD itself lives.

```yaml
# This file lives in argocd-config repo
# You apply it once with: kubectl apply -f podinfo-backend-app.yaml
# ArgoCD never pulls this file

apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git  # ← ArgoCD watches THIS
    path: demo-09/backend                                         # ← for changes HERE
  syncPolicy:
    automated:                    # ← auto-sync applies to podinfo-config only
      prune: true
      selfHeal: true
```

| Repo | Who acts on it | How often |
|---|---|---|
| `podinfo-config` | ArgoCD — continuously polls every ~3 minutes | Ongoing |
| `argocd-config` | You — `kubectl apply` when adding/changing an Application | Once per change |
| `argocd-platform-config` | You — `kubectl apply` when adding/changing an AppProject | Once per change |

**What happens if you register argocd-config or argocd-platform-config anyway?**

Nothing breaks — but nothing extra happens either. ArgoCD stores the credentials but
never uses them because no Application CRD references those repos as a source.
Registration without a matching Application source is harmless but pointless.

**This limitation is exactly what App-of-Apps (Demo-XX) solves — for `argocd-config`:**

Right now `argocd-config` is manual — you `kubectl apply` every time you add or
update an Application CRD. App-of-Apps creates a parent Application that points to
`argocd-config` as its source, making ArgoCD watch and auto-sync that repo too.
At that point `argocd-config` genuinely needs to be registered.

`argocd-platform-config` follows the same pattern in full production App-of-Apps
setups — a dedicated `argocd-projects` Application watches the AppProject folder
and auto-syncs it too. Both repos end up registered and ArgoCD-managed. The safety
for governance changes comes from **Git branch protection and PR review**, not from
keeping the apply step manual. In this demo however, both repos stay manual —
App-of-Apps is covered in Demo-XX.

---



This demo reuses the existing repos from Demo-05 with one addition:

| Repo | Purpose | Change | Registered with ArgoCD |
|---|---|---|---|
| `rselvantech/podinfo-config` | K8s manifests | Restructured into `frontend/` and `backend/` subdirs | ✅ Yes — ArgoCD continuously pulls and syncs from it |
| `rselvantech/argocd-config` | ArgoCD Application CRDs | Two new Application CRDs added | ❌ No — applied once locally with `kubectl apply` |
| `rselvantech/argocd-platform-config` | AppProject YAML | **New repo** | ❌ No — applied once locally with `kubectl apply` |
| `rselvantech/gitops-labs` | Documentation | README added | ❌ No — documentation only |

**Why only `podinfo-config` is registered with ArgoCD:**

ArgoCD only needs credentials for repos it actively pulls from — repos referenced
as `source` in an Application CRD. Both `argocd-config` and `argocd-platform-config`
contain CRD definitions that you apply once with `kubectl apply`. ArgoCD never pulls
from them, so registration adds nothing. Only `podinfo-config` is a source that
ArgoCD watches continuously.

**Why a separate `argocd-platform-config` repo?**

The AppProject YAML is platform-level configuration — it governs applications, not
application manifests themselves. In production organisations, this separation matters:

- Application teams own `podinfo-config` — they push manifests
- Platform/DevOps teams own `argocd-platform-config` — they define governance
- These are separate concerns with separate ownership and separate access controls

---

## Folder Structure

All three repos are initialised as separate git repos under `09-argocd-projects/src/`.
Each subdirectory has its own `.git` folder and points to its respective GitHub remote —
local directory location is irrelevant to git, only the remote URL matters.

```
09-argocd-projects/src/
├── podinfo-config/              ← git init here → remote: rselvantech/podinfo-config
│   └── demo-09/
│       ├── frontend/
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       └── backend/
│           ├── deployment.yaml
│           └── service.yaml
├── argocd-config/               ← git init here → remote: rselvantech/argocd-config
│   └── demo-09/
│       ├── podinfo-frontend-app.yaml
│       └── podinfo-backend-app.yaml
└── argocd-platform-config/      ← git init here → remote: rselvantech/argocd-platform-config (new)
    └── demo-09/
        └── podinfo-project.yaml
```

**What happens on GitHub after all pushes:**

```
rselvantech/podinfo-config (GitHub)
├── deployment.yaml        ← from Demo-05 push
├── service.yaml           ← from Demo-05 push
└── demo-09/               ← from Demo-09 push (different local dir, same remote)
    ├── frontend/
    └── backend/

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml       ← from Demo-05 push
└── demo-09/               ← from Demo-09 push
    ├── podinfo-frontend-app.yaml
    └── podinfo-backend-app.yaml

rselvantech/argocd-platform-config (GitHub)
└── demo-09/               ← new repo, first push
    └── podinfo-project.yaml
```

> **Key concept:** Each `src/` subdirectory is an independent git repo. The local
> directory name and location are irrelevant to git — only the remote URL matters.
> GitHub accumulates files from all pushes regardless of which local directory they
> came from.

---

## The Two-Tier Application: Podinfo Frontend + Backend

We reuse the `podinfo` image built in Demo-05 — `rselvantech/podinfo:v1.0.0` from
your private Docker Hub. The same image supports a two-tier setup natively:

- **Backend** — runs normally, exposes its API on port `9898`
- **Frontend** — configured with `PODINFO_BACKEND_URL` env var pointing to the backend
  Service, proxies calls to it and displays the combined response in its UI

This mirrors the transcript's custom Flask frontend + backend exactly — but with zero
custom code and zero Docker builds. Both tiers use the same public image.

```
Browser → frontend Service (port 9898)
            └── frontend Pod
                  └── calls backend Service (port 9898)  ← via PODINFO_BACKEND_URL
                          └── backend Pod
```

---

## Step 1: Create the `argocd-platform-config` Repository

Create a new **public** GitHub repository named `argocd-platform-config`:

1. Go to GitHub → **New repository**
2. Name: `argocd-platform-config`
3. Visibility: Public (for this demo — private in production)
4. Click **Create repository**

> **Note:** Unlike `podinfo-config`, this repo does not need to be registered with
> ArgoCD using `argocd repo add`. ArgoCD only needs credentials for repos it
> actively pulls from — repos referenced as `source` in an Application CRD.
> `argocd-platform-config` contains the AppProject YAML which you apply once with
> `kubectl apply`. ArgoCD never pulls from it, so no registration is needed.
> The same applies to `argocd-config`. Only `podinfo-config` needs registration
> because ArgoCD continuously watches and syncs from it.

No repo registration step needed. Proceed directly to creating the namespace.

---

## Step 1b: Create the `podinfo-project` Namespace

The namespace must exist before anything else — the docker secret in Step 1c and
the AppProject destination in Step 3 both reference it:

```bash
kubectl create namespace podinfo-project
```

Verify:
```bash
kubectl get namespace podinfo-project
```

Expected:
```
NAME              STATUS   AGE
podinfo-project   Active   5s
```

---

---

---

## Step 1b: Create Docker Registry Secret in `podinfo-project` Namespace

Your podinfo image is private on Docker Hub. The `podinfo-project` namespace needs
a `docker-registry` secret to pull it — same pattern as Demo-05 but in the new namespace:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password='<your-dockerhub-access-token>' \
  --namespace=podinfo-project
```

Verify:
```bash
kubectl get secret dockerhub-secret -n podinfo-project
```

Expected:
```
NAME               TYPE                             DATA   AGE
dockerhub-secret   kubernetes.io/dockerconfigjson   1      5s
```

> **Note:** `imagePullSecrets` is namespace-scoped — the secret from Demo-05's `podinfo`
> namespace is not visible here. It must be recreated in every namespace where private
> images are pulled.

---

## Step 2: Add Manifests to `podinfo-config`

The `podinfo-config` remote repo already exists and is registered with ArgoCD from
Demo-05. Here we initialise `demo-09/src/podinfo-config` as a fresh local git repo
pointing to the same remote — pull the existing history, add `demo-09/` files, push:

```bash
cd gitops-labs/argo-cd-basics-to-prod/09-argocd-projects/src/podinfo-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo-config.git

# Pull existing Demo-05 history into this fresh local repo
git pull origin main --allow-unrelated-histories --no-rebase
```

Create the manifest files below inside `demo-09/frontend/` and `demo-09/backend/`, then push:

```bash
git add demo-09/
git commit -m "feat: add demo-09 frontend and backend manifests"
git push origin main
```

**Backend deployment** (`demo-09/backend/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo-backend
  namespace: podinfo-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podinfo-backend
  template:
    metadata:
      labels:
        app: podinfo-backend
    spec:
      imagePullSecrets:
        - name: dockerhub-secret         # ← same secret type as Demo-05
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0   # ← your private Docker Hub image from Demo-05
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#5a4fcf"      # purple — easy to identify as backend in UI
            - name: PODINFO_UI_MESSAGE
              value: "podinfo backend"
```

**Backend service** (`demo-09/backend/service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo-backend
  namespace: podinfo-project
spec:
  selector:
    app: podinfo-backend
  ports:
    - port: 9898
      targetPort: 9898
```

**Frontend deployment** (`demo-09/frontend/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo-frontend
  namespace: podinfo-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podinfo-frontend
  template:
    metadata:
      labels:
        app: podinfo-frontend
    spec:
      imagePullSecrets:
        - name: dockerhub-secret         # ← same secret type as Demo-05
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0   # ← your private Docker Hub image from Demo-05
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#35b375"      # green — easy to identify as frontend in UI
            - name: PODINFO_UI_MESSAGE
              value: "podinfo frontend"
            - name: PODINFO_BACKEND_URL
              value: "http://podinfo-backend:9898"   # calls the backend Service
```

**Frontend service** (`demo-09/frontend/service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo-frontend
  namespace: podinfo-project
spec:
  selector:
    app: podinfo-frontend
  ports:
    - port: 9898
      targetPort: 9898
```

Push both directories to `podinfo-config`:
```bash
# Already in: gitops-labs/argo-cd-basics-to-prod/09-argocd-projects/src/podinfo-config
git add demo-09/frontend/ demo-09/backend/
git commit -m "feat: add demo-09 frontend and backend manifests"
git push origin main
```

> **Note:** Do not remove the root-level manifests from Demo-05. The new `demo-09/frontend/`
>> and `demo-09/backend/` subdirectories coexist alongside them. Demo-09 Applications point
> to `path: demo-09/frontend` and `path: demo-09/backend` — they do not read the root.
> Demo-05 continues to work unchanged:
> ```
> podinfo-config/
> ├── deployment.yaml        ← root level — Demo-05 Application still reads this
> ├── service.yaml           ← root level — Demo-05 Application still reads this
> ├── demo-09/
> │   ├── frontend/          ← Demo-09 frontend Application reads this
> │   └── backend/           ← Demo-09 backend Application reads this
> └── demo-xx/               ← future demos follow same pattern
> ```

---

## Step 3: Create the AppProject

Create `demo-09/podinfo-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: podinfo-project
  namespace: argocd
spec:
  description: "Project for podinfo frontend and backend workloads"

  # Only these repos may be used as sources by Applications in this project
  sourceRepos:
    - https://github.com/rselvantech/podinfo-config.git

  # Only this cluster + namespace is a valid deployment target
  destinations:
    - server: https://kubernetes.default.svc
      namespace: podinfo-project

  # Cluster-scoped resources: only Namespace is allowed (all others denied)
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace

  # Namespace-scoped resources: only these four kinds are allowed
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ''
      kind: Service
    - group: ''
      kind: ConfigMap
    - group: ''
      kind: Secret

  # ArgoCD project roles — application-level RBAC
  roles:
    - name: viewer
      description: "Read-only access to podinfo applications"
      policies:
        - p, proj:podinfo-project:viewer, applications, get, podinfo-project/*, allow
      groups:
        - podinfo-viewers

    - name: platform-admin
      description: "Full access to podinfo applications"
      policies:
        - p, proj:podinfo-project:platform-admin, applications, *, podinfo-project/*, allow
      groups:
        - podinfo-admins
```

**Field-by-field explanation:**

`sourceRepos` — only `podinfo-config` is trusted. Any Application referencing
`podinfo-project` that points to a different repo will be rejected before sync.

`destinations` — only the local cluster (`kubernetes.default.svc`) and specifically
the `podinfo-project` namespace. Attempting to deploy to `default`, `kube-system`,
or any other namespace will be blocked.

`clusterResourceWhitelist` — only `Namespace` (cluster-scoped) is allowed. If any
manifest in `podinfo-config` tries to create a `PersistentVolume`, `ClusterRole`,
or any other cluster-scoped resource, the sync will fail with a permission error.

`namespaceResourceWhitelist` — only `Deployment`, `Service`, `ConfigMap`, and
`Secret` are allowed. Trying to add a `PersistentVolumeClaim`, `Ingress`, or any
other namespace-scoped resource will be blocked.

`roles` — two project roles defined:
- `viewer`: can only `get` (read) applications in this project
- `platform-admin`: can perform all operations (`sync`, `delete`, `update`, etc.)

The `groups` field maps each role to an SSO group name. In this demo, we simulate
this with local ArgoCD users (Step 8).

Push to `argocd-platform-config` (new repo — initialise from `demo-09/src/`):
```bash
cd gitops-labs/argo-cd-basics-to-prod/09-argocd-projects/src/argocd-platform-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-platform-config.git

git add demo-09/podinfo-project.yaml
git commit -m "feat: add demo-09 podinfo AppProject"
git push origin main
```

Apply the AppProject:
```bash
kubectl apply -f demo-09/podinfo-project.yaml
```

Expected:
```
appproject.argoproj.io/podinfo-project created
```

Verify the project exists:
```bash
kubectl get appproject -n argocd
```

Expected:
```
NAME              AGE
default           10d
podinfo-project   5s
```

Verify in ArgoCD UI — go to **Settings → Projects**. You should see `podinfo-project`
listed with all source repos, destinations, and whitelisted resources visible.

---

## Step 4: Create the ArgoCD Application CRDs

**Backend Application** (`demo-09/podinfo-backend-app.yaml`):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-backend
  namespace: argocd
spec:
  project: podinfo-project
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-09/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-project
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Frontend Application** (`demo-09/podinfo-frontend-app.yaml`):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-frontend
  namespace: argocd
spec:
  project: podinfo-project
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-09/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-project
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Key difference from all previous demos: `project: podinfo-project` instead of
`project: default`. This is what brings both Applications under the AppProject's
governance boundaries.

The `argocd-config` remote repo already exists from Demo-05. Here we initialise
`demo-09/src/argocd-config` as a fresh local git repo pointing to the same remote —
pull existing history, add `demo-09/` files, push:

```bash
cd gitops-labs/argo-cd-basics-to-prod/09-argocd-projects/src/argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git

# Pull existing Demo-05 history into this fresh local repo
git pull origin main --allow-unrelated-histories --no-rebase

git add demo-09/podinfo-backend-app.yaml demo-09/podinfo-frontend-app.yaml
git commit -m "feat: add demo-09 podinfo frontend and backend Applications under podinfo-project"
git push origin main
```

Apply both:
```bash
kubectl apply -f demo-09/podinfo-backend-app.yaml
kubectl apply -f demo-09/podinfo-frontend-app.yaml
```

---

## Step 5: Verify Deployment

Because `automated` sync is enabled, ArgoCD will deploy both tiers automatically
within ~3 minutes of applying the Application CRDs. Click **Refresh** in the UI
to trigger immediately.

```bash
kubectl get pods -n podinfo-project -w
```

Expected — both tiers running:
```
NAME                                READY   STATUS    RESTARTS
podinfo-backend-xxxxxxxxx-xxxxx     1/1     Running   0
podinfo-frontend-xxxxxxxxx-xxxxx    1/1     Running   0
```

```bash
kubectl get all -n podinfo-project
```

Expected:
```
NAME                                    TYPE        PORT(S)
service/podinfo-backend                 ClusterIP   9898/TCP
service/podinfo-frontend                ClusterIP   9898/TCP

NAME                              READY
deployment.apps/podinfo-backend   1/1
deployment.apps/podinfo-frontend  1/1
```

Verify in ArgoCD UI — both applications show `Synced` and `Healthy`, both under the
`podinfo-project` project.

---

## Step 6: Access the Application

**Access the backend directly:**
```bash
kubectl port-forward svc/podinfo-backend -n podinfo-project 9898:9898
```
Open `http://localhost:9898` — purple UI with message `podinfo backend`.

**Access the frontend (which calls the backend):**
```bash
kubectl port-forward svc/podinfo-frontend -n podinfo-project 9899:9898
```
Open `http://localhost:9899` — green UI with message `podinfo frontend`. The frontend
calls the backend and displays its response in the runtime info section.

Verify the backend call from the frontend:
```bash
curl http://localhost:9899/api/echo
```

Expected — response from the backend:
```json
{"hostname":"podinfo-backend-xxxxxxxxx-xxxxx", ...}
```

---

## Step 7: Prove the Guardrails — Trigger a Violation

This is the most important step. We prove the AppProject guardrails are real by
intentionally adding a disallowed resource to `podinfo-config`.

### Violation 1: Disallowed namespace-scoped resource (PersistentVolumeClaim)

Add a PVC to the backend directory in `podinfo-config`:

Create `demo-09/backend/pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backend-storage
  namespace: podinfo-project
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Push to `podinfo-config`:
```bash
cd gitops-labs/argo-cd-basics-to-prod/09-argocd-projects/src/podinfo-config
git add demo-09/backend/pvc.yaml
git commit -m "test: add PVC to demo-09 backend (guardrail test)"
git push origin main
```

Click **Refresh** in ArgoCD UI on the `podinfo-backend` app. Watch what happens:

Expected — ArgoCD marks the app `OutOfSync` and shows a sync error:
```
Resource PersistentVolumeClaim is not permitted in project podinfo-project
```

```bash
argocd app get podinfo-backend
```

Expected:
```
GROUP  KIND                   NAMESPACE         NAME              STATUS     HEALTH   MESSAGE
       PersistentVolumeClaim  podinfo-project   backend-storage   OutOfSync  Missing  Resource not permitted in project
```

**The sync did not proceed.** ArgoCD evaluated the AppProject rules before attempting
to apply any resources, found a violation, and stopped. The Deployment and Service
(which were already synced) are untouched.

**Clean up — remove the PVC:**
```bash
cd gitops-labs/argo-cd-basics-to-prod/09-argocd-projects/src/podinfo-config
git rm demo-09/backend/pvc.yaml
git commit -m "revert: remove PVC guardrail test"
git push origin main
```

Click **Refresh** — the app returns to `Synced` and `Healthy`.

### Violation 2: Disallowed source repository

Apply an Application CRD that points to a repo not in `sourceRepos`:

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-rogue
  namespace: argocd
spec:
  project: podinfo-project
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-project
EOF
```

Check the application status immediately:
```bash
argocd app get podinfo-rogue
```

Expected:
```
SYNC STATUS:   Unknown
CONDITION:     InvalidSpecError: application repo https://github.com/argoproj/argocd-example-apps.git
               is not permitted in project podinfo-project
```

ArgoCD rejected the Application entirely — it cannot even reach `OutOfSync` because
the spec itself is invalid under the project rules.

Clean up:
```bash
kubectl delete app podinfo-rogue -n argocd
```

---


## Background: Users, Groups, Roles and Policies — How They Connect in ArgoCD


### 1. ArgoCD Has No Built-in User Management

ArgoCD has only one built-in user: `admin` (superuser, unrestricted).
For all others, two types are supported:

```
Local users  → defined manually in argocd-cm ConfigMap
SSO users    → authenticated via external provider (Okta, Google, GitHub, Dex)
```

ArgoCD does not have its own identity management system. Users must be defined
via local accounts or SSO before RBAC can be applied to them.

---

### 2. Local Users — Created in `argocd-cm`

```yaml
# kubectl edit cm argocd-cm -n argocd
data:
  accounts.podinfo-viewer: login     # login = can use UI and CLI
  accounts.ci-bot: apiKey            # apiKey = can generate JWT tokens only
```

**`accounts.podinfo-viewer: login` — syntax breakdown:**
```
accounts.<username>: <account-type>
│         │            │
│         │            └── login   = can use UI and CLI
│         │                apiKey  = can generate JWT tokens only
│         │
│         └── username used to log in (e.g. podinfo-viewer)
│
└── fixed prefix — tells ArgoCD this is a user account definition
```

| Account type | Meaning | Used by |
|---|---|---|
| `login` | Can log into UI and CLI | Human users |
| `apiKey` | Can generate JWT tokens only | CI/CD pipelines, automation |

Set a password after creating the account:
```bash
argocd account update-password \
  --account podinfo-viewer \
  --current-password <admin-password> \
  --new-password viewer@1234
```

List all accounts:
```bash
argocd account list
```

> **Important:** Every new local user falls back to `policy.default` from
> `argocd-rbac-cm` unless explicit RBAC rules are set up for them. If
> `policy.default` is empty, the user can log in but sees nothing.

---

### 3. Groups — SSO Only, Not Available for Local Users

Groups are only available for SSO users. When SSO is configured, the identity
provider (Okta, GitHub Teams, Google Groups) sends group membership inside the
JWT token after login. ArgoCD reads these groups and maps them to roles.

Local users cannot be members of groups. This is why in this demo we assign
policies directly to the local username — not via group membership.

```
SSO user logs in
      │
      │ JWT from identity provider contains:
      │ groups: ["podinfo-viewers", "platform-admins"]
      ▼
ArgoCD reads groups → maps to roles via argocd-rbac-cm or AppProject groups field
```

---

### 4. Policy Syntax

Policies are the actual permission rules. ArgoCD uses Casbin as its policy engine.
Two types of lines exist:

**`p` line — assigns a permission:**
```
p, <subject>, <resource>, <action>, <object>, <effect>
```

**`g` line — assigns an SSO group to a role:**
```
g, <sso-group>, <role>
```
> **Note:** `g` line is for SSO groups only. Do not use with local users —
> it is unreliable. For local users always use a direct `p` line instead.

**Field breakdown:**

| Field | Description | Examples |
|---|---|---|
| subject | Who this applies to | `podinfo-viewer`, `role:readonly`, `proj:podinfo-project:viewer` |
| resource | What kind of resource | `applications`, `clusters`, `repositories`, `projects`, `logs`, `exec` |
| action | What operation | `get`, `create`, `update`, `delete`, `sync`, `override`, `action/*`, `*` |
| object | Which specific resource | `podinfo-project/*`, `*/guestbook`, `*` |
| effect | Allow or deny | `allow`, `deny` |

**Examples:**
```
# Allow read-only on all apps in podinfo-project
p, podinfo-viewer, applications, get, podinfo-project/*, allow

# Allow full access on all apps in podinfo-project
p, podinfo-admin, applications, *, podinfo-project/*, allow

# Allow sync only on a specific app
p, podinfo-viewer, applications, sync, podinfo-project/podinfo-frontend, allow

# Assign SSO group to a role (g line)
g, okta-group:platform-admins, role:admin
```

---

### 5. Available Actions Per Resource

| Resource | Available actions |
|---|---|
| `applications` | `get`, `create`, `update`, `delete`, `sync`, `override`, `action/*` |
| `clusters` | `get`, `create`, `update`, `delete` |
| `repositories` | `get`, `create`, `update`, `delete` |
| `projects` | `get`, `create`, `update`, `delete` |
| `accounts` | `get`, `update` |
| `logs` | `get` |
| `exec` | `create` |

---

### 6. Role Reference Syntax

ArgoCD has two types of roles — global built-in roles and project-scoped roles.
The reference syntax differs between them:

**Global built-in roles:**
```
role:admin       → full access to everything
role:readonly    → read-only (get) access to all resources across all projects
```

**Project-scoped role reference:**
```
role:proj:<project-name>:<role-name>
│    │    │               │
│    │    │               └── role name defined inside the AppProject
│    │    │                   (matches roles[].name in AppProject spec)
│    │    │
│    │    └── project name (matches AppProject metadata.name)
│    │
│    └── proj = this is a project-scoped role
│              (vs role:admin or role:readonly which are global)
│
└── fixed prefix — tells ArgoCD this is a role reference
```

**Examples:**
```
role:admin                               → built-in global admin
role:readonly                            → built-in global read-only
role:proj:podinfo-project:viewer         → project-scoped viewer (this demo)
role:proj:podinfo-project:platform-admin → project-scoped admin
```

---

### 7. Where Policies Live — Two Places

**Global — `argocd-rbac-cm`:** applies across all projects

```yaml
# kubectl edit cm argocd-rbac-cm -n argocd
data:
  policy.default: role:readonly      # fallback role for all authenticated users
  policy.csv: |
    # Direct policy for local user (recommended for local users)
    p, podinfo-viewer, applications, get, podinfo-project/*, allow

    # SSO group assigned to built-in role (recommended for SSO)
    g, okta-group:platform-admins, role:admin
```

**Project-scoped — AppProject `roles`:** applies only within that project

```yaml
roles:
  - name: viewer
    policies:
      - p, proj:podinfo-project:viewer, applications, get, podinfo-project/*, allow
    groups:
      - podinfo-viewers              # SSO group only — not for local users
```

### 8. Local User vs SSO — Critical Difference in Role Assignment

This is the most important distinction for this demo:

| | Local user | SSO user |
|---|---|---|
| Role assignment | Direct `p` policy line using username | `g` line mapping SSO group to role |
| AppProject `groups` field | Not applicable | Maps SSO group directly to project role |
| `g` line with `role:proj:...` | Does not work reliably | Works as expected |

**Correct for local user:**
```yaml
policy.csv: |
  p, podinfo-viewer, applications, get, podinfo-project/*, allow
```

**Incorrect for local user (use only for SSO groups):**
```yaml
policy.csv: |
  g, podinfo-viewers-sso-group, role:proj:podinfo-project:viewer  ← SSO groups only
```

---


### 9. Built-in Roles

| Role | Permissions |
|---|---|
| `role:admin` | Full access to everything |
| `role:readonly` | Read-only (`get`) access to all resources across all projects |

Assign as default fallback for all authenticated users:
```yaml
policy.default: role:readonly
```

---

### 10. Verifying Policies

ArgoCD provides a built-in command to test whether a user has a specific permission
without needing to log in as that user:

```bash
# Test if podinfo-viewer can get apps in podinfo-project
argocd admin settings rbac can podinfo-viewer get applications \
  'podinfo-project/podinfo-frontend' --namespace argocd

# Test if a project role has sync permission
argocd admin settings rbac can role:proj:podinfo-project:viewer sync applications \
  'podinfo-project/*' --namespace argocd
```

---

### 11. How Everything Connects — Full Picture
```
argocd-cm                   argocd-rbac-cm                    AppProject
──────────────────          ──────────────────────────────    ─────────────────────
accounts.                   policy.default: role:readonly     roles:
  podinfo-viewer: login                                         - name: viewer
                            policy.csv: |                         policies:
                              # local user — direct p line         - p, proj:...:viewer,
                              p, podinfo-viewer,                      applications,
                                applications, get,                    get,
                                podinfo-project/*, allow              podinfo-project/*,
                                                                      allow
                              # SSO group — g line only            groups:
                              # never use g line for               - okta-group:viewers
                              # local users                          (SSO only)
                              g, okta-group:viewers,
                                role:proj:podinfo-
                                project:viewer
       │                              │                                 │
       ▼                              ▼                                 ▼
User exists               Permissions assigned                 Permissions defined
(local account)           (p line for local users              (scoped to project)
                           g line for SSO groups only)
```

**Login flow for local user `podinfo-viewer`:**
```
podinfo-viewer logs in with password
      │
      │ argocd-cm confirms account exists
      ▼
ArgoCD checks argocd-rbac-cm policy.csv
      │
      │ finds: p, podinfo-viewer, applications, get, podinfo-project/*, allow
      ▼
podinfo-viewer can see podinfo-frontend and podinfo-backend only
Cannot sync, delete, or see any other project's applications
```

**Login flow for SSO user in `okta-group:viewers`:**
```
SSO user logs in via Okta
      │
      │ Okta JWT contains: groups: ["okta-group:viewers"]
      ▼
ArgoCD checks argocd-rbac-cm policy.csv
      │
      │ finds: g, okta-group:viewers, role:proj:podinfo-project:viewer
      ▼
ArgoCD looks up AppProject role: viewer
      │
      │ finds policy: get on podinfo-project/* → allow
      ▼
SSO user can see podinfo-frontend and podinfo-backend only
Cannot sync, delete, or see any other project's applications
```


### 12. This Demo vs Production

| Aspect | This Demo | Production |
|---|---|---|
| User creation | Local user in `argocd-cm` | SSO (Okta, Google, GitHub) |
| Group membership | Not available for local users | SSO groups from identity provider |
| Role assignment | Direct `p` line in `argocd-rbac-cm` by username | `g` line mapping SSO group to role |
| AppProject `groups` field | Not used (local users cannot be in groups) | Maps SSO groups directly to project roles |
| Password management | `argocd account update-password` | Handled by identity provider |

---

## Step 8: Project RBAC — Local User Demo

We create a local ArgoCD user and verify they can only see applications within
the project role assigned to them.

> **Production note:** In production, ArgoCD integrates with SSO (Okta, GitHub,
> Google, AWS IAM Identity Center). The `groups` field in AppProject roles maps
> to SSO groups directly. Local users are used here for simplicity — but the
> key difference is that local users cannot be assigned to groups, so policies
> must be assigned directly using a `p` line in `argocd-rbac-cm`. See the
> Background section above for the full explanation.

### Create a local user

Edit the ArgoCD ConfigMap to add a local user:
```bash
kubectl edit cm argocd-cm -n argocd
```

Add under `data:`:
```yaml
data:
  accounts.podinfo-viewer: login
```

The syntax `accounts.<username>: login` creates a user named `podinfo-viewer`
who can log into the UI and CLI. Save and exit.

Set a password for the new user:
```bash
argocd account update-password \
  --account podinfo-viewer \
  --current-password <admin-password> \
  --new-password viewer@1234
```

Verify the account was created:
```bash
argocd account list
```

Expected:
```
NAME            ENABLED  CAPABILITIES
admin           true     login
podinfo-viewer  true     login
```

---

### Assign the policy directly in `argocd-rbac-cm`

For local users, policies must be assigned directly using a `p` line —
not via a `g` line or AppProject `groups` field (those work for SSO only):
```bash
kubectl edit cm argocd-rbac-cm -n argocd
```

Add under `data:`:
```yaml
data:
  policy.csv: |
    p, podinfo-viewer, applications, get, podinfo-project/*, allow
```

This gives `podinfo-viewer` read-only (`get`) access to all applications
in `podinfo-project` and nothing else. Save and exit.

---

### Verify RBAC enforcement

Log in as `podinfo-viewer` via CLI:
```bash
argocd login localhost:8080 --username podinfo-viewer --password viewer@1234 --insecure
```

Expected:
```
'podinfo-viewer:login' logged in successfully
```

List applications:
```bash
argocd app list
```

Expected — only the two podinfo applications are listed:
```
NAME              PROJECT          SYNC STATUS  HEALTH STATUS
podinfo-backend   podinfo-project  Synced       Healthy
podinfo-frontend  podinfo-project  Synced       Healthy
```

Verify in the ArgoCD UI:
- Log out (top right) → log in with `podinfo-viewer` / `viewer@1234`
- Only `podinfo-frontend` and `podinfo-backend` are visible
- **Sync** button is absent or disabled — `get` permission does not include `sync`
- Applications from all other demos are not visible

Optionally verify the policy is working as expected using the ArgoCD RBAC check command:
```bash
# Switch back to admin first
argocd login localhost:8080 --username admin --password <admin-password> --insecure

# Test get permission — should return: yes
argocd admin settings rbac can podinfo-viewer get applications \
  'podinfo-project/podinfo-frontend' --namespace argocd

# Test sync permission — should return: no
argocd admin settings rbac can podinfo-viewer sync applications \
  'podinfo-project/podinfo-frontend' --namespace argocd
```

Log back in as admin for remaining steps:
```bash
argocd login localhost:8080 --username admin --password <admin-password> --insecure
```

---

## Verify Final State

```bash
# Both applications synced and healthy
argocd app list

# Both pods running
kubectl get pods -n podinfo-project

# AppProject exists with correct spec
kubectl describe appproject podinfo-project -n argocd

# Only podinfo-config registered — argocd-config and argocd-platform-config are never registered
argocd repo list
```

---

## Cleanup

```bash
# Delete both Applications (automated sync + prune will clean up resources)
kubectl delete app podinfo-frontend -n argocd
kubectl delete app podinfo-backend -n argocd

# Delete the AppProject
kubectl delete appproject podinfo-project -n argocd

# Delete the namespace
kubectl delete namespace podinfo-project

# Remove local user (edit argocd-cm and remove the accounts.podinfo-viewer line)
kubectl edit cm argocd-cm -n argocd
kubectl edit cm argocd-rbac-cm -n argocd

# No repo deregistration needed — only podinfo-config was registered, which is reused in later demos
```

---

## Key Concepts Summary

**Default project is permissive by design**
It allows everything — any source, any destination, any resource kind. Suitable for
learning only. Never use `project: default` in production.

**AppProject enforces boundaries before sync**
ArgoCD evaluates all AppProject rules before any resource is applied. A violation
stops the sync entirely — no partial applies.

**sourceRepos — source governance**
Lists trusted Git repositories. Any Application referencing the project that uses
a different repo is rejected with `InvalidSpecError`. Wildcards supported (`*`).

**destinations — deployment governance**
Controls which cluster + namespace combinations are valid targets. Prevents accidental
cross-cluster or cross-namespace deployments.

**Cluster vs namespace resource controls (critical distinction)**

| Resource type | Default | To allow specific kinds | To deny specific kinds |
|---|---|---|---|
| Cluster-scoped (PV, ClusterRole, CRD...) | **Denied** | `clusterResourceWhitelist` | No blacklist — default is already deny |
| Namespace-scoped (Deployment, Service...) | **Allowed** | `namespaceResourceWhitelist` (allow only these) | `namespaceResourceBlacklist` (allow all except these) |

**Project roles are ArgoCD-level RBAC**
Kubernetes RBAC controls who can create `Application` objects. ArgoCD project roles
control who can *sync*, *prune*, *delete*, or *view* within the ArgoCD UI and CLI —
for applications in a specific project only.

**Role policy syntax:**
```
p, proj:<project-name>:<role-name>, applications, <action>, <project>/<app-pattern>, allow
```

Actions: `get`, `sync`, `update`, `delete`, `override`, `action/*`

**Groups map to SSO in production**
The `groups` field in a project role maps to identity provider groups (Okta, GitHub
Teams, Google Groups, AWS IAM Identity Center). Local users are a demo convenience.
The RBAC evaluation is identical.

---

## Commands Reference

```bash
# Create docker registry secret in podinfo-project namespace
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password='<dockerhub-access-token>' \
  --namespace=podinfo-project

# Apply AppProject
kubectl apply -f demo-09/podinfo-project.yaml

# Apply Applications
kubectl apply -f demo-09/podinfo-backend-app.yaml
kubectl apply -f demo-09/podinfo-frontend-app.yaml

# Verify AppProject
kubectl get appproject -n argocd
kubectl describe appproject podinfo-project -n argocd

# Verify Applications
argocd app list
argocd app get podinfo-backend
argocd app get podinfo-frontend

# Trigger immediate check
argocd app refresh podinfo-backend
argocd app refresh podinfo-frontend

# Check repo registrations (only podinfo-config should be listed)
argocd repo list

# Access tiers
kubectl port-forward svc/podinfo-backend -n podinfo-project 9898:9898
kubectl port-forward svc/podinfo-frontend -n podinfo-project 9899:9898


# Local user management
argocd account list
argocd account update-password --account podinfo-viewer \
  --current-password <admin-password> --new-password viewer@1234

# Login as viewer to test RBAC
argocd login localhost:8080 --username podinfo-viewer --password viewer@1234 --insecure

# Login back as admin
argocd login localhost:8080 --username admin --password <admin-password> --insecure
```

---

## Lessons Learned

**1. Always create the AppProject before the Applications that reference it**
ArgoCD validates the project reference at Application creation time. If the project
does not exist, the Application will fail with `InvalidSpecError`. Project first,
Applications second — always.

**2. `clusterResourceWhitelist: []` does not deny cluster resources**
An empty list behaves identically to omitting the field. Cluster-scoped resources are
denied by default in any custom project — you do not need to write anything to achieve
deny-all. Simply omit the field. To allow specific cluster-scoped resources, list them
explicitly. Never write `clusterResourceWhitelist: []` — it is meaningless.

**3. Namespace-scoped resources are allowed by default — choose whitelist or blacklist to restrict**
Unlike cluster-scoped resources, namespace-scoped resources default to allow-all in a
custom project. To restrict them you have two options: `namespaceResourceWhitelist`
(allow only the listed kinds, deny everything else — use when your app needs a short
list of kinds) or `namespaceResourceBlacklist` (allow everything except the listed
kinds — use when your app needs most kinds but you want to block a few dangerous ones
like `ResourceQuota` or `LimitRange`). Omit both fields if no restriction is needed.

**4. Project RBAC operates independently of Kubernetes RBAC**
Kubernetes RBAC cannot control ArgoCD-specific operations like sync, prune, or app
deletion from the ArgoCD UI. You need ArgoCD project roles for that. The two systems
are complementary, not interchangeable.

**5. sourceRepos violations fail at spec validation — not at sync**
If a source repo is not permitted, the Application is marked `InvalidSpecError`
immediately when applied. It never reaches `OutOfSync`. This is stricter and faster
than a sync-time failure.

**6. Groups map to SSO — not to Kubernetes RBAC groups**
The `groups` field in AppProject roles is evaluated by ArgoCD's RBAC engine against
the user's SSO group memberships. It has nothing to do with Kubernetes `Group` subjects
in ClusterRoleBindings.

---

## What's Next

**Demo-XX: App-of-Apps Pattern**
Manage multiple ArgoCD Applications declaratively using a parent Application that
deploys child Applications. Eliminate manual `kubectl apply` for each Application CRD
— let ArgoCD manage them from Git automatically.
