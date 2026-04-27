# 🌟 Task 05 - Provision AWS KMS Key and Manage File Encryption Using Terraform

## 📌 Task Description

The **Nautilus DevOps team** is enhancing their data security posture by implementing encryption rest. The objective is to create an **AWS KMS (Key Management Service) Key** and use it to automate the encryption and decryption of sensitive local files directly within the Terraform lifecycle. This ensures that sensitive information remains encrypted at rest while maintaining a verifiable decryption path for authorized recovery.

**Requirements:**
- Create a symmetric **KMS Key** named **`devops-kms-key`**.
- Encrypt a pre-existing file **`SensitiveData.txt`** (located in `/home/bob/terraform`).
- **Base64 encode** the ciphertext and save it as **`EncryptedData.bin`** in the same directory.
- **Decrypt** the ciphertext and save the result as **`DecryptedData.txt`** to verify it matches the original content.
- All logic must be contained in a single **`main.tf`** file.
- Use **`outputs.tf`** to export:
  - `kke_kms_key_name` — The name of the created KMS key.
- The Terraform working directory is **`/home/bob/terraform`**.
- Verify that `terraform plan` returns **"No changes. Your infrastructure matches the configuration."**

👉 **Your task:** Provision a security-hardened KMS infrastructure and implement a "round-trip" encryption workflow (Encrypt-Save-Decrypt-Verify) using native Terraform resources and data sources.

💡 **Note:** Terraform provides the `aws_kms_ciphertext` resource for encryption and `aws_kms_secrets` data source for decryption. This allows non-infrastructure tasks (like data transformation) to be handled as part of the infrastructure state.

---

## 🔧 Infrastructure Overview

**Target Environment:** AWS Cloud (KMS — regional service)  
**Provider:** AWS (Amazon Web Services)  
**Resources:**
- KMS Symmetric Key (`devops-kms-key`)
- KMS Alias (`alias/devops-kms-key`)
- Encryption Instance (`aws_kms_ciphertext`)
- Local Files:
  - `SensitiveData.txt` (Source)
  - `EncryptedData.bin` (Ciphertext)
  - `DecryptedData.txt` (Verification)

**Working Directory:** `/home/bob/terraform`

**Encryption Workflow:**
```
[SensitiveData.txt] (Plaintext)
        │
        ▼ (data.local_file)
[Terraform State]
        │
        ▼ (aws_kms_ciphertext)
[Encrypted Blob] (KMS Encrypted)
        │
        ├─► [EncryptedData.bin] (Disk)
        │
        ▼ (data.aws_kms_secrets)
[Decrypted Plaintext]
        │
        └─► [DecryptedData.txt] (Disk)
```

---

## 📋 Solution Overview

### 🏗️ Architecture Components
- **`aws_kms_key`:** Creates the master key. By default, it's a symmetric key suitable for `ENCRYPT_DECRYPT` operations.
- **`aws_kms_alias`:** Assigns a human-readable name (`alias/devops-kms-key`) to the cryptic UUID of the KMS key.
- **`data.local_file`:** Reads the existing sensitive file into Terraform memory.
- **`aws_kms_ciphertext`:** Sends the plaintext to AWS KMS for encryption. It stores the resulting ciphertext in its `ciphertext_blob` attribute.
- **`local_file` (Encrypted):** Writes the binary/base64 ciphertext to a file on the local runner.
- **`data.aws_kms_secrets`:** The decryption complement to `aws_kms_ciphertext`. It decrypts the payload back to plaintext.
- **`local_file` (Decrypted):** Writes the recovered data to disk for final verification.

### 📁 File Structure
```bash
/home/bob/terraform/
├── main.tf            # KMS Key, Alias, Encrypt/Decrypt logic
├── outputs.tf         # Export kke_kms_key_name
└── SensitiveData.txt  # Input file (pre-existing)
```

---

## 🚀 Implementation Steps

### Step 1: Navigate to Working Directory

```bash
cd /home/bob/terraform
```

---

### Step 2: Ensure Input File Exists

Ensure the sensitive data is present:

```bash
echo "This is highly sensitive DevOps data." > /home/bob/terraform/SensitiveData.txt
```

---

### Step 3: Create Main Configuration File

Create `main.tf` with the KMS resource and the encryption/decryption pipeline:

```bash
cat > main.tf << 'EOF'
# ---------------------------
# KMS Key and Alias
# ---------------------------
resource "aws_kms_key" "devops_kms_key" {
  description             = "Symmetric KMS key for DevOps data encryption"
  key_usage               = "ENCRYPT_DECRYPT"
  customer_master_key_spec = "SYMMETRIC_DEFAULT"
  deletion_window_in_days = 30

  tags = {
    Name = "devops-kms-key"
  }
}

resource "aws_kms_alias" "devops_kms_alias" {
  name          = "alias/devops-kms-key"
  target_key_id = aws_kms_key.devops_kms_key.key_id
}

# ---------------------------
# Encryption Pipeline
# ---------------------------

# 1. Read the sensitive source file
data "local_file" "sensitive_data" {
  filename = "/home/bob/terraform/SensitiveData.txt"
}

# 2. Encrypt the content using KMS
resource "aws_kms_ciphertext" "encrypted_data" {
  key_id    = aws_kms_key.devops_kms_key.key_id
  plaintext = data.local_file.sensitive_data.content
}

# 3. Save ciphertext to local disk
resource "local_file" "encrypted_file" {
  content  = aws_kms_ciphertext.encrypted_data.ciphertext_blob
  filename = "/home/bob/terraform/EncryptedData.bin"
}

# ---------------------------
# Decryption Verification
# ---------------------------

# 4. Decrypt the blob back to plaintext
data "aws_kms_secrets" "decrypted_data" {
  secret {
    name    = "decrypted"
    payload = aws_kms_ciphertext.encrypted_data.ciphertext_blob
  }
}

# 5. Save the recovered plaintext for verification
resource "local_file" "decrypted_file" {
  content  = data.aws_kms_secrets.decrypted_data.plaintext["decrypted"]
  filename = "/home/bob/terraform/DecryptedData.txt"
}
EOF
```

---

### Step 4: Create Outputs Configuration File

```bash
cat > outputs.tf << 'EOF'
output "kke_kms_key_name" {
  value = "devops-kms-key"
}
EOF
```

---

### Step 5: Initialize, Validate, and Apply

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

kke_kms_key_name = "devops-kms-key"
```

---

### Step 6: Verify Files Created

```bash
ls -l /home/bob/terraform/
```

**Expected Output:**
- `SensitiveData.txt`
- `EncryptedData.bin` (Contains non-readable ciphertext)
- `DecryptedData.txt` (Should be identical to the original)

---

### Step 7: Final Verification (Critical)

```bash
# Compare files
diff /home/bob/terraform/SensitiveData.txt /home/bob/terraform/DecryptedData.txt

# Run plan to ensure state matches config
terraform plan
```

**Expected Output:**
```
No changes. Your infrastructure matches the configuration.
```

✅ **Task complete.**

---

## 🔍 Code Analysis

### Symmetry and Usage
By setting `customer_master_key_spec = "SYMMETRIC_DEFAULT"`, we create a single key used for both encryption and decryption. This is the standard for general-purpose data protection because it's significantly faster and more cost-effective than asymmetric (Public/Private) key pairs.

### `aws_kms_ciphertext` Resource
This resource acts as an "active" participant in the configuration. It doesn't just represent infrastructure; it performs a cryptographic operation.
- **Input:** `plaintext` from the local file.
- **Output:** `ciphertext_blob` containing the versioned, encrypted data.

### `aws_kms_secrets` Data Source
This data source is used to fetch decrypted values. It takes an encrypted `payload` and returns a `plaintext` map. It's particularly useful when you need to use secrets as inputs to other resources (like RDS passwords or environment variables).

---

## 🎯 Task Completion Summary

### Resources Created

| Resource | Value |
|----------|-------|
| KMS Key | `devops-kms-key` |
| Alias | `alias/devops-kms-key` |
| Encrypted File| `EncryptedData.bin` |
| Decrypted File| `DecryptedData.txt` |

### Task Completion Checklist
- [x] ✅ Symmetric KMS Key `devops-kms-key` provisioned.
- [x] ✅ Encrypted `SensitiveData.txt` using the KMS key.
- [x] ✅ Saved ciphertext as `EncryptedData.bin` (Base64 encoded blob).
- [x] ✅ Successfully decrypted back to `DecryptedData.txt`.
- [x] ✅ Verified with `diff` that recovered data matches original.
- [x] ✅ Output `kke_kms_key_name` verified.
- [x] ✅ Final `terraform plan` returns **"No changes"**.
