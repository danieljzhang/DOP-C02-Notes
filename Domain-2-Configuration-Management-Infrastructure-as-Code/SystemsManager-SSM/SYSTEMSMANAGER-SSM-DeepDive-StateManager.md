# SystemsManager-SSM Deep Dive — StateManager

**What it is:** State Manager enforces *desired state* on your fleet on a schedule. You define an **association** — a document + targets + schedule — and State Manager continuously makes reality match: e.g. "this config file must exist on all web servers," "the CloudWatch agent must be installed everywhere."

**Exam-testable facts:**
- **Association** = the unit of work: document (what to enforce), targets (where), schedule (how often to check), parameters.
- **Drift correction:** when an association run finds a target out of compliance, State Manager re-applies the document to bring it back — the self-healing loop for configuration.
- **One-time vs recurring:** associations can run once immediately or on a cron/rate schedule. Compliance reporting works either way.
- **S3 compliance reporting:** association execution results can be written to S3 for audit trails.
- **Common documents:** `AWS-RunShellScript`, `AWS-ConfigureAWSPackage` (install/update packages like the CloudWatch agent), `AWS-ApplyAnsiblePlaybooks`.
- **Rate controls** (same as Run Command): `MaxConcurrency` and `MaxErrors` govern how associations roll across large fleets.

**Common traps:**
- "Keep this setting applied *forever*" → State Manager association. "Run this command *once*" → Run Command. The exam deliberately pairs them.
- State Manager detects and *remediates* drift; AWS Config detects and *reports* drift (remediation needs a separate SSM Automation hookup). Don't swap the roles.
- An association without a schedule still runs once at creation — then never again unless scheduled.

**Sources:** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-state.html

*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation link above.*
