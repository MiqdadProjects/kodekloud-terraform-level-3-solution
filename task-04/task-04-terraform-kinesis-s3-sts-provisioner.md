# 🌟 Task 04 - Provision Kinesis Stream, S3 Bucket, and STS Identity Check with local-exec Logging Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is setting up streaming and storage infrastructure using Terraform against a **LocalStack**-compatible AWS environment. In addition to provisioning the cloud resources, the team requires that each resource creation event is logged to a local file via `local-exec` provisioners. The current AWS account identity is also retrieved using an STS data source and logged.

**Requirements:**
- Create a **Kinesis Stream** named **`devops-dev-stream`**:
  - Shard count: `1`
  - Retention period: `24` hours
  - Tags: `Environment: dev`, `Purpose: Stream ingestion`
  - `local-exec`: Log `"Kinesis Stream devops-dev-stream created"` → `kinesis_creation.log`
- Create an **S3 Bucket** named **`devops-dev-20322`**:
  - Tags: `Environment: dev`, `Owner: devops`
  - `local-exec`: Log `"S3 Bucket devops-dev-20322 created"` → `s3_creation.log`
- Use **`data.aws_caller_identity`** to retrieve the current AWS Account ID:
  - `local-exec` in a `null_resource`: Log `"Logged in as account ID:<Account ID>"` → `account_identity.log`
- All resources in a single **`main.tf`** file.
- Use **`variables.tf`** to declare (with validation):
  - `KKE_ENVIRONMENT`, `KKE_KINESIS_STREAM_NAME` (non-empty), `KKE_S3_BUCKET_NAME`
- Use **`terraform.tfvars`** for actual values.
- Use **`outputs.tf`** to export:
  - `kke_caller_identity_account_id`, `kke_kinesis_stream_name`, `kke_s3_bucket_name`
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Combine Terraform data sources, `null_resource`, and three `local-exec` provisioners to bootstrap a streaming pipeline while generating creation audit logs.

💡 **Note:** Because `local-exec` provisioners only run at creation time, the notes mention you may need to run `terraform apply` multiple times if some resources are already present (e.g., from a prior test run) and the log files need to be regenerated.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud / LocalStack  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- Kinesis Data Stream (`devops-dev-stream`)
- S3 Bucket (`devops-dev-20322`)
- `null_resource` (for STS identity logging)
- Data Source: `aws_caller_identity`

**Working Directory:** `/home/bob/terraform`

**Provisioner Log File Map:**
| Resource | Log File | Message |
|----------|----------|---------|
| `aws_kinesis_stream` | `kinesis_creation.log` | `Kinesis Stream devops-dev-stream created` |
| `aws_s3_bucket` | `s3_creation.log` | `S3 Bucket devops-dev-20322 created` |
| `null_resource` | `account_identity.log` | `Logged in as account ID:<account_id>` |

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`data "aws_caller_identity"`:** A Terraform data source that fetches the AWS account ID, user ID, and ARN of the current caller. No infrastructure is created.
- **`null_resource`:** A placeholder resource with no real AWS counterpart. Used here purely to trigger the `local-exec` provisioner for the STS log, bound to the account ID via `triggers`.
- **`provisioner "local-exec"`:** Attached to the Kinesis stream and S3 bucket resources to write creation confirmation messages to local log files.
- **`triggers` block:** Ensures the `null_resource` re-runs if the account ID changes between applies.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf             # Data source, Kinesis, S3, null_resource + provisioners
├── variables.tf        # KKE_ENVIRONMENT, KKE_KINESIS_STREAM_NAME, KKE_S3_BUCKET_NAME
├── terraform.tfvars    # Values for all three variables
└── outputs.tf          # Account ID, stream name, bucket name
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

variable "KKE_KINESIS_STREAM_NAME" {
  description = "Kinesis Stream name"
  type        = string

  validation {
    condition     = length(var.KKE_KINESIS_STREAM_NAME) > 0
    error_message = "Kinesis stream name must not be empty."
  }
}

variable "KKE_S3_BUCKET_NAME" {
  description = "S3 bucket name"
  type        = string
}
EOF
```

---

### Step 3: Create Terraform Variables File

```bash
cat > terraform.tfvars << 'EOF'
KKE_ENVIRONMENT         = "dev"
KKE_KINESIS_STREAM_NAME = "devops-dev-stream"
KKE_S3_BUCKET_NAME      = "devops-dev-20322"
EOF
```

---

### Step 4: Create Main Configuration File

```bash
cat > main.tf << 'EOF'
# --------------------------------
# Get AWS Account ID using STS
# --------------------------------
data "aws_caller_identity" "current" {}

resource "null_resource" "log_account_identity" {
  triggers = {
    account_id = data.aws_caller_identity.current.account_id
  }

  provisioner "local-exec" {
    command = "echo Logged in as account ID:${data.aws_caller_identity.current.account_id} > /home/bob/terraform/account_identity.log"
  }
}

# --------------------------------
# Kinesis Stream
# --------------------------------
resource "aws_kinesis_stream" "dev_stream" {
  name             = var.KKE_KINESIS_STREAM_NAME
  shard_count      = 1
  retention_period = 24

  tags = {
    Environment = var.KKE_ENVIRONMENT
    Purpose     = "Stream ingestion"
  }

  provisioner "local-exec" {
    command = "echo Kinesis Stream ${var.KKE_KINESIS_STREAM_NAME} created > /home/bob/terraform/kinesis_creation.log"
  }
}

# --------------------------------
# S3 Bucket
# --------------------------------
resource "aws_s3_bucket" "dev_bucket" {
  bucket = var.KKE_S3_BUCKET_NAME

  tags = {
    Environment = var.KKE_ENVIRONMENT
    Owner       = "devops"
  }

  provisioner "local-exec" {
    command = "echo S3 Bucket ${var.KKE_S3_BUCKET_NAME} created > /home/bob/terraform/s3_creation.log"
  }
}
EOF
```

**Configuration Breakdown:**
- **`data "aws_caller_identity"`:** Zero-cost read-only call to AWS STS. Available as `data.aws_caller_identity.current.account_id`.
- **`null_resource.triggers`:** The map `{ account_id = ... }` forces the `null_resource` to be destroyed/recreated if the account ID changes, which re-executes the `local-exec` provisioner.
- **Provisioner `command`:** Uses shell redirection (`>`) to overwrite the log file on each creation event.

---

### Step 5: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_caller_identity_account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "kke_kinesis_stream_name" {
  value = aws_kinesis_stream.dev_stream.name
}

output "kke_s3_bucket_name" {
  value = aws_s3_bucket.dev_bucket.bucket
}
EOF
```

---

### Step 6: Initialize and Apply

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

kke_caller_identity_account_id = "123456789012"
kke_kinesis_stream_name = "devops-dev-stream"
kke_s3_bucket_name = "devops-dev-20322"
```

---

### Step 7: Verify Log Files

```bash
cat /home/bob/terraform/kinesis_creation.log
# Expected: Kinesis Stream devops-dev-stream created

cat /home/bob/terraform/s3_creation.log
# Expected: S3 Bucket devops-dev-20322 created

cat /home/bob/terraform/account_identity.log
# Expected: Logged in as account ID:123456789012
```

---

### Step 8: Final Verification (Critical)

```bash
terraform plan
```

**Expected Output:**
```
No changes. Your infrastructure matches the configuration.
```

---

## ✅ Verification Steps

### Step 1: State List

```bash
terraform state list
```

**Expected Output:**
```
data.aws_caller_identity.current
aws_kinesis_stream.dev_stream
aws_s3_bucket.dev_bucket
null_resource.log_account_identity
```

---

### Step 2: Verify Kinesis Stream Tags

```bash
terraform state show aws_kinesis_stream.dev_stream
```

**Verify:** Tags include `Environment = "dev"` and `Purpose = "Stream ingestion"`.

---

### Step 3: Verify S3 Bucket Tags

```bash
terraform state show aws_s3_bucket.dev_bucket
```

**Verify:** Tags include `Environment = "dev"` and `Owner = "devops"`.

---

## 🔍 Code Analysis

### `null_resource` — Purpose and Lifecycle
`null_resource` is a special Terraform resource with **no corresponding real infrastructure**. Its value lies entirely in its provisioners and `triggers` mechanism.

| Feature | Purpose |
|---------|---------|
| `triggers` | Map of arbitrary strings. If any value changes between applies, the resource is destroyed and re-created, re-running provisioners. |
| `provisioner "local-exec"` | Runs a shell command on the **Terraform runner machine** (not on a remote server). |

### Creation-Time vs. Destroy-Time Provisioners
By default, `local-exec` — and all provisioners — run **only at resource creation time**. To run on deletion, you would add `when = destroy`:
```hcl
provisioner "local-exec" {
  when    = destroy
  command = "echo resource destroyed"
}
```

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Value |
|----------|-------|
| Kinesis Stream | `devops-dev-stream` (1 shard, 24h retention) |
| S3 Bucket | `devops-dev-20322` |
| Identity | STS Account ID (logged to file) |

### Task Completion Checklist
- [x] ✅ Kinesis stream `devops-dev-stream` created with correct tags.
- [x] ✅ S3 bucket `devops-dev-20322` created with correct tags.
- [x] ✅ `data.aws_caller_identity` used to fetch account ID.
- [x] ✅ `null_resource` with `local-exec` logs account identity to file.
- [x] ✅ All three `local-exec` provisioners log correct messages to correct files.
- [x] ✅ Variable validation for non-empty Kinesis stream name.
- [x] ✅ Three outputs exported (account ID, stream name, bucket name).
- [x] ✅ Final `terraform plan` returns **"No changes"**.
