# Demo-16: ApplicationSet Progressive Sync — Safe Multi-Environment Rollouts

## Overview

Demo-14 ended with one ApplicationSet generating three Applications — dev,
staging, prod — each running its own Kustomize overlay. By default, when a
change is pushed to Git, **all three Applications sync simultaneously**. If the
change is bad — a broken image tag, a misconfigured environment variable, a
resource limit too low — it hits dev and production at exactly the same moment.
There is no gate, no pause, no chance to verify dev is healthy before staging
gets the change, and no protection for production.

This is the **blast radius problem**. At small scale it is manageable. At 10
clusters, 50 environments, or 100 microservices, simultaneous sync of a broken
change becomes a global outage.

ApplicationSet Progressive Sync solves this with a `strategy: RollingSync`
block added to the ApplicationSet. You define ordered **steps** — each step
selects a subset of generated Applications by label. The controller syncs step
1, waits for all Applications in that step to reach `Healthy`, then advances
to step 2. A failure in any step halts the entire rollout — subsequent steps
are untouched and remain at their current good state.

**The scenario in this demo:** You manage a company's web platform. The nginx
application is deployed to three environments — dev, staging, and prod — each
with its own Kustomize overlay. You are rolling out a new version of the
platform (`v2`) that updates the index page content. Without Progressive Sync,
v2 hits all three environments simultaneously. With RollingSync, dev gets v2
first. If dev is healthy, staging gets it. If staging is healthy, prod gets it.
A bad v2 config stops at dev. Production stays on v1.

**What you'll learn:**
- The blast radius problem — why simultaneous sync is dangerous at scale
- How `strategy: RollingSync` changes ApplicationSet sync behaviour
- `rollingSync.steps` with `matchExpressions` — defining ordered stages
- How Application `metadata.labels` act as step selectors
- `maxUpdate` — controlling parallelism within a step
- How to enable progressive syncs on the ApplicationSet controller
- How to observe rollout progress step by step
- What happens when a step Application fails to become Healthy — rollout halt
- How to fix a halted rollout and let it resume automatically
- `deletionOrder: Reverse` — safe teardown order

**What you'll do:**
- Build Demo-16's own Kustomize base and overlays in `demo-16-progressive-sync/`
- Push a new ApplicationSet that uses `strategy: RollingSync` with three steps
- Observe a clean rollout: dev → staging → prod in sequence
- Push a broken image tag — watch the rollout halt at dev, staging and prod untouched
- Fix the image in Git — watch the rollout resume and complete
- Add a `canary` overlay and demonstrate `maxUpdate: 1`

**What is reused from Demo-14 and what is new:**

| | Demo-14 | Demo-16 |
|---|---|---|
| Kustomize base + overlays | `demo-14-kustomize/` in `gitops-apps-config` | **New:** `demo-16-progressive-sync/` |
| Namespaces | `demo14-dev`, `demo14-staging`, `demo14-prod` | **New:** `demo16-dev`, `demo16-staging`, `demo16-prod` |
| ApplicationSet | `demo14-kustomize` — no strategy | **New:** `demo16-progressive` — with `strategy: RollingSync` |
| Application names | `demo14-dev/staging/prod` | **New:** `demo16-dev/staging/prod` |
| Content (ConfigMap) | Generic base content | **v1 → v2 rollout** — visible change to demonstrate progressive rollout |
| Demo-14 resources | Running independently | Untouched — Demo-16 runs alongside it |

Demo-14 continues running completely independently. Demo-16 creates its own
manifests, its own overlays, and its own ApplicationSet. Nothing in Demo-14
is modified.

---

## Prerequisites

- ✅ Completed Demo-14 — Kustomize base/overlays pattern understood,
  ApplicationSet + Kustomize combination understood
- ✅ ArgoCD running on minikube default profile
- ✅ ArgoCD CLI installed and logged in
- ✅ `kustomize` CLI installed
- ✅ GitHub PAT with `Contents: Read/Write` access to `gitops-apps-config`
  and `argocd-config`

**Verify Prerequisites:**

### 1. ArgoCD running
```bash
kubectl get pods -n argocd
```

**Expected:** All pods `Running` and `1/1` Ready.

### 2. Demo-14 ApplicationSet still running (verify Demo-14 is complete)
```bash
kubectl get appset demo14-kustomize -n argocd
argocd app list | grep demo14
```

**Expected:**
```text
NAME               AGE
demo14-kustomize   xxm

NAME             SYNC STATUS   HEALTH STATUS
demo14-dev       Synced        Healthy
demo14-staging   Synced        Healthy
demo14-prod      Synced        Healthy
```

### 3. Repos registered with ArgoCD
```bash
argocd repo list
```

**Expected:** `gitops-apps-config` and `argocd-config` both `Successful`.

---

## Concepts

### The Demo Scenario — Platform v1 to v2 Rollout

Your company runs a web platform (nginx-based) across three environments:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Platform Deployment                              │
│                                                                      │
│  dev environment          staging environment      prod environment  │
│  ┌──────────────┐         ┌──────────────┐         ┌─────────────┐  │
│  │  nginx       │         │  nginx       │         │  nginx      │  │
│  │  platform v1 │         │  platform v1 │         │  platform v1│  │
│  │  1 replica   │         │  2 replicas  │         │  3 replicas │  │
│  │  demo16-dev  │         │ demo16-stg   │         │ demo16-prod │  │
│  └──────────────┘         └──────────────┘         └─────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

Each environment runs the same nginx image but with environment-specific
configuration managed by a Kustomize overlay — its own namespace, replica count,
and environment label. This is identical to the Demo-14 pattern.

You now need to roll out **platform v2** — a new version of the index page that
serves updated content. In production this would represent a new application
version, a configuration change, or an updated image tag. The challenge: you
cannot afford to break production while experimenting in dev. You need dev to
succeed before staging gets the change, and staging to succeed before prod.

```
Goal: dev → staging → prod in sequence
  If dev fails → staging and prod stay on v1
  If staging fails → prod stays on v1
  If all succeed → all three environments run v2
```

This is exactly what `strategy: RollingSync` delivers.

---

### The Blast Radius Problem

Without Progressive Sync, all generated Applications sync simultaneously when
a change is pushed:

```
Git push: platform v2
               │
               ├─► demo16-dev     syncs immediately (v1 → v2)
               ├─► demo16-staging syncs immediately (v1 → v2)  ← no gate
               └─► demo16-prod    syncs immediately (v1 → v2)  ← no gate
```

If v2 has a bug — wrong image tag, broken config, resource limit too low to
start — all three environments degrade at the same time:

```
After bad push:
  demo16-dev     → Degraded  ✗
  demo16-staging → Degraded  ✗  ← could have been protected
  demo16-prod    → Degraded  ✗  ← should never have been touched
```

Production is down alongside dev. Recovery requires reverting across all three
environments simultaneously. At 10 clusters this is a global outage.

---

### How Progressive Sync Solves This

`strategy: RollingSync` makes the ApplicationSet controller own sync timing.
It processes steps in order, treating each step's Applications as a gate:

```
Git push: platform v2
               │
               ▼
          Step 1: sync demo16-dev
               │
               ├─ demo16-dev Healthy → advance to step 2
               └─ demo16-dev Degraded → HALT (staging and prod untouched)
               │
               ▼
          Step 2: sync demo16-staging
               │
               ├─ demo16-staging Healthy → advance to step 3
               └─ demo16-staging Degraded → HALT (prod untouched)
               │
               ▼
          Step 3: sync demo16-prod
               └─ rollout complete
```

```
Good change:                    Bad change (v2 broken):
dev   → Healthy ✅              dev   → Degraded ✗
staging → Healthy ✅            staging → on v1 ✅  (protected)
prod  → Healthy ✅              prod  → on v1 ✅  (protected)
```

---

### How Application Labels Connect to Step Selectors

Progressive Sync steps select Applications by their Kubernetes `metadata.labels`.
The labels are set in the ApplicationSet `template.metadata.labels` block — they
are labels on the ArgoCD Application object, not on the Kubernetes workloads.

```yaml
# ApplicationSet template — labels applied to each generated Application
template:
  metadata:
    name: "demo16-{{.path.basename}}"
    labels:
      env: "{{.path.basename}}"     # → dev, staging, prod per overlay
      app: demo16-platform

# RollingSync step — selects Applications by those labels
rollingSync:
  steps:
    - matchExpressions:
        - key: env
          operator: In
          values: [dev]             # matches demo16-dev (env=dev)
    - matchExpressions:
        - key: env
          operator: In
          values: [staging]         # matches demo16-staging
    - matchExpressions:
        - key: env
          operator: In
          values: [prod]            # matches demo16-prod
```

The label value `"{{.path.basename}}"` resolves to the overlay directory name
(`dev`, `staging`, `prod`) — the same variable used to set the Application name
and namespace. This means the step selector and the overlay structure are
automatically consistent.

---

### What Changes vs Demo-14 ApplicationSet

Demo-14's ApplicationSet has no `strategy:` block — all three Applications sync
simultaneously. Demo-16's ApplicationSet adds exactly one new block:

```yaml
# Demo-14 ApplicationSet (no strategy):
spec:
  generators: [...]
  template: [...]

# Demo-16 ApplicationSet (with strategy):
spec:
  strategy:                       ← NEW: this entire block is what changes
    type: RollingSync
    rollingSync:
      steps: [...]
  generators: [...]               ← same generator logic as Demo-14
  template:                       ← same template structure as Demo-14
    metadata:
      labels:
        env: "{{.path.basename}}" ← same label pattern as Demo-14 Step 8
```

The generator, the source path, the destination, the Kustomize integration —
all identical to Demo-14. The only addition is the `strategy:` block. This is
why Progressive Sync is described as a safety layer on top of the ApplicationSet
pattern, not a replacement for it.

---

### Key Behaviours of RollingSync

**RollingSync disables individual Application autosync during a rollout.**
The ApplicationSet controller manages when each Application syncs. Individual
Applications no longer self-sync while a rollout is in progress. The `automated`
syncPolicy in the template is overridden by the strategy during active rollouts.

**All Applications in a step must be Healthy before the next step begins.**
An Application is "complete" when it has synced successfully, entered
`Progressing` state during pod rollout, and exited `Progressing` into `Healthy`.
`ImagePullBackOff`, `CrashLoopBackOff`, or failed readiness probes keep an
Application `Degraded` — the step never completes.

**Applications not matched by any step are excluded from the rollout.**
If a generated Application's labels do not match any `matchExpressions`, it
is not synced by the rolling strategy and must be synced manually.

**Rollout resume is automatic when the root cause is fixed in Git.**
Push the fix → ApplicationSet controller re-evaluates step 1 → if Application
reaches Healthy → rollout advances. No manual trigger needed.

---

### Enabling Progressive Sync on the Controller

Progressive Syncs must be explicitly enabled at the ArgoCD ApplicationSet
controller level. The feature is disabled by default. Without enabling it,
the `strategy:` block in the ApplicationSet YAML is **silently ignored** —
Applications sync simultaneously as before with no error or warning.

Enable via the `argocd-cmd-params-cm` ConfigMap:
```yaml
applicationsetcontroller.enable.progressive.syncs: "true"
```

An ApplicationSet controller pod restart is required after patching the ConfigMap.

---

## Folder Structure

```
gitops-apps-config/
├── demo-14-kustomize/           ← Demo-14 (untouched, running independently)
│   ├── base/
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
└── demo-16-progressive-sync/    ← Demo-16 adds this (new, separate)
    ├── base/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml      ← image: nginx:1.25 (platform v1)
    │   ├── service.yaml
    │   └── configmap.yaml       ← v1 content: "Platform v1"
    └── overlays/
        ├── dev/
        │   ├── kustomization.yaml  ← namespace: demo16-dev, namePrefix: dev-, replicas: 1
        │   └── env-patch.yaml
        ├── staging/
        │   ├── kustomization.yaml  ← namespace: demo16-staging, namePrefix: staging-, replicas: 2
        │   └── env-patch.yaml
        └── prod/
            ├── kustomization.yaml  ← namespace: demo16-prod, namePrefix: prod-, replicas: 3
            └── env-patch.yaml

argocd-config/
├── demo-14-kustomize/           ← Demo-14 (untouched)
│   └── appset-kustomize.yaml
└── demo-16-progressive-sync/    ← Demo-16 adds this (new, separate)
    └── appset-progressive.yaml  ← new ApplicationSet with strategy: RollingSync
```

**Key difference from Demo-14:**
- All resources live under `demo-16-progressive-sync/` — completely separate from Demo-14
- Namespaces are `demo16-*` — no overlap with Demo-14's `demo14-*` namespaces
- The base ConfigMap starts at **v1** — the rollout in this demo updates it to **v2**
- The ApplicationSet is named `demo16-progressive` — a new resource alongside `demo14-kustomize`
- Demo-14's ApplicationSet and manifests are **never modified**

---

## Step 1: Enable Progressive Syncs on the ApplicationSet Controller

Progressive Syncs must be enabled before applying any ApplicationSet with a
`strategy:` block. Without this flag, the strategy block is silently ignored.

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data": {"applicationsetcontroller.enable.progressive.syncs": "true"}}'
```

**Verify:**
```bash
kubectl get configmap argocd-cmd-params-cm -n argocd \
  -o jsonpath='{.data.applicationsetcontroller\.enable\.progressive\.syncs}'
```

**Expected:**
```text
true
```

**Restart the ApplicationSet controller:**
```bash
kubectl rollout restart deployment \
  -l app.kubernetes.io/component=applicationset-controller \
  -n argocd

kubectl rollout status deployment \
  -l app.kubernetes.io/component=applicationset-controller \
  -n argocd
```

**Expected:**
```text
deployment "my-argo-argocd-applicationset-controller" successfully rolled out
```

**Verify the flag is active:**
```bash
kubectl logs -n argocd \
  -l app.kubernetes.io/component=applicationset-controller \
  --tail=30 | grep -i progressive
```

**Expected:**
```text
... "Progressive sync enabled" ...
```

> **Note:** The Demo-14 `demo14-kustomize` ApplicationSet has no `strategy:`
> block. Enabling this flag on the controller does not affect it — it continues
> to sync all three Applications simultaneously as before. Only ApplicationSets
> with an explicit `strategy: RollingSync` block are affected.

---

## Step 2: Push Demo-16 Manifests — Platform v1

Create the Demo-16 Kustomize base and overlays. The base configmap serves
**platform v1** content — this is the baseline all three environments start on.
The rollout in Step 5 will update it to v2.

The structure is identical to Demo-14's base + overlays pattern. The differences
are the folder name (`demo-16-progressive-sync/`), the namespace prefix
(`demo16-`), and the ConfigMap content which explicitly identifies the platform
version to make the rollout observable.

```bash
cd gitops-labs/argo-cd-basics-to-prod/16-progressive-sync/src
mkdir -p gitops-apps-config && cd gitops-apps-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

### Base manifests

**`demo-16-progressive-sync/base/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

**`demo-16-progressive-sync/base/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-app
  # no namespace — set by each overlay kustomization.yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: platform-app
  template:
    metadata:
      labels:
        app: platform-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25           # platform v1 image
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
            name: platform-config
```

**`demo-16-progressive-sync/base/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: platform-svc
  # no namespace — set by each overlay kustomization.yaml
spec:
  selector:
    app: platform-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-16-progressive-sync/base/configmap.yaml`:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-config
  # no namespace — set by each overlay kustomization.yaml
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body style="font-family: sans-serif; padding: 2rem;">
      <h1>Company Platform</h1>
      <h2 style="color: #2563eb;">Version: v1</h2>
      <p>Environment: set by overlay</p>
      <p>This is platform v1 — the baseline version before the rollout.</p>
    </body>
    </html>
```

> **Why the ConfigMap shows the version explicitly:** The platform v1 → v2
> rollout is the central demonstration of this demo. Making the version visible
> in the served page lets you verify exactly which version each environment is
> running at any point during the rollout — by port-forwarding to each
> environment and checking the page content.

### Overlay manifests

**`demo-16-progressive-sync/overlays/dev/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base             # references Demo-16 base (not Demo-14's base)

namespace: demo16-dev      # Demo-16 specific — no overlap with demo14-dev

namePrefix: dev-           # platform-app → dev-platform-app

commonLabels:
  env: dev
  app: demo16-platform

replicas:
  - name: platform-app
    count: 1               # dev: minimal replicas

patches:
  - path: env-patch.yaml
```

**`demo-16-progressive-sync/overlays/dev/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-app       # base name — before namePrefix is applied
spec:
  template:
    metadata:
      labels:
        env: dev
        version: v1        # tracks which version is running — changes to v2 on rollout
```

**`demo-16-progressive-sync/overlays/staging/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: demo16-staging

namePrefix: staging-

commonLabels:
  env: staging
  app: demo16-platform

replicas:
  - name: platform-app
    count: 2               # staging: near-production scale

patches:
  - path: env-patch.yaml
```

**`demo-16-progressive-sync/overlays/staging/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-app
spec:
  template:
    metadata:
      labels:
        env: staging
        version: v1
```

**`demo-16-progressive-sync/overlays/prod/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: demo16-prod

namePrefix: prod-

commonLabels:
  env: prod
  app: demo16-platform

replicas:
  - name: platform-app
    count: 3               # prod: full replicas for availability

patches:
  - path: env-patch.yaml
```

**`demo-16-progressive-sync/overlays/prod/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-app
spec:
  template:
    metadata:
      labels:
        env: prod
        version: v1
```

**Verify each overlay builds correctly before pushing:**
```bash
for env in dev staging prod; do
  echo "=== $env ==="
  kustomize build demo-16-progressive-sync/overlays/$env/ \
    | grep -E "namespace:|replicas:|name: (dev|staging|prod)-platform"
done
```

**Expected:**
```text
=== dev ===
  namespace: demo16-dev
  replicas: 1
  name: dev-platform-app
=== staging ===
  namespace: demo16-staging
  replicas: 2
  name: staging-platform-app
=== prod ===
  namespace: demo16-prod
  replicas: 3
  name: prod-platform-app
```

**Push:**
```bash
git add demo-16-progressive-sync/
git commit -m "feat: add demo-16 progressive sync base and overlays (platform v1)"
git push origin main
```

---

## Step 3: Create the Progressive Sync ApplicationSet

This is the core of Demo-16. The ApplicationSet is structurally the same as
Demo-14's — same Git Directory generator, same Kustomize overlay source path
pattern, same destination logic. The one addition is the `strategy: RollingSync`
block that changes sync behaviour from simultaneous to staged.

```bash
cd gitops-labs/argo-cd-basics-to-prod/16-progressive-sync/src
mkdir -p argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase

mkdir -p demo-16-progressive-sync
```

**`demo-16-progressive-sync/appset-progressive.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo16-progressive
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - "missingkey=error"

  # ── Progressive Sync Strategy ──────────────────────────────────────────────
  # This block is the only structural difference from Demo-14's ApplicationSet.
  # The generator, template source, and destination are identical in pattern.
  # What changes: sync timing is now controlled by the controller, not by
  # individual Application autosync.
  #
  # Step 1: dev syncs first — lowest risk, first to catch problems
  # Step 2: staging syncs after dev is Healthy — validates at near-prod scale
  # Step 3: prod syncs last — only after staging passes
  #
  # Failure at any step: subsequent steps are not attempted.
  # The remaining Applications stay at their current revision.
  # ────────────────────────────────────────────────────────────────────────────
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: env               # matches the label set in template.metadata.labels
              operator: In
              values:
                - dev                # selects demo16-dev (env=dev)

        - matchExpressions:
            - key: env
              operator: In
              values:
                - staging            # selects demo16-staging (env=staging)

        - matchExpressions:
            - key: env
              operator: In
              values:
                - prod               # selects demo16-prod (env=prod)

  generators:
    - git:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        revision: main
        directories:
          - path: "demo-16-progressive-sync/overlays/*"
          # discovers: overlays/dev, overlays/staging, overlays/prod
          # same pattern as Demo-14, different base path

  template:
    metadata:
      name: "demo16-{{.path.basename}}"
      # → demo16-dev, demo16-staging, demo16-prod
      labels:
        env: "{{.path.basename}}"
        # CRITICAL: these labels are what rollingSync.steps matchExpressions select
        # env=dev   → matched by step 1
        # env=staging → matched by step 2
        # env=prod  → matched by step 3
        app: demo16-platform
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "{{.path.path}}"
        # → demo-16-progressive-sync/overlays/dev
        # → demo-16-progressive-sync/overlays/staging
        # → demo-16-progressive-sync/overlays/prod
        # ArgoCD finds kustomization.yaml in each path → runs kustomize build
      destination:
        server: https://kubernetes.default.svc
        namespace: "demo16-{{.path.basename}}"
        # → demo16-dev, demo16-staging, demo16-prod
        # matches namespace: field in each overlay kustomization.yaml
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        # Note: RollingSync overrides automated sync during active rollouts.
        # The controller manages sync timing — individual Applications do not
        # self-sync while a rollout step is in progress.
        syncOptions:
          - CreateNamespace=true
```

**Push and apply:**
```bash
git add demo-16-progressive-sync/appset-progressive.yaml
git commit -m "feat: add demo-16 progressive sync ApplicationSet"
git push origin main

kubectl apply -f demo-16-progressive-sync/appset-progressive.yaml
```

**Verify the ApplicationSet was created with the strategy:**
```bash
kubectl get appset demo16-progressive -n argocd \
  -o jsonpath='{.spec.strategy.type}'
```

**Expected:**
```text
RollingSync
```

**Watch Applications appear — they should be generated in step order:**
```bash
watch -n 2 'argocd app list | grep demo16'
```

**Expected — initial sync in staged order:**
```text
NAME              CLUSTER     NAMESPACE        SYNC STATUS   HEALTH STATUS
demo16-dev        in-cluster  demo16-dev       Synced        Healthy
demo16-staging    in-cluster  demo16-staging   Synced        Healthy
demo16-prod       in-cluster  demo16-prod      Synced        Healthy
```

**Verify Application labels — step selectors match these:**
```bash
kubectl get app -n argocd --show-labels | grep demo16
```

**Expected:**
```text
demo16-dev      ... env=dev,app=demo16-platform
demo16-staging  ... env=staging,app=demo16-platform
demo16-prod     ... env=prod,app=demo16-platform
```

**Verify platform v1 is running on all three environments:**
```bash
kubectl get deployments -A | grep demo16
```

**Expected:**
```text
NAMESPACE       NAME                 READY   UP-TO-DATE   AVAILABLE
demo16-dev      dev-platform-app     1/1     1            1
demo16-staging  staging-platform-app 2/2     2            2
demo16-prod     prod-platform-app    3/3     3            3
```

**Access and confirm v1 is serving on dev:**
```bash
kubectl port-forward svc/dev-platform-svc -n demo16-dev 8085:80
```

Open `http://localhost:8085` — page shows **"Version: v1"**.

---

## Step 4: Understand the ApplicationSet Status Rollout Tracking

Before triggering a rollout, inspect how ArgoCD tracks progressive sync status.
This is the control plane view — it shows which step is active and which
Applications have completed.

```bash
kubectl get appset demo16-progressive -n argocd \
  -o jsonpath='{.status}' | python3 -m json.tool
```

**Expected structure:**
```json
{
  "applicationStatus": [
    {
      "application": "demo16-dev",
      "status": "Healthy",
      "step": "1"
    },
    {
      "application": "demo16-staging",
      "status": "Healthy",
      "step": "2"
    },
    {
      "application": "demo16-prod",
      "status": "Healthy",
      "step": "3"
    }
  ]
}
```

This status is updated in real time during a rollout. The `step` field shows
which step processed each Application. `status` shows the Application's last
known health state as seen by the progressive sync controller.

---

## Step 5: Clean Rollout — Platform v1 → v2

Trigger a rollout by updating the base ConfigMap from v1 to v2. This simulates
a new platform release. The change propagates to all three environments —
but in order, gated by Healthy status at each step.

**In `gitops-apps-config`:**

Edit `demo-16-progressive-sync/base/configmap.yaml` — update the version:

```yaml
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body style="font-family: sans-serif; padding: 2rem;">
      <h1>Company Platform</h1>
      <h2 style="color: #16a34a;">Version: v2</h2>
      <p>Environment: set by overlay</p>
      <p>Platform v2 — new release rolled out via Progressive Sync.</p>
      <p>This environment received the update after the previous stage passed.</p>
    </body>
    </html>
```

```bash
git add demo-16-progressive-sync/base/configmap.yaml
git commit -m "feat: release platform v2 — update base configmap"
git push origin main
```

**Watch the staged rollout in real time:**
```bash
watch -n 2 'argocd app list | grep demo16'
```

**Expected — step-by-step progression:**
```text
# Step 1 active — only dev syncs:
NAME              SYNC STATUS   HEALTH STATUS
demo16-dev        OutOfSync     Healthy      ← ArgoCD syncing step 1
demo16-staging    Synced        Healthy      ← waiting, untouched
demo16-prod       Synced        Healthy      ← waiting, untouched

# Step 1 complete — dev is Healthy, step 2 begins:
demo16-dev        Synced        Healthy      ← step 1 done ✅
demo16-staging    OutOfSync     Healthy      ← step 2 begins
demo16-prod       Synced        Healthy      ← still waiting

# Step 2 complete — staging is Healthy, step 3 begins:
demo16-dev        Synced        Healthy
demo16-staging    Synced        Healthy      ← step 2 done ✅
demo16-prod       OutOfSync     Healthy      ← step 3 begins

# Step 3 complete — rollout done:
demo16-dev        Synced        Healthy
demo16-staging    Synced        Healthy
demo16-prod       Synced        Healthy      ← step 3 done ✅
```

**Verify v2 is now running on all three — check each environment:**
```bash
kubectl port-forward svc/dev-platform-svc -n demo16-dev 8085:80
# http://localhost:8085 → "Version: v2"

kubectl port-forward svc/staging-platform-svc -n demo16-staging 8086:80
# http://localhost:8086 → "Version: v2"

kubectl port-forward svc/prod-platform-svc -n demo16-prod 8087:80
# http://localhost:8087 → "Version: v2"
```

**Key observation:** Platform v2 reached prod only after dev and staging each
confirmed Healthy. The rollout was sequential — never simultaneous.

---

## Step 6: Rollout Halt — Bad Image Tag

Trigger a rollout with a broken image tag. This simulates a typo in the image
name or a non-existent tag. The rollout must stop at dev — staging and prod
remain on v2, untouched.

**In `gitops-apps-config`:**

Edit `demo-16-progressive-sync/base/deployment.yaml` — use a non-existent tag:
```yaml
# change:
image: nginx:1.25
# to:
image: nginx:this-tag-does-not-exist
```

Also update the ConfigMap to simulate v3 so the change is visible:
Edit `demo-16-progressive-sync/base/configmap.yaml`:
```yaml
      <h2 style="color: #dc2626;">Version: v3 (BAD — rollout will halt)</h2>
```

```bash
git add demo-16-progressive-sync/base/
git commit -m "test: bad image tag to demonstrate progressive sync halt"
git push origin main
```

**Watch — rollout starts at dev, halts when dev goes Degraded:**
```bash
watch -n 2 'argocd app list | grep demo16'
```

**Expected — step 1 fails, steps 2 and 3 are never attempted:**
```text
NAME              SYNC STATUS   HEALTH STATUS
demo16-dev        OutOfSync     Degraded     ← ImagePullBackOff — step 1 fails
demo16-staging    Synced        Healthy      ← PROTECTED — still on v2 ✅
demo16-prod       Synced        Healthy      ← PROTECTED — still on v2 ✅
```

**Verify dev pod is in ImagePullBackOff:**
```bash
kubectl get pods -n demo16-dev
```

**Expected:**
```text
NAME                             READY   STATUS             RESTARTS
dev-platform-app-xxx-new         0/1     ImagePullBackOff   0
dev-platform-app-xxx-old         1/1     Running            0   ← old pod kept
```

**Verify staging and prod are still on v2 (the bad image never reached them):**
```bash
kubectl get deployment staging-platform-app -n demo16-staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# → nginx:1.25  (v2 base image — bad tag never reached staging)

kubectl get deployment prod-platform-app -n demo16-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# → nginx:1.25  (same — prod is fully protected)
```

**Inspect the ApplicationSet status — shows step 1 halted:**
```bash
kubectl describe appset demo16-progressive -n argocd | grep -A 30 "Application Status"
```

**Key observation:** The blast radius was contained to `demo16-dev`. Staging
and prod were never touched. This is Progressive Sync's core value — one bad
commit to a shared base affects only the first environment, not all of them.

---

## Step 7: Fix and Resume the Rollout

Fix the image tag in Git. The ApplicationSet controller detects the new desired
state, re-attempts step 1, and if dev becomes Healthy, automatically advances
to staging and then prod.

**In `gitops-apps-config`:**

Edit `demo-16-progressive-sync/base/deployment.yaml` — restore the valid tag:
```yaml
image: nginx:1.26    # valid tag — v3 proceeds with correct image
```

Update the ConfigMap to complete the v3 story:
```yaml
      <h2 style="color: #16a34a;">Version: v3 (FIXED — rollout resumed)</h2>
```

```bash
git add demo-16-progressive-sync/base/
git commit -m "fix: restore valid image tag — resume v3 rollout"
git push origin main
```

**Watch the rollout resume from step 1 and complete:**
```bash
watch -n 2 'argocd app list | grep demo16'
```

**Expected — full staged rollout completes:**
```text
demo16-dev        OutOfSync     Progressing  ← step 1 re-attempted
demo16-staging    Synced        Healthy
demo16-prod       Synced        Healthy

demo16-dev        Synced        Healthy      ← step 1 complete ✅
demo16-staging    OutOfSync     Progressing  ← step 2 begins
demo16-prod       Synced        Healthy

demo16-staging    Synced        Healthy      ← step 2 complete ✅
demo16-prod       OutOfSync     Progressing  ← step 3 begins

demo16-prod       Synced        Healthy      ← rollout complete ✅
```

**Verify v3 with nginx:1.26 is running on all three:**
```bash
for ns in demo16-dev demo16-staging demo16-prod; do
  echo -n "$ns image: "
  kubectl get deployment -n $ns \
    -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
  echo
done
```

**Expected:**
```text
demo16-dev image: nginx:1.26
demo16-staging image: nginx:1.26
demo16-prod image: nginx:1.26
```

---

## Step 8: `maxUpdate` — One at a Time Within a Step

Add a `canary` overlay to demo-16. With `maxUpdate: 1` on step 1, `dev` and
`canary` are both in step 1 — but only one syncs at a time, in sequence.
This shows how `maxUpdate` controls parallelism within a single step.

**Add the canary overlay in `gitops-apps-config`:**

**`demo-16-progressive-sync/overlays/canary/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: demo16-canary
namePrefix: canary-
commonLabels:
  env: canary
  app: demo16-platform
replicas:
  - name: platform-app
    count: 1
patches:
  - path: env-patch.yaml
```

**`demo-16-progressive-sync/overlays/canary/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-app
spec:
  template:
    metadata:
      labels:
        env: canary
        version: v3
```

```bash
git add demo-16-progressive-sync/overlays/canary/
git commit -m "feat: add canary overlay for maxUpdate demo"
git push origin main
```

**Update the ApplicationSet strategy to include canary in step 1 with `maxUpdate: 1`:**

Edit `demo-16-progressive-sync/appset-progressive.yaml` — update only the strategy block:
```yaml
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        # Step 1: dev and canary — but only one at a time
        - matchExpressions:
            - key: env
              operator: In
              values:
                - dev
                - canary
          maxUpdate: 1      # one Application syncs at a time within this step
                            # ensures canary is verified before dev (or vice versa)
                            # order within the step is determined by the controller

        - matchExpressions:
            - key: env
              operator: In
              values:
                - staging

        - matchExpressions:
            - key: env
              operator: In
              values:
                - prod
```

```bash
git add demo-16-progressive-sync/appset-progressive.yaml
git commit -m "feat: add canary to step 1 with maxUpdate: 1"
git push origin main

kubectl apply -f demo-16-progressive-sync/appset-progressive.yaml
```

**Trigger a change to observe maxUpdate in action:**

Edit `demo-16-progressive-sync/base/configmap.yaml` — add a minor change:
```yaml
      <p>maxUpdate: 1 — canary and dev sync one at a time within step 1.</p>
```

```bash
git add demo-16-progressive-sync/base/configmap.yaml
git commit -m "feat: trigger rollout to demonstrate maxUpdate: 1"
git push origin main
```

**Watch — only one of dev/canary syncs at a time in step 1:**
```bash
watch -n 2 'argocd app list | grep demo16'
```

**Expected:**
```text
demo16-dev        OutOfSync  Progressing  ← syncing (maxUpdate=1, one at a time)
demo16-canary     Synced     Healthy      ← waiting its turn in step 1
demo16-staging    Synced     Healthy      ← step 2, not started
demo16-prod       Synced     Healthy      ← step 3, not started

demo16-dev        Synced     Healthy      ← dev done
demo16-canary     OutOfSync  Progressing  ← canary's turn (still step 1)
demo16-staging    Synced     Healthy
demo16-prod       Synced     Healthy

demo16-dev        Synced     Healthy
demo16-canary     Synced     Healthy      ← step 1 complete
demo16-staging    OutOfSync  Progressing  ← step 2 begins
demo16-prod       Synced     Healthy
```

---

## Verify Final State

```bash
# ApplicationSet with RollingSync strategy
kubectl get appset demo16-progressive -n argocd -o yaml | grep -A 25 "strategy:"

# All four Applications Synced and Healthy
argocd app list | grep demo16

# Application labels (used by step selectors)
kubectl get app -n argocd --show-labels | grep demo16

# Correct replica counts per environment
kubectl get deployments -A | grep demo16

# Current image on all environments
for ns in demo16-dev demo16-canary demo16-staging demo16-prod; do
  echo -n "$ns: "
  kubectl get deployment -n $ns \
    -o jsonpath='{.items[0].spec.template.spec.containers[0].image}' 2>/dev/null \
    || echo "(no deployment)"
  echo
done

# ApplicationSet rollout status
kubectl get appset demo16-progressive -n argocd \
  -o jsonpath='{.status.applicationStatus}' | python3 -m json.tool
```

---

## Cleanup

```bash
# Delete the Demo-16 ApplicationSet (cascades to generated Applications)
kubectl delete appset demo16-progressive -n argocd

# Delete namespaces manually — ArgoCD never deletes namespaces
kubectl delete ns demo16-dev demo16-canary demo16-staging demo16-prod

# Disable progressive syncs on controller (restore default)
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data": {"applicationsetcontroller.enable.progressive.syncs": "false"}}'

kubectl rollout restart deployment \
  -l app.kubernetes.io/component=applicationset-controller \
  -n argocd

# Demo-14 ApplicationSet continues running — it is NOT affected
kubectl get appset demo14-kustomize -n argocd   # still present
argocd app list | grep demo14                   # still running
```

---

## Key Concepts Summary

**Progressive Sync adds a safety gate between environments**
`strategy: RollingSync` makes the ApplicationSet controller own sync timing.
Each step must complete (all Applications reach Healthy) before the next step
begins. A failure stops the rollout — not the same as a revert, but protection
for environments not yet updated.

**The `strategy:` block is the only structural difference from Demo-14**
Generators, template, source path, Kustomize integration — all identical to
the Demo-14 ApplicationSet pattern. Progressive Sync is a safety layer on top,
not a replacement. This is how real teams adopt it: existing ApplicationSets
gain a strategy block.

**Application labels in `template.metadata.labels` are the step selectors**
The `env: "{{.path.basename}}"` label on each generated Application is what
the `rollingSync.steps` `matchExpressions` match against. These are labels on
the ArgoCD Application object — not on the Kubernetes workloads. Without these
labels, no Application matches any step and the rollout processes nothing.

**Every generated Application must match exactly one step**
Applications not matched by any step are excluded from the rollout and must be
synced manually. When adding a new overlay (like `canary`), always update the
strategy steps to include it.

**RollingSync disables individual Application autosync during rollouts**
The controller manages timing. `automated: true` in the Application template
is overridden while a rollout is active. Avoid manually syncing individual
Applications mid-rollout — it can cause unexpected step completion behaviour.

**Rollout resume is automatic — fix in Git, wait for the controller**
A halted rollout resumes when the root cause is corrected in Git and the
controller detects the new desired state. No manual `argocd app sync` needed
for normal fix-and-resume scenarios.

**`maxUpdate` controls parallelism within a step, not step ordering**
`maxUpdate: 1` means only one Application in that step syncs at a time.
It does not affect other steps. Without `maxUpdate`, all Applications in a
step sync simultaneously.

---

## Commands Reference

```bash
# Enable progressive syncs on controller
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data": {"applicationsetcontroller.enable.progressive.syncs": "true"}}'

kubectl rollout restart deployment \
  -l app.kubernetes.io/component=applicationset-controller -n argocd

# Watch rollout progress in real time
watch -n 2 'argocd app list | grep demo16'

# Inspect ApplicationSet strategy
kubectl get appset demo16-progressive -n argocd -o yaml | grep -A 25 "strategy:"

# Inspect ApplicationSet rollout status
kubectl get appset demo16-progressive -n argocd \
  -o jsonpath='{.status.applicationStatus}' | python3 -m json.tool

# Describe ApplicationSet for full event log
kubectl describe appset demo16-progressive -n argocd

# Watch Applications with env labels
kubectl get app -n argocd --show-labels | grep demo16

# Manually sync a stalled Application (use only when rollout is permanently stuck)
argocd app sync demo16-dev

# Controller logs for rollout events
kubectl logs -n argocd \
  -l app.kubernetes.io/component=applicationset-controller \
  --tail=50 | grep -i "progressive\|rolling\|step"

# Port-forward to verify platform version per environment
kubectl port-forward svc/dev-platform-svc     -n demo16-dev     8085:80
kubectl port-forward svc/staging-platform-svc -n demo16-staging 8086:80
kubectl port-forward svc/prod-platform-svc    -n demo16-prod    8087:80
```

---

## Lessons Learned

**1. Enable the controller flag before applying a strategy block**
`strategy: RollingSync` is silently ignored if
`applicationsetcontroller.enable.progressive.syncs` is not `"true"` in
`argocd-cmd-params-cm`. Applications sync simultaneously with no error.
Always enable the flag, restart the controller, and verify the log message
before troubleshooting RollingSync behaviour.

**2. Every generated Application must match a step — add new overlays to the strategy**
When a new overlay is added (e.g. `canary`), the git directory generator picks
it up automatically. But if the step selectors do not include `env=canary`, the
`demo16-canary` Application is excluded from the rollout and will never sync
automatically. Always update the strategy when adding overlays.

**3. Application labels in `template.metadata.labels` are ArgoCD-level labels**
Step selectors match labels on the ArgoCD Application object, not on
Kubernetes workloads inside the cluster. The `env: "{{.path.basename}}"` label
in the template resolves to the overlay folder name. It must be present in
`template.metadata.labels` — labels on the Kustomize-generated Deployment or
Service are not visible to the step selector.

**4. A Degraded Application halts the rollout — by design**
`ImagePullBackOff`, `CrashLoopBackOff`, failed readiness probes — anything
that keeps an Application from reaching Healthy stops the rollout at that step.
Subsequent steps are not attempted. Staging and prod stay on the last good
revision. This is the correct, intended behaviour.

**5. Fix in Git — the controller resumes automatically**
After a halt, fix the root cause in Git and push. The controller detects the
new desired state, re-attempts the failed step, and if successful, advances
through remaining steps automatically. Manual `argocd app sync` is not needed
for normal fix-and-resume scenarios.

**6. RollingSync overrides automated sync on generated Applications**
While a rollout is in progress, the ApplicationSet controller owns sync timing.
Individual Application autosync is suppressed. Manually syncing an Application
mid-rollout can interfere with step tracking. Avoid it unless the rollout is
permanently stuck.

**7. Separate Demo-16 manifests from Demo-14 — never modify Demo-14 resources**
Demo-16 uses its own folder (`demo-16-progressive-sync/`), its own namespaces
(`demo16-*`), and its own ApplicationSet (`demo16-progressive`). Demo-14's
ApplicationSet and manifests are untouched. This is the correct course design —
each demo is independently runnable and does not depend on modifying a previous
demo's live resources.

---

## What's Next

**Demo-17: Secrets Management with ArgoCD — Sealed Secrets**
Production GitOps requires secrets to be stored in Git securely. This demo
covers Sealed Secrets: encrypting Kubernetes Secrets into `SealedSecret` CRDs
that are safe to commit, with ArgoCD automatically decrypting and applying them
during sync.