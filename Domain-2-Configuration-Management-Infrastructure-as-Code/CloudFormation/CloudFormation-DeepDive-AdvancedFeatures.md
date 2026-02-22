# AWS CloudFormation Advanced Features - DOP-C02 Exam Notes

## 1. Overview

This document covers advanced CloudFormation features that are critical for the DOP-C02 exam but not covered in the general CloudFormation notes. These include StackSets, Custom Resources, Dynamic References, CloudFormation Registry, and advanced deployment patterns.

### Key Advanced Features
- **StackSets** - Multi-account and multi-region deployments
- **Custom Resources** - Extend CloudFormation with Lambda
- **Dynamic References** - Retrieve values from Parameter Store and Secrets Manager
- **CloudFormation Registry** - Third-party resource providers
- **Rollback Configuration** - Advanced rollback controls
- **Stack Notifications** - SNS integration for stack events
- **Termination Protection** - Prevent accidental stack deletion

---

## 2. StackSets

### Overview
StackSets enable you to create, update, or delete stacks across multiple accounts and regions with a single operation.

### Core Concepts
- **StackSet** - Container for stack instances
- **Stack Instance** - Reference to a stack in a target account and region
- **Administration Account** - Account that creates and manages StackSets
- **Target Account** - Account where stack instances are deployed
- **Organizational Unit (OU)** - AWS Organizations unit for deployment targeting

### StackSet Configuration
```yaml
# StackSet template example
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Multi-account security baseline'

Parameters:
  OrganizationId:
    Type: String
    Description: AWS Organization ID

Resources:
  SecurityRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: OrganizationSecurityRole
      AssumeRolePolicyDocument:
        Version: '2012-10-09'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: sts:AssumeRole
            Condition:
              StringEquals:
                'aws:PrincipalOrgID': !Ref OrganizationId

  CloudTrail:
    Type: AWS::CloudTrail::Trail
    Properties:
      TrailName: OrganizationCloudTrail
      S3BucketName: !Sub 'org-cloudtrail-${AWS::AccountId}-${AWS::Region}'
      IncludeGlobalServiceEvents: true
      IsMultiRegionTrail: true
      EnableLogFileValidation: true

  ConfigurationRecorder:
    Type: AWS::Config::ConfigurationRecorder
    Properties:
      Name: OrganizationConfigRecorder
      RoleARN: !GetAtt ConfigRole.Arn
      RecordingGroup:
        AllSupported: true
        IncludeGlobalResourceTypes: true

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
```

### StackSet Operations
```bash
# Create StackSet
aws cloudformation create-stack-set \
  --stack-set-name SecurityBaseline \
  --template-body file://security-baseline.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --administration-role-arn arn:aws:iam::123456789012:role/AWSCloudFormationStackSetAdministrationRole \
  --execution-role-name AWSCloudFormationStackSetExecutionRole

# Deploy to specific accounts and regions
aws cloudformation create-stack-instances \
  --stack-set-name SecurityBaseline \
  --accounts 111111111111 222222222222 333333333333 \
  --regions us-east-1 us-west-2 eu-west-1 \
  --parameter-overrides ParameterKey=OrganizationId,ParameterValue=o-1234567890

# Deploy to organizational units
aws cloudformation create-stack-instances \
  --stack-set-name SecurityBaseline \
  --deployment-targets OrganizationalUnitIds=ou-root-1234567890 \
  --regions us-east-1 us-west-2 \
  --operation-preferences MaxConcurrentPercentage=50,FailureTolerancePercentage=10

# Update StackSet
aws cloudformation update-stack-set \
  --stack-set-name SecurityBaseline \
  --template-body file://updated-security-baseline.yaml \
  --operation-preferences MaxConcurrentPercentage=25,FailureTolerancePercentage=5

# Delete stack instances
aws cloudformation delete-stack-instances \
  --stack-set-name SecurityBaseline \
  --accounts 111111111111 \
  --regions us-east-1 \
  --retain-stacks
```

### StackSet IAM Roles
```yaml
# Administration Role (in management account)
AWSCloudFormationStackSetAdministrationRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: AWSCloudFormationStackSetAdministrationRole
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: cloudformation.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: AssumeRole-AWSCloudFormationStackSetExecutionRole
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - sts:AssumeRole
              Resource:
                - "arn:aws:iam::*:role/AWSCloudFormationStackSetExecutionRole"

# Execution Role (in target accounts)
AWSCloudFormationStackSetExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: AWSCloudFormationStackSetExecutionRole
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            AWS: arn:aws:iam::123456789012:role/AWSCloudFormationStackSetAdministrationRole
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/PowerUserAccess
```

---

## 3. Custom Resources

### Overview
Custom resources enable you to write custom provisioning logic for resources that CloudFormation doesn't natively support.

### Custom Resource Types
- **Lambda-backed** - Most common, uses Lambda function
- **SNS-backed** - Uses SNS topic (legacy approach)
- **External** - Third-party resource providers

### Lambda-backed Custom Resource
```yaml
Resources:
  CustomResourceFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.9
      Handler: index.handler
      Role: !GetAtt CustomResourceRole.Arn
      Code:
        ZipFile: |
          import json
          import boto3
          import cfnresponse
          import uuid
          
          def handler(event, context):
              print(f"Event: {json.dumps(event)}")
              
              try:
                  request_type = event['RequestType']
                  properties = event['ResourceProperties']
                  
                  if request_type == 'Create':
                      result = create_resource(properties)
                  elif request_type == 'Update':
                      result = update_resource(properties, event['OldResourceProperties'])
                  elif request_type == 'Delete':
                      result = delete_resource(properties, event['PhysicalResourceId'])
                  
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, result, result.get('PhysicalResourceId'))
                  
              except Exception as e:
                  print(f"Error: {str(e)}")
                  cfnresponse.send(event, context, cfnresponse.FAILED, {}, event.get('PhysicalResourceId'))
          
          def create_resource(properties):
              # Custom creation logic
              resource_id = str(uuid.uuid4())
              
              # Example: Create custom configuration
              config_data = {
                  'ConfigValue': properties.get('ConfigValue', 'default'),
                  'Environment': properties.get('Environment', 'dev')
              }
              
              # Store configuration in Parameter Store
              ssm = boto3.client('ssm')
              ssm.put_parameter(
                  Name=f'/custom-resource/{resource_id}',
                  Value=json.dumps(config_data),
                  Type='String'
              )
              
              return {
                  'PhysicalResourceId': resource_id,
                  'Data': {
                      'ResourceId': resource_id,
                      'ConfigValue': config_data['ConfigValue']
                  }
              }
          
          def update_resource(properties, old_properties):
              # Custom update logic
              return {'PhysicalResourceId': event['PhysicalResourceId']}
          
          def delete_resource(properties, physical_resource_id):
              # Custom deletion logic
              ssm = boto3.client('ssm')
              try:
                  ssm.delete_parameter(Name=f'/custom-resource/{physical_resource_id}')
              except ssm.exceptions.ParameterNotFound:
                  pass  # Parameter already deleted
              
              return {'PhysicalResourceId': physical_resource_id}

  CustomResource:
    Type: AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !GetAtt CustomResourceFunction.Arn
      ConfigValue: !Ref ConfigurationValue
      Environment: !Ref Environment

  CustomResourceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: CustomResourcePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ssm:PutParameter
                  - ssm:DeleteParameter
                  - ssm:GetParameter
                Resource: '*'
```

### Advanced Custom Resource Example
```python
import boto3
import json
import cfnresponse
import time

def handler(event, context):
    """
    Custom resource for managing third-party API resources
    """
    
    try:
        request_type = event['RequestType']
        properties = event['ResourceProperties']
        
        # Initialize third-party client
        api_client = ThirdPartyAPIClient(
            api_key=properties['ApiKey'],
            base_url=properties['BaseUrl']
        )
        
        if request_type == 'Create':
            # Create resource via third-party API
            resource_config = {
                'name': properties['ResourceName'],
                'type': properties['ResourceType'],
                'configuration': json.loads(properties.get('Configuration', '{}'))
            }
            
            response = api_client.create_resource(resource_config)
            physical_id = response['id']
            
            # Wait for resource to be ready
            wait_for_resource_ready(api_client, physical_id)
            
            return_data = {
                'ResourceId': physical_id,
                'ResourceUrl': response['url'],
                'Status': response['status']
            }
            
        elif request_type == 'Update':
            physical_id = event['PhysicalResourceId']
            
            # Update resource configuration
            update_config = {
                'configuration': json.loads(properties.get('Configuration', '{}'))
            }
            
            api_client.update_resource(physical_id, update_config)
            
            return_data = {
                'ResourceId': physical_id,
                'Status': 'updated'
            }
            
        elif request_type == 'Delete':
            physical_id = event['PhysicalResourceId']
            
            # Delete resource via API
            api_client.delete_resource(physical_id)
            
            return_data = {
                'ResourceId': physical_id,
                'Status': 'deleted'
            }
        
        cfnresponse.send(event, context, cfnresponse.SUCCESS, return_data, physical_id)
        
    except Exception as e:
        print(f"Error: {str(e)}")
        cfnresponse.send(event, context, cfnresponse.FAILED, {}, 
                        event.get('PhysicalResourceId', 'failed-to-create'))

def wait_for_resource_ready(api_client, resource_id, max_wait=300):
    """Wait for third-party resource to be ready"""
    
    start_time = time.time()
    while time.time() - start_time < max_wait:
        status = api_client.get_resource_status(resource_id)
        if status == 'ready':
            return
        elif status == 'failed':
            raise Exception(f"Resource {resource_id} failed to initialize")
        
        time.sleep(10)
    
    raise Exception(f"Resource {resource_id} not ready within {max_wait} seconds")
```

---

## 4. Dynamic References

### Overview
Dynamic references provide a way to retrieve values stored in AWS Systems Manager Parameter Store and AWS Secrets Manager at stack deployment time.

### Parameter Store References
```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-12345678
      InstanceType: t3.micro
      KeyName: '{{resolve:ssm:/ec2/keypair/name:1}}'  # Version 1
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          DB_PASSWORD='{{resolve:ssm-secure:/myapp/database/password:1}}'
          echo "Database password retrieved: $DB_PASSWORD"

  DatabaseInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: myapp-db
      DBInstanceClass: db.t3.micro
      Engine: mysql
      MasterUsername: admin
      MasterUserPassword: '{{resolve:ssm-secure:/myapp/database/master-password}}'
      AllocatedStorage: 20
```

### Secrets Manager References
```yaml
Resources:
  MyApplication:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: myapp
      ContainerDefinitions:
        - Name: app-container
          Image: myapp:latest
          Environment:
            - Name: DB_HOST
              Value: !GetAtt DatabaseInstance.Endpoint.Address
            - Name: DB_PASSWORD
              Value: '{{resolve:secretsmanager:prod/myapp/db:SecretString:password}}'
            - Name: API_KEY
              Value: '{{resolve:secretsmanager:prod/myapp/api:SecretString:api_key}}'

  DatabaseSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: prod/myapp/db
      Description: Database credentials for MyApp
      GenerateSecretString:
        SecretStringTemplate: '{"username": "admin"}'
        GenerateStringKey: 'password'
        PasswordLength: 16
        ExcludeCharacters: '"@/\'
```

### Dynamic Reference Patterns
```yaml
# Parameter Store (plain text)
'{{resolve:ssm:parameter-name:version}}'
'{{resolve:ssm:parameter-name}}'  # Latest version

# Parameter Store (secure string)
'{{resolve:ssm-secure:parameter-name:version}}'
'{{resolve:ssm-secure:parameter-name}}'  # Latest version

# Secrets Manager
'{{resolve:secretsmanager:secret-id:SecretString:json-key}}'
'{{resolve:secretsmanager:secret-id:SecretString}}'  # Entire secret
'{{resolve:secretsmanager:secret-id:SecretBinary}}'  # Binary secret

# Examples with versions
'{{resolve:secretsmanager:prod/myapp/db:SecretString:password:AWSCURRENT}}'
'{{resolve:secretsmanager:prod/myapp/db:SecretString:password:AWSPENDING}}'
```

---

## 5. CloudFormation Registry

### Overview
The CloudFormation Registry enables you to use third-party resource providers and modules in your CloudFormation templates.

### Resource Providers
```bash
# List available resource types
aws cloudformation list-types --type RESOURCE

# Register a third-party resource provider
aws cloudformation register-type \
  --type RESOURCE \
  --type-name MyCompany::MyService::MyResource \
  --schema-handler-package s3://my-bucket/my-resource-provider.zip

# Activate a resource type
aws cloudformation activate-type \
  --type RESOURCE \
  --type-name MyCompany::MyService::MyResource \
  --publisher-id 123456789012

# Use third-party resource in template
Resources:
  MyCustomResource:
    Type: MyCompany::MyService::MyResource
    Properties:
      Name: MyResource
      Configuration:
        Setting1: Value1
        Setting2: Value2
```

### Module Usage
```yaml
# Using a CloudFormation module
Resources:
  VPCModule:
    Type: MyCompany::Network::VPC::MODULE
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      AvailabilityZones:
        - us-east-1a
        - us-east-1b
      PublicSubnetCidrs:
        - 10.0.1.0/24
        - 10.0.2.0/24
      PrivateSubnetCidrs:
        - 10.0.10.0/24
        - 10.0.20.0/24
```

---

## 6. Rollback Configuration

### Overview
Rollback configuration allows you to specify CloudWatch alarms that CloudFormation monitors during stack creation and update operations.

### Rollback Configuration Example
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Stack with rollback configuration'

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro

Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-12345678
      InstanceType: !Ref InstanceType
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd

  CPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${AWS::StackName}-HighCPU'
      AlarmDescription: 'High CPU utilization'
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref WebServerInstance

  StatusCheckAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${AWS::StackName}-StatusCheck'
      AlarmDescription: 'Instance status check failed'
      MetricName: StatusCheckFailed
      Namespace: AWS/EC2
      Statistic: Maximum
      Period: 60
      EvaluationPeriods: 2
      Threshold: 0
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref WebServerInstance
```

### CLI with Rollback Configuration
```bash
# Create stack with rollback configuration
aws cloudformation create-stack \
  --stack-name MyStack \
  --template-body file://template.yaml \
  --rollback-configuration RollbackTriggers='[
    {
      "Arn": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:MyStack-HighCPU",
      "Type": "AWS::CloudWatch::Alarm"
    },
    {
      "Arn": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:MyStack-StatusCheck",
      "Type": "AWS::CloudWatch::Alarm"
    }
  ]',MonitoringTimeInMinutes=5
```

---

## 7. Stack Notifications

### SNS Integration
```yaml
Resources:
  StackNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: CloudFormationNotifications
      DisplayName: CloudFormation Stack Notifications

  StackNotificationSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: email
      TopicArn: !Ref StackNotificationTopic
      Endpoint: admin@example.com

  # Use in nested stack
  NestedStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/nested-template.yaml
      NotificationARNs:
        - !Ref StackNotificationTopic
      Parameters:
        Environment: Production
```

### EventBridge Integration for Stack Events
```yaml
Resources:
  StackEventRule:
    Type: AWS::Events::Rule
    Properties:
      Description: 'CloudFormation stack state changes'
      EventPattern:
        source:
          - aws.cloudformation
        detail-type:
          - CloudFormation Stack Status Change
        detail:
          status-details:
            status:
              - CREATE_COMPLETE
              - UPDATE_COMPLETE
              - DELETE_COMPLETE
              - CREATE_FAILED
              - UPDATE_FAILED
              - DELETE_FAILED
      Targets:
        - Arn: !GetAtt StackEventProcessor.Arn
          Id: StackEventTarget

  StackEventProcessor:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.9
      Handler: index.handler
      Role: !GetAtt StackEventRole.Arn
      Code:
        ZipFile: |
          import json
          import boto3
          
          def handler(event, context):
              print(f"Stack event: {json.dumps(event, indent=2)}")
              
              detail = event['detail']
              stack_name = detail['stack-name']
              status = detail['status-details']['status']
              
              # Send notification based on status
              if status in ['CREATE_FAILED', 'UPDATE_FAILED', 'DELETE_FAILED']:
                  send_alert(stack_name, status, detail)
              elif status in ['CREATE_COMPLETE', 'UPDATE_COMPLETE']:
                  send_success_notification(stack_name, status)
              
              return {'statusCode': 200}
          
          def send_alert(stack_name, status, detail):
              # Send alert for failed operations
              sns = boto3.client('sns')
              message = f"CloudFormation stack {stack_name} failed with status: {status}"
              
              sns.publish(
                  TopicArn='arn:aws:sns:us-east-1:123456789012:cloudformation-alerts',
                  Message=message,
                  Subject=f'CloudFormation Alert: {stack_name}'
              )
          
          def send_success_notification(stack_name, status):
              # Send success notification
              print(f"Stack {stack_name} completed successfully with status: {status}")
```

---

## 8. Termination Protection

### Enable Termination Protection
```bash
# Enable termination protection
aws cloudformation update-termination-protection \
  --stack-name MyStack \
  --enable-termination-protection

# Disable termination protection
aws cloudformation update-termination-protection \
  --stack-name MyStack \
  --no-enable-termination-protection

# Check termination protection status
aws cloudformation describe-stacks \
  --stack-name MyStack \
  --query 'Stacks[0].EnableTerminationProtection'
```

### Template with Termination Protection
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Critical production stack with termination protection'

Resources:
  ProductionDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceIdentifier: prod-database
      DBInstanceClass: db.r5.large
      Engine: postgres
      MasterUsername: postgres
      MasterUserPassword: !Ref DatabasePassword
      AllocatedStorage: 100
      BackupRetentionPeriod: 30
      MultiAZ: true
      StorageEncrypted: true

  CriticalS3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: critical-production-data
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
```

---

## 9. Common Exam Scenarios

### Scenario 1: Multi-account security baseline deployment
**Solution:**
- Use StackSets to deploy security baseline across organization
- Create administration and execution roles
- Deploy to organizational units for automatic inclusion of new accounts
- Use service-managed permissions for simplified setup

### Scenario 2: Custom resource for third-party API integration
**Solution:**
- Create Lambda-backed custom resource
- Implement proper error handling and rollback logic
- Use cfnresponse module for proper CloudFormation integration
- Handle Create, Update, and Delete operations appropriately

### Scenario 3: Secure parameter management across environments
**Solution:**
- Use dynamic references to retrieve secrets from Secrets Manager
- Store environment-specific parameters in Parameter Store
- Use different parameter paths for different environments
- Implement proper IAM permissions for parameter access

### Scenario 4: Automated rollback on application health issues
**Solution:**
- Configure rollback triggers with CloudWatch alarms
- Set appropriate monitoring time and alarm thresholds
- Implement health checks for application components
- Use multiple alarms for comprehensive monitoring

### Scenario 5: Third-party resource provider integration
**Solution:**
- Register third-party resource provider in CloudFormation Registry
- Activate resource type in target accounts/regions
- Use custom resource types in CloudFormation templates
- Handle provider-specific configuration and lifecycle

### Scenario 6: Cross-region disaster recovery setup
**Solution:**
- Use StackSets to deploy infrastructure across regions
- Implement region-specific parameter overrides
- Handle region-specific resource availability
- Set up cross-region replication and backup strategies

### Scenario 7: Notification and monitoring for stack operations
**Solution:**
- Configure SNS topics for stack notifications
- Set up EventBridge rules for stack state changes
- Implement Lambda functions for custom notification logic
- Integrate with external monitoring and alerting systems

### Scenario 8: Large-scale infrastructure standardization
**Solution:**
- Create reusable CloudFormation modules
- Use StackSets for organization-wide deployment
- Implement governance controls with stack policies
- Use AWS Config rules for compliance monitoring

---

## 10. CLI Commands Reference

### StackSets Commands
```bash
# StackSet management
aws cloudformation create-stack-set --stack-set-name Name --template-body file://template.yaml
aws cloudformation update-stack-set --stack-set-name Name --template-body file://template.yaml
aws cloudformation delete-stack-set --stack-set-name Name

# Stack instance management
aws cloudformation create-stack-instances --stack-set-name Name --accounts 123456789012 --regions us-east-1
aws cloudformation update-stack-instances --stack-set-name Name --accounts 123456789012 --regions us-east-1
aws cloudformation delete-stack-instances --stack-set-name Name --accounts 123456789012 --regions us-east-1

# StackSet operations
aws cloudformation list-stack-set-operations --stack-set-name Name
aws cloudformation describe-stack-set-operation --stack-set-name Name --operation-id OperationId
```

### Registry Commands
```bash
# Resource type management
aws cloudformation register-type --type RESOURCE --type-name MyCompany::MyService::MyResource
aws cloudformation activate-type --type RESOURCE --type-name MyCompany::MyService::MyResource
aws cloudformation deactivate-type --type RESOURCE --type-name MyCompany::MyService::MyResource
aws cloudformation list-types --type RESOURCE
aws cloudformation describe-type --type RESOURCE --type-name MyCompany::MyService::MyResource
```

### Advanced Stack Operations
```bash
# Termination protection
aws cloudformation update-termination-protection --stack-name MyStack --enable-termination-protection

# Rollback configuration
aws cloudformation create-stack --stack-name MyStack --template-body file://template.yaml \
  --rollback-configuration 'RollbackTriggers=[{Arn=arn:aws:cloudwatch:us-east-1:123456789012:alarm:MyAlarm,Type=AWS::CloudWatch::Alarm}],MonitoringTimeInMinutes=5'

# Stack notifications
aws cloudformation create-stack --stack-name MyStack --template-body file://template.yaml \
  --notification-arns arn:aws:sns:us-east-1:123456789012:MyTopic
```

---

## 11. Best Practices for DOP-C02 Exam

### StackSets Best Practices
- Use service-managed permissions when possible
- Implement proper failure tolerance and concurrency settings
- Use organizational units for automatic account inclusion
- Test StackSet operations in non-production environments first
- Monitor StackSet operations and handle failures appropriately

### Custom Resources Best Practices
- Implement comprehensive error handling
- Use proper timeout values for long-running operations
- Handle all three operations: Create, Update, Delete
- Use cfnresponse module for proper CloudFormation integration
- Implement idempotent operations where possible

### Dynamic References Best Practices
- Use Secrets Manager for sensitive data
- Use Parameter Store for configuration values
- Implement proper IAM permissions for parameter access
- Use parameter versioning for controlled updates
- Consider parameter hierarchies for organization

### Security Best Practices
- Use least privilege IAM roles and policies
- Enable termination protection for critical stacks
- Use encrypted storage for sensitive data
- Implement proper audit trails with CloudTrail
- Use stack policies to prevent unauthorized changes

---

## 12. Exam Tips

### What to Remember
- **StackSets** require administration and execution roles
- **Custom resources** must handle Create, Update, Delete operations
- **Dynamic references** are resolved at deployment time
- **Rollback triggers** monitor CloudWatch alarms during deployment
- **Termination protection** prevents accidental stack deletion
- **Registry** enables third-party resource providers
- **Service-managed StackSets** simplify permissions with AWS Organizations

### Common Traps
- StackSet operations are asynchronous and may take time
- Custom resources must send proper response to CloudFormation
- Dynamic references cannot be used in all template sections
- Rollback triggers only work during stack operations, not after
- Registry resource types must be activated before use
- Cross-account StackSets require proper trust relationships

### Scenario-Based Questions
- Focus on multi-account and multi-region deployment patterns
- Understand when to use custom resources vs native resources
- Know security best practices for parameter management
- Understand rollback and failure handling mechanisms
- Know integration patterns with other AWS services

---

## 13. Quick Reference Cheat Sheet

### StackSet Key Concepts
```
Administration Account → Creates and manages StackSets
Target Accounts → Where stack instances are deployed
Stack Instance → Reference to stack in account/region
Operation → Create, update, or delete action on StackSet
```

### Custom Resource Response Format
```python
cfnresponse.send(event, context, status, responseData, physicalResourceId)
# status: cfnresponse.SUCCESS or cfnresponse.FAILED
# responseData: dict with return values
# physicalResourceId: unique identifier for resource
```

### Dynamic Reference Formats
```yaml
# Parameter Store
'{{resolve:ssm:parameter-name:version}}'
'{{resolve:ssm-secure:parameter-name:version}}'

# Secrets Manager
'{{resolve:secretsmanager:secret-id:SecretString:json-key}}'
```

### Essential IAM Actions
```
# StackSets
cloudformation:CreateStackSet, UpdateStackSet, DeleteStackSet
cloudformation:CreateStackInstances, UpdateStackInstances, DeleteStackInstances

# Registry
cloudformation:RegisterType, ActivateType, DeactivateType
cloudformation:ListTypes, DescribeType

# Advanced features
cloudformation:UpdateTerminationProtection
```

---

## 14. Reference Links

### AWS Official Documentation
- [StackSets User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)
- [Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
- [Dynamic References](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html)
- [CloudFormation Registry](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/registry.html)
- [Rollback Triggers](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-rollback-triggers.html)

### Workshops and Examples
- [StackSets Workshop](https://catalog.workshops.aws/cfn-stacksets/en-US)
- [Custom Resources Examples](https://github.com/aws-cloudformation/custom-resource-helper)

---

## 15. Summary

These advanced CloudFormation features are critical for the DOP-C02 exam and represent sophisticated Infrastructure as Code capabilities. Key areas to master:

1. **StackSets** for multi-account and multi-region deployments
2. **Custom Resources** for extending CloudFormation capabilities
3. **Dynamic References** for secure parameter and secret management
4. **CloudFormation Registry** for third-party resource integration
5. **Rollback Configuration** for automated failure handling
6. **Stack Notifications** for monitoring and alerting
7. **Termination Protection** for critical resource safety
8. **Advanced IAM patterns** for cross-account operations
9. **Integration patterns** with other AWS services
10. **Best practices** for enterprise-scale deployments

Understanding these advanced features demonstrates deep CloudFormation expertise and is essential for success on DOP-C02 advanced Infrastructure as Code questions.