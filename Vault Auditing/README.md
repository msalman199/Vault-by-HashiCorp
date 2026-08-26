# 🔐 HashiCorp Vault Auditing

<p align="center">

![Vault](https://img.shields.io/badge/HashiCorp_Vault-1.15.2-FFEC6E?style=for-the-badge\&logo=vault\&logoColor=black)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?style=for-the-badge\&logo=ubuntu\&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=for-the-badge\&logo=gnu-bash\&logoColor=white)
![JSON](https://img.shields.io/badge/JSON-Log_Analysis-000000?style=for-the-badge\&logo=json\&logoColor=white)
![JQ](https://img.shields.io/badge/jq-JSON_Processing-8A2BE2?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-Auditing-red?style=for-the-badge)
![DevSecOps](https://img.shields.io/badge/DevSecOps-Practices-blue?style=for-the-badge)

</p>

<p align="center">
  <b>🔎 Secure • Monitor • Analyze • Protect</b>
</p>

---

## 📌 Overview

This hands-on lab demonstrates how to configure and manage **audit logging in HashiCorp Vault**.

The lab focuses on enabling multiple audit devices, generating audit events, analyzing Vault audit logs, monitoring secret access, identifying security events, implementing log rotation and backups, and generating compliance reports.

Audit logging is an essential security capability because it provides visibility into **who performed an operation, what operation was performed, when it occurred, and whether the operation succeeded or failed**.

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

* 🔐 Understand the importance of Vault audit logging
* ⚙️ Enable and configure Vault audit devices
* 📝 Generate authentication, authorization, and secret-access events
* 🔎 Analyze Vault audit logs
* 👤 Track user activities and system events
* 🚨 Identify denied operations and policy violations
* 📊 Create audit-analysis scripts
* 🔄 Configure audit-log rotation
* 📡 Implement real-time audit monitoring
* 💾 Back up and archive audit logs
* 🛡️ Protect audit logs with secure permissions
* 🔏 Validate audit-log integrity
* 📋 Generate compliance reports

The original lab objectives specifically emphasize audit logging, event generation, log analysis, and audit-management best practices.

---

## 🧰 Technology Stack

| Technology                    | Purpose                            |
| ----------------------------- | ---------------------------------- |
| 🔐 **HashiCorp Vault 1.15.2** | Secrets management and auditing    |
| 🐧 **Linux / Ubuntu**         | Lab operating system               |
| 🐚 **Bash**                   | Automation and analysis scripts    |
| 🔍 **jq**                     | JSON audit-log processing          |
| 📄 **JSON**                   | Vault audit-log format             |
| 🔄 **Logrotate**              | Audit-log rotation                 |
| 📦 **gzip**                   | Audit-log compression              |
| 🔑 **Vault Policies**         | Authorization control              |
| 🛡️ **DevSecOps**             | Security monitoring and compliance |

---

## 📋 Prerequisites

Before starting the lab, you should have:

* 🐧 Basic Linux command-line knowledge
* 🔐 Familiarity with HashiCorp Vault
* 📄 Basic understanding of JSON
* 🔎 Basic log-analysis knowledge
* 🔒 Understanding of Linux file permissions
* 🐚 Basic Bash/text-processing skills

These prerequisites are aligned with the original lab requirements.

---

# 🚀 Lab Environment

The lab uses an **Al Nafi Linux-based cloud machine**. The environment starts without the required tools, so Vault and supporting utilities must be installed during the lab.

---

# 🟢 Task 1 — Install & Configure HashiCorp Vault

## 🔹 Step 1.1 — Install Vault

```bash
sudo apt update

sudo apt install -y wget unzip curl

wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip

unzip vault_1.15.2_linux_amd64.zip

sudo mv vault /usr/local/bin/

vault version
```

✅ **Expected Result:** Vault should display its installed version.

---

## 🔹 Step 1.2 — Create Vault Directories

```bash
sudo mkdir -p /etc/vault
sudo mkdir -p /opt/vault/data
sudo mkdir -p /var/log/vault

sudo chmod 755 /etc/vault
sudo chmod 755 /opt/vault/data
sudo chmod 755 /var/log/vault
```

📁 Important directories:

```text
/etc/vault
/opt/vault/data
/var/log/vault
```

---

## 🔹 Step 1.3 — Create Vault Configuration

Create:

```text
/etc/vault/vault.hcl
```

Configuration:

```hcl
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}

api_addr     = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui           = true
```

⚠️ **Lab Note:** The configuration disables TLS for the local lab environment. Production Vault deployments should use appropriate TLS protection.

---

## 🔹 Step 1.4 — Start Vault

```bash
vault server -config=/etc/vault/vault.hcl &

sleep 5

export VAULT_ADDR='http://127.0.0.1:8200'

vault status
```

---

## 🔹 Step 1.5 — Initialize & Unseal Vault

```bash
vault operator init -key-shares=1 -key-threshold=1 > vault_keys.txt

UNSEAL_KEY=$(grep 'Unseal Key 1:' vault_keys.txt | awk '{print $NF}')

ROOT_TOKEN=$(grep 'Initial Root Token:' vault_keys.txt | awk '{print $NF}')

vault operator unseal $UNSEAL_KEY

vault auth $ROOT_TOKEN

vault status
```

🔐 **Security Warning**

Never expose Vault unseal keys or root tokens in Git repositories, screenshots, README files, or public logs.

---

# 🟡 Task 2 — Enable & Configure Audit Logging

Vault supports multiple audit devices, including:

* 📄 File audit device
* 📡 Syslog audit device
* 🔌 Socket audit device

---

## 🔹 Step 2.1 — Enable File Audit Device

```bash
vault audit enable file file_path=/var/log/vault/audit.log

vault audit list

ls -la /var/log/vault/
```

📍 Audit log:

```text
/var/log/vault/audit.log
```

---

## 🔹 Step 2.2 — Enable Syslog Audit Device

```bash
vault audit enable -path="syslog_audit" syslog facility=LOCAL0 tag=vault

vault audit list
```

You can now inspect the enabled audit devices with:

```bash
vault audit list
```

---

## 🔹 Step 2.3 — Generate Audit Events

Enable the KV secrets engine:

```bash
vault secrets enable -path=secret kv
```

Create a secret:

```bash
vault kv put secret/test-secret \
  username=testuser \
  password=testpass
```

Read the secret:

```bash
vault kv get secret/test-secret
```

List secrets:

```bash
vault kv list secret/
```

Inspect the audit log:

```bash
tail -5 /var/log/vault/audit.log
```

🎯 **Goal:** Every Vault operation should generate corresponding audit activity.

---

# 🔵 Task 3 — Generate Different Audit Events

## 🔹 Step 3.1 — Authentication Events

Enable Userpass:

```bash
vault auth enable userpass
```

Create user `alice`:

```bash
vault write auth/userpass/users/alice \
  password=mypassword \
  policies=default
```

Authenticate:

```bash
vault auth -method=userpass \
  username=alice \
  password=mypassword
```

Store Alice's token:

```bash
ALICE_TOKEN=$(vault auth -method=userpass \
  username=alice \
  password=mypassword \
  -format=json | jq -r '.auth.client_token')
```

Return to root authentication:

```bash
vault auth $ROOT_TOKEN
```

---

## 🔹 Step 3.2 — Policy & Authorization Events

Create a custom policy:

```bash
vault policy write alice-policy - <<EOF
path "secret/alice/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/shared/*" {
  capabilities = ["read", "list"]
}
EOF
```

Assign the policy:

```bash
vault write auth/userpass/users/alice \
  policies=alice-policy
```

Test an authorized operation:

```bash
VAULT_TOKEN=$ALICE_TOKEN \
vault kv put secret/alice/personal key=value
```

Test an unauthorized operation:

```bash
VAULT_TOKEN=$ALICE_TOKEN \
vault kv put secret/admin/config key=value || \
echo "Access denied as expected"
```

🚨 The denied request creates an important audit event that can be analyzed later.

---

## 🔹 Step 3.3 — Secret Engine Events

Enable Database:

```bash
vault secrets enable -path=database database
```

Enable Transit:

```bash
vault secrets enable -path=transit transit
```

Create a Transit encryption key:

```bash
vault write -f transit/keys/my-key
```

Encrypt data:

```bash
vault write transit/encrypt/my-key \
  plaintext=$(base64 <<< "Hello World")
```

Configure a database connection:

```bash
vault write database/config/my-db \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(localhost:3306)/" \
    allowed_roles="my-role" \
    username="root" \
    password="mypassword" || \
    echo "Database not available, but event logged"
```

---

# 🟣 Task 4 — Analyze Vault Audit Logs

Vault audit records are JSON-based, making tools such as `jq` useful for analysis.

## 🔹 Step 4.1 — Inspect Audit Structure

```bash
head -3 /var/log/vault/audit.log | jq '.'
```

Count audit events:

```bash
wc -l /var/log/vault/audit.log
```

---

## 🔹 Step 4.2 — Analyze Authentication Events

Install `jq` if required:

```bash
sudo apt install -y jq
```

Display authentication activity:

```bash
grep '"type":"request"' /var/log/vault/audit.log |
grep '"path":"auth/' |
jq -r '.time + " " + .request.path + " " + .auth.display_name'
```

Successful authentications:

```bash
grep '"type":"response"' /var/log/vault/audit.log |
grep '"path":"auth/' |
grep '"error":""' |
wc -l
```

Failed authentications:

```bash
grep '"type":"response"' /var/log/vault/audit.log |
grep '"path":"auth/' |
grep -v '"error":""' |
wc -l
```

---

## 🔹 Step 4.3 — Monitor Secret Access

Read operations:

```bash
grep '"type":"request"' /var/log/vault/audit.log |
grep '"operation":"read"' |
jq -r '.time + " " + .request.path + " " + (.auth.display_name // "unknown")'
```

Write operations:

```bash
grep '"type":"request"' /var/log/vault/audit.log |
grep '"operation":"create\|update"' |
jq -r '.time + " " + .request.path + " " + (.auth.display_name // "unknown")'
```

Most accessed secrets:

```bash
grep '"type":"request"' /var/log/vault/audit.log |
jq -r '.request.path' |
grep '^secret/' |
sort |
uniq -c |
sort -nr
```

---

## 🔹 Step 4.4 — Identify Security Events

Find denied operations:

```bash
grep '"type":"response"' /var/log/vault/audit.log |
grep '"error"' |
grep -v '"error":""' |
jq -r '.time + " " + .request.path + " " + .error'
```

Find policy violations:

```bash
grep 'permission denied' /var/log/vault/audit.log |
jq -r '.time + " " + .request.path + " " + (.auth.display_name // "unknown")'
```

Analyze operations by user:

```bash
grep '"type":"request"' /var/log/vault/audit.log |
jq -r '(.auth.display_name // "unknown")' |
sort |
uniq -c |
sort -nr
```

---

# 🟠 Task 5 — Create Audit Analysis Automation

Create:

```bash
nano audit_analysis.sh
```

The script should provide:

* 📊 Total events
* 🔐 Authentication summary
* 👤 Top active users
* 🚨 Security alerts
* 🔎 Most accessed secrets

Make executable:

```bash
chmod +x audit_analysis.sh
```

Run:

```bash
./audit_analysis.sh
```

This turns manual log inspection into a repeatable audit-analysis workflow.

---

# 🔄 Task 6 — Advanced Audit Log Management

## 🔹 Step 6.1 — Log Rotation

Create:

```text
/etc/logrotate.d/vault
```

Configuration:

```text
/var/log/vault/audit.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    copytruncate
    postrotate
        pkill -HUP vault || true
    endscript
}
```

Test:

```bash
sudo logrotate -d /etc/logrotate.d/vault
```

### Benefits

* ♻️ Prevents unlimited log growth
* 💾 Compresses old logs
* 📅 Retains historical audit data
* ⚙️ Automates log management

---

# 📡 Step 6.2 — Real-Time Audit Monitoring

Create:

```bash
nano monitor_audit.sh
```

The monitoring script watches:

```text
/var/log/vault/audit.log
```

and identifies events such as:

```text
[ERROR]
[SECRET_ACCESS]
[LOGIN]
```

Make executable:

```bash
chmod +x monitor_audit.sh
```

Run a demonstration:

```bash
timeout 30s ./monitor_audit.sh &
```

Generate events:

```bash
vault kv get secret/test-secret

VAULT_TOKEN=$ALICE_TOKEN \
vault kv get secret/alice/personal

VAULT_TOKEN=$ALICE_TOKEN \
vault kv put secret/admin/test key=value || true
```

---

# 💾 Step 6.3 — Audit Log Backup

Create backup directory:

```bash
sudo mkdir -p /backup/vault/audit
```

The backup process compresses the audit log and keeps the latest seven days of backups.

Example backup location:

```text
/backup/vault/audit/
```

Verify:

```bash
ls -la /backup/vault/audit/
```

---

# 🔐 Task 7 — Audit Log Security & Compliance

## 🔹 Step 7.1 — Secure Audit Log Permissions

Set restrictive permissions:

```bash
sudo chmod 640 /var/log/vault/audit.log
```

Set ownership:

```bash
sudo chown root:vault /var/log/vault/audit.log \
  || sudo chown root:root /var/log/vault/audit.log
```

Verify:

```bash
ls -la /var/log/vault/audit.log
```

---

## 🔹 Step 7.2 — Audit Log Integrity

Generate a SHA-256 checksum:

```bash
sha256sum /var/log/vault/audit.log
```

The provided integrity-check script compares the current checksum against the previous checksum and reports if the audit log has changed unexpectedly.

Run:

```bash
chmod +x check_audit_integrity.sh

./check_audit_integrity.sh
```

Possible results:

```text
Initial checksum created
```

or:

```text
Audit log integrity verified
```

or:

```text
WARNING: Audit log checksum changed!
```

---

# 📋 Task 8 — Compliance Reporting

The lab creates an automated compliance report containing:

* 📊 Total audit events
* 🔐 Authentication events
* 🔑 Secret-access events
* 🚨 Policy violations
* 👤 Active users
* ⚠️ Failed authentication attempts
* 🔎 Most accessed secrets

Run:

```bash
chmod +x compliance_report.sh

./compliance_report.sh
```

Display the generated report:

```bash
cat vault_compliance_report_*.txt
```

---

# 🛠️ Task 9 — Troubleshooting

## 🔍 Check Audit Devices

```bash
vault audit list -detailed
```

## 📄 Check Audit Log

```bash
ls -la /var/log/vault/audit.log
```

## 🧪 Test Audit Logging

```bash
vault kv put secret/audit-test \
  message="audit test $(date)"
```

Inspect the latest event:

```bash
tail -1 /var/log/vault/audit.log | jq '.'
```

---

# 📈 Performance Monitoring

Check audit-log size:

```bash
du -h /var/log/vault/audit.log
```

Count events:

```bash
wc -l /var/log/vault/audit.log
```

Inspect recent timestamps:

```bash
tail -100 /var/log/vault/audit.log |
jq -r '.time'
```

These checks help understand audit-log growth and recent activity.

---

# ✅ Audit Configuration Validation

The lab includes a validation script that checks:

```text
✓ File audit device
✓ Audit log permissions
✓ Log rotation
✓ Backup directory
```

Run:

```bash
chmod +x validate_audit_config.sh

./validate_audit_config.sh
```

Expected validation style:

```text
=== Vault Audit Configuration Validation ===

1. Checking audit devices...
2. Checking audit log permissions...
3. Checking log rotation...
4. Checking backup procedures...

=== Validation Complete ===
```

---


---

# 🔐 Security Best Practices

### 🛡️ 1. Protect Root Credentials

Never commit:

```text
vault_keys.txt
```

or Vault root tokens to Git.

### 🔒 2. Restrict Audit Log Permissions

Use restrictive permissions such as:

```bash
chmod 640 /var/log/vault/audit.log
```

### ♻️ 3. Configure Log Rotation

Prevent audit logs from consuming unlimited disk space.

### 💾 4. Maintain Backups

Keep compressed audit-log backups for investigation and compliance purposes.

### 🔎 5. Monitor Failed Access

Denied operations and authentication failures can indicate security issues.

### 🔏 6. Verify Log Integrity

Use checksums to detect unexpected modifications.

### 📊 7. Automate Analysis

Use Bash and `jq` to make recurring audit reviews easier.

### 🚨 8. Monitor in Real Time

Real-time monitoring helps identify suspicious operations quickly.

---

# 📊 Audit Monitoring Flow

```text
                  ┌───────────────────┐
                  │   Vault Operation │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   Audit Devices   │
                  └─────────┬─────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
      ┌───────────────┐          ┌────────────────┐
      │ audit.log     │          │ Syslog         │
      └───────┬───────┘          └────────────────┘
              │
              ▼
       ┌──────────────┐
       │ jq / Bash    │
       │ Log Analysis │
       └───────┬──────┘
               │
       ┌───────┴────────┐
       ▼                ▼
 ┌───────────┐    ┌──────────────┐
 │ Monitoring│    │ Compliance   │
 │ & Alerts  │    │ Reporting    │
 └───────────┘    └──────────────┘
```

---

# 🧠 What You Learn

After completing this lab, you gain practical experience with:

* 🔐 HashiCorp Vault auditing
* 📝 Audit-device configuration
* 👤 Authentication monitoring
* 🛡️ Policy and authorization auditing
* 🔑 Secret-access tracking
* 📊 JSON log analysis
* 🐚 Bash automation
* 📡 Real-time monitoring
* 🔄 Log rotation
* 💾 Backup and archival
* 🔏 Integrity verification
* 📋 Compliance reporting
* 🚨 Security-event detection

---

# 🏆 Lab Outcome

By completing this lab, you have built an end-to-end Vault auditing workflow:

```text
Install Vault
     ↓
Configure Vault
     ↓
Initialize & Unseal
     ↓
Enable Audit Devices
     ↓
Generate Audit Events
     ↓
Analyze Audit Logs
     ↓
Detect Security Events
     ↓
Automate Analysis
     ↓
Rotate & Backup Logs
     ↓
Monitor in Real Time
     ↓
Verify Integrity
     ↓
Generate Compliance Reports
```

The original lab concludes that these activities support **compliance, security monitoring, forensic analysis, operational insights, and accountability**.

---

# 📌 Why Vault Auditing Matters

Audit logging provides a detailed trail of activity around sensitive secrets and authentication operations.

It helps organizations:

* 🧾 Support compliance requirements
* 🚨 Detect unauthorized access
* 🔎 Investigate security incidents
* 📈 Understand secret-usage patterns
* 👤 Establish user accountability
* 🛡️ Improve security monitoring

Regular review of audit logs, reliable backup procedures, integrity checking, and appropriate audit configuration are important components of maintaining a secure Vault environment.

---

## 👨‍💻 Skills Demonstrated

```text
HashiCorp Vault
Linux Administration
Bash Scripting
JSON / jq
Security Auditing
Secrets Management
Access Control
Log Management
Real-Time Monitoring
Compliance Reporting
DevSecOps
```

---

## ⭐ Final Checklist

* [ ] Install HashiCorp Vault
* [ ] Configure Vault
* [ ] Initialize and unseal Vault
* [ ] Enable file audit logging
* [ ] Enable syslog auditing
* [ ] Generate authentication events
* [ ] Generate authorization events
* [ ] Generate secret-access events
* [ ] Analyze audit logs
* [ ] Detect denied operations
* [ ] Create audit-analysis automation
* [ ] Configure log rotation
* [ ] Configure real-time monitoring
* [ ] Back up audit logs
* [ ] Secure audit-log permissions
* [ ] Verify audit-log integrity
* [ ] Generate compliance reports
* [ ] Validate the complete audit configuration

---

<p align="center">

### 🔐 Secure Secrets. Monitor Access. Protect Infrastructure. 🚀

**HashiCorp Vault Auditing Lab**

</p>
