# Demo-02: Installing Argo Rollouts

## Overview

This demo installs everything needed to work with Argo Rollouts on a local
Minikube cluster. By the end you will have the controller running, the
kubectl plugin installed, and both the web UI dashboard and CLI dashboard
accessible — ready to use from Demo-03 onwards.

**This demo uses the Helm-based install.** Both Helm and raw manifest
approaches are documented in Step 2 — follow Step 2a (Helm) as the
primary path. Step 2b (raw manifest) is provided as the alternative if
you do not have Helm installed.

**What you'll learn:**
- How the Argo Rollouts controller is installed via Helm and raw manifest
- What the Argo Rollouts CRDs are, what each one does, and how they
  compare to Argo CD CRDs
- How standard and namespace-scoped installs differ — and how each install
  method handles this distinction differently
- How to install the `kubectl argo rollouts` plugin and verify it works
- How the web UI dashboard works — and why it differs between Helm and
  raw manifest installs
- How to use the CLI dashboard (`kubectl argo rollouts get rollout --watch`)
- What each dashboard view shows and when to use which

**What you'll do:**
- Create the `argo-rollouts` namespace
- Install the Argo Rollouts controller via Helm (Step 2a) with dashboard enabled
- Verify all controller pods, services, CRDs, and RBAC resources are present
- Install the `kubectl argo rollouts` plugin
- Access the web UI dashboard and orientate yourself before Demo-03
- Understand the CLI dashboard reference before using it in Demo-03

---

## Prerequisites

- ✅ Minikube installed and running
- ✅ `kubectl` configured and pointing to your Minikube cluster
- ✅ `helm` v3.x installed — used in Step 2a for the controller install
- ✅ Internet access (to pull Helm charts and the plugin binary)

**Verify prerequisites:**

```bash
# Minikube is running
minikube status

# kubectl points to minikube
kubectl config current-context
# Expected: minikube

# Helm is installed
helm version --short
# Expected: v3.x.x
```

> **No Helm?** If you do not have Helm installed, use Step 2b (raw
> manifest) instead of Step 2a. Everything from Step 3 onwards is
> identical regardless of which install method you chose.

---

## Concepts

### Argo Rollouts Architecture — What Gets Installed

The Helm install creates more resources than the raw manifest install.
The table below shows exactly what each method creates — verified against
the actual Helm chart templates and the raw `install.yaml`.

```
argo-rollouts namespace
  ├── Deployment: argo-rollouts                  ← controller (BOTH methods)
  │     └── Pod: argo-rollouts-xxxxxxxxx
  │           watches all Rollout CRDs cluster-wide
  │           manages ReplicaSets on behalf of Rollouts
  │           runs analysis, manages traffic weights
  │
  ├── Deployment: argo-rollouts-dashboard        ← Helm only (dashboard.enabled: true)
  │     └── Pod: argo-rollouts-dashboard-xx
  │           serves the web UI on port 3100
  │           reads Rollout state from the Kubernetes API
  │
  ├── Service: argo-rollouts-dashboard           ← Helm only (dashboard.enabled: true)
  │     ClusterIP on port 3100
  │     access via: kubectl port-forward svc/argo-rollouts-dashboard 3100:3100
  │
  ├── Service: argo-rollouts-metrics             ← Helm only (controller.metrics.enabled: true)
  │     ClusterIP on port 8090                     NOT created by default — requires opt-in
  │     exposes /metrics endpoint for Prometheus    Raw manifest: metrics port exists on pod
  │     scraping of controller health metrics       but NO Service is created by install.yaml
  │
  ├── ServiceAccount: argo-rollouts              ← identity for the controller pod (BOTH)
  ├── ServiceAccount: argo-rollouts-dashboard    ← identity for dashboard pod (Helm only)
  │
  ├── ClusterRole: argo-rollouts                 ← controller permissions (BOTH)
  │     read/write Rollouts, ReplicaSets, Services, AnalysisRuns etc.
  ├── ClusterRole: argo-rollouts-aggregate-to-admin  ← RBAC aggregation (BOTH)
  ├── ClusterRole: argo-rollouts-aggregate-to-edit   ← RBAC aggregation (BOTH)
  ├── ClusterRole: argo-rollouts-aggregate-to-view   ← RBAC aggregation (BOTH)
  │     these three add Rollout permissions to Kubernetes built-in admin/edit/view roles
  │     any namespace admin automatically inherits the ability to manage Rollouts
  ├── ClusterRole: argo-rollouts-dashboard       ← dashboard read permissions (Helm only)
  │
  ├── ClusterRoleBinding: argo-rollouts          ← binds controller SA to ClusterRole (BOTH)
  └── ClusterRoleBinding: argo-rollouts-dashboard ← binds dashboard SA (Helm only)

Cluster-wide (not namespaced):
  ├── CRD: rollouts.argoproj.io                 ← the Rollout resource type
  ├── CRD: analysistemplates.argoproj.io        ← reusable analysis definitions
  ├── CRD: clusteranalysistemplates.argoproj.io ← cluster-scoped analysis
  ├── CRD: analysisruns.argoproj.io             ← per-rollout analysis executions
  └── CRD: experiments.argoproj.io              ← A/B experiment runs (advanced)
```

**The controller is cluster-scoped.** One Argo Rollouts controller manages
Rollout resources across all namespaces. You do not install a controller per
namespace. A single installation in `argo-rollouts` namespace watches and
manages Rollouts in `default`, `production`, `staging`, or any other namespace.

**Both install methods — Helm and raw manifest — install the same 5 CRDs
and the same controller binary.** The difference is only in how you manage
the installation going forward: Helm gives you `helm upgrade`, `helm rollback`,
`helm history`, and values-based customisation. The raw manifest is a single
`kubectl apply` with no Helm state to manage.

> **Argo CD comparison — what gets installed and where:**
>
> Argo CD and Argo Rollouts follow the same pattern — a controller
> namespace, cluster-scoped CRDs, and application objects that live in
> your own namespaces. The key differences are:
>
> ```
> Argo CD:
>   Controller namespace: argocd
>   CRDs: Application, AppProject, ApplicationSet (cluster-wide)
>   App objects: Application CRDs live in argocd namespace
>                (the objects ArgoCD manages — Deployments, Services —
>                 live in your app namespace)
>   Dashboard: argocd-server pod — accessed via port-forward or LoadBalancer
>
> Argo Rollouts:
>   Controller namespace: argo-rollouts
>   CRDs: Rollout, AnalysisTemplate, AnalysisRun etc. (cluster-wide)
>   App objects: Rollout resources live IN your app namespace
>                (not in argo-rollouts namespace — unlike Argo CD's
>                 Application CRDs which live in argocd namespace)
>   Dashboard: built into controller pod (raw manifest) or
>              separate pod (Helm with dashboard.enabled: true)
> ```
>
> This distinction matters: when you create a Rollout for your app, it
> lives in `argo-rollouts-demo` namespace alongside your pods and
> services. When you create an Argo CD Application for the same app, the
> Application object lives in `argocd` namespace — separate from the
> workload it manages.

### Standard Install vs Namespace-Scoped Install

Both the Helm chart and the raw manifest support two installation scopes.
Understanding which scope to use is independent of which install method
you choose.

**Standard install (used in this demo):**
Grants the controller a `ClusterRole` — it can watch and manage Rollout
resources in every namespace. This is the right choice when you have
cluster-admin rights and want one central controller for all teams.

**Namespace-scoped install (multi-tenant clusters):**
Grants the controller a `Role` scoped to a single namespace only. The
use case is a multi-tenant cluster where a team has namespace-admin rights
but not cluster-admin — they cannot create ClusterRoles or install
cluster-scoped resources. The namespace-scoped controller manages Rollouts
only in its own namespace.

**Understanding the CRD vs Controller split — why they can exist separately:**

A CRD and a controller are two completely different things, and
understanding this difference is essential to understanding why
namespace-scoped installs work the way they do.

A **CRD** (Custom Resource Definition) is a schema — it tells the
Kubernetes API server "a new resource type called Rollout exists and
has these fields." Once a CRD is installed, *anyone* in any namespace
can create a Rollout object and `kubectl` will accept it. The CRD
itself does nothing — it just enables the object type to exist in the
cluster.

A **controller** is the software that *watches* for objects of that
type and acts on them. Without a controller running, a Rollout object
you create just sits in etcd doing nothing — no pods get created, no
ReplicaSets get managed, no traffic gets shifted. The controller is
what gives the CRD its behaviour.

This separation means CRDs and controllers can be installed at
different times by different people with different permissions:

```
CRD install:        cluster-scoped operation
                    affects the entire cluster's API
                    requires: cluster-admin
                    done once — all teams share the same CRD schema

Controller install: can be cluster-scoped (standard) OR
                    namespace-scoped (one controller per namespace)
                    standard requires: cluster-admin
                    namespace-scoped requires: namespace-admin only
```

**Concrete multi-tenant scenario:**

A company has 5 product teams (Team A, B, C, D, E) sharing one EKS
cluster. Each team has its own namespace and namespace-admin rights.
None of them have cluster-admin.

```
Platform team (cluster-admin) — done once:
  Installs 5 CRDs cluster-wide
  → Now every team can CREATE Rollout objects in their namespace
  → But no controller is watching them yet — nothing happens

Team A (namespace-admin for namespace-a):
  Installs Argo Rollouts controller scoped to namespace-a only
  → Controller watches namespace-a Rollouts and manages them
  → Cannot see or touch namespace-b, namespace-c etc.

Team B (namespace-admin for namespace-b):
  Installs Argo Rollouts controller scoped to namespace-b only
  → Complete isolation from Team A's controller

Result:
  Each team manages their own rollouts independently
  No team needed cluster-admin beyond the initial CRD install
  A bug in Team A's rollout cannot affect Team B
  Platform team does not need to manage each team's controller
```

This is different from Argo CD, which does not natively support
namespace-scoped installation — one Argo CD instance manages all
namespaces with cluster-admin rights, and teams are isolated via
AppProjects rather than separate controller instances.

**How each install method handles the standard vs namespace-scoped split:**

```
Standard install — Helm (this demo):
  helm install argo-rollouts argo/argo-rollouts
    → ClusterRole (watches all namespaces)
    → 5 CRDs installed automatically by Helm
  One controller → manages all teams' Rollouts
  Requires: cluster-admin

Standard install — Raw manifest:
  kubectl apply -f install.yaml
    → ClusterRole (watches all namespaces)
    → 5 CRDs bundled in the same file
  One controller → manages all teams' Rollouts
  Requires: cluster-admin

Namespace-scoped install — Helm (multi-tenant):
  Step 1 — platform team (cluster-admin):
    helm install argo-rollouts argo/argo-rollouts \
      --set installCRDs=true   ← CRDs installed once cluster-wide
  Step 2 — application team (namespace-admin only):
    helm install argo-rollouts argo/argo-rollouts \
      --set installMode=namespace \
      --set installCRDs=false  ← CRDs already installed by platform team
    → Role scoped to one namespace only
  Each team → manages only their own Rollouts
  Requires: namespace-admin only (after platform team installs CRDs)

Namespace-scoped install — Raw manifest (multi-tenant):
  Step 1 — platform team (cluster-admin):
    kubectl apply -k https://github.com/argoproj/argo-rollouts/manifests/crds?ref=stable
    → 5 CRDs cluster-wide, done once
  Step 2 — application team (namespace-admin only):
    kubectl apply -n my-team-namespace -f namespace-install.yaml
    → Role scoped to my-team-namespace only, no CRDs
  Each team → manages only their own Rollouts
  Requires: namespace-admin only (after platform team installs CRDs)
```

This demo runs on a local Minikube cluster where you have full
cluster-admin rights. Use the standard install. The namespace-scoped
pattern is documented here so you recognise it when you encounter it
on a shared EKS or GKE cluster at work.

### The kubectl Plugin

The `kubectl argo rollouts` plugin is a CLI extension that adds rollout-aware
commands to `kubectl`. It is **optional** — you can manage Rollouts with
plain `kubectl apply` and `kubectl get` — but the plugin provides:

- `get rollout <name> --watch` — a live dashboard showing rollout progress,
  revision history, pod status, and analysis results in a tree view
- `promote <name>` — advance a paused rollout to the next step
- `abort <name>` — abort a rollout and revert to stable
- `retry rollout <name>` — retry a failed/aborted rollout
- `set image <name> <container>=<image>` — update image without editing YAML
- `dashboard` — start the web UI dashboard locally (raw manifest installs)

All plugin commands interact with the Kubernetes API server using your
existing kubeconfig. No separate credentials or configuration needed.

### The Web UI Dashboard

The Argo Rollouts dashboard is a web interface that shows:

```
Main view (all rollouts):
  ├── List of all Rollout resources across all namespaces
  ├── Status of each rollout (Healthy, Progressing, Paused, Degraded)
  └── Quick actions (promote, abort, restart)

Rollout detail view (single rollout):
  ├── Current strategy and step progress
  ├── Traffic weight (canary % vs stable %)
  ├── Revision tree — which ReplicaSet is canary, which is stable
  ├── Pod list per revision with individual pod status
  ├── Analysis run results (if analysis is configured)
  └── Action buttons: Promote, Abort, Restart, Promote Full
```

**How the dashboard is deployed differs between install methods:**

With the **raw manifest install**, the dashboard is built into the
controller pod — there is no separate dashboard Service or Deployment.
You access it by running `kubectl argo rollouts dashboard`, which
port-forwards directly to the controller pod on demand. No persistent
Service exists.

With the **Helm install** and `dashboard.enabled: true` (used in this
demo), a separate `argo-rollouts-dashboard` Deployment and Service are
created alongside the controller. The dashboard runs in its own pod and
is accessible via `kubectl port-forward svc/argo-rollouts-dashboard`.
You do not need to run `kubectl argo rollouts dashboard` — the Service
is always available without a plugin command.

```
Raw manifest install:
  argo-rollouts pod (controller + dashboard built-in)
    → access via: kubectl argo rollouts dashboard
    → port-forwards to controller pod on demand
    → stops when the command stops

Helm install (dashboard.enabled: true):
  argo-rollouts pod (controller only)
  argo-rollouts-dashboard pod (separate, always running)
  svc/argo-rollouts-dashboard (ClusterIP Service)
    → access via: kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
    → persistent — does not require a plugin command running
```

Both approaches serve the same UI at the same port (3100).

---

## Folder Structure

```
02-install-rollouts/
├── src/
│    └── values.yaml     ← Helm values file
└── README.md            ← This file
```

---

## Step 1: Start Minikube

```bash
minikube start
```

**Verify:**
```bash
kubectl get nodes
```

**Expected:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   Xs    v1.3x.x
```

---

## Step 2: Install Argo Rollouts Controller

There are two supported ways to install the Argo Rollouts controller.
**This demo uses Helm (Step 2a).** Step 2b is documented as the
alternative — follow one path and skip the other.

The Helm-based install is the recommended approach for production and for
teams who want version pinning, values customisation, and a clean upgrade
path. It uses the same official Argo Helm repository used for Argo CD.

**When to use Helm install over the raw manifest:**
- You want to pin the version explicitly and upgrade on your own schedule
- You want to customise the installation (replica count, resource limits,
  dashboard enabled/disabled, metrics port, plugin configuration)
- You want `helm upgrade` / `helm rollback` as your change management path
  rather than re-applying a raw manifest URL
- Your team already manages other tools via Helm and wants consistency

**When the raw manifest is sufficient:**
- Learning and local development — fastest path to a running controller
- You do not need any custom configuration
- You want to follow the official Argo Rollouts getting-started docs exactly

Both install methods produce an identical running controller. The difference
is in how you manage it going forward.

---

### Step 2a: Install using Helm ← Use this

**Step 2a-1: Add the Argo Helm repository**

The `argo` Helm repository at `https://argoproj.github.io/argo-helm` is
the official Argo project chart repository — the same repo used for Argo
CD. It hosts charts for all four Argo tools maintained by the same team.

```bash
# Add the official Argo Helm repository (same repo used for Argo CD)
helm repo add argo https://argoproj.github.io/argo-helm

# Update the local index
helm repo update

# Verify the repo was added
helm repo list
```

**Expected:**
```
NAME    URL
argo    https://argoproj.github.io/argo-helm
```

> If you already added the `argo` repo during your Argo CD series,
> `helm repo add` returns `"argo" already exists with the same
> configuration`. That is expected — run `helm repo update` to
> refresh the index and proceed.

---

**Step 2a-2: Select the chart version**

```bash
helm search repo argo/argo-rollouts --versions | head -5
```

The `APP VERSION` column shows the Argo Rollouts controller version.
The `CHART VERSION` is what you pass to `--version`. Note the chart
version and app version are different numbers — pin the chart version.

```
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
argo/argo-rollouts      2.40.9          v1.9.0          A Helm chart for Argo Rollouts
argo/argo-rollouts      2.40.8          v1.9.0          A Helm chart for Argo Rollouts
argo/argo-rollouts      2.40.7          v1.8.4          A Helm chart for Argo Rollouts
argo/argo-rollouts      2.40.6          v1.8.4          A Helm chart for Argo Rollouts
```

> **Why pin the version?** Unpinned installs silently pick up new
> versions on the next `helm repo update` + reinstall. Pinning makes
> every upgrade a deliberate decision — the same reason your Argo CD
> install uses a pinned version.

---

**Step 2a-3: Create the Helm values file**

The values file enables the dashboard as a separate Service and sets
sensible resource limits for local development.

**Create `src/values.yaml`:**
```yaml
# values.yaml — Argo Rollouts Helm configuration for this demo series
dashboard:
  # Creates a separate argo-rollouts-dashboard Deployment and Service.
  # Access via: kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
  # Without this, the dashboard is only accessible via the plugin command.
  enabled: true

controller:
  # Keep at 1 for local development. Increase to 2+ for HA in production.
  replicas: 1

  metrics:
    # Creates the argo-rollouts-metrics Service on port 8090.
    # This Service exposes the controller's Prometheus /metrics endpoint so that
    # a monitoring stack (e.g. kube-prometheus-stack in Demo-09) can scrape it.
    # Without this, the controller still exposes /metrics on port 8090 at the
    # pod level — but there is no Kubernetes Service in front of it, so
    # Prometheus cannot discover or scrape it via standard service discovery.
    # Set to false if you have no Prometheus stack — it has no effect otherwise.
    enabled: true

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

---

**Step 2a-4: Install**

```bash
cd argo-rollouts/02-install-rollouts/src

kubectl create namespace argo-rollouts

helm install argo-rollouts argo/argo-rollouts \
  --namespace argo-rollouts \
  --version <chart-version> \
  --values values.yaml
```

Replace `<chart-version>` with the latest stable version from Step 2a-2.

---

### Step 2b: Install using Raw Manifest ← Alternative only

Use this path only if you do not have Helm installed.

```bash
kubectl create namespace argo-rollouts

kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

This single command installs the controller, all 5 CRDs, ServiceAccount,
ClusterRole, and ClusterRoleBinding. No version is pinned — it pulls the
latest stable release at the time of the command.

**Wait for the controller to be ready:**

```bash
kubectl -n argo-rollouts rollout status deployment/argo-rollouts
```

**Expected:**
```
deployment "argo-rollouts" successfully rolled out
```

> **Dashboard access with raw manifest:** There is no separate dashboard
> pod or Service. The dashboard is accessed on demand using the plugin:
> `kubectl argo rollouts dashboard`. This is covered in Step 5.

---

## Step 3: Verify Installation

This step is common to both install methods. Run all checks regardless
of whether you used Helm or the raw manifest.

### Step 3-1: Verify pods

```bash
kubectl get pods -n argo-rollouts
```

**Expected (Helm install with dashboard.enabled: true):**
```
NAME                                        READY   STATUS    RESTARTS   AGE
argo-rollouts-xxxxxxxxxx-xxxxx              1/1     Running   0          Xs
argo-rollouts-dashboard-xxxxxxxxxx-xxxxx    1/1     Running   0          Xs
```

**Expected (raw manifest install):**
```
NAME                             READY   STATUS    RESTARTS   AGE
argo-rollouts-xxxxxxxxxx-xxxxx   1/1     Running   0          Xs
```

**What each pod does — mapped to the architecture diagram:**

| Pod | What it does |
|---|---|
| `argo-rollouts-*` | The controller — watches Rollout CRDs cluster-wide, manages ReplicaSets, runs analysis, shifts traffic weights |
| `argo-rollouts-dashboard-*` | Helm only — serves the web UI on port 3100, reads live Rollout state from the Kubernetes API. Not present in raw manifest install |

---

### Step 3-2: Verify services

```bash
kubectl get svc -n argo-rollouts
```

**Expected (Helm install with `dashboard.enabled: true` and `controller.metrics.enabled: true`):**
```
NAME                        TYPE        CLUSTER-IP       PORT(S)    AGE
argo-rollouts-dashboard     ClusterIP   10.x.x.x         3100/TCP   Xs
argo-rollouts-metrics       ClusterIP   10.x.x.x         8090/TCP   Xs
```

**Expected (Helm install with only `dashboard.enabled: true`, metrics not enabled):**
```
NAME                        TYPE        CLUSTER-IP       PORT(S)    AGE
argo-rollouts-dashboard     ClusterIP   10.x.x.x         3100/TCP   Xs
```

**Expected (raw manifest install):**
```
No resources found in argo-rollouts namespace.
```

> **Why no metrics Service with raw manifest?** The raw `install.yaml`
> does not create a Service for the metrics port. The controller pod
> exposes `/metrics` on port 8090 at the pod level, but without a
> Kubernetes Service in front of it, Prometheus cannot discover or
> scrape it via standard service discovery. If you use the raw manifest
> and want Prometheus scraping, you must add the pod-level annotations
> manually (covered in Demo-09):
> ```yaml
> annotations:
>   prometheus.io/scrape: "true"
>   prometheus.io/path: /metrics
>   prometheus.io/port: "8090"
> ```

**What each service does:**

| Service | Port | Created by | What it does |
|---|---|---|---|
| `argo-rollouts-dashboard` | 3100 | Helm (`dashboard.enabled: true`) | ClusterIP Service fronting the dashboard pod. Port-forward to this for web UI access |
| `argo-rollouts-metrics` | 8090 | Helm (`controller.metrics.enabled: true`) | Exposes the controller's Prometheus `/metrics` endpoint as a Kubernetes Service. Required for monitoring stacks to scrape controller health, queue depth, and reconciliation metrics via service discovery |

> **Argo CD comparison:**
> Argo CD exposes `argocd-server` as a Service for UI and API access,
> and `argocd-metrics` / `argocd-repo-server-metrics` for Prometheus
> scraping. Argo Rollouts follows the same pattern — a Service for the
> UI (Helm + dashboard.enabled) and a metrics Service for observability
> (Helm + metrics.enabled). Both are opt-in with Helm.

---

### Step 3-3: Verify CRDs

```bash
kubectl get crd | grep argoproj
```

**Expected:**
```
analysisruns.argoproj.io                    <timestamp>
analysistemplates.argoproj.io               <timestamp>
clusteranalysistemplates.argoproj.io        <timestamp>
experiments.argoproj.io                     <timestamp>
rollouts.argoproj.io                        <timestamp>
```

All 5 CRDs must be present. If any are missing the controller will not
function correctly.

**What each CRD is and when you use it in this series:**

| CRD | What it is | Used in |
|---|---|---|
| `rollouts.argoproj.io` | The core resource — replaces `Deployment` for workloads that need progressive delivery. Defines the pod template, replica count, and release strategy (canary or blue-green). You write one per application you want to roll out progressively. | Demo-03 onwards |
| `analysistemplates.argoproj.io` | A reusable definition of how to measure whether a rollout is healthy — which Prometheus query to run, which Kubernetes Job to execute, what threshold counts as pass or fail. Defined once, referenced by many Rollouts. Namespace-scoped. | Demo-07 onwards |
| `clusteranalysistemplates.argoproj.io` | Same as AnalysisTemplate but cluster-scoped — shared across all namespaces. A platform team defines success criteria once; every team's Rollout references it. | Demo-07 onwards |
| `analysisruns.argoproj.io` | A per-rollout execution of an AnalysisTemplate. Created automatically by Argo Rollouts when a rollout step triggers analysis. Shows the live result — pass, fail, inconclusive. You do not create these manually. | Demo-07 onwards (created automatically) |
| `experiments.argoproj.io` | Runs multiple versions simultaneously for A/B testing. Advanced pattern — not used in this series but installed by default. | Not covered in this series |

> **Argo CD comparison — CRDs and where objects live:**
>
> ```
> Argo CD CRDs (cluster-wide schema):
>   applications.argoproj.io
>   appprojects.argoproj.io
>   applicationsets.argoproj.io
>
>   → Application objects live in the argocd namespace
>   → The workloads ArgoCD deploys live in your app namespace
>
> Argo Rollouts CRDs (cluster-wide schema):
>   rollouts.argoproj.io
>   analysistemplates.argoproj.io
>   analysisruns.argoproj.io
>   clusteranalysistemplates.argoproj.io
>   experiments.argoproj.io
>
>   → Rollout objects live IN your app namespace (e.g. argo-rollouts-demo)
>   → AnalysisTemplates live in your app namespace (namespace-scoped)
>   → ClusterAnalysisTemplates live cluster-wide (no namespace)
>   → AnalysisRuns are created automatically in your app namespace
> ```
>
> The key difference: Argo CD Application objects always live in the
> `argocd` namespace — separate from the workloads they manage. Argo
> Rollouts objects live alongside the workloads they manage in the same
> app namespace. This reflects the different roles: ArgoCD is a
> platform-level tool managing many apps from one place; Argo Rollouts
> is an application-level tool living with each app it controls.

---

### Step 3-4: Verify RBAC resources

The Helm install creates significantly more RBAC resources than just one
ClusterRole. Run the following to see the complete picture:

```bash
# All ServiceAccounts in the namespace
kubectl get serviceaccounts -n argo-rollouts
```

**Expected (Helm install):**
```
NAME                      SECRETS   AGE
argo-rollouts             0         Xs
argo-rollouts-dashboard   0         Xs
default                   0         Xs
```

**Expected (raw manifest install):**
```
NAME              SECRETS   AGE
argo-rollouts     0         Xs
default           0         Xs
```

```bash
# All ClusterRoles related to Argo Rollouts
kubectl get clusterrole | grep rollout
```

**Expected (Helm install):**
```
argo-rollouts                         <timestamp>
argo-rollouts-aggregate-to-admin      <timestamp>
argo-rollouts-aggregate-to-edit       <timestamp>
argo-rollouts-aggregate-to-view       <timestamp>
argo-rollouts-dashboard               <timestamp>
```

**Expected (raw manifest install):**
```
argo-rollouts                         <timestamp>
argo-rollouts-aggregate-to-admin      <timestamp>
argo-rollouts-aggregate-to-edit       <timestamp>
argo-rollouts-aggregate-to-view       <timestamp>
```

```bash
# All ClusterRoleBindings related to Argo Rollouts
kubectl get clusterrolebinding | grep rollout
```

**Expected (Helm install):**
```
argo-rollouts             ClusterRole/argo-rollouts             Xs
argo-rollouts-dashboard   ClusterRole/argo-rollouts-dashboard   Xs
```

**Expected (raw manifest install):**
```
argo-rollouts   ClusterRole/argo-rollouts   Xs
```

**What each RBAC resource does:**

| Resource | Install method | What it does |
|---|---|---|
| `ServiceAccount: argo-rollouts` | Both | Identity for the controller pod. All Kubernetes API calls from the controller authenticate using this SA |
| `ServiceAccount: argo-rollouts-dashboard` | Helm only | Identity for the dashboard pod. The dashboard needs read access to Rollout resources — it uses its own SA, not the controller's |
| `ClusterRole: argo-rollouts` | Both | Full permissions for the controller — read/write Rollouts, ReplicaSets, Services, Pods, Secrets, AnalysisRuns, AnalysisTemplates, and more across all namespaces |
| `ClusterRole: argo-rollouts-aggregate-to-admin` | Both | RBAC aggregation — adds Rollout management permissions to the built-in `admin` ClusterRole |
| `ClusterRole: argo-rollouts-aggregate-to-edit` | Both | RBAC aggregation — adds Rollout management permissions to the built-in `edit` ClusterRole |
| `ClusterRole: argo-rollouts-aggregate-to-view` | Both | RBAC aggregation — adds Rollout read permissions to the built-in `view` ClusterRole |
| `ClusterRole: argo-rollouts-dashboard` | Helm only | Read-only permissions for the dashboard pod — list/get Rollouts, ReplicaSets, AnalysisRuns etc. |
| `ClusterRoleBinding: argo-rollouts` | Both | Binds `ServiceAccount: argo-rollouts` to `ClusterRole: argo-rollouts` |
| `ClusterRoleBinding: argo-rollouts-dashboard` | Helm only | Binds `ServiceAccount: argo-rollouts-dashboard` to `ClusterRole: argo-rollouts-dashboard` |

**The three `aggregate-to-*` ClusterRoles explained:**

These use Kubernetes RBAC aggregation — a mechanism where rules from one
ClusterRole are automatically merged into another ClusterRole. The built-in
`admin`, `edit`, and `view` ClusterRoles have aggregation labels that Argo
Rollouts uses to inject Rollout-specific rules into them.

```
Without aggregate-to-* roles:
  A user with namespace admin rights can manage Deployments, Services, Pods
  → but gets 403 Forbidden when trying to kubectl get rollout or kubectl apply rollout.yaml
  → Rollout is a custom resource — not included in the built-in admin role

With aggregate-to-* roles (installed by default):
  A user with namespace admin rights automatically inherits:
    → argo-rollouts-aggregate-to-admin rules (via label matching)
    → can manage Rollouts with no extra RBAC setup needed
  A user with view rights automatically inherits:
    → argo-rollouts-aggregate-to-view rules
    → can read Rollout status without any extra configuration
```

This means you do not need to write custom RBAC for your team members to
access Rollouts — their existing namespace permissions automatically extend
to cover Rollout resources.

> **Inspect the full controller permissions:**
> ```bash
> kubectl describe clusterrole argo-rollouts
> ```
> You will see the complete list of API groups and resources — this is
> the definitive answer to "what can the controller do in my cluster?"

> **Argo CD comparison:**
> Argo CD also ships aggregate ClusterRoles
> (`argocd-application-controller`, `argocd-server` etc.) but its RBAC
> model is more complex — it has its own internal RBAC layer (configured
> in `argocd-rbac-cm`) separate from Kubernetes RBAC. Argo Rollouts has
> no internal RBAC layer — it relies entirely on Kubernetes RBAC.
> The aggregate roles are the mechanism that makes Kubernetes RBAC
> "just work" for Rollout access without extra configuration.

---

### Step 3-5: Verify Helm release (Helm install only)

```bash
helm list -n argo-rollouts
```

**Expected:**
```
NAME             NAMESPACE       REVISION  STATUS    CHART                    APP VERSION
argo-rollouts    argo-rollouts   1         deployed  argo-rollouts-2.40.9     v1.9.0
```

> Skip this check if you used the raw manifest install — no Helm
> release exists in that case.

---

## Step 4: Install the kubectl Argo Rollouts Plugin

The plugin is a single binary. Install it based on your operating system.

### macOS (Apple Silicon)
```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-arm64
chmod +x kubectl-argo-rollouts-darwin-arm64
sudo mv kubectl-argo-rollouts-darwin-arm64 /usr/local/bin/kubectl-argo-rollouts
```

### macOS (Intel)
```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64
chmod +x kubectl-argo-rollouts-darwin-amd64
sudo mv kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### Linux (amd64)
```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### Windows (PowerShell)
```powershell
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64
mv kubectl-argo-rollouts-windows-amd64 kubectl-argo-rollouts.exe
# Move to a directory on your PATH
```

**Verify the plugin is installed:**

```bash
kubectl argo rollouts version
```

**Expected:**
```
kubectl-argo-rollouts: v1.9.x+xxxxxxx
  BuildDate: xxxx-xx-xxTxx:xx:xxZ
  GitCommit: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  GitTreeState: clean
  GoVersion: go1.xx
  Compiler: gc
  Platform: darwin/arm64
```

**Verify plugin commands are available:**

```bash
kubectl argo rollouts --help
```

**Expected output (partial):**
```
Available Commands:
  abort          Abort a rollout
  dashboard      Starts the Argo Rollouts Dashboard
  get            Get details about rollouts and experiments
  list           List rollouts or experiments
  pause          Pause a rollout
  promote        Promote a rollout
  restart        Restart the pods of a rollout
  retry          Retry a rollout or experiment
  set            Update various settings on resources
  status         Show the status of a rollout
  undo           Undo a rollout
  version        Print version
```

---

## Step 5: Access the Web UI Dashboard

How you access the dashboard depends on which install method you used.

### Step 5a: Helm install — port-forward the dashboard Service

The Helm install with `dashboard.enabled: true` created a dedicated
`argo-rollouts-dashboard` Service. Port-forward to it:

```bash
kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
```

**Expected output:**
```
Forwarding from 127.0.0.1:3100 -> 3100
Forwarding from [::1]:3100 -> 3100
```

Open your browser: **http://localhost:3100/rollouts**

---

### Step 5b: Raw manifest install — use the plugin command

```bash
kubectl argo rollouts dashboard
```

**Expected output:**
```
INFO[0000] Argo Rollouts Dashboard is now available at http://localhost:3100/rollouts
```

Open your browser: **http://localhost:3100/rollouts**

---

At this point the dashboard shows an empty list — no Rollout resources
exist yet. This is expected.

```
Argo Rollouts Dashboard
─────────────────────────────
Namespace: All namespaces ▼

  No rollouts found.
  Create a rollout to get started.
```

**Key observation:** The dashboard lists Rollouts across all namespaces
by default. The namespace dropdown lets you filter to a specific namespace.
From Demo-03 onwards every Rollout you create appears here automatically —
no configuration needed.

Leave this terminal running. Open a new terminal for subsequent commands.

---

## Step 6: Familiarise with the Argo Rollouts UI

> **Note:** The dashboard is currently empty — no Rollout resources exist
> yet. Read through this section now to understand the layout, then
> **come back here after completing Demo-03** when you have a real
> rollout running. Everything described below will be populated and
> interactive at that point.

The Argo Rollouts dashboard has two levels: the main list view and the
rollout detail view.

### Main List View

The main view at `http://localhost:3100/rollouts` shows all Rollout
resources across all namespaces (or the selected namespace if filtered).

```
┌─────────────────────────────────────────────────────────────────────┐
│  Argo Rollouts                            Namespace: All ▼          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  simple-color-app          argo-rollouts-demo    ✔ Healthy   │   │
│  │  Strategy: Canary  │  Image: podinfo:blue  │  Replicas: 5/5  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**What each element shows:**

| Element | What it tells you |
|---|---|
| Rollout name | The name of the Rollout resource — same as what you see in `kubectl argo rollouts list rollouts` |
| Namespace badge | Which namespace the Rollout lives in — useful when viewing all namespaces |
| Status badge | Overall health: `✔ Healthy`, `⟳ Progressing`, `⏸ Paused`, `✖ Degraded` |
| Strategy | Canary or BlueGreen — set by `spec.strategy` in your rollout manifest |
| Image | The current stable image tag — what 100% of production traffic is running |
| Replicas | `ready/desired` — how many pods are running vs how many are wanted |

**Namespace filter:** The dropdown at the top right filters the list to
a specific namespace. When working with multiple demo namespaces, filter
to `argo-rollouts-demo` to see only the rollouts from this series.

> **Argo CD comparison:**
> The Argo CD main view shows Applications with their sync status
> (Synced/OutOfSync) and health status (Healthy/Degraded/Progressing).
> Argo Rollouts shows Rollouts with their rollout status and current
> image. Both are single-pane-of-glass views for their respective
> concerns — Argo CD for what is deployed, Argo Rollouts for how it is
> being released.

### Rollout Detail View

Click any rollout name to open the detail view. This is the view you
will use most during Demos 03–11.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  simple-color-app                                    ⏸ Paused            │
│  Namespace: argo-rollouts-demo  │  Strategy: Canary                      │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────┐                     │
│  │  ACTIONS                                        │                     │
│  │  [Promote]  [Promote-Full]  [Abort]  [Restart]  │                     │
│  └─────────────────────────────────────────────────┘                     │
│                                                                          │
│  CANARY STEPS                                                            │
│  ● Step 1: setWeight 20%    ✔ complete                                   │
│  ● Step 2: pause {}         ⏸ waiting for promotion    ← current step    │
│                                                                          │
│  REVISIONS                                                               │
│  ┌─────────────────────────────────────────────────────────────┐         │
│  │  Revision 2  [canary]   podinfo:blue   1 pod   ✔ Healthy    │         │
│  └─────────────────────────────────────────────────────────────┘         │
│  ┌─────────────────────────────────────────────────────────────┐         │
│  │  Revision 1  [stable]   podinfo:red    4 pods  ✔ Healthy    │         │
│  └─────────────────────────────────────────────────────────────┘         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Key panels:**

**Actions bar** — The four buttons at the top are the primary way to
interact with a paused rollout:

| Button | What it does | When to use it |
|---|---|---|
| `Promote` | Advances the rollout by one step — executes the next step in the canary steps list | When you have manually verified the canary and want to continue |
| `Promote-Full` | Skips all remaining steps and immediately promotes the canary to 100% | When you are confident and want to skip remaining pauses or timed waits |
| `Abort` | Stops the rollout and reverts all traffic to the stable version | When you spot a problem in the canary and want to roll back immediately |
| `Restart` | Restarts all pods in the rollout (equivalent to `kubectl rollout restart`) | When pods are stale or need a fresh start without an image change |

**Canary Steps panel** — Shows each step defined in `spec.strategy.canary.steps`
with its status. Completed steps show a tick. The currently active step
is highlighted. For a `pause: {}` step, this panel shows it is waiting —
this is where you see *why* the rollout is paused rather than wondering
if something is wrong.

**Revisions panel** — Shows each ReplicaSet currently managed by this
rollout. During a canary, two revisions are visible: the canary (new
version, fewer pods) and the stable (current production version, more
pods). After full promotion, only one revision is shown as stable and
the previous revision is shown as ScaledDown.

> **Argo CD comparison:**
> Argo CD's Application detail view shows a resource tree of everything
> the Application manages (Deployments, Services, Pods, ConfigMaps etc.)
> with their sync and health status. Argo Rollouts' detail view is
> focused specifically on the release lifecycle — steps, revisions, and
> traffic weights. They complement each other: when both are in use
> (proj-02), you use Argo CD to see what is deployed and Argo Rollouts
> to see how the release is progressing.

---

## Step 7: CLI Dashboard — Reference Card

The CLI dashboard command `kubectl argo rollouts get rollout <name> --watch`
produces a live-updating tree view of a rollout. This is the tool you will
use most throughout this series.

> **The full output format with a real rollout is explained in Demo-03**
> once you have a running rollout to look at. This reference card covers
> the symbols and fields so you are not surprised when you first see the
> output.

**Status symbols:**
```
✔  Healthy / Succeeded    — everything is good
⟳  Progressing            — rollout is in progress, pods starting or updating
⏸  Paused                 — waiting at a pause step (manual or timed)
✖  Degraded / Failed      — something is wrong, check the Message field
!  Warning                 — requires attention
•  ScaledDown             — ReplicaSet exists but has 0 pods (kept for rollback)
```

**Key fields in the header:**
```
Status:       overall rollout health
Strategy:     Canary or BlueGreen
Step:         N/total — which step is currently active
SetWeight:    the traffic weight Argo Rollouts has instructed
ActualWeight: the weight currently confirmed as active
              (these briefly differ when a traffic router is updating)
Images:       one line per active image — labelled (stable) or (canary)
Replicas:     Desired / Current / Updated / Ready / Available
```

**The resource tree** shows the rollout, its ReplicaSets, and their pods
in a hierarchy. Each revision is a separate ReplicaSet. The `canary` and
`stable` labels identify which revision is which during an update.

---

## Step 8: Verify Full Installation Readiness

Run this final check to confirm everything is ready for Demo-03.

> **Terminal note:** If the port-forward from Step 5 is still running
> in the foreground, open a new terminal tab for the commands below.

```bash
# Controller is running
kubectl get deployment argo-rollouts -n argo-rollouts
# Expected: READY 1/1

# Dashboard pod is running (Helm install only — skip for raw manifest)
kubectl get deployment argo-rollouts-dashboard -n argo-rollouts
# Expected: READY 1/1

# All 5 CRDs present
kubectl get crd | grep argoproj | wc -l
# Expected: 5

# ServiceAccounts (Helm: 2 + default, raw manifest: 1 + default)
kubectl get serviceaccounts -n argo-rollouts
# Helm expected:    argo-rollouts, argo-rollouts-dashboard, default
# Raw expected:     argo-rollouts, default

# ClusterRoles (Helm: 5, raw manifest: 4)
kubectl get clusterrole | grep rollout
# Helm expected:    argo-rollouts, argo-rollouts-aggregate-to-admin,
#                   argo-rollouts-aggregate-to-edit,
#                   argo-rollouts-aggregate-to-view, argo-rollouts-dashboard
# Raw expected:     argo-rollouts, argo-rollouts-aggregate-to-admin,
#                   argo-rollouts-aggregate-to-edit, argo-rollouts-aggregate-to-view

# ClusterRoleBindings (Helm: 2, raw manifest: 1)
kubectl get clusterrolebinding | grep rollout
# Helm expected:    argo-rollouts, argo-rollouts-dashboard
# Raw expected:     argo-rollouts

# Plugin is working
kubectl argo rollouts version
# Expected: version string printed

# No existing rollouts (clean state for Demo-03)
kubectl argo rollouts list rollouts --all-namespaces
# Expected: No resources found.

# Dashboard is accessible (requires port-forward from Step 5 still running)
curl -s -o /dev/null -w "%{http_code}" http://localhost:3100/rollouts
# Expected: 200
```

**All checks must pass before proceeding to Demo-03.**

If any CRD is missing or the controller pod is not `Running 1/1`, reinstall:

```bash
# Helm reinstall
helm uninstall argo-rollouts -n argo-rollouts
kubectl delete namespace argo-rollouts
kubectl create namespace argo-rollouts
helm install argo-rollouts argo/argo-rollouts \
  --namespace argo-rollouts \
  --version <chart-version> \
  --values values.yaml

# Raw manifest reinstall
kubectl delete namespace argo-rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

---

## Step 9: Essential CLI Commands — Familiarisation

These are the commands you will use in every demo from Demo-03 onwards.
Run each one now to confirm the plugin works and to see what each command
does against an empty cluster. Understanding the output of each command
before you have a real rollout reduces confusion later.

```bash
# List all rollouts across all namespaces
kubectl argo rollouts list rollouts --all-namespaces
# Expected now: No resources found.
# In Demo-03+: shows each rollout with its namespace, status, and strategy

# List rollouts in a specific namespace (used in every demo)
kubectl argo rollouts list rollouts -n argo-rollouts-demo
# Expected now: No resources found.

# Get rollout detail (used extensively — the main observation command)
# kubectl argo rollouts get rollout <name> -n <namespace>
# Run this in Demo-03 after creating your first rollout

# Get rollout detail with live watch (the primary tool during demos)
# kubectl argo rollouts get rollout <name> -n <namespace> --watch
# Keeps refreshing until you press Ctrl+C

# Check plugin version
kubectl argo rollouts version
# Shows installed plugin version — useful when debugging issues

# View all available commands
kubectl argo rollouts --help
```

> **Commands introduced in later demos:**
> The following commands are introduced when they become relevant —
> they are listed here so you know they exist:
>
> | Command | Demo |
> |---|---|
> | `kubectl argo rollouts promote <name>` | Demo-03 — advance past a pause step |
> | `kubectl argo rollouts promote <name> --full` | Demo-03 — skip all remaining steps |
> | `kubectl argo rollouts abort <name>` | Demo-03 — revert to stable |
> | `kubectl argo rollouts set image <name> ...` | Demo-03 — update image without editing YAML |
> | `kubectl argo rollouts undo <name>` | Demo-04 — roll back to previous revision |
> | `kubectl argo rollouts status <name>` | Demo-05 — check rollout status in scripts |
> | `kubectl argo rollouts pause <name>` | Demo-05 — pause a running rollout |

---

## Step 10:  Upgrade and Rollback (Optional)

This section is for reference — not part of the initial install. Use
it when you need to upgrade the Argo Rollouts controller in the future
or need to recover from a bad upgrade.

### Helm — Upgrade and Rollback

Helm provides first-class upgrade and rollback with full revision history:

```bash
# Upgrade to a newer chart version
helm upgrade argo-rollouts argo/argo-rollouts \
  --namespace argo-rollouts \
  --version <newer-chart-version> \
  --values values.yaml

# View full revision history with timestamps and chart versions
helm history argo-rollouts -n argo-rollouts
# Example output:
# REVISION  STATUS     CHART                  APP VERSION  DESCRIPTION
# 1         superseded argo-rollouts-2.40.9   v1.9.0       Install complete
# 2         deployed   argo-rollouts-2.40.10  v1.9.1       Upgrade complete

# Roll back to the previous revision
helm rollback argo-rollouts -n argo-rollouts

# Roll back to a specific revision number
helm rollback argo-rollouts 1 -n argo-rollouts
```

> **`helm rollback` vs `kubectl argo rollouts undo` — important
> distinction:**
> `helm rollback` rolls back the *controller installation itself* —
> the Argo Rollouts binary version and its Helm-managed configuration.
> `kubectl argo rollouts undo` rolls back a specific *application*
> Rollout resource that the controller manages — a deployment of your
> app, not the tool itself. These operate at completely different layers.
>
> ```
> helm rollback argo-rollouts          → reverts the tool version
> kubectl argo rollouts undo my-app    → reverts an app deployment
> ```

### Raw Manifest — Upgrade and Rollback

The raw manifest install has no Helm state, no revision history, and no
built-in rollback mechanism. You manage versioning yourself.

**Upgrade:** Re-apply the manifest with a newer release URL. The
`releases/latest` URL always points to the newest stable release. For
a specific version, use the versioned release URL:

```bash
# Upgrade to latest
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Upgrade to a specific version (replace v1.9.1 with desired version)
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/download/v1.9.1/install.yaml
```

**Rollback:** Re-apply the previous version's manifest URL. You are
responsible for knowing which version you were on before the upgrade.
There is no `helm history` equivalent — if you do not track this in
Git or a runbook, you will not know which version to roll back to.

```bash
# Roll back to a specific known-good version
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/download/v1.9.0/install.yaml
```

> **Why Helm is better for production upgrades:**
> The raw manifest approach is fine for learning and development where
> you always want the latest version. For production, not knowing your
> previous version under pressure is a real problem. Helm's revision
> history means you always have a one-command rollback path regardless
> of what happened during the upgrade.

---

## Cleanup

Nothing to clean up in this demo. The controller and CRDs installed here
are used by all subsequent demos (03–11). Do not remove them.

If you ever need to fully uninstall Argo Rollouts, follow this order.
Sequence matters — removing things in the wrong order leaves ReplicaSets
and pods running with no controller to manage them.

```bash
# Step 1 — Delete Rollout resources in your application namespaces first.
# The controller gracefully scales down the ReplicaSets it manages when
# a Rollout is deleted. If the controller is gone first, it cannot do
# this cleanup and the ReplicaSets become unmanaged orphans.
kubectl delete rollouts --all -n argo-rollouts-demo

# Step 2 — Remove the controller (choose the command for your install method).

# Helm uninstall:
helm uninstall argo-rollouts -n argo-rollouts

# Raw manifest uninstall:
kubectl delete -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Step 3 — Delete the namespace.
kubectl delete namespace argo-rollouts
kubectl delete namespace argo-rollouts-demo

# Step 4 — Remove the CRDs last.
# This immediately garbage-collects every Rollout, AnalysisTemplate,
# AnalysisRun, and Experiment object cluster-wide. The controller must
# already be gone before this step runs.


# list all argoproj.io CRDs - This list might include "Argo CD" + "Argo Rollouts"
kubectl get crd | grep argoproj.io

#Delete All ArgoRollouts CRDs only
kubectl delete crd analysisruns.argoproj.io
kubectl delete crd analysistemplates.argoproj.io
kubectl delete crd clusteranalysistemplates.argoproj.io
kubectl delete crd experiments.argoproj.io
kubectl delete crd rollouts.argoproj.io

#Delete All  CRDs - "Argo CD" + "Argo Rollouts"
kubectl get crd | grep argoproj.io | awk '{print $1}' | xargs kubectl delete crd

# Step 4 — verify CRDs removed
kubectl get crd | grep argoproj.io

```

> **Why Helm does not delete CRDs automatically on `helm uninstall`:**
> Helm deliberately skips CRD deletion on uninstall because CRDs are
> cluster-scoped resources that affect every team in the cluster. If
> Helm auto-deleted CRDs, a single `helm uninstall` could silently
> destroy every Rollout resource across every namespace cluster-wide
> with no warning. Helm treats this as a safety boundary — CRD deletion
> is always a manual, explicit operation regardless of install method.

> **Why CRDs must be deleted last:**
> The moment a CRD is deleted, Kubernetes immediately garbage-collects
> every resource of that type cluster-wide — no graceful shutdown, no
> ReplicaSet scale-down, no pod drain. Deleting the controller first
> gives it the opportunity to finish any in-flight cleanup before the
> schema disappears. If you delete the CRD first, the ReplicaSets and
> pods keep running with no owner and no manager — you must clean them
> up manually.

---

## Key Concepts Summary

**One controller manages all namespaces**
The Argo Rollouts controller is installed once in `argo-rollouts` namespace
and watches Rollout resources cluster-wide. You do not install a controller
per namespace.

**Both Helm and raw manifest install the same 5 CRDs and the same controller**
The install method affects upgrade management and customisation, not what
runs in the cluster. You use `Rollout` from Demo-03, `AnalysisTemplate`
and `AnalysisRun` from Demo-07 onwards — regardless of install method.

**Standard vs namespace-scoped is independent of Helm vs raw manifest**
Both install methods support both scopes. Standard = ClusterRole, CRDs
bundled. Namespace-scoped = Role only, CRDs installed separately by a
platform team. The split exists because CRDs are cluster-scoped but
controllers can be namespace-scoped — these are different things.

**The dashboard deployment model differs between install methods**
Helm with `dashboard.enabled: true` creates a persistent separate pod
and Service — access via port-forward to the Service. Raw manifest builds
the dashboard into the controller pod — access via the plugin command
which port-forwards on demand. Same UI, same port (3100), different pod
topology.

**The plugin is optional but essential for visibility**
`kubectl argo rollouts get rollout <name> --watch` is the primary tool
for observing rollout progress throughout this series. The `promote` and
`abort` commands are used whenever a rollout needs manual intervention.

**Two dashboard options serve different purposes**
Web UI (`http://localhost:3100/rollouts`) — good for visual overview,
action buttons, seeing step progress at a glance, and sharing a screen
during demos. CLI dashboard (`get rollout --watch`) — good for
terminal-first workflows, shows more detail about individual pod status
and analysis runs, scriptable.

**Argo Rollouts objects live in your app namespace — not the controller namespace**
Unlike Argo CD Application objects which live in the `argocd` namespace,
Rollout objects live in the same namespace as the application they manage.
A Rollout in `argo-rollouts-demo` namespace is a peer of the Pods and
Services it controls.

**RBAC aggregation means namespace admins can manage Rollouts without extra setup**
The three `aggregate-to-*` ClusterRoles automatically add Rollout
permissions to the built-in Kubernetes `admin`, `edit`, and `view` roles.
Any user with namespace admin rights inherits the ability to manage Rollouts
in that namespace — no custom RBAC required.

**The metrics Service is opt-in via Helm values**
`controller.metrics.enabled: true` creates the `argo-rollouts-metrics`
Service that exposes the controller's Prometheus endpoint for scraping.
Without it, the metrics port exists on the pod but is not discoverable
by Prometheus service discovery. The raw manifest install never creates
this Service — use pod annotations for scraping instead.

---

## Commands Reference

```bash
# Helm install
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-rollouts --versions | head -5
helm install argo-rollouts argo/argo-rollouts \
  --namespace argo-rollouts --version <chart-version> --values values.yaml

# Raw manifest install
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Verify — pods, services, CRDs, RBAC
kubectl get pods -n argo-rollouts
kubectl get svc -n argo-rollouts
kubectl get crd | grep argoproj
kubectl get serviceaccounts -n argo-rollouts
kubectl get clusterrole | grep rollout
kubectl get clusterrolebinding | grep rollout
helm list -n argo-rollouts  # Helm only

# Plugin — install (macOS ARM)
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-arm64
chmod +x kubectl-argo-rollouts-darwin-arm64
sudo mv kubectl-argo-rollouts-darwin-arm64 /usr/local/bin/kubectl-argo-rollouts

# Plugin — verify
kubectl argo rollouts version
kubectl argo rollouts --help

# Dashboard — Helm install
kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
# Open: http://localhost:3100/rollouts

# Dashboard — raw manifest install
kubectl argo rollouts dashboard
# Open: http://localhost:3100/rollouts

# Essential CLI commands
kubectl argo rollouts list rollouts --all-namespaces
kubectl argo rollouts list rollouts -n argo-rollouts-demo
kubectl argo rollouts get rollout <name> -n <namespace>
kubectl argo rollouts get rollout <name> -n <namespace> --watch
```

---

## What's Next

**Demo-03: Canary Basics — First Rollout**
Write your first Rollout manifest. Convert a standard podinfo Deployment
to a Rollout with a canary strategy. Deploy the initial version, trigger
an update, observe the 20% canary pause in both the web UI and CLI
dashboard, and promote the rollout to completion via both the UI and CLI.
After completing Demo-03, return to Step 6 of this demo to walk through
the UI orientation with a real rollout running.

---

