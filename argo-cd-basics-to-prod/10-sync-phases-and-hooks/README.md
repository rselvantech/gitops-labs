# Demo-10: Sync Phases & Hooks — Lifecycle Control with PreSync, PostSync, SyncFail & Skip

## Overview

Until now, every sync operation has been a single undifferentiated event — ArgoCD
applies manifests and either succeeds or fails. In production, deployments are not
that simple. You need to run database migrations **before** pods start, verify the
application is reachable **after** pods start, alert the team **if** the deployment
fails, and occasionally **skip** a hook when debugging without disrupting others.

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
- How hook failure stops the entire sync lifecycle
- Hook delete policies — why hooks clean themselves up
- How the Skip hook bypasses a hook without removing it
- Why Jobs are the most common hook implementation

**What you'll do:**
- Deploy the Goals App (React + Node.js + MongoDB) with MongoDB authentication
- Attach a PreSync hook to run DB initialisation before any pod starts
- Attach a PostSync hook to run a smoke test after all pods are ready
- Intentionally break the sync to trigger the SyncFail hook
- Use the Skip hook to bypass the PostSync smoke test during a debug cycle

## Prerequisites

- ✅ Completed Demo-09 — AppProject governance understood
- ✅ ArgoCD running on minikube (`kubectl get pods -n argocd`)
- ✅ ArgoCD CLI installed and logged in
  (`argocd login localhost:8080 --username admin --insecure`)
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `goals-app-config` and `argocd-config`
- ✅ Docker Hub images available:
  - `rselvantech/goals-frontend:v1.0.0`
  - `rselvantech/goals-backend:v1.0.0`

> **If images are not yet in Docker Hub:** Build and push from the Goals App
> source code in your Docker course repository before starting this demo.
> See the Docker Compose Goals App README for build instructions.

**Verify prerequisites:**
```bash
kubectl get pods -n argocd
argocd app list
docker pull rselvantech/goals-backend:v1.0.0    # verify image exists
docker pull rselvantech/goals-frontend:v1.0.0   # verify image exists
```

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
| Skip | Skip the PostSync smoke test during a debug redeploy | Hook already ran; running it again wastes time and adds noise |

---

## Background: What Is a Sync Operation?

A sync operation reconciles the Git-declared desired state into the live cluster
until the live state matches. Two important distinctions:

**Synced ≠ Healthy.**
Sync succeeds when all manifests are applied successfully. It does not mean
your pods are running. A Deployment with a wrong image tag will sync successfully
(the Deployment manifest is valid YAML and was applied) but the pods will be in
`ImagePullBackOff`. Sync status and health status are independent.

**Sync success = manifests applied, not workloads healthy.**

```
Sync Status:   Synced    ← all YAMLs applied successfully ✅
Health Status: Degraded  ← pods are crash-looping ❌ (bad image, bad config)
```

---

## Background: Sync Phases

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
They are mutually exclusive.

---

## Background: Hooks

A hook is a Kubernetes resource — most commonly a Job — that ArgoCD handles
differently from normal manifests. Normal manifests are reconciled continuously.
Hooks run once at a specific phase and are then deleted (based on delete policy).

**Hooks are annotated resources:**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync              # which phase to run in
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # when to delete
```

**The six hook types:**

| Hook | When it runs | Common use cases |
|---|---|---|
| `PreSync` | Before any manifest is applied | DB migrations, prerequisite checks |
| `Sync` | During manifest apply (rarely used) | Rarely — no clear use case |
| `PostSync` | After all manifests applied successfully | Smoke tests, integration tests, notifications |
| `SyncFail` | Only when sync fails | Failure alerts, rollback triggers, log collection |
| `PostDelete` | After the Application CRD is deleted | Cleanup of external resources |
| `Skip` | Never — it skips the hook | Temporarily bypass a hook |

**Hook failure stops the lifecycle:**

```
PreSync hook fails  →  Sync phase never starts  →  PostSync never runs
PreSync hook succeeds  →  Sync phase runs  →  PostSync runs (if sync succeeds)
```

This is the most important behaviour to understand. If your PreSync DB migration
fails, ArgoCD will not proceed to deploy pods — protecting you from deploying
application code against an incompatible database schema.

---

## Background: Phases vs Hooks — The Distinction

This is the most common point of confusion:

| | Phase | Hook |
|---|---|---|
| What it is | Where ArgoCD is in the lifecycle | A Kubernetes resource that runs at a phase |
| Controls | Execution sequence and boundaries | User-defined logic at a specific point |
| Examples | PreSync phase, Sync phase, PostSync phase | A Job that runs `mongosh` during PreSync phase |
| Defined by | ArgoCD internally | You — via annotations on a Job or other resource |

A PreSync **hook** runs during the PreSync **phase**. The phase exists regardless
of whether you define any hooks for it. Hooks are optional additions that inject
logic into phases.

---

## Background: Hook Delete Policies

Hooks create Kubernetes resources (usually Jobs). Without cleanup, old hook Jobs
accumulate in the cluster. Delete policies control when ArgoCD removes them:

| Policy | When deleted |
|---|---|
| `HookSucceeded` | Immediately after the hook completes successfully |
| `HookFailed` | Immediately after the hook fails |
| `BeforeHookCreation` | Before a new hook is created (deletes the previous run) |

**Common pattern — use both `BeforeHookCreation` and `HookSucceeded`:**

```yaml
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

- `BeforeHookCreation` — deletes any leftover job from a previous sync before
  creating the new one. Prevents name conflicts.
- `HookSucceeded` — deletes the job after successful completion. Keeps cluster clean.
- If the hook **fails**, the job is **not deleted** — intentionally. The failed job
  stays in the cluster so you can inspect its logs and diagnose the failure.

---

## Folder Structure

```
10-sync-phases-and-hooks/src/
├── goals-app-config/          ← git init → remote: rselvantech/goals-app-config (private)
│   └── demo-10/
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
    └── demo-10/
        └── goals-app.yaml
```

> `mongodb-secret.yaml` is **never committed to Git**. The Secret is created
> imperatively in Step 1c. This follows the pattern established in Demo-04.

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
cd gitops-labs/argo-cd-basics-to-prod/10-sync-phases-and-hooks/src/goals-app-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/goals-app-config.git
git pull origin main --allow-unrelated-histories --no-rebase
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

Verify:
```bash
kubectl get secret mongodb-secret -n goals-app
```

Expected:
```
NAME             TYPE     DATA   AGE
mongodb-secret   Opaque   4      5s
```

### Step 1d: Register `goals-app-config` with ArgoCD

```bash
argocd repo add https://github.com/rselvantech/goals-app-config.git \
  --username rselvantech \
  --password <GITHUB_PAT>
```

Verify:
```bash
argocd repo list
```

Expected — `goals-app-config` listed with `Successful` status.

---

## Step 2: Add Manifests to `goals-app-config`

Create all manifest files under `demo-10/`. Start without any hooks — we add
them progressively through the demo steps.

**`demo-10/namespace.yaml`:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: goals-app
```

**`demo-10/mongodb-pvc.yaml`:**
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

**`demo-10/mongodb-deployment.yaml`:**
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

**`demo-10/mongodb-service.yaml`:**
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

**`demo-10/backend-deployment.yaml`:**
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

**`demo-10/backend-service.yaml`:**
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

**`demo-10/frontend-deployment.yaml`:**
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

**`demo-10/frontend-service.yaml`:**
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

Push to `goals-app-config`:
```bash
git add demo-10/
git commit -m "feat: add demo-10 goals-app manifests (no hooks yet)"
git push origin main
```

---

## Step 3: Create the Application CRD — Manual Sync, No Automation

Set up `argocd-config` local repo:

```bash
cd gitops-labs/argo-cd-basics-to-prod/10-sync-phases-and-hooks/src/argocd-config

git init
git branch -M main
git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/argocd-config.git
git pull origin main --allow-unrelated-histories --no-rebase
```

Create `demo-10/goals-app.yaml`:

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
    path: demo-10
  destination:
    server: https://kubernetes.default.svc
    namespace: goals-app
```

> No `syncPolicy` block — manual sync only. This lets us trigger each sync
> deliberately and observe hooks appearing and completing in real time.

Push and apply:
```bash
git add demo-10/goals-app.yaml
git commit -m "feat: add demo-10 Application CRD for goals-app hooks demo"
git push origin main

kubectl apply -f demo-10/goals-app.yaml
```

Verify:
```bash
argocd app get sync-phase-hook-demo-goals-app
```

Expected:
```
Sync Status:   OutOfSync
Health Status: Missing
```

---

## Step 4: Add PreSync Hook — DB Initialisation

Before any application pod starts, we initialise the MongoDB database — create
the `goals` collection and add a text index on the `text` field. This mirrors
a real production DB migration job.

Create `demo-10/hooks/presync-db-init.yaml` in `goals-app-config`:

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

**Key points in this hook:**
- `argocd.argoproj.io/hook: PreSync` — runs in the PreSync phase, before any
  manifest is applied
- `BeforeHookCreation,HookSucceeded` — deletes the previous run before creating
  a new one, then deletes itself on success. If it fails, the job stays for
  log inspection
- The `until` loop waits for MongoDB to be ready — the MongoDB pod is already
  running from a previous sync cycle. On the very first sync, the PreSync hook
  runs but MongoDB may not be up yet — the retry loop handles this

> **First sync order note:** On the very first sync, the PreSync hook fires
> before any manifest is applied — meaning MongoDB itself is not yet deployed.
> The retry loop in the hook waits for MongoDB to become available. MongoDB is
> applied during the Sync phase while the hook is waiting. This is expected
> behaviour — the hook polls until the dependency is ready.

Push:
```bash
git add demo-10/hooks/presync-db-init.yaml
git commit -m "feat: add PreSync DB initialisation hook for demo-10"
git push origin main
```

### Sync and observe PreSync hook

```bash
argocd app sync sync-phase-hook-demo-goals-app
```

Watch in a second terminal:
```bash
kubectl get jobs -n goals-app -w
```

Expected sequence:
```
NAME                STATUS     COMPLETIONS   AGE
presync-db-init     Running    0/1           3s
presync-db-init     Running    0/1           8s
presync-db-init     Complete   1/1           15s
# job is deleted immediately after completion (HookSucceeded policy)
```

After the PreSync job completes, the Sync phase begins and all application
manifests are applied.

Verify all pods running:
```bash
kubectl get pods -n goals-app
```

Expected:
```
NAME                              READY   STATUS    RESTARTS
mongodb-xxxxxxxxx-xxxxx           1/1     Running   0
goals-backend-xxxxxxxxx-xxxxx     1/1     Running   0
goals-frontend-xxxxxxxxx-xxxxx    1/1     Running   0
```

Inspect PreSync job logs before it is deleted (run quickly after sync starts):
```bash
kubectl logs -l job-name=presync-db-init -n goals-app
```

Expected output:
```
=== PreSync: Starting DB initialisation ===
Waiting for MongoDB to be ready...
MongoDB is ready.
Creating goals collection and text index...
Collection and index created successfully.
=== PreSync: DB initialisation complete ===
```

Verify in ArgoCD UI — go to `sync-phase-hook-demo-goals-app` → you will see
the `presync-db-init` job appear briefly in the resource tree before being
deleted by the delete policy.

---

## Step 5: Access the Goals App

Port-forward the frontend service:
```bash
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
```

Open `http://localhost:3000` in your browser.

Try the full workflow:
1. Type a goal: `"Learn ArgoCD sync hooks"`
2. Click **Add Goal** — goal appears in the list
3. Reload the page — goal persists (stored in MongoDB)
4. Click the goal to delete it — goal disappears

Port-forward the backend directly to verify the API:
```bash
kubectl port-forward svc/goals-backend-svc -n goals-app 8080:80
curl http://localhost:8080/goals
```

Expected:
```json
{"goals": []}
```

---

## Step 6: Add PostSync Hook — Smoke Test

After all manifests are applied successfully, the PostSync hook verifies the
backend API is reachable by curling it from inside the cluster.

Create `demo-10/hooks/postsync-smoke-test.yaml` in `goals-app-config`:

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

Push:
```bash
git add demo-10/hooks/postsync-smoke-test.yaml
git commit -m "feat: add PostSync smoke test hook for demo-10"
git push origin main
```

Trigger a new sync to activate the PostSync hook. Push a small change to
trigger it:

```bash
# In goals-app-config — update a label to trigger a sync
# Edit demo-10/backend-deployment.yaml — add a label:
#   labels:
#     version: "v1.0.0"

git add demo-10/backend-deployment.yaml
git commit -m "chore: add version label to trigger sync with PostSync hook"
git push origin main
```

Sync:
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

Watch the full lifecycle:
```bash
kubectl get jobs -n goals-app -w
```

Expected sequence:
```
presync-db-init      Running    0/1    3s
presync-db-init      Complete   1/1    15s   ← PreSync done, Sync begins
# (manifests applied during Sync phase)
postsync-smoke-test  Running    0/1    45s   ← PostSync begins after Sync
postsync-smoke-test  Complete   1/1    52s   ← smoke test passed, job deleted
```

The full sync lifecycle — PreSync → Sync → PostSync — completed with both hooks
running at the right phases.

---

## Step 7: Trigger SyncFail Hook

### 7a: Add the SyncFail hook first

Create `demo-10/hooks/syncfail-notify.yaml` in `goals-app-config`:

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
              echo "In production: send webhook to Slack/PagerDuty here"
              echo "=== SyncFail: Notification sent ==="
```

Push:
```bash
git add demo-10/hooks/syncfail-notify.yaml
git commit -m "feat: add SyncFail notification hook for demo-10"
git push origin main
```

Sync once cleanly to register the hook:
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

### 7b: Introduce a bad manifest to break sync

Add a deliberately invalid manifest to `goals-app-config/demo-10/`:

Create `demo-10/bad-manifest.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deployment
  namespace: goals-app
spec:
  replicas: 1
  # selector is missing — required field — this will fail validation
  template:
    metadata:
      labels:
        app: broken
    spec:
      containers:
        - name: broken
          image: nginx
```

Push:
```bash
git add demo-10/bad-manifest.yaml
git commit -m "test: add invalid manifest to trigger SyncFail hook"
git push origin main
```

Sync:
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

Watch:
```bash
kubectl get jobs -n goals-app -w
```

Expected — `syncfail-notify` job fires:
```
syncfail-notify   Running    0/1    2s
syncfail-notify   Complete   1/1    8s
```

Check the SyncFail job logs:
```bash
kubectl logs -l job-name=syncfail-notify -n goals-app
```

Expected:
```
=== SyncFail: Sync operation FAILED ===
Application: sync-phase-hook-demo-goals-app
Namespace:   goals-app
Time:        Thu Mar 20 14:30:00 UTC 2026
Action:      Alerting on-call team (simulated)
In production: send webhook to Slack/PagerDuty here
=== SyncFail: Notification sent ===
```

Verify in ArgoCD:
```bash
argocd app get sync-phase-hook-demo-goals-app
```

Expected:
```
Sync Status:   OutOfSync
CONDITION:     SyncError — deployment.apps "broken-deployment" is invalid:
               spec.selector: Required value
```

### 7c: Fix the bad manifest

```bash
git rm demo-10/bad-manifest.yaml
git commit -m "fix: remove invalid manifest"
git push origin main
```

Sync again — lifecycle completes cleanly:
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

Expected: PreSync → Sync → PostSync all complete successfully.

---

## Step 8: Skip Hook — Bypass PostSync Smoke Test

The Skip hook is used to temporarily bypass a hook without removing it. This is
useful when you need to redeploy quickly — the PostSync smoke test already ran
on the last sync and you know the backend is healthy. Running it again wastes
time.

### 8a: Add Skip annotation to the PostSync hook

Edit `demo-10/hooks/postsync-smoke-test.yaml` — add `Skip` to the hook type:

```yaml
annotations:
  argocd.argoproj.io/hook: PostSync,Skip    # ← Skip added
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

Push:
```bash
git add demo-10/hooks/postsync-smoke-test.yaml
git commit -m "chore: add Skip to PostSync hook for debug cycle"
git push origin main
```

Sync:
```bash
argocd app sync sync-phase-hook-demo-goals-app
```

Watch:
```bash
kubectl get jobs -n goals-app -w
```

Expected — only PreSync fires, PostSync is skipped entirely:
```
presync-db-init    Running    0/1    3s
presync-db-init    Complete   1/1    15s
# PostSync hook does NOT appear — it was skipped
```

Verify in ArgoCD UI — the `postsync-smoke-test` job does not appear in the
resource tree during this sync.

> **Skip is not permanent removal.** The hook YAML stays in the repo. When you
> remove the `Skip` annotation and push again, the hook resumes running on the
> next sync. This is the intended pattern — a temporary bypass, fully audited
> in Git history.

### 8b: Restore the PostSync hook

```yaml
annotations:
  argocd.argoproj.io/hook: PostSync    # ← Skip removed
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

```bash
git add demo-10/hooks/postsync-smoke-test.yaml
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

# Goals app accessible
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
# open http://localhost:3000

# Secret exists
kubectl get secret mongodb-secret -n goals-app

# Repo registered
argocd repo list
```

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
Hooks are annotated Kubernetes Jobs that run at a specific phase. The phase
exists whether or not you define a hook for it.

**Hook failure stops the lifecycle**
A failed PreSync hook prevents the Sync phase from starting. Nothing gets
deployed until the hook succeeds. This is the protection mechanism — your DB
migration must succeed before your application pods start.

**PostSync and SyncFail are mutually exclusive**
PostSync runs only if Sync succeeds. SyncFail runs only if Sync fails. They
never both run in the same sync cycle.

**Delete policies keep the cluster clean**
`BeforeHookCreation,HookSucceeded` is the standard pattern:
- Deletes the previous run before creating a new one
- Deletes the job after success
- Leaves the job alive on failure so you can inspect logs

**Skip is a temporary bypass, not a deletion**
Adding `Skip` to a hook annotation makes ArgoCD ignore that hook for the current
sync. The hook YAML stays in Git. Remove `Skip` to restore it. Always audited
in Git history.

**Jobs are the most common hook implementation**
Jobs run to completion — they are naturally suited to one-time tasks like
migrations, smoke tests, and notifications. Any Kubernetes resource can be a
hook, but Jobs are the idiomatic choice.

---

## Commands Reference

```bash
# Apply Application CRD
kubectl apply -f demo-10/goals-app.yaml

# Manual sync
argocd app sync sync-phase-hook-demo-goals-app

# Watch hook jobs in real time
kubectl get jobs -n goals-app -w

# Get hook job logs (run quickly — job is deleted on success)
kubectl logs -l job-name=presync-db-init -n goals-app
kubectl logs -l job-name=postsync-smoke-test -n goals-app
kubectl logs -l job-name=syncfail-notify -n goals-app

# Check app status
argocd app get sync-phase-hook-demo-goals-app

# See sync history
argocd app history sync-phase-hook-demo-goals-app

# Access Goals App
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
kubectl port-forward svc/goals-backend-svc -n goals-app 8080:80

# Test backend API directly
curl http://localhost:8080/goals
```

---

## Lessons Learned

**1. Synced does not mean healthy — understand both status indicators**
A sync succeeds when manifests are applied. A deployment with a wrong image tag
syncs successfully but the pods are in `ImagePullBackOff`. Always check both
Sync Status and Health Status after a deployment.

**2. PreSync hooks protect you from deploying against an incompatible schema**
If your DB migration fails, nothing gets deployed. This is the primary production
value of PreSync hooks — preventing application code from starting against a
database that is not ready for it.

**3. Hook delete policy design — leave failed jobs for inspection**
`BeforeHookCreation,HookSucceeded` deletes on success but keeps the job on
failure. This is intentional — when a hook fails you need its logs to diagnose
the problem. Never use `HookFailed` in the delete policy if you want to inspect
failures.

**4. Skip is an audit-friendly temporary bypass**
Disabling a hook by editing the annotation and pushing a commit is better than
deleting the hook YAML. The Git history shows exactly when the hook was bypassed,
by whom, and the commit message explains why. This is production-grade change
management.

**5. SyncFail hooks require a valid sync target to register**
The SyncFail hook must be present in the repo and applied at least once before
it can fire. If you add the SyncFail hook and break the sync in the same commit,
the hook may not fire on that first broken sync because it was never registered.
Add the SyncFail hook in a clean sync before introducing the failure.

---

## What's Next

**Demo-11: App-of-Apps Pattern**
Manage multiple ArgoCD Applications declaratively using a parent Application
that deploys child Applications. Eliminate manual `kubectl apply` for each
Application CRD — let ArgoCD manage them from Git automatically. Build on the
sync hooks foundation by adding sync waves to control ordering across multiple
applications.