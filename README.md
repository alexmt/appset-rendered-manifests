# ArgoCD Rendered Manifests Pattern with ApplicationSet

This repository demonstrates the **rendered manifests** pattern for ArgoCD ApplicationSets.
Instead of deploying an ApplicationSet controller to the cluster, a CI pipeline renders
the Application manifests and stores them in a dedicated branch. A simple "app-of-apps"
Application syncs those rendered manifests into the cluster.

## Why This Pattern?

- **No ApplicationSet controller required** -- the cluster only needs the core ArgoCD
  controllers, reducing the attack surface and operational overhead.
- **Auditable** -- every change to the generated Applications is tracked as a git commit
  on the `rendered` branch. You can review diffs, revert, and bisect.
- **GitOps-friendly** -- the rendered branch is the single source of truth for what
  Applications exist. Nothing is generated at runtime.
- **Deterministic** -- CI renders manifests in a controlled environment. What you see
  in git is exactly what gets applied.

## Flow

```
 +------------------+       +-------------------+       +------------------+
 |  Developer edits |       |  GitHub Action    |       |  ArgoCD syncs    |
 |  appset/ or apps/| ----> |  renders manifests| ----> |  from rendered   |
 |  on main branch  |       |  to rendered branch|      |  branch          |
 +------------------+       +-------------------+       +------------------+

 Detailed flow:

 1. Developer pushes changes to main branch
    (edits to appset/appset.yaml or apps/**)

 2. GitHub Action triggers:
    a. Checks out main branch
    b. Installs argocd CLI
    c. Runs: argocd appset generate appset/appset.yaml -o yaml
    d. Checks out the orphaned "rendered" branch
    e. Writes the generated Application manifests
    f. Commits and pushes to "rendered" branch

 3. ArgoCD detects changes on the "rendered" branch:
    a. The app-of-apps Application watches the rendered branch
    b. It syncs the rendered Application manifests
    c. Each Application then syncs its own resources from main branch
```

## Repository Structure

```
.
├── README.md                           # This file
├── argocd/                             # ArgoCD installation (kustomize)
│   ├── kustomization.yaml              # Installs ArgoCD + app-of-apps
│   └── app-of-apps.yaml                # Points to the rendered branch
├── appset/
│   └── appset.yaml                     # ApplicationSet template (CI-only)
├── apps/
│   ├── app-a/                          # Sample app: nginx, 2 replicas, port 80
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── app-b/                          # Sample app: nginx, 1 replica, port 8080
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       └── service.yaml
└── .github/
    └── workflows/
        └── render-manifests.yaml       # CI pipeline
```

## Getting Started

### 1. Install ArgoCD

Apply the kustomize-based installation to your cluster:

```bash
kubectl apply -k argocd/
```

This installs ArgoCD and creates the app-of-apps Application that watches the
`rendered` branch.

### 2. Bootstrap the Rendered Branch

On first setup, you need to create the orphaned `rendered` branch. You can do
this manually or let the GitHub Action create it on the first push to `main`.

To create it manually:

```bash
git checkout --orphan rendered
git rm -rf .
echo "# Rendered Application Manifests" > README.md
git add README.md
git commit -m "Initialize rendered branch"
git push origin rendered
git checkout main
```

### 3. Push to Trigger Rendering

Push any change to `appset/` or `apps/` on the `main` branch. The GitHub Action
will render the Application manifests and push them to the `rendered` branch.
ArgoCD will then pick up the changes automatically.

### 4. Verify

```bash
# Check that the app-of-apps is synced
argocd app get app-of-apps

# List the generated applications
argocd app list
```

## Customization

- **Add clusters**: Edit `appset/appset.yaml` and add entries to the `list` generator.
- **Add applications**: Create a new directory under `apps/` with a `kustomization.yaml`.
  The `git` directory generator in the ApplicationSet will pick it up automatically.
- **Change sync policy**: Edit the template in `appset/appset.yaml` to adjust
  sync options, retry strategies, or add sync waves.
