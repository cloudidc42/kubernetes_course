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

---

## Sealed Secrets (Bitnami) - ละเอียด

Sealed Secrets แก้ปัญหาหลักของ Kubernetes Secrets: **ไม่สามารถเก็บใน Git ได้อย่างปลอดภัย** Sealed Secrets ใช้ asymmetric cryptography (RSA) ในการเข้ารหัส โดย Controller ที่รันใน cluster เท่านั้นที่ถอดรหัสได้

### สถาปัตยกรรม Sealed Secrets

```
┌─────────────────────────────────────────────────────┐
│                    Developer                         │
│                                                      │
│  Secret (plaintext) ──► kubeseal ──► SealedSecret   │
│                            ▲                         │
│                    (ใช้ public key ของ cluster)      │
└─────────────────────────────────────────────────────┘
                              │
                              ▼ (push to Git)
┌─────────────────────────────────────────────────────┐
│                   Git Repository                     │
│                                                      │
│  SealedSecret.yaml (encrypted - ปลอดภัยใน Git)     │
└─────────────────────────────────────────────────────┘
                              │
                              ▼ (kubectl apply)
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                      │
│                                                      │
│  SealedSecret ──► Controller ──► Secret (plaintext) │
│                   (ใช้ private key ถอดรหัส)          │
└─────────────────────────────────────────────────────┘
```

### ติดตั้ง Sealed Secrets Controller

```bash
# วิธีที่ 1: ใช้ Helm (แนะนำ)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller

# วิธีที่ 2: ใช้ manifest โดยตรง
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# ตรวจสอบ installation
kubectl get pods -n kube-system | grep sealed-secrets
kubectl get crd sealedsecrets.bitnami.com
```

### ติดตั้ง kubeseal CLI

```bash
# macOS
brew install kubeseal

# Linux
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/tags | jq -r '.[0].name' | cut -c 2-)
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# ตรวจสอบ
kubeseal --version
```

### สร้าง SealedSecret - YAML ครบ

```bash
# Step 1: สร้าง Secret ธรรมดาก่อน (ยังไม่ apply)
kubectl create secret generic my-app-secret \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=API_KEY=myapikey456 \
  --from-literal=JWT_SECRET=jwtsecretxyz \
  --dry-run=client \
  -o yaml > /tmp/my-secret.yaml

# Step 2: Seal ด้วย kubeseal
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/my-secret.yaml \
  > my-sealed-secret.yaml

# ตอนนี้ my-sealed-secret.yaml ปลอดภัยที่จะ push ไป Git
```

**ตัวอย่าง SealedSecret ที่ได้:**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-app-secret
  namespace: default
spec:
  encryptedData:
    DB_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    API_KEY: AgAR8JVwBNkJDz5xvAfNqMJfSB...
    JWT_SECRET: AgBqkMn7tCHw3Kp2...
  template:
    metadata:
      name: my-app-secret
      namespace: default
    type: Opaque
```

### Scope ของ SealedSecret

```bash
# Scope แบบ strict (default) - ผูกกับ name และ namespace
kubeseal --scope strict < secret.yaml

# Scope แบบ namespace-wide - เปลี่ยนชื่อ secret ได้ใน namespace เดิม
kubeseal --scope namespace-wide < secret.yaml

# Scope แบบ cluster-wide - ใช้ได้ทุก namespace
kubeseal --scope cluster-wide < secret.yaml
```

### SealedSecret สำหรับ TLS Certificate

```bash
# สร้าง TLS Secret
kubectl create secret tls my-tls-secret \
  --cert=server.crt \
  --key=server.key \
  --dry-run=client \
  -o yaml | \
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml > my-sealed-tls.yaml
```

```yaml
# my-sealed-tls.yaml ที่ได้
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-tls-secret
  namespace: production
spec:
  encryptedData:
    tls.crt: AgBf3Kp8...
    tls.key: AgCD9xM2...
  template:
    metadata:
      name: my-tls-secret
      namespace: production
    type: kubernetes.io/tls
```

### Rotation ของ Sealed Secrets Key

```bash
# Sealed Secrets จะ rotate key อัตโนมัติทุก 30 วัน
# แต่ old keys ยังถูก retain ไว้เพื่อถอดรหัส existing secrets

# ดู keys ที่มีอยู่
kubectl get secrets -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o name

# Re-seal ด้วย key ใหม่ (แนะนำหลัง key rotation)
kubeseal --re-encrypt < old-sealed-secret.yaml > new-sealed-secret.yaml

# Fetch public key เพื่อ offline sealing
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > mycert.pem

# ใช้ cert file สำหรับ offline sealing (ไม่ต้องเชื่อมต่อ cluster)
kubeseal --cert mycert.pem < secret.yaml > sealed-secret.yaml
```

### GitOps Workflow ด้วย SealedSecrets

```yaml
# ไฟล์โครงสร้างใน Git Repository
kubernetes/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── staging/
│   │   ├── sealed-secret.yaml    # ✅ ปลอดภัยใน Git
│   │   └── kustomization.yaml
│   └── production/
│       ├── sealed-secret.yaml    # ✅ ปลอดภัยใน Git
│       └── kustomization.yaml
└── scripts/
    └── seal-secret.sh
```

```bash
#!/bin/bash
# seal-secret.sh - Script สำหรับสร้าง SealedSecret
set -euo pipefail

SECRET_NAME=$1
NAMESPACE=${2:-default}
ENV_FILE=${3:-.env}

# สร้าง secret จาก env file
kubectl create secret generic "${SECRET_NAME}" \
  --from-env-file="${ENV_FILE}" \
  --namespace="${NAMESPACE}" \
  --dry-run=client \
  -o yaml | \
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --namespace="${NAMESPACE}" \
  --format yaml \
  > "sealed-${SECRET_NAME}.yaml"

echo "Created: sealed-${SECRET_NAME}.yaml"
echo "Safe to commit to Git!"
```

---

## External Secrets Operator - ละเอียด

External Secrets Operator (ESO) เชื่อมต่อ Kubernetes กับ external secret management systems เช่น AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, GCP Secret Manager

### สถาปัตยกรรม External Secrets Operator

```
┌──────────────────────────────────────────────────────────────┐
│                    External Secret Stores                     │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ AWS Secrets  │  │Azure Key Vault│  │ HashiCorp Vault  │   │
│  │   Manager    │  │              │  │                  │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
└─────────┼────────────────┼──────────────────┼──────────────┘
          │                │                  │
          ▼                ▼                  ▼
┌──────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                         │
│                                                               │
│  ExternalSecret ──► ESO Controller ──► Kubernetes Secret    │
│                                                               │
│  SecretStore / ClusterSecretStore (กำหนด connection)         │
└──────────────────────────────────────────────────────────────┘
```

### ติดตั้ง External Secrets Operator

```bash
# ใช้ Helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

# ตรวจสอบ
kubectl get pods -n external-secrets
kubectl get crd | grep external-secrets
```

### AWS Secrets Manager Integration

```yaml
# Step 1: สร้าง IAM Role สำหรับ ESO (ใช้ IRSA)
# Trust Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:external-secrets:external-secrets"
      }
    }
  }]
}
```

```yaml
# Step 2: สร้าง SecretStore สำหรับ AWS
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

```yaml
# Step 3: สร้าง ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-credentials
  namespace: production
spec:
  refreshInterval: 1h   # sync ทุก 1 ชั่วโมง
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  target:
    name: my-app-secret    # ชื่อ K8s Secret ที่จะสร้าง
    creationPolicy: Owner   # ESO เป็นเจ้าของ secret
    deletionPolicy: Retain  # ไม่ลบ secret เมื่อ ExternalSecret ถูกลบ
  data:
  - secretKey: DB_PASSWORD        # key ใน K8s Secret
    remoteRef:
      key: production/myapp/database   # ชื่อใน AWS Secrets Manager
      property: password               # field ใน JSON value
  - secretKey: API_KEY
    remoteRef:
      key: production/myapp/api
      property: key
  # ดึงทั้ง secret มาเลย (ถ้า value เป็น JSON object)
  dataFrom:
  - extract:
      key: production/myapp/all-secrets
```

### Azure Key Vault Integration

```yaml
# สร้าง Service Principal สำหรับ Azure Key Vault
az ad sp create-for-rbac \
  --name "eso-keyvault-sp" \
  --role "Key Vault Secrets User" \
  --scopes "/subscriptions/SUB_ID/resourceGroups/RG_NAME/providers/Microsoft.KeyVault/vaults/VAULT_NAME"
```

```yaml
# สร้าง Secret สำหรับ Azure credentials ใน K8s
apiVersion: v1
kind: Secret
metadata:
  name: azure-sp-credentials
  namespace: external-secrets
type: Opaque
stringData:
  clientId: "your-service-principal-client-id"
  clientSecret: "your-service-principal-client-secret"
```

```yaml
# ClusterSecretStore สำหรับ Azure Key Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault-store
spec:
  provider:
    azurekv:
      tenantId: "your-azure-tenant-id"
      vaultUrl: "https://my-keyvault.vault.azure.net"
      authType: ServicePrincipal
      authSecretRef:
        clientId:
          name: azure-sp-credentials
          namespace: external-secrets
          key: clientId
        clientSecret:
          name: azure-sp-credentials
          namespace: external-secrets
          key: clientSecret
```

```yaml
# ExternalSecret สำหรับ Azure Key Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: azure-app-secrets
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: azure-keyvault-store
    kind: ClusterSecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_URL
    remoteRef:
      key: database-connection-string  # ชื่อ secret ใน Azure Key Vault
  - secretKey: SMTP_PASSWORD
    remoteRef:
      key: smtp-password
      version: "2"   # ดึง version เฉพาะ
  dataFrom:
  - extract:
      key: app-config   # ดึง JSON object ทั้งหมด
```

### GCP Secret Manager Integration

```yaml
# ClusterSecretStore สำหรับ GCP
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: gcp-secret-store
spec:
  provider:
    gcpsm:
      projectID: "my-gcp-project"
      auth:
        workloadIdentity:
          clusterLocation: asia-southeast1
          clusterName: my-cluster
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

```yaml
# ExternalSecret สำหรับ GCP
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gcp-app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: gcp-derived-secret
  data:
  - secretKey: API_KEY
    remoteRef:
      key: my-api-key
      version: latest
```

### PushSecret - ส่ง Secret จาก K8s ไปยัง External Store

```yaml
# ส่ง K8s Secret ไปเก็บใน AWS Secrets Manager
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-to-aws
  namespace: default
spec:
  refreshInterval: 10s
  secretStoreRefs:
  - name: aws-secrets-store
    kind: ClusterSecretStore
  selector:
    secret:
      name: my-local-secret
  data:
  - match:
      secretKey: DB_PASSWORD
      remoteRef:
        remoteKey: production/pushed/db-password
```

---

## Secret Rotation Strategy

### Automatic Rotation ด้วย External Secrets

```yaml
# ตั้ง refreshInterval สั้นๆ สำหรับ rotation บ่อย
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotating-credentials
  namespace: production
spec:
  refreshInterval: 15m    # ดึง secret ใหม่ทุก 15 นาที
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    # ใช้ template เพื่อสร้าง connection string
    template:
      data:
        DB_URL: "postgresql://{{ .username }}:{{ .password }}@db-host:5432/mydb"
  dataFrom:
  - extract:
      key: production/db/credentials
```

### Database Password Rotation ด้วย AWS RDS

```bash
# ตั้งค่า AWS Secrets Manager สำหรับ auto-rotation
aws secretsmanager create-secret \
  --name production/myapp/rds-credentials \
  --description "RDS credentials with auto-rotation" \
  --secret-string '{"username":"admin","password":"initial-password"}'

# Enable rotation
aws secretsmanager rotate-secret \
  --secret-id production/myapp/rds-credentials \
  --rotation-lambda-arn arn:aws:lambda:ap-southeast-1:123456789:function:SecretsManagerRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

### Secret Rotation สำหรับ Application ที่รันอยู่

```yaml
# ใช้ Reloader เพื่อ restart deployment เมื่อ secret เปลี่ยน
# ติดตั้ง Reloader
helm install reloader stakater/reloader \
  --namespace default

# Annotate deployment ให้ watch secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    # reload เมื่อ secret นี้เปลี่ยน
    secret.reloader.stakater.com/reload: "my-app-secret,db-credentials"
spec:
  template:
    spec:
      containers:
      - name: my-app
        envFrom:
        - secretRef:
            name: my-app-secret
```

### Graceful Secret Rotation Strategy

```bash
# Step 1: สร้าง new secret version
aws secretsmanager put-secret-value \
  --secret-id production/myapp/db \
  --secret-string '{"password":"new-password-v2"}'

# Step 2: อัปเดต ExternalSecret ให้ดึง version ใหม่
# (หรือรอ refreshInterval)
kubectl annotate externalsecret rotating-credentials \
  force-sync="$(date +%s)" \
  --overwrite

# Step 3: ตรวจสอบว่า secret อัปเดตแล้ว
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Step 4: ดู rollout status
kubectl rollout status deployment/my-app

# Step 5: ลบ old secret version (หลังจาก verify แล้ว)
aws secretsmanager delete-secret-version \
  --secret-id production/myapp/db \
  --version-id old-version-id
```

---

## Encryption at Rest - ละเอียด

### ตรวจสอบสถานะ Encryption ปัจจุบัน

```bash
# ดู encryption configuration
kubectl get apiserver -o yaml 2>/dev/null || \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption

# ตรวจสอบว่า etcd encrypt secrets หรือยัง
# ต้องรันบน master node
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | \
  hexdump -C | head -5
# ถ้าไม่ encrypted จะเห็น plaintext "k8s" ตอนต้น
# ถ้า encrypted จะเห็น "k8s:enc:aescbc:v1"
```

### ตั้งค่า Encryption at Rest ด้วย AES-CBC

```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    - configmaps   # optional: encrypt configmaps ด้วย
    providers:
    # Provider แรก = ใช้สำหรับ encrypt ใหม่
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0LWtleS0zMi1ieXRlcy1sb25nLXN0cmluZw==  # base64 ของ 32-byte key
    # Identity = อ่าน unencrypted ได้ (สำหรับ migration)
    - identity: {}
```

```bash
# สร้าง encryption key ที่แข็งแกร่ง
head -c 32 /dev/urandom | base64

# อัปเดต kube-apiserver manifest
cat >> /etc/kubernetes/manifests/kube-apiserver.yaml << EOF
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
EOF

# Mount configuration file
# เพิ่ม volumeMount และ volume ใน kube-apiserver pod spec
```

```yaml
# kube-apiserver.yaml (ส่วนที่ต้องเพิ่ม)
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
    volumeMounts:
    - name: enc-cfg
      mountPath: /etc/kubernetes/enc
      readOnly: true
  volumes:
  - name: enc-cfg
    hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
```

### ตั้งค่า Encryption ด้วย KMS (แนะนำสำหรับ Production)

```yaml
# ใช้ AWS KMS กับ Kubernetes KMS Plugin
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - kms:
        name: aws-kms
        endpoint: unix:///var/run/kmsplugin/socket.sock
        cachesize: 1000
        timeout: 3s
    - identity: {}
```

```bash
# ติดตั้ง AWS KMS Plugin
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-encryption-provider/master/deploy/aws-encryption-provider.yaml

# Encrypt existing secrets ที่ยังไม่ได้ encrypt
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
# (ทุก secret จะถูก re-written และ encrypted ด้วย provider แรก)
```

### Verify Encryption

```bash
# หลังจากตั้งค่าแล้ว ตรวจสอบ
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | \
  strings | head -5

# ควรเห็น: k8s:enc:aescbc:v1:key1:...
# หรือ: k8s:enc:kms:v1:...

# ตรวจสอบว่า kubectl ยังอ่าน secret ได้
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## Workshop: Production Secrets Pipeline

### โจทย์
สร้าง Secrets Pipeline ที่:
1. เก็บ secrets ใน AWS Secrets Manager
2. Sync มายัง Kubernetes ด้วย External Secrets Operator
3. App ใช้ secrets โดยอัตโนมัติ
4. เมื่อ rotate secrets app รู้จัก reload

### Step 1: เตรียม AWS Secrets Manager

```bash
# สร้าง secrets ใน AWS Secrets Manager
aws secretsmanager create-secret \
  --name "workshop/myapp/database" \
  --description "Database credentials for workshop app" \
  --secret-string '{
    "host": "postgres.production.svc.cluster.local",
    "port": "5432",
    "database": "myapp",
    "username": "appuser",
    "password": "SecureP@ssw0rd!",
    "ssl_mode": "require"
  }'

aws secretsmanager create-secret \
  --name "workshop/myapp/api-keys" \
  --secret-string '{
    "stripe_key": "sk_live_...",
    "sendgrid_key": "SG...",
    "slack_webhook": "https://hooks.slack.com/..."
  }'

# ตรวจสอบ
aws secretsmanager list-secrets --query 'SecretList[?starts_with(Name, `workshop/`)]'
```

### Step 2: ติดตั้ง ESO และตั้งค่า

```bash
# ติดตั้ง External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace

# สร้าง namespace สำหรับ workshop
kubectl create namespace workshop
```

```yaml
# workshop-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: workshop-aws-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

### Step 3: สร้าง ExternalSecrets

```yaml
# workshop-external-secrets.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: workshop-db-secret
  namespace: workshop
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: workshop-aws-store
    kind: ClusterSecretStore
  target:
    name: app-database-secret
    creationPolicy: Owner
    template:
      data:
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .database }}?sslmode={{ .ssl_mode }}"
  dataFrom:
  - extract:
      key: workshop/myapp/database
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: workshop-api-keys
  namespace: workshop
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: workshop-aws-store
    kind: ClusterSecretStore
  target:
    name: app-api-keys-secret
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: workshop/myapp/api-keys
```

### Step 4: Deploy Application

```yaml
# workshop-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: workshop
  annotations:
    # Auto-reload เมื่อ secrets เปลี่ยน
    secret.reloader.stakater.com/reload: "app-database-secret,app-api-keys-secret"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-app
  template:
    metadata:
      labels:
        app: workshop-app
    spec:
      serviceAccountName: workshop-app-sa
      containers:
      - name: app
        image: nginx:alpine
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-database-secret
              key: DATABASE_URL
        envFrom:
        - secretRef:
            name: app-api-keys-secret
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workshop-app-sa
  namespace: workshop
automountServiceAccountToken: false
```

### Step 5: ทดสอบ Secret Rotation

```bash
# อัปเดต secret ใน AWS Secrets Manager
aws secretsmanager put-secret-value \
  --secret-id "workshop/myapp/database" \
  --secret-string '{
    "host": "postgres.production.svc.cluster.local",
    "port": "5432",
    "database": "myapp",
    "username": "appuser",
    "password": "NewSecureP@ssw0rd!2024",
    "ssl_mode": "require"
  }'

# Force sync ExternalSecret
kubectl annotate externalsecret workshop-db-secret \
  force-sync="$(date +%s)" \
  --overwrite \
  -n workshop

# ดู ExternalSecret status
kubectl get externalsecret -n workshop
kubectl describe externalsecret workshop-db-secret -n workshop

# ตรวจสอบ secret อัปเดต
kubectl get secret app-database-secret -n workshop \
  -o jsonpath='{.data.DATABASE_URL}' | base64 -d

# ดู Deployment restart (Reloader จะ trigger อัตโนมัติ)
kubectl get pods -n workshop -w
```

### Step 6: Verification Checklist

```bash
# 1. ตรวจสอบ ESO ทำงานปกติ
kubectl get pods -n external-secrets
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets | tail -20

# 2. ตรวจสอบ SecretStore
kubectl get clustersecretstore
kubectl describe clustersecretstore workshop-aws-store

# 3. ตรวจสอบ ExternalSecrets ทั้งหมด
kubectl get externalsecrets -n workshop
# ต้องเห็น STATUS: SecretSynced

# 4. ตรวจสอบ secrets ถูกสร้างแล้ว
kubectl get secrets -n workshop

# 5. ตรวจสอบ app ทำงานได้
kubectl get pods -n workshop
kubectl logs -n workshop -l app=workshop-app
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sealed Secrets
สร้าง SealedSecret สำหรับ production environment ที่มี:
- `DB_HOST`: db.production.example.com
- `DB_PORT`: 5432
- `DB_NAME`: myproductiondb
- `DB_USER`: produser
- `DB_PASS`: MyP@ssw0rd#2024!

แล้ว deploy application ที่ใช้ secrets เหล่านี้

**เฉลย:**

```bash
# Step 1: สร้าง secret file
cat > /tmp/prod-db-secret.env << EOF
DB_HOST=db.production.example.com
DB_PORT=5432
DB_NAME=myproductiondb
DB_USER=produser
DB_PASS=MyP@ssw0rd#2024!
EOF

# Step 2: สร้าง K8s Secret (dry-run)
kubectl create secret generic prod-db-secret \
  --from-env-file=/tmp/prod-db-secret.env \
  --namespace=production \
  --dry-run=client \
  -o yaml > /tmp/prod-db-secret.yaml

# Step 3: Seal
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --namespace=production \
  --format yaml \
  < /tmp/prod-db-secret.yaml \
  > sealed-prod-db-secret.yaml

# Step 4: Apply
kubectl create namespace production 2>/dev/null || true
kubectl apply -f sealed-prod-db-secret.yaml

# Step 5: ตรวจสอบ
kubectl get sealedsecret -n production
kubectl get secret prod-db-secret -n production
```

```yaml
# Step 6: Deploy app ที่ใช้ secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-app
  template:
    metadata:
      labels:
        app: db-app
    spec:
      containers:
      - name: db-app
        image: postgres:15-alpine
        command: ["psql"]
        args:
        - "$(DATABASE_URL)"
        - "-c"
        - "SELECT 1"
        env:
        - name: DATABASE_URL
          value: "postgresql://$(DB_USER):$(DB_PASS)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)"
        envFrom:
        - secretRef:
            name: prod-db-secret
```

### แบบฝึกหัดที่ 2: External Secrets พร้อม Template

สร้าง ExternalSecret ที่ดึงข้อมูลจาก Vault (หรือ mock ด้วย SecretStore ชนิด Fake) และสร้าง connection string จาก template

```yaml
# เฉลย: ใช้ Fake SecretStore สำหรับทดสอบ
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: fake-store
spec:
  provider:
    fake:
      data:
      - key: "/db/credentials"
        value: '{"username":"admin","password":"secret123","host":"localhost","port":"5432"}'
      - key: "/api/keys"
        value: '{"stripe":"sk_test_xxx","sendgrid":"SG.xxx"}'
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: exercise-secret
  namespace: default
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: fake-store
    kind: ClusterSecretStore
  target:
    name: exercise-app-secret
    template:
      data:
        POSTGRES_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/appdb"
        STRIPE_KEY: "{{ .stripe }}"
  data:
  - secretKey: username
    remoteRef:
      key: /db/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: /db/credentials
      property: password
  - secretKey: host
    remoteRef:
      key: /db/credentials
      property: host
  - secretKey: port
    remoteRef:
      key: /db/credentials
      property: port
  - secretKey: stripe
    remoteRef:
      key: /api/keys
      property: stripe
```

### แบบฝึกหัดที่ 3: Encryption at Rest Verification

ตรวจสอบว่า cluster ของคุณมี encryption at rest เปิดอยู่หรือไม่

```bash
# เฉลย:
# Step 1: สร้าง test secret
kubectl create secret generic enc-test \
  --from-literal=key=my-super-secret-value

# Step 2: ตรวจสอบใน etcd (ต้องการ access ไปที่ master node)
# ถ้าใช้ kind หรือ minikube:
docker exec -it kind-control-plane bash

ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/enc-test | \
  strings | grep -E "(my-super|k8s:enc)"

# ถ้าเห็น "my-super-secret-value" = ยังไม่ encrypted!
# ถ้าเห็น "k8s:enc:aescbc" หรือ "k8s:enc:kms" = encrypted แล้ว!

# Step 3: ล้าง test
kubectl delete secret enc-test
```

---

## สรุปเพิ่มเติม

### เปรียบเทียบเครื่องมือจัดการ Secrets

| Feature | Native K8s Secrets | Sealed Secrets | External Secrets |
|---------|-------------------|----------------|-----------------|
| เก็บใน Git ได้ | ❌ | ✅ | ✅ (เก็บแค่ reference) |
| Encryption | Base64 เท่านั้น | RSA asymmetric | ขึ้นกับ backend |
| Auto-rotation | ❌ | ❌ | ✅ |
| Central management | ❌ | ❌ | ✅ |
| ความซับซ้อน | ต่ำ | ปานกลาง | สูง |
| เหมาะกับ | dev/test | GitOps | enterprise |
| Multi-cluster | ❌ | ❌ | ✅ |

### Security Checklist สำหรับ Production

```bash
# 1. ตรวจสอบ encryption at rest
kubectl get apiserver -o yaml | grep encryption

# 2. ตรวจสอบ RBAC - ไม่มีใครมี wildcard access ไปยัง secrets
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | .subjects'

# 3. ตรวจสอบ secrets ที่ไม่ได้ใช้แล้ว
kubectl get secrets --all-namespaces | grep -v kubernetes.io

# 4. Audit secrets ที่มีขนาดใหญ่ผิดปกติ
kubectl get secrets --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.data | length) keys"' | \
  sort -t: -k2 -rn | head -20

# 5. ตรวจสอบ pods ที่ mount secrets ไม่จำเป็น
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.secret != null) | 
    .metadata.namespace + "/" + .metadata.name'
```

**ต่อไป**: Part 54 - HashiCorp Vault Integration (เพิ่มเติม)
