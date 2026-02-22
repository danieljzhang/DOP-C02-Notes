# AWS CodeDeploy - DOP-C02 Exam Notes

## 1. Overview

**AWS CodeDeploy** is a fully managed deployment service that automates software deployments to various compute services including EC2, Fargate, Lambda, and on-premises servers.

### Key Characteristics
- **Fully managed** - No deployment infrastructure to maintain
- **Automated** - Handles deployment complexity and rollback
- **Flexible** - Multiple deployment strategies (in-place, blue/green)
- **Platform agnostic** - Works with any application
- **Integrated** - Works with CodePipeline, GitHub, S3
- **Scalable** - Deploy to one or thousands of instances

### What Problem Does It Solve?
- Eliminates manual deployment errors
- Provides consistent deployment process
- Enables automated rollback on failure
- Supports zero-downtime deployments
- Tracks deployment history and status
- Enables gradual rollouts with traffic shifting

---

## 2. Core Concepts

### Application
- Logical container for deployment components
- Unique name within AWS account and region
- Contains deployment groups and revisions
- Compute platform: EC2/On-Premises, Lambda, or ECS

### Deployment Group
- Set of instances or targets for deployment
- Defines deployment configuration
- Specifies deployment type (in-place or blue/green)
- Contains tags, Auto Scaling groups, or ECS services
- One application can have multiple deployment groups

### Deployment Configuration
- Rules for deployment execution
- Defines success/failure criteria
- Controls deployment speed
- Predefined or custom configurations

### Revision
- Deployable content (application files + AppSpec file)
- Stored in S3 or GitHub
- Uniquely identified by location
- Contains AppSpec file defining deployment actions

### AppSpec File
- Application specification file
- Defines deployment actions and lifecycle hooks
- Format: YAML or JSON
- Different structure for EC2, Lambda, and ECS
- Must be named `appspec.yml` or `appspec.yaml`

### Deployment
- Process of installing revision to deployment group
- Tracked with deployment ID
- Has status: Created, Queued, InProgress, Succeeded, Failed, Stopped
- Can be rolled back automatically or manually

### CodeDeploy Agent
- Software installed on EC2/on-premises instances
- Communicates with CodeDeploy service
- Downloads and installs revisions
- Reports deployment status
- Required for EC2/On-Premises deployments

---

## 3. Compute Platforms

### EC2/On-Premises
- Deploy to EC2 instances or on-premises servers
- Requires CodeDeploy agent
- Supports in-place and blue/green deployments
- Uses tags or Auto Scaling groups to identify instances

### AWS Lambda
- Deploy Lambda function versions
- Shift traffic between versions
- Supports canary, linear, and all-at-once deployments
- No agent required
- Integrates with Lambda aliases

### Amazon ECS
- Deploy ECS services (Fargate or EC2)
- Blue/green deployments only
- Traffic shifting via Application Load Balancer
- No agent required
- Updates task definitions

---

## 4. Deployment Types

### In-Place Deployment (Rolling Update)

#### Characteristics
- Application stopped on each instance
- Latest revision installed
- Instance briefly out of service
- Same instances reused
- Capacity reduced during deployment

#### Use Cases
- Cost-sensitive deployments
- Simple applications
- Non-critical environments
- When blue/green not feasible

#### Supported Platforms
- EC2/On-Premises only
- NOT supported for Lambda or ECS

#### Process
1. Stop application on instance
2. Install new revision
3. Start application
4. Validate deployment
5. Move to next instance(s)

### Blue/Green Deployment

#### Characteristics
- New instances (green) provisioned
- Traffic shifted from old (blue) to new (green)
- Old instances kept for rollback
- Zero downtime
- Full capacity maintained

#### Use Cases
- Production deployments
- Zero-downtime requirements
- Easy rollback needed
- Critical applications

#### Supported Platforms
- EC2/On-Premises
- Lambda
- ECS

#### Process
1. Provision new instances/environment
2. Deploy application to new environment
3. Test new environment
4. Shift traffic to new environment
5. Keep old environment for rollback
6. Terminate old environment (optional)

---

## 5. Deployment Configurations

### Predefined Configurations

#### EC2/On-Premises
- **CodeDeployDefault.OneAtATime** - Deploy to one instance at a time
- **CodeDeployDefault.HalfAtATime** - Deploy to half of instances at once
- **CodeDeployDefault.AllAtOnce** - Deploy to all instances simultaneously

#### Lambda
- **CodeDeployDefault.LambdaCanary10Percent5Minutes** - 10% traffic for 5 min, then 100%
- **CodeDeployDefault.LambdaCanary10Percent10Minutes** - 10% traffic for 10 min, then 100%
- **CodeDeployDefault.LambdaCanary10Percent15Minutes** - 10% traffic for 15 min, then 100%
- **CodeDeployDefault.LambdaCanary10Percent30Minutes** - 10% traffic for 30 min, then 100%
- **CodeDeployDefault.LambdaLinear10PercentEvery1Minute** - Add 10% every 1 min
- **CodeDeployDefault.LambdaLinear10PercentEvery2Minutes** - Add 10% every 2 min
- **CodeDeployDefault.LambdaLinear10PercentEvery3Minutes** - Add 10% every 3 min
- **CodeDeployDefault.LambdaLinear10PercentEvery10Minutes** - Add 10% every 10 min
- **CodeDeployDefault.LambdaAllAtOnce** - Immediate 100% traffic shift

#### ECS
- **CodeDeployDefault.ECSCanary10Percent5Minutes** - 10% traffic for 5 min, then 100%
- **CodeDeployDefault.ECSCanary10Percent15Minutes** - 10% traffic for 15 min, then 100%
- **CodeDeployDefault.ECSLinear10PercentEvery1Minute** - Add 10% every 1 min
- **CodeDeployDefault.ECSLinear10PercentEvery3Minutes** - Add 10% every 3 min
- **CodeDeployDefault.ECSAllAtOnce** - Immediate 100% traffic shift

### Custom Configurations
- Define minimum healthy instances/hosts
- Specify as percentage or count
- Example: 75% minimum healthy = deploy to 25% at a time

---

## 6. AppSpec File Structure

### EC2/On-Premises AppSpec

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
  - source: /config/
    destination: /etc/myapp

permissions:
  - object: /var/www/html
    owner: apache
    group: apache
    mode: 755
    type:
      - directory
  - object: /var/www/html/*
    owner: apache
    group: apache
    mode: 644
    type:
      - file

hooks:
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
  
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  
  AfterInstall:
    - location: scripts/configure_app.sh
      timeout: 300
      runas: root
  
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
  
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 300
      runas: root
```

### Lambda AppSpec

```yaml
version: 0.0
Resources:
  - MyFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: myLambdaFunction
        Alias: live
        CurrentVersion: 1
        TargetVersion: 2

Hooks:
  - BeforeAllowTraffic: BeforeAllowTrafficHookFunctionName
  - AfterAllowTraffic: AfterAllowTrafficHookFunctionName
```

### ECS AppSpec

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789012:task-definition/my-task:2"
        LoadBalancerInfo:
          ContainerName: "my-container"
          ContainerPort: 80
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "BeforeInstallHookFunctionName"
  - AfterInstall: "AfterInstallHookFunctionName"
  - AfterAllowTestTraffic: "AfterAllowTestTrafficHookFunctionName"
  - BeforeAllowTraffic: "BeforeAllowTrafficHookFunctionName"
  - AfterAllowTraffic: "AfterAllowTrafficHookFunctionName"
```

---

## 7. Lifecycle Event Hooks

### EC2/On-Premises Lifecycle Events (In Order)

1. **ApplicationStop** - Stop current application version
2. **DownloadBundle** - Download revision (automatic, no script)
3. **BeforeInstall** - Pre-installation tasks (backup, decrypt)
4. **Install** - Copy files to destination (automatic, no script)
5. **AfterInstall** - Post-installation tasks (configuration, permissions)
6. **ApplicationStart** - Start application
7. **ValidateService** - Verify deployment success

### Lambda Lifecycle Events

1. **BeforeAllowTraffic** - Run tasks before traffic shift (tests, validation)
2. **AllowTraffic** - Traffic shift (automatic)
3. **AfterAllowTraffic** - Run tasks after traffic shift (monitoring, validation)

### ECS Lifecycle Events

1. **BeforeInstall** - Before replacement task set created
2. **Install** - Create replacement task set (automatic)
3. **AfterInstall** - After replacement task set created
4. **AfterAllowTestTraffic** - After test traffic routed
5. **BeforeAllowTraffic** - Before production traffic shift
6. **AllowTraffic** - Production traffic shift (automatic)
7. **AfterAllowTraffic** - After production traffic shift

---

## 8. IAM Roles & Permissions

### Service Role (CodeDeploy Service Role)
- Role assumed by CodeDeploy service
- Grants permissions to interact with other AWS services
- Required for all deployments

#### Required Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:TerminateInstances",
        "tag:GetResources",
        "autoscaling:CompleteLifecycleAction",
        "autoscaling:DeleteLifecycleHook",
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeLifecycleHooks",
        "autoscaling:PutLifecycleHook",
        "autoscaling:RecordLifecycleActionHeartbeat"
      ],
      "Resource": "*"
    }
  ]
}
```

#### For Lambda Deployments
```json
{
  "Effect": "Allow",
  "Action": [
    "lambda:InvokeFunction",
    "lambda:GetFunction",
    "lambda:UpdateAlias"
  ],
  "Resource": "*"
}
```

#### For ECS Deployments
```json
{
  "Effect": "Allow",
  "Action": [
    "ecs:DescribeServices",
    "ecs:CreateTaskSet",
    "ecs:UpdateServicePrimaryTaskSet",
    "ecs:DeleteTaskSet",
    "elasticloadbalancing:DescribeTargetGroups",
    "elasticloadbalancing:DescribeListeners",
    "elasticloadbalancing:ModifyListener",
    "elasticloadbalancing:DescribeRules",
    "elasticloadbalancing:ModifyRule"
  ],
  "Resource": "*"
}
```

### Instance Role (EC2 Instance Profile)
- Role attached to EC2 instances
- Allows CodeDeploy agent to access S3 and other services

#### Required Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-deployment-bucket/*",
        "arn:aws:s3:::aws-codedeploy-us-east-1/*"
      ]
    }
  ]
}
```

---

## 9. Auto Scaling Integration

### Deployment to Auto Scaling Groups
- CodeDeploy integrates with Auto Scaling
- Automatically deploys to new instances
- Handles scaling events during deployment

### Deployment Options

#### Suspend Auto Scaling
- Suspend scaling during deployment
- Prevents new instances during deployment
- Resume after deployment completes

#### Keep Auto Scaling Active
- Allow scaling during deployment
- New instances automatically get latest revision
- CodeDeploy handles deployment to new instances

### Blue/Green with Auto Scaling
- Create new Auto Scaling group (green)
- Deploy to new group
- Shift traffic via load balancer
- Terminate old group (blue)

---

## 10. Load Balancer Integration

### Supported Load Balancers
- Application Load Balancer (ALB)
- Network Load Balancer (NLB)
- Classic Load Balancer (CLB)

### Blue/Green Deployment Process
1. Register new instances with load balancer
2. Deploy application to new instances
3. Run validation tests
4. Shift traffic from old to new instances
5. Deregister old instances
6. Terminate old instances (optional)

### Connection Draining
- Allows in-flight requests to complete
- Configurable timeout period
- Prevents abrupt connection termination

---

## 11. Rollback Mechanisms

### Automatic Rollback
- Triggered by deployment failure
- Triggered by CloudWatch alarms
- Redeploys last known good revision
- Configure in deployment group settings

### Automatic Rollback Triggers
- **Deployment fails** - Any instance fails deployment
- **Alarm thresholds met** - CloudWatch alarm triggered
- **Both** - Either condition triggers rollback

### Manual Rollback
- Stop current deployment
- Create new deployment with previous revision
- Use "Redeploy" option in console

### Rollback for Blue/Green
- Keep blue environment for quick rollback
- Shift traffic back to blue environment
- No redeployment needed

---

## 12. Monitoring & Logging

### CloudWatch Integration
- Deployment events sent to CloudWatch Events/EventBridge
- Create alarms for automatic rollback
- Monitor deployment metrics

### CloudWatch Metrics
- No built-in metrics for CodeDeploy
- Create custom metrics via EventBridge + Lambda

### CloudWatch Alarms for Rollback
```json
{
  "AlarmName": "HighErrorRate",
  "MetricName": "5XXError",
  "Namespace": "AWS/ApplicationELB",
  "Statistic": "Sum",
  "Period": 60,
  "EvaluationPeriods": 2,
  "Threshold": 10,
  "ComparisonOperator": "GreaterThanThreshold"
}
```

### CloudWatch Logs
- Deployment logs from lifecycle hooks
- Agent logs on EC2 instances
- Location: `/var/log/aws/codedeploy-agent/`

### AWS CloudTrail
- Logs all API calls to CodeDeploy
- Tracks deployment creation, updates, deletions
- Audit trail for compliance

### EventBridge Events
- Deployment state changes
- Instance state changes
- Deployment group changes
- Trigger Lambda, SNS, or other targets

---

## 13. On-Premises Deployments

### Requirements
- CodeDeploy agent installed
- Instance registered with CodeDeploy
- IAM user credentials configured
- Network connectivity to AWS

### Instance Registration
```bash
aws deploy register-on-premises-instance \
  --instance-name my-server \
  --iam-user-arn arn:aws:iam::123456789012:user/CodeDeployUser
```

### Agent Installation
```bash
# Download agent
wget https://aws-codedeploy-us-east-1.s3.amazonaws.com/latest/install

# Install agent
chmod +x ./install
sudo ./install auto

# Configure credentials
sudo aws configure --profile codedeploy
```

### Tagging On-Premises Instances
```bash
aws deploy add-tags-to-on-premises-instances \
  --instance-names my-server \
  --tags Key=Environment,Value=Production
```

---

## 14. Security Best Practices

### IAM
- Use least privilege for service roles
- Separate roles for different environments
- Use instance profiles for EC2 instances
- Rotate IAM credentials regularly

### Encryption
- Encrypt revisions in S3 with KMS
- Use HTTPS for agent communication
- Encrypt sensitive data in AppSpec hooks

### Network Security
- Use VPC endpoints for private deployments
- Restrict security group rules
- Use private subnets with NAT for outbound

### Compliance
- Enable CloudTrail logging
- Use AWS Config for compliance monitoring
- Tag resources for governance
- Implement approval workflows

---

## 15. Common Exam Scenarios

### Scenario 1: Deployment fails on some instances
**Possible Causes:**
- CodeDeploy agent not running
- Insufficient IAM permissions
- Script errors in lifecycle hooks
- Timeout in lifecycle hooks

**Solution:**
- Check agent status: `sudo service codedeploy-agent status`
- Verify instance role permissions
- Check logs: `/var/log/aws/codedeploy-agent/`
- Increase timeout in AppSpec file

### Scenario 2: Need zero-downtime deployment
**Solution:**
- Use blue/green deployment
- Configure load balancer
- Set appropriate deployment configuration
- Test in staging first

### Scenario 3: Automatic rollback on high error rate
**Solution:**
- Create CloudWatch alarm for error metrics
- Configure automatic rollback in deployment group
- Set alarm as rollback trigger
- Test alarm threshold

### Scenario 4: Deploy Lambda with gradual traffic shift
**Solution:**
- Use Lambda deployment type
- Choose canary or linear deployment configuration
- Configure pre/post traffic hooks for validation
- Use Lambda aliases

### Scenario 5: Deploy to Auto Scaling group
**Solution:**
- Add Auto Scaling group to deployment group
- Choose deployment configuration (OneAtATime, HalfAtATime)
- Decide whether to suspend scaling
- Configure load balancer integration

### Scenario 6: Deployment stuck in pending
**Possible Causes:**
- Agent not installed or not running
- Instance not tagged correctly
- Network connectivity issues
- Service role missing permissions

**Solution:**
- Verify agent installation and status
- Check instance tags match deployment group
- Verify security groups allow outbound HTTPS
- Review service role permissions

### Scenario 7: Need to deploy to on-premises servers
**Solution:**
- Install CodeDeploy agent on servers
- Register instances with CodeDeploy
- Configure IAM user credentials
- Tag instances for deployment targeting
- Create deployment group with on-premises tags

### Scenario 8: Rollback after deployment completes
**Solution:**
- Create new deployment with previous revision
- Use "Redeploy" option in console
- For blue/green: shift traffic back to blue
- Automate with EventBridge + Lambda

---

## 16. Troubleshooting Guide

### Agent Issues

#### Agent Not Running
```bash
# Check status
sudo service codedeploy-agent status

# Start agent
sudo service codedeploy-agent start

# View agent logs
tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

#### Agent Cannot Download Revision
- Check instance role has S3 permissions
- Verify S3 bucket policy
- Check network connectivity
- Verify revision location in S3

### Deployment Failures

#### Script Errors
- Check script syntax
- Verify script permissions (executable)
- Review script logs in CloudWatch or local logs
- Test scripts manually on instance

#### Timeout Errors
- Increase timeout in AppSpec file
- Optimize script performance
- Check for blocking operations
- Review network latency

#### Permission Errors
- Verify instance role permissions
- Check file/directory permissions
- Verify runas user in AppSpec
- Review service role permissions

### Blue/Green Issues

#### Traffic Not Shifting
- Verify load balancer configuration
- Check target group health checks
- Verify security group rules
- Review deployment configuration

#### Old Instances Not Terminating
- Check termination settings in deployment group
- Verify Auto Scaling group configuration
- Review instance protection settings

---

## 17. Integration with CI/CD Pipeline

### CodePipeline Integration
```yaml
- Name: Deploy
  Actions:
    - Name: DeployToProduction
      ActionTypeId:
        Category: Deploy
        Owner: AWS
        Provider: CodeDeploy
        Version: 1
      InputArtifacts:
        - Name: BuildArtifact
      Configuration:
        ApplicationName: MyApp
        DeploymentGroupName: Production
```

### GitHub Integration
- Trigger deployment from GitHub push
- Use GitHub as revision source
- Configure webhook for automatic deployment

### S3 Integration
- Store revisions in S3
- Version revisions with S3 versioning
- Use S3 lifecycle policies for cleanup

---

## 18. Cost Optimization

### Strategies
- Use in-place deployments for non-critical apps
- Terminate blue environment after successful green deployment
- Use appropriate deployment configuration (not AllAtOnce for large fleets)
- Clean up old revisions from S3
- Use spot instances for blue/green (non-production)

### Pricing Model
- No charge for CodeDeploy to EC2/Lambda/ECS
- Charge for on-premises deployments ($0.02 per instance update)
- Pay for underlying resources (EC2, ALB, etc.)

---

## 19. Advanced Features

### Deployment Groups with Multiple Target Types
- Combine EC2 instances and Auto Scaling groups
- Use tags for flexible targeting
- Different configurations per environment

### Custom Deployment Configurations
- Define minimum healthy hosts
- Specify as percentage or count
- Example: 90% minimum = deploy to 10% at a time

### Deployment Triggers
- SNS notifications on deployment events
- Trigger Lambda for custom processing
- Integrate with ChatOps tools

### Blue/Green with Manual Approval
- Pause before traffic shift
- Manual validation of green environment
- Shift traffic after approval

---

## 20. Comparison with Other Deployment Tools

### CodeDeploy vs Elastic Beanstalk
| Feature | CodeDeploy | Elastic Beanstalk |
|---------|------------|-------------------|
| Scope | Deployment only | Full platform (deploy + infrastructure) |
| Flexibility | High | Medium |
| Complexity | Higher | Lower |
| Use Case | Custom deployments | Quick app hosting |

### CodeDeploy vs ECS Rolling Update
| Feature | CodeDeploy | ECS Rolling Update |
|---------|------------|-------------------|
| Traffic Control | Precise (canary, linear) | Basic |
| Rollback | Automatic | Manual |
| Validation Hooks | Yes | No |
| Use Case | Production with validation | Simple updates |

### CodeDeploy vs CloudFormation
| Feature | CodeDeploy | CloudFormation |
|---------|------------|----------------|
| Focus | Application code | Infrastructure |
| Deployment Types | Multiple | Stack updates |
| Rollback | Application-level | Stack-level |
| Use Case | App deployments | Infrastructure as code |

---

## 21. Exam Tips

### What to Remember
- **Deployment types**: In-place (EC2 only) vs Blue/Green (all platforms)
- **Compute platforms**: EC2/On-Premises, Lambda, ECS
- **AppSpec file**: Required, defines deployment actions and hooks
- **Lifecycle hooks**: Different for EC2, Lambda, and ECS
- **Agent**: Required for EC2/On-Premises, not for Lambda/ECS
- **Rollback**: Automatic (on failure/alarm) or manual
- **Deployment configs**: OneAtATime, HalfAtATime, AllAtOnce, Canary, Linear
- **Service role**: Required for CodeDeploy to access AWS resources
- **Instance role**: Required for EC2 to download revisions from S3

### Common Traps
- In-place deployment NOT supported for Lambda/ECS
- Agent required for EC2/On-Premises (often forgotten)
- AppSpec file must be in root of revision
- Blue/Green requires load balancer for EC2
- Automatic rollback requires CloudWatch alarm configuration
- On-premises deployments have per-instance charges

### Scenario-Based Questions
- Focus on deployment type selection (in-place vs blue/green)
- Understand when to use each deployment configuration
- Know rollback mechanisms and triggers
- Understand lifecycle hooks and their order
- Know IAM role requirements (service role vs instance role)
- Understand Auto Scaling integration

### Integration Questions
- CodeCommit/GitHub → CodeBuild → CodeDeploy
- CodePipeline orchestration
- CloudWatch alarms for automatic rollback
- EventBridge for deployment notifications
- Load balancer for blue/green deployments

---

## 22. CLI Commands Reference

### Application Operations
```bash
# Create application
aws deploy create-application --application-name MyApp

# List applications
aws deploy list-applications

# Delete application
aws deploy delete-application --application-name MyApp
```

### Deployment Group Operations
```bash
# Create deployment group
aws deploy create-deployment-group \
  --application-name MyApp \
  --deployment-group-name Production \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployRole \
  --ec2-tag-filters Key=Environment,Value=Production,Type=KEY_AND_VALUE

# List deployment groups
aws deploy list-deployment-groups --application-name MyApp

# Get deployment group details
aws deploy get-deployment-group \
  --application-name MyApp \
  --deployment-group-name Production
```

### Deployment Operations
```bash
# Create deployment
aws deploy create-deployment \
  --application-name MyApp \
  --deployment-group-name Production \
  --s3-location bucket=my-bucket,key=app.zip,bundleType=zip

# List deployments
aws deploy list-deployments --application-name MyApp

# Get deployment details
aws deploy get-deployment --deployment-id d-XXXXXXXXX

# Stop deployment
aws deploy stop-deployment --deployment-id d-XXXXXXXXX --auto-rollback-enabled
```

### Instance Operations
```bash
# List instances in deployment
aws deploy list-deployment-instances --deployment-id d-XXXXXXXXX

# Get instance details
aws deploy get-deployment-instance \
  --deployment-id d-XXXXXXXXX \
  --instance-id i-1234567890abcdef0
```

---

## 23. Architecture Patterns

### Basic CI/CD Pipeline
```
CodeCommit → CodeBuild → CodeDeploy → EC2 Instances
                              ↓
                         Load Balancer
```

### Blue/Green Deployment Pattern
```
CodeDeploy
    ↓
┌───────────────────────────┐
│   Load Balancer           │
└───────┬───────────────┬───┘
        │               │
    Blue Fleet      Green Fleet
    (Current)       (New Version)
        │               │
    Terminate      Production Traffic
```

### Multi-Region Deployment
```
CodePipeline
    ↓
┌───────────────────────────────┐
│   CodeDeploy (us-east-1)      │
│   CodeDeploy (us-west-2)      │
│   CodeDeploy (eu-west-1)      │
└───────────────────────────────┘
```

### Lambda Canary Deployment
```
CodeDeploy
    ↓
Lambda Alias (live)
    ↓
┌─────────────────────┐
│ 90% → Version 1     │
│ 10% → Version 2     │
└─────────────────────┘
    ↓ (gradual shift)
┌─────────────────────┐
│ 100% → Version 2    │
└─────────────────────┘
```

---

## 24. Quick Reference Cheat Sheet

### Deployment Type Selection
- **In-place**: EC2 only, cost-effective, brief downtime
- **Blue/Green**: All platforms, zero downtime, easy rollback

### Lifecycle Hook Order (EC2)
1. ApplicationStop
2. BeforeInstall
3. AfterInstall
4. ApplicationStart
5. ValidateService

### Essential IAM Permissions (Service Role)
```
ec2:DescribeInstances
autoscaling:*
elasticloadbalancing:*
lambda:InvokeFunction (for Lambda)
ecs:* (for ECS)
```

### Essential IAM Permissions (Instance Role)
```
s3:GetObject
s3:ListBucket
```

### Agent Commands
```bash
sudo service codedeploy-agent status
sudo service codedeploy-agent start
sudo service codedeploy-agent stop
tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

---

## 25. Reference Links

### AWS Official Documentation
- [CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html)
- [AppSpec File Reference](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html)
- [Deployment Configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html)
- [IAM Permissions](https://docs.aws.amazon.com/codedeploy/latest/userguide/security-iam.html)
- [Lifecycle Events](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html)
- [Troubleshooting](https://docs.aws.amazon.com/codedeploy/latest/userguide/troubleshooting.html)

---

## 26. Summary

AWS CodeDeploy is essential for automated deployments in AWS and is heavily tested in the DOP-C02 exam. Key areas to master:

1. **Deployment types** (in-place vs blue/green) and when to use each
2. **Compute platforms** (EC2, Lambda, ECS) and their differences
3. **AppSpec file structure** for each platform
4. **Lifecycle hooks** and their execution order
5. **Deployment configurations** (predefined and custom)
6. **IAM roles** (service role vs instance role)
7. **Rollback mechanisms** (automatic and manual)
8. **Auto Scaling and Load Balancer integration**
9. **Monitoring and troubleshooting** with CloudWatch
10. **Agent installation and management** for EC2/On-Premises

Understanding these concepts with hands-on practice will ensure success on CodeDeploy-related questions in the DOP-C02 exam.
