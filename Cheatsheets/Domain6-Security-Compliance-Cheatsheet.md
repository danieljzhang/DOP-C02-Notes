# Domain 6: Security and Compliance - DOP-C02 Cheatsheet

## Weight: 17% | Focus: IAM, KMS, Secrets Manager, WAF, Certificate Manager

---

## 🔥 **MUST KNOW Services**

### **IAM** - Identity & Access Management
- **Users**: Long-term credentials, MFA
- **Roles**: Temporary credentials, cross-account access
- **Policies**: JSON documents, least privilege
- **Groups**: Collection of users with shared permissions

### **KMS** - Key Management Service
- **CMK**: Customer Managed Keys ($1/month)
- **AWS Managed**: Free, automatic rotation
- **Envelope Encryption**: Data keys for large objects
- **Cross-Region**: Multi-region keys available

### **Secrets Manager** - Secrets Storage & Rotation
- **Automatic Rotation**: RDS, Aurora, DocumentDB, Redshift
- **Cross-Region Replication**: Disaster recovery
- **Integration**: Lambda, ECS, RDS
- **Versioning**: AWSCURRENT, AWSPENDING, AWSPREVIOUS

### **WAF** - Web Application Firewall
- **Rule Types**: Rate-based, IP, Geo, String match
- **Managed Rules**: AWS and marketplace rules
- **Integration**: CloudFront, ALB, API Gateway
- **Logging**: CloudWatch Logs, S3, Kinesis

### **Certificate Manager** - SSL/TLS Certificates
- **Public Certs**: Free for AWS services
- **Private CA**: Additional charges
- **Validation**: DNS (automated), Email
- **Integration**: CloudFront, ALB, API Gateway

---

## ⚡ **Key Patterns**

### **Cross-Account Access**
```
Source Account → Assume Role → Target Account Role → Access Resources
```

### **Secrets Rotation**
```
Secrets Manager → Lambda Rotation Function → Database → Update Secret
```

### **Encryption at Rest**
```
Application → KMS Data Key → Encrypt Data → Store Encrypted Data + Encrypted Key
```

---

## 🎯 **Exam Scenarios**

### **Scenario 1: Cross-account S3 access**
- Create IAM role in target account
- Configure trust policy for source account
- Grant S3 permissions to role
- Source account assumes role to access S3

### **Scenario 2: Database password rotation**
- Store credentials in Secrets Manager
- Enable automatic rotation with Lambda
- Update application to retrieve from Secrets Manager
- Monitor rotation success with CloudWatch

### **Scenario 3: Web application security**
- Deploy WAF with managed rule groups
- Configure rate limiting and geo-blocking
- Set up SSL termination with ACM
- Monitor attacks with CloudWatch metrics

---

## 📋 **Quick Commands**

```bash
# IAM
aws iam create-role --role-name MyRole --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name MyRole --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/MyRole --role-session-name MySession

# KMS
aws kms create-key --description "My encryption key"
aws kms encrypt --key-id alias/my-key --plaintext "Hello World"
aws kms decrypt --ciphertext-blob fileb://encrypted-data

# Secrets Manager
aws secretsmanager create-secret --name MySecret --secret-string '{"username":"admin","password":"secret"}'
aws secretsmanager get-secret-value --secret-id MySecret
aws secretsmanager rotate-secret --secret-id MySecret

# WAF
aws wafv2 create-web-acl --name MyWebACL --scope REGIONAL --default-action Allow={}
aws wafv2 associate-web-acl --web-acl-arn MyWebACLArn --resource-arn MyALBArn

# Certificate Manager
aws acm request-certificate --domain-name example.com --validation-method DNS
aws acm list-certificates
```

---

## 🔐 **IAM Policy Evaluation Logic**

### **Order of Evaluation**
1. **Explicit Deny** - Always wins
2. **Explicit Allow** - Required for access
3. **Default Deny** - No explicit allow = deny

### **Policy Types**
- **Identity-based**: Attached to users, groups, roles
- **Resource-based**: Attached to resources (S3, KMS)
- **Permission Boundaries**: Maximum permissions
- **SCPs**: Organization-level guardrails

---

## 🔑 **KMS Key Types & Usage**

### **Symmetric Keys** (AES-256)
- Same key for encrypt/decrypt
- Never leaves AWS unencrypted
- Used for envelope encryption

### **Asymmetric Keys**
- Key pair (public/private)
- Public key can be downloaded
- Used for encrypt/decrypt or sign/verify

### **Key Rotation**
- **Automatic**: Yearly for customer managed keys
- **Manual**: Create new key, update applications

---

## 🔄 **Secrets Manager Rotation**

### **Supported Services**
- RDS (MySQL, PostgreSQL, Oracle, SQL Server)
- Aurora (MySQL, PostgreSQL)
- DocumentDB, Redshift

### **Rotation Process**
1. **Create**: Generate new secret version
2. **Set**: Update service with new credentials
3. **Test**: Verify new credentials work
4. **Finish**: Mark new version as current

---

## 🛡️ **WAF Rule Types**

### **Rate-based Rules**
- Limit requests per 5-minute window
- Automatic blocking when exceeded
- Configurable rate limit

### **Managed Rule Groups**
- **Core Rule Set**: OWASP Top 10
- **Known Bad Inputs**: Common attack patterns
- **SQL Database**: SQL injection protection

---

## 📜 **Certificate Manager**

### **Validation Methods**
- **DNS**: Add CNAME record (recommended)
- **Email**: Click link in validation email

### **Certificate Types**
- **Public**: Free for AWS services
- **Private**: Issued by AWS Private CA (charged)
- **Imported**: Third-party certificates

---

## ⚠️ **Common Mistakes**
- Using root user for daily operations
- Not enabling MFA for privileged accounts
- Hardcoding credentials in code
- Not rotating access keys regularly
- Overly permissive IAM policies
- Not monitoring failed authentication attempts
- Forgetting to validate certificates before expiration

---

## 🔑 **Key Points**
- **IAM** is global, policies are JSON
- **Roles** are preferred over users for applications
- **KMS** keys are regional, multi-region keys available
- **Secrets Manager** charges per secret per month
- **WAF** operates at Layer 7 (application layer)
- **ACM** certificates are free for AWS services
- **Cross-account** access requires proper trust policies