# CLAUDE.md

## Project Overview

This is the **aws-dev-toolkit** Claude Code plugin — a comprehensive AWS development toolkit with 30 skills, 11 specialized agents, and 3 MCP servers for building, migrating, and reviewing well-architected applications on AWS.

## Repository Structure

```
sup-virtual-sa/
├── plugins/aws-dev-toolkit/       # The plugin
│   ├── .claude-plugin/plugin.json # Plugin manifest
│   ├── .mcp.json                  # 3 MCP server configs
│   ├── skills/                    # 30 skills (each with SKILL.md)
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

### Skills = Slash Commands (No Separate `commands/` Directory)

As of Claude Code's current architecture, **custom commands have been merged into skills**. A `commands/deploy.md` and a `skills/deploy/SKILL.md` both create `/deploy` and work identically. The `commands/` directory is legacy — **this plugin uses `skills/` exclusively**, which is the recommended modern pattern.

Why skills over commands:
- **Supporting files**: skills are directories, so you can include templates, reference docs, and scripts alongside SKILL.md
- **Auto-invocation**: Claude can invoke skills based on their `description` without the user typing a slash command
- **Richer frontmatter**: `disable-model-invocation`, `context: fork`, `allowed-tools`, `paths`, `model`, `effort`
- **Every skill is also a slash command**: `/aws-dev-toolkit:skill-name` works automatically

If you see `commands/` in other plugins or older examples, know that it still works but offers fewer capabilities.

> **Source**: [Claude Code Skills Reference](https://code.claude.com/docs/en/skills) — *"Custom commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Your existing `.claude/commands/` files keep working. Skills add optional features."*

### Conventions

- Skills go in `plugins/aws-dev-toolkit/skills/<skill-name>/SKILL.md`
- Agents go in `plugins/aws-dev-toolkit/agents/<agent-name>.md`
- All names use **kebab-case**
- Every SKILL.md must have YAML frontmatter with `name` and `description`
- Every agent .md must have YAML frontmatter with `name`, `description`, and `tools`

### Version Bumping — Required on Every Change

**Every change to the plugin must include a version bump** in **both** files:
- `plugins/aws-dev-toolkit/.claude-plugin/plugin.json` — the plugin version
- `.claude-plugin/marketplace.json` — the marketplace version (both the top-level `metadata.version` and the plugin entry `version`)

All three version fields must stay in sync. Use [semver](https://semver.org/):

| Change type | Bump | Example |
|-------------|------|---------|
| Breaking changes (removed skills, renamed agents, changed hook behavior, incompatible config) | **Major** | `0.4.1` → `1.0.0` |
| New skills, agents, MCP servers, or meaningful capability additions | **Minor** | `0.4.1` → `0.5.0` |
| Bug fixes, doc updates, wording tweaks, config corrections | **Patch** | `0.4.1` → `0.4.2` |

**Rules:**
- Bump the version in the same commit as the change — never defer it
- When multiple changes land in one commit, use the highest applicable bump level
- Include the version bump in the commit message (e.g., `feat: add lambda skill (v0.5.0)`)
- Pre-1.0 (`0.x.y`): minor bumps may include breaking changes per semver convention

## Security Review — Mandatory for IaC Changes

**Any time IaC changes are proposed or written** (CloudFormation, CDK, Terraform, SAM, Pulumi, or raw AWS CLI/SDK scripts that create/modify infrastructure), the `security-review` skill or `iac-reviewer` subagent **MUST** be invoked before the work is considered complete. This is not optional.

- For template/config changes: spawn the `iac-reviewer` agent (`subagent_type: "aws-dev-toolkit:iac-reviewer"`)
- For broader security audits or IAM/networking reviews: invoke the `security-review` skill
- When in doubt, run both

### Hook Policy — Fail Open

Do **NOT** use post-execution shell-script hooks for security review gating. A previous attempt broke the agent flow when the hook script failed. If a hook is ever added in the future, it **must fail open** — log a warning and continue execution. Never block the agent on a hook failure.

### SCP Recommendation — Baseline Guardrails

When advising on AWS account security, **strongly recommend** Service Control Policies (SCPs) to enforce obvious baseline rules that should never be violated. Suggested SCP guardrails to propose:

| Rule | What to Deny | Why |
|------|-------------|-----|
| No public security groups on private resources | `ec2:AuthorizeSecurityGroupIngress` where CIDR is `0.0.0.0/0` and port is not 80/443, applied to EC2, RDS, ElastiCache, Redshift | Prevents accidental internet exposure of databases and compute |
| No unencrypted storage | `s3:CreateBucket` without default encryption, `rds:CreateDBInstance` without `StorageEncrypted`, `ec2:CreateVolume` without encryption | Data at rest encryption is non-negotiable |
| No root access key creation | `iam:CreateAccessKey` for root | Root should use console with MFA only |
| No public RDS instances | `rds:CreateDBInstance` / `rds:ModifyDBInstance` with `PubliclyAccessible=true` | Databases should never face the internet |
| No S3 public access | Deny `s3:PutBucketPolicy` / `s3:PutBucketAcl` that grant public access | Use CloudFront or pre-signed URLs instead |
| Require IMDSv2 | Deny `ec2:RunInstances` without `HttpTokens=required` | Prevents SSRF-based credential theft |

These are "braindead obvious" rules — if someone needs an exception, they should go through a formal exception process, not weaken the SCP.

## Upstream Contributions

Two forked repos live in this directory (gitignored):
- `claude-plugins-official/` — fork of `anthropics/claude-plugins-official` (marketplace entry, auto-closes external PRs — use official submission form instead)
- `agent-plugins/` — fork of `awslabs/agent-plugins` (PR #107 open, requires RFC issue per contributing guidelines)
