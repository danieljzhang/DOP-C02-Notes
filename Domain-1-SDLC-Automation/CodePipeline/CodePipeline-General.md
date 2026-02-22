# AWS CodePipeline - DOP-C02 Exam Notes

## 1. Overview

**AWS CodePipeline** is a fully managed continuous delivery service that automates the build, test, and deploy phases of your release process every time there is a code change.

### Key Characteristics
- **Fully managed** - No servers to provision or manage
- **Automated** - Orchestrates entire release process
- **Flexible** - Integrates with AWS and third-party tools
- **Fast** - Rapid delivery with parallel execution
- **Configurable** - Custom actions and approval gates
- **Visual** - Pipeline visualization in console

### What Problem Does It Solve?
- Eliminates manual release processes
- Provides consistent deployment workflow
- Enables rapid and reliable releases
- Integrates multiple tools into single workflow
- Tracks release history and status
- Enables continuous delivery practices

---

## 2. Core Concepts

### Pipeline
- Workflow that describes release process
- Contains stages executed in sequence
- Defined in JSON or created via console
- Has unique name within region
- Can be started manually or automatically

### Stage
- Logical grouping of actions
- Executed sequentially in pipeline
- Contains one or more actions
- Examples: Source, Build, Test, Deploy, Approval

### Action
- Task performed on artifacts
- Executed within a stage
- Can run in parallel within stage
- Types: Source, Build, Test, Deploy, Approval, Invoke
- Has input/output artifacts

### Artifact
- Files worked on by actions
- Stored in S3 artifact store
- Passed between actions
- Versioned automatically
- Encrypted by default

### Transition
- Connection between stages
- Can be enabled or disabled
- Useful for testing or maintenance
- Prevents automatic progression

### Execution
- Single run through pipeline
- Has unique execution ID
- Tracks status of each action
- Can be stopped or retried

---

## 3. Pipeline Structure

### Basic Pipeline Structure
```json
{
  "pipeline": {
    "name": "MyPipeline",
    "roleArn": "arn:aws:iam::123456789012:role/CodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "configuration": {
              "RepositoryName": "MyRepo",
              "BranchName": "main"
            },
            "outputArtifacts": [
              {"name": "SourceOutput"}
            ]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAction",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "MyBuildProject"
            },
            "inputArtifacts": [
              {"name": "SourceOutput"}
            ],
            "outputArtifacts": [
              {"name": "BuildOutput"}
            ]
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "DeployAction",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeploy",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "MyApp",
              "DeploymentGroupName": "Production"
            },
            "inputArtifacts": [
              {"name": "BuildOutput"}
            ]
          }
        ]
      }
    ]
  }
}
```

---

## 4. Action Types

### Source Actions
- **CodeCommit** - AWS Git repository
- **S3** - S3 bucket as source
- **GitHub** - GitHub repository (v1 and v2)
- **Bitbucket** - Bitbucket repository
- **ECR** - Container image source
- **GitHub Enterprise** - Self-hosted GitHub

### Build Actions
- **CodeBuild** - AWS build service
- **Jenkins** - Jenkins build server
- **CloudBees** - CloudBees CI/CD
- **TeamCity** - JetBrains TeamCity

### Test Actions
- **CodeBuild** - Run tests in CodeBuild
- **AWS Device Farm** - Mobile app testing
- **Jenkins** - Jenkins test execution
- **Third-party** - Custom test providers

### Deploy Actions
- **CodeDeploy** - Deploy to EC2, Lambda, ECS
- **CloudFormation** - Infrastructure deployment
- **ECS** - Direct ECS deployment
- **Elastic Beanstalk** - Beanstalk deployment
- **S3** - Deploy to S3 bucket
- **Service Catalog** - Service Catalog products

### Approval Actions
- **Manual Approval** - Human approval gate
- **SNS notification** - Alert approvers
- **Custom URL** - Link to external review

### Invoke Actions
- **Lambda** - Invoke Lambda function
- **Step Functions** - Start Step Functions execution

---

## 5. Source Providers

### CodeCommit
- Native AWS integration
- Automatic trigger on commit
- Branch and tag filtering
- CloudWatch Events for triggering

### GitHub (Version 2)
- Recommended GitHub integration
- Uses GitHub App connection
- Webhook-based triggering
- Supports GitHub Enterprise Cloud

### GitHub (Version 1)
- Legacy OAuth token integration
- Being deprecated
- Migrate to Version 2

### S3
- Trigger on object upload
- Versioning required
- CloudWatch Events or polling
- Useful for pre-built artifacts

### ECR
- Container image as source
- Trigger on image push
- Tag-based filtering
- Useful for container deployments

### Bitbucket
- Bitbucket Cloud integration
- Webhook-based triggering
- OAuth connection

---

## 6. Artifact Management

### Artifact Store
- S3 bucket for storing artifacts
- One per region
- Encrypted with KMS
- Versioned automatically

### Artifact Types
- **Input artifacts** - Consumed by action
- **Output artifacts** - Produced by action
- Can have multiple per action

### Artifact Naming
- Unique names within pipeline
- Referenced by name in actions
- Example: SourceOutput, BuildOutput

### Cross-Region Artifacts
- Replicate artifacts to multiple regions
- Required for cross-region deployments
- Separate artifact store per region

### Artifact Encryption
- Default: AWS managed KMS key
- Custom: Customer managed KMS key
- Encrypted at rest in S3

---

## 7. Variables

### Pipeline Variables
- Available throughout pipeline execution
- System-generated or custom
- Referenced with `#{variable}` syntax

### System Variables
- `#{codepipeline.PipelineExecutionId}` - Unique execution ID
- `#{codepipeline.PipelineName}` - Pipeline name
- `#{codepipeline.PipelineVersion}` - Pipeline version

### Action Variables
- Output from actions
- Namespace format: `#{Namespace.Variable}`
- Example: `#{SourceVariables.CommitId}`

### Common Source Variables
- **CodeCommit**: CommitId, CommitMessage, AuthorDate, BranchName
- **GitHub**: CommitId, CommitMessage, CommitUrl, BranchName
- **S3**: ETag, VersionId

### Using Variables
```json
{
  "name": "BuildAction",
  "configuration": {
    "ProjectName": "MyProject",
    "EnvironmentVariables": "[{\"name\":\"COMMIT_ID\",\"value\":\"#{SourceVariables.CommitId}\",\"type\":\"PLAINTEXT\"}]"
  },
  "namespace": "BuildVariables"
}
```

---

## 8. Manual Approval Actions

### Configuration
```json
{
  "name": "ManualApproval",
  "actionTypeId": {
    "category": "Approval",
    "owner": "AWS",
    "provider": "Manual",
    "version": "1"
  },
  "configuration": {
    "NotificationArn": "arn:aws:sns:us-east-1:123456789012:ApprovalTopic",
    "CustomData": "Please review and approve deployment to production",
    "ExternalEntityLink": "https://example.com/review"
  }
}
```

### Approval Process
1. Pipeline reaches approval action
2. SNS notification sent (if configured)
3. Approver reviews via console or API
4. Approver approves or rejects
5. Pipeline continues or stops

### IAM Permissions for Approval
```json
{
  "Effect": "Allow",
  "Action": [
    "codepipeline:GetPipelineState",
    "codepipeline:PutApprovalResult"
  ],
  "Resource": "arn:aws:codepipeline:us-east-1:123456789012:MyPipeline/Production/ManualApproval"
}
```

---

## 9. Parallel Actions

### Configuration
- Multiple actions in same stage
- Execute simultaneously
- Improve pipeline speed
- Share input artifacts

### Example: Parallel Deployments
```json
{
  "name": "Deploy",
  "actions": [
    {
      "name": "DeployToUSEast",
      "runOrder": 1,
      "actionTypeId": {"category": "Deploy", "provider": "CodeDeploy"},
      "configuration": {"ApplicationName": "MyApp-USEast"}
    },
    {
      "name": "DeployToUSWest",
      "runOrder": 1,
      "actionTypeId": {"category": "Deploy", "provider": "CodeDeploy"},
      "configuration": {"ApplicationName": "MyApp-USWest"}
    }
  ]
}
```

### Run Order
- Actions with same runOrder execute in parallel
- Lower runOrder executes first
- Default runOrder is 1

---

## 10. Cross-Region Deployments

### Requirements
- Artifact store in each region
- Replication configuration
- Service role with cross-region permissions

### Configuration
```json
{
  "artifactStores": {
    "us-east-1": {
      "type": "S3",
      "location": "pipeline-artifacts-us-east-1"
    },
    "us-west-2": {
      "type": "S3",
      "location": "pipeline-artifacts-us-west-2"
    }
  }
}
```

### Cross-Region Action
```json
{
  "name": "DeployToWest",
  "region": "us-west-2",
  "actionTypeId": {
    "category": "Deploy",
    "provider": "CodeDeploy"
  },
  "configuration": {
    "ApplicationName": "MyApp"
  }
}
```

---

## 11. IAM Roles & Permissions

### Service Role
- Role assumed by CodePipeline
- Grants permissions to interact with other services
- Required for all pipelines

### Service Role Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-pipeline-artifacts",
        "arn:aws:s3:::my-pipeline-artifacts/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codecommit:GetBranch",
        "codecommit:GetCommit",
        "codecommit:UploadArchive",
        "codecommit:GetUploadArchiveStatus"
      ],
      "Resource": "arn:aws:codecommit:us-east-1:123456789012:MyRepo"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild"
      ],
      "Resource": "arn:aws:codebuild:us-east-1:123456789012:project/MyProject"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetApplication",
        "codedeploy:GetApplicationRevision",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision"
      ],
      "Resource": "*"
    }
  ]
}
```

### CloudFormation Permissions
```json
{
  "Effect": "Allow",
  "Action": [
    "cloudformation:CreateStack",
    "cloudformation:DescribeStacks",
    "cloudformation:DeleteStack",
    "cloudformation:UpdateStack",
    "cloudformation:CreateChangeSet",
    "cloudformation:ExecuteChangeSet",
    "cloudformation:DeleteChangeSet",
    "cloudformation:DescribeChangeSet"
  ],
  "Resource": "*"
}
```

---

## 12. Triggers & Event Detection

### CloudWatch Events (EventBridge)
- Recommended trigger method
- Real-time event detection
- More reliable than polling
- Lower latency

### Polling
- Legacy trigger method
- Periodic checks for changes
- Higher latency
- Not recommended for new pipelines

### Webhook
- Used by GitHub, Bitbucket
- Push-based triggering
- Immediate execution
- Requires connection setup

### Event Pattern Example
```json
{
  "source": ["aws.codecommit"],
  "detail-type": ["CodeCommit Repository State Change"],
  "detail": {
    "event": ["referenceCreated", "referenceUpdated"],
    "referenceType": ["branch"],
    "referenceName": ["main"]
  }
}
```

---

## 13. CloudFormation Integration

### Action Types

#### Create/Update Stack
```json
{
  "name": "CreateStack",
  "actionTypeId": {
    "category": "Deploy",
    "provider": "CloudFormation",
    "version": "1"
  },
  "configuration": {
    "ActionMode": "CREATE_UPDATE",
    "StackName": "MyStack",
    "TemplatePath": "BuildOutput::template.yaml",
    "Capabilities": "CAPABILITY_IAM",
    "RoleArn": "arn:aws:iam::123456789012:role/CloudFormationRole"
  }
}
```

#### Change Set
```json
{
  "name": "CreateChangeSet",
  "configuration": {
    "ActionMode": "CHANGE_SET_REPLACE",
    "StackName": "MyStack",
    "ChangeSetName": "MyChangeSet",
    "TemplatePath": "BuildOutput::template.yaml"
  }
}
```

#### Execute Change Set
```json
{
  "name": "ExecuteChangeSet",
  "configuration": {
    "ActionMode": "CHANGE_SET_EXECUTE",
    "StackName": "MyStack",
    "ChangeSetName": "MyChangeSet"
  }
}
```

### Parameter Overrides
```json
{
  "configuration": {
    "ParameterOverrides": "{\"Environment\":\"Production\",\"InstanceType\":\"t3.large\"}"
  }
}
```

---

## 14. Lambda Integration

### Invoke Lambda Action
```json
{
  "name": "InvokeLambda",
  "actionTypeId": {
    "category": "Invoke",
    "owner": "AWS",
    "provider": "Lambda",
    "version": "1"
  },
  "configuration": {
    "FunctionName": "MyFunction",
    "UserParameters": "{\"key\":\"value\"}"
  }
}
```

### Lambda Function Requirements
- Must complete within 15 minutes
- Return success/failure to CodePipeline
- Use `put_job_success_result` or `put_job_failure_result`

### Lambda Function Example
```python
import boto3

codepipeline = boto3.client('codepipeline')

def lambda_handler(event, context):
    job_id = event['CodePipeline.job']['id']
    
    try:
        # Perform custom logic
        result = perform_validation()
        
        if result:
            codepipeline.put_job_success_result(jobId=job_id)
        else:
            codepipeline.put_job_failure_result(
                jobId=job_id,
                failureDetails={'message': 'Validation failed'}
            )
    except Exception as e:
        codepipeline.put_job_failure_result(
            jobId=job_id,
            failureDetails={'message': str(e)}
        )
```

---

## 15. Monitoring & Notifications

### CloudWatch Events/EventBridge
- Pipeline execution state changes
- Stage execution state changes
- Action execution state changes
- Manual approval needed

### Event Pattern
```json
{
  "source": ["aws.codepipeline"],
  "detail-type": ["CodePipeline Pipeline Execution State Change"],
  "detail": {
    "state": ["FAILED"],
    "pipeline": ["MyPipeline"]
  }
}
```

### SNS Notifications
- Manual approval notifications
- Custom notifications via EventBridge + SNS
- Email, SMS, or other endpoints

### CloudWatch Metrics
- No built-in metrics
- Create custom metrics via EventBridge + Lambda

### CloudTrail Logging
- All API calls logged
- Pipeline creation, updates, deletions
- Execution starts, stops
- Approval decisions

---

## 16. Security Best Practices

### IAM
- Use least privilege service roles
- Separate roles per environment
- Use resource-based policies where possible
- Rotate credentials regularly

### Encryption
- Encrypt artifacts with KMS
- Use customer managed keys for compliance
- Encrypt secrets in Parameter Store/Secrets Manager
- Use HTTPS for all connections

### Secrets Management
- Store secrets in Secrets Manager or Parameter Store
- Reference secrets in actions
- Never hardcode credentials
- Rotate secrets regularly

### Network Security
- Use VPC endpoints for private access
- Restrict S3 bucket access
- Use private subnets for build/deploy
- Enable VPC Flow Logs

### Compliance
- Enable CloudTrail logging
- Use AWS Config for compliance monitoring
- Tag all resources
- Implement approval workflows for production

---

## 17. Common Exam Scenarios

### Scenario 1: Pipeline not triggering on commit
**Possible Causes:**
- CloudWatch Events rule not configured
- Service role missing permissions
- Branch filter not matching
- Polling disabled

**Solution:**
- Verify EventBridge rule exists and is enabled
- Check service role has codecommit permissions
- Verify branch name in source configuration
- Enable CloudWatch Events detection

### Scenario 2: Need manual approval before production
**Solution:**
- Add Manual Approval action before deploy stage
- Configure SNS topic for notifications
- Grant approval permissions to approvers
- Add custom data with review instructions

### Scenario 3: Deploy to multiple regions
**Solution:**
- Configure artifact stores for each region
- Add cross-region actions with region parameter
- Ensure service role has cross-region permissions
- Use parallel actions for simultaneous deployment

### Scenario 4: Pass commit ID to build stage
**Solution:**
- Use source action variables
- Reference with `#{SourceVariables.CommitId}`
- Pass as environment variable to CodeBuild
- Use namespace for action outputs

### Scenario 5: CloudFormation deployment with approval
**Solution:**
- Create change set in first action
- Add manual approval action
- Execute change set in third action
- Review changes before execution

### Scenario 6: Custom validation before deployment
**Solution:**
- Add Lambda invoke action
- Implement validation logic in Lambda
- Return success/failure to CodePipeline
- Use put_job_success_result or put_job_failure_result

### Scenario 7: Pipeline fails intermittently
**Possible Causes:**
- Timeout in action
- Resource limits exceeded
- Network connectivity issues
- Race conditions

**Solution:**
- Increase action timeout
- Check service quotas
- Verify network configuration
- Add retry logic in custom actions

### Scenario 8: Need to skip stage temporarily
**Solution:**
- Disable transition to stage
- Pipeline stops before stage
- Enable transition when ready
- Useful for maintenance or testing

---

## 18. Troubleshooting Guide

### Pipeline Not Starting

#### Check Event Detection
```bash
# List EventBridge rules
aws events list-rules --name-prefix codepipeline

# Describe rule
aws events describe-rule --name codepipeline-MyPipeline-rule
```

#### Check Service Role
- Verify role exists
- Check trust policy allows CodePipeline
- Verify permissions for source provider

### Action Failures

#### View Action Details
```bash
# Get pipeline state
aws codepipeline get-pipeline-state --name MyPipeline

# Get execution details
aws codepipeline get-pipeline-execution \
  --pipeline-name MyPipeline \
  --pipeline-execution-id execution-id
```

#### Common Issues
- Missing IAM permissions
- Invalid configuration
- Timeout exceeded
- Artifact not found

### Artifact Issues

#### Verify Artifact Store
- Check S3 bucket exists
- Verify bucket policy
- Check KMS key permissions
- Verify versioning enabled (for S3 source)

#### Artifact Not Found
- Verify output artifact name matches input
- Check action completed successfully
- Verify artifact store accessible

---

## 19. Integration Patterns

### Full CI/CD Pipeline
```
CodeCommit → CodeBuild (Build) → CodeBuild (Test) → Manual Approval → CodeDeploy
```

### Multi-Environment Pipeline
```
Source → Build → Deploy Dev → Test → Deploy Staging → Approval → Deploy Prod
```

### Infrastructure + Application Pipeline
```
Source → Build → CloudFormation (Infra) → CodeDeploy (App) → Validation
```

### Container Pipeline
```
CodeCommit → CodeBuild (Build Image) → ECR → ECS Deploy
```

### Lambda Pipeline
```
CodeCommit → CodeBuild (Package) → CloudFormation (SAM) → Lambda Deploy
```

---

## 20. Advanced Features

### Pipeline Execution Modes
- **Superseded** - New execution stops previous (default)
- **Queued** - Executions queue and run sequentially
- **Parallel** - Multiple executions run simultaneously

### Stage Locking
- Prevents concurrent executions in stage
- Useful for deployment stages
- Configured per stage

### Retry Failed Actions
- Retry individual failed actions
- No need to restart entire pipeline
- Useful for transient failures

### Stop and Abandon Execution
- Stop in-progress execution
- Abandon queued execution
- Useful for emergency stops

### Pipeline Variables in Actions
- Dynamic configuration
- Environment-specific values
- Commit metadata propagation

---

## 21. Cost Optimization

### Strategies
- Use appropriate action types (avoid unnecessary steps)
- Optimize build times in CodeBuild
- Clean up old artifacts from S3
- Use S3 lifecycle policies
- Disable unused pipelines
- Use parallel actions to reduce total time

### Pricing Model
- $1 per active pipeline per month
- Free tier: 1 active pipeline per month
- No charge for executions
- Pay for underlying services (CodeBuild, S3, etc.)

---

## 22. Comparison with Other CI/CD Tools

### CodePipeline vs Jenkins
| Feature | CodePipeline | Jenkins |
|---------|--------------|---------|
| Management | Fully managed | Self-managed |
| Cost | Per pipeline | Infrastructure cost |
| Integration | AWS-native | Plugin-based |
| Scalability | Automatic | Manual |

### CodePipeline vs GitHub Actions
| Feature | CodePipeline | GitHub Actions |
|---------|--------------|----------------|
| Platform | AWS | GitHub |
| Integration | AWS services | Broad ecosystem |
| Cost | Per pipeline | Per minute |
| Flexibility | Structured stages | Flexible workflows |

### CodePipeline vs GitLab CI
| Feature | CodePipeline | GitLab CI |
|---------|--------------|-----------|
| Hosting | AWS | GitLab/Self-hosted |
| Configuration | JSON/Console | YAML in repo |
| Integration | AWS-focused | Git-focused |

---

## 23. CLI Commands Reference

### Pipeline Operations
```bash
# Create pipeline
aws codepipeline create-pipeline --cli-input-json file://pipeline.json

# Get pipeline
aws codepipeline get-pipeline --name MyPipeline

# Update pipeline
aws codepipeline update-pipeline --cli-input-json file://pipeline.json

# Delete pipeline
aws codepipeline delete-pipeline --name MyPipeline

# List pipelines
aws codepipeline list-pipelines
```

### Execution Operations
```bash
# Start pipeline execution
aws codepipeline start-pipeline-execution --name MyPipeline

# Get pipeline state
aws codepipeline get-pipeline-state --name MyPipeline

# Stop pipeline execution
aws codepipeline stop-pipeline-execution \
  --pipeline-name MyPipeline \
  --pipeline-execution-id execution-id \
  --abandon

# Retry stage execution
aws codepipeline retry-stage-execution \
  --pipeline-name MyPipeline \
  --stage-name Build \
  --pipeline-execution-id execution-id \
  --retry-mode FAILED_ACTIONS
```

### Approval Operations
```bash
# Put approval result
aws codepipeline put-approval-result \
  --pipeline-name MyPipeline \
  --stage-name Production \
  --action-name ManualApproval \
  --result status=Approved,summary="Looks good" \
  --token token-from-notification
```

### Transition Operations
```bash
# Disable stage transition
aws codepipeline disable-stage-transition \
  --pipeline-name MyPipeline \
  --stage-name Production \
  --transition-type Inbound \
  --reason "Maintenance window"

# Enable stage transition
aws codepipeline enable-stage-transition \
  --pipeline-name MyPipeline \
  --stage-name Production \
  --transition-type Inbound
```

---

## 24. Exam Tips

### What to Remember
- **Pipeline structure**: Stages → Actions → Artifacts
- **Action types**: Source, Build, Test, Deploy, Approval, Invoke
- **Triggers**: CloudWatch Events (recommended) vs Polling
- **Variables**: System variables and action output variables
- **Parallel actions**: Same runOrder executes simultaneously
- **Cross-region**: Requires artifact store per region
- **Manual approval**: SNS notification + IAM permissions
- **Service role**: Required for pipeline to access AWS services
- **Artifact store**: S3 bucket, encrypted with KMS

### Common Traps
- Polling is legacy, use CloudWatch Events
- GitHub v1 is deprecated, use v2
- Cross-region requires separate artifact stores
- Manual approval requires token for API approval
- Lambda actions must call put_job_success/failure
- Artifact names must match between actions

### Scenario-Based Questions
- Focus on pipeline structure and action sequencing
- Understand manual approval workflows
- Know cross-region deployment requirements
- Understand variable usage and propagation
- Know CloudFormation integration patterns
- Understand parallel vs sequential execution

### Integration Questions
- CodeCommit/GitHub → CodeBuild → CodeDeploy
- CloudFormation change sets with approval
- Lambda for custom validation
- EventBridge for notifications
- Cross-account deployments

---

## 25. Quick Reference Cheat Sheet

### Essential IAM Permissions (Service Role)
```
s3:GetObject, s3:PutObject
codecommit:GetBranch, codecommit:GetCommit
codebuild:StartBuild, codebuild:BatchGetBuilds
codedeploy:CreateDeployment, codedeploy:GetDeployment
cloudformation:CreateStack, cloudformation:DescribeStacks
lambda:InvokeFunction
```

### Variable Syntax
```
#{codepipeline.PipelineExecutionId}
#{SourceVariables.CommitId}
#{BuildVariables.OutputVariable}
```

### Action Run Order
- Lower number executes first
- Same number executes in parallel
- Default is 1

### Artifact Reference
```json
"inputArtifacts": [{"name": "SourceOutput"}]
"outputArtifacts": [{"name": "BuildOutput"}]
```

---

## 26. Reference Links

### AWS Official Documentation
- [CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)
- [Pipeline Structure Reference](https://docs.aws.amazon.com/codepipeline/latest/userguide/reference-pipeline-structure.html)
- [Action Structure Reference](https://docs.aws.amazon.com/codepipeline/latest/userguide/action-reference.html)
- [Variables Reference](https://docs.aws.amazon.com/codepipeline/latest/userguide/reference-variables.html)
- [IAM Permissions](https://docs.aws.amazon.com/codepipeline/latest/userguide/security-iam.html)
- [Troubleshooting](https://docs.aws.amazon.com/codepipeline/latest/userguide/troubleshooting.html)

---

## 27. Summary

AWS CodePipeline is the orchestration service for CI/CD workflows and is heavily tested in the DOP-C02 exam. Key areas to master:

1. **Pipeline structure** (stages, actions, artifacts)
2. **Action types** and their configurations
3. **Variables** (system and action output variables)
4. **Manual approval** workflows
5. **Cross-region deployments**
6. **CloudFormation integration** (change sets, parameter overrides)
7. **Lambda integration** for custom logic
8. **IAM roles and permissions**
9. **Triggers** (CloudWatch Events vs polling)
10. **Monitoring** with EventBridge and CloudTrail

Understanding these concepts with hands-on practice will ensure success on CodePipeline-related questions in the DOP-C02 exam.
