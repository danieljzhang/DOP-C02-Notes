# Domain 5: Incident and Event Response - DOP-C02 Cheatsheet

## Weight: 18% | Focus: EventBridge, Lambda, SNS/SQS, Step Functions, CloudWatch Alarms

---

## 🔥 **MUST KNOW Services**

### **EventBridge** - Event Routing
- **Event Buses**: Default, Custom, Partner
- **Rules**: Event pattern matching and routing
- **Targets**: Lambda, SNS, SQS, Step Functions, etc.
- **Schema Registry**: Event structure management

### **Lambda** - Serverless Compute
- **Triggers**: 200+ event sources
- **Concurrency**: 1000 default, reserved/provisioned
- **Error Handling**: DLQ, retry, exponential backoff
- **Monitoring**: CloudWatch Logs, X-Ray tracing

### **SNS/SQS** - Messaging
- **SNS**: Pub/Sub, fan-out, mobile push
- **SQS**: Queue, FIFO, DLQ, visibility timeout
- **Integration**: SNS → SQS for reliable processing

### **Step Functions** - Workflow Orchestration
- **State Types**: Task, Choice, Parallel, Wait, Pass, Fail
- **Workflows**: Standard (long-running), Express (high-volume)
- **Error Handling**: Retry, Catch, Fallback

### **CloudWatch Alarms** - Threshold Monitoring
- **States**: OK, ALARM, INSUFFICIENT_DATA
- **Types**: Metric, Composite, Anomaly Detection
- **Actions**: SNS, Auto Scaling, EC2, Lambda

---

## ⚡ **Key Patterns**

### **Event-Driven Incident Response**
```
CloudWatch Alarm → EventBridge → Lambda → [SNS + Step Functions + SQS]
```

### **Fan-Out Processing**
```
EventBridge → SNS Topic → Multiple SQS Queues → Lambda Functions
```

### **Automated Remediation**
```
GuardDuty Finding → EventBridge → Step Functions → [Isolate + Notify + Investigate]
```

---

## 🎯 **Exam Scenarios**

### **Scenario 1: Automated incident response**
- CloudWatch Alarm triggers EventBridge rule
- EventBridge routes to Lambda for initial response
- Lambda creates Step Functions workflow for complex remediation
- SNS sends notifications to operations team

### **Scenario 2: High-volume event processing**
- Use Express Step Functions for high throughput
- Implement SQS with batch processing
- Configure DLQ for failed events
- Monitor with CloudWatch metrics

### **Scenario 3: Cross-account event handling**
- Set up cross-account EventBridge permissions
- Use IAM roles for service integration
- Implement proper error handling and monitoring
- Ensure event delivery and processing

---

## 📋 **Quick Commands**

```bash
# EventBridge
aws events put-rule --name MyRule --event-pattern file://pattern.json
aws events put-targets --rule MyRule --targets Id=1,Arn=arn:aws:lambda:us-east-1:123456789012:function:MyFunction

# Lambda
aws lambda create-function --function-name MyFunction --runtime python3.9 --role MyRole --handler index.handler
aws lambda invoke --function-name MyFunction --payload '{"key":"value"}' response.json

# SNS/SQS
aws sns create-topic --name MyTopic
aws sqs create-queue --queue-name MyQueue
aws sns subscribe --topic-arn MyTopicArn --protocol sqs --notification-endpoint MyQueueArn

# Step Functions
aws stepfunctions create-state-machine --name MyStateMachine --definition file://definition.json --role-arn MyRole
aws stepfunctions start-execution --state-machine-arn MyStateMachineArn --input '{"key":"value"}'

# CloudWatch Alarms
aws cloudwatch put-metric-alarm --alarm-name HighCPU --metric-name CPUUtilization --threshold 80
```

---

## 🚨 **CloudWatch Alarm Actions**

### **Supported Actions**
- **SNS**: Send notifications
- **Auto Scaling**: Scale EC2 instances
- **EC2**: Stop, terminate, reboot, recover
- **Systems Manager**: Execute automation

### **Alarm States**
- **OK**: Metric within threshold
- **ALARM**: Metric breached threshold
- **INSUFFICIENT_DATA**: Not enough data points

---

## 🔄 **Step Functions State Types**

### **Task**: Execute work (Lambda, ECS, SNS, etc.)
### **Choice**: Conditional branching
### **Parallel**: Execute branches concurrently
### **Wait**: Delay execution
### **Pass**: Pass input to output
### **Fail**: Stop execution with failure
### **Succeed**: Stop execution successfully

---

## 📨 **SNS/SQS Integration Patterns**

### **Fan-Out Pattern**
```
SNS Topic → Multiple SQS Queues → Different Lambda Functions
```

### **FIFO Processing**
```
Application → SQS FIFO Queue → Lambda (ordered processing)
```

### **Dead Letter Queue**
```
SQS Queue → Failed Messages → DLQ → Manual Investigation
```

---

## ⚡ **Lambda Best Practices**

### **Performance**
- Use provisioned concurrency for consistent latency
- Optimize memory allocation (CPU scales with memory)
- Reuse connections and clients outside handler
- Use environment variables for configuration

### **Error Handling**
- Configure DLQ for async invocations
- Implement exponential backoff for retries
- Use proper exception handling
- Monitor with CloudWatch and X-Ray

---

## ⚠️ **Common Mistakes**
- Not configuring DLQ for failed Lambda invocations
- Forgetting to set SQS visibility timeout properly
- Not implementing proper error handling in Step Functions
- Missing IAM permissions for cross-service integration
- Not monitoring EventBridge rule metrics
- Hardcoding values instead of using environment variables

---

## 🔑 **Key Points**
- **EventBridge** is regional, use replication for multi-region
- **Lambda** has 15-minute max execution time
- **SQS** visibility timeout should be 6x Lambda timeout
- **Step Functions** Standard vs Express (cost vs features)
- **SNS** delivers at-least-once, SQS FIFO exactly-once
- **CloudWatch Alarms** evaluate every period, not continuously