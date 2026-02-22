# AWS GuardDuty - DOP-C02 Exam Notes

## 1. Overview

**AWS GuardDuty** is a threat detection service that continuously monitors for malicious activity and unauthorized behavior to protect your AWS accounts, workloads, and data stored in Amazon S3.

### Key Characteristics
- **Threat intelligence** - Uses machine learning, anomaly detection, and integrated threat intelligence
- **Continuous monitoring** - 24/7 monitoring of AWS accounts and workloads
- **Multi-data source** - Analyzes VPC Flow Logs, DNS logs, CloudTrail logs, and S3 data events
- **Automated response** - Integration with Lambda, SNS, and other services for automated remediation
- **Multi-account support** - Centralized security monitoring across AWS Organizations
- **Malware detection** - Scans EBS volumes and container images for malware
- **Runtime monitoring** - Monitors EKS clusters and EC2 instances for suspicious activity

### What Problem Does It Solve?
- Detects sophisticated threats and attacks in real-time
- Provides centralized security monitoring across multiple accounts
- Reduces time to detect and respond to security incidents
- Automates threat detection without requiring security expertise
- Protects against data exfiltration and cryptocurrency mining
- Monitors for insider threats and compromised credentials

---

## 2. Core Components

### Detector Configuration
```yaml
# CloudFormation for GuardDuty setup
Resources:
  GuardDutyDetector:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: true
      FindingPublishingFrequency: FIFTEEN_MINUTES
      DataSources:
        S3Logs:
          Enable: true
        KubernetesConfiguration:
          AuditLogs:
            Enable: true
        MalwareProtection:
          ScanEc2InstanceWithFindings:
            EbsVolumes: true

  # Custom threat intelligence set
  ThreatIntelSet:
    Type: AWS::GuardDuty::ThreatIntelSet
    Properties:
      DetectorId: !Ref GuardDutyDetector
      Format: TXT
      Location: !Sub "https://${ThreatIntelBucket}.s3.amazonaws.com/threat-intel.txt"
      Name: CustomThreatIntelligence
      Activate: true

  # IP whitelist
  IPSet:
    Type: AWS::GuardDuty::IPSet
    Properties:
      DetectorId: !Ref GuardDutyDetector
      Format: TXT
      Location: !Sub "https://${ThreatIntelBucket}.s3.amazonaws.com/trusted-ips.txt"
      Name: TrustedIPAddresses
      Activate: true

  ThreatIntelBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-threat-intel"
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
```

### Multi-Account Setup
```yaml
# Master account configuration
Resources:
  GuardDutyMaster:
    Type: AWS::GuardDuty::Master
    Properties:
      DetectorId: !Ref GuardDutyDetector
      MasterId: !Ref AWS::AccountId
      InvitationId: !Ref InvitationId

  # Member account invitation
  GuardDutyMember:
    Type: AWS::GuardDuty::Member
    Properties:
      DetectorId: !Ref GuardDutyDetector
      MemberId: "123456789012"
      Email: "security@company.com"
      Message: "Please accept GuardDuty invitation"
      Status: Invited
      DisableEmailNotification: false

  # Organization configuration (for AWS Organizations)
  OrganizationConfiguration:
    Type: AWS::GuardDuty::OrganizationConfiguration
    Properties:
      DetectorId: !Ref GuardDutyDetector
      AutoEnable: true
      DataSources:
        S3Logs:
          AutoEnable: true
        KubernetesConfiguration:
          AuditLogs:
            AutoEnable: true
        MalwareProtection:
          ScanEc2InstanceWithFindings:
            EbsVolumes:
              AutoEnable: true
```

---

## 3. Finding Types and Analysis

### Common Finding Types
```python
# GuardDuty finding categories and examples
GUARDDUTY_FINDING_TYPES = {
    "Backdoor": [
        "Backdoor:EC2/C&CActivity.B!DNS",
        "Backdoor:EC2/DenialOfService.Dns",
        "Backdoor:EC2/Spambot"
    ],
    "Behavior": [
        "Behavior:EC2/NetworkPortUnusual",
        "Behavior:EC2/TrafficVolumeUnusual"
    ],
    "CryptoCurrency": [
        "CryptoCurrency:EC2/BitcoinTool.B!DNS",
        "CryptoCurrency:EC2/BitcoinTool.B"
    ],
    "Malware": [
        "Malware:EC2/SuspiciousFile",
        "Malware:Runtime/SuspiciousFile"
    ],
    "Persistence": [
        "Persistence:IAMUser/NetworkPermissions",
        "Persistence:IAMUser/ResourcePermissions"
    ],
    "Policy": [
        "Policy:IAMUser/RootCredentialUsage",
        "Policy:S3/BucketBlockPublicAccessDisabled"
    ],
    "PrivilegeEscalation": [
        "PrivilegeEscalation:IAMUser/AdministrativePermissions",
        "PrivilegeEscalation:Kubernetes/PrivilegedContainer"
    ],
    "Recon": [
        "Recon:EC2/PortProbeUnprotectedPort",
        "Recon:IAMUser/NetworkPermissions"
    ],
    "ResourceConsumption": [
        "ResourceConsumption:IAMUser/ComputeResources",
        "ResourceConsumption:IAMUser/ComputeResources"
    ],
    "Stealth": [
        "Stealth:IAMUser/CloudTrailLoggingDisabled",
        "Stealth:S3/ServerAccessLoggingDisabled"
    ],
    "Trojan": [
        "Trojan:EC2/BlackholeTraffic",
        "Trojan:EC2/DropPoint"
    ],
    "UnauthorizedAccess": [
        "UnauthorizedAccess:EC2/SSHBruteForce",
        "UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B"
    ]
}
```

### Finding Analysis and Processing
```python
import boto3
import json
from datetime import datetime, timedelta

class GuardDutyAnalyzer:
    def __init__(self):
        self.guardduty_client = boto3.client('guardduty')
        self.sns_client = boto3.client('sns')
        self.lambda_client = boto3.client('lambda')
        self.ec2_client = boto3.client('ec2')
    
    def get_detector_id(self):
        """Get the GuardDuty detector ID"""
        detectors = self.guardduty_client.list_detectors()
        if detectors['DetectorIds']:
            return detectors['DetectorIds'][0]
        return None
    
    def analyze_findings(self, hours_back=24):
        """Analyze GuardDuty findings from the last N hours"""
        
        detector_id = self.get_detector_id()
        if not detector_id:
            return {'error': 'No GuardDuty detector found'}
        
        # Calculate time range
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours_back)
        
        # Get findings
        findings_response = self.guardduty_client.list_findings(
            DetectorId=detector_id,
            FindingCriteria={
                'Criterion': {
                    'updatedAt': {
                        'Gte': int(start_time.timestamp() * 1000),
                        'Lte': int(end_time.timestamp() * 1000)
                    }
                }
            }
        )
        
        if not findings_response['FindingIds']:
            return {'message': 'No findings in the specified time range'}
        
        # Get detailed finding information
        findings_details = self.guardduty_client.get_findings(
            DetectorId=detector_id,
            FindingIds=findings_response['FindingIds']
        )
        
        # Analyze findings
        analysis = {
            'total_findings': len(findings_details['Findings']),
            'severity_breakdown': {'Low': 0, 'Medium': 0, 'High': 0},
            'type_breakdown': {},
            'affected_resources': set(),
            'critical_findings': [],
            'recommendations': []
        }
        
        for finding in findings_details['Findings']:
            # Count by severity
            severity = self.get_severity_label(finding['Severity'])
            analysis['severity_breakdown'][severity] += 1
            
            # Count by type
            finding_type = finding['Type']
            analysis['type_breakdown'][finding_type] = analysis['type_breakdown'].get(finding_type, 0) + 1
            
            # Track affected resources
            if 'Resource' in finding:
                resource_type = finding['Resource']['ResourceType']
                analysis['affected_resources'].add(resource_type)
            
            # Identify critical findings
            if finding['Severity'] >= 7.0:  # High severity
                analysis['critical_findings'].append({
                    'id': finding['Id'],
                    'type': finding['Type'],
                    'severity': finding['Severity'],
                    'title': finding['Title'],
                    'description': finding['Description'],
                    'resource': finding.get('Resource', {})
                })
        
        # Convert set to list for JSON serialization
        analysis['affected_resources'] = list(analysis['affected_resources'])
        
        # Generate recommendations
        analysis['recommendations'] = self.generate_recommendations(analysis)
        
        return analysis
    
    def get_severity_label(self, severity_score):
        """Convert severity score to label"""
        if severity_score >= 7.0:
            return 'High'
        elif severity_score >= 4.0:
            return 'Medium'
        else:
            return 'Low'
    
    def generate_recommendations(self, analysis):
        """Generate security recommendations based on findings"""
        
        recommendations = []
        
        # Check for high-severity findings
        if analysis['critical_findings']:
            recommendations.append({
                'priority': 'CRITICAL',
                'action': 'Immediate investigation required for high-severity findings',
                'details': f"Found {len(analysis['critical_findings'])} critical security findings"
            })
        
        # Check for specific threat types
        for finding_type, count in analysis['type_breakdown'].items():
            if 'CryptoCurrency' in finding_type:
                recommendations.append({
                    'priority': 'HIGH',
                    'action': 'Investigate cryptocurrency mining activity',
                    'details': f"Detected {count} cryptocurrency-related findings"
                })
            
            elif 'Backdoor' in finding_type:
                recommendations.append({
                    'priority': 'CRITICAL',
                    'action': 'Investigate potential backdoor or C&C communication',
                    'details': f"Detected {count} backdoor-related findings"
                })
            
            elif 'UnauthorizedAccess' in finding_type:
                recommendations.append({
                    'priority': 'HIGH',
                    'action': 'Review access controls and authentication mechanisms',
                    'details': f"Detected {count} unauthorized access attempts"
                })
        
        return recommendations
    
    def create_automated_response(self, finding):
        """Create automated response for specific finding types"""
        
        finding_type = finding['Type']
        resource = finding.get('Resource', {})
        
        response_actions = []
        
        # Automated responses based on finding type
        if 'CryptoCurrency' in finding_type and resource.get('ResourceType') == 'Instance':
            # Isolate instance for cryptocurrency mining
            instance_id = resource['InstanceDetails']['InstanceId']
            response_actions.append(self.isolate_instance(instance_id))
        
        elif 'UnauthorizedAccess:EC2/SSHBruteForce' in finding_type:
            # Block source IP in security group
            source_ip = finding['Service']['RemoteIpDetails']['IpAddressV4']
            response_actions.append(self.block_ip_address(source_ip, resource))
        
        elif 'Stealth:IAMUser/CloudTrailLoggingDisabled' in finding_type:
            # Re-enable CloudTrail logging
            response_actions.append(self.enable_cloudtrail_logging())
        
        elif 'Policy:S3/BucketBlockPublicAccessDisabled' in finding_type:
            # Enable S3 bucket public access block
            bucket_name = resource['S3BucketDetails'][0]['Name']
            response_actions.append(self.enable_s3_public_access_block(bucket_name))
        
        return response_actions
    
    def isolate_instance(self, instance_id):
        """Isolate EC2 instance by modifying security groups"""
        
        try:
            # Create isolation security group
            isolation_sg = self.ec2_client.create_security_group(
                GroupName=f'isolation-{instance_id}',
                Description='Isolation security group for compromised instance'
            )
            
            # Modify instance security groups
            self.ec2_client.modify_instance_attribute(
                InstanceId=instance_id,
                Groups=[isolation_sg['GroupId']]
            )
            
            return {
                'action': 'Instance Isolation',
                'status': 'SUCCESS',
                'details': f'Instance {instance_id} isolated with security group {isolation_sg["GroupId"]}'
            }
        
        except Exception as e:
            return {
                'action': 'Instance Isolation',
                'status': 'FAILED',
                'error': str(e)
            }
    
    def block_ip_address(self, ip_address, resource):
        """Block IP address in security group"""
        
        try:
            instance_id = resource['InstanceDetails']['InstanceId']
            
            # Get instance security groups
            instance_response = self.ec2_client.describe_instances(InstanceIds=[instance_id])
            security_groups = instance_response['Reservations'][0]['Instances'][0]['SecurityGroups']
            
            # Add deny rule to security groups
            for sg in security_groups:
                self.ec2_client.authorize_security_group_ingress(
                    GroupId=sg['GroupId'],
                    IpPermissions=[
                        {
                            'IpProtocol': '-1',
                            'IpRanges': [
                                {
                                    'CidrIp': f'{ip_address}/32',
                                    'Description': f'GuardDuty auto-block for {ip_address}'
                                }
                            ]
                        }
                    ]
                )
            
            return {
                'action': 'IP Address Block',
                'status': 'SUCCESS',
                'details': f'Blocked IP {ip_address} in security groups'
            }
        
        except Exception as e:
            return {
                'action': 'IP Address Block',
                'status': 'FAILED',
                'error': str(e)
            }
```

---

## 4. Automated Response and Remediation

### Lambda-based Automated Response
```python
# Lambda function for GuardDuty automated response
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    """
    Automated response to GuardDuty findings
    Triggered by CloudWatch Events when GuardDuty creates findings
    """
    
    # Parse GuardDuty finding from CloudWatch Event
    finding = event['detail']
    
    finding_type = finding['type']
    severity = finding['severity']
    resource = finding.get('resource', {})
    
    print(f"Processing GuardDuty finding: {finding_type} (Severity: {severity})")
    
    # Initialize AWS clients
    ec2 = boto3.client('ec2')
    iam = boto3.client('iam')
    sns = boto3.client('sns')
    ssm = boto3.client('ssm')
    
    response_actions = []
    
    try:
        # High-severity findings require immediate action
        if severity >= 7.0:
            response_actions.extend(handle_critical_finding(finding, ec2, iam, ssm))
        
        # Medium-severity findings require investigation
        elif severity >= 4.0:
            response_actions.extend(handle_medium_finding(finding, ec2, sns))
        
        # Low-severity findings are logged for analysis
        else:
            response_actions.append(log_finding_for_analysis(finding))
        
        # Send notification
        send_notification(finding, response_actions, sns)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'finding_id': finding['id'],
                'actions_taken': len(response_actions),
                'response_actions': response_actions
            })
        }
    
    except Exception as e:
        print(f"Error processing GuardDuty finding: {str(e)}")
        
        # Send error notification
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
            Message=f"Error processing GuardDuty finding {finding['id']}: {str(e)}",
            Subject='GuardDuty Automated Response Error'
        )
        
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def handle_critical_finding(finding, ec2, iam, ssm):
    """Handle critical severity findings with immediate automated response"""
    
    actions = []
    finding_type = finding['type']
    resource = finding.get('resource', {})
    
    # Cryptocurrency mining detection
    if 'CryptoCurrency' in finding_type:
        if resource.get('resourceType') == 'Instance':
            instance_id = resource['instanceDetails']['instanceId']
            
            # Stop the instance immediately
            try:
                ec2.stop_instances(InstanceIds=[instance_id])
                actions.append({
                    'action': 'Stop Instance',
                    'resource': instance_id,
                    'status': 'SUCCESS',
                    'reason': 'Cryptocurrency mining detected'
                })
            except Exception as e:
                actions.append({
                    'action': 'Stop Instance',
                    'resource': instance_id,
                    'status': 'FAILED',
                    'error': str(e)
                })
    
    # Backdoor or C&C communication
    elif 'Backdoor' in finding_type:
        if resource.get('resourceType') == 'Instance':
            instance_id = resource['instanceDetails']['instanceId']
            
            # Isolate instance by creating restrictive security group
            try:
                isolation_sg = create_isolation_security_group(ec2, instance_id)
                
                ec2.modify_instance_attribute(
                    InstanceId=instance_id,
                    Groups=[isolation_sg]
                )
                
                actions.append({
                    'action': 'Isolate Instance',
                    'resource': instance_id,
                    'security_group': isolation_sg,
                    'status': 'SUCCESS',
                    'reason': 'Backdoor communication detected'
                })
            except Exception as e:
                actions.append({
                    'action': 'Isolate Instance',
                    'resource': instance_id,
                    'status': 'FAILED',
                    'error': str(e)
                })
    
    # Compromised IAM credentials
    elif 'UnauthorizedAccess:IAMUser' in finding_type:
        user_name = resource.get('accessKeyDetails', {}).get('userName')
        
        if user_name:
            try:
                # Disable user's access keys
                access_keys = iam.list_access_keys(UserName=user_name)
                
                for key in access_keys['AccessKeyMetadata']:
                    iam.update_access_key(
                        UserName=user_name,
                        AccessKeyId=key['AccessKeyId'],
                        Status='Inactive'
                    )
                
                actions.append({
                    'action': 'Disable Access Keys',
                    'resource': user_name,
                    'status': 'SUCCESS',
                    'reason': 'Unauthorized access detected'
                })
            except Exception as e:
                actions.append({
                    'action': 'Disable Access Keys',
                    'resource': user_name,
                    'status': 'FAILED',
                    'error': str(e)
                })
    
    # CloudTrail logging disabled
    elif 'Stealth:IAMUser/CloudTrailLoggingDisabled' in finding_type:
        try:
            # Re-enable CloudTrail logging via Systems Manager
            ssm.send_command(
                DocumentName='AWS-ConfigureCloudTrail',
                Parameters={
                    'trailName': ['SecurityAuditTrail'],
                    's3BucketName': ['security-audit-logs'],
                    'includeGlobalServiceEvents': ['true'],
                    'isMultiRegionTrail': ['true']
                },
                Targets=[
                    {
                        'Key': 'tag:Role',
                        'Values': ['SecurityManagement']
                    }
                ]
            )
            
            actions.append({
                'action': 'Re-enable CloudTrail',
                'status': 'SUCCESS',
                'reason': 'CloudTrail logging was disabled'
            })
        except Exception as e:
            actions.append({
                'action': 'Re-enable CloudTrail',
                'status': 'FAILED',
                'error': str(e)
            })
    
    return actions

def handle_medium_finding(finding, ec2, sns):
    """Handle medium severity findings with investigation and monitoring"""
    
    actions = []
    finding_type = finding['type']
    
    # SSH brute force attacks
    if 'UnauthorizedAccess:EC2/SSHBruteForce' in finding_type:
        source_ip = finding['service']['remoteIpDetails']['ipAddressV4']
        
        # Add IP to threat intelligence for monitoring
        actions.append({
            'action': 'Monitor IP Address',
            'ip_address': source_ip,
            'status': 'SUCCESS',
            'reason': 'SSH brute force detected'
        })
        
        # Send detailed alert
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:security-investigations',
            Message=f"SSH brute force attack detected from {source_ip}. Please investigate and consider blocking this IP.",
            Subject='Security Investigation Required - SSH Brute Force'
        )
    
    # Unusual network behavior
    elif 'Behavior:EC2' in finding_type:
        instance_id = finding['resource']['instanceDetails']['instanceId']
        
        # Enable detailed monitoring
        try:
            ec2.monitor_instances(InstanceIds=[instance_id])
            
            actions.append({
                'action': 'Enable Detailed Monitoring',
                'resource': instance_id,
                'status': 'SUCCESS',
                'reason': 'Unusual network behavior detected'
            })
        except Exception as e:
            actions.append({
                'action': 'Enable Detailed Monitoring',
                'resource': instance_id,
                'status': 'FAILED',
                'error': str(e)
            })
    
    return actions

def create_isolation_security_group(ec2, instance_id):
    """Create isolation security group for compromised instances"""
    
    # Get VPC ID from instance
    instance_response = ec2.describe_instances(InstanceIds=[instance_id])
    vpc_id = instance_response['Reservations'][0]['Instances'][0]['VpcId']
    
    # Create isolation security group
    sg_response = ec2.create_security_group(
        GroupName=f'isolation-{instance_id}-{int(datetime.now().timestamp())}',
        Description=f'Isolation security group for compromised instance {instance_id}',
        VpcId=vpc_id
    )
    
    # Add minimal egress rules (only for investigation)
    ec2.authorize_security_group_egress(
        GroupId=sg_response['GroupId'],
        IpPermissions=[
            {
                'IpProtocol': 'tcp',
                'FromPort': 443,
                'ToPort': 443,
                'IpRanges': [{'CidrIp': '0.0.0.0/0', 'Description': 'HTTPS for investigation tools'}]
            }
        ]
    )
    
    return sg_response['GroupId']

def send_notification(finding, actions, sns):
    """Send notification about GuardDuty finding and response actions"""
    
    message = f"""
    GuardDuty Security Finding
    =========================
    
    Finding ID: {finding['id']}
    Type: {finding['type']}
    Severity: {finding['severity']}
    Title: {finding['title']}
    Description: {finding['description']}
    
    Automated Response Actions:
    """
    
    for action in actions:
        message += f"\n- {action['action']}: {action['status']}"
        if action['status'] == 'FAILED':
            message += f" (Error: {action.get('error', 'Unknown')})"
    
    message += f"\n\nTimestamp: {datetime.utcnow().isoformat()}"
    
    # Determine topic based on severity
    if finding['severity'] >= 7.0:
        topic_arn = 'arn:aws:sns:us-east-1:123456789012:critical-security-alerts'
        subject = f"CRITICAL: GuardDuty Finding - {finding['type']}"
    else:
        topic_arn = 'arn:aws:sns:us-east-1:123456789012:security-alerts'
        subject = f"GuardDuty Finding - {finding['type']}"
    
    sns.publish(
        TopicArn=topic_arn,
        Message=message,
        Subject=subject
    )
```

### CloudWatch Events Integration
```yaml
# CloudFormation for GuardDuty event processing
Resources:
  GuardDutyEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: GuardDutyFindingProcessor
      Description: Process GuardDuty findings for automated response
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
        detail:
          severity:
            - numeric:
                - ">="
                - 4.0
      State: ENABLED
      Targets:
        - Arn: !GetAtt GuardDutyResponseFunction.Arn
          Id: GuardDutyResponseTarget
        - Arn: !Ref SecurityAlertsTopic
          Id: SecurityAlertsTarget
          InputTransformer:
            InputPathsMap:
              severity: "$.detail.severity"
              type: "$.detail.type"
              title: "$.detail.title"
            InputTemplate: |
              {
                "severity": "<severity>",
                "finding_type": "<type>",
                "title": "<title>",
                "message": "GuardDuty finding requires attention"
              }

  # High-severity findings trigger immediate response
  CriticalFindingRule:
    Type: AWS::Events::Rule
    Properties:
      Name: GuardDutyCriticalFindings
      Description: Process critical GuardDuty findings
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
        detail:
          severity:
            - numeric:
                - ">="
                - 7.0
      State: ENABLED
      Targets:
        - Arn: !GetAtt CriticalResponseFunction.Arn
          Id: CriticalResponseTarget
        - Arn: !Ref CriticalAlertsTopic
          Id: CriticalAlertsTarget

  # Lambda permission for EventBridge
  GuardDutyLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref GuardDutyResponseFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt GuardDutyEventRule.Arn
```

---

## 5. Integration with Security Hub

### Security Hub Integration
```yaml
Resources:
  SecurityHubIntegration:
    Type: AWS::SecurityHub::Hub
    Properties:
      Tags:
        - Key: Purpose
          Value: CentralizedSecurityMonitoring

  # Enable GuardDuty integration with Security Hub
  GuardDutyProductSubscription:
    Type: AWS::SecurityHub::ProductSubscription
    Properties:
      ProductArn: !Sub "arn:aws:securityhub:${AWS::Region}::product/aws/guardduty"
    DependsOn: SecurityHubIntegration

  # Custom insight for GuardDuty findings
  GuardDutyInsight:
    Type: AWS::SecurityHub::Insight
    Properties:
      Name: GuardDuty High Severity Findings
      Description: High severity findings from GuardDuty
      GroupByAttribute: ProductFields/aws/guardduty/service/resourceRole
      Filters:
        ProductName:
          - Value: GuardDuty
            Comparison: EQUALS
        SeverityLabel:
          - Value: HIGH
            Comparison: EQUALS
          - Value: CRITICAL
            Comparison: EQUALS
        RecordState:
          - Value: ACTIVE
            Comparison: EQUALS
```

---

## 6. Malware Protection

### EBS Volume Scanning
```yaml
Resources:
  MalwareProtectionConfiguration:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: true
      DataSources:
        MalwareProtection:
          ScanEc2InstanceWithFindings:
            EbsVolumes: true

  # Lambda function to handle malware findings
  MalwareResponseFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: guardduty-malware-response
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt MalwareResponseRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          
          def lambda_handler(event, context):
              """Handle GuardDuty malware detection findings"""
              
              finding = event['detail']
              
              if 'Malware' in finding['type']:
                  ec2 = boto3.client('ec2')
                  ssm = boto3.client('ssm')
                  
                  instance_id = finding['resource']['instanceDetails']['instanceId']
                  
                  # Immediate isolation
                  isolate_instance(ec2, instance_id)
                  
                  # Create forensic snapshot
                  create_forensic_snapshot(ec2, instance_id)
                  
                  # Run malware removal if safe
                  if finding['service']['additionalInfo']['threatListName'] != 'CRITICAL':
                      run_malware_removal(ssm, instance_id)
              
              return {'statusCode': 200}
          
          def isolate_instance(ec2, instance_id):
              """Isolate infected instance"""
              
              # Create isolation security group
              vpc_response = ec2.describe_instances(InstanceIds=[instance_id])
              vpc_id = vpc_response['Reservations'][0]['Instances'][0]['VpcId']
              
              sg_response = ec2.create_security_group(
                  GroupName=f'malware-isolation-{instance_id}',
                  Description='Isolation for malware-infected instance',
                  VpcId=vpc_id
              )
              
              # Apply isolation security group
              ec2.modify_instance_attribute(
                  InstanceId=instance_id,
                  Groups=[sg_response['GroupId']]
              )
          
          def create_forensic_snapshot(ec2, instance_id):
              """Create forensic snapshot of infected instance"""
              
              # Get instance volumes
              instance_response = ec2.describe_instances(InstanceIds=[instance_id])
              volumes = []
              
              for reservation in instance_response['Reservations']:
                  for instance in reservation['Instances']:
                      for bdm in instance['BlockDeviceMappings']:
                          volumes.append(bdm['Ebs']['VolumeId'])
              
              # Create snapshots
              for volume_id in volumes:
                  ec2.create_snapshot(
                      VolumeId=volume_id,
                      Description=f'Forensic snapshot for malware investigation - {instance_id}',
                      TagSpecifications=[
                          {
                              'ResourceType': 'snapshot',
                              'Tags': [
                                  {'Key': 'Purpose', 'Value': 'Forensic'},
                                  {'Key': 'SourceInstance', 'Value': instance_id},
                                  {'Key': 'Reason', 'Value': 'Malware Detection'}
                              ]
                          }
                      ]
                  )
          
          def run_malware_removal(ssm, instance_id):
              """Run malware removal tools"""
              
              ssm.send_command(
                  InstanceIds=[instance_id],
                  DocumentName='AWS-RunShellScript',
                  Parameters={
                      'commands': [
                          '#!/bin/bash',
                          'yum update -y',
                          'yum install -y clamav clamav-update',
                          'freshclam',
                          'clamscan -r --remove /home /tmp /var/tmp',
                          'echo "Malware scan completed"'
                      ]
                  }
              )

  MalwareEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: GuardDutyMalwareFindings
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
        detail:
          type:
            - prefix: "Malware:"
      Targets:
        - Arn: !GetAtt MalwareResponseFunction.Arn
          Id: MalwareResponseTarget
```

---

## 7. Common Exam Scenarios

### Scenario 1: Multi-Account GuardDuty Setup
```python
# Complete multi-account GuardDuty setup
import boto3
import json

class MultiAccountGuardDutySetup:
    def __init__(self):
        self.guardduty_client = boto3.client('guardduty')
        self.organizations_client = boto3.client('organizations')
        self.sts_client = boto3.client('sts')
    
    def setup_organization_guardduty(self):
        """Setup GuardDuty across AWS Organization"""
        
        # Enable GuardDuty in master account
        master_detector = self.enable_guardduty_master()
        
        # Get organization accounts
        accounts = self.get_organization_accounts()
        
        # Enable GuardDuty in member accounts
        member_results = {}
        
        for account in accounts:
            if account['Status'] == 'ACTIVE' and account['Id'] != self.get_master_account_id():
                result = self.enable_guardduty_member(account['Id'], account['Email'])
                member_results[account['Id']] = result
        
        # Configure organization settings
        org_config = self.configure_organization_settings(master_detector['DetectorId'])
        
        return {
            'master_detector': master_detector,
            'member_results': member_results,
            'organization_config': org_config
        }
    
    def enable_guardduty_master(self):
        """Enable GuardDuty in master account"""
        
        try:
            # Check if detector already exists
            detectors = self.guardduty_client.list_detectors()
            
            if detectors['DetectorIds']:
                detector_id = detectors['DetectorIds'][0]
                
                # Update detector configuration
                self.guardduty_client.update_detector(
                    DetectorId=detector_id,
                    Enable=True,
                    FindingPublishingFrequency='FIFTEEN_MINUTES',
                    DataSources={
                        'S3Logs': {'Enable': True},
                        'KubernetesConfiguration': {
                            'AuditLogs': {'Enable': True}
                        },
                        'MalwareProtection': {
                            'ScanEc2InstanceWithFindings': {
                                'EbsVolumes': True
                            }
                        }
                    }
                )
            else:
                # Create new detector
                response = self.guardduty_client.create_detector(
                    Enable=True,
                    FindingPublishingFrequency='FIFTEEN_MINUTES',
                    DataSources={
                        'S3Logs': {'Enable': True},
                        'KubernetesConfiguration': {
                            'AuditLogs': {'Enable': True}
                        },
                        'MalwareProtection': {
                            'ScanEc2InstanceWithFindings': {
                                'EbsVolumes': True
                            }
                        }
                    }
                )
                detector_id = response['DetectorId']
            
            return {
                'DetectorId': detector_id,
                'Status': 'SUCCESS'
            }
        
        except Exception as e:
            return {
                'Status': 'FAILED',
                'Error': str(e)
            }
    
    def enable_guardduty_member(self, account_id, email):
        """Enable GuardDuty in member account"""
        
        try:
            detector_id = self.guardduty_client.list_detectors()['DetectorIds'][0]
            
            # Create member
            self.guardduty_client.create_members(
                DetectorId=detector_id,
                AccountDetails=[
                    {
                        'AccountId': account_id,
                        'Email': email
                    }
                ]
            )
            
            # Invite member
            self.guardduty_client.invite_members(
                DetectorId=detector_id,
                AccountIds=[account_id],
                Message='Please accept GuardDuty invitation for centralized security monitoring'
            )
            
            return {
                'Status': 'INVITED',
                'AccountId': account_id
            }
        
        except Exception as e:
            return {
                'Status': 'FAILED',
                'AccountId': account_id,
                'Error': str(e)
            }
    
    def configure_organization_settings(self, detector_id):
        """Configure organization-wide GuardDuty settings"""
        
        try:
            # Enable organization configuration
            self.guardduty_client.update_organization_configuration(
                DetectorId=detector_id,
                AutoEnable=True,
                DataSources={
                    'S3Logs': {'AutoEnable': True},
                    'KubernetesConfiguration': {
                        'AuditLogs': {'AutoEnable': True}
                    },
                    'MalwareProtection': {
                        'ScanEc2InstanceWithFindings': {
                            'EbsVolumes': {'AutoEnable': True}
                        }
                    }
                }
            )
            
            return {
                'Status': 'SUCCESS',
                'AutoEnable': True
            }
        
        except Exception as e:
            return {
                'Status': 'FAILED',
                'Error': str(e)
            }
    
    def get_organization_accounts(self):
        """Get all accounts in the organization"""
        
        accounts = []
        paginator = self.organizations_client.get_paginator('list_accounts')
        
        for page in paginator.paginate():
            accounts.extend(page['Accounts'])
        
        return accounts
    
    def get_master_account_id(self):
        """Get master account ID"""
        
        return self.sts_client.get_caller_identity()['Account']
```

### Scenario 2: Threat Intelligence Integration
```yaml
# Threat intelligence integration setup
Resources:
  ThreatIntelligenceBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-threat-intel"
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

  # Lambda function to update threat intelligence
  ThreatIntelUpdateFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: guardduty-threat-intel-updater
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt ThreatIntelRole.Arn
      Code:
        ZipFile: |
          import boto3
          import requests
          import json
          
          def lambda_handler(event, context):
              """Update GuardDuty threat intelligence from external sources"""
              
              s3 = boto3.client('s3')
              guardduty = boto3.client('guardduty')
              
              # Download threat intelligence feeds
              threat_feeds = [
                  'https://rules.emergingthreats.net/open/suricata/rules/compromised-ips.txt',
                  'https://www.spamhaus.org/drop/drop.txt'
              ]
              
              combined_threats = []
              
              for feed_url in threat_feeds:
                  try:
                      response = requests.get(feed_url, timeout=30)
                      if response.status_code == 200:
                          # Parse threat indicators
                          threats = parse_threat_feed(response.text, feed_url)
                          combined_threats.extend(threats)
                  except Exception as e:
                      print(f"Error downloading {feed_url}: {str(e)}")
              
              # Upload to S3
              threat_intel_content = '\n'.join(combined_threats)
              
              s3.put_object(
                  Bucket=os.environ['THREAT_INTEL_BUCKET'],
                  Key='threat-intel.txt',
                  Body=threat_intel_content,
                  ContentType='text/plain'
              )
              
              # Update GuardDuty threat intelligence set
              detector_id = guardduty.list_detectors()['DetectorIds'][0]
              
              try:
                  guardduty.update_threat_intel_set(
                      DetectorId=detector_id,
                      ThreatIntelSetId=os.environ['THREAT_INTEL_SET_ID'],
                      Location=f"https://{os.environ['THREAT_INTEL_BUCKET']}.s3.amazonaws.com/threat-intel.txt",
                      Activate=True
                  )
              except Exception as e:
                  print(f"Error updating threat intelligence set: {str(e)}")
              
              return {
                  'statusCode': 200,
                  'body': json.dumps(f'Updated {len(combined_threats)} threat indicators')
              }
          
          def parse_threat_feed(content, source):
              """Parse threat intelligence feed content"""
              
              threats = []
              
              for line in content.split('\n'):
                  line = line.strip()
                  
                  # Skip comments and empty lines
                  if not line or line.startswith('#') or line.startswith(';'):
                      continue
                  
                  # Extract IP addresses (simple regex)
                  import re
                  ip_pattern = r'\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b'
                  
                  matches = re.findall(ip_pattern, line)
                  for ip in matches:
                      if is_valid_ip(ip):
                          threats.append(ip)
              
              return threats
          
          def is_valid_ip(ip):
              """Validate IP address"""
              
              try:
                  parts = ip.split('.')
                  return len(parts) == 4 and all(0 <= int(part) <= 255 for part in parts)
              except:
                  return False
      Environment:
        Variables:
          THREAT_INTEL_BUCKET: !Ref ThreatIntelligenceBucket
          THREAT_INTEL_SET_ID: !Ref ThreatIntelSet

  # Schedule threat intelligence updates
  ThreatIntelUpdateSchedule:
    Type: AWS::Events::Rule
    Properties:
      Description: Update threat intelligence daily
      ScheduleExpression: rate(1 day)
      State: ENABLED
      Targets:
        - Arn: !GetAtt ThreatIntelUpdateFunction.Arn
          Id: ThreatIntelUpdateTarget
```

---

## 8. Exam Tips

- **Understand finding types** - Know major categories and what they indicate
- **Master automated response** - Lambda integration with CloudWatch Events
- **Know multi-account setup** - Organizations integration and member management
- **Practice threat intelligence** - Custom threat intel sets and IP sets
- **Learn malware protection** - EBS volume scanning and response workflows
- **Understand data sources** - VPC Flow Logs, DNS logs, CloudTrail integration
- **Know Security Hub integration** - Centralized security findings management
- **Practice cost optimization** - Finding frequency, data source configuration
- **Master troubleshooting** - Common issues with detectors and member accounts
- **Understand compliance** - SOC, PCI DSS, and other compliance frameworks