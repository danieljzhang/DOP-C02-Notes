# Domain 2: Configuration Management & Infrastructure as Code - DOP-C02 Cheatsheet

## Weight: 20% | Focus: CloudFormation, CDK, Config, Systems Manager

---

## 🔥 **MUST KNOW Services**

### **CloudFormation** - Infrastructure as Code
- **Templates**: JSON/YAML, max 51,200 characters
- **Stacks**: Collection of resources, regional
- **StackSets**: Deploy across accounts/regions
- **Drift Detection**: Compare actual vs template
- **Change Sets**: Preview changes before execution

### **AWS CDK** - Code-based IaC
- **Constructs**: L1 (CFn resources), L2 (opinionated), L3 (patterns)
- **Apps**: Top-level construct container
- **Stacks**: Unit of deployment
- **Synthesis**: Converts to CloudFormation

### **AWS Config** - Configuration Compliance
- **Rules**: AWS managed, custom (Lambda)
- **Conformance Packs**: Collection of rules
- **Remediation**: Automatic fix via SSM/Lambda
- **Multi-Account**: Aggregator for organization

### **Systems Manager** - Operations Management
- **Parameter Store**: Secure configuration storage
- **Session Manager**: Shell access without SSH
- **Patch Manager**: OS patching automation
- **State Manager**: Desired state configuration

---

## ⚡ **Key Patterns**

### **CloudFormation Stack Dependencies**
```yaml
# Export from Stack A
Outputs:
  VPCId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub "${AWS::StackName}-VPC-ID"

# Import in Stack B
VpcId: !ImportValue "StackA-VPC-ID"
```

### **Cross-Account CloudFormation**
1. Create execution role in target account
2. Trust source account
3. Use role in StackSet operations

### **Config Remediation**
```yaml
RemediationConfiguration:
  ConfigRuleName: !Ref MyConfigRule
  TargetType: SSM_DOCUMENT
  TargetId: AWSConfigRemediation-DeleteUnusedSecurityGroup
```

---

## 🎯 **Exam Scenarios**

### **Scenario 1: Stack update fails**
- Check for resource dependencies
- Review change set before execution
- Verify IAM permissions for resources
- Check for resource limits

### **Scenario 2: Cross-account deployment fails**
- Verify execution role in target account
- Check trust policy allows source account
- Ensure S3 template bucket is accessible

### **Scenario 3: Config rule shows non-compliant**
- Check rule parameters and scope
- Verify resource configuration
- Review evaluation logic
- Check remediation configuration

---

## 📋 **Quick Commands**

```bash
# CloudFormation
aws cloudformation create-stack --stack-name MyStack --template-body file://template.yaml
aws cloudformation update-stack --stack-name MyStack --template-body file://template.yaml
aws cloudformation detect-stack-drift --stack-name MyStack

# CDK
cdk init app --language typescript
cdk synth
cdk deploy
cdk diff

# Config
aws configservice put-config-rule --config-rule file://rule.json
aws configservice start-config-rules-evaluation --config-rule-names MyRule

# Systems Manager
aws ssm get-parameter --name "/myapp/database/url" --with-decryption
aws ssm start-session --target i-1234567890abcdef0
```

---

## 🏗️ **CloudFormation Intrinsic Functions**
- **!Ref**: Reference parameter/resource
- **!GetAtt**: Get resource attribute
- **!Join**: Join values with delimiter
- **!Sub**: Substitute variables
- **!ImportValue**: Import cross-stack value
- **!If**: Conditional logic

---

## 🔧 **Systems Manager Key Features**
- **Parameter Store**: Free tier (10,000 params), Advanced (charges apply)
- **Session Manager**: No inbound ports, logged to CloudWatch
- **Patch Manager**: Maintenance windows, patch baselines
- **State Manager**: Associations, compliance reporting

---

## ⚠️ **Common Mistakes**
- Circular dependencies in CloudFormation stacks
- Not using change sets for critical updates
- Forgetting to enable Config recorder
- Hardcoding values instead of using parameters
- Not setting up proper IAM roles for cross-account access

---

## 🔑 **Key Points**
- **CloudFormation** is declarative, idempotent
- **StackSets** require service-linked roles
- **Config** requires delivery channel and recorder
- **Parameter Store** supports hierarchical naming
- **CDK** synthesizes to CloudFormation templates