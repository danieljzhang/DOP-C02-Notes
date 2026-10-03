# AWS EKS (Elastic Kubernetes Service) - DOP-C02 Study Notes

## 1. Overview

### What is EKS?
- Managed Kubernetes service for running containerized applications
- Provides highly available and secure Kubernetes control plane
- Integrates with AWS services for networking, security, and monitoring
- Supports both EC2 and Fargate compute options

### Key Benefits
- **Managed Control Plane** - AWS manages Kubernetes masters
- **High Availability** - Multi-AZ control plane deployment
- **Security** - Integration with IAM and VPC
- **Scalability** - Auto scaling for nodes and pods
- **AWS Integration** - Native integration with AWS services

---

## 2. Cluster Architecture

### Control Plane
- Managed by AWS across multiple AZs
- Includes API server, etcd, scheduler, and controller manager
- Automatic updates and patching
- 99.95% SLA for control plane availability

### Worker Nodes
```bash
# Create managed node group
aws eks create-nodegroup \
  --cluster-name "my-cluster" \
  --nodegroup-name "worker-nodes" \
  --subnets "subnet-12345678" "subnet-87654321" \
  --instance-types "t3.medium" \
  --ami-type "AL2_x86_64" \
  --node-role "arn:aws:iam::123456789012:role/NodeInstanceRole" \
  --scaling-config minSize=1,maxSize=10,desiredSize=3
```

### Fargate Integration
```bash
# Fargate profiles are created via the AWS CLI/API - NOT via kubectl manifests.
# (There is no "Fargate profile" Kubernetes object; a ConfigMap cannot define one.)
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name default \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/eks-fargate-pod-execution-role \
  --selectors namespace=default,labels={compute-type=fargate} \
  --subnets subnet-12345678 subnet-87654321
```

---

## 3. IAM Roles for Service Accounts (IRSA)

### OIDC Identity Provider Setup
```bash
# Create OIDC identity provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```

### Service Account with IAM Role
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3AccessRole
```

### IAM Role Trust Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:default:s3-access-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

---

## 4. Node Groups and Scaling

### Managed Node Groups
```python
def create_managed_node_group():
    """Create EKS managed node group with auto scaling"""
    
    eks = boto3.client('eks')
    
    response = eks.create_nodegroup(
        clusterName='my-cluster',
        nodegroupName='production-nodes',
        scalingConfig={
            'minSize': 2,
            'maxSize': 20,
            'desiredSize': 5
        },
        instanceTypes=['t3.medium', 't3.large'],
        amiType='AL2_x86_64',
        nodeRole='arn:aws:iam::123456789012:role/NodeInstanceRole',
        subnets=[
            'subnet-12345678',
            'subnet-87654321',
            'subnet-13579246'
        ],
        tags={
            'Environment': 'Production',
            'Team': 'DevOps'
        },
        launchTemplate={
            'name': 'eks-node-template',
            'version': '$Latest'
        }
    )
    
    return response
```

### Cluster Autoscaler
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.21.0  # k8s.gcr.io is deprecated; use registry.k8s.io
        name: cluster-autoscaler
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
```

---

## 5. Networking and Security

### VPC Configuration
```python
def setup_eks_networking():
    """Set up EKS networking with proper security"""
    
    ec2 = boto3.client('ec2')
    
    # Create security group for EKS cluster
    cluster_sg = ec2.create_security_group(
        GroupName='eks-cluster-sg',
        Description='Security group for EKS cluster',
        VpcId='vpc-12345678'
    )
    
    # Allow HTTPS traffic from worker nodes
    ec2.authorize_security_group_ingress(
        GroupId=cluster_sg['GroupId'],
        IpPermissions=[
            {
                'IpProtocol': 'tcp',
                'FromPort': 443,
                'ToPort': 443,
                'UserIdGroupPairs': [
                    {
                        'GroupId': 'sg-worker-nodes',
                        'Description': 'HTTPS from worker nodes'
                    }
                ]
            }
        ]
    )
    
    return cluster_sg['GroupId']
```

### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

## 6. Monitoring and Logging

### Container Insights
```bash
# Deploy CloudWatch Container Insights
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/my-cluster/;s/{{region_name}}/us-east-1/" | kubectl apply -f -
```

### Prometheus and Grafana
```yaml
# Prometheus configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

---

## 7. Application Deployment Patterns

### Blue/Green Deployment
```yaml
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        ports:
        - containerPort: 8080
---
# Service pointing to blue
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80
    targetPort: 8080
```

### Canary Deployment with Istio
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: app-canary
spec:
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: app-service
        subset: v2
  - route:
    - destination:
        host: app-service
        subset: v1
      weight: 90
    - destination:
        host: app-service
        subset: v2
      weight: 10
```

---

## 8. Disaster Recovery and Backup

### Velero Backup
```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.2.0 \
  --bucket my-backup-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Create backup
velero backup create my-backup --include-namespaces production
```

### Cross-Region Cluster Setup
```python
def setup_cross_region_eks():
    """Set up EKS clusters across regions for DR"""
    
    regions = ['us-east-1', 'us-west-2']
    
    for region in regions:
        eks = boto3.client('eks', region_name=region)
        
        # Create cluster in each region
        cluster_config = {
            'name': f'production-cluster-{region}',
            'version': '1.21',
            'roleArn': f'arn:aws:iam::123456789012:role/EKSServiceRole',
            'resourcesVpcConfig': {
                'subnetIds': get_subnets_for_region(region),
                'securityGroupIds': [get_security_group_for_region(region)]
            },
            'logging': {
                'enable': [
                    {
                        'types': ['api', 'audit', 'authenticator', 'controllerManager', 'scheduler']
                    }
                ]
            }
        }
        
        eks.create_cluster(**cluster_config)
```

---

## 9. Common Exam Scenarios

### Scenario 1: Secure microservices communication
**Solution:**
- Implement service mesh (Istio/App Mesh)
- Use network policies for pod-to-pod communication
- Configure IRSA for AWS service access
- Enable mutual TLS between services

### Scenario 2: Auto scaling containerized applications
**Solution:**
- Configure Horizontal Pod Autoscaler (HPA)
- Set up Cluster Autoscaler for node scaling
- Use Vertical Pod Autoscaler (VPA) for right-sizing
- Implement custom metrics scaling

### Scenario 3: Multi-environment deployment pipeline
**Solution:**
- Use GitOps with ArgoCD or Flux
- Implement namespace-based environment separation
- Configure RBAC for environment access
- Use Helm charts for application packaging

### Scenario 4: Disaster recovery for stateful applications
**Solution:**
- Use persistent volumes with cross-AZ storage
- Implement regular backups with Velero
- Set up cross-region cluster replication
- Configure database replication for stateful services

---

## 10. CLI Commands Reference

```bash
# Create EKS cluster
aws eks create-cluster \
  --name my-cluster \
  --version 1.21 \
  --role-arn arn:aws:iam::123456789012:role/EKSServiceRole \
  --resources-vpc-config subnetIds=subnet-12345678,subnet-87654321

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name my-cluster

# Create node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name worker-nodes \
  --subnets subnet-12345678 subnet-87654321 \
  --instance-types t3.medium \
  --node-role arn:aws:iam::123456789012:role/NodeInstanceRole

# Delete cluster
aws eks delete-cluster --name my-cluster
```

---

## 11. Exam Tips

### Key Points to Remember
- EKS control plane is managed by AWS across multiple AZs
- IRSA provides secure access to AWS services without storing credentials
- Managed node groups provide automatic updates and scaling
- Fargate provides serverless container execution
- Network policies control pod-to-pod communication

### Common Mistakes
- Not configuring proper IAM roles for service accounts
- Forgetting to set up OIDC identity provider for IRSA
- Not implementing proper network segmentation
- Overlooking cluster autoscaler configuration for node scaling

### Best Practices for Exam
- Understand IRSA implementation and use cases
- Know node group types and scaling mechanisms
- Understand networking and security best practices
- Know monitoring and logging integration patterns
- Understand disaster recovery and backup strategies