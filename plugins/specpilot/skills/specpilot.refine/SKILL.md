---
name: specpilot.refine
description: Refine the next ticket from the Todo column using speckit. Use this when the user runs /specpilot.refine or asks to refine, spec, or plan the next ticket from the board.
argument-hint: "[--once] [--dry-run]"
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebFetch]
---

# /specpilot.refine

Pick the next ticket from the **To do** column, run it through speckit, and move it to **Ready**.

> Board-first flow: the ticket already exists on the GitHub Project board. Use `/specpilot.specify` instead if you want to start from a feature idea and create the ticket automatically.

## Arguments

`$ARGUMENTS` may contain:
- `--once` — process one ticket then stop
- `--dry-run` — show which ticket would be picked, do nothing

## Steps

### 0. Load config
Read `.specpilot.json` from the project root. If missing, tell the user to run `/specpilot.setup` first and stop.

Identify:
- The repo with `role: "spec"` → speckit writes artifacts here (`speckit.specsDir`)
- `speckit.workspaceDir` → where speckit slash commands run from

### 1. Pick ticket
- **Always fetch fresh**: Run the `gh project item-list` command every time this step is reached — including when looping back from Step 5. NEVER reuse a previously fetched ticket list.
- Query the GitHub Project for the next ticket in the **To do** column: `gh project item-list <projectNumber> --owner <owner> --format json`
- Filter items where `status` exactly matches `config.statuses.todo` (read from `.specpilot.json` — do NOT hardcode "Todo" or any other string; the actual value may contain spaces, e.g. "To do")
- If `assignee` is set in config, additionally filter by that assignee
- If no tickets found, say "No tickets in To do. All done!" and stop
- If `--dry-run`, print the ticket title and stop
- **Re-read the ticket**: After selecting the ticket, fetch its title and body fresh via `gh issue view <number> --repo <owner>/<repo> --json title,body,labels`. Do NOT rely on the `item-list` summary — the ticket may have been updated.
- Move the ticket to the **Refinement** column
- Assign the ticket to `config.github.assignee` using `gh project item-edit` with the assignee field (if `assignee` is set in config)

### 2. Classify: feature or bug
Read the ticket title and body. Determine whether the ticket describes:

- **Feature** — a new capability, enhancement, or significant change that needs its own spec
- **Bug** — a defect, regression, or incorrect behavior in an existing feature

**Classification signals** (check in order):
1. GitHub label: if the ticket has a `bug` label → **Bug**
2. Title keywords: "fix", "bug", "broken", "crash", "error", "regression", "doesn't work", "not working" → likely **Bug**
3. Body structure: references existing behavior that is wrong, describes expected vs actual → likely **Bug**
4. If ambiguous, default to **Feature**

**If Bug** → go to Step 2b (Bug routing)
**If Feature** → go to Step 3 (Run speckit — full pipeline)

### 2b. Bug routing — find the parent spec
Bugs belong in existing specs, not in new spec folders. Route the bug to the correct parent spec:

1. **List existing specs**: Read all `spec.md` files in the specs directory (`speckit.specsDir`), scanning recursively to support both flat structures (`specsDir/005-foo/spec.md`) and app subfolders (`specsDir/member/005-foo/spec.md`). For each spec, note:
   - Spec number and name (from folder name, e.g. `005-recurrent-transactions`)
   - App subfolder (if nested, e.g. `member`)
   - Overview/title (first heading in `spec.md`)
   - Key entities and modules mentioned

2. **Match the bug to a spec**: Compare the ticket description against each spec's scope. Look for:
   - Referenced files, modules, or components that match a spec's affected area
   - Feature names or domain concepts that align with a spec
   - If the ticket mentions specific functionality, match it to the spec that implemented it

3. **Confirm with user**: Present the match:
   > *"This looks like a bug in **[spec name]**. I'll add it to `[spec-folder]/spec.md` as a bug fix entry. Correct?"*
   - If user confirms → continue
   - If user says different spec → use that one
   - If user says "new spec" → fall through to Step 3 (Feature flow)

4. **Write the bug into the existing spec**: Append to the "Bug Fixes" section of the matched `spec.md`:
   - Assign the next bug ID following the spec's existing pattern (e.g. `BUG-R09`, `BUG-047`)
   - Include: priority, affected files (if known), symptom, fix approach, acceptance scenarios
   - Follow the same format as existing bugs in that spec

5. **Update the plan**: If `plan.md` exists for the matched spec, append a bug fix design section with root cause analysis and fix strategy. If the bug is simple enough (clear fix, no design decisions), this step can be skipped.

6. **Update tasks**: If `tasks.md` exists for the matched spec, append a new phase with implementation tasks for the bug fix. Follow the existing task format (task IDs, file paths, test tasks).

7. **Label the ticket as a bug**: Apply the `bug` label to the existing GitHub issue:
   - First, check if the `bug` label exists in the repo: `gh label list --repo <owner>/<repo> --json name`
   - If it doesn't exist, create it: `gh label create bug --repo <owner>/<repo> --color "d73a4a" --description "Something isn't working"`
   - Add the label: `gh issue edit <issue-number> --repo <owner>/<repo> --add-label "bug"`
   - Do NOT add a `[BUG]` prefix to the title — the label is the canonical marker

8. **Push spec branch and open PR**: Create a branch, commit, push, and open a PR — same as features:
   - `git checkout -b bugfix/<BUG-ID>-<short-description>` (e.g. `bugfix/BUG-D01-monthly-summary-overflow`)
   - `git add` the changed spec files and `git commit -m "<ticket title>"`
   - `git push origin <branch-name>`
   - If spec is a separate repo: `gh pr create --title "<ticket title>" --body "Bug fix spec for <ticket title>"`
   - Add the spec PR/branch link as a comment on the GitHub Issue
   - Go to Step 4 (Move ticket to Ready)

   **Exception**: Only push directly to main if the user explicitly asks for it.

### 3. Run speckit — full pipeline (Feature flow)
Run all four speckit commands sequentially from `speckit.workspaceDir`.

**App subfolder detection** → Before running specify, check if `speckit.specsDir` contains app subfolders (directories like `member/`, `admin/` that themselves contain numbered spec folders). If multiple app subfolders exist, ask the user which app this feature targets:
> *"I found multiple app folders in specs: **member**, **admin**. Which app is this feature for?"*
Store the chosen app name for passing to the create-new-feature script via `-App <name>`.

**specify** → Read the ticket title and body from GitHub. Run `/speckit.specify` passing the full ticket description as input. If an app subfolder was selected, the create-new-feature script will receive the `-App` parameter to create the spec in the correct subfolder.

**clarify** → Run `/speckit.clarify`. When clarify has questions:
- Do NOT block waiting for terminal input
- Surface each question here in chat as clear multiple-choice options
- Wait for the user's answers
- Encode answers back into the spec
- Continue with plan and tasks

**plan** → Run `/speckit.plan`

**tasks** → Run `/speckit.tasks`

### 3b. Push spec branch and open PR (Feature flow only)
In the spec location (`speckit.specsDir`):
- The branch was created by speckit during specify (e.g. `012-feature-name`)
- `git push origin <branch>`
- If spec is a separate repo: `gh pr create --title "<ticket title>" --body "Spec for <ticket title>"`
- Add the spec PR/branch link as a comment on the GitHub Issue

### 4. Move ticket to Ready
- Update ticket status to the value of `config.statuses.ready` in the GitHub Project
- Ensure `config.github.assignee` is still assigned to the ticket

### 5. Continue or stop
- If `--once`, stop
- Otherwise ask: *"Refine next ticket? (yes/no)"*
- If yes, go to Step 1. **Important**: you MUST re-query the project board from scratch in Step 1 — do NOT reuse any previously fetched ticket list or ticket data. If no, stop.

## Error handling
- If speckit fails at any step, report the error and stop — do not move to Ready
- If the spec branch already exists, pull latest and continue from the last completed step
- If no matching spec is found for a bug, ask the user which spec to use or whether to create a new one
