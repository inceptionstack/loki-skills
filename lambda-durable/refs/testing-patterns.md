# Testing Patterns

Test durable functions locally and in the cloud with comprehensive test runners.

## Critical Testing Rules

### DO:
- ✅ Use `runner.getOperation("name")` to find operations by name
- ✅ Use `WaitingOperationStatus.STARTED` when waiting for callback operations
- ✅ JSON.stringify callback parameters: `sendCallbackSuccess(JSON.stringify(data))`
- ✅ Parse callback results: `JSON.parse(result.value)`
- ✅ Name all operations for test reliability
- ✅ Use `skipTime: true` in setupTestEnvironment for fast tests
- ✅ Wrap event data in `payload` object: `runner.run({ payload: { ... } })`
- ✅ Cast `getResult()` to appropriate type: `execution.getResult() as ResultType`

### DON'T:
- ❌ Use `getOperationByIndex()` unless absolutely necessary
- ❌ Assume operation indices are stable
- ❌ Send objects to sendCallbackSuccess — stringify first!
- ❌ Forget that callback results are JSON strings — parse them
- ❌ Test callbacks without proper synchronization (leads to race conditions)

## Local Testing Setup

**TypeScript:**
```typescript
import {
  LocalDurableTestRunner,
  OperationType,
  OperationStatus
} from '@aws/durable-execution-sdk-js-testing';

describe('My Durable Function', () => {
  beforeAll(() => 
    LocalDurableTestRunner.setupTestEnvironment({ skipTime: true })
  );
  
  afterAll(() => 
    LocalDurableTestRunner.teardownTestEnvironment()
  );

  it('should execute workflow', async () => {
    const runner = new LocalDurableTestRunner({ 
      handlerFunction: handler 
    });
    
    const execution = await runner.run({ 
      payload: { userId: '123' } 
    });

    expect(execution.getStatus()).toBe('SUCCEEDED');
    expect(execution.getResult()).toEqual({ success: true });
  });
});
```

**Python:**
```python
import pytest
from aws_durable_execution_sdk_python.execution import InvocationStatus

@pytest.mark.durable_execution(
    handler=handler,
    lambda_function_name='my_function'
)
def test_workflow(durable_runner):
    with durable_runner:
        result = durable_runner.run(input={'user_id': '123'}, timeout=10)

    assert result.status is InvocationStatus.SUCCEEDED
    assert result.result == {'success': True}
```

## Getting Operations (CRITICAL: by NAME, not index)

```typescript
it('should execute steps in order', async () => {
  const runner = new LocalDurableTestRunner({ handlerFunction: handler });
  await runner.run({ payload: { test: true } });

  // ✅ CORRECT: Get by name
  const fetchStep = runner.getOperation('fetch-user');
  expect(fetchStep.getType()).toBe(OperationType.STEP);
  expect(fetchStep.getStatus()).toBe(OperationStatus.SUCCEEDED);

  // ❌ WRONG: Get by index (brittle!)
  // const step1 = runner.getOperationByIndex(0);
});
```

## Testing with Fake Clock

```typescript
it('should wait for specified duration', async () => {
  const runner = new LocalDurableTestRunner({ handlerFunction: handler });
  const executionPromise = runner.run({ payload: {} });

  await runner.skipTime({ seconds: 60 });

  const execution = await executionPromise;
  expect(execution.getStatus()).toBe('SUCCEEDED');

  const waitOp = runner.getOperation('delay');
  expect(waitOp.getType()).toBe(OperationType.WAIT);
  expect(waitOp.getWaitDetails()?.waitSeconds).toBe(60);
});
```

## Testing Callbacks

**CRITICAL:** Use `waitForData()` with `WaitingOperationStatus.STARTED` to avoid flaky tests.

```typescript
import { WaitingOperationStatus } from '@aws/durable-execution-sdk-js-testing';

it('should handle callback success', async () => {
  const runner = new LocalDurableTestRunner({ handlerFunction: handler });

  const executionPromise = runner.run({
    payload: { approver: 'alice@example.com' }
  });

  const callbackOp = runner.getOperation('wait-for-approval');

  // ✅ Wait for operation to reach STARTED status
  await callbackOp.waitForData(WaitingOperationStatus.STARTED);

  // ✅ Must JSON.stringify callback data!
  await callbackOp.sendCallbackSuccess(
    JSON.stringify({ approved: true, comments: 'Looks good' })
  );

  const execution = await executionPromise;
  expect(execution.getStatus()).toBe('SUCCEEDED');

  // ✅ Parse JSON string result
  const result: any = execution.getResult();
  const approval = typeof result.approval === 'string'
    ? JSON.parse(result.approval)
    : result.approval;

  expect(approval.approved).toBe(true);
});

it('should handle callback failure', async () => {
  const runner = new LocalDurableTestRunner({ handlerFunction: handler });
  const executionPromise = runner.run({ payload: {} });

  await new Promise(resolve => setTimeout(resolve, 100));

  const callbackOp = runner.getOperation('wait-for-approval');
  await callbackOp.sendCallbackFailure('ApprovalDenied', 'Request was rejected');

  const execution = await executionPromise;
  expect(execution.getStatus()).toBe('FAILED');
});
```

## Testing Error Scenarios

```typescript
it('should retry on failure', async () => {
  let attemptCount = 0;
  
  const testHandler = withDurableExecution(async (event, context: DurableContext) => {
    return await context.step('flaky-operation', async () => {
      attemptCount++;
      if (attemptCount < 3) throw new Error('Temporary failure');
      return { success: true };
    });
  });

  const runner = new LocalDurableTestRunner({ handlerFunction: testHandler });
  const execution = await runner.run({ payload: {} });

  expect(execution.getStatus()).toBe('SUCCEEDED');
  expect(attemptCount).toBe(3);
});

it('should fail after max retries', async () => {
  const testHandler = withDurableExecution(async (event, context: DurableContext) => {
    return await context.step(
      'always-fails',
      async () => { throw new Error('Permanent failure'); },
      { retryStrategy: createRetryStrategy({ maxAttempts: 3 }) }
    );
  });

  const runner = new LocalDurableTestRunner({ handlerFunction: testHandler });
  const execution = await runner.run({ payload: {} });

  expect(execution.getStatus()).toBe('FAILED');
  expect(execution.getError()?.message).toContain('Permanent failure');
});
```

## Testing Concurrent Operations

```typescript
it('should process items concurrently', async () => {
  const runner = new LocalDurableTestRunner({ handlerFunction: handler });
  
  const execution = await runner.run({ 
    payload: { items: [1, 2, 3, 4, 5] } 
  });

  expect(execution.getStatus()).toBe('SUCCEEDED');

  const mapOp = runner.getOperation('process-items');
  expect(mapOp.getType()).toBe(OperationType.MAP);

  const item0 = runner.getOperation('process-0');
  expect(item0.getStatus()).toBe(OperationStatus.SUCCEEDED);
});
```

## Cloud Testing

```typescript
import { CloudDurableTestRunner } from '@aws/durable-execution-sdk-js-testing';

describe('Integration Tests', () => {
  it('should execute in real Lambda', async () => {
    const runner = new CloudDurableTestRunner({
      functionName: 'my-durable-function:1',  // Qualified ARN required
      client: new LambdaClient({ region: 'us-east-1' })
    });

    const execution = await runner.run({
      payload: { userId: '123' },
      config: { pollInterval: 1000 }
    });

    expect(execution.getStatus()).toBe('SUCCEEDED');
  });
});
```

## Common Testing Errors

| Error | Cause | Solution |
|-------|-------|---------|
| `'result' is of type 'unknown'` | Missing type casting | Cast: `as any` or specific type |
| `'payload' does not exist in type` | Wrong API usage | Wrap event in `payload: {}` |
| `Cannot find operation at index` | Using index for unstable ops | Use `getOperation("name")` |
| Flaky callback tests | Race condition | Use `waitForData(STARTED)` |
| `Unexpected token` in callback result | Forgot to stringify | Always `JSON.stringify(data)` |
| Callback result parsing error | Result is JSON string | Parse: `JSON.parse(result.value)` |
| Operation not found by name | Missing name | Always name operations |

## Jest Configuration

```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/*.test.ts'],
  transform: {
    '^.+\\.ts$': 'ts-jest',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
  ],
};
```

Use `skipTime: true` in test setup for fast execution.
