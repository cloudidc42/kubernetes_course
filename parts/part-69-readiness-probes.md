# Part 69: Readiness Probes

## Readiness Probe คืออะไร

Readiness Probe เป็นกลไกที่ Kubernetes ใช้ตรวจสอบว่า Container พร้อมที่จะรับ Traffic หรือไม่ ถ้า Readiness Probe Fail Kubernetes จะ **ไม่ส่ง Traffic** ไปยัง Pod นั้น (ลบออกจาก Service Endpoints)

### ความแตกต่างระหว่าง Liveness และ Readiness Probe

| | Liveness Probe | Readiness Probe |
|--|---------------|-----------------|
| **วัตถุประสงค์** | ตรวจว่า Container ยัง Alive ไหม | ตรวจว่า Container พร้อมรับ Traffic ไหม |
| **Action เมื่อ Fail** | Restart Container | หยุดส่ง Traffic (ไม่ Restart) |
| **ใช้เมื่อ** | Application Stuck/Deadlock | Startup, Dependency Not Ready |
| **Effect** | Container Restart | Service Endpoint Removed |
| **Recovery** | Restart Container | Traffic กลับมาเมื่อ Probe Pass |

### Use Cases หลักของ Readiness Probe

```
1. Application Startup:
   - Application ต้องการเวลา Load Cache, Connect DB
   - ไม่ส่ง Traffic จนกว่าจะพร้อม

2. Rolling Updates:
   - ไม่ส่ง Traffic ไปยัง New Pod จนกว่าจะ Ready
   - ทำให้ Zero-downtime Deployment

3. Temporary Unavailability:
   - Application ไม่พร้อมชั่วคราว (เช่น DB Maintenance)
   - Kubernetes ไม่ส่ง Traffic ไป แต่ไม่ Restart

4. Dependencies Not Ready:
   - DB, Cache, External API ยังไม่พร้อม
   - รอจนกว่า Dependencies จะพร้อม
```

---

## ความแตกต่างจาก Liveness Probe อย่างละเอียด

### พฤติกรรมเมื่อ Probe Fail

```yaml
# Scenario: Application Connect DB ไม่ได้

# ด้วย Liveness Probe เท่านั้น:
livenessProbe:
  httpGet:
    path: /health  # ตรวจสอบรวมถึง DB
    port: 8080
# ผล: Kubernetes Restart Container ซ้ำๆ (CrashLoopBackOff)
# ปัญหา: DB อาจแค่ช้าชั่วคราว ไม่ควร Restart Application

# ด้วย Readiness Probe:
readinessProbe:
  httpGet:
    path: /ready  # ตรวจสอบรวมถึง DB
    port: 8080
# ผล: Kubernetes ไม่ส่ง Traffic ไปยัง Pod นี้
# เมื่อ DB กลับมา Probe จะ Pass และ Traffic จะกลับมา
```

### Endpoint Controller

```
เมื่อ Readiness Probe Fail:
1. Pod Status เปลี่ยนเป็น "Not Ready"
2. Endpoint Controller ลบ Pod IP ออกจาก Service Endpoints
3. kube-proxy หยุดส่ง Traffic ไปยัง Pod นั้น
4. เมื่อ Probe Pass อีกครั้ง Pod จะถูกเพิ่มกลับ
```

---

## Readiness Probe Configuration

### Syntax และ Options

```yaml
readinessProbe:
  # ประเภท Probe (เลือกอย่างใดอย่างหนึ่ง)
  httpGet:
    path: /readyz
    port: 8080
    httpHeaders:
      - name: Accept
        value: application/json
  
  # หรือ TCP
  # tcpSocket:
  #   port: 5432
  
  # หรือ Exec
  # exec:
  #   command:
  #     - sh
  #     - -c
  #     - "pg_isready -U postgres"
  
  # Timing
  initialDelaySeconds: 5    # รอก่อน Probe ครั้งแรก
  periodSeconds: 10         # Probe ทุก N วินาที
  timeoutSeconds: 3         # Timeout
  
  # Thresholds
  successThreshold: 1       # จำนวน Success ที่ต้องการ (อาจตั้งมากกว่า 1)
  failureThreshold: 3       # จำนวน Failure ก่อนถือว่า Not Ready
```

### successThreshold สำหรับ Readiness

```yaml
# Readiness Probe สามารถตั้ง successThreshold มากกว่า 1 ได้
# (ต่างจาก Liveness ที่ต้องเป็น 1)
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  successThreshold: 3  # ต้อง Pass 3 ครั้งติดต่อกันจึงจะถือว่า Ready
  periodSeconds: 5     # นั่นคือรอ 15 วินาทีก่อนรับ Traffic
```

---

## Workshop: Zero-downtime Deployments

### เป้าหมาย

ในส่วนนี้เราจะ:
1. ทำความเข้าใจ Rolling Update
2. Deploy Application พร้อม Readiness Probe
3. ทดสอบ Zero-downtime Deployment
4. จำลองสถานการณ์ที่ Readiness Probe Fail ระหว่าง Deployment
5. ดูพฤติกรรมของ Kubernetes

### ขั้นตอนที่ 1: สร้าง Namespace

```bash
kubectl create namespace readiness-workshop
kubectl config set-context --current --namespace=readiness-workshop
```

### ขั้นตอนที่ 2: Deploy Application เวอร์ชัน 1

```yaml
# app-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: readiness-workshop
  labels:
    app: web-app
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 3
  # Rolling Update Strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # ไม่ยอมให้มี Pod Unavailable ระหว่าง Update
      maxSurge: 1            # สร้าง Extra Pod ได้ 1 ตัว
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: "1.0"
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Version 1.0 Application
              APP_VERSION="1.0"
              echo "App v$APP_VERSION starting..."
              
              # Startup time: 5 วินาที
              sleep 5
              touch /tmp/ready
              echo "App v$APP_VERSION is ready!"
              
              # HTTP Server
              while true; do
                printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"version":"'"$APP_VERSION"'","pod":"'"$HOSTNAME"'","status":"ok"}\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
              name: http
          # Readiness Probe
          readinessProbe:
            exec:
              command:
                - cat
                - /tmp/ready
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
            successThreshold: 1
          # Liveness Probe  
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - "[ -f /tmp/ready ]"
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: readiness-workshop
spec:
  selector:
    app: web-app
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

```bash
# Apply Version 1
kubectl apply -f app-v1.yaml

# รอให้ Pods Ready
kubectl wait --for=condition=Ready pod -l app=web-app \
  -n readiness-workshop --timeout=60s

# ดู Status
kubectl get pods -n readiness-workshop
kubectl get endpoints -n readiness-workshop
```

### ขั้นตอนที่ 3: สร้าง Load Test Script

```bash
# ใน Terminal ใหม่ - ส่ง Traffic ตลอดเวลา
cat > /tmp/load-test.sh << 'LOAD_EOF'
#!/bin/bash
SERVICE_URL="${1:-http://localhost:8080}"
INTERVAL="${2:-0.5}"
SUCCESS=0
FAILED=0
TOTAL=0

echo "Starting load test against $SERVICE_URL"
echo "Press Ctrl+C to stop"
echo ""

while true; do
    RESPONSE=$(curl -s -o /tmp/response.json -w "%{http_code}" "$SERVICE_URL" --max-time 2 2>/dev/null)
    TOTAL=$((TOTAL + 1))
    
    if [ "$RESPONSE" = "200" ]; then
        SUCCESS=$((SUCCESS + 1))
        VERSION=$(cat /tmp/response.json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('version','?'))" 2>/dev/null)
        echo -ne "\r[$(date +%H:%M:%S)] Success: $SUCCESS | Failed: $FAILED | Total: $TOTAL | Version: $VERSION    "
    else
        FAILED=$((FAILED + 1))
        echo -ne "\r[$(date +%H:%M:%S)] Success: $SUCCESS | Failed: $FAILED | Total: $TOTAL | HTTP: $RESPONSE    "
    fi
    
    sleep "$INTERVAL"
done
LOAD_EOF

chmod +x /tmp/load-test.sh
```

```bash
# Port Forward Service
kubectl port-forward -n readiness-workshop svc/web-app 8080:8080 &
sleep 2

# เริ่ม Load Test ใน Background
/tmp/load-test.sh http://localhost:8080 0.2 &
LOAD_TEST_PID=$!

echo "Load test started (PID: $LOAD_TEST_PID)"
echo "Observing traffic..."
sleep 10
```

### ขั้นตอนที่ 4: Deploy Version 2 (Zero-downtime)

```yaml
# app-v2.yaml - Update เป็น Version 2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: readiness-workshop
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: "2.0"
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              APP_VERSION="2.0"
              echo "App v$APP_VERSION starting..."
              
              # Version 2 ใช้เวลา Startup นานขึ้น 10 วินาที
              sleep 10
              touch /tmp/ready
              echo "App v$APP_VERSION is ready!"
              
              while true; do
                printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"version":"'"$APP_VERSION"'","pod":"'"$HOSTNAME"'","status":"ok","new_feature":true}\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
          readinessProbe:
            exec:
              command:
                - cat
                - /tmp/ready
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - "[ -f /tmp/ready ]"
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

```bash
# Apply Version 2 และดู Rolling Update
kubectl apply -f app-v2.yaml

# Watch Rolling Update Progress
kubectl rollout status deployment/web-app -n readiness-workshop -w &

# ดู Pods ระหว่าง Rolling Update
kubectl get pods -n readiness-workshop -w

# Load Test ยังทำงานอยู่ - ควรไม่มี Failed Requests
sleep 60
kill $LOAD_TEST_PID 2>/dev/null
```

### ขั้นตอนที่ 5: จำลอง Broken Deployment

```yaml
# broken-app-v3.yaml - Deployment ที่ Readiness Probe ไม่ Pass
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: readiness-workshop
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: "3.0-broken"
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              APP_VERSION="3.0-broken"
              echo "App v$APP_VERSION starting..."
              
              # Bug: ไม่สร้าง /tmp/ready ดังนั้น Readiness Probe จะไม่ Pass
              echo "Simulating startup bug - not creating readiness file"
              
              while true; do
                printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"version":"'"$APP_VERSION"'"}\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
          readinessProbe:
            exec:
              command:
                - cat
                - /tmp/ready  # ไม่มีไฟล์นี้ -> Probe Fail
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

```bash
# เริ่ม Load Test ใหม่
kubectl port-forward -n readiness-workshop svc/web-app 8080:8080 &
/tmp/load-test.sh http://localhost:8080 0.2 &
LOAD_TEST_PID=$!

sleep 5

# Apply Broken Version
kubectl apply -f broken-app-v3.yaml

# ดูสิ่งที่เกิดขึ้น:
# 1. New Pod จะถูกสร้างแต่ Readiness Probe จะ Fail
# 2. Kubernetes จะไม่ส่ง Traffic ไปยัง New Pod
# 3. Old Pods ยังทำงานอยู่และรับ Traffic
# 4. Rolling Update จะ Stuck (ไม่ Progress)

kubectl get pods -n readiness-workshop -w

# Load Test ควรไม่มี Failed (Old Pods ยังทำงาน)
echo "Observing... Kubernetes should NOT send traffic to broken pods"
sleep 60

# ดู Events
kubectl get events -n readiness-workshop --sort-by='.lastTimestamp'

# ดู Status ของ Deployment
kubectl describe deployment web-app -n readiness-workshop | \
  grep -A 20 "Conditions:"
```

### ขั้นตอนที่ 6: Rollback Deployment

```bash
# หยุด Load Test
kill $LOAD_TEST_PID 2>/dev/null

# Rollback ไปยัง Version ก่อนหน้า
kubectl rollout undo deployment/web-app -n readiness-workshop

# ดู Rollback Progress
kubectl rollout status deployment/web-app -n readiness-workshop -w

# ดู Rollout History
kubectl rollout history deployment/web-app -n readiness-workshop

# ตรวจสอบ Version
kubectl get pods -n readiness-workshop -o jsonpath='{.items[*].metadata.labels.version}'
```

### ขั้นตอนที่ 7: Readiness Probe กับ Database Dependencies

```yaml
# db-dependency-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-dependent-app
  namespace: readiness-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: db-dependent
  template:
    metadata:
      labels:
        app: db-dependent
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # จำลอง Application ที่ต้องการ DB Connection
              echo "Starting application..."
              
              # รอ DB (จำลองด้วย File)
              DB_HOST="${DB_HOST:-localhost}"
              
              # ลอง Connect DB (จำลอง)
              MAX_RETRIES=10
              RETRY=0
              
              while [ $RETRY -lt $MAX_RETRIES ]; do
                # จำลอง DB Check
                if [ -f /tmp/db-connected ]; then
                  echo "DB Connected!"
                  break
                fi
                echo "Waiting for DB... ($RETRY/$MAX_RETRIES)"
                RETRY=$((RETRY + 1))
                sleep 3
              done
              
              # หาก Connect ไม่ได้ก็ยังทำงานต่อ (แต่ไม่ Serve Traffic)
              touch /tmp/app-started
              
              while true; do
                if [ -f /tmp/db-connected ]; then
                  touch /tmp/ready
                  printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"status":"ok","db":"connected"}\n' | \
                    nc -l -p 8080 -q 1 2>/dev/null
                else
                  rm -f /tmp/ready
                  printf 'HTTP/1.0 503 Service Unavailable\r\nContent-Type: application/json\r\n\r\n{"status":"error","db":"disconnected"}\n' | \
                    nc -l -p 8080 -q 1 2>/dev/null
                fi
              done
          env:
            - name: DB_HOST
              value: "my-database"
          ports:
            - containerPort: 8080
          # Readiness ตรวจสอบ DB Connection
          readinessProbe:
            exec:
              command:
                - cat
                - /tmp/ready
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 6  # รอ DB นานขึ้น
            successThreshold: 2  # ต้อง Ready 2 ครั้งติดต่อกัน
          # Liveness ตรวจสอบแค่ว่า Process ยัง Run
          livenessProbe:
            exec:
              command:
                - cat
                - /tmp/app-started
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

```bash
# Apply
kubectl apply -f db-dependency-app.yaml

# ดู Pods - ควรอยู่ใน "Not Ready" State เพราะ DB ยังไม่ Connect
kubectl get pods -n readiness-workshop -l app=db-dependent

# จำลองว่า DB Connect แล้ว
DB_POD=$(kubectl get pods -n readiness-workshop -l app=db-dependent \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec $DB_POD -n readiness-workshop -- touch /tmp/db-connected
sleep 10  # รอ Readiness Probe Check

# ดู Pod Status ควรเปลี่ยนเป็น Ready
kubectl get pods -n readiness-workshop -l app=db-dependent
```

### ขั้นตอนที่ 8: Advanced Readiness Probe

```yaml
# advanced-readiness.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advanced-readiness
  namespace: readiness-workshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: advanced-readiness
  minReadySeconds: 10  # Pod ต้อง Ready อย่างน้อย 10 วินาทีก่อนถือว่า Available
  template:
    metadata:
      labels:
        app: advanced-readiness
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Application starting..."
              
              # Phase 1: Load configuration (3 วินาที)
              echo "Loading configuration..."
              sleep 3
              echo "config_loaded=true" > /tmp/status
              
              # Phase 2: Connect to dependencies (5 วินาที)
              echo "Connecting to dependencies..."
              sleep 5
              echo "deps_ready=true" >> /tmp/status
              
              # Phase 3: Warm up cache (5 วินาที)
              echo "Warming up cache..."
              sleep 5
              echo "cache_warmed=true" >> /tmp/status
              touch /tmp/ready
              
              echo "Application is ready!"
              
              # Serve
              while true; do
                printf 'HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\n\r\nOK\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - |
                  # ตรวจสอบว่าทุก Phase เสร็จแล้ว
                  [ -f /tmp/ready ] && \
                  grep -q "config_loaded=true" /tmp/status && \
                  grep -q "deps_ready=true" /tmp/status && \
                  grep -q "cache_warmed=true" /tmp/status
            initialDelaySeconds: 0
            periodSeconds: 3
            timeoutSeconds: 2
            successThreshold: 1
            failureThreshold: 60  # รอสูงสุด 3 นาที (60 * 3s)
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - "test -f /tmp/status"
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

```bash
# Apply Advanced Readiness
kubectl apply -f advanced-readiness.yaml

# Watch Pod becoming Ready
kubectl get pods -n readiness-workshop -l app=advanced-readiness -w

# ดู Events
kubectl describe pod -l app=advanced-readiness -n readiness-workshop | \
  grep -A 20 "Events:"
```

### ขั้นตอนที่ 9: Monitoring Readiness State

```bash
#!/bin/bash
# monitor-readiness.sh

NAMESPACE="${1:-readiness-workshop}"
INTERVAL="${2:-5}"

echo "=== Readiness Monitor for Namespace: $NAMESPACE ==="

while true; do
    clear
    echo "Time: $(date)"
    echo "Namespace: $NAMESPACE"
    echo ""
    
    # Pod Status
    echo "--- Pod Readiness Status ---"
    kubectl get pods -n "$NAMESPACE" \
        -o custom-columns=\
'NAME:.metadata.name,'\
'READY:.status.containerStatuses[0].ready,'\
'STATUS:.status.phase,'\
'RESTARTS:.status.containerStatuses[0].restartCount,'\
'IP:.status.podIP' \
        2>/dev/null
    
    echo ""
    echo "--- Endpoints ---"
    kubectl get endpoints -n "$NAMESPACE" 2>/dev/null
    
    echo ""
    echo "--- Recent Events (Not Ready) ---"
    kubectl get events -n "$NAMESPACE" \
        --field-selector reason=Unhealthy \
        --sort-by='.lastTimestamp' 2>/dev/null | tail -5
    
    sleep "$INTERVAL"
done
```

### ขั้นตอนที่ 10: Production Readiness Pattern

```yaml
# production-readiness.yaml
# Best Practice สำหรับ Production Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: readiness-workshop
  annotations:
    description: "Production-grade deployment with proper probes"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # Zero-downtime
      maxSurge: 1
  minReadySeconds: 15        # Pod ต้อง Stable 15 วินาทีก่อน Progress
  progressDeadlineSeconds: 300  # Deployment ต้อง Progress ภายใน 5 นาที
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Graceful Shutdown
      terminationGracePeriodSeconds: 30
      
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Setup Signal Handlers
              cleanup() {
                echo "Received shutdown signal..."
                rm -f /tmp/ready  # หยุดรับ Traffic
                echo "Draining connections..."
                sleep 10  # รอให้ Connections หมด
                echo "Shutting down"
                exit 0
              }
              trap cleanup SIGTERM SIGINT
              
              # Startup
              echo "Starting application..."
              sleep 5
              touch /tmp/alive
              sleep 5
              touch /tmp/ready
              echo "Application ready!"
              
              # Serve
              while true; do
                printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"status":"ok"}\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
              name: http
          
          # Startup Probe: รอ Application Start
          startupProbe:
            exec:
              command: ["cat", "/tmp/alive"]
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 12  # รอสูงสุด 60 วินาที
          
          # Readiness Probe: ตรวจสอบพร้อมรับ Traffic
          readinessProbe:
            exec:
              command: ["cat", "/tmp/ready"]
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 2   # ต้อง Ready 2 ครั้งติดต่อกัน
            failureThreshold: 3
          
          # Liveness Probe: ตรวจสอบ Process Healthy
          livenessProbe:
            exec:
              command: ["cat", "/tmp/alive"]
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: production-app
  namespace: readiness-workshop
spec:
  selector:
    app: production-app
  ports:
    - port: 8080
      targetPort: 8080
  # sessionAffinity: ClientIP  # Optional: Sticky Sessions
```

```bash
# Apply Production App
kubectl apply -f production-readiness.yaml

# Verify Zero-downtime Update
kubectl port-forward -n readiness-workshop svc/production-app 8080:8080 &
sleep 2

# Start Load Test
/tmp/load-test.sh http://localhost:8080 0.1 &
LOAD_PID=$!
sleep 10

# Update Image (จำลอง)
kubectl patch deployment production-app \
  -n readiness-workshop \
  --type=json \
  -p='[{"op":"replace","path":"/spec/template/metadata/labels/version","value":"2.0"}]'

# ดู Rolling Update
kubectl rollout status deployment/production-app -n readiness-workshop

# หยุด Load Test
kill $LOAD_PID 2>/dev/null
```

### ขั้นตอนที่ 11: Readiness Gates (Advanced)

```yaml
# readiness-gate.yaml
# Pod Readiness Gates - Custom Conditions
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-gate-demo
  namespace: readiness-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readiness-gate
  template:
    metadata:
      labels:
        app: readiness-gate
    spec:
      # Readiness Gates ต้องถูก Set ด้วย External Controller
      readinessGates:
        - conditionType: "example.com/external-ready"
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "while true; do sleep 60; done"]
          readinessProbe:
            exec:
              command: ["sh", "-c", "exit 0"]
            periodSeconds: 5
          resources:
            requests:
              memory: "16Mi"
              cpu: "5m"
```

### ขั้นตอนที่ 12: ทำความสะอาด

```bash
# หยุด Port Forwards
kill $(lsof -ti:8080) 2>/dev/null || true

# ลบ Namespace
kubectl delete namespace readiness-workshop

# Reset Default Namespace
kubectl config set-context --current --namespace=default

echo "Cleanup เสร็จสิ้น"
```

---

## Tips และ Best Practices

### 1. Separate Liveness and Readiness Endpoints

```python
# Best Practice: แยก Endpoints
@app.route('/livez')
def liveness():
    """ตรวจสอบแค่ว่า Application Process ยัง Alive"""
    return {"status": "alive"}, 200

@app.route('/readyz')
def readiness():
    """ตรวจสอบว่าพร้อมรับ Traffic รวมถึง Dependencies"""
    checks = {}
    
    # Check Database
    try:
        db.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {str(e)}"
    
    # Check Cache
    try:
        cache.ping()
        checks["cache"] = "ok"
    except Exception as e:
        checks["cache"] = f"error: {str(e)}"
    
    # All checks must pass
    if all(v == "ok" for v in checks.values()):
        return {"status": "ready", "checks": checks}, 200
    else:
        return {"status": "not ready", "checks": checks}, 503
```

### 2. ใช้ minReadySeconds

```yaml
# ป้องกัน Pod ที่ Ready แต่ยัง Unstable
spec:
  minReadySeconds: 30  # Pod ต้อง Ready อย่างน้อย 30 วินาทีก่อน Rolling Update จะ Progress
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

### 3. ตั้ง progressDeadlineSeconds

```yaml
# หากการ Deployment ไม่ Progress ภายในเวลาที่กำหนด ให้ถือว่าล้มเหลว
spec:
  progressDeadlineSeconds: 600  # 10 นาที
  strategy:
    type: RollingUpdate
```

---

## สรุป

Readiness Probe เป็นกลไกสำคัญสำหรับ Zero-downtime Deployments:

1. **ความแตกต่างจาก Liveness**: Readiness ไม่ Restart Container แต่หยุดส่ง Traffic
2. **Zero-downtime**: ใช้ maxUnavailable=0 และ Readiness Probe ร่วมกัน
3. **Dependency Checks**: ตรวจสอบ DB, Cache, External APIs ใน Readiness
4. **Rolling Updates**: Kubernetes จะไม่ Progress ถ้า New Pods ไม่ Ready
5. **Rollback**: Kubernetes สามารถ Rollback อัตโนมัติถ้า Deployment ล้มเหลว

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ Startup Probe ซึ่งเป็นตัวช่วยสำหรับ Applications ที่ใช้เวลา Startup นาน
