# loki-skills

OpenClaw workspace skills used by [Loki](https://github.com/openclaw/openclaw) — an AI DevOps agent managing AWS infrastructure.

These skills were adapted for use with OpenClaw from [Kiro Powers](https://kiro.dev/powers/) and other sources. Some are custom-built. Each skill provides structured guidance (steering docs, reference material, and constraints) that help AI agents perform specialized tasks effectively.

## Skills

| Skill | Description | Based on Kiro Power |
|-------|-------------|---------------------|
| `arm-soc-migration` | Guides migration of code from one Arm SoC to another | [Perform Migration between Arm SoC](https://kiro.dev/powers/arm-soc-migration) (Arm) |
| `aws-agentcore` | Build, test, and deploy AI agents using AWS Bedrock AgentCore | [Build an agent with Amazon Bedrock AgentCore](https://kiro.dev/powers/aws-agentcore) (AWS) |
| `aws-amplify` | Build full-stack apps with AWS Amplify Gen 2 | [Build full-stack apps with AWS Amplify](https://kiro.dev/powers/aws-amplify) (AWS) |
| `aws-graviton-migration` | Analyze code compatibility with Graviton/Arm64 processors | [Plan and Migrate to Graviton](https://kiro.dev/powers/aws-graviton-migration) (AWS) |
| `aws-healthomics` | Create, migrate, run, and debug genomics workflows in AWS HealthOmics | [AWS HealthOmics](https://kiro.dev/powers/aws-healthomics) (AWS) |
| `aws-infrastructure-as-code` | Build AWS infrastructure with CDK + CloudFormation best practices | [Build AWS infrastructure with CDK and CloudFormation](https://kiro.dev/powers/aws-infrastructure-as-code) (AWS) |
| `aws-mcp` | Perform multi-step AWS tasks with docs, API calls, and Agent SOPs | — (AWS MCP server, not a Kiro Power) |
| `aws-observability` | CloudWatch Logs, Metrics, Alarms, Application Signals, CloudTrail | [AWS Observability](https://kiro.dev/powers/aws-observability) (AWS) |
| `checkout` | Checkout.com API documentation and payment operations | [Checkout.com Global Payments](https://kiro.dev/powers/checkout) (Checkout.com) |
| `claude-agent-sdk` | Build AI agents using Claude Agent SDK (Python & TypeScript) | — (Custom) |
| `cloud-architect` | Build AWS infrastructure with CDK in Python (Well-Architected) | [Build infrastructure on AWS](https://kiro.dev/powers/cloud-architect) (Christian Bonzelet) |
| `cloudwatch-application-signals` | ⚠️ DEPRECATED — merged into `aws-observability` | [Amazon CloudWatch Application Signals](https://kiro.dev/powers/cloudwatch-application-signals) (AWS) |
| `cross-agent-test` | Zero-context cross-agent testing of CLI tools | — (Custom) |
| `datadog` | Query logs, metrics, traces, incidents from Datadog | [Datadog Observability](https://kiro.dev/powers/datadog) (Datadog) |
| `dynatrace` | Query logs, metrics, traces, problems from Dynatrace via DQL | [Dynatrace Observability](https://kiro.dev/powers/dynatrace) (Dynatrace) |
| `figma` | Connect Figma designs to code components | [Design to Code with Figma](https://kiro.dev/powers/figma) (Figma) |
| `lambda-durable` | Build resilient multi-step apps with AWS Lambda durable functions | [Build applications with Lambda durable functions](https://kiro.dev/powers/lambda-durable) (AWS) |
| `neon` | Serverless Postgres with branching, autoscaling, scale-to-zero | [Build a database with Neon](https://kiro.dev/powers/neon) (Neon) |
| `outline` | Interact with Outline wiki API — CRUD docs and collections | — (Custom) |
| `postman` | Automate API testing and collection management | [API Testing with Postman](https://kiro.dev/powers/postman) (Postman) |
| `reposwarm` | Manage RepoSwarm — investigations, workflows, architecture analysis | — (Custom) |
| `saas-builder` | Build multi-tenant SaaS apps with serverless + billing + security | [SaaS Builder](https://kiro.dev/powers/saas-builder) (Allen Helton) |
| `spark-troubleshooting-agent` | Troubleshoot Spark on EMR, Glue, and SageMaker | — (AWS, not listed on Kiro) |
| `strands` | Build AI agents with Strands SDK (multi-provider) | [Build an agent with Strands](https://kiro.dev/powers/strands) (AWS) |
| `stripe` | Build payment integrations — subscriptions, billing, refunds | [Stripe Payments](https://kiro.dev/powers/stripe) (Stripe) |
| `terraform` | Infrastructure as Code with Terraform — registry, modules, policies | [Deploy infrastructure with Terraform](https://kiro.dev/powers/terraform) (HashiCorp) |

## Credits

Most skills in this repo are adapted from [Kiro Powers](https://kiro.dev/powers/) — reusable capability packages built by AWS, partner companies, and the community for the [Kiro IDE](https://kiro.dev). Skills marked "Custom" were built independently for OpenClaw.

The adaptation involved converting Kiro's power format (MCP servers, steering docs, hooks) into OpenClaw's skill format (SKILL.md + refs/). The underlying guidance, best practices, and reference material remain largely faithful to the original Kiro Powers.

## License

Skills derived from Kiro Powers retain the licensing terms of their original sources. Custom skills are provided as-is.
