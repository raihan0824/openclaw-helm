# OpenClaw Skills Guide

This guide covers adding custom skills to OpenClaw on Cloudeka, including both simple single-file skills and complex nested folder structures.

## What are Skills?

Skills extend OpenClaw's capabilities with specialized tools and knowledge. A skill can:
- Define new tools via MCP (Model Context Protocol) servers
- Add specialized knowledge and instructions
- Provide helper scripts and utilities

Skills are stored in an agent's `workspace/skills/` directory.

## Skill Structure

### Minimal Skill (Single File)

```
my-skill/
└── SKILL.md
```

### Complex Skill (Nested)

```
my-skill/
├── SKILL.md
├── lib/
│   └── helpers.ts
├── templates/
│   └── prompt.md
└── config.json
```

**Important:** Kubernetes ConfigMaps are flat — they don't support nested directories. We'll handle this differently based on skill complexity.

## Method 1: Single-File Skills (Simple)

For skills with only a `SKILL.md` file, use a direct ConfigMap mount.

### Step 1: Create the Skill File

Create `my-skill/SKILL.md`:

```markdown
---
name: my-skill
description: A brief description of what this skill does
metadata: {"openclaw":{"requires":{"bins":[]}}}
---

# My Skill

Describe what the skill does and how to use it.

## Usage

Example usage here.
```

### Step 2: Create ConfigMap

```bash
kubectl create configmap my-skill \
  --from-file=SKILL.md=./my-skill/SKILL.md \
  -n openclaw
```

### Step 3: Add to values.yaml

```yaml
# charts/openclaw/values.yaml
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

### Step 4: Upgrade Helm Release

```bash
helm upgrade openclaw ./charts/openclaw -n openclaw -f charts/openclaw/values.yaml
```

## Method 2: Folder-Based Skills (Tar Archive)

For skills with nested directories, use a tar archive approach.

### Step 1: Prepare Your Skill Folder

```
my-advanced-skill/
├── SKILL.md
├── lib/
│   └── utils.ts
├── templates/
│   └── prompt.md
└── config.json
```

### Step 2: Create Tar Archive

```bash
# Create tar.gz from the skill folder
tar czf /tmp/my-advanced-skill.tar.gz -C ./skills-folder my-advanced-skill

# Verify contents
tar tzf /tmp/my-advanced-skill.tar.gz
# Expected output:
# my-advanced-skill/
# my-advanced-skill/SKILL.md
# my-advanced-skill/lib/utils.ts
# my-advanced-skill/templates/prompt.md
# my-advanced-skill/config.json
```

### Step 3: Create ConfigMap from Tar

```bash
kubectl create configmap my-advanced-skill-skill \
  --from-file=skill.tar.gz=/tmp/my-advanced-skill.tar.gz \
  -n openclaw
```

### Step 4: Add to values.yaml

Mount to `init-skills` container for extraction:

```yaml
# charts/openclaw/values.yaml
app-template:
  persistence:
    my-advanced-skill-skill:
      enabled: true
      type: configMap
      name: my-advanced-skill-skill
      advancedMounts:
        main:
          init-skills:
            - path: /tmp/skills/my-advanced-skill
              readOnly: true
```

### Step 5: Update init-skills Command

Add extraction logic to `init-skills` command in values.yaml:

```yaml
# charts/openclaw/values.yaml
app-template:
  controllers:
    main:
      initContainers:
        init-skills:
          command:
            - sh
            - -c
            - |
              log() { echo "[$(date -Iseconds)] [init-skills] $*"; }

              log "Starting skills initialization"

              # ============================================================
              # Runtime Dependencies
              # ============================================================
              mkdir -p /home/node/.openclaw/bin
              if [ ! -f /home/node/.openclaw/bin/uv ]; then
                log "Installing uv..."
                curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR=/home/node/.openclaw/bin sh
              fi

              mkdir -p /home/node/.openclaw/node_modules/.bin
              if [ ! -f /home/node/.openclaw/node_modules/.bin/mcporter ]; then
                log "Installing mcporter..."
                cd /home/node/.openclaw && npm install mcporter
              fi

              # ============================================================
              # Extract Skill Archives
              # ============================================================
              # Extract tar.gz skill archives to workspace
              for archive in /tmp/skills/*/skill.tar.gz; do
                if [ -f "$archive" ]; then
                  log "Extracting skill archive: $archive"
                  tar xzf "$archive" -C /home/node/.openclaw/workspace/skills/
                fi
              done

              # ============================================================
              # Install ClawHub Skills
              # ============================================================
              mkdir -p /home/node/.openclaw/workspace/skills
              cd /home/node/.openclaw/workspace
              for skill in weather; do
                if [ -n "$skill" ] && [ ! -d "skills/${skill##*/}" ]; then
                  log "Installing skill: $skill"
                  if ! npx -y clawhub install "$skill" --no-input; then
                    log "WARNING: Failed to install skill: $skill"
                  fi
                else
                  log "Skill already installed: $skill"
                fi
              done

              log "Skills initialization complete"
```

### Step 6: Upgrade Helm Release

```bash
helm upgrade openclaw ./charts/openclaw -n openclaw -f charts/openclaw/values.yaml
```

## Updating Skills

### Single-File Skills

Update the ConfigMap and restart:

```bash
# Delete and recreate
kubectl delete configmap my-skill -n openclaw
kubectl create configmap my-skill \
  --from-file=SKILL.md=./my-skill/SKILL.md \
  -n openclaw

# Restart pod
kubectl rollout restart deployment openclaw -n openclaw
```

### Folder-Based Skills

Same process — recreate ConfigMap and restart:

```bash
# Recreate tar
tar czf /tmp/my-advanced-skill.tar.gz -C ./skills-folder my-advanced-skill

# Update ConfigMap
kubectl delete configmap my-advanced-skill-skill -n openclaw
kubectl create configmap my-advanced-skill-skill \
  --from-file=skill.tar.gz=/tmp/my-advanced-skill.tar.gz \
  -n openclaw

# Restart pod
kubectl rollout restart deployment openclaw -n openclaw
```

## Per-Agent Skills

In a multi-agent setup, each agent can have its own set of skills:

```yaml
# Skill for "main" agent
main-weather-skill:
  enabled: true
  type: configMap
  name: weather-skill
  advancedMounts:
    main:
      main:
        - path: /home/node/.openclaw/workspace-main/skills/weather/SKILL.md
          subPath: SKILL.md
          readOnly: true

# Skill for "docs" agent only
docs-wiki-skill:
  enabled: true
  type: configMap
  name: docs-wiki-skill
  advancedMounts:
    main:
      main:
        - path: /home/node/.openclaw/workspace-docs/skills/wiki/SKILL.md
          subPath: SKILL.md
          readOnly: true
```

See [Multi-Agent Guide](./04-multi-agent.md) for complete multi-agent setup.

## MCP Server Integration

Skills can integrate MCP servers for external tools:

### Example: Database Query Skill

1. **Create `SKILL.md`**:

```markdown
---
name: my-db-skill
description: Query incident database
metadata: {"openclaw":{"requires":{"bins":["mcporter"]},"primaryEnv":"DB_URL"}}
---

# Database Query Skill

You have access to the incident database via `mcporter`.

## Available Tools

### query_tickets
Query tickets by filters.

```bash
mcporter call my-db.query_tickets status:open priority:critical
```
```

2. **Create MCP config**:

```json
{
  "mcpServers": {
    "my-db": {
      "description": "Incident database",
      "baseUrl": "http://mcp-server:3000/mcp"
    }
  }
}
```

3. **Create ConfigMap for MCP config**:

```bash
kubectl create configmap my-db-mcporter-config \
  --from-file=mcporter.json=./mcporter.json \
  -n openclaw
```

4. **Mount to agent workspace**:

```yaml
my-db-mcporter-config:
  enabled: true
  type: configMap
  name: my-db-mcporter-config
  advancedMounts:
    main:
      main:
        - path: /home/node/.openclaw/workspace/config/mcporter.json
          subPath: mcporter.json
          readOnly: true
```

5. **Add environment variable**:

```bash
kubectl patch secret openclaw-env-secret -n openclaw \
  --type='merge' \
  -p='{"stringData":{"DB_URL": "postgresql://user:pass@host:5432/db"}}'
```

## Skill Best Practices

1. **Keep skills focused** — One skill should do one thing well
2. **Use metadata** — Declare dependencies and environment variables
3. **Document tools** — Provide clear examples for each tool
4. **Version control** — Store skill definitions in git
5. **Test locally** — Validate skill behavior before deploying

## Common Issues

| Issue | Solution |
|-------|----------|
| Skill not found | Check mount path matches workspace |
| Nested files missing | Use tar archive method |
| Permission denied | Ensure PVC is writable (init-skills has access) |
| MCP server unreachable | Verify baseUrl and network policy |

## Example Skills Repository

Structure for managing multiple skills:

```
openclaw-skills/
├── skills/
│   ├── weather/
│   │   └── SKILL.md
│   ├── aira-db/
│   │   ├── SKILL.md
│   │   ├── lib/
│   │   │   └── db.ts
│   │   └── templates/
│   │       └── query.md
│   └── kubectl/
│       └── SKILL.md
├── deploy.sh
└── README.md
```

## Next Steps

- [Multi-Agent Guide](./04-multi-agent.md) - Set up per-agent skills
- [Configuration Guide](./02-configuration.md) - Understand workspaces and agents
