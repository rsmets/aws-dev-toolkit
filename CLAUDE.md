# CLAUDE.md

## Project Overview

This is the **aws-dev-toolkit** Claude Code plugin — a comprehensive AWS development toolkit with 25 skills, 11 specialized agents, and 3 MCP servers for building, migrating, and reviewing well-architected applications on AWS.

## Repository Structure

```
sup-virtual-sa/
├── plugins/aws-dev-toolkit/       # The plugin
│   ├── .claude-plugin/plugin.json # Plugin manifest
│   ├── .mcp.json                  # 5 MCP server configs
│   ├── skills/                    # 25 skills (each with SKILL.md)
│   ├── agents/                    # 11 sub-agents
│   └── hooks/hooks.json           # Hook definitions
├── claude-plugins-official/       # Fork of anthropics/claude-plugins-official (gitignored)
├── agent-plugins/                 # Fork of awslabs/agent-plugins (gitignored)
├── LICENSE                        # MIT
└── README.md
```

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/) format for all commits:

- `feat:` new feature
- `fix:` bug fix
- `chore:` maintenance/tooling
- `docs:` documentation
- `refactor:` code restructuring
- `test:` adding/updating tests

## Plugin Development

- Skills go in `plugins/aws-dev-toolkit/skills/<skill-name>/SKILL.md`
- Agents go in `plugins/aws-dev-toolkit/agents/<agent-name>.md`
- All names use **kebab-case**
- Every SKILL.md must have YAML frontmatter with `name` and `description`
- Every agent .md must have YAML frontmatter with `name`, `description`, and `tools`

## Upstream Contributions

Two forked repos live in this directory (gitignored):
- `claude-plugins-official/` — fork of `anthropics/claude-plugins-official` (marketplace entry, auto-closes external PRs — use official submission form instead)
- `agent-plugins/` — fork of `awslabs/agent-plugins` (PR #107 open, requires RFC issue per contributing guidelines)
