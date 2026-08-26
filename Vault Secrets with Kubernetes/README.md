<div align="center">

# 🔐 Vault Secrets with Kubernetes

### Dynamic Secret Injection and Kubernetes API Token Generation with HashiCorp Vault

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Vault](https://img.shields.io/badge/HashiCorp_Vault-FFEC6E?style=for-the-badge&logo=vault&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![kind](https://img.shields.io/badge/kind-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

</div>

---

## 📋 Table of Contents

- [🎯 Learning Objectives](#-learning-objectives)
- [🔧 Prerequisites](#-prerequisites)
- [🖥️ Lab Environment](#️-lab-environment)
- [📘 Task 1: Environment Setup and Vault Installation](#-task-1-environment-setup-and-vault-installation)
- [📗 Task 2: Create Kubernetes Cluster and Configure Vault](#-task-2-create-kubernetes-cluster-and-configure-vault)
- [📙 Task 3: Deploy Kubernetes Workloads with Vault Integration](#-task-3-deploy-kubernetes-workloads-with-vault-integration)
- [📕 Task 4: Use Vault for Kubernetes API Tokens](#-task-4-use-vault-for-kubernetes-api-tokens)
- [📔 Task 5: Verification and Testing](#-task-5-verification-and-testing)
- [🛡️ MITRE ATT&CK Mapping](#️-mitre-attck-mapping)
- [📚 Key Concepts](#-key-concepts)
- [🔍 Troubleshooting Tips](#-troubleshooting-tips)
- [🧹 Cleanup](#-cleanup)
- [✅ Conclusion](#-conclusion)

---

## 🎯 Learning Objectives

By the end of this lab, you will be able to:

| # | Objective |
|---|-----------|
| 1 | Install and configure HashiCorp Vault on a single Linux machine |
| 2 | Set up a local Kubernetes cluster using kind (Kubernetes in Docker) |
| 3 | Configure Vault's Kubernetes authentication method |
| 4 | Deploy Kubernetes workloads that dynamically retrieve secrets from Vault |
| 5 | Use Vault to manage Kubernetes service account tokens |
| 6 | Implement secure secret injection into pods using Vault Agent |
| 7 | Understand the integration patterns between Vault and Kubernetes |

---

## 🔧 Prerequisites

| Requirement | Details |
|---|---|
| 🐧 Linux CLI | Basic understanding of Linux command line operations |
| 🐳 Docker | Familiarity with Docker containers and basic concepts |
| ☸️ Kubernetes | Knowledge of Kubernetes fundamentals (pods, services, deployments) |
| 🔑 Secrets Management | Understanding of secrets management concepts |
| 📄 YAML | Basic knowledge of YAML configuration files |

---

## 🖥️ Lab Environment

> **☁️ Al Nafi Cloud Machine**
> Al Nafi provides Linux-based cloud machines for this lab. Simply click **Start Lab** to access your dedicated Linux machine. The provided machine is bare metal with no pre-installed tools, so you will install all required components during the lab exercises.

---

## 📘 Task 1: Environment Setup and Vault Installation

### 🔹 Subtask 1.1: Install Required Dependencies

First, update your system and install essential tools:

```bash
# 🔄 Update package manager
sudo apt update && sudo apt upgrade -y

# 📦 Install essential tools
sudo apt install -y curl wget unzip jq apt-transport-https ca-certificates gnupg lsb-release

# 🐳 Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# 👤 Add current user to docker group
sudo usermod -aG docker $USER
newgrp docker

# ✅ Verify Docker installation
docker --version
```

### 🔹 Subtask 1.2: Install kubectl

Install the Kubernetes command-line tool:

```bash
# ⬇️ Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 🔓 Make it executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# ✅ Verify installation
kubectl version --client
```

### 🔹 Subtask 1.3: Install kind (Kubernetes in Docker)

Install kind to create a local Kubernetes cluster:

```bash
# ⬇️ Download and install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# ✅ Verify installation
kind version
```

### 🔹 Subtask 1.4: Install HashiCorp Vault

Download and install Vault:

```bash
# ⬇️ Download Vault
wget https://releases.hashicorp.com/vault/1.15.2/vault_1.15.2_linux_amd64.zip

# 📂 Extract and install
unzip vault_1.15.2_linux_amd64.zip
sudo mv vault /usr/local/bin/

# ✅ Verify installation
vault version

# 🗑️ Clean up
rm vault_1.15.2_linux_amd64.zip
```

---

## 📗 Task 2: Create Kubernetes Cluster and Configure Vault

### 🔹 Subtask 2.1: Create Kubernetes Cluster

Create a kind cluster configuration file:

```yaml
# 📝 Create kind cluster configuration file
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 8200
    hostPort: 8200
    protocol: TCP
EOF
```

Create the Kubernetes cluster:

```bash
# 🚀 Create cluster
kind create cluster --config=kind-config.yaml --name vault-lab

# ✅ Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

### 🔹 Subtask 2.2: Start Vault Server

Create a Vault configuration file:

```hcl
# ⚙️ Create Vault configuration file
cat > vault-config.hcl << 'EOF'
storage "file" {
  path = "/tmp/vault-data"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = true
}

ui = true
disable_mlock = true
EOF
```

Start Vault server in development mode for this lab:

```bash
# 📁 Create data directory
mkdir -p /tmp/vault-data

# ▶️ Start Vault server in background
vault server -config=vault-config.hcl &

# ⏳ Wait for Vault to start
sleep 5

# 🌐 Set Vault address
export VAULT_ADDR='http://127.0.0.1:8200'

# 🚀 Initialize Vault
vault operator init -key-shares=1 -key-threshold=1 > vault-keys.txt

# 🔑 Extract unseal key and root token
UNSEAL_KEY=$(grep 'Unseal Key 1:' vault-keys.txt | awk '{print $NF}')
ROOT_TOKEN=$(grep 'Initial Root Token:' vault-keys.txt | awk '{print $NF}')

# 🔓 Unseal Vault
vault operator unseal $UNSEAL_KEY

# 🔐 Login with root token
vault auth $ROOT_TOKEN

echo "Vault is now running and unsealed"
echo "Root Token: $ROOT_TOKEN"
```

### 🔹 Subtask 2.3: Configure Vault for Kubernetes Integration

Enable and configure the Kubernetes authentication method:

```bash
# 🔓 Enable Kubernetes auth method
vault auth enable kubernetes

# ℹ️ Get Kubernetes cluster information
KUBERNETES_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}')
KUBERNETES_CA_CERT=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d)

# 👤 Create a service account for Vault
kubectl create serviceaccount vault-auth

# 🔗 Create cluster role binding
kubectl create clusterrolebinding vault-auth-binding \
    --clusterrole=system:auth-delegator \
    --serviceaccount=default:vault-auth

# 🎫 Get the service account token
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
EOF

# ⏳ Wait for token to be created
sleep 5

TOKEN_REVIEW_JWT=$(kubectl get secret vault-auth-secret -o jsonpath='{.data.token}' | base64 -d)

# ⚙️ Configure Kubernetes auth method
vault write auth/kubernetes/config \
    token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
    kubernetes_host="$KUBERNETES_HOST" \
    kubernetes_ca_cert="$KUBERNETES_CA_CERT"

echo "Kubernetes authentication configured successfully"
```

---

## 📙 Task 3: Deploy Kubernetes Workloads with Vault Integration

### 🔹 Subtask 3.1: Create Vault Policies and Secrets

Create a policy for application access:

```bash
# 📜 Create application policy
vault policy write app-policy - << 'EOF'
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "secret/data/database/*" {
  capabilities = ["read"]
}
EOF

# 🔓 Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# 🔐 Create some sample secrets
vault kv put secret/myapp/config \
    username="appuser" \
    password="supersecret123" \
    api_key="abc123def456"

vault kv put secret/database/config \
    host="db.example.com" \
    port="5432" \
    username="dbuser" \
    password="dbpassword123"

echo "Secrets created successfully"
```

### 🔹 Subtask 3.2: Configure Kubernetes Role

Create a Kubernetes role that maps to the policy:

```bash
# 🎭 Create Kubernetes role
vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=default \
    policies=app-policy \
    ttl=1h

echo "Kubernetes role 'myapp' created successfully"
```

### 🔹 Subtask 3.3: Deploy Application with Vault Agent

Create a service account for the application:

```bash
# 👤 Create a service account for the application
kubectl create serviceaccount myapp-sa
```

Create a ConfigMap for Vault Agent configuration:

```yaml
# 📝 Create ConfigMap for Vault Agent configuration
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-agent-config
data:
  vault-agent.hcl: |
    exit_after_auth = false
    pid_file = "/tmp/pidfile"

    auto_auth {
      method "kubernetes" {
        mount_path = "auth/kubernetes"
        config = {
          role = "myapp"
        }
      }

      sink "file" {
        config = {
          path = "/tmp/vault-token"
        }
      }
    }

    vault {
      address = "http://host.docker.internal:8200"
    }

    template {
      source      = "/vault/config/app-config.tpl"
      destination = "/vault/secrets/app-config.json"
    }

    template {
      source      = "/vault/config/db-config.tpl"
      destination = "/vault/secrets/db-config.json"
    }

  app-config.tpl: |
    {
      "username": "{{ with secret "secret/data/myapp/config" }}{{ .Data.data.username }}{{ end }}",
      "password": "{{ with secret "secret/data/myapp/config" }}{{ .Data.data.password }}{{ end }}",
      "api_key": "{{ with secret "secret/data/myapp/config" }}{{ .Data.data.api_key }}{{ end }}"
    }

  db-config.tpl: |
    {
      "host": "{{ with secret "secret/data/database/config" }}{{ .Data.data.host }}{{ end }}",
      "port": "{{ with secret "secret/data/database/config" }}{{ .Data.data.port }}{{ end }}",
      "username": "{{ with secret "secret/data/database/config" }}{{ .Data.data.username }}{{ end }}",
      "password": "{{ with secret "secret/data/database/config" }}{{ .Data.data.password }}{{ end }}"
    }
EOF
```

### 🔹 Subtask 3.4: Deploy Application Pod with Vault Integration

Create a deployment that uses Vault Agent as a sidecar:

```yaml
# 🚀 Deploy application with Vault Agent sidecar
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: vault-agent
        image: hashicorp/vault:1.15.2
        command: ["vault", "agent", "-config=/vault/config/vault-agent.hcl"]
        volumeMounts:
        - name: vault-config
          mountPath: /vault/config
        - name: vault-secrets
          mountPath: /vault/secrets
        env:
        - name: VAULT_ADDR
          value: "http://host.docker.internal:8200"

      - name: myapp
        image: nginx:alpine
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            echo "=== Application Configuration ==="
            if [ -f /vault/secrets/app-config.json ]; then
              echo "App Config:"
              cat /vault/secrets/app-config.json | jq .
            else
              echo "App config not found"
            fi

            echo ""
            echo "=== Database Configuration ==="
            if [ -f /vault/secrets/db-config.json ]; then
              echo "DB Config:"
              cat /vault/secrets/db-config.json | jq .
            else
              echo "DB config not found"
            fi

            echo ""
            echo "Sleeping for 30 seconds..."
            sleep 30
          done
        volumeMounts:
        - name: vault-secrets
          mountPath: /vault/secrets
          readOnly: true

      volumes:
      - name: vault-config
        configMap:
          name: vault-agent-config
      - name: vault-secrets
        emptyDir: {}
EOF
```

Wait for the deployment and check the logs:

```bash
# ⏳ Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=myapp --timeout=60s

# 🏷️ Get pod name
POD_NAME=$(kubectl get pods -l app=myapp -o jsonpath='{.items[0].metadata.name}')

# 📜 Check application logs
echo "Checking application logs:"
kubectl logs $POD_NAME -c myapp

# 📜 Check Vault agent logs
echo "Checking Vault agent logs:"
kubectl logs $POD_NAME -c vault-agent
```

---

## 📕 Task 4: Use Vault for Kubernetes API Tokens

### 🔹 Subtask 4.1: Configure Vault Kubernetes Secrets Engine

Enable the Kubernetes secrets engine in Vault:

```bash
# 🔓 Enable Kubernetes secrets engine
vault secrets enable kubernetes

# ⚙️ Configure the Kubernetes secrets engine
vault write kubernetes/config \
    kubernetes_host="$KUBERNETES_HOST" \
    kubernetes_ca_cert="$KUBERNETES_CA_CERT" \
    service_account_jwt="$TOKEN_REVIEW_JWT"

echo "Kubernetes secrets engine configured"
```

### 🔹 Subtask 4.2: Create Kubernetes Role for Token Generation

Create a role that can generate service account tokens:

```bash
# 🎭 Create a role for generating service account tokens
vault write kubernetes/roles/token-generator \
    allowed_kubernetes_namespaces="default" \
    token_max_ttl="1h" \
    token_default_ttl="10m" \
    service_account_name="dynamic-sa"

echo "Token generator role created"
```

### 🔹 Subtask 4.3: Create Service Account for Dynamic Tokens

Create a service account that will be used for dynamic token generation:

```bash
# 👤 Create service account
kubectl create serviceaccount dynamic-sa

# 🎭 Create a cluster role with specific permissions
kubectl create clusterrole pod-reader \
    --verb=get,list,watch \
    --resource=pods

# 🔗 Bind the role to the service account
kubectl create clusterrolebinding dynamic-sa-binding \
    --clusterrole=pod-reader \
    --serviceaccount=default:dynamic-sa

echo "Dynamic service account created with pod-reader permissions"
```

### 🔹 Subtask 4.4: Test Dynamic Token Generation

Create a test script to generate and use dynamic tokens:

```bash
# 📝 Create dynamic-token test script
cat > test-dynamic-tokens.sh << 'EOF'
#!/bin/bash

echo "=== Testing Dynamic Kubernetes Token Generation ==="

# Generate a dynamic token
echo "Generating dynamic Kubernetes token..."
DYNAMIC_TOKEN_RESPONSE=$(vault write -format=json kubernetes/creds/token-generator)

if [ $? -eq 0 ]; then
    echo "Token generated successfully!"

    # Extract token and service account name
    DYNAMIC_TOKEN=$(echo $DYNAMIC_TOKEN_RESPONSE | jq -r '.data.service_account_token')
    SA_NAME=$(echo $DYNAMIC_TOKEN_RESPONSE | jq -r '.data.service_account_name')
    SA_NAMESPACE=$(echo $DYNAMIC_TOKEN_RESPONSE | jq -r '.data.service_account_namespace')

    echo "Service Account: $SA_NAME"
    echo "Namespace: $SA_NAMESPACE"
    echo "Token (first 20 chars): ${DYNAMIC_TOKEN:0:20}..."

    # Test the token by listing pods
    echo ""
    echo "Testing token by listing pods..."

    # Create a temporary kubeconfig
    TEMP_KUBECONFIG=$(mktemp)

    # Get cluster info
    CLUSTER_NAME=$(kubectl config current-context)
    CLUSTER_SERVER=$(kubectl config view -o jsonpath="{.clusters[?(@.name=='$CLUSTER_NAME')].cluster.server}")
    CLUSTER_CA=$(kubectl config view --raw -o jsonpath="{.clusters[?(@.name=='$CLUSTER_NAME')].cluster.certificate-authority-data}")

    # Create kubeconfig with dynamic token
    cat > $TEMP_KUBECONFIG << EOL
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: $CLUSTER_CA
    server: $CLUSTER_SERVER
  name: $CLUSTER_NAME
contexts:
- context:
    cluster: $CLUSTER_NAME
    user: dynamic-user
  name: dynamic-context
current-context: dynamic-context
users:
- name: dynamic-user
  user:
    token: $DYNAMIC_TOKEN
EOL

    # Test the token
    echo "Attempting to list pods with dynamic token..."
    kubectl --kubeconfig=$TEMP_KUBECONFIG get pods

    if [ $? -eq 0 ]; then
        echo "SUCCESS: Dynamic token works correctly!"
    else
        echo "ERROR: Dynamic token failed to authenticate"
    fi

    # Clean up
    rm $TEMP_KUBECONFIG

else
    echo "ERROR: Failed to generate dynamic token"
    echo $DYNAMIC_TOKEN_RESPONSE
fi

echo ""
echo "=== Token Generation Test Complete ==="
EOF

chmod +x test-dynamic-tokens.sh
./test-dynamic-tokens.sh
```

### 🔹 Subtask 4.5: Create Application Using Dynamic Tokens

Create a deployment that periodically requests new tokens from Vault:

```yaml
# 🚀 Deploy token-consumer application
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: token-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: token-consumer
  template:
    metadata:
      labels:
        app: token-consumer
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: token-consumer
        image: hashicorp/vault:1.15.2
        command: ["/bin/sh"]
        args:
        - -c
        - |
          # Authenticate with Vault using Kubernetes auth
          export VAULT_ADDR="http://host.docker.internal:8200"

          # Get service account token
          SA_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

          while true; do
            echo "=== $(date) ==="
            echo "Authenticating with Vault..."

            # Login to Vault
            VAULT_TOKEN=$(vault write -field=token auth/kubernetes/login \
              role=myapp \
              jwt=$SA_TOKEN)

            if [ $? -eq 0 ]; then
              export VAULT_TOKEN
              echo "Vault authentication successful"

              # Generate dynamic Kubernetes token
              echo "Requesting dynamic Kubernetes token..."
              vault write -format=json kubernetes/creds/token-generator | jq .

            else
              echo "Vault authentication failed"
            fi

            echo "Sleeping for 60 seconds..."
            sleep 60
          done
        env:
        - name: VAULT_ADDR
          value: "http://host.docker.internal:8200"
EOF
```

Check the token consumer logs:

```bash
# ⏳ Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=token-consumer --timeout=60s

# 📜 Get pod name and check logs
TOKEN_POD=$(kubectl get pods -l app=token-consumer -o jsonpath='{.items[0].metadata.name}')
kubectl logs $TOKEN_POD -f
```

---

## 📔 Task 5: Verification and Testing

### 🔹 Subtask 5.1: Verify Secret Injection

Check that secrets are properly injected into the application:

```bash
# 🏷️ Get the application pod
APP_POD=$(kubectl get pods -l app=myapp -o jsonpath='{.items[0].metadata.name}')

# 📂 Check if secrets files exist
echo "Checking secret files in application pod:"
kubectl exec $APP_POD -c myapp -- ls -la /vault/secrets/

# 👀 View the actual secret content
echo "Application configuration:"
kubectl exec $APP_POD -c myapp -- cat /vault/secrets/app-config.json | jq .

echo "Database configuration:"
kubectl exec $APP_POD -c myapp -- cat /vault/secrets/db-config.json | jq .
```

### 🔹 Subtask 5.2: Test Secret Rotation

Update secrets in Vault and verify they are rotated in the application:

```bash
# 🔄 Update application secrets
vault kv put secret/myapp/config \
    username="newappuser" \
    password="newsupersecret456" \
    api_key="xyz789ghi012"

# ⏳ Wait for Vault Agent to pick up changes
echo "Waiting for secret rotation (30 seconds)..."
sleep 30

# ✅ Check updated secrets
echo "Updated application configuration:"
kubectl exec $APP_POD -c myapp -- cat /vault/secrets/app-config.json | jq .
```

### 🔹 Subtask 5.3: Verify Token Lifecycle

Test the lifecycle of dynamically generated tokens:

```bash
# ⏱️ Generate a token and check its TTL
echo "Generating token and checking TTL..."
TOKEN_INFO=$(vault write -format=json kubernetes/creds/token-generator)
echo $TOKEN_INFO | jq .

# 📋 List active leases
echo "Active Kubernetes token leases:"
vault list sys/leases/lookup/kubernetes/creds/token-generator
```

### 🔹 Subtask 5.4: Security Verification

Verify that the integration follows security best practices:

```bash
# 🔍 Check service account permissions
echo "Service account permissions for myapp-sa:"
kubectl auth can-i --list --as=system:serviceaccount:default:myapp-sa

# 📖 Check Vault policies
echo "Vault policies for app-policy:"
vault policy read app-policy

# 🧪 Verify token permissions
echo "Testing token permissions..."
TEMP_TOKEN=$(vault write -field=service_account_token kubernetes/creds/token-generator)
kubectl auth can-i get pods --token=$TEMP_TOKEN
kubectl auth can-i create pods --token=$TEMP_TOKEN
```

---

## 🛡️ MITRE ATT&CK Mapping

| Technique ID | Technique Name | Relevance to This Lab |
|---|---|---|
| [T1552](https://attack.mitre.org/techniques/T1552/) | Unsecured Credentials | Vault removes the need to bake application/database credentials into container images or manifests |
| [T1552.001](https://attack.mitre.org/techniques/T1552/001/) | Credentials In Files | Secrets rendered by Vault Agent live only in an ephemeral `emptyDir`, and the initial `vault-keys.txt` unseal material must be handled and deleted carefully |
| [T1552.007](https://attack.mitre.org/techniques/T1552/007/) | Container API | Kubernetes service account tokens and the `system:auth-delegator` binding are the trust anchor between Vault and the cluster's API |
| [T1078](https://attack.mitre.org/techniques/T1078/) | Valid Accounts | Vault roles bind specific service accounts/namespaces to policies, and dynamically issued Kubernetes tokens are short-lived rather than long-standing credentials |
| [T1555](https://attack.mitre.org/techniques/T1555/) | Credentials from Password Stores | Centralizing app and database secrets in Vault's KV engine, gated by policy, reduces reliance on scattered static credential stores |

---

## 📚 Key Concepts

| Concept | Description |
|---|---|
| **Vault Agent (Sidecar Pattern)** | A Vault Agent container running alongside the application container, authenticating to Vault and rendering secrets to a shared volume |
| **Kubernetes Secrets Engine** | A Vault secrets engine that generates short-lived Kubernetes service account tokens on demand, instead of using static tokens |
| **Dynamic Secrets & Leases** | Credentials generated on-demand with a TTL and a revocable lease, rather than long-lived static secrets |
| **`system:auth-delegator` ClusterRole** | Grants Vault's service account permission to validate other service account tokens via the Kubernetes TokenReview API |
| **Consul Template Syntax** | The `{{ with secret "path" }}...{{ end }}` templating language Vault Agent uses to render secret values into files |
| **Kubernetes Auth Method** | Lets workloads authenticate to Vault using their own projected service account JWT rather than a static Vault token |
| **TTL (Time To Live)** | Controls how long a Vault token or dynamically generated Kubernetes token remains valid before requiring renewal |
| **Zero-Trust Secret Retrieval** | The pattern demonstrated in this lab: workloads authenticate themselves and pull only the secrets/tokens they're authorized for, on demand |

---

## 🔍 Troubleshooting Tips

<details>
<summary><strong>Common Issues and Solutions</strong></summary>

**Issue 1: Vault Agent cannot connect to Vault server**
- Ensure Vault server is running and accessible
- Check that `host.docker.internal` resolves correctly in the kind cluster
- Verify firewall settings allow connection on port 8200

**Issue 2: Kubernetes authentication fails**
- Verify service account token is correctly configured
- Check that the Kubernetes CA certificate is properly set
- Ensure the service account has proper RBAC permissions

**Issue 3: Secrets not appearing in pods**
- Check Vault Agent logs for authentication errors
- Verify template syntax in ConfigMap
- Ensure the shared volume is properly mounted

**Issue 4: Dynamic token generation fails**
- Verify Kubernetes secrets engine configuration
- Check that the target service account exists
- Ensure proper RBAC permissions for token generation

**Debugging Commands**

```bash
# 🔍 Check Vault status
vault status

# 📋 List authentication methods
vault auth list

# ⚙️ Check Kubernetes auth configuration
vault read auth/kubernetes/config

# 📋 List secrets engines
vault secrets list

# 📜 Check pod logs
kubectl logs <pod-name> -c <container-name>

# 🔍 Describe pod for troubleshooting
kubectl describe pod <pod-name>
```

</details>

---

## 🧹 Cleanup

To clean up the lab environment:

```bash
# 🗑️ Delete Kubernetes resources
kubectl delete deployment myapp-deployment token-consumer
kubectl delete serviceaccount myapp-sa dynamic-sa vault-auth
kubectl delete configmap vault-agent-config
kubectl delete clusterrolebinding vault-auth-binding dynamic-sa-binding
kubectl delete clusterrole pod-reader
kubectl delete secret vault-auth-secret

# ⏹️ Stop Vault server
pkill vault

# 🗑️ Delete kind cluster
kind delete cluster --name vault-lab

# 🗑️ Clean up files
rm -f vault-config.hcl vault-keys.txt kind-config.yaml test-dynamic-tokens.sh
rm -rf /tmp/vault-data
```

---

## ✅ Conclusion

In this comprehensive lab, you have successfully:

### 🏆 Key Accomplishments

- Deployed a complete Vault and Kubernetes integration on a single Linux machine using open-source tools
- Implemented dynamic secret injection using Vault Agent as a sidecar container pattern
- Configured Kubernetes authentication with Vault for secure, automated authentication
- Created and managed dynamic Kubernetes service account tokens through Vault's Kubernetes secrets engine
- Established secure communication patterns between Kubernetes workloads and Vault
- Implemented secret rotation capabilities that automatically update application configurations

### 🌍 Real-World Applications

This integration pattern is crucial in modern cloud-native environments where:

- **Security is paramount:** Secrets are never stored in container images or configuration files
- **Automation is essential:** Applications can authenticate and retrieve secrets without manual intervention
- **Compliance is required:** All secret access is logged and auditable through Vault
- **Scalability matters:** The pattern works across multiple applications and environments

The skills you've developed in this lab directly apply to production environments where organizations need to manage secrets securely across containerized applications. You now understand how to implement zero-trust security models where applications authenticate themselves and retrieve only the secrets they need, when they need them, with automatic rotation and revocation capabilities.

This foundation prepares you for advanced topics like multi-cluster secret management, automated certificate management, and integration with service mesh technologies for comprehensive security in cloud-native architectures.

---

<div align="center">

![Al Nafi](https://img.shields.io/badge/Al_Nafi-Cybersecurity_Training-blueviolet?style=for-the-badge)

</div>
