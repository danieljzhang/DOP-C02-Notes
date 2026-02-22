# Building and Managing Artifacts

For the DOP-C02 exam, understanding how to build, manage, secure, and version artifacts is crucial. Artifacts are the outputs of your build process, such as compiled code, container images, or packaged serverless functions.

## Artifact Repositories

Choosing the right repository for your artifact type is a common exam topic.

| Service | Artifact Type | Use Case |
| :--- | :--- | :--- |
| **AWS CodeArtifact** | Language-specific packages (npm, Maven, pip, NuGet) | Centralized repository for application dependencies. Can proxy public repositories like npmjs and Maven Central. |
| **Amazon ECR** | Docker container images | Securely store, manage, and deploy container images for use with ECS, EKS, and Fargate. |
| **Amazon S3** | General-purpose binaries, ZIP files, build outputs | The default artifact store for CodePipeline. Used for storing `.zip` files for Lambda, CodeDeploy application revisions, and other build outputs. |

## Automating Artifact Generation

### EC2 Image Builder for AMIs and Container Images

EC2 Image Builder is a fully managed service for automating the creation, management, and deployment of customized, secure, and up-to-date server images (AMIs and container images). This is a key service for implementing **immutable infrastructure**.

*   **Image Pipeline:** The automation framework for building your images.
*   **Image Recipe:** Defines the base image and the components to be applied.
*   **Components:** Scripts that define the steps to customize an instance or test it before image creation.
*   **Cross-Account Sharing:** Integrates with **AWS Resource Access Manager (RAM)** to share components, recipes, and images with other AWS accounts or within an AWS Organization.

### Generating Lambda Layers with CodeBuild

A common pattern is to use a CI/CD pipeline to build and publish Lambda Layers, which can be shared across multiple functions.

1.  **Source:** A CodeCommit repository contains the layer's dependencies (e.g., `requirements.txt` for Python).
2.  **Trigger:** A commit to the repository triggers CodePipeline.
3.  **Build:** A CodeBuild project runs, using a `buildspec.yml` to install dependencies into the correct directory structure and create a `.zip` archive.
4.  **Artifact:** CodePipeline stores the output `.zip` file in an S3 artifact bucket.
5.  **Publish:** A Lambda function is invoked by CodePipeline. This function downloads the artifact from S3 and uses the `lambda:PublishLayerVersion` API call to create a new layer version.

## Using Artifacts in Deployments

### Reusable CloudFormation Templates in CodePipeline

To avoid manually editing templates for different environments (dev, staging, prod), you can make them reusable.

*   **Problem:** You have a single CloudFormation template but need to deploy it with different parameters (e.g., instance types, VPC IDs) for each environment.
*   **Solution:** In your CodePipeline definition, use a single template file but specify **parameter overrides** for the CloudFormation action in each stage.

```json
// Snippet from a CodePipeline stage definition
{
    "name": "Deploy-Staging",
    "actions": [
        {
            "name": "DeployCFN",
            "actionTypeId": { ... },
            "configuration": {
                "StackName": "MyWebApp-Staging",
                "TemplatePath": "BuildOutput::template.yaml",
                "ParameterOverrides": "{\"InstanceType\":\"t3.medium\", \"Environment\":\"staging\"}"
            },
            ...
        }
    ]
}
```

### Validating ECS Deployments with Lambda Hooks

This is a critical pattern for safe deployments with CodeDeploy.

*   **Scenario:** You are deploying a new version of a Docker container to Amazon ECS. Before shifting all production traffic, you need to run integration tests against the new version. If tests fail, the deployment must be rolled back automatically.
*   **Implementation:**
    1.  **CodeDeploy Setup:** Configure a blue/green deployment for your ECS service. This involves an Application Load Balancer with two target groups (blue and green) and two listeners (a production listener and a test listener).
    2.  **AppSpec File:** In your `appspec.yml`, specify a Lambda function in the `AfterAllowTestTraffic` lifecycle hook.
    3.  **Deployment Flow:**
        *   CodeDeploy provisions the new "green" tasks and registers them with the green target group.
        *   Traffic from the ALB's **test listener port** is routed to the green target group.
        *   The `AfterAllowTestTraffic` hook is triggered, invoking your validation Lambda function.
        *   The Lambda function runs API tests against the test listener endpoint.
        *   The Lambda function calls back to CodeDeploy with a `Success` or `Failure` status.
        *   If `Success`, CodeDeploy reroutes production traffic to the green target group and the deployment completes.
        *   If `Failure`, CodeDeploy initiates an automatic rollback, terminating the green tasks and leaving the blue environment untouched.

## Securing Artifacts and Repositories

### Securing CodeCommit Branches

You can use IAM policies to enforce branch-level permissions.

*   **Scenario:** Allow all developers to push to `dev` and `staging` branches, but only allow team leads to merge or push to the `main` branch.
*   **Solution:**
    1.  Create an IAM group for all developers with the `AWSCodeCommitPowerUser` managed policy.
    2.  Attach an additional **inline policy** to this group that explicitly denies actions on the `main` branch.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "codecommit:GitPush",
                "codecommit:DeleteBranch",
                "codecommit:PutFile",
                "codecommit:MergePullRequestByFastForward"
            ],
            "Resource": "arn:aws:codecommit:us-east-1:111122223333:MyRepo",
            "Condition": {
                "StringEquals": {
                    "codecommit:References": [
                        "refs/heads/main"
                    ]
                }
            }
        }
    ]
}
```

### Securing Artifacts in Multi-Region Pipelines

When using cross-region actions in CodePipeline, artifact security and data residency are key concerns.

*   **How it Works:** When you add an action (e.g., a CodeBuild project) to a pipeline stage in a different region, CodePipeline automatically creates a default S3 artifact bucket in that target region.
*   **Encryption (Exam Pitfall):** To encrypt artifacts in the cross-region bucket using a customer-managed KMS key, the **KMS key must be created in the same region as the cross-region action and the artifact bucket**. You cannot use a KMS key from the pipeline's primary region to encrypt an artifact bucket in a secondary region.
*   **Data Residency:** This setup ensures that artifacts generated in a specific region are stored at rest within that same region, helping to meet data residency requirements.