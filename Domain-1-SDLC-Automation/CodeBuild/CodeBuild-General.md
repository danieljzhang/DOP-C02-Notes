# AWS CodeBuild - DOP-C02 Exam Notes

## 1. Overview

**AWS CodeBuild** is a fully managed continuous integration service that compiles source code, runs tests, and produces software packages ready for deployment.

### Key Characteristics
- **Fully managed** - No servers to provision or manage
- **Pay-as-you-go** - Only pay for build minutes used
- **Scalable** - Automatically scales to meet build demand
- **Pre-configured environments** - Supports Java, Python, Node.js, Ruby, Go, Docker, .NET Core, PHP, and more
- **Custom environments** - Use custom Docker images
- **Integrated** - Works seamlessly with CodePipeline, CodeCommit, S3, GitHub, Bitbucket

---

## 2. Core Components

### Build Project
- Configuration that defines how to run a build
- Includes: source location, build environment, build commands, output location

### Build Environment
- Docker container where build runs
- AWS provides managed images or use custom Docker images
- Compute types: `BUILD_GENERAL1_SMALL`, `MEDIUM`, `LARGE`, `2XLARGE`

### Build Specification (buildspec.yml)
- YAML file defining build commands and settings
- Can be in source code root or specified inline
- Defines phases: install, pre_build, build, post_build

### Source Providers
- AWS CodeCommit
- Amazon S3
- GitHub / GitHub Enterprise
- Bitbucket
- No source (for testing)

### Artifacts
- Output files from build process
- Stored in S3 bucket
- Can be encrypted with KMS

---

## 3. Buildspec.yml Structure

### Basic Structure
```yaml
version: 0.2

env:
  variables:
    JAVA_HOME: "/usr/lib/jvm/java-8-openjdk-amd64"
  parameter-store:
    LOGIN_PASSWORD: /myapp/password
  secrets-manager:
    DOCKER_HUB_TOKEN: dockerhub:token

phases:
  install:
    runtime-versions:
      nodejs: 14
    commands:
      - echo "Installing dependencies"
      - npm install
  
  pre_build:
    commands:
      - echo "Running tests"
      - npm test
  
  build:
    commands:
      - echo "Building application"
      - npm run build
  
  post_build:
    commands:
      - echo "Build completed"

artifacts:
  files:
    - '**/*'
  base-directory: build
  name: myapp-$(date +%Y%m%d-%H%M%S)

cache:
  paths:
    - 'node_modules/**/*'
```

### Build Phases
1. **install** - Install packages in build environment
2. **pre_build** - Commands before build (e.g., login to registries, install dependencies)
3. **build** - Actual build commands
4. **post_build** - Commands after build (e.g., packaging, notifications)

### Environment Variables
- **variables** - Plain text variables
- **parameter-store** - Retrieve from AWS Systems Manager Parameter Store
- **secrets-manager** - Retrieve from AWS Secrets Manager
- **exported-variables** - Export variables to later build stages

---

## 4. Build Environments

### Managed Images
- Amazon Linux 2 (AL2)
- Ubuntu
- Windows Server Core 2019

### Compute Types
| Type | vCPU | Memory | Disk Space |
|------|------|--------|------------|
| BUILD_GENERAL1_SMALL | 2 | 3 GB | 64 GB |
| BUILD_GENERAL1_MEDIUM | 4 | 7 GB | 128 GB |
| BUILD_GENERAL1_LARGE | 8 | 15 GB | 128 GB |
| BUILD_GENERAL1_2XLARGE | 72 | 145 GB | 824 GB |

### Custom Docker Images
- Use custom images from ECR or Docker Hub
- Specify in build project configuration
- Must include CodeBuild agent for managed builds

### Privileged Mode
- Required for Docker builds (Docker-in-Docker)
- Allows access to Docker daemon
- Enable in build project settings

---

## 5. VPC Configuration

### When to Use VPC
- Access private resources (RDS, ElastiCache, internal APIs)
- Access resources in private subnets
- Comply with network isolation requirements

### VPC Setup Requirements
- **VPC ID** - The VPC where resources exist
- **Subnets** - Private subnets (typically)
- **Security Groups** - Control inbound/outbound traffic
- **ENI** - CodeBuild creates ENI in your subnet

### Required IAM Permissions for VPC
```
ec2:CreateNetworkInterface
ec2:DescribeNetworkInterfaces
ec2:DeleteNetworkInterface
ec2:DescribeSubnets
ec2:DescribeSecurityGroups
ec2:DescribeDhcpOptions
ec2:DescribeVpcs
ec2:CreateNetworkInterfacePermission
```

### Internet Access from VPC
- **NAT Gateway** - For internet access from private subnet
- **VPC Endpoints** - For AWS service access (S3, DynamoDB, ECR, etc.)
- **No IGW needed** - If only accessing AWS services via endpoints

---

## 6. Caching

### Cache Types

#### S3 Cache
- Stores cache in S3 bucket
- Shared across build projects
- Slower than local cache
- Good for dependencies that don't change often

#### Local Cache
- Stores cache on build instance
- Faster than S3 cache
- Types:
  - **LOCAL_SOURCE_CACHE** - Caches Git metadata
  - **LOCAL_DOCKER_LAYER_CACHE** - Caches Docker layers
  - **LOCAL_CUSTOM_CACHE** - Caches directories specified in buildspec

### Cache Configuration in buildspec.yml
```yaml
cache:
  paths:
    - '/root/.m2/**/*'
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
```

### Best Practices
- Use local cache for Docker builds
- Use S3 cache for large dependencies
- Combine both for optimal performance
- Clear cache if builds behave unexpectedly

---

## 7. Security & IAM

### Service Role
- IAM role that CodeBuild assumes during build
- Grants permissions to access AWS resources
- Required permissions:
  - CloudWatch Logs (create log groups/streams)
  - S3 (read source, write artifacts)
  - ECR (pull/push images)
  - SSM/Secrets Manager (retrieve secrets)
  - VPC (if VPC-enabled)

### Minimal Service Role Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-source-bucket/*",
        "arn:aws:s3:::my-artifact-bucket/*"
      ]
    }
  ]
}
```

### Encryption
- **Artifacts** - Encrypted with S3-managed keys or KMS
- **Environment Variables** - Encrypted at rest
- **Secrets** - Use Parameter Store or Secrets Manager
- **Cache** - Encrypted with S3 encryption

### Best Practices
- Use least privilege IAM roles
- Store secrets in Parameter Store/Secrets Manager
- Enable artifact encryption with KMS
- Use VPC endpoints for private access
- Rotate credentials regularly
- Never hardcode credentials in buildspec

---

## 8. Integration with Other AWS Services

### CodePipeline
- CodeBuild as build/test stage
- Automatic triggering on source changes
- Pass artifacts between stages
- Use pipeline variables in buildspec

### CloudWatch
- **Logs** - Build logs stored in CloudWatch Logs
- **Events** - Trigger on build state changes
- **Metrics** - Build duration, success/failure rates
- **Alarms** - Alert on build failures

### SNS/EventBridge
- Send notifications on build events
- Trigger Lambda functions
- Integrate with third-party tools

### ECR
- Pull base images for builds
- Push built Docker images
- Scan images for vulnerabilities

### S3
- Source code location
- Artifact storage
- Cache storage
- Build logs (alternative to CloudWatch)

### Systems Manager Parameter Store
- Store configuration values
- Store secrets (encrypted)
- Reference in buildspec.yml

### Secrets Manager
- Store database credentials
- Store API keys
- Automatic rotation support

---

## 9. Monitoring & Troubleshooting

### CloudWatch Logs
- Default logging destination
- Log group: `/aws/codebuild/project-name`
- Stream per build execution
- Retention configurable

### CloudWatch Metrics
- **Builds** - Total number of builds
- **Duration** - Build execution time
- **SucceededBuilds** - Successful builds count
- **FailedBuilds** - Failed builds count

### Build Status
- **SUCCEEDED** - Build completed successfully
- **FAILED** - Build failed
- **FAULT** - Build encountered an error
- **STOPPED** - Build was stopped
- **TIMED_OUT** - Build exceeded timeout
- **IN_PROGRESS** - Build is running

### Common Issues & Solutions

#### Build Timeout
- Default: 60 minutes
- Max: 480 minutes (8 hours)
- Increase timeout in project settings

#### Insufficient Permissions
- Check service role permissions
- Verify S3 bucket policies
- Check KMS key policies

#### VPC Connectivity Issues
- Verify security group rules
- Check subnet routing
- Ensure NAT/VPC endpoints configured

#### Docker Build Failures
- Enable privileged mode
- Check Docker image availability
- Verify ECR permissions

#### Cache Not Working
- Verify cache paths in buildspec
- Check S3 bucket permissions
- Clear cache and rebuild

---

## 10. Test Reports

### Overview
- Visualize test results in CodeBuild console
- Support for multiple test frameworks
- Track test trends over time

### Supported Formats
- JUnit XML
- NUnit XML
- Cucumber JSON
- TestNG XML
- Visual Studio TRX

### Configuration in buildspec.yml
```yaml
reports:
  junit-reports:
    files:
      - 'target/surefire-reports/*.xml'
    file-format: 'JUNITXML'
  
  coverage-reports:
    files:
      - 'coverage/cobertura-coverage.xml'
    file-format: 'COBERTURAXML'
```

### Benefits
- Track test pass/fail rates
- Identify flaky tests
- Monitor code coverage
- Historical trend analysis

---

## 11. Advanced Features

### Batch Builds
- Run multiple builds in parallel
- Build matrix (different configurations)
- Useful for testing across environments

### Build Badges
- Embed build status in README
- Public visibility of build status
- Supports CodeCommit, GitHub, Bitbucket

### Webhooks
- Trigger builds on Git events
- Filter by branch, tag, or file path
- Support for pull requests

### Environment Variables from CodePipeline
```yaml
env:
  exported-variables:
    - BUILD_ID
    - BUILD_NUMBER
```

### Secondary Sources
- Include multiple source repositories
- Combine code from different locations
- Useful for monorepo builds

### Secondary Artifacts
- Produce multiple artifact outputs
- Different artifacts for different purposes
- Example: binaries + documentation

---

## 12. Cost Optimization

### Strategies
- Use appropriate compute type (don't over-provision)
- Enable caching to reduce build time
- Use build timeouts to prevent runaway builds
- Clean up old artifacts from S3
- Use spot instances for non-critical builds (via custom environments)

### Pricing Model
- Charged per build minute
- Varies by compute type
- Free tier: 100 build minutes/month (general1.small)

---

## 13. Common Exam Scenarios

### Scenario 1: Build needs to access private RDS database
**Solution:**
- Enable VPC configuration in CodeBuild
- Place build in same VPC/subnet as RDS
- Configure security group to allow traffic
- No NAT Gateway needed for RDS access

### Scenario 2: Build fails with "Access Denied" to S3
**Solution:**
- Check service role has s3:GetObject and s3:PutObject
- Verify S3 bucket policy allows CodeBuild role
- Check KMS key policy if encryption enabled

### Scenario 3: Docker build fails
**Solution:**
- Enable privileged mode in build project
- Verify Docker image is accessible
- Check ECR permissions if using ECR

### Scenario 4: Need to pass variables between build stages
**Solution:**
- Use exported-variables in buildspec
- Store in SSM Parameter Store
- Use CodePipeline variables

### Scenario 5: Build takes too long
**Solution:**
- Enable caching (S3 or local)
- Use larger compute type
- Optimize build commands
- Use Docker layer caching

### Scenario 6: Need to run builds in multiple environments
**Solution:**
- Use batch builds with build matrix
- Define environment variables per configuration
- Parallel execution for faster results

### Scenario 7: Secrets exposed in logs
**Solution:**
- Use Parameter Store or Secrets Manager
- Never echo secrets in build commands
- Use environment variables, not hardcoded values

### Scenario 8: Build needs internet access from VPC
**Solution:**
- Add NAT Gateway to private subnet
- OR use VPC endpoints for AWS services
- Configure route tables appropriately

---

## 14. Best Practices for DOP-C02 Exam

### Security
- Always use least-privileged IAM roles
- Store secrets in Parameter Store or Secrets Manager
- Enable artifact encryption with KMS
- Use VPC endpoints for private resource access
- Never hardcode credentials

### Performance
- Enable caching (S3 or local) for faster builds
- Use appropriate compute type
- Use local Docker layer cache for Docker builds
- Optimize buildspec commands

### Reliability
- Set appropriate timeouts
- Implement retry logic in build commands
- Use CloudWatch alarms for build failures
- Monitor build metrics

### Cost
- Use smallest compute type that meets needs
- Enable caching to reduce build time
- Set timeouts to prevent runaway builds
- Clean up old artifacts

### Monitoring
- Use CloudWatch Logs for troubleshooting
- Set up EventBridge rules for notifications
- Track test reports for quality metrics
- Monitor build duration trends

### Integration
- Use CodePipeline for full CI/CD workflow
- Integrate with GitHub/Bitbucket via webhooks
- Use SNS for notifications
- Export variables for downstream stages

---

## 15. Key Differences from Other CI Tools

### vs Jenkins
- **CodeBuild**: Fully managed, serverless, pay-per-use
- **Jenkins**: Self-managed, requires EC2 instances, fixed cost

### vs GitHub Actions
- **CodeBuild**: AWS-native, better AWS integration
- **GitHub Actions**: GitHub-native, broader ecosystem

### vs GitLab CI
- **CodeBuild**: Separate service, integrates with any Git provider
- **GitLab CI**: Integrated with GitLab, tightly coupled

---

## 16. Architecture Patterns

### Basic CI/CD Pipeline
```
Developer → Git Push → CodeCommit/GitHub
                            ↓
                      CodePipeline
                            ↓
                       CodeBuild (Build & Test)
                            ↓
                       S3 Artifacts
                            ↓
                       CodeDeploy
```

### VPC-Enabled Build
```
CodeBuild → ENI in Private Subnet
                ↓
         Security Group Rules
                ↓
    ┌───────────┴───────────┐
    ↓                       ↓
RDS Database          Internal API
    
Internet Access via NAT Gateway
AWS Services via VPC Endpoints
```

### Multi-Stage Build with Caching
```
Source → CodeBuild (Build)
              ↓
         S3 Cache ← Dependencies
              ↓
         Docker Build (Local Cache)
              ↓
         ECR Push
              ↓
         Deploy Stage
```

---

## 17. Exam Tips

### What to Remember
- **buildspec.yml phases**: install, pre_build, build, post_build
- **VPC requirements**: ENI, security groups, IAM permissions
- **Caching types**: S3 cache vs local cache
- **Privileged mode**: Required for Docker builds
- **Service role**: Needs CloudWatch Logs, S3, and resource-specific permissions
- **Secrets**: Use Parameter Store or Secrets Manager, never hardcode
- **Test reports**: Support JUnit, NUnit, Cucumber, TestNG
- **Timeout**: Default 60 min, max 480 min
- **Compute types**: Small (2 vCPU), Medium (4 vCPU), Large (8 vCPU), 2XL (72 vCPU)

### Common Traps
- Forgetting to enable privileged mode for Docker
- Not configuring VPC permissions correctly
- Hardcoding secrets instead of using Parameter Store
- Not enabling caching for slow builds
- Using wrong artifact location
- Forgetting NAT Gateway for internet access from VPC

### Scenario-Based Questions
- Focus on troubleshooting (permissions, VPC, Docker)
- Understand when to use VPC configuration
- Know how to optimize build performance
- Understand security best practices
- Know integration points with other AWS services

---

## 18. Reference Links

### AWS Official Documentation
- [CodeBuild User Guide](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html)
- [Buildspec Reference](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
- [VPC Support](https://docs.aws.amazon.com/codebuild/latest/userguide/vpc-support.html)
- [IAM Permissions](https://docs.aws.amazon.com/codebuild/latest/userguide/setting-up.html#setting-up-service-role)
- [Test Reports](https://docs.aws.amazon.com/codebuild/latest/userguide/test-reporting.html)
- [Docker Images](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html)

### Workshops
- [CI/CD Workshop](https://catalog.workshops.aws/cicd/en-US)

### Whitepapers
- Practicing Continuous Integration and Continuous Delivery on AWS
- DevOps on AWS

---

## 19. Quick Reference Cheat Sheet

### Essential Commands
```bash
# Start build
aws codebuild start-build --project-name my-project

# Stop build
aws codebuild stop-build --id build-id

# Get build details
aws codebuild batch-get-builds --ids build-id

# List projects
aws codebuild list-projects
```

### Essential IAM Permissions
```
codebuild:StartBuild
codebuild:StopBuild
codebuild:BatchGetBuilds
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
s3:GetObject
s3:PutObject
```

### Buildspec Minimal Example
```yaml
version: 0.2
phases:
  build:
    commands:
      - echo "Hello World"
artifacts:
  files:
    - '**/*'
```

---

## 20. Summary

AWS CodeBuild is a critical component of AWS CI/CD pipelines and is heavily tested in the DOP-C02 exam. Key areas to master:

1. **Buildspec.yml structure and phases**
2. **VPC configuration and networking**
3. **IAM roles and security best practices**
4. **Caching strategies for performance**
5. **Integration with CodePipeline and other AWS services**
6. **Troubleshooting common build failures**
7. **Docker builds and privileged mode**
8. **Secrets management with Parameter Store/Secrets Manager**
9. **Test reports and monitoring**
10. **Cost optimization strategies**

Understanding these concepts with hands-on practice will ensure success on CodeBuild-related questions in the DOP-C02 exam.
