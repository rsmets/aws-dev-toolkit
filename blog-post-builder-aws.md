# Building an AI Toolkit Your Org Actually Uses

Claude Code plugins let you package domain knowledge (skills, agents, and MCP servers) into a single installable unit. Install it once and it compounds across the org. Hand it to customers at onboarding and they get working tools and best practices directly in their local dev environment. This post walks through the patterns for structuring a plugin so that knowledge compounds over time, best practices stay consistent across teams, and teams get more productive.

I'll use [**aws-dev-toolkit**](https://github.com/rsmets/aws-dev-toolkit), a Claude Code plugin for interfacing with AWS that I built with 34 skills, 11 agents, and 3 MCP servers, as the running case study. AWS is a good fit for this because its service catalog is large, constantly evolving, and full of overlapping options. Engineers need help navigating it, and that kind of complexity is exactly where a consolidated toolkit pays off. The principles apply to any domain with similar characteristics.

## Pattern 1: Two Types of Skills

A useful distinction when structuring a toolkit: **domain-specific** skills versus **cross-cutting** skills. In `aws-dev-toolkit`, this looks like:

**19 domain-specific skills:** Lambda, DynamoDB, EC2, ECS, EKS, S3, CloudFront, API Gateway, IAM, networking (VPC/subnets/security groups), messaging (SQS/SNS/EventBridge), observability (CloudWatch/X-Ray), Step Functions, Bedrock cost modeling, RDS/Aurora (engine selection, HA topology, Serverless v2, blue/green deployments), AgentCore (Runtime, Memory, Gateway, Identity, Policy, Observability, Evaluations, full platform design for production AI agents), IoT (IoT Core MQTT/shadows/rules engine, Greengrass v2 edge compute, SiteWise, fleet provisioning, device security), MLOps (SageMaker training/inference/pipelines, MLflow on AWS, model monitoring, distributed training, Spot/Inferentia cost optimization), and security review.

**15 cross-cutting skills:** architecture design, side-by-side comparison, debugging, diagram generation, account health checks, migration orchestration, end-to-end planning, GCP-to-AWS and Azure-to-AWS migration, IaC scaffolding (CDK, Terraform, SAM, CloudFormation), Strands Agents SDK scaffolding, Well-Architected reviews, customer ideation, cost analysis, and an adversarial `challenger` that stress-tests recommendations.

The cross-cutting skills are where value compounds fastest. A skill that knows about Lambda cold starts is useful but narrow. A skill that chains discovery → design → security review → cost estimate across services is where the real leverage is. The `challenger` skill is a good example: it stress-tests recommendations by poking at reasoning gaps, unsupported assumptions, and missing failure modes before you commit to a design.

Skills activate based on context. Ask *"Design a serverless API for image processing"* and `aws-plan` runs the full chain. *"Compare ECS vs EKS"* triggers `aws-compare`. *"Why is my CloudFormation stack failing?"* invokes `aws-debug`. The user doesn't need to know which skill handles what.

## Pattern 2: Agents as Composable Workers

Skills encode knowledge. Agents act on it. Skills are reference material and procedures; agents are autonomous workers that can read those skills, call tools, and hand off to each other.

The practical consequence: agents that need to compose must be colocated. A planning agent that hands off to a security reviewer, which hands off to a cost estimator, needs all three in the same plugin. Claude Code's plugin architecture doesn't support cross-plugin imports at the filesystem level, so colocation is what makes composition possible.

In `aws-dev-toolkit`, this is 11 agents via Claude Code's [agent system](https://code.claude.com/docs/en/agents). The value is in **composition**: when `aws-plan` runs, it doesn't try to do everything itself. It spawns `iac-reviewer` for security, `cost-optimizer` for estimates, `well-architected-reviewer` for framework alignment. Each agent has its own system prompt, tool access, and focus. The parent orchestrates. Here's what it has to work with:

| Agent | Purpose |
|-------|---------|
| **aws-explorer** | Read-only reconnaissance of your AWS environment |
| **well-architected-reviewer** | Formal six-pillar WA review with CLI evidence |
| **iac-reviewer** | Security and correctness review of IaC changes before deploy |
| **migration-advisor** | Wave planning, dependency mapping, cutover strategy |
| **bedrock-sme** | Cost-efficient Bedrock model selection |
| **agentcore-sme** | PoC-to-production on Bedrock AgentCore |
| **container-sme** | ECS, EKS, and Fargate architecture decisions |
| **serverless-sme** | Lambda, API Gateway, Step Functions, EventBridge |
| **networking-sme** | VPC design, hybrid connectivity, DNS, CDN |
| **observability-sme** | CloudWatch, X-Ray, OpenTelemetry strategies |
| **cost-optimizer** | Rightsizing, Savings Plans, data transfer waste |

## Shared Infrastructure: MCP Servers

MCP servers provide shared data access that any skill or agent in the plugin can use. `aws-dev-toolkit` connects to three:

| Server | Type | What It Provides |
|--------|------|-----------------|
| **awsiac** | stdio | IaC validation and security scanning via `awslabs.aws-iac-mcp-server` |
| **awsknowledge** | HTTP | AWS documentation, recommendations, and regional availability via `knowledge-mcp.global.api.aws` |
| **awspricing** | stdio | Pricing data and cost reports via `awslabs.aws-pricing-mcp-server` |

## Pattern 3: Colocate What Composes

Agent composition only works if the agents can find each other. In practice, that means colocation. **Cross-cutting concerns need a home.**

Security review isn't a feature of Lambda. Cost estimation isn't a feature of DynamoDB. Well-Architected alignment isn't a feature of any single service. These concerns cut across every workload. If your toolkit is split into per-service plugins, there's no natural place for them, and they tend not to get built.

Looking at the existing [awslabs/agent-plugins](https://github.com/awslabs/agent-plugins) landscape (7 plugins, ~21 skills across ~8 services), the per-service skills are solid, but there are gaps in cross-cutting areas: no agents, no workflow orchestration, no security enforcement, no Well-Architected reviews. When tooling is organized per-service, the connective tissue tends to be missing.

[Anthropic's research on harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps) is relevant here: integrated multi-agent systems outperform narrow single-purpose tools for complex tasks. Cursor's [Composer 2 report](https://cursor.com/resources/Composer2.pdf) found the same: complex multi-service workflows rank low on RL task completion without prescriptive harnesses.

For tightly coupled domains (and AWS services are deeply coupled: Lambda → DynamoDB → API Gateway → CloudFront → IAM → CloudWatch → CDK), consolidation is a prerequisite for composition.

## Pattern 4: Guardrails as Workflow, Not Gatekeeping

In `aws-dev-toolkit`, security review runs as a step in the planning chain rather than a post-execution shell hook. The difference is context: the `iac-reviewer` agent understands what's being designed, so it can distinguish an ALB security group from an RDS security group. A shell hook running after the fact can only do static pattern matching — it doesn't know intent.

The principle: **embed guardrails in the workflow, not around it.** Make them participants in the process, not external validators that can break the process.

The reviewer checks for standard issues (public security groups on databases, unencrypted storage, missing IMDSv2, public RDS) and evaluates context. A security group opening for an ALB is fine; the same opening for an RDS instance is not.

For organizations building their own toolkits, consider also encoding your non-negotiable rules as baseline policies. If you're encoding guardrails for an enterprise toolkit, these are the non-negotiables worth hardcoding. In AWS, that's SCPs:

1. No public security groups on non-web resources
2. No unencrypted storage (S3, RDS, EBS)
3. No root access key creation
4. No public RDS instances
5. No S3 public access via bucket policies
6. Require IMDSv2

These should never be violated without a deliberate exception process. Encoding them in your toolkit means the AI proposes compliant architectures by default.

## Pattern 5: Progressive Discovery Over Interrogation

Early versions of the planning skills in `aws-dev-toolkit` dumped 15 surface-level questions at the user at once. It felt like a tax form. The fix: start with 3-5 high-signal questions, follow the user's answers, and ask permission before going deeper. The question set behind `aws-plan` expanded to 30+ across five categories:

- **Problem Statement:** what are you actually trying to solve?
- **Constraints:** budget, timeline, compliance, team skills
- **Workload Characteristics:** traffic patterns, data volume, latency requirements
- **Integration & Dependencies:** what existing systems does this touch?
- **Operations & Day 2:** who's on call, how do you deploy, what breaks at 3am?

The last two categories catch issues that tend to surface late in design. Getting the question flow right matters. Progressive discovery leads to better context and better recommendations than a flat questionnaire.

### A Note on Token Costs

A reasonable concern: does 34 skills bloat the context window? Not really, because Claude Code only injects skill **names and descriptions** into every conversation, roughly 10KB (~2,500 tokens) for all 34 skills. The full skill content (~318KB across all SKILL.md files) is loaded only when a skill is actually invoked, and reference materials (~310KB more) only when the skill explicitly reads them.

The always-on cost of 34 skills is about the same as a single page of documentation. The deep knowledge (service-specific procedures, reference architectures, decision trees) stays on disk until needed. This is the plugin equivalent of lazy loading: you pay for the index, not the library.

For comparison, a typical CLAUDE.md file runs 2-5KB. The skill index adds roughly one CLAUDE.md worth of context. On a 200K context window that's ~1.2%; on a 1M window it's negligible. The trade-off is worth it: 2,500 tokens of routing information gives the model enough signal to activate any of 34 specialized workflows without the user memorizing slash commands.

If you're building a toolkit and worried about index bloat, keep descriptions under 2-3 sentences each. The trigger phrases ("use when...") matter more than prose. They're what the model pattern-matches against. Verbose descriptions don't improve routing; they just waste tokens.

## Getting Started

The patterns generalize to any tightly coupled domain: cloud infrastructure, data engineering, platform engineering, compliance.

1. **Split skills into domain-specific and cross-cutting.** The cross-cutting ones compound fastest.
2. **Define agents for work that requires composition.** Skills are reference; agents are workers.
3. **Colocate what composes.** If agents need to call each other, they need to live together.
4. **Embed guardrails in the workflow.** Don't bolt safety on from the outside.
5. **Use progressive discovery.** Conversations beat questionnaires.

Start with 3-5 skills that cover your team's most repetitive decisions. Add agents when you find yourself chaining skills manually. Consolidate when the cross-cutting concerns start falling through the cracks.

### Try `aws-dev-toolkit`

**Prerequisites:** [Claude Code](https://code.claude.com) v1.0.33+, [uv](https://docs.astral.sh/uv/getting-started/installation/) (for MCP servers via `uvx`), and AWS CLI configured with credentials.

```bash
# Add the marketplace
/plugin marketplace add rsmets/aws-dev-toolkit

# Install the plugin (plugin@marketplace format)
/plugin install aws-dev-toolkit@rsmets
```

Skills activate automatically based on context:

```
"Design me a serverless API for image processing"     → aws-plan
"Compare ECS vs EKS for my workload"                  → aws-compare
"Run a health check on us-east-1"                     → aws-health-check
"Why is my CloudFormation stack failing?"              → aws-debug
"Scaffold a CDK project for a VPC with NAT"           → iac-scaffold
"We're migrating from GCP to AWS"                     → aws-migrate
```

Or invoke explicitly:

```
/aws-dev-toolkit:iac-scaffold cdk "VPC with public/private subnets and NAT"
/aws-dev-toolkit:aws-health-check us-east-1
/aws-dev-toolkit:aws-diagram from-iac
/aws-dev-toolkit:strands-agent "Create a classification agent with Bedrock"
```

Full documentation in the [README](https://github.com/rsmets/aws-dev-toolkit). If something doesn't work or there's a service skill you want, [open an issue](https://github.com/rsmets/aws-dev-toolkit/issues).

Your domain has the same kind of accumulated knowledge worth encoding. [Build your own plugin](https://code.claude.com/docs/en/plugins) and start compounding it.
