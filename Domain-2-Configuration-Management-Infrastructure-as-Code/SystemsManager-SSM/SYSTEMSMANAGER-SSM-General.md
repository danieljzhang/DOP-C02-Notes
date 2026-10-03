# SystemsManager-SSM — General Notes (DOP-C02)

**What it is:** The fleet-operations toolkit. The exam's favorite question shape is "which SSM capability solves this scenario" — learn the one-line job of each.

**Exam-testable facts:**
- **Run Command:** execute scripts/commands across many managed instances at once, no SSH. Rate controls and error thresholds for rolling execution.
- **Session Manager:** interactive shell in browser/CLI with *no inbound ports, no bastion, no SSH keys*. Sessions can be logged to S3/CloudWatch for audit.
- **Patch Manager:** automated OS patching via *patch baselines* (what's approved) and *patch groups* (which instances); *scan* (report only) vs *install*; scheduled through Maintenance Windows.
- **Maintenance Windows:** cron-like schedules defining *when* admin tasks run, with duration, cutoff ("stop initiating N hours before close"), and registered task types (Run Command, Automation, Lambda, Step Functions).
- **State Manager:** *associations* that enforce desired state (e.g., "this config file must exist") and remediate drift on a schedule.
- **Automation:** multi-step *runbooks* (AWS-managed or custom documents) — the engine behind golden-AMI builds and incident runbooks.
- **Inventory:** collects OS, application, and config metadata from managed instances for asset/compliance queries.
- **Fleet Manager / Explorer / OpsCenter:** console views; **OpsCenter** centralizes operational issues as **OpsItems** with deduplication and ownership.
- **Hybrid:** on-prem servers become *managed instances* via activation codes — then all of the above works on them.

**Common traps:**
- Need a shell *right now* with audit trail and no SSH? → Session Manager, not Run Command.
- Recurring patching at 2 AM Sundays? → Patch Manager *inside* a Maintenance Window.
- "Keep this setting applied forever" → State Manager association, not a one-off Run Command.

**Sources:** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html · https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html · https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html · https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-state.html · https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-inventory.html · https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
