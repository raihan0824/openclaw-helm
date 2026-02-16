#  OpenClaw Helm Chart for Cloudeka

[![Helm 3](https://img.shields.io/badge/Helm-3.0+-0f1689?logo=helm&logoColor=white)](https://helm.sh/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.26+-326ce5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Helm chart for deploying OpenClaw on Cloudeka Kubernetes infrastructure — an AI assistant that connects to messaging platforms and executes tasks autonomously.

> **[Documentation](./docs/)** — Full installation guide, multi-agent setup, skills, and troubleshooting

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/raihan0824/openclaw-helm.git
cd openclaw-helm

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

---

## Features

| Feature | Description |
|---------|-------------|
| **Cloudeka DEKALLM** | Pre-configured for `dekallm/zai/glm-4.7-fp8` model |
| **Multi-Agent** | Run multiple isolated agents with different personalities |
| **Channel Support** | Slack, Telegram, Discord |
| **kubectl Access** | Execute kubectl commands from chat |
| **Skills System** | Extend with custom skills via ClawHub or local files |
| **Browser Automation** | Headless Chromium sidecar for web automation |
| **Hooks** | Webhook support for external integrations |

---

## Documentation

| Guide | Description |
|-------|-------------|
| [00. Prerequisites](./docs/00-prerequisites.md) | Required tools, Cloudeka access, kubeconfig |
| [01. Installation](./docs/01-installation.md) | Deployment steps and initial setup |
| [02. Configuration](./docs/02-configuration.md) | Channels, models, sessions, and settings |
| [03. Skills](./docs/03-skills.md) | Adding single-file and folder-based skills |
| [04. Multi-Agent](./docs/04-multi-agent.md) | Running multiple isolated agents |
| [05. Troubleshooting](./docs/05-troubleshooting.md) | Common issues and solutions |

---

## Cloudeka-Specific Configuration

This chart is customized for Cloudeka:

| Setting | Value |
|---------|-------|
| **Image Registry** | `dekaregistry.cloudeka.id/cloudeka-system/openclaw` |
| **LLM Provider** | `https://dekallm.cloudeka.ai/v1` |
| **Model** | `dekallm/zai/glm-4.7-fp8` (GLM 4.7 FP8) |

---

## Requirements

| Requirement | Version/Details |
|-------------|-----------------|
| Kubernetes | >= 1.26.0-0 |
| Helm | 3.0+ |
| Cloudeka Account | Access to [Cloudeka AI Platform](https://ai.cloudeka.id/) |
| DEKALLM_API_KEY | [Create LLM Key](https://docs.cloudeka.ai/deka-llm/create-a-new-llm) |

---

## Architecture

OpenClaw runs as a single-instance deployment (cannot scale horizontally):

| Component | Port | Description |
|-----------|------|-------------|
| Gateway | 18789 | Main HTTP/WebSocket interface |
| Chromium | 9222 | Headless browser for automation (CDP, optional) |
| Init containers | - | Config merge and skill installation |

**App Version:** 2026.2.6

---

## Configuration

All configuration is done via `values.yaml`. See [values.yaml](./charts/openclaw/values.yaml) for full reference.

Key settings:
- **Channels**: Slack, Telegram, Discord (via env variables)
- **Models**: DEKALLM GLM 4.7 FP8
- **Persistence**: 5Gi PVC for workspace, sessions, skills
- **Browser**: Headless Chromium sidecar (optional)

For detailed configuration guide, see [02. Configuration](./docs/02-configuration.md).

---

## Troubleshooting

```bash
# Check pod status
kubectl get pods -n openclaw

# View logs
kubectl logs -n openclaw deployment/openclaw

# Port forward to local
kubectl port-forward -n openclaw svc/openclaw 18789:18789
```

For more troubleshooting help, see [05. Troubleshooting](./docs/05-troubleshooting.md).

---

## Development

```bash
helm lint charts/openclaw
helm dependency update charts/openclaw
helm template test charts/openclaw --debug
```

---

## Chart Dependencies

| Repository | Name | Version |
|------------|------|---------|
| https://bjw-s-labs.github.io/helm-charts/ | app-template | 4.6.2 |

---

## External Resources

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [ClawHub Skills Repository](https://clawhub.com)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)

---

## License

MIT
