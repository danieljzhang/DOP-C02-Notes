# AWS AppConfig - DOP-C02 Exam Notes

**What it is:** Managed application-configuration service: deploy config changes (feature flags, throttling limits, allow-lists) with the same safety as code deploys — validation, deployment strategies, automatic rollback.

**Exam-testable facts:** Configuration stored as hosted documents, validated against a schema before deployment; deployment strategies include all-at-once, linear, and canary with bake time; CloudWatch alarms trigger automatic rollback; integrates with CodePipeline/CodeDeploy as a deployment action.

**Why it matters:** "Safely roll out a config change with canary + auto-rollback" → AppConfig, not a Parameter Store edit.

**Sources:** https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
