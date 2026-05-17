# loki-skills

Skills library for AI agents — works with **OpenClaw**, **Roundhouse**, and any agent that supports the skill format.

Synced from [Kiro Powers](https://github.com/kirodotdev/powers) with custom additions. Each skill provides structured guidance (steering docs, reference material, and constraints) that help AI agents perform specialized tasks effectively.

## Usage

**OpenClaw:** Clone into the workspace — skills are auto-discovered:

```bash
cd ~/.openclaw/workspace
git clone https://github.com/inceptionstack/loki-skills.git skills
```

**Roundhouse / Pi:** Clone and copy to pi agent skills:

```bash
git clone https://github.com/inceptionstack/loki-skills.git /tmp/loki-skills
for d in /tmp/loki-skills/*/; do
  [ -d "$d" ] && cp -r "$d" ~/.pi/agent/skills/
done
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

<!-- custom-skills-start -->
| Skill | Description |
|-------|-------------|
| `claude-agent-sdk` | Build AI agents using Claude Agent SDK |
| `cross-agent-test` | Zero-context cross-agent testing of CLI tools |
| `lambda-durable` | AWS Lambda durable functions |
| `mcporter` | Use MCP servers via CLI |
| `module-organizer` | Prevent namespace drift in TypeScript projects |
| `playwright-cli` | Browser automation via playwright-cli |
| `refactoring` | Refactoring patterns & safe code transformation |
| `unit-testing` | Unit testing with ATAT (3rd Edition) |

**Note:** Custom skills list must match the `CUSTOM_SKILLS` array in `.github/workflows/sync-upstream.yml`. Consistency is verified by CI (`.github/workflows/lint.yml`).

<!-- custom-skills-end -->

## Sync with upstream

**Automated:** Daily sync via GitHub Actions (`.github/workflows/sync-upstream.yml`).
- Runs at 08:00 UTC daily
- Creates PR if changes detected (manual review before merge)
- Clones from `main` branch (upstream has no release tags)
- Preserves custom skills (see above)
- **Prerequisite:** Enable "Allow GitHub Actions to create and approve pull requests" in repo settings (Settings → Actions → General)

**Manual sync:**
```bash
git clone --depth 1 https://github.com/kirodotdev/powers.git /tmp/powers
# Copy new/updated skills, preserve custom ones (see CUSTOM_SKILLS array in workflow)
```

Last synced: 2026-05-17

## License

Skills derived from Kiro Powers retain the licensing terms of their original sources. Custom skills are provided as-is.
