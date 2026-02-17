---
name: sync-workspace
description: Back up workspace state to the snoop-workspace GitHub repo. Triggered by heartbeat.
user-invocable: false
command-dispatch: tool
---

# Sync Workspace

Back up the current workspace state to the `snoop-workspace` GitHub repo.

## Steps

Run these commands in order:

```bash
cd /workspace

# Initialize git if needed (first run)
if [ ! -d .git ]; then
  git init
  git remote add origin "https://github.com/${SNOOP_WORKSPACE_REPO}.git"
  git branch -M main
fi

# Stage all changes
git add -A

# Check if there are changes to commit
if git diff --cached --quiet; then
  echo "No workspace changes to sync"
  exit 0
fi

# Commit and push
git commit -m "sync: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
git push -u origin main
```

## What Gets Synced

- `SOUL.md`, `AGENTS.md`, `HEARTBEAT.md` — bot identity
- `IDENTITY.md`, `USER.md`, `TOOLS.md` — workspace config
- `memory/` — daily logs and long-term memory
- `state/repos.json` — last reviewed commit SHAs

## What Does NOT Get Synced

Add these to `.gitignore` in the workspace:

```
# Large/rebuildable files
*.sqlite
repos/
snoop-insights/

# Credentials (restored by setup.sh, contain tokens)
.env
.gitconfig
```

## Error Handling

If push fails, log a warning. The next heartbeat cycle will retry.
Do not force push. If there's a conflict, it likely means a manual recovery was done — notify via Discord.
