# Changelog — DOP-C02 Study Notes

> Track when each domain was last verified so you know what to recheck before the exam.
> Format: `YYYY-MM` | what was reviewed | who reviewed it.

---

## 2026-10 — Amazon Q Developer review

**Scope:** Full repository review following Muse AI's October 2026 corrections.

### Structural changes
- Added `DEPRECATED-SERVICES.md` — single reference for retired/restricted services
- Added `CHANGELOG.md` (this file)
- README.md Update Plan section added; Phases 1–3 completed

### Phase 1 — New files added
| File | Domain | Notes |
|---|---|---|
| `Domain-6-Security-Compliance/IAM-Identity-Center-General.md` | 6 | Full coverage; IAM-General.md section 12 updated to reference it |
| `Domain-6-Security-Compliance/Resource-Control-Policies-General.md` | 6 | RCPs new in 2024; SCP vs RCP comparison |
| `Domain-3-Resilient-Cloud-Solutions/Global-Accelerator-General.md` | 3 | Anycast IPs, traffic dials, vs Route 53/CloudFront |

### Phase 2 — New files added
| File | Domain | Notes |
|---|---|---|
| `Domain-3-Resilient-Cloud-Solutions/RDS-BlueGreen-General.md` | 3 | Launched Nov 2022; cross-ref added to RDS-Multi-AZ-General.md |
| `Domain-5-Incident-Event-Response/EventBridge-Pipes-General.md` | 5 | Launched Nov 2022; Pipes vs Rules distinction |
| `Domain-4-Monitoring-Logging/CloudWatch-Synthetics-General.md` | 4 | Canary blueprints, proactive vs reactive monitoring |
| `Domain-2-Configuration-Management-Infrastructure-as-Code/CDK-Pipelines-General.md` | 2 | Self-mutating pipelines, waves, cross-account bootstrap |
| `Domain-1-SDLC-Automation/GitHub-Actions-Integration/GitHub-OIDC-AWS.md` | 1 | OIDC setup, trust policy scoping, multi-account pattern |

### Phase 3 — New files added
| File | Domain | Notes |
|---|---|---|
| `Domain-6-Security-Compliance/Audit-Manager-General.md` | 6 | Evidence collection for auditors; vs Config vs Security Hub |
| `Domain-3-Resilient-Cloud-Solutions/Resilience-Hub-General.md` | 3 | Resilience score, RTO/RPO assessment, CI/CD integration |

### Existing files corrected (Muse AI — October 2026)
| File | What was fixed |
|---|---|
| `Cheatsheets/All-Services-Review.md` | Domain weights corrected (20%→17%, 20%→15%, 18%→14%, 15%→17%) |
| `Cheatsheets/Exam-Strategy.md` | Question count 65→75 (65 scored + 10 unscored); domain weights |
| `Cheatsheets/Final-Exam-Tips.md` | Question count and domain weights |
| `Cheatsheets/README.md` | Domain weights; study order updated |
| `Domain-1-SDLC-Automation/CodeStar-CodeCatalyst-General.md` | CodeStar marked retired; CodeCatalyst marked maintenance mode; CLI removed |
| `Domain-1-SDLC-Automation/Deployment-Strategies.md` | OpsWorks scenario replaced with Auto Scaling equivalent |
| `Domain-1-SDLC-Automation/CodePipeline/CodePipeline-DeepDive-CrossAccount.md` | Trust relationship direction corrected |
| `Domain-2-Configuration-Management-Infrastructure-as-Code/OpsWorks/` | Stub file deleted (service retired) |
| `Domain-3-Resilient-Cloud-Solutions/EKS-General.md` | Fargate profile creation corrected (CLI not kubectl); k8s.gcr.io → registry.k8s.io |
| `Domain-4-Monitoring-Logging/GuardDuty-General.md` | Container image scanning → Inspector not GuardDuty; Critical severity tier added; block_ip_address code fixed |
| `Domain-5-Incident-Event-Response/EventBridge-General.md` | Python bug fixed (iterated boolean) |
| `Domain-6-Security-Compliance/KMS-General.md` | Grants do not auto-expire — must be explicitly retired/revoked |
| `Domain-6-Security-Compliance/IAM-General.md` | Section 12 (Identity Center) expanded with link to dedicated file |
| `Domain-4-Monitoring-Logging/CloudWatch/CLOUDWATCH-General.md` | Stub replaced with exam-focused content |
| `Domain-4-Monitoring-Logging/Config/CONFIG-General.md` | Stub replaced with exam-focused content |
| `Domain-2-Configuration-Management-Infrastructure-as-Code/SystemsManager-SSM/SYSTEMSMANAGER-SSM-General.md` | Stub replaced with exam-focused content |

### Known gaps still open after this review

None — all identified gaps were resolved in Phases 1–3 above.

---

## 2026-02 — Muse AI expert review (original)

**Scope:** Full repository review. Score: 95/100.

### Key findings
- Domain weights were wrong in multiple files (original notes used incorrect percentages)
- Question count was 65 (wrong) — correct is 75 (65 scored + 10 unscored)
- OpsWorks was still present despite retirement
- CodeStar project service was still described as active
- Several stub files had no content
- GuardDuty had incorrect container scanning attribution and broken Python code
- KMS grants incorrectly described as auto-expiring
- CloudFormation drift detection had a non-existent CLI command

### Files added in this review
- `Domain-1-SDLC-Automation/AppConfig/APPCONFIG-General.md`
- `Domain-1-SDLC-Automation/CodeArtifact/CODEARTIFACT-General.md`
- `Domain-1-SDLC-Automation/CodePipeline/CodePipeline-V1-vs-V2.md`
- `Domain-2-Configuration-Management-Infrastructure-as-Code/CloudFormation/CloudFormation-Guard-Hooks.md`
- `Domain-3-Resilient-Cloud-Solutions/AWS-Backup-General.md`
- `Domain-3-Resilient-Cloud-Solutions/ECS-Deployments-General.md`
- `Domain-3-Resilient-Cloud-Solutions/ElastiCache-Resilience-General.md`
- `Domain-5-Incident-Event-Response/ChatOps-General.md`
- `Domain-5-Incident-Event-Response/EventBridge-Scheduler-General.md`
- `Domain-5-Incident-Event-Response/Incident-Manager-General.md`
- `Domain-6-Security-Compliance/Macie-General.md`

---

## 2025 — Original repository creation

**Scope:** Initial creation of all domain files, cheatsheets, and practice scenarios.
**Tool:** Amazon Q Developer.
**Coverage:** ~96% of DOP-C02 exam blueprint at time of creation.

---

*Update this file whenever you make significant content changes or verify a domain against the current exam guide.*
