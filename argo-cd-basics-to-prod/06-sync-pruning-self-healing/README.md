# Demo-06: Sync, Pruning & Self-Healing — Automating the GitOps Loop

## Overview

Until now every Application CRD we created required a manual sync click to apply
changes. That works for learning but it is not GitOps — it is just Git-stored
configuration with a human in the middle of every deployment.

In this demo we close the automation loop completely. We start with the default
ArgoCD behaviour (manual sync only), understand exactly why each automation feature
is off by default, and then enable each one step by step while proving its effect
live on the cluster.

By the end, the application will automatically deploy every Git push, clean up
every deleted resource, and immediately revert any manual change made directly to
the cluster — without any human intervention.

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
- ✅ ArgoCD running on minikube (`kubectl get pods -n argocd`)
- ✅ ArgoCD CLI installed and logged in
  (`argocd login localhost:8080 --username admin --insecure`)
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `podinfo-config` and `argocd-config`

**Verify prerequisites:**
```bash
kubectl get pods -n argocd
argocd app list          # should show Demo-05 podinfo app
argocd repo list         # should show podinfo-config registered
```

---

## Background: Why Each Feature Is Off by Default

ArgoCD's default state is deliberate: detect drift, report it, do nothing until you
approve. This is a safety feature, not a limitation.

**Automated sync is off by default** because a bad commit — broken YAML, wrong image
tag, deleted resource — would go straight to production with no review window. The
default gives you a chance to inspect the diff before applying it.

**Pruning is off by default** because ArgoCD cannot distinguish between "this
resource was intentionally deleted from Git" and "someone accidentally deleted the
wrong file." The default is to leave resources in the cluster and let you decide.

**Self-healing is off by default** because there are legitimate reasons to change
the live cluster state temporarily — debugging, incident response, one-off scaling.
Self-healing would undo all of that immediately.

These are not three separate systems. They are three independent toggles on a single
`syncPolicy` block, and you enable exactly the ones your project's maturity justifies.

---

## Background: Three Triggers, Three Behaviours

This is the most important concept in this demo. The three features respond to
different events:

```
Event: Git commit pushed to source repo
  └── Automated sync fires (if enabled)
      └── Runs helm template / kubectl apply
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

## Background: Pruning Is Part of a Sync Operation

This catches many people by surprise.

Pruning does not run independently. It only runs when a sync operation takes place.
If nothing triggers a sync, orphaned resources stay in the cluster even with
`prune: true` set — because there is no sync to attach the pruning to.

```
prune: true + no sync triggered → orphan stays in cluster
prune: true + sync triggered    → orphan is deleted during that sync

prune: false + sync triggered   → orphan stays in cluster (prune skipped)
```

When manually syncing from the UI without `prune: true` in the manifest, ArgoCD
shows a **Prune** checkbox in the sync dialog. You must tick it explicitly for that
manual sync to prune. The checkbox only appears for manual syncs — when automated
pruning is configured, it runs automatically as part of every automated sync.

---

## Background: Sync Options — Replace, Force, and Retry

Beyond the three main automation toggles, ArgoCD has sync options for edge cases.

**`Replace=true`** — uses `kubectl replace` instead of `kubectl apply`. Needed when
a resource has immutable fields that have changed (e.g. `matchLabels` in a
Deployment's `spec.selector`). A normal apply will fail with an immutable field
error. Replace deletes and recreates the resource.

**`Force=true`** — force-deletes the resource before recreating it. Used together
with Replace when a Replace alone is not enough (e.g. the resource is stuck in
terminating state). This is destructive — the resource is briefly absent from
the cluster during the operation.

**`Prune=false` annotation** — applied to individual resources to protect them from
pruning even when `prune: true` is set at the Application level:

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
├── deployment.yaml        ← Demo-05 root (untouched)
├── service.yaml           ← Demo-05 root (untouched)
└── demo-06/               ← Demo-06 adds this
    ├── deployment.yaml
    ├── service.yaml
    └── test-configmap.yaml  (added and deleted during demo)

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml       ← Demo-05 root (untouched)
└── demo-06/               ← Demo-06 adds this
    └── podinfo-app.yaml
```

> The Demo-05 Application CRD (`podinfo-app.yaml` at root) continues pointing to
> `path: .` and remains completely unaffected. Demo-06 uses its own Application CRD
> under `demo-06/` pointing to `path: demo-06` in `podinfo-config`. They are
> independent and deploy into separate namespaces.

---

## Step 1: Create Namespace and Docker Registry Secret

Create the namespace first — both the docker secret and the ArgoCD Application
destination reference it:

```bash
kubectl create namespace podinfo-sync-demo-06
```

Recreate the Docker Hub secret in the new namespace. The secret from Demo-05's
`podinfo` namespace is not visible here — it must be recreated per namespace:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password='<your-dockerhub-access-token>' \
  --namespace=podinfo-sync-demo-06
```

Verify:
```bash
kubectl get namespace podinfo-sync-demo-06
kubectl get secret dockerhub-secret -n podinfo-sync-demo-06
```

Expected:
```
NAME                    STATUS   AGE
podinfo-sync-demo-06    Active   5s

NAME               TYPE                             DATA
dockerhub-secret   kubernetes.io/dockerconfigjson   1
```

---

## Step 2: Add Manifests to `podinfo-config/demo-06/`

Initialise the local git repo pointing to the existing `podinfo-config` remote:

```bash
cd gitops-labs/argo-cd-basics-to-prod/06-sync-pruning-self-healing/src/podinfo-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo-config.git

# Pull existing Demo-05 history
git pull origin main --allow-unrelated-histories --no-rebase
```

Create the manifest files:

**`demo-06/deployment.yaml`:**
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

**`demo-06/service.yaml`:**
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

Push to `podinfo-config`:
```bash
git add demo-06/deployment.yaml demo-06/service.yaml
git commit -m "feat: add demo-06 manifests for sync/pruning/self-healing demo"
git push origin main
```

---

## Step 3: Apply the Application CRD — Manual Sync Only

Set up `argocd-config` local repo:

```bash
cd gitops-labs/argo-cd-basics-to-prod/06-sync-pruning-self-healing/src/argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git

git pull origin main --allow-unrelated-histories --no-rebase
```

Create `demo-06/podinfo-app.yaml` — **no syncPolicy at all**:

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
    path: demo-06
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-sync-demo-06
```

Notice: no `syncPolicy` block at all. This is ArgoCD's default — detect drift,
report it, do nothing automatically.

Push to `argocd-config`:
```bash
git add demo-06/podinfo-app.yaml
git commit -m "feat: add demo-06 Application CRD — manual sync only"
git push origin main
```

Apply the Application CRD:
```bash
kubectl apply -f demo-06/podinfo-app.yaml
```

Check status:
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
# See what ArgoCD wants to apply
argocd app diff podinfo-sync-demo
```

Expected — shows the Deployment and Service that exist in Git but not in cluster:
```
...
+kind: Deployment
+metadata:
+  name: podinfo
...
+kind: Service
+metadata:
+  name: podinfo
...
```

Now make a change — push a label update to `demo-06/deployment.yaml`:

```bash
# Edit deployment.yaml — add a label under metadata:
#   labels:
#     version: "v1"

git add demo-06/deployment.yaml
git commit -m "test: add version label to deployment"
git push origin main
```

Wait ~3 minutes or click **Refresh** in ArgoCD UI, then:
```bash
argocd app get podinfo-sync-demo
```

Expected: still `OutOfSync`. ArgoCD detected the change in Git but applied nothing.
This is intentional — the safety net is working.

**To deploy manually:**
```bash
argocd app sync podinfo-sync-demo
```

Verify deployment:
```bash
kubectl get pods -n podinfo-sync-demo-06
```

Expected:
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

Update `demo-06/podinfo-app.yaml` — add `syncPolicy.automated`:

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
    path: demo-06
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-sync-demo-06
  syncPolicy:
    automated: {}           # enables automated sync — empty object is valid YAML
```

Push and apply:
```bash
git add demo-06/podinfo-app.yaml
git commit -m "feat: enable automated sync for demo-06"
git push origin main

kubectl apply -f demo-06/podinfo-app.yaml
```

**Prove it:** Now make a change to `podinfo-config/demo-06/deployment.yaml` — change
the UI message:

```yaml
env:
  - name: PODINFO_UI_MESSAGE
    value: "Demo-06: automated sync is now enabled"
```

Push to `podinfo-config`:
```bash
git add demo-06/deployment.yaml
git commit -m "feat: update UI message to prove auto-sync"
git push origin main
```

Watch ArgoCD:
```bash
# Wait ~3 minutes or click Refresh in UI, then:
argocd app get podinfo-sync-demo
```

Expected — without any manual sync click, the app transitions through
`OutOfSync → Syncing → Synced` automatically.

```bash
kubectl get pods -n podinfo-sync-demo-06
```

The pod was replaced automatically. The commit SHA in `argocd app get` matches the
commit you just pushed.

> **Note on the ~3 minute interval:** ArgoCD polls Git approximately every 3 minutes.
> This is not a delay in ArgoCD's sync speed — it is the polling interval for
> detecting changes. Once a change is detected, the sync itself is fast. Clicking
> **Refresh** in the UI triggers an immediate poll without waiting for the interval.
> In production, you can also use ArgoCD's webhook support to trigger immediate
> detection on every push, eliminating the polling delay entirely.

---

## Step 5: Prove Pruning Is Not Automatic — Add Then Remove a Resource

First, add a ConfigMap to prove auto-sync picks it up:

Create `demo-06/test-configmap.yaml` in `podinfo-config`:

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

Push:
```bash
git add demo-06/test-configmap.yaml
git commit -m "feat: add test ConfigMap for pruning demo"
git push origin main
```

Wait for auto-sync (~3 min or Refresh), then verify:
```bash
kubectl get configmap demo-06-test -n podinfo-sync-demo-06
```

Expected:
```
NAME           DATA   AGE
demo-06-test   2      30s
```

Auto-sync deployed it. Now delete it from Git:

```bash
git rm demo-06/test-configmap.yaml
git commit -m "test: delete ConfigMap to demonstrate pruning behaviour"
git push origin main
```

Wait for auto-sync, then check:
```bash
kubectl get configmap demo-06-test -n podinfo-sync-demo-06
```

Expected — **ConfigMap still exists in the cluster:**
```
NAME           DATA   AGE
demo-06-test   2      4m
```

Check ArgoCD:
```bash
argocd app get podinfo-sync-demo
```

Expected — the ConfigMap appears as a yellow `OutOfSync` resource with this message:
```
This resource is not present in the application source.
It will be deleted from Kubernetes if the prune option is enabled during sync.
```

**This is the key insight:** Auto-sync ran. It saw the ConfigMap is gone from Git.
But it did not delete it from the cluster because `prune` is not enabled. The
resource is now an orphan — alive in the cluster with no Git backing.

> This is why `prune: false` is the default. ArgoCD cannot know if you deleted
> that file intentionally or by mistake. It leaves the resource and warns you.

---

## Step 6: Enable Pruning — Clean Up the Orphan

Update `demo-06/podinfo-app.yaml`:

```yaml
syncPolicy:
  automated:
    prune: true             # delete resources from cluster when removed from Git
```

Push and apply:
```bash
# In argocd-config/
git add demo-06/podinfo-app.yaml
git commit -m "feat: enable pruning for demo-06"
git push origin main

kubectl apply -f demo-06/podinfo-app.yaml
```

Now trigger a sync to activate pruning. The ConfigMap was already removed from Git —
we just need a sync to run with pruning enabled. Push a trivial change to
`podinfo-config/demo-06/` to trigger it:

```bash
# In podinfo-config/ — bump the UI message
# Edit demo-06/deployment.yaml:
#   value: "Demo-06: pruning is now enabled"

git add demo-06/deployment.yaml
git commit -m "feat: update message to trigger sync with pruning enabled"
git push origin main
```

Wait for auto-sync, then verify:
```bash
kubectl get configmap demo-06-test -n podinfo-sync-demo-06
```

Expected:
```
Error from server (NotFound): configmaps "demo-06-test" not found
```

The orphan is gone. ArgoCD deleted it during the sync because `prune: true` was
set and the ConfigMap had no corresponding manifest in Git.

```bash
argocd app get podinfo-sync-demo
```

Expected: `Synced` and `Healthy` — no more yellow orphan resource in the tree.

> **Remember:** Pruning only runs during a sync. If no sync is triggered, orphans
> stay indefinitely even with `prune: true`. The combination of `automated: {}` +
> `prune: true` means every Git change triggers a sync which includes pruning.

---

## Step 7: Enable Self-Healing — Prove Live Cluster Drift Is Reverted

Update `demo-06/podinfo-app.yaml` — add `selfHeal: true`:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true          # revert any live cluster changes back to Git state
```

Push and apply:
```bash
git add demo-06/podinfo-app.yaml
git commit -m "feat: enable self-healing for demo-06"
git push origin main

kubectl apply -f demo-06/podinfo-app.yaml
```

**Prove it — scale the deployment imperatively:**

Open a second terminal and watch pods:
```bash
kubectl get pods -n podinfo-sync-demo-06 --watch
```

In the first terminal:
```bash
kubectl scale deploy podinfo -n podinfo-sync-demo-06 --replicas=5
```

Expected in the watch terminal — pods are created and almost immediately deleted:
```
podinfo-xxxxxxxxx-aaaaa   0/1   Pending    0   1s
podinfo-xxxxxxxxx-bbbbb   0/1   Pending    0   1s
podinfo-xxxxxxxxx-ccccc   0/1   ContainerCreating  0   2s
podinfo-xxxxxxxxx-aaaaa   1/1   Running    0   4s
podinfo-xxxxxxxxx-aaaaa   1/1   Terminating  0   6s   ← ArgoCD reverted
podinfo-xxxxxxxxx-bbbbb   0/1   Terminating  0   6s
podinfo-xxxxxxxxx-ccccc   0/1   Terminating  0   6s
```

The pods barely reach `Running` before ArgoCD reverts the cluster back to
`replicas: 1`. ArgoCD is continuously watching the cluster via the Kubernetes API
server — the revert is near-instant.

Verify final state:
```bash
kubectl get pods -n podinfo-sync-demo-06
```

Expected: exactly 1 pod running.

```bash
argocd app get podinfo-sync-demo
```

Expected: `Synced` and `Healthy`.

> **Self-heal vs automated sync — the critical distinction:**
> The `kubectl scale` command did NOT push a commit to Git. Automated sync would
> never have caught this because automated sync only watches Git. Self-heal watches
> the live cluster via the Kubernetes API server and fires the moment the live
> state diverges from Git — regardless of whether Git changed.

---

## Step 8: Temporarily Disable Self-Healing for Debugging

Self-healing is aggressive. When you need to manually change the cluster state for
debugging, you need a way to pause it without destroying your configuration.

**Pattern: disable self-heal, do debugging, re-enable.**

Update `demo-06/podinfo-app.yaml`:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: false         # temporarily disabled for debugging
```

Push and apply:
```bash
git add demo-06/podinfo-app.yaml
git commit -m "chore: temporarily disable self-healing for debugging"
git push origin main

kubectl apply -f demo-06/podinfo-app.yaml
```

Verify self-heal is off:
```bash
kubectl scale deploy podinfo -n podinfo-sync-demo-06 --replicas=3
kubectl get pods -n podinfo-sync-demo-06 --watch
```

Expected — pods stay at 3 and are NOT reverted:
```
podinfo-xxxxxxxxx-aaaaa   1/1   Running   0   10m
podinfo-xxxxxxxxx-bbbbb   1/1   Running   0   30s
podinfo-xxxxxxxxx-ccccc   1/1   Running   0   30s
```

ArgoCD will mark the app `OutOfSync` but will not revert. You now have a
debugging window to inspect, exec into pods, test with different replica counts,
or apply temporary patches — without ArgoCD undoing your work.

When debugging is complete, re-enable:
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true          # re-enabled
```

```bash
git add demo-06/podinfo-app.yaml
git commit -m "feat: re-enable self-healing after debugging"
git push origin main

kubectl apply -f demo-06/podinfo-app.yaml
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

# Only podinfo-config registered — argocd-config never registered
argocd repo list

# Confirm Demo-05 application is still running and unaffected
argocd app list
kubectl get pods -n podinfo    # Demo-05 pods still running
```

---

## Cleanup

```bash
# Delete the Application
kubectl delete app podinfo-sync-demo -n argocd

# Delete the namespace (also removes the docker secret)
kubectl delete namespace podinfo-sync-demo-06

# No repo deregistration needed — podinfo-config was already registered in Demo-05
# and is reused in later demos
```

---

## Key Concepts Summary

**Automated sync — Git change trigger**
ArgoCD polls Git every ~3 minutes. When it detects the desired state in Git differs
from the live cluster, it applies the changes automatically without waiting for
manual approval. Enable once your CI pipeline reliably catches bad changes before
merge.

**Pruning — orphan cleanup during sync**
When a resource is removed from Git, ArgoCD marks it as an orphan with a warning
but does not delete it by default. With `prune: true`, the resource is deleted
during the next sync. Pruning never runs independently — a sync must occur for
pruning to run.

**Self-healing — cluster drift trigger**
When the live cluster state diverges from Git (via `kubectl`, direct API calls, or
admission controllers), ArgoCD detects the drift and reverts it. This is completely
independent of Git — it fires on cluster events, not Git events. Disable temporarily
for debugging, re-enable when done.

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
# Apply Application CRD
kubectl apply -f demo-06/podinfo-app.yaml

# Check sync status and health
argocd app get podinfo-sync-demo

# See diff between Git and live cluster
argocd app diff podinfo-sync-demo

# Manual sync (when automated sync is disabled)
argocd app sync podinfo-sync-demo

# Manual sync with prune (when prune: true not in manifest)
argocd app sync podinfo-sync-demo --prune

# Force sync for immutable field errors
argocd app sync podinfo-sync-demo --force

# Watch pods in real time
kubectl get pods -n podinfo-sync-demo-06 --watch

# Scale deployment to prove self-healing
kubectl scale deploy podinfo -n podinfo-sync-demo-06 --replicas=5

# Access the application
kubectl port-forward svc/podinfo -n podinfo-sync-demo-06 9898:9898

# Verify self-heal is enabled in live Application spec
kubectl get app podinfo-sync-demo -n argocd -o yaml | grep -A5 "automated:"

# Check ArgoCD last sync revision
argocd app get podinfo-sync-demo --output json | jq '.status.operationState.syncResult.revision'
```

---

## Lessons Learned

**1. Automated sync is not unconditional — your CI gate must come first**
Enabling automated sync means every merged commit goes straight to the cluster.
This is only safe if your CI pipeline (lint, schema validation, image scanning)
runs before merge and catches bad changes. Automated sync is the reward for a
mature CI process, not a replacement for it.

**2. Pruning requires a sync — design your change triggers accordingly**
If you delete a resource from Git but no other change triggers a sync, the orphan
stays. With automated sync enabled, every push triggers a sync, so pruning runs
reliably. With manual sync, you must remember to tick the Prune checkbox or the
orphan persists silently.

**3. Self-heal and automated sync watch different things — know which you need**
A common mistake is enabling automated sync and expecting it to revert `kubectl`
changes. It will not. Automated sync watches Git. Self-heal watches the cluster.
For a fully closed GitOps loop you need both.

**4. Always use `argocd app diff` before a manual sync**
Before clicking Sync (especially with prune enabled), run `argocd app diff` to see
exactly what will be applied and what will be deleted. This is your last inspection
window before changes hit the cluster.

**5. Self-heal disable pattern is a feature, not a workaround**
Temporarily setting `selfHeal: false` via a commit and `kubectl apply` is the
intended pattern for creating a debugging window. It keeps the configuration in
Git (auditable), limits the blast radius to your debugging window, and re-enabling
is a single commit + apply.

---

## What's Next

**Demo-07: Helm Charts with ArgoCD**
Deploy applications using Helm charts as the source. Understand how ArgoCD treats
Helm as a template engine (not a release manager), override chart values inline,
and use value files for environment-specific configuration.