# CloudWatch Deep Dive — CrossAccountObservability

**What it is:** CloudWatch cross-account observability lets a central **monitoring account** see metrics (and build alarms, dashboards, anomaly detectors) from **source accounts** — typically wired through AWS Organizations — with no account switching.

**Exam-testable facts:**
- **Two roles:** *source accounts* share their observability data; the *monitoring account* views it. Setup is easiest with Organizations (all accounts in the org can be linked at once).
- **What flows:** CloudWatch metrics, Logs Insights queries across accounts, and cross-account alarms. You create the alarm *once* in the monitoring account against source-account data.
- **No data duplication:** the monitoring account queries source data in place; metrics aren't copied over.
- **Permissions:** source accounts grant access via the `CloudWatch-CrossAccountSharing` IAM role pattern (or Organizations trust); the monitoring account's users need permission to use the monitoring console views.
- **Exam signal:** "centralized monitoring dashboard for 50 accounts without switching" → cross-account observability. "Aggregate *logs* centrally for compliance" → that leans toward a central logging account with S3/Kinesis, a different pattern.

**Common traps:**
- Cross-account observability is for *viewing and alarming* — it does not centralize log *storage*. Don't confuse it with a centralized S3 logging bucket.
- Alarms are created in the monitoring account but evaluate source-account metrics — know where each piece lives.
- Linking via Organizations vs manual account linking: Organizations scales; manual is per-account.

**Sources:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Cross-Account-Setup.html

*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation link above.*
