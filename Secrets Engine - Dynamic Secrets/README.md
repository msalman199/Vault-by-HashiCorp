# 🔐 HashiCorp Vault Dynamic Secrets 

![Vault](https://img.shields.io/badge/HashiCorp%20Vault-Dynamic%20Secrets-000000?style=for-the-badge&logo=vault&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-RDS%20Concept-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Environment-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![CLI](https://img.shields.io/badge/CLI-Automation-4EAA25?style=for-the-badge&logo=gnu-bash&logoColor=white)

> 🛡️ **Hands-on Security Lab:** Generate short-lived database credentials with HashiCorp Vault, test access controls, manage leases, revoke credentials, and audit secret operations.

---

## 📚 Overview

This lab demonstrates how **HashiCorp Vault Dynamic Secrets** can replace long-lived static database credentials with temporary credentials generated on demand.

The lab uses a PostgreSQL Docker container to simulate an AWS RDS-style database environment. Vault is configured with the **Database Secrets Engine** to create database users dynamically with different permission levels.

You will also explore the complete secret lifecycle:

**Create → Use → Renew → Revoke → Audit**

The lab is designed around practical Vault administration, database access control, lease management, and security monitoring. fileciteturn0file0L5-L11

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

- 🔐 Understand dynamic secrets and their advantages over static credentials
- 🗄️ Configure Vault's database secrets engine
- 🐘 Deploy and configure PostgreSQL for testing
- 🔗 Create a database connection in Vault
- 👤 Create read-only and read-write dynamic database roles
- 🎫 Generate temporary database credentials
- ⏱️ Configure credential TTLs and expiration
- 🔄 Renew active secret leases
- ❌ Revoke dynamic credentials
- 🧹 Perform bulk lease revocation
- 📋 Enable Vault audit logging
- 🔎 Monitor Vault-generated database users and connections
- 🛠️ Troubleshoot common Vault and PostgreSQL issues

---

## 🧰 Technology Stack

| Technology | Purpose |
|---|---|
| 🔐 **HashiCorp Vault** | Dynamic secret generation and lease management |
| 🐘 **PostgreSQL** | Database used to demonstrate dynamic credentials |
| 🐳 **Docker** | Runs PostgreSQL locally as an RDS simulation |
| ☁️ **AWS RDS Concepts** | Cloud database context for the lab |
| 🐧 **Linux** | Lab operating environment |
| 🖥️ **Vault CLI** | Vault configuration and secret management |
| 🐚 **Bash** | Automation and command execution |
| 🔧 **jq** | Parsing Vault JSON responses |
| 🛠️ **PostgreSQL Client** | Testing dynamically generated credentials |
| 📝 **Vault Audit Logging** | Tracking secret operations |

The source lab specifies Linux command-line knowledge, database/SQL familiarity, AWS RDS knowledge, Vault fundamentals, JSON configuration experience, and basic networking as prerequisites. fileciteturn0file0L14-L27

---

## 🏗️ Lab Architecture

```text
                         ┌─────────────────────────┐
                         │       Linux Host        │
                         │                         │
                         │  ┌───────────────────┐  │
                         │  │  HashiCorp Vault   │  │
                         │  │                    │  │
                         │  │ Database Secrets   │  │
                         │  │     Engine         │  │
                         │  └─────────┬─────────┘  │
                         │            │            │
                         │     Generate Credentials
                         │            │            │
                         │  ┌─────────▼─────────┐  │
                         │  │ PostgreSQL Docker │  │
                         │  │    Container      │  │
                         │  │                   │  │
                         │  │     testdb        │  │
                         │  └───────────────────┘  │
                         └─────────────────────────┘

              Credential Lifecycle:
        Create → Access → Renew → Expire/Revoke
                         │
                         ▼
                  Audit & Monitoring
```

---

# 🚀 Lab Workflow

## 🟢 Step 1 — Prepare the Linux Environment

### ✨ Install Required Dependencies

Update the system and install the tools required by the lab:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y curl wget unzip jq postgresql-client awscli

sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

Log out and log back in so the Docker group membership takes effect.

Verify Docker:

```bash
docker --version
```

---

## 🔐 Step 2 — Install HashiCorp Vault

Create the lab directory and install Vault:

```bash
mkdir -p ~/vault-lab
cd ~/vault-lab

wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip

unzip vault_1.15.2_linux_amd64.zip
sudo mv vault /usr/local/bin/
sudo chmod +x /usr/local/bin/vault

vault version
```

> ⚠️ The source lab uses Vault **1.15.2**. Keep the version consistent when reproducing the lab unless you intentionally adapt the instructions for another version. fileciteturn0file0L64-L84

---

## ⚙️ Step 3 — Configure and Initialize Vault

Create the Vault configuration:

```bash
mkdir -p ~/vault-lab/config

cat > ~/vault-lab/config/vault.hcl << 'EOF'
storage "file" {
  path = "/home/ubuntu/vault-lab/data"
}

listener "tcp" {
  address = "127.0.0.1:8200"
  tls_disable = true
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
disable_mlock = true
EOF

mkdir -p ~/vault-lab/data
```

Start Vault:

```bash
cd ~/vault-lab

vault server -config=config/vault.hcl > vault.log 2>&1 &

sleep 5

export VAULT_ADDR='http://127.0.0.1:8200'
echo 'export VAULT_ADDR="http://127.0.0.1:8200"' >> ~/.bashrc
```

Initialize and unseal Vault:

```bash
vault operator init -key-shares=1 -key-threshold=1 > vault-keys.txt

UNSEAL_KEY=$(grep 'Unseal Key 1:' vault-keys.txt | awk '{print $NF}')
ROOT_TOKEN=$(grep 'Initial Root Token:' vault-keys.txt | awk '{print $NF}')

vault operator unseal $UNSEAL_KEY
vault auth $ROOT_TOKEN

vault status
```

> 🔒 **Security Note:** Never commit `vault-keys.txt`, root tokens, unseal keys, passwords, or generated credentials to Git.

---

# 🐘 Step 4 — Deploy PostgreSQL

The lab uses PostgreSQL in Docker to simulate an AWS RDS database:

```bash
docker run --name postgres-db \
  -e POSTGRES_DB=testdb \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=rootpassword \
  -p 5432:5432 \
  -d postgres:13

sleep 10

docker ps | grep postgres-db
```

---

## 🗃️ Step 5 — Prepare the Database

Create the Vault management user and sample data:

```bash
docker exec -it postgres-db psql -U postgres -d testdb << 'EOF'
CREATE USER vault WITH PASSWORD 'vaultpassword';

ALTER USER vault CREATEDB;
ALTER USER vault CREATEROLE;

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50)
);

INSERT INTO employees (name, department) VALUES
('John Doe', 'Engineering'),
('Jane Smith', 'Marketing'),
('Bob Johnson', 'Sales');

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO vault;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO vault;

\q
EOF
```

Test the database connection:

```bash
PGPASSWORD=vaultpassword psql \
  -h localhost \
  -U vault \
  -d testdb \
  -c "SELECT * FROM employees;"
```

---

# 🔑 Step 6 — Enable the Database Secrets Engine

Enable the Vault database secrets engine:

```bash
vault secrets enable database
```

Verify:

```bash
vault secrets list
```

The lab then configures Vault to communicate with PostgreSQL through the PostgreSQL database plugin. fileciteturn0file0L221-L250

---

## 🔗 Step 7 — Configure the Database Connection

```bash
vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/testdb?sslmode=disable" \
    allowed_roles="readonly,readwrite" \
    username="vault" \
    password="vaultpassword"
```

Verify the configuration:

```bash
vault read database/config/postgresql
```

---

# 👥 Step 8 — Create Dynamic Database Roles

## 🔵 Read-Only Role

```bash
vault write database/roles/readonly \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

## 🟠 Read-Write Role

```bash
vault write database/roles/readwrite \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

Verify:

```bash
vault list database/roles

vault read database/roles/readonly
vault read database/roles/readwrite
```

These roles demonstrate least-privilege access by assigning different database permissions to dynamically generated users. fileciteturn0file0L254-L278

---

# 🎫 Step 9 — Generate Dynamic Credentials

Generate read-only credentials:

```bash
vault read database/creds/readonly
```

Capture them as JSON:

```bash
READONLY_CREDS=$(vault read -format=json database/creds/readonly)

READONLY_USER=$(echo $READONLY_CREDS | jq -r '.data.username')
READONLY_PASS=$(echo $READONLY_CREDS | jq -r '.data.password')

echo "Read-only Username: $READONLY_USER"
echo "Read-only Password: $READONLY_PASS"
```

Generate read-write credentials:

```bash
vault read database/creds/readwrite
```

```bash
READWRITE_CREDS=$(vault read -format=json database/creds/readwrite)

READWRITE_USER=$(echo $READWRITE_CREDS | jq -r '.data.username')
READWRITE_PASS=$(echo $READWRITE_CREDS | jq -r '.data.password')

echo "Read-write Username: $READWRITE_USER"
echo "Read-write Password: $READWRITE_PASS"
```

---

# 🧪 Step 10 — Test Access Control

## 🔵 Test Read-Only Credentials

A `SELECT` should succeed:

```bash
PGPASSWORD=$READONLY_PASS psql \
  -h localhost \
  -U $READONLY_USER \
  -d testdb \
  -c "SELECT * FROM employees;"
```

An `INSERT` should fail:

```bash
PGPASSWORD=$READONLY_PASS psql \
  -h localhost \
  -U $READONLY_USER \
  -d testdb \
  -c "INSERT INTO employees (name, department) VALUES ('Test User', 'Test Dept');"
```

Expected result:

```text
INSERT failed as expected for read-only user
```

---

## 🟠 Test Read-Write Credentials

Test `SELECT`:

```bash
PGPASSWORD=$READWRITE_PASS psql \
  -h localhost \
  -U $READWRITE_USER \
  -d testdb \
  -c "SELECT * FROM employees;"
```

Test `INSERT`:

```bash
PGPASSWORD=$READWRITE_PASS psql \
  -h localhost \
  -U $READWRITE_USER \
  -d testdb \
  -c "INSERT INTO employees (name, department) VALUES ('Dynamic User', 'Test Department');"
```

Test `UPDATE`:

```bash
PGPASSWORD=$READWRITE_PASS psql \
  -h localhost \
  -U $READWRITE_USER \
  -d testdb \
  -c "UPDATE employees SET department = 'Updated Department' WHERE name = 'Dynamic User';"
```

This verifies that Vault-generated credentials receive the permissions defined by their database role. fileciteturn0file0L315-L352

---

# ⏳ Step 11 — Manage Secret Leases

Dynamic credentials are issued with Vault leases.

List active leases:

```bash
vault list sys/leases/lookup/database/creds/readonly
vault list sys/leases/lookup/database/creds/readwrite
```

Inspect a lease:

```bash
vault write sys/leases/lookup \
  lease_id=$(vault read -field=lease_id database/creds/readonly)
```

---

# 🔄 Step 12 — Renew a Dynamic Secret

Generate a credential and capture its lease:

```bash
NEW_CREDS=$(vault read -format=json database/creds/readonly)

LEASE_ID=$(echo $NEW_CREDS | jq -r '.lease_id')
NEW_USER=$(echo $NEW_CREDS | jq -r '.data.username')
NEW_PASS=$(echo $NEW_CREDS | jq -r '.data.password')

echo "New Lease ID: $LEASE_ID"
echo "New Username: $NEW_USER"
```

Test the credential:

```bash
PGPASSWORD=$NEW_PASS psql \
  -h localhost \
  -U $NEW_USER \
  -d testdb \
  -c "SELECT COUNT(*) FROM employees;"
```

Renew the lease:

```bash
vault write sys/leases/renew \
  lease_id=$LEASE_ID \
  increment=3600
```

Verify:

```bash
vault write sys/leases/lookup lease_id=$LEASE_ID
```

---

# ❌ Step 13 — Revoke Dynamic Credentials

Test the credential before revocation:

```bash
PGPASSWORD=$NEW_PASS psql \
  -h localhost \
  -U $NEW_USER \
  -d testdb \
  -c "SELECT 'Credentials work before revocation' as status;"
```

Revoke:

```bash
vault write sys/leases/revoke lease_id=$LEASE_ID
```

Wait:

```bash
sleep 5
```

Test again:

```bash
PGPASSWORD=$NEW_PASS psql \
  -h localhost \
  -U $NEW_USER \
  -d testdb \
  -c "SELECT 'This should fail' as status;"
```

Expected behavior:

```text
Credentials successfully revoked - access denied as expected
```

Verify the database user was removed:

```bash
docker exec -it postgres-db psql \
  -U postgres \
  -d testdb \
  -c "SELECT usename FROM pg_user WHERE usename = '$NEW_USER';"
```

The lab explicitly demonstrates the lifecycle from working credentials to lease revocation and denied access. fileciteturn0file0L399-L420

---

# ⏱️ Step 14 — Configure Short-Lived Credentials

Create a role with a shorter TTL:

```bash
vault write database/roles/short-lived \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="5m" \
    max_ttl="10m"
```

Generate credentials:

```bash
SHORT_CREDS=$(vault read -format=json database/creds/short-lived)

SHORT_USER=$(echo $SHORT_CREDS | jq -r '.data.username')
SHORT_PASS=$(echo $SHORT_CREDS | jq -r '.data.password')
SHORT_LEASE=$(echo $SHORT_CREDS | jq -r '.lease_id')

echo "Short-lived Username: $SHORT_USER"
echo "Lease Duration: $(echo $SHORT_CREDS | jq -r '.lease_duration') seconds"
```

Test:

```bash
PGPASSWORD=$SHORT_PASS psql \
  -h localhost \
  -U $SHORT_USER \
  -d testdb \
  -c "SELECT 'Short-lived credentials work' as status;"
```

---

# 🧹 Step 15 — Perform Bulk Revocation

Generate multiple credential sets:

```bash
for i in {1..3}; do
    vault read database/creds/readonly > /dev/null
    echo "Generated credential set $i"
done
```

List active leases:

```bash
vault list sys/leases/lookup/database/creds/readonly
```

Revoke all leases for the role:

```bash
vault write sys/leases/revoke-prefix \
  prefix=database/creds/readonly
```

Verify:

```bash
vault list sys/leases/lookup/database/creds/readonly 2>/dev/null \
  || echo "No active leases found - bulk revocation successful"
```

---

# 📋 Step 16 — Enable Audit Logging

Enable Vault file audit logging:

```bash
vault audit enable file \
  file_path=/home/ubuntu/vault-lab/audit.log
```

Generate activity:

```bash
vault read database/creds/readonly > /dev/null
vault read database/creds/readwrite > /dev/null
```

Review audit entries:

```bash
tail -n 5 ~/vault-lab/audit.log | \
  jq '.request.path, .response.data.username'
```

Audit logging provides visibility into dynamic secret operations and supports security monitoring. fileciteturn0file0L480-L498

---

# 📊 Step 17 — Monitor Database Connections

Check active Vault-created connections:

```bash
docker exec -it postgres-db psql \
  -U postgres \
  -d testdb \
  -c "SELECT usename, application_name, client_addr, state FROM pg_stat_activity WHERE usename LIKE 'v-token-%';"
```

List Vault-created database users:

```bash
docker exec -it postgres-db psql \
  -U postgres \
  -d testdb \
  -c "SELECT usename, valuntil FROM pg_user WHERE usename LIKE 'v-token-%';"
```

---

# 🛠️ Troubleshooting

## ❗ PostgreSQL Connection Failure

Check the container:

```bash
docker ps | grep postgres-db
```

View logs:

```bash
docker logs postgres-db
```

Test the connection:

```bash
PGPASSWORD=vaultpassword psql \
  -h localhost \
  -U vault \
  -d testdb \
  -c "SELECT 1;"
```

Restart if necessary:

```bash
docker restart postgres-db
```

---

## 🔒 Vault Is Sealed

Check Vault:

```bash
vault status
```

Unseal:

```bash
vault operator unseal $UNSEAL_KEY
```

---

## 🚫 Permission Denied

Check Vault database-user privileges:

```bash
docker exec -it postgres-db psql \
  -U postgres \
  -d testdb \
  -c "SELECT rolname, rolcreaterole, rolcreatedb FROM pg_roles WHERE rolname = 'vault';"
```

Re-grant privileges:

```bash
docker exec -it postgres-db psql \
  -U postgres \
  -d testdb \
  -c "ALTER USER vault CREATEDB; ALTER USER vault CREATEROLE;"
```

These troubleshooting checks are included in the source lab for database connectivity, Vault seal status, and PostgreSQL permissions. fileciteturn0file0L515-L560

---

# ✅ Lab Verification

Use the following checks before considering the lab complete.

### 1️⃣ Verify Vault

```bash
vault status
```

### 2️⃣ Verify Database Secrets Engine

```bash
vault secrets list | grep database
```

### 3️⃣ Verify Database Connection

```bash
vault read database/config/postgresql
```

### 4️⃣ Verify Database Roles

```bash
vault list database/roles
```

### 5️⃣ Generate Test Credentials

```bash
TEST_CREDS=$(vault read -format=json database/creds/readonly)

TEST_USER=$(echo $TEST_CREDS | jq -r '.data.username')
TEST_PASS=$(echo $TEST_CREDS | jq -r '.data.password')

PGPASSWORD=$TEST_PASS psql \
  -h localhost \
  -U $TEST_USER \
  -d testdb \
  -c "SELECT 'Lab verification successful' as result;"
```

### 6️⃣ Verify Audit Logging

```bash
tail -n 1 ~/vault-lab/audit.log | jq '.type'
```

The source verification procedure checks Vault status, the database secrets engine, database configuration, roles, generated credentials, and audit logging. fileciteturn0file0L562-L590

---

# 🧹 Cleanup

Revoke active database leases:

```bash
vault write sys/leases/revoke-prefix prefix=database/
```

Stop and remove PostgreSQL:

```bash
docker stop postgres-db
docker rm postgres-db
```

Stop Vault:

```bash
pkill vault
```

Optionally remove the lab files:

```bash
rm -rf ~/vault-lab
```

> ⚠️ Before deleting the lab directory, make sure you no longer need the Vault configuration, audit logs, or other lab artifacts.

---

# 🎓 Key Learning Outcomes

After completing this lab, you should understand:

- 🔐 Why dynamic secrets are safer than static credentials
- 🎫 How Vault generates temporary database credentials
- 👥 How Vault roles control database permissions
- ⏱️ How TTLs control credential lifetime
- 🔄 How leases can be renewed
- ❌ How leases and credentials can be revoked
- 🧹 How bulk revocation works
- 📋 How Vault audit logging supports security monitoring
- 🔎 How to inspect Vault-created database users
- 🛡️ How dynamic credentials reduce credential exposure

---

# 🌟 Why Dynamic Secrets Matter

Dynamic secrets provide several important security benefits:

### 🔐 Reduced Credential Sprawl
Each access request can receive a unique credential instead of sharing a permanent password.

### 💥 Reduced Blast Radius
Short-lived credentials limit the useful lifetime of a compromised credential.

### 📋 Better Auditing
Vault can record secret-related activity so administrators can investigate access.

### 🔄 Automated Credential Lifecycle
Credentials can be created, renewed, expired, and revoked without manually managing permanent passwords.

### 🛡️ Least Privilege
Different Vault database roles can provide only the permissions required by a workload.

The source lab highlights credential-sprawl reduction, reduced blast radius, audit trails, automatic rotation, and least-privilege access as key reasons dynamic secrets improve security. fileciteturn0file0L613-L635

---

# ☁️ Real-World DevOps Use Cases

The concepts demonstrated in this lab are applicable to:

- ☁️ Cloud-native applications
- 🐳 Containerized workloads
- ☸️ Kubernetes environments
- 🔄 CI/CD pipelines
- 🧩 Microservices architectures
- 🛡️ Zero-trust security models
- 🗄️ Managed database environments
- 🔐 Automated secrets management

The source material also notes that the approach can be extended to services such as AWS IAM, Azure, Google Cloud Platform, and SSH access. fileciteturn0file0L625-L637

---

# 🏁 Conclusion

This lab provides a practical introduction to **HashiCorp Vault Dynamic Secrets**.

You configured Vault, deployed PostgreSQL, enabled the database secrets engine, created permission-based roles, generated temporary credentials, tested access controls, renewed leases, revoked credentials, performed bulk revocation, and enabled auditing.

The core security model can be summarized as:

```text
        Application / User
                │
                ▼
        ┌───────────────┐
        │ HashiCorp     │
        │    Vault      │
        └───────┬───────┘
                │
       Generate temporary
          credentials
                │
                ▼
        ┌───────────────┐
        │  PostgreSQL   │
        │   Database    │
        └───────┬───────┘
                │
        TTL / Lease expires
                │
                ▼
          Credential
            Revoked
```

> 🚀 **Final Takeaway:** Dynamic secrets transform database credential management from a static-password model into an automated, short-lived, auditable security workflow.

---

## 👨‍💻 Author

**Hafiz Muhammad Salman**  
Cloud DevOps Engineer | Linux Administrator

---

⭐ **If this lab helped you understand Vault Dynamic Secrets, consider starring the repository and continuing to practice secure secrets management!**
