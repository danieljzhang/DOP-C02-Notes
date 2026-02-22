# AWS EventBridge - DOP-C02 Exam Notes

## 1. Overview

**Amazon EventBridge** is a serverless event bus service that connects applications using events from AWS services, custom applications, and SaaS applications.

### Key Characteristics
- **Serverless** - No infrastructure to manage
- **Event-driven** - React to state changes in real-time
- **Scalable** - Automatically scales with event volume
- **Flexible routing** - Route events to multiple targets
- **Schema registry** - Discover and manage event schemas
- **Archive & replay** - Store and replay events

### What Problem Does It Solve?
- Decouples event producers from consumers
- Enables event-driven architectures
- Simplifies integration between services
- Provides centralized event routing
- Enables real-time application responses
- Facilitates SaaS integration

---

## 2. Core Concepts

### Event
- JSON object representing state change
- Contains metadata and data
- Immutable once created
- Maximum size: 256 KB

### Event Bus
- Receives events from sources
- Routes events to targets based on rules
- Types: Default, Custom, Partner
- One default bus per account per region

### Rule
- Matches incoming events using patterns
- Routes matched events to targets
- Can have multiple targets (up to 5)
- Can transform event before sending to target

### Event Pattern
- JSON structure for filtering events
- Matches event fields
- Supports exact match, prefix, suffix, numeric ranges
- Logical operators: OR (array), AND (nested)

### Target
- Destination for matched events
- AWS services or custom endpoints
- Can transform input before delivery
- Supports retry and dead-letter queues

### Event Source
- Origin of events
- AWS services, custom applications, SaaS partners
- Publishes events to event bus

---

## 3. Event Structure

### Standard Event Format
```json
{
  "version": "0",
  "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
  "detail-type": "EC2 Instance State-change Notification",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2023-11-15T22:34:12Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
  ],
  "detail": {
    "instance-id": "i-1234567890abcdef0",
    "state": "running"
  }
}
```

### Event Fields
- **version** - Event format version (always "0")
- **id** - Unique event identifier (UUID)
- **detail-type** - Free-form string describing event
- **source** - Event source identifier
- **account** - AWS account ID
- **time** - Event timestamp (ISO 8601)
- **region** - AWS region
- **resources** - ARNs of affected resources
- **detail** - Event-specific data (custom structure)

---

## 4. Event Buses

### Default Event Bus
- Automatically created in each account/region
- Receives events from AWS services
- Cannot be deleted
- Name: "default"

### Custom Event Bus
- Created for custom applications
- Isolate events by application/team
- Can set resource-based policies
- Can be deleted when not needed

### Partner Event Bus
- Created when connecting to SaaS partners
- Receives events from partner applications
- Examples: Datadog, PagerDuty, Zendesk
- Automatically created upon partner authorization

### Cross-Account Event Bus
- Send events to another AWS account
- Requires resource-based policy on target bus
- Use case: Centralized event processing

---

## 5. Event Patterns

### Exact Match
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"]
}
```

### Prefix Match
```json
{
  "source": [{
    "prefix": "aws."
  }]
}
```

### Suffix Match
```json
{
  "detail": {
    "filename": [{
      "suffix": ".jpg"
    }]
  }
}
```

### Anything-but Match
```json
{
  "detail": {
    "state": [{
      "anything-but": "initializing"
    }]
  }
}
```

### Numeric Match
```json
{
  "detail": {
    "temperature": [{
      "numeric": [">", 0, "<=", 100]
    }]
  }
}
```

### Exists Match
```json
{
  "detail": {
    "errorCode": [{
      "exists": true
    }]
  }
}
```

### OR Logic (Array)
```json
{
  "source": ["aws.ec2", "aws.ecs"]
}
```

### AND Logic (Nested)
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["running"]
  }
}
```

### Complex Pattern
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["running", "stopped"],
    "instance-type": [{
      "prefix": "t3."
    }]
  }
}
```

---

## 6. Targets

### Supported Targets
- **Lambda** - Invoke function
- **SNS** - Publish to topic
- **SQS** - Send message to queue
- **Kinesis Data Streams** - Put record
- **Kinesis Data Firehose** - Deliver data
- **Step Functions** - Start execution
- **CodePipeline** - Start pipeline
- **CodeBuild** - Start build
- **ECS Task** - Run task
- **Batch** - Submit job
- **Systems Manager** - Run command, automation
- **EC2** - Create snapshot, reboot, stop, terminate
- **API Gateway** - Invoke REST API
- **EventBridge API Destination** - HTTP endpoint
- **Another Event Bus** - Cross-account or same account

### Target Configuration
```json
{
  "Targets": [
    {
      "Id": "1",
      "Arn": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction",
      "RetryPolicy": {
        "MaximumRetryAttempts": 2,
        "MaximumEventAge": 3600
      },
      "DeadLetterConfig": {
        "Arn": "arn:aws:sqs:us-east-1:123456789012:dlq"
      }
    }
  ]
}
```

---

## 7. Input Transformation

### Input Path
- Extract specific fields from event
- Create variables for use in template
- JSONPath syntax

### Input Template
- Define structure of data sent to target
- Reference variables from input path
- Can add static values

### Example Transformation
```json
{
  "InputPath": "$.detail",
  "InputTemplate": "\"Instance <instance-id> is now <state>\""
}
```

### Full Example
```json
{
  "InputPathsMap": {
    "instance": "$.detail.instance-id",
    "state": "$.detail.state",
    "time": "$.time"
  },
  "InputTemplate": "{\"message\": \"Instance <instance> changed to <state> at <time>\"}"
}
```

---

## 8. Archive & Replay

### Archive
- Store events for later replay
- Retention: indefinite or specified days
- Filter events to archive
- Encrypted at rest

### Archive Configuration
```json
{
  "ArchiveName": "MyArchive",
  "EventSourceArn": "arn:aws:events:us-east-1:123456789012:event-bus/default",
  "Description": "Archive all EC2 events",
  "EventPattern": "{\"source\":[\"aws.ec2\"]}",
  "RetentionDays": 90
}
```

### Replay
- Replay archived events to event bus
- Specify time range
- Events replayed with original timestamp
- Use case: Testing, recovery, reprocessing

### Replay Configuration
```json
{
  "ReplayName": "MyReplay",
  "EventSourceArn": "arn:aws:events:us-east-1:123456789012:archive/MyArchive",
  "EventStartTime": "2023-01-01T00:00:00Z",
  "EventEndTime": "2023-01-31T23:59:59Z",
  "Destination": {
    "Arn": "arn:aws:events:us-east-1:123456789012:event-bus/default"
  }
}
```

---

## 9. Schema Registry

### Overview
- Discover event schemas automatically
- Version control for schemas
- Generate code bindings
- OpenAPI 3.0 format

### Schema Discovery
- Automatically infer schemas from events
- Enable on event bus
- Schemas stored in registry
- Versioned automatically

### Schema Registry Types
- **AWS event schemas** - Pre-defined for AWS services
- **Discovered schemas** - Auto-discovered from custom events
- **Custom schemas** - Manually uploaded

### Code Bindings
- Generate code for Java, Python, TypeScript
- Strongly typed event handling
- Download from console or CLI

---

## 10. API Destinations

### Overview
- Send events to HTTP endpoints
- Support for any HTTP API
- Authentication support
- Retry and DLQ support

### Connection
- Stores authentication credentials
- Types: Basic, OAuth, API Key
- Encrypted with Secrets Manager
- Reusable across destinations

### API Destination Configuration
```json
{
  "Name": "MyAPIDestination",
  "ConnectionArn": "arn:aws:events:us-east-1:123456789012:connection/MyConnection",
  "InvocationEndpoint": "https://api.example.com/webhook",
  "HttpMethod": "POST",
  "InvocationRateLimitPerSecond": 10
}
```

### Connection Configuration
```json
{
  "Name": "MyConnection",
  "AuthorizationType": "OAUTH_CLIENT_CREDENTIALS",
  "AuthParameters": {
    "OAuthParameters": {
      "ClientParameters": {
        "ClientID": "my-client-id"
      },
      "AuthorizationEndpoint": "https://auth.example.com/oauth/authorize",
      "HttpMethod": "POST"
    }
  }
}
```

---

## 11. Cross-Account Events

### Setup Process
1. Create custom event bus in target account
2. Add resource-based policy to allow source account
3. Create rule in source account to send to target bus
4. Create rule in target account to process events

### Resource-Based Policy (Target Account)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountToPublish",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:222222222222:event-bus/central-bus"
    }
  ]
}
```

### Rule in Source Account
```json
{
  "Name": "SendToCentralBus",
  "EventPattern": "{\"source\":[\"custom.app\"]}",
  "Targets": [
    {
      "Arn": "arn:aws:events:us-east-1:222222222222:event-bus/central-bus",
      "RoleArn": "arn:aws:iam::111111111111:role/EventBridgeRole"
    }
  ]
}
```

---

## 12. IAM Permissions

### PutEvents Permission
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:123456789012:event-bus/default"
    }
  ]
}
```

### Rule Management Permissions
```json
{
  "Effect": "Allow",
  "Action": [
    "events:PutRule",
    "events:DeleteRule",
    "events:DescribeRule",
    "events:EnableRule",
    "events:DisableRule",
    "events:PutTargets",
    "events:RemoveTargets"
  ],
  "Resource": "arn:aws:events:us-east-1:123456789012:rule/*"
}
```

### Target Invocation Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction"
    }
  ]
}
```

### Trust Policy for EventBridge
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 13. Integration with AWS Services

### CodePipeline
```json
{
  "source": ["aws.codepipeline"],
  "detail-type": ["CodePipeline Pipeline Execution State Change"],
  "detail": {
    "state": ["FAILED"],
    "pipeline": ["MyPipeline"]
  }
}
```

### CodeBuild
```json
{
  "source": ["aws.codebuild"],
  "detail-type": ["CodeBuild Build State Change"],
  "detail": {
    "build-status": ["FAILED", "STOPPED"]
  }
}
```

### CodeDeploy
```json
{
  "source": ["aws.codedeploy"],
  "detail-type": ["CodeDeploy Deployment State-change Notification"],
  "detail": {
    "state": ["FAILURE"]
  }
}
```

### CodeCommit
```json
{
  "source": ["aws.codecommit"],
  "detail-type": ["CodeCommit Repository State Change"],
  "detail": {
    "event": ["referenceCreated", "referenceUpdated"],
    "referenceType": ["branch"],
    "referenceName": ["main"]
  }
}
```

### EC2
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}
```

### Auto Scaling
```json
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance Launch Successful"]
}
```

### ECS
```json
{
  "source": ["aws.ecs"],
  "detail-type": ["ECS Task State Change"],
  "detail": {
    "lastStatus": ["STOPPED"],
    "stoppedReason": [{
      "exists": true
    }]
  }
}
```

### CloudFormation
```json
{
  "source": ["aws.cloudformation"],
  "detail-type": ["CloudFormation Stack Status Change"],
  "detail": {
    "status-details": {
      "status": ["CREATE_COMPLETE", "UPDATE_COMPLETE"]
    }
  }
}
```

---

## 14. Scheduled Events (Cron/Rate)

### Rate Expression
```
rate(5 minutes)
rate(1 hour)
rate(1 day)
```

### Cron Expression
```
cron(0 12 * * ? *)        # Every day at 12:00 PM UTC
cron(15 10 ? * MON-FRI *) # Weekdays at 10:15 AM UTC
cron(0 18 ? * MON *)      # Every Monday at 6:00 PM UTC
cron(0/5 * * * ? *)       # Every 5 minutes
```

### Cron Format
```
cron(Minutes Hours Day-of-month Month Day-of-week Year)
```

### Schedule Rule Example
```json
{
  "Name": "DailyBackup",
  "ScheduleExpression": "cron(0 2 * * ? *)",
  "State": "ENABLED",
  "Targets": [
    {
      "Arn": "arn:aws:lambda:us-east-1:123456789012:function:BackupFunction",
      "Id": "1"
    }
  ]
}
```

---

## 15. Monitoring & Logging

### CloudWatch Metrics
- **Invocations** - Number of times target invoked
- **FailedInvocations** - Failed target invocations
- **TriggeredRules** - Rules that matched events
- **MatchedEvents** - Events matched by rules
- **ThrottledRules** - Rules throttled

### CloudTrail Logging
- All API calls logged
- Rule creation, updates, deletions
- PutEvents calls (optional)
- Target modifications

### Dead Letter Queues
- Capture failed event deliveries
- SQS queue as DLQ
- Analyze and retry failed events
- Set on target configuration

### Retry Policy
- Automatic retries for failed deliveries
- Configure max retry attempts (0-185)
- Configure max event age (60-86400 seconds)
- Exponential backoff

---

## 16. Common Exam Scenarios

### Scenario 1: Trigger Lambda on EC2 state change
**Solution:**
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["running"]
  }
}
```
Target: Lambda function

### Scenario 2: Send CodePipeline failures to SNS
**Solution:**
```json
{
  "source": ["aws.codepipeline"],
  "detail-type": ["CodePipeline Pipeline Execution State Change"],
  "detail": {
    "state": ["FAILED"]
  }
}
```
Target: SNS topic

### Scenario 3: Cross-account event aggregation
**Solution:**
- Create custom event bus in central account
- Add resource-based policy allowing source accounts
- Create rules in source accounts targeting central bus
- Create rules in central account for processing

### Scenario 4: Schedule Lambda every 5 minutes
**Solution:**
- Create rule with schedule expression: `rate(5 minutes)`
- Add Lambda as target
- Ensure Lambda has resource-based policy

### Scenario 5: Archive events for compliance
**Solution:**
- Create archive on event bus
- Set retention period (e.g., 7 years)
- Filter events to archive
- Enable encryption

### Scenario 6: Send events to external HTTP API
**Solution:**
- Create API Destination with endpoint
- Create Connection with authentication
- Create rule with API Destination as target
- Configure retry and DLQ

### Scenario 7: Transform event before sending to SQS
**Solution:**
- Use input transformation on target
- Define InputPathsMap to extract fields
- Define InputTemplate for output format
- Send transformed data to SQS

### Scenario 8: Replay events for testing
**Solution:**
- Create archive (if not exists)
- Create replay with time range
- Specify destination event bus
- Monitor replay progress

---

## 17. Best Practices

### Design
- Use custom event buses for application isolation
- Create specific event patterns (avoid catch-all)
- Use meaningful detail-type values
- Include relevant data in detail field
- Keep events under 256 KB

### Security
- Use least privilege IAM policies
- Enable encryption for archives
- Use Secrets Manager for API credentials
- Implement resource-based policies for cross-account
- Enable CloudTrail logging

### Reliability
- Configure retry policies on targets
- Use dead letter queues for failed events
- Monitor FailedInvocations metric
- Test event patterns thoroughly
- Use idempotent targets

### Performance
- Use parallel targets for fan-out
- Avoid synchronous processing in Lambda
- Use SQS for buffering high-volume events
- Consider Kinesis for streaming use cases
- Monitor throttling metrics

### Cost Optimization
- Clean up unused rules
- Use appropriate archive retention
- Consolidate similar rules
- Use input transformation to reduce payload size
- Monitor PutEvents costs

### Operational
- Use descriptive names for rules
- Tag resources for organization
- Document event schemas
- Use schema registry for discovery
- Enable schema discovery for custom events

---

## 18. Troubleshooting

### Events Not Matching Rule
- Verify event pattern syntax
- Test pattern with sample event
- Check event source and detail-type
- Use CloudWatch Logs for debugging
- Enable CloudTrail for PutEvents

### Target Not Invoked
- Check IAM role permissions
- Verify target ARN is correct
- Check target service quotas
- Review retry policy configuration
- Check dead letter queue

### Cross-Account Events Not Working
- Verify resource-based policy on target bus
- Check IAM role in source account
- Verify event bus ARN is correct
- Check network connectivity (if applicable)
- Review CloudTrail logs

### High Latency
- Check target processing time
- Review retry attempts
- Monitor throttling metrics
- Consider async processing
- Use SQS for buffering

---

## 19. Integration Patterns

### Event-Driven CI/CD
```
CodeCommit → EventBridge → Lambda → CodePipeline
```

### Centralized Logging
```
Multiple Accounts → EventBridge (Central) → Kinesis Firehose → S3
```

### Automated Remediation
```
AWS Config → EventBridge → Lambda → Remediation Action
```

### Multi-Region Failover
```
Primary Region Events → EventBridge → Cross-Region Bus → Failover Actions
```

### SaaS Integration
```
Partner Event Bus → EventBridge Rule → Lambda → Internal Systems
```

---

## 20. CLI Commands Reference

### Event Bus Operations
```bash
# Create event bus
aws events create-event-bus --name my-custom-bus

# List event buses
aws events list-event-buses

# Delete event bus
aws events delete-event-bus --name my-custom-bus

# Put resource policy
aws events put-permission \
  --event-bus-name my-custom-bus \
  --statement-id AllowAccount \
  --action events:PutEvents \
  --principal 111111111111
```

### Rule Operations
```bash
# Create rule
aws events put-rule \
  --name MyRule \
  --event-pattern '{"source":["aws.ec2"]}' \
  --state ENABLED

# Add target
aws events put-targets \
  --rule MyRule \
  --targets Id=1,Arn=arn:aws:lambda:us-east-1:123456789012:function:MyFunction

# List rules
aws events list-rules

# Delete rule
aws events delete-rule --name MyRule
```

### Event Operations
```bash
# Put events
aws events put-events \
  --entries '[{
    "Source": "custom.app",
    "DetailType": "User Action",
    "Detail": "{\"action\":\"login\",\"user\":\"john\"}"
  }]'

# Test event pattern
aws events test-event-pattern \
  --event-pattern '{"source":["aws.ec2"]}' \
  --event '{"source":"aws.ec2","detail-type":"EC2 Instance State-change Notification"}'
```

### Archive Operations
```bash
# Create archive
aws events create-archive \
  --archive-name MyArchive \
  --event-source-arn arn:aws:events:us-east-1:123456789012:event-bus/default \
  --retention-days 90

# Start replay
aws events start-replay \
  --replay-name MyReplay \
  --event-source-arn arn:aws:events:us-east-1:123456789012:archive/MyArchive \
  --event-start-time 2023-01-01T00:00:00Z \
  --event-end-time 2023-01-31T23:59:59Z \
  --destination Arn=arn:aws:events:us-east-1:123456789012:event-bus/default
```

---

## 21. Comparison with Other Services

### EventBridge vs SNS
| Feature | EventBridge | SNS |
|---------|-------------|-----|
| Pattern matching | Advanced | Basic (filter policies) |
| Targets | 18+ AWS services | Limited |
| Routing | Rule-based | Topic-based |
| Use case | Event routing | Pub/sub messaging |

### EventBridge vs SQS
| Feature | EventBridge | SQS |
|---------|-------------|-----|
| Purpose | Event routing | Message queuing |
| Filtering | Event patterns | Message attributes |
| Targets | Multiple | Single consumer |
| Use case | Event distribution | Decoupling |

### EventBridge vs CloudWatch Events
- EventBridge is evolution of CloudWatch Events
- Same API and functionality
- EventBridge adds: Schema registry, API destinations, SaaS integration
- CloudWatch Events name still used in some contexts

---

## 22. Exam Tips

### What to Remember
- **Event structure**: version, id, detail-type, source, account, time, region, resources, detail
- **Event buses**: Default (AWS services), Custom (applications), Partner (SaaS)
- **Event patterns**: Exact, prefix, suffix, anything-but, numeric, exists
- **Targets**: Up to 5 per rule, supports 18+ AWS services
- **Archive & Replay**: Store events, replay for testing/recovery
- **Cross-account**: Requires resource-based policy on target bus
- **API Destinations**: Send to HTTP endpoints with authentication
- **Scheduled events**: Cron and rate expressions
- **Input transformation**: Extract and format data for targets

### Common Traps
- Event size limit is 256 KB
- Maximum 5 targets per rule
- Cross-account requires resource-based policy (not IAM role alone)
- CloudWatch Events and EventBridge are same service
- PutEvents is not logged in CloudTrail by default
- Cron expressions use UTC timezone

### Scenario-Based Questions
- Focus on event pattern matching
- Understand cross-account event routing
- Know when to use archive & replay
- Understand input transformation
- Know integration with CI/CD services
- Understand scheduled events

### Integration Questions
- CodePipeline/CodeBuild/CodeDeploy event patterns
- Lambda invocation from events
- SNS/SQS as targets
- Step Functions integration
- Cross-account aggregation patterns

---

## 23. Quick Reference Cheat Sheet

### Event Pattern Operators
```
Exact: ["value"]
Prefix: [{"prefix": "val"}]
Suffix: [{"suffix": "ue"}]
Anything-but: [{"anything-but": "value"}]
Numeric: [{"numeric": [">", 0]}]
Exists: [{"exists": true}]
```

### Common Sources
```
aws.ec2
aws.ecs
aws.codepipeline
aws.codebuild
aws.codedeploy
aws.codecommit
aws.autoscaling
aws.cloudformation
```

### Schedule Expressions
```
rate(5 minutes)
rate(1 hour)
rate(1 day)
cron(0 12 * * ? *)
```

### Essential IAM Actions
```
events:PutEvents
events:PutRule
events:PutTargets
events:DeleteRule
events:RemoveTargets
```

---

## 24. Reference Links

### AWS Official Documentation
- [EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/what-is-amazon-eventbridge.html)
- [Event Patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)
- [Event Bus Permissions](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-use-resource-based.html)
- [Archive and Replay](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-archive.html)
- [API Destinations](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-api-destinations.html)
- [Schema Registry](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-schema.html)

---

## 25. Summary

AWS EventBridge is critical for event-driven architectures and CI/CD automation in the DOP-C02 exam. Key areas to master:

1. **Event structure** and standard fields
2. **Event patterns** (exact, prefix, suffix, numeric, exists)
3. **Event buses** (default, custom, partner)
4. **Targets** and target configuration
5. **Cross-account** event routing
6. **Archive & Replay** for testing and recovery
7. **API Destinations** for HTTP endpoints
8. **Scheduled events** (cron and rate)
9. **Input transformation** for targets
10. **Integration** with AWS CI/CD services

Understanding these concepts with hands-on practice will ensure success on EventBridge-related questions in the DOP-C02 exam.
