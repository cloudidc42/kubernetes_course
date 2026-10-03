# Part 54: HashiCorp Vault บน Kubernetes

## บทนำ

HashiCorp Vault เป็น secrets management solution ระดับ enterprise ที่ให้ความสามารถ:
- เก็บและจัดการ secrets อย่างปลอดภัย
- Dynamic secrets (สร้าง credentials ชั่วคราว)
- Encryption as a Service
- Identity-based access control
- Audit logging ครบถ้วน

---

## 54.1 HashiCorp Vault Architecture

### องค์ประกอบหลัก

```
┌─────────────────────────────────────────────────────────┐
│                    HashiCorp Vault                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Auth Methods│  │  Secret      │  │  Audit       │  │
│  │  - Kubernetes│  │  Engines     │  │  Backends    │  │
│  │  - JWT/OIDC  │  │  - KV v2     │  │  - File      │  │
│  │  - AWS       │  │  - Database  │  │  - Syslog    │  │
│  │  - Userpass  │  │  - PKI       │  │  - Socket    │  │
│  └──────────────┘  │  - Transit   │  └──────────────┘  │
│                    │  - AWS       │                     │
│  ┌──────────────┐  └──────────────┘  ┌──────────────┐  │
│  │  Policies    │                    │  Storage     │  │
│  │  - HCL based │                    │  - Consul    │  │
│  │  - RBAC      │                    │  - etcd      │  │
│  └──────────────┘                    │  - File      │  │
│                                      │  - Raft      │  │
└──────────────────────────────────────┴──────────────────┘
```

### Vault Concepts

```
Seal/Unseal: Vault เริ่มต้นใน "sealed" state
ต้องใช้ unseal keys (Shamir's Secret Sharing) เพื่อ unseal

Tokens: ใช้ authenticate กับ Vault
- Root token: สิทธิ์สูงสุด (ใช้เฉพาะตอน setup)
- Service token: ใช้งานประจำวัน มี TTL
- Periodic token: ต่ออายุได้

Leases: secrets มีอายุ (TTL)
เมื่อหมดอายุ Vault จะ revoke

Paths: ทุก operation ใน Vault ใช้ path
- auth/kubernetes/login
- secret/data/myapp/production
- database/creds/my-role
```

---

## 54.2 ติดตั้ง Vault บน Kubernetes

### วิธีที่ 1: Helm Chart (แนะนำ)

```bash
# เพิ่ม Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# สร้าง namespace
kubectl create namespace vault

# ติดตั้ง Vault (development mode)
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=root" \
  --set "injector.enabled=true"

# ตรวจสอบ
kubectl get pods -n vault
kubectl get svc -n vault
```

### วิธีที่ 2: Production Setup พร้อม Raft Storage

```yaml
# vault-values.yaml
server:
  # ใช้ Raft (Integrated Storage) แทน Consul
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        cluster_name = "vault-integrated-storage"
        storage "raft" {
          path    = "/vault/data"
          node_id = "{{ env "HOSTNAME" }}"
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
          }
        }
        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
        }
        service_registration "kubernetes" {}
  
  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: standard
  
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 256Mi
      cpu: 250m

# Vault Agent Injector
injector:
  enabled: true
  replicas: 1
  resources:
    requests:
      memory: 64Mi
      cpu: 50m
    limits:
      memory: 128Mi
      cpu: 100m

# UI (optional)
ui:
  enabled: true
  serviceType: ClusterIP
```

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --values vault-values.yaml

kubectl get pods -n vault -w
```

### Initialize และ Unseal Vault

```bash
# Initialize Vault (ทำครั้งเดียว)
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init-keys.json

# IMPORTANT: เก็บ vault-init-keys.json ไว้ในที่ปลอดภัย!
cat vault-init-keys.json | jq '.'

# Extract keys
UNSEAL_KEY_1=$(cat vault-init-keys.json | jq -r '.unseal_keys_b64[0]')
UNSEAL_KEY_2=$(cat vault-init-keys.json | jq -r '.unseal_keys_b64[1]')
UNSEAL_KEY_3=$(cat vault-init-keys.json | jq -r '.unseal_keys_b64[2]')
ROOT_TOKEN=$(cat vault-init-keys.json | jq -r '.root_token')

# Unseal (ต้องใช้ 3 keys จาก 5 keys)
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_1
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_2
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_3

# ตรวจสอบสถานะ
kubectl exec -n vault vault-0 -- vault status
```

---

## 54.3 Configure Vault สำหรับ Kubernetes

### Enable Kubernetes Auth Method

```bash
# Login ด้วย root token
kubectl exec -n vault vault-0 -- vault login $ROOT_TOKEN

# Enable Kubernetes auth
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# Configure Kubernetes auth
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local:443"

# Enable KV secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2

# สร้าง secrets สำหรับ test
kubectl exec -n vault vault-0 -- vault kv put secret/myapp/production \
  DB_HOST=postgres.production.svc.cluster.local \
  DB_PORT=5432 \
  DB_NAME=myapp_db \
  DB_USER=myapp_user \
  DB_PASSWORD='VaultSecure@2024' \
  API_KEY='vault-api-key-xyz789'

# ตรวจสอบ
kubectl exec -n vault vault-0 -- vault kv get secret/myapp/production
```

### สร้าง Policy

```bash
# สร้าง policy สำหรับ app
kubectl exec -n vault vault-0 -- vault policy write myapp-policy - <<EOF
# อ่าน secrets สำหรับ myapp
path "secret/data/myapp/production" {
  capabilities = ["read"]
}

path "secret/data/myapp/staging" {
  capabilities = ["read"]
}

# ป้องกันการลบ
path "secret/delete/myapp/*" {
  capabilities = ["deny"]
}
EOF

# ตรวจสอบ policy
kubectl exec -n vault vault-0 -- vault policy read myapp-policy
```

### สร้าง Kubernetes Role

```bash
# สร้าง Role ที่ map ระหว่าง Kubernetes ServiceAccount กับ Vault Policy
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=production,staging \
  policies=myapp-policy \
  ttl=1h

# ตรวจสอบ
kubectl exec -n vault vault-0 -- vault read auth/kubernetes/role/myapp-role
```

---

## 54.4 Vault Agent Injector

Vault Agent Injector ใช้ Kubernetes Mutating Webhook เพื่อ inject sidecar container ที่จะ authenticate กับ Vault และดึง secrets ให้อัตโนมัติ

### วิธีการทำงาน

```
Pod สร้าง
    ↓
MutatingWebhook intercepted
    ↓
Vault Agent Injector inject sidecar
    ↓
Sidecar authenticate กับ Vault (kubernetes auth)
    ↓
ดึง secrets จาก Vault
    ↓
เขียน secrets เป็น files ใน shared volume
    ↓
Application container อ่าน files
```

### Annotations สำหรับ Injection

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        # Enable vault agent injection
        vault.hashicorp.com/agent-inject: "true"
        
        # Vault role สำหรับ authenticate
        vault.hashicorp.com/role: "myapp-role"
        
        # Inject secret เป็น file
        vault.hashicorp.com/agent-inject-secret-config.env: "secret/data/myapp/production"
        
        # Template สำหรับ format ของ secret file
        vault.hashicorp.com/agent-inject-template-config.env: |
          {{- with secret "secret/data/myapp/production" -}}
          export DB_HOST="{{ .Data.data.DB_HOST }}"
          export DB_PORT="{{ .Data.data.DB_PORT }}"
          export DB_NAME="{{ .Data.data.DB_NAME }}"
          export DB_USER="{{ .Data.data.DB_USER }}"
          export DB_PASSWORD="{{ .Data.data.DB_PASSWORD }}"
          export API_KEY="{{ .Data.data.API_KEY }}"
          {{- end }}
        
        # Vault address
        vault.hashicorp.com/agent-inject-vault-addr: "http://vault.vault.svc.cluster.local:8200"
        
        # Pre-populate ก่อน app container เริ่ม
        vault.hashicorp.com/agent-pre-populate-only: "false"
        
        # ขนาด memory
        vault.hashicorp.com/agent-limits-memory: "128Mi"
        vault.hashicorp.com/agent-requests-memory: "64Mi"
    
    spec:
      serviceAccountName: myapp-sa
      
      containers:
      - name: myapp
        image: myapp:1.0.0
        command: ["/bin/sh", "-c"]
        args:
        # Source secrets ก่อน run app
        - "source /vault/secrets/config.env && exec /app/start.sh"
        
        # Secrets file จะอยู่ที่ /vault/secrets/
        # ไม่ต้อง mount เอง - vault agent inject ให้อัตโนมัติ
        
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
```

### ServiceAccount สำหรับ Vault Auth

```yaml
# ServiceAccount สำหรับ app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
automountServiceAccountToken: true  # จำเป็นสำหรับ Vault auth
---
# ClusterRoleBinding สำหรับ Vault ให้ verify tokens
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-token-reviewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-sa
  namespace: vault
```

---

## 54.5 Dynamic Secrets

Dynamic Secrets คือ feature ที่ Vault สร้าง credentials ชั่วคราวให้สำหรับแต่ละ request แทนที่จะแชร์ credentials เดิม

### Database Dynamic Secrets

```bash
# Enable database secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable database

# Configure PostgreSQL connection
kubectl exec -n vault vault-0 -- vault write database/config/mypostgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp-db-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.production:5432/mydb?sslmode=disable" \
  username="vault_admin" \
  password="VaultAdminPassword" \
  verify_connection=false

# สร้าง Role สำหรับ generate credentials
kubectl exec -n vault vault-0 -- vault write database/roles/myapp-db-role \
  db_name=mypostgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# ทดสอบ generate credentials
kubectl exec -n vault vault-0 -- vault read database/creds/myapp-db-role
# ได้ credentials ชั่วคราว!
```

### ใช้ Dynamic Database Credentials ใน Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-dynamic
  namespace: production
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-role"
        
        # Dynamic DB credentials
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/myapp-db-role"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/myapp-db-role" -}}
          export DB_USER="{{ .Data.username }}"
          export DB_PASSWORD="{{ .Data.password }}"
          {{- end }}
        
        # Auto-renew secrets before expiry
        vault.hashicorp.com/agent-cache-enable: "true"
```

### AWS Dynamic Credentials

```bash
# Enable AWS secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable aws

# Configure AWS
kubectl exec -n vault vault-0 -- vault write aws/config/root \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  region=ap-southeast-1

# สร้าง Role
kubectl exec -n vault vault-0 -- vault write aws/roles/myapp-aws-role \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": ["arn:aws:s3:::my-app-bucket/*"]
  }]
}
EOF

# Generate temporary AWS credentials
kubectl exec -n vault vault-0 -- vault read aws/creds/myapp-aws-role
```

---

## 54.6 Vault PKI: Certificate Management

```bash
# Enable PKI secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable pki

# Configure Root CA
kubectl exec -n vault vault-0 -- vault write pki/root/generate/internal \
  common_name="My Company CA" \
  ttl=87600h  # 10 years

# สร้าง Role สำหรับ issue certificates
kubectl exec -n vault vault-0 -- vault write pki/roles/myapp-cert \
  allowed_domains=myapp.example.com,*.myapp.svc.cluster.local \
  allow_subdomains=true \
  max_ttl=72h  # 3 days

# Issue certificate
kubectl exec -n vault vault-0 -- vault write pki/issue/myapp-cert \
  common_name=myapp.example.com \
  alt_names=myapp.production.svc.cluster.local \
  ttl=24h
```

---

## 54.7 Workshop: Vault + Kubernetes Integration

### สถานการณ์

เราจะ setup:
1. Vault development mode
2. Enable Kubernetes auth
3. สร้าง dynamic database credentials
4. Deploy application ที่ใช้ Vault Agent Injector
5. ทดสอบ secret rotation

### Step 1: Setup

```bash
kubectl create namespace vault
kubectl create namespace production

# ติดตั้ง Vault dev mode
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=root" \
  --set "injector.enabled=true" \
  --wait

kubectl get pods -n vault
```

### Step 2: Configure Vault

```bash
# Port forward เพื่อ configure
kubectl port-forward svc/vault -n vault 8200:8200 &

# Set Vault address
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# หรือ exec โดยตรง
kubectl exec -n vault vault-0 -- /bin/sh
```

```bash
# Enable KV v2
vault secrets enable -path=secret kv-v2

# เพิ่ม secrets
vault kv put secret/workshop/app \
  DB_HOST="postgres.production.svc.cluster.local" \
  DB_NAME="workshop_db" \
  DB_USER="workshop_user" \
  DB_PASSWORD="Workshop@Vault2024" \
  API_SECRET="vault-workshop-secret-key"

# ตรวจสอบ
vault kv get secret/workshop/app

# Enable Kubernetes auth
vault auth enable kubernetes

# Get Kubernetes CA certificate
K8S_CA_CERT=$(kubectl get configmap kube-root-ca.crt \
  -n default \
  -o jsonpath='{.data.ca\.crt}' | base64 -w 0)

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local:443" \
  kubernetes_ca_cert="$(kubectl get configmap kube-root-ca.crt -n default -o jsonpath='{.data.ca\.crt}')"

# สร้าง Policy
vault policy write workshop-policy - <<EOF
path "secret/data/workshop/app" {
  capabilities = ["read"]
}

path "secret/data/workshop/app/*" {
  capabilities = ["read"]
}
EOF

# สร้าง Role
vault write auth/kubernetes/role/workshop-role \
  bound_service_account_names=workshop-sa \
  bound_service_account_namespaces=production \
  policies=workshop-policy \
  ttl=1h

echo "Vault configuration complete!"
```

### Step 3: สร้าง ServiceAccount

```yaml
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workshop-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: workshop-sa-tokenreview
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: workshop-sa
  namespace: production
```

```bash
kubectl apply -f service-account.yaml
```

### Step 4: Deploy Application กับ Vault Agent Injection

```yaml
# vault-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-workshop-app
  namespace: production
  labels:
    app: vault-workshop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vault-workshop-app
  template:
    metadata:
      labels:
        app: vault-workshop-app
      annotations:
        # Enable Vault Agent injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/role: "workshop-role"
        
        # Vault address
        vault.hashicorp.com/agent-inject-vault-addr: "http://vault.vault.svc.cluster.local:8200"
        
        # Inject secret เป็น environment file
        vault.hashicorp.com/agent-inject-secret-app-secrets.env: "secret/data/workshop/app"
        vault.hashicorp.com/agent-inject-template-app-secrets.env: |
          {{- with secret "secret/data/workshop/app" -}}
          export DB_HOST="{{ .Data.data.DB_HOST }}"
          export DB_NAME="{{ .Data.data.DB_NAME }}"
          export DB_USER="{{ .Data.data.DB_USER }}"
          export DB_PASSWORD="{{ .Data.data.DB_PASSWORD }}"
          export API_SECRET="{{ .Data.data.API_SECRET }}"
          {{- end }}
        
        # Inject JSON ด้วย
        vault.hashicorp.com/agent-inject-secret-config.json: "secret/data/workshop/app"
        vault.hashicorp.com/agent-inject-template-config.json: |
          {{- with secret "secret/data/workshop/app" -}}
          {
            "database": {
              "host": "{{ .Data.data.DB_HOST }}",
              "name": "{{ .Data.data.DB_NAME }}",
              "user": "{{ .Data.data.DB_USER }}",
              "password": "{{ .Data.data.DB_PASSWORD }}"
            },
            "api": {
              "secret": "{{ .Data.data.API_SECRET }}"
            }
          }
          {{- end }}
        
        # Resource limits สำหรับ vault-agent
        vault.hashicorp.com/agent-limits-memory: "64Mi"
        vault.hashicorp.com/agent-requests-memory: "32Mi"
        vault.hashicorp.com/agent-limits-cpu: "100m"
        vault.hashicorp.com/agent-requests-cpu: "50m"
    
    spec:
      serviceAccountName: workshop-sa
      
      containers:
      - name: app
        image: nginx:1.25-alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "Starting application..."
          
          # รอ vault agent inject secrets
          until [ -f /vault/secrets/app-secrets.env ]; do
            echo "Waiting for secrets..."
            sleep 1
          done
          
          # Load secrets
          source /vault/secrets/app-secrets.env
          
          echo "DB_HOST: $DB_HOST"
          echo "DB_USER: $DB_USER"
          echo "DB_NAME: $DB_NAME"
          echo "Secrets loaded! Starting nginx..."
          
          exec nginx -g 'daemon off;'
        
        ports:
        - containerPort: 80
        
        # Secrets จะอยู่ที่ /vault/secrets/ - inject โดย vault-agent
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
```

```bash
kubectl apply -f vault-app-deployment.yaml
kubectl get pods -n production -w
```

### Step 5: ตรวจสอบ Vault Agent Injection

```bash
# ดู pods (จะเห็น 2 containers: vault-agent และ app)
kubectl get pods -n production
# NAME                                  READY   STATUS    RESTARTS   AGE
# vault-workshop-app-xxxx-yyyy         2/2     Running   0          30s

# ดู vault-agent container logs
POD=$(kubectl get pod -l app=vault-workshop-app -n production -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n production -c vault-agent

# ดู app logs (จะเห็น secrets loaded)
kubectl logs $POD -n production -c app

# ดู secrets files ที่ inject มา
kubectl exec -it $POD -n production -c app -- ls -la /vault/secrets/

kubectl exec -it $POD -n production -c app -- cat /vault/secrets/app-secrets.env
# export DB_HOST="postgres.production.svc.cluster.local"
# export DB_NAME="workshop_db"
# ...

kubectl exec -it $POD -n production -c app -- cat /vault/secrets/config.json
```

### Step 6: ทดสอบ Secret Update

```bash
# อัพเดต secret ใน Vault
vault kv put secret/workshop/app \
  DB_HOST="postgres-v2.production.svc.cluster.local" \
  DB_NAME="workshop_db_v2" \
  DB_USER="workshop_user_v2" \
  DB_PASSWORD="Workshop@Vault2024Updated" \
  API_SECRET="vault-workshop-secret-key-v2"

# Vault agent จะ refresh secrets อัตโนมัติ (ตาม TTL)
# รอสักครู่แล้วตรวจสอบ
sleep 30

kubectl exec -it $POD -n production -c app -- cat /vault/secrets/app-secrets.env
# ค่าจะยังไม่เปลี่ยน จน TTL หมด หรือ restart pod
```

### Step 7: Vault UI

```bash
# เข้าถึง Vault UI
kubectl port-forward svc/vault-ui -n vault 8200:8200 &

# เปิด browser: http://localhost:8200
# Login ด้วย Method: Token, Token: root

# ดู secrets, policies, auth methods ผ่าน UI
```

### Step 8: Monitoring

```bash
# ดู Vault health
kubectl exec -n vault vault-0 -- vault status

# ดู auth mounts
kubectl exec -n vault vault-0 -- vault auth list

# ดู secret engines
kubectl exec -n vault vault-0 -- vault secrets list

# ดู policies
kubectl exec -n vault vault-0 -- vault policy list
kubectl exec -n vault vault-0 -- vault policy read workshop-policy

# ดู token info
kubectl exec -n vault vault-0 -- vault token lookup
```

### Step 9: Cleanup

```bash
kill %1 2>/dev/null  # ยกเลิก port-forward
kubectl delete namespace production
helm uninstall vault -n vault
kubectl delete namespace vault
echo "Workshop cleanup complete!"
```

---

## 54.8 Vault Best Practices สำหรับ Kubernetes

### 1. ใช้ Kubernetes Auth Method แทน Static Tokens

```bash
# ไม่ดี: ใช้ static root token ใน pod
env:
- name: VAULT_TOKEN
  value: "root"  # อย่าทำแบบนี้!

# ดี: ใช้ Kubernetes Service Account
serviceAccountName: myapp-sa
# + vault agent injection
```

### 2. ใช้ Least Privilege Policies

```hcl
# เฉพาะ path ที่จำเป็น
path "secret/data/myapp/production" {
  capabilities = ["read"]
}

# ห้าม list หรือ delete
path "secret/data/myapp/*" {
  capabilities = ["deny"]
}
```

### 3. ตั้ง TTL ให้เหมาะสม

```bash
# สั้นสำหรับ dynamic credentials
vault write database/roles/myapp-role \
  default_ttl="15m" \   # ต่ำ = ปลอดภัยกว่า
  max_ttl="1h"

# สำหรับ static secrets
vault write auth/kubernetes/role/myapp \
  ttl="1h" \            # token TTL
  max_ttl="24h"
```

### 4. Enable Audit Logging

```bash
# Enable file audit
kubectl exec -n vault vault-0 -- vault audit enable file file_path=/vault/logs/vault_audit.log

# ดู audit logs
kubectl exec -n vault vault-0 -- tail -f /vault/logs/vault_audit.log | jq '.'
```

### 5. HA Setup และ Auto-unseal

```yaml
# ใช้ AWS KMS auto-unseal
server:
  extraEnvironmentVars:
    VAULT_SEAL_TYPE: awskms
    VAULT_AWSKMS_SEAL_KEY_ID: "arn:aws:kms:region:account:key/key-id"
  ha:
    enabled: true
    replicas: 3
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Vault Architecture**: Seal/Unseal, Auth Methods, Secret Engines, Policies
2. **ติดตั้ง Vault**: Dev mode และ Production mode บน Kubernetes
3. **Vault Agent Injector**: inject secrets เป็น files ใน pods
4. **Dynamic Secrets**: สร้าง credentials ชั่วคราวสำหรับ database และ AWS

**Key Takeaways:**
- Dynamic secrets > static secrets (rotate อัตโนมัติ)
- ใช้ Kubernetes Auth Method เพื่อ avoid static credentials
- Vault Agent Injector ทำให้ app ไม่ต้อง aware Vault
- Enable Audit Logging สำหรับ compliance

---

**ต่อไป**: Part 55 - RBAC (Role-Based Access Control)

---

## Vault Policies - ละเอียด

Vault Policies เป็นกลไกหลักในการควบคุม access โดยใช้ HCL (HashiCorp Configuration Language)

### Anatomy ของ Vault Policy

```hcl
# ไวยากรณ์พื้นฐาน
path "<secret-path>" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Capabilities ที่มี:
# create  - สร้างใหม่
# read    - อ่าน
# update  - แก้ไข
# delete  - ลบ
# list    - แสดงรายการ
# sudo    - bypass root-protected paths
# deny    - ปฏิเสธ (override ทุกอย่าง)
```

### Policy สำหรับ Application แต่ละประเภท

```hcl
# policy: app-readonly
# สำหรับ applications ที่ต้องการแค่อ่าน secrets

# อ่าน secrets ของตัวเอง
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

# อ่าน certificate
path "pki/issue/myapp" {
  capabilities = ["create", "update"]
}

# ดู token ของตัวเอง
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

# Renew token ของตัวเอง
path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

```hcl
# policy: devops-admin
# สำหรับ DevOps team

# จัดการ secrets ใน production namespace
path "secret/data/production/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# ดูประวัติ (metadata)
path "secret/metadata/production/*" {
  capabilities = ["read", "list"]
}

# จัดการ PKI certificates
path "pki/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# จัดการ database roles
path "database/roles/*" {
  capabilities = ["read", "list"]
}

# ขอ database credentials
path "database/creds/*" {
  capabilities = ["read"]
}

# ดู auth methods
path "sys/auth" {
  capabilities = ["read"]
}
```

```hcl
# policy: ci-cd-deploy
# สำหรับ CI/CD pipeline

# อ่าน secrets สำหรับ deploy
path "secret/data/deploy/*" {
  capabilities = ["read"]
}

# สร้าง short-lived tokens สำหรับ child processes
path "auth/token/create" {
  capabilities = ["create", "update"]
}

# ดู database credentials (read-only)
path "database/creds/readonly" {
  capabilities = ["read"]
}

# ดู status ของ Vault
path "sys/health" {
  capabilities = ["read", "sudo"]
}
```

```hcl
# policy: kubernetes-app-policy
# Template policy สำหรับ Kubernetes workloads

# ใช้ path template ด้วย {{identity.entity.aliases.auth_kubernetes_*.metadata.service_account_namespace}}
path "secret/data/{{identity.entity.aliases.auth_kubernetes_cluster1.metadata.service_account_namespace}}/{{identity.entity.aliases.auth_kubernetes_cluster1.metadata.service_account_name}}/*" {
  capabilities = ["read"]
}

# หรือใช้ wildcard อย่างระมัดระวัง
path "secret/data/k8s/+/+/*" {
  capabilities = ["read"]
}
```

### สร้างและจัดการ Policies

```bash
# Apply policy จาก file
vault policy write app-readonly /tmp/app-readonly.hcl

# List policies
vault policy list

# Read policy
vault policy read app-readonly

# ลบ policy
vault policy delete old-policy

# Test policy ด้วย token
vault token create \
  -policy="app-readonly" \
  -ttl="1h" \
  -display-name="test-token"

# Verify capabilities ของ token
vault token capabilities \
  secret/data/myapp/config \
  <token-id>
```

---

## Vault Auth Methods - ละเอียด

### 1. Kubernetes Auth Method - ละเอียด

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# ดึง Kubernetes host และ CA cert
K8S_HOST=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')
K8S_CA=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d)

# ดึง Service Account token สำหรับ Vault
kubectl create serviceaccount vault-auth -n vault
kubectl apply -f - << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-auth-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-auth
  namespace: vault
EOF

# ดึง token
SA_TOKEN=$(kubectl create token vault-auth -n vault --duration=8760h)

# Configure Vault
vault write auth/kubernetes/config \
  kubernetes_host="${K8S_HOST}" \
  kubernetes_ca_cert="${K8S_CA}" \
  token_reviewer_jwt="${SA_TOKEN}"

# สร้าง role สำหรับ application
vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app-sa \
  bound_service_account_namespaces=production \
  policies=app-readonly \
  ttl=1h \
  max_ttl=24h

# ทดสอบ login จาก Pod
kubectl exec -n production my-app-pod -- \
  vault write auth/kubernetes/login \
    role=my-app \
    jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

### 2. AWS Auth Method

```bash
# Enable AWS auth
vault auth enable aws

# Configure AWS credentials (หรือใช้ IAM role)
vault write auth/aws/config/client \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  region=ap-southeast-1

# หรือใช้ instance IAM role (ไม่ต้องใส่ credentials)
vault write auth/aws/config/client \
  iam_server_id_header_value=vault.example.com

# สร้าง role สำหรับ EC2 instances
vault write auth/aws/role/my-ec2-role \
  auth_type=ec2 \
  bound_ami_id=ami-12345678 \
  bound_account_id=123456789012 \
  bound_region=ap-southeast-1 \
  policies=app-readonly \
  max_ttl=1h

# สร้าง role สำหรับ IAM (EKS/Lambda)
vault write auth/aws/role/my-iam-role \
  auth_type=iam \
  bound_iam_principal_arn="arn:aws:iam::123456789012:role/my-k8s-role" \
  policies=app-readonly \
  ttl=30m

# Login จาก EC2 instance
vault login \
  -method=aws \
  role=my-ec2-role
```

### 3. LDAP Auth Method

```bash
# Enable LDAP auth
vault auth enable ldap

# Configure LDAP connection
vault write auth/ldap/config \
  url="ldap://ldap.example.com" \
  starttls=true \
  tls_min_version=tls12 \
  binddn="cn=vault-bind,dc=example,dc=com" \
  bindpass="bind-password" \
  userdn="ou=users,dc=example,dc=com" \
  userattr="uid" \
  groupdn="ou=groups,dc=example,dc=com" \
  groupattr="cn" \
  groupfilter="(|(memberUid={{.Username}})(member={{.UserDN}})(uniqueMember={{.UserDN}}))"

# Map LDAP groups ไป Vault policies
vault write auth/ldap/groups/devops \
  policies=devops-admin

vault write auth/ldap/groups/developers \
  policies=app-readonly

vault write auth/ldap/groups/ci-cd \
  policies=ci-cd-deploy

# Login ด้วย LDAP
vault login \
  -method=ldap \
  username=john.doe

# ดู LDAP config
vault read auth/ldap/config
vault list auth/ldap/groups
```

### 4. AppRole Auth Method (สำหรับ Services)

```bash
# Enable AppRole
vault auth enable approle

# สร้าง role สำหรับ service
vault write auth/approle/role/my-service \
  secret_id_ttl=10m \       # SecretID หมดอายุ 10 นาที
  token_num_uses=10 \        # ใช้ token ได้ 10 ครั้ง
  token_ttl=20m \
  token_max_ttl=30m \
  secret_id_num_uses=40 \
  policies=app-readonly

# ดึง RoleID (เป็น public)
vault read auth/approle/role/my-service/role-id

# ดึง SecretID (เป็น secret)
vault write -f auth/approle/role/my-service/secret-id

# Login
vault write auth/approle/login \
  role_id=<role-id> \
  secret_id=<secret-id>
```

---

## Dynamic Database Credentials - PostgreSQL

### Setup Database Engine

```bash
# Enable Database Secret Engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/my-postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly,readwrite,admin" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.default.svc.cluster.local:5432/mydb?sslmode=require" \
  username="vault-admin" \
  password="vault-admin-password" \
  password_authentication="scram-sha-256"

# ทดสอบ connection
vault write database/config/my-postgresql -output-curl-string
```

### สร้าง Database Roles

```bash
# Role: readonly
vault write database/roles/readonly \
  db_name=my-postgresql \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
    GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO \"{{name}}\";
  " \
  revocation_statements="
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\";
    DROP ROLE IF EXISTS \"{{name}}\";
  " \
  default_ttl="1h" \
  max_ttl="24h"

# Role: readwrite
vault write database/roles/readwrite \
  db_name=my-postgresql \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
    GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO \"{{name}}\";
  " \
  revocation_statements="
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\";
    DROP ROLE IF EXISTS \"{{name}}\";
  " \
  default_ttl="30m" \
  max_ttl="1h"

# Role: admin (short-lived)
vault write database/roles/admin \
  db_name=my-postgresql \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' CREATEROLE;
    GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
  " \
  revocation_statements="
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\";
    DROP ROLE IF EXISTS \"{{name}}\";
  " \
  default_ttl="15m" \
  max_ttl="30m"
```

### ใช้ Dynamic Credentials

```bash
# ขอ credentials ชั่วคราว
vault read database/creds/readonly

# Output:
# Key                Value
# ---                -----
# lease_id           database/creds/readonly/abc123
# lease_duration     1h
# lease_renewable    true
# password           A1a-randompassword123
# username           v-kubernetes-readonly-abc123

# ต่ออายุ lease
vault lease renew database/creds/readonly/abc123

# ยกเลิก credential ก่อนกำหนด
vault lease revoke database/creds/readonly/abc123

# ยกเลิก credentials ทั้งหมดที่เกี่ยวข้องกับ role นี้
vault lease revoke -prefix database/creds/readonly
```

### Application ใช้ Dynamic Database Credentials

```yaml
# pod-with-vault-db.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-db
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-app"
    # Request database credentials
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/readonly"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/readonly" -}}
      export DB_USER="{{ .Data.username }}"
      export DB_PASSWORD="{{ .Data.password }}"
      export DB_HOST="postgres.default.svc.cluster.local"
      export DB_PORT="5432"
      export DB_NAME="mydb"
      {{- end -}}
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: my-app:latest
    command: ["sh", "-c", "source /vault/secrets/db-creds && exec /app/server"]
    env:
    - name: APP_PORT
      value: "8080"
```

### MySQL/MariaDB Dynamic Credentials

```bash
# Configure MySQL
vault write database/config/my-mysql \
  plugin_name=mysql-rds-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(mysql.default.svc.cluster.local:3306)/" \
  username="vault-admin" \
  password="admin-password" \
  allowed_roles="mysql-readonly,mysql-readwrite"

vault write database/roles/mysql-readonly \
  db_name=my-mysql \
  creation_statements="
    CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';
    GRANT SELECT ON *.* TO '{{name}}'@'%';
  " \
  revocation_statements="
    DROP USER IF EXISTS '{{name}}'@'%';
  " \
  default_ttl="1h" \
  max_ttl="24h"
```

---

## Vault Agent vs Vault CSI Provider

### Vault Agent - ละเอียด

Vault Agent เป็น sidecar container ที่จัดการ authentication และ secret fetching แทน application

```yaml
# deployment-with-vault-agent.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-vault-agent
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vault-agent-app
  template:
    metadata:
      labels:
        app: vault-agent-app
      annotations:
        # Enable Vault Agent injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "production-app"
        vault.hashicorp.com/agent-inject-status: "update"
        
        # Static secret
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/production/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/production/config" -}}
          APP_ENV={{ .Data.data.environment }}
          LOG_LEVEL={{ .Data.data.log_level }}
          FEATURE_FLAG={{ .Data.data.feature_flag }}
          {{- end -}}
        
        # Dynamic database credentials
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/readwrite"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/readwrite" -}}
          DB_USER={{ .Data.username }}
          DB_PASSWORD={{ .Data.password }}
          {{- end -}}
        
        # TLS certificate
        vault.hashicorp.com/agent-inject-secret-tls: "pki/issue/production"
        vault.hashicorp.com/agent-inject-template-tls: |
          {{- with secret "pki/issue/production" "common_name=app.production.svc" "ttl=24h" -}}
          {{ .Data.certificate }}
          {{- end -}}
        
        # Vault Agent configuration
        vault.hashicorp.com/agent-limits-cpu: "50m"
        vault.hashicorp.com/agent-limits-mem: "64Mi"
        vault.hashicorp.com/agent-requests-cpu: "10m"
        vault.hashicorp.com/agent-requests-mem: "32Mi"
    spec:
      serviceAccountName: production-app-sa
      containers:
      - name: app
        image: my-app:latest
        command: ["sh", "-c"]
        args:
        - |
          # Load secrets จาก Vault Agent files
          set -a
          source /vault/secrets/config
          source /vault/secrets/db
          set +a
          exec /app/server
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

### Vault CSI Provider - ละเอียด

```bash
# ติดตั้ง Vault CSI Provider
helm install vault hashicorp/vault \
  --namespace vault \
  --set "injector.enabled=false" \
  --set "csi.enabled=true"

# ติดตั้ง Secrets Store CSI Driver
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true  # sync เป็น K8s Secret ด้วย
```

```yaml
# vault-secretprovider.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-credentials
  namespace: production
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.vault.svc.cluster.local:8200"
    roleName: "production-app"
    objects: |
      - objectName: "db-username"
        secretPath: "database/creds/readonly"
        secretKey: "username"
      - objectName: "db-password"
        secretPath: "database/creds/readonly"
        secretKey: "password"
      - objectName: "app-config"
        secretPath: "secret/data/production/config"
        secretKey: "data"
  # Sync เป็น K8s Secret
  secretObjects:
  - secretName: vault-db-k8s-secret
    type: Opaque
    data:
    - objectName: db-username
      key: DB_USER
    - objectName: db-password
      key: DB_PASSWORD
```

```yaml
# pod-using-csi.yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-secret-app
  namespace: production
spec:
  serviceAccountName: production-app-sa
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: vault-secrets
      mountPath: "/mnt/secrets-store"
      readOnly: true
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: vault-db-k8s-secret
          key: DB_USER
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: vault-db-k8s-secret
          key: DB_PASSWORD
  volumes:
  - name: vault-secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: vault-db-credentials
```

### เปรียบเทียบ Vault Agent vs Vault CSI Provider

| Feature | Vault Agent Sidecar | Vault CSI Provider |
|---------|--------------------|--------------------|
| Deployment | Sidecar ใน pod | DaemonSet ระดับ Node |
| Secret Location | Files ใน /vault/secrets | Files ใน volume mount |
| Auto-renewal | ✅ อัตโนมัติ | ✅ ด้วย rotation |
| K8s Secret Sync | ❌ (ไม่สร้าง K8s Secret) | ✅ ผ่าน syncSecret |
| Resource overhead | สูงกว่า (per pod) | ต่ำกว่า (per node) |
| Dynamic secrets | ✅ ดีเยี่ยม | ✅ ได้ |
| Template support | ✅ Go templates | จำกัด |
| Init container | ✅ | ❌ |

---

## Workshop: Full Vault + Kubernetes Integration

### โจทย์
Deploy application ที่:
1. Login Vault ด้วย Kubernetes Auth
2. ดึง dynamic database credentials
3. ใช้ static secrets สำหรับ app config
4. Auto-renew credentials ก่อนหมดอายุ

### Step 1: ติดตั้ง Vault บน Kubernetes (Production Mode)

```bash
# ติดตั้งด้วย Helm
helm repo add hashicorp https://helm.releases.hashicorp.com

# สร้าง values file
cat > vault-values.yaml << 'EOF'
server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
        }
        storage "raft" {
          path = "/vault/data"
          retry_join {
            leader_api_addr = "http://vault-0.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "http://vault-1.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "http://vault-2.vault-internal:8200"
          }
        }
        service_registration "kubernetes" {}
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 500m
  affinity: |
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
          topologyKey: kubernetes.io/hostname

ui:
  enabled: true
  serviceType: ClusterIP

injector:
  enabled: true
  resources:
    requests:
      memory: 64Mi
      cpu: 50m
    limits:
      memory: 128Mi
      cpu: 100m
EOF

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --values vault-values.yaml
```

### Step 2: Initialize และ Unseal

```bash
# รอ pods พร้อม
kubectl wait --for=condition=Ready pod/vault-0 -n vault --timeout=120s

# Initialize
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json

# เก็บ keys และ token
UNSEAL_KEY_1=$(cat vault-init.json | jq -r '.unseal_keys_b64[0]')
UNSEAL_KEY_2=$(cat vault-init.json | jq -r '.unseal_keys_b64[1]')
UNSEAL_KEY_3=$(cat vault-init.json | jq -r '.unseal_keys_b64[2]')
ROOT_TOKEN=$(cat vault-init.json | jq -r '.root_token')

# Unseal vault-0
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_1
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_2
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_3

# Join vault-1 และ vault-2
kubectl exec -n vault vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -n vault vault-1 -- vault operator unseal $UNSEAL_KEY_1
kubectl exec -n vault vault-1 -- vault operator unseal $UNSEAL_KEY_2
kubectl exec -n vault vault-1 -- vault operator unseal $UNSEAL_KEY_3

kubectl exec -n vault vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -n vault vault-2 -- vault operator unseal $UNSEAL_KEY_1
kubectl exec -n vault vault-2 -- vault operator unseal $UNSEAL_KEY_2
kubectl exec -n vault vault-2 -- vault operator unseal $UNSEAL_KEY_3
```

### Step 3: ตั้งค่า Vault

```bash
# Login
export VAULT_ADDR=http://127.0.0.1:8200
kubectl port-forward svc/vault -n vault 8200:8200 &
vault login $ROOT_TOKEN

# Enable KV v2
vault secrets enable -path=secret kv-v2

# Enable database engine
vault secrets enable database

# Enable Kubernetes auth
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

### Step 4: สร้าง Secrets และ Policies

```bash
# สร้าง secrets
vault kv put secret/production/app \
  environment=production \
  log_level=info \
  feature_flags='{"dark_mode":true,"beta_api":false}' \
  jwt_secret="$(openssl rand -hex 32)"

# สร้าง policy
vault policy write production-app - << 'EOF'
path "secret/data/production/*" {
  capabilities = ["read"]
}
path "database/creds/readonly" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF

# สร้าง Kubernetes role
vault write auth/kubernetes/role/production-app \
  bound_service_account_names=production-app-sa \
  bound_service_account_namespaces=production \
  policies=production-app \
  ttl=1h \
  max_ttl=24h
```

### Step 5: Deploy Application

```yaml
# production-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: production-app-sa
  namespace: production
automountServiceAccountToken: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "production-app"
        vault.hashicorp.com/agent-inject-secret-app-config: "secret/data/production/app"
        vault.hashicorp.com/agent-inject-template-app-config: |
          {{- with secret "secret/data/production/app" -}}
          export APP_ENV="{{ .Data.data.environment }}"
          export LOG_LEVEL="{{ .Data.data.log_level }}"
          export JWT_SECRET="{{ .Data.data.jwt_secret }}"
          {{- end -}}
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/readonly"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/readonly" -}}
          export DB_USER="{{ .Data.username }}"
          export DB_PASSWORD="{{ .Data.password }}"
          {{- end -}}
    spec:
      serviceAccountName: production-app-sa
      containers:
      - name: app
        image: nginx:alpine
        command: ["sh", "-c"]
        args:
        - |
          source /vault/secrets/app-config
          source /vault/secrets/db-creds
          echo "App started with user: $DB_USER"
          nginx -g 'daemon off;'
```

### Step 6: Verification

```bash
# ตรวจสอบ pods
kubectl get pods -n production

# ดู vault-agent logs
kubectl logs -n production <pod-name> -c vault-agent

# ตรวจสอบ secrets ถูก inject
kubectl exec -n production <pod-name> -c app -- \
  cat /vault/secrets/app-config

# ดู Vault audit logs
kubectl exec -n vault vault-0 -- \
  vault audit list

# Monitor token renewal
kubectl exec -n production <pod-name> -c vault-agent -- \
  cat /tmp/vault-agent.log
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Vault Policy Design

ออกแบบ policies สำหรับ team structure ดังนี้:
- `developers`: อ่าน dev/staging secrets ได้ ไม่แตะ production
- `sre`: อ่าน production secrets ได้ manage infrastructure secrets ได้
- `ci-cd-bot`: อ่าน secrets ทุก environment ได้ สร้าง deploy tokens ได้

```hcl
# เฉลย: developers policy
path "secret/data/development/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/data/staging/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/development/*" {
  capabilities = ["read", "list", "delete"]
}
# อ่าน database creds สำหรับ dev environment
path "database/creds/dev-*" {
  capabilities = ["read"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

```hcl
# เฉลย: sre policy
path "secret/data/production/*" {
  capabilities = ["read", "list"]
}
path "secret/data/infrastructure/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/data/+/*" {
  capabilities = ["read", "list"]
}
path "database/creds/*" {
  capabilities = ["read"]
}
path "sys/health" {
  capabilities = ["read", "sudo"]
}
path "sys/metrics" {
  capabilities = ["read"]
}
```

```hcl
# เฉลย: ci-cd-bot policy
path "secret/data/+/*" {
  capabilities = ["read"]
}
path "secret/data/deploy/*" {
  capabilities = ["create", "read", "update"]
}
# สร้าง limited tokens สำหรับ deployment jobs
path "auth/token/create/deploy-role" {
  capabilities = ["create", "update"]
}
path "auth/token/roles/deploy-role" {
  capabilities = ["read"]
}
```

### แบบฝึกหัดที่ 2: Dynamic Database Credentials

Setup database engine พร้อม roles และทดสอบขอ credentials:

```bash
# เฉลย:
# Step 1: เตรียม PostgreSQL
kubectl run postgres-test \
  --image=postgres:15 \
  --env="POSTGRES_PASSWORD=admin123" \
  --env="POSTGRES_DB=testdb" \
  --expose --port=5432

# รอ postgres พร้อม
kubectl wait --for=condition=Ready pod/postgres-test --timeout=60s

# สร้าง vault admin user ใน postgres
kubectl exec postgres-test -- psql -U postgres -c "
  CREATE ROLE vault_admin WITH LOGIN PASSWORD 'vault_admin_pass' CREATEROLE;
  GRANT ALL PRIVILEGES ON DATABASE testdb TO vault_admin;
"

# Step 2: Configure Vault database engine
vault secrets enable database
vault write database/config/workshop-pg \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres-test.default.svc.cluster.local:5432/testdb?sslmode=disable" \
  username="vault_admin" \
  password="vault_admin_pass" \
  allowed_roles="workshop-readonly"

# Step 3: สร้าง role
vault write database/roles/workshop-readonly \
  db_name=workshop-pg \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
  " \
  default_ttl="5m" \
  max_ttl="30m"

# Step 4: ทดสอบขอ credentials
vault read database/creds/workshop-readonly

# Step 5: ทดสอบ login ด้วย credentials ที่ได้
DB_USER=$(vault read -field=username database/creds/workshop-readonly)
DB_PASS=$(vault read -field=password database/creds/workshop-readonly)
kubectl exec postgres-test -- psql -U "$DB_USER" -d testdb \
  -c "SELECT current_user, now();"
```

### แบบฝึกหัดที่ 3: Kubernetes Auth Integration

ทดสอบ application pod login Vault และดึง secrets:

```yaml
# เฉลย: สร้าง test pod
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-vault-sa
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: vault-test-pod
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "test-role"
    vault.hashicorp.com/agent-inject-secret-test: "secret/data/test/config"
    vault.hashicorp.com/agent-inject-template-test: |
      {{- with secret "secret/data/test/config" -}}
      message={{ .Data.data.message }}
      {{- end -}}
spec:
  serviceAccountName: test-vault-sa
  containers:
  - name: test
    image: alpine
    command: ["sh", "-c", "cat /vault/secrets/test && sleep 3600"]
```

```bash
# สร้าง test secret และ role
vault kv put secret/test/config message="Hello from Vault!"
vault write auth/kubernetes/role/test-role \
  bound_service_account_names=test-vault-sa \
  bound_service_account_namespaces=default \
  policies=default \
  ttl=1h

# Deploy pod
kubectl apply -f vault-test-pod.yaml

# ตรวจสอบ
kubectl logs vault-test-pod -c vault-agent
kubectl exec vault-test-pod -c test -- cat /vault/secrets/test
# ควรเห็น: message=Hello from Vault!
```

---

## สรุปเพิ่มเติม

### Vault Best Practices

```bash
# 1. ใช้ token policies ที่ least privilege
vault token create \
  -policy="my-app" \
  -ttl="1h" \
  -use-limit=10 \
  -orphan \
  -no-default-policy

# 2. Enable audit logging เสมอ
vault audit enable file \
  file_path=/vault/logs/audit.log \
  log_raw=false  # ไม่ log raw values

# 3. ตั้งค่า token renewal
vault token renew <token>

# 4. Monitor expiring leases
vault list sys/leases/lookup/database/creds/readonly

# 5. Revoke all leases เมื่อ incident
vault lease revoke -prefix database/creds/
```

### Vault Monitoring

```yaml
# vault-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vault-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: vault
  namespaceSelector:
    matchNames:
    - vault
  endpoints:
  - port: https-internal
    scheme: https
    path: /v1/sys/metrics
    params:
      format: ["prometheus"]
    tlsConfig:
      insecureSkipVerify: true
    interval: 30s
```

**ต่อไป**: Part 55 - RBAC (Role-Based Access Control)
