# AWS S3 Cross-Region Replication - DOP-C02 Study Notes

## 1. Overview

### What is S3 Cross-Region Replication (CRR)?
- Automatic, asynchronous replication of objects across AWS regions
- Provides disaster recovery and compliance capabilities
- Reduces latency for global applications
- Maintains object metadata and access control lists

### Key Benefits
- **Disaster Recovery** - Geographic separation of data
- **Compliance** - Meet regulatory requirements for data locality
- **Latency Reduction** - Serve content from nearest region
- **Operational Efficiency** - Centralized data management

---

## 2. Replication Configuration

### Basic CRR Setup
```bash
# Create replication configuration
aws s3api put-bucket-replication \
  --bucket "source-bucket-us-east-1" \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
    "Rules": [
      {
        "ID": "ReplicateToWest",
        "Status": "Enabled",
        "Priority": 1,
        "Filter": {
          "Prefix": "documents/"
        },
        "Destination": {
          "Bucket": "arn:aws:s3:::destination-bucket-us-west-2",
          "StorageClass": "STANDARD_IA"
        }
      }
    ]
  }'
```

### Advanced Replication Rules
```python
def create_advanced_replication_config():
    """Create advanced S3 replication configuration"""
    
    s3 = boto3.client('s3')
    
    replication_config = {
        'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
        'Rules': [
            {
                'ID': 'ReplicateDocuments',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'And': {
                        'Prefix': 'documents/',
                        'Tags': [
                            {
                                'Key': 'Replicate',
                                'Value': 'true'
                            }
                        ]
                    }
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::backup-bucket-eu-west-1',
                    'StorageClass': 'GLACIER',
                    'EncryptionConfiguration': {
                        'ReplicaKmsKeyID': 'arn:aws:kms:eu-west-1:123456789012:key/12345678-1234-1234-1234-123456789012'
                    },
                    'ReplicationTime': {
                        'Status': 'Enabled',
                        'Time': {
                            'Minutes': 15
                        }
                    },
                    'Metrics': {
                        'Status': 'Enabled',
                        'EventThreshold': {
                            'Minutes': 15
                        }
                    }
                }
            },
            {
                'ID': 'ReplicateImages',
                'Status': 'Enabled',
                'Priority': 2,
                'Filter': {
                    'Prefix': 'images/'
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::cdn-bucket-ap-southeast-1',
                    'StorageClass': 'STANDARD',
                    'AccessControlTranslation': {
                        'Owner': 'Destination'
                    }
                }
            }
        ]
    }
    
    response = s3.put_bucket_replication(
        Bucket='source-bucket-us-east-1',
        ReplicationConfiguration=replication_config
    )
    
    return response
```

---

## 3. IAM Role and Permissions

### Replication Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Replication Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::source-bucket-us-east-1/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::source-bucket-us-east-1"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::destination-bucket-us-west-2/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/source-key-id",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-west-2:123456789012:key/destination-key-id",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-west-2.amazonaws.com"
        }
      }
    }
  ]
}
```

---

## 4. Same-Region Replication (SRR)

### SRR Configuration
```python
def setup_same_region_replication():
    """Set up Same-Region Replication for compliance"""
    
    s3 = boto3.client('s3')
    
    srr_config = {
        'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
        'Rules': [
            {
                'ID': 'ComplianceReplication',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'Tag': {
                        'Key': 'Compliance',
                        'Value': 'Required'
                    }
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::compliance-backup-bucket',
                    'StorageClass': 'STANDARD_IA',
                    'EncryptionConfiguration': {
                        'ReplicaKmsKeyID': 'arn:aws:kms:us-east-1:123456789012:key/compliance-key-id'
                    }
                }
            }
        ]
    }
    
    response = s3.put_bucket_replication(
        Bucket='production-data-bucket',
        ReplicationConfiguration=srr_config
    )
    
    return response
```

---

## 5. Replication Time Control (RTC)

### RTC Configuration
```python
def enable_replication_time_control():
    """Enable Replication Time Control for predictable replication"""
    
    s3 = boto3.client('s3')
    
    rtc_config = {
        'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
        'Rules': [
            {
                'ID': 'RTCReplication',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'Prefix': 'critical-data/'
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::dr-bucket-us-west-2',
                    'StorageClass': 'STANDARD',
                    'ReplicationTime': {
                        'Status': 'Enabled',
                        'Time': {
                            'Minutes': 15
                        }
                    },
                    'Metrics': {
                        'Status': 'Enabled',
                        'EventThreshold': {
                            'Minutes': 15
                        }
                    }
                }
            }
        ]
    }
    
    response = s3.put_bucket_replication(
        Bucket='critical-data-bucket',
        ReplicationConfiguration=rtc_config
    )
    
    return response
```

---

## 6. Monitoring Replication

### CloudWatch Metrics
```python
def monitor_replication_metrics():
    """Monitor S3 replication metrics"""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Create alarm for replication failures
    cloudwatch.put_metric_alarm(
        AlarmName='S3-ReplicationFailures',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=1,
        MetricName='ReplicationLatency',
        Namespace='AWS/S3',
        Period=300,
        Statistic='Maximum',
        Threshold=900,  # 15 minutes
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:s3-replication-alerts'
        ],
        Dimensions=[
            {
                'Name': 'SourceBucket',
                'Value': 'source-bucket-us-east-1'
            },
            {
                'Name': 'DestinationBucket',
                'Value': 'destination-bucket-us-west-2'
            }
        ]
    )
    
    # Create alarm for replication rule status
    cloudwatch.put_metric_alarm(
        AlarmName='S3-ReplicationRuleFailures',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        MetricName='ReplicationRuleFailures',
        Namespace='AWS/S3',
        Period=300,
        Statistic='Sum',
        Threshold=0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:s3-replication-alerts'
        ]
    )
```

### Replication Status Tracking
```python
def check_replication_status():
    """Check replication status for objects"""
    
    s3 = boto3.client('s3')
    
    # List objects and check replication status
    response = s3.list_objects_v2(
        Bucket='source-bucket-us-east-1',
        Prefix='documents/'
    )
    
    replication_status = {}
    
    for obj in response.get('Contents', []):
        key = obj['Key']
        
        # Get object replication status
        obj_response = s3.head_object(
            Bucket='source-bucket-us-east-1',
            Key=key
        )
        
        replication_status[key] = {
            'replication_status': obj_response.get('ReplicationStatus', 'PENDING'),
            'last_modified': obj['LastModified'],
            'size': obj['Size']
        }
    
    return replication_status
```

---

## 7. Bi-Directional Replication

### Two-Way Replication Setup
```python
def setup_bidirectional_replication():
    """Set up bi-directional replication between regions"""
    
    s3_us_east = boto3.client('s3', region_name='us-east-1')
    s3_us_west = boto3.client('s3', region_name='us-west-2')
    
    # Replication from US East to US West
    east_to_west_config = {
        'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
        'Rules': [
            {
                'ID': 'EastToWest',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'Prefix': 'shared-data/'
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::shared-bucket-us-west-2',
                    'StorageClass': 'STANDARD'
                }
            }
        ]
    }
    
    # Replication from US West to US East
    west_to_east_config = {
        'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
        'Rules': [
            {
                'ID': 'WestToEast',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'Prefix': 'shared-data/'
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::shared-bucket-us-east-1',
                    'StorageClass': 'STANDARD'
                }
            }
        ]
    }
    
    # Apply configurations
    s3_us_east.put_bucket_replication(
        Bucket='shared-bucket-us-east-1',
        ReplicationConfiguration=east_to_west_config
    )
    
    s3_us_west.put_bucket_replication(
        Bucket='shared-bucket-us-west-2',
        ReplicationConfiguration=west_to_east_config
    )
```

---

## 8. Disaster Recovery Patterns

### Multi-Region Backup Strategy
```python
def implement_multi_region_backup():
    """Implement comprehensive multi-region backup strategy"""
    
    regions = ['us-east-1', 'us-west-2', 'eu-west-1']
    primary_region = 'us-east-1'
    
    for backup_region in [r for r in regions if r != primary_region]:
        s3_primary = boto3.client('s3', region_name=primary_region)
        
        # Create replication configuration for each backup region
        replication_config = {
            'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
            'Rules': [
                {
                    'ID': f'BackupTo{backup_region.replace("-", "")}',
                    'Status': 'Enabled',
                    'Priority': 1,
                    'Filter': {},  # Replicate all objects
                    'Destination': {
                        'Bucket': f'arn:aws:s3:::backup-bucket-{backup_region}',
                        'StorageClass': 'GLACIER',
                        'ReplicationTime': {
                            'Status': 'Enabled',
                            'Time': {'Minutes': 15}
                        }
                    }
                }
            ]
        }
        
        s3_primary.put_bucket_replication(
            Bucket='production-data-bucket',
            ReplicationConfiguration=replication_config
        )
```

### Failover Automation
```python
def automate_s3_failover():
    """Automate S3 failover for disaster recovery"""
    
    def lambda_handler(event, context):
        """Lambda function for S3 failover automation"""
        
        # Parse CloudWatch alarm or manual trigger
        primary_region = 'us-east-1'
        failover_region = 'us-west-2'
        
        route53 = boto3.client('route53')
        
        # Update Route 53 records to point to failover region
        response = route53.change_resource_record_sets(
            HostedZoneId='Z123456789',
            ChangeBatch={
                'Changes': [{
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'data.example.com',
                        'Type': 'CNAME',
                        'TTL': 60,
                        'ResourceRecords': [
                            {'Value': f's3-{failover_region}.amazonaws.com'}
                        ]
                    }
                }]
            }
        )
        
        # Send notification
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:disaster-recovery',
            Message=f'S3 failover completed to {failover_region}',
            Subject='S3 Disaster Recovery Activated'
        )
        
        return {'statusCode': 200, 'body': 'Failover completed'}
```

---

## 9. Cost Optimization

### Storage Class Optimization
```python
def optimize_replication_costs():
    """Optimize replication costs with appropriate storage classes"""
    
    s3 = boto3.client('s3')
    
    # Different storage classes for different data types
    cost_optimized_config = {
        'Role': 'arn:aws:iam::123456789012:role/S3ReplicationRole',
        'Rules': [
            {
                'ID': 'FrequentAccessData',
                'Status': 'Enabled',
                'Priority': 1,
                'Filter': {
                    'Tag': {
                        'Key': 'AccessPattern',
                        'Value': 'Frequent'
                    }
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::backup-bucket-us-west-2',
                    'StorageClass': 'STANDARD'
                }
            },
            {
                'ID': 'InfrequentAccessData',
                'Status': 'Enabled',
                'Priority': 2,
                'Filter': {
                    'Tag': {
                        'Key': 'AccessPattern',
                        'Value': 'Infrequent'
                    }
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::backup-bucket-us-west-2',
                    'StorageClass': 'STANDARD_IA'
                }
            },
            {
                'ID': 'ArchiveData',
                'Status': 'Enabled',
                'Priority': 3,
                'Filter': {
                    'Tag': {
                        'Key': 'AccessPattern',
                        'Value': 'Archive'
                    }
                },
                'Destination': {
                    'Bucket': 'arn:aws:s3:::backup-bucket-us-west-2',
                    'StorageClass': 'GLACIER'
                }
            }
        ]
    }
    
    response = s3.put_bucket_replication(
        Bucket='source-bucket-us-east-1',
        ReplicationConfiguration=cost_optimized_config
    )
    
    return response
```

---

## 10. Common Exam Scenarios

### Scenario 1: Disaster recovery for critical data
**Solution:**
- Cross-region replication to geographically separated region
- Replication Time Control for predictable recovery times
- Monitoring and alerting for replication failures
- Automated failover procedures

### Scenario 2: Global content distribution
**Solution:**
- Multi-region replication for reduced latency
- CloudFront integration for edge caching
- Intelligent tiering for cost optimization
- Regional access patterns analysis

### Scenario 3: Compliance and data sovereignty
**Solution:**
- Same-region replication for compliance backup
- Encryption in transit and at rest
- Access logging and audit trails
- Data residency controls

### Scenario 4: Development and testing environments
**Solution:**
- Cross-region replication for test data
- Lifecycle policies for cost management
- Selective replication based on tags
- Automated environment provisioning

### Scenario 5: Backup and archival strategy
**Solution:**
- Multi-tier replication with different storage classes
- Long-term retention with Glacier
- Point-in-time recovery capabilities
- Cost-optimized storage transitions

---

## 11. CLI Commands Reference

### Replication Configuration
```bash
# Put replication configuration
aws s3api put-bucket-replication \
  --bucket "source-bucket" \
  --replication-configuration file://replication-config.json

# Get replication configuration
aws s3api get-bucket-replication \
  --bucket "source-bucket"

# Delete replication configuration
aws s3api delete-bucket-replication \
  --bucket "source-bucket"
```

### Monitoring Commands
```bash
# Get replication metrics
aws s3api get-bucket-metrics-configuration \
  --bucket "source-bucket" \
  --id "replication-metrics"

# List multipart uploads (for troubleshooting)
aws s3api list-multipart-uploads \
  --bucket "source-bucket"
```

---

## 12. Exam Tips

### Key Points to Remember
- Replication requires versioning enabled on both source and destination buckets
- Objects existing before replication configuration are not replicated
- Delete markers and deletes of specific versions are not replicated by default
- Replication is asynchronous and eventual consistency applies
- Cross-region replication incurs data transfer charges
- Same-region replication is useful for compliance and backup

### Common Mistakes
- Forgetting to enable versioning before setting up replication
- Not configuring proper IAM permissions for replication role
- Overlooking KMS key permissions for encrypted objects
- Not monitoring replication status and failures
- Assuming immediate replication completion

### Best Practices for Exam
- Understand replication requirements and limitations
- Know IAM role and policy requirements
- Understand monitoring and troubleshooting approaches
- Know cost implications and optimization strategies
- Understand integration with other AWS services
- Know disaster recovery and failover patterns