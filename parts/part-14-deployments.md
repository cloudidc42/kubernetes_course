# Part 14: Deployments - จัดการ Application อย่างมืออาชีพ

## สารบัญ
1. [Deployment คืออะไร](#deployment-คืออะไร)
2. [Deployment YAML](#deployment-yaml)
3. [Rolling Update Strategy](#rolling-update-strategy)
4. [Rollback](#rollback)
5. [Scaling](#scaling)
6. [Workshop: Deploy Web App และทำ Rolling Update](#workshop)

---

## 1. Deployment คืออะไร

**Deployment** คือ Higher-level Controller ใน Kubernetes ที่จัดการ ReplicaSets และให้ความสามารถในการ:
- **Rolling Updates**: อัปเดต application โดยไม่มี downtime
- **Rollbacks**: ย้อนกลับไปยัง version ก่อนหน้าได้ทันที
- **Scaling**: ปรับขนาดได้ง่าย
- **Self-healing**: ผ่าน ReplicaSet ที่มันจัดการ

### Deployment Architecture

```
Deployment
└── ReplicaSet (v1) ──── Pod 1 (nginx:1.20)
    └── เก็บเป็น history    Pod 2 (nginx:1.20)
                           Pod 3 (nginx:1.20)

หลังจาก Rolling Update:

Deployment
├── ReplicaSet (v1) ──── (0 Pods - kept as history)
└── ReplicaSet (v2) ──── Pod 1 (nginx:1.25)
                         Pod 2 (nginx:1.25)
                         Pod 3 (nginx:1.25)
```

### ทำไมต้องใช้ Deployment?

```
ปัญหาของ ReplicaSet:
- ไม่มี Rolling Update อัตโนมัติ
- ไม่มี Rollback
- ต้อง manage ReplicaSet version เอง

Deployment แก้ปัญหาเหล่านี้:
✓ Rolling Update แบบ zero-downtime
✓ Rollback ในไม่กี่วินาที
✓ Deployment History
✓ Pause/Resume updates
✓ Scale ได้ง่าย
```

---

## 2. Deployment YAML

### โครงสร้าง Deployment YAML พื้นฐาน

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:               # Pod template
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Deployment YAML ที่สมบูรณ์

```yaml
# full-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  namespace: production
  labels:
    app: webapp
    version: "2.0"
    team: frontend
  annotations:
    kubernetes.io/change-cause: "Update to version 2.0 with performance improvements"
    deployment.kubernetes.io/revision: "1"
spec:
  # จำนวน Replicas ที่ต้องการ
  replicas: 5
  
  # ประวัติ ReplicaSet ที่เก็บไว้ (สำหรับ rollback)
  revisionHistoryLimit: 10
  
  # รอกี่วินาทีหลัง Pod available ก่อนถือว่า update สำเร็จ
  minReadySeconds: 10
  
  # timeout สำหรับ deployment progress
  progressDeadlineSeconds: 600
  
  # Label selector สำหรับ Pods ที่จะจัดการ
  selector:
    matchLabels:
      app: webapp
  
  # Update Strategy
  strategy:
    type: RollingUpdate           # หรือ Recreate
    rollingUpdate:
      maxUnavailable: 1           # Pods ที่ unavailable ได้สูงสุดในขณะ update
      maxSurge: 1                 # Pods เพิ่มเติมที่สร้างได้สูงสุดในขณะ update
  
  # Pod Template
  template:
    metadata:
      labels:
        app: webapp               # ต้อง match selector.matchLabels
        version: "2.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      # Graceful termination
      terminationGracePeriodSeconds: 60
      
      # Image Pull Policy
      containers:
      - name: webapp
        image: myapp/webapp:2.0
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        
        # Environment variables
        env:
        - name: APP_ENV
          value: "production"
        - name: APP_VERSION
          value: "2.0"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: db-password
        
        # Resources
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        
        # Health Probes
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        startupProbe:
          httpGet:
            path: /health/started
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        
        # Lifecycle
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        
        # Volume Mounts
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /app/logs
        
        # Security Context
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      
      # Volumes
      volumes:
      - name: config
        configMap:
          name: webapp-config
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}
      
      # Security Context สำหรับ Pod
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
      
      # Affinity
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
                  - webapp
              topologyKey: kubernetes.io/hostname
      
      # Service Account
      serviceAccountName: webapp-sa
      automountServiceAccountToken: false
      
      # Image Pull Secrets
      imagePullSecrets:
      - name: registry-secret
```

---

## 3. Rolling Update Strategy

### Rolling Update คืออะไร

Rolling Update คือ strategy ที่ทำการอัปเดต Pods ทีละน้อย ทำให้ application ยังพร้อมให้บริการตลอดเวลา

```
Rolling Update Process (replicas: 4, maxUnavailable: 1, maxSurge: 1):

Step 1: เริ่มต้น (v1)
Pod1(v1) Pod2(v1) Pod3(v1) Pod4(v1)

Step 2: สร้าง Pod ใหม่ (v2)
Pod1(v1) Pod2(v1) Pod3(v1) Pod4(v1) Pod5(v2)[new]
                                     maxSurge=1 (5 Pods total)

Step 3: Pod5(v2) พร้อม, ลบ Pod1(v1)
         Pod2(v1) Pod3(v1) Pod4(v1) Pod5(v2)
                                     still 4 Pods

Step 4: สร้าง Pod6(v2), ลบ Pod2(v1) เมื่อพร้อม
         Pod3(v1) Pod4(v1) Pod5(v2) Pod6(v2)

Step 5: วนซ้ำจนครบ
                          Pod5(v2) Pod6(v2) Pod7(v2) Pod8(v2)
```

### Strategy Types

#### 1. RollingUpdate (default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # จำนวนหรือเปอร์เซ็นต์ (เช่น "25%")
      maxSurge: 1         # จำนวนหรือเปอร์เซ็นต์ (เช่น "25%")
```

**ตัวอย่างการคำนวณ** (replicas: 4):
- `maxUnavailable: 1` → ต่ำสุดมี 3 Pods running ในขณะ update
- `maxSurge: 1` → สูงสุดมี 5 Pods running ในขณะ update
- `maxUnavailable: "25%"` → ต่ำสุดมี 3 Pods running (4 - 25% = 3)
- `maxSurge: "25%"` → สูงสุดมี 5 Pods running (4 + 25% = 5)

#### 2. Recreate

```yaml
spec:
  strategy:
    type: Recreate        # ลบ Pods เก่าทั้งหมดก่อน แล้วสร้างใหม่
    # ไม่มี rolling update config
```

ใช้เมื่อ:
- Application ไม่รองรับหลาย versions พร้อมกัน
- Database migration ที่ต้องการ exclusive access
- ยอมรับ downtime ชั่วคราวได้

### ตัวอย่าง Strategy ต่างๆ

```yaml
# Strategy 1: Conservative (ไม่ยอม downtime เลย)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0     # ไม่มี Pod unavailable เลย
    maxSurge: 1           # สร้างเพิ่มทีละ 1

# Strategy 2: Fast Update (ยอม downtime บางส่วน)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 2     # ลบได้สูงสุด 2 ตัวพร้อมกัน
    maxSurge: 2           # สร้างเพิ่มได้สูงสุด 2 ตัวพร้อมกัน

# Strategy 3: Canary-like (อัปเดตทีละน้อยมาก)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: "10%"       # เพิ่ม 10% ของ replicas ทีละครั้ง
```

---

## 4. Rollback

### Deployment History

ทุกครั้งที่ทำ update Kubernetes จะเก็บ history ไว้:

```bash
# ดู rollout history
kubectl rollout history deployment/webapp

# Output:
# REVISION  CHANGE-CAUSE
# 1         kubectl create --filename=webapp.yaml
# 2         Update nginx to 1.24
# 3         Update nginx to 1.25 (CURRENT)

# ดูรายละเอียดของแต่ละ revision
kubectl rollout history deployment/webapp --revision=2
```

### วิธีการ Rollback

```bash
# Rollback ไปยัง revision ก่อนหน้า
kubectl rollout undo deployment/webapp

# Rollback ไปยัง revision เฉพาะ
kubectl rollout undo deployment/webapp --to-revision=1

# ดูสถานะ rollback
kubectl rollout status deployment/webapp

# ยืนยัน version ที่ใช้งาน
kubectl get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### บันทึก Change Cause

```bash
# วิธี 1: ใช้ --record flag (deprecated ใน Kubernetes 1.22+)
kubectl set image deployment/webapp webapp=nginx:1.25 --record

# วิธี 2: ใช้ annotation (แนะนำ)
kubectl annotate deployment/webapp \
    kubernetes.io/change-cause="Update to nginx 1.25 for security patch"

# วิธี 3: เขียนใน YAML
metadata:
  annotations:
    kubernetes.io/change-cause: "Update to nginx 1.25 for security patch"
```

---

## 5. Scaling

### Manual Scaling

```bash
# Scale up
kubectl scale deployment webapp --replicas=10

# Scale down
kubectl scale deployment webapp --replicas=2

# ตรวจสอบ
kubectl get deployment webapp
kubectl get pods -l app=webapp
```

### Horizontal Pod Autoscaler (HPA)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp-deployment
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # scale เมื่อ CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80   # scale เมื่อ Memory > 80%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # รอ 60s ก่อน scale up
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # รอ 5 min ก่อน scale down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# ดู HPA status
kubectl get hpa
kubectl describe hpa webapp-hpa

# ดู scaling events
kubectl get events --field-selector reason=SuccessfulRescale
```

---

## 6. Workshop: Deploy Web App และทำ Rolling Update

### Workshop Setup

```bash
# สร้าง namespace
kubectl create namespace deployment-workshop

# ตั้ง default namespace
kubectl config set-context --current --namespace=deployment-workshop
```

### Lab 1: Deploy Web Application

```bash
# Step 1: สร้าง ConfigMap สำหรับ nginx
cat <<'EOF' > /tmp/workshop-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: deployment-workshop
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Kubernetes Workshop - Version 1.0</title>
      <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #f0f0f0; }
        .version { font-size: 3em; color: #4CAF50; }
        .info { background: white; padding: 20px; border-radius: 10px; margin: 20px auto; max-width: 600px; }
      </style>
    </head>
    <body>
      <h1>🚀 Kubernetes Deployment Workshop</h1>
      <div class="version">Version 1.0</div>
      <div class="info">
        <h2>Application Info</h2>
        <p>This is Version 1.0 of our web application</p>
        <p>Deployed with Kubernetes Deployment</p>
      </div>
    </body>
    </html>
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
        location /health {
          return 200 'OK';
          add_header Content-Type text/plain;
        }
      }
    }
EOF

# Step 2: สร้าง Deployment v1.0
cat <<'EOF' > /tmp/workshop-deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: deployment-workshop
  labels:
    app: webapp
    version: "1.0"
  annotations:
    kubernetes.io/change-cause: "Initial deployment - version 1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: webapp
        version: "1.0"
    spec:
      containers:
      - name: webapp
        image: nginx:1.24
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: config
        configMap:
          name: webapp-config
          items:
          - key: nginx.conf
            path: nginx.conf
      - name: html
        configMap:
          name: webapp-config
          items:
          - key: index.html
            path: index.html
EOF

# Step 3: สร้าง Service
cat <<'EOF' > /tmp/workshop-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: deployment-workshop
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    name: http
  type: ClusterIP
EOF

# Step 4: Apply ทั้งหมด
kubectl apply -f /tmp/workshop-configmap.yaml
kubectl apply -f /tmp/workshop-deployment-v1.yaml
kubectl apply -f /tmp/workshop-service.yaml

# Step 5: ดูสถานะ
kubectl get all -n deployment-workshop

# รอ Deployment พร้อม
kubectl rollout status deployment/webapp

# Step 6: ทดสอบ
kubectl port-forward service/webapp-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080
kill $PF_PID
```

### Lab 2: Rolling Update ไปยัง Version 2.0

```bash
# Step 1: สร้าง ConfigMap v2.0
cat <<'EOF' > /tmp/workshop-configmap-v2.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: deployment-workshop
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Kubernetes Workshop - Version 2.0</title>
      <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #e3f2fd; }
        .version { font-size: 3em; color: #2196F3; }
        .info { background: white; padding: 20px; border-radius: 10px; margin: 20px auto; max-width: 600px; }
        .new-feature { background: #e8f5e9; padding: 10px; border-radius: 5px; margin: 10px 0; }
      </style>
    </head>
    <body>
      <h1>🚀 Kubernetes Deployment Workshop</h1>
      <div class="version">Version 2.0</div>
      <div class="info">
        <h2>What's New in v2.0</h2>
        <div class="new-feature">✨ Improved Performance</div>
        <div class="new-feature">🔒 Enhanced Security</div>
        <div class="new-feature">📊 Better Monitoring</div>
      </div>
    </body>
    </html>
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
        location /health {
          return 200 'OK';
          add_header Content-Type text/plain;
        }
      }
    }
EOF

# Step 2: Update ConfigMap
kubectl apply -f /tmp/workshop-configmap-v2.yaml

# Step 3: Update Deployment (เปลี่ยน image และ annotations)
kubectl annotate deployment/webapp \
    kubernetes.io/change-cause="Update to version 2.0 with new features" \
    --overwrite

# สร้าง Deployment v2 manifest
cat <<'EOF' > /tmp/workshop-deployment-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: deployment-workshop
  labels:
    app: webapp
    version: "2.0"
  annotations:
    kubernetes.io/change-cause: "Update to version 2.0 - new UI and features"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: webapp
        version: "2.0"
    spec:
      containers:
      - name: webapp
        image: nginx:1.25    # อัปเกรด nginx version
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: config
        configMap:
          name: webapp-config
          items:
          - key: nginx.conf
            path: nginx.conf
      - name: html
        configMap:
          name: webapp-config
          items:
          - key: index.html
            path: index.html
EOF

# Step 4: Watch rolling update ในอีก terminal
kubectl get pods -l app=webapp --watch &
WATCH_PID=$!

# Step 5: Apply v2 deployment
kubectl apply -f /tmp/workshop-deployment-v2.yaml

# ดู rollout progress
kubectl rollout status deployment/webapp

kill $WATCH_PID 2>/dev/null

# Step 6: ยืนยัน version
kubectl get pods -l app=webapp \
    -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'

# ทดสอบ
kubectl port-forward service/webapp-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080
kill $PF_PID
```

### Lab 3: Rollback

```bash
# Step 1: สร้าง "bad" deployment (version 3.0 ที่มีปัญหา)
cat <<'EOF' > /tmp/workshop-deployment-v3-bad.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: deployment-workshop
  annotations:
    kubernetes.io/change-cause: "Update to version 3.0 - BUGGY VERSION"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: "3.0-bad"
    spec:
      containers:
      - name: webapp
        image: nginx:this-version-does-not-exist   # Image ที่ไม่มีจริง
        ports:
        - containerPort: 80
EOF

kubectl apply -f /tmp/workshop-deployment-v3-bad.yaml

# Step 2: ดูว่า update fail
kubectl rollout status deployment/webapp --timeout=60s || echo "Rollout failed!"

# Step 3: ดูสถานะ Pods
kubectl get pods -l app=webapp
# จะเห็น Pod ใหม่อยู่ใน ImagePullBackOff

# Step 4: ดู rollout history
kubectl rollout history deployment/webapp

# Step 5: Rollback ทันที!
kubectl rollout undo deployment/webapp

# Step 6: ดูการ rollback
kubectl rollout status deployment/webapp

# Step 7: ยืนยัน
kubectl get pods -l app=webapp
kubectl get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].image}'

# ทดสอบว่า app ทำงานปกติ
kubectl port-forward service/webapp-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080
kill $PF_PID
```

### Lab 4: Pause และ Resume Rollout

```bash
# สถานการณ์: ต้องการอัปเดตหลาย settings พร้อมกัน

# Pause deployment ก่อน
kubectl rollout pause deployment/webapp

# ทำการ update หลายอย่าง (จะไม่ trigger rollout ทันที)
kubectl set image deployment/webapp webapp=nginx:1.25
kubectl set resources deployment/webapp \
    -c=webapp \
    --limits=cpu=300m,memory=256Mi
kubectl set env deployment/webapp APP_VERSION="3.0-stable"

# ดูว่ายังไม่มีการ rollout
kubectl get rs -l app=webapp  # ยังมีแค่ RS เดิม

# Resume rollout (จะ apply การเปลี่ยนแปลงทั้งหมดพร้อมกัน)
kubectl rollout resume deployment/webapp

# ดู rollout progress
kubectl rollout status deployment/webapp
```

### Lab 5: Scaling

```bash
# Manual scale
kubectl scale deployment webapp --replicas=6
kubectl get pods -l app=webapp

# ดู ReplicaSet ที่ Deployment จัดการ
kubectl get rs -l app=webapp

# Scale down
kubectl scale deployment webapp --replicas=3
kubectl get pods -l app=webapp

# ตั้งค่า HPA (ต้องมี metrics-server)
kubectl autoscale deployment webapp \
    --min=3 \
    --max=10 \
    --cpu-percent=70

kubectl get hpa

# ลบ HPA
kubectl delete hpa webapp
```

### Lab 6: Canary Deployment (Manual)

```bash
# สถานการณ์: ต้องการทดสอบ version ใหม่กับ traffic จริง 20%

# Step 1: Deployment v1 (production) 80% traffic (4 replicas)
cat <<'EOF' > /tmp/canary-stable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-stable
  namespace: deployment-workshop
  labels:
    app: webapp
    track: stable
spec:
  replicas: 4        # 80% traffic
  selector:
    matchLabels:
      app: webapp
      track: stable
  template:
    metadata:
      labels:
        app: webapp   # Service จะเลือก Pod ทั้ง stable และ canary
        track: stable
    spec:
      containers:
      - name: webapp
        image: nginx:1.24
        ports:
        - containerPort: 80
EOF

# Step 2: Canary deployment 20% traffic (1 replica)
cat <<'EOF' > /tmp/canary-new.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-canary
  namespace: deployment-workshop
  labels:
    app: webapp
    track: canary
spec:
  replicas: 1        # 20% traffic
  selector:
    matchLabels:
      app: webapp
      track: canary
  template:
    metadata:
      labels:
        app: webapp   # Service จะเลือก Pod นี้ด้วย
        track: canary
    spec:
      containers:
      - name: webapp
        image: nginx:1.25     # New version
        ports:
        - containerPort: 80
EOF

# Step 3: Service ที่ load balance ทั้งสอง
cat <<'EOF' > /tmp/canary-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-canary-service
  namespace: deployment-workshop
spec:
  selector:
    app: webapp     # เลือก Pods ทั้ง stable และ canary
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f /tmp/canary-stable.yaml
kubectl apply -f /tmp/canary-new.yaml
kubectl apply -f /tmp/canary-service.yaml

# ดู Pods
kubectl get pods -l app=webapp --show-labels

# ถ้า canary ดี → scale canary up, scale stable down
# kubectl scale deployment webapp-canary --replicas=5
# kubectl scale deployment webapp-stable --replicas=0

# Cleanup canary
kubectl delete deployment webapp-stable webapp-canary
kubectl delete service webapp-canary-service
```

### Cleanup Workshop

```bash
# ลบ resources ทั้งหมด
kubectl delete -f /tmp/workshop-deployment-v1.yaml 2>/dev/null || true
kubectl delete -f /tmp/workshop-deployment-v2.yaml 2>/dev/null || true
kubectl delete -f /tmp/workshop-service.yaml 2>/dev/null || true
kubectl delete -f /tmp/workshop-configmap.yaml 2>/dev/null || true
kubectl delete -f /tmp/workshop-configmap-v2.yaml 2>/dev/null || true

# หรือลบ namespace ทั้งหมด
kubectl delete namespace deployment-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/workshop-*.yaml /tmp/canary-*.yaml
```

### Deployment Best Practices

```yaml
# ✓ Best Practice 1: ตั้งค่า Resources เสมอ
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# ✓ Best Practice 2: Health Probes ครบถ้วน
livenessProbe:   # restart container ถ้า fail
  httpGet:
    path: /health/live
    port: 8080
readinessProbe:  # ไม่ส่ง traffic ถ้า fail
  httpGet:
    path: /health/ready
    port: 8080
startupProbe:    # สำหรับ app ที่ start ช้า
  httpGet:
    path: /health/started
    port: 8080

# ✓ Best Practice 3: RollingUpdate strategy ที่เหมาะสม
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # zero downtime
    maxSurge: "25%"

# ✓ Best Practice 4: บันทึก change-cause
metadata:
  annotations:
    kubernetes.io/change-cause: "Update nginx to 1.25 - security fix CVE-2024-xxx"

# ✓ Best Practice 5: กำหนด revisionHistoryLimit
spec:
  revisionHistoryLimit: 5   # เก็บ 5 revision ล่าสุด

# ✓ Best Practice 6: Pod Anti-affinity เพื่อ spread
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: my-app
        topologyKey: kubernetes.io/hostname
```

---

## สรุป

Deployment คือ Controller ที่ใช้งานบ่อยที่สุดใน Kubernetes สำหรับ Stateless Applications:

1. **Declarative**: บอกว่าต้องการ state อะไร ไม่ใช่วิธีทำ
2. **Rolling Updates**: อัปเดตโดยไม่มี downtime
3. **Rollbacks**: ย้อนกลับได้ทันที
4. **Scaling**: scale up/down ได้ง่าย
5. **Self-healing**: ผ่าน ReplicaSet ที่มันจัดการ

ในบทต่อไปเราจะเรียนรู้ **Services** ซึ่งเป็นวิธีที่ Kubernetes จัดการ network access ไปยัง Pods
