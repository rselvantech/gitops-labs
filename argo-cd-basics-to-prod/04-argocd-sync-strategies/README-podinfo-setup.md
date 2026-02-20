# Demo-04 (Part 1): Podinfo Setup & Three-Repo Strategy

## Overview

Before diving into the ArgoCD sync strategies demo, this README covers two foundational things:

1. **What podinfo is** — understanding the demo application you will use throughout this course
2. **The three-repo strategy** — setting up your GitHub repositories the professional, production-aligned way

Completing this README gives you a clean foundation so that Demo-04 Part 2 can focus entirely on GitOps workflows without setup interruptions.

**What you'll learn:**
- What podinfo is, who created it, and why it is used in the GitOps ecosystem
- Podinfo's key endpoints, ports, and features relevant to GitOps demos
- Why production GitOps uses three separate repositories — not one or two
- What each of the three repos holds and who owns it
- How to fork podinfo, build it from source, and push it to a private Docker Hub registry
- How to deploy podinfo directly with `kubectl` (no ArgoCD yet) to verify it works
- How to explore the podinfo UI and its visual features

**What you'll do:**
- Fork `stefanprodan/podinfo` to your GitHub account
- Create three private GitHub repositories
- Build podinfo from your fork and push to your private Docker Hub
- Deploy podinfo on minikube using plain `kubectl`
- Explore the podinfo UI, health endpoints, and metrics endpoint

## Prerequisites

- ✅ Completed Demo-01 — core concepts understood
- ✅ Completed Demo-02 — ArgoCD installed
- ✅ Completed Demo-03 — ArgoCD Application CRD understood
- ✅ Docker Desktop installed and logged into Docker Hub
- ✅ GitHub account (`rselvantech`)
- ✅ `kubectl`, `git`, `docker` available in terminal

**Verify Prerequisites:**

### 1. Minikube is running
```bash
minikube status
kubectl get nodes
```

**Expected:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1d    v1.xx.x
```

### 2. Docker Hub logged in
```bash
docker login -u rselvantech
# Password: <paste your Docker Hub access token>

docker info | grep Username
```

**Expected:**
```
Username: rselvantech
```

### 3. Git is configured
```bash
git config --global user.name
git config --global user.email
```

---

## Demo Objectives

By the end of this README, you will:

1. ✅ Understand what podinfo is and why it is the standard GitOps demo application
2. ✅ Understand podinfo's ports, endpoints, and configurable features
3. ✅ Understand the three-repo strategy and why it matters in production
4. ✅ Have your fork of podinfo on GitHub
5. ✅ Have three private GitHub repositories created and ready
6. ✅ Have a podinfo image built from your fork and pushed to private Docker Hub
7. ✅ Have podinfo running on minikube and accessible via port-forwarding
8. ✅ Explored the podinfo UI and verified health and metrics endpoints

---

## Concepts

### What is Podinfo?

**Podinfo** is a tiny web application made with Go that showcases best practices of running microservices in Kubernetes. It is used by CNCF projects like Flux and Flagger for end-to-end testing and workshops.

It was created by **Stefan Prodan** — the same engineer who created **Flux CD**, one of the two reference implementations of GitOps (ArgoCD being the other). This is not a toy demo app. It is the application the GitOps community itself uses to validate pipelines and demonstrate patterns. Recognising podinfo in a project signals real-world GitOps maturity.

**Why podinfo is used throughout this course instead of a custom app:**

| Reason | Explanation |
|---|---|
| Purpose-built for GitOps | Designed specifically to demonstrate Kubernetes and GitOps patterns |
| Helm chart included | Official chart available — used in Helm + ArgoCD demos |
| Kustomize overlays included | `kustomize/` directory ships with base + overlay structure |
| Visual multi-env differentiation | `PODINFO_UI_COLOR` env var — visually distinguish dev/staging/prod |
| Built-in health endpoints | `/healthz` and `/readyz` — real liveness and readiness probe patterns |
| Prometheus metrics | `/metrics` on port `9797` — observability without instrumentation |
| Progressive delivery ready | The canonical demo app for Argo Rollouts canary and blue-green demos |
| Community recognised | Anyone in the cloud-native space immediately recognises it |

---

### Podinfo Architecture — Ports and Endpoints

Understanding podinfo's ports before deploying it avoids confusion:

```
┌─────────────────────────────────────────────────┐
│  podinfo container                              │
│                                                 │
│  Port 9898  ← HTTP (UI + API)                  │
│  Port 9797  ← Prometheus metrics               │
│  Port 9999  ← gRPC                             │
└─────────────────────────────────────────────────┘
```

**Key endpoints on port 9898:**

| Endpoint | Method | Description |
|---|---|---|
| `/` | GET | Web UI — shows version, hostname, color, message |
| `/healthz` | GET | Liveness probe — returns `{"status":"ok"}` |
| `/readyz` | GET | Readiness probe — returns `{"status":"ok"}` |
| `/version` | GET | Returns JSON with version, revision, runtime |
| `/metrics` | GET | Prometheus metrics (port `9797`) |
| `/env` | GET | Returns environment variables |
| `/configs` | GET | Returns mounted ConfigMap/Secret values |
| `/token` | POST | Issues a JWT token (auth demo) |
| `/cache/{key}` | GET/POST | Redis cache operations (when Redis enabled) |

**Key configurable environment variables:**

| Variable | Effect | Example |
|---|---|---|
| `PODINFO_UI_COLOR` | Changes UI background colour | `#blue`, `#green`, `#red` |
| `PODINFO_UI_MESSAGE` | Displays a custom message in UI | `"Production Environment"` |
| `PODINFO_UI_LOGO` | Custom logo URL | Any image URL |

**Why these matter for GitOps demos:**
- **`/healthz` and `/readyz`** — you add real `livenessProbe` and `readinessProbe` to your Deployment manifests
- **`PODINFO_UI_COLOR`** — when deploying to dev/staging/prod namespaces, each environment shows a different colour in the browser — visually proving multi-env GitOps works
- **`/version`** — instantly verify which image version is running after an image update

---

### The Three-Repo Strategy — Why It Matters

This is one of the most important architectural decisions in GitOps. Understanding it before you start setting up repos will make every subsequent demo clearer.

**The wrong way — one repo for everything:**
```
rselvantech/everything/
├── src/           ← application code
├── Dockerfile
├── k8s/           ← Kubernetes manifests
└── argocd/        ← ArgoCD Application CRDs
```

This collapses three distinct concerns into one repository. It breaks separation of duties, makes access control impossible, and prevents ArgoCD from managing itself independently.

**The professional way — three repos:**

```
GitHub:
├── rselvantech/podinfo              ← Repo 1: App source code (developer owns)
├── rselvantech/podinfo-config       ← Repo 2: K8s manifests (DevOps/platform owns)
└── rselvantech/argocd-config        ← Repo 3: ArgoCD CRDs (platform team owns)
```

**What each repo contains:**

**Repo 1 — `rselvantech/podinfo` (your fork)**
- Go source code (`main.go`, etc.)
- `Dockerfile` — builds the container image
- CI pipeline files (GitHub Actions, Jenkinsfile) — added in later demos
- **Who pushes here:** Developers
- **Does ArgoCD read this?** No — ArgoCD never needs access to application source code

**Repo 2 — `rselvantech/podinfo-config`**
- `deployment.yaml`, `service.yaml`, `configmap.yaml`
- Kustomize overlays (`base/`, `overlays/dev`, `overlays/prod`) — added in later demos
- Helm values files — added in Helm demo
- **Who pushes here:** DevOps / platform engineers
- **Does ArgoCD read this?** Yes — this is ArgoCD's source of truth for what to deploy

**Repo 3 — `rselvantech/argocd-config`**
- ArgoCD `Application` CRDs (one per application)
- `AppProject` definitions
- ArgoCD's own Helm values (for managing ArgoCD itself — App-of-Apps pattern, covered later)
- **Who pushes here:** Platform team
- **Does ArgoCD read this?** Yes — in the App-of-Apps demo, ArgoCD manages its own Application CRDs from here

**Why three and not two:**

| Concern | Two-Repo | Three-Repo |
|---|---|---|
| ArgoCD CRDs live in | app-config repo (mixed) | dedicated repo (clean) |
| ArgoCD manages itself | No — manual `kubectl apply` always | Yes — App-of-Apps pattern |
| Onboard a new app | Edit app-config repo | Edit only argocd-config repo |
| Access control | Mixed — hard to RBAC properly | Clean — each repo has its own access policy |
| Platform team separation | None | Platform team fully owns repo 3 |
| Production alignment | Learning-level | What real teams use |

**How the three repos relate to each other:**

```
Developer                DevOps Engineer              Platform Team
     │                          │                           │
     ▼                          ▼                           ▼
rselvantech/podinfo    rselvantech/podinfo-config   rselvantech/argocd-config
(source + Dockerfile)  (k8s manifests)              (ArgoCD Application CRDs)
     │                          │                           │
     │  CI builds image         │                           │
     │  pushes to Docker Hub    │ ArgoCD reads              │ ArgoCD reads
     ▼                          ▼                           ▼
Docker Hub              Kubernetes Cluster           ArgoCD (manages itself)
(private registry)      (running pods)               via App-of-Apps (later demo)
```

> **Note for this demo:** In Demo-04 Part 2, you will apply the ArgoCD `Application` CRD manually with `kubectl apply`. The `argocd-config` repo exists from day one, but ArgoCD will not yet be reading it automatically. That fully GitOps-managed state is achieved in the App-of-Apps demo.

---

### Your `gitops-labs` Repo — Where It Fits

Your `gitops-labs` repo is your **learning and documentation repo**. It is separate from all three production repos. Think of it as your course notebook — it holds READMEs and reference copies of manifests, but ArgoCD never reads from it.

```
rselvantech/gitops-labs          ← READMEs + reference src (this course)
rselvantech/podinfo              ← actual app source (ArgoCD never reads)
rselvantech/podinfo-config       ← actual manifests (ArgoCD reads this)
rselvantech/argocd-config        ← actual ArgoCD CRDs (ArgoCD reads this later)
```

---

## Directory Structure

```
04-argocd-sync-strategies/
├── README-podinfo-setup.md          ← This file
├── README.md                        ← Demo-04 Part 2 (GitOps workflow)
├── images/
└── src/
    ├── podinfo-config/              ← reference copy → goes to rselvantech/podinfo-config
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── namespace.yaml
    └── argocd-config/               ← reference copy → goes to rselvantech/argocd-config
        └── podinfo-app.yaml
```

---

## Part 1: Fork Podinfo

### Step 1: Fork `stefanprodan/podinfo` on GitHub

1. Open [https://github.com/stefanprodan/podinfo](https://github.com/stefanprodan/podinfo)
2. Click **Fork** (top right)
3. Under **Owner**, select `rselvantech`
4. Repository name: keep as `podinfo`
5. Uncheck **"Copy the master branch only"** — keep all branches
6. Click **Create fork**

Your fork is now at: `https://github.com/rselvantech/podinfo`

**Verify the fork:**
```bash
# Clone your fork locally
git clone https://github.com/rselvantech/podinfo.git
cd podinfo
ls
```

**Expected — you should see:**
```
Dockerfile   Makefile   README.md   charts/   cmd/   kustomize/   pkg/   ...
```

---

## Part 2: Create Three Private GitHub Repositories

### Step 2: Create `rselvantech/podinfo-config`

1. Go to GitHub → **New repository**
2. Repository name: `podinfo-config`
3. Visibility: **Private**
4. Description: `Kubernetes manifests for podinfo — GitOps config repo`
5. Do NOT initialise with README (you will push from local)
6. Click **Create repository**

---

### Step 3: Create `rselvantech/argocd-config`

1. Go to GitHub → **New repository**
2. Repository name: `argocd-config`
3. Visibility: **Private**
4. Description: `ArgoCD Application CRDs and AppProjects — platform config repo`
5. Do NOT initialise with README
6. Click **Create repository**

> Your fork `rselvantech/podinfo` already exists from Step 1. You now have all three repos.

**Verify all three repos exist:**

```
https://github.com/rselvantech/podinfo          ← your fork (public or private)
https://github.com/rselvantech/podinfo-config   ← private ✅
https://github.com/rselvantech/argocd-config    ← private ✅
```

---

## Part 3: Build Podinfo Image and Push to Private Docker Hub

### Step 4: Create a Private Docker Hub Repository

1. Log into [hub.docker.com](https://hub.docker.com)
2. Go to **Repositories** → **Create Repository**
3. Repository name: `podinfo`
4. Visibility: **Private**
5. Click **Create**

Your private image repository is now: `rselvantech/podinfo`

---

### Step 5: Build Podinfo from Your Fork

```bash
# Ensure you are inside your cloned fork
cd podinfo

# Build the image from the Dockerfile in the repo root
# Tag format: dockerhub-username/repo-name:version
docker build -t rselvantech/podinfo:v1.0.0 .
```

**Verify the image was built:**
```bash
docker images | grep podinfo
```

**Expected:**
```
rselvantech/podinfo   v1.0.0   abc123def456   2 minutes ago   50MB
```

> Podinfo is a Go application — the image is small (~50MB) because it uses a scratch/distroless base.

---

### Step 6: Push Image to Private Docker Hub

```bash
# Docker Desktop handles authentication if you are already logged in
# If not logged in via Docker Desktop, run: docker login

docker push rselvantech/podinfo:v1.0.0
```

**Verify in Docker Hub:**
1. Go to `hub.docker.com/r/rselvantech/podinfo`
2. You should see tag `v1.0.0` listed

---

## Part 4: Deploy Podinfo on Minikube (kubectl only — no ArgoCD)

This section deploys podinfo directly using `kubectl` — no ArgoCD involved yet. The purpose is to verify the image works, understand the manifests, and explore the podinfo UI before wiring everything through ArgoCD in Part 2.

### Step 7: Create Namespace

```bash
kubectl create ns podinfo
```

---

### Step 8: Create Docker Registry Secret

Your image is private — minikube's container runtime must authenticate with Docker Hub to pull it. This requires a `docker-registry` type Kubernetes secret.

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password=<your-dockerhub-access-token> \
  --docker-email=<your-email> \      
  --namespace=podinfo
```

**Important:** 
>- Always use `https://index.docker.io/v1/` as `--docker-server` for Docker Hub secrets regardless of what URL you use in the browser.
>- Use `Read-only` access permission while creating PAT in docker hub,it is the minimum permission required to pull `private images` 
>- Always Wrap access token in single quotes `--docker-password='dckr_pat_xxxxx-Q'`.

> **What is this secret type?** `docker-registry` is a Kubernetes secret type specifically for image registry credentials. Despite the name, it works with any OCI-compliant registry — Docker Hub, Amazon ECR, GitHub Container Registry, Google Artifact Registry, etc. The name reflects Docker's historical role as the first container registry, not a restriction to Docker Hub only.

**Verify the secret was created:**
```bash
kubectl get secret dockerhub-secret -n podinfo
```

**Expected:**
```
NAME               TYPE                             DATA   AGE
dockerhub-secret   kubernetes.io/dockerconfigjson   1      10s
```

---

### Step 9: Create Kubernetes Manifests

**Note:** These are changes in `gitops-labs` local repo 

Create the reference manifests in `src/podinfo-config/` of your `gitops-labs` repo. These same files will be pushed to `rselvantech/podinfo-config` in Demo-04 Part 2.

**`src/podinfo-config/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: podinfo
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
        - name: dockerhub-secret         # ← references the secret created in Step 8
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0
          ports:
            - name: http
              containerPort: 9898        # ← podinfo's HTTP port
            - name: http-metrics
              containerPort: 9797        # ← Prometheus metrics port
          env:
            - name: PODINFO_UI_COLOR
              value: "#34577c"           # ← default blue-grey colour
            - name: PODINFO_UI_MESSAGE
              value: "GitOps with ArgoCD — Dev Environment"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 9898
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: 9898
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              memory: 256Mi
```

**`src/podinfo-config/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: podinfo
spec:
  selector:
    app: podinfo
  ports:
    - name: http
      port: 9898
      targetPort: 9898
    - name: metrics
      port: 9797
      targetPort: 9797
  type: ClusterIP
```

---

### Step 10: Apply Manifests Directly

```bash
kubectl apply -f src/podinfo-config/deployment.yaml
kubectl apply -f src/podinfo-config/service.yaml
```

**Watch pods come up:**
```bash
kubectl get pods -n podinfo -w
```

**Expected:**
```
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

---

### Step 11: Verify Resources

```bash
# Check deployment
kubectl get deploy -n podinfo

# Check service
kubectl get svc -n podinfo

# Check pod logs
kubectl logs -n podinfo -l app=podinfo
```

**Expected pod logs:**
```
{"level":"info","ts":"...","caller":"podinfo/main.go:...","msg":"Starting podinfo","version":"...","revision":"..."}
{"level":"info","ts":"...","msg":"HTTP server listening on port 9898"}
```

---

## Part 5: Explore Podinfo

### Step 12: Access the Podinfo UI

```bash
# Use a separate terminal — keep this running
kubectl port-forward svc/podinfo -n podinfo 9898:9898
```

Open `http://localhost:9898` in your browser.

**What you should see:**
- Background colour matching `PODINFO_UI_COLOR` (`#34577c` — blue-grey)
- Custom message: `"GitOps with ArgoCD — Dev Environment"`
- Runtime info: hostname (pod name), version, Go runtime version
- The podinfo logo

---

### Step 13: Explore the API Endpoints

Open a new terminal and run:

```bash
# Health check — liveness probe equivalent
curl http://localhost:9898/healthz
```
**Expected:** `{"status":"ok"}`

```bash
# Readiness check — readiness probe equivalent
curl http://localhost:9898/readyz
```
**Expected:** `{"status":"ok"}`

```bash
# Version information
curl http://localhost:9898/version
```
**Expected:**
```json
{
  "commit": "",
  "version": "6.5.4"
}
```

```bash
# Prometheus metrics (in a separate port-forward if needed)
kubectl port-forward svc/podinfo -n podinfo 9797:9797
curl http://localhost:9797/metrics | head -20
```

**Expected:** Standard Prometheus metrics output (`go_*`, `process_*`, `podinfo_*` metrics)

---

### Step 14: Test the `PODINFO_UI_COLOR` Feature

This feature becomes essential in multi-environment GitOps demos. Test it now to understand how it works.

```bash
# Patch the deployment to change colour to green
kubectl set env deployment/podinfo \
  PODINFO_UI_COLOR="#00ff00" \
  PODINFO_UI_MESSAGE="Staging Environment" \
  -n podinfo
```

Refresh `http://localhost:9898` — the background changes to green immediately after the pod restarts.

Change it back:
```bash
kubectl set env deployment/podinfo \
  PODINFO_UI_COLOR="#34577c" \
  PODINFO_UI_MESSAGE="GitOps with ArgoCD — Dev Environment" \
  -n podinfo
```

> **Why this matters:** In later demos, you will have `overlays/dev`, `overlays/staging`, `overlays/prod` Kustomize directories — each setting a different `PODINFO_UI_COLOR`. When you deploy to different namespaces, you can immediately confirm in the browser which environment is which.

---

## Validation Checklist

Before proceeding to Demo-04 Part 2, verify:

- [ ] `rselvantech/podinfo` fork exists on GitHub
- [ ] `rselvantech/podinfo-config` private repo exists on GitHub
- [ ] `rselvantech/argocd-config` private repo exists on GitHub
- [ ] `rselvantech/podinfo` private repo exists on Docker Hub
- [ ] Docker image `rselvantech/podinfo:v1.0.0` is visible in Docker Hub with `Private` label
- [ ] `kubectl get pods -n podinfo` shows pod in `Running` state
- [ ] `curl http://localhost:9898/healthz` returns `{"status":"ok"}`
- [ ] `curl http://localhost:9898/readyz` returns `{"status":"ok"}`
- [ ] Podinfo UI is visible in browser at `http://localhost:9898`
- [ ] UI colour changes work when `PODINFO_UI_COLOR` is updated

---

## Cleanup

This cleanup is **optional** — you can leave podinfo running and proceed directly to Demo-04 Part 2. If you prefer a clean state before Part 2:

```bash
# Remove the direct kubectl deployment
kubectl delete -f src/podinfo-config/deployment.yaml
kubectl delete -f src/podinfo-config/service.yaml

# Remove the namespace (also removes the secret)
kubectl delete ns podinfo

# Stop port-forwarding (Ctrl+C in the respective terminals)
```

> **Do NOT delete** the Docker Hub image, GitHub repos, or your local clone of the podinfo fork — these are needed for Demo-04 Part 2.

---

## What You Learned

In this setup guide, you:

- ✅ Understood what podinfo is — a CNCF-used GitOps reference application by Stefan Prodan
- ✅ Understood podinfo's ports (9898 HTTP, 9797 metrics, 9999 gRPC) and key endpoints
- ✅ Understood the three-repo strategy — why production GitOps separates app source, config, and ArgoCD CRDs
- ✅ Forked podinfo to your GitHub account
- ✅ Created `podinfo-config` and `argocd-config` private repos
- ✅ Built a podinfo image from your fork and pushed it to a private Docker Hub repo
- ✅ Deployed podinfo on minikube using plain `kubectl` with a `docker-registry` secret
- ✅ Explored the podinfo UI, health endpoints, metrics, and the colour-change feature

**Key Insight:**
The three-repo strategy is not overhead — it is the architecture that enables clean RBAC, independent CI/CD pipelines per concern, and ArgoCD self-management via the App-of-Apps pattern. Setting it up correctly now means every subsequent demo builds on a professional foundation rather than requiring restructuring later.

---

## Lessons Learned

### 1. Docker Hub PAT (Personal Access Token) Permission Levels Are Not Intuitive

Docker Hub has three PAT permission levels that are easy to confuse:

| Permission | Pull Public | Pull Private | Push |
|---|---|---|---|
| `Public repo read-only` | ✅ | ❌ | ❌ |
| `Read-only` | ✅ | ✅ | ❌ |
| `Read & Write` | ✅ | ✅ | ✅ |

**Rule:** Use `Read-only` PAT for Kubernetes `dockerhub-secret` — it is the minimum permission required to pull private images and follows least privilege.

---

### 2. Always Wrap Credentials in Single Quotes in CLI Commands

Shell interprets special characters in tokens without quotes — causing silent truncation or corruption:

```bash
# Wrong — shell may truncate or misinterpret ❌
--docker-password=dckr_pat_xxxxx-Q

# Wrong — $ ! ` still interpreted ❌
--docker-password="dckr_pat_xxxxx-Q"

# Correct — exact literal value, nothing interpreted ✅
--docker-password='dckr_pat_xxxxx-Q'
```

**Rule:** Always use single quotes for any credential, token, or password passed as a CLI argument — `kubectl`, `docker`, `argocd`, or any other tool.

---

### 3. Always Verify Secret Content After Creation

Never assume the secret was created correctly. Always decode and verify:

```bash
kubectl get secret dockerhub-secret -n podinfo \
  -o go-template='{{index .data ".dockerconfigjson"}}' | base64 --decode
"
```

Verify:
- Username is correct
- Token length is expected (Docker Hub PATs are ~36 chars including `dckr_pat_` prefix)
- Token starts with `dckr_pat_`

**Rule:** Verify secret content immediately after creation — before deploying anything that depends on it.

---

### 5. `docker-registry` Secret Type Works With Any OCI Registry

Despite the name, `kubernetes.io/dockerconfigjson` is not Docker Hub specific:

```bash
# Same secret type for all registries
kubectl create secret docker-registry my-secret \
  --docker-server=https://index.docker.io/v1/      # Docker Hub
  --docker-server=<aws-account>.dkr.ecr.<region>.amazonaws.com  # Amazon ECR
  --docker-server=ghcr.io                          # GitHub Container Registry
  --docker-server=gcr.io                           # Google Container Registry
```

**Rule:** Always use `docker-registry` secret type for image pull credentials regardless of which registry you use.

---

### 6. `imagePullSecrets` Must Exist in the Same Namespace as the Pod

The secret is namespace-scoped — a secret in `podinfo` namespace is invisible to `staging` or `prod` namespaces:

```bash
# Must recreate secret in every namespace where private images are used
kubectl create secret docker-registry dockerhub-secret ... --namespace=podinfo
kubectl create secret docker-registry dockerhub-secret ... --namespace=staging
kubectl create secret docker-registry dockerhub-secret ... --namespace=prod
```

**Rule:** Create `docker-registry` secret in every namespace where pods with private images will run. `ImagePullBackOff` in a new namespace is often caused by a missing secret.

---

### 7. Use `jsonpath` Dot-Prefix Escape or `go-template` for Dot-Prefixed Keys

`.dockerconfigjson` starts with a dot — standard `jsonpath` treats it as a path separator and returns empty:

```bash
# Wrong — dot treated as path separator, returns empty ❌
-o jsonpath='{.data.dockerconfigjson}'

# Dot-prefixed keys — double quotes outer, single quotes inner, bracket notation✅
-o jsonpath="{.data['\.dockerconfigjson']}"

# Also correct — go-template with index ✅
-o go-template='{{index .data ".dockerconfigjson"}}'
```

**Rule:** For any Kubernetes field with a dot-prefixed key, use bracket notation with escaped dot or `go-template` with `index`.

---

### 8. `index.docker.io/v1/` Is the Correct Auth Endpoint — Not `hub.docker.com`

```
hub.docker.com          ← human-facing website (browser)
index.docker.io/v1/     ← machine-facing auth endpoint (CLI, Kubernetes)
registry-1.docker.io    ← actual image data endpoint
```

**Rule:** Always use `https://index.docker.io/v1/` as `--docker-server` for Docker Hub secrets — regardless of what URL you use in the browser.


### 9. The Podinfo Fork is Your Long-Term Demo App

You will use `rselvantech/podinfo` and `rselvantech/podinfo-config` throughout this course — for Helm demos, Kustomize demos, progressive delivery, and multi-cluster demos. Treat the fork as a real repository. Keep it clean, use proper commit messages, and follow branching practices as you would in a production project.

**Rule:** Never push credentials, tokens, or secrets to any of the three repos — even though they are private.

### 10. Repo 3 (`argocd-config`) Is Created Now But Fully Used Later

The `argocd-config` repo is created in this setup but ArgoCD will not read from it automatically until the App-of-Apps demo. In Demo-04 Part 2, you will manually `kubectl apply` the Application CRD from your local machine. The repo exists so your workflow is already structured correctly — you are building a habit of version-controlling ArgoCD configuration from day one.

**Rule:** Always version-control your ArgoCD `Application` CRDs in `argocd-config`. Never manage them purely via the UI or ad-hoc `kubectl apply` without committing to Git.
---

## Next Steps

**Demo-04 Part 2 — Production GitOps: Private Repos, Secrets & Advanced Sync**
- Generate GitHub fine-grained PAT for repository access
- Push manifests to `podinfo-config` and Application CRD to `argocd-config`
- Configure ArgoCD to access private repositories via CLI
- Apply the ArgoCD Application CRD and perform manual sync
- Enable Automated Sync, Pruning, and Self-Healing

**Explore on your own (optional):**
- Browse the `kustomize/` directory inside your podinfo fork — notice the `base/` structure and how overlays would work
- Browse the `charts/podinfo/` directory — see the full Helm values available
- Try `curl http://localhost:9898/env` — observe what environment variables are exposed

