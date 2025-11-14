# Karpenter Setup Guide for EKS

## Prerequisites
- AWS CLI configured
- kubectl installed
- eksctl installed
- helm installed

## Step 1: Set Environment Variables

```bash
export KARPENTER_NAMESPACE="kube-system"
export KARPENTER_VERSION="1.8.1"
export K8S_VERSION="1.34"
export AWS_PARTITION="aws"
export CLUSTER_NAME="eks-v134-karpenter-ami-AL2023"
export AWS_DEFAULT_REGION="us-west-2"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT="$(mktemp)"
```

## Step 2: Create EKS Cluster

```bash
eksctl create cluster \
  --name=${CLUSTER_NAME} \
  --region=${AWS_DEFAULT_REGION} \
  --version=1.34 \
  --node-ami-family=AmazonLinux2023
```

## Step 3: Create IAM Roles via CloudFormation

```bash
curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml > "${TEMPOUT}" \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

This creates:
- KarpenterNodeRole - IAM role for EC2 instances launched by Karpenter
- KarpenterControllerPolicy - IAM policy for Karpenter controller

## Step 4: Enable Pod Identity Agent Addon

**What is Pod Identity?**
Pod Identity allows Kubernetes pods to assume IAM roles without needing to manage AWS credentials or use IRSA (IAM Roles for Service Accounts).

**Why we need it:**
- Karpenter pods need AWS permissions (to launch EC2 instances, describe subnets, etc.)
- Pod Identity automatically injects temporary AWS credentials into pods
- More secure than storing credentials in pods
- Simpler than IRSA (no OIDC provider needed)

**What this does:**
Installs the EKS Pod Identity Agent as a DaemonSet on all nodes. This agent:
1. Intercepts AWS API calls from pods
2. Exchanges the pod's service account token for temporary AWS credentials
3. Returns credentials to the pod

```bash
eksctl create addon \
  --cluster=${CLUSTER_NAME} \
  --name=eks-pod-identity-agent
```

## Step 5: Create Pod Identity Association

**What is Pod Identity Association?**
A mapping that tells EKS: "When a pod uses this ServiceAccount, give it permissions from this IAM role."

**Why we need it:**
- Links the Karpenter ServiceAccount to the KarpenterControllerRole
- Allows Karpenter pods to call AWS APIs (EC2, Pricing, SSM, etc.)
- Without this, Karpenter can't launch instances

**What this creates:**
- Association between:
  - Kubernetes: `kube-system/karpenter` ServiceAccount
  - AWS: `KarpenterControllerRole` IAM role
- When Karpenter pod starts, it automatically gets AWS credentials for this role

**Example flow:**
1. Karpenter pod starts with `serviceAccountName: karpenter`
2. Pod Identity Agent sees the association
3. Agent provides temporary AWS credentials to the pod
4. Karpenter can now call `ec2:RunInstances`, `ec2:DescribeSubnets`, etc.

```bash
eksctl create podidentityassociation \
  --cluster ${CLUSTER_NAME} \
  --namespace ${KARPENTER_NAMESPACE} \
  --service-account-name karpenter \
  --role-name ${CLUSTER_NAME}-karpenter \
  --permission-policy-arns arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
```

## Step 6: Update aws-auth ConfigMap

**What is aws-auth ConfigMap?**
A Kubernetes ConfigMap that maps AWS IAM roles/users to Kubernetes RBAC groups. It's how EKS controls which AWS identities can join the cluster.

**Why we need it:**
- EC2 instances launched by Karpenter need to join the cluster as worker nodes
- They authenticate using the KarpenterNodeRole IAM role
- Without this mapping, instances can't register as nodes

**What this does:**
Adds an entry to aws-auth ConfigMap:
```yaml
- rolearn: arn:aws:iam::123456789:role/KarpenterNodeRole-cluster-name
  username: system:node:{{EC2PrivateDNSName}}  # Node's hostname
  groups:
  - system:bootstrappers  # Allows node bootstrap process
  - system:nodes          # Gives node-level permissions
```

**Example flow:**
1. Karpenter launches EC2 instance with KarpenterNodeRole
2. Instance runs kubelet, which tries to register with EKS
3. EKS checks aws-auth ConfigMap
4. Finds KarpenterNodeRole → grants `system:nodes` permissions
5. Node successfully joins cluster

```bash
eksctl create iamidentitymapping \
  --cluster ${CLUSTER_NAME} \
  --arn arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
  --username system:node:{{EC2PrivateDNSName}} \
  --group system:bootstrappers \
  --group system:nodes
```

## Step 7: Create EC2 Spot Service-Linked Role

**What is a Service-Linked Role?**
A special IAM role that's linked to an AWS service. AWS manages the permissions, you just create it once.

**Why we need it:**
- Required for launching EC2 Spot instances
- Even if you're only using on-demand now, it's needed for future spot usage
- AWS Spot service needs permissions to manage spot requests on your behalf

**What this does:**
Creates `AWSServiceRoleForEC2Spot` role with permissions to:
- Request spot instances
- Terminate spot instances when interrupted
- Send interruption notifications

**Note:** 
- This is account-wide (not cluster-specific)
- If it already exists, the command fails gracefully (`|| true`)
- You only need to create it once per AWS account

**Example use case:**
If you later change NodePool to use spot instances:
```yaml
requirements:
  - key: karpenter.sh/capacity-type
    values: ["spot"]  # This requires the service-linked role
```

```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

## Step 8: Tag Subnets for Karpenter Discovery

**What is subnet tagging?**
Adding metadata tags to subnets so Karpenter can automatically find them.

**Why we need it:**
- Karpenter needs to know which subnets to launch instances in
- Instead of hardcoding subnet IDs, we use tags for discovery
- Makes configuration portable and easier to manage

**What this does:**
1. Finds your cluster's VPC
2. Finds private subnets (tagged with `kubernetes.io/role/internal-elb`)
3. Tags them with `karpenter.sh/discovery=cluster-name`

**How Karpenter uses it:**
In EC2NodeClass, you specify:
```yaml
subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: "cluster-name"
```
Karpenter queries AWS: "Give me all subnets with this tag" and uses them for instance placement.

**Why private subnets?**
- Worker nodes should be in private subnets for security
- They access internet via NAT Gateway
- Public subnets are for load balancers

```bash
# Get VPC ID
VPC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query 'cluster.resourcesVpcConfig.vpcId' --output text)

# Tag private subnets
for SUBNET in $(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${VPC_ID}" "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
  --query 'Subnets[*].SubnetId' --output text); do
  aws ec2 create-tags --resources $SUBNET --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
done
```

## Step 9: Tag Security Groups

**What is security group tagging?**
Adding metadata tags to security groups so Karpenter can automatically find and attach them to instances.

**Why we need it:**
- Instances need security groups to control network traffic
- EKS creates a cluster security group that allows pod-to-pod communication
- Karpenter needs to attach this to new instances

**What this does:**
Tags the EKS cluster security group with `karpenter.sh/discovery=cluster-name`

**How Karpenter uses it:**
In EC2NodeClass:
```yaml
securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: "cluster-name"
```
Karpenter finds and attaches this security group to all instances it launches.

**What the cluster security group allows:**
- Pod-to-pod communication across nodes
- Node-to-control-plane communication
- Control-plane-to-node communication (for kubelet)

**Without this:**
- Instances would launch without proper security groups
- Pods couldn't communicate across nodes
- Nodes couldn't join the cluster

```bash
SECURITY_GROUP=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query cluster.resourcesVpcConfig.clusterSecurityGroupId --output text)
aws ec2 create-tags --resources $SECURITY_GROUP --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
```

## Step 10: Install Karpenter with Helm

```bash
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"

helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

## Step 11: Verify Karpenter Installation

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter --tail=50
```

## Step 12: Create NodePool and EC2NodeClass

Create `karpenter-nodepool.yaml`:

```yaml
# NodePool defines the template for nodes that Karpenter will provision
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      # Requirements: Constraints for instance selection
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]  # Only x86_64 architecture
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]  # Only Linux OS
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]  # Use on-demand instances (not spot)
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]  # Compute, Memory, or General purpose instances
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]  # Only instance generations > 2 (e.g., m5, m6, not m4)
      
      # Taints: Prevent pods without matching tolerations from scheduling here
      # This ensures only workloads that explicitly tolerate "workload=karpenter" can run
      taints:
        - key: workload
          value: "karpenter"
          effect: NoSchedule  # Pods must have toleration to schedule
      
      # Reference to EC2NodeClass for AWS-specific configuration
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  
  # Limits: Maximum resources Karpenter can provision
  limits:
    cpu: 1000  # Max 1000 vCPUs across all nodes in this pool
  
  # Disruption: How Karpenter handles node consolidation and removal
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized  # Consolidate underutilized nodes
    consolidateAfter: 1m  # Wait 1 minute before consolidating
---
# EC2NodeClass defines AWS-specific configuration for nodes
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # AMI family to use for nodes
  amiFamily: AL2023  # Amazon Linux 2023
  
  # IAM role for EC2 instances (created by CloudFormation in Step 3)
  role: "KarpenterNodeRole-eks-v134-karpenter-ami-AL2023"
  
  # Subnet selection: Where to launch instances
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "eks-v134-karpenter-ami-AL2023"  # Use tagged subnets
  
  # Security group selection: Which security groups to attach
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "eks-v134-karpenter-ami-AL2023"  # Use tagged SGs
  
  # AMI selection: Which AMI to use
  amiSelectorTerms:
    - alias: al2023@latest  # Use latest AL2023 EKS-optimized AMI
```

Apply the configuration:

```bash
kubectl apply -f karpenter-nodepool.yaml
```

## Step 13: Verify NodePool and EC2NodeClass

```bash
kubectl get nodepool
kubectl get ec2nodeclass
kubectl describe ec2nodeclass default
```

## Step 14: Test Karpenter with Sample Workload

Create `inflate-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 1  # Start with 1 replica to trigger node creation
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      # Tolerations: Allow this pod to schedule on tainted nodes
      # Must match the taint in NodePool (workload=karpenter:NoSchedule)
      tolerations:
        - key: workload
          operator: Equal
          value: "karpenter"
          effect: NoSchedule
      
      containers:
      - name: inflate
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.2  # Minimal pause container
        resources:
          requests:
            cpu: 1  # Request 1 CPU to trigger node provisioning
```

**What happens when you deploy this:**

1. Pod is created but can't schedule on existing nodes (they don't have the taint toleration)
2. Karpenter sees the pending pod
3. Karpenter provisions a new node matching the NodePool requirements
4. New node has the `workload=karpenter:NoSchedule` taint
5. Pod schedules on the new node (because it has the matching toleration)

Deploy and watch:

```bash
# Apply the deployment
kubectl apply -f inflate-deployment.yaml

# Watch nodes being created (you'll see a new node appear)
kubectl get nodes -w

# In another terminal, watch pod status
kubectl get pods -o wide -w

# Check Karpenter logs to see provisioning decisions
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f
```

**Expected output:**
- Initially: Pod in `Pending` state
- After ~30-60 seconds: New node appears
- Pod transitions to `Running` on the new node
- Node will have label `karpenter.sh/nodepool=default`

## Troubleshooting

### Check Karpenter Logs
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter --tail=100
```

### Verify Pod Identity Association
```bash
aws eks list-pod-identity-associations --cluster-name ${CLUSTER_NAME}
kubectl describe sa karpenter -n kube-system
```

### Check EC2NodeClass Status
```bash
kubectl describe ec2nodeclass default
```

### Verify Subnet Tags
```bash
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}" \
  --query 'Subnets[*].[SubnetId,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

## IAM Roles Summary

### KarpenterNodeRole
- **Purpose**: Assumed by EC2 instances launched by Karpenter
- **Policies**:
  - AmazonEKSWorkerNodePolicy
  - AmazonEKS_CNI_Policy
  - AmazonEC2ContainerRegistryReadOnly
  - AmazonSSMManagedInstanceCore

### KarpenterControllerRole
- **Purpose**: Assumed by Karpenter controller pods via Pod Identity
- **Permissions**:
  - ec2:CreateFleet, RunInstances, TerminateInstances
  - ec2:DescribeInstances, DescribeInstanceTypes, DescribeSubnets
  - ec2:CreateTags, DeleteTags
  - iam:PassRole (for KarpenterNodeRole)
  - eks:DescribeCluster
  - pricing:GetProducts
  - ssm:GetParameter

## Understanding Taints and Tolerations

**Taints** (on nodes): Repel pods that don't have matching tolerations
- Applied to NodePool in Step 12
- Format: `key=value:effect`
- Effect `NoSchedule`: Pods without toleration cannot schedule

**Tolerations** (on pods): Allow pods to schedule on tainted nodes
- Applied to Deployment in Step 14
- Must match the taint key, value, and effect

**Why use them together?**
- Ensures only specific workloads run on Karpenter-managed nodes
- Prevents system pods or other workloads from using these nodes
- Gives you control over which pods trigger node provisioning

## Cleanup

```bash
# Delete test workload
kubectl delete deployment inflate

# Delete Karpenter resources
kubectl delete nodepool default
kubectl delete ec2nodeclass default

# Uninstall Karpenter
helm uninstall karpenter -n kube-system

# Delete cluster
eksctl delete cluster --name ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}

# Delete CloudFormation stack
aws cloudformation delete-stack --stack-name Karpenter-${CLUSTER_NAME}
```
