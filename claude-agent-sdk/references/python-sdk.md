# Python SDK Reference

## `query()` — One-shot agent

```python
async def query(
    *,
    prompt: str | AsyncIterable[dict[str, Any]],
    options: ClaudeAgentOptions | None = None,
    transport: Transport | None = None
) -> AsyncIterator[Message]
```

Creates a new session each call. Returns async iterator of messages.

## `ClaudeSDKClient` — Multi-turn conversation

```python
class ClaudeSDKClient:
    def __init__(self, options=None, transport=None)
    async def connect(self, prompt=None) -> None
    async def query(self, prompt, session_id="default") -> None
    async def receive_messages(self) -> AsyncIterator[Message]
    async def receive_response(self) -> AsyncIterator[Message]
    async def interrupt(self) -> None
    async def set_permission_mode(self, mode: str) -> None
    async def set_model(self, model: str | None) -> None
    async def disconnect(self) -> None
```

Use as async context manager:
```python
async with ClaudeSDKClient(options=opts) as client:
    await client.query("prompt")
    async for msg in client.receive_response():
        ...
```

## `ClaudeAgentOptions`

```python
ClaudeAgentOptions(
    # Tools & permissions
    allowed_tools: list[str] = None,         # ["Read", "Edit", "Bash", ...]
    permission_mode: str = "default",         # "default"|"acceptEdits"|"bypassPermissions"

    # Prompts
    system_prompt: str = None,                # custom system prompt
    cwd: str = None,                          # working directory

    # Model
    model: str = None,                        # override model ID

    # Limits
    max_turns: int = None,                    # max agent turns

    # MCP servers
    mcp_servers: dict = None,                 # {"name": server_config}

    # Hooks
    hooks: dict = None,                       # {"PreToolUse": [HookMatcher(...)]}

    # Subagents
    agents: dict = None,                      # {"name": AgentDefinition(...)}

    # Sessions
    resume: str = None,                       # session ID to resume
)
```

## `@tool` decorator

```python
@tool(name: str, description: str, input_schema: type | dict)
async def my_tool(args: dict) -> dict:
    return {"content": [{"type": "text", "text": "result"}]}
```

Input schema — simple type mapping:
```python
{"text": str, "count": int, "enabled": bool}
```

Or JSON Schema:
```python
{"type": "object", "properties": {"text": {"type": "string"}}, "required": ["text"]}
```

## `create_sdk_mcp_server()`

```python
server = create_sdk_mcp_server(
    name="myserver",
    version="1.0.0",
    tools=[my_tool_1, my_tool_2],
)
```

Use in options: `mcp_servers={"myserver": server}`
Reference tools as: `mcp__myserver__toolname`

## `AgentDefinition`

```python
AgentDefinition(
    description="What the subagent does",
    prompt="System prompt for the subagent",
    tools=["Read", "Glob", "Grep"],  # tools available to subagent
)
```

## Message Types

```python
from claude_agent_sdk import (
    AssistantMessage,    # Claude's response (has .content blocks)
    ResultMessage,       # Final result (has .result, .subtype)
    TextBlock,           # Text content block (has .text)
    HookMatcher,         # Hook configuration
    AgentDefinition,     # Subagent definition
)
```

## LiteLLM Integration

```python
import os
os.environ["ANTHROPIC_BASE_URL"] = "http://localhost:4000"  # LiteLLM proxy
os.environ["ANTHROPIC_API_KEY"] = "sk-1234"                 # LiteLLM key

options = ClaudeAgentOptions(
    model="bedrock-claude-sonnet-4",  # any model from LiteLLM config.yaml
)
```
