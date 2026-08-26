# 🔐 Implementing Policies in HashiCorp Vault

![Vault](https://img.shields.io/badge/HashiCorp-Vault-FF0000?style=for-the-badge\&logo=vault\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?style=for-the-badge\&logo=ubuntu\&logoColor=white)
![HCL](https://img.shields.io/badge/HCL-Configuration-844FBA?style=for-the-badge\&logo=hashicorp\&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Secrets-326CE5?style=for-the-badge\&logo=kubernetes\&logoColor=white)
![DevOps](https://img.shields.io/badge/DevOps-Access_Control-0A0A0A?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-RBAC-2E8B57?style=for-the-badge)

> 🛡️ **A practical HashiCorp Vault lab for designing, applying, testing, and managing fine-grained security policies using HCL.**

---

## 📚 Table of Contents

* [🎯 Lab Objectives](#-lab-objectives)
* [🧰 Prerequisites](#-prerequisites)
* [🏗️ Lab Environment](#️-lab-environment)
* [🛠️ Technology Stack](#️-technology-stack)
* [📁 Lab Structure](#-lab-structure)
* [1️⃣ Environment Setup](#1️⃣-environment-setup-and-vault-installation)
* [2️⃣ Understanding Vault Policies](#2️⃣-understanding-vault-policy-structure)
* [3️⃣ Creating Policies](#3️⃣-creating-basic-policies-in-hcl)
* [4️⃣ Applying Policies](#4️⃣-applying-policies-to-control-access)
* [5️⃣ Testing Policies](#5️⃣-testing-policy-effectiveness)
* [6️⃣ Advanced Policies](#6️⃣-advanced-policy-features)
* [7️⃣ Policy Management](#7️⃣-policy-management-and-maintenance)
* [8️⃣ Cleanup](#8️⃣-cleanup-and-policy-removal)
* [🔧 Troubleshooting](#-troubleshooting)
* [✅ Validation](#-validation-checklist)
* [🏁 Conclusion](#-conclusion)

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

* 🔐 Understand HashiCorp Vault policy concepts and structure.
* 📝 Create access policies using **HCL**.
* 🛡️ Control access to secrets and Vault paths.
* 🧪 Test and validate policy effectiveness.
* 🔄 Manage the complete policy lifecycle.
* 👥 Implement role-based access control using Vault policies.
* 🚫 Apply explicit deny rules where appropriate.
* 🧩 Use policy path templating for dynamic access control.
* 📖 Document policies for operational and security teams.

---

## 🧰 Prerequisites

Before starting, you should have:

* 🐧 Basic Linux command-line knowledge.
* 📝 Familiarity with `nano`, `vim`, or another text editor.
* 🔐 Basic understanding of authentication and authorization.
* 📄 Basic knowledge of JSON and configuration files.
* 🏦 Basic HashiCorp Vault knowledge.
* 💻 Access to the Al Nafi lab environment.

---

## 🏗️ Lab Environment

This lab is designed for the **Al Nafi Linux-based cloud environment**.

The machine is provided as a bare-metal Linux system, so required tools are installed during the exercises.

### 🧪 Environment Components

| Component          | Purpose                             |
| ------------------ | ----------------------------------- |
| 🐧 Ubuntu/Linux    | Lab operating system                |
| 🔐 HashiCorp Vault | Secret management and authorization |
| 📜 HCL             | Vault policy definition             |
| 🖥️ Vault CLI      | Vault administration                |
| 🔑 KV v2           | Secret storage                      |
| 🧪 Bash            | Automation and testing              |
| 🔎 jq              | JSON/token processing               |

---

# 🛠️ Technology Stack

### 🔐 Security & Secret Management

![Vault](https://img.shields.io/badge/HashiCorp_Vault-Secret_Management-FF0000?style=flat-square\&logo=vault\&logoColor=white)
![RBAC](https://img.shields.io/badge/RBAC-Authorization-2E8B57?style=flat-square)
![ACL](https://img.shields.io/badge/ACL-Policies-7952B3?style=flat-square)

### 🐧 Operating System

![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?style=flat-square\&logo=ubuntu\&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=flat-square\&logo=gnu-bash\&logoColor=white)

### 📜 Configuration

![HCL](https://img.shields.io/badge/HCL-HashiCorp_Configuration_Language-844FBA?style=flat-square\&logo=hashicorp\&logoColor=white)
![JSON](https://img.shields.io/badge/JSON-Configuration-000000?style=flat-square\&logo=json\&logoColor=white)

### 🚀 DevOps

![DevOps](https://img.shields.io/badge/DevOps-Security_Automation-0A0A0A?style=flat-square)
![CLI](https://img.shields.io/badge/CLI-Administration-555555?style=flat-square)

---


---

# 1️⃣ Environment Setup and Vault Installation

## 🟢 Step 1.1 — Update the System

### 🏷️ Technology Badge

![Linux](https://img.shields.io/badge/Technology-Linux-E95420?style=flat-square\&logo=linux\&logoColor=white)

Update the package repository and install required utilities:

```bash
sudo apt update

sudo apt install -y curl unzip wget gpg software-properties-common jq
```

Verify the tools:

```bash
curl --version
unzip -v
wget --version
jq --version
```

✅ **Expected Result:** Required Linux utilities are available.

---

## 🟢 Step 1.2 — Install HashiCorp Vault

### 🏷️ Technology Badge

![Vault](https://img.shields.io/badge/Technology-HashiCorp_Vault-FF0000?style=flat-square\&logo=vault\&logoColor=white)

Add the HashiCorp repository:

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

sudo apt-add-repository \
"deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

sudo apt update
```

Install Vault:

```bash
sudo apt install -y vault
```

Verify:

```bash
vault version
```

🎉 **Success:** Vault CLI is installed successfully.

---

## 🟢 Step 1.3 — Start Vault Development Server

### 🏷️ Technology Badge

![Vault Dev](https://img.shields.io/badge/Vault-Development_Server-FF0000?style=flat-square\&logo=vault\&logoColor=white)

Start the development server:

```bash
vault server \
  -dev \
  -dev-root-token-id="root-token" \
  -dev-listen-address="0.0.0.0:8200"
```

> ⚠️ Keep this terminal running.

Open another terminal and configure the environment:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN="root-token"
```

Verify connectivity:

```bash
vault status
```

🎯 **Expected Result:** Vault reports that the development server is initialized and running.

---

# 2️⃣ Understanding Vault Policy Structure

## 🟡 Step 2.1 — Examine Default Policies

### 🏷️ Technology Badge

![Vault Policies](https://img.shields.io/badge/Vault-ACL_Policies-FF0000?style=flat-square\&logo=vault\&logoColor=white)

List policies:

```bash
vault policy list
```

Read the default policy:

```bash
vault policy read default
```

Read the root policy:

```bash
vault policy read root
```

### 🔎 Key Concept

Vault policies define **what an authenticated identity can or cannot do**.

Common capabilities include:

| Capability | Purpose                     |
| ---------- | --------------------------- |
| `create`   | Create data                 |
| `read`     | Read data                   |
| `update`   | Modify data                 |
| `delete`   | Delete data                 |
| `list`     | List paths                  |
| `sudo`     | Bypass certain restrictions |
| `deny`     | Explicitly deny access      |

---

## 🟡 Step 2.2 — Create Lab Directory

### 🏷️ Technology Badge

![Linux](https://img.shields.io/badge/Linux-Filesystem-E95420?style=flat-square\&logo=linux\&logoColor=white)

```bash
mkdir -p ~/vault-policies-lab
cd ~/vault-policies-lab

mkdir policies
mkdir test-data
```

Verify:

```bash
tree ~/vault-policies-lab
```

If `tree` is unavailable:

```bash
find ~/vault-policies-lab -maxdepth 2 -type d
```

---

# 3️⃣ Creating Basic Policies in HCL

## 🔵 Step 3.1 — Create Read-Only Policy

### 🏷️ Technology Badge

![HCL](https://img.shields.io/badge/HCL-Vault_Policy-844FBA?style=flat-square\&logo=hashicorp\&logoColor=white)

Create:

```bash
cat > policies/readonly-policy.hcl << 'EOF'
path "secret/data/app/*" {
  capabilities = ["read", "list"]
}

path "secret/metadata" {
  capabilities = ["list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF
```

This policy allows users to read application secrets without modifying them.

---

## 🔵 Step 3.2 — Create Developer Policy

### 🏷️ Technology Badge

![RBAC](https://img.shields.io/badge/RBAC-Developer_Access-2E8B57?style=flat-square)

```bash
cat > policies/developer-policy.hcl << 'EOF'
path "secret/data/dev/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/data/shared/config" {
  capabilities = ["read"]
}

path "secret/metadata/dev/*" {
  capabilities = ["list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/revoke-self" {
  capabilities = ["update"]
}
EOF
```

🎯 **Purpose:** Developers receive full CRUD access to development secrets while remaining restricted from production application secrets.

---

## 🔵 Step 3.3 — Create Admin Policy

### 🏷️ Technology Badge

![Admin](https://img.shields.io/badge/RBAC-Administrator-DC143C?style=flat-square)

```bash
cat > policies/admin-policy.hcl << 'EOF'
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "sys/config/*" {
  capabilities = ["read", "update"]
}

path "sys/audit" {
  capabilities = ["read", "list"]
}

path "sys/health" {
  capabilities = ["read"]
}

path "auth/token/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
```

> ⚠️ Administrative policies should be granted only to trusted Vault administrators.

---

# 4️⃣ Applying Policies to Control Access

## 🟣 Step 4.1 — Load Policies into Vault

### 🏷️ Technology Badge

![Vault CLI](https://img.shields.io/badge/Vault_CLI-Policy_Management-FF0000?style=flat-square\&logo=vault\&logoColor=white)

```bash
vault policy write readonly-policy policies/readonly-policy.hcl

vault policy write developer-policy policies/developer-policy.hcl

vault policy write admin-policy policies/admin-policy.hcl
```

Verify:

```bash
vault policy list
```

🎉 **Success:** The policies are now registered in Vault.

---

## 🟣 Step 4.2 — Create Test Secrets

### 🏷️ Technology Badge

![KV](https://img.shields.io/badge/Vault-KV_v2-FF0000?style=flat-square\&logo=vault\&logoColor=white)

Enable KV v2:

```bash
vault secrets enable -path=secret kv-v2
```

Create application secrets:

```bash
vault kv put secret/app/database \
  username="app_user" \
  password="app_pass123"

vault kv put secret/app/api-key \
  key="api_key_12345" \
  environment="production"
```

Create development secrets:

```bash
vault kv put secret/dev/database \
  username="dev_user" \
  password="dev_pass456"

vault kv put secret/dev/api-key \
  key="dev_api_key_67890" \
  environment="development"
```

Create shared configuration:

```bash
vault kv put secret/shared/config \
  timeout="30s" \
  max_connections="100"
```

Verify:

```bash
vault kv list secret/
```

---

## 🟣 Step 4.3 — Create Policy-Specific Tokens

### 🏷️ Technology Badge

![Tokens](https://img.shields.io/badge/Vault-Tokens-FF0000?style=flat-square\&logo=vault\&logoColor=white)

```bash
READONLY_TOKEN=$(vault token create \
  -policy="readonly-policy" \
  -format=json | jq -r '.auth.client_token')

DEVELOPER_TOKEN=$(vault token create \
  -policy="developer-policy" \
  -format=json | jq -r '.auth.client_token')

ADMIN_TOKEN=$(vault token create \
  -policy="admin-policy" \
  -format=json | jq -r '.auth.client_token')
```

Save them:

```bash
echo "READONLY_TOKEN=$READONLY_TOKEN" > test-data/tokens.env
echo "DEVELOPER_TOKEN=$DEVELOPER_TOKEN" >> test-data/tokens.env
echo "ADMIN_TOKEN=$ADMIN_TOKEN" >> test-data/tokens.env
```

> 🔒 **Security:** Never commit real Vault tokens to GitHub or another public repository.

---

# 5️⃣ Testing Policy Effectiveness

## 🧪 Step 5.1 — Test Read-Only Policy

### 🏷️ Technology Badge

![Testing](https://img.shields.io/badge/Testing-Policy_Validation-FFB000?style=flat-square)

```bash
export VAULT_TOKEN=$READONLY_TOKEN
```

Read an authorized secret:

```bash
vault kv get secret/app/database
```

List authorized secrets:

```bash
vault kv list secret/app
```

Attempt a write:

```bash
vault kv put secret/app/test key="value" \
  || echo "Write operation denied as expected"
```

Attempt unauthorized access:

```bash
vault kv get secret/dev/database \
  || echo "Access denied as expected"
```

### ✅ Expected Behavior

| Operation        | Result    |
| ---------------- | --------- |
| Read app secret  | ✅ Allowed |
| List app secrets | ✅ Allowed |
| Write app secret | ❌ Denied  |
| Read dev secret  | ❌ Denied  |

---

## 🧪 Step 5.2 — Test Developer Policy

```bash
export VAULT_TOKEN=$DEVELOPER_TOKEN
```

Read development data:

```bash
vault kv get secret/dev/database
```

Create development data:

```bash
vault kv put secret/dev/test-key value="test123"
```

Read shared configuration:

```bash
vault kv get secret/shared/config
```

Attempt production write:

```bash
vault kv put secret/app/test key="value" \
  || echo "Write to app path denied as expected"
```

Attempt shared-data deletion:

```bash
vault kv delete secret/shared/config \
  || echo "Delete shared config denied as expected"
```

🎯 **Result:** Developer permissions are limited according to the policy.

---

## 🧪 Step 5.3 — Test Admin Policy

```bash
export VAULT_TOKEN=$ADMIN_TOKEN
```

Test application access:

```bash
vault kv get secret/app/database
```

Test development access:

```bash
vault kv get secret/dev/database
```

Create an administrative test secret:

```bash
vault kv put secret/admin/test admin="true"
```

Read policies:

```bash
vault policy read readonly-policy
vault policy list
```

Inspect a token:

```bash
vault token lookup $READONLY_TOKEN
```

🎉 **Expected Result:** Administrative operations are available according to the admin policy.

---

# 6️⃣ Advanced Policy Features

## 🧩 Step 6.1 — Path Templating

### 🏷️ Technology Badge

![HCL](https://img.shields.io/badge/HCL-Dynamic_Policies-844FBA?style=flat-square\&logo=hashicorp\&logoColor=white)

Create:

```bash
cat > policies/user-specific-policy.hcl << 'EOF'
path "secret/data/users/{{identity.entity.name}}/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/metadata/users/{{identity.entity.name}}" {
  capabilities = ["list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF
```

Load:

```bash
vault policy write user-specific-policy \
  policies/user-specific-policy.hcl
```

💡 **Concept:** Identity-based templating can create dynamic paths based on the authenticated entity.

---

## 🧩 Step 6.2 — Conditional Access Concepts

### 🏷️ Technology Badge

![Security](https://img.shields.io/badge/Security-Conditional_Access-2E8B57?style=flat-square)

Create:

```bash
cat > policies/conditional-policy.hcl << 'EOF'
path "secret/data/time-sensitive/*" {
  capabilities = ["read"]
}

path "secret/data/restricted/*" {
  capabilities = ["read", "list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF
```

Load:

```bash
vault policy write conditional-policy \
  policies/conditional-policy.hcl
```

> ℹ️ Advanced conditional controls may depend on the Vault edition and authentication architecture being used.

---

## 🧩 Step 6.3 — Explicit Deny Rules

### 🏷️ Technology Badge

![Deny](https://img.shields.io/badge/Security-Deny_Rules-DC143C?style=flat-square)

Create:

```bash
cat > policies/deny-example-policy.hcl << 'EOF'
path "secret/data/*" {
  capabilities = ["read", "list"]
}

path "secret/data/sensitive/*" {
  capabilities = ["deny"]
}

path "secret/data/user-data/*" {
  capabilities = ["create", "read", "update", "list"]
}
EOF
```

Load:

```bash
vault policy write deny-example-policy \
  policies/deny-example-policy.hcl
```

### 🚨 Important

A `deny` capability takes precedence over permissions granted by other matching policy rules.

---

# 7️⃣ Policy Management and Maintenance

## 🔄 Step 7.1 — Update an Existing Policy

### 🏷️ Technology Badge

![Lifecycle](https://img.shields.io/badge/Policy-Lifecycle_Management-844FBA?style=flat-square)

Return to root:

```bash
export VAULT_TOKEN="root-token"
```

Create an updated developer policy:

```bash
cat > policies/developer-policy-updated.hcl << 'EOF'
path "secret/data/dev/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/data/staging/*" {
  capabilities = ["read", "list"]
}

path "secret/data/shared/config" {
  capabilities = ["read"]
}

path "secret/metadata/dev/*" {
  capabilities = ["list"]
}

path "secret/metadata/staging/*" {
  capabilities = ["list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/revoke-self" {
  capabilities = ["update"]
}
EOF
```

Update:

```bash
vault policy write developer-policy \
  policies/developer-policy-updated.hcl
```

Verify:

```bash
vault policy read developer-policy
```

---

## 🧪 Step 7.2 — Automate Policy Testing

### 🏷️ Technology Badge

![Bash](https://img.shields.io/badge/Bash-Automated_Testing-4EAA25?style=flat-square\&logo=gnu-bash\&logoColor=white)

Create:

```bash
cat > test-data/policy-test.sh << 'EOF'
#!/bin/bash

echo "=== Vault Policy Testing Script ==="

source test-data/tokens.env

test_operation() {
    local token=$1
    local operation=$2
    local expected=$3

    export VAULT_TOKEN=$token

    echo "Testing: $operation"

    if eval "$operation" >/dev/null 2>&1; then
        if [ "$expected" = "success" ]; then
            echo "✓ PASS: Operation succeeded as expected"
        else
            echo "✗ FAIL: Operation succeeded but should have failed"
        fi
    else
        if [ "$expected" = "fail" ]; then
            echo "✓ PASS: Operation failed as expected"
        else
            echo "✗ FAIL: Operation failed but should have succeeded"
        fi
    fi

    echo ""
}

echo "=== Testing Read-Only Policy ==="

test_operation \
  "$READONLY_TOKEN" \
  "vault kv get secret/app/database" \
  "success"

test_operation \
  "$READONLY_TOKEN" \
  "vault kv put secret/app/test key=value" \
  "fail"

echo "=== Testing Developer Policy ==="

test_operation \
  "$DEVELOPER_TOKEN" \
  "vault kv get secret/dev/database" \
  "success"

test_operation \
  "$DEVELOPER_TOKEN" \
  "vault kv put secret/dev/test key=value" \
  "success"

test_operation \
  "$DEVELOPER_TOKEN" \
  "vault kv get secret/app/database" \
  "fail"

echo "=== Policy Testing Complete ==="
EOF
```

Make executable:

```bash
chmod +x test-data/policy-test.sh
```

Run:

```bash
./test-data/policy-test.sh
```

🎉 **Success:** Policy behavior is automatically validated.

---

# 📖 Step 7.3 — Document Policies

### 🏷️ Technology Badge

![Documentation](https://img.shields.io/badge/Documentation-Policy_Governance-0366D6?style=flat-square\&logo=markdown\&logoColor=white)

Create:

```bash
cat > policies/README.md << 'EOF'
# Vault Policies Documentation

## Policy Overview

This directory contains HashiCorp Vault policies for access control.

### readonly-policy

Provides read-only access to application secrets.

- Path: `secret/data/app/*`
- Capabilities: read, list
- Use Case: Applications requiring read-only production access

### developer-policy

Provides development access.

- Paths: `secret/data/dev/*`
- Capabilities: create, read, update, delete, list
- Use Case: Development teams

### admin-policy

Provides administrative access.

- Paths: `secret/*`, `auth/*`, `sys/policies/*`
- Capabilities: Administrative
- Use Case: Vault administrators

## Best Practices

1. Follow the principle of least privilege.
2. Prefer specific paths over unnecessary wildcards.
3. Review policies regularly.
4. Test policies before production deployment.
5. Keep policies version controlled.
6. Never commit Vault tokens or credentials.
7. Document the business purpose of every policy.
EOF
```

Verify:

```bash
cat policies/README.md
```

---

# 8️⃣ Cleanup and Policy Removal

## 🧹 Step 8.1 — Revoke Test Tokens

### 🏷️ Technology Badge

![Security](https://img.shields.io/badge/Security-Token_Revocation-DC143C?style=flat-square)

Switch to root:

```bash
export VAULT_TOKEN="root-token"
```

Load tokens:

```bash
source test-data/tokens.env
```

Revoke:

```bash
vault token revoke "$READONLY_TOKEN"
vault token revoke "$DEVELOPER_TOKEN"
vault token revoke "$ADMIN_TOKEN"
```

Verify:

```bash
vault token lookup "$READONLY_TOKEN"
```

The revoked token should no longer be usable.

---

## 🧹 Step 8.2 — Remove Test Policies

List policies:

```bash
vault policy list
```

Remove optional test policies:

```bash
vault policy delete user-specific-policy
vault policy delete conditional-policy
vault policy delete deny-example-policy
```

Verify:

```bash
vault policy list
```

> ⚠️ Never remove the built-in `root` or `default` policies as part of normal lab cleanup.

---

# 🔧 Troubleshooting

## ❌ Issue: Policy Not Taking Effect

### 🔍 Solution

Tokens receive policies when they are created. Create a new token after making significant policy changes:

```bash
vault token create -policy="developer-policy"
```

---

## ❌ Issue: Permission Denied

### 🔍 Solution

Check the active token:

```bash
vault token lookup
```

Check its capabilities:

```bash
vault token capabilities secret/data/dev/database
```

Check the policy:

```bash
vault policy read developer-policy
```

---

## ❌ Issue: Policy Syntax Error

Validate and format the policy:

```bash
vault policy fmt policies/developer-policy.hcl
```

Then load it again:

```bash
vault policy write developer-policy \
  policies/developer-policy.hcl
```

---

## ❌ Issue: KV Path Confusion

KV v2 uses different API paths for data and metadata.

For example:

```text
secret/data/app/database
secret/metadata/app/database
```

The Vault CLI abstracts some of this when using:

```bash
vault kv get secret/app/database
```

---

# 🔎 Useful Validation Commands

### Check Vault Status

```bash
vault status
```

### List Policies

```bash
vault policy list
```

### Read a Policy

```bash
vault policy read developer-policy
```

### Check Token

```bash
vault token lookup
```

### Check Token Capabilities

```bash
vault token capabilities secret/data/app/database
```

### List Secrets

```bash
vault kv list secret/
```

### Read Secret

```bash
vault kv get secret/app/database
```

### Format Policy

```bash
vault policy fmt policies/readonly-policy.hcl
```

---

# ✅ Validation Checklist

Use this checklist to confirm successful completion:

* [ ] 🐧 Linux environment prepared.
* [ ] 🔐 Vault installed successfully.
* [ ] 🟢 Vault development server started.
* [ ] 🔑 Vault CLI authenticated.
* [ ] 📋 Default Vault policies inspected.
* [ ] 📁 Lab directory created.
* [ ] 📜 Read-only policy created.
* [ ] 👨‍💻 Developer policy created.
* [ ] 👑 Admin policy created.
* [ ] 📥 Policies loaded into Vault.
* [🔐] KV v2 secrets engine configured.
* [🗝️] Test secrets created.
* [🎫] Policy-specific tokens created.
* [🧪] Read-only policy tested.
* [🧪] Developer policy tested.
* [🧪] Admin policy tested.
* [🧩] Advanced policy concepts explored.
* [🔄] Policy update performed.
* [🤖] Automated policy testing executed.
* [📖] Policy documentation created.
* [🧹] Test tokens revoked.
* [🗑️] Optional test policies removed.

---

# 🧠 Key Concepts Learned

| Concept         | Description                                        |
| --------------- | -------------------------------------------------- |
| 🔐 Vault Policy | Defines authorization rules                        |
| 📜 HCL          | Language used to define Vault policies             |
| 🛡️ ACL         | Access control mechanism used by Vault             |
| 👥 RBAC         | Role-based access control using policies           |
| 🔑 Token        | Authentication credential associated with policies |
| 📂 Path         | Resource location controlled by a policy           |
| 🚫 Deny         | Explicitly prevents access                         |
| 🧩 Templating   | Creates dynamic identity-based paths               |
| 🧪 Testing      | Validates authorization behavior                   |
| 🔄 Lifecycle    | Creation, update, review, and deletion of policies |

---

# 🌍 Real-World Applications

The skills from this lab can be applied to:

### ☁️ Cloud Security

Control access to cloud application credentials and infrastructure secrets.

### 🚀 DevOps

Provide CI/CD pipelines with only the secrets and permissions they require.

### 🏢 Enterprise Security

Separate access between development, staging, production, and administrative teams.

### 🔐 Secret Management

Centralize credentials while preventing unauthorized access.

### 🧑‍💻 Multi-Team Environments

Give developers, operations teams, and administrators different access levels.

### 📋 Compliance

Create auditable and documented access-control structures based on least privilege.

---

# 🏆 Best Practices

### 1. 🔒 Follow Least Privilege

Give identities only the permissions they actually require.

### 2. 🎯 Prefer Specific Paths

Avoid overly broad wildcards unless they are genuinely required.

### 3. 🚫 Minimize `sudo`

Administrative capabilities should be tightly controlled.

### 4. 🔄 Review Policies Regularly

Remove permissions that are no longer required.

### 5. 🧪 Test Before Production

Always validate both allowed and denied operations.

### 6. 📖 Document Everything

Record the purpose, owner, paths, and intended users for every policy.

### 7. 🔐 Protect Tokens

Never store Vault tokens in Git repositories, public documentation, or insecure files.

### 8. 📦 Version Control Policies

Store sanitized HCL policy files in a secure source-control repository.

---

# 🏁 Conclusion

By completing this lab, you have built a practical foundation for **HashiCorp Vault authorization and policy management**.

You successfully:

* 🔐 Installed and configured HashiCorp Vault.
* 📜 Created policies using HCL.
* 👥 Implemented role-based access control.
* 🗝️ Created and protected test secrets.
* 🎫 Associated tokens with different policies.
* 🧪 Tested authorized and unauthorized operations.
* 🧩 Explored advanced policy concepts.
* 🔄 Updated policies throughout their lifecycle.
* 🤖 Automated policy validation.
* 📖 Documented security controls.
* 🧹 Revoked temporary credentials and cleaned up test resources.

> 🚀 **Final Takeaway:** Vault policies form a critical part of a secure secret-management architecture. When combined with least-privilege access, proper testing, documentation, and lifecycle management, they provide a scalable foundation for protecting sensitive infrastructure and application credentials.

---

## 👨‍💻 Lab Skills Demonstrated

`HashiCorp Vault` • `Vault CLI` • `HCL` • `ACL Policies` • `RBAC` • `KV v2` • `Linux` • `Bash` • `Token Management` • `Secret Management` • `Policy Testing` • `DevOps Security`

---

### ⭐ DevOps Security Lab

**HashiCorp Vault | Policy Engineering | Secret Management | RBAC | DevSecOps**

> 🔐 **Secure access. Least privilege. Automated validation. Production-ready thinking.**
