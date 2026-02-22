# AWS Config - DOP-C02 Exam Notes

## 1. Overview

**AWS Config** is a service that enables you to assess, audit, and evaluate the configurations of your AWS resources. It continuously monitors and records AWS resource configurations and allows you to automate the evaluation of recorded configurations against desired configurations.

### Key Characteristics
- **Configuration monitoring** - Continuous tracking of resource configurations
- **Compliance evaluation** - Automated compliance checking with rules
- **Change tracking** - Historical configuration changes and relationships
- **Remediation** - Automated remediation of non-compliant resources
- **Multi-account support** - Centralized compliance across AWS accounts
- **Integration** - Works with CloudTrail, CloudWatch, SNS, Lambda
- **Governance** - Supports organizational compliance and governance

### What Problem Does It Solve?
- Provides visibility into resource configurations and changes
- Enables automated compliance checking and governance
- Facilitates security analysis and audit requirements
- Supports change management and impact analysis
- Enables automated remediation of configuration drift
- Provides historical configuration data for troubleshooting

---

## 2. Core Concepts

### Configuration Item (CI)
- Point-in-time snapshot of resource configuration
- Contains resource metadata, attributes, and relationships
- Automatically created when resource changes occur
- Stored in S3 bucket for historical analysis

### Configuration Recorder
- Records configuration changes for AWS resources
- Must be enabled to start recording configurations
- Can be configured for specific resource types
- One recorder per region per account

### Delivery Channel
- Delivers configuration snapshots and history files to S3
- Can send notifications via SNS
- Configurable delivery frequency
- Required for Config to function

### Config Rules
- Desired configuration settings for resources
- Evaluate compliance automatically
- Can be AWS managed or custom (Lambda-based)
- Trigger evaluation on configuration changes or periodically

### Remediation Configuration
- Automated actions to fix non-compliant resources
- Uses Systems Manager Automation documents
- Can be automatic or manual remediation
- Supports rollback and error handling

---

## 3. Configuration Recorder Setup

### Basic Configuration Recorder
```yaml
Resources:
  ConfigurationRecorder:
    Type: AWS::Config::ConfigurationRecorder
    Properties:
      Name: default
      RoleARN: !GetAtt ConfigRole.Arn
      RecordingGroup:
        AllSupported: true
        IncludeGlobalResourceTypes: true
        ResourceTypes: []

  DeliveryChannel:
    Type: AWS::Config::DeliveryChannel
    Properties:
      Name: default
      S3BucketName: !Ref ConfigBucket
      S3KeyPrefix: config/
      ConfigSnapshotDeliveryProperties:
        DeliveryFrequency: Daily
      
  ConfigBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'config-bucket-${AWS::AccountId}-${AWS::Region}'
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  ConfigRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: config.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/ConfigRole
      Policies:
        - PolicyName: ConfigBucketPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetBucketAcl
                  - s3:ListBucket
                Resource: !Sub 'arn:aws:s3:::config-bucket-${AWS::AccountId}-${AWS::Region}'
              - Effect: Allow
                Action: s3:PutObject
                Resource: !Sub 'arn:aws:s3:::config-bucket-${AWS::AccountId}-${AWS::Region}/*'
                Condition:
                  StringEquals:
                    's3:x-amz-acl': bucket-owner-full-control
```

### Selective Resource Recording
```yaml
ConfigurationRecorder:
  Type: AWS::Config::ConfigurationRecorder
  Properties:
    Name: selective-recorder
    RoleARN: !GetAtt ConfigRole.Arn
    RecordingGroup:
      AllSupported: false
      IncludeGlobalResourceTypes: false
      ResourceTypes:
        - AWS::EC2::Instance
        - AWS::EC2::SecurityGroup
        - AWS::S3::Bucket
        - AWS::IAM::Role
        - AWS::Lambda::Function
        - AWS::RDS::DBInstance
```

---

## 4. Config Rules

### AWS Managed Rules
```yaml
Resources:
  # Security group rule
  SecurityGroupSSHRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: incoming-ssh-disabled
      Description: Checks whether security groups disallow unrestricted incoming SSH traffic
      Source:
        Owner: AWS
        SourceIdentifier: INCOMING_SSH_DISABLED
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::SecurityGroup

  # S3 bucket encryption rule
  S3BucketEncryptionRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-server-side-encryption-enabled
      Description: Checks that S3 buckets have server-side encryption enabled
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED
      Scope:
        ComplianceResourceTypes:
          - AWS::S3::Bucket

  # Root access key rule
  RootAccessKeyRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: root-access-key-check
      Description: Checks whether root access key is available
      Source:
        Owner: AWS
        SourceIdentifier: ROOT_ACCESS_KEY_CHECK
      MaximumExecutionFrequency: Daily
```

### Custom Config Rule with Lambda
```python
import boto3
import json

def lambda_handler(event, context):
    """
    Custom Config rule to check EC2 instance tags
    """
    
    config = boto3.client('config')
    
    # Get the configuration item
    configuration_item = event['configurationItem']
    
    # Check if resource is EC2 instance
    if configuration_item['resourceType'] != 'AWS::EC2::Instance':
        return {
            'compliance_type': 'NOT_APPLICABLE',
            'annotation': 'Rule only applies to EC2 instances'
        }
    
    # Check required tags
    required_tags = ['Environment', 'Owner', 'Project']
    resource_tags = configuration_item.get('tags', {})
    
    missing_tags = []
    for tag in required_tags:
        if tag not in resource_tags:
            missing_tags.append(tag)
    
    if missing_tags:
        compliance_type = 'NON_COMPLIANT'
        annotation = f'Missing required tags: {", ".join(missing_tags)}'
    else:
        compliance_type = 'COMPLIANT'
        annotation = 'All required tags are present'
    
    # Put evaluation result
    evaluation = {
        'ComplianceResourceType': configuration_item['resourceType'],
        'ComplianceResourceId': configuration_item['resourceId'],
        'ComplianceType': compliance_type,
        'Annotation': annotation,
        'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
    }
    
    config.put_evaluations(
        Evaluations=[evaluation],
        ResultToken=event['resultToken']
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Evaluation completed')
    }

# CloudFormation template for custom rule
CustomTagRule:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: ec2-required-tags
    Description: Checks if EC2 instances have required tags
    Source:
      Owner: AWS_LAMBDA
      SourceIdentifier: !GetAtt CustomTagRuleFunction.Arn
      SourceDetails:
        - EventSource: aws.config
          MessageType: ConfigurationItemChangeNotification
          MaximumExecutionFrequency: Daily
    Scope:
      ComplianceResourceTypes:
        - AWS::EC2::Instance
```

---

## 5. Remediation Configuration

### Automatic Remediation
```yaml
Resources:
  # Remediation for non-compliant S3 buckets
  S3EncryptionRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref S3BucketEncryptionRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-EnableS3BucketDefaultEncryption
      TargetVersion: '1'
      Parameters:
        AutomationAssumeRole:
          StaticValue: !GetAtt RemediationRole.Arn
        BucketName:
          ResourceValue: RESOURCE_ID
        SSEAlgorithm:
          StaticValue: AES256
      Automatic: true
      MaximumAutomaticAttempts: 3

  # Remediation for security group rules
  SecurityGroupRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref SecurityGroupSSHRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-RemoveUnrestrictedSourceInSecurityGroup
      TargetVersion: '1'
      Parameters:
        AutomationAssumeRole:
          StaticValue: !GetAtt RemediationRole.Arn
        GroupId:
          ResourceValue: RESOURCE_ID
        IpProtocol:
          StaticValue: tcp
        FromPort:
          StaticValue: '22'
        ToPort:
          StaticValue: '22'
      Automatic: false  # Manual approval required

  RemediationRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ssm.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: RemediationPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:PutEncryptionConfiguration
                  - ec2:AuthorizeSecurityGroupIngress
                  - ec2:RevokeSecurityGroupIngress
                  - ec2:DescribeSecurityGroups
                Resource: '*'
```

### Custom Remediation with Lambda
```python
def lambda_handler(event, context):
    """
    Custom remediation for non-compliant resources
    """
    
    # Parse SSM automation event
    resource_id = event['ResourceId']
    resource_type = event['ResourceType']
    
    if resource_type == 'AWS::EC2::Instance':
        remediate_ec2_instance(resource_id)
    elif resource_type == 'AWS::S3::Bucket':
        remediate_s3_bucket(resource_id)
    
    return {
        'statusCode': 200,
        'body': json.dumps('Remediation completed')
    }

def remediate_ec2_instance(instance_id):
    """Remediate non-compliant EC2 instance"""
    
    ec2 = boto3.client('ec2')
    
    # Add required tags
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'Environment', 'Value': 'Unknown'},
            {'Key': 'Owner', 'Value': 'AutoRemediation'},
            {'Key': 'Project', 'Value': 'Compliance'}
        ]
    )
    
    print(f"Added compliance tags to instance {instance_id}")

def remediate_s3_bucket(bucket_name):
    """Remediate non-compliant S3 bucket"""
    
    s3 = boto3.client('s3')
    
    # Enable default encryption
    s3.put_bucket_encryption(
        Bucket=bucket_name,
        ServerSideEncryptionConfiguration={
            'Rules': [
                {
                    'ApplyServerSideEncryptionByDefault': {
                        'SSEAlgorithm': 'AES256'
                    }
                }
            ]
        }
    )
    
    print(f"Enabled encryption for bucket {bucket_name}")
```

---

## 6. Multi-Account Configuration

### Config Aggregator
```yaml
Resources:
  ConfigAggregator:
    Type: AWS::Config::ConfigurationAggregator
    Properties:
      ConfigurationAggregatorName: OrganizationAggregator
      OrganizationAggregationSource:
        RoleArn: !GetAtt AggregatorRole.Arn
        AwsRegions:
          - us-east-1
          - us-west-2
          - eu-west-1
        AllAwsRegions: false

  AggregatorRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: config.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/ConfigRole
      Policies:
        - PolicyName: OrganizationAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - organizations:DescribeOrganization
                  - organizations:ListAccounts
                  - organizations:ListAWSServiceAccessForOrganization
                Resource: '*'
```

### Cross-Account Authorization
```yaml
# In member accounts
ConfigAuthorizationRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            AWS: arn:aws:iam::MASTER-ACCOUNT-ID:root
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/ConfigRole

# CLI command to authorize aggregator
aws configservice put-aggregation-authorization \
  --authorized-account-id MASTER-ACCOUNT-ID \
  --authorized-aws-region us-east-1
```

---

## 7. CI/CD Integration

### Config Rules in CI/CD Pipeline
```yaml
- Name: ComplianceCheck
  Actions:
    - Name: ConfigComplianceGate
      ActionTypeId:
        Category: Invoke
        Owner: AWS
        Provider: Lambda
        Version: 1
      Configuration:
        FunctionName: ConfigComplianceChecker
        UserParameters: |
          {
            "required_rules": [
              "s3-bucket-server-side-encryption-enabled",
              "incoming-ssh-disabled",
              "ec2-required-tags"
            ],
            "compliance_threshold": 95
          }
      InputArtifacts:
        - Name: DeploymentOutput
```

### Compliance Gate Lambda
```python
def lambda_handler(event, context):
    """
    Check Config compliance before allowing deployment
    """
    
    codepipeline = boto3.client('codepipeline')
    config = boto3.client('config')
    
    job_id = event['CodePipeline.job']['id']
    user_parameters = json.loads(
        event['CodePipeline.job']['data']['actionConfiguration']['configuration']['UserParameters']
    )
    
    try:
        required_rules = user_parameters['required_rules']
        threshold = user_parameters.get('compliance_threshold', 100)
        
        # Check compliance for each rule
        compliance_results = []
        
        for rule_name in required_rules:
            response = config.get_compliance_details_by_config_rule(
                ConfigRuleName=rule_name
            )
            
            total_resources = len(response['EvaluationResults'])
            compliant_resources = len([
                r for r in response['EvaluationResults'] 
                if r['ComplianceType'] == 'COMPLIANT'
            ])
            
            compliance_percentage = (compliant_resources / total_resources * 100) if total_resources > 0 else 100
            
            compliance_results.append({
                'rule': rule_name,
                'compliance_percentage': compliance_percentage,
                'compliant_resources': compliant_resources,
                'total_resources': total_resources
            })
        
        # Calculate overall compliance
        overall_compliance = sum(r['compliance_percentage'] for r in compliance_results) / len(compliance_results)
        
        if overall_compliance >= threshold:
            codepipeline.put_job_success_result(
                jobId=job_id,
                outputVariables={
                    'CompliancePercentage': str(overall_compliance),
                    'ComplianceStatus': 'PASSED'
                }
            )
        else:
            failure_details = f"Compliance {overall_compliance:.1f}% below threshold {threshold}%"
            codepipeline.put_job_failure_result(
                jobId=job_id,
                failureDetails={'message': failure_details}
            )
            
    except Exception as e:
        codepipeline.put_job_failure_result(
            jobId=job_id,
            failureDetails={'message': str(e)}
        )
```

---

## 8. Common Exam Scenarios

### Scenario 1: Implement organization-wide compliance monitoring
**Solution:**
- Set up Config in all accounts and regions
- Create Config aggregator in master account
- Deploy standardized Config rules across organization
- Implement automated remediation for common violations

### Scenario 2: Automated security compliance checking
**Solution:**
- Deploy AWS managed security rules
- Create custom rules for organization-specific requirements
- Set up automatic remediation for critical violations
- Integrate with Security Hub for centralized findings

### Scenario 3: Change impact analysis and rollback
**Solution:**
- Use Config timeline to track configuration changes
- Correlate changes with CloudTrail events
- Implement automated rollback for unauthorized changes
- Set up alerts for critical configuration changes

### Scenario 4: Compliance gate in CI/CD pipeline
**Solution:**
- Create Lambda function to check Config compliance
- Integrate compliance check as CodePipeline action
- Set compliance thresholds for deployment approval
- Generate compliance reports as pipeline artifacts

### Scenario 5: Cost optimization compliance
**Solution:**
- Create rules to check for untagged resources
- Monitor for oversized instances or unused resources
- Implement automated remediation for cost optimization
- Generate cost compliance reports

### Scenario 6: Data governance and encryption compliance
**Solution:**
- Create rules to ensure S3 bucket encryption
- Monitor RDS encryption compliance
- Check for public access configurations
- Implement automated encryption remediation

### Scenario 7: Network security compliance
**Solution:**
- Monitor security group configurations
- Check for unrestricted access rules
- Validate VPC and subnet configurations
- Implement automated security group remediation

### Scenario 8: Hybrid environment compliance
**Solution:**
- Extend Config to on-premises resources
- Create custom rules for hybrid configurations
- Implement unified compliance reporting
- Set up cross-environment remediation workflows

---

## 9. CLI Commands Reference

### Configuration Recorder Operations
```bash
# Start configuration recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Stop configuration recorder
aws configservice stop-configuration-recorder \
  --configuration-recorder-name default

# Get recorder status
aws configservice describe-configuration-recorder-status

# Get configuration history
aws configservice get-resource-config-history \
  --resource-type AWS::EC2::Instance \
  --resource-id i-1234567890abcdef0
```

### Config Rules Operations
```bash
# Create config rule
aws configservice put-config-rule \
  --config-rule file://config-rule.json

# Get compliance details
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-server-side-encryption-enabled

# Get compliance summary
aws configservice get-compliance-summary-by-config-rule

# Delete config rule
aws configservice delete-config-rule \
  --config-rule-name my-custom-rule
```

### Remediation Operations
```bash
# Put remediation configuration
aws configservice put-remediation-configurations \
  --remediation-configurations file://remediation-config.json

# Start remediation execution
aws configservice start-remediation-execution \
  --config-rule-name s3-bucket-server-side-encryption-enabled \
  --resource-keys ResourceType=AWS::S3::Bucket,ResourceId=my-bucket

# Get remediation execution status
aws configservice describe-remediation-execution-status \
  --config-rule-name s3-bucket-server-side-encryption-enabled
```

---

## 10. Best Practices for DOP-C02 Exam

### Configuration Management
- Enable Config in all regions and accounts
- Use selective recording for cost optimization
- Implement proper S3 bucket lifecycle policies
- Set up cross-region replication for compliance data

### Rule Management
- Start with AWS managed rules before creating custom rules
- Use rule parameters for flexibility
- Implement proper error handling in custom rules
- Regular review and update of rule configurations

### Remediation
- Test remediation actions in non-production first
- Implement proper IAM permissions for remediation
- Use manual approval for critical remediations
- Monitor remediation execution and success rates

### Multi-Account Strategy
- Use Config aggregator for centralized compliance view
- Implement consistent rule deployment across accounts
- Set up proper cross-account IAM permissions
- Use AWS Organizations for automated setup

---

## 11. Exam Tips

### What to Remember
- **Config records configuration changes** continuously for supported resources
- **Configuration recorder must be enabled** before Config can function
- **Delivery channel is required** to store configuration data in S3
- **Rules evaluate compliance** automatically on changes or periodically
- **Remediation can be automatic or manual** using SSM documents
- **Aggregators provide multi-account compliance views**
- **Custom rules use Lambda functions** for evaluation logic

### Common Traps
- Forgetting to enable configuration recorder (Config won't work)
- Not configuring proper IAM permissions for Config service role
- Overlooking S3 bucket policy requirements for delivery channel
- Missing remediation IAM permissions for automated fixes
- Not considering cost implications of recording all resource types
- Forgetting to set up delivery channel (required for Config operation)

### Scenario-Based Questions
- Focus on compliance automation and governance use cases
- Understand remediation strategies and automation
- Know multi-account configuration and aggregation patterns
- Understand integration with CI/CD for compliance gates
- Know cost optimization strategies for Config
- Understand security and compliance monitoring patterns

---

## 12. Quick Reference Cheat Sheet

### Core Components
```
Configuration Recorder: Records resource configurations
Delivery Channel: Stores data in S3, sends SNS notifications
Config Rules: Evaluate compliance automatically
Remediation: Automated fixing of non-compliant resources
Aggregator: Multi-account compliance view
```

### Rule Types
```
AWS Managed: Pre-built rules for common compliance checks
Custom: Lambda-based rules for specific requirements
Periodic: Evaluated on schedule
Change-triggered: Evaluated when resources change
```

### Compliance States
```
COMPLIANT: Resource meets rule requirements
NON_COMPLIANT: Resource violates rule requirements
NOT_APPLICABLE: Rule doesn't apply to resource
INSUFFICIENT_DATA: Not enough data to evaluate
```

### Common IAM Actions
```
config:PutConfigurationRecorder
config:StartConfigurationRecorder
config:PutConfigRule
config:GetComplianceDetailsByConfigRule
config:PutRemediationConfigurations
```

---

## 13. Summary

AWS Config is essential for compliance monitoring and governance in AWS environments and is heavily tested in the DOP-C02 exam. Key areas to master:

1. **Configuration recording** (recorder setup, resource types, delivery channels)
2. **Compliance rules** (AWS managed rules, custom rules, evaluation triggers)
3. **Automated remediation** (SSM documents, Lambda functions, approval workflows)
4. **Multi-account management** (aggregators, cross-account permissions)
5. **CI/CD integration** (compliance gates, automated checking)
6. **Cost optimization** (selective recording, lifecycle policies)
7. **Security compliance** (encryption, access controls, governance)
8. **Operational excellence** (monitoring, alerting, reporting)

Understanding these concepts with hands-on practice will ensure success on Config-related questions in the DOP-C02 exam.