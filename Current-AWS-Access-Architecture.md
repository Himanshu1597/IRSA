# How fs-blueprint Pod Accesses AWS Services in EKS Cluster

## Overview
The `fs-blueprint` pod running in the `fs-ue1-dev-eks` cluster accesses AWS services through **EC2 Node IAM Roles** attached to the EKS worker nodes. This is the traditional approach where all pods on a node inherit the IAM permissions of that node's IAM role.

## Architecture Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         fs-blueprint Pod                                 │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Application (Java Spring Boot)                                  │     │
│  │ - AWS Secrets Manager JDBC Driver                              │     │
│  │ - AWS Glue Schema Registry                                     │     │
│  │ - AWS SDK (implicit authentication)                            │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                              ↓                                           │
│         Uses EC2 Instance Metadata Service (IMDS v2)                     │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    EKS Worker Node (EC2 Instance)                        │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Instance Metadata Service (IMDS)                                │     │
│  │ http://169.254.169.254/latest/meta-data/iam/security-credentials│    │
│  └────────────────────────────────────────────────────────────────┘     │
│                              ↓                                           │
│              Returns temporary AWS credentials from                      │
│           EC2 Node IAM Role (attached to node group)                     │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                  EKS Node IAM Role with Policies                         │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Policies Attached:                                              │     │
│  │ • CloudWatchAgentServerPolicy                                   │     │
│  │ • AmazonDynamoDBFullAccess                                      │     │
│  │ • AmazonSNSFullAccess                                           │     │
│  │ • AmazonSSMManagedInstanceCore                                  │     │
│  │ • fs-ue1-lower-policy-cloudfront-access-iam                     │     │
│  │ • fs-ue1-lower-policy-custom-msk-iam                            │     │
│  │ • fs-ue1-lower-policy-kafka-access-iam                          │     │
│  │ • secret-manager-read                                           │     │
│  │ • custom-read-write-s3-docms                                    │     │
│  │ • crowdstrike_sensor_access                                     │     │
│  │ • ses_send_mail_policy (ses:SendEmail, ses:SendRawEmail)        │     │
│  │ • alt_glue_schema_access (Glue Schema Registry)                 │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                        AWS Services                                      │
│  • AWS Secrets Manager (Database credentials)                            │
│  • AWS Glue Schema Registry (Kafka schemas)                              │
│  • Amazon DynamoDB                                                       │
│  • Amazon SNS                                                            │
│  • Amazon MSK (Kafka)                                                    │
│  • Amazon S3                                                             │
│  • Amazon SES                                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. Pod Configuration
**Location:** `blueprint-service/app/tools/helm-charts/blueprint/templates/deployment.yaml`

The pod deployment does **NOT** specify a `serviceAccountName`, meaning it uses the `default` service account which has no AWS IAM role annotation. Instead, the pod relies on the node's IAM role.

```yaml
# From deployment.yaml - NO serviceAccount specified
spec:
  containers:
    - name: fs-blueprint-container
      image: 126586115171.dkr.ecr.us-east-1.amazonaws.com/fs-ue1-dev-ecr-blueprint:latest
```

### 2. EKS Node IAM Role Configuration
**Location:** `FS-Infra/fs-dev/eks/main.tf`

The EKS module is configured with additional IAM policies that are attached to the managed node group:

```hcl
iam_role_additional_policies = [
  "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy",
  "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess",
  "arn:aws:iam::aws:policy/AmazonSNSFullAccess",
  "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
  "arn:aws:iam::126586115171:policy/fs-ue1-lower-policy-cloudfront-access-iam",
  "arn:aws:iam::126586115171:policy/fs-ue1-lower-policy-custom-msk-iam",
  "arn:aws:iam::126586115171:policy/fs-ue1-lower-policy-kafka-access-iam",
  "arn:aws:iam::126586115171:policy/secret-manager-read",
  "arn:aws:iam::126586115171:policy/custom-read-write-s3-docms"
]
```

**Location:** `FS-Infra/modules/aws/eks/main.tf` (lines 91, 439)

These policies are merged with additional policies for:
- Crowdstrike sensor access
- SES email sending
- Glue Schema Registry access

### 3. Application AWS Integration
**Location:** `blueprint-service/app/src/main/resources/application.yml`

The application uses AWS services through:

#### a) AWS Secrets Manager JDBC Driver
```yaml
spring:
  datasource:
    url: ${DB_AWS_SECRET_NAME}
    username: ${DB_AWS_SECRET_NAME}
    driver-class-name: com.amazonaws.secretsmanager.sql.AWSSecretsManagerPostgreSQLDriver
```

#### b) AWS Glue Schema Registry for Kafka
```yaml
kafka:
  avro:
    config:
      region: ${AWS_REGION:us-east-1}
      schema.registry.url: ${FS_AWS_GLUE_SCHEMA_REGISTRY_URL}
      registry.name: ${CDE_GLUE_SCHEMA_REGISTRY_NAME}
```

## How Authentication Works

### Step-by-Step Process:

1. **Pod Starts**: The `fs-blueprint` pod starts on an EKS worker node
2. **AWS SDK Initialization**: When the application initializes AWS SDK clients (Secrets Manager, Glue, etc.), the SDK looks for credentials in this order:
   - Environment variables (not set)
   - System properties (not set)
   - Web Identity Token (IRSA - not configured for this pod)
   - **EC2 Instance Metadata Service (IMDS)** ← Used here
3. **IMDS Query**: The AWS SDK queries the EC2 IMDS endpoint at `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
4. **Credential Retrieval**: IMDS returns temporary security credentials from the IAM role attached to the EC2 instance
5. **AWS API Calls**: The pod uses these credentials to make AWS API calls

### IMDS Configuration:
The EKS module configures IMDS v2 with hop limit:
```hcl
metadata_options = {
  http_put_response_hop_limit = 2
}
```
This allows the pod to access IMDS through the container runtime.

## Important Security Notes

### Current Setup (Node IAM Role)
✅ **Pros:**
- Simple to configure
- No service account annotations needed
- Works out-of-the-box

❌ **Cons:**
- **All pods on the node share the same permissions**
- Overly broad access (e.g., DynamoDB FullAccess, SNS FullAccess)
- Violates principle of least privilege
- Cannot restrict access per-pod/per-service

### IRSA Support Available But Not Used
The EKS cluster **does support IRSA** (IAM Roles for Service Accounts):
- OIDC provider is created: `arn:aws:iam::126586115171:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/795E2CC0E5FA0431029B21C3DC0082AB`
- `enable_irsa` is enabled in the EKS module (line 394)
- ADOT collector uses IRSA (see lines 553-577 in `modules/aws/eks/main.tf`)

**However, the blueprint-service pod does not use IRSA** because:
1. No Kubernetes ServiceAccount with IAM role annotation is created
2. The deployment doesn't reference a custom ServiceAccount
3. The Helm chart has `serviceAccount.create: true` but doesn't configure IAM annotations

## AWS Services Accessed by fs-blueprint Pod

Based on the configuration and IAM policies:

1. **AWS Secrets Manager** - Database credentials retrieval
2. **AWS Glue Schema Registry** - Kafka schema management
3. **Amazon DynamoDB** - Data storage (full access via node role)
4. **Amazon SNS** - Notifications (full access via node role)
5. **Amazon MSK (Kafka)** - Event streaming
6. **Amazon S3** - Document storage via custom-read-write-s3-docms policy
7. **Amazon SES** - Email sending (can send emails)
8. **Amazon CloudWatch** - Logging and metrics

## Recommended Improvements

To follow AWS best practices, consider implementing IRSA for the blueprint service:

### 1. Create IAM Role for Service Account
```hcl
resource "aws_iam_role" "blueprint_service" {
  name = "fs-ue1-dev-eks-blueprint-service-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::126586115171:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/795E2CC0E5FA0431029B21C3DC0082AB"
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "oidc.eks.us-east-1.amazonaws.com/id/795E2CC0E5FA0431029B21C3DC0082AB:sub" = "system:serviceaccount:default:fs-blueprint"
          "oidc.eks.us-east-1.amazonaws.com/id/795E2CC0E5FA0431029B21C3DC0082AB:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

# Attach only the required policies
resource "aws_iam_role_policy_attachment" "blueprint_secrets" {
  role       = aws_iam_role.blueprint_service.name
  policy_arn = "arn:aws:iam::126586115171:policy/secret-manager-read"
}

# ... attach other minimal required policies
```

### 2. Create Kubernetes ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fs-blueprint
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::126586115171:role/fs-ue1-dev-eks-blueprint-service-role
```

### 3. Update Helm Deployment
```yaml
spec:
  serviceAccountName: fs-blueprint  # Add this line
  containers:
    - name: fs-blueprint-container
      ...
```

## Summary

The `fs-blueprint` pod accesses AWS services through the **EC2 Node IAM Role** mechanism:
- Pod inherits IAM permissions from the EKS worker node it runs on
- AWS SDK automatically discovers credentials via EC2 IMDS
- Multiple IAM policies attached to node group provide broad access
- While IRSA is available in the cluster, it's not configured for this specific pod
- Current approach works but provides excessive permissions to all pods on the node
