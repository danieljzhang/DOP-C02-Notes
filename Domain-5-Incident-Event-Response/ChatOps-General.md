# ChatOps (Amazon Q Developer in Chat Applications) - DOP-C02 Exam Notes

**What it is:** Operate AWS from Slack or Microsoft Teams. **Important rename: AWS Chatbot became "Amazon Q Developer in chat applications" on Feb 19, 2025** — same service, same functionality, new name. The repo's single mention still uses the old name.

**Exam-testable facts:**
- **Notifications in chat:** CloudWatch alarms, CodePipeline manual approvals, GuardDuty findings (via EventBridge → SNS → chat client) land in the channel where responders already are.
- **Run commands from chat:** `@Amazon Q` followed by AWS CLI commands (invoke Lambda, pull diagnostics), natural-language questions about resources, and interactive message buttons (approve/deny).
- **Incident response:** integrates with Incident Manager — responders collaborate in the channel and their actions are visible in real time.
- **Support cases:** create and manage AWS Support cases from chat.
- **Permissions:** IAM controls who can do what from chat (read-only vs command execution via guardrail policies on the SNS-to-chat mapping).
- **Regional footprint:** the console works only in us-east-2 (Ohio); APIs in us-east-2, us-west-2, ap-southeast-1, eu-west-1.

**Common traps:**
- The exam may still say "AWS Chatbot" — know both names, one service.
- ChatOps approval: a CodePipeline manual approval action whose notification goes to Slack, approved via interactive message — the classic ChatOps exam scenario.

**Sources:** https://docs.aws.amazon.com/chatbot/latest/adminguide/doc-history.html · https://docs.aws.amazon.com/chatbot/latest/adminguide/performing-actions.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
