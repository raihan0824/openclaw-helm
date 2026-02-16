# OpenClaw on Cloudeka - Documentation

This folder contains comprehensive documentation for deploying and managing OpenClaw on Cloudeka Kubernetes infrastructure.

## Documentation Index

| Guide | Description |
|-------|-------------|
| [00. Prerequisites](./00-prerequisites.md) | Required tools, Cloudeka access, and setup |
| [01. Installation](./01-installation.md) | Deployment steps and initial setup |
| [02. Configuration](./02-configuration.md) | Channels, models, sessions, and settings |
| [03. Skills](./03-skills.md) | Adding single-file and folder-based skills |
| [04. Multi-Agent](./04-multi-agent.md) | Running multiple isolated agents |
| [05. Troubleshooting](./05-troubleshooting.md) | Common issues and solutions |

## Quick Start

```bash
# 1. Ensure prerequisites are met
# See: 00. Prerequisites

# 2. Create namespace
kubectl create namespace openclaw

# 3. Create secrets
kubectl create secret generic openclaw-env-secret -n openclaw \
  --from-literal=DEKALLM_API_KEY=xxx \
  --from-literal=OPENCLAW_GATEWAY_TOKEN=yyy

# 4. Install
helm install openclaw ./charts/openclaw -n openclaw -f charts/openclaw/values.yaml

# 5. Port forward
kubectl port-forward -n openclaw svc/openclaw 18789:18789
```

Open http://localhost:18789 and enter your gateway token.

## Architecture

![OpenClaw Architecture](./assets/architecture.png)

OpenClaw runs as a single-pod deployment with:
- **Main container**: OpenClaw Gateway (port 18789)
- **Chromium sidecar**: Headless browser for automation (port 9222)
- **Init containers**: Config merge and skill installation
- **PVC**: Persistent storage for workspace, sessions, and dependencies

## Multi-Agent Support

OpenClaw supports multiple isolated agents with:
- Separate workspaces and skills
- Per-agent tool restrictions
- Message routing via bindings

See [Multi-Agent Guide](./04-multi-agent.md) for setup.

## Cloudeka-Specific Configuration

| Setting | Value |
|---------|-------|
| Image Registry | `dekaregistry.cloudeka.id/cloudeka-system/openclaw` |
| LLM Provider | `https://dekallm.cloudeka.ai/v1` |
| Model | `dekallm/zai/glm-4.7-fp8` |

## External Resources

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [ClawHub Skills Repository](https://clawhub.com)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
