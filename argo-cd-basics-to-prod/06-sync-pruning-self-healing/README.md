# Demo-06: Sync, Pruning & Self-Healing — Automating the GitOps Loop

## Overview

Demo-05 introduced Automated Sync, Pruning, and Self-Healing as part of the production setup. You saw each feature enabled and working. This demo goes deeper.

Here we start fresh — with automation deliberately stripped off — and build back up to full automation one feature at a time. The goal is to understand **exactly**
what each feature does, what it does NOT do, and when to use or avoid it. Each step proves the behaviour live before moving to the next.

The critical distinctions covered here:
- Automated sync watches Git — it does NOT revert manual cluster changes
- Self-heal watches the cluster — it does NOT fire on Git changes
- Pruning is part of a sync operation — it does NOT run independently
- These are three independent toggles, not one combined feature

By the end, you will know not just how to enable these features but precisely
which problem each one solves and why each is off by default.

**What you'll learn:**
- Why ArgoCD's default behaviour is manual-sync-only and when that is the right choice
- The difference between automated sync (triggered by Git changes) and self-healing
  (triggered by live cluster changes) — these are different events with different triggers
- Why pruning is part of a sync operation, not an independent background process
- How to temporarily disable self-healing for debugging without breaking your setup
- When `Replace` and `Force` sync options are needed and why they exist

**What you'll do:**
- Deploy podinfo into a new namespace using the Demo-05 image — no rebuild needed
- Observe the default manual-sync behaviour and understand what OutOfSync means
- Enable automated sync and prove it fires on a Git commit
- Enable pruning and prove it cleans up a deleted resource during sync
- Enable self-healing and prove it reverts a `kubectl scale` within seconds
- Temporarily disable self-healing to simulate a safe debugging window

## Prerequisites

- ✅ Completed Demo-05 — `podinfo-config` registered with ArgoCD, image
  `rselvantech/podinfo:v1.0.0` in Docker Hub, `argocd-config` repo exists
- ✅ ArgoCD running on minikube
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `podinfo-config` and `argocd-config`

**Verify Prerequisites:**

### 1. ArgoCD pods are running
```bash
kubectl get pods -n argocd
```

**Expected:** All pods `Running` and `1/1` Ready.

### 2. ArgoCD UI is accessible
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `http://localhost:8080` — you should see the ArgoCD login page.

### 3. ArgoCD CLI is installed and logged in
```bash
argocd version --client
argocd login localhost:8080 --username admin --insecure
```

**Expected:**
```
argocd: v3.x.x
'admin:login' logged in successfully
```

---

## Concepts

### Why Each Feature Is Off by Default

ArgoCD's default state is deliberate: detect drift, report it, do nothing until
you approve. This is a safety feature, not a limitation.

**Automated sync is off by default** because a bad commit — broken YAML, wrong
image tag, deleted resource — would go straight to production with no review
window. The default gives you a chance to inspect the diff before applying it.

**Pruning is off by default** because ArgoCD cannot distinguish between "this
resource was intentionally deleted from Git" and "someone accidentally deleted
the wrong file." The default is to leave resources in the cluster and let you
decide.

**Self-healing is off by default** because there are legitimate reasons to change
the live cluster state temporarily — debugging, incident response, one-off
scaling. Self-healing would undo all of that immediately.

These are not three separate systems. They are three independent toggles on a
single `syncPolicy` block, and you enable exactly the ones your project's
maturity justifies.

---

### Three Triggers, Three Behaviours

This is the most important concept in this demo. The three features respond to
different events:

```
Event: Git commit pushed to source repo
  └── Automated sync fires (if enabled)
      └── Runs kubectl apply on changed manifests
          └── If resource deleted from Git AND prune: true → deletes from cluster

Event: Live cluster state changes (kubectl apply, kubectl scale, kubectl edit)
  └── Self-heal fires (if enabled)
      └── Immediately reverts cluster back to desired state in Git

Event: Neither of the above
  └── Nothing happens. ArgoCD polls Git every ~3 minutes but takes no action
      unless a difference is detected AND the relevant automation is enabled.
```

**Key distinction:**

| Trigger | Feature | What it watches |
|---|---|---|
| Git changed | Automated sync | Git repository |
| Cluster changed | Self-heal | Live Kubernetes resources |

Automated sync will NOT revert a `kubectl scale`. Self-heal will NOT fire when
you push a commit. They are completely independent.

---

### Pruning Is Part of a Sync Operation

This catches many people by surprise.

Pruning does not run independently. It only runs when a sync operation takes
place. If nothing triggers a sync, orphaned resources stay in the cluster even
with `prune: true` set — because there is no sync to attach the pruning to.

```
prune: true + no sync triggered → orphan stays in cluster
prune: true + sync triggered    → orphan is deleted during that sync
prune: false + sync triggered   → orphan stays in cluster (prune skipped)
```

When manually syncing from the UI without `prune: true` in the manifest, ArgoCD
shows a **Prune** checkbox in the sync dialog. You must tick it explicitly for
that manual sync to prune. The checkbox only appears for manual syncs — when
automated pruning is configured, it runs automatically as part of every automated
sync.

---

### Sync Options — Replace, Force, and Retry

Beyond the three main automation toggles, ArgoCD has sync options for edge cases.

**`Replace=true`** — uses `kubectl replace` instead of `kubectl apply`. Needed
when a resource has immutable fields that have changed (e.g. `matchLabels` in a
Deployment's `spec.selector`). A normal apply will fail with an immutable field
error. Replace deletes and recreates the resource.

**`Force=true`** — force-deletes the resource before recreating it. Used together
with Replace when a Replace alone is not enough (e.g. the resource is stuck in
terminating state). This is destructive — the resource is briefly absent from
the cluster during the operation.

**`Prune=false` annotation** — applied to individual resources to protect them
from pruning even when `prune: true` is set at the Application level:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
```

Use this on resources that ArgoCD manages but should never be auto-deleted — for
example a Namespace that contains resources from multiple Applications.

**`retry` block** — retries a failed sync automatically:

```yaml
syncPolicy:
  retry:
    limit: 3
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

Useful when syncs fail transiently (e.g. a CRD is not yet ready when a CR that
depends on it is applied). Not needed in this demo but worth knowing exists.

---

### Refresh vs Sync — What Is the Difference

**Refresh** tells ArgoCD to immediately re-read the source repository and compare
the latest Git state against the live cluster state. It updates the sync status
(Synced / OutOfSync) but does **not** apply any changes to the cluster. Use
refresh when you want ArgoCD to detect changes without deploying.

**Sync** applies the Git state to the cluster. It implicitly performs a refresh
first to get the latest state, then applies any differences. Use sync when you
want changes deployed.

```
Refresh                   → detects changes, updates status, does NOT apply
Sync ( Refresh + Apply)   → detects changes, updates status, AND applies
```

```bash
# Refresh — re-read repo and update sync status immediately
argocd app get podinfo-sync-demo --refresh

# Sync — apply latest Git state to cluster
argocd app sync podinfo-sync-demo
```

---

### Understanding `argocd app diff`

`argocd app diff` shows the difference between the **desired state in Git**
and the **live state in the cluster**. It uses standard unified diff format:
```
>  lines prefixed with >  →  exists in Git but NOT in cluster (added on sync)
<  lines prefixed with <  →  exists in cluster but NOT in Git (removed on sync if prune enabled)
   no prefix              →  same in both Git and cluster (no change)
```

Each section header identifies the resource being compared:
```
===== /Service podinfo-sync-demo-06/podinfo ======
===== apps/Deployment podinfo-sync-demo-06/podinfo ======
```

Run this before every sync — especially when prune is enabled — to see
exactly what ArgoCD will apply or delete before it touches the cluster.

---

## Folder Structure

```
06-sync-pruning-self-healing/src/
├── podinfo-config/              ← git init → remote: rselvantech/podinfo-config
│   └── demo-06/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── test-configmap.yaml  ← added in Step 5, deleted in Step 6
└── argocd-config/               ← git init → remote: rselvantech/argocd-config
    └── demo-06/
        └── podinfo-app.yaml
```

**What happens on GitHub after all pushes:**

```
rselvantech/podinfo-config (GitHub)
├── deployment.yaml                     ← Demo-05 root (untouched)
├── service.yaml                        ← Demo-05 root (untouched)
└── demo-06-sync-pruning/               ← Demo-06 adds this
    ├── deployment.yaml
    ├── service.yaml
    └── test-configmap.yaml  (added and deleted during demo)

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml                    ← Demo-05 root (untouched)
└── demo-06-sync-pruning/               ← Demo-06 adds this
    └── podinfo-app.yaml
```

> The Demo-05 Application CRD (`podinfo-app.yaml` at root) continues pointing
> to `path: .` and remains completely unaffected. Demo-06 uses its own
> Application CRD under `demo-06/` pointing to `path: demo-06` in
> `podinfo-config`. They are independent and deploy into separate namespaces.

---

## Step 1: Create Namespace and Docker Registry Secret

Create the namespace first — both the docker secret and the ArgoCD Application
destination reference it:

```bash
kubectl create namespace podinfo-sync-demo-06
```

Recreate the Docker Hub secret in the new namespace. The secret from Demo-05's
`podinfo` namespace is not visible here — `imagePullSecrets` is namespace-scoped
and must be recreated in every namespace where private images are pulled:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password='<your-dockerhub-access-token>' \
  --namespace=podinfo-sync-demo-06
```

**Verify:**
```bash
kubectl get secret dockerhub-secret -n podinfo-sync-demo-06
```

**Expected:**
```
NAME               TYPE                             DATA
dockerhub-secret   kubernetes.io/dockerconfigjson   1
```

## Step 1b: Register `podinfo-config` with ArgoCD

`podinfo-config` was registered in Demo-05. Verify it is still registered:
```bash
argocd repo list
```

**Expected:**
```
TYPE  NAME  REPO                                                   STATUS
git         https://github.com/rselvantech/podinfo-config.git     Successful
```

If it is not listed (e.g. ArgoCD was reinstalled or credentials expired),
re-register:
```bash
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

---

## Step 2: Add Manifests to `podinfo-config/demo-06/`

Initialise the local git repo pointing to the existing `podinfo-config` remote:

```bash
cd 06-sync-pruning-self-healing/src
mkdir podinfo-config && cd podinfo-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo-config.git

# Pull existing Demo-05 history
git pull origin main --allow-unrelated-histories --no-rebase

mkdir demo-06-sync-pruning
```

**Create the manifest files:**

**`demo-06-sync-pruning/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: podinfo-sync-demo-06
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      imagePullSecrets:
        - name: dockerhub-secret
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#3d6b9e"
            - name: PODINFO_UI_MESSAGE
              value: "Demo-06: sync, pruning and self-healing"
```

**`demo-06-sync-pruning/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: podinfo-sync-demo-06
spec:
  selector:
    app: podinfo
  ports:
    - port: 9898
      targetPort: 9898
```

**Push to `podinfo-config`:**
```bash
git add demo-06-sync-pruning
git commit -m "feat: add demo-06-sync-pruning-self-healing manifests"
git push origin main
```

---

## Step 3: Apply the Application CRD — Manual Sync Only


> **Why push to Git and `kubectl apply` the Application CRD changes?**
> ArgoCD watches `podinfo-config` (the source repo) — not `argocd-config`
> where the Application CRD lives. Pushing to `argocd-config` does not
> trigger ArgoCD to pick up the change automatically — `kubectl apply`
> is still required to register it.
>
> ```
> podinfo-config  ← ArgoCD watches this (auto-sync applies changes here)
> argocd-config   ← ArgoCD does NOT watch this (kubectl apply required)
> ```
>
> We still push to Git to version-control the Application CRD — keeping
> it auditable and recoverable. This manual apply step is eliminated in
> Demo-11 (App-of-Apps). See Demo-05 Concepts for the full explanation.


**Set up `argocd-config` local repo:**

```bash
cd 06-sync-pruning-self-healing/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git

git pull origin main --allow-unrelated-histories --no-rebase

mkdir demo-06-sync-pruning
```

**Create `argocd-config/demo-06-sync-pruning/podinfo-app.yaml`** — **no syncPolicy at all**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-sync-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-06-sync-pruning
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-sync-demo-06
```

No `syncPolicy` block at all. This is ArgoCD's default — detect drift, report
it, do nothing automatically.

**Push to `argocd-config`:**
```bash
git add demo-06-sync-pruning
git commit -m "feat: add demo-06 Application CRD — manual sync only"
git push origin main
```

**Apply the Application CRD:**
```bash
kubectl apply -f demo-06-sync-pruning/podinfo-app.yaml
```

**Check status:**
```bash
argocd app get podinfo-sync-demo
```

Expected:
```
Sync Status:   OutOfSync
Health Status: Missing
```

`OutOfSync/Missing` — ArgoCD can see the desired state in Git but has not applied
anything. This is the default. No automation will run. Nothing deploys until you
manually sync.

---

## Step 3b: Observe Default Behaviour — OutOfSync Without Auto-Apply

Before enabling any automation, see exactly what ArgoCD does by default:

```bash
# See what ArgoCD wants to apply — inspect before syncing
argocd app diff podinfo-sync-demo
```

**Expected** — all lines prefixed with `>` meaning resources exist in Git but not yet in the cluster. Each `=====` header shows the resource type,namespace and name being compared:

```text
===== /Service podinfo-sync-demo-06/podinfo ======
0a1,13
> apiVersion: v1
> kind: Service
> metadata:
>   name: podinfo
>   namespace: podinfo-sync-demo-06
> spec:
>   ports:
>   - port: 9898
>     targetPort: 9898
>   selector:
>     app: podinfo
===== apps/Deployment podinfo-sync-demo-06/podinfo ======
0a1,29
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: podinfo
>   namespace: podinfo-sync-demo-06
> spec:
>   replicas: 1
...
```

All `>` lines — nothing exists in the cluster yet. A sync will create
everything shown. No `<` lines — nothing will be deleted.


**Now make a change (add a label) to `podinfo-config/demo-06-sync-pruning/deployment.yaml` and push:**


```yaml
cd 06-sync-pruning-self-healing/src/podinfo-config

# Edit demo-06-sync-pruning/deployment.yaml
# add a label under metadata:
  labels:
     version: "v1"
```

```bash
git add demo-06-sync-pruning/deployment.yaml
git commit -m "test: add version label to deployment"
git push origin main
```

**Wait ~3 minutes or run refresh, then check:**
```bash
argocd app get podinfo-sync-demo --refresh
```

Expected: still `OutOfSync`. ArgoCD detected the change in Git but applied nothing. This is intentional — the safety net is working.

```bash
# check new label in the ouput
argocd app diff podinfo-sync-demo
```

**Verify deployment:**
```bash
kubectl get pods -n podinfo-sync-demo-06
```

**Expected:**
```
No resources found in podinfo-sync-demo-06 namespace.
```

**Sync manually:**
```bash
argocd app sync podinfo-sync-demo
```

**Verify deployment:**
```bash
kubectl get pods -n podinfo-sync-demo-06
```

**Expected:**
```
NAME                       READY   STATUS    RESTARTS
podinfo-xxxxxxxxx-xxxxx    1/1     Running   0
```

> **Key observation:** Without `syncPolicy.automated`, every single Git change
> requires a manual sync. In a team with 10 engineers pushing 20 times a day,
> this creates a bottleneck. Automated sync eliminates that bottleneck — once
> the PR is merged, the change is in the cluster without any extra steps.

---

## Step 4: Enable Automated Sync

**Update `argocd-config/demo-06-sync-pruning/podinfo-app.yaml` — add `syncPolicy.automated`:**

```bash
cd 06-sync-pruning-self-healing/src/argocd-config
```

**Update `argocd-config/demo-06-sync-pruning/podinfo-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-sync-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: demo-06-sync-pruning
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-sync-demo-06
  syncPolicy:
    automated: {}           # enables automated sync — empty object is valid YAML
```

**Push and apply:**
```bash
git add demo-06-sync-pruning/podinfo-app.yaml
git commit -m "feat: enable automated sync for demo-06"
git push origin main

kubectl apply -f demo-06-sync-pruning/podinfo-app.yaml
```

**Prove Auto Sync** — make changes in `podinfo-config/demo-06-sync-pruning/deployment.yaml`:

```yaml
cd 06-sync-pruning-self-healing/src/podinfo-config

#Change-1 : update label version to v2
labels:
  version: "v2"

#Change-2 : update PODINFO_UI_MESSAGE value 
env:
  - name: PODINFO_UI_MESSAGE
    value: "Demo-06: automated sync is now enabled"
```

**Push to `podinfo-config`:**
```bash
git add demo-06-sync-pruning/deployment.yaml
git commit -m "feat: update label & UI message to prove auto-sync"
git push origin main
```

Watch ArgoCD — without any manual sync click, the app transitions through
`OutOfSync → Syncing → Synced` automatically:

```bash
argocd app get podinfo-sync-demo --refresh
```

**Expected output:**
```
...
...
Sync Policy:        Automated
Sync Status:        Synced to HEAD (0041a82)
Health Status:      Healthy

GROUP  KIND        NAMESPACE             NAME     STATUS  HEALTH   HOOK  MESSAGE
       Service     podinfo-sync-demo-06  podinfo  Synced  Healthy        service/podinfo unchanged
apps   Deployment  podinfo-sync-demo-06  podinfo  Synced  Healthy        deployment.apps/podinfo configured
```

**Note:**
- The commit SHA in `Synced to HEAD (0041a82)` matches the commit you just pushed.
- `Sync Policy:` is `Automated`
- `STATUS` is `Synced`

**Verify deployment:**
```bash
kubectl get deployments -n podinfo-sync-demo-06 podinfo  -o yaml | grep -E -A1 "labels:|name: PODINFO_UI_MESSAGE"
```

**Expected output:**
```
  labels:
    version: v2
--
      labels:
        app: podinfo
--
        - name: PODINFO_UI_MESSAGE
          value: 'Demo-06: automated sync is now enabled'
```

> **Note on the ~3 minute interval:** ArgoCD polls Git approximately every 3
> minutes. This is the polling interval for detecting changes — not a delay in
> sync speed. Once a change is detected, the sync itself is fast. Clicking
> **Refresh** triggers an immediate poll. 

---

## Step 5: Prove Pruning Is Not Automatic — Add Then Remove a Resource

First, add a ConfigMap to prove auto-sync picks it up:

Create `podinfo-config/demo-06-sync-pruning//test-configmap.yaml` in `podinfo-config`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-06-test
  namespace: podinfo-sync-demo-06
data:
  purpose: "this configmap will be deleted to demonstrate pruning"
  created-by: "Demo-06"
```

**Push:**
```bash
git add demo-06-sync-pruning/test-configmap.yaml
git commit -m "feat: add test ConfigMap for pruning demo"
git push origin main
```

**Wait for auto-sync (~3 min or Refresh), then verify:**
```bash
#Do app Refresh
argocd app get podinfo-sync-demo --refresh
```

**Expected:**
```
GROUP  KIND        NAMESPACE             NAME          STATUS  HEALTH   HOOK  MESSAGE
       ConfigMap   podinfo-sync-demo-06  demo-06-test  Synced                 configmap/demo-06-test created
       Service     podinfo-sync-demo-06  podinfo       Synced  Healthy        service/podinfo unchanged
apps   Deployment  podinfo-sync-demo-06  podinfo       Synced  Healthy        deployment.apps/podinfo unchanged
```
The `demo-06-test` ConfigMap shows `Synced` — it is created in the cluster

**Check configmap:**
```
kubectl get configmap demo-06-test -n podinfo-sync-demo-06
```

**Expected:**
```
NAME           DATA   AGE
demo-06-test   2      30s
```

**Auto-sync deployed configmap automatically. Now delete it from Git:**

```bash
git rm demo-06-sync-pruning/test-configmap.yaml
git commit -m "test: delete ConfigMap to demonstrate pruning behaviour"
git push origin main
```

**Wait for auto-sync (~3 min or Refresh):**
```bash
argocd app get podinfo-sync-demo --refresh
```

**Expected:**
```text
GROUP  KIND        NAMESPACE             NAME          STATUS     HEALTH
       ConfigMap   podinfo-sync-demo-06  demo-06-test  OutOfSync
       Service     podinfo-sync-demo-06  podinfo       Synced     Healthy
apps   Deployment  podinfo-sync-demo-06  podinfo       Synced     Healthy
```

The `demo-06-test` ConfigMap shows `OutOfSync` — it exists in the cluster
but no longer has a corresponding manifest in Git. This is an orphaned
resource. ArgoCD has detected it but will not delete it because `prune`
is not enabled.

**To see the full orphan warning in UI:**
```
Click on the podinfo-sync-demo application
  → find demo-06-test ConfigMap in the resource tree (shown in yellow/orange)
  → click on it
  → you will see the "status:OutOfSync (This resource is not present in the application's source. It will be deleted from Kubernetes if the prune option is enabled during sync.)"
```

**Confirm Configmap still exist in the cluste**
```bash
kubectl get configmap demo-06-test -n podinfo-sync-demo-06
```

**Expected:**
```
NAME           DATA   AGE
demo-06-test   2      4m
```

**This is the key insight:** Auto-sync ran. It saw the ConfigMap is gone from
Git. But it did not delete it from the cluster because `prune` is not enabled.
The resource is now an orphan — alive in the cluster with no Git backing.

> This is why `prune: false` is the default. ArgoCD cannot know if you deleted
> that file intentionally or by mistake. It leaves the resource and warns you.

---

## Step 6: Enable Pruning — Clean Up the Orphan

```bash
cd 06-sync-pruning-self-healing/src/argocd-config
```

**Update `argocd-config/demo-06-sync-pruning/podinfo-app.yaml`:**
```yaml
syncPolicy:
  automated:
    prune: true             # delete resources from cluster when removed from Git
```

**Push and apply:**
```bash
git add demo-06-sync-pruning/podinfo-app.yaml
git commit -m "feat: enable pruning for demo-06"
git push origin main

kubectl apply -f demo-06-sync-pruning/podinfo-app.yaml
```


**Trigger a sync to activate pruning:**

Enabling `prune: true` in the Application CRD does not immediately delete
the orphan. Remember — **pruning only runs during a sync operation**. The
orphan ConfigMap was already removed from Git in the previous step, but no
sync has happened since we enabled pruning. We need to trigger a sync now.

We push a small change to `podinfo-config` to trigger auto-sync. This is
intentional — it proves that pruning runs as part of a normal sync, not as
a separate background process:
```bash
cd 06-sync-pruning-self-healing/src/podinfo-config

# Edit `podinfo-config/demo-06-sync-pruning/deployment.yaml` — update the UI message:
#   value: "Demo-06: pruning is now enabled"

git add demo-06-sync-pruning/deployment.yaml
git commit -m "feat: update message to trigger sync with pruning enabled"
git push origin main
```

When auto-sync fires it will do two things in the same sync operation:
- Apply the updated deployment message
- Delete the orphaned ConfigMap because `prune: true` is now set

This is the exact behaviour to understand: **pruning piggybacks on a sync
triggered by any Git change — it does not run on its own.**

**Wait for auto-sync (~3 min or Refresh), then verify the orphan is gone:**
```bash
kubectl get configmap demo-06-test -n podinfo-sync-demo-06
```

**Expected:**
```
Error from server (NotFound): configmaps "demo-06-test" not found
```

```bash
argocd app get podinfo-sync-demo
```

**Expected:**
```text
Name:               argocd/podinfo-sync-demo
...
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (bf59677)
Health Status:      Healthy

GROUP  KIND        NAMESPACE             NAME          STATUS     HEALTH   HOOK  MESSAGE
       ConfigMap   podinfo-sync-demo-06  demo-06-test  Succeeded  Pruned         pruned
       Service     podinfo-sync-demo-06  podinfo       Synced     Healthy        service/podinfo unchanged
apps   Deployment  podinfo-sync-demo-06  podinfo       Synced     Healthy        deployment.apps/podinfo unchanged
```

**Key observations:**

- `Sync Policy: Automated (Prune)` — confirms pruning is now active.
  Compare with earlier where it showed `Automated` only
- `demo-06-test ConfigMap → STATUS: Succeeded, HEALTH: Pruned, MESSAGE: pruned`
  — ArgoCD explicitly records the pruning action in the sync result.
  The resource is gone from the cluster and the sync history shows it was
  pruned, not just removed silently
- `Synced to HEAD` — the entire application is now in sync. No orphans
  remain

> The `Pruned` health status and `pruned` message are only visible
> immediately after the sync that removed the resource. On the next
> `argocd app get` call the ConfigMap row will disappear entirely from
> the output since it no longer exists in Git or the cluster.

> **Remember:** Pruning only runs during a sync. The combination of
> `automated: {}` + `prune: true` means every Git change triggers a sync
> which includes pruning. Without automated sync, you must remember to tick
> the Prune checkbox on every manual sync or the orphan persists silently.

---

## Step 7: Enable Self-Healing — Prove Live Cluster Drift Is Reverted


```bash
cd 06-sync-pruning-self-healing/src/argocd-config
```

**Update `argocd-config/demo-06-sync-pruning/podinfo-app.yaml`:**
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true          # revert any live cluster changes back to Git state
```

**Push and apply:**
```bash
git add demo-06-sync-pruning/podinfo-app.yaml
git commit -m "feat: enable self-healing for demo-06"
git push origin main

kubectl apply -f demo-06-sync-pruning/podinfo-app.yaml
```

**Prove it — scale the deployment imperatively:**

**Open a second terminal and watch pods:**
```bash
kubectl get pods -n podinfo-sync-demo-06 --watch
```

**In the first terminal:**
```bash
kubectl scale deploy podinfo -n podinfo-sync-demo-06 --replicas=5
```

**Expected in the watch terminal — pods are created and almost immediately deleted:**
```
podinfo-xxxxxxxxx-aaaaa   0/1   Pending              0   1s
podinfo-xxxxxxxxx-bbbbb   0/1   Pending              0   1s
podinfo-xxxxxxxxx-ccccc   0/1   ContainerCreating    0   2s
podinfo-xxxxxxxxx-aaaaa   1/1   Running              0   4s
podinfo-xxxxxxxxx-aaaaa   1/1   Terminating          0   6s   ← ArgoCD reverted
podinfo-xxxxxxxxx-bbbbb   0/1   Terminating          0   6s
podinfo-xxxxxxxxx-ccccc   0/1   Terminating          0   6s
```

The pods barely reach `Running` before ArgoCD reverts the cluster back to
`replicas: 1`. ArgoCD continuously watches the cluster via the Kubernetes API
server — the revert is near-instant.


**Verify final state — after self-heal completes:**
```bash
argocd app get podinfo-sync-demo
```

**Expected:**
```text
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (035de4d)
Health Status:      Healthy

GROUP  KIND        NAMESPACE             NAME     STATUS  HEALTH   HOOK  MESSAGE
apps   Deployment  podinfo-sync-demo-06  podinfo  Synced  Healthy        deployment.apps/podinfo configured
       Service     podinfo-sync-demo-06  podinfo  Synced  Healthy
```

**Key observation — `configured` not `unchanged`:**

`MESSAGE: deployment.apps/podinfo configured` is the self-heal fingerprint
in the CLI output. `configured` means ArgoCD re-applied the Deployment to
correct the manual `kubectl scale` drift. A normal sync with no cluster
changes would show `unchanged` instead.

**Verify self-healing is enabled in App Details:**
```
ArgoCD UI → podinfo-sync-demo → App Details → SYNC POLICY section
```

**Expected:**
```text
AUTO-SYNC:  Enabled
PRUNE:      Enabled
SELF HEAL:  Enabled
```

**Verify using Deployment events — the most reliable CLI evidence:**
```bash
kubectl describe deployment podinfo -n podinfo-sync-demo-06 | grep -A20 Events
```

**Expected — shows scale up then immediate scale down:**
```text
Events:
  Type    Reason            Message
  Normal  ScalingReplicaSet  Scaled up replica set to 5    ← your kubectl scale
  Normal  ScalingReplicaSet  Scaled down replica set to 1  ← ArgoCD self-heal
```

This is the definitive proof — the same events list shows your manual
change immediately followed by ArgoCD reverting it.

> **Self-heal vs automated sync — the critical distinction:**
> The `kubectl scale` command did NOT push a commit to Git. Automated sync
> would never have caught this — it only watches Git. Self-heal watches the
> live cluster and fires the moment the live state diverges from Git,
> regardless of whether Git changed.

---

## Step 8: Temporarily Disable Self-Healing for Debugging

Self-healing is aggressive. When you need to manually change the cluster state
for debugging, you need a way to pause it without destroying your configuration.

**Pattern: disable self-heal via Git commit → debug → re-enable via Git commit.**

**Update `argocd-config/demo-06-sync-pruning/podinfo-app.yaml`:**

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: false         # temporarily disabled for debugging
```

**Push and apply:**
```bash
git add demo-06-sync-pruning/podinfo-app.yaml
git commit -m "chore: temporarily disable self-healing for debugging"
git push origin main

kubectl apply -f demo-06-sync-pruning/podinfo-app.yaml
```

**Verify self-heal is off:**
```bash
kubectl scale deploy podinfo -n podinfo-sync-demo-06 --replicas=3
kubectl get pods -n podinfo-sync-demo-06 --watch
```

**Expected — pods stay at 3 and are NOT reverted:**
```
podinfo-xxxxxxxxx-aaaaa   1/1   Running   0   10m
podinfo-xxxxxxxxx-bbbbb   1/1   Running   0   30s
podinfo-xxxxxxxxx-ccccc   1/1   Running   0   30s
```

ArgoCD marks the app `OutOfSync` but does not revert. You now have a debugging
window — inspect pods, exec into containers, test with different replica counts,
apply temporary patches — without ArgoCD undoing your work.

When debugging is complete, re-enable:
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true          # re-enabled
```

```bash
git add demo-06-sync-pruning/podinfo-app.yaml
git commit -m "feat: re-enable self-healing after debugging"
git push origin main

kubectl apply -f demo-06-sync-pruning/podinfo-app.yaml
```

ArgoCD immediately reverts the cluster back to `replicas: 1` upon re-enabling.

---

## Verify Final State

```bash
# Application synced and healthy
argocd app get podinfo-sync-demo

# One pod running
kubectl get pods -n podinfo-sync-demo-06

# Final Application CRD has all three features enabled
kubectl get app podinfo-sync-demo -n argocd -o yaml | grep -A5 "automated:"

# podinfo-config registered, argocd-config not registered
argocd repo list
```

---

## Cleanup

```bash
# Delete the Application
kubectl delete app podinfo-sync-demo -n argocd

# Delete the namespace — also removes the docker secret
kubectl delete namespace podinfo-sync-demo-06

# No repo deregistration needed — podinfo-config was registered in Demo-05
# and is reused in later demos
```

---

## Key Concepts Summary

**Automated sync — Git change trigger**
ArgoCD polls Git every ~3 minutes. When it detects the desired state in Git
differs from the live cluster, it applies the changes automatically without
waiting for manual approval. Enable once your CI pipeline reliably catches
bad changes before merge.

**Pruning — orphan cleanup during sync**
When a resource is removed from Git, ArgoCD marks it as an orphan with a
warning but does not delete it by default. With `prune: true`, the resource
is deleted during the next sync. Pruning never runs independently — a sync
must occur for pruning to run.

**Self-healing — cluster drift trigger**
When the live cluster state diverges from Git (via `kubectl`, direct API
calls, or admission controllers), ArgoCD detects the drift and reverts it.
This is completely independent of Git — it fires on cluster events, not
Git events. Disable temporarily for debugging, re-enable when done.

**The three triggers in one table:**

| What changed | Feature that responds | How fast |
|---|---|---|
| Git commit pushed | Automated sync | ~3 min poll, or instant via webhook |
| `kubectl` change to cluster | Self-heal | Near-instant (continuous watch) |
| Resource deleted from Git | Prune (during next sync) | On next sync cycle |
| Nothing changed | Nothing | — |

**Full syncPolicy — all three enabled:**
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

---

## Commands Reference

```bash

# Check sync status and health
argocd app get podinfo-sync-demo

# Refresh — re-read repo and update sync status immediately (does not apply)
argocd app get podinfo-sync-demo --refresh

# See diff between Git and live cluster
argocd app diff podinfo-sync-demo

# Manual sync with prune (when prune: true not in manifest)
argocd app sync podinfo-sync-demo --prune

# Force sync for immutable field errors
argocd app sync podinfo-sync-demo --force

# Verify all three sync features enabled in live Application spec
kubectl get app podinfo-sync-demo -n argocd -o yaml | grep -A5 "automated:"

# Check ArgoCD reconciliation interval
kubectl describe cm argocd-cm -n argocd | grep timeout.reconciliation
```

---

## Lessons Learned

**1. Automated sync is not unconditional — your CI gate must come first**
Enabling automated sync means every merged commit goes straight to the cluster.
This is only safe if your CI pipeline (lint, schema validation, image scanning)
runs before merge and catches bad changes. Automated sync is the reward for a
mature CI process, not a replacement for it.

**2. Pruning requires a sync — design your change triggers accordingly**
If you delete a resource from Git but no other change triggers a sync, the
orphan stays. With automated sync enabled, every push triggers a sync so
pruning runs reliably. With manual sync, you must remember to tick the Prune
checkbox or the orphan persists silently.

**3. Self-heal and automated sync watch different things — know which you need**
A common mistake is enabling automated sync and expecting it to revert `kubectl`
changes. It will not. Automated sync watches Git. Self-heal watches the cluster.
For a fully closed GitOps loop you need both.

**4. Always use `argocd app diff` before a manual sync**
Before clicking Sync (especially with prune enabled), run `argocd app diff` to
see exactly what will be applied and what will be deleted. This is your last
inspection window before changes hit the cluster.

---

## What's Next

**Demo-07: Helm Charts with ArgoCD**
Deploy applications using Helm charts as the source. Understand how ArgoCD
treats Helm as a template engine (not a release manager), override chart
values inline, and use value files for environment-specific configuration.
The sync features enabled in this demo — automated sync, pruning, and
self-healing — carry forward as the baseline for all subsequent demos.