# AWS API Gateway - DOP-C02 Study Notes

## 1. Overview

### What is API Gateway?
- Fully managed service for creating, publishing, and managing APIs
- Supports REST, HTTP, and WebSocket APIs
- Handles traffic management, authorization, monitoring, and API versioning
- Integrates with Lambda, EC2, and other AWS services

### Key Benefits
- **Serverless** - No infrastructure to manage
- **Scalable** - Handles thousands of concurrent API calls
- **Secure** - Built-in authorization and authentication
- **Monitoring** - CloudWatch integration and X-Ray tracing
- **Cost Effective** - Pay per API call

---

## 2. API Types

### REST API
```bash
# Create REST API
aws apigateway create-rest-api \
  --name "my-rest-api" \
  --description "REST API for microservices" \
  --endpoint-configuration types=REGIONAL
```

### HTTP API
```bash
# Create HTTP API (faster and cheaper)
aws apigatewayv2 create-api \
  --name "my-http-api" \
  --protocol-type HTTP \
  --target "arn:aws:lambda:us-east-1:123456789012:function:MyFunction"
```

### WebSocket API
```bash
# Create WebSocket API
aws apigatewayv2 create-api \
  --name "my-websocket-api" \
  --protocol-type WEBSOCKET \
  --route-selection-expression '$request.body.action'
```

---

## 3. Integration with CI/CD

### CodePipeline Integration
```yaml
# buildspec.yml for API deployment
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: 14
  pre_build:
    commands:
      - npm install -g aws-cli
  build:
    commands:
      - aws apigateway create-deployment --rest-api-id $API_ID --stage-name prod
  post_build:
    commands:
      - echo "API deployed successfully"
```

### SAM Template for API
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: MyCognitoAuthorizer
        Authorizers:
          MyCognitoAuthorizer:
            UserPoolArn: !GetAtt MyCognitoUserPool.Arn
      
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: index.handler
      Runtime: nodejs14.x
      Events:
        ApiEvent:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /users
            Method: get
```

---

## 4. Authentication and Authorization

### Cognito Integration
```python
def create_cognito_authorizer():
    """Create Cognito User Pool authorizer"""
    
    apigateway = boto3.client('apigateway')
    
    response = apigateway.create_authorizer(
        restApiId='api-id',
        name='CognitoAuthorizer',
        type='COGNITO_USER_POOLS',
        providerARNs=[
            'arn:aws:cognito-idp:us-east-1:123456789012:userpool/us-east-1_ABC123'
        ],
        identitySource='method.request.header.Authorization'
    )
    
    return response
```

### Lambda Authorizer
```python
def lambda_authorizer(event, context):
    """Custom Lambda authorizer"""
    
    token = event['authorizationToken']
    
    # Validate token
    if validate_token(token):
        policy = generate_policy('user', 'Allow', event['methodArn'])
    else:
        policy = generate_policy('user', 'Deny', event['methodArn'])
    
    return policy

def generate_policy(principal_id, effect, resource):
    """Generate IAM policy for API Gateway"""
    
    return {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Action': 'execute-api:Invoke',
                    'Effect': effect,
                    'Resource': resource
                }
            ]
        }
    }
```

---

## 5. Deployment Strategies

### Blue/Green Deployment
```python
def blue_green_api_deployment():
    """Implement blue/green deployment for API Gateway"""
    
    apigateway = boto3.client('apigateway')
    
    # Create new deployment (green)
    green_deployment = apigateway.create_deployment(
        restApiId='api-id',
        stageName='green',
        description='Green deployment for testing'
    )
    
    # Test green deployment
    if test_deployment('green'):
        # Switch traffic to green
        apigateway.update_stage(
            restApiId='api-id',
            stageName='prod',
            patchOps=[
                {
                    'op': 'replace',
                    'path': '/deploymentId',
                    'value': green_deployment['id']
                }
            ]
        )
        
        # Clean up old blue deployment
        cleanup_old_deployment('blue')
    else:
        # Rollback - keep blue active
        rollback_deployment('blue')
```

### Canary Deployment
```bash
# Create canary deployment
aws apigateway create-deployment \
  --rest-api-id "api-id" \
  --stage-name "prod" \
  --canary-settings '{
    "percentTraffic": 10,
    "deploymentId": "new-deployment-id",
    "useStageCache": false
  }'
```

---

## 6. Monitoring and Logging

### CloudWatch Integration
```python
def setup_api_monitoring():
    """Set up API Gateway monitoring"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create alarm for high error rate
    cloudwatch.put_metric_alarm(
        AlarmName='API-HighErrorRate',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        MetricName='4XXError',
        Namespace='AWS/ApiGateway',
        Period=300,
        Statistic='Sum',
        Threshold=10,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:api-alerts'
        ],
        Dimensions=[
            {
                'Name': 'ApiName',
                'Value': 'my-rest-api'
            }
        ]
    )
```

### X-Ray Tracing
```bash
# Enable X-Ray tracing
aws apigateway update-stage \
  --rest-api-id "api-id" \
  --stage-name "prod" \
  --patch-ops op=replace,path=/tracingConfig/tracingEnabled,value=true
```

---

## 7. Common Exam Scenarios

### Scenario 1: Secure API with authentication
**Solution:**
- Use Cognito User Pool authorizer
- Implement proper IAM roles
- Enable API keys for additional security
- Set up rate limiting and throttling

### Scenario 2: CI/CD pipeline for API deployment
**Solution:**
- Use SAM or CloudFormation templates
- Implement automated testing stages
- Use canary or blue/green deployments
- Monitor deployment success with CloudWatch

### Scenario 3: Multi-environment API management
**Solution:**
- Use stage variables for environment configuration
- Implement proper stage promotion process
- Use different Lambda aliases per stage
- Configure environment-specific monitoring

---

## 8. CLI Commands Reference

```bash
# Create REST API
aws apigateway create-rest-api --name "my-api"

# Create resource
aws apigateway create-resource \
  --rest-api-id "api-id" \
  --parent-id "parent-resource-id" \
  --path-part "users"

# Create method
aws apigateway put-method \
  --rest-api-id "api-id" \
  --resource-id "resource-id" \
  --http-method GET \
  --authorization-type NONE

# Deploy API
aws apigateway create-deployment \
  --rest-api-id "api-id" \
  --stage-name prod
```

---

## 9. Exam Tips

### Key Points to Remember
- API Gateway integrates with Lambda, EC2, and HTTP endpoints
- Supports multiple authentication methods (Cognito, Lambda, IAM)
- Canary deployments allow gradual traffic shifting
- CloudWatch provides comprehensive monitoring
- Stage variables enable environment-specific configuration

### Common Mistakes
- Not enabling CORS for browser-based applications
- Forgetting to deploy API after making changes
- Not implementing proper error handling and validation
- Overlooking rate limiting and throttling configuration

### Best Practices for Exam
- Understand integration patterns with Lambda and other services
- Know authentication and authorization options
- Understand deployment strategies and stage management
- Know monitoring and troubleshooting approaches