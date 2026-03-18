# cogloop-install

Install the specpilot plugin into the current project.

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

- **Separate repo** → ask for the relative path (e.g. `../my-project-spec`). The default repo name convention is `<project-name>-spec`.
- **Folder inside this project** → default to `./spec`. Create the folder if it doesn't exist. Ask the user to confirm or change the path.

Then ask: *"Where should speckit slash commands run from?"* — this is the `workspaceDir`. Default: the current project root (`.`). For multi-repo setups where a parent directory has the CLAUDE.md with speckit commands, use `..`.

### 3. Collect repos
Tell the user: *"Now let's add your feature repos — these are the repos where code gets implemented. Add as many as your project needs (backend, frontend, mobile, infra, etc.)."*

For each repo, ask:
- Name (used in PR titles and comments)
- Relative path from the project root (e.g. `../my-api`, `../my-web`, `./`)
- Should speckit implement code changes in this repo? (yes/no, default: yes)
- Test command (e.g. `npm test`) — press Enter to skip
- Build command (e.g. `npm run build`) — press Enter to skip
- Check if `.github/workflows/` exists in that path. If it does, ask: *"Wait for CI checks to pass before merging PRs in `<repo-name>`? (yes/no)"* — save as `waitForCI`. If no workflows folder exists, set `waitForCI: false` silently.

Ask *"Add another repo? (yes/no)"* after each one.

### 4. Collect column names
Show the defaults and ask the user to confirm or override each:
- Todo (default: `Todo`)
- Refinement (default: `Refinement`)
- Ready (default: `Ready`)
- In Progress (default: `In Progress`)
- Done (default: `Done`)

Verify the columns exist by running:
```
gh project field-list <projectNumber> --owner <owner> --format json
```
If any configured column name doesn't match, warn the user and suggest corrections before continuing.

### 5. Choose models
Ask:
- Model for `/refine` (spec phase) — default: `claude-opus-4-6`
- Model for `/implement` (build phase) — default: `claude-sonnet-4-6`

### 6. Write `.specpilot.json`
Write the collected config to `.specpilot.json` in the project root. Follow `config.schema.json` for the exact shape.

### 7. Install skill files
Fetch the skill files from the cogloop repo and write them to `.claude/commands/`:

- `https://raw.githubusercontent.com/eduardo-caua/cogloop/main/plugins/specpilot/skills/refine.md` → `.claude/commands/refine.md`
- `https://raw.githubusercontent.com/eduardo-caua/cogloop/main/plugins/specpilot/skills/implement.md` → `.claude/commands/implement.md`

### 8. Update CLAUDE.md
Append a `## specpilot` section to `CLAUDE.md`:

```markdown
## specpilot

This project uses the [specpilot](https://github.com/eduardo-caua/cogloop/tree/main/plugins/specpilot) plugin from [cogloop](https://github.com/eduardo-caua/cogloop).

### Commands

- `/refine` — picks the next ticket from **Todo**, runs specify → clarify → plan → tasks, creates a spec branch + PR, moves ticket to **Ready**
- `/implement` — picks the next ticket from **Ready**, implements → reviews → tests → ships PRs, moves ticket to **Done**

### Flags

`--once` (process one ticket then stop), `--dry-run` (preview only)

### Config

See `.specpilot.json` in the project root.
```

### 9. Done
Print a summary:
- Config written to `.specpilot.json`
- Skills installed: `/refine`, `/implement`
- Spec location: `<specsDir>` (`<separate repo | folder>`)
- Feature repos: list each repo name
- GitHub Project: `<owner>/<projectNumber>`
- Next step: add tickets to the **Todo** column and run `/refine`
