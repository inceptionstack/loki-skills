# Advanced Patterns

Advanced techniques for sophisticated durable function workflows.

## GenAI Agent with Reasoning and Dynamic Step Naming

**TypeScript:**
```typescript
export const handler = withDurableExecution(async (event, context: DurableContext) => {
  context.logger.info('Starting AI agent', { prompt: event.prompt });
  const messages = [{ role: 'user', content: event.prompt }];

  while (true) {
    const { response, reasoning, tool } = await context.step(
      'invoke-model',
      async (stepCtx) => {
        stepCtx.logger.info('Invoking AI model', { messageCount: messages.length });
        return await invokeAIModel(messages);
      }
    );

    if (reasoning) context.logger.debug('AI reasoning', { reasoning });

    if (tool == null) {
      context.logger.info('AI agent completed');
      return response;
    }

    const toolResult = await context.step(
      `execute-tool-${tool.name}`,  // Dynamic step name
      async (stepCtx) => {
        stepCtx.logger.info('Executing tool', { toolName: tool.name });
        return await executeTool(tool, response);
      }
    );

    messages.push({ role: 'assistant', content: toolResult });
  }
});
```

**Python:**
```python
@durable_execution
def handler(event: dict, context: DurableContext) -> str:
    context.logger.info('Starting AI agent', extra={'prompt': event['prompt']})
    messages = [{'role': 'user', 'content': event['prompt']}]

    while True:
        result = context.step(invoke_ai_model(messages))

        if result.get('tool') is None:
            return result['response']

        tool = result['tool']
        tool_result = context.step(
            func=execute_tool(tool, result['response']),
            name=f"execute-tool-{tool['name']}"
        )

        messages.append({'role': 'assistant', 'content': tool_result})
```

## Step Semantics Deep Dive

### AtMostOncePerRetry vs AtLeastOncePerRetry

```typescript
import { StepSemantics } from '@aws/durable-execution-sdk-js';

// AtMostOncePerRetry (DEFAULT) — idempotent operations
await context.step(
  'update-database',
  async () => updateUserRecord(userId, data),
  { semantics: StepSemantics.AtMostOncePerRetry }
);

// AtLeastOncePerRetry — external deduplication exists
await context.step(
  'send-notification',
  async () => sendEmail(email, message),
  { semantics: StepSemantics.AtLeastOncePerRetry }
);
```

| Semantic | Use When | Examples |
|----------|----------|---------|
| **AtMostOncePerRetry** | Operation is idempotent | DB updates, API calls with idempotency keys |
| **AtLeastOncePerRetry** | External deduplication exists | Queuing systems, event streams |

## Completion Policies — Interaction and Combination

Policies can be combined. Execution **stops when the first constraint is met**:

```typescript
const results = await context.map(
  'process-items',
  items,
  processFunc,
  {
    completionConfig: {
      minSuccessful: 8,              // Need at least 8 successes
      toleratedFailureCount: 2,       // OR can tolerate 2 failures
      toleratedFailurePercentage: 20, // OR can tolerate 20% failures
    }
  }
);

// Stops when ANY condition is met:
// 1. 8 successful items (minSuccessful reached)
// 2. 2 failures occur (toleratedFailureCount reached)
// 3. 20% of items fail (toleratedFailurePercentage reached)
```

### Early Termination Pattern

```typescript
// Stop after finding first match
const results = await context.map(
  'find-match',
  candidates,
  async (ctx, candidate) => ctx.step(async () => checkMatch(candidate)),
  { completionConfig: { minSuccessful: 1 } }
);
```

## Timeout Handling with waitForCallback

```typescript
export const handler = withDurableExecution(async (event, context) => {
  try {
    const approval = await context.waitForCallback(
      'wait-for-approval',
      async (callbackId, ctx) => {
        ctx.logger.info('Sending approval request', { callbackId });
        await sendApprovalEmail(event.approverEmail, callbackId);
      },
      { timeout: { hours: 24 } }
    );

    return { status: 'approved', approval };

  } catch (error: any) {
    if (error.name === 'CallbackTimeoutError' ||
        error.message?.includes('timeout')) {

      context.logger.warn('Approval timed out after 24 hours');

      await context.step('handle-timeout', async (stepCtx) => {
        stepCtx.logger.info('Escalating to manager');
        await escalateToManager(event);
      });

      return { status: 'timeout', escalated: true };
    }

    throw error;
  }
});
```

## Custom Serialization

### Class with Date Fields

```typescript
import { createClassSerdesWithDates } from '@aws/durable-execution-sdk-js';

class User {
  constructor(
    public name: string,
    public email: string,
    public createdAt: Date,
    public updatedAt: Date
  ) {}
}

const result = await context.step(
  'create-user',
  async () => new User('Alice', 'alice@example.com', new Date(), new Date()),
  { serdes: createClassSerdesWithDates(User, ['createdAt', 'updatedAt']) }
);

console.log(result.createdAt instanceof Date); // true
```

### Complex Object Graphs

```typescript
import { createClassSerdes } from '@aws/durable-execution-sdk-js';

class Order {
  constructor(
    public id: string,
    public items: OrderItem[],
    public customer: Customer
  ) {}
}

const orderSerdes = createClassSerdes(Order);

const result = await context.step(
  'process-order',
  async () => new Order('ORD-456', items, customer),
  { serdes: orderSerdes }
);
```

## Nested Workflows (Parent-Child)

### Parent Orchestrator

```typescript
export const orchestrator = withDurableExecution(
  async (event, context: DurableContext) => {
    const childFunctionArn = process.env.CHILD_FUNCTION_ARN!;

    const results = await context.parallel(
      'process-batches',
      [
        {
          name: 'batch-1',
          func: async (ctx) => ctx.invoke(
            'process-batch-1',
            childFunctionArn,
            { batch: event.batches[0] }
          )
        },
        {
          name: 'batch-2',
          func: async (ctx) => ctx.invoke(
            'process-batch-2',
            childFunctionArn,
            { batch: event.batches[1] }
          )
        }
      ]
    );

    return results.getResults();
  }
);
```

### Child Worker

```typescript
export const worker = withDurableExecution(
  async (event, context: DurableContext) => {
    const results = await context.map(
      'process-items',
      event.batch.items,
      async (ctx, item) => ctx.step(async () => processItem(item))
    );

    return results.getResults();
  }
);
```

## Best Practices

1. **Dynamic Step Naming**: Use template literals for dynamic operation names
2. **Structured Logging**: Log reasoning and context with each operation
3. **Timeout Handling**: Always have fallback logic for callback timeouts
4. **Completion Policies**: Understand how combined constraints interact
5. **Custom Serialization**: Use proper serdes for complex objects
6. **Nested Workflows**: Use invoke for modular, composable architectures
