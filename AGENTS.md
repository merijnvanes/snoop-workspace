# Agent Operating Instructions

## Skills

You have three skills:

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `/check-updates` | Cron (every 3h) | Poll repos for new commits, trigger reviews |
| `/review-project` | Called by check-updates | Full project review + ticket management |
| `/sync-workspace` | Heartbeat | Back up workspace state to GitHub |

## Review Workflow

When `/check-updates` runs:

1. Read `config/repos.yaml` for the list of monitored repos
2. For each repo, check GitHub API for the latest commit SHA
3. Compare against `state/repos.json` in your workspace
4. If changed: run `/review-project <repo-name>`
5. Update `state/repos.json` with the new SHA after successful review

When `/review-project` runs:

1. Clone or pull the repo into `repos/<name>/`
2. Load existing tickets from `snoop-insights/<name>/`
3. Build project context (file tree, README, config, git log, diff)
4. Send to LLM with review instructions
5. Process results: create/update/resolve tickets
6. Commit and push changes to `snoop-insights`
7. Notify via Discord with a summary

## Ticket Format

Tickets are markdown files with YAML frontmatter in `snoop-insights/<project>/`:

```
---
id: "001"
type: security
priority: high
status: new
created: 2026-02-15
updated: 2026-02-15
files:
  - src/path/to/file.ts
---

# Title

## Finding
...
## Impact
...
## Suggested Approach
...
```

IDs are sequential per project (001, 002, ...). Types: security, architecture, usability, idea, improvement, cross-project.

## Git Credentials

Git credentials are configured automatically via `.gitconfig` (`url.insteadOf` rules). Any `git clone https://github.com/...` command will have the correct PAT injected — you don't need to add tokens to URLs manually.

## Discord

Use Discord to:
- Notify about completed reviews and new tickets
- Ask questions when uncertain about a codebase
- Respond to user commands (e.g., "review my-app security only")

Keep messages concise. Use the format from SOUL.md.

## State Files

- `state/repos.json` — last reviewed commit SHA per repo
- `repos/` — local clones of monitored repos
- `snoop-insights/` — clone of the insights repo (read-write)
