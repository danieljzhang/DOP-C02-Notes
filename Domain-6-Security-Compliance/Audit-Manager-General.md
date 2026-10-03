# AWS Audit Manager - DOP-C02 Exam Notes

## 1. Overview

**What it is:** A service that continuously collects evidence from your AWS environment to help you audit compliance with frameworks like PCI-DSS, HIPAA, SOC 2, CIS, and GDPR — replacing manual evidence collection with automated, always-on gathering.

**What problem it solves:** Compliance audits traditionally require manually pulling screenshots, logs, and config snapshots before an audit. Audit Manager automates that collection continuously, so when an auditor asks for evidence you already have it.

---

## 2. Core Concepts

### Frameworks
A **framework** is the compliance standard you're auditing against. Audit Manager provides:
- **Pre-built frameworks** — PCI-DSS, HIPAA, SOC 2, CIS AWS Foundations, GDPR, FedRAMP, NIST
- **Custom frameworks** — build your own from controls

### Controls
A **control** is a specific requirement within a framework (e.g., "MFA must be enabled for all IAM users"). Each control has:
- **Data sources** — where Audit Manager collects evidence from (AWS Config rules, CloudTrail, Security Hub, AWS API calls)
- **Evidence** — the collected proof that the control is met or not

### Assessments
An **assessment** is a running instance of a framework applied to specific AWS accounts and services. It continuously collects evidence for all controls in the framework.

```
Framework (PCI-DSS)
    └── Assessment (applied to prod account, us-east-1)
            └── Controls (100+ PCI controls)
                    └── Evidence (collected daily from Config, CloudTrail, etc.)
```

### Evidence
Automatically collected from:
- **AWS Config** — resource configuration snapshots
- **CloudTrail** — API call records
- **Security Hub** — security findings
- **Direct AWS API calls** — Audit Manager calls describe/list APIs directly

---

## 3. Assessment Reports

When you're ready for an audit, generate an **assessment report** — a ZIP file containing all collected evidence, organised by control. Share directly with auditors.

```bash
# Create assessment
aws auditmanager create-assessment \
  --name "PCI-DSS-2024" \
  --assessment-reports-destination '{"destinationType":"S3","destination":"s3://my-audit-bucket"}' \
  --scope '{"awsAccounts":[{"id":"123456789012"}],"awsServices":[{"serviceName":"S3"},{"serviceName":"IAM"}]}' \
  --roles '[{"roleType":"PROCESS_OWNER","roleArn":"arn:aws:iam::123456789012:role/AuditManagerRole"}]' \
  --framework-id <pci-dss-framework-id>

# Generate assessment report
aws auditmanager create-assessment-report \
  --name "PCI-DSS-Q4-2024" \
  --assessment-id <assessment-id>
```

---

## 4. Audit Manager vs Config vs Security Hub

| | Audit Manager | AWS Config | Security Hub |
|---|---|---|---|
| **Primary purpose** | Compliance evidence collection for auditors | Configuration compliance monitoring | Security findings aggregation |
| **Output** | Audit-ready evidence packages | Compliance dashboards + remediation | Security findings + scores |
| **Audience** | Auditors, compliance teams | DevOps, security teams | Security teams |
| **Frameworks** | PCI, HIPAA, SOC 2, etc. | Custom rules | CIS, PCI, AWS Foundational |
| **Remediation** | No (evidence only) | Yes (SSM Automation) | Yes (via integrations) |

**Exam angle:** "Automate evidence collection for an upcoming PCI-DSS audit" → Audit Manager. "Detect and remediate non-compliant resources" → Config. "Centralise security findings" → Security Hub.

---

## 5. Delegated Administrator

In a multi-account Organisation, designate one account as the **delegated administrator** for Audit Manager. It can create assessments that span multiple member accounts — one assessment, org-wide evidence.

```bash
aws auditmanager register-organization-admin-account \
  --admin-account-id 123456789012
```

---

## 6. Common Exam Scenarios

### Scenario 1: Prepare for annual PCI-DSS audit with minimal manual effort
**Solution:** Enable Audit Manager, select the PCI-DSS pre-built framework, create an assessment covering the production account. Evidence collects continuously. Generate the assessment report when the auditor arrives — no last-minute scramble.

### Scenario 2: Custom internal compliance framework
**Solution:** Create a custom framework in Audit Manager with controls mapped to internal policy requirements. Attach Config rules and CloudTrail as data sources. Assessments collect evidence automatically.

### Scenario 3: Multi-account compliance evidence
**Solution:** Register a delegated administrator account. Create one assessment scoped to all member accounts via Organizations. Evidence from all accounts flows into one report.

---

## 7. Exam Tips

- Audit Manager = **evidence collection for auditors**. It does not remediate — that's Config + SSM.
- Pre-built frameworks save setup time; custom frameworks handle internal policies.
- Evidence sources: Config, CloudTrail, Security Hub, direct API calls — know all four.
- **Assessment report** is the deliverable — a ZIP of evidence for the auditor.
- Delegated administrator enables org-wide assessments from one account.
- "Automate compliance evidence", "audit-ready", "evidence collection" → Audit Manager.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
