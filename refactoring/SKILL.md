---
name: refactoring
description: Inspect, review, and refactor code using patterns from Working Effectively with Legacy Code (Feathers), Refactoring (Fowler), and Clean Code (Martin). Use when editing code, reviewing code for quality, planning refactoring, or assessing technical debt. Provides specific techniques, thresholds, and step-by-step safe refactoring workflows.
---

# Refactoring

A disciplined approach to improving code structure without changing behavior, grounded in three canonical texts:

- **Working Effectively with Legacy Code** (Michael Feathers) — safely changing code without tests
- **Refactoring: Improving the Design of Existing Code** (Martin Fowler) — catalog of transformations
- **Clean Code** (Robert C. Martin) — readability and design principles

## When to Use This Skill

- Reviewing code for quality issues (code review, PR review)
- Planning a refactoring campaign across a codebase
- Editing code and noticing adjacent smells worth fixing
- Assessing technical debt and prioritizing cleanup
- Breaking apart god classes, long methods, or tangled modules

## Core Principles

### The Boy Scout Rule
> Leave the code cleaner than you found it.

When editing code for any reason, fix adjacent smells if the fix is safe and small.

### Green Bar Discipline
> Never refactor on a red bar.

1. Ensure tests pass before touching anything
2. Make ONE structural change
3. Run tests immediately
4. Commit if green
5. Repeat

### Behavioral Preservation
Refactoring changes structure, never behavior. If you're tempted to fix a bug while refactoring — stop, commit the refactoring, THEN fix the bug in a separate commit.

---

## Code Quality Thresholds

Apply these when reviewing or inspecting code:

| Metric | Threshold | Severity | Action |
|--------|-----------|----------|--------|
| File length | >400 lines | FAIL | Split module (SRP violation) |
| Function/method length | >50 lines | FAIL | Extract Method |
| Function/method length | >30 lines | WARN | Consider extraction |
| Class methods | >8 public methods | FAIL | God class — extract collaborators |
| Nesting depth | >3 levels | WARN | Extract, use guard clauses |
| Nested try/catch | >2 levels | FAIL | Replace with chain/pipeline |
| Parameters | >4 params | WARN | Introduce Parameter Object |
| Cyclomatic complexity | >10 per function | FAIL | Decompose conditionals |
| Identical patterns | 3+ occurrences | WARN | DRY — extract shared abstraction |
| Platform branching | >2× in same file | FAIL | Replace Conditional with Polymorphism |
| Mixed abstraction levels | orchestration + detail | WARN | Separate into layers |

---

## Smell Catalog (Detection)

When inspecting code, look for these smells in priority order:

### Critical (fix immediately)
1. **God Class** — one class/module does everything; >8 methods, >400 lines
2. **Long Method** — can't describe in one sentence; >50 lines
3. **Feature Envy** — method uses another class's data more than its own
4. **Shotgun Surgery** — one change requires editing many files

### Serious (fix soon)
5. **Primitive Obsession** — using strings/numbers where a type would clarify
6. **Data Clumps** — same 3+ params travel together across functions
7. **Divergent Change** — one module changes for multiple unrelated reasons
8. **Parallel Inheritance** — adding a subclass in one hierarchy forces another

### Moderate (fix when nearby)
9. **Comments explaining what** — code should be self-documenting
10. **Dead Code** — unreachable branches, unused params, stale imports
11. **Message Chains** — `a.b().c().d()` — tight coupling to structure
12. **Speculative Generality** — abstractions for hypothetical future needs

---

## Safe Refactoring Workflow (Feathers)

When you lack tests or confidence, follow this sequence:

### Step 1: Characterize
Write tests that pin current behavior (even if ugly):
```
// Characterization test: documents ACTUAL behavior, not ideal behavior
test("handles null input by returning empty array", () => {
  expect(processItems(null)).toEqual([]);  // Even if this seems wrong
});
```

### Step 2: Find Seams
A **seam** is a place where you can alter behavior without editing the code at that point.

Types of seams:
- **Object seam** — pass a different implementation (dependency injection)
- **Link seam** — import from a different module (re-exports)
- **Preprocessing seam** — environment variables, feature flags

### Step 3: Break Dependencies
Use these techniques to make code testable:

| Technique | When to Use |
|-----------|------------|
| **Extract Interface** | Class has too many responsibilities |
| **Parameterize Constructor** | Hard-coded dependencies |
| **Extract and Override Call** | Can't test a method due to one line |
| **Introduce Instance Delegator** | Static method prevents testing |
| **Wrap Method** | Add behavior before/after existing method |
| **Sprout Method/Class** | New feature in untestable code |

### Step 4: Refactor Under Test
Now that you have tests and seams, apply Fowler's catalog.

---

## Refactoring Catalog (Key Techniques)

### Extraction Family
- **Extract Method** — pull lines into a named function
- **Extract Variable** — name a complex expression
- **Extract Class** — split a class into two collaborators
- **Extract Module** — move functions to a new file (Sprout Module)

### Movement Family
- **Move Method** — relocate to the class that uses the data
- **Move Field** — same, for data
- **Pull Up / Push Down** — move along inheritance hierarchy

### Simplification Family
- **Replace Conditional with Polymorphism** — eliminate if/switch on type
- **Replace Nested Conditional with Guard Clauses** — early returns flatten logic
- **Decompose Conditional** — name the branches
- **Replace Parameter with Method** — caller computes what callee needs
- **Introduce Parameter Object** — group related params into a type
- **Replace Temp with Query** — eliminate intermediate variables

### Organization Family
- **Inline Method** — when indirection adds no value
- **Rename** — the most powerful refactoring; names reveal intent
- **Replace Magic Number with Constant** — self-documenting literals
- **Encapsulate Collection** — don't expose raw arrays/maps

---

## Patterns for Common Situations

### Breaking a God Class
1. List all public methods and group by responsibility
2. For each group, create a new class/module
3. Use **Wrap Method** to delegate from old → new (preserves callers)
4. Once all callers migrate, remove delegation

### Untangling a Long Method
1. Draw dependency arrows between statement groups
2. Identify "paragraphs" (groups separated by blank lines)
3. Name each paragraph — that's your extracted method name
4. Extract from bottom up (fewest dependencies first)
5. If a paragraph uses many locals → Introduce Parameter Object

### Eliminating Platform Branching
```typescript
// Before: scattered if/else
if (process.platform === "darwin") { ... }
else { ... }

// After: polymorphic interface
interface ServiceManager { start(); stop(); status(); }
class LaunchdManager implements ServiceManager { ... }
class SystemdManager implements ServiceManager { ... }
```

### Replacing Nested Conditionals
```typescript
// Before: 5-level nesting
function loadConfig() {
  try { ... try { ... try { ... } } }
}

// After: resolver chain (guard clauses)
function loadConfig() {
  const fromEnv = resolveFromEnv();
  if (fromEnv) return fromEnv;
  const fromFlag = resolveFromFlag();
  if (fromFlag) return fromFlag;
  return DEFAULT_CONFIG;
}
```

### Adding Behavior to Untestable Code (Sprout)
When you can't refactor existing code safely:
1. Write the new behavior in a **new** function/class
2. Test the new code thoroughly
3. Call the new code from the old code (minimal edit)
4. Later, migrate old code's other responsibilities

---

## Review Checklist

When reviewing code (yours or others'), check in this order:

1. **[ ] Naming** — Can I understand each function/variable without reading the body?
2. **[ ] Size** — Any file >400 lines? Any function >50 lines?
3. **[ ] SRP** — Does each module/class have one reason to change?
4. **[ ] DRY** — Any structural duplication (3+ similar patterns)?
5. **[ ] Abstraction levels** — Does each function operate at one level?
6. **[ ] Error handling** — Clean separation of happy path and error path?
7. **[ ] Dependencies** — Minimal coupling? Can I test this in isolation?
8. **[ ] Dead code** — Any unused imports, unreachable branches, stale comments?

---

## Commit Strategy

- One refactoring step per commit (never mix with behavioral changes)
- Commit message format: `refactor(scope): technique applied`
- Example: `refactor(gateway): extract command handlers (Extract Method)`
- Run tests between EVERY commit — never batch untested changes
- If tests break, `git reset --hard` and try a smaller step

---

## When NOT to Refactor

- Code is being deleted soon
- No tests exist AND you can't write characterization tests
- You're on a deadline and the refactoring isn't on the critical path
- The code works, is rarely changed, and isn't blocking anything
- You'd be refactoring speculatively (for hypothetical future needs)

> Refactoring is investment. Invest where the code changes frequently.

See [references/smell-priority.md](references/smell-priority.md) for a prioritization framework.
See [references/feathers-techniques.md](references/feathers-techniques.md) for all 25 Feathers dependency-breaking techniques.
