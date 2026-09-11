# EKS DevSecOps Pipeline

[![CI](https://github.com/TushaeBXN/eks-devsecops-pipeline/actions/workflows/devsecops.yml/badge.svg)](https://github.com/TushaeBXN/eks-devsecops-pipeline/actions/workflows/devsecops.yml)

A GitOps CI/CD pipeline that enforces security policy before anything reaches the cluster. Built on top of the [eks-terraform-project](https://github.com/TushaeBXN/eks-terraform-project) — the cluster is already provisioned; this repo owns the pipeline that deploys to it.

## How It Works

Every push to `main` triggers a four-stage pipeline. If any security stage fails, deployment stops.

```
push to main
     |
     v
1. Build   - Docker image built from app/
     |
     v
2. Trivy   - Scans image for HIGH/CRITICAL CVEs, fails pipeline if found
     |
     v
3. Checkov - Scans Terraform for misconfigs (open ports, missing encryption)
     |
     v
4. Deploy  - Pushes image to ghcr.io, ArgoCD syncs k8s/ to the cluster
```

## What This Demonstrates

| Capability | Detail |
| --- | --- |
| **GitOps** | ArgoCD watches this repo, every merge to `main` is automatically reflected on the cluster |
| **Container security** | Trivy blocks deployment on HIGH/CRITICAL CVEs before the image is ever pushed |
| **IaC policy enforcement** | Checkov validates Terraform against 1,000+ misconfiguration rules |
| **GitHub Container Registry** | Image built and published to `ghcr.io`, no external registry account needed |
| **Least-privilege IAM** | Pipeline authenticates via `GITHUB_TOKEN` scoped to this repo only |

## Project Structure

```
eks-devsecops-pipeline/
|-- .github/
|   `-- workflows/
|       `-- devsecops.yml   # Full pipeline definition
|-- app/
|   |-- Dockerfile          # nginx:alpine, patched at build time
|   `-- index.html          # App served by the container
|-- k8s/
|   |-- deployment.yaml     # 2-replica Deployment, resource limits set
|   `-- service.yaml        # NodePort Service
`-- argocd/
    `-- application.yaml    # ArgoCD Application, auto-sync + self-heal
```

## Prerequisites

- [eks-terraform-project](https://github.com/TushaeBXN/eks-terraform-project) provisioned and running
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed on the cluster
- GitHub Actions secrets configured (see below)

## GitHub Actions Secrets Required

| Secret | Value |
| --- | --- |
| `ARGOCD_SERVER` | Hostname or IP of your ArgoCD server |
| `ARGOCD_AUTH_TOKEN` | ArgoCD API token for the `devsecops-demo` app |

`GITHUB_TOKEN` is provided automatically by GitHub Actions, no setup needed.

## Local Setup

**1. Install ArgoCD on your cluster:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts
```

(Server-side apply avoids a known annotation-size limit on one of ArgoCD's CRDs, see Problems Encountered below.)

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

Open `https://localhost:8080`, login with `admin` and the password above.

## Key Design Decisions

**Trivy with `ignore-unfixed: true`** - only fails on CVEs that have a fix available. Flagging unfixable CVEs just creates noise with no action item.

**Checkov on `soft_fail: true`** - reports findings without blocking the pipeline. In a production setup you'd harden this to `exit-code: 1` once the baseline is clean.

**ArgoCD `selfHeal: true`** - if someone manually changes something on the cluster, ArgoCD reverts it back to match the repo. The repo is the single source of truth.

## Problems Encountered & Fixed

Real issues hit while building and running this pipeline, not hypothetical "gotchas" - things that actually broke.

### 1. Trivy caught real CVEs and blocked the first build

The first real push to `main` triggered the pipeline as designed. Trivy scanned the built image (`nginx:1.27-alpine`) and found **36 vulnerabilities with available fixes - 2 CRITICAL, 34 HIGH** - including two CRITICAL OpenSSL heap buffer overflow CVEs (CVE-2026-31789) in `libssl3` and `libcrypto3`. Because the pipeline enforces `exit-code: "1"` on HIGH/CRITICAL findings, the build failed and the image was never pushed.

```
Total: 36 (HIGH: 34, CRITICAL: 2)
libssl3    CVE-2026-31789  CRITICAL  Fixed: 3.3.7-r0
libcrypto3 CVE-2026-31789  CRITICAL  Fixed: 3.3.7-r0
c-ares     CVE-2026-33630  HIGH      Fixed: 1.34.8-r0
... (33 more)
Error: Process completed with exit code 1.
```

**Fix:** the base Alpine packages hadn't been patched since the image tag was last built. Adding one line to the Dockerfile resolved it - patching packages at build time instead of floating to an unpinned `latest` tag:

```dockerfile
FROM nginx:1.27-alpine
RUN apk update && apk upgrade --no-cache
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

The next push rebuilt clean, Trivy passed, and the image reached `ghcr.io`. A pipeline that never fails hasn't proven it can catch anything - this one did, exactly as designed.

### 2. ArgoCD's CRD exceeded Kubernetes' annotation size limit

Installing ArgoCD with a plain `kubectl apply` failed on one CRD:

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

This happens because `kubectl apply` stores the previous config as an annotation for diffing, and ArgoCD's `applicationsets.argoproj.io` CRD is large enough to exceed that limit on install.

**Fix:** installed with server-side apply instead, which does not hit the same annotation-size ceiling:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts
```

### 3. The `deploy` job can't reach a local ArgoCD instance

The pipeline's final job runs `argocd app sync` from GitHub's own runners, authenticating against `ARGOCD_SERVER`. That works cleanly against a real, publicly reachable EKS cluster - but a `kind` cluster running on a local machine at `127.0.0.1` is not reachable from GitHub's infrastructure. The `deploy` job fails at that step by design in this local setup, while the `build-and-scan` job - the part that actually matters for security enforcement - runs and passes independently of where the cluster lives.

This is a real architectural boundary, not a bug: in production, this job would target a real ArgoCD instance with a public (or VPN/peered) endpoint. Locally, the same GitOps sync was verified manually by applying `argocd/application.yaml` directly against the `kind` cluster and confirming ArgoCD picked up and deployed the change.

## What I'd Add in Production

- **SBOM generation** (Syft) - a full software bill of materials published alongside every image
- **Secrets scanning** (Gitleaks) - block commits that accidentally include API keys or credentials
- **OPA/Gatekeeper policies** - enforce cluster-side rules (no privileged pods, required resource limits)
- **Slack notifications** - alert on pipeline failures and successful deploys
- **Automated base-image patching** (Dependabot or Renovate) - catch the next CVE before a push does
