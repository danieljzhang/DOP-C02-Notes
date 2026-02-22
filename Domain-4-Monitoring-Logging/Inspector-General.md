# AWS Inspector - DOP-C02 Exam Notes

## 1. Overview

**AWS Inspector** is an automated security assessment service that helps improve the security and compliance of applications deployed on AWS. It automatically assesses applications for exposure, vulnerabilities, and deviations from best practices.

### Key Characteristics
- **Automated vulnerability assessment** - Continuous scanning of EC2 instances and container images
- **Network reachability analysis** - Identifies network paths that allow access to instances
- **Software vulnerability assessment** - Scans for known vulnerabilities in installed packages
- **Security best practices** - Checks against CIS benchmarks and security standards
- **Integration with CI/CD** - Automated scanning in deployment pipelines
- **Centralized findings** - Integration with Security Hub for unified security management
- **Continuous monitoring** - Ongoing assessment as infrastructure changes

### What Problem Does It Solve?
- Identifies security vulnerabilities before they can be exploited
- Provides continuous security assessment without manual intervention
- Ensures compliance with security standards and best practices
- Integrates security scanning into DevOps workflows
- Reduces time to identify and remediate security issues
- Provides detailed remediation guidance for found vulnerabilities

---

## 2. Inspector V2 (Current Version)

### Assessment Types
```yaml
# CloudFormation for Inspector V2 setup
Resources:
  InspectorConfiguration:
    Type: AWS::Inspector2::EnabledConfiguration
    Properties:
      AccountId: !Ref AWS::AccountId
      ResourceTypes:
        - EC2
        - ECR
      AutoEnable:
        Ec2: true
        Ecr: true

  # Organization-wide Inspector setup
  InspectorOrganizationConfiguration:
    Type: AWS::Inspector2::OrganizationConfiguration
    Properties:
      AutoEnable:
        Ec2: true
        Ecr: true
```

### Programmatic Configuration
```python
import boto3
import json
from datetime import datetime, timedelta

class InspectorManager:
    def __init__(self):
        self.inspector2_client = boto3.client('inspector2')
        self.ec2_client = boto3.client('ec2')
        self.ecr_client = boto3.client('ecr')
        self.organizations_client = boto3.client('organizations')
    
    def enable_inspector(self, resource_types=['EC2', 'ECR']):
        """Enable Inspector for specified resource types"""
        
        try:
            response = self.inspector2_client.enable(
                resourceTypes=resource_types,
                accountIds=[boto3.Session().get_credentials().access_key[:12]]
            )
            
            return {
                'status': 'SUCCESS',
                'enabled_resources': resource_types,
                'accounts': response.get('accounts', [])
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'error': str(e)
            }
    
    def get_findings_summary(self, days_back=7):
        """Get summary of Inspector findings"""
        
        # Calculate time range
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=days_back)
        
        try:
            # Get findings
            findings_response = self.inspector2_client.list_findings(
                filterCriteria={
                    'firstObservedAt': [
                        {
                            'startInclusive': start_time,
                            'endInclusive': end_time
                        }
                    ]
                },
                maxResults=1000
            )
            
            # Analyze findings
            summary = {
                'total_findings': len(findings_response['findings']),
                'severity_breakdown': {
                    'CRITICAL': 0,
                    'HIGH': 0,
                    'MEDIUM': 0,
                    'LOW': 0,
                    'INFORMATIONAL': 0
                },
                'resource_breakdown': {
                    'EC2': 0,
                    'ECR': 0
                },
                'top_vulnerabilities': {},
                'affected_resources': set()
            }
            
            for finding in findings_response['findings']:
                # Count by severity
                severity = finding.get('severity', 'INFORMATIONAL')
                summary['severity_breakdown'][severity] += 1
                
                # Count by resource type
                resource_type = finding.get('type', 'UNKNOWN')
                if 'PACKAGE_VULNERABILITY' in resource_type:
                    if 'ec2-instance' in finding.get('resources', [{}])[0].get('id', ''):
                        summary['resource_breakdown']['EC2'] += 1
                    else:
                        summary['resource_breakdown']['ECR'] += 1
                
                # Track vulnerabilities
                vuln_id = finding.get('packageVulnerabilityDetails', {}).get('vulnerabilityId', 'Unknown')
                summary['top_vulnerabilities'][vuln_id] = summary['top_vulnerabilities'].get(vuln_id, 0) + 1
                
                # Track affected resources
                for resource in finding.get('resources', []):
                    summary['affected_resources'].add(resource.get('id', 'Unknown'))
            
            # Convert set to list for JSON serialization
            summary['affected_resources'] = list(summary['affected_resources'])
            
            # Get top 10 vulnerabilities
            summary['top_vulnerabilities'] = dict(
                sorted(summary['top_vulnerabilities'].items(), 
                      key=lambda x: x[1], reverse=True)[:10]
            )
            
            return summary
        
        except Exception as e:
            return {
                'error': str(e),
                'status': 'FAILED'
            }
    
    def get_vulnerability_details(self, vulnerability_id):
        """Get detailed information about a specific vulnerability"""
        
        try:
            findings_response = self.inspector2_client.list_findings(
                filterCriteria={
                    'packageVulnerabilityDetails': {
                        'vulnerabilityId': [vulnerability_id]
                    }
                }
            )
            
            if not findings_response['findings']:
                return {'error': 'Vulnerability not found'}
            
            finding = findings_response['findings'][0]
            vuln_details = finding.get('packageVulnerabilityDetails', {})
            
            return {
                'vulnerability_id': vulnerability_id,
                'severity': finding.get('severity'),
                'title': finding.get('title'),
                'description': finding.get('description'),
                'cvss_score': vuln_details.get('cvss', {}).get('baseScore'),
                'cvss_vector': vuln_details.get('cvss', {}).get('scoringVector'),
                'affected_packages': [
                    {
                        'name': vuln_details.get('vulnerablePackages', [{}])[0].get('name'),
                        'version': vuln_details.get('vulnerablePackages', [{}])[0].get('version'),
                        'fixed_version': vuln_details.get('vulnerablePackages', [{}])[0].get('fixedInVersion')
                    }
                ],
                'references': vuln_details.get('referenceUrls', []),
                'affected_resources': [resource.get('id') for resource in finding.get('resources', [])]
            }
        
        except Exception as e:
            return {
                'error': str(e),
                'status': 'FAILED'
            }
    
    def create_suppression_rule(self, rule_name, filter_criteria, reason):
        """Create suppression rule for specific findings"""
        
        try:
            response = self.inspector2_client.create_filter(
                name=rule_name,
                description=f"Suppression rule: {reason}",
                filterCriteria=filter_criteria,
                action='SUPPRESS',
                reason=reason
            )
            
            return {
                'status': 'SUCCESS',
                'filter_arn': response['arn']
            }
        
        except Exception as e:
            return {
                'status': 'FAILED',
                'error': str(e)
            }
```

---

## 3. EC2 Instance Assessment

### Instance Configuration
```yaml
# CloudFormation for Inspector-ready EC2 instances
Resources:
  InspectorRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: InspectorEC2Role
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
        - PolicyName: InspectorAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - inspector2:ListFindings
                  - inspector2:GetMember
                Resource: '*'

  InspectorInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref InspectorRole

  # EC2 instance with Inspector scanning enabled
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890  # Amazon Linux 2
      InstanceType: t3.medium
      IamInstanceProfile: !Ref InspectorInstanceProfile
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      SubnetId: !Ref PrivateSubnet
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          
          # Install SSM Agent (usually pre-installed on Amazon Linux 2)
          yum install -y amazon-ssm-agent
          systemctl enable amazon-ssm-agent
          systemctl start amazon-ssm-agent
          
          # Install Inspector agent (for Inspector Classic - not needed for Inspector V2)
          # wget https://inspector-agent.amazonaws.com/linux/latest/install
          # bash install
          
          # Install application packages
          yum install -y httpd mysql php
          systemctl enable httpd
          systemctl start httpd
          
          # Create sample application
          echo "<?php phpinfo(); ?>" > /var/www/html/info.php
      Tags:
        - Key: Name
          Value: Inspector-WebServer
        - Key: Environment
          Value: Production
        - Key: InspectorScan
          Value: Enabled

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web server
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: HTTPS outbound
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: HTTP outbound
```

### Network Reachability Assessment
```python
# Network reachability analysis
import boto3
import json

class NetworkReachabilityAnalyzer:
    def __init__(self):
        self.inspector2_client = boto3.client('inspector2')
        self.ec2_client = boto3.client('ec2')
    
    def analyze_network_reachability(self, instance_ids=None):
        """Analyze network reachability for EC2 instances"""
        
        if not instance_ids:
            # Get all instances if none specified
            instances_response = self.ec2_client.describe_instances(
                Filters=[
                    {'Name': 'instance-state-name', 'Values': ['running']}
                ]
            )
            
            instance_ids = []
            for reservation in instances_response['Reservations']:
                for instance in reservation['Instances']:
                    instance_ids.append(instance['InstanceId'])
        
        reachability_analysis = {
            'analyzed_instances': len(instance_ids),
            'findings': [],
            'summary': {
                'internet_reachable': 0,
                'vpc_reachable': 0,
                'isolated': 0
            }
        }
        
        for instance_id in instance_ids:
            try:
                # Get Inspector findings for network reachability
                findings_response = self.inspector2_client.list_findings(
                    filterCriteria={
                        'resourceId': [instance_id],
                        'findingType': ['NETWORK_REACHABILITY']
                    }
                )
                
                instance_analysis = self.analyze_instance_network(instance_id)
                
                if findings_response['findings']:
                    for finding in findings_response['findings']:
                        network_details = finding.get('networkReachabilityDetails', {})
                        
                        reachability_finding = {
                            'instance_id': instance_id,
                            'finding_id': finding['findingArn'],
                            'severity': finding.get('severity'),
                            'network_path': network_details.get('networkPath', {}),
                            'open_port_range': network_details.get('openPortRange', {}),
                            'protocol': network_details.get('protocol'),
                            'reachable_from': self.determine_reachability_source(network_details)
                        }
                        
                        reachability_analysis['findings'].append(reachability_finding)
                        
                        # Update summary
                        if 'internet' in reachability_finding['reachable_from'].lower():
                            reachability_analysis['summary']['internet_reachable'] += 1
                        elif 'vpc' in reachability_finding['reachable_from'].lower():
                            reachability_analysis['summary']['vpc_reachable'] += 1
                
                else:
                    # No network reachability findings - likely isolated
                    reachability_analysis['summary']['isolated'] += 1
            
            except Exception as e:
                print(f"Error analyzing instance {instance_id}: {str(e)}")
        
        return reachability_analysis
    
    def analyze_instance_network(self, instance_id):
        """Analyze network configuration of an instance"""
        
        try:
            instance_response = self.ec2_client.describe_instances(InstanceIds=[instance_id])
            instance = instance_response['Reservations'][0]['Instances'][0]
            
            network_analysis = {
                'instance_id': instance_id,
                'vpc_id': instance.get('VpcId'),
                'subnet_id': instance.get('SubnetId'),
                'public_ip': instance.get('PublicIpAddress'),
                'private_ip': instance.get('PrivateIpAddress'),
                'security_groups': [sg['GroupId'] for sg in instance.get('SecurityGroups', [])],
                'network_interfaces': []
            }
            
            # Analyze network interfaces
            for eni in instance.get('NetworkInterfaces', []):
                network_analysis['network_interfaces'].append({
                    'interface_id': eni.get('NetworkInterfaceId'),
                    'public_ip': eni.get('Association', {}).get('PublicIp'),
                    'private_ip': eni.get('PrivateIpAddress'),
                    'security_groups': [sg['GroupId'] for sg in eni.get('Groups', [])]
                })
            
            # Check subnet route table
            subnet_response = self.ec2_client.describe_subnets(SubnetIds=[instance['SubnetId']])
            subnet = subnet_response['Subnets'][0]
            
            # Get route table associations
            route_tables_response = self.ec2_client.describe_route_tables(
                Filters=[
                    {'Name': 'association.subnet-id', 'Values': [subnet['SubnetId']]}
                ]
            )
            
            if route_tables_response['RouteTables']:
                route_table = route_tables_response['RouteTables'][0]
                network_analysis['route_table_id'] = route_table['RouteTableId']
                network_analysis['routes'] = route_table['Routes']
                
                # Check for internet gateway route
                has_igw_route = any(
                    route.get('GatewayId', '').startswith('igw-') and route.get('DestinationCidrBlock') == '0.0.0.0/0'
                    for route in route_table['Routes']
                )
                network_analysis['has_internet_route'] = has_igw_route
            
            return network_analysis
        
        except Exception as e:
            return {
                'instance_id': instance_id,
                'error': str(e)
            }
    
    def determine_reachability_source(self, network_details):
        """Determine the source of network reachability"""
        
        network_path = network_details.get('networkPath', {})
        
        # Check for internet gateway in path
        for step in network_path.get('steps', []):
            component = step.get('component', {})
            if component.get('type') == 'AWS_INTERNET_GATEWAY':
                return 'Internet'
        
        # Check for VPC peering or transit gateway
        for step in network_path.get('steps', []):
            component = step.get('component', {})
            if component.get('type') in ['AWS_VPC_PEERING_CONNECTION', 'AWS_TRANSIT_GATEWAY']:
                return 'VPC Peering/Transit Gateway'
        
        return 'VPC Internal'
```

---

## 4. Container Image Assessment

### ECR Integration
```yaml
# CloudFormation for ECR with Inspector scanning
Resources:
  ECRRepository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: my-application
      ImageScanningConfiguration:
        ScanOnPush: true
      ImageTagMutability: MUTABLE
      LifecyclePolicy:
        LifecyclePolicyText: |
          {
            "rules": [
              {
                "rulePriority": 1,
                "description": "Keep last 10 images",
                "selection": {
                  "tagStatus": "tagged",
                  "tagPrefixList": ["v"],
                  "countType": "imageCountMoreThan",
                  "countNumber": 10
                },
                "action": {
                  "type": "expire"
                }
              }
            ]
          }

  # Lambda function to process ECR scan results
  ECRScanProcessor:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: ecr-scan-processor
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt ECRScanProcessorRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          
          def lambda_handler(event, context):
              """Process ECR image scan results"""
              
              ecr = boto3.client('ecr')
              inspector2 = boto3.client('inspector2')
              sns = boto3.client('sns')
              
              # Parse ECR scan event
              detail = event['detail']
              repository_name = detail['repository-name']
              image_tag = detail['image-tags'][0] if detail['image-tags'] else 'latest'
              scan_status = detail['scan-status']
              
              if scan_status == 'COMPLETE':
                  # Get scan results from Inspector
                  findings = get_image_findings(inspector2, repository_name, image_tag)
                  
                  # Process findings
                  scan_summary = process_scan_findings(findings)
                  
                  # Send notification if critical vulnerabilities found
                  if scan_summary['critical_count'] > 0:
                      send_critical_alert(sns, repository_name, image_tag, scan_summary)
                  
                  # Block deployment if too many high-severity vulnerabilities
                  if scan_summary['high_count'] > 5:
                      block_deployment(repository_name, image_tag)
              
              return {
                  'statusCode': 200,
                  'body': json.dumps('ECR scan processed successfully')
              }
          
          def get_image_findings(inspector2, repository_name, image_tag):
              """Get Inspector findings for container image"""
              
              try:
                  response = inspector2.list_findings(
                      filterCriteria={
                          'ecrImageTags': [image_tag],
                          'ecrRepositoryName': [repository_name]
                      }
                  )
                  
                  return response['findings']
              
              except Exception as e:
                  print(f"Error getting findings: {str(e)}")
                  return []
          
          def process_scan_findings(findings):
              """Process and summarize scan findings"""
              
              summary = {
                  'total_findings': len(findings),
                  'critical_count': 0,
                  'high_count': 0,
                  'medium_count': 0,
                  'low_count': 0,
                  'vulnerabilities': []
              }
              
              for finding in findings:
                  severity = finding.get('severity', 'INFORMATIONAL')
                  
                  if severity == 'CRITICAL':
                      summary['critical_count'] += 1
                  elif severity == 'HIGH':
                      summary['high_count'] += 1
                  elif severity == 'MEDIUM':
                      summary['medium_count'] += 1
                  elif severity == 'LOW':
                      summary['low_count'] += 1
                  
                  # Extract vulnerability details
                  vuln_details = finding.get('packageVulnerabilityDetails', {})
                  if vuln_details:
                      summary['vulnerabilities'].append({
                          'id': vuln_details.get('vulnerabilityId'),
                          'severity': severity,
                          'package': vuln_details.get('vulnerablePackages', [{}])[0].get('name'),
                          'fixed_version': vuln_details.get('vulnerablePackages', [{}])[0].get('fixedInVersion')
                      })
              
              return summary
          
          def send_critical_alert(sns, repository_name, image_tag, summary):
              """Send alert for critical vulnerabilities"""
              
              message = f"""
              Critical Vulnerabilities Detected
              ================================
              
              Repository: {repository_name}
              Image Tag: {image_tag}
              
              Critical Vulnerabilities: {summary['critical_count']}
              High Vulnerabilities: {summary['high_count']}
              
              Top Vulnerabilities:
              """
              
              for vuln in summary['vulnerabilities'][:5]:
                  message += f"\n- {vuln['id']} ({vuln['severity']}) in {vuln['package']}"
              
              sns.publish(
                  TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
                  Message=message,
                  Subject=f'Critical Vulnerabilities in {repository_name}:{image_tag}'
              )
          
          def block_deployment(repository_name, image_tag):
              """Block deployment of vulnerable image"""
              
              # Add tag to indicate blocked status
              ecr = boto3.client('ecr')
              
              try:
                  ecr.put_image_tag_mutability(
                      repositoryName=repository_name,
                      imageTagMutability='IMMUTABLE'
                  )
                  
                  print(f"Blocked deployment of {repository_name}:{image_tag}")
              
              except Exception as e:
                  print(f"Error blocking deployment: {str(e)}")

  # EventBridge rule for ECR scan completion
  ECRScanEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: ECRScanCompletion
      Description: Process ECR image scan completion
      EventPattern:
        source:
          - aws.ecr
        detail-type:
          - ECR Image Scan
        detail:
          scan-status:
            - COMPLETE
      State: ENABLED
      Targets:
        - Arn: !GetAtt ECRScanProcessor.Arn
          Id: ECRScanProcessorTarget
```

### CI/CD Integration
```yaml
# CodePipeline with Inspector scanning
Resources:
  BuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: secure-container-build
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_MEDIUM
        Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0
        PrivilegedMode: true
      Source:
        Type: CODEPIPELINE
        BuildSpec: |
          version: 0.2
          phases:
            pre_build:
              commands:
                - echo Logging in to Amazon ECR...
                - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
            build:
              commands:
                - echo Build started on `date`
                - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
                - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
            post_build:
              commands:
                - echo Build completed on `date`
                - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
                - echo Waiting for Inspector scan...
                - python3 wait_for_scan.py
                - python3 check_vulnerabilities.py
          artifacts:
            files:
              - '**/*'

  # Security gate Lambda function
  SecurityGateFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: container-security-gate
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt SecurityGateRole.Arn
      Timeout: 300
      Code:
        ZipFile: |
          import boto3
          import json
          import time
          
          def lambda_handler(event, context):
              """Security gate for container deployment"""
              
              codepipeline = boto3.client('codepipeline')
              inspector2 = boto3.client('inspector2')
              
              # Get job details
              job_id = event['CodePipeline.job']['id']
              job_data = event['CodePipeline.job']['data']
              
              # Extract image details from input artifacts
              repository_name = job_data['actionConfiguration']['configuration']['RepositoryName']
              image_tag = job_data['actionConfiguration']['configuration']['ImageTag']
              
              try:
                  # Wait for scan to complete
                  scan_complete = wait_for_scan_completion(inspector2, repository_name, image_tag)
                  
                  if not scan_complete:
                      codepipeline.put_job_failure_result(
                          jobId=job_id,
                          failureDetails={'message': 'Scan did not complete within timeout', 'type': 'JobFailed'}
                      )
                      return
                  
                  # Check vulnerability threshold
                  vulnerability_check = check_vulnerability_threshold(inspector2, repository_name, image_tag)
                  
                  if vulnerability_check['passed']:
                      codepipeline.put_job_success_result(jobId=job_id)
                  else:
                      codepipeline.put_job_failure_result(
                          jobId=job_id,
                          failureDetails={
                              'message': f"Security gate failed: {vulnerability_check['reason']}",
                              'type': 'JobFailed'
                          }
                      )
              
              except Exception as e:
                  codepipeline.put_job_failure_result(
                      jobId=job_id,
                      failureDetails={'message': str(e), 'type': 'JobFailed'}
                  )
          
          def wait_for_scan_completion(inspector2, repository_name, image_tag, max_wait=300):
              """Wait for Inspector scan to complete"""
              
              start_time = time.time()
              
              while time.time() - start_time < max_wait:
                  try:
                      findings = inspector2.list_findings(
                          filterCriteria={
                              'ecrRepositoryName': [repository_name],
                              'ecrImageTags': [image_tag]
                          }
                      )
                      
                      # If we get findings, scan is complete
                      if findings['findings']:
                          return True
                      
                      time.sleep(30)  # Wait 30 seconds before checking again
                  
                  except Exception as e:
                      print(f"Error checking scan status: {str(e)}")
                      time.sleep(30)
              
              return False
          
          def check_vulnerability_threshold(inspector2, repository_name, image_tag):
              """Check if vulnerabilities exceed threshold"""
              
              try:
                  findings = inspector2.list_findings(
                      filterCriteria={
                          'ecrRepositoryName': [repository_name],
                          'ecrImageTags': [image_tag]
                      }
                  )
                  
                  critical_count = 0
                  high_count = 0
                  
                  for finding in findings['findings']:
                      severity = finding.get('severity', 'INFORMATIONAL')
                      if severity == 'CRITICAL':
                          critical_count += 1
                      elif severity == 'HIGH':
                          high_count += 1
                  
                  # Security policy: No critical vulnerabilities, max 3 high
                  if critical_count > 0:
                      return {
                          'passed': False,
                          'reason': f'{critical_count} critical vulnerabilities found'
                      }
                  
                  if high_count > 3:
                      return {
                          'passed': False,
                          'reason': f'{high_count} high vulnerabilities found (max 3 allowed)'
                      }
                  
                  return {
                      'passed': True,
                      'reason': 'Security requirements met'
                  }
              
              except Exception as e:
                  return {
                      'passed': False,
                      'reason': f'Error checking vulnerabilities: {str(e)}'
                  }
```

---

## 5. Findings Management and Remediation

### Automated Remediation
```python
# Automated remediation for Inspector findings
import boto3
import json
from datetime import datetime

class InspectorRemediationManager:
    def __init__(self):
        self.inspector2_client = boto3.client('inspector2')
        self.ssm_client = boto3.client('ssm')
        self.ec2_client = boto3.client('ec2')
        self.sns_client = boto3.client('sns')
    
    def process_findings_for_remediation(self, severity_threshold='HIGH'):
        """Process Inspector findings and trigger remediation"""
        
        # Get findings above severity threshold
        findings_response = self.inspector2_client.list_findings(
            filterCriteria={
                'severity': [severity_threshold, 'CRITICAL']
            }
        )
        
        remediation_results = []
        
        for finding in findings_response['findings']:
            try:
                remediation_result = self.remediate_finding(finding)
                remediation_results.append(remediation_result)
            except Exception as e:
                remediation_results.append({
                    'finding_id': finding.get('findingArn'),
                    'status': 'FAILED',
                    'error': str(e)
                })
        
        return remediation_results
    
    def remediate_finding(self, finding):
        """Remediate a specific Inspector finding"""
        
        finding_type = finding.get('type')
        severity = finding.get('severity')
        resources = finding.get('resources', [])
        
        remediation_actions = []
        
        # Package vulnerability remediation
        if 'PACKAGE_VULNERABILITY' in finding_type:
            for resource in resources:
                if 'ec2-instance' in resource.get('id', ''):
                    # EC2 instance vulnerability
                    instance_id = resource['id'].split('/')[-1]
                    action = self.remediate_ec2_vulnerability(finding, instance_id)
                    remediation_actions.append(action)
                
                elif 'ecr' in resource.get('id', ''):
                    # Container image vulnerability
                    action = self.remediate_container_vulnerability(finding, resource)
                    remediation_actions.append(action)
        
        # Network reachability remediation
        elif 'NETWORK_REACHABILITY' in finding_type:
            for resource in resources:
                if 'ec2-instance' in resource.get('id', ''):
                    instance_id = resource['id'].split('/')[-1]
                    action = self.remediate_network_exposure(finding, instance_id)
                    remediation_actions.append(action)
        
        return {
            'finding_id': finding.get('findingArn'),
            'finding_type': finding_type,
            'severity': severity,
            'remediation_actions': remediation_actions,
            'status': 'COMPLETED'
        }
    
    def remediate_ec2_vulnerability(self, finding, instance_id):
        """Remediate vulnerability on EC2 instance"""
        
        vuln_details = finding.get('packageVulnerabilityDetails', {})
        vulnerable_packages = vuln_details.get('vulnerablePackages', [])
        
        if not vulnerable_packages:
            return {
                'action': 'EC2 Vulnerability Remediation',
                'status': 'SKIPPED',
                'reason': 'No vulnerable packages identified'
            }
        
        package_name = vulnerable_packages[0].get('name')
        fixed_version = vulnerable_packages[0].get('fixedInVersion')
        
        try:
            # Create remediation script
            remediation_script = self.create_package_update_script(package_name, fixed_version)
            
            # Execute via Systems Manager
            response = self.ssm_client.send_command(
                InstanceIds=[instance_id],
                DocumentName='AWS-RunShellScript',
                Parameters={
                    'commands': remediation_script
                },
                Comment=f'Inspector vulnerability remediation for {package_name}'
            )
            
            return {
                'action': 'EC2 Vulnerability Remediation',
                'status': 'INITIATED',
                'command_id': response['Command']['CommandId'],
                'package': package_name,
                'fixed_version': fixed_version
            }
        
        except Exception as e:
            return {
                'action': 'EC2 Vulnerability Remediation',
                'status': 'FAILED',
                'error': str(e)
            }
    
    def create_package_update_script(self, package_name, fixed_version):
        """Create script to update vulnerable package"""
        
        script = [
            '#!/bin/bash',
            'set -e',
            '',
            '# Update package repositories',
            'if command -v yum &> /dev/null; then',
            '    yum update -y',
            f'    yum update -y {package_name}',
            'elif command -v apt-get &> /dev/null; then',
            '    apt-get update',
            f'    apt-get install -y {package_name}',
            'elif command -v apk &> /dev/null; then',
            '    apk update',
            f'    apk upgrade {package_name}',
            'fi',
            '',
            '# Verify update',
            f'echo "Package {package_name} updated successfully"',
            '',
            '# Check if reboot is required',
            'if [ -f /var/run/reboot-required ]; then',
            '    echo "REBOOT_REQUIRED: System requires reboot"',
            'fi'
        ]
        
        return script
    
    def remediate_container_vulnerability(self, finding, resource):
        """Remediate vulnerability in container image"""
        
        # Extract repository and tag information
        resource_id = resource.get('id', '')
        repository_name = resource_id.split('/')[-2] if '/' in resource_id else 'unknown'
        
        vuln_details = finding.get('packageVulnerabilityDetails', {})
        vulnerable_packages = vuln_details.get('vulnerablePackages', [])
        
        if not vulnerable_packages:
            return {
                'action': 'Container Vulnerability Remediation',
                'status': 'SKIPPED',
                'reason': 'No vulnerable packages identified'
            }
        
        # Create remediation guidance
        remediation_guidance = self.create_container_remediation_guidance(vulnerable_packages)
        
        # Send notification to development team
        self.send_container_remediation_notification(repository_name, finding, remediation_guidance)
        
        return {
            'action': 'Container Vulnerability Remediation',
            'status': 'NOTIFICATION_SENT',
            'repository': repository_name,
            'guidance': remediation_guidance
        }
    
    def create_container_remediation_guidance(self, vulnerable_packages):
        """Create remediation guidance for container vulnerabilities"""
        
        guidance = {
            'dockerfile_updates': [],
            'base_image_recommendations': [],
            'package_updates': []
        }
        
        for package in vulnerable_packages:
            package_name = package.get('name')
            current_version = package.get('version')
            fixed_version = package.get('fixedInVersion')
            
            if fixed_version:
                guidance['package_updates'].append({
                    'package': package_name,
                    'current_version': current_version,
                    'fixed_version': fixed_version,
                    'dockerfile_instruction': f'RUN apt-get update && apt-get install -y {package_name}={fixed_version}'
                })
        
        return guidance
    
    def remediate_network_exposure(self, finding, instance_id):
        """Remediate network exposure issues"""
        
        network_details = finding.get('networkReachabilityDetails', {})
        open_port_range = network_details.get('openPortRange', {})
        protocol = network_details.get('protocol', 'TCP')
        
        try:
            # Get instance security groups
            instance_response = self.ec2_client.describe_instances(InstanceIds=[instance_id])
            security_groups = instance_response['Reservations'][0]['Instances'][0]['SecurityGroups']
            
            remediation_actions = []
            
            for sg in security_groups:
                sg_id = sg['GroupId']
                
                # Get security group rules
                sg_response = self.ec2_client.describe_security_groups(GroupIds=[sg_id])
                sg_rules = sg_response['SecurityGroups'][0]['IpPermissions']
                
                # Check for overly permissive rules
                for rule in sg_rules:
                    if self.is_rule_overly_permissive(rule, open_port_range, protocol):
                        # Create more restrictive rule
                        new_rule = self.create_restrictive_rule(rule, open_port_range)
                        
                        # Remove old rule and add new one
                        self.ec2_client.revoke_security_group_ingress(
                            GroupId=sg_id,
                            IpPermissions=[rule]
                        )
                        
                        self.ec2_client.authorize_security_group_ingress(
                            GroupId=sg_id,
                            IpPermissions=[new_rule]
                        )
                        
                        remediation_actions.append({
                            'security_group': sg_id,
                            'action': 'Rule Updated',
                            'old_rule': rule,
                            'new_rule': new_rule
                        })
            
            return {
                'action': 'Network Exposure Remediation',
                'status': 'COMPLETED',
                'instance_id': instance_id,
                'remediation_actions': remediation_actions
            }
        
        except Exception as e:
            return {
                'action': 'Network Exposure Remediation',
                'status': 'FAILED',
                'error': str(e)
            }
    
    def is_rule_overly_permissive(self, rule, open_port_range, protocol):
        """Check if security group rule is overly permissive"""
        
        # Check for 0.0.0.0/0 source
        for ip_range in rule.get('IpRanges', []):
            if ip_range.get('CidrIp') == '0.0.0.0/0':
                return True
        
        # Check for wide port ranges
        from_port = rule.get('FromPort', 0)
        to_port = rule.get('ToPort', 65535)
        
        if to_port - from_port > 100:  # Wide port range
            return True
        
        return False
    
    def send_container_remediation_notification(self, repository_name, finding, guidance):
        """Send notification about container vulnerability remediation"""
        
        message = f"""
        Container Vulnerability Remediation Required
        ==========================================
        
        Repository: {repository_name}
        Finding ID: {finding.get('findingArn')}
        Severity: {finding.get('severity')}
        
        Vulnerable Packages:
        """
        
        for update in guidance['package_updates']:
            message += f"\n- {update['package']}: {update['current_version']} -> {update['fixed_version']}"
        
        message += "\n\nRemediation Steps:\n"
        for update in guidance['package_updates']:
            message += f"\n{update['dockerfile_instruction']}"
        
        self.sns_client.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:container-security',
            Message=message,
            Subject=f'Container Vulnerability Remediation: {repository_name}'
        )
```

---

## 6. Common Exam Scenarios

### Scenario 1: Automated Security Pipeline
```yaml
# Complete automated security pipeline with Inspector
Resources:
  SecurityPipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt CodePipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactsBucket
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeCommit
                Version: '1'
              Configuration:
                RepositoryName: secure-application
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
                Version: '1'
              Configuration:
                ProjectName: !Ref SecureBuildProject
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput

        - Name: SecurityScan
          Actions:
            - Name: InspectorScan
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: Lambda
                Version: '1'
              Configuration:
                FunctionName: !Ref SecurityScanFunction
              InputArtifacts:
                - Name: BuildOutput
              OutputArtifacts:
                - Name: ScanOutput

        - Name: SecurityGate
          Actions:
            - Name: SecurityApproval
              ActionTypeId:
                Category: Invoke
                Owner: AWS
                Provider: Lambda
                Version: '1'
              Configuration:
                FunctionName: !Ref SecurityGateFunction
              InputArtifacts:
                - Name: ScanOutput

        - Name: Deploy
          Actions:
            - Name: DeployAction
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: ECS
                Version: '1'
              Configuration:
                ClusterName: !Ref ECSCluster
                ServiceName: !Ref ECSService
                FileName: imagedefinitions.json
              InputArtifacts:
                - Name: BuildOutput
```

---

## 7. Exam Tips

- **Understand Inspector V2** - Know the differences from Inspector Classic
- **Master finding types** - Package vulnerabilities vs network reachability
- **Know ECR integration** - Automatic scanning on image push
- **Practice remediation** - Automated and manual remediation workflows
- **Learn CI/CD integration** - Security gates and pipeline integration
- **Understand suppression** - When and how to suppress findings
- **Know multi-account setup** - Organization-wide Inspector deployment
- **Practice cost optimization** - Resource type selection and scanning frequency
- **Master troubleshooting** - Common issues with scanning and findings
- **Understand compliance** - Integration with Security Hub and compliance frameworks