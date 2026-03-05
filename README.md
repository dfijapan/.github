# dfijapan/.github

Shared GitHub Actions workflows and organization-level configuration.

## Reusable Workflows

### `build-and-push.yml` — Build and Push Docker Image to ACR

Builds a Docker image and pushes it to Azure Container Registry with automatic tagging.

#### Tagging strategy

| Branch | Tag format | Example |
|--------|-----------|---------|
| `master` (prod) | Semver via conventional commits | `v1.2.3` |
| `develop` (dev) | Short git hash | `a1b2c3d` |
| Pull request | `pr-` prefixed hash (not pushed by default) | `pr-a1b2c3d` |

#### Minimal usage

```yaml
name: Build docker

on:
  workflow_dispatch:
  pull_request:
    paths-ignore: ['k8s/**']
  push:
    branches: [master, develop]
    paths-ignore: ['k8s/**']

jobs:
  build:
    uses: dfijapan/.github/.github/workflows/build-and-push.yml@master
    with:
      image_name: my-app
    secrets: inherit
```

#### All options

```yaml
jobs:
  build:
    uses: dfijapan/.github/.github/workflows/build-and-push.yml@master
    with:
      image_name: my-app              # Required: image name in ACR
      dockerfile: docker/Dockerfile   # Default: Dockerfile
      context: .                      # Default: .
      build_args: |                   # Default: ""
        BP_VERSION=abc1234
      enable_cache: true              # Default: false (GHA Docker layer cache)
      push_pr_images: false           # Default: false (skip push on PRs)
      semver_tagging: true            # Default: true (v-prefixed semver on prod)
      prod_branch: master             # Default: master
    secrets: inherit
```

#### With deploy key (private dependencies)

For repos that need SSH access during build (e.g. private Go/Python modules):

```yaml
jobs:
  build:
    uses: dfijapan/.github/.github/workflows/build-and-push.yml@master
    with:
      image_name: callcenter
      enable_cache: true
    secrets:
      AZURE_CONTAINER_REGISTRY_SP_CLIENT_ID_DEV: ${{ secrets.AZURE_CONTAINER_REGISTRY_SP_CLIENT_ID_DEV }}
      AZURE_CONTAINER_REGISTRY_SP_CLIENT_SECRET_DEV: ${{ secrets.AZURE_CONTAINER_REGISTRY_SP_CLIENT_SECRET_DEV }}
      AZURE_CONTAINER_REGISTRY_SP_CLIENT_ID_PROD: ${{ secrets.AZURE_CONTAINER_REGISTRY_SP_CLIENT_ID_PROD }}
      AZURE_CONTAINER_REGISTRY_SP_CLIENT_SECRET_PROD: ${{ secrets.AZURE_CONTAINER_REGISTRY_SP_CLIENT_SECRET_PROD }}
      DEPLOY_KEY: ${{ secrets.DFIUTILS_DEPLOY_KEY }}
```

The deploy key is written to `/tmp/deploy_key` and available as a Docker build secret:

```dockerfile
RUN --mount=type=secret,id=deploy_key,dst=/root/.ssh/id_ed25519 \
    pip install git+ssh://git@github.com/dfijapan/dfiutils.git
```

#### Outputs

| Output | Description |
|--------|-------------|
| `image_tag` | The tag that was pushed (e.g. `v1.2.3` or `a1b2c3d`) |
| `image_url` | Full image URL (e.g. `dfijapandev.azurecr.io/my-app:a1b2c3d`) |

#### Prerequisites

The workflow requires these organization-level resources:

**Secrets** (already configured):
- `AZURE_CONTAINER_REGISTRY_SP_CLIENT_ID_DEV` / `_PROD`
- `AZURE_CONTAINER_REGISTRY_SP_CLIENT_SECRET_DEV` / `_PROD`

**Variables** (already configured):
- `AZURE_CONTAINER_REGISTRY_NAME_DEV` — ACR name for dev (e.g. `dfijapandev`)
- `AZURE_CONTAINER_REGISTRY_NAME_PROD` — ACR name for prod (e.g. `dfijapanprod`)
