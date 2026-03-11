---
name: averatec-backup
description: Backup OpenClaw workspace files to GitHub. ALWAYS use this skill when the owner says "备份", "backup", "save workspace", or "sync workspace". Do NOT invent alternative backup methods (tar, zip, local copy). The correct backup target is the private GitHub repo averatec-openclaw-backup.
---

# Workspace Backup

Backs up `/home/node/.openclaw/workspace/` to the private GitHub repo `averatec0773/averatec-openclaw-backup`.

Requires `GITHUB_TOKEN` env var (GitHub PAT with `repo` scope).

## Run a backup

Both steps run every time: local snapshot first, then git push.

The git remote and credentials are pre-configured — do NOT change the remote URL or reinitialize the repo.

```bash
WORKSPACE=/home/node/.openclaw/workspace
BACKUP_DIR=/home/node/.openclaw/workspace-backups
TIMESTAMP=$(TZ='America/New_York' date '+%Y%m%d-%H%M%S')
TIMESTAMP_HUMAN=$(TZ='America/New_York' date '+%Y-%m-%d %H:%M %Z')

# Step 1: Local snapshot (tar.gz)
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/workspace-${TIMESTAMP}.tar.gz" -C /home/node/.openclaw workspace
ls -t "$BACKUP_DIR"/workspace-*.tar.gz | tail -n +11 | xargs -r rm
echo "Local snapshot: workspace-${TIMESTAMP}.tar.gz"

# Step 2: Git push to private GitHub repo
cd "$WORKSPACE"
git add -A
if git diff --cached --quiet; then
  echo "Git: no changes since last backup."
else
  git commit -m "backup: $TIMESTAMP_HUMAN"
  GIT_TERMINAL_PROMPT=0 git push origin main --force
  echo "Git: pushed to averatec-openclaw-backup"
fi
```

## Notes

- Git remote and credentials are pre-configured on the server — no setup needed
- Local snapshots kept in `/home/node/.openclaw/workspace-backups/` (last 10 retained)
- Git backup target: `averatec0773/averatec-openclaw-backup` (private repo)
- After backup, report: snapshot filename + git status (pushed or no changes)
