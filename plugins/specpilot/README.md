# specpilot

> A Claude Code + GitHub Projects spec-driven development loop.

specpilot automates the full journey from GitHub ticket to merged PR using a spec-first methodology. It splits the work into two phases — **refine** (spec the ticket) and **implement** (build it) — each driven by a Claude Code slash command.

Works with **existing projects** and **brand new projects** — the setup skill detects which situation you're in and guides you through the right flow.

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

## Installation

### 1. Install speckit

speckit is a hard requirement — specpilot's `/refine` command uses it to generate specs, plans, and tasks.

```bash
claude plugin marketplace add <speckit-marketplace-url>
claude plugin install speckit
```

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

The setup skill detects your situation and runs one of two flows:

---

## Setup flows

### Flow A — Existing project

If you already have code repos and a constitution, setup runs in "guess-first" mode:

1. Silently detects your GitHub owner, repos, spec location, CI setup, and project columns
2. Presents a single summary of everything it found
3. Asks once: *"Does this look right? Anything to change or add?"*
4. Applies corrections, writes `.specpilot.json`, done

No lengthy Q&A — one confirmation round and you're set up.

If a constitution is missing, it will run `/speckit.constitution` inline before finishing.

---

### Flow B — New project

If you're starting from scratch, setup acts as a project wizard:

1. Asks: *"Tell me about what you're building"* — product description + any tech preferences
2. Suggests an architecture in plain language (web app, API, database — no jargon)
3. Asks how far to go:
   - **Config only** — writes `.specpilot.json` and constitution, you handle the rest
   - **Scaffold** — also creates folders and runs framework CLI tools
   - **Everything** — scaffold + creates GitHub repos + sets up CI pipelines
4. Runs `/speckit.constitution` to capture your project's conventions and standards
5. Scaffolds and configures based on your chosen scope
6. Writes `.specpilot.json` and you're ready to add tickets

---

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

**Five repos (multi-repo):**
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
- speckit installed (`claude plugin install speckit`)
- Git configured with push access to all repos
