# AWS IAM (Identity and Access Management) - DOP-C02 Study Notes

## 1. Overview

### What is IAM?
- Web service for securely controlling access to AWS resources
- Centralized management of users, groups, roles, and permissions
- Global service (not region-specific)
- Free service - no additional charges

### Key Components
- **Users** - Individual identities for people or applications
- **Groups** - Collections of users with shared permissions
- **Roles** - Temporary credentials for services or cross-account access
- **Policies** - JSON documents defining permissions
- **Identity Providers** - External identity systems (SAML, OIDC)

### Core Principles
- **Least Privilege** - Grant minimum permissions needed
- **Defense in Depth** - Multiple layers of security
- **Principle of Separation** - Separate duties and responsibilities
- **Regular Auditing** - Monitor and review access patterns

---

## 2. Users and Groups

### IAM Users
- Permanent identity with long-term credentials
- Used for people or applications needing AWS access
- Maximum 5,000 users per AWS account
- Can have access keys, passwords, MFA devices

### User Creation
```bash
# Create user
aws iam create-user --user-name john-doe

# Create access key
aws iam create-access-key --user-name john-doe

# Set password
aws iam create-login-profile --user-name john-doe --password MyPassword123!
```

### IAM Groups
- Collection of users with shared permissions
- Users inherit group permissions
- Maximum 300 groups per account
- Users can belong to multiple groups (max 10)

### Group Management
```bash
# Create group
aws iam create-group --group-name developers

# Add user to group
aws iam add-user-to-group --user-name john-doe --group-name developers

# Attach policy to group
aws iam attach-group-policy --group-name developers --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
```

---

## 3. IAM Roles

### What are Roles?
- Temporary security credentials
- No permanent credentials (no access keys)
- Assumed by users, applications, or services
- Cross-account access mechanism

### Role Types
- **Service Roles** - For AWS services (EC2, Lambda, etc.)
- **Cross-Account Roles** - Access between AWS accounts
- **Identity Provider Roles** - For federated users
- **Instance Profiles** - Container for EC2 service roles

### Role Creation
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Assume Role Process
```bash
# Assume role
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name MySession

# Use temporary credentials
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
```

---

## 4. IAM Policies

### Policy Types
- **AWS Managed Policies** - Created and maintained by AWS
- **Customer Managed Policies** - Created by you
- **Inline Policies** - Embedded directly in user/group/role

### Policy Structure
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### Policy Elements
- **Version** - Policy language version (2012-10-17)
- **Statement** - Array of permission statements
- **Effect** - Allow or Deny
- **Action** - API actions (s3:GetObject)
- **Resource** - ARN of resources
- **Principal** - Who the policy applies to
- **Condition** - When the policy applies

### Policy Evaluation Logic
1. **Explicit Deny** - Always wins
2. **Explicit Allow** - Required for access
3. **Default Deny** - No explicit allow = deny

---

## 5. Multi-Factor Authentication (MFA)

### MFA Device Types
- **Virtual MFA** - Smartphone apps (Google Authenticator, Authy)
- **Hardware MFA** - Physical tokens (YubiKey, Gemalto)
- **SMS MFA** - Text message (not recommended for root)

### Enable MFA
```bash
# Create virtual MFA device
aws iam create-virtual-mfa-device --virtual-mfa-device-name MyMFA --outfile QRCode.png

# Enable MFA for user
aws iam enable-mfa-device \
  --user-name john-doe \
  --serial-number arn:aws:iam::123456789012:mfa/MyMFA \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

### MFA Policy Enforcement
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

---

## 6. Identity Federation

### Federation Types
- **SAML 2.0** - Enterprise identity providers
- **OpenID Connect (OIDC)** - Web identity providers
- **Custom Identity Broker** - Your own authentication system

### SAML Federation
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/MyProvider"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

### Web Identity Federation
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/accounts.google.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "accounts.google.com:aud": "my-app-id"
        }
      }
    }
  ]
}
```

---

## 7. Cross-Account Access

### Cross-Account Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

### External ID Pattern
- Additional security for cross-account access
- Prevents confused deputy problem
- Shared secret between accounts

### Cross-Account Access Steps
1. Create role in target account
2. Define trust policy with source account
3. Attach permissions policy to role
4. Source account assumes role
5. Use temporary credentials

---

## 8. IAM Best Practices

### Security Best Practices
- Enable MFA for all users
- Use roles instead of users for applications
- Rotate access keys regularly
- Use least privilege principle
- Enable CloudTrail logging
- Regular access reviews

### Operational Best Practices
- Use groups for permission management
- Use AWS managed policies when possible
- Tag IAM resources for organization
- Use policy conditions for fine-grained control
- Implement break-glass procedures

### Development Best Practices
- Use IAM roles for EC2 instances
- Use temporary credentials in applications
- Don't embed credentials in code
- Use AWS SDKs for credential management
- Implement proper error handling

---

## 9. IAM Conditions

### Common Condition Keys
- **aws:RequestedRegion** - Restrict by region
- **aws:SourceIp** - Restrict by IP address
- **aws:CurrentTime** - Time-based access
- **aws:userid** - User identifier
- **aws:username** - User name

### IP Address Restriction
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
        }
      }
    }
  ]
}
```

### Time-Based Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        }
      }
    }
  ]
}
```

---

## 10. IAM Access Analyzer

### What is Access Analyzer?
- Identifies resources shared with external entities
- Uses provable security (mathematical analysis)
- Generates findings for review
- Validates policies against security best practices

### Access Analyzer Features
- **External Access Findings** - Resources accessible outside account
- **Policy Validation** - Check policy syntax and logic
- **Policy Generation** - Generate policies from CloudTrail logs
- **Unused Access** - Identify unused permissions

### Enable Access Analyzer
```bash
# Create analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name MyAnalyzer \
  --type ACCOUNT

# List findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/MyAnalyzer
```

---

## 11. IAM Credential Reports

### Credential Report Contents
- User creation date
- Password last used
- Password last changed
- Password next rotation
- MFA device status
- Access key status and usage

### Generate Report
```bash
# Generate credential report
aws iam generate-credential-report

# Get credential report
aws iam get-credential-report --output text --query Content | base64 -d > credential-report.csv
```

### Access Advisor
- Shows service permissions granted to user/role
- Shows last accessed information
- Helps identify unused permissions
- Available via console or API

---

## 12. IAM Identity Center (SSO)

### What is Identity Center?
- Centrally manage access to AWS accounts and applications
- Single sign-on (SSO) experience
- Integration with external identity providers
- Successor to AWS SSO

### Key Features
- **Multi-Account Access** - Manage access across AWS accounts
- **Application Integration** - SSO to cloud applications
- **Permission Sets** - Reusable permission templates
- **Identity Source** - Internal or external identity providers

### Permission Sets
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 13. Service-Linked Roles

### What are Service-Linked Roles?
- Predefined roles for AWS services
- Created and managed by AWS services
- Cannot be modified by customers
- Automatically created when needed

### Common Service-Linked Roles
- **AWSServiceRoleForAutoScaling** - Auto Scaling service
- **AWSServiceRoleForECS** - ECS service
- **AWSServiceRoleForLambda** - Lambda service
- **AWSServiceRoleForRDS** - RDS service

### Managing Service-Linked Roles
```bash
# List service-linked roles
aws iam list-roles --path-prefix /aws-service-role/

# Delete service-linked role (if supported)
aws iam delete-service-linked-role --role-name AWSServiceRoleForAutoScaling
```

---

## 14. IAM Security Tools

### AWS CloudTrail
- Logs all IAM API calls
- Track user activity and API usage
- Detect unusual access patterns
- Compliance and auditing

### AWS Config
- Track IAM resource configurations
- Compliance rules for IAM resources
- Configuration change notifications
- Remediation actions

### Amazon GuardDuty
- Threat detection for IAM
- Unusual API call patterns
- Compromised credentials detection
- Machine learning-based analysis

---

## 15. Common Exam Scenarios

### Scenario 1: User cannot access S3 bucket
**Troubleshooting Steps:**
1. Check user permissions (direct or group)
2. Verify bucket policy doesn't deny access
3. Check for explicit deny in any policy
4. Verify MFA requirements
5. Check IP address restrictions

### Scenario 2: Cross-account access not working
**Common Issues:**
- Trust policy missing or incorrect
- External ID mismatch
- Insufficient permissions in target account
- Source account lacks AssumeRole permission

### Scenario 3: Application getting access denied
**Solutions:**
- Use IAM roles instead of users
- Attach role to EC2 instance profile
- Use AWS SDK credential chain
- Check CloudTrail for specific denied actions

### Scenario 4: Need temporary elevated access
**Solution:**
- Create break-glass role with required permissions
- Require MFA for role assumption
- Set maximum session duration
- Monitor usage with CloudTrail

### Scenario 5: Implement least privilege
**Approach:**
1. Start with minimal permissions
2. Use Access Advisor to identify unused permissions
3. Generate policies from CloudTrail logs
4. Regular access reviews and cleanup
5. Use permission boundaries for developers

---

## 16. CLI Commands Reference

### User Management
```bash
# Create user
aws iam create-user --user-name username

# Delete user
aws iam delete-user --user-name username

# List users
aws iam list-users

# Get user details
aws iam get-user --user-name username
```

### Role Management
```bash
# Create role
aws iam create-role --role-name MyRole --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy --role-name MyRole --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# List roles
aws iam list-roles

# Assume role
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/MyRole --role-session-name MySession
```

### Policy Management
```bash
# Create policy
aws iam create-policy --policy-name MyPolicy --policy-document file://policy.json

# List policies
aws iam list-policies --scope Local

# Get policy version
aws iam get-policy-version --policy-arn arn:aws:iam::123456789012:policy/MyPolicy --version-id v1
```

---

## 17. Exam Tips

### Key Points to Remember
- IAM is global (not region-specific)
- Root user has full access - secure it with MFA
- Use roles for applications, not users
- Explicit deny always wins
- Default is deny (no explicit allow = deny)
- Maximum 5,000 users per account
- Users can be in maximum 10 groups

### Common Mistakes
- Embedding credentials in code
- Using root user for daily tasks
- Not enabling MFA
- Overly permissive policies
- Not using roles for cross-account access
- Ignoring unused permissions

### Best Practices for Exam
- Understand policy evaluation logic
- Know when to use users vs roles
- Understand federation scenarios
- Know cross-account access patterns
- Understand service-linked roles
- Know IAM limits and quotas