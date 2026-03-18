# cogloop

> A plugin marketplace for Claude Code + GitHub native development workflows.

cogloop is a collection of installable plugins for [Claude Code](https://claude.ai/code) that encode proven development methodologies. Each plugin adds slash commands to your project that automate specific workflows — from spec-driven development to release management.

## How it works

1. Add cogloop as a marketplace in Claude Code
2. Install the plugin you want
3. Run the plugin's setup skill — it configures everything automatically

```bash
# Add the cogloop marketplace
claude plugin marketplace add https://github.com/eduardo-caua/cogloop

# Install a plugin
claude plugin install specpilot
```

Then in Claude Code:
```
/specpilot-setup
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
