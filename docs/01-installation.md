# OpenClaw Installation on Cloudeka

This guide covers deploying OpenClaw on Cloudeka Kubernetes infrastructure using the customized Helm chart.

## What is OpenClaw?

OpenClaw is an AI assistant gateway that connects to messaging platforms (Slack, Telegram, Discord) and executes tasks autonomously. It can:

- Chat via Slack, Telegram, Discord
- Execute shell commands and kubectl operations
- Query databases via MCP (Model Context Protocol) servers
- Run custom skills for specialized tasks
- Support multiple isolated agents with different personalities

## Architecture Overview

![OpenClaw Architecture](./assets/architecture.png)

## Prerequisites

| Requirement | Version/Details |
|-------------|-----------------|
| Kubernetes | >= 1.26.0-0 |
| Helm | 3.0+ |
| Namespace | `openclaw` (create if not exists) |
| Container Registry | `dekaregistry.cloudeka.id` |

### Required Secrets

You'll need to prepare these secrets:

| Secret Key | Description | Required |
|------------|-------------|----------|
| `DEKALLM_API_KEY` | Cloudeka LLM API key | **Yes** |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token | **Yes** |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token | If using Telegram |
| `SLACK_BOT_TOKEN` | Slack bot token | If using Slack |
| `SLACK_APP_TOKEN` | Slack app token | If using Slack |
| `OPENCLAW_HOOKS_TOKEN` | Webhook auth token | Optional |

> **Important:** Only add secrets for channels you are using. If a channel is enabled in config but its token is not set, OpenClaw will fail to start. Comment out unused channels in `openclaw.json`.

### Cloudeka-Specific Configuration

- **Image Registry**: `dekaregistry.cloudeka.id/cloudeka-system/openclaw`
- **LLM Provider**: `https://dekallm.cloudeka.ai/v1`
- **Model**: `dekallm/zai/glm-4.7-fp8` (GLM 4.7 FP8)

## Installation Steps

### Step 1: Clone the Repository

```bash
git clone https://github.com/raihan0824/openclaw-helm.git
cd openclaw-helm
```

### Step 2: Create Namespace

```bash
kubectl create namespace openclaw
```

### Step 3: Create Environment Secrets

Create a file `secrets.yaml` with your values (DO NOT commit this):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openclaw-env-secret
  namespace: openclaw
type: Opaque
stringData:
  DEKALLM_API_KEY: "your-dekallm-api-key"
  OPENCLAW_GATEWAY_TOKEN: "your-random-gateway-token"
  TELEGRAM_BOT_TOKEN: "your-telegram-bot-token"
  SLACK_BOT_TOKEN: "xoxb-your-slack-bot-token"
  SLACK_APP_TOKEN: "xapp-1-your-slack-app-token"
  OPENCLAW_HOOKS_TOKEN: "your-random-hooks-token"
```

Apply it:

```bash
kubectl apply -f secrets.yaml
```

Or create directly via kubectl:

```bash
kubectl create secret generic openclaw-env-secret -n openclaw \
  --from-literal=DEKALLM_API_KEY=sk-xxx \
  --from-literal=OPENCLAW_GATEWAY_TOKEN=yyy \
  --from-literal=TELEGRAM_BOT_TOKEN=zzz
```

### Step 4: Create Kubeconfig Secret (for kubectl access)

```bash
kubectl create secret generic openclaw-kubeconfig -n openclaw \
  --from-file=config=/path/to/your/kubeconfig.yaml
```

This enables OpenClaw to run kubectl commands against your cluster.

### Step 5: Configure values.yaml

Review and modify `charts/openclaw/values.yaml` for your environment:

```yaml
app-template:
  openclawVersion: "2026.2.6"

  controllers:
    main:
      containers:
        main:
          # Reference your secret
          envFrom:
            - secretRef:
                name: openclaw-env-secret

  # Update trustedProxies with your ingress/gateway IPs
  configMaps:
    config:
      data:
        openclaw.json: |
          {
            "gateway": {
              "trustedProxies": ["10.250.199.254"]  # Your gateway IP
            },
            ...
          }
```

### Step 6: Install the Chart

```bash
helm install openclaw ./charts/openclaw -n openclaw -f charts/openclaw/values.yaml
```

### Step 7: Verify Deployment

```bash
# Check pod status
kubectl get pods -n openclaw

# View logs
kubectl logs -n openclaw deployment/openclaw -f

# Port forward to access gateway
kubectl port-forward -n openclaw svc/openclaw 18789:18789
```

Expected output:
```
NAME                       READY   STATUS    RESTARTS   AGE
openclaw-xxx               3/3     Running   0          1m
```

### Step 8: Pair Your Device

1. Open http://localhost:18789 in your browser
2. Enter your `OPENCLAW_GATEWAY_TOKEN`
3. Click "Connect"

Then approve the pairing request:

```bash
# List pending requests
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices list

# Approve (replace <REQUEST_ID>)
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices approve <REQUEST_ID>
```

### Step 9: Pair Telegram (If Using Telegram)

**Important:** If you enabled Telegram channel, you must complete pairing after deployment:

1. Start a chat with your Telegram bot in the Telegram app
2. Send `/start` or any message to the bot
3. The bot will reply with a pairing code (e.g., `ABCD1234`)

Approve the pairing from within the pod:

```bash
kubectl exec -n openclaw deployment/openclaw -c main -- \
  node dist/index.js pairing approve telegram <PAIRING_CODE>
```

Example:
```bash
kubectl exec -n openclaw deployment/openclaw -c main -- \
  node dist/index.js pairing approve telegram PML9NL9U
```

After pairing, the bot will respond to your messages in Telegram.

## Upgrading

```bash
helm upgrade openclaw ./charts/openclaw -n openclaw -f charts/openclaw/values.yaml
```

## Uninstalling

```bash
# Remove the deployment
helm uninstall openclaw -n openclaw

# Optionally delete the PVC (WARNING: deletes all data)
kubectl delete pvc -n openclaw -l app.kubernetes.io/name=openclaw

# Delete secrets
kubectl delete secret openclaw-env-secret openclaw-kubeconfig -n openclaw
```

## Post-Installation

After installation, see:
- [Configuration Guide](./02-configuration.md) - Channels, models, and settings
- [Skills Guide](./03-skills.md) - Adding custom skills
- [Multi-Agent Guide](./04-multi-agent.md) - Running multiple isolated agents

## Troubleshooting

See [Troubleshooting](./05-troubleshooting.md) for common issues.
