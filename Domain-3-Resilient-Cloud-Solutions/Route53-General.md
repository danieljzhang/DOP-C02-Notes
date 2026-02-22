# AWS Route 53 - DOP-C02 Study Notes

## 1. Overview

### What is Route 53?
- Highly available and scalable DNS web service
- Domain registration and DNS routing service
- Health checking and failover capabilities
- Global service with 100% availability SLA

### Key Features
- **DNS Resolution** - Authoritative DNS service
- **Domain Registration** - Register and manage domains
- **Health Checks** - Monitor endpoint health
- **Traffic Routing** - Intelligent routing policies
- **DNS Failover** - Automatic failover capabilities

---

## 2. DNS Fundamentals

### DNS Record Types
```bash
# A Record - IPv4 address
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "192.0.2.1"}]
      }
    }]
  }'

# CNAME Record - Canonical name
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "alb-123456789.us-east-1.elb.amazonaws.com"}]
      }
    }]
  }'

# MX Record - Mail exchange
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "example.com",
        "Type": "MX",
        "TTL": 300,
        "ResourceRecords": [
          {"Value": "10 mail1.example.com"},
          {"Value": "20 mail2.example.com"}
        ]
      }
    }]
  }'
```

### Alias Records
```python
def create_alias_record():
    """Create alias record for AWS resources"""
    
    route53 = boto3.client('route53')
    
    # Alias record for ALB
    response = route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'A',
                    'AliasTarget': {
                        'DNSName': 'alb-123456789.us-east-1.elb.amazonaws.com',
                        'EvaluateTargetHealth': True,
                        'HostedZoneId': 'Z35SXDOTRQ7X7K'  # ALB hosted zone ID
                    }
                }
            }]
        }
    )
    
    return response
```

---

## 3. Routing Policies

### Simple Routing
```python
def create_simple_routing():
    """Create simple routing policy"""
    
    route53 = boto3.client('route53')
    
    response = route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'simple.example.com',
                    'Type': 'A',
                    'TTL': 300,
                    'ResourceRecords': [
                        {'Value': '192.0.2.1'},
                        {'Value': '192.0.2.2'}
                    ]
                }
            }]
        }
    )
    
    return response
```

### Weighted Routing
```python
def create_weighted_routing():
    """Create weighted routing for traffic distribution"""
    
    route53 = boto3.client('route53')
    
    # 70% traffic to primary
    route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'Primary-70',
                    'Weight': 70,
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.1'}]
                }
            }]
        }
    )
    
    # 30% traffic to secondary
    route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'Secondary-30',
                    'Weight': 30,
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.2'}]
                }
            }]
        }
    )
```

### Latency-Based Routing
```python
def create_latency_routing():
    """Create latency-based routing for global applications"""
    
    route53 = boto3.client('route53')
    
    regions = [
        {'region': 'us-east-1', 'ip': '192.0.2.1'},
        {'region': 'eu-west-1', 'ip': '192.0.2.2'},
        {'region': 'ap-southeast-1', 'ip': '192.0.2.3'}
    ]
    
    for region_config in regions:
        route53.change_resource_record_sets(
            HostedZoneId='Z123456789',
            ChangeBatch={
                'Changes': [{
                    'Action': 'CREATE',
                    'ResourceRecordSet': {
                        'Name': 'global.example.com',
                        'Type': 'A',
                        'SetIdentifier': f"Latency-{region_config['region']}",
                        'Region': region_config['region'],
                        'TTL': 60,
                        'ResourceRecords': [{'Value': region_config['ip']}]
                    }
                }]
            }
        )
```

### Geolocation Routing
```python
def create_geolocation_routing():
    """Create geolocation-based routing"""
    
    route53 = boto3.client('route53')
    
    # US traffic
    route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'geo.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'US-Traffic',
                    'GeoLocation': {'CountryCode': 'US'},
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.1'}]
                }
            }]
        }
    )
    
    # European traffic
    route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'geo.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'EU-Traffic',
                    'GeoLocation': {'ContinentCode': 'EU'},
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.2'}]
                }
            }]
        }
    )
    
    # Default for all other locations
    route53.change_resource_record_sets(
        HostedZoneId='Z123456',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'geo.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'Default-Traffic',
                    'GeoLocation': {'CountryCode': '*'},
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.3'}]
                }
            }]
        }
    )
```

---

## 4. Health Checks and Failover

### HTTP Health Checks
```python
def create_health_checks():
    """Create comprehensive health checks"""
    
    route53 = boto3.client('route53')
    
    # HTTP health check
    http_health_check = route53.create_health_check(
        Type='HTTP',
        ResourcePath='/health',
        FullyQualifiedDomainName='app1.example.com',
        Port=80,
        RequestInterval=30,
        FailureThreshold=3,
        Tags=[
            {
                'Key': 'Name',
                'Value': 'App1-HealthCheck'
            }
        ]
    )
    
    # HTTPS health check with string matching
    https_health_check = route53.create_health_check(
        Type='HTTPS_STR_MATCH',
        ResourcePath='/api/status',
        FullyQualifiedDomainName='api.example.com',
        Port=443,
        RequestInterval=30,
        FailureThreshold=3,
        SearchString='{"status":"healthy"}',
        Tags=[
            {
                'Key': 'Name',
                'Value': 'API-HealthCheck'
            }
        ]
    )
    
    return {
        'http_health_check_id': http_health_check['HealthCheck']['Id'],
        'https_health_check_id': https_health_check['HealthCheck']['Id']
    }
```

### Failover Routing
```python
def create_failover_routing():
    """Create active-passive failover configuration"""
    
    route53 = boto3.client('route53')
    
    # Create health checks first
    health_checks = create_health_checks()
    
    # Primary record (active)
    route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'Primary',
                    'Failover': 'PRIMARY',
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.1'}],
                    'HealthCheckId': health_checks['http_health_check_id']
                }
            }]
        }
    )
    
    # Secondary record (passive)
    route53.change_resource_record_sets(
        HostedZoneId='Z123456',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'A',
                    'SetIdentifier': 'Secondary',
                    'Failover': 'SECONDARY',
                    'TTL': 60,
                    'ResourceRecords': [{'Value': '192.0.2.2'}]
                }
            }]
        }
    )
```

### Calculated Health Checks
```python
def create_calculated_health_check():
    """Create calculated health check based on multiple checks"""
    
    route53 = boto3.client('route53')
    
    # Create individual health checks
    web_health_check = route53.create_health_check(
        Type='HTTP',
        ResourcePath='/health',
        FullyQualifiedDomainName='web.example.com',
        Port=80
    )
    
    api_health_check = route53.create_health_check(
        Type='HTTPS',
        ResourcePath='/api/health',
        FullyQualifiedDomainName='api.example.com',
        Port=443
    )
    
    db_health_check = route53.create_health_check(
        Type='TCP',
        FullyQualifiedDomainName='db.example.com',
        Port=3306
    )
    
    # Create calculated health check
    calculated_check = route53.create_health_check(
        Type='CALCULATED',
        ChildHealthChecks=[
            web_health_check['HealthCheck']['Id'],
            api_health_check['HealthCheck']['Id'],
            db_health_check['HealthCheck']['Id']
        ],
        HealthThreshold=2,  # At least 2 out of 3 must be healthy
        Tags=[
            {
                'Key': 'Name',
                'Value': 'Application-Overall-Health'
            }
        ]
    )
    
    return calculated_check['HealthCheck']['Id']
```

---

## 5. Multi-Region Resilience

### Global Load Balancing
```python
def setup_global_load_balancing():
    """Set up global load balancing across regions"""
    
    route53 = boto3.client('route53')
    
    # Create health checks for each region
    regions = [
        {'name': 'us-east-1', 'endpoint': 'us-east-alb.example.com', 'weight': 50},
        {'name': 'eu-west-1', 'endpoint': 'eu-west-alb.example.com', 'weight': 30},
        {'name': 'ap-southeast-1', 'endpoint': 'ap-southeast-alb.example.com', 'weight': 20}
    ]
    
    health_check_ids = {}
    
    for region in regions:
        health_check = route53.create_health_check(
            Type='HTTPS',
            ResourcePath='/health',
            FullyQualifiedDomainName=region['endpoint'],
            Port=443,
            RequestInterval=30,
            FailureThreshold=3
        )
        health_check_ids[region['name']] = health_check['HealthCheck']['Id']
    
    # Create weighted routing with health checks
    for region in regions:
        route53.change_resource_record_sets(
            HostedZoneId='Z123456789',
            ChangeBatch={
                'Changes': [{
                    'Action': 'CREATE',
                    'ResourceRecordSet': {
                        'Name': 'global.example.com',
                        'Type': 'CNAME',
                        'SetIdentifier': f"Region-{region['name']}",
                        'Weight': region['weight'],
                        'TTL': 60,
                        'ResourceRecords': [{'Value': region['endpoint']}],
                        'HealthCheckId': health_check_ids[region['name']]
                    }
                }]
            }
        )
```

### Disaster Recovery DNS
```python
def setup_disaster_recovery_dns():
    """Set up DNS for disaster recovery scenarios"""
    
    route53 = boto3.client('route53')
    
    # Primary site health check
    primary_health_check = route53.create_health_check(
        Type='HTTPS',
        ResourcePath='/health',
        FullyQualifiedDomainName='primary.example.com',
        Port=443,
        RequestInterval=30,
        FailureThreshold=2  # Faster failover
    )
    
    # DR site health check
    dr_health_check = route53.create_health_check(
        Type='HTTPS',
        ResourcePath='/health',
        FullyQualifiedDomainName='dr.example.com',
        Port=443,
        RequestInterval=30,
        FailureThreshold=3
    )
    
    # Primary site record
    route53.change_resource_record_sets(
        HostedZoneId='Z123456789',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'CNAME',
                    'SetIdentifier': 'Primary-Site',
                    'Failover': 'PRIMARY',
                    'TTL': 30,  # Low TTL for faster failover
                    'ResourceRecords': [{'Value': 'primary.example.com'}],
                    'HealthCheckId': primary_health_check['HealthCheck']['Id']
                }
            }]
        }
    )
    
    # DR site record
    route53.change_resource_record_sets(
        HostedZoneId='Z123456',
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'app.example.com',
                    'Type': 'CNAME',
                    'SetIdentifier': 'DR-Site',
                    'Failover': 'SECONDARY',
                    'TTL': 30,
                    'ResourceRecords': [{'Value': 'dr.example.com'}],
                    'HealthCheckId': dr_health_check['HealthCheck']['Id']
                }
            }]
        }
    )
```

---

## 6. Private DNS and Resolver

### Private Hosted Zones
```python
def create_private_hosted_zone():
    """Create private hosted zone for VPC"""
    
    route53 = boto3.client('route53')
    
    # Create private hosted zone
    response = route53.create_hosted_zone(
        Name='internal.example.com',
        VPC={
            'VPCRegion': 'us-east-1',
            'VPCId': 'vpc-12345678'
        },
        CallerReference=str(uuid.uuid4()),
        HostedZoneConfig={
            'PrivateZone': True,
            'Comment': 'Private zone for internal services'
        }
    )
    
    hosted_zone_id = response['HostedZone']['Id']
    
    # Add internal service records
    route53.change_resource_record_sets(
        HostedZoneId=hosted_zone_id,
        ChangeBatch={
            'Changes': [{
                'Action': 'CREATE',
                'ResourceRecordSet': {
                    'Name': 'database.internal.example.com',
                    'Type': 'A',
                    'TTL': 300,
                    'ResourceRecords': [{'Value': '10.0.1.100'}]
                }
            }]
        }
    )
    
    return hosted_zone_id
```

### Route 53 Resolver
```bash
# Create resolver endpoint for hybrid DNS
aws route53resolver create-resolver-endpoint \
  --creator-request-id "$(uuidgen)" \
  --security-group-ids "sg-12345678" \
  --direction "OUTBOUND" \
  --ip-addresses SubnetId=subnet-12345678,Ip=10.0.1.10 SubnetId=subnet-87654321,Ip=10.0.2.10 \
  --tags Key=Name,Value=OutboundResolver

# Create resolver rule for on-premises domain
aws route53resolver create-resolver-rule \
  --creator-request-id "$(uuidgen)" \
  --domain-name "onprem.example.com" \
  --rule-type "FORWARD" \
  --resolver-endpoint-id "rslvr-out-12345678" \
  --target-ips Ip=192.168.1.10,Port=53 Ip=192.168.1.11,Port=53 \
  --tags Key=Name,Value=OnPremRule
```

---

## 7. Monitoring and Logging

### CloudWatch Integration
```python
def setup_route53_monitoring():
    """Set up Route 53 monitoring and alerting"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Health check status alarm
    cloudwatch.put_metric_alarm(
        AlarmName='Route53-HealthCheck-Failed',
        ComparisonOperator='LessThanThreshold',
        EvaluationPeriods=1,
        MetricName='HealthCheckStatus',
        Namespace='AWS/Route53',
        Period=60,
        Statistic='Minimum',
        Threshold=1.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:route53-alerts'
        ],
        Dimensions=[
            {
                'Name': 'HealthCheckId',
                'Value': 'health-check-id-12345'
            }
        ]
    )
    
    # Health check percentage alarm
    cloudwatch.put_metric_alarm(
        AlarmName='Route53-HealthCheck-LowPercentage',
        ComparisonOperator='LessThanThreshold',
        EvaluationPeriods=2,
        MetricName='HealthCheckPercentHealthy',
        Namespace='AWS/Route53',
        Period=300,
        Statistic='Average',
        Threshold=80.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:route53-alerts'
        ]
    )
```

### Query Logging
```bash
# Enable query logging for hosted zone
aws route53 create-query-logging-config \
  --hosted-zone-id "Z123456789" \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:us-east-1:123456789012:log-group:/aws/route53/queries"
```

---

## 8. Common Exam Scenarios

### Scenario 1: Global application with automatic failover
**Solution:**
- Latency-based routing for optimal performance
- Health checks on all endpoints
- Failover routing for disaster recovery
- Low TTL values for fast failover

### Scenario 2: Blue/green deployment with DNS switching
**Solution:**
- Weighted routing for gradual traffic shift
- Health checks on both environments
- Alias records for AWS resources
- Monitoring and rollback procedures

### Scenario 3: Multi-region disaster recovery
**Solution:**
- Primary/secondary failover configuration
- Cross-region health checks
- Automated DNS failover
- RTO/RPO requirements consideration

### Scenario 4: Hybrid cloud DNS resolution
**Solution:**
- Route 53 Resolver for hybrid connectivity
- Private hosted zones for internal services
- Conditional forwarding rules
- On-premises DNS integration

### Scenario 5: Microservices service discovery
**Solution:**
- Private hosted zones for service names
- Service registration automation
- Health check integration
- Load balancing across service instances

---

## 9. CLI Commands Reference

### Hosted Zone Management
```bash
# Create hosted zone
aws route53 create-hosted-zone \
  --name "example.com" \
  --caller-reference "$(date +%s)"

# List hosted zones
aws route53 list-hosted-zones

# Delete hosted zone
aws route53 delete-hosted-zone \
  --id "Z123456789"
```

### DNS Records Management
```bash
# Create A record
aws route53 change-resource-record-sets \
  --hosted-zone-id "Z123456789" \
  --change-batch file://change-batch.json

# List records
aws route53 list-resource-record-sets \
  --hosted-zone-id "Z123456789"
```

### Health Checks
```bash
# Create health check
aws route53 create-health-check \
  --type "HTTP" \
  --resource-path "/health" \
  --fully-qualified-domain-name "example.com"

# List health checks
aws route53 list-health-checks

# Get health check status
aws route53 get-health-check-status \
  --health-check-id "health-check-id"
```

---

## 10. Exam Tips

### Key Points to Remember
- Route 53 is a global service with 100% availability SLA
- Alias records are free and automatically resolve to AWS resource IPs
- Health checks can monitor HTTP/HTTPS/TCP endpoints
- Failover routing requires health checks on primary record
- TTL affects how quickly DNS changes propagate
- Private hosted zones work within VPC boundaries

### Common Mistakes
- Not configuring health checks for failover routing
- Using high TTL values when fast failover is needed
- Forgetting to associate private hosted zones with VPCs
- Not understanding the difference between CNAME and Alias records
- Overlooking geolocation default record requirements

### Best Practices for Exam
- Understand all routing policy types and use cases
- Know health check types and configuration options
- Understand DNS failover mechanisms and timing
- Know private DNS and hybrid connectivity patterns
- Understand monitoring and troubleshooting approaches
- Know cost optimization strategies for DNS queries