# Part 55: RBAC (Role-Based Access Control)

## บทนำ

RBAC (Role-Based Access Control) เป็นกลไก authorization หลักของ Kubernetes ที่ควบคุมว่า "ใคร" (subjects) สามารถ "ทำอะไร" (verbs) กับ "อะไร" (resources) ใน Kubernetes cluster

---

## 55.1 RBAC คืออะไร

### แนวคิดพื้นฐาน

```
RBAC = ตอบคำถาม:
  - WHO can do WHAT on WHICH resource?
  
WHO (Subjects):
  - User (human users)
  - Group (set of users)
  - ServiceAccount (for pods/applications)

WHAT (Verbs):
  - get, list, watch
  - create, update, patch
  - delete, deletecollection

WHICH Resource:
  - pods, deployments, services
  - secrets, configmaps
  - nodes, namespaces
  - custom resources
```

### Components ของ RBAC

```
┌────────────────────────────────────────────────────────────┐
│                      RBAC Components                        │
│                                                            │
│  ┌──────────┐    ┌──────────────────┐    ┌────────────┐   │
│  │ Subject  │    │      Role        │    │  Resource  │   │
│  │          │    │  (Rules/Perms)   │    │            │   │
│  │ - User   │    │ - get pods       │    │ - pods     │   │
│  │ - Group  │    │ - list services  │    │ - services │   │
│  │ - SA     │    │ - create secrets │    │ - secrets  │   │
│  └────┬─────┘    └───────┬──────────┘    └────────────┘   │
│       │                  │                                  │
│       └──────────────────┘                                  │
│            RoleBinding                                       │
│         (connects Subject to Role)                          │
└────────────────────────────────────────────────────────────┘
```

---

## 55.2 Roles vs ClusterRoles

### ความแตกต่าง

| | Role | ClusterRole |
|--|------|-------------|
| Scope | Namespace-specific | Cluster-wide |
| Resources | Namespace resources only | All resources incl. non-namespaced |
| Use case | App-specific permissions | Admin, node resources, cross-namespace |

### Role (Namespaced)

```yaml
# Role: สิทธิ์ใน namespace เดียวเท่านั้น
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production      # จำกัดอยู่ใน namespace นี้
rules:
# Rule 1: อ่าน pods
- apiGroups: [""]            # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Rule 2: อ่าน pod logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]

# Rule 3: exec ใน pods
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

### ClusterRole (Cluster-wide)

```yaml
# ClusterRole: สิทธิ์ระดับ cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
  # ไม่มี namespace - เพราะเป็น cluster-scoped
rules:
# อ่าน nodes (cluster-scoped resource)
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

# อ่าน persistent volumes
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch"]

# อ่าน namespaces ทั้งหมด
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
```

### ClusterRole สำหรับ Custom Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-resource-admin
rules:
# Standard resources
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Batch jobs
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Custom resources (CRD)
- apiGroups: ["mycompany.io"]
  resources: ["myapps", "myapps/status"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Non-resource URLs
- nonResourceURLs: ["/healthz", "/metrics"]
  verbs: ["get"]
```

### Built-in ClusterRoles

```bash
# ดู built-in cluster roles
kubectl get clusterroles

# สำคัญ:
# cluster-admin  - สิทธิ์สูงสุด ทุกอย่าง
# admin          - admin ใน namespace
# edit           - แก้ไขได้ แต่ไม่แก้ RBAC
# view           - อ่านอย่างเดียว

# ดูรายละเอียด
kubectl describe clusterrole view
kubectl describe clusterrole edit
kubectl describe clusterrole admin
kubectl describe clusterrole cluster-admin
```

---

## 55.3 RoleBindings และ ClusterRoleBindings

### RoleBinding

```yaml
# RoleBinding: ผูก Subject กับ Role (ใน namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
# User
- kind: User
  name: john.doe@example.com  # case-sensitive
  apiGroup: rbac.authorization.k8s.io

# Group
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io

# ServiceAccount
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring    # namespace ของ ServiceAccount

roleRef:
  kind: Role               # Role หรือ ClusterRole
  name: pod-reader         # ชื่อ Role
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

```yaml
# ClusterRoleBinding: ผูก Subject กับ ClusterRole (ทั้ง cluster)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin      # Built-in cluster-admin role
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding กับ ClusterRole (hybrid)

```yaml
# ใช้ ClusterRole ใน namespace เดียว
# ประโยชน์: reuse role definition ข้ามหลาย namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-binding
  namespace: team-a        # จำกัดอยู่ใน team-a เท่านั้น
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole        # ใช้ ClusterRole แต่ scope ใน namespace
  name: view               # built-in view ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

---

## 55.4 Verbs และ Resources

### ทุก Verb ที่รองรับ

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs:
  - "get"           # GET /api/v1/namespaces/{ns}/pods/{name}
  - "list"          # GET /api/v1/namespaces/{ns}/pods
  - "watch"         # GET /api/v1/namespaces/{ns}/pods?watch=true
  - "create"        # POST /api/v1/namespaces/{ns}/pods
  - "update"        # PUT /api/v1/namespaces/{ns}/pods/{name}
  - "patch"         # PATCH /api/v1/namespaces/{ns}/pods/{name}
  - "delete"        # DELETE /api/v1/namespaces/{ns}/pods/{name}
  - "deletecollection"  # DELETE /api/v1/namespaces/{ns}/pods
  - "*"             # ทุก verb
```

### Resource Subresources

```yaml
rules:
# Pods และ subresources
- apiGroups: [""]
  resources:
  - "pods"          # pod object
  - "pods/log"      # kubectl logs
  - "pods/exec"     # kubectl exec
  - "pods/portforward"  # kubectl port-forward
  - "pods/status"   # pod status
  verbs: ["get", "create"]

# Deployments และ scale
- apiGroups: ["apps"]
  resources:
  - "deployments"
  - "deployments/scale"    # kubectl scale
  - "deployments/status"
  verbs: ["get", "list", "update", "patch"]
```

### ResourceNames: จำกัดเฉพาะ Resource ที่ระบุ

```yaml
# อนุญาตเฉพาะ secrets ที่มีชื่อที่ระบุ
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames:
  - "db-secret"       # เฉพาะ secret ชื่อนี้
  - "api-secret"      # และชื่อนี้
  verbs: ["get"]
  
# ห้าม list เพราะ list ไม่รองรับ resourceNames
# (ต้องใช้ verbs ที่ระบุ resource ชัดเจน)
```

---

## 55.5 ตัวอย่าง Role Patterns ที่ใช้บ่อย

### 1. Developer Role (Read/Debug)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
# อ่านและ debug pods
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create"]

# อ่าน services และ endpoints
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]

# อ่าน configmaps (ไม่รวม secrets)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

# อ่าน deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]

# Port forward
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
```

### 2. Deployment Manager Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
# จัดการ deployments
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "list", "watch", "update", "patch"]

# อ่าน pods (ไม่ลบ)
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

# อ่าน services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]

# Rollout restart (ต้องการ update pods)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["delete"]  # สำหรับ force restart
```

### 3. Monitoring Role (Read-only)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
# ข้อมูล pods/nodes
- apiGroups: [""]
  resources:
  - "pods"
  - "nodes"
  - "nodes/metrics"
  - "nodes/proxy"
  - "services"
  - "endpoints"
  - "namespaces"
  verbs: ["get", "list", "watch"]

# Metrics
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]

# Deployments สำหรับ health check
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]

# Non-resource URLs
- nonResourceURLs:
  - "/metrics"
  - "/healthz"
  - "/readyz"
  verbs: ["get"]
```

### 4. CI/CD Service Account Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
# Deploy/update apps
- apiGroups: ["apps"]
  resources:
  - "deployments"
  - "statefulsets"
  - "daemonsets"
  verbs: ["get", "list", "create", "update", "patch"]

# Manage configmaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

# อ่าน pods สำหรับ rollout status
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

# Manage services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update", "patch"]

# Jobs สำหรับ database migrations
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create", "delete"]
```

---

## 55.6 การ Debug และ ตรวจสอบ RBAC

### auth can-i

```bash
# ตรวจสอบว่า user ทำอะไรได้บ้าง
kubectl auth can-i get pods -n production
kubectl auth can-i delete pods -n production
kubectl auth can-i create secrets -n production

# ตรวจสอบในชื่อของ user อื่น (admin only)
kubectl auth can-i get pods \
  --as=john.doe@example.com \
  -n production

# ตรวจสอบในชื่อของ ServiceAccount
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:webapp-sa \
  -n production

# ดูทุกอย่างที่ user ทำได้
kubectl auth can-i --list -n production
kubectl auth can-i --list -n production \
  --as=john.doe@example.com

# ดู cluster-level permissions
kubectl auth can-i --list
```

### kubectl whoami

```bash
# ดูว่าเราเป็น user อะไรอยู่
kubectl auth whoami

# ดู context ปัจจุบัน
kubectl config current-context
kubectl config view --minify
```

### rbac-lookup (tool เพิ่มเติม)

```bash
# ติดตั้ง kube-rbac-proxy หรือ rakkess
# https://github.com/corneliusweig/rakkess

# ดูสิทธิ์ทั้งหมดของ user
rakkess --sa monitoring:prometheus
rakkess --as john@example.com
```

---

## 55.7 Workshop: Implement Least Privilege Access

### สถานการณ์

เราจะสร้าง multi-team setup:
- **Team Alpha**: developers ที่ deploy ใน namespace `alpha`
- **Team Beta**: developers ที่ deploy ใน namespace `beta`
- **Platform Team**: cluster admins
- **Monitoring**: service account สำหรับ prometheus

### Step 1: Setup Namespaces และ Teams

```bash
kubectl create namespace alpha
kubectl create namespace beta
kubectl create namespace monitoring

# Label namespaces
kubectl label namespace alpha team=alpha
kubectl label namespace beta team=beta
kubectl label namespace monitoring purpose=monitoring
```

### Step 2: สร้าง Roles

```yaml
# developer-role.yaml
# Role สำหรับ developers ใน namespace ตัวเอง
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: alpha
rules:
# Workloads - deploy และ debug
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Pods - debug
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec", "pods/portforward"]
  verbs: ["get", "list", "watch", "create", "delete"]

# Services
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# ConfigMaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# HPA
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Jobs
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Events (อ่านเพื่อ debug)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
---
# Copy ไปที่ beta ด้วย
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: beta
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec", "pods/portforward"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f developer-role.yaml
```

### Step 3: สร้าง ClusterRole สำหรับ Monitoring

```yaml
# monitoring-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-monitoring
  labels:
    app: prometheus
rules:
# Scrape metrics จาก pods
- apiGroups: [""]
  resources:
  - "pods"
  - "pods/metrics"
  - "nodes"
  - "nodes/metrics"
  - "nodes/proxy"
  - "services"
  - "endpoints"
  - "namespaces"
  verbs: ["get", "list", "watch"]

# Services metrics
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]

# Apps metrics
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]

# Metrics API
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]

# Non-resource URLs
- nonResourceURLs:
  - "/metrics"
  - "/metrics/cadvisor"
  - "/healthz"
  verbs: ["get"]
---
# ServiceAccount สำหรับ Prometheus
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-monitoring-binding
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-monitoring
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f monitoring-clusterrole.yaml
```

### Step 4: สร้าง RoleBindings

```yaml
# team-bindings.yaml

# Team Alpha - Group binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alpha-team-binding
  namespace: alpha
subjects:
- kind: Group
  name: "team-alpha"  # OIDC group หรือ LDAP group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
# Team Beta - Group binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: beta-team-binding
  namespace: beta
subjects:
- kind: Group
  name: "team-beta"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
# Platform Team - Cluster Admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-admin
subjects:
- kind: Group
  name: "platform-team"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f team-bindings.yaml
```

### Step 5: ServiceAccount สำหรับ Applications

```yaml
# app-service-accounts.yaml

# ServiceAccount สำหรับ Alpha app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alpha-app-sa
  namespace: alpha
  labels:
    team: alpha
automountServiceAccountToken: false
---
# Role: อ่าน configmap เฉพาะของตัวเอง
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: alpha-app-role
  namespace: alpha
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["alpha-app-config"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["alpha-app-secret"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alpha-app-binding
  namespace: alpha
subjects:
- kind: ServiceAccount
  name: alpha-app-sa
  namespace: alpha
roleRef:
  kind: Role
  name: alpha-app-role
  apiGroup: rbac.authorization.k8s.io
---
# ServiceAccount สำหรับ Beta app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: beta-app-sa
  namespace: beta
  labels:
    team: beta
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: beta-app-role
  namespace: beta
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["beta-app-config"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["beta-app-secret"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: beta-app-binding
  namespace: beta
subjects:
- kind: ServiceAccount
  name: beta-app-sa
  namespace: beta
roleRef:
  kind: Role
  name: beta-app-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f app-service-accounts.yaml
```

### Step 6: ทดสอบ RBAC

```bash
# ทดสอบ Team Alpha permissions
echo "=== Testing Team Alpha ==="

# Alpha ควรทำได้ใน alpha namespace
kubectl auth can-i get pods -n alpha \
  --as=user:alice@example.com \
  --as-group=team-alpha
# yes

kubectl auth can-i create deployments -n alpha \
  --as=user:alice@example.com \
  --as-group=team-alpha
# yes

# Alpha ไม่ควรทำได้ใน beta namespace
kubectl auth can-i get pods -n beta \
  --as=user:alice@example.com \
  --as-group=team-alpha
# no

kubectl auth can-i get secrets -n alpha \
  --as=user:alice@example.com \
  --as-group=team-alpha
# no (developer role ไม่ให้ get secrets)

echo ""
echo "=== Testing Monitoring SA ==="
kubectl auth can-i get pods --all-namespaces \
  --as=system:serviceaccount:monitoring:prometheus-sa
# yes

kubectl auth can-i create pods -n monitoring \
  --as=system:serviceaccount:monitoring:prometheus-sa
# no

echo ""
echo "=== Testing App SA ==="
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:alpha:alpha-app-sa \
  -n alpha
# yes (but only alpha-app-config)

kubectl auth can-i list secrets \
  --as=system:serviceaccount:alpha:alpha-app-sa \
  -n alpha
# no (ไม่ให้ list ทุก secret)
```

### Step 7: ตรวจสอบ Roles และ Bindings

```bash
# ดู roles ทั้งหมดใน namespace
kubectl get roles -n alpha
kubectl get rolebindings -n alpha
kubectl describe role developer -n alpha
kubectl describe rolebinding alpha-team-binding -n alpha

# ดู cluster roles
kubectl get clusterroles | grep -v "^system:"
kubectl get clusterrolebindings | grep -v "^system:"

# ดู who can do what
kubectl auth can-i --list -n alpha \
  --as=system:serviceaccount:alpha:alpha-app-sa
```

### Step 8: สร้าง RBAC Audit Policy

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log ทุก request กับ secrets
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  namespaces: ["alpha", "beta", "production"]

# Log ทุก failure
- level: Metadata
  omitStages:
  - RequestReceived
  verbs: ["*"]
  namespaces: ["*"]
```

### Step 9: Cleanup

```bash
kubectl delete namespace alpha beta monitoring
echo "Workshop cleanup complete!"
```

---

## 55.8 RBAC Best Practices

### 1. Principle of Least Privilege

```yaml
# ไม่ดี: ให้สิทธิ์มากเกินไป
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# ดี: ให้เฉพาะที่จำเป็น
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

### 2. ห้ามใช้ cluster-admin สำหรับ Applications

```yaml
# ไม่ดี: ผูก cluster-admin กับ default ServiceAccount
# ห้ามทำแบบนี้!
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
```

### 3. ใช้ Groups แทน Users

```yaml
# ดี: ใช้ groups เพื่อจัดการง่าย
subjects:
- kind: Group
  name: "engineering@example.com"
  apiGroup: rbac.authorization.k8s.io

# เมื่อ user เข้า-ออก group แก้ที่ identity provider
# ไม่ต้องแก้ RBAC
```

### 4. Regular RBAC Audit

```bash
# Script สำหรับ audit RBAC
#!/bin/bash
echo "=== Cluster Admin Bindings ==="
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | 
    .metadata.name + ": " + (.subjects[]? | .kind + "/" + .name)'

echo ""
echo "=== Wildcard Permissions ==="
kubectl get clusterroles -o json | \
  jq -r '.items[] | select(.rules[]? | .verbs[] == "*" or .resources[] == "*") | 
    .metadata.name'

echo ""
echo "=== ServiceAccounts with cluster-admin ==="
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | 
    select(.roleRef.name=="cluster-admin") | 
    select(.subjects[]?.kind=="ServiceAccount") | 
    .subjects[] | select(.kind=="ServiceAccount") | 
    .namespace + "/" + .name'
```

### 5. Namespace Isolation

```yaml
# สร้าง default deny ด้วย LimitRange และ NetworkPolicy
# RBAC ให้สิทธิ์เฉพาะ namespace ตัวเอง
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-role
  namespace: team-a  # เข้าถึงได้เฉพาะ namespace นี้
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["*"]
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **RBAC Concepts**: Who + What + Which = Permissions
2. **Roles vs ClusterRoles**: namespace-scoped vs cluster-scoped
3. **RoleBindings**: เชื่อม subjects กับ roles
4. **Verbs และ Resources**: การกำหนดสิทธิ์ละเอียด
5. **Best Practices**: Least Privilege, Groups, Regular Audit

**Key Takeaways:**
- เสมอใช้ namespace-scoped roles เมื่อทำได้
- ห้ามให้ `*` wildcards ใน production
- ใช้ `kubectl auth can-i` เพื่อ test permissions
- Audit RBAC เป็นประจำ
- ใช้ Groups แทน individual users

---

**ต่อไป**: Part 56 - Service Accounts
