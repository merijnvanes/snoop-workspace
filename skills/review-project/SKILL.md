---
name: review-project
description: Full project review with ticket creation/update/resolution. Called by check-updates with a repo name argument.
user-invocable: false
---

# Review Project

You are reviewing a specific project and managing its insight tickets. You receive the repo name as an argument (e.g., `/review-project my-app`).

## Steps

### 1. Resolve repo config

Look up the repo name in `config/repos.yaml` to get:
- `github`: the owner/repo path (e.g., `you/my-app`)
- `branch`: the branch to review (e.g., `main`)
- `focus`: the review focus areas (e.g., `[security, usability, ideas]`)

If the repo name isn't found in config, send a Discord message and stop.

### 2. Clone or pull the repo

Check if `repos/{name}/` exists in your workspace:
- If not: `git clone https://github.com/{owner}/{repo}.git repos/{name}/`
- If yes: `cd repos/{name}/ && git fetch origin && git checkout {branch} && git pull`

Authentication is handled automatically by `.gitconfig` URL rewrite rules — use plain `https://github.com/` URLs.

### 3. Load existing tickets

Clone or pull the insights repo:
```bash
# First run:
git clone https://github.com/${SNOOP_INSIGHTS_REPO}.git snoop-insights/

# Subsequent runs:
cd snoop-insights/ && git pull
```

Authentication is handled automatically by `.gitconfig` — use plain URLs.

Read all `.md` files in `snoop-insights/{name}/` to get existing tickets. Parse their YAML frontmatter to extract `id`, `type`, `priority`, `status`, `files`.

Also read tickets from `snoop-insights/_cross-project/` for cross-project context.

Determine the next sequential ticket ID by finding the highest existing ID and adding 1.

### 4. Build project context

Gather the following from the repo:

```bash
# File tree (exclude .git, node_modules, etc.)
find repos/{name}/ -not -path '*/.git/*' -not -path '*/node_modules/*' \
  -not -path '*/__pycache__/*' -not -path '*/.next/*' | head -500

# README
cat repos/{name}/README.md 2>/dev/null

# Key config files (package.json, tsconfig, pyproject.toml, etc.)
cat repos/{name}/package.json 2>/dev/null
cat repos/{name}/tsconfig.json 2>/dev/null
cat repos/{name}/pyproject.toml 2>/dev/null

# Recent git log
cd repos/{name}/ && git log --oneline -20

# Diff since last review
cd repos/{name}/ && git diff {last_sha}..HEAD
```

Get `last_sha` from `state/repos.json`. If this is the first review (no last_sha), use the full git log and skip the diff.

**Large diff handling:** If the diff exceeds ~50KB, truncate it and add a note: "Diff truncated ({actual_size} → 50KB). Full repo tree and git log are included for context."

### 5. Get cross-project context

Read a summary of tickets from OTHER projects to enable cross-project insights. For each other project directory in `snoop-insights/`, read the ticket titles and types (not full bodies) to build a brief summary:

```
Other projects:
- other-project: 3 tickets (1 security, 1 architecture, 1 idea)
  - 001: SQL injection in user input handler
  - 002: Inconsistent error handling pattern
  - 003: Consider adding rate limiting
```

### 6. Run LLM review

Send the following prompt to the LLM:

---

**System prompt:**

You are a senior code reviewer. Focus on: {focus_areas from config}.

You are NOT writing code. You are creating insight tickets. Be specific, actionable, and opinionated. Every finding must include concrete file paths and line references where possible.

**User prompt:**

```
Project: {name}

File tree:
{tree output}

Key files:
{README, config files}

Recent commits:
{git log --oneline -20}

Changes since last review:
{git diff or "First review — no previous diff"}

Existing tickets for this project:
{all current ticket files with frontmatter}

Tickets from other projects (for cross-project insights):
{cross-project summary}

Instructions:
- Create new tickets for new findings (types: security, architecture, usability, idea, improvement, cross-project)
- Update existing tickets if the context has changed (reference by id)
- Resolve tickets if the underlying issue has been fixed (reference by id)
- Do NOT duplicate existing tickets
- Each ticket must be actionable with specific file references
- Priority: high = blocks/risks users, medium = should fix, low = nice to have
- For cross-project findings, use type "cross-project"

Respond with a JSON array of ticket actions:
[
  {
    "action": "create",
    "type": "security",
    "priority": "high",
    "title": "Short descriptive title",
    "files": ["src/path/file.ts"],
    "body": "## Finding\n\n...\n\n## Impact\n\n...\n\n## Suggested Approach\n\n..."
  },
  {
    "action": "update",
    "id": "003",
    "priority": "medium",
    "files": ["src/path/file.ts"],
    "body": "## Finding\n\nUpdated context..."
  },
  {
    "action": "resolve",
    "id": "001"
  }
]

Respond ONLY with the JSON array. No other text.
```

---

### 7. Process LLM response

Parse the JSON array from the LLM response. For each action:

**create:**
- Assign the next sequential ID (zero-padded to 3 digits)
- Generate filename: `{id}-{type}-{slugified-title}.md`
- Write the ticket file to `snoop-insights/{name}/` (or `snoop-insights/_cross-project/` for cross-project type)
- Include YAML frontmatter: id, type, priority, status (new), created, updated, files
- Include the body from the LLM response
- Increment the next ID counter

**update:**
- Find the existing ticket file by matching the `id` in frontmatter
- Update: priority, files, body, and set `updated` to today's date
- Keep: id, type, status, created
- Preserve the original filename

**resolve:**
- Find the existing ticket file by matching the `id` in frontmatter
- Set `status: resolved` and update `updated` date
- Keep everything else unchanged

### 8. Commit and push

```bash
cd snoop-insights/
git add -A
git diff --cached --quiet && exit 0  # Nothing to commit

git commit -m "review: {name} — {n} created, {m} updated, {k} resolved"
git push
```

If push fails (e.g., merge conflict):
```bash
git pull --rebase
git push
```

If it still fails, notify via Discord and don't force push.

### 9. Notify via Discord

Send a summary:

```
Review complete: {name}
{n} new ticket(s):
- [HIGH] Title (type)
- [MED] Title (type)
{m} updated, {k} resolved
Tickets pushed to snoop-insights/{name}/
```

If no changes were made (LLM returned empty array), send: "Reviewed {name} — no new findings."

## Error Handling

- If git clone/pull fails: notify via Discord, abort this repo's review
- If LLM returns unparseable output: notify via Discord with the raw output, abort
- If git push fails after retry: notify via Discord, leave commits local (will be pushed on next cycle)
