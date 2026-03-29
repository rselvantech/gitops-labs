# Demo-01: GitOps & ArgoCD Core Concepts

## Overview

Before deploying a single manifest with ArgoCD, you need a clear mental model
of what GitOps is, why it exists, and how ArgoCD fits into it. This demo covers
the foundation — the concepts every subsequent demo builds on.

There is no cluster work here. This is deliberate. Rushing into installation
without understanding the model leads to confusion about why things are done
the way they are. Every architectural decision in this course — separate repos,
no CI cluster access, pull-based sync — has a reason. This demo explains those
reasons.

**What you'll learn:**
- What GitOps is — and what it is not
- Why GitOps exists — the problems it solves over traditional CI/CD
- The four GitOps principles and what each means in practice
- Pull-based vs push-based delivery — why the difference matters
- What ArgoCD is and how it implements the GitOps model
- ArgoCD's three core components and three supporting components
- How DevOps engineers, CI systems, and ArgoCD each relate to the cluster
- What multi-cluster management means and when to use it
- What is still ahead — concepts covered in later demos

---

## Concepts

### What Is GitOps — and What It Is Not

GitOps is an **operating model** — not a tool. ArgoCD is not GitOps. FluxCD
is not GitOps. These are tools that implement the GitOps operating model, which
is why they are often called **GitOps controllers**.

GitOps is an operating model where:
- The desired state of applications and infrastructure is declared in Git
- Git is the single source of truth — not a database, not a ticket, not a chat message
- Software agents continuously reconcile the running system to match that state

```
Traditional CI/CD:              GitOps:
─────────────────               ──────────────────────────────
CI builds artifact              CI builds artifact
CI deploys to cluster    →      CI pushes config to Git
   (push model)                 ArgoCD pulls config from Git
CI has cluster access           ArgoCD deploys to cluster
                                CI has NO cluster access
```

---

### Why GitOps Exists — The Problems It Solves

**Problem 1: No audit trail for cluster changes**
In traditional CI/CD, deployments are imperative commands — `kubectl apply`,
shell scripts, Helm upgrade commands. If something breaks, the question
"what changed and when?" is hard to answer. GitOps stores every desired state
change as a Git commit — with author, timestamp, and message.

**Problem 2: Cluster drift goes undetected**
Someone runs `kubectl edit deployment` to fix an urgent issue. Six months
later, the cluster no longer matches what is in source control. No one knows.
GitOps controllers detect this drift continuously and can revert it automatically.

**Problem 3: CI systems have cluster credentials**
In push-based CD, your CI server (Jenkins, GitHub Actions) needs `kubectl`
access or a kubeconfig. These credentials are long-lived, hard to rotate, and
represent a significant blast radius if compromised. In GitOps, the CI server
only pushes to Git — ArgoCD pulls and deploys. CI never touches the cluster.

**Problem 4: Rollback is complex**
Rolling back a push-based deployment requires re-running the pipeline with an
older version or manual intervention. In GitOps, rollback is a `git revert`
— the single source of truth goes back and ArgoCD reconciles the cluster to
match.

---

### The Four GitOps Principles

**1. Declarative**
Everything is described as desired state in YAML or JSON files. Not imperative
scripts, not shell commands, not `kubectl run`. The file declares what should
exist — not how to get there.

```
Imperative:   kubectl scale deployment podinfo --replicas=3
Declarative:  spec:
                replicas: 3   ← version-controlled, auditable, repeatable
```

**2. Versioned and Immutable**
All desired state is stored in Git. Immutable here does not mean you cannot
change the files — it means you can always go back in time to any previous
state. Every change has a history. If a change breaks the system, `git revert`
restores the working state.

**3. Pulled Automatically**
The GitOps controller (ArgoCD) pulls the desired state from Git — you do not
push deployments to the controller. This inverts the traditional model and
removes the need for CI to have cluster credentials.

```
Push model:  CI server → cluster (CI has credentials)
Pull model:  ArgoCD → Git → cluster (CI has no cluster credentials)
```

**4. Continuously Reconciled**
The GitOps controller continuously compares the desired state (Git) with the
live state (cluster). If they differ, the controller corrects the drift.
This is not a one-time operation — it is a continuous loop running every
~3 minutes (configurable).

```
Git state ≠ Cluster state → ArgoCD detects → ArgoCD corrects
```

---

### Pull-Based vs Push-Based Delivery

This distinction is fundamental to understanding ArgoCD's design.

**Push-based (traditional CI/CD):**
```
Developer → Git commit → CI pipeline triggers
  → CI builds image → CI pushes image to registry
  → CI runs kubectl apply / helm upgrade → cluster updated
```
The CI server initiates the deployment. It needs cluster credentials.
If CI is compromised, the attacker has cluster access.

**Pull-based (GitOps):**
```
Developer → Git commit → CI pipeline triggers
  → CI builds image → CI pushes image to registry
  → CI updates manifest in config repo (image tag bump)
  → ArgoCD detects change → ArgoCD pulls manifest → ArgoCD applies to cluster
```
ArgoCD initiates the pull. CI has no cluster credentials — only Git access.
The cluster's security boundary is much tighter.

**Why this matters in practice:**
- CI credentials leak → attacker gets Git access, not cluster access
- ArgoCD credentials are inside the cluster — not exposed to external systems
- Separation of concerns: CI = build + test + publish. ArgoCD = deploy.

---

### What ArgoCD Is

ArgoCD is a **Kubernetes-native, pull-based GitOps controller**. It:
- Watches Git repositories for desired state (Kubernetes manifests, Helm charts,
  Kustomize overlays)
- Compares desired state with live cluster state
- Reconciles the cluster when they differ
- Enforces Git as the single source of truth

**Kubernetes-native** means ArgoCD is built using Kubernetes extension
mechanisms — Custom Resource Definitions (CRDs) and controllers. ArgoCD runs
inside the cluster as pods, extends the Kubernetes API with new resource types
(`Application`, `AppProject`, `ApplicationSet`), and uses the Kubernetes
controller pattern to continuously reconcile state.

**ArgoCD does not build. ArgoCD does not test. ArgoCD only deploys.**

```
CI responsibility:    build → test → scan → publish image
ArgoCD responsibility: pull manifest → compare → apply to cluster
```

---

### ArgoCD Architecture — Three Core Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                        ArgoCD Control Plane                          │
│                                                                      │
│  ┌──────────────┐    ┌───────────────────┐    ┌────────────────────┐ │
│  │  API Server  │    │ Repository Server │    │ App Controller     │ │
│  │              │    │                   │    │   ♥ (heart)        │ │
│  │ Entry point  │    │ Fetches desired   │    │ Watches Application│ │
│  │ for UI, CLI  │    │ state from Git    │    │ CRDs               │ │
│  │ and CI/REST  │    │ Renders manifests │    │ Compares desired   │ │
│  │              │    │ (Helm, Kustomize) │    │ vs live state      │ │
│  │              │    │ No cluster access │    │ Reconciles drift   │ │
│  └──────────────┘    └───────────────────┘    └────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

**API Server**
ArgoCD's entry point — the equivalent of the Kubernetes API server. All
interaction with ArgoCD goes through here: the UI, the CLI (`argocd` commands),
and any CI system checking deployment status via REST. It coordinates with
internal components to fulfil requests.

**Repository Server**
Responsible for everything Git-related. It authenticates to Git repositories
(using credentials you configure), fetches the desired state, and renders
manifests into plain Kubernetes YAML. This last point is important — it
understands Helm charts, Kustomize overlays, Jsonnet, and plain YAML and
converts all of them to raw manifests that the Kubernetes API understands.
The Repository Server never touches the cluster directly.

**Application Controller**
The heart of ArgoCD. It watches Application custom resources (CRDs you create
that define source repo + destination cluster), continuously compares desired
state (from Repository Server) with live state (from Kubernetes API), and
reconciles drift. This is the component that actually applies manifests to
the cluster.

---

### ArgoCD Architecture — Three Supporting Components

**Redis**
A caching layer. Caches desired state and cluster state to improve performance
without affecting reconciliation correctness. ArgoCD can function without it
but is significantly slower.

**ApplicationSet Controller**
Enables dynamic generation of multiple Application CRDs from templates. Instead
of writing one Application CRD per app, you write one ApplicationSet that
generates many Applications — based on Git directory structure, cluster list,
or a static list. Covered in Demo-12.

**Dex Server**
Handles SSO (Single Sign-On) integration — OIDC, SAML, LDAP, GitHub OAuth.
When you want your team to log into ArgoCD using their company Google/GitHub
credentials instead of a local password, Dex handles the identity federation.
Covered as an optional advanced topic.

---

### Who Accesses What — The Access Model

This is a common source of confusion. Here is the clear breakdown:

```
DevOps Engineer
  → accesses ArgoCD UI/CLI (via port-forward or LoadBalancer)
  → does NOT need Kubernetes cluster access for day-to-day ArgoCD work
  → DOES need cluster access for troubleshooting ArgoCD itself

CI System (Jenkins, GitHub Actions)
  → pushes image to container registry ✅
  → pushes manifest changes to config repo ✅
  → optionally checks deployment status via ArgoCD REST API ✅
  → NEVER deploys to Kubernetes cluster directly ❌

ArgoCD
  → reads from Git repositories (needs Git credentials)
  → writes to Kubernetes cluster (has in-cluster ServiceAccount)
  → is the ONLY system with cluster deployment credentials
```

**Why DevOps engineers don't need cluster access for ArgoCD work:**
ArgoCD is an application deployed on Kubernetes. Accessing ArgoCD does not
require accessing Kubernetes — just like accessing a web application does not
require SSH access to the server it runs on. You access ArgoCD at its URL.
If ArgoCD itself has a problem (pod crashes, resource issues), then cluster
access is needed for troubleshooting.

---

### Multi-Cluster Management

One ArgoCD installation can manage multiple Kubernetes clusters. This is
useful for organisations with many clusters (dev, staging, prod, or multiple
regions) that want a single control plane.

```
ArgoCD (installed in cluster-hub)
  ├── manages cluster-hub (local)
  ├── manages cluster-dev (remote)
  ├── manages cluster-staging (remote)
  └── manages cluster-prod (remote)
```

Each managed remote cluster requires credentials registered with ArgoCD.
The Application CRD specifies which cluster each application deploys to —
which is why the `destination.server` field exists in every Application CRD
we write.

**Security consideration:** In regulated environments, giving one ArgoCD
instance credentials for all clusters raises compliance concerns. Many
organisations choose one ArgoCD per cluster instead. The right answer
depends on your compliance requirements.

---

### What Is Still Ahead — Course Concepts Map

| Demo | Topic | Status |
|---|---|---|
| Demo-01 | GitOps & ArgoCD Core Concepts | This demo |
| Demo-02 | Installing ArgoCD & CLI/UI Familiarisation | Next |
| Demo-03 | Application CRD — first deployment | ✅ |
| Demo-04 | Private Repository Authentication | ✅ |
| Demo-05 | Production GitOps Setup | ✅ |
| Demo-06 | Sync, Pruning & Self-Healing | ✅ |
| Demo-07 | Helm Charts with ArgoCD | ✅ |
| Demo-08 | Public Helm Chart (Headlamp) | ✅ |
| Demo-09 | AppProject — Governance | ✅ |
| Demo-10 | Sync Phases & Hooks | ✅ |
| Demo-11 | App-of-Apps Pattern | Upcoming |
| Demo-12 | ApplicationSets | Upcoming |
| Demo-13 | Kustomize with ArgoCD | Upcoming |
| Demo-14 | Sync Waves | Upcoming |

**Additional concepts not yet scheduled — worth being aware of:**
- **ArgoCD Notifications** — alert Slack/PagerDuty on sync events
- **ArgoCD Image Updater** — automatically bump image tags when new images
  are pushed to registry
- **ArgoCD RBAC** — fine-grained access control within ArgoCD (who can sync
  which app, who can delete)
- **ArgoCD SSO with Dex** — integrate with company identity provider

---

## Key Concepts Summary

**GitOps is an operating model, not a tool**
ArgoCD implements GitOps — it is not synonymous with it. The principles
(Declarative, Versioned, Pulled, Reconciled) define the model. ArgoCD enforces it.

**Git is the single source of truth**
Every desired state change goes through Git. Nothing is applied to the cluster
directly. If it is not in Git, ArgoCD does not know about it.

**Pull-based inverts the security model**
CI pushes to Git. ArgoCD pulls from Git. CI never touches the cluster. The
only system with cluster deployment credentials is ArgoCD — and those
credentials never leave the cluster.

**The three core components each have one job**
API Server = entry point. Repository Server = fetch and render manifests.
Application Controller = compare and reconcile. They work together —
none of them alone can deploy anything.

**ArgoCD does not build or test**
It only deploys. Build, test, scan, publish — that is CI's job. The
handoff between CI and ArgoCD is a Git commit to the config repo.

---

## What's Next

**Demo-02: Installing ArgoCD & Familiarising with CLI and UI**
Install ArgoCD on a local Kubernetes cluster using Kind and Helm. Verify
all components are running. Access the ArgoCD UI, change the admin password,
and walk through every section of the interface. Install the ArgoCD CLI and
run the essential commands that are used throughout every subsequent demo.