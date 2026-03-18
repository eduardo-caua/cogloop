# specpilot

> A Claude Code + GitHub Projects spec-driven development loop.

specpilot automates the full journey from GitHub ticket to merged PR using a spec-first methodology. It splits the work into two phases — **refine** (spec the ticket) and **implement** (build it) — each driven by a Claude Code slash command.

## Stack requirements

| Tool | Purpose |
|------|---------|
| [Claude Code](https://claude.ai/code) | Executes the skills |
| [GitHub Projects](https://docs.github.com/en/issues/planning-and-tracking-with-projects) | Board with columns for pipeline stages |
| [GitHub Issues](https://docs.github.com/en/issues) | Tickets |
| [speckit](https://github.com/speckit) | Spec generation (specify, clarify, plan, tasks) |
| `gh` CLI | GitHub API interactions |

## How it works

```
Todo → Refinement → Ready → In Progress → Done
        /refine                /implement
```

### `/refine`
1. Picks the next ticket from **Todo**
2. Moves it to **Refinement**
3. Runs `speckit.specify` → `speckit.clarify` → `speckit.plan` → `speckit.tasks` in one session
4. Surfaces any clarify questions to you directly in chat
5. Creates a branch + PR in your spec repo
6. Moves ticket to **Ready**
7. Asks: *"Refine next ticket?"*

### `/implement`
1. Picks the next ticket from **Ready**
2. Moves it to **In Progress**
3. Reads the spec from the open PR branch
4. Implements → reviews → tests → ships PRs for all feature repos
5. Updates `tasks.md` with completed checkboxes
6. Merges all PRs (feature repos + spec repo)
7. Adds PR links as a comment on the GitHub issue
8. Moves ticket to **Done**
9. Asks: *"Implement next ticket?"*

## Installation

### 1. Set up your GitHub Project board

Create a GitHub Project with these columns (exact names matter — you can customize in config):

- `Todo`
- `Refinement`
- `Ready`
- `In Progress`
- `Done`

### 2. Install the plugin

Copy the install skill into your project:

```bash
cp path/to/cogloop/plugins/specpilot/install.md .claude/commands/cogloop-install.md
```

### 3. Run the install skill

Open Claude Code in your project root and run:

```
/cogloop-install
```

The install skill will:
- Ask you for your GitHub org/user, project number, and repo names
- Copy `/refine` and `/implement` skills into `.claude/commands/`
- Write `.specpilot.json` config to your project root
- Update your `CLAUDE.md` with usage instructions

## Configuration

specpilot uses a `.specpilot.json` file in your project root. See [config.schema.json](./config.schema.json) for all options and [config.example.json](./config.example.json) for a ready-to-copy example.

## Usage

```
/refine              # Refine next Todo ticket, loop until empty
/refine --once       # Refine one ticket then stop
/refine --dry-run    # Show which ticket would be picked, do nothing

/implement           # Implement next Ready ticket, loop until empty
/implement --once    # Implement one ticket then stop
/implement --dry-run # Show which ticket would be picked, do nothing
```

## Requirements

- Claude Code with `--dangerously-skip-permissions` or appropriate tool permissions
- `gh` CLI authenticated (`gh auth login`)
- speckit installed and configured in your spec repo
- Git configured with push access to all repos
