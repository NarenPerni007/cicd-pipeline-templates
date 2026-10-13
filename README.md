# CI/CD Pipeline Templates

Reusable GitHub Actions workflows for **Terraform**, **Docker**, and **Kubernetes** deployments — the same patterns I use to ship infrastructure and containerized apps on AWS.

No static AWS keys anywhere: every workflow authenticates to AWS with **OIDC** (`aws-actions/configure-aws-credentials`), and every workflow declares **least-privilege permissions**.

## Workflows

| Workflow | Trigger | What it does |
|---|---|---|
| [`terraform.yml`](.github/workflows/terraform.yml) | PR → `main`: fmt/init/validate/plan (plan posted as PR comment) · Push → `main`: plan + `apply` behind the `production` environment | Full Terraform CI/CD |
| [`docker-build-push.yml`](.github/workflows/docker-build-push.yml) | Push → `main`, tags `v*.*.*` | Buildx build → Trivy scan (fails on HIGH/CRITICAL) → push to GHCR with `latest` / `sha-*` / semver tags |
| [`k8s-deploy.yml`](.github/workflows/k8s-deploy.yml) | Auto: after "Docker Build & Push" succeeds · Manual: `workflow_dispatch` (pick image tag + environment) | OIDC → EKS kubeconfig → kustomize set image → `kubectl apply` → `rollout status` |

Detailed docs per workflow (triggers, flow diagrams, notes): [`docs/`](docs/).

## Required secrets & variables

Set these at repo level (Settings → Secrets and variables → Actions), or per-environment for `production`.

| Name | Type | Used by | Description |
|---|---|---|---|
| `AWS_ROLE_TO_ASSUME` | 🔑 secret | terraform, k8s-deploy | IAM role ARN GitHub assumes via OIDC, e.g. `arn:aws:iam::123456789012:role/github-actions` |
| `AWS_REGION` | 📦 variable | terraform, k8s-deploy | AWS region, e.g. `us-east-1` |
| `EKS_CLUSTER_NAME` | 📦 variable | k8s-deploy | Name of the EKS cluster |
| `TF_WORKING_DIR` | 📦 variable | terraform | Path to Terraform root module (default: `terraform`) |
| `TF_VERSION` | 📦 variable | terraform | Terraform version (default: `1.16.5`) |
| `K8S_NAMESPACE` | 📦 variable | k8s-deploy | Target namespace (default: `default`) |
| `KUSTOMIZE_DIR` | 📦 variable | k8s-deploy | Path to kustomize overlay (default: `k8s/overlays/prod`) |
| `DOCKERFILE_PATH` | 📦 variable | docker-build-push | Path to Dockerfile (default: `Dockerfile`) |

## OIDC setup (AWS side) — 5 minutes

1. **Create an OIDC provider** in IAM for `token.actions.githubusercontent.com` (audience `sts.amazonaws.com`) — one-time per AWS account.
2. **Create an IAM role** (e.g. `github-actions`) with the policies your workflows need (Terraform: your infra policies; EKS deploy: `eks:DescribeCluster` plus cluster access via access entries).
3. **Set the trust policy** so only your repo can assume it:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
         "StringLike": { "token.actions.githubusercontent.com:sub": "repo:<YOUR_GITHUB_USERNAME>/*:ref:refs/heads/main" }
       }
     }]
   }
   ```
4. **Store the role ARN** as the `AWS_ROLE_TO_ASSUME` secret in the repo (Settings → Secrets and variables → Actions).
5. (Optional) **Protect the `production` environment** with required reviewers (Settings → Environments) to get manual approval before `terraform apply` and production deploys.

## How to reuse

Copy the `.github/workflows/` directory into your own repository, then:

1. Add the secrets/variables from the table above.
2. Adjust `TF_WORKING_DIR` / `KUSTOMIZE_DIR` to match your repo layout.
3. For EKS: grant the IAM role access to the cluster (EKS access entries or `aws-auth` ConfigMap).
4. For Terraform: point `TF_WORKING_DIR` at a real root module (or drop in one from my [`terraform-aws-eks`](https://github.com/NarenPerni007/terraform-aws-eks) module repo).

## Repo layout

```
.
├── .github/workflows/
│   ├── terraform.yml          # Terraform plan on PR, apply on merge (production env)
│   ├── docker-build-push.yml  # Buildx + Trivy scan + push to GHCR
│   └── k8s-deploy.yml         # Deploy to EKS (auto after docker build, or manual)
├── docs/
│   ├── terraform.md
│   ├── docker-build-push.md
│   └── k8s-deploy.md
└── README.md
```
