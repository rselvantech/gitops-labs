# Demo-15: ArgoCD Image Updater — Automated Image Tag Propagation

## Overview

Every demo so far has used a fixed image tag — `rselvantech/podinfo:v1.0.0` pinned
directly in the Deployment manifest. That works for learning. In production it creates
a manual bottleneck: every time your CI pipeline builds and pushes a new image, someone
must update the YAML file in the config repo, commit it, and push. At scale, this
breaks the automation loop.

**ArgoCD Image Updater** closes this gap. It watches your container registry for new
image tags, evaluates them against a strategy (latest, semver, digest), and writes the
updated tag back to your Git config repo automatically. ArgoCD then picks up the change
and deploys it — without anyone touching a YAML file.

This demo isolates and demonstrates Image Updater as a standalone capability, using the
same `podinfo` application from earlier demos with a second image version you push to
Docker Hub.

**What you'll learn:**
- Why automated image tag propagation is necessary in production GitOps
- What ArgoCD Image Updater is and how it fits into the ArgoCD ecosystem
- The difference between `argocd` write-back and `git` write-back strategies
- The three update strategies: `semver`, `latest`, `digest`
- How to configure Image Updater via Application CRD annotations
- How credentials for private registries are provided to Image Updater
- What the `.argocd-source-<app-name>.yaml` write-back file looks like
- How to verify Image Updater is watching, detecting, and writing

**What you'll do:**
- Install ArgoCD Image Updater into the `argocd` namespace via Helm
- Push a second image version (`rselvantech/podinfo:v1.1.0`) to Docker Hub
- Configure the `podinfo` application with Image Updater annotations
- Observe Image Updater detect the new tag, update Git, and trigger a sync
- Understand the full automated pipeline from image push to pod restart

---

## Prerequisites

- ✅ Completed Demo-05 — `podinfo-config` and `argocd-config` repos exist,
  podinfo image `rselvantech/podinfo:v1.0.0` in Docker Hub
- ✅ ArgoCD running on minikube (default profile)
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ Docker CLI installed and authenticated (`docker login`)
- ✅ GitHub PAT with `Contents: Read and write` access to `podinfo-config`
  — Image Updater needs write access to commit the updated tag

> **Why `Read and write` for the PAT?** Earlier demos only needed read access
> because ArgoCD only reads your config repo. Image Updater is different — it
> needs to write a commit back to `podinfo-config` when it detects a new image
> tag. This is the one case where your config repo credential requires write
> access.

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

### 4. Docker Hub image exists and Docker is authenticated
```bash
docker pull rselvantech/podinfo:v1.0.0
docker info | grep Username
```

**Expected:** Image pulled successfully. Docker is logged in as `rselvantech`.

If not logged in:
```bash
docker login --username rselvantech
```

---

## Concepts

### The Missing Link in Static GitOps

Traditional GitOps manages infrastructure and application configuration via Git.
But it has a blind spot — the image tag in your Deployment manifest is just a
string. Nothing automatically updates it when your CI pipeline builds a new version.

The typical manual workflow:
```
CI builds rselvantech/podinfo:v1.1.0
  → pushes to Docker Hub
  → CI engineer updates deployment.yaml
  → commits and pushes to podinfo-config
  → ArgoCD detects the change and syncs
  → pod restarts with new image
```

The problem is the middle three steps. They are manual, error-prone, and slow.
Teams add custom CI scripts to do this update — which means every team reinvents
the same wheel and every implementation differs.

ArgoCD Image Updater is the standard solution to this problem.

---

### What ArgoCD Image Updater Is

ArgoCD Image Updater is a separate controller that runs alongside ArgoCD. It
periodically polls container registries for new image tags, compares them against
a configured update strategy, and writes updated tags back to Git automatically.

```
Container Registry (Docker Hub)
        │
        │ Image Updater polls every N seconds
        │ Detects: rselvantech/podinfo:v1.1.0
        ▼
ArgoCD Image Updater
        │ Update strategy: semver — v1.1.0 > v1.0.0 ✅
        │ Write-back method: git
        ▼
podinfo-config (GitHub)
        │ Commits: .argocd-source-podinfo.yaml
        │ tag: "v1.1.0"
        ▼
ArgoCD Application Controller
        │ Detects diff — deployment uses v1.0.0, Git now says v1.1.0
        ▼
Kubernetes Cluster
        │ Deployment rolls out with rselvantech/podinfo:v1.1.0
        ▼
Pod running v1.1.0
```

This is the full automation loop. No manual YAML editing. No CI scripts.
No human intervention after the image is pushed.

---

### Write-Back Strategies — `argocd` vs `git`

Image Updater supports two methods for persisting the updated image tag:

**`argocd` (direct write-back):**
Image Updater updates the Application object directly in Kubernetes, overriding
the image tag at runtime. The Git repo is never touched. The override lives
in the Application's `status` field.

```
Pros:  Fast — no Git commit delay
Cons:  The change is not in Git — if the Application is deleted and recreated,
       the override is lost. Not truly GitOps.
```

**`git` (write-back to repo) — recommended:**
Image Updater commits a small override file (`.argocd-source-<app-name>.yaml`)
to your config repo. ArgoCD detects this commit and syncs the deployment with
the new tag.

```
Pros:  Fully GitOps — the new tag is in Git, auditable, recoverable
Cons:  Slightly slower — requires a Git commit → ArgoCD poll cycle
```

> **This demo uses `git` write-back.** It is the production-aligned approach —
> the updated tag exists in Git history and is recoverable. The `argocd` method
> is convenient for experimentation but is not durable.

---

### Update Strategies

Image Updater supports three strategies for deciding which new tag to use:

**`semver` — semantic version ordering:**
Only updates to a newer version according to semver rules. `v1.1.0` is newer
than `v1.0.0`. `v2.0.0-beta` is not newer than `v1.9.9` (pre-release constraint).

```yaml
# Accepts any new semver-compatible tag
argocd-image-updater.argoproj.io/image-list: podinfo=rselvantech/podinfo
argocd-image-updater.argoproj.io/podinfo.update-strategy: semver
```

Best for: applications that use proper semantic versioning. Most production cases.

**`latest` — newest by push date:**
Always uses the most recently pushed tag. No version ordering — just whichever
tag was pushed most recently to the registry.

```yaml
argocd-image-updater.argoproj.io/podinfo.update-strategy: latest
```

Best for: `latest` or rolling tags where no semantic version exists.

**`digest` — image digest tracking:**
Tracks the digest (`sha256:...`) of a specific tag. When the same tag (e.g.
`latest`) is overwritten with a new image, the digest changes — Image Updater
detects this and updates.

```yaml
argocd-image-updater.argoproj.io/podinfo.update-strategy: digest
```

Best for: mutable tags like `main` or `latest` where the tag never changes but
the image content does.

> **In this demo:** We use `semver`. We push `v1.1.0` to Docker Hub and Image
> Updater detects it as newer than the current `v1.0.0`.

---

### The Write-Back File — What Git Receives

When Image Updater uses `git` write-back, it commits a file named
`.argocd-source-<application-name>.yaml` to the root of your config repo
on the tracked branch (`main`).

For the `podinfo` application, the file looks like:

```yaml
# .argocd-source-podinfo.yaml
# Auto-generated by ArgoCD Image Updater — do not edit manually
kustomize:
  images:
  - rselvantech/podinfo:v1.1.0
```

ArgoCD reads this file and merges it with the Application's source configuration —
overriding the image tag in the Deployment without touching `deployment.yaml` directly.
This is a deliberate design: Image Updater touches only this one file. Your
actual manifests remain unchanged.

> **Important:** The Application must track a branch (`main`) — not `HEAD`.
> Image Updater commits to a branch reference. If `targetRevision: HEAD` is
> used (which resolves to the default branch tip), commits from Image Updater
> are not detected reliably. Always set `targetRevision: main` when using
> Image Updater.

---

### Credential Scopes

Image Updater needs two separate credentials:

| Credential | Purpose | Where configured |
|---|---|---|
| **Docker Hub** (or registry) | Pull image metadata — list tags, read digests | Kubernetes Secret in `argocd` namespace |
| **GitHub PAT** | Write the `.argocd-source-*.yaml` commit to config repo | ArgoCD repository credential (`argocd repo add`) |

These are independent. Image Updater authenticates to the registry to read tag
information and to Git to write the override file.

**Registry credential secret format:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: image-updater-dockerhub
  namespace: argocd
  labels:
    argocd-image-updater.argoproj.io/credentials: "true"  # ← mandatory label
type: Opaque
stringData:
  credentials: "rselvantech:<DOCKERHUB_TOKEN>"
```

The `credentials` label tells Image Updater to use this secret when authenticating
with Docker Hub. Without the label, the secret is ignored.

---

### Architecture: Where Image Updater Lives

```
┌──────────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (minikube)                                   │
│                                                                  │
│  namespace: argocd                                               │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                                                            │  │
│  │  ArgoCD Application Controller  ◄─────── podinfo-config   │  │
│  │  (watches Git, syncs cluster)           (GitHub)          │  │
│  │                                              ▲            │  │
│  │  ArgoCD Image Updater           ─ ─ ─ ─ ─ ─ ┘            │  │
│  │  (polls registry, writes Git)   commits .argocd-source-*  │  │
│  │           │                                                │  │
│  └───────────┼────────────────────────────────────────────────┘  │
│              │                                                   │
└──────────────┼───────────────────────────────────────────────────┘
               │ polls for new tags
               ▼
       Docker Hub Registry
       rselvantech/podinfo:v1.0.0  ← current
       rselvantech/podinfo:v1.1.0  ← new — detected by Image Updater
```

Both ArgoCD and Image Updater run in the same `argocd` namespace. Image Updater
is installed separately from ArgoCD core — it is an optional add-on.

---

## Directory Structure

```
15-image-updater/src/
├── podinfo-config/              ← git init → remote: rselvantech/podinfo-config
│   └── demo-15-image-updater/
│       ├── deployment.yaml      ← uses v1.0.0 initially
│       └── service.yaml
└── argocd-config/               ← git init → remote: rselvantech/argocd-config
    └── demo-15-image-updater/
        └── podinfo-image-updater-app.yaml
```

**What happens in GitHub after all pushes:**

```
rselvantech/podinfo-config (GitHub)
├── deployment.yaml              ← Demo-05 root (untouched)
├── demo-06-sync-pruning/        ← Demo-06 (untouched)
├── demo-09-argocd-projects/     ← Demo-09 (untouched)
└── demo-15-image-updater/       ← This demo adds this
    ├── deployment.yaml
    └── service.yaml
    # After Image Updater runs:
    └── .argocd-source-podinfo-image-updater-demo.yaml  ← auto-committed by Image Updater

rselvantech/argocd-config (GitHub)
├── podinfo-app.yaml             ← Demo-05 (untouched)
├── demo-06-sync-pruning/        ← Demo-06 (untouched)
├── demo-09-argocd-projects/     ← Demo-09 (untouched)
├── demo-10-sync-hooks/          ← Demo-10 (untouched)
├── demo-11-sync-waves/          ← Demo-11 (untouched)
└── demo-15-image-updater/       ← This demo adds this
    └── podinfo-image-updater-app.yaml
```

> Only `podinfo-config` needs to be registered with ArgoCD — it is the source
> ArgoCD and Image Updater both interact with. `argocd-config` is applied once
> with `kubectl apply` and does not need ArgoCD registration.

---

## Step 1: Install ArgoCD Image Updater

Image Updater is a separate Helm chart. Install it into the same `argocd` namespace
as ArgoCD.

**Add the Image Updater Helm repo:**
```bash
helm repo add argocd-image-updater https://argoproj.github.io/argocd-image-updater
helm repo update
```

**Install Image Updater:**
```bash
helm install argocd-image-updater argocd-image-updater/argocd-image-updater \
  --namespace argocd \
  --set config.argocd.insecure=true \
  --set config.argocd.serverAddress=argocd-server.argocd.svc.cluster.local
```

> **`config.argocd.insecure=true`** — required because our ArgoCD server uses
> a self-signed certificate. In production with a valid TLS certificate, omit this.
>
> **`config.argocd.serverAddress`** — the in-cluster service address for ArgoCD.
> Image Updater communicates with ArgoCD via its API to retrieve Application
> definitions and trigger sync.

**Verify Image Updater is running:**
```bash
kubectl get pods -n argocd | grep image-updater
```

**Expected:**
```text
argocd-image-updater-xxxxxxxxx-xxxxx   1/1   Running   0   30s
```

**Check Image Updater logs to confirm it started successfully:**
```bash
kubectl logs -l app.kubernetes.io/name=argocd-image-updater -n argocd --tail=20
```

**Expected:**
```text
time="..." level=info msg="ArgoCD Image Updater started"
time="..." level=info msg="Starting image update cycle"
time="..." level=info msg="Starting metrics server"
```

---

## Step 2: Push a New Image Version to Docker Hub

We need a second image version for Image Updater to detect. We build `v1.1.0` from
the same podinfo image with a different environment variable — no source code changes
needed.

**Build and tag `v1.1.0`:**
```bash
# Pull v1.0.0 as the base (we already have this from Demo-05)
docker pull rselvantech/podinfo:v1.0.0

# Re-tag as v1.1.0
docker tag rselvantech/podinfo:v1.0.0 rselvantech/podinfo:v1.1.0

# Push v1.1.0 to Docker Hub
docker push rselvantech/podinfo:v1.1.0
```

**Verify both tags exist on Docker Hub:**
```bash
docker pull rselvantech/podinfo:v1.1.0
```

**Expected:** `Already exists` or pulled successfully.

**Verify on Docker Hub UI:**
Go to `hub.docker.com/r/rselvantech/podinfo/tags` — you should see both
`v1.0.0` and `v1.1.0` listed.

> **Why re-tag instead of building a truly different image?** For this demo the
> goal is to show Image Updater detecting a new tag — not to demonstrate a
> different application version. In production, your CI pipeline builds a
> genuinely different image from new source code and pushes it with the new tag.
> The Image Updater mechanism is identical either way.

---

## Step 3: Configure Registry Credentials for Image Updater

Image Updater needs credentials to query Docker Hub for image tags. Create a
Kubernetes Secret in the `argocd` namespace with the mandatory label.

```bash
kubectl create secret generic image-updater-dockerhub \
  --namespace argocd \
  --from-literal=credentials="rselvantech:<DOCKERHUB_TOKEN>"
```

**Apply the mandatory label** — without this label Image Updater ignores the secret:
```bash
kubectl label secret image-updater-dockerhub \
  -n argocd \
  argocd-image-updater.argoproj.io/credentials="true"
```

**Verify the secret and label:**
```bash
kubectl get secret image-updater-dockerhub -n argocd \
  --show-labels
```

**Expected:**
```text
NAME                      TYPE     DATA   AGE   LABELS
image-updater-dockerhub   Opaque   1      10s   argocd-image-updater.argoproj.io/credentials=true
```

> **Why a separate secret from the existing `dockerhub-secret` in the `podinfo`
> namespace?** The existing pull secret is namespace-scoped and used by the
> kubelet to pull images. Image Updater's credential secret is a different
> concern — it lives in the `argocd` namespace and is used to query the
> registry API (tag listing), not to pull images. They solve different problems
> at different layers and must be created separately.

---

## Step 4: Configure Git Write-Back Credentials

Image Updater needs write access to `podinfo-config` to commit the
`.argocd-source-*.yaml` override file. This uses the same ArgoCD repo credential
mechanism as always — but the PAT must have `Contents: Read and write` permission
(not just read-only).

**Verify or update your PAT scope:**
- Go to GitHub → Settings → Developer settings → Personal access tokens
- Confirm `podinfo-config` has `Contents: Read and write` permission
- If read-only, edit the token and update to `Read and write`

**Register or re-register `podinfo-config` with updated credentials:**
```bash
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password <GITHUB_PAT> \
  --name podinfo-config \
  --upsert
```

> **`--upsert`** — updates the existing credential entry instead of failing if
> it already exists. Use this whenever re-registering a repo that was registered
> in a prior demo.

**Verify:**
```bash
argocd repo list
```

**Expected:** `podinfo-config` listed with `Successful` status.

---

## Step 5: Add Manifests to `podinfo-config`

The application manifests use `v1.0.0` as the starting image tag. Image Updater will
update this to `v1.1.0` automatically after detecting it in the registry.

**Set up `podinfo-config` local repo:**
```bash
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src
mkdir podinfo-config && cd podinfo-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

**Create `demo-15-image-updater/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: podinfo-image-updater
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
          image: rselvantech/podinfo:v1.0.0    # ← Image Updater will update this
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#3d8eb9"
            - name: PODINFO_UI_MESSAGE
              value: "Image Updater Demo — watch this tag change automatically"
```

**Create `demo-15-image-updater/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: podinfo-image-updater
spec:
  selector:
    app: podinfo
  ports:
    - port: 9898
      targetPort: 9898
```

**Push to `podinfo-config`:**
```bash
git add demo-15-image-updater/
git commit -m "feat: add demo-15 image updater manifests starting at v1.0.0"
git push origin main
```

---

## Step 6: Create the Namespace and Docker Registry Secret

```bash
kubectl create namespace podinfo-image-updater
```

Create the pull secret in the new namespace — same pattern as Demo-05:
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password=<DOCKERHUB_TOKEN> \
  --namespace=podinfo-image-updater
```

**Verify:**
```bash
kubectl get namespace podinfo-image-updater
kubectl get secret dockerhub-secret -n podinfo-image-updater
```

**Expected:**
```text
NAME                     STATUS   AGE
podinfo-image-updater    Active   5s

NAME               TYPE                             DATA   AGE
dockerhub-secret   kubernetes.io/dockerconfigjson   1      5s
```

---

## Step 7: Create the Application CRD with Image Updater Annotations

The Application CRD is where Image Updater is configured. All Image Updater
behaviour is driven by annotations on the Application object — no separate
CRD or configuration file is needed.

**Set up `argocd-config` local repo:**
```bash
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

**Create `demo-15-image-updater/podinfo-image-updater-app.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-image-updater-demo
  namespace: argocd
  annotations:
    # --- Image Updater annotations ---

    # 1. Define the image(s) to watch — alias=registry/image
    argocd-image-updater.argoproj.io/image-list: podinfo=rselvantech/podinfo

    # 2. Update strategy for the 'podinfo' alias — semver means newer version wins
    argocd-image-updater.argoproj.io/podinfo.update-strategy: semver

    # 3. Constraint — only update within the v1.x.x minor range
    argocd-image-updater.argoproj.io/podinfo.allow-tags: regexp:^v1\.\d+\.\d+$

    # 4. Write-back method — git commits the override file to the config repo
    argocd-image-updater.argoproj.io/write-back-method: git

    # 5. Git credentials — uses the ArgoCD repo credential already registered
    argocd-image-updater.argoproj.io/git-repository: https://github.com/rselvantech/podinfo-config.git

    # 6. Registry credentials — references the secret created in Step 3
    argocd-image-updater.argoproj.io/podinfo.pull-secret: secret:argocd/image-updater-dockerhub#credentials

spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/podinfo-config.git
    targetRevision: main                  # ← must be a branch name, not HEAD
    path: demo-15-image-updater
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo-image-updater
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Annotation field-by-field explanation:**

`image-list: podinfo=rselvantech/podinfo` — defines the image to watch.
`podinfo` is an alias used to reference this image in all other annotations.
`rselvantech/podinfo` is the registry path (Docker Hub shorthand for
`docker.io/rselvantech/podinfo`).

`podinfo.update-strategy: semver` — tells Image Updater to apply semantic
version ordering. `v1.1.0 > v1.0.0` — the update will be applied.

`podinfo.allow-tags: regexp:^v1\.\d+\.\d+$` — constrains which tags are
considered. Only tags matching `v1.x.x` are evaluated. This prevents Image
Updater from picking up `latest`, `main`, or other non-semver tags if they
exist on the same image.

`write-back-method: git` — use Git write-back (commits override file to repo)
rather than direct ArgoCD write-back.

`git-repository` — the URL of the config repo to write back to. Must match
the repo registered with ArgoCD credentials.

`podinfo.pull-secret: secret:argocd/image-updater-dockerhub#credentials` — tells
Image Updater which secret to use for registry authentication. Format:
`secret:<namespace>/<secret-name>#<key-in-secret>`.

> **`targetRevision: main`** — this is the single most important rule when using
> Image Updater with git write-back. Image Updater commits to the `main` branch.
> If `targetRevision` is set to `HEAD`, ArgoCD resolves it to a commit SHA at
> sync time — and Image Updater's new commits on `main` will not be detected
> because the SHA changes but ArgoCD keeps checking the old one. Always set
> `targetRevision` to an explicit branch name.

**Push and apply:**
```bash
git add demo-15-image-updater/podinfo-image-updater-app.yaml
git commit -m "feat: add demo-15 Application CRD with Image Updater annotations"
git push origin main

kubectl apply -f demo-15-image-updater/podinfo-image-updater-app.yaml
```

**Verify the Application was created:**
```bash
argocd app get podinfo-image-updater-demo
```

**Expected:**
```text
Sync Status:   OutOfSync
Health Status: Missing
```

Because `automated` sync is enabled, ArgoCD will sync within ~3 minutes —
or click **Refresh** in the UI to trigger immediately.

**Verify the initial deployment uses `v1.0.0`:**
```bash
kubectl get pods -n podinfo-image-updater -w
```

**Expected:**
```text
NAME                       READY   STATUS    RESTARTS
podinfo-xxxxxxxxx-xxxxx    1/1     Running   0
```

```bash
kubectl get deployment podinfo -n podinfo-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected:**
```text
rselvantech/podinfo:v1.0.0
```

---

## Step 8: Observe Image Updater Detecting and Writing

With the application deployed at `v1.0.0` and `v1.1.0` already pushed to Docker Hub,
Image Updater will detect the newer tag on its next polling cycle and commit the
override file to `podinfo-config`.

**Watch Image Updater logs in real time:**
```bash
kubectl logs -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd -f --tail=50
```

**Expected — Image Updater discovers v1.1.0 and commits:**
```text
time="..." level=info msg="Starting image update cycle, considering 1 annotated application(s)"
time="..." level=info msg="Processing application podinfo-image-updater-demo"
time="..." level=info msg="Fetching available tags and digest for image rselvantech/podinfo"
time="..." level=info msg="Found 2 tags for image rselvantech/podinfo" tags="[v1.0.0 v1.1.0]"
time="..." level=info msg="Latest image according to semver: rselvantech/podinfo:v1.1.0"
time="..." level=info msg="Updating image rselvantech/podinfo:v1.0.0 to rselvantech/podinfo:v1.1.0"
time="..." level=info msg="Successfully updated the live application spec"
time="..." level=info msg="Committing 1 parameter update(s) for application podinfo-image-updater-demo"
time="..." level=info msg="Successfully updated Git"
```

> Image Updater polls every 2 minutes by default. If you do not see the log
> output immediately, wait up to 2 minutes. Logs are streamed in real time.

**Verify the write-back file was committed to `podinfo-config`:**

```bash
# Pull latest from podinfo-config to see the committed file
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src/podinfo-config
git pull origin main

ls -la demo-15-image-updater/
```

**Expected — new file committed by Image Updater:**
```text
-rw-r--r--  deployment.yaml
-rw-r--r--  service.yaml
-rw-r--r--  .argocd-source-podinfo-image-updater-demo.yaml   ← committed by Image Updater
```

**Inspect the write-back file:**
```bash
cat demo-15-image-updater/.argocd-source-podinfo-image-updater-demo.yaml
```

**Expected:**
```yaml
kustomize:
  images:
  - rselvantech/podinfo:v1.1.0
```

**Check the Git commit history on `podinfo-config`:**
```bash
git log --oneline -5
```

**Expected — Image Updater commit visible:**
```text
a3f2c1d (HEAD -> main, origin/main) Update image rselvantech/podinfo:v1.1.0  ← Image Updater
b4e1a2f feat: add demo-15 image updater manifests starting at v1.0.0
```

---

## Step 9: Verify ArgoCD Syncs with the New Tag

After Image Updater commits the write-back file, ArgoCD detects the Git change and
triggers an automated sync. The Deployment rolls out with `v1.1.0`.

**Watch the rollout:**
```bash
kubectl rollout status deployment/podinfo -n podinfo-image-updater
```

**Expected:**
```text
Waiting for deployment "podinfo" rollout to finish: 1 out of 1 new replicas have been updated...
deployment "podinfo" successfully rolled out
```

**Verify the running pod uses `v1.1.0`:**
```bash
kubectl get deployment podinfo -n podinfo-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected:**
```text
rselvantech/podinfo:v1.1.0
```

**Verify in ArgoCD UI:**
- Go to `http://localhost:8080` → click `podinfo-image-updater-demo`
- `SYNC STATUS:   Synced ✅`
- `HEALTH STATUS: Healthy ✅`
- Click the `podinfo` Deployment resource → `LIVE MANIFEST` tab
- Confirm `image: rselvantech/podinfo:v1.1.0`

**Access the podinfo UI to confirm the new version:**
```bash
kubectl port-forward svc/podinfo -n podinfo-image-updater 9898:9898
```

Open `http://localhost:9898` — the podinfo UI confirms the running version.

---

## Verify Final State

```bash
# Application synced and healthy
argocd app get podinfo-image-updater-demo

# Pod running with v1.1.0
kubectl get deployment podinfo -n podinfo-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Write-back file committed to podinfo-config
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src/podinfo-config
git log --oneline -3

# Image Updater pod is running
kubectl get pods -n argocd | grep image-updater

# Registry credential secret has the required label
kubectl get secret image-updater-dockerhub -n argocd --show-labels

# podinfo-config registered with write access
argocd repo list
```

---

## Cleanup

```bash
# Delete the Application (automated sync + prune removes namespace resources)
kubectl delete app podinfo-image-updater-demo -n argocd

# Delete the namespace
kubectl delete namespace podinfo-image-updater

# Remove the Image Updater registry credential secret
kubectl delete secret image-updater-dockerhub -n argocd

# Uninstall Image Updater (only if you are done with the course)
helm uninstall argocd-image-updater -n argocd
```

> **Do NOT delete `podinfo-config`** — it is used in later demos.
> The `.argocd-source-*.yaml` file committed by Image Updater can be left in
> the repo — it will not affect other demos that point to different paths.

**Verify cleanup:**
```bash
kubectl get namespace podinfo-image-updater   # Error: not found
kubectl get pods -n argocd | grep image-updater  # No output (if uninstalled)
argocd app list  # podinfo-image-updater-demo not listed
```

---

## Key Concepts Summary

**The gap that Image Updater fills**
Static GitOps requires someone to update the image tag in the config repo after
every CI build. Image Updater automates this step — the config repo stays updated
without human intervention, preserving the full GitOps audit trail.

**Image Updater is a separate controller, not part of ArgoCD core**
It runs as its own pod in the `argocd` namespace and communicates with ArgoCD via
its API. It is installed and upgraded independently from ArgoCD.

**`git` write-back vs `argocd` write-back**
`git` write-back commits an override file (`.argocd-source-<app-name>.yaml`) to
your config repo. The new tag exists in Git history — recoverable, auditable, and
durable. `argocd` write-back updates the Application object directly in Kubernetes
without touching Git — fast but not durable. Always prefer `git` in production.

**`targetRevision: main` is mandatory with git write-back**
Image Updater commits to a branch. If `targetRevision` is `HEAD`, ArgoCD resolves
it to a commit SHA and may not detect subsequent commits. Always use an explicit
branch name when using Image Updater.

**The write-back file is auto-managed — never edit it manually**
`.argocd-source-<app-name>.yaml` is generated and maintained by Image Updater.
Editing it manually will be overwritten on the next Image Updater cycle.

**Three update strategies for three different tagging patterns**
`semver` for versioned releases, `latest` for rolling tags by push date,
`digest` for mutable tags where content changes but the tag name stays the same.

**Two separate credential types**
Registry credentials (querying tag lists from Docker Hub) and Git credentials
(writing commits to the config repo) are independent. Both must be configured
for `git` write-back to work end-to-end.

**`allow-tags` prevents unintended updates**
A regexp constraint on `allow-tags` ensures Image Updater only considers tags
matching your versioning scheme. Without it, tags like `latest`, `main`, `sha-abc`
could be picked up by the `semver` strategy if they happen to be the newest.

---

## Commands Reference

```bash
# Install Image Updater
helm repo add argocd-image-updater https://argoproj.github.io/argocd-image-updater
helm repo update
helm install argocd-image-updater argocd-image-updater/argocd-image-updater \
  --namespace argocd \
  --set config.argocd.insecure=true \
  --set config.argocd.serverAddress=argocd-server.argocd.svc.cluster.local

# Check Image Updater pod
kubectl get pods -n argocd | grep image-updater

# Stream Image Updater logs
kubectl logs -l app.kubernetes.io/name=argocd-image-updater -n argocd -f

# Create registry credential secret
kubectl create secret generic image-updater-dockerhub \
  --namespace argocd \
  --from-literal=credentials="rselvantech:<DOCKERHUB_TOKEN>"
kubectl label secret image-updater-dockerhub \
  -n argocd \
  argocd-image-updater.argoproj.io/credentials="true"

# Register podinfo-config with write access
argocd repo add https://github.com/rselvantech/podinfo-config.git \
  --username rselvantech \
  --password <GITHUB_PAT> \
  --upsert

# Push v1.1.0 image
docker tag rselvantech/podinfo:v1.0.0 rselvantech/podinfo:v1.1.0
docker push rselvantech/podinfo:v1.1.0

# Apply Application CRD
kubectl apply -f demo-15-image-updater/podinfo-image-updater-app.yaml

# Verify current image in deployment
kubectl get deployment podinfo -n podinfo-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Watch Image Updater logs for update cycle
kubectl logs -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd -f --tail=50

# Pull latest from podinfo-config to inspect write-back file
git pull origin main
cat demo-15-image-updater/.argocd-source-podinfo-image-updater-demo.yaml

# Verify ArgoCD synced with new tag
kubectl rollout status deployment/podinfo -n podinfo-image-updater

# Access podinfo UI
kubectl port-forward svc/podinfo -n podinfo-image-updater 9898:9898
```

---

## Lessons Learned

**1. `targetRevision: main` is not optional when using git write-back**
Using `HEAD` causes ArgoCD to resolve to a commit SHA, making it unable to
track new commits from Image Updater. This is one of the most common setup
mistakes. Always use an explicit branch name.

**2. The registry credential label is mandatory**
The `argocd-image-updater.argoproj.io/credentials: "true"` label on the
secret is not optional. Without it, Image Updater ignores the secret entirely
and falls back to unauthenticated access — which fails for private registries
and hits rate limits on Docker Hub for public ones.

**3. Two credential types — registry and Git — serve different purposes**
Registry credentials (for reading tag metadata) and Git credentials (for writing
commits) are configured at different layers. Confusing them or assuming one covers
the other is a common source of errors during setup.

**4. `allow-tags` is a production safeguard, not a nice-to-have**
Without a tag constraint, Image Updater may pick up tags like `latest`, `main`,
feature branch SHAs, or other non-release tags that happen to exist on your image.
Always define an `allow-tags` regexp that matches only your release tag pattern.

**5. The write-back file is owned by Image Updater — do not edit it**
`.argocd-source-<app-name>.yaml` is fully managed by Image Updater. Any manual
edit will be overwritten on the next update cycle. Treat it as generated output,
not a configuration file.

**6. Image Updater does not build images — it only watches and updates tags**
Image Updater has no CI capability. It observes what is already in the registry.
Your CI pipeline is still responsible for building and pushing images. Image
Updater picks up what CI has pushed and propagates it to your config repo.

**7. Polling interval is 2 minutes by default**
Image Updater checks the registry every 2 minutes. This is configurable via the
`ARGOCD_IMAGE_UPDATER_INTERVAL` environment variable on the Image Updater pod.
After Image Updater commits, ArgoCD also needs a polling cycle (3 minutes by
default) to detect the Git change — unless automated sync triggers immediately
on the push event. Total latency from push to running pod: 2–5 minutes typical.

---

## What's Next

**Project-01: E2E GitOps on EKS — GitLab CI + Image Updater + Cognito RBAC**
Bring everything together in a real cloud environment — EKS cluster on AWS,
GitLab CI building the Goals App and pushing to GitLab Container Registry,
Image Updater using the semver strategy to propagate new builds automatically,
Cognito SSO for ArgoCD login, and RBAC separating admin and read-only access.
Image Updater is used here with a GitLab Container Registry private image
instead of Docker Hub — the configuration pattern is identical, only the
registry URL and credential format differ.