# GitHub-Actions-Integration — General Notes (DOP-C02)

**What it is:** How GitHub Actions CI/CD connects to AWS — and how that connection shows up in DOP-C02 scenarios (hybrid pipelines, OIDC auth, CodePipeline integrations).

**Exam-testable facts:**
- **OIDC, not long-lived keys:** the exam-correct pattern is GitHub's OIDC provider federated to an AWS IAM role (`token.actions.githubusercontent.com`). Workflows assume the role with short-lived credentials. Storing `AWS_ACCESS_KEY_ID` as a repo secret is the anti-pattern the exam wants you to reject.
- **`aws-actions/configure-aws-credentials`:** the standard action that performs the OIDC role assumption (or uses static keys if you insist). Constrain the role with `Condition` on `sub` (repo/branch) so only the right workflow can assume it.
- **Two integration directions:**
  - *GitHub Actions → AWS:* deploy to AWS (CodeDeploy, S3, ECR/ECS, CloudFormation) from a workflow.
  - *AWS → GitHub:* CodePipeline source action via a **CodeStar connection** to a GitHub repo (push triggers the pipeline).
- **Self-hosted runners on AWS:** for custom networking or capacity, run Actions runners on EC2/ASG (the `actions-runner-controller` on EKS also appears). Exam angle: private-subnet access without exposing runners publicly.
- **CodePipeline alternative:** when a scenario says "keep everything in AWS-native services," the answer is CodePipeline, not GitHub Actions — know which the question is steering toward.

**Common traps:**
- OIDC federation vs stored access keys — the exam almost always wants OIDC.
- The IAM role's trust policy must allow the *specific* repo (`repo:org/name:*`); a wildcard trust is the planted wrong answer.
- GitHub-hosted runners can't reach private VPC resources — that's what self-hosted runners are for.

**Sources:** https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html

*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation link above.*
