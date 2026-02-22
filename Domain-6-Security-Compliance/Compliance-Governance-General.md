# AWS Compliance and Governance - DOP-C02 Study Notes

## 1. Overview

### AWS Compliance Framework
- Shared responsibility model for compliance
- AWS manages infrastructure compliance
- Customers responsible for data and application compliance
- Extensive compliance certifications and attestations

### Key Compliance Programs
- **SOC 1/2/3** - Service Organization Control reports
- **PCI DSS** - Payment Card Industry Data Security Standard
- **HIPAA** - Health Insurance Portability and Accountability Act
- **FedRAMP** - Federal Risk and Authorization Management Program
- **ISO 27001** - Information Security Management System
- **GDPR** - General Data Protection Regulation

### Governance Principles
- **Accountability** - Clear ownership and responsibility
- **Transparency** - Visibility into operations and decisions
- **Compliance** - Adherence to regulations and standards
- **Risk Management** - Identification and mitigation of risks
- **Continuous Improvement** - Regular assessment and enhancement

---

## 2. AWS Config for Compliance

### What is AWS Config?
- Configuration management and compliance service
- Tracks resource configurations and changes
- Evaluates configurations against compliance rules
- Provides configuration history and change notifications

### Config Rules
- **AWS Managed Rules** - Pre-built compliance checks
- **Custom Rules** - Lambda-based custom logic
- **Conformance Packs** - Collection of related rules
- **Organizational Rules** - Rules applied across AWS Organizations

### Common Compliance Rules
```bash
# Enable required tags rule
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"Environment\",\"tag2Key\":\"Owner\"}"
  }'
```

### Remediation Actions
```yaml
# CloudFormation template for Config remediation
Resources:
  RemediationConfiguration:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref ConfigRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-RemoveUnrestrictedSourceInSecurityGroup
      TargetVersion: "1"
      Parameters:
        AutomationAssumeRole:
          StaticValue: !GetAtt RemediationRole.Arn
        GroupId:
          ResourceValue: RESOURCE_ID
```

---

## 3. AWS Organizations for Governance

### Organizational Structure
- **Management Account** - Central billing and administration
- **Member Accounts** - Individual AWS accounts in organization
- **Organizational Units (OUs)** - Logical groupings of accounts
- **Service Control Policies (SCPs)** - Guardrails for account permissions

### Service Control Policies
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalType": "Root"
        }
      }
    },
    {
      "Sid": "RequireSSLRequestsOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::*/*",
        "arn:aws:s3:::*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### Account Management
```bash
# Create new account in organization
aws organizations create-account \
  --email "newaccount@example.com" \
  --account-name "Production Account" \
  --role-name "OrganizationAccountAccessRole"

# Move account to OU
aws organizations move-account \
  --account-id "123456789012" \
  --source-parent-id "r-examplerootid111" \
  --destination-parent-id "ou-exampleou111"
```

---

## 4. AWS Control Tower

### What is Control Tower?
- Landing zone setup and governance service
- Pre-configured multi-account environment
- Guardrails for compliance and security
- Account Factory for standardized account provisioning

### Guardrails
- **Preventive Guardrails** - SCPs that prevent actions
- **Detective Guardrails** - Config rules that detect violations
- **Mandatory Guardrails** - Cannot be disabled
- **Strongly Recommended** - Best practice guardrails
- **Elective Guardrails** - Optional additional controls

### Account Factory
```yaml
# Service Catalog product for new accounts
Parameters:
  AccountName:
    Type: String
    Description: Name for the new account
  AccountEmail:
    Type: String
    Description: Email address for the new account
  OrganizationalUnit:
    Type: String
    Description: OU to place the account in
    AllowedValues:
      - Production
      - Development
      - Sandbox
```

### Customizations
```python
# Custom Control Tower customization
import boto3

def deploy_custom_guardrail():
    """Deploy custom guardrail across organization"""
    
    config = boto3.client('config')
    organizations = boto3.client('organizations')
    
    # Get all accounts in organization
    accounts = organizations.list_accounts()
    
    for account in accounts['Accounts']:
        if account['Status'] == 'ACTIVE':
            # Deploy Config rule to each account
            deploy_config_rule_to_account(account['Id'])

def deploy_config_rule_to_account(account_id):
    """Deploy Config rule to specific account"""
    # Implementation for cross-account Config rule deployment
    pass
```

---

## 5. AWS CloudTrail for Audit

### CloudTrail Configuration
- **Management Events** - Control plane operations
- **Data Events** - Data plane operations (S3, Lambda)
- **Insight Events** - Unusual activity patterns
- **Multi-Region Trails** - Global event logging

### Compliance Logging
```bash
# Create compliance trail
aws cloudtrail create-trail \
  --name "ComplianceTrail" \
  --s3-bucket-name "compliance-audit-logs" \
  --s3-key-prefix "cloudtrail-logs/" \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
```

### Log Analysis
```python
import boto3
import json

def analyze_compliance_events():
    """Analyze CloudTrail events for compliance violations"""
    
    s3 = boto3.client('s3')
    
    # List CloudTrail log files
    response = s3.list_objects_v2(
        Bucket='compliance-audit-logs',
        Prefix='cloudtrail-logs/'
    )
    
    violations = []
    
    for obj in response.get('Contents', []):
        # Download and analyze log file
        log_data = s3.get_object(Bucket='compliance-audit-logs', Key=obj['Key'])
        
        # Parse CloudTrail events
        events = json.loads(log_data['Body'].read())
        
        for event in events.get('Records', []):
            if is_compliance_violation(event):
                violations.append(event)
    
    return violations

def is_compliance_violation(event):
    """Check if event represents compliance violation"""
    # Implement compliance checking logic
    return False
```

---

## 6. Data Protection and Privacy

### Data Classification
- **Public** - No restrictions on access
- **Internal** - Restricted to organization
- **Confidential** - Restricted to specific individuals
- **Restricted** - Highest level of protection

### Data Encryption
```yaml
# CloudFormation template for encrypted resources
Resources:
  EncryptedS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref KMSKey
            BucketKeyEnabled: true
      
  EncryptedRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      StorageEncrypted: true
      KmsKeyId: !Ref KMSKey
      
  EncryptedEBSVolume:
    Type: AWS::EC2::Volume
    Properties:
      Encrypted: true
      KmsKeyId: !Ref KMSKey
```

### Data Loss Prevention
```python
import boto3

def implement_dlp_controls():
    """Implement data loss prevention controls"""
    
    # Enable GuardDuty for threat detection
    guardduty = boto3.client('guardduty')
    guardduty.create_detector(Enable=True)
    
    # Enable Macie for data discovery
    macie = boto3.client('macie2')
    macie.enable_macie()
    
    # Configure CloudWatch for monitoring
    cloudwatch = boto3.client('cloudwatch')
    cloudwatch.put_metric_alarm(
        AlarmName='UnauthorizedDataAccess',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=1,
        MetricName='UnauthorizedAPICallsCount',
        Namespace='CloudTrailMetrics',
        Period=300,
        Statistic='Sum',
        Threshold=10.0,
        ActionsEnabled=True,
        AlarmActions=['arn:aws:sns:us-east-1:123456789012:security-alerts']
    )
```

---

## 7. Identity and Access Management Governance

### IAM Best Practices
- **Least Privilege** - Minimum necessary permissions
- **Regular Reviews** - Periodic access reviews
- **Role-Based Access** - Use roles instead of users
- **MFA Enforcement** - Multi-factor authentication
- **Password Policies** - Strong password requirements

### Access Reviews
```python
import boto3
from datetime import datetime, timedelta

def perform_access_review():
    """Perform regular IAM access review"""
    
    iam = boto3.client('iam')
    
    # Get all users
    users = iam.list_users()
    
    inactive_users = []
    
    for user in users['Users']:
        username = user['UserName']
        
        # Check last activity
        last_used = get_user_last_activity(username)
        
        if last_used and (datetime.now() - last_used).days > 90:
            inactive_users.append(username)
    
    return inactive_users

def get_user_last_activity(username):
    """Get user's last activity date"""
    iam = boto3.client('iam')
    
    try:
        response = iam.get_user(UserName=username)
        return response['User'].get('PasswordLastUsed')
    except:
        return None
```

### Permission Boundaries
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::allowed-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 8. Compliance Monitoring and Reporting

### Automated Compliance Checks
```python
import boto3

def run_compliance_checks():
    """Run automated compliance checks"""
    
    config = boto3.client('config')
    
    # Trigger Config rule evaluations
    rules = config.describe_config_rules()
    
    results = {}
    
    for rule in rules['ConfigRules']:
        rule_name = rule['ConfigRuleName']
        
        # Get compliance status
        compliance = config.get_compliance_details_by_config_rule(
            ConfigRuleName=rule_name
        )
        
        results[rule_name] = {
            'compliant': len([r for r in compliance['EvaluationResults'] 
                            if r['ComplianceType'] == 'COMPLIANT']),
            'non_compliant': len([r for r in compliance['EvaluationResults'] 
                                if r['ComplianceType'] == 'NON_COMPLIANT'])
        }
    
    return results
```

### Compliance Dashboard
```python
import boto3
import json

def generate_compliance_dashboard():
    """Generate compliance dashboard data"""
    
    # Collect compliance data from various sources
    config_compliance = get_config_compliance()
    security_findings = get_security_hub_findings()
    access_analyzer_findings = get_access_analyzer_findings()
    
    dashboard_data = {
        'timestamp': datetime.now().isoformat(),
        'config_compliance': config_compliance,
        'security_findings': security_findings,
        'access_findings': access_analyzer_findings,
        'overall_score': calculate_compliance_score(
            config_compliance, security_findings, access_analyzer_findings
        )
    }
    
    return dashboard_data

def calculate_compliance_score(config_data, security_data, access_data):
    """Calculate overall compliance score"""
    # Implementation for compliance scoring
    return 85.5  # Example score
```

---

## 9. Incident Response and Forensics

### Incident Response Plan
1. **Detection** - Identify security incidents
2. **Analysis** - Assess impact and scope
3. **Containment** - Limit damage and spread
4. **Eradication** - Remove threats
5. **Recovery** - Restore normal operations
6. **Lessons Learned** - Improve processes

### Forensic Data Collection
```python
import boto3

def collect_forensic_data(incident_id, start_time, end_time):
    """Collect forensic data for incident investigation"""
    
    # Collect CloudTrail logs
    cloudtrail_logs = collect_cloudtrail_logs(start_time, end_time)
    
    # Collect VPC Flow Logs
    vpc_flow_logs = collect_vpc_flow_logs(start_time, end_time)
    
    # Collect GuardDuty findings
    guardduty_findings = collect_guardduty_findings(start_time, end_time)
    
    # Create forensic package
    forensic_package = {
        'incident_id': incident_id,
        'collection_time': datetime.now().isoformat(),
        'cloudtrail_logs': cloudtrail_logs,
        'vpc_flow_logs': vpc_flow_logs,
        'guardduty_findings': guardduty_findings
    }
    
    # Store in secure S3 bucket
    store_forensic_data(forensic_package)
    
    return forensic_package
```

### Automated Response
```python
import boto3

def automated_incident_response(finding):
    """Automated response to security findings"""
    
    if finding['severity'] == 'HIGH':
        # Isolate affected resources
        isolate_compromised_instance(finding['resource_id'])
        
        # Create snapshot for forensics
        create_forensic_snapshot(finding['resource_id'])
        
        # Notify security team
        send_security_alert(finding)
    
    elif finding['severity'] == 'MEDIUM':
        # Log for investigation
        log_security_event(finding)
        
        # Apply remediation if available
        apply_automated_remediation(finding)

def isolate_compromised_instance(instance_id):
    """Isolate compromised EC2 instance"""
    ec2 = boto3.client('ec2')
    
    # Create isolation security group
    isolation_sg = create_isolation_security_group()
    
    # Apply isolation security group
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[isolation_sg]
    )
```

---

## 10. Regulatory Compliance Frameworks

### GDPR Compliance
- **Data Protection by Design** - Privacy built into systems
- **Data Subject Rights** - Access, rectification, erasure
- **Data Processing Records** - Documentation of processing activities
- **Data Protection Impact Assessments** - Risk assessments

### HIPAA Compliance
- **Administrative Safeguards** - Policies and procedures
- **Physical Safeguards** - Physical access controls
- **Technical Safeguards** - Access controls and encryption
- **Business Associate Agreements** - Third-party compliance

### PCI DSS Compliance
- **Build and Maintain Secure Networks** - Firewalls and security
- **Protect Cardholder Data** - Encryption and access controls
- **Maintain Vulnerability Management** - Security testing and updates
- **Implement Strong Access Control** - Authentication and authorization

---

## 11. Common Exam Scenarios

### Scenario 1: Implement organization-wide compliance monitoring
**Solution:**
- Set up AWS Organizations with appropriate OUs
- Deploy AWS Config across all accounts
- Implement Config rules for compliance requirements
- Set up automated remediation actions
- Create compliance reporting dashboard

### Scenario 2: Ensure data encryption compliance
**Solution:**
- Implement KMS key policies for encryption
- Enable encryption at rest for all data stores
- Configure encryption in transit for all communications
- Set up Config rules to monitor encryption compliance
- Implement automated remediation for non-compliant resources

### Scenario 3: Implement access governance
**Solution:**
- Set up IAM policies with least privilege
- Implement permission boundaries
- Enable AWS Access Analyzer
- Set up regular access reviews
- Implement automated user lifecycle management

### Scenario 4: Create audit trail for compliance
**Solution:**
- Enable CloudTrail in all regions
- Configure log file validation
- Set up log aggregation in central account
- Implement log analysis and alerting
- Create compliance reporting from audit logs

### Scenario 5: Implement incident response procedures
**Solution:**
- Set up GuardDuty for threat detection
- Configure Security Hub for centralized findings
- Implement automated response workflows
- Set up forensic data collection procedures
- Create incident response playbooks

---

## 12. Exam Tips

### Key Points to Remember
- Shared responsibility model applies to compliance
- AWS Config provides configuration compliance monitoring
- Organizations and Control Tower provide governance at scale
- CloudTrail provides audit trails for compliance
- Encryption is required for most compliance frameworks
- Regular access reviews are essential for governance

### Common Mistakes
- Not understanding shared responsibility for compliance
- Forgetting to enable CloudTrail in all regions
- Not implementing proper data classification
- Overlooking the need for regular access reviews
- Not automating compliance monitoring and remediation

### Best Practices for Exam
- Understand compliance frameworks and their requirements
- Know AWS services that support compliance
- Understand governance patterns and best practices
- Know incident response and forensics procedures
- Understand data protection and privacy requirements
- Know automation patterns for compliance monitoring