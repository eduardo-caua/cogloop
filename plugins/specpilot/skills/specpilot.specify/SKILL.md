---
name: specpilot.specify
description: Start from a feature idea, run it through speckit, and create the GitHub ticket automatically. Use this when the user runs /specpilot.specify or describes a feature they want to build.
argument-hint: "[--dry-run] \"<feature description>\""
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebFetch]
---

# /specpilot.specify

Start from a feature idea, run it through speckit to produce a full spec, then create the GitHub ticket and subtasks automatically — ready for `/specpilot.implement` to pick up.

> Spec-first flow: you drive from an idea, Claude creates the ticket. Use `/specpilot.refine` instead if you want to pick up an existing ticket from the board.

## Arguments

`$ARGUMENTS` may contain:
- `--dry-run` — run speckit and show what ticket would be created, but don't create it
- A feature description inline (e.g. `/specpilot.specify "add dark mode toggle"`) — skip the prompt

## Steps

### 0. Load config
Read `.specpilot.json` from the project root. If missing, tell the user to run `/specpilot.setup` first and stop.

Identify:
- The repo with `role: "spec"` → speckit writes artifacts here
- `speckit.workspaceDir` → where speckit slash commands run from
- `github.owner`, `github.projectNumber` → where to create the ticket

### 1. Get the feature description
- If a description was passed inline in `$ARGUMENTS`, use it
- Otherwise ask: *"What do you want to build? Describe the feature — as much or as little detail as you have."*
- Wait for the user's input

### 2. Run speckit in one session
Run all four speckit commands sequentially from `speckit.workspaceDir`, using the feature description as input.

**specify** → Run `/speckit.specify "<feature description>"`

**clarify** → Run `/speckit.clarify`. When clarify has questions:
- Surface each question here in chat as clear multiple-choice options
- Wait for the user's answers
- Encode answers back into the spec
- Continue with plan and tasks

**plan** → Run `/speckit.plan`

**tasks** → Run `/speckit.tasks`

### 3. Derive ticket content from spec artifacts
Read the generated `spec.md` and `tasks.md`. Extract:
- **Title** — a short, clear ticket title from the spec (one sentence)
- **Body** — a summary of what this feature does (2–4 sentences from spec.md)
- **Subtasks** — each top-level task from `tasks.md` becomes a GitHub sub-issue or task checklist item

### 4. Create the GitHub ticket
- If `--dry-run`, print the ticket title, body, and subtasks — stop here
- Create the issue: `gh issue create --repo <owner>/<repo> --title "<title>" --body "<body>"`
- For each subtask, create a sub-issue or append a task list to the issue body
- Add the issue to the GitHub Project board in the **Ready** column directly (skipping To do → Refinement since speckit already ran)

### 5. Push spec branch and link to ticket
In the spec location (`speckit.specsDir`):
- The branch was created by speckit during specify (e.g. `013-feature-name`)
- `git push origin <branch>`
- If spec is a separate repo: `gh pr create --title "<ticket title>" --body "Spec for #<issue-number>"`
- Add the spec PR/branch link as a comment on the GitHub Issue

### 6. Done
Print:
- Ticket created: `<owner>/<repo>#<number>` — `<title>`
- Spec branch: `<branch-name>`
- Subtasks: N tasks added
- Next step: run `/specpilot.implement` to build it

## Error handling
- If speckit fails at any step, report the error and stop — do not create the ticket
- If the spec branch already exists, pull latest and continue from the last completed step
