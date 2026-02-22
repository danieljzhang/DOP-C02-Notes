# CI/CD Security Integration - DOP-C02 Exam Notes

## 1. Overview

**CI/CD Security Integration** encompasses the practices, tools, and processes for embedding security throughout the software development lifecycle. This approach, known as DevSecOps, ensures security is built into applications from the beginning rather than added as an afterthought.

### Key Characteristics
- **Shift-left security** - Security testing early in development cycle
- **Automated scanning** - Continuous security assessment in pipelines
- **Compliance automation** - Automated compliance checking and reporting
- **Vulnerability management** - Continuous monitoring and remediation
- **Secrets management** - Secure handling of credentials and sensitive data
- **Infrastructure security** - Security validation of infrastructure code
- **Threat modeling** - Automated security risk assessment

### What Problem Does It Solve?
- Identifies security vulnerabilities early in development
- Automates compliance checking and reporting
- Reduces security debt and technical debt
- Enables faster, more secure software delivery
- Provides continuous security monitoring and feedback
- Ensures consistent security standards across teams

---

## 2. Security Scanning Types

### Static Application Security Testing (SAST)
- **Source code analysis** - Scans code without execution
- **Early detection** - Finds vulnerabilities during development
- **Language support** - Supports multiple programming languages
- **Integration** - Embeds in IDE and CI/CD pipelines

### Dynamic Application Security Testing (DAST)
- **Runtime analysis** - Tests running applications
- **Black-box testing** - No source code access required
- **Real-world scenarios** - Tests actual attack vectors
- **Web application focus** - Primarily for web applications

### Interactive Application Security Testing (IAST)
- **Hybrid approach** - Combines SAST and DAST
- **Real-time analysis** - Tests during application execution
- **Accurate results** - Reduces false positives
- **Performance impact** - Minimal overhead during testing

### Software Composition Analysis (SCA)
- **Dependency scanning** - Analyzes third-party components
- **License compliance** - Checks open source licenses
- **Vulnerability database** - Cross-references known vulnerabilities
- **Supply chain security** - Monitors entire dependency chain

---

## 3. AWS Security Services Integration

### Amazon CodeGuru Security
```yaml
# CodeBuild integration with CodeGuru Security
version: 0.2
phases:
  install:
    runtime-versions:
      python: 3.9
  pre_build:
    commands:
      - echo "Installing security scanning tools"
      - pip install bandit safety
  build:
    commands:
      - echo "Running security scans"
      # SAST scanning with Bandit
      - bandit -r src/ -f json -o bandit-report.json
      # Dependency vulnerability scanning
      - safety check --json --output safety-report.json
      # CodeGuru Security scanning
      - aws codeguru-security create-scan --scan-name "pipeline-scan-${CODEBUILD_BUILD_NUMBER}"
  post_build:
    commands:
      - echo "Processing security scan results"
      - python process_security_results.py
artifacts:
  files:
    - bandit-report.json
    - safety-report.json
    - security-summary.json
```

### AWS Security Hub Integration
```python
import boto3
import json

def publish_to_security_hub(findings):
    """
    Publish security findings to AWS Security Hub
    """
    
    securityhub = boto3.client('securityhub')
    
    # Convert findings to Security Hub format
    hub_findings = []
    
    for finding in findings:
        hub_finding = {
            'SchemaVersion': '2018-10-08',
            'Id': f"codepipeline-security-{finding['id']}",
            'ProductArn': f"arn:aws:securityhub:{boto3.Session().region_name}:{boto3.client('sts').get_caller_identity()['Account']}:product/custom/pipeline-security",
            'GeneratorId': 'pipeline-security-scanner',
            'AwsAccountId': boto3.client('sts').get_caller_identity()['Account'],
            'Types': ['Software and Configuration Checks/Vulnerabilities/CVE'],
            'CreatedAt': finding['timestamp'],
            'UpdatedAt': finding['timestamp'],
            'Severity': {
                'Label': finding['severity'].upper()
            },
            'Title': finding['title'],
            'Description': finding['description'],
            'Resources': [
                {
                    'Type': 'AwsCodeBuildProject',
                    'Id': finding['resource_id'],
                    'Region': boto3.Session().region_name
                }
            ],
            'Compliance': {
                'Status': 'FAILED' if finding['severity'] in ['HIGH', 'CRITICAL'] else 'WARNING'
            }
        }
        
        hub_findings.append(hub_finding)
    
    # Batch import findings
    if hub_findings:
        response = securityhub.batch_import_findings(Findings=hub_findings)
        print(f"Imported {len(hub_findings)} findings to Security Hub")
        return response
    
    return None

def lambda_handler(event, context):
    """
    Process CodeBuild security scan results and publish to Security Hub
    """
    
    # Parse CodeBuild event
    build_id = event['detail']['build-id']
    project_name = event['detail']['project-name']
    build_status = event['detail']['build-status']
    
    if build_status == 'SUCCEEDED':
        # Get security scan results from S3
        s3 = boto3.client('s3')
        
        try:
            # Download security reports
            bandit_report = download_security_report(s3, 'bandit-report.json', build_id)
            safety_report = download_security_report(s3, 'safety-report.json', build_id)
            
            # Process and normalize findings
            findings = []
            findings.extend(process_bandit_findings(bandit_report, project_name))
            findings.extend(process_safety_findings(safety_report, project_name))
            
            # Publish to Security Hub
            if findings:
                publish_to_security_hub(findings)
                
                # Check if critical findings should fail the pipeline
                critical_findings = [f for f in findings if f['severity'] == 'CRITICAL']
                if critical_findings:
                    # Fail the CodePipeline stage
                    fail_pipeline_stage(event, f"Found {len(critical_findings)} critical security issues")
            
        except Exception as e:
            print(f"Error processing security results: {str(e)}")
            raise
    
    return {
        'statusCode': 200,
        'body': json.dumps('Security processing completed')
    }
```

### Amazon Inspector Integration
```yaml
Resources:
  InspectorAssessmentTarget:
    Type: AWS::Inspector::AssessmentTarget
    Properties:
      AssessmentTargetName: PipelineSecurityAssessment
      ResourceGroupArn: !GetAtt InspectorResourceGroup.Arn

  InspectorAssessmentTemplate:
    Type: AWS::Inspector::AssessmentTemplate
    Properties:
      AssessmentTargetArn: !Ref InspectorAssessmentTarget
      AssessmentTemplateName: SecurityAssessmentTemplate
      DurationInSeconds: 3600
      RulesPackageArns:
        - arn:aws:inspector:us-east-1:316112463485:rulespackage/0-gEjTy7T7  # Security Best Practices
        - arn:aws:inspector:us-east-1:316112463485:rulespackage/0-rExsr2X8  # Network Reachability
        - arn:aws:inspector:us-east-1:316112463485:rulespackage/0-R01qwB5Q  # Runtime Behavior Analysis

  # Lambda function to trigger Inspector assessment
  InspectorTriggerFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.9
      Handler: index.handler
      Code:
        ZipFile: |
          import boto3
          import json
          
          def handler(event, context):
              inspector = boto3.client('inspector')
              
              # Start assessment run
              response = inspector.start_assessment_run(
                  assessmentTemplateArn=event['assessmentTemplateArn'],
                  assessmentRunName=f"pipeline-assessment-{event['buildId']}"
              )
              
              return {
                  'statusCode': 200,
                  'assessmentRunArn': response['assessmentRunArn']
              }
```

---

## 4. Container Security Integration

### ECR Image Scanning
```yaml
# CodeBuild project for container security
version: 0.2
phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR"
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo "Building Docker image"
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo "Pushing image to ECR"
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
      
      - echo "Starting ECR image scan"
      - aws ecr start-image-scan --repository-name $IMAGE_REPO_NAME --image-id imageTag=$IMAGE_TAG
      
      - echo "Waiting for scan completion"
      - aws ecr wait image-scan-complete --repository-name $IMAGE_REPO_NAME --image-id imageTag=$IMAGE_TAG
      
      - echo "Getting scan results"
      - aws ecr describe-image-scan-findings --repository-name $IMAGE_REPO_NAME --image-id imageTag=$IMAGE_TAG > scan-results.json
      
      - echo "Processing scan results"
      - python process_ecr_scan.py
artifacts:
  files:
    - scan-results.json
    - security-report.json
```

### Container Security Scanning Script
```python
import json
import sys

def process_ecr_scan_results():
    """
    Process ECR scan results and determine if deployment should proceed
    """
    
    with open('scan-results.json', 'r') as f:
        scan_results = json.load(f)
    
    findings = scan_results.get('imageScanFindings', {}).get('findings', [])
    
    # Categorize findings by severity
    severity_counts = {
        'CRITICAL': 0,
        'HIGH': 0,
        'MEDIUM': 0,
        'LOW': 0,
        'INFORMATIONAL': 0
    }
    
    for finding in findings:
        severity = finding.get('severity', 'UNKNOWN')
        if severity in severity_counts:
            severity_counts[severity] += 1
    
    # Security policy: Fail if critical or high severity vulnerabilities
    critical_threshold = 0
    high_threshold = 5
    
    security_report = {
        'scan_status': 'PASSED',
        'total_findings': len(findings),
        'severity_breakdown': severity_counts,
        'policy_violations': []
    }
    
    if severity_counts['CRITICAL'] > critical_threshold:
        security_report['scan_status'] = 'FAILED'
        security_report['policy_violations'].append(
            f"Critical vulnerabilities ({severity_counts['CRITICAL']}) exceed threshold ({critical_threshold})"
        )
    
    if severity_counts['HIGH'] > high_threshold:
        security_report['scan_status'] = 'FAILED'
        security_report['policy_violations'].append(
            f"High severity vulnerabilities ({severity_counts['HIGH']}) exceed threshold ({high_threshold})"
        )
    
    # Write security report
    with open('security-report.json', 'w') as f:
        json.dump(security_report, f, indent=2)
    
    # Exit with error code if security scan failed
    if security_report['scan_status'] == 'FAILED':
        print("Security scan failed - deployment blocked")
        for violation in security_report['policy_violations']:
            print(f"Policy violation: {violation}")
        sys.exit(1)
    else:
        print("Security scan passed - deployment approved")
        print(f"Total findings: {security_report['total_findings']}")
        for severity, count in severity_counts.items():
            if count > 0:
                print(f"{severity}: {count}")

if __name__ == '__main__':
    process_ecr_scan_results()
```

---

## 5. Secrets Management in CI/CD

### AWS Secrets Manager Integration
```yaml
# CodePipeline with Secrets Manager
- Name: Deploy
  Actions:
    - Name: DeployWithSecrets
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
            "DatabasePassword": "{{resolve:secretsmanager:prod/myapp/db:SecretString:password}}",
            "ApiKey": "{{resolve:secretsmanager:prod/myapp/api:SecretString:key}}",
            "Environment": "Production"
          }
        Capabilities: CAPABILITY_IAM
      InputArtifacts:
        - Name: BuildOutput
```

### Secrets Scanning in Pipeline
```python
import re
import json
import os

def scan_for_secrets(directory):
    """
    Scan source code for potential secrets
    """
    
    # Common secret patterns
    secret_patterns = {
        'aws_access_key': r'AKIA[0-9A-Z]{16}',
        'aws_secret_key': r'[0-9a-zA-Z/+]{40}',
        'api_key': r'api[_-]?key["\']?\s*[:=]\s*["\']?[0-9a-zA-Z]{20,}',
        'password': r'password["\']?\s*[:=]\s*["\']?[^\s"\']{8,}',
        'private_key': r'-----BEGIN\s+(RSA\s+)?PRIVATE\s+KEY-----',
        'jwt_token': r'eyJ[A-Za-z0-9-_=]+\.[A-Za-z0-9-_=]+\.?[A-Za-z0-9-_.+/=]*'
    }
    
    findings = []
    
    for root, dirs, files in os.walk(directory):
        # Skip common directories that shouldn't contain secrets
        dirs[:] = [d for d in dirs if d not in ['.git', 'node_modules', '__pycache__', '.venv']]
        
        for file in files:
            # Skip binary files and common non-source files
            if file.endswith(('.pyc', '.jpg', '.png', '.gif', '.pdf', '.zip')):
                continue
                
            file_path = os.path.join(root, file)
            
            try:
                with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                    content = f.read()
                    
                    for secret_type, pattern in secret_patterns.items():
                        matches = re.finditer(pattern, content, re.IGNORECASE)
                        
                        for match in matches:
                            line_number = content[:match.start()].count('\n') + 1
                            
                            findings.append({
                                'type': secret_type,
                                'file': file_path,
                                'line': line_number,
                                'match': match.group()[:20] + '...',  # Truncate for security
                                'severity': 'HIGH'
                            })
                            
            except Exception as e:
                print(f"Error scanning file {file_path}: {str(e)}")
    
    return findings

def lambda_handler(event, context):
    """
    CodeBuild post-build hook for secrets scanning
    """
    
    # Scan source code directory
    findings = scan_for_secrets('/codebuild/output/src')
    
    # Generate report
    report = {
        'scan_timestamp': datetime.utcnow().isoformat(),
        'total_findings': len(findings),
        'findings': findings,
        'status': 'FAILED' if findings else 'PASSED'
    }
    
    # Write report
    with open('/codebuild/output/secrets-scan-report.json', 'w') as f:
        json.dump(report, f, indent=2)
    
    # Fail build if secrets found
    if findings:
        print(f"SECURITY ALERT: Found {len(findings)} potential secrets in source code")
        for finding in findings:
            print(f"  {finding['type']} in {finding['file']}:{finding['line']}")
        
        # Publish to Security Hub
        publish_secrets_findings_to_security_hub(findings)
        
        return {
            'statusCode': 400,
            'body': json.dumps('Secrets detected - build failed')
        }
    
    return {
        'statusCode': 200,
        'body': json.dumps('No secrets detected')
    }
```

---

## 6. Infrastructure Security Validation

### CloudFormation Security Scanning
```python
import boto3
import json
import yaml

def scan_cloudformation_template(template_content):
    """
    Scan CloudFormation template for security issues
    """
    
    findings = []
    
    try:
        # Parse template (YAML or JSON)
        if template_content.strip().startswith('{'):
            template = json.loads(template_content)
        else:
            template = yaml.safe_load(template_content)
        
        resources = template.get('Resources', {})
        
        for resource_name, resource in resources.items():
            resource_type = resource.get('Type', '')
            properties = resource.get('Properties', {})
            
            # Check S3 bucket security
            if resource_type == 'AWS::S3::Bucket':
                findings.extend(check_s3_security(resource_name, properties))
            
            # Check Security Group rules
            elif resource_type == 'AWS::EC2::SecurityGroup':
                findings.extend(check_security_group_rules(resource_name, properties))
            
            # Check IAM policies
            elif resource_type in ['AWS::IAM::Role', 'AWS::IAM::Policy', 'AWS::IAM::User']:
                findings.extend(check_iam_security(resource_name, resource_type, properties))
            
            # Check RDS security
            elif resource_type == 'AWS::RDS::DBInstance':
                findings.extend(check_rds_security(resource_name, properties))
    
    except Exception as e:
        findings.append({
            'type': 'TEMPLATE_PARSING_ERROR',
            'severity': 'HIGH',
            'resource': 'Template',
            'message': f'Failed to parse template: {str(e)}'
        })
    
    return findings

def check_s3_security(resource_name, properties):
    """Check S3 bucket security configuration"""
    
    findings = []
    
    # Check for public access
    public_access_block = properties.get('PublicAccessBlockConfiguration', {})
    if not all([
        public_access_block.get('BlockPublicAcls', False),
        public_access_block.get('BlockPublicPolicy', False),
        public_access_block.get('IgnorePublicAcls', False),
        public_access_block.get('RestrictPublicBuckets', False)
    ]):
        findings.append({
            'type': 'S3_PUBLIC_ACCESS_NOT_BLOCKED',
            'severity': 'HIGH',
            'resource': resource_name,
            'message': 'S3 bucket does not block all public access'
        })
    
    # Check for encryption
    encryption = properties.get('BucketEncryption', {})
    if not encryption:
        findings.append({
            'type': 'S3_ENCRYPTION_NOT_ENABLED',
            'severity': 'MEDIUM',
            'resource': resource_name,
            'message': 'S3 bucket encryption is not configured'
        })
    
    # Check for versioning
    versioning = properties.get('VersioningConfiguration', {})
    if versioning.get('Status') != 'Enabled':
        findings.append({
            'type': 'S3_VERSIONING_NOT_ENABLED',
            'severity': 'LOW',
            'resource': resource_name,
            'message': 'S3 bucket versioning is not enabled'
        })
    
    return findings

def check_security_group_rules(resource_name, properties):
    """Check Security Group rules for security issues"""
    
    findings = []
    
    ingress_rules = properties.get('SecurityGroupIngress', [])
    
    for rule in ingress_rules:
        cidr_ip = rule.get('CidrIp', '')
        from_port = rule.get('FromPort', 0)
        to_port = rule.get('ToPort', 0)
        
        # Check for unrestricted access
        if cidr_ip == '0.0.0.0/0':
            if from_port == 22 or to_port == 22:
                findings.append({
                    'type': 'SECURITY_GROUP_SSH_UNRESTRICTED',
                    'severity': 'HIGH',
                    'resource': resource_name,
                    'message': 'Security group allows unrestricted SSH access (0.0.0.0/0:22)'
                })
            
            if from_port == 3389 or to_port == 3389:
                findings.append({
                    'type': 'SECURITY_GROUP_RDP_UNRESTRICTED',
                    'severity': 'HIGH',
                    'resource': resource_name,
                    'message': 'Security group allows unrestricted RDP access (0.0.0.0/0:3389)'
                })
    
    return findings
```

---

## 7. Common Exam Scenarios

### Scenario 1: Implement security scanning in CI/CD pipeline
**Solution:**
- Add SAST scanning in build stage using CodeGuru or third-party tools
- Implement dependency vulnerability scanning with SCA tools
- Add container image scanning with ECR or third-party scanners
- Create security gates that fail pipeline on critical findings

### Scenario 2: Automate compliance checking in deployments
**Solution:**
- Use AWS Config rules for infrastructure compliance
- Implement CloudFormation template security scanning
- Add compliance validation in CodePipeline stages
- Generate compliance reports as pipeline artifacts

### Scenario 3: Secure secrets management in pipelines
**Solution:**
- Use AWS Secrets Manager for sensitive data storage
- Implement secrets scanning in source code
- Use dynamic references in CloudFormation templates
- Rotate secrets automatically and update deployments

### Scenario 4: Container security in CI/CD
**Solution:**
- Enable ECR image scanning for vulnerability detection
- Implement base image security policies
- Add runtime security scanning with third-party tools
- Create security policies for container deployments

### Scenario 5: Infrastructure security validation
**Solution:**
- Scan CloudFormation templates for security misconfigurations
- Implement security linting in pre-deployment stages
- Use AWS Config rules for post-deployment validation
- Add automated remediation for security violations

### Scenario 6: Security Hub integration for centralized findings
**Solution:**
- Configure Security Hub to aggregate security findings
- Integrate pipeline security tools with Security Hub
- Create custom findings for pipeline-specific security checks
- Set up automated response workflows for critical findings

### Scenario 7: Threat modeling automation
**Solution:**
- Implement automated threat modeling in design phase
- Use security scanning results for threat assessment
- Create security risk scores for deployments
- Integrate threat intelligence feeds for vulnerability prioritization

### Scenario 8: Security testing automation
**Solution:**
- Add DAST scanning for deployed applications
- Implement penetration testing automation
- Create security test suites for different environments
- Add security regression testing in pipelines

---

## 8. CLI Commands Reference

### Security Hub Operations
```bash
# Enable Security Hub
aws securityhub enable-security-hub

# Get findings
aws securityhub get-findings \
  --filters '{"ProductName":[{"Value":"CodePipeline Security","Comparison":"EQUALS"}]}'

# Batch import findings
aws securityhub batch-import-findings \
  --findings file://security-findings.json

# Update findings
aws securityhub batch-update-findings \
  --finding-identifiers Id=finding-id,ProductArn=product-arn \
  --workflow Status=RESOLVED
```

### CodeGuru Security Operations
```bash
# Create security scan
aws codeguru-security create-scan \
  --scan-name "pipeline-security-scan" \
  --resource-id repository-arn

# Get scan results
aws codeguru-security get-scan \
  --scan-name "pipeline-security-scan"

# List findings
aws codeguru-security list-findings-metrics \
  --start-date 2024-01-01 \
  --end-date 2024-01-31
```

### ECR Security Operations
```bash
# Start image scan
aws ecr start-image-scan \
  --repository-name my-repo \
  --image-id imageTag=latest

# Get scan results
aws ecr describe-image-scan-findings \
  --repository-name my-repo \
  --image-id imageTag=latest

# Put image scanning configuration
aws ecr put-image-scanning-configuration \
  --repository-name my-repo \
  --image-scanning-configuration scanOnPush=true
```

---

## 9. Best Practices for DOP-C02 Exam

### Security Integration
- Implement security scanning at multiple stages of pipeline
- Use both SAST and DAST for comprehensive coverage
- Integrate security tools with existing CI/CD workflows
- Automate security policy enforcement and compliance checking

### Vulnerability Management
- Prioritize vulnerabilities based on severity and exploitability
- Implement automated remediation for common security issues
- Track security metrics and trends over time
- Create security dashboards for visibility and reporting

### Secrets Management
- Never store secrets in source code or configuration files
- Use centralized secrets management services
- Implement secrets rotation and lifecycle management
- Monitor for secrets exposure and unauthorized access

### Compliance Automation
- Automate compliance checking and reporting
- Implement security gates in deployment pipelines
- Use infrastructure as code for consistent security configurations
- Regular audit and review of security policies and procedures

---

## 10. Exam Tips

### What to Remember
- **Shift-left security** means integrating security early in development
- **SAST scans source code** without execution
- **DAST tests running applications** for vulnerabilities
- **SCA analyzes dependencies** for known vulnerabilities
- **Security Hub centralizes** security findings across AWS
- **Secrets Manager** should be used for sensitive data in pipelines
- **ECR image scanning** detects vulnerabilities in container images
- **Config rules** can enforce security compliance automatically

### Common Traps
- Not implementing security scanning early enough in pipeline
- Storing secrets in source code or environment variables
- Not failing pipelines on critical security findings
- Overlooking dependency vulnerabilities in third-party libraries
- Not integrating security tools with centralized reporting
- Missing infrastructure security validation in IaC templates

### Scenario-Based Questions
- Focus on DevSecOps implementation patterns
- Understand when to use different types of security scanning
- Know integration patterns with AWS security services
- Understand compliance automation and reporting
- Know secrets management best practices
- Understand container security in CI/CD pipelines

---

## 11. Quick Reference Cheat Sheet

### Security Scanning Types
```
SAST: Static source code analysis
DAST: Dynamic application testing
IAST: Interactive application security testing
SCA: Software composition analysis (dependencies)
```

### AWS Security Services
```
Security Hub: Centralized security findings
CodeGuru Security: AI-powered code security
Inspector: Infrastructure security assessment
ECR: Container image vulnerability scanning
Secrets Manager: Secure secrets storage
```

### Pipeline Integration Points
```
Source: Secrets scanning, license checking
Build: SAST, dependency scanning
Test: DAST, security testing
Deploy: Infrastructure validation, compliance checking
Monitor: Runtime security, threat detection
```

---

## 12. Summary

CI/CD Security Integration is critical for modern DevSecOps practices and is heavily emphasized in the DOP-C02 exam. Key areas to master:

1. **Security scanning types** (SAST, DAST, IAST, SCA)
2. **AWS security services integration** (Security Hub, CodeGuru, Inspector)
3. **Container security** (ECR scanning, image policies)
4. **Secrets management** (Secrets Manager, Parameter Store)
5. **Infrastructure security** (CloudFormation scanning, Config rules)
6. **Compliance automation** (automated checking, reporting)
7. **Vulnerability management** (prioritization, remediation)
8. **Security monitoring** (continuous assessment, alerting)

Understanding these concepts with hands-on practice will ensure success on security integration questions in the DOP-C02 exam.