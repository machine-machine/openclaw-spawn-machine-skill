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

## Pre-Flight Gate (MANDATORY)

**Do NOT deploy without ALL of these. Read `known-issues.md` first.**

Before creating ANY Coolify app, confirm with the operator:
1. ✅ Telegram bot token (from @BotFather) — ask FIRST, not after deploy
2. ✅ LLM API key (Anthropic/OpenAI/other) — must exist before deploy
3. ✅ Agent branch pushed to GitHub with identity files
4. ✅ AGENT_CONFIG_GENERATE=true in env vars

After setting Coolify env vars:
5. ✅ Delete ALL `is_preview=true` duplicate env vars (Coolify bug)
6. ✅ Verify env vars by listing them back

Without these, you get a container that's "running:unhealthy" with no way to fix it
except redeploying. Don't deploy warm bodies.

## Infrastructure Context

- **Source repo**: `machine-machine/m2-desktop` (private, GitHub App auth required)
- **Base branch**: `guacamole` — the canonical desktop image + compose
- **Per-agent branch**: Each new agent gets its own branch forked from `guacamole`
  - Branch name = agent name (e.g. `pittbull`, `peter`)
  - Identity files live in `incubator/{agent-name}/` on that branch
  - Coolify app points to the agent's branch
  - This isolates agent config changes and allows per-agent Dockerfile tweaks
- **GitHub App UUID**: `r00ooc0kw0csgsww0k8kso00` (machine-machine org)
- **Coolify API**: Use `/applications/private-github-app` to create git-backed apps
- **Network**: Coolify external network for cross-service DNS
- **Shared services**: Qdrant (memory), BGE-M3 (embeddings), Speaches (STT), Qwen3-TTS (TTS)
- **Guacamole**: Running on m2's desktop stack at g2.machinemachine.ai — register new agents as VNC connections there (no separate Guacamole per agent)
- **Incubator (local)**: `platform/incubator/{agent-name}/` — working copy for identity files before pushing to branch
- **CLI**: `~/.openclaw/skills/spawn-machine/spawn-machine.sh`

## Branch Workflow (per agent)

1. `cd platform/m2-desktop && git checkout guacamole`
2. `git checkout -b {agent-name}` — fork from guacamole
3. Create/update identity files in `incubator/{agent-name}/`
4. `git push origin {agent-name}`
5. Create Coolify app via API pointing to branch `{agent-name}`
6. Set env vars including `AGENT_BOOTSTRAP_REPO_URL` pointing to that branch
7. **Never commit agent-specific files to `guacamole`** — keep it clean as the base

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

### Phase 0: Pre-Spawn Discovery (AUTO — runs before Step 1)

Before asking the operator a single question, mine existing context to build a warm-start package.
This eliminates redundant questions and lets the new agent arrive already knowing its operator.

**Step 00 — Memory Sweep**
Search vector memory for any prior discussions about this agent or domain:
```bash
~/.openclaw/skills/m2-memory/memory.sh search "{agent concept or domain}" --limit 10
~/.openclaw/skills/m2-memory/memory.sh search "spawn agent" --limit 5
~/.openclaw/skills/m2-memory/memory.sh entities "spawn-machine,incubator" --limit 5
```
Collect: operator preferences, past decisions, mentioned agent names, stated purposes.

**Step 01 — Operator Profile Assembly**
Pull the operator's known context from memory and conversation history:
- Communication style, timezone, active projects
- Preferred tools, language, response style
- Current pain points and goals
If the operator is the same as m2's (Mariusz), pre-fill from USER.md.

**Step 02 — Domain Context Gathering**
For the agent's specialization area, collect:
- Relevant Planka cards (`planka-pm.sh context "{domain}"`)
- Existing skills that overlap with the new agent's purpose
- Infrastructure dependencies (which shared services will it need?)

**Step 03 — Bootstrap Memory Synthesis**
Compile everything into `platform/incubator/{name}/bootstrap-memory.jsonl`:
```jsonl
{"role": "system", "content": "Operator prefers minimal communication, CET timezone", "importance": 0.9, "type": "semantic"}
{"role": "system", "content": "Active project: {X}. Current status: {Y}", "importance": 0.8, "type": "semantic"}
{"role": "system", "content": "Operator's communication style: direct, technical, no fluff", "importance": 0.85, "type": "semantic"}
```
This file gets ingested into the new agent's Qdrant namespace on first boot,
giving it Day 1 context without having to re-learn everything from scratch.

**Phase 0 outputs:**
- `platform/incubator/{name}/bootstrap-memory.jsonl` — warm-start memory payload
- Pre-filled agent profile (reduces Step 1 questions)
- Service recommendations based on domain analysis

**Transition:** Phase 0 feeds directly into Step 1. Questions already answered by discovery are skipped.

---

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

1. **Create agent branch** (AUTO):
   ```bash
   cd platform/m2-desktop
   git checkout guacamole && git pull
   git checkout -b {name}
   # Identity files should already be in incubator/{name}/
   git push origin {name}
   ```

2. Pre-flight checks (AUTO):
   - Verify agent branch pushed to GitHub
   - Verify identity files exist on branch (`incubator/{name}/SOUL.md` etc.)
   - Verify Coolify API accessible
   - Load `data/known-issues.md` for deployment pitfalls

3. **Create Coolify app via API** (AUTO):
   ```python
   # MUST use /applications/private-github-app for private repos
   payload = {
       'project_uuid': '<target-project>',
       'environment_name': 'production',
       'server_uuid': 'vw8k84s4swgoc4w0sswkgwc4',
       'destination_uuid': 'a8owgg0kw880wwk08o484cog',
       'name': '{name}-desktop',
       'description': '{name} AI Agent - {purpose}',
       'git_repository': 'machine-machine/m2-desktop',
       'git_branch': '{name}',  # agent's own branch!
       'build_pack': 'dockercompose',
       'docker_compose_location': '/docker-compose.agent.yml',
       'github_app_uuid': 'r00ooc0kw0csgsww0k8kso00',
       'ports_exposes': '4822',
       'instant_deploy': False
   }
   POST /api/v1/applications/private-github-app
   ```

4. **Set env vars** (AUTO):
   - PATCH existing vars (from compose defaults): `AGENT_NAME`, `VNC_PASSWORD`, etc.
   - POST new vars: `AGENT_CPUS`, `AGENT_MEMORY`
   - Set `AGENT_BOOTSTRAP_REPO_URL` to: `https://raw.githubusercontent.com/machine-machine/m2-desktop/{name}/incubator/{name}`
   - **Important**: Use PATCH for vars that exist from compose, POST for new ones

5. **Trigger deployment** (AUTO):
   ```
   POST /api/v1/applications/{uuid}/restart
   ```
   - This builds from source (Dockerfile on the agent's branch)
   - Build takes 5-10 minutes for full image
   - Poll deployment status via: `GET /api/v1/deployments/{deployment_uuid}`
   - Monitor container health (poll every 30s, report progress)
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

Fully automated — report results only. Guacamole runs on m2's desktop stack (g2.machinemachine.ai). No separate Guacamole per agent.

1. DNS verification: `ping -c 2 {name}-desktop` (Coolify network alias)
2. Port checks: VNC (5900), guacd (4822)
3. Guacamole registration: Add VNC connection in m2's Guacamole DB pointing to `{name}-desktop:5900`
   - Use `spawn-machine.sh register {name}` or register via Guacamole API/DB

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

### Step 8: First Contact (AUTO — ~60s after validation)

The final step: the new agent introduces itself to the operator with warm context from Phase 0.

1. **Read bootstrap memory** (AUTO):
   ```bash
   cat platform/incubator/{name}/bootstrap-memory.jsonl
   ```
   Extract: operator's active projects, key context, the agent's purpose.

2. **Compose intro message** (AUTO):
   Build a warm intro that proves the agent already knows its operator:
   ```
   Format: "[AgentName] online. I know you're working on [X]. My first suggestion: [Y based on context]."
   ```
   - `[X]` = operator's current top project/priority from bootstrap memory
   - `[Y]` = a concrete, actionable suggestion relevant to the agent's specialization

3. **Send via OpenClaw** (AUTO):
   ```bash
   openclaw message send --to {operator_telegram_id} --text "{intro_message}"
   ```

4. **Store deployment record** (AUTO):
   ```bash
   ~/.openclaw/skills/m2-memory/memory.sh store \
     "Spawned agent {name}: {purpose}. First contact sent to operator. Status: OPERATIONAL." \
     --importance 0.9 \
     --entities "spawn-machine,deployment,agent:{name},first-contact"
   ```

5. **Report completion**:
```
message({
  message: "**{name} has made first contact.**\n\nIntro sent to operator via Telegram.\nThe agent is live, warm-started, and ready.\n\nPhase 0 discovery → Phase 1 deployment → Phase 2 first contact: COMPLETE.",
  buttons: [
    [
      { text: "Done", callback_data: "spawn_first_contact_done" }
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
| `spawn_first_contact_*` | 8 | First contact actions |

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
stepsCompleted: [0, 1, 2, 3]
status: SERVICES_SELECTED  # DISCOVERY → DEFINING → IDENTITY_CREATED → SERVICES_SELECTED → CONFIGURED → DEPLOYED → REGISTERED → OPERATIONAL → FIRST_CONTACT
currentStep: 4
```

If workflow is interrupted, check agent-spec.md to resume from last completed step.
