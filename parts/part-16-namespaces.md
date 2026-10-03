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

---

## Namespace Design Patterns

### Pattern 1: Team-based Namespaces

ใช้เมื่อองค์กรแบ่งตามทีม แต่ละทีมเป็นเจ้าของ namespace ของตัวเอง

```
cluster/
├── team-frontend/       # Frontend Team
├── team-backend/        # Backend Team
├── team-data/           # Data Engineering Team
├── team-security/       # Security Team
└── shared-infra/        # Shared Infrastructure (monitoring, logging)
```

```yaml
# สร้าง Namespaces สำหรับแต่ละทีม
apiVersion: v1
kind: Namespace
metadata:
  name: team-frontend
  labels:
    team: frontend
    cost-center: "engineering-fe"
    owner: "frontend-lead@company.com"
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-backend
  labels:
    team: backend
    cost-center: "engineering-be"
    owner: "backend-lead@company.com"
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-data
  labels:
    team: data
    cost-center: "data-engineering"
    owner: "data-lead@company.com"
---
apiVersion: v1
kind: Namespace
metadata:
  name: shared-infra
  labels:
    team: platform
    cost-center: "platform"
```

```yaml
# ResourceQuota ต่อทีม
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

### Pattern 2: Environment-based Namespaces

ใช้เมื่อต้องการแยก environment อย่างชัดเจน

```
cluster/
├── production/          # Live system
├── staging/             # Pre-production
├── development/         # Active development
└── testing/             # Automated testing
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    tier: prod
    sla: "99.9"
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    tier: pre-prod
---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development
    tier: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: testing
  labels:
    environment: testing
    tier: test
```

```bash
# ตั้งค่า context สำหรับแต่ละ environment
kubectl config set-context prod-ctx \
    --cluster=my-cluster \
    --namespace=production \
    --user=prod-user

kubectl config set-context dev-ctx \
    --cluster=my-cluster \
    --namespace=development \
    --user=dev-user

# Switch ระหว่าง environments
kubectl config use-context prod-ctx
kubectl config use-context dev-ctx
```

### Pattern 3: Service-based Namespaces (Microservices)

ใช้เมื่อมี microservices หลายตัวและต้องการ isolate แต่ละ service

```
cluster/
├── payment-service/     # Payment Service และ dependencies
├── user-service/        # User Service
├── order-service/       # Order Service
├── notification-service/ # Notification Service
└── api-gateway/         # API Gateway
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payment-service
  labels:
    service: payment
    domain: finance
    pci-compliant: "true"
    data-classification: sensitive
---
apiVersion: v1
kind: Namespace
metadata:
  name: user-service
  labels:
    service: user
    domain: identity
    gdpr-relevant: "true"
---
apiVersion: v1
kind: Namespace
metadata:
  name: order-service
  labels:
    service: order
    domain: commerce
```

### Pattern 4: Hybrid Pattern (ทีม + Environment)

สำหรับองค์กรขนาดใหญ่ที่ต้องการทั้ง team isolation และ environment isolation

```
cluster/
├── frontend-prod/       # Frontend Production
├── frontend-staging/    # Frontend Staging
├── frontend-dev/        # Frontend Development
├── backend-prod/        # Backend Production
├── backend-staging/     # Backend Staging
└── backend-dev/         # Backend Development
```

```bash
# Script สร้าง namespaces แบบ hybrid
TEAMS=("frontend" "backend" "data")
ENVIRONMENTS=("prod" "staging" "dev")

for team in "${TEAMS[@]}"; do
  for env in "${ENVIRONMENTS[@]}"; do
    ns="${team}-${env}"
    kubectl create namespace "$ns" \
        --dry-run=client -o yaml | \
    kubectl apply -f -
    
    kubectl label namespace "$ns" \
        team="$team" \
        environment="$env"
  done
done
```

---

## Cross-Namespace Communication

### Service DNS Resolution

ใน Kubernetes Services มี DNS record ในรูปแบบ:
```
<service-name>.<namespace>.svc.cluster.local
```

```bash
# ภายใน namespace เดียวกัน
curl http://my-service

# ข้าม namespace
curl http://my-service.other-namespace.svc.cluster.local

# ตรวจสอบ DNS resolution
kubectl run dns-test --image=busybox --restart=Never -it --rm \
    -- nslookup my-service.production.svc.cluster.local
```

### ตัวอย่าง Cross-Namespace Access

```yaml
# Service ใน namespace 'backend'
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: backend
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
```

```yaml
# Pod ใน namespace 'frontend' เข้าถึง backend service
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  namespace: frontend
spec:
  containers:
  - name: frontend
    image: myapp-frontend:1.0
    env:
    - name: API_URL
      # ใช้ full DNS name เพื่อ cross-namespace access
      value: "http://api-service.backend.svc.cluster.local:8080"
```

### ExternalName Service สำหรับ Cross-Namespace

```yaml
# สร้าง alias ใน namespace ปัจจุบันที่ชี้ไป namespace อื่น
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: frontend
spec:
  type: ExternalName
  # ชี้ไปที่ service ใน namespace อื่น
  externalName: api-service.backend.svc.cluster.local
```

```yaml
# ตอนนี้ frontend Pods เรียก backend-api ได้เลย ไม่ต้องระบุ full path
# http://backend-api:8080 จะถูก resolve ไปที่ backend/api-service
```

### NetworkPolicy สำหรับ Cross-Namespace

```yaml
# อนุญาตเฉพาะ traffic จาก namespace ที่มี label team=frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-ns
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    # อนุญาต traffic จาก namespace ที่มี label team=frontend
    - namespaceSelector:
        matchLabels:
          team: frontend
    # และต้องมาจาก Pod ที่มี label app=frontend-web
    - podSelector:
        matchLabels:
          app: frontend-web
  ports:
  - protocol: TCP
    port: 8080
```

```yaml
# NetworkPolicy: ปิดกั้นทุก cross-namespace traffic โดย default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: payment-service
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  # อนุญาต DNS เท่านั้น
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

## Namespace-scoped vs Cluster-scoped Resources

### Namespace-scoped Resources (ต้องระบุ namespace)

| Resource | API Group |
|----------|-----------|
| Pod | core |
| Service | core |
| ConfigMap | core |
| Secret | core |
| PersistentVolumeClaim | core |
| Deployment | apps |
| ReplicaSet | apps |
| StatefulSet | apps |
| DaemonSet | apps |
| Job | batch |
| CronJob | batch |
| Ingress | networking.k8s.io |
| HorizontalPodAutoscaler | autoscaling |
| NetworkPolicy | networking.k8s.io |
| ServiceAccount | core |
| Role | rbac.authorization.k8s.io |
| RoleBinding | rbac.authorization.k8s.io |

```bash
# ดู Resources ที่ namespace-scoped
kubectl api-resources --namespaced=true

# ดูจำนวน Resources ทั้งหมดใน namespace
kubectl get all -n production
```

### Cluster-scoped Resources (ไม่ต้องระบุ namespace)

| Resource | API Group |
|----------|-----------|
| Node | core |
| PersistentVolume | core |
| Namespace | core |
| ClusterRole | rbac.authorization.k8s.io |
| ClusterRoleBinding | rbac.authorization.k8s.io |
| StorageClass | storage.k8s.io |
| IngressClass | networking.k8s.io |
| CustomResourceDefinition | apiextensions.k8s.io |

```bash
# ดู Resources ที่ cluster-scoped
kubectl api-resources --namespaced=false

# Cluster-scoped resources ไม่มี namespace field
kubectl get nodes
kubectl get persistentvolumes
kubectl get clusterroles
```

### ตัวอย่างความแตกต่าง

```yaml
# Role (namespace-scoped) - ใช้ได้แค่ใน namespace ที่กำหนด
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development     # จำกัดอยู่ใน development namespace
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```yaml
# ClusterRole (cluster-scoped) - ใช้ได้ทั้ง cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-global    # ไม่มี namespace field
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

---

## ResourceQuota per Namespace

### Quota สำหรับ CPU และ Memory

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # CPU requests และ limits
    requests.cpu: "4"       # ทุก Pod รวมกันต้อง request ไม่เกิน 4 CPU
    limits.cpu: "8"         # ทุก Pod รวมกันต้อง limit ไม่เกิน 8 CPU
    # Memory requests และ limits
    requests.memory: 8Gi   # ทุก Pod รวมกันต้อง request ไม่เกิน 8Gi
    limits.memory: 16Gi    # ทุก Pod รวมกันต้อง limit ไม่เกิน 16Gi
```

### Quota สำหรับ Storage

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: development
spec:
  hard:
    # PersistentVolumeClaims
    persistentvolumeclaims: "10"         # สร้าง PVC ได้ไม่เกิน 10 ตัว
    requests.storage: "100Gi"            # Request storage รวมไม่เกิน 100Gi
    # Storage class specific
    standard.storageclass.storage.k8s.io/requests.storage: "50Gi"
    premium.storageclass.storage.k8s.io/requests.storage: "20Gi"
    premium.storageclass.storage.k8s.io/persistentvolumeclaims: "3"
```

### Quota สำหรับ Object Count

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-count-quota
  namespace: development
spec:
  hard:
    # Workload objects
    pods: "50"
    replicationcontrollers: "5"
    count/deployments.apps: "20"
    count/replicasets.apps: "40"
    count/statefulsets.apps: "5"
    # Service objects
    services: "20"
    services.loadbalancers: "2"
    services.nodeports: "5"
    # Config objects
    configmaps: "30"
    secrets: "30"
    # Networking
    count/ingresses.networking.k8s.io: "10"
```

### ตัวอย่าง Complete ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    # Storage
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
    # Object counts
    pods: "100"
    services: "30"
    configmaps: "50"
    secrets: "50"
    count/deployments.apps: "30"
    count/statefulsets.apps: "10"
    count/cronjobs.batch: "10"
    count/ingresses.networking.k8s.io: "20"
    # Load Balancers (expensive)
    services.loadbalancers: "5"
    services.nodeports: "0"
```

### ตรวจสอบ Quota Usage

```bash
# ดู Quota usage ใน namespace
kubectl describe resourcequota -n development

# Output ตัวอย่าง:
# Name:                   compute-quota
# Namespace:              development
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              2       8
# limits.memory           4Gi     16Gi
# requests.cpu            1       4
# requests.memory         2Gi     8Gi
# pods                    12      50

# ดู Quota ทุก namespace
kubectl get resourcequota --all-namespaces

# Script ตรวจสอบ Quota usage
for ns in $(kubectl get namespaces -o name | cut -d/ -f2); do
  echo "=== Namespace: $ns ==="
  kubectl describe resourcequota -n "$ns" 2>/dev/null || \
      echo "  (no quota)"
  echo ""
done
```

---

## LimitRange

LimitRange กำหนดค่า default, minimum, maximum สำหรับ Resources ใน namespace

### LimitRange สำหรับ Container

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: development
spec:
  limits:
  - type: Container
    # ค่า default ถ้า Pod ไม่ระบุ limits
    default:
      cpu: 500m
      memory: 256Mi
    # ค่า default ถ้า Pod ไม่ระบุ requests
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    # ค่า maximum ที่ Pod ระบุได้
    max:
      cpu: "2"
      memory: 2Gi
    # ค่า minimum ที่ Pod ต้องมี
    min:
      cpu: 50m
      memory: 64Mi
    # อัตราส่วน max/min
    maxLimitRequestRatio:
      cpu: "4"
      memory: "4"
```

### LimitRange สำหรับ Pod

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: production
spec:
  limits:
  - type: Pod
    # ค่า maximum รวมของทุก container ใน Pod
    max:
      cpu: "4"
      memory: 4Gi
    min:
      cpu: 100m
      memory: 128Mi
```

### LimitRange สำหรับ PersistentVolumeClaim

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: development
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 50Gi
    min:
      storage: 1Gi
```

### ตรวจสอบ LimitRange

```bash
# ดู LimitRanges ใน namespace
kubectl describe limitrange -n development

# Output ตัวอย่าง:
# Name:       container-limits
# Namespace:  development
# Type        Resource  Min    Max    Default Request  Default Limit  Max Limit/Request Ratio
# ----        --------  ---    ---    ---------------  -------------  -----------------------
# Container   cpu       50m    2      100m             500m           4
# Container   memory    64Mi   2Gi    128Mi            256Mi          4
```

---

## Workshop: Multi-team Namespace Setup

### ภาพรวม Workshop

ในบทนี้จะทำ Workshop สร้าง Multi-team Setup ที่สมจริง:
- 2 ทีม: `team-alpha` และ `team-beta`
- 2 environments: `staging` และ `production`
- ResourceQuota แตกต่างกันตาม environment
- RBAC สำหรับแต่ละทีม

### ขั้นตอนที่ 1: สร้าง Namespaces

```bash
# สร้าง namespaces
cat > namespaces.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: alpha-staging
  labels:
    team: alpha
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: alpha-production
  labels:
    team: alpha
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: beta-staging
  labels:
    team: beta
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: beta-production
  labels:
    team: beta
    environment: production
EOF

kubectl apply -f namespaces.yaml
kubectl get namespaces -l team=alpha
kubectl get namespaces -l team=beta
```

### ขั้นตอนที่ 2: ตั้ง ResourceQuota

```bash
cat > quotas.yaml << 'EOF'
# Staging quota (ขนาดเล็กกว่า)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: alpha-staging
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: beta-staging
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
---
# Production quota (ขนาดใหญ่กว่า)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: alpha-production
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "100"
    services: "30"
    services.loadbalancers: "3"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: beta-production
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "100"
    services: "30"
    services.loadbalancers: "3"
EOF

kubectl apply -f quotas.yaml
```

### ขั้นตอนที่ 3: ตั้ง LimitRange

```bash
cat > limitranges.yaml << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: alpha-staging
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: beta-staging
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
EOF

kubectl apply -f limitranges.yaml
```

### ขั้นตอนที่ 4: ตั้ง RBAC

```bash
cat > rbac-alpha.yaml << 'EOF'
# ServiceAccount สำหรับ team alpha
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alpha-developer
  namespace: alpha-staging
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alpha-developer
  namespace: alpha-production
---
# Role สำหรับ staging (full access)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-full
  namespace: alpha-staging
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
# Role สำหรับ production (read + deploy only)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-deploy
  namespace: alpha-production
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "update", "patch"]
---
# RoleBinding สำหรับ staging
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alpha-dev-binding
  namespace: alpha-staging
subjects:
- kind: ServiceAccount
  name: alpha-developer
  namespace: alpha-staging
roleRef:
  kind: Role
  name: developer-full
  apiGroup: rbac.authorization.k8s.io
---
# RoleBinding สำหรับ production
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alpha-deploy-binding
  namespace: alpha-production
subjects:
- kind: ServiceAccount
  name: alpha-developer
  namespace: alpha-staging
roleRef:
  kind: Role
  name: developer-deploy
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f rbac-alpha.yaml
```

### ขั้นตอนที่ 5: ทดสอบ Deploy Application

```bash
cat > app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: alpha-staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      team: alpha
  template:
    metadata:
      labels:
        app: webapp
        team: alpha
        environment: staging
    spec:
      containers:
      - name: webapp
        image: nginx:1.21
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
  name: webapp-svc
  namespace: alpha-staging
spec:
  selector:
    app: webapp
    team: alpha
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f app-deployment.yaml
kubectl get all -n alpha-staging
```

### ขั้นตอนที่ 6: ตรวจสอบ Quota Usage

```bash
# ดู quota usage
kubectl describe resourcequota staging-quota -n alpha-staging

# ดู events ถ้ามี quota exceeded
kubectl get events -n alpha-staging | grep -i quota

# Script ดู quota usage ทุก namespace
for ns in alpha-staging alpha-production beta-staging beta-production; do
  echo "=== $ns ==="
  kubectl describe resourcequota -n "$ns" 2>/dev/null | \
      grep -E "Resource|requests|limits|pods" | head -10
  echo ""
done
```

---

## แบบฝึกหัด: Namespaces

### แบบฝึกหัดที่ 1: สร้าง Namespace พร้อม Labels

**โจทย์**: สร้าง namespace `my-project` พร้อม labels: `project=ecommerce`, `owner=alice`, `environment=dev`

**เฉลย**:
```bash
kubectl create namespace my-project
kubectl label namespace my-project \
    project=ecommerce \
    owner=alice \
    environment=dev

# ตรวจสอบ
kubectl get namespace my-project --show-labels
```

---

### แบบฝึกหัดที่ 2: ตั้ง Default Namespace

**โจทย์**: เปลี่ยน default namespace ใน kubeconfig เป็น `my-project` โดยไม่ต้องระบุ `-n` ทุกครั้ง

**เฉลย**:
```bash
# ดู context ปัจจุบัน
kubectl config current-context

# ตั้ง default namespace สำหรับ context ปัจจุบัน
kubectl config set-context --current --namespace=my-project

# ทดสอบ
kubectl get pods  # จะดูใน my-project โดยอัตโนมัติ

# กลับไป default
kubectl config set-context --current --namespace=default
```

---

### แบบฝึกหัดที่ 3: ResourceQuota และ Pod ที่เกิน Quota

**โจทย์**: สร้าง Quota ที่จำกัด pods ได้ 2 ตัว แล้วพยายามสร้าง 3 Pods สังเกตผลลัพธ์

**เฉลย**:
```bash
# สร้าง namespace
kubectl create namespace quota-test

# สร้าง Quota
kubectl create quota pod-quota \
    --hard=pods=2 \
    -n quota-test

# สร้าง Pods
kubectl run pod1 --image=nginx -n quota-test
kubectl run pod2 --image=nginx -n quota-test
kubectl run pod3 --image=nginx -n quota-test
# Pod 3 จะ fail ด้วย Error: pods "pod3" is forbidden:
# exceeded quota: pod-quota, requested: pods=1, used: pods=2, limited: pods=2

# ตรวจสอบ events
kubectl get events -n quota-test | grep Forbidden
```

---

### แบบฝึกหัดที่ 4: LimitRange Default Values

**โจทย์**: สร้าง LimitRange ที่กำหนด default resources แล้วสร้าง Pod ที่ไม่ระบุ resources สังเกตว่า Kubernetes ใส่ค่าให้อัตโนมัติ

**เฉลย**:
```bash
# สร้าง namespace
kubectl create namespace lr-test

# สร้าง LimitRange
cat > lr.yaml << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: auto-limits
  namespace: lr-test
spec:
  limits:
  - type: Container
    default:
      cpu: 300m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
EOF
kubectl apply -f lr.yaml

# สร้าง Pod ไม่ระบุ resources
kubectl run no-resources \
    --image=nginx \
    -n lr-test

# ตรวจสอบว่า Kubernetes ใส่ค่า default ให้
kubectl get pod no-resources -n lr-test \
    -o jsonpath='{.spec.containers[0].resources}'
# จะเห็น requests.cpu=100m, limits.cpu=300m ฯลฯ
```

---

### แบบฝึกหัดที่ 5: Cross-Namespace Service Access

**โจทย์**: สร้าง Service ใน namespace `backend-ns` แล้วเข้าถึงจาก namespace `frontend-ns`

**เฉลย**:
```bash
# สร้าง namespaces
kubectl create namespace backend-ns
kubectl create namespace frontend-ns

# Deploy service ใน backend-ns
kubectl create deployment backend-app \
    --image=nginx \
    -n backend-ns
kubectl expose deployment backend-app \
    --port=80 \
    -n backend-ns

# ทดสอบเข้าถึงจาก frontend-ns
kubectl run test-pod \
    --image=busybox \
    --restart=Never \
    -n frontend-ns \
    --rm -it \
    -- wget -qO- http://backend-app.backend-ns.svc.cluster.local
# ควรได้รับ HTML response จาก nginx

# ทดสอบ short name (จะไม่ทำงาน - ต้องใช้ full DNS name)
kubectl run test-pod2 \
    --image=busybox \
    --restart=Never \
    -n frontend-ns \
    --rm -it \
    -- wget -qO- http://backend-app  # จะ fail
```

---

## สรุปทบทวน Namespaces

Namespaces เป็นกลไก isolation ที่สำคัญใน Kubernetes:

1. **Design Patterns**: เลือก pattern ตามโครงสร้างองค์กร (Team, Environment, Service)
2. **Cross-Namespace**: Services เข้าถึงกันผ่าน full DNS name
3. **Scope**: รู้ว่า resource ใด namespace-scoped หรือ cluster-scoped
4. **ResourceQuota**: จำกัด compute, storage และจำนวน objects
5. **LimitRange**: กำหนด default, min, max สำหรับ container resources
6. **Workshop**: Multi-team setup ต้องการ RBAC + Quota ทำงานร่วมกัน
