# Demo-03 (Prerequisite): Build Podinfo Canary Images

## Overview

Before running Demo-03, this README covers one foundational task:

**Building and pushing two colour-coded podinfo images** — `:red` (stable) and
`:blue` (canary) — with their UI colour and message baked in at build time.

Completing this README gives you the two image tags that Demo-03 uses to
demonstrate a canary rollout. The stable version (`:red`) is deployed first;
the canary version (`:blue`) is what you promote through the rollout steps.

**What you'll learn:**
- Why colour and message are baked into the image rather than set in the manifest
- How `ARG` and `ENV` work together in a multi-stage Dockerfile
- Why the Dockerfile is moved to a subfolder while the build context stays at the repo root
- How to verify baked-in env vars using `docker inspect`
- How to run and visually verify each image locally before deploying to Kubernetes

**What you'll do:**
- Clone your `rselvantech/podinfo` fork
- Create a `03-canary-basics/` subfolder and copy the Dockerfile into it
- Add `ARG UI_COLOR` and `ARG UI_MESSAGE` with `ENV` to the final stage
- Build and push `rselvantech/podinfo:red` and `rselvantech/podinfo:blue`
- Commit the modified Dockerfile back to your podinfo fork
- Run each image locally and verify colour and message in the browser

## Prerequisites

- ✅ ArgoCD Demo-05 completed — `rselvantech/podinfo` fork exists on GitHub
- ✅ `rselvantech/podinfo` private Docker Hub repo exists (created in Demo-05 Step 4)
- ✅ Docker Desktop installed and logged into Docker Hub (`rselvantech`)
- ✅ `git` and `docker` available in terminal

> **Reference:** Forking podinfo, creating the private Docker Hub repo, and
> setting up your Docker Hub PAT are all covered in
> **ArgoCD Demo-05 — `README-podinfo-setup.md` Steps 1–6**.
> If you have not completed those steps, do them first before continuing here.

**Verify Prerequisites:**

### 1. Podinfo fork exists on GitHub

```
git ls-remote https://github.com/rselvantech/podinfo.git HEAD
```

**Expected:** 
```
<commit-sha>    HEAD
```
> **Note:** If you see `fatal: repository not found` or an `authentication error`, the fork does not exist or your git credentials are not configured. Complete ArgoCD Demo-05 Step 1 first.

Your fork is visible with the `stefanprodan/podinfo` source listed.

### 2. Docker Hub logged in

```bash
docker info | grep Username
```

**Expected:**
```text
Username: rselvantech
```

### 3. Private Docker Hub repo exists

```
docker pull rselvantech/podinfo:v1.0.0
```

**Expected:**
```
Status: Image is up to date for rselvantech/podinfo:v1.0.0
```
> **Note:** If you get `Error: pull access denied` or `manifest unknown` means the private Docker Hub repo does not exist yet. Complete ArgoCD Demo-05 Steps 4–6 before continuing.


---

## Demo Objectives

By the end of this README, you will:

1. ✅ Understand why colour and message are baked into the image at build time
2. ✅ Understand how `ARG` + `ENV` enables a single Dockerfile to produce multiple images
3. ✅ Understand why the build context must remain the repo root even when the Dockerfile moves
4. ✅ Have `rselvantech/podinfo:red` pushed to Docker Hub with `#e74c3c` and `v1 — stable (red)` baked in
5. ✅ Have `rselvantech/podinfo:blue` pushed to Docker Hub with `#3498db` and `v2 — canary (blue)` baked in
6. ✅ Have the modified Dockerfile committed to `rselvantech/podinfo` on GitHub
7. ✅ Visually verified both images in the browser

---

## Concepts

### Why Bake Colour and Message Into the Image?

In this demo, `PODINFO_UI_COLOR` and `PODINFO_UI_MESSAGE` are set at **build
time** via `ARG` and `ENV` in the Dockerfile — not at runtime via manifest env
vars. This is intentional.

In production, a new version means a new image — new code, new build, new tag.
The image *is* the version. By baking the colour and message into the image,
you can confirm the correct image is running just by looking at the UI, with no
need to inspect manifests or env vars.

`:red` represents your stable version (`v1.0.0` equivalent).
`:blue` represents your canary version (`v2.0.0` equivalent).

Anyone looking at the browser, `kubectl describe pod`, or the Argo Rollouts
dashboard immediately sees which image tag is serving traffic. This mirrors
how real teams use `:v1.0.0` vs `:v2.0.0` — the tag identifies the version,
and the visual difference confirms the right image is running.

This Demo 03 uses two image tags to represent the stable version (`red`) and
the new canary version (`blue`). Both are built from your existing
`rselvantech/podinfo` fork — the same source, different tags and different
`PODINFO_UI_COLOR` and `PODINFO_UI_MESSAGE` defaults baked into their respective manifests and images.

> **Why two separate image tags rather than one image with different env vars?**
> Changing only an env var on the same image tag is not how production
> deployments work. A new version means a new image tag — new code, new
> build, identifiable by tag. Using `:red` and `:blue` as version identifiers
> mirrors the production pattern of `:v1.0.0` and `:v2.0.0`. The colour
> serves as a visual confirmation that the right image is running. Anyone
> looking at `kubectl describe pod` or the rollout dashboard immediately sees
> which image tag is serving traffic without needing to decode env vars.

---

### How `ARG` + `ENV` Works in a Multi-Stage Dockerfile

The podinfo Dockerfile is a **two-stage build**:

```
Stage 1 (builder):   golang image — compiles the Go binary
Stage 2 (final):     alpine image — runs the compiled binary
```

`ARG` declares a build-time variable passed via `--build-arg`.
`ENV` sets a runtime environment variable in the final image.

Using both together:

```dockerfile
ARG UI_COLOR="#34577c"       # build-time default — overridden by --build-arg
ENV PODINFO_UI_COLOR=${UI_COLOR}  # baked into the image layer as a runtime env var
```

The `ENV` must go in the **final stage** — not the builder stage — because it
is the final stage that becomes the running container. The builder stage is
discarded after compilation.

---

### Why the Build Context Must Stay at the Repo Root

The Dockerfile is copied into `03-canary-basics/` — but the source files
(`cmd/`, `pkg/`, `ui/`, `go.mod`) remain in the repo root. The Dockerfile
contains:

```dockerfile
COPY . .       # ← copies the entire build context into the builder
COPY ./ui ./ui # ← copies ui/ from the build context into the final stage
```

Both lines require the **repo root** as the build context. If you ran
`docker build 03-canary-basics/` (subfolder as context), Docker would look
for `cmd/`, `pkg/`, `ui/` inside that subfolder and fail.

The solution: pass `.` (repo root) as context and point to the Dockerfile
with `-f`:

```bash
docker build -f 03-canary-basics/Dockerfile --build-arg UI_COLOR="..." -t ... .
#                ↑ Dockerfile location                                      ↑ build context
```

---

## Folder Structure

```
03-canary-basics/
├── README-build-podinfo-images.md   ← This file
├── README.md                        ← Demo-03 main walkthrough
└── src/
    └── argo-rollouts-config/        ← git init → remote: rselvantech/argo-rollouts-config
        └── 03-canary-basics/
            ├── rollout.yaml         ← Rollout CRD with canary strategy
            └── service.yaml         ← Service pointing to Rollout pods
```

> **Note:** The `src/podinfo/` clone is a local working directory for the
> build steps below. It is not committed to `gitops-labs`. The modified
> `Dockerfile` is committed to `rselvantech/podinfo` — the app source repo.

---

## Step 1: Clone Your Podinfo Fork

```bash
cd ~/gitops-labs/argo-rollouts-basics-to-prod/03-canary-basics/src
mkdir -p cd podinfo &&  cd podinfo

git init

git remote add origin https://rselvantech:<GITHUB_PAT>@github.com/rselvantech/podinfo.git

# Pull existing history into this fresh local repo
git pull origin main --allow-unrelated-histories --no-rebase
```

**Verify:**

```bash
ls
```

**Expected:**
```text
Dockerfile   Makefile   README.md   charts/   cmd/   kustomize/   pkg/   ui/   ...
```

---

## Step 2: Create the Subfolder and Copy the Dockerfile

```bash
mkdir -p 03-canary-basics
cp Dockerfile 03-canary-basics/Dockerfile
```

**Verify:**

```bash
ls 03-canary-basics/
```

**Expected:**
```text
Dockerfile
```

---

## Step 3: Modify the Dockerfile — Add `ARG` and `ENV` in the Final Stage

Open `03-canary-basics/Dockerfile` and add the following four lines in the
**final (`alpine`) stage**, immediately before the `USER app` line:

```dockerfile
# ↓ ADD THESE FOUR LINES — bake colour and message into the image at build time
ARG UI_COLOR="#34577c"
ARG UI_MESSAGE="podinfo"
ENV PODINFO_UI_COLOR=${UI_COLOR}
ENV PODINFO_UI_MESSAGE=${UI_MESSAGE}

USER app
```

The surrounding context in the final stage should look like this after the edit:

```dockerfile
COPY --from=builder /podinfo/bin/podinfo .
COPY --from=builder /podinfo/bin/podcli /usr/local/bin/podcli
COPY ./ui ./ui
RUN chown -R app:app ./

# ↓ bake colour and message into the image at build time
ARG UI_COLOR="#34577c"
ARG UI_MESSAGE="podinfo"
ENV PODINFO_UI_COLOR=${UI_COLOR}
ENV PODINFO_UI_MESSAGE=${UI_MESSAGE}

USER app
```

> **Why in the final stage and not the builder stage?** The `PODINFO_UI_COLOR` env var is read by the running Go binary at startup — it is not a compile-time constant. The builder stage compiles the binary; the final stage is what becomes the running container. `ENV` set in the final stage is what the container process inherits.

> **Why `ARG` + `ENV` rather than just `ENV`?** A plain `ENV PODINFO_UI_COLOR=#ff0000` hardcodes the value — you would need two separate Dockerfiles. `ARG UI_COLOR` makes it a build-time parameter with a default, so a single Dockerfile produces both images by passing `--build-arg` at build time.

---


## Step 4: Build `rselvantech/podinfo:red`

```bash
# Build context = repo root (.)  |  Dockerfile = subfolder
docker build \
  -f 03-canary-basics/Dockerfile \
  --build-arg UI_COLOR="#e74c3c" \
  --build-arg UI_MESSAGE="v1 — stable (red)" \
  -t rselvantech/podinfo:red \
  .
```

**Verify the image was built:**

```bash
docker images | grep "podinfo.*red"
```

**Expected:**
```text
rselvantech/podinfo   red    <digest>   2 minutes ago   ~50MB
```

---

## Step 5: Build  `rselvantech/podinfo:blue`

```bash
docker build \
  -f 03-canary-basics/Dockerfile \
  --build-arg UI_COLOR="#3498db" \
  --build-arg UI_MESSAGE="v2 — canary (blue)" \
  -t rselvantech/podinfo:blue \
  .
```

**Verify the image was built:**

```bash
docker images | grep "podinfo.*blue"
```

**Expected:**
```text
rselvantech/podinfo   blue   <digest>   1 minute ago    ~50MB
```

---

## Step 6: Verify Both Env Vars Are Baked Into Each Image

```bash
docker inspect rselvantech/podinfo:red \
  --format '{{range .Config.Env}}{{println .}}{{end}}' | grep PODINFO_UI
```

**Expected:**
```text
PODINFO_UI_COLOR=#e74c3c
PODINFO_UI_MESSAGE=v1 — stable (red)
```

```bash
docker inspect rselvantech/podinfo:blue \
  --format '{{range .Config.Env}}{{println .}}{{end}}' | grep PODINFO_UI
```

**Expected:**
```text
PODINFO_UI_COLOR=#3498db
PODINFO_UI_MESSAGE=v2 — canary (blue)
```

---

## Step 7: Run and Visually Verify Each Image Locally

> This confirms the baked-in values are actually served by the running binary —
> not just present as metadata in the image config.

**Run the red image:**

```bash
docker run --rm -d \
  --name podinfo-red \
  -p 9898:9898 \
  rselvantech/podinfo:red
```

**Verify via curl:**

```bash
curl -s http://localhost:9898 | grep -E "color|message"
```

**Expected — the UI HTML contains:**
```text
#e74c3c
v1 — stable (red)
```

Open `http://localhost:9898` in your browser.

**Expected:**
- Background colour: red (`#e74c3c`)
- Message displayed: `v1 — stable (red)`
- Hostname: your container ID

**Stop the red container:**

```bash
docker stop podinfo-red
```

**Run the blue image:**

```bash
docker run --rm -d \
  --name podinfo-blue \
  -p 9898:9898 \
  rselvantech/podinfo:blue
```

**Verify via curl:**

```bash
curl -s http://localhost:9898 | grep -E "color|message"
```

**Expected:**
```text
#3498db
v2 — canary (blue)
```

Open `http://localhost:9898` in your browser.

**Expected:**
- Background colour: blue (`#3498db`)
- Message displayed: `v2 — canary (blue)`

**Stop the blue container:**

```bash
docker stop podinfo-blue
```

> **What this proves:** No manifest env vars, no `kubectl`, no Argo Rollouts —
> the running container itself carries the correct colour and message baked at
> build time. When these images are later deployed via a Rollout, the stable
> vs canary split is immediately visible in the browser without inspecting any
> Kubernetes resource.


## Step 8: Verify both tags locally and Push now:**

```bash
docker images | grep podinfo

docker push rselvantech/podinfo:red
docker push rselvantech/podinfo:blue
```

**Expected:**

```text
rselvantech/podinfo   blue   <digest>   1 minute ago    ~50MB
rselvantech/podinfo   red    <digest>   3 minutes ago   ~50MB
```

> **Private Docker Hub repo:** Both tags push to the same private `rselvantech/podinfo` repo from Demo-05. Verify in Docker Hub that `:red` and `:blue` appear alongside `:v1.0.0` with the `Private` label.


## Step 9: Commit and Push the Modified Dockerfile to Your Podinfo Fork

```bash
# Ensure you are in the repo root
cd ~/gitops-labs/argo-rollouts-basics-to-prod/03-canary-basics/src/podinfo

git add 03-canary-basics/Dockerfile

git commit -m "feat(canary): add canary-basics Dockerfile with baked UI_COLOR and UI_MESSAGE args"

git push origin feat/notation
```

**Verify:**

```bash
git log --oneline -1
```

**Expected:**
```text
abc1234 feat(canary): add canary-basics Dockerfile with baked UI_COLOR and UI_MESSAGE args
```

**Verify on GitHub:**

Open `https://github.com/rselvantech/podinfo/tree/main/03-canary-basics` in
your browser.

**Expected:** `03-canary-basics/Dockerfile` is visible and the `ARG UI_COLOR`,
`ARG UI_MESSAGE`, and `ENV` lines are present in the file preview.

> **What lives where — summary:**
>
> | File | Repo | Why |
> |---|---|---|
> | `03-canary-basics/Dockerfile` | `rselvantech/podinfo` | Defines how the image is built — part of app source |
> | `rollout.yaml`, `service.yaml` | `rselvantech/argo-rollouts-config` | Defines how the image is deployed — config repo |
>
> The Dockerfile belongs in the source repo because it defines the build.
> The manifests belong in the config repo because they define the deployment.
> This preserves the same separation of concerns established in ArgoCD Demo-05.


---

## Validation Checklist

Before proceeding to Demo-03, verify:

- [ ] `03-canary-basics/Dockerfile` is committed and visible on `https://github.com/rselvantech/podinfo/tree/main/03-canary-basics`
- [ ] `rselvantech/podinfo:red` tag is visible in Docker Hub with `Private` label
- [ ] `rselvantech/podinfo:blue` tag is visible in Docker Hub with `Private` label
- [ ] `docker inspect rselvantech/podinfo:red` shows `PODINFO_UI_COLOR=#e74c3c`
- [ ] `docker inspect rselvantech/podinfo:red` shows `PODINFO_UI_MESSAGE=v1 — stable (red)`
- [ ] `docker inspect rselvantech/podinfo:blue` shows `PODINFO_UI_COLOR=#3498db`
- [ ] `docker inspect rselvantech/podinfo:blue` shows `PODINFO_UI_MESSAGE=v2 — canary (blue)`
- [ ] Red image renders red background and message `v1 — stable (red)` in browser
- [ ] Blue image renders blue background and message `v2 — canary (blue)` in browser

---

## What's Next

**Demo-03 — Canary Basics: Your First Rollout**

With both images built and pushed, you are ready to:
- Create the `argo-rollouts-config` private GitHub repository
- Write `rollout.yaml` (podinfo:red, 5 replicas, canary strategy)
- Write `service.yaml` pointing to the Rollout pods
- Deploy the stable version and trigger a canary update to `:blue`
- Observe the 20% weight pause and promote to completion