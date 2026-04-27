# 🌟 Task 07 - Provision Multi-Environment S3 Buckets with for_each and Lifecycle Policies

## 📌 Task Description

The **Nautilus DevOps team** is standardizing S3 bucket deployments across three environments: **Dev**, **Staging**, and **Prod**. This task requires using Terraform's `for_each` meta-argument to dynamically provision buckets with environment-specific tags, lifecycle rules for cost optimization, and public read policies.

**Requirements:**
- Create three **S3 Buckets** using **`for_each`**:
  - `datacenter-dev-bucket-27681`
  - `datacenter-staging-bucket-27681`
  - `datacenter-prod-bucket-27681`
- **Tagging Configuration:**
  - Every bucket must have `Name`, `Environment`, and `Owner` tags.
  - Staging and Prod buckets must also have `Backup = true`.
- **Lifecycle Management:**
  - For **Staging** and **Prod** only, transition objects to **GLACIER** storage after **30 days** (ID: `MoveToGlacier`).
- **State Protection:**
  - Use the `lifecycle` block with `ignore_changes = [tags]` to prevent Terraform from fighting with external tag modifications.
- **Security Policy:**
  - Apply a bucket policy to all three buckets that allows **public read access** (`s3:GetObject`) to all objects.
  - Use **`depends_on`** to ensure the policy is applied *after* the bucket is created.
- All resource logic must be in a single **`main.tf`** file.
- Use **`variables.tf`** to declare the map `KKE_ENV_TAGS`.
- Use **`outputs.tf`** to export:
  - `kke_bucket_names`
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Implement a dynamic S3 provisioning system using `for_each` and conditional lifecycle logic to manage cross-environment storage requirements efficiently.

💡 **Note:** Transitioning data to Glacier storage after 30 days is a key cost-optimization strategy for backup and production data that is rarely accessed but must be retained.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- S3 Buckets (3x)
- S3 Bucket Lifecycle Configurations (2x - Staging & Prod)
- S3 Bucket Policies (3x)

**Working Directory:** `/home/bob/terraform`

**Infrastructure Matrix:**
| Environment | Bucket Name | Owner | Backup | Glacier Transition |
|-------------|-------------|-------|--------|--------------------|
| **Dev** | `datacenter-dev-bucket-27681` | Alice | False | ❌ None |
| **Staging** | `datacenter-staging-bucket-27681` | Bob | True | ✅ 30 Days |
| **Prod** | `datacenter-prod-bucket-27681` | Carol | True | ✅ 30 Days |

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`for_each = var.KKE_ENV_TAGS`:** Used in `aws_s3_bucket` to iterate through the map and create all three buckets with a single resource block.
- **Conditional for_each:** The `aws_s3_bucket_lifecycle_configuration` uses a `for` loop filter (`if v.backup == true`) to only apply lifecycle rules to the environments marked for backup.
- **`aws_s3_bucket_policy`:** Dynamically generates a JSON policy for each bucket, using the bucket's ARN to define the resource scope.
- **`ignore_changes`:** Ensures that if an administrator manually adds or changes a tag in the AWS Console, Terraform won't attempt to overwrite it, reducing state churn.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # S3 bucket, Lifecycle, and Policy logic
├── variables.tf       # KKE_ENV_TAGS map declaration
├── terraform.tfvars   # Input data for the three environments
└── outputs.tf         # Exported bucket names list
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
variable "KKE_ENV_TAGS" {
  description = "Map of environment-specific metadata for S3 buckets"
  type = map(object({
    bucket_name = string
    environment = string
    owner       = string
    backup      = bool
  }))
}
EOF
```

---

### Step 3: Create Terraform Variables File

```bash
cat > terraform.tfvars << 'EOF'
KKE_ENV_TAGS = {
  dev = {
    bucket_name = "datacenter-dev-bucket-27681"
    environment = "Dev"
    owner       = "Alice"
    backup      = false
  }
  staging = {
    bucket_name = "datacenter-staging-bucket-27681"
    environment = "Staging"
    owner       = "Bob"
    backup      = true
  }
  prod = {
    bucket_name = "datacenter-prod-bucket-27681"
    environment = "Prod"
    owner       = "Carol"
    backup      = true
  }
}
EOF
```

---

### Step 4: Create Main Configuration File

```bash
cat > main.tf << 'EOF'
# ---------------------------
# S3 Buckets (Multi-Environment)
# ---------------------------
resource "aws_s3_bucket" "datacenter_buckets" {
  for_each = var.KKE_ENV_TAGS

  bucket = each.value.bucket_name

  tags = {
    Name        = each.value.bucket_name
    Environment = each.value.environment
    Owner       = each.value.owner
    Backup      = each.value.backup
  }

  lifecycle {
    ignore_changes = [tags]
  }
}

# ---------------------------
# Lifecycle Configuration
# ---------------------------
# Applied only to Staging and Prod (v.backup == true)
resource "aws_s3_bucket_lifecycle_configuration" "glacier_lifecycle" {
  for_each = { for k, v in var.KKE_ENV_TAGS : k => v if v.backup == true }

  bucket = aws_s3_bucket.datacenter_buckets[each.key].id

  rule {
    id     = "MoveToGlacier"
    status = "Enabled"

    filter {
      prefix = "" # Apply to all objects in the bucket
    }

    transition {
      days          = 30
      storage_class = "GLACIER"
    }
  }
}

# ---------------------------
# Bucket Policy (Public Read)
# ---------------------------
resource "aws_s3_bucket_policy" "public_read" {
  for_each = var.KKE_ENV_TAGS

  bucket = aws_s3_bucket.datacenter_buckets[each.key].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.datacenter_buckets[each.key].arn}/*"
      }
    ]
  })

  depends_on = [aws_s3_bucket.datacenter_buckets]
}
EOF
```

---

### Step 5: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_bucket_names" {
  description = "Names of the created S3 buckets"
  value       = [for b in aws_s3_bucket.datacenter_buckets : b.bucket]
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
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

kke_bucket_names = [
  "datacenter-dev-bucket-27681",
  "datacenter-prod-bucket-27681",
  "datacenter-staging-bucket-27681",
]
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

**Expected Output:**
- `aws_s3_bucket.datacenter_buckets["dev"]`
- `aws_s3_bucket.datacenter_buckets["prod"]`
- `aws_s3_bucket.datacenter_buckets["staging"]`
- `aws_s3_bucket_lifecycle_configuration.glacier_lifecycle["prod"]`
- `aws_s3_bucket_lifecycle_configuration.glacier_lifecycle["staging"]`
- `aws_s3_bucket_policy.public_read["dev"]`
- `aws_s3_bucket_policy.public_read["prod"]`
- `aws_s3_bucket_policy.public_read["staging"]`

---

### Step 2: Verify Lifecycle Configuration

```bash
aws s3api get-bucket-lifecycle-configuration --bucket datacenter-prod-bucket-27681
```

**Verify:** Confirm the `MoveToGlacier` rule with `StorageClass: GLACIER` and `Days: 30`.

---

### Step 3: Verify Public Read Policy

```bash
aws s3api get-bucket-policy --bucket datacenter-dev-bucket-27681
```

**Verify:** Confirm the `Principal: *` and `Action: s3:GetObject` are present.

---

## 🔍 Code Analysis

### Dynamic Iteration with `for_each`
`for_each` is preferred over `count` when resources have distinct configurations. Because each bucket has a different name, owner, and backup status, `for_each` allows us to reference individual resource instances by their key (e.g., `["dev"]`) rather than an index number. This makes the state much more resilient to deletions or reorderings.

### Filtering in `for_each`
The lifecycle resource uses a sophisticated filter:
`for_each = { for k, v in var.KKE_ENV_TAGS : k => v if v.backup == true }`
This ensures that Terraform only attempts to create a lifecycle configuration for buckets where the metadata specifies `backup = true`. If we added a fourth environment with `backup = false`, Terraform would automatically skip creating a lifecycle rule for it.

### The `depends_on` Attribute
Applying a bucket policy often requires the bucket itself to be in a "Ready" state. While referencing `aws_s3_bucket.datacenter_buckets[each.key].id` creates an implicit dependency, adding an explicit `depends_on = [aws_s3_bucket.datacenter_buckets]` ensures the S3 service has finished propagating the bucket creation across all regional endpoints before the policy application is attempted.

---

## 🎯 Task Completion Summary

### Resources Created

| Environment | Resource Type | Managed Resource |
|-------------|---------------|------------------|
| **Dev** | S3 Bucket | `datacenter-dev-bucket-27681` |
| **Dev** | Policy | Public Read Policy |
| **Staging** | S3 Bucket | `datacenter-staging-bucket-27681` |
| **Staging** | Lifecycle | Glacier Transition (30 days) |
| **Staging** | Policy | Public Read Policy |
| **Prod** | S3 Bucket | `datacenter-prod-bucket-27681` |
| **Prod** | Lifecycle | Glacier Transition (30 days) |
| **Prod** | Policy | Public Read Policy |

### Task Completion Checklist
- [x] ✅ Three S3 buckets provisioned using `for_each`.
- [x] ✅ Environment-specific tags (Name, Environment, Owner, Backup) applied.
- [x] ✅ Lifecycle rule `MoveToGlacier` applied only to Staging and Prod.
- [x] ✅ `ignore_changes = [tags]` implemented for all buckets.
- [x] ✅ Public read policy (`s3:GetObject`) applied to all buckets.
- [x] ✅ `depends_on` used to sequence policy application.
- [x] ✅ Variables and tfvars used for multi-env metadata.
- [x] ✅ Output `kke_bucket_names` exported and verified.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
