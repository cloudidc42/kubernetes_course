# Part 26: Resource Limits - การจัดการทรัพยากรใน Kubernetes

## สารบัญ
1. [Resource Requests vs Limits](#resource-requests-vs-limits)
2. [CPU Management](#cpu-management)
3. [Memory Management](#memory-management)
4. [QoS Classes](#qos-classes)
5. [LimitRange](#limitrange)
6. [ResourceQuota](#resourcequota)
7. [Workshop: กำหนด Resources อย่างเหมาะสม](#workshop-กำหนด-resources-อย่างเหมาะสม)
8. [Workshop: Namespace Resource Management](#workshop-namespace-resource-management)
9. [Troubleshooting Resource Issues](#troubleshooting-resource-issues)
10. [Best Practices](#best-practices)

---

## Resource Requests vs Limits

### Requests

**Request** คือ resources ที่ **รับประกัน** ว่า container จะได้รับ:
- Kubernetes scheduler ใช้ requests เพื่อตัดสินใจ schedule Pod บน Node ไหน
- Node ต้องมี resources ว่างอย่างน้อยเท่ากับ requests
- Container **รับประกันว่าจะได้** resources เท่ากับ requests

### Limits

**Limit** คือ resources **สูงสุด** ที่ container ใช้ได้:
- CPU: ถ้า container ใช้เกิน limit → **throttled** (ช้าลง แต่ไม่ kill)
- Memory: ถ้า container ใช้เกิน limit → **OOMKilled** (killed ทันที)

### ภาพรวม

```
Node Capacity: 8 CPU, 16Gi Memory
─────────────────────────────────────────────────────────────

Pod A: request=2CPU/4Gi, limit=4CPU/8Gi
Pod B: request=2CPU/4Gi, limit=4CPU/8Gi

Scheduled to this node:
  ┌──────────────────────────────────────────────┐
  │ Node (8CPU, 16Gi)                            │
  │                                              │
  │  ┌─────────────┐  ┌─────────────┐           │
  │  │  Pod A      │  │  Pod B      │           │
  │  │ req: 2CPU   │  │ req: 2CPU   │           │
  │  │ lim: 4CPU   │  │ lim: 4CPU   │           │
  │  └─────────────┘  └─────────────┘           │
  │                                              │
  │  Used requests: 4CPU/8Gi ✓                  │
  │  Available: 4CPU/8Gi                         │
  └──────────────────────────────────────────────┘

ถ้า Pod A ใช้ CPU เกิน 4CPU → Throttled
ถ้า Pod A ใช้ Memory เกิน 8Gi → OOMKilled!
```

### Resource Units

```
CPU Units:
- 1 CPU = 1000 millicores (m)
- 0.5 CPU = 500m
- 100m = 0.1 CPU

Memory Units:
- Ki (Kibibyte)  = 1,024 bytes
- Mi (Mebibyte)  = 1,024 Ki = 1,048,576 bytes
- Gi (Gibibyte)  = 1,024 Mi = 1,073,741,824 bytes
- K = 1,000 bytes (ต่างจาก Ki!)
- M = 1,000 K
- G = 1,000 M
```

### ตั้งค่า Resources

```yaml
# resource-example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.21
    
    resources:
      # Requests: รับประกันว่าได้
      requests:
        cpu: "250m"    # 0.25 CPU
        memory: "64Mi" # 64 MiB
      
      # Limits: สูงสุดที่ใช้ได้
      limits:
        cpu: "500m"    # 0.5 CPU
        memory: "128Mi" # 128 MiB
```

---

## CPU Management

### CPU Throttling

เมื่อ container ใช้ CPU เกิน limit:
- Linux CFS (Completely Fair Scheduler) จะ **throttle** process
- Container ยังคงทำงาน แต่ **ช้าลง**
- ไม่ถูก kill

```bash
# ดู CPU throttling metrics
kubectl exec my-pod -- cat /sys/fs/cgroup/cpu/cpu.stat
# throttled_time แสดงเวลาที่ถูก throttle (nanoseconds)

# ดู CPU metrics ด้วย kubectl top
kubectl top pod my-pod --containers

# ดู throttling ใน Prometheus
# container_cpu_cfs_throttled_seconds_total
```

### CPU Request ส่งผลต่อ Scheduling

```
Node-1: 4 CPU available
Node-2: 2 CPU available

Pod ที่ต้องการ request 3 CPU:
→ ไม่ schedule บน Node-2 (ไม่พอ)
→ Schedule บน Node-1 ✓

แม้ว่า Node-2 จะมี capacity จริงเหลืออยู่มาก
แต่ request ที่ยังไม่ได้ถูก allocate ต้องพอ
```

### CPU Bursting

```yaml
# Container ที่ใช้ CPU burst
resources:
  requests:
    cpu: "100m"   # ได้รับรับประกัน 0.1 CPU
  limits:
    cpu: "1000m"  # สามารถ burst ถึง 1 CPU ถ้า Node มีเหลือ
```

---

## Memory Management

### OOMKilled

เมื่อ container ใช้ memory เกิน limit:
- Linux OOM Killer จะ **kill process ทันที**
- Pod จะ restart (ถ้า restartPolicy อนุญาต)
- แสดงใน `kubectl describe pod` ว่า `OOMKilled`

```bash
# ตรวจสอบ OOMKilled
kubectl describe pod my-pod | grep OOMKilled
# State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

# ดู memory usage
kubectl top pod my-pod --containers

# ดู memory limit ของ container
kubectl get pod my-pod -o yaml | grep -A 5 limits
```

### Memory Request

```yaml
resources:
  requests:
    memory: "128Mi"  # รับประกัน 128 MiB
  limits:
    memory: "256Mi"  # สูงสุด 256 MiB
```

**ข้อควรระวัง**: Memory ไม่ compressible เหมือน CPU
- CPU throttle → process ช้าลง (recoverable)
- Memory OOM → process ถูก kill (abrupt termination)

### Memory OverCommit

Kubernetes อนุญาต memory overcommit (total requests > node capacity):

```
Node Memory: 8Gi

Pod A: request=4Gi, limit=8Gi
Pod B: request=4Gi, limit=8Gi

Total requests = 8Gi (เท่ากับ node capacity)

ถ้าทั้งคู่ใช้ memory ใกล้ limit:
→ OOM Killer จะ kill pod ที่มี priority ต่ำกว่า
→ Priority ขึ้นกับ QoS Class
```

---

## QoS Classes

Kubernetes จัดประเภท Pod เป็น 3 QoS Classes ตาม resource configuration:

### 1. Guaranteed (QoS สูงสุด)

**เงื่อนไข**: ทุก container ต้องมี requests = limits สำหรับทั้ง CPU และ Memory

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"    # = requests
    memory: "256Mi" # = requests
# QoS: Guaranteed
```

- OOM killer ไม่ kill Pod นี้ก่อน Burstable/BestEffort
- เหมาะสำหรับ: critical workloads, databases

### 2. Burstable (QoS กลาง)

**เงื่อนไข**: มี requests หรือ limits แต่ไม่ใช่ Guaranteed

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "128Mi"
  limits:
    cpu: "500m"    # != requests
    memory: "256Mi" # != requests
# QoS: Burstable
```

- OOM killer kill ก่อน Guaranteed แต่หลัง BestEffort
- เหมาะสำหรับ: web applications, APIs

### 3. BestEffort (QoS ต่ำสุด)

**เงื่อนไข**: ไม่มี requests หรือ limits เลย

```yaml
# ไม่ตั้ง resources เลย
# QoS: BestEffort
```

- OOM killer kill Pod นี้ก่อนเมื่อ memory pressure สูง
- ไม่แนะนำสำหรับ production

### ดู QoS Class

```bash
# ดู QoS class ของ Pod
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'

# ดู QoS ของทุก Pod ใน namespace
kubectl get pods -n production -o custom-columns=\
'NAME:.metadata.name,QOS:.status.qosClass,CPU_REQ:.spec.containers[0].resources.requests.cpu,MEM_REQ:.spec.containers[0].resources.requests.memory'
```

---

## LimitRange

**LimitRange** กำหนด default resource values และ constraints สำหรับ containers ใน namespace:

### ทำไมต้องใช้ LimitRange

```
ปัญหาที่ไม่มี LimitRange:
- Developer ลืมตั้ง resources → BestEffort QoS
- Developer ตั้ง request/limit สูงเกินไป
- Namespace ใช้ resources เกิน quota ได้

LimitRange แก้ปัญหา:
- ตั้ง default values ให้ container ที่ไม่ระบุ
- จำกัด min/max ที่ container ขอได้
```

### LimitRange YAML

```yaml
# namespace-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  # Limits สำหรับ Container
  - type: Container
    # Default values ถ้าไม่ระบุ
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    # Min/Max ที่อนุญาต
    min:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    # Max ratio ระหว่าง limit และ request
    maxLimitRequestRatio:
      cpu: "4"       # limit สูงสุดได้ 4x request
      memory: "2"    # limit สูงสุดได้ 2x request
  
  # Limits สำหรับ Pod (ผลรวมของทุก container)
  - type: Pod
    max:
      cpu: "8"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
  
  # Limits สำหรับ PersistentVolumeClaim
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

### ทดสอบ LimitRange

```bash
# Apply LimitRange
kubectl apply -f namespace-limitrange.yaml

# สร้าง Pod โดยไม่ตั้ง resources (ใช้ default)
kubectl run test-pod --image=nginx:1.21 -n production

# ดู resources ที่ถูก inject
kubectl get pod test-pod -n production -o yaml | grep -A 10 resources

# ลอง create Pod ที่เกิน max
kubectl run big-pod --image=nginx:1.21 \
  -n production \
  --limits='cpu=4,memory=10Gi'
# Error: LimitRange "default-limits" violated

# ดู LimitRange
kubectl describe limitrange default-limits -n production
```

### LimitRange ตัวอย่าง สำหรับ Development Namespace

```yaml
# dev-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: "100m"       # Development: ใช้ resource น้อย
      memory: "128Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "500m"       # จำกัด max ในการพัฒนา
      memory: "512Mi"
  - type: PersistentVolumeClaim
    max:
      storage: "5Gi"    # จำกัด storage ใน dev
```

---

## ResourceQuota

**ResourceQuota** จำกัด resources รวมสำหรับทั้ง namespace:

### ResourceQuota YAML

```yaml
# namespace-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "20"          # รวม CPU requests ทั้ง namespace ไม่เกิน 20 cores
    requests.memory: "40Gi"     # รวม memory requests ไม่เกิน 40Gi
    limits.cpu: "40"            # รวม CPU limits ไม่เกิน 40 cores
    limits.memory: "80Gi"       # รวม memory limits ไม่เกิน 80Gi
    
    # Object count
    pods: "100"                  # จำนวน Pods ไม่เกิน 100
    services: "20"              # จำนวน Services ไม่เกิน 20
    configmaps: "50"            # จำนวน ConfigMaps ไม่เกิน 50
    secrets: "50"               # จำนวน Secrets ไม่เกิน 50
    persistentvolumeclaims: "20" # จำนวน PVCs ไม่เกิน 20
    
    # Storage
    requests.storage: "500Gi"   # รวม storage requests ไม่เกิน 500Gi
    
    # Count by QoS class
    count/pods.spec.priorityClassName.system-cluster-critical: "5"
```

### ResourceQuota ตาม QoS Class

```yaml
# qos-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guaranteed-quota
  namespace: production
spec:
  # กำหนด quota เฉพาะสำหรับ Guaranteed QoS Pods
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values:
      - guaranteed
  hard:
    pods: "20"
    requests.cpu: "10"
    requests.memory: "20Gi"
```

### ResourceQuota ตาม Priority Class

```yaml
# priority-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - high-priority
  hard:
    pods: "10"
    requests.cpu: "5"
    requests.memory: "10Gi"
```

### ดู ResourceQuota Status

```bash
# ดู Quota status
kubectl describe resourcequota production-quota -n production

# Output ตัวอย่าง:
# Name:            production-quota
# Namespace:       production
# Resource         Used    Hard
# --------         ----    ----
# limits.cpu       4       40
# limits.memory    8Gi     80Gi
# pods             12      100
# requests.cpu     2       20
# requests.memory  4Gi     40Gi

# ดูแบบ table
kubectl get resourcequota -n production

# ดู ResourceQuota ทั้งหมด
kubectl get resourcequota -A
```

---

## Workshop: กำหนด Resources อย่างเหมาะสม

### 1. วิเคราะห์ Resource Usage ปัจจุบัน

```bash
# ดู resource usage ของ pods ทั้งหมด
kubectl top pods -A --sort-by=cpu | head -20
kubectl top pods -A --sort-by=memory | head -20

# ดู node resource usage
kubectl top nodes

# ดู resource requests vs actual usage
kubectl get pods -n production -o yaml | \
  python3 -c "
import yaml, sys

data = yaml.safe_load(sys.stdin.read())
print(f'{'POD':<30} {'CONTAINER':<20} {'CPU_REQ':<10} {'MEM_REQ':<10}')
print('-' * 75)
for item in data.get('items', []):
    for container in item.get('spec', {}).get('containers', []):
        resources = container.get('resources', {})
        requests = resources.get('requests', {})
        name = item['metadata']['name']
        cname = container['name']
        cpu = requests.get('cpu', 'none')
        mem = requests.get('memory', 'none')
        print(f'{name:<30} {cname:<20} {cpu:<10} {mem:<10}')
"
```

### 2. สร้าง Namespace สำหรับ Workshop

```bash
kubectl create namespace resource-demo
```

### 3. สร้าง LimitRange

```yaml
# resource-demo-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-demo-limits
  namespace: resource-demo
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    min:
      cpu: "50m"
      memory: "32Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    maxLimitRequestRatio:
      cpu: "4"
      memory: "2"
```

### 4. สร้าง ResourceQuota

```yaml
# resource-demo-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-demo-quota
  namespace: resource-demo
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    pods: "20"
    services: "10"
    persistentvolumeclaims: "10"
    requests.storage: "20Gi"
```

### 5. Deploy Applications ด้วย Resource Configurations

```yaml
# well-configured-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: resource-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        
        # Well-configured resources
        resources:
          requests:
            cpu: "100m"     # 0.1 CPU guaranteed
            memory: "64Mi"  # 64Mi guaranteed
          limits:
            cpu: "300m"     # สูงสุด 0.3 CPU
            memory: "128Mi" # สูงสุด 128Mi
        
        # Readiness probe สำคัญ
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
---
# Database with Guaranteed QoS
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: resource-demo
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.0
        
        # Guaranteed QoS (requests = limits)
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "200m"     # = requests → Guaranteed
            memory: "256Mi" # = requests → Guaranteed
        
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5
```

### 6. ทดสอบ Resource Enforcement

```bash
# Apply ทั้งหมด
kubectl apply -f resource-demo-limitrange.yaml
kubectl apply -f resource-demo-quota.yaml
kubectl apply -f well-configured-app.yaml

# ดู LimitRange
kubectl describe limitrange resource-demo-limits -n resource-demo

# ดู Quota usage
kubectl describe resourcequota resource-demo-quota -n resource-demo

# ลอง deploy Pod ที่เกิน quota
kubectl run excess-pod \
  --image=nginx:1.21 \
  -n resource-demo \
  --requests='cpu=10,memory=10Gi'
# Error: exceeded quota

# ลอง deploy Pod ที่ไม่มี resources (ใช้ LimitRange default)
kubectl run no-resources-pod \
  --image=nginx:1.21 \
  -n resource-demo

# ดู resources ที่ inject โดย LimitRange
kubectl get pod no-resources-pod -n resource-demo -o yaml | grep -A 10 resources

# ดู QoS class
kubectl get pod no-resources-pod -n resource-demo -o jsonpath='{.status.qosClass}'
```

---

## Workshop: Namespace Resource Management

### สร้าง Multi-tenant Resource Policy

```bash
# สร้าง namespaces สำหรับทีมต่างๆ
kubectl create namespace team-frontend
kubectl create namespace team-backend
kubectl create namespace team-data
```

```yaml
# team-frontend-resources.yaml
# LimitRange สำหรับ frontend team
apiVersion: v1
kind: LimitRange
metadata:
  name: frontend-limits
  namespace: team-frontend
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "1"
      memory: "1Gi"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: frontend-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "50"
---
# LimitRange สำหรับ backend team
apiVersion: v1
kind: LimitRange
metadata:
  name: backend-limits
  namespace: team-backend
spec:
  limits:
  - type: Container
    default:
      cpu: "300m"
      memory: "512Mi"
    defaultRequest:
      cpu: "150m"
      memory: "256Mi"
    max:
      cpu: "2"
      memory: "4Gi"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: team-backend
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
---
# LimitRange สำหรับ data team
apiVersion: v1
kind: LimitRange
metadata:
  name: data-limits
  namespace: team-data
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "1Gi"
    defaultRequest:
      cpu: "250m"
      memory: "512Mi"
    max:
      cpu: "4"
      memory: "8Gi"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: data-quota
  namespace: team-data
spec:
  hard:
    requests.cpu: "40"
    requests.memory: "80Gi"
    limits.cpu: "80"
    limits.memory: "160Gi"
    pods: "50"
    persistentvolumeclaims: "20"
    requests.storage: "1Ti"
```

### Script ตรวจสอบ Resource Usage ทุก Namespace

```bash
#!/bin/bash
# check-resource-usage.sh

echo "=== Resource Usage by Namespace ==="
echo ""
printf "%-25s %-15s %-15s %-15s %-15s\n" "NAMESPACE" "CPU_REQ" "CPU_LIMIT" "MEM_REQ" "MEM_LIMIT"
printf "%-25s %-15s %-15s %-15s %-15s\n" "---------" "-------" "---------" "-------" "---------"

for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  quota=$(kubectl get resourcequota -n $ns -o json 2>/dev/null)
  if [ ! -z "$quota" ]; then
    cpu_req=$(echo $quota | python3 -c "import json,sys; d=json.load(sys.stdin); items=d.get('items',[]); print(items[0]['status']['used'].get('requests.cpu','0') if items else '0')" 2>/dev/null)
    cpu_lim=$(echo $quota | python3 -c "import json,sys; d=json.load(sys.stdin); items=d.get('items',[]); print(items[0]['status']['used'].get('limits.cpu','0') if items else '0')" 2>/dev/null)
    mem_req=$(echo $quota | python3 -c "import json,sys; d=json.load(sys.stdin); items=d.get('items',[]); print(items[0]['status']['used'].get('requests.memory','0') if items else '0')" 2>/dev/null)
    mem_lim=$(echo $quota | python3 -c "import json,sys; d=json.load(sys.stdin); items=d.get('items',[]); print(items[0]['status']['used'].get('limits.memory','0') if items else '0')" 2>/dev/null)
    printf "%-25s %-15s %-15s %-15s %-15s\n" "$ns" "$cpu_req" "$cpu_lim" "$mem_req" "$mem_lim"
  fi
done
```

---

## Troubleshooting Resource Issues

### ปัญหา: Pod OOMKilled

```bash
# ดู Pod status
kubectl get pod my-pod -n production
# STATUS: OOMKilled

# ดู exit code (137 = OOMKilled)
kubectl describe pod my-pod -n production | grep -A 10 "Last State"

# ดู memory limit ที่ตั้งไว้
kubectl get pod my-pod -n production -o yaml | grep -A 5 limits

# ดู actual memory usage ก่อน kill (ถ้ายังอยู่)
kubectl top pod my-pod -n production --containers

# แก้ไข: เพิ่ม memory limit
kubectl patch deployment my-app -n production --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"512Mi"}]'
```

### ปัญหา: Pod Pending (Insufficient Resources)

```bash
# ดู events ของ Pod
kubectl describe pod my-pod -n production | grep -A 10 Events
# "0/3 nodes are available: insufficient cpu."

# ดู node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# ดู pod ที่ใช้ resource มากที่สุด
kubectl top pods -A --sort-by=cpu | head -10

# ดู total requests per node
kubectl describe nodes | grep -E "Name:|requests"
```

### ปัญหา: CPU Throttling

```bash
# ดู throttling metrics (ถ้ามี Prometheus)
# rate(container_cpu_cfs_throttled_seconds_total[5m]) / rate(container_cpu_cfs_periods_total[5m])

# ดู container CPU usage vs limit
kubectl top pod my-pod -n production --containers

# ตรวจสอบใน container
kubectl exec my-pod -n production -- cat /sys/fs/cgroup/cpu/cpu.stat
# nr_throttled: จำนวนครั้งที่ถูก throttle
# throttled_time: เวลารวมที่ถูก throttle (nanoseconds)

# แก้ไข: เพิ่ม CPU limit
kubectl patch deployment my-app -n production --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/cpu","value":"500m"}]'
```

### ปัญหา: ResourceQuota Exceeded

```bash
# ดู error เมื่อ deploy
# Error: exceeded quota: production-quota, requested: pods=1, used: pods=100, limited: pods=100

# ดู quota usage
kubectl describe resourcequota -n production

# ดูว่า pod ไหนใช้มาก
kubectl top pods -n production --sort-by=cpu

# ลด pods ที่ไม่จำเป็น
kubectl scale deployment old-app --replicas=0 -n production

# หรือขอเพิ่ม quota (ต้องได้รับอนุมัติ)
kubectl edit resourcequota production-quota -n production
```

---

## Best Practices

### 1. Always Set Requests

```yaml
# ไม่ดี
containers:
- name: app
  image: nginx
  # ไม่มี resources → BestEffort QoS, อาจถูก kill ก่อน

# ดี
containers:
- name: app
  image: nginx
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "300m"
      memory: "256Mi"
```

### 2. กำหนด Guaranteed QoS สำหรับ Critical Services

```yaml
# Database, Critical APIs
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "1"       # = requests
    memory: "2Gi"  # = requests
# QoS: Guaranteed - ไม่ถูก kill ก่อน
```

### 3. ใช้ LimitRange เป็น Safety Net

```yaml
# ตั้ง default และ max สำหรับทุก namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: safety-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "200m"
      memory: "256Mi"
    max:
      cpu: "4"
      memory: "4Gi"
```

### 4. Monitor และ Adjust ตาม Actual Usage

```bash
# ดู actual vs requested (ควรตรวจสอบทุกสัปดาห์)
kubectl top pods -A | awk '{
  if (NR > 1) {
    split($3, cpu_arr, "m")
    split($4, mem_arr, "Mi")
    print $1, $2, cpu_arr[1] "m", mem_arr[1] "Mi"
  }
}'

# Goldilocks tool ช่วย recommend
# https://github.com/FairwindsOps/goldilocks
kubectl apply -f https://github.com/FairwindsOps/goldilocks/releases/latest/download/install.yaml
kubectl label namespace production goldilocks.fairwinds.com/enabled=true
```

### 5. Resource Ratio Guidelines

```
CPU:
  Request: actual_usage × 1.5   (buffer 50%)
  Limit:   request × 2-4        (burst headroom)

Memory:
  Request: actual_usage × 1.5
  Limit:   request × 1.5-2      (ไม่ควรมาก เพราะ OOMKill)

สำหรับ Java applications:
  - JVM Heap อาจใช้ memory มาก
  - ตั้ง -Xms/-Xmx ให้ตรงกับ memory limit
  - ตั้ง limit สูงกว่า JVM heap ประมาณ 20-30%
```

---

## สรุป

Resource Management ใน Kubernetes:

| Component | ควบคุมอะไร |
|-----------|-----------|
| Requests | รับประกันว่า container ได้ resources ตามนี้ |
| Limits | สูงสุดที่ container ใช้ได้ |
| QoS Guaranteed | requests=limits → ปลอดภัยที่สุด |
| QoS Burstable | มี requests/limits → ปกติ |
| QoS BestEffort | ไม่มีเลย → ถูก kill ก่อน |
| LimitRange | default + constraints ต่อ container |
| ResourceQuota | total usage constraints ต่อ namespace |

ในบทต่อไป เราจะเรียนรู้เรื่อง **Pod Disruption Budget** - การป้องกัน downtime ขณะทำ maintenance
