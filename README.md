# skill-design-philosophy

Design philosophy and principles for creating outstanding [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills.

Everything lives in [SKILL.md](skills/skill-design-philosophy/SKILL.md): core principles, script template, project structure, and checklists.

## Installation

### As a plugin

Installs through the [fgilio marketplace](https://github.com/fgilio/claude-plugins) and receives updates as the skill evolves:

```text
/plugin marketplace add fgilio/claude-plugins
/plugin install skill-design-philosophy@fgilio
```

### Manual clone

Clone and symlink into your Claude Code skills directory:

```bash
git clone https://github.com/fgilio/skill-design-philosophy-skill.git ~/dev/skills/skill-design-philosophy-skill
ln -s ~/dev/skills/skill-design-philosophy-skill/skills/skill-design-philosophy ~/.claude/skills/skill-design-philosophy
```

This path also works with any agent that supports the open [Agent Skills](https://agentskills.io) format. Updates require a manual `git pull`.

Claude Code will automatically discover the skill and use it when creating or improving skills.

## Development

```bash
bun install
bun run check
```

`bun install` installs the pinned tools from `bun.lock` and the pre-commit hook, which formats and lints the staged Markdown, JSON and YAML. `bun run check` runs the format and Markdown gates. CI also runs the prose and plugin checks. [fleet/README.md](https://github.com/fgilio/claude-plugins/blob/main/fleet/README.md) in the marketplace repository describes the standard and the shared workflow that checks it.

## License

[MIT](LICENSE)
