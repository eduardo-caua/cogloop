# cogloop

> A plugin marketplace for Claude Code + GitHub native development workflows.

cogloop is a collection of installable plugins for [Claude Code](https://claude.ai/code) that encode proven development methodologies. Each plugin adds slash commands to your project that automate specific workflows — from spec-driven development to release management.

## How it works

1. Browse the plugins below
2. Copy the plugin's `install.md` into your project's `.claude/commands/` folder
3. Run the install skill in Claude Code — it sets up everything else automatically

```bash
# Example: installing specpilot
cp plugins/specpilot/install.md your-project/.claude/commands/cogloop-install.md
```

Then in Claude Code:
```
/cogloop-install
```

## Plugins

| Plugin | Description | Stack |
|--------|-------------|-------|
| [specpilot](./plugins/specpilot/) | Spec-driven development loop with Claude + GitHub Projects | Claude Code, GitHub Projects, speckit |

## Philosophy

cogloop plugins are **methodology-first** — they encode how to build software well, not just automate tasks. Each plugin is:

- **Stack-specific** — built for a defined set of tools, not generic
- **Interactive** — uses Claude Code's chat interface for decisions that need a human
- **Self-contained** — one install command, no external dependencies beyond the defined stack
- **Open** — all skills are readable markdown files, no black boxes

## Contributing

Want to add a plugin? Open a PR with your plugin under `plugins/<plugin-name>/` following the structure in any existing plugin.

---

Made with [Claude Code](https://claude.ai/code)
