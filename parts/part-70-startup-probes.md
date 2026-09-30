# Part 70: Startup Probes

## Startup Probe คืออะไร

Startup Probe เป็น Probe ประเภทที่สามใน Kubernetes (เพิ่มมาใน Kubernetes 1.16) มีวัตถุประสงค์เฉพาะสำหรับ Applications ที่ใช้เวลา Startup นาน

### ปัญหาที่ Startup Probe แก้

ก่อนมี Startup Probe เราต้องใช้ `initialDelaySeconds` ใน Liveness Probe เพื่อรอให้ Application Start แต่มีปัญหา:

```
ปัญหาด้วย initialDelaySeconds:
1. ถ้าตั้งน้อยเกินไป: Application ยัง Start ไม่เสร็จ Liveness Probe จะ Fail
   -> Container Restart ก่อนจะ Start ได้
   
2. ถ้าตั้งมากเกินไป: รอนาน แม้ Application Start เร็ว
   -> Detection เมื่อ Application Crash ช้าลง

Startup Probe แก้โดย:
- รอจนกว่า Application จะ Start เสร็จ (ไม่ว่าจะนานแค่ไหน)
- หลังจากนั้น Liveness/Readiness Probe จึงเริ่มทำงาน
- ถ้า Startup Probe Fail ถึง Threshold จึง Restart
```

### เมื่อไหรควรใช้ Startup Probe

```
ควรใช้เมื่อ:
1. Application ใช้เวลา Startup มาก (>60 วินาที)
2. Startup Time ไม่แน่นอน (อาจเร็วหรือช้า)
3. Legacy Applications ที่ Startup ช้า
4. Applications ที่ต้อง Load ข้อมูลมากก่อน Start
5. ML Models ที่ต้อง Load Model ก่อน Serve

ไม่จำเป็นต้องใช้เมื่อ:
1. Application Start เร็ว (<30 วินาที)
2. Startup Time คงที่และสั้น
```

---

## Use Cases

### Use Case 1: Java/Spring Boot Application

```yaml
# Java Spring Boot มักใช้เวลา Startup 30-120 วินาที
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot
  template:
    metadata:
      labels:
        app: spring-boot
    spec:
      containers:
        - name: app
          image: my-spring-app:1.0
          ports:
            - containerPort: 8080
          
          # Startup Probe: รอจนกว่า JVM จะ Start และ Context จะ Load
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30   # รอสูงสุด 30 * 10s = 5 นาที
            periodSeconds: 10
            timeoutSeconds: 3
          
          # Liveness Probe: เริ่มทำงานหลัง Startup Probe Pass
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Readiness Probe: ตรวจสอบว่า App พร้อมรับ Traffic
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

### Use Case 2: Machine Learning Model

```yaml
# ML Model ต้อง Load Model File ก่อน Serve
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-inference
  template:
    metadata:
      labels:
        app: ml-inference
    spec:
      containers:
        - name: model-server
          image: my-ml-service:1.0
          ports:
            - containerPort: 8501
          
          # Startup Probe: รอจนกว่า Model จะ Load (อาจนาน 5 นาที)
          startupProbe:
            httpGet:
              path: /v1/models/my_model  # TensorFlow Serving Health
              port: 8501
            failureThreshold: 60   # 60 * 5s = 5 นาที
            periodSeconds: 5
            timeoutSeconds: 10     # Model Health Check อาจช้า
          
          livenessProbe:
            httpGet:
              path: /v1/models/my_model
              port: 8501
            periodSeconds: 30
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /v1/models/my_model:predict
              port: 8501
            periodSeconds: 10
            failureThreshold: 3
          
          resources:
            requests:
              memory: "4Gi"
              cpu: "2000m"
            limits:
              memory: "8Gi"
              cpu: "4000m"
          
          volumeMounts:
            - name: model-storage
              mountPath: /models
      
      volumes:
        - name: model-storage
          persistentVolumeClaim:
            claimName: ml-models-pvc
```

### Use Case 3: Database Migration Application

```yaml
# Application ที่ต้อง Run Database Migration ก่อน Start
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-migration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-with-migration
  template:
    metadata:
      labels:
        app: web-with-migration
    spec:
      initContainers:
        - name: db-migration
          image: my-app:1.0
          command: ["./migrate.sh"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
      
      containers:
        - name: app
          image: my-app:1.0
          command: ["./start.sh"]
          ports:
            - containerPort: 8080
          
          # Startup Probe: รอ Warm-up Process
          startupProbe:
            httpGet:
              path: /ready
              port: 8080
            failureThreshold: 24   # 24 * 5s = 2 นาที
            periodSeconds: 5
          
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 15
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
            failureThreshold: 3
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

---

## ความแตกต่างจาก Probes อื่น

### การทำงานร่วมกันของทั้ง 3 Probes

```
Container Start
     |
     v
+------------------+
|  Startup Probe   |  <- ทำงานก่อน ตรวจสอบว่า App Start เสร็จ
|  (ถ้ามี)         |
+--------+---------+
         |
         | Pass (Startup Probe สำเร็จ)
         |
         v
+------------------+          +------------------+
|  Liveness Probe  |          | Readiness Probe  |
| (ทำงานพร้อมกัน)  |          | (ทำงานพร้อมกัน)  |
+------------------+          +------------------+
  Fail -> Restart               Fail -> No Traffic
```

### Timeline ของ Probes

```
t=0s   Container Start
t=0s   Startup Probe เริ่ม Probe (periodSeconds ทุกครั้ง)
t=5s   Startup Probe Probe ครั้งที่ 1
t=10s  Startup Probe Probe ครั้งที่ 2
t=15s  Startup Probe PASS (Application Ready)
       -> Liveness และ Readiness Probe เริ่มทำงาน
t=15s  Liveness Probe เริ่ม (initialDelaySeconds = 0)
t=15s  Readiness Probe เริ่ม (initialDelaySeconds = 0)
```

### ตาราง เปรียบเทียบ Probes

| | Startup | Liveness | Readiness |
|--|---------|----------|-----------|
| **ทำงานเมื่อ** | Container Start | ตลอดเวลา (หลัง Startup Pass) | ตลอดเวลา (หลัง Startup Pass) |
| **Action เมื่อ Fail** | Restart (หลัง Threshold) | Restart | No Traffic |
| **วัตถุประสงค์** | รอ Application Start | ตรวจ Process Alive | ตรวจพร้อมรับ Traffic |
| **successThreshold** | ต้องเป็น 1 | ต้องเป็น 1 | 1 หรือมากกว่า |
| **ทำงานพร้อมกัน** | ไม่ (ทำก่อน) | ใช่ (พร้อมกัน) | ใช่ (พร้อมกัน) |

---

## Workshop: Handle Slow-starting Applications

### เป้าหมาย

ในส่วนนี้เราจะ:
1. ทดสอบปัญหาที่เกิดโดยไม่มี Startup Probe
2. แก้ไขด้วย Startup Probe
3. ปรับแต่ง Startup Probe Configuration
4. ทดสอบกับ Applications ประเภทต่างๆ

### ขั้นตอนที่ 1: เตรียม Namespace

```bash
kubectl create namespace startup-workshop
kubectl config set-context --current --namespace=startup-workshop
```

### ขั้นตอนที่ 2: จำลองปัญหา - Application Restart เพราะ Liveness Probe

```yaml
# problem-without-startup.yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-problem
  namespace: startup-workshop
  labels:
    app: slow-start-problem
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Application starting slowly..."
          # จำลอง Application ที่ใช้เวลา 60 วินาทีในการ Start
          sleep 60
          echo "Application ready!"
          touch /tmp/ready
          while true; do
            printf 'HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\n\r\nOK\n' | \
              nc -l -p 8080 -q 1 2>/dev/null
          done
      ports:
        - containerPort: 8080
      # Liveness Probe ที่ initialDelaySeconds น้อยเกินไป
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/ready
        initialDelaySeconds: 10  # ตั้งน้อยเกินไป! Application ใช้เวลา 60 วินาที
        periodSeconds: 5
        failureThreshold: 3  # จะ Fail หลัง 3 ครั้ง (10+5+5+5 = 25 วินาที)
      resources:
        requests:
          memory: "16Mi"
          cpu: "5m"
        limits:
          memory: "32Mi"
          cpu: "20m"
```

```bash
# Apply
kubectl apply -f problem-without-startup.yaml

# ดูสิ่งที่เกิดขึ้น: Pod จะ Restart ก่อน Start เสร็จ
kubectl get pod slow-start-problem -n startup-workshop -w

# ดู Events
kubectl get events -n startup-workshop \
  --field-selector involvedObject.name=slow-start-problem \
  --sort-by='.lastTimestamp'

# ควรเห็น:
# Warning  Unhealthy  Liveness probe failed
# Normal   Killing    Container was killed
# Normal   Started    Started container
# (แล้วก็ Restart ซ้ำๆ)
```

### ขั้นตอนที่ 3: แก้ไขด้วย Startup Probe

```yaml
# fixed-with-startup.yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-fixed
  namespace: startup-workshop
  labels:
    app: slow-start-fixed
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Application starting slowly..."
          sleep 60
          echo "Application ready!"
          touch /tmp/ready
          while true; do
            printf 'HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\n\r\nOK\n' | \
              nc -l -p 8080 -q 1 2>/dev/null
          done
      ports:
        - containerPort: 8080
      
      # Startup Probe: รอจนกว่า Application จะ Start
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/ready
        failureThreshold: 30   # รอสูงสุด 30 * 5s = 150 วินาที
        periodSeconds: 5
        timeoutSeconds: 3
      
      # Liveness Probe: เริ่มทำงานหลัง Startup Pass
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/ready
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 3
      
      resources:
        requests:
          memory: "16Mi"
          cpu: "5m"
        limits:
          memory: "32Mi"
          cpu: "20m"
```

```bash
# Apply Fixed Version
kubectl apply -f fixed-with-startup.yaml

# ดู Pods - Fixed version จะไม่ Restart
kubectl get pods -n startup-workshop -w

# เปรียบเทียบ Restart Count
echo "=== Restart Count Comparison ==="
kubectl get pod slow-start-problem slow-start-fixed \
  -n startup-workshop \
  -o custom-columns='NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount' \
  2>/dev/null
```

### ขั้นตอนที่ 4: Slow-starting Java Application Demo

```yaml
# java-slow-start.yaml
# จำลอง Java/Spring Boot Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-demo
  namespace: startup-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-app-demo
  template:
    metadata:
      labels:
        app: java-app-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "JVM starting..."
              sleep 5
              
              echo "Loading Spring Context..."
              sleep 10
              
              echo "Connecting to database..."
              sleep 5
              
              echo "Running database migrations..."
              sleep 10
              
              echo "Loading application data..."
              sleep 5
              
              echo "Warming up JIT compiler..."
              sleep 5
              
              # Total startup time: ~40 วินาที
              touch /tmp/startup-complete
              echo "Application started! Ready to serve traffic."
              
              while true; do
                if [ -f /tmp/startup-complete ]; then
                  printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"status":"ok","type":"java-app"}\n' | \
                    nc -l -p 8080 -q 1 2>/dev/null
                fi
              done
          ports:
            - containerPort: 8080
          
          # Startup Probe: รอ JVM และ Spring Context Load
          startupProbe:
            exec:
              command: ["cat", "/tmp/startup-complete"]
            failureThreshold: 20   # 20 * 5s = 100 วินาที maximum
            periodSeconds: 5
            timeoutSeconds: 3
          
          # Liveness Probe: ตรวจสอบหลัง Start
          livenessProbe:
            exec:
              command: ["cat", "/tmp/startup-complete"]
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Readiness Probe
          readinessProbe:
            exec:
              command: ["cat", "/tmp/startup-complete"]
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

```bash
# Apply Java App Demo
kubectl apply -f java-slow-start.yaml

# Watch Pods - ควรไม่มี Restart แม้ Startup จะช้า
kubectl get pods -n startup-workshop -l app=java-app-demo -w

# ดู Events
kubectl get events -n startup-workshop \
  --field-selector reason=Started \
  --sort-by='.lastTimestamp'
```

### ขั้นตอนที่ 5: Startup Probe สำหรับ Database

```yaml
# database-startup.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql-demo
  namespace: startup-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: "mysecretpassword"
            - name: POSTGRES_DB
              value: "mydb"
            - name: POSTGRES_USER
              value: "myuser"
          
          # Startup Probe: รอ PostgreSQL Ready
          startupProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myuser
                - -d
                - mydb
            failureThreshold: 30   # รอสูงสุด 30 * 5s = 150 วินาที
            periodSeconds: 5
            timeoutSeconds: 5
          
          # Liveness Probe
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myuser
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Readiness Probe  
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - myuser
                - -d
                - mydb
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      
      volumes:
        - name: postgres-data
          emptyDir: {}
```

```bash
# Apply PostgreSQL
kubectl apply -f database-startup.yaml

# Watch PostgreSQL Start
kubectl get pods -n startup-workshop -l app=postgresql -w

# ดู Events
kubectl describe pod -l app=postgresql -n startup-workshop
```

### ขั้นตอนที่ 6: คำนวณ Startup Probe Timing

```bash
# Script คำนวณ Startup Probe Timing
calculate_startup_probe() {
    local max_startup_seconds="$1"
    local period="${2:-10}"
    
    # คำนวณ failureThreshold
    FAILURE_THRESHOLD=$(echo "$max_startup_seconds / $period" | bc)
    
    echo "=== Startup Probe Calculation ==="
    echo "Maximum Startup Time: ${max_startup_seconds}s"
    echo "Period Seconds: ${period}s"
    echo "Failure Threshold: ${FAILURE_THRESHOLD}"
    echo ""
    echo "Resulting YAML:"
    echo "startupProbe:"
    echo "  httpGet:"
    echo "    path: /health"
    echo "    port: 8080"
    echo "  failureThreshold: $FAILURE_THRESHOLD"
    echo "  periodSeconds: $period"
    echo "  timeoutSeconds: 5"
}

# ตัวอย่าง
calculate_startup_probe 120 10  # App ใช้เวลา Startup 120 วินาที, Check ทุก 10 วินาที
calculate_startup_probe 300 15  # App ใช้เวลา Startup 5 นาที, Check ทุก 15 วินาที
```

### ขั้นตอนที่ 7: ทดสอบ Startup Probe Failure

```yaml
# startup-probe-failure.yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-failure-demo
  namespace: startup-workshop
  labels:
    app: startup-failure
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          # Application ที่ไม่ Start ได้ในเวลาที่กำหนด
          echo "Application starting..."
          # ใช้เวลา 120 วินาที แต่ Startup Probe มี Threshold แค่ 15 วินาที
          sleep 120
          touch /tmp/ready
          while true; do sleep 60; done
      # Startup Probe ที่ Threshold น้อยเกินไป
      startupProbe:
        exec:
          command: ["cat", "/tmp/ready"]
        failureThreshold: 3   # รอแค่ 3 * 5s = 15 วินาที (น้อยเกินไป!)
        periodSeconds: 5
      resources:
        requests:
          memory: "16Mi"
          cpu: "5m"
```

```bash
# Apply
kubectl apply -f startup-probe-failure.yaml

# ดูสิ่งที่เกิดขึ้น: Startup Probe Fail -> Restart
kubectl get pod startup-failure-demo -n startup-workshop -w

# ดู Events
kubectl get events -n startup-workshop \
  --field-selector involvedObject.name=startup-failure-demo \
  --sort-by='.lastTimestamp'

# ควรเห็น:
# Warning  Unhealthy  Startup probe failed: command cat failed with code 1
# Warning  BackOff    Back-off restarting failed container
```

### ขั้นตอนที่ 8: All Three Probes ใน Production Setup

```yaml
# production-all-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app-all-probes
  namespace: startup-workshop
  annotations:
    description: "Complete example with all 3 probe types"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-all-probes
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: production-all-probes
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Signal Handler สำหรับ Graceful Shutdown
              cleanup() {
                echo "Shutdown signal received"
                rm -f /tmp/ready  # หยุดรับ Traffic
                sleep 5           # Drain connections
                exit 0
              }
              trap cleanup SIGTERM SIGINT
              
              echo "=== Phase 1: Loading Configuration ==="
              sleep 5
              echo "config" > /tmp/status
              
              echo "=== Phase 2: Connecting Dependencies ==="
              sleep 10
              echo "deps" >> /tmp/status
              
              echo "=== Phase 3: Application Start ==="
              sleep 5
              touch /tmp/alive
              
              echo "=== Phase 4: Ready to Serve ==="
              sleep 5
              touch /tmp/ready
              
              echo "=== Application Fully Started ==="
              
              REQUESTS=0
              while true; do
                REQUESTS=$((REQUESTS + 1))
                printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"status":"ok","requests":'"$REQUESTS"'}\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
              name: http
          
          # ====================
          # STARTUP PROBE
          # ====================
          # รอจนกว่า Application จะ Start เสร็จ
          # ทำงานก่อน Liveness และ Readiness
          startupProbe:
            exec:
              command:
                - sh
                - -c
                - |
                  # ตรวจสอบว่า Phase ทั้งหมดเสร็จ
                  [ -f /tmp/alive ] && [ -f /tmp/ready ]
            failureThreshold: 20   # รอสูงสุด 100 วินาที (20 * 5s)
            periodSeconds: 5
            timeoutSeconds: 3
          
          # ====================
          # LIVENESS PROBE
          # ====================
          # เริ่มทำงานหลัง Startup Pass
          # Restart เมื่อ Application Stuck
          livenessProbe:
            exec:
              command:
                - cat
                - /tmp/alive
            initialDelaySeconds: 0  # เริ่มทันทีหลัง Startup Pass
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3     # Restart หลัง 45 วินาที
          
          # ====================
          # READINESS PROBE
          # ====================
          # เริ่มทำงานหลัง Startup Pass
          # หยุดส่ง Traffic เมื่อ Not Ready
          readinessProbe:
            exec:
              command:
                - cat
                - /tmp/ready
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 2     # ต้อง Ready 2 ครั้งต่อกัน
            failureThreshold: 3     # Not Ready หลัง 15 วินาที
          
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
  name: production-all-probes
  namespace: startup-workshop
spec:
  selector:
    app: production-all-probes
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
# Apply
kubectl apply -f production-all-probes.yaml

# Watch Pods - ดู Startup Sequence
kubectl get pods -n startup-workshop -l app=production-all-probes -w

# ดู Probe Details
kubectl describe pod -l app=production-all-probes -n startup-workshop | \
  grep -A 8 -E "Startup|Liveness|Readiness"

# ดู Events
kubectl get events -n startup-workshop --sort-by='.lastTimestamp' | tail -20
```

### ขั้นตอนที่ 9: Startup Probe Best Practices Demo

```bash
#!/bin/bash
# startup-probe-calculator.sh
# Calculate appropriate probe settings for your application

echo "=== Startup Probe Configuration Calculator ==="
echo ""

read -p "What is the maximum expected startup time (seconds)? " MAX_STARTUP
read -p "What is the typical startup time (seconds)? " TYPICAL_STARTUP
read -p "Is startup time consistent (yes/no)? " CONSISTENT

echo ""
echo "=== Recommendations ==="

if [ "$CONSISTENT" = "yes" ]; then
    # Consistent startup time
    PERIOD=10
    FAILURE_THRESHOLD=$(( (MAX_STARTUP + PERIOD - 1) / PERIOD ))
    INITIAL_DELAY=$(( TYPICAL_STARTUP - 10 ))
    [ $INITIAL_DELAY -lt 0 ] && INITIAL_DELAY=0
    
    echo "Since startup is consistent:"
    echo ""
    echo "startupProbe:"
    echo "  httpGet:"
    echo "    path: /health"
    echo "    port: 8080"
    echo "  failureThreshold: $FAILURE_THRESHOLD"
    echo "  periodSeconds: $PERIOD"
    echo ""
    echo "livenessProbe:"
    echo "  httpGet:"
    echo "    path: /health"
    echo "    port: 8080"
    echo "  initialDelaySeconds: 0"
    echo "  periodSeconds: 20"
    echo "  failureThreshold: 3"
else
    # Variable startup time
    PERIOD=5
    FAILURE_THRESHOLD=$(( (MAX_STARTUP + PERIOD - 1) / PERIOD ))
    
    echo "Since startup varies:"
    echo ""
    echo "startupProbe:"
    echo "  httpGet:"
    echo "    path: /health"
    echo "    port: 8080"
    echo "  failureThreshold: $FAILURE_THRESHOLD"
    echo "  periodSeconds: $PERIOD"
    echo "  timeoutSeconds: 5"
    echo ""
    echo "livenessProbe:"
    echo "  httpGet:"
    echo "    path: /health"
    echo "    port: 8080"
    echo "  initialDelaySeconds: 0"
    echo "  periodSeconds: 10"
    echo "  failureThreshold: 3"
fi

echo ""
echo "Note: Startup Probe runs BEFORE Liveness and Readiness"
echo "Maximum wait time: $((FAILURE_THRESHOLD * PERIOD)) seconds"
```

### ขั้นตอนที่ 10: Monitor Probe Performance

```bash
# Script สำหรับ Monitor Probe Events
monitor_probe_events() {
    local namespace="${1:-startup-workshop}"
    local pod_filter="${2:-}"
    
    echo "=== Monitoring Probe Events ==="
    echo "Namespace: $namespace"
    echo "Filter: ${pod_filter:-All pods}"
    echo ""
    
    if [ -n "$pod_filter" ]; then
        kubectl get events -n "$namespace" \
            --field-selector "involvedObject.name=$pod_filter" \
            --sort-by='.lastTimestamp' -w
    else
        kubectl get events -n "$namespace" \
            --sort-by='.lastTimestamp' \
            --field-selector 'reason in (Unhealthy,BackOff,Started,Killing)' -w
    fi
}

# ดู All Pods Summary
kubectl get pods -n startup-workshop \
  -o custom-columns=\
'NAME:.metadata.name,'\
'READY:.status.containerStatuses[0].ready,'\
'STATUS:.status.phase,'\
'RESTARTS:.status.containerStatuses[0].restartCount'
```

### ขั้นตอนที่ 11: Advanced Startup Pattern กับ Init Containers

```yaml
# advanced-startup-with-init.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advanced-startup
  namespace: startup-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: advanced-startup
  template:
    metadata:
      labels:
        app: advanced-startup
    spec:
      # Init Containers รันก่อน Main Container
      initContainers:
        # Init 1: รอ Database พร้อม
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Waiting for database..."
              until nc -z db-service 5432; do
                echo "Database not ready, retrying..."
                sleep 3
              done
              echo "Database is ready!"
          resources:
            requests:
              memory: "16Mi"
              cpu: "5m"
        
        # Init 2: Run migrations
        - name: run-migrations
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Running database migrations..."
              sleep 5
              echo "Migrations complete!"
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
      
      # Main Container
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Main application starting..."
              sleep 10  # Load application data
              touch /tmp/ready
              echo "Application ready!"
              
              while true; do
                printf 'HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\n\r\nOK\n' | \
                  nc -l -p 8080 -q 1 2>/dev/null
              done
          ports:
            - containerPort: 8080
          
          # ใช้ Startup Probe สำหรับ Main App Startup
          startupProbe:
            exec:
              command: ["cat", "/tmp/ready"]
            failureThreshold: 12  # 60 วินาที
            periodSeconds: 5
          
          livenessProbe:
            exec:
              command: ["cat", "/tmp/ready"]
            periodSeconds: 15
            failureThreshold: 3
          
          readinessProbe:
            exec:
              command: ["cat", "/tmp/ready"]
            periodSeconds: 5
            failureThreshold: 3
          
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

### ขั้นตอนที่ 12: ทำความสะอาด

```bash
# ลบ Resources
kubectl delete namespace startup-workshop

# Reset Default Namespace
kubectl config set-context --current --namespace=default

echo "Workshop เสร็จสิ้น! ล้างข้อมูลแล้ว"
```

---

## สรุป Probe Comparison

### เมื่อไหรควรใช้แต่ละ Probe

```yaml
# Template สมบูรณ์สำหรับ Production
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          
          # ใช้ Startup Probe เมื่อ:
          # - Application ใช้เวลา Start > 30 วินาที
          # - Startup Time ไม่แน่นอน
          # - ต้องการแยก Startup Phase ออกจาก Running Phase
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30  # คำนวณ: max_startup_seconds / periodSeconds
            periodSeconds: 10
          
          # ใช้ Liveness Probe เสมอ เพื่อ:
          # - ตรวจสอบว่า Application ไม่ Deadlock
          # - Auto-restart เมื่อ Application Stuck
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 20
            failureThreshold: 3
          
          # ใช้ Readiness Probe เสมอ เพื่อ:
          # - Zero-downtime Deployments
          # - ไม่ส่ง Traffic เมื่อ Dependencies ไม่พร้อม
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
```

### Decision Tree

```
Application Start > 30s?
├── YES -> ใช้ Startup Probe
│         failureThreshold = max_startup_seconds / periodSeconds
│
└── NO  -> ใช้ initialDelaySeconds บน Liveness Probe
          initialDelaySeconds = typical_startup_seconds + buffer

Application อาจ Stuck/Deadlock?
└── YES -> ใช้ Liveness Probe (เสมอแนะนำ)

Application มี Dependencies (DB, Cache)?
└── YES -> ใช้ Readiness Probe ตรวจสอบ Dependencies
          (ต่างจาก Liveness ที่ไม่ควรตรวจสอบ Dependencies)
```

---

## Tips สุดท้าย

### 1. Probe Endpoint Best Practices

```
/healthz หรือ /livez   -> Liveness Check (ไม่ตรวจสอบ External Dependencies)
/readyz               -> Readiness Check (ตรวจสอบ Dependencies)
/startupz             -> Optional: สำหรับ Startup Probe เฉพาะ
```

### 2. ตั้งค่า Resources ให้ Probe ทำงานได้

```yaml
# Container ต้องมี Resources พอสำหรับ Probe
# Probe Execution ต้องการ CPU/Memory เพิ่มเติม
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "200m"
# ถ้า Resources ไม่พอ Probe อาจ Timeout
```

### 3. Testing Probes ก่อน Deploy

```bash
# Test Probe endpoint ก่อน Deploy
kubectl exec my-pod -- curl -s http://localhost:8080/healthz
kubectl exec my-pod -- nc -z localhost 5432 && echo "Port open" || echo "Port closed"
kubectl exec my-pod -- cat /tmp/ready
```

---

## สรุปทั้ง Series (Part 61-70)

ในส่วนของ Monitoring & Observability เราได้เรียนรู้:

| Part | หัวข้อ | สาระสำคัญ |
|------|--------|----------|
| 61 | Kubernetes Events | ติดตามสิ่งที่เกิดขึ้นใน Cluster |
| 62 | kubectl logs | ดู Container Logs อย่างมีประสิทธิภาพ |
| 63 | EFK Stack | Centralized Logging System |
| 64 | Prometheus | Metrics Collection และ PromQL |
| 65 | Grafana | Visualization Dashboard |
| 66 | AlertManager | Production Alerting |
| 67 | Jaeger | Distributed Tracing |
| 68 | Liveness Probe | Auto-restart Container เมื่อ Stuck |
| 69 | Readiness Probe | Zero-downtime Deployment |
| 70 | Startup Probe | Handle Slow-starting Applications |

**ทั้ง 3 Probes ทำงานร่วมกัน:**
- **Startup**: รอ Application Start
- **Liveness**: ตรวจสอบ Application ยัง Alive
- **Readiness**: ตรวจสอบพร้อมรับ Traffic

การ Observe Kubernetes Cluster ที่ดีต้องใช้ทั้ง:
- **Metrics** (Prometheus + Grafana)
- **Logs** (EFK Stack)
- **Traces** (Jaeger)
- **Events** (kubectl events)
- **Probes** (Health Checks)
