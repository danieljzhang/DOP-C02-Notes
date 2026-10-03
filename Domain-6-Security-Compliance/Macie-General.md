# Amazon Macie - DOP-C02 Exam Notes

**What it is:** Data security for S3 — machine learning plus pattern matching that finds sensitive data *and* bucket misconfigurations, then routes findings into your response workflow.

**Exam-testable facts:**
- **Two finding types:** *policy findings* (bucket-level: publicly accessible, unencrypted, etc.) and *sensitive data findings* (object-level: credentials, financial data, personal/PII/PHI, custom identifiers, or multiple categories at once).
- **Discovery modes:** *automated sensitive data discovery* (continuous, sampled) vs *sensitive data discovery jobs* (targeted, one-time or scheduled).
- **Custom data identifiers** (regex) catch proprietary formats; **allow lists** suppress known-safe matches to cut noise.
- **Finding flow:** findings go to EventBridge and Security Hub for automated response; full discovery details land in a customer-owned S3 bucket; findings retained 30 days.
- **Sensitive data findings are always treated as new** — re-detecting the same object creates a fresh finding (unlike policy findings, which update in place).

**Common traps:**
- Macie is **S3-only**. "Find PII in RDS/EBS" is not Macie.
- Policy finding (the bucket is misconfigured) vs sensitive-data finding (the object contains secrets) — the exam will make you pick.

**Sources:** https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html · https://docs.aws.amazon.com/macie/latest/user/findings-types.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
