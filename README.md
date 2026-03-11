# shared-actions

[![CI](https://github.com/albatross-core/shared-actions/actions/workflows/ci.yml/badge.svg)](https://github.com/albatross-core/shared-actions/actions/workflows/ci.yml)

Reusable GitHub Actions and workflows for Albatross CI/CD pipelines. Supports both GCP and AWS with a simple toggle.

## Composite Actions

### `build-and-push`

Docker build & push to GCP Artifact Registry or AWS ECR.

```yaml
- uses: albatross-core/shared-actions/build-and-push@main
  with:
    cloud: gcp              # or aws
    image-name: website
    environment: qa
    # GCP
    gcp-credentials-json: ${{ secrets.GCP_SA_KEY }}
    gcp-project-id: albatross-qa
    # AWS
    # aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    # aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    # aws-ecr-repository: albatross/website
    build-args: |
      VITE_PUBLIC_POSTHOG_KEY=${{ secrets.VITE_PUBLIC_POSTHOG_KEY }}
```

`GIT_SHA` and `GIT_VERSION` build args are always included automatically.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `cloud` | yes | | `gcp` or `aws` |
| `image-name` | yes | | Image name (e.g., `website`) |
| `environment` | yes | | Environment tag (`dev`, `qa`, `prod`) |
| `dockerfile` | no | `./Dockerfile` | Path to Dockerfile |
| `build-args` | no | | Extra build args (multiline `KEY=VALUE`) |
| `gcp-credentials-json` | no | | GCP service account key JSON |
| `gcp-project-id` | no | | GCP project ID |
| `gcp-ar-location` | no | `europe-west3` | Artifact Registry location |
| `gcp-ar-repository` | no | `albatross` | Artifact Registry repository |
| `aws-access-key-id` | no | | AWS access key ID |
| `aws-secret-access-key` | no | | AWS secret access key |
| `aws-region` | no | `eu-central-1` | AWS region |
| `aws-ecr-repository` | no | | ECR repository path |

### `update-gitops`

Checkout the gitops repo, update `image_tag` and `redeploy_ts` in service `.hcl` patches, commit and push.

```yaml
- uses: albatross-core/shared-actions/update-gitops@main
  with:
    environment: qa
    service-name: website
    image-tag: qa
    gitops-token: ${{ secrets.GITOPS_ACCESS_TOKEN }}
    cloud: gcp
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | yes | | Target environment (`dev`, `qa`, `prod`) |
| `service-name` | yes | | Service name matching the `.hcl` filename |
| `image-tag` | yes | | Image tag to set in the patch |
| `gitops-token` | yes | | GitHub token with gitops repo access |
| `cloud` | yes | | `gcp` or `aws` (determines path prefix) |
| `gitops-repo` | no | `albatross-core/gitops` | GitOps repository |
| `gitops-ref` | no | `main` | GitOps branch |

## Reusable Workflows

### `deploy.yml`

Full deploy pipeline: build + push + gitops update. Combines both composite actions above.

```yaml
jobs:
  deploy:
    uses: albatross-core/shared-actions/.github/workflows/deploy.yml@main
    with:
      cloud: gcp
      image-name: website
      environment: qa
      gcp-project-id: albatross-qa
      build-args: |
        VITE_PUBLIC_POSTHOG_KEY=${{ secrets.VITE_PUBLIC_POSTHOG_KEY }}
    secrets:
      GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
      GITOPS_ACCESS_TOKEN: ${{ secrets.GITOPS_ACCESS_TOKEN }}
```

### `create-deployment-record.yml`

Creates a GitHub deployment record for tracking.

```yaml
jobs:
  record:
    uses: albatross-core/shared-actions/.github/workflows/create-deployment-record.yml@main
    with:
      environment: qa
      sha: ${{ github.sha }}
      environment_url: https://qa.usealbatross.ai
```

### `wait-for-deploy.yml`

Polls a version endpoint until the deployed SHA matches the expected commit.

```yaml
jobs:
  wait:
    uses: albatross-core/shared-actions/.github/workflows/wait-for-deploy.yml@main
    with:
      environment: qa
      sha: ${{ github.sha }}
      version-url: https://qa-landing.usealbatross.ai/api/version
    secrets:
      INTEGRATION_TESTS_TOKEN: ${{ secrets.INTEGRATION_TESTS_TOKEN }}
```

### `yaml-lint.yml`

Self-contained YAML linting (no dependencies needed in the calling repo).

```yaml
jobs:
  lint:
    uses: albatross-core/shared-actions/.github/workflows/yaml-lint.yml@main
```

### `push-tag`

Increment the latest semver patch tag and push it. Replaces the repeated `actions/github-script` snippet.

```yaml
- uses: albatross-core/shared-actions/push-tag@v0.0.2
  with:
    github-token: ${{ secrets.INTEGRATION_TESTS_TOKEN }}
```

| Input | Required | Description |
|-------|----------|-------------|
| `github-token` | yes | GitHub token with permission to create tags |

## Full Example

A typical service deploy workflow goes from ~120 lines to ~30:

```yaml
name: Deploy QA

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: albatross-core/shared-actions/.github/workflows/deploy.yml@main
    with:
      cloud: gcp
      image-name: my-service
      environment: qa
      gcp-project-id: albatross-qa
    secrets:
      GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
      GITOPS_ACCESS_TOKEN: ${{ secrets.GITOPS_ACCESS_TOKEN }}

  wait:
    needs: deploy
    uses: albatross-core/shared-actions/.github/workflows/wait-for-deploy.yml@main
    with:
      environment: qa
      sha: ${{ github.sha }}
      version-url: https://qa.usealbatross.ai/api/version
    secrets:
      INTEGRATION_TESTS_TOKEN: ${{ secrets.INTEGRATION_TESTS_TOKEN }}

  record:
    needs: wait
    uses: albatross-core/shared-actions/.github/workflows/create-deployment-record.yml@main
    with:
      environment: qa
      sha: ${{ github.sha }}
      environment_url: https://qa.usealbatross.ai
```
