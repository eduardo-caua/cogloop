---
name: specpilot.implement
description: Implement the next ticket from the Ready column and ship it. Use this when the user runs /specpilot.implement or asks to implement, build, or ship the next ticket.
argument-hint: "[--once] [--dry-run] [--from <step>]"
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebFetch]
---

# /specpilot.implement

Implement the next ticket from the **Ready** column and ship it.

> Works with tickets created by either `/specpilot.refine` (board-first) or `/specpilot.specify` (spec-first) — both land in **Ready** with a linked spec branch.

## Arguments

`$ARGUMENTS` may contain:
- `--once` — process one ticket then stop
- `--dry-run` — show which ticket would be picked, do nothing
- `--from <step>` — resume from a specific step: `implement`, `review`, `test`, `ship`, `merge`

## Steps

### 0. Load config
Read `.specpilot.json` from the project root. If missing, tell the user to run `/specpilot.setup` first and stop.

From config, identify:
- **Spec location**: the repo/folder with `role: "spec"` — read-only during implement
- **Feature repos**: all repos with `role: "feature"` — implement, test, build, and ship these
- Per repo: check `implement`, `test`, `build`, and `waitForCI` flags

### 1. Pick ticket
- Query the GitHub Project for the next ticket in the **Ready** column: `gh project item-list <projectNumber> --owner <owner> --format json`
- Filter items where `status` exactly matches `config.statuses.ready` (read from `.specpilot.json` — do NOT hardcode "Ready" or any other string)
- If `assignee` is set in config, additionally filter by that assignee
- If no tickets found, say "No tickets in Ready. All done!" and stop
- If `--dry-run`, print the ticket title and stop
- Move the ticket to **In Progress** (use `config.statuses.inProgress` value)
- Assign the ticket to `config.github.assignee` using `gh project item-edit` with the assignee field (if `assignee` is set in config)

### 2. Read spec
- Find the open PR (or branch) in the spec location for this ticket (e.g. branch `012-feature-name`)
- Check out that branch in the spec location
- Read `spec.md`, `plan.md`, and `tasks.md`
- The spec location is **read-only** at this stage — do not commit yet

### 3. Checkout feature branches
For each feature repo (`role: "feature"`):
- `git checkout main && git pull origin main`
- `git checkout -b <branch-name>` (same branch name as the spec branch)

### 4. Implement
Run `/speckit.implement <branch-name>` from `speckit.workspaceDir`.
Only repos with `implement: true` (default) receive code changes.
Repos with `implement: false` still get a PR but no speckit changes.

### 5. Code review
Review changes in each feature repo for:
- Correctness against spec requirements
- Adherence to project constitution (if `.specify/memory/constitution.md` exists in the spec location)
- Test coverage
- No obvious bugs or security issues

Apply any fixes found.

### 6. Test and build
For each feature repo:
- If `test` is set → run it (up to 3 attempts, apply fixes on failure)
- If `build` is set → run it (up to 3 attempts, apply fixes on failure)
- If still failing after 3 attempts, report the error and stop

### 7. Ship — commit, push, open PRs
For each feature repo with changes:
- `git add -A && git commit -m "<ticket title>"`
- `git push origin <branch-name>`
- `gh pr create --title "<ticket title>" --body "Closes #<issue-number>"`

### 8. Wait for CI
For each open PR:
- If `waitForCI: false` → skip, proceed to merge
- If `waitForCI: true` → poll `gh pr checks` until all checks pass (timeout: 30 min)
  - If checks fail → attempt to fix and push again (up to 2 attempts)
  - If no checks reported → treat as pass and continue
  - If still running after timeout → ask: *"Wait longer or merge anyway?"*

### 9. Merge all PRs
Merge feature repos first, then spec location last:
- `gh pr merge <number> --squash --delete-branch`

If spec is a folder inside the project (not a separate repo), commit and push directly to main instead.

### 10. Add comment to issue
Post a comment on the GitHub Issue:
```
PRs merged:
- [repo-name#N](url)
- [spec-repo#N](url)
```

### 11. Move ticket to Done
- Update ticket status to the value of `config.statuses.done` in the GitHub Project
- Ensure `config.github.assignee` is still assigned to the ticket

### 12. Continue or stop
- If `--once`, stop
- Otherwise ask: *"Implement next ticket? (yes/no)"*
- If yes, go to Step 1. If no, stop.

## Error handling
- If any step fails, report clearly with the step name and error
- Do not move to Done unless all PRs are merged
- On CI failure: attempt fix and repush before giving up
- On merge conflict: pull latest main, rebase the feature branch, then retry
