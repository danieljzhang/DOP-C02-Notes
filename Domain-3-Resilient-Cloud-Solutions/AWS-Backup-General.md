# AWS Backup - DOP-C02 Exam Notes

**What it is:** Centralized, policy-based backup orchestration across AWS services — one place to define *what* gets backed up, *how often*, *how long* it's kept, and *where copies go*, replacing per-service snapshot scripts.

**Exam-testable facts:**
- **Backup plan:** one or more *rules*; each rule sets a schedule (cron/rate), a backup window, a *lifecycle* (transition to cold storage after N days, expire after M days), copy actions, and the target vault.
- **Backup vault:** the encrypted container holding recovery points (KMS-encrypted; each vault has its own key).
- **Vault Lock:** makes recovery points *immutable* (WORM). **Compliance mode cannot be undone — not even by the root user** — which is the ransomware-protection answer. Governance mode is reversible.
- **Resource assignment:** pick protected resources by *tag*, resource type, or ARN — policy decoupled from individual resources.
- **Cross-Region and cross-account copy:** built into the plan for DR and blast-radius isolation; Organizations integration pushes data-protection policies org-wide.
- **Coverage:** EBS, EC2, RDS/Aurora, DynamoDB, EFS, FSx, S3, DocumentDB, Neptune, Redshift, Storage Gateway, and more.
- **Backup Audit Manager:** frameworks that *prove* resources are backed up per policy. **Restore testing** automatically validates that backups are actually recoverable.
- **Continuous backup** enables point-in-time recovery (PITR) for supported services.

**Common traps:**
- Vault Lock *Compliance* vs *Governance* — compliance is irreversible.
- Backup protects and restores; **DataSync** *moves* live data. Different jobs.
- Vaults are per-Region resources; cross-Region DR means copy rules, not one global vault.

**Sources:** http://docs.aws.amazon.com/aws-backup/latest/devguide/cross-region-backup.html · http://aws.amazon.com/documentation-overview/backup/

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
