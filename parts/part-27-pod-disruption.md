# Part 27: Pod Disruption Budget (PDB) - ป้องกัน Downtime ขณะ Maintenance

## สารบัญ
1. [Pod Disruption Budget คืออะไร](#pod-disruption-budget-คืออะไร)
2. [Voluntary vs Involuntary Disruptions](#voluntary-vs-involuntary-disruptions)
3. [PDB YAML ละเอียด](#pdb-yaml-ละเอียด)
4. [minAvailable vs maxUnavailable](#minavailable-vs-maxunavailable)
5. [PDB กับ Rolling Updates](#pdb-กับ-rolling-updates)
6. [Workshop: ทำ Node Maintenance อย่างปลอดภัย](#workshop-ทำ-node-maintenance-อย่างปลอดภัย)
7. [Workshop: PDB สำหรับ StatefulSet](#workshop-pdb-สำหรับ-statefulset)
8. [Workshop: Multi-zone High Availability](#workshop-multi-zone-high-availability)
9. [Troubleshooting PDB](#troubleshooting-pdb)
10. [Best Practices](#best-practices)

---

## Pod Disruption Budget คืออะไร

**Pod Disruption Budget (PDB)** คือ Kubernetes resource ที่กำหนดจำนวน Pod **ขั้นต่ำที่ต้องรัน** (หรือสูงสุดที่หยุดได้) ขณะเกิด **voluntary disruptions**:

### ทำไมต้องใช้ PDB

```
ปัญหาที่ไม่มี PDB:
─────────────────────────────────────────────────────────
  Deployment: 3 replicas
  
  Node maintenance (drain):
  → kubectl drain node-1  → evict Pod-1
  → kubectl drain node-2  → evict Pod-2  ← อาจ evict พร้อมกัน!
  → kubectl drain node-3  → evict Pod-3
  
  ผลลัพธ์: ไม่มี Pod เลย → Service Down! 💥

ด้วย PDB (minAvailable: 2):
─────────────────────────────────────────────────────────
  kubectl drain node-1  → evict Pod-1  (เหลือ 2 pods → OK)
  kubectl drain node-2  → evict Pod-2  (จะเหลือ 1 < 2 → BLOCKED!)
  
  drain node-2 รอจนกว่า Pod ใหม่ถูกสร้างบน node อื่น
  → ไม่มี downtime ✓
```

---

## Voluntary vs Involuntary Disruptions

### Voluntary Disruptions (PDB ควบคุมได้)

สาเหตุที่เกิดจากการกระทำของ admin หรือระบบ:

- `kubectl drain node`: drain node เพื่อ maintenance
- `kubectl delete pod`: ลบ Pod โดยตรง  
- การ update Deployment (rolling update)
- Cluster autoscaler ลด node
- Node preemption (สำหรับ higher priority pods)

### Involuntary Disruptions (PDB ควบคุมไม่ได้)

สาเหตุที่เกิดจาก hardware/software failure:

- Node hardware failure
- Node kernel panic
- Network partition
- Pod crash (OOMKilled, CrashLoopBackOff)
- ไฟดับ

```
PDB ปกป้องเฉพาะ Voluntary Disruptions!
Involuntary disruptions ต้องใช้ Replication + Anti-affinity
```

---

## PDB YAML ละเอียด

### PDB ด้วย minAvailable

```yaml
# pdb-min-available.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  # ต้องการ Pod รันอยู่อย่างน้อย 2 ตัว
  minAvailable: 2
  
  # Selector ต้องตรงกับ Pod ที่ต้องการปกป้อง
  selector:
    matchLabels:
      app: web-app
```

### PDB ด้วย maxUnavailable

```yaml
# pdb-max-unavailable.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # ยอมให้ Pod หยุดทำงานได้สูงสุด 1 ตัว
  maxUnavailable: 1
  
  selector:
    matchLabels:
      app: api-server
```

### PDB ด้วย Percentage

```yaml
# pdb-percentage.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
  namespace: production
spec:
  # ต้องการ Pod รันอยู่อย่างน้อย 80%
  minAvailable: "80%"
  
  # หรือ ยอมให้หยุดได้สูงสุด 20%
  # maxUnavailable: "20%"
  
  selector:
    matchLabels:
      app: worker
```

### PDB ที่สมบูรณ์

```yaml
# full-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
  namespace: production
  labels:
    app: critical-app
    team: platform
  annotations:
    description: "PDB for critical-app - minimum 3 replicas must be available"
spec:
  # ใช้ minAvailable หรือ maxUnavailable (ไม่ใช้ทั้งสองพร้อมกัน)
  minAvailable: 3
  
  selector:
    matchLabels:
      app: critical-app
      tier: web
```

---

## minAvailable vs maxUnavailable

### minAvailable

```
minAvailable: N = ต้องมี Pod รัน อย่างน้อย N ตัว ตลอดเวลา

ตัวอย่าง:
  Deployment replicas: 5
  PDB minAvailable: 3
  
  จำนวน Pod ที่ evict ได้พร้อมกัน = 5 - 3 = 2
  
  minAvailable: "80%"
  จำนวน Pod ที่ต้องรัน = ceil(5 × 0.8) = ceil(4) = 4
  จำนวน Pod ที่ evict ได้พร้อมกัน = 5 - 4 = 1
```

### maxUnavailable

```
maxUnavailable: N = ยอมให้ Pod หยุดได้พร้อมกันสูงสุด N ตัว

ตัวอย่าง:
  Deployment replicas: 5
  PDB maxUnavailable: 2
  
  จำนวน Pod ที่รัน ต้องมีอย่างน้อย = 5 - 2 = 3
  
  maxUnavailable: "20%"
  จำนวน Pod ที่ evict ได้ = floor(5 × 0.2) = floor(1) = 1
  จำนวน Pod ที่ต้องรัน = 5 - 1 = 4
```

### เปรียบเทียบ

```
Deployment replicas: 5, PDB: minAvailable=3

Scenario:
  Current pods: [p1, p2, p3, p4, p5]
  
  Drain node-A (has p1, p2):
  → Evict p1 → pods: [p2, p3, p4, p5] = 4 ≥ 3 ✓
  → Evict p2 → pods: [p3, p4, p5] = 3 ≥ 3 ✓
  → Drain complete
  
  Drain node-B (has p3, p4):
  → Evict p3 → pods: [p4, p5] = 2 < 3 ✗ BLOCKED
  → รอให้ Pod ใหม่ถูกสร้างบน node อื่น
  → Pod ใหม่ p6 ถูกสร้าง: [p4, p5, p6] = 3 ≥ 3 ✓
  → Evict p3 → [p4, p5, p6] = 3 ≥ 3 ✓
  
  ไม่มี downtime!
```

---

## PDB กับ Rolling Updates

### Rolling Update ของ Deployment

```yaml
# deployment กับ PDB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # สร้าง Pod ใหม่เพิ่มได้ 2 ตัว
      maxUnavailable: 1  # หยุด Pod เก่าได้ 1 ตัวพร้อมกัน
  selector:
    matchLabels:
      app: web-app
  template:
    ...
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  minAvailable: 3   # ต้องมี 3 pods ตลอดเวลา
  selector:
    matchLabels:
      app: web-app
```

### ตรวจสอบ PDB Status

```bash
# ดู PDB status
kubectl get pdb -n production

# Output:
# NAME          MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# web-app-pdb   3               N/A               2                     5d

# ALLOWED DISRUPTIONS = 5 (current) - 3 (minAvailable) = 2
# ยังยอมให้ evict ได้ 2 pods

# ดูรายละเอียด
kubectl describe pdb web-app-pdb -n production
```

---

## Workshop: ทำ Node Maintenance อย่างปลอดภัย

เป้าหมาย: ทำ Node maintenance โดยไม่มี downtime

### 1. Setup: Deploy Application

```yaml
# web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      # Spread pods across nodes
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
      
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

### 2. สร้าง PDB

```yaml
# web-app-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  minAvailable: 3    # Deployment มี 4 replicas → evict ได้ 1 ตัว
  selector:
    matchLabels:
      app: web-app
```

### 3. Deploy และดู Distribution

```bash
# Apply
kubectl apply -f web-app.yaml
kubectl apply -f web-app-pdb.yaml

# ดู pod distribution บน nodes
kubectl get pods -n production -l app=web-app -o wide

# ดู PDB status
kubectl get pdb web-app-pdb -n production
# ALLOWED DISRUPTIONS = 4 - 3 = 1

# ดู pods ที่อยู่บนแต่ละ node
kubectl get pods -n production -l app=web-app \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'
```

### 4. ทำ Node Maintenance (Drain)

```bash
# ดู nodes ทั้งหมด
kubectl get nodes

# Cordon node: หยุดรับ pods ใหม่ (แต่ pods เก่ายังรัน)
kubectl cordon node-1
# node/node-1 cordoned

# ตรวจสอบ node status
kubectl get node node-1
# STATUS: Ready,SchedulingDisabled

# Drain node: evict pods ออก
kubectl drain node-1 \
  --ignore-daemonsets \      # ไม่ evict DaemonSet pods
  --delete-emptydir-data \   # อนุญาตลบ emptyDir data
  --timeout=300s              # timeout 5 นาที

# ถ้า PDB บล็อก drain จะรอและแสดง:
# evicting pod production/web-app-xxx
# error when evicting pods/"web-app-xxx" -n "production" (will retry after 5s)
# Waiting for pod web-app-xxx to terminate...

# ดู pod ใหม่ที่ถูกสร้างบน node อื่น
kubectl get pods -n production -l app=web-app -o wide -w

# หลัง maintenance เสร็จ: uncordon node
kubectl uncordon node-1
```

### 5. ตรวจสอบ Service Availability ขณะ Drain

```bash
# Terminal 1: Monitor pods
watch -n 1 kubectl get pods -n production -l app=web-app -o wide

# Terminal 2: ส่ง requests อย่างต่อเนื่อง
while true; do
  curl -s -o /dev/null -w "%{http_code}\n" http://web-app.production.svc.cluster.local/
  sleep 0.1
done
# ควรได้ 200 ตลอดเวลา

# Terminal 3: Drain node
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
```

---

## Workshop: PDB สำหรับ StatefulSet

### สร้าง StatefulSet + PDB

```yaml
# postgres-with-pdb.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        role: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: "secret"
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 15
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
# PDB สำหรับ PostgreSQL cluster
# ต้องมี primary (postgres-0) อยู่เสมอ
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  # ต้องมี pod อย่างน้อย 2 ตัว (quorum)
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
```

### ทดสอบ: ลอง Drain Node ที่มี postgres-0

```bash
# ดูว่า postgres-0 อยู่ Node ไหน
kubectl get pod postgres-0 -n production -o wide

# Drain node นั้น
NODE=$(kubectl get pod postgres-0 -n production -o jsonpath='{.spec.nodeName}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

# PDB จะอนุญาตให้ evict postgres-0 เฉพาะถ้า pod อื่นรันอยู่ครบ 2 ตัว
# StatefulSet จะสร้าง postgres-0 ใหม่บน node อื่น
```

---

## Workshop: Multi-zone High Availability

### PDB ที่ป้องกัน Zone Failure

```yaml
# multi-zone-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-app-pdb
  namespace: production
spec:
  # ต้องมี pod อย่างน้อย 2 ตัว (จาก 3 zones)
  # ถ้า 1 zone fail → ยังมี 2 zones → service ยังทำงานได้
  minAvailable: 2
  selector:
    matchLabels:
      app: ha-app
---
# Deployment ที่ spread across zones
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
  namespace: production
spec:
  replicas: 6   # 2 pods per zone
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      # Spread across zones
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ha-app
      
      # Anti-affinity: ไม่ให้ 2 pods อยู่บน node เดียวกัน
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [ha-app]
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
```

### ตรวจสอบ Zone Distribution

```bash
# ดู pods กระจายอยู่ zones ไหน
kubectl get pods -n production -l app=ha-app \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'

# ดู node zones
kubectl get nodes -o custom-columns='NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'

# ดู PDB status
kubectl describe pdb ha-app-pdb -n production
```

---

## Troubleshooting PDB

### ปัญหา: Drain ถูก Block นานเกิน

```bash
# ดูว่า PDB block drain อยู่หรือไม่
kubectl get pdb -n production

# ดู pods ที่ต้อง evict
kubectl get pods -n production -o wide | grep <node-name>

# ดู disruption ที่เกิดขึ้น
kubectl describe pdb web-app-pdb -n production
# Disruptions allowed: 0  ← ถ้าเป็น 0 drain จะรอ

# Force evict (bypass PDB - ระวัง!)
kubectl delete pod <pod-name> -n production --grace-period=0 --force

# หรือลบ PDB ชั่วคราว (ระวัง!)
kubectl delete pdb web-app-pdb -n production
kubectl drain <node-name> --ignore-daemonsets
kubectl apply -f web-app-pdb.yaml  # restore
```

### ปัญหา: PDB ไม่ match Pods

```bash
# ดู PDB selector
kubectl get pdb web-app-pdb -n production -o yaml | grep selector

# ดู pod labels
kubectl get pods -n production -l app=web-app --show-labels

# ตรวจสอบว่า selector match
kubectl get pods -n production \
  -l app=web-app \
  -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels'
```

### ปัญหา: PDB แสดง "0 allowed disruptions" ทั้งๆ ที่มี Pods พอ

```bash
# ดู pods ที่ไม่ ready
kubectl get pods -n production -l app=web-app \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[0].ready,STATUS:.status.phase'

# PDB นับเฉพาะ Ready pods!
# ถ้า pod ไม่ ready → นับเป็น unavailable → PDB อาจ block

# ตรวจสอบว่า pods ทั้งหมด ready
kubectl rollout status deployment/web-app -n production
```

### คำสั่ง PDB ที่ใช้บ่อย

```bash
# ดู PDB ทั้งหมด
kubectl get pdb -A

# ดู PDB รายละเอียด
kubectl describe pdb my-pdb -n production

# ดู PDB ที่มี 0 allowed disruptions (อาจ block operations)
kubectl get pdb -A -o json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data.get('items', []):
    name = item['metadata']['name']
    ns = item['metadata']['namespace']
    status = item.get('status', {})
    allowed = status.get('disruptionsAllowed', 0)
    if allowed == 0:
        print(f'WARNING: {ns}/{name} - 0 disruptions allowed!')
"

# ลบ PDB
kubectl delete pdb my-pdb -n production

# ดู drain status
kubectl get pods -o wide -A | grep Terminating
```

---

## Best Practices

### 1. ตั้งค่า PDB สำหรับทุก Production Workload

```yaml
# Rule of thumb:
# - Replicas 1: ไม่สามารถตั้ง minAvailable=1 แบบ meaningful
#   → เพิ่ม replicas เป็น 2 ก่อน
# - Replicas 2: minAvailable=1 หรือ maxUnavailable=1
# - Replicas 3+: minAvailable ≥ 2 หรือ maxUnavailable ≤ 1
# - Replicas 5+: maxUnavailable: "20%"

# ตัวอย่าง
spec:
  minAvailable: 2  # สำหรับ 3+ replicas
  # หรือ
  maxUnavailable: 1  # conservative
  # หรือ
  minAvailable: "80%"  # percentage สำหรับ large deployments
```

### 2. ทดสอบ PDB ด้วย Dry-run

```bash
# ทดสอบ drain โดยไม่ทำจริง
kubectl drain node-1 \
  --ignore-daemonsets \
  --dry-run=client

# Output จะแสดงว่า pod ไหนจะถูก evict
# และ PDB ไหนที่อาจบล็อก
```

### 3. ตั้ง PDB ให้เหมาะกับ Deployment Strategy

```yaml
# ถ้า Deployment ใช้ maxUnavailable: 1
# PDB ต้องอนุญาต 1 disruption
spec:
  maxUnavailable: 1   # ตรงกับ Deployment strategy

# ถ้า StatefulSet:
spec:
  minAvailable: 2     # สำหรับ 3-node cluster (quorum)
```

### 4. Monitor PDB Violations

```bash
# ดู PDB events
kubectl get events -A | grep "violates PodDisruptionBudget"

# Script ตรวจสอบ PDB health
kubectl get pdb -A -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name,DESIRED:.status.desiredHealthy,CURRENT:.status.currentHealthy,ALLOWED:.status.disruptionsAllowed'
```

### 5. PDB สำหรับ Kubernetes System Components

```yaml
# PDB สำหรับ kube-dns
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: coredns-pdb
  namespace: kube-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
```

---

## Workshop: PDB สำหรับ Critical Microservices

เป้าหมาย: กำหนด PDB สำหรับ microservices หลายตัวอย่างครบถ้วน

### 1. สร้าง Application Topology

```yaml
# microservices-pdb.yaml
# PDB สำหรับ API Gateway (traffic entry point - critical!)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: production
  labels:
    tier: critical
    team: platform
spec:
  maxUnavailable: 0    # ← ห้าม unavailable เลย! (ถ้า replicas=2+ จะยอม 0)
  selector:
    matchLabels:
      app: api-gateway
      tier: critical
---
# PDB สำหรับ Auth Service
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: auth-service-pdb
  namespace: production
spec:
  minAvailable: 2      # ต้องมีอย่างน้อย 2 replicas
  selector:
    matchLabels:
      app: auth-service
---
# PDB สำหรับ Payment Service (critical!)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: production
  annotations:
    note: "Payment service PDB - NEVER bring below 2 replicas"
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment-service
---
# PDB สำหรับ Worker Service (less critical)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
  namespace: production
spec:
  maxUnavailable: "30%"    # 30% ลดได้
  selector:
    matchLabels:
      app: worker
---
# PDB สำหรับ Cache Service
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cache-pdb
  namespace: production
spec:
  minAvailable: "50%"   # ต้องมีอย่างน้อย 50%
  selector:
    matchLabels:
      app: redis-cluster
```

### 2. Deployments ที่ตรงกับ PDB

```yaml
# microservices-deployments.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api-gateway
      tier: critical
  template:
    metadata:
      labels:
        app: api-gateway
        tier: critical
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [api-gateway]
            topologyKey: kubernetes.io/hostname
      containers:
      - name: api-gateway
        image: nginx:1.21
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          periodSeconds: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth
        image: auth-service:latest
        resources:
          requests:
            cpu: "300m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      # กระจายไปคนละ zone
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: payment-service
      containers:
      - name: payment
        image: payment-service:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
```

### 3. Script ตรวจสอบ PDB ก่อน Node Maintenance

```bash
#!/bin/bash
# pre-drain-check.sh: ตรวจสอบ PDB ก่อน drain node

NODE=${1:-$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')}

echo "=== Pre-drain Check for Node: $NODE ==="
echo ""

# ดู pods บน node นี้
echo "Pods on this node:"
kubectl get pods -A --field-selector spec.nodeName=$NODE \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase'
echo ""

# ตรวจสอบ PDB ที่อาจ block
echo "Checking PDBs that might block drain..."
kubectl get pdb -A -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name,MIN_AVAIL:.spec.minAvailable,MAX_UNAVAIL:.spec.maxUnavailable,DISRUPTIONS:.status.disruptionsAllowed'
echo ""

# PDB ที่มี 0 disruptions allowed
echo "PDBs with 0 disruptions allowed (WILL BLOCK drain):"
kubectl get pdb -A -o json | python3 -c "
import json, sys
data = json.load(sys.stdin)
blocked = False
for item in data.get('items', []):
    name = item['metadata']['name']
    ns = item['metadata']['namespace']
    allowed = item.get('status', {}).get('disruptionsAllowed', 0)
    if allowed == 0:
        print(f'  ⚠️  {ns}/{name}: 0 disruptions allowed - DRAIN MAY BE BLOCKED')
        blocked = True
if not blocked:
    print('  ✓ No PDBs are blocking disruptions')
"
echo ""

# Dry-run drain
echo "Dry-run drain result:"
kubectl drain $NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --dry-run=client 2>&1 | head -30
```

### 4. ทดสอบ Cluster Upgrade Scenario

```bash
# Simulate cluster upgrade workflow

# 1. ตรวจสอบ PDB ทั้งหมด
kubectl get pdb -A

# 2. ดู nodes
kubectl get nodes

# 3. Cordon master node
kubectl cordon k8s-master-1

# 4. Drain worker nodes ทีละ node
for NODE in $(kubectl get nodes -l node-role.kubernetes.io/worker= -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== Draining $NODE ==="
  
  # ตรวจสอบก่อน
  kubectl get pdb -A -o json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data['items']:
    if item['status']['disruptionsAllowed'] == 0:
        print(f\"WARNING: {item['metadata']['namespace']}/{item['metadata']['name']} blocks drain!\")
"
  
  # Drain
  kubectl drain $NODE \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --timeout=300s
  
  # ทำ upgrade operations ที่ node
  echo "Upgrading $NODE..."
  # ssh $NODE "sudo kubeadm upgrade node && sudo apt-get install -y kubelet kubeadm"
  
  # Uncordon
  kubectl uncordon $NODE
  
  echo "=== $NODE upgraded ==="
  sleep 30  # รอ pod schedule ใหม่ก่อน drain node ถัดไป
done
```

---

## สรุป

Pod Disruption Budget เป็นเครื่องมือสำคัญสำหรับ:

1. **Node Maintenance**: drain node โดยไม่มี downtime
2. **Cluster Upgrades**: upgrade kubernetes version อย่างปลอดภัย
3. **Scaling Operations**: ลด nodes ด้วย cluster autoscaler
4. **High Availability**: รับประกัน minimum replicas ตลอดเวลา

### Quick Reference

```bash
# สร้าง PDB อย่างรวดเร็ว
kubectl create poddisruptionbudget my-pdb \
  --selector=app=my-app \
  --min-available=2 \
  -n production

# ดู PDB status summary
kubectl get pdb -A -o custom-columns=\
'NS:.metadata.namespace,NAME:.metadata.name,MIN:.spec.minAvailable,MAX:.spec.maxUnavailable,ALLOWED:.status.disruptionsAllowed,CURRENT:.status.currentHealthy,DESIRED:.status.desiredHealthy'

# ดู PDB YAML
kubectl get pdb my-pdb -n production -o yaml

# ลบ PDB
kubectl delete pdb my-pdb -n production
```

ข้อสำคัญ:
- PDB ปกป้องเฉพาะ **voluntary disruptions**
- ใช้ minAvailable สำหรับ absolute minimum
- ใช้ maxUnavailable สำหรับ maximum disruptions
- Percentage เหมาะกับ large deployments
- ต้องมี replicas ≥ 2 จึงจะ PDB มีความหมาย
- **ทดสอบ PDB** ด้วย dry-run drain ก่อน maintenance จริง

ในบทต่อไป เราจะเรียนรู้เรื่อง **Init Containers** - containers พิเศษที่รันก่อน main container
