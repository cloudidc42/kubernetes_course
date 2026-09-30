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
