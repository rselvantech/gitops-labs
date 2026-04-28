# Demo-13: ApplicationSets — Scalable GitOps Across Clusters and Environments

## Overview

Demo-12 introduced App-of-Apps — one parent Application managing child
Applications declaratively from Git. That pattern solves the manual `kubectl
apply` problem for a small, fixed set of components. But it still requires
writing one Application YAML per component per environment. For 10 microservices
across dev, staging, and prod that is 30 Application YAMLs — all manually
maintained.

ApplicationSets solve this. Instead of writing individual Application CRDs, you
write one ApplicationSet that generates them dynamically based on a **generator**
— a rule that defines how many Applications to create and for whom. Add a new
environment? Push one change. Add a new cluster? Apply a label. ApplicationSet
handles the rest.

This demo walks through all five production-relevant generators, progressing
from the simplest (List) to the most powerful (Matrix). Each step is a
self-contained demonstration that builds directly on the previous one.

> **Prerequisite:** Complete `README-cluster-setup.md` first. This README assumes
> both clusters (`us-east-ohio` and `middle-east-uae`) are registered with ArgoCD,
> labelled, and verified working. The Docker network wiring must also be active.
>
> **Quick check before starting:**
> ```bash
> argocd cluster list       # both clusters must show STATUS: Successful
> docker exec minikube nc -zv -w 3 192.168.67.2 8443   # must succeed
> ```

**What you'll learn:**
- Why ApplicationSets exist — the scaling limits of App-of-Apps
- The two building blocks of every ApplicationSet: generators and templates
- How generators define HOW MANY and FOR WHOM Applications are created
- How templates define HOW each Application looks — same spec as Application CRD
- Go templating syntax — what it is and how it works in ApplicationSets
- The `goTemplate` and `goTemplateOptions` fields — what they do and why
- All five generators in depth: List, Cluster, Git Directory, Git File, Matrix
- How to choose the right generator for a given requirement

**What you'll do:**
- Walk through all five generators end-to-end with working demos
- Push region- and environment-specific manifests to `gitops-apps-config`
- Combine generators with the Matrix generator to deploy dev/staging/prod
  across two regions — six Applications from one YAML

---

## Prerequisites

- ✅ Completed `README-cluster-setup.md` — both clusters registered, labelled,
  and verified. `argocd cluster list` shows both as `Successful`.
- ✅ ArgoCD CLI logged in: `argocd login localhost:8080 --username admin --insecure`
- ✅ Active context is `us-east-ohio`: `kubectl config use-context us-east-ohio`
- ✅ Docker network wiring active: `docker exec minikube nc -zv -w 3 192.168.67.2 8443`
- ✅ GitHub PAT with access to `gitops-apps-config` and `argocd-config`
- ✅ `gitops-apps-config` already registered with ArgoCD from Demo-10

**Verify before proceeding:**
```bash
# Both clusters registered
argocd cluster list
```

**Expected:**
```text
SERVER                          NAME              VERSION  STATUS      MESSAGE
https://kubernetes.default.svc  in-cluster        1.xx     Successful
https://192.168.67.2:8443        middle-east-uae   1.xx     Successful
```

```bash
# No existing demo-13 applications (clean state from Part 1 cleanup)
argocd app list
```

**Expected:** Empty or only unrelated applications.

**Recovery — if Docker/WSL2 restarted since Part 1:**
```bash
docker network connect middle-east-uae minikube
docker exec minikube nc -zv -w 3 192.168.67.2 8443
# Expected: Connection ... succeeded!
```

---

## Concepts

### The Demo Scenario

Imagine you are a platform engineer at a company whose web application serves
customers across two geographic regions — the United States and the Middle East.
To meet both **latency requirements** (users in each region are served by a
nearby cluster) and **data residency regulations** (customer data must stay
within the region it was collected), you run two Kubernetes clusters:

```
┌─────────────────────────────────────────────────────────┐
│                   Production Architecture                │
│                                                          │
│   US Customers                   ME Customers            │
│        │                               │                 │
│        ▼                               ▼                 │
│  ┌───────────────┐           ┌──────────────────┐        │
│  │  us-east-ohio │           │ middle-east-uae  │        │
│  │   (Ohio, US)  │           │  (UAE, ME)       │        │
│  │               │           │                  │        │
│  │  nginx app    │           │  nginx app       │        │
│  │  (same code)  │           │  (same code)     │        │
│  └───────────────┘           └──────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

The application code is identical across both clusters. Only the deployment
target differs. Without ApplicationSets, deploying the same application to
two clusters means writing and maintaining two Application CRDs — and as
environments multiply (dev/staging/prod × two clusters), that quickly becomes
dozens of YAMLs.

ApplicationSets reduce this to **one YAML** that generates all the required
Application CRDs dynamically.

---


### How the Two Clusters Are Connected — The Control Plane Model

ArgoCD is not installed on both clusters. It runs on **one cluster only** —
the **control plane cluster**. That one ArgoCD instance manages both clusters
from a central location.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     ArgoCD Control Plane Model                           │
│                                                                          │
│   GitHub (gitops-apps-config)                                                    │
│   ┌──────────────────┐                                                   │
│   │  us-east-ohio/   │                                                   │
│   │  middle-east-uae/│◄──── ArgoCD pulls desired state ────┐             │
│   │  dev/ staging/   │                                     │             │
│   │  prod/           │                                     │             │
│   └──────────────────┘                                     │             │
│                                                            │             │
│   ┌──────────────────────────────────────────────────────────────────┐   │
│   │              us-east-ohio cluster  (ArgoCD Control Plane)        │   │
│   │                                                                  │   │
│   │   ┌─────────────────────────────────────────────────────────┐    │   │
│   │   │  ArgoCD (argocd namespace)                              │    │   │
│   │   │                                                         │    │   │
│   │   │  API Server ──► Repository Server ──► App Controller    │    │   │
│   │   │                        │                     │          │    │   │
│   │   │                   pulls Git            applies to       │    │   │
│   │   │                   manifests            both clusters    │    │   │
│   │   └─────────────────────────────────────────────────────────┘    │   │
│   │                                                                  │   │
│   │   nginx pods  (deployed to us-east-ohio by ArgoCD)               │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                            │                                             │
│                            │  argocd cluster add                         │
│                            │  (registers credentials,                    │
│                            │   creates argocd-manager SA)                │
│                            ▼                                             │
│   ┌──────────────────────────────────────────────────────────────────┐   │
│   │              middle-east-uae cluster  (Target only)              │   │
│   │                                                                  │   │
│   │   argocd-manager ServiceAccount  (created during registration)   │   │
│   │   nginx pods  (deployed here remotely by ArgoCD)                 │   │
│   └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

**The connection in plain language:**

ArgoCD runs inside `us-east-ohio`. During `argocd cluster add middle-east-uae`,
ArgoCD reads your local kubeconfig to connect to `middle-east-uae` and creates
an `argocd-manager` ServiceAccount there with cluster-admin permissions. It then
stores the cluster URL, CA certificate, and ServiceAccount token as a Kubernetes
Secret in the `argocd` namespace on `us-east-ohio`.

From that point on, whenever ArgoCD syncs an Application whose destination is
`middle-east-uae`, the Application Controller uses those stored credentials to
authenticate to `middle-east-uae` and apply the manifests — without any further
input from you. Your kubeconfig is no longer involved.

---

### Why the Manifests Differ Across Region and Environment Folders

A natural question when looking at the folder structure: if the application
code is the same, why do we have `deployment.yaml` in `us-east-ohio/manifests/`,
`middle-east-uae/manifests/`, `dev/manifests/`, `staging/manifests/`, and
`prod/manifests/`? Are these unnecessary duplicates?

They are not duplicates — each folder represents different **intent** that must
be expressed in the manifests:

**Differences across regions (`us-east-ohio` vs `middle-east-uae`):**

| Field | `us-east-ohio` | `middle-east-uae` |
|---|---|---|
| `namespace` | `demo13-us-east-ohio` | `demo13-middle-east-uae` |
| `ConfigMap` → `index.html` | "Region: US East Ohio" | "Region: Middle East UAE" |

The namespace must differ because each cluster is a separate Kubernetes control
plane. The ConfigMap content differs because in a real deployment, localisation
(language, regional settings, CDN endpoints) would differ per region.

**Differences across environments (`dev`, `staging`, `prod`):**

| Field | `dev` | `staging` | `prod` |
|---|---|---|---|
| `namespace` | `demo13-dev` | `demo13-staging` | `demo13-prod` |
| `replicas` | `1` | `2` | `3` |
| `ConfigMap` → `index.html` | "Environment: dev" | "Environment: staging" | "Environment: prod" |

**Is this the real-world pattern?**

Yes — this is precisely how production GitOps works. Each environment or region
has its own folder containing manifests that express the desired state for that
specific target. The ApplicationSet generator discovers these folders and creates
one Application per folder, each deploying to its intended destination.

The alternative — a single manifest with templated values — would require
Kustomize or Helm overlays. Demo-14 covers that pattern. For this demo,
explicit per-folder manifests make the connection between generator → folder →
Application → cluster maximally visible and easy to reason about.

---

### Why ApplicationSets Exist — Beyond App-of-Apps

App-of-Apps manages Applications declared in Git. It eliminates `kubectl apply`
for child Applications. But you still write one Application YAML per component
per environment:

```
10 microservices × 3 environments = 30 Application YAMLs
30 Application YAMLs × 2 clusters = 60 Application YAMLs
→ 60 files to write, maintain, and keep consistent
→ New cluster = add 30 more files
→ New microservice = add 6 more files
```

ApplicationSets collapse this to a template and a generator rule:

```
1 ApplicationSet YAML → 60 Applications generated automatically
New cluster = update generator rule (one line change)
New microservice = add one directory in Git
```

**The Deployment analogy:**
Kubernetes Deployment creates Pods from a pod template. You do not write 10
Pod YAMLs for 10 replicas — you write one Deployment and set `replicas: 10`.

```
Deployment      + pod template → creates N Pods
ApplicationSet  + app template → creates N Applications
```

---

### The Two Building Blocks: Generators and Templates

Every ApplicationSet YAML has exactly two key sections under `spec`:

```yaml
spec:
  generators:   # WHO gets an Application, and HOW MANY
    - list:
        elements:
          - region: us-east-ohio
          - region: middle-east-uae

  template:     # WHAT each generated Application looks like
    metadata:
      name: "app1-{{.region}}"
    spec:
      source:
        path: "{{.region}}/manifests"
      destination:
        name: "{{.clusterName}}"
```

**Generators** answer: for which clusters, environments, or directories should
ArgoCD create an Application? They expose key-value variables (like `region`,
`clusterName`, `namespace`) that the template uses.

**Templates** answer: what should each Application look like? The template is
identical to a standard Application CRD spec — the same `source`, `destination`,
and `syncPolicy` fields you have used since Demo-03. Generator variables are
injected using Go template syntax `{{.variableName}}`.

The template runs once per generator output item. If the generator produces
3 items, the template runs 3 times and 3 Applications are created.

---

### Go Templating Syntax in ApplicationSets

ApplicationSets support Go's `text/template` package for variable injection.
Enabling it via `goTemplate: true` unlocks a concise syntax for injecting
generator variables.

**Core syntax:**

```yaml
{{.variableName}}                  # variable injection
{{.path.basename}}                 # nested field access
{{.metadata.labels.region}}        # deeply nested label value
{{.region | upper}}                # string function
{{default "dev" .environment}}     # default value
```

**Why Go templating over the default:**

The default ApplicationSet syntax uses `{{region}}`. Go templating uses
`{{.region}}`. The dot notation is required for nested field access like
`{{.metadata.labels.region}}` (used by the Cluster generator) — not possible
without `goTemplate: true`.

**The standard production block — always use both together:**

```yaml
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
```

`missingkey=error` causes the ApplicationSet to fail loudly if a template
references a variable the generator does not expose — instead of silently
creating Applications with empty `name` or `path` fields.

---

### The Five Generators

| Generator | Selects by | Best for |
|---|---|---|
| **List** | Inline items in the YAML | Small, fixed set of targets |
| **Cluster** | Labels on registered clusters | Dynamic cluster fleet |
| **Git Directory** | Directories in a Git repo | Microservice per directory pattern |
| **Git File** | Structured YAML files in Git | Complex parameters, version-controlled intent |
| **Matrix** | Combines two generators | Multi-environment × multi-cluster |

---

### How Cluster Labels Are Stored

When you registered clusters in Part 1 and applied labels, ArgoCD stored those
labels inside Kubernetes Secrets in the `argocd` namespace. The Cluster
generator reads these secrets to find clusters matching its label selector.

```bash
# See the secrets backing registered clusters
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
```

This is why the `--upsert` flag is required when updating labels on an
already-registered cluster — it overwrites the existing Secret.

---

### How ArgoCD Resolves the Deployment Namespace

Every Kubernetes resource that is namespace-scoped (Deployment, Service,
ConfigMap, etc.) must land in a specific namespace. There are two places
this can be specified — and understanding which one wins is important:

**1. Inside the manifest (`metadata.namespace`)**

```yaml
metadata:
  name: demo13-app
  namespace: demo13-us-east-ohio   # hardcoded in the file
```

**2. In the Application CRD (`destination.namespace`)**

```yaml
destination:
  name: in-cluster
  namespace: demo13-us-east-ohio   # set by the ApplicationSet template
```

**The rule: manifest namespace always wins over destination namespace.**

If a resource has `metadata.namespace` set, Kubernetes applies it to that
exact namespace — regardless of what `destination.namespace` says. ArgoCD
does not rewrite manifest namespaces. This means:

- `destination.namespace: demo13-us-east-ohio-gf` in the Application
- `metadata.namespace: demo13-us-east-ohio` in the manifest

→ ArgoCD tries to create the resource in `demo13-us-east-ohio` (from the
manifest), not `demo13-us-east-ohio-gf` (from the destination). If
`demo13-us-east-ohio` does not exist on that cluster, the sync fails with
`namespace "demo13-us-east-ohio" not found`.

**Without `metadata.namespace` in the manifest (the correct pattern):**

```yaml
metadata:
  name: demo13-app
  # no namespace field
```

ArgoCD deploys the resource into `destination.namespace` from the Application
CRD. The ApplicationSet template controls the namespace entirely — one manifest
works for any namespace the generator produces.

```
Generator variable  →  destination.namespace  →  resource lands here
{{.namespace}}         demo13-us-east-ohio-gf     demo13-us-east-ohio-gf  ✅
{{.namespace}}         demo13-dev                 demo13-dev              ✅
{{.namespace}}         demo13-prod-middle-east-uae demo13-prod-middle-east-uae ✅
```

**Why this matters for ApplicationSets specifically:**

ApplicationSets are designed to deploy the same manifest to many different
namespaces via generator variables. Hardcoding `metadata.namespace` in the
manifest defeats this entirely — every generated Application would fight the
manifest for namespace control. Keeping manifests namespace-free is what makes
the generator's `destination.namespace` variable actually work.

> **Rule of thumb:** Never hardcode `metadata.namespace` in manifests managed
> by ArgoCD. Let `destination.namespace` in the Application CRD own it.
> `CreateNamespace=true` in `syncOptions` handles namespace creation automatically.

---

## Folder Structure

```
13-applicationset/src/
├── gitops-apps-config/                          ← rselvantech/gitops-apps-config (private)
│   └── demo-13-applicationset/
│       ├── us-east-ohio/
│       │   └── manifests/
│       │       ├── deployment.yaml   ← namespace: demo13-us-east-ohio, US index.html
│       │       └── service.yaml
│       ├── middle-east-uae/
│       │   └── manifests/
│       │       ├── deployment.yaml   ← namespace: demo13-middle-east-uae, UAE index.html
│       │       └── service.yaml
│       ├── dev/
│       │   └── manifests/
│       │       └── deployment.yaml   ← namespace: demo13-dev, replicas: 1
│       ├── staging/
│       │   └── manifests/
│       │       └── deployment.yaml   ← namespace: demo13-staging, replicas: 2
│       ├── prod/
│       │   └── manifests/
│       │       └── deployment.yaml   ← namespace: demo13-prod, replicas: 3
│       └── clusters.yaml             ← used by Git File generator
└── argocd-config/                    ← rselvantech/argocd-config
    └── demo-13-applicationset/
        ├── 01-list-generator.yaml
        ├── 02-cluster-generator.yaml
        ├── 03-git-directory-generator.yaml
        ├── 04-git-file-generator.yaml
        └── 05-matrix-generator.yaml
```

---

## Step 1: Add All Manifests to `gitops-apps-config`

All five generators read application manifests from `gitops-apps-config`.
All manifests are pushed now — in a single step — so that each generator step
only requires applying one ApplicationSet YAML.

The folder structure directly maps to what each generator discovers:
- `us-east-ohio/manifests/` and `middle-east-uae/manifests/` → List + Cluster generators
- `dev/`, `staging/`, `prod/` → Git Directory + Matrix generators
- `clusters.yaml` → Git File generator


**Initialise local repo structure:**
```bash
cd argo-cd-basics-to-prod/13-applicationset/src
mkdir -p gitops-apps-config && cd gitops-apps-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

**Create Files:**
```bash
mkdir -p demo-13-applicationset/us-east-ohio/manifests
mkdir -p demo-13-applicationset/middle-east-uae/manifests
mkdir -p demo-13-applicationset/dev/manifests
mkdir -p demo-13-applicationset/staging/manifests
mkdir -p demo-13-applicationset/prod/manifests

touch demo-13-applicationset/us-east-ohio/manifests/deployment.yaml
touch demo-13-applicationset/us-east-ohio/manifests/service.yaml
touch demo-13-applicationset/middle-east-uae/manifests/deployment.yaml
touch demo-13-applicationset/middle-east-uae/manifests/service.yaml
touch demo-13-applicationset/dev/manifests/deployment.yaml
touch demo-13-applicationset/staging/manifests/deployment.yaml
touch demo-13-applicationset/prod/manifests/deployment.yaml
touch demo-13-applicationset/clusters.yaml

```


**`demo-13-applicationset/us-east-ohio/manifests/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo13-app
  template:
    metadata:
      labels:
        app: demo13-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: demo13-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo13-config
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Region: US East Ohio | Cluster: us-east-ohio (in-cluster)</p>
    <p>Generator: List / Cluster</p>
    </body></html>
```

**`demo-13-applicationset/us-east-ohio/manifests/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo13-svc
spec:
  selector:
    app: demo13-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-13-applicationset/middle-east-uae/manifests/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo13-app
  template:
    metadata:
      labels:
        app: demo13-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: demo13-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo13-config
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Region: Middle East UAE | Cluster: middle-east-uae</p>
    <p>Generator: List / Cluster</p>
    </body></html>
```

**`demo-13-applicationset/middle-east-uae/manifests/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo13-svc
spec:
  selector:
    app: demo13-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-13-applicationset/dev/manifests/deployment.yaml`** (1 replica):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo13-app
  template:
    metadata:
      labels:
        app: demo13-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: demo13-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo13-config
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Environment: dev | Replicas: 1</p>
    <p>Generator: Git Directory / Matrix</p>
    </body></html>
```

**`demo-13-applicationset/staging/manifests/deployment.yaml`** (2 replicas):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo13-app
  template:
    metadata:
      labels:
        app: demo13-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: demo13-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo13-config
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Environment: staging | Replicas: 2</p>
    <p>Generator: Git Directory / Matrix</p>
    </body></html>
```

**`demo-13-applicationset/prod/manifests/deployment.yaml`** (3 replicas):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo13-app
  template:
    metadata:
      labels:
        app: demo13-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: demo13-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo13-config
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Environment: prod | Replicas: 3</p>
    <p>Generator: Git Directory / Matrix</p>
    </body></html>
```

**`demo-13-applicationset/clusters.yaml`** (for Git File generator — Step 5):
```yaml
- name: us-east-ohio
  region: us-east-ohio
  clusterName: in-cluster
  namespace: demo13-us-east-ohio-gf
- name: middle-east-uae
  region: middle-east-uae
  clusterName: middle-east-uae
  namespace: demo13-middle-east-uae-gf
```

**Push all manifests:**
```bash
git add demo-13-applicationset/
git commit -m "feat: add demo-13 applicationset manifests for all five generators"
git push origin main
```

---

## Step 2: Set Up `argocd-config` Local Repo

All ApplicationSet YAMLs are stored in `argocd-config` — the same repo
ArgoCD already watches. Each generator step adds one file here.

```bash
cd argo-cd-basics-to-prod/13-applicationset/src
mkdir -p argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

---

## Step 3: Generator 1 — List Generator

### What this step demonstrates

We deploy the same nginx application to two clusters — `us-east-ohio` and
`middle-east-uae` — using one ApplicationSet YAML with a List generator.
The list of targets is defined directly inline.

### About the List Generator

The List generator takes a list of `elements` defined directly in the
ApplicationSet YAML. Each element is a set of key-value pairs you choose
freely. ArgoCD iterates over the list and runs the template once per element,
injecting the key-value pairs as template variables.

**How it works:**

```
generators.list.elements[0]:          generators.list.elements[1]:
  region: us-east-ohio                  region: middle-east-uae
  clusterName: in-cluster               clusterName: middle-east-uae
  namespace: demo13-us-east-ohio        namespace: demo13-middle-east-uae
         │                                      │
         ▼ template runs with element[0]         ▼ template runs with element[1]
Application: demo13-us-east-ohio       Application: demo13-middle-east-uae
  path: .../us-east-ohio/manifests       path: .../middle-east-uae/manifests
  destination: in-cluster                destination: middle-east-uae
```

**Key fields:**
```yaml
generators:
  - list:
      elements:
        - key1: value1    # any keys you choose — referenced as {{.key1}}
          key2: value2
        - key1: value3
          key2: value4
```

**When to use:** 2 to 5 deployment targets with a stable, known list. Getting
started with ApplicationSets — lowest cognitive overhead.

**Create `demo-13-applicationset/01-list-generator.yaml` in `argocd-config`:**


```bash
cd argo-cd-basics-to-prod/13-applicationset/src/argocd-config
touch demo-13-applicationset/02-cluster-generator.yaml
```

**`demo-13-applicationset/01-list-generator.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo13-list-generator
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
  generators:
    - list:
        elements:
          - region: us-east-ohio        # variable: {{.region}}
            clusterName: in-cluster     # variable: {{.clusterName}}
            namespace: demo13-us-east-ohio  # variable: {{.namespace}}
          - region: middle-east-uae
            clusterName: middle-east-uae
            namespace: demo13-middle-east-uae
  template:
    metadata:
      name: "demo13-{{.region}}"        # → demo13-us-east-ohio, demo13-middle-east-uae
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "demo-13-applicationset/{{.region}}/manifests"
        # → demo-13-applicationset/us-east-ohio/manifests
        # → demo-13-applicationset/middle-east-uae/manifests
      destination:
        name: "{{.clusterName}}"        # → in-cluster, middle-east-uae
        namespace: "{{.namespace}}"     # → demo13-us-east-ohio, demo13-middle-east-uae
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Variable-to-Manifest Tracing: List Generator

```
List element[0]:
  region: "us-east-ohio"
  clusterName: "in-cluster"
  namespace: "demo13-us-east-ohio"
         │  template substitution
         ▼
Application CRD:
  metadata.name: "demo13-us-east-ohio"           ← {{.region}}
  spec.source.path: ".../us-east-ohio/manifests"  ← {{.region}}
  spec.destination.name: "in-cluster"             ← {{.clusterName}}
  spec.destination.namespace: "demo13-us-east-ohio" ← {{.namespace}}
         │  ArgoCD syncs
         ▼
Manifests: us-east-ohio/manifests/deployment.yaml
  → Deployed to: in-cluster, namespace: demo13-us-east-ohio
  → nginx with "Region: US East Ohio" index.html
```

**Push and apply:**
```bash
git add demo-13-applicationset/01-list-generator.yaml
git commit -m "feat: add demo-13 list generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationset/01-list-generator.yaml
```

**Verify ApplicationSet created:**
```bash
kubectl get appset -n argocd
```

**Expected:**
```text
NAME                    AGE
demo13-list-generator   10s
```

**Watch two Applications generated:**
```bash
argocd app list
```

**Expected:**
```text
NAME                     CLUSTER           SYNC STATUS   HEALTH STATUS
demo13-us-east-ohio      in-cluster        Synced        Healthy
demo13-middle-east-uae   middle-east-uae   Synced        Healthy
```

**Verify pods on `us-east-ohio`:**
```bash
kubectl config use-context us-east-ohio
kubectl get pods -n demo13-us-east-ohio
```

**Expected:**
```text
NAME                          READY   STATUS
demo13-app-xxxxxxxxx-xxxxx    1/1     Running
```

**Verify pods on `middle-east-uae`:**
```bash
kubectl config use-context middle-east-uae
kubectl get pods -n demo13-middle-east-uae
```

**Expected:**
```text
NAME                          READY   STATUS
demo13-app-xxxxxxxxx-xxxxx    1/1     Running
```

**Access the US East Ohio deployment:**
```bash
kubectl config use-context us-east-ohio
kubectl port-forward svc/demo13-svc -n demo13-us-east-ohio 8081:80
```

Open `http://localhost:8081` — page shows "Region: US East Ohio".

**Access the Middle East UAE deployment:**
```bash
kubectl config use-context middle-east-uae
kubectl port-forward svc/demo13-svc -n demo13-middle-east-uae 8082:80
```

Open `http://localhost:8082` — page shows "Region: Middle East UAE".

**Key observation:** One ApplicationSet YAML created two Applications deployed
to two different clusters. To add a third region, add one more element to the
list and create its manifests folder — no new Application YAML needed.

**Clean up before next step:**
```bash
kubectl config use-context us-east-ohio

#Delete ApplicationSet
kubectl delete appset demo13-list-generator -n argocd

#Verify Applications are deleted
argocd app list

#Delete Namespace
kubectl delete ns demo13-us-east-ohio
kubectl config use-context middle-east-uae
kubectl delete ns demo13-middle-east-uae

kubectl config use-context us-east-ohio
```

---

## Step 4: Generator 2 — Cluster Generator

### What this step demonstrates

The same two-cluster deployment as Step 3, but targets are discovered
automatically from labels on registered clusters — not declared inline.
Adding a new cluster with the right label auto-generates an Application with
zero YAML changes.

### About the Cluster Generator

The Cluster generator scans ArgoCD's registered cluster Secrets in the `argocd`
namespace and selects those matching a label selector. The labels you applied
in Part 1 (`app=demo13`, `region=<n>`) are exactly what this generator reads.

**Template variables exposed by the Cluster generator:**

| Variable | Value | Example |
|---|---|---|
| `{{.name}}` | Cluster name as registered | `in-cluster`, `middle-east-uae` |
| `{{.server}}` | Cluster API server URL | `https://kubernetes.default.svc` |
| `{{.metadata.labels.<key>}}` | Any label on the cluster | `{{.metadata.labels.region}}` |

> `goTemplate: true` is essential here — dot-notation is required to access
> nested fields like `{{.metadata.labels.region}}`.

**When to use:** Dynamic cluster fleets where clusters are added and removed
regularly. Adding a new cluster with the right label → Application auto-generated.

```bash
cd argo-cd-basics-to-prod/13-applicationset/src/argocd-config
touch demo-13-applicationset/02-cluster-generator.yaml
```

**Create `demo-13-applicationset/02-cluster-generator.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo13-cluster-generator
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
  generators:
    - clusters:
        selector:
          matchLabels:
            app: demo13       # selects both clusters labelled in Part 1
  template:
    metadata:
      name: "demo13-{{.metadata.labels.region}}"
      # → demo13-us-east-ohio, demo13-middle-east-uae
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "demo-13-applicationset/{{.metadata.labels.region}}/manifests"
      destination:
        name: "{{.name}}"          # cluster name as registered in ArgoCD
        namespace: "demo13-{{.metadata.labels.region}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Variable-to-Cluster Tracing: Cluster Generator

```
Cluster Secret (in argocd namespace on us-east-ohio):
  data.name: "in-cluster"
  metadata.labels:
    app: "demo13"           ← matches selector → cluster included
    region: "us-east-ohio"  ← exposed as {{.metadata.labels.region}}
         │  template substitution
         ▼
Application CRD:
  metadata.name: "demo13-us-east-ohio"            ← {{.metadata.labels.region}}
  spec.source.path: ".../us-east-ohio/manifests"   ← {{.metadata.labels.region}}
  spec.destination.name: "in-cluster"              ← {{.name}}
  spec.destination.namespace: "demo13-us-east-ohio" ← {{.metadata.labels.region}}
         │  ArgoCD syncs
         ▼
Manifests: us-east-ohio/manifests/ → deployed to in-cluster

────────────────────────────────────────────────

Cluster Secret for second cluster:
  data.name: "middle-east-uae"
  metadata.labels.region: "middle-east-uae"
         ▼
Application CRD:
  metadata.name: "demo13-middle-east-uae"
  spec.source.path: ".../middle-east-uae/manifests"
  spec.destination.name: "middle-east-uae"
         ▼
Manifests: middle-east-uae/manifests/ → deployed to middle-east-uae cluster
```

**Push and apply:**
```bash
git add demo-13-applicationset/02-cluster-generator.yaml
git commit -m "feat: add demo-13 cluster generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationset/02-cluster-generator.yaml
```

**Verify two Applications generated — driven by cluster labels:**
```bash
argocd app list
```

**Expected:**
```text
NAME                     CLUSTER           SYNC STATUS   HEALTH STATUS
demo13-us-east-ohio      in-cluster        Synced        Healthy
demo13-middle-east-uae   middle-east-uae   Synced        Healthy
```

**Key observation — label-driven fleet:**
Remove the `app=demo13` label from `middle-east-uae` in ArgoCD UI (Settings
→ Clusters → Edit) and the `demo13-middle-east-uae` Application is automatically
deleted. Add the label back and it reappears — no ApplicationSet YAML touched.

**Clean up:**
```bash
kubectl delete appset demo13-cluster-generator -n argocd
```

**Clean up before next step:**
```bash
kubectl config use-context us-east-ohio

#Delete ApplicationSet
kubectl delete appset demo13-list-generator -n argocd

#Verify Applications are removed
argocd app list

#Delete Namespace
kubectl delete ns demo13-us-east-ohio
kubectl config use-context middle-east-uae
kubectl delete ns demo13-middle-east-uae

kubectl config use-context us-east-ohio
```

---

## Step 5: Generator 3 — Git Directory Generator

### What this step demonstrates

We deploy three environment versions (dev, staging, prod) to `us-east-ohio`
by discovering directories in `gitops-apps-config`. Adding a new environment
directory to Git automatically creates a new Application — no ApplicationSet
YAML change required.

### About the Git Directory Generator

The Git Directory generator scans specified path patterns in a Git repository
and creates one Application for each matching directory.

**Template variables exposed:**

| Variable | Value | Example |
|---|---|---|
| `{{.path.path}}` | Full path of discovered directory | `demo-13-applicationset/dev` |
| `{{.path.basename}}` | Directory name only | `dev` |

> **Why the `exclude` pattern is needed:** If your structure is
> `dev/manifests/deployment.yaml`, the generator discovers both `dev` and
> `dev/manifests` as separate paths — creating two Applications per environment.
> Explicitly excluding `*/manifests` prevents this.

**When to use:** Microservice-per-directory or environment-per-directory
patterns where the list of Applications should grow automatically as directories
are added.

```bash
cd argo-cd-basics-to-prod/13-applicationset/src/argocd-config
touch demo-13-applicationset/03-git-directory-generator.yaml
```

**Create `demo-13-applicationset/03-git-directory-generator.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo13-git-directory
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
  generators:
    - git:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        revision: main
        directories:
          - path: "demo-13-applicationset/*/manifests"
            exclude: true         # do NOT generate for manifests/ subdirectories
          - path: "demo-13-applicationset/*"
            exclude: false        # generate for dev, staging, prod (and region folders)
  template:
    metadata:
      name: "demo13-{{.path.basename}}"
      # → demo13-dev, demo13-staging, demo13-prod
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "{{.path.path}}/manifests"
        # → demo-13-applicationset/dev/manifests
        # → demo-13-applicationset/staging/manifests
        # → demo-13-applicationset/prod/manifests
      destination:
        name: in-cluster
        namespace: "demo13-{{.path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Variable-to-Manifest Tracing: Git Directory Generator

```
Git repo scan — directories matching "demo-13-applicationset/*":
  demo-13-applicationset/dev        ← discovered
  demo-13-applicationset/staging    ← discovered
  demo-13-applicationset/prod       ← discovered
  demo-13-applicationset/dev/manifests ← excluded by exclude rule

For discovered path "demo-13-applicationset/dev":
  {{.path.path}}     = "demo-13-applicationset/dev"
  {{.path.basename}} = "dev"
         │  template substitution
         ▼
Application CRD:
  metadata.name: "demo13-dev"                       ← {{.path.basename}}
  spec.source.path: "demo-13-applicationset/dev/manifests" ← {{.path.path}}/manifests
  spec.destination.namespace: "demo13-dev"          ← {{.path.basename}}
         │  ArgoCD syncs
         ▼
Manifests: dev/manifests/deployment.yaml
  → replicas=1, namespace=demo13-dev
  → Deployed to: in-cluster
```

**Push and apply:**
```bash
git add demo-13-applicationset/03-git-directory-generator.yaml
git commit -m "feat: add demo-13 git directory generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationset/03-git-directory-generator.yaml
```

**Watch Applications appear:**
```bash
watch -n 2 'argocd app list'
```

**Expected:**
```text
NAME                     SYNC STATUS   HEALTH STATUS
demo13-dev               Synced        Healthy
demo13-staging           Synced        Healthy
demo13-prod              Synced        Healthy
demo13-us-east-ohio      Synced        Healthy
demo13-middle-east-uae   Synced        Healthy
```

**Verify Pods and replica counts reflect environment manifests:**
```bash
kubectl get pods -n demo13-dev
#replica=1

kubectl get pods -n demo13-staging 
#replica=2

kubectl get pods -n demo13-prod
#replica=3

##replica=1 for below
kubectl get pods -n demo13-us-east-ohio
kubectl get pods -n demo13-middle-east-uae
```

**Prove directory = new Application — no YAML change needed:**
```bash
cd argo-cd-basics-to-prod/13-applicationset/src/gitops-apps-config

mkdir -p demo-13-applicationset/hotfix/manifests
cp demo-13-applicationset/dev/manifests/deployment.yaml \
   demo-13-applicationset/hotfix/manifests/deployment.yaml

git add demo-13-applicationset/hotfix/
git commit -m "feat: add hotfix environment directory"
git push origin main
# ArgoCD detects new directory → creates demo13-hotfix Application automatically in a different namespace=demo13-hotfix
```

**Watch Applications appear:**
```bash
watch -n 2 'argocd app list'
```

**Expected:**
```text
NAME                     SYNC STATUS   HEALTH STATUS
demo13-dev               Synced        Healthy
demo13-staging           Synced        Healthy
demo13-prod              Synced        Healthy
demo13-us-east-ohio      Synced        Healthy
demo13-middle-east-uae   Synced        Healthy
demo13-hotfix            Synced        Healthy
```

**Verify Pods:**
```bash
kubectl get pods -n demo13-hotfix
#Note: namespace changed, though using the same dev mainfest
```


**Clean up:**
```bash
kubectl delete appset demo13-git-directory -n argocd

#Confirm All Applications removed
argocd app list
```

---

## Step 6: Generator 4 — Git File Generator

### What this step demonstrates

We deploy to both clusters again, but target definitions live in a `clusters.yaml`
file in the Git repository — not inline in the ApplicationSet YAML. Adding a
new target requires only updating `clusters.yaml` and pushing. The ApplicationSet
YAML itself never changes.

### About the Git File Generator

The Git File generator reads structured YAML (or JSON) files from a Git repository.
Each entry in the file provides the parameter set for one Application. The file
is version-controlled independently of the ApplicationSet.

**How the file maps to variables:**
Every key in each YAML entry becomes a template variable directly:

```yaml
# clusters.yaml entry:
- name: us-east-ohio
  region: us-east-ohio
  clusterName: in-cluster
  namespace: demo13-us-east-ohio-gf

# Becomes template variables:
{{.name}}         → us-east-ohio
{{.region}}       → us-east-ohio
{{.clusterName}}  → in-cluster
{{.namespace}}    → demo13-us-east-ohio-gf
```

**When to use:** Complex parameter sets unwieldy in an ApplicationSet YAML;
when a different team manages cluster targets and should not need to edit the
ApplicationSet YAML; when targets have many fields (VPC IDs, account IDs, etc.).

**The `clusters.yaml` file was pushed in Step 1.** Its contents:
```yaml
- name: us-east-ohio
  region: us-east-ohio
  clusterName: in-cluster
  namespace: demo13-us-east-ohio-gf
- name: middle-east-uae
  region: middle-east-uae
  clusterName: middle-east-uae
  namespace: demo13-middle-east-uae-gf
```

```bash
cd argo-cd-basics-to-prod/13-applicationset/src/argocd-config
touch demo-13-applicationset/04-git-file-generator.yaml
```

**Create `demo-13-applicationset/04-git-file-generator.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo13-git-file
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
  generators:
    - git:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        revision: main
        files:
          - path: "demo-13-applicationset/clusters.yaml"
  template:
    metadata:
      name: "demo13-gf-{{.name}}"
      # → demo13-gf-us-east-ohio, demo13-gf-middle-east-uae
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "demo-13-applicationset/{{.region}}/manifests"
      destination:
        name: "{{.clusterName}}"
        namespace: "{{.namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Variable-to-Manifest Tracing: Git File Generator

```
clusters.yaml entry[0]:
  name: "us-east-ohio"
  region: "us-east-ohio"
  clusterName: "in-cluster"
  namespace: "demo13-us-east-ohio-gf"
         │  template substitution
         ▼
Application CRD:
  metadata.name: "demo13-gf-us-east-ohio"          ← {{.name}}
  spec.source.path: ".../us-east-ohio/manifests"    ← {{.region}}
  spec.destination.name: "in-cluster"               ← {{.clusterName}}
  spec.destination.namespace: "demo13-us-east-ohio-gf" ← {{.namespace}}
         │  ArgoCD syncs
         ▼
Manifests: us-east-ohio/manifests/ → deployed to in-cluster

clusters.yaml entry[1]:
  name: "middle-east-uae" → Application: demo13-gf-middle-east-uae
  clusterName: "middle-east-uae" → deployed to middle-east-uae cluster
```

**Push and apply:**
```bash
git add demo-13-applicationset/04-git-file-generator.yaml
git commit -m "feat: add demo-13 git file generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationset/04-git-file-generator.yaml
```

**Verify Applications generated from file:**
```bash
argocd app list
```

**Expected:**
```text
NAME                       SYNC STATUS   HEALTH STATUS
demo13-gf-us-east-ohio     Synced        Healthy
demo13-gf-middle-east-uae  Synced        Healthy
```

**Demonstrate Git-only addition — add a third region to `clusters.yaml`:**
```bash
cd argo-cd-basics-to-prod/13-applicationset/src/gitops-apps-config
#update demo-13-applicationset/clusters.yaml
```

Add to `demo-13-applicationset/clusters.yaml`:
```yaml
- name: eu-central
  region: us-east-ohio     # reuse manifests for demo
  clusterName: in-cluster  # deploy to same cluster locally
  namespace: demo13-eu-central-gf
```

```bash
git add demo-13-applicationset/clusters.yaml
git commit -m "feat: add eu-central entry to clusters.yaml"
git push origin main
```

**Expected — third Application created with zero ApplicationSet YAML change:**
```bash
argocd app list
# demo13-gf-eu-central  Synced  Healthy  ← appeared from file change only
```

**Verify Pods:**
```bash
#Check in us-east-ohio
kubectl config use-context us-east-ohio
kubectl get pods -n demo13-us-east-ohio-gf
kubectl get pods -n demo13-eu-central-gf

#Check in us-east-ohio
kubectl config use-context middle-east-uae
kubectl get pods -n demo13-middle-east-uae-gf
```

**Clean up:**
```bash
kubectl delete appset demo13-git-file -n argocd

#Verify All Applications Remvoed
argocd app list
```

---

## Step 7: Generator 5 — Matrix Generator

### What this step demonstrates

We deploy three environments (dev, staging, prod) to both clusters — producing
six Applications from one ApplicationSet YAML. The Matrix generator combines
the Git Directory generator (discovers environments) with the List generator
(defines clusters). Every environment × every cluster = one Application.

### About the Matrix Generator

The Matrix generator combines exactly two generators and produces one Application
for every valid combination of their outputs.

**Key rules:**
- Matrix wraps exactly two generators — no more, no less
- Both generators must be self-sufficient — Matrix itself provides no variables
- All variables from both generators are merged and available in the template
- If both generators expose a variable with the same name, the second generator's
  value wins

**When to use:** Every environment deployed to every cluster; any two-dimensional
deployment scenario.

```bash
cd argo-cd-basics-to-prod/13-applicationset/src/argocd-config
touch demo-13-applicationset/05-matrix-generator.yaml
```

**Create `demo-13-applicationset/05-matrix-generator.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo13-matrix
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
  generators:
    - matrix:
        generators:
          # Generator 1: discovers dev, staging, prod directories
          - git:
              repoURL: https://github.com/rselvantech/gitops-apps-config.git
              revision: main
              directories:
                - path: "demo-13-applicationset/dev"
                - path: "demo-13-applicationset/staging"
                - path: "demo-13-applicationset/prod"

          # Generator 2: defines the two target clusters
          - list:
              elements:
                - region: us-east-ohio
                  clusterName: in-cluster
                - region: middle-east-uae
                  clusterName: middle-east-uae
  template:
    metadata:
      name: "demo13-{{.path.basename}}-{{.region}}"
      # 6 combinations (3 directories × 2 clusters):
      # demo13-dev-us-east-ohio, demo13-dev-middle-east-uae
      # demo13-staging-us-east-ohio, demo13-staging-middle-east-uae
      # demo13-prod-us-east-ohio, demo13-prod-middle-east-uae
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "demo-13-applicationset/{{.path.basename}}/manifests"
        # {{.path.basename}} from Git Directory generator (dev/staging/prod)
      destination:
        name: "{{.clusterName}}"
        # {{.clusterName}} from List generator
        namespace: "demo13-{{.path.basename}}-{{.region}}"
        # combines both generator variables
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Variable-to-Manifest Tracing: Matrix Generator

```
Git Directory generator output:
  [0] path.basename="dev",     path.path="demo-13-applicationset/dev"
  [1] path.basename="staging", path.path="demo-13-applicationset/staging"
  [2] path.basename="prod",    path.path="demo-13-applicationset/prod"

List generator output:
  [0] region="us-east-ohio",    clusterName="in-cluster"
  [1] region="middle-east-uae", clusterName="middle-east-uae"

Matrix combines: 3 × 2 = 6 combinations

Combination [dev × us-east-ohio]:
  Merged: path.basename="dev", region="us-east-ohio", clusterName="in-cluster"
         │  template substitution
         ▼
  Application: demo13-dev-us-east-ohio
    source.path: demo-13-applicationset/dev/manifests   ← {{.path.basename}}
    destination.name: in-cluster                        ← {{.clusterName}}
    destination.namespace: demo13-dev-us-east-ohio      ← {{.path.basename}}-{{.region}}
         ▼
  Manifests: dev/manifests → replicas=1, deployed to in-cluster

Combination [prod × middle-east-uae]:
  Merged: path.basename="prod", region="middle-east-uae", clusterName="middle-east-uae"
         ▼
  Application: demo13-prod-middle-east-uae
    source.path: demo-13-applicationset/prod/manifests
    destination.name: middle-east-uae
    destination.namespace: demo13-prod-middle-east-uae
         ▼
  Manifests: prod/manifests → replicas=3, deployed to middle-east-uae cluster
```

> **All 6 combinations:**
> ```
> dev     × us-east-ohio    → demo13-dev-us-east-ohio      (in-cluster)
> dev     × middle-east-uae → demo13-dev-middle-east-uae   (middle-east-uae)
> staging × us-east-ohio    → demo13-staging-us-east-ohio  (in-cluster)
> staging × middle-east-uae → demo13-staging-middle-east-uae (middle-east-uae)
> prod    × us-east-ohio    → demo13-prod-us-east-ohio     (in-cluster)
> prod    × middle-east-uae → demo13-prod-middle-east-uae  (middle-east-uae)
> ```

**Push and apply:**
```bash
git add demo-13-applicationset/05-matrix-generator.yaml
git commit -m "feat: add demo-13 matrix generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationset/05-matrix-generator.yaml
```

**Watch all six Applications appear:**
```bash
watch -n 2 'argocd app list'
```

**Expected:**
```text
NAME                              CLUSTER           SYNC STATUS   HEALTH STATUS
demo13-dev-us-east-ohio           in-cluster        Synced        Healthy
demo13-dev-middle-east-uae        middle-east-uae   Synced        Healthy
demo13-staging-us-east-ohio       in-cluster        Synced        Healthy
demo13-staging-middle-east-uae    middle-east-uae   Synced        Healthy
demo13-prod-us-east-ohio          in-cluster        Synced        Healthy
demo13-prod-middle-east-uae       middle-east-uae   Synced        Healthy
```

**Verify Pods:**
```bash
#Check in us-east-ohio
kubectl config use-context us-east-ohio
kubectl get pods -n demo13-dev-us-east-ohio
kubectl get pods -n demo13-staging-us-east-ohio
kubectl get pods -n demo13-prod-us-east-ohio 

#Check in us-east-ohio
kubectl config use-context middle-east-uae
kubectl get pods -n demo13-dev-middle-east-uae
kubectl get pods -n demo13-staging-middle-east-uae
kubectl get pods -n demo13-prod-middle-east-uae
```

**Key observation — adding a new environment across both clusters:**
Add a `hotfix` directory to `gitops-apps-config`. The Git Directory sub-generator
detects it. Matrix multiplies: `hotfix × us-east-ohio` and
`hotfix × middle-east-uae` — two new Applications created automatically with
zero ApplicationSet YAML change.


**Clean up:**
```bash
kubectl config use-context us-east-ohio
kubectl delete appset demo13-matrix -n argocd

#Verify All Applications Remvoed
argocd app list
```


---

## Cleanup

```bash
# check if any ApplicationSets exist 
kubectl get appset  -n argocd


# Delete AppSet, If any exist
kubectl delete appset <Appset name> -n argocd


# Verify all demo-13 Applications gone
argocd app list
```


>**Why namespaces must be deleted manually:**
> ArgoCD creates namespaces as infrastructure via `CreateNamespace=true` — they
> are not part of the Git manifests and therefore not tracked as managed resources.
> `prune: true` only removes resources that exist in Git and have been deleted from
> it. Since the namespace was never in Git, it is never pruned. Delete namespaces
> manually after removing Applications.
>

**Delete namespaces on us-east-ohio (in-cluster)**
```bash
kubectl config use-context us-east-ohio
kubectl delete ns demo13-us-east-ohio demo13-middle-east-uae \
   demo13-dev demo13-staging demo13-prod \
   demo13-dev-us-east-ohio demo13-staging-us-east-ohio \
   demo13-prod-us-east-ohio demo13-eu-central-gf demo13-us-east-ohio-gf
```
**Delete namespaces on middle-east-uae**
```bash
kubectl config use-context middle-east-uae
kubectl delete ns demo13-middle-east-uae \
   demo13-dev-middle-east-uae demo13-staging-middle-east-uae \
   demo13-prod-middle-east-uae demo13-middle-east-uae-gf
``` 


```bash 
kubectl config use-context us-east-ohio

# Delete the second minikube profile
minikube delete -p middle-east-uae

# Rename the context back to 'minikube'
kubectl config rename-context us-east-ohio minikube
```

**Verify context restored:**
```bash
kubectl config get-contexts
```

**Expected:**
```text
CURRENT   NAME       CLUSTER    AUTHINFO
*         minikube   minikube   minikube
```

---

## Key Concepts Summary

**ApplicationSet = generators + template**
Generators define how many Applications to create and for whom. Templates define
what each Application looks like — identical to a standard Application CRD spec.

**Manifests differ across region and environment folders intentionally**
Each folder represents a distinct deployment target — different namespace,
replica count, or configuration. This is the standard GitOps pattern. Demo-14
(Kustomize) covers an alternative approach that reduces manifest repetition.

**`goTemplate: true` with `missingkey=error` is the production default**
Always pair them. Go templating enables nested variable access. `missingkey=error`
fails loudly on undefined variables instead of creating broken Applications.

**Cluster generator requires Part 1 labels — applied before the ApplicationSet**
The Cluster generator selects by label at sync time. Labels were set in
`README-cluster-setup.md`. Applying the ApplicationSet before labelling generates
zero Applications.

**Matrix multiplies — it does not create intent**
Matrix combines two generators. 3 environments × 2 clusters = 6 Applications.
Matrix contributes no variables of its own.

**Generator selection guide:**

| Situation | Generator |
|---|---|
| Small fixed set of 2-5 targets | List |
| Dynamic cluster fleet, label-driven | Cluster |
| Directory per microservice or environment | Git Directory |
| Complex parameters, separate ownership | Git File |
| Every environment on every cluster | Matrix |

---

## Commands Reference

```bash
# Context management
kubectl config get-contexts
kubectl config use-context us-east-ohio
kubectl config use-context middle-east-uae
kubectl config rename-context us-east-ohio minikube

# Cluster list
argocd cluster list
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster

# ApplicationSet operations
kubectl apply -f <appset.yaml>
kubectl get appset -n argocd
kubectl describe appset <n> -n argocd
kubectl delete appset <n> -n argocd

# Generated Applications
argocd app list
kubectl get apps -n argocd
watch -n 2 'argocd app list'

# Check Docker network wiring (re-run after Docker/WSL2 restart)
docker network connect middle-east-uae minikube
docker exec minikube nc -zv -w 3 192.168.67.2 8443
```

---

## Lessons Learned

**1. ApplicationSets manage Applications — not workloads**
Troubleshoot in order: ApplicationSet status → generated Application sync
status → Kubernetes resources. Each tier has its own starting point.

**2. Always use `goTemplate: true` and `missingkey=error`**
Default string interpolation is fragile and limited. Go templating with
`missingkey=error` fails loudly on undefined variables.

**3. Cluster labels from Part 1 are what the Cluster generator reads**
The labels applied to clusters in `README-cluster-setup.md` are stored in
cluster Secrets in the `argocd` namespace. The Cluster generator reads these
Secrets at sync time. Labels must be set before the ApplicationSet is applied.

**4. Deleting an ApplicationSet cascades to Applications and resources**
With `prune: true` in the template, deleting the ApplicationSet removes all
generated Applications and their Kubernetes resources.

**5. Matrix does not create intent — it multiplies existing intent**
Both inner generators must be self-sufficient. Matrix only multiplies their
outputs. If a variable is missing from either generator, the template fails.

**6. Git Directory generator: exclude inner directories explicitly**
If your structure is `env/manifests/deployment.yaml`, the generator discovers
both `env` and `env/manifests`. Exclude `*/manifests` explicitly.

**7. Context switching is the most common error source in multi-cluster work**
Always verify active context before any `kubectl` command. ArgoCD commands
always target `us-east-ohio`. Pod verification for `middle-east-uae` requires
explicitly switching context.

**8. Docker network wiring must be re-run after Docker/WSL2 restart**
The ArgoCD credential Secret persists. Only the network route needs
re-establishing. See `README-cluster-setup.md` Step 3 for the commands.

**9. Never hardcode `metadata.namespace` in ArgoCD-managed manifests**
ArgoCD does not rewrite resource namespaces — `metadata.namespace` in a manifest
always overrides `destination.namespace` in the Application CRD. For ApplicationSets
this is critical: generator variables set `destination.namespace` dynamically per
Application, but a hardcoded manifest namespace ignores that entirely and causes
sync failures when the hardcoded namespace does not exist on the target cluster.
Remove `namespace:` from all manifest metadata and let `destination.namespace`
own it. `CreateNamespace=true` in `syncOptions` handles namespace creation.

**10. ArgoCD never deletes namespaces — even with prune and selfHeal enabled**
Namespaces are created as infrastructure via `CreateNamespace=true` and are not
tracked as managed resources because they are not in the Git manifests. `prune: true`
only removes resources that were in Git and have since been deleted — namespaces were
never in Git so they are never pruned. This is also intentional safety behaviour:
automatic namespace deletion would cascade-delete everything inside it, including
resources ArgoCD did not create. Always delete namespaces manually after removing
Applications or ApplicationSets.

---

## What's Next

**Demo-14: Kustomize with ArgoCD**
Use Kustomize base and overlays with ArgoCD to manage environment-specific
configuration without duplicating manifests. The natural complement to
ApplicationSets — combine Kustomize overlays with a Git Directory or Matrix
generator so each generated Application automatically applies its
environment-specific overlay. This directly addresses the "why do manifests
look similar across folders" question from this demo.