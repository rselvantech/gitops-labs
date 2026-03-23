# Demo-07: Deploying Helm Charts with ArgoCD

## Overview

So far in this series we have deployed applications using plain Kubernetes manifests. In this demo we introduce **Helm charts** as the source type in ArgoCD — and explore how ArgoCD treats Helm differently from what you might expect.

The key insight: **ArgoCD does not run `helm install` or `helm upgrade`**. It uses Helm purely as a template engine — runs `helm template` to generate plain Kubernetes manifests, then applies them with `kubectl apply`. This means there is no Helm history, no Helm releases, no Helm secrets in the cluster. ArgoCD manages the lifecycle entirely on its own terms.

**What you'll learn:**
- What a Helm chart structure looks like and what each file does
- How to configure an ArgoCD Application CRD to deploy a Helm chart
- The difference between `helm.values` (inline) and `helm.valueFiles` (file reference)
- How different value files change the deployed resources
- What pruning is and when to use it during sync
- When `replace` and `force` are needed and the risks involved

**What you'll do:**
- Deploy the `helm-guestbook` app from `argoproj/argocd-example-apps` using a Helm chart source
- Switch between `values.yaml` and `values-production.yaml` to see config changes
- Observe ArgoCD diff output when values change
- Trigger a manual sync with pruning enabled
- Explore the `replace` and `force` sync options

## Prerequisites

- ✅ Completed Demo-03 — ArgoCD Application CRD understood
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

### Helm Chart Structure

Before deploying, understand what a Helm chart looks like. The `helm-guestbook` chart used in this demo has this structure:

The chart lives in the public [`argoproj/argocd-example-apps`](https://github.com/argoproj/argocd-example-apps/tree/master/helm-guestbook) repository.ArgoCD reads it directly — no need to clone or copy it locally.


```
helm-guestbook/
├── Chart.yaml             ← chart metadata (name, version, description)
├── values.yaml            ← default values (used unless overridden)
├── values-production.yaml ← production override values
└── templates/
    ├── deployment.yaml    ← Kubernetes Deployment template (Go template syntax)
    ├── service.yaml       ← Kubernetes Service template
    └── _helpers.tpl       ← reusable template helpers (fullname, labels etc.)
```

**Key files explained:**

`Chart.yaml` — identifies the chart. Contains name, version, and description. ArgoCD reads this to understand what it is deploying.

`values.yaml` — default configuration values built into the Helm chart. **This is standard Helm behaviour, not ArgoCD specific** — Helm automatically loads `values.yaml` as the base whenever `helm template` runs, regardless of whether ArgoCD is involved. Think of it as the "base config" that is always loaded first. Any additional value files you specify are merged on top of it.

`values-production.yaml` — an additional values file with production-specific overrides. **Again, standard Helm — not ArgoCD specific.** When specified in `valueFiles`, Helm merges it on top of `values.yaml`, with the production file taking precedence for any overlapping keys. In the guestbook example this changes the Service type from `ClusterIP` to `LoadBalancer`.

`templates/` — Go template files that reference values using `{{ .Values.xxx }}` syntax. ArgoCD runs `helm template` on these to produce plain Kubernetes manifests.

`_helpers.tpl` — defines named template functions like `fullname` that are reused across templates. This is where resource naming logic lives.

> **Note:**
Both `values.yaml` and `values-production.yaml` live inside the chart directory in the
source Git repo — ArgoCD fetches and reads them directly from there at sync time.

**What ArgoCD actually does with this:**
```
Git repo (helm-guestbook/)
         ↓
ArgoCD repository server runs:
  helm template <release-name> <chart-path> --values <values-file>
         ↓
Plain Kubernetes manifests (Deployment, Service)
         ↓
ArgoCD diffs against live cluster state
         ↓
kubectl apply (only what changed)
```

Because ArgoCD generates manifests from templates at sync time, **what gets compared to the live cluster is the rendered output — not the template files themselves**. This is an important distinction.

---

## Folder Structure

```
07-helm-charts-with-argocd/
├── README.md
├── images/
└── src/
    └── guestbook-helm-app.yaml
```

The `helm-guestbook` chart lives in the public `argoproj/argocd-example-apps` repo — ArgoCD reads it directly. No separate config repo needed for this demo.

---

## Step 1: Create the ArgoCD Application CRD

Create `src/guestbook-helm-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: helm-guestbook
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook-helm
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

**Key differences from Demo-03 (plain manifests):**

| Field | Demo-03 (plain) | Demo-07 (Helm) |
|---|---|---|
| `path` | `guestbook` | `helm-guestbook` |
| `helm` block | not present | optional — only needed to customise Helm behaviour |
| `helm.valueFiles` | not applicable | optional — `values.yaml` loaded automatically as base, specify only when using a different file |

ArgoCD auto-detects a Helm chart when it finds `Chart.yaml` in the specified `path`. The `helm` block is only needed when customising Helm behaviour such as specifying a different values file, passing inline values, or setting a custom release name.


`CreateNamespace=true` under `syncOptions` tells ArgoCD to create the `guestbook-helm` namespace automatically if it does not exist — so you do not need to `kubectl create namespace` manually.

> **Note:** `values.yaml` is always loaded automatically by Helm as the base layer — specifying it explicitly in `valueFiles` is redundant but makes the intent clear in the Application CRD.

---

## Step 2: Apply and Verify Initial State

```bash
kubectl apply -f src/guestbook-helm-app.yaml
```

Check the application status:
```bash
kubectl get app guestbook-helm -n argocd
```

Expected:
```
NAME             SYNC STATUS   HEALTH STATUS
guestbook-helm   OutOfSync     Missing
```

`OutOfSync` — ArgoCD has read the desired state from the Helm chart but has not yet applied it to the cluster. `Missing` — the namespace and resources do not exist yet.

This is expected. ArgoCD does **not** auto-deploy without automated sync configured (covered in Demo-06). You must trigger the first sync manually.

---

## Step 3: Manual Sync — Deploy the Application

```bash
argocd app sync guestbook-helm
```

Watch the sync progress:
```bash
kubectl get pods -n guestbook-helm -w
```

Expected after sync completes:
```
NAME                                READY   STATUS    RESTARTS
helm-guestbook-xxxxxxxxx-xxxxx      1/1     Running   0
```

Verify the Application status:
```bash
argocd app get guestbook-helm
```

Expected:
```
Name:               argocd/guestbook-helm
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          guestbook-helm
URL:                https://argocd.example.com/applications/guestbook-helm
Source:
- Repo:             https://github.com/argoproj/argocd-example-apps.git
  Target:           HEAD
  Path:             helm-guestbook
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to HEAD (723b86e)
Health Status:      Healthy

GROUP  KIND        NAMESPACE       NAME                           STATUS   HEALTH   HOOK  MESSAGE
       Namespace                   guestbook-helm                 Running  Synced         namespace/guestbook-helm created
       Service     guestbook-helm  guestbook-helm-helm-guestbook  Synced   Healthy        service/guestbook-helm-helm-guestbook created
apps   Deployment  guestbook-helm  guestbook-helm-helm-guestbook  Synced   Healthy        deployment.apps/guestbook-helm-helm-guestbook created
```

**Verify ArgoCD is tracking the correct Git commit:**
```bash
# Sync Status: Synced to HEAD (xxxxxxx)  ← confirms Git commit ArgoCD deployed from
```

**Notice the resource name**
```bash
GROUP  KIND        NAMESPACE       NAME                           STATUS   HEALTH   HOOK  MESSAGE
       Service     guestbook-helm  guestbook-helm-helm-guestbook  Synced   Healthy        service/guestbook-helm-helm-guestbook created
apps   Deployment  guestbook-helm  guestbook-helm-helm-guestbook  Synced   Healthy        deployment.apps/guestbook-helm-helm-guestbook created
```

- `guestbook-helm-helm-guestbook` — Helm generates this by combining
the release name (`guestbook-helm`, taken from the ArgoCD Application name) and the chart name
(`helm-guestbook` from `Chart.yaml`) using the pattern `{{ .Release.Name }}-{{ .Chart.Name }}`.

- The name looks redundant here because both parts contain "helm" and "guestbook" in reverse order.
This is exactly why `fullNameOverride` in `helm.values` exists — to override this generated name
when the default is too verbose or confusing.

---

## Step 4: Verify the deployment
```bash
# Check all resources created in the namespace
kubectl get all -n guestbook-helm
```

Expected:
```
NAME                                                READY   STATUS    RESTARTS   AGE
pod/guestbook-helm-helm-guestbook-xxxxxxxxx-xxxxx   1/1     Running   0          1m

NAME                                    TYPE        CLUSTER-IP     PORT(S)   AGE
service/guestbook-helm-helm-guestbook   ClusterIP   10.xxx.xxx.x   80/TCP    1m

NAME                                            READY   UP-TO-DATE   AVAILABLE
deployment.apps/guestbook-helm-helm-guestbook   1/1     1            1
```

**Access the guestbook UI:**
```bash
kubectl port-forward svc/guestbook-helm-helm-guestbook -n guestbook-helm 8888:80
```
Open [http://localhost:8888](http://localhost:8888) in your browser.

**Verify the Helm-generated resource name:**
```bash
# Confirm release name and chart name combining to form the resource name
kubectl get deploy -n guestbook-helm
# NAME                            READY
# guestbook-helm-helm-guestbook   1/1
#  ↑ release name   ↑ chart name
#  guestbook-helm + helm-guestbook = guestbook-helm-helm-guestbook
```

---

## Step 5: Explore the ArgoCD Diff

Before making any changes, understand how to read the diff output — this is your primary tool for understanding what ArgoCD will do before it does it:

```bash
argocd app diff guestbook-helm
```

When everything is synced this returns no output — meaning live cluster matches Git exactly.

Now create drift intentionally to see the diff in action:

```bash
# Scale the deployment manually — creates drift from Git
kubectl scale deployment guestbook-helm-helm-guestbook -n guestbook-helm --replicas=3
```

Check the diff:
```bash
argocd app diff guestbook-helm
```

Expected:
```
===== apps/Deployment guestbook-helm/guestbook-helm-helm-guestbook ======
142c142
<  replicas: 3    ← live cluster
---
>  replicas: 1    ← Git (desired state)
```

ArgoCD detected the manual change. The live cluster drifted from the desired state in Git.

```bash
# Sync back to Git state
argocd app sync guestbook-helm
```

After sync, replicas return to 1 as defined in `values.yaml`. **This is exactly how ArgoCD enforces Git as the single source of truth.**

---

## Step 6: Switch Value Files — Dev vs Production


This is where Helm's real value becomes visible in GitOps. Update the Application CRD to use the production values file in `src/guestbook-helm-app.yaml`:
```yaml
    helm:
      valueFiles:
        - values-production.yaml    # changed from values.yaml
```

Apply the change:
```bash
kubectl apply -f src/guestbook-helm-app.yaml
```

Check the diff before syncing:
```bash
argocd app diff guestbook-helm
```

Expected diff:
```
===== /Service guestbook-helm/helm-guestbook ======
spec:
  type: ClusterIP      ← current (from values.yaml)
  type: LoadBalancer   ← desired (from values-production.yaml)
```

The production values file changes the Service type from `ClusterIP` to `LoadBalancer`. ArgoCD detected this and is waiting for you to sync.


> **Important:** `valueFiles` paths are resolved relative to the chart directory
> **inside the source Git repo** — ArgoCD fetches them directly from the repo at sync time.
> ArgoCD cannot reference value files outside the source path. If the file does not exist
> inside the chart directory in the source repo, the sync will fail with:
> `Error: open /tmp/...: no such file or directory`

Sync to apply:
```bash
argocd app sync guestbook-helm
```

Verify the service changed:
```bash
kubectl get svc -n guestbook-helm
```

Expected:
```
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
guestbook-helm-helm-guestbook   LoadBalancer   10.105.204.75   <pending>     80:30609/TCP   100m
```

Switch back to dev values when done:
```yaml
    helm:
      valueFiles:
        - values.yaml
```

```bash
kubectl apply -f src/guestbook-helm-app.yaml
argocd app sync guestbook-helm
```

---

## Step 7: Inline Values with `helm.values`

Instead of referencing a values file, you can provide values inline directly in the
Application CRD using `helm.values`. This is useful for overriding individual values
without creating a separate values file.

### Step 7a: Override `replicaCount`

Update `guestbook-helm-app.yaml`:
```yaml
    helm:
      valueFiles:
        - values.yaml
      values: |
        replicaCount: 2
```

Apply and sync:
```bash
kubectl apply -f guestbook-helm-app.yaml
argocd app sync guestbook-helm
```

Verify replicas scaled to 2:
```bash
kubectl get deploy -n guestbook-helm
```

Expected:
```
NAME                            READY   UP-TO-DATE   AVAILABLE
guestbook-helm-helm-guestbook   2/2     2            2
```
```bash
kubectl get pods -n guestbook-helm
```

Expected:
```
NAME                                             READY   STATUS    RESTARTS
guestbook-helm-helm-guestbook-68bf689bbc-xxxxx   1/1     Running   0
guestbook-helm-helm-guestbook-68bf689bbc-xxxxx   1/1     Running   0
```

Both pods running with `image:v5` from the chart's default `values.yaml` — the inline
`helm.values` only overrode `replicaCount`, everything else remains from `values.yaml`.

Now revert back to 1 replica — remove the `values` block entirely:
```yaml
    helm:
      valueFiles:
        - values.yaml
```
```bash
kubectl apply -f guestbook-helm-app.yaml
kubectl get pods -n guestbook-helm
kubectl get deploy -n guestbook-helm
```

> **Note:** `replicaCount` is still 2, there are 2 pods running still

```bash
argocd app sync guestbook-helm --prune
kubectl get pods -n guestbook-helm
kubectl get deploy -n guestbook-helm
```

Expected — back to 1 pod:
```
NAME                                             READY   STATUS    RESTARTS
guestbook-helm-helm-guestbook-68bf689bbc-xxxxx   1/1     Running   0
```

> **Note:** `--prune` is required when scaling down — the extra pod is a tracked
> resource that ArgoCD will not delete without pruning enabled.More details on pruning in Step 8

---

### Step 7b: Fix Resource Naming with `fullnameOverride`

From Step 3 we noted the generated resource name `guestbook-helm-helm-guestbook` is
awkward. Now we fix it using `fullnameOverride`.

The `_helpers.tpl` (part of helm chat) naming logic explained:
```
fullnameOverride set?
  └── YES → use fullnameOverride directly          → guestbook ✅
  └── NO  → combine release name + chart name
              guestbook-helm + helm-guestbook
              contains? NO
              → guestbook-helm-helm-guestbook      ← current awkward name
```

The chart supports two override options:

| Override | Result |
|---|---|
| none | `guestbook-helm-helm-guestbook` |
| `nameOverride: guestbook` | `guestbook-helm` |
| `fullnameOverride: guestbook` | `guestbook` |

`fullnameOverride` bypasses the naming logic entirely and uses the value directly —
giving the cleanest result. `nameOverride` only replaces the chart name part, so the
release name prefix remains.

Update `guestbook-helm-app.yaml`:
```yaml
    helm:
      valueFiles:
        - values.yaml
      values: |
        fullnameOverride: guestbook
```

Apply and sync **without** `--prune` first — to observe what happens:
```bash
kubectl apply -f guestbook-helm-app.yaml
argocd app sync guestbook-helm
kubectl get all -n guestbook-helm
```

Expected — both old and new named resources exist:
```
NAME                                                  READY   STATUS
pod/guestbook-xxxxxxxxx-xxxxx                         1/1     Running   ← new
pod/guestbook-helm-helm-guestbook-68bf689bbc-xxxxx    1/1     Running   ← old (orphaned)

NAME                                   TYPE        PORT(S)
service/guestbook                      ClusterIP   80/TCP    ← new
service/guestbook-helm-helm-guestbook  ClusterIP   80/TCP    ← old (orphaned)

NAME                                            READY
deployment.apps/guestbook                       1/1   ← new
deployment.apps/guestbook-helm-helm-guestbook   1/1   ← old (orphaned)
```

ArgoCD created the new resources with the correct name but left the old ones untouched.
This is the default behaviour — pruning is disabled. We will clean this up in Step 8.

---

## Step 8: Pruning — Cleaning Up Removed Resources

Continuing from Step 7b — the cluster has both old and new named resources coexisting.
This is exactly the scenario where pruning is needed.

Check what ArgoCD considers out of sync:
```bash
argocd app get guestbook-helm
```

The old resources will be still shown:
```
GROUP  KIND        NAMESPACE       NAME                           STATUS     HEALTH   HOOK  MESSAGE
       Service     guestbook-helm  guestbook-helm-helm-guestbook  OutOfSync  Healthy        ignored (requires pruning)
apps   Deployment  guestbook-helm  guestbook-helm-helm-guestbook  OutOfSync  Healthy        ignored (requires pruning)
       Service     guestbook-helm  guestbook                      Synced     Healthy        service/guestbook created
apps   Deployment  guestbook-helm  guestbook                      Synced     Healthy        deployment.apps/guestbook created
```

>**Note:** orphaned resources are shown with status `OutOfSync` and message `ignored (requires pruning)`.The warning is also visible in the ArgoCD UI — orphaned resources appear with an orange warning icon in the application detail view.


Check pods,deployment and replicaset:
```bash
kubectl get all -n guestbook-helm
```

Old resources still exist — ArgoCD will not delete them without pruning:
```
pod/guestbook-68bf689bbc-zh927                       1/1     Running   ← new
pod/guestbook-helm-helm-guestbook-68bf689bbc-rjjj7   1/1     Running   ← old (still here)

service/guestbook                                    ClusterIP  ← new
service/guestbook-helm-helm-guestbook                ClusterIP  ← old (still here)

deployment.apps/guestbook                            1/1        ← new
deployment.apps/guestbook-helm-helm-guestbook        1/1        ← old (still here)
```


**Sync with prune — clean up orphaned resources:**
```bash
argocd app sync guestbook-helm --prune
kubectl get all -n guestbook-helm
```

Expected — orphaned resources deleted, only new names remain:
```
NAME                               READY   STATUS    RESTARTS   AGE
pod/guestbook-68bf689bbc-zh927     1/1     Running   0          

NAME                TYPE        CLUSTER-IP       PORT(S)   AGE
service/guestbook   ClusterIP   10.101.112.248   80/TCP    

NAME                       READY   UP-TO-DATE   AVAILABLE
deployment.apps/guestbook  1/1     1            1

NAME                                DESIRED   CURRENT   READY
replicaset.apps/guestbook-68bf689bbc  1       1         1
```

Clean cluster state — no orphaned resources.

**When to use pruning:**

| Scenario | Use prune? |
|---|---|
| Normal sync — no removed resources | No |
| Resource renamed via `fullnameOverride` | Yes |
| Removed a resource from Helm chart | Yes |
| Switched from one chart to another | Yes |
| Scaled down replicas via Git | Yes |
| Day-to-day syncing | No |

> **Warning:** Always run `argocd app diff guestbook-helm` before syncing with
> `--prune`. Pruning permanently deletes tracked resources during the sync operation
> — verify exactly what will be removed before proceeding.

---

## Step 9: Replace and Force — Handling Immutable Fields

> **Note:** This is a reference section. This is documented here because it is a common
> real-world issue you will encounter when working with Helm charts in production.

Some Kubernetes fields are **immutable** — they cannot be changed after the resource
is created. The most common example is `spec.selector.matchLabels` on a Deployment.
If a Helm chart update changes these labels, a normal sync will fail with:
```
Deployment.apps "helm-guestbook" is invalid:
spec.selector: Invalid value: ... field is immutable
```

This typically happens when:
- A Helm chart upgrade changes label selectors
- The ArgoCD Application is renamed (changing `{{ .Release.Name }}` in templates)
- A chart refactor renames or restructures selector labels

**Fix — Replace sync:**
```bash
argocd app sync guestbook-helm --replace
```

`--replace` uses `kubectl replace` instead of `kubectl apply` — deletes and recreates
the resource, allowing immutable fields to change. This is a **destructive action** —
the resource and its pods will be briefly unavailable during replacement.

**Force sync** — use when `--replace` alone is not enough (resource stuck in
terminating state):
```bash
argocd app sync guestbook-helm --replace --force
```

**Decision guide:**
```
Normal sync fails?
  └── Immutable field error in output?
        └── Try --replace
              └── Resource stuck in Terminating?
                    └── Use --replace --force  ← last resort
```

> **Warning:** Never use `--replace` or `--force` as default sync flags. Reserve them
> strictly for immutable field errors. In production, Helm chart changes that affect
> immutable fields should be planned carefully to minimise downtime — consider blue/green
> deployment or a delete-and-recreate strategy with a maintenance window.

---

## Verify Final State
```bash
# Application is synced and healthy
argocd app get guestbook-helm
```

Expected:
```
Sync Policy:        Manual
Sync Status:        Synced to HEAD (723b86e)
Health Status:      Healthy

GROUP  KIND        NAMESPACE       NAME                           STATUS     HEALTH   HOOK  MESSAGE
       Service     guestbook-helm  guestbook-helm-helm-guestbook  Succeeded  Pruned         pruned
apps   Deployment  guestbook-helm  guestbook-helm-helm-guestbook  Succeeded  Pruned         pruned
       Service     guestbook-helm  guestbook                      Synced     Healthy        service/guestbook unchanged
apps   Deployment  guestbook-helm  guestbook                      Synced     Healthy        deployment.apps/guestbook unchanged
```
```bash
# All resources running
kubectl get all -n guestbook-helm

# Access the guestbook UI
kubectl port-forward svc/guestbook -n guestbook-helm 8888:80
```

Open [http://localhost:8888](http://localhost:8888) in your browser.

---

## Cleanup
```bash
# Delete the ArgoCD Application
kubectl delete -f guestbook-helm-app.yaml

# Delete the namespace and all deployed resources
kubectl delete namespace guestbook-helm
```

> **Note:** Deleting the ArgoCD Application resource does not automatically delete
> the deployed Kubernetes resources. Always clean up the namespace separately to
> avoid orphaned resources in the cluster.


---

## Key Concepts Summary

**ArgoCD as Helm template engine**
ArgoCD runs `helm template` to generate manifests — never `helm install` or
`helm upgrade`. No Helm history, no Helm releases, no `helm list` output.
ArgoCD owns the full application lifecycle.

**Helm block is optional**
ArgoCD auto-detects a Helm chart when it finds `Chart.yaml` in the specified `path`.
The `helm` block is only needed when customising Helm behaviour — specifying a
different values file, passing inline values, or setting a custom release name.

**valueFiles vs values**
`helm.valueFiles` references files inside the chart directory in the source Git repo.
`helm.values` provides inline YAML overrides directly in the Application CRD.
Both can be combined — inline values always take the highest precedence.

**Values precedence (highest to lowest) — per [official ArgoCD docs](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#values-files):**
```
parameters        (argocd app set -p)    ← highest priority
valuesObject      (inline object block)
values            (inline string block)
valueFiles        (last file listed wins over first)
chart values.yaml                        ← lowest priority, always loaded as base
```

**Pruning**
Disabled by default. Enable with `--prune` only when intentionally removing resources.
Always diff before pruning — it is irreversible during the sync operation.

**Replace**
Required when a Helm chart change affects an immutable field such as
`spec.selector.matchLabels`. Destructive — deletes and recreates the resource.
Use only when a normal sync fails with an immutable field error.

**No Helm rollback**
Since ArgoCD manages the release, `helm rollback` does not work. Rollback is done
via Git — revert the commit in the source repo and sync. Git history is your
rollback history.

**ArgoCD Application deletion does not cascade by default**
Deleting the ArgoCD Application object only removes the pointer — the deployed
Kubernetes resources continue running untouched. This is intentional:

- Protects against accidental deletion of production workloads
- Allows handing back control from ArgoCD to manual management without downtime
- Destruction requires explicit intent — ArgoCD will never silently delete
  your workloads

Always clean up the namespace separately after deleting the Application:
```bash
kubectl delete -f guestbook-helm-app.yaml    # removes ArgoCD pointer
kubectl delete namespace guestbook-helm      # removes deployed resources
```

---

## Lessons Learned

**1. ArgoCD auto-detects Helm — `helm` block is optional**
ArgoCD detects a Helm chart via `Chart.yaml` in the source path. The `helm` block
is only needed for customisation.

**2. `valueFiles` paths are resolved from the source Git repo**
ArgoCD fetches value files directly from the source repo at sync time. Files must
exist inside the chart directory — ArgoCD cannot reference files outside the source
path. If the file is missing the sync fails with:
`Error: open /tmp/...: no such file or directory`

**3. `values.yaml` is always loaded as the base layer**
Helm loads `values.yaml` automatically regardless of what else is specified.
Specifying it explicitly in `valueFiles` is redundant but makes intent clear.

**4. Diff before every sync — especially with prune or replace**
`argocd app diff <app>` shows exactly what will change before anything is applied.
Make this a habit before every manual sync — it is free, safe, and prevents surprises.

**5. Pruning is disabled by default — intentionally**
ArgoCD never deletes resources unless explicitly told to. This is a safety feature —
accidental resource deletion in production can be catastrophic. Always enable pruning
consciously and always diff first.

**6. `--replace` and `--force` are last resort options**
Reserve these for immutable field errors only. In production, plan Helm chart changes
that affect immutable fields carefully — consider a maintenance window or blue/green
strategy to avoid downtime.

---

## What's Next

**Kustomize with ArgoCD**
Enable kustomize block in Application and show how ArgoCD detects Kustomize via kustomization.yaml

## Commands Reference - ArgoCD CLI

```bash
# Login
argocd login localhost:8080 --username admin --insecure

# List all applications
argocd app list

# Check application status
argocd app get guestbook-helm

# Check diff between Git and cluster — run before every sync
argocd app diff guestbook-helm

# Manual sync
argocd app sync guestbook-helm

# Sync with prune — removes orphaned resources
argocd app sync guestbook-helm --prune

# Sync with replace — for immutable field changes
argocd app sync guestbook-helm --replace

# Sync with replace and force — last resort for stuck resources
argocd app sync guestbook-helm --replace --force

# Check repo connection
argocd repo list

# Add repo
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password '<GITHUB-PAT>'
```
