# AWS CodePipeline Cross-Account Deployments - DOP-C02 Exam Notes

## 1. Overview

**AWS CodePipeline Cross-Account Deployments** enable you to deploy applications across multiple AWS accounts from a centralized CI/CD pipeline. This pattern is essential for enterprise environments where different accounts represent different environments (dev, staging, production) or organizational boundaries.

### Key Characteristics
- **Centralized CI/CD** - Single pipeline manages multi-account deployments
- **Account isolation** - Each environment in separate AWS account
- **Cross-account IAM** - Secure role assumption across accounts
- **Artifact sharing** - S3 buckets accessible across accounts
- **Governance** - Centralized control with distributed execution
- **Security** - Least privilege access with account boundaries
- **Compliance** - Audit trails across organizational structure

### What Problem Does It Solve?
- Enables enterprise-scale CI/CD across account boundaries
- Provides secure deployment to production accounts
- Maintains environment isolation while enabling automation
- Supports organizational governance and compliance requirements
- Enables centralized pipeline management with distributed resources
- Facilitates secure artifact sharing across accounts
- Supports complex approval workflows across teams

---

## 2. Core Concepts

### Cross-Account Pipeline Architecture
- **Central Account** - Hosts the CodePipeline and shared resources
- **Target Accounts** - Destination accounts for deployments
- **Cross-Account Roles** - IAM roles for secure access across accounts
- **Artifact Buckets** - S3 buckets accessible from multiple accounts
- **KMS Keys** - Encryption keys shared across accounts

### Account Structure Patterns
- **Environment-based** - Dev, Staging, Production accounts
- **Team-based** - Separate accounts per team or business unit
- **Workload-based** - Accounts per application or service
- **Compliance-based** - Accounts based on regulatory requirements

### Trust Relationships
- **Role Assumption** - Central account assumes roles in target accounts
- **Resource Policies** - S3 buckets and KMS keys allow cross-account access
- **External ID** - Additional security for role assumption
- **Condition Keys** - Fine-grained access control

---

## 3. Cross-Account IAM Setup

### Central Account Pipeline Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codepipeline:*",
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild",
        "codecommit:GetBranch",
        "codecommit:GetCommit",
        "codecommit:UploadArchive",
        "codecommit:GetUploadArchiveStatus"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::central-pipeline-artifacts",
        "arn:aws:s3:::central-pipeline-artifacts/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:111111111111:key/12345678-1234-1234-1234-123456789012"
    },
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::222222222222:role/CrossAccountDeploymentRole",
        "arn:aws:iam::333333333333:role/CrossAccountDeploymentRole"
      ]
    }
  ]
}
```

### Target Account Deployment Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DeleteStack",
        "cloudformation:DescribeStacks",
        "cloudformation:DescribeStackEvents",
        "cloudformation:DescribeStackResources",
        "cloudformation:CreateChangeSet",
        "cloudformation:ExecuteChangeSet",
        "cloudformation:DeleteChangeSet",
        "cloudformation:DescribeChangeSet"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::central-pipeline-artifacts",
        "arn:aws:s3:::central-pipeline-artifacts/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:111111111111:key/12345678-1234-1234-1234-123456789012"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::222222222222:role/CloudFormationExecutionRole"
    }
  ]
}
```

### Trust Policy for Cross-Account Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/CodePipelineServiceRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-12345"
        }
      }
    }
  ]
}
```

---

## 4. S3 Artifact Bucket Configuration

### Cross-Account Bucket Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCentralAccount",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::central-pipeline-artifacts",
        "arn:aws:s3:::central-pipeline-artifacts/*"
      ]
    },
    {
      "Sid": "AllowTargetAccounts",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::222222222222:role/CrossAccountDeploymentRole",
          "arn:aws:iam::333333333333:role/CrossAccountDeploymentRole"
        ]
      },
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::central-pipeline-artifacts",
        "arn:aws:s3:::central-pipeline-artifacts/*"
      ]
    },
    {
      "Sid": "DenyInsecureConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::central-pipeline-artifacts",
        "arn:aws:s3:::central-pipeline-artifacts/*"
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

### KMS Key Policy for Cross-Account Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableCentralAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowTargetAccountDecryption",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::222222222222:role/CrossAccountDeploymentRole",
          "arn:aws:iam::333333333333:role/CrossAccountDeploymentRole"
        ]
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowCodePipelineAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/CodePipelineServiceRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 5. Cross-Account Pipeline Configuration

### Complete Pipeline Example
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Cross-account CodePipeline'

Parameters:
  DevAccountId:
    Type: String
    Default: '222222222222'
  
  ProdAccountId:
    Type: String
    Default: '333333333333'

Resources:
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: central-pipeline-artifacts
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref ArtifactKMSKey
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  ArtifactKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: 'KMS Key for cross-account pipeline artifacts'
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: EnableCentralAccountAccess
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          - Sid: AllowTargetAccountDecryption
            Effect: Allow
            Principal:
              AWS:
                - !Sub 'arn:aws:iam::${DevAccountId}:role/CrossAccountDeploymentRole'
                - !Sub 'arn:aws:iam::${ProdAccountId}:role/CrossAccountDeploymentRole'
            Action:
              - kms:Decrypt
              - kms:DescribeKey
            Resource: '*'

  CodePipelineServiceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: PipelineExecutionPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                  - s3:ListBucket
                Resource:
                  - !Sub '${ArtifactBucket}/*'
                  - !Ref ArtifactBucket
              - Effect: Allow
                Action:
                  - kms:Decrypt
                  - kms:GenerateDataKey
                  - kms:DescribeKey
                Resource: !GetAtt ArtifactKMSKey.Arn
              - Effect: Allow
                Action: sts:AssumeRole
                Resource:
                  - !Sub 'arn:aws:iam::${DevAccountId}:role/CrossAccountDeploymentRole'
                  - !Sub 'arn:aws:iam::${ProdAccountId}:role/CrossAccountDeploymentRole'
              - Effect: Allow
                Action:
                  - codecommit:GetBranch
                  - codecommit:GetCommit
                  - codecommit:UploadArchive
                  - codecommit:GetUploadArchiveStatus
                  - codebuild:BatchGetBuilds
                  - codebuild:StartBuild
                Resource: '*'

  CrossAccountPipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: CrossAccountDeploymentPipeline
      RoleArn: !GetAtt CodePipelineServiceRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
        EncryptionKey:
          Id: !GetAtt ArtifactKMSKey.Arn
          Type: KMS
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeCommit
                Version: 1
              Configuration:
                RepositoryName: MyApplication
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
                Version: 1
              Configuration:
                ProjectName: MyApplicationBuild
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput

        - Name: DeployToDev
          Actions:
            - Name: CreateChangeSetDev
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: 1
              Configuration:
                ActionMode: CHANGE_SET_REPLACE
                StackName: MyApp-Dev
                ChangeSetName: MyApp-Dev-ChangeSet
                TemplatePath: BuildOutput::packaged-template.yaml
                Capabilities: CAPABILITY_IAM
                RoleArn: !Sub 'arn:aws:iam::${DevAccountId}:role/CloudFormationExecutionRole'
                ParameterOverrides: |
                  {
                    "Environment": "Development"
                  }
              InputArtifacts:
                - Name: BuildOutput
              RoleArn: !Sub 'arn:aws:iam::${DevAccountId}:role/CrossAccountDeploymentRole'
              RunOrder: 1

            - Name: ExecuteChangeSetDev
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: 1
              Configuration:
                ActionMode: CHANGE_SET_EXECUTE
                StackName: MyApp-Dev
                ChangeSetName: MyApp-Dev-ChangeSet
              RoleArn: !Sub 'arn:aws:iam::${DevAccountId}:role/CrossAccountDeploymentRole'
              RunOrder: 2

        - Name: ApprovalForProduction
          Actions:
            - Name: ManualApproval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: 1
              Configuration:
                NotificationArn: !Ref ApprovalTopic
                CustomData: 'Please review the development deployment and approve for production'

        - Name: DeployToProduction
          Actions:
            - Name: CreateChangeSetProd
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: 1
              Configuration:
                ActionMode: CHANGE_SET_REPLACE
                StackName: MyApp-Prod
                ChangeSetName: MyApp-Prod-ChangeSet
                TemplatePath: BuildOutput::packaged-template.yaml
                Capabilities: CAPABILITY_IAM
                RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CloudFormationExecutionRole'
                ParameterOverrides: |
                  {
                    "Environment": "Production"
                  }
              InputArtifacts:
                - Name: BuildOutput
              RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CrossAccountDeploymentRole'
              RunOrder: 1

            - Name: ExecuteChangeSetProd
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: 1
              Configuration:
                ActionMode: CHANGE_SET_EXECUTE
                StackName: MyApp-Prod
                ChangeSetName: MyApp-Prod-ChangeSet
              RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CrossAccountDeploymentRole'
              RunOrder: 2

  ApprovalTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: PipelineApprovals
      DisplayName: Pipeline Approval Notifications
```

---

## 6. Advanced Cross-Account Patterns

### Multi-Region Cross-Account Deployment
```yaml
CrossAccountMultiRegionPipeline:
  Type: AWS::CodePipeline::Pipeline
  Properties:
    Name: MultiRegionCrossAccountPipeline
    RoleArn: !GetAtt CodePipelineServiceRole.Arn
    ArtifactStores:
      - Region: us-east-1
        ArtifactStore:
          Type: S3
          Location: !Ref ArtifactBucketUSEast1
          EncryptionKey:
            Id: !GetAtt ArtifactKMSKeyUSEast1.Arn
            Type: KMS
      - Region: us-west-2
        ArtifactStore:
          Type: S3
          Location: !Ref ArtifactBucketUSWest2
          EncryptionKey:
            Id: !GetAtt ArtifactKMSKeyUSWest2.Arn
            Type: KMS
    Stages:
      - Name: Source
        Actions:
          - Name: SourceAction
            ActionTypeId:
              Category: Source
              Owner: AWS
              Provider: CodeCommit
              Version: 1
            Configuration:
              RepositoryName: MyApplication
              BranchName: main
            OutputArtifacts:
              - Name: SourceOutput

      - Name: ParallelRegionDeployment
        Actions:
          - Name: DeployUSEast1
            ActionTypeId:
              Category: Deploy
              Owner: AWS
              Provider: CloudFormation
              Version: 1
            Configuration:
              ActionMode: CREATE_UPDATE
              StackName: MyApp-USEast1
              TemplatePath: SourceOutput::template.yaml
              Capabilities: CAPABILITY_IAM
              RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CloudFormationExecutionRole'
            InputArtifacts:
              - Name: SourceOutput
            RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CrossAccountDeploymentRole'
            Region: us-east-1
            RunOrder: 1

          - Name: DeployUSWest2
            ActionTypeId:
              Category: Deploy
              Owner: AWS
              Provider: CloudFormation
              Version: 1
            Configuration:
              ActionMode: CREATE_UPDATE
              StackName: MyApp-USWest2
              TemplatePath: SourceOutput::template.yaml
              Capabilities: CAPABILITY_IAM
              RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CloudFormationExecutionRole'
            InputArtifacts:
              - Name: SourceOutput
            RoleArn: !Sub 'arn:aws:iam::${ProdAccountId}:role/CrossAccountDeploymentRole'
            Region: us-west-2
            RunOrder: 1
```

### Cross-Account with External ID
```python
# Lambda function for dynamic external ID generation
import boto3
import json
import uuid

def lambda_handler(event, context):
    """
    Generate unique external ID for cross-account role assumption
    """
    
    # Generate unique external ID
    external_id = str(uuid.uuid4())
    
    # Store in Parameter Store for pipeline use
    ssm = boto3.client('ssm')
    ssm.put_parameter(
        Name='/pipeline/cross-account/external-id',
        Value=external_id,
        Type='SecureString',
        Overwrite=True
    )
    
    # Update trust policy in target accounts
    update_trust_policies(external_id, event['target_accounts'])
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'external_id': external_id,
            'message': 'External ID updated successfully'
        })
    }

def update_trust_policies(external_id, target_accounts):
    """Update trust policies in target accounts"""
    
    for account_id in target_accounts:
        # Assume role in target account to update trust policy
        sts = boto3.client('sts')
        assumed_role = sts.assume_role(
            RoleArn=f'arn:aws:iam::{account_id}:role/TrustPolicyUpdateRole',
            RoleSessionName='UpdateTrustPolicy'
        )
        
        # Create IAM client with assumed role credentials
        iam = boto3.client(
            'iam',
            aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
            aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
            aws_session_token=assumed_role['Credentials']['SessionToken']
        )
        
        # Update trust policy
        trust_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::111111111111:role/CodePipelineServiceRole"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "sts:ExternalId": external_id
                        }
                    }
                }
            ]
        }
        
        iam.update_assume_role_policy(
            RoleName='CrossAccountDeploymentRole',
            PolicyDocument=json.dumps(trust_policy)
        )
```

---

## 7. Security Best Practices

### Least Privilege Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DescribeStacks",
        "cloudformation:CreateChangeSet",
        "cloudformation:ExecuteChangeSet",
        "cloudformation:DescribeChangeSet"
      ],
      "Resource": [
        "arn:aws:cloudformation:*:*:stack/MyApp-*/*",
        "arn:aws:cloudformation:*:*:changeSet/MyApp-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::central-pipeline-artifacts/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/Pipeline": "MyApp"
        }
      }
    }
  ]
}
```

### Time-Based Access Control
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/CrossAccountDeploymentRole",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        },
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
        }
      }
    }
  ]
}
```

### Audit and Compliance
```yaml
Resources:
  CrossAccountAuditTrail:
    Type: AWS::CloudTrail::Trail
    Properties:
      TrailName: CrossAccountPipelineAudit
      S3BucketName: !Ref AuditBucket
      IncludeGlobalServiceEvents: true
      IsMultiRegionTrail: true
      EnableLogFileValidation: true
      EventSelectors:
        - ReadWriteType: All
          IncludeManagementEvents: true
          DataResources:
            - Type: AWS::S3::Object
              Values: 
                - !Sub '${ArtifactBucket}/*'

  ComplianceRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cross-account-pipeline-compliance
      Description: Ensures cross-account pipeline follows security best practices
      Source:
        Owner: AWS
        SourceIdentifier: IAM_ROLE_MANAGED_POLICY_CHECK
      InputParameters: |
        {
          "managedPolicyArns": "arn:aws:iam::aws:policy/ReadOnlyAccess"
        }
```

---

## 8. Monitoring & Troubleshooting

### CloudWatch Monitoring
```python
import boto3
import json

def monitor_cross_account_pipeline(event, context):
    """
    Monitor cross-account pipeline executions
    """
    
    codepipeline = boto3.client('codepipeline')
    cloudwatch = boto3.client('cloudwatch')
    
    # Get pipeline execution details
    pipeline_name = event['detail']['pipeline']
    execution_id = event['detail']['execution-id']
    
    try:
        execution = codepipeline.get_pipeline_execution(
            pipelineName=pipeline_name,
            pipelineExecutionId=execution_id
        )
        
        # Check for cross-account action failures
        if execution['pipelineExecution']['status'] == 'Failed':
            failed_actions = get_failed_cross_account_actions(pipeline_name, execution_id)
            
            for action in failed_actions:
                # Publish custom metrics
                cloudwatch.put_metric_data(
                    Namespace='Pipeline/CrossAccount',
                    MetricData=[
                        {
                            'MetricName': 'CrossAccountActionFailure',
                            'Dimensions': [
                                {'Name': 'PipelineName', 'Value': pipeline_name},
                                {'Name': 'ActionName', 'Value': action['actionName']},
                                {'Name': 'TargetAccount', 'Value': action['targetAccount']}
                            ],
                            'Value': 1,
                            'Unit': 'Count'
                        }
                    ]
                )
                
                # Send detailed alert
                send_cross_account_failure_alert(pipeline_name, action)
        
    except Exception as e:
        print(f"Error monitoring pipeline: {str(e)}")

def get_failed_cross_account_actions(pipeline_name, execution_id):
    """Get details of failed cross-account actions"""
    
    codepipeline = boto3.client('codepipeline')
    
    # Get action execution details
    response = codepipeline.list_action_executions(
        pipelineName=pipeline_name,
        filter={
            'pipelineExecutionId': execution_id
        }
    )
    
    failed_actions = []
    for action in response['actionExecutionDetails']:
        if (action['status'] == 'Failed' and 
            'roleArn' in action.get('actionExecutionId', {}).get('actionTypeId', {})):
            
            # Extract target account from role ARN
            role_arn = action['actionExecutionId']['actionTypeId'].get('roleArn', '')
            if 'arn:aws:iam::' in role_arn:
                target_account = role_arn.split(':')[4]
                failed_actions.append({
                    'actionName': action['actionName'],
                    'targetAccount': target_account,
                    'errorMessage': action.get('output', {}).get('executionResult', {}).get('errorDetails', {}).get('message', 'Unknown error')
                })
    
    return failed_actions
```

### Troubleshooting Common Issues
```bash
#!/bin/bash
# Cross-account pipeline troubleshooting script

PIPELINE_NAME="CrossAccountDeploymentPipeline"
CENTRAL_ACCOUNT="111111111111"
TARGET_ACCOUNTS=("222222222222" "333333333333")

echo "Troubleshooting cross-account pipeline: $PIPELINE_NAME"

# Check pipeline status
echo "=== Pipeline Status ==="
aws codepipeline get-pipeline-state --name $PIPELINE_NAME

# Check artifact bucket access
echo "=== Artifact Bucket Access ==="
aws s3 ls s3://central-pipeline-artifacts/ --recursive

# Test cross-account role assumption
echo "=== Testing Cross-Account Role Assumption ==="
for account in "${TARGET_ACCOUNTS[@]}"; do
    echo "Testing access to account: $account"
    
    # Attempt to assume role
    ROLE_ARN="arn:aws:iam::$account:role/CrossAccountDeploymentRole"
    
    aws sts assume-role \
        --role-arn $ROLE_ARN \
        --role-session-name "TroubleshootingSession" \
        --external-id "unique-external-id-12345" \
        --query 'Credentials.AccessKeyId' \
        --output text
    
    if [ $? -eq 0 ]; then
        echo "✓ Successfully assumed role in account $account"
    else
        echo "✗ Failed to assume role in account $account"
    fi
done

# Check KMS key access
echo "=== KMS Key Access ==="
KMS_KEY_ID="12345678-1234-1234-1234-123456789012"
aws kms describe-key --key-id $KMS_KEY_ID

# Check recent pipeline executions
echo "=== Recent Pipeline Executions ==="
aws codepipeline list-pipeline-executions \
    --pipeline-name $PIPELINE_NAME \
    --max-items 5
```

---

## 9. Common Exam Scenarios

### Scenario 1: Setup cross-account deployment from central CI/CD account
**Solution:**
- Create cross-account IAM roles with trust relationships
- Configure S3 bucket policy for artifact sharing
- Set up KMS key policy for cross-account encryption
- Configure pipeline with RoleArn parameter for cross-account actions

### Scenario 2: Secure artifact sharing across accounts with encryption
**Solution:**
- Create KMS key in central account with cross-account permissions
- Configure S3 bucket with KMS encryption
- Grant decrypt permissions to target account roles
- Use encrypted artifact store in pipeline configuration

### Scenario 3: Deploy to multiple accounts in parallel
**Solution:**
- Create multiple actions in same stage with different RoleArn
- Use RunOrder to control parallel vs sequential execution
- Configure separate artifact stores for multi-region deployments
- Implement proper error handling for partial failures

### Scenario 4: Implement approval workflow for production account
**Solution:**
- Add manual approval action before production deployment
- Configure SNS topic for approval notifications
- Use IAM conditions to restrict who can approve
- Implement custom approval logic with Lambda if needed

### Scenario 5: Troubleshoot cross-account role assumption failures
**Solution:**
- Verify trust policy allows central account to assume role
- Check external ID matches between pipeline and trust policy
- Ensure role has necessary permissions for deployment actions
- Verify account IDs and role names are correct

### Scenario 6: Implement least privilege cross-account access
**Solution:**
- Use resource-level permissions in IAM policies
- Implement condition keys for additional security
- Use separate roles for different deployment stages
- Regular audit of cross-account permissions

### Scenario 7: Handle cross-account deployment failures
**Solution:**
- Implement CloudWatch monitoring for cross-account actions
- Set up EventBridge rules for failure notifications
- Create automated rollback procedures
- Implement retry logic for transient failures

### Scenario 8: Migrate from single-account to cross-account pipeline
**Solution:**
- Create target account infrastructure first
- Set up cross-account IAM roles and policies
- Modify existing pipeline to use cross-account actions
- Test thoroughly in non-production environments

---

## 10. CLI Commands Reference

### Cross-Account Role Management
```bash
# Assume cross-account role
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/CrossAccountDeploymentRole \
  --role-session-name CrossAccountSession \
  --external-id unique-external-id-12345

# Test role permissions
aws sts get-caller-identity

# List assumable roles
aws iam list-roles --query 'Roles[?contains(RoleName, `CrossAccount`)]'
```

### Pipeline Operations
```bash
# Create cross-account pipeline
aws codepipeline create-pipeline \
  --cli-input-json file://cross-account-pipeline.json

# Update pipeline with cross-account configuration
aws codepipeline update-pipeline \
  --cli-input-json file://updated-cross-account-pipeline.json

# Get pipeline execution details
aws codepipeline get-pipeline-execution \
  --pipeline-name CrossAccountPipeline \
  --pipeline-execution-id execution-id

# List action executions
aws codepipeline list-action-executions \
  --pipeline-name CrossAccountPipeline \
  --filter pipelineExecutionId=execution-id
```

### Artifact Management
```bash
# List artifacts in cross-account bucket
aws s3 ls s3://central-pipeline-artifacts/ --recursive

# Copy artifacts between accounts (with assumed role)
aws s3 cp s3://central-pipeline-artifacts/artifact.zip . \
  --profile cross-account-profile

# Test KMS key access
aws kms decrypt \
  --ciphertext-blob fileb://encrypted-artifact \
  --key-id arn:aws:kms:us-east-1:111111111111:key/12345678-1234-1234-1234-123456789012
```

---

## 11. Architecture Patterns

### Hub and Spoke Model
```
Central Account (Hub)
├── CodePipeline
├── Artifact S3 Bucket
├── KMS Key
└── Build Resources
    ↓
┌─────────────────────────────────────┐
│ Development │ Staging │ Production  │
│ Account     │ Account │ Account     │
│ (Spoke)     │ (Spoke) │ (Spoke)     │
└─────────────────────────────────────┘
```

### Multi-Region Cross-Account
```
Central Account (us-east-1)
├── CodePipeline
├── Artifact Stores (us-east-1, us-west-2)
└── KMS Keys (per region)
    ↓
┌─────────────────────────────────────┐
│     Target Account                  │
│ ┌─────────────┐ ┌─────────────────┐ │
│ │ us-east-1   │ │ us-west-2       │ │
│ │ Resources   │ │ Resources       │ │
│ └─────────────┘ └─────────────────┘ │
└─────────────────────────────────────┘
```

### Environment Promotion Pipeline
```
Source → Build → Dev Account → Test Account → Approval → Prod Account
                     ↓             ↓                        ↓
                Auto Deploy   Auto Deploy              Manual Deploy
                     ↓             ↓                        ↓
                 Dev Tests    Integration Tests        Prod Validation
```

### Compliance-Driven Architecture
```
Central Governance Account
├── Pipeline Templates
├── Compliance Policies
├── Audit Trails
└── Approval Workflows
    ↓
┌─────────────────────────────────────┐
│ Business Unit A │ Business Unit B   │
│ ├── Dev Account │ ├── Dev Account   │
│ ├── Test Account│ ├── Test Account  │
│ └── Prod Account│ └── Prod Account  │
└─────────────────────────────────────┘
```

---

## 12. Best Practices for DOP-C02 Exam

### Security
- Use external IDs for additional security in trust relationships
- Implement least privilege access with resource-level permissions
- Enable CloudTrail logging for all cross-account activities
- Use separate KMS keys per environment for additional isolation
- Implement time-based and IP-based access controls

### Operational Excellence
- Use consistent naming conventions across accounts
- Implement comprehensive monitoring and alerting
- Create runbooks for common troubleshooting scenarios
- Use Infrastructure as Code for all cross-account resources
- Implement automated testing of cross-account permissions

### Reliability
- Design for partial failure scenarios
- Implement retry logic for transient failures
- Use multiple availability zones and regions
- Create backup and recovery procedures
- Test disaster recovery scenarios regularly

### Performance
- Use regional artifact stores for better performance
- Implement parallel deployments where appropriate
- Optimize artifact sizes and transfer methods
- Monitor deployment times and optimize bottlenecks
- Use caching strategies for build artifacts

### Cost Optimization
- Use lifecycle policies for artifact storage
- Implement resource tagging for cost allocation
- Monitor cross-account data transfer costs
- Use appropriate storage classes for artifacts
- Regular cleanup of unused resources

---

## 13. Comparison with Alternative Approaches

### Cross-Account Pipeline vs Account-Per-Pipeline
| Feature | Cross-Account Pipeline | Account-Per-Pipeline |
|---------|----------------------|---------------------|
| Complexity | Higher setup, simpler operation | Lower setup, complex coordination |
| Security | Centralized control | Distributed control |
| Cost | Lower (shared resources) | Higher (duplicated resources) |
| Governance | Easier to enforce | Harder to standardize |
| Scalability | Better for many environments | Better for independent teams |

### Cross-Account vs Cross-Region
| Feature | Cross-Account | Cross-Region |
|---------|---------------|--------------|
| Isolation | Account boundaries | Regional boundaries |
| Compliance | Better for regulatory | Better for DR/availability |
| Latency | Network dependent | Geographic dependent |
| Cost | Data transfer charges | Regional pricing differences |

---

## 14. Exam Tips

### What to Remember
- **Trust relationships** are one-directional: only the *target* role carries a trust policy allowing the central account to assume it; the central side grants `sts:AssumeRole` via an *identity* policy (no trust policy needed there)
- **External IDs** provide additional security for role assumption
- **S3 bucket policies** must allow cross-account access for artifacts
- **KMS key policies** must grant decrypt permissions to target accounts
- **RoleArn parameter** in pipeline actions enables cross-account deployment
- **Artifact stores** can be regional for multi-region deployments
- **IAM permissions** need both assume role and target resource permissions

### Common Traps
- Forgetting to configure S3 bucket policy for cross-account access
- Not setting up KMS key policy for cross-account decryption
- Missing IAM permissions for PassRole in target accounts
- Incorrect external ID configuration between pipeline and trust policy
- Not handling partial failures in parallel cross-account deployments
- Forgetting regional artifact stores for multi-region deployments

### Scenario-Based Questions
- Focus on IAM role setup and trust relationships
- Understand artifact sharing and encryption requirements
- Know troubleshooting steps for cross-account failures
- Understand security best practices for cross-account access
- Know when to use cross-account vs other deployment patterns
- Understand compliance and governance implications

### Key Integration Points
- **IAM** - Cross-account roles and trust relationships
- **S3** - Artifact storage with cross-account access
- **KMS** - Cross-account encryption key management
- **CloudFormation** - Cross-account stack deployments
- **CloudWatch** - Cross-account monitoring and logging
- **EventBridge** - Cross-account event notifications

---

## 15. Quick Reference Cheat Sheet

### Essential IAM Actions
```
# Central Account
sts:AssumeRole (to target accounts)
s3:GetObject, s3:PutObject (artifact bucket)
kms:Decrypt, kms:GenerateDataKey (KMS key)

# Target Account
cloudformation:* (deployment actions)
s3:GetObject (artifact bucket)
kms:Decrypt (KMS key)
iam:PassRole (CloudFormation execution role)
```

### Trust Policy Template
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::CENTRAL-ACCOUNT:role/CodePipelineServiceRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UNIQUE-EXTERNAL-ID"
        }
      }
    }
  ]
}
```

### Pipeline Action Configuration
```yaml
RoleArn: arn:aws:iam::TARGET-ACCOUNT:role/CrossAccountDeploymentRole
Configuration:
  RoleArn: arn:aws:iam::TARGET-ACCOUNT:role/CloudFormationExecutionRole
```

### Troubleshooting Checklist
```
□ Trust policy allows central account
□ External ID matches
□ S3 bucket policy allows cross-account access
□ KMS key policy allows cross-account decrypt
□ Target role has necessary permissions
□ CloudFormation execution role exists
□ Network connectivity between accounts
```

---

## 16. Reference Links

### AWS Official Documentation
- [Cross-Account Pipeline Tutorial](https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create-cross-account.html)
- [Cross-Account IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
- [S3 Cross-Account Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-walkthroughs-managing-access-example2.html)
- [KMS Cross-Account Access](https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-modifying-external-accounts.html)
- [CodePipeline Security](https://docs.aws.amazon.com/codepipeline/latest/userguide/security.html)

### Best Practices Guides
- [AWS Multi-Account Strategy](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)
- [DevOps on AWS](https://docs.aws.amazon.com/whitepapers/latest/practicing-continuous-integration-continuous-delivery/practicing-continuous-integration-continuous-delivery.html)

### Workshops
- [Cross-Account CI/CD Workshop](https://catalog.workshops.aws/cross-account-cicd/en-US)
- [Multi-Account DevOps](https://catalog.workshops.aws/multi-account-devops/en-US)

---

## 17. Summary

AWS CodePipeline Cross-Account Deployments are essential for enterprise-scale CI/CD and are heavily tested in the DOP-C02 exam. Key areas to master:

1. **Cross-account IAM setup** (trust relationships, external IDs, least privilege)
2. **Artifact sharing** (S3 bucket policies, KMS key policies, encryption)
3. **Pipeline configuration** (RoleArn parameters, cross-account actions)
4. **Security best practices** (audit trails, compliance, access controls)
5. **Monitoring and troubleshooting** (CloudWatch, EventBridge, failure handling)
6. **Architecture patterns** (hub-and-spoke, multi-region, environment promotion)
7. **Integration patterns** (CloudFormation, multi-account strategies)
8. **Operational excellence** (automation, testing, documentation)
9. **Performance optimization** (regional stores, parallel deployments)
10. **Cost management** (lifecycle policies, resource optimization)

Understanding these concepts with hands-on practice will ensure success on cross-account CodePipeline questions in the DOP-C02 exam. Cross-account deployments represent advanced DevOps practices and demonstrate enterprise-level AWS expertise.