# Demo-10: Sync Phases & Hooks — Lifecycle Control with PreSync, PostSync, SyncFail & Skip

## Overview

Demo-09 introduced AppProject governance — controlling what can be deployed,
where, and by whom, all enforced before sync begins. This demo goes one level
deeper into the sync operation itself.

Until now every sync has been a single undifferentiated event — ArgoCD applies
manifests and either succeeds or fails. In production, deployments are not that
simple. You need to run database migrations **before** pods start, verify the
application is reachable **after** pods start, alert the team **if** the
deployment fails, and occasionally **skip** a hook when debugging without
disrupting others.

ArgoCD solves all of this through **sync phases** and **hooks** — a lifecycle
control mechanism that wraps the core sync operation with conditional, ordered,
one-time logic at precisely the right moments.

In this demo we deploy the **Goals App** — a three-tier goal tracking application
(React frontend, Node.js/Express backend, MongoDB database) — and progressively
attach hooks to each phase of the sync lifecycle. Every hook has a real production
motivation, not a contrived example.

**What you'll learn:**
- What a sync operation actually does — and why Synced ≠ Healthy
- The four sync phases: PreSync, Sync, PostSync, SyncFail
- The six hook types and how they map to phases
- The critical difference between phases (where ArgoCD is) and hooks
  (what runs at that point)
- How hook failure propagates through the lifecycle
- Hook delete policies — why hooks clean themselves up
- How the Skip hook bypasses a hook without removing it
- Why Jobs are the most common hook implementation — and what else can be used

**What you'll do:**
- Deploy the Goals App (React + Node.js + MongoDB) with MongoDB authentication
- Attach a PreSync hook to run DB initialisation before any pod starts
- Attach a PostSync hook to run a smoke test after all pods are ready
- Intentionally break the sync to trigger the SyncFail hook
- Use the Skip hook to bypass the PostSync smoke test during a debug cycle

---

## Prerequisites

- ✅ Completed Demo-09 — AppProject governance understood
- ✅ Completed Demo-06 — automated sync, pruning and self-healing understood
- ✅ ArgoCD running on minikube
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `goals-app-config` and `argocd-config`
- ✅ Docker Hub images available:
  - `rselvantech/goals-frontend:v1.0.0`
  - `rselvantech/goals-backend:v1.0.0`

> **If images are not yet in Docker Hub:** Build and push from the Goals App
> source code in your Docker course repository before starting this demo.
> See the Docker Compose Goals App README for build instructions.

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

### 4. Images exist in Docker Hub
```bash
docker pull rselvantech/goals-backend:v1.0.0
docker pull rselvantech/goals-frontend:v1.0.0
```

**Expected:** Both images pulled successfully or `Already up to date`.

If either image is missing, build and push from your Docker course source
before proceeding.

---

## The Goals App — What It Does

The Goals App is a three-tier goal tracking web application from your Docker
course. Users type goals into the React frontend, which calls a Node.js/Express
backend REST API, which persists goals to a MongoDB database. Full CRUD — add
a goal, see it listed, click to delete it.

```
Browser
    │  HTTP :3000
    ▼
goals-frontend pod  (React)
    │  HTTP :80/goals  (REST API calls)
    ▼
goals-backend pod  (Node.js / Express)
    │  MongoDB wire protocol :27017
    ▼
mongodb pod  (mongo official image)
    │
    ▼
PersistentVolumeClaim  (goals-db-data)
```

**Why this app is ideal for hooks:**

Every hook has a genuine, motivated use case with this application:

| Hook | Use case | Why it matters |
|---|---|---|
| PreSync | Initialise MongoDB collection + indexes before backend pods start | Backend crashes on startup if collection schema is wrong |
| PostSync | Curl `goals-backend-svc/goals` and verify HTTP 200 | Proves the full stack is reachable before marking deployment done |
| SyncFail | Echo failure alert (simulates Slack/PagerDuty notification) | Team must know immediately when a production sync fails |
| Skip | Skip the PostSync smoke test during a debug redeploy | Hook already ran — running it again wastes time and adds noise |

---

## Concepts

### What Is a Sync Operation?

A sync operation reconciles the Git-declared desired state into the live cluster
until the live state matches. Two important distinctions:

**Synced ≠ Healthy.**
Sync succeeds when all manifests are applied successfully — not when your pods
are running. A Deployment with a wrong image tag will sync successfully (the YAML
was applied) but the pods will be in `ImagePullBackOff`. Sync status and health
status are independent.

```
Sync Status:   Synced    ← all YAMLs applied successfully ✅
Health Status: Degraded  ← pods are crash-looping ❌ (bad image, bad config)
```

---

### Sync Phases

Sync phases are named execution stages that represent **where ArgoCD is** during
a sync operation. They define the sequence and boundaries:

```
┌─────────────────────────────────────────────────────────────┐
│                    ArgoCD Sync Lifecycle                     │
├──────────┬──────────────────┬──────────┬────────────────────┤
│ PreSync  │      Sync        │ PostSync │     SyncFail       │
│  phase   │      phase       │  phase   │      phase         │
│          │                  │          │                     │
│ Before   │ Manifests        │ After    │ Only if Sync       │
│ sync     │ applied to       │ sync     │ phase fails        │
│ begins   │ cluster          │ completes│                     │
└──────────┴──────────────────┴──────────┴────────────────────┘
```

**PostSync only runs if Sync succeeds. SyncFail only runs if Sync fails.**
They are mutually exclusive — both can never run in the same sync cycle.

---

### Hooks — What They Are and What Can Be a Hook

A hook is **any Kubernetes resource** that ArgoCD handles differently from
normal manifests by annotating it with a hook type. Normal manifests are
reconciled continuously — hooks run once at a specific phase and are then
deleted (based on delete policy).

**Any Kubernetes resource can be a hook** — native or custom. Examples:

| Resource type | Use case |
|---|---|
| `Job` | Most common — DB migrations, smoke tests, notifications. Runs to completion with clear success/failure signal |
| `Pod` | Direct container execution without Job overhead |
| `Workflow` (Argo Workflows) | Complex multi-step hook logic |
| Any CRD | Custom resources that trigger external systems |

Jobs are the idiomatic choice because they run to completion, have a clear
exit code for success/failure, and their logs are preserved after completion
for debugging.

**Hooks are annotated resources:**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync              # which phase to run in
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

**The six hook types:**

| Hook | When it runs | Common use cases |
|---|---|---|
| `PreSync` | Before any manifest is applied | DB migrations, prerequisite checks |
| `Sync` | During manifest apply (rarely used) | Rarely — no clear production use case |
| `PostSync` | After all manifests applied successfully | Smoke tests, integration tests, notifications |
| `SyncFail` | Only when sync fails | Failure alerts, rollback triggers, log collection |
| `PostDelete` | After the Application CRD is deleted | Cleanup of external resources |
| `Skip` | Never — it skips the hook itself | Temporarily bypass a hook without removing it |

---

### Hook Failure — How It Propagates Through the Lifecycle

Hook failure behaviour depends on which phase the hook belongs to:

**PreSync hook fails:**
```
PreSync hook fails
  → Sync phase NEVER starts
  → PostSync NEVER runs
  → SyncFail hook fires (if defined)
  → Application shows SyncError
```
The most protective behaviour — if DB migration fails, no pods are deployed.

**Sync hook fails** (rare):
```
Sync hook fails
  → Sync phase is marked as failed
  → PostSync NEVER runs
  → SyncFail hook fires (if defined)
```

**PostSync hook fails:**
```
Sync phase completed successfully
  → PostSync hook fires
  → PostSync hook fails
  → Application shows SyncError
  → SyncFail hook fires (if defined)
```
Note: The manifests are already applied — only the post-deployment check failed.

**SyncFail hook fails:**
```
SyncFail hook fails
  → Logged as error but no further hooks fire
  → Application still shows SyncError from the original failure
```

**The general rule:**
```
Any hook failure → stops remaining hooks in that phase
                 → triggers SyncFail hook (if defined)
                 → failed hook job stays in cluster for log inspection
```

---

### Phases vs Hooks — The Distinction

This is the most common point of confusion:

| | Phase | Hook |
|---|---|---|
| What it is | Where ArgoCD is in the lifecycle | A Kubernetes resource that runs at a phase |
| Controls | Execution sequence and boundaries | User-defined logic at a specific point |
| Examples | PreSync phase, Sync phase, PostSync phase | A Job that runs `mongosh` during PreSync phase |
| Defined by | ArgoCD internally | You — via annotations on any Kubernetes resource |

A PreSync **hook** runs during the PreSync **phase**. The phase exists regardless
of whether you define any hooks for it. Hooks are optional additions that inject
logic into phases.

---

### Hook Delete Policies

Hooks create Kubernetes resources (usually Jobs). Without cleanup, old hook
Jobs accumulate in the cluster. Delete policies control when ArgoCD removes them:

| Policy | When deleted |
|---|---|
| `HookSucceeded` | Immediately after the hook completes successfully |
| `HookFailed` | Immediately after the hook fails |
| `BeforeHookCreation` | Before a new hook is created (deletes the previous run) |

**Standard pattern — always use both:**
```yaml
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

- `BeforeHookCreation` — deletes any leftover job from a previous sync before
  creating a new one. Prevents name conflicts across multiple syncs.
- `HookSucceeded` — deletes the job after successful completion. Keeps the
  cluster clean.
- If the hook **fails**, the job is **not deleted** — intentionally. The failed
  job stays in the cluster so you can inspect its logs and diagnose the failure.
  Never add `HookFailed` to the delete policy if you want to inspect failures.

---

### Why We Push Application CRD to Git AND `kubectl apply`

Every Application CRD change in this demo follows the two-step pattern:
push to `argocd-config` first, then `kubectl apply`. The push keeps the
configuration version-controlled and recoverable. The apply registers the
change with ArgoCD immediately — because ArgoCD does not watch `argocd-config`
and will not pick it up automatically. This manual apply step is eliminated
in Demo-11 (App-of-Apps). See Demo-05 Concepts for the full explanation.

---

## Folder Structure

```
10-sync-phases-and-hooks/src/
├── goals-app-config/          ← git init → remote: rselvantech/goals-app-config (private)
│   └── demo-10-sync-hooks/
│       ├── namespace.yaml
│       ├── mongodb-pvc.yaml
│       ├── mongodb-deployment.yaml
│       ├── mongodb-service.yaml
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       └── hooks/
│           ├── presync-db-init.yaml
│           ├── postsync-smoke-test.yaml
│           └── syncfail-notify.yaml
└── argocd-config/             ← git init → remote: rselvantech/argocd-config
    └── demo-10-sync-hooks/
        └── goals-app.yaml
```

> `mongodb-secret.yaml` is **never committed to Git**. The Secret is created
> imperatively in Step 1c. This follows the pattern established in Demo-04.

> Only `goals-app-config` needs to be registered with ArgoCD — ArgoCD clones
> it at sync time to read manifests. `argocd-config` is applied manually with
> `kubectl apply` and does not need ArgoCD registration.

---

## Step 1: Setup — Namespace, Secret and Repos

### Step 1a: Create the private `goals-app-config` repo on GitHub

Go to GitHub → New repository:
- Name: `goals-app-config`
- Visibility: **Private**
- Add a README: yes
- Click **Create repository**

### Step 1b: Initialise local repo structure

```bash
cd gitops-labs/argo-cd-basics-to-prod/10-sync-phases-and-hooks/src
mkdir goals-app-config && cd goals-app-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/goals-app-config.git
git pull origin main
```

### Step 1c: Create namespace and MongoDB secret imperatively

Create the namespace first — the secret references it:

```bash
kubectl create namespace goals-app
```

Create the MongoDB credentials secret — **never commit this to Git**:

```bash
kubectl create secret generic mongodb-secret \
  --namespace goals-app \
  --from-literal=MONGODB_USERNAME=rselvantech \
  --from-literal=MONGODB_PASSWORD=passWD \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=rselvantech \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=passWD
```

**Verify:**
```bash
kubectl get secret mongodb-secret -n goals-app
```

**Expected:**
```text
NAME             TYPE     DATA   AGE
mongodb-secret   Opaque   4      5s
```

### Step 1d: Register `goals-app-config` with ArgoCD

`goals-app-config` is a new private repo — ArgoCD needs credentials to clone it:

```bash
argocd repo add https://github.com/rselvantech/goals-app-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

**Verify:**
```bash
argocd repo list
```

**Expected:** `goals-app-config` listed with `Successful` status.

---

## Step 2: Add Manifests to `goals-app-config`

Create all manifest files under `demo-10-sync-hooks/`. We start **without any
hooks** — the Goals App works correctly on its own. Hooks are added progressively
in later steps to demonstrate each phase of the lifecycle.

**`demo-10-sync-hooks/namespace.yaml`:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: goals-app
```

**`demo-10-sync-hooks/mongodb-pvc.yaml`:**
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

**`demo-10-sync-hooks/mongodb-deployment.yaml`:**
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

**`demo-10-sync-hooks/mongodb-service.yaml`:**
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

**`demo-10-sync-hooks/backend-deployment.yaml`:**
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
```

**`demo-10-sync-hooks/backend-service.yaml`:**
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

**`demo-10-sync-hooks/frontend-deployment.yaml`:**
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
            - name: REACT_APP_BACKEND_URL
              value: "http://goals-backend-svc:80"
          stdin: true
          tty: true
```

**`demo-10-sync-hooks/frontend-service.yaml`:**
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

**Push to `goals-app-config`:**
```bash
git add demo-10-sync-hooks/
git commit -m "feat: add demo-10-sync-hooks goals-app manifests (no hooks yet)"
git push origin main
```

---

## Step 3: Create the Application CRD — Manual Sync, No Automation

Manual sync is used deliberately here — hooks appear and complete within seconds.
With automated sync you would miss them. Manual sync lets you trigger each sync
deliberately and observe the full lifecycle in real time.

**Set up `argocd-config` local repo:**

```bash
cd gitops-labs/argo-cd-basics-to-prod/10-sync-phases-and-hooks/src
mkdir argocd-config && cd argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

**Create `demo-10-sync-hooks/goals-app.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sync-phase-hook-demo-goals-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rselvantech/goals-app-config.git
    targetRevision: HEAD
    path: demo-10-sync-hooks
  destination:
    server: https://kubernetes.default.svc
    namespace: goals-app
```

> **Why push to Git AND `kubectl apply`?** ArgoCD watches `goals-app-config`
> (the source repo) — not `argocd-config` where the Application CRD lives.
> Pushing to `argocd-config` does not trigger ArgoCD to pick up the change
> automatically — `kubectl apply` is still required. We push to Git to
> version-control the Application CRD — keeping it auditable and recoverable.
> This manual apply step is eliminated in Demo-11 (App-of-Apps).

**Push and apply:**
```bash
git add demo-10-sync-hooks/goals-app.yaml
git commit -m "feat: add demo-10 Application CRD for goals-app hooks demo"
git push origin main

kubectl apply -f demo-10-sync-hooks/goals-app.yaml
```

**Verify:**
```bash
argocd app get sync-phase-hook-demo-goals-app
```

**Expected:**
```text
Sync Status:   OutOfSync
Health Status: Missing
```

**Sync manually to deploy the Goals App without any hooks:**
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

**Verify all pods running:**
```bash
kubectl get pods -n goals-app
```

**Expected:**
```text
NAME                              READY   STATUS    RESTARTS
mongodb-xxxxxxxxx-xxxxx           1/1     Running   0
goals-backend-xxxxxxxxx-xxxxx     1/1     Running   0
goals-frontend-xxxxxxxxx-xxxxx    1/1     Running   0
```

The Goals App is now running correctly **without any hooks**. This is the
baseline — the app works on its own. Hooks are added in the next steps to
demonstrate lifecycle control, not to make the app work.

---

## Step 4: Add PreSync Hook — DB Initialisation

### Why this hook — and why it matters even though the app already works

The Goals App is running correctly in Step 3 without this hook. So why add it?

In production, new releases often require database changes — adding a collection,
creating an index, changing a schema. If the new version of the backend pod starts
before the migration runs, it crashes because the database is not ready for it.
The PreSync hook solves this by ensuring the migration completes **before any
manifest is applied** — including the backend Deployment.

```
Without PreSync hook:           With PreSync hook:
─────────────────────           ──────────────────────────────────
Sync starts                     PreSync phase starts
  → backend pod starts          → DB migration job runs
  → tries to use new schema     → waits for MongoDB to be ready
  → MongoDB not ready           → creates collection + index
  → backend crashes             → migration completes
  → ImagePullBackOff or         Sync phase starts
    CrashLoopBackOff              → backend pod starts
                                  → schema is ready
                                  → backend works correctly
```

### What this hook does — command by command

The hook is a Kubernetes **Job** running the `mongo:6.0` image. It executes
a shell script that does two things:

**1. Wait for MongoDB to be ready:**
```bash
until mongosh \
  --host mongodb.goals-app.svc.cluster.local \
  --username $MONGODB_USERNAME \
  --password $MONGODB_PASSWORD \
  --authenticationDatabase admin \
  --eval "db.adminCommand('ping')" > /dev/null 2>&1; do
  echo "MongoDB not ready yet, retrying in 3s..."
  sleep 3
done
```
- `mongosh` connects to MongoDB using the credentials from the `mongodb-secret`
- `db.adminCommand('ping')` is the standard MongoDB health check — returns
  `{ok: 1}` when MongoDB is accepting connections
- `> /dev/null 2>&1` suppresses output — we only care about the exit code
- The `until` loop retries every 3 seconds until the ping succeeds
- On the first sync MongoDB is being deployed in parallel during the Sync phase
  while this hook is in the PreSync phase — the retry loop waits for it

**2. Create the collection and index:**
```bash
mongosh \
  --host mongodb.goals-app.svc.cluster.local \
  --username $MONGODB_USERNAME \
  --password $MONGODB_PASSWORD \
  --authenticationDatabase admin \
  course-goals \                        # ← connect to this database
  --eval "
    db.createCollection('goals');       # ← create the goals collection
    db.goals.createIndex({ text: 'text' });  # ← add text index for search
    print('Collection and index created successfully.');
  "
```
- `course-goals` is the database name the backend uses
- `createCollection` creates the `goals` collection if it does not exist
- `createIndex({ text: 'text' })` adds a full-text search index on the `text`
  field — this is what the backend queries when listing goals
- Running this again on a subsequent sync is safe — MongoDB ignores
  `createCollection` and `createIndex` if they already exist

**Create `demo-10-sync-hooks/hooks/presync-db-init.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: presync-db-init
  namespace: goals-app
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-init
          image: mongo:6.0
          command:
            - /bin/sh
            - -c
            - |
              echo "=== PreSync: Starting DB initialisation ==="
              echo "Waiting for MongoDB to be ready..."
              until mongosh \
                --host mongodb.goals-app.svc.cluster.local \
                --username $MONGODB_USERNAME \
                --password $MONGODB_PASSWORD \
                --authenticationDatabase admin \
                --eval "db.adminCommand('ping')" > /dev/null 2>&1; do
                echo "MongoDB not ready yet, retrying in 3s..."
                sleep 3
              done
              echo "MongoDB is ready."
              echo "Creating goals collection and text index..."
              mongosh \
                --host mongodb.goals-app.svc.cluster.local \
                --username $MONGODB_USERNAME \
                --password $MONGODB_PASSWORD \
                --authenticationDatabase admin \
                course-goals \
                --eval "
                  db.createCollection('goals');
                  db.goals.createIndex({ text: 'text' });
                  print('Collection and index created successfully.');
                "
              echo "=== PreSync: DB initialisation complete ==="
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
```

**Push:**
```bash
git add demo-10-sync-hooks/hooks/presync-db-init.yaml
git commit -m "feat: add PreSync DB initialisation hook for demo-10"
git push origin main
```

### Sync and observe PreSync hook

Open a second terminal and watch jobs:
```bash
kubectl get jobs -n goals-app -w
```

In the first terminal, sync:
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

**Expected sequence in the watch terminal:**
```text
NAME                STATUS     COMPLETIONS   AGE
presync-db-init     Running    0/1           3s
presync-db-init     Running    0/1           8s
presync-db-init     Complete   1/1           15s
# job deleted immediately after completion (HookSucceeded policy)
```

After the PreSync job completes, the Sync phase begins and all application
manifests are applied.

**Inspect PreSync job logs** — run quickly before the job is deleted:
```bash
kubectl logs -l job-name=presync-db-init -n goals-app
```

**Expected:**
```text
=== PreSync: Starting DB initialisation ===
Waiting for MongoDB to be ready...
MongoDB is ready.
Creating goals collection and text index...
Collection and index created successfully.
=== PreSync: DB initialisation complete ===
```

**Verify all pods still running:**
```bash
kubectl get pods -n goals-app
```

**Expected:**
```text
NAME                              READY   STATUS    RESTARTS
mongodb-xxxxxxxxx-xxxxx           1/1     Running   0
goals-backend-xxxxxxxxx-xxxxx     1/1     Running   0
goals-frontend-xxxxxxxxx-xxxxx    1/1     Running   0
```

> The PreSync hook ran before the Sync phase. The Deployment manifests were
> already applied in Step 3 — this sync re-applied them unchanged while the
> hook ran first. In ArgoCD UI, click on `sync-phase-hook-demo-goals-app` and
> you will briefly see the `presync-db-init` job appear in the resource tree
> before being deleted by the delete policy.

---

## Step 5: Access the Goals App

**Port-forward the frontend service:**
```bash
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
```

Open `http://localhost:3000` in your browser.

**Try the full workflow:**
1. Type a goal: `"Learn ArgoCD sync hooks"`
2. Click **Add Goal** — goal appears in the list
3. Reload the page — goal persists (stored in MongoDB via the initialised collection)
4. Click the goal to delete it — goal disappears

**Verify the backend API directly:**
```bash
kubectl port-forward svc/goals-backend-svc -n goals-app 8080:80
curl http://localhost:8080/goals
```

**Expected:**
```json
{"goals": []}
```

---

## Step 6: Add PostSync Hook — Smoke Test

After all manifests are applied successfully, the PostSync hook verifies the
backend API is reachable by curling it from inside the cluster. This runs
**after** the Sync phase completes — confirming the full stack is healthy
before marking the deployment done.

**Create `demo-10-sync-hooks/hooks/postsync-smoke-test.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postsync-smoke-test
  namespace: goals-app
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: smoke-test
          image: curlimages/curl:8.5.0
          command:
            - /bin/sh
            - -c
            - |
              echo "=== PostSync: Starting smoke test ==="
              echo "Waiting for backend to be ready..."
              until curl -sf http://goals-backend-svc.goals-app.svc.cluster.local/goals \
                -o /dev/null; do
                echo "Backend not ready yet, retrying in 3s..."
                sleep 3
              done
              echo "Backend responded successfully."
              echo "=== PostSync: Smoke test PASSED ==="
```

**What this hook does:**
- Uses `curlimages/curl` — a minimal image with only `curl` installed
- `curl -sf` — `-s` suppresses progress output, `-f` returns non-zero exit
  code on HTTP errors (4xx, 5xx)
- `-o /dev/null` suppresses the response body — we only care about the
  HTTP status code
- The `until` loop retries every 3 seconds until the backend responds
  with HTTP 200 — accounts for pod startup time
- If the backend never responds, the job fails → PostSync fails →
  SyncFail hook fires (once we add it in Step 7)

**Push:**
```bash
git add demo-10-sync-hooks/hooks/postsync-smoke-test.yaml
git commit -m "feat: add PostSync smoke test hook for demo-10"
git push origin main
```

**Trigger a sync to see the PostSync hook in action:**

We need to trigger a sync to see both PreSync and PostSync hooks running.
Push a small change to trigger it:

```bash
# Edit demo-10-sync-hooks/backend-deployment.yaml — add a label:
#   labels:
#     version: "v1.0.0"

git add demo-10-sync-hooks/backend-deployment.yaml
git commit -m "chore: add version label to trigger sync with PostSync hook"
git push origin main
```

**Sync:**
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

**Watch the full lifecycle in the second terminal:**
```bash
kubectl get jobs -n goals-app -w
```

**Expected sequence:**
```text
presync-db-init      Running    0/1    3s
presync-db-init      Complete   1/1    15s   ← PreSync done, Sync begins
# (manifests applied during Sync phase)
postsync-smoke-test  Running    0/1    45s   ← PostSync begins after Sync
postsync-smoke-test  Complete   1/1    52s   ← smoke test passed, job deleted
```

The full sync lifecycle — PreSync → Sync → PostSync — completed with both hooks
running at the correct phases.

---

## Step 7: Trigger SyncFail Hook

### 7a: Add the SyncFail hook first — before breaking anything

The SyncFail hook must exist in the repo and be synced at least once before it
can fire. This is because ArgoCD only knows about resources that have been
successfully applied to the cluster. If you add the SyncFail hook and break
the sync in the same commit, ArgoCD has not yet registered the hook and it
will not fire on that first broken sync.

**The correct sequence is:**
1. Add SyncFail hook → sync cleanly → hook is registered ✅
2. Break the sync → SyncFail hook fires ✅

**What this hook does:**
The `syncfail-notify` job runs an `alpine` container that echoes a simulated
alert message. In production, this command would be replaced with a `curl`
webhook call to Slack, PagerDuty, or your alerting system. The pattern is
identical — only the command changes.

**Create `demo-10-sync-hooks/hooks/syncfail-notify.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: syncfail-notify
  namespace: goals-app
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: alpine:3.19
          command:
            - /bin/sh
            - -c
            - |
              echo "=== SyncFail: Sync operation FAILED ==="
              echo "Application: sync-phase-hook-demo-goals-app"
              echo "Namespace:   goals-app"
              echo "Time:        $(date)"
              echo "Action:      Alerting on-call team (simulated)"
              echo "In production: replace with curl webhook to Slack/PagerDuty"
              echo "=== SyncFail: Notification sent ==="
```

**Push and sync cleanly to register the hook:**
```bash
git add demo-10-sync-hooks/hooks/syncfail-notify.yaml
git commit -m "feat: add SyncFail notification hook for demo-10"
git push origin main

argocd app sync sync-phase-hook-demo-goals-app
```

**Verify:**
```bash
argocd app get sync-phase-hook-demo-goals-app
```

**Expected:** `Synced` and `Healthy` — the SyncFail hook is now registered but
did not fire because the sync succeeded.

### 7b: Introduce a bad manifest to break sync

Now that the SyncFail hook is registered, deliberately break the sync by adding
an invalid manifest — a Deployment with a missing required field (`spec.selector`).
This causes `kubectl apply` to fail, which triggers the SyncFail phase.

**Create `demo-10-sync-hooks/bad-manifest.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deployment
  namespace: goals-app
spec:
  replicas: 1
  # spec.selector is missing — required field — kubectl apply will fail
  template:
    metadata:
      labels:
        app: broken
    spec:
      containers:
        - name: broken
          image: nginx
```

**Push:**
```bash
git add demo-10-sync-hooks/bad-manifest.yaml
git commit -m "test: add invalid manifest to trigger SyncFail hook"
git push origin main
```

**Sync:**
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

**Watch:**
```bash
kubectl get jobs -n goals-app -w
```

**Expected — `syncfail-notify` job fires:**
```text
syncfail-notify   Running    0/1    2s
syncfail-notify   Complete   1/1    8s
```

**Check the SyncFail job logs:**
```bash
kubectl logs -l job-name=syncfail-notify -n goals-app
```

**Expected:**
```text
=== SyncFail: Sync operation FAILED ===
Application: sync-phase-hook-demo-goals-app
Namespace:   goals-app
Time:        Mon Mar 23 14:30:00 UTC 2026
Action:      Alerting on-call team (simulated)
In production: replace with curl webhook to Slack/PagerDuty
=== SyncFail: Notification sent ===
```

**Verify the sync error in ArgoCD:**
```bash
argocd app get sync-phase-hook-demo-goals-app
```

**Expected:**
```text
Sync Status:   OutOfSync
CONDITION:     SyncError — deployment.apps "broken-deployment" is invalid:
               spec.selector: Required value
```

**Key observation:** The existing Goals App pods are still running and healthy.
The bad manifest caused the Sync phase to fail but the previously applied
Deployments (mongodb, goals-backend, goals-frontend) were not affected — ArgoCD
only failed on the new broken manifest.

### 7c: Fix the bad manifest

```bash
git rm demo-10-sync-hooks/bad-manifest.yaml
git commit -m "fix: remove invalid manifest"
git push origin main
```

**Sync again — full lifecycle completes cleanly:**
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

**Expected:** PreSync → Sync → PostSync all complete successfully.

---

## Step 8: Skip Hook — Bypass PostSync Smoke Test

The Skip hook is used to temporarily bypass a hook without removing it. This is
useful when you need to redeploy quickly — the PostSync smoke test already ran
on the last sync and you know the backend is healthy. Running it again adds time
and noise without adding value.

**Why not just delete the hook YAML?** Deleting the YAML removes the hook
permanently and the change is not obviously reversible. Adding `Skip` to the
annotation is a deliberate, auditable, temporary bypass — the Git history shows
exactly when it was skipped, by whom, and the commit message explains why.

### 8a: Add Skip annotation to the PostSync hook

Edit `demo-10-sync-hooks/hooks/postsync-smoke-test.yaml` — add `Skip`:

```yaml
annotations:
  argocd.argoproj.io/hook: PostSync,Skip    # ← Skip added
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

**Push:**
```bash
git add demo-10-sync-hooks/hooks/postsync-smoke-test.yaml
git commit -m "chore: add Skip to PostSync hook for debug cycle"
git push origin main
```

**Sync:**
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

**Watch:**
```bash
kubectl get jobs -n goals-app -w
```

**Expected — only PreSync fires, PostSync is skipped entirely:**
```text
presync-db-init    Running    0/1    3s
presync-db-init    Complete   1/1    15s
# postsync-smoke-test does NOT appear — it was skipped
```

Verify in ArgoCD UI — the `postsync-smoke-test` job does not appear in the
resource tree during this sync.

### 8b: Restore the PostSync hook

```yaml
annotations:
  argocd.argoproj.io/hook: PostSync    # ← Skip removed
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

```bash
git add demo-10-sync-hooks/hooks/postsync-smoke-test.yaml
git commit -m "feat: restore PostSync hook after debug cycle"
git push origin main
```

---

## Verify Final State

```bash
# Application synced and healthy
argocd app get sync-phase-hook-demo-goals-app

# All three pods running
kubectl get pods -n goals-app

# No leftover hook jobs (all deleted by delete policy)
kubectl get jobs -n goals-app

# Secret exists
kubectl get secret mongodb-secret -n goals-app

# goals-app-config registered
argocd repo list
```

**Access the Goals App to confirm end-to-end:**
```bash
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
```

Open `http://localhost:3000` — add a goal, reload, confirm it persists.

---

## Cleanup

```bash
# Delete the Application
kubectl delete app sync-phase-hook-demo-goals-app -n argocd

# Delete the namespace (removes all resources including the secret)
kubectl delete namespace goals-app

# Deregister the repo
argocd repo rm https://github.com/rselvantech/goals-app-config.git
```

> The `goals-app-config` GitHub repo can be kept — it is useful for future
> reference. The MongoDB secret is deleted with the namespace.

---

## Key Concepts Summary

**Sync operation — what it really does**
A sync operation applies all Git manifests to the cluster. Success means all
manifests were applied without error — not that pods are healthy. Sync status
and health status are independent.

**Phases are where ArgoCD is. Hooks are what runs there.**
Phases (PreSync, Sync, PostSync, SyncFail) define the lifecycle sequence.
Hooks are annotated Kubernetes resources that run at a specific phase. The
phase exists whether or not you define a hook for it.

**Any Kubernetes resource can be a hook**
Jobs are the most common choice — they run to completion with a clear
success/failure signal. But Pods, Workflows, or any CRD can be annotated
as a hook. The annotation is what makes it a hook, not the resource type.

**Hook failure propagates through the lifecycle**
A failed PreSync hook prevents Sync from starting. A failed PostSync hook
marks the sync as failed even though manifests were applied. Any hook failure
triggers the SyncFail hook if one is defined.

**PostSync and SyncFail are mutually exclusive**
PostSync runs only if Sync succeeds. SyncFail runs only if Sync fails. Both
can never run in the same sync cycle.

**Delete policies keep the cluster clean**
`BeforeHookCreation,HookSucceeded` is the standard pattern — deletes the
previous run before creating a new one, and deletes on success. Failed jobs
stay in the cluster intentionally for log inspection.

**Skip is a temporary bypass, not a deletion**
Adding `Skip` to a hook annotation makes ArgoCD ignore that hook for the
sync. The hook YAML stays in Git. Audited in Git history. Remove `Skip` to
restore normal behaviour.

---

## Commands Reference

```bash
# Apply Application CRD
kubectl apply -f demo-10-sync-hooks/goals-app.yaml

# Manual sync
argocd app sync sync-phase-hook-demo-goals-app

# Refresh without applying
argocd app get sync-phase-hook-demo-goals-app --refresh

# Watch hook jobs in real time
kubectl get jobs -n goals-app -w

# Get hook job logs (run quickly — job is deleted on success)
kubectl logs -l job-name=presync-db-init -n goals-app
kubectl logs -l job-name=postsync-smoke-test -n goals-app
kubectl logs -l job-name=syncfail-notify -n goals-app

# Check app status and sync error details
argocd app get sync-phase-hook-demo-goals-app

# See sync history
argocd app history sync-phase-hook-demo-goals-app

# Access Goals App
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
kubectl port-forward svc/goals-backend-svc -n goals-app 8080:80

# Test backend API directly
curl http://localhost:8080/goals

# Check Deployment events (useful for debugging hook interactions)
kubectl describe deployment goals-backend -n goals-app | grep -A20 Events
```

---

## Lessons Learned

**1. Synced does not mean healthy — understand both status indicators**
A sync succeeds when manifests are applied. A deployment with a wrong image tag
syncs successfully but the pods are in `ImagePullBackOff`. Always check both
Sync Status and Health Status after a deployment.

**2. Hooks add lifecycle control — not functionality**
The Goals App works correctly without any hooks. Hooks add control around
deployment safety (PreSync migration), deployment verification (PostSync smoke
test), and operational visibility (SyncFail alert). Add hooks when you need to
control what happens before or after deployment — not to make the app work.

**3. PreSync hooks protect you from deploying against an incompatible schema**
If your DB migration fails, nothing gets deployed. This is the primary production
value of PreSync hooks — preventing application code from starting against a
database that is not ready for it.

**4. SyncFail hooks must be registered before they can fire**
Add the SyncFail hook in a clean sync first. If you add it and break the sync
in the same commit, the hook was never registered and will not fire on that
first broken sync.

**5. Hook delete policy — leave failed jobs for inspection**
`BeforeHookCreation,HookSucceeded` deletes on success but keeps the job on
failure. This is intentional — when a hook fails you need its logs to diagnose
the problem. Never add `HookFailed` to the delete policy if you want to inspect
failures.

**6. Skip is an audit-friendly temporary bypass**
Disabling a hook by editing the annotation and pushing a commit is better than
deleting the hook YAML. The Git history shows exactly when the hook was bypassed,
by whom, and the commit message explains why.

**7. Existing resources survive a failed sync**
When sync fails due to a bad manifest, previously applied resources (pods,
services) continue running. ArgoCD only failed on the new broken manifest.
The app remains healthy even when the sync is broken.

---

## What's Next

**Demo-11: App-of-Apps Pattern**
Manage multiple ArgoCD Applications declaratively using a parent Application
that deploys child Applications. Eliminate the manual `kubectl apply` for each
Application CRD and AppProject YAML — let ArgoCD manage them from Git
automatically. The sync hooks foundation built in this demo carries forward —
Demo-11 introduces sync waves to control ordering across multiple applications.