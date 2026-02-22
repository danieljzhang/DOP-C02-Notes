# AWS Secrets Manager - DOP-C02 Study Notes

## 1. Overview

### What is Secrets Manager?
- Managed service for storing, retrieving, and rotating secrets
- Centralized secret management across AWS services and applications
- Automatic rotation capabilities for supported services
- Integration with RDS, DocumentDB, Redshift, and other services

### Key Benefits
- **Automatic Rotation** - Built-in rotation for database credentials
- **Fine-grained Access Control** - IAM and resource-based policies
- **Encryption** - Secrets encrypted at rest and in transit
- **Audit Trail** - CloudTrail integration for compliance
- **Cross-Region Replication** - Disaster recovery and multi-region apps

### Use Cases
- Database credentials rotation
- API keys and tokens management
- Application configuration secrets
- Third-party service credentials
- Certificate and private key storage

---

## 2. Secret Types and Structure

### Supported Secret Types
- **Database Credentials** - RDS, Aurora, DocumentDB, Redshift
- **Other Credentials** - API keys, OAuth tokens, certificates
- **Arbitrary Text** - JSON, plain text, binary data
- **Key-Value Pairs** - Structured configuration data

### Secret Structure
```json
{
  "username": "admin",
  "password": "MySecretPassword123!",
  "engine": "mysql",
  "host": "mydb.cluster-xyz.us-east-1.rds.amazonaws.com",
  "port": 3306,
  "dbname": "production"
}
```

### Secret Versions
- Each secret can have multiple versions
- **AWSCURRENT** - Current active version
- **AWSPENDING** - Version being rotated to
- **AWSPREVIOUS** - Previous version (for rollback)
- Custom version stages for testing

---

## 3. Creating and Managing Secrets

### Create Database Secret
```bash
# Create RDS secret with automatic rotation
aws secretsmanager create-secret \
  --name "prod/db/mysql" \
  --description "Production MySQL credentials" \
  --secret-string '{
    "username": "admin",
    "password": "MyPassword123!",
    "engine": "mysql",
    "host": "mydb.cluster-xyz.us-east-1.rds.amazonaws.com",
    "port": 3306,
    "dbname": "production"
  }'
```

### Create API Key Secret
```bash
# Create API key secret
aws secretsmanager create-secret \
  --name "prod/api/external-service" \
  --description "External service API key" \
  --secret-string '{
    "api_key": "sk-1234567890abcdef",
    "api_url": "https://api.example.com",
    "timeout": 30
  }'
```

### Update Secret Value
```bash
# Update secret value
aws secretsmanager update-secret \
  --secret-id "prod/db/mysql" \
  --secret-string '{
    "username": "admin",
    "password": "NewPassword456!",
    "engine": "mysql",
    "host": "mydb.cluster-xyz.us-east-1.rds.amazonaws.com",
    "port": 3306,
    "dbname": "production"
  }'
```

---

## 4. Retrieving Secrets

### Get Secret Value
```bash
# Retrieve current secret
aws secretsmanager get-secret-value \
  --secret-id "prod/db/mysql" \
  --query SecretString \
  --output text
```

### Application Integration
```python
import boto3
import json

def get_secret(secret_name, region_name="us-east-1"):
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])
        return secret
    except Exception as e:
        print(f"Error retrieving secret: {e}")
        return None

# Usage
db_credentials = get_secret("prod/db/mysql")
if db_credentials:
    username = db_credentials['username']
    password = db_credentials['password']
    host = db_credentials['host']
```

### Batch Retrieval
```bash
# Get multiple secrets
aws secretsmanager batch-get-secret-value \
  --secret-id-list "prod/db/mysql" "prod/api/external-service"
```

---

## 5. Automatic Rotation

### Supported Services
- **Amazon RDS** - MySQL, PostgreSQL, Oracle, SQL Server
- **Amazon Aurora** - MySQL, PostgreSQL
- **Amazon DocumentDB** - MongoDB-compatible
- **Amazon Redshift** - Data warehouse

### Enable Rotation for RDS
```bash
# Enable automatic rotation
aws secretsmanager rotate-secret \
  --secret-id "prod/db/mysql" \
  --rotation-rules AutomaticallyAfterDays=30 \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSMySQLRotationSingleUser
```

### Rotation Configuration
```json
{
  "AutomaticallyAfterDays": 30,
  "RotationLambdaARN": "arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSMySQLRotationSingleUser"
}
```

### Rotation Process
1. **Create New Version** - Generate new password
2. **Set Secret** - Update database with new password
3. **Test Secret** - Verify new credentials work
4. **Finish Secret** - Mark new version as current

---

## 6. Custom Rotation with Lambda

### Lambda Rotation Function
```python
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Custom rotation function for API keys"""
    
    client = boto3.client('secretsmanager')
    secret_arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']
    
    try:
        if step == "createSecret":
            create_secret(client, secret_arn, token)
        elif step == "setSecret":
            set_secret(client, secret_arn, token)
        elif step == "testSecret":
            test_secret(client, secret_arn, token)
        elif step == "finishSecret":
            finish_secret(client, secret_arn, token)
        else:
            logger.error(f"Invalid step parameter: {step}")
            
    except Exception as e:
        logger.error(f"Rotation failed: {e}")
        raise e

def create_secret(client, secret_arn, token):
    """Create new secret version"""
    try:
        # Get current secret
        current_secret = client.get_secret_value(
            SecretId=secret_arn,
            VersionStage="AWSCURRENT"
        )
        
        # Generate new API key (example)
        new_secret = json.loads(current_secret['SecretString'])
        new_secret['api_key'] = generate_new_api_key()
        
        # Store new version
        client.put_secret_value(
            SecretId=secret_arn,
            ClientRequestToken=token,
            SecretString=json.dumps(new_secret),
            VersionStages=['AWSPENDING']
        )
        
    except client.exceptions.ResourceExistsException:
        logger.info("Secret version already exists")

def generate_new_api_key():
    """Generate new API key - implement your logic"""
    import secrets
    return f"sk-{secrets.token_hex(16)}"
```

### Rotation Lambda Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:DescribeSecret",
        "secretsmanager:GetSecretValue",
        "secretsmanager:PutSecretValue",
        "secretsmanager:UpdateSecretVersionStage"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:*"
    }
  ]
}
```

---

## 7. Cross-Region Replication

### Enable Replication
```bash
# Replicate secret to another region
aws secretsmanager replicate-secret-to-regions \
  --secret-id "prod/db/mysql" \
  --add-replica-regions Region=us-west-2,KmsKeyId=alias/aws/secretsmanager
```

### Replication Configuration
```json
{
  "ReplicationStatus": [
    {
      "Region": "us-west-2",
      "KmsKeyId": "alias/aws/secretsmanager",
      "Status": "InSync",
      "StatusMessage": "Replication successful"
    }
  ]
}
```

### Multi-Region Application
```python
import boto3

def get_secret_multi_region(secret_name, preferred_regions):
    """Get secret from multiple regions with failover"""
    
    for region in preferred_regions:
        try:
            client = boto3.client('secretsmanager', region_name=region)
            response = client.get_secret_value(SecretId=secret_name)
            return json.loads(response['SecretString'])
        except Exception as e:
            print(f"Failed to get secret from {region}: {e}")
            continue
    
    raise Exception("Failed to retrieve secret from all regions")

# Usage with failover
regions = ['us-east-1', 'us-west-2', 'eu-west-1']
secret = get_secret_multi_region("prod/db/mysql", regions)
```

---

## 8. Access Control and Security

### Resource-Based Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowApplicationAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MyApplicationRole"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }
  ]
}
```

### IAM Policy for Applications
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db/*",
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/api/*"
      ],
      "Condition": {
        "StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }
  ]
}
```

### Encryption Configuration
```bash
# Create secret with custom KMS key
aws secretsmanager create-secret \
  --name "prod/db/mysql" \
  --kms-key-id "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab" \
  --secret-string '{"username":"admin","password":"MyPassword123!"}'
```

---

## 9. Integration Patterns

### RDS Integration
```python
import boto3
import pymysql

def connect_to_rds():
    """Connect to RDS using Secrets Manager"""
    
    # Get database credentials
    secret = get_secret("prod/db/mysql")
    
    # Connect to database
    connection = pymysql.connect(
        host=secret['host'],
        user=secret['username'],
        password=secret['password'],
        database=secret['dbname'],
        port=secret['port']
    )
    
    return connection
```

### Lambda Environment Variables
```python
import os
import boto3
import json

def lambda_handler(event, context):
    """Lambda function using Secrets Manager"""
    
    secret_name = os.environ['SECRET_NAME']
    
    # Get secret
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    secret = json.loads(response['SecretString'])
    
    # Use secret in application logic
    api_key = secret['api_key']
    
    return {'statusCode': 200, 'body': 'Success'}
```

### ECS Task Definition
```json
{
  "family": "my-app",
  "taskRoleArn": "arn:aws:iam::123456789012:role/MyTaskRole",
  "containerDefinitions": [
    {
      "name": "my-container",
      "image": "my-app:latest",
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db/mysql:password::"
        },
        {
          "name": "API_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/api/external:api_key::"
        }
      ]
    }
  ]
}
```

---

## 10. Monitoring and Logging

### CloudTrail Events
```json
{
  "eventTime": "2024-01-15T10:30:00Z",
  "eventName": "GetSecretValue",
  "eventSource": "secretsmanager.amazonaws.com",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AIDACKCEVSQ6C2EXAMPLE",
    "arn": "arn:aws:sts::123456789012:assumed-role/MyRole/MySession"
  },
  "requestParameters": {
    "secretId": "prod/db/mysql",
    "versionStage": "AWSCURRENT"
  },
  "responseElements": null
}
```

### CloudWatch Metrics
- No native Secrets Manager metrics
- Create custom metrics from CloudTrail logs
- Monitor secret access patterns
- Alert on unusual activity

### Custom Monitoring
```python
import boto3
import json

def monitor_secret_access():
    """Monitor secret access patterns"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Put custom metric
    cloudwatch.put_metric_data(
        Namespace='SecretsManager/Usage',
        MetricData=[
            {
                'MetricName': 'SecretAccess',
                'Dimensions': [
                    {
                        'Name': 'SecretName',
                        'Value': 'prod/db/mysql'
                    }
                ],
                'Value': 1,
                'Unit': 'Count'
            }
        ]
    )
```

---

## 11. Cost Optimization

### Pricing Components
- **Secret Storage** - $0.40 per secret per month
- **API Requests** - $0.05 per 10,000 requests
- **Replication** - Additional storage costs per region

### Cost Optimization Strategies
- Consolidate related secrets
- Use secret caching in applications
- Monitor API request patterns
- Clean up unused secrets
- Optimize rotation frequency

### Secret Caching
```python
import boto3
import time
from functools import lru_cache

class SecretCache:
    def __init__(self, cache_ttl=300):  # 5 minutes
        self.cache = {}
        self.cache_ttl = cache_ttl
        self.client = boto3.client('secretsmanager')
    
    def get_secret(self, secret_name):
        """Get secret with caching"""
        current_time = time.time()
        
        # Check cache
        if secret_name in self.cache:
            cached_secret, timestamp = self.cache[secret_name]
            if current_time - timestamp < self.cache_ttl:
                return cached_secret
        
        # Fetch from Secrets Manager
        response = self.client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])
        
        # Update cache
        self.cache[secret_name] = (secret, current_time)
        
        return secret

# Usage
cache = SecretCache(cache_ttl=600)  # 10 minutes
secret = cache.get_secret("prod/db/mysql")
```

---

## 12. Disaster Recovery

### Backup Strategies
- Cross-region replication
- Export secrets for backup
- Document rotation procedures
- Test recovery processes

### Export Secrets
```bash
# Export secret for backup
aws secretsmanager get-secret-value \
  --secret-id "prod/db/mysql" \
  --query SecretString \
  --output text > backup-secret.json
```

### Recovery Procedures
```bash
# Restore secret from backup
aws secretsmanager create-secret \
  --name "prod/db/mysql-restored" \
  --secret-string file://backup-secret.json \
  --kms-key-id alias/aws/secretsmanager
```

---

## 13. Common Exam Scenarios

### Scenario 1: RDS password rotation
**Solution:**
- Create secret for RDS credentials
- Enable automatic rotation with Lambda
- Use AWS-provided rotation templates
- Configure rotation schedule (30-90 days)

### Scenario 2: Application needs database credentials
**Solution:**
- Store credentials in Secrets Manager
- Grant application IAM role GetSecretValue permission
- Implement secret retrieval in application code
- Use connection pooling and caching

### Scenario 3: Multi-region application deployment
**Solution:**
- Enable cross-region replication
- Configure application to use regional endpoints
- Implement failover logic for secret retrieval
- Test disaster recovery scenarios

### Scenario 4: API key rotation for third-party service
**Solution:**
- Create custom Lambda rotation function
- Implement rotation steps (create, set, test, finish)
- Handle API key generation and validation
- Configure rotation schedule

### Scenario 5: Secure CI/CD pipeline credentials
**Solution:**
- Store deployment credentials in Secrets Manager
- Use IAM roles for CodeBuild/CodeDeploy
- Retrieve secrets during build/deployment
- Rotate credentials regularly

---

## 14. Troubleshooting Guide

### Access Denied Errors
- Check IAM permissions for GetSecretValue
- Verify resource-based policy on secret
- Ensure secret exists and is enabled
- Check KMS key permissions for encryption

### Rotation Failures
- Verify Lambda function permissions
- Check Lambda function logs in CloudWatch
- Ensure database connectivity from Lambda
- Validate rotation function implementation

### Secret Not Found
- Verify secret name and region
- Check if secret was deleted
- Ensure proper IAM permissions
- Verify cross-region replication status

---

## 15. CLI Commands Reference

### Secret Management
```bash
# List secrets
aws secretsmanager list-secrets

# Describe secret
aws secretsmanager describe-secret --secret-id "prod/db/mysql"

# Delete secret
aws secretsmanager delete-secret --secret-id "prod/db/mysql" --force-delete-without-recovery

# Restore secret
aws secretsmanager restore-secret --secret-id "prod/db/mysql"
```

### Rotation Management
```bash
# Start rotation
aws secretsmanager rotate-secret --secret-id "prod/db/mysql"

# Update rotation configuration
aws secretsmanager update-secret \
  --secret-id "prod/db/mysql" \
  --rotation-rules AutomaticallyAfterDays=60
```

---

## 16. Exam Tips

### Key Points to Remember
- Secrets Manager is regional service
- Automatic rotation available for RDS, Aurora, DocumentDB, Redshift
- Secrets encrypted at rest with KMS
- Cross-region replication for disaster recovery
- Resource-based policies for fine-grained access control
- Integration with ECS, Lambda, and other services

### Common Mistakes
- Not implementing proper error handling for secret retrieval
- Forgetting to grant KMS permissions for encrypted secrets
- Not using caching for frequently accessed secrets
- Overlooking cross-region replication for DR
- Not monitoring secret access patterns

### Best Practices for Exam
- Understand automatic vs custom rotation
- Know integration patterns with AWS services
- Understand cost optimization strategies
- Know disaster recovery patterns
- Understand access control mechanisms
- Know monitoring and auditing capabilities