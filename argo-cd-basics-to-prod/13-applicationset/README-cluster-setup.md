# Demo-13 (Part 1): Multi-Cluster Setup — Two Clusters, One ArgoCD

## Overview

Before exploring ApplicationSets, this README covers the foundational infrastructure
this demo depends on: a two-cluster Kubernetes environment, network connectivity
between both clusters, ArgoCD cluster registration, and a verified working
deployment to both the local and remote cluster.

Completing this README gives you a clean, verified multi-cluster foundation so
that Demo-13 Part 2 can focus entirely on ApplicationSet generators without
setup interruptions.

> **Prerequisite for Demo-13 Part 2:** Complete this README first. Part 2 assumes
> both clusters are registered, labelled, and verified working via a test
> deployment.

**What you'll learn:**
- Why multi-cluster Kubernetes architectures exist in production
- How ArgoCD manages multiple clusters from a single control plane
- How cluster registration works — what is created, where, and why
- How `in-cluster` auto-registration works and why `STATUS=Unknown` is normal
- How real-world cluster connectivity (VPC peering, Transit Gateway) maps to this
  local WSL2 Docker network setup
- How ArgoCD monitors cluster health and when `Successful` status appears
- How the `argocd-manager` ServiceAccount and credential Secret relate to each other

**What you'll do:**
- Rename the default minikube context to `us-east-ohio`
- Start a second minikube cluster `middle-east-uae`
- Wire Docker networks so ArgoCD can reach the second cluster's API
- Register `middle-east-uae` with ArgoCD and apply labels to both clusters
- Deploy the official ArgoCD guestbook (Demo-03) to both clusters and verify it is accessible
- Confirm cluster status transitions from `Unknown` to `Successful`

---

## Prerequisites

- ✅ Completed Demo-12 — App-of-Apps pattern understood
- ✅ ArgoCD running on minikube default profile (installed in Demo-02)
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with `Contents: Read` access to `gitops-apps-config`
  (Not needed for Part 1 — the guestbook verification uses the public
  `argoproj/argocd-example-apps` repo. PAT is required for Part 2.)

**Verify Prerequisites:**

### 1. ArgoCD pods running on default minikube profile
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

---

## Demo Objectives

By the end of this README, you will:

1. ✅ Understand why multi-cluster architectures exist — latency, compliance, failure isolation
2. ✅ Understand ArgoCD's control plane model — one ArgoCD, many clusters
3. ✅ Understand where `argocd-manager` SA is created and where the credential Secret lives
4. ✅ Understand why `STATUS=Unknown` is normal before the first Application sync
5. ✅ Understand how real-world network connectivity maps to this WSL2 Docker setup
6. ✅ Have two running minikube clusters with ArgoCD managing both
7. ✅ Have the official guestbook Application deployed to both clusters — fully verified and accessible
8. ✅ Have both clusters labelled and ready for ApplicationSet generators in Part 2

---

## Concepts

### The Demo Scenario

Imagine you are a platform engineer at a company whose web application serves
customers across two geographic regions — the United States and the Middle East.
To meet both **latency requirements** (users in each region are served by a
nearby cluster) and **data residency regulations** (customer data must stay
within the region it was collected), you run two Kubernetes clusters:

```
┌─────────────────────────────────────────────────────────┐
│                   Production Architecture               │
│                                                         │
│   US Customers                   ME Customers           │
│        │                               │                │
│        ▼                               ▼                │
│  ┌───────────────┐           ┌──────────────────┐       │
│  │  us-east-ohio │           │ middle-east-uae  │       │
│  │   (Ohio, US)  │           │  (UAE, ME)       │       │
│  │               │           │                  │       │
│  │  guestbook    │           │  guestbook       │       │
│  │    app        │           │     app          │       │
│  └───────────────┘           └──────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

The application code base is identical. What differs is the cluster and the namespace it deploys into. Part 2 of this demo demonstrates how ApplicationSets generate one Application per cluster automatically from a single YAML.

---

### Why Two Clusters — The Real-World Reasons

In production, running the same application across multiple clusters is driven
by four common requirements:

**1. Geographic latency** — Serve users from the nearest cluster. A customer
in Dubai should not have their requests routed to Ohio data centres.

**2. Data residency and compliance** — Many governments mandate that user data
must not leave the country. Separate clusters in separate regions satisfy these
regulatory requirements without special application logic.

**3. Failure domain isolation** — If the Ohio cluster has an incident, the UAE
cluster is completely unaffected. Users in the Middle East continue to be served.

**4. Environment separation** — Dev and staging workloads run on one cluster,
production on another. This prevents a broken staging deployment from affecting
production.

---

### How ArgoCD Controls Multiple Clusters — The Control Plane Model

ArgoCD is not installed on both clusters. It runs on **one cluster only** —
the **control plane cluster**. That single ArgoCD instance manages all clusters
from a central location.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     ArgoCD Control Plane Model                           │
│                                                                          │
│   GitHub (argoproj/argocd-example-apps)                                  │
│   ┌──────────────────┐                                                   │
│   │ branch: HEAD     │◄──── ArgoCD pulls desired state ────┐             │
│   │ path: guestbook/ │                                     │             │
│   └──────────────────┘                                     │             │
│                                                            │             │
│   ┌──────────────────────────────────────────────────────────────────┐   │
│   │              us-east-ohio cluster  (ArgoCD Control Plane)        │   │
│   │                                                                  │   │
│   │   ┌──────────────────────────────────────────────────────────┐   │   │
│   │   │  ArgoCD (argocd namespace)                               │   │   │
│   │   │  API Server → Repository Server → Application Controller │   │   │
│   │   │                      │                     │             │   │   │
│   │   │                 pulls Git            applies to          │   │   │
│   │   │                 manifests            both clusters       │   │   │
│   │   └──────────────────────────────────────────────────────────┘   │   │
│   │   guestbook pods (deployed to us-east-ohio by ArgoCD)            │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                            │                                             │
│                            │  argocd cluster add                         │
│                            │  (creates argocd-manager SA on target)      │
│                            ▼                                             │
│   ┌──────────────────────────────────────────────────────────────────┐   │
│   │              middle-east-uae cluster  (Target only)              │   │
│   │   argocd-manager ServiceAccount  (created during registration)   │   │
│   │   guestbook pods  (deployed here remotely by ArgoCD)             │   │
│   └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

**The connection in plain language:**

ArgoCD runs inside `us-east-ohio`. During `argocd cluster add middle-east-uae`,
ArgoCD reads your local kubeconfig to connect to `middle-east-uae` and creates
an `argocd-manager` ServiceAccount there with cluster-admin permissions. It then
stores the cluster URL, CA certificate, and ServiceAccount token as a Kubernetes
Secret in the `argocd` namespace on `us-east-ohio`.

From that point on, whenever ArgoCD syncs an Application whose destination is
`middle-east-uae`, the Application Controller uses those stored credentials to
authenticate to `middle-east-uae` and apply the manifests — without any further
input from you. Your kubeconfig is no longer involved.

```
One-time setup (you run once):
  argocd cluster add middle-east-uae
    → creates argocd-manager SA on middle-east-uae
    → stores credentials as Secret on us-east-ohio

Every subsequent sync (ArgoCD runs automatically):
  App Controller reads Secret → authenticates to middle-east-uae
    → applies manifests → reconciles drift
```

**What you need access to** in this demo:

| Who | Accesses | How |
|---|---|---|
| You (setup) | `us-east-ohio` | kubectl, kubeconfig |
| You (setup) | `middle-east-uae` | kubectl, kubeconfig (only for `argocd cluster add`) |
| You (day-to-day) | ArgoCD UI/CLI on `us-east-ohio` | port-forward 8080 |
| ArgoCD | `us-east-ohio` (local) | in-cluster ServiceAccount |
| ArgoCD | `middle-east-uae` (remote) | argocd-manager SA token stored as Secret |

After setup, the only cluster you interact with directly is `us-east-ohio` via
the ArgoCD UI and CLI. ArgoCD manages `middle-east-uae` on your behalf.

---

### Where Credentials Live — Source vs Target

This is the most commonly confused aspect of multi-cluster ArgoCD setup.
Understanding it prevents hours of troubleshooting:

| What is created | Where it lives | Purpose |
|---|---|---|
| `argocd-manager` ServiceAccount | **TARGET** cluster (`middle-east-uae`), `kube-system` | ArgoCD authenticates as this SA to deploy resources |
| `argocd-manager-role` ClusterRole | **TARGET** cluster (`middle-east-uae`) | Grants cluster-admin level permissions to the SA |
| `argocd-manager-role-binding` | **TARGET** cluster (`middle-east-uae`) | Binds the role to the SA |
| Cluster Secret (`argocd.argoproj.io/secret-type=cluster`) | **SOURCE** cluster (`us-east-ohio`), `argocd` namespace | Stores target cluster URL, CA cert, SA token — ArgoCD reads this |

The SA lives on the target so it has permissions there. The Secret lives on
the source so ArgoCD's Application Controller can read it locally and use it
to connect out to the target.

```
One-time setup (you run argocd cluster add once):
  → creates argocd-manager SA on TARGET (middle-east-uae)
  → stores credentials as Secret on SOURCE (us-east-ohio, argocd namespace)

Every subsequent sync (ArgoCD runs automatically):
  App Controller reads Secret on SOURCE
    → authenticates to TARGET
    → applies manifests
    → reconciles drift
```

---

### How the in-cluster Registration Works

When ArgoCD starts, it automatically registers its own cluster — the cluster
it is running in — as `in-cluster` with address `https://kubernetes.default.svc`.
No `argocd cluster add` command is required.

The `argocd-application-controller` pod runs with a Kubernetes ServiceAccount
that has cluster-admin level permissions, granted by the ClusterRoleBinding
installed during ArgoCD setup. The pod's mounted ServiceAccount token is used
to authenticate to the local Kubernetes API at startup — no credential Secret
is created for `in-cluster`.

```
argocd-application-controller pod
  → mounts /var/run/secrets/kubernetes.io/serviceaccount/token
  → connects to https://kubernetes.default.svc at startup
  → registers as in-cluster automatically
```

---

### Why Cluster STATUS=Unknown Is Normal Before the First Sync

After registration, `argocd cluster list` shows `Unknown` status for all
clusters — including `in-cluster`. This surprises many people, but it is
expected and correct behaviour:

```text
SERVER                          NAME              VERSION  STATUS   MESSAGE
https://kubernetes.default.svc  in-cluster                 Unknown  Cluster has no applications and is not being monitored.
https://192.168.67.2:8443        middle-east-uae            Unknown  Cluster has no applications and is not being monitored.
```

ArgoCD initialises its cluster resource watch (informer) **lazily** — only
when the first Application is deployed to that cluster. Before that, there
are no resources to watch, so no watch is started and the status is `Unknown`.

```
Before any Application deployed → STATUS: Unknown (normal)
After first Application syncs   → STATUS: Successful, VERSION: 1.xx
```

This behaviour applies to both `in-cluster` and remote clusters. The `Unknown`
message "Cluster has no applications and is not being monitored" is the exact
reason — not a connectivity failure.

---

### How Real-World Connectivity Maps to This Demo

**In real production environments**, both clusters run on cloud infrastructure
and are connected at the network level through one of:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Real-World Cluster Connectivity Options                  │
│                                                                             │
│  Option 1: VPC Peering                                                      │
│  ┌────────────────────┐         ┌──────────────────────────┐               │
│  │  us-east-1 VPC     │◄───────►│  me-south-1 VPC          │               │
│  │  (ArgoCD here)     │ peering │  (target cluster)        │               │
│  └────────────────────┘         └──────────────────────────┘               │
│                                                                             │
│  Option 2: Transit Gateway (hub-and-spoke, scales to many VPCs)            │
│  ┌──────────┐    ┌────────────────┐    ┌──────────┐  ┌──────────┐         │
│  │ us-east  │───►│Transit Gateway │◄───│ me-south │  │ eu-west  │         │
│  └──────────┘    └────────────────┘    └──────────┘  └──────────┘         │
│                                                                             │
│  Option 3: Site-to-Site VPN (encrypted tunnel over public internet)        │
│  ┌────────────────────┐  VPN tunnel  ┌──────────────────────────┐          │
│  │  us-east-1 VPC     │◄────────────►│  me-south-1 VPC          │          │
│  └────────────────────┘              └──────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
```

At runtime, ArgoCD's Application Controller opens HTTPS connections to the
target cluster's Kubernetes API server (port 6443 or 443), using the stored
bearer token and CA certificate.

**In this demo — WSL2 with Docker driver:**

Both clusters run as Docker containers on your local machine. Each minikube
profile creates its own Docker bridge network — isolated from each other by
default. The `docker network connect` command is the local equivalent of
VPC peering: it adds a second network interface to the `minikube` container,
giving it a route to the `middle-east-uae` Docker network.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  WSL2 Host                                                               │
│                                                                          │
│  ┌─────────────────────────────────────────┐                             │
│  │  Docker bridge: minikube (172.17.0.0/16)│                             │
│  │  ┌──────────────────────────────────┐   │                             │
│  │  │  minikube container              │   │ ← isolated by default       │
│  │  │  (us-east-ohio / ArgoCD inside)  │   │                             │
│  │  └──────────────────────────────────┘   │                             │
│  └─────────────────────────────────────────┘                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────┐                  │
│  │  Docker bridge: middle-east-uae (192.168.67.0/24)  │                  │
│  │  ┌───────────────────────────────────────────┐     │                  │
│  │  │  middle-east-uae container (192.168.67.2) │     │                  │
│  │  │  (API server on port 8443)                │     │                  │
│  │  └───────────────────────────────────────────┘     │                  │
│  └────────────────────────────────────────────────────┘                  │
│                                                                          │
│  Fix: docker network connect middle-east-uae minikube                    │
│  → adds eth1 to minikube container                                       │
│  → minikube container reaches 192.168.67.2:8443 directly                 │
│  → equivalent to VPC peering in production                               │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### How ArgoCD Monitors Cluster Health

ArgoCD continuously checks cluster connectivity through its reconciliation loop:

```
STATUS=Unknown    → no Applications on this cluster yet (watch not started)
STATUS=Successful → at least one Application deployed; watch active; API reachable
STATUS=Unknown (with "connection refused") → network failure after initial success
```

The `VERSION` field in `argocd cluster list` is populated by calling the
cluster's `/version` API endpoint — it appears only after the cluster watch
initialises (i.e., after the first Application sync).

---

## Folder Structure

```
13-applicationset/src/
└── (no local files needed for Part 1)
```

> Part 1 uses only the official `argoproj/argocd-example-apps` public repository
> for cluster verification — no local files, no Git push required. Part 2 adds
> the full generator manifest structure to `gitops-apps-config`.

---

## Multi-Cluster Setup — Rename Default Profile and Add Second Cluster

This demo simulates a two-cluster environment. Rather than creating fresh
clusters from scratch (which would require reinstalling ArgoCD), we reuse the
existing default minikube profile where ArgoCD is already running — renaming it
to reflect a real-world region name — and add a second lightweight profile
representing a second region.

## Step 1: Rename Default Context to `us-east-ohio`

minikube does not support renaming profiles directly. The workaround is to
update the kubeconfig context name, which is what ArgoCD and kubectl use.

```bash
# Rename the context in kubeconfig from 'minikube' to 'us-east-ohio'
kubectl config rename-context minikube us-east-ohio
```

**Verify:**
```bash
kubectl config get-contexts
```

**Expected:**
```text
CURRENT   NAME           CLUSTER    AUTHINFO   NAMESPACE
*         us-east-ohio   minikube   minikube
```

All ArgoCD pods and previous demo resources are still running — only the
context name changed. The cluster itself is unchanged.

---

## Step 2: Start the Second Cluster — `middle-east-uae`

```bash
#Create and start a new single node cluster
minikube start -p middle-east-uae --cpus 2 --memory 2048
```

This starts a second, independent minikube cluster in a new profile. ArgoCD
is NOT installed here — it only runs in the `us-east-ohio` cluster.

**Verify Node is ready**
```bash
kubectl get nodes
```

**Expected:**
```text
NAME              STATUS   ROLES           AGE     VERSION
middle-east-uae   Ready    control-plane   2d11h   v1.34.0
```

**Verify Context**
```bash
kubectl config get-contexts
```

**Expected:**
```text
CURRENT   NAME              CLUSTER           AUTHINFO          NAMESPACE
          us-east-ohio      minikube          minikube
*         middle-east-uae   middle-east-uae   middle-east-uae
```

> The asterisk is on `middle-east-uae` because it was created last. Switch
> back now — all ArgoCD commands require `us-east-ohio`:

```bash
kubectl config use-context us-east-ohio
```

**Verify ArgoCD is still healthy:**
```bash
kubectl get pods -n argocd
```

**Expected:** All pods `Running` and `1/1` Ready.

---

## Step 3: Wire the Docker Network (WSL2 Connectivity)

Both minikube clusters run as Docker containers on separate Docker bridge
networks. By default these networks are fully isolated — ArgoCD running inside
the `minikube` container cannot reach the `middle-east-uae` container's API
server.

This is the local equivalent of two VPCs with no peering. In production, VPC
peering, a Transit Gateway, or a site-to-site VPN would provide this connectivity.
Here, `docker network connect` is the fix.

**Get the `middle-east-uae` container IP:**
```bash
MINIKUBE_UAE_IP=$(docker inspect middle-east-uae \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  | tail -1)
echo "middle-east-uae IP: $MINIKUBE_UAE_IP"
```

**Expected:**
```text
middle-east-uae IP: 192.168.67.2
```

**Connect the two Docker networks:**
```bash
docker network connect middle-east-uae minikube
```
> **WSL2 side-effect — read this before continuing:** On some WSL2 setups,
> `docker network connect` triggers a brief restart of the minikube container's
> network stack. Docker reassigns the host port that was forwarding to the
> minikube API server (8443 inside container → random port on host). This
> invalidates the `server:` URL in kubeconfig and breaks both `kubectl` and
> `argocd` immediately.
>
> The next check detects this and the recovery is one command.

**Step 3a: Verify kubectl still works after network connect — do this immediately:**
```bash
kubectl config use-context us-east-ohio
kubectl get pods -n argocd
```

**Expected — all pods Running:**
```text
NAME                                            READY   STATUS    RESTARTS
my-argo-argocd-application-controller-0         1/1     Running   ...
my-argo-argocd-server-xxx                       1/1     Running   ...
...
```

**If instead you see `The connection to the server ... was refused`:**
```bash
# minikube container restarted and the kubeconfig port is stale
# This command reads the current Docker port mapping and updates kubeconfig
minikube update-context

# Verify kubeconfig is now correct
kubectl get pods -n argocd
```

**Then restart the ArgoCD port-forward** (the old process died with the container restart):
```bash
# In a new terminal or background:
kubectl port-forward svc/my-argo-argocd-server -n argocd 8080:443 &

# Verify ArgoCD CLI works
argocd cluster list
```

> Replace `my-argo-argocd-server` with your actual ArgoCD server service name
> (`kubectl get svc -n argocd` to find it).

**Verify Cluster Connectivity:**

**Step 3b:** Ping — confirms routing works:
```bash
docker exec minikube ping -c 2 $MINIKUBE_UAE_IP
```

**Expected:**
```text
64 bytes from 192.168.67.2: icmp_seq=1 ttl=64 time=0.5ms
64 bytes from 192.168.67.2: icmp_seq=2 ttl=64 time=0.4ms
```

**Step 3c:** TCP port check — confirms API server is reachable (use `nc`, not `cat < /dev/tcp` for TLS):
```bash
docker exec minikube nc -zv -w 3 $MINIKUBE_UAE_IP 8443
```

**Expected:**
```text
Connection to 192.168.67.2 8443 port [tcp/*] succeeded!
```

> **This setup persists only for the current Docker session.** If Docker or
> WSL2 restarts, run `docker network connect middle-east-uae minikube` again
> and follow the Step 3a recovery check. The ArgoCD credential Secret persists
> — only the network wiring and port-forward need re-establishing.

---

## Step 4: Register `middle-east-uae` with ArgoCD

ArgoCD is running in `us-east-ohio` and must be told about `middle-east-uae`
before it can deploy resources there.

```bash
argocd cluster add middle-east-uae \
  --label app=demo13 \
  --label region=middle-east-uae \
  --name middle-east-uae
```

**What this command does — step by step:**

1. Reads your local kubeconfig to find the `middle-east-uae` context
2. Connects to `middle-east-uae` and creates on the **TARGET** cluster:
   - `argocd-manager` ServiceAccount in `kube-system`
   - `argocd-manager-role` ClusterRole (cluster-admin permissions)
   - `argocd-manager-role-binding` ClusterRoleBinding
3. Retrieves the `argocd-manager` SA token from TARGET
4. Creates a Kubernetes Secret in the `argocd` namespace on **SOURCE**
   (`us-east-ohio`) containing: cluster URL, CA certificate, SA token, labels
5. From this point, ArgoCD uses the stored Secret — your kubeconfig is no
   longer needed for ongoing operations

**When prompted:** Type `y` to confirm.

**Expected CLI output:**
```text
WARNING: This will create a service account `argocd-manager` on the cluster
referenced by context `middle-east-uae` with full cluster level admin privileges.
Do you want to continue [y/N]? y
INFO[0001] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0001] ClusterRole "argocd-manager-role" created
INFO[0001] ClusterRoleBinding "argocd-manager-role-binding" created
INFO[0002] Created bearer token secret for ServiceAccount "argocd-manager"
Cluster 'https://192.168.67.2:8443' added
```

**Verify both clusters are visible to ArgoCD:**
```bash
argocd cluster list
```

**Expected:**
```text
SERVER                          NAME             VERSION  STATUS      MESSAGE                                                  PROJECT
https://kubernetes.default.svc  in-cluster                Unknown     Cluster has no applications and is not being monitored.  
https://192.168.67.2:8443       middle-east-uae  1.34     Successful
```

> Both may show status as `Unknown`. `Successful` status and VERSION appear after the first sync in
> Step 6. See the Concepts section "Why Cluster STATUS=Unknown Is Normal"
> for the full explanation.

**Verify the credential Secret was created on the SOURCE cluster:**
```bash
kubectl config use-context us-east-ohio
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
```

**Expected:**
```text
NAME                                        TYPE     DATA   AGE
cluster-192.168.67.2-2580991329             Opaque   3      89s
cluster-kubernetes.default.svc-3396314289   Opaque   3      2d11h
```

**Verify `argocd-manager` SA was created on the TARGET cluster:**
```bash
kubectl config use-context middle-east-uae
kubectl get serviceaccount argocd-manager -n kube-system
kubectl get clusterrole argocd-manager-role
```

**Expected:**
```text
NAME              SECRETS   AGE
argocd-manager    1         1m

NAME                   CREATED AT
argocd-manager-role    2024-xx-xx
```

```bash
# Switch back to control cluster
kubectl config use-context us-east-ohio
```

---

## Step 5: Label the `in-cluster` (us-east-ohio) Cluster

The `in-cluster` cluster was auto-registered when ArgoCD started. It has no
labels yet. The Cluster generator in Part 2 selects clusters by label — add
them now.

```bash
argocd cluster add us-east-ohio \
  --label app=demo13 \
  --label region=us-east-ohio \
  --in-cluster \
  --upsert
```

> **Why `--upsert`:** The `in-cluster` entry already exists. Without `--upsert`
> the command fails with "cluster already exists". The flag overwrites the
> existing registration with the new labels.
>
> **Why `--in-cluster`:** Tells ArgoCD this is the same cluster it runs in —
> skips creating a redundant `argocd-manager` SA.

Alternatively, add labels via the ArgoCD UI:
```
http://localhost:8080 → Settings → Clusters → in-cluster → Edit
→ Add labels: app=demo13, region=us-east-ohio → Save
```

**Verify both cluster secrets have labels:**
```bash
kubectl get secret -n argocd \
  -l argocd.argoproj.io/secret-type=cluster \
  -o yaml | grep -A 8 "labels:"
```

**Expected:** Both secrets show `app: demo13` and their respective `region` label.

---

## Step 6: Deploy Guestbook to Both Clusters and Verify

This step confirms that ArgoCD can successfully deploy to both clusters before
Part 2 uses ApplicationSets to automate this. We reuse the same official
ArgoCD guestbook application used in Demo-03 — no Git push, no custom manifests.
We create two temporary Application CRDs directly via `kubectl apply`, each
pointing at the public `argoproj/argocd-example-apps` repository. Once verified,
we clean them up so Part 2 starts from a clean state.

> **Why guestbook?** The ArgoCD guestbook (`https://github.com/argoproj/argocd-example-apps.git`,
> path `guestbook`) is the official ArgoCD demo application — a public repo
> that requires no authentication, no Git setup, and no image pull secrets.
> It is exactly what was used in Demo-03. It deploys a `guestbook-ui` Deployment
> and Service and is accessible via port-forward. Using it here keeps the focus
> on verifying cluster connectivity, not on setting up test fixtures.

### Step 6a: Deploy guestbook to `in-cluster` (us-east-ohio)

```bash
kubectl config use-context us-east-ohio

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-us-east-ohio
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    name: in-cluster
    namespace: guestbook-ohio
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```
> **What this Application CRD does:**
> | Field | Value | Purpose |
> |---|---|---|
> | `source.repoURL` | `https://github.com/argoproj/argocd-example-apps.git` | Public Git repo ArgoCD pulls manifests from — no auth needed |
> | `source.path` | `guestbook` | Folder inside the repo containing the Deployment and Service |
> | `source.targetRevision` | `HEAD` | Always uses the latest commit on the default branch |
> | `destination.name` | `in-cluster` | Tells ArgoCD to deploy to the **local** cluster (us-east-ohio) where ArgoCD itself runs |
> | `destination.namespace` | `guestbook-ohio` | Kubernetes namespace created and deployed into on that cluster |

**Verify the Application synced:**
```bash
argocd app get guestbook-us-east-ohio --refresh
```

**Expected:**
```text
Name:               argocd/guestbook-us-east-ohio
Project:            default
Sync Status:        Synced
Health Status:      Healthy
```

**Verify:**
```bash
kubectl get pods -n guestbook-ohio
```

**Expected:**
```text
NAME                            READY   STATUS    RESTARTS
guestbook-ui-xxxxxxxxx-xxxxx    1/1     Running   0
```

**Access the guestbook UI:**
```bash
kubectl port-forward svc/guestbook-ui -n guestbook-ohio 8081:80
```

Open `http://localhost:8081` — the guestbook web UI appears (the same page
familiar from Demo-03).

**Observation:** ArgoCD pulled the manifest from the public GitHub repo,
deployed to `in-cluster`, and the pod is running — the full ArgoCD sync chain
works for the local cluster.

### Step 6b: Deploy guestbook to `middle-east-uae` (remote cluster)

All ArgoCD commands target the control cluster (`us-east-ohio`), even when
the destination is the remote cluster. Do not switch context before this step.

```bash
kubectl config use-context us-east-ohio   # stay on the control cluster

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-middle-east-uae
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    name: middle-east-uae
    namespace: guestbook-uae
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```
> **What this Application CRD does:**
> | Field | Value | Purpose |
> |---|---|---|
> | `source.repoURL` | `https://github.com/argoproj/argocd-example-apps.git` | Same public Git repo — identical source as the Ohio app |
> | `source.path` | `guestbook` | Same folder — same manifests deployed to both clusters |
> | `source.targetRevision` | `HEAD` | Latest commit on default branch |
> | `destination.name` | `middle-east-uae` | Tells ArgoCD to deploy to the **remote** cluster registered in Step 4 — this is the only field that differs from the Ohio Application |
> | `destination.namespace` | `guestbook-uae` | Kubernetes namespace created and deployed into on the remote cluster |
>
> The two Applications are **identical except for `destination.name` and `destination.namespace`**. This is the manual version of exactly what ApplicationSets automate in Part 2 — one template, `destination` driven by a generator variable.

**Verify the Application synced:**
```bash
argocd app get guestbook-middle-east-uae --refresh
```

**Expected:**
```text
Name:               argocd/guestbook-middle-east-uae
Project:            default
Sync Status:        Synced
Health Status:      Healthy
```

**Verify:**

Switch to the UAE context to check that pods are running on the **remote** cluster:
```bash
kubectl config use-context middle-east-uae
kubectl get pods -n guestbook-uae
```

**Expected:**
```text
NAME                            READY   STATUS    RESTARTS
guestbook-ui-xxxxxxxxx-xxxxx    1/1     Running   0
```

**Access the guestbook UI from the UAE cluster:**
```bash
kubectl port-forward svc/guestbook-ui -n guestbook-uae 8082:80
```

Open `http://localhost:8082` — same guestbook UI, now served from the
`middle-east-uae` cluster. This confirms ArgoCD on `us-east-ohio` successfully
deployed to the remote cluster using the stored `argocd-manager` credentials.

```bash
# Switch back to the control cluster for all subsequent steps
kubectl config use-context us-east-ohio
```

**Observation:** The same Application CRD — identical source, different
`destination.name` — deployed the same guestbook to a completely separate
cluster. This is exactly what ApplicationSets automate in Part 2: one template,
multiple destinations.

---

## Step 7: Verify Cluster Status is Now Successful

With Applications deployed to both clusters, the cluster watch is now active
and health monitoring is running.

```bash
argocd cluster list
```

**Expected:**
```text
SERVER                          NAME             VERSION  STATUS      MESSAGE  PROJECT
https://192.168.67.2:8443       middle-east-uae  1.34     Successful           
https://kubernetes.default.svc  us-east-ohio     1.34     Successful  
```

Both clusters now show `Successful` and a Kubernetes VERSION. This confirms:
- Network connectivity is working (ArgoCD can reach both API servers)
- Credentials are valid (ArgoCD can authenticate to both clusters)
- ArgoCD can apply manifests to both clusters

**Verify the full Application list:**
```bash
argocd app list
```

**Expected:**
```text
NAME                          CLUSTER           SYNC STATUS   HEALTH STATUS
guestbook-us-east-ohio        in-cluster        Synced        Healthy
guestbook-middle-east-uae     middle-east-uae   Synced        Healthy
```

---

## Step 8: Clean Up Guestbook Applications

The verification is complete. Remove the guestbook Applications so Part 2
starts from a clean state. The cluster registrations, labels, and network
wiring all persist.

```bash
kubectl config use-context us-east-ohio

# Delete both guestbook Applications (cascade deletes pods and namespace resources)
argocd app delete guestbook-us-east-ohio --cascade
argocd app delete guestbook-middle-east-uae --cascade
```

**Verify cleanup:**
```bash
argocd app list
```

**Expected:** No applications listed.

```bash
# Confirm namespaces are gone
kubectl get ns guestbook-ohio 2>&1 || echo "Namespace gone — cleanup successful"
kubectl delete ns guestbook-ohio
kubectl config use-context middle-east-uae
kubectl get ns guestbook-uae 2>&1 || echo "Namespace gone — cleanup successful"
kubectl delete ns guestbook-uae 
kubectl config use-context us-east-ohio
```

---

## Validation Checklist

Before proceeding to Demo-13 Part 2, verify:

- [ ] `kubectl config get-contexts` shows both `us-east-ohio` and `middle-east-uae`
- [ ] `kubectl get pods -n argocd` — all ArgoCD pods `Running` and `1/1` Ready
- [ ] `docker exec minikube nc -zv -w 3 192.168.67.2 8443` — TCP `succeeded!`
- [ ] `argocd cluster list` shows both clusters with `STATUS: Successful`
- [ ] Both cluster secrets exist: `kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster`
- [ ] `argocd-manager` SA exists on UAE cluster: `kubectl config use-context middle-east-uae && kubectl get sa argocd-manager -n kube-system`
- [ ] Both clusters have `app=demo13` and `region=<name>` labels in ArgoCD
- [ ] `argocd app list` is empty (guestbook apps cleaned up)

---

## Cleanup (End of Demo Only)

> **Do NOT run this now.** Run it only after completing Demo-13 Part 2.
> The two clusters and all registrations are required for Part 2.

```bash
# Delete all Demo-13 ApplicationSets (run after Part 2 is complete)
# kubectl delete appset --all -n argocd

# Delete the second minikube profile
minikube delete -p middle-east-uae

# Rename the context back to 'minikube'
kubectl config rename-context us-east-ohio minikube
```

---

## What You Learned

In this setup guide, you:

- ✅ Understood the four reasons multi-cluster architectures exist in production
- ✅ Understood ArgoCD's control plane model — one ArgoCD, many clusters
- ✅ Understood that `argocd-manager` SA is created on the TARGET cluster and the credential Secret is stored on the SOURCE cluster
- ✅ Understood how `in-cluster` is auto-registered at ArgoCD startup using the pod's mounted ServiceAccount token
- ✅ Understood why `STATUS=Unknown` is normal before the first Application sync — lazy watch initialisation
- ✅ Understood how `docker network connect` maps to VPC peering in production
- ✅ Configured WSL2 Docker network wiring for two minikube clusters
- ✅ Registered a remote cluster with ArgoCD and applied labels for the Cluster generator
- ✅ Deployed a test Application to both clusters and verified end-to-end connectivity
- ✅ Confirmed both clusters transition to `STATUS: Successful` after first sync

**Key Insight:**
ArgoCD's cluster registration separates a one-time setup action (creating the
`argocd-manager` SA on the target) from ongoing runtime (reading the credential
Secret on the source). After registration, your kubeconfig is no longer needed.
The entire multi-cluster control plane runs autonomously — ArgoCD manages the
credentials, the connections, and the reconciliation loop.

---

## Lessons Learned

**1. argocd-manager SA on TARGET, credential Secret on SOURCE**
The ServiceAccount and RBAC live on the cluster you are deploying to. The Secret
storing those credentials lives in the `argocd` namespace on the control plane
cluster where ArgoCD runs. Confusion between these two locations is the most
common multi-cluster troubleshooting mistake.

**2. Unknown cluster status before first sync is expected — not an error**
ArgoCD initialises its cluster watch lazily. `STATUS=Unknown` with "Cluster has
no applications and is not being monitored" is the correct pre-sync state. Do
not attempt to troubleshoot network connectivity based on this status alone —
deploy an Application first and check again.

**3. `docker network connect` can break the minikube kubeconfig port — run the check immediately**
On WSL2, `docker network connect` sometimes triggers a network stack restart on the
minikube container. Docker then reassigns the host port that was forwarding to the
minikube API server — the old port in kubeconfig becomes stale and `kubectl` fails
with "connection refused". Fix: run `minikube update-context` immediately after
`docker network connect` to refresh the kubeconfig, then restart the ArgoCD
port-forward. Always run `kubectl get pods -n argocd` right after network connect
to detect this before attempting `argocd cluster add`.

**4. `docker network connect` must be re-run after every Docker/WSL2 restart**
The network wiring is session-scoped. The ArgoCD credential Secret persists across
restarts — only the Docker network connection and the ArgoCD port-forward need
re-establishing. Run `docker network connect middle-east-uae minikube` followed by
the Step 3a kubectl check every time Docker or WSL2 restarts.

**5. Use nc -zv for TLS port testing — not cat < /dev/tcp**
`cat < /dev/tcp` does not handle TLS handshakes and gives misleading results
for Kubernetes API servers. `nc -zv -w 3 <ip> <port>` correctly tests TCP
connectivity without attempting a TLS negotiation.

**6. --upsert is required when updating already-registered cluster labels**
The `in-cluster` cluster is auto-registered at startup. Re-running
`argocd cluster add` without `--upsert` fails with "cluster already exists".
Always use `--upsert` when modifying labels on an existing registration.

**7. Context switching is the most common error source in multi-cluster work**
Always verify active context with `kubectl config current-context` before any
`kubectl` command. ArgoCD commands always target the control plane cluster
(`us-east-ohio`). Pod verification for `middle-east-uae` requires explicitly
switching context — `kubectl` does not follow ArgoCD's cluster destination.

---

## Next Steps

**Demo-13 Part 2 — ApplicationSets: Scalable GitOps Across Clusters and Environments**

With both clusters running, registered, labelled, and verified, Part 2 focuses
entirely on ApplicationSet generators:

- **List generator** — deploy the same app to both clusters via inline list
- **Cluster generator** — deploy based on cluster labels (the labels set in this README)
- **Git Directory generator** — deploy dev/staging/prod discovered from Git folders
- **Git File generator** — deploy based on a `clusters.yaml` file in Git
- **Matrix generator** — deploy all three environments to both clusters — six Applications from one YAML

> **Before starting Part 2:** Confirm the docker network wiring is still active
> (Docker/WSL2 may have restarted):
> ```bash
> docker exec minikube nc -zv -w 3 192.168.67.2 8443
> # Expected: Connection ... succeeded!
> # If it fails: docker network connect middle-east-uae minikube
> ```