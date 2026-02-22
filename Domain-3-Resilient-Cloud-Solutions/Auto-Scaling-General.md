# AWS Auto Scaling - DOP-C02 Study Notes

## 1. Overview

### What is Auto Scaling?
- Automatically adjusts compute capacity based on demand
- Maintains application availability and performance
- Optimizes costs by scaling resources up and down
- Multiple scaling services across different AWS resources

### Auto Scaling Services
- **EC2 Auto Scaling** - EC2 instances
- **Application Auto Scaling** - ECS, Lambda, DynamoDB, etc.
- **AWS Auto Scaling** - Unified scaling across services
- **Predictive Scaling** - ML-based capacity planning

---

## 2. EC2 Auto Scaling

### Auto Scaling Groups (ASG)
```bash
# Create launch template
aws ec2 create-launch-template \
  --launch-template-name "web-server-template" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.micro",
    "SecurityGroupIds": ["sg-12345678"],
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQp5dW0gaW5zdGFsbCAteSBodHRwZApzZXJ2aWNlIGh0dHBkIHN0YXJ0"
  }'

# Create Auto Scaling group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "web-servers-asg" \
  --launch-template LaunchTemplateName=web-server-template,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 3 \
  --vpc-zone-identifier "subnet-12345678,subnet-87654321" \
  --target-group-arns "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-servers/1234567890123456"
```

### Scaling Policies
```bash
# Target tracking scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-name "cpu-target-tracking" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 300
  }'

# Step scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "web-servers-asg" \
  --policy-name "scale-out-policy" \
  --policy-type "StepScaling" \
  --adjustment-type "ChangeInCapacity" \
  --step-adjustments '[
    {
      "MetricIntervalLowerBound": 0,
      "MetricIntervalUpperBound": 50,
      "ScalingAdjustment": 1
    },
    {
      "MetricIntervalLowerBound": 50,
      "ScalingAdjustment": 2
    }
  ]'
```

---

## 3. Application Auto Scaling

### ECS Service Scaling
```python
import boto3

def setup_ecs_auto_scaling():
    """Configure Auto Scaling for ECS service"""
    
    autoscaling = boto3.client('application-autoscaling')
    
    # Register scalable target
    autoscaling.register_scalable_target(
        ServiceNamespace='ecs',
        ResourceId='service/my-cluster/my-service',
        ScalableDimension='ecs:service:DesiredCount',
        MinCapacity=2,
        MaxCapacity=20
    )
    
    # Create scaling policy
    autoscaling.put_scaling_policy(
        PolicyName='cpu-scaling-policy',
        ServiceNamespace='ecs',
        ResourceId='service/my-cluster/my-service',
        ScalableDimension='ecs:service:DesiredCount',
        PolicyType='TargetTrackingScaling',
        TargetTrackingScalingPolicyConfiguration={
            'TargetValue': 70.0,
            'PredefinedMetricSpecification': {
                'PredefinedMetricType': 'ECSServiceAverageCPUUtilization'
            },
            'ScaleOutCooldown': 300,
            'ScaleInCooldown': 300
        }
    )
```

### DynamoDB Auto Scaling
```bash
# Register DynamoDB table for scaling
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/MyTable" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 100

# Create scaling policy for DynamoDB
aws application-autoscaling put-scaling-policy \
  --policy-name "MyTableReadScalingPolicy" \
  --service-namespace dynamodb \
  --resource-id "table/MyTable" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
    }
  }'
```

---

## 4. Predictive Scaling

### ML-Based Capacity Planning
```python
def enable_predictive_scaling():
    """Enable predictive scaling for ASG"""
    
    autoscaling = boto3.client('autoscaling')
    
    # Create predictive scaling policy
    response = autoscaling.put_scaling_policy(
        AutoScalingGroupName='web-servers-asg',
        PolicyName='predictive-scaling-policy',
        PolicyType='PredictiveScaling',
        PredictiveScalingConfiguration={
            'MetricSpecifications': [
                {
                    'TargetValue': 70.0,
                    'PredefinedMetricPairSpecification': {
                        'PredefinedMetricType': 'ASGCPUUtilization'
                    }
                }
            ],
            'Mode': 'ForecastAndScale',
            'SchedulingBufferTime': 300,
            'MaxCapacityBreachBehavior': 'HonorMaxCapacity',
            'MaxCapacityBuffer': 10
        }
    )
    
    return response
```

---

## 5. Multi-AZ and Cross-Region Resilience

### Multi-AZ Deployment
```bash
# Create ASG across multiple AZs
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "resilient-web-asg" \
  --launch-template LaunchTemplateName=web-template,Version='$Latest' \
  --min-size 3 \
  --max-size 15 \
  --desired-capacity 6 \
  --vpc-zone-identifier "subnet-1a,subnet-1b,subnet-1c" \
  --availability-zones "us-east-1a,us-east-1b,us-east-1c" \
  --health-check-type "ELB" \
  --health-check-grace-period 300
```

### Cross-Region Auto Scaling
```python
def setup_cross_region_scaling():
    """Set up Auto Scaling across multiple regions"""
    
    regions = ['us-east-1', 'us-west-2', 'eu-west-1']
    
    for region in regions:
        autoscaling = boto3.client('autoscaling', region_name=region)
        
        # Create ASG in each region
        autoscaling.create_auto_scaling_group(
            AutoScalingGroupName=f'web-asg-{region}',
            LaunchTemplate={
                'LaunchTemplateName': f'web-template-{region}',
                'Version': '$Latest'
            },
            MinSize=2,
            MaxSize=10,
            DesiredCapacity=3,
            VPCZoneIdentifier=get_subnets_for_region(region),
            HealthCheckType='ELB',
            HealthCheckGracePeriod=300
        )
```

---

## 6. Health Checks and Instance Replacement

### Health Check Configuration
```python
def configure_health_checks():
    """Configure comprehensive health checks"""
    
    autoscaling = boto3.client('autoscaling')
    
    # Update ASG with ELB health checks
    autoscaling.update_auto_scaling_group(
        AutoScalingGroupName='web-servers-asg',
        HealthCheckType='ELB',
        HealthCheckGracePeriod=300
    )
    
    # Create custom health check Lambda
    lambda_code = '''
import boto3
import json

def lambda_handler(event, context):
    """Custom health check for application"""
    
    instance_id = event['instance_id']
    
    # Perform application-specific health checks
    if check_application_health(instance_id):
        return {'healthy': True}
    else:
        # Mark instance unhealthy
        autoscaling = boto3.client('autoscaling')
        autoscaling.set_instance_health(
            InstanceId=instance_id,
            HealthStatus='Unhealthy'
        )
        return {'healthy': False}

def check_application_health(instance_id):
    """Check application-specific health metrics"""
    # Implementation for custom health checks
    return True
'''
```

---

## 7. Lifecycle Hooks

### Instance Lifecycle Management
```bash
# Create lifecycle hook for launch
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name "launch-hook" \
  --auto-scaling-group-name "web-servers-asg" \
  --lifecycle-transition "autoscaling:EC2_INSTANCE_LAUNCHING" \
  --heartbeat-timeout 300 \
  --notification-target-arn "arn:aws:sns:us-east-1:123456789012:asg-notifications" \
  --role-arn "arn:aws:iam::123456789012:role/AutoScalingNotificationRole"

# Create lifecycle hook for termination
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name "terminate-hook" \
  --auto-scaling-group-name "web-servers-asg" \
  --lifecycle-transition "autoscaling:EC2_INSTANCE_TERMINATING" \
  --heartbeat-timeout 300 \
  --default-result "ABANDON"
```

### Lifecycle Hook Processing
```python
def process_lifecycle_hook(event, context):
    """Process Auto Scaling lifecycle hooks"""
    
    message = json.loads(event['Records'][0]['Sns']['Message'])
    
    lifecycle_transition = message['LifecycleTransition']
    instance_id = message['EC2InstanceId']
    lifecycle_action_token = message['LifecycleActionToken']
    auto_scaling_group_name = message['AutoScalingGroupName']
    lifecycle_hook_name = message['LifecycleHookName']
    
    autoscaling = boto3.client('autoscaling')
    
    try:
        if lifecycle_transition == 'autoscaling:EC2_INSTANCE_LAUNCHING':
            # Perform launch initialization
            initialize_instance(instance_id)
            result = 'CONTINUE'
        
        elif lifecycle_transition == 'autoscaling:EC2_INSTANCE_TERMINATING':
            # Perform graceful shutdown
            graceful_shutdown(instance_id)
            result = 'CONTINUE'
        
        # Complete lifecycle action
        autoscaling.complete_lifecycle_action(
            LifecycleHookName=lifecycle_hook_name,
            AutoScalingGroupName=auto_scaling_group_name,
            InstanceId=instance_id,
            LifecycleActionToken=lifecycle_action_token,
            LifecycleActionResult=result
        )
        
    except Exception as e:
        # Abandon on error
        autoscaling.complete_lifecycle_action(
            LifecycleHookName=lifecycle_hook_name,
            AutoScalingGroupName=auto_scaling_group_name,
            InstanceId=instance_id,
            LifecycleActionToken=lifecycle_action_token,
            LifecycleActionResult='ABANDON'
        )
```

---

## 8. Monitoring and Metrics

### CloudWatch Integration
```python
def monitor_auto_scaling_metrics():
    """Monitor Auto Scaling group metrics"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create custom metrics
    cloudwatch.put_metric_data(
        Namespace='AutoScaling/Custom',
        MetricData=[
            {
                'MetricName': 'HealthyInstanceCount',
                'Value': get_healthy_instance_count(),
                'Unit': 'Count',
                'Dimensions': [
                    {
                        'Name': 'AutoScalingGroupName',
                        'Value': 'web-servers-asg'
                    }
                ]
            }
        ]
    )
    
    # Create alarms for scaling events
    cloudwatch.put_metric_alarm(
        AlarmName='ASGScalingActivity',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=1,
        MetricName='GroupTotalInstances',
        Namespace='AWS/AutoScaling',
        Period=300,
        Statistic='Average',
        Threshold=8.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:scaling-alerts'
        ],
        Dimensions=[
            {
                'Name': 'AutoScalingGroupName',
                'Value': 'web-servers-asg'
            }
        ]
    )
```

---

## 9. Cost Optimization

### Spot Instance Integration
```bash
# Create mixed instances policy with Spot
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "cost-optimized-asg" \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "web-template",
        "Version": "$Latest"
      },
      "Overrides": [
        {
          "InstanceType": "t3.micro",
          "WeightedCapacity": "1"
        },
        {
          "InstanceType": "t3.small",
          "WeightedCapacity": "2"
        }
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "diversified",
      "SpotInstancePools": 4,
      "SpotMaxPrice": "0.05"
    }
  }' \
  --min-size 2 \
  --max-size 20 \
  --desired-capacity 4
```

### Scheduled Scaling
```bash
# Create scheduled action for predictable load
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name "web-servers-asg" \
  --scheduled-action-name "morning-scale-up" \
  --recurrence "0 8 * * MON-FRI" \
  --desired-capacity 10 \
  --min-size 5 \
  --max-size 15

# Scale down for off-hours
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name "web-servers-asg" \
  --scheduled-action-name "evening-scale-down" \
  --recurrence "0 18 * * MON-FRI" \
  --desired-capacity 3 \
  --min-size 2 \
  --max-size 10
```

---

## 10. Common Exam Scenarios

### Scenario 1: High availability web application
**Solution:**
- Multi-AZ Auto Scaling group with ELB health checks
- Target tracking scaling based on CPU and request count
- Lifecycle hooks for graceful instance initialization
- CloudWatch alarms for monitoring scaling events

### Scenario 2: Cost-optimized batch processing
**Solution:**
- Mixed instances policy with Spot instances
- Scheduled scaling for predictable workloads
- Custom metrics for queue depth-based scaling
- Lifecycle hooks for job completion handling

### Scenario 3: Database read replica scaling
**Solution:**
- Application Auto Scaling for RDS read replicas
- Target tracking based on CPU utilization
- Custom CloudWatch metrics for connection count
- Automated failover handling

### Scenario 4: Microservices container scaling
**Solution:**
- ECS Service Auto Scaling with target tracking
- Multiple scaling policies for different metrics
- Service discovery integration
- Blue/green deployment support

### Scenario 5: Global application resilience
**Solution:**
- Cross-region Auto Scaling groups
- Route 53 health checks and failover
- Predictive scaling for capacity planning
- Disaster recovery automation

---

## 11. CLI Commands Reference

### Auto Scaling Group Management
```bash
# Create ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "my-asg" \
  --launch-template LaunchTemplateName=my-template,Version='$Latest' \
  --min-size 1 --max-size 5 --desired-capacity 2

# Update ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "my-asg" \
  --desired-capacity 3

# Describe ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "my-asg"

# Delete ASG
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name "my-asg" \
  --force-delete
```

### Scaling Policy Management
```bash
# Create scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "my-asg" \
  --policy-name "scale-up" \
  --scaling-adjustment 1 \
  --adjustment-type "ChangeInCapacity"

# Execute scaling policy
aws autoscaling execute-policy \
  --policy-name "scale-up" \
  --auto-scaling-group-name "my-asg"
```

---

## 12. Exam Tips

### Key Points to Remember
- Auto Scaling maintains desired capacity across AZs
- Health checks determine instance replacement
- Scaling policies can be target tracking, step, or simple
- Lifecycle hooks enable custom initialization/termination logic
- Predictive scaling uses ML for capacity planning
- Mixed instances policy optimizes costs with Spot instances

### Common Mistakes
- Not configuring proper health check grace periods
- Forgetting to set up lifecycle hooks for stateful applications
- Not considering cooldown periods in scaling policies
- Overlooking cross-AZ distribution requirements
- Not monitoring scaling activities and metrics

### Best Practices for Exam
- Understand different scaling policy types and use cases
- Know health check mechanisms and configuration
- Understand lifecycle hooks and their applications
- Know cost optimization strategies with Spot instances
- Understand multi-AZ and cross-region resilience patterns
- Know monitoring and alerting best practices