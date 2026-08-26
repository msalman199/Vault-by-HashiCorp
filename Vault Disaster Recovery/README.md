# 🔐 HashiCorp Vault Disaster Recovery 

<p align="center">
  <img src="https://img.shields.io/badge/HashiCorp%20Vault-1.15.2-000000?style=for-the-badge&logo=vault&logoColor=white" alt="Vault">
  <img src="https://img.shields.io/badge/Linux-Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Linux">
  <img src="https://img.shields.io/badge/Bash-Scripting-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white" alt="Bash">
  <img src="https://img.shields.io/badge/JSON-Data-000000?style=for-the-badge&logo=json&logoColor=white" alt="JSON">
</p>

<p align="center">
  <b>Backup • Restore • Integrity • Automation • Monitoring</b>
</p>

---

## 📖 Overview

This lab demonstrates how to design and implement a **disaster recovery workflow for HashiCorp Vault** on a Linux-based environment.

The lab covers the complete recovery lifecycle:

- Installing and configuring Vault
- Initializing and unsealing Vault
- Creating sample secrets, policies, and authentication methods
- Creating physical backups and Vault snapshots
- Verifying backup integrity with SHA-256 checksums
- Simulating data loss and corruption
- Restoring Vault data and configuration
- Verifying restored secrets, policies, and authentication
- Automating backup and restoration procedures
- Monitoring backup health
- Documenting operational disaster recovery procedures
- Troubleshooting common recovery failures

> **Lab environment:** The source material uses a single Linux machine provided through the Al Nafi training environment. No additional virtual machines or remote hosts are required. fileciteturn0file0L24-L28

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

- 🔹 Understand the importance of disaster recovery planning for HashiCorp Vault
- 🔹 Implement backup strategies for Vault data and configuration
- 🔹 Perform complete Vault data restoration from backups
- 🔹 Verify data consistency and integrity after restoration
- 🔹 Configure automated backup procedures for production environments
- 🔹 Troubleshoot common disaster recovery scenarios fileciteturn0file0L5-L12

---

## 🧰 Technology Stack

| Technology | Purpose |
|---|---|
| 🔐 **HashiCorp Vault 1.15.2** | Secrets management and recovery target |
| 🐧 **Linux / Ubuntu** | Lab operating system |
| 🐚 **Bash** | Backup, restoration, and monitoring automation |
| 📦 **tar / gzip** | Backup archive creation and extraction |
| 🔎 **SHA-256** | Backup integrity verification |
| 🧩 **jq** | JSON/data processing utility |
| 🌐 **Vault CLI / API** | Vault administration and snapshot operations |
| ⏱️ **cron** | Intended automated backup scheduling |

---

## ✅ Prerequisites

Before starting, you should have:

- Basic Linux command-line knowledge
- Familiarity with filesystem navigation and permissions
- Basic understanding of Vault concepts such as secrets, policies, and authentication
- Understanding of JSON and data structures
- Experience with editors such as `nano` or `vim` fileciteturn0file0L14-L21

---

# 🏗️ Lab Architecture

```text
                    ┌──────────────────────────────┐
                    │        HashiCorp Vault       │
                    │        127.0.0.1:8200        │
                    └──────────────┬───────────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                │                  │                  │
                ▼                  ▼                  ▼
        ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
        │ Vault Data   │   │ Configuration│   │ Init Keys    │
        │ /opt/vault/  │   │ vault.hcl    │   │ root token   │
        │ data         │   │              │   │ unseal keys  │
        └──────┬───────┘   └──────────────┘   └──────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ Backup Infrastructure │
        │ /opt/vault/backups/  │
        └──────────┬───────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌─────────────────┐   ┌─────────────────────┐
│ TAR.GZ Backup   │   │ Vault Snapshot       │
│ + SHA-256       │   │ .snap                │
└────────┬────────┘   └──────────┬──────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
             ┌───────────────┐
             │ Disaster      │
             │ Simulation    │
             └───────┬───────┘
                     ▼
             ┌───────────────┐
             │ Restoration   │
             │ + Verification│
             └───────────────┘
```

---

# 🚀 Lab Workflow

## 1️⃣ Install Required Packages

Update the system and install the utilities required by the lab:

```bash
sudo apt update
sudo apt install -y wget unzip curl jq
```

---

## 2️⃣ Install HashiCorp Vault

The lab uses **Vault 1.15.2**:

```bash
wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip

unzip vault_1.15.2_linux_amd64.zip

sudo mv vault /usr/local/bin/

vault version
```

---

## 3️⃣ Create the Vault Directory Structure

```bash
sudo mkdir -p /opt/vault/data
sudo mkdir -p /opt/vault/config
sudo mkdir -p /opt/vault/logs
sudo mkdir -p /opt/vault/backups
```

The lab configures Vault with file storage and a local TCP listener on port `8200`. fileciteturn0file0L61-L88

Example configuration:

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
disable_mlock = true
```

---

## 4️⃣ Start and Initialize Vault

Start Vault:

```bash
vault server -config=/opt/vault/config/vault.hcl \
  > /opt/vault/logs/vault.log 2>&1 &

sleep 5

export VAULT_ADDR='http://127.0.0.1:8200'
```

Initialize Vault using five key shares and a threshold of three:

```bash
vault operator init -key-shares=5 -key-threshold=3 \
  > /opt/vault/init-keys.txt
```

The lab then uses three of the five unseal keys to unseal Vault. fileciteturn0file0L108-L147

---

# 🔑 5️⃣ Create Sample Vault Data

## Enable KV v2

```bash
vault secrets enable -version=2 kv
vault secrets list
```

## Create Sample Secrets

```bash
vault kv put kv/database \
  username="dbadmin" \
  password="secretpass123"

vault kv put kv/api-keys \
  github="ghp_1234567890abcdef" \
  aws="AKIA1234567890ABCDEF"

vault kv put kv/application \
  config='{"debug": true, "port": 8080}' \
  version="1.2.3"

vault kv put kv/prod/database \
  host="prod-db.example.com" \
  port="5432" \
  username="produser"

vault kv put kv/dev/database \
  host="dev-db.example.com" \
  port="5432" \
  username="devuser"
```

The source lab also creates certificate-related sample data and an application policy. fileciteturn0file0L151-L200

---

# 🛡️ 6️⃣ Create Backup Strategy

The backup workflow stores:

- Vault data
- Vault configuration
- Initialization keys
- Backup manifest
- SHA-256 checksums
- Compressed TAR.GZ archive

Backup directory:

```text
/opt/vault/backups/
```

The lab creates:

```text
/opt/vault/backup-vault.sh
```

The script generates timestamped backups such as:

```text
vault_backup_YYYYMMDD_HHMMSS.tar.gz
```

It also creates checksums and a backup manifest for verification. fileciteturn0file0L204-L268

---

# 💾 7️⃣ Run a Backup

Execute:

```bash
/opt/vault/backup-vault.sh
```

Verify:

```bash
ls -la /opt/vault/backups/
```

Find the latest archive:

```bash
LATEST_BACKUP=$(ls -t /opt/vault/backups/*.tar.gz | head -1)

echo "Latest backup: $LATEST_BACKUP"
```

Inspect its contents:

```bash
tar -tzf "$LATEST_BACKUP" | head -20
```

---

# 📸 8️⃣ Create a Vault Snapshot

The lab also demonstrates a Vault snapshot operation:

```bash
vault operator raft snapshot save \
  /opt/vault/backups/vault-snapshot-$(date +%Y%m%d_%H%M%S).snap
```

Verify:

```bash
ls -la /opt/vault/backups/*.snap
```

---

# 🔍 9️⃣ Verify Backup Integrity

Move into the extracted backup directory and verify the recorded checksums:

```bash
sha256sum -c checksums.txt
```

Review the backup manifest:

```bash
cat backup_manifest.txt
```

The source also validates the Vault configuration with:

```bash
vault server -config=config/vault.hcl -test-config
```

These checks help confirm that the backup contents and configuration are usable before relying on them for recovery. fileciteturn0file0L299-L323

---

# 💥 🔟 Simulate a Disaster

Before destroying data, the lab records the current Vault state:

```bash
vault kv list kv/
vault policy list
vault auth list
```

Then the disaster scenario is simulated by stopping Vault and removing its data:

```bash
pkill vault
sleep 3

rm -rf /opt/vault/data/*
rm -f /opt/vault/logs/vault.log
```

This creates a controlled data-loss scenario for testing the recovery process. fileciteturn0file0L327-L360

---

# ♻️ 1️⃣1️⃣ Restore Vault from Backup

Find and extract the latest backup:

```bash
cd /opt/vault/backups

LATEST_BACKUP=$(ls -t vault_backup_*.tar.gz | head -1)

tar -xzf "$LATEST_BACKUP"
```

Restore the Vault data:

```bash
cp -r "${BACKUP_DIR}/data/"* /opt/vault/data/
```

Restore the initialization information:

```bash
cp "${BACKUP_DIR}/init-keys.txt" \
  /opt/vault/init-keys-restored.txt
```

Correct permissions:

```bash
chown -R $USER:$USER /opt/vault/data
```

---

# 🔓 1️⃣2️⃣ Start and Unseal Restored Vault

Start Vault:

```bash
vault server -config=/opt/vault/config/vault.hcl \
  > /opt/vault/logs/vault-restored.log 2>&1 &
```

Check status:

```bash
vault status
```

Use the restored unseal keys:

```bash
vault operator unseal <UNSEAL_KEY_1>
vault operator unseal <UNSEAL_KEY_2>
vault operator unseal <UNSEAL_KEY_3>
```

Verify:

```bash
vault status
```

The restoration workflow then authenticates with the restored root token and checks Vault functionality. fileciteturn0file0L383-L424

---

# ✅ 1️⃣3️⃣ Verify Data Consistency

Create a post-restoration state document:

```bash
vault kv list kv/
vault policy list
vault auth list
```

Compare the pre-disaster and restored states:

```bash
diff \
  /opt/vault/pre-disaster-state.txt \
  /opt/vault/post-restore-state.txt
```

A matching result indicates that the tested state is consistent with the recorded pre-disaster state. fileciteturn0file0L428-L460

---

# 🔐 1️⃣4️⃣ Verify Individual Secrets

Test restored secrets:

```bash
vault kv get kv/database
vault kv get kv/api-keys
vault kv get kv/prod/database
vault kv get kv/dev/database
vault kv get kv/application
```

This confirms that the restored Vault can retrieve the expected secret data. fileciteturn0file0L464-L487

---

# 👤 1️⃣5️⃣ Verify Policies and Authentication

Check the application policy:

```bash
vault policy read app-policy
```

Authenticate with the configured user:

```bash
vault auth -method=userpass \
  username=appuser \
  password=userpass123
```

Test permitted access:

```bash
vault kv get kv/application
```

Test restricted access:

```bash
vault kv get kv/api-keys
```

The lab expects the restricted operation to fail with a permission-denied response. fileciteturn0file0L491-L511

---

# 🤖 1️⃣6️⃣ Automated Restoration

The lab creates:

```text
/opt/vault/restore-vault.sh
```

The restoration script supports:

```bash
/opt/vault/restore-vault.sh latest
```

or a specific backup:

```bash
/opt/vault/restore-vault.sh vault_backup_YYYYMMDD_HHMMSS
```

The automated process:

```text
Select Backup
     ↓
Stop Vault
     ↓
Extract Archive
     ↓
Verify SHA-256 Checksums
     ↓
Restore Data
     ↓
Restore Keys
     ↓
Start Vault
     ↓
Unseal Vault
     ↓
Verify Restoration
```

The source script performs integrity validation before restoring the Vault data. fileciteturn0file0L515-L651

---

# 📋 1️⃣7️⃣ Disaster Recovery Documentation

The lab generates:

```text
/opt/vault/DISASTER_RECOVERY_PROCEDURES.md
```

The documented recovery plan includes:

### Backup

- Automated backup script
- Backup location
- Daily backup frequency
- 30-day retention target

### Restoration

- Vault binary requirement
- Configuration availability
- Backup availability
- Unseal key availability
- Automated and manual restoration procedures

### Verification

- Vault is unsealed
- Secrets engines are available
- Authentication methods work
- Policies are intact
- Sample secrets are accessible
- User authentication works fileciteturn0file0L677-L743

---

# ⏱️ Recovery Objectives

The source lab defines:

| Objective | Target | Maximum |
|---|---:|---:|
| **RTO** | 30 minutes | 2 hours |
| **RPO** | 24 hours | 48 hours |

The documented testing schedule includes:

- 📅 Monthly — Test backup creation
- 📅 Quarterly — Test full restoration
- 📅 Annually — Perform a disaster recovery drill fileciteturn0file0L751-L765

---

# 📡 1️⃣8️⃣ Backup Monitoring

The lab creates:

```text
/opt/vault/monitor-backups.sh
```

The monitoring script checks:

- 📦 Whether the backup directory exists
- 🔢 Number of available backups
- ⏰ Age of the latest backup
- 🔐 Integrity of the latest backup
- ⚠️ Backup extraction failures

The source configuration uses:

```bash
MAX_AGE_HOURS=25
MIN_BACKUPS=3
```

Alerts are emitted when backup health falls outside the configured limits. fileciteturn0file0L771-L848

---

# 🛠️ Troubleshooting

## ❌ Vault Won't Start After Restoration

Check configuration:

```bash
vault server \
  -config=/opt/vault/config/vault.hcl \
  -test-config
```

Check permissions:

```bash
ls -la /opt/vault/data/
chown -R $USER:$USER /opt/vault/data
```

Check port usage:

```bash
netstat -tlnp | grep 8200
```

Review logs:

```bash
tail -f /opt/vault/logs/vault-restored.log
```

---

## ❌ Cannot Unseal Vault

Check the restored keys:

```bash
cat /opt/vault/init-keys-restored.txt
```

Check Vault state:

```bash
vault status
```

Then retry using the appropriate unseal-key combination.

---

## ❌ Secrets Are Not Accessible

Check secrets engines:

```bash
vault secrets list
```

Check Vault data:

```bash
ls -la /opt/vault/data/
```

Check authentication and policies:

```bash
vault token lookup
vault policy list
```

---

## ❌ Backup Integrity Check Fails

Inspect the archive:

```bash
tar -tzf backup_file.tar.gz
```

List available backups:

```bash
ls -la /opt/vault/backups/
```

Perform checksum verification:

```bash
sha256sum backup_file.tar.gz
```

The source also documents Vault snapshot restoration as an alternative recovery path. fileciteturn0file0L918-L935

---

# 📁 Important Files

```text
/opt/vault/
├── data/
├── config/
│   └── vault.hcl
├── logs/
├── backups/
├── init-keys.txt
├── init-keys-restored.txt
├── backup-vault.sh
├── restore-vault.sh
├── monitor-backups.sh
├── DISASTER_RECOVERY_PROCEDURES.md
├── pre-disaster-state.txt
└── post-restore-state.txt
```

---

# 🔄 Complete Disaster Recovery Lifecycle

```text
┌─────────────────────┐
│ Install & Configure │
│       Vault         │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Initialize &        │
│ Unseal Vault        │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Create Secrets &    │
│ Policies            │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Create Backup       │
│ + Snapshot          │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Verify Integrity    │
│ with SHA-256        │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Simulate Disaster   │
│ / Data Loss         │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Restore Vault Data  │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Start & Unseal      │
│ Restored Vault      │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Verify Secrets,     │
│ Policies & Auth     │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Monitor Backups &   │
│ Test Recovery       │
└─────────────────────┘
```

---

# 🎓 Key Learning Outcomes

After completing the lab, you should be able to:

- ✅ Build a repeatable Vault backup process
- ✅ Store Vault data and configuration in recoverable archives
- ✅ Use checksums to validate backup integrity
- ✅ Create and use Vault snapshots
- ✅ Simulate controlled Vault data loss
- ✅ Restore Vault from a backup
- ✅ Unseal and authenticate against the restored Vault
- ✅ Verify secrets, policies, and authentication methods
- ✅ Automate restoration workflows with Bash
- ✅ Monitor backup freshness and integrity
- ✅ Define RTO/RPO targets
- ✅ Document operational disaster recovery procedures

---

# ⚠️ Security Notes

The source lab intentionally uses a local, simplified environment for training. In a real production deployment:

- 🔒 Never store unseal keys or root tokens in unsecured plaintext files.
- 🔒 Protect backup archives because Vault backups may contain highly sensitive information.
- 🔒 Use secure, access-controlled backup storage.
- 🔒 Encrypt backups according to organizational requirements.
- 🔒 Restrict access to recovery credentials.
- 🔒 Test restoration regularly.
- 🔒 Define retention policies appropriate to business and compliance requirements.
- 🔒 Adapt network, TLS, storage, and credential-management practices to the production environment.

The source material specifically emphasizes adapting the procedures to the environment, including secure storage of unseal keys, network configuration, backup retention, and testing requirements. fileciteturn0file0L939-L953

---

# 🏁 Conclusion

This lab provides a complete hands-on introduction to **HashiCorp Vault disaster recovery**.

You create backups, validate them, simulate a failure, restore Vault data, verify consistency, automate recovery, and monitor backup health. The resulting workflow demonstrates how backup and restoration procedures can support business continuity during system failures, corruption, or other disaster scenarios. fileciteturn0file0L939-L949

<p align="center">
  <b>🔐 Backup • ♻️ Restore • ✅ Verify • 📡 Monitor • 🛡️ Recover</b>
</p>

<p align="center">
  <i>HashiCorp Vault Disaster Recovery Lab</i>
</p>
