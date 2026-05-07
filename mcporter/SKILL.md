---
name: mcporter
description: "Use MCP (Model Context Protocol) servers via the mcporter CLI. Call AWS APIs, browse docs, manage infrastructure, and more through configured MCP servers."
---

# MCPorter — MCP Server CLI

MCPorter lets you discover and call tools from configured MCP servers directly via bash.

> **Note:** If `mcporter` or `uvx` isn't found, ensure PATH includes `~/.local/bin`:
> `export PATH="$HOME/.local/bin:$PATH"`

## Quick Reference

### List available servers

```bash
mcporter list
```

### List tools on a specific server (with parameter docs)

```bash
mcporter list <server-name> --schema
```

### Call a tool

```bash
mcporter call <server>.<tool> key=value key2=value2
```

Or function-call style:

```bash
mcporter call '<server>.<tool>(param: "value", param2: "value2")'
```

---

## Available Servers

The following MCP servers are configured (check live status with `mcporter list`):

| Server | What it does |
|---|---|
| `aws-mcp` | Execute 15K+ AWS APIs with SigV4 auth, guided SOPs, troubleshooting |
| `aws-knowledge` | Search AWS best practices, documentation, regional availability, agent skills |
| `aws-documentation` | Deeper doc reading (full text), works offline, broader content parsing |
| `aws-iac` | CloudFormation docs, CDK best practices, security validation, IaC troubleshooting |
| `aws-pricing` | Real-time pricing data, cost estimation, CDK/Terraform project cost analysis |

**Browser automation?** Use the `playwright-cli` skill instead — direct CLI, no MCP overhead. See `~/.pi/agent/skills/playwright-cli/SKILL.md`.

### Server availability

Some servers require `uvx` (Python) and start on first call. If a server shows "offline" in `mcporter list`, calling a tool on it will auto-start it.

---

## Usage Patterns

### Discover tools

```bash
# See all servers and their status
mcporter list

# See tools on a specific server with full parameter docs
mcporter list aws-mcp --schema
mcporter list aws-knowledge --schema

# Search for a tool by name pattern
mcporter list aws-knowledge --schema | grep -i "search"
```

### Call AWS APIs

```bash
# List Lambda functions
mcporter call aws-mcp.aws___call_aws cli_command="aws lambda list-functions"

# Describe an EC2 instance
mcporter call aws-mcp.aws___call_aws cli_command="aws ec2 describe-instances --instance-ids i-1234567890abcdef0"

# Get command suggestions
mcporter call aws-mcp.aws___suggest_aws_commands query="find unused EBS volumes"

# Search AWS documentation
mcporter call aws-knowledge.aws___search_documentation search_phrase="Lambda best practices"

# Read specific AWS docs
mcporter call aws-knowledge.aws___read_documentation requests='[{"url":"https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html"}]'
```

### Infrastructure as Code (aws-iac)

```bash
# Get CDK best practices
mcporter call aws-iac.cdk_best_practices

# Validate a CloudFormation template
mcporter call aws-iac.validate_cloudformation_template template_content="$(cat template.yaml)"

# Search CDK samples
mcporter call aws-iac.search_cdk_samples_and_constructs query="Lambda with DynamoDB" language=typescript

# Troubleshoot a failed CFN deployment
mcporter call aws-iac.troubleshoot_cloudformation_deployment stack_name="my-stack" region="us-east-1"
```

### Pricing & Cost Estimation (aws-pricing)

```bash
# Get Lambda pricing in us-east-1
mcporter call aws-pricing.get_pricing service_code=AWSLambda region=us-east-1

# List available service codes
mcporter call aws-pricing.get_pricing_service_codes

# Analyze a CDK project for cost
mcporter call aws-pricing.analyze_cdk_project project_path="./my-cdk-app"

# Analyze a Terraform project for cost
mcporter call aws-pricing.analyze_terraform_project project_path="./my-terraform"
```

### AWS Documentation (aws-documentation — deeper reading)

```bash
# Read a specific doc page
mcporter call aws-documentation.read_documentation url="https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html"

# Search docs with intent
mcporter call aws-documentation.search_documentation search_phrase="S3 bucket policies" limit=5

# Read specific sections of a long doc
mcporter call aws-documentation.read_sections url="https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html" section_titles='["Handler naming conventions"]'
```

### AWS guided workflows (Skills)

```bash
# First search for a skill
mcporter call aws-knowledge.aws___search_documentation search_phrase="set up production VPC" topics='["agent_skills"]'

# Then retrieve the skill using the exact skill_name from search results
mcporter call aws-knowledge.aws___retrieve_skill skill_name="exact-skill-name-from-search"

# Get command suggestions for a task
mcporter call aws-mcp.aws___suggest_aws_commands query="set up a production VPC"
```

---

## Tips

1. **Use `--schema` first** to understand tool parameters before calling
2. **JSON values**: Use single quotes around JSON: `parameters='{"key":"value"}'`
3. **Long output**: Pipe through `head` or `jq` for readability
4. **Errors**: If a server is offline, try calling it — mcporter will attempt to start it
5. **Config**: Server definitions live in the mcporter config file (check `mcporter --help` for paths)
6. **Timeout**: Calls timeout at 30s by default; set `MCPORTER_CALL_TIMEOUT=60000` for slow operations

---

## Config Location

MCPorter reads config from (in priority order):
1. `~/.mcporter/mcporter.json` (or `$XDG_CONFIG_HOME/mcporter/mcporter.json`)
2. `./config/mcporter.json` (project-local)
3. Imports from Cursor, Claude Code, Codex, VS Code configs

To add a new MCP server, edit the config file:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "uvx",
      "args": ["my-mcp-server@latest"]
    }
  }
}
```
