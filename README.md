# AWS Certified DevOps Engineer – Professional (DOP-C02) Study Notes

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Exam: DOP-C02](https://img.shields.io/badge/Exam-DOP--C02-orange.svg)](https://aws.amazon.com/certification/certified-devops-engineer-professional/)

## 🎯 **Complete Exam Preparation Repository**

> **Free, comprehensive, and exam-focused study materials for AWS Certified DevOps Engineer – Professional (DOP-C02) certification. Created with Amazon Q Developer and covering 50+ AWS services across all 6 exam domains.**

This repository contains comprehensive study materials for the AWS Certified DevOps Engineer – Professional (DOP-C02) exam, organized by exam domains with detailed service coverage, integration patterns, and practical examples.

### **📊 Repository Stats**
- **50 Service Files** with comprehensive coverage
- **10 Deep-Dive Topics** for advanced concepts
- **10 Exam Cheatsheets** for quick review
- **10 Practice Scenarios** with real exam-style questions
- **100% Free** and open-source

## 📚 **Repository Structure**

### **📁 Domain-Specific Study Materials**

#### **[Domain 1: SDLC Automation (22%)](./Domain-1-SDLC-Automation/)**
- **Core Services**: CodePipeline, CodeBuild, CodeDeploy, EventBridge, AppConfig, CodeArtifact
- **Source Control**: GitHub/GitLab via CodeStar connections (CodeCommit status uncertain — verify before exam)
- **Integration Patterns**: Cross-account deployments, GitHub Actions OIDC, third-party integrations
- **Deep Dives**: CodePipeline V1 vs V2, cross-account patterns, service integrations
- **Coverage**: CI/CD pipelines, automated testing, deployment strategies

#### **[Domain 2: Infrastructure as Code (17%)](./Domain-2-Configuration-Management-Infrastructure-as-Code/)**
- **Core Services**: CloudFormation, CDK, SAM, Terraform, Systems Manager
- **Governance**: Organizations, Control Tower, Service Catalog, Config
- **Deep Dives**: CloudFormation macros, drift detection, advanced features
- **Coverage**: IaC best practices, configuration management, governance

#### **[Domain 3: Resilient Cloud Solutions (15%)](./Domain-3-Resilient-Cloud-Solutions/)**
- **Core Services**: Auto Scaling, ELB, RDS Multi-AZ, Route 53, S3 CRR, EKS, ECS, AWS Backup, ElastiCache
- **Patterns**: High availability, disaster recovery, multi-region architectures
- **Missing**: Global Accelerator, RDS Blue/Green Deployments, AWS Resilience Hub *(planned — see Update Plan)*
- **Coverage**: Resilience design, fault tolerance, scalability patterns

#### **[Domain 4: Monitoring and Logging (15%)](./Domain-4-Monitoring-Logging/)**
- **Core Services**: CloudWatch, CloudTrail, X-Ray, Config, GuardDuty, Inspector
- **Deep Dives**: Cross-account observability, metrics insights
- **Coverage**: Comprehensive monitoring, security monitoring, compliance

#### **[Domain 5: Incident and Event Response (14%)](./Domain-5-Incident-Event-Response/)**
- **Core Services**: EventBridge, EventBridge Scheduler, Lambda, SNS/SQS, Step Functions, CloudWatch Alarms, Incident Manager, ChatOps
- **Patterns**: Event-driven architectures, automated response, workflow orchestration
- **Missing**: EventBridge Pipes *(planned — see Update Plan)*
- **Coverage**: Incident detection, automated remediation, notification systems

#### **[Domain 6: Security and Compliance (17%)](./Domain-6-Security-Compliance/)**
- **Core Services**: IAM, KMS, Secrets Manager, WAF/Shield, Certificate Manager, Macie
- **Governance**: Compliance frameworks, security automation, governance patterns
- **Missing**: IAM Identity Center, Resource Control Policies (RCPs), Audit Manager *(planned — see Update Plan)*
- **Coverage**: Security best practices, compliance monitoring, access management

### **📁 [Cheatsheets](./Cheatsheets/) - Last-Minute Review Materials**
- **[All Services Review](./Cheatsheets/All-Services-Review.md)** - Comprehensive service reference
- **[Domain-Specific Cheatsheets](./Cheatsheets/)** - Quick review for each domain
- **[Final Exam Tips](./Cheatsheets/Final-Exam-Tips.md)** - Last-night preparation guide
- **[Exam Strategy](./Cheatsheets/Exam-Strategy.md)** - Test-taking strategies and time management

## 🚀 **Quick Start Guide**

### **For Comprehensive Study (4-6 weeks)**
1. **Week 1-2**: Study Domain 1 (SDLC) and Domain 2 (IaC) - Highest weight domains
2. **Week 3**: Study Domain 3 (Resilience) and Domain 5 (Incident Response)
3. **Week 4**: Study Domain 4 (Monitoring) and Domain 6 (Security)
4. **Week 5-6**: Review cheatsheets, practice scenarios, take practice exams

### **For Review/Refresh (1-2 weeks)**
1. **Days 1-7**: Review all domain cheatsheets and identify weak areas
2. **Days 8-10**: Deep dive into weak areas using detailed service files
3. **Days 11-14**: Final review using [All Services Review](./Cheatsheets/All-Services-Review.md) and [Final Exam Tips](./Cheatsheets/Final-Exam-Tips.md)

### **Last-Night Preparation (2-3 hours)**
1. **[Final Exam Tips](./Cheatsheets/Final-Exam-Tips.md)** (30 minutes)
2. **[Exam Strategy](./Cheatsheets/Exam-Strategy.md)** (30 minutes)
3. **[All Services Review](./Cheatsheets/All-Services-Review.md)** (60 minutes)
4. **Domain-specific cheatsheets** for weak areas (30 minutes)

## 📊 **Exam Information**

### **Exam Details**
- **Duration**: 180 minutes (3 hours)
- **Questions**: 75 questions (65 scored + 10 unscored; multiple choice and multiple response)
- **Passing Score**: 750/1000 (approximately 75%)
- **Cost**: $300 USD
- **Validity**: 3 years

### **Domain Weights**
| Domain | Weight | Est. Questions | Study Priority |
|--------|--------|----------------|----------------|
| 1. SDLC Automation | 22% | 14-15 | High |
| 2. Infrastructure as Code | 17% | 11 | High |
| 3. Resilient Cloud Solutions | 15% | 10 | Medium |
| 4. Monitoring and Logging | 15% | 10 | Medium |
| 5. Incident and Event Response | 14% | 9 | Medium |
| 6. Security and Compliance | 17% | 11 | High |

## 🎯 **Key Features of This Repository**

### **Comprehensive Coverage**
- **50+ AWS services** covered with exam-focused details
- **Integration patterns** between services with practical examples
- **Real-world scenarios** with step-by-step solutions
- **CLI commands** with common flags and use cases
- **Architecture diagrams** and decision trees

### **Exam-Focused Content**
- **Common exam scenarios** identified and explained
- **Service comparison matrices** for decision-making
- **Troubleshooting guides** for common issues
- **Best practices** aligned with AWS Well-Architected Framework
- **Security patterns** and compliance considerations

### **Multiple Learning Formats**
- **Detailed service guides** for comprehensive understanding
- **Deep-dive topics** for advanced concepts
- **Quick reference cheatsheets** for rapid review
- **Practical examples** with code snippets
- **Visual diagrams** for architecture understanding

## 🔥 **High-Impact Study Areas**

### **Must-Know Services (Appear in Multiple Domains)**
1. **CloudFormation** - IaC foundation, appears in Domains 1, 2, 3
2. **Lambda** - Event processing, appears in Domains 1, 4, 5
3. **IAM** - Security foundation, appears in Domains 1, 2, 6
4. **CloudWatch** - Monitoring foundation, appears in Domains 3, 4, 5
5. **Auto Scaling** - Resilience foundation, appears in Domains 1, 3, 5
6. **EventBridge** - Event routing foundation, appears in Domains 1, 4, 5
7. **Step Functions** - Workflow orchestration, appears in Domains 1, 5

### **Critical Integration Patterns**
- **Cross-account deployments** - IAM roles + trust policies + resource sharing
- **Event-driven architectures** - EventBridge + Lambda + SNS/SQS
- **CI/CD pipelines** - CodePipeline + CodeBuild + CodeDeploy + monitoring
- **Multi-region resilience** - Route 53 + replication + failover
- **Security automation** - Config + Lambda + remediation actions

### **Common Exam Scenarios**
1. **Design CI/CD pipeline with cross-account deployment**
2. **Implement blue/green deployment strategy**
3. **Create event-driven incident response system**
4. **Design multi-region disaster recovery architecture**
5. **Implement automated compliance monitoring**
6. **Configure auto scaling with custom metrics**
7. **Set up centralized logging and monitoring**
8. **Design secure secrets management solution**

## 📖 **How to Use This Repository**

### **Study Approach**
1. **Start with domain README files** to understand scope and key concepts
2. **Study individual service files** for detailed understanding
3. **Review deep-dive topics** for advanced concepts
4. **Practice with cheatsheets** for quick recall
5. **Use exam strategy guide** for test-taking preparation

### **File Organization**
```
Domain-X-Name/
├── README.md                    # Domain overview and navigation
├── ServiceName/                 # Individual service directories
│   ├── ServiceName-General.md   # Comprehensive service guide
│   └── ServiceName-DeepDive-*.md # Advanced topics
├── ServiceName-General.md       # Standalone service files
└── Integration-Pattern-*.md     # Cross-service patterns
```

### **Content Structure (Each Service File)**
1. **Overview** - Service purpose and key characteristics
2. **Core Concepts** - Fundamental concepts and components
3. **Configuration Examples** - YAML/JSON/CLI examples
4. **IAM Permissions** - Required roles and policies
5. **Integration Patterns** - How it works with other services
6. **Security Best Practices** - Security considerations
7. **Monitoring and Troubleshooting** - Operational guidance
8. **Common Exam Scenarios** - Practical exam questions
9. **CLI Commands** - Essential command reference
10. **Best Practices** - AWS recommendations
11. **Exam Tips** - Key points for exam success

## 🎓 **Prerequisites and Recommendations**

### **Required Experience**
- **2+ years** of AWS experience in DevOps role
- **Hands-on experience** with CI/CD pipelines
- **Infrastructure as Code** experience (CloudFormation/Terraform)
- **Monitoring and logging** implementation experience
- **AWS security** best practices knowledge

### **Recommended Preparation**
- **AWS Solutions Architect Associate** certification (helpful but not required)
- **Hands-on labs** with AWS services covered in exam
- **Practice exams** from reputable providers
- **AWS documentation** review for key services
- **AWS whitepapers** on DevOps and Well-Architected Framework

## 🔗 **Additional Resources**

### **Official AWS Resources**
- [AWS Certified DevOps Engineer – Professional Exam Guide](https://aws.amazon.com/certification/certified-devops-engineer-professional/)
- [AWS DevOps Best Practices](https://aws.amazon.com/devops/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)

### **Practice and Training**
- [AWS Skill Builder](https://skillbuilder.aws/) - Free AWS training
- [AWS Workshops](https://workshops.aws/) - Hands-on labs
- [AWS Samples](https://github.com/aws-samples) - Code examples
- [AWS Solutions Library](https://aws.amazon.com/solutions/) - Reference architectures

### **Community Resources**
- [AWS re:Invent Sessions](https://reinvent.awsevents.com/) - Latest AWS updates
- [AWS Blogs](https://aws.amazon.com/blogs/) - Technical deep dives
- [AWS re:Post](https://repost.aws/) - Community Q&A (replaced AWS Forums)
- [AWS User Groups](https://aws.amazon.com/developer/community/usergroups/) - Local communities

## 📝 **Study Tips**

### **Effective Study Strategies**
1. **Hands-on practice** - Build actual solutions, don't just read
2. **Scenario-based learning** - Focus on real-world use cases
3. **Integration understanding** - Learn how services work together
4. **Cost optimization** - Always consider cost implications
5. **Security first** - Security should be built-in, not bolted-on

### **Time Management**
- **Allocate time by domain weight** - More time for higher-weight domains
- **Focus on weak areas** - Identify and strengthen knowledge gaps
- **Regular review** - Spaced repetition for better retention
- **Practice exams** - Simulate real exam conditions
- **Final review** - Use cheatsheets for last-minute preparation

### **Common Mistakes to Avoid**
- **Memorizing without understanding** - Focus on concepts, not facts
- **Ignoring integration patterns** - Services rarely work in isolation
- **Overlooking security** - Security is embedded in every domain
- **Neglecting cost considerations** - Cost optimization is always important
- **Rushing through scenarios** - Take time to understand requirements

## 🏆 **Success Indicators**

### **You're Ready When You Can:**
- [ ] Design end-to-end CI/CD pipelines with cross-account deployment
- [ ] Explain the difference between blue/green, rolling, and canary deployments
- [ ] Configure Auto Scaling policies for different scenarios
- [ ] Design event-driven architectures using EventBridge and Lambda
- [ ] Implement multi-region disaster recovery strategies
- [ ] Set up comprehensive monitoring and alerting systems
- [ ] Design secure, compliant AWS architectures
- [ ] Troubleshoot common DevOps issues and implement solutions

### **Final Confidence Check**
- [ ] Completed all domain study materials
- [ ] Reviewed all cheatsheets multiple times
- [ ] Practiced with sample questions and scenarios
- [ ] Comfortable with CLI commands for key services
- [ ] Understand service integration patterns
- [ ] Can explain AWS best practices and design principles
- [ ] Ready to apply practical DevOps knowledge in exam scenarios

## 🤝 **Contributing**

Contributions are welcome! Please read our [Contributing Guidelines](CONTRIBUTING.md) before submitting pull requests.

### **Ways to Contribute**
- 🐛 Report bugs or inaccuracies
- 📝 Improve documentation
- ✨ Add new content or examples
- 💡 Share exam experiences (without violating NDA)
- ⭐ Star this repository if you find it helpful!

## 📄 **License**

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 **Acknowledgments**

- Created with [Amazon Q Developer](https://aws.amazon.com/q/developer/)
- Inspired by the AWS DevOps community
- Based on official AWS documentation and best practices

## 📞 **Contact & Feedback**

- **Issues**: [GitHub Issues](../../issues)
- **Discussions**: [GitHub Discussions](../../discussions)
- **LinkedIn**: Share your success stories!

## ⭐ **Show Your Support**

If this repository helped you pass the DOP-C02 exam, please:
- ⭐ Star this repository
- 🔄 Share it with others preparing for the exam
- 💬 Share your success story on LinkedIn
- 🤝 Contribute improvements back to the community

## 🎯 **Repository Maintenance**

This repository is actively maintained and updated based on:
- **AWS service updates** and new features
- **Exam blueprint changes** and feedback
- **Community contributions** and suggestions
- **Real exam experiences** and patterns

---

## 🗺️ **Update Plan**

> Last reviewed: October 2026 (Muse AI review + Amazon Q corrections applied).
> Items below are confirmed gaps — not yet in the repo. Tracked here so nothing gets lost.

### Phase 1 — High Priority ✅ Done

| # | File | Domain | Status |
|---|---|---|---|
| 1 | `Domain-6-Security-Compliance/IAM-Identity-Center-General.md` | 6 (17%) | ✅ Added |
| 2 | `Domain-6-Security-Compliance/Resource-Control-Policies-General.md` | 6 (17%) | ✅ Added |
| 3 | `Domain-3-Resilient-Cloud-Solutions/Global-Accelerator-General.md` | 3 (15%) | ✅ Added |

### Phase 2 — Medium Priority ✅ Done

| # | File | Domain | Status |
|---|---|---|---|
| 4 | `Domain-3-Resilient-Cloud-Solutions/RDS-BlueGreen-General.md` | 3 (15%) | ✅ Added |
| 5 | `Domain-5-Incident-Event-Response/EventBridge-Pipes-General.md` | 5 (14%) | ✅ Added |
| 6 | `Domain-4-Monitoring-Logging/CloudWatch-Synthetics-General.md` | 4 (15%) | ✅ Added |
| 7 | `Domain-2-Configuration-Management-Infrastructure-as-Code/CDK-Pipelines-General.md` | 2 (17%) | ✅ Added |
| 8 | `Domain-1-SDLC-Automation/GitHub-Actions-Integration/GitHub-OIDC-AWS.md` | 1 (22%) | ✅ Added |

### Phase 3 — Lower Priority (nice to have) ✅ Done

| # | File | Domain | Status |
|---|---|---|---|
| 9 | `Domain-6-Security-Compliance/Audit-Manager-General.md` | 6 (17%) | ✅ Added |
| 10 | `Domain-3-Resilient-Cloud-Solutions/Resilience-Hub-General.md` | 3 (15%) | ✅ Added |
| 11 | `DEPRECATED-SERVICES.md` | structural | ✅ Added |
| 12 | `CHANGELOG.md` | structural | ✅ Added |

### Known content corrections still needed in existing files

| File | Issue |
|---|---|
| `Domain-4-Monitoring-Logging/CloudWatch/CLOUDWATCH-General.md` | ✅ Application Signals and Internet Monitor added |
| `Cheatsheets/All-Services-Review.md` | ✅ IAM Identity Center, RCPs, EventBridge Pipes, Macie, Audit Manager added; OpsWorks removed |
| `Cheatsheets/Practice-Scenarios.md` | ✅ Scenarios 11–13 added (RDS blue/green, EventBridge Pipes, IAM Identity Center) |
| `Domain-6-Security-Compliance/IAM-General.md` | ✅ Section 12 updated to IAM Identity Center with link to dedicated file |

> See [DEPRECATED-SERVICES.md](./DEPRECATED-SERVICES.md) for the single reference on retired/restricted services (OpsWorks, CodeStar, CodeCatalyst, CodeCommit).
> See [CHANGELOG.md](./CHANGELOG.md) for a full history of what was reviewed and when.

---

**Good luck with your DOP-C02 exam preparation! Remember: practical experience combined with structured study is the key to success. Focus on understanding concepts and integration patterns rather than memorizing facts. You've got this! 🚀**