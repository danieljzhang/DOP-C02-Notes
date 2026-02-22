Below is a clear, **actionable step-by-step guide** to create an **AWS CodeStar Connection** and install the **AWS CodeStar Connections GitHub App** correctly so your CodePipeline will receive push events. I’ll include verification steps and troubleshooting tips so you can confirm the webhook got created and fix the common problems.

---

# Quick checklist (high level)

1. Create a CodeStar Connection in AWS Console → this starts the GitHub OAuth/install flow.
2. On GitHub, **install** the “AWS CodeStar Connections” App and grant it access to the repo.
3. Confirm the GitHub App is listed in **GitHub → Settings → Installed GitHub Apps** (or Org Installations).
4. Ensure your pipeline source action has `DetectChanges = true` and the correct `ConnectionArn`, `FullRepositoryId`, `BranchName`.
5. Verify webhook exists in GitHub repo → test a push → pipeline triggers.

---

## Step 1 — Create a CodeStar Connection (AWS Console)

1. Sign in to the AWS account used for the pipeline.
2. In the AWS Console search, type **“Developer Tools”** → click **Connections** (or search **CodeStar Connections**).
3. Click **Create connection**.
4. Provider: **GitHub** (or **GitHub Enterprise** if applicable).
5. Click **Connect** (this will redirect you to GitHub).

   * If prompted, sign in to **the GitHub account that has access to the repository** (very important).
6. On GitHub, you’ll see the install/authorize screen for **AWS CodeStar Connections**.

   * Choose **All repositories** or **Only select repositories** (if you choose “Only select”, make sure to check the repo you want).
   * Click **Install** / **Authorize**.
7. Back in the AWS Console, you should now see the new connection and an **ARN**. Note the ARN (copy it).

> If the GitHub install page never appears when you click Connect, try an incognito window or disable content blockers. Also ensure you're logged into the correct GitHub account.

---

## Step 2 — Verify GitHub App is installed (GitHub side)

1. Go to GitHub → your **repo** → **Settings** → **Installed GitHub Apps** (or for org-level install, Organization → Settings → Installed GitHub Apps / Apps → Installations).
2. Confirm **AWS CodeStar Connections** is listed and **the repository is included** in its access.

   * If it’s installed at org-level, click it and verify the repo is selected.
3. If you don’t see the app: you didn’t finish the AWS → GitHub install flow. Repeat Step 1 and make sure to complete the GitHub authorization.

---

## Step 3 — Confirm the Connection in AWS

1. AWS Console → Developer Tools → Connections → find your connection.
2. Status should be **Available**.
3. Copy the **Connection ARN** to use in Terraform or pipeline config.

**(Optional AWS CLI checks)**

```bash
aws codestar-connections list-connections
# or get the specific connection
aws codestar-connections get-connection --connection-arn arn:aws:codestar-connections:...
```

---

## Step 4 — Ensure CodePipeline source action will create webhook

In your Terraform `aws_codepipeline` source action configuration, you **must** include:

```hcl
configuration = {
  ConnectionArn    = var.codestar_connection_arn
  FullRepositoryId = "${var.github_owner}/${var.github_repo}"
  BranchName       = var.github_branch
  DetectChanges    = true          # ❗ REQUIRED to auto-create webhooks/triggering
}
```

If you change Terraform, run:

```bash
terraform apply
```

This causes CodePipeline to register the webhook and EventBridge rule.

---

## Step 5 — Verify the webhook was created in GitHub

1. Go to your GitHub repo → **Settings → Webhooks**.
2. You should see a webhook URL that looks like an AWS connections endpoint (it will be created automatically).
3. Click the webhook → **Recent Deliveries** to view events and responses.

   * Successful deliveries should show **200** or **202** responses.

If there is no webhook after `terraform apply`, go back to Step 1 and ensure the GitHub App was installed correctly and `DetectChanges` is set.

---

## Step 6 — Test a commit and watch the pipeline

1. Make a small commit and push to the configured branch (e.g., `main`).
2. In AWS Console → CodePipeline → your pipeline → check the **Execution history**.
3. If the pipeline was triggered, you’ll see a new execution; otherwise check logs below.

---

## Troubleshooting — if pipeline still doesn’t trigger

### A. Confirm GitHub App installation & account

* Make sure you installed the **app with the same GitHub account that owns the repository** or that the organization admin installed it for your repo.
* If your repo is in an **organization**, an organization admin must approve GitHub App installation.

### B. Confirm Connection status in AWS is **Available**

* If status is **Pending** or **Failed**, re-run the install flow.

### C. Browser issues

* OAuth/redirect can be blocked by ad blockers or privacy extensions — use an incognito window.

### D. Permissions in Terraform/IAM

* Ensure the **CodePipeline role** has this permission:

  ```json
  "codestar-connections:UseConnection"
  ```

  If not, the webhook creation may fail silently.

### E. Manually re-create Source stage (quick fix)

* In CodePipeline → Edit pipeline → Delete Source stage → Add Source stage again (select your CodeStar connection and enable DetectChanges). This will force AWS to recreate webhook.

### F. Check EventBridge rules

* Go to EventBridge → Rules → look for a rule created for your pipeline source webhook. If missing, webhook registration failed.

### G. Check GitHub webhook delivery failures

* In GitHub → Webhooks → recent deliveries — review any non-200 responses and error details.

---

## Extra commands (AWS CLI) to help debug

* List your CodePipeline webhooks (if any):

```bash
aws codepipeline list-webhooks
```

* Get pipeline details:

```bash
aws codepipeline get-pipeline --name <pipeline-name>
```

* Check connections:

```bash
aws codestar-connections list-connections
```

---
