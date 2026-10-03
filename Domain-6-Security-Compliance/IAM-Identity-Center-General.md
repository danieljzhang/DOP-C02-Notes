# AWS IAM Identity Center - DOP-C02 Exam Notes

> **Naming:** AWS IAM Identity Center is the current name. It was called **AWS Single Sign-On (AWS SSO)** until July 2022. The exam may use either name — they are the same service. Any file in this repo that still says "AWS SSO" means IAM Identity Center.

## 1. Overview

**What it is:** The centralized workforce identity service for AWS. One place to manage *who* can access *which AWS accounts and applications*, using *which permissions*, with a single login experience.

**What problem it solves:** Without it, each AWS account has its own IAM users, each with separate passwords and access keys. IAM Identity Center replaces that with one identity source, one login portal, and permission sets that deploy consistently across hundreds of accounts.

---

## 2. Core Concepts

### Identity Sources
Where users and groups come from. You pick exactly one at a time:

| Source | When to use |
|---|---|
| **IAM Identity Center directory** (built-in) | No existing IdP; small teams; quick setup |
| **Active Directory** (AWS Managed AD or AD Connector) | Enterprise with on-prem AD |
| **External IdP via SAML 2.0** | Okta, Azure AD, Ping, Google Workspace, etc. |

Changing the identity source is possible but resets group memberships — plan carefully.

### Permission Sets
A **permission set** is a reusable template that defines what a user can do inside one AWS account. It is *not* an IAM role — IAM Identity Center creates and manages the underlying IAM roles in each account automatically when you assign a permission set.

- Contains AWS managed policies, customer managed policies, inline policies, and/or permission boundaries
- Deployed to target accounts on assignment; updated across all accounts when you update the permission set
- Session duration configurable (1–12 hours)

### Assignments
The three-way link: **user or group** → **permission set** → **AWS account (or OU)**

- One assignment = one role created in the target account
- Multiple permission sets on the same account = multiple roles; user picks at login
- Assignments to an OU propagate to all accounts in that OU (requires AWS Organizations)

### AWS Access Portal
The web URL (`https://<subdomain>.awsapps.com/start`) where users log in, see their assigned accounts and applications, and get temporary credentials or console access.

---

## 3. How It Works End-to-End

```
User logs in at Access Portal
        ↓
IAM Identity Center authenticates against identity source
        ↓
User selects account + permission set
        ↓
IAM Identity Center calls sts:AssumeRoleWithSAML on the
auto-created IAM role in the target account
        ↓
User gets temporary credentials (console session or CLI token)
```

For CLI access: `aws sso login --profile my-profile` stores a short-lived token; the SDK/CLI refreshes it automatically within the session window.

---

## 4. Multi-Account Access Pattern (Exam Favourite)

```
AWS Organizations
├── Management account
└── Member accounts (dev, staging, prod)
        ↑
IAM Identity Center (enabled in management account)
        ↑
Identity source (Okta / AD / built-in)

Permission sets:
  - ReadOnly      → assigned to all accounts for all developers
  - PowerUser     → assigned to dev account for developers
  - AdminAccess   → assigned to prod account for ops team only
```

Key exam point: **one permission set, many accounts** — update the permission set once, IAM Identity Center propagates the change to every assigned account automatically.

---

## 5. Attribute-Based Access Control (ABAC)

IAM Identity Center can pass user attributes (department, cost-center, team) as session tags into the assumed role. IAM policies in the target account can then use `aws:PrincipalTag` conditions to make access decisions without maintaining separate permission sets per team.

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::company-data/${aws:PrincipalTag/Department}/*"
}
```

Exam angle: "grant access to resources tagged with the user's department without creating one permission set per department" → ABAC with IAM Identity Center session tags.

---

## 6. Integration with AWS CLI / SDK

```bash
# Configure a named profile backed by IAM Identity Center
aws configure sso
# prompts: SSO start URL, SSO region, account, role, output format

# Login (opens browser for IdP authentication)
aws sso login --profile prod-readonly

# Use the profile
aws s3 ls --profile prod-readonly

# Logout (revokes the SSO token)
aws sso logout --profile prod-readonly
```

The credentials are stored in `~/.aws/sso/cache/` and are short-lived (match the permission set session duration).

---

## 7. IAM Identity Center vs IAM Roles — When to Use Which

| Scenario | Use |
|---|---|
| Human workforce accessing AWS accounts | IAM Identity Center |
| Machine-to-machine / service-to-service | IAM roles |
| CI/CD pipeline assuming a role | IAM role (or OIDC federation) |
| Contractor needing temporary multi-account access | IAM Identity Center |
| Application running on EC2 needing S3 access | EC2 instance profile (IAM role) |

---

## 8. Common Exam Scenarios

### Scenario 1: Centralise access for 50 AWS accounts
**Solution:** Enable IAM Identity Center in the management account, connect to existing AD via AD Connector, create permission sets matching job functions, assign to OUs. Users log in once and switch accounts without re-authenticating.

### Scenario 2: Developer needs read-only prod access, full dev access
**Solution:** Two permission sets (ReadOnly, PowerUser). Assign ReadOnly → prod account, PowerUser → dev account, both to the developer's group. One login, two roles available at the portal.

### Scenario 3: Enforce MFA for all AWS console access
**Solution:** Configure MFA in IAM Identity Center (not per-account IAM). Set MFA enforcement to "Required for all users". This applies consistently across all accounts — no per-account IAM MFA policy needed.

### Scenario 4: Audit who accessed which account and when
**Solution:** IAM Identity Center writes sign-in and permission-set assumption events to CloudTrail. Filter on `sso.amazonaws.com` event source. For the full picture, correlate with CloudTrail in each member account.

---

## 9. CLI Commands Reference

```bash
# List permission sets in an instance
aws sso-admin list-permission-sets \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxx

# Describe a permission set
aws sso-admin describe-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxx/ps-xxx

# List accounts a permission set is assigned to
aws sso-admin list-accounts-for-provisioned-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxx/ps-xxx

# List assignments for an account
aws sso-admin list-account-assignments \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxx \
  --account-id 123456789012 \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxx/ps-xxx

# Provision (push) a permission set to all assigned accounts
aws sso-admin provision-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxx/ps-xxx \
  --target-type ALL_PROVISIONED_ACCOUNTS
```

---

## 10. Exam Tips

- **"AWS SSO" = IAM Identity Center** — same service, old name. Don't let the name change throw you.
- Permission sets are **templates**; IAM Identity Center creates the actual IAM roles in each account. You manage the template, not the roles directly.
- **One identity source at a time** — you can't mix AD and Okta simultaneously.
- MFA enforcement lives in IAM Identity Center, not in per-account IAM — this is the scalable answer for multi-account MFA.
- ABAC with session tags is the answer to "grant access based on user attributes without a permission set per team."
- CloudTrail source for IAM Identity Center events: `sso.amazonaws.com` and `sso-directory.amazonaws.com`.
- IAM Identity Center requires **AWS Organizations** for multi-account assignment. Single-account use is possible but uncommon in exam scenarios.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
