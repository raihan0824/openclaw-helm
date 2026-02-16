# OpenClaw Troubleshooting Guide

This guide covers common issues and solutions when running OpenClaw on Cloudeka.

## Quick Diagnostic Commands

```bash
# Check pod status
kubectl get pods -n openclaw

# Check pod events
kubectl describe pod -n openclaw <pod-name>

# View logs
kubectl logs -n openclaw deployment/openclaw -f

# View logs for specific container
kubectl logs -n openclaw deployment/openclaw -c main -f
kubectl logs -n openclaw deployment/openclaw -c init-skills -f

# Check PVC
kubectl get pvc -n openclaw

# Port forward to test locally
kubectl port-forward -n openclaw svc/openclaw 18789:18789
```

## Common Issues

### Pod Not Starting

#### Issue: CrashLoopBackOff

**Symptoms:**
```
NAME                       READY   STATUS             RESTARTS   AGE
openclaw-xxx               0/3     CrashLoopBackOff   5          5m
```

**Possible Causes:**

1. **Missing required environment variables**
   ```bash
   # Check if secret exists
   kubectl get secret openclaw-env-secret -n openclaw

   # Verify secret contents
   kubectl get secret openclaw-env-secret -n openclaw -o jsonpath='{.data}'
   ```

   Solution: Create the secret with required variables:
   ```bash
   kubectl create secret generic openclaw-env-secret -n openclaw \
     --from-literal=DEKALLM_API_KEY=xxx \
     --from-literal=OPENCLAW_GATEWAY_TOKEN=yyy
   ```

2. **Channel enabled but token not set**

   If you enabled a channel (Slack, Telegram, Discord) in `openclaw.json` but didn't set the corresponding environment variable, OpenClaw will fail to start.

   **Error example:** `SLACK_BOT_TOKEN is not set`

   Solution: Either add the token to your secret OR comment out the channel in openclaw.json:
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

3. **Persistence mount references non-existent secret/ConfigMap**

   If you uncommented persistence entries (kubeconfig, skills, MCP config) without creating the corresponding secret/ConfigMap, the pod will fail.

   **Error example:** `Error: couldn't find secret openclaw-kubeconfig`

   Solution: Create the secret first, or keep the persistence commented out:
   ```bash
   # Create the secret BEFORE uncommenting in values.yaml
   kubectl create secret generic openclaw-kubeconfig \
     --from-file=config=/path/to/kubeconfig -n openclaw
   ```

4. **Invalid openclaw.json syntax**
   ```bash
   # Check init-config logs
   kubectl logs -n openclaw deployment/openclaw -c init-config
   ```

   Look for: `Warning: existing config is not valid JSON`

   Solution: Fix JSON syntax in values.yaml

5. **Image pull errors**
   ```bash
   # Check if image exists
   kubectl describe pod -n openclaw <pod-name>
   ```

   Solution: Verify image registry access and credentials

#### Issue: Init container fails

**Symptoms:**
```
init-config container has failed
init-skills container has failed
```

**Solution:**
```bash
# Check init-config logs
kubectl logs -n openclaw deployment/openclaw -c init-config

# Check init-skills logs
kubectl logs -n openclaw deployment/openclaw -c init-skills
```

### Cannot Pair Device

#### Issue: Pairing request not appearing

**Symptoms:** You open http://localhost:18789 but no pairing request shows

**Solutions:**

1. **Check if port-forward is running:**
   ```bash
   kubectl port-forward -n openclaw svc/openclaw 18789:18789
   ```

2. **Verify gateway token:**
   ```bash
   kubectl get secret openclaw-env-secret -n openclaw \
     -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
   ```

3. **Check gateway logs:**
   ```bash
   kubectl logs -n openclaw deployment/openclaw -c main -f
   ```

#### Issue: Pairing approval fails

**Symptoms:** Request appears but approval doesn't work

**Solution:**
```bash
# List pending requests
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices list

# Approve (replace <REQUEST_ID>)
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices approve <REQUEST_ID>

# If that fails, check for errors
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices approve --help
```

### Skills Not Loading

#### Issue: Skill not found

**Symptoms:** Agent says skill doesn't exist

**Diagnosis:**
```bash
# Check if skill files exist in pod
kubectl exec -n openclaw deployment/openclaw -- ls -la /home/node/.openclaw/workspace/skills/

# Check skill mount in pod spec
kubectl describe pod -n openclaw <pod-name> | grep -A 20 Mounts
```

**Solutions:**

1. **Single-file skill:** Verify ConfigMap exists and path matches
   ```bash
   kubectl get configmap my-skill -n openclaw
   kubectl describe configmap my-skill -n openclaw
   ```

2. **Tar-based skill:** Verify archive structure
   ```bash
   # Check init-skills extraction logs
   kubectl logs -n openclaw deployment/openclaw -c init-skills | grep -i skill
   ```

3. **Redeploy after adding skill:**
   ```bash
   kubectl rollout restart deployment openclaw -n openclaw
   ```

#### Issue: Nested skill files missing

**Symptoms:** SKILL.md loads but other files in skill folder are not found

**Solution:** Use the tar archive method instead of flat ConfigMap mounts. See [Skills Guide](./03-skills.md).

### Agent Not Responding

#### Issue: Agent replies to wrong channel

**Symptoms:** Message sent to Slack but agent responds elsewhere or not at all

**Diagnosis:**
```bash
# Check agent configuration
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js agents list --bindings
```

**Solution:** Verify bindings in openclaw.json match your channel IDs:
```json
"bindings": [
  {
    "agentId": "main",
    "match": {
      "channel": "slack",
      "peer": { "kind": "group", "id": "C0YOURCHANNEL" }
    }
  }
]
```

#### Issue: Multi-agent routing not working

**Diagnosis:**
```bash
# Check which agents are configured
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js agents list

# Check bindings
kubectl exec -n openclaw deployment/openclaw -- node dist/index.js agents list --bindings
```

**Solution:** Verify:
1. Each agent has unique `workspace` path
2. Bindings are in order (most specific first)
3. Channel IDs are correct

### Channel Connection Issues

#### Slack: App not responding

**Diagnosis:**
```bash
# Check logs for Slack errors
kubectl logs -n openclaw deployment/openclaw -c main | grep -i slack
```

**Common issues:**
1. **Socket Mode not enabled** — Enable in Slack App settings
2. **Missing scopes** — Add required OAuth scopes
3. **Invalid tokens** — Verify bot token and app token

#### Telegram: Bot not responding

**Diagnosis:**
```bash
# Check logs for Telegram errors
kubectl logs -n openclaw deployment/openclaw -c main | grep -i telegram
```

**Common issues:**
1. **Bot token invalid** — Regenerate via @BotFather
2. **Webhook conflicts** — Ensure no other webhook is set
3. **Not paired** — Complete Telegram pairing after installation

**Telegram Pairing:**

After enabling Telegram, you must pair the bot:

```bash
# 1. Start chat with your bot in Telegram app
# 2. Send /start to get pairing code
# 3. Approve the pairing (replace <PAIRING_CODE>)
kubectl exec -n openclaw deployment/openclaw -c main -- \
  node dist/index.js pairing approve telegram <PAIRING_CODE>
```

Example: `kubectl exec -n openclaw deployment/openclaw -c main -- node dist/index.js pairing approve telegram PML9NL9U`

### kubectl Access Not Working

#### Issue: kubectl commands fail from agent

**Symptoms:** Agent says kubectl not found or cannot access cluster

**Diagnosis:**
```bash
# Check if kubeconfig is mounted
kubectl exec -n openclaw deployment/openclaw -- ls -la /home/node/.openclaw/workspace/.kube/config

# For multi-agent, check specific workspace
kubectl exec -n openclaw deployment/openclaw -- ls -la /home/node/.openclaw/workspace-main/.kube/config
```

**Solution:** Verify kubeconfig secret exists and is mounted correctly:
```yaml
persistence:
  kubeconfig:
    enabled: true
    type: secret
    name: openclaw-kubeconfig
    advancedMounts:
      main:
        main:
          - path: /home/node/.openclaw/workspace/.kube/config
            subPath: config
            readOnly: true
```

### MCP Server Connection Issues

#### Issue: MCP server unreachable

**Symptoms:** `mcporter call` fails with connection error

**Diagnosis:**
```bash
# Check if mcporter is installed
kubectl exec -n openclaw deployment/openclaw -- which mcporter

# Check MCP config
kubectl exec -n openclaw deployment/openclaw -- cat /home/node/.openclaw/workspace/config/mcporter.json

# Test connectivity from pod
kubectl exec -n openclaw deployment/openclaw -- curl -v http://mcp-server:3000/mcp
```

**Solution:** Verify network policy allows egress to MCP server

### PVC Issues

#### Issue: PVC in pending state

**Diagnosis:**
```bash
kubectl get pvc -n openclaw
kubectl describe pvc -n openclaw openclaw-data
```

**Solution:** Check StorageClass is available in your cluster

#### Issue: Data lost after restart

**Cause:** PVC was deleted or `configMode: overwrite` is set

**Solution:**
- Use `configMode: merge` to preserve runtime changes
- Don't delete PVC unless intentionally

### Performance Issues

#### Issue: Slow agent responses

**Diagnosis:**
```bash
# Check resource usage
kubectl top pod -n openclaw

# Check resource limits
kubectl describe pod -n openclaw <pod-name> | grep -A 10 Limits
```

**Solution:** Increase resource limits in values.yaml:
```yaml
resources:
  limits:
    cpu: 4000m      # Increase from 2000m
    memory: 4Gi     # Increase from 2Gi
```

## Getting Help

If issues persist:

1. **Collect diagnostic info:**
   ```bash
   # Pod info
   kubectl get pods -n openclaw -o yaml > pod-info.yaml

   # Logs
   kubectl logs -n openclaw deployment/openclaw --all-containers > openclaw.log

   # Events
   kubectl get events -n openclaw > events.log
   ```

2. **Check OpenClaw documentation:** https://docs.openclaw.ai

3. **Check GitHub issues:** https://github.com/openclaw/openclaw/issues

## Useful Commands Reference

| Purpose | Command |
|---------|---------|
| List agents | `kubectl exec -n openclaw deployment/openclaw -- node dist/index.js agents list` |
| List devices | `kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices list` |
| Approve device | `kubectl exec -n openclaw deployment/openclaw -- node dist/index.js devices approve <id>` |
| Pair Telegram | `kubectl exec -n openclaw deployment/openclaw -c main -- node dist/index.js pairing approve telegram <CODE>` |
| Shell into pod | `kubectl exec -n openclaw deployment/openclaw -it -- sh` |
| Check config | `kubectl exec -n openclaw deployment/openclaw -- cat /home/node/.openclaw/openclaw.json` |
| Restart pod | `kubectl rollout restart deployment openclaw -n openclaw` |
