# gitops-cicd-pipeline-config

> The GitOps configuration repository for [gitops-cicd-pipeline](https://github.com/irfanjat/gitops-cicd-pipeline).
> This repo is the **single source of truth** for the Kubernetes cluster state.
> ArgoCD watches this repo and ensures the cluster always matches what is defined here.

---

## What This Repo Is

In a GitOps workflow, the cluster state is never changed manually with `kubectl apply`.
Instead, all changes go through Git — this repo is that Git source.

**The rule is simple:**
- Want to deploy a new version? CI updates `deployment.yaml` here automatically.
- Want to change replicas? Edit `deployment.yaml` here and push.
- Want to roll back? Revert a commit here. ArgoCD does the rest.

This repo is written to by the CI pipeline in `gitops-cicd-pipeline` — a `github-actions[bot]`
commit appears here after every successful build, updating the Docker image tag to the new commit SHA.

---

## How It Fits Into the Pipeline

```
gitops-cicd-pipeline (app repo)          gitops-cicd-pipeline-config (this repo)
─────────────────────────────────        ────────────────────────────────────────
Developer pushes code                         
        │                                          
        ▼                                          
GitHub Actions CI runs                            
  - pytest tests                                  
  - Docker build + push to GHCR                   
  - Updates deployment.yaml  ──────────────►  k8s/deployment.yaml
    with new image SHA                            (image tag updated by bot)
                                                       │
                                                       ▼
                                              ArgoCD detects change
                                              Syncs cluster to match
                                                       │
                                                       ▼
                                              Rolling deployment
                                              on Kubernetes cluster
```

---

## Repository Structure

```
gitops-cicd-pipeline-config/
├── k8s/
│   ├── namespace.yaml      ← creates the myapp namespace
│   ├── deployment.yaml     ← 2 replicas, resource limits, health probes
│   └── service.yaml        ← ClusterIP service, port 80 → container 5000
└── argocd-app.yaml         ← ArgoCD Application manifest
```

---

## Kubernetes Manifests

### namespace.yaml
Creates an isolated namespace `myapp` inside the cluster.
All application resources (pods, services) live inside this namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

### deployment.yaml
The most important file. Defines how the app runs on the cluster.

Key settings:
- `replicas: 2` — two pods run at all times for availability
- `image: ghcr.io/irfanjat/gitops-cicd-pipeline:<SHA>` — updated automatically by CI
- `readinessProbe` — pod only receives traffic after `/health` returns 200
- `livenessProbe` — pod is restarted if `/health` fails after startup
- `resources.requests/limits` — prevents any single pod from consuming all node resources

### service.yaml
A `ClusterIP` service that load-balances traffic across the 2 running pods.
Maps port 80 on the service to port 5000 on the container.

### argocd-app.yaml
The ArgoCD Application manifest. Tells ArgoCD:
- **Where to look:** this repo, `k8s/` folder, HEAD of main branch
- **Where to deploy:** `https://kubernetes.default.svc`, namespace `myapp`
- **How to sync:** automated with `prune: true` and `selfHeal: true`

---

## ArgoCD Sync Policy Explained

| Setting | Value | What it does |
|---------|-------|-------------|
| `automated` | enabled | ArgoCD syncs without manual trigger |
| `prune` | true | Deletes cluster resources removed from Git |
| `selfHeal` | true | Reverts any manual `kubectl` changes back to Git state |
| `CreateNamespace` | true | Creates `myapp` namespace if it doesn't exist |

**selfHeal in practice:** If someone runs `kubectl scale deployment myapp --replicas=5`,
ArgoCD will detect the drift and revert to `replicas: 2` (as defined here) within minutes.
Git is always the authority — not the person with `kubectl` access.

---

## How Deployments Work

### Automatic (normal flow)
1. Developer pushes code to `gitops-cicd-pipeline`
2. CI tests pass, Docker image builds with new commit SHA
3. `github-actions[bot]` commits a new image tag to this repo:
   ```
   ci: update image tag to a2daf83faa2a509d72cb8a5b3a11ddb012ff20c7
   ```
4. ArgoCD detects the commit, applies the updated `deployment.yaml`
5. Kubernetes performs a rolling update — zero downtime

### Manual config change (e.g. scaling up)
```bash
# Edit deployment.yaml — change replicas: 2 to replicas: 3
git add k8s/deployment.yaml
git commit -m "chore: scale replicas to 3"
git push origin main
# ArgoCD syncs automatically — no kubectl needed
```

### Rollback
```bash
# Find the commit you want to roll back to
git log --oneline

# Revert the bad commit
git revert <commit-sha>
git push origin main
# ArgoCD syncs to the reverted state — previous image tag is restored
```

---

## Current Deployment Status

| Property | Value |
|----------|-------|
| Application | myapp — Flask API |
| Namespace | myapp |
| Replicas | 2 |
| Registry | ghcr.io/irfanjat/gitops-cicd-pipeline |
| Image tag | Updated automatically on every CI run |
| Cluster | kind (local) |
| ArgoCD version | v3.3.4 |

---

## Related Repository

| Repo | Purpose |
|------|---------|
| [gitops-cicd-pipeline](https://github.com/irfanjat/gitops-cicd-pipeline) | Application source code, Dockerfile, tests, GitHub Actions CI pipeline |
| **gitops-cicd-pipeline-config** (this repo) | Kubernetes manifests, ArgoCD config — cluster source of truth |

---

## Why a Separate Config Repo?

Most beginners put everything in one repo. Here is why that is wrong at scale:

1. **No unnecessary rebuilds** — changing replica count should not trigger a Docker build
2. **Clean audit log** — every change to the cluster is a Git commit. Who changed what, when, and why
3. **Separation of concerns** — developers own the app repo, platform team owns the config repo
4. **No CI loop** — if the config repo was the same as the app repo, the bot commit would trigger CI infinitely
5. **RBAC friendly** — you can give developers write access to the app repo without giving them cluster config access

---

## Author

**Irfan Jat** — CS student building production-grade DevOps infrastructure.

[![App Repo](https://img.shields.io/badge/App_Repo-gitops--cicd--pipeline-185FA5?logo=github)](https://github.com/irfanjat/gitops-cicd-pipeline)
[![GitHub](https://img.shields.io/badge/GitHub-irfanjat-181717?logo=github)](https://github.com/irfanjat)
