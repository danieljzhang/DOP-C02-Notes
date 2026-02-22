# AWS Certificate Manager (ACM) - DOP-C02 Study Notes

## 1. Overview

### What is ACM?
- Managed service for provisioning and managing SSL/TLS certificates
- Free SSL/TLS certificates for AWS services
- Automatic certificate renewal
- Integration with AWS services (CloudFront, ALB, API Gateway)

### Key Benefits
- **Free Certificates** - No cost for AWS-integrated certificates
- **Automatic Renewal** - Certificates renewed automatically before expiration
- **Easy Deployment** - One-click deployment to AWS services
- **Centralized Management** - Single place to manage certificates
- **Security** - Private keys never leave AWS infrastructure

### Use Cases
- HTTPS for web applications
- SSL/TLS termination at load balancers
- API Gateway custom domains
- CloudFront distributions
- Elastic Beanstalk applications

---

## 2. Certificate Types

### Public Certificates
- Issued by Amazon Certificate Authority (CA)
- Free for use with AWS services
- Domain validation (DV) certificates
- Wildcard certificates supported
- Subject Alternative Names (SAN) supported

### Private Certificates
- Issued by AWS Private Certificate Authority (PCA)
- For internal applications and services
- Additional charges apply
- Full control over certificate hierarchy
- Custom certificate policies

### Imported Certificates
- Third-party certificates imported into ACM
- Self-signed certificates
- Certificates from other CAs
- Manual renewal required
- Additional validation steps

---

## 3. Certificate Provisioning

### Request Public Certificate
```bash
# Request certificate with DNS validation
aws acm request-certificate \
  --domain-name "example.com" \
  --subject-alternative-names "*.example.com" "api.example.com" \
  --validation-method DNS \
  --tags Key=Environment,Value=Production
```

### Domain Validation Methods
- **DNS Validation** - Add CNAME record to DNS (recommended)
- **Email Validation** - Receive validation email at domain contacts

### DNS Validation Process
1. Request certificate with domain names
2. ACM provides CNAME records for validation
3. Add CNAME records to DNS zone
4. ACM validates domain ownership
5. Certificate issued and ready for use

### Email Validation Process
1. Request certificate with email validation
2. ACM sends validation emails to domain contacts
3. Click validation link in email
4. Certificate issued after validation

---

## 4. Certificate Management

### List Certificates
```bash
# List all certificates
aws acm list-certificates

# List certificates with specific status
aws acm list-certificates --certificate-statuses ISSUED PENDING_VALIDATION
```

### Describe Certificate
```bash
# Get certificate details
aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"
```

### Certificate Renewal
- **Automatic Renewal** - ACM handles renewal for in-use certificates
- **Manual Renewal** - Required for imported certificates
- **Renewal Timeline** - Attempts renewal 60 days before expiration
- **Validation Required** - DNS/email validation must remain valid

### Certificate Deletion
```bash
# Delete certificate (only if not in use)
aws acm delete-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"
```

---

## 5. Integration with AWS Services

### CloudFront Integration
```bash
# Create CloudFront distribution with ACM certificate
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "my-distribution-2024",
    "Aliases": {
      "Quantity": 1,
      "Items": ["example.com"]
    },
    "DefaultRootObject": "index.html",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "my-origin",
        "DomainName": "my-bucket.s3.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "my-origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "TrustedSigners": {
        "Enabled": false,
        "Quantity": 0
      },
      "ForwardedValues": {
        "QueryString": false,
        "Cookies": {"Forward": "none"}
      }
    },
    "ViewerCertificate": {
      "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012",
      "SSLSupportMethod": "sni-only",
      "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "Enabled": true
  }'
```

### Application Load Balancer Integration
```bash
# Create HTTPS listener with ACM certificate
aws elbv2 create-listener \
  --load-balancer-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/1234567890123456" \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012" \
  --default-actions Type=forward,TargetGroupArn="arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/1234567890123456"
```

### API Gateway Integration
```bash
# Create custom domain for API Gateway
aws apigateway create-domain-name \
  --domain-name "api.example.com" \
  --certificate-arn "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012" \
  --security-policy TLS_1_2
```

---

## 6. Private Certificate Authority (PCA)

### Create Private CA
```bash
# Create private certificate authority
aws acm-pca create-certificate-authority \
  --certificate-authority-configuration '{
    "KeyAlgorithm": "RSA_2048",
    "SigningAlgorithm": "SHA256WITHRSA",
    "Subject": {
      "Country": "US",
      "Organization": "Example Corp",
      "OrganizationalUnit": "IT Department",
      "CommonName": "Example Corp Root CA"
    }
  }' \
  --certificate-authority-type ROOT \
  --tags Key=Environment,Value=Production
```

### Issue Private Certificate
```bash
# Issue certificate from private CA
aws acm-pca issue-certificate \
  --certificate-authority-arn "arn:aws:acm-pca:us-east-1:123456789012:certificate-authority/12345678-1234-1234-1234-123456789012" \
  --csr fileb://certificate-request.csr \
  --signing-algorithm SHA256WITHRSA \
  --validity Value=365,Type=DAYS
```

### Private CA Use Cases
- Internal applications and services
- IoT device certificates
- Code signing certificates
- Custom certificate hierarchies
- Air-gapped environments

---

## 7. Certificate Import

### Import Third-Party Certificate
```bash
# Import certificate from external CA
aws acm import-certificate \
  --certificate fileb://certificate.pem \
  --private-key fileb://private-key.pem \
  --certificate-chain fileb://certificate-chain.pem \
  --tags Key=Source,Value=ThirdParty
```

### Certificate Requirements
- **Certificate Format** - PEM-encoded X.509
- **Private Key** - RSA or ECC private key
- **Certificate Chain** - Intermediate certificates (if any)
- **Key Size** - Minimum 1024-bit RSA or 224-bit ECC

### Import Validation
- Certificate and private key must match
- Certificate must be valid (not expired)
- Certificate chain must be complete
- Private key must not be encrypted

---

## 8. Monitoring and Compliance

### CloudWatch Integration
- Certificate expiration monitoring
- Custom metrics for certificate status
- Alarms for certificate renewal failures

### Certificate Expiration Monitoring
```python
import boto3
from datetime import datetime, timedelta

def check_certificate_expiration():
    """Monitor certificate expiration dates"""
    
    acm = boto3.client('acm')
    cloudwatch = boto3.client('cloudwatch')
    
    # List all certificates
    response = acm.list_certificates(CertificateStatuses=['ISSUED'])
    
    for cert in response['CertificateSummaryList']:
        cert_arn = cert['CertificateArn']
        
        # Get certificate details
        cert_details = acm.describe_certificate(CertificateArn=cert_arn)
        
        # Check expiration
        not_after = cert_details['Certificate']['NotAfter']
        days_until_expiry = (not_after - datetime.now()).days
        
        # Send metric to CloudWatch
        cloudwatch.put_metric_data(
            Namespace='ACM/Certificates',
            MetricData=[
                {
                    'MetricName': 'DaysUntilExpiry',
                    'Dimensions': [
                        {
                            'Name': 'CertificateArn',
                            'Value': cert_arn
                        }
                    ],
                    'Value': days_until_expiry,
                    'Unit': 'Count'
                }
            ]
        )
```

### CloudTrail Integration
- All ACM API calls logged
- Certificate lifecycle events
- Compliance and audit trails

### Config Integration
- Track certificate configuration changes
- Compliance rules for certificate management
- Automated remediation actions

---

## 9. Security Best Practices

### Certificate Security
- Use strong key algorithms (RSA 2048+ or ECC 256+)
- Enable perfect forward secrecy
- Use modern TLS versions (1.2+)
- Implement HSTS headers
- Regular security assessments

### Access Control
- Use IAM policies for ACM access
- Principle of least privilege
- Separate roles for certificate management
- Monitor certificate operations

### Operational Security
- Automate certificate deployment
- Monitor certificate expiration
- Implement certificate rotation procedures
- Document certificate management processes

---

## 10. Automation and DevOps

### Infrastructure as Code
```yaml
# CloudFormation template for ACM certificate
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  SSLCertificate:
    Type: AWS::CertificateManager::Certificate
    Properties:
      DomainName: example.com
      SubjectAlternativeNames:
        - '*.example.com'
        - api.example.com
      ValidationMethod: DNS
      DomainValidationOptions:
        - DomainName: example.com
          HostedZoneId: !Ref HostedZone
        - DomainName: '*.example.com'
          HostedZoneId: !Ref HostedZone
      Tags:
        - Key: Environment
          Value: Production
```

### Terraform Configuration
```hcl
# Terraform configuration for ACM certificate
resource "aws_acm_certificate" "example" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com", "api.example.com"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Environment = "Production"
  }
}

resource "aws_acm_certificate_validation" "example" {
  certificate_arn         = aws_acm_certificate.example.arn
  validation_record_fqdns = [for record in aws_route53_record.example : record.fqdn]
}
```

### Automated Certificate Deployment
```python
import boto3

def deploy_certificate_to_alb(cert_arn, alb_arn):
    """Deploy certificate to Application Load Balancer"""
    
    elbv2 = boto3.client('elbv2')
    
    # Get existing listeners
    listeners = elbv2.describe_listeners(LoadBalancerArn=alb_arn)
    
    # Update HTTPS listener with new certificate
    for listener in listeners['Listeners']:
        if listener['Protocol'] == 'HTTPS':
            elbv2.modify_listener(
                ListenerArn=listener['ListenerArn'],
                Certificates=[{'CertificateArn': cert_arn}]
            )
            print(f"Updated listener {listener['ListenerArn']} with certificate {cert_arn}")
```

---

## 11. Cost Optimization

### ACM Pricing
- **Public Certificates** - Free for AWS services
- **Private Certificates** - Charged per certificate issued
- **Private CA** - Monthly fee per CA
- **API Calls** - No additional charges

### Cost Optimization Strategies
- Use public certificates when possible
- Consolidate domains with wildcard certificates
- Monitor private certificate usage
- Automate certificate lifecycle management
- Regular cleanup of unused certificates

---

## 12. Multi-Region Considerations

### Regional Availability
- ACM is available in most AWS regions
- Certificates are region-specific
- CloudFront requires certificates in us-east-1
- Cross-region certificate replication not supported

### Global Applications
```bash
# Request certificate in us-east-1 for CloudFront
aws acm request-certificate \
  --region us-east-1 \
  --domain-name "global.example.com" \
  --validation-method DNS

# Request certificate in application region for ALB
aws acm request-certificate \
  --region us-west-2 \
  --domain-name "app.example.com" \
  --validation-method DNS
```

---

## 13. Common Exam Scenarios

### Scenario 1: Enable HTTPS for web application
**Solution:**
- Request ACM certificate for domain
- Use DNS validation method
- Deploy certificate to ALB or CloudFront
- Configure security groups and listeners

### Scenario 2: Wildcard certificate for subdomains
**Solution:**
- Request certificate with wildcard domain (*.example.com)
- Include root domain as SAN if needed
- Use DNS validation for automation
- Deploy to multiple services as needed

### Scenario 3: Certificate for CloudFront distribution
**Solution:**
- Request certificate in us-east-1 region
- Use DNS validation method
- Configure CloudFront with certificate ARN
- Set up proper CNAME records

### Scenario 4: Private certificates for internal services
**Solution:**
- Create private certificate authority
- Issue certificates from private CA
- Deploy certificates to internal applications
- Configure trust stores on clients

### Scenario 5: Automated certificate renewal monitoring
**Solution:**
- Set up CloudWatch alarms for expiration
- Create Lambda function for monitoring
- Implement SNS notifications
- Automate certificate replacement process

---

## 14. Troubleshooting Guide

### Certificate Request Issues
- Verify domain ownership and DNS configuration
- Check validation method requirements
- Ensure proper IAM permissions
- Validate domain name format

### Validation Failures
- Verify DNS records are correctly configured
- Check email delivery for email validation
- Ensure domain is publicly resolvable
- Wait for DNS propagation (up to 72 hours)

### Deployment Issues
- Verify certificate is in correct region
- Check service-specific requirements
- Ensure certificate status is "Issued"
- Validate certificate ARN format

### Renewal Problems
- Ensure certificate is in use by AWS service
- Verify DNS validation records remain valid
- Check for domain ownership changes
- Monitor CloudWatch for renewal events

---

## 15. CLI Commands Reference

### Certificate Operations
```bash
# Request certificate
aws acm request-certificate \
  --domain-name example.com \
  --validation-method DNS

# List certificates
aws acm list-certificates

# Describe certificate
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:region:account:certificate/cert-id

# Import certificate
aws acm import-certificate \
  --certificate fileb://cert.pem \
  --private-key fileb://key.pem

# Delete certificate
aws acm delete-certificate \
  --certificate-arn arn:aws:acm:region:account:certificate/cert-id
```

### Validation Operations
```bash
# Get certificate validation records
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:region:account:certificate/cert-id \
  --query 'Certificate.DomainValidationOptions'

# Resend validation email
aws acm resend-validation-email \
  --certificate-arn arn:aws:acm:region:account:certificate/cert-id \
  --domain example.com \
  --validation-domain example.com
```

---

## 16. Exam Tips

### Key Points to Remember
- ACM certificates are free for AWS services
- Certificates are region-specific (except CloudFront uses us-east-1)
- Automatic renewal only works for certificates in use
- DNS validation is preferred over email validation
- Private CA incurs additional charges
- Wildcard certificates support unlimited subdomains

### Common Mistakes
- Requesting certificate in wrong region for CloudFront
- Not maintaining DNS validation records
- Forgetting to deploy certificate to AWS service
- Not monitoring certificate expiration for imported certificates
- Incorrect domain name format in certificate request

### Best Practices for Exam
- Understand regional requirements for different services
- Know validation methods and their use cases
- Understand automatic renewal requirements
- Know integration patterns with AWS services
- Understand private CA use cases and pricing
- Know monitoring and automation patterns