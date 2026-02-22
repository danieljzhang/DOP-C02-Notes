# AWS Config - DOP-C02 Exam Notes

## 1. Overview

**AWS Config** is a service that enables you to assess, audit, and evaluate the configurations of your AWS resources. It continuously monitors and records AWS resource configurations and allows you to automate the evaluation of recorded configurations against desired configurations.

### Key Characteristics
- **Configuration recording** - Tracks resource configuration changes over time
- **Compliance monitoring** - Evaluates resources against configuration rules
- **Change tracking** - Historical view of configuration changes
- **Automated remediation** - Automatic correction of non-compliant resources
- **Multi-account aggregation** - Centralized compliance across accounts
- **Integration** - Works with CloudTrail, CloudWatch, SNS, Lambda
- **Inventory management** - Complete inventory of AWS resources

### What Problem Does It Solve?
- Provides visibility into resource configurations and changes
- Enables compliance monitoring and governance
- Facilitates security analysis and troubleshooting
- Supports change management and audit requirements
- Automates compliance remediation
- Enables configuration drift detection

---

## 2. Core Components

### Configuration Recorder
```json
{
  "name": "default",
  "roleARN": "arn:aws:iam::123456789012:role/config-role",
  "recordingGroup": {
    "allSupported": true,
    "includeGlobalResourceTypes": true,
    "resourceTypes": []
  }
}
```

### Delivery Channel
```json
{
  "name": "default",
  "s3BucketName": "config-bucket-123456789012",
  "s3KeyPrefix": "config",
  "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:config-topic",
  "configSnapshotDeliveryProperties": {
    "deliveryFrequency": "TwentyFour_Hours"
  }
}
```

### Configuration Items
```json
{
  "version": "1.3",
  "accountId": "123456789012",
  "configurationItemCaptureTime": "2023-01-15T12:00:00.000Z",
  "configurationItemStatus": "ResourceDiscovered",
  "configurationStateId": "1642248000000",
  "resourceType": "AWS::EC2::Instance",
  "resourceId": "i-1234567890abcdef0",
  "resourceName": "MyInstance",
  "ARN": "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0",
  "awsRegion": "us-east-1",
  "availabilityZone": "us-east-1a",
  "resourceCreationTime": "2023-01-15T10:00:00.000Z",
  "tags": {
    "Environment": "Production",
    "Owner": "TeamA"
  },
  "configuration": {
    "instanceId": "i-1234567890abcdef0",
    "imageId": "ami-0abcdef1234567890",
    "state": {
      "code": 16,
      "name": "running"
    },
    "privateDnsName": "ip-10-0-1-100.ec2.internal",
    "publicDnsName": "ec2-54-123-45-67.compute-1.amazonaws.com",
    "instanceType": "t3.medium",
    "keyName": "my-key-pair",
    "securityGroups": [
      {
        "groupName": "default",
        "groupId": "sg-12345678"
      }
    ],
    "subnetId": "subnet-12345678",
    "vpcId": "vpc-12345678"
  }
}
```

---

## 3. Config Rules

### AWS Managed Rules
```yaml
# CloudFormation template for Config rules
Resources:
  # Check if EC2 instances are in VPC
  EC2InstancesInVPCRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ec2-instances-in-vpc
      Description: Checks whether EC2 instances are in a VPC
      Source:
        Owner: AWS
        SourceIdentifier: INSTANCES_IN_VPC
      DependsOn: ConfigurationRecorder

  # Check if S3 buckets are publicly readable
  S3BucketPublicReadProhibitedRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Description: Checks that S3 buckets do not allow public read access
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED
      DependsOn: ConfigurationRecorder

  # Check if RDS instances have backup enabled
  RDSBackupEnabledRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: rds-backup-enabled
      Description: Checks whether RDS DB instances have backups enabled
      Source:
        Owner: AWS
        SourceIdentifier: DB_INSTANCE_BACKUP_ENABLED
      InputParameters: |
        {
          "backupRetentionPeriod": "7"
        }
      DependsOn: ConfigurationRecorder

  # Check if security groups allow unrestricted SSH
  SecurityGroupSSHRestrictedRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: incoming-ssh-disabled
      Description: Checks whether security groups disallow unrestricted incoming SSH traffic
      Source:
        Owner: AWS
        SourceIdentifier: INCOMING_SSH_DISABLED
      DependsOn: ConfigurationRecorder
```

### Custom Config Rules
```python
# Lambda function for custom Config rule
import boto3
import json

def lambda_handler(event, context):
    """
    Custom Config rule to check if EC2 instances have required tags
    """
    
    # Get the configuration item
    config_item = event['configurationItem']
    
    # Check if this is an EC2 instance
    if config_item['resourceType'] != 'AWS::EC2::Instance':
        return {
            'compliance_type': 'NOT_APPLICABLE',
            'annotation': 'Rule only applies to EC2 instances'
        }
    
    # Required tags
    required_tags = ['Environment', 'Owner', 'Project']
    
    # Get instance tags
    instance_tags = config_item.get('tags', {})
    
    # Check if all required tags are present
    missing_tags = []
    for tag in required_tags:
        if tag not in instance_tags:
            missing_tags.append(tag)
    
    if missing_tags:
        return {
            'compliance_type': 'NON_COMPLIANT',
            'annotation': f'Missing required tags: {", ".join(missing_tags)}'
        }
    else:
        return {
            'compliance_type': 'COMPLIANT',
            'annotation': 'All required tags are present'
        }

# CloudFormation template for custom rule
Resources:
  CustomConfigRuleFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: custom-config-rule-required-tags
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt ConfigRuleLambdaRole.Arn
      Code:
        ZipFile: |
          # Lambda function code here

  CustomConfigRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ec2-required-tags
      Description: Checks if EC2 instances have required tags
      Source:
        Owner: AWS_LAMBDA
        SourceIdentifier: !GetAtt CustomConfigRuleFunction.Arn
        SourceDetail:
          - EventSource: aws.config
            MessageType: ConfigurationItemChangeNotification
          - EventSource: aws.config
            MessageType: OversizedConfigurationItemChangeNotification
      DependsOn: ConfigurationRecorder

  ConfigRulePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref CustomConfigRuleFunction
      Action: lambda:InvokeFunction
      Principal: config.amazonaws.com
```

---

## 4. Remediation Actions

### Automatic Remediation
```yaml
Resources:
  # Remediation configuration for S3 bucket encryption
  S3BucketEncryptionRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref S3BucketServerSideEncryptionEnabledRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-EnableS3BucketDefaultEncryption
      TargetVersion: "1"
      Parameters:
        AutomationAssumeRole:
          StaticValue: !GetAtt RemediationRole.Arn
        BucketName:
          ResourceValue: RESOURCE_ID
        SSEAlgorithm:
          StaticValue: AES256
      Automatic: true
      MaximumAutomaticAttempts: 3

  # Remediation for security group with unrestricted SSH
  SecurityGroupSSHRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref SecurityGroupSSHRestrictedRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-RemoveUnrestrictedSourceInSecurityGroup
      TargetVersion: "1"
      Parameters:
        AutomationAssumeRole:
          StaticValue: !GetAtt RemediationRole.Arn
        GroupId:
          ResourceValue: RESOURCE_ID
        IpProtocol:
          StaticValue: tcp
        FromPort:
          StaticValue: "22"
        ToPort:
          StaticValue: "22"
      Automatic: true
      MaximumAutomaticAttempts: 2

  # Custom remediation using Lambda
  CustomRemediationFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: custom-config-remediation
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt RemediationLambdaRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          
          def lambda_handler(event, context):
              """Custom remediation function"""
              
              # Get resource information from Config
              resource_id = event['configurationItem']['resourceId']
              resource_type = event['configurationItem']['resourceType']
              
              if resource_type == 'AWS::EC2::Instance':
                  ec2 = boto3.client('ec2')
                  
                  # Add required tags to EC2 instance
                  ec2.create_tags(
                      Resources=[resource_id],
                      Tags=[
                          {'Key': 'ComplianceStatus', 'Value': 'Remediated'},
                          {'Key': 'RemediationDate', 'Value': str(datetime.now())}
                      ]
                  )
              
              return {
                  'statusCode': 200,
                  'body': json.dumps('Remediation completed')
              }
```

### Manual Remediation Workflow
```python
# Lambda function for manual remediation workflow
import boto3
import json

def lambda_handler(event, context):
    """
    Creates remediation tickets for non-compliant resources
    """
    
    # Parse Config rule evaluation result
    config_item = event['configurationItem']
    compliance_type = event['newEvaluationResult']['complianceType']
    
    if compliance_type == 'NON_COMPLIANT':
        # Create ServiceNow ticket or send notification
        sns = boto3.client('sns')
        
        message = {
            'resource_id': config_item['resourceId'],
            'resource_type': config_item['resourceType'],
            'compliance_issue': event['newEvaluationResult']['annotation'],
            'account_id': config_item['accountId'],
            'region': config_item['awsRegion']
        }
        
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:compliance-alerts',
            Message=json.dumps(message),
            Subject=f'Compliance Issue: {config_item["resourceType"]} {config_item["resourceId"]}'
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Notification sent')
    }
```

---

## 5. Multi-Account Configuration

### Organization Config Setup
```yaml
# Master account setup
Resources:
  ConfigurationAggregator:
    Type: AWS::Config::ConfigurationAggregator
    Properties:
      ConfigurationAggregatorName: OrganizationConfigAggregator
      OrganizationAggregationSource:
        RoleArn: !GetAtt AggregatorRole.Arn
        AllAwsRegions: true

  # Organizational Config rules
  OrganizationalConfigRule:
    Type: AWS::Config::OrganizationConfigRule
    Properties:
      OrganizationConfigRuleName: s3-bucket-ssl-requests-only
      OrganizationManagedRuleMetadata:
        RuleIdentifier: S3_BUCKET_SSL_REQUESTS_ONLY
        Description: Checks whether S3 buckets have policies that require SSL requests
      ExcludedAccounts:
        - "111111111111"  # Exclude specific accounts if needed

  # Organizational remediation
  OrganizationalRemediationConfiguration:
    Type: AWS::Config::OrganizationRemediationConfiguration
    Properties:
      OrganizationConfigRuleName: !Ref OrganizationalConfigRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-EnableS3BucketSSLRequestsOnly
      TargetVersion: "1"
      Parameters:
        AutomationAssumeRole:
          StaticValue: !Sub "arn:aws:iam::{account}:role/ConfigRemediationRole"
        BucketName:
          ResourceValue: RESOURCE_ID
      Automatic: false
      MaximumAutomaticAttempts: 1
```

### Cross-Account Access
```yaml
# Member account role for Config aggregation
Resources:
  ConfigAggregationRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ConfigAggregationRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub "arn:aws:iam::${MasterAccountId}:root"
            Action: sts:AssumeRole
            Condition:
              StringEquals:
                'sts:ExternalId': !Ref ExternalId
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/ConfigRole
      Policies:
        - PolicyName: ConfigAggregationPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - config:GetComplianceDetailsByConfigRule
                  - config:GetComplianceDetailsByResource
                  - config:ListDiscoveredResources
                  - config:GetResourceConfigHistory
                Resource: '*'
```

---

## 6. Config with CI/CD Integration

### Compliance Gates in Pipeline
```python
# Lambda function for pipeline compliance check
import boto3
import json

def lambda_handler(event, context):
    """
    Check Config compliance before deployment
    """
    
    config_client = boto3.client('config')
    codepipeline = boto3.client('codepipeline')
    
    # Get job details from CodePipeline
    job_id = event['CodePipeline.job']['id']
    
    try:
        # Check compliance for critical rules
        critical_rules = [
            's3-bucket-public-read-prohibited',
            'incoming-ssh-disabled',
            'rds-backup-enabled'
        ]
        
        non_compliant_resources = []
        
        for rule_name in critical_rules:
            response = config_client.get_compliance_details_by_config_rule(
                ConfigRuleName=rule_name,
                ComplianceTypes=['NON_COMPLIANT']
            )
            
            if response['EvaluationResults']:
                non_compliant_resources.extend(response['EvaluationResults'])
        
        if non_compliant_resources:
            # Fail the pipeline
            codepipeline.put_job_failure_result(
                jobId=job_id,
                failureDetails={
                    'message': f'Found {len(non_compliant_resources)} non-compliant resources',
                    'type': 'JobFailed'
                }
            )
        else:
            # Continue the pipeline
            codepipeline.put_job_success_result(jobId=job_id)
    
    except Exception as e:
        codepipeline.put_job_failure_result(
            jobId=job_id,
            failureDetails={
                'message': str(e),
                'type': 'JobFailed'
            }
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Compliance check completed')
    }
```

### Infrastructure Compliance Validation
```yaml
# CodeBuild project for Config compliance validation
Resources:
  ComplianceValidationProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: config-compliance-validation
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0
      Source:
        Type: CODEPIPELINE
        BuildSpec: |
          version: 0.2
          phases:
            install:
              runtime-versions:
                python: 3.9
              commands:
                - pip install boto3
            pre_build:
              commands:
                - echo "Starting compliance validation..."
            build:
              commands:
                - python compliance_check.py
            post_build:
              commands:
                - echo "Compliance validation completed"
```

---

## 7. Config API and CLI

### Common CLI Commands
```bash
# Start configuration recorder
aws configservice start-configuration-recorder --configuration-recorder-name default

# Stop configuration recorder
aws configservice stop-configuration-recorder --configuration-recorder-name default

# Get configuration recorder status
aws configservice describe-configuration-recorder-status

# Get compliance by config rule
aws configservice get-compliance-details-by-config-rule --config-rule-name s3-bucket-public-read-prohibited

# Get compliance by resource
aws configservice get-compliance-details-by-resource --resource-type AWS::S3::Bucket --resource-id my-bucket

# List discovered resources
aws configservice list-discovered-resources --resource-type AWS::EC2::Instance

# Get resource config history
aws configservice get-resource-config-history --resource-type AWS::EC2::Instance --resource-id i-1234567890abcdef0

# Deliver configuration snapshot
aws configservice deliver-config-snapshot --delivery-channel-name default

# Put evaluations (for custom rules)
aws configservice put-evaluations --evaluations file://evaluations.json --result-token <token>
```

### Programmatic Access
```python
import boto3
from datetime import datetime, timedelta

def get_compliance_summary():
    """Get compliance summary across all rules"""
    
    config_client = boto3.client('config')
    
    # Get compliance summary
    response = config_client.get_compliance_summary_by_config_rule()
    
    summary = {
        'compliant': response['ComplianceSummary']['CompliantResourceCount']['CappedCount'],
        'non_compliant': response['ComplianceSummary']['NonCompliantResourceCount']['CappedCount'],
        'not_applicable': response['ComplianceSummary']['NotApplicableResourceCount']['CappedCount']
    }
    
    return summary

def get_configuration_changes(resource_type, hours=24):
    """Get configuration changes for a resource type in the last N hours"""
    
    config_client = boto3.client('config')
    
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=hours)
    
    # List resources
    resources = config_client.list_discovered_resources(
        resourceType=resource_type
    )
    
    changes = []
    
    for resource in resources['resourceIdentifiers']:
        # Get configuration history
        history = config_client.get_resource_config_history(
            resourceType=resource_type,
            resourceId=resource['resourceId'],
            laterTime=end_time,
            earlierTime=start_time
        )
        
        if history['configurationItems']:
            changes.extend(history['configurationItems'])
    
    return changes

def remediate_non_compliant_resources(rule_name):
    """Trigger remediation for non-compliant resources"""
    
    config_client = boto3.client('config')
    
    # Get non-compliant resources
    response = config_client.get_compliance_details_by_config_rule(
        ConfigRuleName=rule_name,
        ComplianceTypes=['NON_COMPLIANT']
    )
    
    resource_keys = []
    for evaluation in response['EvaluationResults']:
        resource_keys.append({
            'resourceType': evaluation['EvaluationResultIdentifier']['EvaluationResultQualifier']['ResourceType'],
            'resourceId': evaluation['EvaluationResultIdentifier']['EvaluationResultQualifier']['ResourceId']
        })
    
    if resource_keys:
        # Start remediation
        config_client.start_remediation_execution(
            ConfigRuleName=rule_name,
            ResourceKeys=resource_keys
        )
    
    return len(resource_keys)
```

---

## 8. Common Exam Scenarios

### Scenario 1: Compliance Monitoring Setup
```yaml
# Complete Config setup for compliance monitoring
Resources:
  # S3 bucket for Config
  ConfigBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "config-bucket-${AWS::AccountId}-${AWS::Region}"
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

  # Config service role
  ConfigRole:
    Type: AWS::IAM::ServiceLinkedRole
    Properties:
      AWSServiceName: config.amazonaws.com

  # Configuration recorder
  ConfigurationRecorder:
    Type: AWS::Config::ConfigurationRecorder
    Properties:
      Name: default
      RoleARN: !Sub "arn:aws:iam::${AWS::AccountId}:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig"
      RecordingGroup:
        AllSupported: true
        IncludeGlobalResourceTypes: true

  # Delivery channel
  DeliveryChannel:
    Type: AWS::Config::DeliveryChannel
    Properties:
      Name: default
      S3BucketName: !Ref ConfigBucket
      ConfigSnapshotDeliveryProperties:
        DeliveryFrequency: TwentyFour_Hours

  # Essential compliance rules
  S3BucketPublicAccessProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-access-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_ACCESS_PROHIBITED
      DependsOn: ConfigurationRecorder

  EC2SecurityGroupAttachedToENI:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ec2-security-group-attached-to-eni
      Source:
        Owner: AWS
        SourceIdentifier: EC2_SECURITY_GROUP_ATTACHED_TO_ENI
      DependsOn: ConfigurationRecorder
```

### Scenario 2: Automated Remediation Pipeline
```python
# Complete automated remediation workflow
import boto3
import json
from datetime import datetime

class ConfigRemediationOrchestrator:
    def __init__(self):
        self.config_client = boto3.client('config')
        self.ssm_client = boto3.client('ssm')
        self.sns_client = boto3.client('sns')
    
    def lambda_handler(self, event, context):
        """Main handler for Config rule evaluation changes"""
        
        # Parse the Config evaluation result
        config_item = event['configurationItem']
        evaluation_result = event.get('newEvaluationResult', {})
        
        if evaluation_result.get('complianceType') == 'NON_COMPLIANT':
            return self.handle_non_compliance(config_item, evaluation_result)
        
        return {'statusCode': 200, 'body': 'No action required'}
    
    def handle_non_compliance(self, config_item, evaluation_result):
        """Handle non-compliant resources"""
        
        resource_type = config_item['resourceType']
        resource_id = config_item['resourceId']
        rule_name = evaluation_result.get('configRuleName', '')
        
        # Define remediation actions
        remediation_actions = {
            'AWS::S3::Bucket': {
                's3-bucket-public-read-prohibited': self.remediate_s3_public_access,
                's3-bucket-ssl-requests-only': self.remediate_s3_ssl_policy
            },
            'AWS::EC2::SecurityGroup': {
                'incoming-ssh-disabled': self.remediate_security_group_ssh
            },
            'AWS::EC2::Instance': {
                'ec2-instance-managed-by-systems-manager': self.remediate_ssm_agent
            }
        }
        
        # Execute remediation if available
        if resource_type in remediation_actions and rule_name in remediation_actions[resource_type]:
            try:
                result = remediation_actions[resource_type][rule_name](resource_id, config_item)
                self.send_notification(resource_id, rule_name, 'SUCCESS', result)
                return {'statusCode': 200, 'body': f'Remediation successful: {result}'}
            except Exception as e:
                self.send_notification(resource_id, rule_name, 'FAILED', str(e))
                return {'statusCode': 500, 'body': f'Remediation failed: {str(e)}'}
        else:
            # Manual remediation required
            self.create_manual_remediation_ticket(resource_id, resource_type, rule_name)
            return {'statusCode': 200, 'body': 'Manual remediation ticket created'}
    
    def remediate_s3_public_access(self, bucket_name, config_item):
        """Remove public access from S3 bucket"""
        s3_client = boto3.client('s3')
        
        # Block public access
        s3_client.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        
        return f'Public access blocked for bucket {bucket_name}'
    
    def remediate_security_group_ssh(self, sg_id, config_item):
        """Remove unrestricted SSH access from security group"""
        ec2_client = boto3.client('ec2')
        
        # Revoke unrestricted SSH access
        ec2_client.revoke_security_group_ingress(
            GroupId=sg_id,
            IpPermissions=[
                {
                    'IpProtocol': 'tcp',
                    'FromPort': 22,
                    'ToPort': 22,
                    'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                }
            ]
        )
        
        return f'Removed unrestricted SSH access from security group {sg_id}'
    
    def send_notification(self, resource_id, rule_name, status, details):
        """Send remediation notification"""
        message = {
            'timestamp': datetime.utcnow().isoformat(),
            'resource_id': resource_id,
            'rule_name': rule_name,
            'remediation_status': status,
            'details': details
        }
        
        self.sns_client.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:config-remediation',
            Message=json.dumps(message),
            Subject=f'Config Remediation {status}: {resource_id}'
        )
```

---

## 9. Exam Tips

- **Understand Config components** - Recorder, delivery channel, rules, remediation
- **Know rule types** - AWS managed vs custom Lambda-based rules
- **Master remediation** - Automatic vs manual remediation workflows
- **Practice multi-account** - Organization Config and aggregation
- **Learn integration patterns** - Config with CloudTrail, CloudWatch, SNS
- **Understand compliance** - Evaluation results and compliance types
- **Know API operations** - Key CLI commands and programmatic access
- **Practice troubleshooting** - Common Config setup and rule issues
- **Understand costs** - Configuration items, rule evaluations, remediation costs
- **Master CI/CD integration** - Compliance gates and infrastructure validation