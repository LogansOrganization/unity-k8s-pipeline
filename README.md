# unity-k8s-pipeline

Shared build/deploy pipeline for Unity dedicated servers (Netcode for GameObjects)
running on Kubernetes via [Agones](https://agones.dev/). One reusable GitHub
Actions workflow + one shared Docker image, used by every game repo that
deploys to this cluster — a new game repo needs 3 small files and 3 secrets,
not a copy of this whole setup.

## Why this is split out

There are two Docker images in this pipeline, and only one of them belongs here:

- **The Unity build toolchain** (`Dockerfile.unity-build` / `entrypoint.sh`,
  published as `ghcr.io/logansorganization/unity-ci-build:v1`) is just a Unity
  Editor container that knows how to `git clone` whatever commit it's told to,
  activate a license, run `-executeMethod <method>`, and return the license.
  It contains **no game code** — the same image builds every project. This is
  what lives in this repo.
- **The compiled server image** (a game repo's own `Dockerfile`, packaging its
  `buildServer/` output) is different per project and rebuilt on every push —
  that stays in each game repo, same as before.

## What's here

- `Dockerfile.unity-build` / `entrypoint.sh` — the shared toolchain image.
  `entrypoint.sh` takes `GIT_REPO_URL`, `GIT_REF`, and `BUILD_METHOD` as env
  vars (no project-specific hardcoding), clones, builds, and writes its exit
  code to `/workspace/BUILD_EXIT_CODE`, then holds the pod open (`sleep 300`)
  so CI can `kubectl cp` the output before the Job exits.
- `.github/workflows/deploy.yml` — the reusable workflow (`workflow_call`)
  every game repo's own tiny `deploy.yml` calls. Does the full build → spin up
  a throwaway Kubernetes Job to run the Unity build → extract output → build
  the runtime image → push it → apply the game's Fleet → wait for rollout.
- `.github/workflows/build-toolchain-image.yml` — builds and publishes the
  toolchain image itself, only runs when this repo's own Dockerfile/entrypoint
  change (not on every game's push).
- `k8s/rbac-build.yaml`, `k8s/rbac-deploy.yaml` — one-time cluster RBAC,
  shared across every game repo (not per-project).
- `examples/` — starter files to copy into a new game repo: `deploy.yml`
  (caller workflow), `fleet.yaml`, `Dockerfile`, `Build.cs`.

## One-time cluster setup

Only needed once, ever, regardless of how many game repos use this:

```bash
helm repo add agones https://agones.dev/chart/stable
helm repo update
helm install agones --namespace agones-system --create-namespace agones/agones

kubectl apply -f k8s/rbac-build.yaml
kubectl apply -f k8s/rbac-deploy.yaml

kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=LoganCHobson \
  --docker-password="<PAT with read:packages>" \
  --namespace=default
```

## Adding a new game repo

1. Copy `examples/deploy.yml` to `.github/workflows/deploy.yml` in the new
   repo, and set `server_image_name` to something unique for that game (e.g.
   `ghcr.io/logansorganization/unity-server-mygame`).
2. Copy `examples/fleet.yaml` to `k8s/fleet.yaml`, rename `CHANGEME` to the
   game's name, adjust replicas/resources if needed.
3. Copy `examples/Dockerfile` to the repo root.
4. Copy `examples/Build.cs` to `Assets/Editor/Build.cs`, point `scenes` at the
   project's actual scene(s).
5. Set repo secrets (or org-level secrets, since these can be shared across
   your own projects): `UNITY_LICENSE`, `UNITY_EMAIL`, `UNITY_PASSWORD`.
6. Push to `dev`. That's it — no Dockerfile.unity-build, no entrypoint.sh, no
   build-job.yaml, no RBAC files needed in the new repo at all.

If a game needs a different Unity Editor version than the default toolchain
image, override the `unity_ci_image` input in that repo's `deploy.yml` (see
`build-toolchain-image.yml`'s `workflow_dispatch` for publishing an
alternate-version tag).

## Versioning

Game repos pin to `@v1` (or whatever tag) when calling the reusable workflow.
Bug fixes here land on `main`; cut a new tag when you want existing games to
pick them up, or move the `v1` tag forward if you want it to happen
automatically (tradeoff: less predictable, but zero-touch for every game repo).

## Troubleshooting a stuck build Job

A pod wedged in `ImagePullBackOff` (or anything else that never reaches
Complete/Failed) won't be cleaned up by `ttlSecondsAfterFinished` — that only
fires once the Job actually finishes. Check for and remove strays manually:

```bash
kubectl get jobs -A | grep unity-build
kubectl delete job unity-build-<sha> -n default
```
