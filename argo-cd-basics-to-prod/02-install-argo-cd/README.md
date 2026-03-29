# Demo-02: Installing ArgoCD & Familiarising with CLI and UI

## Overview

Demo-01 built the mental model — what GitOps is, why it exists, and how
ArgoCD's components work together. This demo puts that into practice.

We install ArgoCD on a local Kubernetes cluster using minikube and Helm,
verify every component is running, access the UI, and walk through the
ArgoCD CLI commands used throughout every subsequent demo. By the end, your
environment is fully ready for Demo-03 where we deploy the first application.

**What you'll learn:**
- How to start a local Kubernetes cluster using minikube
- How to install ArgoCD using Helm — the production-aligned approach
- What each ArgoCD pod does in the running cluster
- How to access the ArgoCD UI via port-forwarding
- How to navigate every section of the ArgoCD UI
- How to install and use the ArgoCD CLI
- The essential CLI commands used throughout this course

**What you'll do:**
- Install prerequisites: Docker Desktop, kubectl, Helm, minikube
- Start a single-node minikube cluster using the default profile
- Add the ArgoCD Helm repository and install ArgoCD in the `argocd` namespace
- Verify all ArgoCD pods and services are running
- Port-forward and access the ArgoCD UI
- Change the default admin password
- Walk through UI: Applications, Settings, Repositories, Clusters, Projects
- Install the ArgoCD CLI and log in
- Run essential CLI commands: app list, repo list, cluster list, version

---

## Prerequisites

- ✅ Docker Desktop installed and running (Mac/Windows) or Docker Engine (Linux)
- ✅ `kubectl` installed
- ✅ `helm` installed (v3.x)
- ✅ `minikube` installed
- ✅ IDE installed (VS Code recommended)

**Install links:**
- Docker Desktop: https://www.docker.com/products/docker-desktop
- kubectl: https://kubernetes.io/docs/tasks/tools/
- Helm: https://helm.sh/docs/intro/install/
- minikube: https://minikube.sigs.k8s.io/docs/start/
- VS Code: https://code.visualstudio.com/

**Verify all prerequisites are installed:**
```bash
docker --version
kubectl version --client
helm version
minikube version
```

**Expected — all four commands return version strings without errors.**

---

## Concepts

### Why Helm for ArgoCD Installation

ArgoCD can be installed two ways — raw manifests (`kubectl apply -f install.yaml`)
or Helm chart. Helm is the better choice for production and learning because:

- Version is explicit and pinned — you always know exactly what you installed
- Upgrades are managed — `helm upgrade` handles rolling updates cleanly
- Configuration is centralised — one `values.yaml` for all customisation
- Rollback is built in — `helm rollback` if an upgrade goes wrong

Throughout this course we use `helm install` for ArgoCD. This matches
how most production teams install it.

---

### What Minikube Is

Minikube runs a single-node Kubernetes cluster locally. It is the standard
local cluster tool used throughout this entire course. All demos use the
default minikube profile named `minikube` — no custom profile name is needed.

```bash
minikube start     # start the cluster (creates if not exists)
minikube stop      # stop without deleting — resumes from here next time
minikube delete    # delete the cluster and all data entirely
minikube status    # check current state
```

Minikube persists state between stops and starts. If you run `minikube stop`
at the end of a session and `minikube start` the next day, your ArgoCD
installation and all deployed applications are restored exactly as you left them.

---

### Port-Forwarding — How We Access ArgoCD Locally

ArgoCD's API server service (`argocd-server`) runs as a `ClusterIP` service
inside the cluster — not exposed externally by default. To access it from your
browser or CLI, you use `kubectl port-forward` which creates a tunnel from your
local machine to the service inside the cluster.

```
Your browser (localhost:8080)
    │
    │  kubectl port-forward tunnel
    ▼
argocd-server service (port 443) inside minikube cluster
    │
    ▼
argocd-server pod
```

In production, ArgoCD is typically exposed via an Ingress or LoadBalancer.
Port-forwarding is the local development equivalent.

---

### The `argocd` Namespace

All ArgoCD components are installed in a dedicated namespace named `argocd`.
This namespace is used consistently across every demo in this course. When
any command references ArgoCD pods, services, or secrets — it always uses
`-n argocd`.

---

## Step 1: Start the Minikube Cluster

```bash
minikube start
```

This starts the default `minikube` profile — a single-node cluster. No
additional flags needed. Minikube selects the best available driver
(Docker, VirtualBox, or HyperKit) automatically.

**Verify:**
```bash
minikube status
kubectl get nodes
```

**Expected:**
```text
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   60s   v1.xx.x
```

---

## Step 2: Add the ArgoCD Helm Repository

Artifact Hub (https://artifacthub.io) is the central registry for Helm charts.
Search for `argocd` → filter by Official → select `argo-cd` to find the chart.

```bash
# Add the official ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm

# Update the local index — good habit before any install
helm repo update

# Verify the repo was added
helm repo list
```

**Expected:**
```text
NAME    URL
argo    https://argoproj.github.io/argo-helm
```

**Search available ArgoCD chart versions:**
```bash
helm search repo argo/argo-cd --versions
```

This lists all available versions. We install a pinned version for
predictability — not `latest`.

---

## Step 3: Install ArgoCD

```bash
# Create the ArgoCD namespace — all ArgoCD components live here
kubectl create namespace argocd

# Install ArgoCD — pin the version for predictability
helm install argocd argo/argo-cd \
  --version 7.8.23 \
  --namespace argocd
```

> **Why pin the version?** In production, unpredictable upgrades cause outages.
> Pinning the version means you know exactly what is installed and any upgrade
> is a deliberate decision. Use `helm search repo argo/argo-cd --versions` to
> find the latest stable version and update the `--version` flag accordingly.

**Wait for all pods to be ready:**
```bash
kubectl get pods -n argocd --watch
```

**Expected — all pods `Running` and `1/1` Ready:**
```text
NAME                                                READY   STATUS    RESTARTS
argocd-application-controller-0                     1/1     Running   0
argocd-applicationset-controller-xxxxxxxxx-xxxxx    1/1     Running   0
argocd-dex-server-xxxxxxxxx-xxxxx                   1/1     Running   0
argocd-notifications-controller-xxxxxxxxx-xxxxx     1/1     Running   0
argocd-redis-xxxxxxxxx-xxxxx                        1/1     Running   0
argocd-repo-server-xxxxxxxxx-xxxxx                  1/1     Running   0
argocd-server-xxxxxxxxx-xxxxx                       1/1     Running   0
```

**What each pod does — mapped to Demo-01 architecture:**

| Pod | Component | Role |
|---|---|---|
| `argocd-server` | API Server | Entry point for UI and CLI |
| `argocd-repo-server` | Repository Server | Fetches and renders manifests from Git |
| `argocd-application-controller` | Application Controller | Compares desired vs live, reconciles |
| `argocd-redis` | Redis | Caching layer |
| `argocd-applicationset-controller` | ApplicationSet Controller | Dynamic app generation (Demo-12) |
| `argocd-dex-server` | Dex Server | SSO/OIDC integration |
| `argocd-notifications-controller` | Notifications | Slack/webhook alerts on sync events |

**Verify services:**
```bash
kubectl get svc -n argocd
```

**Expected — note `argocd-server` on ports 80 and 443:**
```text
NAME                                      TYPE        PORT(S)
argocd-server                             ClusterIP   80/TCP,443/TCP
argocd-repo-server                        ClusterIP   8081/TCP
argocd-redis                              ClusterIP   6379/TCP
argocd-applicationset-controller          ClusterIP   7000/TCP
argocd-dex-server                         ClusterIP   5556/TCP,5557/TCP,5558/TCP
```

**Verify Helm installation:**
```bash
helm list -n argocd
```

**Expected:**
```text
NAME     NAMESPACE  REVISION  STATUS    CHART          APP VERSION
argocd   argocd     1         deployed  argo-cd-7.8.23 v2.x.x
```

---

## Step 4: Get the Initial Admin Password

ArgoCD generates a random initial admin password stored in a Kubernetes Secret
in the `argocd` namespace:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

**Copy this password** — you will use it to log in for the first time.

> This secret exists until you explicitly delete it. After changing the
> password, delete the secret:
> `kubectl delete secret argocd-initial-admin-secret -n argocd`

---

## Step 5: Access the ArgoCD UI

Port-forward the ArgoCD server service:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

> Leave this running in a dedicated terminal tab throughout all demo work.
> Use `8080:443` — not `8080:80`. Port 443 is the HTTPS port serving
> the full ArgoCD API and UI. Port 80 only redirects to 443.

Open `http://localhost:8080` in your browser.

**You will see a security warning** — this is expected. ArgoCD uses a
self-signed TLS certificate by default. Click **Advanced → Proceed to
localhost (unsafe)**.

**Log in:**
- Username: `admin`
- Password: paste the password from Step 4

**Change the default password immediately:**
```
Top right avatar → User Info → Update Password
  Current password: paste initial password
  New password: choose a memorable password
  Confirm: repeat
→ Save
```

> Always change the default password before doing any real work. The initial
> password is stored in a recoverable Kubernetes Secret — anyone with cluster
> access can read it. A changed password is not stored in any Secret.

---

## Step 6: Familiarise with the ArgoCD UI

Walk through every section of the UI now — before starting Demo-03.
Understanding the layout makes every subsequent demo easier to follow.

### Applications page (home)
```
http://localhost:8080/applications
```
Where all ArgoCD-managed Applications appear. Currently empty — the first
Application is created in Demo-03. Each app card shows:
- Application name and sync status (Synced / OutOfSync)
- Health status (Healthy / Degraded / Missing / Progressing)
- Source repo and path
- Destination cluster and namespace

### Settings → Repositories
```
Settings → Repositories
```
Where connected Git repositories are listed with connection status (`Successful`
or `Failed`). This is where credentials for private repos are configured —
covered in Demo-04. Currently empty.

### Settings → Clusters
```
Settings → Clusters
```
Lists all Kubernetes clusters ArgoCD can deploy to. The local minikube cluster
(`https://kubernetes.default.svc`) is registered automatically when ArgoCD
starts — you will see it here. Additional clusters are registered here for
multi-cluster management.

### Settings → Projects
```
Settings → Projects
```
Lists AppProjects — governance boundaries that control what Applications can
deploy, where, and from which repos — covered in Demo-09. The `default` project
exists automatically and allows everything.

### Settings → Accounts
```
Settings → Accounts
```
Local user accounts. The `admin` account is the only one by default. For team
use, SSO via Dex is the production approach.

---

## Step 7: Install the ArgoCD CLI

The ArgoCD CLI (`argocd`) is used throughout every subsequent demo for status
checks, sync operations, and repo management.

**Mac (Homebrew):**
```bash
brew install argocd
```

**Linux:**
```bash
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

**Windows (via Chocolatey):**
```bash
choco install argocd-cli
```

**Verify:**
```bash
argocd version --client
```

**Expected:**
```text
argocd: v3.x.x
```

---

## Step 8: Log in via ArgoCD CLI

```bash
argocd login localhost:8080 \
  --username admin \
  --password <your-new-password> \
  --insecure
```

> `--insecure` skips TLS certificate verification — required because minikube
> uses a self-signed certificate. In production with a valid certificate,
> omit this flag.

**Expected:**
```text
'admin:login' logged in successfully
Context 'localhost:8080' updated
```

**Verify you are logged in:**
```bash
argocd account get-user-info
```

**Expected:**
```text
Logged In: true
Username: admin
Issuer: argocd
Groups:
```

**Check the saved context:**
```bash
argocd context
```

**Expected:**
```text
CURRENT  NAME            SERVER
*        localhost:8080  localhost:8080
```

> The context is saved locally — you do not need to log in again until the
> session token expires. At the start of each subsequent demo session, just run:
> `argocd login localhost:8080 --username admin --insecure`

---

## Step 9: Essential CLI Commands — Familiarisation

Run each command now to understand what it returns. These are used throughout
every subsequent demo in this course.

### Application commands
```bash
# List all applications
argocd app list

# Get detailed status of one application (used after every sync)
argocd app get <app-name>

# Get status and trigger immediate Git refresh (does NOT apply changes)
argocd app get <app-name> --refresh

# See diff between Git desired state and live cluster state
argocd app diff <app-name>

# Manually sync an application (apply Git state to cluster)
argocd app sync <app-name>

# View sync history
argocd app history <app-name>

# Delete an application
argocd app delete <app-name>
```

### Repository commands
```bash
# List connected repositories
argocd repo list

# Add a private repository (HTTPS)
argocd repo add https://github.com/org/repo.git \
  --username <username> \
  --password <PAT>

# Add a private repository (SSH)
argocd repo add git@github.com:org/repo.git \
  --ssh-private-key-path ~/.ssh/argocd-deploy-key

# Remove a repository
argocd repo rm https://github.com/org/repo.git
```

### Cluster commands
```bash
# List registered clusters (minikube is auto-registered)
argocd cluster list

# Add a remote cluster
argocd cluster add <kubectl-context-name>
```

### Account and version commands
```bash
# Check current logged-in user
argocd account get-user-info

# Check ArgoCD server and client version
argocd version

# List saved CLI contexts
argocd context
```

> Most commands return empty lists now — no applications or repos configured
> yet. The important thing is confirming each command works without errors.

---

## Verify Final State

```bash
# minikube cluster running
minikube status

# All ArgoCD pods running in argocd namespace
kubectl get pods -n argocd

# ArgoCD Helm release installed
helm list -n argocd

# CLI version confirmed
argocd version --client

# Logged in
argocd account get-user-info

# Local cluster auto-registered
argocd cluster list
```

**Expected — minikube cluster in argocd cluster list:**
```text
SERVER                          NAME        VERSION  STATUS
https://kubernetes.default.svc  in-cluster  1.xx     Successful
```

---

## Cleanup

This minikube cluster is used in **all subsequent demos**. Do NOT stop or
delete it between demos unless you intend to take a long break.

**Between sessions — stop without losing state:**
```bash
minikube stop    # stops the cluster, preserves all data
minikube start   # resumes exactly where you left off
```

**After completing the course — full teardown:**
```bash
minikube delete    # removes the cluster and all data
```

---

## Key Concepts Summary

**Helm is the production-aligned install method**
It pins the version, manages upgrades cleanly, and centralises configuration.
Always pin the Helm chart version — never install `latest` in a shared
environment.

**All ArgoCD components live in the `argocd` namespace**
Every command referencing ArgoCD pods, secrets, services, or configs uses
`-n argocd`. This is consistent across every demo in this course.

**Default minikube profile — no custom name needed**
All demos use `minikube start` with the default `minikube` profile. No
`--profile` flag is needed anywhere in this course.

**Port 8080:443 — always use 443, not 80**
`argocd-server` serves the full API and UI on port 443. Port 80 redirects
to 443. Always port-forward to 443 for full functionality.

**Initial admin password — change it immediately**
The initial password is stored in `argocd-initial-admin-secret` — readable
by anyone with cluster access. Change it on first login. Delete the secret
after changing: `kubectl delete secret argocd-initial-admin-secret -n argocd`

**`--insecure` is for local minikube development only**
Required because minikube uses a self-signed TLS certificate. In production
with a valid certificate, omit this flag entirely.

**`argocd app get <n> --refresh` — not `argocd app refresh`**
The `refresh` subcommand does not exist. The `--refresh` flag on `app get`
triggers an immediate re-evaluation of the Git source without waiting for
the ~3 minute poll cycle.

**minikube stop preserves state**
`minikube stop` does not delete anything. Your ArgoCD installation and all
deployed applications are restored on the next `minikube start`. Use
`minikube delete` only for a full teardown.

---

## Commands Reference

```bash
# Minikube cluster management
minikube start
minikube stop
minikube delete
minikube status
kubectl get nodes
kubectl get pods -n argocd
kubectl get svc -n argocd

# Helm — ArgoCD install
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm repo list
helm search repo argo/argo-cd --versions
helm install argocd argo/argo-cd --version <version> --namespace argocd
helm list -n argocd
helm uninstall argocd -n argocd

# ArgoCD initial password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d

# Delete initial password secret after changing password
kubectl delete secret argocd-initial-admin-secret -n argocd

# Port-forward (keep running in dedicated terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ArgoCD CLI login (run at the start of each session)
argocd login localhost:8080 --username admin --insecure

# ArgoCD CLI — essential commands
argocd version --client
argocd account get-user-info
argocd context
argocd app list
argocd app get <app-name>
argocd app get <app-name> --refresh
argocd app diff <app-name>
argocd app sync <app-name>
argocd app history <app-name>
argocd repo list
argocd cluster list
```

---

## Lessons Learned

**1. Always pin the Helm chart version**
`helm install` without `--version` installs latest — unpredictable in teams
and CI pipelines. Always specify the version and update it deliberately.

**2. Change the initial admin password as the first action**
`argocd-initial-admin-secret` is readable by anyone with cluster access.
Change the password on first login, then delete the secret.

**3. Keep port-forward running in a dedicated terminal**
Kill the port-forward and both the UI and CLI stop working. Dedicate one
terminal tab to port-forwarding throughout all demo work.

**4. `argocd app get --refresh` is the correct refresh command**
`argocd app refresh <n>` does not exist. Use `argocd app get <n> --refresh`.

**5. The local cluster is auto-registered**
`https://kubernetes.default.svc` is registered automatically when ArgoCD
starts — no manual registration needed. External clusters require
`argocd cluster add`.

**6. minikube stop is safe — minikube delete is destructive**
`minikube stop` preserves everything. `minikube delete` removes the cluster
and all installed components including ArgoCD. Use stop between sessions,
delete only at end of course.

---

## What's Next

**Demo-03: Application CRD — First Deployment**
Create your first ArgoCD Application CRD — the resource that tells ArgoCD
which Git repository to watch and which cluster to deploy to. Deploy the
guestbook application from a public repository and observe the full sync
lifecycle for the first time.
