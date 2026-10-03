# DOP-C02 Exam Cheatsheets - Last Night Review

## 📚 **Quick Access Cheatsheets**

### **Domain-Specific Cheatsheets**
- **[Domain 1: SDLC Automation (22%)](./Domain1-SDLC-Automation-Cheatsheet.md)** - CI/CD, CodePipeline, CodeBuild, CodeDeploy
- **[Domain 2: Infrastructure as Code (17%)](./Domain2-Infrastructure-as-Code-Cheatsheet.md)** - CloudFormation, CDK, Config, Systems Manager
- **[Domain 3: Resilient Cloud Solutions (15%)](./Domain3-Resilient-Cloud-Solutions-Cheatsheet.md)** - Auto Scaling, ELB, RDS Multi-AZ, Route 53
- **[Domain 4: Monitoring and Logging (15%)](./Domain4-Monitoring-Logging-Cheatsheet.md)** - CloudWatch, X-Ray, CloudTrail, Config
- **[Domain 5: Incident and Event Response (14%)](./Domain5-Incident-Event-Response-Cheatsheet.md)** - EventBridge, Lambda, SNS/SQS, Step Functions
- **[Domain 6: Security and Compliance (17%)](./Domain6-Security-Compliance-Cheatsheet.md)** - IAM, KMS, Secrets Manager, WAF, ACM

### **Exam Preparation**
- **[Final Exam Tips](./Final-Exam-Tips.md)** - Last-minute review, common traps, time management
- **[All Services Review](./All-Services-Review.md)** - Comprehensive service overview
- **[Exam Strategy](./Exam-Strategy.md)** - Test-taking strategies and tips
- **[Practice Scenarios](./Practice-Scenarios.md)** - Real exam-style scenarios with solutions

## 🎯 **How to Use These Cheatsheets**

### **The Night Before Exam**
1. **Start with [Final Exam Tips](./Final-Exam-Tips.md)** - Overview and strategy
2. **Review highest-weight domains first**: Domain 1 (22%) → Domain 2 (17%) → Domain 6 (17%)
3. **Focus on "Key Points" and "Common Mistakes"** sections in each cheatsheet
4. **Practice CLI commands** from each domain
5. **Review scenario patterns** and recognition triggers

### **Day of Exam**
1. **Quick scan of [Final Exam Tips](./Final-Exam-Tips.md)** (10 minutes)
2. **Review domain weights** and time allocation
3. **Mental checklist** of key concepts per domain

## ⚡ **Quick Reference**

### **Domain Weights**
- Domain 1: SDLC Automation - **22%** (14-15 questions)
- Domain 2: Infrastructure as Code - **17%** (11 questions)  
- Domain 3: Resilient Cloud Solutions - **15%** (10 questions)
- Domain 4: Monitoring and Logging - **15%** (10 questions)
- Domain 5: Incident and Event Response - **14%** (9 questions)
- Domain 6: Security and Compliance - **17%** (11 questions)

### **High-Impact Services** (Appear in Multiple Domains)
- **CloudFormation** - Domains 1, 2, 3
- **Lambda** - Domains 1, 4, 5
- **IAM** - Domains 1, 2, 6
- **CloudWatch** - Domains 3, 4, 5
- **Auto Scaling** - Domains 1, 3, 5

## 🔥 **Last-Minute Focus Areas**

### **Must Know Cold**
- Cross-account deployment patterns (IAM roles + trust policies)
- Blue/green vs rolling vs canary deployment differences
- Multi-AZ vs Read Replicas (RDS)
- EventBridge event routing and targets
- CloudFormation intrinsic functions (!Ref, !GetAtt, !Sub)

### **Common Exam Traps**
- Service confusion (CloudWatch vs CloudTrail vs Config)
- Deployment strategy selection
- Cross-account access methods
- Load balancer type selection (ALB vs NLB vs GWLB)
- IAM policy evaluation logic

## 💡 **Study Tips**

### **Effective Review Strategy**
1. **Skim entire cheatsheet** (5 minutes per domain)
2. **Deep dive on weak areas** (focus on "Common Mistakes")
3. **Practice scenario recognition** (read "Exam Scenarios")
4. **Review CLI commands** (muscle memory for syntax)
5. **Final mental walkthrough** of key concepts

### **Time Management**
- **Total exam time**: 180 minutes (3 hours)
- **Average per question**: 2.7 minutes
- **Strategy**: First pass (90 minutes) → Review flagged (60 minutes) → Final check (30 minutes)

## 🎯 **Success Checklist**

Before entering the exam, ensure you can quickly explain:
- [ ] Cross-account IAM role assumption process
- [ ] Difference between blue/green and rolling deployments  
- [ ] When to use ALB vs NLB vs GWLB
- [ ] Multi-AZ vs Read Replica use cases
- [ ] EventBridge event routing patterns
- [ ] CloudFormation stack dependencies and exports
- [ ] Lambda error handling and DLQ configuration
- [ ] Auto Scaling policy types and use cases

**You're ready! Trust your preparation and think practically. Good luck! 🚀**