# Domain 6: Security and Compliance - DOP-C02 Study Notes

## Overview
This domain covers security and compliance aspects of AWS DevOps practices, focusing on implementing security controls, managing compliance requirements, and ensuring governance across AWS environments.

## Domain Weight: 17% of exam

## Key Topics Covered

### 6.1 Security Controls and Compliance
- Identity and Access Management (IAM)
- AWS Key Management Service (KMS)
- AWS Secrets Manager
- AWS Certificate Manager (ACM)
- AWS WAF and Shield

### 6.2 Governance and Compliance Frameworks
- AWS Organizations
- AWS Control Tower
- AWS Config
- Compliance monitoring and reporting
- Regulatory compliance (GDPR, HIPAA, PCI DSS, SOC)

### 6.3 Security Automation
- Automated security scanning
- Compliance monitoring
- Incident response automation
- Security remediation workflows

## Files in this Domain

### Core Security Services
- **[IAM-General.md](./IAM-General.md)** - Identity and Access Management comprehensive guide
- **[IAM-Identity-Center-General.md](./IAM-Identity-Center-General.md)** - Workforce SSO, permission sets, multi-account access
- **[Resource-Control-Policies-General.md](./Resource-Control-Policies-General.md)** - RCPs: restrict what can be done to resources org-wide
- **[KMS-General.md](./KMS-General.md)** - Key Management Service for encryption
- **[Secrets-Manager-General.md](./Secrets-Manager-General.md)** - Secrets management and rotation
- **[Certificate-Manager-General.md](./Certificate-Manager-General.md)** - SSL/TLS certificate management
- **[WAF-Shield-General.md](./WAF-Shield-General.md)** - Web Application Firewall and DDoS protection
- **[Macie-General.md](./Macie-General.md)** - Sensitive data discovery in S3
- **[Audit-Manager-General.md](./Audit-Manager-General.md)** - Automated compliance evidence collection

### Governance and Compliance
- **[Compliance-Governance-General.md](./Compliance-Governance-General.md)** - Compliance frameworks and governance patterns

## Key Concepts

### Security Best Practices
- **Least Privilege Access** - Grant minimum necessary permissions
- **Defense in Depth** - Multiple layers of security controls
- **Encryption Everywhere** - Data at rest and in transit
- **Continuous Monitoring** - Real-time security monitoring
- **Automated Response** - Automated incident response and remediation

### Compliance Requirements
- **Data Protection** - GDPR, CCPA compliance
- **Healthcare** - HIPAA compliance
- **Financial Services** - PCI DSS compliance
- **Government** - FedRAMP compliance
- **Industry Standards** - SOC 1/2/3, ISO 27001

### Governance Patterns
- **Multi-Account Strategy** - Account separation for security
- **Centralized Logging** - Audit trails and compliance
- **Policy Enforcement** - Service Control Policies (SCPs)
- **Access Reviews** - Regular permission audits
- **Incident Response** - Structured response procedures

## Integration Patterns

### CI/CD Security Integration
- Security scanning in pipelines
- Secrets management in deployments
- Compliance validation in builds
- Automated security testing

### Multi-Account Security
- Cross-account role assumptions
- Centralized security monitoring
- Shared security services
- Compliance across accounts

### Automation and Orchestration
- Security event response
- Compliance remediation
- Access provisioning
- Certificate management

## Monitoring and Alerting

### Security Monitoring
- AWS CloudTrail for audit logs
- AWS Config for compliance monitoring
- Amazon GuardDuty for threat detection
- AWS Security Hub for centralized findings

### Compliance Reporting
- Automated compliance dashboards
- Regular compliance assessments
- Audit trail generation
- Violation alerting and remediation

## Common Exam Scenarios

1. **Implement least privilege access** - IAM policies and roles
2. **Encrypt data at rest and in transit** - KMS and certificate management
3. **Manage secrets securely** - Secrets Manager integration
4. **Protect web applications** - WAF rules and Shield protection
5. **Ensure compliance monitoring** - Config rules and remediation
6. **Implement governance controls** - Organizations and Control Tower
7. **Automate security responses** - Lambda-based automation
8. **Manage certificates** - ACM integration with services
9. **Cross-account security** - Role assumptions and policies
10. **Incident response procedures** - Automated response workflows

## Study Tips

### Focus Areas
- Understand IAM policy evaluation logic
- Know KMS key types and use cases
- Understand Secrets Manager rotation patterns
- Know WAF rule types and integration
- Understand compliance frameworks and requirements

### Hands-on Practice
- Create IAM policies with conditions
- Set up KMS keys with cross-account access
- Configure Secrets Manager with rotation
- Deploy WAF rules for web applications
- Set up Config rules for compliance monitoring

### Integration Knowledge
- Security service integrations with other AWS services
- Cross-account security patterns
- Automation patterns for security and compliance
- Monitoring and alerting configurations

## Additional Resources

### AWS Documentation
- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [AWS Compliance Programs](https://aws.amazon.com/compliance/programs/)
- [AWS Security Hub User Guide](https://docs.aws.amazon.com/securityhub/)

### Compliance Frameworks
- [AWS GDPR Center](https://aws.amazon.com/compliance/gdpr-center/)
- [AWS HIPAA Compliance](https://aws.amazon.com/compliance/hipaa-compliance/)
- [AWS PCI DSS Compliance](https://aws.amazon.com/compliance/pci-dss-level-1-faqs/)

### Security Tools
- [AWS Config Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [AWS Security Hub Findings](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-findings.html)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)