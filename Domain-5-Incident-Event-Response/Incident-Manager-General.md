# Incident Manager Concepts - DOP-C02 Exam Notes

**⚠️ Status caveat (verified):** AWS Systems Manager Incident Manager is **closed to new customers** — existing customers continue normally. Learn the *concepts and workflow* (the exam tests the response model, not new provisioning); do not present it as a service a new team would adopt today.

**What it is:** Incident-response orchestration: who gets paged, what runs automatically, where responders collaborate, and how you learn afterward.

**Exam-testable facts:**
- **Contacts:** individuals with engagement channels (email, SMS, voice). **On-call schedules** rotate contacts (up to 30 per rotation) so there's always an owner.
- **Escalation plans:** staged paging — notify stage 1, wait N minutes, escalate to stage 2. Nobody waits on one silent phone.
- **Response plans:** the master playbook launched at incident start — engagements + runbooks + chat channel + an *incident template* (title, impact rating, **dedupe string**, tags).
- **Runbooks:** SSM Automation documents mixing automated steps with manual ones.
- **Chat channels:** responders collaborate in Slack/Teams/Chime; every responder action is visible in the channel in real time.
- **Post-incident analysis:** timeline, metrics, and action items (tracked as OpsItems) to harden runbooks and response plans.
- **Trigger path:** CloudWatch alarm or EventBridge rule → start incident from a response plan. The **dedupe string** stops a flapping alarm from spawning duplicate incidents.
- **Multi-Region replication set** recommended so incident data survives a Regional outage.

**Common traps:**
- Escalation plan = *who and when*; response plan = *what to do*. Different artifacts.
- Dedupe string is the "don't page me five times for one outage" answer.

**Sources:** https://docs.aws.amazon.com/en_us/incident-manager/latest/APIReference/API_CreateResponsePlan.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
