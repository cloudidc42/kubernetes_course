# Part 20: Secrets - จัดการข้อมูล Sensitive อย่างปลอดภัย

## สารบัญ
1. [Secrets คืออะไร](#secrets-คืออะไร)
2. [Types of Secrets](#types-of-secrets)
3. [สร้างและใช้ Secrets](#สร้างและใช้-secrets)
4. [Secret Security Considerations](#security-considerations)
5. [Workshop: จัดการ Database Credentials อย่างปลอดภัย](#workshop)

---

## 1. Secrets คืออะไร

**Secret** คือ Kubernetes object ที่ออกแบบมาเพื่อเก็บข้อมูล sensitive เช่น passwords, tokens, SSH keys, TLS certificates โดยแยกออกจาก Pod specification

### ทำไมต้องใช้ Secrets

```
ไม่มี Secrets (อันตราย!):
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PASSWORD
      value: "super-secret-password"  # ❌ ถูก expose ใน YAML!
    - name: API_KEY
      value: "my-private-api-key"     # ❌ เห็นใน git history!

มี Secrets:
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password               # ✓ ไม่ expose ค่าจริงใน YAML
```

### Secrets vs ConfigMaps

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **ข้อมูล** | Non-sensitive config | Sensitive data |
| **Encoding** | Plain text | Base64 encoded |
| **Encryption** | ✗ ใน etcd | ✓ ได้ (ต้องตั้งค่า) |
| **Access Control** | Standard RBAC | Stricter RBAC |
| **ตัวอย่าง** | DB host, feature flags | Passwords, API keys |

### ข้อจำกัดของ Secrets

```
⚠️ สำคัญ:
1. Base64 ≠ Encryption! 
   Secret data เป็นแค่ base64 encoded ไม่ใช่ encrypted
   
2. Stored ใน etcd (default: unencrypted)
   ต้องเปิด "Encryption at Rest" เพื่อ encrypt ใน etcd
   
3. Visible ใน Node ที่ Pod รัน
   kubelet เก็บ Secret values บน Node
   
4. Base64 decoded ได้ง่ายมาก:
   echo "cGFzc3dvcmQ=" | base64 -d
   # password
```

---

## 2. Types of Secrets

Kubernetes มี Secret types หลักๆ ดังนี้:

### 2.1 Opaque (generic) - ทั่วไป

```yaml
# opaque-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque             # default type
data:
  username: YWRtaW4=     # "admin" in base64
  password: cGFzc3dvcmQ= # "password" in base64
  api-key: c3VwZXItc2VjcmV0LWtleQ==  # "super-secret-key" in base64
```

```bash
# แปลง base64 ด้วยมือ
echo -n "admin" | base64          # YWRtaW4=
echo -n "password" | base64       # cGFzc3dvcmQ=

# decode
echo "YWRtaW4=" | base64 -d       # admin
```

### 2.2 kubernetes.io/service-account-token

```yaml
# Service Account Token - สร้างอัตโนมัติ
# ใช้โดย Pods เพื่อ authenticate กับ API Server
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token
  annotations:
    kubernetes.io/service-account.name: my-service-account
type: kubernetes.io/service-account-token
```

### 2.3 kubernetes.io/dockerconfigjson

```yaml
# Docker Registry Credentials
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

```bash
# สร้างจาก command
kubectl create secret docker-registry registry-credentials \
    --docker-server=registry.example.com \
    --docker-username=myuser \
    --docker-password=mypassword \
    --docker-email=myuser@example.com
```

### 2.4 kubernetes.io/basic-auth

```yaml
# HTTP Basic Authentication Credentials
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
data:
  username: YWRtaW4=     # required field
  password: cGFzc3dvcmQ= # required field
```

### 2.5 kubernetes.io/ssh-auth

```yaml
# SSH Authentication Credentials
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key-secret
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64-encoded-private-key>  # required field
```

### 2.6 kubernetes.io/tls

```yaml
# TLS Certificate and Key
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>  # required field
  tls.key: <base64-encoded-key>          # required field
```

```bash
# สร้าง TLS Secret จาก certificate files
kubectl create secret tls my-tls-secret \
    --cert=path/to/tls.crt \
    --key=path/to/tls.key

# หรือสร้าง self-signed certificate (สำหรับ testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /tmp/tls.key \
    -out /tmp/tls.crt \
    -subj "/CN=example.com/O=MyOrg"

kubectl create secret tls my-tls-secret \
    --cert=/tmp/tls.crt \
    --key=/tmp/tls.key
```

### 2.7 bootstrap.kubernetes.io/token

```yaml
# Bootstrap Token สำหรับ Node registration
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-07401b
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  auth-extra-groups: <base64>
  expiration: <base64>
  token-id: <base64>
  token-secret: <base64>
  usage-bootstrap-authentication: <base64>
  usage-bootstrap-signing: <base64>
```

---

## 3. สร้างและใช้ Secrets

### สร้าง Secret

#### วิธีที่ 1: จาก Literals

```bash
# สร้าง generic secret
kubectl create secret generic db-credentials \
    --from-literal=username=admin \
    --from-literal=password=supersecretpassword \
    --from-literal=host=postgres.prod.svc.cluster.local

# ดู Secret (ค่าเป็น base64)
kubectl get secret db-credentials -o yaml

# ดูค่าจริง
kubectl get secret db-credentials \
    -o jsonpath='{.data.password}' | base64 -d
```

#### วิธีที่ 2: จาก File

```bash
# สร้างไฟล์ก่อน
echo -n "my-super-secret-api-key" > /tmp/api-key.txt
echo -n "private-ssh-key-content" > /tmp/ssh-key.pem

# สร้าง Secret จากไฟล์
kubectl create secret generic api-secrets \
    --from-file=api-key=/tmp/api-key.txt \
    --from-file=ssh-key=/tmp/ssh-key.pem
```

#### วิธีที่ 3: จาก YAML (ด้วยมือ base64)

```bash
# แปลงค่าเป็น base64 ก่อน
echo -n "admin" | base64          # YWRtaW4=
echo -n "my-secure-db-password" | base64  # bXktc2VjdXJlLWRiLXBhc3N3b3Jk
echo -n "my-secret-api-key" | base64     # bXktc2VjcmV0LWFwaS1rZXk=
```

```yaml
# secret-manual.yaml
apiVersion: v1
kind: Secret
metadata:
  name: manual-secret
  namespace: production
  labels:
    app: webapp
    type: database-credentials
  annotations:
    description: "Database credentials for production webapp"
    rotation-date: "2024-01-15"
type: Opaque
data:
  username: YWRtaW4=                      # admin
  password: bXktc2VjdXJlLWRiLXBhc3N3b3Jk # my-secure-db-password
  api-key: bXktc2VjcmV0LWFwaS1rZXk=       # my-secret-api-key
stringData:                               # สามารถใส่ plain text ได้!
  host: "postgres.production.svc.cluster.local"  # ไม่ต้อง base64!
  port: "5432"
  database: "webapp_prod"
```

> **หมายเหตุ**: `stringData` จะ auto-encode เป็น base64 เมื่อ apply

```bash
kubectl apply -f secret-manual.yaml
```

### ใช้ Secrets ใน Pod

#### วิธีที่ 1: Environment Variables (envFrom)

```yaml
# pod-with-secret-envfrom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - secretRef:
        name: db-credentials    # โหลดทั้ง Secret เป็น ENV vars
        optional: false
    - secretRef:
        name: api-secrets
        optional: true
```

#### วิธีที่ 2: Specific Keys (valueFrom)

```yaml
# pod-with-secret-valuefrom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-selective-secrets
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-secrets
          key: api-key
          optional: true    # ไม่ fail ถ้าไม่มี key
```

#### วิธีที่ 3: Volume Mount (ไฟล์)

```yaml
# pod-with-secret-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-secret-volume
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets    # directory ใน container
      readOnly: true             # ควรเป็น readonly เสมอ
    - name: tls-vol
      mountPath: /etc/tls
      readOnly: true
  
  volumes:
  - name: secret-vol
    secret:
      secretName: db-credentials
      defaultMode: 0400          # ให้สิทธิ์อ่านเฉพาะ owner
      # เลือก keys ที่จะ mount
      items:
      - key: username
        path: db-username        # ชื่อไฟล์ใน volume
      - key: password
        path: db-password
        mode: 0400               # permission สำหรับ file นี้
  
  - name: tls-vol
    secret:
      secretName: my-tls-secret
      defaultMode: 0400
```

#### วิธีที่ 4: Image Pull Secrets

```yaml
# pod-with-imagepullsecret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: private.registry.example.com/myapp:latest
  imagePullSecrets:
  - name: registry-credentials    # Secret ประเภท dockerconfigjson
```

```bash
# ตั้งค่า default imagePullSecrets ใน Service Account
kubectl patch serviceaccount default \
    -p '{"imagePullSecrets": [{"name": "registry-credentials"}]}'
# หลังจากนี้ทุก Pod ที่ใช้ default SA จะ pull image ด้วย credentials นี้
```

---

## 4. Secret Security Considerations

### การรักษาความปลอดภัยของ Secrets

#### 1. Encryption at Rest

```yaml
# EncryptionConfiguration สำหรับ etcd
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>  # openssl rand -base64 32
  - identity: {}  # fallback
```

```bash
# เปิดใช้งาน encryption ใน kube-apiserver
# เพิ่ม flag: --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# ตรวจสอบว่า Secrets encrypted แล้ว
ETCDCTL_API=3 etcdctl \
    --endpoints=https://localhost:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
    get /registry/secrets/default/my-secret | hexdump -C
# ถ้า encrypted จะเห็น k8s:enc:aescbc:v1: prefix
```

#### 2. RBAC สำหรับ Secrets

```yaml
# ควรจำกัดการเข้าถึง Secrets ด้วย RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]           # อย่าให้ "watch" ถ้าไม่จำเป็น
  resourceNames: ["app-secret"]   # จำกัดเฉพาะ secret นี้
---
# ให้ Service Account เข้าถึงเฉพาะ Secrets ที่จำเป็น
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-secret-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: webapp-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 3. External Secrets Management

สำหรับ Production ควรใช้ External Secret Management:

```yaml
# External Secrets Operator (ESO) - integration กับ AWS, GCP, HashiCorp Vault

# ExternalSecret ดึงค่าจาก AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: webapp-secrets
  namespace: production
spec:
  refreshInterval: 1h           # sync ทุก 1 ชั่วโมง
  secretStoreRef:
    name: aws-secrets-manager   # SecretStore ที่ตั้งค่าไว้
    kind: ClusterSecretStore
  target:
    name: webapp-secret         # ชื่อ Secret ที่จะสร้างใน Kubernetes
    creationPolicy: Owner
  data:
  - secretKey: db-password      # key ใน Kubernetes Secret
    remoteRef:
      key: production/webapp/db # path ใน AWS Secrets Manager
      property: password        # field ใน secret
  - secretKey: api-key
    remoteRef:
      key: production/webapp/api
      property: key
```

```yaml
# HashiCorp Vault integration
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secrets
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
  data:
  - secretKey: db-password
    remoteRef:
      key: secret/production/webapp
      property: db_password
```

#### 4. Best Practices

```bash
# ✓ 1. ไม่ commit Secrets ใน Git
echo "*.yaml" >> .gitignore     # อย่าทำแบบนี้ มัน block ทุก yaml!
# ใช้ git-secrets หรือ pre-commit hooks แทน
git secrets --register-aws
git secrets --install

# ✓ 2. Rotate Secrets เป็นประจำ
# ตั้ง rotation schedule ใน CI/CD pipeline

# ✓ 3. ใช้ Least Privilege
# ให้สิทธิ์เข้าถึง Secrets เฉพาะที่จำเป็น

# ✓ 4. Monitor Secret Access
# ใช้ Audit Logging เพื่อ track ว่าใครเข้าถึง Secrets

# ✓ 5. ไม่ log Secret values
# ตรวจสอบว่า application ไม่ log environment variables

# ✓ 6. ใช้ readOnly volume mount
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true   # เสมอ!

# ✓ 7. ตั้ง automountServiceAccountToken: false
# เมื่อ Pod ไม่จำเป็นต้องเข้าถึง API Server
spec:
  automountServiceAccountToken: false
```

---

## 5. Workshop: จัดการ Database Credentials อย่างปลอดภัย

### สถานการณ์

เรามี Web Application ที่ต้องการ connect กับ PostgreSQL Database อย่างปลอดภัย รวมถึงมี External API keys

### Workshop Setup

```bash
kubectl create namespace secrets-workshop
kubectl config set-context --current --namespace=secrets-workshop
```

### Lab 1: สร้าง Secrets ต่างๆ

```bash
# === Lab 1.1: Database Credentials ===
kubectl create secret generic postgres-credentials \
    --from-literal=username=webapp_user \
    --from-literal=password=S3cur3P@ssw0rd123 \
    --from-literal=host=postgres.secrets-workshop.svc.cluster.local \
    --from-literal=port=5432 \
    --from-literal=database=webapp_db \
    --namespace=secrets-workshop

# ดู Secret
kubectl describe secret postgres-credentials
kubectl get secret postgres-credentials -o yaml

# ดูค่าจริง (decoded)
echo "Username: $(kubectl get secret postgres-credentials \
    -o jsonpath='{.data.username}' | base64 -d)"
echo "Password: $(kubectl get secret postgres-credentials \
    -o jsonpath='{.data.password}' | base64 -d)"
echo "Host: $(kubectl get secret postgres-credentials \
    -o jsonpath='{.data.host}' | base64 -d)"

# === Lab 1.2: API Keys ===
kubectl create secret generic api-keys \
    --from-literal=stripe-api-key=sk_live_xxxxxxxxxxxxxxxxxxxxx \
    --from-literal=sendgrid-api-key=SG.xxxxxxxxxxxxxxxxxxxxxx \
    --from-literal=aws-access-key-id=AKIAIOSFODNN7EXAMPLE \
    --from-literal=aws-secret-access-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
    --namespace=secrets-workshop

# === Lab 1.3: TLS Certificate ===
# สร้าง self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /tmp/workshop-tls.key \
    -out /tmp/workshop-tls.crt \
    -subj "/CN=webapp.example.com/O=Workshop" \
    2>/dev/null

kubectl create secret tls webapp-tls \
    --cert=/tmp/workshop-tls.crt \
    --key=/tmp/workshop-tls.key \
    --namespace=secrets-workshop

# ดู TLS Secret
kubectl describe secret webapp-tls

# === Lab 1.4: Docker Registry ===
# (ถ้ามี private registry)
kubectl create secret docker-registry private-registry \
    --docker-server=registry.example.com \
    --docker-username=deploy-user \
    --docker-password=deploy-password123 \
    --docker-email=deploy@example.com \
    --namespace=secrets-workshop

# ดู Secrets ทั้งหมด
kubectl get secrets --namespace=secrets-workshop
```

### Lab 2: Deploy Application ที่ใช้ Secrets

```bash
# สร้าง ConfigMap สำหรับ non-sensitive config
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: secrets-workshop
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
EOF

# สร้าง Service Account
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-sa
  namespace: secrets-workshop
automountServiceAccountToken: false   # security best practice
EOF

# Deploy Application
cat <<'EOF' > /tmp/secure-webapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-webapp
  namespace: secrets-workshop
  labels:
    app: secure-webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-webapp
  template:
    metadata:
      labels:
        app: secure-webapp
    spec:
      serviceAccountName: webapp-sa
      automountServiceAccountToken: false
      
      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      
      containers:
      - name: webapp
        image: nginx:1.25
        
        ports:
        - containerPort: 80
          name: http
        
        # Non-sensitive config dari ConfigMap
        envFrom:
        - configMapRef:
            name: webapp-config
        
        # Sensitive: Environment Variables dari Secrets
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: host
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: database
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: stripe-api-key
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Mount TLS certificate เป็นไฟล์
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true      # เสมอ!
        - name: db-creds
          mountPath: /etc/db-credentials
          readOnly: true
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false  # nginx ต้องเขียน pid file
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      
      volumes:
      - name: tls-certs
        secret:
          secretName: webapp-tls
          defaultMode: 0400     # ให้สิทธิ์อ่านเฉพาะ owner
      - name: db-creds
        secret:
          secretName: postgres-credentials
          defaultMode: 0400
          items:
          - key: username
            path: db-username
          - key: password
            path: db-password
EOF

kubectl apply -f /tmp/secure-webapp.yaml

# รอ Deployment พร้อม
kubectl rollout status deployment/secure-webapp

# ตรวจสอบ ENV variables
POD_NAME=$(kubectl get pods -l app=secure-webapp \
    -o jsonpath='{.items[0].metadata.name}')
echo "Pod: $POD_NAME"

# ดู ENV variables ที่ inject (ค่าจริง!)
kubectl exec $POD_NAME -- env | grep -E "(DB_|STRIPE_)"

# ตรวจสอบ volume mounts
kubectl exec $POD_NAME -- ls -la /etc/db-credentials/
kubectl exec $POD_NAME -- cat /etc/db-credentials/db-username
kubectl exec $POD_NAME -- ls -la /etc/tls/
```

### Lab 3: Secret Rotation

```bash
# สถานการณ์: ต้อง rotate database password

# Step 1: สร้าง new credentials
NEW_PASSWORD="NewS3cur3P@ssw0rd456"

# Step 2: อัปเดต Secret
kubectl patch secret postgres-credentials \
    --type='json' \
    -p="[{\"op\": \"replace\", \"path\": \"/data/password\", \"value\": \"$(echo -n $NEW_PASSWORD | base64)\"}]"

# ยืนยันการเปลี่ยนแปลง
kubectl get secret postgres-credentials \
    -o jsonpath='{.data.password}' | base64 -d
echo ""

# Step 3: Restart pods เพื่อ reload ENV (ถ้าใช้ ENV vars)
kubectl rollout restart deployment/secure-webapp
kubectl rollout status deployment/secure-webapp

# Step 4: ยืนยัน ENV ใหม่
POD_NAME=$(kubectl get pods -l app=secure-webapp \
    -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- env | grep DB_PASSWORD
```

### Lab 4: RBAC สำหรับ Secrets

```bash
# สร้าง Role ที่จำกัดการเข้าถึง Secrets
cat <<'EOF' | kubectl apply -f -
# Role: อ่านได้เฉพาะ secrets ที่ระบุ
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: secrets-workshop
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["postgres-credentials"]  # เฉพาะ secret นี้เท่านั้น
---
# Role: admin สำหรับทีม Ops
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-admin
  namespace: secrets-workshop
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-secret-access
  namespace: secrets-workshop
subjects:
- kind: ServiceAccount
  name: webapp-sa
  namespace: secrets-workshop
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# ทดสอบการเข้าถึง
kubectl auth can-i get secrets \
    --namespace=secrets-workshop \
    --as=system:serviceaccount:secrets-workshop:webapp-sa

kubectl auth can-i list secrets \
    --namespace=secrets-workshop \
    --as=system:serviceaccount:secrets-workshop:webapp-sa

kubectl auth can-i get secret/postgres-credentials \
    --namespace=secrets-workshop \
    --as=system:serviceaccount:secrets-workshop:webapp-sa
```

### Lab 5: Sealed Secrets (Production Pattern)

```bash
# Sealed Secrets เป็น tool ที่ encrypt Secrets เพื่อ commit ใน Git ได้

# ติดตั้ง kubeseal (CLI tool)
# curl -L https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz | tar -xz
# sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# ติดตั้ง Sealed Secrets controller ใน cluster
# kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# สร้าง Secret ปกติก่อน
cat <<'EOF' > /tmp/plain-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: sealed-demo-secret
  namespace: secrets-workshop
type: Opaque
stringData:
  password: "my-super-secret-password"
  api-key: "my-private-api-key"
EOF

# Seal Secret (สร้าง SealedSecret ที่ safe to commit)
# kubeseal --format yaml < /tmp/plain-secret.yaml > /tmp/sealed-secret.yaml

# SealedSecret ที่ได้ดูหน้าตาแบบนี้:
cat <<'EXAMPLE'
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: sealed-demo-secret
  namespace: secrets-workshop
spec:
  encryptedData:
    password: AgCvbMSIBfhA7...  # encrypted! safe to commit to git
    api-key: AgB3xKlPMxNJ...    # encrypted!
EXAMPLE
```

### Lab 6: Init Container ที่ดึง Secret

```bash
# Pattern: Init Container ดึง credentials จาก Vault/External store
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-init-demo
  namespace: secrets-workshop
spec:
  initContainers:
  - name: fetch-secrets
    image: busybox:1.36
    command: 
    - sh
    - -c
    - |
      # จำลองการดึง secrets จาก external store
      echo "Fetching secrets from vault..."
      DB_PASSWORD=$(cat /etc/db-creds/db-password)
      echo "db_password=${DB_PASSWORD}" > /shared-secrets/db.env
      echo "Secrets fetched successfully"
    volumeMounts:
    - name: db-creds
      mountPath: /etc/db-creds
      readOnly: true
    - name: shared-secrets
      mountPath: /shared-secrets
  
  containers:
  - name: webapp
    image: busybox:1.36
    command: ['sh', '-c', 
      'echo "Starting webapp..."; cat /secrets/db.env; sleep 3600']
    volumeMounts:
    - name: shared-secrets
      mountPath: /secrets
      readOnly: true
  
  volumes:
  - name: db-creds
    secret:
      secretName: postgres-credentials
      defaultMode: 0400
  - name: shared-secrets
    emptyDir:
      medium: Memory   # เก็บใน RAM ไม่ write ลง disk
EOF

kubectl wait --for=condition=Ready pod/secret-init-demo --timeout=60s
kubectl logs secret-init-demo -c fetch-secrets
kubectl logs secret-init-demo -c webapp
kubectl delete pod secret-init-demo
```

### Cleanup Workshop

```bash
# ลบ namespace ทั้งหมด (ลบ secrets และ resources ทั้งหมด)
kubectl delete namespace secrets-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/secure-webapp.yaml \
    /tmp/plain-secret.yaml \
    /tmp/workshop-tls.key \
    /tmp/workshop-tls.crt
```

### Secret Command Reference

```bash
# สร้าง Secrets
kubectl create secret generic NAME \
    --from-literal=key=value
kubectl create secret generic NAME \
    --from-file=key=file
kubectl create secret tls NAME \
    --cert=cert.pem --key=key.pem
kubectl create secret docker-registry NAME \
    --docker-server=URL \
    --docker-username=USER \
    --docker-password=PASS

# ดู Secrets
kubectl get secrets
kubectl get secret NAME -o yaml
kubectl describe secret NAME     # ค่าถูก mask

# ดูค่าจริง (decoded)
kubectl get secret NAME \
    -o jsonpath='{.data.KEY}' | base64 -d

# แก้ไข Secret
kubectl edit secret NAME
kubectl patch secret NAME \
    --type='json' \
    -p='[{"op":"replace","path":"/data/KEY","value":"<base64>"}]'

# ลบ Secret
kubectl delete secret NAME
```

### Security Checklist

```
Security Checklist สำหรับ Kubernetes Secrets:

□ เปิดใช้งาน Encryption at Rest ใน etcd
□ ใช้ RBAC จำกัดการเข้าถึง Secrets
□ อย่า commit Secrets ใน Git (ใช้ Sealed Secrets หรือ External Secrets)
□ Rotate Secrets เป็นประจำ (อย่างน้อยทุก 90 วัน)
□ Mount Secrets เป็น read-only volumes
□ ใช้ automountServiceAccountToken: false
□ Enable Audit Logging เพื่อ track Secret access
□ ใช้ External Secret Management (Vault, AWS SM, GCP SM)
□ ไม่ log Secret values ใน application logs
□ ใช้ namespace separation สำหรับ environments
□ ตรวจสอบ Pod Security Standards
□ ใช้ Network Policies จำกัด access
```

---

## สรุป

Secrets เป็น object สำคัญที่ Kubernetes ให้มาเพื่อจัดการ sensitive data:

1. **Base64 Encoding**: ข้อมูลถูก encode แต่ไม่ใช่ encrypted (ต้องตั้งค่า encryption at rest)
2. **Multiple Types**: Opaque, TLS, Docker credentials, etc.
3. **Multiple Usage Patterns**: ENV vars, Volume mounts, Image pull
4. **Access Control**: ใช้ RBAC จำกัดการเข้าถึง

**Production Recommendations:**
- ใช้ **External Secrets Operator** กับ HashiCorp Vault หรือ Cloud KMS
- เปิด **Encryption at Rest** เสมอ
- ใช้ **Sealed Secrets** ถ้าต้องเก็บใน Git
- **Rotate** credentials เป็นประจำ

---

## จบ Part 11-20: Core Kubernetes Concepts

คุณได้เรียนรู้ Core Concepts สำคัญทั้งหมดใน Kubernetes แล้ว:

| Part | หัวข้อ | สิ่งสำคัญ |
|------|--------|-----------|
| 11 | kubectl | CLI tool หลัก |
| 12 | Pods | Building block พื้นฐาน |
| 13 | ReplicaSets | Self-healing replicas |
| 14 | Deployments | Rolling updates + Rollbacks |
| 15 | Services | Network exposure |
| 16 | Namespaces | Resource isolation |
| 17 | Labels & Selectors | Resource organization |
| 18 | Annotations | Non-selection metadata |
| 19 | ConfigMaps | Non-sensitive configuration |
| 20 | Secrets | Sensitive data management |

ในบทต่อไป (Part 21+) เราจะเรียนรู้ Advanced Concepts เช่น Persistent Storage, StatefulSets, Jobs, DaemonSets และอื่นๆ
