# EventBridge Scheduler - DOP-C02 Exam Notes

**What it is:** The purpose-built serverless scheduler — cron, rate, or one-time invocation of 200+ AWS API targets. For "run X on a schedule," this has replaced CloudWatch Events scheduled rules as the recommended answer.

**Exam-testable facts:**
- **Occurrence types:** *one-time* (`at(...)` timestamp) or *recurring* (cron or rate expression).
- **Flexible time windows:** OFF, or FLEXIBLE with a 1–1440 minute window — the invocation lands *somewhere inside* the window, which smooths out thundering-herd load and cost.
- **Schedule groups:** tag and organize schedules as a set.
- **Timeframes:** timezone-aware evaluation plus optional start/end dates (start date is ignored for one-time schedules).
- **Reliability:** per-target retry policy and an SQS dead-letter queue for failed invocations.
- **Permissions:** the scheduler assumes an execution *role* to invoke the target — the role, not the schedule creator, needs target permissions.

**Common traps:**
- "Run this Lambda every hour, but spread invocations over 15 minutes" → flexible time window, not a cron hack.
- Flexible window controls *when within the window* it fires; it is not a delay or a timeout.
- CloudWatch Events *scheduled rules* still exist, but new designs should use Scheduler.

**Sources:** https://docs.aws.amazon.com/eventbridge/latest/userguide/using-eventbridge-scheduler.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
