# Part 16: Namespaces - จัดระเบียบ Kubernetes Cluster

## สารบัญ
1. [Namespace คืออะไร](#namespace-คืออะไร)
2. [Default Namespaces](#default-namespaces)
3. [สร้างและจัดการ Namespace](#สร้างและจัดการ-namespace)
4. [Resource Quota ต่อ Namespace](#resource-quota)
5. [Workshop: จัดระเบียบ Apps ด้วย Namespaces](#workshop)

---

## 1. Namespace คืออะไร

**Namespace** คือ mechanism สำหรับแบ่งแยก (isolate) Resources ภายใน Kubernetes Cluster เดียวกัน ออกเป็นกลุ่มๆ

### เปรียบเทียบ Namespace

```
Kubernetes Cluster
├── Namespace: production
│   ├── Deployment: webapp (nginx:1.25)
│   ├── Service: webapp-service
│   └── Pod: webapp-xxx
│
├── Namespace: staging
│   ├── Deployment: webapp (nginx:1.24-beta)
│   ├── Service: webapp-service
│   └── Pod: webapp-yyy
│
└── Namespace: development
    ├── Deployment: webapp (nginx:latest)
    └── Service: webapp-service
```

เหมือนกับ Namespaces ใน Programming:
- แต่ละ Namespace มี "scope" ของตัวเอง
- ชื่อ Resource ต้อง unique ภายใน Namespace เดียวกัน
- แต่ Resource ใน Namespace ต่างกันสามารถชื่อเดียวกันได้

### ทำไมต้องใช้ Namespaces

1. **Multi-team/Project Isolation**: แยก Resources ของแต่ละทีม
2. **Environment Separation**: แยก prod/staging/dev
3. **Resource Management**: กำหนด Resource Quota ต่อ Namespace
4. **Access Control**: กำหนด RBAC per Namespace
5. **Network Isolation**: ใช้ NetworkPolicy กำหนด traffic rules

### สิ่งที่ Namespaced vs Non-namespaced

```bash
# ดู Resources ที่ namespaced
kubectl api-resources --namespaced=true

# รายการสำคัญที่ namespaced:
# - pods, services, deployments, replicasets
# - configmaps, secrets
# - serviceaccounts, roles, rolebindings
# - persistentvolumeclaims
# - ingresses, networkpolicies

# ดู Resources ที่ NOT namespaced (Cluster-level)
kubectl api-resources --namespaced=false

# รายการสำคัญที่ NOT namespaced:
# - nodes
# - persistentvolumes
# - clusterroles, clusterrolebindings
# - storageclasses
# - namespaces (namespace ไม่ใช้ namespace!)
```

---

## 2. Default Namespaces

Kubernetes มี Namespaces ที่สร้างมาให้โดย default 4 ตัว:

### default

```bash
# Namespace ที่ใช้เมื่อไม่ระบุ namespace
# ใช้สำหรับ workloads ที่ไม่ได้กำหนด namespace

kubectl get pods         # ดู pods ใน default namespace
kubectl get pods -n default   # เหมือนกัน
```

### kube-system

```bash
# Namespace สำหรับ Kubernetes system components
kubectl get pods -n kube-system

# รายการ Pods ที่พบได้:
# kube-apiserver           - API Server
# kube-controller-manager  - Controller Manager
# kube-scheduler           - Scheduler
# etcd                     - Key-Value Store
# coredns                  - DNS Server
# kube-proxy               - Network proxy

kubectl describe namespace kube-system
```

### kube-public

```bash
# Namespace ที่ทุกคนอ่านได้ (แม้ไม่ authenticate)
# ใช้สำหรับ cluster information
kubectl get configmap cluster-info -n kube-public
kubectl get configmap cluster-info -n kube-public -o yaml
```

### kube-node-lease

```bash
# Namespace สำหรับ Node Heartbeat Lease objects
# ช่วย scalability ของ Node Health checking
kubectl get lease -n kube-node-lease

# แต่ละ Node มี Lease object:
# NAME         HOLDER          AGE
# node-1       node-1          5d
# node-2       node-2          5d
```

---

## 3. สร้างและจัดการ Namespace

### สร้าง Namespace

#### วิธีที่ 1: Imperative

```bash
# สร้าง namespace
kubectl create namespace my-namespace
kubectl create namespace production
kubectl create namespace staging

# ดู namespaces ทั้งหมด
kubectl get namespaces
kubectl get ns    # short name

# Output:
# NAME              STATUS   AGE
# default           Active   5d
# kube-node-lease   Active   5d
# kube-public       Active   5d
# kube-system       Active   5d
# my-namespace      Active   2s
# production        Active   2s
# staging           Active   2s
```

#### วิธีที่ 2: Declarative (YAML)

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: platform
    cost-center: "12345"
  annotations:
    contact: "platform-team@example.com"
    description: "Production environment namespace"
    created-by: "terraform"
```

```bash
kubectl apply -f namespace.yaml
```

### จัดการ Resources ใน Namespace

```bash
# สร้าง Pod ใน namespace เฉพาะ
kubectl run nginx --image=nginx --namespace=production
kubectl run nginx --image=nginx -n production   # short flag

# ดู Resources ใน namespace เฉพาะ
kubectl get pods -n production
kubectl get all -n production

# ดู Resources ทุก namespaces
kubectl get pods --all-namespaces
kubectl get pods -A   # short flag

# ลบ Resource ใน namespace เฉพาะ
kubectl delete pod nginx -n production

# ลบ Resources ทั้งหมดใน namespace
kubectl delete all --all -n production

# ลบ namespace (และทุกอย่างในนั้น!)
kubectl delete namespace staging
```

### เปลี่ยน Default Namespace

```bash
# ดู current context
kubectl config current-context
kubectl config view --minify | grep namespace

# เปลี่ยน namespace สำหรับ current context
kubectl config set-context --current --namespace=production

# ตอนนี้ไม่ต้องระบุ -n production
kubectl get pods   # ดู pods ใน production

# กลับ default
kubectl config set-context --current --namespace=default
```

### Namespace Labels สำหรับ Network Policies

```yaml
# namespace-with-labels.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend
  labels:
    # Labels สำคัญสำหรับ Network Policies
    kubernetes.io/metadata.name: frontend   # ชื่อ namespace (auto-set)
    tier: frontend
    environment: production
    # สำหรับ Pod Security Admission
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 4. Resource Quota

**ResourceQuota** กำหนด limits ของ Resources ที่ใช้ได้ใน Namespace

### Resource Quota YAML

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute Resources
    requests.cpu: "10"          # CPU requests รวมกันไม่เกิน 10 cores
    requests.memory: 20Gi       # Memory requests รวมกันไม่เกิน 20Gi
    limits.cpu: "20"            # CPU limits รวมกันไม่เกิน 20 cores
    limits.memory: 40Gi         # Memory limits รวมกันไม่เกิน 40Gi
    
    # Object Count
    pods: "50"                  # Pods ได้ไม่เกิน 50 ตัว
    services: "10"              # Services ได้ไม่เกิน 10 ตัว
    deployments.apps: "20"      # Deployments ได้ไม่เกิน 20 ตัว
    configmaps: "30"            # ConfigMaps ได้ไม่เกิน 30 ตัว
    secrets: "20"               # Secrets ได้ไม่เกิน 20 ตัว
    
    # Storage
    requests.storage: 100Gi     # PVC storage requests รวมไม่เกิน 100Gi
    persistentvolumeclaims: "10" # PVCs ได้ไม่เกิน 10 ตัว
    
    # LoadBalancer Services
    services.loadbalancers: "2"  # LB Services ได้ไม่เกิน 2 ตัว
    services.nodeports: "5"      # NodePort Services ไม่เกิน 5 ตัว
```

### LimitRange สำหรับ Default Limits

```yaml
# limit-range.yaml
# กำหนด Default Resource Requests/Limits สำหรับ Pods ที่ไม่ระบุ
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  # Container limits
  - type: Container
    default:                    # Default limits ถ้าไม่ระบุ
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:             # Default requests ถ้าไม่ระบุ
      cpu: "100m"
      memory: "128Mi"
    max:                        # Maximum ที่อนุญาต
      cpu: "2"
      memory: "2Gi"
    min:                        # Minimum ที่ต้องมี
      cpu: "50m"
      memory: "64Mi"
  
  # Pod limits
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
  
  # PVC limits
  - type: PersistentVolumeClaim
    max:
      storage: "10Gi"
    min:
      storage: "1Gi"
```

### ดู Resource Quota Status

```bash
# ดู quota ทั้งหมดใน namespace
kubectl get resourcequota -n production
kubectl describe resourcequota production-quota -n production

# Output ตัวอย่าง:
# Name:            production-quota
# Namespace:       production
# Resource         Used   Hard
# --------         ----   ----
# limits.cpu       2      20
# limits.memory    512Mi  40Gi
# pods             3      50
# requests.cpu     500m   10
# requests.memory  256Mi  20Gi
# services         2      10

# ดู LimitRange
kubectl get limitrange -n production
kubectl describe limitrange production-limits -n production
```

---

## 5. Workshop: จัดระเบียบ Apps ด้วย Namespaces

### สถานการณ์: Multi-team E-commerce Platform

เรามีทีม 3 ทีม:
- **frontend-team**: Frontend web application
- **backend-team**: API services
- **data-team**: Database services

และมี 3 environments:
- **production**: สำหรับ users จริง
- **staging**: สำหรับ QA testing
- **development**: สำหรับ developers

### Lab 1: Setup Namespace Structure

```bash
# สร้าง Namespaces สำหรับแต่ละ environment
cat <<'EOF' > /tmp/namespaces.yaml
# Production namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: prod-frontend
  labels:
    env: production
    team: frontend
    tier: presentation
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod-backend
  labels:
    env: production
    team: backend
    tier: application
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod-data
  labels:
    env: production
    team: data
    tier: data
---
# Staging namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: staging-frontend
  labels:
    env: staging
    team: frontend
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging-backend
  labels:
    env: staging
    team: backend
---
# Development namespace (ทีมรวม)
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: development
EOF

kubectl apply -f /tmp/namespaces.yaml

# ดู namespaces ทั้งหมด
kubectl get namespaces --show-labels
```

### Lab 2: Resource Quotas ต่อ Environment

```bash
# Production quota (มาก)
cat <<'EOF' > /tmp/prod-quotas.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: prod-frontend
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: prod-backend
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "30"
    services: "15"
---
# Development quota (น้อยกว่า)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "15"
    services: "10"
EOF

kubectl apply -f /tmp/prod-quotas.yaml

# ดู quotas
kubectl get resourcequota -n prod-frontend
kubectl get resourcequota -n development
```

### Lab 3: LimitRange ต่อ Namespace

```bash
cat <<'EOF' > /tmp/limit-ranges.yaml
# Production LimitRange (conservative)
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: prod-frontend
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
---
# Development LimitRange (relaxed)
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "10m"
      memory: "32Mi"
EOF

kubectl apply -f /tmp/limit-ranges.yaml
```

### Lab 4: Deploy Applications ใน Namespaces

```bash
# Deploy Frontend ใน prod-frontend
cat <<'EOF' > /tmp/frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  namespace: prod-frontend
  labels:
    app: web-frontend
    team: frontend
    env: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
        team: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: prod-frontend
spec:
  selector:
    app: web-frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f /tmp/frontend-deployment.yaml

# Deploy Backend ใน prod-backend
cat <<'EOF' > /tmp/backend-deployment-ns.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: prod-backend
  labels:
    app: api-backend
    team: backend
    env: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
        team: backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args:
        - "-text=Backend API Response"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: prod-backend
spec:
  selector:
    app: api-backend
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
EOF

kubectl apply -f /tmp/backend-deployment-ns.yaml

# ดู resources ใน production namespaces
echo "=== Production Frontend ==="
kubectl get all -n prod-frontend

echo "=== Production Backend ==="
kubectl get all -n prod-backend

# ดู quota usage
kubectl describe resourcequota -n prod-frontend
kubectl describe resourcequota -n prod-backend
```

### Lab 5: Cross-namespace Communication

```bash
# Frontend Pod เรียก Backend Service ข้าม namespace

# สร้าง test pod ใน prod-frontend
kubectl run frontend-test \
    --image=curlimages/curl:latest \
    --namespace=prod-frontend \
    --restart=Never \
    -- sleep 3600

kubectl wait --for=condition=Ready pod/frontend-test \
    -n prod-frontend --timeout=60s

# เรียก Backend ข้าม namespace ด้วย FQDN
kubectl exec -n prod-frontend frontend-test -- \
    curl -s http://api-service.prod-backend.svc.cluster.local:8080

# ตรวจสอบ DNS
kubectl exec -n prod-frontend frontend-test -- \
    nslookup api-service.prod-backend.svc.cluster.local

# Cleanup
kubectl delete pod frontend-test -n prod-frontend
```

### Lab 6: Namespace Context Management

```bash
# สร้าง kubeconfig contexts สำหรับแต่ละ namespace
kubectl config set-context prod-frontend-ctx \
    --cluster=$(kubectl config current-context) \
    --namespace=prod-frontend

kubectl config set-context prod-backend-ctx \
    --cluster=$(kubectl config current-context) \
    --namespace=prod-backend

kubectl config set-context dev-ctx \
    --cluster=$(kubectl config current-context) \
    --namespace=development

# สลับ context ตาม namespace ที่ต้องการทำงาน
kubectl config use-context dev-ctx
kubectl get pods   # ดู pods ใน development namespace

kubectl config use-context prod-frontend-ctx
kubectl get pods   # ดู pods ใน prod-frontend namespace

# กลับ context ปกติ
kubectl config use-context $(kubectl config current-context)
kubectl config set-context --current --namespace=default
```

### Lab 7: ทดสอบ Quota Enforcement

```bash
# ทดสอบว่า Quota ทำงานจริง

# เพิ่ม quota ให้ development namespace ก่อน
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "3"       # จำกัด 3 pods เท่านั้น!
    cpu: "2"
    memory: 2Gi
EOF

# สร้าง Pods จนเกิน quota
for i in 1 2 3 4; do
    echo "Creating pod-$i..."
    kubectl run "test-pod-$i" \
        --image=nginx:1.25 \
        --namespace=development \
        --restart=Never 2>&1 || echo "Pod $i REJECTED by quota!"
done

# ดูสถานะ
kubectl get pods -n development

# ดูว่า quota exceeded ที่ pod ที่ 4
kubectl describe resourcequota dev-quota -n development

# Cleanup
kubectl delete pods --all -n development
kubectl delete resourcequota dev-quota -n development
```

### Lab 8: Namespace Cleanup

```bash
# ลบ namespace พร้อม resources ทั้งหมด
kubectl delete namespace staging-frontend
kubectl delete namespace staging-backend

# ดูว่า resources ถูกลบหมด
kubectl get all -n staging-frontend 2>/dev/null || echo "Namespace deleted"
```

### Cleanup Workshop

```bash
# ลบ namespaces ทั้งหมด
kubectl delete -f /tmp/namespaces.yaml
kubectl delete namespace prod-frontend prod-backend prod-data \
    staging-frontend staging-backend development 2>/dev/null || true

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/namespaces.yaml /tmp/prod-quotas.yaml \
    /tmp/limit-ranges.yaml /tmp/frontend-deployment.yaml \
    /tmp/backend-deployment-ns.yaml
```

### Namespace Best Practices

```bash
# 1. สร้าง Namespace structure ที่ชัดเจน
# Pattern: <environment>-<team> หรือ <team>-<environment>
# Examples:
# - prod-frontend, prod-backend, prod-data
# - staging-apps, development
# - team-alpha, team-beta

# 2. ตั้ง Resource Quota เสมอ (ป้องกัน resource starvation)
# ตั้ง LimitRange เพื่อ set default limits

# 3. ใช้ Labels บน Namespaces
# kubectl label namespace production env=production tier=production

# 4. ตั้งค่า Network Policies ต่อ Namespace
# จำกัด traffic เฉพาะที่จำเป็น

# 5. อย่า deploy workloads ใน kube-system namespace

# 6. ตั้ง RBAC ต่อ Namespace
# แต่ละทีมควรมีสิทธิ์เฉพาะ namespace ของตัวเอง

# 7. ใช้ Namespace เป็น scope สำหรับ secrets
# ไม่ share secrets ข้าม namespaces
```

### Namespace และ RBAC

```yaml
# ตัวอย่าง Role ที่ใช้งาน namespace เฉพาะ
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: development
  name: developer-binding
subjects:
- kind: User
  name: john.developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

---

## สรุป

Namespaces เป็นเครื่องมือสำคัญสำหรับจัดระเบียบ Kubernetes Cluster:

1. **Isolation**: แยก Resources ของแต่ละทีมหรือ environment
2. **Resource Management**: กำหนด Quotas และ Limits ต่อ Namespace
3. **Access Control**: กำหนด RBAC per Namespace
4. **Organization**: จัดระเบียบตาม environment หรือ team

**Namespace Naming Patterns:**
- By environment: `production`, `staging`, `development`
- By team: `team-frontend`, `team-backend`
- By both: `prod-frontend`, `staging-backend`
- By application: `payment-service`, `user-service`

ในบทต่อไปเราจะเรียนรู้ **Labels และ Selectors** ซึ่งเป็น metadata ที่ช่วยในการจัดการและค้นหา Resources
