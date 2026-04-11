# Demo-11: Sync Waves — Changes and Additions

Apply these changes to the existing Demo-11 README. Each section is self-contained
and labelled with where it goes.

---

## Change 1 — Prerequisites: update PAT note and repo reference

**REPLACE** the entire Prerequisites section's bullet list and verify steps with:

```markdown
## Prerequisites

- ✅ Completed Demo-10 — sync hooks and lifecycle phases understood
- ✅ ArgoCD running on minikube (default profile)
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `gitops-apps-config` and `argocd-config`
- ✅ `gitops-apps-config` already registered with ArgoCD from Demo-10 prereq

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
```text
argocd: v3.x.x
'admin:login' logged in successfully
```

### 4. `gitops-apps-config` is registered with ArgoCD
```bash
argocd repo list | grep gitops-apps-config
```

**Expected:**
```text
git   gitops-apps-config   https://github.com/rselvantech/gitops-apps-config.git   Successful
```

If not listed, it means Demo-10's `README-goals-app-setup.md` was not completed.
Register it now:
```bash
argocd repo add https://github.com/rselvantech/gitops-apps-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

### 5. No leftover namespace from a previous run
```bash
kubectl get namespace sync-waves-demo 2>&1
```

**Expected:** `Error from server (NotFound)` — namespace does not exist.

If it exists from a previous run:
```bash
kubectl delete namespace sync-waves-demo
```
```

---

## Change 2 — Folder Structure: update `app-config` to `gitops-apps-config`

**REPLACE** the entire Folder Structure section with:

```markdown
## Folder Structure

```
11-sync-waves/src/
├── gitops-apps-config/      ← git init → remote: rselvantech/gitops-apps-config
│   └── demo-11-sync-waves-nginx-app/
│       ├── namespace.yaml
│       ├── resourcequota.yaml
│       ├── limitrange.yaml
│       ├── serviceaccount.yaml
│       ├── secret.yaml
│       ├── configmap.yaml
│       ├── networkpolicy.yaml
│       ├── service.yaml
│       ├── deployment.yaml
│       └── poddisruptionbudget.yaml    ← added in Step 6
└── argocd-config/           ← git init → remote: rselvantech/argocd-config
    └── demo-11-sync-waves/
        └── sync-waves-app.yaml
```

**What happens on GitHub after all pushes:**

```
rselvantech/gitops-apps-config (GitHub) — existing repo from Demo-10
├── demo-10-sync-hooks-goals-app/       ← Demo-10 (untouched)
└── demo-11-sync-waves-nginx-app/       ← Demo-11 adds this
    ├── namespace.yaml          ← wave -1
    ├── resourcequota.yaml      ← wave -1
    ├── limitrange.yaml         ← wave -1
    ├── serviceaccount.yaml     ← wave -1
    ├── secret.yaml             ← wave -1
    ├── configmap.yaml          ← wave  0
    ├── networkpolicy.yaml      ← wave  0
    ├── service.yaml            ← wave  1
    └── deployment.yaml         ← wave  5

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml            ← Demo-05 (untouched)
├── demo-06-sync-pruning/       ← Demo-06 (untouched)
├── demo-09-argocd-projects/    ← Demo-09 (untouched)
├── demo-10-sync-hooks/         ← Demo-10 (untouched)
└── demo-11-sync-waves/         ← Demo-11 adds this
    └── sync-waves-app.yaml
```

> `gitops-apps-config` is already registered with ArgoCD from Demo-10. No
> new repo registration needed. `argocd-config` is applied manually with
> `kubectl apply` and does not need ArgoCD registration.
```

---

## Change 3 — Step 1: replace entirely

**REPLACE** the entire Step 1 section with:

```markdown
## Step 1: Setup

### Step 1a: Add `gitops-apps-config` to PAT (if not already there)

From Demo-10 onwards, all application manifests live in `gitops-apps-config`.
This repo was created and registered with ArgoCD in Demo-10's
`README-goals-app-setup.md`.

> **PAT check:** Your PAT from Demo-09 covers `podinfo-config`, `argocd-config`,
> and `argocd-platform-config`. Demo-10 added `gitops-apps-config`. If you
> completed Demo-10's prerequisite README, `gitops-apps-config` is already in
> your PAT. If not, edit your PAT on GitHub and add `gitops-apps-config` with
> `Contents: Read-only` permission.

**Verify your PAT covers `gitops-apps-config`:**
```bash
argocd repo list | grep gitops-apps-config
```

**Expected:** `Successful` status. If missing, see Prerequisite step 4 above.

### Step 1b: Initialise local repo structure

```bash
cd gitops-labs/argo-cd-basics-to-prod/11-sync-waves/src

# gitops-apps-config — already exists from Demo-10
mkdir -p gitops-apps-config && cd gitops-apps-config
git init
git branch -M main
git remote add origin \
  https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main

cd ..

# argocd-config — existing repo from prior demos
mkdir -p argocd-config && cd argocd-config
git init
git branch -M main
git remote add origin \
  https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```
```

---

## Change 4 — Step 2: add directory creation commands at the top

**INSERT** immediately after the Step 2 heading and before the first manifest YAML:

```markdown
**Create the directory and all manifest files:**
```bash
cd gitops-labs/argo-cd-basics-to-prod/11-sync-waves/src/gitops-apps-config
mkdir -p demo-11-sync-waves-nginx-app
touch demo-11-sync-waves-nginx-app/namespace.yaml
touch demo-11-sync-waves-nginx-app/resourcequota.yaml
touch demo-11-sync-waves-nginx-app/limitrange.yaml
touch demo-11-sync-waves-nginx-app/serviceaccount.yaml
touch demo-11-sync-waves-nginx-app/secret.yaml
touch demo-11-sync-waves-nginx-app/configmap.yaml
touch demo-11-sync-waves-nginx-app/networkpolicy.yaml
touch demo-11-sync-waves-nginx-app/service.yaml
touch demo-11-sync-waves-nginx-app/deployment.yaml
```

Fill each file with the content below.
```

**Also update all YAML file paths** — replace every occurrence of
`demo-11-sync-waves/` with `demo-11-sync-waves-nginx-app/` throughout Step 2.

**Also update the git push command at the end of Step 2:**
```bash
git add demo-11-sync-waves-nginx-app/
git commit -m "feat: add demo-11 sync waves nginx manifests with wave annotations"
git push origin main
```

---

## Change 5 — Step 3: update Application CRD

**REPLACE** `demo-11-sync-waves/sync-waves-app.yaml` content with:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sync-waves-demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    path: demo-11-sync-waves-nginx-app
  destination:
    server: https://kubernetes.default.svc
    namespace: sync-waves-demo
```

> **`targetRevision: main`** — explicit branch name from Demo-10 onwards.
> `HEAD` is not used in this course.

---

## Change 6 — Step 5: replace unreliable NetworkPolicy Events check with proper wave ordering proof

**REMOVE** this entire block from Step 5:
```
**Verify NetworkPolicy existed before Deployment pod started:**
```bash
kubectl describe deployment nginx-app -n sync-waves-demo | grep -A5 Events
```

**Expected — Deployment created after NetworkPolicy was already in place:**
```text
Events:
  Normal  ScalingReplicaSet  Scaled up replica set nginx-app-xxx to 1
```
```

**REPLACE with:**

```markdown
**Prove wave ordering — verify resources exist before later waves started:**

The most reliable proof is checking the `creationTimestamp` of resources across
waves. Wave -1 resources must have an earlier timestamp than wave 5 resources.

```bash
# Show creation timestamps for key resources across all waves
kubectl get namespace,resourcequota,configmap,networkpolicy,service,deployment \
  -n sync-waves-demo \
  -o custom-columns=\
"KIND:.kind,NAME:.metadata.name,CREATED:.metadata.creationTimestamp,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave" \
  2>/dev/null | sort -k4
```

**Expected — timestamps increase as wave number increases:**
```text
KIND            NAME                   CREATED                WAVE
Namespace       sync-waves-demo        2026-04-08T10:00:00Z   -1
ResourceQuota   sync-waves-quota       2026-04-08T10:00:00Z   -1
ConfigMap       nginx-config           2026-04-08T10:00:02Z   0
NetworkPolicy   nginx-network-policy   2026-04-08T10:00:02Z   0
Service         nginx-service          2026-04-08T10:00:04Z   1
Deployment      nginx-app              2026-04-08T10:00:06Z   5
```

The timestamps show wave -1 resources were created ~2 seconds before wave 0,
wave 0 before wave 1, and wave 1 before wave 5 — exactly the 2-second inter-wave
delay ArgoCD enforces.

> **The ArgoCD UI is the clearest visual proof.** During sync, go to
> `http://localhost:8080` → click `sync-waves-demo-app` → **Sync** button →
> watch the resource tree. Resources appear in batches — all wave -1 together,
> then wave 0 together after a brief pause, and so on. The UI shows each
> resource's health check completing before the next wave begins.

**Also verify using ArgoCD sync operation detail:**
```bash
argocd app get sync-waves-demo-app -o json \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
resources = data.get('status', {}).get('resources', [])
for r in sorted(resources, key=lambda x: x.get('syncWave', 0)):
    print(f\"Wave {r.get('syncWave','?'):>3}  {r['kind']:<20} {r['name']}\")"
```

**Expected — resources listed in wave order:**
```text
Wave  -1  Namespace            sync-waves-demo
Wave  -1  ResourceQuota        sync-waves-quota
Wave  -1  LimitRange           sync-waves-limits
Wave  -1  ServiceAccount       nginx-service-account
Wave  -1  Secret               app-secret
Wave   0  ConfigMap            nginx-config
Wave   0  NetworkPolicy        nginx-network-policy
Wave   1  Service              nginx-service
Wave   5  Deployment           nginx-app
```
```

---

## Change 7 — Step 6: update PDB path

**UPDATE** the PDB yaml file path and git commands in Step 6:

File: `demo-11-sync-waves-nginx-app/poddisruptionbudget.yaml`

Git commands:
```bash
git add demo-11-sync-waves-nginx-app/poddisruptionbudget.yaml
git commit -m "feat: add PodDisruptionBudget at wave 3 — no renumbering needed"
git push origin main
```

---

## Change 8 — Cleanup: update repo reference

**REPLACE** the Cleanup section with:

```markdown
## Cleanup

```bash
# Delete the Application
kubectl delete app sync-waves-demo-app -n argocd

# Delete the namespace (removes all resources)
kubectl delete namespace sync-waves-demo
```

> **Do NOT deregister `gitops-apps-config`** — it is used in Demo-12, Demo-13,
> Demo-14, and all future demos. Leave it registered with ArgoCD.

**Verify:**
```bash
kubectl get namespace sync-waves-demo          # Error: not found
argocd app list | grep sync-waves-demo-app     # No output
argocd repo list | grep gitops-apps-config     # Still listed — correct
```
```

---

## Addition A — New optional section: "What the Application Does"

**ADD this as a new section after "The Application — What We Deploy"** and before the Folder Structure section.

```markdown
## Optional: What the Application Does — Full Functional Walkthrough

> **This section is optional.** Skip it if you only want to understand sync
> wave ordering. Read it if you want to understand what each manifest actually
> does and how the resources interact.

The nginx application in this demo is not just a placeholder — every resource
has a real function. Together they form a complete, production-patterned nginx
deployment that you can browse, inspect, and verify.

### The Full Architecture

```
Browser (on your laptop)
    │
    │  kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80
    ▼
nginx-service (ClusterIP, port 80)
    │  governed by: NetworkPolicy (allows ingress on port 80)
    │                ResourceQuota (limits pods in namespace)
    │                LimitRange (sets default CPU/memory)
    ▼
nginx-app pod
    │  identity: nginx-service-account (ServiceAccount)
    │  config:   nginx-config (ConfigMap → /usr/share/nginx/html/index.html)
    │  secret:   app-secret (Secret → env var DEMO_API_KEY)
    ▼
Serves: custom index.html showing the wave ordering that created this pod
```

### What Each Resource Does

**Namespace (`sync-waves-demo`)** — Isolates all demo resources. Deleted in
cleanup to remove everything at once.

**ResourceQuota** — Hard ceiling on the namespace:
```bash
kubectl describe resourcequota sync-waves-quota -n sync-waves-demo
```
```text
Resource          Used   Hard
--------          ----   ----
limits.cpu        200m   1
limits.memory     128Mi  512Mi
pods              1      2
requests.cpu      100m   500m
requests.memory   64Mi   256Mi
```
The Deployment uses 100m CPU and 64Mi memory — within quota. If you tried to
scale to 3 replicas, the third pod would be rejected by quota.

**LimitRange** — Applies default resource requests/limits to any pod that
doesn't specify them. Without this, a pod without `resources:` would have no
limits and could consume the entire node.
```bash
kubectl describe limitrange sync-waves-limits -n sync-waves-demo
```
```text
Type        Resource  Default  DefaultRequest
----        --------  -------  --------------
Container   cpu       200m     100m
Container   memory    128Mi    64Mi
```

**ServiceAccount** — The Deployment uses `nginx-service-account` instead of
the `default` service account. This follows least-privilege — the nginx pod
has only the permissions explicitly granted to this service account (none, in
this demo). In production, this is where you bind IAM roles or RBAC rules.

**Secret** — The `DEMO_API_KEY` is injected as an environment variable into
the nginx container:
```bash
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- env | grep DEMO_API_KEY
```
```text
DEMO_API_KEY=demo-key-sync-waves-2026
```
In production this would be a real API token, DB password, or certificate.

**ConfigMap** — The `index.html` is mounted directly into nginx's serving
directory. Any change to the ConfigMap triggers a sync — nginx serves the
updated page without rebuilding the image.
```bash
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- cat /usr/share/nginx/html/index.html
```
```text
<h1>ArgoCD Sync Waves — Demo-11</h1>
<p>Resources were created in this order:</p>
<ol>
  <li>Wave -1: Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret</li>
  ...
```

**NetworkPolicy** — Restricts which traffic can reach the nginx pod:

```yaml
spec:
  podSelector:
    matchLabels:
      app: nginx-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports:
        - port: 80     # Only HTTP ingress allowed
  egress:
    - {}               # All egress allowed (nginx needs no outbound restrictions)
```

> **Important — minikube limitation:** Minikube's default CNI (Container
> Network Interface) does not enforce NetworkPolicy rules. The NetworkPolicy
> object is created and ArgoCD shows it as Healthy, but traffic blocking
> is not actually enforced on minikube. This demo proves the *ordering*
> guarantee (NetworkPolicy exists before the Deployment pod starts), not
> enforcement. In production with Calico or Cilium, this policy would
> actively block traffic on any port other than 80.

**Service** — ClusterIP exposing the nginx pod on port 80. Only accessible
within the cluster — accessible from your laptop via `kubectl port-forward`.

**Deployment** — The nginx pod itself. References ServiceAccount, ConfigMap
(volume), and Secret (env var) explicitly. ArgoCD's default ordering ensures
all three are created before the Deployment — no wave annotation needed for
these explicit dependencies. Wave 5 is needed only for the *implicit*
dependencies (NetworkPolicy, ResourceQuota, LimitRange).

### Test the Running Application

```bash
kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80
```

Open `http://localhost:8081` — you should see:

```
ArgoCD Sync Waves — Demo-11

Resources were created in this order:
1. Wave -1: Namespace, ResourceQuota, LimitRange, ServiceAccount, Secret
2. Wave  0: ConfigMap, NetworkPolicy
3. Wave  1: Service
4. Wave  5: Deployment (this pod)

Governance resources (quota, network policy) existed before this pod started.
```

**Verify ResourceQuota is enforced — try to exceed it:**
```bash
kubectl scale deployment nginx-app --replicas=3 -n sync-waves-demo
```

```bash
kubectl get pods -n sync-waves-demo
```

**Expected — only 2 pods start, third is blocked by ResourceQuota (max pods: 2):**
```text
NAME                         READY   STATUS    RESTARTS
nginx-app-xxxxxxxxx-aaaaa    1/1     Running   0
nginx-app-xxxxxxxxx-bbbbb    1/1     Running   0
```

```bash
kubectl describe replicaset -n sync-waves-demo | grep -A3 "Warning\|quota"
```

**Expected — quota exceeded event:**
```text
Warning  FailedCreate  pods "nginx-app-xxxxxxxxx-ccccc" is forbidden:
         exceeded quota: sync-waves-quota, requested: pods=1, used: pods=2, limited: pods=2
```

This confirms the ResourceQuota from wave -1 is actively enforced on the
Deployment from wave 5. Scale back to 1 replica:

```bash
kubectl scale deployment nginx-app --replicas=1 -n sync-waves-demo
```
```

---

## Change 9 — Commands Reference: update paths

**REPLACE** the Commands Reference section with:

```markdown
## Commands Reference

```bash
# Apply Application CRD
kubectl apply -f demo-11-sync-waves/sync-waves-app.yaml

# Manual sync
argocd app sync sync-waves-demo-app

# Refresh without applying
argocd app get sync-waves-demo-app --refresh

# Watch resources being created
kubectl get all,resourcequota,limitrange,networkpolicy -n sync-waves-demo -w

# Check app status
argocd app get sync-waves-demo-app

# Prove wave ordering via creation timestamps
kubectl get namespace,resourcequota,configmap,networkpolicy,service,deployment \
  -n sync-waves-demo \
  -o custom-columns=\
"KIND:.kind,NAME:.metadata.name,CREATED:.metadata.creationTimestamp,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave" \
  2>/dev/null | sort -k4

# View resources with wave annotations
kubectl get all -n sync-waves-demo \
  -o custom-columns=NAME:.metadata.name,WAVE:.metadata.annotations."argocd\.argoproj\.io/sync-wave"

# Access the nginx application
kubectl port-forward svc/nginx-service -n sync-waves-demo 8081:80

# Verify Secret injection
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- env | grep DEMO_API_KEY

# Check ResourceQuota usage
kubectl describe resourcequota sync-waves-quota -n sync-waves-demo

# Check NetworkPolicy
kubectl describe networkpolicy nginx-network-policy -n sync-waves-demo

# Verify ConfigMap content served by nginx
kubectl exec -n sync-waves-demo \
  $(kubectl get pod -n sync-waves-demo -o name) \
  -- cat /usr/share/nginx/html/index.html
```
```

---

## Change 10 — What's Next: update Demo-12 reference

**REPLACE** the What's Next section with:

```markdown
## What's Next

**Demo-12: App-of-Apps Pattern**
Manage multiple ArgoCD Applications declaratively using a parent Application
that watches `argocd-config` and auto-syncs child Applications from Git.
Eliminate the manual `kubectl apply` for every Application CRD. Sync waves
from this demo carry forward — App-of-Apps uses wave annotations on child
Application CRDs to control the order in which applications are deployed.
The `gitops-apps-config` repo used in this demo continues as the source for
all application manifests in Demo-12.
```