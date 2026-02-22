# AWS CloudWatch Logs - DOP-C02 Exam Notes

## 1. Overview

**AWS CloudWatch Logs** enables you to monitor, store, and access your log files from Amazon EC2 instances, AWS CloudTrail, Route 53, and other sources. It provides centralized log management with real-time monitoring and analysis capabilities.

### Key Characteristics
- **Centralized logging** - Collect logs from multiple sources in one location
- **Real-time monitoring** - Stream logs in real-time for immediate analysis
- **Log retention** - Configurable retention periods from 1 day to never expire
- **Log insights** - Query and analyze log data using CloudWatch Logs Insights
- **Metric filters** - Extract metrics from log data for monitoring and alerting
- **Subscription filters** - Stream log data to other services for processing
- **Cross-account access** - Share logs across AWS accounts
- **Encryption** - Server-side encryption for log data at rest

### What Problem Does It Solve?
- Centralizes log management across distributed systems
- Provides real-time log monitoring and alerting
- Enables log-based troubleshooting and debugging
- Facilitates compliance and audit requirements
- Supports automated log analysis and processing
- Reduces operational overhead of log management

---

## 2. Core Components

### Log Groups and Log Streams
```bash
# Create log group
aws logs create-log-group --log-group-name /aws/lambda/my-function

# Set retention policy
aws logs put-retention-policy \
  --log-group-name /aws/lambda/my-function \
  --retention-in-days 30

# Create log stream
aws logs create-log-stream \
  --log-group-name /aws/lambda/my-function \
  --log-stream-name 2023/12/01/stream1

# Put log events
aws logs put-log-events \
  --log-group-name /aws/lambda/my-function \
  --log-stream-name 2023/12/01/stream1 \
  --log-events timestamp=1640995200000,message="Application started"
```

### CloudFormation Configuration
```yaml
Resources:
  ApplicationLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/application/myapp
      RetentionInDays: 14
      KmsKeyId: !Ref LogsKMSKey

  LambdaLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/aws/lambda/${LambdaFunction}"
      RetentionInDays: 30

  # Custom log group with subscription filter
  CustomLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /custom/application/logs
      RetentionInDays: 7

  LogSubscriptionFilter:
    Type: AWS::Logs::SubscriptionFilter
    Properties:
      LogGroupName: !Ref CustomLogGroup
      FilterPattern: "[timestamp, request_id, level=\"ERROR\", ...]"
      DestinationArn: !GetAtt ProcessingLambda.Arn

  # Metric filter for error counting
  ErrorMetricFilter:
    Type: AWS::Logs::MetricFilter
    Properties:
      LogGroupName: !Ref ApplicationLogGroup
      FilterPattern: "ERROR"
      MetricTransformations:
        - MetricNamespace: MyApp/Errors
          MetricName: ErrorCount
          MetricValue: "1"
          DefaultValue: 0
```

---

## 3. Log Agents and Collection

### CloudWatch Agent Configuration
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/aws/ec2/system",
            "log_stream_name": "{instance_id}/messages",
            "timezone": "UTC",
            "timestamp_format": "%b %d %H:%M:%S"
          },
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/aws/ec2/httpd",
            "log_stream_name": "{instance_id}/access",
            "timezone": "UTC",
            "timestamp_format": "[%d/%b/%Y:%H:%M:%S %z]"
          },
          {
            "file_path": "/opt/myapp/logs/application.log",
            "log_group_name": "/aws/application/myapp",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC",
            "multi_line_start_pattern": "^\\d{4}-\\d{2}-\\d{2}",
            "timestamp_format": "%Y-%m-%d %H:%M:%S"
          }
        ]
      },
      "windows_events": {
        "collect_list": [
          {
            "event_name": "System",
            "event_levels": ["ERROR", "WARNING"],
            "log_group_name": "/aws/ec2/windows/system",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_iowait"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["used_percent"],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

### Container Logging
```yaml
# ECS Task Definition with CloudWatch Logs
Resources:
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: myapp-task
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: 256
      Memory: 512
      ExecutionRoleArn: !GetAtt TaskExecutionRole.Arn
      ContainerDefinitions:
        - Name: myapp
          Image: myapp:latest
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref ApplicationLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs
          Environment:
            - Name: LOG_LEVEL
              Value: INFO

# Kubernetes with Fluent Bit
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.*
        region              us-east-1
        log_group_name      /aws/eks/cluster/logs
        log_stream_prefix   ${HOSTNAME}-
        auto_create_group   true
```

---

## 4. Log Insights and Querying

### CloudWatch Logs Insights Queries
```sql
-- Find errors in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Analyze Lambda function performance
fields @timestamp, @duration, @billedDuration, @memorySize, @maxMemoryUsed
| filter @type = "REPORT"
| stats avg(@duration), max(@duration), min(@duration) by bin(5m)

-- Parse and analyze application logs
fields @timestamp, @message
| parse @message /(?<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) \[(?<level>\w+)\] (?<message>.*)/
| filter level = "ERROR"
| stats count() by bin(1h)

-- Monitor API Gateway access patterns
fields @timestamp, @message
| filter @message like /\[.*\] ".*" \d{3}/
| parse @message /\[(?<timestamp>.*)\] "(?<method>\w+) (?<path>.*) HTTP\/\d\.\d" (?<status>\d{3})/
| stats count() by status, method
| sort count desc

-- Analyze ECS container logs
fields @timestamp, @message, @logStream
| filter @message like /ERROR/
| parse @logStream /ecs\/(?<service>.*?)\/(?<task>.*)/
| stats count() by service
| sort count desc

-- Custom application metrics from logs
fields @timestamp, @message
| parse @message /response_time=(?<response_time>\d+)ms/
| filter ispresent(response_time)
| stats avg(response_time), max(response_time), min(response_time) by bin(5m)
```

### Programmatic Log Analysis
```python
import boto3
import json
from datetime import datetime, timedelta

class CloudWatchLogsAnalyzer:
    def __init__(self):
        self.logs_client = boto3.client('logs')
    
    def run_insights_query(self, log_group, query, start_time, end_time):
        """Run CloudWatch Logs Insights query"""
        
        response = self.logs_client.start_query(
            logGroupName=log_group,
            startTime=int(start_time.timestamp()),
            endTime=int(end_time.timestamp()),
            queryString=query
        )
        
        query_id = response['queryId']
        
        # Wait for query to complete
        while True:
            result = self.logs_client.get_query_results(queryId=query_id)
            
            if result['status'] == 'Complete':
                return result['results']
            elif result['status'] == 'Failed':
                raise Exception(f"Query failed: {result.get('statistics', {}).get('recordsMatched', 0)}")
            
            time.sleep(1)
    
    def analyze_error_patterns(self, log_group, hours_back=24):
        """Analyze error patterns in logs"""
        
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours_back)
        
        query = """
        fields @timestamp, @message
        | filter @message like /ERROR/
        | parse @message /ERROR: (?<error_type>.*?) - (?<error_message>.*)/
        | stats count() by error_type
        | sort count desc
        """
        
        results = self.run_insights_query(log_group, query, start_time, end_time)
        
        error_summary = {}
        for result in results:
            error_type = next((field['value'] for field in result if field['field'] == 'error_type'), 'Unknown')
            count = int(next((field['value'] for field in result if field['field'] == 'count'), 0))
            error_summary[error_type] = count
        
        return error_summary
    
    def get_log_events(self, log_group, log_stream=None, start_time=None, end_time=None, filter_pattern=None):
        """Get log events with optional filtering"""
        
        params = {
            'logGroupName': log_group,
            'startFromHead': True
        }
        
        if log_stream:
            params['logStreamNames'] = [log_stream]
        
        if start_time:
            params['startTime'] = int(start_time.timestamp() * 1000)
        
        if end_time:
            params['endTime'] = int(end_time.timestamp() * 1000)
        
        if filter_pattern:
            params['filterPattern'] = filter_pattern
        
        events = []
        paginator = self.logs_client.get_paginator('filter_log_events')
        
        for page in paginator.paginate(**params):
            events.extend(page['events'])
        
        return events
    
    def create_metric_filter(self, log_group, filter_name, filter_pattern, metric_name, metric_namespace):
        """Create metric filter from log data"""
        
        self.logs_client.put_metric_filter(
            logGroupName=log_group,
            filterName=filter_name,
            filterPattern=filter_pattern,
            metricTransformations=[
                {
                    'metricName': metric_name,
                    'metricNamespace': metric_namespace,
                    'metricValue': '1',
                    'defaultValue': 0
                }
            ]
        )
    
    def export_logs_to_s3(self, log_group, s3_bucket, s3_prefix, start_time, end_time):
        """Export logs to S3 for long-term storage"""
        
        response = self.logs_client.create_export_task(
            logGroupName=log_group,
            fromTime=int(start_time.timestamp() * 1000),
            to=int(end_time.timestamp() * 1000),
            destination=s3_bucket,
            destinationPrefix=s3_prefix
        )
        
        return response['taskId']
```

---

## 5. Subscription Filters and Real-time Processing

### Lambda Processing
```python
# Lambda function for log processing
import json
import gzip
import base64
import boto3

def lambda_handler(event, context):
    """Process CloudWatch Logs via subscription filter"""
    
    # Decode and decompress log data
    compressed_payload = base64.b64decode(event['awslogs']['data'])
    uncompressed_payload = gzip.decompress(compressed_payload)
    log_data = json.loads(uncompressed_payload)
    
    processed_events = []
    
    for log_event in log_data['logEvents']:
        message = log_event['message']
        timestamp = log_event['timestamp']
        
        # Parse log message
        if 'ERROR' in message:
            # Extract error information
            error_info = parse_error_message(message)
            
            # Send to SNS for alerting
            send_error_alert(error_info, timestamp)
            
            # Store in DynamoDB for tracking
            store_error_event(error_info, timestamp)
        
        # Process metrics
        if 'response_time=' in message:
            response_time = extract_response_time(message)
            send_custom_metric('ResponseTime', response_time, timestamp)
        
        processed_events.append({
            'timestamp': timestamp,
            'message': message,
            'processed': True
        })
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Processed {len(processed_events)} log events')
    }

def parse_error_message(message):
    """Parse error message to extract structured information"""
    
    # Example parsing logic
    import re
    
    pattern = r'ERROR: (?P<error_type>.*?) - (?P<error_message>.*?) \[(?P<component>.*?)\]'
    match = re.search(pattern, message)
    
    if match:
        return {
            'error_type': match.group('error_type'),
            'error_message': match.group('error_message'),
            'component': match.group('component')
        }
    
    return {'error_type': 'Unknown', 'error_message': message, 'component': 'Unknown'}

def send_error_alert(error_info, timestamp):
    """Send error alert via SNS"""
    
    sns = boto3.client('sns')
    
    message = f"""
    Error Alert:
    Type: {error_info['error_type']}
    Message: {error_info['error_message']}
    Component: {error_info['component']}
    Timestamp: {timestamp}
    """
    
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:error-alerts',
        Message=message,
        Subject=f"Application Error: {error_info['error_type']}"
    )

def send_custom_metric(metric_name, value, timestamp):
    """Send custom metric to CloudWatch"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_data(
        Namespace='MyApp/Performance',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': float(value),
                'Timestamp': datetime.fromtimestamp(timestamp / 1000),
                'Unit': 'Milliseconds'
            }
        ]
    )
```

### Kinesis Data Streams Integration
```yaml
# CloudFormation for Kinesis integration
Resources:
  LogProcessingStream:
    Type: AWS::Kinesis::Stream
    Properties:
      Name: log-processing-stream
      ShardCount: 2
      RetentionPeriodHours: 24

  LogSubscriptionFilter:
    Type: AWS::Logs::SubscriptionFilter
    Properties:
      LogGroupName: !Ref ApplicationLogGroup
      FilterPattern: "[timestamp, request_id, level, message]"
      DestinationArn: !GetAtt LogProcessingStream.Arn
      RoleArn: !GetAtt LogsRole.Arn

  # Kinesis Analytics for real-time processing
  LogAnalyticsApplication:
    Type: AWS::KinesisAnalytics::Application
    Properties:
      ApplicationName: log-analytics
      ApplicationDescription: Real-time log analysis
      ApplicationCode: |
        CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" (
          timestamp TIMESTAMP,
          level VARCHAR(10),
          message VARCHAR(1000),
          error_count INTEGER
        );
        
        CREATE OR REPLACE PUMP "STREAM_PUMP" AS INSERT INTO "DESTINATION_SQL_STREAM"
        SELECT STREAM
          ROWTIME_TO_TIMESTAMP(ROWTIME) as timestamp,
          level,
          message,
          CASE WHEN level = 'ERROR' THEN 1 ELSE 0 END as error_count
        FROM "SOURCE_SQL_STREAM_001"
        WHERE level IN ('ERROR', 'WARN', 'INFO');
      Inputs:
        - NamePrefix: SOURCE_SQL_STREAM
          KinesisStreamsInput:
            ResourceARN: !GetAtt LogProcessingStream.Arn
            RoleARN: !GetAtt AnalyticsRole.Arn
          InputSchema:
            RecordColumns:
              - Name: timestamp
                SqlType: TIMESTAMP
                Mapping: $.timestamp
              - Name: level
                SqlType: VARCHAR(10)
                Mapping: $.level
              - Name: message
                SqlType: VARCHAR(1000)
                Mapping: $.message
```

---

## 6. Cross-Account Log Sharing

### Cross-Account Access Setup
```yaml
# Destination account (log aggregation account)
Resources:
  LogDestination:
    Type: AWS::Logs::Destination
    Properties:
      DestinationName: central-log-destination
      RoleArn: !GetAtt CrossAccountLogsRole.Arn
      TargetArn: !GetAtt CentralLogGroup.Arn
      DestinationPolicy: !Sub |
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "AllowSourceAccounts",
              "Effect": "Allow",
              "Principal": {
                "AWS": [
                  "arn:aws:iam::111111111111:root",
                  "arn:aws:iam::222222222222:root"
                ]
              },
              "Action": "logs:PutSubscriptionFilter",
              "Resource": "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:destination:central-log-destination"
            }
          ]
        }

  CrossAccountLogsRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: logs.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: LogsDeliveryPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:PutLogEvents
                Resource: !GetAtt CentralLogGroup.Arn

  CentralLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /central/aggregated-logs
      RetentionInDays: 90
```

### Source Account Configuration
```python
# Python script to setup cross-account log streaming
import boto3

def setup_cross_account_logging(source_log_group, destination_arn, filter_pattern=""):
    """Setup cross-account log streaming"""
    
    logs_client = boto3.client('logs')
    
    # Create subscription filter pointing to destination account
    logs_client.put_subscription_filter(
        logGroupName=source_log_group,
        filterName=f"{source_log_group}-cross-account-filter",
        filterPattern=filter_pattern,
        destinationArn=destination_arn
    )
    
    print(f"Cross-account logging setup for {source_log_group}")

# Usage
setup_cross_account_logging(
    source_log_group="/aws/lambda/my-function",
    destination_arn="arn:aws:logs:us-east-1:999999999999:destination:central-log-destination",
    filter_pattern="[timestamp, request_id, level=\"ERROR\", ...]"
)
```

---

## 7. Security and Encryption

### Encryption Configuration
```yaml
Resources:
  LogsKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for CloudWatch Logs encryption
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "kms:*"
            Resource: "*"
          - Sid: Allow CloudWatch Logs
            Effect: Allow
            Principal:
              Service: !Sub "logs.${AWS::Region}.amazonaws.com"
            Action:
              - kms:Encrypt
              - kms:Decrypt
              - kms:ReEncrypt*
              - kms:GenerateDataKey*
              - kms:DescribeKey
            Resource: "*"
            Condition:
              ArnEquals:
                "kms:EncryptionContext:aws:logs:arn": !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/secure/application/*"

  LogsKMSKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: alias/cloudwatch-logs
      TargetKeyId: !Ref LogsKMSKey

  SecureLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /secure/application/logs
      RetentionInDays: 30
      KmsKeyId: !GetAtt LogsKMSKey.Arn
```

### IAM Policies for Log Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogsFullAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": [
        "arn:aws:logs:*:*:log-group:/aws/lambda/*",
        "arn:aws:logs:*:*:log-group:/application/*"
      ]
    },
    {
      "Sid": "CloudWatchLogsInsights",
      "Effect": "Allow",
      "Action": [
        "logs:StartQuery",
        "logs:StopQuery",
        "logs:GetQueryResults",
        "logs:DescribeQueries"
      ],
      "Resource": "*"
    },
    {
      "Sid": "MetricFiltersAccess",
      "Effect": "Allow",
      "Action": [
        "logs:PutMetricFilter",
        "logs:DeleteMetricFilter",
        "logs:DescribeMetricFilters"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:*"
    },
    {
      "Sid": "SubscriptionFiltersAccess",
      "Effect": "Allow",
      "Action": [
        "logs:PutSubscriptionFilter",
        "logs:DeleteSubscriptionFilter",
        "logs:DescribeSubscriptionFilters"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:*",
      "Condition": {
        "StringEquals": {
          "logs:destination-arn": [
            "arn:aws:lambda:*:*:function:log-processor",
            "arn:aws:kinesis:*:*:stream/log-stream"
          ]
        }
      }
    }
  ]
}
```

---

## 8. Integration with CI/CD

### CodePipeline Integration
```yaml
# CloudFormation for CI/CD log monitoring
Resources:
  PipelineLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/codepipeline/deployment-pipeline
      RetentionInDays: 14

  BuildLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/codebuild/build-project
      RetentionInDays: 7

  # Metric filter for deployment failures
  DeploymentFailureMetric:
    Type: AWS::Logs::MetricFilter
    Properties:
      LogGroupName: !Ref PipelineLogGroup
      FilterPattern: "[timestamp, request_id, level=\"FAILED\", stage, ...]"
      MetricTransformations:
        - MetricNamespace: CI/CD/Pipeline
          MetricName: DeploymentFailures
          MetricValue: "1"
          DefaultValue: 0

  # Alarm for deployment failures
  DeploymentFailureAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: DeploymentFailures
      AlarmDescription: Alert on deployment failures
      MetricName: DeploymentFailures
      Namespace: CI/CD/Pipeline
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      AlarmActions:
        - !Ref DeploymentAlertsTopic
```

### Application Deployment Monitoring
```python
# Lambda function for deployment log analysis
import boto3
import json
from datetime import datetime, timedelta

def lambda_handler(event, context):
    """Analyze deployment logs for issues"""
    
    logs_client = boto3.client('logs')
    cloudwatch = boto3.client('cloudwatch')
    
    # Analyze recent deployment logs
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)
    
    # Query for deployment events
    query = """
    fields @timestamp, @message
    | filter @message like /DEPLOYMENT/
    | parse @message /DEPLOYMENT (?<status>\w+): (?<service>.*?) - (?<details>.*)/
    | stats count() by status, service
    """
    
    try:
        response = logs_client.start_query(
            logGroupName='/aws/codepipeline/deployment-pipeline',
            startTime=int(start_time.timestamp()),
            endTime=int(end_time.timestamp()),
            queryString=query
        )
        
        query_id = response['queryId']
        
        # Wait for results
        while True:
            result = logs_client.get_query_results(queryId=query_id)
            
            if result['status'] == 'Complete':
                break
            elif result['status'] == 'Failed':
                raise Exception("Query failed")
            
            time.sleep(1)
        
        # Process results and send metrics
        for row in result['results']:
            status = next((field['value'] for field in row if field['field'] == 'status'), 'Unknown')
            service = next((field['value'] for field in row if field['field'] == 'service'), 'Unknown')
            count = int(next((field['value'] for field in row if field['field'] == 'count'), 0))
            
            # Send custom metrics
            cloudwatch.put_metric_data(
                Namespace='CI/CD/Deployments',
                MetricData=[
                    {
                        'MetricName': f'Deployment{status}',
                        'Dimensions': [
                            {
                                'Name': 'Service',
                                'Value': service
                            }
                        ],
                        'Value': count,
                        'Unit': 'Count',
                        'Timestamp': end_time
                    }
                ]
            )
        
        return {
            'statusCode': 200,
            'body': json.dumps('Deployment analysis completed')
        }
    
    except Exception as e:
        print(f"Error analyzing deployment logs: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps(f'Error: {str(e)}')
        }
```

---

## 9. Common Exam Scenarios

### Scenario 1: Centralized Logging Architecture
```python
# Complete centralized logging setup
import boto3
import json

class CentralizedLoggingSetup:
    def __init__(self):
        self.logs_client = boto3.client('logs')
        self.iam_client = boto3.client('iam')
        self.lambda_client = boto3.client('lambda')
    
    def setup_centralized_logging(self, applications):
        """Setup centralized logging for multiple applications"""
        
        # Create central log groups
        central_groups = {}
        
        for app in applications:
            log_group_name = f"/central/{app['name']}/logs"
            
            try:
                self.logs_client.create_log_group(
                    logGroupName=log_group_name,
                    kmsKeyId=app.get('kms_key_id'),
                    tags={
                        'Application': app['name'],
                        'Environment': app['environment'],
                        'LogType': 'Centralized'
                    }
                )
                
                # Set retention policy
                self.logs_client.put_retention_policy(
                    logGroupName=log_group_name,
                    retentionInDays=app.get('retention_days', 30)
                )
                
                central_groups[app['name']] = log_group_name
                
            except self.logs_client.exceptions.ResourceAlreadyExistsException:
                central_groups[app['name']] = log_group_name
        
        # Setup log aggregation Lambda
        aggregation_function = self.create_log_aggregation_function()
        
        # Create subscription filters
        for app in applications:
            for source_log_group in app['source_log_groups']:
                self.create_subscription_filter(
                    source_log_group,
                    aggregation_function['FunctionArn'],
                    app.get('filter_pattern', '')
                )
        
        return {
            'central_log_groups': central_groups,
            'aggregation_function': aggregation_function['FunctionArn']
        }
    
    def create_log_aggregation_function(self):
        """Create Lambda function for log aggregation"""
        
        function_code = '''
import json
import gzip
import base64
import boto3
import re
from datetime import datetime

def lambda_handler(event, context):
    # Decode log data
    compressed_payload = base64.b64decode(event['awslogs']['data'])
    uncompressed_payload = gzip.decompress(compressed_payload)
    log_data = json.loads(uncompressed_payload)
    
    logs_client = boto3.client('logs')
    
    # Process each log event
    for log_event in log_data['logEvents']:
        # Parse and enrich log message
        enriched_message = enrich_log_message(
            log_event['message'],
            log_data['logGroup'],
            log_data['logStream']
        )
        
        # Send to central log group
        central_log_group = determine_central_log_group(log_data['logGroup'])
        
        logs_client.put_log_events(
            logGroupName=central_log_group,
            logStreamName=f"aggregated-{datetime.now().strftime('%Y/%m/%d')}",
            logEvents=[
                {
                    'timestamp': log_event['timestamp'],
                    'message': enriched_message
                }
            ]
        )
    
    return {'statusCode': 200}

def enrich_log_message(message, log_group, log_stream):
    # Add metadata to log message
    enriched = {
        'original_message': message,
        'source_log_group': log_group,
        'source_log_stream': log_stream,
        'processed_timestamp': datetime.utcnow().isoformat(),
        'metadata': extract_metadata(message)
    }
    
    return json.dumps(enriched)

def extract_metadata(message):
    # Extract structured data from log message
    metadata = {}
    
    # Extract log level
    level_match = re.search(r'\\b(DEBUG|INFO|WARN|ERROR|FATAL)\\b', message)
    if level_match:
        metadata['level'] = level_match.group(1)
    
    # Extract request ID
    request_id_match = re.search(r'request[_-]?id[=:]\\s*([a-zA-Z0-9-]+)', message, re.IGNORECASE)
    if request_id_match:
        metadata['request_id'] = request_id_match.group(1)
    
    return metadata

def determine_central_log_group(source_log_group):
    # Map source log groups to central log groups
    mapping = {
        '/aws/lambda/': '/central/lambda/logs',
        '/aws/apigateway/': '/central/apigateway/logs',
        '/aws/ecs/': '/central/ecs/logs'
    }
    
    for prefix, central_group in mapping.items():
        if source_log_group.startswith(prefix):
            return central_group
    
    return '/central/unknown/logs'
        '''
        
        # Create IAM role for Lambda
        role_response = self.iam_client.create_role(
            RoleName='LogAggregationLambdaRole',
            AssumeRolePolicyDocument=json.dumps({
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {"Service": "lambda.amazonaws.com"},
                        "Action": "sts:AssumeRole"
                    }
                ]
            })
        )
        
        # Attach policies
        self.iam_client.attach_role_policy(
            RoleName='LogAggregationLambdaRole',
            PolicyArn='arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole'
        )
        
        # Create custom policy for logs access
        self.iam_client.put_role_policy(
            RoleName='LogAggregationLambdaRole',
            PolicyName='LogsAccess',
            PolicyDocument=json.dumps({
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Action": [
                            "logs:CreateLogStream",
                            "logs:PutLogEvents",
                            "logs:DescribeLogGroups",
                            "logs:DescribeLogStreams"
                        ],
                        "Resource": "arn:aws:logs:*:*:log-group:/central/*"
                    }
                ]
            })
        )
        
        # Create Lambda function
        function_response = self.lambda_client.create_function(
            FunctionName='log-aggregation-function',
            Runtime='python3.9',
            Role=role_response['Role']['Arn'],
            Handler='index.lambda_handler',
            Code={'ZipFile': function_code.encode()},
            Description='Centralized log aggregation function',
            Timeout=60,
            MemorySize=256
        )
        
        return function_response
```

### Scenario 2: Real-time Log Monitoring and Alerting
```yaml
# Complete real-time monitoring setup
Resources:
  # Application log group
  ApplicationLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/application/myapp
      RetentionInDays: 14

  # Error rate metric filter
  ErrorRateMetricFilter:
    Type: AWS::Logs::MetricFilter
    Properties:
      LogGroupName: !Ref ApplicationLogGroup
      FilterPattern: "ERROR"
      MetricTransformations:
        - MetricNamespace: MyApp/Errors
          MetricName: ErrorRate
          MetricValue: "1"
          DefaultValue: 0

  # Response time metric filter
  ResponseTimeMetricFilter:
    Type: AWS::Logs::MetricFilter
    Properties:
      LogGroupName: !Ref ApplicationLogGroup
      FilterPattern: "[timestamp, request_id, level, message, response_time_ms]"
      MetricTransformations:
        - MetricNamespace: MyApp/Performance
          MetricName: ResponseTime
          MetricValue: "$response_time_ms"
          DefaultValue: 0

  # High error rate alarm
  HighErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: HighErrorRate
      AlarmDescription: Alert when error rate is high
      MetricName: ErrorRate
      Namespace: MyApp/Errors
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref ErrorAlertsTopic
      TreatMissingData: notBreaching

  # Slow response time alarm
  SlowResponseAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: SlowResponseTime
      AlarmDescription: Alert when response time is slow
      MetricName: ResponseTime
      Namespace: MyApp/Performance
      Statistic: Average
      Period: 300
      EvaluationPeriods: 3
      Threshold: 2000
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref PerformanceAlertsTopic

  # Real-time log processing
  LogProcessingFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: real-time-log-processor
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt LogProcessingRole.Arn
      Code:
        ZipFile: |
          import json
          import gzip
          import base64
          import boto3
          import re
          from datetime import datetime
          
          def lambda_handler(event, context):
              # Process real-time log events
              compressed_payload = base64.b64decode(event['awslogs']['data'])
              uncompressed_payload = gzip.decompress(compressed_payload)
              log_data = json.loads(uncompressed_payload)
              
              sns = boto3.client('sns')
              cloudwatch = boto3.client('cloudwatch')
              
              critical_errors = []
              performance_issues = []
              
              for log_event in log_data['logEvents']:
                  message = log_event['message']
                  timestamp = log_event['timestamp']
                  
                  # Detect critical errors
                  if re.search(r'CRITICAL|FATAL|OutOfMemoryError', message, re.IGNORECASE):
                      critical_errors.append({
                          'timestamp': timestamp,
                          'message': message
                      })
                  
                  # Detect performance issues
                  response_time_match = re.search(r'response_time=(\d+)ms', message)
                  if response_time_match:
                      response_time = int(response_time_match.group(1))
                      if response_time > 5000:  # 5 seconds
                          performance_issues.append({
                              'timestamp': timestamp,
                              'response_time': response_time,
                              'message': message
                          })
              
              # Send immediate alerts for critical errors
              if critical_errors:
                  sns.publish(
                      TopicArn='arn:aws:sns:us-east-1:123456789012:critical-alerts',
                      Message=json.dumps(critical_errors, indent=2),
                      Subject=f'CRITICAL: {len(critical_errors)} critical errors detected'
                  )
              
              # Send performance alerts
              if performance_issues:
                  sns.publish(
                      TopicArn='arn:aws:sns:us-east-1:123456789012:performance-alerts',
                      Message=json.dumps(performance_issues, indent=2),
                      Subject=f'Performance Issue: {len(performance_issues)} slow responses detected'
                  )
              
              return {
                  'statusCode': 200,
                  'body': json.dumps({
                      'critical_errors': len(critical_errors),
                      'performance_issues': len(performance_issues)
                  })
              }

  # Subscription filter for real-time processing
  RealTimeSubscriptionFilter:
    Type: AWS::Logs::SubscriptionFilter
    Properties:
      LogGroupName: !Ref ApplicationLogGroup
      FilterPattern: ""
      DestinationArn: !GetAtt LogProcessingFunction.Arn

  # Permission for CloudWatch Logs to invoke Lambda
  LogsLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref LogProcessingFunction
      Action: lambda:InvokeFunction
      Principal: logs.amazonaws.com
      SourceArn: !Sub "${ApplicationLogGroup}:*"
```

---

## 10. Exam Tips

- **Understand log hierarchy** - Log groups contain log streams, streams contain events
- **Master retention policies** - Know default retention (never expire) and cost implications
- **Know metric filters** - Extract metrics from log data for monitoring
- **Practice Logs Insights** - Query syntax and common patterns
- **Understand subscription filters** - Real-time log processing and destinations
- **Learn cross-account sharing** - Log destinations and IAM policies
- **Know encryption options** - KMS integration and key policies
- **Practice troubleshooting** - Common issues with log agents and permissions
- **Understand cost optimization** - Retention policies, log compression, S3 export
- **Master integration patterns** - Lambda, Kinesis, ElasticSearch, S3 integration