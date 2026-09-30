# Part 56: Service Accounts

## บทนำ

Service Account เป็น identity สำหรับ processes ที่รันใน Pods เพื่อ authenticate กับ Kubernetes API Server และ external services ต่างจาก User Accounts ที่เป็น identity ของมนุษย์ Service Account เป็น identity ของ applications/workloads

---

## 56.1 ServiceAccount คืออะไร

### ความแตกต่างระหว่าง User Account และ Service Account

```
User Account:
- สำหรับมนุษย์ (developers, admins)
- จัดการนอก cluster (LDAP, OIDC, certificates)
- ไม่ถูกสร้างผ่าน Kubernetes API

Service Account:
- สำหรับ processes ที่รันใน Pods
- สร้างและจัดการใน Kubernetes
- Namespace-scoped
- มี token สำหรับ authenticate กับ API server
```

### Default Service Account

```bash
# ทุก namespace มี "default" ServiceAccount
kubectl get serviceaccount -n default
kubectl get serviceaccount -n kube-system

# Pods ที่ไม่ระบุ ServiceAccount จะใช้ "default" อัตโนมัติ
kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}'
```

### ServiceAccount Token

```bash
# ดู ServiceAccount ที่มีอยู่
kubectl describe serviceaccount default -n default

# ดู token (Kubernetes 1.24+ ใช้ projected tokens)
# ดู secrets ที่เกี่ยวข้อง (เก่า)
kubectl get secrets -n default | grep default-token

# ใหม่ (1.24+): tokens ถูก project โดยตรงใน pod
kubectl exec -it <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

---

## 56.2 สร้างและใช้งาน Service Accounts

### สร้าง ServiceAccount

```yaml
# basic-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  labels:
    app: myapp
    team: platform
  annotations:
    description: "Service Account for MyApp"
    # สำหรับ AWS IRSA
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/MyAppRole"
# ไม่ auto-mount token โดยปริยาย
automountServiceAccountToken: false
```

```bash
kubectl apply -f basic-serviceaccount.yaml
kubectl get serviceaccount myapp-sa -n production
kubectl describe serviceaccount myapp-sa -n production
```

### ใช้ ServiceAccount ใน Pod

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
    spec:
      # ระบุ ServiceAccount
      serviceAccountName: myapp-sa
      
      # ควบคุมการ mount token
      automountServiceAccountToken: true  # เปิดใช้ที่ pod level
      
      containers:
      - name: myapp
        image: myapp:1.0.0
        # Token จะถูก mount ที่:
        # /var/run/secrets/kubernetes.io/serviceaccount/token
        # /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        # /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

### ตรวจสอบ Token ใน Container

```bash
POD=$(kubectl get pod -l app=myapp -n production -o jsonpath='{.items[0].metadata.name}')

# ดู token
kubectl exec -it $POD -n production -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token

# ดู namespace
kubectl exec -it $POD -n production -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/namespace

# Decode token (JWT)
TOKEN=$(kubectl exec -it $POD -n production -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Decode payload (ไม่ verify signature)
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | jq '.'
```

---

## 56.3 Auto-mounting Tokens

### ปัญหาของ Auto-mounting

```yaml
# ค่า default คือ automountServiceAccountToken: true
# ทำให้ทุก pod ได้รับ token โดยอัตโนมัติ
# ถ้า pod ถูก compromise ผู้โจมตีได้ token ไปใช้

# แนวทางที่ดีกว่า: ปิด auto-mount แล้วเปิดเมื่อจำเป็น
```

### ปิด Auto-mount ที่ ServiceAccount Level

```yaml
# ปิดสำหรับทุก pod ที่ใช้ SA นี้
apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: production
automountServiceAccountToken: false  # ปิด
```

### ปิด Auto-mount ที่ Pod Level

```yaml
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false  # override SA setting
  containers:
  - name: myapp
    # ไม่มี token mount
```

### เปิด Auto-mount เฉพาะที่จำเป็น

```yaml
# ServiceAccount ปิด auto-mount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: safe-sa
automountServiceAccountToken: false

---
# Pod ที่ต้องการ token - เปิด override
spec:
  serviceAccountName: safe-sa
  automountServiceAccountToken: true  # เปิดเฉพาะ pod นี้
```

### Projected Service Account Tokens (Kubernetes 1.20+)

```yaml
# Projected tokens มี TTL และ audience
spec:
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600    # หมดอายุ 1 ชั่วโมง
          audience: "vault"          # token สำหรับ Vault เท่านั้น
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
  
  containers:
  - name: app
    volumeMounts:
    - name: token-vol
      mountPath: /var/run/secrets/tokens
      readOnly: true
```

---

## 56.4 OIDC Integration

### OpenID Connect (OIDC) Authentication

OIDC ช่วยให้ Kubernetes สามารถ authenticate users จาก external identity providers เช่น:
- Google Workspace
- Azure AD
- Okta
- Dex
- Keycloak

### Configure kube-apiserver สำหรับ OIDC

```bash
# เพิ่ม flags ใน kube-apiserver:
# --oidc-issuer-url=https://accounts.google.com
# --oidc-client-id=kubernetes
# --oidc-username-claim=email
# --oidc-groups-claim=groups

# สำหรับ kubeadm
cat > /etc/kubernetes/kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  extraArgs:
    oidc-issuer-url: "https://accounts.google.com"
    oidc-client-id: "kubernetes"
    oidc-username-claim: "email"
    oidc-groups-claim: "groups"
    oidc-username-prefix: "oidc:"
    oidc-groups-prefix: "oidc:"
EOF
```

### Dex: Identity Broker

```yaml
# dex-config.yaml - Deploy Dex เป็น OIDC provider
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex-config
  namespace: dex
data:
  config.yaml: |
    issuer: https://dex.example.com
    
    storage:
      type: kubernetes
      config:
        inCluster: true
    
    web:
      http: 0.0.0.0:5556
    
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: $GITHUB_CLIENT_ID
        clientSecret: $GITHUB_CLIENT_SECRET
        redirectURI: https://dex.example.com/callback
        orgs:
        - name: my-github-org
    
    oauth2:
      skipApprovalScreen: true
    
    staticClients:
    - id: kubernetes
      redirectURIs:
      - 'urn:ietf:wg:oauth:2.0:oob'
      - 'http://localhost:8000'
      name: 'Kubernetes'
      secret: kubernetes-client-secret
    
    - id: gangway
      redirectURIs:
      - https://gangway.example.com/callback
      name: 'Gangway'
      secret: gangway-client-secret
```

### Workload Identity Federation (AWS IRSA)

```yaml
# ใช้ ServiceAccount กับ AWS IAM Role
# 1. สร้าง IAM Role ที่ trust EKS OIDC provider
# 2. Annotate ServiceAccount

apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  namespace: production
  annotations:
    # IAM Role ที่จะ assume
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/S3AccessRole"
    # Token expiry
    eks.amazonaws.com/token-expiration: "86400"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-app
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: s3-access-sa
      containers:
      - name: app
        image: my-s3-app:1.0
        env:
        - name: AWS_REGION
          value: "ap-southeast-1"
        # Token จะถูก inject อัตโนมัติโดย EKS
        # และ AWS SDK จะใช้ token นี้ assume role
```

### Google Workload Identity

```yaml
# GKE Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gcp-sa
  namespace: production
  annotations:
    # Google Service Account
    iam.gke.io/gcp-service-account: "my-gsa@my-project.iam.gserviceaccount.com"
```

```bash
# ผูก Kubernetes SA กับ Google Service Account
gcloud iam service-accounts add-iam-policy-binding \
  my-gsa@my-project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[production/gcp-sa]"
```

---

## 56.5 Workshop: Secure ServiceAccount Setup

### สถานการณ์

เราจะสร้าง multi-tier application ที่มี security requirements:
- Frontend: อ่าน ConfigMap, ไม่ต้องการ k8s API access
- Backend: อ่าน Secrets เฉพาะของตัวเอง
- Batch Job: สร้างและดู Jobs
- Monitoring: อ่าน metrics ทั้ง cluster

### Step 1: Setup

```bash
kubectl create namespace workshop-sa
kubectl config set-context --current --namespace=workshop-sa
```

### Step 2: สร้าง ServiceAccounts

```yaml
# service-accounts.yaml
# Frontend - ไม่ต้องการ k8s API access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-sa
  namespace: workshop-sa
  labels:
    app: frontend
    tier: frontend
automountServiceAccountToken: false  # ไม่ mount token

---
# Backend - ต้องการอ่าน secrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: workshop-sa
  labels:
    app: backend
    tier: backend
automountServiceAccountToken: false  # จะ mount แบบ projected token

---
# Batch Job - ต้องการสร้าง jobs
apiVersion: v1
kind: ServiceAccount
metadata:
  name: batch-job-sa
  namespace: workshop-sa
  labels:
    app: batch
    tier: jobs
automountServiceAccountToken: true  # ต้องการ token

---
# Monitoring - อ่าน cluster metrics
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: workshop-sa
  labels:
    app: monitoring
automountServiceAccountToken: true
```

```bash
kubectl apply -f service-accounts.yaml
kubectl get serviceaccounts -n workshop-sa
```

### Step 3: สร้าง RBAC

```yaml
# rbac-workshop.yaml

# Backend Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: workshop-sa
rules:
# อ่าน secrets เฉพาะของตัวเอง
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["backend-db-secret", "backend-api-secret"]
  verbs: ["get"]
# อ่าน configmaps
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["backend-config"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-binding
  namespace: workshop-sa
subjects:
- kind: ServiceAccount
  name: backend-sa
  namespace: workshop-sa
roleRef:
  kind: Role
  name: backend-role
  apiGroup: rbac.authorization.k8s.io

---
# Batch Job Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: batch-role
  namespace: workshop-sa
rules:
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: batch-binding
  namespace: workshop-sa
subjects:
- kind: ServiceAccount
  name: batch-job-sa
  namespace: workshop-sa
roleRef:
  kind: Role
  name: batch-role
  apiGroup: rbac.authorization.k8s.io

---
# Monitoring ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: workshop-monitoring-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: workshop-monitoring-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: workshop-sa
roleRef:
  kind: ClusterRole
  name: workshop-monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac-workshop.yaml
```

### Step 4: สร้าง Secrets และ ConfigMaps

```bash
# Backend secrets
kubectl create secret generic backend-db-secret \
  -n workshop-sa \
  --from-literal=DB_HOST=postgres.workshop-sa.svc.local \
  --from-literal=DB_PASSWORD='BackendDB@Pass'

kubectl create secret generic backend-api-secret \
  -n workshop-sa \
  --from-literal=API_KEY='backend-api-key-abc123'

# Backend configmap
kubectl create configmap backend-config \
  -n workshop-sa \
  --from-literal=APP_ENV=development \
  --from-literal=LOG_LEVEL=debug
```

### Step 5: Deploy Applications

```yaml
# deployments-workshop.yaml

# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: workshop-sa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      serviceAccountName: frontend-sa
      automountServiceAccountToken: false  # ไม่ mount token
      containers:
      - name: frontend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "32Mi"
            cpu: "50m"

---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: workshop-sa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: backend-sa
      # ใช้ projected token แทน auto-mount
      volumes:
      - name: kube-api-access
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: api
          - configMap:
              name: kube-root-ca.crt
              items:
              - key: ca.crt
                path: ca.crt
          - downwardAPI:
              items:
              - path: namespace
                fieldRef:
                  fieldPath: metadata.namespace
      
      containers:
      - name: backend
        image: nginx:1.25-alpine
        
        # Secrets จาก RBAC
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: backend-db-secret
              key: DB_HOST
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-db-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: backend-api-secret
              key: API_KEY
        
        volumeMounts:
        - name: kube-api-access
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          readOnly: true
        
        resources:
          limits:
            memory: "64Mi"
            cpu: "100m"
```

```bash
kubectl apply -f deployments-workshop.yaml
kubectl get pods -n workshop-sa
```

### Step 6: ทดสอบ

```bash
# ทดสอบ Frontend - ไม่ควรมี token
FRONTEND_POD=$(kubectl get pod -l app=frontend -n workshop-sa -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $FRONTEND_POD -n workshop-sa -- ls /var/run/secrets/ 2>/dev/null || echo "No secrets mounted (expected)"

# ทดสอบ Backend - ควรมี token และ secrets
BACKEND_POD=$(kubectl get pod -l app=backend -n workshop-sa -o jsonpath='{.items[0].metadata.name}')

# ตรวจสอบ token มีอยู่
kubectl exec -it $BACKEND_POD -n workshop-sa -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# token  ca.crt  namespace

# ตรวจสอบ token audience
TOKEN=$(kubectl exec -it $BACKEND_POD -n workshop-sa -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool 2>/dev/null || echo $TOKEN | cut -d'.' -f2 | base64 -d

# ทดสอบ permissions
echo "=== Backend SA permissions ==="
kubectl auth can-i get secret/backend-db-secret \
  --as=system:serviceaccount:workshop-sa:backend-sa \
  -n workshop-sa
# yes

kubectl auth can-i list secrets \
  --as=system:serviceaccount:workshop-sa:backend-sa \
  -n workshop-sa
# no

kubectl auth can-i get secret/other-secret \
  --as=system:serviceaccount:workshop-sa:backend-sa \
  -n workshop-sa
# no (เฉพาะ resourceNames ที่ระบุ)

echo "=== Frontend SA permissions ==="
kubectl auth can-i get pods \
  --as=system:serviceaccount:workshop-sa:frontend-sa \
  -n workshop-sa
# no (ไม่มี role)

echo "=== Monitoring SA permissions ==="
kubectl auth can-i get pods --all-namespaces \
  --as=system:serviceaccount:workshop-sa:monitoring-sa
# yes (cluster-wide)

kubectl auth can-i create pods \
  --as=system:serviceaccount:workshop-sa:monitoring-sa \
  -n workshop-sa
# no
```

### Step 7: Token Rotation

```bash
# ดู token expiry (projected tokens มี TTL)
BACKEND_POD=$(kubectl get pod -l app=backend -n workshop-sa -o jsonpath='{.items[0].metadata.name}')
TOKEN=$(kubectl exec -it $BACKEND_POD -n workshop-sa -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Decode expiry
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -c "
import sys, json, datetime
payload = json.loads(sys.stdin.read())
if 'exp' in payload:
    exp = datetime.datetime.fromtimestamp(payload['exp'])
    print(f'Token expires: {exp}')
    print(f'Audience: {payload.get(\"aud\", \"default\")}')
print(f'Service Account: {payload.get(\"sub\", \"unknown\")}')
" 2>/dev/null || echo "Cannot decode token"

# kubelet จะ rotate tokens อัตโนมัติก่อนหมดอายุ
```

### Step 8: Cleanup

```bash
kubectl delete namespace workshop-sa
kubectl delete clusterrole workshop-monitoring-reader
kubectl delete clusterrolebinding workshop-monitoring-binding
echo "Workshop cleanup complete!"
```

---

## 56.6 ServiceAccount Security Best Practices

### 1. ปิด Auto-mount สำหรับ Apps ที่ไม่ต้องการ API access

```yaml
# ปิดที่ ServiceAccount level
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-sa
automountServiceAccountToken: false

# และปิดที่ Pod level ด้วย (defense in depth)
spec:
  automountServiceAccountToken: false
```

### 2. อย่าใช้ default ServiceAccount

```yaml
# ห้ามปล่อย pods ใช้ default SA
# ให้สร้าง dedicated SA สำหรับแต่ละ application

# ตรวจสอบ pods ที่ใช้ default SA
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceAccountName}{"\n"}{end}'
```

### 3. ใช้ Projected Tokens

```yaml
# Projected tokens ดีกว่า legacy tokens เพราะ:
# - มี TTL
# - Audience-specific
# - kubelet rotate อัตโนมัติ
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600    # 1 hour
        audience: myapp
```

### 4. ตรวจสอบ ServiceAccount เป็นประจำ

```bash
# ดู ServiceAccounts ทั้งหมดที่มี auto-mount เปิด
kubectl get serviceaccounts --all-namespaces -o json | \
  jq -r '.items[] | 
    select(.automountServiceAccountToken != false) | 
    .metadata.namespace + "/" + .metadata.name'

# ดู pods ที่ mount token
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | 
    select(.spec.automountServiceAccountToken != false) | 
    .metadata.namespace + "/" + .metadata.name'
```

### 5. Workload Identity สำหรับ Cloud Provider

```yaml
# AWS IRSA - ดีกว่าการใช้ static credentials ใน secrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-uploader-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123:role/S3UploaderRole"
    # ไม่ต้องใส่ AWS credentials ใน secrets!
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ServiceAccount Basics**: identity สำหรับ pods ต่างจาก user accounts
2. **Auto-mounting**: ควรปิด auto-mount สำหรับ apps ที่ไม่ต้องการ API access
3. **Projected Tokens**: ดีกว่า legacy tokens มี TTL และ audience
4. **OIDC Integration**: AWS IRSA, GKE Workload Identity
5. **Least Privilege**: ให้สิทธิ์เฉพาะที่จำเป็น

**Key Takeaways:**
- แต่ละ application ควรมี ServiceAccount ของตัวเอง
- ปิด automountServiceAccountToken ถ้าไม่ต้องการ
- ใช้ Projected Tokens สำหรับ security
- Workload Identity ดีกว่า static cloud credentials

---

**ต่อไป**: Part 57 - Security Contexts
