# 🌟 Task 02 - Provision Kinesis Firehose Data Pipeline to S3 Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is building a real-time data ingestion pipeline. The objective is to create an **AWS Kinesis Firehose** delivery stream that receives streaming records, appends a newline delimiter after each one, and delivers batched data to an **S3 bucket**. The Firehose must securely access the S3 bucket by assuming an **IAM role via STS**, following the principle of least privilege.

**Requirements:**
- Create an **S3 bucket** named **`xfusion-stream-bucket-15589`**.
- Create an **IAM role** named **`firehose-sts-role`** trusted by `firehose.amazonaws.com`.
- Create an **IAM policy** (`firehose-s3-policy`) granting `s3:PutObject` and `s3:PutObjectAcl` on the bucket.
- Attach the policy to the role.
- Create a **Kinesis Firehose delivery stream** named **`xfusion-firehose-stream`**:
  - Destination: `extended_s3`
  - Role ARN: from `firehose-sts-role`
  - **Buffering:** 5 MB size / 300 second interval
  - **Processing:** `AppendDelimiterToRecord` processor with `Delimiter = "\n"`
  - Use **`depends_on`** to wait for the IAM policy attachment.
- Use **`variables.tf`** to declare:
  - `KKE_S3_BUCKET_NAME`, `KKE_FIREHOSE_STREAM_NAME`, `KKE_FIREHOSE_ROLE_NAME`
- Use **`terraform.tfvars`** for actual values.
- Use **`outputs.tf`** to export:
  - `kke_firehose_stream_name`, `kke_s3_bucket_name`, `kke_firehose_role_arn`
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Build the IAM trust infrastructure and Kinesis Firehose stream, ensuring secure S3 delivery with proper buffering and newline formatting.

💡 **Note:** The `depends_on` block on the Firehose resource is critical. Without it, Terraform may try to create the stream before the IAM policy is fully attached, causing a permissions error on the first apply.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- S3 Bucket (`xfusion-stream-bucket-15589`) — delivery destination
- IAM Role (`firehose-sts-role`) — identity for Firehose
- IAM Policy (`firehose-s3-policy`) — S3 write permissions
- IAM Policy Attachment — links role and policy
- Kinesis Firehose Stream (`xfusion-firehose-stream`) — data ingestion

**Working Directory:** `/home/bob/terraform`

**Data Pipeline Flow:**
```
[Data Producer]
        │
        ▼ PutRecord / PutRecordBatch
[Kinesis Firehose: xfusion-firehose-stream]
        │ Assume Role via STS (firehose-sts-role)
        │ Buffer: 5 MB OR 300 sec
        │ Process: Append "\n" delimiter
        ▼
[S3 Bucket: xfusion-stream-bucket-15589]
```

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`aws_s3_bucket`:** The final destination for all Firehose records.
- **`aws_iam_role`:** Grants Firehose the identity it needs via STS Assume Role.
- **`aws_iam_policy`:** Scoped `s3:PutObject` permission on the bucket's ARN prefix.
- **`aws_iam_role_policy_attachment`:** Binds the policy to the role.
- **`aws_kinesis_firehose_delivery_stream`:** The core pipeline resource. Uses `extended_s3_configuration` for buffering and the `AppendDelimiterToRecord` processor for newline formatting.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # All resources
├── variables.tf       # Variable declarations
├── terraform.tfvars   # Values for bucket, stream, and role names
└── outputs.tf         # Stream name, bucket name, role ARN outputs
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
variable "KKE_S3_BUCKET_NAME" {
  description = "Name of the S3 bucket"
  type        = string
}

variable "KKE_FIREHOSE_STREAM_NAME" {
  description = "Name of the Firehose delivery stream"
  type        = string
}

variable "KKE_FIREHOSE_ROLE_NAME" {
  description = "Name of the Firehose IAM role"
  type        = string
}
EOF
```

---

### Step 3: Create Terraform Variables File

```bash
cat > terraform.tfvars << 'EOF'
KKE_S3_BUCKET_NAME       = "xfusion-stream-bucket-15589"
KKE_FIREHOSE_STREAM_NAME = "xfusion-firehose-stream"
KKE_FIREHOSE_ROLE_NAME   = "firehose-sts-role"
EOF
```

---

### Step 4: Create Main Configuration File

```bash
cat > main.tf << 'EOF'
# ---------------------------
# S3 Bucket
# ---------------------------
resource "aws_s3_bucket" "firehose_bucket" {
  bucket = var.KKE_S3_BUCKET_NAME
}

# ---------------------------
# IAM Role for Firehose (STS Assume Role)
# ---------------------------
resource "aws_iam_role" "firehose_role" {
  name = var.KKE_FIREHOSE_ROLE_NAME

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "firehose.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

# ---------------------------
# IAM Policy for S3 Access
# ---------------------------
resource "aws_iam_policy" "firehose_s3_policy" {
  name = "firehose-s3-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:PutObjectAcl"
        ]
        Resource = "${aws_s3_bucket.firehose_bucket.arn}/*"
      }
    ]
  })
}

# ---------------------------
# Attach Policy to Role
# ---------------------------
resource "aws_iam_role_policy_attachment" "firehose_attach" {
  role       = aws_iam_role.firehose_role.name
  policy_arn = aws_iam_policy.firehose_s3_policy.arn
}

# ---------------------------
# Firehose Delivery Stream
# ---------------------------
resource "aws_kinesis_firehose_delivery_stream" "firehose_stream" {
  name        = var.KKE_FIREHOSE_STREAM_NAME
  destination = "extended_s3"

  extended_s3_configuration {
    role_arn           = aws_iam_role.firehose_role.arn
    bucket_arn         = aws_s3_bucket.firehose_bucket.arn
    buffering_size     = 5
    buffering_interval = 300

    processing_configuration {
      enabled = true

      processors {
        type = "AppendDelimiterToRecord"

        parameters {
          parameter_name  = "Delimiter"
          parameter_value = "\\n"
        }
      }
    }
  }

  # Wait for the IAM policy to be fully attached before creating the stream
  depends_on = [
    aws_iam_role_policy_attachment.firehose_attach
  ]
}
EOF
```

**Key Configuration Details:**
- **`destination = "extended_s3"`:** Required for using `extended_s3_configuration` with processing features.
- **`buffering_size = 5`:** Triggers delivery when the buffer reaches 5 MB.
- **`buffering_interval = 300`:** Fallback — delivers after 300s even if buffer isn't full.
- **`AppendDelimiterToRecord`:** Post-processes each record to add a `\n`, making each S3 file line-delimited for easier parsing.
- **`depends_on`:** Explicit dependency ensuring IAM propagation has fully completed before Firehose attempts to assume the role.

---

### Step 5: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_firehose_stream_name" {
  value = aws_kinesis_firehose_delivery_stream.firehose_stream.name
}

output "kke_s3_bucket_name" {
  value = aws_s3_bucket.firehose_bucket.bucket
}

output "kke_firehose_role_arn" {
  value = aws_iam_role.firehose_role.arn
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
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

kke_firehose_role_arn = "arn:aws:iam::123456789012:role/firehose-sts-role"
kke_firehose_stream_name = "xfusion-firehose-stream"
kke_s3_bucket_name = "xfusion-stream-bucket-15589"
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

---

## ✅ Verification Steps

### Step 1: Verify State List

```bash
terraform state list
```

**Expected Output:**
```
aws_iam_policy.firehose_s3_policy
aws_iam_role.firehose_role
aws_iam_role_policy_attachment.firehose_attach
aws_kinesis_firehose_delivery_stream.firehose_stream
aws_s3_bucket.firehose_bucket
```

---

### Step 2: Send Test Data to Firehose

```bash
aws firehose put-record \
  --delivery-stream-name xfusion-firehose-stream \
  --record '{"Data":"dGVzdC1kYXRhLXJlY29yZA=="}'
```

*(The `Data` value is the Base64 encoding of `test-data-record`)*

---

### Step 3: Verify S3 Data with Newline Delimiter

After waiting for the buffer interval, list and check the delivered files:

```bash
# List delivered files
aws s3 ls s3://xfusion-stream-bucket-15589/ --recursive

# Check that records end with newline
aws s3 cp s3://xfusion-stream-bucket-15589/<path-to-file> - | cat -A | head
```

**Expected:** Each record line ends with `$` (cat -A shows `\n` as `$`), confirming the delimiter processor worked correctly.

---

## 🔍 Code Analysis

### `extended_s3` vs `s3` Destination

| Feature | `s3` Destination | `extended_s3` Destination |
|---------|-----------------|--------------------------|
| Record Processing | ❌ Not supported | ✅ `AppendDelimiterToRecord` |
| Data Transformation | ❌ No Lambda support | ✅ Lambda transforms supported |
| Prefix Configuration | Basic | Advanced dynamic prefixes |

### Why `depends_on` is Required Here
IAM policies involve **eventual consistency**. When Terraform creates the `aws_iam_role_policy_attachment`, the permissions may not have fully propagated to AWS's internal systems before the Firehose resource is created. The `depends_on` introduces a deliberate sequencing order to avoid a `AccessDeniedException` during Firehose stream creation.

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Value |
|----------|-------|
| S3 Bucket | `xfusion-stream-bucket-15589` |
| IAM Role | `firehose-sts-role` |
| IAM Policy | `firehose-s3-policy` |
| Firehose Stream | `xfusion-firehose-stream` |

### Task Completion Checklist
- [x] ✅ S3 bucket `xfusion-stream-bucket-15589` provisioned.
- [x] ✅ IAM role `firehose-sts-role` created with Firehose trust.
- [x] ✅ IAM policy granting S3 `PutObject` created and attached.
- [x] ✅ Firehose stream with `extended_s3` destination created.
- [x] ✅ Buffering: 5 MB size / 300 second interval.
- [x] ✅ `AppendDelimiterToRecord` processor with `\n` configured.
- [x] ✅ `depends_on` used to ensure IAM propagation.
- [x] ✅ Variables and tfvars used for all names.
- [x] ✅ Three outputs exported (stream, bucket, role ARN).
- [x] ✅ Final `terraform plan` returns **"No changes"**.
