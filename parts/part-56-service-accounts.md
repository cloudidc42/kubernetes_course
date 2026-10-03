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

---

## ServiceAccount Token Projection - ละเอียด

Projected Volumes รวม Sources หลายอย่างไว้ใน Volume เดียว รองรับ serviceAccountToken, configMap, secret, downwardAPI

### Anatomy ของ Projected ServiceAccount Token

```yaml
# ความแตกต่างระหว่าง token รุ่นเก่าและใหม่
# 
# Legacy token (ไม่มี expiry, ไม่มี audience binding):
# eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
# {
#   "iss": "kubernetes/serviceaccount",
#   "kubernetes.io/serviceaccount/namespace": "default",
#   "kubernetes.io/serviceaccount/service-account.name": "my-sa",
#   "kubernetes.io/serviceaccount/service-account.uid": "abc123"
# }
#
# Projected token (มี expiry, audience binding):
# {
#   "aud": ["https://kubernetes.default.svc"],
#   "exp": 1735689600,
#   "iat": 1704153600,
#   "iss": "https://kubernetes.default.svc.cluster.local",
#   "kubernetes.io": {
#     "namespace": "production",
#     "pod": {"name": "my-pod", "uid": "def456"},
#     "serviceaccount": {"name": "my-sa", "uid": "abc123"}
#   },
#   "nbf": 1704153600,
#   "sub": "system:serviceaccount:production:my-sa"
# }
```

### Projected Volume ครบรูปแบบ

```yaml
# projected-volume-complete.yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-token-demo
  namespace: production
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    # Mount 1: ServiceAccount Token (projected)
    - name: sa-token
      mountPath: /var/run/secrets/tokens
      readOnly: true
    # Mount 2: ConfigMap + ServiceAccount Token รวมกัน
    - name: combined-secrets
      mountPath: /var/run/combined
      readOnly: true
    # Mount 3: DownwardAPI + Token
    - name: pod-info
      mountPath: /var/run/podinfo
      readOnly: true
  volumes:
  # Projected ServiceAccount Token
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600        # ต่ออายุทุก 1 ชั่วโมง
          audience: "https://api.example.com"  # audience เฉพาะ
  # Combined projected volume
  - name: combined-secrets
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200
          audience: "https://kubernetes.default.svc"
      - configMap:
          name: app-config
          items:
          - key: config.yaml
            path: config.yaml
      - secret:
          name: app-tls-secret
          items:
          - key: tls.crt
            path: tls.crt
          - key: tls.key
            path: tls.key
  # DownwardAPI
  - name: pod-info
    projected:
      sources:
      - downwardAPI:
          items:
          - path: pod-name
            fieldRef:
              fieldPath: metadata.name
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
          - path: pod-ip
            fieldRef:
              fieldPath: status.podIP
          - path: node-name
            fieldRef:
              fieldPath: spec.nodeName
          - path: cpu-limit
            resourceFieldRef:
              containerName: app
              resource: limits.cpu
          - path: mem-limit
            resourceFieldRef:
              containerName: app
              resource: limits.memory
```

### Token Rotation และ Refresh

```bash
# ดู token ที่ได้รับ
kubectl exec projected-token-demo -- cat /var/run/secrets/tokens/token | \
  cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# ตรวจสอบว่า kubelet refresh token อัตโนมัติ
# token จะถูก refresh เมื่อ:
# 1. เหลืออายุน้อยกว่า 80% ของ expirationSeconds
# 2. เหลืออายุน้อยกว่า 24 ชั่วโมง (whichever comes first)

# ดู expiry time
kubectl exec projected-token-demo -- \
  cat /var/run/secrets/tokens/token | \
  python3 -c "
import sys, base64, json
token = sys.stdin.read().strip().split('.')[1]
# add padding
token += '=' * (4 - len(token) % 4)
payload = json.loads(base64.b64decode(token))
import datetime
print('Expires:', datetime.datetime.fromtimestamp(payload['exp']))
"
```

### Token สำหรับ Multiple Audiences

```yaml
# ใช้ token ต่างๆ สำหรับ services ต่างๆ
apiVersion: v1
kind: Pod
metadata:
  name: multi-audience-app
spec:
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: k8s-api-token
      mountPath: /var/run/secrets/k8s
      readOnly: true
    - name: aws-token
      mountPath: /var/run/secrets/aws
      readOnly: true
    - name: vault-token
      mountPath: /var/run/secrets/vault
      readOnly: true
  volumes:
  # Token สำหรับ Kubernetes API
  - name: k8s-api-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: "https://kubernetes.default.svc.cluster.local"
          expirationSeconds: 3600
  # Token สำหรับ AWS IRSA
  - name: aws-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: "sts.amazonaws.com"
          expirationSeconds: 86400
  # Token สำหรับ Vault
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: "vault"
          expirationSeconds: 3600
```

---

## Workload Identity - ละเอียด

### GKE Workload Identity

Workload Identity ช่วยให้ Kubernetes ServiceAccount สามารถ impersonate Google Service Account (GSA) ได้ โดยไม่ต้องใช้ service account keys

```bash
# Step 1: Enable Workload Identity บน GKE cluster
gcloud container clusters update my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --region=asia-southeast1

# Step 2: Enable Workload Identity บน Node Pool
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --region=asia-southeast1 \
  --workload-metadata=GKE_METADATA

# Step 3: สร้าง Google Service Account
gcloud iam service-accounts create k8s-workload-sa \
  --description="ServiceAccount for Kubernetes Workload" \
  --display-name="K8s Workload SA"

# Step 4: ให้สิทธิ์ GSA เข้าถึง GCS
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:k8s-workload-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Step 5: Allow KSA to impersonate GSA
gcloud iam service-accounts add-iam-policy-binding \
  k8s-workload-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[production/my-app-ksa]"
```

```yaml
# Step 6: Annotate Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-ksa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: k8s-workload-sa@PROJECT_ID.iam.gserviceaccount.com
---
# Step 7: Pod ที่ใช้ Workload Identity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gcs-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gcs-app
  template:
    metadata:
      labels:
        app: gcs-app
    spec:
      serviceAccountName: my-app-ksa
      containers:
      - name: app
        image: google/cloud-sdk:slim
        command: ["sh", "-c"]
        args:
        - |
          # ไม่ต้องใส่ credentials อะไรเลย!
          gsutil ls gs://my-bucket/
          sleep 3600
```

```bash
# Verify Workload Identity ทำงาน
kubectl exec -n production gcs-app-xxxx -- \
  gcloud auth list
# ควรเห็น k8s-workload-sa@PROJECT_ID.iam.gserviceaccount.com active
```

### EKS IRSA (IAM Roles for Service Accounts)

```bash
# Step 1: Enable OIDC Provider บน EKS cluster
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --region ap-southeast-1 \
  --approve

# ดู OIDC endpoint
aws eks describe-cluster \
  --name my-eks-cluster \
  --region ap-southeast-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text

# Step 2: สร้าง IAM Role
OIDC_PROVIDER=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --region ap-southeast-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed -e "s/^https:\/\///")

# สร้าง trust policy
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:production:s3-app-sa",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# สร้าง IAM Role
aws iam create-role \
  --role-name EKS-S3-App-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach Policy
aws iam attach-role-policy \
  --role-name EKS-S3-App-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

```yaml
# Step 3: Annotate ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT_ID:role/EKS-S3-App-Role"
    # Optional: ตั้ง token expiry
    eks.amazonaws.com/token-expiration: "86400"
---
# Step 4: Deploy Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: s3-app
  template:
    metadata:
      labels:
        app: s3-app
    spec:
      serviceAccountName: s3-app-sa
      containers:
      - name: app
        image: amazon/aws-cli
        command: ["sh", "-c"]
        args:
        - |
          # ไม่ต้องใส่ AWS credentials!
          aws s3 ls s3://my-bucket/
          sleep 3600
        env:
        - name: AWS_DEFAULT_REGION
          value: ap-southeast-1
```

```bash
# ตรวจสอบ IRSA ทำงาน
kubectl exec -n production s3-app-xxxx -- aws sts get-caller-identity
# ควรเห็น Role ARN ที่ assign ไป
```

---

## ServiceAccount Security Best Practices

### 1. Disable Automounting สำหรับทุก Service ที่ไม่ต้องการ K8s API

```yaml
# Disable ที่ ServiceAccount level (apply ทุก pods ที่ใช้ SA นี้)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stateless-app-sa
  namespace: production
automountServiceAccountToken: false  # ปิด default
---
# Override สำหรับ pod เฉพาะที่ต้องการ
apiVersion: v1
kind: Pod
metadata:
  name: needs-api-access
spec:
  serviceAccountName: stateless-app-sa
  automountServiceAccountToken: true  # เปิดเฉพาะ pod นี้
  containers:
  - name: app
    image: my-app:latest
```

### 2. One ServiceAccount Per Workload

```yaml
# ❌ Bad: ใช้ default SA ร่วมกัน
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      # ไม่ระบุ serviceAccountName = ใช้ default

---
# ✅ Good: แต่ละ workload มี SA ของตัวเอง
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-sa
  namespace: production
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: production
automountServiceAccountToken: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      serviceAccountName: frontend-sa
      automountServiceAccountToken: false
```

### 3. Minimal RBAC สำหรับแต่ละ ServiceAccount

```yaml
# RBAC ที่ specific ที่สุดเท่าที่เป็นไปได้
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]   # เฉพาะ configmap ชื่อนี้
  verbs: ["get"]                  # แค่ get ไม่ใช่ list/watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-configmap-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: frontend-sa
  namespace: production
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4. Audit ServiceAccount Permissions

```bash
# ตรวจสอบ permissions ของ ServiceAccount
kubectl auth can-i --list \
  --as=system:serviceaccount:production:frontend-sa

# ดู RoleBindings ทั้งหมดที่ผูกกับ SA
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces \
  -o json | \
  jq -r '
    .items[] | 
    select(.subjects[]? | 
      select(.kind == "ServiceAccount" and 
             .name == "frontend-sa" and 
             .namespace == "production")
    ) | 
    .metadata.namespace + "/" + .metadata.name + ": " + .roleRef.name
  '

# ตรวจสอบว่า SA ไม่มี cluster-admin
kubectl get clusterrolebindings -o json | \
  jq -r '
    .items[] | 
    select(.roleRef.name == "cluster-admin") | 
    select(.subjects[]? | select(.kind == "ServiceAccount")) |
    .metadata.name + ": " + 
    (.subjects[] | select(.kind == "ServiceAccount") | .namespace + "/" + .name)
  '
```

### 5. ServiceAccount Network Policy

```yaml
# จำกัด network access ของ pods ที่ใช้ SA เฉพาะ
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: load-balancer
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # ออกไปหา backend เท่านั้น
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## Automounting vs Manual Token

### ความแตกต่าง

```bash
# Default behavior (automount=true):
# - /var/run/secrets/kubernetes.io/serviceaccount/token
# - /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
# - /var/run/secrets/kubernetes.io/serviceaccount/namespace
# Token นี้เป็น legacy token ที่ไม่มี expiry ใน K8s เก่า
# ใน K8s 1.21+ token มี expiry แต่ยังไม่มี audience binding

# Manual projection (แนะนำ):
# ใช้ projected volume กำหนด audience และ expiry เอง
```

```yaml
# Manual Token Projection - Full Example
apiVersion: v1
kind: Pod
metadata:
  name: manual-token-app
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: false  # ปิด auto-mount
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: k8s-api-token
      mountPath: /var/run/secrets/k8s-api
      readOnly: true
  volumes:
  - name: k8s-api-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: "https://kubernetes.default.svc.cluster.local"
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
```

### การใช้ Token ใน Application

```python
# Python example - ใช้ projected token
import os
import time

TOKEN_PATH = "/var/run/secrets/k8s-api/token"
CA_PATH = "/var/run/secrets/k8s-api/ca.crt"

def get_current_token():
    """Read token - kubelet จะ refresh อัตโนมัติ"""
    with open(TOKEN_PATH, 'r') as f:
        return f.read().strip()

def make_api_call():
    """ทุกครั้งที่ call API ให้อ่าน token ใหม่"""
    token = get_current_token()
    import urllib.request
    req = urllib.request.Request(
        "https://kubernetes.default.svc.cluster.local/api/v1/namespaces/default/pods",
        headers={"Authorization": f"Bearer {token}"}
    )
    # ใช้ CA cert สำหรับ verify
    import ssl
    ctx = ssl.create_default_context(cafile=CA_PATH)
    with urllib.request.urlopen(req, context=ctx) as response:
        return response.read()
```

---

## Workshop: Secure App Authentication

### โจทย์
Deploy microservice ที่:
1. มี ServiceAccount ของตัวเอง
2. ใช้ IRSA เพื่อ access S3 (EKS) หรือ Workload Identity (GKE)
3. ใช้ projected token สำหรับ Vault auth
4. ไม่มี static credentials ใดๆ เลย

### Step 1: สร้าง Namespace และ ServiceAccount

```yaml
# secure-app-setup.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-app
  labels:
    environment: production
    team: platform
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-app-sa
  namespace: secure-app
  annotations:
    # EKS IRSA
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT_ID:role/SecureAppRole"
    # GKE Workload Identity (ใช้อย่างใดอย่างหนึ่ง)
    # iam.gke.io/gcp-service-account: secure-app@PROJECT.iam.gserviceaccount.com
  labels:
    app: secure-app
    version: v1
automountServiceAccountToken: false   # Manual mounting
```

### Step 2: สร้าง RBAC ที่จำเป็น

```yaml
# secure-app-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secure-app-role
  namespace: secure-app
rules:
# อ่าน configmaps ที่จำเป็น
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config", "feature-flags"]
  verbs: ["get"]
# ดู pod ของตัวเองได้ (สำหรับ health checks)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]
  resourceNames: []  # จำกัดด้วย fieldSelector ใน code
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secure-app-binding
  namespace: secure-app
subjects:
- kind: ServiceAccount
  name: secure-app-sa
  namespace: secure-app
roleRef:
  kind: Role
  name: secure-app-role
  apiGroup: rbac.authorization.k8s.io
```

### Step 3: Deploy Application

```yaml
# secure-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: secure-app
  labels:
    app: secure-app
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
        version: v1
      annotations:
        # Vault injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "secure-app"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/secure-app/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/secure-app/config" -}}
          DB_URL={{ .Data.data.db_url }}
          REDIS_URL={{ .Data.data.redis_url }}
          {{- end -}}
    spec:
      serviceAccountName: secure-app-sa
      automountServiceAccountToken: false  # Manual
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: secure-app:v1
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: AWS_DEFAULT_REGION
          value: ap-southeast-1
        volumeMounts:
        # K8s API token (audience: kubernetes)
        - name: k8s-token
          mountPath: /var/run/secrets/kubernetes
          readOnly: true
        # AWS token (audience: sts.amazonaws.com)
        - name: aws-token
          mountPath: /var/run/secrets/aws
          readOnly: true
        # Vault secrets
        - name: vault-secrets
          mountPath: /vault/secrets
          readOnly: true
        # Temp directory (readOnlyRootFilesystem)
        - name: tmp
          mountPath: /tmp
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
      volumes:
      # K8s API Access
      - name: k8s-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: "https://kubernetes.default.svc.cluster.local"
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
      # AWS IRSA Access
      - name: aws-token
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 86400
              audience: "sts.amazonaws.com"
      # Vault secrets (injected by vault-agent)
      - name: vault-secrets
        emptyDir:
          medium: Memory
      # Temp
      - name: tmp
        emptyDir:
          medium: Memory
          sizeLimit: "50Mi"
```

### Step 4: Verification Script

```bash
#!/bin/bash
# verify-secure-app.sh

NAMESPACE="secure-app"
APP="secure-app"

echo "=== Verifying Secure App Setup ==="

# 1. Check ServiceAccount
echo -e "\n1. ServiceAccount:"
kubectl get sa -n $NAMESPACE $APP-sa -o yaml | \
  grep -E "(automountServiceAccountToken|eks.amazonaws.com|iam.gke.io)"

# 2. Check RBAC
echo -e "\n2. RBAC Bindings:"
kubectl get rolebindings -n $NAMESPACE -l "" -o wide

# 3. Check Pod Security
echo -e "\n3. Pod Security Context:"
kubectl get pod -n $NAMESPACE -l app=$APP -o jsonpath=\
  '{range .items[*]}{.metadata.name}{"\n"}{.spec.securityContext}{"\n"}{end}'

# 4. Check Volume Mounts
echo -e "\n4. Volume Mounts:"
kubectl get pod -n $NAMESPACE -l app=$APP -o jsonpath=\
  '{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[0].volumeMounts[*]}  {.name}: {.mountPath}{"\n"}{end}{end}'

# 5. Test K8s API access (should work)
POD=$(kubectl get pod -n $NAMESPACE -l app=$APP -o jsonpath='{.items[0].metadata.name}')
echo -e "\n5. K8s API Test:"
kubectl exec -n $NAMESPACE $POD -c app -- \
  wget -qO- \
    --header="Authorization: Bearer $(cat /var/run/secrets/kubernetes/token)" \
    --ca-certificate=/var/run/secrets/kubernetes/ca.crt \
    "https://kubernetes.default.svc.cluster.local/api/v1/namespaces/secure-app/configmaps/app-config" \
  2>&1 | head -5

# 6. Test AWS Access (ถ้า EKS)
echo -e "\n6. AWS Identity:"
kubectl exec -n $NAMESPACE $POD -c app -- \
  aws sts get-caller-identity 2>&1 || echo "AWS test skipped (not EKS)"

# 7. Verify no static credentials
echo -e "\n7. Checking for static credentials:"
kubectl get secrets -n $NAMESPACE | \
  grep -v "kubernetes.io\|sealed\|service-account-token" || \
  echo "✅ No static credential secrets found"

echo -e "\n=== Verification Complete ==="
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ServiceAccount Audit

ค้นหา ServiceAccounts ที่มี automount เปิดและมีสิทธิ์ cluster-admin

```bash
# เฉลย:
# ขั้นตอนที่ 1: หา SAs ที่ automount เปิด
kubectl get sa --all-namespaces -o json | jq -r '
  .items[] | 
  select(.automountServiceAccountToken != false) | 
  .metadata.namespace + "/" + .metadata.name
' | sort

# ขั้นตอนที่ 2: หา SAs ที่ผูกกับ cluster-admin
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | 
  select(.roleRef.name == "cluster-admin") | 
  .subjects[]? | 
  select(.kind == "ServiceAccount") | 
  .namespace + "/" + .name
'

# ขั้นตอนที่ 3: Fix - ปิด automount และลด permissions
# สำหรับแต่ละ SA ที่พบ:
kubectl patch sa <sa-name> -n <namespace> \
  -p '{"automountServiceAccountToken": false}'
```

### แบบฝึกหัดที่ 2: Projected Token Lab

สร้าง pod ที่ใช้ projected token สำหรับ 2 purposes:
1. Kubernetes API access (expirationSeconds: 3600)
2. Custom service authentication (audience: "my-internal-service", expirationSeconds: 300)

```yaml
# เฉลย:
apiVersion: v1
kind: Pod
metadata:
  name: projected-token-lab
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  containers:
  - name: test
    image: alpine
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: k8s-token
      mountPath: /var/run/k8s
      readOnly: true
    - name: service-token
      mountPath: /var/run/service
      readOnly: true
  volumes:
  - name: k8s-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: "https://kubernetes.default.svc.cluster.local"
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
  - name: service-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 300
          audience: "my-internal-service"
```

```bash
# ตรวจสอบ tokens
kubectl apply -f projected-token-lab.yaml
kubectl exec projected-token-lab -- cat /var/run/k8s/token | \
  python3 -c "
import sys, base64, json
t = sys.stdin.read().strip().split('.')
payload = json.loads(base64.b64decode(t[1] + '=='))
print('K8s token audience:', payload.get('aud'))
print('Expires in:', payload.get('exp'))
"

kubectl exec projected-token-lab -- cat /var/run/service/token | \
  python3 -c "
import sys, base64, json
t = sys.stdin.read().strip().split('.')
payload = json.loads(base64.b64decode(t[1] + '=='))
print('Service token audience:', payload.get('aud'))
"
```

### แบบฝึกหัดที่ 3: Workload Identity Simulation

จำลอง Workload Identity pattern โดยไม่ใช้ cloud provider (สำหรับ local testing)

```bash
# เฉลย: ใช้ Keycloak เพื่อ simulate OIDC provider
# Step 1: Deploy Keycloak
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install keycloak bitnami/keycloak \
  --namespace keycloak \
  --create-namespace \
  --set auth.adminUser=admin \
  --set auth.adminPassword=adminpass

# Step 2: สร้าง Policy ที่รับ K8s projected tokens
# (ใน real scenario นี้คือส่วนของ identity federation)
cat > /tmp/token-review.sh << 'EOF'
#!/bin/bash
# Test token review
TOKEN=$(kubectl create token my-app-sa -n production)
kubectl create -f - << YAML
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: $TOKEN
  audiences:
  - "https://kubernetes.default.svc.cluster.local"
YAML
EOF
chmod +x /tmp/token-review.sh
bash /tmp/token-review.sh
```

---

## สรุปเพิ่มเติม

### Security Checklist สำหรับ ServiceAccounts

```bash
# รัน script ตรวจสอบ security posture
cat << 'EOF' > /tmp/sa-security-check.sh
#!/bin/bash
echo "=== ServiceAccount Security Audit ==="

echo -e "\n[1] SAs with automount enabled:"
kubectl get sa --all-namespaces -o json | jq -r '
  .items[] | 
  select(
    .automountServiceAccountToken == null or 
    .automountServiceAccountToken == true
  ) | 
  .metadata.namespace + "/" + .metadata.name
' | grep -v kube-system | sort

echo -e "\n[2] Pods using default SA:"
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] | 
  select(.spec.serviceAccountName == "default" or .spec.serviceAccountName == null) |
  .metadata.namespace + "/" + .metadata.name
' | sort

echo -e "\n[3] ClusterRoleBindings with SAs:"
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | 
  .metadata.name + ": " + .roleRef.name + " -> " + 
  ([.subjects[]? | select(.kind == "ServiceAccount") | .namespace + "/" + .name] | join(", "))
' | grep "ServiceAccount" | sort

echo -e "\n[4] SA tokens in secrets (legacy):"
kubectl get secrets --all-namespaces --field-selector type=kubernetes.io/service-account-token \
  -o json | jq -r '.items[] | .metadata.namespace + "/" + .metadata.name'

echo -e "\n=== Audit Complete ==="
EOF
bash /tmp/sa-security-check.sh
```

**ต่อไป**: Part 57 - Security Contexts
