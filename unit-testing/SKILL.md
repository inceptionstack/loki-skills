# Unit Testing (Osherove 3rd Edition)

Write and review unit tests following Roy Osherove's "The Art of Unit Testing" (3rd Edition) principles, including entry points, exit points, and trust boundaries.

## When to Use

Activate this skill when:
- Writing new unit tests
- Reviewing existing test quality
- Refactoring tests for readability/maintainability
- Deciding what to test and how to structure tests
- Identifying missing test coverage via exit points

## Core Concepts

### Entry Points
The public API surface through which the unit under test is invoked:
- Public methods/functions
- Constructors
- Event subscriptions
- CLI arguments
- IPC messages / HTTP endpoints

### Exit Points (what to verify)
Every unit has 1+ exit points. Each test should verify exactly **one exit point**:

1. **Return value** — function returns a result
2. **State change** — observable state mutation (property, file, DB)
3. **Dependency call** — calls to a dependency (3rd party, network, I/O)

### Trust Boundaries
- **Don't mock what you own** — test real code paths unless I/O is involved
- **Mock at the boundary** — mock only external dependencies (network, filesystem, time)
- **Don't test internals** — test behavior through entry points, not implementation

## Naming Convention

```
[UnitOfWork]_[Scenario]_[ExpectedBehavior]
```

Examples:
```typescript
describe("IpcServer", () => {
  it("start_whenSocketAlreadyInUse_throwsAlreadyRunningError")
  it("start_withStaleSocket_removesAndListens")
  it("handler_withInvalidJson_returnsErrorResponse")
  it("handler_whenPayloadExceeds64KB_destroysConnection")
});
```

Or the shorter readable form (acceptable in vitest/jest):
```typescript
it("returns error response for invalid JSON")
it("destroys connection when payload exceeds 64KB")
```

## Test Structure: AAA (Arrange-Act-Assert)

```typescript
it("returns sum of two numbers", () => {
  // Arrange
  const calc = new Calculator();

  // Act
  const result = calc.add(2, 3);

  // Assert
  expect(result).toBe(5);
});
```

**Rules:**
- One logical assertion per test (multiple `expect` calls OK if verifying one exit point)
- No logic in tests (no `if`, `switch`, `for`, `try/catch`)
- No shared mutable state between tests
- Tests must be independent and run in any order

## Exit Point Categories & Patterns

### 1. Return Value Tests (simplest)
```typescript
it("parses duration string to milliseconds", () => {
  expect(parseDuration("6h")).toBe(21_600_000);
});
```

### 2. State Change Tests
```typescript
it("marks job as paused after pause()", async () => {
  const store = new CronStore(tmpDir);
  await store.writeJob(makeJob({ id: "x", enabled: true }));

  await store.pauseJob("x");

  const job = await store.readJob("x");
  expect(job.enabled).toBe(false);
});
```

### 3. Dependency Call Tests (use fakes/stubs)
```typescript
it("sends notification to transport on notify request", async () => {
  const sent: string[] = [];
  const fakeTransport = { notify: async (_ids: number[], text: string) => { sent.push(text); } };

  await handler({ type: "notify", text: "hello" }, fakeTransport);

  expect(sent).toEqual(["hello"]);
});
```

## Stub vs Mock vs Fake

| Type | Purpose | Assertion target? |
|------|---------|-------------------|
| **Stub** | Returns canned data | No — just enables the test |
| **Mock** | Records calls for verification | Yes — assert it was called correctly |
| **Fake** | Working implementation (in-memory) | No — used as real substitute |

**Rule:** Prefer fakes > stubs > mocks. Only mock when verifying a dependency call exit point.

## Anti-Patterns to Avoid

1. **Testing implementation** — Don't assert private methods, internal state, or call order unless it's the exit point
2. **Overspecified tests** — Don't verify every parameter; verify what matters
3. **Test-per-method** — Don't mirror 1:1 with methods; test behaviors/scenarios
4. **Shared state** — Don't use `beforeAll` to set up mutable state shared across tests
5. **Magic values** — Use named constants or builder functions for test data
6. **Flaky async** — Don't use `setTimeout` to "wait" for async; use proper awaits/events
7. **Testing the framework** — Don't test that vitest/jest assertions work

## Isolation Strategies

### For I/O (filesystem, network):
```typescript
// Use temp directories
const tmpDir = resolve(tmpdir(), `test-${process.pid}`);
afterEach(() => rmSync(tmpDir, { recursive: true, force: true }));

// Use injectable paths/sockets
const server = new IpcServer(handler, TEST_SOCKET_PATH);
```

### For time:
```typescript
// Use vi.useFakeTimers() or inject a clock
vi.useFakeTimers();
vi.advanceTimersByTime(60_000);
vi.useRealTimers();
```

### For external services:
```typescript
// Inject fakes via constructor or parameter
const fakeNotifier = { notify: vi.fn().mockResolvedValue(undefined) };
const runner = new CronRunner(store, fakeNotifier);
```

## Coverage Checklist

For each unit, ask:
1. What are the **entry points**? (How is it invoked?)
2. What are the **exit points**? (What observable things happen?)
3. For each exit point, what are the **scenarios**?
   - Happy path
   - Edge cases (empty input, boundary values)
   - Error paths (invalid input, dependency failure)
   - Concurrency (if applicable)

## Readability Rules

- Test file mirrors source: `src/ipc/server.ts` → `test/ipc-server.test.ts`
- Group by unit: `describe("IpcServer", () => { ... })`
- Helper functions at bottom of file or in `test/helpers/`
- Builder pattern for complex test objects:
  ```typescript
  function makeJob(overrides: Partial<CronJobConfig> = {}): CronJobConfig {
    return { id: "test", enabled: true, prompt: "do stuff", ...overrides };
  }
  ```

## When to Stop Testing

- Don't test trivial code (getters, simple delegation)
- Don't test third-party libraries
- Don't test generated/derived code
- Stop at the trust boundary — if a dependency is tested elsewhere, stub it
