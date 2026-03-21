# Demo-04: Private Repository Authentication — HTTPS & SSH

## Overview

Demo-03 deployed the guestbook from a **public** repository — no credentials
needed, ArgoCD could clone it freely. That is the exception in real organisations.
Almost every production repository is private.

In this demo we take the **same guestbook application** from Demo-03 and move it
into a private GitHub repository. Nothing about the manifests changes — only the
authentication layer does. We will see exactly what fails without credentials, why
it fails, and then fix it two ways: HTTPS using a Personal Access Token, and SSH
using a deploy key.

Understanding how ArgoCD stores credentials is essential before building anything
that uses private repos — which means every demo from Demo-05 onwards depends on
this foundation.

**What you'll learn:**
- The difference between ArgoCD repository credentials and Docker registry secrets
  — two completely different secret types that are often confused
- How ArgoCD discovers credentials — Kubernetes secrets with a specific label in
  the `argocd` namespace
- The exact secret schema for HTTPS and SSH authentication
- Why the `argocd.argoproj.io/secret-type: repository` label is mandatory — and
  what happens without it
- Why the URL in the credential secret must match the URL in the Application CRD
- Three ways to configure credentials: ArgoCD UI, `kubectl`, and `argocd` CLI
- Why credentials should never be stored in declarative YAML files committed to Git
- What credential templates are and when to use them in production

**What you'll do:**
- Create a private GitHub repository and push the guestbook manifests into it
- Point an Application CRD at the private repo and observe the auth error
- Configure HTTPS authentication via ArgoCD UI, then rebuild it via `kubectl`
- Configure SSH authentication via ArgoCD UI, then rebuild it via `kubectl`
- Clean up all credentials safely

## Prerequisites

- ✅ Completed Demo-03 — guestbook deployed from public repo, ArgoCD working
- ✅ ArgoCD running on minikube (`kubectl get pods -n argocd`)
- ✅ ArgoCD CLI installed and logged in
  (`argocd login localhost:8080 --username admin --insecure`)
- ✅ GitHub account with ability to create private repositories
- ✅ `kubectl` available in terminal
- ✅ `ssh-keygen` available in terminal

**Verify prerequisites:**
```bash
kubectl get pods -n argocd
argocd app list          # should show guestbook from Demo-03
argocd repo list         # should show no repos registered (Demo-03 used a public repo)
```

---

## Background: Two Different Secret Types — Do Not Confuse Them

This is the most important concept to get right before this demo. ArgoCD GitOps
involves two completely different kinds of secrets, created differently, with
different schemas, for different purposes:

```
Secret type              Label / Type                      Purpose
─────────────────────────────────────────────────────────────────────────────
ArgoCD repository        argocd.argoproj.io/               Lets ArgoCD CLONE
credential               secret-type: repository           your Git repo to
(this demo)              (in argocd namespace)             read manifests

Docker registry          type:                             Lets Kubernetes PULL
secret                   kubernetes.io/                    your container image
(Demo-05)                dockerconfigjson                  from a private registry
                         (in app namespace)
```

**They solve different problems at different layers:**

```
Git repo (private)                     Container registry (private)
      │                                          │
      │ ArgoCD needs to READ manifests           │ Kubernetes needs to PULL image
      │                                          │
      ▼                                          ▼
argocd.argoproj.io/                    kubernetes.io/
secret-type: repository                dockerconfigjson
(ArgoCD reads this)                    (kubelet reads this via imagePullSecrets)
```

Demo-05 covers the Docker registry secret. This demo covers the ArgoCD repository
credential. They are created with different `kubectl` commands, stored in different
namespaces, and consumed by entirely different components.

---

## Background: How ArgoCD Discovers Repository Credentials

ArgoCD does not have its own credential database. It discovers credentials by
looking for Kubernetes secrets in the `argocd` namespace with a specific label.

**The mandatory label:**
```yaml
labels:
  argocd.argoproj.io/secret-type: repository
```

Without this label, ArgoCD ignores the secret completely — even if it is in the
`argocd` namespace with perfectly correct credentials. The label is ArgoCD's
signal that this secret contains repository authentication data.

**How ArgoCD matches credentials to repos:**

ArgoCD matches the `url` field inside the secret to the `repoURL` field in the
Application CRD. The match must be exact — same URL format, same case.

```
Application CRD                    Credential secret
─────────────────────────────────────────────────────────
repoURL: https://github.com/...    url: https://github.com/...   ← MATCH ✅
repoURL: git@github.com:...        url: git@github.com:...       ← MATCH ✅
repoURL: https://github.com/...    url: git@github.com:...       ← NO MATCH ❌
```

This means: if you use an HTTPS credential secret, the Application CRD must use
an HTTPS `repoURL`. If you switch to SSH, the `repoURL` must change to SSH format.

---

## Background: Three Secret Label Types

ArgoCD uses three different label values, each serving a different purpose:

| Label value | Purpose | URL field |
|---|---|---|
| `repository` | Credentials for one specific repo | Full repo URL |
| `repo-creds` | Credential template covering all repos under a URL prefix | URL prefix only |
| `cluster` | Credentials for a remote Kubernetes cluster | Cluster API URL |

This demo uses `repository` (single repo). See the credential templates section
below for when `repo-creds` is the better choice.

---

## Background: Credential Templates (`repo-creds`) — The Production Pattern

In this demo, one secret covers one repository. In production, you often have
many repositories under the same GitHub organisation. Creating one secret per
repo is repetitive and hard to maintain.

For credential reuse across multiple repositories — for example, all repositories
under a GitHub organisation — use repository credential templates with
`argocd.argoproj.io/secret-type: repo-creds`. This provides better UX than
creating individual repository secrets.

**How credential templates work:**

The `url` field contains a **prefix**, not a full repo URL. Any repository whose
URL starts with that prefix automatically inherits the credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-org-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds   # ← template, not repository
stringData:
  type: git
  url: https://github.com/rselvantech            # ← prefix, not full URL
  username: rselvantech
  password: <PAT>
```

With this template in place, any Application CRD pointing to
`https://github.com/rselvantech/any-repo` automatically uses these credentials —
no per-repo secret needed. In this demo we use per-repo secrets so you understand
the mechanics before the abstraction.

---

## Background: Three Ways to Configure Credentials

| Method | How | When to use |
|---|---|---|
| ArgoCD UI | Settings → Repositories → Connect Repo | Learning, one-off setup |
| `kubectl create secret` + `kubectl label` | Two imperative commands | Any non-UI setup |
| `argocd repo add` | CLI command | Quick scripted setup |

All three methods create the same Kubernetes secret with the same label. The UI
and `argocd repo add` just wrap the `kubectl` operations. In this demo we practice
all three so you understand what is actually happening at the secret level.

**Why not use a declarative YAML file for credentials?**

Never commit secrets to a Git repository. If you write a YAML file containing
a PAT or SSH key and push it, the credential is exposed. The `kubectl create secret`
imperative command keeps the sensitive value in your terminal only — it never
touches a file. This is the correct approach for secrets.

---

## Folder Structure

```
04-private-repo-auth/
├── README.md
└── src/
    └── guestbook-private-app.yaml    ← Application CRD (no credentials here)
```

The credential secrets are **never stored in files** — they are created imperatively
and live only in the `argocd` namespace in your cluster.

**The private repository** (`argocd-guestbook-config-private`) is a separate GitHub
repo you create in Step 1. It is not part of `gitops-labs` — it is the private
source repo that ArgoCD reads from.

```
argocd-guestbook-config-private (GitHub — private)
└── guestbook/
    ├── guestbook-ui-deployment.yaml
    └── guestbook-ui-svc.yaml
```

---

## Step 1: Create the Private Repository

### 1a: Create the repo on GitHub

Go to GitHub → **New repository**:
- Name: `argocd-guestbook-config-private`
- Visibility: **Private** ← this is the entire point of this demo
- Add a README: yes (so the repo is not empty)
- Click **Create repository**

### 1b: Push the guestbook manifests

Clone your new repo locally:
```bash
git clone https://github.com/rselvantech/argocd-guestbook-config-private.git
cd argocd-guestbook-config-private
```

Create the guestbook manifests — same content as Demo-03's public source:

**`guestbook/guestbook-ui-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
      - name: guestbook-ui
        image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        ports:
        - containerPort: 80
```

**`guestbook/guestbook-ui-svc.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
spec:
  selector:
    app: guestbook-ui
  ports:
  - port: 80
    targetPort: 80
```

Push to GitHub:
```bash
mkdir guestbook
# create the two files above inside guestbook/
git add guestbook/
git commit -m "feat: add guestbook manifests for private repo auth demo"
git push origin main
```

Verify on GitHub — the `guestbook/` folder should be visible in the repo.

---

## Step 2: Create the Application CRD and Observe the Auth Error

Create `04-private-repo-auth/src/guestbook-private-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-private
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/argocd-guestbook-config-private.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook-private
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Create the namespace and apply:
```bash
kubectl apply -f src/guestbook-private-app.yaml
```

Expected:
```
application.argoproj.io/guestbook-private created
```

Check status immediately:
```bash
argocd app get guestbook-private
```

Expected — authentication error:
```
CONDITION   MESSAGE
CompError   rpc error: Failed to load target state:
            failed to load app list: authentication required
```

In the ArgoCD UI, clicking on `guestbook-private` shows the error:
```
Failed to load target state
authentication required
```

This is the correct behaviour. ArgoCD tried to clone the private repo, received
a 401/403 from GitHub, and reported the error. No manifests were read. No
resources were created.

> **This is exactly what Demo-03 avoided** by using a public repo. Switching to
> private without credentials produces this error on every sync attempt until
> credentials are configured.

---

## Step 3: Generate a Fine-Grained Personal Access Token

Before connecting the repo, generate a PAT with minimum required permissions.

**In GitHub:**
```
Settings (top right avatar)
  → Developer Settings (bottom left)
  → Personal Access Tokens
  → Fine-grained tokens
  → Generate new token
```

Fill in:
- **Token name:** `argocd-guestbook-private-https`
- **Resource owner:** your account
- **Repository access:** Only selected repositories →
  select `argocd-guestbook-config-private`
- **Permissions → Repository permissions → Contents:** Read-only

Click **Generate token**. Copy it immediately — GitHub shows it only once.

> **Minimum permissions:** ArgoCD only clones repositories — it never pushes.
> `Contents: Read-only` is all it needs. Never grant write access to ArgoCD
> credentials.

---

## Step 4: Configure HTTPS Authentication — ArgoCD UI

### 4a: Connect via UI

In ArgoCD UI:
```
Settings → Repositories → Connect Repo
  Connection Method: HTTPS
  Type: git
  Project: default
  Repository URL: https://github.com/rselvantech/argocd-guestbook-config-private.git
  Username: rselvantech
  Password: <paste your PAT>
→ Connect
```

Expected: **Connection Status: Successful**

### 4b: Verify the application syncs

```bash
argocd app get guestbook-private
```

Expected — error is gone, app is syncing:
```
Sync Status:   OutOfSync (or Synced if auto-detected)
Health Status: Missing → Healthy after sync
```

Trigger sync:
```bash
argocd app sync guestbook-private
```

Verify pods running:
```bash
kubectl get pods -n guestbook-private
```

Expected:
```
NAME                            READY   STATUS    RESTARTS
guestbook-ui-xxxxxxxxx-xxxxx    1/1     Running   0
```

### 4c: Inspect what the UI created

The ArgoCD UI created a Kubernetes secret behind the scenes:

```bash
kubectl get secrets -n argocd | grep repo
```

Expected — a secret was created:
```
repo-xxxxxxxxxxxxxxxxxx    Opaque    4    1m
```

Describe it to see the schema:
```bash
kubectl describe secret repo-xxxxxxxxxxxxxxxxxx -n argocd
```

Expected — note the label and data keys:
```
Labels:  argocd.argoproj.io/secret-type=repository
Data:
  password:  40 bytes     ← your PAT
  type:      3 bytes      ← "git"
  url:       71 bytes     ← the repo URL
  username:  12 bytes     ← your GitHub username
```

This is the Kubernetes secret the UI created. Everything in this demo is about
understanding and recreating this object.

---

## Step 5: Recreate HTTPS Credentials via `kubectl`

### 5a: Delete the UI-created secret

```bash
# Disconnect via UI: Settings → Repositories → Disconnect → OK
# Or via kubectl — find and delete the secret
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
kubectl delete secret <secret-name> -n argocd
```

Verify the error returns:
```bash
argocd app get guestbook-private
# Should show: authentication required
```

### 5b: Create the secret imperatively

```bash
kubectl create secret generic guestbook-private-https \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/rselvantech/argocd-guestbook-config-private.git \
  --from-literal=username=rselvantech \
  --from-literal=password=<your-PAT>
```

Expected:
```
secret/guestbook-private-https created
```

Check ArgoCD status — still shows auth error:
```bash
argocd app get guestbook-private
# Still: authentication required
```

**The secret exists but ArgoCD ignores it** — because the mandatory label is
missing. This is the critical learning moment.

### 5c: Add the mandatory label

```bash
kubectl label secret guestbook-private-https \
  -n argocd \
  argocd.argoproj.io/secret-type=repository
```

Expected:
```
secret/guestbook-private-https labeled
```

Now check ArgoCD — within seconds the error disappears:
```bash
argocd app get guestbook-private
# Should show: Synced / Healthy
```

```bash
# Verify label is present
kubectl describe secret guestbook-private-https -n argocd | grep -A2 Labels
```

Expected:
```
Labels: argocd.argoproj.io/secret-type=repository
```

> **Why two steps?** `kubectl create secret` does not support arbitrary label
> flags during creation (only standard ones). The label must be added separately
> with `kubectl label`. This is normal — it is not a workaround.

### 5d: Verify via `argocd` CLI (third method)

The `argocd repo add` command is a single-step alternative that handles both
secret creation and labelling internally:

```bash
# This is equivalent to create secret + label secret in one command
argocd repo add https://github.com/rselvantech/argocd-guestbook-config-private.git \
  --username rselvantech \
  --password <your-PAT>
```

```bash
# Verify repo is registered
argocd repo list
```

Expected:
```
TYPE  NAME  REPO                                                           INSECURE  OCI  LFS  CREDS  STATUS     MESSAGE
git         https://github.com/rselvantech/argocd-guestbook-config-private.git  false  false  false  false  Successful
```

---

## Step 6: Switch to SSH Authentication

SSH is the recommended method for production. Deploy keys have no user account
dependency — if the person who created the PAT leaves the organisation, the
HTTPS authentication breaks. SSH deploy keys are repository-level, not user-level.

### 6a: Generate SSH key pair

```bash
mkdir -p .ssh
ssh-keygen -t ed25519 \
  -f .ssh/argocd-deploy-key \
  -N ""                        # empty passphrase — required for automated use
```

Expected output:
```
Your identification has been saved in .ssh/argocd-deploy-key
Your public key has been saved in .ssh/argocd-deploy-key.pub
```

Two files created:
- `.ssh/argocd-deploy-key` — **private key** → goes into ArgoCD
- `.ssh/argocd-deploy-key.pub` — **public key** → goes into GitHub

### 6b: Add public key as a deploy key in GitHub

Print the public key:
```bash
cat .ssh/argocd-deploy-key.pub
```

Copy the full output. In GitHub:
```
argocd-guestbook-config-private repo
  → Settings → Deploy keys → Add deploy key
    Title: argocd-deploy-key
    Key: paste the public key contents
    Allow write access: ❌ unchecked (read-only)
  → Add key
```

### 6c: Update the Application CRD — HTTPS URL → SSH URL

The Application CRD must use the SSH URL format. Edit
`04-private-repo-auth/src/guestbook-private-app.yaml`:

```yaml
spec:
  source:
    repoURL: git@github.com:rselvantech/argocd-guestbook-config-private.git
    #         ↑ SSH format — different from HTTPS format
```

Apply the updated CRD:
```bash
kubectl apply -f src/guestbook-private-app.yaml
```

At this point the application shows an error — the URL changed to SSH but there
are no SSH credentials yet. This proves the URL-matching requirement.

### 6d: Delete the HTTPS credential

```bash
kubectl delete secret guestbook-private-https -n argocd
# Or disconnect from UI: Settings → Repositories → Disconnect
```

### 6e: Configure SSH via ArgoCD UI

In ArgoCD UI:
```
Settings → Repositories → Connect Repo
  Connection Method: SSH
  Repository URL: git@github.com:rselvantech/argocd-guestbook-config-private.git
  SSH Private Key Data: paste the contents of .ssh/argocd-deploy-key
→ Connect
```

Print private key to paste:
```bash
cat .ssh/argocd-deploy-key
```

Expected: **Connection Status: Successful**

Verify application recovers:
```bash
argocd app get guestbook-private
# Should show: Synced / Healthy
```

Inspect the SSH secret created by the UI:
```bash
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
kubectl describe secret <ssh-secret-name> -n argocd
```

Expected — different schema from HTTPS:
```
Labels:  argocd.argoproj.io/secret-type=repository
Data:
  sshPrivateKey:  411 bytes    ← private key (no username or password)
  type:           3 bytes      ← "git"
  url:            59 bytes     ← SSH format URL
```

---

## Step 7: Recreate SSH Credentials via `kubectl`

### 7a: Delete the UI-created SSH secret

```bash
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
kubectl delete secret <secret-name> -n argocd
```

### 7b: Store private key in a variable

```bash
private_key=$(cat .ssh/argocd-deploy-key)
```

### 7c: Create the SSH secret

```bash
kubectl create secret generic guestbook-private-ssh \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:rselvantech/argocd-guestbook-config-private.git \
  --from-literal=sshPrivateKey="$private_key"
```

### 7d: Add the mandatory label

```bash
kubectl label secret guestbook-private-ssh \
  -n argocd \
  argocd.argoproj.io/secret-type=repository
```

Verify application is synced:
```bash
argocd app get guestbook-private
kubectl get pods -n guestbook-private
```

Expected: app `Synced` and `Healthy`, pod `Running`.

---

## Step 8: Cleanup — Remove All Sensitive Data

Always clean up credentials after a demo. Leaving PATs and SSH keys active is
a security risk.

```bash
# 1. Delete ArgoCD credential secret from cluster
kubectl delete secret guestbook-private-ssh -n argocd

# 2. Delete the Application
kubectl delete app guestbook-private -n argocd

# 3. Delete the namespace
kubectl delete namespace guestbook-private

# 4. Delete local SSH key files
rm -rf .ssh/argocd-deploy-key .ssh/argocd-deploy-key.pub
```

In GitHub:
```
# 5. Delete the deploy key
argocd-guestbook-config-private repo → Settings → Deploy keys → Delete

# 6. Delete the PAT
Settings → Developer Settings → Fine-grained tokens → Delete token
```

Verify clean state:
```bash
argocd app list               # guestbook-private gone
argocd repo list              # no repos registered
kubectl get ns guestbook-private  # not found
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
# No resources found
```

> The `argocd-guestbook-config-private` GitHub repo can be kept — it is not
> sensitive. The credentials (PAT and SSH keys) are what must be deleted.

---

## Verify Final State

```bash
# ArgoCD running and healthy
kubectl get pods -n argocd

# Demo-03 guestbook still running (this demo did not touch it)
argocd app get guestbook
kubectl get pods -n guestbook

# No repository credentials registered
argocd repo list

# No leftover secrets from this demo
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

---

## Key Concepts Summary

**Two secret types — never confuse them**

ArgoCD repository credentials (`secret-type: repository`) let ArgoCD clone Git
repos to read manifests. Docker registry secrets (`type: kubernetes.io/dockerconfigjson`)
let Kubernetes pull container images. They are created differently, stored in
different namespaces, and consumed by different components. Demo-05 covers the
Docker registry secret.

**The mandatory label is non-negotiable**

A secret without `argocd.argoproj.io/secret-type: repository` is invisible to
ArgoCD. The secret can be correctly formed, in the right namespace, with valid
credentials — ArgoCD still ignores it. Always add the label.

**URL matching is exact**

ArgoCD matches credentials to repos by comparing the `url` field in the secret
to the `repoURL` in the Application CRD. HTTPS URL in the secret requires HTTPS
URL in the Application. SSH URL in the secret requires SSH URL in the Application.
A mismatch causes authentication failure even with valid credentials.

**HTTPS vs SSH — secret schema comparison**

```
HTTPS secret                        SSH secret
───────────────────────────────     ───────────────────────────────
type: git                           type: git
url: https://github.com/...         url: git@github.com:...
username: your-username             sshPrivateKey: |
password: <PAT>                       -----BEGIN OPENSSH PRIVATE KEY-----
                                      ...
                                      -----END OPENSSH PRIVATE KEY-----
```

**Deploy keys vs PATs — production choice**

| | Fine-grained PAT | SSH Deploy Key |
|---|---|---|
| Tied to a user account | ✅ yes — breaks if user leaves | ❌ no — repo-level |
| Scope | Can cover multiple repos | One repo per key |
| Default access | You choose | Read-only by default |
| Production recommendation | Learning / simple | ✅ Preferred |

**Never put credentials in declarative YAML files**

If you write a YAML manifest containing a PAT or SSH key and push it to Git,
the credential is exposed. Use `kubectl create secret` imperatively — the
sensitive value stays in your terminal and never touches a file.

**Three methods, one result**

All three approaches — ArgoCD UI, `kubectl create secret` + `kubectl label`,
and `argocd repo add` — create the same Kubernetes secret with the same label.
Understanding the underlying secret means you can debug any credential issue
regardless of how it was created.

---

## Commands Reference

```bash
# Create HTTPS credential secret
kubectl create secret generic <name> \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/org/repo.git \
  --from-literal=username=<github-username> \
  --from-literal=password=<personal-access-token>

# Create SSH credential secret
kubectl create secret generic <name> \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:org/repo.git \
  --from-literal=sshPrivateKey="$(cat .ssh/argocd-deploy-key)"

# Add mandatory label (required for both)
kubectl label secret <name> \
  -n argocd \
  argocd.argoproj.io/secret-type=repository

# Verify label is present
kubectl describe secret <name> -n argocd | grep -A2 Labels

# List all ArgoCD repo credential secrets
kubectl get secrets -n argocd \
  -l argocd.argoproj.io/secret-type=repository

# Register via argocd CLI (HTTPS)
argocd repo add https://github.com/org/repo.git \
  --username <username> \
  --password <PAT>

# Register via argocd CLI (SSH)
argocd repo add git@github.com:org/repo.git \
  --ssh-private-key-path .ssh/argocd-deploy-key

# List registered repos
argocd repo list

# Remove registered repo
argocd repo rm https://github.com/org/repo.git

# Generate SSH key pair
ssh-keygen -t ed25519 -f .ssh/argocd-deploy-key -N ""
```

---

## Lessons Learned

**1. The label is the on/off switch — not the secret content**

A perfectly valid secret without the label does nothing. This is by design —
ArgoCD scans for secrets with `argocd.argoproj.io/secret-type: repository` and
ignores everything else. If credentials appear correct but ArgoCD still shows
authentication errors, always check the label first.

**2. URL format determines which credential type is used**

When you change a repo from HTTPS to SSH (or vice versa), you must update both
the credential secret URL and the `repoURL` in the Application CRD. Updating
one but not the other causes an auth failure that looks like the credentials are
wrong — but they are just not being matched.

**3. SSH deploy keys do not expire — PATs do**

Fine-grained PATs have configurable expiry dates. When a PAT expires, every
ArgoCD application using it breaks simultaneously with authentication errors.
SSH deploy keys have no mandatory expiry. In production, SSH is more robust.

**4. The three methods are interchangeable — understand the underlying secret**

ArgoCD UI, `kubectl`, and `argocd` CLI all create the same Kubernetes secret.
Knowing the secret schema means you can inspect, debug, and recreate credentials
regardless of how they were originally created — and you will always be able to
understand what ArgoCD is doing internally.

**5. Credential templates save time with multiple repos**

Once you manage more than 2–3 repos under the same GitHub organisation, per-repo
secrets become tedious. The `repo-creds` label value creates a credential template
that matches all repos under a URL prefix. One secret covers all current and
future repos in that org — no per-repo setup needed.

---

## What's Next

**Demo-05: Private Repos and Production Setup**
Build on credential mechanics to establish a full three-repo GitOps structure —
separating application manifests, ArgoCD Application CRDs, and platform config.
Introduce the Docker registry secret (a different secret type from this demo)
for pulling private container images, and deploy a real private image end-to-end.
