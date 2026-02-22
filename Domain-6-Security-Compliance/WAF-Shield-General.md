# AWS WAF & Shield - DOP-C02 Study Notes

## 1. Overview

### AWS WAF (Web Application Firewall)
- Layer 7 firewall for web applications
- Protects against common web exploits and attacks
- Integrates with CloudFront, ALB, API Gateway, AppSync
- Rule-based filtering with custom and managed rules

### AWS Shield
- DDoS (Distributed Denial of Service) protection service
- **Shield Standard** - Free, automatic protection
- **Shield Advanced** - Enhanced protection with 24/7 support
- Protects against network and transport layer attacks

### Key Benefits
- **Real-time Protection** - Immediate threat response
- **Scalable** - Handles traffic spikes automatically
- **Cost Effective** - Pay for what you use
- **Integration** - Native AWS service integration
- **Monitoring** - CloudWatch metrics and logging

---

## 2. AWS WAF Core Concepts

### Web ACLs (Access Control Lists)
- Container for rules that define traffic filtering
- Associated with AWS resources (CloudFront, ALB, etc.)
- Default action: Allow or Block
- Rule evaluation order matters

### Rules and Rule Groups
- **Rules** - Individual filtering logic
- **Rule Groups** - Collection of related rules
- **Managed Rule Groups** - AWS and marketplace rules
- **Custom Rule Groups** - Your own rules

### Rule Actions
- **Allow** - Permit the request
- **Block** - Block the request (403 Forbidden)
- **Count** - Count but don't block (monitoring mode)
- **CAPTCHA** - Challenge with CAPTCHA
- **Challenge** - JavaScript challenge

---

## 3. WAF Rule Types

### Rate-Based Rules
- Limit requests from single IP address
- Configurable rate limit (requests per 5 minutes)
- Automatic blocking when threshold exceeded

```json
{
  "Name": "RateLimitRule",
  "Priority": 1,
  "Statement": {
    "RateBasedStatement": {
      "Limit": 2000,
      "AggregateKeyType": "IP"
    }
  },
  "Action": {
    "Block": {}
  }
}
```

### Geographic Rules
- Block or allow traffic from specific countries
- Based on source IP geolocation
- Useful for compliance requirements

```json
{
  "Name": "GeoBlockRule",
  "Priority": 2,
  "Statement": {
    "GeoMatchStatement": {
      "CountryCodes": ["CN", "RU", "KP"]
    }
  },
  "Action": {
    "Block": {}
  }
}
```

### IP Set Rules
- Match against list of IP addresses or CIDR blocks
- Reusable across multiple rules
- Support for IPv4 and IPv6

```json
{
  "Name": "IPBlockRule",
  "Priority": 3,
  "Statement": {
    "IPSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:us-east-1:123456789012:global/ipset/BlockedIPs/12345678-1234-1234-1234-123456789012"
    }
  },
  "Action": {
    "Block": {}
  }
}
```

### String Match Rules
- Match specific strings in request components
- Case sensitive or insensitive matching
- Multiple transformation options

```json
{
  "Name": "SQLInjectionRule",
  "Priority": 4,
  "Statement": {
    "ByteMatchStatement": {
      "SearchString": "union select",
      "FieldToMatch": {
        "Body": {}
      },
      "TextTransformations": [
        {
          "Priority": 0,
          "Type": "LOWERCASE"
        }
      ],
      "PositionalConstraint": "CONTAINS"
    }
  },
  "Action": {
    "Block": {}
  }
}
```

---

## 4. Managed Rule Groups

### AWS Managed Rules
- **Core Rule Set** - General OWASP protection
- **Known Bad Inputs** - Common malicious patterns
- **SQL Database** - SQL injection protection
- **Linux Operating System** - Linux-specific attacks
- **Windows Operating System** - Windows-specific attacks
- **PHP Application** - PHP-specific vulnerabilities

### Marketplace Rules
- Third-party security vendors
- Specialized protection (e.g., bot management)
- Additional cost beyond WAF pricing

### Rule Group Configuration
```bash
# Create Web ACL with managed rules
aws wafv2 create-web-acl \
  --name "MyWebACL" \
  --scope CLOUDFRONT \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSetMetric"
      }
    }
  ]'
```

---

## 5. WAF Integration

### CloudFront Integration
- Global protection at edge locations
- Lowest latency for users worldwide
- Cached content protection

```bash
# Associate WAF with CloudFront distribution
aws cloudfront update-distribution \
  --id E1234567890123 \
  --distribution-config '{
    "WebACLId": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/MyWebACL/12345678-1234-1234-1234-123456789012"
  }'
```

### Application Load Balancer Integration
- Regional protection for ALB
- Protects backend applications
- Integration with Auto Scaling

```bash
# Associate WAF with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/MyWebACL/12345678-1234-1234-1234-123456789012" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/1234567890123456"
```

### API Gateway Integration
- Protect REST and HTTP APIs
- Rate limiting and request filtering
- Integration with Lambda authorizers

```bash
# Associate WAF with API Gateway
aws apigateway update-stage \
  --rest-api-id abc123 \
  --stage-name prod \
  --patch-ops op=replace,path=/webAclArn,value="arn:aws:wafv2:us-east-1:123456789012:regional/webacl/MyWebACL/12345678-1234-1234-1234-123456789012"
```

---

## 6. AWS Shield Standard

### Automatic Protection
- Included free with all AWS accounts
- Protects against common DDoS attacks
- Layer 3 and 4 protection (network/transport)
- Always-on detection and mitigation

### Protected Services
- **CloudFront** - Global edge locations
- **Route 53** - DNS service
- **Elastic Load Balancing** - ALB, NLB, CLB
- **AWS Global Accelerator** - Network optimization

### Attack Types Covered
- **SYN/UDP Floods** - Network layer attacks
- **Reflection Attacks** - Amplification attacks
- **Layer 3/4 Attacks** - Network and transport layer

---

## 7. AWS Shield Advanced

### Enhanced Protection
- $3,000/month per organization
- 24/7 access to DDoS Response Team (DRT)
- Advanced attack diagnostics and mitigation
- DDoS cost protection (scaling charges)

### Additional Features
- **Real-time Attack Notifications** - SNS integration
- **Attack Forensics** - Detailed attack analysis
- **Proactive Engagement** - DRT monitors your resources
- **Global Threat Environment Dashboard** - Attack visibility

### Enable Shield Advanced
```bash
# Subscribe to Shield Advanced
aws shield subscribe-to-proactive-engagement \
  --proactive-engagement-status ENABLED \
  --emergency-contact-list '[
    {
      "EmailAddress": "security@example.com",
      "PhoneNumber": "+1-555-123-4567",
      "ContactNotes": "Primary security contact"
    }
  ]'
```

### Protected Resources
```bash
# Enable protection for specific resource
aws shield create-protection \
  --name "MyALBProtection" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/1234567890123456"
```

---

## 8. Monitoring and Logging

### CloudWatch Metrics
- **AllowedRequests** - Requests allowed by WAF
- **BlockedRequests** - Requests blocked by WAF
- **CountedRequests** - Requests counted but not blocked
- **SampledRequests** - Sample of requests for analysis

### WAF Logs
```json
{
  "timestamp": 1576280412771,
  "formatVersion": 1,
  "webaclId": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/MyWebACL/12345678-1234-1234-1234-123456789012",
  "terminatingRuleId": "RateLimitRule",
  "terminatingRuleType": "RATE_BASED",
  "action": "BLOCK",
  "httpSourceName": "ALB",
  "httpSourceId": "123456789012-app/my-alb/1234567890123456",
  "ruleGroupList": [],
  "rateBasedRuleList": [
    {
      "rateBasedRuleId": "RateLimitRule",
      "limitKey": "IP",
      "maxRateAllowed": 2000
    }
  ],
  "nonTerminatingMatchingRules": [],
  "httpRequest": {
    "clientIp": "192.0.2.44",
    "country": "US",
    "headers": [
      {
        "name": "Host",
        "value": "example.com"
      }
    ],
    "uri": "/path",
    "args": "arg1=value1&arg2=value2",
    "httpVersion": "HTTP/1.1",
    "httpMethod": "GET",
    "requestId": "rid"
  }
}
```

### Enable Logging
```bash
# Enable WAF logging to S3
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/MyWebACL/12345678-1234-1234-1234-123456789012",
    "LogDestinationConfigs": [
      "arn:aws:s3:::my-waf-logs-bucket"
    ],
    "RedactedFields": [
      {
        "SingleHeader": {
          "Name": "authorization"
        }
      }
    ]
  }'
```

---

## 9. Security Automations

### Automated Response to Attacks
```python
import boto3
import json

def lambda_handler(event, context):
    """Automated WAF rule updates based on CloudWatch alarms"""
    
    wafv2 = boto3.client('wafv2')
    
    # Parse CloudWatch alarm
    message = json.loads(event['Records'][0]['Sns']['Message'])
    
    if message['NewStateValue'] == 'ALARM':
        # Block suspicious IP
        suspicious_ip = extract_ip_from_alarm(message)
        
        # Update IP set
        response = wafv2.update_ip_set(
            Scope='REGIONAL',
            Id='blocked-ips-set-id',
            Addresses=[suspicious_ip + '/32'],
            LockToken='lock-token'
        )
        
        print(f"Blocked IP: {suspicious_ip}")
    
    return {'statusCode': 200}

def extract_ip_from_alarm(message):
    """Extract IP address from alarm details"""
    # Implementation depends on alarm configuration
    return "192.0.2.100"
```

### Rate Limiting Automation
```python
import boto3

def update_rate_limit(web_acl_id, rule_name, new_limit):
    """Update rate limit based on traffic patterns"""
    
    wafv2 = boto3.client('wafv2')
    
    # Get current Web ACL
    response = wafv2.get_web_acl(
        Scope='REGIONAL',
        Id=web_acl_id
    )
    
    web_acl = response['WebACL']
    
    # Update rate limit rule
    for rule in web_acl['Rules']:
        if rule['Name'] == rule_name:
            rule['Statement']['RateBasedStatement']['Limit'] = new_limit
    
    # Update Web ACL
    wafv2.update_web_acl(
        Scope='REGIONAL',
        Id=web_acl_id,
        DefaultAction=web_acl['DefaultAction'],
        Rules=web_acl['Rules'],
        LockToken=response['LockToken']
    )
```

---

## 10. Cost Optimization

### WAF Pricing Components
- **Web ACL** - $1.00 per month
- **Rules** - $0.60 per rule per month
- **Requests** - $0.60 per million requests
- **Rule Group** - $1.50 per rule group per month

### Shield Advanced Pricing
- **Monthly Fee** - $3,000 per organization
- **Data Transfer** - Standard rates apply
- **DDoS Cost Protection** - Covers scaling charges during attacks

### Cost Optimization Strategies
- Use managed rule groups efficiently
- Monitor rule effectiveness and remove unused rules
- Optimize rule priorities for performance
- Use sampling for logging to reduce costs
- Regular review of protection requirements

---

## 11. Best Practices

### Security Best Practices
- Start with managed rule groups
- Use count mode for testing new rules
- Implement rate limiting for APIs
- Monitor and analyze WAF logs regularly
- Keep rules updated with threat intelligence

### Performance Best Practices
- Order rules by likelihood of match
- Use specific match conditions
- Avoid overly broad rules
- Monitor rule evaluation metrics
- Test rules in staging environment

### Operational Best Practices
- Automate rule updates based on threat intelligence
- Implement incident response procedures
- Regular security assessments
- Document rule purposes and configurations
- Train team on WAF management

---

## 12. Common Exam Scenarios

### Scenario 1: Protect web application from SQL injection
**Solution:**
- Enable AWS Managed Rules SQL Database rule group
- Create custom string match rules for specific patterns
- Use count mode for testing before blocking
- Monitor CloudWatch metrics for effectiveness

### Scenario 2: Block traffic from specific countries
**Solution:**
- Create geographic match rule
- Specify country codes to block
- Set rule action to Block
- Monitor blocked requests in CloudWatch

### Scenario 3: Rate limiting for API protection
**Solution:**
- Create rate-based rule with appropriate limit
- Set aggregation key type (IP or forwarded IP)
- Configure rule action (Block or Count)
- Monitor rate limit violations

### Scenario 4: DDoS protection for critical application
**Solution:**
- Enable Shield Advanced for enhanced protection
- Configure proactive engagement with DRT
- Set up CloudWatch alarms for attack detection
- Implement automated scaling responses

### Scenario 5: Automated threat response
**Solution:**
- Enable WAF logging to CloudWatch Logs
- Create CloudWatch alarms for suspicious patterns
- Use Lambda for automated rule updates
- Integrate with SNS for notifications

---

## 13. Troubleshooting Guide

### False Positives
- Review sampled requests in WAF console
- Adjust rule sensitivity or conditions
- Use count mode to test rule changes
- Implement whitelisting for legitimate traffic

### Performance Issues
- Check rule evaluation order and complexity
- Monitor CloudWatch metrics for latency
- Optimize rule conditions for efficiency
- Consider rule group consolidation

### Integration Issues
- Verify resource association with Web ACL
- Check IAM permissions for WAF operations
- Validate CloudFront or ALB configuration
- Test with simple rules first

---

## 14. CLI Commands Reference

### Web ACL Management
```bash
# Create Web ACL
aws wafv2 create-web-acl \
  --name "MyWebACL" \
  --scope REGIONAL \
  --default-action Allow={}

# List Web ACLs
aws wafv2 list-web-acls --scope REGIONAL

# Get Web ACL details
aws wafv2 get-web-acl \
  --scope REGIONAL \
  --id "12345678-1234-1234-1234-123456789012"

# Delete Web ACL
aws wafv2 delete-web-acl \
  --scope REGIONAL \
  --id "12345678-1234-1234-1234-123456789012" \
  --lock-token "lock-token"
```

### IP Set Management
```bash
# Create IP set
aws wafv2 create-ip-set \
  --name "BlockedIPs" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "192.0.2.0/24" "203.0.113.0/24"

# Update IP set
aws wafv2 update-ip-set \
  --scope REGIONAL \
  --id "ip-set-id" \
  --addresses "192.0.2.0/24" "203.0.113.0/24" "198.51.100.0/24" \
  --lock-token "lock-token"
```

---

## 15. Exam Tips

### Key Points to Remember
- WAF operates at Layer 7 (application layer)
- Shield Standard is free and automatic
- Shield Advanced costs $3,000/month with DRT support
- WAF rules are evaluated in priority order
- Rate-based rules aggregate over 5-minute windows
- Managed rule groups provide OWASP protection

### Common Mistakes
- Not testing rules in count mode first
- Incorrect rule priority ordering
- Forgetting to associate Web ACL with resources
- Not monitoring false positives
- Overlooking cost implications of rule complexity

### Best Practices for Exam
- Understand WAF vs Shield differences
- Know integration patterns with AWS services
- Understand rule types and use cases
- Know monitoring and logging capabilities
- Understand cost optimization strategies
- Know automation and response patterns