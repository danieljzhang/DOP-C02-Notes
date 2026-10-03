# DOP-C02 Practice Scenarios - Exam Simulation

## 🎯 **High-Frequency Exam Scenarios**

### **Scenario 1: Cross-Account CI/CD Pipeline**
**Question**: A company needs to deploy applications from a development account to production accounts across multiple regions. The solution must be secure, automated, and support rollback capabilities.

**Key Requirements**:
- Cross-account deployment
- Multi-region support
- Automated rollback
- Security best practices

**Solution Pattern**:
```
Dev Account: CodeCommit → CodePipeline → CodeBuild
     ↓
Cross-Account IAM Role Assumption
     ↓
Prod Accounts: CloudFormation → Application Deployment
     ↓
CloudWatch Monitoring → Automated Rollback (if needed)
```

**Implementation Steps**:
1. **IAM Setup**: Create cross-account roles with trust policies
2. **Pipeline Configuration**: Configure CodePipeline with cross-account actions
3. **Artifact Management**: S3 bucket policies for cross-account artifact access
4. **Deployment Strategy**: Blue/green deployment with automated rollback
5. **Monitoring**: CloudWatch alarms for deployment health

**Exam Tips**:
- Focus on IAM role assumption and trust policies
- Remember S3 bucket policies for artifact sharing
- Consider KMS key policies for cross-account encryption
- Blue/green deployment is preferred for zero-downtime

---

### **Scenario 2: Event-Driven Auto Scaling**
**Question**: An e-commerce application experiences unpredictable traffic spikes. Design an event-driven architecture that automatically scales resources based on custom business metrics and integrates with existing monitoring systems.

**Key Requirements**:
- Custom business metrics
- Event-driven scaling
- Integration with existing systems
- Cost optimization

**Solution Pattern**:
```
Application → Custom CloudWatch Metrics → CloudWatch Alarms
     ↓                    ↓                      ↓
EventBridge → Lambda → Auto Scaling Actions
     ↓           ↓              ↓
SNS Notifications → Step Functions → Cost Optimization
```

**Implementation Steps**:
1. **Custom Metrics**: Publish business metrics to CloudWatch
2. **Alarm Configuration**: Set up composite alarms for scaling triggers
3. **Event Processing**: EventBridge rules for alarm state changes
4. **Scaling Logic**: Lambda functions for intelligent scaling decisions
5. **Cost Control**: Scheduled scaling and spot instance integration

**Exam Tips**:
- Custom metrics require CloudWatch agent or SDK
- Composite alarms reduce false positives
- EventBridge enables decoupled event processing
- Consider predictive scaling for cost optimization

---

### **Scenario 3: Multi-Region Disaster Recovery**
**Question**: A financial services company requires a disaster recovery solution with RPO of 1 hour and RTO of 4 hours. The solution must maintain data consistency and support automated failover.

**Key Requirements**:
- RPO: 1 hour
- RTO: 4 hours
- Data consistency
- Automated failover

**Solution Pattern**:
```
Primary Region: Application + RDS Multi-AZ + S3
     ↓
Cross-Region Replication
     ↓
DR Region: Warm Standby + RDS Read Replica + S3 CRR
     ↓
Route 53 Health Checks → Automated Failover
```

**Implementation Steps**:
1. **Database Replication**: RDS cross-region read replicas
2. **Data Replication**: S3 cross-region replication
3. **Infrastructure**: CloudFormation templates for DR region
4. **Health Monitoring**: Route 53 health checks and failover routing
5. **Automation**: Lambda functions for failover orchestration

**Exam Tips**:
- RDS read replicas provide cross-region DR capability
- S3 CRR ensures data availability in DR region
- Route 53 health checks enable automated DNS failover
- Warm standby balances cost and RTO requirements

---

### **Scenario 4: Compliance Monitoring and Remediation**
**Question**: A healthcare organization needs to ensure HIPAA compliance across all AWS accounts. Implement automated compliance monitoring with immediate remediation for violations.

**Key Requirements**:
- HIPAA compliance
- Multi-account monitoring
- Automated remediation
- Audit trails

**Solution Pattern**:
```
AWS Config → Compliance Rules → Non-Compliant Resources
     ↓              ↓                    ↓
EventBridge → Lambda → Automated Remediation
     ↓           ↓              ↓
CloudTrail → S3 → Audit Reports
```

**Implementation Steps**:
1. **Config Setup**: Enable Config in all accounts and regions
2. **Compliance Rules**: Deploy HIPAA-specific Config rules
3. **Event Processing**: EventBridge for compliance violations
4. **Remediation**: Lambda functions for automated fixes
5. **Audit Trail**: CloudTrail integration with centralized logging

**Exam Tips**:
- Config rules can be deployed via CloudFormation
- EventBridge enables real-time compliance monitoring
- Lambda remediation should follow least privilege principle
- CloudTrail provides comprehensive audit capabilities

---

### **Scenario 5: Secrets Management in CI/CD**
**Question**: A development team needs to securely manage database credentials and API keys in their CI/CD pipeline without hardcoding secrets in code or configuration files.

**Key Requirements**:
- Secure secrets management
- CI/CD integration
- No hardcoded secrets
- Rotation capabilities

**Solution Pattern**:
```
Secrets Manager → Automatic Rotation → Database/APIs
     ↓                    ↓                ↓
CodeBuild → Retrieve Secrets → Application Deployment
     ↓              ↓                    ↓
IAM Roles → Least Privilege → Audit Logging
```

**Implementation Steps**:
1. **Secrets Storage**: Store secrets in AWS Secrets Manager
2. **Rotation Setup**: Configure automatic secret rotation
3. **IAM Configuration**: Create roles with minimal required permissions
4. **Pipeline Integration**: Retrieve secrets in CodeBuild buildspec
5. **Monitoring**: CloudTrail logging for secret access

**Exam Tips**:
- Secrets Manager provides automatic rotation capabilities
- Use IAM roles instead of access keys in CI/CD
- Parameter Store SecureString is alternative for simple secrets
- Never store secrets in code repositories or container images

---

### **Scenario 6: Container Deployment Pipeline**
**Question**: A microservices application needs a CI/CD pipeline for containerized deployments to EKS with blue/green deployment strategy and automated testing.

**Key Requirements**:
- Container deployment
- EKS integration
- Blue/green deployment
- Automated testing

**Solution Pattern**:
```
CodeCommit → CodeBuild → ECR → EKS Blue/Green Deployment
     ↓           ↓        ↓           ↓
Unit Tests → Integration Tests → Security Scanning → Health Checks
     ↓           ↓        ↓           ↓
Quality Gates → Approval → Deployment → Monitoring
```

**Implementation Steps**:
1. **Container Build**: CodeBuild with Docker support
2. **Image Registry**: Push images to Amazon ECR
3. **Security Scanning**: ECR image vulnerability scanning
4. **Deployment**: EKS blue/green deployment with ALB
5. **Testing**: Automated testing at each stage

**Exam Tips**:
- ECR provides built-in vulnerability scanning
- EKS supports blue/green deployments via ALB
- Use Kubernetes health checks for deployment validation
- Consider Fargate for serverless container execution

---

### **Scenario 7: Infrastructure Drift Detection**
**Question**: A company's infrastructure frequently drifts from its defined CloudFormation templates. Implement automated drift detection and remediation to maintain infrastructure consistency.

**Key Requirements**:
- Drift detection
- Automated remediation
- Infrastructure consistency
- Change tracking

**Solution Pattern**:
```
CloudFormation Stacks → Drift Detection → EventBridge
     ↓                        ↓              ↓
Config Rules → Compliance Monitoring → Lambda Remediation
     ↓              ↓                    ↓
CloudTrail → Change Tracking → Notifications
```

**Implementation Steps**:
1. **Drift Detection**: Schedule regular CloudFormation drift detection
2. **Event Processing**: EventBridge rules for drift events
3. **Remediation**: Lambda functions to correct drift
4. **Monitoring**: Config rules for infrastructure compliance
5. **Alerting**: SNS notifications for drift detection

**Exam Tips**:
- CloudFormation drift detection identifies manual changes
- Config rules can detect configuration compliance
- EventBridge enables automated response to drift events
- Consider using CloudFormation StackSets for multi-account consistency

---

### **Scenario 8: Performance Monitoring and Optimization**
**Question**: A web application experiences intermittent performance issues. Implement comprehensive monitoring to identify bottlenecks and automatically optimize performance.

**Key Requirements**:
- Performance monitoring
- Bottleneck identification
- Automated optimization
- User experience tracking

**Solution Pattern**:
```
Application → X-Ray Tracing → Service Maps → Bottleneck Analysis
     ↓              ↓              ↓              ↓
CloudWatch Metrics → Alarms → Auto Scaling → Performance Optimization
     ↓              ↓         ↓              ↓
Real User Monitoring → Dashboards → Alerts → Remediation
```

**Implementation Steps**:
1. **Tracing Setup**: Integrate X-Ray SDK in application
2. **Metrics Collection**: Custom CloudWatch metrics for business KPIs
3. **Monitoring**: Comprehensive dashboards and alarms
4. **Optimization**: Auto Scaling based on performance metrics
5. **Analysis**: Regular performance analysis and optimization

**Exam Tips**:
- X-Ray provides distributed tracing capabilities
- Custom metrics enable business-specific monitoring
- CloudWatch Insights helps analyze performance patterns
- Auto Scaling can be based on custom metrics

---

### **Scenario 9: Security Incident Response**
**Question**: Design an automated security incident response system that detects threats, isolates affected resources, and initiates remediation procedures while maintaining audit trails.

**Key Requirements**:
- Threat detection
- Automated isolation
- Incident response
- Audit trails

**Solution Pattern**:
```
GuardDuty → Security Findings → EventBridge → Lambda Response
     ↓              ↓              ↓              ↓
Security Hub → Centralized Findings → Step Functions → Orchestrated Response
     ↓              ↓                    ↓              ↓
CloudTrail → Audit Logs → S3 → Compliance Reporting
```

**Implementation Steps**:
1. **Threat Detection**: Enable GuardDuty across all accounts
2. **Centralization**: Security Hub for unified findings
3. **Event Processing**: EventBridge for real-time response
4. **Orchestration**: Step Functions for complex response workflows
5. **Audit**: Comprehensive logging and reporting

**Exam Tips**:
- GuardDuty uses ML for threat detection
- Security Hub centralizes security findings
- EventBridge enables real-time incident response
- Step Functions orchestrate complex response procedures

---

### **Scenario 10: Cost Optimization Automation**
**Question**: Implement automated cost optimization for a development environment that scales resources based on usage patterns and automatically shuts down unused resources.

**Key Requirements**:
- Automated cost optimization
- Usage-based scaling
- Resource cleanup
- Cost monitoring

**Solution Pattern**:
```
CloudWatch Metrics → Cost Analysis → EventBridge → Lambda Optimization
     ↓                    ↓              ↓              ↓
Trusted Advisor → Recommendations → Scheduled Actions → Resource Management
     ↓                    ↓                    ↓              ↓
Cost Explorer → Budget Alerts → SNS → Automated Response
```

**Implementation Steps**:
1. **Monitoring**: CloudWatch metrics for resource utilization
2. **Analysis**: Cost Explorer and Trusted Advisor insights
3. **Automation**: Lambda functions for resource optimization
4. **Scheduling**: EventBridge scheduled rules for cleanup
5. **Alerting**: Budget alerts for cost thresholds

**Exam Tips**:
- Trusted Advisor provides cost optimization recommendations
- EventBridge scheduled rules enable time-based automation
- Lambda can automate resource lifecycle management
- Cost Explorer APIs enable programmatic cost analysis

---

## 🎯 **Scenario Recognition Patterns**

### **Keywords to Service Mapping**
| Keyword/Phrase | Primary Service | Supporting Services |
|----------------|----------------|-------------------|
| "Cross-account deployment" | IAM Roles | CodePipeline, S3, KMS |
| "Zero-downtime deployment" | CodeDeploy | ALB, Auto Scaling |
| "Event-driven architecture" | EventBridge | Lambda, SNS, SQS |
| "Secrets management" | Secrets Manager | Parameter Store, KMS |
| "Compliance monitoring" | Config | CloudTrail, Lambda |
| "Performance monitoring" | X-Ray | CloudWatch, Lambda |
| "Threat detection" | GuardDuty | Security Hub, EventBridge |
| "Infrastructure as Code" | CloudFormation | CDK, CodePipeline |
| "Container deployment" | ECS/EKS | ECR, CodeBuild |
| "Cost optimization" | Cost Explorer | Trusted Advisor, Lambda |

### **Architecture Decision Trees**

#### **Deployment Strategy Selection**
```
Zero-downtime required?
├── Yes → Blue/Green (CodeDeploy) or Canary (ALB)
└── No → Rolling deployment acceptable

High availability required?
├── Yes → Multi-AZ deployment
└── No → Single AZ acceptable

Cross-account deployment?
├── Yes → IAM cross-account roles + trust policies
└── No → Same-account deployment
```

#### **Monitoring Strategy Selection**
```
What needs monitoring?
├── Application Performance → X-Ray + CloudWatch
├── Infrastructure → CloudWatch + Systems Manager
├── Security → GuardDuty + Security Hub + Config
└── Compliance → Config + CloudTrail + Lambda

Real-time response needed?
├── Yes → EventBridge + Lambda
└── No → Scheduled analysis acceptable
```

## 🚨 **Common Exam Traps**

### **Service Confusion**
1. **Parameter Store vs Secrets Manager**
   - Parameter Store: Configuration data, simple secrets
   - Secrets Manager: Complex secrets with rotation

2. **CloudWatch vs X-Ray vs CloudTrail**
   - CloudWatch: Metrics and monitoring
   - X-Ray: Application tracing
   - CloudTrail: API audit logging

3. **Blue/Green vs Rolling vs Canary**
   - Blue/Green: Complete environment switch
   - Rolling: Gradual instance replacement
   - Canary: Gradual traffic shifting

### **Architecture Mistakes**
1. **Over-engineering** - Choose simple solutions over complex ones
2. **Ignoring cost** - Always consider cost implications
3. **Missing security** - Security should be built-in
4. **Poor integration** - Consider how services work together

### **Implementation Errors**
1. **Wrong IAM permissions** - Use least privilege principle
2. **Missing encryption** - Encrypt data at rest and in transit
3. **No monitoring** - Always include monitoring and alerting
4. **Poor error handling** - Implement proper error handling and retries

## 💡 **Success Strategies**

### **Scenario Analysis Process**
1. **Identify requirements** - What are the key requirements?
2. **Consider constraints** - Budget, timeline, compliance
3. **Map to services** - Which AWS services address requirements?
4. **Design integration** - How do services work together?
5. **Validate solution** - Does it meet all requirements?

### **Answer Selection Tips**
1. **Eliminate obviously wrong** - Remove clearly incorrect options
2. **Look for AWS-native** - Prefer AWS services over third-party
3. **Consider simplicity** - Simple solutions are usually better
4. **Think security** - Security should be built-in
5. **Validate completeness** - Does the solution address all requirements?

### **Time Management**
- **Quick scan** - Read all options quickly first
- **Identify key requirements** - Focus on critical requirements
- **Eliminate wrong answers** - Use process of elimination
- **Make educated guess** - If unsure, make best guess and move on
- **Flag for review** - Mark uncertain questions for later review

**Remember**: Practice these scenarios until you can quickly identify the pattern and select the appropriate AWS services and architecture. Focus on understanding the integration between services rather than memorizing individual service features. Good luck! 🚀

---

### **Scenario 11: RDS Major Version Upgrade with Rollback**
**Question**: A company needs to upgrade their production RDS MySQL 5.7 instance to MySQL 8.0. The upgrade must have less than 5 minutes of downtime and must support instant rollback if the new version causes application issues.

**Key Requirements**:
- Minimal downtime (< 5 minutes)
- Instant rollback capability
- No data loss
- Validate before cutover

**Solution Pattern**:
```
RDS Blue/Green Deployment
Blue (prod MySQL 5.7) → Green (MySQL 8.0 copy)
        ↓
Logical replication keeps green in sync
        ↓
Run integration tests against green endpoint
        ↓
switchover-blue-green-deployment (< 1 min downtime)
        ↓
Keep blue for rollback → delete after validation
```

**Implementation Steps**:
1. Create blue/green deployment targeting engine version 8.0
2. Wait for green to be available and in sync
3. Run application tests against the green endpoint
4. Execute switchover — RDS swaps DNS names, downtime < 1 minute
5. Monitor for 24–48 hours, then delete the blue environment

**Exam Tips**:
- Blue/green is the answer for "minimal downtime + rollback" RDS upgrades
- Endpoints don't change — applications need no reconfiguration
- Blue is retained after switchover — you pay for it until you delete it
- Not supported for Oracle or SQL Server

---

### **Scenario 12: Stream Processing with Enrichment**
**Question**: An e-commerce company has orders arriving in an SQS queue. Each order needs to be enriched with customer loyalty tier data from a Lambda function before being sent to a Step Functions workflow for fulfilment. The team wants to avoid writing polling and routing code.

**Key Requirements**:
- Poll SQS automatically
- Filter out test orders before enrichment
- Enrich with customer data
- Route to Step Functions
- No custom polling Lambda

**Solution Pattern**:
```
SQS Queue (source)
    → Filter: body.orderType != "TEST"
    → Enrichment: Lambda adds loyalty tier from DynamoDB
    → Target: Step Functions Express Workflow
```

**Implementation Steps**:
1. Create an EventBridge Pipe with the SQS queue as source
2. Add a filter pattern to drop test orders
3. Configure Lambda enrichment to look up customer loyalty tier
4. Set Step Functions Express Workflow as the target
5. Attach an execution role with SQS, Lambda, and Step Functions permissions

**Exam Tips**:
- EventBridge Pipes = point-to-point with built-in enrichment; no polling Lambda needed
- Filter before enrichment saves Lambda invocations and cost
- Step Functions enrichment must be Express Workflow (synchronous) — not Standard
- Pipes vs Rules: Pipes = one source → one target; Rules = fan-out to many targets

---

### **Scenario 13: Centralised Multi-Account Access Management**
**Question**: A company has 40 AWS accounts across dev, staging, and production OUs. Developers need read-only access to all accounts and full access to dev accounts. The security team needs admin access to all accounts. Currently each account has separate IAM users. The company wants a single login experience with MFA enforced everywhere.

**Key Requirements**:
- Single sign-on across all 40 accounts
- Role-based access (developer vs security team)
- MFA enforced centrally
- No per-account IAM user management

**Solution Pattern**:
```
Corporate IdP (Okta/Azure AD)
    ↓ SAML 2.0
IAM Identity Center (management account)
    ↓
Permission Sets:
  - ReadOnly → all 40 accounts (developers)
  - PowerUser → dev OU accounts (developers)
  - AdminAccess → all 40 accounts (security team)
    ↓
AWS Access Portal (single login URL)
    ↓
Temporary credentials per account/role selection
```

**Implementation Steps**:
1. Enable IAM Identity Center in the management account
2. Connect to corporate IdP via SAML 2.0
3. Create three permission sets (ReadOnly, PowerUser, AdminAccess)
4. Assign permission sets to groups at the OU level via Organizations
5. Enable MFA enforcement in Identity Center — applies to all accounts automatically
6. Users log in at the Access Portal, select account + role

**Exam Tips**:
- IAM Identity Center = the answer for "single login across multiple accounts"
- Permission sets deploy as IAM roles automatically — no manual role creation per account
- MFA configured once in Identity Center applies everywhere — no per-account IAM MFA policy
- "AWS SSO" in older questions = IAM Identity Center — same service
- For CI/CD pipelines, use IAM roles or OIDC — not Identity Center
**Question**: A company needs to upgrade their production RDS MySQL 5.7 instance to MySQL 8.0. The upgrade must have less than 5 minutes of downtime and must support instant rollback if the new version causes application issues.

**Key Requirements**:
- Minimal downtime (< 5 minutes)
- Instant rollback capability
- No data loss
- Validate before cutover

**Solution Pattern**:
```
RDS Blue/Green Deployment
Blue (prod MySQL 5.7) → Green (MySQL 8.0 copy)
        ↓
Logical replication keeps green in sync
        ↓
Run integration tests against green endpoint
        ↓
switchover-blue-green-deployment (< 1 min downtime)
        ↓
Keep blue for rollback → delete after validation
```

**Implementation Steps**:
1. Create blue/green deployment targeting engine version 8.0
2. Wait for green to be available and in sync
3. Run application tests against the green endpoint
4. Execute switchover — RDS swaps DNS names, downtime < 1 minute
5. Monitor for 24–48 hours, then delete the blue environment

**Exam Tips**:
- Blue/green is the answer for "minimal downtime + rollback" RDS upgrades
- Endpoints don't change — applications need no reconfiguration
- Blue is retained after switchover — you pay for it until you delete it
- Not supported for Oracle or SQL Server

---

### **Scenario 12: Stream Processing with Enrichment**
**Question**: An e-commerce company has orders arriving in an SQS queue. Each order needs to be enriched with customer loyalty tier data from a Lambda function before being sent to a Step Functions workflow for fulfilment. The team wants to avoid writing polling and routing code.

**Key Requirements**:
- Poll SQS automatically
- Filter out test orders before enrichment
- Enrich with customer data
- Route to Step Functions
- No custom polling Lambda

**Solution Pattern**:
```
SQS Queue (source)
    → Filter: body.orderType != "TEST"
    → Enrichment: Lambda adds loyalty tier from DynamoDB
    → Target: Step Functions Express Workflow
```

**Implementation Steps**:
1. Create an EventBridge Pipe with the SQS queue as source
2. Add a filter pattern to drop test orders
3. Configure Lambda enrichment to look up customer loyalty tier
4. Set Step Functions Express Workflow as the target
5. Attach an execution role with SQS, Lambda, and Step Functions permissions

**Exam Tips**:
- EventBridge Pipes = point-to-point with built-in enrichment; no polling Lambda needed
- Filter before enrichment saves Lambda invocations and cost
- Step Functions enrichment must be Express Workflow (synchronous) — not Standard
- Pipes vs Rules: Pipes = one source → one target; Rules = fan-out to many targets

---

### **Scenario 13: Centralised Multi-Account Access Management**
**Question**: A company has 40 AWS accounts across dev, staging, and production OUs. Developers need read-only access to all accounts and full access to dev accounts. The security team needs admin access to all accounts. Currently each account has separate IAM users. The company wants a single login experience with MFA enforced everywhere.

**Key Requirements**:
- Single sign-on across all 40 accounts
- Role-based access (developer vs security team)
- MFA enforced centrally
- No per-account IAM user management

**Solution Pattern**:
```
Corporate IdP (Okta/Azure AD)
    ↓ SAML 2.0
IAM Identity Center (management account)
    ↓
Permission Sets:
  - ReadOnly → all 40 accounts (developers)
  - PowerUser → dev OU accounts (developers)
  - AdminAccess → all 40 accounts (security team)
    ↓
AWS Access Portal (single login URL)
    ↓
Temporary credentials per account/role selection
```

**Implementation Steps**:
1. Enable IAM Identity Center in the management account
2. Connect to corporate IdP via SAML 2.0
3. Create three permission sets (ReadOnly, PowerUser, AdminAccess)
4. Assign permission sets to groups at the OU level via Organizations
5. Enable MFA enforcement in Identity Center — applies to all accounts automatically
6. Users log in at the Access Portal, select account + role

**Exam Tips**:
- IAM Identity Center = the answer for "single login across multiple accounts"
- Permission sets deploy as IAM roles automatically — no manual role creation per account
- MFA configured once in Identity Center applies everywhere — no per-account IAM MFA policy
- "AWS SSO" in older questions = IAM Identity Center — same service
- For CI/CD pipelines, use IAM roles or OIDC — not Identity Center
