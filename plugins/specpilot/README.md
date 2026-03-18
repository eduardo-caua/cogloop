# specpilot

> A Claude Code + GitHub Projects spec-driven development loop.

specpilot automates the full journey from GitHub ticket to merged PR using a spec-first methodology. It splits the work into two phases — **refine** (spec the ticket) and **implement** (build it) — each driven by a Claude Code slash command.

## Stack requirements

| Tool | Purpose |
|------|---------|
| [Claude Code](https://claude.ai/code) | Executes the skills |
| [GitHub Projects](https://docs.github.com/en/issues/planning-and-tracking-with-projects) | Board with pipeline columns |
| [GitHub Issues](https://docs.github.com/en/issues) | Tickets |
| [speckit](https://github.com/speckit) | Spec generation (specify, clarify, plan, tasks) |
| `gh` CLI | GitHub API interactions |

## How it works

```
Todo → Refinement → Ready → In Progress → Done
        /refine                /implement
```

### `/refine`
1. Picks the next ticket from **Todo**, moves it to **Refinement**
2. Runs `speckit.specify` → `speckit.clarify` → `speckit.plan` → `speckit.tasks` in one session
3. Surfaces clarify questions to you directly in chat — no terminal blocking
4. Pushes the spec branch, opens a PR (if spec is a separate repo)
5. Moves ticket to **Ready**
6. Asks: *"Refine next ticket?"*

### `/implement`
1. Picks the next ticket from **Ready**, moves it to **In Progress**
2. Reads the spec from the branch
3. Implements across all configured feature repos
4. Reviews, tests, and builds
5. Ships PRs, waits for CI (per-repo, optional), merges
6. Updates `tasks.md`, adds PR links to the issue
7. Moves ticket to **Done**
8. Asks: *"Implement next ticket?"*

## Repo setup — flexible for any project shape

specpilot works with any number of repos. You define each one in `.specpilot.json` with a `role` and capability flags:

| Role | Behavior |
|------|---------|
| `spec` | Where speckit writes artifacts. Read during implement, updated last. Exactly one per project. |
| `feature` | Any repo that gets implemented — backend, frontend, mobile, infra, docs, etc. |

### The spec location

The spec can live in **two places**:

**Separate repo** (recommended for multi-repo projects)
```
my-org/
  my-project-spec/    ← role: "spec" — speckit artifacts live here
  my-api/             ← role: "feature"
  my-web/             ← role: "feature"
```

**Folder inside the project** (simpler for single-repo projects)
```
my-project/
  spec/               ← role: "spec", path: "./spec"
  src/
```

The default folder name is `spec/`. For separate repos, the convention is `<project-name>-spec`.

### Example configs

**Two repos (API + Web) with separate spec repo:**
```json
{
  "repos": [
    { "name": "my-spec", "path": "../my-spec", "role": "spec" },
    { "name": "my-api",  "path": "../my-api",  "role": "feature", "test": "npm test", "build": "npm run build", "waitForCI": true },
    { "name": "my-web",  "path": "../my-web",  "role": "feature", "test": "npm test", "build": "npm run build", "waitForCI": true }
  ]
}
```

**Single repo with spec folder:**
```json
{
  "repos": [
    { "name": "spec", "path": "./spec", "role": "spec" }
  ],
  "speckit": { "specsDir": "./spec", "workspaceDir": "." }
}
```

**Five repos (monorepo-adjacent):**
```json
{
  "repos": [
    { "name": "spec",    "path": "../spec",    "role": "spec" },
    { "name": "api",     "path": "../api",     "role": "feature", "test": "npm test",    "build": "npm run build", "waitForCI": true },
    { "name": "web",     "path": "../web",     "role": "feature", "test": "npm test",    "build": "npm run build", "waitForCI": true },
    { "name": "mobile",  "path": "../mobile",  "role": "feature", "test": "yarn test",   "build": "yarn build",    "waitForCI": true },
    { "name": "infra",   "path": "../infra",   "role": "feature", "implement": false,    "waitForCI": false }
  ]
}
```

> `implement: false` means speckit won't touch that repo — but a PR is still opened for any manual infrastructure changes.

## Installation

### 1. Set up your GitHub Project board

Create a GitHub Project with these columns (names are configurable):

- `Todo` → `Refinement` → `Ready` → `In Progress` → `Done`

### 2. Add the cogloop marketplace

```bash
claude plugin marketplace add https://github.com/eduardo-caua/cogloop
```

### 3. Install specpilot

```bash
claude plugin install specpilot
```

### 4. Run setup in Claude Code

```
/specpilot-setup
```

The setup skill will:
- Walk you through config (GitHub project, repos, columns, models)
- Auto-detect CI presence per repo and ask about `waitForCI`
- Write `.specpilot.json` to your project root
- Update your `CLAUDE.md` with usage instructions

## Usage

```
/refine              # Refine next Todo ticket, ask after each
/refine --once       # Refine one ticket then stop
/refine --dry-run    # Show which ticket would be picked, do nothing

/implement              # Implement next Ready ticket, ask after each
/implement --once       # Implement one ticket then stop
/implement --dry-run    # Show which ticket would be picked, do nothing
/implement --from test  # Resume from the test step
```

## Requirements

- Claude Code with appropriate tool permissions
- `gh` CLI authenticated (`gh auth login`)
- speckit installed and configured
- Git configured with push access to all repos
