# SystemsManager-SSM Deep Dive — RunCommand

**What it is:** Run Command lets you execute scripts or commands across many managed instances at once — no SSH, no bastion hosts. You pick targets (by tags, instance IDs, or resource groups), pick a document (e.g. `AWS-RunShellScript`), and it runs everywhere in parallel.

**Exam-testable facts:**
- **No inbound ports needed.** The SSM Agent on each instance polls Systems Manager outbound (HTTPS). This is the exam's favorite contrast with SSH: "run commands on 500 instances without opening port 22" → Run Command.
- **Rate controls:** set how many targets run concurrently (`MaxConcurrency`: count or percentage) and how many errors you tolerate before stopping (`MaxErrors`). Rolling execution across a fleet uses these two knobs.
- **Targets:** instance IDs, tags (`tag:Environment=prod`), or resource groups. Tag-based targeting is how you hit "all web servers" without listing them.
- **Command documents:** `AWS-RunShellScript` (Linux), `AWS-RunPowerShellScript` (Windows), plus AWS-managed documents for common tasks. Custom documents package your own scripts.
- **Output:** command output is captured per instance and can be written to S3 or CloudWatch Logs; each invocation gets a Command ID you can query with `list-command-invocations`.
- **Notifications:** SNS notifications on command state changes (InProgress, Success, Failed, TimedOut).
- **Prerequisite:** instances must be *managed instances* — SSM Agent installed + an IAM instance profile with the `AmazonSSMManagedInstanceCore` policy.

**Common traps:**
- "Interactive shell right now" → Session Manager, not Run Command. Run Command is fire-and-forget batch execution.
- Rate control vs error threshold: `MaxConcurrency` limits parallelism; `MaxErrors` stops the rollout. The exam swaps them.
- Run Command needs the SSM Agent *and* the instance profile — missing either is the classic "command stays Pending" troubleshooting answer.

**Sources:** https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html

*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation link above.*
