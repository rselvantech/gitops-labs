# Project-01: End-to-End GitOps on EKS — GitLab CI + ArgoCD + Image Updater + Cognito RBAC

## Overview

Every demo so far has used a hardcoded image tag (`goals-frontend:v1.0.0`,
`goals-backend:v1.0.0`). The full GitOps loop was never closed — someone still
had to manually update the image tag in the config repo whenever a new version
was built. This is the production gap.

This demo closes that gap entirely. It builds the complete pipeline:

```
Developer pushes code to GitLab
         ↓
GitLab CI builds and pushes new image to GitLab Container Registry
         ↓
ArgoCD Image Updater detects new tag in registry
         ↓
Image Updater writes updated tag back to GitLab config repo (git write-back)
         ↓
ArgoCD detects config repo change and syncs
         ↓
EKS cluster runs new image — Goals saved in MongoDB survive the update
         ↓
AWS Cognito SSO controls who can view and who can sync in ArgoCD
```

Every step is automated. After the first deployment, the only human action
needed to roll out a new version is `git tag v1.1.0 && git push --tags`.

**What you'll learn:**
- How to install and configure ArgoCD on Amazon EKS
- How to use GitLab as the source repository for ArgoCD
- How to set up a GitLab CI pipeline that builds and pushes Docker images
- How ArgoCD Image Updater monitors a container registry for new image tags
- The two Image Updater write-back methods: `argocd` and `git`
- Semver update strategy — what `semver` means and how constraints work
- How AWS Cognito integrates with ArgoCD via OIDC for SSO
- ArgoCD RBAC — `argocd-cm` and `argocd-rbac-cm` ConfigMaps
- Fine-grained permissions: read-only vs admin access per user group

**What you'll do (five stages):**
- **Stage A:** Provision EKS cluster, install ArgoCD and ALB Ingress Controller
- **Stage B:** Set up GitLab repos, push Goals App source, configure and prove CI pipeline
- **Stage C:** Install ArgoCD Image Updater and prove the full automated loop end-to-end
- **Stage D:** Configure AWS Cognito OIDC SSO for ArgoCD login
- **Stage E:** Configure ArgoCD RBAC with two user groups — admin and read-only

---

## The Application — Goals App

The Goals App is the same three-tier application used in Demo-10 (sync hooks).
You already know it: React frontend → Node.js/Express backend → MongoDB.

**Why Goals App for this demo:**
- You own the Dockerfiles for both frontend and backend — real builds
- Already proven on Kubernetes in Demo-10 — no learning curve
- MongoDB data persistence across pod restarts is a visible proof point
- Three-tier architecture is representative of real production applications
- Images already on Docker Hub as baseline (`rselvantech/goals-frontend:v1.0.0`)

**What changes in Demo-15 vs Demo-10:**
- Source code and images move to GitLab (not GitHub/Docker Hub)
- Kubernetes cluster moves to EKS (not minikube)
- Image tags are updated automatically by Image Updater (not hardcoded)
- ArgoCD login uses Cognito SSO (not local admin password)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Demo-15 Production GitOps Architecture               │
│                                                                           │
│  Developer                                                                │
│     │ git tag v1.1.0 + push                                              │
│     ▼                                                                     │
│  GitLab (gitlab.com)                                                      │
│  ├── goals-app-source (private)   ← source code + Dockerfiles            │
│  │        │ .gitlab-ci.yml                                                │
│  │        │ triggers on new tag                                           │
│  │        ▼                                                               │
│  │   GitLab CI Pipeline                                                   │
│  │        │ docker build goals-backend                                    │
│  │        │ docker push registry.gitlab.com/rselvantech/goals-backend:v1.1.0
│  │        │                                                               │
│  └── goals-app-config (private)   ← Kubernetes manifests                 │
│           ▲                                                               │
│           │ Image Updater writes: goals-backend:v1.1.0                   │
│           │ .argocd-source-goals-app.yaml committed                       │
│                                                                           │
│  AWS (us-east-1)                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  EKS Cluster                                                        │  │
│  │                                                                      │  │
│  │  ┌─────────────────────────────────────────┐                        │  │
│  │  │  argocd namespace                        │                        │  │
│  │  │  ├── ArgoCD (API Server, Repo Server,    │                        │  │
│  │  │  │         App Controller)               │◄── Cognito OIDC SSO   │  │
│  │  │  └── ArgoCD Image Updater               │                        │  │
│  │  │       │ polls registry.gitlab.com        │                        │  │
│  │  │       │ every 2 minutes                  │                        │  │
│  │  └─────────────────────────────────────────┘                        │  │
│  │                                                                      │  │
│  │  ┌─────────────────────────────────────────┐                        │  │
│  │  │  goals-app namespace                     │                        │  │
│  │  │  ├── goals-frontend pod                  │                        │  │
│  │  │  ├── goals-backend pod  ← updated image  │                        │  │
│  │  │  ├── mongodb pod                         │                        │  │
│  │  │  └── goals-db-data PVC  (data persists)  │                        │  │
│  │  └─────────────────────────────────────────┘                        │  │
│  │                                                                      │  │
│  │  ALB Ingress → goals-frontend-svc → frontend pod                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  AWS Cognito                                                              │
│  ├── User Pool: goals-argocd-pool                                         │
│  ├── Group: argocd-admin → can sync, create, delete                      │
│  └── Group: argocd-readonly → can view only                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- ✅ AWS account with IAM user having EKS, EC2, IAM, Cognito permissions
- ✅ AWS CLI v2 configured (`aws configure`)
- ✅ `eksctl` installed
- ✅ `kubectl` available
- ✅ `helm` installed
- ✅ ArgoCD CLI installed
- ✅ GitLab.com account (free tier sufficient)
- ✅ Docker installed locally (for building test images)
- ✅ Goals App source code (from Demo-10 / Docker course)

**Install links:**
- eksctl: https://eksctl.io/installation/
- AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html

**Verify AWS credentials:**
```bash
aws sts get-caller-identity
```

**Expected:**
```json
{
  "Account": "123456789012",
  "UserId": "AIDAXXXXXXXXXXXXXXXXX",
  "Arn": "arn:aws:iam::123456789012:user/your-user"
}
```

---

---

# Stage A: Infrastructure — EKS Cluster + ArgoCD + ALB Controller

---

## A1: Create the EKS Cluster

Create `eks-cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: goals-gitops
  region: us-east-1
  version: "1.31"

managedNodeGroups:
  - name: goals-nodes
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 3
    volumeSize: 20
    iam:
      withAddonPolicies:
        albIngress: true      # required for AWS ALB Controller
        certManager: true

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
```

**Create the cluster:**
```bash
eksctl create cluster -f eks-cluster.yaml
```

> This takes 15-20 minutes. eksctl creates the VPC, subnets, node group,
> and kubeconfig automatically.

**Verify:**
```bash
kubectl get nodes
```

**Expected:**
```text
NAME                          STATUS   ROLES    AGE   VERSION
ip-192-168-x-x.ec2.internal   Ready    <none>   2m    v1.31.x
ip-192-168-y-y.ec2.internal   Ready    <none>   2m    v1.31.x
```

---

## A2: Install ArgoCD on EKS

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --version 7.8.23 \
  --namespace argocd \
  --set server.service.type=ClusterIP
```

> `server.service.type=ClusterIP` — ArgoCD will be exposed via ALB Ingress
> (provisioned in A3), not a LoadBalancer service. This is the production pattern.

**Wait for all pods:**
```bash
kubectl get pods -n argocd --watch
```

**Expected:** All 7 pods `Running` and `1/1` Ready.

**Get initial admin password:**
```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## A3: Install AWS Load Balancer Controller

The ALB Controller provisions AWS Application Load Balancers from Kubernetes
Ingress resources. Required to expose ArgoCD and the Goals App externally.

**Create IAM policy for ALB Controller:**
```bash
curl -o iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

**Create IRSA (IAM Role for Service Account):**
```bash
eksctl create iamserviceaccount \
  --cluster=goals-gitops \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Install the controller:**
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=goals-gitops \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**Verify:**
```bash
kubectl get deployment aws-load-balancer-controller -n kube-system
```

**Expected:** `READY 2/2`

---

## A4: Expose ArgoCD via ALB Ingress

Create `argocd-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS
    alb.ingress.kubernetes.io/success-codes: "200"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

```bash
kubectl apply -f argocd-ingress.yaml
```

**Wait for ALB to provision (2-3 minutes):**
```bash
kubectl get ingress argocd-ingress -n argocd -w
```

**Expected:**
```text
NAME              CLASS   HOSTS   ADDRESS                                      PORTS
argocd-ingress    alb     *       k8s-argocd-xxx.us-east-1.elb.amazonaws.com   80
```

**Save the ALB URL:**
```bash
ARGOCD_URL=$(kubectl get ingress argocd-ingress -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD URL: http://$ARGOCD_URL"
```

**Open in browser:** `http://<ARGOCD_ALB_URL>` — you should see the ArgoCD
login page.

**Configure ArgoCD server URL** (required for OIDC redirect in Stage D):
```bash
kubectl patch configmap argocd-cm -n argocd \
  --patch "{\"data\":{\"url\":\"http://$ARGOCD_URL\"}}"
```

**Log in via CLI:**
```bash
argocd login $ARGOCD_URL \
  --username admin \
  --password <initial-password> \
  --insecure
```

**Change admin password:**
```bash
argocd account update-password \
  --current-password <initial-password> \
  --new-password <your-new-password>
```

---

# Stage B: GitLab Setup + Goals App + CI Pipeline

---

## B1: Create GitLab Repositories

Go to `gitlab.com` and create two private repositories:

**Repository 1: `goals-app-source`**
- Contains the Goals App source code (React frontend + Node.js backend)
- Contains `.gitlab-ci.yml` for the CI pipeline
- Contains `Dockerfile` for both services

**Repository 2: `goals-app-config`**
- Contains Kubernetes manifests for EKS deployment
- ArgoCD watches this repo
- Image Updater writes new image tags back to this repo

---

## B2: Push Goals App Source to GitLab

The Goals App source code comes from Demo-10 (goals-app-config GitHub repo).
Copy the source to a local directory and push to GitLab.

**Local directory structure for `goals-app-source`:**
```
goals-app-source/
├── .gitlab-ci.yml
├── backend/
│   ├── Dockerfile
│   ├── app.js
│   ├── package.json
│   └── models/
│       └── goal.js
└── frontend/
    ├── Dockerfile
    ├── package.json
    └── src/
        └── (React source files)
```

**Initialise and push:**
```bash
cd goals-app-source

git init
git branch -M main
git remote add origin https://oauth2:<GITLAB_PAT>@gitlab.com/rselvantech/goals-app-source.git
git add .
git commit -m "feat: initial Goals App source — backend and frontend"
git push origin main
```

> **GitLab PAT scope:** Create a Personal Access Token in GitLab
> (Settings → Access Tokens) with `read_repository` and `write_repository`
> scopes. This PAT is used for git push. The CI pipeline uses its own
> built-in `CI_REGISTRY_*` variables — no manual configuration needed.

---

## B3: Create the GitLab CI Pipeline

Create `.gitlab-ci.yml` in the `goals-app-source` repo root:

```yaml
# .gitlab-ci.yml
# Triggers only on git tags (v1.0.0, v1.1.0, etc.)
# Builds and pushes both frontend and backend images to GitLab Container Registry

stages:
  - build

variables:
  BACKEND_IMAGE: $CI_REGISTRY_IMAGE/goals-backend
  FRONTEND_IMAGE: $CI_REGISTRY_IMAGE/goals-frontend
  IMAGE_TAG: $CI_COMMIT_TAG       # uses the git tag as image tag

build-backend:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - echo "Building goals-backend:$IMAGE_TAG"
    - docker build -t $BACKEND_IMAGE:$IMAGE_TAG ./backend
    - docker push $BACKEND_IMAGE:$IMAGE_TAG
    - echo "Pushed $BACKEND_IMAGE:$IMAGE_TAG"
  rules:
    - if: $CI_COMMIT_TAG        # only run when a git tag is pushed

build-frontend:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - echo "Building goals-frontend:$IMAGE_TAG"
    - docker build -t $FRONTEND_IMAGE:$IMAGE_TAG ./frontend
    - docker push $FRONTEND_IMAGE:$IMAGE_TAG
    - echo "Pushed $FRONTEND_IMAGE:$IMAGE_TAG"
  rules:
    - if: $CI_COMMIT_TAG
```

> **`$CI_REGISTRY_IMAGE`** is a GitLab built-in variable — automatically set
> to `registry.gitlab.com/rselvantech/goals-app-source`. The full image paths
> become:
> - `registry.gitlab.com/rselvantech/goals-app-source/goals-backend:v1.0.0`
> - `registry.gitlab.com/rselvantech/goals-app-source/goals-frontend:v1.0.0`
>
> **Why trigger on tags only?** Every commit to main would rebuild images
> unnecessarily. Tags represent intentional releases. `v1.0.0`, `v1.1.0` etc.
> are the semver tags that Image Updater tracks.

**Push the CI file:**
```bash
git add .gitlab-ci.yml
git commit -m "feat: add GitLab CI pipeline for backend and frontend images"
git push origin main
```

---

## B4: Trigger the First Build

Push the first release tag to trigger the CI pipeline:

```bash
git tag v1.0.0
git push origin v1.0.0
```

**Watch the pipeline in GitLab:**
- Go to `gitlab.com/rselvantech/goals-app-source` → CI/CD → Pipelines
- You should see a pipeline triggered by `v1.0.0` tag
- Two jobs: `build-backend` and `build-frontend`

**Wait for both jobs to complete (3-5 minutes).**

**Verify images in GitLab Container Registry:**
- Go to `gitlab.com/rselvantech/goals-app-source` → Deploy → Container Registry
- You should see:
  - `goals-backend` with tag `v1.0.0`
  - `goals-frontend` with tag `v1.0.0`

**Expected in registry:**
```text
registry.gitlab.com/rselvantech/goals-app-source/goals-backend:v1.0.0
registry.gitlab.com/rselvantech/goals-app-source/goals-frontend:v1.0.0
```

---

## B5: Add Config Repo Manifests

Push the Kubernetes manifests to `goals-app-config`:

```bash
cd goals-app-config

git init
git branch -M main
git remote add origin https://oauth2:<GITLAB_PAT>@gitlab.com/rselvantech/goals-app-config.git
```

**`demo-15/namespace.yaml`:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: goals-app
```

**`demo-15/mongodb-pvc.yaml`:**
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
  storageClassName: gp2   # AWS EBS default storage class
```

**`demo-15/mongodb-deployment.yaml`:**
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

**`demo-15/mongodb-service.yaml`:**
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

**`demo-15/backend-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goals-backend
  namespace: goals-app
  annotations:
    argocd-image-updater.argoproj.io/image-list: >
      backend=registry.gitlab.com/rselvantech/goals-app-source/goals-backend
    argocd-image-updater.argoproj.io/backend.update-strategy: semver
    argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
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
          image: registry.gitlab.com/rselvantech/goals-app-source/goals-backend:v1.0.0
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

**`demo-15/backend-service.yaml`:**
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

**`demo-15/frontend-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goals-frontend
  namespace: goals-app
  annotations:
    argocd-image-updater.argoproj.io/image-list: >
      frontend=registry.gitlab.com/rselvantech/goals-app-source/goals-frontend
    argocd-image-updater.argoproj.io/frontend.update-strategy: semver
    argocd-image-updater.argoproj.io/frontend.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
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
          image: registry.gitlab.com/rselvantech/goals-app-source/goals-frontend:v1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: REACT_APP_BACKEND_URL
              value: "http://goals-backend-svc:80"
          stdin: true
          tty: true
```

**`demo-15/frontend-service.yaml`:**
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

**`demo-15/frontend-ingress.yaml`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: goals-frontend-ingress
  namespace: goals-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/success-codes: "200"
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: goals-frontend-svc
                port:
                  number: 3000
```

**Push to `goals-app-config`:**
```bash
git add demo-15/
git commit -m "feat: add demo-15 Goals App manifests for EKS deployment"
git push origin main
```

---

## B6: Register GitLab Repos with ArgoCD

ArgoCD needs credentials to clone both GitLab repos.

**Create a GitLab Deploy Token** (preferred over PAT for ArgoCD read access):
- Go to `goals-app-config` repo → Settings → Repository → Deploy Tokens
- Create token with name `argocd-reader`, scope: `read_repository`
- Save the username and token value

**Register `goals-app-config` with ArgoCD:**
```bash
argocd repo add https://gitlab.com/rselvantech/goals-app-config.git \
  --username <deploy-token-username> \
  --password <deploy-token-value>
```

**Verify:**
```bash
argocd repo list
```

**Expected:** `goals-app-config` listed with `Successful` status.

---

## B7: Create the MongoDB Secret

The MongoDB secret is never committed to Git — created imperatively:

```bash
kubectl create namespace goals-app

kubectl create secret generic mongodb-secret \
  --namespace goals-app \
  --from-literal=MONGODB_USERNAME=rselvantech \
  --from-literal=MONGODB_PASSWORD=passWD \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=rselvantech \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=passWD
```

---

## B8: Create the Goals App ArgoCD Application

**Create `goals-app.yaml` in `argocd-config`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: goals-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/rselvantech/goals-app-config.git
    targetRevision: HEAD
    path: demo-15
  destination:
    server: https://kubernetes.default.svc
    namespace: goals-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f goals-app.yaml
```

**Watch sync:**
```bash
argocd app get goals-app --watch
```

**Expected — all pods running:**
```bash
kubectl get pods -n goals-app
```

```text
NAME                              READY   STATUS
mongodb-xxxxxxxxx-xxxxx           1/1     Running
goals-backend-xxxxxxxxx-xxxxx     1/1     Running
goals-frontend-xxxxxxxxx-xxxxx    1/1     Running
```

**Get the Goals App ALB URL:**
```bash
GOALS_URL=$(kubectl get ingress goals-frontend-ingress -n goals-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Goals App: http://$GOALS_URL"
```

**Open `http://$GOALS_URL` in browser. Add a goal. Verify it persists on
page reload.** This proves the full stack is working on EKS before we
add Image Updater.

---

# Stage C: ArgoCD Image Updater — Closing the GitOps Loop

---

## C1: What ArgoCD Image Updater Does

ArgoCD Image Updater is a controller that runs inside the cluster. It:
1. Reads annotations on ArgoCD Application resources (or Deployment resources)
   to find which images to watch
2. Polls the container registry for new tags matching the update strategy
3. When a newer tag is found, updates the Application to use the new image

**Two write-back methods:**

| Method | What it does | Persistent? |
|---|---|---|
| `argocd` | Updates the ArgoCD Application resource parameter directly | ❌ Lost if Application is resynced from Git |
| `git` | Commits `.argocd-source-<appname>.yaml` to the config repo | ✅ Survives full resync |

**This demo uses `git` write-back** — the production method. Image Updater
commits to `goals-app-config`, ArgoCD detects the commit and syncs.

**Update strategies:**

| Strategy | Selects |
|---|---|
| `semver` | Highest version satisfying the semver constraint |
| `newest-build` | Most recently pushed image |
| `alphabetical` | Last tag alphabetically |
| `digest` | Updates when image digest changes (for `latest` tracking) |

This demo uses `semver` — the production default. Any tag matching `^v[0-9]+\.[0-9]+\.[0-9]+$`
(e.g. `v1.0.0`, `v1.1.0`) is considered. The highest semver is deployed.

---

## C2: Install ArgoCD Image Updater

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

**Verify:**
```bash
kubectl get pods -n argocd | grep image-updater
```

**Expected:** `argocd-image-updater-xxx   1/1   Running`

**Check logs to confirm it started:**
```bash
kubectl logs -n argocd deployment/argocd-image-updater --tail=20
```

**Expected:**
```text
time="..." level=info msg="ArgoCD Image Updater starting ..."
time="..." level=info msg="Starting image update cycle ..."
```

---

## C3: Configure GitLab Registry Access

Image Updater needs credentials to pull from the private GitLab Container
Registry.

**Create a GitLab Deploy Token for registry pull access:**
- Go to `goals-app-source` repo → Settings → Repository → Deploy Tokens
- Name: `image-updater-reader`
- Scopes: `read_registry`
- Save the username and token

**Create the registry credentials secret:**
```bash
kubectl create secret docker-registry gitlab-registry-credentials \
  --namespace argocd \
  --docker-server=registry.gitlab.com \
  --docker-username=<deploy-token-username> \
  --docker-password=<deploy-token-value>
```

**Configure Image Updater to use this registry:**

```bash
kubectl patch configmap argocd-image-updater-config \
  -n argocd \
  --patch '
data:
  registries.conf: |
    registries:
      - name: GitLab Registry
        api_url: https://registry.gitlab.com
        prefix: registry.gitlab.com
        credentials: pullsecret:argocd/gitlab-registry-credentials
        default: false
        insecure: false
'
```

**Restart Image Updater to pick up the config:**
```bash
kubectl rollout restart deployment argocd-image-updater -n argocd
```

---

## C4: Configure Git Write-Back Credentials

Image Updater needs to **write** to `goals-app-config` to commit the updated
image tag. This requires write access to the GitLab repo.

**Create a GitLab PAT with write access:**
- Go to GitLab → User Settings → Access Tokens
- Name: `image-updater-writer`
- Scopes: `write_repository`
- Save the token value

**Create the git credentials secret:**
```bash
kubectl create secret generic argocd-image-updater-gitlab-creds \
  --namespace argocd \
  --from-literal=username=rselvantech \
  --from-literal=password=<gitlab-pat-write>
```

**Configure Image Updater to use these credentials for write-back:**
```bash
kubectl patch configmap argocd-image-updater-config \
  -n argocd \
  --patch-type=merge \
  -p '{"data":{"git.credentials-store":"secret","git.user":"ArgoCD Image Updater","git.email":"image-updater@goals-app.local"}}'
```

---

## C5: Add Image Updater Annotations to the ArgoCD Application

The annotations on the Application CRD tell Image Updater which images to
watch and how to update them. Update `goals-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: goals-app
  namespace: argocd
  annotations:
    # Watch these two images
    argocd-image-updater.argoproj.io/image-list: >
      backend=registry.gitlab.com/rselvantech/goals-app-source/goals-backend,
      frontend=registry.gitlab.com/rselvantech/goals-app-source/goals-frontend

    # Update strategy: semver — pick the highest v*.*.* tag
    argocd-image-updater.argoproj.io/backend.update-strategy: semver
    argocd-image-updater.argoproj.io/frontend.update-strategy: semver

    # Only consider tags matching semantic versioning
    argocd-image-updater.argoproj.io/backend.allow-tags: "regexp:^v[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/frontend.allow-tags: "regexp:^v[0-9]+\\.[0-9]+\\.[0-9]+$"

    # Use git write-back — commits to goals-app-config repo
    argocd-image-updater.argoproj.io/write-back-method: git

    # Which branch to commit to
    argocd-image-updater.argoproj.io/git-branch: main

    # Credentials for writing to GitLab
    argocd-image-updater.argoproj.io/write-back-target: kustomization

spec:
  project: default
  source:
    repoURL: https://gitlab.com/rselvantech/goals-app-config.git
    targetRevision: main           # must track a branch, not HEAD, for write-back
    path: demo-15
  destination:
    server: https://kubernetes.default.svc
    namespace: goals-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f goals-app.yaml
```

---

## C6: Prove the End-to-End Loop

This is the payoff step. Make a code change, push a new tag, and watch
the entire pipeline run automatically.

**Make a visible change to the backend:**

Edit `backend/app.js` in `goals-app-source` — add a version endpoint:

```javascript
// Add this route to app.js
app.get('/version', (req, res) => {
  res.json({ version: process.env.APP_VERSION || 'v1.1.0', service: 'goals-backend' });
});
```

**Commit and push the new tag:**
```bash
cd goals-app-source
git add backend/app.js
git commit -m "feat: add /version endpoint to backend"
git tag v1.1.0
git push origin main
git push origin v1.1.0
```

**Watch GitLab CI build the new image:**
- GitLab → CI/CD → Pipelines — pipeline triggered by `v1.1.0` tag
- Wait for `build-backend` to complete (~3 minutes)

**Watch Image Updater detect the new tag:**
```bash
kubectl logs -n argocd deployment/argocd-image-updater -f | grep goals
```

**Expected output when new tag is detected:**
```text
level=info msg="Found 1 application(s) with image registry.gitlab.com/.../goals-backend"
level=info msg="Setting new image to registry.gitlab.com/.../goals-backend:v1.1.0"
level=info msg="Successfully updated image to v1.1.0"
level=info msg="Committing 1 parameter update(s) for application goals-app"
level=info msg="Successfully pushed commit to repository"
```

**Watch ArgoCD detect the write-back commit and sync:**
```bash
watch -n 5 'argocd app get goals-app | grep -E "Sync|Health|Image"'
```

**Expected — application goes OutOfSync then Synced with new image:**
```text
Sync Status:   OutOfSync  ← Image Updater committed new tag
Health Status: Healthy
# ...seconds later...
Sync Status:   Synced     ← ArgoCD applied the new image
Health Status: Healthy
```

**Verify new image is running:**
```bash
kubectl get deployment goals-backend -n goals-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected:** `registry.gitlab.com/rselvantech/goals-app-source/goals-backend:v1.1.0`

**Verify goals persisted across the update:**
```bash
# Get the ALB URL
echo "http://$GOALS_URL"
```

Open in browser — any goals added before the update should still be there.
The MongoDB PVC data survived the backend pod restart.

**Verify the new endpoint works:**
```bash
curl http://$GOALS_URL/goals/version
# Expected: {"version":"v1.1.0","service":"goals-backend"}
```

**The complete loop is proven:**
```
git tag v1.1.0 → GitLab CI → registry push → Image Updater → git commit
→ ArgoCD sync → new pod → goals still in database
```

---

# Stage D: AWS Cognito SSO for ArgoCD

---

## D1: Create a Cognito User Pool

Go to the AWS Console → Cognito → Create User Pool.

**Step through the wizard:**

| Setting | Value |
|---|---|
| Sign-in options | Email |
| Security requirements | Defaults |
| Sign-up experience | Disable self-registration |
| Email delivery | Cognito default (SES not required for demo) |
| User pool name | `goals-argocd-pool` |
| App client name | `argocd-client` |
| App type | Confidential client |
| Callback URL | `http://<ARGOCD_ALB_URL>/auth/callback` |
| Sign-out URL | `http://<ARGOCD_ALB_URL>/logout` |
| OAuth grant types | ✅ Authorization code grant |
| OpenID scopes | ✅ openid ✅ profile ✅ email |

**Configure Cognito domain** (required for OAuth redirect):
- App Integration → Domain → Create Cognito domain
- Domain prefix: `goals-argocd-demo` (must be globally unique)
- Note the full domain: `https://goals-argocd-demo.auth.us-east-1.amazoncognito.com`

**Save for later:**
- User Pool ID: `us-east-1_XXXXXXXXX`
- App Client ID: `xxxxxxxxxxxxxxxxxxxxxxxxxx`
- App Client Secret: `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- Cognito Domain: `https://goals-argocd-demo.auth.us-east-1.amazoncognito.com`

---

## D2: Create Cognito Groups and Users

**Create two groups:**

```
Group 1: argocd-admin
  Description: ArgoCD administrators — can sync, create, delete applications

Group 2: argocd-readonly
  Description: Read-only access to ArgoCD — can view applications and logs
```

In the Cognito console:
- User Pool → Groups → Create group: `argocd-admin`
- User Pool → Groups → Create group: `argocd-readonly`

**Create two users:**

```
User 1: admin-user@example.com
  Add to group: argocd-admin

User 2: dev-user@example.com
  Add to group: argocd-readonly
```

In the Cognito console:
- User Pool → Users → Create user (for each)
- After creating: select user → Add user to group

---

## D3: Configure ArgoCD OIDC

Update the `argocd-cm` ConfigMap to point to Cognito as the OIDC provider:

```bash
kubectl patch configmap argocd-cm -n argocd --patch "
data:
  url: http://$ARGOCD_URL
  oidc.config: |
    name: AWS Cognito
    issuer: https://cognito-idp.us-east-1.amazonaws.com/<YOUR_USER_POOL_ID>
    clientID: <YOUR_APP_CLIENT_ID>
    clientSecret: <YOUR_APP_CLIENT_SECRET>
    requestedScopes: [\"openid\", \"profile\", \"email\"]
    requestedIDTokenClaims: {\"cognito:groups\": {\"essential\": true}}
"
```

**Restart ArgoCD server to apply:**
```bash
kubectl rollout restart deployment argocd-server -n argocd
```

**Verify SSO button appears:**

Open `http://<ARGOCD_ALB_URL>` in browser — you should see a
"LOG IN VIA AWS COGNITO" button alongside the local username/password form.

---

## D4: Test SSO Login

Click "LOG IN VIA AWS COGNITO" → redirects to Cognito hosted UI →
log in as `admin-user@example.com` with the password set in D2.

After successful login you are redirected back to ArgoCD. However, you may
see no applications or a "permission denied" message. This is expected —
Cognito has authenticated you but ArgoCD's RBAC has not yet assigned a role
to your group. Configure RBAC in Stage E.

---

# Stage E: ArgoCD RBAC — Group-Based Access Control

---

## E1: Understand ArgoCD RBAC

ArgoCD RBAC is configured via two ConfigMaps in the `argocd` namespace:

**`argocd-cm`** — OIDC configuration + which claim carries group information

**`argocd-rbac-cm`** — who (which group) gets what permission (which role)

**Built-in roles:**

| Role | Permissions |
|---|---|
| `role:admin` | Full access — create, sync, delete everything |
| `role:readonly` | View only — cannot sync, create, or delete |

**Custom role syntax (policy.csv):**
```
p, <role>, <resource>, <action>, <project>/<app>, <allow/deny>
g, <group-or-user>, <role>
```

---

## E2: Configure RBAC ConfigMap

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --patch "
data:
  policy.default: role:readonly
  policy.csv: |
    g, argocd-admin, role:admin
    g, argocd-readonly, role:readonly
  scopes: '[cognito:groups, email]'
"
```

**What this does:**
- `policy.default: role:readonly` — any authenticated user with no matching
  group gets read-only access. A safety net.
- `g, argocd-admin, role:admin` — Cognito group `argocd-admin` gets full admin
  access. The `g` means "group mapping".
- `g, argocd-readonly, role:readonly` — Cognito group `argocd-readonly` gets
  read-only access.
- `scopes: '[cognito:groups, email]'` — tells ArgoCD to look in the
  `cognito:groups` claim of the OIDC token for group membership.

**Restart to apply:**
```bash
kubectl rollout restart deployment argocd-server -n argocd
```

---

## E3: Verify RBAC — Admin User

Log in as `admin-user@example.com` via SSO:

```
http://<ARGOCD_ALB_URL> → LOG IN VIA AWS COGNITO → admin-user credentials
```

**Expected — admin user can:**
- ✅ See all applications (`goals-app`)
- ✅ Click SYNC and trigger a manual sync
- ✅ Access Settings → Repositories → manage repos
- ✅ Delete applications (do not actually delete)

**Verify via CLI:**
```bash
argocd login $ARGOCD_URL --sso
# Opens browser for SSO — log in as admin-user
argocd app list
# Expected: goals-app visible
argocd app sync goals-app
# Expected: sync triggers successfully
```

---

## E4: Verify RBAC — Read-Only User

Log out of ArgoCD and log in as `dev-user@example.com`:

```
ArgoCD UI → Log Out → LOG IN VIA AWS COGNITO → dev-user credentials
```

**Expected — read-only user can:**
- ✅ See all applications (`goals-app`)
- ✅ View sync status, health, resource tree
- ❌ Cannot trigger SYNC (button is greyed out or returns permission error)
- ❌ Cannot access Settings → Repositories
- ❌ Cannot delete applications

**Verify SYNC is blocked:**
- Click on `goals-app` → try to click SYNC
- Expected: Permission denied error or button disabled

This confirms that Cognito group membership (`argocd-admin` vs `argocd-readonly`)
directly controls what each user can do in ArgoCD — without managing local
ArgoCD accounts manually.

---

## Verify Final State — Complete Demo Checklist

```bash
# EKS cluster healthy
kubectl get nodes

# All pods running
kubectl get pods -n goals-app
kubectl get pods -n argocd

# Goals App accessible
echo "http://$GOALS_URL"
# Open in browser — add a goal, verify it persists

# ArgoCD accessible
echo "http://$ARGOCD_URL"
# SSO login works for both users

# Image Updater running
kubectl get pods -n argocd | grep image-updater

# Both repos registered
argocd repo list

# Application synced
argocd app get goals-app
```

---

## Cleanup

```bash
# Delete ArgoCD Applications
kubectl delete app goals-app -n argocd

# Delete EKS cluster (deletes all AWS resources including ALBs)
eksctl delete cluster --name goals-gitops

# Delete Cognito User Pool
# AWS Console → Cognito → goals-argocd-pool → Delete

# Delete IAM policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy

# GitLab repos — can keep for reference
# GitLab → goals-app-source → Settings → General → Advanced → Delete project
# GitLab → goals-app-config → Settings → General → Advanced → Delete project
```

> **Cost:** EKS cluster + 2 × t3.medium nodes + 2 ALBs ≈ $5-8/day.
> Delete immediately after completing the demo.

---

## Key Concepts Summary

**The complete GitOps loop is now closed**
Before this demo: image tags were hardcoded, deployments required manual
config repo updates. After: code change → git tag → CI builds → Image Updater
detects → writes config → ArgoCD syncs. Zero manual steps after the tag push.

**Image Updater git write-back is the production method**
The `argocd` write-back method updates the Application directly and is lost on
resync. The `git` method commits `.argocd-source-<appname>.yaml` to the config
repo — the update is versioned, auditable, and survives any ArgoCD restart or
resync.

**Semver strategy requires proper git tags**
Image Updater's semver strategy selects the highest semver-compatible tag.
Using `v1.0.0`, `v1.1.0`, `v2.0.0` format is required. Tags like `latest`,
`main`, or commit SHAs will not be selected by the semver strategy.

**Cognito group names in policy.csv must match exactly**
The Cognito group name (`argocd-admin`) must match exactly what appears in
the `cognito:groups` claim of the OIDC token. Any mismatch means the group
mapping is never applied and users get the `policy.default` role.

**`policy.default: role:readonly` is safer than `policy.default: ''`**
An empty default denies all access to authenticated users with no matching
group. `role:readonly` is more forgiving — anyone who can log in can at least
see what is deployed. Choose based on your security requirements.

**ALB is required for OIDC redirect — port-forward does not work**
Cognito requires a real URL for the callback redirect. `localhost:8080`
cannot be the callback URL for a Cognito User Pool app client. The ALB
provides the stable public URL needed for the OIDC flow.

---

## Lessons Learned

**1. ArgoCD Application must track a branch for git write-back**
`targetRevision: HEAD` does not work with Image Updater git write-back.
Image Updater commits to a branch — use `targetRevision: main` so ArgoCD
picks up the commit.

**2. GitLab Container Registry requires a Deploy Token for Image Updater**
A PAT works but is user-scoped and expires. A Deploy Token is repo-scoped,
has no expiry by default, and survives user account changes.

**3. MongoDB Secret must be created before the ArgoCD Application syncs**
ArgoCD will attempt to create pods that reference `mongodb-secret` immediately
on first sync. If the secret does not exist, pods go into `CreateContainerConfigError`.
Always create the secret before applying the Application CRD.

**4. EKS requires `gp2` storage class for PVC**
The default storage class on EKS is `gp2` (EBS). Omitting `storageClassName`
in the PVC should work but specifying it explicitly avoids any ambiguity.

**5. Cognito requires a domain for the hosted UI**
Without a Cognito domain, there is no hosted login page and no OAuth redirect.
Always configure the domain immediately after creating the User Pool.
