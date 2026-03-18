# /refine

Refine the next ticket from the Todo column using speckit, then move it to Ready.

## Flags

- `--once` — process one ticket then stop (default: ask after each ticket)
- `--dry-run` — show which ticket would be picked, do nothing

## Steps

### 0. Load config
Read `.specpilot.json` from the project root. If missing, tell the user to run `/cogloop-install` first and stop.

Identify the spec location from config:
- Find the repo with `role: "spec"` — this is where speckit writes artifacts
- `speckit.specsDir` is the path to it
- `speckit.workspaceDir` is where speckit slash commands run from

### 1. Pick ticket
- Query the GitHub Project for the next ticket in the **Todo** column (`gh project item-list`)
- If `assignee` is set in config, filter by that assignee
- If no tickets found, say "No tickets in Todo. All done!" and stop
- If `--dry-run`, print the ticket title and stop
- Move the ticket to the **Refinement** column

### 2. Run speckit in one session
Run all four speckit commands sequentially from `speckit.workspaceDir`.

**specify** → Read the ticket title and body from the GitHub Issue. Run `/speckit.specify` passing the full ticket description as input.

**clarify** → Run `/speckit.clarify`. When clarify has questions:
- Do NOT block waiting for terminal input
- Surface each question to the user here in chat using a clear multiple-choice format
- Wait for the user's answers
- Encode the answers back into the spec
- Continue with plan and tasks

**plan** → Run `/speckit.plan`

**tasks** → Run `/speckit.tasks`

### 3. Create spec branch and PR
In the spec location (`speckit.specsDir`):
- The branch was created by speckit during specify (e.g. `012-feature-name`)
- Push the branch: `git push origin <branch>`
- If the spec is a separate repo (not a folder inside the project): create a PR with `gh pr create --title "<ticket title>" --body "Spec for <ticket title>"`
- Add the spec branch/PR link as a comment on the GitHub Issue

### 4. Move ticket to Ready
Update the ticket status to **Ready** in the GitHub Project.

### 5. Continue or stop
- If `--once` flag was set, stop
- Otherwise ask: *"Refine next ticket? (yes/no)"*
- If yes, go back to Step 1
- If no, stop

## Error handling
- If speckit fails at any step, report the error clearly and stop (do not move the ticket to Ready)
- If the spec branch already exists, pull latest and continue from the last completed step
