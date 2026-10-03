# Deprecated and Restricted AWS Services - DOP-C02 Reference

> Single reference for services that are retired, restricted, or in maintenance mode. Kept here so warnings aren't scattered across individual service files. Check this before the exam — the status of some services changed in 2024–2025.

---

## AWS OpsWorks (All Flavours) — Retired 2024

| Variant | End of life |
|---|---|
| OpsWorks Stacks | May 26, 2024 |
| OpsWorks for Chef Automate | May 26, 2024 |
| OpsWorks for Puppet Enterprise | May 26, 2024 |

**Status:** Fully retired. The service no longer exists. The EventBridge source `aws.opsworks` no longer emits events.

**Exam impact:**
- If an answer choice uses `aws.opsworks` as an EventBridge source → it is a distractor, eliminate it.
- OpsWorks auto-healing scenarios in old study materials should be replaced with Auto Scaling equivalents (see `Domain-1-SDLC-Automation/Deployment-Strategies.md`).
- The `aws opsworks` CLI namespace is gone.

**Modern replacement:** Systems Manager (Run Command, State Manager, Patch Manager) for configuration management. Chef/Puppet can still be self-managed on EC2 if needed.

---

## AWS CodeStar (Project Service) — Retired July 31, 2024

**Status:** Fully retired. Support ended July 31, 2024. The `aws codestar` CLI namespace was removed.

**Exam impact:**
- `aws codestar` CLI commands in answer choices → distractors, eliminate them.
- ⚠️ **Do not confuse with CodeStar connections** — `aws codestar-connections` is a **live, fully supported** service for connecting CodePipeline to GitHub/GitLab/Bitbucket. Different service, similar name.

**Modern replacement:** CodePipeline + CodeBuild + CodeDeploy directly, or GitHub Actions.

---

## Amazon CodeCatalyst — Maintenance Mode (November 2025)

**Status:** Closed to new customers November 7, 2025. Existing customers continue normally. No new features being added (maintenance mode).

**Exam impact:**
- Unlikely to appear as the correct answer for new architecture questions.
- Know what it was: a unified development environment (spaces, projects, dev environments, workflows).
- The exam may still reference it in historical context — know it existed and what it did.

**Modern replacement:** For new projects, use CodePipeline + CodeBuild + CodeDeploy (or GitHub Actions).

---

## AWS CodeCommit — Status Uncertain

**Status:** Restricted to existing customers July 25, 2024 (new customers could not create repositories). Subsequent status changes have been reported but not independently verified.

**Exam impact:**
- Treat with caution — verify the current status on the [AWS CodeCommit service page](https://aws.amazon.com/codecommit/) before the exam.
- The service's core concepts (Git repositories, IAM-based auth, EventBridge triggers, CodePipeline source) remain valid exam knowledge regardless of availability status.
- If the exam presents CodeCommit as a source in a CI/CD scenario, it is still a valid answer — the exam blueprint may not reflect real-time service availability changes.

**Alternative:** GitHub/GitLab/Bitbucket via CodeStar connections for new architectures.

---

## AWS Proton — Reduced Investment

**Status:** Still available but AWS has significantly reduced investment. Not actively promoted for new use cases.

**Exam impact:** Very low exam frequency. If it appears, know it was for platform teams to provide self-service infrastructure templates to application teams.

---

## Quick Reference: Dead CLI Namespaces

If you see these in exam answer choices, they are distractors:

| CLI namespace | Status |
|---|---|
| `aws opsworks` | ❌ Removed (service retired May 2024) |
| `aws codestar` | ❌ Removed (service retired July 2024) |
| `aws codestar-connections` | ✅ Live (GitHub/GitLab/Bitbucket connector) |
| `aws codecommit` | ⚠️ Uncertain — verify before exam |
| `aws codecatalyst` | ⚠️ Maintenance mode — existing customers only |

---

*Last updated: October 2026 (Amazon Q review). Verify service statuses against official AWS documentation before your exam date.*
