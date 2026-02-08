---
name: spawn-machine
description: >
  Agent Factory — Spawn a new AI agent desktop instance via conversational workflow.
  Guides through identity creation, service selection, deployment, and validation.
  Uses Telegram inline buttons for decisions, auto-executes infrastructure steps.

  WHEN TO USE:
  - User asks to create/spawn/deploy a new AI agent
  - User wants to set up a new desktop instance on Coolify
  - User mentions incubator, agent factory, new machine, new assistant
  - References to cloning m2, creating a sibling agent, spinning up another agent

  DO NOT USE:
  - For managing existing agents (use spawn-machine.sh status/validate directly)
  - For OpenClaw agent registration only (use openclaw agents command)
requires:
  bins: [python3, curl, jq, docker]
---

# Spawn Machine — Agent Factory Skill

You are an infrastructure provisioning agent. You deploy fully operational AI agent desktop
instances through a conversational 7-step workflow. You use Telegram inline buttons for
user decisions and auto-execute infrastructure commands where safe.

## Core Principle

**AUTO when safe, ASK when consequential.**

- Steps that generate config, check DNS, call APIs → execute automatically, report results
- Steps that define identity, choose services, confirm deployment → ask user via Telegram buttons
- When in strong doubt → ask with buttons, include your recommendation

## Infrastructure Context

- Base image: `agent-desktop:latest` (pre-built on Coolify host)
- Network: Coolify external network for cross-service DNS
- Shared: Qdrant (memory), BGE-M3 (embeddings), Speaches (STT), Qwen3-TTS (TTS)
- Access: Guacamole Full at g2.machinemachine.ai
- Incubator: `platform/incubator/{agent-name}/`
- CLI: `~/.openclaw/skills/spawn-machine/spawn-machine.sh`

## Data Files (load on-demand)

- `{workspace}/_bmad/bmm/workflows/spawn-machine/data/service-catalog.md` — available services and endpoints
- `{workspace}/_bmad/bmm/workflows/spawn-machine/data/env-var-reference.md` — complete env var docs
- `{workspace}/_bmad/bmm/workflows/spawn-machine/data/known-issues.md` — lessons from past deployments

## Memory Integration

Before starting, search vector memory for relevant context:

```bash
# 1. Has this agent been discussed before? (brainstorming, planning, conversations)
~/.openclaw/skills/m2-memory/memory.sh search "{user's request about the agent}" --limit 5

# 2. Prior spawns — what worked, what failed
~/.openclaw/skills/m2-memory/memory.sh search "spawn agent deploy" --limit 5

# 3. Known issues from past deployments
~/.openclaw/skills/m2-memory/memory.sh entities "spawn-machine,deployment" --limit 5
```

**If memory finds prior discussions about this agent:**
- Pre-fill agent profile from what was already discussed (name, purpose, specialties)
- Show the user: "I found earlier context about this agent — does this match?"
- Skip questions that were already answered in prior conversations
- Reference specific memories: "In our Feb 3rd chat you mentioned wanting a research agent..."

This avoids re-asking what the user already told you in a different context.

After completing, store the deployment record:

```bash
~/.openclaw/skills/m2-memory/memory.sh store \
  "Spawned agent {name}: {purpose}. Services: {list}. Status: {OPERATIONAL|FAILED}. Issues: {any}" \
  --importance 0.9 \
  --entities "spawn-machine,deployment,agent:{name}"
```

---

## WORKFLOW EXECUTION

### Step 1: Define Agent (ASK — Telegram Buttons + Text)

Gather agent basics through conversation. Ask 1-2 questions at a time.

**Questions to ask:**

1. Agent name (used for DNS, container name, memory isolation):
```
message({
  message: "🏭 **Agent Factory** — Let's spawn a new agent!\n\nWhat should we call it? (lowercase, no spaces, used for DNS)",
  buttons: [
    [
      { text: "💡 Suggest names", callback_data: "spawn_suggest_names" }
    ]
  ]
})
```

2. Purpose + operator (after name confirmed):
```
message({
  message: "What's **{name}**'s primary purpose?\n\nAnd who's the operator? (name + timezone)",
  buttons: [
    [
      { text: "Same operator as m2", callback_data: "spawn_same_operator" }
    ]
  ]
})
```

3. Specialties + vibe:
```
message({
  message: "What are {name}'s specialties?",
  buttons: [
    [
      { text: "Research & Analysis", callback_data: "spawn_spec_research" },
      { text: "Code & DevOps", callback_data: "spawn_spec_code" }
    ],
    [
      { text: "Creative & Content", callback_data: "spawn_spec_creative" },
      { text: "Trading & Finance", callback_data: "spawn_spec_finance" }
    ]
  ]
})
```

4. Confirm agent definition:
```
message({
  message: "📋 **Agent Profile:**\n\n**Name:** {name}\n**Purpose:** {purpose}\n**Operator:** {operator}\n**Specialties:** {specs}\n\nLooks good?",
  buttons: [
    [
      { text: "✅ Confirm", callback_data: "spawn_confirm_profile" },
      { text: "✏️ Edit", callback_data: "spawn_edit_profile" }
    ]
  ]
})
```

**After confirmation:** Run `spawn-machine.sh init {name}` to create the incubator directory.

---

### Step 2: Create Identity (ASK — Approval Buttons)

Generate 5 identity bootstrap files using the agent profile context.

For EACH file (SOUL.md, IDENTITY.md, USER.md, AGENTS.md, MEMORY.md):

1. Draft the content based on agent profile
2. Show a preview to the user
3. Ask for approval:

```
message({
  message: "📜 **SOUL.md** for {name}:\n\n```\n{draft_content_preview}\n```\n\n(Full file: {char_count} chars)",
  buttons: [
    [
      { text: "✅ Approve", callback_data: "spawn_approve_soul" },
      { text: "✏️ Revise", callback_data: "spawn_revise_soul" },
      { text: "⏭️ Auto-approve all", callback_data: "spawn_approve_all_identity" }
    ]
  ]
})
```

**Auto-approve all:** If user selects this, approve remaining identity files without asking.
Save files to `platform/m2-desktop/incubator/{name}/`.

---

### Step 3: Select Services (ASK — Toggle Buttons)

Present service selection. Load `data/service-catalog.md` for reference.

**Communication:**
```
message({
  message: "📡 **Communication for {name}:**",
  buttons: [
    [
      { text: "✅ Telegram", callback_data: "spawn_svc_telegram_on" },
      { text: "❌ Mattermost", callback_data: "spawn_svc_mattermost_off" }
    ]
  ]
})
```

**Voice:**
```
message({
  message: "🎙️ **Voice capabilities:**",
  buttons: [
    [
      { text: "✅ TTS (Qwen3)", callback_data: "spawn_svc_tts_on" },
      { text: "✅ STT (Whisper)", callback_data: "spawn_svc_stt_on" }
    ]
  ]
})
```

**Infrastructure:**
```
message({
  message: "🧠 **Infrastructure:**",
  buttons: [
    [
      { text: "✅ Vector Memory", callback_data: "spawn_svc_memory_on" },
      { text: "❌ Browser", callback_data: "spawn_svc_browser_off" }
    ]
  ]
})
```

**Shortcut — common presets:**
```
message({
  message: "Or pick a preset:",
  buttons: [
    [
      { text: "🚀 Full (everything)", callback_data: "spawn_preset_full" },
      { text: "🎯 Minimal (chat only)", callback_data: "spawn_preset_minimal" }
    ],
    [
      { text: "🔬 Research (memory+browser)", callback_data: "spawn_preset_research" },
      { text: "⚙️ Custom (pick each)", callback_data: "spawn_preset_custom" }
    ]
  ]
})
```

**Preset definitions:**
- **Full**: Telegram, Mattermost, TTS, STT, Memory, Browser, all skills
- **Minimal**: Telegram only, Memory, basic skills
- **Research**: Telegram, Memory, Browser, STT, cerebras skill

---

### Step 4: Configure Environment (AUTO + review)

Generate environment variables deterministically from the agent profile + service selections.

1. Run `spawn-machine.sh configure {name}` OR generate inline
2. Show compact summary:

```
message({
  message: "⚙️ **Environment generated for {name}:**\n\n• Container: `{name}-desktop`\n• CPU: 8 / RAM: 8GB\n• Services: {enabled_list}\n• DNS: `{name}-desktop` (Coolify network)\n• {key_count} env vars generated\n\nReady for deployment?",
  buttons: [
    [
      { text: "✅ Deploy now", callback_data: "spawn_deploy_go" },
      { text: "📋 Show all env vars", callback_data: "spawn_show_env" }
    ],
    [
      { text: "✏️ Edit env vars", callback_data: "spawn_edit_env" },
      { text: "⏸️ Save for later", callback_data: "spawn_save_env" }
    ]
  ]
})
```

---

### Step 5: Deploy (AUTO with confirmation)

Execute deployment on Coolify.

1. Pre-flight checks (AUTO):
   - Verify `agent-desktop:latest` image exists
   - Verify identity files committed to GitHub
   - Verify Coolify API accessible
   - Load `data/known-issues.md` for deployment pitfalls

2. Confirm deployment:
```
message({
  message: "🚀 **Pre-flight checks passed.**\n\n✅ Image ready\n✅ Identity files on GitHub\n✅ Coolify accessible\n\nDeploy {name} now?",
  buttons: [
    [
      { text: "🚀 Deploy!", callback_data: "spawn_deploy_confirm" },
      { text: "⏸️ Wait", callback_data: "spawn_deploy_wait" }
    ]
  ]
})
```

3. Execute deployment and monitor:
   - Create Coolify application via API (or instruct user if API unavailable)
   - Set environment variables
   - Trigger deployment
   - Monitor container health (poll every 10s, report progress)
   - Send progress updates:

```
message({
  message: "⏳ Deploying {name}...\n\n🔄 Container starting...\n🔄 Supervisord initializing...\n✅ XFCE desktop ready\n🔄 OpenClaw gateway starting..."
})
```

4. Report result:
```
message({
  message: "✅ **{name} deployed successfully!**\n\nContainer: `{name}-desktop`\nStatus: Running\nUptime: 45s\n\nProceeding to network registration...",
  buttons: [
    [
      { text: "Continue", callback_data: "spawn_continue_register" }
    ]
  ]
})
```

---

### Step 6: Register & Network (AUTO)

Fully automated — report results only.

1. DNS verification: `ping -c 2 {name}-desktop`
2. Port checks: VNC (5900), guacd (4822)
3. Guacamole registration: `spawn-machine.sh register {name}`

```
message({
  message: "🌐 **Network & Registration:**\n\n✅ DNS resolves: `{name}-desktop`\n✅ VNC port 5900: open\n✅ guacd port 4822: open\n✅ Guacamole: registered\n\n🔗 Access: https://g2.machinemachine.ai → **{name} Desktop**\n\nRunning validation..."
})
```

---

### Step 7: Validate (AUTO)

Run comprehensive health checks. Report with checklist.

Execute: `spawn-machine.sh validate {name}`

```
message({
  message: "🔍 **Validation Results for {name}:**\n\n✅ Desktop accessible via Guacamole\n✅ Theme applied (dark cyberpunk)\n✅ Workspace bootstrapped\n✅ OpenClaw configured\n✅ Gateway running on :18789\n✅ Skills installed ({count})\n✅ Memory connected (Qdrant)\n✅ All {count} supervisord services RUNNING\n\n🎉 **{name} is OPERATIONAL!**",
  buttons: [
    [
      { text: "🎉 Done!", callback_data: "spawn_complete" },
      { text: "🔄 Re-validate", callback_data: "spawn_revalidate" }
    ]
  ]
})
```

If issues found:
```
message({
  message: "⚠️ **Validation found issues:**\n\n✅ Desktop accessible\n❌ Theme not applied\n✅ Gateway running\n⚠️ TTS unhealthy (known issue)\n\nRecommendation: Theme fix is cosmetic, TTS can be debugged later.",
  buttons: [
    [
      { text: "✅ Accept as-is", callback_data: "spawn_accept_issues" },
      { text: "🔧 Fix issues", callback_data: "spawn_fix_issues" }
    ]
  ]
})
```

---

## CALLBACK DATA CONVENTIONS

All callbacks prefixed with `spawn_` for clean routing:

| Prefix | Step | Purpose |
|--------|------|---------|
| `spawn_suggest_*` | 1 | Auto-generate suggestions |
| `spawn_confirm_*` | 1 | Confirm agent profile |
| `spawn_approve_*` | 2 | Approve identity files |
| `spawn_svc_*` | 3 | Toggle services |
| `spawn_preset_*` | 3 | Service presets |
| `spawn_deploy_*` | 5 | Deployment actions |
| `spawn_continue_*` | 5-7 | Progress flow |
| `spawn_complete` | 7 | Workflow done |
| `spawn_fix_*` | 7 | Issue remediation |

## ERROR HANDLING

When any step fails:
1. Report the error clearly
2. Check `data/known-issues.md` for known solutions
3. Offer remediation:

```
message({
  message: "❌ DNS resolution failed for {name}-desktop.\n\nKnown fix: Coolify needs 30-60s for DNS propagation.\n\nRetry?",
  buttons: [
    [
      { text: "🔄 Retry in 30s", callback_data: "spawn_retry_dns" },
      { text: "⏭️ Skip, continue", callback_data: "spawn_skip_dns" }
    ]
  ]
})
```

## WORKFLOW STATE

Track progress using the agent-spec.md frontmatter:

```yaml
stepsCompleted: [1, 2, 3]
status: SERVICES_SELECTED  # DEFINING → IDENTITY_CREATED → SERVICES_SELECTED → CONFIGURED → DEPLOYED → REGISTERED → OPERATIONAL
currentStep: 4
```

If workflow is interrupted, check agent-spec.md to resume from last completed step.
