# Goals App — Kubernetes Setup (Prerequisite for Demo-10)

## Overview

This README deploys the Goals App to Kubernetes, verifies it works end-to-end,
and tears it down — giving you a confirmed working baseline before Demo-10 adds
sync hooks on top of it.

**What this README does:**
- Creates `gitops-apps-config` — the single private GitHub repo used for all
  application manifests from Demo-10 onwards (replaces per-demo config repos)
- Registers `gitops-apps-config` with ArgoCD once — reused in all future demos
- Pushes all Goals App Kubernetes manifests into `demo-10-sync-hooks-goals-app/`
- Use Sealed Secrets to encrypt a Kubernetes secret. Store the encrypted YAML locally and commit it to Git.
- ArgoCD syncs the `SealedSecret` from Git. The `Sealed Secrets controller` decrypts it and creates the `mongodb-secret` automatically.
- Deploys the full three-tier app via ArgoCD and verifies CRUD works in a browser
- Tears down cleanly so Demo-10 starts from a known state

**What this README does NOT cover:**
- Building or modifying source code (covered in Docker Demo-14)
- ArgoCD sync hooks (covered in Demo-10)

> **Complete Docker [Demo-14](https://github.com/rselvantech/docker/blob/main/docker-practical-guide-2025/14-goals-app-production/README.md) first.** Images `rselvantech/goals-frontend:v1.0.0`
> and `rselvantech/goals-backend:v1.0.0` must exist in Docker Hub before
> proceeding.

---

## About `gitops-apps-config` Repo — One Repo for All Demos

Earlier demos created a separate private repo per application (`podinfo-config`,
`goals-app-config`). From Demo-10 onwards, a single repo `gitops-apps-config`
holds all application manifests, organised by demo subdirectory:

```
gitops-apps-config/
├── demo-10-sync-hooks-goals-app/   ← this README + Demo-10
├── demo-11-sync-waves/             ← Demo-11 onwards adds here
├── demo-12-app-of-apps/
└── ...
```

**Benefits:**
- Register with ArgoCD once — every future demo uses the same registration
- Single place to see all application manifests across the course
- Mirrors how teams structure real multi-app GitOps repos

Each Application CRD points to its own subdirectory path within
`gitops-apps-config` — ArgoCD only syncs the files it is pointed at, so
demos are fully isolated from each other.

---

## How the Goals Application Works — Kubernetes

### Modules and How They Interwork

The Goals App runs as three Kubernetes Deployments with three Services in the
`goals-app` namespace. ArgoCD manages all resources from `gitops-apps-config`.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Namespace: goals-app                                                   │
│                                                                         │
│  ┌─────────────────────┐    ┌──────────────────┐    ┌───────────────┐   │
│  │  goals-frontend pod │    │ goals-backend pod│    │  mongodb pod  │   │
│  │  nginx:1.25         │    │ node:18-alpine   │    │  mongo:6.0    │   │
│  │  port 3000          │    │ port 80          │    │  port 27017   │   │
│  │                     │    │                  │    │               │   │
│  │  static files       │    │  REST API        │    │  data in      │   │
│  │  + API proxy        │──▶| /goals CRUD      │──▶ |   PVC         │   │
│  └─────────────────────┘    └──────────────────┘    └───────────────┘   │
│           │                          │                                  │
│  ┌────────▼────────┐     ┌───────────▼──────┐  ┌──────────────────┐     │
│  │goals-frontend   │     │goals-backend-svc │  │    mongodb       │     │
│  │svc              │     │ClusterIP         │  │    ClusterIP     │     │
│  │ClusterIP :3000  │     │:80               │  │    :27017        │     │
│  └────────┬────────┘     └──────────────────┘  └──────────────────┘     │
│           │                                                             │
└───────────┼─────────────────────────────────────────────────────────────┘
            │ kubectl port-forward :3000
            ▼
       Your laptop
       http://localhost:3000
```

**Kubernetes DNS:**
CoreDNS runs as a Service in the `kube-system` namespace. Every pod's
`/etc/resolv.conf` points to CoreDNS and has the namespace's search domains:

```
nameserver 10.96.0.10          ← CoreDNS ClusterIP
search goals-app.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Short names like `goals-backend-svc` are expanded to the full FQDN
`goals-backend-svc.goals-app.svc.cluster.local` via the search domains.
This resolution works for any process in the pod — including nginx.

---

### nginx's Dual Role — Static File Server and API Proxy

nginx inside `goals-frontend` serves two completely different purposes
depending on the URL path:

```
Incoming request to goals-frontend pod
              │
              ▼
         nginx :3000
              │
              ├── /goals or /goals/* ──▶ proxy_pass http://goals-backend-svc:80
              │                          (API proxy → backend pod)
              │
              └── everything else ─────▶ /usr/share/nginx/html/
                                          (static React build files)
```

**Role 1 — Static File Server (`location /`):**

The compiled React app — one `index.html`, chunked JavaScript bundles, CSS —
was produced by `npm run build` during `docker build` and lives inside the
nginx image at `/usr/share/nginx/html/`. nginx serves these as plain files:

```
GET /                     → index.html       (React entry point)
GET /static/js/main.js    → main.js          (React bundle)
GET /static/css/main.css  → main.css
GET /favicon.ico          → favicon.ico
GET /dashboard            → index.html       (try_files fallback — React Router)
```

No network calls for static files — nginx reads from local disk inside the pod.
Fast, no backend dependency, no DNS involved.

**Role 2 — API Proxy (`location /goals`):**

The React JavaScript bundle runs in the browser. API calls use a relative URL
(`/goals`) which the browser sends to the same host and port the page loaded
from (`localhost:3000`). The port-forwarded connection delivers this to the
`goals-frontend` pod. nginx intercepts and proxies it:

```
POST /goals  (from browser)
  → nginx receives on port 3000
  → location /goals matches
  → proxy_pass http://goals-backend-svc:80/goals
  → CoreDNS resolves goals-backend-svc → 10.98.64.226 (Service ClusterIP)
  → kube-proxy routes to goals-backend pod
  → backend processes request → MongoDB
  → response flows back through nginx to browser
```

---

### How `${BACKEND_HOST}` Works at Container Start

`BACKEND_HOST=goals-backend-svc` is set in the Deployment's `env:` block.
When the container starts, the nginx Docker image's built-in entrypoint runs
`envsubst` on `/etc/nginx/templates/default.conf.template`:

```
Pod starts
    ↓
nginx entrypoint reads: BACKEND_HOST=goals-backend-svc
    ↓
envsubst replaces ${BACKEND_HOST}:
  proxy_pass http://${BACKEND_HOST}:80;
         ↓
  proxy_pass http://goals-backend-svc:80;   ← literal hostname
    ↓
Written to: /etc/nginx/conf.d/default.conf
    ↓
nginx starts, reads config
nginx resolves "goals-backend-svc":
  → /etc/resolv.conf: nameserver 10.96.0.10
  → CoreDNS: goals-backend-svc.goals-app.svc.cluster.local → 10.98.64.226
  → kube-proxy: 10.98.64.226 → goals-backend pod IP
```

nginx uses the pod's `/etc/resolv.conf` — the same resolver every other
process in the pod uses. No hardcoded DNS IPs, no special configuration.

---

### Complete Message Flow — Adding a Goal

```
Your laptop (browser)
    │
    │  kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
    │
    │  GET http://localhost:3000        ← React app loads
    │  GET http://localhost:3000/goals  ← relative URL from App.js
    ▼
goals-frontend pod: nginx (port 3000)
    │
    │  location /goals → proxy_pass http://goals-backend-svc:80
    │  nginx is INSIDE the cluster — K8s DNS resolves goals-backend-svc
    ▼
goals-backend pod (Node.js, port 80)
    │
    │  mongodb://rselvantech:passWD@mongodb:27017/course-goals?authSource=admin
    ▼
mongodb pod (mongo:6.0, port 27017)
    │
    ▼
PersistentVolumeClaim: goals-db-data
```

**Step 1: Developer runs port-forward**
```
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000

Creates a tunnel:
  localhost:3000 on your laptop
    → Kubernetes API server
    → goals-frontend-svc (ClusterIP 10.105.116.0:3000)
    → goals-frontend pod :3000
```

**Step 2: Browser loads the React app**
```
User types: http://localhost:3000

Browser: GET http://localhost:3000/
  → port-forward tunnel → goals-frontend pod
  → nginx: path "/" → try_files → serve index.html from disk
  → Browser receives HTML

Browser: GET http://localhost:3000/static/js/main.js
  → nginx: serve from /usr/share/nginx/html/static/js/
  → React bundle downloads and executes in browser

React initialises:
  → calls GET /goals to load existing goals
  → browser: GET http://localhost:3000/goals
     → port-forward → goals-frontend pod
     → nginx: path "/goals" → proxy_pass http://goals-backend-svc:80/goals
        → CoreDNS: goals-backend-svc → 10.98.64.226
        → kube-proxy: 10.98.64.226 → goals-backend pod :80
        → Node.js: mongoose query db.goals.find()
        → MongoDB pod :27017 → reads from PVC
        → returns {"goals": []}
     → nginx passes JSON response back through port-forward to browser
  → React renders empty goals list
```

**Step 3: User types a goal and clicks Add**
```
User: types "Learn ArgoCD sync hooks" → clicks Add Goal

React: addGoalHandler() runs
  Browser: POST http://localhost:3000/goals
           body: {"text": "Learn ArgoCD sync hooks"}

  → port-forward tunnel → goals-frontend pod :3000
  → nginx: path "/goals" → proxy_pass http://goals-backend-svc:80
     → DNS: goals-backend-svc → CoreDNS → 10.98.64.226 (ClusterIP)
     → kube-proxy: iptables rule → goals-backend pod IP (e.g. 10.244.0.8:80)
     → Node.js receives POST /goals
     → goalText = req.body.text = "Learn ArgoCD sync hooks"
     → new Goal({text: goalText}).save()
     → mongoose: mongodb://rselvantech:passWD@mongodb:27017/course-goals
        → DNS: mongodb → CoreDNS → 10.110.71.21 (ClusterIP)
        → kube-proxy → mongodb pod :27017
        → MongoDB writes to /data/db on PVC (goals-db-data)
     → returns 201: {"goal": {"id": "abc123", "text": "..."}}
  → nginx passes response back to browser
  → React: adds goal to UI state → renders in list
```

**Step 4: Verify data persisted — delete and recreate the backend pod**
```
kubectl delete pod -l app=goals-backend -n goals-app
  → Kubernetes recreates the pod (Deployment desired state = 1)
  → New pod starts, connects to MongoDB (same PVC, same data)

Browser: GET /goals
  → same flow as Step 2
  → goal "Learn ArgoCD sync hooks" still returned
  → PVC is the source of truth — not the pod
```

---

### DNS Resolution — Full Picture

```
goals-frontend pod
  /etc/resolv.conf:
    nameserver 10.96.0.10    ← CoreDNS
    search goals-app.svc.cluster.local svc.cluster.local cluster.local

nginx resolves "goals-backend-svc":
  1. Tries: goals-backend-svc.goals-app.svc.cluster.local ✅
  2. CoreDNS returns: 10.98.64.226 (ClusterIP)
  3. kube-proxy intercepts traffic to 10.98.64.226
  4. Routes to actual pod IP via iptables rules

goals-backend pod
  /etc/resolv.conf: (same nameserver, same search domains)

Node.js resolves "mongodb":
  1. Tries: mongodb.goals-app.svc.cluster.local ✅
  2. CoreDNS returns: 10.110.71.21 (ClusterIP)
  3. kube-proxy routes to mongodb pod IP

MongoDB pod
  /etc/resolv.conf: (same)
  writes data to: /data/db → PersistentVolumeClaim → host disk on minikube
```

**Why ClusterIP and not pod IP directly?**
Pod IPs change every time a pod restarts. Services have stable ClusterIPs for
their entire lifetime. nginx resolves `goals-backend-svc` to the ClusterIP
once — if the backend pod restarts and gets a new IP, kube-proxy silently
updates its routing rules. nginx keeps using the same ClusterIP and traffic
continues to flow without any nginx restart or re-resolution.

---

### Why the Browser Never Talks to the Backend Directly

The browser runs on the developer's laptop — outside the cluster, outside any
Kubernetes network. It cannot resolve `goals-backend-svc` or any Kubernetes
internal DNS name.

```
Developer's laptop (browser)
    ↕ only via: port-forward to localhost:3000

Kubernetes cluster (invisible to browser)
    goals-frontend ←→ goals-backend ←→ mongodb
    (all internal ClusterIP communication)
```

The relative URL `/goals` is the key. The browser sends it to `localhost:3000`
— the only address it knows. nginx (running inside the cluster) receives the
request, resolves `goals-backend-svc` using Kubernetes DNS, and proxies it.
The browser never needs to know how many backend services exist or where they are.

---


## Secrets in GitOps — The Core Tension and the Solution Used Here

### The problem

Git should be the single source of truth for everything ArgoCD manages. But
Kubernetes Secret resources contain base64-encoded values — not encrypted.
Anyone with read access to the Git repository can decode them instantly:

```bash
echo "cGFzc1dE" | base64 -d
# passWD
```

Committing a Secret YAML to Git is equivalent to committing plaintext passwords.
This is the fundamental tension in GitOps: **everything in Git, but secrets
must never be in Git as plaintext**.

### What ArgoCD recommends

ArgoCD does not have built-in secret management and deliberately does not
enforce one approach. Its documentation lists the options and their trade-offs.
The recommended approaches in order of complexity are:

| Approach | How it works | Requires |
|---|---|---|
| **Imperative `kubectl create secret`** | Create secret manually, outside Git. ArgoCD never manages it. | Nothing extra |
| **Sealed Secrets** | Encrypt secret locally → commit encrypted form to Git → controller decrypts in cluster | `kubeseal` CLI + controller in cluster |
| **SOPS + Age** | Encrypt secret values in-place in YAML → commit encrypted YAML → ArgoCD plugin decrypts | `sops`, `age`, ArgoCD plugin config |
| **External Secrets Operator** | Store secrets in Vault/AWS/GCP → ESO pulls them into K8s at sync time | External secret manager + ESO operator |

ArgoCD explicitly cautions against using manifest-generation plugins to inject
secrets at sync time — it increases risk because ArgoCD caches generated
manifests in plaintext in Redis.

### What every demo used until now — and why it is incomplete

In earlier demos, secrets were created imperatively using kubectl create secret. That approach worked because those secrets were external to the application (e.g., Docker registry access or ArgoCD repo credentials).

In this case, however, the MongoDB secret is part of the Goals App itself. This makes imperative creation unsuitable.

```
kubectl create secret → exists only in the cluster
                     → deleted when namespace is deleted
                     → no recovery path if lost
                     → completely outside ArgoCD's awareness
                     → must be recreated manually every time
```

Since the secret is now an application dependency, it must be defined declaratively and managed through Git. Imperative creation falls outside the GitOps workflow, meaning ArgoCD cannot track, reconcile, or restore it—making it operationally unreliable for this use case.

### The solution used in this demo — Sealed Secrets

Sealed Secrets (by Bitnami) closes this gap without requiring any external
secret manager. It works in two parts:

**1. `kubeseal` CLI (on your laptop)** — encrypts a regular Kubernetes Secret
using the cluster's public key. The encrypted result is a `SealedSecret` CRD.

**2. Sealed Secrets controller (in the cluster)** — watches for `SealedSecret`
resources, decrypts them using its private key, and creates native Kubernetes
Secrets. Only this controller can decrypt — not ArgoCD, not anyone with Git
access.

```bash
Your laptop:
  kubectl create secret ... --dry-run=client -o yaml
    → piped to kubeseal
    → produces sealed-mongodb-secret.yaml (encrypted, safe to commit)

Git (gitops-apps-config):
  sealed-mongodb-secret.yaml committed ← safe, encrypted

ArgoCD syncs:
  SealedSecret applied to cluster

Sealed Secrets controller:
  Decrypts SealedSecret → creates mongodb-secret (native K8s Secret)

Backend pod:
  Reads mongodb-secret ← works exactly as before
```

**The key property:** The `SealedSecret` YAML in Git is safe to read by anyone.
It can only be decrypted by the controller running in the specific cluster it
was sealed for. This is called **cluster-scoped encryption**.

**Trade-off to understand:** A SealedSecret is tied to the cluster it was
created for (specifically, to the controller's private key). If the cluster is
destroyed and a new one created, you need to either:
- Re-seal the secret against the new controller (run `kubeseal` again), or
- Back up the controller's private key and restore it to the new cluster

For a learning environment like minikube, re-sealing on cluster recreation is
perfectly acceptable. For production, backing up the controller key is standard
practice.

---

## Prerequisites

- ✅ Completed Docker [Demo-14](https://github.com/rselvantech/docker/blob/main/docker-practical-guide-2025/14-goals-app-production/README.md) — both images in Docker Hub
- ✅ ArgoCD running on minikube (default profile)
- ✅ ArgoCD CLI installed and logged in as admin
- ✅ `kubectl` available
- ✅ GitHub PAT with `Contents: Read-only` for `gitops-apps-config` and
  `Contents: Read and write` for any repo ArgoCD commits to

**Verify Prerequisites:**

### 1. Both images exist in Docker Hub
```bash
docker pull rselvantech/goals-backend:v1.0.0
docker pull rselvantech/goals-frontend:v1.0.0
```

**Expected:** Both pull successfully or `Already up to date`.

**If either fails:** Complete Docker Demo-14 first.

### 2. ArgoCD pods running
```bash
kubectl get pods -n argocd
```

### 3. ArgoCD UI is accessible
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `http://localhost:8080` — you should see the ArgoCD login page.

### 4. ArgoCD CLI is installed and logged in
```bash
argocd version --client
argocd login localhost:8080 --username admin --insecure
```

**Expected:**
```text
argocd: v3.x.x
'admin:login' logged in successfully
```

**Expected:** All pods `Running` and `1/1` Ready.

### 5. ArgoCD CLI logged in
```bash
argocd account get-user-info
```

**Expected:**
```text
Logged In: true
Username:  admin
```

---

## Folder Structure (What Gets Created)

```
gitops-apps-config (GitHub — new private repo)
└── demo-10-sync-hooks-goals-app/
    ├── namespace.yaml
    ├── mongodb-pvc.yaml
    ├── mongodb-deployment.yaml
    ├── mongodb-service.yaml
    ├── sealed-mongodb-secret.yaml      ← SealedSecret yaml
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── frontend-deployment.yaml
    └── frontend-service.yaml

rselvantech/argocd-config (GitHub — existing)
└── demo-10-sync-hooks/
    └── goals-app-demo.yaml             ← Application CRD for this demo
```

Local working directory:
```
gitops-labs/argo-cd-basics-to-prod/10-sync-phases-and-hooks/src/
├── gitops-apps-config/                  ← git init → remote: gitops-apps-config
│   └── demo-10-sync-hooks-goals-app/
└── argocd-config/                       ← git init → remote: argocd-config
    └── demo-10-sync-hooks/
        └── goals-app-demo.yaml
```

---

## Step 1: Install Sealed Secrets Controller

The Sealed Secrets controller must be installed in the cluster before any
secrets can be sealed. Install it via Helm into the `kube-system` namespace.

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --version 2.18.5 \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller
```

> **`fullnameOverride=sealed-secrets-controller`** — sets a predictable
> service name that `kubeseal` looks for by default when fetching the
> public key. Without this, the service name is auto-generated and you
> would need to pass `--controller-name` to every `kubeseal` command.

**Verify the controller is running:**
```bash
kubectl get pods -n kube-system | grep sealed-secrets
```

**Expected:**
```text
sealed-secrets-controller-xxxxxxxxx-xxxxx   1/1   Running   0   30s
```

**Wait for it to be fully ready before proceeding:**
```bash
kubectl rollout status deployment/sealed-secrets-controller -n kube-system
```

**Expected:**
```text
deployment "sealed-secrets-controller" successfully rolled out
```

### Install `kubeseal` CLI

`kubeseal` is the client-side tool that encrypts secrets using the controller's
public key.


**Linux:**
```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/tags \
  | grep 'name' | head -1 | cut -d'"' -f4)

curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION#v}-linux-amd64.tar.gz"

tar -xvzf kubeseal-${KUBESEAL_VERSION#v}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

rm -f kubeseal*
```

**Verify:**
```bash
kubeseal --version
```

**Expected:**
```text
kubeseal version: 0.36.6     #Version used in this demo
```

**Test that `kubeseal` can reach the controller:**
```bash
kubeseal --fetch-cert
```

**Expected:** A PEM certificate block is printed — the controller's public key
that `kubeseal` uses to encrypt secrets.
```text
-----BEGIN CERTIFICATE-----
MIIEzT...
-----END CERTIFICATE-----
```

If this fails, the controller is not running or `kubeseal` cannot reach it.
Check `kubectl get pods -n kube-system | grep sealed-secrets`.

---

## Step 2: Create `gitops-apps-config` Repo and Register with ArgoCD

Go to GitHub → **New repository**:
- Name: `gitops-apps-config`
- Visibility: **Private**
- Add a README: **yes**
- Click **Create repository**

**Create GitHub PAT:** 
 - Create a new PAT on GitHub and add `gitops-apps-config` with `Contents: Read-only` permission.


**Initialise Local Repo**

```bash
cd 10-sync-phases-and-hooks/src
mkdir gitops-apps-config && cd gitops-apps-config

git init
git branch -M main
git remote add origin \
  https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/gitops-apps-config.git
git pull origin main
```

**Register `gitops-apps-config` with ArgoCD:**

```bash
argocd repo add https://github.com/rselvantech/gitops-apps-config.git \
  --username rselvantech \
  --password <GITHUB_PAT> \
  --name gitops-apps-config
```

**Verify:**
```bash
argocd repo list
```

**Expected:** `gitops-apps-config` listed with `Successful` status.

```text
TYPE  NAME                REPO                                                      STATUS
git   gitops-apps-config  https://github.com/rselvantech/gitops-apps-config.git     Successful
```

---

## Step 3: Create All Kubernetes Manifests

Create the `demo-10-sync-hooks-goals-app/` directory and all manifest files.

```bash
cd 10-sync-phases-and-hooks/src/gitops-apps-config
mkdir -p demo-10-sync-hooks-goals-app

touch demo-10-sync-hooks-goals-app/namespace.yaml
touch demo-10-sync-hooks-goals-app/mongodb-pvc.yaml
touch demo-10-sync-hooks-goals-app/mongodb-deployment.yaml
touch demo-10-sync-hooks-goals-app/mongodb-service.yaml
touch demo-10-sync-hooks-goals-app/sealed-mongodb-secret.yaml
touch demo-10-sync-hooks-goals-app/backend-deployment.yaml
touch demo-10-sync-hooks-goals-app/backend-service.yaml
touch demo-10-sync-hooks-goals-app/frontend-deployment.yaml
touch demo-10-sync-hooks-goals-app/frontend-service.yaml
```

### `demo-10-sync-hooks-goals-app/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: goals-app
```

### `demo-10-sync-hooks-goals-app/mongodb-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: goals-db-data
  namespace: goals-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### `demo-10-sync-hooks-goals-app/mongodb-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: goals-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:6.0
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
      volumes:
        - name: mongodb-data
          persistentVolumeClaim:
            claimName: goals-db-data
```

### `demo-10-sync-hooks-goals-app/mongodb-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: goals-app
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

### `demo-10-sync-hooks-goals-app/sealed-mongodb-secret.yaml`

The MongoDB secret contains four keys that two different components read:

| Key | Read by | Purpose |
|---|---|---|
| `MONGO_INITDB_ROOT_USERNAME` | MongoDB container | Creates the root user on first start |
| `MONGO_INITDB_ROOT_PASSWORD` | MongoDB container | Root user password |
| `MONGODB_USERNAME` | Backend app | Authenticates to MongoDB |
| `MONGODB_PASSWORD` | Backend app | Authentication password |

Both pairs must have identical values — the backend authenticates as the root user
using `authSource=admin`.

**Step 1 — Create the secret locally (dry-run, not applied to cluster):**

```bash
kubectl create secret generic mongodb-secret \
  --namespace goals-app \
  --from-literal=MONGODB_USERNAME=rselvantech \
  --from-literal=MONGODB_PASSWORD=passWD \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=rselvantech \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=passWD \
  --dry-run=client \
  --output yaml \
  | kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml \
  > demo-10-sync-hooks-goals-app/sealed-mongodb-secret.yaml
```

> **Do NOT manually apply `sealed-mongodb-secret.yaml` with `kubectl apply`.**
> ArgoCD will apply it during sync. Let ArgoCD manage it — that is the point
> of putting it in Git.

**What each part does:**

```
kubectl create secret ... --dry-run=client --output yaml
  → Generates the Secret YAML without applying it to the cluster
  → Outputs to stdout

| kubeseal ...
  → Reads the Secret YAML from stdin
  → Fetches the controller's public key from the cluster
  → Encrypts each secret value with that key
  → Outputs a SealedSecret YAML

> sealed-mongodb-secret.yaml
  → Writes the encrypted SealedSecret to the file
  → This file is SAFE to commit to Git
```

**Inspect the generated file:**
```bash
cat demo-10-sync-hooks-goals-app/sealed-mongodb-secret.yaml
```

**Expected — encrypted values, not plaintext:**
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mongodb-secret
  namespace: goals-app
spec:
  encryptedData:
    MONGO_INITDB_ROOT_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    MONGO_INITDB_ROOT_USERNAME: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    MONGODB_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    MONGODB_USERNAME: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
  template:
    metadata:
      name: mongodb-secret
      namespace: goals-app
```

The `encryptedData` values are ciphertext — decoding them reveals nothing.
This file is safe to commit to `gitops-apps-config`.

ArgoCD will pick up the `SealedSecret` automatically on the next sync because
`gitops-apps-config` is already registered and the Application CRD points to
`demo-10-sync-hooks-goals-app/`. Also No changes to the Application CRD are needed.


### `demo-10-sync-hooks-goals-app/backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goals-backend
  namespace: goals-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: goals-backend
  template:
    metadata:
      labels:
        app: goals-backend
    spec:
      containers:
        - name: goals-backend
          image: rselvantech/goals-backend:v1.0.0
          ports:
            - containerPort: 80
          env:
            - name: MONGODB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGODB_USERNAME
            - name: MONGODB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGODB_PASSWORD
            - name: MONGODB_HOST
              value: "mongodb"
            - name: MONGODB_DATABASE
              value: "course-goals"
```

> **`MONGODB_HOST=mongodb`** — matches the Kubernetes Service name in
> `mongodb-service.yaml` above. The backend connection string becomes:
> `mongodb://rselvantech:passWD@mongodb:27017/course-goals?authSource=admin`

### `demo-10-sync-hooks-goals-app/backend-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: goals-backend-svc
  namespace: goals-app
spec:
  selector:
    app: goals-backend
  ports:
    - port: 80
      targetPort: 80
```

### `demo-10-sync-hooks-goals-app/frontend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goals-frontend
  namespace: goals-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: goals-frontend
  template:
    metadata:
      labels:
        app: goals-frontend
    spec:
      containers:
        - name: goals-frontend
          image: rselvantech/goals-frontend:v1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: BACKEND_HOST
              value: "goals-backend-svc"
```

> **`BACKEND_HOST=goals-backend-svc`** — nginx reads this at container start
> via `envsubst` and sets its `proxy_pass` to `http://goals-backend-svc:80`.
> When The browser calls `/goals` (relative URL), nginx proxies to the backend
> Service internally using Kubernetes DNS.

### `demo-10-sync-hooks-goals-app/frontend-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: goals-frontend-svc
  namespace: goals-app
spec:
  selector:
    app: goals-frontend
  ports:
    - port: 3000
      targetPort: 3000
```

**Push all manifests to `gitops-apps-config`:**
```bash
git add demo-10-sync-hooks-goals-app/
git commit -m "feat: add demo-10 goals-app manifests (no hooks — prereq baseline)"
git push origin main
```

**Verify on GitHub:** Go to `github.com/rselvantech/gitops-apps-config` —
you should see `demo-10-sync-hooks-goals-app/` with all nine YAML files.

---

## Step 4: Create the Application CRD

```bash
cd 10-sync-phases-and-hooks/src
mkdir -p argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin \
  https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase

mkdir -p demo-10-sync-hooks 
touch demo-10-sync-hooks/goals-app-demo.yaml
```

**Create `demo-10-sync-hooks/goals-app-demo.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: goals-app-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/gitops-apps-config.git
    targetRevision: main
    path: demo-10-sync-hooks-goals-app
  destination:
    server: https://kubernetes.default.svc
    namespace: goals-app
#  syncPolicy:        
#    syncOptions:
#      - CreateNamespace=true    <- Git repo has ns yaml
```

> **`targetRevision: main`** — explicit branch name, not `HEAD`. Required for
> consistency and for future Image Updater compatibility (Demo-15).

> **No `automated` sync** — manual sync only. This is a baseline test — we sync
> once, verify it works, then tear down. No automated reconciliation needed.

**Push and apply:**
```bash
git add demo-10-sync-hooks/goals-app-demo.yaml
git commit -m "feat: add demo-10 Application CRD"
git push origin main

kubectl apply -f demo-10-sync-hooks/goals-app-demo.yaml
```

**Verify Application CRD was created:**
```bash
argocd app get goals-app-demo
```

**Expected:**
```text
Sync Status:   OutOfSync
Health Status: Missing
```

---

## Step 5: Sync and Verify Deployment

```bash
argocd app sync goals-app-demo
```

**Watch pods come up:**
```bash
kubectl get pods -n goals-app -w
```

**Expected — all three pods running:**
```text
NAME                                READY   STATUS    RESTARTS
mongodb-xxxxxxxxx-xxxxx             1/1     Running   0
goals-backend-xxxxxxxxx-xxxxx       1/1     Running   0
goals-frontend-xxxxxxxxx-xxxxx      1/1     Running   0
```

**Verify all resources created:**
```bash
kubectl get all,pvc -n goals-app
```

**Expected:**
```text
NAME                                    READY   STATUS    RESTARTS
pod/goals-backend-xxxxxxxxx-xxxxx       1/1     Running   0
pod/goals-frontend-xxxxxxxxx-xxxxx      1/1     Running   0
pod/mongodb-xxxxxxxxx-xxxxx             1/1     Running   0

NAME                        TYPE        PORT(S)
service/goals-backend-svc   ClusterIP   80/TCP
service/goals-frontend-svc  ClusterIP   3000/TCP
service/mongodb             ClusterIP   27017/TCP

NAME                            READY   UP-TO-DATE
deployment.apps/goals-backend   1/1     1
deployment.apps/goals-frontend  1/1     1
deployment.apps/mongodb         1/1     1

NAME                                   STATUS   CAPACITY
persistentvolumeclaim/goals-db-data    Bound    1Gi
```

**Verify backend connected to MongoDB:**
```bash
kubectl logs -l app=goals-backend -n goals-app | grep -E "CONNECTED|FAILED"
```

**Expected:**
```text
CONNECTED TO MONGODB at mongodb/course-goals
```

**Verify nginx resolved `BACKEND_HOST` correctly:**
```bash
kubectl exec -n goals-app \
  $(kubectl get pod -n goals-app -l app=goals-frontend -o name) \
  -- cat /etc/nginx/conf.d/default.conf | grep "set \$backend"
```

**Expected — `goals-backend-svc` substituted in:**
```text
        set $backend    http://goals-backend-svc:80;
```

**Verify ArgoCD shows Synced and Healthy:**
```bash
argocd app get goals-app-demo
```

**Expected:**
```text
Sync Status:   Synced
Health Status: Healthy
```

---

## Step 6: Verify Sealed Mangodb secret - Decrypted and Deployed 

> **No `kubectl create secret` needed.** The MongoDB credentials are stored as
> a `SealedSecret` in `gitops-apps-config/demo-10-sync-hooks-goals-app/`.
> ArgoCD syncs it and the Sealed Secrets controller decrypts it into
> `mongodb-secret` automatically.


**How the sealed secret decryption happens, When ArgoCD syncs the application:**

```
ArgoCD applies sealed-mongodb-secret.yaml to the cluster
    ↓
Sealed Secrets controller detects the new SealedSecret resource
    ↓
Controller decrypts encryptedData using its private key
    ↓
Controller creates native Kubernetes Secret named mongodb-secret
    with the original four plaintext keys
    ↓
Backend pod reads mongodb-secret as normal — no change needed in Deployment YAML
```

**Verify the controller decrypted it after sync:**
```bash
kubectl get secret mongodb-secret -n goals-app
```

**Expected:**
```text
NAME             TYPE     DATA   AGE
mongodb-secret   Opaque   4      10s
```

```bash
kubectl get sealedsecret mongodb-secret -n goals-app
```

**Expected:**
```text
NAME             STATUS   SYNCED   AGE
mongodb-secret             True     10s
```

`SYNCED: True` confirms the controller successfully decrypted the SealedSecret
and created the native Secret.


---

## Step 7: End-to-End Browser Test

```bash
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
```

Open `http://localhost:3000` in your browser.

**Test 1 — UI loads:**
The Goals Tracker page appears with an empty list. ✅

**Test 2 — Add a goal:**
Type `"Goals App Demo test"` → click **Add Goal**.
Goal appears in the list. ✅

**Test 3 — Verify API went through nginx:**
```bash
kubectl logs -l app=goals-frontend -n goals-app | grep goals
```

**Expected — nginx proxied the request:**
```text
127.x.x.x - "POST /goals HTTP/1.1" 201
127.x.x.x - "GET /goals HTTP/1.1" 200
```

**Test 4 — Verify data stored in MongoDB:**
```bash
kubectl exec -n goals-app \
  $(kubectl get pod -n goals-app -l app=mongodb -o name) \
  -- mongosh \
  --username rselvantech \
  --password passWD \
  --authenticationDatabase admin \
  course-goals \
  --eval "db.goals.find().pretty()"
```

**Expected — the goal you added is in MongoDB:**
```text
[
  {
    _id: ObjectId('...'),
    text: 'Goals App Demo test',
    __v: 0
  }
]
```

**Test 5 — Reload page (persistence):**
Refresh the browser — goal still appears (loaded from MongoDB). ✅

**Test 6 — Delete the goal:**
Click the goal in the list — it disappears. ✅

**Stop port-forward:** `Ctrl+C`

---

## Step 8: Verify Final State

```bash
# Application synced and healthy
argocd app get goals-app-demo

# All three pods running
kubectl get pods -n goals-app

# gitops-apps-config registered
argocd repo list | grep gitops-apps-config

# MongoDB secret has four keys
kubectl get secret mongodb-secret -n goals-app

# PVC is Bound
kubectl get pvc -n goals-app
```

---

## Step 9: Tear Down

Demo-10 redeploys the Goals App from scratch and adds hooks progressively.
Delete everything now so Demo-10 starts from a clean state.

```bash
# Delete the ArgoCD Application
kubectl delete app goals-app-demo -n argocd

# Delete the namespace (removes all pods, services, PVC, and data)
kubectl delete namespace goals-app
```

**Verify everything is gone:**
```bash
kubectl get namespace goals-app
```

**Expected:**
```text
Error from server (NotFound): namespaces "goals-app" not found
```

```bash
argocd app list | grep goals-app-demo
```

**Expected:** No output (application deleted).

> **Do NOT delete:**
> - `gitops-apps-config` GitHub repo — Demo-10 adds hooks manifests here
> - ArgoCD registration for `gitops-apps-config` — reused in Demo-10
> - `argocd-config` GitHub repo — Demo-10 adds its Application CRD here
>
> **Do NOT delete the manifests from `gitops-apps-config`.** Demo-10 adds
> hook manifests alongside these — not replacing them.
>
> **Do NOT delete the Sealed Secrets controller** — it is reused in Demo-10
> and all future demos that use `gitops-apps-config`. The controller's private
> key must persist across demos or secrets will need to be re-sealed.
>
> **The `SealedSecret` in `gitops-apps-config` stays committed.** When
> Demo-10 recreates the namespace, ArgoCD syncs the `SealedSecret` and the
> controller automatically recreates `mongodb-secret`. No manual
> `kubectl create secret` step is needed in Demo-10.

---

## Ready for Demo-10 Checklist

```bash
# Images exist in Docker Hub
docker pull rselvantech/goals-backend:v1.0.0
docker pull rselvantech/goals-frontend:v1.0.0

# gitops-apps-config registered with ArgoCD
argocd repo list | grep gitops-apps-config

# goals-app namespace does NOT exist (torn down)
kubectl get namespace goals-app

# goals-app-demo Application does NOT exist
argocd app list | grep goals-app-demo

# Sealed Secrets controller is running
kubectl get pods -n kube-system | grep sealed-secrets

# SealedSecret is in gitops-apps-config
git -C gitops-apps-config log --oneline | grep sealed

# kubeseal can reach the controller
kubeseal --fetch-cert > /dev/null && echo "kubeseal OK"
```

**All checks pass — start Demo-10.**

---

## What Carries Forward into Demo-10

| Item | Status | Action in Demo-10 |
|---|---|---|
| `gitops-apps-config` GitHub repo | ✅ Exists | Add hook YAML files under `demo-10-sync-hooks-goals-app/hooks/` |
| `gitops-apps-config` ArgoCD registration | ✅ Registered | No action — already registered |
| All manifest YAMLs | ✅ In `gitops-apps-config` | Demo-10 adds hooks alongside them |
| `goals-app` namespace | ❌ Deleted | Demo-10 Step 1 recreates it |
| `goals-app-demo` Application | ❌ Deleted | Demo-10 creates `sync-phase-hook-demo-goals-app` instead |