# AWS Security Hub - DOP-C02 Exam Notes

## 1. Overview

**AWS Security Hub** provides a comprehensive view of your security posture across your AWS accounts. It aggregates, organizes, and prioritizes security findings from multiple AWS security services and third-party tools.

### Key Characteristics
- **Centralized security dashboard** - Single pane of glass for security findings
- **Multi-service integration** - Aggregates findings from GuardDuty, Inspector, Macie, and more
- **Security standards compliance** - Built-in compliance checks for industry standards
- **Custom insights** - Create custom views and filters for security data
- **Automated remediation** - Integration with Lambda and Systems Manager for response
- **Cross-account management** - Centralized security monitoring across AWS Organizations
- **Third-party integrations** - Support for partner security tools and services

### What Problem Does It Solve?
- Eliminates security tool silos by centralizing findings
- Provides consistent security posture visibility across accounts
- Automates compliance checking against security standards
- Enables faster incident response and remediation
- Reduces operational overhead of managing multiple security tools
- Facilitates security reporting and audit requirements

---

## 2. Core Components

### Security Hub Setup
```yaml
# CloudFormation for Security Hub configuration
Resources:
  SecurityHub:
    Type: AWS::SecurityHub::Hub
    Properties:
      Tags:
        - Key: Purpose
          Value: CentralizedSecurityMonitoring
        - Key: Environment
          Value: Production

  # Enable security standards
  CISStandard:
    Type: AWS::SecurityHub::Standard
    Properties:
      StandardsArn: !Sub "arn:aws:securityhub:::ruleset/finding-format/aws-foundational-security-standard/v/1.0.0"
      DisabledStandardsControls:
        - StandardsControlArn: !Sub "arn:aws:securityhub:${AWS::Region}:${AWS::AccountId}:control/aws-foundational-security-standard/v/1.0.0/S3.1"
          DisabledReason: "S3 bucket public access is required for static website hosting"

  PCIStandard:
    Type: AWS::SecurityHub::Standard
    Properties:
      StandardsArn: !Sub "arn:aws:securityhub:::ruleset/finding-format/pci-dss/v/3.2.1"

  # Product subscriptions
  GuardDutySubscription:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/guardduty"

  InspectorSubscription:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/inspector"

  ConfigSubscription:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/config"

  MacieSubscription:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/macie"
```

### Custom Insights
```yaml
Resources:
  # High severity findings insight
  HighSeverityInsight:
    Type: AWS::SecurityHub::Insight
    Properties:
      Name: High Severity Security Findings
      Description: All high and critical severity findings across services
      GroupByAttribute: ProductFields/aws/securityhub/ProductName
      Filters:
        SeverityLabel:
          - Value: HIGH
            Comparison: EQUALS
          - Value: CRITICAL
            Comparison: EQUALS
        RecordState:
          - Value: ACTIVE
            Comparison: EQUALS

  # Non-compliant resources insight
  ComplianceInsight:
    Type: AWS::SecurityHub::Insight
    Properties:
      Name: Non-Compliant Resources
      Description: Resources failing compliance checks
      GroupByAttribute: ResourceType
      Filters:
        ComplianceStatus:
          - Value: FAILED
            Comparison: EQUALS
        RecordState:
          - Value: ACTIVE
            Comparison: EQUALS

  # Network security findings
  NetworkSecurityInsight:
    Type: AWS::SecurityHub::Insight
    Properties:
      Name: Network Security Issues
      Description: Network-related security findings
      GroupByAttribute: ResourceId
      Filters:
        Type:
          - Value: "Effects/Data Exfiltration"
            Comparison: PREFIX
          - Value: "Unusual Behaviors/Network"
            Comparison: PREFIX
        RecordState:
          - Value: ACTIVE
            Comparison: EQUALS

  # Unresolved findings by age
  AgingFindingsInsight:
    Type: AWS::SecurityHub::Insight
    Properties:
      Name: Aging Security Findings
      Description: Findings older than 30 days
      GroupByAttribute: SeverityLabel
      Filters:
        CreatedAt:
          - DateRange:
              Unit: DAYS
              Value: 30
            Comparison: GREATER_THAN
        RecordState:
          - Value: ACTIVE
            Comparison: EQUALS
```

---

## 3. Multi-Account Management

### Organization Integration
```python
import boto3
import json
from datetime import datetime, timedelta

class SecurityHubOrganizationManager:
    def __init__(self):
        self.securityhub_client = boto3.client('securityhub')
        self.organizations_client = boto3.client('organizations')
        self.sts_client = boto3.client('sts')
    
    def setup_organization_security_hub(self):
        """Setup Security Hub across AWS Organization"""
        
        # Enable Security Hub in master account
        master_setup = self.enable_security_hub_master()
        
        # Get organization accounts
        accounts = self.get_organization_accounts()
        
        # Enable Security Hub in member accounts
        member_results = {}
        
        for account in accounts:
            if account['Status'] == 'ACTIVE' and account['Id'] != self.get_master_account_id():
                result = self.enable_security_hub_member(account['Id'], account['Email'])
                member_results[account['Id']] = result
        
        # Configure organization settings
        org_config = self.configure_organization_settings()
        
        return {
            'master_setup': master_setup,
            'member_results': member_results,
            'organization_config': org_config
        }
    
    def enable_security_hub_master(self):
        """Enable Security Hub in master account"""
        
        try:
            # Enable Security Hub
            self.securityhub_client.enable_security_hub(
                Tags={
                    'Purpose': 'OrganizationSecurityHub',
                    'ManagedBy': 'SecurityTeam'
                },
                EnableDefaultStandards=True
            )
            
            # Enable product integrations
            product_arns = [
                f"arn:aws:securityhub:{boto3.Session().region_name}::product/aws/guardduty",
                f"arn:aws:securityhub:{boto3.Session().region_name}::product/aws/inspector",
                f"arn:aws:securityhub:{boto3.Session().region_name}::product/aws/config",
                f"arn:aws:securityhub:{boto3.Session().region_name}::product/aws/macie"
            ]
            
            for product_arn in product_arns:
                try:
                    self.securityhub_client.enable_import_findings_for_product(
                        ProductArn=product_arn
                    )
                except Exception as e:
                    print(f"Error enabling product {product_arn}: {str(e)}")
            
            return {
                'status': 'SUCCESS',
                'enabled_products': len(product_arns)
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'error': str(e)
            }
    
    def enable_security_hub_member(self, account_id, email):
        """Enable Security Hub in member account and invite"""
        
        try:
            # Create member
            self.securityhub_client.create_members(
                AccountDetails=[
                    {
                        'AccountId': account_id,
                        'Email': email
                    }
                ]
            )
            
            # Invite member
            self.securityhub_client.invite_members(
                AccountIds=[account_id],
                Message='Please accept Security Hub invitation for centralized security monitoring'
            )
            
            return {
                'status': 'INVITED',
                'account_id': account_id
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'account_id': account_id,
                'error': str(e)
            }
    
    def configure_organization_settings(self):
        """Configure organization-wide Security Hub settings"""
        
        try:
            # Enable organization configuration
            self.securityhub_client.update_organization_configuration(
                AutoEnable=True,
                AutoEnableStandards='DEFAULT'
            )
            
            return {
                'status': 'SUCCESS',
                'auto_enable': True
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'error': str(e)
            }
    
    def get_organization_accounts(self):
        """Get all accounts in the organization"""
        
        accounts = []
        paginator = self.organizations_client.get_paginator('list_accounts')
        
        for page in paginator.paginate():
            accounts.extend(page['Accounts'])
        
        return accounts
    
    def get_master_account_id(self):
        """Get master account ID"""
        
        return self.sts_client.get_caller_identity()['Account']
    
    def aggregate_findings_across_accounts(self):
        """Aggregate Security Hub findings across all member accounts"""
        
        aggregated_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'total_findings': 0,
            'severity_breakdown': {
                'CRITICAL': 0,
                'HIGH': 0,
                'MEDIUM': 0,
                'LOW': 0,
                'INFORMATIONAL': 0
            },
            'compliance_breakdown': {
                'PASSED': 0,
                'FAILED': 0,
                'WARNING': 0,
                'NOT_AVAILABLE': 0
            },
            'account_summaries': {}
        }
        
        try:
            # Get findings from all accounts
            paginator = self.securityhub_client.get_paginator('get_findings')
            
            for page in paginator.paginate(
                Filters={
                    'RecordState': [
                        {
                            'Value': 'ACTIVE',
                            'Comparison': 'EQUALS'
                        }
                    ]
                }
            ):
                for finding in page['Findings']:
                    aggregated_data['total_findings'] += 1
                    
                    # Count by severity
                    severity = finding.get('Severity', {}).get('Label', 'INFORMATIONAL')
                    aggregated_data['severity_breakdown'][severity] += 1
                    
                    # Count by compliance
                    compliance = finding.get('Compliance', {}).get('Status', 'NOT_AVAILABLE')
                    aggregated_data['compliance_breakdown'][compliance] += 1
                    
                    # Track by account
                    account_id = finding.get('AwsAccountId', 'Unknown')
                    if account_id not in aggregated_data['account_summaries']:
                        aggregated_data['account_summaries'][account_id] = {
                            'total_findings': 0,
                            'critical_findings': 0,
                            'high_findings': 0
                        }
                    
                    aggregated_data['account_summaries'][account_id]['total_findings'] += 1
                    
                    if severity == 'CRITICAL':
                        aggregated_data['account_summaries'][account_id]['critical_findings'] += 1
                    elif severity == 'HIGH':
                        aggregated_data['account_summaries'][account_id]['high_findings'] += 1
        
        except Exception as e:
            aggregated_data['error'] = str(e)
        
        return aggregated_data
```

---

## 4. Automated Response and Remediation

### Lambda-based Response Function
```python
# Lambda function for Security Hub automated response
import boto3
import json
from datetime import datetime

def lambda_handler(event, context):
    """
    Automated response to Security Hub findings
    Triggered by CloudWatch Events when findings are created or updated
    """
    
    # Parse Security Hub finding from CloudWatch Event
    findings = event['detail']['findings']
    
    response_results = []
    
    for finding in findings:
        try:
            response_result = process_security_finding(finding)
            response_results.append(response_result)
        except Exception as e:
            response_results.append({
                'finding_id': finding.get('Id'),
                'status': 'ERROR',
                'error': str(e)
            })
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'processed_findings': len(findings),
            'response_results': response_results
        })
    }

def process_security_finding(finding):
    """Process individual Security Hub finding"""
    
    finding_id = finding.get('Id')
    severity = finding.get('Severity', {}).get('Label', 'INFORMATIONAL')
    product_name = finding.get('ProductName', 'Unknown')
    finding_type = finding.get('Types', ['Unknown'])[0]
    
    print(f"Processing finding: {finding_id} ({severity}) from {product_name}")
    
    # Initialize AWS clients
    ec2 = boto3.client('ec2')
    iam = boto3.client('iam')
    s3 = boto3.client('s3')
    ssm = boto3.client('ssm')
    sns = boto3.client('sns')
    
    response_actions = []
    
    # Critical and High severity findings require immediate action
    if severity in ['CRITICAL', 'HIGH']:
        response_actions.extend(handle_high_severity_finding(finding, ec2, iam, s3, ssm))
    
    # Compliance failures
    if finding.get('Compliance', {}).get('Status') == 'FAILED':
        response_actions.extend(handle_compliance_failure(finding, ec2, iam, s3, ssm))
    
    # Product-specific handling
    if product_name == 'GuardDuty':
        response_actions.extend(handle_guardduty_finding(finding, ec2, iam))
    elif product_name == 'Inspector':
        response_actions.extend(handle_inspector_finding(finding, ssm))
    elif product_name == 'Config':
        response_actions.extend(handle_config_finding(finding, ec2, s3))
    
    # Send notifications
    send_finding_notification(finding, response_actions, sns)
    
    return {
        'finding_id': finding_id,
        'severity': severity,
        'product_name': product_name,
        'response_actions': response_actions,
        'status': 'PROCESSED'
    }

def handle_high_severity_finding(finding, ec2, iam, s3, ssm):
    """Handle critical and high severity findings"""
    
    actions = []
    resources = finding.get('Resources', [])
    
    for resource in resources:
        resource_type = resource.get('Type', '')
        resource_id = resource.get('Id', '')
        
        # EC2 instance security issues
        if resource_type == 'AwsEc2Instance':
            instance_id = resource_id.split('/')[-1]
            
            # Isolate instance if it's a security threat
            if any(threat in finding.get('Title', '') for threat in ['Backdoor', 'Malware', 'Cryptocurrency']):
                action = isolate_ec2_instance(ec2, instance_id)
                actions.append(action)
        
        # S3 bucket security issues
        elif resource_type == 'AwsS3Bucket':
            bucket_name = resource_id.split('/')[-1]
            
            # Enable public access block for exposed buckets
            if 'public' in finding.get('Title', '').lower():
                action = enable_s3_public_access_block(s3, bucket_name)
                actions.append(action)
        
        # IAM security issues
        elif resource_type in ['AwsIamUser', 'AwsIamRole']:
            # Disable compromised IAM entities
            if 'compromised' in finding.get('Description', '').lower():
                action = disable_iam_entity(iam, resource_type, resource_id)
                actions.append(action)
    
    return actions

def handle_compliance_failure(finding, ec2, iam, s3, ssm):
    """Handle compliance failures"""
    
    actions = []
    compliance_type = finding.get('Compliance', {}).get('StatusReasons', [{}])[0].get('ReasonCode', '')
    
    # Common compliance remediation patterns
    if 'ENCRYPTION' in compliance_type:
        actions.extend(remediate_encryption_compliance(finding, ec2, s3))
    elif 'ACCESS_CONTROL' in compliance_type:
        actions.extend(remediate_access_control_compliance(finding, iam, s3))
    elif 'LOGGING' in compliance_type:
        actions.extend(remediate_logging_compliance(finding, ssm))
    
    return actions

def handle_guardduty_finding(finding, ec2, iam):
    """Handle GuardDuty-specific findings"""
    
    actions = []
    finding_type = finding.get('Title', '')
    
    # Cryptocurrency mining
    if 'CryptoCurrency' in finding_type:
        resources = finding.get('Resources', [])
        for resource in resources:
            if resource.get('Type') == 'AwsEc2Instance':
                instance_id = resource.get('Id', '').split('/')[-1]
                action = stop_ec2_instance(ec2, instance_id, 'Cryptocurrency mining detected')
                actions.append(action)
    
    # Compromised credentials
    elif 'UnauthorizedAccess:IAMUser' in finding_type:
        # Disable access keys for compromised users
        user_name = extract_user_name_from_finding(finding)
        if user_name:
            action = disable_user_access_keys(iam, user_name)
            actions.append(action)
    
    return actions

def handle_inspector_finding(finding, ssm):
    """Handle Inspector-specific findings"""
    
    actions = []
    
    # Package vulnerabilities
    if 'Package Vulnerability' in finding.get('Title', ''):
        resources = finding.get('Resources', [])
        for resource in resources:
            if resource.get('Type') == 'AwsEc2Instance':
                instance_id = resource.get('Id', '').split('/')[-1]
                action = trigger_patch_installation(ssm, instance_id)
                actions.append(action)
    
    return actions

def handle_config_finding(finding, ec2, s3):
    """Handle Config-specific findings"""
    
    actions = []
    rule_name = finding.get('ProductFields', {}).get('aws/config/rule/name', '')
    
    # Security group rules
    if 'security-group' in rule_name:
        resources = finding.get('Resources', [])
        for resource in resources:
            if resource.get('Type') == 'AwsEc2SecurityGroup':
                sg_id = resource.get('Id', '').split('/')[-1]
                action = remediate_security_group(ec2, sg_id)
                actions.append(action)
    
    return actions

def isolate_ec2_instance(ec2, instance_id):
    """Isolate EC2 instance by creating restrictive security group"""
    
    try:
        # Get instance VPC
        instance_response = ec2.describe_instances(InstanceIds=[instance_id])
        vpc_id = instance_response['Reservations'][0]['Instances'][0]['VpcId']
        
        # Create isolation security group
        sg_response = ec2.create_security_group(
            GroupName=f'isolation-{instance_id}-{int(datetime.now().timestamp())}',
            Description=f'Isolation security group for compromised instance {instance_id}',
            VpcId=vpc_id
        )
        
        # Apply isolation security group
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[sg_response['GroupId']]
        )
        
        return {
            'action': 'Instance Isolation',
            'resource': instance_id,
            'status': 'SUCCESS',
            'security_group': sg_response['GroupId']
        }
    
    except Exception as e:
        return {
            'action': 'Instance Isolation',
            'resource': instance_id,
            'status': 'FAILED',
            'error': str(e)
        }

def enable_s3_public_access_block(s3, bucket_name):
    """Enable S3 public access block"""
    
    try:
        s3.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        
        return {
            'action': 'Enable S3 Public Access Block',
            'resource': bucket_name,
            'status': 'SUCCESS'
        }
    
    except Exception as e:
        return {
            'action': 'Enable S3 Public Access Block',
            'resource': bucket_name,
            'status': 'FAILED',
            'error': str(e)
        }

def send_finding_notification(finding, actions, sns):
    """Send notification about Security Hub finding and response"""
    
    severity = finding.get('Severity', {}).get('Label', 'INFORMATIONAL')
    
    message = f"""
    Security Hub Finding Alert
    =========================
    
    Finding ID: {finding.get('Id')}
    Title: {finding.get('Title')}
    Severity: {severity}
    Product: {finding.get('ProductName')}
    
    Description: {finding.get('Description')}
    
    Automated Response Actions:
    """
    
    for action in actions:
        message += f"\n- {action['action']}: {action['status']}"
        if action['status'] == 'FAILED':
            message += f" (Error: {action.get('error', 'Unknown')})"
    
    # Determine topic based on severity
    if severity in ['CRITICAL', 'HIGH']:
        topic_arn = 'arn:aws:sns:us-east-1:123456789012:critical-security-alerts'
        subject = f"CRITICAL: Security Hub Finding - {finding.get('Title')}"
    else:
        topic_arn = 'arn:aws:sns:us-east-1:123456789012:security-alerts'
        subject = f"Security Hub Finding - {finding.get('Title')}"
    
    sns.publish(
        TopicArn=topic_arn,
        Message=message,
        Subject=subject
    )
```

### CloudWatch Events Integration
```yaml
# CloudFormation for Security Hub event processing
Resources:
  SecurityHubEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: SecurityHubFindingProcessor
      Description: Process Security Hub findings for automated response
      EventPattern:
        source:
          - aws.securityhub
        detail-type:
          - Security Hub Findings - Imported
        detail:
          findings:
            Severity:
              Label:
                - HIGH
                - CRITICAL
      State: ENABLED
      Targets:
        - Arn: !GetAtt SecurityHubResponseFunction.Arn
          Id: SecurityHubResponseTarget

  # Compliance findings rule
  ComplianceFindingsRule:
    Type: AWS::Events::Rule
    Properties:
      Name: SecurityHubComplianceFindings
      Description: Process compliance-related findings
      EventPattern:
        source:
          - aws.securityhub
        detail-type:
          - Security Hub Findings - Imported
        detail:
          findings:
            Compliance:
              Status:
                - FAILED
      State: ENABLED
      Targets:
        - Arn: !GetAtt ComplianceRemediationFunction.Arn
          Id: ComplianceRemediationTarget

  # Lambda permission for EventBridge
  SecurityHubLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref SecurityHubResponseFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt SecurityHubEventRule.Arn
```

---

## 5. Custom Security Standards

### Custom Standard Creation
```python
# Create custom security standard
import boto3
import json

class CustomSecurityStandard:
    def __init__(self):
        self.securityhub_client = boto3.client('securityhub')
    
    def create_custom_standard(self, standard_name, controls):
        """Create custom security standard with controls"""
        
        try:
            # Create custom standard
            standard_response = self.securityhub_client.create_standard(
                Name=standard_name,
                Description=f"Custom security standard: {standard_name}",
                Controls=controls
            )
            
            return {
                'status': 'SUCCESS',
                'standard_arn': standard_response['StandardArn']
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'error': str(e)
            }
    
    def create_devops_security_standard(self):
        """Create DevOps-specific security standard"""
        
        devops_controls = [
            {
                'StandardsControlArn': 'arn:aws:securityhub:::control/custom/devops-1',
                'Title': 'CI/CD Pipeline Security',
                'Description': 'Ensure CI/CD pipelines follow security best practices',
                'RemediationUrl': 'https://docs.aws.amazon.com/codepipeline/latest/userguide/security.html',
                'SeverityRating': 'HIGH',
                'RelatedRequirements': ['DevOps.1', 'DevOps.2']
            },
            {
                'StandardsControlArn': 'arn:aws:securityhub:::control/custom/devops-2',
                'Title': 'Container Image Security',
                'Description': 'Container images must be scanned for vulnerabilities',
                'RemediationUrl': 'https://docs.aws.amazon.com/inspector/latest/userguide/inspector_container-image-scanning.html',
                'SeverityRating': 'HIGH',
                'RelatedRequirements': ['DevOps.3', 'DevOps.4']
            },
            {
                'StandardsControlArn': 'arn:aws:securityhub:::control/custom/devops-3',
                'Title': 'Infrastructure as Code Security',
                'Description': 'IaC templates must be validated for security issues',
                'RemediationUrl': 'https://docs.aws.amazon.com/cloudformation/latest/userguide/security-best-practices.html',
                'SeverityRating': 'MEDIUM',
                'RelatedRequirements': ['DevOps.5', 'DevOps.6']
            },
            {
                'StandardsControlArn': 'arn:aws:securityhub:::control/custom/devops-4',
                'Title': 'Secrets Management',
                'Description': 'Secrets must not be hardcoded in source code',
                'RemediationUrl': 'https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html',
                'SeverityRating': 'CRITICAL',
                'RelatedRequirements': ['DevOps.7', 'DevOps.8']
            }
        ]
        
        return self.create_custom_standard('DevOps Security Standard', devops_controls)
    
    def batch_import_custom_findings(self, findings):
        """Import custom findings to Security Hub"""
        
        try:
            response = self.securityhub_client.batch_import_findings(
                Findings=findings
            )
            
            return {
                'status': 'SUCCESS',
                'successful_count': response['SuccessCount'],
                'failed_count': response['FailedCount'],
                'failed_findings': response.get('FailedFindings', [])
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'error': str(e)
            }
    
    def create_pipeline_security_finding(self, pipeline_arn, issue_description, severity='MEDIUM'):
        """Create custom finding for CI/CD pipeline security issue"""
        
        finding = {
            'SchemaVersion': '2018-10-08',
            'Id': f"{pipeline_arn}/pipeline-security-check",
            'ProductArn': f"arn:aws:securityhub:{boto3.Session().region_name}:{boto3.Session().get_credentials().access_key[:12]}:product/custom/devops-security",
            'GeneratorId': 'custom-pipeline-scanner',
            'AwsAccountId': boto3.Session().get_credentials().access_key[:12],
            'Types': ['Software and Configuration Checks/Industry and Regulatory Standards/DevOps Security'],
            'CreatedAt': datetime.utcnow().isoformat() + 'Z',
            'UpdatedAt': datetime.utcnow().isoformat() + 'Z',
            'Severity': {
                'Label': severity
            },
            'Title': 'CI/CD Pipeline Security Issue',
            'Description': issue_description,
            'Resources': [
                {
                    'Type': 'AwsCodePipelinePipeline',
                    'Id': pipeline_arn,
                    'Region': boto3.Session().region_name
                }
            ],
            'Compliance': {
                'Status': 'FAILED'
            },
            'RecordState': 'ACTIVE',
            'WorkflowState': 'NEW'
        }
        
        return self.batch_import_custom_findings([finding])
```

---

## 6. Reporting and Analytics

### Security Metrics Dashboard
```python
# Security Hub metrics and reporting
import boto3
import json
from datetime import datetime, timedelta

class SecurityHubAnalytics:
    def __init__(self):
        self.securityhub_client = boto3.client('securityhub')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def generate_security_metrics(self):
        """Generate comprehensive security metrics"""
        
        metrics = {
            'timestamp': datetime.utcnow().isoformat(),
            'findings_summary': self.get_findings_summary(),
            'compliance_summary': self.get_compliance_summary(),
            'trend_analysis': self.get_trend_analysis(),
            'top_issues': self.get_top_security_issues(),
            'remediation_metrics': self.get_remediation_metrics()
        }
        
        # Send metrics to CloudWatch
        self.send_metrics_to_cloudwatch(metrics)
        
        return metrics
    
    def get_findings_summary(self):
        """Get summary of all active findings"""
        
        summary = {
            'total_findings': 0,
            'by_severity': {'CRITICAL': 0, 'HIGH': 0, 'MEDIUM': 0, 'LOW': 0, 'INFORMATIONAL': 0},
            'by_product': {},
            'by_compliance_status': {'PASSED': 0, 'FAILED': 0, 'WARNING': 0, 'NOT_AVAILABLE': 0}
        }
        
        try:
            paginator = self.securityhub_client.get_paginator('get_findings')
            
            for page in paginator.paginate(
                Filters={
                    'RecordState': [{'Value': 'ACTIVE', 'Comparison': 'EQUALS'}]
                }
            ):
                for finding in page['Findings']:
                    summary['total_findings'] += 1
                    
                    # Count by severity
                    severity = finding.get('Severity', {}).get('Label', 'INFORMATIONAL')
                    summary['by_severity'][severity] += 1
                    
                    # Count by product
                    product = finding.get('ProductName', 'Unknown')
                    summary['by_product'][product] = summary['by_product'].get(product, 0) + 1
                    
                    # Count by compliance status
                    compliance = finding.get('Compliance', {}).get('Status', 'NOT_AVAILABLE')
                    summary['by_compliance_status'][compliance] += 1
        
        except Exception as e:
            summary['error'] = str(e)
        
        return summary
    
    def get_compliance_summary(self):
        """Get compliance summary across all standards"""
        
        compliance_summary = {
            'standards': {},
            'overall_score': 0
        }
        
        try:
            # Get enabled standards
            standards_response = self.securityhub_client.get_enabled_standards()
            
            for standard in standards_response['StandardsSubscriptions']:
                standard_arn = standard['StandardsArn']
                standard_name = standard_arn.split('/')[-1]
                
                # Get controls for this standard
                controls_response = self.securityhub_client.describe_standards_controls(
                    StandardsSubscriptionArn=standard['StandardsSubscriptionArn']
                )
                
                standard_compliance = {
                    'total_controls': len(controls_response['Controls']),
                    'enabled_controls': 0,
                    'passed_controls': 0,
                    'failed_controls': 0,
                    'compliance_score': 0
                }
                
                for control in controls_response['Controls']:
                    if control['ControlStatus'] == 'ENABLED':
                        standard_compliance['enabled_controls'] += 1
                        
                        # Check control compliance (simplified - would need actual findings analysis)
                        # This is a placeholder for actual compliance checking logic
                        if control.get('ControlStatusUpdatedAt'):  # Assume passed if recently updated
                            standard_compliance['passed_controls'] += 1
                        else:
                            standard_compliance['failed_controls'] += 1
                
                # Calculate compliance score
                if standard_compliance['enabled_controls'] > 0:
                    standard_compliance['compliance_score'] = (
                        standard_compliance['passed_controls'] / 
                        standard_compliance['enabled_controls']
                    ) * 100
                
                compliance_summary['standards'][standard_name] = standard_compliance
            
            # Calculate overall compliance score
            total_controls = sum(s['enabled_controls'] for s in compliance_summary['standards'].values())
            total_passed = sum(s['passed_controls'] for s in compliance_summary['standards'].values())
            
            if total_controls > 0:
                compliance_summary['overall_score'] = (total_passed / total_controls) * 100
        
        except Exception as e:
            compliance_summary['error'] = str(e)
        
        return compliance_summary
    
    def get_trend_analysis(self, days_back=30):
        """Analyze security trends over time"""
        
        end_date = datetime.utcnow()
        start_date = end_date - timedelta(days=days_back)
        
        trend_data = {
            'period': f"{start_date.date()} to {end_date.date()}",
            'daily_findings': {},
            'severity_trends': {},
            'product_trends': {}
        }
        
        try:
            # Get findings created in the time period
            paginator = self.securityhub_client.get_paginator('get_findings')
            
            for page in paginator.paginate(
                Filters={
                    'CreatedAt': [
                        {
                            'Start': start_date.isoformat() + 'Z',
                            'End': end_date.isoformat() + 'Z'
                        }
                    ]
                }
            ):
                for finding in page['Findings']:
                    created_date = finding.get('CreatedAt', '').split('T')[0]
                    severity = finding.get('Severity', {}).get('Label', 'INFORMATIONAL')
                    product = finding.get('ProductName', 'Unknown')
                    
                    # Daily findings count
                    trend_data['daily_findings'][created_date] = trend_data['daily_findings'].get(created_date, 0) + 1
                    
                    # Severity trends
                    if severity not in trend_data['severity_trends']:
                        trend_data['severity_trends'][severity] = {}
                    trend_data['severity_trends'][severity][created_date] = trend_data['severity_trends'][severity].get(created_date, 0) + 1
                    
                    # Product trends
                    if product not in trend_data['product_trends']:
                        trend_data['product_trends'][product] = {}
                    trend_data['product_trends'][product][created_date] = trend_data['product_trends'][product].get(created_date, 0) + 1
        
        except Exception as e:
            trend_data['error'] = str(e)
        
        return trend_data
    
    def send_metrics_to_cloudwatch(self, metrics):
        """Send security metrics to CloudWatch"""
        
        metric_data = []
        
        # Findings metrics
        findings_summary = metrics.get('findings_summary', {})
        
        metric_data.append({
            'MetricName': 'TotalFindings',
            'Value': findings_summary.get('total_findings', 0),
            'Unit': 'Count',
            'Timestamp': datetime.utcnow()
        })
        
        # Severity metrics
        for severity, count in findings_summary.get('by_severity', {}).items():
            metric_data.append({
                'MetricName': f'{severity}SeverityFindings',
                'Value': count,
                'Unit': 'Count',
                'Timestamp': datetime.utcnow()
            })
        
        # Compliance metrics
        compliance_summary = metrics.get('compliance_summary', {})
        
        metric_data.append({
            'MetricName': 'OverallComplianceScore',
            'Value': compliance_summary.get('overall_score', 0),
            'Unit': 'Percent',
            'Timestamp': datetime.utcnow()
        })
        
        # Send to CloudWatch
        try:
            self.cloudwatch.put_metric_data(
                Namespace='SecurityHub/Metrics',
                MetricData=metric_data
            )
        except Exception as e:
            print(f"Error sending metrics to CloudWatch: {str(e)}")
```

---

## 7. Common Exam Scenarios

### Scenario 1: Complete Security Hub Implementation
```yaml
# Complete Security Hub setup with all integrations
Resources:
  # Enable Security Hub
  SecurityHub:
    Type: AWS::SecurityHub::Hub
    Properties:
      Tags:
        - Key: Purpose
          Value: CentralizedSecurityMonitoring

  # Enable all security standards
  AWSFoundationalStandard:
    Type: AWS::SecurityHub::Standard
    Properties:
      StandardsArn: !Sub "arn:aws:securityhub:::ruleset/finding-format/aws-foundational-security-standard/v/1.0.0"

  CISStandard:
    Type: AWS::SecurityHub::Standard
    Properties:
      StandardsArn: !Sub "arn:aws:securityhub:::ruleset/finding-format/cis-aws-foundations-benchmark/v/1.2.0"

  PCIStandard:
    Type: AWS::SecurityHub::Standard
    Properties:
      StandardsArn: !Sub "arn:aws:securityhub:::ruleset/finding-format/pci-dss/v/3.2.1"

  # Enable all product integrations
  GuardDutyIntegration:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/guardduty"

  InspectorIntegration:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/inspector"

  ConfigIntegration:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/config"

  MacieIntegration:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/macie"

  # Automated response system
  SecurityHubResponseFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: security-hub-automated-response
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt SecurityHubResponseRole.Arn
      Timeout: 300
      Code:
        ZipFile: |
          # Complete automated response code here
          import boto3
          import json
          
          def lambda_handler(event, context):
              # Process Security Hub findings and trigger appropriate responses
              return process_security_hub_findings(event)

  # EventBridge rules for different finding types
  CriticalFindingsRule:
    Type: AWS::Events::Rule
    Properties:
      EventPattern:
        source: [aws.securityhub]
        detail-type: [Security Hub Findings - Imported]
        detail:
          findings:
            Severity:
              Label: [CRITICAL]
      Targets:
        - Arn: !GetAtt SecurityHubResponseFunction.Arn
          Id: CriticalFindingsTarget

  ComplianceFailuresRule:
    Type: AWS::Events::Rule
    Properties:
      EventPattern:
        source: [aws.securityhub]
        detail-type: [Security Hub Findings - Imported]
        detail:
          findings:
            Compliance:
              Status: [FAILED]
      Targets:
        - Arn: !GetAtt ComplianceRemediationFunction.Arn
          Id: ComplianceFailuresTarget
```

---

## 8. Exam Tips

- **Understand centralized security** - Security Hub as single pane of glass
- **Master multi-account setup** - Organization integration and member management
- **Know product integrations** - GuardDuty, Inspector, Config, Macie integration
- **Practice automated response** - EventBridge and Lambda integration patterns
- **Learn security standards** - AWS Foundational, CIS, PCI DSS standards
- **Understand custom insights** - Creating custom views and filters
- **Know finding formats** - ASFF (AWS Security Finding Format) structure
- **Practice remediation workflows** - Automated and manual remediation patterns
- **Master troubleshooting** - Common issues with integrations and findings
- **Understand compliance reporting** - Standards compliance and audit requirements