# 🌟 Task 03 - Provision IAM User and Role with Locals-Based Naming and Tagging Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is enforcing strict naming and tagging conventions for all IAM resources. Resource names must be derived dynamically from input variables using Terraform's `locals` block, ensuring names are consistently lowercase and hyphen-separated regardless of how variables are supplied.

**Requirements:**
- Create an **IAM User** with name: `{project}-{team}-user` (lowercase, non-alphanumeric replaced with `-`).
- Create an **IAM Role** with name: `{project}-{team}-role` with EC2 trust policy.
- Apply the following **tags** to **both** resources:
  - `Project`, `Team`, `ManagedBy`, `Env`
- Apply an **additional tag** to the IAM role only:
  - `RoleType: EC2`
- Use a **`locals` block in `main.tf`** to:
  - Sanitize project/team names (lowercase, replace invalid chars with `-`)
  - Create a `name_prefix` local (`project-team`)
  - Define reusable `common_tags` map
- Use **`variables.tf`** to declare (with validations):
  - `KKE_PROJECT` — must be non-empty
  - `KKE_TEAM` — only letters, digits, dashes, underscores
  - `KKE_ENVIRONMENT` — environment name
- Use **`terraform.tfvars`** to supply actual values.
- Use **`outputs.tf`** to export:
  - `kke_user_name`, `kke_role_name`, `kke_tags_applied`
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Implement a variables-to-names pipeline using Terraform locals, producing clean, reusable identifiers for IAM resources with consistent tagging metadata.

💡 **Note:** The `replace()` function and `lower()` function are combined in `locals` to sanitize input. `merge()` is used to combine the common tags with a role-specific tag, keeping the tag definitions DRY.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud (IAM — global service)  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- IAM User (`xfusion-dev-team-user`)
- IAM Role (`xfusion-dev-team-role`)

**Working Directory:** `/home/bob/terraform`

**Naming Derivation Logic:**
```
KKE_PROJECT = "xfusion"   ──► sanitize ──► "xfusion"
KKE_TEAM    = "dev-team"  ──► sanitize ──► "dev-team"
                                               │
                          name_prefix = "xfusion-dev-team"
                                               │
                          ┌────────────────────┴──────────────────────┐
                          │                                           │
               IAM User: "xfusion-dev-team-user"    IAM Role: "xfusion-dev-team-role"
```

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`locals` block:** Central logic hub. Sanitizes variable inputs and creates the shared `name_prefix` and `common_tags` map.
- **`replace(var.KKE_PROJECT, "[^a-zA-Z0-9-]", "-")`:** Replaces any character that is NOT alphanumeric or a dash with a dash.
- **`lower()`:** Converts the result to lowercase to enforce naming convention.
- **`merge(local.common_tags, {...})`:** Non-destructively adds role-specific tags on top of the shared tag set.
- **Variable validations:** `condition` blocks ensure invalid inputs are rejected at plan-time before any API calls.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # locals block + IAM User + IAM Role
├── variables.tf       # KKE_PROJECT, KKE_TEAM, KKE_ENVIRONMENT (with validations)
├── terraform.tfvars   # xfusion, dev-team, dev
└── outputs.tf         # user name, role name, tags applied
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
variable "KKE_PROJECT" {
  description = "Project name"
  type        = string

  validation {
    condition     = length(var.KKE_PROJECT) > 0
    error_message = "Project name must not be empty."
  }
}

variable "KKE_TEAM" {
  description = "Team name"
  type        = string

  validation {
    condition     = can(regex("^[A-Za-z0-9_-]+$", var.KKE_TEAM))
    error_message = "Team name can only contain letters, numbers, underscores or dashes."
  }
}

variable "KKE_ENVIRONMENT" {
  description = "Environment name"
  type        = string
}
EOF
```

**Validation Notes:**
- `length(var.KKE_PROJECT) > 0`: Rejects empty string input.
- `can(regex(...))`: Returns `true` if the regex matches — effectively a format whitelist. Invalid formats cause a helpful error message at `terraform plan` time.

---

### Step 3: Create Terraform Variables File

```bash
cat > terraform.tfvars << 'EOF'
KKE_PROJECT     = "xfusion"
KKE_TEAM        = "dev-team"
KKE_ENVIRONMENT = "dev"
EOF
```

---

### Step 4: Create Main Configuration File

```bash
cat > main.tf << 'EOF'
locals {
  # Sanitize inputs: lowercase, replace invalid chars with "-"
  project_sanitized = lower(replace(var.KKE_PROJECT, "[^a-zA-Z0-9-]", "-"))
  team_sanitized    = lower(replace(var.KKE_TEAM, "[^a-zA-Z0-9-]", "-"))

  # Shared name prefix
  name_prefix = "${local.project_sanitized}-${local.team_sanitized}"

  # Reusable common tags
  common_tags = {
    Project   = "xfusion"
    Team      = "dev-team"
    ManagedBy = "Terraform"
    Env       = var.KKE_ENVIRONMENT
  }
}

# ---------------------------
# IAM User
# ---------------------------
resource "aws_iam_user" "user" {
  name = "${local.name_prefix}-user"

  tags = local.common_tags
}

# ---------------------------
# IAM Role
# ---------------------------
resource "aws_iam_role" "role" {
  name = "${local.name_prefix}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  # Merge common tags with a role-specific tag
  tags = merge(
    local.common_tags,
    {
      RoleType = "EC2"
    }
  )
}
EOF
```

**Configuration Breakdown:**
- `local.name_prefix`: Computed once and reused across both resources — no repetition.
- `local.common_tags`: A `map(string)` applied directly to the IAM User and used in the `merge()` call for the IAM Role.
- `merge(local.common_tags, { RoleType = "EC2" })`: The second map takes precedence on key conflicts, but here it only adds the new `RoleType` key.

---

### Step 5: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_user_name" {
  value = aws_iam_user.user.name
}

output "kke_role_name" {
  value = aws_iam_role.role.name
}

output "kke_tags_applied" {
  value = aws_iam_user.user.tags
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

kke_role_name = "xfusion-dev-team-role"
kke_tags_applied = tomap({
  "Env"       = "dev"
  "ManagedBy" = "Terraform"
  "Project"   = "xfusion"
  "Team"      = "dev-team"
})
kke_user_name = "xfusion-dev-team-user"
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
aws_iam_role.role
aws_iam_user.user
```

---

### Step 2: Verify IAM User Tags

```bash
terraform state show aws_iam_user.user
```

**Verify:** `tags` map contains `Project`, `Team`, `ManagedBy`, `Env`.

---

### Step 3: Verify IAM Role Tags (includes RoleType)

```bash
terraform state show aws_iam_role.role
```

**Verify:** `tags` map contains all common tags **plus** `RoleType = "EC2"`.

---

## 🔍 Code Analysis

### `locals` vs `variables` — When to Use Which

| Concern | `variable` | `local` |
|---------|-----------|---------|
| Accepts external input | ✅ Yes | ❌ No |
| Can be validated | ✅ Yes | ❌ No |
| Used for derived/computed values | ❌ No | ✅ Yes |
| Promotes DRY code | Partial | ✅ Yes |

In this task, `variables` take the raw user input and `locals` do the heavy lifting of sanitizing and deriving values.

### `merge()` Function
`merge(map1, map2, ...)` combines multiple maps into one. Later maps override earlier ones on key conflicts:
```hcl
merge({A = "1", B = "2"}, {B = "3", C = "4"})
# Result: {A = "1", B = "3", C = "4"}
```

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Derived Name |
|----------|-------------|
| IAM User | `xfusion-dev-team-user` |
| IAM Role | `xfusion-dev-team-role` |

### Task Completion Checklist
- [x] ✅ IAM User provisioned with sanitized, lowercase name.
- [x] ✅ IAM Role provisioned with EC2 trust policy.
- [x] ✅ Both resources tagged with `Project`, `Team`, `ManagedBy`, `Env`.
- [x] ✅ IAM Role has additional `RoleType: EC2` tag via `merge()`.
- [x] ✅ `locals` block in `main.tf` used for all derived values.
- [x] ✅ `variables.tf` includes input validation for both project and team names.
- [x] ✅ All three outputs exported and verified.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
