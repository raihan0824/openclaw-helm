# OpenClaw Configuration Guide

This guide covers configuring OpenClaw for Cloudeka, including channels, models, sessions, and security settings.

## Configuration Overview

OpenClaw configuration is stored in `openclaw.json` and managed via Helm ConfigMap:

```yaml
# charts/openclaw/values.yaml
app-template:
  configMaps:
    config:
      data:
        openclaw.json: |
          { ... }
```

## Config Mode: Merge vs Overwrite

| Mode | Behavior | Use Case |
|------|----------|----------|
| `merge` (default) | Helm values are deep-merged with existing config. Runtime changes (paired devices, UI settings) are preserved. | Development, manual testing |
| `overwrite` | Helm values completely replace existing config. | GitOps, strict infrastructure-as-code |

```yaml
app-template:
  configMode: merge  # or "overwrite"
```

### Important: ArgoCD with Merge Mode

If using ArgoCD with `configMode: merge`, prevent ArgoCD from overwriting runtime changes:

```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: openclaw
spec:
  ignoreDifferences:
    - group: ""
      kind: ConfigMap
      name: openclaw
      jsonPointers:
        - /data
```

## Gateway Configuration

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    // IMPORTANT: trustedProxies uses exact IP matching only
    // - CIDR notation is NOT supported
    // - List each proxy IP individually
    "trustedProxies": ["10.250.199.254"]
  }
}
```

### trustedProxies Explained

The `trustedProxies` setting controls which IPs are trusted for forwarding client connection information. This affects:
- Rate limiting
- Access control
- IP-based routing

**Common values for Cloudeka:**
- Ingress controller IP: `10.250.199.254`
- Gateway system IPs: Add each individually, no CIDR

## Browser Configuration (Chromium Sidecar)

The Chromium sidecar enables browser automation via CDP (Chrome DevTools Protocol):

```json
{
  "browser": {
    "enabled": true,
    "defaultProfile": "default",
    "profiles": {
      "default": {
        "cdpUrl": "http://localhost:9222",
        "color": "#4285F4"
      }
    }
  }
}
```

To disable browser automation, set `"enabled": false` and disable the sidecar in values.yaml:

```yaml
app-template:
  controllers:
    main:
      containers:
        chromium:
          enabled: false
```

## Agent Configuration

### Single Agent (Default)

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/node/.openclaw/workspace",
      "model": {
        "primary": "dekallm/zai/glm-4.7-fp8"
      },
      "userTimezone": "UTC",
      "timeoutSeconds": 600,
      "maxConcurrent": 1
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "identity": {
          "name": "OpenClaw",
          "emoji": "🦞"
        }
      }
    ]
  }
}
```

### Multi-Agent Configuration

For multiple agents with different personalities and permissions:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "workspace": "/home/node/.openclaw/workspace-main",
        "identity": { "name": "OpenClaw Main", "emoji": "🦞" }
      },
      {
        "id": "aira",
        "workspace": "/home/node/.openclaw/workspace-aira",
        "agentDir": "/home/node/.openclaw/agents/aira/agent",
        "identity": { "name": "Aira", "emoji": "🤖" },
        "model": { "primary": "dekallm/zai/glm-4.7-fp8" },
        "tools": {
          "allow": ["exec", "read", "write", "edit"],
          "deny": ["browser"]
        }
      }
    ]
  }
}
```

See [Multi-Agent Guide](./04-multi-agent.md) for complete setup.

## Model Providers

### Cloudeka DEKALLM (Default)

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "dekallm": {
        "baseUrl": "https://dekallm.cloudeka.ai/v1",
        "apiKey": "${DEKALLM_API_KEY}",
        "api": "openai-completions",
        "models": [
          { "id": "zai/glm-4.7-fp8", "name": "GLM 4.7" }
        ]
      }
    }
  }
}
```

### Adding Anthropic (Optional)

```json
{
  "models": {
    "providers": {
      "anthropic": {
        "baseUrl": "https://api.anthropic.com/v1",
        "apiKey": "${ANTHROPIC_API_KEY}",
        "api": "openai-completions",
        "models": [
          { "id": "claude-opus-4-6", "name": "Claude Opus 4.6" },
          { "id": "claude-sonnet-4-5", "name": "Claude Sonnet 4.5" }
        ]
      }
    }
  }
}
```

Add `ANTHROPIC_API_KEY` to your secrets.

## Session Management

```json
{
  "session": {
    "scope": "per-sender",        // or "global" for shared sessions
    "store": "/home/node/.openclaw/sessions",
    "reset": {
      "mode": "idle",             // or "manual", "never"
      "idleMinutes": 60           // Reset after 60min of inactivity
    }
  }
}
```

### Session Scopes

| Scope | Behavior |
|-------|----------|
| `per-sender` | Each user gets their own isolated session history |
| `global` | All users share one conversation (not recommended for multi-user) |

## Channel Configuration

> **Important:** Only enable channels you have configured. If a channel is enabled but its environment variable (token) is not set, OpenClaw will fail to start. Comment out any channels you are not using.

### Telegram

```json
{
  "channels": {
    "telegram": {
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "enabled": true
    }
  }
}
```

**Telegram Pairing Required:**

After enabling Telegram, you must pair the bot with OpenClaw:

1. Start a conversation with your Telegram bot
2. Send `/start` or any message to get a pairing code
3. Approve the pairing from within the pod:

```bash
kubectl exec -n openclaw deployment/openclaw -c main -- \
  node dist/index.js pairing approve telegram <PAIRING_CODE>
```

**Setup Steps:**
1. Create a Telegram bot via [@BotFather](https://t.me/botfather)
2. Add `TELEGRAM_BOT_TOKEN` to your secrets
3. Deploy OpenClaw
4. Get pairing code from Telegram and approve it

### Slack

```json
{
  "channels": {
    "slack": {
      "botToken": "${SLACK_BOT_TOKEN}",
      "appToken": "${SLACK_APP_TOKEN}",
      "enabled": true,
      "replyToMode": "all",       // or "replies" only
      "channels": {
        "C0AEWJDTED6": {
          "allow": true,
          "requireMention": false  // Set true for @mention only
        }
      }
    }
  }
}
```

**Only enable Slack if you have configured the tokens. Comment out the entire `slack` block if using Telegram only.**

#### Setting up Slack

1. Create a Slack App: https://api.slack.com/apps
2. Enable **Socket Mode**
3. Add OAuth Scopes: `chat:write`, `channels:history`, `groups:history`, `im:history`, `mpim:history`
4. Add `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` to secrets
5. Install app to your workspace

### Discord

```json
{
  "channels": {
    "discord": {
      "token": "${DISCORD_BOT_TOKEN}",
      "enabled": true
    }
  }
}
```

#### Setting up Discord

1. Create application: https://discord.com/developers/applications
2. Create bot user
3. Enable Privileged Intents: **Server Members Intent**, **Message Content Intent**
4. Add `DISCORD_BOT_TOKEN` to secrets
5. Invite bot with `applications.commands` and `bot` scopes

### Example: Using Only Telegram

If you only want Telegram (not Slack), comment out the Slack section:

```json
{
  "channels": {
    "telegram": {
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "enabled": true
    }
    // "slack": {
    //   "botToken": "${SLACK_BOT_TOKEN}",
    //   "appToken": "${SLACK_APP_TOKEN}",
    //   "enabled": true
    // }
  }
}
```

## Logging Configuration

```json
{
  "logging": {
    "level": "info",              // debug, info, warn, error
    "consoleLevel": "info",
    "consoleStyle": "compact",    // or "pretty"
    "redactSensitive": "tools"    // Redact sensitive data in tool outputs
  }
}
```

## Tools Configuration

```json
{
  "tools": {
    "profile": "full",            // or "minimal"
    "web": {
      "search": {
        "enabled": false          // Web search (requires API key)
      },
      "fetch": {
        "enabled": true           // HTTP fetch
      }
    }
  }
}
```

### Tool Profiles

| Profile | Available Tools |
|---------|-----------------|
| `full` | All tools enabled (exec, read, write, edit, browser, etc.) |
| `minimal` | Read-only tools, no execution or modification |

## Hooks Configuration

Enable webhooks for external integrations:

```json
{
  "hooks": {
    "enabled": true,
    "token": "${OPENCLAW_HOOKS_TOKEN}"
  }
}
```

Example webhook call:

```bash
curl -X POST http://openclaw:18789/hooks/myhook \
  -H "Authorization: Bearer ${OPENCLAW_HOOKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'
```

## Network Policy (Optional)

Enable network policy to restrict OpenClaw's network access:

```yaml
app-template:
  networkpolicies:
    main:
      enabled: true
```

Default policy allows:
- Ingress from `gateway-system` namespace
- DNS (kube-dns)
- Egress to public internet (blocks private RFC1918 ranges)

## Resource Limits

Adjust based on your workload:

```yaml
app-template:
  controllers:
    main:
      containers:
        main:
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
```

## Environment Variables

Additional environment variables can be added:

```yaml
app-template:
  controllers:
    main:
      containers:
        main:
          env:
            CUSTOM_VAR: "value"
            NODE_ENV: "production"
```

## Optional Persistence (Comment Out When Not Used)

Some persistence entries in `values.yaml` are commented out by default. **Leave them commented unless you have created the required secrets/configmaps**, otherwise OpenClaw will fail to start.

### Kubeconfig (for kubectl access)

```yaml
# Uncomment ONLY after creating the secret:
# kubectl create secret generic openclaw-kubeconfig \
#   --from-file=config=/path/to/kubeconfig \
#   -n openclaw

app-template:
  persistence:
    kubeconfig:
      enabled: true
      type: secret
      name: openclaw-kubeconfig
      advancedMounts:
        main:
          main:
            - path: /home/node/.kube/config
              subPath: config
              readOnly: true
```

### Custom Skills

```yaml
# Uncomment ONLY after creating the skill ConfigMap:
# kubectl create configmap my-skill \
#   --from-file=SKILL.md=./skills/my-skill/SKILL.md \
#   -n openclaw

app-template:
  persistence:
    my-skill:
      enabled: true
      type: configMap
      name: my-skill
      advancedMounts:
        main:
          main:
            - path: /home/node/.openclaw/workspace/skills/my-skill/SKILL.md
              subPath: SKILL.md
              readOnly: true
```

### MCP Server Config

```yaml
# Uncomment ONLY after creating the MCP config:
# kubectl create configmap my-mcporter-config \
#   --from-file=mcporter.json=./mcporter.json \
#   -n openclaw

app-template:
  persistence:
    my-mcporter-config:
      enabled: true
      type: configMap
      name: my-mcporter-config
      advancedMounts:
        main:
          main:
            - path: /home/node/.openclaw/workspace/config/mcporter.json
              subPath: mcporter.json
              readOnly: true
```

**Important:** Always create the secret/ConfigMap before uncommenting the persistence entry, or the pod will fail with "mount failed" errors.

## Next Steps

- [Skills Guide](./03-skills.md) - Add custom skills
- [Multi-Agent Guide](./04-multi-agent.md) - Configure multiple agents
