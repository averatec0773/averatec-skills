# averatec-skills

Personal OpenClaw skill library for averatec's self-hosted instance.

## Structure

```
averatec-skills/
├── discord/        # Discord messaging, DM, notifications
└── ...             # more domains coming
```

## Installation

```bash
# Install all skills from this repo
docker compose exec --user root openclaw-gateway \
  clawhub sync --workdir /home/node/.openclaw --dir skills

# Or install individual skill by path
# (after cloning this repo into the container or config volume)
```

## Skills

| Skill | Description |
|---|---|
| `averatec-discord` | Send DMs, channel messages, reactions via Discord API |
