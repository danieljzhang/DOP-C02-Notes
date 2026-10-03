# Domain 1: SDLC Automation - DOP-C02 Study Notes

## Overview
This domain covers the implementation and management of continuous integration and continuous deployment (CI/CD) pipelines, automated testing, and software development lifecycle automation.

## Domain Weight: 22% of exam

## Key Topics Covered

### 1.1 CI/CD Pipeline Implementation
- AWS CodePipeline for orchestration
- AWS CodeBuild for build automation
- AWS CodeDeploy for deployment automation
- AWS CodeCommit for source control

### 1.2 Testing and Quality Assurance
- Automated testing strategies
- Test integration in pipelines
- Quality gates and approvals
- Security scanning integration

### 1.3 Artifact Management
- Build artifact storage and versioning
- Dependency management
- Container image management
- Package distribution

### 1.4 Infrastructure Automation
- Infrastructure as Code integration
- Environment provisioning
- Configuration management
- Deployment strategies

## Files in this Domain

### Core CI/CD Services
- **CodeBuild/** - Build service for compiling and testing code
- **CodeCommit/** - Git-based source control service
- **CodeDeploy/** - Deployment service for applications
- **CodePipeline/** - CI/CD orchestration service
- **EventBridge/** - Event-driven automation and integration
- **GitHub-Actions-Integration/** - Third-party CI/CD integration
- **[API-Gateway-General.md](./API-Gateway-General.md)** - API management and deployment

### Supporting Services and Concepts
- **[Artifacts.md](./Artifacts.md)** - Build artifact management
- **[Automated-Testing.md](./Automated-Testing.md)** - Testing strategies and implementation
- **[AWS-XRay-General.md](./AWS-XRay-General.md)** - Application tracing and debugging
- **[CICD-MonitoringObservability-General.md](./CICD-MonitoringObservability-General.md)** - Pipeline monitoring
- **[CICD-SecurityIntegration-General.md](./CICD-SecurityIntegration-General.md)** - Security in CI/CD
- **[CICD-TestingStrategies-General.md](./CICD-TestingStrategies-General.md)** - Testing methodologies
- **[CodeStar-CodeCatalyst-General.md](./CodeStar-CodeCatalyst-General.md)** - Development environments
- **[Config-General.md](./Config-General.md)** - Configuration management
- **[Deployment-Strategies.md](./Deployment-Strategies.md)** - Deployment patterns
- **[Implement-CICD-Pipelines.md](./Implement-CICD-Pipelines.md)** - Pipeline implementation
- **[StepFunctions-General.md](./StepFunctions-General.md)** - Workflow orchestration
- **[SystemsManager-General.md](./SystemsManager-General.md)** - Systems management

## Key Concepts

### CI/CD Pipeline Stages
1. **Source** - Code repository integration
2. **Build** - Compilation and packaging
3. **Test** - Automated testing execution
4. **Deploy** - Application deployment
5. **Monitor** - Post-deployment monitoring

### Deployment Strategies
- **Blue/Green** - Zero-downtime deployments
- **Rolling** - Gradual instance replacement
- **Canary** - Gradual traffic shifting
- **Immutable** - Complete infrastructure replacement

### Testing Strategies
- **Unit Testing** - Individual component testing
- **Integration Testing** - Component interaction testing
- **End-to-End Testing** - Full application workflow testing
- **Security Testing** - Vulnerability and compliance scanning
- **Performance Testing** - Load and stress testing

## Integration Patterns

### Multi-Service Pipeline
```
CodeCommit → CodeBuild → CodeDeploy → CloudWatch
     ↓           ↓           ↓           ↓
EventBridge → Lambda → SNS → Step Functions
```

### Cross-Account Deployment
- Cross-account IAM roles
- Artifact sharing strategies
- Environment isolation
- Security boundaries

### Third-Party Integration
- GitHub integration via CodeStar connections (live service; not the retired CodeStar)
- Jenkins pipeline integration
- External testing tools
- Monitoring and alerting systems

## Common Exam Scenarios

1. **Multi-stage CI/CD pipeline** - CodePipeline + CodeBuild + CodeDeploy
2. **Cross-account deployment** - IAM roles and artifact sharing
3. **Automated testing integration** - Test stages in pipelines
4. **Blue/green deployment** - Zero-downtime deployment strategy
5. **Container deployment pipeline** - ECR + ECS/EKS deployment
6. **Infrastructure as Code pipeline** - CloudFormation/CDK deployment
7. **Security scanning integration** - SAST/DAST in pipelines
8. **Multi-environment promotion** - Dev → Test → Prod pipeline
9. **Rollback and recovery** - Automated rollback mechanisms
10. **Pipeline monitoring and alerting** - CloudWatch integration

## Study Tips

### Focus Areas
- Understand CodePipeline stage types and actions
- Know CodeBuild buildspec.yml configuration
- Practice CodeDeploy deployment configurations
- Understand cross-account deployment patterns
- Know testing integration strategies

### Hands-on Practice
- Build end-to-end CI/CD pipelines
- Practice blue/green and canary deployments
- Implement automated testing in pipelines
- Set up cross-account deployments
- Configure pipeline monitoring and alerting

### Integration Knowledge
- Service-to-service communication patterns
- IAM roles and policies for CI/CD
- Event-driven automation with EventBridge
- Container deployment strategies
- Infrastructure as Code integration

## Additional Resources

### AWS Documentation
- [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/)
- [AWS CodeBuild User Guide](https://docs.aws.amazon.com/codebuild/)
- [AWS CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/)

### Best Practices
- [CI/CD Best Practices](https://aws.amazon.com/devops/continuous-integration/)
- [AWS DevOps Best Practices](https://aws.amazon.com/devops/)
- [Security in CI/CD](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)