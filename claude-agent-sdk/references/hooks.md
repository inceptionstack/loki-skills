# Hooks Reference

Hooks are callbacks that run at key agent lifecycle points.

## Available Hooks

| Hook Event | Python | TypeScript | Trigger |
|---|---|---|---|
| `PreToolUse` | ✅ | ✅ | Before tool executes (can block/modify) |
| `PostToolUse` | ✅ | ✅ | After tool returns result |
| `PostToolUseFailure` | ✅ | ✅ | After tool execution fails |
| `UserPromptSubmit` | ✅ | ✅ | User prompt submitted |
| `Stop` | ✅ | ✅ | Agent execution stops |
| `SubagentStart` | ✅ | ✅ | Subagent initialized |
| `SubagentStop` | ✅ | ✅ | Subagent completed |
| `PreCompact` | ✅ | ✅ | Before conversation compaction |
| `PermissionRequest` | ✅ | ✅ | Permission dialog would show |
| `Notification` | ✅ | ✅ | Agent status message |
| `SessionStart` | ❌ | ✅ | Session init |
| `SessionEnd` | ❌ | ✅ | Session termination |

## Callback Signature (Python)

```python
async def my_hook(input_data: dict, tool_use_id: str, context: dict) -> dict:
    # input_data contains: hook_event_name, tool_input (for tool hooks)
    # Return {} to allow, or hookSpecificOutput to block/modify
    return {}
```

## Matcher Patterns

Matchers are regex strings that filter which tools trigger the hook:

```python
HookMatcher(matcher="Write|Edit", hooks=[my_callback])  # only Write and Edit
HookMatcher(matcher="Bash", hooks=[my_callback])         # only Bash
HookMatcher(hooks=[my_callback])                          # all tools (no matcher)
```

## Blocking a Tool

```python
async def block_dangerous(input_data, tool_use_id, context):
    return {"hookSpecificOutput": {
        "hookEventName": input_data["hook_event_name"],
        "permissionDecision": "deny",
        "permissionDecisionReason": "Operation blocked by policy",
    }}
```

## Modifying Tool Input

```python
async def add_readonly_flag(input_data, tool_use_id, context):
    return {"hookSpecificOutput": {
        "hookEventName": input_data["hook_event_name"],
        "suppressOutput": False,
    }}
```

## Logging/Audit

```python
async def audit_log(input_data, tool_use_id, context):
    tool_name = input_data.get("tool_name", "unknown")
    file_path = input_data.get("tool_input", {}).get("file_path", "")
    print(f"AUDIT: {tool_name} on {file_path}")
    return {}  # allow everything, just log
```
