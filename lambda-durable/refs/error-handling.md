# Error Handling and Retry Strategies

Comprehensive error handling patterns for durable functions.

## Retry Strategies

### Exponential Backoff with Jitter

**TypeScript:**
```typescript
import { createRetryStrategy, JitterStrategy } from '@aws/durable-execution-sdk-js';

const result = await context.step(
  'api-call',
  async () => callAPI(),
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
from aws_durable_execution_sdk_python.retries import RetryStrategyConfig, create_retry_strategy

retry_config = RetryStrategyConfig(
    max_attempts=5,
    initial_delay_seconds=1,
    max_delay_seconds=60,
    backoff_rate=2.0,
    jitter='full'
)

result = context.step(
    api_call(),
    config=StepConfig(retry_strategy=create_retry_strategy(retry_config))
)
```

### Custom Retry Logic

**TypeScript:**
```typescript
const result = await context.step(
  'custom-retry',
  async () => riskyOperation(),
  {
    retryStrategy: (error, attemptCount) => {
      // Don't retry client errors
      if (error.statusCode >= 400 && error.statusCode < 500) {
        return { shouldRetry: false };
      }
      // Retry server errors with exponential backoff
      if (attemptCount < 5) {
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

## Saga Pattern (Compensating Transactions)

**TypeScript:**
```typescript
export const handler = withDurableExecution(async (event, context: DurableContext) => {
  const compensations: Array<{
    name: string;
    fn: () => Promise<void>;
  }> = [];

  try {
    const reservation = await context.step('reserve-inventory', async () =>
      inventoryService.reserve(event.items)
    );
    compensations.push({
      name: 'cancel-reservation',
      fn: () => inventoryService.cancelReservation(reservation.id)
    });

    const payment = await context.step('charge-payment', async () =>
      paymentService.charge(event.paymentMethod, event.amount)
    );
    compensations.push({
      name: 'refund-payment',
      fn: () => paymentService.refund(payment.id)
    });

    const shipment = await context.step('create-shipment', async () =>
      shippingService.createShipment(event.address, event.items)
    );

    return { success: true, orderId: shipment.orderId };

  } catch (error) {
    context.logger.error('Order failed, executing compensations', error);
    
    for (const comp of compensations.reverse()) {
      try {
        await context.step(comp.name, async () => comp.fn());
      } catch (compError) {
        context.logger.error(`Compensation ${comp.name} failed`, compError);
      }
    }
    
    throw error;
  }
});
```

**Python:**
```python
@durable_execution
def handler(event: dict, context: DurableContext) -> dict:
    compensations = []

    try:
        reservation = context.step(reserve_inventory(event['items']))
        compensations.append(('cancel-reservation', cancel_reservation, reservation['id']))

        payment = context.step(charge_payment(event['payment_method'], event['amount']))
        compensations.append(('refund-payment', refund_payment, payment['id']))

        shipment = context.step(create_shipment(event['address'], event['items']))
        return {'success': True, 'order_id': shipment['order_id']}

    except Exception as error:
        context.logger.error('Order failed, executing compensations', error)
        
        for name, comp_step, resource_id in reversed(compensations):
            try:
                context.step(comp_step(resource_id))
            except Exception as comp_error:
                context.logger.error(f'Compensation {name} failed', comp_error)
        
        raise error
```

## Unrecoverable Errors

Stop execution immediately — no retries:

**TypeScript:**
```typescript
import { UnrecoverableInvocationError } from '@aws/durable-execution-sdk-js';

export const handler = withDurableExecution(async (event, context: DurableContext) => {
  const user = await context.step('fetch-user', async () => {
    const user = await fetchUser(event.userId);
    if (!user) {
      throw new UnrecoverableInvocationError('User not found');
    }
    return user;
  });
});
```

**Python:**
```python
from aws_durable_execution_sdk_python.exceptions import InvocationError

@durable_execution
def handler(event: dict, context: DurableContext) -> dict:
    @durable_step
    def fetch_user_step(step_ctx: StepContext):
        user = fetch_user(event['user_id'])
        if not user:
            raise InvocationError('User not found')
        return user
    
    user = context.step(fetch_user_step())
```

## Partial Failure Handling

```typescript
const results = await context.map(
  'process-items',
  event.items,
  async (ctx, item, index) => {
    return await ctx.step(async () => processItem(item));
  },
  { completionConfig: { toleratedFailurePercentage: 10 } }
);

if (results.hasFailure()) {
  context.logger.warn('Some items failed', {
    failureCount: results.failureCount,
    failures: results.failed.map(f => f.index)
  });

  // Store failed items for later retry
  await context.step('store-failures', async () => {
    const failedItems = results.failed.map(f => event.items[f.index]);
    return await storeFailedItems(failedItems);
  });
}
```

## Best Practices

1. **Use appropriate retry strategies** — exponential backoff for most cases
2. **Classify errors correctly** — distinguish retryable from non-retryable
3. **Implement compensating transactions** for distributed workflows
4. **Make errors deterministic** — same input produces same error
5. **Use unrecoverable errors** to stop execution early when appropriate
6. **Log errors with context** using `context.logger`
7. **Handle partial failures** gracefully in batch operations
8. **Implement circuit breakers** for external service calls
9. **Test error scenarios** thoroughly with test runners
