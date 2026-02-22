# AWS Organizations - DOP-C02 Exam Notes

## 1. Overview

**AWS Organizations** is an account management service that enables you to centrally manage and govern your environment as you grow and scale your AWS resources. It provides centralized billing, access control, compliance, security, and resource sharing across your AWS accounts.

### Key Characteristics
- **Centralized management** - Manage multiple AWS accounts from a single location
- **Consolidated billing** - Single payment method and billing for all accounts
- **Hierarchical organization** - Organize accounts using Organizational Units (OUs)
- **Service Control Policies** - Centralized access control across accounts
- **Account creation** - Programmatically create and manage AWS accounts
- **Resource sharing** - Share resources across accounts in your organization
- **Compliance governance** - Enforce policies and compliance requirements

### What Problem Does It Solve?
- Simplifies management of multiple AWS accounts
- Provides centralized billing and cost management
- Enables consistent security and compliance policies
- Facilitates resource sharing and collaboration
- Reduces administrative overhead
- Supports organizational governance and control

---

## 2. Core Components

### Organization Structure
```
Root Organization
├── Master Account (Management Account)
├── Security OU
│   ├── Log Archive Account
│   ├── Audit Account
│   └── Security Tools Account
├── Production OU
│   ├── Prod App Account 1
│   ├── Prod App Account 2
│   └── Prod Database Account
├── Development OU
│   ├── Dev Account 1
│   ├── Dev Account 2
│   └── Sandbox Account
└── Shared Services OU
    ├── Network Account
    ├── DNS Account
    └── Monitoring Account
```

### Account Creation
```python
# Programmatic account creation
import boto3
import json
import time

def create_account(account_name, email, role_name="OrganizationAccountAccessRole"):
    """Create new AWS account in organization"""
    
    organizations = boto3.client('organizations')
    
    try:
        # Create account
        response = organizations.create_account(
            Email=email,
            AccountName=account_name,
            RoleName=role_name
        )
        
        request_id = response['CreateAccountStatus']['Id']
        
        # Wait for account creation to complete
        while True:
            status_response = organizations.describe_create_account_status(
                CreateAccountRequestId=request_id
            )
            
            status = status_response['CreateAccountStatus']['State']
            
            if status == 'SUCCEEDED':
                account_id = status_response['CreateAccountStatus']['AccountId']
                print(f"Account created successfully: {account_id}")
                return account_id
            
            elif status == 'FAILED':
                failure_reason = status_response['CreateAccountStatus']['FailureReason']
                raise Exception(f"Account creation failed: {failure_reason}")
            
            print(f"Account creation in progress: {status}")
            time.sleep(30)
    
    except Exception as e:
        print(f"Error creating account: {str(e)}")
        raise

# CloudFormation template for account creation
Resources:
  NewAccount:
    Type: AWS::Organizations::Account
    Properties:
      AccountName: !Ref AccountName
      Email: !Ref AccountEmail
      RoleName: OrganizationAccountAccessRole
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Department
          Value: !Ref Department
```

### Organizational Units (OUs)
```yaml
# CloudFormation template for OU structure
Resources:
  SecurityOU:
    Type: AWS::Organizations::OrganizationalUnit
    Properties:
      Name: Security
      ParentId: !Ref RootId
      Tags:
        - Key: Purpose
          Value: Security and Compliance

  ProductionOU:
    Type: AWS::Organizations::OrganizationalUnit
    Properties:
      Name: Production
      ParentId: !Ref RootId
      Tags:
        - Key: Purpose
          Value: Production Workloads

  DevelopmentOU:
    Type: AWS::Organizations::OrganizationalUnit
    Properties:
      Name: Development
      ParentId: !Ref RootId
      Tags:
        - Key: Purpose
          Value: Development and Testing

  # Move account to OU
  MoveAccountToOU:
    Type: AWS::Organizations::Account
    Properties:
      AccountId: !Ref TargetAccountId
      ParentId: !Ref ProductionOU
```

---

## 3. Service Control Policies (SCPs)

### Preventive Controls
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Principal": {
        "AWS": "*"
      },
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalType": "Root"
        },
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        }
      }
    },
    {
      "Sid": "DenyUnencryptedStorage",
      "Effect": "Deny",
      "Action": [
        "s3:PutObject",
        "ec2:CreateVolume",
        "rds:CreateDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "RequireMFAForSensitiveActions",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket",
        "iam:DeleteRole",
        "iam:DeleteUser"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    },
    {
      "Sid": "RestrictRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "sts:*",
        "cloudfront:*",
        "route53:*",
        "support:*",
        "trustedadvisor:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2",
            "eu-west-1"
          ]
        }
      }
    }
  ]
}
```

### Environment-Specific SCPs
```json
{
  "DevelopmentSCP": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyExpensiveInstances",
        "Effect": "Deny",
        "Action": [
          "ec2:RunInstances"
        ],
        "Resource": "arn:aws:ec2:*:*:instance/*",
        "Condition": {
          "ForAllValues:StringNotLike": {
            "ec2:InstanceType": [
              "t3.*",
              "t2.*"
            ]
          }
        }
      },
      {
        "Sid": "DenyProductionResources",
        "Effect": "Deny",
        "Action": [
          "rds:CreateDBInstance"
        ],
        "Resource": "*",
        "Condition": {
          "StringNotEquals": {
            "rds:db-instance-class": [
              "db.t3.micro",
              "db.t3.small"
            ]
          }
        }
      }
    ]
  },
  "ProductionSCP": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "RequireEncryption",
        "Effect": "Deny",
        "Action": [
          "s3:PutObject",
          "ec2:CreateVolume",
          "rds:CreateDBInstance"
        ],
        "Resource": "*",
        "Condition": {
          "Bool": {
            "aws:SecureTransport": "false"
          }
        }
      },
      {
        "Sid": "PreventConfigChanges",
        "Effect": "Deny",
        "Action": [
          "config:StopConfigurationRecorder",
          "config:DeleteConfigurationRecorder",
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail"
        ],
        "Resource": "*",
        "Condition": {
          "StringNotEquals": {
            "aws:PrincipalArn": [
              "arn:aws:iam::*:role/OrganizationAccountAccessRole"
            ]
          }
        }
      }
    ]
  }
}
```

### SCP Management
```python
# SCP management functions
import boto3
import json

class SCPManager:
    def __init__(self):
        self.organizations = boto3.client('organizations')
    
    def create_scp(self, name, description, policy_document):
        """Create a new Service Control Policy"""
        
        response = self.organizations.create_policy(
            Name=name,
            Description=description,
            Type='SERVICE_CONTROL_POLICY',
            Content=json.dumps(policy_document, indent=2)
        )
        
        return response['Policy']['PolicySummary']['Id']
    
    def attach_scp_to_ou(self, policy_id, ou_id):
        """Attach SCP to an Organizational Unit"""
        
        self.organizations.attach_policy(
            PolicyId=policy_id,
            TargetId=ou_id
        )
    
    def attach_scp_to_account(self, policy_id, account_id):
        """Attach SCP to a specific account"""
        
        self.organizations.attach_policy(
            PolicyId=policy_id,
            TargetId=account_id
        )
    
    def get_effective_policies(self, account_id):
        """Get all effective policies for an account"""
        
        policies = []
        
        # Get policies attached directly to account
        account_policies = self.organizations.list_policies_for_target(
            TargetId=account_id,
            Filter='SERVICE_CONTROL_POLICY'
        )
        policies.extend(account_policies['Policies'])
        
        # Get policies from parent OUs
        parents = self.organizations.list_parents(ChildId=account_id)
        
        for parent in parents['Parents']:
            if parent['Type'] == 'ORGANIZATIONAL_UNIT':
                ou_policies = self.get_ou_policies(parent['Id'])
                policies.extend(ou_policies)
        
        return policies
    
    def get_ou_policies(self, ou_id):
        """Get policies for an OU and its parents"""
        
        policies = []
        
        # Get policies attached to this OU
        ou_policies = self.organizations.list_policies_for_target(
            TargetId=ou_id,
            Filter='SERVICE_CONTROL_POLICY'
        )
        policies.extend(ou_policies['Policies'])
        
        # Get policies from parent OUs
        parents = self.organizations.list_parents(ChildId=ou_id)
        
        for parent in parents['Parents']:
            if parent['Type'] == 'ORGANIZATIONAL_UNIT':
                parent_policies = self.get_ou_policies(parent['Id'])
                policies.extend(parent_policies)
        
        return policies
    
    def simulate_policy_effect(self, account_id, action, resource):
        """Simulate the effect of SCPs on a specific action"""
        
        # This is a simplified simulation
        # In practice, you'd need to evaluate all effective policies
        
        effective_policies = self.get_effective_policies(account_id)
        
        for policy in effective_policies:
            policy_content = json.loads(policy['Content'])
            
            for statement in policy_content.get('Statement', []):
                if statement.get('Effect') == 'Deny':
                    # Check if action matches
                    actions = statement.get('Action', [])
                    if isinstance(actions, str):
                        actions = [actions]
                    
                    for policy_action in actions:
                        if self.action_matches(action, policy_action):
                            return {
                                'allowed': False,
                                'reason': f'Denied by SCP: {policy["Name"]}',
                                'policy_id': policy['Id']
                            }
        
        return {
            'allowed': True,
            'reason': 'No explicit deny found in SCPs'
        }
    
    def action_matches(self, action, policy_action):
        """Check if an action matches a policy action pattern"""
        
        if policy_action == '*':
            return True
        
        if '*' in policy_action:
            # Simple wildcard matching
            pattern = policy_action.replace('*', '.*')
            import re
            return bool(re.match(pattern, action))
        
        return action == policy_action
```

---

## 4. Consolidated Billing and Cost Management

### Cost Allocation and Tagging
```python
# Cost management and allocation
import boto3
from datetime import datetime, timedelta

class OrganizationCostManager:
    def __init__(self):
        self.ce = boto3.client('ce')  # Cost Explorer
        self.organizations = boto3.client('organizations')
        self.budgets = boto3.client('budgets')
    
    def get_organization_costs(self, start_date, end_date):
        """Get cost breakdown by account and service"""
        
        response = self.ce.get_cost_and_usage(
            TimePeriod={
                'Start': start_date.strftime('%Y-%m-%d'),
                'End': end_date.strftime('%Y-%m-%d')
            },
            Granularity='MONTHLY',
            Metrics=['BlendedCost', 'UnblendedCost'],
            GroupBy=[
                {'Type': 'DIMENSION', 'Key': 'LINKED_ACCOUNT'},
                {'Type': 'DIMENSION', 'Key': 'SERVICE'}
            ]
        )
        
        return response['ResultsByTime']
    
    def create_account_budget(self, account_id, budget_amount, notification_email):
        """Create budget for a specific account"""
        
        budget_name = f"Account-{account_id}-Monthly-Budget"
        
        budget = {
            'BudgetName': budget_name,
            'BudgetLimit': {
                'Amount': str(budget_amount),
                'Unit': 'USD'
            },
            'TimeUnit': 'MONTHLY',
            'BudgetType': 'COST',
            'CostFilters': {
                'LinkedAccount': [account_id]
            }
        }
        
        notifications = [
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
                        'Address': notification_email
                    }
                ]
            },
            {
                'Notification': {
                    'NotificationType': 'FORECASTED',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 100.0,
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': notification_email
                    }
                ]
            }
        ]
        
        # Create budget in master account
        master_account_id = self.get_master_account_id()
        
        self.budgets.create_budget(
            AccountId=master_account_id,
            Budget=budget,
            NotificationsWithSubscribers=notifications
        )
        
        return budget_name
    
    def get_cost_by_ou(self, ou_id, start_date, end_date):
        """Get aggregated costs for all accounts in an OU"""
        
        # Get all accounts in the OU
        accounts = self.get_accounts_in_ou(ou_id)
        account_ids = [acc['Id'] for acc in accounts]
        
        if not account_ids:
            return {'TotalCost': 0, 'AccountCosts': []}
        
        response = self.ce.get_cost_and_usage(
            TimePeriod={
                'Start': start_date.strftime('%Y-%m-%d'),
                'End': end_date.strftime('%Y-%m-%d')
            },
            Granularity='MONTHLY',
            Metrics=['BlendedCost'],
            GroupBy=[
                {'Type': 'DIMENSION', 'Key': 'LINKED_ACCOUNT'}
            ],
            Filter={
                'Dimensions': {
                    'Key': 'LINKED_ACCOUNT',
                    'Values': account_ids
                }
            }
        )
        
        total_cost = 0
        account_costs = []
        
        for result in response['ResultsByTime']:
            for group in result['Groups']:
                account_id = group['Keys'][0]
                cost = float(group['Metrics']['BlendedCost']['Amount'])
                total_cost += cost
                
                account_costs.append({
                    'AccountId': account_id,
                    'Cost': cost
                })
        
        return {
            'TotalCost': total_cost,
            'AccountCosts': account_costs
        }
    
    def get_accounts_in_ou(self, ou_id):
        """Get all accounts in an organizational unit"""
        
        accounts = []
        
        # Get direct child accounts
        children = self.organizations.list_children(
            ParentId=ou_id,
            ChildType='ACCOUNT'
        )
        
        for child in children['Children']:
            account = self.organizations.describe_account(AccountId=child['Id'])
            accounts.append(account['Account'])
        
        # Get accounts from child OUs
        child_ous = self.organizations.list_children(
            ParentId=ou_id,
            ChildType='ORGANIZATIONAL_UNIT'
        )
        
        for child_ou in child_ous['Children']:
            child_accounts = self.get_accounts_in_ou(child_ou['Id'])
            accounts.extend(child_accounts)
        
        return accounts
    
    def generate_cost_report(self):
        """Generate comprehensive cost report for the organization"""
        
        end_date = datetime.utcnow()
        start_date = end_date - timedelta(days=30)
        
        # Get organization structure
        root_id = self.organizations.list_roots()['Roots'][0]['Id']
        ous = self.get_all_ous(root_id)
        
        report = {
            'ReportDate': end_date.isoformat(),
            'Period': f"{start_date.strftime('%Y-%m-%d')} to {end_date.strftime('%Y-%m-%d')}",
            'OrganizationCosts': []
        }
        
        total_org_cost = 0
        
        for ou in ous:
            ou_costs = self.get_cost_by_ou(ou['Id'], start_date, end_date)
            total_org_cost += ou_costs['TotalCost']
            
            report['OrganizationCosts'].append({
                'OUName': ou['Name'],
                'OUId': ou['Id'],
                'TotalCost': ou_costs['TotalCost'],
                'AccountCosts': ou_costs['AccountCosts']
            })
        
        report['TotalOrganizationCost'] = total_org_cost
        
        return report
```

### Reserved Instance Management
```yaml
# CloudFormation template for RI management
Resources:
  RIRecommendationFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: ri-recommendation-analyzer
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt RIAnalysisRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          from datetime import datetime, timedelta
          
          def lambda_handler(event, context):
              """Analyze RI recommendations across organization"""
              
              ce = boto3.client('ce')
              organizations = boto3.client('organizations')
              
              # Get all accounts
              accounts = organizations.list_accounts()
              
              recommendations = []
              
              for account in accounts['Accounts']:
                  if account['Status'] != 'ACTIVE':
                      continue
                  
                  # Get RI recommendations for account
                  response = ce.get_reservation_purchase_recommendation(
                      Service='Amazon Elastic Compute Cloud - Compute',
                      AccountId=account['Id'],
                      LookbackPeriodInDays='SIXTY_DAYS',
                      TermInYears='ONE_YEAR',
                      PaymentOption='PARTIAL_UPFRONT'
                  )
                  
                  if response['Recommendations']:
                      for rec in response['Recommendations']:
                          recommendations.append({
                              'AccountId': account['Id'],
                              'AccountName': account['Name'],
                              'InstanceType': rec['RecommendationDetails']['InstanceDetails']['EC2InstanceDetails']['InstanceType'],
                              'EstimatedMonthlySavings': rec['RecommendationDetails']['EstimatedMonthlySavingsAmount'],
                              'UpfrontCost': rec['RecommendationDetails']['UpfrontCost'],
                              'RecommendedQuantity': rec['RecommendationDetails']['RecommendedNumberOfInstancesToPurchase']
                          })
              
              # Send recommendations to SNS
              sns = boto3.client('sns')
              sns.publish(
                  TopicArn='arn:aws:sns:us-east-1:123456789012:ri-recommendations',
                  Message=json.dumps(recommendations, indent=2),
                  Subject='Reserved Instance Recommendations'
              )
              
              return {
                  'statusCode': 200,
                  'body': json.dumps(f'Found {len(recommendations)} RI recommendations')
              }

  RIRecommendationSchedule:
    Type: AWS::Events::Rule
    Properties:
      Description: Weekly RI recommendation analysis
      ScheduleExpression: rate(7 days)
      State: ENABLED
      Targets:
        - Arn: !GetAtt RIRecommendationFunction.Arn
          Id: RIRecommendationTarget

  RIRecommendationPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref RIRecommendationFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt RIRecommendationSchedule.Arn
```

---

## 5. Resource Sharing

### AWS Resource Access Manager (RAM)
```yaml
# Resource sharing with RAM
Resources:
  SharedVPCResourceShare:
    Type: AWS::RAM::ResourceShare
    Properties:
      Name: SharedVPCResourceShare
      ResourceArns:
        - !Sub "arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PrivateSubnet1}"
        - !Sub "arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PrivateSubnet2}"
      Principals:
        - !Ref TargetAccountId1
        - !Ref TargetAccountId2
      AllowExternalPrincipals: false
      Tags:
        - Key: Purpose
          Value: SharedNetworking

  SharedTransitGateway:
    Type: AWS::RAM::ResourceShare
    Properties:
      Name: SharedTransitGatewayShare
      ResourceArns:
        - !Sub "arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:transit-gateway/${TransitGateway}"
      Principals:
        - !Ref ProductionOUId
      AllowExternalPrincipals: false
      Tags:
        - Key: Purpose
          Value: NetworkConnectivity
```

### Cross-Account Resource Access
```python
# Cross-account resource management
import boto3
import json

class CrossAccountResourceManager:
    def __init__(self):
        self.ram = boto3.client('ram')
        self.organizations = boto3.client('organizations')
        self.sts = boto3.client('sts')
    
    def share_resources_with_ou(self, resource_arns, ou_id, share_name):
        """Share resources with all accounts in an OU"""
        
        # Get all accounts in the OU
        accounts = self.get_accounts_in_ou(ou_id)
        account_ids = [acc['Id'] for acc in accounts]
        
        # Create resource share
        response = self.ram.create_resource_share(
            name=share_name,
            resourceArns=resource_arns,
            principals=account_ids,
            allowExternalPrincipals=False,
            tags=[
                {'key': 'CreatedBy', 'value': 'CrossAccountResourceManager'},
                {'key': 'TargetOU', 'value': ou_id}
            ]
        )
        
        return response['resourceShare']['arn']
    
    def accept_resource_share_invitation(self, invitation_arn):
        """Accept a resource share invitation"""
        
        self.ram.accept_resource_share_invitation(
            resourceShareInvitationArn=invitation_arn
        )
    
    def get_shared_resources(self, account_id):
        """Get resources shared with a specific account"""
        
        # Get resource shares where account is a principal
        shares = self.ram.get_resource_shares(
            resourceOwner='OTHER-ACCOUNTS',
            resourceShareStatus='ACTIVE'
        )
        
        shared_resources = []
        
        for share in shares['resourceShares']:
            # Get resources in this share
            resources = self.ram.get_resource_share_associations(
                resourceShareArns=[share['arn']],
                associationType='RESOURCE'
            )
            
            for resource in resources['resourceShareAssociations']:
                shared_resources.append({
                    'ResourceArn': resource['associatedEntity'],
                    'ShareName': share['name'],
                    'ShareArn': share['arn'],
                    'Status': resource['status']
                })
        
        return shared_resources
    
    def create_cross_account_role(self, target_account_id, role_name, policy_document):
        """Create a role that can be assumed by another account"""
        
        # Assume role in target account
        assumed_role = self.sts.assume_role(
            RoleArn=f"arn:aws:iam::{target_account_id}:role/OrganizationAccountAccessRole",
            RoleSessionName='CrossAccountRoleCreation'
        )
        
        # Create session with assumed role
        session = boto3.Session(
            aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
            aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
            aws_session_token=assumed_role['Credentials']['SessionToken']
        )
        
        iam = session.client('iam')
        
        # Create trust policy
        trust_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": f"arn:aws:iam::{self.get_master_account_id()}:root"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "sts:ExternalId": "unique-external-id"
                        }
                    }
                }
            ]
        }
        
        # Create role
        iam.create_role(
            RoleName=role_name,
            AssumeRolePolicyDocument=json.dumps(trust_policy),
            Description=f'Cross-account role for {role_name}',
            Tags=[
                {'Key': 'CreatedBy', 'Value': 'CrossAccountResourceManager'},
                {'Key': 'Purpose', 'Value': 'CrossAccountAccess'}
            ]
        )
        
        # Attach policy
        iam.put_role_policy(
            RoleName=role_name,
            PolicyName=f'{role_name}Policy',
            PolicyDocument=json.dumps(policy_document)
        )
        
        return f"arn:aws:iam::{target_account_id}:role/{role_name}"
```

---

## 6. Integration with Other Services

### Organizations with Control Tower
```python
# Integration with Control Tower
import boto3
import json

class OrganizationsControlTowerIntegration:
    def __init__(self):
        self.organizations = boto3.client('organizations')
        self.controltower = boto3.client('controltower')
        self.servicecatalog = boto3.client('servicecatalog')
    
    def setup_control_tower_integration(self):
        """Setup Control Tower with existing Organizations"""
        
        # Ensure organization has all features enabled
        org = self.organizations.describe_organization()
        
        if org['Organization']['FeatureSet'] != 'ALL':
            self.organizations.enable_all_features()
        
        # Create required OUs for Control Tower
        root_id = self.organizations.list_roots()['Roots'][0]['Id']
        
        # Security OU
        security_ou = self.organizations.create_organizational_unit(
            ParentId=root_id,
            Name='Security'
        )
        
        # Sandbox OU
        sandbox_ou = self.organizations.create_organizational_unit(
            ParentId=root_id,
            Name='Sandbox'
        )
        
        return {
            'SecurityOU': security_ou['OrganizationalUnit']['Id'],
            'SandboxOU': sandbox_ou['OrganizationalUnit']['Id']
        }
    
    def create_account_with_control_tower(self, account_details):
        """Create account using Control Tower Account Factory"""
        
        # Find Account Factory product
        products = self.servicecatalog.search_products(
            Filters={'FullTextSearch': ['AWS Control Tower Account Factory']}
        )
        
        if not products['ProductViewSummaries']:
            raise ValueError("Control Tower Account Factory not found")
        
        product = products['ProductViewSummaries'][0]
        
        # Provision account
        response = self.servicecatalog.provision_product(
            ProductId=product['ProductId'],
            ProvisioningArtifactId='pa-latest',
            ProvisionedProductName=f"Account-{account_details['AccountName']}",
            ProvisioningParameters=[
                {'Key': 'AccountName', 'Value': account_details['AccountName']},
                {'Key': 'AccountEmail', 'Value': account_details['AccountEmail']},
                {'Key': 'OrganizationalUnitName', 'Value': account_details['OrganizationalUnit']},
                {'Key': 'SSOUserEmail', 'Value': account_details['SSOUserEmail']},
                {'Key': 'SSOUserFirstName', 'Value': account_details['SSOUserFirstName']},
                {'Key': 'SSOUserLastName', 'Value': account_details['SSOUserLastName']}
            ]
        )
        
        return response['RecordDetail']
```

### Organizations with Config
```yaml
# Organization-wide Config setup
Resources:
  OrganizationConfigurationAggregator:
    Type: AWS::Config::ConfigurationAggregator
    Properties:
      ConfigurationAggregatorName: OrganizationAggregator
      OrganizationAggregationSource:
        RoleArn: !GetAtt ConfigAggregatorRole.Arn
        AllAwsRegions: true

  OrganizationConfigRule:
    Type: AWS::Config::OrganizationConfigRule
    Properties:
      OrganizationConfigRuleName: s3-bucket-public-read-prohibited
      OrganizationManagedRuleMetadata:
        RuleIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED
        Description: Checks that S3 buckets do not allow public read access
      ExcludedAccounts:
        - !Ref LogArchiveAccountId  # Exclude log archive account

  OrganizationRemediationConfiguration:
    Type: AWS::Config::OrganizationRemediationConfiguration
    Properties:
      OrganizationConfigRuleName: !Ref OrganizationConfigRule
      TargetType: SSM_DOCUMENT
      TargetId: AWSConfigRemediation-RemoveS3BucketPublicReadAccess
      TargetVersion: "1"
      Parameters:
        AutomationAssumeRole:
          StaticValue: !Sub "arn:aws:iam::{account}:role/ConfigRemediationRole"
        BucketName:
          ResourceValue: RESOURCE_ID
      Automatic: false
      MaximumAutomaticAttempts: 1
```

---

## 7. Common Exam Scenarios

### Scenario 1: Multi-Account Security Setup
```python
# Complete multi-account security setup
import boto3
import json
from datetime import datetime

class MultiAccountSecuritySetup:
    def __init__(self):
        self.organizations = boto3.client('organizations')
        self.iam = boto3.client('iam')
        self.cloudtrail = boto3.client('cloudtrail')
        self.config = boto3.client('config')
    
    def setup_organization_security(self):
        """Setup comprehensive security across organization"""
        
        # 1. Create security-focused OU structure
        security_structure = self.create_security_ou_structure()
        
        # 2. Apply security SCPs
        security_policies = self.create_security_scps()
        
        # 3. Setup centralized logging
        logging_setup = self.setup_centralized_logging()
        
        # 4. Configure organization-wide Config
        config_setup = self.setup_organization_config()
        
        # 5. Create cross-account security roles
        security_roles = self.create_security_roles()
        
        return {
            'SecurityStructure': security_structure,
            'SecurityPolicies': security_policies,
            'LoggingSetup': logging_setup,
            'ConfigSetup': config_setup,
            'SecurityRoles': security_roles
        }
    
    def create_security_ou_structure(self):
        """Create security-focused OU structure"""
        
        root_id = self.organizations.list_roots()['Roots'][0]['Id']
        
        # Security OU for security-related accounts
        security_ou = self.organizations.create_organizational_unit(
            ParentId=root_id,
            Name='Security'
        )
        
        # Production OU with strict controls
        production_ou = self.organizations.create_organizational_unit(
            ParentId=root_id,
            Name='Production'
        )
        
        # Development OU with relaxed controls
        development_ou = self.organizations.create_organizational_unit(
            ParentId=root_id,
            Name='Development'
        )
        
        # Sandbox OU for experimentation
        sandbox_ou = self.organizations.create_organizational_unit(
            ParentId=root_id,
            Name='Sandbox'
        )
        
        return {
            'SecurityOU': security_ou['OrganizationalUnit']['Id'],
            'ProductionOU': production_ou['OrganizationalUnit']['Id'],
            'DevelopmentOU': development_ou['OrganizationalUnit']['Id'],
            'SandboxOU': sandbox_ou['OrganizationalUnit']['Id']
        }
    
    def create_security_scps(self):
        """Create and apply security-focused SCPs"""
        
        # Base security policy for all accounts
        base_security_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "DenyRootUserActions",
                    "Effect": "Deny",
                    "Principal": {"AWS": "*"},
                    "Action": "*",
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {"aws:PrincipalType": "Root"}
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
                        "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
                    }
                },
                {
                    "Sid": "PreventSecurityServiceDisabling",
                    "Effect": "Deny",
                    "Action": [
                        "config:StopConfigurationRecorder",
                        "cloudtrail:StopLogging",
                        "guardduty:DeleteDetector"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringNotEquals": {
                            "aws:PrincipalArn": [
                                "arn:aws:iam::*:role/OrganizationAccountAccessRole"
                            ]
                        }
                    }
                }
            ]
        }
        
        # Production-specific policy
        production_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "RequireEncryption",
                    "Effect": "Deny",
                    "Action": [
                        "s3:PutObject",
                        "ec2:CreateVolume",
                        "rds:CreateDBInstance"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "Bool": {"aws:SecureTransport": "false"}
                    }
                },
                {
                    "Sid": "RestrictInstanceTypes",
                    "Effect": "Deny",
                    "Action": "ec2:RunInstances",
                    "Resource": "arn:aws:ec2:*:*:instance/*",
                    "Condition": {
                        "ForAllValues:StringNotLike": {
                            "ec2:InstanceType": [
                                "t3.*", "m5.*", "c5.*", "r5.*"
                            ]
                        }
                    }
                }
            ]
        }
        
        # Create policies
        base_policy_id = self.organizations.create_policy(
            Name='BaseSecurityPolicy',
            Description='Base security policy for all accounts',
            Type='SERVICE_CONTROL_POLICY',
            Content=json.dumps(base_security_policy)
        )['Policy']['PolicySummary']['Id']
        
        production_policy_id = self.organizations.create_policy(
            Name='ProductionSecurityPolicy',
            Description='Additional security policy for production accounts',
            Type='SERVICE_CONTROL_POLICY',
            Content=json.dumps(production_policy)
        )['Policy']['PolicySummary']['Id']
        
        return {
            'BasePolicyId': base_policy_id,
            'ProductionPolicyId': production_policy_id
        }
    
    def setup_centralized_logging(self):
        """Setup centralized CloudTrail logging"""
        
        # Create organization trail
        trail_response = self.cloudtrail.create_trail(
            Name='OrganizationTrail',
            S3BucketName='organization-cloudtrail-logs',
            IncludeGlobalServiceEvents=True,
            IsMultiRegionTrail=True,
            EnableLogFileValidation=True,
            IsOrganizationTrail=True,
            EventSelectors=[
                {
                    'ReadWriteType': 'All',
                    'IncludeManagementEvents': True,
                    'DataResources': [
                        {
                            'Type': 'AWS::S3::Object',
                            'Values': ['arn:aws:s3:::*/*']
                        }
                    ]
                }
            ]
        )
        
        # Start logging
        self.cloudtrail.start_logging(Name='OrganizationTrail')
        
        return {
            'TrailArn': trail_response['TrailARN']
        }
    
    def setup_organization_config(self):
        """Setup organization-wide Config"""
        
        # Create configuration aggregator
        aggregator_response = self.config.put_configuration_aggregator(
            ConfigurationAggregatorName='OrganizationAggregator',
            OrganizationAggregationSource={
                'RoleArn': 'arn:aws:iam::123456789012:role/ConfigAggregatorRole',
                'AllAwsRegions': True
            }
        )
        
        # Create organization Config rules
        config_rules = [
            {
                'name': 's3-bucket-public-read-prohibited',
                'identifier': 'S3_BUCKET_PUBLIC_READ_PROHIBITED'
            },
            {
                'name': 'cloudtrail-enabled',
                'identifier': 'CLOUD_TRAIL_ENABLED'
            },
            {
                'name': 'root-mfa-enabled',
                'identifier': 'ROOT_MFA_ENABLED'
            }
        ]
        
        created_rules = []
        
        for rule in config_rules:
            rule_response = self.config.put_organization_config_rule(
                OrganizationConfigRuleName=rule['name'],
                OrganizationManagedRuleMetadata={
                    'RuleIdentifier': rule['identifier'],
                    'Description': f'Organization-wide {rule["name"]} rule'
                }
            )
            created_rules.append(rule_response['OrganizationConfigRuleArn'])
        
        return {
            'AggregatorArn': aggregator_response['ConfigurationAggregator']['ConfigurationAggregatorArn'],
            'ConfigRules': created_rules
        }
```

---

## 8. Exam Tips

- **Understand organization structure** - Root, OUs, accounts hierarchy
- **Master SCPs** - Preventive controls and policy inheritance
- **Know consolidated billing** - Cost allocation, budgets, RI management
- **Practice account management** - Creation, movement, lifecycle
- **Learn resource sharing** - RAM integration and cross-account access
- **Understand integration** - Control Tower, Config, CloudTrail integration
- **Know limitations** - What Organizations can and cannot control
- **Practice troubleshooting** - Common SCP and account issues
- **Understand governance** - Compliance, security, and operational governance
- **Master automation** - Programmatic account and policy management