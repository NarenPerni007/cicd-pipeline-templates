# Terraform CI/CD — `terraform.yml`

## Trigger

| Event | Branches / paths | What happens |
|---|---|---|
| `pull_request` → `main` | changes under `terraform/**` or to this workflow file | fmt check → init → validate → plan; the plan is posted as a PR comment |
| `push` → `main` | same path filter | plan runs again, then `terraform apply` — gated behind the `production` environment |

The path filter keeps unrelated PRs (docs, app code) from burning Terraform runs.

## Required secrets & variables

| Name | Type | Description |
|---|---|---|
| `AWS_ROLE_TO_ASSUME` | secret | IAM role ARN that GitHub assumes via OIDC (e.g. `arn:aws:iam::123456789012:role/github-terraform`) |
| `AWS_REGION` | variable | AWS region, e.g. `us-east-1` (default: `us-east-1`) |
| `TF_WORKING_DIR` | variable | Path to the Terraform root module (default: `terraform`) |
| `TF_VERSION` | variable | Terraform version to install (default: `1.9.8`) |

Protect the `production` environment with **required reviewers** (Settings → Environments → production) to get manual approval before every apply.

## Flow

```
PR opened / updated (terraform/** changed)
        │
        ▼
┌─────────────────────┐
│  terraform-plan job │
├─────────────────────┤
│ 1. fmt -check       │── fails on unformatted code
│ 2. init             │
│ 3. validate         │
│ 4. plan             │── continue-on-error, captured for comment
│ 5. PR comment       │── posts fmt/init/validate/plan summary + full plan
└─────────────────────┘
        │
        ▼ (merge to main)
┌─────────────────────┐
│  terraform-plan job │── runs again on the merged commit
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ terraform-apply job │
│ environment:        │
│   production        │── pauses for manual approval if reviewers are set
├─────────────────────┤
│ 1. init             │
│ 2. apply            │── auto-approve (approval already happened via environment)
└─────────────────────┘
```

## Notes

- Authentication is **OIDC** (`aws-actions/configure-aws-credentials@v4` with `role-to-assume`). No static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` secrets exist anywhere.
- `concurrency: group: terraform-${{ github.ref }}` with `cancel-in-progress: false` serializes runs per branch without killing an in-flight apply — important because cancelling an apply mid-run can corrupt state.
- Permissions are least-privilege: `contents: read` (checkout), `pull-requests: write` (post plan comment), `id-token: write` (OIDC).
