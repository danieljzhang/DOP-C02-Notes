# AWS KMS (Key Management Service) - DOP-C02 Study Notes

## 1. Overview

### What is KMS?
- Managed service for creating and controlling encryption keys
- Integrates with most AWS services for encryption
- FIPS 140-2 Level 2 validated hardware security modules (HSMs)
- Regional service with global key management capabilities

### Key Benefits
- **Centralized Key Management** - Single place to manage encryption keys
- **Integration** - Native integration with AWS services
- **Compliance** - Meets various compliance requirements
- **Audit Trail** - All key usage logged in CloudTrail
- **Cost Effective** - Pay per use model

### Use Cases
- Encrypt data at rest in AWS services
- Encrypt data in transit
- Digital signing and verification
- Generate random data
- Compliance requirements (HIPAA, PCI DSS, etc.)

---

## 2. KMS Key Types

### Customer Managed Keys (CMK)
- Created and managed by you
- Full control over key policies and permissions
- Can be enabled/disabled, rotated, deleted
- Used for most encryption scenarios

### AWS Managed Keys
- Created and managed by AWS services
- Used automatically by AWS services
- Cannot be deleted or disabled
- Automatic rotation every year
- Key alias format: `aws/service-name`

### AWS Owned Keys
- Owned and managed by AWS
- Used by AWS services for internal operations
- Not visible in your account
- No additional charges

### Key Specifications
- **Symmetric Keys** - Same key for encrypt/decrypt (AES-256)
- **Asymmetric Keys** - Key pair for encrypt/decrypt or sign/verify
- **HMAC Keys** - Hash-based message authentication codes

---

## 3. Key Creation and Management

### Create Symmetric Key
```bash
# Create customer managed key
aws kms create-key \
  --description "My encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT

# Create key alias
aws kms create-alias \
  --alias-name alias/my-key \
  --target-key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```

### Create Asymmetric Key
```bash
# Create RSA key for encryption
aws kms create-key \
  --description "RSA encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec RSA_2048

# Create ECC key for signing
aws kms create-key \
  --description "ECC signing key" \
  --key-usage SIGN_VERIFY \
  --key-spec ECC_NIST_P256
```

### Key Policy Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow use of the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MyRole"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 4. Encryption Operations

### Data Encryption
```bash
# Encrypt small data (up to 4KB)
aws kms encrypt \
  --key-id alias/my-key \
  --plaintext "Hello World" \
  --output text \
  --query CiphertextBlob

# Decrypt data
aws kms decrypt \
  --ciphertext-blob fileb://encrypted-data \
  --output text \
  --query Plaintext | base64 -d
```

### Data Key Operations
```bash
# Generate data key for large data encryption
aws kms generate-data-key \
  --key-id alias/my-key \
  --key-spec AES_256

# Generate data key without plaintext
aws kms generate-data-key-without-plaintext \
  --key-id alias/my-key \
  --key-spec AES_256
```

### Envelope Encryption Pattern
1. Generate data key using KMS
2. Use plaintext data key to encrypt large data
3. Store encrypted data key with encrypted data
4. Discard plaintext data key from memory
5. To decrypt: decrypt data key first, then use it to decrypt data

---

## 5. Key Rotation

### Automatic Rotation
- Available for customer managed symmetric keys
- Rotates key material annually
- Old key material retained for decryption
- No impact on applications (same key ID/ARN)

```bash
# Enable automatic rotation
aws kms enable-key-rotation --key-id alias/my-key

# Check rotation status
aws kms get-key-rotation-status --key-id alias/my-key
```

### Manual Rotation
- Create new key with new key material
- Update applications to use new key
- Keep old key for decrypting existing data
- More control but requires application changes

### Rotation Best Practices
- Enable automatic rotation for symmetric keys
- Plan manual rotation for asymmetric keys
- Test rotation process in non-production
- Monitor rotation events in CloudTrail
- Update key aliases to point to new keys

---

## 6. Cross-Account Access

### Cross-Account Key Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow external account access",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

### Cross-Account Grants
```bash
# Create grant for cross-account access
aws kms create-grant \
  --key-id arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab \
  --grantee-principal arn:aws:iam::111122223333:role/MyRole \
  --operations Encrypt Decrypt GenerateDataKey
```

### Cross-Region Considerations
- KMS keys are regional resources
- Cannot use key from one region in another
- Must copy encrypted data and re-encrypt with regional key
- Use AWS services that handle cross-region encryption

---

## 7. Integration with AWS Services

### S3 Integration
```bash
# Upload with KMS encryption
aws s3 cp file.txt s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id alias/my-key

# Set default encryption on bucket
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/my-key"
      }
    }]
  }'
```

### EBS Integration
```bash
# Create encrypted EBS volume
aws ec2 create-volume \
  --size 10 \
  --volume-type gp3 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id alias/my-key
```

### RDS Integration
```bash
# Create encrypted RDS instance
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --storage-encrypted \
  --kms-key-id alias/my-key
```

### Lambda Integration
```python
import boto3
import os

# Lambda function with KMS encrypted environment variables
def lambda_handler(event, context):
    # Decrypt environment variable
    kms = boto3.client('kms')
    encrypted_value = os.environ['ENCRYPTED_VALUE']
    
    decrypted = kms.decrypt(CiphertextBlob=encrypted_value)
    plaintext_value = decrypted['Plaintext'].decode('utf-8')
    
    return {'statusCode': 200, 'body': 'Success'}
```

---

## 8. KMS Grants

### What are Grants?
- Alternative to key policies for temporary access
- Programmatically created and managed
- Can be constrained by conditions
- Automatically cleaned up when no longer needed

### Grant Operations
```bash
# Create grant
aws kms create-grant \
  --key-id alias/my-key \
  --grantee-principal arn:aws:iam::123456789012:role/MyRole \
  --operations Encrypt Decrypt GenerateDataKey \
  --constraints EncryptionContextSubset='{Department=Finance}'

# List grants
aws kms list-grants --key-id alias/my-key

# Retire grant
aws kms retire-grant \
  --key-id alias/my-key \
  --grant-token AQpAM2RhZTk1MGMyNzEyMWI2N2VjODg4MzMzMzMzMzMzMzM
```

### Grant Constraints
- **EncryptionContextEquals** - Exact match required
- **EncryptionContextSubset** - Subset match required
- Used to limit grant usage to specific contexts

---

## 9. Encryption Context

### What is Encryption Context?
- Additional authenticated data (AAD)
- Key-value pairs that provide additional security
- Not secret but provides integrity protection
- Logged in CloudTrail for auditing

### Using Encryption Context
```bash
# Encrypt with context
aws kms encrypt \
  --key-id alias/my-key \
  --plaintext "sensitive data" \
  --encryption-context Department=Finance,Project=Alpha

# Decrypt requires same context
aws kms decrypt \
  --ciphertext-blob fileb://encrypted-data \
  --encryption-context Department=Finance,Project=Alpha
```

### Best Practices
- Use meaningful key-value pairs
- Include resource identifiers
- Don't include sensitive information
- Use for access control and auditing
- Consistent naming conventions

---

## 10. Multi-Region Keys

### What are Multi-Region Keys?
- Primary key in one region with replica keys in other regions
- Same key material and key ID across regions
- Encrypt in one region, decrypt in another
- Useful for global applications and disaster recovery

### Create Multi-Region Key
```bash
# Create multi-region key
aws kms create-key \
  --description "Multi-region key" \
  --multi-region \
  --key-usage ENCRYPT_DECRYPT

# Replicate to another region
aws kms replicate-key \
  --key-id arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab \
  --replica-region us-west-2 \
  --description "Replica in us-west-2"
```

### Multi-Region Use Cases
- Global applications with regional data
- Cross-region backup and disaster recovery
- Content distribution networks
- Multi-region database replication

---

## 11. Custom Key Stores

### CloudHSM Key Store
- Use AWS CloudHSM cluster as key store
- FIPS 140-2 Level 3 validation
- Single-tenant hardware security modules
- Full control over HSM and key material

### External Key Store (XKS)
- Use external key management system
- Keys remain in your infrastructure
- KMS proxies requests to external system
- Ultimate control over key material

### Setup CloudHSM Key Store
```bash
# Create custom key store
aws kms create-custom-key-store \
  --custom-key-store-name MyCloudHSMKeyStore \
  --cloud-hsm-cluster-id cluster-1234567890abcdef0 \
  --trust-anchor-certificate file://customerCA.crt \
  --key-store-password MyPassword123!
```

---

## 12. Monitoring and Logging

### CloudTrail Integration
- All KMS API calls logged
- Key usage events recorded
- Encryption context included in logs
- Cross-account access visibility

### Key Usage Monitoring
```json
{
  "eventTime": "2024-01-15T10:30:00Z",
  "eventName": "Decrypt",
  "eventSource": "kms.amazonaws.com",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AIDACKCEVSQ6C2EXAMPLE",
    "arn": "arn:aws:sts::123456789012:assumed-role/MyRole/MySession"
  },
  "requestParameters": {
    "keyId": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
    "encryptionContext": {
      "Department": "Finance"
    }
  }
}
```

### CloudWatch Metrics
- No native KMS metrics
- Create custom metrics from CloudTrail logs
- Monitor key usage patterns
- Alert on unusual activity

---

## 13. Security Best Practices

### Key Management
- Use customer managed keys for sensitive data
- Enable automatic rotation when possible
- Implement least privilege access
- Use encryption context for additional security
- Regular key policy reviews

### Access Control
- Use IAM policies and key policies together
- Implement separation of duties
- Use grants for temporary access
- Monitor cross-account access
- Audit key usage regularly

### Operational Security
- Enable CloudTrail logging
- Monitor key usage patterns
- Implement key lifecycle management
- Test disaster recovery procedures
- Document key management procedures

---

## 14. Cost Optimization

### KMS Pricing Components
- **Customer Managed Keys** - $1/month per key
- **API Requests** - $0.03 per 10,000 requests
- **AWS Managed Keys** - Free (requests still charged)
- **Custom Key Stores** - Additional HSM costs

### Cost Optimization Strategies
- Use AWS managed keys when appropriate
- Consolidate keys where possible
- Monitor API request patterns
- Use data key caching
- Implement efficient key rotation

### Data Key Caching
```python
import boto3
from aws_encryption_sdk import encrypt, decrypt
from aws_encryption_sdk.caches import LocalCryptoMaterialsCache
from aws_encryption_sdk.key_providers.kms import KMSMasterKeyProvider

# Setup caching to reduce KMS API calls
cache = LocalCryptoMaterialsCache(capacity=100)
key_provider = KMSMasterKeyProvider(key_ids=['alias/my-key'])

# Encrypt with caching
ciphertext, _ = encrypt(
    source=b'my data',
    key_provider=key_provider,
    materials_manager=cache
)
```

---

## 15. Common Exam Scenarios

### Scenario 1: Cross-account S3 bucket encryption
**Solution:**
- Create KMS key in bucket owner account
- Add cross-account permissions to key policy
- Grant external account access to key operations
- Configure S3 bucket policy for cross-account access

### Scenario 2: Lambda function needs to decrypt data
**Solution:**
- Attach IAM role to Lambda with KMS permissions
- Grant decrypt permissions for specific key
- Use encryption context for additional security
- Handle decryption errors gracefully

### Scenario 3: Multi-region application encryption
**Solution:**
- Create multi-region KMS key
- Replicate key to required regions
- Use same key ID across regions
- Handle regional failover scenarios

### Scenario 4: Compliance requires key rotation
**Solution:**
- Enable automatic rotation for symmetric keys
- Plan manual rotation for asymmetric keys
- Document rotation procedures
- Test rotation impact on applications

### Scenario 5: Need to audit key usage
**Solution:**
- Enable CloudTrail logging
- Monitor KMS API calls
- Use encryption context for tracking
- Create CloudWatch alarms for unusual activity

---

## 16. Troubleshooting Guide

### Access Denied Errors
- Check IAM permissions for user/role
- Verify key policy allows the operation
- Ensure key is enabled and available
- Check encryption context requirements
- Verify cross-account trust relationships

### Key Not Found Errors
- Verify key exists in correct region
- Check key alias spelling and format
- Ensure key hasn't been deleted
- Verify account has access to key

### Encryption/Decryption Failures
- Check key usage permissions
- Verify encryption context matches
- Ensure key is enabled
- Check for key rotation issues
- Verify data key generation permissions

---

## 17. CLI Commands Reference

### Key Management
```bash
# List keys
aws kms list-keys

# Describe key
aws kms describe-key --key-id alias/my-key

# Enable key
aws kms enable-key --key-id alias/my-key

# Disable key
aws kms disable-key --key-id alias/my-key

# Schedule key deletion
aws kms schedule-key-deletion --key-id alias/my-key --pending-window-in-days 30
```

### Encryption Operations
```bash
# Encrypt data
aws kms encrypt --key-id alias/my-key --plaintext "Hello World"

# Decrypt data
aws kms decrypt --ciphertext-blob fileb://encrypted-data

# Generate data key
aws kms generate-data-key --key-id alias/my-key --key-spec AES_256

# Generate random data
aws kms generate-random --number-of-bytes 32
```

---

## 18. Exam Tips

### Key Points to Remember
- KMS is regional service
- Customer managed keys cost $1/month
- Automatic rotation available for symmetric keys only
- Encryption context provides additional security
- Multi-region keys have same key ID across regions
- AWS managed keys rotate automatically every year

### Common Mistakes
- Forgetting to enable key in key policy
- Not understanding encryption context requirements
- Mixing up symmetric vs asymmetric key capabilities
- Not considering cross-region key availability
- Overlooking IAM and key policy interaction

### Best Practices for Exam
- Understand envelope encryption pattern
- Know when to use customer vs AWS managed keys
- Understand cross-account access patterns
- Know multi-region key use cases
- Understand grant vs policy differences
- Know integration patterns with AWS services