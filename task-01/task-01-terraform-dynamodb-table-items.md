# 🌟 Task 01 - Provision DynamoDB Table and Insert Items Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is building a simple **To-Do application** backed by **AWS DynamoDB**. The goal is to provision the database table and seed it with initial task records using Terraform, ensuring the application has a pre-configured, reproducible data state on every fresh deployment.

**Requirements:**
- Create a **DynamoDB table** named **`devops-tasks`** with:
  - Primary Key: `taskId` (String type)
  - Billing Mode: `PAY_PER_REQUEST`
- Insert **two task items** directly via Terraform:
  - **Task 1:** `taskId: "1"`, `description: "Learn DynamoDB"`, `status: "completed"`
  - **Task 2:** `taskId: "2"`, `description: "Build To-Do App"`, `status: "in-progress"`
- All resources must be in a single **`main.tf`** file.
- Use **`variables.tf`** to declare:
  - `KKE_TABLE_NAME` — Name of the DynamoDB table
- Use **`terraform.tfvars`** to supply the table name value.
- Use **`outputs.tf`** to export:
  - `kke_dynamodb_table_name` — The name of the created table
- The Terraform working directory is **`/home/bob/terraform`**
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Create a self-contained Terraform configuration that provisions a DynamoDB table *and* seeds it with application data in a single `terraform apply`.

💡 **Note:** The `aws_dynamodb_table_item` resource manages a single DynamoDB item in Terraform state. The `item` attribute uses the native DynamoDB JSON format, where each value must specify its type (e.g., `{"S": "value"}` for String).

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud (us-east-1 region)  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- DynamoDB Table (`devops-tasks`) — NoSQL task store
- DynamoDB Table Item (`task1`) — "Learn DynamoDB" (completed)
- DynamoDB Table Item (`task2`) — "Build To-Do App" (in-progress)

**Working Directory:** `/home/bob/terraform`

**Data Model:**
| taskId | description | status |
|--------|-------------|--------|
| `"1"` | Learn DynamoDB | completed |
| `"2"` | Build To-Do App | in-progress |

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`aws_dynamodb_table`:** Defines the table schema (partition key: `taskId`).
- **`aws_dynamodb_table_item`:** Inserts individual items into the table using DynamoDB's native JSON format. Both items reference the parent table using `table_name` and `hash_key` from the `aws_dynamodb_table` resource.
- **Variable-Driven Design:** Table name is injected via `variables.tf` and `terraform.tfvars`.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # Table + Item insertions
├── variables.tf       # KKE_TABLE_NAME declaration
├── terraform.tfvars   # Table name value ("devops-tasks")
└── outputs.tf         # Table name output
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
variable "KKE_TABLE_NAME" {
  description = "Name of the DynamoDB table"
  type        = string
}
EOF
```

---

### Step 3: Create Terraform Variables File

```bash
cat > terraform.tfvars << 'EOF'
KKE_TABLE_NAME = "devops-tasks"
EOF
```

---

### Step 4: Create Main Configuration File

```bash
cat > main.tf << 'EOF'
# ---------------------------
# DynamoDB Table
# ---------------------------
resource "aws_dynamodb_table" "devops_tasks" {
  name         = var.KKE_TABLE_NAME
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "taskId"

  attribute {
    name = "taskId"
    type = "S"
  }
}

# ---------------------------
# Insert Task 1
# ---------------------------
resource "aws_dynamodb_table_item" "task1" {
  table_name = aws_dynamodb_table.devops_tasks.name
  hash_key   = aws_dynamodb_table.devops_tasks.hash_key

  item = <<ITEM
{
  "taskId":      {"S": "1"},
  "description": {"S": "Learn DynamoDB"},
  "status":      {"S": "completed"}
}
ITEM
}

# ---------------------------
# Insert Task 2
# ---------------------------
resource "aws_dynamodb_table_item" "task2" {
  table_name = aws_dynamodb_table.devops_tasks.name
  hash_key   = aws_dynamodb_table.devops_tasks.hash_key

  item = <<ITEM
{
  "taskId":      {"S": "2"},
  "description": {"S": "Build To-Do App"},
  "status":      {"S": "in-progress"}
}
ITEM
}
EOF
```

**Configuration Breakdown:**
- **`hash_key = "taskId"`:** Defines the partition key for the table.
- **`billing_mode = "PAY_PER_REQUEST"`:** On-demand billing — no capacity planning needed.
- **`aws_dynamodb_table_item`:** Each block inserts one row into the DynamoDB table.
- **DynamoDB JSON Format:** Every attribute requires an explicit type. `"S"` denotes String.
- **Implicit Dependencies:** Both item resources reference `aws_dynamodb_table.devops_tasks.name` and `.hash_key`, so Terraform automatically creates the table first.

---

### Step 5: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_dynamodb_table_name" {
  value = aws_dynamodb_table.devops_tasks.name
}
EOF
```

---

### Step 6: Initialize, Validate, and Apply

```bash
# Initialize Terraform
terraform init

# Validate syntax
terraform validate

# Format code
terraform fmt

# Apply
terraform apply -auto-approve
```

**Expected Success Message:**
```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

kke_dynamodb_table_name = "devops-tasks"
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
```
aws_dynamodb_table.devops_tasks
aws_dynamodb_table_item.task1
aws_dynamodb_table_item.task2
```

---

### Step 2: Verify Task 1 Status in AWS

```bash
aws dynamodb get-item \
  --table-name devops-tasks \
  --key '{"taskId": {"S": "1"}}'
```

**Expected Output:**
```json
{
    "Item": {
        "taskId": {"S": "1"},
        "description": {"S": "Learn DynamoDB"},
        "status": {"S": "completed"}
    }
}
```

---

### Step 3: Verify Task 2 Status in AWS

```bash
aws dynamodb get-item \
  --table-name devops-tasks \
  --key '{"taskId": {"S": "2"}}'
```

**Expected Output confirms:** `"status": {"S": "in-progress"}`.

---

## 🔍 Code Analysis

### DynamoDB JSON Format vs Standard JSON
DynamoDB uses a unique typed JSON format. Each value must be wrapped in a type descriptor.

| DynamoDB Type Code | AWS Type | Example |
|--------------------|----------|---------|
| `S` | String | `{"S": "hello"}` |
| `N` | Number | `{"N": "42"}` |
| `BOOL` | Boolean | `{"BOOL": true}` |
| `L` | List | `{"L": [...]}` |
| `M` | Map | `{"M": {...}}` |

### `aws_dynamodb_table_item` Lifecycle
- Terraform tracks each item as a separate state resource.
- If an item is modified outside of Terraform (e.g., via the AWS console), a subsequent `terraform apply` will **revert** the change to match the state in `main.tf`.

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Value |
|----------|-------|
| DynamoDB Table | `devops-tasks` |
| Task 1 | `taskId: "1"`, status: `completed` |
| Task 2 | `taskId: "2"`, status: `in-progress` |

### Task Completion Checklist
- [x] ✅ DynamoDB table `devops-tasks` with `taskId` (String) primary key.
- [x] ✅ Task 1 seeded: `Learn DynamoDB` — status `completed`.
- [x] ✅ Task 2 seeded: `Build To-Do App` — status `in-progress`.
- [x] ✅ Variable-driven naming via `variables.tf` and `terraform.tfvars`.
- [x] ✅ All logic in `main.tf`.
- [x] ✅ Output `kke_dynamodb_table_name` verified.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
