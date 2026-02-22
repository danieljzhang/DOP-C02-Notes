# AWS SNS & SQS - DOP-C02 Study Notes

## 1. Overview

### Amazon SNS (Simple Notification Service)
- Fully managed pub/sub messaging service
- Push-based message delivery
- Fan-out messaging to multiple subscribers
- Integration with AWS services and external endpoints

### Amazon SQS (Simple Queue Service)
- Fully managed message queuing service
- Pull-based message consumption
- Decouples application components
- Reliable message delivery with visibility timeout

### Use Cases
- **SNS**: Notifications, alerts, fan-out messaging
- **SQS**: Task queues, batch processing, decoupling
- **Together**: Reliable message processing with fan-out

---

## 2. SNS Core Concepts

### Topics
- Communication channels for messages
- Publishers send messages to topics
- Subscribers receive messages from topics
- Support for standard and FIFO topics

### Subscriptions
- Endpoints that receive messages from topics
- Multiple subscription types supported
- Message filtering capabilities
- Delivery retry policies

### Supported Protocols
- **HTTP/HTTPS** - Web endpoints
- **Email/Email-JSON** - Email notifications
- **SMS** - Text messages
- **SQS** - Queue integration
- **Lambda** - Function invocation
- **Mobile Push** - iOS, Android, Windows

---

## 3. SQS Core Concepts

### Queue Types
- **Standard Queues** - At-least-once delivery, best-effort ordering
- **FIFO Queues** - Exactly-once delivery, strict ordering
- **Dead Letter Queues** - Failed message handling

### Message Attributes
- **Message Body** - Up to 256KB of data
- **Message Attributes** - Metadata key-value pairs
- **Visibility Timeout** - Message processing time
- **Retention Period** - How long messages are kept

### Polling Types
- **Short Polling** - Immediate response (may be empty)
- **Long Polling** - Wait for messages (up to 20 seconds)

---

## 4. Incident Response with SNS

### Alert Distribution
```python
import boto3
import json

def send_incident_alert(incident_details):
    """Send incident alert to multiple channels"""
    
    sns = boto3.client('sns')
    
    # Create alert message
    message = {
        "incident_id": incident_details['id'],
        "severity": incident_details['severity'],
        "description": incident_details['description'],
        "timestamp": incident_details['timestamp'],
        "affected_services": incident_details['services']
    }
    
    # Send to incident response topic
    response = sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:incident-alerts',
        Message=json.dumps(message),
        Subject=f"INCIDENT: {incident_details['severity']} - {incident_details['description']}",
        MessageAttributes={
            'severity': {
                'DataType': 'String',
                'StringValue': incident_details['severity']
            },
            'service': {
                'DataType': 'String',
                'StringValue': incident_details['primary_service']
            }
        }
    )
    
    return response
```

### Multi-Channel Notifications
```bash
# Create incident response topic
aws sns create-topic --name "incident-alerts"

# Subscribe email for notifications
aws sns subscribe \
  --topic-arn "arn:aws:sns:us-east-1:123456789012:incident-alerts" \
  --protocol email \
  --notification-endpoint "oncall@example.com"

# Subscribe SMS for critical alerts
aws sns subscribe \
  --topic-arn "arn:aws:sns:us-east-1:123456789012:incident-alerts" \
  --protocol sms \
  --notification-endpoint "+1234567890"

# Subscribe Lambda for automated response
aws sns subscribe \
  --topic-arn "arn:aws:sns:us-east-1:123456789012:incident-alerts" \
  --protocol lambda \
  --notification-endpoint "arn:aws:lambda:us-east-1:123456789012:function:IncidentResponse"
```

### Message Filtering
```json
{
  "severity": ["HIGH", "CRITICAL"],
  "service": ["web-app", "database"],
  "region": ["us-east-1", "us-west-2"]
}
```

---

## 5. Event Processing with SQS

### Reliable Event Processing
```python
import boto3
import json

def process_incident_events():
    """Process incident events from SQS queue"""
    
    sqs = boto3.client('sqs')
    queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789012/incident-events'
    
    while True:
        # Receive messages
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,  # Long polling
            MessageAttributeNames=['All']
        )
        
        messages = response.get('Messages', [])
        
        for message in messages:
            try:
                # Process event
                event_data = json.loads(message['Body'])
                process_incident_event(event_data)
                
                # Delete message after successful processing
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
                
            except Exception as e:
                print(f"Error processing message: {e}")
                # Message will become visible again after visibility timeout

def process_incident_event(event_data):
    """Process individual incident event"""
    
    event_type = event_data.get('event_type')
    
    if event_type == 'service_down':
        handle_service_down(event_data)
    elif event_type == 'high_error_rate':
        handle_high_error_rate(event_data)
    elif event_type == 'resource_exhaustion':
        handle_resource_exhaustion(event_data)
```

### Dead Letter Queue Configuration
```bash
# Create main queue
aws sqs create-queue \
  --queue-name "incident-events" \
  --attributes '{
    "VisibilityTimeoutSeconds": "300",
    "MessageRetentionPeriod": "1209600",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'

# Create dead letter queue
aws sqs create-queue --queue-name "incident-events-dlq"

# Configure redrive policy
aws sqs set-queue-attributes \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/incident-events" \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456789012:incident-events-dlq\",\"maxReceiveCount\":3}"
  }'
```

---

## 6. SNS + SQS Integration Pattern

### Fan-Out Architecture
```python
def setup_fanout_architecture():
    """Set up SNS topic with multiple SQS subscribers"""
    
    sns = boto3.client('sns')
    sqs = boto3.client('sqs')
    
    # Create SNS topic
    topic_response = sns.create_topic(Name='incident-fanout')
    topic_arn = topic_response['TopicArn']
    
    # Create SQS queues for different processors
    queues = [
        'incident-logging',
        'incident-metrics',
        'incident-notifications',
        'incident-automation'
    ]
    
    for queue_name in queues:
        # Create queue
        queue_response = sqs.create_queue(QueueName=queue_name)
        queue_url = queue_response['QueueUrl']
        
        # Get queue ARN
        attrs = sqs.get_queue_attributes(
            QueueUrl=queue_url,
            AttributeNames=['QueueArn']
        )
        queue_arn = attrs['Attributes']['QueueArn']
        
        # Subscribe queue to topic
        sns.subscribe(
            TopicArn=topic_arn,
            Protocol='sqs',
            Endpoint=queue_arn
        )
        
        # Set queue policy to allow SNS
        set_queue_policy_for_sns(queue_url, queue_arn, topic_arn)

def set_queue_policy_for_sns(queue_url, queue_arn, topic_arn):
    """Set SQS queue policy to allow SNS to send messages"""
    
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {"Service": "sns.amazonaws.com"},
                "Action": "sqs:SendMessage",
                "Resource": queue_arn,
                "Condition": {
                    "ArnEquals": {
                        "aws:SourceArn": topic_arn
                    }
                }
            }
        ]
    }
    
    sqs = boto3.client('sqs')
    sqs.set_queue_attributes(
        QueueUrl=queue_url,
        Attributes={'Policy': json.dumps(policy)}
    )
```

---

## 7. FIFO Queues for Ordered Processing

### FIFO Queue Configuration
```bash
# Create FIFO queue
aws sqs create-queue \
  --queue-name "incident-events.fifo" \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true",
    "VisibilityTimeoutSeconds": "300"
  }'
```

### Ordered Event Processing
```python
def send_ordered_incident_events(incident_id, events):
    """Send incident events in order using FIFO queue"""
    
    sqs = boto3.client('sqs')
    queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789012/incident-events.fifo'
    
    for i, event in enumerate(events):
        response = sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(event),
            MessageGroupId=incident_id,  # Ensures ordering per incident
            MessageDeduplicationId=f"{incident_id}-{i}",  # Prevents duplicates
            MessageAttributes={
                'event_type': {
                    'DataType': 'String',
                    'StringValue': event['type']
                },
                'sequence': {
                    'DataType': 'Number',
                    'StringValue': str(i)
                }
            }
        )
```

---

## 8. Monitoring and Alerting

### CloudWatch Metrics
```python
def monitor_queue_metrics():
    """Monitor SQS queue metrics for incident response"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create alarm for queue depth
    cloudwatch.put_metric_alarm(
        AlarmName='IncidentQueueDepthHigh',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        MetricName='ApproximateNumberOfVisibleMessages',
        Namespace='AWS/SQS',
        Period=300,
        Statistic='Average',
        Threshold=100,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:queue-alerts'
        ],
        AlarmDescription='Incident queue depth is high',
        Dimensions=[
            {
                'Name': 'QueueName',
                'Value': 'incident-events'
            }
        ]
    )
    
    # Create alarm for message age
    cloudwatch.put_metric_alarm(
        AlarmName='IncidentMessageAgeHigh',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=1,
        MetricName='ApproximateAgeOfOldestMessage',
        Namespace='AWS/SQS',
        Period=300,
        Statistic='Maximum',
        Threshold=600,  # 10 minutes
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:queue-alerts'
        ],
        Dimensions=[
            {
                'Name': 'QueueName',
                'Value': 'incident-events'
            }
        ]
    )
```

### Custom Metrics
```python
def publish_custom_metrics(queue_name, processed_count, error_count):
    """Publish custom metrics for incident processing"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Publish processing metrics
    cloudwatch.put_metric_data(
        Namespace='IncidentResponse',
        MetricData=[
            {
                'MetricName': 'ProcessedEvents',
                'Dimensions': [
                    {
                        'Name': 'QueueName',
                        'Value': queue_name
                    }
                ],
                'Value': processed_count,
                'Unit': 'Count'
            },
            {
                'MetricName': 'ProcessingErrors',
                'Dimensions': [
                    {
                        'Name': 'QueueName',
                        'Value': queue_name
                    }
                ],
                'Value': error_count,
                'Unit': 'Count'
            }
        ]
    )
```

---

## 9. Security and Access Control

### SNS Topic Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowIncidentServices",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/IncidentDetectionRole",
          "arn:aws:iam::123456789012:role/MonitoringRole"
        ]
      },
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:incident-alerts"
    },
    {
      "Sid": "AllowCrossAccountSubscription",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sns:Subscribe",
      "Resource": "arn:aws:sns:us-east-1:123456789012:incident-alerts"
    }
  ]
}
```

### SQS Queue Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowIncidentProcessors",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/IncidentProcessorRole"
      },
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:incident-events"
    }
  ]
}
```

### Encryption Configuration
```bash
# Create encrypted SNS topic
aws sns create-topic \
  --name "secure-incident-alerts" \
  --attributes '{
    "KmsMasterKeyId": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
  }'

# Create encrypted SQS queue
aws sqs create-queue \
  --queue-name "secure-incident-events" \
  --attributes '{
    "KmsMasterKeyId": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012",
    "KmsDataKeyReusePeriodSeconds": "300"
  }'
```

---

## 10. Error Handling and Resilience

### Retry Logic
```python
import time
import random
from botocore.exceptions import ClientError

def send_message_with_retry(queue_url, message, max_retries=3):
    """Send SQS message with exponential backoff retry"""
    
    sqs = boto3.client('sqs')
    
    for attempt in range(max_retries + 1):
        try:
            response = sqs.send_message(
                QueueUrl=queue_url,
                MessageBody=json.dumps(message)
            )
            return response
            
        except ClientError as e:
            if attempt == max_retries:
                raise e
            
            # Exponential backoff with jitter
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait_time)
```

### Circuit Breaker Pattern
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

---

## 11. Performance Optimization

### Batch Operations
```python
def send_messages_batch(queue_url, messages):
    """Send multiple messages in batch for better performance"""
    
    sqs = boto3.client('sqs')
    
    # SQS supports up to 10 messages per batch
    batch_size = 10
    
    for i in range(0, len(messages), batch_size):
        batch = messages[i:i + batch_size]
        
        entries = []
        for j, message in enumerate(batch):
            entries.append({
                'Id': str(j),
                'MessageBody': json.dumps(message)
            })
        
        response = sqs.send_message_batch(
            QueueUrl=queue_url,
            Entries=entries
        )
        
        # Handle failed messages
        if 'Failed' in response:
            for failed in response['Failed']:
                print(f"Failed to send message {failed['Id']}: {failed['Message']}")

def receive_messages_batch(queue_url):
    """Receive multiple messages for efficient processing"""
    
    sqs = boto3.client('sqs')
    
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=10,  # Maximum batch size
        WaitTimeSeconds=20,      # Long polling
        MessageAttributeNames=['All']
    )
    
    return response.get('Messages', [])
```

---

## 12. Common Exam Scenarios

### Scenario 1: Implement incident notification system
**Solution:**
- Create SNS topic for incident alerts
- Subscribe multiple endpoints (email, SMS, Lambda)
- Use message filtering for severity-based routing
- Implement retry and DLQ for reliability

### Scenario 2: Decouple incident processing components
**Solution:**
- Use SQS queues between processing stages
- Implement dead letter queues for failed messages
- Use visibility timeout for processing time
- Monitor queue depth and message age

### Scenario 3: Fan-out incident events to multiple processors
**Solution:**
- Create SNS topic for incident events
- Subscribe multiple SQS queues for different processors
- Configure queue policies for SNS access
- Implement parallel processing of events

### Scenario 4: Ensure ordered processing of incident events
**Solution:**
- Use SQS FIFO queue for ordered processing
- Set message group ID for ordering scope
- Use message deduplication ID to prevent duplicates
- Implement sequential processing logic

### Scenario 5: Cross-account incident notifications
**Solution:**
- Configure SNS topic policy for cross-account access
- Set up cross-account subscriptions
- Implement proper IAM roles and policies
- Monitor cross-account message delivery

---

## 13. CLI Commands Reference

### SNS Operations
```bash
# Create topic
aws sns create-topic --name "incident-alerts"

# Subscribe to topic
aws sns subscribe \
  --topic-arn "arn:aws:sns:us-east-1:123456789012:incident-alerts" \
  --protocol email \
  --notification-endpoint "admin@example.com"

# Publish message
aws sns publish \
  --topic-arn "arn:aws:sns:us-east-1:123456789012:incident-alerts" \
  --message "Incident detected" \
  --subject "Alert"
```

### SQS Operations
```bash
# Create queue
aws sqs create-queue --queue-name "incident-events"

# Send message
aws sqs send-message \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/incident-events" \
  --message-body "Event data"

# Receive messages
aws sqs receive-message \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/incident-events" \
  --max-number-of-messages 10
```

---

## 14. Exam Tips

### Key Points to Remember
- SNS is push-based, SQS is pull-based
- SNS supports fan-out to multiple subscribers
- SQS provides reliable message queuing with visibility timeout
- FIFO queues ensure exactly-once delivery and ordering
- Dead letter queues handle failed message processing
- Both services support encryption and access control

### Common Mistakes
- Not configuring proper queue policies for SNS integration
- Forgetting to delete messages after processing in SQS
- Not implementing proper error handling and retry logic
- Overlooking message filtering capabilities in SNS
- Not monitoring queue metrics and setting up alarms

### Best Practices for Exam
- Understand push vs pull messaging patterns
- Know when to use standard vs FIFO queues
- Understand SNS + SQS integration patterns
- Know error handling and DLQ configurations
- Understand security and encryption options
- Know monitoring and alerting capabilities