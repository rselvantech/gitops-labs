# Demo-14: Kustomize with ArgoCD — Template-Free Environment Management

## Overview

Demo-13 showed how ApplicationSets generate many Applications from one template.
This demo solves the complementary problem: how do those Applications keep their
**manifests DRY** across environments?

Without Kustomize, deploying to dev, staging, and prod means three nearly
identical copies of every YAML file — same Deployment, same Service, same
ConfigMap — differing only in a handful of values (replicas, image tag, resource
limits). Any structural change must be made in three places, and they drift apart
over time.

Kustomize solves this with a **base + overlays** pattern. The base holds the
complete, environment-neutral manifests once. Each overlay holds only the delta
for its environment — a patch that says "change replicas to 3" or "use this
image tag". ArgoCD has native Kustomize support built in: it automatically
detects a `kustomization.yaml` and runs `kustomize build` under the hood before
applying the rendered manifests to the cluster.

**What you'll learn:**
- What Kustomize is — template-free configuration management
- The base + overlays pattern and why it eliminates duplication
- What `kustomization.yaml` is and how it works
- How ArgoCD detects and runs Kustomize automatically
- Kustomize patch types — strategic merge patch vs JSON 6902 patch
- Key `kustomization.yaml` fields: `namePrefix`, `commonLabels`, `images`,
  `replicas`, `patches`
- How to point an ArgoCD Application at an overlay path
- The `spec.source.kustomize` override fields in the Application CRD
- ApplicationSet + Kustomize together — theory only

**What you'll do:**
- Build a base nginx application with Deployment, Service, and ConfigMap
- Create three overlays — dev, staging, prod — each patching only what differs
- Deploy each overlay as a separate ArgoCD Application
- Prove: one base change propagates to all three environments on next sync
- Prove: ArgoCD self-heal reverts a manual `kubectl` patch to the overlay value

---

## Prerequisites

- ✅ Completed Demo-13 — ApplicationSets understood
- ✅ ArgoCD running on minikube default profile
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ `kustomize` CLI installed (optional — for local verification)
- ✅ GitHub PAT with access to `app-config` and `argocd-config`

**Verify Prerequisites:**

### 1. ArgoCD pods running
```bash
kubectl get pods -n argocd
```

**Expected:** All pods `Running` and `1/1` Ready.

### 2. ArgoCD CLI logged in
```bash
argocd login localhost:8080 --username admin --insecure
argocd version --client
```

**Expected:** Login successful, `argocd: v3.x.x`

### 3. Kustomize CLI installed (optional but recommended)
```bash
kustomize version
```

**Expected:** `{Version:kustomize/v5.x.x ...}`

If not installed, install via:
```bash
# Mac
brew install kustomize

# Linux
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

**Recovery:** If ArgoCD is not running, start minikube and port-forward:
```bash
minikube start
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## Concepts

### What Kustomize Is

Kustomize is a **template-free** Kubernetes configuration management tool built
into both `kubectl` and ArgoCD. Unlike Helm, it does not use a templating
language with `{{ }}` placeholders. Instead it takes real, valid Kubernetes
YAML as input and applies **patches** on top — leaving the original YAML intact.

```
Without Kustomize:          With Kustomize:
────────────────────        ────────────────────────────────────────
deployment-dev.yaml         base/deployment.yaml      ← defined once
deployment-staging.yaml     overlays/dev/patch.yaml   ← only the delta
deployment-prod.yaml        overlays/staging/patch.yaml
                            overlays/prod/patch.yaml
```

**The key insight:** Kustomize transforms YAML into more YAML. The output is
always plain Kubernetes manifests — no runtime engine involved. ArgoCD runs
`kustomize build <path>` and applies the resulting manifests.

---

### The Base + Overlays Pattern

```
app-config/
└── demo-14-kustomize/
    ├── base/                      ← shared across all environments
    │   ├── kustomization.yaml     ← lists the base resources
    │   ├── deployment.yaml        ← 1 replica, generic config
    │   ├── service.yaml
    │   └── configmap.yaml
    └── overlays/
        ├── dev/                   ← only what differs from base
        │   ├── kustomization.yaml ← references base + declares patches
        │   └── replica-patch.yaml ← replicas: 1 (same as base — shown for clarity)
        ├── staging/
        │   ├── kustomization.yaml
        │   └── replica-patch.yaml ← replicas: 2
        └── prod/
            ├── kustomization.yaml
            └── replica-patch.yaml ← replicas: 3
```

**What the base contains:** Complete, working Kubernetes YAML. You could apply
the base directly with `kubectl apply -k base/` and get a running application.
The base is environment-neutral — no hard-coded environment names, no specific
replica counts.

**What overlays contain:** Only the differences. An overlay does not copy or
repeat base manifests — it references the base path and declares patches.
Kustomize merges base + patches to produce the final manifests.

---

### How `kustomization.yaml` Works

Every directory managed by Kustomize must contain a `kustomization.yaml` file.
This file is the control plane for that layer.

**Base `kustomization.yaml` — declares resources:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml    # files in this directory
  - service.yaml
  - configmap.yaml
```

**Overlay `kustomization.yaml` — references base and declares customisations:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base         # relative path to the base directory

namePrefix: dev-       # prepends "dev-" to all resource names
                       # deployment.yaml → dev-deployment

commonLabels:
  env: dev             # adds label to all resources

replicas:              # overrides replica count by name
  - name: nginx-app
    count: 1

images:                # overrides image tag by name
  - name: nginx
    newTag: "1.25"
```

---

### Kustomize Patch Types

**Strategic Merge Patch** — partial YAML that is merged with the base resource.
Most natural — looks like a normal Kubernetes manifest with only the fields
you want to change:

```yaml
# replica-patch.yaml — strategic merge patch
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app    # identifies which resource to patch
spec:
  replicas: 3        # only this field is changed, rest is from base
```

**JSON 6902 Patch** — explicit RFC 6902 JSON Patch operations. More precise
for targeted changes deep in a nested structure:

```yaml
# In kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: nginx-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
```

This demo uses strategic merge patches — they are more readable and the more
common choice in production.

---

### How ArgoCD Detects Kustomize Automatically

ArgoCD's Repository Server scans the source path for a `kustomization.yaml`
file. If one is found, it runs:

```bash
kustomize build <path>
```

This produces plain Kubernetes YAML which the Application Controller then
applies. No special annotation or flag is needed in the Application CRD —
the presence of `kustomization.yaml` is enough.

```
Application CRD:
  source.path: demo-14-kustomize/overlays/prod
                      ↓
Repository Server finds kustomization.yaml in that path
                      ↓
Repository Server runs: kustomize build overlays/prod
                      ↓
Output: plain Kubernetes YAML (Deployment with 3 replicas, prod- prefix)
                      ↓
Application Controller applies to cluster
```

---

### `spec.source.kustomize` — Optional Overrides in Application CRD

ArgoCD also allows you to override Kustomize parameters directly in the
Application CRD without modifying the overlay files. This is useful for
dynamic values like image tags:

```yaml
spec:
  source:
    path: demo-14-kustomize/overlays/prod
    kustomize:
      images:
        - nginx:1.26          # override image tag at deploy time
      namePrefix: prod-       # can also be set here instead of overlay
      commonLabels:
        version: "v2.0"
      replicas:
        - name: nginx-app
          count: 5
```

**When to use overlay files vs Application CRD overrides:**

| Approach | Use when |
|---|---|
| Overlay `kustomization.yaml` | Values are stable and version-controlled |
| Application CRD `spec.source.kustomize` | Values are dynamic (e.g. image tag from CI) |
| ArgoCD Image Updater | Image tag is auto-updated on new registry push |

For this demo, all customisations live in overlay files — the GitOps way.

---

### ApplicationSet + Kustomize Together (Theory)

In Demo-13, the Git Directory generator created one Application per directory.
Combined with Kustomize, each Application's path points to an overlay:

```yaml
# ApplicationSet — generates one Application per overlay directory
spec:
  generators:
    - git:
        directories:
          - path: "demo-14-kustomize/overlays/*"
  template:
    spec:
      source:
        path: "{{.path.path}}"   # → overlays/dev, overlays/staging, overlays/prod
```

**Result:** Adding a new environment = create a new overlay directory. The
ApplicationSet detects it, creates a new Application pointing to that overlay,
ArgoCD runs `kustomize build` on it, and the environment is deployed. This is
the production pattern for managing dozens of environments from a single repo.

This demo demonstrates this manually (three Application CRDs) so you see and
understand each step. The ApplicationSet automation sits on top of exactly this.

---

## Folder Structure

```
14-kustomize/src/
├── app-config/                        ← rselvantech/app-config (private)
│   └── demo-14-kustomize/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── configmap.yaml
│       └── overlays/
│           ├── dev/
│           │   ├── kustomization.yaml
│           │   └── replica-patch.yaml
│           ├── staging/
│           │   ├── kustomization.yaml
│           │   └── replica-patch.yaml
│           └── prod/
│               ├── kustomization.yaml
│               └── replica-patch.yaml
└── argocd-config/                     ← rselvantech/argocd-config
    └── demo-14-kustomize/
        ├── dev-app.yaml
        ├── staging-app.yaml
        └── prod-app.yaml
```

---

## Step 1: Create the Base Manifests

Initialise the local `app-config` repo:

```bash
cd gitops-labs/argo-cd-basics-to-prod/14-kustomize/src
mkdir app-config && cd app-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/app-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

Create the base directory and manifests.

**`demo-14-kustomize/base/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

**`demo-14-kustomize/base/configmap.yaml`:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: demo14
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Demo-14: Kustomize with ArgoCD</title></head>
    <body>
      <h1>Demo-14: Kustomize with ArgoCD</h1>
      <p>Base manifest — environment-neutral</p>
      <p>This content is from the base ConfigMap.</p>
      <p>Each environment overlay patches only what differs.</p>
    </body>
    </html>
```

**`demo-14-kustomize/base/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: demo14
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
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "100m"
              memory: "64Mi"
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: nginx-config
```

**`demo-14-kustomize/base/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo14
spec:
  selector:
    app: nginx-app
  ports:
    - port: 80
      targetPort: 80
```

**Verify the base builds correctly (optional but recommended):**
```bash
kustomize build demo-14-kustomize/base/
```

**Expected:** Full YAML output — Deployment with 1 replica, Service, ConfigMap.
No errors.

---

## Step 2: Create the Overlays

Each overlay references the base and patches only what differs between
environments. The three overlays differ only in replica count and a `namePrefix`
that makes the resource names environment-specific.

### dev overlay

**`demo-14-kustomize/overlays/dev/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: dev-

commonLabels:
  env: dev

replicas:
  - name: nginx-app
    count: 1

patches:
  - path: replica-patch.yaml
```

**`demo-14-kustomize/overlays/dev/replica-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app       # must match base resource name (before namePrefix is applied)
  namespace: demo14
spec:
  template:
    metadata:
      labels:
        env: dev
        version: "1.25"
```

### staging overlay

**`demo-14-kustomize/overlays/staging/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: staging-

commonLabels:
  env: staging

replicas:
  - name: nginx-app
    count: 2

patches:
  - path: replica-patch.yaml
```

**`demo-14-kustomize/overlays/staging/replica-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: demo14
spec:
  template:
    metadata:
      labels:
        env: staging
        version: "1.25"
```

### prod overlay

**`demo-14-kustomize/overlays/prod/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: prod-

commonLabels:
  env: prod

replicas:
  - name: nginx-app
    count: 3

patches:
  - path: replica-patch.yaml
```

**`demo-14-kustomize/overlays/prod/replica-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: demo14
spec:
  template:
    metadata:
      labels:
        env: prod
        version: "1.25"
```

**Verify each overlay builds locally:**
```bash
kustomize build demo-14-kustomize/overlays/dev/
kustomize build demo-14-kustomize/overlays/staging/
kustomize build demo-14-kustomize/overlays/prod/
```

**Expected for dev:** Deployment named `dev-nginx-app`, 1 replica, label `env: dev`
**Expected for staging:** Deployment named `staging-nginx-app`, 2 replicas
**Expected for prod:** Deployment named `prod-nginx-app`, 3 replicas

**Push to `app-config`:**
```bash
git add demo-14-kustomize/
git commit -m "feat: add demo-14 kustomize base and overlays"
git push origin main
```

---

## Step 3: Create the Namespace

The namespace is shared across all three overlay deployments:

```bash
kubectl create namespace demo14
```

> **Why not use `CreateNamespace=true` sync option here?** All three overlays
> deploy into the same `demo14` namespace. If all three Applications have
> `CreateNamespace=true`, the first sync creates the namespace and the other
> two re-apply it harmlessly. Both approaches work — creating it imperatively
> here is simpler and avoids any race condition on first sync.

---

## Step 4: Create Application CRDs

Set up `argocd-config` local repo:

```bash
cd gitops-labs/argo-cd-basics-to-prod/14-kustomize/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

Each Application CRD points to one overlay path. ArgoCD detects
`kustomization.yaml` in the path and runs Kustomize automatically — no
special configuration needed in the Application CRD.

**`demo-14-kustomize/dev-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo14-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/app-config.git
    targetRevision: HEAD
    path: demo-14-kustomize/overlays/dev    # ← ArgoCD finds kustomization.yaml here
  destination:
    server: https://kubernetes.default.svc
    namespace: demo14
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**`demo-14-kustomize/staging-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo14-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/app-config.git
    targetRevision: HEAD
    path: demo-14-kustomize/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: demo14
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**`demo-14-kustomize/prod-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo14-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/app-config.git
    targetRevision: HEAD
    path: demo-14-kustomize/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: demo14
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Push and apply:**
```bash
git add demo-14-kustomize/
git commit -m "feat: add demo-14 Application CRDs for dev, staging, prod"
git push origin main

kubectl apply -f demo-14-kustomize/dev-app.yaml
kubectl apply -f demo-14-kustomize/staging-app.yaml
kubectl apply -f demo-14-kustomize/prod-app.yaml
```

**Verify all three Applications created:**
```bash
argocd app list
```

**Expected:**
```text
NAME             SYNC STATUS   HEALTH STATUS
demo14-dev       Synced        Healthy
demo14-staging   Synced        Healthy
demo14-prod      Synced        Healthy
```

---

## Step 5: Verify Kustomize Was Applied Correctly

Check that each environment has the correct replica count and name prefix:

```bash
kubectl get deployments -n demo14
```

**Expected — three deployments, each with correct namePrefix and replicas:**
```text
NAME                  READY   UP-TO-DATE   AVAILABLE
dev-nginx-app         1/1     1            1
staging-nginx-app     2/2     2            2
prod-nginx-app        3/3     3            3
```

**Verify labels were applied by Kustomize `commonLabels`:**
```bash
kubectl get pods -n demo14 --show-labels | grep env
```

**Expected:** Each pod has `env=dev`, `env=staging`, or `env=prod` label.

**Verify what ArgoCD rendered from Kustomize — inspect a deployed manifest:**
```bash
kubectl get deployment prod-nginx-app -n demo14 -o yaml | grep -A5 "replicas:"
```

**Expected:** `replicas: 3` — confirming the prod overlay patch was applied.

**Access the dev application:**
```bash
kubectl port-forward svc/dev-nginx-service -n demo14 8081:80
```

Open `http://localhost:8081` — you should see the nginx page from the base
ConfigMap.

---

## Step 6: Prove One Base Change Propagates to All Environments

This step demonstrates the core value of Kustomize: change the base once,
all environments pick it up.

Update the base `configmap.yaml` to add a new message:

```yaml
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Demo-14: Kustomize with ArgoCD</title></head>
    <body>
      <h1>Demo-14: Kustomize with ArgoCD</h1>
      <p>Base manifest — environment-neutral</p>
      <p>This content is from the base ConfigMap.</p>
      <p>Each environment overlay patches only what differs.</p>
      <p><strong>Updated: base change propagates to all environments.</strong></p>
    </body>
    </html>
```

**Push the base change:**
```bash
git add demo-14-kustomize/base/configmap.yaml
git commit -m "feat: update base configmap with new message"
git push origin main
```

**Watch all three Applications detect the change:**
```bash
watch -n 3 'argocd app list'
```

**Expected — all three go OutOfSync, then Synced:**
```text
NAME             SYNC STATUS   HEALTH STATUS
demo14-dev       OutOfSync     Healthy      ← detected base change
demo14-staging   OutOfSync     Healthy
demo14-prod      OutOfSync     Healthy

# Seconds later (automated sync):
NAME             SYNC STATUS   HEALTH STATUS
demo14-dev       Synced        Healthy
demo14-staging   Synced        Healthy
demo14-prod      Synced        Healthy
```

**Verify the update reached all three:**
```bash
kubectl port-forward svc/dev-nginx-service -n demo14 8081:80
# open http://localhost:8081 — see updated message

kubectl port-forward svc/prod-nginx-service -n demo14 8082:80
# open http://localhost:8082 — same updated message
```

**Key observation:** One Git commit to the base updated all three environments
automatically. The overlay patch files were not touched.

---

## Step 7: Prove Self-Heal Reverts Manual Replica Change

Manually scale the prod deployment, then watch ArgoCD revert it:

```bash
# Manually scale prod to 1 replica (should be 3)
kubectl scale deployment prod-nginx-app -n demo14 --replicas=1

# Watch ArgoCD detect drift and self-heal
kubectl get deployment prod-nginx-app -n demo14 -w
```

**Expected — ArgoCD immediately reverts to 3 replicas:**
```text
NAME            READY   UP-TO-DATE   AVAILABLE
prod-nginx-app  1/1     1            1    ← manual scale
prod-nginx-app  3/3     3            3    ← ArgoCD self-healed back to overlay value
```

**Verify ArgoCD UI shows the self-heal event:**
```
http://localhost:8080 → demo14-prod → App Details → History
```

---

## Verify Final State

```bash
# All three Applications synced and healthy
argocd app list

# Correct deployments with namePrefix
kubectl get deployments -n demo14

# Correct replica counts
kubectl get deployment dev-nginx-app     -n demo14 -o jsonpath='{.spec.replicas}'  # 1
kubectl get deployment staging-nginx-app -n demo14 -o jsonpath='{.spec.replicas}'  # 2
kubectl get deployment prod-nginx-app    -n demo14 -o jsonpath='{.spec.replicas}'  # 3

# Environment labels applied by Kustomize
kubectl get pods -n demo14 -L env
```

---

## Cleanup

```bash
# Delete all three Applications
kubectl delete app demo14-dev demo14-staging demo14-prod -n argocd

# Delete namespace and all resources
kubectl delete namespace demo14
```

---

## Key Concepts Summary

**Kustomize is template-free**
Base manifests are real, valid Kubernetes YAML — no placeholders, no templating
syntax. Overlays declare only the differences. The output is always plain YAML.

**ArgoCD detects Kustomize automatically**
Pointing `spec.source.path` at a directory containing `kustomization.yaml` is
all that is needed. ArgoCD runs `kustomize build` under the hood — same as
plain YAML, same as Helm. No special flag required.

**`namePrefix` prevents resource name collisions across environments**
Three overlays deploying to the same namespace need distinct resource names.
`namePrefix: dev-` ensures `nginx-app` becomes `dev-nginx-app` — different from
`staging-nginx-app` and `prod-nginx-app`.

**One base change reaches all environments**
The overlay only patches what is different. Everything else comes from the base.
Update the base → all overlays pick it up on the next sync.

**Kustomize vs Helm — when to choose which**

| Aspect | Kustomize | Helm |
|---|---|---|
| Syntax | Plain YAML + patches | Go templates with `{{ }}` |
| Base manifests | Valid YAML, deployable as-is | Templates, not deployable alone |
| Environment differences | Patches in overlay | `values.yaml` per environment |
| Learning curve | Lower — no templating to learn | Higher — template language |
| Best for | Kubernetes-native, small teams | Complex charts, public distribution |
| ArgoCD detection | Automatic (`kustomization.yaml`) | Automatic (`Chart.yaml`) |

---

## Commands Reference

```bash
# Local Kustomize commands
kustomize build base/
kustomize build overlays/dev/
kustomize build overlays/prod/

# Apply with kubectl (outside ArgoCD)
kubectl apply -k overlays/dev/

# Application CRD apply
kubectl apply -f demo-14-kustomize/dev-app.yaml

# ArgoCD
argocd app list
argocd app get demo14-prod
argocd app diff demo14-prod

# Verify rendered manifests
kubectl get deployment prod-nginx-app -n demo14 -o yaml

# Check labels
kubectl get pods -n demo14 --show-labels
kubectl get pods -n demo14 -L env
```

---

## Lessons Learned

**1. `namePrefix` is essential when multiple overlays deploy to the same namespace**
Without `namePrefix`, all three overlays try to create `nginx-app` in `demo14`
— they conflict. The first sync creates it, subsequent syncs fight over ownership.
Always use `namePrefix` or `nameSuffix` in overlays sharing a namespace.

**2. Patch target names refer to base names, not post-prefix names**
A strategic merge patch identifies the resource by its name in the base YAML.
The `namePrefix` is applied after patching. If your base has `name: nginx-app`
and your overlay has `namePrefix: prod-`, the patch still targets `nginx-app`
not `prod-nginx-app`.

**3. `kustomize build` locally before pushing**
Run `kustomize build overlays/dev/` before committing. ArgoCD will show the
same error if the build fails, but catching it locally is faster and cleaner.

**4. Base changes are powerful — use them carefully in production**
A base change propagates to all environments simultaneously on the next sync.
In production, use separate branches or progressive delivery strategies if you
need to promote changes from dev → staging → prod sequentially.

**5. Kustomize is built into `kubectl` — no installation needed in CI**
`kubectl apply -k overlays/prod/` runs Kustomize internally. No separate
`kustomize` binary needed in CI pipelines or in ArgoCD.

---

## What's Next

**Demo-15: End-to-End GitOps on EKS — GitLab CI + ArgoCD + Image Updater + Cognito RBAC**
Deploy the Goals App on Amazon EKS using GitLab as the source code and config
repository. Build a complete CI/CD pipeline where GitLab CI builds and pushes
images to GitLab Container Registry, ArgoCD Image Updater detects new tags and
writes back to the config repo, ArgoCD syncs the cluster, and AWS Cognito
provides SSO with role-based access control — completing the full production
GitOps lifecycle.