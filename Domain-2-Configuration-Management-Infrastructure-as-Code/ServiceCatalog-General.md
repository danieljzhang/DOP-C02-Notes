# AWS Service Catalog - DOP-C02 Exam Notes

## 1. Overview

**AWS Service Catalog** allows organizations to create and manage catalogs of IT services that are approved for use on AWS. It enables centralized management of commonly deployed IT services and helps achieve consistent governance and compliance requirements.

### Key Characteristics
- **Self-service provisioning** - End users can launch pre-approved products
- **Centralized governance** - IT administrators control what can be deployed
- **Standardized templates** - CloudFormation templates as products
- **Portfolio management** - Organize products into portfolios
- **Access control** - IAM-based permissions for portfolios and products
- **Versioning** - Multiple versions of products with upgrade paths
- **Constraints** - Launch, template, and notification constraints
- **Cost control** - Budget and resource constraints

### What Problem Does It Solve?
- Enables self-service IT provisioning while maintaining governance
- Standardizes infrastructure deployments across the organization
- Reduces shadow IT by providing approved alternatives
- Ensures compliance with organizational policies and standards
- Facilitates cost control and resource management
- Accelerates time-to-market for development teams

---

## 2. Core Components

### Products
```yaml
# CloudFormation template for a Service Catalog product
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Standard Web Server Product'

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 instance type
  
  KeyPairName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: EC2 Key Pair for SSH access
  
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Environment name

Mappings:
  EnvironmentMap:
    dev:
      VpcId: vpc-12345678
      SubnetId: subnet-12345678
    staging:
      VpcId: vpc-87654321
      SubnetId: subnet-87654321
    prod:
      VpcId: vpc-11111111
      SubnetId: subnet-11111111

Resources:
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web server
      VpcId: !FindInMap [EnvironmentMap, !Ref Environment, VpcId]
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-web-server-sg"

  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890  # Amazon Linux 2
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyPairName
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      SubnetId: !FindInMap [EnvironmentMap, !Ref Environment, SubnetId]
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Web Server - ${Environment}</h1>" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-web-server"
        - Key: Environment
          Value: !Ref Environment

Outputs:
  InstanceId:
    Description: EC2 Instance ID
    Value: !Ref WebServerInstance
  
  PublicIP:
    Description: Public IP address
    Value: !GetAtt WebServerInstance.PublicIp
  
  WebsiteURL:
    Description: Website URL
    Value: !Sub "http://${WebServerInstance.PublicIp}"
```

### Portfolio Management
```yaml
# CloudFormation template for Service Catalog setup
Resources:
  # Create portfolio
  WebServicesPortfolio:
    Type: AWS::ServiceCatalog::Portfolio
    Properties:
      DisplayName: Web Services Portfolio
      Description: Standardized web server and database products
      ProviderName: IT Operations Team
      AcceptLanguage: en

  # Create product
  WebServerProduct:
    Type: AWS::ServiceCatalog::CloudFormationProduct
    Properties:
      Name: Standard Web Server
      Description: Pre-configured web server with security groups
      Owner: IT Operations
      AcceptLanguage: en
      ProvisioningArtifactParameters:
        - Name: v1.0
          Description: Initial version
          Info:
            LoadTemplateFromURL: !Sub "https://${TemplateBucket}.s3.amazonaws.com/web-server-template.yaml"

  # Associate product with portfolio
  ProductPortfolioAssociation:
    Type: AWS::ServiceCatalog::PortfolioProductAssociation
    Properties:
      PortfolioId: !Ref WebServicesPortfolio
      ProductId: !Ref WebServerProduct

  # Grant access to portfolio
  PortfolioPrincipalAssociation:
    Type: AWS::ServiceCatalog::PortfolioPrincipalAssociation
    Properties:
      PortfolioId: !Ref WebServicesPortfolio
      PrincipalARN: !Sub "arn:aws:iam::${AWS::AccountId}:role/DeveloperRole"
      PrincipalType: IAM

  # Add launch constraint
  LaunchConstraint:
    Type: AWS::ServiceCatalog::LaunchRoleConstraint
    Properties:
      PortfolioId: !Ref WebServicesPortfolio
      ProductId: !Ref WebServerProduct
      RoleArn: !GetAtt ServiceCatalogLaunchRole.Arn
      AcceptLanguage: en

  # Service Catalog launch role
  ServiceCatalogLaunchRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ServiceCatalogLaunchRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: servicecatalog.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEC2FullAccess
        - arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
      Policies:
        - PolicyName: ServiceCatalogLaunchPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - cloudformation:CreateStack
                  - cloudformation:UpdateStack
                  - cloudformation:DeleteStack
                  - cloudformation:DescribeStacks
                  - cloudformation:DescribeStackEvents
                  - cloudformation:DescribeStackResources
                  - cloudformation:GetTemplate
                Resource: '*'
```

---

## 3. Constraints

### Launch Constraints
```yaml
# Launch role constraint
Resources:
  LaunchRoleConstraint:
    Type: AWS::ServiceCatalog::LaunchRoleConstraint
    Properties:
      PortfolioId: !Ref Portfolio
      ProductId: !Ref Product
      RoleArn: !GetAtt LaunchRole.Arn
      Description: Role for launching EC2 instances

  LaunchRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: servicecatalog.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: LaunchPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ec2:*
                  - cloudformation:*
                Resource: '*'
              - Effect: Deny
                Action:
                  - ec2:TerminateInstances
                Resource: '*'
                Condition:
                  StringNotEquals:
                    'ec2:ResourceTag/Environment': ['dev', 'test']
```

### Template Constraints
```yaml
# Template constraint to restrict parameter values
Resources:
  TemplateConstraint:
    Type: AWS::ServiceCatalog::LaunchTemplateConstraint
    Properties:
      PortfolioId: !Ref Portfolio
      ProductId: !Ref Product
      Rules: |
        {
          "Rules": {
            "InstanceTypeRule": {
              "Assertions": [
                {
                  "Assert": {
                    "Fn::Contains": [
                      ["t3.micro", "t3.small", "t3.medium"],
                      {"Ref": "InstanceType"}
                    ]
                  },
                  "AssertDescription": "Instance type must be t3.micro, t3.small, or t3.medium"
                }
              ]
            },
            "EnvironmentTagRule": {
              "Assertions": [
                {
                  "Assert": {
                    "Fn::Not": [
                      {"Fn::Equals": [{"Ref": "Environment"}, "prod"]}
                    ]
                  },
                  "AssertDescription": "Production environment not allowed for this product"
                }
              ]
            }
          }
        }
```

### Notification Constraints
```yaml
# Notification constraint for provisioning events
Resources:
  NotificationConstraint:
    Type: AWS::ServiceCatalog::LaunchNotificationConstraint
    Properties:
      PortfolioId: !Ref Portfolio
      ProductId: !Ref Product
      NotificationArns:
        - !Ref ProvisioningNotificationTopic
      Description: Notify on provisioning events

  ProvisioningNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: service-catalog-provisioning
      DisplayName: Service Catalog Provisioning Notifications
      Subscription:
        - Protocol: email
          Endpoint: admin@company.com
```

---

## 4. Advanced Features

### Multi-Account Sharing
```yaml
# Share portfolio across accounts
Resources:
  PortfolioShare:
    Type: AWS::ServiceCatalog::PortfolioShare
    Properties:
      PortfolioId: !Ref Portfolio
      AccountId: "123456789012"  # Target account ID
      ShareTagOptions: true

  # Accept shared portfolio (in target account)
  AcceptedPortfolioShare:
    Type: AWS::ServiceCatalog::AcceptedPortfolioShare
    Properties:
      PortfolioId: port-abcdefghijklm  # Shared portfolio ID
      AcceptLanguage: en
```

### TagOptions
```yaml
# Create tag options for consistent tagging
Resources:
  EnvironmentTagOption:
    Type: AWS::ServiceCatalog::TagOption
    Properties:
      Key: Environment
      Value: dev
      Active: true

  ProjectTagOption:
    Type: AWS::ServiceCatalog::TagOption
    Properties:
      Key: Project
      Value: WebApp
      Active: true

  # Associate tag options with portfolio
  PortfolioTagOptionAssociation:
    Type: AWS::ServiceCatalog::TagOptionAssociation
    Properties:
      ResourceId: !Ref Portfolio
      TagOptionId: !Ref EnvironmentTagOption

  # Associate tag options with product
  ProductTagOptionAssociation:
    Type: AWS::ServiceCatalog::TagOptionAssociation
    Properties:
      ResourceId: !Ref Product
      TagOptionId: !Ref ProjectTagOption
```

### Stack Set Constraints
```yaml
# Stack set constraint for multi-region deployment
Resources:
  StackSetConstraint:
    Type: AWS::ServiceCatalog::StackSetConstraint
    Properties:
      PortfolioId: !Ref Portfolio
      ProductId: !Ref Product
      Description: Deploy across multiple regions
      AccountList:
        - "123456789012"
        - "210987654321"
      RegionList:
        - us-east-1
        - us-west-2
        - eu-west-1
      AdminRole: !GetAtt StackSetAdminRole.Arn
      ExecutionRole: StackSetExecutionRole
```

---

## 5. Service Catalog with CI/CD

### Automated Product Updates
```python
# Lambda function to update Service Catalog products
import boto3
import json

def lambda_handler(event, context):
    """
    Update Service Catalog product when CloudFormation template changes
    """
    
    servicecatalog = boto3.client('servicecatalog')
    s3 = boto3.client('s3')
    
    # Get S3 event details
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Extract product information from S3 key
    # Expected format: products/{product-name}/template.yaml
    if key.startswith('products/') and key.endswith('/template.yaml'):
        product_name = key.split('/')[1]
        
        try:
            # Find the product
            products = servicecatalog.search_products_as_admin(
                Filters={
                    'FullTextSearch': [product_name]
                }
            )
            
            if products['ProductViewDetails']:
                product_id = products['ProductViewDetails'][0]['ProductViewSummary']['ProductId']
                
                # Create new provisioning artifact (version)
                template_url = f"https://{bucket}.s3.amazonaws.com/{key}"
                
                response = servicecatalog.create_provisioning_artifact(
                    ProductId=product_id,
                    Parameters={
                        'Name': f"v{int(time.time())}",  # Version based on timestamp
                        'Description': f"Auto-updated from {key}",
                        'Info': {
                            'LoadTemplateFromURL': template_url
                        },
                        'Type': 'CLOUD_FORMATION_TEMPLATE'
                    }
                )
                
                print(f"Created new version for product {product_name}: {response['ProvisioningArtifactDetail']['Id']}")
                
        except Exception as e:
            print(f"Error updating product {product_name}: {str(e)}")
            raise
    
    return {
        'statusCode': 200,
        'body': json.dumps('Product update completed')
    }
```

### CodePipeline Integration
```yaml
# CodePipeline for Service Catalog product deployment
Resources:
  ServiceCatalogPipeline:
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
                Provider: CodeCommit
                Version: '1'
              Configuration:
                RepositoryName: service-catalog-products
                BranchName: main
              OutputArtifacts:
                - Name: SourceOutput

        - Name: Validate
          Actions:
            - Name: ValidateTemplates
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref ValidationProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: ValidatedOutput

        - Name: Deploy-Dev
          Actions:
            - Name: UpdateProducts
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: Lambda
                Version: '1'
              Configuration:
                FunctionName: !Ref ProductUpdateFunction
              InputArtifacts:
                - Name: ValidatedOutput

        - Name: Test
          Actions:
            - Name: TestProvisioning
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref TestProject
              InputArtifacts:
                - Name: ValidatedOutput

        - Name: Deploy-Prod
          Actions:
            - Name: ApprovalAction
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
            
            - Name: UpdateProdProducts
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: Lambda
                Version: '1'
              Configuration:
                FunctionName: !Ref ProductUpdateFunction
                UserParameters: '{"environment": "prod"}'
              InputArtifacts:
                - Name: ValidatedOutput
              RunOrder: 2
```

---

## 6. Governance and Compliance

### Budget Constraints
```python
# Lambda function for budget constraint enforcement
import boto3
import json

def lambda_handler(event, context):
    """
    Enforce budget constraints for Service Catalog provisioning
    """
    
    # Parse the provisioning request
    request_detail = event['detail']
    product_id = request_detail['productId']
    user_arn = request_detail['userArn']
    
    # Get user's department from IAM tags or external system
    department = get_user_department(user_arn)
    
    # Check department budget
    budgets_client = boto3.client('budgets')
    
    try:
        budget_response = budgets_client.describe_budget(
            AccountId=boto3.Session().get_credentials().access_key[:12],
            BudgetName=f"{department}-monthly-budget"
        )
        
        budget = budget_response['Budget']
        current_spend = float(budget['CalculatedSpend']['ActualSpend']['Amount'])
        budget_limit = float(budget['BudgetLimit']['Amount'])
        
        # Calculate estimated cost for the product
        estimated_cost = get_product_estimated_cost(product_id)
        
        if (current_spend + estimated_cost) > budget_limit:
            # Reject provisioning
            return {
                'status': 'REJECTED',
                'reason': f'Provisioning would exceed department budget. Current: ${current_spend:.2f}, Estimated: ${estimated_cost:.2f}, Limit: ${budget_limit:.2f}'
            }
        
        return {
            'status': 'APPROVED',
            'estimated_cost': estimated_cost
        }
        
    except Exception as e:
        # Default to approval if budget check fails
        return {
            'status': 'APPROVED',
            'note': f'Budget check failed: {str(e)}'
        }

def get_user_department(user_arn):
    """Extract department from user ARN or tags"""
    iam = boto3.client('iam')
    
    username = user_arn.split('/')[-1]
    
    try:
        user_tags = iam.list_user_tags(UserName=username)
        for tag in user_tags['Tags']:
            if tag['Key'] == 'Department':
                return tag['Value']
    except:
        pass
    
    return 'default'

def get_product_estimated_cost(product_id):
    """Get estimated monthly cost for a product"""
    
    # This would typically integrate with AWS Pricing API
    # or maintain a cost database
    cost_estimates = {
        'prod-web-server': 50.0,
        'prod-database': 100.0,
        'prod-load-balancer': 25.0
    }
    
    return cost_estimates.get(product_id, 0.0)
```

### Compliance Monitoring
```python
# Lambda function for compliance monitoring
import boto3
import json
from datetime import datetime, timedelta

def lambda_handler(event, context):
    """
    Monitor Service Catalog provisioned products for compliance
    """
    
    servicecatalog = boto3.client('servicecatalog')
    config_client = boto3.client('config')
    
    # Get all provisioned products
    provisioned_products = servicecatalog.search_provisioned_products()
    
    compliance_report = {
        'timestamp': datetime.utcnow().isoformat(),
        'total_products': len(provisioned_products['ProvisionedProducts']),
        'compliant': 0,
        'non_compliant': 0,
        'violations': []
    }
    
    for product in provisioned_products['ProvisionedProducts']:
        product_id = product['Id']
        product_name = product['Name']
        
        # Get provisioned product details
        details = servicecatalog.describe_provisioned_product(Id=product_id)
        
        # Check compliance rules
        violations = check_product_compliance(details, config_client)
        
        if violations:
            compliance_report['non_compliant'] += 1
            compliance_report['violations'].extend(violations)
        else:
            compliance_report['compliant'] += 1
    
    # Send compliance report
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:service-catalog-compliance',
        Message=json.dumps(compliance_report, indent=2),
        Subject='Service Catalog Compliance Report'
    )
    
    return compliance_report

def check_product_compliance(product_details, config_client):
    """Check compliance rules for a provisioned product"""
    
    violations = []
    
    # Get CloudFormation stack resources
    stack_id = product_details['ProvisionedProductDetail']['CloudformationStackArn']
    
    try:
        cf_client = boto3.client('cloudformation')
        resources = cf_client.describe_stack_resources(StackName=stack_id)
        
        for resource in resources['StackResources']:
            resource_type = resource['ResourceType']
            resource_id = resource['PhysicalResourceId']
            
            # Check Config compliance for each resource
            if resource_type in ['AWS::EC2::Instance', 'AWS::S3::Bucket', 'AWS::RDS::DBInstance']:
                compliance = config_client.get_compliance_details_by_resource(
                    ResourceType=resource_type,
                    ResourceId=resource_id
                )
                
                for evaluation in compliance['EvaluationResults']:
                    if evaluation['ComplianceType'] == 'NON_COMPLIANT':
                        violations.append({
                            'resource_type': resource_type,
                            'resource_id': resource_id,
                            'rule': evaluation['ConfigRuleName'],
                            'annotation': evaluation.get('Annotation', '')
                        })
    
    except Exception as e:
        print(f"Error checking compliance: {str(e)}")
    
    return violations
```

---

## 7. API and CLI Operations

### Common CLI Commands
```bash
# List portfolios
aws servicecatalog list-portfolios

# Describe portfolio
aws servicecatalog describe-portfolio --id port-abcdefghijklm

# Search products
aws servicecatalog search-products --filters FullTextSearch=web

# Describe product
aws servicecatalog describe-product --id prod-abcdefghijklm

# Provision product
aws servicecatalog provision-product \
  --product-id prod-abcdefghijklm \
  --provisioning-artifact-id pa-abcdefghijklm \
  --provisioned-product-name my-web-server \
  --provisioning-parameters Key=InstanceType,Value=t3.small

# List provisioned products
aws servicecatalog search-provisioned-products

# Update provisioned product
aws servicecatalog update-provisioned-product \
  --provisioned-product-id pp-abcdefghijklm \
  --provisioning-artifact-id pa-newversion

# Terminate provisioned product
aws servicecatalog terminate-provisioned-product \
  --provisioned-product-id pp-abcdefghijklm

# Create product
aws servicecatalog create-product \
  --name "Database Server" \
  --owner "IT Team" \
  --description "Standard database server" \
  --provisioning-artifact-parameters Name=v1.0,Info='{LoadTemplateFromURL=https://bucket.s3.amazonaws.com/template.yaml}'

# Associate product with portfolio
aws servicecatalog associate-product-with-portfolio \
  --product-id prod-abcdefghijklm \
  --portfolio-id port-abcdefghijklm
```

### Programmatic Access
```python
import boto3

class ServiceCatalogManager:
    def __init__(self):
        self.client = boto3.client('servicecatalog')
    
    def provision_product_for_user(self, product_name, user_id, parameters):
        """Provision a product for a specific user"""
        
        # Search for product
        products = self.client.search_products(
            Filters={'FullTextSearch': [product_name]}
        )
        
        if not products['ProductViewSummaries']:
            raise ValueError(f"Product {product_name} not found")
        
        product = products['ProductViewSummaries'][0]
        product_id = product['ProductId']
        
        # Get latest provisioning artifact
        artifacts = self.client.describe_product(Id=product_id)
        latest_artifact = artifacts['ProvisioningArtifacts'][-1]
        
        # Provision product
        response = self.client.provision_product(
            ProductId=product_id,
            ProvisioningArtifactId=latest_artifact['Id'],
            ProvisionedProductName=f"{product_name}-{user_id}",
            ProvisioningParameters=[
                {'Key': k, 'Value': v} for k, v in parameters.items()
            ],
            Tags=[
                {'Key': 'Owner', 'Value': user_id},
                {'Key': 'ProvisionedBy', 'Value': 'ServiceCatalogManager'}
            ]
        )
        
        return response['RecordDetail']
    
    def get_user_products(self, user_id):
        """Get all products provisioned by a user"""
        
        provisioned_products = self.client.search_provisioned_products(
            Filters={
                'SearchQuery': [f'owner:{user_id}']
            }
        )
        
        return provisioned_products['ProvisionedProducts']
    
    def update_product_template(self, product_id, template_url, version_name):
        """Update product with new template version"""
        
        response = self.client.create_provisioning_artifact(
            ProductId=product_id,
            Parameters={
                'Name': version_name,
                'Description': f'Updated template version {version_name}',
                'Info': {'LoadTemplateFromURL': template_url},
                'Type': 'CLOUD_FORMATION_TEMPLATE'
            }
        )
        
        return response['ProvisioningArtifactDetail']
    
    def enforce_tagging_policy(self, portfolio_id, required_tags):
        """Enforce tagging policy for a portfolio"""
        
        # Get all products in portfolio
        products = self.client.search_products_as_admin(
            PortfolioId=portfolio_id
        )
        
        for product in products['ProductViewDetails']:
            product_id = product['ProductViewSummary']['ProductId']
            
            # Add tag options for required tags
            for tag_key in required_tags:
                try:
                    # Create tag option if it doesn't exist
                    tag_option = self.client.create_tag_option(
                        Key=tag_key,
                        Value='*'  # Wildcard to allow any value
                    )
                    
                    # Associate with product
                    self.client.associate_tag_option_with_resource(
                        ResourceId=product_id,
                        TagOptionId=tag_option['TagOptionDetail']['Id']
                    )
                except self.client.exceptions.DuplicateResourceException:
                    pass  # Tag option already exists
```

---

## 8. Common Exam Scenarios

### Scenario 1: Self-Service Infrastructure
```yaml
# Complete self-service infrastructure setup
Resources:
  # Developer Portfolio
  DeveloperPortfolio:
    Type: AWS::ServiceCatalog::Portfolio
    Properties:
      DisplayName: Developer Self-Service
      Description: Pre-approved infrastructure for development teams
      ProviderName: Platform Engineering

  # Web Application Product
  WebAppProduct:
    Type: AWS::ServiceCatalog::CloudFormationProduct
    Properties:
      Name: Web Application Stack
      Description: Complete web application with database
      Owner: Platform Engineering
      ProvisioningArtifactParameters:
        - Name: v1.0
          Description: Initial version with basic web app
          Info:
            LoadTemplateFromURL: !Sub "https://${TemplateBucket}.s3.amazonaws.com/web-app-stack.yaml"

  # Database Product
  DatabaseProduct:
    Type: AWS::ServiceCatalog::CloudFormationProduct
    Properties:
      Name: RDS Database
      Description: Managed database instance
      Owner: Platform Engineering
      ProvisioningArtifactParameters:
        - Name: v1.0
          Description: MySQL database with backup
          Info:
            LoadTemplateFromURL: !Sub "https://${TemplateBucket}.s3.amazonaws.com/rds-database.yaml"

  # Associate products with portfolio
  WebAppAssociation:
    Type: AWS::ServiceCatalog::PortfolioProductAssociation
    Properties:
      PortfolioId: !Ref DeveloperPortfolio
      ProductId: !Ref WebAppProduct

  DatabaseAssociation:
    Type: AWS::ServiceCatalog::PortfolioProductAssociation
    Properties:
      PortfolioId: !Ref DeveloperPortfolio
      ProductId: !Ref DatabaseProduct

  # Grant access to developer role
  DeveloperAccess:
    Type: AWS::ServiceCatalog::PortfolioPrincipalAssociation
    Properties:
      PortfolioId: !Ref DeveloperPortfolio
      PrincipalARN: !Sub "arn:aws:iam::${AWS::AccountId}:role/DeveloperRole"
      PrincipalType: IAM

  # Launch constraints
  WebAppLaunchConstraint:
    Type: AWS::ServiceCatalog::LaunchRoleConstraint
    Properties:
      PortfolioId: !Ref DeveloperPortfolio
      ProductId: !Ref WebAppProduct
      RoleArn: !GetAtt ServiceCatalogLaunchRole.Arn

  # Template constraints for cost control
  WebAppTemplateConstraint:
    Type: AWS::ServiceCatalog::LaunchTemplateConstraint
    Properties:
      PortfolioId: !Ref DeveloperPortfolio
      ProductId: !Ref WebAppProduct
      Rules: |
        {
          "Rules": {
            "InstanceSizeRule": {
              "Assertions": [
                {
                  "Assert": {
                    "Fn::Contains": [
                      ["t3.micro", "t3.small"],
                      {"Ref": "InstanceType"}
                    ]
                  },
                  "AssertDescription": "Only t3.micro and t3.small allowed for development"
                }
              ]
            }
          }
        }
```

### Scenario 2: Multi-Environment Deployment
```python
# Automated multi-environment deployment
import boto3
import json

class MultiEnvironmentDeployer:
    def __init__(self):
        self.servicecatalog = boto3.client('servicecatalog')
        self.organizations = boto3.client('organizations')
    
    def deploy_to_environments(self, product_template, environments):
        """Deploy product to multiple environments"""
        
        deployment_results = {}
        
        for env_name, env_config in environments.items():
            try:
                # Create environment-specific product
                product_name = f"{env_config['product_base_name']}-{env_name}"
                
                # Customize template for environment
                customized_template = self.customize_template_for_environment(
                    product_template, env_config
                )
                
                # Upload template to S3
                template_url = self.upload_template_to_s3(
                    customized_template, product_name
                )
                
                # Create or update product
                product_id = self.create_or_update_product(
                    product_name, template_url, env_config
                )
                
                # Share with target accounts if needed
                if env_config.get('target_accounts'):
                    self.share_product_with_accounts(
                        product_id, env_config['target_accounts']
                    )
                
                deployment_results[env_name] = {
                    'status': 'SUCCESS',
                    'product_id': product_id,
                    'template_url': template_url
                }
                
            except Exception as e:
                deployment_results[env_name] = {
                    'status': 'FAILED',
                    'error': str(e)
                }
        
        return deployment_results
    
    def customize_template_for_environment(self, template, env_config):
        """Customize CloudFormation template for specific environment"""
        
        # Parse template
        template_dict = json.loads(template) if isinstance(template, str) else template
        
        # Update parameters with environment-specific defaults
        if 'Parameters' in template_dict:
            for param_name, param_config in template_dict['Parameters'].items():
                if param_name in env_config.get('parameter_defaults', {}):
                    param_config['Default'] = env_config['parameter_defaults'][param_name]
        
        # Update mappings for environment-specific values
        if 'Mappings' in template_dict and 'EnvironmentMap' in template_dict['Mappings']:
            template_dict['Mappings']['EnvironmentMap'][env_config['name']] = env_config.get('mappings', {})
        
        return json.dumps(template_dict, indent=2)

# Example usage
environments = {
    'dev': {
        'product_base_name': 'WebApp',
        'name': 'dev',
        'parameter_defaults': {
            'InstanceType': 't3.micro',
            'MinSize': '1',
            'MaxSize': '2'
        },
        'mappings': {
            'VpcId': 'vpc-dev123',
            'SubnetIds': ['subnet-dev1', 'subnet-dev2']
        }
    },
    'prod': {
        'product_base_name': 'WebApp',
        'name': 'prod',
        'parameter_defaults': {
            'InstanceType': 't3.large',
            'MinSize': '3',
            'MaxSize': '10'
        },
        'mappings': {
            'VpcId': 'vpc-prod123',
            'SubnetIds': ['subnet-prod1', 'subnet-prod2']
        },
        'target_accounts': ['123456789012', '210987654321']
    }
}
```

---

## 9. Exam Tips

- **Understand portfolio structure** - Portfolios, products, provisioning artifacts
- **Master constraints** - Launch, template, notification, and stack set constraints
- **Know access control** - IAM integration and principal associations
- **Practice multi-account** - Portfolio sharing and cross-account access
- **Learn automation** - CI/CD integration and automated product updates
- **Understand governance** - Budget constraints, compliance monitoring, tagging
- **Know API operations** - Key CLI commands and programmatic access
- **Practice troubleshooting** - Common provisioning and constraint issues
- **Understand costs** - Service Catalog pricing and cost optimization
- **Master integration** - CloudFormation, Organizations, Config integration