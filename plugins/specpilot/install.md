# cogloop-install

Install the specpilot plugin into the current project.

## Steps

1. **Check prerequisites**
   - Verify `gh` CLI is authenticated: run `gh auth status`
   - Verify Claude Code is running from the project root
   - Confirm a `.claude/commands/` directory exists (create it if not)

2. **Collect configuration** by asking the user the following questions one at a time:
   - GitHub owner (org or username) that owns the GitHub Project board
   - GitHub Project number (visible in the project URL: `/projects/N`)
   - GitHub username to filter tickets by assignee (optional, press Enter to skip)
   - For each repo involved: name, relative path from project root, type (frontend/backend/spec), test command, build command
   - Path to the spec repo (where speckit writes artifacts)
   - Path to the workspace root (where speckit slash commands are available — usually `..` from the project root)
   - GitHub Project column names (show defaults: Todo, Refinement, Ready, In Progress, Done — let user confirm or override each)
   - Model preference for refine phase (default: claude-opus-4-6)
   - Model preference for implement phase (default: claude-sonnet-4-6)

3. **Write `.specpilot.json`** to the project root using the collected values. Follow the schema at `cogloop/plugins/specpilot/config.schema.json`.

4. **Copy skill files** from the cogloop repo into `.claude/commands/`:
   - Fetch `cogloop/plugins/specpilot/skills/refine.md` → `.claude/commands/refine.md`
   - Fetch `cogloop/plugins/specpilot/skills/implement.md` → `.claude/commands/implement.md`

   Fetch from: `https://raw.githubusercontent.com/eduardo-caua/cogloop/main/plugins/specpilot/skills/refine.md`
   and: `https://raw.githubusercontent.com/eduardo-caua/cogloop/main/plugins/specpilot/skills/implement.md`

5. **Update CLAUDE.md** — append a `## specpilot` section explaining:
   - `/refine` — picks next Todo ticket, runs full spec cycle, moves to Ready
   - `/implement` — picks next Ready ticket, implements and ships, moves to Done
   - Config file: `.specpilot.json`

6. **Verify GitHub Project columns** exist by running:
   ```
   gh project field-list <projectNumber> --owner <owner>
   ```
   If any configured column name doesn't match, warn the user and suggest corrections.

7. **Done** — print a summary:
   - Config written to `.specpilot.json`
   - Skills installed: `/refine`, `/implement`
   - Next step: add tickets to the **Todo** column and run `/refine`
