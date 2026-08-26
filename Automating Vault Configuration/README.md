<div align="center">

# ⚙️ Automating Vault Configuration

### Scripted Initialization, Unsealing, and Policy Management for HashiCorp Vault

![Vault](https://img.shields.io/badge/HashiCorp_Vault-FFEC6E?style=for-the-badge&logo=vault&logoColor=black)
![Bash](https://img.shields.io/badge/Bash_Scripting-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![JSON](https://img.shields.io/badge/JSON-000000?style=for-the-badge&logo=json&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

</div>

---

## 📋 Table of Contents

- [🎯 Learning Objectives](#-learning-objectives)
- [🔧 Prerequisites](#-prerequisites)
- [🖥️ Lab Environment](#️-lab-environment)
- [📘 Task 1: Install and Start Vault Locally](#-task-1-install-and-start-vault-locally)
- [📗 Task 2: Automate Initialization, Unsealing, and Policy Creation](#-task-2-automate-initialization-unsealing-and-policy-creation)
- [📙 Task 3: Build a Dynamic Policy Manager Script](#-task-3-build-a-dynamic-policy-manager-script)
- [🎯 Expected Outcomes](#-expected-outcomes)
- [🛡️ MITRE ATT&CK Mapping](#️-mitre-attck-mapping)
- [📚 Key Concepts](#-key-concepts)
- [🔍 Troubleshooting](#-troubleshooting)
- [✅ Conclusion](#-conclusion)

---

## 🎯 Learning Objectives

By the end of this lab, students will be able to:

| # | Objective |
|---|-----------|
| 1 | Install and configure HashiCorp Vault on a Linux system for local development use |
| 2 | Automate Vault initialization, unsealing, and policy creation using shell scripts |
| 3 | Create and apply Vault ACL policies programmatically instead of through manual CLI edits |
| 4 | Configure a KV version 2 secrets engine and populate it with sample secrets via automation |
| 5 | Build a reusable script that generates, applies, lists, and deletes Vault policies on demand |

---

## 🔧 Prerequisites

| Requirement | Details |
|---|---|
| 🐧 Linux CLI | Basic understanding of Linux command line operations |
| 📝 Text Editors | Familiarity with text editors (nano, vim, or similar) |
| 📄 JSON/HCL | Basic knowledge of JSON and HCL (HashiCorp Configuration Language) syntax |
| 🔐 Security Concepts | Understanding of basic security concepts (authentication, authorization, least privilege) |
| 🖋️ Shell Scripting | Basic shell scripting knowledge (variables, functions, case statements) |

---

## 🖥️ Lab Environment

> **☁️ Single Linux Machine**
> All tasks are performed on one Linux machine (bare metal or VM) with root/sudo access and internet access to download packages. No additional virtual machines or remote hosts are required. The environment is isolated and safe for experimentation. Vault will be run in dev-like local mode (file storage backend, TLS disabled, listening only on `127.0.0.1`) so that the lab focuses on automation rather than production hardening. **This setup is explicitly for learning and must never be used to store real production secrets.**

Install the following before starting:

```bash
# 📦 Install required packages
sudo apt update && sudo apt install -y curl wget unzip jq gnupg lsb-release
```

---

## 📘 Task 1: Install and Start Vault Locally

### 🔹 Step 1.1: Install HashiCorp Vault

```bash
# 🔑 Add the HashiCorp GPG key and repository
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 📦 Install Vault
sudo apt update && sudo apt install -y vault

# ✅ Verify installation
vault version
```

> **Expected result:** `vault version` prints a version string such as `Vault v1.x.x`.

### 🔹 Step 1.2: Create the Working Directory Structure

```bash
# 🗂️ Create the working directory structure
mkdir -p ~/vault-automation-lab/{scripts,configs,configs/policies,data}
cd ~/vault-automation-lab
ls -la
```

> **Expected result:** the directory listing shows `scripts`, `configs`, `configs/policies`, and `data`.

### 🔹 Step 1.3: Create a Vault Configuration File and Start the Server

```hcl
# ⚙️ Create a Vault configuration file
mkdir -p ~/vault-automation-lab/vault-data

cat > ~/vault-automation-lab/configs/vault.hcl << 'EOF'
ui = true
disable_mlock = true

storage "file" {
  path = "/home/USERNAME/vault-automation-lab/vault-data"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}

api_addr = "http://127.0.0.1:8200"
EOF

# 🔄 Replace USERNAME with your actual home directory owner
sed -i "s#/home/USERNAME#$HOME#" ~/vault-automation-lab/configs/vault.hcl
cat ~/vault-automation-lab/configs/vault.hcl
```

Start Vault in the background using this configuration:

```bash
# ▶️ Start Vault in the background
cd ~/vault-automation-lab
nohup vault server -config=configs/vault.hcl > data/vault.log 2>&1 &
sleep 3
export VAULT_ADDR='http://127.0.0.1:8200'
vault status || true
```

> **Expected result:** `vault status` connects successfully and reports `Sealed: true` and `Initialized: false` (Vault is running but not yet initialized). If you get a connection error, wait a few more seconds and retry — the server may still be starting.

> **📦 Deliverable for Task 1:** the Vault process is running locally, `vault status` returns output (not a connection error), and `configs/vault.hcl` exists with the correct storage path substituted.

---

## 📗 Task 2: Automate Initialization, Unsealing, and Policy Creation

### 🔹 Step 2.1: Write an Initialization and Unseal Script

```bash
# 📝 Write the initialization and unseal script
cd ~/vault-automation-lab/scripts

cat > initialize_vault.sh << 'EOF'
#!/bin/bash
set -e

export VAULT_ADDR='http://127.0.0.1:8200'
DATA_DIR="../data"

if vault status 2>/dev/null | grep -q "Initialized.*true"; then
    echo "Vault is already initialized. Skipping init."
else
    echo "Initializing Vault with 5 key shares and a threshold of 3..."
    vault operator init -key-shares=5 -key-threshold=3 -format=json > "$DATA_DIR/vault-init.json"
    echo "Initialization data saved to $DATA_DIR/vault-init.json"
fi

UNSEAL_KEY_1=$(jq -r '.unseal_keys_b64[0]' "$DATA_DIR/vault-init.json")
UNSEAL_KEY_2=$(jq -r '.unseal_keys_b64[1]' "$DATA_DIR/vault-init.json")
UNSEAL_KEY_3=$(jq -r '.unseal_keys_b64[2]' "$DATA_DIR/vault-init.json")
ROOT_TOKEN=$(jq -r '.root_token' "$DATA_DIR/vault-init.json")

echo "Unsealing Vault..."
vault operator unseal "$UNSEAL_KEY_1" > /dev/null
vault operator unseal "$UNSEAL_KEY_2" > /dev/null
vault operator unseal "$UNSEAL_KEY_3" > /dev/null

export VAULT_TOKEN="$ROOT_TOKEN"

echo "Vault status after unseal:"
vault status

cat > "$DATA_DIR/vault-env.sh" << EOL
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='$ROOT_TOKEN'
EOL
chmod 600 "$DATA_DIR/vault-env.sh"

echo "Environment file written to $DATA_DIR/vault-env.sh (contains root token, keep it private)"
EOF

chmod +x initialize_vault.sh
./initialize_vault.sh
```

> **Expected result:** the script prints `Sealed: false` and `Initialized: true` in the final `vault status` output, and `../data/vault-env.sh` is created containing `VAULT_ADDR` and `VAULT_TOKEN`. Keep `vault-init.json` and `vault-env.sh` private — in a real deployment the root token and unseal keys must never be stored in plaintext like this; here it is acceptable only because this is an isolated practice environment.

### 🔹 Step 2.2: Enable the KV Version 2 Secrets Engine and Add Sample Secrets

```bash
# 🔗 Source the Vault environment
cd ~/vault-automation-lab/scripts
source ../data/vault-env.sh

# 🔓 Enable KV v2 at path "secret" if not already enabled
vault secrets enable -path=secret -version=2 kv 2>/dev/null || echo "secret/ already enabled"

# ✅ Confirm it is mounted
vault secrets list

# 🔐 Add sample secrets used later for policy testing
vault kv put secret/dev/app-config database_url="postgresql://dev:devpass@localhost:5432/devdb" debug_mode="true"
vault kv put secret/shared/common-config company_name="Al Nafi Technologies" version="1.0.0"

vault kv get secret/dev/app-config
vault kv get secret/shared/common-config
```

> **Expected result:** `vault secrets list` shows a `secret/` mount with type `kv` and version `2`, and both `vault kv get` commands print the key-value data you just wrote under a `====== Data ======` section.

### 🔹 Step 2.3: Create Static Policies with a Script

```bash
# 📝 Create the base-policies script
cd ~/vault-automation-lab/scripts
source ../data/vault-env.sh

cat > create_base_policies.sh << 'EOF'
#!/bin/bash
set -e
source ../data/vault-env.sh

POLICY_DIR="../configs/policies"
mkdir -p "$POLICY_DIR"

cat > "$POLICY_DIR/developer-policy.hcl" << EOL
path "secret/data/dev/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/metadata/dev/*" {
  capabilities = ["list", "read", "delete"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOL

cat > "$POLICY_DIR/readonly-policy.hcl" << EOL
path "secret/data/shared/*" {
  capabilities = ["read", "list"]
}

path "secret/metadata/shared/*" {
  capabilities = ["read", "list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOL

vault policy write developer-policy "$POLICY_DIR/developer-policy.hcl"
vault policy write readonly-policy "$POLICY_DIR/readonly-policy.hcl"

echo "Policies created:"
vault policy list
EOF

chmod +x create_base_policies.sh
./create_base_policies.sh
```

> **Expected result:** `vault policy list` includes `default`, `developer-policy`, and `readonly-policy`. Verify the contents with `vault policy read developer-policy`, which should print the exact HCL you wrote to the file.

> **📦 Deliverable for Task 2:** Vault is initialized and unsealed, the KV v2 engine is mounted at `secret/`, two sample secrets exist under `secret/dev/` and `secret/shared/`, and two named policies (`developer-policy`, `readonly-policy`) exist and can be read back with `vault policy read`.

---

## 📙 Task 3: Build a Dynamic Policy Manager Script

In this task you will build a single reusable script that generates a policy file from a template, applies it to Vault, lists existing policies, and deletes policies — all driven by command-line arguments instead of one-off manual commands.

### 🔹 Step 3.1: Write the Dynamic Policy Manager Script

```bash
# 📝 Write the dynamic policy manager script
cd ~/vault-automation-lab/scripts

cat > dynamic_policy_manager.sh << 'EOF'
#!/bin/bash
set -e

source ../data/vault-env.sh

POLICY_DIR="../configs/policies"
mkdir -p "$POLICY_DIR"

function generate_policy() {
    local policy_type=$1
    local policy_name=$2
    local resource_path=$3

    if [ -z "$policy_type" ] || [ -z "$policy_name" ] || [ -z "$resource_path" ]; then
        echo "Usage: $0 generate <read-only|read-write|admin> <policy_name> <resource_path>"
        exit 1
    fi

    local policy_file="$POLICY_DIR/${policy_name}.hcl"

    case "$policy_type" in
        "read-only")
            cat > "$policy_file" << EOL
path "$resource_path" {
  capabilities = ["read", "list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOL
            ;;
        "read-write")
            cat > "$policy_file" << EOL
path "$resource_path" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOL
            ;;
        "admin")
            cat > "$policy_file" << EOL
path "$resource_path" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOL
            ;;
        *)
            echo "Unknown policy type: $policy_type"
            echo "Valid types are: read-only, read-write, admin"
            exit 1
            ;;
    esac

    vault policy write "$policy_name" "$policy_file"
    echo "Policy '$policy_name' (type: $policy_type) generated at $policy_file and applied to Vault"
}

function list_policies() {
    echo "Policies currently defined in Vault:"
    vault policy list
    echo
    echo "Local policy files in $POLICY_DIR:"
    ls -la "$POLICY_DIR" 2>/dev/null || echo "No local policy files found"
}

function show_policy() {
    local policy_name=$1
    if [ -z "$policy_name" ]; then
        echo "Usage: $0 show <policy_name>"
        exit 1
    fi
    vault policy read "$policy_name"
}

function delete_policy() {
    local policy_name=$1
    if [ -z "$policy_name" ]; then
        echo "Usage: $0 delete <policy_name>"
        exit 1
    fi
    vault policy delete "$policy_name"
    rm -f "$POLICY_DIR/${policy_name}.hcl"
    echo "Policy '$policy_name' deleted from Vault and removed from $POLICY_DIR"
}

function print_usage() {
    echo "Dynamic Vault Policy Manager"
    echo "Usage: $0 <command> [arguments]"
    echo
    echo "Commands:"
    echo "  generate <read-only|read-write|admin> <policy_name> <resource_path>"
    echo "      Generate a policy file from a template and apply it to Vault"
    echo "  list"
    echo "      List all policies currently defined in Vault and locally"
    echo "  show <policy_name>"
    echo "      Print the contents of a policy stored in Vault"
    echo "  delete <policy_name>"
    echo "      Remove a policy from Vault and delete its local file"
    echo
    echo "Example:"
    echo "  $0 generate read-only myapp-policy secret/data/myapp/*"
}

case "$1" in
    "generate")
        generate_policy "$2" "$3" "$4"
        ;;
    "list")
        list_policies
        ;;
    "show")
        show_policy "$2"
        ;;
    "delete")
        delete_policy "$2"
        ;;
    *)
        print_usage
        ;;
esac
EOF

chmod +x dynamic_policy_manager.sh
```

### 🔹 Step 3.2: Exercise Every Command in the Script

```bash
# 🧪 Exercise every command in the script
cd ~/vault-automation-lab/scripts

# 📖 Show usage with no arguments
./dynamic_policy_manager.sh

# ➕ Generate a read-only policy scoped to a sample application path
./dynamic_policy_manager.sh generate read-only myapp-readonly secret/data/myapp/*

# ➕ Generate a read-write policy for the same application
./dynamic_policy_manager.sh generate read-write myapp-readwrite secret/data/myapp/*

# 📋 List all policies (Vault-side and local files)
./dynamic_policy_manager.sh list

# 👀 Show the contents of the generated read-only policy
./dynamic_policy_manager.sh show myapp-readonly

# 🗑️ Delete the read-write policy to demonstrate cleanup
./dynamic_policy_manager.sh delete myapp-readwrite

# ✅ Confirm it is gone
./dynamic_policy_manager.sh list
```

> **Expected result:** after `generate`, `vault policy list` (via the `list` command) shows `myapp-readonly` and `myapp-readwrite`; `show myapp-readonly` prints HCL with a `path "secret/data/myapp/*"` block granting `read` and `list`; after `delete myapp-readwrite`, the final `list` no longer shows `myapp-readwrite` in either the Vault policy list or the local `configs/policies` directory listing.

### 🔹 Step 3.3: Validate Policy Enforcement with a Scoped Token

```bash
# 🔐 Validate policy enforcement with a scoped token
cd ~/vault-automation-lab/scripts
source ../data/vault-env.sh

# 🔑 Put a sample secret under the path the read-only policy covers
vault kv put secret/myapp/config api_key="sample-key-123" env="lab"

# 🎫 Create a token restricted to the read-only policy
MYAPP_TOKEN=$(vault token create -policy="myapp-readonly" -field=token)

# ✅ This should succeed (read is allowed)
VAULT_TOKEN=$MYAPP_TOKEN vault kv get secret/myapp/config

# ❌ This should fail (write is not allowed by a read-only policy)
VAULT_TOKEN=$MYAPP_TOKEN vault kv put secret/myapp/config api_key="changed" || echo "Write correctly denied by policy"
```

> **Expected result:** the `vault kv get` command succeeds and prints the secret data; the `vault kv put` command fails with a permission denied error, and the `echo` confirms the policy is enforcing least privilege as designed.

> **📦 Deliverable for Task 3:** a working `dynamic_policy_manager.sh` supporting `generate`, `list`, `show`, and `delete` commands, at least one policy generated and verified through its `show` output, and a scoped token demonstrating that the generated policy actually restricts access (read succeeds, write is denied).

---

## 🎯 Expected Outcomes

- A locally running Vault server that has been initialized, unsealed, and configured entirely through scripts rather than manual one-off CLI edits
- A KV v2 secrets engine populated with sample secrets and protected by named ACL policies
- A reusable `dynamic_policy_manager.sh` tool that can generate, list, inspect, and delete Vault policies from templates, with confirmed enforcement of least-privilege access via a scoped token

---

## 🛡️ MITRE ATT&CK Mapping

| Technique ID | Technique Name | Relevance to This Lab |
|---|---|---|
| [T1552](https://attack.mitre.org/techniques/T1552/) | Unsecured Credentials | Automation replaces manual, error-prone credential handling with scripted policy enforcement around Vault's KV store |
| [T1552.001](https://attack.mitre.org/techniques/T1552/001/) | Credentials In Files | `vault-init.json` and `vault-env.sh` hold the root token and unseal keys in plaintext and must be locked down (`chmod 600`) and treated as highly sensitive |
| [T1078](https://attack.mitre.org/techniques/T1078/) | Valid Accounts | Scoped tokens tied to named policies (`myapp-readonly`) restrict what an authenticated identity can actually do, demonstrated by the denied write test |
| [T1555](https://attack.mitre.org/techniques/T1555/) | Credentials from Password Stores | Centralizing dev/shared/app secrets in Vault's KV v2 engine, gated by scripted ACL policies, reduces reliance on scattered static credential stores |

---

## 📚 Key Concepts

| Concept | Description |
|---|---|
| **KV Version 2 Secrets Engine** | Vault's versioned key-value secrets backend, mounted at `secret/` in this lab |
| **ACL Policy (HCL)** | A `path { capabilities = [...] }` document defining exactly which operations an identity may perform on which paths |
| **Least Privilege / Scoped Token** | A Vault token created with `-policy=` that only grants the capabilities defined in that policy, nothing more |
| **Shamir's Secret Sharing** | The scheme behind `-key-shares`/`-key-threshold`, splitting Vault's master key into multiple unseal key shares |
| **Idempotent Script Design** | The `initialize_vault.sh` pattern of checking `Initialized.*true` before running `vault operator init`, so the script is safe to re-run |
| **Root Token** | The initial, highest-privilege Vault authentication token generated at `vault operator init` |
| **Policy Lifecycle Automation** | The `generate` / `list` / `show` / `delete` command pattern that manages a policy's full lifecycle from a single script |
| **Environment Sourcing Pattern** | Writing `VAULT_ADDR`/`VAULT_TOKEN` to a `vault-env.sh` file that later scripts `source` instead of re-authenticating each time |

---

## 🔍 Troubleshooting

<details>
<summary><strong>Issue 1: <code>vault status</code> reports a connection refused error</strong></summary>

**Diagnostic hint:** check that the server process is still running with `ps aux | grep "vault server"` and inspect `~/vault-automation-lab/data/vault.log` for startup errors, such as a port already in use or a malformed `vault.hcl` file.

</details>

<details>
<summary><strong>Issue 2: <code>vault policy write</code> or <code>vault kv put</code> commands fail with a permission or authentication error</strong></summary>

**Diagnostic hint:** confirm `VAULT_TOKEN` is set correctly by running `echo $VAULT_TOKEN` and `vault token lookup`; re-source `../data/vault-env.sh` if the token is empty or you opened a new shell session.

</details>

---

## ✅ Conclusion

In this lab, students installed HashiCorp Vault, brought it up using a scripted configuration file, and fully automated the initialization and unsealing process instead of performing those steps manually. They then enabled a KV version 2 secrets engine, populated it with sample secrets, and authored ACL policies both as static files and through a dynamic, template-driven `dynamic_policy_manager.sh` script capable of generating, listing, inspecting, and deleting policies from the command line. Finally, students validated that the generated policies actually enforce least privilege by issuing a scoped token and confirming that permitted reads succeeded while disallowed writes were denied. Next steps for extending this work include integrating the policy manager with Terraform's Vault provider for full infrastructure-as-code management, switching from file storage to a production-grade storage backend with TLS enabled, and adding audit logging to track every policy change and secret access in the deployment.

---

<div align="center">

![Al Nafi](https://img.shields.io/badge/Al_Nafi-Cybersecurity_Training-blueviolet?style=for-the-badge)

</div>
