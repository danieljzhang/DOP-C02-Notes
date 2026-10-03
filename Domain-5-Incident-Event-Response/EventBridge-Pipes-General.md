# Amazon EventBridge Pipes - DOP-C02 Exam Notes

> **Launched:** November 2022. Pipes are a distinct EventBridge feature from rules/buses — know the difference.

## 1. Overview

**What it is:** A point-to-point integration that connects a single *source* to a single *target* with optional *filtering* and *enrichment* in between — all without writing glue code.

**What problem it solves:** Before Pipes, connecting an SQS queue to a Lambda enricher to a Step Functions workflow required custom Lambda code just to pass messages along. Pipes does that wiring declaratively, with built-in batching, filtering, and enrichment.

**EventBridge Pipes vs EventBridge Rules:**

| | Rules (event bus) | Pipes |
|---|---|---|
| **Topology** | One-to-many fan-out | Point-to-point (one source → one target) |
| **Source** | AWS services pushing events | Polling sources (SQS, Kinesis, DynamoDB Streams, etc.) |
| **Filtering** | Event pattern matching | Source filtering + input transformation |
| **Enrichment** | Not built-in | Built-in (Lambda, Step Functions, API Gateway, API Destination) |
| **Use case** | Broadcast events to multiple consumers | Transform and route a stream to one destination |

---

## 2. Pipe Anatomy

```
Source → [Filter] → [Enrichment] → Target
```

Every pipe has exactly:
- **1 source** (required)
- **1 filter** (optional) — drop unwanted records before enrichment
- **1 enrichment** (optional) — call Lambda/Step Functions/API to add data
- **1 target** (required)

---

## 3. Supported Sources

All are *polling* sources — Pipes polls them on your behalf:

- Amazon SQS queue
- Amazon Kinesis Data Stream
- Amazon DynamoDB Stream
- Amazon MQ (ActiveMQ, RabbitMQ)
- Amazon MSK (Managed Streaming for Apache Kafka)
- Self-managed Apache Kafka

---

## 4. Supported Enrichments

- AWS Lambda function
- AWS Step Functions (Express Workflow — synchronous)
- Amazon API Gateway REST API
- EventBridge API Destination (any HTTP endpoint)

The enrichment receives the (filtered) batch, adds data, and returns the enriched payload to the pipe. The target receives the enriched payload.

---

## 5. Supported Targets

A wide set including:
- Lambda, Step Functions (Standard or Express)
- SQS, SNS, Kinesis, Firehose
- EventBridge event bus (fan-out after enrichment)
- API Gateway, API Destination
- ECS task, Batch job, SageMaker Pipeline
- CloudWatch Logs, Redshift

---

## 6. Filtering

Filtering happens *before* enrichment — records that don't match are dropped, saving enrichment invocations and cost.

```json
{
  "Filters": [
    {
      "Pattern": {
        "body": {
          "eventType": ["ORDER_PLACED"],
          "amount": [{ "numeric": [">", 100] }]
        }
      }
    }
  ]
}
```

Only records matching the pattern proceed to enrichment and target.

---

## 7. Example: DynamoDB Stream → Enrich → EventBridge Bus

```
DynamoDB Stream (source)
  → Filter: only INSERT events on Orders table
  → Enrichment: Lambda adds customer details from DynamoDB
  → Target: EventBridge custom bus (fan-out to multiple consumers)
```

```bash
aws pipes create-pipe \
  --name order-enrichment-pipe \
  --source arn:aws:dynamodb:us-east-1:123456789012:table/Orders/stream/... \
  --source-parameters '{
    "DynamoDBStreamParameters": {
      "StartingPosition": "LATEST",
      "BatchSize": 10
    },
    "FilterCriteria": {
      "Filters": [{"Pattern": "{\"eventName\": [\"INSERT\"]}"}]
    }
  }' \
  --enrichment arn:aws:lambda:us-east-1:123456789012:function:EnrichOrder \
  --target arn:aws:events:us-east-1:123456789012:event-bus/OrdersBus \
  --role-arn arn:aws:iam::123456789012:role/pipes-execution-role
```

---

## 8. IAM — Execution Role

The pipe assumes an **execution role** that needs permissions to:
- Poll the source (e.g., `sqs:ReceiveMessage`, `dynamodb:GetRecords`)
- Invoke the enrichment (e.g., `lambda:InvokeFunction`)
- Send to the target (e.g., `events:PutEvents`)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sqs:ReceiveMessage", "sqs:DeleteMessage", "sqs:GetQueueAttributes"],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue"
    },
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:EnrichOrder"
    },
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:123456789012:event-bus/OrdersBus"
    }
  ]
}
```

---

## 9. Common Exam Scenarios

### Scenario 1: Process SQS messages, enrich with external data, send to Step Functions
**Without Pipes:** Lambda polls SQS → calls enrichment API → starts Step Functions execution (3 Lambda functions or custom code).
**With Pipes:** One pipe: SQS source → Lambda enrichment → Step Functions target. No polling Lambda needed.

### Scenario 2: Filter DynamoDB stream events before processing
**Solution:** Pipe with DynamoDB Stream source + filter pattern. Only matching change events reach the target — reduces Lambda invocations and cost vs filtering inside Lambda.

### Scenario 3: Connect Kafka to EventBridge for fan-out
**Solution:** Pipe with MSK source → EventBridge bus target. The bus then fans out to multiple rule targets. Pipes handles the Kafka polling; EventBridge handles the fan-out.

---

## 10. Exam Tips

- Pipes = **point-to-point** with enrichment. Rules = **fan-out**. The exam will describe a scenario — pick the right one.
- The enrichment step is what makes Pipes unique — it's built-in, no glue Lambda needed.
- Pipes **poll** their sources; they don't receive pushed events like an event bus rule does.
- Filter before enrichment = cost savings (don't invoke enrichment for records you'll discard).
- Step Functions enrichment must be an **Express Workflow** (synchronous) — Standard Workflows are async and can't return a result to the pipe.
- Pipes require an **execution role** — a common exam trap is forgetting the IAM permissions.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
