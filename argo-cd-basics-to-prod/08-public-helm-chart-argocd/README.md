# Demo-08: Deploying a Public Helm Chart with ArgoCD — Headlamp Kubernetes Dashboard

## Overview

In Demo-07 we deployed a Helm chart sourced from a **Git repository**. In this demo
we deploy a Helm chart from a **public Helm repository** — no Git repo involved as
the chart source.

This is a common real-world pattern: deploying third-party tooling directly from its
official, publicly maintained Helm chart. ArgoCD supports this natively — you point
it at the Helm repository URL, specify a chart name and version, and ArgoCD handles
the rest.

**The application: Headlamp**

Headlamp is the official Kubernetes web UI — actively maintained under
`kubernetes-sigs` as part of Kubernetes SIG-UI. It replaced the now-archived
Kubernetes Dashboard, and is the Kubernetes-recommended UI going forward.

> **Why not the Kubernetes Dashboard?**
> The `kubernetes/dashboard` repository is officially archived and no longer
> maintained — no security patches, no bug fixes, no new features. Headlamp was
> moved under Kubernetes SIG-UI and is the recommended replacement. It is more
> capable, actively developed, and better aligned with modern Kubernetes practices.

**What you'll learn:**
- The difference between a Git-sourced chart and a Helm-repository-sourced chart
- How `repoURL`, `chart`, `targetRevision`, and `path` work differently for each source type
- How the Helm repository `index.yaml` is generated and how ArgoCD resolves it
- How to safely override chart RBAC defaults via `helm.values`
- Headlamp's architecture, components, and key capabilities
- How Kubernetes RBAC is enforced in the UI via Headlamp's Adaptive UI model
- The difference between `view` and `cluster-admin` permissions in a real UI context

**What you'll do:**
- Deploy Headlamp (`0.40.0`) via ArgoCD using a public Helm repository as source
- Override the chart's default `cluster-admin` binding to `view`
- Create a ServiceAccount, generate a token, and log in to Headlamp
- Verify the ArgoCD deployment through Headlamp
- Explore Headlamp's key functions and observe RBAC enforcement in action
- Upgrade to `cluster-admin` and verify the difference — Secret access, edit
  controls, Cluster Overview, and resource creation

## Prerequisites

- ✅ Completed Demo-07 — Helm chart deployment with ArgoCD understood
- ✅ ArgoCD running on minikube (`kubectl get pods -n argocd`)
- ✅ ArgoCD CLI installed and logged in (`argocd login localhost:8080 --username admin --insecure`)
- ✅ `kubectl` available in terminal

**Verify ArgoCD is running:**
```bash
kubectl get pods -n argocd
argocd app list
```

---

## Concepts

### Background: What is Headlamp?

Headlamp is a user-friendly and extensible Kubernetes UI designed to provide a
seamless experience for managing Kubernetes clusters. It can connect to one or
multiple clusters and can be deployed as a web app in-cluster or as a desktop app.

In 2025, Headlamp became officially part of Kubernetes SIG UI — bringing roadmap
and design discussions closer to the core Kubernetes community, reinforcing its role
as the modern, extensible UI for the project.

**Key design principles:**

- **Multi-cluster** — connects to and manages more than one Kubernetes cluster simultaneously
- **Security** — all actions follow Kubernetes RBAC rules; the UI adapts to what your token allows
- **Realtime** — updates automatically when something changes in the cluster
- **Adaptive UI** — if a user cannot perform an action, the control is not rendered at all — not greyed out, not shown and blocked, simply absent
- **Extensible** — plugin architecture allows custom views, workflows, and integrations

---

### Headlamp Architecture

Headlamp has two main components working together:

```
Your Browser (React frontend)
        │
        │  HTTP / WebSocket
        ▼
headlamp-server (Go backend — runs as a pod in kube-system)
        │  Acts as proxy + auth layer
        │  Handles RBAC, API aggregation, Helm integration
        │
        │  Kubernetes API calls
        ▼
Kubernetes API Server
        │
        ▼
Cluster resources (Pods, Deployments, Services, ConfigMaps...)
```

**Frontend** — React-based UI. Handles user interactions and displays data. Never
talks to the Kubernetes API directly — all calls go through the backend.

**Backend (`headlamp-server`)** — written in Go. Acts as a proxy and service layer.
Handles authentication, RBAC enforcement, and API aggregation. Headlamp uses one
WebSocket from the browser to the backend; the backend then opens WebSockets to
Kubernetes API servers for real-time updates.

**Why this matters for RBAC:** Because the backend proxies all API calls, the pod's
ServiceAccount permissions determine what the UI can display — not just what your
login token allows. Both identities must be scoped correctly.

**Deployment modes:**

| Mode | How | Best for |
|---|---|---|
| In-cluster via Helm | Pod in `kube-system`, browser access | Teams, shared access, production |
| Desktop app | Native app using local kubeconfig | Individual developers |
| Minikube addon | `minikube addons enable headlamp` | Quick local evaluation |

In this demo we use **in-cluster via Helm** — the production-aligned approach.

**Plugin system:**

Headlamp supports a plugin architecture for extending the UI. Notable capabilities
available through plugins:

- **App Catalog** — browse and install Helm charts from ArtifactHub directly from the UI
- **Prometheus integration** — CPU, memory, and custom metrics in resource detail views
- **Backstage plugin** — surface Headlamp views inside your Backstage portal
- **Projects view** — application-centric grouping of resources across namespaces and clusters

Plugins are beyond the scope of this demo but worth knowing — Headlamp is a platform,
not just a read-only viewer.

---

### Git Repo Source vs Helm Repo Source

When configuring the `source` block in an ArgoCD Application CRD, the fields you use
depend entirely on whether your chart lives in a **Git repository** or a **Helm
repository**. These are two fundamentally different systems.

#### Git Repository as Source

A Git repository is a file system you navigate. ArgoCD clones the repo and reads
the directory you point `path` at.

```
github.com/argoproj/argocd-example-apps/    ← repoURL
├── guestbook/                               ← path: guestbook
├── helm-guestbook/                          ← path: helm-guestbook
└── kustomize-guestbook/
```

```yaml
source:
  repoURL: https://github.com/argoproj/argocd-example-apps.git
  targetRevision: HEAD       # Git ref — branch, tag, or commit SHA
  path: helm-guestbook       # directory inside the repo
```

| Field | Meaning |
|---|---|
| `repoURL` | Git clone URL |
| `targetRevision` | Git ref — `HEAD`, `main`, `v1.2.0`, or a commit SHA |
| `path` | Directory inside the repo containing `Chart.yaml` or manifests |
| `chart` | **Not used** |

#### Helm Repository as Source

A Helm repository is not a file system — it is an HTTP catalogue. It has no folders
to navigate. ArgoCD fetches `index.yaml` from the URL, looks up the chart by name
and version, downloads the packaged `.tgz`, unpacks it, and runs `helm template` on it.

```
kubernetes-sigs.github.io/headlamp/         ← repoURL
└── index.yaml                              ← catalogue ArgoCD reads
      entries:
        headlamp:
          - version: 0.40.0
            url: headlamp-0.40.0.tgz        ← resolved via chart + targetRevision
          - version: 0.39.0
            url: headlamp-0.39.0.tgz
```

```yaml
source:
  repoURL: https://kubernetes-sigs.github.io/headlamp/
  chart: headlamp            # lookup key in index.yaml
  targetRevision: 0.40.0     # chart version — not a Git ref
```

| Field | Meaning |
|---|---|
| `repoURL` | Helm repo HTTP URL |
| `chart` | Chart name — lookup key in `index.yaml` |
| `targetRevision` | Chart version — `0.40.0`, not a Git ref |
| `path` | **Not used** |

#### Side-by-Side Comparison

| Field | Git repo source | Helm repo source |
|---|---|---|
| `repoURL` | `https://github.com/org/repo.git` | `https://kubernetes-sigs.github.io/headlamp/` |
| `path` | `helm-guestbook` ✅ | **not used** ❌ |
| `chart` | **not used** ❌ | `headlamp` ✅ |
| `targetRevision` | `HEAD` / `main` / `v1.2.0` (Git ref) | `0.40.0` (chart version) |

#### Where Does `index.yaml` Come From?

The `index.yaml` is **not** hand-authored — it is generated by CI. The GitHub source
code repo and the Helm repository are two completely separate things:

```
github.com/kubernetes-sigs/headlamp         ← source code repo (main branch)
  └── charts/headlamp/                      ← chart source — humans edit this
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
              │
              │  on every release — GitHub Actions:
              │  1. helm package charts/headlamp/ → headlamp-0.40.0.tgz
              │  2. helm repo index . → generates/updates index.yaml
              │  3. git push both to gh-pages branch
              ▼
kubernetes-sigs.github.io/headlamp/         ← Helm repo (GitHub Pages, gh-pages branch)
  ├── index.yaml                            ← what ArgoCD/Helm reads
  ├── headlamp-0.40.0.tgz
  └── headlamp-0.39.0.tgz
```

The `index.yaml` lives in the `gh-pages` branch — not `main`. Switch to the
`gh-pages` branch on GitHub to see the published catalogue directly.

| URL | Purpose | Who uses it |
|---|---|---|
| `github.com/kubernetes-sigs/headlamp` | Source code — developers edit this | Humans, CI |
| `kubernetes-sigs.github.io/headlamp/` | Helm repo — packaged charts | Helm CLI, ArgoCD |

#### How ArgoCD Resolves a Helm Repo Source — Step by Step

```
1. Fetch index.yaml from repoURL
   GET https://kubernetes-sigs.github.io/headlamp/index.yaml

2. Find entry where name = "headlamp" AND version = "0.40.0"

3. Download the .tgz
   GET https://kubernetes-sigs.github.io/headlamp/headlamp-0.40.0.tgz

4. Unpack the .tgz — get Chart.yaml, values.yaml, templates/

5. Run: helm template headlamp <unpacked-chart> --values <your-values>

6. Diff rendered manifests against live cluster state

7. kubectl apply only what changed
```

No Git clone, no directory traversal, no `path` needed — just an HTTP download
of a versioned package.

> **No `helm repo add` needed.** ArgoCD fetches from the Helm repository URL directly
> at sync time — no local Helm setup required on your machine.

---

### Chart RBAC — Why We Override the Default

The Headlamp Helm chart installs **two distinct** RBAC identities:

**1. Pod ServiceAccount — Headlamp's backend**
The Headlamp pod uses this to proxy Kubernetes API calls from your browser. The chart
default is `clusterRoleName: cluster-admin` — intentionally broad so nothing is
hidden from operators. We override this to `view` for a safer learning environment.

**2. Login ServiceAccount — your browser session**
A separate ServiceAccount whose token you paste into the Headlamp login screen. This
controls what you can see and do in the UI. We create this ourselves in Step 4.

```
Pod ServiceAccount (headlamp-admin)  → backend proxies API calls    → view (we override)
Login ServiceAccount (headlamp-viewer) → your browser session       → view (we create)
```

Using `view` for both means full read visibility across most resource types, with
no accidental edits or deletions via the UI — correct for learning.

> **Production note:** In production, the pod ServiceAccount should have enough
> permissions to serve all resources your users need to see. Scope it deliberately —
> `cluster-admin` is convenient but broad. Always review chart RBAC defaults before
> applying any third-party chart to a cluster.

---

## Folder Structure

```
08-public-helm-chart-argocd/
├── README.md
├── images/
└── src/
    ├── headlamp-app.yaml       ← ArgoCD Application CRD
    └── headlamp-rbac.yaml      ← ServiceAccount + ClusterRoleBinding for login
```

---

## Step 1: Create the ArgoCD Application CRD

Create `src/headlamp-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: headlamp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes-sigs.github.io/headlamp/
    chart: headlamp
    targetRevision: 0.40.0
    helm:
      values: |
        clusterRoleBinding:
          create: true
          clusterRoleName: view        # override default cluster-admin → view
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

**Key differences from Demo-07 (Git-sourced chart):**

| Field | Demo-07 | Demo-08 |
|---|---|---|
| `repoURL` | Git URL | `https://kubernetes-sigs.github.io/headlamp/` |
| `path` | `helm-guestbook` | **removed — not used with Helm repos** |
| `chart` | not present | `headlamp` |
| `targetRevision` | `HEAD` (Git ref) | `0.40.0` (chart version) |
| `helm.values` | value overrides | `clusterRoleName: view` |

**Why `namespace: kube-system`?**
Official Headlamp docs recommend `kube-system` for in-cluster deployment. The
Headlamp backend pod needs access to system-level API resources to proxy calls
correctly, and `kube-system` is the conventional home for cluster infrastructure.

**Why the `helm.values` block?**
The chart defaults `clusterRoleName` to `cluster-admin`. We override to `view`
so the pod's ServiceAccount has read-only access. The `helm` block here works
identically to Demo-07 — inline value overrides apply regardless of whether the
chart source is Git or a Helm repository.

---

## Step 2: Apply and Verify Initial State

```bash
cd 08-public-helm-chart-argocd/src
kubectl apply -f headlamp-app.yaml
```

Check the application status:
```bash
kubectl get app headlamp -n argocd
```

Expected:
```
NAME       SYNC STATUS   HEALTH STATUS
headlamp   OutOfSync     Missing
```

`OutOfSync/Missing` — ArgoCD has registered the application but has not yet deployed
it. Manual sync required.

---

## Step 3: Sync — Deploy Headlamp

```bash
argocd app sync headlamp
```

Watch the pod come up:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=headlamp -w
```

Expected after sync completes:
```
NAME                        READY   STATUS    RESTARTS
headlamp-xxxxxxxxx-xxxxx    1/1     Running   0
```

Verify application status:
```bash
argocd app get headlamp
```

Expected:
```
Sync Status:   Synced to 0.40.0
Health Status: Healthy
```

Verify the chart's ClusterRoleBinding was created with `view` — not `cluster-admin`:
```bash
kubectl get clusterrolebinding headlamp-admin -o yaml | grep -A3 "roleRef:"
```

Expected:
```
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
```

Verify resources created:
```bash
kubectl get all -n kube-system -l app.kubernetes.io/name=headlamp
```

Expected:
```
NAME                            TYPE        PORT(S)
service/headlamp                ClusterIP   80/TCP

NAME                      READY
deployment.apps/headlamp  1/1
```

> **Notice:** Headlamp's service is plain HTTP on port `80` — no Kong proxy, no
> HTTPS-only requirement, no certificate warnings. A clean, single-pod deployment.

---

## Step 4: Create RBAC for the Login Token

The Headlamp login screen requires a bearer token from a ServiceAccount. We create
a dedicated one — separate from the chart's pod ServiceAccount.

Create `src/headlamp-rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: headlamp-viewer
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: headlamp-viewer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: headlamp-viewer
    namespace: kube-system
```

Apply:
```bash
kubectl apply -f headlamp-rbac.yaml
```

Verify:
```bash
kubectl get serviceaccount headlamp-viewer -n kube-system
kubectl get clusterrolebinding headlamp-viewer-binding
```

**Two separate identities — summary:**

| ServiceAccount | Created by | Purpose | Role |
|---|---|---|---|
| `headlamp-admin` | Helm chart | Pod identity — backend proxies API calls | `view` (overridden) |
| `headlamp-viewer` | Our RBAC file | Login token — your browser session | `view` |

> **Note:** `view` ClusterRole covers Deployments, Services, Pods, ConfigMaps,
> Namespaces — and intentionally **excludes Secrets and cluster-scoped resources
> like Nodes**. Both exclusions are by Kubernetes design, not a Headlamp limitation.

---

## Step 5: Generate a Bearer Token

```bash
kubectl create token headlamp-viewer -n kube-system
```

Copy the token output — you will paste this into the Headlamp login screen in the
next step.

> `kubectl create token` generates a short-lived token valid for **1 hour** by
> default. It is not stored anywhere in the cluster — run the command again when
> it expires. This is the correct approach for Kubernetes 1.24+.

---

## Step 6: Access Headlamp

Port-forward the Headlamp service:
```bash
kubectl port-forward svc/headlamp -n kube-system 4466:80
```

Open in your browser:
```
http://localhost:4466
```

You will see the Headlamp login screen. Paste the token from Step 5 and click
**Authenticate**.

---

## Step 7: Explore and Verify Headlamp

This section is split into two parts:

**Part A — ArgoCD Deployment Verification (required)**
Confirms the deployment is correct using Headlamp as an inspection tool. Work
through all steps in Part A before moving on.

**Part B — Headlamp Feature Exploration (optional)**
Explores Headlamp's broader capabilities — RBAC enforcement, Adaptive UI, and
the difference between `view` and `cluster-admin`. Recommended for understanding
Headlamp as a production tool.

---

## Part A: ArgoCD Deployment Verification

### 7a: Verify Namespace Visibility

In the top navigation bar, click the **namespace dropdown**. You should see all
namespaces in your cluster:
```
All namespaces
argocd
default
kube-system
guestbook-helm        ← from Demo-07 if not cleaned up
...
```

This confirms Headlamp can read cluster-wide namespace metadata. The `view`
ClusterRole is working correctly.

---

### 7b: Verify Headlamp Deployment

Navigate to **Workloads → Deployments** in the left sidebar. Set namespace to
**All namespaces**.

Confirm Headlamp itself is running:

| Namespace | Deployment | Status |
|---|---|---|
| `kube-system` | `headlamp` | Should show `1/1` Ready |
| `kube-system` | `coredns` | System DNS — should already be running |

Click on the `headlamp` Deployment. Verify:
- Replica count: `1/1`
- The ReplicaSet it manages
- Labels and annotations applied by the Helm chart

Navigate to **Workloads → Pods**. Click on the `headlamp-xxx` pod and verify:
- **Status:** `Running`
- **Namespace:** `kube-system`
- **Container image:** `ghcr.io/headlamp-k8s/headlamp:v0.40.0` (or similar)

---

### 7c: View Live Pod Logs

While on the Headlamp pod detail page, click the **Logs** tab.

You should see Headlamp's server startup logs streaming in real-time — equivalent
to `kubectl logs -f` but in the browser. This is Headlamp's WebSocket-based live
log streaming. Logs update continuously without reloading the page.

This is a key operational function — in real troubleshooting you navigate to a
failing pod and read its logs directly from the browser without needing terminal
access or `kubectl`.

---

### 7d: Verify ArgoCD Components via Headlamp

Switch to the `argocd` namespace. Navigate to **Workloads → Deployments**.

Confirm all ArgoCD components are running:

| Deployment | What it is |
|---|---|
| `argocd-server` | ArgoCD UI and API |
| `argocd-repo-server` | Git repo and Helm chart fetching |
| `argocd-application-controller` | Sync and reconciliation controller |
| `argocd-dex-server` | SSO / authentication |
| `argocd-redis` | Cache layer |

Click on `argocd-server`. Navigate to its **Logs** tab — you can read ArgoCD's
own server logs from inside Headlamp, useful for diagnosing sync failures without
switching tools.

Navigate to **Config → ConfigMaps** in the `argocd` namespace. Locate `argocd-cm`
— the ArgoCD configuration ConfigMap we referenced in earlier demos. Click it to
see `timeout.reconciliation` and other settings rendered cleanly in the UI, without
running `kubectl describe`.

---

### 7e: Verify Headlamp Service

Navigate to **Network → Services** with namespace set to `kube-system`.

Click on the `headlamp` service. Verify:
- **Type:** `ClusterIP`
- **Port:** `80`
- **Target port:** `4466`
- **Selector:** `app.kubernetes.io/name=headlamp`

This confirms the service is correctly routing to the Headlamp pod via the label
selector — same mechanism we have seen in all previous demos, now visible in the UI.

---

## Part B: Headlamp Feature Exploration (Optional)

This part explores Headlamp's key capabilities and demonstrates how Kubernetes RBAC
is reflected in the UI. Two phases — first with `view` permissions, then with
`cluster-admin`.

---

### 7f: Secrets — Access Denied by Design (`view`)

Navigate to **Configuration → Secrets** in any namespace.

You will see an access error — secrets are not listed. This is **expected and correct**.

The built-in `view` ClusterRole intentionally excludes Secrets. This is a deliberate
Kubernetes security decision — `get`, `list`, and `watch` on Secrets all reveal
credential values. The `view` role does not mean "read everything" — Secrets are
explicitly excluded.

```
view ClusterRole grants:     Deployments, Services, Pods, ConfigMaps, Events...
view ClusterRole excludes:   Secrets ← intentional, by design
```

Verify directly:
```bash
kubectl auth can-i list secrets \
  --as=system:serviceaccount:kube-system:headlamp-viewer
```

Expected:
```
no
```

---

### 7g: Nodes and Cluster Overview — Access Denied by Design (`view`)

Navigate to **Cluster → Overview** and **Cluster → Nodes** in the left sidebar.

You will see an access error or empty state for both. This is **expected and correct**.

`Nodes` are a cluster-scoped resource. The `view` ClusterRole grants read access
to namespace-scoped resources but excludes most cluster-scoped resources like Nodes
and PersistentVolumes. `Namespaces` are the exception — explicitly included in
`view` because namespace awareness is needed for almost every workload operation.

Verify directly:
```bash
kubectl auth can-i list nodes \
  --as=system:serviceaccount:kube-system:headlamp-viewer
```

Expected:
```
no
```

---

### 7h: Adaptive UI — No Edit Controls (`view`)

Navigate to any Deployment (e.g. `headlamp` in `kube-system`).

You will notice there is **no Edit button** — not hidden, not greyed out, simply
absent. This is Headlamp's Adaptive UI. Headlamp queries your token's permissions
when it loads a resource and removes controls for actions you cannot perform.

Verify RBAC enforcement at the API level:
```bash
kubectl auth can-i update deployments \
  --as=system:serviceaccount:kube-system:headlamp-viewer \
  -n kube-system
```

Expected:
```
no
```

```bash
kubectl auth can-i get deployments \
  --as=system:serviceaccount:kube-system:headlamp-viewer \
  -n kube-system
```

Expected:
```
yes
```

**`view` permissions — summary so far:**

| Resource | Scoped | `view` access |
|---|---|---|
| Deployments, Services, Pods | Namespace | ✅ |
| ConfigMaps, Events | Namespace | ✅ |
| Namespaces | Cluster | ✅ — explicitly included in `view` |
| Secrets | Namespace | ❌ — escalation risk, excluded by design |
| Nodes, Cluster Overview | Cluster | ❌ — cluster-scoped, excluded |
| Edit / Create / Delete controls | — | ❌ — Adaptive UI removes them |

---

### 7i: Grant `cluster-admin` and Verify Full Control

Now upgrade the login token to `cluster-admin` and verify what changes — Secret
access, Cluster Overview, Nodes, edit controls, and resource creation.

**Update the ClusterRoleBinding:**

`roleRef` is immutable — delete and recreate:
```bash
kubectl delete clusterrolebinding headlamp-viewer-binding

kubectl create clusterrolebinding headlamp-viewer-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:headlamp-viewer
```

Generate a fresh token:
```bash
kubectl create token headlamp-viewer -n kube-system
```

Log out of Headlamp (top-right corner → Sign out) and log back in with the new token.

---

**Verify 1: Cluster Overview and Nodes**

Navigate to **Cluster → Overview**. You should now see cluster information — node
count, resource usage summary, and cluster-level metrics.

Navigate to **Cluster → Nodes**. The nodes list is now populated — you can see your
minikube node with its status, capacity, and allocated resources.

---

**Verify 2: Secret Access**

Navigate to **Configuration → Secrets** in the `argocd` namespace.

Secrets are now listed — `argocd-initial-admin-secret`, `argocd-secret`, etc.
Click on `argocd-initial-admin-secret`. You will see:
- **Data section** — `password` key is visible
- **Value is masked** by default — shown as `•••••`
- **Eye icon** next to the value — click to unmask and reveal the actual secret value

Even with `cluster-admin`, values are masked by default — Headlamp requires a
deliberate click to reveal them. This is a Headlamp UI design decision — preventing
accidental exposure of credentials while still allowing authorised access.

---

**Verify 3: Edit Controls Appear**

Navigate to **Workloads → Deployments** → `headlamp` in `kube-system`.

Edit is now available via two entry points:
- **3-dot menu** under the Action column in the Deployments list view → Edit option
- **Edit (pencil) icon** in the Deployment detail view

Click Edit in the detail view. Change `replicas: 1` to `replicas: 2` and save.
The edit succeeds — no permission error.

```bash
kubectl get deploy headlamp -n kube-system
```

Expected:
```
NAME       READY   UP-TO-DATE   AVAILABLE
headlamp   2/2     2            2
```

Scale it back to 1:
```bash
kubectl scale deploy headlamp -n kube-system --replicas=1
```

---

**Verify 4: Create a Resource from Headlamp UI**

Click the **`+ Create`** button in the **bottom-left corner** of Headlamp.

Paste the following YAML and click **Apply**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: headlamp-demo-test
  namespace: default
data:
  demo: "created from Headlamp UI with cluster-admin"
```

Verify it was created:
```bash
kubectl get configmap headlamp-demo-test -n default
```

Expected:
```
NAME                  DATA   AGE
headlamp-demo-test    1      5s
```

Navigate to **Configuration → ConfigMaps** in the `default` namespace — confirm
`headlamp-demo-test` is listed and its key-value data is visible in the UI.

Clean up:
```bash
kubectl delete configmap headlamp-demo-test -n default
```

---

**`view` vs `cluster-admin` in Headlamp — full comparison:**

| Capability | `view` | `cluster-admin` |
|---|---|---|
| List Deployments, Services, Pods | ✅ | ✅ |
| Read ConfigMaps | ✅ | ✅ |
| Namespace list | ✅ | ✅ |
| Cluster Overview | ❌ | ✅ |
| Nodes list | ❌ | ✅ |
| List Secrets | ❌ | ✅ |
| Secret values visible in UI | ❌ | ✅ (masked by default, eye icon to reveal) |
| Edit button (list + detail view) | ❌ Adaptive UI removes it | ✅ both entry points |
| Edit succeeds | ❌ | ✅ |
| `+ Create` resource from UI | ❌ | ✅ |
| Multi-cluster selector | ✅ (informational) | ✅ |

---

**Revert to `view` after this step — restore least privilege:**
```bash
kubectl delete clusterrolebinding headlamp-viewer-binding

kubectl create clusterrolebinding headlamp-viewer-binding \
  --clusterrole=view \
  --serviceaccount=kube-system:headlamp-viewer
```

---

### 7j: Multi-Cluster Awareness (Informational)

In the top-left corner, locate the **cluster selector** — it shows your current
cluster name (e.g. `minikube`).

In this demo you see only one cluster. In a real multi-cluster environment, additional
clusters appear here — each independently accessible from the same Headlamp instance
with its own RBAC token and context. This is a core Headlamp capability absent from
the archived Kubernetes Dashboard which was always single-cluster only.

---

## Verify Final State

```bash
# ArgoCD application synced and healthy
argocd app get headlamp

# Headlamp pod running
kubectl get pods -n kube-system -l app.kubernetes.io/name=headlamp

# Chart ClusterRoleBinding uses view — not cluster-admin
kubectl get clusterrolebinding headlamp-admin -o yaml | grep -A3 "roleRef:"

# Login ServiceAccount and binding exist
kubectl get serviceaccount headlamp-viewer -n kube-system
kubectl get clusterrolebinding headlamp-viewer-binding

# Confirm viewer binding is back to view after Part B
kubectl get clusterrolebinding headlamp-viewer-binding -o yaml | grep -A3 "roleRef:"

# Service exists on port 80
kubectl get svc headlamp -n kube-system
```

---

## Cleanup

```bash
# Delete the ArgoCD Application (does not cascade-delete cluster resources)
kubectl delete -f headlamp-app.yaml

# Delete login RBAC resources
kubectl delete -f headlamp-rbac.yaml

# Delete remaining Helm chart resources
kubectl delete all -n kube-system -l app.kubernetes.io/name=headlamp
kubectl delete clusterrolebinding headlamp-admin
kubectl delete serviceaccount headlamp-admin -n kube-system
```

> **Note:** Deleting the ArgoCD Application does not cascade-delete deployed
> Kubernetes resources. Always clean up separately — or add the ArgoCD finalizer
> to enable cascade delete, covered in a later demo.

---

## Key Concepts Summary

**Helm repo source vs Git source**
When `chart` is specified instead of `path`, ArgoCD treats `repoURL` as a Helm
repository HTTP URL and fetches the packaged chart directly. `targetRevision` is
the chart version, not a Git ref. `path` is not used and should be omitted.

**`targetRevision` is the chart version — always pin it**
Use an explicit version (`0.40.0`). Pinning ensures every sync uses exactly the same
chart — no surprise upgrades when the chart publisher releases a new version.

**`index.yaml` is generated by CI — not hand-authored**
It lives in the `gh-pages` branch, not `main`. The source chart lives in
`charts/headlamp/` in `main`. GitHub Actions packages it and publishes the `.tgz`
and updated `index.yaml` to GitHub Pages on every release.

**No `helm repo add` needed**
ArgoCD fetches Helm charts directly using `repoURL`. Nothing needs to be configured
locally with `helm repo add`.

**Always review chart RBAC defaults before deploying**
The Headlamp chart defaults to `cluster-admin` for the pod ServiceAccount. Always
check a chart's `values.yaml` for RBAC-related defaults before applying any
third-party chart to a cluster.

**Headlamp uses a backend proxy model**
The React frontend never calls the Kubernetes API directly. All calls go through
the `headlamp-server` Go backend. The pod's ServiceAccount permissions set the
ceiling on what can be displayed — even if your login token has broader permissions.

**Headlamp vs archived Kubernetes Dashboard**

| | Kubernetes Dashboard (archived) | Headlamp |
|---|---|---|
| Maintenance | Archived — no updates | Active — Kubernetes SIG-UI |
| Access | HTTPS only via Kong proxy | Plain HTTP (port 80) |
| Setup complexity | Kong dependency, double-sync quirk | Single pod, clean sync |
| Multi-cluster | No | Yes |
| Plugin system | No | Yes — ArtifactHub, Prometheus, Backstage |
| Deployment modes | In-cluster only | In-cluster, desktop app, minikube addon |
| Secret value exposure | N/A | Masked by default, eye icon to reveal |

---

## Commands Reference

```bash
# Sync application
argocd app sync headlamp

# Check status
argocd app get headlamp

# Diff before syncing
argocd app diff headlamp

# Verify chart ClusterRoleBinding uses view
kubectl get clusterrolebinding headlamp-admin -o yaml | grep -A3 "roleRef:"

# Generate bearer token (1 hour)
kubectl create token headlamp-viewer -n kube-system

# Port-forward to Headlamp UI
kubectl port-forward svc/headlamp -n kube-system 4466:80

# Verify RBAC permissions for viewer ServiceAccount
kubectl auth can-i list secrets \
  --as=system:serviceaccount:kube-system:headlamp-viewer
kubectl auth can-i list nodes \
  --as=system:serviceaccount:kube-system:headlamp-viewer
kubectl auth can-i update deployments \
  --as=system:serviceaccount:kube-system:headlamp-viewer -n kube-system
kubectl auth can-i get deployments \
  --as=system:serviceaccount:kube-system:headlamp-viewer -n kube-system

# Upgrade to cluster-admin (Part B)
kubectl delete clusterrolebinding headlamp-viewer-binding
kubectl create clusterrolebinding headlamp-viewer-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:headlamp-viewer

# Revert to view (after Part B)
kubectl delete clusterrolebinding headlamp-viewer-binding
kubectl create clusterrolebinding headlamp-viewer-binding \
  --clusterrole=view \
  --serviceaccount=kube-system:headlamp-viewer

```

---

## Lessons Learned

**1. `chart` replaces `path` for Helm repository sources**
Using `path` with a Helm repo URL will cause the sync to fail. Use `chart` (chart
name) instead. This is the single most common mistake when switching from Git-sourced
to Helm-repo-sourced charts in ArgoCD.

**2. `targetRevision` is the chart version — `HEAD` does not apply**
For Helm repo sources, always use an explicit version. Pin it for reproducibility.

**3. `index.yaml` is not in the source repo — it is generated by CI**
The GitHub source repo and the Helm repository are two separate things. The
`index.yaml` lives in the `gh-pages` branch, not `main`. Understanding this
separation prevents confusion when inspecting any Helm chart's source repository.

**4. Always review chart RBAC defaults before applying to a cluster**
The Headlamp chart defaults to `cluster-admin`. This would have gone unnoticed
without checking `values.yaml`. Make reviewing RBAC defaults a habit for any
third-party chart before applying it to a cluster.

**5. Two RBAC identities — keep them separate and named clearly**
The pod's ServiceAccount and the login ServiceAccount serve different purposes.
Keeping them separate makes permissions clear, auditable, and independently
revocable.

**6. `view` is not "read everything" — know what it excludes**
Secrets and most cluster-scoped resources (Nodes, PersistentVolumes) are excluded
from `view` by design. This is not a Headlamp limitation — it is intentional
Kubernetes RBAC behaviour. Understand what your role grants and excludes before
applying it.

---

## What's Next

**Demo-09: ArgoCD Projects — Governance, Guardrails & RBAC**
Introduce `AppProject` to enforce boundaries around source repositories, deployment
destinations, allowed resource kinds, and application-level RBAC — all evaluated
before sync begins.
