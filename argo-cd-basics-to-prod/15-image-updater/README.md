# Demo-15: ArgoCD Image Updater — Automated Image Tag Propagation

## Overview

Every demo so far has used a fixed image tag — `rselvantech/podinfo:v1.0.0`
pinned in the Deployment manifest. In production, this creates a manual
bottleneck. Every time a CI pipeline builds and pushes a new image, someone
must open the config repo, edit the image tag in the YAML, commit, and push.
At scale — multiple applications, multiple environments, multiple teams — this
manual loop breaks down. Tags fall out of sync. Deployments lag behind builds.
Human errors introduce stale or wrong tags.

**ArgoCD Image Updater** closes this gap. It is a separate controller that
watches your container registry for new image tags, evaluates them against an
update strategy (semver, latest, digest), and writes the updated tag back to
your Git config repo automatically. ArgoCD detects the Git change and deploys
the new image — without anyone touching a YAML file.

**The scenario in this demo:** Your team develops the `podinfo` status dashboard
that was introduced in Demo-05. CI now builds and pushes new image versions to
Docker Hub on every release. The platform team needs the dev environment to
automatically track new patch versions (`v1.x.x`) as soon as they are pushed —
while staging and prod use manual approval gates. We configure Image Updater
to watch Docker Hub for new `podinfo` semver tags, write the updated tag back
to the config repo, and let ArgoCD complete the deployment cycle — all without
a human touching the manifests.

**Why this demo combines Image Updater with Kustomize:**

Demo-14 introduced Kustomize base + overlays as the production manifest pattern.
Image Updater's Git write-back produces a file (`.argocd-source-<app>.yaml`)
that uses Kustomize image override syntax. The two features are designed to work
together — Image Updater writes the image tag, Kustomize applies it over the
base manifest. This is how production teams use both features. Demonstrating
Image Updater with raw manifests misses this integration and produces a
write-back file format that would confuse students who have just learned Kustomize.

**What you'll learn:**
- Why automated image tag propagation is necessary in production GitOps
- What ArgoCD Image Updater is and how it fits into the ArgoCD ecosystem
- The `argocd` vs `git` write-back strategies — why `git` is the only
  true GitOps approach
- The three update strategies: `semver`, `latest`, `digest` — when to use each
- How to configure Image Updater entirely through Application CRD annotations
- Why two separate credentials are needed — registry vs Git write-back
- Why the `argocd-image-updater.argoproj.io/credentials: "true"` label
  is mandatory on the registry secret
- Why `targetRevision: main` (not `HEAD`) is required for git write-back
  — and what happens silently when you use `HEAD`
- What the `.argocd-source-<app>.yaml` write-back file contains and how
  ArgoCD uses it with Kustomize
- How Image Updater + Kustomize work together — the production pattern

**What you'll do:**
- Install ArgoCD Image Updater into the `argocd` namespace via Helm
- Build the `podinfo` application using Kustomize base + overlays
  (same pattern as Demo-14)
- Push a second image version (`rselvantech/podinfo:v1.1.0`) to Docker Hub
- Configure Image Updater annotations on the Application CRD
- Observe Image Updater detect the new tag, commit the override file to Git,
  and trigger an ArgoCD sync
- Verify the full automated chain: registry push → Git commit → ArgoCD sync → pod update
- Observe the write-back file format and understand how Kustomize consumes it

---

## Prerequisites

- ✅ Completed Demo-14 — Kustomize base + overlays pattern understood,
  `gitops-apps-config` repo in use
- ✅ ArgoCD running on minikube default profile
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` and `kustomize` CLIs available
- ✅ Docker CLI installed and authenticated (`docker login` as `rselvantech`)
- ✅ GitHub PAT with `Contents: Read and write` access to `gitops-apps-config`
  — Image Updater needs write access to commit the updated tag back to Git

**Verify Prerequisites:**

### 1. ArgoCD running
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

### 3. Repos registered
```bash
argocd repo list
```

**Expected:** `gitops-apps-config` and `argocd-config` both showing `Successful`.

### 4. Docker Hub image and authentication
```bash
docker pull rselvantech/podinfo:v1.0.0
docker info | grep Username
```

**Expected:** Image pulled successfully. Logged in as `rselvantech`.

**If not logged in:**
```bash
docker login --username rselvantech
```

---

## Concepts

### The Demo Scenario — podinfo Continuous Delivery

`podinfo` is a lightweight Go web application from Stefan Prodan used throughout
this course as a reference application. It has a colour-coded web UI, version
information on the home page, and a `/version` JSON endpoint — making it easy to
visually confirm which image version is running without inspecting manifests.

Your team follows trunk-based development. Every time a new version is ready,
CI builds `rselvantech/podinfo:vX.Y.Z` and pushes it to Docker Hub. Before
Image Updater, the workflow looked like this:

```
CI pushes rselvantech/podinfo:v1.1.0 to Docker Hub
         │
         │  Manual step — someone must do this:
         ▼
Developer opens gitops-apps-config/demo-15.../overlays/dev/kustomization.yaml
Updates: images:
           - name: rselvantech/podinfo
             newTag: v1.1.0
Commits and pushes to main
         │
         ▼
ArgoCD detects change → syncs → pod updated
```

The manual middle step is the problem. It delays deployment, requires human
attention for every release, and breaks if the person responsible is unavailable.

**With Image Updater:**

```
CI pushes rselvantech/podinfo:v1.1.0 to Docker Hub
         │
         │  Image Updater polls registry every 2 minutes
         ▼
Image Updater: v1.1.0 > v1.0.0 (semver) → update required
         │
         │  Image Updater commits to gitops-apps-config
         ▼
.argocd-source-podinfo-image-updater-demo.yaml written to Git:
  kustomize:
    images:
    - rselvantech/podinfo:v1.1.0
         │
         ▼
ArgoCD detects Git commit → syncs → pod updated with v1.1.0
```

No human involvement. No YAML editing. The full cycle from image push to running
pod is automated.

---

### The Missing Link in Static GitOps

Traditional GitOps has a blind spot: the image tag in your config repo is a
string. Nothing connects it to what your CI pipeline builds and pushes. The
config repo and the registry are two separate systems with no automated bridge.

```
CI system               Config Repo              Cluster
────────────            ───────────────          ──────────────────
build v1.1.0  ──push──▶ Docker Hub    ◄──read──  ArgoCD syncs
push v1.1.0                │                     whatever is in
                           │                     config repo
                    config repo still
                    says v1.0.0        ← gap
```

Image Updater is that bridge — the automated connection between what the registry
contains and what the config repo declares.

---

### What ArgoCD Image Updater Is

Image Updater is a separate Kubernetes controller that runs in the `argocd`
namespace alongside ArgoCD. It is not part of ArgoCD core — it is an optional
add-on. It does two things:

1. **Polls the registry** — queries Docker Hub (or any OCI-compatible registry)
   for the list of tags on a watched image. Applies the configured strategy to
   determine if a newer tag is available.

2. **Writes back** — if a newer tag is found, writes the updated tag either
   directly to the ArgoCD Application object (fast, not GitOps) or commits a
   small override file to the Git config repo (slower, fully GitOps).

```
┌──────────────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (minikube)                                       │
│                                                                      │
│  namespace: argocd                                                   │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                                                              │    │
│  │  ArgoCD Application Controller ◄──── gitops-apps-config     │    │
│  │  (watches Git, syncs cluster)         (GitHub)              │    │
│  │                                            ▲                │    │
│  │  ArgoCD Image Updater ─────────────────────┘                │    │
│  │  (polls registry, writes Git)  commits .argocd-source-*     │    │
│  │           │                                                  │    │
│  └───────────┼──────────────────────────────────────────────────┘    │
│              │ polls every 2 minutes                                 │
└──────────────┼───────────────────────────────────────────────────────┘
               ▼
       Docker Hub Registry
       rselvantech/podinfo:v1.0.0  ← current
       rselvantech/podinfo:v1.1.0  ← detected → triggers write-back
```

---

### Write-Back Strategies — `argocd` vs `git`

**`argocd` write-back — fast but not GitOps:**
Updates the Application object in the cluster directly. Git is never touched.
If the Application is deleted and recreated, the override is lost — the
Deployment returns to whatever tag the manifest declares.

**`git` write-back — the correct GitOps approach:**
Commits a `.argocd-source-<app-name>.yaml` file to the config repo. ArgoCD
detects this commit and syncs. The tag change is in Git history — auditable,
recoverable, version-controlled.

```
git write-back produces this commit in gitops-apps-config:

.argocd-source-podinfo-image-updater-demo.yaml:
  kustomize:
    images:
    - rselvantech/podinfo:v1.1.0

ArgoCD merges this with the Application source → overrides the base image tag
Kustomize sees: image override for rselvantech/podinfo → uses v1.1.0
```

**This demo uses `git` write-back exclusively.** The `argocd` method is useful
for experimentation but is not durable and is not covered here.

---

### Update Strategies

**`semver` — semantic version ordering (used in this demo):**
Only updates to a newer version by semver rules. `v1.1.0 > v1.0.0` → update.
`v2.0.0-beta` is a pre-release — not newer than `v1.9.9` by semver rules.

```yaml
argocd-image-updater.argoproj.io/podinfo.update-strategy: semver
```

Best for: applications with proper semantic versioning. Most production cases.

**`latest` — newest by push date:**
Uses the most recently pushed tag regardless of version ordering. If `v1.0.0`
was pushed after `v1.1.0`, it would be selected.

```yaml
argocd-image-updater.argoproj.io/podinfo.update-strategy: latest
```

Best for: rolling tags with no semantic version (`main`, `edge`).

**`digest` — image content tracking:**
Tracks the SHA256 digest of a tag. When the same tag name is overwritten with
a new image (common with `latest`), the digest changes — Image Updater detects
this and updates.

```yaml
argocd-image-updater.argoproj.io/podinfo.update-strategy: digest
```

Best for: mutable tags like `latest` where the tag name never changes.

---

### The `allow-tags` Constraint — Why It Matters

Without `allow-tags`, Image Updater considers every tag on the image. For
`rselvantech/podinfo`, this includes tags like `latest`, `main`, `sha-abc123`
alongside `v1.0.0` and `v1.1.0`. With `semver` strategy, non-semver tags like
`latest` are ignored — but this is implicit behaviour. Making the constraint
explicit is safer and self-documenting:

```yaml
argocd-image-updater.argoproj.io/podinfo.allow-tags: regexp:^v1\.\d+\.\d+$
```

This constraint says: only consider tags matching `v1.x.x` where x is a number.
`v1.0.0` ✅, `v1.1.0` ✅, `v2.0.0` ✗ (major version bump — not in range),
`latest` ✗, `main` ✗.

**Why constrain to `v1.x.x` specifically:** In the demo scenario, dev tracks
patch and minor versions automatically. A major version bump (`v2.0.0`) requires
deliberate human review — it should not auto-deploy even in dev. The regexp
implements this policy declaratively in the Application annotation.

---

### Why `targetRevision: main` Is Required — The Silent Failure Mode

Image Updater's `git` write-back commits to a **branch reference** (`main`).
ArgoCD must be configured to track the same branch for it to detect the commit.

If `targetRevision: HEAD` is used:

```
ArgoCD resolves HEAD → commits to a SHA (e.g. abc123) at Application creation time
Image Updater commits .argocd-source-*.yaml to main branch
main branch tip is now commit def456

ArgoCD is still tracking SHA abc123 → never sees def456
Image Updater's commits are silently ignored
The pod is never updated
No error message
```

This is a silent failure — the logs show Image Updater committing successfully,
ArgoCD reports Synced, but the image never updates. The fix is always
`targetRevision: main` — an explicit branch name, never `HEAD`.

**Course rule:** `targetRevision: main` from Demo-10 onwards — consistent with
this requirement and with Git File/Directory generators in Demo-13.

---

### Two Separate Credentials — Why Each Is Needed

Image Updater requires two completely independent credentials for two different
operations:

| Credential | What it authenticates to | What it does | Where it lives |
|---|---|---|---|
| Registry secret | Docker Hub API | Lists available tags, reads digests | Kubernetes Secret in `argocd` namespace, with mandatory label |
| Git credential | GitHub | Commits `.argocd-source-*.yaml` to config repo | ArgoCD repo credential (`argocd repo add`) |

**The registry secret is NOT the same as the image pull secret:**

```
Image pull secret (kubelet):
  kubernetes.io/dockerconfigjson type
  Lives in the application namespace (e.g. demo15-image-updater)
  Used by kubelet to pull the container image when starting a pod
  Scoped to a namespace

Image Updater registry secret (API polling):
  Opaque type
  Lives in argocd namespace
  Used by Image Updater to call Docker Hub API: GET /v2/rselvantech/podinfo/tags/list
  Requires argocd-image-updater.argoproj.io/credentials: "true" label — mandatory
  Without this label the secret is completely ignored by Image Updater
```

**The mandatory label — the most common setup error:**

```yaml
metadata:
  labels:
    argocd-image-updater.argoproj.io/credentials: "true"  ← MANDATORY
```

Image Updater scans all Secrets in the `argocd` namespace and only uses those
with this exact label. A secret without the label is invisible to Image Updater
regardless of its name or content. Always verify the label is present.

---

### Image Updater + Kustomize — How the Write-Back File Works

When Image Updater uses `git` write-back with a Kustomize application, it
commits a file that uses Kustomize image override syntax:

```yaml
# .argocd-source-podinfo-image-updater-demo.yaml
# Auto-generated by ArgoCD Image Updater — do not edit manually
kustomize:
  images:
  - rselvantech/podinfo:v1.1.0
```

ArgoCD reads this file and merges it with the Application's Kustomize source —
overriding the image tag without touching any overlay `kustomization.yaml` or
base manifest. The override is applied on top of whatever Kustomize builds from
the overlay, then the combined output is applied to the cluster.

```
kustomize build overlays/dev/
  → Deployment: image: rselvantech/podinfo:v1.0.0  ← from base manifest

ArgoCD merges .argocd-source-podinfo-image-updater-demo.yaml override:
  image override: rselvantech/podinfo:v1.1.0

Final manifest applied to cluster:
  → Deployment: image: rselvantech/podinfo:v1.1.0  ← override wins
```

This is the production pattern: Image Updater touches only the write-back file.
Base manifests and overlay `kustomization.yaml` files remain unchanged.
The override file is the audit trail of what Image Updater did and when.

---

## Folder Structure

```
15-image-updater/
└── src/
    ├── gitops-apps-config/
    │   └── demo-15-image-updater/
    │       ├── base/
    │       │   ├── kustomization.yaml     ← lists resources, no namespace
    │       │   ├── deployment.yaml        ← image: rselvantech/podinfo:v1.0.0
    │       │   └── service.yaml
    │       └── overlays/
    │           └── dev/
    │               ├── kustomization.yaml ← namespace: demo15-image-updater, replicas: 1
    │               └── env-patch.yaml
    │   # After Image Updater runs — auto-committed by Image Updater:
    │   # .argocd-source-podinfo-image-updater-demo.yaml
    └── argocd-config/
        └── demo-15-image-updater/
            └── podinfo-image-updater-app.yaml  ← Application with Image Updater annotations
```

> **Why `gitops-apps-config` instead of `podinfo-config`:**
> From Demo-10 onwards, all application manifests live in `gitops-apps-config`
> — a single repo for all demo manifests, already registered with ArgoCD.
> The original Demo-15 used `podinfo-config` for historical reasons. This
> updated version aligns with the course convention.

> **Why only one overlay (dev):**
> Image Updater auto-deployment is appropriate for dev environments. Staging
> and prod typically use manual approval gates or tag constraints that limit
> auto-updates. Demonstrating one overlay keeps the focus on Image Updater
> mechanics without complicating the demo with multi-environment sync management.

---

## Step 1: Install ArgoCD Image Updater

Image Updater is a separate Helm chart installed into the same `argocd`
namespace as ArgoCD. It runs as an additional controller pod and communicates
with ArgoCD via its in-cluster API server address.

```bash
helm repo add argocd-image-updater https://argoproj.github.io/argocd-image-updater
helm repo update
```

**Verify the repo was added:**
```bash
helm repo list | grep image-updater
```

**Expected:**
```text
argocd-image-updater   https://argoproj.github.io/argocd-image-updater
```

**Install Image Updater:**
```bash
helm install argocd-image-updater argocd-image-updater/argocd-image-updater \
  --namespace argocd \
  --set config.argocd.insecure=true \
  --set config.argocd.serverAddress=argocd-server.argocd.svc.cluster.local
```

> **`config.argocd.insecure=true`** — our ArgoCD server uses a self-signed
> TLS certificate. This flag tells Image Updater to skip certificate verification
> when communicating with the ArgoCD API. In production with a valid TLS
> certificate signed by a trusted CA, omit this flag.
>
> **`config.argocd.serverAddress`** — the in-cluster DNS name for the ArgoCD
> server Service. Image Updater calls the ArgoCD API to discover which
> Applications have Image Updater annotations configured.

**Verify Image Updater pod is running:**
```bash
kubectl get pods -n argocd | grep image-updater
```

**Expected:**
```text
argocd-image-updater-xxxxxxxxx-xxxxx   1/1   Running   0   30s
```

**Verify startup in logs:**
```bash
kubectl logs -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd --tail=20
```

**Expected:**
```text
time="..." level=info msg="ArgoCD Image Updater started"
time="..." level=info msg="Starting image update cycle"
time="..." level=info msg="Starting metrics server"
```

---

## Step 2: Push a New Image Version to Docker Hub

We need a second image version for Image Updater to detect. We re-tag
`v1.0.0` as `v1.1.0` — no source code change needed. In production your CI
pipeline builds a genuinely different image; the Image Updater mechanism is
identical.

```bash
docker pull rselvantech/podinfo:v1.0.0
docker tag  rselvantech/podinfo:v1.0.0 rselvantech/podinfo:v1.1.0
docker push rselvantech/podinfo:v1.1.0
```

**Verify:**
```bash
docker pull rselvantech/podinfo:v1.1.0
```

**Expected:** `Already exists` or downloaded successfully.

**Verify both tags visible on Docker Hub:**

Open `hub.docker.com/r/rselvantech/podinfo/tags` — both `v1.0.0` and `v1.1.0`
must be listed. Image Updater queries this tag list on its polling cycle.

---

## Step 3: Configure Registry Credentials for Image Updater

Image Updater needs credentials to call the Docker Hub API and list available
tags. This is separate from the image pull secret used by the kubelet — they
solve different problems at different layers.

**Create the registry credential secret:**
```bash
kubectl create secret generic image-updater-dockerhub \
  --namespace argocd \
  --from-literal=credentials="rselvantech:<DOCKERHUB_TOKEN>"
```

**Apply the mandatory label — without this, Image Updater completely ignores the secret:**
```bash
kubectl label secret image-updater-dockerhub \
  --namespace argocd \
  argocd-image-updater.argoproj.io/credentials="true"
```

**Verify both the secret and the label:**
```bash
kubectl get secret image-updater-dockerhub -n argocd --show-labels
```

**Expected:**
```text
NAME                      TYPE     DATA   LABELS
image-updater-dockerhub   Opaque   1      argocd-image-updater.argoproj.io/credentials=true
```

> **Why `Opaque` type, not `kubernetes.io/dockerconfigjson`:**
> Image Updater's registry credential is for API access (tag listing via
> Docker Hub HTTP API), not for image pulling. It uses a simple
> `username:password` format stored in an Opaque secret. The kubelet pull
> secret (dockerconfigjson) is used separately by Kubernetes when pulling
> images into the node. These are two independent credential systems.

---

## Step 4: Configure Git Write-Back Credentials

Image Updater needs write access to `gitops-apps-config` to commit the
`.argocd-source-*.yaml` file. It uses the ArgoCD repo credential that was
already registered — but that credential must have `Contents: Read and write`
permission, not just read-only.

**Verify or update the GitHub PAT scope:**
- GitHub → Settings → Developer settings → Personal access tokens
- Find the token used for `gitops-apps-config`
- Confirm or update to `Contents: Read and write`

**Re-register `gitops-apps-config` with the updated credential:**
```bash
argocd repo add https://github.com/rselvantech/gitops-apps-config.git \
  --username rselvantech \
  --password <GITHUB_PAT> \
  --upsert
```

> **`--upsert`** — updates the existing credential entry. Without this flag,
> the command fails if the repo is already registered. Always use `--upsert`
> when updating credentials for an already-registered repo.

**Verify:**
```bash
argocd repo list
```

**Expected:**
```text
TYPE  NAME                REPO                                              STATUS
git   gitops-apps-config  https://github.com/rselvantech/gitops-apps-config  Successful
```

---

## Step 5: Build the Application Manifests — Kustomize Base + Overlay

The `podinfo` application uses the same Kustomize base + overlay pattern from
Demo-14. The base declares `podinfo:v1.0.0` as the starting image tag. The
overlay sets the namespace and replica count for the dev environment. Image
Updater will write an override file that replaces the image tag without touching
either the base or the overlay.

```bash
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src
mkdir -p gitops-apps-config && cd gitops-apps-config

git init
git branch -M main
git remote add origin \
  https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

### Base manifests

**`demo-15-image-updater/base/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

**`demo-15-image-updater/base/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  # no namespace — set by overlay kustomization.yaml
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
        - name: dockerhub-secret      # pull secret — created in Step 6
      containers:
        - name: podinfo
          image: rselvantech/podinfo:v1.0.0
          # ↑ Image Updater will write an override — this line is NOT edited directly.
          # The .argocd-source-*.yaml write-back file overrides this at sync time.
          ports:
            - containerPort: 9898
          env:
            - name: PODINFO_UI_COLOR
              value: "#3d8eb9"
            - name: PODINFO_UI_MESSAGE
              value: "Image Updater Demo — watch this tag change automatically"
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "100m"
              memory: "64Mi"
```

**`demo-15-image-updater/base/service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  # no namespace — set by overlay kustomization.yaml
spec:
  selector:
    app: podinfo
  ports:
    - port: 9898
      targetPort: 9898
```

### Overlay — dev environment

**`demo-15-image-updater/overlays/dev/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base         # use 'resources', not deprecated 'bases'

namespace: demo15-image-updater   # sets namespace on all resources

namePrefix: dev-                  # podinfo → dev-podinfo

commonLabels:
  env: dev

replicas:
  - name: podinfo
    count: 1

patches:
  - path: env-patch.yaml
```

**`demo-15-image-updater/overlays/dev/env-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo              # base name — before namePrefix is applied
spec:
  template:
    metadata:
      labels:
        env: dev
```

**Verify the overlay builds correctly:**
```bash
kustomize build demo-15-image-updater/overlays/dev/
```

**Verify:**
```bash
kustomize build demo-15-image-updater/overlays/dev/ \
  | grep -E "namespace:|image:|name: dev-"
```

**Expected:**
```text
  namespace: demo15-image-updater
  image: rselvantech/podinfo:v1.0.0
  name: dev-podinfo
```

**Push:**
```bash
git add demo-15-image-updater/
git commit -m "feat: add demo-15 podinfo kustomize base and dev overlay"
git push origin main
```

---

## Step 6: Create the Namespace and Image Pull Secret

The namespace and pull secret must exist before ArgoCD can sync the Deployment.

```bash
kubectl create namespace demo15-image-updater
```

**Create the image pull secret in the application namespace:**
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=rselvantech \
  --docker-password=<DOCKERHUB_TOKEN> \
  --namespace=demo15-image-updater
```

> **This secret (`dockerhub-secret`) is separate from the Image Updater registry
> secret (`image-updater-dockerhub` in `argocd` namespace) created in Step 3.**
> This one is used by the kubelet to pull the `podinfo` image when starting the
> pod. The other is used by Image Updater to call the Docker Hub API and list tags.
> Same registry, two different credential consumers, two different secrets.

**Verify:**
```bash
kubectl get ns demo15-image-updater
kubectl get secret dockerhub-secret -n demo15-image-updater
```

**Expected:**
```text
NAME                    STATUS   AGE
demo15-image-updater    Active   5s

NAME               TYPE                             DATA
dockerhub-secret   kubernetes.io/dockerconfigjson   1
```

---

## Step 7: Create the Application CRD with Image Updater Annotations

All Image Updater behaviour is configured through annotations on the Application
CRD. No separate CRD or configuration file is needed. The annotations tell Image
Updater which images to watch, which strategy to use, and how to write back.

```bash
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src
mkdir -p argocd-config/demo-15-image-updater && cd argocd-config

git init
git branch -M main
git remote add origin \
  https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

**`demo-15-image-updater/podinfo-image-updater-app.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-image-updater-demo
  namespace: argocd
  annotations:
    # ── Image Updater annotations ──────────────────────────────────────────────
    #
    # 1. image-list: defines which image(s) to watch
    #    Format: <alias>=<registry>/<image>
    #    'podinfo' is the alias — used in all other annotation keys below
    #    'rselvantech/podinfo' is the Docker Hub image path
    argocd-image-updater.argoproj.io/image-list: podinfo=rselvantech/podinfo

    # 2. update-strategy: how to select the new tag
    #    semver → use semantic version ordering (v1.1.0 > v1.0.0)
    argocd-image-updater.argoproj.io/podinfo.update-strategy: semver

    # 3. allow-tags: constrain which tags are evaluated
    #    Without this, Image Updater considers ALL tags on the image —
    #    including 'latest', 'main', 'sha-abc123' which are not semver.
    #    This regexp limits evaluation to v1.x.x tags only.
    #    v2.0.0 would NOT be selected — major bumps require human review.
    argocd-image-updater.argoproj.io/podinfo.allow-tags: regexp:^v1\.\d+\.\d+$

    # 4. write-back-method: how the updated tag is persisted
    #    'git' → commits .argocd-source-<app>.yaml to the config repo
    #    This is the only truly GitOps approach — the tag is in Git history
    argocd-image-updater.argoproj.io/write-back-method: git

    # 5. git-repository: which repo to commit the write-back file to
    #    Must match a repo registered with ArgoCD credentials
    argocd-image-updater.argoproj.io/git-repository: \
      https://github.com/rselvantech/gitops-apps-config.git

    # 6. pull-secret: registry credentials for tag listing
    #    Format: secret:<namespace>/<secret-name>#<key-in-secret>
    #    References the secret created in Step 3 (with mandatory label)
    argocd-image-updater.argoproj.io/podinfo.pull-secret: \
      secret:argocd/image-updater-dockerhub#credentials
    # ───────────────────────────────────────────────────────────────────────────

spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    # ↑ CRITICAL: must be a branch name — never HEAD
    # Image Updater commits to the 'main' branch.
    # If targetRevision: HEAD, ArgoCD resolves HEAD to a commit SHA at creation
    # time and never detects Image Updater's new commits on main.
    # Result: Image Updater logs show successful commits, ArgoCD shows Synced,
    # but the pod image never updates. Silent failure — no error message.
    path: demo-15-image-updater/overlays/dev
    # ArgoCD finds kustomization.yaml → runs kustomize build → applies manifests
    # Image Updater's write-back file is merged on top of kustomize output
  destination:
    server: https://kubernetes.default.svc
    namespace: demo15-image-updater
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false   # namespace created manually in Step 6
```

**Push and apply:**
```bash
git add demo-15-image-updater/podinfo-image-updater-app.yaml
git commit -m "feat: add demo-15 Application CRD with Image Updater annotations"
git push origin main

kubectl apply -f demo-15-image-updater/podinfo-image-updater-app.yaml
```

**Verify the Application was created:**
```bash
argocd app get podinfo-image-updater-demo --refresh
```

**Expected:**
```text
Sync Status:   Synced
Health Status: Healthy
```

**Verify the initial deployment uses `v1.0.0`:**
```bash
kubectl get deployment dev-podinfo -n demo15-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected:**
```text
rselvantech/podinfo:v1.0.0
```

**Access the podinfo UI to confirm v1.0.0:**
```bash
kubectl port-forward svc/dev-podinfo -n demo15-image-updater 9898:9898
```

Open `http://localhost:9898` — the version shown is `v1.0.0`.

---

## Step 8: Observe Image Updater Detecting and Writing

With `v1.1.0` already pushed to Docker Hub and the Application deployed at
`v1.0.0`, Image Updater will detect the newer tag on its next polling cycle
(every 2 minutes by default) and commit the override file to `gitops-apps-config`.

**Watch Image Updater logs in real time:**
```bash
kubectl logs -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd -f --tail=50
```

**Expected — Image Updater discovers `v1.1.0` and commits:**
```text
time="..." level=info  msg="Starting image update cycle, considering 1 annotated application(s)"
time="..." level=info  msg="Processing application podinfo-image-updater-demo"
time="..." level=info  msg="Fetching available tags and digest for image rselvantech/podinfo"
time="..." level=info  msg="Found 2 tags for image rselvantech/podinfo" tags="[v1.0.0 v1.1.0]"
time="..." level=info  msg="Latest image according to semver: rselvantech/podinfo:v1.1.0"
time="..." level=info  msg="Updating image rselvantech/podinfo:v1.0.0 to rselvantech/podinfo:v1.1.0"
time="..." level=info  msg="Committing 1 parameter update(s) for application podinfo-image-updater-demo"
time="..." level=info  msg="Successfully updated Git"
```

> Image Updater polls every 2 minutes by default. If you do not see output
> immediately, wait up to 2 minutes. The `-f` flag streams logs in real time.

**Verify the write-back file was committed to `gitops-apps-config`:**
```bash
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src/gitops-apps-config
git pull origin main
cat .argocd-source-podinfo-image-updater-demo.yaml
```

**Expected:**
```yaml
kustomize:
  images:
  - rselvantech/podinfo:v1.1.0
```

> This file uses Kustomize image override syntax — the same format as
> `spec.source.kustomize.images` in an Application CRD. ArgoCD merges this
> with the Kustomize build output, replacing `rselvantech/podinfo:v1.0.0`
> in the rendered Deployment with `rselvantech/podinfo:v1.1.0`.

**Watch ArgoCD detect the Git commit and sync:**
```bash
watch -n 3 'argocd app get podinfo-image-updater-demo | grep -E "Sync|Health|Image"'
```

**Expected:**
```text
Sync Status:   OutOfSync    ← ArgoCD detected the new commit from Image Updater
Sync Status:   Synced       ← ArgoCD applied the updated manifest
Health Status: Healthy
```

---

## Step 9: Verify the Full Chain — Registry to Running Pod

Confirm the complete automation chain ran end-to-end correctly.

**Verify the running image is now `v1.1.0`:**
```bash
kubectl get deployment dev-podinfo -n demo15-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected:**
```text
rselvantech/podinfo:v1.1.0
```

**Verify via the podinfo UI:**
```bash
kubectl port-forward svc/dev-podinfo -n demo15-image-updater 9898:9898
```

Open `http://localhost:9898` — the version shown is now `v1.1.0`. The image
changed from the automated Git commit by Image Updater, not from any manual
YAML edit.

**Verify the write-back file is the source of the image override:**
```bash
argocd app get podinfo-image-updater-demo -o yaml \
  | grep -A5 "kustomize:"
```

**Expected:**
```yaml
kustomize:
  images:
  - rselvantech/podinfo:v1.1.0
```

This is ArgoCD showing the active Kustomize image override — sourced from the
write-back file committed by Image Updater.

**Verify the base manifest was NOT modified:**
```bash
grep "image:" \
  gitops-apps-config/demo-15-image-updater/base/deployment.yaml
```

**Expected:**
```text
          image: rselvantech/podinfo:v1.0.0
```

The base still declares `v1.0.0`. The override file is what changed. Image
Updater never modifies manifests directly — it only writes the
`.argocd-source-*.yaml` override file.

**Verify the full chain visually:**
```
Docker Hub: rselvantech/podinfo:v1.1.0  ← you pushed this
     │
     │  Image Updater detected (semver: v1.1.0 > v1.0.0)
     ▼
gitops-apps-config:                     ← Image Updater committed this
  .argocd-source-podinfo-image-updater-demo.yaml
    kustomize.images: [rselvantech/podinfo:v1.1.0]
     │
     │  ArgoCD detected Git commit
     ▼
Cluster: dev-podinfo Deployment         ← ArgoCD synced this
  image: rselvantech/podinfo:v1.1.0
     │
     │  Pod rolled out
     ▼
http://localhost:9898 shows v1.1.0      ← visible in UI
```

---

## Verify Final State

```bash
# Image Updater running
kubectl get pods -n argocd | grep image-updater

# Application Synced and Healthy
argocd app get podinfo-image-updater-demo

# Running image is v1.1.0
kubectl get deployment dev-podinfo -n demo15-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Write-back file exists in Git (pull first)
git -C gitops-apps-config pull origin main --quiet
ls -la gitops-apps-config/.argocd-source-podinfo-image-updater-demo.yaml
cat gitops-apps-config/.argocd-source-podinfo-image-updater-demo.yaml

# Base manifest still at v1.0.0 — unchanged by Image Updater
grep "image:" \
  gitops-apps-config/demo-15-image-updater/base/deployment.yaml
```

---

## Cleanup

```bash
# Delete the Application
kubectl delete app podinfo-image-updater-demo -n argocd

# Delete namespace (ArgoCD never deletes namespaces)
kubectl delete ns demo15-image-updater

# Remove the write-back file from gitops-apps-config
cd gitops-labs/argo-cd-basics-to-prod/15-image-updater/src/gitops-apps-config
git pull origin main
git rm .argocd-source-podinfo-image-updater-demo.yaml 2>/dev/null || true
git commit -m "cleanup: remove demo-15 image updater write-back file"
git push origin main

# Uninstall Image Updater (keep if using in Project-01)
helm uninstall argocd-image-updater -n argocd

# Remove registry credential secret
kubectl delete secret image-updater-dockerhub -n argocd

# Restore gitops-apps-config credential to read-only PAT if desired
# argocd repo add ... --upsert with read-only token
```

---

## Key Concepts Summary

**Image Updater closes the CI → GitOps gap**
It is the automated bridge between what your registry contains and what your
config repo declares. Without it, someone must manually update the image tag
after every CI build.

**`git` write-back is the only true GitOps approach**
The `argocd` write-back is fast but ephemeral — lost on Application recreation.
The `git` write-back commits a `.argocd-source-*.yaml` file to the config repo,
making every tag update auditable, version-controlled, and recoverable.

**`targetRevision: main` is required — `HEAD` causes silent failure**
Image Updater commits to a branch. If ArgoCD tracks a SHA (what `HEAD` resolves
to), it never detects Image Updater's commits. The logs look correct but the pod
never updates. Always use an explicit branch name.

**The mandatory credentials label is the most common setup error**
`argocd-image-updater.argoproj.io/credentials: "true"` on the registry secret
is required. Without it, Image Updater ignores the secret entirely — no warning,
no error, just no tag updates.

**Two credentials, two separate purposes**
The registry secret (Opaque, in `argocd` namespace) enables tag listing via the
Docker Hub API. The image pull secret (dockerconfigjson, in the app namespace)
enables the kubelet to pull the image. They are independent and must both be present.

**Image Updater never modifies base manifests**
It writes only to the `.argocd-source-*.yaml` override file. Base and overlay
files remain unchanged. The override file is merged by ArgoCD on top of
Kustomize output at sync time.

**`allow-tags` makes the strategy explicit and safe**
Without `allow-tags`, Image Updater evaluates all tags including mutable ones
like `latest`. A regexp constraint ensures only the intended tag pattern is
considered. Major version bumps can be excluded to prevent accidental
auto-deployment of breaking changes.

---

## Commands Reference

```bash
# Install Image Updater
helm install argocd-image-updater argocd-image-updater/argocd-image-updater \
  --namespace argocd \
  --set config.argocd.insecure=true \
  --set config.argocd.serverAddress=argocd-server.argocd.svc.cluster.local

# Registry credential secret (mandatory label required)
kubectl create secret generic image-updater-dockerhub \
  --namespace argocd \
  --from-literal=credentials="user:<token>"
kubectl label secret image-updater-dockerhub -n argocd \
  argocd-image-updater.argoproj.io/credentials="true"

# Watch Image Updater logs
kubectl logs -l app.kubernetes.io/name=argocd-image-updater -n argocd -f

# Push a new image version
docker tag rselvantech/podinfo:v1.0.0 rselvantech/podinfo:v1.1.0
docker push rselvantech/podinfo:v1.1.0

# Re-register repo with write access
argocd repo add https://github.com/rselvantech/gitops-apps-config.git \
  --username rselvantech --password <PAT> --upsert

# Check running image
kubectl get deployment dev-podinfo -n demo15-image-updater \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check write-back file
cat .argocd-source-podinfo-image-updater-demo.yaml

# Verify kustomize output pre-sync
kustomize build demo-15-image-updater/overlays/dev/
```

---

## Lessons Learned

**1. `targetRevision: main` is required — `HEAD` causes silent failure**
Image Updater commits to a branch (`main`). If `targetRevision: HEAD` is set,
ArgoCD resolves it to a commit SHA at Application creation time and never detects
subsequent commits. Image Updater logs show success, ArgoCD shows Synced, but
the pod image never changes. There is no error message. Always use an explicit
branch name — `main`, never `HEAD`. This is also the course-wide rule from Demo-10.

**2. The mandatory label is the most common Image Updater setup error**
`argocd-image-updater.argoproj.io/credentials: "true"` must be on the registry
secret. Without it, Image Updater scans the `argocd` namespace for secrets and
ignores any without this label — silently. No error, no warning, no tag updates.
Always verify the label with `kubectl get secret --show-labels`.

**3. Two separate credentials for two separate purposes**
The image pull secret (kubelet, app namespace, dockerconfigjson) and the registry
API credential (Image Updater, argocd namespace, Opaque) solve different problems.
Both must exist. Reusing or confusing them causes failures at different points
in the chain.

**4. `allow-tags` prevents accidental major version auto-deployment**
Without a tag constraint, Image Updater considers all tags. A major version bump
(`v2.0.0`) may be selected by semver if it is newer — deploying a breaking change
automatically. Use `regexp:^v1\.\d+\.\d+$` to limit auto-updates within a major
version. Major bumps require human review.

**5. Image Updater never modifies base manifests — only the write-back file**
The `.argocd-source-*.yaml` file is the only thing Image Updater writes. Base
manifests and overlay `kustomization.yaml` files are never touched. The base
image tag stays at its original value — the write-back file is the override.
Verify this with `grep "image:" base/deployment.yaml` after Image Updater runs.

**6. Image Updater + Kustomize write-back uses Kustomize image override syntax**
The write-back file contains `kustomize.images: [image:tag]` — the same format
as `spec.source.kustomize.images` in an Application CRD. ArgoCD merges this
into the Kustomize build output. Using Image Updater with Kustomize overlays
is the production pattern — not raw manifests.

**7. Use `resources:` not `bases:` in overlay kustomization.yaml**
Consistent with Demo-14. `bases:` is deprecated since Kustomize v3.2 and removed
in v4+. Always reference the base with `resources: - ../../base`.

**8. `--upsert` is required when re-registering an already-registered repo**
`argocd repo add` fails if the repo already exists. `--upsert` updates the
existing credential entry. Always include `--upsert` when updating credentials
for repos registered in prior demos.

**9. ArgoCD never deletes namespaces — delete manually after cleanup**
Consistent with Demo-13, Demo-14, Demo-15. `CreateNamespace=true` creates the
namespace. Deleting an Application or ApplicationSet does not remove it.
Always `kubectl delete ns` explicitly.

---

## What's Next

**Demo-16: ApplicationSet Progressive Sync — Safe Multi-Environment Rollouts**
Demo-16 is a standalone demo that builds its own Kustomize base, overlays,
and ApplicationSet in a separate `demo-16-progressive-sync/` folder. It adds
`strategy: RollingSync` to control rollout order — dev syncs first, must be
Healthy, then staging, then prod. A bad change stops at dev and never reaches
production. Demo-15's resources are untouched.