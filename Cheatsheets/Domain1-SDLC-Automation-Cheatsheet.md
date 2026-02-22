# Domain 1: SDLC Automation - DOP-C02 Cheatsheet

## Weight: 22% | Focus: CI/CD Pipelines, Testing, Deployment

---

## 🔥 **MUST KNOW Services**

### **CodePipeline** - Orchestration
- **Stages**: Source → Build → Test → Deploy → Monitor
- **Actions**: Source (CodeCommit/GitHub), Build (CodeBuild), Deploy (CodeDeploy/CloudFormation)
- **Cross-Account**: Use IAM roles, not access keys
- **Artifacts**: Stored in S3, encrypted with KMS

### **CodeBuild** - Build Service
- **buildspec.yml**: Root of source, defines build commands
- **Phases**: install → pre_build → build → post_build
- **Environment**: Managed images vs custom Docker
- **Artifacts**: Output location in S3

### **CodeDeploy** - Deployment
- **Deployment Types**: In-place, Blue/Green
- **Compute Platforms**: EC2, Lambda, ECS
- **Deployment Configs**: OneAtATime, HalfAtATime, AllAtOnce
- **appspec.yml**: Defines deployment steps

### **CodeCommit** - Git Repository
- **Authentication**: IAM users, federated access
- **Encryption**: At rest (KMS), in transit (HTTPS/SSH)
- **Triggers**: Lambda, SNS for notifications

---

## ⚡ **Key Patterns**

### **Multi-Stage Pipeline**
```
Source (CodeCommit) → Build (CodeBuild) → Test → Deploy (CodeDeploy) → Monitor
```

### **Cross-Account Deployment**
1. Create cross-account IAM role in target account
2. Grant CodePipeline permission to assume role
3. Use role in deployment action

### **Blue/Green Deployment**
- **EC2**: Use Auto Scaling + ELB
- **Lambda**: Use aliases and weighted routing
- **ECS**: Use service with task sets

---

## 🎯 **Exam Scenarios**

### **Scenario 1: Pipeline fails at build stage**
- Check buildspec.yml syntax
- Verify IAM permissions for CodeBuild
- Check environment variables and parameters

### **Scenario 2: Cross-account deployment not working**
- Verify cross-account IAM role trust policy
- Check S3 bucket policy for artifacts
- Ensure KMS key permissions for encryption

### **Scenario 3: Blue/green deployment stuck**
- Check health checks configuration
- Verify target group settings
- Review deployment configuration timeouts

---

## 📋 **Quick Commands**

```bash
# Create pipeline
aws codepipeline create-pipeline --pipeline file://pipeline.json

# Start pipeline execution
aws codepipeline start-pipeline-execution --name MyPipeline

# Get pipeline state
aws codepipeline get-pipeline-state --name MyPipeline

# Create CodeBuild project
aws codebuild create-project --name MyProject --source type=CODEPIPELINE

# Create CodeDeploy application
aws deploy create-application --application-name MyApp --compute-platform Server
```

---

## ⚠️ **Common Mistakes**
- Forgetting to enable versioning on S3 artifact bucket
- Not configuring proper IAM permissions for cross-account access
- Missing buildspec.yml or appspec.yml files
- Incorrect deployment configuration for target platform
- Not setting up proper health checks for blue/green deployments

---

## 🔑 **Key Points**
- **CodePipeline** orchestrates, doesn't execute
- **Cross-account** requires IAM roles, not users
- **Artifacts** must be in same region as pipeline
- **Blue/Green** requires health checks and load balancers
- **buildspec.yml** and **appspec.yml** are mandatory files