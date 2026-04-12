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
- How Multiple PreSync Hooks can be executed in a order using Sync Waves

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
- ✅ Completed **Docker Demo-14** (Goals App — Production-Ready Build) in the
  Docker course — `rselvantech/goals-frontend:v1.0.0` and
  `rselvantech/goals-backend:v1.0.0` pushed to Docker Hub
- ✅ Completed **`README-goals-app-setup.md`** in this folder:
  - `gitops-apps-config` private repo created on GitHub
  - `gitops-apps-config` registered with ArgoCD (Status: Successful)
  - All base manifests pushed to `demo-10-sync-hooks-goals-app/`
  - End-to-end browser test passed and environment torn down
- ✅ ArgoCD running on minikube (default profile)
- ✅ ArgoCD CLI installed and logged in
- ✅ `kubectl` available in terminal
- ✅ GitHub PAT with access to `gitops-apps-config` and `argocd-config`

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

**Expected:** Both pulled successfully or `Already up to date`.

If either image is missing, complete Docker Demo-14 before proceeding.

### 5. `gitops-apps-config` is registered with ArgoCD
```bash
argocd repo list | grep gitops-apps-config
```

**Expected:**
```text
git   gitops-apps-config   https://github.com/rselvantech/gitops-apps-config.git   Successful
```

If not listed, complete `README-goals-app-setup.md` before proceeding.

### 6. `goals-app` namespace does not exist (clean state)
```bash
kubectl get namespace goals-app 2>&1
```

**Expected:**
```text
Error from server (NotFound): namespaces "goals-app" not found
```

If it exists from a previous run, delete it:
```bash
kubectl delete namespace goals-app
```

---

## The Goals App — What It Does

The Goals App is a three-tier goal tracking web application from the Docker course.
Users type goals into the React frontend, which calls a Node.js/Express backend
REST API, which persists goals to a MongoDB database.

```
Browser (on your laptop)
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
goals-backend pod  (Node.js / Express, port 80)
    │  mongodb://...@mongodb:27017/course-goals?authSource=admin
    ▼
mongodb pod  (mongo:6.0, port 27017)
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
    argocd.argoproj.io/hook: PreSync
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

> **Ordering multiple hooks in the same phase:**
> If you have two PreSync hooks where one depends on the other completing
> first, use sync wave annotations on the hook Jobs themselves. ArgoCD runs
> wave 1 hook first, waits for completion, then starts wave 2 hook — within
> the same PreSync phase. This is demonstrated in Step 8 of this demo.

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


> **When to omit `HookSucceeded`:** If you want to inspect the hook job's
> logs after a successful run — for debugging or auditing — omit
> `HookSucceeded` from the delete policy. The job stays in the cluster
> after success and you can run `kubectl logs` at any time. Clean it up
> manually with `kubectl delete job <name> -n <namespace>` when done.
> Only `BeforeHookCreation` ensures a fresh job is created on the next
> sync without name conflicts.

---


### Hooks Only Fire for Their Own Application

A Kubernetes Job with hook annotations that exists in the cluster but is **not**
defined in the application's source repository will NOT be executed during that
application's sync.

ArgoCD only runs hooks that are part of the application's source manifests —
not all jobs with hook annotations that happen to exist in the cluster.
```
# Job created externally with hook annotations — NOT run during app sync
kubectl apply -f external-job.yaml    # has PreSync annotation

# Trigger sync of the application
argocd app sync goals-app-demo
# → only runs hooks defined in goals-app-config source
# → external-job.yaml is NOT executed
```

This is a common source of confusion — hook annotations on externally created
jobs are ignored by ArgoCD during sync operations for other applications.


---

## Folder Structure

```
10-sync-phases-and-hooks/src/
├── gitops-apps-config/      ← git init → remote: rselvantech/gitops-apps-config
│   └── demo-10-sync-hooks-goals-app/
│       ├── namespace.yaml               ← from README-goals-app-setup.md
│       ├── mongodb-pvc.yaml             ← from README-goals-app-setup.md
│       ├── mongodb-deployment.yaml      ← from README-goals-app-setup.md
│       ├── mongodb-service.yaml         ← from README-goals-app-setup.md
│       ├── sealed-mongodb-secret.yaml   ← from README-goals-app-setup.md
│       ├── backend-deployment.yaml      ← from README-goals-app-setup.md
│       ├── backend-service.yaml         ← from README-goals-app-setup.md
│       ├── frontend-deployment.yaml     ← from README-goals-app-setup.md
│       ├── frontend-service.yaml        ← from README-goals-app-setup.md
│       ├── presync-db-init.yaml         ← added in this demo
│       ├── presync-seed-data.yaml       ← added in Step 8 of this demo
│       ├── postsync-smoke-test.yaml     ← added in this demo
│       └── syncfail-notify.yaml         ← added in this demo
└── argocd-config/           ← git init → remote: rselvantech/argocd-config
    └── demo-10-sync-hooks/
        └── goals-app-demo.yaml
```

> The base manifests in `demo-10-sync-hooks-goals-app/` already
> exist from `README-goals-app-setup.md`. This demo only adds the fe files mentioned above


> Only `gitops-apps-config` needs to be registered with ArgoCD — already done
> in `README-goals-app-setup.md`. `argocd-config` is applied manually with
> `kubectl apply`.

---

## Step 1: Verify the Goals App Project Files 

All these files are alreay generated as part of **`README-goals-app-setup.md`**

**Verify the base manifests are already present:**
```bash
cd  10-sync-phases-and-hooks/src/
ls gitops-apps-config/demo-10-sync-hooks-goals-app/
```

**Expected:**
```text
backend-deployment.yaml   frontend-deployment.yaml  mongodb-pvc.yaml
backend-service.yaml      frontend-service.yaml     mongodb-service.yaml
namespace.yaml            mongodb-deployment.yaml   
sealed-mongodb-secret.yaml
```

**Verify Application CDR present:**
```bash
ls argocd-config/demo-10-sync-hooks/
```

**Expected:**
```text
goals-app-demo.yaml
```


## Step 2: Apply Application CRD & Verify Goals App

**Apply & Sync:**
```bash
kubectl apply -f argocd-config/demo-10-sync-hooks/goals-app-demo.yaml

argocd app sync goals-app-demo
argocd app get goals-app-demo
```

**Expected:**
```text
Sync Status:   Synced
Health Status: Healthy
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

**Verify the controller decrypted it after sync:**
```bash
kubectl get sealedsecret mongodb-secret -n goals-app
```

**Expected:**
```text
NAME             STATUS   SYNCED   AGE
mongodb-secret             True     10s
```

**Test - Goals App:**
```bash
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000 &
```

Open `http://localhost:3000` in your browser.

**Test 1 — UI loads:**
The Goals Tracker page appears with an empty list. ✅

**Test 2 — Add a goal:**
Type `"Goals App Demo test"` → click **Add Goal**.
Goal appears in the list. ✅

**Test 5 — Reload page (persistence):**
Refresh the browser — goal still appears (loaded from MongoDB). ✅

**Test 6 — Delete the goal:**
Click the goal in the list — it disappears. ✅


---

## Step 3: Add PreSync Hook — DB Initialisation

### Why this hook — and why it matters even though the app already works

The Goals App is running correctly  without this hook. So why add it?

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
### Do These Hooks Add Real Value? — An Honest Answer

**The DB init hook (presync-db-init):**

For this specific Goals App in this demo — the collection and text index are
not strictly required for the app to function. MongoDB creates the `goals`
collection automatically when the first document is inserted. The app works
without this hook.

The hook's real value is in the **pattern it teaches**, not in this app's
specific need. In production, this hook would run actual schema migrations —
adding required fields, creating compound indexes, renaming collections — that
the new version of the backend absolutely requires before it starts. Without the
PreSync hook, the new backend pod starts against an incompatible database and
crashes.

Use this hook to learn the pattern. In your real applications, this is where
database migrations live.

**The seed data hook (presync-seed-data, Step 8):**

This hook adds clear, observable value — you see the pre-seeded goal in the
browser immediately after sync without adding anything manually. It demonstrates
wave ordering with a concrete visible result. In production it would seed
reference data that the application requires to function (lookup tables, default
configuration, initial admin users).

---

### What Is a Text Index — and Why Does It Matter?

A MongoDB text index (`{ text: 'text' }`) enables **full-text search** on the
`text` field. Without it, you can only find documents by exact value or range.
With it, MongoDB can search for words and phrases across the field's content.

**Without a text index:**
```javascript
// This fails — text index required
db.goals.find({ $text: { $search: "docker kubernetes" } })
// Error: text index required for $text query
```

**With a text index:**
```javascript
// This works — finds any goal containing "docker" or "kubernetes"
db.goals.find({ $text: { $search: "docker kubernetes" } })
// Returns: goals with "Learn docker" and "Kubernetes basics"
```

**In this demo:** The backend currently only calls `Goal.find()` (return all
goals) and `Goal.deleteOne()` (delete by ID). It never uses `$text` search,
so the index has no functional effect right now. The PreSync hook creates it
pre-emptively — if a future version of the backend adds a search endpoint,
the index is already there.

**Why create it in PreSync rather than letting the backend create it?**

The backend could create the index on startup using Mongoose's schema
definition (`{ text: String, index: 'text' }`). The problem is timing — if
the backend tries to create the index while MongoDB is under load or the
collection already has millions of documents, index creation blocks all
operations and can take minutes. A PreSync hook runs before any traffic hits
the database, in a controlled window when the cluster is in a known state.

---

### What this hook does — command by command

The hook is a Kubernetes **Job** running the `mongo:6.0` image. It executes
a shell script that does two things:

**1. Wait for MongoDB to be ready:**
```bash
until mongosh \
  --host "$MONGODB_HOST" \
  --username "$MONGODB_USERNAME" \
  --password "$MONGODB_PASSWORD" \
  --authenticationDatabase admin \
  --eval "db.adminCommand('ping')" > /dev/null 2>&1; do
  echo "MongoDB not ready yet, retrying in 3s..."
  sleep 3
done
```
- `mongosh` connects to MongoDB using the credentials from the `mongodb-secret`
- Host name name is stored in the env varaible `MONGODB_HOST`
- `db.adminCommand('ping')` is the standard MongoDB health check — returns
  `{ok: 1}` when MongoDB is accepting connections
- `> /dev/null 2>&1` suppresses output — we only care about the exit code
- The `until` loop retries every 3 seconds until the ping succeeds
- On the first sync MongoDB is being deployed in parallel during the Sync phase
  while this hook is in the PreSync phase — the retry loop waits for it

**2. Create the collection and index:**
```bash
mongosh \
  --host "$MONGODB_HOST" \
  --username "$MONGODB_USERNAME" \
  --password "$MONGODB_PASSWORD" \
  --authenticationDatabase admin \
  "$MONGODB_DATABASE" \
  --eval "
    db.createCollection('goals');
    db.goals.createIndex({ text: 'text' });
    print('Collection and text index created successfully.');
  "
echo "=== PreSync: DB initialisation complete ==="
```
-  database name is stored in the env varaible `MONGODB_DATABASE`
- `createCollection` creates the `goals` collection if it does not exist
- `createIndex({ text: 'text' })` adds a full-text search index on the `text`
  field — this is what the backend queries when listing goals
- Running this again on a subsequent sync is safe — MongoDB ignores
  `createCollection` and `createIndex` if they already exist

**Create the presync hook job:**
```bash
cd 10-sync-phases-and-hooks/src/gitops-apps-config
mkdir -p demo-10-sync-hooks-goals-app

touch demo-10-sync-hooks-goals-app/presync-db-init.yaml
```

**Create `demo-10-sync-hooks-goals-app/presync-db-init.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: presync-db-init
  namespace: goals-app
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
#    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
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
              echo "MongoDB host:     $MONGODB_HOST"
              echo "MongoDB database: $MONGODB_DATABASE"
              echo "Waiting for MongoDB to be ready..."
              until mongosh \
                --host "$MONGODB_HOST" \
                --username "$MONGODB_USERNAME" \
                --password "$MONGODB_PASSWORD" \
                --authenticationDatabase admin \
                --eval "db.adminCommand('ping')" > /dev/null 2>&1; do
                echo "MongoDB not ready yet, retrying in 3s..."
                sleep 3
              done
              echo "MongoDB is ready."
              echo "Initialising database: $MONGODB_DATABASE"
              mongosh \
                --host "$MONGODB_HOST" \
                --username "$MONGODB_USERNAME" \
                --password "$MONGODB_PASSWORD" \
                --authenticationDatabase admin \
                "$MONGODB_DATABASE" \
                --eval "
                  db.createCollection('goals');
                  db.goals.createIndex({ text: 'text' });
                  print('Collection and text index created successfully.');
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
            - name: MONGODB_HOST
              value: "mongodb"
            - name: MONGODB_DATABASE
              value: "course-goals"
```

> **`MONGODB_HOST` and `MONGODB_DATABASE`** match the values set in
> `backend-deployment.yaml`. Both the backend and the PreSync hook connect to
> the same host and database — consistency is enforced by using the same values.
> If you ever change the MongoDB Service name or database name, update both the
> backend deployment and the hook.
>
> **Why plain `value:` and not `secretKeyRef`?** `MONGODB_HOST` and
> `MONGODB_DATABASE` are not sensitive — they are configuration, not credentials.
> Only `MONGODB_USERNAME` and `MONGODB_PASSWORD` are secret values that must
> come from `mongodb-secret`.

**Push:**
```bash
git add demo-10-sync-hooks-goals-app/presync-db-init.yaml
git commit -m "feat: add PreSync DB initialisation hook"
git push origin main
```

### Sync and observe PreSync hook

**Open a second terminal and watch jobs:**
```bash
kubectl get jobs -n goals-app -w
```

**In the first terminal, Sync:**
```bash
argocd app sync goals-app-demo
```

**Expected sequence in the watch terminal:**
```text
NAME              STATUS              COMPLETIONS   DURATION   AGE
presync-db-init   Running              0/1                      0s
presync-db-init   Running              0/1           0s         0s
presync-db-init   Running              0/1           3s         3s
presync-db-init   Running              0/1           6s         6s
presync-db-init   SuccessCriteriaMet   0/1           7s         7s
presync-db-init   Complete             1/1           7s         7s
presync-db-init   Complete             1/1           7s         7s
# job will not be deleted after completion (since no HookSucceeded policy). it will be deleted only before next Creation
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
MongoDB host:     mongodb
MongoDB database: course-goals
Waiting for MongoDB to be ready...
MongoDB is ready.
Initialising database: course-goals
MongoServerError: Collection course-goals.goals already exists.
=== PreSync: DB initialisation complete ===
```

**Verify all pods still running:**
```bash
kubectl get pods -n goals-app
```

**Expected:**
```text
NAME                             READY   STATUS      RESTARTS   AGE
goals-backend-5fff555549-t4h5t   1/1     Running     0          44m
goals-frontend-dbdf87ff7-spkl2   1/1     Running     0          44m
mongodb-778c6d5b4b-vx8gd         1/1     Running     0          44m
presync-db-init-rsfc5            0/1     Completed   0          43s
```

> The PreSync hook ran before the Sync phase. The Deployment manifests were
> already applied in Step 3 — this sync re-applied them unchanged while the
> hook ran first. In ArgoCD UI, click on `goals-app-demo` and
> you will briefly see the `presync-db-init` job appear in the resource tree
> before being deleted by the delete policy.

---

## Step 4: Access the Goals App

**Port-forward the frontend service — one port-forward is enough:**
```bash
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000
```

Open `http://localhost:3000` in your browser.

**Try the full workflow:**
1. Type a goal: `"Learn ArgoCD sync hooks"` ✅
2. Click **Add Goal** — goal appears in the list ✅
3. Reload the page — goal persists (stored in MongoDB via the initialised collection) ✅
4. Click the goal to delete it — goal disappears ✅

**Verify API calls went through nginx:**
```bash
kubectl logs -l app=goals-frontend -n goals-app | grep goals
```

**Expected — nginx proxied the requests:**
```text
172.x.x.x - "GET /goals HTTP/1.1" 200
172.x.x.x - "POST /goals HTTP/1.1" 201
```

**Verify directly via backend API (optional):**
```bash
kubectl port-forward svc/goals-backend-svc -n goals-app 8090:80 &
curl http://localhost:8090/goals
```

**Expected:**
```json
{"goals": [{"id": "...", "text": "Learn ArgoCD sync hooks"}]}
#Also shows any existing goals
```

---

## Step 5: Add PostSync Hook — Smoke Test

After all manifests are applied successfully, the PostSync hook verifies the
backend API is reachable by curling it from inside the cluster. This runs
**after** the Sync phase completes — confirming the full stack is healthy
before marking the deployment done.

**Create the postsync hook job:**
```bash
cd 10-sync-phases-and-hooks/src/gitops-apps-config

touch demo-10-sync-hooks-goals-app/postsync-smoke-test.yaml
```

**Create `demo-10-sync-hooks-goals-app/postsync-smoke-test.yaml`:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postsync-smoke-test
  namespace: goals-app
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
#    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
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
              echo "Testing backend: http://$BACKEND_SVC/goals"
              until curl -sf "http://$BACKEND_SVC/goals" -o /dev/null; do
                echo "Backend not ready yet, retrying in 3s..."
                sleep 3
              done
              echo "Backend responded successfully."
              echo "=== PostSync: Smoke test PASSED ==="
          env:
            - name: BACKEND_SVC
              value: "goals-backend-svc.goals-app.svc.cluster.local"
```

> **Why use FQDN for `BACKEND_SVC`?** The smoke test Job runs inside the
> cluster. Using the FQDN (`goals-backend-svc.goals-app.svc.cluster.local`)
> is explicit and portable — it works regardless of the Job's namespace. If
> the Job were in a different namespace, the short name `goals-backend-svc`
> would not resolve without the correct search domain. The FQDN always works.

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
git add demo-10-sync-hooks-goals-app/postsync-smoke-test.yaml
git commit -m "feat: add PostSync smoke test hook"
git push origin main
```

**Watch the full lifecycle in the second terminal:**
```bash
kubectl get jobs -n goals-app -w
```

**Sync:**
```bash
argocd app sync goals-app-demo
```


**Expected sequence in the watch terminal:**
```text
NAME              STATUS                  COMPLETIONS   DURATION   AGE
presync-db-init   Complete                 1/1           7s         25m        ← PreSync job from previous run
# Above job will be deleted before creating next PreSync job (hook-delete-policy: BeforeHookCreation )
presync-db-init   Running                  0/1           3s         3s         ← New PreSync job created, PreSync begins
presync-db-init   Running                  0/1           4s         4s
presync-db-init   SuccessCriteriaMet       0/1           5s         5s
presync-db-init   Complete                 1/1           5s         5s
presync-db-init   Complete                 1/1           5s         5s         ← PreSync done, Sync begins
postsync-smoke-test   Running              0/1                      0s         ← PostSync begins after Sync
postsync-smoke-test   Running              0/1           0s         0s
postsync-smoke-test   SuccessCriteriaMet   0/1           7s         7s
postsync-smoke-test   Complete             1/1           7s         7s
postsync-smoke-test   Complete             1/1           7s         7s         ← PreSync done, smoke test passed
# Both the jobs will not be deleted after completion (since no HookSucceeded policy)
```

The full sync lifecycle — PreSync → Sync → PostSync — completed with both hooks
running at the correct phases.



**Inspect PreSync job logs** — run quickly before the job is deleted:
```bash
kubectl logs -l job-name=postsync-smoke-test -n goals-app
```

**Expected:**
```text
=== PostSync: Starting smoke test ===
Testing backend: http://goals-backend-svc.goals-app.svc.cluster.local/goals
Backend responded successfully.
=== PostSync: Smoke test PASSED ===
```

**Verify all pods still running:**
```bash
kubectl get pods -n goals-app
```

**Expected:**
```text
NAME                             READY   STATUS      RESTARTS   AGE
goals-backend-5fff555549-t4h5t   1/1     Running     0          72m
goals-frontend-dbdf87ff7-spkl2   1/1     Running     0          72m
mongodb-778c6d5b4b-vx8gd         1/1     Running     0          72m
postsync-smoke-test-mqx7w        0/1     Completed   0          3m30s
presync-db-init-flm52            0/1     Completed   0          3m38s
```

After the PreSync job completes, the Sync phase begins and all application
manifests are applied.

---

## Step 6: Trigger SyncFail Hook

### 6a: Add the SyncFail hook and sync cleanly first — before breaking anything

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

**Create the syncfail hook job:**
```bash
cd 10-sync-phases-and-hooks/src/gitops-apps-config

touch demo-10-sync-hooks-goals-app/syncfail-notify.yaml
```


**Create `demo-10-sync-hooks-goals-app/syncfail-notify.yaml`:**

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
              echo "Application: goals-app-demo"
              echo "Namespace:   goals-app"
              echo "Time:        $(date)"
              echo "Action:      Alerting on-call team (simulated)"
              echo "In production: replace with curl webhook to Slack/PagerDuty"
              echo "=== SyncFail: Notification sent ==="
```

**Push and sync cleanly to register the hook:**
```bash
git add demo-10-sync-hooks-goals-app/syncfail-notify.yaml
git commit -m "feat: add SyncFail notification hook"
git push origin main

argocd app sync goals-app-demo
```

**Verify sync succeeded (SyncFail did not fire — correct):**
```bash
argocd app get goals-app-demo | grep -E "Sync|Health"
```

**Expected:**
```text
Sync Status:   Synced
Health Status: Healthy
```
The SyncFail hook is now registered but did not fire because the sync succeeded.

### 6b: Introduce a bad manifest to break sync

Now that the SyncFail hook is registered, deliberately break the sync by adding
an invalid manifest — a Deployment with a missing required field (`spec.selector`).
This causes `kubectl apply` to fail, which triggers the SyncFail phase.

**Create a bad manifest :**
```bash
cd 10-sync-phases-and-hooks/src/gitops-apps-config

touch demo-10-sync-hooks-goals-app/bad-manifest.yaml
```

**Create `demo-10-sync-hooks-goals-app/bad-manifest.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deployment
  namespace: goals-app
spec:
  replicas: 1
  # spec.selector is intentionally missing — required field — kubectl apply will fail
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
git add demo-10-sync-hooks-goals-app/bad-manifest.yaml
git commit -m "test: add invalid manifest to trigger SyncFail hook"
git push origin main
```

**Watch — `syncfail-notify` fires:**
```bash
kubectl get jobs -n goals-app -w
```

**Sync:**
```bash
argocd app sync goals-app-demo
```


**Expected sequence in the watch terminal:**
```text
NAME                  STATUS              COMPLETIONS   DURATION   AGE
postsync-smoke-test   Complete             1/1           7s         49m      ← PostSync job from previous run
presync-db-init       Complete             1/1           5s         52m      ← PreSync job from previous run
presync-db-init       Running              0/1                      0s       ← New PreSync job created, PreSync begins
presync-db-init       Running              0/1           4s         4s
presync-db-init       SuccessCriteriaMet   0/1           5s         5s
presync-db-init       Complete             1/1           5s         5s
presync-db-init       Complete             1/1           5s         5s       ← PreSync done, Sync begins
syncfail-notify       Running              0/1                      0s       ← SyncFail begins after unsuccessful Sync
syncfail-notify       Running              0/1           0s         0s
syncfail-notify       SuccessCriteriaMet   0/1           6s         6s
syncfail-notify       Complete             1/1           6s         6s
syncfail-notify       Complete             1/1           6s         6s      ← SyncFail done

# Both the jobs will not be deleted after completion (since no HookSucceeded policy) -check
```

**Check the SyncFail job logs:**
```bash
kubectl logs -l job-name=syncfail-notify -n goals-app
```

**Expected:**
```text
=== SyncFail: Sync operation FAILED ===
Application: goals-app-demo
Namespace:   goals-app
Time:        Sat Apr 11 19:05:27 UTC 2026
Action:      Alerting on-call team (simulated)
In production: replace with curl webhook to Slack/PagerDuty
=== SyncFail: Notification sent ===
```

**Verify the sync error in ArgoCD:**
```bash
argocd app get goals-app-demo
```

**Expected:**
```text
Sync Status:        OutOfSync from main (3cdb7bf)
Health Status:      Missing

GROUP        KIND                   NAMESPACE  NAME                 STATUS     HEALTH   HOOK      MESSAGE
             Namespace              goals-app  goals-app            Succeeded  Synced             namespace/goals-app unchanged
batch        Job                    goals-app  presync-db-init      Succeeded           PreSync   Reached expected number of succeeded pods
             PersistentVolumeClaim  goals-app  goals-db-data        Synced     Healthy            persistentvolumeclaim/goals-db-data unchanged
             Service                goals-app  goals-backend-svc    Synced     Healthy            service/goals-backend-svc unchanged
             Service                goals-app  mongodb              Synced     Healthy            service/mongodb unchanged
             Service                goals-app  goals-frontend-svc   Synced     Healthy            service/goals-frontend-svc unchanged
apps         Deployment             goals-app  goals-frontend       Synced     Healthy            deployment.apps/goals-frontend unchanged
apps         Deployment             goals-app  mongodb              Synced     Healthy            deployment.apps/mongodb unchanged
apps         Deployment             goals-app  goals-backend        Synced     Healthy            deployment.apps/goals-backend unchanged
apps         Deployment             goals-app  broken-deployment    OutOfSync  Missing            Deployment.apps "broken-deployment" is invalid: [spec.selector: Required value, spec.template.metadata.labels: Invalid value: {"app":"broken"}: `selector` does not match template `labels`]
bitnami.com  SealedSecret           goals-app  mongodb-secret       Synced     Healthy            sealedsecret.bitnami.com/mongodb-secret unchanged
batch        Job                    goals-app  syncfail-notify      Succeeded           SyncFail  Reached expected number of succeeded pods
             Namespace                         goals-app            Synced                        
batch        Job                    goals-app  postsync-smoke-test             Healthy
```

**Key Observations (from the output):**

The existing Goals App pods are still running and healthy.The bad manifest caused the Sync phase to fail but the previously applied Deployments (mongodb, oals-backend,goals-frontend) were not affected — ArgoCD only failed on the new broken manifest.

- **Execution flow:**
  - `presync-db-init` → Sync attempted → failure → `syncfail-notify`
  - `postsync-smoke-test` did NOT run in this cycle

- **PreSync job:**
  - `presync-db-init` shows `Succeeded` with `HOOK: PreSync`
  - Confirms PreSync completed successfully before Sync started

- **Sync failure (root cause):**
  - `broken-deployment` is `OutOfSync` and `Missing`
  - Explicit validation error:
    ```
    spec.selector: Required value
    selector does not match template labels
    ```
  - This single invalid manifest caused the entire Sync to fail

- **Existing resources:**
  - All other resources (Deployments, Services, PVC, SealedSecret)
    remain `Synced` and `Healthy`
  - No impact from the failed manifest

- **SyncFail job:**
  - `syncfail-notify` shows `Succeeded` with `HOOK: SyncFail`
  - Confirms it executed after Sync failure

- **PostSync job:**
  - `postsync-smoke-test` has no `HOOK` execution in this run
  - Only appears as an existing resource from a previous successful sync

### 6c: Fix the bad manifest

```bash
git rm demo-10-sync-hooks-goals-app/bad-manifest.yaml
git commit -m "fix: remove invalid manifest"
git push origin main
```

**Sync again — full lifecycle completes cleanly:**
```bash
argocd app sync goals-app-demo

argocd app get goals-app-demo
```

**Expected:**
```
Sync Policy:        Manual
Sync Status:        Synced to main (e1f0447)
Health Status:      Healthy

GROUP        KIND                   NAMESPACE  NAME                 STATUS     HEALTH   HOOK      MESSAGE
             Namespace              goals-app  goals-app            Succeeded  Synced             namespace/goals-app unchanged
batch        Job                    goals-app  presync-db-init      Succeeded           PreSync   Reached expected number of succeeded pods
             PersistentVolumeClaim  goals-app  goals-db-data        Synced     Healthy            persistentvolumeclaim/goals-db-data unchanged
             Service                goals-app  goals-frontend-svc   Synced     Healthy            service/goals-frontend-svc unchanged
             Service                goals-app  goals-backend-svc    Synced     Healthy            service/goals-backend-svc unchanged
             Service                goals-app  mongodb              Synced     Healthy            service/mongodb unchanged
apps         Deployment             goals-app  goals-backend        Synced     Healthy            deployment.apps/goals-backend unchanged
apps         Deployment             goals-app  mongodb              Synced     Healthy            deployment.apps/mongodb unchanged
apps         Deployment             goals-app  goals-frontend       Synced     Healthy            deployment.apps/goals-frontend unchanged
bitnami.com  SealedSecret           goals-app  mongodb-secret       Synced     Healthy            sealedsecret.bitnami.com/mongodb-secret unchanged
batch        Job                    goals-app  postsync-smoke-test  Succeeded           PostSync  Reached expected number of succeeded pods
             Namespace                         goals-app            Synced                        

```

**Key Observations (from the output):**

Full lifecycle — PreSync → Sync → PostSync — all complete successfully.

- **Execution flow:**
  - `presync-db-init` → Sync succeeded → `postsync-smoke-test`
  - No `SyncFail` hook executed

- **Overall status:**
  - `Sync Status: Synced`
  - `Health Status: Healthy`
  - Indicates complete successful reconciliation

- **PreSync job:**
  - `presync-db-init` shows `Succeeded` with `HOOK: PreSync`
  - Confirms PreSync executed before Sync

- **Sync phase:**
  - All resources are `Synced` and `Healthy`
  - No validation or apply errors

- **PostSync job:**
  - `postsync-smoke-test` shows `Succeeded` with `HOOK: PostSync`
  - Confirms it executed after successful Sync

- **SyncFail job:**
  - Not present in this output
  - Confirms no failure occurred in this cycle


---

## Step 7: Skip Hook — Bypass PostSync Smoke Test

The Skip hook is used to temporarily bypass a hook without removing it. This is
useful when you need to redeploy quickly — the PostSync smoke test already ran
on the last sync and you know the backend is healthy. Running it again adds time
and noise without adding value.

**Why not just delete the hook YAML?** Deleting the YAML removes the hook
permanently and the change is not obviously reversible. Adding `Skip` to the
annotation is a deliberate, auditable, temporary bypass — the Git history shows
exactly when it was skipped, by whom, and the commit message explains why.

### 7a: Add Skip annotation to the PostSync hook

Edit `demo-10-sync-hooks-goals-app/postsync-smoke-test.yaml` — add `Skip`:

```yaml
annotations:
  argocd.argoproj.io/hook: Skip    # ← Skip added
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```
**Push:**
```bash
git add demo-10-sync-hooks-goals-app/postsync-smoke-test.yaml
git commit -m "chore: add Skip to PostSync hook for debug cycle"
git push origin main
```

**Watch — only PreSync fires, PostSync skipped:**
```bash
kubectl get jobs -n goals-app -w
```

**Sync:**
```bash
argocd app sync goals-app-demo
```


**Expected sequence in the watch terminal:**
```text
NAME                  STATUS     COMPLETIONS   DURATION   AGE
presync-db-init       Complete   1/1           5s         2m15s
presync-db-init       Complete   1/1           5s         2m23s
presync-db-init       Complete   1/1           5s         2m23s
presync-db-init       Running    0/1                      0s
presync-db-init       Running    0/1           0s         0s
presync-db-init       Running    0/1           2s         2s
presync-db-init       Running    0/1           5s         5s
presync-db-init       SuccessCriteriaMet   0/1           6s         6s
presync-db-init       Complete             1/1           6s         6s
presync-db-init       Complete             1/1           6s         6s
# postsync-smoke-test does NOT appear — skipped
```


**Get App**
```bash
argocd app get goals-app-demo
```

**Expected:**
```
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to main (46a7665)
Health Status:      Healthy

GROUP        KIND                   NAMESPACE  NAME                 STATUS     HEALTH   HOOK     MESSAGE
             Namespace              goals-app  goals-app            Succeeded  Synced            namespace/goals-app unchanged
batch        Job                    goals-app  presync-db-init      Succeeded           PreSync  Reached expected number of succeeded pods
             PersistentVolumeClaim  goals-app  goals-db-data        Synced     Healthy           persistentvolumeclaim/goals-db-data unchanged
             Service                goals-app  mongodb              Synced     Healthy           service/mongodb unchanged
             Service                goals-app  goals-frontend-svc   Synced     Healthy           service/goals-frontend-svc unchanged
             Service                goals-app  goals-backend-svc    Synced     Healthy           service/goals-backend-svc unchanged
apps         Deployment             goals-app  goals-backend        Synced     Healthy           deployment.apps/goals-backend unchanged
apps         Deployment             goals-app  mongodb              Synced     Healthy           deployment.apps/mongodb unchanged
apps         Deployment             goals-app  goals-frontend       Synced     Healthy           deployment.apps/goals-frontend unchanged
bitnami.com  SealedSecret           goals-app  mongodb-secret       Synced     Healthy           sealedsecret.bitnami.com/mongodb-secret unchanged
             Namespace                         goals-app            Synced                       
batch        Job                    goals-app  postsync-smoke-test             Healthy    
```

**Key Observations (PostSync changed to Skip):**

- **Execution flow:**
  - `presync-db-init` → Sync succeeded
  - PostSync phase is **intentionally skipped**

- **PreSync job:**
  - `presync-db-init` shows `Succeeded` with `HOOK: PreSync`
  - Confirms normal execution before Sync

- **Sync phase:**
  - All resources are `Synced` and `Healthy`
  - No errors

- **PostSync job (`postsync-smoke-test`):**
  - Present in output but:
    - **No `HOOK: PostSync`**
    - No `Succeeded` status
  - Appears only as a regular resource (`Job`, `Healthy`)

- **Key distinction:**
  - When `hook: Skip` is set:
    - ArgoCD does NOT show a “Skipped” state
    - ArgoCD **does not treat it as a hook at all**
    - It is **excluded from hook lifecycle**
    - Resource is rendered like a normal manifest

- **Conclusion:**
  - `Skip` ≠ “skipped with status”
  - `Skip` = “hook disabled / not part of hook execution”


### 7b: Restore the PostSync hook

```yaml
annotations:
  argocd.argoproj.io/hook: PostSync    # ← Skip removed
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

```bash
git add demo-10-sync-hooks-goals-app/postsync-smoke-test.yaml
git commit -m "feat: restore PostSync hook after debug cycle"
git push origin main
```

---

## Step 8: Order Multiple PreSync Hooks with Sync Waves

The Goals App currently has one PreSync hook — the DB initialisation job.
In a real deployment you might need a second PreSync step that depends on
the first completing successfully. For example:

- **Wave 1:** DB schema migration — creates the `goals` collection and index
- **Wave 2:** DB seed data — inserts default data that requires the schema to exist

Without wave ordering, both jobs start simultaneously. The seed job tries to
insert into a collection that does not exist yet and fails.


**`sync-wave` annotation — quick introduction**

`argocd.argoproj.io/sync-wave: "N"` tells ArgoCD to apply this resource in wave N. Lower number runs first. Wave ordering is covered in full in Demo-11 — here we use it only to order two PreSync jobs relative to each other.


**Add a wave annotation to the existing PreSync hook:**

Edit `demo-10-sync-hooks-goals-app/presync-db-init.yaml` — add wave 1:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "1"         # ← runs first
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
```

**Create presync seed data hook job:**
```bash
cd 10-sync-phases-and-hooks/src/gitops-apps-config

touch demo-10-sync-hooks-goals-app/presync-seed-data.yaml
```

**Create a second PreSync hook at wave 2:**

Create `demo-10-sync-hooks-goals-app/presync-seed-data.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: presync-seed-data
  namespace: goals-app
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "2"
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: seed-data
          image: mongo:6.0
          command:
            - /bin/sh
            - -c
            - |
              echo "=== PreSync wave 2: Seeding default data ==="
              echo "Database: $MONGODB_DATABASE"
              mongosh \
                --host "$MONGODB_HOST" \
                --username "$MONGODB_USERNAME" \
                --password "$MONGODB_PASSWORD" \
                --authenticationDatabase admin \
                "$MONGODB_DATABASE" \
                --eval "
                  db.goals.insertOne({
                    text: 'Welcome — your first goal is pre-seeded',
                    _id: ObjectId('000000000000000000000001')
                  });
                  print('Seed data inserted successfully.');
                "
              echo "=== PreSync wave 2: Seeding complete ==="
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

**Push:**
```bash
git add demo-10-sync-hooks-goals-app/
git commit -m "feat: add wave ordering to PreSync hooks — schema before seed"
git push origin main
```

**Watch the wave ordering:**
Open a second terminal and watch:
```bash
kubectl get jobs -n goals-app -w
```

Sync:
```bash
argocd app sync goals-app-demo
```

**Expected — PreSync wave 1 completes before wave 2 starts:**
```text
presync-db-init      Running    0/1    3s    ← wave 1
presync-db-init      Complete   1/1    15s   ← wave 1 done
presync-seed-data    Running    0/1    17s   ← wave 2 starts
presync-seed-data    Complete   1/1    22s   ← wave 2 done
# Both deleted by HookSucceeded policy
# Sync phase begins

NAME                  STATUS            COMPLETIONS   DURATION      AGE
postsync-smoke-test   Complete             1/1           4s         23m
presync-db-init       Complete             1/1           6s         21m
presync-db-init       Complete             1/1           6s         23m
presync-db-init       Complete             1/1           6s         23m
presync-db-init       Running              0/1                      0s
presync-db-init       Running              0/1           0s         0s
presync-db-init       SuccessCriteriaMet   0/1           5s         5s
presync-db-init       Complete             1/1           5s         5s
presync-db-init       Complete             1/1           5s         5s
presync-seed-data     Running              0/1                      0s
presync-seed-data     Running              0/1           0s         0s
presync-seed-data     Running              0/1           3s         3s
presync-seed-data     Running              0/1           4s         4s
presync-seed-data     SuccessCriteriaMet   0/1           5s         5s
presync-seed-data     Complete             1/1           5s         5s
presync-seed-data     Complete             1/1           5s         5s
postsync-smoke-test   Complete             1/1           4s         25m
postsync-smoke-test   Complete             1/1           4s         25m
postsync-smoke-test   Running              0/1                      0s
postsync-smoke-test   Running              0/1           0s         0s
postsync-smoke-test   SuccessCriteriaMet   0/1           3s         3s
postsync-smoke-test   Complete             1/1           3s         3s
postsync-smoke-test   Complete             1/1           3s         4s
presync-seed-data     Complete             1/1           5s         12s
presync-seed-data     Complete             1/1           5s         12s
# presync-seed-data deleted by HookSucceeded policy
```

**Key observation:** `presync-seed-data` did not appear until `presync-db-init`
reached `Complete`. Without the wave annotation, both jobs would have started
at second 3 simultaneously.

**Verify the seeded goal appears in the app:**
```bash
kubectl port-forward svc/goals-backend-svc -n goals-app 8085:80 &
curl http://localhost:8085/goals
```

**Expected:**
```json
{"goals": [{"text": "Welcome — your first goal is pre-seeded", ...}]}
```

The seeded goal exists because the seed job ran after the schema was ready —
guaranteed by wave ordering within the PreSync phase.

> **The combined lifecycle in this demo:**
> ```
> PreSync phase:
>   wave 1 → presync-db-init      (schema migration)
>   wave 2 → presync-seed-data    (seed data — depends on wave 1)
>
> Sync phase:
>   wave 0 → all manifests applied (Deployments, Services etc.)
>
> PostSync phase:
>   wave 0 → postsync-smoke-test   (verifies API is reachable)
> ```
> This is the complete ArgoCD sync lifecycle — phases controlling the
> macro order, waves controlling the micro order within each phase.


## Verify Final State

```bash
# Application synced and healthy
argocd app get goals-app-demo

# All three pods running
kubectl get pods -n goals-app

# No leftover hook jobs or they are in Completed status
kubectl get jobs -n goals-app

# Secret exists
kubectl get secret mongodb-secret -n goals-app

# gitops-apps-config registered
argocd repo list | grep gitops-apps-config
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
kubectl delete app goals-app-demo -n argocd

# Delete the namespace (removes all resources including the secret)
kubectl delete namespace goals-app

# No repo deregistration — gitops-apps-config is used in all future demos
```

**Verify:**
```bash
kubectl get namespace goals-app          # Error: not found
argocd app list | grep goals-app         # No output
argocd repo list | grep gitops-apps-config  # Still listed — correct
```

> **Do NOT deregister `gitops-apps-config`** — it is used in Demo-11, Demo-12,
> Demo-13, Demo-14, and beyond. Leave it registered.

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

**Wave ordering within a phase**
`argocd.argoproj.io/sync-wave: "N"` on hook Jobs controls ordering within a
phase. Lower number runs first. Wave 1 must complete before wave 2 starts.

---

## Commands Reference

```bash
# Bootstrap namespace and secret (first run only)
kubectl create namespace goals-app
kubectl create secret generic mongodb-secret \
  --namespace goals-app \
  --from-literal=MONGODB_USERNAME=rselvantech \
  --from-literal=MONGODB_PASSWORD=passWD \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=rselvantech \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=passWD
# Note: only needed on first run — SealedSecret manages it on subsequent syncs

# Apply Application CRD
kubectl apply -f demo-10-sync-hooks/goals-app.yaml

# Manual sync
argocd app sync goals-app-demo

# Refresh without syncing
argocd app get goals-app-demo --refresh

# Watch hook jobs
kubectl get jobs -n goals-app -w

# Hook job logs
kubectl logs -l job-name=presync-db-init -n goals-app
kubectl logs -l job-name=postsync-smoke-test -n goals-app
kubectl logs -l job-name=syncfail-notify -n goals-app

# App status and sync errors
argocd app get goals-app-demo

# Sync history
argocd app history goals-app-demo

# Access Goals App
kubectl port-forward svc/goals-frontend-svc -n goals-app 3000:3000

# Verify nginx proxy is working
kubectl logs -l app=goals-frontend -n goals-app | grep goals

# Verify backend API directly
kubectl port-forward svc/goals-backend-svc -n goals-app 8080:80
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

**8. Multiple hooks in the same phase have no ordering guarantee without waves**
If you have two PreSync hooks and the second depends on the first completing,
add `argocd.argoproj.io/sync-wave` to both. Without waves, ArgoCD runs all
hooks in the same phase simultaneously — the dependent job will fail if the
dependency has not completed yet.

**9. GitOps has a bootstrap problem — and that is normal**
The SealedSecret manages `mongodb-secret` declaratively in Git. But the
PreSync hook needs `mongodb-secret` before the Sync phase creates it from
the SealedSecret. The solution is a one-time bootstrap `kubectl create secret`
before the first sync. This is not a failure of GitOps — it is an honest
acknowledgement that every system has a first-run bootstrapping step that
cannot be fully automated without additional tooling. The SealedSecret takes
over management from the second sync onwards.

**10. Make hook env vars consistent with application env vars**
The PreSync DB init hook connects to the same MongoDB as the backend. Both
use `MONGODB_HOST=mongodb` and `MONGODB_DATABASE=course-goals`. Keeping these
values identical and explicit in both places ensures that if you ever change
the MongoDB Service name or database, you update both the backend Deployment
and the hook — and both stay in sync. Never hardcode connection strings in
hooks that differ from the application's own configuration.
---

## What's Next

**Demo-11: Sync Waves**
Sync waves control the order of regular resource creation — not just hooks.
Governance resources (ResourceQuota, NetworkPolicy) must exist before Deployment
pods start. Waves enforce this ordering across any Kubernetes resource, using the
same annotation pattern introduced in Step 8 of this demo.
