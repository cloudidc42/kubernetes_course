# Part 13: ReplicaSet - การจัดการ Pod Replicas

## สารบัญ
1. [ReplicaSet คืออะไร](#replicaset-คืออะไร)
2. [ReplicaSet YAML](#replicaset-yaml)
3. [Selector และ Labels](#selector-และ-labels)
4. [ความแตกต่างจาก ReplicationController](#ความแตกต่างจาก-replicationcontroller)
5. [Workshop: สร้าง ReplicaSet และทดสอบ Self-healing](#workshop)

---

## 1. ReplicaSet คืออะไร

**ReplicaSet** คือ Kubernetes Controller ที่ทำหน้าที่รับประกันว่าจะมี Pod ที่ตรงกับ selector จำนวนหนึ่งรันอยู่เสมอ (desired state)

### ปัญหาที่ ReplicaSet แก้ไข

ลองนึกภาพว่าคุณ deploy application โดยสร้าง Pod โดยตรง:

```
ปัญหาที่ 1: Pod ล้มเหลว
┌──────────┐
│  Pod A   │  ← crash! → Pod A หายไป
│ (nginx)  │    ไม่มีใครสร้างใหม่!
└──────────┘
Application down!

ปัญหาที่ 2: ต้องการ Scaling
ต้องสร้าง Pod B, C, D ด้วยมือ...
ไม่มีการจัดการอัตโนมัติ!
```

ReplicaSet แก้ปัญหาเหล่านี้:

```
ReplicaSet (desired: 3 replicas)
┌─────────────────────────────────────────┐
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │  Pod A   │  │  Pod B   │  │ Pod C  ││
│  │(running) │  │(running) │  │(running││
│  └──────────┘  └──────────┘  └────────┘│
│                                         │
│  ReplicaSet Controller Monitor ตลอดเวลา │
└─────────────────────────────────────────┘

Pod A crash! → ReplicaSet สร้าง Pod D ทันที!
Pod A   Pod D ใหม่
crash!  ← ReplicaSet สร้าง
```

### วิธีที่ ReplicaSet ทำงาน

ReplicaSet Controller ทำงานใน **Reconciliation Loop**:

```
1. ดู current state: มี Pods ที่ match selector กี่ตัว?
2. เปรียบเทียบกับ desired state: ต้องการ 3 replicas
3. ถ้า current < desired: สร้าง Pods เพิ่ม
4. ถ้า current > desired: ลบ Pods ออก
5. วนซ้ำตลอดเวลา
```

### ReplicaSet vs Pod โดยตรง

| | Pod โดยตรง | ReplicaSet |
|---|---|---|
| Self-healing | ✗ | ✓ |
| Scaling | ด้วยมือ | อัตโนมัติ |
| Pod replacement | ✗ | ✓ |
| Rolling updates | ✗ | ✗ (ใช้ Deployment แทน) |
| แนะนำสำหรับ production | ✗ | (ใช้ Deployment แทน) |

> **หมายเหตุ**: ในทางปฏิบัติ ควรใช้ **Deployment** แทน ReplicaSet โดยตรง เพราะ Deployment จัดการ ReplicaSet ให้อัตโนมัติและมีฟีเจอร์เพิ่มเติม (rolling updates, rollbacks)

---

## 2. ReplicaSet YAML

### โครงสร้าง ReplicaSet YAML

```yaml
apiVersion: apps/v1     # API group และ version
kind: ReplicaSet        # ประเภท Resource
metadata:
  name: my-replicaset   # ชื่อ ReplicaSet
  namespace: default
  labels:
    app: my-app         # Labels บน ReplicaSet object
spec:
  replicas: 3           # จำนวน Pods ที่ต้องการ (desired state)
  selector:             # กำหนดว่า ReplicaSet จะ manage Pods ไหน
    matchLabels:
      app: my-app       # ต้องตรงกับ template.metadata.labels
  template:             # Pod template - ใช้สร้าง Pods ใหม่
    metadata:
      labels:
        app: my-app     # ต้องตรงกับ selector.matchLabels
    spec:               # Pod spec
      containers:
      - name: nginx
        image: nginx:1.25
```

### ReplicaSet YAML ที่สมบูรณ์

```yaml
# full-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  namespace: default
  labels:
    app: nginx
    version: "1.25"
  annotations:
    description: "ReplicaSet for nginx web server"
spec:
  replicas: 3
  
  # minReadySeconds - รอกี่วินาทีหลัง Pod ready ก่อนถือว่า available
  minReadySeconds: 10
  
  selector:
    matchLabels:
      app: nginx
      tier: frontend
  
  template:
    metadata:
      labels:
        app: nginx           # ต้อง match selector.matchLabels
        tier: frontend       # ต้อง match selector.matchLabels
        version: "1.25"      # เพิ่ม labels เพิ่มเติมได้
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: http
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        
        env:
        - name: NGINX_HOST
          value: "localhost"
        - name: NGINX_PORT
          value: "80"
        
        volumeMounts:
        - name: nginx-logs
          mountPath: /var/log/nginx
      
      volumes:
      - name: nginx-logs
        emptyDir: {}
      
      terminationGracePeriodSeconds: 30
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - nginx
              topologyKey: kubernetes.io/hostname
```

### ทำความเข้าใจ Pod Template

Pod template ใน ReplicaSet คือ "แม่แบบ" ที่ใช้สร้าง Pods ใหม่:

```yaml
spec:
  template:           # นี่คือ Pod template
    metadata:
      labels:         # Labels ที่ Pods ที่สร้างจะมี
        app: nginx
    spec:             # Pod spec ปกติ
      containers:
      - name: nginx
        image: nginx:1.25
```

**ข้อสำคัญ**: เมื่อแก้ไข Pod template มันจะ**ไม่**ทำให้ Pods ที่มีอยู่เปลี่ยนแปลง - Pods ใหม่ที่สร้างหลังจากนั้นเท่านั้นที่จะใช้ template ใหม่

---

## 3. Selector และ Labels

### ความสัมพันธ์ระหว่าง Selector และ Labels

ReplicaSet ใช้ **selector** เพื่อ "identify" ว่า Pods ไหนที่มันต้องจัดการ:

```
ReplicaSet                              Pods
┌─────────────────────┐                ┌──────────────────┐
│ selector:           │                │ Pod A            │
│   matchLabels:      │    ค้นหา       │ labels:          │
│     app: nginx  ────┼──────────────► │   app: nginx     │ ✓ match
│     tier: web   ────┼──────────────► │   tier: web      │
└─────────────────────┘                └──────────────────┘
                                       ┌──────────────────┐
                                       │ Pod B            │
                                       │ labels:          │
                                       │   app: nginx     │ ✓ match
                                       │   tier: web      │
                                       └──────────────────┘
                                       ┌──────────────────┐
                                       │ Pod C            │
                                       │ labels:          │
                                       │   app: redis     │ ✗ no match
                                       └──────────────────┘
```

### Types of Selectors

#### 1. matchLabels (Equality-based)

```yaml
selector:
  matchLabels:
    app: nginx         # app == nginx
    environment: prod  # AND environment == prod
    tier: frontend     # AND tier == frontend
```

#### 2. matchExpressions (Set-based)

```yaml
selector:
  matchExpressions:
  - key: app
    operator: In        # app ต้องอยู่ใน list นี้
    values:
    - nginx
    - apache
  - key: environment
    operator: NotIn     # environment ต้องไม่อยู่ใน list นี้
    values:
    - dev
    - test
  - key: tier
    operator: Exists    # ต้องมี key นี้ (ไม่สนใจค่า)
  - key: deprecated
    operator: DoesNotExist  # ต้องไม่มี key นี้
```

#### 3. Combining matchLabels and matchExpressions

```yaml
selector:
  matchLabels:
    app: nginx
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
    - staging
```

### ระวัง: Label Selector ที่ไม่ถูกต้อง

```yaml
# ❌ อันตราย: selector ไม่ match template labels
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: apache    # ไม่ match! จะได้ error
```

```yaml
# ✓ ถูกต้อง: selector เป็น subset ของ template labels
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx       # match selector
        version: "1.25"  # label เพิ่มเติมได้
        tier: frontend   # label เพิ่มเติมได้
```

---

## 4. ความแตกต่างจาก ReplicationController

### ReplicationController (เก่า) vs ReplicaSet (ใหม่)

**ReplicationController** เป็น predecessor ของ ReplicaSet ที่ถูก deprecate แล้ว

| Feature | ReplicationController | ReplicaSet |
|---------|----------------------|------------|
| API Version | v1 | apps/v1 |
| Label Selector | Equality-based เท่านั้น | Equality + Set-based |
| matchLabels | ✓ | ✓ |
| matchExpressions | ✗ | ✓ |
| ใช้งานปัจจุบัน | Deprecated | ใช้ผ่าน Deployment |

### ตัวอย่าง ReplicationController (สำหรับเปรียบเทียบ)

```yaml
# replication-controller.yaml (เก่า - ไม่แนะนำใช้)
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:          # แค่ equality-based
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```yaml
# replicaset.yaml (ใหม่ - แนะนำ)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:     # ใช้ matchLabels
      app: nginx
    matchExpressions:  # หรือ matchExpressions ที่ยืดหยุ่นกว่า
    - key: tier
      operator: In
      values:
      - frontend
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

---

## 5. Workshop: สร้าง ReplicaSet และทดสอบ Self-healing

### Workshop Setup

```bash
# สร้าง namespace
kubectl create namespace rs-workshop

# ตั้ง default namespace
kubectl config set-context --current --namespace=rs-workshop
```

### Lab 1: สร้างและสังเกต ReplicaSet

```bash
# Step 1: สร้าง ReplicaSet manifest
cat <<'EOF' > /tmp/nginx-rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  namespace: rs-workshop
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: "1.25"
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
EOF

# Step 2: Apply ReplicaSet
kubectl apply -f /tmp/nginx-rs.yaml

# Step 3: ดูสถานะ ReplicaSet
kubectl get replicaset nginx-rs
kubectl get rs nginx-rs           # rs = short name สำหรับ replicaset

# Output:
# NAME       DESIRED   CURRENT   READY   AGE
# nginx-rs   3         3         3       30s

# Step 4: ดู Pods ที่ถูกสร้าง
kubectl get pods -l app=nginx
kubectl get pods --show-labels

# Output: จะเห็น Pods ชื่อ nginx-rs-xxxxx 3 ตัว

# Step 5: ดูรายละเอียด
kubectl describe rs nginx-rs
```

### Lab 2: ทดสอบ Self-healing

```bash
# Step 1: ดู Pods ปัจจุบัน
kubectl get pods -l app=nginx -w &
WATCH_PID=$!

# Step 2: ลบ Pod หนึ่งตัว
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
echo "Deleting pod: $POD_NAME"
kubectl delete pod $POD_NAME

# สังเกต: ReplicaSet จะสร้าง Pod ใหม่ทันที!
# จะเห็น Pod ใหม่ถูกสร้างมาแทน

sleep 15
kill $WATCH_PID 2>/dev/null

# Step 3: ยืนยันว่ายังมี 3 Pods
kubectl get pods -l app=nginx
# ยังคงมี 3 Pods!

# Step 4: ลบหลาย Pods พร้อมกัน
kubectl delete pods -l app=nginx
kubectl get pods -l app=nginx --watch &
WATCH_PID=$!
sleep 20
kill $WATCH_PID 2>/dev/null

# ReplicaSet จะสร้างใหม่ 3 ตัวอีกครั้ง
kubectl get pods -l app=nginx
```

### Lab 3: Scaling ReplicaSet

```bash
# Method 1: kubectl scale
kubectl scale rs nginx-rs --replicas=5

# ดูการเปลี่ยนแปลง
kubectl get pods -l app=nginx

# Scale down
kubectl scale rs nginx-rs --replicas=2
kubectl get pods -l app=nginx

# Method 2: แก้ไข YAML
kubectl edit rs nginx-rs
# เปลี่ยน replicas: 2 → replicas: 4
# บันทึกและออก

kubectl get rs nginx-rs
# DESIRED ควรเป็น 4

# Method 3: patch
kubectl patch rs nginx-rs -p '{"spec":{"replicas":3}}'
kubectl get rs nginx-rs
```

### Lab 4: เข้าใจ Label Selector

```bash
# สร้าง Pod ที่มี label ตรงกับ ReplicaSet selector
# ReplicaSet จะ "adopt" Pod นี้!
cat <<'EOF' > /tmp/orphan-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: orphan-pod
  namespace: rs-workshop
  labels:
    app: nginx           # ตรงกับ ReplicaSet selector!
    version: "manual"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
EOF

# ตรวจสอบก่อน: มี Pods เท่าไหร่
kubectl get pods -l app=nginx
# ควรมี 3 Pods

# สร้าง orphan pod
kubectl apply -f /tmp/orphan-pod.yaml

# ตรวจสอบอีกครั้ง
kubectl get pods -l app=nginx
# ตอนนี้มี 4 Pods (3 จาก RS + 1 orphan)
# ReplicaSet เห็นว่ามี 4 แต่ต้องการ 3
# จะลบ Pod หนึ่งตัว! (อาจเป็น orphan-pod ก็ได้)

sleep 5
kubectl get pods -l app=nginx
# กลับมาเป็น 3 Pods

# ลบ orphan pod manifest
kubectl delete -f /tmp/orphan-pod.yaml 2>/dev/null || true
```

### Lab 5: ทำ Rolling Update ด้วยมือ (ไม่ได้ใช้ Deployment)

```bash
# อธิบาย: ReplicaSet ไม่มี rolling update built-in
# ต้องทำด้วยมือ (จึงควรใช้ Deployment แทน)

# Step 1: Scale down จาก 3 → 0
kubectl scale rs nginx-rs --replicas=0
kubectl get pods -l app=nginx
# ทุก Pod ถูกลบ

# Step 2: เปลี่ยน image
kubectl set image rs nginx-rs nginx=nginx:1.26
# หรือ
kubectl patch rs nginx-rs -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.26"}]}}}}'

# Step 3: Scale up อีกครั้ง
kubectl scale rs nginx-rs --replicas=3

# Step 4: ตรวจสอบ image
kubectl get pods -l app=nginx \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# นี่คือเหตุผลที่ควรใช้ Deployment แทน!
# Deployment ทำ rolling update อัตโนมัติ
```

### Lab 6: ReplicaSet กับ Service

```bash
# สร้าง Service ที่ชี้ไปยัง Pods ของ ReplicaSet
cat <<'EOF' > /tmp/nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: rs-workshop
spec:
  selector:
    app: nginx         # ตรงกับ Pod labels
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f /tmp/nginx-service.yaml

# ดู Service
kubectl get service nginx-service

# ดู Endpoints (IPs ของ Pods)
kubectl get endpoints nginx-service

# Port forward ผ่าน Service
kubectl port-forward service/nginx-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080
kill $PF_PID

# ทดสอบ load balancing: request หลายๆ ครั้ง จะกระจายไปยัง Pods ต่างกัน
kubectl delete -f /tmp/nginx-service.yaml
```

### Lab 7: HorizontalPodAutoscaler กับ ReplicaSet

```bash
# ตั้งค่า Autoscaling (ต้องมี metrics-server)
kubectl autoscale rs nginx-rs \
    --min=2 \
    --max=10 \
    --cpu-percent=70

# ดู HPA
kubectl get hpa
kubectl describe hpa nginx-rs

# HPA จะ scale replicas อัตโนมัติตาม CPU usage
# ลบ HPA
kubectl delete hpa nginx-rs
```

### Cleanup Workshop

```bash
# ลบทุกอย่าง
kubectl delete -f /tmp/nginx-rs.yaml 2>/dev/null || true
kubectl delete namespace rs-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/nginx-rs.yaml /tmp/orphan-pod.yaml /tmp/nginx-service.yaml
```

### สรุป ReplicaSet Commands

```bash
# ดู ReplicaSets
kubectl get replicasets
kubectl get rs
kubectl get rs -A   # ทุก namespaces

# ดูรายละเอียด
kubectl describe rs nginx-rs

# Scaling
kubectl scale rs nginx-rs --replicas=5

# ดู Pods ที่ managed โดย ReplicaSet
kubectl get pods -l <selector-labels>

# ดู Owner References ของ Pod (ว่า ReplicaSet ไหนเป็นเจ้าของ)
kubectl get pod <pod-name> -o jsonpath='{.metadata.ownerReferences}'

# ลบ ReplicaSet (Pods ถูกลบด้วย)
kubectl delete rs nginx-rs

# ลบ ReplicaSet แต่เก็บ Pods ไว้ (orphan)
kubectl delete rs nginx-rs --cascade=orphan
```

---

### Lab 8: ReplicaSet Debugging

```bash
# สร้าง namespace ใหม่สำหรับ debug lab
kubectl create namespace rs-debug

# สร้าง ReplicaSet ที่มีปัญหา (image ไม่มีอยู่)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: broken-rs
  namespace: rs-debug
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken
  template:
    metadata:
      labels:
        app: broken
    spec:
      containers:
      - name: broken-app
        image: nginx:this-tag-does-not-exist
        ports:
        - containerPort: 80
EOF

# ดูสถานะ ReplicaSet
kubectl get rs broken-rs -n rs-debug
# DESIRED: 3, CURRENT: 3, READY: 0

# ดูสถานะ Pods
kubectl get pods -n rs-debug
# จะเห็น ImagePullBackOff หรือ ErrImagePull

# Debug ด้วย describe
kubectl describe rs broken-rs -n rs-debug
kubectl describe pods -n rs-debug | grep -A 5 "Events:"

# แก้ไข image ที่ถูกต้อง
kubectl patch rs broken-rs -n rs-debug \
    -p '{"spec":{"template":{"spec":{"containers":[{"name":"broken-app","image":"nginx:latest"}]}}}}'

# ลบ Pods เก่าที่ใช้ image ผิด (RS จะสร้างใหม่ด้วย image ที่ถูก)
kubectl delete pods -n rs-debug --all
kubectl get pods -n rs-debug --watch &
WATCH_PID=$!
sleep 20
kill $WATCH_PID 2>/dev/null

# ตรวจสอบ
kubectl get rs broken-rs -n rs-debug
# DESIRED: 3, CURRENT: 3, READY: 3

# Cleanup
kubectl delete namespace rs-debug
```

### Lab 9: ReplicaSet Ownership

```bash
# ทำความเข้าใจ ownerReferences

# สร้าง ReplicaSet
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: ownership-rs
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ownership-demo
  template:
    metadata:
      labels:
        app: ownership-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

# รอ Pods พร้อม
kubectl wait --for=condition=Ready pods -l app=ownership-demo --timeout=60s

# ดู ownerReferences ของ Pod
POD_NAME=$(kubectl get pods -l app=ownership-demo -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD_NAME -o jsonpath='{.metadata.ownerReferences}' | python3 -m json.tool

# Output จะแสดง:
# [
#   {
#     "apiVersion": "apps/v1",
#     "blockOwnerDeletion": true,
#     "controller": true,
#     "kind": "ReplicaSet",
#     "name": "ownership-rs",
#     "uid": "..."
#   }
# ]

# ลบ ReplicaSet แต่เก็บ Pods (orphan)
kubectl delete rs ownership-rs --cascade=orphan

# Pods ยังอยู่ แต่ไม่มีเจ้าของแล้ว
kubectl get pods -l app=ownership-demo
kubectl get pod $POD_NAME -o jsonpath='{.metadata.ownerReferences}'
# Empty! Pods กลายเป็น orphan

# Cleanup
kubectl delete pods -l app=ownership-demo
```

### Lab 10: ReplicaSet กับ Pod Disruption Budget

```bash
# PodDisruptionBudget (PDB) ป้องกัน disruption มากเกินไป

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: stable-rs
  namespace: default
spec:
  replicas: 5
  selector:
    matchLabels:
      app: stable-app
  template:
    metadata:
      labels:
        app: stable-app
    spec:
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
---
# PDB: รับประกันว่าจะมีอย่างน้อย 3 Pods พร้อมเสมอ
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: stable-pdb
  namespace: default
spec:
  minAvailable: 3        # อย่างน้อย 3 Pods ต้อง available
  selector:
    matchLabels:
      app: stable-app
EOF

# ดู PDB
kubectl get pdb stable-pdb
kubectl describe pdb stable-pdb

# Output:
# Min available: 3
# Current healthy: 5
# Desired healthy: 3
# Total replicas: 5
# Disruptions allowed: 2

# ลบทุกอย่าง
kubectl delete rs stable-rs
kubectl delete pdb stable-pdb
```

### คำสั่ง ReplicaSet ที่ใช้บ่อย

```bash
# ดู ReplicaSets
kubectl get replicasets
kubectl get rs -A                              # ทุก namespaces
kubectl get rs -n my-namespace                 # เฉพาะ namespace
kubectl get rs -l app=nginx                    # filter ด้วย label

# รายละเอียด
kubectl describe rs my-rs
kubectl describe rs -l app=nginx               # ทุก RS ที่มี label

# Scale
kubectl scale rs my-rs --replicas=5
kubectl scale rs my-rs --replicas=0            # ลบ Pods ชั่วคราว

# YAML
kubectl get rs my-rs -o yaml                   # ดู YAML
kubectl edit rs my-rs                          # แก้ไข live

# ลบ
kubectl delete rs my-rs                        # ลบ RS + Pods
kubectl delete rs my-rs --cascade=orphan       # ลบ RS เก็บ Pods
kubectl delete rs -l app=nginx                 # ลบตาม label

# ดู Pods ที่ managed
kubectl get pods -l app=nginx                  # ดู Pods
kubectl get pods -o wide -l app=nginx          # ดู Pods พร้อม Node info

# ตรวจสอบ ownership
kubectl get pod <pod-name> -o jsonpath='{.metadata.ownerReferences[0].name}'
```

### ตัวอย่าง ReplicaSet ใน Real World

```yaml
# production-replicaset.yaml
# ตัวอย่าง ReplicaSet สำหรับ Production
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: api-server-rs
  namespace: production
  labels:
    app: api-server
    tier: backend
    env: production
  annotations:
    description: "ReplicaSet for API server - managed by Deployment"
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api-server
      tier: backend
  template:
    metadata:
      labels:
        app: api-server
        tier: backend
        version: "3.2.1"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api-server
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api-server
        image: mycompany/api-server:3.2.1
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: APP_ENV
          value: "production"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
```

---

## สรุป

ReplicaSet เป็น Controller สำคัญใน Kubernetes ที่:

1. **รับประกัน Availability**: รักษาจำนวน Pod replicas ตามที่กำหนด
2. **Self-healing**: สร้าง Pod ใหม่โดยอัตโนมัติเมื่อ Pod ล้มเหลว
3. **Label-based Selection**: ใช้ labels เพื่อ identify Pods ที่จัดการ
4. **Flexible Selectors**: รองรับทั้ง equality-based และ set-based selectors
5. **Ownership**: Pod รู้ว่าใครเป็นเจ้าของผ่าน ownerReferences

**คำแนะนำ**: ในทางปฏิบัติ ให้ใช้ **Deployment** แทน ReplicaSet โดยตรง เพราะ:
- Deployment จัดการ ReplicaSet ให้อัตโนมัติ
- มี Rolling Update และ Rollback built-in
- ง่ายต่อการจัดการมากกว่า

ในบทต่อไปเราจะเรียนรู้ **Deployment** ซึ่งเป็น Higher-level Controller ที่ใช้งานจริงใน Production

---

## ReplicaSet Ownership และ Adoption

### ความหมายของ Ownership

ใน Kubernetes ทุก Resource สามารถมี "Owner" ได้ผ่าน `ownerReferences` ใน metadata เมื่อ ReplicaSet สร้าง Pod จะตั้ง ownerReference ให้ Pod นั้นชี้กลับมาที่ ReplicaSet

```yaml
# ตัวอย่าง ownerReference ที่ ReplicaSet ตั้งให้ Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-rs-abc12
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: my-rs
    uid: 9dbf0c6f-c0b9-4e8a-a1d2-5f12345abc67
    controller: true
    blockOwnerDeletion: true
```

### คุณสมบัติของ ownerReferences

| Field | ความหมาย |
|-------|----------|
| `controller: true` | Resource นี้ถูก manage โดย controller นี้ |
| `blockOwnerDeletion: true` | ห้ามลบ Owner จนกว่า Resource นี้จะถูกลบก่อน |
| `uid` | UID ของ Owner ใช้ตรวจสอบว่าเป็น Owner ตัวจริง |

### Garbage Collection

เมื่อลบ ReplicaSet ด้วย `--cascade=foreground` (default) Kubernetes จะ:
1. ตั้ง `deletionTimestamp` บน ReplicaSet
2. GarbageCollector ลบ Pods ที่ ownerReference ชี้มาที่ ReplicaSet นั้น
3. เมื่อ Pods ถูกลบหมดแล้ว จึงลบ ReplicaSet ออก

```bash
# ลบ ReplicaSet และ Pods ที่เกี่ยวข้อง (default)
kubectl delete replicaset my-rs

# ลบ ReplicaSet อย่างเดียว ไม่ลบ Pods (Orphan)
kubectl delete replicaset my-rs --cascade=orphan

# ดู Pods ที่ถูก Orphan (ไม่มี ownerReference อีกต่อไป)
kubectl get pods --show-labels
```

### Pod Adoption

ReplicaSet สามารถ "adopt" Pod ที่ไม่มี owner ได้ถ้า Pod นั้น match กับ selector ของ ReplicaSet

```yaml
# สถานการณ์: มี orphan Pod อยู่ก่อน
apiVersion: v1
kind: Pod
metadata:
  name: standalone-pod
  labels:
    app: my-app
    version: "1.0"
spec:
  containers:
  - name: app
    image: nginx:1.20
```

```yaml
# สร้าง ReplicaSet ที่ selector ตรงกัน
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: adopting-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0"
    spec:
      containers:
      - name: app
        image: nginx:1.20
```

เมื่อสร้าง `adopting-rs` ด้วย replicas=3 แต่มี orphan Pod ที่ match อยู่แล้ว 1 ตัว:
- ReplicaSet จะ adopt orphan Pod นั้น
- สร้าง Pod เพิ่มอีกเพียง 2 ตัว (ไม่ใช่ 3)

```bash
# ตรวจสอบว่า Pod ถูก adopt หรือไม่
kubectl get pod standalone-pod -o yaml | grep -A5 ownerReferences

# ผลลัพธ์จะแสดง ownerReference ชี้มาที่ adopting-rs
```

### การป้องกัน Unintended Adoption

```yaml
# ใช้ selector ที่เฉพาะเจาะจงเพื่อป้องกัน accidental adoption
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: specific-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      # เพิ่ม unique label เพื่อป้องกัน accidental match
      rs-managed: "specific-rs-v1"
  template:
    metadata:
      labels:
        app: my-app
        rs-managed: "specific-rs-v1"
    spec:
      containers:
      - name: app
        image: nginx:1.20
```

---

## ReplicaSet vs ReplicationController

### ประวัติ

- **ReplicationController**: เครื่องมือดั้งเดิมก่อน Kubernetes 1.2
- **ReplicaSet**: แนะนำใน Kubernetes 1.2 เพื่อแทน ReplicationController
- **ความแตกต่างหลัก**: ReplicaSet รองรับ set-based selectors ที่ทรงพลังกว่า

### เปรียบเทียบ YAML

```yaml
# ReplicationController (รูปแบบเก่า)
apiVersion: v1
kind: ReplicationController
metadata:
  name: old-style-rc
spec:
  replicas: 3
  # selector ใช้ map ธรรมดา - รองรับเฉพาะ equality-based
  selector:
    app: nginx
    version: "1.0"
  template:
    metadata:
      labels:
        app: nginx
        version: "1.0"
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

```yaml
# ReplicaSet (รูปแบบใหม่)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: new-style-rs
spec:
  replicas: 3
  # selector ใช้ matchLabels และ matchExpressions
  selector:
    matchLabels:
      app: nginx
    matchExpressions:
    - key: version
      operator: In
      values: ["1.0", "1.1", "1.2"]
    - key: environment
      operator: NotIn
      values: ["deprecated"]
  template:
    metadata:
      labels:
        app: nginx
        version: "1.0"
        environment: production
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

### ตารางเปรียบเทียบคุณสมบัติ

| คุณสมบัติ | ReplicationController | ReplicaSet |
|-----------|----------------------|------------|
| Equality-based selector | รองรับ | รองรับ |
| Set-based selector | ไม่รองรับ | รองรับ |
| ใช้โดย Deployment | ไม่ | ใช่ |
| API Group | core (v1) | apps/v1 |
| Status ใน Kubernetes | Deprecated | Active |
| แนะนำให้ใช้ | ไม่ | ใช่ (ผ่าน Deployment) |

### คำสั่ง Migration

```bash
# ตรวจสอบ ReplicationControllers ที่มีอยู่
kubectl get replicationcontrollers --all-namespaces

# แสดงรายละเอียด ReplicationController
kubectl describe rc old-style-rc

# แปลง ReplicationController เป็น ReplicaSet
# ขั้นตอน:
# 1. Backup manifest เดิม
kubectl get rc old-style-rc -o yaml > rc-backup.yaml

# 2. Scale RC เป็น 0
kubectl scale rc old-style-rc --replicas=0

# 3. สร้าง ReplicaSet ใหม่
kubectl apply -f new-replicaset.yaml

# 4. ตรวจสอบว่า RS ทำงานถูกต้อง
kubectl get rs new-style-rs

# 5. ลบ RC เก่า
kubectl delete rc old-style-rc
```

---

## Debugging ReplicaSet ที่ไม่ Scale

### สถานการณ์ที่พบบ่อย

#### 1. Pods ไม่ถูกสร้างเพิ่ม

```bash
# ตรวจสอบ ReplicaSet status
kubectl describe rs my-rs

# มองหาใน Events section:
# - FailedCreate: ไม่สามารถสร้าง Pod ได้
# - SuccessfulCreate: สร้าง Pod สำเร็จ
# - SuccessfulDelete: ลบ Pod สำเร็จ

# ดู Events ทั้งหมดใน namespace
kubectl get events --sort-by='.lastTimestamp'
```

#### 2. Pods อยู่ใน Pending State

```bash
# ดู Pending Pods
kubectl get pods | grep Pending

# ตรวจสอบสาเหตุ
kubectl describe pod <pending-pod-name>

# สาเหตุที่พบบ่อย:
# - Insufficient resources (CPU/Memory)
# - No nodes match nodeSelector/affinity
# - PVC ไม่สามารถ bind ได้
# - Taints/Tolerations ไม่ตรงกัน
```

#### 3. Pods ถูกสร้างแล้วล้มเหลว

```bash
# ดู Pods ที่ CrashLoopBackOff
kubectl get pods | grep -E "Error|CrashLoop|OOMKilled"

# ดู logs ของ Pod ที่ fail
kubectl logs <pod-name> --previous

# ดู container exit code
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

#### 4. ReplicaSet ไม่ match กับ Pods ที่มีอยู่

```bash
# ตรวจสอบ labels ของ Pods
kubectl get pods --show-labels

# ตรวจสอบ selector ของ ReplicaSet
kubectl get rs my-rs -o jsonpath='{.spec.selector}'

# เปรียบเทียบ labels กับ selector
kubectl get rs my-rs -o yaml | grep -A10 selector
```

### Script Debugging อัตโนมัติ

```bash
#!/bin/bash
# debug-replicaset.sh - ตรวจสอบ ReplicaSet แบบอัตโนมัติ

RS_NAME="${1:-my-rs}"
NAMESPACE="${2:-default}"

echo "=== ReplicaSet Debug Report: $RS_NAME ==="
echo ""

# 1. ตรวจสอบ RS status
echo "--- ReplicaSet Status ---"
kubectl get rs "$RS_NAME" -n "$NAMESPACE"
echo ""

# 2. ตรวจสอบ desired vs current
DESIRED=$(kubectl get rs "$RS_NAME" -n "$NAMESPACE" \
    -o jsonpath='{.spec.replicas}')
READY=$(kubectl get rs "$RS_NAME" -n "$NAMESPACE" \
    -o jsonpath='{.status.readyReplicas}')
CURRENT=$(kubectl get rs "$RS_NAME" -n "$NAMESPACE" \
    -o jsonpath='{.status.replicas}')

echo "Desired: $DESIRED, Current: $CURRENT, Ready: $READY"

if [ "$DESIRED" != "$READY" ]; then
    echo "WARNING: Not all replicas are ready!"
fi
echo ""

# 3. ตรวจสอบ Pods
echo "--- Pods managed by $RS_NAME ---"
SELECTOR=$(kubectl get rs "$RS_NAME" -n "$NAMESPACE" \
    -o jsonpath='{.spec.selector.matchLabels}' | \
    python3 -c "import sys,json; d=json.load(sys.stdin); \
    print(','.join([f'{k}={v}' for k,v in d.items()]))")

kubectl get pods -n "$NAMESPACE" -l "$SELECTOR" -o wide
echo ""

# 4. ดู Events ที่เกี่ยวข้อง
echo "--- Recent Events ---"
kubectl get events -n "$NAMESPACE" \
    --field-selector "involvedObject.name=$RS_NAME" \
    --sort-by='.lastTimestamp' | tail -10
echo ""

# 5. ตรวจสอบ Node resources
echo "--- Node Resource Availability ---"
kubectl describe nodes | grep -A5 "Allocated resources"
```

### แก้ปัญหา Image Pull Error

```bash
# ตรวจสอบ image pull error
kubectl describe pod <pod-name> | grep -A5 "Events:"
# หากเห็น: Failed to pull image "myimage:tag"

# แก้ไข: ตรวจสอบ ImagePullSecrets
kubectl get rs my-rs -o yaml | grep imagePullSecrets

# เพิ่ม imagePullSecrets ถ้าจำเป็น
kubectl patch serviceaccount default \
    -p '{"imagePullSecrets": [{"name": "registry-secret"}]}'

# หรือแก้ไข ReplicaSet template
kubectl edit rs my-rs
# เพิ่มใน spec.template.spec:
# imagePullSecrets:
# - name: registry-secret
```

### แก้ปัญหา Resource Quota Exceeded

```bash
# ตรวจสอบ ResourceQuota ใน namespace
kubectl describe resourcequota -n my-namespace

# ดู Events เกี่ยวกับ quota
kubectl get events -n my-namespace | grep -i quota

# แก้ไข: เพิ่ม Quota
kubectl patch resourcequota my-quota -n my-namespace \
    --type=merge \
    -p '{"spec":{"hard":{"pods":"20","requests.cpu":"4"}}}'
```

---

## แบบฝึกหัด: ReplicaSets

### แบบฝึกหัดที่ 1: สร้างและตรวจสอบ ReplicaSet พื้นฐาน

**โจทย์**: สร้าง ReplicaSet ชื่อ `web-rs` ที่รัน Nginx 3 replicas

**ขั้นตอน**:
```bash
# 1. สร้างไฟล์ web-rs.yaml
cat > web-rs.yaml << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF

# 2. Apply
kubectl apply -f web-rs.yaml

# 3. ตรวจสอบ
kubectl get rs web-rs
kubectl get pods -l app=web
```

**เฉลย - การตรวจสอบ**:
```bash
# ผลลัพธ์ที่ถูกต้อง:
NAME     DESIRED   CURRENT   READY   AGE
web-rs   3         3         3       30s

# Pods ที่ควรเห็น:
NAME           READY   STATUS    RESTARTS   AGE
web-rs-abc12   1/1     Running   0          30s
web-rs-def34   1/1     Running   0          30s
web-rs-ghi56   1/1     Running   0          30s
```

---

### แบบฝึกหัดที่ 2: ทดสอบ Self-healing

**โจทย์**: ลบ Pod หนึ่งตัวจาก ReplicaSet และสังเกตว่า ReplicaSet สร้าง Pod ใหม่

```bash
# 1. ดู Pods ปัจจุบัน
kubectl get pods -l app=web

# 2. ลบ Pod หนึ่งตัว (ใช้ชื่อจากขั้นตอนที่ 1)
kubectl delete pod web-rs-abc12

# 3. สังเกตการสร้าง Pod ใหม่
kubectl get pods -l app=web -w
```

**เฉลย**:
```
# ขณะลบ Pod จะเห็น:
web-rs-abc12   1/1     Terminating   0          2m
web-rs-xyz99   0/1     ContainerCreating   0   1s

# หลังจากนั้น:
web-rs-def34   1/1     Running   0          2m
web-rs-ghi56   1/1     Running   0          2m
web-rs-xyz99   1/1     Running   0          15s
# (abc12 ถูกแทนที่ด้วย xyz99)
```

---

### แบบฝึกหัดที่ 3: Scale ReplicaSet

**โจทย์**: Scale `web-rs` จาก 3 เป็น 5 replicas ด้วย 3 วิธีต่างกัน

**เฉลย**:
```bash
# วิธีที่ 1: kubectl scale
kubectl scale rs web-rs --replicas=5

# วิธีที่ 2: kubectl patch
kubectl patch rs web-rs -p '{"spec":{"replicas":5}}'

# วิธีที่ 3: แก้ไขไฟล์ YAML แล้ว apply ใหม่
# แก้ไข replicas: 3 → replicas: 5 ใน web-rs.yaml
kubectl apply -f web-rs.yaml

# ตรวจสอบ
kubectl get rs web-rs
# ควรแสดง DESIRED=5, CURRENT=5, READY=5
```

---

### แบบฝึกหัดที่ 4: Set-based Selector

**โจทย์**: สร้าง ReplicaSet ที่ใช้ `matchExpressions` เพื่อจัดการ Pods ที่มี tier เป็น `frontend` หรือ `backend`

**เฉลย**:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: multi-tier-rs
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
    matchExpressions:
    - key: tier
      operator: In
      values: ["frontend", "backend"]
    - key: environment
      operator: NotIn
      values: ["deprecated", "test"]
  template:
    metadata:
      labels:
        app: myapp
        tier: frontend
        environment: production
    spec:
      containers:
      - name: app
        image: nginx:1.21
```

---

### แบบฝึกหัดที่ 5: Orphan Pods และ Adoption

**โจทย์**: ทำความเข้าใจ Pod Adoption โดย:
1. สร้าง standalone Pod ที่มี labels ตรงกัน
2. สร้าง ReplicaSet ที่ match กับ Pod นั้น
3. สังเกตว่า RS adopt Pod แทนที่จะสร้างใหม่

**เฉลย**:
```bash
# 1. สร้าง standalone Pod
cat > standalone.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: standalone-nginx
  labels:
    app: adopted-app
    env: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.21
EOF
kubectl apply -f standalone.yaml

# 2. รอให้ Pod running
kubectl wait --for=condition=ready pod/standalone-nginx

# 3. สร้าง ReplicaSet ที่ selector match
cat > adopting-rs.yaml << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: adopting-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: adopted-app
  template:
    metadata:
      labels:
        app: adopted-app
        env: dev
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
EOF
kubectl apply -f adopting-rs.yaml

# 4. สังเกตผลลัพธ์ - RS ควร create เพียง 2 Pods ใหม่ (ไม่ใช่ 3)
kubectl get pods -l app=adopted-app

# 5. ยืนยันว่า standalone-nginx ถูก adopt
kubectl get pod standalone-nginx -o yaml | grep -A5 ownerReferences
```

---

### แบบฝึกหัดที่ 6: ลบ ReplicaSet แบบ Orphan

**โจทย์**: ลบ ReplicaSet โดยไม่ลบ Pods ที่มันจัดการ

**เฉลย**:
```bash
# ลบ RS แบบ orphan (Pods จะไม่ถูกลบ)
kubectl delete rs web-rs --cascade=orphan

# ตรวจสอบว่า RS ถูกลบแล้ว
kubectl get rs

# ตรวจสอบว่า Pods ยังอยู่ (แต่ไม่มี owner แล้ว)
kubectl get pods -l app=web

# ดู ownerReferences ที่หายไป
kubectl get pod <pod-name> -o yaml | grep ownerReferences
# ควรเป็น ownerReferences: null หรือไม่มี field นี้
```

---

### แบบฝึกหัดที่ 7: HPA กับ ReplicaSet

**โจทย์**: ตั้ง HPA สำหรับ ReplicaSet

**เฉลย**:
```bash
# สร้าง ReplicaSet ก่อน
kubectl apply -f web-rs.yaml

# สร้าง HPA สำหรับ RS
kubectl autoscale rs web-rs \
    --min=2 \
    --max=10 \
    --cpu-percent=70

# ดู HPA
kubectl get hpa web-rs

# ดูรายละเอียด
kubectl describe hpa web-rs
```

```yaml
# หรือสร้าง HPA ด้วย YAML
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-rs-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: ReplicaSet
    name: web-rs
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

### แบบฝึกหัดที่ 8: Production-ready ReplicaSet

**โจทย์**: สร้าง ReplicaSet แบบ production-ready ที่มี resource limits, probes, และ security context

**เฉลย**:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: production-rs
  labels:
    app: api-server
    version: "2.0"
    environment: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
      environment: production
  template:
    metadata:
      labels:
        app: api-server
        version: "2.0"
        environment: production
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: api
        image: myregistry/api-server:2.0
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
        securityContext:
          allowPrivilegeEscalation: false
          runAsUser: 1000
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api-server
              topologyKey: kubernetes.io/hostname
```

---

### แบบฝึกหัดที่ 9: Debugging ReplicaSet ที่ไม่ Scale

**โจทย์**: ระบุและแก้ไขปัญหาทำไม ReplicaSet จึงไม่ scale

```bash
# สร้าง RS ที่มีปัญหา (image ไม่ถูกต้อง)
cat > broken-rs.yaml << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: broken-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken
  template:
    metadata:
      labels:
        app: broken
    spec:
      containers:
      - name: app
        image: nonexistent-registry/no-such-image:latest
EOF
kubectl apply -f broken-rs.yaml
```

**เฉลย - ขั้นตอน Debugging**:
```bash
# ขั้นที่ 1: ดู RS status
kubectl get rs broken-rs
# จะเห็น READY=0

# ขั้นที่ 2: ดู Pod status
kubectl get pods -l app=broken
# จะเห็น ImagePullBackOff หรือ ErrImagePull

# ขั้นที่ 3: ดูรายละเอียด Pod
kubectl describe pod <broken-pod-name>
# ดู Events section - จะเห็น Failed to pull image

# ขั้นที่ 4: แก้ไข image
kubectl set image rs/broken-rs app=nginx:1.21

# ขั้นที่ 5: ตรวจสอบ (RS ไม่ restart Pods อัตโนมัติ)
# ต้องลบ Pods เก่าด้วยตนเอง
kubectl delete pods -l app=broken

# ขั้นที่ 6: ยืนยันว่า RS scale ได้แล้ว
kubectl get pods -l app=broken
```

---

### แบบฝึกหัดที่ 10: เปรียบเทียบ ReplicaSet กับ Deployment

**โจทย์**: เข้าใจความแตกต่างระหว่าง ReplicaSet และ Deployment โดยทดลอง update image

**เฉลย**:

```bash
# สร้าง ReplicaSet
cat > compare-rs.yaml << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: compare-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: compare
  template:
    metadata:
      labels:
        app: compare
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
EOF
kubectl apply -f compare-rs.yaml

# พยายาม update image
kubectl set image rs/compare-rs nginx=nginx:1.21

# ตรวจสอบ: RS ไม่ restart Pods ที่มีอยู่!
kubectl get pods -l app=compare -o jsonpath='{.items[*].spec.containers[0].image}'
# จะยังเห็น nginx:1.20 ใน Pods ที่มีอยู่

# Pods ใหม่เท่านั้นที่จะใช้ image ใหม่
kubectl delete pod <one-pod-name>
kubectl get pods -l app=compare -o jsonpath='{.items[*].spec.containers[0].image}'
# Pod ใหม่จะใช้ nginx:1.21 แต่ Pods เก่ายังคง 1.20
```

```bash
# เปรียบเทียบกับ Deployment
cat > compare-deploy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compare-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: compare-deploy
  template:
    metadata:
      labels:
        app: compare-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
EOF
kubectl apply -f compare-deploy.yaml

# Update image ใน Deployment
kubectl set image deployment/compare-deploy nginx=nginx:1.21

# Deployment จะ Rolling Update ทุก Pod อัตโนมัติ!
kubectl rollout status deployment/compare-deploy
kubectl get pods -l app=compare-deploy \
    -o jsonpath='{.items[*].spec.containers[0].image}'
# จะเห็น nginx:1.21 ทุกตัว
```

**ข้อสรุป**: 
- ReplicaSet: ไม่ update Pods ที่มีอยู่เมื่อเปลี่ยน template
- Deployment: จัดการ Rolling Update ให้อัตโนมัติ ผ่านการสร้าง ReplicaSet ใหม่

---

## สรุปทบทวน ReplicaSets

### Cheat Sheet

```bash
# สร้าง/อัพเดต
kubectl apply -f replicaset.yaml

# ดูรายการ
kubectl get rs
kubectl get rs -n <namespace>

# ดูรายละเอียด
kubectl describe rs <name>

# Scale
kubectl scale rs <name> --replicas=<count>

# Edit
kubectl edit rs <name>

# ลบ (พร้อม Pods)
kubectl delete rs <name>

# ลบ (ไม่ลบ Pods)
kubectl delete rs <name> --cascade=orphan

# ดู Pods ของ RS
kubectl get pods -l <selector>
```

ReplicaSet เป็นรากฐานของ Deployment ที่ใช้งานจริงใน Production ความเข้าใจ ReplicaSet จะช่วยให้ Debug Deployment ได้ง่ายขึ้น
