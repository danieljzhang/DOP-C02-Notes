# GitHub Actions OIDC with AWS - DOP-C02 Exam Notes

> The stub file at `Github-Actions-Integration-General.md` covers the overview. This file goes deeper on the OIDC setup, trust policy anatomy, and exam scenarios.

## 1. Why OIDC Instead of Access Keys

The old pattern: store `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as GitHub repository secrets. Problems:
- Long-lived credentials that never expire
- Stored in GitHub's secret store (third-party)
- Must be rotated manually
- If leaked, valid until rotated

The OIDC pattern: GitHub acts as an identity provider. Each workflow run gets a short-lived OIDC token from GitHub. AWS exchanges that token for temporary STS credentials. No stored secrets.

**Exam rule:** "GitHub Actions accessing AWS without long-lived credentials" → OIDC federation.

---

## 2. How It Works

```
GitHub Actions workflow starts
    ↓
GitHub issues a signed OIDC JWT token for this workflow run
(contains: repo, branch, workflow, environment, etc.)
    ↓
Workflow calls aws-actions/configure-aws-credentials action
    ↓
Action calls sts:AssumeRoleWithWebIdentity
  - presents the JWT to AWS STS
  - specifies the IAM role ARN to assume
    ↓
AWS STS validates the JWT against GitHub's OIDC provider
  (fetches GitHub's public keys from token.actions.githubusercontent.com)
    ↓
STS returns temporary credentials (valid 1 hour by default)
    ↓
Workflow uses credentials for AWS API calls
```

---

## 3. Setup — Step by Step

### Step 1: Create the OIDC Provider in IAM (once per AWS account)

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Or in CloudFormation:
```yaml
GitHubOIDCProvider:
  Type: AWS::IAM::OIDCProvider
  Properties:
    Url: https://token.actions.githubusercontent.com
    ClientIdList:
      - sts.amazonaws.com
    ThumbprintList:
      - 6938fd4d98bab03faadb97b34396831e3780aea1
```

### Step 2: Create the IAM Role with a Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:*"
        }
      }
    }
  ]
}
```

**Critical:** The `sub` condition scopes the trust to a specific repo. Without it, any GitHub repo could assume this role.

### Step 3: Use in the Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # required to request the OIDC JWT
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1
          # role-session-name is optional; defaults to GitHubActions

      - name: Deploy
        run: aws s3 sync ./dist s3://my-bucket/
```

---

## 4. Trust Policy Conditions — Scoping Options

The `sub` claim in the JWT identifies the workflow context. You can scope the trust to:

| Condition | `sub` value | Allows |
|---|---|---|
| Any ref in a repo | `repo:org/name:*` | Any branch, tag, PR |
| Specific branch only | `repo:org/name:ref:refs/heads/main` | Only `main` branch |
| Specific environment | `repo:org/name:environment:production` | Only `production` environment |
| Pull requests | `repo:org/name:pull_request` | Only PR workflows |

**Exam trap:** A wildcard `*` on the `sub` condition (or omitting it entirely) lets any GitHub repo assume the role — the exam will present this as the wrong answer.

---

## 5. Multiple Accounts — Hub and Spoke Pattern

For deploying to multiple AWS accounts from one GitHub workflow:

```
GitHub Actions workflow
    ↓ OIDC → assume role in CICD account
    ↓ then assume cross-account role in target account
    ↓ deploy to dev / staging / prod
```

```yaml
- name: Configure AWS credentials (CICD account)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::CICD_ACCOUNT:role/GitHubActionsRole
    aws-region: us-east-1

- name: Assume role in prod account
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::PROD_ACCOUNT:role/DeployRole
    aws-region: us-east-1
    role-chaining: true  # chains from the CICD account credentials
```

---

## 6. GitHub Environments for Deployment Protection

GitHub Environments add approval gates and environment-specific secrets:

```yaml
jobs:
  deploy-prod:
    environment: production   # requires approval from configured reviewers
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ProdDeployRole
          # trust policy scoped to: repo:org/name:environment:production
```

The IAM role trust policy uses `environment:production` in the `sub` condition — only workflows running in the `production` environment can assume the prod role.

---

## 7. OIDC vs CodeStar Connections — Which to Use

| Scenario | Use |
|---|---|
| GitHub Actions workflow deploying to AWS | OIDC (`configure-aws-credentials`) |
| CodePipeline triggered by GitHub push | CodeStar connection |
| Both GitHub Actions AND CodePipeline | Both — different integration points |

---

## 8. Common Exam Scenarios

### Scenario 1: GitHub Actions deploys to AWS without stored credentials
**Solution:** OIDC provider in IAM + IAM role with trust policy scoped to the repo. Workflow uses `configure-aws-credentials` with `role-to-assume`. No secrets stored in GitHub.

### Scenario 2: Only the main branch can deploy to production
**Solution:** Trust policy `sub` condition: `repo:org/name:ref:refs/heads/main`. Feature branch workflows get credentials denied when they try to assume the prod role.

### Scenario 3: Require human approval before prod deployment
**Solution:** GitHub Environment named `production` with required reviewers. Trust policy scoped to `environment:production`. Workflow pauses for approval before the deploy job runs.

### Scenario 4: Deploy to multiple AWS accounts from one workflow
**Solution:** OIDC to a CICD account role, then cross-account role assumption to each target account using `role-chaining: true`.

---

## 9. Exam Tips

- `permissions: id-token: write` in the workflow YAML is **required** — without it, GitHub won't issue the OIDC token. This is a common trap.
- The OIDC provider is created **once per AWS account**, not per repo or role.
- Scope the trust policy `sub` condition to the **specific repo** — wildcard org-level trust is the planted wrong answer.
- OIDC credentials are **temporary** (default 1 hour) — no rotation needed.
- "No long-lived credentials", "no stored secrets", "short-lived tokens" → OIDC.
- The `aud` condition must be `sts.amazonaws.com` — this is fixed, not configurable.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
