# AWS SAM (Serverless Application Model) - DOP-C02 Exam Notes

## 1. Overview

**AWS SAM** is an open-source framework for building serverless applications on AWS. It provides shorthand syntax to express functions, APIs, databases, and event source mappings.

### Key Characteristics
- **Simplified syntax** - Shorthand CloudFormation for serverless resources
- **Local development** - Test and debug locally with SAM CLI
- **Built-in best practices** - Security, performance, and cost optimization
- **CloudFormation extension** - Transforms SAM templates to CloudFormation
- **CI/CD integration** - Native support for deployment pipelines
- **Multi-runtime support** - Node.js, Python, Java, C#, Go, Ruby, PowerShell
- **Event source integration** - API Gateway, S3, DynamoDB, SQS, SNS, etc.

### What Problem Does It Solve?
- Simplifies serverless application development and deployment
- Provides local testing environment for Lambda functions
- Reduces CloudFormation template complexity for serverless resources
- Enables rapid prototyping and development of serverless applications
- Facilitates serverless application packaging and deployment
- Supports serverless application monitoring and debugging

---

## 2. SAM Template Structure

### Basic SAM Template
```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Simple SAM application

Globals:
  Function:
    Timeout: 30
    Runtime: python3.9
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /hello
            Method: get

  HelloWorldApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: !Ref Environment
      Cors:
        AllowMethods: "'GET,POST,OPTIONS'"
        AllowHeaders: "'content-type'"
        AllowOrigin: "'*'"

Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}/hello/"
  HelloWorldFunction:
    Description: "Hello World Lambda Function ARN"
    Value: !GetAtt HelloWorldFunction.Arn
```

### Function with Multiple Event Sources
```yaml
Resources:
  ProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: processor/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        # API Gateway event
        ApiEvent:
          Type: Api
          Properties:
            Path: /process
            Method: post
        
        # S3 event
        S3Event:
          Type: S3
          Properties:
            Bucket: !Ref ProcessingBucket
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: uploads/
        
        # DynamoDB event
        DynamoDBEvent:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt ProcessingTable.StreamArn
            StartingPosition: TRIM_HORIZON
            BatchSize: 10
        
        # SQS event
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt ProcessingQueue.Arn
            BatchSize: 10
        
        # CloudWatch Events
        ScheduleEvent:
          Type: Schedule
          Properties:
            Schedule: rate(5 minutes)
            Input: '{"source": "scheduled"}'

  ProcessingBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-processing-bucket"

  ProcessingTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub "${AWS::StackName}-processing-table"
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES
      BillingMode: PAY_PER_REQUEST

  ProcessingQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${AWS::StackName}-processing-queue"
```

---

## 3. SAM CLI Commands

### Local Development
```bash
# Initialize new SAM application
sam init
sam init --runtime python3.9 --name my-sam-app

# Build application
sam build
sam build --use-container  # Build in Docker container

# Local testing
sam local start-api                    # Start API Gateway locally
sam local start-api --port 8080       # Custom port
sam local start-lambda                # Start Lambda service locally

# Invoke function locally
sam local invoke HelloWorldFunction
sam local invoke HelloWorldFunction --event events/event.json
sam local invoke --env-vars env.json

# Generate sample events
sam local generate-event apigateway aws-proxy
sam local generate-event s3 put
sam local generate-event dynamodb update

# Local debugging
sam local start-api --debug-port 5858
sam local invoke --debug-port 5858 HelloWorldFunction
```

### Deployment Commands
```bash
# Deploy application
sam deploy --guided                    # Interactive deployment
sam deploy                            # Use samconfig.toml
sam deploy --parameter-overrides Environment=prod

# Package application
sam package --s3-bucket my-deployment-bucket --output-template-file packaged.yaml

# Validate template
sam validate
sam validate --template template.yaml

# Delete application
sam delete
sam delete --stack-name my-sam-app
```

### Configuration File
```toml
# samconfig.toml
version = 0.1
[default]
[default.deploy]
[default.deploy.parameters]
stack_name = "my-sam-app"
s3_bucket = "my-deployment-bucket"
s3_prefix = "my-sam-app"
region = "us-east-1"
capabilities = "CAPABILITY_IAM"
parameter_overrides = "Environment=dev"
confirm_changeset = true
fail_on_empty_changeset = false

[production]
[production.deploy]
[production.deploy.parameters]
stack_name = "my-sam-app-prod"
s3_bucket = "my-deployment-bucket-prod"
region = "us-east-1"
parameter_overrides = "Environment=prod"
```

---

## 4. Advanced SAM Features

### Nested Applications
```yaml
# Parent template
Resources:
  AuthenticationApp:
    Type: AWS::Serverless::Application
    Properties:
      Location: auth/template.yaml
      Parameters:
        Environment: !Ref Environment

  ProcessingApp:
    Type: AWS::Serverless::Application
    Properties:
      Location: processing/template.yaml
      Parameters:
        Environment: !Ref Environment
        AuthTable: !GetAtt AuthenticationApp.Outputs.UserTable

# Child template (auth/template.yaml)
Parameters:
  Environment:
    Type: String

Resources:
  UserTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub "${Environment}-users"
      # ... table configuration

Outputs:
  UserTable:
    Description: User table name
    Value: !Ref UserTable
    Export:
      Name: !Sub "${AWS::StackName}-UserTable"
```

### Layers
```yaml
Resources:
  SharedLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: shared-dependencies
      Description: Shared dependencies layer
      ContentUri: layers/shared/
      CompatibleRuntimes:
        - python3.9
        - python3.8
      RetentionPolicy: Delete

  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      Layers:
        - !Ref SharedLayer
        - arn:aws:lambda:us-east-1:123456789012:layer:external-layer:1
```

### Custom Authorizers
```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: !Ref Environment
      Auth:
        DefaultAuthorizer: MyLambdaAuthorizer
        Authorizers:
          MyLambdaAuthorizer:
            FunctionArn: !GetAtt AuthorizerFunction.Arn
            Identity:
              Headers:
                - Authorization
              ReauthorizeEvery: 300

  AuthorizerFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: authorizer/
      Handler: app.lambda_handler
      Runtime: python3.9

  ProtectedFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: protected/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        ProtectedApi:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /protected
            Method: get
            Auth:
              Authorizer: MyLambdaAuthorizer
```

---

## 5. SAM with CI/CD

### CodePipeline Integration
```yaml
# pipeline.yaml
Resources:
  ArtifactsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-artifacts"
      VersioningConfiguration:
        Status: Enabled

  CodePipeline:
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
                RepositoryName: !Ref RepoName
                BranchName: main
              OutputArtifacts:
                - Name: SourceOutput

        - Name: Build
          Actions:
            - Name: BuildAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref CodeBuildProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput

        - Name: Deploy-Dev
          Actions:
            - Name: DeployDev
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: '1'
              Configuration:
                ActionMode: CREATE_UPDATE
                StackName: !Sub "${AWS::StackName}-dev"
                TemplatePath: BuildOutput::packaged-template.yaml
                Capabilities: CAPABILITY_IAM
                ParameterOverrides: |
                  Environment=dev
                RoleArn: !GetAtt CloudFormationRole.Arn
              InputArtifacts:
                - Name: BuildOutput

        - Name: Deploy-Prod
          Actions:
            - Name: ApprovalAction
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              Configuration:
                CustomData: 'Please review and approve production deployment'
            
            - Name: DeployProd
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: '1'
              Configuration:
                ActionMode: CREATE_UPDATE
                StackName: !Sub "${AWS::StackName}-prod"
                TemplatePath: BuildOutput::packaged-template.yaml
                Capabilities: CAPABILITY_IAM
                ParameterOverrides: |
                  Environment=prod
                RoleArn: !GetAtt CloudFormationRole.Arn
              InputArtifacts:
                - Name: BuildOutput
              RunOrder: 2
```

### CodeBuild Project
```yaml
Resources:
  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Sub "${AWS::StackName}-build"
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
                - pip install aws-sam-cli
            pre_build:
              commands:
                - echo Build started on `date`
            build:
              commands:
                - sam build
                - sam package --s3-bucket $ARTIFACTS_BUCKET --output-template-file packaged-template.yaml
            post_build:
              commands:
                - echo Build completed on `date`
          artifacts:
            files:
              - packaged-template.yaml
```

---

## 6. Testing SAM Applications

### Unit Testing
```python
# tests/unit/test_handler.py
import json
import pytest
from hello_world import app

@pytest.fixture()
def apigw_event():
    """Generates API GW Event"""
    return {
        "body": '{"test": "body"}',
        "resource": "/{proxy+}",
        "httpMethod": "POST",
        "isBase64Encoded": False,
        "queryStringParameters": {"foo": "bar"},
        "pathParameters": {"proxy": "/path/to/resource"},
        "stageVariables": {"baz": "qux"},
        "headers": {
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
            "Accept-Encoding": "gzip, deflate, sdch",
            "Accept-Language": "en-US,en;q=0.8",
            "Cache-Control": "max-age=0",
            "CloudFront-Forwarded-Proto": "https",
            "CloudFront-Is-Desktop-Viewer": "true",
            "CloudFront-Is-Mobile-Viewer": "false",
            "CloudFront-Is-SmartTV-Viewer": "false",
            "CloudFront-Is-Tablet-Viewer": "false",
            "CloudFront-Viewer-Country": "US",
            "Host": "1234567890.execute-api.us-east-1.amazonaws.com",
            "Upgrade-Insecure-Requests": "1",
            "User-Agent": "Custom User Agent String",
            "Via": "1.1 08f323deadbeefa7af34d5feb414ce27.cloudfront.net (CloudFront)",
            "X-Amz-Cf-Id": "cDehVQoZnx43VYQb9j2-nvCh-9z396Uhbp027Y2JvkCPNLmGJHqlaA==",
            "X-Forwarded-For": "127.0.0.1, 127.0.0.2",
            "X-Forwarded-Port": "443",
            "X-Forwarded-Proto": "https"
        },
        "requestContext": {
            "accountId": "123456789012",
            "resourceId": "123456",
            "stage": "prod",
            "requestId": "c6af9ac6-7b61-11e6-9a41-93e8deadbeef",
            "requestTime": "09/Apr/2015:12:34:56 +0000",
            "requestTimeEpoch": 1428582896000,
            "identity": {
                "cognitoIdentityPoolId": None,
                "accountId": None,
                "cognitoIdentityId": None,
                "caller": None,
                "accessKey": None,
                "sourceIp": "127.0.0.1",
                "cognitoAuthenticationType": None,
                "cognitoAuthenticationProvider": None,
                "userArn": None,
                "userAgent": "Custom User Agent String",
                "user": None
            },
            "path": "/prod/path/to/resource",
            "resourcePath": "/{proxy+}",
            "httpMethod": "POST",
            "apiId": "1234567890",
            "protocol": "HTTP/1.1"
        }
    }

def test_lambda_handler(apigw_event, mocker):
    ret = app.lambda_handler(apigw_event, "")
    data = json.loads(ret["body"])

    assert ret["statusCode"] == 200
    assert "message" in ret["body"]
    assert data["message"] == "hello world"
```

### Integration Testing
```python
# tests/integration/test_api_gateway.py
import boto3
import pytest
import requests

class TestApiGateway:
    @pytest.fixture(autouse=True)
    def setup(self):
        """Setup test fixtures"""
        self.api_endpoint = self.get_stack_output("HelloWorldApi")
    
    def get_stack_output(self, output_key):
        """Get CloudFormation stack output"""
        cf = boto3.client('cloudformation')
        stacks = cf.describe_stacks(StackName='sam-app')
        outputs = stacks['Stacks'][0]['Outputs']
        
        for output in outputs:
            if output['OutputKey'] == output_key:
                return output['OutputValue']
        
        raise ValueError(f"Output {output_key} not found")
    
    def test_api_gateway_response(self):
        """Test API Gateway endpoint"""
        response = requests.get(self.api_endpoint)
        
        assert response.status_code == 200
        data = response.json()
        assert "message" in data
        assert data["message"] == "hello world"
```

---

## 7. SAM Policy Templates

### Built-in Policy Templates
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      Policies:
        # Simple policy templates
        - S3ReadPolicy:
            BucketName: !Ref MyBucket
        - DynamoDBCrudPolicy:
            TableName: !Ref MyTable
        - SQSPollerPolicy:
            QueueName: !GetAtt MyQueue.QueueName
        - SNSPublishMessagePolicy:
            TopicName: !GetAtt MyTopic.TopicName
        - VPCAccessPolicy: {}
        - CloudWatchLogsFullAccess
        
        # Custom policy
        - Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - secretsmanager:GetSecretValue
              Resource: !Ref MySecret
```

### Custom Policy Templates
```yaml
# Create custom policy template
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      Policies:
        - Statement:
            - Effect: Allow
              Action:
                - dynamodb:GetItem
                - dynamodb:PutItem
                - dynamodb:UpdateItem
                - dynamodb:DeleteItem
              Resource: 
                - !GetAtt MyTable.Arn
                - !Sub "${MyTable.Arn}/index/*"
            - Effect: Allow
              Action:
                - s3:GetObject
                - s3:PutObject
              Resource: !Sub "${MyBucket}/*"
```

---

## 8. Environment Variables and Secrets

### Environment Variables
```yaml
Globals:
  Function:
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment
        LOG_LEVEL: INFO
        REGION: !Ref AWS::Region

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      Environment:
        Variables:
          TABLE_NAME: !Ref MyTable
          BUCKET_NAME: !Ref MyBucket
          API_ENDPOINT: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}"
```

### Secrets Management
```yaml
Resources:
  DatabaseSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: !Sub "${AWS::StackName}-db-secret"
      Description: Database credentials
      GenerateSecretString:
        SecretStringTemplate: '{"username": "admin"}'
        GenerateStringKey: 'password'
        PasswordLength: 32
        ExcludeCharacters: '"@/\'

  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      Environment:
        Variables:
          SECRET_ARN: !Ref DatabaseSecret
      Policies:
        - Statement:
            - Effect: Allow
              Action:
                - secretsmanager:GetSecretValue
              Resource: !Ref DatabaseSecret
```

---

## 9. Common Exam Scenarios

### Scenario 1: Multi-Stage Deployment
```yaml
# Different configurations per environment
Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Mappings:
  EnvironmentMap:
    dev:
      InstanceType: t3.micro
      MinCapacity: 1
      MaxCapacity: 2
    staging:
      InstanceType: t3.small
      MinCapacity: 2
      MaxCapacity: 5
    prod:
      InstanceType: t3.medium
      MinCapacity: 5
      MaxCapacity: 20

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      ReservedConcurrencyLimit: !FindInMap [EnvironmentMap, !Ref Environment, MaxCapacity]
      Environment:
        Variables:
          ENVIRONMENT: !Ref Environment
```

### Scenario 2: Blue/Green Deployment
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: function/
      Handler: app.lambda_handler
      Runtime: python3.9
      AutoPublishAlias: live
      DeploymentPreference:
        Type: Canary10Percent5Minutes
        Alarms:
          - !Ref AliasErrorMetricGreaterThanZeroAlarm
          - !Ref LatestVersionErrorMetricGreaterThanZeroAlarm
        Hooks:
          PreTraffic: !Ref PreTrafficHook
          PostTraffic: !Ref PostTrafficHook

  AliasErrorMetricGreaterThanZeroAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Lambda function errors
      ComparisonOperator: GreaterThanThreshold
      EvaluationPeriods: 2
      MetricName: Errors
      Namespace: AWS/Lambda
      Period: 60
      Statistic: Sum
      Threshold: 0
      Dimensions:
        - Name: FunctionName
          Value: !Ref MyFunction
        - Name: Resource
          Value: !Sub "${MyFunction}:live"
```

### Scenario 3: Event-Driven Architecture
```yaml
Resources:
  # S3 trigger
  ProcessingFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: processing/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        S3Event:
          Type: S3
          Properties:
            Bucket: !Ref InputBucket
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: input/
                  - Name: suffix
                    Value: .json

  # DynamoDB trigger
  NotificationFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: notification/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        DynamoDBEvent:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt ProcessingTable.StreamArn
            StartingPosition: TRIM_HORIZON
            BatchSize: 10
            FilterCriteria:
              Filters:
                - Pattern: '{"eventName": ["INSERT", "MODIFY"]}'
```

---

## 10. Exam Tips

- **Understand SAM transforms** - How SAM templates become CloudFormation
- **Master local development** - SAM CLI commands for testing and debugging
- **Know event sources** - API Gateway, S3, DynamoDB, SQS, SNS integrations
- **Practice deployment** - Guided deployment and CI/CD integration
- **Understand policy templates** - Built-in and custom IAM policies
- **Learn nested applications** - Modular serverless architecture
- **Know deployment preferences** - Blue/green and canary deployments
- **Practice with layers** - Shared code and dependencies
- **Understand globals** - Template-wide configuration
- **Master troubleshooting** - Common SAM CLI and deployment issues