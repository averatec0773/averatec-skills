# averatec-skills

Personal OpenClaw skill library for averatec's self-hosted instance.

## Skills

| Skill | Description |
|---|---|
| `averatec-discord` | Proactive outbound DMs and channel messages via Discord API |
| `averatec-email` | Send formatted HTML emails via Gmail (gog CLI) |

## Deploy a skill

```bash
scp <skill>/SKILL.md openclaw:/root/.openclaw/skills/<skill>/SKILL.md
# OpenClaw hot-reloads — no restart needed
```
