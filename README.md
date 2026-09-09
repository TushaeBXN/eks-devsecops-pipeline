# EKS DevSecOps Pipeline

![CI](https://github.com/TushaeBXN/eks-devsecops-pipeline/actions/workflows/devsecops.yml/badge.svg)
![Trivy](https://img.shields.io/badge/Security-Trivy%20Scanned-blue?logo=aqua)
![Checkov](https://img.shields.io/badge/IaC-Checkov%20Validated-brightgreen)
![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-orange?logo=argo)

A GitOps CI/CD pipeline that enforces security policy before anything reaches the cluster. Built on top of the [eks-terraform-project](https://github.com/TushaeBXN/eks-terraform-project) — the cluster is already provisioned; this repo owns the pipeline that deploys to it.

## How It Works

Every push to `main` triggers a four-stage pipeline. If any security stage fails, deployment stops.

```
push to main
     │
     ▼
┌─────────────┐
│  1. Build   │  Docker image built from app/
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  2. Trivy   │  Scans image for HIGH/CRITICAL CVEs — fails pipeline if found
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  3. Checkov │  Scans Terraform for misconfigs (open ports, missing encryption)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  4. Deploy  │  Pushes image to ghcr.io, ArgoCD syncs k8s/ to the cluster
└─────────────┘
```

## What This Demonstrates

| Capability | Detail |
|---|---|
| **GitOps** | ArgoCD watches this repo — every merge to `main` is automatically reflected on the cluster |
| **Container security** | Trivy blocks deployment on HIGH/CRITICAL CVEs before the image is ever pushed |
| **IaC policy enforcement** | Checkov validates Terraform against 1,000+ misconfiguration rules |
| **GitHub Container Registry** | Image built and published to `ghcr.io` — no external registry account needed |
| **Least-privilege IAM** | Pipeline authenticates via `GITHUB_TOKEN` scoped to this repo only |

## Project Structure

```
eks-devsecops-pipeline/
├── .github/
│   └── workflows/
│       └── devsecops.yml   # Full pipeline definition
├── app/
│   ├── Dockerfile          # nginx:alpine — minimal attack surface
│   └── index.html          # App served by the container
├── k8s/
│   ├── deployment.yaml     # 2-replica Deployment, resource limits set
│   └── service.yaml        # NodePort Service
└── argocd/
    └── application.yaml    # ArgoCD Application — auto-sync + self-heal
```

## Prerequisites

- [eks-terraform-project](https://github.com/TushaeBXN/eks-terraform-project) provisioned and running
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed on the cluster
- GitHub Actions secrets configured (see below)

## GitHub Actions Secrets Required

| Secret | Value |
|---|---|
| `ARGOCD_SERVER` | Hostname or IP of your ArgoCD server |
| `ARGOCD_AUTH_TOKEN` | ArgoCD API token for the `devsecops-demo` app |

`GITHUB_TOKEN` is provided automatically by GitHub Actions — no setup needed.

## Local Setup

**1. Install ArgoCD on your cluster:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**2. Apply the ArgoCD application manifest:**
```bash
kubectl apply -f argocd/application.yaml
```

**3. Get the ArgoCD admin password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**4. Port-forward the ArgoCD UI:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open `https://localhost:8080` — login with `admin` and the password above.

## Key Design Decisions

**Trivy with `ignore-unfixed: true`** — only fails on CVEs that have a fix available. Flagging unfixable CVEs just creates noise with no action item.

**Checkov on `soft_fail: true`** — reports findings without blocking the pipeline. In a production setup you'd harden this to `exit-code: 1` once the baseline is clean.

**ArgoCD `selfHeal: true`** — if someone manually changes something on the cluster, ArgoCD reverts it back to match the repo. The repo is the single source of truth.

## What I'd Add in Production

- **SBOM generation** (Syft) — a full software bill of materials published alongside every image
- **Secrets scanning** (Gitleaks) — block commits that accidentally include API keys or credentials
- **OPA/Gatekeeper policies** — enforce cluster-side rules (no privileged pods, required resource limits)
- **Slack notifications** — alert on pipeline failures and successful deploys
