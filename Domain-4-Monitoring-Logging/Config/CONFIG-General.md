# Config — General Notes (DOP-C02)

**What it is:** The configuration recorder and compliance evaluator — the *detective* control answering "is my infrastructure still compliant, and what changed?"

**Exam-testable facts:**
- **Config rules:** managed (AWS-provided, e.g., "S3 buckets must be encrypted") or custom (Lambda-backed or Guard `CUSTOM_POLICY`). Triggered on configuration *changes*, on a *schedule*, or both.
- **Conformance packs:** a *bundle* of rules plus remediation actions, deployed as one YAML unit (backed by CloudFormation). **Organizational conformance packs** roll out from the management/delegated-admin account across the whole Organization.
- **Remediation:** automatic via SSM Automation documents, or manual. Can ship inside the conformance pack — the self-healing compliance loop (Config detects → SSM remediates → Config re-evaluates).
- **Proactive evaluation:** evaluate resource configuration *before* deployment — the preventative flavor, pairing with Guard in pipelines.
- **Aggregators:** one dashboard for compliance across accounts and Regions.
- **Prerequisites:** a recorder (what to record) + delivery channel (S3 bucket) + IAM role; without the recorder running, nothing is evaluated.

**Common traps:**
- Config rules are *detective* (post-deploy) by default; Guard-in-pipeline is *preventative* (pre-deploy). The exam pairs them deliberately.
- A conformance pack is more than rules — it carries remediation with it.
- Config tells you *what changed*; CloudTrail tells you *who changed it* (API audit). Don't swap them.

**Sources:** https://docs.aws.amazon.com/lambda/latest/dg/governance-config.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
