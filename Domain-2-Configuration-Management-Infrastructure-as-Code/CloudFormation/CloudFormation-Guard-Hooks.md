# CloudFormation Guard and Hooks - DOP-C02 Exam Notes

**What it is:** An open-source policy-as-code engine. You write rules; Guard validates any JSON/YAML structured data against them — CloudFormation templates, change sets, Terraform JSON, Kubernetes manifests. It is the shift-left compliance tool: catch violations *before* anything deploys.

**Exam-testable facts:**
- **Three governance uses:** *preventative* (validate IaC/templates pre-deploy — shift-left), *detective* (validate AWS Config configuration items continuously), *deployment safety* (validate change sets to catch replacement-causing edits, e.g., renaming a DynamoDB table).
- **Rule syntax** (Guard 2.x): `rule <name> [when <condition>] { <assertion> }`.
- **Guard vs Hooks:** Guard is the *rule language*; **CloudFormation Hooks** are the *enforcement point* — Hooks proactively run your Guard rules before create/update/delete operations (and Cloud Control API calls). Know which is which.
- **In CI/CD:** run `cfn-guard validate` as a CodeBuild step — a policy violation fails the build, blocking the deploy.
- **Config custom rules** can be authored in Guard (`CUSTOM_POLICY` type); **conformance packs** can bundle Guard-based rules for org-wide rollout.

**Common traps:**
- Config rules = detective (after deploy) unless *proactive evaluation* is enabled; Guard in the pipeline = preventative (before deploy). The exam loves this pairing.
- A Hook *enforces*; Guard *evaluates*. Don't swap them.

**Sources:** https://docs.aws.amazon.com/cfn-guard/latest/ug/what-is-guard.html · https://docs.aws.amazon.com/lambda/latest/dg/governance-config.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
