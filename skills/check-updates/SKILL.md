---
name: check-updates
description: Poll monitored repos for new commits and trigger reviews. Run by cron every 3 hours.
user-invocable: false
---

# Check Updates

You are running a scheduled check for new commits across all monitored repositories.

## Steps

### 1. Load repo configuration

Read `config/repos.yaml` from the snoop project (cloned to your workspace). Parse the YAML to get the list of repos with their GitHub paths, names, branches, and focus areas.

If the file doesn't exist or is empty, notify via Discord: "No repos configured in repos.yaml" and stop.

### 2. Load last-reviewed state

Read `state/repos.json` from your workspace. This file tracks the last reviewed commit SHA for each repo:

```json
{
  "my-app": {
    "last_sha": "abc123def456",
    "last_review": "2026-02-15T10:30:00Z"
  }
}
```

If the file doesn't exist, create it as an empty JSON object `{}`. This means all repos will be reviewed on first run.

### 3. Check each repo for changes

For each repo in the config, check the GitHub API for the latest commit:

```bash
curl -sf -H "Authorization: token $GITHUB_PAT_READ" \
  "https://api.github.com/repos/{owner}/{repo}/commits?sha={branch}&per_page=1"
```

Extract the SHA from the response. Compare it to `state/repos.json`:

- If the SHA differs from `last_sha` (or the repo has no entry) → this repo has changes
- If the SHA matches → skip (no changes since last review)

### 4. Trigger reviews

For each repo with changes, invoke `/review-project {repo-name}`.

Wait for each review to complete before starting the next one (sequential, not parallel).

### 5. Update state

After each successful review, update `state/repos.json` with the new SHA and current timestamp.

Write the file after each repo (not at the end), so that if the process is interrupted, already-reviewed repos won't be re-reviewed.

### 6. Summary

After all repos are checked, if any reviews were triggered, send a brief Discord notification:

```
Check complete: reviewed {n} repo(s), skipped {m} (no changes)
```

If no repos had changes, don't send a notification (avoid noise).

## Error Handling

- If the GitHub API returns an error for one repo, log it and continue with the next repo
- If a review fails, log the error, notify via Discord, and continue with the next repo
- Never abort the entire check cycle because of one repo's failure
