# EKS Pod Identity Implementation Guide for fs-blueprint Service

## What is EKS Pod Identity?

EKS Pod Identity is AWS's **newer and simpler** authentication method for pods to access AWS services, introduced in November 2023 as an alternative to IRSA.

### IRSA vs EKS Pod Identity Comparison

| Feature | IRSA (Older) | EKS Pod Identity (Newer) |
|---------|--------------|--------------------------|
| **Setup Complexity** | Complex (OIDC provider, trust policies with conditions) | Simple (no OIDC needed) |
| **Trust Policy** | Long, with StringEquals conditions | Short, simple principal |
| **Required Add-on** | None (uses OIDC) | EKS Pod Identity Agent |
| **Credential Delivery** | JWT token mounted as file | Credentials via agent |
| **Region Support** | All regions | Limited regions (check availability) |
| **Use Case** | Mature, battle-tested | Newer, cleaner approach |
| **AWS Managed** | Partial | Fully managed |

---

## Architecture: How EKS Pod Identity Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         fs-blueprint Pod                                 │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Application (Java Spring Boot)                                  │     │
│  │ AWS SDK makes API call                                          │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                              ↓                                           │
│         AWS SDK queries for credentials                                  │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────────┐
│              EKS Pod Identity Agent (DaemonSet on Node)                  │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ 1. Intercepts credential request from pod                       │     │
│  │ 2. Checks pod's ServiceAccount                                  │     │
│  │ 3. Queries EKS Pod Identity Association                         │     │
│  │ 4. Calls STS::AssumeRole with pod identity                      │     │
│  │ 5. Returns temporary credentials to pod                         │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    EKS Pod Identity Association                          │
│  Links: namespace/serviceaccount → IAM Role                              │
│  • Namespace: default                                                    │
│  • ServiceAccount: fs-blueprint                                          │
│  • IAM Role: fs-ue1-dev-eks-blueprint-pod-identity-role                  │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                  IAM Role (Simple Trust Policy)                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Trust Policy (No OIDC conditions needed!)                       │     │
│  │ {                                                                │     │
│  │   "Principal": {                                                │     │
│  │     "Service": "pods.eks.amazonaws.com"                         │     │
│  │   }                                                              │     │
│  │ }                                                                │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
                         AWS Services
```

---

## Prerequisites

### Check EKS Version
EKS Pod Identity requires **EKS 1.24+**

```bash
# Check your cluster version
aws eks describe-cluster --name fs-ue1-dev-eks --query 'cluster.version' --output text
```

### Check Region Support
EKS Pod Identity is available in most regions. Verify:
```bash
aws eks list-addons --region us-east-1 | grep pod-identity
```

---

## Implementation Steps

### Step 1: Install EKS Pod Identity Agent Add-on

#### Option A: Using Terraform (Recommended)

Update: `FS-Infra/modules/aws/eks/main.tf`

```hcl
# Find the cluster_addons local variable and add pod-identity-agent

locals {
  cluster_addons = merge(
    { 
      "vpc-cni" = {}, 
      "kube-proxy" = {}, 
      "coredns" = {}, 
      "adot" = local.adot_configuration,
      # ADD THIS:
      "eks-pod-identity-agent" = {
        most_recent = true
      }
    }, 
    var.cluster_addons
  )
}
```

Then apply:
```bash
cd FS-Infra/fs-dev/eks
terraform plan
terraform apply
```

#### Option B: Using AWS CLI

```bash
aws eks create-addon \
  --cluster-name fs-ue1-dev-eks \
  --addon-name eks-pod-identity-agent \
  --region us-east-1
```

#### Verify Installation

```bash
# Check add-on status
aws eks describe-addon \
  --cluster-name fs-ue1-dev-eks \
  --addon-name eks-pod-identity-agent \
  --region us-east-1

# Check DaemonSet is running
kubectl get daemonset eks-pod-identity-agent -n kube-system

# Should show running pods on each node
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

---

### Step 2: Create IAM Role with Simple Trust Policy

Create: `blueprint-service/infra/pod-identity/main.tf`

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_default_region
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  account_id   = data.aws_caller_identity.current.account_id
  region       = data.aws_region.current.name
  service_name = "blueprint"
  namespace    = "default"
  sa_name      = "fs-blueprint"
}

# IAM Role for EKS Pod Identity
resource "aws_iam_role" "blueprint_pod_identity" {
  name        = "fs-ue1-${var.environment}-eks-${local.service_name}-pod-identity-role"
  description = "IAM role for ${local.service_name} service using EKS Pod Identity"

  # SIMPLE TRUST POLICY - No OIDC conditions needed!
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "pods.eks.amazonaws.com"
        }
        Action = [
          "sts:AssumeRole",
          "sts:TagSession"
        ]
      }
    ]
  })

  tags = {
    Name        = "fs-ue1-${var.environment}-eks-${local.service_name}-pod-identity-role"
    Environment = var.environment
    Service     = local.service_name
    ManagedBy   = "Terraform"
  }
}

# Policy 1: Secrets Manager Access
resource "aws_iam_policy" "blueprint_secrets_manager" {
  name        = "fs-ue1-${var.environment}-${local.service_name}-secrets-manager-policy"
  description = "Allow blueprint service to read secrets from Secrets Manager"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          "arn:aws:secretsmanager:${local.region}:${local.account_id}:secret:*blueprint*",
          "arn:aws:secretsmanager:${local.region}:${local.account_id}:secret:*database*",
          "arn:aws:secretsmanager:${local.region}:${local.account_id}:secret:*rds*"
        ]
      }
    ]
  })
}

# Policy 2: Glue Schema Registry Access
resource "aws_iam_policy" "blueprint_glue_schema_registry" {
  name        = "fs-ue1-${var.environment}-${local.service_name}-glue-schema-registry-policy"
  description = "Allow blueprint service to access Glue Schema Registry"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "glue:GetSchemaVersion",
          "glue:GetSchema",
          "glue:GetSchemaByDefinition",
          "glue:ListSchemaVersions",
          "glue:GetRegistry",
          "glue:ListRegistries"
        ]
        Resource = [
          "arn:aws:glue:${local.region}:${local.account_id}:registry/*",
          "arn:aws:glue:${local.region}:${local.account_id}:schema/*/*"
        ]
      }
    ]
  })
}

# Policy 3: CloudWatch Logs
resource "aws_iam_policy" "blueprint_cloudwatch_logs" {
  name        = "fs-ue1-${var.environment}-${local.service_name}-cloudwatch-logs-policy"
  description = "Allow blueprint service to write logs to CloudWatch"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogStreams"
        ]
        Resource = [
          "arn:aws:logs:${local.region}:${local.account_id}:log-group:/aws/eks/${var.cluster_name}/blueprint*"
        ]
      }
    ]
  })
}

# Attach policies to role
resource "aws_iam_role_policy_attachment" "blueprint_secrets_manager" {
  role       = aws_iam_role.blueprint_pod_identity.name
  policy_arn = aws_iam_policy.blueprint_secrets_manager.arn
}

resource "aws_iam_role_policy_attachment" "blueprint_glue_schema_registry" {
  role       = aws_iam_role.blueprint_pod_identity.name
  policy_arn = aws_iam_policy.blueprint_glue_schema_registry.arn
}

resource "aws_iam_role_policy_attachment" "blueprint_cloudwatch_logs" {
  role       = aws_iam_role.blueprint_pod_identity.name
  policy_arn = aws_iam_policy.blueprint_cloudwatch_logs.arn
}

# EKS Pod Identity Association
resource "aws_eks_pod_identity_association" "blueprint" {
  cluster_name    = var.cluster_name
  namespace       = local.namespace
  service_account = local.sa_name
  role_arn        = aws_iam_role.blueprint_pod_identity.arn

  tags = {
    Name        = "fs-${local.service_name}-pod-identity"
    Environment = var.environment
    Service     = local.service_name
  }
}

# Outputs
output "pod_identity_role_arn" {
  description = "ARN of the IAM role for pod identity"
  value       = aws_iam_role.blueprint_pod_identity.arn
}

output "pod_identity_association_id" {
  description = "ID of the pod identity association"
  value       = aws_eks_pod_identity_association.blueprint.id
}

output "service_account_namespace" {
  description = "Kubernetes namespace for the service account"
  value       = local.namespace
}

output "service_account_name" {
  description = "Kubernetes service account name"
  value       = local.sa_name
}
```

### Create variables.tf

```hcl
# blueprint-service/infra/pod-identity/variables.tf

variable "aws_default_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, qa, uat, etc.)"
  type        = string
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "fs-ue1-dev-eks"
}
```

### Create terraform.tfvars

```hcl
# blueprint-service/infra/pod-identity/dev.tfvars

environment        = "dev"
aws_default_region = "us-east-1"
cluster_name       = "fs-ue1-dev-eks"
```

---

### Step 3: Create Kubernetes ServiceAccount

**Important:** With EKS Pod Identity, you **DON'T need** the `eks.amazonaws.com/role-arn` annotation!

Create: `blueprint-service/app/tools/helm-charts/blueprint/templates/serviceaccount.yaml`

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "blueprint.serviceAccountName" . }}
  namespace: {{ .Release.Namespace | default "default" }}
  labels:
    {{- include "blueprint.labels" . | nindent 4 }}
  # NO eks.amazonaws.com/role-arn annotation needed for Pod Identity!
automountServiceAccountToken: {{ .Values.serviceAccount.automount }}
{{- end }}
```

### Update values.yaml

```yaml
# blueprint-service/app/tools/helm-charts/blueprint/values.yaml

serviceAccount:
  create: true
  automount: true
  name: "fs-blueprint"
  # No annotations needed for Pod Identity!
```

---

### Step 4: Update Deployment

Same as IRSA - add `serviceAccountName`:

```yaml
# blueprint-service/app/tools/helm-charts/blueprint/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  template:
    spec:
      serviceAccountName: {{ include "blueprint.serviceAccountName" . }}  # ADD THIS
      containers:
        - name: {{ .Release.Name }}-container
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          # ... rest of config
```

---

## Deployment Steps

### 1. Deploy Terraform

```bash
cd blueprint-service/infra/pod-identity

# Initialize
terraform init

# Plan
terraform plan -var-file=dev.tfvars

# Apply
terraform apply -var-file=dev.tfvars

# Note the outputs:
# pod_identity_role_arn = "arn:aws:iam::126586115171:role/fs-ue1-dev-eks-blueprint-pod-identity-role"
# pod_identity_association_id = "a-xxxxxxxxxxxxx"
```

### 2. Verify Pod Identity Association

```bash
# List pod identity associations
aws eks list-pod-identity-associations \
  --cluster-name fs-ue1-dev-eks \
  --region us-east-1

# Describe specific association
aws eks describe-pod-identity-association \
  --cluster-name fs-ue1-dev-eks \
  --association-id a-xxxxxxxxxxxxx \
  --region us-east-1
```

### 3. Deploy Kubernetes Resources

```bash
cd blueprint-service/app/tools/helm-charts/blueprint

# Deploy with Helm
helm upgrade --install fs-blueprint . \
  -f values.dev.yaml \
  --namespace default \
  --create-namespace
```

### 4. Verify Pod Identity is Working

```bash
# Get pod name
POD_NAME=$(kubectl get pods -n default -l product=fs-blueprint -o jsonpath='{.items[0].metadata.name}')

# Check environment variables
# NOTE: Pod Identity does NOT set AWS_ROLE_ARN or AWS_WEB_IDENTITY_TOKEN_FILE
# It works differently - credentials are provided by the agent
kubectl exec -n default $POD_NAME -- env | grep -E "AWS|KUBERNETES"

# Check that the pod can access AWS services
kubectl exec -n default $POD_NAME -- aws sts get-caller-identity --region us-east-1

# Expected output shows the pod identity role:
# {
#     "UserId": "AROAXXXXXXXXX:eks-default-fs-blueprint-xxxxx",
#     "Account": "126586115171",
#     "Arn": "arn:aws:sts::126586115171:assumed-role/fs-ue1-dev-eks-blueprint-pod-identity-role/eks-default-fs-blueprint-xxxxx"
# }

# Test Secrets Manager access
kubectl exec -n default $POD_NAME -- \
  aws secretsmanager list-secrets --region us-east-1 --max-items 1

# Test Glue Schema Registry access
kubectl exec -n default $POD_NAME -- \
  aws glue list-registries --region us-east-1

# Check application logs
kubectl logs -n default $POD_NAME --tail=100 | grep -i "credential\|aws"
```

---

## Key Differences from IRSA

### IRSA Setup:
```yaml
# ServiceAccount with annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::xxx:role/my-role  # REQUIRED
```

```hcl
# Complex trust policy with OIDC conditions
assume_role_policy = jsonencode({
  Statement = [{
    Principal = {
      Federated = "arn:aws:iam::xxx:oidc-provider/oidc.eks..."
    }
    Condition = {
      StringEquals = {
        "oidc.eks....:sub" = "system:serviceaccount:ns:sa"
        "oidc.eks....:aud" = "sts.amazonaws.com"
      }
    }
  }]
})
```

### Pod Identity Setup:
```yaml
# ServiceAccount without annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fs-blueprint  # NO annotations needed!
```

```hcl
# Simple trust policy
assume_role_policy = jsonencode({
  Statement = [{
    Principal = {
      Service = "pods.eks.amazonaws.com"  # That's it!
    }
  }]
})

# Separate association resource links SA to role
resource "aws_eks_pod_identity_association" "blueprint" {
  cluster_name    = "fs-ue1-dev-eks"
  namespace       = "default"
  service_account = "fs-blueprint"
  role_arn        = aws_iam_role.blueprint_pod_identity.arn
}
```

---

## Troubleshooting

### Issue 1: Pod Identity Agent not running

```bash
# Check DaemonSet
kubectl get daemonset eks-pod-identity-agent -n kube-system

# Check pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent

# Check logs
kubectl logs -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent --tail=50
```

### Issue 2: Association not found

```bash
# List all associations
aws eks list-pod-identity-associations \
  --cluster-name fs-ue1-dev-eks \
  --region us-east-1

# Verify association exists for your namespace/SA
aws eks describe-pod-identity-association \
  --cluster-name fs-ue1-dev-eks \
  --association-id a-xxxxx \
  --region us-east-1
```

### Issue 3: Access denied

```bash
# Check IAM role trust policy
aws iam get-role \
  --role-name fs-ue1-dev-eks-blueprint-pod-identity-role \
  --query 'Role.AssumeRolePolicyDocument'

# Must have:
# Principal.Service = "pods.eks.amazonaws.com"

# Check attached policies
aws iam list-attached-role-policies \
  --role-name fs-ue1-dev-eks-blueprint-pod-identity-role

# Check policy permissions
aws iam get-policy-version \
  --policy-arn <arn> \
  --version-id $(aws iam get-policy --policy-arn <arn> --query 'Policy.DefaultVersionId' --output text)
```

### Issue 4: Pod can't assume role

```bash
# Check pod events
kubectl describe pod $POD_NAME -n default

# Check if ServiceAccount is correctly set
kubectl get pod $POD_NAME -n default -o jsonpath='{.spec.serviceAccountName}'

# Should return: fs-blueprint

# Restart pod to pick up association
kubectl rollout restart deployment fs-blueprint -n default
```

---

## Migration from IRSA to Pod Identity

If you've already implemented IRSA and want to migrate:

### Step 1: Keep IRSA running while testing
- Deploy Pod Identity alongside IRSA
- Test with a single pod first

### Step 2: Switch gradually
```bash
# Update deployment to use new ServiceAccount (without annotation)
# Pod Identity will take over automatically

# Monitor CloudTrail to see which role is being used
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=fs-ue1-dev-eks-blueprint-pod-identity-role \
  --region us-east-1
```

### Step 3: Remove IRSA resources
Once confirmed working:
- Remove OIDC-based IAM role
- Remove `eks.amazonaws.com/role-arn` annotation
- Clean up old Terraform resources

---

## Advantages of Pod Identity over IRSA

✅ **Simpler Setup**
- No OIDC provider configuration
- No complex trust policies with conditions
- Cleaner separation of concerns

✅ **Better Management**
- Association is separate from IAM role
- Can see all pod-role mappings in one place
- Easier to audit

✅ **More Flexible**
- Can associate multiple namespaces/SAs to same role
- Can update associations without changing IAM role
- Better for multi-tenant clusters

✅ **AWS Managed**
- EKS Pod Identity Agent is fully managed by AWS
- Automatic updates and security patches

---

## When to Use What?

### Use **IRSA** if:
- ✅ You need to work in regions where Pod Identity isn't available
- ✅ You're already heavily invested in IRSA
- ✅ You have strict compliance requiring OIDC-based auth

### Use **Pod Identity** if:
- ✅ Starting fresh (no existing IRSA setup)
- ✅ Want simpler management
- ✅ Your region supports it
- ✅ You prefer AWS-managed solutions

---

## Complete File Structure

```
blueprint-service/
├── infra/
│   ├── pod-identity/              # NEW
│   │   ├── main.tf                # IAM role + association
│   │   ├── variables.tf           # Variables
│   │   └── dev.tfvars             # Dev config
│   └── irsa/                      # OLD (can coexist)
│       └── ...
└── app/tools/helm-charts/blueprint/
    ├── templates/
    │   ├── serviceaccount.yaml    # NO annotations for Pod Identity
    │   └── deployment.yaml         # serviceAccountName: fs-blueprint
    └── values.dev.yaml            # No eks.amazonaws.com/role-arn needed
```

---

## Summary

**EKS Pod Identity** provides a cleaner, simpler way to give pods access to AWS services:

1. **Install**: EKS Pod Identity Agent add-on
2. **Create**: IAM role with simple trust policy (`pods.eks.amazonaws.com`)
3. **Associate**: Link namespace/SA to role via `aws_eks_pod_identity_association`
4. **Deploy**: Pod with ServiceAccount (no annotations needed!)

That's it! No OIDC provider, no complex conditions, no annotations. 🎉
