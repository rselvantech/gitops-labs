# Demo-14: Kustomize with ArgoCD — Template-Free Environment Management

## Overview

Demo-13 showed how ApplicationSets generate many Applications from one YAML.
Each generated Application pointed at its own folder of manifests — `dev/`,
`staging/`, `prod/` — and those folders were nearly identical copies of the
same Deployment, differing only in replica count and namespace. Any structural
change to the Deployment (a new environment variable, a resource limit, a
liveness probe) had to be made in three separate files. Mistakes creep in.
The files drift apart over time. The more environments you have, the worse
this problem becomes.

Kustomize solves this with a **base + overlays** pattern. The base holds the
complete manifest **once**. Each overlay holds only the **delta** for its
environment — a few lines that say "use 3 replicas" or "add this label". ArgoCD
has native Kustomize support: it automatically detects a `kustomization.yaml`
file and runs `kustomize build` before applying the rendered manifests. No
plugins, no extra configuration, no flags needed.

By the end of this demo, three ArgoCD Applications each point at their own
overlay. A single change to the base propagates to all three environments on
the next sync. Then we extend this into a fully working ApplicationSet +
Kustomize setup — the dominant production pattern — where one ApplicationSet
YAML generates all three Applications automatically from overlay directories.

**What you'll learn:**
- What Kustomize is and why it solves the manifest duplication problem
- The base + overlays pattern — one source of truth, many environments
- What `kustomization.yaml` does at the base layer and at the overlay layer
- Every Kustomize field used in this demo: `resources`, `namespace`,
  `namePrefix`, `labels`, `replicas`, `patches`
- Why `namespace` belongs in the overlay `kustomization.yaml` — not in
  manifests and not only in `destination.namespace`
- What a strategic merge patch is, what it can and cannot do, and when
  to use it over the `labels` field or JSON6902 patches
- How ArgoCD auto-detects Kustomize — no flags, no annotations
- How ApplicationSet + Kustomize work together — the full translation chain
  from generator variable to rendered manifest to running pod

**What you'll do:**
- Deploy an nginx web application — a company's internal status page —
  across three environments (dev, staging, prod) using Kustomize
- Build a base manifest with no namespace, no environment values
- Create three overlays — each with its own namespace, replica count, and
  environment label
- Deploy each overlay as a separate ArgoCD Application and verify it
- Prove one base change propagates to all three environments automatically
- Prove ArgoCD self-heal reverts a manual replica change to the overlay value
- Replace the three manual Application CRDs with one ApplicationSet that
  generates them automatically from overlay directories

---

## Prerequisites

- ✅ Completed Demo-13 — ApplicationSets and Git Directory generator understood
- ✅ ArgoCD running on minikube default profile
- ✅ ArgoCD CLI installed and logged in
- ✅ `kustomize` CLI installed (for local build verification)
- ✅ GitHub PAT with `Contents: Read/Write` access to `gitops-apps-config`
  and `argocd-config`

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

**Expected:** Login successful, `argocd: v2.x.x`

### 3. Kustomize CLI installed
```bash
kustomize version
```

**Expected:** `{Version:kustomize/v5.x.x ...}`

**Install if missing:**
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

### 4. Repos registered with ArgoCD
```bash
argocd repo list
```

**Expected:** `gitops-apps-config` and `argocd-config` both showing `Successful`.

---

## Concepts

### The Demo Scenario — Company Status Page

Your company runs an internal status page — a simple nginx web application
that shows a health dashboard. It is deployed across three environments:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    Status Page Deployment                                │
│                                                                          │
│  dev environment          staging environment      prod environment      │
│  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐      │
│  │  nginx       │         │  nginx       │         │  nginx       │      │
│  │  status page │         │  status page │         │  status page │      │
│  │  1 replica   │         │  2 replicas  │         │  3 replicas  │      │
│  │  demo14-dev  │         │ demo14-stg   │         │  demo14-prod │      │
│  └──────────────┘         └──────────────┘         └──────────────┘      │
│                                                                          │
│  Same nginx image. Same Deployment structure. Same Service type.         │
│  Only the replica count, namespace, and environment label differ.        │
└──────────────────────────────────────────────────────────────────────────┘
```

**The problem without Kustomize:**

Without Kustomize, managing three environments means three separate copies of
`deployment.yaml`. Each is nearly identical — the only real differences are
the namespace name and the replica count:

```
demo-13 approach (without Kustomize):
  dev/manifests/deployment.yaml      → namespace: demo13-dev,     replicas: 1
  staging/manifests/deployment.yaml  → namespace: demo13-staging,  replicas: 2
  prod/manifests/deployment.yaml     → namespace: demo13-prod,     replicas: 3
```

Now your team needs to add a liveness probe to the Deployment. You edit
`dev/manifests/deployment.yaml`, then `staging/manifests/deployment.yaml`,
then `prod/manifests/deployment.yaml`. Three edits, three PRs, or one PR that
touches three files. Miss one and the environments diverge. Add more environments
and the problem multiplies.

**The solution with Kustomize:**

```
demo-14 approach (with Kustomize):
  base/deployment.yaml               → no namespace, no replica value — defined once
  overlays/dev/kustomization.yaml    → namespace: demo14-dev,    replicas: 1
  overlays/staging/kustomization.yaml→ namespace: demo14-staging, replicas: 2
  overlays/prod/kustomization.yaml   → namespace: demo14-prod,   replicas: 3
```

The liveness probe change goes into `base/deployment.yaml` once — Kustomize
merges it into all three environments automatically on the next `kustomize build`.

---

### What Kustomize Is

Kustomize is a **template-free** Kubernetes configuration management tool
built into both `kubectl` (`kubectl apply -k`) and ArgoCD natively. Unlike
Helm, it does not use a templating language with `{{ }}` placeholders. It takes
real, valid Kubernetes YAML as input and applies **transformations** on top —
the original files are never modified.

```
Without Kustomize:               With Kustomize:
───────────────────────          ────────────────────────────────────
dev/deployment.yaml              base/deployment.yaml     ← defined once
staging/deployment.yaml          overlays/dev/kustomization.yaml
prod/deployment.yaml             overlays/staging/kustomization.yaml
                                 overlays/prod/kustomization.yaml
```

Kustomize transforms YAML into more YAML. The output is always plain Kubernetes
manifests — no runtime engine, no templating language to learn. ArgoCD runs
`kustomize build <overlay-path>` internally and applies the resulting plain
YAML to the cluster. The presence of `kustomization.yaml` in the source path
is all ArgoCD needs to detect and invoke Kustomize — no annotation or flag needed.

**Kustomize vs Helm — when to choose which:**

| Aspect | Kustomize | Helm |
|---|---|---|
| Config syntax | Plain YAML + patches | Go templates `{{ }}` |
| Base manifests | Valid Kubernetes YAML, deployable directly | Templates, not directly deployable |
| Environment differences | Overlays with patches | `values.yaml` per environment |
| ArgoCD detection | Automatic via `kustomization.yaml` | Automatic via `Chart.yaml` |
| Best for | Kubernetes-native config management, simple env patches | Complex packaging, versioned chart distribution |

---

### The Base + Overlays Pattern

```
gitops-apps-config/demo-14-kustomize/
├── base/                             ← shared across all environments
│   ├── kustomization.yaml            ← lists which files are resources
│   ├── deployment.yaml               ← 1 replica, NO namespace, no env values
│   ├── service.yaml                  ← NO namespace
│   └── configmap.yaml                ← NO namespace, generic content
└── overlays/
    ├── dev/                          ← ONLY what differs from base
    │   ├── kustomization.yaml        ← namespace + namePrefix + replicas + patch ref
    │   └── env-patch.yaml            ← adds env label to pod template
    ├── staging/
    │   ├── kustomization.yaml
    │   └── env-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── env-patch.yaml
```

**The base** contains complete, working Kubernetes YAML with no hardcoded
environment values. It has no namespace field, no environment-specific replica
count, no environment labels. Running `kubectl apply -k base/` directly would
produce a working application — it is not a template, it is deployable YAML.

**Each overlay** references the base via its `kustomization.yaml` and declares
only the differences. Kustomize merges base + overlay at build time to produce
the final manifests. The overlay files contain only the delta — they do not
copy or repeat any base content.

---

### How `kustomization.yaml` Works at Each Layer — With Fields Explained

**Base `kustomization.yaml` — declares resources only:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:            # lists the YAML files that form the base application
  - deployment.yaml   # each file is a standard Kubernetes manifest
  - service.yaml
  - configmap.yaml
```

`resources:` is the only required field in the base. It tells Kustomize which
files to load. No transformations are declared at the base level — transformations
belong in overlays so each environment controls its own customisations.

**Overlay `kustomization.yaml` — references base, declares all customisations:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base       # path to the base directory — 'resources' not deprecated 'bases'
                     # Kustomize loads all base resources as the starting point

namespace: demo14-dev
  # Kustomize namespace transformer — sets metadata.namespace on EVERY resource
  # in this overlay. Base manifests have no namespace; this field supplies it.
  # More reliable than destination.namespace alone (covers all resource types).

namePrefix: dev-
  # Prepends "dev-" to the name of every resource this overlay produces.
  # nginx-app → dev-nginx-app | nginx-service → dev-nginx-service
  # nginx-config → dev-nginx-config
  # Prevents name collisions when the same base is deployed to multiple
  # environments on the same cluster. Also makes debugging easier —
  # "dev-nginx-app" is unambiguous at a glance.

labels:
  - pairs:
      env: dev
    includeSelectors: false
    includeTemplates: true
  # Replaces deprecated 'commonLabels'. Two explicit boolean flags control scope:
  #   includeSelectors: false → does NOT add to spec.selector.matchLabels
  #                             Safe on live Deployments — selector is immutable
  #   includeTemplates: true  → DOES add to spec.template.metadata.labels (pod labels)
  # This gives pods the env label without risking an immutable selector update.

replicas:
  - name: nginx-app  # refers to the BASE resource name (before namePrefix)
    count: 1         # overrides spec.replicas on the named Deployment
                     # Kustomize applies this as a targeted transformer —
                     # only the specified Deployment is affected.

patches:
  - path: env-patch.yaml
  # Strategic merge patch — a partial YAML document that targets a specific
  # resource by kind+name and merges only the specified fields.
  # Explained in detail in the next section.
```

> **`resources` not `bases`:** The `bases:` field was deprecated in Kustomize
> v3.2 and is not supported in Kustomize v4+. Always use `resources:` to
> reference the base directory from an overlay.

**What `kustomize build overlays/dev/` produces — key fields from the output:**

```yaml
# Deployment (merged: base + namespace + namePrefix + labels + replicas + patch)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-nginx-app          # ← namePrefix applied
  namespace: demo14-dev        # ← namespace transformer applied
  labels:
    env: dev                   # ← labels transformer applied (metadata.labels)
spec:
  replicas: 1                  # ← replicas transformer applied
  selector:
    matchLabels:
      app: nginx-app           # ← selector NOT touched (includeSelectors: false)
                               #   safe on live Deployments
  template:
    metadata:
      labels:
        app: nginx-app
        env: dev               # ← labels transformer applied (includeTemplates: true)
---
# Service
metadata:
  name: dev-nginx-service      # ← namePrefix applied
  namespace: demo14-dev        # ← namespace transformer applied
  labels:
    env: dev                   # ← labels transformer applied
---
# ConfigMap
metadata:
  name: dev-nginx-config       # ← namePrefix applied
  namespace: demo14-dev        # ← namespace transformer applied
  labels:
    env: dev                   # ← labels transformer applied
```

The full base content (container image, ports, volume mounts, service ports)
is carried forward unchanged. Only the fields declared in the overlay are
different.

---

### What Is a Strategic Merge Patch?

A **Strategic Merge Patch** is a partial Kubernetes YAML document that describes
only the fields you want to change or add. Kustomize merges it into the full
base resource — fields not mentioned in the patch are carried forward unchanged
from the base. It is called "strategic" because the merge logic is not a simple
key overwrite — it understands Kubernetes resource structure and merges lists
intelligently (e.g. containers are merged by `name`, not replaced entirely).

**The format:** A patch file looks like a normal Kubernetes manifest, but only
contains the fields you are changing. It must identify its target via
`apiVersion`, `kind`, and `metadata.name`:

```yaml
# env-patch.yaml — only the fields to change/add
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app        # identifies which base resource to patch
spec:
  template:
    metadata:
      labels:
        env: dev         # only this field is added — everything else from base
```

**What you CAN do with a strategic merge patch:**

- Add or change any field anywhere in a Kubernetes resource — labels, annotations,
  environment variables, resource limits, liveness/readiness probes, volume mounts,
  init containers, sidecar containers, image tags, ports, args
- Target **any Kubernetes resource type** — Deployment, StatefulSet, Service,
  ConfigMap, Job, CronJob, Ingress, custom resources — not just Deployments
- Target multiple resources with one patch using the `target:` selector in
  `kustomization.yaml` (select by kind, name, label, or namespace)
- Add an entire new container (sidecar injection pattern)
- Merge into a list — adding a new container to `spec.template.spec.containers`
  merges by container `name`, not replaces the whole list

```yaml
# Patch targeting a specific Deployment by name
patches:
  - path: env-patch.yaml    # targets Deployment named nginx-app (from patch metadata.name)

# Patch targeting ALL Deployments in the overlay using target selector
patches:
  - path: sidecar-patch.yaml
    target:
      kind: Deployment      # applies to every Deployment regardless of name
```

**What you CANNOT do with a strategic merge patch (use JSON6902 patch instead):**

- **Delete a field** — SMP merges fields in; it cannot remove an existing field.
  Use `$patch: delete` directive or a JSON6902 patch with `op: remove`
- **Replace an entire list** — by default SMP merges lists by strategic key.
  You can use `$patch: replace` directive, but the syntax is non-obvious and
  fragile. JSON6902 is cleaner for this
- **Target resources without knowing their name** — SMP identifies target by
  `metadata.name` in the patch file. For dynamic targets use the `target:`
  selector in `kustomization.yaml`
- **Not all custom resources support SMP** — SMP works on resources that have
  strategic merge patch annotations in their Go struct definitions (all core
  Kubernetes resources do). Custom resources without those annotations fall back
  to JSON merge behaviour. For arbitrary custom resources, JSON6902 patch is safer

---

### Why a Patch Is Still Needed — `commonLabels`, `labels`, and What Each Does

This section answers three related questions together because they are connected:
why is `commonLabels` deprecated, what does `labels` do differently, and does
a patch become unnecessary when using `labels`?

**`commonLabels` — what it does and why it is deprecated:**

`commonLabels` adds a label to `metadata.labels` on every resource AND to
`spec.selector.matchLabels` AND to `spec.template.metadata.labels` on
Deployments. The selector inclusion is the problem.

Kubernetes treats `spec.selector.matchLabels` on a Deployment as immutable
once the Deployment is created. If you add a new label via `commonLabels`
after the Deployment already exists, Kubernetes rejects the update because it
would require changing the immutable selector. `commonLabels` is therefore
dangerous on any live Deployment. The official Kustomize API marks it explicitly
as deprecated: "Deprecated: Use the Labels field instead, which provides a superset
of the functionality of CommonLabels."

**`labels` — the replacement, with explicit control:**

The `labels` field is the modern replacement. It gives you explicit control
over which fields receive the label via two boolean flags:

```yaml
labels:
  - pairs:
      env: dev
    includeSelectors: true    # also adds to spec.selector.matchLabels
    includeTemplates: true    # also adds to spec.template.metadata.labels
```

`includeSelectors` indicates whether the transformer should include the
`spec/selector` fieldSpec. `includeTemplates` indicates whether the transformer
should include the `spec/template/metadata` fieldSpec.

**The safe production setting is `includeSelectors: false`:**

```yaml
labels:
  - pairs:
      env: dev
    includeSelectors: false   # do NOT touch spec.selector.matchLabels
    includeTemplates: true    # DO add to spec.template.metadata.labels
```

Use the labels transformer with `includeSelectors: false` to safely add
labels without modifying immutable selectors.

This is the correct pattern for existing Deployments. You get labels on all
resource metadata and pod templates, but the Deployment selector — which was
set at creation time from the base manifest — is never touched.

**Do we still need a patch when using `labels`?**

For this demo's specific `env` label, the answer depends on what `labels` with
`includeTemplates: true` covers:

```
labels with includeTemplates: true covers:
  ✅ metadata.labels on all resources
  ✅ spec.template.metadata.labels on Deployments  ← pod template labels

labels with includeSelectors: true additionally covers:
  ✅ spec.selector.matchLabels  ← dangerous on existing Deployments
```

So with `labels` + `includeTemplates: true` + `includeSelectors: false`,
the `env` label IS applied to pod templates. The patch becomes unnecessary
**for this specific label in this specific demo**.

**But patches are still needed for everything `labels` cannot do:**

The `labels` field in `kustomization.yaml` only adds labels. It cannot:

| Requirement | `labels` field | Patch |
|---|---|---|
| Add env label to pod template | ✅ (`includeTemplates: true`) | ✅ |
| Add env-specific image tag | ✗ | ✅ |
| Set resource limits per environment | ✗ | ✅ |
| Add a liveness probe with env-specific thresholds | ✗ | ✅ |
| Inject a sidecar container in prod only | ✗ | ✅ |
| Add environment variables per environment | ✗ | ✅ |
| Change service port in staging only | ✗ | ✅ |

The `labels` field is a single-purpose transformer. Patches are the general-purpose
mechanism for any per-environment structural change that goes beyond labels.

**In this demo — using `labels` instead of `commonLabels`:**

Replace `commonLabels` with `labels` in all three overlays:

```yaml
# Replace this (deprecated):
commonLabels:
  env: dev

# With this (correct):
labels:
  - pairs:
      env: dev
    includeSelectors: false   # safe — does not touch immutable selector
    includeTemplates: true    # adds env label to pod template
```

With this change, the `env-patch.yaml` is technically redundant for the `env`
label — `labels` with `includeTemplates: true` already adds it to the pod
template. However, the patch file is kept in this demo to show the strategic
merge patch pattern, which you will need the moment you have any per-environment
change beyond a label — image tags, resource limits, probes, sidecars.

---
### Why `namespace` Belongs in the Overlay `kustomization.yaml` — Not Just `destination.namespace`

This is the most important Kustomize + ArgoCD pattern to understand correctly.
There are three places where a namespace could be set, and they are not equivalent.

**Option 1: Hardcode in each manifest `metadata.namespace`**
```yaml
# deployment.yaml
metadata:
  name: nginx-app
  namespace: demo14-dev   ← hardcoded
```
This is what Demo-13 manifests did. The problem: as established in Demo-13,
`metadata.namespace` in a manifest always overrides `destination.namespace` in
the Application CRD. If the manifest says `demo14-dev` and the ApplicationSet
sets `destination.namespace` to `demo14-prod`, the resource lands in `demo14-dev`
— the wrong namespace. It also means the manifest is not reusable across environments.

**Option 2: Use only `destination.namespace` in the Application CRD**
```yaml
# Application CRD
destination:
  namespace: demo14-dev
```
As confirmed by the official ArgoCD docs: `spec.destination.namespace` only adds a namespace when it's missing from the manifests generated by Kustomize. It also uses kubectl to set the namespace, which sometimes misses namespace fields in certain resources (for example, custom resources).

This means `destination.namespace` is a fallback — it only works if the manifest
has no namespace field at all. It also fails silently for ClusterRoleBindings,
custom resources, and other resources that need namespace set in specific nested
fields. It is insufficient for reliable multi-environment Kustomize deployments.

**Option 3 (correct): Use `namespace:` in the overlay `kustomization.yaml`**
```yaml
# overlays/dev/kustomization.yaml
namespace: demo14-dev
```
Kustomize handles changing the namespace in all the right places, even custom places via transformerConfigs, all by setting `namespace: <name>` in the kustomization.yaml.

This is the correct pattern. Kustomize's `namespace:` transformer applies the
namespace to all resources in the overlay — `metadata.namespace` on every
resource, `subjects[*].namespace` in ClusterRoleBindings, and any other
namespace reference Kustomize is aware of. It is reliable, complete, and leaves
base manifests fully namespace-agnostic:

```
base/deployment.yaml        → no namespace field (portable)
overlays/dev/kustomization  → namespace: demo14-dev
kustomize build overlays/dev → Deployment.metadata.namespace = demo14-dev  ✅
ArgoCD destination.namespace = demo14-dev  (consistent, used as fallback only)
```

**Summary — three options compared:**

| Approach | Works reliably | Manifests reusable | Correct for ArgoCD + Kustomize |
|---|---|---|---|
| Hardcode in manifest | ✗ (breaks ApplicationSet destination) | ✗ | ✗ |
| Only `destination.namespace` | Partial (misses some resource types) | ✓ | ✗ |
| `namespace:` in overlay kustomization.yaml | ✓ (all resource types) | ✓ | ✓ |

---

### How ArgoCD Auto-Detects Kustomize

ArgoCD's Repository Server scans the source path for a `kustomization.yaml`
file. If found, it runs `kustomize build <path>` internally and applies the
resulting plain YAML manifests to the cluster. No annotation, no flag, no
special Application CRD field is needed:

```
Application CRD source.path: demo-14-kustomize/overlays/prod
                        │
                        ▼
  Repo Server: "kustomization.yaml found in this path"
                        │
                        ▼
  Repo Server runs internally: kustomize build overlays/prod
                        │
                        ▼
  Output — plain Kubernetes YAML:
    Deployment: prod-nginx-app, namespace: demo14-prod, replicas: 3, env: prod
    Service:    prod-nginx-service, namespace: demo14-prod
    ConfigMap:  prod-nginx-config, namespace: demo14-prod
                        │
                        ▼
  App Controller applies rendered manifests to cluster
```

This is what "native Kustomize support" means in ArgoCD — the tool is built in,
not a plugin. The same `kustomize` binary version that ships with ArgoCD is used.

---

### ApplicationSet + Kustomize — The Production Pattern

This is the combination that solves both the manifest duplication problem (Kustomize)
and the Application CRD duplication problem (ApplicationSet) simultaneously.
Understanding how each layer connects is essential before Step 8.

**The problem chain this pattern solves:**

```
Without Kustomize alone:
  → 3 near-identical deployment.yaml files (one per environment)
  → Any structural change = edit 3 files

Without ApplicationSet alone (Steps 4-7):
  → 3 separate Application CRDs (dev-app.yaml, staging-app.yaml, prod-app.yaml)
  → New environment = write a new Application YAML

With both together (Step 8):
  → 1 base manifest (defined once)
  → 3 overlay kustomization.yaml files (env-specific deltas only)
  → 1 ApplicationSet YAML (discovers overlays, generates Applications automatically)
  → New environment = create a new overlay directory. Done. Zero other changes.
```

**The full translation chain — from ApplicationSet to running pod:**

```
Step A: ApplicationSet Git Directory generator scans:
  gitops-apps-config/demo-14-kustomize/overlays/*

  Discovers three directories:
    [1] demo-14-kustomize/overlays/dev
    [2] demo-14-kustomize/overlays/staging
    [3] demo-14-kustomize/overlays/prod

  For each discovered directory, generator exposes variables:
    {{.path.path}}     = "demo-14-kustomize/overlays/dev"
    {{.path.basename}} = "dev"

────────────────────────────────────────────────────────────

Step B: ApplicationSet template runs once per discovered directory.
  For overlays/dev:

  template substitution:
    name:      "demo14-{{.path.basename}}"  → "demo14-dev"
    labels:    env: "{{.path.basename}}"    → env: dev
    source.path: "{{.path.path}}"           → "demo-14-kustomize/overlays/dev"
    destination.namespace: "demo14-{{.path.basename}}" → "demo14-dev"

  Result — Application CRD created in the cluster:
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: demo14-dev
      labels:
        env: dev
    spec:
      source:
        path: demo-14-kustomize/overlays/dev   ← points at Kustomize overlay
      destination:
        namespace: demo14-dev

────────────────────────────────────────────────────────────

Step C: ArgoCD Repository Server processes the Application.
  source.path = "demo-14-kustomize/overlays/dev"
  → finds kustomization.yaml in that path
  → runs: kustomize build demo-14-kustomize/overlays/dev/

  Kustomize loads:
    base/deployment.yaml  + overlays/dev/kustomization.yaml
    base/service.yaml     + overlays/dev/env-patch.yaml
    base/configmap.yaml

  Applies transformers in order:
    1. resources     → loads base manifests
    2. namespace     → sets metadata.namespace: demo14-dev on all resources
    3. namePrefix    → dev-nginx-app, dev-nginx-service, dev-nginx-config
    4. labels        → adds env: dev to metadata.labels and pod template labels
                       (includeSelectors: false — selector not touched)
    5. replicas      → sets spec.replicas: 1 on dev-nginx-app Deployment
    6. patches       → merges env-patch.yaml into Deployment pod template

  Output — rendered plain Kubernetes YAML:
    Deployment: dev-nginx-app (replicas:1, namespace:demo14-dev, env:dev label)
    Service:    dev-nginx-service (namespace:demo14-dev)
    ConfigMap:  dev-nginx-config (namespace:demo14-dev, base index.html content)

────────────────────────────────────────────────────────────

Step D: ArgoCD Application Controller applies rendered manifests to cluster.
  destination.namespace: demo14-dev
  destination.server: https://kubernetes.default.svc

  kubectl apply (equivalent):
    Deployment dev-nginx-app  → demo14-dev namespace
    Service dev-nginx-service → demo14-dev namespace
    ConfigMap dev-nginx-config → demo14-dev namespace

  CreateNamespace=true → creates demo14-dev namespace if it does not exist

────────────────────────────────────────────────────────────

Step E: Same chain runs for overlays/staging and overlays/prod.
  staging → demo14-staging Application → kustomize build staging → replicas:2, staging- prefix
  prod    → demo14-prod Application    → kustomize build prod    → replicas:3, prod- prefix

Result: 3 Applications, 3 namespaces, 3 Deployments —
        all from 1 ApplicationSet YAML + 1 base + 3 overlay kustomization files
```

**Why adding a new environment requires zero YAML changes:**

If you create `overlays/hotfix/kustomization.yaml` in Git, the Git Directory
generator discovers it on the next ArgoCD refresh, generates a `demo14-hotfix`
Application, ArgoCD runs `kustomize build overlays/hotfix/`, and the environment
is live. The ApplicationSet YAML is never touched. The base is never touched.
Only a new overlay directory is added.

---

## Folder Structure

```
14-kustomize/
└── src/
    ├── gitops-apps-config/
    │   └── demo-14-kustomize/
    │       ├── base/
    │       │   ├── kustomization.yaml  ← lists resources, no transformations
    │       │   ├── deployment.yaml     ← no namespace, replicas:1 (base default)
    │       │   ├── service.yaml        ← no namespace
    │       │   └── configmap.yaml      ← no namespace, base page content
    │       └── overlays/
    │           ├── dev/
    │           │   ├── kustomization.yaml  ← namespace:demo14-dev, prefix:dev-, replicas:1
    │           │   └── env-patch.yaml      ← adds env:dev to pod template
    │           ├── staging/
    │           │   ├── kustomization.yaml  ← namespace:demo14-staging, prefix:staging-, replicas:2
    │           │   └── env-patch.yaml      ← adds env:staging to pod template
    │           └── prod/
    │               ├── kustomization.yaml  ← namespace:demo14-prod, prefix:prod-, replicas:3
    │               └── env-patch.yaml      ← adds env:prod to pod template
    └── argocd-config/
        └── demo-14-kustomize/
            ├── dev-app.yaml          ← Step 4: manual Application for dev overlay
            ├── staging-app.yaml      ← Step 4: manual Application for staging overlay
            ├── prod-app.yaml         ← Step 4: manual Application for prod overlay
            └── appset-kustomize.yaml ← Step 8: ApplicationSet replaces all three
```

---

## Step 1: Initialise `gitops-apps-config` Local Repo

```bash
cd argo-cd-basics-to-prod/14-kustomize
mkdir -p src/gitops-apps-config && cd src/gitops-apps-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

---

## Step 2: Create the Base Manifests

The base contains complete, environment-neutral Kubernetes manifests. No
`namespace:` field appears anywhere in these files — that is supplied by each
overlay's `kustomization.yaml`. The base Deployment uses `replicas: 1` as a
sensible starting point — each overlay overrides this value independently.
```bash
mkdir -p demo-14-kustomize/base
touch demo-14-kustomize/base/kustomization.yaml
touch demo-14-kustomize/base/deployment.yaml
touch demo-14-kustomize/base/service.yaml
touch demo-14-kustomize/base/configmap.yaml
```

**`demo-14-kustomize/base/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

**`demo-14-kustomize/base/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  # no namespace — set by each overlay kustomization.yaml
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
  # no namespace — set by each overlay kustomization.yaml
spec:
  selector:
    app: nginx-app
  ports:
    - port: 80
      targetPort: 80
```

**`demo-14-kustomize/base/configmap.yaml`:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  # no namespace — set by each overlay kustomization.yaml
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body style="font-family: sans-serif; padding: 2rem;">
      <h1>Company Status Page</h1>
      <p>Status: <strong style="color: green;">Operational</strong></p>
      <p>This page is served from the base ConfigMap.
         The environment is determined by the overlay.</p>
    </body>
    </html>
```

**Verify the base builds and contains no namespace field:**
```bash
kustomize build demo-14-kustomize/base/ | grep -E "^  name:|replicas:|namespace:"
```

**Expected:**
```text
  name: nginx-app
  replicas: 1
  name: nginx-config
  name: nginx-service
  name: nginx-app
```

No `namespace:` lines — confirming the base is fully namespace-agnostic and
portable. Any overlay can deploy it to any namespace.

---

## Step 3: Create the Overlays

Each overlay declares only the delta from the base. The `namespace:` field in
`kustomization.yaml` is the Kustomize namespace transformer — it applies the
namespace to all resources in this overlay. No manifest file needs a namespace field.

### dev overlay — 1 replica, namespace demo14-dev

```bash
mkdir -p demo-14-kustomize/overlays/dev
touch demo-14-kustomize/overlays/dev/kustomization.yaml
touch demo-14-kustomize/overlays/dev/env-patch.yaml
```

**`demo-14-kustomize/overlays/dev/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base          # load base manifests — use 'resources', not deprecated 'bases'

namespace: demo14-dev   # Kustomize sets metadata.namespace: demo14-dev on all resources

namePrefix: dev-        # nginx-app → dev-nginx-app (all resources prefixed)

labels:
  - pairs:
      env: dev
    includeSelectors: false   # do NOT touch spec.selector.matchLabels — immutable on live Deployments
    includeTemplates: true    # add env label to spec.template.metadata.labels (pod labels)
    # 'labels' replaces deprecated 'commonLabels'. Explicit control over what gets labelled.

replicas:
  - name: nginx-app     # base name — before namePrefix is applied
    count: 1

patches:
  - path: env-patch.yaml
```

**`demo-14-kustomize/overlays/dev/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app       # base name — Kustomize resolves namePrefix after patching
spec:
  template:
    metadata:
      labels:
        env: dev        # targets pod template labels only — not the selector
```

### staging overlay — 2 replicas, namespace demo14-staging

```bash
mkdir -p demo-14-kustomize/overlays/staging
touch demo-14-kustomize/overlays/staging/kustomization.yaml
touch demo-14-kustomize/overlays/staging/env-patch.yaml
```

**`demo-14-kustomize/overlays/staging/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: demo14-staging

namePrefix: staging-

labels:
  - pairs:
      env: staging
    includeSelectors: false
    includeTemplates: true

replicas:
  - name: nginx-app
    count: 2

patches:
  - path: env-patch.yaml
```

**`demo-14-kustomize/overlays/staging/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  template:
    metadata:
      labels:
        env: staging
```

### prod overlay — 3 replicas, namespace demo14-prod

```bash
mkdir -p demo-14-kustomize/overlays/prod
touch demo-14-kustomize/overlays/prod/kustomization.yaml
touch demo-14-kustomize/overlays/prod/env-patch.yaml
```

**`demo-14-kustomize/overlays/prod/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: demo14-prod

namePrefix: prod-

labels:
  - pairs:
      env: prod
    includeSelectors: false
    includeTemplates: true

replicas:
  - name: nginx-app
    count: 3

patches:
  - path: env-patch.yaml
```

**`demo-14-kustomize/overlays/prod/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  template:
    metadata:
      labels:
        env: prod
```

**Verify each overlay builds and produces the correct output:**
```bash
for env in dev staging prod; do
  echo "=== $env ==="
  kustomize build demo-14-kustomize/overlays/$env/ \
    | grep -E "^  name:|namespace:|replicas:|env:"
done
```

**Expected:**
```text
=== dev ===
  name: dev-nginx-app
  namespace: demo14-dev
  replicas: 1
  env: dev
=== staging ===
  name: staging-nginx-app
  namespace: demo14-staging
  replicas: 2
  env: staging
=== prod ===
  name: prod-nginx-app
  namespace: demo14-prod
  replicas: 3
  env: prod
```

All three overlays produce distinct namespaces, names, replica counts, and
environment labels — from one base + three small overlay files.

**Push all manifests:**
```bash
git add demo-14-kustomize/
git commit -m "feat: add demo-14 kustomize base and dev/staging/prod overlays"
git push origin main
```

---

## Step 4: Create ArgoCD Applications — One Per Overlay

Each Application CRD points at one overlay path in `gitops-apps-config`. When
ArgoCD syncs the Application, its Repository Server scans `source.path`, finds
`kustomization.yaml`, runs `kustomize build` internally, and applies the rendered
plain YAML to the cluster. No flags or annotations are needed to enable Kustomize
— ArgoCD detects it automatically from the presence of `kustomization.yaml`.

The `destination.namespace` is set to match the `namespace:` field in each
overlay's `kustomization.yaml`. Since the overlay `namespace:` transformer
already sets the namespace on all resources, `destination.namespace` acts as a
consistent fallback and namespace creation target.

```bash
cd argo-cd-basics-to-prod/14-kustomize
mkdir -p src/argocd-config && cd src/argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase

mkdir -p demo-14-kustomize
touch demo-14-kustomize/dev-app.yaml
touch demo-14-kustomize/staging-app.yaml
touch demo-14-kustomize/prod-app.yaml
```

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
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    path: demo-14-kustomize/overlays/dev
    # ArgoCD finds kustomization.yaml here → runs kustomize build automatically
    # No 'kustomize:' field needed — auto-detected from kustomization.yaml presence
  destination:
    server: https://kubernetes.default.svc
    namespace: demo14-dev   # matches namespace: in overlays/dev/kustomization.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
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
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    path: demo-14-kustomize/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: demo14-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
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
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    path: demo-14-kustomize/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: demo14-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Push and apply all three Applications:**
```bash
git add demo-14-kustomize/
git commit -m "feat: add demo-14 Application CRDs for dev, staging, prod overlays"
git push origin main

kubectl apply -f demo-14-kustomize/dev-app.yaml
kubectl apply -f demo-14-kustomize/staging-app.yaml
kubectl apply -f demo-14-kustomize/prod-app.yaml
```

**Verify all three Applications synced:**
```bash
argocd app list
```

**Expected:**
```text
NAME             CLUSTER     NAMESPACE       SYNC STATUS   HEALTH STATUS
demo14-dev       in-cluster  demo14-dev      Synced        Healthy
demo14-staging   in-cluster  demo14-staging  Synced        Healthy
demo14-prod      in-cluster  demo14-prod     Synced        Healthy
```

---

## Step 5: Verify Kustomize Was Applied Correctly

With all three Applications synced, verify that Kustomize's transformations
were applied as expected — correct resource names (namePrefix), correct replica
counts, correct namespaces, and correct environment labels.

**Verify namePrefix and replica counts:**
```bash
kubectl get deployments -A | grep demo14
```

**Expected:**
```text
NAMESPACE        NAME               READY   UP-TO-DATE   AVAILABLE
demo14-dev       dev-nginx-app      1/1     1            1
demo14-staging   staging-nginx-app  2/2     2            2
demo14-prod      prod-nginx-app     3/3     3            3
```

**Verify namespace was set by overlay `kustomization.yaml`, not by manifest:**
```bash
kubectl get deployment dev-nginx-app -n demo14-dev \
  -o jsonpath='{.metadata.namespace}'
# → demo14-dev
```

**Verify env labels were applied by Kustomize to the pods:**
```bash
kubectl get pods -n demo14-dev -L env
kubectl get pods -n demo14-staging -L env
kubectl get pods -n demo14-prod -L env
```

**Expected:** Each pod shows `env=dev`, `env=staging`, `env=prod` in the ENV column.

**Access the status page from the dev environment:**
```bash
kubectl port-forward svc/dev-nginx-service -n demo14-dev 8081:80
```

Open `http://localhost:8081` — the company status page from the base ConfigMap,
served from the `demo14-dev` namespace by the `dev-nginx-app` deployment.

---

## Step 6: Prove One Base Change Propagates to All Three Environments

This step demonstrates the core Kustomize value: a single change to the base
automatically propagates to all environments on the next ArgoCD sync cycle —
without touching any overlay file.

Update the base `configmap.yaml` to add a maintenance notice to the status page:

```bash
cd gitops-labs/argo-cd-basics-to-prod/14-kustomize/src/gitops-apps-config
```

Edit `demo-14-kustomize/base/configmap.yaml` — update the `index.html`:
```yaml
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body style="font-family: sans-serif; padding: 2rem;">
      <h1>Company Status Page</h1>
      <p>Status: <strong style="color: green;">Operational</strong></p>
      <p>This page is served from the base ConfigMap.
         The environment is determined by the overlay.</p>
      <p><em>Scheduled maintenance: Sunday 02:00-04:00 UTC.</em></p>
    </body>
    </html>
```

**Push ConfigMap changes:**
```bash
git add demo-14-kustomize/base/configmap.yaml
git commit -m "feat: add maintenance notice to base status page"
git push origin main
```

**Watch all three Applications detect the base change simultaneously:**
```bash
watch -n 3 'argocd app list'
```

**Expected — all three go OutOfSync then Synced automatically:**
```text
NAME             SYNC STATUS   HEALTH STATUS
demo14-dev       OutOfSync     Healthy      ← all three detect the base change
demo14-staging   OutOfSync     Healthy
demo14-prod      OutOfSync     Healthy

# Seconds later — automated sync kicks in:
demo14-dev       Synced        Healthy
demo14-staging   Synced        Healthy
demo14-prod      Synced        Healthy
```

**Verify the maintenance notice appears on all three environments:**
```bash
kubectl port-forward svc/prod-nginx-service -n demo14-prod 8083:80
```

Open `http://localhost:8083` — the maintenance notice now appears in prod, from
a single base commit. Neither the overlay files nor the Application CRDs were
touched.

---

## Step 7: Prove Self-Heal Reverts a Manual Replica Change

This step demonstrates ArgoCD's `selfHeal: true` enforcement against manual
`kubectl` changes that deviate from the Kustomize overlay-declared state.

Manually scale prod to 1 replica (the overlay declares 3):

```bash
kubectl scale deployment prod-nginx-app -n demo14-prod --replicas=1
```

**Watch ArgoCD detect the drift and self-heal back to the overlay value:**
```bash
kubectl get deployment prod-nginx-app -n demo14-prod -w
```

**Expected:**
```text
NAME            READY   UP-TO-DATE   AVAILABLE
prod-nginx-app  3/3     3            3    ← initial state from overlay
prod-nginx-app  1/1     1            1    ← manual scale applied
prod-nginx-app  3/3     3            3    ← ArgoCD self-healed to overlay value
```

The source of truth is the Kustomize overlay (`replicas: 3` in
`overlays/prod/kustomization.yaml`). Any deviation — whether from a manual
`kubectl scale` or a pod crash — is corrected by ArgoCD within the reconciliation
interval.

---

## Step 8: ApplicationSet + Kustomize — Replace Three Applications with One YAML

Steps 4-7 used three manually written Application CRDs, one per overlay. This
step replaces all three with a single ApplicationSet that automatically discovers
overlay directories and generates the Applications. This is the production pattern.

Refer to the "ApplicationSet + Kustomize — The Production Pattern" section in
Concepts for the full translation chain explanation. This step is the working
implementation of that pattern.

**The problem Steps 4-7 leave unsolved:**

With three manual Application CRDs, adding a new environment (e.g. `hotfix`)
requires writing a new `hotfix-app.yaml` in `argocd-config`. That is still
manual YAML maintenance — just at the Application level rather than the manifest
level. The ApplicationSet solves this final layer of duplication.

**How the ApplicationSet discovers overlays and connects to Kustomize:**

The Git Directory generator scans `demo-14-kustomize/overlays/*` in
`gitops-apps-config`. For each directory it finds, it generates one Application
whose `source.path` points at that overlay directory. ArgoCD then finds
`kustomization.yaml` in that path and runs Kustomize automatically — the same
way it did in Steps 4-7, but without any manually written Application YAML.

**Delete the three manual Applications first:**
```bash
kubectl delete app demo14-dev demo14-staging demo14-prod -n argocd
kubectl delete ns demo14-dev demo14-staging demo14-prod

cd argo-cd-basics-to-prod/14-kustomize/src/argocd-config

touch demo-14-kustomize/appset-kustomize.yaml
```

**`demo-14-kustomize/appset-kustomize.yaml` in `argocd-config`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo14-kustomize
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
          - path: "demo-14-kustomize/overlays/*"
          # Git Directory generator scans this pattern.
          # Discovers: overlays/dev, overlays/staging, overlays/prod
          # Each discovered directory becomes one Application.
          # Variables exposed per directory:
          #   {{.path.path}}     = "demo-14-kustomize/overlays/dev"
          #   {{.path.basename}} = "dev"
  template:
    metadata:
      name: "demo14-{{.path.basename}}"
      # → demo14-dev, demo14-staging, demo14-prod
      labels:
        env: "{{.path.basename}}"
        # Label on the ArgoCD Application object — not on Kubernetes workloads.
        # Used by monitoring, policy tools, and Demo-16 Progressive Sync selectors.
    spec:
      project: default
      source:
        repoURL: https://github.com/rselvantech/gitops-apps-config.git
        targetRevision: main
        path: "{{.path.path}}"
        # → demo-14-kustomize/overlays/dev     (for dev)
        # → demo-14-kustomize/overlays/staging  (for staging)
        # → demo-14-kustomize/overlays/prod     (for prod)
        #
        # ArgoCD Repository Server finds kustomization.yaml at this path
        # and runs kustomize build automatically — same as Steps 4-7,
        # but driven by the ApplicationSet generator instead of a manual Application YAML.
      destination:
        server: https://kubernetes.default.svc
        namespace: "demo14-{{.path.basename}}"
        # → demo14-dev, demo14-staging, demo14-prod
        # Consistent with namespace: field in each overlay kustomization.yaml.
        # Kustomize sets the namespace on resources; destination.namespace
        # is used for CreateNamespace=true and as a fallback.
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**Push and apply:**
```bash
git add demo-14-kustomize/appset-kustomize.yaml
git commit -m "feat: add demo-14 ApplicationSet with Kustomize overlays"
git push origin main

kubectl apply -f demo-14-kustomize/appset-kustomize.yaml
```

**Watch three Applications appear — generated from overlay directories:**
```bash
watch -n 2 'argocd app list'
```

**Expected:**
```text
NAME             CLUSTER     NAMESPACE       SYNC STATUS   HEALTH STATUS
demo14-dev       in-cluster  demo14-dev      Synced        Healthy
demo14-staging   in-cluster  demo14-staging  Synced        Healthy
demo14-prod      in-cluster  demo14-prod     Synced        Healthy
```

**Verify Kustomize was applied correctly through the ApplicationSet:**
```bash
kubectl get deployments -A | grep demo14
```

**Expected — same result as Steps 4-7, but now fully automated:**
```text
demo14-dev       dev-nginx-app      1/1     1
demo14-staging   staging-nginx-app  2/2     2
demo14-prod      prod-nginx-app     3/3     3
```


**Prove a new environment requires only a new overlay directory:**
```bash
cd argo-cd-basics-to-prod/14-kustomize/src/gitops-apps-config

# Create hotfix environment files in "gitops-apps-config" repo
mkdir -p demo-14-kustomize/overlays/hotfix
touch demo-14-kustomize/overlays/hotfix/kustomization.yaml
touch demo-14-kustomize/overlays/hotfix/env-patch.yaml
```

**`demo-14-kustomize/overlays/hotfix/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: demo14-hotfix
namePrefix: hotfix-
labels:
  - pairs:
      env: hotfix
    includeSelectors: false
    includeTemplates: true
replicas:
  - name: nginx-app
    count: 1
patches:
  - path: env-patch.yaml
```

**`demo-14-kustomize/overlays/hotfix/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  template:
    metadata:
      labels:
        env: hotfix
```

**Push new hotfix environement:**

```bash
git add demo-14-kustomize/overlays/hotfix/
git commit -m "feat: add hotfix overlay"
git push origin main
```

**Watch a fourth Application appear automatically:**
```bash
watch -n 3 'argocd app list | grep demo14'
```

**Expected:**
```text
demo14-dev       Synced   Healthy
demo14-staging   Synced   Healthy
demo14-prod      Synced   Healthy
demo14-hotfix    Synced   Healthy   ← appeared automatically, zero ApplicationSet YAML change
```

---

## Verify Final State

```bash
# ApplicationSet managing all Applications
kubectl get appset demo14-kustomize -n argocd

# All Applications synced
argocd app list | grep demo14

# Correct deployments: namePrefix, replica counts
kubectl get deployments -A | grep demo14

# env labels applied by Kustomize to pods & Correct replica counts per overlay
kubectl get pods -n demo14-dev -L env
kubectl get pods -n demo14-staging -L env
kubectl get pods -n demo14-prod -L env

```

---

## Cleanup

```bash
# Delete the ApplicationSet (cascades to generated Applications and resources)
kubectl delete appset demo14-kustomize -n argocd

# Delete namespaces manually — ArgoCD never deletes namespaces automatically
# (ArgoCD created namespaces via CreateNamespace=true but does not own them)
kubectl delete ns demo14-dev demo14-staging demo14-prod demo14-hotfix
```

---

## Key Concepts Summary

**Kustomize is template-free — base manifests are real, deployable Kubernetes YAML**
No placeholders, no templating syntax to learn. Overlays declare only the delta.
ArgoCD runs `kustomize build` automatically when `kustomization.yaml` is present.

**`namespace:` in overlay `kustomization.yaml` is the correct namespace pattern**
It applies to all resources reliably, including ClusterRoleBinding subjects and
custom resources that `destination.namespace` misses. Base manifests carry no
namespace field. Each overlay sets its own namespace in one place.

**`resources:` not `bases:` for referencing the base directory**
`bases:` was deprecated in Kustomize v3.2 and removed in v4+. Always use
`resources: - ../../base` in overlay `kustomization.yaml`.

**Patch target names refer to base names — before `namePrefix` is applied**
A strategic merge patch targets `name: nginx-app` (the base name). Kustomize
applies patches first, then applies `namePrefix`. Targeting `name: dev-nginx-app`
in a patch file will not match and the patch silently does nothing.

**`labels` with `includeSelectors: false` replaces deprecated `commonLabels`**
`commonLabels` is deprecated — it adds labels to immutable Deployment selectors,
causing update failures on live clusters. Use `labels` with `includeSelectors: false`
and `includeTemplates: true` to safely add labels to resource metadata and pod
templates without touching the selector.

**Strategic merge patch targets any Kubernetes resource — not just Deployments**
A patch is a partial YAML that merges into any base resource by `kind` and `name`.
It handles everything the `labels` transformer cannot: image tags, resource limits,
environment variables, liveness probes, sidecar containers, service ports.
Use `target:` in the patch entry to apply one patch to multiple resources by kind.

**ApplicationSet + Kustomize eliminates both layers of YAML duplication**
Kustomize eliminates manifest duplication (one base, many overlays). ApplicationSet
eliminates Application CRD duplication (one ApplicationSet, many generated Apps).
Together: one base + one overlay per environment + one ApplicationSet = zero manual
YAML maintenance when adding new environments.

---

## Commands Reference

```bash
# Local Kustomize verification (run before every push)
kustomize build 14-kustomize/src/gitops-apps-config/demo-14-kustomize/base/
kustomize build 14-kustomize/src/gitops-apps-config/demo-14-kustomize/overlays/dev/
kustomize build 14-kustomize/src/gitops-apps-config/demo-14-kustomize/overlays/prod/

# Apply with kubectl directly (bypasses ArgoCD — for local testing only)
kubectl apply -k demo-14-kustomize/overlays/dev/

# ApplicationSet
kubectl get appset demo14-kustomize -n argocd
kubectl describe appset demo14-kustomize -n argocd

# Applications
argocd app list
argocd app get demo14-prod
argocd app diff demo14-prod    # shows diff between desired and live state

# Verify rendered output on cluster
kubectl get deployment prod-nginx-app -n demo14-prod -o yaml
kubectl get pods -A -L env | grep demo14

# Access status page per environment
kubectl port-forward svc/dev-nginx-service     -n demo14-dev     8081:80
kubectl port-forward svc/staging-nginx-service -n demo14-staging 8082:80
kubectl port-forward svc/prod-nginx-service    -n demo14-prod    8083:80
```

---

## Lessons Learned

**1. Use `resources:` not `bases:` in overlay `kustomization.yaml`**
`bases:` was deprecated in Kustomize v3.2 and removed in v4+. ArgoCD's bundled
Kustomize version will warn or fail on `bases:`. Always use `resources: - ../../base`.

**2. `namespace:` in overlay `kustomization.yaml` is more reliable than `destination.namespace` alone**
`destination.namespace` only fills in namespace where it is missing, using kubectl —
which misses certain resource types (ClusterRoleBinding subjects, custom resources).
Kustomize's `namespace:` transformer handles all resource types correctly. Set it
in the overlay and leave base manifests namespace-free.

**3. Never hardcode `namespace:` in base manifests**
Hardcoded namespace in a base manifest overrides both Kustomize's `namespace:`
transformer and ArgoCD's `destination.namespace`. The resource lands in the wrong
namespace, which is especially dangerous with ApplicationSets where destination
namespace is set dynamically by a generator variable.

**4. Patch target names are base names — before `namePrefix`**
If your overlay sets `namePrefix: prod-`, your `env-patch.yaml` targets
`name: nginx-app` (the base name), not `name: prod-nginx-app`. Kustomize applies
patches before renaming. A wrong name in the patch silently produces no effect.

**5. `commonLabels` is deprecated — always use `labels` with explicit flags**
`commonLabels` adds labels to `spec.selector.matchLabels` on Deployments, which
Kubernetes treats as immutable once the Deployment is created. Attempting to add
a new label via `commonLabels` after Deployment creation fails with an immutable
field error. The official Kustomize API marks `commonLabels` as deprecated in
favour of `labels`. Always use `labels` with `includeSelectors: false` and
`includeTemplates: true` in production. Never use deprecated features in demos —
they produce warnings in ArgoCD sync output (`Warning: 'commonLabels' is deprecated`)
that break build verification and obscure real errors.

**5a. Use `labels` not `commonLabels` — the correct overlay pattern:**
```yaml
labels:
  - pairs:
      env: dev
    includeSelectors: false   # safe — does not touch immutable selector
    includeTemplates: true    # adds to pod template labels
```

**6. Patches are still needed even when using `labels`**
The `labels` field only adds labels. Any other per-environment structural change
— image tags, resource limits, liveness probes, environment variables, sidecar
containers — requires a strategic merge patch. Patches can target any Kubernetes
resource type (Deployment, StatefulSet, Service, ConfigMap, custom resources)
by `kind` and `name`, or by `target:` selector for multi-resource patches.
Use JSON6902 patches for delete operations or when SMP's merge logic is insufficient.

**6. Run `kustomize build` locally before every push**
ArgoCD's sync errors are the same errors `kustomize build` produces locally.
Catching a malformed `kustomization.yaml` or a patch that targets the wrong
resource name locally is far faster than waiting for an ArgoCD sync cycle.

**7. ApplicationSet `template.metadata.labels` are labels on the Application object**
Labels set in `template.metadata.labels` (e.g. `env: "{{.path.basename}}"`) are
labels on the ArgoCD Application CRD — not on the Kubernetes workloads inside
the cluster. They are visible to `kubectl get app --show-labels` and are used
by Progressive Sync step selectors (Demo-16) and policy tools.

**8. ArgoCD never deletes namespaces — delete manually after cleanup**
`CreateNamespace=true` creates the namespace. Deleting an Application or
ApplicationSet does not remove the namespace — ArgoCD does not own it. Always
run `kubectl delete ns <name>` explicitly.

---

## What's Next

**Demo-16: ApplicationSet Progressive Sync — Safe Multi-Environment Rollouts**
Demo-16 is a standalone demo that builds its own Kustomize base, overlays, and
ApplicationSet in a separate `demo-16-progressive-sync/` folder. It reuses the
same structural patterns from Demo-14 — base + overlays, Git Directory generator,
Kustomize auto-detection — and adds `strategy: RollingSync` to its ApplicationSet.
This enables staged rollouts where dev syncs first, must be Healthy, then staging,
then prod — so a bad change stops at dev and never reaches production.
Demo-14's resources remain untouched.