# Prompt Template

## Template

```
You are an AI coding agent on a fresh machine with zero prior context. You need to test a CLI tool called '{tool_name}'.

SETUP:
- Install from: {install_source}
- Expected version: {expected_version} — verify with `{version_command}`.
  If version doesn't match, STOP immediately and report: "VERSION MISMATCH: expected {expected_version}, got <actual>"
{extra_setup}

CRITICAL RULES:
- ALWAYS add {agent_flag} to EVERY {tool_name} command (machine-readable output)
- Read --help output carefully before running any command
- Do NOT skip steps or assume anything — discover everything from help text
- Do NOT ask questions — figure everything out from help and error messages
{extra_rules}

YOUR TASK:
{numbered_steps}

FEEDBACK REQUIREMENTS:
After completing (or failing) each step, document:
1. The exact command you ran
2. The exact output (or error) you got
3. Whether it worked, failed, or needed a workaround
4. If workaround needed: what you did and why
5. UX assessment: was the help text clear? Was the error actionable?

Write a structured report to ./agent-feedback.md containing:
- Executive summary (pass/fail, overall score 1-10)
- Complete command log (every command in execution order)
- Bug list with severity (CRITICAL/HIGH/MEDIUM/LOW)
  - CRITICAL = blocks agent entirely, requires manual file editing or external knowledge
  - HIGH = significant friction, may need workaround
  - MEDIUM = confusing but solvable from error messages
  - LOW = cosmetic or minor UX
- Workarounds used (what and why)
- What worked well (positive UX observations)
- Security observations
- Performance notes (install time, command latency, operation durations)
- Final verdict: "Could an agent complete this without manual fixes? Yes/No"
  If No, list the specific blockers.
```

## Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `tool_name` | CLI binary name | `reposwarm` |
| `install_source` | How to install | `curl -fsSL https://...` |
| `expected_version` | Pinned version string | `1.3.109` |
| `version_command` | Command to check version | `reposwarm version --for-agent` |
| `agent_flag` | Flag for machine output | `--for-agent` |
| `extra_setup` | Environment details | IAM role, region, credentials |
| `extra_rules` | Tool-specific constraints | "Use Bedrock not API keys" |
| `numbered_steps` | The actual test scenario | 1. Install 2. Setup 3. Config... |

## Example: RepoSwarm CLI

```
tool_name: reposwarm
install_source: curl -fsSL https://raw.githubusercontent.com/reposwarm/reposwarm-cli/main/install.sh | sh
expected_version: 1.3.109
version_command: reposwarm version --for-agent
agent_flag: --for-agent
extra_setup: |
  - This machine has an IAM role with full AWS access (us-east-1)
  - Use AWS Bedrock with IAM role auth — do NOT use API keys
  - GITHUB_TOKEN is not needed for public repos
extra_rules: |
  - Use model alias "sonnet" when configuring provider
  - If docker containers need env changes, use `docker compose down <svc> && docker compose up -d <svc>`
numbered_steps: |
  1. Install the reposwarm CLI
  2. Run reposwarm --help and explore subcommands
  3. Set up a local instance
  4. Configure provider: AWS Bedrock, IAM role, us-east-1, model sonnet
  5. Add repository https://github.com/jonschlinkert/is-odd
  6. Start investigation and monitor progress
  7. Check results when complete
```
