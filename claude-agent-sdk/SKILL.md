---
name: claude-agent-sdk
description: "Build AI agents using the Claude Agent SDK (Python and TypeScript). Use when: (1) building autonomous agents that read files, run commands, search the web, or edit code, (2) creating code review or bug-fixing agents, (3) deploying agents with custom tools via MCP, (4) using Claude as a library for agentic workflows, (5) integrating with Bedrock, Vertex AI, Azure, or LiteLLM proxy. NOT for: simple one-shot API calls (use Anthropic SDK directly), non-agentic chat completions, or Strands SDK tasks."
---

# Claude Agent SDK

Build production AI agents with Claude Code as a library. The SDK provides built-in tools (Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch), an agent loop, and context management.

## Install

```bash
# Python
pip install claude-agent-sdk

# TypeScript
npm install @anthropic-ai/claude-agent-sdk
```

Requires Node.js 18+ (both SDKs depend on Claude Code CLI internally).

## Auth

Set ONE of these:

```bash
# Direct Anthropic API
export ANTHROPIC_API_KEY=sk-ant-...

# Amazon Bedrock
export CLAUDE_CODE_USE_BEDROCK=1
# + AWS credentials (env vars, profile, or instance role)

# Google Vertex AI
export CLAUDE_CODE_USE_VERTEX=1

# Microsoft Azure
export CLAUDE_CODE_USE_FOUNDRY=1

# LiteLLM proxy
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_API_KEY=sk-1234  # your LiteLLM key
```

## Core Pattern: `query()`

One-shot agent execution. New session each call.

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find all TODO comments and create a summary",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
            cwd="/path/to/project",
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

## Core Pattern: `ClaudeSDKClient`

Multi-turn conversation with session continuity.

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, AssistantMessage, TextBlock

async def main():
    async with ClaudeSDKClient(options=ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep"]
    )) as client:
        await client.query("Read the auth module")
        async for msg in client.receive_response():
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(block.text)

        # Follow-up — session retains context
        await client.query("Now find all places that call it")
        async for msg in client.receive_response():
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(block.text)

asyncio.run(main())
```

## Built-in Tools

| Tool | What it does |
|------|-------------|
| `Read` | Read any file in the working directory |
| `Write` | Create new files |
| `Edit` | Make precise edits to existing files |
| `Bash` | Run terminal commands, scripts, git ops |
| `Glob` | Find files by pattern (`**/*.ts`) |
| `Grep` | Search file contents with regex |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch and parse web pages |
| `Task` | Spawn subagents |
| `AskUserQuestion` | Ask the user clarifying questions |

## Permission Modes

| Mode | Behavior |
|------|----------|
| `acceptEdits` | Auto-approves file edits, asks for other actions |
| `bypassPermissions` | Runs every tool without prompts (sandboxed/CI) |
| `default` | Requires `canUseTool` callback for approval |

For headless/container use, always use `bypassPermissions` or `acceptEdits`.

## Key Options (`ClaudeAgentOptions`)

```python
ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash", "Glob", "Grep"],
    permission_mode="acceptEdits",
    system_prompt="You are an expert architect...",
    cwd="/path/to/workdir",
    model="claude-sonnet-4-20250514",  # override model
    max_turns=50,                       # limit agent turns
    mcp_servers={...},                  # external MCP tools
    hooks={...},                        # lifecycle hooks
    agents={...},                       # subagent definitions
)
```

## Custom Tools (MCP)

Define tools inline using the `@tool` decorator:

```python
from claude_agent_sdk import tool, create_sdk_mcp_server, ClaudeAgentOptions

@tool("lookup_user", "Look up user by ID", {"user_id": str})
async def lookup_user(args):
    user = await db.get_user(args["user_id"])
    return {"content": [{"type": "text", "text": f"User: {user['name']}"}]}

server = create_sdk_mcp_server("mytools", tools=[lookup_user])

options = ClaudeAgentOptions(
    mcp_servers={"mytools": server},
    allowed_tools=["Read", "Glob", "mcp__mytools__lookup_user"],
)
```

## Subagents

Spawn specialized agents for subtasks:

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "code-reviewer": AgentDefinition(
            description="Expert code reviewer.",
            prompt="Analyze code quality and suggest improvements.",
            tools=["Read", "Glob", "Grep"],
        )
    },
)
```

Messages from subagents include `parent_tool_use_id` for tracking.

## Hooks

Intercept agent behavior at lifecycle points. For details see `references/hooks.md`.

```python
async def block_env_writes(input_data, tool_use_id, context):
    file_path = input_data["tool_input"].get("file_path", "")
    if file_path.endswith(".env"):
        return {"hookSpecificOutput": {
            "hookEventName": input_data["hook_event_name"],
            "permissionDecision": "deny",
            "permissionDecisionReason": "Cannot modify .env files",
        }}
    return {}

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Write|Edit", hooks=[block_env_writes])]}
)
```

## Hosting in Docker

For container/cloud deployment, see `references/hosting.md`.

Minimal Dockerfile:

```dockerfile
FROM node:22-slim
RUN npm install -g @anthropic-ai/claude-code
RUN pip install claude-agent-sdk
COPY agent.py .
CMD ["python", "agent.py"]
```

Resource needs: 1GiB RAM, 5GiB disk, 1 CPU minimum. Outbound HTTPS to `api.anthropic.com` (or your LiteLLM/Bedrock endpoint).

## LiteLLM Proxy Integration

Point the SDK at a LiteLLM proxy to use any model:

```python
import os
os.environ["ANTHROPIC_BASE_URL"] = "http://localhost:4000"
os.environ["ANTHROPIC_API_KEY"] = "sk-1234"

options = ClaudeAgentOptions(
    model="bedrock-claude-sonnet-4",  # any model from LiteLLM config
)
```

## Choosing `query()` vs `ClaudeSDKClient`

| | `query()` | `ClaudeSDKClient` |
|---|---|---|
| Session | New each time | Reuses same session |
| Follow-ups | ❌ | ✅ Maintains context |
| Interrupts | ❌ | ✅ `client.interrupt()` |
| Use case | One-off tasks | Continuous conversations |

## Message Types

When iterating the stream, filter for:

```python
from claude_agent_sdk import AssistantMessage, ResultMessage, TextBlock

async for message in query(...):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)       # Claude's reasoning
            elif hasattr(block, "name"):
                print(f"Tool: {block.name}")  # tool call
    elif isinstance(message, ResultMessage):
        print(message.result)           # final answer
```
