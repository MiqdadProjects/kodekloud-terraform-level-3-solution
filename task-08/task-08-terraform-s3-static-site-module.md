# 🌟 Task 08 - Create an S3 Static Website Module Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is building an internal information portal for public access. The portal will be hosted as a static website on AWS S3. To ensure reusability and maintainability, the infrastructure must be encapsulated within a **Terraform module**. This module will handle bucket creation, website configuration, and public access policies.

**Requirements:**
- Create a Terraform module named `s3-static-site` in `/home/bob/terraform/modules/s3-static-site/`.
- Inside the module:
  - Provision an **S3 Bucket** named `nautilus-web-11545` (value passed via variable).
  - Configure the bucket for **Static Website Hosting**, using an `index_document` (value passed via variable, default `index.html`).
  - Attach an **S3 Bucket Policy** allowing public read access (`s3:GetObject`).
  - Tag the bucket with `Project = "StaticWeb"`.
  - Use `variables.tf` to define `bucket_name` and `index_document`.
  - Upload the `index.html` file using `aws_s3_object`.
  - Use `outputs.tf` to export `website_url` formatted for LocalStack: `http://aws:4566/<bucket_name>/<index_document>`.
- In the **root module** (`/home/bob/terraform/main.tf`):
  - Call the `s3-static-site` module, passing the required variables.
  - Export the `website_url` from the module as a root output.
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Build a reusable Terraform module for S3 static website hosting, and implement a root configuration that instantiates it, ensuring a clean separation of concerns.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud / LocalStack  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- S3 Bucket (`nautilus-web-11545`)
- S3 Bucket Website Configuration
- S3 Bucket Policy (Public Read)
- S3 Object (`index.html`)

**Working Directory:** `/home/bob/terraform`

**Module Structure:**
```
/home/bob/terraform/
├── main.tf                     # Root configuration (calls module)
├── index.html                  # Static website content
└── modules/
    └── s3-static-site/         # Reusable S3 Website Module
        ├── main.tf             # Module resources (Bucket, Policy, Object)
        ├── variables.tf        # Module inputs
        └── outputs.tf          # Module outputs
```

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **Terraform Module (`s3-static-site`):** Encapsulates all resources needed for a static site.
- **`aws_s3_bucket`:** The core storage for the website.
- **`aws_s3_bucket_website_configuration`:** Enables the static website hosting feature on the bucket.
- **`aws_s3_bucket_policy`:** Grants `s3:GetObject` permission to `*` (everyone), making the site public.
- **`aws_s3_object`:** Uploads the local `index.html` file to the S3 bucket during provisioning. `depends_on` ensures the bucket exists first.
- **Root `main.tf`:** Instantiates the module, passing the specific bucket name and index document for this deployment.

---

## 🚀 Implementation Steps

### Step 1: Navigate to Working Directory

```bash
cd /home/bob/terraform
```

---

### Step 2: Create Module Variables File

```bash
mkdir -p modules/s3-static-site
cat > modules/s3-static-site/variables.tf << 'EOF'
variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
}

variable "index_document" {
  description = "Index document for static website hosting"
  type        = string
  default     = "index.html"
}
EOF
```

---

### Step 3: Create Module Main Configuration

```bash
cat > modules/s3-static-site/main.tf << 'EOF'
resource "aws_s3_bucket" "static_site" {
  bucket = var.bucket_name

  tags = {
    Project = "StaticWeb"
  }
}

resource "aws_s3_bucket_website_configuration" "static_site" {
  bucket = aws_s3_bucket.static_site.id

  index_document {
    suffix = var.index_document
  }
}

resource "aws_s3_bucket_policy" "public_read" {
  bucket = aws_s3_bucket.static_site.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.static_site.arn}/*"
      }
    ]
  })
}

# Upload index.html from root using variable
resource "aws_s3_object" "index_html" {
  bucket       = aws_s3_bucket.static_site.id
  key          = var.index_document
  source       = "${path.root}/${var.index_document}"
  content_type = "text/html"

  depends_on = [aws_s3_bucket.static_site]
}
EOF
```

---

### Step 4: Create Module Outputs File

```bash
cat > modules/s3-static-site/outputs.tf << 'EOF'
output "website_url" {
  description = "S3 static website URL"
  value       = "http://aws:4566/${var.bucket_name}/${var.index_document}"
}
EOF
```

---

### Step 5: Create Root Configuration File

```bash
cat > main.tf << 'EOF'
module "s3_static_site" {
  source = "./modules/s3-static-site"

  bucket_name    = "nautilus-web-11545"
  index_document = "index.html"
}

output "website_url" {
  value = module.s3_static_site.website_url
}
EOF
```

---

### Step 6: Create Dummy Index.html

```bash
echo "<h1>Welcome to the Nautilus Internal Portal</h1>" > index.html
```

---

### Step 7: Initialize, Validate, and Apply

```bash
terraform init
terraform validate
terraform fmt -recursive
terraform apply -auto-approve
```

**Expected Success Message:**
```
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

website_url = "http://aws:4566/nautilus-web-11545/index.html"
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

✅ **Task complete.**

---

## 🔍 Code Analysis

### Reusability with Modules
By encapsulating the S3 bucket, policy, website configuration, and object upload into a module, we create a reusable component. If we need to spin up another static site tomorrow (e.g., `nautilus-docs`), we simply add another `module` block in the root `main.tf` with a different `bucket_name`, without duplicating all the underlying resource logic.

### `path.root`
In the module's `aws_s3_object`, `source = "${path.root}/${var.index_document}"` is used. `path.root` dynamically resolves to the directory of the root module (where `terraform apply` is run). This ensures the module looks for `index.html` in the root workspace, not inside the module's own directory.

---

## 🎯 Task Completion Summary

### Resources Created

| Component | Resource | Details |
|-----------|----------|---------|
| **Module** | `s3-static-site` | Reusable Terraform module |
| **Bucket** | `nautilus-web-11545` | Configured for static hosting |
| **Policy** | Bucket Policy | Public Read (`s3:GetObject`) |
| **Object** | `index.html` | Uploaded via `aws_s3_object` |

### Task Completion Checklist
- [x] ✅ Module directory `modules/s3-static-site/` structure created.
- [x] ✅ Module `variables.tf` defined for `bucket_name` and `index_document`.
- [x] ✅ Module `main.tf` provisions bucket, website config, policy, and uploads object.
- [x] ✅ Bucket tagged with `Project = "StaticWeb"`.
- [x] ✅ Module `outputs.tf` exports the LocalStack-formatted `website_url`.
- [x] ✅ Root `main.tf` successfully calls the module and passes inputs.
- [x] ✅ Root `main.tf` exposes the module's output.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
