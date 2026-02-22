# AWS Systems Manager - DOP-C02 Exam Notes

## 1. Overview

**AWS Systems Manager** is a unified interface for managing AWS resources and on-premises systems at scale. It provides operational insights and enables automation of common administrative and operational tasks.

### Key Characteristics
- **Unified management** - Single interface for multiple AWS and on-premises resources
- **Automation** - Runbooks and automated remediation
- **Configuration management** - Parameter Store and configuration compliance
- **Patch management** - Automated patching across instances
- **Session management** - Secure shell access without SSH keys
- **Operational insights** - Centralized operational data and dashboards
- **Compliance** - Configuration compliance and drift detection

### What Problem Does It Solve?
- Centralizes operational management across hybrid environments
- Automates routine administrative tasks and remediation
- Provides secure access to instances without managing SSH keys
- Enables configuration management and compliance at scale
- Facilitates patch management and security updates
- Supports operational insights and troubleshooting

---

## 2. Core Components

### Parameter Store
- Secure storage for configuration data and secrets
- Hierarchical parameter organization
- Integration with KMS for encryption
- Version control and change tracking
- Cross-service integration

### Session Manager
- Browser-based shell access to instances
- No SSH keys or bastion hosts required
- Session logging and auditing
- Port forwarding capabilities
- Cross-platform support (Linux, Windows)

### Patch Manager
- Automated patch deployment
- Patch baselines and approval rules
- Maintenance windows for scheduling
- Compliance reporting
- Support for multiple operating systems

### Automation
- Runbook execution and workflow automation
- Document-based automation (JSON/YAML)
- Integration with other AWS services
- Error handling and rollback capabilities
- Cross-account automation

### OpsCenter
- Centralized operational issues management
- Integration with CloudWatch and Config
- Automated remediation workflows
- Operational insights and analytics
- Issue tracking and resolution

---

## 3. Parameter Store

### Parameter Types
```bash
# String parameter
aws ssm put-parameter \
  --name "/myapp/database/host" \
  --value "db.example.com" \
  --type "String" \
  --description "Database hostname"

# SecureString parameter (encrypted)
aws ssm put-parameter \
  --name "/myapp/database/password" \
  --value "supersecret" \
  --type "SecureString" \
  --key-id "alias/parameter-store-key" \
  --description "Database password"

# StringList parameter
aws ssm put-parameter \
  --name "/myapp/allowed-ips" \
  --value "10.0.0.1,10.0.0.2,10.0.0.3" \
  --type "StringList" \
  --description "Allowed IP addresses"
```

### Parameter Hierarchies
```bash
# Hierarchical parameter structure
/myapp/
├── database/
│   ├── host
│   ├── port
│   ├── username
│   └── password
├── api/
│   ├── endpoint
│   └── key
└── features/
    ├── feature-flag-1
    └── feature-flag-2

# Get parameters by path
aws ssm get-parameters-by-path \
  --path "/myapp/database" \
  --recursive \
  --with-decryption
```

### Parameter Store in CloudFormation
```yaml
Resources:
  DatabaseHost:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /myapp/database/host
      Type: String
      Value: !GetAtt DatabaseInstance.Endpoint.Address
      Description: Database endpoint

  DatabasePassword:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /myapp/database/password
      Type: SecureString
      Value: !Ref DatabaseMasterPassword
      Description: Database master password
      KeyId: alias/parameter-store-key

  # Reference parameter in other resources
  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          DB_HOST: '{{resolve:ssm:/myapp/database/host}}'
          DB_PASSWORD: '{{resolve:ssm-secure:/myapp/database/password}}'
```

---

## 4. Session Manager

### Session Manager Configuration
```yaml
Resources:
  SessionManagerRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Policies:
        - PolicyName: SessionManagerPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ssm:UpdateInstanceInformation
                  - ssmmessages:CreateControlChannel
                  - ssmmessages:CreateDataChannel
                  - ssmmessages:OpenControlChannel
                  - ssmmessages:OpenDataChannel
                Resource: '*'

  SessionManagerInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref SessionManagerRole

  # Session Manager preferences
  SessionManagerPreferences:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Session
      DocumentFormat: JSON
      Content:
        schemaVersion: '1.0'
        description: Session Manager preferences
        sessionType: Standard_Stream
        inputs:
          s3BucketName: !Ref SessionLogsBucket
          s3KeyPrefix: session-logs/
          s3EncryptionEnabled: true
          cloudWatchLogGroupName: !Ref SessionLogsGroup
          cloudWatchEncryptionEnabled: true
```

### Session Manager CLI Usage
```bash
# Start session
aws ssm start-session --target i-1234567890abcdef0

# Start session with specific document
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"],"localPortNumber":["8080"]}'

# List active sessions
aws ssm describe-sessions --state Active

# Terminate session
aws ssm terminate-session --session-id session-id
```

---

## 5. Patch Manager

### Patch Baseline Configuration
```yaml
Resources:
  CustomPatchBaseline:
    Type: AWS::SSM::PatchBaseline
    Properties:
      Name: CustomLinuxBaseline
      Description: Custom patch baseline for Linux instances
      OperatingSystem: AMAZON_LINUX_2
      PatchGroups:
        - WebServers
        - DatabaseServers
      ApprovalRules:
        PatchRules:
          - PatchFilterGroup:
              PatchFilters:
                - Key: CLASSIFICATION
                  Values:
                    - Security
                    - Bugfix
                - Key: SEVERITY
                  Values:
                    - Critical
                    - Important
            ApproveAfterDays: 7
            ComplianceLevel: CRITICAL
          - PatchFilterGroup:
              PatchFilters:
                - Key: CLASSIFICATION
                  Values:
                    - Enhancement
            ApproveAfterDays: 30
            ComplianceLevel: MEDIUM
      ApprovedPatches:
        - kernel-4.14.123-*
      RejectedPatches:
        - kernel-4.14.100-*

  MaintenanceWindow:
    Type: AWS::SSM::MaintenanceWindow
    Properties:
      Name: PatchingMaintenanceWindow
      Description: Weekly patching window
      Schedule: cron(0 2 ? * SUN *)
      Duration: 4
      Cutoff: 1
      AllowUnassociatedTargets: false

  MaintenanceWindowTarget:
    Type: AWS::SSM::MaintenanceWindowTarget
    Properties:
      WindowId: !Ref MaintenanceWindow
      ResourceType: INSTANCE
      Targets:
        - Key: tag:PatchGroup
          Values:
            - WebServers
```

### Patch Compliance Reporting
```python
import boto3

def generate_patch_compliance_report():
    """Generate patch compliance report"""
    
    ssm = boto3.client('ssm')
    
    # Get patch compliance summary
    response = ssm.describe_instance_patch_states()
    
    compliance_data = []
    
    for instance in response['InstancePatchStates']:
        instance_id = instance['InstanceId']
        
        # Get detailed patch information
        patches = ssm.describe_instance_patches(InstanceId=instance_id)
        
        compliance_info = {
            'InstanceId': instance_id,
            'InstalledCount': instance['InstalledCount'],
            'MissingCount': instance['MissingCount'],
            'FailedCount': instance['FailedCount'],
            'NotApplicableCount': instance['NotApplicableCount'],
            'CriticalNonCompliantCount': 0,
            'SecurityNonCompliantCount': 0
        }
        
        # Analyze patch details
        for patch in patches['Patches']:
            if patch['State'] == 'Missing':
                if patch['Classification'] == 'Security':
                    compliance_info['SecurityNonCompliantCount'] += 1
                if patch['Severity'] == 'Critical':
                    compliance_info['CriticalNonCompliantCount'] += 1
        
        compliance_data.append(compliance_info)
    
    return compliance_data
```

---

## 6. Automation Documents

### Custom Automation Document
```yaml
Resources:
  CustomAutomationDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Automation
      DocumentFormat: YAML
      Content:
        schemaVersion: '0.3'
        description: 'Automated EC2 instance recovery'
        assumeRole: '{{ AutomationAssumeRole }}'
        parameters:
          InstanceId:
            type: String
            description: EC2 Instance ID to recover
          AutomationAssumeRole:
            type: String
            description: IAM role for automation
            default: ''
        mainSteps:
          - name: CheckInstanceStatus
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: DescribeInstanceStatus
              InstanceIds:
                - '{{ InstanceId }}'
            outputs:
              - Name: InstanceStatus
                Selector: '$.InstanceStatuses[0].InstanceStatus.Status'
                Type: String

          - name: StopInstance
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: StopInstances
              InstanceIds:
                - '{{ InstanceId }}'
            isEnd: false

          - name: WaitForInstanceStopped
            action: 'aws:waitForAwsResourceProperty'
            inputs:
              Service: ec2
              Api: DescribeInstances
              InstanceIds:
                - '{{ InstanceId }}'
              PropertySelector: '$.Reservations[0].Instances[0].State.Name'
              DesiredValues:
                - stopped
            timeoutSeconds: 300

          - name: StartInstance
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: StartInstances
              InstanceIds:
                - '{{ InstanceId }}'

          - name: WaitForInstanceRunning
            action: 'aws:waitForAwsResourceProperty'
            inputs:
              Service: ec2
              Api: DescribeInstances
              InstanceIds:
                - '{{ InstanceId }}'
              PropertySelector: '$.Reservations[0].Instances[0].State.Name'
              DesiredValues:
                - running
            timeoutSeconds: 300
```

### Automation Execution
```bash
# Execute automation document
aws ssm start-automation-execution \
  --document-name "CustomInstanceRecovery" \
  --parameters "InstanceId=i-1234567890abcdef0,AutomationAssumeRole=arn:aws:iam::123456789012:role/AutomationRole"

# Get automation execution status
aws ssm get-automation-execution \
  --automation-execution-id "execution-id"

# Stop automation execution
aws ssm stop-automation-execution \
  --automation-execution-id "execution-id"
```

---

## 7. CI/CD Integration

### Parameter Store in CodePipeline
```yaml
- Name: Deploy
  Actions:
    - Name: DeployApplication
      ActionTypeId:
        Category: Deploy
        Owner: AWS
        Provider: CloudFormation
        Version: 1
      Configuration:
        ActionMode: CREATE_UPDATE
        StackName: MyApplication
        TemplatePath: BuildOutput::template.yaml
        ParameterOverrides: |
          {
            "DatabaseHost": "{{resolve:ssm:/myapp/database/host}}",
            "DatabasePassword": "{{resolve:ssm-secure:/myapp/database/password}}",
            "Environment": "Production"
          }
        Capabilities: CAPABILITY_IAM
      InputArtifacts:
        - Name: BuildOutput
```

### Automation in CI/CD
```python
def lambda_handler(event, context):
    """Trigger Systems Manager automation from CodePipeline"""
    
    codepipeline = boto3.client('codepipeline')
    ssm = boto3.client('ssm')
    
    job_id = event['CodePipeline.job']['id']
    
    try:
        # Get deployment parameters
        user_parameters = json.loads(
            event['CodePipeline.job']['data']['actionConfiguration']['configuration']['UserParameters']
        )
        
        # Start automation execution
        response = ssm.start_automation_execution(
            DocumentName='DeploymentValidation',
            Parameters={
                'InstanceIds': user_parameters['instance_ids'],
                'ApplicationName': user_parameters['application_name'],
                'DeploymentId': event['CodePipeline.job']['data']['actionConfiguration']['configuration']['DeploymentId']
            }
        )
        
        execution_id = response['AutomationExecutionId']
        
        # Wait for automation completion
        waiter = ssm.get_waiter('automation_execution_success')
        waiter.wait(
            AutomationExecutionId=execution_id,
            WaiterConfig={'Delay': 30, 'MaxAttempts': 20}
        )
        
        # Get execution results
        execution = ssm.get_automation_execution(AutomationExecutionId=execution_id)
        
        if execution['AutomationExecution']['AutomationExecutionStatus'] == 'Success':
            codepipeline.put_job_success_result(jobId=job_id)
        else:
            codepipeline.put_job_failure_result(
                jobId=job_id,
                failureDetails={'message': 'Automation execution failed'}
            )
            
    except Exception as e:
        codepipeline.put_job_failure_result(
            jobId=job_id,
            failureDetails={'message': str(e)}
        )
```

---

## 8. Common Exam Scenarios

### Scenario 1: Centralized configuration management
**Solution:**
- Use Parameter Store for application configuration
- Implement hierarchical parameter structure
- Use SecureString for sensitive data
- Integrate with CloudFormation dynamic references

### Scenario 2: Secure instance access without SSH keys
**Solution:**
- Configure Session Manager on instances
- Use IAM policies for access control
- Enable session logging for audit
- Implement port forwarding for applications

### Scenario 3: Automated patch management
**Solution:**
- Create custom patch baselines
- Configure maintenance windows
- Set up patch groups for different environments
- Implement compliance reporting and alerting

### Scenario 4: Automated incident response
**Solution:**
- Create automation documents for common issues
- Integrate with CloudWatch alarms
- Use EventBridge for event-driven automation
- Implement rollback procedures

### Scenario 5: Cross-account parameter sharing
**Solution:**
- Use cross-account IAM roles
- Implement parameter access policies
- Use resource-based policies for Parameter Store
- Set up parameter replication across accounts

### Scenario 6: Configuration compliance monitoring
**Solution:**
- Use Config rules with Systems Manager
- Implement automated remediation
- Set up compliance dashboards
- Create compliance reports

### Scenario 7: Hybrid environment management
**Solution:**
- Install SSM Agent on on-premises servers
- Configure hybrid activations
- Use managed instances for on-premises resources
- Implement unified patch management

### Scenario 8: Application deployment automation
**Solution:**
- Create deployment automation documents
- Integrate with CodeDeploy and CodePipeline
- Implement blue/green deployment automation
- Use parameter store for deployment configuration

---

## 9. CLI Commands Reference

### Parameter Store Operations
```bash
# Put parameter
aws ssm put-parameter \
  --name "/myapp/config/setting" \
  --value "value" \
  --type "String"

# Get parameter
aws ssm get-parameter \
  --name "/myapp/config/setting" \
  --with-decryption

# Get parameters by path
aws ssm get-parameters-by-path \
  --path "/myapp" \
  --recursive

# Delete parameter
aws ssm delete-parameter \
  --name "/myapp/config/setting"
```

### Session Manager Operations
```bash
# Start session
aws ssm start-session --target i-1234567890abcdef0

# List sessions
aws ssm describe-sessions --state Active

# Terminate session
aws ssm terminate-session --session-id session-id
```

### Patch Manager Operations
```bash
# Create patch baseline
aws ssm create-patch-baseline \
  --name "MyBaseline" \
  --operating-system "AMAZON_LINUX_2"

# Register patch baseline
aws ssm register-patch-baseline-for-patch-group \
  --baseline-id "pb-1234567890abcdef0" \
  --patch-group "WebServers"

# Get patch compliance
aws ssm describe-instance-patch-states
```

---

## 10. Best Practices for DOP-C02 Exam

### Parameter Store
- Use hierarchical naming conventions
- Implement least privilege access policies
- Use SecureString for sensitive data
- Enable parameter versioning and change tracking
- Implement parameter validation and constraints

### Session Manager
- Use IAM policies for granular access control
- Enable session logging for compliance
- Implement session timeout policies
- Use port forwarding instead of direct access
- Regular audit of session access patterns

### Patch Management
- Create environment-specific patch baselines
- Use maintenance windows for controlled patching
- Implement patch testing in non-production first
- Set up automated compliance reporting
- Use patch groups for organized management

### Automation
- Design idempotent automation documents
- Implement proper error handling and rollback
- Use least privilege IAM roles for automation
- Test automation documents thoroughly
- Document automation procedures and dependencies

---

## 11. Exam Tips

### What to Remember
- **Parameter Store supports hierarchical organization** and cross-service integration
- **Session Manager eliminates SSH key management** and provides audit trails
- **Patch Manager automates patching** with baselines and maintenance windows
- **Automation documents enable workflow automation** with error handling
- **OpsCenter centralizes operational issues** and remediation
- **Systems Manager works with hybrid environments** (AWS + on-premises)
- **Integration with other AWS services** is extensive and native

### Common Traps
- Forgetting IAM permissions for Systems Manager operations
- Not configuring SSM Agent on instances (required for most features)
- Overlooking parameter store encryption for sensitive data
- Not implementing proper patch testing procedures
- Missing session logging configuration for compliance
- Not using least privilege for automation roles

### Scenario-Based Questions
- Focus on configuration management use cases
- Understand automation and remediation patterns
- Know security best practices for parameter management
- Understand patch management strategies
- Know integration patterns with CI/CD pipelines
- Understand hybrid environment management

---

## 12. Quick Reference Cheat Sheet

### Core Components
```
Parameter Store: Configuration and secrets management
Session Manager: Secure instance access
Patch Manager: Automated patching
Automation: Workflow automation
OpsCenter: Operational issue management
```

### Parameter Types
```
String: Plain text configuration
SecureString: Encrypted sensitive data
StringList: Comma-separated values
```

### Common IAM Actions
```
ssm:GetParameter, ssm:PutParameter
ssm:StartSession, ssm:TerminateSession
ssm:SendCommand, ssm:StartAutomationExecution
ssm:DescribeInstanceInformation
```

### Integration Patterns
```
CloudFormation: {{resolve:ssm:parameter-name}}
Lambda: boto3.client('ssm').get_parameter()
CodePipeline: Parameter overrides
EventBridge: Automation triggers
```

---

## 13. Summary

AWS Systems Manager is essential for operational management and automation in AWS environments and is heavily tested in the DOP-C02 exam. Key areas to master:

1. **Parameter Store** (configuration management, secrets, hierarchies)
2. **Session Manager** (secure access, auditing, port forwarding)
3. **Patch Manager** (automated patching, baselines, compliance)
4. **Automation** (runbooks, workflows, error handling)
5. **CI/CD integration** (parameter injection, automation triggers)
6. **Security best practices** (IAM, encryption, least privilege)
7. **Hybrid management** (on-premises integration, unified management)
8. **Operational excellence** (monitoring, compliance, remediation)

Understanding these concepts with hands-on practice will ensure success on Systems Manager-related questions in the DOP-C02 exam.