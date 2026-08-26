# 🔐 Vault High Availability with Consul Storage Backend

![Vault](https://img.shields.io/badge/HashiCorp-Vault-000000?logo=vault\&logoColor=white)
![Consul](https://img.shields.io/badge/HashiCorp-Consul-F24C53?logo=consul\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Ubuntu%2FDebian-E95420?logo=ubuntu\&logoColor=white)
![Bash](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnu-bash\&logoColor=white)
![systemd](https://img.shields.io/badge/Service-systemd-000000?logo=linux\&logoColor=white)
![JSON](https://img.shields.io/badge/Config-JSON-000000?logo=json\&logoColor=white)
![HCL](https://img.shields.io/badge/Config-HCL-7B42BC?logo=hashicorp\&logoColor=white)

> 🧪 **Hands-on DevOps / Security Lab**
> Configure Vault with Consul storage, operate multiple Vault server processes, monitor their health, and simulate node failure and recovery.

---

## 📌 Lab Overview

This lab demonstrates the operational concepts behind **HashiCorp Vault High Availability (HA)** using **Consul as the storage backend**.

The practice environment uses a **single Linux VM** and runs:

* 🟢 One Consul agent
* 🔐 Three Vault server processes
* ⚙️ Three systemd service units
* 📦 Consul-backed Vault storage
* 📊 A custom Vault/Consul monitoring script
* 💥 Node failure and recovery simulations

> ⚠️ **Important architecture note:**
> This lab uses separate Consul storage paths for the three Vault processes so they can be initialized independently on one machine. Therefore, this is an **HA simulation and operational exercise**, not a production-grade Vault HA cluster. In a real Consul-backed Vault deployment, Vault servers share the same Consul storage path and communicate through a multi-server Consul cluster.

---

# 🎯 Objectives

By completing this lab, you will be able to:

* 🔐 Configure a multi-node Vault environment using Consul storage
* 🗄️ Configure Consul as the Vault storage backend
* 🚀 Run multiple Vault server instances with systemd
* 🔑 Initialize and unseal Vault nodes
* 📦 Store and retrieve KV secrets
* 📊 Build a Vault/Consul health-monitoring script
* 💥 Simulate a Vault node failure
* 🔄 Restore a failed Vault node
* 🔎 Diagnose common Vault and Consul configuration problems
* 🛡️ Understand important considerations for production Vault HA deployments

---

# 🧰 Technology Stack

| Technology               | Purpose                        |
| ------------------------ | ------------------------------ |
| 🔐 **HashiCorp Vault**   | Secrets management             |
| 🗄️ **HashiCorp Consul** | Storage backend                |
| 🐧 **Linux**             | Lab operating system           |
| ⚙️ **systemd**           | Service management             |
| 🐚 **Bash**              | Automation and monitoring      |
| 🔎 **jq**                | JSON processing                |
| 🌐 **Netcat / TCP**      | Connectivity testing           |
| 📄 **HCL**               | Vault and Consul configuration |
| 📋 **JSON**              | Vault initialization output    |

---

# 🏗️ Lab Architecture

```text
                         ┌─────────────────────────┐
                         │       Linux VM           │
                         │                         │
                         │      Consul Agent       │
                         │      127.0.0.1:8500     │
                         │            │            │
                         │     Consul Storage      │
                         │            │            │
                         │    ┌───────┼───────┐    │
                         │    │       │       │    │
                         │    ▼       ▼       ▼    │
                         │ Vault 1  Vault 2  Vault 3│
                         │ :8200    :8210    :8220 │
                         │    │       │       │    │
                         │    └───────┼───────┘    │
                         │            │            │
                         │      Monitoring Script  │
                         └─────────────────────────┘
```

### 🔌 Port Mapping

| Component    |   Port | Purpose         |
| ------------ | -----: | --------------- |
| Vault Node 1 | `8200` | Vault API/UI    |
| Vault Node 1 | `8201` | Cluster address |
| Vault Node 2 | `8210` | Vault API/UI    |
| Vault Node 2 | `8211` | Cluster address |
| Vault Node 3 | `8220` | Vault API/UI    |
| Vault Node 3 | `8221` | Cluster address |
| Consul       | `8500` | HTTP API/UI     |

---

# 📋 Prerequisites

Before starting, you should have:

* Basic Linux command-line knowledge
* Basic Vault knowledge
* Understanding of:

  * Seal/unseal
  * Vault tokens
  * Secrets engines
  * KV secrets
* Familiarity with:

  * systemd
  * JSON
  * HCL
  * Bash
* Ubuntu 20.04+ or another Debian-based Linux system
* `sudo` access
* A single isolated practice VM

---

# 🚀 Step 1 — Prepare the Environment

## 🏷️ Technology Badges

`🐧 Linux` `📦 APT` `🔐 Vault` `🗄️ Consul`

Update the package index and install the required utilities:

```bash
sudo apt update
sudo apt install -y wget curl unzip jq netcat-openbsd

mkdir -p ~/vault-ha-lab
cd ~/vault-ha-lab
```

---

# 🔑 Step 2 — Install Vault and Consul

## 🏷️ Technology Badges

`🔐 HashiCorp Vault` `🗄️ HashiCorp Consul` `📦 APT Repository`

Add the HashiCorp repository:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
sudo gpg --dearmor \
-o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com \
$(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
```

Install Vault and Consul:

```bash
sudo apt update
sudo apt install -y vault consul
```

Verify:

```bash
vault version
consul version
```

### ✅ Expected Result

Both commands should return installed version information without errors.

---

# 👤 Step 3 — Create Vault and Consul Users

## 🏷️ Technology Badges

`🐧 Linux` `👤 Linux Users` `📁 File Permissions`

Create dedicated system users:

```bash
sudo useradd --system \
  --home /etc/vault.d \
  --shell /bin/false vault 2>/dev/null || true

sudo useradd --system \
  --home /etc/consul.d \
  --shell /bin/false consul 2>/dev/null || true
```

Create directories:

```bash
sudo mkdir -p \
  /opt/vault/data \
  /opt/consul/data \
  /etc/vault.d \
  /etc/consul.d
```

Set ownership:

```bash
sudo chown -R vault:vault /opt/vault /etc/vault.d
sudo chown -R consul:consul /opt/consul /etc/consul.d
```

---

# 🗄️ Step 4 — Configure Consul

## 🏷️ Technology Badges

`🗄️ Consul` `📄 HCL` `⚙️ systemd`

Create the Consul configuration:

```bash
sudo tee /etc/consul.d/consul.hcl > /dev/null <<'EOF'
datacenter = "dc1"
data_dir = "/opt/consul/data"
log_level = "INFO"
node_name = "consul-node-1"
bind_addr = "127.0.0.1"
client_addr = "127.0.0.1"
server = true
bootstrap_expect = 1

ui_config {
  enabled = true
}
EOF
```

Set ownership:

```bash
sudo chown consul:consul /etc/consul.d/consul.hcl
```

---

# ⚙️ Step 5 — Create the Consul systemd Service

```bash
sudo tee /etc/systemd/system/consul.service > /dev/null <<'EOF'
[Unit]
Description=Consul
Documentation=https://www.consul.io/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/consul.d/consul.hcl

[Service]
Type=notify
User=consul
Group=consul
ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d/
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

Reload systemd and start Consul:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now consul
```

Check the service:

```bash
sudo systemctl status consul --no-pager
```

Check cluster membership:

```bash
consul members
```

### ✅ Expected Result

You should see:

```text
consul-node-1
Status = alive
```

---

# 🔐 Step 6 — Configure Three Vault Nodes

## 🏷️ Technology Badges

`🔐 Vault` `📄 HCL` `🗄️ Consul Backend` `⚙️ systemd`

Generate three Vault configurations:

```bash
for i in 1 2 3; do
  port_api=$((8200 + (i-1)*10))
  port_cluster=$((8201 + (i-1)*10))

  sudo tee /etc/vault.d/vault-node${i}.hcl > /dev/null <<EOF
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault-node${i}/"
}

listener "tcp" {
  address     = "127.0.0.1:${port_api}"
  tls_disable = 1
}

api_addr     = "http://127.0.0.1:${port_api}"
cluster_addr = "http://127.0.0.1:${port_cluster}"
ui = true
disable_mlock = true
cluster_name = "vault-ha-cluster"
EOF
done
```

Set ownership:

```bash
sudo chown -R vault:vault /etc/vault.d
```

Verify:

```bash
ls -la /etc/vault.d/
```

### 📁 Expected Files

```text
/etc/vault.d/
├── vault-node1.hcl
├── vault-node2.hcl
└── vault-node3.hcl
```

Each node should have a unique API and cluster port.

---

# ⚙️ Step 7 — Create Vault systemd Services

## 🏷️ Technology Badges

`⚙️ systemd` `🔐 Vault` `🐧 Linux`

Create the three service units:

```bash
for i in 1 2 3; do
sudo tee /etc/systemd/system/vault-node${i}.service > /dev/null <<EOF
[Unit]
Description=Vault Node ${i}
Documentation=https://developer.hashicorp.com/vault/docs
Requires=network-online.target
After=network-online.target consul.service
ConditionFileNotEmpty=/etc/vault.d/vault-node${i}.hcl

[Service]
Type=notify
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
AmbientCapabilities=CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault-node${i}.hcl
ExecReload=/bin/kill --signal HUP \$MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF
done
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable the services:

```bash
sudo systemctl enable vault-node1 vault-node2 vault-node3
```

Start them sequentially:

```bash
sudo systemctl start vault-node1
sleep 3

sudo systemctl start vault-node2
sleep 3

sudo systemctl start vault-node3
sleep 3
```

Check:

```bash
sudo systemctl is-active \
  vault-node1 \
  vault-node2 \
  vault-node3
```

### ✅ Expected Result

```text
active
active
active
```

---

# 🔑 Step 8 — Initialize Vault Nodes

## 🏷️ Technology Badges

`🔐 Vault` `🔑 Initialization` `📄 JSON` `jq`

Create the initialization directory:

```bash
mkdir -p ~/vault-ha-lab/init
```

Initialize each node:

```bash
for i in 1 2 3; do
  port=$((8200 + (i-1)*10))

  export VAULT_ADDR="http://127.0.0.1:${port}"

  vault operator init \
    -key-shares=3 \
    -key-threshold=2 \
    -format=json \
    > ~/vault-ha-lab/init/node${i}-init.json

  echo "Node ${i} initialized on port ${port}"
done
```

Inspect Node 1:

```bash
cat ~/vault-ha-lab/init/node1-init.json | \
jq '.unseal_keys_b64, .root_token'
```

Each initialization file contains:

* 🔑 Three unseal keys
* 🎯 Root token
* 📄 JSON-formatted initialization data

> ⚠️ **Security Warning:** Never commit these initialization files to GitHub or any public repository. They contain highly sensitive Vault credentials.

---

# 🔓 Step 9 — Unseal All Vault Nodes

## 🏷️ Technology Badges

`🔓 Vault Unseal` `🔑 Key Shares` `jq`

Run:

```bash
for i in 1 2 3; do
  port=$((8200 + (i-1)*10))

  export VAULT_ADDR="http://127.0.0.1:${port}"

  KEY1=$(jq -r '.unseal_keys_b64[0]' \
    ~/vault-ha-lab/init/node${i}-init.json)

  KEY2=$(jq -r '.unseal_keys_b64[1]' \
    ~/vault-ha-lab/init/node${i}-init.json)

  vault operator unseal "$KEY1" > /dev/null
  vault operator unseal "$KEY2" > /dev/null

  echo "Node ${i} unseal status:"
  vault status -format=json | \
    jq '{sealed, initialized}'
done
```

### ✅ Expected Result

Each node should show:

```json
{
  "sealed": false,
  "initialized": true
}
```

---

# 📦 Step 10 — Store Test Secrets

## 🏷️ Technology Badges

`🔐 Vault KV` `📦 Secrets Engine` `🔑 Authentication`

Use Node 1:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

Load the root token:

```bash
ROOT1=$(jq -r '.root_token' \
  ~/vault-ha-lab/init/node1-init.json)
```

Login:

```bash
vault login -no-print "$ROOT1"
```

Enable KV v2:

```bash
vault secrets enable -path=secret kv-v2
```

Write a test secret:

```bash
vault kv put secret/app1 \
  username="admin" \
  password="secret123"
```

Read the secret:

```bash
vault kv get -format=json secret/app1 | \
jq '.data.data'
```

### ✅ Expected Result

```json
{
  "password": "secret123",
  "username": "admin"
}
```

---

# 📊 Step 11 — Create the Monitoring Script

## 🏷️ Technology Badges

`📊 Monitoring` `🐚 Bash` `jq` `systemctl` `TCP`

Create:

```bash
tee ~/vault-ha-lab/vault-ha-monitor.sh > /dev/null <<'EOF'
#!/bin/bash

echo "Vault HA Cluster Monitoring Report"
echo "==================================="
echo "Timestamp: $(date)"
echo ""

echo "System Services Status:"
echo "------------------------"

for svc in consul vault-node1 vault-node2 vault-node3; do
    echo "$svc: $(sudo systemctl is-active $svc)"
done

echo ""

echo "Consul Cluster Members:"
echo "------------------------"

consul members

echo ""

echo "Vault Node Status:"
echo "------------------"

for i in 1 2 3; do
    port=$((8200 + (i-1)*10))

    echo "Node $i (127.0.0.1:$port):"

    export VAULT_ADDR="http://127.0.0.1:$port"

    if status_json=$(vault status -format=json 2>/dev/null); then

        sealed=$(echo "$status_json" | jq -r '.sealed')
        initialized=$(echo "$status_json" | jq -r '.initialized')
        version=$(echo "$status_json" | jq -r '.version')

        echo "  Initialized: $initialized"
        echo "  Sealed: $sealed"
        echo "  Version: $version"

    else
        echo "  Status: UNREACHABLE or SEALED"
    fi

    echo ""
done

echo "Port Connectivity:"
echo "------------------"

for port in 8200 8210 8220 8500; do

    if timeout 1 bash -c \
        "</dev/tcp/127.0.0.1/$port" 2>/dev/null; then

        echo "  Port $port: OPEN"

    else
        echo "  Port $port: CLOSED"
    fi

done
EOF
```

Make it executable:

```bash
chmod +x ~/vault-ha-lab/vault-ha-monitor.sh
```

Run:

```bash
~/vault-ha-lab/vault-ha-monitor.sh
```

---

# 🟢 Step 12 — Verify Healthy Cluster

The monitoring report should confirm:

```text
consul: active
vault-node1: active
vault-node2: active
vault-node3: active
```

Vault nodes:

```text
Initialized: true
Sealed: false
```

Ports:

```text
Port 8200: OPEN
Port 8210: OPEN
Port 8220: OPEN
Port 8500: OPEN
```

---

# 💥 Step 13 — Simulate Node Failure

## 🏷️ Technology Badges

`💥 Failure Simulation` `⚙️ systemd` `📊 Monitoring`

Stop Node 1:

```bash
echo "Stopping Node 1..."

sudo systemctl stop vault-node1
sleep 3
```

Run the monitor:

```bash
~/vault-ha-lab/vault-ha-monitor.sh
```

### Expected State

Node 1:

```text
vault-node1: inactive
Port 8200: CLOSED
```

Nodes 2 and 3 should remain operational.

---

# 🔄 Step 14 — Test Operations During Failure

Use Node 2:

```bash
export VAULT_ADDR='http://127.0.0.1:8210'
```

Load Node 2's root token:

```bash
ROOT2=$(jq -r '.root_token' \
  ~/vault-ha-lab/init/node2-init.json)
```

Login:

```bash
vault login -no-print "$ROOT2"
```

Enable the KV engine if necessary:

```bash
vault secrets enable -path=secret kv-v2
```

Write a failover test secret:

```bash
vault kv put secret/failover-test \
  note="written while node1 is down"
```

Read it:

```bash
vault kv get -format=json secret/failover-test | \
jq '.data.data'
```

### ✅ Expected Result

Node 2 successfully accepts operations while Node 1 is stopped.

> 🧠 **Important:** Because this lab deliberately gives each Vault process a separate Consul path, this demonstrates **service continuity between independent Vault instances**, not true shared-storage Vault HA. A production HA cluster must use a shared storage backend and proper Vault cluster configuration.

---

# 🔄 Step 15 — Restore Node 1

## 🏷️ Technology Badges

`🔄 Recovery` `🔓 Vault Unseal` `⚙️ systemd`

Start Node 1:

```bash
sudo systemctl start vault-node1
sleep 3
```

Set the API address:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

Load the unseal keys:

```bash
KEY1=$(jq -r '.unseal_keys_b64[0]' \
  ~/vault-ha-lab/init/node1-init.json)

KEY2=$(jq -r '.unseal_keys_b64[1]' \
  ~/vault-ha-lab/init/node1-init.json)
```

Unseal:

```bash
vault operator unseal "$KEY1" > /dev/null
vault operator unseal "$KEY2" > /dev/null
```

Verify:

```bash
vault status -format=json | \
jq '{sealed, initialized}'
```

Run the monitor:

```bash
~/vault-ha-lab/vault-ha-monitor.sh
```

### ✅ Expected Result

Node 1 returns to:

```json
{
  "sealed": false,
  "initialized": true
}
```

All required services should report `active`.

---

# 🧪 Step 16 — Capture Lab Evidence

Save evidence before failure:

```bash
~/vault-ha-lab/vault-ha-monitor.sh \
  | tee ~/vault-ha-lab/monitor-before-failure.txt
```

Capture failure state:

```bash
sudo systemctl stop vault-node1
sleep 3

~/vault-ha-lab/vault-ha-monitor.sh \
  | tee ~/vault-ha-lab/monitor-during-failure.txt
```

Restore:

```bash
sudo systemctl start vault-node1
sleep 3
```

Unseal Node 1 again and capture the final state:

```bash
~/vault-ha-lab/vault-ha-monitor.sh \
  | tee ~/vault-ha-lab/monitor-after-recovery.txt
```

---

# 🔎 Troubleshooting

## ❌ Issue 1 — `consul members` Hangs or Shows Nothing

### Diagnose

```bash
sudo systemctl status consul --no-pager
```

```bash
sudo journalctl -u consul --no-pager -n 30
```

### Restart

```bash
sudo systemctl restart consul
```

Wait a few seconds:

```bash
consul members
```

Check for:

* Incorrect bind address
* Permission problems
* Consul service failure
* Invalid HCL configuration

---

# ❌ Issue 2 — Vault Returns `connection refused`

Check the service:

```bash
sudo systemctl is-active vault-node1
```

Test the port:

```bash
timeout 1 bash -c \
"</dev/tcp/127.0.0.1/8200" && \
echo OPEN || echo CLOSED
```

Inspect configuration:

```bash
sudo cat /etc/vault.d/vault-node1.hcl
```

Restart:

```bash
sudo systemctl restart vault-node1
```

Check logs:

```bash
sudo journalctl \
  -u vault-node1 \
  --no-pager \
  -n 30
```

---

# ❌ Issue 3 — High CPU or Memory Usage

Check CPU:

```bash
top -bn1 | head -15
```

Check memory:

```bash
free -h
```

If available:

```bash
iostat -x 1 3
```

Inspect Vault logs:

```bash
sudo journalctl \
  -u vault-node1 \
  --no-pager \
  -n 50
```

Look for:

* Crash/restart loops
* Configuration errors
* Permission errors
* Port conflicts
* Excessive resource consumption

---

# ❌ Issue 4 — Vault Already Initialized

If you see:

```text
error initializing vault: Vault is already initialized
```

The Consul storage path may already contain Vault data.

Check the path:

```bash
consul kv get -recurse vault-node1/
```

Only remove the data if you are intentionally resetting the lab:

```bash
consul kv delete -recurse vault-node1/
```

Repeat for the appropriate node if required.

> ⚠️ Never delete production Vault storage data as a troubleshooting shortcut.

---

# 🔐 Security Considerations

This lab intentionally disables TLS:

```hcl
tls_disable = 1
```

and uses:

```text
127.0.0.1
```

This is acceptable only for an isolated practice environment.

### Production environments should use:

* 🔒 TLS for Vault API and cluster communication
* 🔒 TLS for Consul communication
* 🔑 Secure token management
* 🛡️ Vault policies with least privilege
* 🔐 Auto-unseal
* ☁️ Cloud KMS/HSM where appropriate
* 🖥️ Multiple physical/virtual hosts
* 🗄️ A real multi-server Consul cluster or Vault Integrated Storage
* 📊 Centralized monitoring and alerting
* 💾 Backup and disaster-recovery procedures
* 🚫 No root tokens stored in plaintext files
* 🚫 No initialization keys committed to Git

---

# 📁 Lab Directory Structure

After completing the lab, your working directory should resemble:

```text
~/vault-ha-lab/
├── init/
│   ├── node1-init.json
│   ├── node2-init.json
│   └── node3-init.json
│
├── vault-ha-monitor.sh
├── monitor-before-failure.txt
├── monitor-during-failure.txt
└── monitor-after-recovery.txt
```

System configuration:

```text
/etc/vault.d/
├── vault-node1.hcl
├── vault-node2.hcl
└── vault-node3.hcl

/etc/consul.d/
└── consul.hcl

/etc/systemd/system/
├── consul.service
├── vault-node1.service
├── vault-node2.service
└── vault-node3.service
```

---

# 📊 Verification Checklist

* [ ] Vault installed successfully
* [ ] Consul installed successfully
* [ ] Vault system user created
* [ ] Consul system user created
* [ ] Consul service running
* [ ] `consul members` reports an alive member
* [ ] Three Vault configurations created
* [ ] Three Vault systemd services created
* [ ] All Vault services started successfully
* [ ] All three Vault nodes initialized
* [ ] All three Vault nodes unsealed
* [ ] KV secrets engine configured
* [ ] Test secret successfully written
* [ ] Test secret successfully read
* [ ] Monitoring script created
* [ ] Monitoring script reports service health
* [ ] Monitoring script reports port connectivity
* [ ] Node 1 failure simulated
* [ ] Node 2 successfully tested during Node 1 outage
* [ ] Node 1 successfully restarted
* [ ] Node 1 successfully unsealed
* [ ] Final health report verified
* [ ] Monitoring evidence saved

---

# 🎓 Expected Outcomes

After completing this lab, you should have practical experience with:

### 🔐 Vault Operations

* Vault initialization
* Vault sealing and unsealing
* Root-token authentication
* KV secrets engines
* Vault status verification

### 🗄️ Consul

* Consul installation
* Consul server configuration
* Consul service management
* Consul membership verification
* Consul-backed Vault storage

### ⚙️ Linux Administration

* systemd service creation
* Service lifecycle management
* Linux users and groups
* File permissions
* Journal/log inspection
* TCP port troubleshooting

### 📊 Monitoring

* Bash scripting
* `jq` JSON processing
* Service status checks
* TCP connectivity checks
* Vault health checks
* Failure-state reporting

### 💥 Reliability Engineering

* Failure simulation
* Service recovery
* Health verification
* Operational troubleshooting
* HA architecture analysis

---

# 🏆 Conclusion

This lab provided hands-on experience with the operational building blocks of **Vault high availability and Consul-backed storage**.

You configured Consul, deployed multiple Vault server processes, initialized and unsealed each instance, stored secrets, and created a custom monitoring script capable of checking service status, Vault health, and network connectivity.

You also simulated a Vault node outage and practiced recovering the failed service.

The lab highlights an important distinction between a **single-host HA simulation** and a **true production HA architecture**. For production environments, Vault nodes should share a properly designed storage backend, run across separate hosts or failure domains, use TLS, and implement secure automated unsealing.

### 🚀 Recommended Next Steps

1. **Vault Integrated Storage (Raft)** — Build a true multi-node Vault HA cluster.
2. **Multi-server Consul** — Deploy Consul across multiple hosts.
3. **TLS Security** — Enable encrypted Vault and Consul communication.
4. **Auto-Unseal** — Integrate Vault with AWS KMS, Azure Key Vault, or another supported KMS/HSM.
5. **Monitoring** — Integrate Vault metrics with Prometheus and Grafana.
6. **Automation** — Automate deployment using Terraform and Ansible.
7. **Disaster Recovery** — Practice backup, restore, and recovery procedures.

---

## ⭐ Lab Summary

```text
┌─────────────────────────────────────────────────────────┐
│             VAULT HA + CONSUL LAB                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  🗄️ Consul Backend                                     │
│          │                                              │
│          ├── 🔐 Vault Node 1 :8200                     │
│          ├── 🔐 Vault Node 2 :8210                     │
│          └── 🔐 Vault Node 3 :8220                     │
│                                                         │
│  📊 Monitoring Script                                   │
│          │                                              │
│          ├── ⚙️ Service Health                         │
│          ├── 🔐 Vault Seal State                       │
│          ├── 🗄️ Consul Membership                      │
│          └── 🌐 Port Connectivity                      │
│                                                         │
│  💥 Failure → 🔄 Recovery → ✅ Verification             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> 💡 **DevOps Takeaway:** High availability is not simply running multiple processes. A production HA design requires shared, durable storage, correct leader/standby behavior, secure communication, failure-domain separation, automated recovery, and continuous health monitoring.
