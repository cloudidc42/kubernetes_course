# Part 17: Labels และ Selectors - จัดการ Resources อย่างมีระบบ

## สารบัญ
1. [Labels คืออะไร](#labels-คืออะไร)
2. [Label Selectors](#label-selectors)
3. [Equality-based vs Set-based Selectors](#equality-based-vs-set-based)
4. [Best Practices สำหรับ Labeling](#best-practices)
5. [Workshop: จัดการ Resources ด้วย Labels](#workshop)

---

## 1. Labels คืออะไร

**Labels** คือ key-value pairs ที่ attach กับ Kubernetes Objects เพื่อ:
- จัดกลุ่ม (organize) Resources
- เลือก (select) Resources
- ค้นหา (query) Resources

### ทำไมต้องใช้ Labels

```
ไม่มี Labels:
kubectl get pods
# NAME         READY   STATUS    
# pod-abc123   1/1     Running  
# pod-def456   1/1     Running  
# pod-ghi789   1/1     Running  
# ไม่รู้ว่า pod ไหนเป็น app อะไร!

มี Labels:
kubectl get pods --show-labels
# NAME         READY   STATUS    LABELS
# pod-abc123   1/1     Running   app=frontend,env=prod,version=1.0
# pod-def456   1/1     Running   app=backend,env=prod,version=2.0
# pod-ghi789   1/1     Running   app=frontend,env=staging,version=1.1
# รู้ทันทีว่า pod ไหนเป็นอะไร!
```

### Label Syntax

```
key: value

Key format:
  [prefix/]name
  
  prefix (optional):
  - DNS subdomain: kubernetes.io, app.kubernetes.io
  - ไม่เกิน 253 characters
  
  name (required):
  - ตัวอักษร ตัวเลข - _ .
  - ขึ้นต้นและจบด้วย alphanumeric
  - ไม่เกิน 63 characters

Value:
  - ตัวอักษร ตัวเลข - _ .
  - ขึ้นต้นและจบด้วย alphanumeric
  - ไม่เกิน 63 characters
  - สามารถเป็น string ว่าง ""

ตัวอย่าง:
  app: nginx
  app.kubernetes.io/name: nginx
  app.kubernetes.io/version: "1.25.0"
  kubernetes.io/os: linux
  beta.kubernetes.io/arch: amd64
```

### Labels บน Resources ต่างๆ

```yaml
# labels-examples.yaml
# Pod with labels
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    app: webapp
    version: "1.0"
    environment: production
    tier: frontend
    release: stable
    team: frontend-team
---
# Service with labels
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  labels:
    app: webapp
    tier: frontend
spec:
  selector:
    app: webapp   # เลือก Pods ที่มี label นี้
  ports:
  - port: 80
---
# Deployment with labels
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
    version: "1.0"
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: "1.0"
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
```

### Kubernetes Recommended Labels

Kubernetes มี standard labels ที่แนะนำให้ใช้:

```yaml
metadata:
  labels:
    # ชื่อของ Application
    app.kubernetes.io/name: mysql
    
    # ชื่อที่ user-friendly
    app.kubernetes.io/instance: mysql-production
    
    # เวอร์ชันของ Application
    app.kubernetes.io/version: "8.0.35"
    
    # Component ภายใน Application
    app.kubernetes.io/component: database
    
    # Application/Project ที่ component นี้เป็นส่วนหนึ่ง
    app.kubernetes.io/part-of: ecommerce-platform
    
    # Tool ที่ใช้ manage Resource นี้
    app.kubernetes.io/managed-by: helm
    
    # ชื่อ Helm chart
    helm.sh/chart: mysql-9.12.3
```

---

## 2. Label Selectors

**Label Selector** ใช้เพื่อเลือก Objects ที่มี labels ตรงตามเงื่อนไข

### ใช้งาน Label Selectors

```bash
# kubectl get -l <selector>

# Equality-based
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,environment=production  # AND

# Set-based
kubectl get pods -l 'app in (nginx,apache)'
kubectl get pods -l 'environment notin (dev,test)'
kubectl get pods -l tier                   # มี label key 'tier'
kubectl get pods -l '!deprecated'          # ไม่มี label key 'deprecated'

# Complex selectors
kubectl get pods -l 'app=nginx,version in (1.24,1.25),!beta'

# ใช้กับ Resource types อื่น
kubectl get services -l app=nginx
kubectl get deployments -l tier=frontend
kubectl get nodes -l kubernetes.io/role=worker
```

### Label Selectors ใน YAML

```yaml
# ใน Service (selector ธรรมดา)
spec:
  selector:
    app: nginx        # Equality-based เท่านั้น!
    tier: frontend

# ใน ReplicaSet/Deployment (matchLabels + matchExpressions)
spec:
  selector:
    matchLabels:
      app: nginx
    matchExpressions:
    - key: tier
      operator: In
      values: [frontend, web]

# ใน Job/CronJob
spec:
  selector:
    matchLabels:
      job-name: my-job

# ใน NetworkPolicy
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

---

## 3. Equality-based vs Set-based Selectors

### Equality-based Selectors

ใช้ operators: `=`, `==`, `!=`

```bash
# Equality (=, ==)
kubectl get pods -l app=nginx
kubectl get pods -l app==nginx   # เหมือนกัน

# Inequality (!=)
kubectl get pods -l environment!=production
kubectl get pods -l 'app=nginx,environment!=dev'  # AND condition
```

```yaml
# ใน YAML
selector:
  app: nginx           # equality
  environment: production
# หมายความว่า: app == nginx AND environment == production
```

### Set-based Selectors

ใช้ operators: `in`, `notin`, `exists`, `!` (doesnotexist)

```bash
# in: ค่าต้องอยู่ใน set
kubectl get pods -l 'app in (nginx,apache,haproxy)'
kubectl get pods -l 'version in (1.24,1.25)'

# notin: ค่าต้องไม่อยู่ใน set
kubectl get pods -l 'environment notin (dev,test)'

# exists: ต้องมี key นี้ (ไม่สนใจค่า)
kubectl get pods -l tier
kubectl get pods -l 'tier'    # เหมือนกัน

# doesnotexist: ต้องไม่มี key นี้
kubectl get pods -l '!deprecated'
kubectl get pods -l '!beta'
```

```yaml
# ใน YAML matchExpressions
matchExpressions:
- key: app
  operator: In           # in
  values:
  - nginx
  - apache

- key: environment
  operator: NotIn        # notin
  values:
  - dev
  - test

- key: tier
  operator: Exists       # exists

- key: deprecated
  operator: DoesNotExist # doesnotexist (ไม่มี values)
```

### ตัวอย่างการรวม Selectors

```yaml
# complex-selector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-select-demo
spec:
  replicas: 3
  selector:
    matchLabels:           # AND
      app: webapp
    matchExpressions:      # AND กับ matchLabels ทั้งหมด
    - key: environment
      operator: In
      values:
      - production
      - staging
    - key: version
      operator: NotIn
      values:
      - beta
      - alpha
    - key: team
      operator: Exists
  template:
    metadata:
      labels:
        app: webapp
        environment: production
        version: "1.0"
        team: frontend
    spec:
      containers:
      - name: app
        image: nginx:1.25
```

### การใช้ Field Selectors

Field Selectors ต่างจาก Label Selectors - ใช้ filter ด้วย Resource fields:

```bash
# Filter pods ที่กำลัง Running
kubectl get pods --field-selector=status.phase=Running

# Filter services ที่เป็น NodePort
kubectl get services --field-selector spec.type=NodePort

# Filter pods บน node เฉพาะ
kubectl get pods --field-selector spec.nodeName=worker-1

# Filter events ที่เป็น Warning
kubectl get events --field-selector type=Warning

# รวมกัน
kubectl get pods --field-selector=status.phase=Running,spec.nodeName=worker-1

# ใช้กับ --selector (labels) และ field-selector พร้อมกัน
kubectl get pods -l app=nginx --field-selector=status.phase=Running
```

---

## 4. Best Practices สำหรับ Labeling

### Recommended Label Structure

```yaml
metadata:
  labels:
    # === Identity Labels ===
    # ชื่อ Application (required)
    app.kubernetes.io/name: myapp
    
    # Instance ที่ unique (required สำหรับ Helm)
    app.kubernetes.io/instance: myapp-production
    
    # เวอร์ชัน
    app.kubernetes.io/version: "1.2.3"
    
    # Component ใน Application
    # values: frontend, backend, api, database, cache, queue
    app.kubernetes.io/component: frontend
    
    # Application ที่ใหญ่กว่า
    app.kubernetes.io/part-of: ecommerce
    
    # Tool ที่ manage
    # values: helm, kustomize, terraform, manual
    app.kubernetes.io/managed-by: helm
    
    # === Operational Labels ===
    # Environment
    # values: production, staging, development, testing
    environment: production
    
    # Team ที่ดูแล
    team: frontend-team
    
    # Tier
    # values: frontend, backend, database, cache, messaging
    tier: frontend
    
    # Release track
    # values: stable, canary, beta
    release: stable
    
    # === Business Labels ===
    # Cost center
    cost-center: "12345"
    
    # Project
    project: website-redesign
```

### Label Naming Conventions

```bash
# ✓ ดี: ใช้ lowercase
app: webapp
environment: production

# ✓ ดี: ใช้ kebab-case สำหรับ multi-word
app-name: my-webapp
release-track: stable

# ✓ ดี: ใช้ prefix สำหรับ organization-specific labels
mycompany.io/cost-center: "12345"
mycompany.io/project: website

# ✗ ไม่ดี: ใช้ camelCase
appName: myWebapp        # ❌

# ✗ ไม่ดี: ใช้ uppercase
Environment: Production  # ❌

# ✗ ไม่ดี: ใช้ spaces
"app name": webapp       # ❌ ไม่ valid
```

### Labels vs Annotations

| | Labels | Annotations |
|--|--------|-------------|
| ใช้เพื่อ | Select/Group Resources | เก็บข้อมูล metadata |
| Querying | ✓ (kubectl -l) | ✗ |
| Value size | เล็ก (63 chars) | ใหญ่ได้ |
| ตัวอย่าง | app=nginx, env=prod | deployment-url, build-info |

---

## 5. Workshop: จัดการ Resources ด้วย Labels

### Workshop Setup

```bash
# สร้าง namespace สำหรับ workshop
kubectl create namespace label-workshop
kubectl config set-context --current --namespace=label-workshop
```

### Lab 1: สร้าง Resources พร้อม Labels

```bash
# สร้าง Pods หลายตัวพร้อม Labels ต่างๆ
cat <<'EOF' > /tmp/labeled-pods.yaml
# Frontend Production Pod
apiVersion: v1
kind: Pod
metadata:
  name: frontend-prod-1
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: webapp
    app.kubernetes.io/component: frontend
    environment: production
    version: "2.0"
    team: frontend
    release: stable
    tier: presentation
spec:
  containers:
  - name: nginx
    image: nginx:1.25
---
# Frontend Production Pod 2
apiVersion: v1
kind: Pod
metadata:
  name: frontend-prod-2
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: webapp
    app.kubernetes.io/component: frontend
    environment: production
    version: "2.0"
    team: frontend
    release: stable
    tier: presentation
spec:
  containers:
  - name: nginx
    image: nginx:1.25
---
# Frontend Staging Pod
apiVersion: v1
kind: Pod
metadata:
  name: frontend-staging-1
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: webapp
    app.kubernetes.io/component: frontend
    environment: staging
    version: "2.1"
    team: frontend
    release: canary
    tier: presentation
spec:
  containers:
  - name: nginx
    image: nginx:1.25
---
# Backend Production Pod
apiVersion: v1
kind: Pod
metadata:
  name: backend-prod-1
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: api
    app.kubernetes.io/component: backend
    environment: production
    version: "1.5"
    team: backend
    release: stable
    tier: application
spec:
  containers:
  - name: api
    image: hashicorp/http-echo:latest
    args: ["-text=Backend API", "-listen=:5678"]
---
# Backend Production Pod 2
apiVersion: v1
kind: Pod
metadata:
  name: backend-prod-2
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: api
    app.kubernetes.io/component: backend
    environment: production
    version: "1.5"
    team: backend
    release: stable
    tier: application
spec:
  containers:
  - name: api
    image: hashicorp/http-echo:latest
    args: ["-text=Backend API", "-listen=:5678"]
---
# Database Pod
apiVersion: v1
kind: Pod
metadata:
  name: database-prod-1
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: postgres
    app.kubernetes.io/component: database
    environment: production
    version: "15.0"
    team: data
    release: stable
    tier: data
spec:
  containers:
  - name: postgres
    image: busybox:1.36
    command: ['sleep', '3600']
---
# Old/Deprecated Pod
apiVersion: v1
kind: Pod
metadata:
  name: old-service-1
  namespace: label-workshop
  labels:
    app.kubernetes.io/name: oldapp
    environment: production
    version: "0.9"
    deprecated: "true"
    team: legacy
spec:
  containers:
  - name: old
    image: busybox:1.36
    command: ['sleep', '3600']
EOF

kubectl apply -f /tmp/labeled-pods.yaml

# รอ Pods พร้อม
kubectl wait --for=condition=Ready pods --all --timeout=120s

# ดู Pods ทั้งหมดพร้อม Labels
kubectl get pods --show-labels
```

### Lab 2: Querying ด้วย Label Selectors

```bash
echo "=== ดู Pods ทั้งหมด ==="
kubectl get pods --show-labels

echo ""
echo "=== Production Pods เท่านั้น ==="
kubectl get pods -l environment=production

echo ""
echo "=== Frontend Pods เท่านั้น ==="
kubectl get pods -l "app.kubernetes.io/component=frontend"

echo ""
echo "=== Backend หรือ Frontend ==="
kubectl get pods -l "app.kubernetes.io/component in (frontend,backend)"

echo ""
echo "=== Production Stable Pods ==="
kubectl get pods -l "environment=production,release=stable"

echo ""
echo "=== ไม่ใช่ Deprecated ==="
kubectl get pods -l "!deprecated"

echo ""
echo "=== Deprecated ==="
kubectl get pods -l "deprecated"

echo ""
echo "=== Team Frontend ==="
kubectl get pods -l "team=frontend"

echo ""
echo "=== Version ใน [2.0, 1.5] ==="
kubectl get pods -l "version in (2.0,1.5)"

echo ""
echo "=== Presentation Tier ==="
kubectl get pods -l "tier=presentation"

echo ""
echo "=== Count โดย Environment ==="
echo "Production: $(kubectl get pods -l environment=production --no-headers | wc -l)"
echo "Staging: $(kubectl get pods -l environment=staging --no-headers | wc -l)"
```

### Lab 3: Operations บน Label Groups

```bash
# เพิ่ม Labels
kubectl label pods frontend-prod-1 monitored=true
kubectl label pods frontend-prod-2 monitored=true
kubectl label pods backend-prod-1 monitored=true

# ดู Monitored Pods
kubectl get pods -l monitored=true

# แก้ไข Label (overwrite)
kubectl label pods frontend-staging-1 release=stable --overwrite
kubectl get pods -l release=stable

# ลบ Label
kubectl label pods old-service-1 deprecated-   # ลบ deprecated label
kubectl get pods -l "!deprecated"   # ควรเห็น old-service-1 ด้วย

# ทำ action กับ group ของ Pods
echo "=== Delete all staging pods ==="
kubectl delete pods -l environment=staging

echo ""
echo "=== Restart (delete) all deprecated pods ==="
kubectl delete pods -l "app.kubernetes.io/name=oldapp"

echo ""
echo "=== Check remaining ==="
kubectl get pods --show-labels
```

### Lab 4: Label-based Service Discovery

```bash
# สร้าง Services ที่ใช้ Label Selectors

cat <<'EOF' > /tmp/label-services.yaml
# Service สำหรับ Backend ทุก version
apiVersion: v1
kind: Service
metadata:
  name: backend-all
  namespace: label-workshop
spec:
  selector:
    app.kubernetes.io/component: backend    # เลือก backend ทุกตัว
  ports:
  - port: 8080
    targetPort: 5678
---
# Service สำหรับ Backend Production
apiVersion: v1
kind: Service
metadata:
  name: backend-production
  namespace: label-workshop
spec:
  selector:
    app.kubernetes.io/component: backend
    environment: production    # เฉพาะ production
  ports:
  - port: 8080
    targetPort: 5678
---
# Service สำหรับ Frontend Production  
apiVersion: v1
kind: Service
metadata:
  name: frontend-production
  namespace: label-workshop
spec:
  selector:
    app.kubernetes.io/component: frontend
    environment: production
    release: stable    # เฉพาะ stable release
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f /tmp/label-services.yaml

# ดู Endpoints
kubectl get endpoints -n label-workshop
kubectl describe endpoints backend-all -n label-workshop
kubectl describe endpoints backend-production -n label-workshop
kubectl describe endpoints frontend-production -n label-workshop
```

### Lab 5: Node Labels และ NodeSelector

```bash
# ดู Node Labels ปัจจุบัน
kubectl get nodes --show-labels

# เพิ่ม custom labels ให้ Node
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Node: $NODE_NAME"

kubectl label node $NODE_NAME disktype=ssd
kubectl label node $NODE_NAME zone=us-east-1a

# ดู labels หลังเพิ่ม
kubectl get node $NODE_NAME --show-labels

# สร้าง Pod ที่ต้องการรันบน SSD node
cat <<'EOF' > /tmp/node-selector-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-workload
  namespace: label-workshop
spec:
  nodeSelector:
    disktype: ssd    # ต้องมี label นี้
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi
EOF

kubectl apply -f /tmp/node-selector-pod.yaml

# ดูว่า Pod ถูก schedule บน Node ไหน
kubectl get pod ssd-workload -n label-workshop -o wide

# ลบ custom labels จาก Node
kubectl label node $NODE_NAME disktype-
kubectl label node $NODE_NAME zone-

# Cleanup
kubectl delete pod ssd-workload -n label-workshop
```

### Lab 6: Labels สำหรับ Canary Deployment

```bash
# สถานการณ์: Deploy version ใหม่แบบ Canary

cat <<'EOF' > /tmp/canary-labels.yaml
# Version เดิม (stable) - 80% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
  namespace: label-workshop
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      release: stable
  template:
    metadata:
      labels:
        app: myapp
        release: stable
        version: "1.0"
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:latest
        args: ["-text=Stable v1.0", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
# Version ใหม่ (canary) - 20% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  namespace: label-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      labels:
        app: myapp
        release: canary
        version: "2.0"
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:latest
        args: ["-text=Canary v2.0", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
# Service รับ traffic ทั้งหมด (stable + canary)
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: label-workshop
spec:
  selector:
    app: myapp   # เลือกทั้ง stable และ canary
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl apply -f /tmp/canary-labels.yaml

# ดู Pods
kubectl get pods -l app=myapp --show-labels

# ทดสอบ traffic distribution
kubectl run traffic-test \
    --image=curlimages/curl:latest \
    --namespace=label-workshop \
    --restart=Never \
    -- sleep 3600

kubectl wait --for=condition=Ready pod/traffic-test \
    -n label-workshop --timeout=60s

echo "Testing traffic distribution (5 requests):"
for i in {1..10}; do
    kubectl exec -n label-workshop traffic-test -- \
        curl -s http://app-service
    echo ""
done
# ควรเห็น ~80% Stable v1.0 และ ~20% Canary v2.0

# Cleanup canary test
kubectl delete pod traffic-test -n label-workshop
kubectl delete -f /tmp/canary-labels.yaml
```

### Cleanup Workshop

```bash
# ลบทุกอย่าง
kubectl delete namespace label-workshop

# กลับ default
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/labeled-pods.yaml /tmp/label-services.yaml \
    /tmp/node-selector-pod.yaml /tmp/canary-labels.yaml
```

### Label Quick Reference

```bash
# จัดการ Labels
kubectl label pods my-pod key=value        # เพิ่ม/แก้ไข label
kubectl label pods my-pod key-             # ลบ label
kubectl label pods my-pod key=value --overwrite  # แก้ไข

# Filter ด้วย Labels
kubectl get pods -l key=value             # equality
kubectl get pods -l key!=value            # inequality
kubectl get pods -l 'key in (v1,v2)'      # set-based in
kubectl get pods -l 'key notin (v1,v2)'   # set-based notin
kubectl get pods -l key                   # exists
kubectl get pods -l '!key'               # doesnotexist

# แสดง Labels
kubectl get pods --show-labels
kubectl get pods -L key1,key2   # แสดงเป็น column

# Count pods per label value
kubectl get pods --show-labels | grep 'environment=' | \
    grep -o 'environment=[^ ]*' | sort | uniq -c
```

---

## สรุป

Labels และ Selectors เป็น core concept ของ Kubernetes ที่:

1. **Organization**: จัดกลุ่ม Resources ตาม properties ต่างๆ
2. **Selection**: เลือก Resources เพื่อทำ operations
3. **Service Discovery**: Services ใช้ Selectors เพื่อ route traffic
4. **Scheduling**: NodeSelector/Affinity ใช้ Node Labels
5. **Monitoring**: Prometheus, Grafana ใช้ Labels filter metrics

**Key Takeaways:**
- ตั้ง Labels ตั้งแต่ต้น - ยากที่จะเพิ่มทีหลัง
- ใช้ standard labels `app.kubernetes.io/*`
- Labels สำหรับ select, Annotations สำหรับ metadata

ในบทต่อไปเราจะเรียนรู้ **Annotations** ซึ่งเป็น metadata ที่เก็บข้อมูลเพิ่มเติมที่ไม่ได้ใช้สำหรับ select

---

## Recommended Label Schema (app.kubernetes.io/*)

### มาตรฐาน Kubernetes Labels

Kubernetes แนะนำ labels ชุด `app.kubernetes.io/*` สำหรับ best practices ใน production

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
  labels:
    app.kubernetes.io/name: my-application
    app.kubernetes.io/instance: my-application-prod
    app.kubernetes.io/version: "1.5.2"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: my-platform
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/created-by: "team-frontend"
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: my-application
      app.kubernetes.io/instance: my-application-prod
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-application
        app.kubernetes.io/instance: my-application-prod
        app.kubernetes.io/version: "1.5.2"
        app.kubernetes.io/component: frontend
    spec:
      containers:
      - name: app
        image: my-application:1.5.2
```

### ความหมายของแต่ละ Label

| Label Key | ตัวอย่างค่า | ความหมาย |
|-----------|------------|-----------|
| `app.kubernetes.io/name` | `my-app` | ชื่อ application |
| `app.kubernetes.io/instance` | `my-app-prod` | instance ที่เฉพาะเจาะจง |
| `app.kubernetes.io/version` | `1.2.3` | version ของ application |
| `app.kubernetes.io/component` | `frontend`, `backend`, `database` | component ใน architecture |
| `app.kubernetes.io/part-of` | `ecommerce-platform` | application หลักที่เป็นส่วนหนึ่ง |
| `app.kubernetes.io/managed-by` | `helm`, `kubectl`, `argocd` | เครื่องมือที่ manage |
| `app.kubernetes.io/created-by` | `team-frontend` | ทีมที่สร้าง |

### ตัวอย่าง Microservices ที่ใช้ Standard Labels

```yaml
# Payment Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    app.kubernetes.io/name: payment-service
    app.kubernetes.io/instance: payment-service-production
    app.kubernetes.io/version: "3.1.0"
    app.kubernetes.io/component: payment
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: argocd
spec:
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-service
      app.kubernetes.io/instance: payment-service-production
  template:
    metadata:
      labels:
        app.kubernetes.io/name: payment-service
        app.kubernetes.io/instance: payment-service-production
        app.kubernetes.io/version: "3.1.0"
    spec:
      containers:
      - name: payment
        image: payment-service:3.1.0
---
# User Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app.kubernetes.io/name: user-service
    app.kubernetes.io/instance: user-service-production
    app.kubernetes.io/version: "2.0.1"
    app.kubernetes.io/component: user-management
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: argocd
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: user-service
      app.kubernetes.io/instance: user-service-production
  template:
    metadata:
      labels:
        app.kubernetes.io/name: user-service
        app.kubernetes.io/instance: user-service-production
        app.kubernetes.io/version: "2.0.1"
    spec:
      containers:
      - name: user
        image: user-service:2.0.1
```

---

## Labels สำหรับ Cost Allocation

### ความสำคัญของ Cost Labels

ในองค์กรขนาดใหญ่ Labels ใช้สำหรับ:
- ติดตามค่าใช้จ่ายต่อทีม/โปรเจกต์
- Chargeback ไปยังแต่ละ Business Unit
- Optimize การใช้ Resources
- Budget Planning

### Schema สำหรับ Cost Allocation

```yaml
# Pod ที่มี cost allocation labels ครบ
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  labels:
    # Cost Allocation Labels
    cost-center: "CC-1234"           # รหัส cost center
    business-unit: "payments"         # business unit
    team: "backend-team"              # ทีมที่รับผิดชอบ
    project: "project-phoenix"        # โปรเจกต์
    environment: "production"         # environment
    # Application Labels
    app: "api-server"
    version: "2.1.0"
    # Billing Labels
    billing-category: "compute"
    owner-email: "backend@company.com"
spec:
  containers:
  - name: api
    image: api-server:2.1.0
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
```

### ใช้ Labels กับ Kubecost หรือ OpenCost

```bash
# ดู cost ต่อทีม (ต้องติดตั้ง Kubecost)
kubectl cost namespace \
    --historical \
    --window 7d \
    -l team=backend-team

# ดู cost ต่อ business unit
kubectl cost namespace \
    --historical \
    -l business-unit=payments

# ดู cost สรุปทั้งหมด
kubectl cost deployment \
    --all-namespaces \
    -l cost-center=CC-1234
```

### Script คำนวณ Cost Estimate

```bash
#!/bin/bash
# estimate-cost.sh - ประมาณค่าใช้จ่ายจาก resource requests

TEAM="${1:-backend-team}"

echo "Cost Estimate for Team: $TEAM"
echo "================================"

# ดู CPU requests รวม (หน่วย millicores)
CPU_TOTAL=$(kubectl get pods \
    --all-namespaces \
    -l "team=$TEAM" \
    -o jsonpath='{range .items[*]}{.spec.containers[*].resources.requests.cpu}{"\n"}{end}' | \
    sed 's/m//' | \
    awk '{sum += $1} END {print sum}')

# ดู Memory requests รวม (หน่วย Mi)
MEM_TOTAL=$(kubectl get pods \
    --all-namespaces \
    -l "team=$TEAM" \
    -o jsonpath='{range .items[*]}{.spec.containers[*].resources.requests.memory}{"\n"}{end}' | \
    sed 's/Mi//' | \
    awk '{sum += $1} END {print sum}')

echo "Total CPU Requests: ${CPU_TOTAL}m ($(echo "scale=2; $CPU_TOTAL/1000" | bc) cores)"
echo "Total Memory Requests: ${MEM_TOTAL}Mi ($(echo "scale=2; $MEM_TOTAL/1024" | bc) Gi)"

# ราคาประมาณ (สมมติ $0.048/core/hour และ $0.006/Gi/hour)
CPU_COST=$(echo "scale=4; $CPU_TOTAL/1000 * 0.048 * 24 * 30" | bc)
MEM_COST=$(echo "scale=4; $MEM_TOTAL/1024 * 0.006 * 24 * 30" | bc)
TOTAL=$(echo "scale=2; $CPU_COST + $MEM_COST" | bc)

echo "Estimated Monthly Cost: \$$TOTAL"
```

---

## Advanced Selector Examples

### Multiple Label Conditions

```bash
# Pods ที่มี app=nginx AND environment=production
kubectl get pods -l "app=nginx,environment=production"

# Pods ที่ version เป็น 1.0 หรือ 2.0 แต่ไม่ใช่ deprecated
kubectl get pods -l 'version in (1.0, 2.0),status notin (deprecated)'

# Pods ที่มี label 'team' อยู่ และ environment ไม่ใช่ test
kubectl get pods -l 'team,environment!=test'

# เลือก Nodes ที่ เป็น worker แต่ไม่ใช่ gpu node
kubectl get nodes -l 'role=worker,!gpu'
```

### Selector ใน kubectl

```bash
# ดู Deployments พร้อม Labels ที่ต้องการ
kubectl get deployments \
    -l "app.kubernetes.io/part-of=ecommerce-platform" \
    --all-namespaces

# ดู Resources หลายประเภทพร้อมกัน
kubectl get pods,services,deployments \
    -l app=my-app \
    -n production

# Count resources ต่อ label value
kubectl get pods --all-namespaces \
    --selector='environment=production' \
    -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name" | \
    wc -l
```

### Selector ใน Service

```yaml
# Service ที่เลือก Pods หลายเวอร์ชันในช่วง Canary Deployment
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  # ใช้ selector ที่กว้างพอสำหรับทุก version
  selector:
    app: my-app
    environment: production
    # ไม่ระบุ version - รับทั้ง stable และ canary
  ports:
  - port: 80
    targetPort: 8080
```

```yaml
# Service แยกสำหรับ version เฉพาะเจาะจง
apiVersion: v1
kind: Service
metadata:
  name: my-service-v2
spec:
  selector:
    app: my-app
    version: "2.0"
  ports:
  - port: 80
    targetPort: 8080
```

### Field Selectors (ไม่ใช่ Label แต่เกี่ยวข้อง)

```bash
# เลือก Pods ตาม status.phase
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase=Pending

# เลือก Pods ที่รันบน node เฉพาะ
kubectl get pods \
    --field-selector spec.nodeName=node-01

# รวม field selector กับ label selector
kubectl get pods \
    --field-selector status.phase=Running \
    -l app=my-app

# ดู Events ของ resource เฉพาะ
kubectl get events \
    --field-selector "involvedObject.name=my-pod,type=Warning"
```

---

## Labels ใน kubectl Output

### แสดง Labels เป็น Columns

```bash
# แสดง labels ทั้งหมด
kubectl get pods --show-labels

# แสดง labels เฉพาะที่ต้องการเป็น columns
kubectl get pods -L app,version,environment

# Output ตัวอย่าง:
# NAME              READY   STATUS    APP      VERSION   ENVIRONMENT
# api-pod-abc123    1/1     Running   my-api   1.5.0     production
# api-pod-def456    1/1     Running   my-api   1.5.0     production
# api-pod-xyz789    1/1     Running   my-api   1.5.0     staging

# แสดงหลาย resource types พร้อม labels
kubectl get pods,services -L app,environment -n production
```

### Custom Columns Output

```bash
# แสดงข้อมูล custom ด้วย labels
kubectl get pods \
    -o custom-columns=\
"NAME:.metadata.name,\
APP:.metadata.labels.app,\
VERSION:.metadata.labels.version,\
ENV:.metadata.labels.environment,\
NODE:.spec.nodeName" \
    --all-namespaces

# Export เป็น CSV สำหรับวิเคราะห์
kubectl get pods \
    --all-namespaces \
    -o custom-columns=\
"NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
APP:.metadata.labels.app,\
TEAM:.metadata.labels.team" \
    | tail -n +2 | tr -s ' ' ',' > pods-labels.csv
```

### JSONPath สำหรับดู Labels

```bash
# ดู label เฉพาะตัวของ resource
kubectl get pod my-pod \
    -o jsonpath='{.metadata.labels.app}'

# ดูทุก labels ของ Pods ทั้งหมด
kubectl get pods \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}'

# หา Pods ที่ไม่มี label 'version'
kubectl get pods -o json | \
    jq -r '.items[] | select(.metadata.labels.version == null) | .metadata.name'
```

---

## Workshop: Label-based Management

### ภาพรวม Workshop

Workshop นี้จะจัดการ microservices หลายตัวด้วย Labels:
1. Deploy หลาย services ด้วย standard labels
2. ทดสอบ Blue-Green deployment ด้วย Labels
3. ใช้ Labels สำหรับ monitoring queries

### ขั้นตอนที่ 1: Deploy Microservices

```bash
cat > microservices.yaml << 'EOF'
# API Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    app.kubernetes.io/name: api-gateway
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/component: gateway
    team: platform
    environment: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: api-gateway
  template:
    metadata:
      labels:
        app.kubernetes.io/name: api-gateway
        app.kubernetes.io/part-of: ecommerce
        team: platform
        environment: staging
        version: "1.0"
    spec:
      containers:
      - name: gateway
        image: nginx:1.21
        ports:
        - containerPort: 80
---
# Product Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app.kubernetes.io/name: product-service
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/component: products
    team: backend
    environment: staging
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: product-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: product-service
        app.kubernetes.io/part-of: ecommerce
        team: backend
        environment: staging
        version: "2.1"
    spec:
      containers:
      - name: product
        image: nginx:1.21
        ports:
        - containerPort: 8080
---
# Order Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/component: orders
    team: backend
    environment: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service
        app.kubernetes.io/part-of: ecommerce
        team: backend
        environment: staging
        version: "1.5"
    spec:
      containers:
      - name: order
        image: nginx:1.21
        ports:
        - containerPort: 8080
EOF

kubectl apply -f microservices.yaml
kubectl get pods --show-labels
```

### ขั้นตอนที่ 2: Query ด้วย Labels

```bash
# ดู services ทั้งหมดของ ecommerce platform
kubectl get all \
    -l "app.kubernetes.io/part-of=ecommerce"

# ดู backend team resources
kubectl get deployments \
    -l "team=backend"

# ดู Pod counts ต่อ component
for component in gateway products orders; do
  count=$(kubectl get pods \
      -l "app.kubernetes.io/part-of=ecommerce" \
      --field-selector status.phase=Running | \
      grep -c "$component" || echo 0)
  echo "$component: $count running pods"
done
```

### ขั้นตอนที่ 3: Blue-Green Deployment ด้วย Labels

```bash
# สร้าง Service ที่ชี้ไปหา "green" version
cat > bg-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: product-service-svc
spec:
  selector:
    app.kubernetes.io/name: product-service
    slot: green              # ชี้ไปหา green
  ports:
  - port: 80
    targetPort: 8080
EOF
kubectl apply -f bg-service.yaml

# Label Pods เป็น green (current)
kubectl label pods \
    -l "app.kubernetes.io/name=product-service" \
    slot=green

# Deploy Blue version (ใหม่)
cat > product-blue.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: product-service
      slot: blue
  template:
    metadata:
      labels:
        app.kubernetes.io/name: product-service
        slot: blue
        version: "2.2"
    spec:
      containers:
      - name: product
        image: nginx:1.22  # new version
        ports:
        - containerPort: 8080
EOF
kubectl apply -f product-blue.yaml

# รอ Blue version ready
kubectl rollout status deployment/product-service-blue

# Switch traffic ไปที่ Blue
kubectl patch service product-service-svc \
    -p '{"spec":{"selector":{"slot":"blue"}}}'

# ตรวจสอบว่า traffic ไปที่ Blue แล้ว
kubectl get service product-service-svc -o yaml | grep -A5 selector
```

### ขั้นตอนที่ 4: Cleanup ด้วย Labels

```bash
# ลบทุกอย่างใน staging environment
kubectl delete all \
    -l "environment=staging"

# ลบเฉพาะ backend team resources
kubectl delete deployments \
    -l "team=backend"

# ลบทั้ง platform ในคราวเดียว
kubectl delete all \
    -l "app.kubernetes.io/part-of=ecommerce"
```

---

## แบบฝึกหัด: Labels and Selectors

### แบบฝึกหัดที่ 1: เพิ่ม Labels ให้ Resources ที่มีอยู่

**โจทย์**: เพิ่ม label `monitored=true` ให้ Pods ทุกตัวที่รันอยู่ใน namespace default

**เฉลย**:
```bash
# ดู Pods ที่มี
kubectl get pods

# เพิ่ม label ทุก Pods
kubectl label pods --all monitored=true

# ตรวจสอบ
kubectl get pods --show-labels | grep monitored

# เพิ่ม label เฉพาะ Pods ที่ running
kubectl get pods --field-selector status.phase=Running \
    -o name | xargs kubectl label monitored=true
```

---

### แบบฝึกหัดที่ 2: ใช้ Set-based Selector

**โจทย์**: แสดง Pods ทั้งหมดที่ environment เป็น `staging` หรือ `development` แต่ไม่ใช่ `deprecated`

**เฉลย**:
```bash
# สร้าง Pods ทดสอบ
kubectl run pod-staging \
    --image=nginx \
    --labels="app=test,environment=staging"
kubectl run pod-dev \
    --image=nginx \
    --labels="app=test,environment=development"
kubectl run pod-prod \
    --image=nginx \
    --labels="app=test,environment=production"
kubectl run pod-deprecated \
    --image=nginx \
    --labels="app=test,environment=deprecated"

# Query ด้วย set-based selector
kubectl get pods \
    -l 'environment in (staging, development)'

# ผลลัพธ์ที่ถูกต้อง: จะเห็นเฉพาะ pod-staging และ pod-dev
```

---

### แบบฝึกหัดที่ 3: NodeSelector

**โจทย์**: กำหนดให้ Pod รันเฉพาะบน Nodes ที่มี SSD disk

**เฉลย**:
```bash
# เพิ่ม label ให้ Node (ต้องเป็น admin)
kubectl label node <node-name> disk-type=ssd

# สร้าง Pod ที่ต้องการ SSD
cat > ssd-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: ssd-workload
spec:
  nodeSelector:
    disk-type: ssd
  containers:
  - name: app
    image: nginx
EOF
kubectl apply -f ssd-pod.yaml

# ตรวจสอบว่า Pod รันบน SSD node
kubectl get pod ssd-workload -o wide
kubectl describe pod ssd-workload | grep Node:
```

---

### แบบฝึกหัดที่ 4: Service Selector

**โจทย์**: สร้าง Service ที่รับ traffic เฉพาะจาก Pods ที่เป็น `ready=true` (simulated)

**เฉลย**:
```yaml
# สร้าง Deployment พร้อม labels
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        traffic: enabled
    spec:
      containers:
      - name: app
        image: nginx:1.21
---
# Service เลือก Pods ที่ traffic=enabled เท่านั้น
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app
    traffic: enabled
  ports:
  - port: 80
    targetPort: 80
```

```bash
# ทดสอบโดย disable traffic บาง Pod
kubectl label pod <pod-name> traffic=disabled --overwrite
# Pod นั้นจะถูกถอดออกจาก Service endpoints

# ดู endpoints
kubectl get endpoints my-app-svc
```

---

### แบบฝึกหัดที่ 5: ลบ Label

**โจทย์**: ลบ label `monitored` ออกจาก Pod เฉพาะตัว และ `environment` จากทุก Pods

**เฉลย**:
```bash
# ลบ label จาก Pod เฉพาะตัว (ใส่ - ท้ายชื่อ label)
kubectl label pod my-pod monitored-

# ลบ label จากทุก Pods
kubectl label pods --all environment-

# ยืนยันว่าถูกลบแล้ว
kubectl get pods --show-labels
```

---

## สรุปทบทวน Labels and Selectors

### Cheat Sheet

```bash
# เพิ่ม/แก้ไข label
kubectl label <resource> <name> key=value
kubectl label <resource> <name> key=value --overwrite

# ลบ label
kubectl label <resource> <name> key-

# เพิ่ม label ทุก resources
kubectl label <resource> --all key=value

# Query ด้วย selector
kubectl get pods -l key=value
kubectl get pods -l 'key in (v1,v2)'
kubectl get pods -l 'key notin (v1,v2)'
kubectl get pods -l key          # has label
kubectl get pods -l '!key'       # not has label

# แสดง labels
kubectl get pods --show-labels
kubectl get pods -L key1,key2    # แสดงเป็น columns
```

Labels เป็น core mechanism ที่ทุก Kubernetes component ใช้ในการ identify และ group resources การเข้าใจ Labels อย่างถ่องแท้จะช่วยให้จัดการ cluster ขนาดใหญ่ได้อย่างมีประสิทธิภาพ
