# Part 53: Secrets Management

## บทนำ

Kubernetes Secrets เป็นวิธีการเก็บข้อมูลที่ละเอียดอ่อน (sensitive data) เช่น passwords, tokens, SSH keys แต่ Secrets ใน Kubernetes มีข้อจำกัดด้านความปลอดภัยหลายอย่าง ในบทนี้เราจะเรียนรู้:

1. Kubernetes Secrets Types และ Base64 Encoding
2. Sealed Secrets (เข้ารหัสจริง ๆ)
3. External Secrets Operator (ดึงจาก external secret stores)
4. Best Practices สำหรับ Production

---

## 53.1 Kubernetes Secrets Types

Kubernetes รองรับ Secret หลายประเภท:

| Type | ใช้สำหรับ |
|------|-----------|
| `Opaque` | ข้อมูลทั่วไป (default) |
| `kubernetes.io/service-account-token` | Service account tokens |
| `kubernetes.io/dockercfg` | ~/.dockercfg serialized |
| `kubernetes.io/dockerconfigjson` | ~/.docker/config.json |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH private key |
| `kubernetes.io/tls` | TLS certificate และ private key |
| `bootstrap.kubernetes.io/token` | Bootstrap token data |

### Opaque Secret

```yaml
# ประเภทที่ใช้บ่อยที่สุด
apiVersion: v1
kind: Secret
metadata:
  name: my-opaque-secret
  namespace: default
type: Opaque
data:
  # ค่าต้อง base64 encoded
  username: YWRtaW4=           # "admin"
  password: UGFzc3dvcmQxMjM=   # "Password123"
  api-key: YWJjMTIzeHl6Nzg5   # "abc123xyz789"
stringData:
  # ใช้ stringData เพื่อใส่ plain text (Kubernetes จะ encode ให้อัตโนมัติ)
  connection-string: "postgresql://admin:Password123@postgres:5432/mydb"
```

### TLS Secret

```bash
# สร้าง TLS secret จาก certificate files
kubectl create secret tls my-tls-secret \
  --cert=/path/to/tls.crt \
  --key=/path/to/tls.key \
  --namespace=default

# หรือสร้าง self-signed cert สำหรับ testing
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.example.com/O=MyOrg"

kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
# TLS Secret YAML
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi... # base64 encoded private key
```

### Docker Registry Secret

```bash
# สร้าง secret สำหรับ pull images จาก private registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com \
  --namespace=default
```

```yaml
# ใช้ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
  - name: regcred   # อ้างอิง Docker registry secret
  containers:
  - name: app
    image: registry.example.com/myapp:1.0.0
```

### Basic Auth Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
  namespace: default
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: secure-password-here
```

### SSH Auth Secret

```bash
# สร้าง SSH key pair
ssh-keygen -t rsa -b 4096 -f /tmp/id_rsa -N ""

# สร้าง Secret
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=/tmp/id_rsa \
  --from-file=ssh-publickey=/tmp/id_rsa.pub
```

---

## 53.2 Base64 Encoding

Base64 **ไม่ใช่** การเข้ารหัส (encryption) แต่เป็นการ encode ข้อมูลเท่านั้น ใครก็ได้ที่เข้าถึง Secret สามารถ decode ได้ทันที

```bash
# Encode
echo -n 'mypassword' | base64
# Output: bXlwYXNzd29yZA==

# Decode
echo 'bXlwYXNzd29yZA==' | base64 -d
# Output: mypassword

# ดูค่าใน Secret
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

### ทำไม Base64 ถึงไม่เพียงพอ?

```
ปัญหาของ Native Kubernetes Secrets:
1. เก็บใน etcd แบบ unencrypted (by default)
2. ใครที่มีสิทธิ์ get/list secrets สามารถ decode ได้
3. Secret ใน Git repo = disaster
4. ไม่มี audit trail ว่าใครเข้าถึงค่าจริง
5. Rotation ทำได้ยาก
```

### ปรับปรุงด้วย Encryption at Rest

```yaml
# /etc/kubernetes/enc/enc.yaml (สำหรับ kube-apiserver)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  # เข้ารหัสด้วย AES-CBC
  - aescbc:
      keys:
      - name: key1
        secret: <32-byte-key-base64-encoded>
  # Identity = ไม่เข้ารหัส (fallback)
  - identity: {}
```

```bash
# สร้าง encryption key
head -c 32 /dev/urandom | base64

# เพิ่ม flag ใน kube-apiserver
# --encryption-provider-config=/etc/kubernetes/enc/enc.yaml

# Encrypt secrets ที่มีอยู่แล้ว
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
```

---

## 53.3 Sealed Secrets

Sealed Secrets แก้ปัญหา "Secrets in Git" โดยใช้ asymmetric cryptography:
- Public key: ใช้ encrypt (ทุกคนทำได้)
- Private key: ใช้ decrypt (อยู่ใน cluster เท่านั้น)

### ติดตั้ง Sealed Secrets Controller

```bash
# ติดตั้งด้วย Helm
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller

# ตรวจสอบ
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
kubectl get crd sealedsecrets.bitnami.com
```

### ติดตั้ง kubeseal CLI

```bash
# macOS
brew install kubeseal

# Linux
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/tags | \
  jq -r '.[0].name' | cut -c 2-)

curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# ตรวจสอบ
kubeseal --version
```

### สร้าง SealedSecret

```bash
# สร้าง regular Secret ก่อน (ยังไม่ apply!)
kubectl create secret generic my-db-secret \
  --from-literal=DB_PASSWORD='MySecurePassword123' \
  --from-literal=DB_USER='myapp_user' \
  --dry-run=client \
  -o yaml > my-secret.yaml

cat my-secret.yaml
```

```yaml
# my-secret.yaml (plain - อย่า commit ไฟล์นี้!)
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: my-db-secret
data:
  DB_PASSWORD: TXlTZWN1cmVQYXNzd29yZDEyMw==
  DB_USER: bXlhcHBfdXNlcg==
```

```bash
# Seal the secret
kubeseal < my-secret.yaml > my-sealed-secret.yaml

cat my-sealed-secret.yaml
```

```yaml
# my-sealed-secret.yaml (ปลอดภัย - commit ได้!)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: my-db-secret
  namespace: default
spec:
  encryptedData:
    DB_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # encrypted!
    DB_USER: AgAUsCmmiqHMCRFVOqKDGQxsRuAjxBlH...      # encrypted!
  template:
    metadata:
      creationTimestamp: null
      name: my-db-secret
      namespace: default
```

```bash
# Apply SealedSecret
kubectl apply -f my-sealed-secret.yaml

# Controller จะ decrypt และสร้าง Secret จริง
kubectl get secret my-db-secret
kubectl get sealedsecret my-db-secret

# ตรวจสอบ
kubectl get secret my-db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### Scope ของ SealedSecret

```bash
# strict scope: namespace + secret name ต้องตรงทั้งคู่ (default)
kubeseal --scope strict < my-secret.yaml > my-sealed.yaml

# namespace-wide: ต้องอยู่ใน namespace เดิม แต่ secret name เปลี่ยนได้
kubeseal --scope namespace-wide < my-secret.yaml > my-sealed.yaml

# cluster-wide: deploy ที่ไหนก็ได้ใน cluster
kubeseal --scope cluster-wide < my-secret.yaml > my-sealed.yaml
```

### Re-encryption เมื่อ Controller Key หมุน

```bash
# ดู controller public key
kubeseal --fetch-cert

# สำรอง master key
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key.yaml

# Re-seal ด้วย key ใหม่
kubeseal --re-encrypt < my-sealed-secret.yaml > re-sealed-secret.yaml
kubectl apply -f re-sealed-secret.yaml
```

---

## 53.4 External Secrets Operator

External Secrets Operator (ESO) ดึง secrets จาก external stores มาเก็บเป็น Kubernetes Secrets โดยอัตโนมัติ

### Supported Backends

- AWS Secrets Manager
- AWS SSM Parameter Store
- HashiCorp Vault
- Azure Key Vault
- Google Cloud Secret Manager
- 1Password
- Doppler
- และอื่นๆ

### ติดตั้ง External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

kubectl get pods -n external-secrets
kubectl get crd | grep external-secrets
```

### ตัวอย่าง: AWS Secrets Manager

```yaml
# 1. สร้าง Secret สำหรับ AWS credentials
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: default
stringData:
  access-key: "AKIAIOSFODNN7EXAMPLE"
  secret-access-key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
---
# 2. สร้าง SecretStore (namespace-scoped)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1    # Bangkok region
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
---
# 3. สร้าง ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: default
spec:
  refreshInterval: 1h         # sync ทุก 1 ชั่วโมง
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: my-app-secret       # ชื่อ Secret ที่จะสร้างใน Kubernetes
    creationPolicy: Owner     # ESO manages this secret
  data:
  # ดึง field เฉพาะ
  - secretKey: DB_PASSWORD    # key ใน Kubernetes Secret
    remoteRef:
      key: myapp/production/database    # path ใน AWS Secrets Manager
      property: password                 # field ใน JSON secret
  
  - secretKey: DB_USER
    remoteRef:
      key: myapp/production/database
      property: username
  
  # ดึง secret ทั้งหมด
  dataFrom:
  - extract:
      key: myapp/production/api-keys   # ดึงทุก key จาก secret นี้
```

### ตัวอย่าง: ClusterSecretStore (cluster-wide)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: cluster-aws-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        # ใช้ IRSA (IAM Roles for Service Accounts)
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### ตัวอย่าง: HashiCorp Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: default
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-app-role"
          serviceAccountRef:
            name: default
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: default
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: vault-app-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/myapp
      property: password
```

### ตัวอย่าง: Google Cloud Secret Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-store
  namespace: default
spec:
  provider:
    gcpsm:
      projectID: my-gcp-project
      auth:
        workloadIdentity:
          clusterLocation: asia-southeast1
          clusterName: my-gke-cluster
          serviceAccountRef:
            name: workload-identity-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gcp-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-store
    kind: SecretStore
  target:
    name: gcp-app-secret
  data:
  - secretKey: API_KEY
    remoteRef:
      key: projects/my-project/secrets/api-key
      version: latest
```

---

## 53.5 Workshop: Secure Secrets Workflow

### สถานการณ์

เราจะสร้าง workflow ที่ปลอดภัยสำหรับการจัดการ secrets:
1. ใช้ Sealed Secrets เพื่อเก็บ secrets ใน Git ได้อย่างปลอดภัย
2. ใช้ External Secrets Operator สำหรับ integration กับ secret store
3. Implement secret rotation

### Step 1: Setup Namespace

```bash
kubectl create namespace workshop-secrets
kubectl config set-context --current --namespace=workshop-secrets
```

### Step 2: ติดตั้ง Sealed Secrets (ใน workshop environment)

```bash
# ใช้ Helm
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller \
  --wait

kubectl get pods -n kube-system | grep sealed-secrets
```

### Step 3: สร้าง Application Secrets ด้วย Sealed Secrets

```bash
# Step 3a: สร้าง plain Secret (temporary)
kubectl create secret generic webapp-db-secret \
  --from-literal=DB_HOST=postgres.workshop-secrets.svc.cluster.local \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=webapp_db \
  --from-literal=DB_USER=webapp_user \
  --from-literal=DB_PASSWORD='Secure@Passw0rd2024' \
  --namespace=workshop-secrets \
  --dry-run=client \
  -o yaml > /tmp/plain-secret.yaml

cat /tmp/plain-secret.yaml

# Step 3b: Seal ด้วย kubeseal
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/plain-secret.yaml \
  > sealed-webapp-secret.yaml

# ตรวจสอบผลลัพธ์
cat sealed-webapp-secret.yaml

# Step 3c: ลบไฟล์ plain secret
rm /tmp/plain-secret.yaml

# Step 3d: Apply SealedSecret
kubectl apply -f sealed-webapp-secret.yaml -n workshop-secrets

# ตรวจสอบ
kubectl get sealedsecret -n workshop-secrets
kubectl get secret webapp-db-secret -n workshop-secrets
kubectl describe sealedsecret webapp-db-secret -n workshop-secrets
```

### Step 4: สร้าง Application ที่ใช้ Sealed Secret

```yaml
# webapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: workshop-secrets
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: DB_PASSWORD
        
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
```

```bash
kubectl apply -f webapp-deployment.yaml

# ตรวจสอบ env vars
kubectl wait --for=condition=ready pod -l app=webapp -n workshop-secrets --timeout=60s
POD=$(kubectl get pod -l app=webapp -n workshop-secrets -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -n workshop-secrets -- env | grep DB_
```

### Step 5: Secret Rotation

```bash
# สร้าง secret ใหม่ (password ใหม่)
kubectl create secret generic webapp-db-secret \
  --from-literal=DB_HOST=postgres.workshop-secrets.svc.cluster.local \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=webapp_db \
  --from-literal=DB_USER=webapp_user \
  --from-literal=DB_PASSWORD='NewSecure@Passw0rd2024' \
  --namespace=workshop-secrets \
  --dry-run=client \
  -o yaml > /tmp/new-secret.yaml

# Seal ใหม่
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/new-secret.yaml \
  > sealed-webapp-secret-v2.yaml

rm /tmp/new-secret.yaml

# Apply
kubectl apply -f sealed-webapp-secret-v2.yaml

# Restart deployment เพื่อ pick up ค่าใหม่
kubectl rollout restart deployment/webapp -n workshop-secrets
kubectl rollout status deployment/webapp -n workshop-secrets

# ตรวจสอบ password ใหม่
POD=$(kubectl get pod -l app=webapp -n workshop-secrets -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -n workshop-secrets -- printenv DB_PASSWORD
```

### Step 6: RBAC สำหรับ Secrets

```yaml
# secret-rbac.yaml
# Role สำหรับ app - อ่าน secret เฉพาะ
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: workshop-secrets
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["webapp-db-secret"]  # เฉพาะ secret นี้เท่านั้น
  verbs: ["get"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: webapp-secret-binding
  namespace: workshop-secrets
subjects:
- kind: ServiceAccount
  name: webapp-sa
  namespace: workshop-secrets
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
---
# ServiceAccount สำหรับ app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-sa
  namespace: workshop-secrets
automountServiceAccountToken: false
```

```bash
kubectl apply -f secret-rbac.yaml

# ทดสอบ: ใช้ service account ลอง get secret
kubectl auth can-i get secret/webapp-db-secret \
  --as=system:serviceaccount:workshop-secrets:webapp-sa \
  -n workshop-secrets
# yes

kubectl auth can-i list secrets \
  --as=system:serviceaccount:workshop-secrets:webapp-sa \
  -n workshop-secrets
# no
```

### Step 7: ตรวจสอบ Secret Audit

```bash
# Enable audit logging (ปกติต้องตั้งค่า kube-apiserver)
# ดู audit events ที่เกี่ยวกับ secrets

# ดู events ใน namespace
kubectl get events -n workshop-secrets --sort-by='.lastTimestamp'

# ดู who accessed secrets (ถ้ามี audit log)
# grep '"resource":"secrets"' /var/log/kubernetes/audit.log | jq '.'
```

### Step 8: Secret Backup

```bash
# สำรอง SealedSecrets (ปลอดภัยที่จะ backup)
kubectl get sealedsecrets -n workshop-secrets -o yaml > backup-sealed-secrets.yaml

# สำรอง master key ของ Sealed Secrets controller
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key-BACKUP.yaml
# เก็บไว้ในที่ปลอดภัย! ใครมีไฟล์นี้ = สามารถ decrypt secrets ได้

echo "IMPORTANT: Keep sealed-secrets-master-key-BACKUP.yaml SECURE!"
```

### Step 9: Cleanup

```bash
kubectl delete namespace workshop-secrets
echo "Workshop cleanup complete!"
```

---

## 53.6 Comparison: ตัวเลือกสำหรับ Secret Management

| Feature | Native K8s Secrets | Sealed Secrets | External Secrets |
|---------|-------------------|----------------|------------------|
| Git Safe | ❌ | ✅ | ✅ |
| Encryption | Base64 only | Strong encryption | External store |
| Rotation | Manual | Manual + re-seal | Automatic |
| Audit Trail | Limited | Limited | Full |
| Learning Curve | Low | Medium | Medium-High |
| Dependencies | None | Controller | Operator + Backend |
| Production Ready | Limited | Yes | Yes |

---

## 53.7 Best Practices สำหรับ Secret Management

### 1. ห้ามเก็บ plain secrets ใน Git

```bash
# .gitignore ควรมี:
echo "*.secret.yaml" >> .gitignore
echo "*-secret.yaml" >> .gitignore
echo "secrets/" >> .gitignore

# ใช้ pre-commit hook ป้องกัน
# .git/hooks/pre-commit
#!/bin/bash
if git diff --cached --name-only | grep -E "(secret|password|credential)" > /dev/null; then
  echo "WARNING: Potential secret file detected!"
  exit 1
fi
```

### 2. Rotate Secrets สม่ำเสมอ

```bash
# สร้าง CronJob สำหรับ rotation reminder
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-rotation-reminder
spec:
  schedule: "0 9 1 */3 *"  # ทุก 3 เดือน
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: reminder
            image: curlimages/curl:latest
            command: ['curl', '-X', 'POST', 
                      'https://hooks.slack.com/...', 
                      '-d', '{"text":"Time to rotate secrets!"}']
          restartPolicy: Never
```

### 3. ใช้ Least Privilege สำหรับ Secret Access

```yaml
# Role เฉพาะสำหรับแต่ละ app
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-specific-secret"]  # ระบุชื่อ secret ชัดเจน
  verbs: ["get"]
# ไม่ให้ list หรือ watch เพื่อป้องกัน secret enumeration
```

### 4. Enable Encryption at Rest

```bash
# ตรวจสอบว่า etcd encrypt secrets หรือเปล่า
kubectl get secret test-secret -o jsonpath='{.data}' > /dev/null
# ถ้า encrypt ได้ จะเห็น encrypted data ใน etcd

# Verify encryption
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C
# ถ้า encrypted จะเห็น "k8s:enc:aescbc" นำหน้า
```

### 5. Monitor Secret Access

```yaml
# Falco rule สำหรับ detect secret access
- rule: Read sensitive file untrusted
  desc: Detects reads to sensitive files by untrusted programs
  condition: >
    sensitive_files and evt.type = open
    and not trusted_logging_paths
    and not trusted_secret_processes
  output: >
    Sensitive file opened for reading (user=%user.name 
    command=%proc.cmdline file=%fd.name)
  priority: WARNING
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Kubernetes Secret Types**: Opaque, TLS, Docker registry, SSH, Basic auth
2. **Base64 Encoding**: ไม่ใช่ encryption แต่ encoding เท่านั้น
3. **Sealed Secrets**: encrypt secrets ได้จริง เก็บใน Git ได้ปลอดภัย
4. **External Secrets Operator**: ดึง secrets จาก AWS, Vault, GCP และอื่นๆ

**Key Takeaways:**
- Native K8s Secrets ไม่เพียงพอสำหรับ production โดยไม่มี encryption at rest
- Sealed Secrets เหมาะสำหรับ GitOps workflow
- External Secrets เหมาะสำหรับ enterprise ที่มี secret management system แล้ว
- เสมอใช้ RBAC จำกัดการเข้าถึง Secrets

---

**ต่อไป**: Part 54 - HashiCorp Vault Integration
