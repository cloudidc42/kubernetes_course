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
