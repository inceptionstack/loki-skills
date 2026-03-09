# Step Operations

Steps are atomic operations with automatic retry and state persistence.

## Basic Step Patterns

### Python: Two Ways to Define Steps

**Recommended: `@durable_step` Decorator**
```python
from aws_durable_execution_sdk_python import durable_step, StepContext

@durable_step
def fetch_user(step_ctx: StepContext, user_id: str):
    """Fetch user from database - reusable step function."""
    return fetch_user_from_api(user_id)

result = context.step(fetch_user(user_id))
```

**Alternative: Inline Lambda**
```python
result = context.step(
    func=lambda step_ctx: fetch_user_from_api(user_id),
    name='fetch-user'
)
```

### TypeScript: Named Steps

```typescript
const result = await context.step('fetch-user', async () => {
  return await fetchUserFromAPI(userId);
});
```

## Retry Configuration

### Exponential Backoff

**TypeScript:**
```typescript
import { createRetryStrategy, JitterStrategy } from '@aws/durable-execution-sdk-js';

const result = await context.step(
  'api-call',
  async () => callExternalAPI(),
  {
    retryStrategy: createRetryStrategy({
      maxAttempts: 5,
      initialDelay: { seconds: 1 },
      maxDelay: { seconds: 60 },
      backoffRate: 2.0,
      jitter: JitterStrategy.FULL
    })
  }
);
```

**Python:**
```python
from aws_durable_execution_sdk_python.config import StepConfig, Duration
from aws_durable_execution_sdk_python.retries import RetryStrategyConfig, create_retry_strategy, JitterStrategy

retry_config = RetryStrategyConfig(
    max_attempts=5,
    initial_delay=Duration.from_seconds(5),
    max_delay=Duration.from_seconds(60),
    backoff_rate=2.0,
    jitter_strategy=JitterStrategy.FULL
)

result = context.step(
    func=api_call,
    config=StepConfig(retry_strategy=create_retry_strategy(retry_config))
)
```

### Custom Retry Strategy

**TypeScript:**
```typescript
const result = await context.step(
  'custom-retry',
  async () => riskyOperation(),
  {
    retryStrategy: (error, attemptCount) => {
      if (error.name === 'ValidationError') {
        return { shouldRetry: false };
      }
      if (attemptCount < 3) {
        return {
          shouldRetry: true,
          delay: { seconds: Math.pow(2, attemptCount) }
        };
      }
      return { shouldRetry: false };
    }
  }
);
```

### Retryable Error Types

```typescript
const result = await context.step(
  'selective-retry',
  async () => operation(),
  {
    retryStrategy: createRetryStrategy({
      maxAttempts: 3,
      retryableErrorTypes: ['NetworkError', 'TimeoutError']
    })
  }
);
```

## Step Semantics

### AT_LEAST_ONCE (Default)

Step executes at least once, may execute multiple times on failure/retry.

```typescript
const result = await context.step(
  'idempotent-operation',
  async () => idempotentAPI(),
  { semantics: 'AT_LEAST_ONCE' }
);
```

### AT_MOST_ONCE

Step executes at most once, never retries. Use for non-idempotent operations.

```typescript
const result = await context.step(
  'charge-payment',
  async () => chargeCard(amount),
  { semantics: 'AT_MOST_ONCE' }
);
```

## Custom Serialization

```typescript
import { createClassSerdesWithDates } from '@aws/durable-execution-sdk-js';

class User {
  constructor(
    public id: string,
    public name: string,
    public createdAt: Date
  ) {}
}

const userSerdes = createClassSerdesWithDates(User, ['createdAt']);

const user = await context.step(
  'fetch-user',
  async () => new User('123', 'Alice', new Date()),
  { serdes: userSerdes }
);
```

## When to Use Steps vs Child Contexts

### Use Steps For:
- Single atomic operations
- API calls, database queries
- Data transformations
- Operations that should retry as a unit

### Use Child Contexts For:
- Grouping multiple durable operations
- Complex workflows with steps + waits + invokes
- Isolating state tracking

```typescript
// ❌ WRONG: Cannot nest durable operations in step
await context.step('process', async () => {
  await context.wait({ seconds: 1 });  // ERROR!
});

// ✅ CORRECT: Use child context
await context.runInChildContext('process', async (childCtx) => {
  const data = await childCtx.step('fetch', async () => fetch());
  await childCtx.wait({ seconds: 1 });
  return await childCtx.step('save', async () => save(data));
});
```

## Best Practices

1. **Always name steps** for debugging and testing
2. **Keep steps atomic** — one logical operation per step
3. **Make steps idempotent** when possible
4. **Use appropriate retry strategies** based on operation type
5. **Handle errors explicitly** — don't let them propagate unexpectedly
6. **Use custom serialization** for complex types
7. **Choose correct semantics** (AT_LEAST_ONCE vs AT_MOST_ONCE)
