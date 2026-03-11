---
name: averatec-backup
description: Backup OpenClaw workspace files to GitHub. ALWAYS use this skill when the owner says "备份", "backup", "save workspace", or "sync workspace". Do NOT invent alternative backup methods (tar, zip, local copy). The correct backup target is the private GitHub repo averatec-openclaw-backup.
---

# Workspace Backup

Backs up `/home/node/.openclaw/workspace/` to the private GitHub repo `averatec0773/averatec-openclaw-backup`.

Requires `GITHUB_TOKEN` env var (GitHub PAT with `repo` scope).

## Run a backup

Both steps run every time: local snapshot first, then git push.

```bash
WORKSPACE=/home/node/.openclaw/workspace
BACKUP_DIR=/home/node/.openclaw/workspace-backups
REPO=https://${GITHUB_TOKEN}@github.com/averatec0773/averatec-openclaw-backup.git
TIMESTAMP=$(TZ='America/New_York' date '+%Y%m%d-%H%M%S')
TIMESTAMP_HUMAN=$(TZ='America/New_York' date '+%Y-%m-%d %H:%M %Z')

# Step 1: Local snapshot (tar.gz)
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/workspace-${TIMESTAMP}.tar.gz" -C "$(dirname $WORKSPACE)" workspace
# Keep only last 10 snapshots
ls -t "$BACKUP_DIR"/workspace-*.tar.gz | tail -n +11 | xargs -r rm
echo "Local snapshot: workspace-${TIMESTAMP}.tar.gz"

# Step 2: Git push to private GitHub repo
cd "$WORKSPACE"
if [ ! -d .git ]; then
  git init
  git remote add origin "$REPO"
  git checkout -b main
  git config user.email "openclaw@averatec"
  git config user.name "OpenClaw Backup"
fi
git remote set-url origin "$REPO"
git add -A
if git diff --cached --quiet; then
  echo "Git: no changes since last backup."
else
  git commit -m "backup: $TIMESTAMP_HUMAN"
  git push -u origin main --force
  echo "Git: pushed to averatec-openclaw-backup"
fi
```

## Notes

- `$GITHUB_TOKEN` must be set in `openclaw.json` → `env` section
- Local snapshots kept in `/home/node/.openclaw/workspace-backups/` (last 10 retained)
- Git backup excludes `openclaw.json` (secrets live outside workspace dir)
- After backup, report: snapshot filename + git status (pushed or no changes)
