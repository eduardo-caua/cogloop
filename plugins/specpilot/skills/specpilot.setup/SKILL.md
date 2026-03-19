---
name: specpilot.setup
description: Set up the specpilot plugin in the current project. Use this when the user runs /specpilot.setup or asks to install, configure, or set up specpilot.
argument-hint: (no arguments needed)
allowed-tools: [Read, Write, Edit, Bash, Glob, WebFetch]
---

# /specpilot.setup

Set up specpilot in the current project. Collects your project configuration and makes `/specpilot.refine`, `/specpilot.specify`, and `/specpilot.implement` ready to use.

---

## Step 0 — Prerequisites (hard stop if any fail)

Run these checks silently before anything else:

1. **gh CLI authenticated** — run `gh auth status`. If not authenticated, stop:
   > "Please run `gh auth login` first, then re-run `/specpilot.setup`."

2. **speckit commands available** — check that the following slash commands are available in this session (listed in the system-reminder's available skills):
   `/speckit.constitution`, `/speckit.specify`, `/speckit.clarify`, `/speckit.plan`, `/speckit.tasks`
   If any are missing, stop:
   > "speckit commands are required but not available in this session. Make sure the speckit skills are installed as custom slash commands in your Claude Code configuration, then restart the session and re-run `/specpilot.setup`."

3. **Create `.claude/commands/`** if it doesn't exist.

---

## Step 1 — Silent detection

Detect the current project state to decide which flow to run:

- Run `git remote get-url origin` → infer GitHub owner and repo name
- Check for existing code repos: subdirectories or siblings containing `package.json`, `pom.xml`, `go.mod`, `Cargo.toml`, or similar
- Check for a constitution: `spec-kit/constitution.md`, `./spec/constitution.md`, or any `*constitution*` file under common spec folders
- Check for an existing `.specpilot.json`

**Route:**
- If constitution exists AND at least one code repo detected → **Flow A (Existing project)**
- Otherwise → **Flow B (New project)**

Tell the user which flow you're running before proceeding.

---

## Flow A — Existing project

### A1 — Guess everything, present once

Detect silently:

**GitHub:**
- Owner from `git remote get-url origin`
- Project number: look in `.specpilot.json`, `CLAUDE.md`, or `README.md` for hints. If not found, leave blank.
- Assignee: run `gh api user --jq .login` → use as default

**Spec location:**
- Look for `spec-kit/`, `spec/`, or a sibling `*-spec` repo at `../`
- Default `workspaceDir` to `.`; use `..` if `../CLAUDE.md` exists

**Repos:**
- List subfolders and `../` siblings with recognizable project structures
- For each: detect test command from `package.json` scripts (`test`, `test:ci`), `Makefile`, `pom.xml`
- Detect build command from `package.json` scripts (`build`)
- Check `.github/workflows/` → default `waitForCI: true` if found

**Columns:**
- Run `gh project field-list <projectNumber> --owner <owner> --format json` if project number is known
- Match to standard names. If project number unknown, default to: `Todo`, `Refinement`, `Ready`, `In Progress`, `Done`

**Models:** default `claude-opus-4-6` / `claude-sonnet-4-6`

### A2 — Present summary, ask once

Show a formatted summary of everything detected. Example:

---
I detected this is an existing project. Here's what I found — let me know if anything needs changing:

**GitHub**
- Owner: `brivioapp`
- Project number: `4`
- Assignee filter: `eduardo-caua`

**Spec**
- Location: `./spec-kit` (folder inside this project)
- speckit runs from: `.` (project root)

**Repos**
- `brivio-spec` → `../brivio-spec` (spec role, no CI)
- `finance-api` → `./finance-api` (feature, test: `npm test`, build: `npm run build`, CI: yes)
- `finance-web` → `./finance-web` (feature, test: `npm test`, build: `npm run build`, CI: yes)

**Project columns**
- To do → Refinement → Ready → In progress → Done

**Models**
- `/refine`: `claude-opus-4-6`
- `/implement`: `claude-sonnet-4-6`

Does this look right? Let me know anything you'd like to change or add.

---

Wait for the user's reply. Apply corrections if needed. Ask a single follow-up only if a correction is ambiguous.

### A3 — Constitution check

After confirmation, check if constitution exists:
- If yes → continue to A4
- If no → tell the user:
  > "No constitution found. I'll run `/speckit.constitution` now to define your project's conventions — this helps `/refine` generate consistent, high-quality specs."
  > Then invoke `/speckit.constitution` inline before continuing.

### A4 — Write config and finish

Write `.specpilot.json` and update `CLAUDE.md` (see schema below), then print the Done summary.

---

## Flow B — New project

### B1 — Ask about the product

Ask a single open question:

> "Tell me about what you're building — what does it do, who is it for, and do you have any preferences on technology (language, framework, database)? If you're not sure about the tech, I'll suggest something."

Wait for the user's answer.

### B2 — Suggest architecture in plain language

Based on the answer, suggest an architecture without using jargon. Frame it around what the user described, not technical terms. Example:

> "Based on what you described, here's what I'd suggest:
>
> - A **web app** your users open in the browser (React)
> - An **API** that handles the business logic and data (Node.js + NestJS)
> - A **database** to store everything (PostgreSQL)
>
> These would live as two folders inside one project:
> - `web/` — the browser app
> - `api/` — the backend
>
> Does this sound right? You can change anything — the technology, the names, or the structure."

Wait for confirmation or corrections. Apply changes if needed.

### B3 — Ask how far to go

> "How much do you want me to set up?
>
> **A) Just the config** — I'll write `.specpilot.json` and the project constitution. You handle the code setup yourself.
> **B) Scaffold the project** — I'll also create the folders and run the framework CLI tools to generate starter code.
> **C) Everything** — Scaffold + create GitHub repositories + set up CI pipelines for tests.
>
> Which sounds right? (A / B / C)"

Wait for the user's choice.

### B4 — GitHub project board

Ask:
> "What's the name of your GitHub organization or username? And do you already have a GitHub Project board set up for this project, or should I create one?"

- If board exists → ask for the project number
- If not → run `gh project create --owner <owner> --title "<project name>"` and note the project number

### B5 — Run speckit constitution

> "Now let's define your project's conventions. I'll ask you a few questions about how you like to work — this guides everything that comes after."

Invoke `/speckit.constitution` inline. This captures the product vision, coding conventions, and tech stack in detail.

### B6 — Execute chosen scope (B or C only)

**If B or C:**
- Create project folder structure
- Run appropriate CLI tools based on the chosen stack:
  - React → `npx create-react-app <name>` or `npm create vite@latest <name>`
  - NestJS → `npx @nestjs/cli new <name>`
  - Next.js → `npx create-next-app <name>`
  - Spring Boot → suggest `start.spring.io` or use `spring init`
  - Other → use the standard CLI for the detected framework
- Ask confirmation before running each CLI tool

**If C (also):**
- For each repo that doesn't exist on GitHub: ask *"Create `<repo-name>` on GitHub? (public/private)"* then run `gh repo create`
- Generate GitHub Actions workflow files appropriate for the stack:
  - Node/npm → workflow with `npm ci` + `npm test` + `npm run build`
  - Java/Maven → workflow with `mvn test`
  - Go → workflow with `go test ./...`
  - Other → generate a sensible default based on the language
- Commit and push initial scaffolding

### B7 — Write config and finish

Write `.specpilot.json` and update `CLAUDE.md` (see schema below), then print the Done summary.

---

## Config schema

`.specpilot.json`:

```json
{
  "github": { "owner": "...", "projectNumber": N, "assignee": "..." },
  "repos": [
    { "name": "my-spec", "path": "../my-spec", "role": "spec", "waitForCI": false },
    { "name": "my-api",  "path": "./api",  "role": "feature", "implement": true, "test": "npm test", "build": "npm run build", "waitForCI": true }
  ],
  "speckit": { "specsDir": "./spec-kit", "workspaceDir": "." },
  "statuses": { "todo": "Todo", "refinement": "Refinement", "ready": "Ready", "inProgress": "In Progress", "done": "Done" },
  "models": { "refine": "claude-opus-4-6", "implement": "claude-sonnet-4-6" }
}
```

`CLAUDE.md` section to append (skip if `## specpilot` already exists):

```markdown
## specpilot

This project uses the [specpilot](https://github.com/eduardo-caua/cogloop) plugin.

### Commands
- `/specpilot.refine` — picks next To do ticket from the board, runs speckit, moves to Ready
- `/specpilot.specify` — start from a feature idea, runs speckit, creates ticket in Ready
- `/specpilot.implement` — picks next Ready ticket, implements→tests→ships, moves to Done

### Flags
`--once` (one ticket then stop), `--dry-run` (preview only)

### Config
See `.specpilot.json` in the project root.
```

---

## Done summary (both flows)

Print:
- Config written to `.specpilot.json`
- Skills available: `/specpilot.refine`, `/specpilot.specify`, `/specpilot.implement`
- Spec location: `<specsDir>`
- Feature repos: list each with role and CI status
- GitHub Project: `<owner>/#<projectNumber>`
- Next step:
  - Board-first: add tickets to the **To do** column and run `/specpilot.refine`
  - Spec-first: run `/specpilot.specify` and describe what you want to build
