# 🌟 Task 06 - Provision Multi-Tier Infrastructure (DynamoDB, SNS, SSM) Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is building a secure, modular multi-tier AWS infrastructure to support a modern cloud-native application stack. This involves provisioning core storage (DynamoDB), messaging (SNS), and configuration management (SSM Parameter Store) components using a clean, variable-driven approach.

**Requirements:**
- Create a **DynamoDB Table** named **`devops-app-table`** with:
  - Hash Key: `id` (String)
  - Billing Mode: `PAY_PER_REQUEST`
- Create an **SNS Topic** named **`devops-app-topic`** for application notifications.
- Create an **SSM Parameter** named **`/devops/app/config`**:
  - Type: `SecureString`
  - Value: `devops-secure-config-value`
- Resources must be tagged with the environment name.
- Use a single **`main.tf`** for all resources.
- Use **`variables.tf`** for:
  - `KKE_ENVIRONMENT` (e.g., `devEnvironment`)
  - `KKE_DYNAMODB_TABLE_NAME`
  - `KKE_SNS_TOPIC_NAME`
  - `KKE_SSM_PARAM_NAME`
- Use **`terraform.tfvars`** for variable inputs.
- Use **`outputs.tf`** to export:
  - `kke_dynamodb_table_name`
  - `kke_sns_topic_arn`
  - `kke_ssm_parameter_name`
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Orchestrate a cross-service infrastructure deployment including database, messaging, and secret management layers using Terraform.

💡 **Note:** Using `SecureString` for SSM Parameters ensures that the sensitive configuration value is encrypted at rest in the AWS Systems Manager service.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- DynamoDB Table (`devops-app-table`)
- SNS Topic (`devops-app-topic`)
- SSM Parameter (`/devops/app/config`)

**Working Directory:** `/home/bob/terraform`

**Logical Architecture:**
```
[Application Layer]
        │
        ├─► [SSM Parameter Store] (Configuration & Secrets)
        │
        ├─► [DynamoDB Table] (NoSQL Storage)
        │
        └─► [SNS Topic] (Event Messaging)
```

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`aws_dynamodb_table`:** Provides a schema-less NoSQL database table.
- **`aws_sns_topic`:** Enables Pub/Sub messaging for the application.
- **`aws_ssm_parameter`:** Centralizes configuration management, using `SecureString` for best practices.
- **Variable-Centric Logic:** All resource names and environment tags are driven by external variables, making the configuration modular and environment-agnostic.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # Cross-service resource definitions
├── variables.tf       # Parameter declarations
├── terraform.tfvars   # Variable values (Table, Topic, Param names)
└── outputs.tf         # Exported names and ARNs
```

---

## 🚀 Implementation Steps

### Step 1: Navigate to Working Directory

```bash
cd /home/bob/terraform
```

---

### Step 2: Create Variables Definition File

```bash
cat > variables.tf << 'EOF'
variable "KKE_ENVIRONMENT" {
  description = "Environment name"
  type        = string
}

variable "KKE_DYNAMODB_TABLE_NAME" {
  description = "Name of the DynamoDB table"
  type        = string
}

variable "KKE_SNS_TOPIC_NAME" {
  description = "Name of the SNS topic"
  type        = string
}

variable "KKE_SSM_PARAM_NAME" {
  description = "Name of the SSM parameter"
  type        = string
}
EOF
```

---

### Step 3: Create Terraform Variables File

```bash
cat > terraform.tfvars << 'EOF'
KKE_ENVIRONMENT         = "devEnvironment"
KKE_DYNAMODB_TABLE_NAME = "devops-app-table"
KKE_SNS_TOPIC_NAME      = "devops-app-topic"
KKE_SSM_PARAM_NAME      = "/devops/app/config"
EOF
```

---

### Step 4: Create Main Configuration File

```bash
cat > main.tf << 'EOF'
# ---------------------------
# DynamoDB Table
# ---------------------------
resource "aws_dynamodb_table" "devops_dynamodb" {
  name         = var.KKE_DYNAMODB_TABLE_NAME
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {
    name = "id"
    type = "S"
  }

  tags = {
    Environment = var.KKE_ENVIRONMENT
  }
}

# ---------------------------
# SNS Topic
# ---------------------------
resource "aws_sns_topic" "devops_sns" {
  name = var.KKE_SNS_TOPIC_NAME

  tags = {
    Environment = var.KKE_ENVIRONMENT
  }
}

# ---------------------------
# SSM Parameter
# ---------------------------
resource "aws_ssm_parameter" "devops_ssm" {
  name  = var.KKE_SSM_PARAM_NAME
  type  = "SecureString"
  value = "devops-secure-config-value"

  tags = {
    Environment = var.KKE_ENVIRONMENT
  }
}
EOF
```

---

### Step 5: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_dynamodb_table_name" {
  description = "Name of the DynamoDB table"
  value       = aws_dynamodb_table.devops_dynamodb.name
}

output "kke_sns_topic_arn" {
  description = "ARN of the SNS topic"
  value       = aws_sns_topic.devops_sns.arn
}

output "kke_ssm_parameter_name" {
  description = "Name of the SSM parameter"
  value       = aws_ssm_parameter.devops_ssm.name
}
EOF
```

---

### Step 6: Initialize, Validate, and Apply

```bash
terraform init
terraform validate
terraform fmt
terraform apply -auto-approve
```

**Expected Success Message:**
```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

kke_dynamodb_table_name = "devops-app-table"
kke_sns_topic_arn = "arn:aws:sns:us-east-1:123456789012:devops-app-topic"
kke_ssm_parameter_name = "/devops/app/config"
```

---

### Step 7: Final Verification (Critical)

```bash
terraform plan
```

**Expected Output:**
```
No changes. Your infrastructure matches the configuration.
```

✅ **Task complete.**

---

## ✅ Verification Steps

### Step 1: Verify State List

```bash
terraform state list
```

---

### Step 2: Verify SSM Parameter Encryption

```bash
aws ssm get-parameter --name "/devops/app/config" --with-decryption
```

**Verify:** Confirm the `Type` is `SecureString` and the value matches.

---

### Step 3: Verify DynamoDB Table Status

```bash
aws dynamodb describe-table --table-name devops-app-table
```

---

## 🔍 Code Analysis

### Resource Tagging Strategy
Each resource includes an `Environment` tag driven by the `KKE_ENVIRONMENT` variable. In professional environments, this allows for cost allocation and access control filtering across different stages (dev, staging, prod) using the same code base.

### SSM SecureString
By choosing `type = "SecureString"`, Terraform tells AWS to encrypt the parameter value using the default KMS key for SSM (`alias/aws/ssm`). This is a security requirement for any sensitive application configuration.

### Pay-Per-Request Billing
For the DynamoDB table, `billing_mode = "PAY_PER_REQUEST"` (On-Demand) is chosen. This is ideal for development environments or applications with unpredictable traffic, as you only pay for the actual read/write requests performed.

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Value |
|----------|-------|
| DynamoDB Table | `devops-app-table` |
| SNS Topic | `devops-app-topic` |
| SSM Parameter | `/devops/app/config` |

### Task Completion Checklist
- [x] ✅ DynamoDB table `devops-app-table` provisioned.
- [x] ✅ SNS topic `devops-app-topic` created.
- [x] ✅ SSM parameter `/devops/app/config` created as `SecureString`.
- [x] ✅ Environment-based tagging applied to all resources.
- [x] ✅ Modular variable usage via `variables.tf` and `terraform.tfvars`.
- [x] ✅ Outputs for table name, SNS ARN, and SSM name exported.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
