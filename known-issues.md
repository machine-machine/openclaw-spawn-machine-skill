# Known Issues — Spawn Machine

Lessons from deploying miauczek, peter, pittbull (Feb 2026).

## Critical: Pre-flight Requirements

**DO NOT deploy without these. You'll get a warm body, not an agent.**

1. **Telegram bot token** — Create via @BotFather BEFORE deploying. Ask operator upfront.
2. **LLM API key** — Anthropic, OpenAI, or other. Must be provisioned BEFORE deploy.
3. **AGENT_CONFIG_GENERATE=true** — Without this, the entire provisioning block is skipped. The container starts but OpenClaw doesn't configure.

## Coolify Env Var Duplication Bug

**Problem:** When you POST env vars to a Coolify docker-compose app, it creates BOTH a production AND a preview copy. The preview copy has default/empty values and can override the real ones.

**Fix:** After setting all env vars, immediately delete all `is_preview=true` entries:
```bash
# Get all envs, filter preview ones, delete them
curl -s -H "Auth..." "$APP/envs" | jq '.[] | select(.is_preview==true) | .uuid' | while read uuid; do
    curl -X DELETE "$APP/envs/$uuid"
done
```

**Always verify** by listing env vars after creation and checking for duplicates.

## Branch Must Exist Before Deploy

**Problem:** Setting `AGENT_BOOTSTRAP_REPO_URL` to a branch that doesn't exist = silent failure. Identity files don't download, agent boots with empty workspace.

**Fix:** Create and push the branch BEFORE creating the Coolify app:
```bash
cd platform/m2-desktop
git checkout guacamole && git checkout -b {name}
# Add identity files
git push origin {name}
# THEN create Coolify app
```

## Health Check Lies

**Problem:** Coolify health check on port 4822 (guacd) passes even when OpenClaw gateway isn't running. Container shows "running:healthy" but the agent is braindead.

**Better validation:** After deploy, check OpenClaw gateway directly:
```bash
curl -s http://{name}-desktop:18789/health
```
This requires docker socket mounted on m2 (or network access).

## Docker Socket Required for Fleet Management

m2 needs `/var/run/docker.sock` mounted to exec into sibling containers.
Without it, you can't debug agents after deploy. Added to m2's compose in Feb 2026.

## Fleet Governance Bootstrap

New agents should clone `machine-machine/fleet-governance` during bootstrap.
Add to the agent's AGENTS.md or provision-bootstrap.sh.
Planka credentials should be in env vars for coordination access.
