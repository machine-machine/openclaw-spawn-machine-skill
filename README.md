# openclaw-spawn-machine-skill

Agent Factory skill for OpenClaw — spawn new AI agent desktop instances through a conversational Telegram workflow with inline buttons.

## How It Works

7-step workflow from "name your agent" to "agent OPERATIONAL":

| Step | What | User Interaction |
|------|------|-----------------|
| 1. Define | Name, purpose, operator, specialties | Text input + confirm button |
| 2. Identity | SOUL.md, IDENTITY.md, USER.md, AGENTS.md, MEMORY.md | Approve/revise buttons per file |
| 3. Services | Telegram, TTS, STT, Memory, Browser, Skills | Toggle buttons or preset selection |
| 4. Configure | Generate env vars for Coolify | Auto-generate, optional review |
| 5. Deploy | Coolify deployment + monitoring | Confirm button, auto-monitor |
| 6. Register | Guacamole + DNS + networking | Fully automated |
| 7. Validate | Health checks across all systems | Auto-check, report results |

## Design Principle

**AUTO when safe, ASK when consequential.**

Steps 4-7 are mostly automated with progress reporting. Steps 1-3 need real user decisions via Telegram inline buttons.

## Dependencies

- `spawn-machine.sh` CLI (existing, at `~/.openclaw/skills/spawn-machine/`)
- BMAD workflow data files (`_bmad/bmm/workflows/spawn-machine/data/`)
- Coolify API access (`~/.config/coolify/config`)
- Guacamole API access (`~/.config/guacamole/config`)
- m2-memory skill (vector memory for deployment records)

## Memory Integration

- Searches past deployments before starting (known issues, patterns)
- Stores deployment record after completion (agent name, services, status)
- Future spawns benefit from accumulated deployment knowledge
