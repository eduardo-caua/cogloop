# /implement

Implement the next ticket from the Ready column and ship it.

## Flags

- `--once` — process one ticket then stop (default: ask after each ticket)
- `--dry-run` — show which ticket would be picked, do nothing
- `--from <step>` — resume from a specific step (implement, review, test, ship, merge)

## Steps

### 0. Load config
Read `.specpilot.json` from the project root. If missing, tell the user to run `/cogloop-install` first and stop.

From config, identify:
- **Spec repo/folder**: the entry with `role: "spec"` — read-only during implement, updated last
- **Feature repos**: all entries with `role: "feature"` — these get implemented, tested, built, and shipped
- Per repo: check `implement`, `test`, `build`, and `waitForCI` flags to know what to do

### 1. Pick ticket
- Query the GitHub Project for the next ticket in the **Ready** column
- If `assignee` is set in config, filter by that assignee
- If no tickets found, say "No tickets in Ready. All done!" and stop
- If `--dry-run`, print the ticket title and stop
- Move the ticket to the **In Progress** column

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
speckit reads the tasks and implements them across all feature repos.

Only repos with `implement: true` (the default) receive code changes.
Repos with `implement: false` still get a PR opened but no code changes are made by speckit.

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

### 8. Update tasks.md
In the spec location:
- Check off completed tasks in `tasks.md`
- `git add tasks.md && git commit -m "Mark tasks complete: <ticket title>"`
- `git push origin <branch-name>`

### 9. Wait for CI
For each open PR:
- If `waitForCI: false` for that repo → skip, proceed to merge
- If `waitForCI: true` → poll `gh pr checks` until all checks pass (timeout: 30 minutes)
  - If checks fail → attempt to fix and push again (up to 2 attempts)
  - If `gh pr checks` returns no checks → treat as pass and continue
  - If still running after timeout → warn the user and ask: *"Wait longer or merge anyway?"*

### 10. Merge all PRs
Merge feature repos first, then the spec location last:
- `gh pr merge <number> --squash --delete-branch`

If the spec location is a folder inside the project (not a separate repo), commit and push directly to main instead of opening a PR.

### 11. Add comment to issue
Post a comment on the GitHub Issue listing all merged PRs:
```
PRs merged:
- [repo-name#N](url)
- [spec-repo#N](url)
```

### 12. Move ticket to Done
Update the ticket status to **Done** in the GitHub Project.

### 13. Continue or stop
- If `--once` flag was set, stop
- Otherwise ask: *"Implement next ticket? (yes/no)"*
- If yes, go back to Step 1
- If no, stop

## Error handling
- If any step fails, report clearly with the step name and error
- Do not move ticket to Done unless all PRs are merged
- On CI failure: attempt fix and repush before giving up
- On merge conflict: pull latest main, rebase the feature branch, then retry
