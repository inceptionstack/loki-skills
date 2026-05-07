# loki-skills

Skills library for AI agents — works with **OpenClaw**, **Roundhouse**, and any agent that supports the skill format.

Synced from [Kiro Powers](https://github.com/kirodotdev/powers) with custom additions. Each skill provides structured guidance (steering docs, reference material, and constraints) that help AI agents perform specialized tasks effectively.

## Usage

**OpenClaw:** Clone into the workspace — skills are auto-discovered:

```bash
cd ~/.openclaw/workspace
git clone https://github.com/inceptionstack/loki-skills.git skills
```

**Roundhouse / Pi:** Copy skills to the pi agent skills directory:

```bash
git clone https://github.com/inceptionstack/loki-skills.git /tmp/loki-skills
cp -r /tmp/loki-skills/*/ ~/.pi/agent/skills/
```

## Skills

### From Kiro Powers (upstream synced)

| Skill | Description |
|-------|-------------|
| `arm-soc-migration` | Migration between Arm SoC platforms |
| `aws-agentcore` | Build agents with Amazon Bedrock AgentCore |
| `aws-amplify` | Full-stack apps with AWS Amplify Gen 2 |
| `aws-devops-agent` | AWS DevOps automation and CI/CD |
| `aws-graviton-migration` | Code compatibility with Graviton/Arm64 |
| `aws-healthomics` | Genomics workflows in AWS HealthOmics |
| `aws-infrastructure-as-code` | CDK + CloudFormation best practices |
| `aws-mcp` | Multi-step AWS tasks with MCP server |
| `aws-observability` | CloudWatch, X-Ray, Application Signals |
| `aws-sam` | Serverless Application Model |
| `aws-step-functions` | AWS Step Functions workflows |
| `aws-transform` | AWS application modernization/transformation |
| `checkout` | Checkout.com payments API |
| `cloud-architect` | AWS infrastructure (Well-Architected) |
| `cloudwatch-application-signals` | CloudWatch Application Signals |
| `datadog` | Datadog observability |
| `dynatrace` | Dynatrace observability via DQL |
| `gcp-aws-migrate` | GCP to AWS migration |
| `neon` | Serverless Postgres with Neon |
| `postman` | API testing with Postman |
| `power-builder` | Build new Kiro powers |
| `saas-builder` | Multi-tenant SaaS apps |
| `spark-troubleshooting-agent` | Spark on EMR/Glue troubleshooting |
| `stackgen` | StackGen infrastructure generation |
| `strands` | AI agents with Strands SDK |
| `stripe` | Stripe payment integrations |
| `terraform` | Infrastructure as Code with Terraform |

### Custom (loki-only)

| Skill | Description |
|-------|-------------|
| `claude-agent-sdk` | Build AI agents using Claude Agent SDK |
| `cross-agent-test` | Zero-context cross-agent testing of CLI tools |
| `lambda-durable` | AWS Lambda durable functions |

## Sync with upstream

```bash
git clone --depth 1 https://github.com/kirodotdev/powers.git /tmp/powers
# Copy new/updated skills, preserve custom ones
```

Last synced: 2026-05-07

## License

Skills derived from Kiro Powers retain the licensing terms of their original sources. Custom skills are provided as-is.
