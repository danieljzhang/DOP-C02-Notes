# All AWS Services Review - DOP-C02 Exam

## 🎯 **Quick Service Reference for DOP-C02**

### **Domain 1: SDLC Automation (22%)**

#### **CI/CD Core Services**
- **CodePipeline** - Orchestrates CI/CD workflows, supports cross-account deployments
- **CodeBuild** - Managed build service, uses buildspec.yml, supports Docker
- **CodeDeploy** - Deployment service, supports blue/green, rolling, canary strategies
- **CodeCommit** - Git repositories (returned to full GA for new customers, Nov 2025)
- **CodeStar/CodeCatalyst** - Development environments and project templates

#### **Integration & Automation**
- **EventBridge** - Event routing, custom events, cross-account event sharing
- **Step Functions** - Workflow orchestration, state machines, error handling
- **Lambda** - Serverless compute, event-driven processing, 15-minute max runtime
- **API Gateway** - REST/HTTP APIs, WebSocket, request/response transformation
- **X-Ray** - Distributed tracing, performance analysis, service maps

#### **Testing & Quality**
- **Automated Testing** - Unit, integration, security, performance testing
- **Artifacts Management** - S3 storage, versioning, lifecycle policies
- **Security Integration** - SAST/DAST scanning, vulnerability assessments

### **Domain 2: Infrastructure as Code (17%)**

#### **IaC Services**
- **CloudFormation** - Native IaC, YAML/JSON, intrinsic functions, drift detection
- **CDK** - Code-based IaC, synthesizes to CloudFormation, multiple languages
- **SAM** - Serverless application model, extends CloudFormation
- **Terraform** - Third-party IaC, state management, workspaces

#### **Configuration Management**
- **Systems Manager** - Parameter Store, Session Manager, Run Command, Patch Manager
- **OpsWorks** - Chef/Puppet managed instances, application lifecycle
- **Config** - Configuration compliance, rules, remediation actions
- **Service Catalog** - Standardized product portfolios, governance

#### **Governance**
- **Organizations** - Multi-account management, SCPs, consolidated billing
- **Control Tower** - Landing zones, guardrails, account factory

### **Domain 3: Resilient Cloud Solutions (15%)**

#### **High Availability**
- **Auto Scaling** - EC2, ECS, Application Auto Scaling, predictive scaling
- **Elastic Load Balancing** - ALB (Layer 7), NLB (Layer 4), GWLB (Layer 3)
- **Route 53** - DNS, health checks, failover routing, geolocation
- **Multi-AZ Deployments** - RDS, ElastiCache, EFS cross-AZ replication

#### **Disaster Recovery**
- **S3 Cross-Region Replication** - Automatic replication, versioning, lifecycle
- **RDS Multi-AZ** - Synchronous replication, automatic failover
- **Aurora Global Database** - Cross-region read replicas, fast recovery
- **EKS** - Kubernetes orchestration, multi-AZ clusters, managed node groups

### **Domain 4: Monitoring and Logging (15%)**

#### **Monitoring Services**
- **CloudWatch** - Metrics, alarms, dashboards, custom metrics, anomaly detection
- **CloudWatch Logs** - Log aggregation, insights, metric filters, retention
- **X-Ray** - Application tracing, service maps, performance bottlenecks
- **CloudTrail** - API logging, governance, compliance, data events

#### **Security Monitoring**
- **Config** - Resource compliance, configuration history, rules
- **GuardDuty** - Threat detection, ML-based analysis, findings
- **Inspector** - Vulnerability assessments, network reachability
- **Security Hub** - Centralized security findings, compliance standards

#### **Systems Management**
- **Systems Manager** - Operational insights, automation, patch management
- **Personal Health Dashboard** - Service health, maintenance notifications

### **Domain 5: Incident and Event Response (14%)**

#### **Event Processing**
- **EventBridge** - Event routing, custom events, third-party integrations
- **SNS** - Pub/sub messaging, mobile push, email, SMS notifications
- **SQS** - Message queuing, FIFO, dead letter queues, visibility timeout
- **Lambda** - Event-driven processing, automatic scaling, multiple triggers

#### **Workflow Orchestration**
- **Step Functions** - State machines, parallel processing, error handling
- **CloudWatch Alarms** - Threshold monitoring, composite alarms, actions
- **Auto Scaling** - Dynamic scaling based on metrics, scheduled scaling

### **Domain 6: Security and Compliance (17%)**

#### **Identity & Access**
- **IAM** - Users, roles, policies, conditions, cross-account access
- **Organizations** - SCPs, account management, consolidated billing
- **Control Tower** - Governance, guardrails, compliance monitoring

#### **Encryption & Secrets**
- **KMS** - Key management, envelope encryption, cross-account access
- **Secrets Manager** - Secret rotation, database credentials, API keys
- **Certificate Manager** - SSL/TLS certificates, automatic renewal

#### **Security Services**
- **WAF** - Web application firewall, rate limiting, geo-blocking
- **Shield** - DDoS protection, Standard (free), Advanced (paid)
- **GuardDuty** - Threat detection, behavioral analysis, findings

## 🔥 **High-Impact Integration Patterns**

### **Cross-Account Patterns**
```
Source Account → Assume Role → Target Account
S3 Bucket Policy + KMS Key Policy + IAM Role
```

### **CI/CD Pipeline Pattern**
```
CodeCommit → CodeBuild → CodeDeploy → CloudWatch
EventBridge → Lambda → SNS → Step Functions
```

### **Event-Driven Architecture**
```
API Gateway → Lambda → EventBridge → [SNS/SQS/Lambda]
CloudWatch Alarm → EventBridge → Lambda → Auto Scaling
```

### **Multi-Region Resilience**
```
Route 53 → CloudFront → ALB (Multi-AZ) → Auto Scaling
RDS Multi-AZ → Cross-Region Read Replicas
S3 → Cross-Region Replication
```

## ⚡ **Critical CLI Commands**

### **CodePipeline**
```bash
aws codepipeline start-pipeline-execution --name MyPipeline
aws codepipeline get-pipeline-state --name MyPipeline
aws codepipeline stop-pipeline-execution --pipeline-name MyPipeline
```

### **CodeBuild**
```bash
aws codebuild start-build --project-name MyProject
aws codebuild batch-get-builds --ids build-id
aws codebuild list-projects
```

### **CloudFormation**
```bash
aws cloudformation deploy --template-file template.yaml --stack-name MyStack
aws cloudformation describe-stacks --stack-name MyStack
aws cloudformation detect-stack-drift --stack-name MyStack
```

### **Auto Scaling**
```bash
aws autoscaling describe-auto-scaling-groups
aws autoscaling set-desired-capacity --auto-scaling-group-name MyASG --desired-capacity 3
aws autoscaling put-scaling-policy --policy-name MyPolicy
```

### **Systems Manager**
```bash
aws ssm get-parameter --name /myapp/database/url --with-decryption
aws ssm send-command --document-name "AWS-RunShellScript" --targets "Key=tag:Environment,Values=prod"
aws ssm start-session --target i-1234567890abcdef0
```

## 🎯 **Exam Recognition Patterns**

### **Service Selection Triggers**
- **"Cross-account deployment"** → IAM roles + trust policies
- **"Zero-downtime deployment"** → Blue/green or canary
- **"Automatic scaling"** → Auto Scaling Groups + CloudWatch
- **"Event-driven processing"** → EventBridge + Lambda
- **"Secure secrets"** → Secrets Manager or Parameter Store (SecureString)
- **"Infrastructure as Code"** → CloudFormation, CDK, or Terraform
- **"Monitoring and alerting"** → CloudWatch + SNS
- **"Compliance monitoring"** → Config + remediation actions

### **Common Exam Traps**
- **CloudWatch vs CloudTrail vs Config** - Metrics vs API logs vs compliance
- **ALB vs NLB vs GWLB** - Layer 7 vs Layer 4 vs Layer 3
- **Multi-AZ vs Read Replicas** - HA vs read scaling
- **SNS vs SQS** - Pub/sub vs queuing
- **Parameter Store vs Secrets Manager** - Configuration vs secrets with rotation

### **Architecture Decision Trees**
1. **Need automatic failover?** → Multi-AZ deployment
2. **Need cross-region disaster recovery?** → Cross-region replication
3. **Need to process events?** → EventBridge + Lambda
4. **Need workflow orchestration?** → Step Functions
5. **Need secure parameter storage?** → Parameter Store (config) vs Secrets Manager (secrets)

## 📊 **Performance & Cost Optimization**

### **Auto Scaling Optimization**
- **Target Tracking** - Maintain specific metric value
- **Step Scaling** - Scale based on alarm breach size
- **Predictive Scaling** - ML-based capacity planning
- **Scheduled Scaling** - Time-based scaling

### **Lambda Optimization**
- **Memory allocation** - Affects CPU and network performance
- **Provisioned concurrency** - Eliminates cold starts
- **Dead letter queues** - Handle failed invocations
- **Reserved concurrency** - Limit concurrent executions

### **Cost Management**
- **Reserved Instances** - Long-term capacity reservations
- **Spot Instances** - Fault-tolerant workloads
- **S3 Intelligent Tiering** - Automatic cost optimization
- **CloudWatch Logs retention** - Manage log storage costs

## 🚨 **Security Best Practices**

### **IAM Principles**
- **Least privilege** - Minimum necessary permissions
- **Role-based access** - Use roles instead of users for services
- **Conditions** - Restrict access based on context
- **Regular reviews** - Audit permissions regularly

### **Encryption Standards**
- **Data at rest** - S3, EBS, RDS encryption
- **Data in transit** - TLS/SSL for all communications
- **Key management** - KMS for centralized key management
- **Envelope encryption** - Encrypt data keys with master keys

### **Network Security**
- **VPC isolation** - Separate environments
- **Security groups** - Stateful firewall rules
- **NACLs** - Stateless subnet-level rules
- **VPC endpoints** - Private service access

## 🎯 **Final Exam Checklist**

### **Must Know Cold**
- [ ] Cross-account IAM role assumption process
- [ ] Blue/green vs rolling vs canary deployment differences
- [ ] Multi-AZ vs Read Replica use cases
- [ ] EventBridge event routing and filtering
- [ ] CloudFormation intrinsic functions (!Ref, !GetAtt, !Sub)
- [ ] Auto Scaling policy types and triggers
- [ ] Lambda error handling and retry behavior
- [ ] Step Functions state types and error handling

### **Service Integration Patterns**
- [ ] CodePipeline + CodeBuild + CodeDeploy workflow
- [ ] EventBridge + Lambda + SNS/SQS integration
- [ ] CloudWatch + Auto Scaling + ELB integration
- [ ] Route 53 + CloudFront + S3 global architecture
- [ ] Systems Manager + EC2 + Auto Scaling management

### **Troubleshooting Scenarios**
- [ ] Pipeline failures and rollback procedures
- [ ] Auto Scaling issues and debugging
- [ ] Cross-account access problems
- [ ] Performance bottlenecks identification
- [ ] Security incident response procedures

**Remember**: Think practically, consider cost implications, and always choose the most AWS-native solution when multiple options exist. Good luck! 🚀