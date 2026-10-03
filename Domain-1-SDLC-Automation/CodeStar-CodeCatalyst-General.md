# AWS CodeStar & CodeCatalyst - DOP-C02 Exam Notes

> **Status correction (2026 review):** AWS CodeStar support ended **July 31, 2024** (retired).
> Amazon CodeCatalyst closed to new customers **November 7, 2025** and is in **maintenance mode**
> (no new features). Do NOT build new projects on either. This file is kept as historical
> reference: know what these services were in case the exam mentions them, and know that the
> live **CodeStar connections** service (GitHub/GitLab/Bitbucket source connections for
> CodePipeline) is a *separate, still-supported* service despite the similar name.

## 1. Overview

**AWS CodeStar** (retired July 2024) and **Amazon CodeCatalyst** (maintenance mode since November 2025, closed to new customers) *were* unified development services providing integrated development environments and project management capabilities. The exam-relevant takeaway: both are dead ends for new architectures - use CodePipeline + CodeBuild + CodeDeploy directly.

### Key Characteristics
- **Unified development experience** - Integrated project management and development tools
- **Project templates** - Pre-configured templates for common application types
- **Team collaboration** - Built-in collaboration and project management features
- **Integrated toolchain** - Seamless integration with AWS development services
- **Visual project dashboard** - Centralized view of project status and metrics
- **Role-based access** - Team member access control and permissions
- **Modern workflows** - Support for modern development practices and methodologies

### What Problem Does It Solve?
- Simplifies project setup and configuration
- Provides standardized development workflows
- Enables team collaboration and project management
- Reduces time to market for new projects
- Ensures consistent development practices across teams
- Integrates development tools and services seamlessly

---

## 2. CodeStar (Retired July 31, 2024)

### Core Components
- **Project Templates** - Pre-configured application templates
- **Toolchain** - Integrated CI/CD pipeline
- **Team Management** - User access and role management
- **Project Dashboard** - Centralized project monitoring
- **IDE Integration** - AWS Cloud9 integration

### Project Templates
```yaml
# Example CodeStar project template structure
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::CodeStar

Parameters:
  ProjectId:
    Type: String
    Description: CodeStar project ID
  
  CodeCommitRepoName:
    Type: String
    Description: CodeCommit repository name

Resources:
  # CodeCommit Repository
  CodeCommitRepo:
    Type: AWS::CodeCommit::Repository
    Properties:
      RepositoryName: !Ref CodeCommitRepoName
      RepositoryDescription: !Sub 'Repository for ${ProjectId}'

  # CodeBuild Project
  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Sub '${ProjectId}-build'
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0
      Source:
        Type: CODEPIPELINE
        BuildSpec: |
          version: 0.2
          phases:
            install:
              runtime-versions:
                python: 3.8
            build:
              commands:
                - echo "Building the application"
                - pip install -r requirements.txt
                - python -m pytest tests/
          artifacts:
            files:
              - '**/*'

  # CodePipeline
  CodePipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: !Sub '${ProjectId}-pipeline'
      RoleArn: !GetAtt CodePipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeCommit
                Version: 1
              Configuration:
                RepositoryName: !Ref CodeCommitRepoName
                BranchName: main
              OutputArtifacts:
                - Name: SourceOutput

        - Name: Build
          Actions:
            - Name: BuildAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              Configuration:
                ProjectName: !Ref CodeBuildProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput

        - Name: Deploy
          Actions:
            - Name: DeployAction
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: 1
              Configuration:
                ActionMode: CREATE_UPDATE
                StackName: !Sub '${ProjectId}-stack'
                TemplatePath: BuildOutput::template.yaml
                Capabilities: CAPABILITY_IAM
                RoleArn: !GetAtt CloudFormationRole.Arn
              InputArtifacts:
                - Name: BuildOutput
```

### Team Management
> The `aws codestar` CLI was removed when CodeStar support ended (July 2024).
> Team management for modern projects is done with IAM directly.

---

## 3. Amazon CodeCatalyst (Maintenance Mode Since Nov 2025 - Closed to New Customers)

### Core Features
- **Spaces** - Organizational units for projects and teams
- **Projects** - Individual development projects with integrated tools
- **Workflows** - Modern CI/CD pipelines with visual editor
- **Dev Environments** - Cloud-based development environments
- **Issues** - Integrated issue tracking and project management
- **Source Repositories** - Git repositories with collaboration features

### Space and Project Structure
```yaml
# CodeCatalyst space configuration
Space:
  Name: MyOrganization
  Description: Development space for my organization
  
Projects:
  - Name: WebApplication
    Description: Customer-facing web application
    
    SourceRepositories:
      - Name: frontend
        Description: React frontend application
      - Name: backend
        Description: Node.js backend API
      - Name: infrastructure
        Description: Infrastructure as code
    
    Workflows:
      - Name: CI-CD-Pipeline
        Definition: .codecatalyst/workflows/ci-cd.yaml
    
    DevEnvironments:
      - Name: development
        InstanceType: dev.standard1.small
        Image: public.ecr.aws/amazonlinux/amazonlinux:2
    
    Issues:
      - Enabled: true
      - Labels: [bug, feature, enhancement]
      - Priorities: [low, medium, high, critical]
```

### Workflow Configuration
```yaml
# .codecatalyst/workflows/ci-cd.yaml
Name: CI-CD-Pipeline
SchemaVersion: "1.0"

Triggers:
  - Type: PUSH
    Branches:
      - main
      - develop
  - Type: PULLREQUEST
    Branches:
      - main

Actions:
  Build:
    Identifier: aws/build@v1
    Inputs:
      Sources:
        - WorkflowSource
    Configuration:
      Steps:
        - Run: echo "Installing dependencies"
        - Run: npm install
        - Run: echo "Running tests"
        - Run: npm test
        - Run: echo "Building application"
        - Run: npm run build
    Outputs:
      Artifacts:
        - Name: BuildArtifacts
          Files:
            - "dist/**"
            - "package.json"

  SecurityScan:
    Identifier: aws/managed-test@v1
    DependsOn:
      - Build
    Inputs:
      Sources:
        - WorkflowSource
    Configuration:
      Steps:
        - Run: echo "Running security scan"
        - Run: npm audit
        - Run: echo "Running SAST scan"
        - Run: npm run security-scan

  Deploy-Dev:
    Identifier: aws/build@v1
    DependsOn:
      - Build
      - SecurityScan
    Environment:
      Name: development
      Connections:
        - Name: aws-connection
          Role: CodeCatalystWorkflowDevelopmentRole-MySpace
    Inputs:
      Artifacts:
        - BuildArtifacts
    Configuration:
      Steps:
        - Run: echo "Deploying to development environment"
        - Run: aws s3 sync dist/ s3://my-dev-bucket/
        - Run: aws cloudfront create-invalidation --distribution-id $DEV_DISTRIBUTION_ID --paths "/*"

  Deploy-Prod:
    Identifier: aws/build@v1
    DependsOn:
      - Deploy-Dev
    Environment:
      Name: production
      Connections:
        - Name: aws-connection
          Role: CodeCatalystWorkflowProductionRole-MySpace
    Inputs:
      Artifacts:
        - BuildArtifacts
    Configuration:
      Steps:
        - Run: echo "Deploying to production environment"
        - Run: aws s3 sync dist/ s3://my-prod-bucket/
        - Run: aws cloudfront create-invalidation --distribution-id $PROD_DISTRIBUTION_ID --paths "/*"
    Compute:
      Type: Lambda
```

### Dev Environments
```yaml
# Dev environment configuration
DevEnvironment:
  Name: full-stack-dev
  InstanceType: dev.standard1.medium
  Image: public.ecr.aws/amazonlinux/amazonlinux:2
  
  PersistentStorage:
    SizeInGiB: 16
  
  Ides:
    - Name: VSCode
      Runtime: public.ecr.aws/aws-cloud9/ide:latest
  
  Repositories:
    - Name: frontend
      BranchName: main
    - Name: backend
      BranchName: main
  
  EnvironmentVariables:
    - Name: NODE_ENV
      Value: development
    - Name: API_URL
      Value: http://localhost:3001
  
  Timeout:
    InactivityTimeoutMinutes: 30
    
  PostCreationScript: |
    #!/bin/bash
    # Install Node.js
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
    source ~/.bashrc
    nvm install 18
    nvm use 18
    
    # Install dependencies
    cd /projects/frontend && npm install
    cd /projects/backend && npm install
    
    # Start development servers
    cd /projects/backend && npm run dev &
    cd /projects/frontend && npm start
```

---

## 4. Migration from CodeStar to CodeCatalyst (Historical - Path Is Obsolete)

> This section is kept for reference only. The CodeStar-to-CodeCatalyst migration path made
> sense when CodeCatalyst was the strategic service; with CodeCatalyst in maintenance mode
> since November 2025, the correct modern target is **CodePipeline + CodeBuild + CodeDeploy**
> (or GitHub Actions), not CodeCatalyst.

### Migration Strategy
```python
import boto3
import json

def migrate_codestar_to_codecatalyst():
    """
    Migration strategy from CodeStar to CodeCatalyst
    """
    
    # Step 1: Inventory existing CodeStar projects
    codestar_projects = inventory_codestar_projects()
    
    # Step 2: Create CodeCatalyst space
    codecatalyst_space = create_codecatalyst_space()
    
    # Step 3: Migrate each project
    for project in codestar_projects:
        migrate_project(project, codecatalyst_space)
    
    # Step 4: Update team access and permissions
    migrate_team_permissions(codestar_projects, codecatalyst_space)
    
    # Step 5: Validate migration
    validate_migration(codecatalyst_space)

def inventory_codestar_projects():
    """
    Inventory existing CodeStar projects
    """
    
    codestar = boto3.client('codestar')
    
    projects = []
    paginator = codestar.get_paginator('list_projects')
    
    for page in paginator.paginate():
        for project in page['projects']:
            project_details = codestar.describe_project(id=project['projectId'])
            
            # Get project resources
            resources = codestar.list_resources(projectId=project['projectId'])
            
            # Get team members
            team_members = codestar.list_team_members(projectId=project['projectId'])
            
            projects.append({
                'id': project['projectId'],
                'name': project['projectId'],
                'description': project_details['description'],
                'resources': resources['resources'],
                'team_members': team_members['teamMembers'],
                'created_time': project['createdTimeStamp']
            })
    
    return projects

def create_codecatalyst_space():
    """
    Create CodeCatalyst space for migrated projects
    """
    
    # CodeCatalyst space creation (via API or console)
    space_config = {
        'name': 'MigratedFromCodeStar',
        'description': 'Projects migrated from AWS CodeStar',
        'region': 'us-west-2'
    }
    
    return space_config

def migrate_project(codestar_project, codecatalyst_space):
    """
    Migrate individual CodeStar project to CodeCatalyst
    """
    
    migration_plan = {
        'source_repositories': extract_repositories(codestar_project),
        'ci_cd_pipeline': extract_pipeline_config(codestar_project),
        'deployment_targets': extract_deployment_config(codestar_project),
        'team_members': codestar_project['team_members']
    }
    
    # Create CodeCatalyst project
    codecatalyst_project = {
        'name': codestar_project['name'],
        'description': codestar_project['description'],
        'space': codecatalyst_space['name']
    }
    
    # Migrate source repositories
    migrate_repositories(migration_plan['source_repositories'], codecatalyst_project)
    
    # Convert CodePipeline to CodeCatalyst workflow
    migrate_pipeline(migration_plan['ci_cd_pipeline'], codecatalyst_project)
    
    # Set up environments and connections
    setup_environments(migration_plan['deployment_targets'], codecatalyst_project)
    
    return codecatalyst_project
```

### CodePipeline to Workflow Conversion
```python
def convert_pipeline_to_workflow(codepipeline_config):
    """
    Convert CodePipeline configuration to CodeCatalyst workflow
    """
    
    workflow = {
        'Name': f"{codepipeline_config['Name']}-workflow",
        'SchemaVersion': '1.0',
        'Triggers': [],
        'Actions': {}
    }
    
    # Convert triggers
    workflow['Triggers'] = [
        {
            'Type': 'PUSH',
            'Branches': ['main']
        }
    ]
    
    # Convert stages to actions
    for stage in codepipeline_config['Stages']:
        stage_name = stage['Name']
        
        if stage_name == 'Source':
            # Source stage becomes trigger
            continue
        elif stage_name == 'Build':
            workflow['Actions']['Build'] = convert_build_stage(stage)
        elif stage_name == 'Test':
            workflow['Actions']['Test'] = convert_test_stage(stage)
        elif stage_name == 'Deploy':
            workflow['Actions']['Deploy'] = convert_deploy_stage(stage)
    
    return workflow

def convert_build_stage(stage):
    """
    Convert CodeBuild stage to CodeCatalyst build action
    """
    
    build_action = stage['Actions'][0]  # Assume single build action
    
    return {
        'Identifier': 'aws/build@v1',
        'Inputs': {
            'Sources': ['WorkflowSource']
        },
        'Configuration': {
            'Steps': [
                'Run: echo "Building application"',
                'Run: npm install',
                'Run: npm run build'
            ]
        },
        'Outputs': {
            'Artifacts': [
                {
                    'Name': 'BuildArtifacts',
                    'Files': ['dist/**', 'package.json']
                }
            ]
        }
    }

def convert_deploy_stage(stage):
    """
    Convert CodeDeploy/CloudFormation stage to CodeCatalyst deploy action
    """
    
    deploy_action = stage['Actions'][0]
    provider = deploy_action['ActionTypeId']['Provider']
    
    if provider == 'CloudFormation':
        return {
            'Identifier': 'aws/build@v1',
            'DependsOn': ['Build'],
            'Environment': {
                'Name': 'production',
                'Connections': [
                    {
                        'Name': 'aws-connection',
                        'Role': 'CodeCatalystWorkflowRole'
                    }
                ]
            },
            'Configuration': {
                'Steps': [
                    'Run: echo "Deploying with CloudFormation"',
                    f"Run: aws cloudformation deploy --template-file {deploy_action['Configuration']['TemplatePath']} --stack-name {deploy_action['Configuration']['StackName']}"
                ]
            }
        }
    
    return {}
```

---

## 5. Integration Patterns

### CodeCatalyst with AWS Services
```yaml
# Integration with AWS services through connections
Connections:
  - Name: aws-production
    Type: AWS
    Role: arn:aws:iam::123456789012:role/CodeCatalystProductionRole
    
  - Name: aws-development  
    Type: AWS
    Role: arn:aws:iam::123456789012:role/CodeCatalystDevelopmentRole

Environments:
  development:
    Connections:
      - aws-development
    
  production:
    Connections:
      - aws-production
    ApprovalRequired: true
    Approvers:
      - team-lead
      - devops-engineer

# Workflow using connections
Actions:
  DeployInfrastructure:
    Identifier: aws/build@v1
    Environment:
      Name: production
      Connections:
        - Name: aws-production
    Configuration:
      Steps:
        - Run: |
            aws cloudformation deploy \
              --template-file infrastructure/template.yaml \
              --stack-name myapp-infrastructure \
              --capabilities CAPABILITY_IAM \
              --parameter-overrides Environment=production
```

### Third-Party Integrations
```yaml
# Integration with external services
Actions:
  NotifySlack:
    Identifier: aws/build@v1
    DependsOn:
      - Deploy
    Configuration:
      Steps:
        - Run: |
            curl -X POST -H 'Content-type: application/json' \
              --data '{"text":"Deployment completed successfully!"}' \
              $SLACK_WEBHOOK_URL
    
  UpdateJira:
    Identifier: aws/build@v1
    DependsOn:
      - Deploy
    Configuration:
      Steps:
        - Run: |
            curl -X POST \
              -H "Authorization: Bearer $JIRA_TOKEN" \
              -H "Content-Type: application/json" \
              --data '{"transition":{"id":"31"}}' \
              "https://mycompany.atlassian.net/rest/api/3/issue/$JIRA_ISSUE_KEY/transitions"
```

---

## 6. Common Exam Scenarios

### Scenario 1: Migrate legacy CodeStar project to CodeCatalyst (obsolete path - know it existed, build on CodePipeline instead)
**Solution:**
- Inventory existing CodeStar projects and resources
- Create CodeCatalyst space and projects
- Migrate source repositories and preserve history
- Convert CodePipeline to CodeCatalyst workflows
- Update team permissions and access controls

### Scenario 2: Set up modern development workflow with CodeCatalyst
**Solution:**
- Create CodeCatalyst project with integrated repositories
- Configure CI/CD workflows with quality gates
- Set up dev environments for team collaboration
- Implement issue tracking and project management
- Configure automated testing and deployment

### Scenario 3: Multi-environment deployment with approvals
**Solution:**
- Configure multiple environments (dev, staging, prod)
- Set up environment-specific AWS connections
- Implement approval workflows for production deployments
- Configure environment-specific variables and secrets
- Set up monitoring and rollback capabilities

### Scenario 4: Team collaboration and access management
**Solution:**
- Configure space and project permissions
- Set up role-based access controls
- Implement code review workflows
- Configure issue tracking and assignment
- Set up team notifications and communication

### Scenario 5: Integration with existing AWS infrastructure
**Solution:**
- Configure AWS connections with appropriate IAM roles
- Set up VPC and security group configurations
- Integrate with existing CI/CD tools and processes
- Configure monitoring and logging integration
- Implement cost optimization strategies

### Scenario 6: Standardize development practices across teams
**Solution:**
- Create reusable project templates
- Implement standardized workflow patterns
- Set up code quality and security scanning
- Configure consistent deployment processes
- Implement governance and compliance controls

---

## 7. CLI Commands Reference

### CodeStar Operations (Retired)
> The entire `aws codestar` CLI namespace was removed when CodeStar support ended on
> July 31, 2024. These commands no longer exist. (Kept as a reminder: if an exam
> question shows `aws codestar ...` as an answer choice, it is a distractor.)

### CodeCatalyst Operations
```bash
# Note: CodeCatalyst primarily uses web console and API
# CLI operations are limited compared to other AWS services

# List spaces (via AWS CLI v2 with CodeCatalyst extension)
aws codecatalyst list-spaces

# Create project (typically done via console or API)
aws codecatalyst create-project \
  --space-name MySpace \
  --display-name "My Project" \
  --description "Project description"
```

---

## 8. Best Practices for DOP-C02 Exam

### Project Organization
- Use meaningful project and space names
- Implement consistent naming conventions
- Organize projects by team or application domain
- Use templates for standardized project setup

### Workflow Design
- Design workflows for different environments
- Implement proper testing and quality gates
- Use environment-specific configurations
- Implement approval processes for production

### Team Collaboration
- Set up appropriate access controls and permissions
- Use issue tracking for project management
- Implement code review processes
- Configure team notifications and communication

### Migration Strategy
- Plan migration from CodeStar to CodeCatalyst
- Preserve project history and configurations
- Update team training and documentation
- Implement gradual migration approach

---

## 9. Exam Tips

### What to Remember
- **CodeStar is legacy** - Being replaced by CodeCatalyst
- **CodeCatalyst provides modern workflows** - Visual editor, better collaboration
- **Spaces organize projects** - Organizational units in CodeCatalyst
- **Dev environments** - Cloud-based development environments
- **Workflows replace pipelines** - Modern CI/CD with visual editor
- **Integration with AWS** - Through connections and IAM roles
- **Issue tracking** - Built-in project management capabilities

### Common Traps
- Confusing CodeStar (legacy) with CodeCatalyst (modern)
- Not understanding migration path from CodeStar to CodeCatalyst
- Missing workflow configuration syntax differences
- Not configuring proper AWS connections and permissions
- Overlooking team collaboration and access management features

### Scenario-Based Questions
- Focus on migration strategies and modernization
- Understand workflow design and configuration
- Know team collaboration and project management features
- Understand integration with existing AWS infrastructure
- Know cost optimization and governance strategies

---

## 10. Quick Reference Cheat Sheet

### Service Comparison
```
CodeStar (Legacy):
- Project templates
- Basic CI/CD pipelines
- Team management
- AWS Cloud9 integration

CodeCatalyst (Modern):
- Spaces and projects
- Visual workflow editor
- Dev environments
- Issue tracking
- Enhanced collaboration
```

### Key Components
```
CodeCatalyst:
- Spaces: Organizational units
- Projects: Development projects
- Workflows: CI/CD pipelines
- Dev Environments: Cloud IDEs
- Issues: Project management
- Source Repositories: Git repos
```

### Migration Path
```
CodeStar → CodeCatalyst:
1. Inventory existing projects
2. Create CodeCatalyst space
3. Migrate repositories
4. Convert pipelines to workflows
5. Update team permissions
6. Validate migration
```

---

## 11. Summary

AWS CodeStar and CodeCatalyst represent the evolution of unified development experiences on AWS and may appear in the DOP-C02 exam. Key areas to understand:

1. **Service evolution** (CodeStar legacy to CodeCatalyst modern)
2. **Project organization** (spaces, projects, templates)
3. **Workflow configuration** (modern CI/CD with visual editor)
4. **Team collaboration** (access control, issue tracking)
5. **Migration strategies** (CodeStar to CodeCatalyst transition)
6. **AWS integration** (connections, IAM roles, environments)
7. **Development environments** (cloud-based IDEs and collaboration)
8. **Best practices** (standardization, governance, cost optimization)

Understanding these concepts will help with questions about modern development practices and AWS development service evolution in the DOP-C02 exam.