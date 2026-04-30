# 🌟 Task 09 - Securely Manage Sensitive Information using AWS Secrets Manager

## 📌 Task Description

The **Nautilus DevOps team** needs to securely manage sensitive information, specifically a database password, using **AWS Secrets Manager**. The objective is to provision the secret using Terraform while ensuring that the actual password is never exposed in the console output or logs. This is achieved by marking the Terraform variables and outputs as `sensitive`.

**Requirements:**
- Create an **AWS Secrets Manager secret** named **`xfusion-db-password`**.
- Store the database password **`SuperSecretPassword123!`** in the secret.
- The password must be passed via a variable marked as **sensitive**.
- The output displaying the password must also be marked as **sensitive** to prevent exposure.
- All resource definitions should be in a single **`main.tf`** file.
- Use **`variables.tf`** to declare:
  - `KKE_DB_PASSWORD` (must be sensitive).
- Use **`terraform.tfvars`** to supply the actual password value.
- Use **`outputs.tf`** to export:
  - `kke_secret_arn` — ARN of the created secret.
  - `kke_secret_string` — The database password (must be sensitive).
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Provision a Secrets Manager secret and its version while strictly adhering to Terraform's security best practices for handling sensitive data.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- AWS Secrets Manager Secret (`xfusion-db-password`)
- AWS Secrets Manager Secret Version (holds the actual string)

**Working Directory:** `/home/bob/terraform`

**Data Flow (Secure):**
```
[terraform.tfvars] (Raw Password)
        │
        ▼ (Injected as sensitive variable)
[variables.tf] (KKE_DB_PASSWORD marked sensitive=true)
        │
        ▼ (Used in resource, hidden from CLI output)
[main.tf] (aws_secretsmanager_secret_version)
        │
        ▼ (Stored securely in AWS)
[AWS Secrets Manager]
```

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`aws_secretsmanager_secret`:** Creates the logical container for the secret, including its name and metadata (like tags).
- **`aws_secretsmanager_secret_version`:** Attaches the actual sensitive payload (the password string) to the logical secret container created above.
- **`sensitive = true`:** A critical Terraform meta-argument used in both `variables.tf` and `outputs.tf` to instruct Terraform to redact the value from CLI output during `plan` and `apply` operations.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # Secret and Secret Version resources
├── variables.tf       # Sensitive variable declaration
├── terraform.tfvars   # The actual secret value
└── outputs.tf         # ARN output and sensitive string output
```

---

## 🚀 Implementation Steps

### Step 1: Navigate to Working Directory

```bash
cd /home/bob/terraform
```

---

### Step 2: Create Variables Definition File

Ensure the variable is explicitly marked as sensitive to prevent it from leaking into the console output.

```bash
cat > variables.tf << 'EOF'
variable "KKE_DB_PASSWORD" {
  description = "Database password to be stored in AWS Secrets Manager"
  type        = string
  sensitive   = true
}
EOF
```

---

### Step 3: Create Terraform Variables File

Provide the actual password. While this file contains the plaintext password, in a real CI/CD environment, this value would typically be injected via environment variables (e.g., `TF_VAR_KKE_DB_PASSWORD`).

```bash
cat > terraform.tfvars << 'EOF'
KKE_DB_PASSWORD = "SuperSecretPassword123!"
EOF
```

---

### Step 4: Create Main Configuration File

Define the secret container and its version.

```bash
cat > main.tf << 'EOF'
# --------------------------------
# AWS Secrets Manager Secret
# --------------------------------
resource "aws_secretsmanager_secret" "xfusion_db_password" {
  name = "xfusion-db-password"

  tags = {
    Environment = "dev"
  }
}

# --------------------------------
# AWS Secrets Manager Secret Version
# --------------------------------
resource "aws_secretsmanager_secret_version" "xfusion_db_password_version" {
  secret_id     = aws_secretsmanager_secret.xfusion_db_password.id
  secret_string = var.KKE_DB_PASSWORD
}
EOF
```

---

### Step 5: Create Outputs Configuration File

Export the ARN and the password. Because the password variable is sensitive, the output must also be marked as sensitive, otherwise Terraform will throw an error.

```bash
cat > outputs.tf << 'EOF'
output "kke_secret_arn" {
  description = "ARN of the secret created in AWS Secrets Manager"
  value       = aws_secretsmanager_secret.xfusion_db_password.arn
}

output "kke_secret_string" {
  description = "Database password stored in the secret"
  value       = var.KKE_DB_PASSWORD
  sensitive   = true
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
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

kke_secret_arn = "arn:aws:secretsmanager:us-east-1:123456789012:secret:xfusion-db-password-xxxxx"
kke_secret_string = <sensitive>
```
*Notice how the `kke_secret_string` output is redacted as `<sensitive>`.*

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

## 🔍 Code Analysis

### Secrets Manager: Secret vs. Secret Version
AWS Secrets Manager separates the concept of a secret (the metadata, rotation policies, and ARNs) from the secret version (the actual encrypted payload). 
- `aws_secretsmanager_secret` creates the outer shell.
- `aws_secretsmanager_secret_version` populates it with data. This separation allows you to update the password later, creating a new version while keeping the same ARN.

### The `sensitive = true` Flag
When `sensitive = true` is set on a variable:
1. Terraform suppresses the value in CLI output (e.g., when running `terraform plan` or `terraform apply`).
2. If that variable is used to construct an output, Terraform enforces that the corresponding `output` block must *also* have `sensitive = true`.
**Important:** The value is still stored in plaintext within the `terraform.tfstate` file. Securing the state file (e.g., using an S3 backend with encryption and strict IAM policies) is a separate but equally critical security requirement.

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Value |
|----------|-------|
| Secret Container | `xfusion-db-password` |
| Secret Payload | `<sensitive>` |

### Task Completion Checklist
- [x] ✅ AWS Secrets Manager secret `xfusion-db-password` created.
- [x] ✅ Database password stored as a secret version.
- [x] ✅ Variable `KKE_DB_PASSWORD` explicitly marked as `sensitive = true`.
- [x] ✅ Output `kke_secret_string` explicitly marked as `sensitive = true`.
- [x] ✅ Password successfully redacted from console output (`<sensitive>`).
- [x] ✅ Output `kke_secret_arn` exported correctly.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
