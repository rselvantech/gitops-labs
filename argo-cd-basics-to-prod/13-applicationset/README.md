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

**What you'll learn:**
- Why ApplicationSets exist — the scaling limits of App-of-Apps
- The two building blocks of every ApplicationSet: generators and templates
- How generators define HOW MANY and FOR WHOM Applications are created
- How templates define HOW each Application looks — same spec as Application CRD
- Go templating syntax — what it is and how it works in ApplicationSets
- The `goTemplate` and `goTemplateOptions` fields — what they do and why
- All five generators in depth: List, Cluster, Git Directory, Git File, Matrix
- How to choose the right generator for a given requirement
- Multi-cluster setup using the default minikube profile and a second profile

**What you'll do:**
- Rename the default minikube profile to `us-east-ohio` to simulate a real
  cluster name, register a second minikube profile `middle-east-uae`
- Register the second cluster with ArgoCD and apply labels
- Walk through all five generators end-to-end with working demos
- Combine generators with the Matrix generator to deploy dev/staging/prod
  across two regions — six Applications from one YAML

---

## Prerequisites

- ✅ Completed Demo-12 — App-of-Apps pattern understood
- ✅ ArgoCD running on minikube default profile (installed in Demo-02)
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with `Contents: Read` access to `app-config` and `argocd-config`
  (PAT setup covered in Demo-04 — reuse the same token)

**Verify Prerequisites:**

### 1. ArgoCD pods running on default minikube profile
```bash
kubectl get pods -n argocd
```

**Expected:** All pods `Running` and `1/1` Ready.

**Recovery:** If ArgoCD is not running, start minikube and port-forward:
```bash
minikube start
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2. ArgoCD CLI is installed and logged in
```bash
argocd version --client
argocd login localhost:8080 --username admin --insecure
```

**Expected:**
```text
argocd: v3.x.x
'admin:login' logged in successfully
```

### 3. Both repos registered with ArgoCD
```bash
argocd repo list
```

**Expected:** `app-config` and `argocd-config` both showing `Successful`.

**Recovery:** Re-register if missing:
```bash
argocd repo add https://github.com/rselvantech/app-config.git \
  --username rselvantech --password <GITHUB_PAT>
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

### Why Two Clusters — The Real-World Reasons

In production, running the same application across multiple clusters is driven
by four common requirements:

**1. Geographic latency** — Serve users from the nearest cluster. A customer
in Dubai should not have their requests routed to Ohio data centres. Each
cluster handles traffic for its region.

**2. Data residency and compliance** — Many governments mandate that user data
must not leave the country. Separate clusters in separate regions satisfy these
regulatory requirements without special application logic.

**3. Failure domain isolation** — If the Ohio cluster has an incident (node
failures, networking issues, a bad deployment), the UAE cluster is completely
unaffected. Users in the Middle East continue to be served.

**4. Environment separation** — Dev and staging workloads run on one cluster,
production on another. This prevents a broken staging deployment from affecting
production.

In this demo we simulate **geographic separation** — `us-east-ohio` serves US
users and `middle-east-uae` serves Middle East users — the most direct
illustration of why multi-cluster deployments exist.

---

### How the Two Clusters Are Connected — The Control Plane Model

ArgoCD is not installed on both clusters. It runs on **one cluster only** —
the **control plane cluster**. That one ArgoCD instance manages both clusters
from a central location.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     ArgoCD Control Plane Model                           │
│                                                                          │
│   GitHub (app-config)                                                    │
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

```
One-time setup (you run once):
  argocd cluster add middle-east-uae
    → creates argocd-manager SA on middle-east-uae
    → stores credentials as Secret on us-east-ohio

Every subsequent sync (ArgoCD runs automatically):
  App Controller reads Secret → authenticates to middle-east-uae
    → applies manifests → reconciles drift
```

**What you need access to** in this demo:

| Who | Accesses | How |
|---|---|---|
| You (setup) | `us-east-ohio` | kubectl, kubeconfig |
| You (setup) | `middle-east-uae` | kubectl, kubeconfig (only for `argocd cluster add`) |
| You (day-to-day) | ArgoCD UI/CLI on `us-east-ohio` | port-forward 8080 |
| ArgoCD | `us-east-ohio` (local) | in-cluster ServiceAccount |
| ArgoCD | `middle-east-uae` (remote) | argocd-manager SA token stored as Secret |

After setup, the only cluster you interact with directly is `us-east-ohio` via
the ArgoCD UI and CLI. ArgoCD manages `middle-east-uae` on your behalf.

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

Similarly: ApplicationSet creates Applications from an application template.
You do not write 60 Application YAMLs — you write one ApplicationSet and
define the generation rules.

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

ApplicationSets support Go's `text/template` package for variable injection —
the same engine used by Helm and Kubernetes itself. Enabling it via
`goTemplate: true` unlocks a concise, expressive syntax for injecting generator
variables into the application template.

**Core syntax:**

```yaml
# Variable injection — reads a generator variable
{{.variableName}}

# Nested field access — reads a nested variable
{{.path.basename}}      # accesses the "basename" field inside "path"
{{.metadata.labels.region}}  # accesses nested label value

# String operations
{{.region | upper}}     # converts value to uppercase
{{.name | lower}}       # converts value to lowercase
{{printf "%s-%s" .env .region}}   # formats two variables together

# Default value when variable might be empty
{{default "dev" .environment}}
```

**Why Go templating over the default:**

The default ApplicationSet syntax uses simple interpolation: `{{region}}`.
Go templating switches this to dot-notation: `{{.region}}`. The dot notation
is richer — it supports nested field access (required for Cluster generator
variables like `{{.metadata.labels.region}}`), string functions, and
conditionals. It is also consistent with Helm which most teams already use.

```yaml
# Default interpolation (fragile, limited)
name: "app-{{region}}"

# Go templating (powerful, consistent with Helm)
name: "app-{{.region}}"
name: "app-{{.metadata.labels.region}}"   # nested — not possible without goTemplate
```

**The standard production block:**

```yaml
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"
```

`missingkey=error` is a safety net — if a template references a variable that
the generator does not expose, the ApplicationSet fails loudly with a clear
error instead of silently creating Applications with empty `name` or `path`
fields. Always include both fields together.

---

### The Five Generators

| Generator | Selects by | Best for |
|---|---|---|
| **List** | Inline items in the YAML | Small, fixed set of targets |
| **Cluster** | Labels on registered clusters | Dynamic cluster fleet |
| **Git Directory** | Directories in a Git repo | Microservice per directory pattern |
| **Git File** | Structured YAML files in Git | Complex parameters, version-controlled intent |
| **Matrix** | Combines two generators | Multi-environment × multi-cluster |

Each generator answers the same question — "for whom should Applications be
created?" — using a different source of truth:

**List generator** — you declare the targets directly in the ApplicationSet
YAML as a list of items. Each item is a set of key-value pairs that become
template variables. Simplest generator. Best for small, stable, fixed sets
of 2-5 targets where the configuration rarely changes.

**Cluster generator** — targets are determined by labels on ArgoCD-registered
clusters. ArgoCD reads the labels from cluster registration secrets and creates
one Application per matching cluster. New cluster added with the right label →
Application auto-generated. Best for dynamic cluster fleets.

**Git Directory generator** — targets are discovered by scanning directory
paths in a Git repository. Each matching directory becomes one Application.
Adding a new directory to Git creates a new Application without touching the
ApplicationSet YAML. Best for microservice-per-directory or
environment-per-directory patterns.

**Git File generator** — targets are defined in structured YAML or JSON files
stored in Git. Each entry in the file becomes one Application. The file is
version-controlled and auditable separately from the ApplicationSet YAML.
Best when parameters are complex or when different teams own the generator
configuration vs the ApplicationSet template.

**Matrix generator** — combines the output of two other generators and produces
one Application for every valid combination. Does not create new intent — it
multiplies existing intent from two generators. Best for deploying every
environment to every cluster (the most common enterprise pattern).

---

### How Cluster Labels Are Stored

When you register a cluster with ArgoCD and add labels, ArgoCD stores those
labels inside a Kubernetes Secret in the `argocd` namespace. The Cluster
generator reads these secrets to find clusters matching its label selector.

```bash
# See the secrets backing registered clusters
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster

# Inspect a cluster secret — labels are stored in the secret data
kubectl get secret <cluster-secret-name> -n argocd -o yaml
```

This is why adding a label to an already-registered cluster requires the
`--upsert` flag on `argocd cluster add` — it overwrites the existing secret
with the new labels. Without `--upsert` the command fails saying the cluster
already exists.

---

## Folder Structure

```
13-applicationsets/src/
├── app-config/                          ← rselvantech/app-config (private)
│   └── demo-13-applicationsets/
│       ├── us-east-ohio/
│       │   └── manifests/
│       │       ├── deployment.yaml
│       │       └── service.yaml
│       ├── middle-east-uae/
│       │   └── manifests/
│       │       ├── deployment.yaml
│       │       └── service.yaml
│       ├── dev/
│       │   └── manifests/
│       │       └── deployment.yaml
│       ├── staging/
│       │   └── manifests/
│       │       └── deployment.yaml
│       ├── prod/
│       │   └── manifests/
│       │       └── deployment.yaml
│       └── clusters.yaml               ← used by Git File generator
└── argocd-config/                       ← rselvantech/argocd-config
    └── demo-13-applicationsets/
        ├── 01-list-generator.yaml
        ├── 02-cluster-generator.yaml
        ├── 03-git-directory-generator.yaml
        ├── 04-git-file-generator.yaml
        └── 05-matrix-generator.yaml
```

**What exists in GitHub after all pushes:**

```
rselvantech/app-config (GitHub)
├── demo-11-sync-waves/          ← Demo-11 (untouched)
├── demo-12-app-of-apps/         ← Demo-12 (untouched)
└── demo-13-applicationsets/     ← Demo-13 adds this
    ├── us-east-ohio/manifests/
    ├── middle-east-uae/manifests/
    ├── dev/manifests/
    ├── staging/manifests/
    ├── prod/manifests/
    └── clusters.yaml

rselvantech/argocd-config (GitHub)
├── demo-11-sync-waves/          ← Demo-11 (untouched)
├── demo-12-app-of-apps/         ← Demo-12 (untouched)
└── demo-13-applicationsets/     ← Demo-13 adds this
    ├── 01-list-generator.yaml
    ├── 02-cluster-generator.yaml
    ├── 03-git-directory-generator.yaml
    ├── 04-git-file-generator.yaml
    └── 05-matrix-generator.yaml
```

---

## Step 1: Multi-Cluster Setup — Rename Default Profile and Add Second Cluster

This demo simulates a two-cluster environment. Rather than creating fresh
clusters from scratch (which would require reinstalling ArgoCD), we reuse the
existing default minikube profile where ArgoCD is already running — renaming it
to reflect a real-world region name — and add a second lightweight profile
representing a second region.

### Step 1a: Rename the default profile to `us-east-ohio`

minikube does not support renaming profiles directly. The workaround is to
update the kubeconfig context name, which is what ArgoCD and kubectl use:

```bash
# Rename the context in kubeconfig from 'minikube' to 'us-east-ohio'
kubectl config rename-context minikube us-east-ohio
```

**Verify:**
```bash
kubectl config get-contexts
```

**Expected:**
```text
CURRENT   NAME           CLUSTER    AUTHINFO   NAMESPACE
*         us-east-ohio   minikube   minikube
```

All ArgoCD pods and previous demo resources are still running — only the
context name changed. The cluster itself is unchanged.

### Step 1b: Start the second cluster — `middle-east-uae`

```bash
minikube start -p middle-east-uae --cpus 2 --memory 2048
```

This starts a second, independent minikube cluster in a new profile. ArgoCD
is NOT installed here — it only runs in the `us-east-ohio` (default) cluster.

**Verify:**
```bash
kubectl config get-contexts
```

**Expected:**
```text
CURRENT   NAME              CLUSTER           AUTHINFO          NAMESPACE
          us-east-ohio      minikube          minikube
*         middle-east-uae   middle-east-uae   middle-east-uae
```

> The asterisk is on `middle-east-uae` because it was created last. All
> ArgoCD commands must target `us-east-ohio`. Switch context now:
> ```bash
> kubectl config use-context us-east-ohio
> ```

**Verify ArgoCD is still healthy on `us-east-ohio`:**
```bash
kubectl config use-context us-east-ohio
kubectl get pods -n argocd
```

**Expected:** All ArgoCD pods `Running` and `1/1` Ready.

### Step 1c: Log in to ArgoCD CLI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 --username admin --insecure
```

**Expected:**
```text
'admin:login' logged in successfully
```

### Step 1d: Register `middle-east-uae` cluster with ArgoCD

ArgoCD is running in `us-east-ohio` and must be told about `middle-east-uae`
before it can deploy resources there.

```bash
argocd cluster add middle-east-uae \
  --label app=demo13 \
  --label region=middle-east-uae \
  --name middle-east-uae
```

> **What this command does:**
> 1. Reads your local kubeconfig to find the `middle-east-uae` cluster
>    credentials
> 2. Connects to `middle-east-uae` and creates an `argocd-manager`
>    ServiceAccount with cluster-admin permissions
> 3. Stores the cluster URL, CA certificate, and ServiceAccount token as a
>    Secret in the `argocd` namespace on `us-east-ohio`
> 4. Tags the secret with the labels you provide — used by the Cluster
>    generator in Step 5
>
> **Why labels at registration time:** The Cluster generator (Step 5) selects
> clusters by label. Providing labels now avoids a separate `--upsert` step later.

**When prompted:** Type `y` to confirm ServiceAccount creation.

**Verify both clusters visible to ArgoCD:**
```bash
argocd cluster list
```

**Expected:**
```text
SERVER                          NAME              STATUS
https://kubernetes.default.svc  in-cluster        Successful
https://192.168.x.x:8443        middle-east-uae   Successful
```

### Step 1e: Add labels to the `us-east-ohio` (in-cluster) cluster

The `in-cluster` cluster was auto-registered when ArgoCD started. It has no
labels yet. Add them via the ArgoCD UI so the Cluster generator can select it:

```
http://localhost:8080
→ Settings → Clusters → in-cluster → Edit
→ Add labels:
    app=demo13
    region=us-east-ohio
→ Save
```

**Verify via CLI:**
```bash
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster -o yaml \
  | grep -A5 "labels:"
```

**Expected:** Both cluster secrets show `app: demo13` and their respective
`region` labels.

---

## Step 2: Add All Manifests to `app-config`

All five generators in this demo read application manifests from `app-config`.
All manifests are pushed now so each generator step only requires applying
the ApplicationSet YAML.

**Initialise local repo:**
```bash
cd gitops-labs/argo-cd-basics-to-prod/13-applicationsets/src
mkdir app-config && cd app-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/app-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

**`demo-13-applicationsets/us-east-ohio/manifests/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
  namespace: demo13-us-east-ohio
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
  namespace: demo13-us-east-ohio
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Region: US East Ohio | Cluster: us-east-ohio (in-cluster)</p>
    <p>Generator: List / Cluster</p>
    </body></html>
```

**`demo-13-applicationsets/us-east-ohio/manifests/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo13-svc
  namespace: demo13-us-east-ohio
spec:
  selector:
    app: demo13-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-13-applicationsets/middle-east-uae/manifests/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
  namespace: demo13-middle-east-uae
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
  namespace: demo13-middle-east-uae
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Region: Middle East UAE | Cluster: middle-east-uae</p>
    <p>Generator: List / Cluster</p>
    </body></html>
```

**`demo-13-applicationsets/middle-east-uae/manifests/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo13-svc
  namespace: demo13-middle-east-uae
spec:
  selector:
    app: demo13-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-13-applicationsets/dev/manifests/deployment.yaml`** (1 replica):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
  namespace: demo13-dev
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
  namespace: demo13-dev
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Environment: dev | Replicas: 1</p>
    <p>Generator: Git Directory / Matrix</p>
    </body></html>
```

**`demo-13-applicationsets/staging/manifests/deployment.yaml`** (2 replicas):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
  namespace: demo13-staging
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
  namespace: demo13-staging
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Environment: staging | Replicas: 2</p>
    <p>Generator: Git Directory / Matrix</p>
    </body></html>
```

**`demo-13-applicationsets/prod/manifests/deployment.yaml`** (3 replicas):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo13-app
  namespace: demo13-prod
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
  namespace: demo13-prod
data:
  index.html: |
    <html><body>
    <h1>Demo-13: ApplicationSets</h1>
    <p>Environment: prod | Replicas: 3</p>
    <p>Generator: Git Directory / Matrix</p>
    </body></html>
```

**`demo-13-applicationsets/clusters.yaml`** (for Git File generator — Step 6):
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
git add demo-13-applicationsets/
git commit -m "feat: add demo-13 applicationsets manifests for all five generators"
git push origin main
```

---

## Step 3: Set Up `argocd-config` Local Repo

All ApplicationSet YAMLs are stored in `argocd-config` — the same repo used
since Demo-12 which ArgoCD already watches. Each generator step adds one file
here.

```bash
cd gitops-labs/argo-cd-basics-to-prod/13-applicationsets/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

---

## Step 4: Generator 1 — List Generator

### What this step demonstrates

We deploy the same nginx application to two clusters — `us-east-ohio` and
`middle-east-uae` — using one ApplicationSet YAML with a List generator.
The list of targets is defined directly inline. This is the simplest
generator and the best starting point for understanding how generators and
templates interact.

### About the List Generator

The List generator takes a list of `elements` defined directly in the
ApplicationSet YAML. Each element is a set of key-value pairs you choose
freely. ArgoCD iterates over the list and runs the template once per element,
injecting the key-value pairs as template variables.

**Key fields:**
```yaml
generators:
  - list:
      elements:
        - key1: value1    # any keys you choose — referenced in template as {{.key1}}
          key2: value2
        - key1: value3
          key2: value4
```

**How it works:** The generator exposes only the keys you define in `elements`.
No automatic variables — everything must be explicitly declared. This makes it
transparent and easy to reason about, but it means adding a new target requires
editing the ApplicationSet YAML.

**When to use:**
- 2 to 5 deployment targets with a stable, known list
- Getting started with ApplicationSets — lowest cognitive overhead
- When targets are permanent and rarely change

**Create `demo-13-applicationsets/01-list-generator.yaml` in `argocd-config`:**

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
        repoURL: https://github.com/rselvantech/app-config.git
        targetRevision: HEAD
        path: "demo-13-applicationsets/{{.region}}/manifests"
        # → demo-13-applicationsets/us-east-ohio/manifests
        # → demo-13-applicationsets/middle-east-uae/manifests
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

**Push and apply:**
```bash
git add demo-13-applicationsets/01-list-generator.yaml
git commit -m "feat: add demo-13 list generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationsets/01-list-generator.yaml
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
NAME                     SYNC STATUS   HEALTH STATUS
demo13-us-east-ohio      Synced        Healthy
demo13-middle-east-uae   Synced        Healthy
```

**Verify pods on `us-east-ohio` (in-cluster):**
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
to two different clusters. To add a third region, add one more item to
`elements` — no new Application YAML needed.

**Clean up before next step:**
```bash
kubectl config use-context us-east-ohio
kubectl delete appset demo13-list-generator -n argocd
```

---

## Step 5: Generator 2 — Cluster Generator

### What this step demonstrates

We achieve the same two-cluster deployment as Step 4, but this time the
targets are not declared in the ApplicationSet YAML — they are discovered
automatically from labels on the registered clusters. Adding a new cluster
with the right label will auto-generate an Application with zero YAML changes.

### About the Cluster Generator

The Cluster generator scans ArgoCD's registered cluster secrets in the `argocd`
namespace and selects those matching a label selector. For each matching cluster,
it exposes the cluster's metadata as template variables.

**Key fields:**
```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          app: demo13       # select all clusters with this label
```

**Template variables exposed by the Cluster generator:**

| Variable | Value | Example |
|---|---|---|
| `{{.name}}` | Cluster name as registered | `in-cluster`, `middle-east-uae` |
| `{{.server}}` | Cluster API server URL | `https://kubernetes.default.svc` |
| `{{.metadata.labels.<key>}}` | Any label on the cluster | `{{.metadata.labels.region}}` |
| `{{.metadata.annotations.<key>}}` | Any annotation on cluster | — |

> Unlike the List generator where you define all variables explicitly, the
> Cluster generator exposes variables derived from the cluster Secret's
> metadata. This is why `goTemplate: true` is essential here — dot-notation
> is required to access nested fields like `{{.metadata.labels.region}}`.

**When to use:**
- Dynamic cluster fleets where clusters are added and removed regularly
- When the set of deployment targets changes over time
- When teams manage clusters independently and just need to apply a label
  to onboard into the ApplicationSet

**Create `demo-13-applicationsets/02-cluster-generator.yaml`:**

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
            app: demo13             # selects both clusters labelled in Step 1
  template:
    metadata:
      name: "demo13-{{.metadata.labels.region}}"
      # → demo13-us-east-ohio, demo13-middle-east-uae
      # reads the 'region' label set during cluster registration
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/app-config.git
        targetRevision: HEAD
        path: "demo-13-applicationsets/{{.metadata.labels.region}}/manifests"
        # → us-east-ohio/manifests, middle-east-uae/manifests
      destination:
        name: "{{.name}}"           # cluster name as registered in ArgoCD
        namespace: "demo13-{{.metadata.labels.region}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**Push and apply:**
```bash
git add demo-13-applicationsets/02-cluster-generator.yaml
git commit -m "feat: add demo-13 cluster generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationsets/02-cluster-generator.yaml
```

**Verify two Applications generated — driven by cluster labels:**
```bash
argocd app list
```

**Expected:**
```text
NAME                     SYNC STATUS   HEALTH STATUS
demo13-us-east-ohio      Synced        Healthy
demo13-middle-east-uae   Synced        Healthy
```

**Inspect the cluster secrets that power this generator:**
```bash
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
```

**Expected:**
```text
NAME                       TYPE     DATA   AGE
cluster-middle-east-uae-x  Opaque   3      xxm
```

**Key observation — label-driven fleet:**
Remove the `app=demo13` label from `middle-east-uae` in ArgoCD UI (Settings
→ Clusters → Edit) and the `demo13-middle-east-uae` Application is automatically
deleted. Add the label back and it reappears — no ApplicationSet YAML touched.

**Clean up:**
```bash
kubectl delete appset demo13-cluster-generator -n argocd
```

---

## Step 6: Generator 3 — Git Directory Generator

### What this step demonstrates

We deploy three environment versions of the application (dev, staging, prod)
to the `us-east-ohio` cluster by discovering directories in the `app-config`
repository. Adding a new environment directory to Git automatically creates a
new Application — no ApplicationSet YAML change required.

### About the Git Directory Generator

The Git Directory generator scans specified path patterns in a Git repository
and creates one Application for each matching directory. The directory structure
in Git becomes the source of truth for what Applications exist.

**Key fields:**
```yaml
generators:
  - git:
      repoURL: https://github.com/org/app-config.git
      revision: HEAD
      directories:
        - path: "some/path/*"     # pattern — discovers all matching directories
          exclude: false          # include these (default)
        - path: "some/path/*/manifests"
          exclude: true           # exclude inner directories from selection
```

**Template variables exposed by the Git Directory generator:**

| Variable | Value | Example |
|---|---|---|
| `{{.path.path}}` | Full path of discovered directory | `demo-13-applicationsets/dev` |
| `{{.path.basename}}` | Directory name only | `dev` |
| `{{.path.segments}}` | Path split into array | `["demo-13-applicationsets", "dev"]` |
| `{{.path[N]}}` | Nth segment of the path | `demo-13-applicationsets` (index 0) |

> **Why the `exclude` pattern is needed:**
> If your structure is `dev/manifests/deployment.yaml`, the generator discovers
> both `dev` and `dev/manifests` as separate paths — creating two Applications
> per environment. Explicitly excluding `*/manifests` prevents this.

**When to use:**
- Microservice-per-directory pattern — one team, one directory, one Application
- Environment-per-directory pattern — dev, staging, prod as top-level directories
- When the list of Applications should grow automatically as directories are added

**Create `demo-13-applicationsets/03-git-directory-generator.yaml`:**

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
        repoURL: https://github.com/rselvantech/app-config.git
        revision: HEAD
        directories:
          - path: "demo-13-applicationsets/*/manifests"
            exclude: true         # do NOT generate Applications for manifests/ directories
          - path: "demo-13-applicationsets/*"
            exclude: false        # generate Applications for dev, staging, prod, us-east-ohio, middle-east-uae
  template:
    metadata:
      name: "demo13-{{.path.basename}}"
      # → demo13-dev, demo13-staging, demo13-prod
      # (also picks up us-east-ohio, middle-east-uae — acceptable for demo purposes)
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/app-config.git
        targetRevision: HEAD
        path: "{{.path.path}}/manifests"
        # → demo-13-applicationsets/dev/manifests
        # → demo-13-applicationsets/staging/manifests
        # → demo-13-applicationsets/prod/manifests
      destination:
        name: in-cluster          # deploy all environments to the us-east-ohio cluster
        namespace: "demo13-{{.path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**Push and apply:**
```bash
git add demo-13-applicationsets/03-git-directory-generator.yaml
git commit -m "feat: add demo-13 git directory generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationsets/03-git-directory-generator.yaml
```

**Watch Applications appear — one per directory:**
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

**Verify replica counts reflect environment manifests:**
```bash
kubectl get deployment demo13-app -n demo13-dev \
  -o jsonpath='{.spec.replicas}'
# → 1

kubectl get deployment demo13-app -n demo13-prod \
  -o jsonpath='{.spec.replicas}'
# → 3
```

**Prove directory = new Application — no YAML change needed:**
```bash
# In app-config local repo — add a hotfix environment
mkdir -p demo-13-applicationsets/hotfix/manifests
cp demo-13-applicationsets/dev/manifests/deployment.yaml \
   demo-13-applicationsets/hotfix/manifests/deployment.yaml
# Edit namespace in the copied file: change demo13-dev → demo13-hotfix

git add demo-13-applicationsets/hotfix/
git commit -m "feat: add hotfix environment directory"
git push origin main
# ArgoCD detects new directory → creates demo13-hotfix Application automatically
# No ApplicationSet YAML change needed
```

**Clean up:**
```bash
kubectl delete appset demo13-git-directory -n argocd
```

---

## Step 7: Generator 4 — Git File Generator

### What this step demonstrates

We deploy to both clusters again, but this time the target definitions live in
a `clusters.yaml` file inside the Git repository — not inline in the
ApplicationSet YAML. Adding a new target requires only updating `clusters.yaml`
and pushing. The ApplicationSet YAML itself never changes.

### About the Git File Generator

The Git File generator reads structured YAML (or JSON) files from a Git
repository. Each entry in the file provides the parameter set for one
Application. The file is version-controlled independently of the ApplicationSet
— different teams can own them separately.

**Key fields:**
```yaml
generators:
  - git:
      repoURL: https://github.com/org/app-config.git
      revision: HEAD
      files:
        - path: "path/to/clusters.yaml"   # one or more files to read
```

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

**Contrast with List generator:**
The List generator embeds items directly in the ApplicationSet YAML — useful
but couples generator configuration to the ApplicationSet definition. The Git
File generator moves that configuration to a separate file — better for teams
where the person authoring ApplicationSets is different from the person
managing cluster targets.

**When to use:**
- Complex parameter sets that are unwieldy inside an ApplicationSet YAML
- When generator configuration needs to be version-controlled separately
- When a different team (e.g. platform team) manages cluster targets and should
  not need to edit the ApplicationSet YAML itself
- When targets have many fields (VPC IDs, region codes, account IDs, etc.)

**The `clusters.yaml` file was already pushed in Step 2.** Its contents:
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

**Create `demo-13-applicationsets/04-git-file-generator.yaml`:**

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
        repoURL: https://github.com/rselvantech/app-config.git
        revision: HEAD
        files:
          - path: "demo-13-applicationsets/clusters.yaml"
          # ArgoCD reads each entry in this file and runs the template once per entry
  template:
    metadata:
      name: "demo13-gf-{{.name}}"
      # → demo13-gf-us-east-ohio, demo13-gf-middle-east-uae
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/app-config.git
        targetRevision: HEAD
        path: "demo-13-applicationsets/{{.region}}/manifests"
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

**Push and apply:**
```bash
git add demo-13-applicationsets/04-git-file-generator.yaml
git commit -m "feat: add demo-13 git file generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationsets/04-git-file-generator.yaml
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

Edit `demo-13-applicationsets/clusters.yaml` to add a third entry:
```yaml
- name: us-east-ohio
  region: us-east-ohio
  clusterName: in-cluster
  namespace: demo13-us-east-ohio-gf
- name: middle-east-uae
  region: middle-east-uae
  clusterName: middle-east-uae
  namespace: demo13-middle-east-uae-gf
- name: eu-central
  region: us-east-ohio          # reuse us-east-ohio manifests for demo
  clusterName: in-cluster       # deploy to same cluster for local demo
  namespace: demo13-eu-central-gf
```

```bash
git add demo-13-applicationsets/clusters.yaml
git commit -m "feat: add eu-central entry to clusters.yaml"
git push origin main
```

**Expected — third Application created with zero ApplicationSet YAML change:**
```bash
argocd app list
# NAME                        SYNC STATUS   HEALTH
# demo13-gf-us-east-ohio      Synced        Healthy
# demo13-gf-middle-east-uae   Synced        Healthy
# demo13-gf-eu-central        Synced        Healthy  ← appeared from file change only
```

**Clean up:**
```bash
kubectl delete appset demo13-git-file -n argocd
```

---

## Step 8: Generator 5 — Matrix Generator

### What this step demonstrates

We deploy three environments (dev, staging, prod) to both clusters — producing
six Applications from one ApplicationSet YAML. The Matrix generator combines
the Git Directory generator (which discovers environments) with the List
generator (which defines clusters). Every environment × every cluster = one
Application per combination.

### About the Matrix Generator

The Matrix generator combines exactly two generators and produces one
Application for every valid combination of their outputs. It is not a
standalone generator — it wraps two other generators and multiplies their results.

**Key fields:**
```yaml
generators:
  - matrix:
      generators:
        - <generator-type-1>:   # first generator — produces M items
            ...
        - <generator-type-2>:   # second generator — produces N items
            ...
        # Result: M × N Applications
```

**Template variables:**
All variables from both inner generators are merged and available in the
template simultaneously. If generator 1 exposes `{{.path.basename}}` and
generator 2 exposes `{{.region}}`, both are available in the template without
any special syntax.

**Important rules:**
- Matrix wraps exactly two generators — no more, no less
- Both generators must be self-sufficient — Matrix itself provides no variables
- If the two generators expose variables with the same name, the second
  generator's value wins (last-write wins)
- Matrix can wrap any two generator types — including Cluster, Git Directory,
  Git File, or another List

**When to use:**
- Every environment (dev/staging/prod) deployed to every cluster
- Every microservice deployed to every environment
- Any scenario where two independent dimensions both need to be fully covered

**Create `demo-13-applicationsets/05-matrix-generator.yaml`:**

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
              repoURL: https://github.com/rselvantech/app-config.git
              revision: HEAD
              directories:
                - path: "demo-13-applicationsets/dev"
                - path: "demo-13-applicationsets/staging"
                - path: "demo-13-applicationsets/prod"

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
      # Runs 6 times (3 directories × 2 clusters):
      # demo13-dev-us-east-ohio, demo13-dev-middle-east-uae
      # demo13-staging-us-east-ohio, demo13-staging-middle-east-uae
      # demo13-prod-us-east-ohio, demo13-prod-middle-east-uae
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/app-config.git
        targetRevision: HEAD
        path: "demo-13-applicationsets/{{.path.basename}}/manifests"
        # uses {{.path.basename}} from Git Directory generator (dev/staging/prod)
      destination:
        name: "{{.clusterName}}"
        # uses {{.clusterName}} from List generator
        namespace: "demo13-{{.path.basename}}-{{.region}}"
        # combines both variables
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

> **What Matrix generates — 6 combinations:**
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
git add demo-13-applicationsets/05-matrix-generator.yaml
git commit -m "feat: add demo-13 matrix generator ApplicationSet"
git push origin main

kubectl config use-context us-east-ohio
kubectl apply -f demo-13-applicationsets/05-matrix-generator.yaml
```

**Watch all six Applications appear:**
```bash
watch -n 2 'argocd app list'
```

**Expected — six Applications from one ApplicationSet:**
```text
NAME                              SYNC STATUS   HEALTH STATUS
demo13-dev-us-east-ohio           Synced        Healthy
demo13-dev-middle-east-uae        Synced        Healthy
demo13-staging-us-east-ohio       Synced        Healthy
demo13-staging-middle-east-uae    Synced        Healthy
demo13-prod-us-east-ohio          Synced        Healthy
demo13-prod-middle-east-uae       Synced        Healthy
```

**Verify correct replica counts across environments:**
```bash
kubectl get deployment demo13-app -n demo13-dev-us-east-ohio \
  -o jsonpath='{.spec.replicas}'     # → 1

kubectl get deployment demo13-app -n demo13-prod-us-east-ohio \
  -o jsonpath='{.spec.replicas}'     # → 3
```

**Key observation — adding a new environment across both clusters:**
Add a `hotfix` directory to `app-config`. The Git Directory sub-generator
detects it. Matrix multiplies: `hotfix × us-east-ohio` and
`hotfix × middle-east-uae` — two new Applications created automatically with
zero ApplicationSet YAML change.

---

## Verify Final State

```bash
# ApplicationSet
kubectl get appset -n argocd

# All six Applications
argocd app list

# All pods across all namespaces
kubectl get pods -A | grep demo13

# Verify replica count per environment
kubectl get deployment demo13-app -n demo13-dev-us-east-ohio \
  -o jsonpath='{.spec.replicas}'       # → 1
kubectl get deployment demo13-app -n demo13-staging-us-east-ohio \
  -o jsonpath='{.spec.replicas}'       # → 2
kubectl get deployment demo13-app -n demo13-prod-us-east-ohio \
  -o jsonpath='{.spec.replicas}'       # → 3
```

---

## Cleanup

```bash
# Delete all ApplicationSets (cascades to generated Applications and resources)
kubectl delete appset demo13-matrix -n argocd

# If any previous steps were not cleaned up:
kubectl delete appset demo13-git-file -n argocd
kubectl delete appset demo13-git-directory -n argocd
kubectl delete appset demo13-cluster-generator -n argocd
kubectl delete appset demo13-list-generator -n argocd

# Verify all demo-13 Applications gone
argocd app list

# Delete the second minikube profile
minikube delete -p middle-east-uae

# Rename the context back to 'minikube' (restores default profile name)
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
what each Application looks like — identical to a standard Application CRD spec
with generator variables injected via `{{.variableName}}`.

**ApplicationSets do not deploy workloads — they create Applications**
An ApplicationSet creates Application CRDs. Those Applications then deploy
workloads. Troubleshoot in order: ApplicationSet → generated Applications →
Kubernetes resources.

**`goTemplate: true` with `missingkey=error` is the production default**
Always enable Go templating for the dot-notation syntax required by nested
variable access. Always set `missingkey=error` to fail loudly on undefined
variables instead of silently creating Applications with empty fields.

**Deleting an ApplicationSet cascades to all generated Applications**
With `prune: true` in the template, this also deletes all Kubernetes resources.
This cascade is correct and intentional.

**Matrix multiplies — it does not create intent**
Matrix combines two generators. If generator 1 produces 3 items and generator
2 produces 2 items, Matrix produces 6 Applications. Matrix contributes no
variables of its own.

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
kubectl config rename-context minikube us-east-ohio
kubectl config rename-context us-east-ohio minikube

# Second minikube profile
minikube start -p middle-east-uae --cpus 2 --memory 2048
minikube delete -p middle-east-uae

# Register cluster with labels
argocd cluster add middle-east-uae \
  --label app=demo13 \
  --label region=middle-east-uae \
  --name middle-east-uae
argocd cluster list

# Update existing cluster labels
argocd cluster add <context> \
  --label key=value \
  --upsert

# ApplicationSet operations
kubectl apply -f <appset.yaml>
kubectl get appset -n argocd
kubectl describe appset <name> -n argocd
kubectl delete appset <name> -n argocd

# Generated Applications
argocd app list
kubectl get apps -n argocd

# Watch generation in real time
watch -n 2 'argocd app list'

# Inspect cluster secrets (used by Cluster generator)
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
```

---

## Lessons Learned

**1. ApplicationSets manage Applications — not workloads**
If pods are not running after applying an ApplicationSet, check the generation
chain: ApplicationSet status → generated Application sync status → Kubernetes
resources. Each tier has its own troubleshooting starting point.

**2. Always use `goTemplate: true` and `missingkey=error`**
Default string interpolation is fragile and limited. Go templating with
`missingkey=error` fails loudly on undefined variables instead of silently
creating broken Applications with empty fields.

**3. Deleting an ApplicationSet cascades to Applications and resources**
With `prune: true` in the template, deleting the ApplicationSet removes all
generated Applications and their Kubernetes resources. Manage at the
ApplicationSet level — not individual Applications.

**4. Matrix does not create intent — it multiplies existing intent**
Both inner generators must be self-sufficient. Matrix only multiplies their
outputs. If a variable is missing from either generator, the template fails.

**5. Cluster generator requires labels set before ApplicationSet is applied**
The Cluster generator selects by label at sync time. Label your clusters first,
then apply the ApplicationSet. Applying before labelling generates zero
Applications.

**6. Git Directory generator: exclude inner directories explicitly**
If your structure is `env/manifests/deployment.yaml`, the generator discovers
both `env` and `env/manifests`. Exclude `*/manifests` explicitly to prevent
unintended Applications for subdirectories.

**7. Context switching is essential in multi-cluster work**
Always verify which context is active before running `kubectl` commands.
ArgoCD runs on `us-east-ohio` — all `kubectl apply` for ApplicationSets
and all `argocd` CLI commands target this cluster. Pod verification for
`middle-east-uae` requires explicitly switching context first.

---

## What's Next

**Demo-14: Kustomize with ArgoCD**
Use Kustomize base and overlays with ArgoCD to manage environment-specific
configuration without duplicating manifests. The natural next step after
ApplicationSets — combine Kustomize overlays with a Git Directory or Matrix
generator so each generated Application automatically applies its
environment-specific overlay.