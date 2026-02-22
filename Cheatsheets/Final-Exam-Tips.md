# DOP-C02 Final Exam Tips - Last Night Review

## 🎯 **Exam Format**
- **65 Questions** | **180 Minutes** | **Pass Score: 750/1000**
- **Multiple Choice** and **Multiple Response**
- **Scenario-based** questions with real-world situations

---

## ⚡ **Time Management**
- **2.7 minutes per question** average
- **Flag difficult questions** and return later
- **Read questions carefully** - look for key words
- **Eliminate wrong answers** first

---

## 🔥 **High-Impact Topics (Study These First)**

### **Domain 1 (22%) - CI/CD Automation**
- ✅ CodePipeline cross-account deployments
- ✅ Blue/green vs rolling deployments
- ✅ buildspec.yml and appspec.yml syntax
- ✅ CodeDeploy deployment configurations

### **Domain 3 (20%) - Resilient Solutions**
- ✅ Multi-AZ vs Read Replicas
- ✅ Auto Scaling policies and health checks
- ✅ Route 53 routing policies and failover
- ✅ Load balancer types and use cases

### **Domain 2 (20%) - Infrastructure as Code**
- ✅ CloudFormation intrinsic functions
- ✅ StackSets cross-account deployment
- ✅ Config rules and remediation
- ✅ Systems Manager Parameter Store

### **Domain 5 (18%) - Incident Response**
- ✅ EventBridge event patterns and routing
- ✅ Step Functions state types and error handling
- ✅ Lambda error handling and DLQ
- ✅ SNS/SQS integration patterns

---

## 🚨 **Common Exam Traps**

### **Service Confusion**
- **CloudWatch vs CloudTrail**: Metrics/Logs vs API Auditing
- **Config vs CloudTrail**: Configuration vs API calls
- **SNS vs SQS**: Push vs Pull messaging
- **ALB vs NLB**: Layer 7 vs Layer 4

### **Cross-Account Patterns**
- Always use **IAM roles**, never access keys
- **Trust policies** must allow source account
- **S3 bucket policies** for artifact sharing
- **KMS key policies** for encrypted resources

### **Deployment Strategies**
- **Blue/Green**: Zero downtime, full environment swap
- **Rolling**: Gradual replacement, some downtime
- **Canary**: Gradual traffic shift, risk mitigation
- **Immutable**: New instances, terminate old

---

## 📝 **Key Formulas & Numbers**

### **RDS**
- Multi-AZ failover: **60-120 seconds**
- Read replica lag: **Asynchronous**
- Backup retention: **0-35 days**

### **Lambda**
- Max execution time: **15 minutes**
- Default concurrency: **1000**
- Memory range: **128MB - 10,240MB**

### **Auto Scaling**
- Health check grace period: **300 seconds default**
- Cooldown period: **300 seconds default**
- Scale-out: **Faster** | Scale-in: **Slower**

### **CloudFormation**
- Template size limit: **51,200 characters**
- Stack limit per region: **200**
- Parameter limit: **200**

---

## 🎯 **Scenario Recognition Patterns**

### **"Zero Downtime Deployment"** → Blue/Green
### **"Cross-Account Access"** → IAM Roles + Trust Policies
### **"High Availability Database"** → RDS Multi-AZ
### **"Global Application"** → Route 53 + CloudFront + Multi-Region
### **"Automated Compliance"** → Config Rules + Remediation
### **"Event-Driven Architecture"** → EventBridge + Lambda
### **"Secure Secrets"** → Secrets Manager + Rotation
### **"Web Application Security"** → WAF + Shield + ACM

---

## ⚠️ **Last-Minute Gotchas**

### **IAM**
- Root user ≠ Administrator user
- Explicit deny always wins
- Cross-account requires trust policy

### **KMS**
- Customer managed keys cost $1/month
- AWS managed keys are free
- Multi-region keys have same key ID

### **S3**
- Cross-region replication requires versioning
- Objects existing before CRR setup are NOT replicated
- Same-region replication for compliance

### **CloudFormation**
- Outputs can be exported for cross-stack reference
- DependsOn for explicit dependencies
- DeletionPolicy: Retain/Snapshot/Delete

---

## 🔧 **CLI Command Patterns**

### **Always Remember**
```bash
# Assume role pattern
aws sts assume-role --role-arn ROLE_ARN --role-session-name SESSION_NAME

# CloudFormation with parameters
aws cloudformation create-stack --stack-name NAME --template-body file://template.yaml --parameters file://params.json

# CodePipeline execution
aws codepipeline start-pipeline-execution --name PIPELINE_NAME

# Auto Scaling policy
aws autoscaling put-scaling-policy --auto-scaling-group-name ASG_NAME --policy-name POLICY_NAME
```

---

## 🎯 **Final 30-Minute Review**

### **Read These Sections**
1. Domain weights and key services
2. Cross-account deployment patterns
3. Blue/green deployment steps
4. Multi-AZ vs Read Replica differences
5. EventBridge event routing patterns
6. IAM policy evaluation logic
7. Common service integration patterns

### **Mental Checklist**
- ✅ Can I explain cross-account IAM roles?
- ✅ Do I know when to use each deployment strategy?
- ✅ Can I differentiate between AWS services?
- ✅ Do I understand event-driven architectures?
- ✅ Can I design resilient multi-AZ solutions?

---

## 💪 **You've Got This!**

**Remember**: The exam tests **practical knowledge** and **real-world scenarios**. Trust your preparation and think like a DevOps engineer solving actual problems.

**Good Luck! 🚀**