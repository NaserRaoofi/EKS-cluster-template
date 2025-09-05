# EKS Production Cluster Configuration

This repository contains the configuration and setup for a production-ready Amazon EKS cluster with optimized node groups and essential addons.

## Cluster Overview

- **Cluster Name**: `test-cluster`
- **Region**: `us-east-1`
- **Kubernetes Version**: `1.29`
- **VPC CIDR**: `10.0.0.0/16`
- **NAT Gateway**: Single (cost-optimized)

## Architecture

### Node Groups

The cluster consists of three specialized node groups:

#### 1. System Nodes

- **Purpose**: Cluster management and system workloads
- **Instance Type**: `t3.small`
- **Capacity**: 1 node (min: 1, max: 1)
- **Taints**: `node-role.kubernetes.io/system=true:NoSchedule`
- **Labels**: `role=system`, `workload-type=system`

#### 2. Database Nodes

- **Purpose**: Database workloads with dedicated resources
- **Instance Type**: `t3.small`
- **Capacity**: 1 node (min: 1, max: 2)
- **Taints**: `workload-type=database:NoSchedule`
- **Labels**: `role=database`, `workload-type=database`

#### 3. Public Nodes

- **Purpose**: General application workloads and public-facing services
- **Instance Type**: `t3.small`
- **Capacity**: 1 node (min: 1, max: 2)
- **Taints**: None (accepts any workload)
- **Labels**: `role=public`, `workload-type=public`

## Addons Configuration

### Core Addons

- **eks-pod-identity-agent**: v1.3.8 - Enables Pod Identity Associations
- **vpc-cni**: v1.19.0 - Pod networking with Pod Identity Associations
- **coredns**: v1.11.1 - DNS service
- **kube-proxy**: v1.29.10 - Network proxy
- **metrics-server**: v0.8.0 - Resource metrics

### Storage Addon

- **aws-ebs-csi-driver**: v1.48.0 - EBS volume management with OIDC

## IAM Configuration

### OIDC Provider

- **Enabled**: `withOIDC: true`
- **Used for**: EBS CSI Controller service account

### Service Accounts

- **EBS CSI Controller**: Uses `wellKnownPolicies.ebsCSIController`
- **VPC CNI**: Uses Pod Identity Associations with `AmazonEKS_CNI_Policy`

### Node Group Policies

All node groups include:

- Auto Scaler permissions
- External DNS permissions
- Cert Manager permissions
- EBS permissions

## Files

- `prod-cluster-config.yaml`: Main EKS cluster configuration
- `ebs-csi-policy.json`: Custom IAM policy for EBS CSI driver
- `aws-auth-configmap.yaml`: Kubernetes RBAC configuration (if present)

## Deployment

### Prerequisites

- AWS CLI configured with appropriate permissions
- eksctl installed
- kubectl installed

### Create Cluster

```bash
# Create the cluster
eksctl create cluster -f prod-cluster-config.yaml

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name test-cluster
```

### Update Cluster

```bash
# Update addons
eksctl update addon -f prod-cluster-config.yaml

# Update specific addon
eksctl update addon --name vpc-cni --cluster test-cluster --region us-east-1
```

### Verify Deployment

```bash
# Check cluster status
kubectl get nodes

# Check addons
eksctl get addons --cluster test-cluster --region us-east-1

# Check pods
kubectl get pods -A
```

## Usage

### Deploying to Specific Node Groups

#### System Workloads

```yaml
spec:
  tolerations:
    - key: "node-role.kubernetes.io/system"
      value: "true"
      effect: "NoSchedule"
  nodeSelector:
    role: system
```

#### Database Workloads

```yaml
spec:
  tolerations:
    - key: "workload-type"
      value: "database"
      effect: "NoSchedule"
  nodeSelector:
    role: database
```

#### Public Workloads

```yaml
spec:
  nodeSelector:
    role: public
```

### Using EBS Storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
allowVolumeExpansion: true
```

## Cost Optimization Features

- Single NAT Gateway (reduces costs)
- Small instance types (`t3.small`)
- Minimal node counts with auto-scaling
- Optimized addon configuration

## Security Features

- Node groups with SSH disabled
- Taints and tolerations for workload isolation
- OIDC provider for secure service account access
- Pod Identity Associations for modern IAM integration

## Monitoring and Observability

- Metrics Server for resource monitoring
- CloudWatch logging available (can be enabled)
- Default EKS control plane logging

## Cleanup

```bash
# Delete the cluster and all resources
eksctl delete cluster --region=us-east-1 --name=test-cluster
```

## Support

For issues and questions:

1. Check CloudFormation console for stack events
2. Use `eksctl utils describe-stacks --region=us-east-1 --cluster=test-cluster`
3. Check EKS cluster logs in CloudWatch

---

**Last Updated**: September 5, 2025
**EKS Version**: 1.29
**eksctl Version**: 0.214.0
