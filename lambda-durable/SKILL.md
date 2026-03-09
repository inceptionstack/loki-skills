---
name: lambda-durable
description: Build resilient multi-step applications and AI workflows with AWS Lambda durable functions. Automatic state persistence, retry logic, and workflow orchestration for long-running executions (up to 1 year). Use when building durable Lambda functions, step-based Lambda orchestration with checkpointing, saga pattern / compensating transactions, human-in-the-loop approvals, GenAI agentic loops with durable state, callback-based workflows, or polling patterns. Supports TypeScript/JavaScript and Python. NOT for Step Functions, SQS-based workflows, or non-Lambda orchestration.
---

# Lambda Durable Functions

Build resilient multi-step applications and AI workflows with **AWS Lambda Durable Functions** — automatic state persistence, retry logic, workflow orchestration for executions up to 1 year.

> Adapted from the [Kiro Power for AWS Lambda Durable Functions](https://github.com/aws/aws-durable-execution-docs/tree/main/aws-lambda-durable-functions-power) (Apache-2.0).

---

## Prerequisites

Before building a durable function, verify the user has:

1. **AWS CLI** ≥ 2.33.22 and configured (`aws sts get-caller-identity`)
2. **Runtime:**
   - TypeScript/JavaScript: Node.js 22+
   - Python: 3.11+ (Lambda runtimes 3.13+ have SDK pre-installed; for 3.11-3.12 use OCI/container image)
3. **IaC tool** (one of): SAM CLI ≥ 1.153.1, CDK ≥ v2.237.1, or direct Lambda deployment
4. **Ask the user** which language (TS/JS/Python) and which IaC framework (SAM/CDK/CloudFormation) if not clear

## SDK Installation

**TypeScript/JavaScript:**
```bash
npm install @aws/durable-execution-sdk-js
npm install --save-dev @aws/durable-execution-sdk-js-testing
npm install --save-dev @aws/durable-execution-sdk-js-eslint-plugin
```

**Python:**
```bash
pip install aws-durable-execution-sdk-python
pip install aws-durable-execution-sdk-python-testing
```

## IAM Requirements (CRITICAL)

The Lambda execution role **MUST** have `AWSLambdaBasicDurableExecutionRolePolicy` attached. This includes:
- `lambda:CheckpointDurableExecutions` — persist execution state
- `lambda:GetDurableExecutionState` — retrieve execution state
- CloudWatch Logs permissions

Additional permissions needed for:
- **Durable invokes:** `lambda:InvokeFunction` on target function ARNs
- **External callbacks:** `lambda:SendDurableExecutionCallbackSuccess`, `lambda:SendDurableExecutionCallbackHeartbeat`, `lambda:SendDurableExecutionCallbackFailure`

## Invocation Requirements (CRITICAL)

Durable functions require **qualified ARNs** (version, alias, or `$LATEST`):
```bash
# ✅ Valid
aws lambda invoke --function-name my-function:1 output.json
aws lambda invoke --function-name my-function:prod output.json
aws lambda invoke --function-name 'my-function:$LATEST' output.json

# ❌ Invalid — will fail
aws lambda invoke --function-name my-function output.json
```

---

## Quick Start

### Minimal Handler

**TypeScript:**
```typescript
import { withDurableExecution, DurableContext } from '@aws/durable-execution-sdk-js';

export const handler = withDurableExecution(async (event, context: DurableContext) => {
  const result = await context.step('process', async () => processData(event));
  return result;
});
```

**Python:**
```python
from aws_durable_execution_sdk_python import durable_execution, DurableContext

@durable_execution
def handler(event: dict, context: DurableContext) -> dict:
    result = context.step(lambda _: process_data(event), name='process')
    return result
```

---

## Reference Docs (load on demand)

Route the user's question to the appropriate reference file. Read the file before answering.

| User is asking about... | Load this file |
|---|---|
| Getting started, basic setup, project structure, ESLint, common patterns | `refs/getting-started.md` |
| **Replay model, determinism, non-deterministic bugs** (CRITICAL — read this for ANY durable function work) | `refs/replay-model-rules.md` |
| Steps, atomic operations, retry config, step semantics, custom serialization | `refs/step-operations.md` |
| Waits, delays, callbacks, external systems, polling, waitForCondition | `refs/wait-operations.md` |
| Parallel execution, map operations, batch processing, concurrency, completion policies | `refs/concurrent-operations.md` |
| Error handling, retry strategies, saga pattern, compensating transactions, circuit breaker | `refs/error-handling.md` |
| Testing, LocalDurableTestRunner, CloudDurableTestRunner, test patterns | `refs/testing-patterns.md` |
| Deployment, CloudFormation, CDK, SAM, infrastructure, invocation | `refs/deployment-iac.md` |
| Advanced patterns, GenAI agents, step semantics deep dive, nested workflows, custom serialization | `refs/advanced-patterns.md` |

**Rule:** When implementing ANY durable function, ALWAYS read `refs/replay-model-rules.md` first — replay model violations cause subtle, hard-to-debug issues.

---

## The 4 Critical Replay Rules (Always Apply)

These rules are non-negotiable. Violating them causes silent bugs that only appear during replay:

### Rule 1: Deterministic Code Outside Steps
ALL code outside steps MUST produce the same result on every replay.

```typescript
// ❌ WRONG — changes on each replay
const id = uuid.v4();
const timestamp = Date.now();

// ✅ CORRECT — wrapped in steps
const id = await context.step('generate-id', async () => uuid.v4());
const timestamp = await context.step('get-time', async () => Date.now());
```

**Must be in steps:** `Date.now()`, `Math.random()`, UUID generation, API calls, DB queries, file I/O, environment variable reads (if they can change).

### Rule 2: No Nested Durable Operations
Cannot call `context.step()`, `context.wait()`, or `context.invoke()` inside a step function. Use `context.runInChildContext()` instead.

### Rule 3: Closure Mutations Are Lost
Variables mutated inside steps are NOT preserved. Return values from steps instead.

### Rule 4: Side Effects Outside Steps Repeat
Side effects outside steps execute on EVERY replay. Use `context.logger` (replay-aware) for logging. Put all other side effects in steps.

---

## Common Patterns Summary

| Pattern | Key Concept | Reference |
|---|---|---|
| Multi-step workflow | Sequential `context.step()` calls | `refs/getting-started.md` |
| GenAI agentic loop | `while(true)` with model invoke + tool execution steps | `refs/advanced-patterns.md` |
| Human-in-the-loop | `context.waitForCallback()` with email/webhook | `refs/wait-operations.md` |
| Saga / compensating transactions | Try/catch with reversed compensation steps | `refs/error-handling.md` |
| Batch processing | `context.map()` with concurrency control | `refs/concurrent-operations.md` |
| Async job polling | `context.waitForCondition()` with backoff | `refs/wait-operations.md` |
| Parallel fan-out | `context.parallel()` for heterogeneous operations | `refs/concurrent-operations.md` |
| Nested workflows | `context.invoke()` to call child durable functions | `refs/advanced-patterns.md` |

---

## Project Structure Template

```
my-durable-function/
├── src/
│   ├── handler.ts              # Main handler with withDurableExecution
│   ├── steps/                  # Step functions
│   │   ├── validate.ts
│   │   └── process.ts
│   └── utils/
│       └── retry-strategies.ts
├── tests/
│   └── handler.test.ts         # Tests with LocalDurableTestRunner
├── infrastructure/
│   └── template.yaml           # SAM/CDK/CloudFormation
├── eslint.config.js            # With durable execution plugin
├── jest.config.js              # ts-jest preset
├── tsconfig.json
└── package.json
```

---

## Testing Checklist

When implementing or reviewing tests for durable functions:

- [ ] All operations have descriptive **names**
- [ ] Tests get operations by **NAME** (`runner.getOperation('name')`), never by index
- [ ] Replay behavior is tested with multiple invocations
- [ ] Use `LocalDurableTestRunner` with `skipTime: true` for fast tests
- [ ] Wrap event data in `payload` object: `runner.run({ payload: { ... } })`
- [ ] Cast `getResult()` to appropriate type
- [ ] Callbacks use `waitForData(WaitingOperationStatus.STARTED)` before sending
- [ ] Callback data is `JSON.stringify()`'d before sending
- [ ] Callback results are `JSON.parse()`'d when reading

---

## Deployment Checklist

- [ ] `DurableConfig` property set on Lambda function
- [ ] `AWSLambdaBasicDurableExecutionRolePolicy` attached to execution role
- [ ] Version or alias created (required for invocation)
- [ ] `ExecutionTimeout` set appropriately (seconds, up to 1 year)
- [ ] `RetentionPeriodInDays` configured (how long to keep state)
- [ ] CloudWatch log group with retention policy
- [ ] Alarms configured for errors and throttles
- [ ] Using qualified ARN in all invocations

---

## Links

- [AWS Lambda Durable Functions Docs](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
- [JavaScript SDK](https://github.com/aws/aws-durable-execution-sdk-js)
- [Python SDK](https://github.com/aws/aws-durable-execution-sdk-python)
- [IAM Policy Reference](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSLambdaBasicDurableExecutionRolePolicy.html)
- [Original Kiro Power](https://github.com/aws/aws-durable-execution-docs/tree/main/aws-lambda-durable-functions-power)
