# Building an AI Toolkit Your Org Actually Uses

Claude Code plugins let you package domain knowledge (skills, agents, and MCP servers) into a single installable unit. Shared across an engineering org, they compound developer experience. Provided to customers at onboarding, they put working tools and best practices directly in the local dev environment. This post walks through the patterns for structuring a plugin so that knowledge compounds over time rather than scattering across disconnected tools.

I'll use [**aws-dev-toolkit**](https://github.com/rsmets/aws-dev-toolkit), a Claude Code plugin I built with 32 skills, 11 agents, and 3 MCP servers, as the running case study. AWS is a good fit for this because its service catalog is large, constantly evolving, and full of overlapping options. Engineers need help navigating it, and that kind of complexity is exactly where a consolidated toolkit pays off. The principles apply to any domain with similar characteristics.

## Pattern 1: Two Types of Skills

A useful distinction when structuring a toolkit: **domain-specific** skills versus **cross-cutting** skills. In `aws-dev-toolkit`, this looks like:

**17 domain-specific skills:** Lambda, DynamoDB, EC2, ECS, EKS, S3, CloudFront, API Gateway, IAM, networking (VPC/subnets/security groups), messaging (SQS/SNS/EventBridge), observability (CloudWatch/X-Ray), Step Functions, Bedrock cost modeling, RDS/Aurora (engine selection, HA topology, Serverless v2, blue/green deployments), AgentCore (Runtime, Memory, Gateway, Identity, Policy, Observability, Evaluations — full platform design for production AI agents), and security review.

**15 cross-cutting skills:** architecture design, side-by-side comparison, debugging, diagram generation, account health checks, migration orchestration, end-to-end planning, GCP-to-AWS and Azure-to-AWS migration, IaC scaffolding (CDK, Terraform, SAM, CloudFormation), Strands Agents SDK scaffolding, Well-Architected reviews, customer ideation, cost analysis, and an adversarial `challenger` that stress-tests recommendations.

The cross-cutting skills are where value compounds fastest. A skill that knows about Lambda cold starts is useful but narrow. A skill that chains discovery → design → security review → cost estimate across services is where the real leverage is. The `challenger` skill is a good example: it stress-tests recommendations by poking at reasoning gaps, unsupported assumptions, and missing failure modes before you commit to a design.

Skills activate based on context. Ask *"Design a serverless API for image processing"* and `aws-plan` runs the full chain. *"Compare ECS vs EKS"* triggers `aws-compare`. *"Why is my CloudFormation stack failing?"* invokes `aws-debug`. The user doesn't need to know which skill handles what.

## Pattern 2: Agents as Composable Workers

Skills encode knowledge. Agents act on it. Skills are reference material and procedures; agents are autonomous workers that can read those skills, call tools, and hand off to each other.

The practical consequence: agents that need to compose must be colocated. A planning agent that hands off to a security reviewer, which hands off to a cost estimator, needs all three in the same plugin. Claude Code's plugin architecture doesn't support cross-plugin imports at the filesystem level, so colocation is what makes composition possible.

In `aws-dev-toolkit`, this is 11 agents via Claude Code's [agent system](https://code.claude.com/docs/en/agents):

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

The value is in **composition**. When `aws-plan` runs, it doesn't try to do everything itself. It spawns `iac-reviewer` for security, `cost-optimizer` for estimates, `well-architected-reviewer` for framework alignment. Each agent has its own system prompt, tool access, and focus. The parent orchestrates.

## Shared Infrastructure: MCP Servers

MCP servers provide shared data access that any skill or agent in the plugin can use. `aws-dev-toolkit` connects to three:

| Server | Type | What It Provides |
|--------|------|-----------------|
| **awsiac** | stdio | IaC validation and security scanning via `awslabs.aws-iac-mcp-server` |
| **awsknowledge** | HTTP | AWS documentation, recommendations, and regional availability via `knowledge-mcp.global.api.aws` |
| **awspricing** | stdio | Pricing data and cost reports via `awslabs.aws-pricing-mcp-server` |

## Pattern 3: Colocate What Composes

The core argument for consolidation: **cross-cutting concerns need a home.**

Security review isn't a feature of Lambda. Cost estimation isn't a feature of DynamoDB. Well-Architected alignment isn't a feature of any single service. These concerns cut across every workload. If your toolkit is split into per-service plugins, there's no natural place for them, and they tend not to get built.

Looking at the existing [awslabs/agent-plugins](https://github.com/awslabs/agent-plugins) landscape (7 plugins, ~21 skills across ~8 services), the per-service skills are solid, but there are gaps in cross-cutting areas: no agents, no workflow orchestration, no security enforcement, no Well-Architected reviews. That's not a knock on those plugins; it illustrates the pattern. When tooling is organized per-service, the connective tissue tends to be missing.

[Anthropic's research on harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps) is relevant here: integrated multi-agent systems outperform narrow single-purpose tools for complex tasks. Cursor's [Composer 2 report](https://cursor.com/resources/Composer2.pdf) found the same: complex multi-service workflows rank low on RL task completion without prescriptive harnesses.

For tightly coupled domains (and AWS services are deeply coupled: Lambda → DynamoDB → API Gateway → CloudFront → IAM → CloudWatch → CDK), consolidation is a prerequisite for composition.

## Pattern 4: Guardrails as Workflow, Not Gatekeeping

In `aws-dev-toolkit`, the first implementation of mandatory security review used a post-execution shell hook on every IaC file edit. When the script failed, the agent flow broke entirely. The fix was moving enforcement into the AI workflow itself. The `iac-reviewer` agent now runs as a mandatory step in the planning chain. If something goes wrong, it surfaces the issue rather than crashing the session.

The principle: **embed guardrails in the workflow, not around it.** Make them participants in the process, not external validators that can break the process.

The reviewer checks for standard issues (public security groups on databases, unencrypted storage, missing IMDSv2, public RDS) and evaluates context. A security group opening for an ALB is fine; the same opening for an RDS instance is not.

For organizations building their own toolkits, consider also encoding your non-negotiable rules as baseline policies. In AWS, that's SCPs:

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

## Try It or Build Your Own

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
/aws-dev-toolkit:strands-agent "Classification agent with Bedrock"
```

Full documentation in the [README](https://github.com/rsmets/aws-dev-toolkit). If something doesn't work or there's a service skill you want, [open an issue](https://github.com/rsmets/aws-dev-toolkit/issues).

Thanks for making it this far!
