# kit-manage

A GitHub Actions plugin for managing [Gravwell](https://www.gravwell.io) kits using git.  It enables two workflows:

- **Pull** — sync a kit from a remote Gravwell instance into a git repository via a pull request
- **Push** — pack a locally checked-out kit and deploy it to a remote Gravwell instance

The action builds two Go tools during initialization:

| Tool | Source |
|------|--------|
| `kitctl` | `github.com/gravwell/gravwell/v3/kitctl` |
| `kitmanager` | `github.com/gravwell/gravwell/v3/tools/actions/kitmanager` |

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `operation` | **yes** | — | `pull` or `push` |
| `go-version` | no | `1.24` | Go version for building tools |
| `ignore-cert` | no | `false` | Skip TLS certificate verification |
| `kit-global` | no | `false` | Deploy with global access (push only) |
| `kit-write-global` | no | `false` | Deploy with global write access (push only) |
| `kit-groups` | no | `""` | Comma-separated groups for deployment (push only) |
| `kit-write-groups` | no | `""` | Comma-separated groups with write access (push only) |
| `kit-labels` | no | `""` | Comma-separated labels for the kit (push only) |

## Environment Variables

These **must** be set in the calling workflow's `env` block:

| Variable | Description |
|----------|-------------|
| `GRAVWELL_HOST` | URL of the Gravwell instance (e.g. `https://gravwell.example.com`) |
| `GRAVWELL_KIT_ID` | Kit build ID on the remote Gravwell system |
| `GRAVWELL_KIT_DIR` | Local directory containing (or receiving) the unpacked kit |
| `GRAVWELL_TOKEN` | API token for authentication (**use a repository secret**) |

## Usage

### Manual Pull

Sync a kit from a Gravwell instance into the repository. The action automatically creates or updates a `kit-manage-sync` branch (synced from `main`), commits changes there, and opens a pull request:

```yaml
name: 'Kit Pull'

on:
  workflow_dispatch:
    inputs:
      kit_dir:
        description: 'Directory in the repo to sync kit contents into'
        required: false
        default: 'kit'

permissions:
  contents: write
  pull-requests: write

jobs:
  pull:
    runs-on: ubuntu-latest
    env:
      GRAVWELL_HOST: ${{ vars.GRAVWELL_HOST }}
      GRAVWELL_TOKEN: ${{ secrets.GRAVWELL_TOKEN }}
      GRAVWELL_KIT_ID: ${{ vars.GRAVWELL_KIT_ID }}
      GRAVWELL_KIT_DIR: ${{ github.workspace }}/${{ inputs.kit_dir }}
    steps:
      - uses: actions/checkout@v4
      - uses: gravwell/kit-manage@v1
        with:
          operation: pull
```

### Manual Push

Push a locally checked-out kit to a Gravwell instance:

```yaml
name: 'Kit Push'

on:
  workflow_dispatch:
    inputs:
      kit_dir:
        description: 'Directory in the repo containing the kit'
        required: false
        default: 'kit'

jobs:
  push:
    runs-on: ubuntu-latest
    env:
      GRAVWELL_HOST: ${{ vars.GRAVWELL_HOST }}
      GRAVWELL_TOKEN: ${{ secrets.GRAVWELL_TOKEN }}
      GRAVWELL_KIT_ID: ${{ vars.GRAVWELL_KIT_ID }}
      GRAVWELL_KIT_DIR: ${{ github.workspace }}/${{ inputs.kit_dir }}
    steps:
      - uses: actions/checkout@v4
      - uses: gravwell/kit-manage@v1
        with:
          operation: push
```

### Push on Merge (⚠ Dangerous)

> **Warning:** This workflow automatically deploys kit changes to a live Gravwell instance every time a commit is merged to `main`.  Any merged change—including mistakes—will be immediately deployed to the target system.
>
> Before enabling this, ensure:
> - Branch protection rules require PR reviews before merging
> - The target Gravwell instance is **not** a production system, or you have a tested rollback strategy
> - All contributors understand that merging == deploying
>
> Consider using the manual push workflow instead for safer, human-gated deployments.
>
> **Note:** Pull operations create a pull request to `main` via the `kit-manage-sync` branch, so they will not directly trigger a Push on Merge workflow.

```yaml
name: 'Kit Push on Merge'

on:
  push:
    branches:
      - main
    paths:
      - 'kit/**'

jobs:
  push:
    runs-on: ubuntu-latest
    env:
      GRAVWELL_HOST: ${{ vars.GRAVWELL_HOST }}
      GRAVWELL_TOKEN: ${{ secrets.GRAVWELL_TOKEN }}
      GRAVWELL_KIT_ID: ${{ vars.GRAVWELL_KIT_ID }}
      GRAVWELL_KIT_DIR: ${{ github.workspace }}/kit
    steps:
      - uses: actions/checkout@v4
      - uses: gravwell/kit-manage@v1
        with:
          operation: push
```

## Repository Setup

1. Go to your repository **Settings → Secrets and variables → Actions**
2. Add a **secret** named `GRAVWELL_TOKEN` with your Gravwell API token
3. Add **variables** for `GRAVWELL_HOST` and `GRAVWELL_KIT_ID`
4. Copy one of the example workflows from `examples/` into your own repository

## License

BSD 2-Clause. See [LICENSE](LICENSE).
