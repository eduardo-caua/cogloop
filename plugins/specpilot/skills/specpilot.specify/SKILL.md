---
name: specpilot.specify
description: Start from a feature idea or bug report, run it through speckit, and create the GitHub ticket automatically. Use this when the user runs /specpilot.specify or describes a feature they want to build.
argument-hint: "[--dry-run] \"<feature or bug description>\""
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebFetch]
---

# /specpilot.specify

Start from a feature idea or bug report, run it through speckit to produce a full spec, then create the GitHub ticket and subtasks automatically — ready for `/specpilot.implement` to pick up.

> Spec-first flow: you drive from an idea, Claude creates the ticket. Use `/specpilot.refine` instead if you want to pick up an existing ticket from the board.

## Arguments

`$ARGUMENTS` may contain:
- `--dry-run` — run speckit and show what ticket would be created, but don't create it
- A feature or bug description inline (e.g. `/specpilot.specify "add dark mode toggle"`) — skip the prompt

## Steps

### 0. Load config
Read `.specpilot.json` from the project root. If missing, tell the user to run `/specpilot.setup` first and stop.

Identify:
- The repo with `role: "spec"` → speckit writes artifacts here
- `speckit.workspaceDir` → where speckit slash commands run from
- `github.owner`, `github.projectNumber` → where to create the ticket

### 1. Get the description
- If a description was passed inline in `$ARGUMENTS`, use it
- Otherwise ask: *"What do you want to build or fix? Describe the feature or bug — as much or as little detail as you have."*
- Wait for the user's input

### 2. Classify: feature or bug
Analyze the description to determine whether it is:

- **Feature** — a new capability, enhancement, or significant change that needs its own spec
- **Bug** — a defect, regression, or incorrect behavior in an existing feature

**Classification signals** (check in order):
1. Explicit keywords: "fix", "bug", "broken", "crash", "error", "regression", "doesn't work", "not working" → likely **Bug**
2. Description structure: describes expected vs actual behavior, references existing functionality that is wrong → likely **Bug**
3. If the description talks about building something new, adding a feature, or changing how things work significantly → **Feature**
4. If ambiguous, ask the user: *"Is this a bug in an existing feature, or a new feature/enhancement?"*

**If Bug** → go to Step 2b (Bug routing)
**If Feature** → go to Step 3 (Run speckit — full pipeline)

### 2b. Bug routing — find the parent spec
Bugs belong in existing specs, not in new spec folders. Route the bug to the correct parent spec:

1. **List existing specs**: Read all `spec.md` files in the specs directory (`speckit.specsDir`). For each spec, note:
   - Spec number and name (from folder name, e.g. `005-recurrent-transactions`)
   - Overview/title (first heading in `spec.md`)
   - Key entities and modules mentioned

2. **Match the bug to a spec**: Compare the description against each spec's scope. Look for:
   - Referenced files, modules, or components that match a spec's affected area
   - Feature names or domain concepts that align with a spec
   - If the description mentions specific functionality, match it to the spec that implemented it

3. **Confirm with user**: Present the match:
   > *"This looks like a bug in **[spec name]**. I'll add it to `[spec-folder]/spec.md` as a bug fix entry. Correct?"*
   - If user confirms → continue
   - If user says different spec → use that one
   - If user says "new spec" → fall through to Step 3 (Feature flow)

4. **Write the bug into the existing spec**: Append to the "Bug Fixes" section of the matched `spec.md`:
   - Assign the next bug ID following the spec's existing pattern (e.g. `BUG-R09`, `BUG-047`)
   - Include: priority, affected files (if known), symptom, fix approach, acceptance scenarios
   - Follow the same format as existing bugs in that spec

5. **Update the plan**: If `plan.md` exists for the matched spec, append a bug fix design section with root cause analysis and fix strategy (run `/speckit.plan` for the matched spec). If the bug is simple enough (clear fix, no design decisions), this step can be skipped.

6. **Update tasks**: If `tasks.md` exists for the matched spec, append a new phase with implementation tasks for the bug fix (run `/speckit.tasks` for the matched spec). Follow the existing task format (task IDs, file paths, test tasks).

7. Go to Step 4b (Create bug ticket)

### 3. Run speckit — full pipeline (Feature flow)
Run all four speckit commands sequentially from `speckit.workspaceDir`, using the feature description as input.

**specify** → Run `/speckit.specify "<feature description>"`

**clarify** → Run `/speckit.clarify`. When clarify has questions:
- Surface each question here in chat as clear multiple-choice options
- Wait for the user's answers
- Encode answers back into the spec
- Continue with plan and tasks

**plan** → Run `/speckit.plan`

**tasks** → Run `/speckit.tasks`

### 3b. Derive ticket content from spec artifacts (Feature flow)
Read the generated `spec.md` and `tasks.md`. Extract:
- **Title** — a short, clear ticket title from the spec (one sentence)
- **Body** — a summary of what this feature does (2–4 sentences from spec.md)
- **Subtasks** — each top-level task from `tasks.md` becomes a GitHub sub-issue or task checklist item

### 4. Resolve project title
Both feature and bug flows need the project **title** (not number) for `gh issue create --project`.
- Resolve it once: `gh project list --owner <owner> --format json`, then find the project whose `number` matches `config.github.projectNumber` and extract its `title`
- Cache the title for the rest of this run

### 4a. Create the GitHub ticket (Feature flow)
- If `--dry-run`, print the ticket title, body, and subtasks — stop here
- Determine the target repo: use the first `role: "feature"` repo from config. If none, use the spec repo.
- Create a real issue linked to the project:
  `gh issue create --repo <owner>/<repo> --title "<title>" --body "<body>" --project "<project title>"`
- For each subtask, append a task list checkbox (`- [ ] ...`) to the issue body
- Set the ticket status to **Ready** (use `config.statuses.ready`):
  `gh project item-edit --project-id <projectId> --id <itemId> --field-id <statusFieldId> --single-select-option-id <readyOptionId>`
- **Important**: use `config.github.owner` for the `--owner` flag, NOT the current user's login. The project belongs to the org/owner, not the individual user.

### 4b. Create the GitHub ticket (Bug flow)
- If `--dry-run`, print what would be created and stop
- Determine the target repo: use the first `role: "feature"` repo from config. If none, use the spec repo.
- **Ensure the `bug` label exists** in the target repo:
  - Check: `gh label list --repo <owner>/<repo> --json name`
  - If `bug` label is missing, create it: `gh label create bug --repo <owner>/<repo> --color "d73a4a" --description "Something isn't working"`
- **Create a real issue with label and project linked**:
  `gh issue create --repo <owner>/<repo> --title "<short description>" --body "<body>" --label "bug" --project "<project title>"`
  - **Title**: Use a clean title — do NOT add a `[BUG]` prefix. The `bug` label is the canonical marker.
  - **Body**: Bug description, affected spec reference (`Part of spec: [spec-name]`), and the bug ID assigned in the spec (e.g. `BUG-R09`)
- If `gh issue create` with `--label` fails (e.g. permissions), fall back to creating without label and add `[BUG]` prefix to the title instead
- Set the ticket status to **Ready** (use `config.statuses.ready`)
- **Important**: use `config.github.owner` for the `--owner` flag

### 5. Push changes
**Feature flow**: In the spec location (`speckit.specsDir`):
- The branch was created by speckit during specify (e.g. `013-feature-name`)
- `git push origin <branch>`
- If spec is a separate repo: `gh pr create --title "<ticket title>" --body "Spec for <ticket title>"`

**Bug flow**: Commit the spec/plan/task updates to the current branch (or main) and push:
- `git add <modified spec files>`
- `git commit -m "Add bug fix <BUG-ID> to <spec-name>"`
- `git push`

### 6. Done
Print:
- Ticket created on project board: `<owner>#<projectNumber>` — `<title>`
- **Feature flow**: Spec branch: `<branch-name>`, Subtasks: N tasks added
- **Bug flow**: Bug added to existing spec: `<spec-folder>/spec.md` as `<BUG-ID>`
- Next step: run `/specpilot.implement` to build it

## Error handling
- If speckit fails at any step, report the error and stop — do not create the ticket
- If the spec branch already exists, pull latest and continue from the last completed step
- If no matching spec is found for a bug, ask the user which spec to use or whether to create a new one
