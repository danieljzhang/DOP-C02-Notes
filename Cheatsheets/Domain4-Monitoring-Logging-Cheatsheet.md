# Domain 4: Monitoring and Logging - DOP-C02 Cheatsheet

## Weight: 15% | Focus: CloudWatch, X-Ray, CloudTrail, Config

---

## 🔥 **MUST KNOW Services**

### **CloudWatch** - Metrics & Monitoring
- **Metrics**: Standard (5min), Detailed (1min), Custom
- **Alarms**: Metric, Composite, Anomaly Detection
- **Logs**: Centralized logging, retention policies
- **Dashboards**: Visual monitoring interface

### **X-Ray** - Distributed Tracing
- **Traces**: End-to-end request tracking
- **Segments**: Individual service calls
- **Service Map**: Visual application topology
- **Sampling**: Control trace collection rate

### **CloudTrail** - API Auditing
- **Management Events**: Control plane operations (free)
- **Data Events**: Data plane operations (charged)
- **Insights**: Unusual activity detection
- **Multi-Region**: Global event logging

### **Config** - Configuration Monitoring
- **Configuration Items**: Resource snapshots
- **Rules**: Compliance evaluation
- **Conformance Packs**: Rule collections
- **Remediation**: Automatic compliance fixes

---

## ⚡ **Key Patterns**

### **Comprehensive Monitoring Stack**
```
Application → X-Ray (Tracing) → CloudWatch (Metrics/Logs) → CloudTrail (Audit)
```

### **Log Aggregation**
```
EC2/Lambda/ECS → CloudWatch Logs → Log Groups → Metric Filters → Alarms
```

### **Compliance Monitoring**
```
Resources → Config Rules → Non-Compliance → Remediation Actions
```

---

## 🎯 **Exam Scenarios**

### **Scenario 1: Application performance issues**
- Enable X-Ray tracing to identify bottlenecks
- Create CloudWatch custom metrics for business KPIs
- Set up alarms for response time and error rates
- Use CloudWatch Insights for log analysis

### **Scenario 2: Security incident investigation**
- Check CloudTrail for API calls and user activity
- Review VPC Flow Logs for network traffic
- Use Config to track resource configuration changes
- Correlate events across multiple log sources

### **Scenario 3: Cost optimization monitoring**
- Set up billing alarms in CloudWatch
- Use Cost Explorer with custom metrics
- Monitor resource utilization metrics
- Create dashboards for cost tracking

---

## 📋 **Quick Commands**

```bash
# CloudWatch
aws cloudwatch put-metric-data --namespace "MyApp" --metric-data MetricName=Errors,Value=1
aws cloudwatch put-metric-alarm --alarm-name HighCPU --metric-name CPUUtilization
aws logs create-log-group --log-group-name /aws/lambda/my-function

# X-Ray
aws xray get-trace-summaries --time-range-type TimeRangeByStartTime
aws xray get-service-graph --start-time 2024-01-01T00:00:00 --end-time 2024-01-01T23:59:59

# CloudTrail
aws cloudtrail create-trail --name MyTrail --s3-bucket-name my-trail-bucket
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser

# Config
aws configservice put-config-rule --config-rule file://rule.json
aws configservice get-compliance-details-by-config-rule --config-rule-name required-tags
```

---

## 📊 **CloudWatch Key Metrics**

### **EC2**
- CPUUtilization, NetworkIn/Out, DiskReadOps/WriteOps
- StatusCheckFailed (Instance/System)

### **RDS**
- CPUUtilization, DatabaseConnections, FreeableMemory
- ReadLatency, WriteLatency, ReplicaLag

### **Lambda**
- Duration, Errors, Throttles, ConcurrentExecutions
- DeadLetterErrors, IteratorAge (for streams)

### **ALB**
- RequestCount, TargetResponseTime, HTTPCode_Target_2XX_Count
- HealthyHostCount, UnHealthyHostCount

---

## 🔍 **X-Ray Integration**

### **Supported Services**
- Lambda (automatic), EC2 (daemon), ECS (sidecar)
- API Gateway, ALB, SNS, SQS
- RDS, DynamoDB (via SDK)

### **Sampling Rules**
```json
{
  "version": 2,
  "default": {
    "fixed_target": 1,
    "rate": 0.1
  },
  "rules": [
    {
      "description": "High priority service",
      "service_name": "critical-service",
      "http_method": "*",
      "url_path": "*",
      "fixed_target": 2,
      "rate": 0.5
    }
  ]
}
```

---

## 📋 **CloudTrail Event Types**

### **Management Events** (Free)
- IAM actions, EC2 RunInstances, S3 CreateBucket
- VPC operations, Route 53 changes

### **Data Events** (Charged)
- S3 object operations (GetObject, PutObject)
- Lambda function invocations
- DynamoDB item operations

### **Insight Events**
- Unusual patterns in management events
- Machine learning-based detection

---

## ⚠️ **Common Mistakes**
- Not enabling detailed monitoring when needed
- Forgetting to set up log retention policies
- Not configuring proper IAM permissions for cross-account logging
- Missing X-Ray daemon configuration
- Not setting up CloudTrail in all regions
- Ignoring CloudWatch costs for high-volume logs

---

## 🔑 **Key Points**
- **CloudWatch Logs** retention is forever by default (set retention!)
- **X-Ray** requires SDK instrumentation or daemon
- **CloudTrail** logs to S3, can integrate with CloudWatch Logs
- **Config** requires delivery channel and recorder setup
- **Custom metrics** can have up to 10 dimensions
- **Composite alarms** combine multiple alarm states