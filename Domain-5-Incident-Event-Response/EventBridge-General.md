# Amazon EventBridge - DOP-C02 Study Notes

## 1. Overview

### What is EventBridge?
- Serverless event bus service for application integration
- Routes events between AWS services, SaaS applications, and custom applications
- Formerly known as CloudWatch Events
- Event-driven architecture foundation

### Key Components
- **Event Buses** - Receive and route events
- **Rules** - Match events and route to targets
- **Targets** - Destinations for matched events
- **Events** - JSON messages describing state changes

### Use Cases
- Application decoupling
- Real-time data processing
- Automated incident response
- Cross-account event routing
- SaaS integration

---

## 2. Event Buses

### Default Event Bus
- Automatically created in each account
- Receives events from AWS services
- Cannot be deleted
- Regional service

### Custom Event Buses
- User-created event buses
- Isolate events by application or team
- Cross-account sharing capabilities
- Resource-based policies

```bash
# Create custom event bus
aws events create-event-bus --name "MyApplicationBus"

# Put resource policy on event bus
aws events put-permission \
  --principal "123456789012" \
  --action "events:PutEvents" \
  --statement-id "CrossAccountAccess"
```

### Partner Event Buses
- Created by SaaS providers
- Receive events from partner applications
- Automatic provisioning when configured

---

## 3. Events and Event Patterns

### Event Structure
```json
{
  "version": "0",
  "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
  "detail-type": "EC2 Instance State-change Notification",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2024-01-15T10:30:00Z",
  "region": "us-east-1",
  "detail": {
    "instance-id": "i-1234567890abcdef0",
    "state": "running"
  }
}
```

### Event Patterns
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["running", "stopped"]
  }
}
```

### Custom Events
```bash
# Send custom event
aws events put-events \
  --entries '[
    {
      "Source": "myapp.orders",
      "DetailType": "Order Placed",
      "Detail": "{\"orderId\":\"12345\",\"customerId\":\"67890\",\"amount\":99.99}"
    }
  ]'
```

---

## 4. Rules and Targets

### Create Rule
```bash
# Create rule for EC2 state changes
aws events put-rule \
  --name "EC2StateChange" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"],
    "detail": {
      "state": ["running", "stopped"]
    }
  }' \
  --state ENABLED
```

### Rule Targets
```bash
# Add Lambda target
aws events put-targets \
  --rule "EC2StateChange" \
  --targets '[
    {
      "Id": "1",
      "Arn": "arn:aws:lambda:us-east-1:123456789012:function:ProcessEC2StateChange"
    }
  ]'

# Add SNS target
aws events put-targets \
  --rule "EC2StateChange" \
  --targets '[
    {
      "Id": "2",
      "Arn": "arn:aws:sns:us-east-1:123456789012:ec2-notifications"
    }
  ]'
```

### Supported Targets
- Lambda functions
- SNS topics
- SQS queues
- Kinesis streams
- Step Functions
- ECS tasks
- Systems Manager Run Command
- API Gateway
- HTTP endpoints

---

## 5. Incident Response Automation

### Automated EC2 Response
```python
import boto3
import json

def lambda_handler(event, context):
    """Automated response to EC2 incidents"""
    
    detail = event['detail']
    instance_id = detail['instance-id']
    state = detail['state']
    
    if state == 'stopped' and 'unexpected' in detail.get('reason', ''):
        # Investigate unexpected stop
        investigate_instance_stop(instance_id)
        
        # Restart if healthy
        if is_instance_healthy(instance_id):
            restart_instance(instance_id)
        else:
            create_incident_ticket(instance_id, 'Instance health check failed')

def investigate_instance_stop(instance_id):
    """Investigate why instance stopped"""
    ec2 = boto3.client('ec2')
    
    # Get instance details
    response = ec2.describe_instances(InstanceIds=[instance_id])
    
    # Check CloudTrail for stop events
    cloudtrail = boto3.client('cloudtrail')
    events = cloudtrail.lookup_events(
        LookupAttributes=[
            {
                'AttributeKey': 'ResourceName',
                'AttributeValue': instance_id
            }
        ]
    )
    
    return events
```

### Security Incident Response
```python
def handle_security_incident(event, context):
    """Handle GuardDuty security findings"""
    
    finding = event['detail']
    severity = finding['severity']
    
    if severity >= 7.0:  # High severity
        # Isolate affected resources
        isolate_compromised_resources(finding)
        
        # Create incident
        create_security_incident(finding)
        
        # Notify security team
        notify_security_team(finding)
    
    elif severity >= 4.0:  # Medium severity
        # Log for investigation
        log_security_event(finding)
        
        # Apply automated remediation
        apply_security_remediation(finding)

def isolate_compromised_resources(finding):
    """Isolate compromised EC2 instances"""
    ec2 = boto3.client('ec2')
    
    for resource in finding.get('service', {}).get('resources', []):  # was: iterated a boolean (== comparison), which raises TypeError
        if resource['resourceType'] == 'Instance':
            instance_id = resource['instanceDetails']['instanceId']
            
            # Create isolation security group
            isolation_sg = create_isolation_security_group()
            
            # Apply to instance
            ec2.modify_instance_attribute(
                InstanceId=instance_id,
                Groups=[isolation_sg]
            )
```

---

## 6. Application Integration Patterns

### Microservices Communication
```json
{
  "Rules": [
    {
      "Name": "OrderProcessing",
      "EventPattern": {
        "source": ["ecommerce.orders"],
        "detail-type": ["Order Placed"]
      },
      "Targets": [
        {
          "Id": "inventory-service",
          "Arn": "arn:aws:lambda:us-east-1:123456789012:function:UpdateInventory"
        },
        {
          "Id": "payment-service", 
          "Arn": "arn:aws:lambda:us-east-1:123456789012:function:ProcessPayment"
        },
        {
          "Id": "notification-service",
          "Arn": "arn:aws:sns:us-east-1:123456789012:order-notifications"
        }
      ]
    }
  ]
}
```

### Cross-Account Event Routing
```bash
# Grant cross-account permissions
aws events put-permission \
  --principal "111122223333" \
  --action "events:PutEvents" \
  --statement-id "CrossAccountOrderEvents"

# Create cross-account rule
aws events put-rule \
  --name "CrossAccountOrders" \
  --event-pattern '{
    "source": ["partner.orders"],
    "account": ["111122223333"]
  }'
```

---

## 7. Monitoring and Observability

### EventBridge Metrics
```python
import boto3

def monitor_eventbridge_metrics():
    """Monitor EventBridge performance"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Get rule invocation metrics
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/Events',
        MetricName='InvocationsCount',
        Dimensions=[
            {
                'Name': 'RuleName',
                'Value': 'EC2StateChange'
            }
        ],
        StartTime=datetime.utcnow() - timedelta(hours=1),
        EndTime=datetime.utcnow(),
        Period=300,
        Statistics=['Sum']
    )
    
    return response['Datapoints']
```

### Dead Letter Queues
```bash
# Create DLQ for failed events
aws sqs create-queue --queue-name "eventbridge-dlq"

# Configure rule with DLQ
aws events put-targets \
  --rule "MyRule" \
  --targets '[
    {
      "Id": "1",
      "Arn": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction",
      "DeadLetterConfig": {
        "Arn": "arn:aws:sqs:us-east-1:123456789012:eventbridge-dlq"
      }
    }
  ]'
```

---

## 8. Schema Registry

### Event Schema Management
```bash
# Create schema registry
aws schemas create-registry --registry-name "MyAppRegistry"

# Create schema
aws schemas create-schema \
  --registry-name "MyAppRegistry" \
  --schema-name "OrderPlaced" \
  --type "JSONSchemaDraft4" \
  --content '{
    "type": "object",
    "properties": {
      "orderId": {"type": "string"},
      "customerId": {"type": "string"},
      "amount": {"type": "number"}
    },
    "required": ["orderId", "customerId", "amount"]
  }'
```

### Schema Discovery
```python
import boto3

def discover_event_schemas():
    """Automatically discover event schemas"""
    
    schemas = boto3.client('schemas')
    
    # Start schema discovery
    response = schemas.start_discoverer(
        DiscovererId='MyAppDiscoverer',
        SourceArn='arn:aws:events:us-east-1:123456789012:event-bus/MyAppBus'
    )
    
    return response
```

---

## 9. Error Handling and Retry

### Retry Configuration
```json
{
  "RetryPolicy": {
    "MaximumRetryAttempts": 3,
    "MaximumEventAge": 3600
  },
  "DeadLetterConfig": {
    "Arn": "arn:aws:sqs:us-east-1:123456789012:failed-events"
  }
}
```

### Error Handling Patterns
```python
def robust_event_handler(event, context):
    """Event handler with error handling"""
    
    try:
        # Process event
        result = process_event(event)
        
        # Log success
        logger.info(f"Successfully processed event: {event['id']}")
        
        return result
        
    except RetryableError as e:
        # Log for retry
        logger.warning(f"Retryable error processing event: {e}")
        raise e
        
    except FatalError as e:
        # Log and send to DLQ
        logger.error(f"Fatal error processing event: {e}")
        send_to_dlq(event, str(e))
        
    except Exception as e:
        # Unknown error - log and investigate
        logger.error(f"Unknown error processing event: {e}")
        create_investigation_ticket(event, str(e))
        raise e
```

---

## 10. Performance Optimization

### Batch Processing
```python
def batch_event_processor(event, context):
    """Process multiple events in batch"""
    
    # EventBridge can send up to 10 events per invocation
    records = event.get('Records', [])
    
    batch_size = 10
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        process_event_batch(batch)

def process_event_batch(events):
    """Process batch of events efficiently"""
    
    # Group events by type
    events_by_type = {}
    for event in events:
        event_type = event['detail-type']
        if event_type not in events_by_type:
            events_by_type[event_type] = []
        events_by_type[event_type].append(event)
    
    # Process each type efficiently
    for event_type, type_events in events_by_type.items():
        processor = get_processor_for_type(event_type)
        processor.process_batch(type_events)
```

### Connection Pooling
```python
import boto3
from botocore.config import Config

# Configure connection pooling
config = Config(
    max_pool_connections=50,
    retries={'max_attempts': 3}
)

# Reuse clients
eventbridge = boto3.client('events', config=config)
lambda_client = boto3.client('lambda', config=config)
```

---

## 11. Common Exam Scenarios

### Scenario 1: Automate incident response for failed deployments
**Solution:**
- Create EventBridge rule for CodeDeploy failure events
- Target Lambda function for automated rollback
- Send notifications to operations team
- Create incident tickets automatically

### Scenario 2: Decouple microservices communication
**Solution:**
- Use custom event bus for application events
- Create rules for each service integration
- Implement retry and DLQ for reliability
- Use schema registry for event validation

### Scenario 3: Cross-account event routing for multi-account setup
**Solution:**
- Configure cross-account permissions on event bus
- Create rules in target accounts
- Implement event filtering and routing
- Monitor cross-account event flow

### Scenario 4: Real-time security incident response
**Solution:**
- Create rules for GuardDuty findings
- Implement automated isolation procedures
- Trigger incident response workflows
- Notify security teams immediately

### Scenario 5: Application monitoring and alerting
**Solution:**
- Send custom application events to EventBridge
- Create rules for error conditions
- Target CloudWatch alarms and SNS topics
- Implement escalation procedures

---

## 12. CLI Commands Reference

### Event Bus Management
```bash
# List event buses
aws events list-event-buses

# Create custom event bus
aws events create-event-bus --name "MyBus"

# Delete event bus
aws events delete-event-bus --name "MyBus"
```

### Rules Management
```bash
# List rules
aws events list-rules

# Create rule
aws events put-rule --name "MyRule" --event-pattern '{}'

# Delete rule
aws events delete-rule --name "MyRule"
```

### Event Operations
```bash
# Send events
aws events put-events --entries file://events.json

# Test event pattern
aws events test-event-pattern \
  --event-pattern '{"source":["aws.ec2"]}' \
  --event '{"source":"aws.ec2","detail-type":"EC2 Instance State-change Notification"}'
```

---

## 13. Exam Tips

### Key Points to Remember
- EventBridge is regional service
- Default event bus receives AWS service events
- Custom event buses provide isolation
- Rules use event patterns for matching
- Multiple targets per rule supported
- Cross-account sharing requires permissions

### Common Mistakes
- Not configuring proper IAM permissions for targets
- Forgetting to enable rules after creation
- Not implementing error handling and DLQ
- Overlooking cross-account permission requirements
- Not monitoring rule performance and failures

### Best Practices for Exam
- Understand event-driven architecture patterns
- Know integration with AWS services
- Understand cross-account event routing
- Know error handling and retry mechanisms
- Understand monitoring and troubleshooting
- Know schema registry capabilities