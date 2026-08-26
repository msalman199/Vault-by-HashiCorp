# 🔐 Vault Integration with CI/CD

<p align="center">
  <img src="https://img.shields.io/badge/HashiCorp%20Vault-1.15.2-FFEC6E?style=for-the-badge&logo=vault&logoColor=black" alt="Vault">
  <img src="https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=for-the-badge&logo=jenkins&logoColor=white" alt="Jenkins">
  <img src="https://img.shields.io/badge/Linux-Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Linux">
  <img src="https://img.shields.io/badge/AppRole-Authentication-623CE4?style=for-the-badge" alt="AppRole">
  <img src="https://img.shields.io/badge/KV-v2-FFEC6E?style=for-the-badge&logo=vault&logoColor=black" alt="KV v2">
  <img src="https://img.shields.io/badge/CI%2FCD-Secrets%20Management-0A66C2?style=for-the-badge" alt="CI/CD">
</p>

<p align="center">
  <b>🚀 Secure Secrets Management with HashiCorp Vault and Jenkins</b>
</p>

<p align="center">
  This hands-on lab demonstrates how to integrate HashiCorp Vault with Jenkins CI/CD using AppRole authentication, least-privilege policies, and runtime secret injection.
</p>

---

## 📚 Table of Contents

- [🎯 Lab Objectives](#-lab-objectives)
- [🧰 Prerequisites](#-prerequisites)
- [🏗️ Architecture](#️-architecture)
- [🛠️ Technology Stack](#️-technology-stack)
- [📁 Lab Deliverables](#-lab-deliverables)
- [🚀 Task 1 - Install and Initialize Vault and Jenkins](#-task-1---install-and-initialize-vault-and-jenkins)
- [🔑 Task 2 - Configure AppRole and Jenkins Pipeline](#-task-2---configure-approle-and-jenkins-pipeline)
- [🩺 Task 3 - Troubleshoot Vault-Jenkins Integration](#-task-3---troubleshoot-vault-jenkins-integration)
- [🔒 Security Best Practices](#-security-best-practices)
- [✅ Expected Outcomes](#-expected-outcomes)
- [🐛 Common Troubleshooting](#-common-troubleshooting)
- [📊 Validation Checklist](#-validation-checklist)
- [🏁 Conclusion](#-conclusion)
- [🚀 Next Steps](#-next-steps)

---

# 🎯 Lab Objectives

By completing this lab, you will learn how to:

- 🔐 Install and configure HashiCorp Vault on Linux.
- ⚙️ Install and configure Jenkins as a CI/CD server.
- 🔑 Configure Vault's **AppRole authentication method**.
- 📜 Create a least-privilege Vault policy for Jenkins.
- 🔗 Integrate Jenkins with Vault using the HashiCorp Vault plugin.
- 🧩 Inject secrets into Jenkins pipelines at build time.
- 🚫 Prevent sensitive values from being hardcoded into source code.
- 🩺 Diagnose common Vault-Jenkins integration failures.
- 📋 Build a reusable troubleshooting script.
- 🛡️ Apply security best practices for CI/CD secret management.

---

# 🧰 Prerequisites

Before starting this lab, you should have:

- Basic Linux command-line knowledge.
- Familiarity with `systemd`.
- Basic understanding of CI/CD pipelines.
- Basic JSON/HCL knowledge.
- Understanding of token-based authentication.
- Basic Git knowledge.
- Basic understanding of environment variables.
- Access to an Ubuntu 20.04/22.04 Linux machine.

> **⚠️ Important:** This lab assumes a bare Linux machine. Required software is installed during the lab.

---

# 🏗️ Architecture

The completed environment follows this general architecture:

```text
                         ┌───────────────────────┐
                         │       Developer       │
                         │        / Git          │
                         └───────────┬───────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │       Jenkins         │
                         │       CI/CD Server    │
                         └───────────┬───────────┘
                                     │
                              AppRole Login
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │    HashiCorp Vault    │
                         │                       │
                         │  AppRole Auth         │
                         │  jenkins-policy       │
                         │  KV v2 Secrets        │
                         └───────────┬───────────┘
                                     │
                              Runtime Secrets
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │   Jenkins Pipeline    │
                         │                       │
                         │ Build → Test → Deploy  │
                         └───────────────────────┘
```

### 🔐 Authentication Flow

```text
Jenkins
   │
   │ Role ID + Secret ID
   ▼
Vault AppRole
   │
   │ Validates credentials
   ▼
jenkins-policy
   │
   │ Read-only access
   ▼
secret/data/ci-cd/*
   │
   │ Secrets
   ▼
Jenkins Environment Variables
```

---

# 🛠️ Technology Stack

| Technology | Purpose |
|---|---|
| 🟡 HashiCorp Vault | Secrets management |
| 🔴 Jenkins | CI/CD automation |
| 🐧 Ubuntu | Lab operating system |
| 🔑 AppRole | Machine authentication |
| 📦 KV v2 | Secret storage |
| 📜 HCL | Vault configuration and policies |
| 🧪 Bash | Automation and troubleshooting |
| 🔎 jq | JSON processing |
| 🌐 curl | API communication |
| 📂 Git | Source-code management |
| ☕ Java | Jenkins runtime |
| ⚙️ systemd | Service management |

---

# 📁 Lab Deliverables

At the end of the lab, you should have:

```text
~/sample-app/
├── Jenkinsfile
└── package.json

~/jenkins-vault-creds.txt

~/troubleshoot-vault-jenkins.sh

~/troubleshoot-output.txt

build-console-output.txt
```

> **🔒 Security Warning:** `jenkins-vault-creds.txt` contains sensitive AppRole information. Never commit this file to Git or upload it to a public repository.

---

# 🚀 Task 1 - Install and Initialize Vault and Jenkins

## 🧩 Step 1.0 - Install Required Packages

Start by updating the system:

```bash
sudo apt update && sudo apt upgrade -y
```

Install the required utilities:

```bash
sudo apt install -y \
  curl \
  wget \
  unzip \
  git \
  jq \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release \
  netcat-openbsd
```

Verify important tools:

```bash
curl --version
wget --version
git --version
jq --version
```

---

## 🟡 Step 1.1 - Install HashiCorp Vault

Create the Vault directory:

```bash
sudo mkdir -p /opt/vault
cd /opt/vault
```

Set the Vault version:

```bash
VAULT_VERSION="1.15.2"
```

Download Vault:

```bash
wget https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip
```

Extract and install:

```bash
sudo unzip -o vault_${VAULT_VERSION}_linux_amd64.zip
sudo mv vault /usr/local/bin/
sudo chmod +x /usr/local/bin/vault
```

Verify:

```bash
vault version
```

Create the Vault service user:

```bash
sudo useradd --system \
  --home /etc/vault.d \
  --shell /bin/false vault || true
```

Create required directories:

```bash
sudo mkdir -p \
  /etc/vault.d \
  /opt/vault/data \
  /opt/vault/logs \
  /opt/vault/audit
```

Set ownership:

```bash
sudo chown -R vault:vault /etc/vault.d /opt/vault
```

---

## ⚙️ Step 1.1.1 - Configure Vault

Create the Vault configuration:

```bash
sudo tee /etc/vault.d/vault.hcl > /dev/null <<EOF
ui = true
disable_mlock = true

storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
EOF
```

Secure the configuration:

```bash
sudo chown vault:vault /etc/vault.d/vault.hcl
sudo chmod 640 /etc/vault.d/vault.hcl
```

> **⚠️ Production Note:** `tls_disable = 1` is acceptable for this local lab but should not be used for a production Vault deployment.

---

## ⚙️ Step 1.1.2 - Create the Vault systemd Service

```bash
sudo tee /etc/systemd/system/vault.service > /dev/null <<EOF
[Unit]
Description=HashiCorp Vault
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl

[Service]
Type=notify
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP \$MAINPID
KillMode=process
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable Vault:

```bash
sudo systemctl enable vault
```

Start Vault:

```bash
sudo systemctl start vault
```

Verify:

```bash
sudo systemctl status vault --no-pager
```

---

# 🔐 Step 1.2 - Initialize and Unseal Vault

Configure the Vault address:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

Persist the setting:

```bash
echo 'export VAULT_ADDR="http://127.0.0.1:8200"' >> ~/.bashrc
```

Initialize Vault:

```bash
cd /opt/vault

vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  > vault-init.txt
```

Display the generated initialization information:

```bash
cat vault-init.txt
```

> **🚨 CRITICAL:** The unseal keys and root token are highly sensitive. Protect `vault-init.txt` and never commit it to Git.

Extract the required values:

```bash
UNSEAL_KEY_1=$(grep 'Unseal Key 1:' vault-init.txt | awk '{print $4}')
UNSEAL_KEY_2=$(grep 'Unseal Key 2:' vault-init.txt | awk '{print $4}')
UNSEAL_KEY_3=$(grep 'Unseal Key 3:' vault-init.txt | awk '{print $4}')
ROOT_TOKEN=$(grep 'Initial Root Token:' vault-init.txt | awk '{print $4}')
```

Unseal Vault:

```bash
vault operator unseal $UNSEAL_KEY_1
vault operator unseal $UNSEAL_KEY_2
vault operator unseal $UNSEAL_KEY_3
```

Configure the root token:

```bash
export VAULT_TOKEN=$ROOT_TOKEN
```

> **🔒 Production Recommendation:** Do not permanently store the root token in `.bashrc`. This is used here only to simplify the lab.

Verify Vault:

```bash
vault status
```

Expected result should indicate:

```text
Initialized: true
Sealed: false
```

---

# 📦 Step 1.2.1 - Enable KV v2

Enable the KV v2 secrets engine:

```bash
vault secrets enable -path=secret kv-v2
```

Store sample CI/CD secrets:

```bash
vault kv put secret/ci-cd/api-keys \
    github_token="ghp_example_github_token_12345" \
    docker_registry_password="docker_password_example" \
    database_password="db_secret_password_123"
```

Store application secrets:

```bash
vault kv put secret/ci-cd/app-config \
    app_secret_key="super_secret_app_key_456" \
    jwt_secret="jwt_signing_secret_abc"
```

Verify:

```bash
vault kv get secret/ci-cd/api-keys
```

---

# 📝 Step 1.2.2 - Enable Vault Audit Logging

Set the audit directory ownership:

```bash
sudo chown vault:vault /opt/vault/audit
```

Enable file-based audit logging:

```bash
vault audit enable file \
  file_path=/opt/vault/audit/vault-audit.log
```

Generate an audit event:

```bash
vault kv get secret/ci-cd/api-keys
```

Inspect the log:

```bash
sudo tail -n 5 /opt/vault/audit/vault-audit.log
```

Audit logging provides valuable evidence when investigating authentication and authorization problems.

---

# 🔴 Step 1.3 - Install Jenkins

Install Java:

```bash
sudo apt install -y openjdk-11-jdk
```

Verify:

```bash
java -version
```

Add the Jenkins repository key:

```bash
curl -fsSL \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

Add the repository:

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Update packages:

```bash
sudo apt update
```

Install Jenkins:

```bash
sudo apt install -y jenkins
```

Start Jenkins:

```bash
sudo systemctl start jenkins
```

Enable Jenkins at boot:

```bash
sudo systemctl enable jenkins
```

Check status:

```bash
sudo systemctl status jenkins --no-pager
```

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open:

```text
http://localhost:8080
```

Complete the Jenkins setup wizard:

1. 🔑 Enter the initial administrator password.
2. 📦 Select **Install suggested plugins**.
3. 👤 Create an administrator account.
4. 🌐 Confirm the Jenkins URL.
5. 🚀 Complete the setup.

---

# 🔌 Step 1.3.1 - Install the HashiCorp Vault Plugin

In Jenkins navigate to:

```text
Manage Jenkins
   ↓
Plugins
   ↓
Available plugins
```

Search for:

```text
HashiCorp Vault Plugin
```

Install the plugin and restart Jenkins when prompted.

---

## ✅ Task 1 Validation

Run:

```bash
vault status
```

Expected:

```text
Initialized: true
Sealed: false
```

Check Jenkins:

```bash
sudo systemctl is-active jenkins
```

Expected:

```text
active
```

### 📸 Evidence

Capture the output of both commands as evidence that the base platform is operational.

---

# 🔑 Task 2 - Configure AppRole and Jenkins Pipeline

## 🧩 Step 2.1 - Enable AppRole Authentication

Enable AppRole:

```bash
vault auth enable approle
```

Verify:

```bash
vault auth list
```

---

# 📜 Step 2.1.1 - Create the Jenkins Policy

Create a least-privilege policy:

```bash
vault policy write jenkins-policy - <<EOF
path "secret/data/ci-cd/*" {
  capabilities = ["read"]
}

path "secret/metadata/ci-cd/*" {
  capabilities = ["list", "read"]
}
EOF
```

Verify:

```bash
vault policy read jenkins-policy
```

The Jenkins policy should only provide the permissions required by the pipeline.

---

# 🔐 Step 2.1.2 - Create the Jenkins AppRole

Create the AppRole:

```bash
vault write auth/approle/role/jenkins \
    token_policies="jenkins-policy" \
    token_ttl=1h \
    token_max_ttl=4h \
    bind_secret_id=true
```

Retrieve the Role ID:

```bash
ROLE_ID=$(vault read \
  -field=role_id \
  auth/approle/role/jenkins/role-id)
```

Generate a Secret ID:

```bash
SECRET_ID=$(vault write \
  -field=secret_id \
  -f auth/approle/role/jenkins/secret-id)
```

Store them for the lab:

```bash
echo "ROLE_ID=$ROLE_ID" | tee ~/jenkins-vault-creds.txt
echo "SECRET_ID=$SECRET_ID" | tee -a ~/jenkins-vault-creds.txt
```

> **🚨 Security:** Never commit `jenkins-vault-creds.txt` to Git.

---

# 🔗 Step 2.1.3 - Configure Jenkins Vault Credentials

In Jenkins:

```text
Manage Jenkins
   ↓
Credentials
   ↓
(global)
   ↓
Add Credentials
```

Configure:

| Setting | Value |
|---|---|
| Kind | Vault App Role Credential |
| Role ID | `$ROLE_ID` |
| Secret ID | `$SECRET_ID` |
| Path | `approle` |
| ID | `vault-approle` |

Save the credential.

Next navigate to:

```text
Manage Jenkins
   ↓
System
```

Find the **Vault Plugin** configuration.

Set:

```text
Vault URL:
http://127.0.0.1:8200
```

Select:

```text
vault-approle
```

as the default Vault credential.

Save the configuration.

---

# 🧪 Step 2.2 - Create the Sample Application

Create the application directory:

```bash
mkdir -p ~/sample-app
cd ~/sample-app
```

Initialize Git:

```bash
git init
```

Create `package.json`:

```bash
cat > package.json <<'EOF'
{
  "name": "vault-cicd-demo",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo 'Running tests...' && exit 0"
  }
}
EOF
```

---

# 📜 Step 2.2.1 - Create the Jenkinsfile

Create the pipeline:

```bash
cat > Jenkinsfile <<'EOF'
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
            }
        }

        stage('Load Secrets from Vault') {
            steps {
                script {
                    def secrets = [
                        [path: 'secret/ci-cd/api-keys', engineVersion: 2, secretValues: [
                            [envVar: 'GITHUB_TOKEN', vaultKey: 'github_token'],
                            [envVar: 'DATABASE_PASSWORD', vaultKey: 'database_password']
                        ]],
                        [path: 'secret/ci-cd/app-config', engineVersion: 2, secretValues: [
                            [envVar: 'JWT_SECRET', vaultKey: 'jwt_secret']
                        ]]
                    ]

                    withVault([
                        configuration: [
                            timeout: 60,
                            vaultCredentialId: 'vault-approle',
                            vaultUrl: 'http://127.0.0.1:8200'
                        ],
                        vaultSecrets: secrets
                    ]) {
                        sh '''
                            echo "Secrets loaded successfully"
                            echo "GitHub token length: ${#GITHUB_TOKEN}"
                            echo "DB password length: ${#DATABASE_PASSWORD}"
                            echo "JWT secret length: ${#JWT_SECRET}"
                        '''
                    }
                }
            }
        }

        stage('Build and Test') {
            steps {
                sh 'echo "Build step placeholder" && npm test'
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded - secrets were never printed in plaintext'
        }

        failure {
            echo 'Pipeline failed - see console output for the failing stage'
        }
    }
}
EOF
```

Review the file:

```bash
cat Jenkinsfile
```

Commit the project:

```bash
git add .
git commit -m "Initial commit with Vault-integrated pipeline"
```

---

# 🚀 Step 2.2.2 - Create the Jenkins Pipeline Job

In Jenkins:

```text
New Item
   ↓
Pipeline
```

Name:

```text
vault-cicd-demo
```

Under **Pipeline**, select:

```text
Pipeline script
```

Paste the contents of `Jenkinsfile`.

Click:

```text
Save
```

Then:

```text
Build Now
```

---

# 📊 Step 2.2.3 - Validate Secret Injection

Open:

```text
Build
   ↓
Console Output
```

You should see:

```text
Secrets loaded successfully
GitHub token length: <non-zero number>
DB password length: <non-zero number>
JWT secret length: <non-zero number>
```

You should **not** see the actual secret values.

Save the console output as:

```text
build-console-output.txt
```

### 🎯 Success Criteria

```text
✅ Vault authentication succeeds
✅ Secrets are injected into environment variables
✅ Pipeline completes successfully
✅ Secret lengths are non-zero
❌ Raw secrets are never printed
```

---

# 🩺 Task 3 - Troubleshoot Vault-Jenkins Integration

Common failures include:

- 🔒 Vault is sealed.
- 🌐 Vault is unreachable.
- 🔴 Jenkins is unavailable.
- 📜 Vault policy is missing.
- 🔑 AppRole is incorrectly configured.
- ⏱️ AppRole credentials have expired.
- 🔗 Jenkins is configured with the wrong Vault URL.
- 📦 Secret path is incorrect.
- 🚫 Policy does not grant required permissions.

---

# 💥 Step 3.1 - Deliberately Break the Integration

Generate and revoke a Secret ID to simulate credential failure:

```bash
vault write -f \
  auth/approle/role/jenkins/secret-id-accessor/destroy \
  secret_id_accessor="$(vault list -format=json \
  auth/approle/role/jenkins/secret-id | jq -r '.[0]')" || true
```

Test a non-existent secret:

```bash
vault kv get secret/ci-cd/does-not-exist \
  || echo "Expected failure: path does not exist"
```

This creates controlled failures for troubleshooting practice.

---

# 🛠️ Step 3.2 - Create the Troubleshooting Script

Create:

```bash
nano ~/troubleshoot-vault-jenkins.sh
```

Add the complete diagnostic script from the lab instructions.

The script performs four major checks:

```text
1. Vault Connectivity
        ↓
2. Jenkins Connectivity
        ↓
3. Vault Policy + AppRole
        ↓
4. End-to-End Secret Retrieval
```

The script should contain:

```bash
#!/bin/bash
```

and implement:

```text
check_vault_connectivity()
check_jenkins_connectivity()
check_vault_policy()
test_secret_retrieval()
```

Make it executable:

```bash
chmod +x ~/troubleshoot-vault-jenkins.sh
```

---

# 🔍 Troubleshooting Script Capabilities

The script validates:

### 1️⃣ Vault Connectivity

Checks:

```text
Vault API availability
Vault sealed/unsealed state
VAULT_ADDR configuration
```

Example success:

```text
[PASS] Vault is reachable and unsealed
```

---

### 2️⃣ Jenkins Connectivity

Checks:

```text
Jenkins HTTP endpoint
Port 8080
Jenkins service status
```

Example:

```text
[PASS] Jenkins is reachable
```

---

### 3️⃣ Vault Policy and AppRole

Checks:

```text
jenkins-policy
secret/data/ci-cd/*
AppRole existence
Policy binding
```

Example:

```text
[PASS] Policy 'jenkins-policy' grants access
[PASS] AppRole 'jenkins' is correctly bound
```

---

### 4️⃣ End-to-End Secret Retrieval

The script:

```text
Generate AppRole credentials
        ↓
Authenticate against Vault
        ↓
Receive Vault client token
        ↓
Read CI/CD secret
        ↓
Validate secret retrieval
```

Example:

```text
[PASS] AppRole login succeeded
[PASS] Successfully retrieved secret
```

---

# ▶️ Step 3.3 - Run the Troubleshooting Script

Run:

```bash
~/troubleshoot-vault-jenkins.sh
```

For evidence:

```bash
~/troubleshoot-vault-jenkins.sh | tee ~/troubleshoot-output.txt
```

Expected healthy output:

```text
=== Vault-Jenkins Integration Troubleshooting ===

1. Checking Vault connectivity...
   [PASS] Vault is reachable and unsealed

2. Checking Jenkins connectivity...
   [PASS] Jenkins is reachable

3. Checking Vault policy for Jenkins...
   [PASS] Policy 'jenkins-policy' grants access
   [PASS] AppRole 'jenkins' is correctly bound

4. Testing end-to-end secret retrieval via AppRole...
   [PASS] AppRole login succeeded
   [PASS] Successfully retrieved secret

=== Summary ===
Checks passed: 4
Checks failed: 0
Result: Integration is healthy.
```

---

# 🧯 Handling Failed Checks

If a check returns:

```text
[FAIL]
```

read the suggested fix printed underneath it.

Typical fixes include:

### 🔒 Vault Sealed

```bash
vault operator unseal <key>
```

Repeat until the threshold is reached.

---

### ⚙️ Vault Service Down

```bash
sudo systemctl status vault
sudo systemctl start vault
```

---

### 🔴 Jenkins Service Down

```bash
sudo systemctl status jenkins
sudo systemctl restart jenkins
```

---

### 🌐 Check Jenkins Port

```bash
sudo ss -ltnp | grep 8080
```

---

### 📜 Check Policy

```bash
vault policy read jenkins-policy
```

---

### 🔑 Check AppRole

```bash
vault read auth/approle/role/jenkins
```

---

### 🧪 Generate a New Secret ID

```bash
vault write -field=secret_id \
  -f auth/approle/role/jenkins/secret-id
```

Update the Jenkins credential if necessary.

---

# 🔒 Security Best Practices

## 🛡️ 1. Never Hardcode Secrets

❌ Avoid:

```groovy
environment {
    API_KEY = "my-secret-key"
}
```

✅ Prefer:

```text
Jenkins → Vault → Runtime Secret Injection
```

---

## 🔐 2. Use Least Privilege

The Jenkins policy should only grant the required permissions:

```hcl
path "secret/data/ci-cd/*" {
  capabilities = ["read"]
}
```

Avoid:

```hcl
capabilities = ["create", "read", "update", "delete", "sudo"]
```

unless those permissions are genuinely required.

---

## ⏱️ 3. Use Short-Lived Credentials

The AppRole configuration uses:

```text
Token TTL:     1 hour
Maximum TTL:   4 hours
```

Short-lived credentials reduce the impact of credential compromise.

---

## 🚫 4. Never Print Secrets

Avoid:

```bash
echo "$GITHUB_TOKEN"
```

Use:

```bash
echo "GitHub token length: ${#GITHUB_TOKEN}"
```

---

## 🧹 5. Protect Secret Files

Sensitive files such as:

```text
vault-init.txt
jenkins-vault-creds.txt
```

must never be committed.

Add them to `.gitignore`:

```bash
cat >> .gitignore <<'EOF'
vault-init.txt
jenkins-vault-creds.txt
*.secret
.env
EOF
```

---

## 🔐 6. Use TLS in Production

This lab uses:

```text
tls_disable = 1
```

for simplicity.

Production environments should use:

```text
HTTPS
TLS certificates
Certificate rotation
Secure client communication
```

---

## 📝 7. Enable Audit Logging

Audit logging provides evidence for:

```text
Authentication attempts
Secret access
Policy decisions
Operational troubleshooting
Security investigations
```

---

## 👤 8. Avoid Using the Root Token

The Vault root token should only be used for initial administration.

For normal operations:

```text
Dedicated policies
AppRole
Short-lived tokens
Least privilege
```

should be used.

---

# 🐛 Common Troubleshooting

| Problem | Possible Cause | Solution |
|---|---|---|
| Vault unreachable | Service stopped | `sudo systemctl restart vault` |
| Vault sealed | Restart/reboot | Run `vault operator unseal` |
| Jenkins unreachable | Jenkins stopped | `sudo systemctl restart jenkins` |
| HTTP 403 | Jenkins authentication/security configuration | Verify Jenkins configuration |
| Policy missing | Policy not created | `vault policy read jenkins-policy` |
| AppRole login fails | Invalid Role ID/Secret ID | Generate a new Secret ID |
| Secret not found | Wrong KV path | Verify `vault kv get` path |
| Permission denied | Policy mismatch | Review `jenkins-policy` |
| Plugin failure | Vault URL incorrect | Verify Vault plugin configuration |
| Empty environment variable | Wrong `vaultKey` | Compare Jenkinsfile with Vault secret |
| Pipeline exposes secret | Secret printed by shell | Remove secret `echo` commands |

---

# 📊 Validation Checklist

## 🏗️ Infrastructure

- [ ] Vault installed.
- [ ] Vault service enabled.
- [ ] Vault service running.
- [ ] Vault initialized.
- [ ] Vault unsealed.
- [ ] Jenkins installed.
- [ ] Jenkins service running.
- [ ] Jenkins UI accessible.

## 🔐 Vault

- [ ] KV v2 enabled.
- [ ] CI/CD secrets created.
- [ ] Audit logging enabled.
- [ ] AppRole authentication enabled.
- [ ] Jenkins policy created.
- [ ] Jenkins AppRole created.
- [ ] AppRole has correct policy.

## 🔗 Jenkins Integration

- [ ] HashiCorp Vault plugin installed.
- [ ] Vault AppRole credential configured.
- [ ] Vault URL configured.
- [ ] Jenkins pipeline created.
- [ ] Pipeline successfully authenticates to Vault.
- [ ] Secrets injected at runtime.
- [ ] Secrets are not exposed in console output.

## 🩺 Troubleshooting

- [ ] Troubleshooting script created.
- [ ] Script is executable.
- [ ] Vault connectivity check passes.
- [ ] Jenkins connectivity check passes.
- [ ] Policy check passes.
- [ ] AppRole check passes.
- [ ] Secret retrieval check passes.
- [ ] `troubleshoot-output.txt` saved.

---

# 📸 Evidence Requirements

Recommended evidence files:

```text
📄 build-console-output.txt
📄 troubleshoot-output.txt
```

### Task 1 Evidence

Capture:

```bash
vault status
sudo systemctl is-active jenkins
```

### Task 2 Evidence

Capture Jenkins Console Output showing:

```text
Secrets loaded successfully
GitHub token length: <number>
DB password length: <number>
JWT secret length: <number>
```

Verify that no plaintext secret is visible.

### Task 3 Evidence

Capture:

```bash
~/troubleshoot-vault-jenkins.sh | tee ~/troubleshoot-output.txt
```

Expected:

```text
Checks passed: 4
Checks failed: 0
Result: Integration is healthy.
```

---

# 🎯 Expected Outcomes

After completing this lab, you should have:

```text
                    ┌─────────────────┐
                    │     Jenkins     │
                    │     CI/CD       │
                    └────────┬────────┘
                             │
                        AppRole Auth
                             │
                             ▼
                    ┌─────────────────┐
                    │ HashiCorp Vault │
                    │                 │
                    │ AppRole          │
                    │ Policy           │
                    │ KV v2            │
                    └────────┬────────┘
                             │
                       Runtime Secrets
                             │
                             ▼
                    ┌─────────────────┐
                    │ Jenkins Build   │
                    │                 │
                    │ Build + Test    │
                    └─────────────────┘
```

The completed environment demonstrates:

- 🔐 Secure secret storage.
- 🔑 Machine-to-machine authentication.
- 📜 Least-privilege authorization.
- 🔄 Runtime secret injection.
- 🛡️ Reduced secret exposure.
- 🩺 Automated troubleshooting.
- 📋 Auditable secret access.

---

# 🧠 Key Concepts Learned

### HashiCorp Vault

A centralized platform for securely storing, accessing, and managing sensitive information.

### KV v2

A versioned key-value secrets engine used to store application credentials and configuration values.

### AppRole

A machine-oriented Vault authentication method designed for workloads such as Jenkins.

### Vault Policy

Defines which Vault paths a token can access and which operations it can perform.

### Jenkins Pipeline

A CI/CD workflow defined as code, commonly through a `Jenkinsfile`.

### Runtime Secret Injection

Secrets are retrieved when the pipeline executes instead of being stored inside source code.

### Audit Logging

Provides visibility into Vault operations for troubleshooting and security analysis.

---

# 🏁 Conclusion

This lab demonstrates a complete **Vault-to-Jenkins CI/CD secrets-management workflow**.

You installed and initialized HashiCorp Vault, configured the KV v2 secrets engine, enabled audit logging, installed Jenkins, and connected Jenkins to Vault using AppRole authentication.

You then created a least-privilege policy that limits Jenkins to the required CI/CD secret paths. The Jenkins pipeline retrieves GitHub, database, and JWT-related secrets dynamically at build time without hardcoding them into the pipeline source code.

The troubleshooting component extends the lab beyond basic integration. The diagnostic script checks Vault availability, Jenkins availability, policy configuration, AppRole bindings, and end-to-end secret retrieval. This provides a reusable method for identifying common failures in a Vault-Jenkins environment.

The central security principle demonstrated throughout the lab is:

```text
❌ Hardcoded Secrets
        ↓
❌ Secrets in Git
        ↓
❌ Secrets in Jenkinsfile
        ↓
        X

        VS

        ✓ HashiCorp Vault
                ↓
        ✓ AppRole Authentication
                ↓
        ✓ Least-Privilege Policy
                ↓
        ✓ Runtime Secret Injection
                ↓
        ✓ Secure CI/CD Pipeline
```

> 🔐 **Key Takeaway:** Secure CI/CD is not simply about storing secrets in a secure location. It is about designing the complete authentication and authorization boundary so that a compromised pipeline has access only to the secrets it actually needs.

---

# 🚀 Next Steps

After completing this lab, consider extending the environment with:

### 🗄️ Dynamic Database Credentials

Use Vault's database secrets engine to generate temporary database credentials automatically.

```text
Jenkins
   ↓
Vault
   ↓
Dynamic DB Credential
   ↓
Database
   ↓
Credential Expires
```

### 🔐 Production TLS

Replace:

```text
tls_disable = 1
```

with a properly configured TLS listener.

### 🌎 Multi-Environment Policies

Create separate policies for:

```text
development
staging
production
```

### 🔄 Secret Rotation

Implement automated secret rotation without modifying application source code.

### 🌿 Multi-Branch Jenkins Pipelines

Integrate Vault into:

```text
feature/*
develop
staging
main
production
```

with environment-specific policies.

### 🤖 Automated Security Validation

Add security checks to the CI/CD pipeline to detect:

```text
Hardcoded credentials
Leaked API keys
Sensitive files
Insecure configurations
```

---

<p align="center">

## 🔐 Secure Secrets. Secure Pipelines. Secure Infrastructure. 🚀

**HashiCorp Vault + Jenkins = Strong CI/CD Secret Management**

</p>
