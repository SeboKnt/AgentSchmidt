# AgentSchmidt - Cloud Deploy + Cloud Run

This repository deploys three services to **Google Cloud Run** via **Cloud Deploy**:

| Service        | Image                                          | Port | Purpose                                     |
|----------------|------------------------------------------------|------|---------------------------------------------|
| `freellmapi`   | `ghcr.io/tashfeenahmed/freellmapi:latest`      | 3001 | LLM provider gateway (unified API proxy)    |
| `hermes`       | `nousresearch/hermes-agent:latest`             | 8642 | AI agent that calls the gateway             |
| `dashboard`    | `nousresearch/hermes-agent:latest`             | 9119 | Localhost-only dashboard (SSH tunnel)       |

Both use `minInstances=0` (scale to zero) for zero baseline cost.

## How It Works

1. **Push to `main`** → Cloud Build trigger fires
2. **Cloud Deploy** applies the pipeline & target, then creates a release
3. **Cloud Run** rolls out both services with secrets from Secret Manager
4. **IAM** grants your Google account + Hermes SA invoker access

## Prerequisites

- GCP project with billing enabled
- APIs enabled: `run.googleapis.com`, `cloudbuild.googleapis.com`, `clouddeploy.googleapis.com`

## First-Time Setup

1. **Create Cloud Build trigger in Console:**
   - Event: Push to branch (main)
   - Included files: **
   - Configuration: cloudbuild.yaml (root)


## Repository Structure

```
.
├── cloudbuild.yaml          # Cloud Build trigger handler
├── clouddeploy.yaml         # DeliveryPipeline + Target definition
├── skaffold.yaml            # Local development config
├── services/
│   ├── freellmapi.yaml      # LLM gateway (port 3001, GCS volume)
│   ├── hermes.yaml          # AI agent (port 8642, GCS volume)
│   └── dashboard.yaml       # Dashboard (localhost:9119, GCS volume)
└── README.md
```

## Cost Control

- `minInstances=0` → scales to zero when idle (€0 baseline)
- `maxInstances=1` per service → caps concurrent cost
- Only Cloud Run invocation costs when actively used
- And maybe the Bucket
- And the vault
