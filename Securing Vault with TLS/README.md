<div align="center">

# 🔒 Securing Vault with TLS

### Encrypting Vault Communications with TLS and Mutual TLS Authentication

![Vault](https://img.shields.io/badge/HashiCorp_Vault-FFEC6E?style=for-the-badge&logo=vault&logoColor=black)
![TLS](https://img.shields.io/badge/TLS-mTLS-005571?style=for-the-badge&logo=letsencrypt&logoColor=white)
![OpenSSL](https://img.shields.io/badge/OpenSSL-721412?style=for-the-badge&logo=openssl&logoColor=white)
![systemd](https://img.shields.io/badge/systemd-1A1A1A?style=for-the-badge&logo=linux&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

</div>

---

## 📋 Table of Contents

- [🎯 Learning Objectives](#-learning-objectives)
- [🔧 Prerequisites](#-prerequisites)
- [🖥️ Lab Environment](#️-lab-environment)
- [📘 Task 1: Set up Vault with TLS Encryption](#-task-1-set-up-vault-with-tls-encryption)
- [📗 Task 2: Implement Mutual TLS for Secure Communication](#-task-2-implement-mutual-tls-for-secure-communication)
- [🔎 Verification Steps](#-verification-steps)
- [🛡️ MITRE ATT&CK Mapping](#️-mitre-attck-mapping)
- [📚 Key Concepts](#-key-concepts)
- [🔍 Troubleshooting Tips](#-troubleshooting-tips)
- [✅ Conclusion](#-conclusion)

---

## 🎯 Learning Objectives

By the end of this lab, you will be able to:

| # | Objective |
|---|-----------|
| 1 | Install and configure HashiCorp Vault on a Linux system |
| 2 | Generate and configure TLS certificates for Vault encryption |
| 3 | Set up Vault with TLS encryption to secure data in transit |
| 4 | Implement mutual TLS (mTLS) authentication for enhanced security |
| 5 | Verify TLS and mTLS configurations are working correctly |
| 6 | Understand the importance of certificate-based security in production environments |

---

## 🔧 Prerequisites

| Requirement | Details |
|---|---|
| 🐧 Linux CLI | Basic understanding of Linux command line operations |
| 📁 File Permissions | Familiarity with file permissions and directory structures |
| 🔑 PKI | Basic knowledge of PKI (Public Key Infrastructure) concepts |
| 🔒 TLS/SSL | Understanding of TLS/SSL fundamentals |
| 📄 JSON | Basic knowledge of JSON configuration files |

---

## 🖥️ Lab Environment

> **☁️ Al Nafi Cloud Machine**
> Al Nafi provides Linux-based cloud machines for this lab. Simply click **Start Lab** to access your dedicated Linux machine. The provided Linux machine is bare metal with no pre-installed tools, so you will need to install all required tools during the lab following the instructions provided.

---

## 📘 Task 1: Set up Vault with TLS Encryption

### 🔹 Subtask 1.1: Install Required Tools

First, we need to install HashiCorp Vault and OpenSSL for certificate generation.

```bash
# 🔄 Update the system package manager
sudo apt update && sudo apt upgrade -y

# 📦 Install required dependencies
sudo apt install -y wget unzip openssl curl jq

# ⬇️ Download Vault
wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip

# 📂 Unzip the binary
unzip vault_1.15.2_linux_amd64.zip

# 📁 Move to system path
sudo mv vault /usr/local/bin/

# ✅ Verify installation
vault version

# 👤 Create a dedicated user for Vault
sudo useradd --system --home /etc/vault.d --shell /bin/false vault
```

### 🔹 Subtask 1.2: Create Directory Structure

Create the necessary directories for Vault configuration and data:

```bash
# 🗂️ Create Vault directories
sudo mkdir -p /etc/vault.d
sudo mkdir -p /opt/vault/data
sudo mkdir -p /opt/vault/logs
sudo mkdir -p /opt/vault/tls

# 🔑 Set proper ownership
sudo chown -R vault:vault /etc/vault.d
sudo chown -R vault:vault /opt/vault
```

### 🔹 Subtask 1.3: Generate TLS Certificates

Create a Certificate Authority (CA) and server certificates for TLS encryption:

```bash
# 🔐 Create a CA private key
sudo openssl genrsa -out /opt/vault/tls/ca-key.pem 4096

# 📜 Create a CA certificate
sudo openssl req -new -x509 -days 365 -key /opt/vault/tls/ca-key.pem -out /opt/vault/tls/ca-cert.pem -subj "/C=US/ST=CA/L=San Francisco/O=Lab/OU=IT/CN=Vault-CA"

# 🔐 Create a server private key
sudo openssl genrsa -out /opt/vault/tls/vault-key.pem 4096

# 📝 Create a certificate signing request (CSR)
sudo openssl req -new -key /opt/vault/tls/vault-key.pem -out /opt/vault/tls/vault.csr -subj "/C=US/ST=CA/L=San Francisco/O=Lab/OU=IT/CN=localhost"

# ✍️ Create a server certificate signed by the CA
sudo openssl x509 -req -days 365 -in /opt/vault/tls/vault.csr -CA /opt/vault/tls/ca-cert.pem -CAkey /opt/vault/tls/ca-key.pem -CAcreateserial -out /opt/vault/tls/vault-cert.pem -extensions v3_req -extfile <(cat <<EOF
[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
IP.1 = 127.0.0.1
EOF
)

# 🔒 Set proper permissions on certificates
sudo chmod 400 /opt/vault/tls/vault-key.pem
sudo chmod 444 /opt/vault/tls/vault-cert.pem
sudo chmod 444 /opt/vault/tls/ca-cert.pem
sudo chown -R vault:vault /opt/vault/tls
```

### 🔹 Subtask 1.4: Configure Vault with TLS

Create a Vault configuration file with TLS enabled:

```hcl
# ⚙️ Create the Vault configuration file
sudo tee /etc/vault.d/vault.hcl > /dev/null <<EOF
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_cert_file = "/opt/vault/tls/vault-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-key.pem"
  tls_min_version = "tls12"
}

api_addr = "https://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
log_level = "INFO"
EOF
```

### 🔹 Subtask 1.5: Start Vault Server

```ini
# ⚙️ Create a systemd service file for Vault
sudo tee /etc/systemd/system/vault.service > /dev/null <<EOF
[Unit]
Description=HashiCorp Vault
Documentation=https://www.vaultproject.io/docs/
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
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill -HUP \$MAINPID
KillMode=process
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitBurst=3
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

```bash
# ▶️ Enable and start the Vault service
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl start vault

# ✅ Check the service status
sudo systemctl status vault
```

### 🔹 Subtask 1.6: Initialize and Unseal Vault

```bash
# 🌐 Set the Vault address environment variable
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CACERT='/opt/vault/tls/ca-cert.pem'

# 🚀 Initialize Vault
vault operator init -key-shares=5 -key-threshold=3 > /tmp/vault-init.txt

# 👀 Display the initialization output
cat /tmp/vault-init.txt

# 🔑 Extract unseal keys (you'll need these)
UNSEAL_KEY_1=$(grep 'Unseal Key 1:' /tmp/vault-init.txt | awk '{print $4}')
UNSEAL_KEY_2=$(grep 'Unseal Key 2:' /tmp/vault-init.txt | awk '{print $4}')
UNSEAL_KEY_3=$(grep 'Unseal Key 3:' /tmp/vault-init.txt | awk '{print $4}')
ROOT_TOKEN=$(grep 'Initial Root Token:' /tmp/vault-init.txt | awk '{print $4}')

# 🔓 Unseal Vault using three keys
vault operator unseal $UNSEAL_KEY_1
vault operator unseal $UNSEAL_KEY_2
vault operator unseal $UNSEAL_KEY_3

# 🔐 Authenticate with the root token
vault auth $ROOT_TOKEN

# ✅ Verify TLS is working
vault status
```

---

## 📗 Task 2: Implement Mutual TLS for Secure Communication

### 🔹 Subtask 2.1: Generate Client Certificates

Create client certificates for mutual TLS authentication:

```bash
# 🔐 Generate a client private key
sudo openssl genrsa -out /opt/vault/tls/client-key.pem 4096

# 📝 Create a client certificate signing request
sudo openssl req -new -key /opt/vault/tls/client-key.pem -out /opt/vault/tls/client.csr -subj "/C=US/ST=CA/L=San Francisco/O=Lab/OU=IT/CN=vault-client"

# ✍️ Sign the client certificate with the CA
sudo openssl x509 -req -days 365 -in /opt/vault/tls/client.csr -CA /opt/vault/tls/ca-cert.pem -CAkey /opt/vault/tls/ca-key.pem -CAcreateserial -out /opt/vault/tls/client-cert.pem -extensions v3_req -extfile <(cat <<EOF
[v3_req]
keyUsage = keyEncipherment, dataEncipherment, digitalSignature
extendedKeyUsage = clientAuth
EOF
)

# 🔒 Set proper permissions
sudo chmod 400 /opt/vault/tls/client-key.pem
sudo chmod 444 /opt/vault/tls/client-cert.pem
sudo chown -R vault:vault /opt/vault/tls
```

### 🔹 Subtask 2.2: Configure Vault for Mutual TLS

```bash
# ⏹️ Stop the Vault service
sudo systemctl stop vault
```

```hcl
# ⚙️ Update the Vault configuration to require client certificates
sudo tee /etc/vault.d/vault.hcl > /dev/null <<EOF
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_cert_file = "/opt/vault/tls/vault-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-key.pem"
  tls_client_ca_file = "/opt/vault/tls/ca-cert.pem"
  tls_require_and_verify_client_cert = true
  tls_min_version = "tls12"
}

api_addr = "https://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
log_level = "INFO"
EOF
```

```bash
# ▶️ Start the Vault service
sudo systemctl start vault

# ✅ Check the service status
sudo systemctl status vault
```

### 🔹 Subtask 2.3: Configure Client for Mutual TLS

```bash
# 🌐 Update environment variables to include client certificates
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CACERT='/opt/vault/tls/ca-cert.pem'
export VAULT_CLIENT_CERT='/opt/vault/tls/client-cert.pem'
export VAULT_CLIENT_KEY='/opt/vault/tls/client-key.pem'

# 🧪 Test the mutual TLS connection
vault status

# 🔐 Authenticate with the root token
vault auth $ROOT_TOKEN
```

### 🔹 Subtask 2.4: Test Mutual TLS Authentication

```bash
# 🔐 Create a test secret to verify functionality
vault kv put secret/test-mtls username="testuser" password="securepass123"

# 📖 Retrieve the secret
vault kv get secret/test-mtls

# ❌ Temporarily unset client certificate variables
unset VAULT_CLIENT_CERT
unset VAULT_CLIENT_KEY

# ⚠️ This should fail with a TLS error
vault status

# ♻️ Restore client certificates
export VAULT_CLIENT_CERT='/opt/vault/tls/client-cert.pem'
export VAULT_CLIENT_KEY='/opt/vault/tls/client-key.pem'

# ✅ This should work again
vault status
```

### 🔹 Subtask 2.5: Create Additional Client Certificates

Create another client certificate to demonstrate multiple client access:

```bash
# 🔐 Generate another client private key
sudo openssl genrsa -out /opt/vault/tls/client2-key.pem 4096

# 📝 Create a certificate signing request
sudo openssl req -new -key /opt/vault/tls/client2-key.pem -out /opt/vault/tls/client2.csr -subj "/C=US/ST=CA/L=San Francisco/O=Lab/OU=IT/CN=vault-client2"

# ✍️ Sign the certificate
sudo openssl x509 -req -days 365 -in /opt/vault/tls/client2.csr -CA /opt/vault/tls/ca-cert.pem -CAkey /opt/vault/tls/ca-key.pem -CAcreateserial -out /opt/vault/tls/client2-cert.pem -extensions v3_req -extfile <(cat <<EOF
[v3_req]
keyUsage = keyEncipherment, dataEncipherment, digitalSignature
extendedKeyUsage = clientAuth
EOF
)

# 🧪 Test with the second client certificate
export VAULT_CLIENT_CERT='/opt/vault/tls/client2-cert.pem'
export VAULT_CLIENT_KEY='/opt/vault/tls/client2-key.pem'

vault status
vault auth $ROOT_TOKEN
vault kv get secret/test-mtls
```

### 🔹 Subtask 2.6: Verify Certificate Details

```bash
# 🔍 Examine the server certificate
openssl x509 -in /opt/vault/tls/vault-cert.pem -text -noout | grep -A 5 "Subject:"

# 🔍 Examine the client certificate
openssl x509 -in /opt/vault/tls/client-cert.pem -text -noout | grep -A 5 "Subject:"

# ✅ Verify certificate chain
openssl verify -CAfile /opt/vault/tls/ca-cert.pem /opt/vault/tls/vault-cert.pem
openssl verify -CAfile /opt/vault/tls/ca-cert.pem /opt/vault/tls/client-cert.pem
```

---

## 🔎 Verification Steps

```bash
# ✅ Verify TLS is enabled
curl -k https://127.0.0.1:8200/v1/sys/health

# 🔐 Verify mutual TLS is working
curl --cert /opt/vault/tls/client-cert.pem --key /opt/vault/tls/client-key.pem --cacert /opt/vault/tls/ca-cert.pem https://127.0.0.1:8200/v1/sys/health

# 📅 Verify certificate expiration
openssl x509 -in /opt/vault/tls/vault-cert.pem -noout -dates
```

---

## 🛡️ MITRE ATT&CK Mapping

| Technique ID | Technique Name | Relevance to This Lab |
|---|---|---|
| [T1557](https://attack.mitre.org/techniques/T1557/) | Adversary-in-the-Middle | TLS encryption on the Vault listener prevents interception and tampering of traffic between clients and the server |
| [T1040](https://attack.mitre.org/techniques/T1040/) | Network Sniffing | Encrypting the Vault API endpoint in transit defeats passive traffic capture of secrets and tokens |
| [T1552.004](https://attack.mitre.org/techniques/T1552/004/) | Private Keys | CA, server, and client private keys are generated locally and locked down with `chmod 400`/ownership restricted to the `vault` user |
| [T1078](https://attack.mitre.org/techniques/T1078/) | Valid Accounts | `tls_require_and_verify_client_cert` restricts API access to holders of a client certificate signed by the trusted CA |
| [T1552.001](https://attack.mitre.org/techniques/T1552/001/) | Credentials In Files | Unseal keys and the root token are written to `/tmp/vault-init.txt`, a file that must be protected and cleared after use |

---

## 📚 Key Concepts

| Concept | Description |
|---|---|
| **Certificate Authority (CA)** | A self-signed root certificate used to sign and trust the server and client certificates in this lab |
| **Mutual TLS (mTLS)** | A TLS mode where both the server and the client present certificates, so each side authenticates the other |
| **Certificate Signing Request (CSR)** | A request containing a public key and identity info, submitted to a CA to obtain a signed certificate |
| **Subject Alternative Name (SAN)** | Certificate extension specifying which hostnames/IPs a certificate is valid for (here, `localhost`/`127.0.0.1`) |
| **Shamir's Secret Sharing** | The scheme behind `-key-shares`/`-key-threshold`, splitting Vault's master key into multiple unseal key shares |
| **Seal / Unseal** | Vault's default-locked state after startup; a threshold number of unseal keys must be supplied to decrypt the storage backend |
| **Root Token** | The initial, highest-privilege Vault authentication token generated at `vault operator init` |
| **systemd Hardening Directives** | Unit-file options such as `ProtectSystem`, `PrivateTmp`, and `NoNewPrivileges` that sandbox the Vault process at the OS level |

---

## 🔍 Troubleshooting Tips

<details>
<summary><strong>Issue: Vault fails to start with TLS configuration</strong></summary>

**Solution:** Check certificate file permissions and paths in the configuration file

```bash
# 📜 View logs
sudo journalctl -u vault -f
```

</details>

<details>
<summary><strong>Issue: Client certificate authentication fails</strong></summary>

**Solution:** Verify that the client certificate is signed by the same CA as configured in Vault

```bash
openssl verify -CAfile /opt/vault/tls/ca-cert.pem /opt/vault/tls/client-cert.pem
```

</details>

<details>
<summary><strong>Issue: TLS handshake errors</strong></summary>

**Solution:** Ensure that the server certificate includes the correct Subject Alternative Names (SAN)

```bash
openssl x509 -in /opt/vault/tls/vault-cert.pem -text -noout | grep -A 5 "Subject Alternative Name"
```

</details>

<details>
<summary><strong>Issue: Permission denied errors</strong></summary>

**Solution:** Verify that the vault user has read access to certificate files

```bash
sudo ls -la /opt/vault/tls/
```

</details>

---

## ✅ Conclusion

In this lab, you have successfully:

### 🏆 Key Accomplishments

- Installed and configured HashiCorp Vault on a Linux system
- Generated a complete PKI infrastructure with CA and server certificates
- Configured Vault to use TLS encryption for secure communication
- Implemented mutual TLS authentication requiring client certificates
- Created and tested multiple client certificates
- Verified that both server and client authentication work correctly

### 💡 Why This Matters

TLS and mutual TLS are critical security measures in production environments. TLS encrypts data in transit, preventing eavesdropping and man-in-the-middle attacks. Mutual TLS adds an additional layer of security by requiring clients to present valid certificates, ensuring that only authorized clients can access the Vault server. This is especially important for secrets management systems like Vault, where unauthorized access could compromise sensitive data across an entire organization.

### 🌍 Real-World Applications

The skills you've learned in this lab are directly applicable to securing production Vault deployments and other services that require strong authentication and encryption. Understanding certificate management and TLS configuration is essential for maintaining secure infrastructure in modern cloud and on-premises environments.

---

<div align="center">

![Al Nafi](https://img.shields.io/badge/Al_Nafi-Cybersecurity_Training-blueviolet?style=for-the-badge)

</div>
