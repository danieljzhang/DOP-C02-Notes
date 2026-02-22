# Implementing Deployment Strategies

For the DOP-C02 exam, you must have a deep understanding of various deployment strategies, their configurations, and their application across different AWS compute platforms like EC2, ECS, and Lambda.

## Core Deployment Strategies

| Strategy | Description | Pros | Cons | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **In-Place (Rolling)** | The application on each instance is stopped, the new version is installed, and the new version is started. The number of instances in service can decrease during the deployment. | Simple, fast for small fleets. No additional cost for new instances. | Can reduce capacity. Rollback is complex (requires redeploying the old version). | Development environments, applications that can tolerate a temporary reduction in capacity. |
| **Rolling with an additional batch** | A new batch of instances is launched to ensure full capacity during the deployment. As new instances come online, old ones are terminated. | Maintains full capacity. Safer than a standard rolling update. | Slower than in-place. Temporarily increases costs. | Production applications where maintaining capacity is important. |
| **Immutable** | A new Auto Scaling Group with new instances running the new application version is launched. Once healthy, traffic is switched. The old ASG is then terminated. | Highly reliable, easy rollback (just terminate new ASG). No configuration drift. | Slower deployment. Higher temporary cost. Can be complex to set up. | Mission-critical applications, implementing immutable infrastructure principles. |
| **Blue/Green** | Two identical, independent environments ("Blue" and "Green"). New version is deployed to the idle environment (Green). After testing, traffic is switched from Blue to Green. | Instant traffic switching and rollback. No downtime. Test in a production-like environment. | Can be expensive (doubles resource cost). Complex database schema changes. | Critical applications requiring zero downtime. |
| **Canary / Linear** | Traffic is gradually shifted from the old version to the new version in increments (e.g., 10% of traffic for 15 mins, then 50%, then 100%). | Safest deployment. Catches issues with a small blast radius. Allows for performance monitoring. | Slowest deployment method. Can be complex to manage traffic shifting. | High-risk changes, performance-sensitive applications. |

## Deployments to EC2/On-Premises with CodeDeploy

*   **CodeDeploy Agent:** Must be installed and running on all EC2 instances and on-premises servers in the deployment group. The agent polls CodeDeploy for new revisions to deploy.
*   **AppSpec File (`appspec.yml`):** The core configuration file that tells the CodeDeploy agent what to do at each stage of the deployment.
    *   **`files` section:** Specifies source files from the application revision to copy to the destination on the instance.
    *   **`hooks` section:** Defines scripts to run at specific lifecycle events. This is critical for exam questions.

### EC2/On-Premises Lifecycle Hooks

| Hook | Description | Use Case Example |
| :--- | :--- | :--- |
| `ApplicationStop` | Runs before the application revision is downloaded. | Gracefully stop the application service. |
| `DownloadBundle` | The agent copies the revision from S3 to a temp location. | (No script execution) |
| `BeforeInstall` | Runs before the `Install` hook. | Back up the current version, decrypt files, **purge temporary files from previous deployments**. |
| `Install` | The agent copies files from the temp location to the final destination. | (No script execution) |
| `AfterInstall` | Runs after the `Install` hook. | Configure application properties, set file permissions. |
| `ApplicationStart` | Runs after `AfterInstall`. | Start the application services. |
| `ValidateService` | The final hook in the deployment. | Run health checks, integration tests to confirm the application is working correctly. |

## Deployments to Amazon ECS

ECS has three deployment types: Rolling Update, **Blue/Green (via CodeDeploy)**, and External.

*   **Rolling Update:** The default method. The ECS service scheduler stops old tasks and starts new tasks. You can configure `minimumHealthyPercent` and `maximumPercent` to control the process.
*   **Deployment Circuit Breaker:** A safety mechanism for rolling updates. It monitors deployment failures and can automatically roll back the service to the last stable state. This is a faster way to detect failures than relying solely on CloudWatch alarms.
*   **Blue/Green with CodeDeploy:** The most robust method. It uses an Application Load Balancer with two target groups and a test listener to validate the new version before shifting production traffic.

## Deployments to AWS Lambda (Serverless)

*   **Versioning and Aliases:** This is fundamental.
    *   **Versions:** An immutable snapshot of your function's code and configuration (`$LATEST` is mutable).
    *   **Aliases:** A pointer to a specific version (e.g., `PROD` points to version `5`). You can shift traffic between two versions associated with a single alias.
*   **CodeDeploy for Lambda:** Automates traffic shifting using Canary or Linear deployment preferences.

### Lambda Deployment Lifecycle Hooks

These hooks are **Lambda functions** you define in your `appspec.yml` to validate the deployment.

| Hook | Description | Use Case Example |
| :--- | :--- | :--- |
| `BeforeAllowTraffic` | Runs *before* any traffic is shifted to the new Lambda version. | Run pre-deployment checks, warm up the function, or check dependencies. If this hook fails, the deployment is stopped and rolled back. |
| `AfterAllowTraffic` | Runs *after* all traffic has been shifted to the new Lambda version. | Run post-deployment integration tests or sanity checks against the live version. |

```yaml
# Example AppSpec for a Lambda Canary Deployment
version: 0.0
Resources:
  - MyLambdaFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: "MyLambdaFunction"
        Alias: "production"
        CurrentVersion: "1"
        TargetVersion: "2"
Hooks:
  - BeforeAllowTraffic: "LambdaValidatorFunction" # This Lambda runs tests
  - AfterAllowTraffic: "LambdaPostDeploymentCheck"
```

## Troubleshooting Deployment Issues

### Scenario 1: Application Fails After Deployment
*   **Problem:** An application on EC2 instances in an ASG stops working after a deployment that changed an external API URL from `http` to `https`.
*   **Troubleshooting Steps:**
    1.  **Security Group Egress:** Check the Security Group attached to the EC2 instances. The egress (outbound) rules must allow traffic on port `443` (for HTTPS) to the destination API's IP address or range. This is the most likely cause.
    2.  **VPC Flow Logs:** If the Security Group seems correct, enable VPC Flow Logs. Filter for `REJECT` records originating from the private IP addresses of your EC2 instances. This will confirm if a Security Group or NACL is blocking the traffic.

### Scenario 2: Unexplained Instance Restarts in OpsWorks
*   **Problem:** AWS OpsWorks is restarting EC2 instances, and you need to be alerted.
*   **Solution:**
    1.  **Monitoring:** Create an Amazon EventBridge (CloudWatch Events) rule to capture state changes from OpsWorks.
    2.  **Event Pattern:** The event pattern should filter for events where the `detail.initiated_by` field is `"auto-healing"`. This indicates OpsWorks is replacing an unhealthy instance.
    3.  **Action:** Configure the rule's target to be an Amazon SNS topic, which then sends an email or SMS notification.

```json
// EventBridge Rule Pattern for OpsWorks Auto-Healing
{
  "source": ["aws.opsworks"],
  "detail-type": ["OpsWorks Instance State Change"],
  "detail": {
    "status": ["stopped"],
    "initiated_by": ["auto-healing"]
  }
}
```

### Scenario 3: Validating a Lambda Deployment
*   **Problem:** You need to run API checks during a Lambda deployment and automatically roll back on failure.
*   **Solution:**
    1.  **Use CodeDeploy:** Configure a deployment group for your Lambda function.
    2.  **AppSpec Hooks:** Define a validation Lambda function in the `BeforeAllowTraffic` hook of your AppSpec file. This function will execute your API tests.
    3.  **CloudWatch Alarms:** Associate a CloudWatch alarm with the CodeDeploy deployment group. For example, an alarm that triggers on a high number of Lambda `Errors` metric. If the alarm breaches during the deployment, CodeDeploy will automatically initiate a rollback.
    4.  **Notifications:** The CloudWatch alarm can also be configured to send a notification to an SNS topic to alert the DevOps team of the failure and rollback.

## Example: Full Container Deployment Pipeline

1.  **Infrastructure as Code (IaC):** Use AWS CDK or CloudFormation to provision all necessary resources:
    *   VPC, Subnets, Security Groups.
    *   Amazon ECR repository to store the Docker image.
    *   ECS Cluster (using Fargate for serverless container orchestration).
    *   ECS Task Definition, including the IAM Task Execution Role with permissions to pull from ECR.
    *   ECS Service to run and maintain the desired number of tasks.
2.  **CI/CD Pipeline (CodePipeline):**
    *   **Source Stage:** Connects to a CodeCommit (or GitHub) repository. A push to the `main` branch triggers the pipeline.
    *   **Build Stage:** An AWS CodeBuild project is triggered.
        *   It runs tests (`npm test`).
        *   It builds the Docker image (`docker build -t ...`).
        *   It pushes the new image to the ECR repository (`docker push ...`).
        *   It creates an `imagedefinitions.json` file as an output artifact, which tells ECS which image to deploy.
    *   **Deploy Stage:** An "Amazon ECS (Blue/Green)" action type is used.
        *   It points to the CodeDeploy application, ECS cluster, and service.
        *   It uses the `imagedefinitions.json` from the build stage as input.
        *   CodeDeploy orchestrates the blue/green deployment, creating new tasks, running validation hooks, and shifting traffic.