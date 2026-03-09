---
name: reposwarm
description: Manage RepoSwarm — trigger investigations, check workflow progress, manage repos, and browse architecture results using the reposwarm CLI. Use when the user asks about repo investigations, architecture analysis, workflow status, or RepoSwarm management.
---

# RepoSwarm CLI Skill

RepoSwarm is an AI-powered multi-repo architecture discovery platform. It investigates codebases using Claude/Bedrock, producing architecture documentation for each repository.

## CLI: `reposwarm`

Binary at `/usr/local/bin/reposwarm`. Config at `~/.reposwarm/config.json`.

**Output modes:**
- Default: agent-friendly plain text (no colors/emojis). Use this.
- `--human`: rich terminal output (colors, progress bars)
- `--json`: structured JSON

## Common Commands

```bash
# Check progress of active daily investigation
reposwarm wf progress

# List tracked repositories
reposwarm repos list

# Show specific repo details
reposwarm repos show <name>

# Discover and add CodeCommit repos
reposwarm repos discover

# Trigger investigation
reposwarm investigate <repo>          # single repo
reposwarm investigate --all           # all enabled repos

# List workflows
reposwarm wf list

# Watch a specific workflow
reposwarm workflows watch <workflow-id>

# Terminate a stuck workflow
reposwarm wf terminate <workflow-id> -y

# Browse investigation results
reposwarm results list                # repos with results
reposwarm results sections <repo>     # list sections for a repo
reposwarm results read <repo>         # read all sections
reposwarm results read <repo> <sect>  # read one section

# Validate section coverage across all repos
reposwarm results audit               # checks all repos have all expected sections
reposwarm results audit --json        # machine-readable with per-repo details

# Search across results (use filters to avoid slow full scans)
reposwarm results search "Cognito" --repo bedlam-infra
reposwarm results search "DynamoDB" --section DBs --max 20

# Generate report
reposwarm results report              # all repos
reposwarm results report <repo>       # specific repo

# Compare repos
reposwarm results diff <repo1> <repo2>

# Health & diagnostics
reposwarm status
reposwarm doctor

# Prompts/sections management (derives from results if API empty)
reposwarm prompts list
reposwarm prompts show <name>

# Server config
reposwarm config server
```

## Infrastructure Context

- **Cluster:** `reposwarm-cluster` (ECS Fargate, us-east-1)
- **VPC:** `reposwarm-vpc` (10.195.0.0/16) — NOT peered to OpenClaw VPC
- **Services:** temporal-server, temporal-ui-v2, reposwarm-worker, reposwarm-api, reposwarm-ui
- **UI:** https://dkhtk1q9b2nii.cloudfront.net
- **Worker:** Python, uses Temporal workflows + Bedrock Claude for analysis
- **Worker has NO pipeline** — manual image builds only
- **Daily workflow:** `InvestigateReposWorkflow` with `sleep_hours: 24`, auto-repeats via `continue_as_new`

## Gotchas

- Only two workflow types exist: `InvestigateReposWorkflow` and `InvestigateSingleRepoWorkflow`
- `CODECOMMIT_ENABLED=true` must be set on worker for auto-discovery
- Stuck workflows need termination + restart (Temporal replay won't fix code bugs)
- ECS Exec into containers requires SSM endpoints + IAM policy on task role
- Full deployment docs: `~/.openclaw/workspace/infra/reposwarm/INFRASTRUCTURE.md`
