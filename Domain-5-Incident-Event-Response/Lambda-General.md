# AWS Lambda - DOP-C02 Study Notes

## 1. Overview

### What is Lambda?
- Serverless compute service for running code without managing servers
- Event-driven execution model
- Automatic scaling and high availability
- Pay-per-request pricing model

### Key Benefits
- **No Server Management** - Focus on code, not infrastructure
- **Automatic Scaling** - Scales from zero to thousands of concurrent executions
- **Built-in Fault Tolerance** - Automatic failure handling and retry
- **Cost Effective** - Pay only for compute time used
- **Event Integration** - Native integration with AWS services

### Use Cases for Incident Response
- Automated incident response workflows
- Real-time log processing and analysis
- Alert processing and notification routing
- Infrastructure remediation actions
- Security incident automation

---

## 2. Lambda Function Basics

### Function Configuration
```python
import json
import boto3
import logging

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Main Lambda function handler"""
    
    try:
        # Log incoming event
        logger.info(f"Received event: {json.dumps(event)}")
        
        # Process event
        result = process_incident_event(event)
        
        # Return response
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
        
    except Exception as e:
        logger.error(f"Error processing event: {str(e)}")
        raise e

def process_incident_event(event):
    """Process incident-related event"""
    
    event_source = event.get('source', '')
    
    if event_source == 'aws.cloudwatch':
        return handle_cloudwatch_alarm(event)
    elif event_source == 'aws.guardduty':
        return handle_security_finding(event)
    elif event_source == 'aws.ec2':
        return handle_ec2_event(event)
    else:
        return handle_generic_event(event)
```

### Environment Variables
```bash
# Set environment variables
aws lambda update-function-configuration \
  --function-name IncidentResponseFunction \
  --environment Variables='{
    "SNS_TOPIC_ARN":"arn:aws:sns:us-east-1:123456789012:incident-alerts",
    "SLACK_WEBHOOK_URL":"https://hooks.slack.com/services/...",
    "LOG_LEVEL":"INFO"
  }'
```

---

## 3. Event Sources and Triggers

### CloudWatch Events/EventBridge
```python
def handle_cloudwatch_alarm(event):
    """Handle CloudWatch alarm events"""
    
    detail = event['detail']
    alarm_name = detail['alarmName']
    new_state = detail['newState']['value']
    reason = detail['newState']['reason']
    
    if new_state == 'ALARM':
        # Create incident
        incident = create_incident(
            title=f"CloudWatch Alarm: {alarm_name}",
            description=reason,
            severity=determine_severity(alarm_name),
            source='cloudwatch'
        )
        
        # Send notifications
        send_incident_notification(incident)
        
        # Trigger automated response
        trigger_automated_response(incident)
        
        return incident
    
    elif new_state == 'OK':
        # Resolve incident if exists
        resolve_incident_by_alarm(alarm_name)
        
    return {'status': 'processed'}
```

### S3 Events
```python
def handle_s3_event(event, context):
    """Process S3 events for log analysis"""
    
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        if key.endswith('.log'):
            # Process log file for incidents
            analyze_log_file(bucket, key)

def analyze_log_file(bucket, key):
    """Analyze log file for incident patterns"""
    
    s3 = boto3.client('s3')
    
    # Download log file
    response = s3.get_object(Bucket=bucket, Key=key)
    log_content = response['Body'].read().decode('utf-8')
    
    # Analyze for error patterns
    error_patterns = [
        r'ERROR.*database connection failed',
        r'CRITICAL.*out of memory',
        r'FATAL.*service unavailable'
    ]
    
    incidents = []
    for pattern in error_patterns:
        matches = re.findall(pattern, log_content, re.IGNORECASE)
        if matches:
            incident = create_incident_from_log_pattern(pattern, matches, bucket, key)
            incidents.append(incident)
    
    return incidents
```

### SNS Integration
```python
def handle_sns_message(event, context):
    """Process SNS messages for incident escalation"""
    
    for record in event['Records']:
        message = json.loads(record['Sns']['Message'])
        
        # Check if this is an incident escalation
        if message.get('type') == 'incident_escalation':
            escalate_incident(message['incident_id'])
        
        # Check if this is a security alert
        elif message.get('type') == 'security_alert':
            handle_security_alert(message)

def escalate_incident(incident_id):
    """Escalate incident to higher tier support"""
    
    # Get incident details
    incident = get_incident_details(incident_id)
    
    # Update incident severity
    incident['severity'] = 'HIGH'
    incident['escalated'] = True
    incident['escalation_time'] = datetime.utcnow().isoformat()
    
    # Notify escalation team
    send_escalation_notification(incident)
    
    # Update incident in database
    update_incident(incident)
```

---

## 4. Automated Incident Response

### EC2 Instance Recovery
```python
def handle_ec2_incident(event):
    """Automated EC2 instance incident response"""
    
    instance_id = event['detail']['instance-id']
    state = event['detail']['state']
    
    if state == 'stopped' and is_unexpected_stop(event):
        # Investigate the stop
        investigation_result = investigate_instance_stop(instance_id)
        
        if investigation_result['can_restart']:
            # Attempt automatic restart
            restart_instance(instance_id)
            
            # Monitor restart success
            monitor_instance_restart(instance_id)
        else:
            # Create incident for manual intervention
            create_manual_intervention_incident(instance_id, investigation_result)

def investigate_instance_stop(instance_id):
    """Investigate why instance stopped"""
    
    ec2 = boto3.client('ec2')
    cloudtrail = boto3.client('cloudtrail')
    
    # Get instance details
    response = ec2.describe_instances(InstanceIds=[instance_id])
    instance = response['Reservations'][0]['Instances'][0]
    
    # Check CloudTrail for stop events
    events = cloudtrail.lookup_events(
        LookupAttributes=[
            {
                'AttributeKey': 'ResourceName',
                'AttributeValue': instance_id
            }
        ],
        StartTime=datetime.utcnow() - timedelta(hours=1)
    )
    
    # Analyze stop reason
    stop_reason = instance.get('StateReason', {}).get('Message', '')
    
    # Determine if safe to restart
    can_restart = (
        'user initiated' not in stop_reason.lower() and
        'spot interruption' not in stop_reason.lower() and
        len([e for e in events['Events'] if e['EventName'] == 'StopInstances']) == 0
    )
    
    return {
        'can_restart': can_restart,
        'stop_reason': stop_reason,
        'recent_events': events['Events']
    }

def restart_instance(instance_id):
    """Restart EC2 instance"""
    
    ec2 = boto3.client('ec2')
    
    try:
        response = ec2.start_instances(InstanceIds=[instance_id])
        logger.info(f"Started instance {instance_id}")
        return response
    except Exception as e:
        logger.error(f"Failed to start instance {instance_id}: {str(e)}")
        raise e
```

### Security Incident Automation
```python
def handle_security_incident(event):
    """Automated security incident response"""
    
    finding = event['detail']
    severity = finding['severity']
    
    if severity >= 7.0:  # High/Critical
        # Immediate isolation
        isolate_compromised_resources(finding)
        
        # Create high-priority incident
        incident = create_security_incident(finding, 'HIGH')
        
        # Notify security team immediately
        notify_security_team(incident, urgent=True)
        
        # Start forensic data collection
        start_forensic_collection(finding)
    
    elif severity >= 4.0:  # Medium
        # Log and investigate
        incident = create_security_incident(finding, 'MEDIUM')
        
        # Apply automated remediation if available
        apply_security_remediation(finding)
        
        # Notify security team
        notify_security_team(incident)

def isolate_compromised_resources(finding):
    """Isolate compromised resources"""
    
    ec2 = boto3.client('ec2')
    
    # Get affected resources
    for resource in finding['service']['resourceRole'] == 'TARGET':
        if resource['resourceType'] == 'Instance':
            instance_id = resource['instanceDetails']['instanceId']
            
            # Create isolation security group
            isolation_sg = create_isolation_security_group()
            
            # Apply isolation
            ec2.modify_instance_attribute(
                InstanceId=instance_id,
                Groups=[isolation_sg]
            )
            
            logger.info(f"Isolated instance {instance_id}")

def create_isolation_security_group():
    """Create security group for isolation"""
    
    ec2 = boto3.client('ec2')
    
    # Create security group with no inbound rules
    response = ec2.create_security_group(
        GroupName=f'isolation-{int(time.time())}',
        Description='Isolation security group for compromised resources'
    )
    
    return response['GroupId']
```

---

## 5. Error Handling and Resilience

### Retry Logic and Dead Letter Queues
```python
def lambda_handler_with_retry(event, context):
    """Lambda handler with built-in retry logic"""
    
    max_retries = 3
    retry_count = 0
    
    while retry_count < max_retries:
        try:
            return process_event(event)
            
        except RetryableError as e:
            retry_count += 1
            logger.warning(f"Retryable error (attempt {retry_count}): {str(e)}")
            
            if retry_count >= max_retries:
                # Send to DLQ
                send_to_dlq(event, str(e))
                raise e
            
            # Exponential backoff
            time.sleep(2 ** retry_count)
            
        except FatalError as e:
            logger.error(f"Fatal error: {str(e)}")
            send_to_dlq(event, str(e))
            raise e

def send_to_dlq(event, error_message):
    """Send failed event to dead letter queue"""
    
    sqs = boto3.client('sqs')
    
    dlq_message = {
        'original_event': event,
        'error_message': error_message,
        'timestamp': datetime.utcnow().isoformat(),
        'function_name': context.function_name
    }
    
    sqs.send_message(
        QueueUrl=os.environ['DLQ_URL'],
        MessageBody=json.dumps(dlq_message)
    )
```

### Circuit Breaker Pattern
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'
    
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

# Global circuit breaker instance
external_api_breaker = CircuitBreaker()

def call_external_api_safely(data):
    """Call external API with circuit breaker protection"""
    return external_api_breaker.call(call_external_api, data)
```

---

## 6. Monitoring and Observability

### CloudWatch Metrics and Alarms
```python
def publish_custom_metrics(function_name, incident_count, response_time):
    """Publish custom metrics for incident response"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_data(
        Namespace='IncidentResponse/Lambda',
        MetricData=[
            {
                'MetricName': 'IncidentsProcessed',
                'Dimensions': [
                    {
                        'Name': 'FunctionName',
                        'Value': function_name
                    }
                ],
                'Value': incident_count,
                'Unit': 'Count'
            },
            {
                'MetricName': 'ResponseTime',
                'Dimensions': [
                    {
                        'Name': 'FunctionName',
                        'Value': function_name
                    }
                ],
                'Value': response_time,
                'Unit': 'Milliseconds'
            }
        ]
    )
```

### X-Ray Tracing
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch AWS SDK calls
patch_all()

@xray_recorder.capture('incident_response')
def lambda_handler(event, context):
    """Lambda handler with X-Ray tracing"""
    
    # Add metadata to trace
    xray_recorder.put_metadata('event_source', event.get('source'))
    xray_recorder.put_annotation('incident_type', event.get('detail-type'))
    
    with xray_recorder.in_subsegment('process_incident'):
        result = process_incident(event)
    
    return result

@xray_recorder.capture('external_api_call')
def call_external_service(data):
    """External service call with tracing"""
    
    # This will be traced automatically
    response = requests.post('https://api.example.com/incidents', json=data)
    
    # Add response metadata
    xray_recorder.put_metadata('response_status', response.status_code)
    
    return response.json()
```

---

## 7. Performance Optimization

### Provisioned Concurrency
```bash
# Configure provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name IncidentResponseFunction \
  --qualifier '$LATEST' \
  --provisioned-concurrency-units 10
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

# Reuse clients outside handler
dynamodb = boto3.resource('dynamodb', config=config)
sns = boto3.client('sns', config=config)
ec2 = boto3.client('ec2', config=config)

def lambda_handler(event, context):
    """Optimized handler with connection reuse"""
    
    # Use pre-initialized clients
    table = dynamodb.Table('incidents')
    
    # Process event efficiently
    return process_incident_optimized(event, table, sns, ec2)
```

### Memory and Timeout Optimization
```bash
# Optimize function configuration
aws lambda update-function-configuration \
  --function-name IncidentResponseFunction \
  --memory-size 1024 \
  --timeout 300 \
  --reserved-concurrent-executions 100
```

---

## 8. Security Best Practices

### IAM Roles and Policies
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:ModifyInstanceAttribute"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": ["Production", "Staging"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:*:*:incident-*"
    }
  ]
}
```

### Environment Variable Encryption
```bash
# Encrypt environment variables with KMS
aws lambda update-function-configuration \
  --function-name IncidentResponseFunction \
  --kms-key-arn "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012" \
  --environment Variables='{
    "ENCRYPTED_API_KEY":"AQICAHi...",
    "DATABASE_PASSWORD":"AQICAHi..."
  }'
```

---

## 9. Common Exam Scenarios

### Scenario 1: Automated CloudWatch alarm response
**Solution:**
- Create Lambda function triggered by CloudWatch Events
- Implement alarm evaluation and incident creation logic
- Add automated remediation actions based on alarm type
- Configure proper IAM permissions and error handling

### Scenario 2: Real-time log analysis for incidents
**Solution:**
- Use Lambda with S3 or Kinesis triggers for log processing
- Implement pattern matching for incident detection
- Create incidents automatically for critical patterns
- Send notifications and trigger response workflows

### Scenario 3: Security incident automation
**Solution:**
- Create Lambda function for GuardDuty findings
- Implement resource isolation for high-severity findings
- Automate forensic data collection
- Integrate with incident management systems

### Scenario 4: Cross-service incident correlation
**Solution:**
- Use Lambda to process events from multiple sources
- Implement correlation logic to group related events
- Create unified incidents from correlated events
- Reduce alert fatigue through intelligent grouping

### Scenario 5: Incident escalation automation
**Solution:**
- Create Lambda function for time-based escalation
- Monitor incident age and severity
- Automatically escalate unresolved incidents
- Notify appropriate teams based on escalation rules

---

## 10. CLI Commands Reference

### Function Management
```bash
# Create function
aws lambda create-function \
  --function-name IncidentResponseFunction \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

# Update function code
aws lambda update-function-code \
  --function-name IncidentResponseFunction \
  --zip-file fileb://function.zip

# Invoke function
aws lambda invoke \
  --function-name IncidentResponseFunction \
  --payload '{"test": "data"}' \
  response.json
```

### Event Source Mapping
```bash
# Create event source mapping for SQS
aws lambda create-event-source-mapping \
  --function-name IncidentResponseFunction \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:incident-queue \
  --batch-size 10

# Create event source mapping for Kinesis
aws lambda create-event-source-mapping \
  --function-name LogAnalysisFunction \
  --event-source-arn arn:aws:kinesis:us-east-1:123456789012:stream/log-stream \
  --starting-position LATEST
```

---

## 11. Exam Tips

### Key Points to Remember
- Lambda is event-driven and serverless
- Automatic scaling from zero to thousands of concurrent executions
- 15-minute maximum execution time
- Built-in integration with AWS services
- Pay-per-request pricing model
- Supports multiple programming languages

### Common Mistakes
- Not implementing proper error handling and retry logic
- Forgetting to configure dead letter queues for failed executions
- Not optimizing memory and timeout settings
- Overlooking IAM permission requirements
- Not monitoring function performance and errors

### Best Practices for Exam
- Understand event sources and triggers
- Know error handling and resilience patterns
- Understand performance optimization techniques
- Know security best practices and IAM integration
- Understand monitoring and observability options
- Know cost optimization strategies