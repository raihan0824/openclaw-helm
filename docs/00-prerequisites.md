# Prerequisites

Before deploying OpenClaw on Cloudeka, ensure you have the following tools and access set up.

## Required Tools

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **kubectl** | >= 1.26.0 | Kubernetes CLI | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) |
| **helm** | >= 3.0 | Package manager | [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) |
| **git** | Any | Clone repository | [git-scm.com](https://git-scm.com/) |

### Verify Installation

```bash
kubectl version --client
helm version
git --version
```

## Cloudeka Requirements

### Cloudeka Account

You need access to [Cloudeka AI Platform](https://ai.cloudeka.id/) with:
- Access to a Kubernetes cluster
- Ability to create namespaces and secrets

### Download Kubeconfig

Get your kubeconfig from the Cloudeka platform:

**[Download Kubeconfig Guide](https://docs.cloudeka.ai/starter-guide/download-kubeconfig)**

1. Log in to [Cloudeka AI](https://ai.cloudeka.id/)
2. Navigate to your cluster
3. Download the kubeconfig file
4. Test connectivity:

```bash
# Point kubectl to your downloaded kubeconfig
export KUBECONFIG=/path/to/your-kubeconfig.yaml

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### Container Registry Access

OpenClaw uses images from `dekaregistry.cloudeka.id`. Ensure you have:
- Network access to the registry
- Valid authentication (usually automatic within Cloudeka clusters)

## Required Information

Before proceeding to installation, have the following ready:

| Item | Description | Source |
|------|-------------|--------|
| **Namespace** | Target namespace for deployment | `openclaw` |
| **DEKALLM_API_KEY** | Cloudeka LLM API key | [Create LLM Key Guide](https://docs.cloudeka.ai/deka-llm/create-a-new-llm) |
| **Gateway Token** | Random token for device pairing | Generate a secure random string |
| **Slack Tokens** | Bot token + App token (if using Slack) | From Slack App settings |
| **Telegram Token** | Bot token (if using Telegram) | From @BotFather |

## Optional: Local Testing Tools

| Tool | Purpose |
|------|---------|
| **k9s** | Terminal-based Kubernetes UI (optional but helpful) |
| **stern** | Multi-pod log tailing (useful for debugging) |

## Next Steps

Once prerequisites are met, proceed to:

→ [01. Installation Guide](./01-installation.md)
