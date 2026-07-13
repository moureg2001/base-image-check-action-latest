# Base Image Check Action

Checks if Docker base images have been updated (same tag, new SHA) without modifying the Dockerfile.

## Features

- **Dynamic FROM detection** — automatically handles any number of stages (single-stage, multi-stage, 3+ stages)
- **Private registry support** — optional login before pulling the app image
- **Webhook notifications** — sends a message (RocketChat, Slack, etc.) when a new base image is detected
- **Dockerfile auto-detection** — set `dockerfile: auto` to find it automatically

## How it works

1. Reads previous base image digests from labels of the current app image
2. Parses ALL `FROM` lines from the Dockerfile
3. Queries the registry for current digests (metadata only, no layer download)
4. Compares previous vs. current — outputs `changed=true` if any digest differs
5. Generates a summary table on the workflow run page
6. Optionally sends a webhook notification

## Quick Start

```yaml
- uses: ADEBAR-DevOps/base-image-check-action@v1
  id: check
  with:
    app-image: ghcr.io/my-org/my-app

- name: Rebuild
  if: steps.check.outputs.changed == 'true'
  run: docker build --pull -t my-app:latest .
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `app-image` | ✅ | — | Full app image path (for reading previous labels) |
| `app-image-tag` | ❌ | `latest` | Tag to read previous labels from |
| `dockerfile` | ❌ | `Dockerfile` | Path to Dockerfile, or `auto` for auto-detection |
| `registry` | ❌ | — | Registry to login to before pulling app image |
| `registry-username` | ❌ | — | Registry username |
| `registry-password` | ❌ | — | Registry password |
| `notify-webhook` | ❌ | — | Webhook URL (only notifies on change) |
| `notify-title` | ❌ | `Base Image Update Detected` | Notification title |

## Outputs

| Output | Description |
|---|---|
| `changed` | `true` if any digest changed |
| `stages` | JSON array with all stage details |
| `stage-count` | Number of FROM stages found |
| `dockerfile-path` | Resolved Dockerfile path |

### Stages JSON Format

```json
[
  {
    "index": 0,
    "name": "builder",
    "image": "golang:1.24-alpine",
    "old_digest": "sha256:aaa...",
    "new_digest": "sha256:aaa...",
    "status": "unchanged"
  },
  {
    "index": 1,
    "name": "runtime",
    "image": "alpine:3.21",
    "old_digest": "sha256:bbb...",
    "new_digest": "sha256:ccc...",
    "status": "new"
  }
]
```

Stage names are auto-assigned:
- **2 stages**: `builder` (first), `runtime` (last)
- **1 stage**: `main`
- **3+ stages**: `builder`, `stage-1`, `stage-2`, ..., `runtime`

## Labels

For the memory to work, your build must write labels into the app image.
The label names follow the pattern `base.<stage-name>.digest`:

```yaml
- uses: docker/build-push-action@v5
  with:
    labels: |
      base.builder.digest=${{ steps.check.outputs.new-builder-digest }}
      base.runtime.digest=${{ steps.check.outputs.new-runtime-digest }}
```

Or dynamically from the stages JSON:

```yaml
- name: Generate labels
  id: labels
  run: |
    LABELS=$(echo '${{ steps.check.outputs.stages }}' | jq -r '.[] | "base.\(.name).digest=\(.new_digest)"' | tr '\n' ',')
    echo "value=${LABELS%,}" >> $GITHUB_OUTPUT

- uses: docker/build-push-action@v5
  with:
    labels: ${{ steps.labels.outputs.value }}
```

## Examples

### Simple (public registry)

```yaml
- uses: ADEBAR-DevOps/base-image-check-action@v1
  id: check
  with:
    app-image: ghcr.io/my-org/my-app
```

### Private registry with login

```yaml
- uses: ADEBAR-DevOps/base-image-check-action@v1
  id: check
  with:
    app-image: registry.company.com/my-app
    registry: registry.company.com
    registry-username: ${{ secrets.REG_USER }}
    registry-password: ${{ secrets.REG_PASS }}
```

### With webhook notification

```yaml
- uses: ADEBAR-DevOps/base-image-check-action@v1
  id: check
  with:
    app-image: ghcr.io/my-org/my-app
    notify-webhook: ${{ secrets.ROCKETCHAT_WEBHOOK }}
    notify-title: 'Base image update in my-app'
```

### Auto-detect Dockerfile

```yaml
- uses: ADEBAR-DevOps/base-image-check-action@v1
  id: check
  with:
    app-image: ghcr.io/my-org/my-app
    dockerfile: auto
```

### 3+ stage Dockerfile

```dockerfile
FROM node:20-alpine AS frontend
FROM golang:1.24-alpine AS backend
FROM alpine:3.21
```

Summary will show:

| # | Stage | Image | Status |
|---|---|---|---|
| 0 | builder | `node:20-alpine` | unchanged |
| 1 | stage-1 | `golang:1.24-alpine` | **NEW** |
| 2 | runtime | `alpine:3.21` | unchanged |
