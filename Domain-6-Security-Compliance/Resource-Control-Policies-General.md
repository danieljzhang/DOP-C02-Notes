# Resource Control Policies (RCPs) - DOP-C02 Exam Notes

> **New in 2024.** RCPs are a type of AWS Organizations policy that controls the *maximum permissions that can be granted to resources* in member accounts. They are the resource-side complement to SCPs (which control the principal side).

## 1. Overview

**What it is:** An Organizations policy type that sets a permission ceiling on AWS *resources* (not principals). Where an SCP limits what a principal *can do*, an RCP limits what *can be done to a resource* — regardless of who is making the request.

**What problem it solves:** SCPs protect against over-permissioned IAM principals inside your org, but they don't stop an external principal (another AWS account, a public S3 bucket policy, a cross-account role) from accessing your resources. RCPs close that gap by restricting the resource itself.

---

## 2. RCPs vs SCPs — The Critical Distinction

This is the most exam-tested concept. Get this table right:

| | SCP (Service Control Policy) | RCP (Resource Control Policy) |
|---|---|---|
| **Controls** | What principals *in the account* can do | What can be done *to resources* in the account |
| **Attached to** | Root, OU, or account in Organizations | Root, OU, or account in Organizations |
| **Affects** | IAM users, roles (not the management account) | AWS resources (S3, SQS, KMS, etc.) |
| **Blocks** | Principals from taking actions | Requests to resources (even from outside the org) |
| **Management account** | Does NOT apply to management account | Does NOT apply to management account |
| **Introduced** | 2017 | 2024 |

**Together:** SCP + RCP = defence in depth. SCP says "our principals can't exfiltrate data." RCP says "our resources can't be accessed by untrusted principals even if a bucket policy accidentally allows it."

---

## 3. How RCPs Work

RCPs use the same JSON policy language as IAM policies and SCPs. An RCP attached to an OU applies to all resources in all accounts under that OU.

**Evaluation logic:** A request to a resource succeeds only if ALL of the following allow it:
1. The resource-based policy (e.g., S3 bucket policy)
2. The identity-based policy on the caller
3. Any applicable SCP (on the caller's account)
4. **Any applicable RCP (on the resource's account)** ← new layer

An explicit Deny in any layer blocks the request.

---

## 4. Example: Prevent S3 Access from Outside the Organisation

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyS3AccessOutsideOrg",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:PrincipalOrgID": "o-exampleorgid"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```

This denies any S3 request where the caller is not a principal from your organisation (and is not an AWS service acting on your behalf). Even if a developer accidentally makes a bucket public, this RCP blocks external access at the organisation level.

---

## 5. Example: Enforce Encryption in Transit for SQS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonTLSSQS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "sqs:*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

Applied at the root of the organisation, this enforces TLS for all SQS queues across every account without touching individual queue policies.

---

## 6. Supported Resource Types (at launch, 2024)

RCPs initially support a subset of services. Confirmed at launch:
- Amazon S3
- AWS KMS
- Amazon SQS
- AWS Secrets Manager
- Amazon STS (for cross-account role assumptions)

> Verify the current list in the AWS documentation before the exam — AWS adds services over time.

---

## 7. RCPs and the Management Account

Like SCPs, RCPs **do not apply to the management account**. This is a deliberate design choice — the management account is the trust anchor of the organisation. For exam questions: if the scenario involves the management account accessing resources, RCPs won't block it.

---

## 8. RCPs vs Resource-Based Policies

RCPs are **not** a replacement for resource-based policies (S3 bucket policies, KMS key policies, etc.). They work together:

- Resource-based policy: grants access to specific principals
- RCP: sets the maximum boundary — even a permissive resource-based policy can't exceed what the RCP allows

Think of it as: resource-based policy = what you *want* to allow; RCP = what the organisation *permits* to be allowed.

---

## 9. Common Exam Scenarios

### Scenario 1: Prevent data exfiltration to external accounts
**Requirement:** Ensure no S3 bucket in any account can be accessed by principals outside the organisation, even if a bucket policy is misconfigured.
**Solution:** Attach an RCP at the root OU that denies S3 actions where `aws:PrincipalOrgID` does not match your org ID.

### Scenario 2: Enforce KMS encryption org-wide
**Requirement:** All KMS key usage must come from within the organisation.
**Solution:** RCP on the root OU denying `kms:*` where `aws:PrincipalOrgID` is not your org. Complements SCPs that prevent principals from disabling KMS encryption.

### Scenario 3: SCP vs RCP — which to use?
- "Prevent our developers from deleting S3 buckets" → **SCP** (restricts the principal)
- "Prevent anyone outside our org from accessing our S3 buckets" → **RCP** (restricts the resource)
- "Both" → use both, they're complementary

---

## 10. Exam Tips

- RCPs are **new (2024)** — expect 1–2 questions testing whether you know they exist and how they differ from SCPs.
- The key mental model: **SCP = fence around the principal; RCP = fence around the resource**.
- Both SCPs and RCPs use Deny-based logic for restrictions (allow-list or deny-list patterns).
- Neither applies to the management account.
- RCPs don't replace resource-based policies — they constrain them.
- `aws:PrincipalOrgID` is the condition key used in both SCPs and RCPs to scope to your organisation.
- If an exam question asks how to prevent cross-account access to resources *even if the resource policy allows it*, the answer is RCP.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
