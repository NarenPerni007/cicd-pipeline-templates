# Deploy to EKS — `k8s-deploy.yml`

## Trigger

| Event | Detail |
|---|---|
| `workflow_run` — "Docker Build & Push" completes on `main` | automatic continuous delivery; deploys the image tag the docker build just pushed |
| `workflow_dispatch` | manual run; choose `image_tag` (e.g. `sha-a1b2c3d` or `1.2.3`) and `environment` (`production`/`staging`) — useful for rollbacks or promoting a specific build |

The automatic trigger only proceeds when the docker workflow **concluded with success** (`github.event.workflow_run.conclusion == 'success'`), so a failed scan or build never deploys.

## Required secrets & variables

| Name | Type | Description |
|---|---|---|
| `AWS_ROLE_TO_ASSUME` | secret | IAM role ARN that GitHub assumes via OIDC |
| `AWS_REGION` | variable | AWS region of the cluster, e.g. `us-east-1` |
| `EKS_CLUSTER_NAME` | variable | Name of the EKS cluster |
| `K8S_NAMESPACE` | variable | Target namespace (default: `default`) |
| `KUSTOMIZE_DIR` | variable | Path to the kustomize overlay/base (default: `k8s/overlays/prod`) |

## Flow

```
"Docker Build & Push" succeeds on main
        │                    OR manual dispatch (image_tag + environment)
        ▼
┌─────────────────────────────────┐
│          deploy job             │
│  environment: production        │── manual approval if reviewers are set
├─────────────────────────────────┤
│ 1. Configure AWS creds (OIDC)   │
│ 2. aws eks update-kubeconfig    │── kubectl context for the cluster
│ 3. Resolve image tag            │── from input, or sha-<short-sha> of the
│                                 │   commit that triggered the docker build
│ 4. kustomize edit set image     │── pins the new image in the overlay
│ 5. kubectl apply -k             │
│ 6. rollout status (300s timeout)│── blocks until the Deployment is ready
│ 7. kubectl get pods             │── final health check
└─────────────────────────────────┘
```

## Notes

- Image tagging is the contract between this workflow and `docker-build-push.yml`: the docker workflow pushes `sha-<short-sha>`, and this workflow resolves the same tag from the triggering run's commit SHA. They stay in sync without hardcoding.
- `concurrency: group: deploy-<environment>` with `cancel-in-progress: false` serializes deployments per environment — two revisions can never deploy simultaneously.
- The kustomize overlay is expected to define a `kustomization.yaml` whose `images` entry is named `app`; `kustomize edit set image` updates it in place before `kubectl apply -k`.
- Authentication to AWS is **OIDC** — no static keys. The role needs `eks:DescribeCluster` (for `update-kubeconfig`) plus the EKS cluster's `aws-auth`/access-entry mapping for the deploying identity.
- Permissions are least-privilege: `contents: read` (checkout), `id-token: write` (OIDC).
