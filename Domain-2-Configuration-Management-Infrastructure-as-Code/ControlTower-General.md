# AWS Control Tower - DOP-C02 Exam Notes

## 1. Overview

**AWS Control Tower** provides an easy way to set up and govern a secure, multi-account AWS environment based on best practices established through AWS's experience working with thousands of enterprises.

### Key Characteristics
- **Landing Zone** - Pre-configured multi-account environment
- **Guardrails** - Preventive and detective controls for governance
- **Account Factory** - Automated account provisioning and configuration
- **Dashboard** - Centralized visibility and compliance monitoring
- **Service Catalog integration** - Self-service account provisioning
- **Organizations integration** - Built on AWS Organizations foundation
- **Config integration** - Continuous compliance monitoring
- **CloudTrail integration** - Centralized logging and auditing

### What Problem Does It Solve?
- Simplifies multi-account AWS environment setup and governance
- Enforces security and compliance policies across accounts
- Provides centralized visibility and control
- Automates account provisioning with consistent configuration
- Reduces operational overhead for managing multiple accounts
- Ensures adherence to AWS best practices and compliance requirements

---

## 2. Core Components

### Landing Zone
```yaml
# Landing Zone automatically creates:
# 1. Root Organization Unit (OU)
# 2. Security OU (for shared security accounts)
# 3. Sandbox OU (for development/testing accounts)
# 4. Log Archive Account (centralized logging)
# 5. Audit Account (security and compliance monitoring)

# Example OU structure after Control Tower setup:
Root
├── Security OU
│   ├── Log Archive Account
│   └── Audit Account
├── Sandbox OU
│   ├── Development Account 1
│   ├── Development Account 2
│   └── Testing Account
└── Production OU (custom)
    ├── Production Account 1
    └── Production Account 2
```

### Guardrails
```json
{
  "preventiveGuardrails": [
    {
      "name": "Disallow policy changes to log archive",
      "description": "Prevents modification of CloudTrail logs in log archive account",
      "severity": "Mandatory",
      "implementation": "SCP (Service Control Policy)"
    },
    {
      "name": "Disallow changes to encryption configuration for log archive",
      "description": "Prevents changes to S3 bucket encryption in log archive",
      "severity": "Mandatory",
      "implementation": "SCP"
    },
    {
      "name": "Disallow configuration changes to CloudTrail",
      "description": "Prevents unauthorized changes to CloudTrail configuration",
      "severity": "Mandatory",
      "implementation": "SCP"
    }
  ],
  "detectiveGuardrails": [
    {
      "name": "Detect whether public read access to Amazon S3 buckets is allowed",
      "description": "Monitors S3 buckets for public read access",
      "severity": "Strongly Recommended",
      "implementation": "AWS Config Rule"
    },
    {
      "name": "Detect whether public write access to Amazon S3 buckets is allowed",
      "description": "Monitors S3 buckets for public write access",
      "severity": "Strongly Recommended",
      "implementation": "AWS Config Rule"
    },
    {
      "name": "Detect whether MFA for the root user is enabled",
      "description": "Checks if MFA is enabled for root user",
      "severity": "Strongly Recommended",
      "implementation": "AWS Config Rule"
    }
  ]
}
```

### Account Factory
```yaml
# Account Factory creates accounts with:
# 1. Baseline security configuration
# 2. Guardrails applied
# 3. CloudTrail logging enabled
# 4. Config recording enabled
# 5. Cross-account access roles

# Example Account Factory configuration
AccountFactoryConfiguration:
  AccountEmail: "new-account@company.com"
  AccountName: "Production-WebApp"
  OrganizationalUnit: "Production OU"
  SSOUserEmail: "admin@company.com"
  SSOUserFirstName: "Admin"
  SSOUserLastName: "User"
  ManagedOrganizationalUnit: "Production OU"
  
  # Baseline configuration applied:
  BaselineConfiguration:
    - CloudTrail: Enabled
    - Config: Enabled
    - GuardrailsApplied: All mandatory + selected optional
    - CrossAccountRoles:
        - AWSControlTowerExecution
        - OrganizationAccountAccessRole
```

---

## 3. Guardrails Implementation

### Service Control Policies (SCPs)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PreventCloudTrailDisabling",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:PutEventSelectors"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/AWSControlTowerExecution"
          ]
        }
      }
    },
    {
      "Sid": "PreventConfigDisabling",
      "Effect": "Deny",
      "Action": [
        "config:StopConfigurationRecorder",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/AWSControlTowerExecution"
          ]
        }
      }
    },
    {
      "Sid": "PreventLogArchiveModification",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteBucketPolicy",
        "s3:PutBucketPolicy"
      ],
      "Resource": [
        "arn:aws:s3:::aws-controltower-logs-*",
        "arn:aws:s3:::aws-controltower-s3-access-logs-*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/AWSControlTowerExecution"
          ]
        }
      }
    }
  ]
}
```

### Config Rules for Detective Guardrails
```yaml
# CloudFormation template for custom guardrails
Resources:
  # Custom detective guardrail for EC2 instance compliance
  EC2InstanceComplianceRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ct-ec2-instance-compliance
      Description: Detect non-compliant EC2 instances
      Source:
        Owner: AWS_LAMBDA
        SourceIdentifier: !GetAtt ComplianceFunction.Arn
        SourceDetail:
          - EventSource: aws.config
            MessageType: ConfigurationItemChangeNotification
            MaximumExecutionFrequency: TwentyFour_Hours

  ComplianceFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: ct-ec2-compliance-check
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt ComplianceFunctionRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          
          def lambda_handler(event, context):
              config_item = event['configurationItem']
              
              if config_item['resourceType'] != 'AWS::EC2::Instance':
                  return {
                      'compliance_type': 'NOT_APPLICABLE',
                      'annotation': 'Rule only applies to EC2 instances'
                  }
              
              # Check required tags
              required_tags = ['Environment', 'Owner', 'CostCenter']
              instance_tags = config_item.get('tags', {})
              
              missing_tags = [tag for tag in required_tags if tag not in instance_tags]
              
              if missing_tags:
                  return {
                      'compliance_type': 'NON_COMPLIANT',
                      'annotation': f'Missing required tags: {", ".join(missing_tags)}'
                  }
              
              # Check instance type restrictions
              allowed_types = ['t3.micro', 't3.small', 't3.medium', 't3.large']
              instance_type = config_item['configuration']['instanceType']
              
              if instance_type not in allowed_types:
                  return {
                      'compliance_type': 'NON_COMPLIANT',
                      'annotation': f'Instance type {instance_type} not allowed'
                  }
              
              return {
                  'compliance_type': 'COMPLIANT',
                  'annotation': 'Instance meets all compliance requirements'
              }

  # Remediation for non-compliant resources
  ComplianceRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref EC2InstanceComplianceRule
      TargetType: SSM_DOCUMENT
      TargetId: !Ref RemediationDocument
      TargetVersion: "1"
      Parameters:
        AutomationAssumeRole:
          StaticValue: !GetAtt RemediationRole.Arn
        InstanceId:
          ResourceValue: RESOURCE_ID
      Automatic: false
      MaximumAutomaticAttempts: 1

  RemediationDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Automation
      DocumentFormat: YAML
      Content:
        schemaVersion: '0.3'
        description: Remediate non-compliant EC2 instances
        assumeRole: '{{ AutomationAssumeRole }}'
        parameters:
          InstanceId:
            type: String
            description: EC2 Instance ID
          AutomationAssumeRole:
            type: String
            description: IAM role for automation
        mainSteps:
          - name: AddComplianceTags
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: CreateTags
              Resources:
                - '{{ InstanceId }}'
              Tags:
                - Key: ComplianceStatus
                  Value: Remediated
                - Key: RemediationDate
                  Value: '{{ global:DATE_TIME }}'
```

---

## 4. Account Factory Automation

### Service Catalog Integration
```yaml
# Account Factory as Service Catalog product
Resources:
  AccountFactoryPortfolio:
    Type: AWS::ServiceCatalog::Portfolio
    Properties:
      DisplayName: AWS Control Tower Account Factory
      Description: Automated account provisioning with Control Tower
      ProviderName: AWS Control Tower

  AccountFactoryProduct:
    Type: AWS::ServiceCatalog::CloudFormationProduct
    Properties:
      Name: AWS Control Tower Account Factory
      Description: Create new AWS account with Control Tower baseline
      Owner: AWS Control Tower
      ProvisioningArtifactParameters:
        - Name: v1.0
          Description: Account Factory template
          Info:
            LoadTemplateFromURL: !Sub "https://${TemplateBucket}.s3.amazonaws.com/account-factory-template.yaml"

  # Custom account provisioning workflow
  CustomAccountFactory:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: custom-account-factory
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt AccountFactoryRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          import time
          
          def lambda_handler(event, context):
              """Custom account factory with additional configuration"""
              
              servicecatalog = boto3.client('servicecatalog')
              organizations = boto3.client('organizations')
              
              # Extract account details from event
              account_name = event['AccountName']
              account_email = event['AccountEmail']
              ou_name = event['OrganizationalUnit']
              
              try:
                  # Provision account through Service Catalog
                  response = servicecatalog.provision_product(
                      ProductId='prod-account-factory',
                      ProvisioningArtifactId='pa-latest',
                      ProvisionedProductName=f"Account-{account_name}",
                      ProvisioningParameters=[
                          {'Key': 'AccountName', 'Value': account_name},
                          {'Key': 'AccountEmail', 'Value': account_email},
                          {'Key': 'OrganizationalUnitName', 'Value': ou_name},
                          {'Key': 'SSOUserEmail', 'Value': event.get('SSOUserEmail', account_email)},
                          {'Key': 'SSOUserFirstName', 'Value': event.get('SSOUserFirstName', 'Admin')},
                          {'Key': 'SSOUserLastName', 'Value': event.get('SSOUserLastName', 'User')}
                      ]
                  )
                  
                  record_id = response['RecordDetail']['RecordId']
                  
                  # Wait for provisioning to complete
                  while True:
                      record = servicecatalog.describe_record(Id=record_id)
                      status = record['RecordDetail']['Status']
                      
                      if status == 'SUCCEEDED':
                          # Get account ID from outputs
                          account_id = None
                          for output in record['RecordDetail']['RecordOutputs']:
                              if output['OutputKey'] == 'AccountId':
                                  account_id = output['OutputValue']
                                  break
                          
                          if account_id:
                              # Apply additional configuration
                              apply_custom_configuration(account_id, event)
                          
                          return {
                              'statusCode': 200,
                              'body': json.dumps({
                                  'AccountId': account_id,
                                  'Status': 'SUCCESS'
                              })
                          }
                      
                      elif status in ['FAILED', 'ERROR']:
                          return {
                              'statusCode': 500,
                              'body': json.dumps({
                                  'Status': 'FAILED',
                                  'Error': record['RecordDetail'].get('RecordErrors', [])
                              })
                          }
                      
                      time.sleep(30)  # Wait 30 seconds before checking again
              
              except Exception as e:
                  return {
                      'statusCode': 500,
                      'body': json.dumps({
                          'Status': 'FAILED',
                          'Error': str(e)
                      })
                  }
          
          def apply_custom_configuration(account_id, config):
              """Apply custom configuration to newly created account"""
              
              # Assume role in target account
              sts = boto3.client('sts')
              assumed_role = sts.assume_role(
                  RoleArn=f"arn:aws:iam::{account_id}:role/OrganizationAccountAccessRole",
                  RoleSessionName='CustomConfiguration'
              )
              
              # Create session with assumed role credentials
              session = boto3.Session(
                  aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
                  aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
                  aws_session_token=assumed_role['Credentials']['SessionToken']
              )
              
              # Apply custom configurations
              if config.get('EnableCloudTrail'):
                  setup_additional_cloudtrail(session, config)
              
              if config.get('CreateVPC'):
                  create_default_vpc(session, config)
              
              if config.get('SetupBudgets'):
                  setup_cost_budgets(session, config)
```

### Automated Account Onboarding
```python
# Complete account onboarding workflow
import boto3
import json
from datetime import datetime, timedelta

class ControlTowerAccountManager:
    def __init__(self):
        self.servicecatalog = boto3.client('servicecatalog')
        self.organizations = boto3.client('organizations')
        self.sts = boto3.client('sts')
        self.stepfunctions = boto3.client('stepfunctions')
    
    def create_account_with_workflow(self, account_request):
        """Create account with complete onboarding workflow"""
        
        # Start Step Functions workflow for account creation
        workflow_input = {
            'AccountRequest': account_request,
            'Timestamp': datetime.utcnow().isoformat()
        }
        
        response = self.stepfunctions.start_execution(
            stateMachineArn='arn:aws:states:us-east-1:123456789012:stateMachine:AccountOnboarding',
            name=f"account-{account_request['AccountName']}-{int(time.time())}",
            input=json.dumps(workflow_input)
        )
        
        return response['executionArn']
    
    def provision_account_factory(self, account_details):
        """Provision account through Control Tower Account Factory"""
        
        # Find Account Factory product
        products = self.servicecatalog.search_products(
            Filters={'FullTextSearch': ['AWS Control Tower Account Factory']}
        )
        
        if not products['ProductViewSummaries']:
            raise ValueError("Account Factory product not found")
        
        product = products['ProductViewSummaries'][0]
        
        # Get latest provisioning artifact
        product_details = self.servicecatalog.describe_product(Id=product['ProductId'])
        latest_artifact = product_details['ProvisioningArtifacts'][-1]
        
        # Provision account
        response = self.servicecatalog.provision_product(
            ProductId=product['ProductId'],
            ProvisioningArtifactId=latest_artifact['Id'],
            ProvisionedProductName=f"Account-{account_details['AccountName']}",
            ProvisioningParameters=[
                {'Key': 'AccountName', 'Value': account_details['AccountName']},
                {'Key': 'AccountEmail', 'Value': account_details['AccountEmail']},
                {'Key': 'OrganizationalUnitName', 'Value': account_details['OrganizationalUnit']},
                {'Key': 'SSOUserEmail', 'Value': account_details['SSOUserEmail']},
                {'Key': 'SSOUserFirstName', 'Value': account_details['SSOUserFirstName']},
                {'Key': 'SSOUserLastName', 'Value': account_details['SSOUserLastName']}
            ],
            Tags=[
                {'Key': 'CreatedBy', 'Value': 'ControlTowerAccountManager'},
                {'Key': 'CreationDate', 'Value': datetime.utcnow().isoformat()},
                {'Key': 'Department', 'Value': account_details.get('Department', 'Unknown')}
            ]
        )
        
        return response['RecordDetail']
    
    def setup_account_baseline(self, account_id, baseline_config):
        """Setup baseline configuration for new account"""
        
        # Assume role in target account
        assumed_role = self.sts.assume_role(
            RoleArn=f"arn:aws:iam::{account_id}:role/OrganizationAccountAccessRole",
            RoleSessionName='BaselineSetup'
        )
        
        # Create session with assumed role
        session = boto3.Session(
            aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
            aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
            aws_session_token=assumed_role['Credentials']['SessionToken']
        )
        
        # Setup baseline components
        results = {}
        
        if baseline_config.get('CreateVPC'):
            results['VPC'] = self.create_baseline_vpc(session, baseline_config['VPC'])
        
        if baseline_config.get('SetupBudgets'):
            results['Budgets'] = self.setup_cost_budgets(session, baseline_config['Budgets'])
        
        if baseline_config.get('CreateRoles'):
            results['Roles'] = self.create_baseline_roles(session, baseline_config['Roles'])
        
        if baseline_config.get('SetupCloudWatch'):
            results['CloudWatch'] = self.setup_cloudwatch_baseline(session, baseline_config['CloudWatch'])
        
        return results
    
    def create_baseline_vpc(self, session, vpc_config):
        """Create baseline VPC configuration"""
        
        ec2 = session.client('ec2')
        
        # Create VPC
        vpc_response = ec2.create_vpc(
            CidrBlock=vpc_config.get('CidrBlock', '10.0.0.0/16'),
            TagSpecifications=[
                {
                    'ResourceType': 'vpc',
                    'Tags': [
                        {'Key': 'Name', 'Value': vpc_config.get('Name', 'BaselineVPC')},
                        {'Key': 'CreatedBy', 'Value': 'ControlTowerBaseline'}
                    ]
                }
            ]
        )
        
        vpc_id = vpc_response['Vpc']['VpcId']
        
        # Create subnets
        subnets = []
        for i, subnet_config in enumerate(vpc_config.get('Subnets', [])):
            subnet_response = ec2.create_subnet(
                VpcId=vpc_id,
                CidrBlock=subnet_config['CidrBlock'],
                AvailabilityZone=subnet_config['AvailabilityZone'],
                TagSpecifications=[
                    {
                        'ResourceType': 'subnet',
                        'Tags': [
                            {'Key': 'Name', 'Value': subnet_config['Name']},
                            {'Key': 'Type', 'Value': subnet_config.get('Type', 'Private')}
                        ]
                    }
                ]
            )
            subnets.append(subnet_response['Subnet']['SubnetId'])
        
        return {
            'VpcId': vpc_id,
            'Subnets': subnets
        }
    
    def setup_cost_budgets(self, session, budget_config):
        """Setup cost budgets for the account"""
        
        budgets = session.client('budgets')
        account_id = session.client('sts').get_caller_identity()['Account']
        
        created_budgets = []
        
        for budget in budget_config:
            budget_definition = {
                'BudgetName': budget['Name'],
                'BudgetLimit': {
                    'Amount': str(budget['Amount']),
                    'Unit': 'USD'
                },
                'TimeUnit': budget.get('TimeUnit', 'MONTHLY'),
                'BudgetType': budget.get('Type', 'COST'),
                'CostFilters': budget.get('CostFilters', {}),
                'TimePeriod': {
                    'Start': datetime.utcnow().replace(day=1),
                    'End': datetime.utcnow().replace(day=28) + timedelta(days=4)
                }
            }
            
            # Create budget
            budgets.create_budget(
                AccountId=account_id,
                Budget=budget_definition,
                NotificationsWithSubscribers=[
                    {
                        'Notification': {
                            'NotificationType': 'ACTUAL',
                            'ComparisonOperator': 'GREATER_THAN',
                            'Threshold': 80.0,
                            'ThresholdType': 'PERCENTAGE'
                        },
                        'Subscribers': [
                            {
                                'SubscriptionType': 'EMAIL',
                                'Address': budget['NotificationEmail']
                            }
                        ]
                    }
                ]
            )
            
            created_budgets.append(budget['Name'])
        
        return created_budgets

# Step Functions state machine for account onboarding
step_functions_definition = {
    "Comment": "Account onboarding workflow",
    "StartAt": "ProvisionAccount",
    "States": {
        "ProvisionAccount": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:provision-account",
            "Next": "WaitForProvisioning"
        },
        "WaitForProvisioning": {
            "Type": "Wait",
            "Seconds": 300,
            "Next": "CheckProvisioningStatus"
        },
        "CheckProvisioningStatus": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:check-provisioning-status",
            "Next": "IsProvisioningComplete"
        },
        "IsProvisioningComplete": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.Status",
                    "StringEquals": "SUCCEEDED",
                    "Next": "SetupBaseline"
                },
                {
                    "Variable": "$.Status",
                    "StringEquals": "FAILED",
                    "Next": "ProvisioningFailed"
                }
            ],
            "Default": "WaitForProvisioning"
        },
        "SetupBaseline": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:setup-baseline",
            "Next": "NotifyCompletion"
        },
        "NotifyCompletion": {
            "Type": "Task",
            "Resource": "arn:aws:states:::sns:publish",
            "Parameters": {
                "TopicArn": "arn:aws:sns:us-east-1:123456789012:account-onboarding",
                "Message.$": "$.CompletionMessage"
            },
            "End": True
        },
        "ProvisioningFailed": {
            "Type": "Task",
            "Resource": "arn:aws:states:::sns:publish",
            "Parameters": {
                "TopicArn": "arn:aws:sns:us-east-1:123456789012:account-onboarding-failures",
                "Message.$": "$.ErrorMessage"
            },
            "End": True
        }
    }
}
```

---

## 5. Customization and Extensions

### Custom Guardrails
```yaml
# Custom guardrail for encryption compliance
Resources:
  EncryptionComplianceRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ct-encryption-compliance
      Description: Ensure all resources use encryption
      Source:
        Owner: AWS_LAMBDA
        SourceIdentifier: !GetAtt EncryptionComplianceFunction.Arn
        SourceDetail:
          - EventSource: aws.config
            MessageType: ConfigurationItemChangeNotification

  EncryptionComplianceFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: ct-encryption-compliance
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt ComplianceFunctionRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          
          def lambda_handler(event, context):
              config_item = event['configurationItem']
              resource_type = config_item['resourceType']
              
              # Check encryption for different resource types
              if resource_type == 'AWS::S3::Bucket':
                  return check_s3_encryption(config_item)
              elif resource_type == 'AWS::RDS::DBInstance':
                  return check_rds_encryption(config_item)
              elif resource_type == 'AWS::EBS::Volume':
                  return check_ebs_encryption(config_item)
              else:
                  return {
                      'compliance_type': 'NOT_APPLICABLE',
                      'annotation': f'Encryption check not applicable for {resource_type}'
                  }
          
          def check_s3_encryption(config_item):
              """Check S3 bucket encryption"""
              configuration = config_item['configuration']
              
              # Check if bucket has encryption configuration
              if 'serverSideEncryptionConfiguration' in configuration:
                  encryption_rules = configuration['serverSideEncryptionConfiguration']['rules']
                  if encryption_rules:
                      return {
                          'compliance_type': 'COMPLIANT',
                          'annotation': 'S3 bucket has encryption enabled'
                      }
              
              return {
                  'compliance_type': 'NON_COMPLIANT',
                  'annotation': 'S3 bucket does not have encryption enabled'
              }
          
          def check_rds_encryption(config_item):
              """Check RDS encryption"""
              configuration = config_item['configuration']
              
              if configuration.get('storageEncrypted', False):
                  return {
                      'compliance_type': 'COMPLIANT',
                      'annotation': 'RDS instance has encryption enabled'
                  }
              
              return {
                  'compliance_type': 'NON_COMPLIANT',
                  'annotation': 'RDS instance does not have encryption enabled'
              }
          
          def check_ebs_encryption(config_item):
              """Check EBS volume encryption"""
              configuration = config_item['configuration']
              
              if configuration.get('encrypted', False):
                  return {
                      'compliance_type': 'COMPLIANT',
                      'annotation': 'EBS volume is encrypted'
                  }
              
              return {
                  'compliance_type': 'NON_COMPLIANT',
                  'annotation': 'EBS volume is not encrypted'
              }

  # Deploy custom guardrail to all accounts
  CustomGuardrailStackSet:
    Type: AWS::CloudFormation::StackSet
    Properties:
      StackSetName: ct-custom-encryption-guardrail
      Description: Deploy custom encryption guardrail to all accounts
      Capabilities:
        - CAPABILITY_IAM
      Parameters:
        - ParameterKey: GuardrailName
          ParameterValue: ct-encryption-compliance
      PermissionModel: SERVICE_MANAGED
      AutoDeployment:
        Enabled: true
        RetainStacksOnAccountRemoval: false
      OperationPreferences:
        RegionConcurrencyType: PARALLEL
        MaxConcurrentPercentage: 100
        FailureTolerancePercentage: 10
      TemplateBody: !Sub |
        AWSTemplateFormatVersion: '2010-09-09'
        Description: Custom encryption compliance guardrail
        Parameters:
          GuardrailName:
            Type: String
        Resources:
          ${EncryptionComplianceRule}
          ${EncryptionComplianceFunction}
          ${ComplianceFunctionRole}
```

### Lifecycle Management
```python
# Account lifecycle management
import boto3
import json
from datetime import datetime, timedelta

class ControlTowerLifecycleManager:
    def __init__(self):
        self.organizations = boto3.client('organizations')
        self.controltower = boto3.client('controltower')
        self.config = boto3.client('config')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def monitor_account_compliance(self):
        """Monitor compliance across all managed accounts"""
        
        # Get all accounts in organization
        accounts = self.organizations.list_accounts()
        
        compliance_report = {
            'timestamp': datetime.utcnow().isoformat(),
            'total_accounts': len(accounts['Accounts']),
            'compliant_accounts': 0,
            'non_compliant_accounts': 0,
            'violations': []
        }
        
        for account in accounts['Accounts']:
            account_id = account['Id']
            account_name = account['Name']
            
            # Skip master account and suspended accounts
            if account['Status'] != 'ACTIVE':
                continue
            
            # Check guardrail compliance
            violations = self.check_account_guardrails(account_id)
            
            if violations:
                compliance_report['non_compliant_accounts'] += 1
                compliance_report['violations'].extend([
                    {
                        'account_id': account_id,
                        'account_name': account_name,
                        'violation': violation
                    } for violation in violations
                ])
            else:
                compliance_report['compliant_accounts'] += 1
        
        # Send compliance metrics to CloudWatch
        self.send_compliance_metrics(compliance_report)
        
        return compliance_report
    
    def check_account_guardrails(self, account_id):
        """Check guardrail compliance for a specific account"""
        
        violations = []
        
        try:
            # Assume role in target account to check Config compliance
            sts = boto3.client('sts')
            assumed_role = sts.assume_role(
                RoleArn=f"arn:aws:iam::{account_id}:role/AWSControlTowerExecution",
                RoleSessionName='ComplianceCheck'
            )
            
            # Create session with assumed role
            session = boto3.Session(
                aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
                aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
                aws_session_token=assumed_role['Credentials']['SessionToken']
            )
            
            config_client = session.client('config')
            
            # Get compliance summary
            compliance_summary = config_client.get_compliance_summary_by_config_rule()
            
            # Check for non-compliant rules
            for rule_name in self.get_mandatory_guardrails():
                try:
                    compliance_details = config_client.get_compliance_details_by_config_rule(
                        ConfigRuleName=rule_name,
                        ComplianceTypes=['NON_COMPLIANT']
                    )
                    
                    if compliance_details['EvaluationResults']:
                        violations.append({
                            'rule_name': rule_name,
                            'non_compliant_resources': len(compliance_details['EvaluationResults']),
                            'details': compliance_details['EvaluationResults'][:5]  # First 5 violations
                        })
                
                except config_client.exceptions.NoSuchConfigRuleException:
                    violations.append({
                        'rule_name': rule_name,
                        'error': 'Config rule not found - guardrail may not be enabled'
                    })
        
        except Exception as e:
            violations.append({
                'error': f'Unable to check compliance: {str(e)}'
            })
        
        return violations
    
    def get_mandatory_guardrails(self):
        """Get list of mandatory Control Tower guardrails"""
        return [
            'ct-s3-bucket-public-read-prohibited',
            'ct-s3-bucket-public-write-prohibited',
            'ct-cloudtrail-enabled',
            'ct-config-enabled',
            'ct-root-mfa-enabled'
        ]
    
    def remediate_account_violations(self, account_id, violations):
        """Attempt to remediate violations in an account"""
        
        remediation_results = []
        
        for violation in violations:
            rule_name = violation.get('rule_name')
            
            if rule_name == 'ct-s3-bucket-public-read-prohibited':
                result = self.remediate_s3_public_access(account_id, violation)
                remediation_results.append(result)
            
            elif rule_name == 'ct-cloudtrail-enabled':
                result = self.remediate_cloudtrail_configuration(account_id, violation)
                remediation_results.append(result)
            
            # Add more remediation actions as needed
        
        return remediation_results
    
    def send_compliance_metrics(self, compliance_report):
        """Send compliance metrics to CloudWatch"""
        
        self.cloudwatch.put_metric_data(
            Namespace='ControlTower/Compliance',
            MetricData=[
                {
                    'MetricName': 'CompliantAccounts',
                    'Value': compliance_report['compliant_accounts'],
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                },
                {
                    'MetricName': 'NonCompliantAccounts',
                    'Value': compliance_report['non_compliant_accounts'],
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                },
                {
                    'MetricName': 'TotalViolations',
                    'Value': len(compliance_report['violations']),
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
```

---

## 6. Integration with CI/CD

### Account Provisioning Pipeline
```yaml
# CodePipeline for automated account provisioning
Resources:
  AccountProvisioningPipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt CodePipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactsBucket
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: S3
                Version: '1'
              Configuration:
                S3Bucket: !Ref RequestsBucket
                S3ObjectKey: account-requests/
                PollForSourceChanges: true
              OutputArtifacts:
                - Name: SourceOutput

        - Name: Validate
          Actions:
            - Name: ValidateRequest
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: Lambda
                Version: '1'
              Configuration:
                FunctionName: !Ref ValidateAccountRequestFunction
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: ValidatedOutput

        - Name: Approval
          Actions:
            - Name: ManualApproval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              Configuration:
                CustomData: 'Please review the account provisioning request'

        - Name: Provision
          Actions:
            - Name: ProvisionAccount
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: StepFunctions
                Version: '1'
              Configuration:
                StateMachineArn: !Ref AccountProvisioningStateMachine
                Input: |
                  {
                    "AccountRequest.$": "$.AccountRequest"
                  }
              InputArtifacts:
                - Name: ValidatedOutput

        - Name: Configure
          Actions:
            - Name: ConfigureBaseline
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: Lambda
                Version: '1'
              Configuration:
                FunctionName: !Ref ConfigureBaselineFunction
              InputArtifacts:
                - Name: ValidatedOutput

        - Name: Notify
          Actions:
            - Name: NotifyCompletion
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: SNS
                Version: '1'
              Configuration:
                TopicArn: !Ref AccountProvisioningNotifications
                Message: 'Account provisioning completed successfully'
```

---

## 7. Common Exam Scenarios

### Scenario 1: Multi-Account Governance Setup
```yaml
# Complete Control Tower setup with custom guardrails
Resources:
  # Custom OU for production workloads
  ProductionOU:
    Type: AWS::Organizations::OrganizationalUnit
    Properties:
      Name: Production
      ParentId: !Ref RootId

  # Custom guardrail for production OU
  ProductionGuardrail:
    Type: AWS::Config::OrganizationConfigRule
    Properties:
      OrganizationConfigRuleName: production-encryption-required
      OrganizationManagedRuleMetadata:
        RuleIdentifier: ENCRYPTED_VOLUMES
        Description: Ensure all EBS volumes are encrypted in production
      IncludedAccounts:
        - !Ref ProductionAccountId1
        - !Ref ProductionAccountId2

  # SCP for production OU
  ProductionSCP:
    Type: AWS::Organizations::Policy
    Properties:
      Name: ProductionSecurityPolicy
      Description: Security policy for production accounts
      Type: SERVICE_CONTROL_POLICY
      Content: |
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "DenyUnencryptedStorage",
              "Effect": "Deny",
              "Action": [
                "ec2:CreateVolume",
                "rds:CreateDBInstance"
              ],
              "Resource": "*",
              "Condition": {
                "Bool": {
                  "ec2:Encrypted": "false"
                }
              }
            },
            {
              "Sid": "RequireMFAForSensitiveActions",
              "Effect": "Deny",
              "Action": [
                "ec2:TerminateInstances",
                "rds:DeleteDBInstance",
                "s3:DeleteBucket"
              ],
              "Resource": "*",
              "Condition": {
                "BoolIfExists": {
                  "aws:MultiFactorAuthPresent": "false"
                }
              }
            }
          ]
        }

  # Attach SCP to production OU
  ProductionSCPAttachment:
    Type: AWS::Organizations::PolicyAttachment
    Properties:
      PolicyId: !Ref ProductionSCP
      TargetId: !Ref ProductionOU
      TargetType: ORGANIZATIONAL_UNIT
```

### Scenario 2: Automated Compliance Monitoring
```python
# Complete compliance monitoring and remediation system
import boto3
import json
from datetime import datetime, timedelta

class ControlTowerComplianceOrchestrator:
    def __init__(self):
        self.organizations = boto3.client('organizations')
        self.config = boto3.client('config')
        self.sns = boto3.client('sns')
        self.stepfunctions = boto3.client('stepfunctions')
    
    def lambda_handler(self, event, context):
        """Main handler for compliance monitoring"""
        
        # Scheduled compliance check
        if event.get('source') == 'aws.events':
            return self.run_scheduled_compliance_check()
        
        # Config rule evaluation change
        elif event.get('source') == 'aws.config':
            return self.handle_config_evaluation_change(event)
        
        # Manual compliance check
        else:
            return self.run_manual_compliance_check(event)
    
    def run_scheduled_compliance_check(self):
        """Run scheduled compliance check across all accounts"""
        
        # Get all accounts in organization
        accounts = self.organizations.list_accounts()
        
        compliance_summary = {
            'timestamp': datetime.utcnow().isoformat(),
            'total_accounts': 0,
            'compliant_accounts': 0,
            'non_compliant_accounts': 0,
            'critical_violations': 0,
            'accounts_details': []
        }
        
        for account in accounts['Accounts']:
            if account['Status'] != 'ACTIVE':
                continue
            
            compliance_summary['total_accounts'] += 1
            account_compliance = self.check_account_compliance(account['Id'])
            
            if account_compliance['is_compliant']:
                compliance_summary['compliant_accounts'] += 1
            else:
                compliance_summary['non_compliant_accounts'] += 1
                
                # Count critical violations
                critical_violations = [v for v in account_compliance['violations'] 
                                     if v.get('severity') == 'CRITICAL']
                compliance_summary['critical_violations'] += len(critical_violations)
                
                # Trigger remediation for critical violations
                if critical_violations:
                    self.trigger_remediation_workflow(account['Id'], critical_violations)
            
            compliance_summary['accounts_details'].append({
                'account_id': account['Id'],
                'account_name': account['Name'],
                'compliance_status': 'COMPLIANT' if account_compliance['is_compliant'] else 'NON_COMPLIANT',
                'violations_count': len(account_compliance['violations'])
            })
        
        # Send summary notification
        self.send_compliance_summary(compliance_summary)
        
        return compliance_summary
    
    def check_account_compliance(self, account_id):
        """Check compliance for a specific account"""
        
        violations = []
        is_compliant = True
        
        try:
            # Assume role in target account
            sts = boto3.client('sts')
            assumed_role = sts.assume_role(
                RoleArn=f"arn:aws:iam::{account_id}:role/AWSControlTowerExecution",
                RoleSessionName='ComplianceCheck'
            )
            
            session = boto3.Session(
                aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
                aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
                aws_session_token=assumed_role['Credentials']['SessionToken']
            )
            
            config_client = session.client('config')
            
            # Check mandatory guardrails
            mandatory_rules = [
                {'name': 'ct-s3-bucket-public-read-prohibited', 'severity': 'CRITICAL'},
                {'name': 'ct-s3-bucket-public-write-prohibited', 'severity': 'CRITICAL'},
                {'name': 'ct-cloudtrail-enabled', 'severity': 'HIGH'},
                {'name': 'ct-config-enabled', 'severity': 'HIGH'},
                {'name': 'ct-root-mfa-enabled', 'severity': 'CRITICAL'}
            ]
            
            for rule in mandatory_rules:
                try:
                    compliance_details = config_client.get_compliance_details_by_config_rule(
                        ConfigRuleName=rule['name'],
                        ComplianceTypes=['NON_COMPLIANT']
                    )
                    
                    if compliance_details['EvaluationResults']:
                        is_compliant = False
                        violations.append({
                            'rule_name': rule['name'],
                            'severity': rule['severity'],
                            'non_compliant_resources': len(compliance_details['EvaluationResults']),
                            'resources': [r['EvaluationResultIdentifier']['EvaluationResultQualifier'] 
                                        for r in compliance_details['EvaluationResults'][:10]]
                        })
                
                except config_client.exceptions.NoSuchConfigRuleException:
                    is_compliant = False
                    violations.append({
                        'rule_name': rule['name'],
                        'severity': 'CRITICAL',
                        'error': 'Config rule not found - guardrail not enabled'
                    })
        
        except Exception as e:
            is_compliant = False
            violations.append({
                'error': f'Unable to check compliance: {str(e)}',
                'severity': 'HIGH'
            })
        
        return {
            'is_compliant': is_compliant,
            'violations': violations
        }
    
    def trigger_remediation_workflow(self, account_id, violations):
        """Trigger Step Functions workflow for remediation"""
        
        workflow_input = {
            'AccountId': account_id,
            'Violations': violations,
            'Timestamp': datetime.utcnow().isoformat()
        }
        
        self.stepfunctions.start_execution(
            stateMachineArn='arn:aws:states:us-east-1:123456789012:stateMachine:ComplianceRemediation',
            name=f"remediation-{account_id}-{int(datetime.utcnow().timestamp())}",
            input=json.dumps(workflow_input)
        )
    
    def send_compliance_summary(self, summary):
        """Send compliance summary notification"""
        
        message = f"""
Control Tower Compliance Summary
===============================

Timestamp: {summary['timestamp']}
Total Accounts: {summary['total_accounts']}
Compliant Accounts: {summary['compliant_accounts']}
Non-Compliant Accounts: {summary['non_compliant_accounts']}
Critical Violations: {summary['critical_violations']}

Compliance Rate: {(summary['compliant_accounts'] / summary['total_accounts'] * 100):.1f}%

Account Details:
{json.dumps(summary['accounts_details'], indent=2)}
        """
        
        self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:controltower-compliance',
            Message=message,
            Subject=f"Control Tower Compliance Report - {summary['non_compliant_accounts']} Non-Compliant Accounts"
        )
```

---

## 8. Exam Tips

- **Understand Landing Zone** - Core components and automatic setup
- **Master Guardrails** - Preventive (SCP) vs Detective (Config) guardrails
- **Know Account Factory** - Service Catalog integration and automation
- **Practice customization** - Custom guardrails and OU structures
- **Learn integration** - Organizations, Config, CloudTrail, Service Catalog
- **Understand governance** - Multi-account governance and compliance
- **Know limitations** - What Control Tower can and cannot do
- **Practice troubleshooting** - Common setup and guardrail issues
- **Understand costs** - Control Tower pricing and cost optimization
- **Master automation** - Account provisioning and lifecycle management