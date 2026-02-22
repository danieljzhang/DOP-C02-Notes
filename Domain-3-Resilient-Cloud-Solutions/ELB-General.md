# AWS Elastic Load Balancing (ELB) - DOP-C02 Study Notes

## 1. Overview

### What is ELB?
- Distributes incoming traffic across multiple targets
- Provides high availability and fault tolerance
- Automatically scales to handle traffic demands
- Integrates with Auto Scaling and health checks

### Load Balancer Types
- **Application Load Balancer (ALB)** - Layer 7 (HTTP/HTTPS)
- **Network Load Balancer (NLB)** - Layer 4 (TCP/UDP/TLS)
- **Gateway Load Balancer (GWLB)** - Layer 3 (IP packets)
- **Classic Load Balancer (CLB)** - Legacy (Layer 4 & 7)

---

## 2. Application Load Balancer (ALB)

### Core Features
- HTTP/HTTPS traffic routing
- Content-based routing
- WebSocket and HTTP/2 support
- Integration with AWS services

### Target Groups and Health Checks
```bash
# Create target group
aws elbv2 create-target-group \
  --name "web-servers-tg" \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-12345678 \
  --health-check-protocol HTTP \
  --health-check-path "/health" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Register targets
aws elbv2 register-targets \
  --target-group-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-servers-tg/1234567890123456" \
  --targets Id=i-1234567890abcdef0,Port=80 Id=i-0987654321fedcba0,Port=80
```

### Advanced Routing
```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name "web-application-alb" \
  --subnets subnet-12345678 subnet-87654321 \
  --security-groups sg-12345678 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4

# Create listener with rules
aws elbv2 create-listener \
  --load-balancer-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/web-application-alb/1234567890123456" \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012" \
  --default-actions Type=forward,TargetGroupArn="arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-servers-tg/1234567890123456"

# Create routing rule
aws elbv2 create-rule \
  --listener-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/web-application-alb/1234567890123456/1234567890123456" \
  --priority 100 \
  --conditions Field=path-pattern,Values="/api/*" \
  --actions Type=forward,TargetGroupArn="arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/api-servers-tg/1234567890123456"
```

### Sticky Sessions
```python
def configure_sticky_sessions():
    """Configure session stickiness for ALB"""
    
    elbv2 = boto3.client('elbv2')
    
    # Enable stickiness on target group
    response = elbv2.modify_target_group_attributes(
        TargetGroupArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-servers-tg/1234567890123456',
        Attributes=[
            {
                'Key': 'stickiness.enabled',
                'Value': 'true'
            },
            {
                'Key': 'stickiness.type',
                'Value': 'lb_cookie'
            },
            {
                'Key': 'stickiness.lb_cookie.duration_seconds',
                'Value': '86400'  # 24 hours
            }
        ]
    )
    
    return response
```

---

## 3. Network Load Balancer (NLB)

### High Performance Layer 4 Routing
```bash
# Create NLB
aws elbv2 create-load-balancer \
  --name "high-performance-nlb" \
  --subnets subnet-12345678 subnet-87654321 \
  --scheme internet-facing \
  --type network \
  --ip-address-type ipv4

# Create target group for NLB
aws elbv2 create-target-group \
  --name "tcp-servers-tg" \
  --protocol TCP \
  --port 80 \
  --vpc-id vpc-12345678 \
  --health-check-protocol TCP \
  --health-check-port 80 \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3
```

### Static IP and Cross-Zone Load Balancing
```python
def configure_nlb_features():
    """Configure NLB advanced features"""
    
    elbv2 = boto3.client('elbv2')
    
    # Enable cross-zone load balancing
    elbv2.modify_load_balancer_attributes(
        LoadBalancerArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/net/high-performance-nlb/1234567890123456',
        Attributes=[
            {
                'Key': 'load_balancing.cross_zone.enabled',
                'Value': 'true'
            },
            {
                'Key': 'deletion_protection.enabled',
                'Value': 'true'
            }
        ]
    )
    
    # Configure target group attributes
    elbv2.modify_target_group_attributes(
        TargetGroupArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/tcp-servers-tg/1234567890123456',
        Attributes=[
            {
                'Key': 'preserve_client_ip.enabled',
                'Value': 'true'
            },
            {
                'Key': 'proxy_protocol_v2.enabled',
                'Value': 'false'
            }
        ]
    )
```

---

## 4. Gateway Load Balancer (GWLB)

### Third-Party Appliance Integration
```bash
# Create GWLB for security appliances
aws elbv2 create-load-balancer \
  --name "security-appliance-gwlb" \
  --subnets subnet-12345678 subnet-87654321 \
  --type gateway

# Create target group for GWLB
aws elbv2 create-target-group \
  --name "security-appliances-tg" \
  --protocol GENEVE \
  --port 6081 \
  --vpc-id vpc-12345678 \
  --health-check-protocol HTTP \
  --health-check-path "/health" \
  --health-check-port 80
```

---

## 5. Multi-AZ and Cross-Region Resilience

### Multi-AZ Deployment
```python
def create_resilient_alb():
    """Create highly available ALB across multiple AZs"""
    
    elbv2 = boto3.client('elbv2')
    ec2 = boto3.client('ec2')
    
    # Get subnets across multiple AZs
    subnets = ec2.describe_subnets(
        Filters=[
            {'Name': 'vpc-id', 'Values': ['vpc-12345678']},
            {'Name': 'availability-zone', 'Values': ['us-east-1a', 'us-east-1b', 'us-east-1c']}
        ]
    )
    
    subnet_ids = [subnet['SubnetId'] for subnet in subnets['Subnets']]
    
    # Create ALB with multi-AZ subnets
    response = elbv2.create_load_balancer(
        Name='resilient-web-alb',
        Subnets=subnet_ids,
        SecurityGroups=['sg-12345678'],
        Scheme='internet-facing',
        Type='application',
        IpAddressType='ipv4'
    )
    
    return response
```

### Cross-Region Load Balancing with Route 53
```python
def setup_cross_region_load_balancing():
    """Set up cross-region load balancing with Route 53"""
    
    route53 = boto3.client('route53')
    
    # Create health checks for each region
    health_checks = []
    regions = ['us-east-1', 'us-west-2', 'eu-west-1']
    
    for region in regions:
        health_check = route53.create_health_check(
            Type='HTTPS',
            ResourcePath='/health',
            FullyQualifiedDomainName=f'alb-{region}.example.com',
            Port=443,
            RequestInterval=30,
            FailureThreshold=3
        )
        health_checks.append(health_check['HealthCheck']['Id'])
    
    # Create weighted routing records
    for i, region in enumerate(regions):
        route53.change_resource_record_sets(
            HostedZoneId='Z123456789',
            ChangeBatch={
                'Changes': [{
                    'Action': 'CREATE',
                    'ResourceRecordSet': {
                        'Name': 'app.example.com',
                        'Type': 'CNAME',
                        'SetIdentifier': f'region-{region}',
                        'Weight': 100,
                        'TTL': 60,
                        'ResourceRecords': [{'Value': f'alb-{region}.example.com'}],
                        'HealthCheckId': health_checks[i]
                    }
                }]
            }
        )
```

---

## 6. SSL/TLS Termination and Security

### SSL Certificate Management
```bash
# Create HTTPS listener with SSL termination
aws elbv2 create-listener \
  --load-balancer-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/web-alb/1234567890123456" \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012" \
  --ssl-policy "ELBSecurityPolicy-TLS-1-2-2017-01" \
  --default-actions Type=forward,TargetGroupArn="arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-tg/1234567890123456"

# Redirect HTTP to HTTPS
aws elbv2 create-listener \
  --load-balancer-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/web-alb/1234567890123456" \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

### WAF Integration
```python
def integrate_waf_with_alb():
    """Integrate AWS WAF with Application Load Balancer"""
    
    wafv2 = boto3.client('wafv2')
    
    # Associate WAF with ALB
    response = wafv2.associate_web_acl(
        WebACLArn='arn:aws:wafv2:us-east-1:123456789012:regional/webacl/MyWebACL/12345678-1234-1234-1234-123456789012',
        ResourceArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/web-alb/1234567890123456'
    )
    
    return response
```

---

## 7. Health Checks and Monitoring

### Advanced Health Check Configuration
```python
def configure_advanced_health_checks():
    """Configure comprehensive health checks"""
    
    elbv2 = boto3.client('elbv2')
    
    # Configure detailed health check
    response = elbv2.modify_target_group(
        TargetGroupArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-tg/1234567890123456',
        HealthCheckProtocol='HTTP',
        HealthCheckPath='/health/detailed',
        HealthCheckIntervalSeconds=15,
        HealthCheckTimeoutSeconds=10,
        HealthyThresholdCount=2,
        UnhealthyThresholdCount=5,
        Matcher={'HttpCode': '200,202'}
    )
    
    return response
```

### CloudWatch Metrics and Alarms
```python
def setup_elb_monitoring():
    """Set up comprehensive ELB monitoring"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create alarm for high latency
    cloudwatch.put_metric_alarm(
        AlarmName='ALB-HighLatency',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        MetricName='TargetResponseTime',
        Namespace='AWS/ApplicationELB',
        Period=300,
        Statistic='Average',
        Threshold=2.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:alb-alerts'
        ],
        Dimensions=[
            {
                'Name': 'LoadBalancer',
                'Value': 'app/web-alb/1234567890123456'
            }
        ]
    )
    
    # Create alarm for high error rate
    cloudwatch.put_metric_alarm(
        AlarmName='ALB-HighErrorRate',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        MetricName='HTTPCode_ELB_5XX_Count',
        Namespace='AWS/ApplicationELB',
        Period=300,
        Statistic='Sum',
        Threshold=10,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:error-alerts'
        ]
    )
```

---

## 8. Auto Scaling Integration

### Target Group Auto Scaling
```python
def integrate_with_auto_scaling():
    """Integrate ELB with Auto Scaling groups"""
    
    autoscaling = boto3.client('autoscaling')
    
    # Create Auto Scaling group with target group
    autoscaling.create_auto_scaling_group(
        AutoScalingGroupName='web-servers-asg',
        LaunchTemplate={
            'LaunchTemplateName': 'web-server-template',
            'Version': '$Latest'
        },
        MinSize=2,
        MaxSize=10,
        DesiredCapacity=3,
        TargetGroupARNs=[
            'arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-tg/1234567890123456'
        ],
        HealthCheckType='ELB',
        HealthCheckGracePeriod=300,
        VPCZoneIdentifier='subnet-12345678,subnet-87654321'
    )
    
    # Create scaling policy based on ALB metrics
    autoscaling.put_scaling_policy(
        AutoScalingGroupName='web-servers-asg',
        PolicyName='alb-request-count-scaling',
        PolicyType='TargetTrackingScaling',
        TargetTrackingConfiguration={
            'TargetValue': 1000.0,
            'PredefinedMetricSpecification': {
                'PredefinedMetricType': 'ALBRequestCountPerTarget',
                'ResourceLabel': 'app/web-alb/1234567890123456/targetgroup/web-tg/1234567890123456'
            }
        }
    )
```

---

## 9. Connection Draining and Blue/Green Deployments

### Connection Draining
```python
def configure_connection_draining():
    """Configure connection draining for graceful shutdowns"""
    
    elbv2 = boto3.client('elbv2')
    
    # Configure deregistration delay
    elbv2.modify_target_group_attributes(
        TargetGroupArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-tg/1234567890123456',
        Attributes=[
            {
                'Key': 'deregistration_delay.timeout_seconds',
                'Value': '300'  # 5 minutes
            }
        ]
    )
```

### Blue/Green Deployment Support
```python
def blue_green_deployment():
    """Implement blue/green deployment with ELB"""
    
    elbv2 = boto3.client('elbv2')
    
    # Create green target group
    green_tg = elbv2.create_target_group(
        Name='web-servers-green-tg',
        Protocol='HTTP',
        Port=80,
        VpcId='vpc-12345678',
        HealthCheckPath='/health'
    )
    
    green_tg_arn = green_tg['TargetGroups'][0]['TargetGroupArn']
    
    # Register new instances to green target group
    elbv2.register_targets(
        TargetGroupArn=green_tg_arn,
        Targets=[
            {'Id': 'i-newinstance1', 'Port': 80},
            {'Id': 'i-newinstance2', 'Port': 80}
        ]
    )
    
    # Wait for health checks to pass
    wait_for_healthy_targets(green_tg_arn)
    
    # Switch traffic to green target group
    elbv2.modify_listener(
        ListenerArn='arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/web-alb/1234567890123456/1234567890123456',
        DefaultActions=[
            {
                'Type': 'forward',
                'TargetGroupArn': green_tg_arn
            }
        ]
    )
```

---

## 10. Common Exam Scenarios

### Scenario 1: High availability web application
**Solution:**
- ALB with multi-AZ target groups
- Auto Scaling group integration
- Health checks with proper thresholds
- SSL termination with ACM certificates

### Scenario 2: Microservices routing
**Solution:**
- ALB with path-based routing rules
- Multiple target groups for different services
- Host-based routing for multi-tenant applications
- WebSocket support for real-time features

### Scenario 3: High-performance TCP application
**Solution:**
- Network Load Balancer for Layer 4 routing
- Static IP addresses with Elastic IPs
- Cross-zone load balancing for even distribution
- Connection preservation for stateful applications

### Scenario 4: Security appliance integration
**Solution:**
- Gateway Load Balancer for traffic inspection
- Third-party security appliance target groups
- Transparent traffic forwarding
- High availability across multiple AZs

### Scenario 5: Global application with failover
**Solution:**
- Multiple ALBs across regions
- Route 53 health checks and failover routing
- Cross-region Auto Scaling groups
- Disaster recovery automation

---

## 11. CLI Commands Reference

### Load Balancer Management
```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name "my-alb" \
  --subnets subnet-12345678 subnet-87654321 \
  --security-groups sg-12345678

# Describe load balancers
aws elbv2 describe-load-balancers

# Delete load balancer
aws elbv2 delete-load-balancer \
  --load-balancer-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/1234567890123456"
```

### Target Group Management
```bash
# Create target group
aws elbv2 create-target-group \
  --name "my-targets" \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-12345678

# Register targets
aws elbv2 register-targets \
  --target-group-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/1234567890123456" \
  --targets Id=i-1234567890abcdef0,Port=80

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/1234567890123456"
```

---

## 12. Exam Tips

### Key Points to Remember
- ALB operates at Layer 7 (HTTP/HTTPS), NLB at Layer 4 (TCP/UDP)
- Health checks determine target availability
- Cross-zone load balancing distributes traffic evenly across AZs
- SSL termination reduces backend server load
- Connection draining enables graceful shutdowns
- Integration with Auto Scaling provides automatic capacity management

### Common Mistakes
- Not configuring proper health check paths and thresholds
- Forgetting to enable cross-zone load balancing when needed
- Not setting up SSL termination properly
- Overlooking security group configurations
- Not monitoring load balancer metrics and setting up alarms

### Best Practices for Exam
- Understand different load balancer types and use cases
- Know health check configuration and troubleshooting
- Understand SSL/TLS termination and security features
- Know integration patterns with Auto Scaling and Route 53
- Understand monitoring and alerting capabilities
- Know blue/green deployment patterns with load balancers