# AWS CodeCommit - DOP-C02 Exam Notes

## 1. Overview

**AWS CodeCommit** is a fully managed source control service that hosts secure Git-based repositories. It eliminates the need to operate your own source control system or worry about scaling its infrastructure.

### Key Characteristics
- **Fully managed Git** - Standard Git functionality
- **Secure** - Encrypted at rest and in transit
- **Scalable** - No repository size limits
- **Highly available** - Redundant storage across multiple facilities
- **Integrated** - Works with existing Git tools and AWS services
- **No servers** - No infrastructure to manage

### What Problem Does It Solve?
- Eliminates need for self-hosted Git servers
- Provides secure, compliant source control
- Integrates natively with AWS CI/CD services
- Scales automatically without capacity planning

---

## 2. Core Concepts

### Repositories
- Git repositories hosted in AWS
- Support all standard Git operations (clone, push, pull, branch, merge)
- No size limits on repositories
- Encrypted by default

### Branches
- Standard Git branching model
- Default branch (usually main or master)
- Feature branches, release branches, hotfix branches
- Branch permissions via IAM

### Commits
- Standard Git commits
- Immutable once pushed
- Tracked with commit ID (SHA-1 hash)

### Pull Requests
- Code review mechanism
- Approval rules can be enforced
- Integration with approval rule templates
- Comments and discussions
- Merge strategies: Fast-forward, Squash, Three-way

### Triggers
- Automated actions on repository events
- Can invoke SNS topics or Lambda functions
- Events: Push to branch, create/delete branch, create/delete tag

### Notifications
- EventBridge integration for repository events
- SNS notifications for pull request events
- CloudWatch Events (legacy, use EventBridge)

---

## 3. Authentication & Access Control

### Authentication Methods

#### HTTPS (Git Credentials)
- Generate Git credentials in IAM console
- Username and password for Git operations
- Recommended for most users
- Credentials are IAM user-specific

#### HTTPS (AWS CLI Credential Helper)
- Uses AWS CLI credentials
- No separate Git credentials needed
- Automatically refreshes credentials
- Configure in `.gitconfig`:
```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

#### SSH
- Upload SSH public key to IAM user
- Use SSH key for authentication
- SSH Key ID required in SSH config
- Configure in `~/.ssh/config`:
```
Host git-codecommit.*.amazonaws.com
  User APKAEIBAERJR2EXAMPLE
  IdentityFile ~/.ssh/codecommit_rsa
```

#### Federated Access (SAML/OIDC)
- Use temporary credentials from federation
- Requires credential helper
- Good for enterprise environments

### IAM Permissions

#### Repository-Level Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codecommit:GitPull",
        "codecommit:GitPush"
      ],
      "Resource": "arn:aws:codecommit:us-east-1:123456789012:MyRepo"
    }
  ]
}
```

#### Common Actions
- `codecommit:GitPull` - Clone, fetch, pull
- `codecommit:GitPush` - Push commits
- `codecommit:CreateBranch` - Create branches
- `codecommit:DeleteBranch` - Delete branches
- `codecommit:CreatePullRequest` - Create pull requests
- `codecommit:MergePullRequest` - Merge pull requests
- `codecommit:GetBranch` - View branch details
- `codecommit:ListRepositories` - List repositories

#### Branch-Level Permissions
- Restrict push/merge to specific branches
- Protect main/production branches
- Example: Deny direct push to main
```json
{
  "Effect": "Deny",
  "Action": [
    "codecommit:GitPush",
    "codecommit:DeleteBranch",
    "codecommit:PutFile",
    "codecommit:MergeBranchesByFastForward"
  ],
  "Resource": "arn:aws:codecommit:us-east-1:123456789012:MyRepo",
  "Condition": {
    "StringEqualsIfExists": {
      "codecommit:References": [
        "refs/heads/main"
      ]
    }
  }
}
```

---

## 4. Security

### Encryption

#### At Rest
- Encrypted using AWS KMS
- Automatic encryption
- Can use AWS managed key or customer managed key
- Applies to all repository data

#### In Transit
- All data encrypted via HTTPS or SSH
- TLS 1.2 or higher
- Git protocol over HTTPS/SSH

### Cross-Account Access
- Use IAM roles for cross-account access
- Resource-based policies not supported
- Assume role from another account
- Example:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Compliance
- HIPAA eligible
- PCI DSS compliant
- SOC compliant
- ISO compliant
- FedRAMP authorized

---

## 5. Pull Requests & Approval Rules

### Pull Request Workflow
1. Create feature branch
2. Make changes and commit
3. Create pull request
4. Code review and comments
5. Approval (if rules configured)
6. Merge to target branch

### Approval Rules
- Require approvals before merge
- Define number of approvals needed
- Specify approval pool (IAM users/roles)
- Can use approval rule templates

### Approval Rule Template
```json
{
  "Version": "2018-11-08",
  "DestinationReferences": ["refs/heads/main"],
  "Statements": [
    {
      "Type": "Approvers",
      "NumberOfApprovalsNeeded": 2,
      "ApprovalPoolMembers": [
        "arn:aws:sts::123456789012:assumed-role/CodeReviewRole/*"
      ]
    }
  ]
}
```

### Merge Strategies
- **Fast-forward** - Linear history, no merge commit
- **Squash** - Combine all commits into one
- **Three-way merge** - Creates merge commit

---

## 6. Triggers & Notifications

### Triggers
- Execute actions on repository events
- Target: SNS topic or Lambda function
- Events:
  - All repository events
  - Push to existing branch
  - Create branch or tag
  - Delete branch or tag

### Trigger Configuration
```json
{
  "repositoryName": "MyRepo",
  "trigger": {
    "name": "MyTrigger",
    "destinationArn": "arn:aws:sns:us-east-1:123456789012:MyTopic",
    "events": ["all"],
    "branches": ["main", "develop"]
  }
}
```

### EventBridge Integration
- More flexible than triggers
- Filter events with patterns
- Multiple targets per rule
- Events:
  - Pull request state change
  - Comment on pull request
  - Comment on commit
  - Branch/tag reference created/deleted
  - Repository state change

### EventBridge Pattern Example
```json
{
  "source": ["aws.codecommit"],
  "detail-type": ["CodeCommit Pull Request State Change"],
  "detail": {
    "event": ["pullRequestCreated"],
    "repositoryNames": ["MyRepo"]
  }
}
```

---

## 7. Integration with CI/CD

### CodePipeline Integration
- CodeCommit as source stage
- Automatic pipeline trigger on commit
- Branch filtering
- Detect changes via CloudWatch Events

### CodeBuild Integration
- Build source code from CodeCommit
- Webhook triggers on push
- Pull request builds
- Branch-specific builds

### Third-Party Integration
- Jenkins via AWS CodeCommit plugin
- GitHub Actions via OIDC
- GitLab CI via mirroring
- Any Git-compatible tool

---

## 8. Migration & Mirroring

### Migrate from Other Git Providers

#### From GitHub
```bash
git clone --mirror https://github.com/user/repo.git
cd repo.git
git push https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyRepo --all
git push https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyRepo --tags
```

#### From Bitbucket/GitLab
- Same process as GitHub
- Use `git clone --mirror` and `git push --mirror`

### Repository Mirroring
- Keep CodeCommit in sync with external Git
- Use scheduled Lambda or CodeBuild
- Bidirectional or unidirectional sync

---

## 9. Monitoring & Logging

### CloudWatch Metrics
- No built-in metrics for CodeCommit
- Create custom metrics via EventBridge + Lambda

### CloudWatch Logs
- No direct logging to CloudWatch
- Use triggers/EventBridge to log events

### AWS CloudTrail
- Logs all API calls to CodeCommit
- Tracks:
  - Repository creation/deletion
  - Branch creation/deletion
  - IAM permission changes
  - Pull request actions
- Does NOT log Git operations (push, pull, clone)

### CloudTrail Event Example
```json
{
  "eventName": "CreateRepository",
  "eventSource": "codecommit.amazonaws.com",
  "requestParameters": {
    "repositoryName": "MyRepo"
  },
  "responseElements": {
    "repositoryMetadata": {
      "repositoryId": "12345678-1234-1234-1234-123456789012"
    }
  }
}
```

---

## 10. Best Practices

### Security
- Use IAM roles instead of IAM users where possible
- Enable MFA for sensitive repositories
- Use branch protection for main/production branches
- Implement approval rules for critical branches
- Rotate Git credentials regularly
- Use credential helper instead of storing passwords
- Enable CloudTrail logging

### Repository Management
- Use meaningful branch names
- Implement branching strategy (GitFlow, trunk-based)
- Keep repositories focused (single purpose)
- Use tags for releases
- Regular cleanup of stale branches
- Document repository structure in README

### Code Review
- Require pull requests for all changes to main
- Use approval rules with minimum reviewers
- Use approval rule templates for consistency
- Comment on code during review
- Link pull requests to issue tracking

### CI/CD Integration
- Trigger builds on every commit
- Use branch-specific pipelines
- Implement automated testing
- Use CodePipeline for full automation
- Tag releases for deployment tracking

### Performance
- Use shallow clones for large repositories
- Use Git LFS for large binary files
- Keep repository size manageable
- Archive old repositories

---

## 11. Common Exam Scenarios

### Scenario 1: Developer cannot push to repository
**Possible Causes:**
- Missing `codecommit:GitPush` permission
- Branch protection policy denies push
- Authentication failure (expired credentials)

**Solution:**
- Verify IAM permissions
- Check branch-level restrictions
- Regenerate Git credentials or configure credential helper

### Scenario 2: Need to restrict direct pushes to main branch
**Solution:**
- Create IAM policy with Deny on `codecommit:GitPush` for main branch
- Require pull requests with approval rules
- Use condition: `codecommit:References: refs/heads/main`

### Scenario 3: Trigger Lambda on every commit to main
**Solution:**
- Create CodeCommit trigger targeting Lambda
- Or use EventBridge rule with pattern matching
- Filter by branch name in trigger configuration

### Scenario 4: Cross-account access to repository
**Solution:**
- Create IAM role in repository account
- Grant assume role permission to external account
- External account assumes role to access repository
- Use credential helper with assumed role credentials

### Scenario 5: Migrate from GitHub to CodeCommit
**Solution:**
- Use `git clone --mirror` to clone GitHub repo
- Push to CodeCommit with `git push --mirror`
- Update CI/CD pipelines to use CodeCommit
- Update developer Git remotes

### Scenario 6: Require 2 approvals before merging to production
**Solution:**
- Create approval rule template
- Set `NumberOfApprovalsNeeded: 2`
- Specify approval pool members
- Apply to pull requests targeting production branch

### Scenario 7: Audit who made changes to repository
**Solution:**
- Enable CloudTrail logging
- Query CloudTrail logs for CodeCommit events
- Note: Git operations (push/pull) not logged in CloudTrail
- Use Git commit history for code changes

### Scenario 8: Automate notifications for pull request events
**Solution:**
- Create EventBridge rule for pull request events
- Target SNS topic for email notifications
- Or target Lambda for custom processing
- Filter by repository and event type

---

## 12. Comparison with Other Git Services

### CodeCommit vs GitHub
| Feature | CodeCommit | GitHub |
|---------|------------|--------|
| Hosting | AWS-managed | GitHub-managed |
| Authentication | IAM | GitHub accounts |
| Cost | Pay per user | Free/paid tiers |
| Integration | AWS-native | Broader ecosystem |
| Actions/CI | Use CodePipeline | GitHub Actions |

### CodeCommit vs GitLab
| Feature | CodeCommit | GitLab |
|---------|------------|--------|
| Self-hosted | No | Yes (option) |
| CI/CD | Separate services | Integrated |
| Issue tracking | No | Yes |
| Wiki | No | Yes |

### CodeCommit vs Bitbucket
| Feature | CodeCommit | Bitbucket |
|---------|------------|-----------|
| Hosting | AWS | Atlassian Cloud |
| Integration | AWS services | Jira, Confluence |
| Pipelines | CodePipeline | Bitbucket Pipelines |
| Authentication | IAM | Atlassian accounts |

---

## 13. Limitations & Quotas

### Service Limits
- **Repository name**: 1-100 characters
- **File size**: 2 GB per file (use Git LFS for larger)
- **Commit size**: 2 GB
- **Branch/tag names**: 256 characters
- **Number of repositories**: 5,000 per account (soft limit)
- **Concurrent connections**: Varies by region

### API Rate Limits
- Most APIs: 1,000 requests per second
- Git operations: No documented limit
- Can request limit increases via AWS Support

---

## 14. Troubleshooting

### Common Issues

#### Authentication Failures
- **Symptom**: "fatal: Authentication failed"
- **Causes**: Expired credentials, wrong region, incorrect username
- **Fix**: Regenerate Git credentials, verify region in URL, check IAM permissions

#### Permission Denied
- **Symptom**: "fatal: unable to access repository"
- **Causes**: Missing IAM permissions, branch protection
- **Fix**: Add required IAM permissions, check branch policies

#### Repository Not Found
- **Symptom**: "fatal: repository not found"
- **Causes**: Wrong repository name, wrong region, no access
- **Fix**: Verify repository name and region, check IAM permissions

#### Large File Push Fails
- **Symptom**: Push fails with size error
- **Causes**: File exceeds 2 GB limit
- **Fix**: Use Git LFS for large files

#### Slow Clone/Pull
- **Symptom**: Git operations take long time
- **Causes**: Large repository, network issues
- **Fix**: Use shallow clone, check network connectivity

---

## 15. CLI Commands Reference

### Repository Operations
```bash
# Create repository
aws codecommit create-repository --repository-name MyRepo

# List repositories
aws codecommit list-repositories

# Get repository details
aws codecommit get-repository --repository-name MyRepo

# Delete repository
aws codecommit delete-repository --repository-name MyRepo
```

### Branch Operations
```bash
# List branches
aws codecommit list-branches --repository-name MyRepo

# Get branch details
aws codecommit get-branch --repository-name MyRepo --branch-name main

# Create branch
aws codecommit create-branch --repository-name MyRepo \
  --branch-name feature --commit-id abc123
```

### Pull Request Operations
```bash
# Create pull request
aws codecommit create-pull-request \
  --title "My PR" \
  --targets repositoryName=MyRepo,sourceReference=feature,destinationReference=main

# List pull requests
aws codecommit list-pull-requests --repository-name MyRepo

# Get pull request
aws codecommit get-pull-request --pull-request-id 1

# Merge pull request
aws codecommit merge-pull-request-by-fast-forward --pull-request-id 1
```

### Trigger Operations
```bash
# Create trigger
aws codecommit put-repository-triggers --repository-name MyRepo \
  --triggers file://trigger.json

# List triggers
aws codecommit get-repository-triggers --repository-name MyRepo

# Test trigger
aws codecommit test-repository-triggers --repository-name MyRepo \
  --triggers file://trigger.json
```

---

## 16. Architecture Patterns

### Basic CI/CD Pipeline
```
Developer → Git Push → CodeCommit
                           ↓
                    EventBridge Rule
                           ↓
                      CodePipeline
                           ↓
                  CodeBuild → CodeDeploy
```

### Multi-Environment Deployment
```
CodeCommit (main branch)
    ↓
CodePipeline
    ├─→ Dev Environment (auto-deploy)
    ├─→ Test Environment (auto-deploy)
    └─→ Prod Environment (manual approval)
```

### Pull Request Automation
```
Pull Request Created
    ↓
EventBridge Rule
    ↓
Lambda Function
    ├─→ Run CodeBuild (tests)
    ├─→ Post results as comment
    └─→ Notify reviewers (SNS)
```

### Cross-Account Repository Access
```
Account A (Repository)
    ↓
IAM Role (with trust policy)
    ↓
Account B (Developer)
    ↓
Assume Role → Access Repository
```

---

## 17. Exam Tips

### What to Remember
- **Authentication**: Git credentials, credential helper, SSH keys
- **IAM permissions**: GitPull, GitPush, branch-level restrictions
- **Encryption**: Automatic at rest (KMS), in transit (TLS)
- **Triggers**: SNS or Lambda on repository events
- **EventBridge**: More flexible than triggers, multiple targets
- **Approval rules**: Require approvals before merge
- **CloudTrail**: Logs API calls, NOT Git operations
- **Cross-account**: Use IAM roles, not resource policies
- **Branch protection**: Use IAM conditions with codecommit:References

### Common Traps
- CloudTrail does NOT log Git push/pull operations
- CodeCommit does NOT have resource-based policies (use IAM roles)
- Git credentials are IAM user-specific, not account-wide
- Branch protection requires IAM policy with Deny + Condition
- Triggers are less flexible than EventBridge rules

### Scenario-Based Questions
- Focus on IAM permissions and branch protection
- Understand trigger vs EventBridge differences
- Know how to set up cross-account access
- Understand approval rules for pull requests
- Know migration strategies from other Git providers

### Integration Questions
- CodeCommit → CodePipeline → CodeBuild → CodeDeploy
- EventBridge for event-driven automation
- Lambda for custom processing of repository events
- SNS for notifications

---

## 18. Quick Reference Cheat Sheet

### Essential IAM Permissions
```
codecommit:GitPull          # Clone, fetch, pull
codecommit:GitPush          # Push commits
codecommit:CreateBranch     # Create branches
codecommit:DeleteBranch     # Delete branches
codecommit:CreatePullRequest # Create PR
codecommit:MergePullRequest  # Merge PR
```

### Git Credential Helper Setup
```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

### Clone Repository
```bash
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyRepo
```

### Branch Protection Pattern
```json
{
  "Condition": {
    "StringEqualsIfExists": {
      "codecommit:References": ["refs/heads/main"]
    }
  }
}
```

---

## 19. Reference Links

### AWS Official Documentation
- [CodeCommit User Guide](https://docs.aws.amazon.com/codecommit/latest/userguide/welcome.html)
- [IAM Permissions Reference](https://docs.aws.amazon.com/codecommit/latest/userguide/auth-and-access-control-permissions-reference.html)
- [Setup for HTTPS Users](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html)
- [Setup for SSH Users](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html)
- [Approval Rules](https://docs.aws.amazon.com/codecommit/latest/userguide/approval-rules.html)
- [Triggers](https://docs.aws.amazon.com/codecommit/latest/userguide/how-to-notify.html)

---

## 20. Summary

AWS CodeCommit is a foundational service for AWS CI/CD pipelines and appears frequently in DOP-C02 exam scenarios. Key areas to master:

1. **IAM authentication and authorization** (Git credentials, credential helper, SSH)
2. **Branch-level permissions** using IAM conditions
3. **Pull requests and approval rules** for code review workflows
4. **Triggers and EventBridge integration** for automation
5. **Cross-account access** using IAM roles
6. **Security** (encryption, CloudTrail, compliance)
7. **Integration with CodePipeline and CodeBuild**
8. **Migration strategies** from other Git providers
9. **Troubleshooting** authentication and permission issues
10. **Best practices** for repository management and CI/CD

Understanding these concepts with hands-on practice will ensure success on CodeCommit-related questions in the DOP-C02 exam.
