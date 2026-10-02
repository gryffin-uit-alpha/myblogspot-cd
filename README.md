# MyBlogSpot Application CD GitOps Repository

> **Repository 3 in the Standard 3-Repo Production Model:**
> 1. `MyBlogSpot`: Application Codebase + CI Pipeline (Docker build, test, push image).
> 2. `cluster-bootstrap` (`config`): Cluster Addons (StorageClass, Metrics Server, Kyverno, Prometheus, Loki).
> 3. **`myblogspot-cd` (This Repo)**: Application Delivery, Helm Charts, Environment Values, and ArgoCD Workload Manifests.

---

## 1. Directory Structure

```text
.
├── apps/                               # ArgoCD Application manifests for environments
│   ├── app-staging.yaml                # Deploy MyBlogSpot to 'staging' namespace
│   └── app-prod.yaml                   # Deploy MyBlogSpot to 'production' namespace
│
├── helm/
│   └── myblogspot/                     # Helm chart packaging backend, frontend, ingress
│       ├── Chart.yaml
│       ├── values.yaml                 # Base values
│       ├── values-staging.yaml         # Staging overrides (1 replica, staging domain)
│       ├── values-prod.yaml            # Production overrides (multi-replica, prod domain)
│       └── templates/
│           ├── _helpers.tpl
│           ├── backend-deployment.yaml
│           ├── backend-service.yaml
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           ├── ingress.yaml
│           ├── networkpolicy.yaml
│           └── secrets.yaml
│
├── legacy/                             # Preserved initial raw manifests for reference
└── README.md
```

---

## 2. GitOps Continuous Delivery Workflow

```text
[Developer]
    │
    ▼ git push
[Repo 1: MyBlogSpot (Codebase & CI)]
    │
    ├── 1. Run Unit Tests & Lint
    ├── 2. Build Docker Images (Backend & Frontend)
    ├── 3. Trivy Vulnerability Scan
    ├── 4. Push Images to Amazon ECR
    ▼
    └── 5. Auto-commit new image tag to Repo 3 (This CD Repo):
           git checkout main
           sed -i 's/tag: .*/tag: <new-sha>/' helm/myblogspot/values-staging.yaml
           git commit -m "chore(cd): update staging image tag to <new-sha>"
           git push
                   │
                   ▼
[Repo 3: myblogspot-cd (This Repo)] ─── Webhook / Polling ───► [ArgoCD Controller]
                                                                        │
                                                                        ▼
                                                             Deploy to EKS Cluster
                                                             (namespace: staging)
```

---

## 3. Quickstart: Deploying Workloads to EKS

### Step 1: Set Remote Git Repository
```bash
git remote add origin git@github.com:<YOUR_USERNAME>/myblogspot-cd.git
git branch -M main
git push -u origin main
```

### Step 2: Update the repoURL in App Manifests
```bash
sed -i 's|https://github.com/gryffin-uit-alpha/myblogspot-cd.git|git@github.com:<YOUR_USERNAME>/myblogspot-cd.git|g' apps/*.yaml
```

### Step 3: Register the Application into ArgoCD
To deploy the Staging environment:
```bash
kubectl apply -f apps/app-staging.yaml
```

To deploy the Production environment:
```bash
kubectl apply -f apps/app-prod.yaml
```
ArgoCD will continuously monitor this repository and keep the running workloads synchronized with Git!
