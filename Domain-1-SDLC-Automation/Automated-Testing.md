# Automated Testing in the SDLC

Automated testing is a cornerstone of modern DevOps practices and a critical topic for the DOP-C02 exam. Each stage of a CI/CD pipeline acts as a quality gate. If a problem is found, the code is prevented from progressing, ensuring that only high-quality, verified code reaches production.

For the exam, you must know the different types of tests, where they fit into the CI/CD pipeline, and how to implement automated testing using various AWS services.

## The Role of Testing in CI/CD

*   **Continuous Integration (CI):** The practice of frequently merging code changes into a central repository, after which automated builds and tests are run. This stage focuses on compiling code, running unit tests, and performing static code analysis.
*   **Continuous Delivery (CD):** An extension of CI where code changes are automatically built, tested, and prepared for a release to production. The output is a deployable artifact.
*   **Continuous Deployment (CD):** The final stage where every change that passes all automated tests is automatically deployed to production.

## Types of Software Tests (DOP-C02 Context)

| Test Type | Purpose & Position in Pipeline | AWS Services |
| :--- | :--- | :--- |
| **Unit Testing** | Validates the smallest individual units of code (e.g., functions, methods) in isolation. Runs early in the CI phase, immediately after code checkout and compilation. | `CodeBuild` |
| **Static Code Analysis** | Analyzes source code for potential vulnerabilities, bugs, and adherence to coding standards without executing it. Runs early in the CI phase. | `CodeGuru Reviewer`, `SonarQube` (via CodeBuild) |
| **Integration Testing** | Verifies interfaces and interactions between different software components or services. Runs after unit tests pass. | `CodeBuild`, `Lambda` (for serverless) |
| **Component Testing** | Tests message passing and outcomes between various system components, often in isolation from the rest of the system. | `CodeBuild` |
| **System Testing** | Tests the fully integrated system end-to-end to verify that it meets specified requirements. | `CodeBuild`, `CodeDeploy` (to a test environment) |
| **Performance Testing** | Determines the responsiveness, stability, and scalability of a system under a specific workload. Includes Load, Stress, and Spike testing. | `Distributed Load Testing on AWS` solution, 3rd party tools on EC2/Fargate |
| **Compliance Testing** | Checks if the code or infrastructure change complies with regulatory requirements, specifications, or defined corporate standards. | `AWS Config`, `CodeBuild` |
| **User Acceptance (UAT)** | Validates the end-to-end business flow from the user's perspective. Often involves a manual approval step after deployment to a pre-production environment. | `CodePipeline` (Manual Approval Action) |

## Implementing Automated Testing on AWS

### Testing Pull Requests with CodeCommit and CodeBuild

You can enforce code quality by automatically testing changes in a pull request (PR) *before* it is merged.

1.  **Trigger:** A developer creates a PR in an AWS CodeCommit repository.
2.  **Automation:** An Amazon EventBridge (CloudWatch Events) rule detects the `pullRequestStateChanged` event.
3.  **Action:** The rule triggers an AWS Lambda function or directly starts an AWS CodeBuild project.
4.  **Test:** CodeBuild checks out the proposed code, runs unit tests, integration tests, and static analysis.
5.  **Feedback:** CodeBuild posts the build status (success/failure) back to the CodeCommit PR. You can configure repository settings to block merging if the build fails.

### Example `buildspec.yml` for Testing

This file, placed in the root of your repository, tells CodeBuild which commands to run.

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "Installing dependencies..."
      - npm install
  pre_build:
    commands:
      - echo "Running static code analysis..."
      # Example with a linter
      - npm run lint
  build:
    commands:
      - echo "Running unit and integration tests..."
      - npm test
reports:
  TestReports: # A report group in CodeBuild
    files:
      - 'reports/test-results.xml'
    file-format: 'JUNITXML'
```

### Building a Test and Deployment Pipeline

A common exam scenario involves a multi-stage pipeline that includes testing and manual gates.

1.  **Source:** `CodePipeline` pulls the latest commit from `CodeCommit` (or another source repository) after a PR is merged.
2.  **Build:** A `CodeBuild` action compiles the code, runs tests, and creates a deployable artifact (e.g., a Docker image, a .zip file).
3.  **Test Stage:**
    *   A `CodeDeploy` action deploys the application to a dedicated "Test" or "Staging" environment.
    *   A `CodeBuild` action runs integration or system tests against the newly deployed environment.
4.  **Manual Approval:** A `CodePipeline` "Manual Approval" action pauses the pipeline. An operator or QA engineer performs UAT on the staging environment and then approves or rejects the change.
5.  **Production Deploy:** After approval, another `CodeDeploy` action deploys the verified code to the production environment, often using a safe deployment strategy like blue/green or a canary release.

## Observability and Application Health

Testing confirms behavior at a point in time; observability provides continuous insight into application health in production.

### AWS X-Ray for Distributed Tracing

When you have performance issues (e.g., high latency) in a microservices or serverless architecture, X-Ray is the key service. It helps you trace user requests as they travel through your application.

*   **Core Components:** The X-Ray SDK (instrumented in your code) sends trace data to the X-Ray Daemon. The Daemon batches this data and sends it to the X-Ray service.
*   **Exam Pitfall (ECS Configuration):** To run the X-Ray daemon alongside your application in Amazon ECS, you must configure your task definition correctly. This is a very common exam topic.
    *   Deploy the daemon as a **sidecar container** in the same task definition as your application.
    *   The application container sends trace data to the daemon over the local container network.
    *   The X-Ray daemon listens for trace data on **UDP port 2000**.
    *   You must link the containers and expose this port from the daemon container to the application container.

### Amazon CloudWatch

*   **CloudWatch Synthetics:** Creates "canaries"—configurable scripts that run on a schedule to monitor your API endpoints and UI workflows. They act as automated, synthetic users to proactively detect issues.
*   **CloudWatch ServiceLens:** Provides an interactive service map that integrates traces from X-Ray, metrics from CloudWatch, and logs from CloudWatch Logs. It gives you a unified view to correlate performance issues.
*   **CloudWatch Metrics:** Know which metrics to monitor for key services. For **Amazon Route 53 Health Checks**, monitor the `HealthCheckStatus` metric. A value of `0` indicates failure, and `1` indicates success.

### AWS Systems Manager Run Command

You can use Run Command to execute scripts on your instances. For monitoring and automation, it's important to handle script outcomes.

*   **Exit Codes:** You can specify how Run Command should handle exit codes from your script. By default, an exit code other than `0` causes the command invocation to fail. You can customize this behavior.
*   **CloudWatch Integration:** Systems Manager publishes metrics on the status of Run Command executions to CloudWatch, which you can use to trigger alarms or other automations.