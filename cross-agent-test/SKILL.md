---
name: cross-agent-test
description: "Run zero-context cross-agent testing of CLI tools using Claude Code. Use when: (1) testing a CLI tool's agent usability from scratch, (2) running end-to-end smoke tests with a fresh agent, (3) evaluating CLI UX for AI agents, (4) iterating on bug fixes until a clean pass. Triggers on: 'cross-agent test', 'agent test', 'test with claude code', 'zero-context test', 'agent usability test'."
---

# Cross-Agent Testing

Use Claude Code as a zero-context agent to test CLI tools, find bugs, and iterate until clean.

## The Loop

1. Build prompt from user input (tool, scenario, version, rules)
2. Launch Claude Code in fresh dir with `pty:true`, `IS_SANDBOX=1`, background, timeout 900s
3. Monitor every 2-3 min (docker ps, logs, file existence)
4. On completion: read `agent-feedback.md` + session JSONL + container logs
5. Report findings to user
6. Fix bugs, push, rebuild
7. Retest on fresh folder until **zero bugs, zero workarounds**

## Prompt Construction

Ask the user for (or infer from context):
- **Tool name** and install source (URL/script)
- **Task scenario** (numbered steps for the agent)
- **Expected version** — agent verifies and stops if wrong
- **Agent flag** (e.g. `--for-agent`, `--json`)
- **Environment** (IAM role, API keys, region, etc.)
- **Extra rules** (tool-specific constraints)

Then generate prompt from template. See [references/prompt-template.md](references/prompt-template.md).

## Launch

```bash
rm -rf /tmp/cross-agent-test && mkdir -p /tmp/cross-agent-test && cd /tmp/cross-agent-test && git init
```

Use exec with:
- `pty: true` (CRITICAL — Claude Code needs a terminal)
- `background: true`
- `timeout: 900`
- `workdir: /tmp/cross-agent-test`
- `env: IS_SANDBOX=1, CLAUDE_CODE_USE_BEDROCK=1, AWS_REGION=us-east-1`
- Command: `claude -p --dangerously-skip-permissions "{prompt}"`

## Monitor

- Poll every 2-3 min: `ps -p PID`, `docker ps`, container logs
- Don't interfere unless stuck >10 min
- Watch for `agent-feedback.md` as completion signal

## Analyze

After Claude Code exits:

1. **Read `agent-feedback.md`** — the agent's self-report
2. **Extract commands from session JSONL:**
   ```bash
   cat ~/.claude/projects/-tmp-cross-agent-test/*.jsonl | python3 -c "
   import json, sys
   for line in sys.stdin:
       obj = json.loads(line)
       for block in obj.get('message',{}).get('content',[]):
           if isinstance(block,dict) and block.get('type')=='tool_use' and 'Bash' in block.get('name',''):
               cmd = block.get('input',{}).get('command','')
               if cmd: print(f'$ {cmd}\n')
   "
   ```
3. **Check container/worker logs** for errors agent may have missed
4. **Cross-reference**: does the report match reality?
5. **Categorize bugs**: CRITICAL (blocks agent) / HIGH / MEDIUM / LOW

## Report to User

Format: What worked → Bugs found (with root cause) → Priority fix list → Agent's verdict

## Fix & Retest

- Fix all bugs, push, ensure CI passes
- **Fresh folder, fresh install** — no leftovers from prior runs
- Clean up: `docker compose down`, remove config dirs, remove old binary
- Re-run until clean pass

## Key Rules

- **Always pin expected version** — agent verifies on startup
- **Always use `--for-agent`** — drill it into the prompt
- **Fresh everything** — no cached binaries, no leftover configs, no prior containers
- **IS_SANDBOX=1** — enables dangerous mode
- **Git init required** — Claude Code needs a git repo
- **900s timeout** — enough for install + investigation, kill if stuck
- **Never run in ~/clawd workspace** — use /tmp/cross-agent-test
