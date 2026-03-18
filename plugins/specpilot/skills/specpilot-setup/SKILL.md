---
name: specpilot-setup
description: Set up the specpilot plugin in the current project. Use this when the user runs /specpilot-setup or asks to install, configure, or set up specpilot.
argument-hint: (no arguments needed)
allowed-tools: [Read, Write, Edit, Bash, Glob, WebFetch]
---

# /specpilot-setup

Set up specpilot in the current project. Collects your project configuration and makes `/refine` and `/implement` ready to use.

## Steps

### 0. Check prerequisites
- Verify `gh` CLI is authenticated: run `gh auth status`. If not, stop and tell the user to run `gh auth login` first.
- Verify Claude Code is running from the project root.
- Create `.claude/commands/` if it doesn't exist.

### 1. Collect GitHub settings
Ask the user:
- GitHub owner (org or username) that owns the GitHub Project board
- GitHub Project number (visible in the project URL: `github.com/orgs/<org>/projects/N` or `github.com/users/<user>/projects/N`)
- GitHub username to filter tickets by assignee (optional — press Enter to skip)

### 2. Locate the spec
Ask: *"Is your spec in a separate repo or a folder inside this project?"*

- **Separate repo** → ask for the relative path (e.g. `../my-project-spec`). Default name convention: `<project-name>-spec`.
- **Folder inside this project** → default to `./spec`. Create the folder if it doesn't exist.

Then ask: *"Where should speckit slash commands run from?"* — this is the `workspaceDir`. Default: `.` (current project root). For multi-repo setups where a parent directory has the CLAUDE.md with speckit commands, use `..`.

### 3. Collect repos
Tell the user: *"Now let's add your feature repos — these are the repos where code gets implemented. Add as many as your project needs."*

For each repo, ask:
- Name (used in PR titles and comments)
- Relative path from the project root (e.g. `../my-api`, `../my-web`)
- Should speckit implement code changes in this repo? (yes/no, default: yes)
- Test command (e.g. `npm test`) — press Enter to skip
- Build command (e.g. `npm run build`) — press Enter to skip
- Check if `.github/workflows/` exists at that path. If yes, ask: *"Wait for CI before merging PRs in `<repo-name>`? (yes/no)"* — save as `waitForCI`. If no workflows folder, set `waitForCI: false` silently.

Ask *"Add another repo? (yes/no)"* after each one.

### 4. Collect column names
Show defaults and ask to confirm or override:
- Todo (default: `Todo`)
- Refinement (default: `Refinement`)
- Ready (default: `Ready`)
- In Progress (default: `In Progress`)
- Done (default: `Done`)

Verify columns exist: `gh project field-list <projectNumber> --owner <owner> --format json`
Warn if any column name doesn't match.

### 5. Choose models
Ask:
- Model for `/refine` (spec phase) — default: `claude-opus-4-6`
- Model for `/implement` (build phase) — default: `claude-sonnet-4-6`

### 6. Write `.specpilot.json`
Write the collected config to `.specpilot.json` in the project root using this shape:

```json
{
  "github": { "owner": "...", "projectNumber": N, "assignee": "..." },
  "repos": [
    { "name": "my-spec", "path": "../my-spec", "role": "spec", "waitForCI": false },
    { "name": "my-api",  "path": "../my-api",  "role": "feature", "implement": true, "test": "npm test", "build": "npm run build", "waitForCI": true }
  ],
  "speckit": { "specsDir": "../my-spec", "workspaceDir": ".." },
  "statuses": { "todo": "Todo", "refinement": "Refinement", "ready": "Ready", "inProgress": "In Progress", "done": "Done" },
  "models": { "refine": "claude-opus-4-6", "implement": "claude-sonnet-4-6" }
}
```

### 7. Update CLAUDE.md
Append a `## specpilot` section to `CLAUDE.md`:

```markdown
## specpilot

This project uses the [specpilot](https://github.com/eduardo-caua/cogloop) plugin.

### Commands
- `/refine` — picks next Todo ticket, runs specify→clarify→plan→tasks, moves to Ready
- `/implement` — picks next Ready ticket, implements→tests→ships, moves to Done

### Flags
`--once` (one ticket then stop), `--dry-run` (preview only)

### Config
See `.specpilot.json` in the project root.
```

### 8. Done
Print a summary:
- Config written to `.specpilot.json`
- Skills available: `/refine`, `/implement`
- Spec location: `<specsDir>`
- Feature repos: list each
- GitHub Project: `<owner>/#<projectNumber>`
- Next step: add tickets to the **Todo** column and run `/refine`
