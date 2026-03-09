# Deployment with Infrastructure as Code

Deploy durable functions using CloudFormation, CDK, or SAM.

## Requirements

All durable functions require:

1. **DurableConfig** property on the function
2. **AWSLambdaBasicDurableExecutionRolePolicy** attached to execution role
3. **Qualified ARN** (version or alias) for invocation

## AWS CloudFormation

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  DurableFunctionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicDurableExecutionRolePolicy

  DurableFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: myDurableFunction
      Runtime: nodejs24.x  # or python3.14
      Handler: index.handler
      Role: !GetAtt DurableFunctionRole.Arn
      Code:
        ZipFile: |
          // Your durable function code
      DurableConfig:
        ExecutionTimeout: 3600        # Max execution time (seconds)
        RetentionPeriodInDays: 7      # How long to keep execution state
      Environment:
        Variables:
          LOG_LEVEL: INFO

  DurableFunctionVersion:
    Type: AWS::Lambda::Version
    Properties:
      FunctionName: !Ref DurableFunction

  DurableFunctionAlias:
    Type: AWS::Lambda::Alias
    Properties:
      FunctionName: !Ref DurableFunction
      FunctionVersion: !GetAtt DurableFunctionVersion.Version
      Name: prod

Outputs:
  FunctionArn:
    Value: !GetAtt DurableFunction.Arn
  AliasArn:
    Value: !Ref DurableFunctionAlias
```

**Deploy:**
```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-durable-function \
  --capabilities CAPABILITY_IAM
```

## AWS CDK

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';

export class DurableFunctionStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const durableFunction = new lambda.Function(this, 'DurableFunction', {
      runtime: lambda.Runtime.NODEJS_24_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda'),
      durableConfig: {
        executionTimeout: cdk.Duration.hours(1),
        retentionPeriod: cdk.Duration.days(7)
      },
      environment: {
        LOG_LEVEL: 'INFO'
      }
    });

    // CDK automatically adds checkpoint permissions when durableConfig is set

    const version = durableFunction.currentVersion;
    const alias = new lambda.Alias(this, 'ProdAlias', {
      aliasName: 'prod',
      version: version
    });

    new cdk.CfnOutput(this, 'FunctionAliasArn', {
      value: alias.functionArn
    });
  }
}
```

### CDK Custom Log Group Management

```typescript
import * as logs from 'aws-cdk-lib/aws-logs';

const functionLogGroup = new logs.LogGroup(this, 'DurableFunctionLogGroup', {
  logGroupName: '/aws/lambda/myDurableFunction',
  retention: logs.RetentionDays.ONE_WEEK,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

const durableFunction = new lambda.Function(this, 'DurableFunction', {
  runtime: lambda.Runtime.NODEJS_24_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  logGroup: functionLogGroup,
  durableConfig: {
    executionTimeout: cdk.Duration.hours(1),
    retentionPeriod: cdk.Duration.days(7)
  }
});

// Required with explicit log groups
durableFunction.role?.addManagedPolicy(
  iam.ManagedPolicy.fromAwsManagedPolicyName(
    'service-role/AWSLambdaBasicDurableExecutionRolePolicy'
  )
);
```

## AWS SAM

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 900
    MemorySize: 512

Resources:
  DurableFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: myDurableFunction
      Runtime: nodejs24.x
      Handler: index.handler
      CodeUri: ./src
      DurableConfig:
        ExecutionTimeout: 3600
        RetentionPeriodInDays: 7
      Policies:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicDurableExecutionRolePolicy
      AutoPublishAlias: prod
      Environment:
        Variables:
          LOG_LEVEL: INFO
```

**Deploy:**
```bash
sam build
sam deploy --guided
```

## Durable Invokes (Cross-Function)

For functions that invoke other durable functions, add invoke permissions:

**CloudFormation:**
```yaml
Policies:
  - PolicyName: InvokeOtherFunctions
    PolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Action: lambda:InvokeFunction
          Resource:
            - !GetAtt TargetFunction.Arn
            - !Sub '${TargetFunction.Arn}:*'
```

**CDK:**
```typescript
targetFunction.grantInvoke(orchestratorFunction);
```

## External Callbacks IAM

For external systems to send callbacks:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:SendDurableExecutionCallbackSuccess",
        "lambda:SendDurableExecutionCallbackFailure",
        "lambda:SendDurableExecutionCallbackHeartbeat"
      ],
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:myDurableFunction:*"
    }
  ]
}
```

## Invocation Examples

**⚠️ Critical:** Always use qualified function name (version, alias, or `$LATEST`).

### Synchronous (RequestResponse)
```bash
aws lambda invoke \
  --function-name 'myDurableFunction:$LATEST' \
  --invocation-type RequestResponse \
  --durable-execution-name "execution-123" \
  --payload '{"userId":"12345"}' \
  --cli-binary-format raw-in-base64-out \
  response.json
```

### Asynchronous (Event)
```bash
aws lambda invoke \
  --function-name 'myDurableFunction:$LATEST' \
  --invocation-type Event \
  --durable-execution-name "background-task-456" \
  --payload '{"orderId":"ORD-789"}' \
  --cli-binary-format raw-in-base64-out \
  response.json
```

### Idempotency with durable-execution-name
The `--durable-execution-name` parameter ensures the same execution is never created twice. Second invocation with same name returns existing execution result.

## Environment Configuration

**Development:**
```yaml
DurableConfig:
  ExecutionTimeout: 900          # 15 minutes
  RetentionPeriodInDays: 1
```

**Production:**
```yaml
DurableConfig:
  ExecutionTimeout: 86400        # 24 hours
  RetentionPeriodInDays: 30
```

## Troubleshooting

| Issue | Solution |
|-------|---------|
| Function doesn't checkpoint | Verify `DurableConfig` is set and role has policy |
| `Unqualified ARN` error | Use version, alias, or `$LATEST` |
| `CheckpointDurableExecutions` denied | Add `AWSLambdaBasicDurableExecutionRolePolicy` |
