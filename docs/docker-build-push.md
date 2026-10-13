# Docker Build & Push - `docker-build-push.yml`

## Trigger

| Event | Detail |
|---|---|
| `push` → `main` | builds and pushes; tagged `latest` + `sha-<short-sha>` |
| `push` → tag `v*.*.*` (e.g. `v1.2.3`) | builds and pushes; tagged `1.2.3`, `1.2`, `1`, `sha-<short-sha>` |

Tag generation is handled by `docker/metadata-action@v5`, so release tags follow semver automatically.

## Required secrets & variables

| Name | Type | Description |
|---|---|---|
| *(none required)* | - | Auth to GHCR uses the built-in `GITHUB_TOKEN` (`packages: write`) |
| `DOCKERFILE_PATH` | variable | Path to the Dockerfile (default: `Dockerfile`) |
| `BUILD_CONTEXT` | variable | Docker build context (default: `.`) |

## Flow

```
push to main  OR  tag v*.*.*
        │
        ▼
┌──────────────────────────┐
│      build job           │
├──────────────────────────┤
│ 1. Checkout              │
│ 2. Set up Docker Buildx  │── multi-arch capable builder
│ 3. Log in to GHCR        │── docker/login-action with GITHUB_TOKEN
│ 4. metadata-action       │── computes tags (latest/sha-*/semver)
│ 5. Build (load: true)    │── builds locally, NOT pushed yet
│ 6. Trivy scan            │── fails on HIGH/CRITICAL vulns
│        │                 │   (ignore-unfixed: true to reduce noise)
│        ▼ (scan passed)   │
│ 7. Build + push to GHCR  │── pushed only if the scan is clean
│    cache: gha            │── GitHub Actions cache for fast rebuilds
└──────────────────────────┘
```

## Notes

- The image is built **twice in one job** (once with `load: true` for the scanner, once with `push: true`) so a vulnerable image can never reach the registry. Buildx layer caching (`type=gha`) keeps the second build fast.
- Trivy (`aquasecurity/trivy-action`) runs with `exit-code: '1'` and `severity: 'HIGH,CRITICAL'` - the run fails before any push happens. `ignore-unfixed: true` avoids failing on CVEs with no available fix.
- Permissions are least-privilege: `contents: read` (checkout), `packages: write` (push to GHCR).
- `concurrency: group: docker-${{ github.ref }}` with `cancel-in-progress: true` cancels stale builds on rapid pushes - safe here because nothing is deployed directly by this workflow.
