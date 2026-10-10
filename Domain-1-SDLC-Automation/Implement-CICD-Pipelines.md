# Task 1: Implement CI/CD Pipelines

This section covers the fundamentals of the Software Development Lifecycle (SDLC), CI/CD concepts, and how to implement CI/CD pipelines using AWS services, focusing on the knowledge required for the DOP-C02 exam.

### Software Development Lifecycle (SDLC)

#### SDLC Phases
*   **Requirement Analysis**: Define requirements, goals, and timelines in a Software Requirement Specification (SRS) document.
*   **Design Plan**: Create the architecture, feature list, infrastructure requirements, UI, and security plan.
*   **Development**: Write the source code.
*   **Testing**: Integrate automated testing into the CI/CD pipeline.
*   **Deployment**: Automatically deploy the tested version to production. This requires knowledge of deployment patterns (single/multi-account) and strategies.
*   **Maintenance**: Remediate bugs, set up automation, monitor, upgrade, and add new features.

#### SDLC Methodologies
Different methodologies prioritize different aspects of development. Common models include:
*   Linear Sequential Model (Waterfall)
*   Verification and Validation (V-Model)
*   Prototype Model
*   Agile Model
*   Big Bang Model

### CI/CD Concepts

**CI/CD is an enhancement to the SDLC, not a replacement.** It is a set of practices that uses automation to speed up software delivery.

A CI/CD pipeline consists of stages (e.g., Source, Build, Test, Staging, Production). Each stage acts as a quality gate. If a stage fails, the pipeline stops, providing immediate feedback and preventing bugs from reaching production (**fail fast**).

*   **Continuous Integration (CI)**: The practice of frequently merging code changes into a central repository, followed by automated builds and tests.
*   **Continuous Delivery (CD)**: Extends CI by automatically deploying all code changes to a testing and/or production-like environment after the build stage. May include a manual approval step before final production deployment.
*   **Continuous Deployment**: Extends Continuous Delivery by automatically deploying every change that passes all stages of the pipeline to production without human intervention.

A key best practice is **automation** using scripts, templates, and tools. **Infrastructure as Code (IaC)** is crucial for rapidly and reliably setting up test and production environments.

### AWS CI/CD Services (The Developer Tools Suite)

AWS provides a complete set of tools to build and manage your CI/CD pipelines.

*   **AWS CodePipeline**: A continuous delivery service that models, visualizes, and automates the steps required to release software. It orchestrates the entire process.
*   **AWS CodeCommit**: A managed source control service that hosts secure Git-based repositories.
*   **AWS CodeBuild**: A fully managed continuous integration service that compiles source code, runs tests, and produces software packages that are ready to deploy.
*   **AWS CodeDeploy**: A service that automates code deployments to any instance, including Amazon EC2 instances and on-premises servers. It handles the complexity of updating your applications.
*   **AWS CodeArtifact**: A secure, scalable, and cost-effective artifact management for software development.
*   **Amazon ECR**: A managed container image registry service.
*   **AWS CodeStar**: (Retired July 2024 - do not use for new projects) A unified UI that used to set up the entire CI/CD toolchain.
*   **Infrastructure as Code (IaC)**:
    *   **AWS CloudFormation**: Declarative IaC for provisioning AWS resources.
    *   **AWS SAM (Serverless Application Model)**: An open-source framework for building serverless applications.
    *   **AWS CDK (Cloud Development Kit)**: Imperative IaC using familiar programming languages.

### Exam-Heavy Scenarios & Integrations

#### Scenario 1: Securing Credentials in ECS

*   **Problem**: An application on Amazon ECS needs database credentials. The credentials must be secure, managed with a lifecycle, and support key rotation. They should be passed as environment variables.
*   **Solution**:
    1.  Store the database credentials in **AWS Secrets Manager**.
    2.  Encrypt the secret using a customer-managed **AWS KMS** key for full control.
    3.  Create an IAM Role for the ECS Task (**Task Execution Role**) with permissions to access both the specific secret in Secrets Manager (`secretsmanager:GetSecretValue`) and the KMS key (`kms:Decrypt`).
    4.  In the ECS Task Definition, reference the secret. Do not hardcode the value.

```json
// Example snippet from an ECS Task Definition
"secrets": [
    {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-db-secret-AbCdEf"
    }
]
```

#### Scenario 2: Automating AMI Generation in a Pipeline

*   **Problem**: Multiple applications use their own AMIs. You need a process to automatically create and deploy new AMIs as part of a CodePipeline workflow. The new AMI ID must be accessible to other pipelines.
*   **Solution**:
    1.  Use **AWS Systems Manager (SSM) Automation** to define the steps for creating a new AMI (e.g., start an instance, install software, create image). The `AWS-UpdateLinuxAmi` or `AWS-UpdateWindowsAmi` documents are good starting points.
    2.  In **AWS CodePipeline**, add a stage with a custom action that invokes the SSM Automation document. This can be done via a Lambda function.
    3.  The Lambda function, triggered by CodePipeline, starts the SSM Automation execution.
    4.  Configure the SSM Automation document to write the newly created AMI ID into **AWS Systems Manager Parameter Store** upon successful completion.
    5.  Subsequent stages or other pipelines can then read the AMI ID from Parameter Store to use in their deployment steps (e.g., updating an Auto Scaling Group's Launch Template).

#### Scenario 3: CI/CD for Network Function Virtualization (NFV)

*   **Problem**: Design a CI/CD process for an NFV framework using AWS services.
*   **Solution**:
    1.  **Infrastructure**: Use **AWS CDK** or **CloudFormation** to define and deploy network prerequisites, infrastructure, and the network functions themselves.
    2.  **Pipeline**: Use **AWS CodePipeline** to orchestrate the workflow.
    3.  **Source**: Store CDK/CloudFormation templates and application code in **AWS CodeCommit** or GitHub.
    4.  **Build**: Use **AWS CodeBuild** to compile code, build container images, and synthesize IaC templates.
    5.  **Test**: Deploy the infrastructure and application to a dedicated test environment. Run functional, integration, performance, and reliability tests.
    6.  **Approval**: Add a manual approval stage in CodePipeline before deploying to production.
    7.  **Deploy**: Upon approval, use CodePipeline to execute the CDK/CloudFormation deployment to the production environment.
