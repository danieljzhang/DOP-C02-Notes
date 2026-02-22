# AWS Systems Manager - DOP-C02 Exam Notes

## 1. Overview

**AWS Systems Manager** provides operational insights and enables automation of common administrative and operational tasks across AWS and on-premises resources. It's a unified interface for managing infrastructure at scale.

### Key Characteristics
- **Unified management** - Single interface for AWS and on-premises resources
- **Automation** - Runbooks, patch management, and operational tasks
- **Configuration management** - Parameter Store and configuration compliance
- **Session management** - Secure shell access without SSH keys or bastion hosts
- **Patch management** - Automated patching across instances and environments
- **Operational insights** - Centralized operational data and compliance reporting
- **Inventory management** - Track software, configurations, and metadata

### What Problem Does It Solve?
- Centralizes operational management across hybrid environments
- Automates routine administrative tasks and remediation workflows
- Provides secure access to instances without managing SSH infrastructure
- Enables configuration management and compliance at scale
- Facilitates patch management and security updates
- Supports operational insights and troubleshooting across fleets

---

## 2. Core Components

### Systems Manager Agent (SSM Agent)
```bash
# Check SSM Agent status
sudo systemctl status amazon-ssm-agent

# Start SSM Agent
sudo systemctl start amazon-ssm-agent

# Enable SSM Agent on boot
sudo systemctl enable amazon-ssm-agent

# View SSM Agent logs
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log

# Update SSM Agent
sudo yum update amazon-ssm-agent
```

### Instance Registration
```yaml
# CloudFormation for SSM-managed instances
Resources:
  SSMRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: SSMInstanceRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

  SSMInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref SSMRole

  ManagedInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890
      InstanceType: t3.micro
      IamInstanceProfile: !Ref SSMInstanceProfile
      SecurityGroupIds:
        - !Ref SSMSecurityGroup
      SubnetId: !Ref PrivateSubnet
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y amazon-ssm-agent
          systemctl enable amazon-ssm-agent
          systemctl start amazon-ssm-agent
      Tags:
        - Key: Name
          Value: SSM-Managed-Instance
        - Key: Environment
          Value: Production
        - Key: Patch Group
          Value: WebServers

  SSMSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for SSM managed instances
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: HTTPS for SSM communication
```

---

## 3. Session Manager

### Session Manager Configuration
```yaml
Resources:
  SessionManagerPreferences:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Session
      DocumentFormat: JSON
      Name: SSM-SessionManagerRunShell
      Content:
        schemaVersion: '1.0'
        description: Document to hold regional settings for Session Manager
        sessionType: Standard_Stream
        inputs:
          s3BucketName: !Ref SessionLogsBucket
          s3KeyPrefix: session-logs/
          s3EncryptionEnabled: true
          cloudWatchLogGroupName: !Ref SessionLogsGroup
          cloudWatchEncryptionEnabled: true
          idleSessionTimeout: '20'
          maxSessionDuration: '60'
          runAsEnabled: false
          runAsDefaultUser: ssm-user
          shellProfile:
            windows: 'date'
            linux: 'pwd;whoami'

  SessionLogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-session-logs"
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  SessionLogsGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/ssm/session-logs
      RetentionInDays: 30
```

### Session Manager CLI Usage
```bash
# Start interactive session
aws ssm start-session --target i-1234567890abcdef0

# Start session with specific document
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3306"],"localPortNumber":["3306"]}'

# Start session with port forwarding
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.cluster-xyz.us-east-1.rds.amazonaws.com"],"portNumber":["3306"],"localPortNumber":["3306"]}'

# List active sessions
aws ssm describe-sessions --state Active

# Terminate session
aws ssm terminate-session --session-id session-id

# Get session history
aws ssm get-connection-status --target i-1234567890abcdef0
```

---

## 4. Parameter Store

### Parameter Management
```bash
# Create string parameter
aws ssm put-parameter \
  --name "/myapp/database/host" \
  --value "db.example.com" \
  --type "String" \
  --description "Database hostname"

# Create secure string parameter
aws ssm put-parameter \
  --name "/myapp/database/password" \
  --value "supersecret" \
  --type "SecureString" \
  --key-id "alias/parameter-store-key" \
  --description "Database password"

# Create string list parameter
aws ssm put-parameter \
  --name "/myapp/allowed-ips" \
  --value "10.0.0.1,10.0.0.2,10.0.0.3" \
  --type "StringList" \
  --description "Allowed IP addresses"

# Get parameter
aws ssm get-parameter --name "/myapp/database/host"

# Get parameter with decryption
aws ssm get-parameter --name "/myapp/database/password" --with-decryption

# Get parameters by path
aws ssm get-parameters-by-path \
  --path "/myapp/database" \
  --recursive \
  --with-decryption

# Get parameter history
aws ssm get-parameter-history --name "/myapp/database/password"
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
      Tags:
        Environment: !Ref Environment
        Application: MyApp

  DatabasePassword:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /myapp/database/password
      Type: SecureString
      Value: !Ref DatabaseMasterPassword
      Description: Database master password
      KeyId: !Ref ParameterStoreKMSKey

  # Reference parameters in other resources
  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          DB_HOST: '{{resolve:ssm:/myapp/database/host}}'
          DB_PASSWORD: '{{resolve:ssm-secure:/myapp/database/password}}'
      Code:
        ZipFile: |
          import os
          import boto3
          
          def lambda_handler(event, context):
              # Parameters are automatically resolved
              db_host = os.environ['DB_HOST']
              db_password = os.environ['DB_PASSWORD']
              
              # Use parameters in application logic
              return {'statusCode': 200}

  # Parameter Store KMS Key
  ParameterStoreKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for Parameter Store encryption
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "kms:*"
            Resource: "*"
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
            ApproveAfterDays: 0
            ComplianceLevel: CRITICAL
            EnableNonSecurity: false
          - PatchFilterGroup:
              PatchFilters:
                - Key: CLASSIFICATION
                  Values:
                    - Enhancement
                    - Recommended
            ApproveAfterDays: 7
            ComplianceLevel: MEDIUM
      ApprovedPatches:
        - kernel-4.14.123-*
      RejectedPatches:
        - kernel-4.14.100-*
      RejectedPatchesAction: BLOCK

  MaintenanceWindow:
    Type: AWS::SSM::MaintenanceWindow
    Properties:
      Name: PatchingMaintenanceWindow
      Description: Weekly patching window for production servers
      Schedule: cron(0 2 ? * SUN *)
      Duration: 4
      Cutoff: 1
      AllowUnassociatedTargets: false
      Tags:
        - Key: Environment
          Value: Production

  MaintenanceWindowTarget:
    Type: AWS::SSM::MaintenanceWindowTarget
    Properties:
      WindowId: !Ref MaintenanceWindow
      ResourceType: INSTANCE
      Targets:
        - Key: tag:PatchGroup
          Values:
            - WebServers
      Name: WebServersTarget
      Description: Web servers for patching

  PatchingTask:
    Type: AWS::SSM::MaintenanceWindowTask
    Properties:
      WindowId: !Ref MaintenanceWindow
      TaskType: RUN_COMMAND
      TaskArn: AWS-RunPatchBaseline
      Targets:
        - Key: WindowTargetIds
          Values:
            - !Ref MaintenanceWindowTarget
      ServiceRoleArn: !GetAtt MaintenanceWindowRole.Arn
      TaskParameters:
        Operation:
          - Install
      Priority: 1
      MaxConcurrency: "50%"
      MaxErrors: "10%"
      Name: PatchInstallationTask
```

### Patch Compliance Monitoring
```python
import boto3
import json
from datetime import datetime, timedelta

class PatchComplianceMonitor:
    def __init__(self):
        self.ssm_client = boto3.client('ssm')
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
    
    def generate_patch_compliance_report(self):
        """Generate comprehensive patch compliance report"""
        
        # Get all managed instances
        instances_response = self.ssm_client.describe_instance_information()
        
        compliance_data = {
            'report_timestamp': datetime.utcnow().isoformat(),
            'total_instances': len(instances_response['InstanceInformationList']),
            'compliant_instances': 0,
            'non_compliant_instances': 0,
            'instances_details': []
        }
        
        for instance in instances_response['InstanceInformationList']:
            instance_id = instance['InstanceId']
            
            try:
                # Get patch compliance for instance
                patch_state = self.ssm_client.describe_instance_patch_states(
                    InstanceIds=[instance_id]
                )
                
                if patch_state['InstancePatchStates']:
                    state = patch_state['InstancePatchStates'][0]
                    
                    instance_compliance = {
                        'instance_id': instance_id,
                        'platform_name': instance.get('PlatformName', 'Unknown'),
                        'platform_version': instance.get('PlatformVersion', 'Unknown'),
                        'installed_count': state['InstalledCount'],
                        'missing_count': state['MissingCount'],
                        'failed_count': state['FailedCount'],
                        'not_applicable_count': state['NotApplicableCount'],
                        'operation': state['Operation'],
                        'operation_start_time': state.get('OperationStartTime', '').isoformat() if state.get('OperationStartTime') else None,
                        'operation_end_time': state.get('OperationEndTime', '').isoformat() if state.get('OperationEndTime') else None
                    }
                    
                    # Determine compliance status
                    if state['MissingCount'] == 0 and state['FailedCount'] == 0:
                        instance_compliance['compliance_status'] = 'COMPLIANT'
                        compliance_data['compliant_instances'] += 1
                    else:
                        instance_compliance['compliance_status'] = 'NON_COMPLIANT'
                        compliance_data['non_compliant_instances'] += 1
                        
                        # Get details of missing patches
                        instance_compliance['missing_patches'] = self.get_missing_patches(instance_id)
                    
                    compliance_data['instances_details'].append(instance_compliance)
            
            except Exception as e:
                print(f"Error getting patch state for {instance_id}: {str(e)}")
        
        # Send compliance metrics to CloudWatch
        self.send_compliance_metrics(compliance_data)
        
        # Send alert if compliance is below threshold
        compliance_percentage = (compliance_data['compliant_instances'] / compliance_data['total_instances']) * 100
        if compliance_percentage < 90:  # 90% compliance threshold
            self.send_compliance_alert(compliance_data, compliance_percentage)
        
        return compliance_data
    
    def get_missing_patches(self, instance_id):
        """Get details of missing patches for an instance"""
        
        try:
            patches_response = self.ssm_client.describe_instance_patches(
                InstanceId=instance_id,
                Filters=[
                    {
                        'Key': 'State',
                        'Values': ['Missing']
                    }
                ]
            )
            
            missing_patches = []
            for patch in patches_response['Patches']:
                missing_patches.append({
                    'id': patch['Id'],
                    'classification': patch['Classification'],
                    'severity': patch['Severity'],
                    'title': patch['Title'],
                    'release_date': patch['ReleaseDate'].isoformat() if patch.get('ReleaseDate') else None
                })
            
            return missing_patches[:10]  # Return first 10 missing patches
        
        except Exception as e:
            print(f"Error getting missing patches for {instance_id}: {str(e)}")
            return []
    
    def send_compliance_metrics(self, compliance_data):
        """Send patch compliance metrics to CloudWatch"""
        
        self.cloudwatch.put_metric_data(
            Namespace='AWS/SSM/PatchCompliance',
            MetricData=[
                {
                    'MetricName': 'CompliantInstances',
                    'Value': compliance_data['compliant_instances'],
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                },
                {
                    'MetricName': 'NonCompliantInstances',
                    'Value': compliance_data['non_compliant_instances'],
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                },
                {
                    'MetricName': 'CompliancePercentage',
                    'Value': (compliance_data['compliant_instances'] / compliance_data['total_instances']) * 100,
                    'Unit': 'Percent',
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
    
    def send_compliance_alert(self, compliance_data, compliance_percentage):
        """Send compliance alert when threshold is not met"""
        
        message = f"""
        Patch Compliance Alert
        =====================
        
        Compliance Percentage: {compliance_percentage:.1f}%
        Total Instances: {compliance_data['total_instances']}
        Compliant Instances: {compliance_data['compliant_instances']}
        Non-Compliant Instances: {compliance_data['non_compliant_instances']}
        
        Non-compliant instances require immediate attention.
        Please review the patch compliance report for details.
        """
        
        self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:patch-compliance-alerts',
            Message=message,
            Subject=f'Patch Compliance Alert - {compliance_percentage:.1f}% Compliant'
        )
```

---

## 6. Automation Documents

### Custom Automation Document
```yaml
Resources:
  CustomRemediationDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Automation
      DocumentFormat: YAML
      Name: CustomSecurityRemediation
      Content:
        schemaVersion: '0.3'
        description: Automated security remediation for non-compliant instances
        assumeRole: '{{ AutomationAssumeRole }}'
        parameters:
          InstanceId:
            type: String
            description: EC2 Instance ID to remediate
          AutomationAssumeRole:
            type: String
            description: IAM role for automation execution
            default: ''
          NotificationTopicArn:
            type: String
            description: SNS topic for notifications
            default: ''
        mainSteps:
          - name: CheckInstanceStatus
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: DescribeInstances
              InstanceIds:
                - '{{ InstanceId }}'
            outputs:
              - Name: InstanceState
                Selector: '$.Reservations[0].Instances[0].State.Name'
                Type: String
          
          - name: StopInstanceIfRunning
            action: 'aws:changeInstanceState'
            inputs:
              InstanceIds:
                - '{{ InstanceId }}'
              DesiredState: stopped
            isEnd: false
            onFailure: 'step:NotifyFailure'
          
          - name: CreateSecuritySnapshot
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: CreateSnapshot
              VolumeId: '{{ GetVolumeId.VolumeId }}'
              Description: 'Security remediation snapshot for {{ InstanceId }}'
            outputs:
              - Name: SnapshotId
                Selector: '$.SnapshotId'
                Type: String
          
          - name: ApplySecurityPatches
            action: 'aws:runCommand'
            inputs:
              DocumentName: AWS-RunPatchBaseline
              InstanceIds:
                - '{{ InstanceId }}'
              Parameters:
                Operation: Install
            outputs:
              - Name: CommandId
                Selector: '$.CommandId'
                Type: String
          
          - name: UpdateSecurityGroups
            action: 'aws:executeAwsApi'
            inputs:
              Service: ec2
              Api: ModifyInstanceAttribute
              InstanceId: '{{ InstanceId }}'
              Groups:
                - sg-secure123456
          
          - name: StartInstance
            action: 'aws:changeInstanceState'
            inputs:
              InstanceIds:
                - '{{ InstanceId }}'
              DesiredState: running
          
          - name: VerifyCompliance
            action: 'aws:executeScript'
            inputs:
              Runtime: python3.8
              Handler: verify_compliance
              Script: |
                import boto3
                import time
                
                def verify_compliance(events, context):
                    ssm = boto3.client('ssm')
                    instance_id = events['InstanceId']
                    
                    # Wait for compliance scan
                    time.sleep(300)  # 5 minutes
                    
                    # Check compliance status
                    response = ssm.describe_instance_patch_states(
                        InstanceIds=[instance_id]
                    )
                    
                    if response['InstancePatchStates']:
                        state = response['InstancePatchStates'][0]
                        if state['MissingCount'] == 0 and state['FailedCount'] == 0:
                            return {'ComplianceStatus': 'COMPLIANT'}
                    
                    return {'ComplianceStatus': 'NON_COMPLIANT'}
              InputPayload:
                InstanceId: '{{ InstanceId }}'
            outputs:
              - Name: ComplianceStatus
                Selector: '$.Payload.ComplianceStatus'
                Type: String
          
          - name: NotifySuccess
            action: 'aws:executeAwsApi'
            inputs:
              Service: sns
              Api: Publish
              TopicArn: '{{ NotificationTopicArn }}'
              Message: 'Security remediation completed successfully for instance {{ InstanceId }}'
              Subject: 'Security Remediation Success'
            isEnd: true
          
          - name: NotifyFailure
            action: 'aws:executeAwsApi'
            inputs:
              Service: sns
              Api: Publish
              TopicArn: '{{ NotificationTopicArn }}'
              Message: 'Security remediation failed for instance {{ InstanceId }}'
              Subject: 'Security Remediation Failure'
            isEnd: true
```

### Run Command Documents
```bash
# Execute run command
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["echo Hello World","uptime","df -h"]' \
  --targets "Key=tag:Environment,Values=Production" \
  --max-concurrency "10" \
  --max-errors "2"

# Execute PowerShell command on Windows
aws ssm send-command \
  --document-name "AWS-RunPowerShellScript" \
  --parameters 'commands=["Get-Process","Get-Service"]' \
  --targets "Key=instanceids,Values=i-1234567890abcdef0"

# Install application via run command
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --parameters 'action=Install,name=AmazonCloudWatchAgent' \
  --targets "Key=tag:Role,Values=WebServer"

# Get command execution results
aws ssm get-command-invocation \
  --command-id "command-id-here" \
  --instance-id "i-1234567890abcdef0"
```

---

## 7. State Manager

### State Manager Association
```yaml
Resources:
  CloudWatchAgentAssociation:
    Type: AWS::SSM::Association
    Properties:
      Name: AmazonCloudWatch-ManageAgent
      Targets:
        - Key: tag:Environment
          Values:
            - Production
            - Staging
      Parameters:
        action: configure
        mode: ec2
        optionalConfigurationSource: ssm
        optionalConfigurationLocation: !Ref CloudWatchAgentConfig
        optionalRestart: 'yes'
      ScheduleExpression: rate(30 minutes)
      ComplianceSeverity: MEDIUM
      MaxConcurrency: "50%"
      MaxErrors: "10%"

  SecurityComplianceAssociation:
    Type: AWS::SSM::Association
    Properties:
      Name: !Ref SecurityComplianceDocument
      Targets:
        - Key: tag:SecurityCompliance
          Values:
            - Required
      ScheduleExpression: cron(0 2 ? * SUN *)
      Parameters:
        ComplianceType: Security
        RemediationAction: Automatic
      ComplianceSeverity: HIGH

  # Custom compliance document
  SecurityComplianceDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Command
      DocumentFormat: YAML
      Name: SecurityComplianceCheck
      Content:
        schemaVersion: '2.2'
        description: Security compliance check and remediation
        parameters:
          ComplianceType:
            type: String
            description: Type of compliance check
            default: Security
          RemediationAction:
            type: String
            description: Remediation action to take
            allowedValues:
              - Automatic
              - Manual
            default: Manual
        mainSteps:
          - action: 'aws:runShellScript'
            name: SecurityComplianceCheck
            inputs:
              runCommand:
                - |
                  #!/bin/bash
                  
                  # Check SSH configuration
                  if grep -q "PermitRootLogin yes" /etc/ssh/sshd_config; then
                    echo "COMPLIANCE_VIOLATION: Root login enabled"
                    if [ "{{ RemediationAction }}" == "Automatic" ]; then
                      sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
                      systemctl reload sshd
                      echo "REMEDIATION: Root login disabled"
                    fi
                  fi
                  
                  # Check for unencrypted volumes
                  unencrypted_volumes=$(lsblk -f | grep -v LUKS | grep -c ext4)
                  if [ $unencrypted_volumes -gt 0 ]; then
                    echo "COMPLIANCE_VIOLATION: Unencrypted volumes detected"
                  fi
                  
                  # Check firewall status
                  if ! systemctl is-active --quiet iptables; then
                    echo "COMPLIANCE_VIOLATION: Firewall not active"
                    if [ "{{ RemediationAction }}" == "Automatic" ]; then
                      systemctl enable iptables
                      systemctl start iptables
                      echo "REMEDIATION: Firewall enabled"
                    fi
                  fi
                  
                  echo "Security compliance check completed"
```

---

## 8. Inventory and Compliance

### Inventory Collection
```yaml
Resources:
  InventoryAssociation:
    Type: AWS::SSM::Association
    Properties:
      Name: AWS-GatherSoftwareInventory
      Targets:
        - Key: InstanceIds
          Values:
            - "*"
      ScheduleExpression: rate(1 day)
      Parameters:
        applications: Enabled
        awsComponents: Enabled
        customInventory: Enabled
        files: |
          [
            {
              "Path": "/etc",
              "Pattern": ["*.conf", "*.cfg"],
              "Recursive": true
            },
            {
              "Path": "/var/log",
              "Pattern": ["*.log"],
              "Recursive": false
            }
          ]
        networkConfig: Enabled
        services: Enabled
        windowsRegistry: Enabled
        windowsRoles: Enabled

  # Custom inventory for application metadata
  CustomInventoryDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Command
      DocumentFormat: YAML
      Name: CustomApplicationInventory
      Content:
        schemaVersion: '2.2'
        description: Collect custom application inventory
        mainSteps:
          - action: 'aws:runShellScript'
            name: CollectApplicationInventory
            inputs:
              runCommand:
                - |
                  #!/bin/bash
                  
                  # Collect application versions
                  app_inventory=$(cat << EOF
                  {
                    "SchemaVersion": "1.0",
                    "TypeName": "Custom:ApplicationInventory",
                    "Content": [
                      {
                        "ApplicationName": "MyWebApp",
                        "Version": "$(cat /opt/myapp/VERSION 2>/dev/null || echo 'Unknown')",
                        "InstallDate": "$(stat -c %y /opt/myapp 2>/dev/null || echo 'Unknown')",
                        "Status": "$(systemctl is-active myapp 2>/dev/null || echo 'Unknown')"
                      }
                    ]
                  }
                  EOF
                  )
                  
                  # Send inventory to Systems Manager
                  aws ssm put-inventory \
                    --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id) \
                    --items "$app_inventory"
```

### Compliance Monitoring
```python
import boto3
import json
from datetime import datetime, timedelta

class SSMComplianceMonitor:
    def __init__(self):
        self.ssm_client = boto3.client('ssm')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_compliance_summary(self):
        """Get overall compliance summary across all resources"""
        
        compliance_summary = {
            'timestamp': datetime.utcnow().isoformat(),
            'compliance_types': {},
            'overall_compliance': 0
        }
        
        # Get compliance summary by type
        response = self.ssm_client.list_compliance_summaries()
        
        total_compliant = 0
        total_resources = 0
        
        for summary in response['ComplianceSummaryItems']:
            compliance_type = summary['ComplianceType']
            compliant_count = summary['CompliantSummary']['CompliantCount']
            non_compliant_count = summary['CompliantSummary']['NonCompliantCount']
            
            total_compliant += compliant_count
            total_resources += compliant_count + non_compliant_count
            
            compliance_summary['compliance_types'][compliance_type] = {
                'compliant': compliant_count,
                'non_compliant': non_compliant_count,
                'compliance_percentage': (compliant_count / (compliant_count + non_compliant_count)) * 100 if (compliant_count + non_compliant_count) > 0 else 0
            }
        
        if total_resources > 0:
            compliance_summary['overall_compliance'] = (total_compliant / total_resources) * 100
        
        return compliance_summary
    
    def get_non_compliant_resources(self, compliance_type=None):
        """Get details of non-compliant resources"""
        
        filters = [
            {
                'Key': 'ComplianceType',
                'Values': [compliance_type] if compliance_type else ['Patch', 'Association', 'Custom:Security']
            },
            {
                'Key': 'Status',
                'Values': ['NON_COMPLIANT']
            }
        ]
        
        non_compliant_resources = []
        
        paginator = self.ssm_client.get_paginator('list_compliance_items')
        
        for page in paginator.paginate(Filters=filters):
            for item in page['ComplianceItems']:
                non_compliant_resources.append({
                    'resource_id': item['ResourceId'],
                    'resource_type': item['ResourceType'],
                    'compliance_type': item['ComplianceType'],
                    'status': item['Status'],
                    'severity': item['Severity'],
                    'execution_summary': item.get('ExecutionSummary', {}),
                    'details': item.get('Details', {})
                })
        
        return non_compliant_resources
    
    def remediate_non_compliant_resources(self, resources):
        """Trigger remediation for non-compliant resources"""
        
        remediation_results = []
        
        for resource in resources:
            resource_id = resource['resource_id']
            compliance_type = resource['compliance_type']
            
            try:
                if compliance_type == 'Patch':
                    # Trigger patch installation
                    response = self.ssm_client.send_command(
                        InstanceIds=[resource_id],
                        DocumentName='AWS-RunPatchBaseline',
                        Parameters={'Operation': ['Install']}
                    )
                    
                    remediation_results.append({
                        'resource_id': resource_id,
                        'remediation_type': 'Patch Installation',
                        'command_id': response['Command']['CommandId'],
                        'status': 'INITIATED'
                    })
                
                elif compliance_type == 'Association':
                    # Re-run association
                    associations = self.ssm_client.describe_instance_associations_status(
                        InstanceId=resource_id
                    )
                    
                    for assoc in associations['InstanceAssociationStatusInfos']:
                        if assoc['Status'] != 'Success':
                            self.ssm_client.send_command(
                                InstanceIds=[resource_id],
                                DocumentName=assoc['Name']
                            )
                    
                    remediation_results.append({
                        'resource_id': resource_id,
                        'remediation_type': 'Association Re-run',
                        'status': 'INITIATED'
                    })
            
            except Exception as e:
                remediation_results.append({
                    'resource_id': resource_id,
                    'remediation_type': 'Failed',
                    'error': str(e),
                    'status': 'FAILED'
                })
        
        return remediation_results
```

---

## 9. Common Exam Scenarios

### Scenario 1: Automated Patch Management
```yaml
# Complete automated patch management setup
Resources:
  # Production patch baseline
  ProductionPatchBaseline:
    Type: AWS::SSM::PatchBaseline
    Properties:
      Name: Production-Linux-Baseline
      Description: Production patch baseline with strict approval rules
      OperatingSystem: AMAZON_LINUX_2
      PatchGroups:
        - Production-WebServers
        - Production-AppServers
      ApprovalRules:
        PatchRules:
          - PatchFilterGroup:
              PatchFilters:
                - Key: CLASSIFICATION
                  Values: [Security]
                - Key: SEVERITY
                  Values: [Critical, Important]
            ApproveAfterDays: 0
            ComplianceLevel: CRITICAL
          - PatchFilterGroup:
              PatchFilters:
                - Key: CLASSIFICATION
                  Values: [Bugfix]
                - Key: SEVERITY
                  Values: [Medium, Low]
            ApproveAfterDays: 7
            ComplianceLevel: MEDIUM

  # Maintenance window for production patching
  ProductionMaintenanceWindow:
    Type: AWS::SSM::MaintenanceWindow
    Properties:
      Name: Production-Patching-Window
      Description: Production patching during maintenance window
      Schedule: cron(0 3 ? * SUN *)  # 3 AM every Sunday
      Duration: 6
      Cutoff: 1
      AllowUnassociatedTargets: false

  # Maintenance window targets
  ProductionPatchTargets:
    Type: AWS::SSM::MaintenanceWindowTarget
    Properties:
      WindowId: !Ref ProductionMaintenanceWindow
      ResourceType: INSTANCE
      Targets:
        - Key: tag:Environment
          Values: [Production]
        - Key: tag:PatchGroup
          Values: [Production-WebServers, Production-AppServers]

  # Patch installation task
  PatchInstallationTask:
    Type: AWS::SSM::MaintenanceWindowTask
    Properties:
      WindowId: !Ref ProductionMaintenanceWindow
      TaskType: RUN_COMMAND
      TaskArn: AWS-RunPatchBaseline
      Targets:
        - Key: WindowTargetIds
          Values: [!Ref ProductionPatchTargets]
      ServiceRoleArn: !GetAtt MaintenanceWindowRole.Arn
      TaskParameters:
        Operation: [Install]
      Priority: 1
      MaxConcurrency: "25%"
      MaxErrors: "5%"

  # Post-patch compliance check
  ComplianceCheckTask:
    Type: AWS::SSM::MaintenanceWindowTask
    Properties:
      WindowId: !Ref ProductionMaintenanceWindow
      TaskType: RUN_COMMAND
      TaskArn: !Ref PostPatchComplianceDocument
      Targets:
        - Key: WindowTargetIds
          Values: [!Ref ProductionPatchTargets]
      ServiceRoleArn: !GetAtt MaintenanceWindowRole.Arn
      Priority: 2
      MaxConcurrency: "50%"
      MaxErrors: "10%"

  # Custom document for post-patch compliance
  PostPatchComplianceDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Command
      DocumentFormat: YAML
      Name: PostPatchComplianceCheck
      Content:
        schemaVersion: '2.2'
        description: Post-patch compliance verification
        mainSteps:
          - action: 'aws:runShellScript'
            name: ComplianceCheck
            inputs:
              runCommand:
                - |
                  #!/bin/bash
                  
                  # Check if reboot is required
                  if [ -f /var/run/reboot-required ]; then
                    echo "REBOOT_REQUIRED: System requires reboot"
                    # Schedule reboot during next maintenance window
                    echo "reboot" | at now + 5 minutes
                  fi
                  
                  # Verify critical services are running
                  services=("httpd" "mysqld" "nginx")
                  for service in "${services[@]}"; do
                    if systemctl is-active --quiet $service; then
                      echo "SERVICE_OK: $service is running"
                    else
                      echo "SERVICE_ERROR: $service is not running"
                      systemctl start $service
                    fi
                  done
                  
                  # Send compliance status to CloudWatch
                  aws cloudwatch put-metric-data \
                    --namespace "AWS/SSM/PatchCompliance" \
                    --metric-data MetricName=PostPatchCheck,Value=1,Unit=Count
```

### Scenario 2: Cross-Account Systems Management
```python
# Cross-account Systems Manager setup
import boto3
import json

class CrossAccountSSMManager:
    def __init__(self):
        self.sts_client = boto3.client('sts')
        self.ssm_client = boto3.client('ssm')
    
    def setup_cross_account_access(self, target_accounts, cross_account_role_name):
        """Setup cross-account access for Systems Manager"""
        
        setup_results = {}
        
        for account_id in target_accounts:
            try:
                # Assume role in target account
                assumed_role = self.sts_client.assume_role(
                    RoleArn=f"arn:aws:iam::{account_id}:role/{cross_account_role_name}",
                    RoleSessionName='CrossAccountSSMSetup'
                )
                
                # Create session with assumed role credentials
                target_session = boto3.Session(
                    aws_access_key_id=assumed_role['Credentials']['AccessKeyId'],
                    aws_secret_access_key=assumed_role['Credentials']['SecretAccessKey'],
                    aws_session_token=assumed_role['Credentials']['SessionToken']
                )
                
                target_ssm = target_session.client('ssm')
                
                # Setup resource data sync in target account
                sync_name = f"CrossAccountSync-{account_id}"
                
                try:
                    target_ssm.create_resource_data_sync(
                        SyncName=sync_name,
                        S3Destination={
                            'BucketName': 'central-ssm-inventory-bucket',
                            'Prefix': f'account-{account_id}/',
                            'SyncFormat': 'JsonSerDe',
                            'Region': 'us-east-1'
                        }
                    )
                    
                    setup_results[account_id] = {
                        'status': 'SUCCESS',
                        'sync_name': sync_name
                    }
                
                except target_ssm.exceptions.ResourceDataSyncAlreadyExistsException:
                    setup_results[account_id] = {
                        'status': 'EXISTS',
                        'sync_name': sync_name
                    }
                
                # Setup compliance data aggregation
                self.setup_compliance_aggregation(target_ssm, account_id)
                
            except Exception as e:
                setup_results[account_id] = {
                    'status': 'FAILED',
                    'error': str(e)
                }
        
        return setup_results
    
    def setup_compliance_aggregation(self, target_ssm, account_id):
        """Setup compliance data aggregation"""
        
        # Create association for compliance data collection
        association_name = f"ComplianceDataCollection-{account_id}"
        
        try:
            target_ssm.create_association(
                Name='AWS-GatherSoftwareInventory',
                Targets=[
                    {
                        'Key': 'InstanceIds',
                        'Values': ['*']
                    }
                ],
                ScheduleExpression='rate(1 day)',
                AssociationName=association_name,
                Parameters={
                    'applications': ['Enabled'],
                    'awsComponents': ['Enabled'],
                    'networkConfig': ['Enabled'],
                    'windowsUpdates': ['Enabled'],
                    'instanceDetailedInformation': ['Enabled']
                }
            )
        
        except target_ssm.exceptions.AssociationAlreadyExists:
            pass  # Association already exists
    
    def aggregate_compliance_data(self, accounts):
        """Aggregate compliance data from multiple accounts"""
        
        aggregated_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'accounts': {},
            'summary': {
                'total_instances': 0,
                'compliant_instances': 0,
                'non_compliant_instances': 0
            }
        }
        
        for account_id in accounts:
            try:
                # Get compliance data for account
                account_compliance = self.get_account_compliance(account_id)
                aggregated_data['accounts'][account_id] = account_compliance
                
                # Update summary
                aggregated_data['summary']['total_instances'] += account_compliance['total_instances']
                aggregated_data['summary']['compliant_instances'] += account_compliance['compliant_instances']
                aggregated_data['summary']['non_compliant_instances'] += account_compliance['non_compliant_instances']
            
            except Exception as e:
                aggregated_data['accounts'][account_id] = {
                    'error': str(e),
                    'status': 'FAILED'
                }
        
        # Calculate overall compliance percentage
        if aggregated_data['summary']['total_instances'] > 0:
            compliance_percentage = (aggregated_data['summary']['compliant_instances'] / 
                                   aggregated_data['summary']['total_instances']) * 100
            aggregated_data['summary']['compliance_percentage'] = compliance_percentage
        
        return aggregated_data
```

---

## 10. Exam Tips

- **Understand SSM Agent** - Installation, configuration, and troubleshooting
- **Master Session Manager** - Secure access without SSH keys or bastion hosts
- **Know Parameter Store** - Hierarchical parameters, encryption, and CloudFormation integration
- **Practice Patch Manager** - Baselines, maintenance windows, and compliance reporting
- **Learn Automation** - Documents, workflows, and cross-service integration
- **Understand State Manager** - Associations, compliance, and configuration drift
- **Know Run Command** - Document execution, targeting, and result handling
- **Practice Inventory** - Collection, custom inventory, and compliance monitoring
- **Master troubleshooting** - Common issues with agents, permissions, and connectivity
- **Understand integration** - CloudWatch, SNS, Lambda, and cross-account scenarios