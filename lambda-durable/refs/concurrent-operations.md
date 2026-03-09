# Concurrent Operations

Process arrays and run operations in parallel with concurrency control.

## Map Operations

**TypeScript:**
```typescript
const items = [1, 2, 3, 4, 5];

const results = await context.map(
  'process-items',
  items,
  async (ctx, item, index) => {
    return await ctx.step(`process-${index}`, async () => 
      processItem(item)
    );
  },
  {
    maxConcurrency: 3,
    completionConfig: {
      minSuccessful: 4,
      toleratedFailureCount: 1
    }
  }
);

results.throwIfError();
const allResults = results.getResults();
```

**Python:**
```python
from aws_durable_execution_sdk_python.concurrency import MapConfig, CompletionConfig

items = [1, 2, 3, 4, 5]

def process_item(ctx: DurableContext, item: int, index: int):
    return ctx.step(process(item), name=f'process-{index}')

results = context.map(
    inputs=items,
    func=process_item,
    name='process-items',
    config=MapConfig(
        max_concurrency=3,
        completion_config=CompletionConfig(
            min_successful=4,
            tolerated_failure_count=1
        )
    )
)

results.throw_if_error()
all_results = results.get_results()
```

## Parallel Operations

Run heterogeneous operations concurrently:

**TypeScript:**
```typescript
const results = await context.parallel(
  'parallel-ops',
  [
    {
      name: 'fetch-user',
      func: async (ctx) => ctx.step(async () => fetchUser(userId))
    },
    {
      name: 'fetch-orders',
      func: async (ctx) => ctx.step(async () => fetchOrders(userId))
    },
    {
      name: 'fetch-preferences',
      func: async (ctx) => ctx.step(async () => fetchPreferences(userId))
    }
  ],
  { maxConcurrency: 3 }
);

const [user, orders, preferences] = results.getResults();
```

## Completion Policies

### Minimum Successful
```typescript
const results = await context.map('process-batch', items, processFunc, {
  completionConfig: { minSuccessful: 8 }
});
```

### Tolerated Failures
```typescript
const results = await context.map('process-batch', items, processFunc, {
  completionConfig: { toleratedFailureCount: 2 }
});
```

### Tolerated Failure Percentage
```typescript
const results = await context.map('process-batch', items, processFunc, {
  completionConfig: { toleratedFailurePercentage: 10 }
});
```

## Batch Result Handling

```typescript
const results = await context.map('process', items, processFunc);

console.log(results.status);           // 'COMPLETED' | 'FAILED'
console.log(results.totalCount);
console.log(results.successCount);
console.log(results.failureCount);
console.log(results.hasFailure());

// Get successful results only
const successful = results.succeeded.map(item => item.result);

// Get failed items
const failed = results.failed.map(item => ({
  index: item.index,
  error: item.error
}));

// Retry failed items
if (results.hasFailure()) {
  const failedItems = results.failed.map(f => items[f.index]);
  await context.map('retry-failed', failedItems, processFunc);
}
```

## Advanced Patterns

### Map with Callbacks
```typescript
const results = await context.map(
  'process-with-approval',
  items,
  async (ctx, item, index) => {
    const processed = await ctx.step('process', async () => process(item));
    const approved = await ctx.waitForCallback(
      'approval',
      async (callbackId) => sendApproval(item, callbackId),
      { timeout: { hours: 24 } }
    );
    return { processed, approved };
  },
  { maxConcurrency: 3 }
);
```

### Map with Child Contexts
```typescript
const results = await context.map(
  'complex-process',
  items,
  async (ctx, item, index) => {
    return await ctx.runInChildContext(`item-${index}`, async (childCtx) => {
      const validated = await childCtx.step('validate', async () => validate(item));
      await childCtx.wait({ seconds: 1 });
      return await childCtx.step('process', async () => process(validated));
    });
  },
  { maxConcurrency: 5 }
);
```

### Early Termination
```typescript
const results = await context.map(
  'find-match',
  candidates,
  async (ctx, candidate) => {
    return await ctx.step(async () => checkMatch(candidate));
  },
  { completionConfig: { minSuccessful: 1 } }  // Stop after first success
);
```

## Best Practices

1. **Set appropriate maxConcurrency** based on downstream system capacity
2. **Use completion policies** to handle partial failures gracefully
3. **Name all operations** for debugging
4. **Handle batch results explicitly** — check for failures
5. **Consider retry strategies** for failed items
6. **Use child contexts** for complex per-item workflows
7. **Implement circuit breakers** for external service calls
