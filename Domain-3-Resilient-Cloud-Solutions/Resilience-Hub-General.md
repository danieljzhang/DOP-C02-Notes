# AWS Resilience Hub - DOP-C02 Exam Notes

## 1. Overview

**What it is:** A service that assesses, validates, and tracks the resilience of your AWS applications against defined RTO and RPO targets — giving you a **resilience score** and actionable recommendations.

**What problem it solves:** You might think your application is resilient because it uses Multi-AZ RDS and Auto Scaling, but you don't know if those configurations actually meet your RTO/RPO targets until you test them. Resilience Hub analyses your infrastructure and tells you before an outage.

---

## 2. Core Concepts

### Resiliency Policy
Define your targets:
- **RTO** (Recovery Time Objective) — how long can the app be down?
- **RPO** (Recovery Point Objective) — how much data loss is acceptable?
- Set separately for **Software**, **Hardware**, and **AZ** disruption tiers

### Application
Import your application definition from:
- CloudFormation stacks
- Terraform state files
- AWS Resource Groups
- AppRegistry

### Resilience Score
After assessment, Resilience Hub gives a score (0–100). Below your policy targets = failing score. The score covers:
- **Availability** — can the app survive component failures?
- **Recovery** — can it recover within RTO/RPO?

### Recommendations
Specific, actionable fixes:
- "Add a read replica to this RDS instance"
- "Enable Multi-AZ for this ElastiCache cluster"
- "Add an alarm for this Lambda function's error rate"
- "Configure a DLQ for this SQS queue"

---

## 3. Assessment Workflow

```
1. Define resiliency policy (RTO/RPO targets)
        ↓
2. Import application (CloudFormation / Terraform / Resource Groups)
        ↓
3. Run assessment
        ↓
4. Review resilience score + recommendations
        ↓
5. Implement recommendations
        ↓
6. Re-assess → score improves
        ↓
7. Schedule recurring assessments (drift detection)
```

---

## 4. Resilience Hub vs Other Services

| | Resilience Hub | AWS Backup | Multi-AZ / Auto Scaling |
|---|---|---|---|
| **Purpose** | Assess and score resilience | Backup and restore | Provide resilience |
| **Output** | Score + recommendations | Recovery points | HA/DR capability |
| **Action** | Tells you what to fix | Restores data | Handles failures |
| **Exam angle** | "Measure resilience against RTO/RPO" | "Protect data" | "Survive failures" |

---

## 5. Integration with CI/CD

Resilience Hub can be integrated into pipelines — run an assessment as a pipeline stage and fail the deployment if the resilience score drops below a threshold:

```bash
# Run assessment
aws resiliencehub start-app-assessment \
  --app-arn arn:aws:resiliencehub:us-east-1:123456789012:app/my-app \
  --assessment-name pipeline-check

# Get score (poll until complete)
aws resiliencehub describe-app-assessment \
  --assessment-arn <assessment-arn> \
  --query 'assessment.resiliencyScore'
```

---

## 6. Common Exam Scenarios

### Scenario 1: Validate application meets RTO/RPO before go-live
**Solution:** Import the application into Resilience Hub, define a resiliency policy matching the SLA, run an assessment. Fix any failing recommendations before launch.

### Scenario 2: Detect resilience drift after infrastructure changes
**Solution:** Schedule recurring Resilience Hub assessments. If a change (e.g., someone disabled Multi-AZ) drops the score below the policy, trigger an alert via EventBridge.

### Scenario 3: Prove resilience posture to stakeholders
**Solution:** Resilience Hub assessment report shows the score and which components meet/fail the RTO/RPO targets — a concrete, auditable artefact.

---

## 7. Exam Tips

- Resilience Hub **measures** resilience; it doesn't provide it — Multi-AZ, Auto Scaling, and Backup do that.
- The **resilience score** is the key output — know it's 0–100 and tied to your RTO/RPO policy.
- Recommendations are **specific** (not generic) — "add a replica to *this* resource."
- Supports CloudFormation, Terraform, and Resource Groups as input — not just CloudFormation.
- "Assess whether the application meets its RTO/RPO" → Resilience Hub.
- Can be integrated into CI/CD pipelines as a quality gate.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
