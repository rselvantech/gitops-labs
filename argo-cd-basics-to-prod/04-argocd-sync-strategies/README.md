# Demo-04: Production GitOps — Private Repos, Secrets & Advanced Sync

## Demo Overview

This demo covers how ArgoCD is used in **production environments** where almost everything is private by default. We connect ArgoCD to a private Git repository, configure Kubernetes to pull images from a private container registry, and handle credentials and secrets in a GitOps-aligned way. Once the production setup is in place, we extend the same demo to explore ArgoCD's advanced sync features — Automated Sync, Pruning, and Self-Healing.

By the end of this demo you will have a clear end-to-end understanding of how ArgoCD operates securely and predictably in real projects — not just demos.

> **Prerequisite:** Complete `README-podinfo-setup.md` first. This README assumes your three repos exist, your podinfo image is in Docker Hub, and you are familiar with podinfo's UI and endpoints.

**What you'll learn:**
- How to authenticate ArgoCD with a private GitHub repository
- How PATs, SSH keys, and Deploy Keys differ — and when to use each
- How container runtimes pull private images using `docker-registry` secrets
- The secrets security hierarchy — from beginner mistakes to production best practices
- How to configure ArgoCD repo access via the ArgoCD CLI
- How Automated Sync, Pruning, and Self-Healing work and when to use each
- What the ArgoCD reconciliation interval is and how to verify it

**What you'll do:**
- Generate a GitHub fine-grained Personal Access Token (PAT)
- Push manifests to `podinfo-config` and ArgoCD CRD to `argocd-config`
- Configure ArgoCD to access your private `podinfo-config` repo via CLI
- Apply the ArgoCD Application CRD and perform a manual sync
- Enable Automated Sync and observe ArgoCD react to a Git change
- Enable Pruning and observe ArgoCD delete a resource removed from Git
- Enable Self-Healing and observe ArgoCD revert a manual `kubectl edit`

## Prerequisites

- ✅ Completed `README-podinfo-setup.md` — three repos exist, podinfo image in Docker Hub
- ✅ `rselvantech/podinfo-config` private repo exists and is empty
- ✅ `rselvantech/argocd-config` private repo exists and is empty
- ✅ Docker image `rselvantech/podinfo:v1.0.0` pushed to private Docker Hub
- ✅ ArgoCD running on minikube with port-forwarding active on `localhost:8080`
- ✅ ArgoCD CLI (`argocd`) installed

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

### 3. ArgoCD CLI is installed
```bash
argocd version --client
```

**Expected:**
```
argocd: v3.x.x
```

### 4. Docker image exists in Docker Hub
```bash
docker pull rselvantech/podinfo:v1.0.0
```

**Expected:** Image pulled successfully (or `Already up to date`).

---

## Demo Objectives

By the end of this demo, you will:

1. ✅ Understand the difference between PATs, SSH keys, and Deploy Keys — and when to use each
2. ✅ Understand the secrets security hierarchy from beginner to production-grade
3. ✅ Know how `docker-registry` secrets wire to `imagePullSecrets` in a Deployment
4. ✅ Know how to configure ArgoCD private repo access via CLI
5. ✅ Know what the ArgoCD reconciliation interval is and where it is configured
6. ✅ Have a working ArgoCD application synced from a private Git repo using a private image
7. ✅ Understand Automated Sync, Pruning, and Self-Healing — what each does, when to use it, and when not to

---

## Concepts

### The Two-Repository Pattern in Practice

ArgoCD only needs access to your **config repository** — never your application source code repository. This is intentional:

```
rselvantech/podinfo          ← source code (ArgoCD never reads this)
rselvantech/podinfo-config   ← manifests   (ArgoCD reads THIS)
rselvantech/argocd-config    ← ArgoCD CRDs (ArgoCD reads THIS)
```

Separating these concerns means:
- Your source code repo's credentials are never shared with ArgoCD
- A compromised ArgoCD credential gives read access to manifests only — not to application logic or CI secrets
- Your CI system (Jenkins, GitHub Actions) and ArgoCD have completely independent access scopes

---

### GitHub Access Credentials — PAT vs SSH Keys vs Deploy Keys

Understanding which credential type to use and when is a production-critical skill.

| Credential Type | Tied To | Scope | Best Used By |
|---|---|---|---|
| **Personal Access Token (PAT)** | A user account | One or many repos | Developers, DevOps engineers (humans) |
| **SSH Key** | A user account | All repos the user can access | Developers, DevOps engineers (humans) |
| **Fine-grained PAT** | A user account | Specific repos + specific permissions | Tighter human access |
| **Deploy Key** | A repository | One repo only (1:1) | GitOps tools (ArgoCD), CI systems |
| **GitHub App** | An organisation | Many repos with fine-grained control | Enterprise CI systems |

**The rule for ArgoCD — always use Deploy Keys in production:**

Deploy Keys have a 1:1 relationship with a repository — one deploy key can only access one repository. This is exactly what ArgoCD needs: read-only access to your `podinfo-config` repo and nothing else.

```
ArgoCD → Deploy Key A → podinfo-config repo only  ✅
ArgoCD → PAT          → all repos the user owns   ❌ (over-privileged)
```

If ArgoCD manages 10 applications across 10 repositories, you create 10 deploy keys — one per repository. Each is independently revocable without affecting the others.

> **In this demo:** We use a fine-grained PAT for simplicity — it is a good learning step. Deploy Keys are introduced in a later demo. The concepts of least-privilege and credential separation apply equally to both.

**Principle of Least Privilege applied to ArgoCD:**
- ArgoCD reads Git — it needs **read-only** access
- Developers push code — they need **read-write** access
- These must be separate credentials. Do not give ArgoCD write access because it is convenient.

---

### Secrets Security Hierarchy

When dealing with Kubernetes secrets and image pull credentials, there is a clear hierarchy from least to most secure. Knowing where you are in this hierarchy is what separates a production engineer from a beginner.

```
Level 1 — Beginner (avoid in production)
─────────────────────────────────────────
Use Docker Hub username + password directly in the secret
Risk: Password exposure, no rotation, no scope restriction

Level 2 — Intermediate (acceptable for learning)
─────────────────────────────────────────────────
Use a Docker Hub Access Token (scoped, revocable)
Risk: Token still stored in Kubernetes secret as base64 (not encrypted)
      → anyone with `kubectl get secret` access can decode it

Level 3 — Good (recommended minimum for production)
────────────────────────────────────────────────────
Use an External Secrets Manager:
  - HashiCorp Vault
  - AWS Secrets Manager
  - GCP Secret Manager
  - Azure Key Vault
Your Kubernetes manifests reference the secret path, not the value.
Secret value is fetched at runtime. Never stored in Git or etcd plaintext.

Level 4 — Best (cloud-native, no secrets at all)
─────────────────────────────────────────────────
Use IAM Roles (cloud-provider native):
  - AWS: IRSA (IAM Roles for Service Accounts) → ECR access
  - GCP: Workload Identity → Artifact Registry access
  - Azure: Managed Identity → ACR access
No secret to create, rotate, or manage. The cloud provider handles auth.
Only applicable when your image registry is in the same cloud provider.
```

> **In this demo:** We use Level 2 (Docker Hub Access Token). This is appropriate for learning. The architecture for Levels 3 and 4 will be covered in the security-focused demo.

**The RBAC consideration for secrets:**

```bash
# Anyone with this access can decode your token
kubectl get secret dockerhub-secret -n podinfo -o yaml | \
  grep .dockerconfigjson | awk '{print $2}' | base64 -d
```

A developer who can create Deployments referencing a secret does **not** need `get` access to the secret itself. The container runtime uses the secret — not the deployment YAML author. Keep RBAC tight:

- Developers: can create `Deployment` resources ✅
- Developers: cannot `get` or `list` secrets ❌ (they should not need to)
- ArgoCD service account: can apply manifests ✅
- ArgoCD service account: cannot `get` secrets ❌ (it only applies the YAML reference)

---

### `imagePullSecrets` — How It Wires Together

```
Deployment manifest
  └── spec.imagePullSecrets[].name: "dockerhub-secret"
                                          │
                                          ▼
                              Kubernetes Secret
                              name: dockerhub-secret
                              type: kubernetes.io/dockerconfigjson
                              data: .dockerconfigjson: <base64>
                                          │
                                          ▼
                              Container Runtime (containerd/docker)
                              uses credentials to authenticate with
                              Docker Hub before pulling the image
```

The `imagePullSecrets` field in the Deployment spec tells the kubelet which secret to use when pulling the container image. Without it, the pull will fail with `ImagePullBackOff` for any private image.

---

### ArgoCD Reconciliation Interval

By default, ArgoCD checks your Git repository for changes **every 3 minutes (180 seconds)**. This is defined in the `argocd-cm` ConfigMap:

```bash
kubectl describe cm argocd-cm -n argocd | grep -A 2 timeout.reconciliation
```

**Expected:**
```
timeout.reconciliation:
----
180s
```

This means if you push a change to Git, ArgoCD will detect it and sync (if auto-sync is enabled) within 3 minutes — without you doing anything. If you click **Refresh** in the UI, ArgoCD checks immediately without waiting for the timer.

**Hard Refresh vs Refresh:**

| Action | What happens |
|---|---|
| `Refresh` | Re-checks Git and cluster state immediately (ignores the 3-min timer) |
| `Hard Refresh` | Discards all cached state and recalculates everything from scratch |

Use Hard Refresh when you suspect a caching issue and the UI is showing stale state.

---

### ArgoCD Advanced Sync Features

These three features build on each other. You must understand the dependency:

```
Automated Sync    ← must be enabled first
      │
      ├── Pruning        ← requires Automated Sync to be ON
      └── Self-Healing   ← requires Automated Sync to be ON
```

**Automated Sync**

| | |
|---|---|
| **Purpose** | Automatically apply Git changes to the cluster without manual click |
| **Default** | Disabled — manual sync required |
| **When to use** | Stable GitOps workflows with strong PR governance and code review |
| **When NOT to use** | When your Git access is not governed or PR process is immature |
| **Key principle** | Git must always hold the correct desired state for this to be safe |

**Pruning**

| | |
|---|---|
| **Purpose** | Delete cluster resources that have been removed from Git |
| **Default** | Disabled — removed resources are left in cluster |
| **When to use** | When Git is authoritative for the full lifecycle of resources |
| **When NOT to use** | Early stages where Git changes may be incomplete or experimental |
| **Example** | You replace a `ClusterIP` Service with an `Ingress` — old Service is deleted from Git → pruning removes it from cluster |

**Self-Healing**

| | |
|---|---|
| **Purpose** | Revert manual `kubectl` changes to match Git state |
| **Default** | Disabled |
| **When to use** | Production environments where manual cluster changes are prohibited |
| **When NOT to use** | Environments where operators legitimately need to make temporary manual changes |
| **Example** | `kubectl edit deployment` changes replicas from 3 to 1 → self-healing restores to 3 |

---

### Architecture: Full Production GitOps Flow

```
Developer                          DevOps Engineer
pushes code                        updates deployment manifest
     │                                      │
     ▼                                      ▼
rselvantech/podinfo            rselvantech/podinfo-config
(CI builds image → Docker Hub)  (ArgoCD reads this)
                                            │
                         ┌──────────────────┘
                         │
                         ▼
              ┌──────────────────────────────────────────────┐
              │  ArgoCD (namespace: argocd)                  │
              │                                              │
              │  Application CR: podinfo                     │
              │    source → rselvantech/podinfo-config       │
              │    destination → podinfo namespace           │
              │                                              │
              │  Application Controller                      │
              │    ↻ checks Git every 180s                   │
              │    ↻ auto-syncs if enabled                   │
              │    ↻ prunes removed resources if enabled     │
              │    ↻ self-heals manual changes if enabled    │
              └───────────────────────┬──────────────────────┘
                                      │
                                      ▼
              ┌──────────────────────────────────────────────┐
              │  Kubernetes Cluster                          │
              │  namespace: podinfo                          │
              │                                              │
              │  Secret: dockerhub-secret                    │
              │  Deployment: podinfo (imagePullSecrets ↑)    │
              │    └── ReplicaSet                            │
              │          └── Pod → pulls rselvantech/podinfo │
              │  Service: podinfo (ClusterIP, port 9898)     │
              └──────────────────────────────────────────────┘
                                      ↑
                              Docker Hub
                              rselvantech/podinfo:v1.0.0
                              (private — pulled via secret)
```

---

## Directory Structure

```
04-argocd-sync-strategies/
├── README-podinfo-setup.md          ← Part 1: Podinfo setup + 3-repo strategy
├── README.md                        ← This file (Part 2: GitOps workflow)
├── images/
└── src/
    ├── podinfo-config/              ← pushed to rselvantech/podinfo-config
    │   ├── deployment.yaml
    │   └── service.yaml
    └── argocd-config/               ← pushed to rselvantech/argocd-config
        └── podinfo-app.yaml
```

---

## Part A: Production Setup (Pre-GitOps + GitOps Foundation)

### Step 1: Generate GitHub Fine-Grained Personal Access Token

A fine-grained PAT lets you restrict access to specific repositories and specific permissions — unlike classic PATs which grant broad access.

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
2. Click **Generate new token**
3. Token name: `argocd-demo`
4. Expiration: 30 days
5. Resource owner: `rselvantech`
6. Repository access: **Only select repositories** → select `podinfo-config` and `argocd-config`
7. Permissions → **Contents**: `Read and write`
8. Click **Generate token**
9. **Copy the token immediately** — it is shown only once

![alt text](images/image.png)

Save it securely in a local credentials file (never commit this file to Git):
```bash
# Create a local creds file — gitignored, never committed
cat > ~/gitops-creds.txt << 'EOF'
# GitHub PAT
GITHUB_PAT=<paste-your-token-here>

# Docker Hub Token (from podinfo setup)
DOCKERHUB_TOKEN=<paste-your-token-here>
EOF
```

> **Production note:** In this demo, a single PAT is used for both pushing to Git (as a DevOps engineer) and for ArgoCD's read access. In production, ArgoCD would use a separate Deploy Key with read-only access — never a shared user credential. Deploy Keys are covered in a later demo.

---

### Step 2: Push Manifests to `podinfo-config`

```bash
# Navigate to your gitops-labs src directory
cd gitops-labs/argo-cd-basics-to-prod/04-argocd-sync-strategies/src/podinfo-config

# Initialise as a git repo
git init
git add .
git commit -m "initial: add podinfo deployment and service manifests"
git branch -M main

# Add the remote with credentials embedded in URL
# Format: https://<username>:<token>@github.com/<org>/<repo>.git
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo-config.git

git push origin main
```

**Verify in GitHub:** Go to `github.com/rselvantech/podinfo-config` — you should see `deployment.yaml` and `service.yaml`.

---

### Step 3: Push ArgoCD Application CRD to `argocd-config`

First, create the Application CRD manifest in `src/argocd-config/podinfo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo
  namespace: argocd                    # must be the ArgoCD namespace
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: .                            # manifests are at root of config repo
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo
  syncPolicy:
    syncOptions:
      - CreateNamespace=true           # ArgoCD creates the namespace if it doesn't exist
```

> **`syncOptions: CreateNamespace=true`** — In Demo-03 we manually ran `kubectl create ns`. This sync option tells ArgoCD to create the destination namespace automatically if it does not exist. This is the production-aligned approach — your namespace becomes part of the GitOps-managed desired state.

```bash
cd gitops-labs/argo-cd-basics-to-prod/04-argocd-sync-strategies/src/argocd-config

git init
git add .
git commit -m  
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git push origin main
```

**Verify in GitHub:** Go to `github.com/rselvantech/argocd-config` — you should see `podinfo-app.yaml`.

---

### Step 4: Create Namespace and Docker Registry Secret

**Note:** Skip this step , if done already in Part 1

```bash
# Create namespace (if not using CreateNamespace=true yet)
kubectl create ns podinfo

# Create docker-registry secret for pulling private image
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password=<DOCKERHUB_TOKEN> \
  --docker-email=<your-email> \
  --namespace=podinfo
```

**Verify secret was created:**
```bash
kubectl get secret dockerhub-secret -n podinfo
```

**Expected:**
```
NAME               TYPE                             DATA   AGE
dockerhub-secret   kubernetes.io/dockerconfigjson   1      5s
```

> **Why create the secret imperatively and not via Git?** Secrets must never be committed to Git — even to a private repository. Credentials should never be version-controlled. The declarative, production-grade approach is to use an External Secrets Operator pulling from HashiCorp Vault or AWS Secrets Manager. That is covered in the secrets management demo. For now, imperative creation is intentional and correct for this stage.

---

### Step 5: Configure ArgoCD to Access the Private `podinfo-config` Repo

ArgoCD cannot read your private repository without credentials. You configure this via the ArgoCD CLI.

**First, log in to your ArgoCD instance:**
```bash
argocd login localhost:8080 --username admin --password <argocd-admin-password> --insecure
```

**Add the private repository:**
```bash
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password <GITHUB_PAT> \
  --name podinfo-config
```

**Verify the repo was added and connection is successful:**
```bash
argocd repo list
```

**Expected:**
```
TYPE  NAME            REPO                                                    INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE
git   podinfo-config  https://github.com/rselvantech/podinfo-config.git       false     false  false  true   Successful
```

**Also verify in ArgoCD UI:**
- Go to **Settings** → **Repositories**
- You should see `podinfo-config` with `CONNECTION STATUS: Successful`

> You can also add repositories via the UI: **Settings → Repositories → Connect Repo**. The CLI approach is shown here because it is scriptable and infrastructure-as-code friendly — you can add it to your bootstrap scripts.

---

### Step 6: Apply the ArgoCD Application CRD

```bash
kubectl apply -f src/argocd-config/podinfo-app.yaml
```

**Expected:**
```
application.argoproj.io/podinfo created
```

**Verify the application resource was created:**
```bash
kubectl get app -n argocd
```

**Expected:**
```
NAME      SYNC STATUS   HEALTH STATUS
podinfo   OutOfSync     Missing
```

**Check the ArgoCD UI:**
- Go to `http://localhost:8080`
- You should see the `podinfo` application card showing `OutOfSync` and `Missing`

> `OutOfSync` here means ArgoCD can now read your private repo (credentials configured in Step 5) and has determined that the desired state from Git is not yet applied to the cluster. This is correct before the first sync.

---

### Step 7: Manual Sync — Verify the Full Pipeline

```bash
# Sync via CLI
argocd app sync podinfo

# Or use the ArgoCD UI: click Sync → Synchronize
```

**Watch resources appear:**
```bash
kubectl get pods -n podinfo -w
```

**Expected:**
```
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**Verify full deployment:**
```bash
kubectl get all -n podinfo
```

**Expected:**
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          1m

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/podinfo   ClusterIP   10.96.xxx.xxx   <none>        9898/TCP,9797/TCP   1m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/podinfo   1/1     1            1           1m
```

**ArgoCD UI expected state:**
```
SYNC STATUS:    Synced   ✅
HEALTH STATUS:  Healthy  ✅
```

**Access the podinfo UI:**
```bash
kubectl port-forward svc/podinfo -n podinfo 9898:9898
```

Open `http://localhost:9898` — podinfo UI should be visible with your custom message and colour.

---

### Step 8: Verify the Reconciliation Interval

```bash
kubectl describe cm argocd-cm -n argocd | grep -A 2 timeout.reconciliation
```

**Expected:**
```
timeout.reconciliation: 180s
```

This confirms the default 3-minute check interval. Every 180 seconds, the ArgoCD Application Controller checks your Git repository for changes. If auto-sync is enabled, detected changes are applied automatically. If not, the status changes to `OutOfSync` and waits for manual sync.

---

## Part B: Advanced Sync Features

### Step 9: Enable Automated Sync

Update `src/argocd-config/podinfo-app.yaml` — add the `automated` block under `syncPolicy`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo
  syncPolicy:
    automated: {}                      # ← enables automated sync
    syncOptions:
      - CreateNamespace=true
```

Push this change to `argocd-config`:
```bash
cd src/argocd-config
git add podinfo-app.yaml
git commit -m "feat: enable automated sync"
git push origin main
```

Apply the updated Application CRD:
```bash
kubectl apply -f src/argocd-config/podinfo-app.yaml
```

**Verify automated sync is enabled in ArgoCD UI:**
- Click on the `podinfo` application → **Details**
- Scroll to **SYNC POLICY** — `Auto sync` should be checked ✅

**Test automated sync — change replica count in `podinfo-config`:**

Update `src/podinfo-config/deployment.yaml` — change `replicas: 1` to `replicas: 2`:

```yaml
spec:
  replicas: 2    # ← changed from 1
```

Push to `podinfo-config`:
```bash
cd src/podinfo-config
git add deployment.yaml
git commit -m "feat: scale podinfo to 2 replicas"
git push origin main
```

**Click Refresh in ArgoCD UI** — ArgoCD immediately detects the change and creates a second pod:

```bash
kubectl get pods -n podinfo
```

**Expected:**
```
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-xxxxxxxxxx-aaaaa   1/1     Running   0          5m
podinfo-xxxxxxxxxx-bbbbb   1/1     Running   0          15s   ← new pod
```

> Without clicking Refresh, ArgoCD would detect this change within 3 minutes (the reconciliation interval). Refresh bypasses the timer and triggers an immediate check.

---

### Step 10: Enable Pruning

Pruning automatically deletes cluster resources that have been removed from your Git repository. Without pruning, deleted manifests leave orphaned resources in your cluster.

Update `src/argocd-config/podinfo-app.yaml`:

```yaml
syncPolicy:
  automated:
    prune: true                        # ← enables pruning
  syncOptions:
    - CreateNamespace=true
```

Push and apply:
```bash
cd src/argocd-config
git add podinfo-app.yaml
git commit -m "feat: enable pruning"
git push origin main
kubectl apply -f src/argocd-config/podinfo-app.yaml
```

**Verify automated sync is enabled in ArgoCD UI:**
- Click on the `podinfo` application → **Details**
- Scroll to **SYNC POLICY** — `AUTO-SYNC`and `PRUNE RESOURCES` should be checked ✅

**Test pruning — delete the Service from `podinfo-config`:**

```bash
cd src/podinfo-config
rm service.yaml
git add .
git commit -m "refactor: remove ClusterIP service (will be replaced by Ingress)"
git push origin main
```

Click **Refresh** in ArgoCD UI — observe the `Service` resource disappear from the resource tree:

![alt text](images/image-2.png)


```bash
kubectl get svc -n podinfo
```

**Expected:**
```
No resources found in podinfo namespace.
```

The Service is deleted from the cluster because it was removed from Git — exactly the desired behaviour when migrating from ClusterIP to Ingress.

> **Important:** Pruning only works when `automated` sync is enabled. Without automated sync, ArgoCD will mark the resource as an orphan but will not delete it.

---

### Step 11: Enable Self-Healing

Self-healing automatically reverts manual changes made directly to the cluster — enforcing Git as the single source of truth.

Update `src/argocd-config/podinfo-app.yaml`:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true                     # ← enables self-healing
  syncOptions:
    - CreateNamespace=true
```

**Push to Git first — then apply:**

```bash
cd src/argocd-config
git add podinfo-app.yaml
git commit -m "feat: enable self-healing"
git push origin main
```

> **Why push before applying?** Because automated sync is enabled. If you `kubectl apply` without pushing first, ArgoCD will immediately revert your change back to what Git says — which would be the version without `selfHeal: true`. Always push to Git first when automated sync is on.

```bash
kubectl apply -f src/argocd-config/podinfo-app.yaml
```

**Verify self-healing is enabled:**
- ArgoCD UI → `podinfo` app → **Details** → `SELF HEAL` also should be checked now ✅

![alt text](images/image-1.png)

**Test self-healing — manually change replicas in the cluster:**

```bash
# Direct cluster change — bypasses Git
kubectl scale deploy -n podinfo podinfo --replicas=4
# Change replicas from  2 → 4
```

Watch ArgoCD respond:
```bash
kubectl get pods -n podinfo -w
```

**Expected — ArgoCD immediately restores to 2 replicas:**
```
NAME                       READY   STATUS        RESTARTS   AGE
podinfo-bb8799d6-czp7k   1/1     Running       0          19m     ← ArgoCD restored
podinfo-bb8799d6-dbvj6   0/1     Terminating   0          3s      ← manual change detected
podinfo-bb8799d6-jgkn4   1/1     Running       0          10m     ← ArgoCD restored
podinfo-bb8799d6-z9lsf   0/1     Terminating   0          3s      ← manual change detected
```

> ArgoCD restores so quickly because it had the desired state cached — it does not need to wait for the 3-minute reconciliation timer. Self-healing operates on the cached state and detects the drift immediately.

---

## Validation Checklist

Before marking this demo complete, verify:

- [ ] `argocd repo list` shows `podinfo-config` with `STATUS: Successful`
- [ ] `kubectl get app -n argocd` shows `podinfo` with `Synced` and `Healthy`
- [ ] `kubectl get pods -n podinfo` shows 2 pods in `Running` state
- [ ] Podinfo UI accessible at `http://localhost:9898` showing correct colour and message
- [ ] `curl http://localhost:9898/healthz` returns `{"status":"ok"}`
- [ ] ArgoCD UI → podinfo app → Details shows `Auto Sync: enabled`, `Prune: enabled`, `Self Heal: enabled`
- [ ] Pushing a replica count change to `podinfo-config` triggers automatic sync within 3 minutes (or immediately on Refresh)
- [ ] Deleting `service.yaml` from `podinfo-config` removes the Service from the cluster
- [ ] Manual `kubectl edit` to change replicas is reverted by ArgoCD automatically
- [ ] `kubectl describe cm argocd-cm -n argocd | grep timeout.reconciliation` shows `180s`

---

## Cleanup

```bash
# Delete the ArgoCD Application resource
# Note: with pruning enabled, this will also delete the deployed resources
kubectl delete -f src/argocd-config/podinfo-app.yaml

# If resources remain (pruning not enabled), clean up manually
kubectl delete ns podinfo

# Remove the repo from ArgoCD
argocd repo rm https://github.com/rselvantech/podinfo-config.git

# Stop port-forwarding (Ctrl+C in respective terminals)
```

**Verify cleanup:**
```bash
kubectl get app -n argocd          # No resources found
kubectl get ns podinfo             # Error: not found
argocd repo list                   # podinfo-config removed
```

> **Do NOT delete** your GitHub repos or Docker Hub image — these are used in subsequent demos.

---

## What You Learned

In this demo, you:

- ✅ Understood the difference between PATs, SSH keys, and Deploy Keys and when each is appropriate
- ✅ Understood the secrets security hierarchy from password to IAM roles
- ✅ Understood how `docker-registry` secrets wire to `imagePullSecrets` in a Deployment
- ✅ Understood why secrets should never be committed to Git — even private repos
- ✅ Configured ArgoCD to access a private Git repository via the ArgoCD CLI
- ✅ Applied the `CreateNamespace=true` sync option — eliminating the need for manual namespace creation
- ✅ Performed a manual sync and verified the full private-image pipeline
- ✅ Understood the 3-minute reconciliation interval and where it is configured
- ✅ Enabled Automated Sync and observed ArgoCD respond to a Git push
- ✅ Enabled Pruning and observed ArgoCD delete a resource removed from Git
- ✅ Enabled Self-Healing and observed ArgoCD revert a manual `kubectl edit`

**Key Insight:**
Automated Sync + Pruning + Self-Healing together is what makes ArgoCD a true GitOps tool — not just a deployment helper. Git becomes the authoritative, enforced source of truth. Any deviation from Git — whether a missing resource, an extra resource, or a manual change — is automatically corrected. This is the shift from traditional imperative operations to declarative, continuous reconciliation.

---

## Lessons Learned

### 1. Always Use Deploy Keys for GitOps Tools in Production

PATs are tied to a user account. If that user leaves the organisation or their account is compromised, all ArgoCD access breaks. Deploy keys are tied to a repository — they can be rotated independently per repository and give only the access needed.

**Rule:** In production, ArgoCD should authenticate to each repository using a read-only Deploy Key, not a shared PAT.

### 2. Push to Git Before Applying When Auto-Sync is Enabled

When automated sync is on, `kubectl apply` on your Application CRD will be immediately reverted to match Git if the two differ. Always commit and push your Application CRD change to `argocd-config` before running `kubectl apply` — or better yet, let ArgoCD apply it automatically from Git once the App-of-Apps pattern is in place.

**Rule:** With auto-sync enabled — Git first, then cluster. Always.

### 3. Pruning and Self-Healing Require Automated Sync

These two features are additive — they only function when `automated` sync is enabled. Enabling `prune: true` without `automated` has no effect. This dependency is intentional: ArgoCD will only delete or revert things automatically if it is already trusted to apply things automatically.

**Rule:** Enable automated sync first. Only then add pruning and self-healing.

### 4. `CreateNamespace=true` Is the Production-Aligned Approach

Manually creating namespaces with `kubectl create ns` before each sync is error-prone and not GitOps. Using `syncOptions: [CreateNamespace=true]` means the namespace is part of ArgoCD's reconciled desired state — it is created if missing and the sync never fails due to a missing namespace.

**Rule:** Always use `CreateNamespace=true` in your Application CRD `syncOptions` unless you have a specific reason to pre-create namespaces separately.

### 5. Self-Healing Uses Cached State — No Timer Needed

ArgoCD detects and reverts manual cluster changes almost immediately — not after the 3-minute reconciliation interval. This is because the Application Controller caches the desired state and watches the cluster in real time. The 3-minute interval is for checking Git for new changes, not for watching the cluster.

**Rule:** Do not rely on the reconciliation timer for self-healing response time. It is near-instant.

### 6. Secrets Should Never Be Version-Controlled

Even in a private repository, secrets must not be committed to Git. Base64 encoding is not encryption. Anyone with repo access can decode it. The correct approach is external secrets management (Vault, AWS Secrets Manager) or cloud-native IAM roles. For this demo, imperative `kubectl create secret` is intentional — it keeps the secret out of Git entirely.

**Rule:** `kubectl create secret` is the right approach until you implement an External Secrets Operator. Never `kubectl create secret -o yaml > secret.yaml && git add secret.yaml`.

---

## Next Steps

**Demo-05: Kustomize with ArgoCD**
- Use the `kustomize/` directory from your podinfo fork
- Deploy podinfo to `dev`, `staging`, and `prod` namespaces with different configs
- See `PODINFO_UI_COLOR` visually differentiate environments
- Understand how Kustomize base + overlays integrates with ArgoCD

**Explore on your own (optional):**
- Run `argocd app diff podinfo` to see what ArgoCD sees as the difference between Git and cluster
- Run `argocd app history podinfo` to view the full sync history
- Try enabling pruning without automated sync and confirm it has no effect
- Try `kubectl scale deployment podinfo --replicas=5 -n podinfo` with self-healing enabled — observe how fast ArgoCD reverts it

---

## Troubleshooting

**`ImagePullBackOff` on pods:**
```bash
kubectl describe pod -n podinfo | grep -A5 Events
# Look for: "Failed to pull image" or "unauthorized"

# Verify secret exists in correct namespace
kubectl get secret dockerhub-secret -n podinfo

# Verify imagePullSecrets is set in deployment
kubectl get deploy podinfo -n podinfo -o yaml | grep -A3 imagePullSecrets
```

**ArgoCD shows "authentication required" or "failed to load target state":**
```bash
# Verify repo credentials are configured
argocd repo list

# Re-add if missing
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

**Auto-sync not triggering after Git push:**
```bash
# Verify auto-sync is enabled
argocd app get podinfo | grep -i auto

# Check reconciliation interval
kubectl describe cm argocd-cm -n argocd | grep timeout

# Force immediate check
argocd app refresh podinfo
```

**Self-healing not working after `kubectl edit`:**
```bash
# Verify self-healing is enabled in the Application CRD
kubectl get app podinfo -n argocd -o yaml | grep -A5 syncPolicy

# Self-healing requires automated sync to also be enabled
# Check: automated.selfHeal: true AND automated: {} both present
```

**Pruning not removing deleted resource:**
```bash
# Verify pruning is enabled
kubectl get app podinfo -n argocd -o yaml | grep prune

# Verify automated sync is also enabled (pruning requires it)
# Force refresh to trigger immediate check
argocd app refresh podinfo
```