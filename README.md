# Ouro Skills

Reusable AI agent skills for building on the Ouro platform. Each skill teaches an AI coding agent how to perform a specific workflow — deploying models, integrating with Ouro APIs, creating assets, and more.

## Skills

| Skill | Description |
|-------|-------------|
| [magnets](magnets/) | How to search for permanent magnets with Ouro's models: the targets, the cheap-to-expensive funnel of routes, their calibration against measured magnets, the traps that waste DFT money, and the structure families already mapped. |
| [matsci-modal-app](matsci-modal-app/) | Build Modal apps for open-source materials science AI models with Ouro integration. Covers model discovery, Modal deployment, file I/O, webhook callbacks, and action logging. |

## How It Works

Skills are markdown files that give AI agents domain-specific knowledge they wouldn't otherwise have. Each skill lives in its own directory with a `SKILL.md` file and optional reference material.

```
skill-name/
├── SKILL.md        # Main instructions (the agent reads this first)
├── reference.md    # Detailed templates, patterns, examples
└── scripts/        # Optional utility scripts
```

When you attach a skill (or it's auto-matched by description), the agent reads `SKILL.md` and follows its workflow. Reference files are read on demand to keep context lean.

## Install

### skills.sh

Install any skill using the [skills CLI](https://skills.sh/docs):

```bash
npx skills add ourofoundation/skills
```

Or one skill at a time:

```bash
npx skills add ourofoundation/skills --skill magnets
```

### Cursor

Clone into your project or global skills directory:

```bash
# Project-level
git clone https://github.com/ourofoundation/skills.git .cursor/skills/ouro

# Global (available across all projects)
git clone https://github.com/ourofoundation/skills.git ~/.cursor/skills/ouro
```

### Claude Code

Add skills as project-level context by cloning into `.claude/skills/`:

```bash
git clone https://github.com/ourofoundation/skills.git .claude/skills/ouro
```

Then reference a skill in conversation by pointing to its `SKILL.md` file, or add it to your `CLAUDE.md`:

```markdown
## Skills
When working on Modal apps for materials science, follow the instructions in `.claude/skills/ouro/matsci-modal-app/SKILL.md`.
```

### Manual

Any agent can use a skill — just include the `SKILL.md` contents in your prompt or attach the file to a conversation.

## Creating a New Skill

1. Create a directory: `mkdir my-skill`
2. Add a `SKILL.md` with YAML frontmatter (`name` and `description` are required)
3. Keep `SKILL.md` under 500 lines — put detailed content in separate reference files
4. Write the description in third person, including both *what* the skill does and *when* to use it

```yaml
---
name: my-skill
description: Does X and Y. Use when the user wants to Z or asks about W.
---
```

## License

MIT
