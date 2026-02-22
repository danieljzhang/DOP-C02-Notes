# AWS CloudWatch Alarms - DOP-C02 Study Notes

## 1. Overview

### What are CloudWatch Alarms?
- Monitoring service that watches CloudWatch metrics
- Triggers actions when metric thresholds are breached
- Essential component of incident detection and response
- Integrates with SNS, Auto Scaling, EC2, and other services

### Key Components
- **Metrics** - Data points that alarms monitor
- **Thresholds** - Values that trigger alarm state changes
- **Actions** - What happens when alarm state changes
- **States** - OK, ALARM, INSUFFICIENT_DATA

### Use Cases for Incident Response
- Infrastructure monitoring and alerting
- Application performance monitoring
- Security incident detection
- Automated scaling and remediation
- Cost monitoring and optimization

---

## 2. Alarm States and Transitions

### Alarm States
- **OK** - Metric is within threshold
- **ALARM** - Metric has breached threshold
- **INSUFFICIENT_DATA** - Not enough data to determine state

### State Transitions
```python
def monitor_alarm_state_changes(event, context):
    """Handle CloudWatch alarm state changes"""
    
    detail = event['detail']
    alarm_name = detail['alarmName']
    old_state = detail['previousState']['value']
    new_state = detail['newState']['value']
    reason = detail['newState']['reason']
    
    logger.info(f"Alarm {alarm_name} changed from {old_state} to {new_state}")
    
    if new_state == 'ALARM':
        handle_alarm_triggered(alarm_name, reason, detail)
    elif new_state == 'OK' and old_state == 'ALARM':
        handle_alarm_resolved(alarm_name, detail)
    elif new_state == 'INSUFFICIENT_DATA':
        handle_insufficient_data(alarm_name, detail)

def handle_alarm_triggered(alarm_name, reason, detail):
    """Handle alarm entering ALARM state"""
    
    # Create incident
    incident = create_incident(
        title=f"CloudWatch Alarm: {alarm_name}",
        description=reason,
        severity=determine_severity_from_alarm(alarm_name),
        alarm_details=detail
    )
    
    # Send notifications
    send_alarm_notification(incident)
    
    # Trigger automated response
    trigger_automated_response(alarm_name, incident)
```

---

## 3. Metric-Based Alarms

### EC2 Instance Monitoring
```bash
# CPU utilization alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPUUtilization" \
  --alarm-description "Triggers when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:cpu-alerts" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0

# Memory utilization alarm (custom metric)
aws cloudwatch put-metric-alarm \
  --alarm-name "HighMemoryUtilization" \
  --alarm-description "Triggers when memory exceeds 85%" \
  --metric-name MemoryUtilization \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:memory-alerts" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0
```

### Application Load Balancer Monitoring
```bash
# High error rate alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ALBHighErrorRate" \
  --alarm-description "Triggers when 5XX error rate exceeds 5%" \
  --metric-name HTTPCode_ELB_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alb-alerts" \
  --dimensions Name=LoadBalancer,Value=app/my-alb/50dc6c495c0c9188

# High response time alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ALBHighResponseTime" \
  --alarm-description "Triggers when response time exceeds 2 seconds" \
  --metric-name TargetResponseTime \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 300 \
  --threshold 2 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:performance-alerts"
```

### RDS Database Monitoring
```bash
# Database connection alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RDSHighConnections" \
  --alarm-description "Triggers when connection count exceeds 80% of max" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:rds-alerts" \
  --dimensions Name=DBInstanceIdentifier,Value=mydb-instance
```

---

## 4. Composite Alarms

### Multi-Metric Monitoring
```python
def create_composite_alarm():
    """Create composite alarm for application health"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create composite alarm that triggers when multiple conditions are met
    response = cloudwatch.put_composite_alarm(
        AlarmName='ApplicationHealthComposite',
        AlarmDescription='Composite alarm for overall application health',
        AlarmRule=(
            "ALARM('HighCPUUtilization') OR "
            "ALARM('ALBHighErrorRate') OR "
            "ALARM('RDSHighConnections')"
        ),
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:critical-alerts'
        ],
        OKActions=[
            'arn:aws:sns:us-east-1:123456789012:recovery-notifications'
        ]
    )
    
    return response
```

### Complex Logic Alarms
```bash
# Composite alarm with AND/OR logic
aws cloudwatch put-composite-alarm \
  --alarm-name "CriticalSystemFailure" \
  --alarm-description "Critical system failure detected" \
  --alarm-rule "(ALARM('HighCPUUtilization') AND ALARM('ALBHighErrorRate')) OR ALARM('DatabaseDown')" \
  --actions-enabled \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:emergency-alerts"
```

---

## 5. Custom Metrics and Alarms

### Application Metrics
```python
import boto3
import time

def publish_custom_metrics():
    """Publish custom application metrics"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Business metrics
    cloudwatch.put_metric_data(
        Namespace='MyApp/Business',
        MetricData=[
            {
                'MetricName': 'OrdersPerMinute',
                'Value': get_orders_per_minute(),
                'Unit': 'Count/Second',
                'Dimensions': [
                    {
                        'Name': 'Environment',
                        'Value': 'Production'
                    }
                ]
            },
            {
                'MetricName': 'PaymentFailureRate',
                'Value': get_payment_failure_rate(),
                'Unit': 'Percent',
                'Dimensions': [
                    {
                        'Name': 'PaymentProvider',
                        'Value': 'Stripe'
                    }
                ]
            }
        ]
    )

def create_business_metric_alarms():
    """Create alarms for business metrics"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Low order rate alarm
    cloudwatch.put_metric_alarm(
        AlarmName='LowOrderRate',
        AlarmDescription='Order rate has dropped significantly',
        MetricName='OrdersPerMinute',
        Namespace='MyApp/Business',
        Statistic='Average',
        Period=300,
        Threshold=10,
        ComparisonOperator='LessThanThreshold',
        EvaluationPeriods=3,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:business-alerts'
        ],
        Dimensions=[
            {
                'Name': 'Environment',
                'Value': 'Production'
            }
        ]
    )
    
    # High payment failure rate alarm
    cloudwatch.put_metric_alarm(
        AlarmName='HighPaymentFailureRate',
        AlarmDescription='Payment failure rate is too high',
        MetricName='PaymentFailureRate',
        Namespace='MyApp/Business',
        Statistic='Average',
        Period=300,
        Threshold=5,
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:payment-alerts'
        ]
    )
```

---

## 6. Alarm Actions and Automation

### SNS Integration
```python
def setup_alarm_notifications():
    """Set up SNS topics for alarm notifications"""
    
    sns = boto3.client('sns')
    
    # Create topics for different severity levels
    topics = {
        'critical': 'critical-alerts',
        'warning': 'warning-alerts',
        'info': 'info-alerts'
    }
    
    topic_arns = {}
    
    for severity, topic_name in topics.items():
        response = sns.create_topic(Name=topic_name)
        topic_arn = response['TopicArn']
        topic_arns[severity] = topic_arn
        
        # Subscribe appropriate endpoints
        if severity == 'critical':
            # SMS for critical alerts
            sns.subscribe(
                TopicArn=topic_arn,
                Protocol='sms',
                Endpoint='+1234567890'
            )
            # Email for critical alerts
            sns.subscribe(
                TopicArn=topic_arn,
                Protocol='email',
                Endpoint='oncall@example.com'
            )
        
        # Email for all alerts
        sns.subscribe(
            TopicArn=topic_arn,
            Protocol='email',
            Endpoint='alerts@example.com'
        )
    
    return topic_arns
```

### Auto Scaling Integration
```bash
# Create scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "my-asg" \
  --policy-name "scale-up-policy" \
  --policy-type "StepScaling" \
  --adjustment-type "ChangeInCapacity" \
  --step-adjustments MetricIntervalLowerBound=0,MetricIntervalUpperBound=50,ScalingAdjustment=1 \
                     MetricIntervalLowerBound=50,ScalingAdjustment=2

# Create alarm that triggers scaling
aws cloudwatch put-metric-alarm \
  --alarm-name "ScaleUpAlarm" \
  --alarm-description "Scale up when CPU is high" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:autoscaling:us-east-1:123456789012:scalingPolicy:policy-id"
```

### Lambda Integration
```python
def create_lambda_alarm_handler():
    """Create Lambda function to handle alarm events"""
    
    lambda_code = '''
import json
import boto3

def lambda_handler(event, context):
    """Handle CloudWatch alarm events"""
    
    # Parse SNS message
    message = json.loads(event['Records'][0]['Sns']['Message'])
    
    alarm_name = message['AlarmName']
    new_state = message['NewStateValue']
    reason = message['NewStateReason']
    
    if new_state == 'ALARM':
        # Trigger incident response
        trigger_incident_response(alarm_name, reason)
    
    return {'statusCode': 200}

def trigger_incident_response(alarm_name, reason):
    """Trigger automated incident response"""
    
    if 'HighCPU' in alarm_name:
        # Scale up resources
        trigger_auto_scaling()
    elif 'HighErrorRate' in alarm_name:
        # Restart application
        restart_application()
    elif 'DatabaseDown' in alarm_name:
        # Failover to backup
        trigger_database_failover()
'''
    
    # Create Lambda function
    lambda_client = boto3.client('lambda')
    
    response = lambda_client.create_function(
        FunctionName='AlarmResponseHandler',
        Runtime='python3.9',
        Role='arn:aws:iam::123456789012:role/LambdaExecutionRole',
        Handler='lambda_function.lambda_handler',
        Code={'ZipFile': lambda_code.encode()},
        Description='Handles CloudWatch alarm events'
    )
    
    return response
```

---

## 7. Anomaly Detection

### Anomaly Detector Alarms
```python
def create_anomaly_detection_alarm():
    """Create alarm based on anomaly detection"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create anomaly detector
    cloudwatch.put_anomaly_detector(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[
            {
                'Name': 'InstanceId',
                'Value': 'i-1234567890abcdef0'
            }
        ],
        Stat='Average'
    )
    
    # Create alarm based on anomaly detection
    cloudwatch.put_metric_alarm(
        AlarmName='CPUAnomalyDetection',
        AlarmDescription='Detects CPU usage anomalies',
        ComparisonOperator='LessThanLowerOrGreaterThanUpperThreshold',
        EvaluationPeriods=2,
        Metrics=[
            {
                'Id': 'm1',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/EC2',
                        'MetricName': 'CPUUtilization',
                        'Dimensions': [
                            {
                                'Name': 'InstanceId',
                                'Value': 'i-1234567890abcdef0'
                            }
                        ]
                    },
                    'Period': 300,
                    'Stat': 'Average'
                }
            },
            {
                'Id': 'ad1',
                'Expression': 'ANOMALY_DETECTION_FUNCTION(m1, 2)'
            }
        ],
        ThresholdMetricId='ad1',
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:anomaly-alerts'
        ]
    )
```

---

## 8. Monitoring and Troubleshooting

### Alarm History
```python
def analyze_alarm_history(alarm_name, days=7):
    """Analyze alarm history for patterns"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=days)
    
    # Get alarm history
    response = cloudwatch.describe_alarm_history(
        AlarmName=alarm_name,
        StartDate=start_time,
        EndDate=end_time,
        MaxRecords=100
    )
    
    # Analyze patterns
    state_changes = []
    for item in response['AlarmHistoryItems']:
        if item['HistoryItemType'] == 'StateUpdate':
            state_changes.append({
                'timestamp': item['Timestamp'],
                'summary': item['HistorySummary'],
                'data': json.loads(item['HistoryData'])
            })
    
    # Calculate alarm frequency
    alarm_count = len([sc for sc in state_changes 
                      if 'to ALARM' in sc['summary']])
    
    return {
        'alarm_name': alarm_name,
        'period_days': days,
        'total_state_changes': len(state_changes),
        'alarm_count': alarm_count,
        'alarm_frequency': alarm_count / days,
        'state_changes': state_changes
    }
```

### Alarm Effectiveness Metrics
```python
def calculate_alarm_effectiveness():
    """Calculate metrics for alarm effectiveness"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Get all alarms
    alarms = cloudwatch.describe_alarms()
    
    effectiveness_metrics = {}
    
    for alarm in alarms['MetricAlarms']:
        alarm_name = alarm['AlarmName']
        
        # Get alarm history
        history = analyze_alarm_history(alarm_name, days=30)
        
        # Calculate effectiveness metrics
        effectiveness_metrics[alarm_name] = {
            'total_alarms': history['alarm_count'],
            'false_positive_rate': calculate_false_positive_rate(alarm_name),
            'mean_time_to_resolution': calculate_mttr(alarm_name),
            'alarm_frequency': history['alarm_frequency']
        }
    
    return effectiveness_metrics

def calculate_false_positive_rate(alarm_name):
    """Calculate false positive rate for alarm"""
    # Implementation would analyze incident outcomes
    # vs alarm triggers to determine false positives
    return 0.15  # Example: 15% false positive rate

def calculate_mttr(alarm_name):
    """Calculate mean time to resolution"""
    # Implementation would analyze time between
    # alarm trigger and resolution
    return 45  # Example: 45 minutes average
```

---

## 9. Cost Optimization

### Alarm Cost Management
```python
def optimize_alarm_costs():
    """Optimize CloudWatch alarm costs"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Get all alarms
    alarms = cloudwatch.describe_alarms()
    
    optimization_recommendations = []
    
    for alarm in alarms['MetricAlarms']:
        alarm_name = alarm['AlarmName']
        
        # Check if alarm has triggered recently
        history = analyze_alarm_history(alarm_name, days=90)
        
        if history['alarm_count'] == 0:
            # Alarm hasn't triggered in 90 days
            optimization_recommendations.append({
                'alarm_name': alarm_name,
                'recommendation': 'Consider disabling - no triggers in 90 days',
                'potential_savings': 0.10  # $0.10 per alarm per month
            })
        
        elif history['false_positive_rate'] > 0.8:
            # High false positive rate
            optimization_recommendations.append({
                'alarm_name': alarm_name,
                'recommendation': 'Tune threshold - high false positive rate',
                'false_positive_rate': history['false_positive_rate']
            })
    
    return optimization_recommendations
```

---

## 10. Common Exam Scenarios

### Scenario 1: High CPU utilization incident response
**Solution:**
- Create CloudWatch alarm for CPU > 80%
- Configure SNS notification to operations team
- Set up Auto Scaling policy triggered by alarm
- Implement Lambda function for additional automation

### Scenario 2: Application error rate monitoring
**Solution:**
- Create custom metric for application error rate
- Set up alarm when error rate exceeds threshold
- Configure immediate notification for critical errors
- Implement automated rollback for high error rates

### Scenario 3: Database performance monitoring
**Solution:**
- Create alarms for database connection count, CPU, and I/O
- Use composite alarm for overall database health
- Configure escalation for prolonged issues
- Implement automated failover for critical failures

### Scenario 4: Cost anomaly detection
**Solution:**
- Set up billing alarms for cost thresholds
- Create anomaly detection for unusual spending patterns
- Configure notifications for budget overruns
- Implement automated resource shutdown for runaway costs

### Scenario 5: Multi-service health monitoring
**Solution:**
- Create composite alarm combining multiple service metrics
- Use alarm actions to trigger incident response workflow
- Configure different notification channels based on severity
- Implement automated remediation for known issues

---

## 11. CLI Commands Reference

### Alarm Management
```bash
# Create metric alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "MyAlarm" \
  --alarm-description "Description" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2

# List alarms
aws cloudwatch describe-alarms

# Get alarm history
aws cloudwatch describe-alarm-history --alarm-name "MyAlarm"

# Delete alarm
aws cloudwatch delete-alarms --alarm-names "MyAlarm"
```

### Composite Alarms
```bash
# Create composite alarm
aws cloudwatch put-composite-alarm \
  --alarm-name "CompositeAlarm" \
  --alarm-rule "ALARM('Alarm1') OR ALARM('Alarm2')" \
  --actions-enabled \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alerts"
```

---

## 12. Exam Tips

### Key Points to Remember
- Alarms have three states: OK, ALARM, INSUFFICIENT_DATA
- Evaluation periods determine how many consecutive periods must breach threshold
- Composite alarms can combine multiple alarms with AND/OR logic
- Anomaly detection uses machine learning to detect unusual patterns
- Alarms can trigger SNS, Auto Scaling, EC2 actions, and Lambda functions

### Common Mistakes
- Setting evaluation periods too low causing false alarms
- Not configuring proper alarm actions for incident response
- Forgetting to set up OK actions for recovery notifications
- Not using composite alarms for complex monitoring scenarios
- Overlooking cost implications of too many alarms

### Best Practices for Exam
- Understand alarm states and transitions
- Know integration patterns with other AWS services
- Understand composite alarm logic and use cases
- Know anomaly detection capabilities and limitations
- Understand cost optimization strategies for alarms
- Know troubleshooting and monitoring patterns