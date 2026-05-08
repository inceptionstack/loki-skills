---
name: module-organizer
description: Prevent namespace drift in TypeScript projects. Use when adding new files, moving modules, reviewing structure, or planning reorganizations. Provides placement rules, a decision checklist, and anti-patterns to avoid.
---

# Module Organizer

A discipline for keeping source tree layout coherent as projects grow. Prevents the drift that happens when files are placed by convenience (current caller, hurry, copy-paste) instead of by domain ownership.

## When to Use This Skill

- Adding a new source file or directory
- Moving/renaming modules during refactoring
- Reviewing PRs that add new files
- Planning a structural reorganization sprint
- Resolving "where does X live?" confusion

## Core Principle

> Place files where a new maintainer would look for the **capability itself**, not where the current caller lives.

Test: "If someone asks 'where does Telegram pairing live?' does the file path answer that question?"

---

## Domain Classification

Every source file belongs to exactly one domain. Identify it first:

| Domain | Description | Example Path |
|--------|-------------|--------------|
| **Core** | App-wide types, config, entrypoint | `src/types.ts`, `src/config.ts` |
| **Gateway/Runtime** | Request handling, streaming, lifecycle | `src/gateway/` |
| **Transport** | Platform-specific adapters (Telegram, Slack, etc.) | `src/transports/telegram/` |
| **Agent** | AI agent adapters and SDK integration | `src/agents/pi/` |
| **CLI** | Command-line interface, commands, TUI | `src/cli/commands/` |
| **CLI Support** | Setup, doctor, prompts — serves CLI | `src/cli/setup/` |
| **Feature** | Self-contained features (cron, memory, voice) | `src/cron/`, `src/memory/` |
| **Provisioning** | Install, update, bundle operations | `src/provisioning/` |
| **Shared** | Pure utilities used across 3+ domains | `src/shared/` |

---

## Placement Checklist

Run through these 10 checks for every new or moved file:

### 1. Identify the owning domain
What capability does this file provide? Match to table above.

### 2. Reject caller-based placement
❌ "Gateway calls it, so put it in gateway/"
✅ "It's Telegram formatting, so put it in transports/telegram/"

### 3. Check for namespace collisions
Never create `foo.ts` beside `foo/` unless `foo.ts` IS the entrypoint (and should become `foo/index.ts`).

### 4. Check for transport leakage
If the filename contains a platform name (`telegram`, `slack`, `discord`), it belongs under that transport's folder.

### 5. Check for CLI ↔ runtime leakage
- If gateway/runtime imports it → it should NOT be under `cli/`
- If only CLI uses it → it should NOT be in root `src/`

### 6. Check singleton-folder pressure
Don't create a folder for one file unless:
- It represents a stable abstraction boundary (e.g., `providers/`)
- You can name 2+ future files that will join it

### 7. Check command metadata vs command behavior
- Bot command definitions (slash commands) → transport folder
- CLI command execution → `cli/commands/`
- Feature logic invoked by commands → feature folder

### 8. Check for catch-all growth
If adding to a file that already has 3+ unrelated responsibilities → split first.

### 9. Apply barrel (`index.ts`) policy
- Add `index.ts` only for **public module boundaries**
- It must export the folder's primary API
- Internal implementation folders skip barrels
- Never create partial barrels that omit the main export

### 10. Verify import direction
```
core ← gateway ← transport
core ← agents
core ← cli → cli-support
core ← features
shared ← everything
```
Arrows show allowed dependency direction. Never invert.

---

## Anti-Patterns

| Anti-Pattern | Example | Fix |
|--------------|---------|-----|
| **Prefix Soup** | `telegram-format.ts`, `telegram-html.ts`, `telegram-progress.ts` in root | Move to `transports/telegram/` |
| **Orphan Namespace** | `notify/telegram.ts` (only file in folder) | Merge into `transports/telegram/notify.ts` |
| **Split Identity** | `gateway.ts` + `gateway/` | Move class into `gateway/index.ts` |
| **Caller Placement** | `cli/setup-telegram.ts` (it's setup code that touches Telegram API) | Move to `cli/setup/telegram.ts` |
| **Ambiguous Name** | `commands.ts` (Telegram bot commands? CLI commands? Both exist) | Qualify: `transports/telegram/bot-commands.ts` |
| **God Util** | `util.ts` with auth + messaging + ids + debug flags | Split by capability into `shared/` |
| **Hidden Dependency** | `gateway.ts` imports from `cli/doctor/` | Extract shared logic to feature-level module |

---

## Decision Template

When proposing a file location, document:

```
File: <filename>
Domain: <from table above>
Path: <full path>
Justification: <why this domain owns it>
New folder needed: <yes/no + reason>
Barrel needed: <yes/no>
Prerequisite renames: <any existing collision to fix first>
```

---

## Structural Review Process

When reviewing an entire tree (e.g., quarterly cleanup):

1. **List all root-level files** — each needs justification for being at root
2. **Find namespace collisions** — `foo.ts` beside `foo/` 
3. **Find prefix clusters** — 3+ files with same prefix = missing folder
4. **Find singleton folders** — folder with 1 file = likely misplaced
5. **Check transport leakage** — platform names outside transport folder
6. **Check import inversions** — runtime importing from CLI namespace
7. **Check catch-all growth** — any file >8 exports or >3 unrelated concerns

---

## Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Feature folder | noun, singular | `memory/`, `voice/`, `cron/` |
| Adapter file | `<name>-adapter.ts` | `pi-adapter.ts`, `kiro-adapter.ts` |
| CLI command | noun matching the command | `cli/commands/doctor.ts` |
| Types file | `types.ts` within owning folder | `cron/types.ts` |
| Barrel | `index.ts` | Only at public boundaries |
| Shared utility | capability name | `shared/auth.ts`, `shared/ids.ts` |
| Transport | platform name folder | `transports/telegram/` |

---

## When NOT to Reorganize

- Code is being deleted soon
- The project is <10 files (overhead exceeds benefit)
- You'd break published import paths without a major version bump
- You're on a deadline and the structure isn't blocking anyone
- A rename would touch >50% of files in one PR (break into phases)

> Reorganization is investment. Invest when confusion is measurably slowing people down.
