# Domain 3: Resilient Cloud Solutions - DOP-C02 Cheatsheet

## Weight: 15% | Focus: High Availability, Auto Scaling, Load Balancing, Multi-AZ

---

## 🔥 **MUST KNOW Services**

### **Auto Scaling** - Dynamic Capacity
- **ASG**: EC2 instances across AZs
- **Scaling Policies**: Target Tracking, Step, Simple
- **Health Checks**: EC2, ELB
- **Lifecycle Hooks**: Custom launch/terminate actions

### **Elastic Load Balancer** - Traffic Distribution
- **ALB**: Layer 7 (HTTP/HTTPS), path/host routing
- **NLB**: Layer 4 (TCP/UDP), static IPs, high performance
- **GWLB**: Layer 3, third-party appliances
- **Health Checks**: Determine target availability

### **RDS Multi-AZ** - Database HA
- **Multi-AZ**: Synchronous replication, automatic failover
- **Read Replicas**: Asynchronous, read scaling
- **Cross-Region**: Disaster recovery
- **Failover**: 60-120 seconds

### **Route 53** - DNS & Global Load Balancing
- **Routing Policies**: Simple, Weighted, Latency, Failover, Geolocation
- **Health Checks**: HTTP/HTTPS/TCP monitoring
- **Alias Records**: Free, automatic IP resolution

### **S3 Cross-Region Replication** - Data Resilience
- **CRR**: Cross-region disaster recovery
- **SRR**: Same-region compliance
- **RTC**: 15-minute replication SLA

---

## ⚡ **Key Patterns**

### **Multi-AZ Web Application**
```
Route 53 → CloudFront → ALB (Multi-AZ) → ASG (Multi-AZ) → RDS Multi-AZ
```

### **Auto Scaling Policies**
- **Target Tracking**: Maintain specific metric value
- **Step Scaling**: Scale based on metric ranges
- **Predictive**: ML-based forecasting

### **RDS Failover Scenarios**
- Primary DB failure → Automatic failover to standby
- AZ outage → Failover within 60-120 seconds
- Maintenance → Zero-downtime updates

---

## 🎯 **Exam Scenarios**

### **Scenario 1: Application not scaling properly**
- Check Auto Scaling policies and thresholds
- Verify CloudWatch metrics are being published
- Review health check configuration
- Check cooldown periods

### **Scenario 2: Database failover taking too long**
- Verify Multi-AZ is enabled
- Check application connection string (use endpoint, not IP)
- Review security group rules
- Ensure proper error handling in application

### **Scenario 3: Global application latency issues**
- Implement Route 53 latency-based routing
- Use CloudFront for static content
- Deploy read replicas in multiple regions
- Consider Aurora Global Database

---

## 📋 **Quick Commands**

```bash
# Auto Scaling
aws autoscaling create-auto-scaling-group --auto-scaling-group-name MyASG
aws autoscaling put-scaling-policy --policy-name ScaleUp --auto-scaling-group-name MyASG

# Load Balancer
aws elbv2 create-load-balancer --name MyALB --subnets subnet-12345 subnet-67890
aws elbv2 create-target-group --name MyTargets --protocol HTTP --port 80

# RDS
aws rds create-db-instance --db-instance-identifier MyDB --multi-az
aws rds reboot-db-instance --db-instance-identifier MyDB --force-failover

# Route 53
aws route53 create-health-check --type HTTP --resource-path /health
aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch file://change.json

# S3 Replication
aws s3api put-bucket-replication --bucket MyBucket --replication-configuration file://replication.json
```

---

## 🏗️ **Resilience Design Principles**

### **High Availability**
- Deploy across multiple AZs
- Use load balancers for distribution
- Implement health checks
- Plan for component failures

### **Fault Tolerance**
- Design for graceful degradation
- Implement circuit breakers
- Use retry mechanisms with backoff
- Isolate failure domains

### **Disaster Recovery**
- **RTO**: Recovery Time Objective
- **RPO**: Recovery Point Objective
- **Strategies**: Backup/Restore, Pilot Light, Warm Standby, Multi-Site

---

## 🔧 **Auto Scaling Best Practices**
- Use target tracking for most scenarios
- Set appropriate health check grace periods
- Implement lifecycle hooks for stateful apps
- Monitor scaling activities and metrics
- Use mixed instance types for cost optimization

---

## ⚠️ **Common Mistakes**
- Not enabling Multi-AZ for production databases
- Using IP addresses instead of DNS names
- Insufficient health check grace periods
- Not testing failover procedures
- Forgetting cross-zone load balancing
- Not implementing proper retry logic in applications

---

## 🔑 **Key Points**
- **Multi-AZ** ≠ Read Replicas (HA vs Scaling)
- **ALB** supports path/host routing, **NLB** for performance
- **Auto Scaling** maintains desired capacity across AZs
- **Route 53** health checks enable DNS failover
- **RDS failover** is automatic with Multi-AZ
- **S3 replication** requires versioning enabled