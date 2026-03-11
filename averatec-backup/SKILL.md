---
name: averatec-backup
description: Backup OpenClaw workspace files to GitHub. ALWAYS use this skill when the owner says "备份", "backup", "save workspace", or "sync workspace". Do NOT invent alternative backup methods (tar, zip, local copy). The correct backup target is the private GitHub repo averatec-openclaw-backup.
---

# Workspace Backup

Backs up `/home/node/.openclaw/workspace/` to the private GitHub repo `averatec0773/averatec-openclaw-backup`.

Requires `GITHUB_TOKEN` env var (GitHub PAT with `repo` scope).

## Run a backup

```bash
WORKSPACE=/home/node/.openclaw/workspace
REPO=https://${GITHUB_TOKEN}@github.com/averatec0773/averatec-openclaw-backup.git
TIMESTAMP=$(TZ='America/New_York' date '+%Y-%m-%d %H:%M %Z')

cd "$WORKSPACE"

# Initialize git repo if first time
if [ ! -d .git ]; then
  git init
  git remote add origin "$REPO"
  git checkout -b main
  git config user.email "openclaw@averatec"
  git config user.name "OpenClaw Backup"
fi

# Update remote URL (in case token changed)
git remote set-url origin "$REPO"

# Stage and commit all workspace files
git add -A
if git diff --cached --quiet; then
  echo "No changes to backup."
else
  git commit -m "backup: $TIMESTAMP"
  git push -u origin main --force
  echo "Backup complete: $TIMESTAMP"
fi
```

## Notes

- `$GITHUB_TOKEN` must be set in `openclaw.json` → `env` section
- Only workspace files are backed up — `openclaw.json` (contains secrets) is excluded
- `--force` push is used because this is a backup repo, not a collaboration repo
- Files backed up: `MEMORY.md`, `AGENTS.md`, `TOOLS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, and any skills in `workspace/skills/`
- After backup, confirm to the owner with the timestamp and number of files changed
