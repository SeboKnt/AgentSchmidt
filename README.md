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
├── config/
│   └── config.yaml          # Hermes configuration
└── README.md
```

## How to Use Hermes on Cloud Run

### 1. Google Chat Bot via Pub/Sub
Hermes listens to incoming events via the Google Cloud Pub/Sub pull subscription `agentschmidt-hermes-inbound-sub`.
- Google Chat API sends incoming user events to `projects/project-ecf5ff03-f941-4f63-a23/topics/agentschmidt-hermes-inbound`.
- Hermes processes messages and replies directly in the Google Chat space or 1:1 conversation.
- Allowed users configured: `sebastian.alxr@gmail.com`.

### 2. OpenAI-Compatible API Endpoint (Alternative)
Cloud Run exposes the Hermes API Server on port `8642` with `API_SERVER_ENABLED=true`.

You can interact with Hermes directly via `curl` or any OpenAI client:
```bash
SERVICE_URL=$(gcloud run services describe agentschmidt-hermes --region=europe-west1 --format='value(status.url)')
API_KEY=$(gcloud secrets versions access latest --secret=agentschmidt-hermes-api-server-key)

curl -X POST "$SERVICE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "Hello Hermes, run a quick status check!"}
    ]
  }'
```

### 2. Using Pub/Sub or Google Chat (Optional)
If you want asynchronous triggers via Google Cloud Pub/Sub:
- **Cloud Run Push Subscription**: Create a Cloud Run push subscription targeting `$SERVICE_URL/v1/chat/completions` (or a webhook handler).
- **Google Chat Bot**: Hermes has a built-in `google_chat` adapter that uses Cloud Pub/Sub pull subscriptions for Google Chat spaces and direct messages (configured via `GOOGLE_CHAT_PROJECT_ID` and `GOOGLE_CHAT_SUBSCRIPTION_NAME`).

## Cost Control

- `minInstances=0` → scales to zero when idle (€0 baseline)
- `maxInstances=1` per service → caps concurrent cost
- Only Cloud Run invocation costs when actively used
- And maybe the Bucket
- And the vault
