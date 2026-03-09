# Hosting & Deployment

## System Requirements

Per SDK instance:
- 1GiB RAM, 5GiB disk, 1 CPU (minimum)
- Python 3.10+ or Node.js 18+
- Claude Code CLI: `npm install -g @anthropic-ai/claude-code`
- Outbound HTTPS to `api.anthropic.com` (or Bedrock/LiteLLM endpoint)

## Deployment Patterns

### Pattern 1: Ephemeral Sessions
New container per task, destroyed on completion. Best for one-off jobs (bug fixes, code review, file processing).

### Pattern 2: Long-Running Sessions
Persistent container running multiple agent processes. Best for proactive agents (email bots, chat handlers, site builders).

### Pattern 3: Hybrid Sessions
Ephemeral containers hydrated with session history from database or SDK session resumption. Best for intermittent interaction (project management, research, support tickets).

## Docker Example

```dockerfile
FROM node:22-slim
RUN npm install -g @anthropic-ai/claude-code
RUN apt-get update && apt-get install -y python3 python3-pip && \
    pip3 install claude-agent-sdk && \
    rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY agent.py .
CMD ["python3", "agent.py"]
```

## Permission Modes for Containers

Use `bypassPermissions` for fully sandboxed environments:

```python
options = ClaudeAgentOptions(
    permission_mode="bypassPermissions",
    allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
)
```

Use `acceptEdits` for semi-trusted environments where you want file edits auto-approved but other actions need approval.

## Sandbox Providers

- Modal Sandbox (modal.com)
- Cloudflare Sandboxes
- Daytona (daytona.io)
- E2B (e2b.dev)
- Fly Machines (fly.io)
- Vercel Sandbox

## Cost Considerations

- Dominant cost: LLM tokens (not container hosting)
- Container cost: ~$0.05/hour minimum
- Set `max_turns` to prevent runaway loops
- Monitor with hooks (`PostToolUse` for token tracking)

## Session Resumption

Resume sessions across container restarts:

```python
# Capture session ID
async for message in query(prompt="Start analysis", options=options):
    if hasattr(message, "subtype") and message.subtype == "init":
        session_id = message.session_id

# Later, resume with full context
async for message in query(
    prompt="Continue the analysis",
    options=ClaudeAgentOptions(resume=session_id),
):
    ...
```
