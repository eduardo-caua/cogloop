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
- Query the GitHub Project for the next ticket in the **To do** column: `gh project item-list <projectNumber> --owner <owner> --format json`
- Filter items where `status` exactly matches `config.statuses.todo` (read from `.specpilot.json` — do NOT hardcode "Todo" or any other string; the actual value may contain spaces, e.g. "To do")
- If `assignee` is set in config, additionally filter by that assignee
- If no tickets found, say "No tickets in To do. All done!" and stop
- If `--dry-run`, print the ticket title and stop
- Move the ticket to the **Refinement** column
- Assign the ticket to `config.github.assignee` using `gh project item-edit` with the assignee field (if `assignee` is set in config)

### 2. Run speckit in one session
Run all four speckit commands sequentially from `speckit.workspaceDir`.

**specify** → Read the ticket title and body from GitHub. Run `/speckit.specify` passing the full ticket description as input.

**clarify** → Run `/speckit.clarify`. When clarify has questions:
- Do NOT block waiting for terminal input
- Surface each question here in chat as clear multiple-choice options
- Wait for the user's answers
- Encode answers back into the spec
- Continue with plan and tasks

**plan** → Run `/speckit.plan`

**tasks** → Run `/speckit.tasks`

### 3. Push spec branch and open PR
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
- If yes, go to Step 1. If no, stop.

## Error handling
- If speckit fails at any step, report the error and stop — do not move to Ready
- If the spec branch already exists, pull latest and continue from the last completed step
