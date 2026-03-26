# sup-virtual-sa

A Claude Code plugin marketplace for AWS development. Ships skills, sub-agents, MCP servers, and hooks that help you build well-architected applications on AWS.

## Quick Start

```bash
# Add the marketplace
/plugin marketplace add https://github.com/rsmets/startup-virtual-sa

# Install the AWS dev toolkit
/plugin install aws-dev-toolkit@sup-virtual-sa
```

Or test locally during development:

```bash
claude --plugin-dir ./plugins/aws-dev-toolkit
```

## Usage

Once installed, the plugin's skills, agents, and MCP servers are available automatically in Claude Code. Here's how each piece works:

### Skills (Automatic)

Skills activate automatically based on context — no special commands needed. Just ask naturally:

```
"Review this architecture for Well-Architected best practices"  → aws-architect
"Why is my CloudFormation stack failing?"                       → aws-debug
"How much is this infrastructure costing me?"                   → cost-check
"Are there security issues in my Terraform?"                    → security-review
"Estimate Bedrock costs for 50k daily invocations"              → bedrock-cost
```

The one exception is `iac-scaffold`, which uses a slash command:

```
/iac-scaffold terraform "VPC with public/private subnets and NAT"
/iac-scaffold cdk "Serverless API with Lambda and DynamoDB"
```

### Sub-Agents (Automatic)

Sub-agents are spun up automatically when Claude determines a specialist is needed. You can also invoke them directly:

```
"Explore my AWS environment and summarize what's deployed"      → aws-explorer
"Review my IaC changes before I deploy"                         → iac-reviewer
"Help me pick the right Bedrock model for classification"       → bedrock-sme
"I have a PoC agent, help me productionize it"                  → agentcore-sme
```

### MCP Servers

The plugin references several AWS MCP servers. In Kiro, MCP configs are not auto-loaded from the plugin directory — you need to add them to your Kiro MCP settings.

Add to `~/.kiro/settings/mcp.json` (user-level) or `.kiro/settings/mcp.json` (workspace-level):

```jsonc
{
  "mcpServers": {
    // AWS IaC validation and security scanning
    "aws-iac": {
      "command": "uvx",
      "args": ["awslabs.aws-iac-mcp-server@latest"],
      "env": { "FASTMCP_LOG_LEVEL": "ERROR" },
      "disabled": false
    },
    // AWS documentation search
    "aws-docs": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      "env": { "FASTMCP_LOG_LEVEL": "ERROR" },
      "disabled": false
    },
    // Core AWS resource access
    "aws-core": {
      "command": "uvx",
      "args": ["awslabs.core-mcp-server@latest"],
      "env": { "FASTMCP_LOG_LEVEL": "ERROR", "AWS_PROFILE": "default" },
      "disabled": false
    },
    // Cost analysis
    "aws-cost": {
      "command": "uvx",
      "args": ["awslabs.cost-analysis-mcp-server@latest"],
      "env": { "FASTMCP_LOG_LEVEL": "ERROR", "AWS_PROFILE": "default" },
      "disabled": false
    }
  }
}
```

These are used behind the scenes by skills and agents — you don't need to invoke them directly.

| Server | Package | Description |
|---|---|---|
| `aws-iac` | `awslabs.aws-iac-mcp-server` | CDK/Terraform/CloudFormation development with security scanning |
| `aws-docs` | `awslabs.aws-documentation-mcp-server` | Latest AWS documentation and code samples |
| `aws-core` | `awslabs.core-mcp-server` | Proxy server that dynamically imports other AWS MCP servers |
| `aws-cost` | `awslabs.cost-analysis-mcp-server` | Cost analysis and optimization |

### Hooks

Hooks run automatically on events. Currently configured:

- After editing an IaC file (`.tf`, `template.yaml`, `*-stack.ts`, etc.), Claude reminds you to validate before deploying

### Example Workflows

**"I need a new service on AWS"**
1. Describe what you're building — `aws-architect` kicks in with a design
2. Ask to scaffold it — `/iac-scaffold cdk "your description"`
3. Edit the generated code — the hook reminds you to `cdk synth && cdk diff`
4. Ask for a security review before deploying

**"My Bedrock agent is too expensive"**
1. Ask about your Bedrock usage — `bedrock-sme` analyzes your patterns
2. Get model selection guidance — it'll steer you toward the cheapest model that works
3. Ask `cost-check` to look at your overall AWS bill for context

**"I built a PoC agent, now what?"**
1. Share your agent code — `agentcore-sme` reviews it against the production checklist
2. Get guidance on adding DeepEval for model evaluation
3. Choose between AgentCore native observability or Langfuse
4. Walk through the PoC → production migration path

**"We're moving from GCP to AWS"**
1. Describe your GCP environment — `gcp-to-aws` maps services to AWS equivalents
2. Run the assessment commands to inventory what's deployed
3. Review the gotchas for your specific services (global VPCs, Spanner, BigQuery)
4. Use `iac-scaffold` to generate the target AWS infrastructure
5. Ask `migration-advisor` for wave planning and cutover strategy

**"We're moving from Azure to AWS"**
1. Describe your Azure environment — `azure-to-aws` maps services to AWS equivalents
2. Run az CLI discovery commands to inventory resources
3. Pay special attention to identity migration (Azure AD → IAM Identity Center)
4. Review Cosmos DB and Synapse migration paths (these are complex)
5. Use `iac-scaffold` to generate the target AWS infrastructure

**"I have an idea for something on AWS"**
1. Describe your idea — `customer-ideation` guides you through discovery
2. Answer the structured questions about requirements and constraints
3. Review the proposed architecture with Well-Architected checklist
4. Use `/iac-scaffold` to generate starter infrastructure code
5. Ask for a cost estimate before committing

## What's Included

### Plugins

#### `aws-dev-toolkit`

**Skills:**
| Skill | Trigger | Description |
|---|---|---|
| `aws-architect` | Auto | Design & review AWS architectures against Well-Architected Framework |
| `iac-scaffold` | `/iac-scaffold <framework> <description>` | Scaffold CDK, Terraform, SAM, or CloudFormation projects |
| `aws-debug` | Auto | Debug AWS deployment failures, Lambda errors, permission issues |
| `cost-check` | Auto | Analyze and optimize AWS costs |
| `security-review` | Auto | Audit IaC and AWS configs for security issues |
| `strands-agent` | `/strands-agent <description>` | Scaffold Strands Agents SDK projects on Bedrock AgentCore (TS/Python) |
| `bedrock-cost` | Auto | Bedrock pricing, token economics, and cost modeling |
| `gcp-to-aws` | Auto | GCP to AWS migration service mapping, gotchas, and environment assessment |
| `azure-to-aws` | Auto | Azure to AWS migration service mapping, gotchas, and environment assessment |
| `customer-ideation` | Auto | Guided ideation from concept to AWS architecture with Well-Architected review |

**Sub-Agents:**
| Agent | Model | Description |
|---|---|---|
| `aws-explorer` | Haiku | Read-only AWS environment exploration and context gathering |
| `iac-reviewer` | Sonnet | Reviews IaC changes for correctness, security, and best practices |
| `bedrock-sme` | Sonnet | Bedrock subject matter expert emphasizing cost-efficient usage patterns |
| `agentcore-sme` | Sonnet | AgentCore expert for PoC-to-production agent development with DeepEval and Langfuse guidance |

**MCP Servers:**
| Server | Package | Description |
|---|---|---|
| `aws-iac` | `awslabs.aws-iac-mcp-server` | CDK/Terraform/CloudFormation development with security scanning |
| `aws-docs` | `awslabs.aws-documentation-mcp-server` | Latest AWS documentation and code samples |
| `aws-core` | `awslabs.core-mcp-server` | Proxy server that dynamically imports other AWS MCP servers |
| `aws-cost` | `awslabs.cost-analysis-mcp-server` | Cost analysis and optimization |

**Hooks:**
- Post-edit reminder to validate IaC files before deploying

## Prerequisites

- [Claude Code](https://code.claude.com) v1.0.33+
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (for MCP servers via `uvx`)
- AWS CLI configured with appropriate credentials
- (Optional) `checkov`, `cfn-nag`, `tfsec` for security scanning

## Project Structure

```
sup-virtual-sa/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace catalog
├── plugins/
│   └── aws-dev-toolkit/          # First plugin
│       ├── .claude-plugin/
│       │   └── plugin.json       # Plugin manifest
│       ├── .mcp.json             # MCP server configs
│       ├── skills/               # Agent skills
│       │   ├── aws-architect/
│       │   ├── iac-scaffold/
│       │   ├── aws-debug/
│       │   ├── cost-check/
│       │   ├── security-review/
│       │   ├── bedrock-cost/
│       │   ├── gcp-to-aws/
│       │   ├── azure-to-aws/
│       │   └── customer-ideation/
│       ├── agents/               # Sub-agents
│       │   ├── aws-explorer.md
│       │   ├── iac-reviewer.md
│       │   ├── bedrock-sme.md
│       │   └── agentcore-sme.md
│       └── hooks/
│           └── hooks.json
└── README.md
```

## Adding More Plugins

This marketplace is designed to host multiple plugins. To add a new one:

1. Create a directory under `plugins/<your-plugin-name>/`
2. Add `.claude-plugin/plugin.json` with the manifest
3. Add your skills, agents, hooks, and MCP configs
4. Register it in `.claude-plugin/marketplace.json`

## Available AWS MCP Servers

The [awslabs/mcp](https://awslabs.github.io/mcp/servers) project provides 60+ official MCP servers. Some notable ones to consider adding:

| Server | Use Case |
|---|---|
| `awslabs.aws-api-mcp-server` | Direct AWS API access via CLI |
| `awslabs.cdk-mcp-server` | CDK-specific development |
| `awslabs.terraform-mcp-server` | Terraform-specific workflows |
| `awslabs.lambda-mcp-server` | Lambda function management |
| `awslabs.s3-mcp-server` | S3 operations |
| `awslabs.cloudformation-mcp-server` | CloudFormation resource management |
| `awslabs.bedrock-mcp-server` | Bedrock AI model integration |
| `awslabs.cloudwatch-mcp-server` | Metrics, alarms, and log analysis |
| `awslabs.iam-mcp-server` | IAM user, role, and policy management |
| `awslabs.well-architected-mcp-server` | Well-Architected Framework reviews |

## License

MIT
