# 🏗️ Serverless Order Processing Pipeline on AWS

> **Production-grade, event-driven microservices architecture** — decoupled, auto-scaling, zero server management.  
> Built with AWS Lambda · API Gateway · DynamoDB · SQS · SNS · CloudWatch


---

## 🏛️ Architecture Overview

```
┌──────────────┐     POST /order     ┌─────────────────┐     Enqueue      ┌──────────────┐
│   API Client │ ──────────────────► │   API Gateway    │ ───────────────► │  SQS Queue   │
│  (REST/HTTP) │                     │  (REST API)      │                  │  (Standard)  │
└──────────────┘                     └─────────────────┘                  └──────┬───────┘
                                                                                  │ Trigger
                                                                                  ▼
                                                                    ┌─────────────────────┐
                                                                    │   Lambda: Processor  │
                                                                    │  (order-processor)   │
                                                                    └──────────┬──────────┘
                                                                               │ PutItem
                                                                               ▼
                                                                    ┌─────────────────────┐
                                                                    │   DynamoDB: orders   │
                                                                    │   (+ Streams ON)     │
                                                                    └──────────┬──────────┘
                                                                               │ Stream Event
                                                                               ▼
                                                                    ┌─────────────────────┐
                                                                    │  Lambda: Dispatcher  │
                                                                    │  (stream-consumer)   │
                                                                    └──────────┬──────────┘
                                                                               │ Publish
                                                                               ▼
                                                                    ┌─────────────────────┐
                                                                    │     SNS Topic        │
                                                                    │   (order-events)     │
                                                                    └──────┬──────┬────────┘
                                                              ┌───────────┘      └───────────┐
                                                              ▼                              ▼
                                                   ┌──────────────────┐         ┌──────────────────┐
                                                   │  Fulfillment Svc │         │  Accounting Svc  │
                                                   │  (HTTPS endpoint)│         │  (SQS subscriber)│
                                                   └──────────────────┘         └──────────────────┘
```

---

## 🤔 Why Serverless?

| Requirement | EC2 | ECS + Fargate | **Lambda (chosen)** |
|---|---|---|---|
| Auto-scaling | Manual ASG config | Managed | ✅ Native, per-request |
| Operational overhead | High | Medium | ✅ Minimal |
| Idle cost | 💸 Always running | 💸 Always running | ✅ Pay-per-invocation |
| Deployment complexity | High | Medium (containers) | ✅ Zip/SAM deploy |
| Team skill requirement | Linux/Ops | Docker/K8s | ✅ Python/Node only |
| DB connection pooling problem | Manual | Manual | ✅ No persistent connections |

> **Why not ECS/Fargate?** Technically viable, but requires container expertise the team doesn't currently have, and introduces Docker build pipelines, ECR registries, and task definition management — all unnecessary complexity for this workload.

> **Why not Aurora Serverless?** Also viable, but Lambda's fan-out creates N concurrent DB connections during traffic spikes. Aurora Serverless would require RDS Proxy as an additional component. DynamoDB handles connectionless access natively.

---

## 🔧 Tech Stack Decision Log

### Compute → AWS Lambda
- Stateless, event-driven workload — perfect fit
- Execution time well under the 15-minute cap
- No container skill requirement
- Scales from 0 to thousands of concurrent executions automatically

### API Layer → Amazon API Gateway (REST)
- Handles auth, throttling, CORS, request validation without custom code
- Direct SQS service integration — no Lambda needed in the hot path for ingestion
- Decouples HTTP concerns from business logic

### Queue → Amazon SQS (Standard)
- **Storage-first pattern**: order is persisted to the queue before any processing begins
- Protects against Lambda cold-start failures or downstream unavailability
- Visibility timeout prevents duplicate processing across concurrent consumers
- Message retention up to 14 days — no order lost during outages

### Database → Amazon DynamoDB
- Access pattern is simple: CRUD by `OrderID`, list by `CustomerID` — no joins needed
- Fully serverless, storage auto-scales without capacity planning
- On-demand mode handles spiky Black Friday-style traffic without pre-provisioning
- Native DynamoDB Streams eliminates the need for change-data-capture middleware

### Messaging → Amazon SNS (Standard Topic)
- Fan-out to multiple downstream consumers (fulfillment, accounting, inventory) with a single publish call
- Adding a new consumer = one new subscription, zero code changes
- Sub-30ms delivery latency — well under EventBridge's ~500ms
- Simpler and cheaper for this use case vs. EventBridge (no schema registry or SaaS integration needed)

### Why SNS over EventBridge?

| | SNS | EventBridge |
|---|---|---|
| Latency | ~30ms | ~500ms |
| Throughput | Nearly unlimited | Limited (raiseable) |
| SaaS integration | ❌ | ✅ |
| Payload filtering | Attribute-based | Full JSON body |
| Cost | Lower | Higher |
| **Fit for this use case** | ✅ **Winner** | Overkill |

---

## 🧩 System Components

### `order-processor` Lambda
Reads from SQS, validates order payload, persists to DynamoDB.

```python
import boto3
import uuid
import json

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("orders")

def handler(event, context):
    for record in event["Records"]:
        payload = json.loads(record["body"])

        order_id = str(uuid.uuid4())
        item = {
            "OrderID": order_id,
            "CustomerID": payload["customer_id"],
            "Items": payload["items"],
            "Status": "RECEIVED",
            "CreatedAt": payload.get("timestamp"),
        }

        table.put_item(Item=item)
        print(f"[order-processor] Persisted order {order_id} for customer {payload['customer_id']}")

    return {"statusCode": 200}
```

### `stream-consumer` Lambda
Triggered by DynamoDB Streams on `INSERT` events, publishes to SNS.

```python
import boto3
import json
import os

sns = boto3.client("sns")
TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]

def handler(event, context):
    for record in event["Records"]:
        if record["eventName"] != "INSERT":
            continue

        new_image = record["dynamodb"].get("NewImage", {})

        # Deserialize DynamoDB typed attributes to plain dict
        order = {k: list(v.values())[0] for k, v in new_image.items()}

        sns.publish(
            TopicArn=TOPIC_ARN,
            Message=json.dumps(order),
            MessageStructure="string",
            MessageAttributes={
                "event_type": {
                    "DataType": "String",
                    "StringValue": "ORDER_CREATED",
                }
            },
        )
        print(f"[stream-consumer] Published ORDER_CREATED event for {order.get('OrderID')}")
```

---

## 📊 Data Flow

```
1. Client sends POST /order  →  { "customer_id": "C-001", "items": [...] }
2. API Gateway validates schema, forwards to SQS via service integration
3. SQS enqueues the message (MessageRetentionPeriod: 14 days)
4. order-processor Lambda polls SQS (long-poll, 20s wait)
5. Lambda deserializes, assigns UUID OrderID, writes to DynamoDB orders table
6. DynamoDB Streams captures the INSERT event (NewImage)
7. stream-consumer Lambda reads from stream, extracts new order
8. Lambda publishes to SNS topic with event_type=ORDER_CREATED attribute
9. All subscribers (Fulfillment, Accounting, Inventory) receive the message concurrently
```

### DynamoDB Table Schema

| Attribute | Type | Notes |
|---|---|---|
| `OrderID` | String (PK) | UUID v4, generated at ingest |
| `CustomerID` | String (GSI PK) | Enables `query by customer` |
| `Items` | List | Array of `{ sku, qty, price }` |
| `Status` | String | `RECEIVED` → `PROCESSING` → `FULFILLED` |
| `CreatedAt` | String | ISO 8601 timestamp |

**GSI:** `CustomerIndex` — Partition key: `CustomerID`, for `query orders by customer` access pattern.

---

## 🔐 IAM Roles & Least-Privilege Policy Design

All roles follow **least-privilege**. No wildcard `*` actions. No wildcard resources.

### `order-processor` execution role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadFromSQS",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "arn:aws:sqs:us-east-1:ACCOUNT_ID:order-queue"
    },
    {
      "Sid": "WriteToDynamoDB",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID:table/orders"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:ACCOUNT_ID:*"
    }
  ]
}
```

### `stream-consumer` execution role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadDynamoDBStreams",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetShardIterator",
        "dynamodb:DescribeStream",
        "dynamodb:ListStreams",
        "dynamodb:GetRecords"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID:table/orders/stream/*"
    },
    {
      "Sid": "PublishToSNS",
      "Effect": "Allow",
      "Action": [
        "sns:Publish",
        "sns:GetTopicAttributes",
        "sns:ListTopics"
      ],
      "Resource": "arn:aws:sns:us-east-1:ACCOUNT_ID:order-events"
    }
  ]
}
```

### `apigw-sqs` execution role
Attached to API Gateway to allow it to call `sqs:SendMessage` on the order queue directly.

---

## 🏗️ Infrastructure as Code

All resources are defined in AWS SAM (`template.yaml`). No point-and-click.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    Environment:
      Variables:
        LOG_LEVEL: INFO

Resources:

  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: order-queue
      VisibilityTimeout: 60
      MessageRetentionPeriod: 1209600  # 14 days
      ReceiveMessageWaitTimeSeconds: 20  # long polling
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderDLQ.Arn
        maxReceiveCount: 3

  OrderDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: order-queue-dlq
      MessageRetentionPeriod: 1209600

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: orders
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: OrderID
          AttributeType: S
        - AttributeName: CustomerID
          AttributeType: S
      KeySchema:
        - AttributeName: OrderID
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: CustomerIndex
          KeySchema:
            - AttributeName: CustomerID
              KeyType: HASH
          Projection:
            ProjectionType: ALL
      StreamSpecification:
        StreamViewType: NEW_IMAGE

  OrderEventsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: order-events

  OrderProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: order-processor
      CodeUri: src/order_processor/
      Handler: handler.handler
      Events:
        SQSTrigger:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10
            FunctionResponseTypes:
              - ReportBatchItemFailures
      Policies:
        - SQSPollerPolicy:
            QueueName: !GetAtt OrderQueue.QueueName
        - DynamoDBWritePolicy:
            TableName: !Ref OrdersTable

  StreamConsumerFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: stream-consumer
      CodeUri: src/stream_consumer/
      Handler: handler.handler
      Environment:
        Variables:
          SNS_TOPIC_ARN: !Ref OrderEventsTopic
      Events:
        DynamoDBTrigger:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt OrdersTable.StreamArn
            StartingPosition: TRIM_HORIZON
            BisectBatchOnFunctionError: true
            MaximumRetryAttempts: 2
      Policies:
        - DynamoDBStreamReadPolicy:
            TableName: !Ref OrdersTable
            StreamName: !Select [3, !Split ["/", !GetAtt OrdersTable.StreamArn]]
        - SNSPublishMessagePolicy:
            TopicName: !GetAtt OrderEventsTopic.TopicName
```

---

## 🧪 Local Development & Testing

### Prerequisites

```bash
pip install aws-sam-cli boto3 pytest moto
```

### Unit Tests (mocked AWS with `moto`)

```python
# tests/test_order_processor.py
import json
import pytest
import boto3
from moto import mock_dynamodb
from src.order_processor.handler import handler

@mock_dynamodb
def test_order_is_persisted():
    # Arrange
    dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
    table = dynamodb.create_table(
        TableName="orders",
        KeySchema=[{"AttributeName": "OrderID", "KeyType": "HASH"}],
        AttributeDefinitions=[{"AttributeName": "OrderID", "AttributeType": "S"}],
        BillingMode="PAY_PER_REQUEST",
    )

    event = {
        "Records": [{
            "body": json.dumps({
                "customer_id": "C-001",
                "items": [{"sku": "GLOVE-L", "qty": 2, "price": 9.99}],
                "timestamp": "2025-01-01T10:00:00Z"
            })
        }]
    }

    # Act
    response = handler(event, {})

    # Assert
    items = table.scan()["Items"]
    assert len(items) == 1
    assert items[0]["CustomerID"] == "C-001"
    assert items[0]["Status"] == "RECEIVED"
    assert response["statusCode"] == 200
```

### Local Integration with SAM

```bash
# Start local Lambda + API Gateway
sam local start-api --env-vars env.json

# Invoke a single function locally
sam local invoke OrderProcessorFunction --event events/sqs_event.json

# Run full integration test
curl -X POST http://localhost:3000/order \
  -H "Content-Type: application/json" \
  -d '{"customer_id": "C-001", "items": [{"sku": "GLOVE-L", "qty": 2}]}'
```

`events/sqs_event.json` sample:

```json
{
  "Records": [
    {
      "messageId": "abc123",
      "body": "{\"customer_id\": \"C-001\", \"items\": [{\"sku\": \"GLOVE-L\", \"qty\": 2, \"price\": 9.99}], \"timestamp\": \"2025-01-01T10:00:00Z\"}",
      "attributes": {},
      "messageAttributes": {},
      "md5OfBody": "",
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:us-east-1:000000000000:order-queue",
      "awsRegion": "us-east-1"
    }
  ]
}
```

---

## 🚀 Deployment

```bash
# Build
sam build

# Deploy (first time — guided)
sam deploy --guided

# Deploy (subsequent)
sam deploy --config-env prod

# Rollback if needed
aws lambda update-function-code \
  --function-name order-processor \
  --s3-bucket your-deployment-bucket \
  --s3-key previous-version.zip
```

### CI/CD Pipeline (GitHub Actions snippet)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/setup-sam@v2
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - run: sam build
      - run: pytest tests/ -v
      - run: sam deploy --no-confirm-changeset --config-env prod
```

---

## 📈 Observability

### CloudWatch Dashboard — Key Metrics

| Metric | Source | Alert Threshold |
|---|---|---|
| `Lambda Errors` | order-processor | > 5 in 5 min |
| `Lambda Duration p99` | order-processor | > 5000ms |
| `SQS ApproximateNumberOfMessagesNotVisible` | order-queue | > 500 |
| `SQS NumberOfMessagesSent` to DLQ | order-dlq | > 0 |
| `DynamoDB ConsumedWriteCapacityUnits` | orders | > 80% of provisioned |
| `SNS NumberOfNotificationsFailed` | order-events | > 0 |

### Structured Logging

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def log(level, message, **kwargs):
    logger.log(level, json.dumps({"message": message, **kwargs}))

# Usage
log(logging.INFO, "Order persisted", order_id=order_id, customer_id=payload["customer_id"])
```

### Dead Letter Queue Alarm

```yaml
OrderDLQAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: order-dlq-messages-visible
    AlarmDescription: "Messages landed in DLQ — processing failure requires investigation"
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Dimensions:
      - Name: QueueName
        Value: order-queue-dlq
    Statistic: Sum
    Period: 60
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    AlarmActions:
      - !Ref OpsTeamSNSTopic
```

---

## 💰 Cost Considerations

| Service | Pricing Model | Estimated (1M orders/month) |
|---|---|---|
| API Gateway | $3.50 per 1M REST API calls | ~$3.50 |
| SQS | $0.40 per 1M requests | ~$2.00 |
| Lambda | $0.20 per 1M invocations + compute | ~$0.50 |
| DynamoDB | On-demand: $1.25 per 1M WCU | ~$1.25 |
| DynamoDB Streams | $0.02 per 100K read requests | ~$0.20 |
| SNS | $0.50 per 1M publishes | ~$0.50 |
| **Total** | | **~$8/month** |

> 💡 Compare to a single `t3.medium` EC2 instance running 24/7: ~$30/month, before RDS, load balancer, or NAT Gateway costs. And that's for a **single point of failure** with zero auto-scaling.

---

## ⚖️ Known Trade-offs

| Decision | Trade-off |
|---|---|
| **SQS Standard (not FIFO)** | Message ordering not guaranteed. Acceptable here — orders are independent. Use FIFO only if processing order matters. |
| **DynamoDB over Aurora** | No complex SQL joins. If reporting requirements emerge (cross-order analytics), consider adding a separate read model or streaming to Redshift. |
| **SNS over EventBridge** | No built-in schema enforcement. Consider EventBridge + schema registry if the number of event types grows significantly or SaaS integrations are added. |
| **Lambda max 15 min timeout** | Long-running fulfillment processes must be offloaded to Step Functions or ECS if they exceed this limit. |
| **Eventual consistency** | The async pattern means the caller gets a 200 before downstream services process the order. This is by design — communicate clearly in the API contract. |

---

## 🤝 Contributing

```bash
# Clone
git clone https://github.com/your-org/order-pipeline.git
cd order-pipeline

# Install dev dependencies
pip install -r requirements-dev.txt

# Pre-commit hooks (linting, type checks)
pre-commit install

# Run tests
pytest tests/ -v --cov=src --cov-report=term-missing

# Validate SAM template
sam validate --lint
```

> All PRs must pass unit tests, have > 80% coverage on new code, and include an updated `CHANGELOG.md` entry.

---

<details>
<summary>📚 Reference Links</summary>

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [DynamoDB Streams & Lambda](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.html)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SNS Developer Guide](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [SAM CLI Reference](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-command-reference.html)
- [AWS Well-Architected — Serverless Lens](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/welcome.html)

</details>
