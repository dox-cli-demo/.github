# 🚀 DOX GitOps + CI/CD Platform Setup

This repository documents the complete setup of a GitOps + GitHub Actions-based CI/CD platform using **Argo CD**, **custom GitHub runners**, and **templated pipelines**, powered by the DOX framework.

---

## 🧩 Architecture Overview

```plaintext
GitHub Repositories
├── .github                  → Org-level workflows that delegate to reusable templates
├── dox-github-actions      → Centralized GitHub Action workflows (e.g., Java, Node, Docker)
├── dox-gitops-runners      → Argo CD App definitions to deploy GitHub self-hosted runners
├── dox-gitops-deployments  → GitOps apps and Helm chart deployments (Dev/PreProd/Prod)
├── dox-devops-base         → Builds Helm charts and manages build workflows
```

---

## 📋 Prerequisites

* A Kubernetes cluster (EKS, GKE, AKS, etc.)
* `kubectl` configured with cluster access
* GitHub Organization (e.g., `dox-cli-demo`)
* DockerHub or other OCI registry account
* `helm` and `base64` installed locally

---

## 🧱 STEP 1: Install Argo CD (Non-HA)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.1.0-rc1/manifests/install.yaml
```

📖 Docs: [https://argo-cd.readthedocs.io/en/stable/](https://argo-cd.readthedocs.io/en/stable/)
📦 Releases: [https://github.com/argoproj/argo-cd/releases](https://github.com/argoproj/argo-cd/releases)

---

## 🔑 STEP 2: Retrieve Argo CD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

👉 To access the UI locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then open: https://localhost:8080
```

---

## 🔐 STEP 3: Create `dox-runners` Secret

```bash
kubectl create ns dox-runners

kubectl create secret generic dox-runner-secret \
  -n dox-runners \
  --from-literal=debug_mode=true \
  --from-literal=github_token=ghp_xxxxxx \
  --from-literal=docker_image_push_prefix=devstackhub \
  --from-literal=gitops_repo=github.com/dox-cli-demo/dox-gitops-deployments.git \
  --from-literal=helm_oci_url=oci://docker.io/devstackhub \
  --from-literal=oci_reg_user=devstackhub \
  --from-literal=oci_reg_password=dckr_pat_xxxxx
```

---

## ⚙️ STEP 4: Allow GitHub Runner Group Access

Go to the following link and **allow public repositories** for the runner group:

🔗 [https://github.com/organizations/dox-cli-demo/settings/actions/runner-groups/1](https://github.com/organizations/dox-cli-demo/settings/actions/runner-groups/1)

---

## 🏃 STEP 5: Deploy GitHub Runners via Argo CD

```yaml
# argo-apps/dox-gitops-runners.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dox-gitops-runners
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  source:
    path: argo-apps
    repoURL: https://github.com/dox-cli-demo/dox-gitops-runners.git
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

🧩 Runner Images:
[https://github.com/actions/runner/pkgs/container/actions-runner](https://github.com/actions/runner/pkgs/container/actions-runner)

---

## 🧪 STEP 6: Generate Helm Chart Using `dox-devops-base`

Make sure:

* `dox-devops-base` builds a Helm chart (`Chart.yaml`, `values.yaml`, templates)
* Helm chart is pushed to DockerHub as OCI using `helm push`

---

## 🚀 STEP 7: Deploy Apps with GitOps

```yaml
# argo-apps/dox-gitops-deployments-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dox-gitops-deployments-dev
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  source:
    path: argo-apps
    repoURL: https://github.com/dox-cli-demo/dox-gitops-deployments.git
    targetRevision: HEAD
    helm:
      valueFiles:
        - values-dev.yaml
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 🧬 STEP 8: CI/CD via `.github` + `dox-github-actions`

The **`.github`** repo in the GitHub organization contains simplified **templated workflows** that **delegate to `dox-github-actions`**.

### Example: `.github` → `workflows/java-cicd.yml`

```yaml
name: DOX Java CICD

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  call-dox-java-cicd:
    uses: dox-cli-demo/dox-github-actions/.github/workflows/dox-java-cicd.yml@main
    # with:
    #   jdk_version: 17
```

✅ Any repository in the org can now use this workflow without duplicating logic.

---

## 📦 Repositories Overview

| Repository                                                                         | Description                                                                                     |
| ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| [`dox-cli-demo/.github`](https://github.com/dox-cli-demo/.github)                  | Org-level reusable GitHub Actions templates that delegate to workflows in `dox-github-actions`. |
| [`dox-github-actions`](https://github.com/dox-cli-demo/dox-github-actions)         | Centralized and reusable workflows like `dox-java-cicd.yml`, `docker-build.yml`, etc.           |
| [`dox-devops-base`](https://github.com/dox-cli-demo/dox-devops-base)               | Contains Helm chart generators and base CI/CD logic for building and publishing.                |
| [`dox-gitops-runners`](https://github.com/dox-cli-demo/dox-gitops-runners)         | ArgoCD apps to manage and sync GitHub Actions self-hosted runners.                              |
| [`dox-gitops-deployments`](https://github.com/dox-cli-demo/dox-gitops-deployments) | GitOps app manifests (Dev/PreProd/Prod) for deploying applications with Helm.                   |

---

## 📌 Notes

* All tokens must be scoped for `repo`, `workflow`, and `packages:write`
* `argocd` syncs runners and apps periodically
* Helm charts follow OCI standard (`oci://docker.io/devstackhub/app-name`)
* CI/CD logic is DRY and centrally maintained via `.github` + `dox-github-actions`

---

## ✅ TODO

* [ ] Add `preprod` and `prod` `values-*.yaml` to `dox-gitops-deployments`
* [ ] Enable monitoring via Argo CD Notifications or Prometheus
* [ ] Add Helm chart signing & provenance verification

---

## 📬 Support

Raise an issue in the respective repository or reach out via internal DevOps Slack channel.

---

Let me know if you'd like this split into `docs/` with diagrams and folder examples or a full scaffolded GitHub Actions + Argo CD setup in a zip.

