# Part 68: Liveness Probes

## Liveness Probe คืออะไร

Liveness Probe เป็นกลไกที่ Kubernetes ใช้ตรวจสอบว่า Container ยังทำงานอยู่หรือไม่ ถ้า Liveness Probe Fail Kubernetes จะ Restart Container นั้น

### ปัญหาที่ Liveness Probe แก้

บางครั้ง Application อาจอยู่ใน State ที่ "Stuck" หรือ "Deadlock" คือยังทำงานอยู่แต่ไม่สามารถ Serve Requests ได้ ถ้าไม่มี Liveness Probe Kubernetes จะไม่รู้ว่าต้อง Restart Container

```
ตัวอย่าง Dead Application:
- Application กำลัง Run แต่ทุก Request Return 500
- Application Deadlock และไม่ตอบสนอง
- Memory Leak ทำให้ Application ช้ามาก
- Database Connection Pool หมด

ถ้าไม่มี Liveness Probe:
- Container ยังทำงานอยู่ (จาก Kubernetes มองเห็น)
- แต่ Application ไม่ทำงาน
- ต้องรอ Operator Restart ด้วยตนเอง

ด้วย Liveness Probe:
- Kubernetes ตรวจสอบ Health ทุกๆ N วินาที
- ถ้า Probe Fail Kubernetes จะ Restart Container อัตโนมัติ
```

### เปรียบเทียบกับ Restart Policy

```yaml
# Restart Policy เป็นการ Restart เมื่อ Container Exit
restartPolicy: Always  # Restart ทุกครั้งที่ Container Exit

# Liveness Probe เป็นการ Restart เมื่อ Container ยัง Run แต่ไม่ Healthy
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

---

## HTTP, TCP, Exec Probes

Kubernetes รองรับ 3 ประเภทของ Probes:

### 1. HTTP GET Probe

ส่ง HTTP GET Request ไปยัง Container ถ้า Response Code อยู่ระหว่าง 200-399 ถือว่าสำเร็จ

```yaml
# http-liveness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: http-liveness-demo
  namespace: default
  labels:
    app: http-liveness
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /        # Path ที่จะ Check
          port: 80       # Port ที่จะ Check
          scheme: HTTP   # HTTP หรือ HTTPS
          httpHeaders:   # Optional Custom Headers
            - name: X-Health-Check
              value: "true"
        initialDelaySeconds: 15  # รอก่อน Probe ครั้งแรก
        periodSeconds: 20        # Check ทุก 20 วินาที
        timeoutSeconds: 5        # Timeout สำหรับแต่ละ Probe
        successThreshold: 1      # จำนวน Success ที่ต้องการ
        failureThreshold: 3      # จำนวน Failure ก่อน Restart
      resources:
        requests:
          memory: "64Mi"
          cpu: "50m"
        limits:
          memory: "128Mi"
          cpu: "100m"
```

### 2. TCP Socket Probe

ทดสอบว่า TCP Connection ไปยัง Port ที่กำหนดสำเร็จหรือไม่

```yaml
# tcp-liveness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tcp-liveness-demo
  namespace: default
  labels:
    app: tcp-liveness
spec:
  containers:
    - name: redis
      image: redis:7.2
      ports:
        - containerPort: 6379
      livenessProbe:
        tcpSocket:
          port: 6379  # ทดสอบว่า Redis Port เปิดอยู่
        initialDelaySeconds: 10
        periodSeconds: 15
        timeoutSeconds: 5
        failureThreshold: 3
      resources:
        requests:
          memory: "64Mi"
          cpu: "50m"
        limits:
          memory: "256Mi"
          cpu: "200m"
```

### 3. Exec Command Probe

รัน Command ภายใน Container ถ้า Exit Code เป็น 0 ถือว่าสำเร็จ

```yaml
# exec-liveness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: exec-liveness-demo
  namespace: default
  labels:
    app: exec-liveness
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ['sh', '-c', 'touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600']
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy  # ตรวจสอบว่า File มีอยู่
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
        limits:
          memory: "64Mi"
          cpu: "50m"
```

### 4. gRPC Probe (Kubernetes 1.24+)

```yaml
# grpc-liveness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: grpc-liveness-demo
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 50051
      livenessProbe:
        grpc:
          port: 50051
          service: "my.service.Health"  # Optional gRPC Health Service
        initialDelaySeconds: 10
        periodSeconds: 15
```

---

## Probe Configuration

### Parameters ที่สำคัญ

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  
  # initialDelaySeconds: รอ N วินาทีหลัง Container Start ก่อน Probe ครั้งแรก
  # ค่า Default: 0 วินาที
  # ควรตั้งให้มากกว่าเวลาที่ Application ต้องการใน Startup
  initialDelaySeconds: 30
  
  # periodSeconds: ทำ Probe ทุก N วินาที
  # ค่า Default: 10 วินาที, Minimum: 1 วินาที
  periodSeconds: 10
  
  # timeoutSeconds: Timeout สำหรับ Probe แต่ละครั้ง
  # ค่า Default: 1 วินาที, Minimum: 1 วินาที
  timeoutSeconds: 5
  
  # successThreshold: จำนวน Consecutive Success ที่ต้องการ
  # ค่า Default: 1, Minimum: 1
  # สำหรับ Liveness Probe ต้องเป็น 1 เสมอ
  successThreshold: 1
  
  # failureThreshold: จำนวน Consecutive Failure ก่อน Kubernetes จะ Act
  # ค่า Default: 3, Minimum: 1
  # สำหรับ Liveness: Restart Container
  failureThreshold: 3
  
  # terminationGracePeriodSeconds: เวลาให้ Container Shutdown อย่าง Graceful
  # ค่า Default: Pod's terminationGracePeriodSeconds
  terminationGracePeriodSeconds: 30
```

### เข้าใจ Probe Timing

```
Container Start
     |
     | initialDelaySeconds (รอ Application เริ่มทำงาน)
     |
     v
First Probe
     |
     | periodSeconds (ตรวจสอบซ้ำๆ)
     |
     v
Second Probe
     |
     v
... (ซ้ำๆ)
     |
     v
ถ้า Fail failureThreshold ครั้ง -> Restart Container
```

---

## Best Practices

### 1. ตั้ง initialDelaySeconds ให้เหมาะสม

```yaml
# ไม่ดี: initialDelaySeconds น้อยเกินไป
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5  # Application อาจยัง Start ไม่เสร็จ

# ดี: initialDelaySeconds มากพอที่ Application จะ Start
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # ให้เวลา Application Start 30 วินาที
  
# ดีกว่า: ใช้ Startup Probe แทน initialDelaySeconds
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
# ต่อจาก Startup Probe
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
```

### 2. Health Endpoint ควร Lightweight

```python
# ไม่ดี: Health Endpoint ทำ DB Query
@app.route('/health')
def health():
    # ทำ Database Query - ช้าและอาจ Fail
    result = db.execute("SELECT 1").fetchone()
    if result:
        return jsonify({"status": "ok"})
    return jsonify({"status": "error"}), 503

# ดี: Health Endpoint ตรวจสอบแค่ Process เป็นหลัก
@app.route('/health/live')  # Liveness
def health_live():
    # ตรวจสอบแค่ว่า Application ยังทำงานอยู่
    return jsonify({"status": "ok"})

@app.route('/health/ready')  # Readiness
def health_ready():
    # ตรวจสอบว่าพร้อมรับ Traffic
    try:
        db.execute("SELECT 1")
        return jsonify({"status": "ok"})
    except Exception as e:
        return jsonify({"status": "error", "reason": str(e)}), 503
```

### 3. Probe Values ที่เหมาะสม

```yaml
# สำหรับ Web Application ทั่วไป
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30    # ให้เวลา Start
  periodSeconds: 10          # ตรวจสอบทุก 10 วินาที
  timeoutSeconds: 5          # Timeout 5 วินาที
  failureThreshold: 3        # 3 ครั้งก่อน Restart (30 วินาทีรวม)

# สำหรับ Slow-starting Application
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 120   # รอ Application Start 2 นาที
  periodSeconds: 20
  timeoutSeconds: 5
  failureThreshold: 3

# สำหรับ Critical Service ที่ต้อง Restart เร็ว
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5           # ตรวจสอบทุก 5 วินาที
  timeoutSeconds: 3
  failureThreshold: 2        # Restart หลัง 2 ครั้ง (10 วินาที)
```

### 4. อย่าใช้ Liveness Probe สำหรับ Dependency Checks

```yaml
# ไม่ดี: Liveness Probe ตรวจสอบ External Dependencies
livenessProbe:
  httpGet:
    path: /health/with-db-check  # ถ้า DB Down -> Restart Container ซ้ำๆ

# ดี: แยก Liveness และ Readiness
livenessProbe:
  httpGet:
    path: /health/live  # แค่ตรวจสอบ Process
readinessProbe:
  httpGet:
    path: /health/ready  # ตรวจสอบรวมถึง Dependencies
```

---

## Workshop: Configure Liveness Probes

### เป้าหมาย

ในส่วนนี้เราจะ:
1. ทดสอบ Liveness Probe ประเภทต่างๆ
2. จำลองสถานการณ์ที่ Probe Fail
3. ดูพฤติกรรมของ Kubernetes เมื่อ Probe Fail
4. ปรับแต่ง Probe Configuration

### ขั้นตอนที่ 1: สร้าง Namespace

```bash
kubectl create namespace probe-workshop
kubectl config set-context --current --namespace=probe-workshop
```

### ขั้นตอนที่ 2: HTTP Liveness Probe Basic

```yaml
# basic-http-liveness.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-liveness-app
  namespace: probe-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: http-liveness
  template:
    metadata:
      labels:
        app: http-liveness
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
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
```

```bash
# Apply
kubectl apply -f basic-http-liveness.yaml

# ดู Pods
kubectl get pods -n probe-workshop -w &

# ดู Details ของ Probe
kubectl describe pod -l app=http-liveness -n probe-workshop | \
  grep -A 10 "Liveness:"
```

### ขั้นตอนที่ 3: จำลอง Liveness Probe Failure

```yaml
# failing-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-failure-demo
  namespace: probe-workshop
  labels:
    app: liveness-failure
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Starting healthy period..."
          touch /tmp/healthy
          # ทำงานปกติ 60 วินาที
          sleep 60
          # จากนั้นทำ Unhealthy
          echo "Going unhealthy..."
          rm -f /tmp/healthy
          # Container ยังทำงานอยู่แต่ Probe จะ Fail
          sleep 3600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
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
# Apply failing pod
kubectl apply -f failing-liveness.yaml

# Watch Pod Status
kubectl get pod liveness-failure-demo -n probe-workshop -w &

# ดู Events
kubectl get events -n probe-workshop --field-selector involvedObject.name=liveness-failure-demo -w
```

Expected Behavior:
```
NAME                    READY   STATUS    RESTARTS   AGE
liveness-failure-demo   1/1     Running   0          0s
liveness-failure-demo   1/1     Running   0          1m
liveness-failure-demo   1/1     Running   1          2m  <- Restart!
liveness-failure-demo   1/1     Running   2          3m  <- อีกครั้ง!
```

### ขั้นตอนที่ 4: HTTP Custom Health Endpoint

```yaml
# custom-health-endpoint.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-health-app
  namespace: probe-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-health
  template:
    metadata:
      labels:
        app: custom-health
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # สร้าง HTTP Server แบบง่ายที่มี Health Endpoint
              # ใช้ nc (netcat) สร้าง HTTP Server
              HEALTHY=true
              
              # Background process เพื่อเปลี่ยน Health State
              (
                sleep 120
                echo "Setting unhealthy..."
                HEALTHY=false
                touch /tmp/unhealthy
              ) &
              
              # HTTP Server
              while true; do
                if [ -f /tmp/unhealthy ]; then
                  # ตอบ 503
                  printf 'HTTP/1.0 503 Service Unavailable\r\nContent-Type: text/plain\r\n\r\nUnhealthy\n' | \
                    nc -l -p 8080 -q 1
                else
                  # ตอบ 200
                  printf 'HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\n\r\nHealthy\n' | \
                    nc -l -p 8080 -q 1
                fi
              done
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              memory: "16Mi"
              cpu: "5m"
            limits:
              memory: "32Mi"
              cpu: "20m"
```

### ขั้นตอนที่ 5: TCP Liveness Probe

```yaml
# tcp-liveness-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tcp-liveness-test
  namespace: probe-workshop
  labels:
    app: tcp-liveness-test
spec:
  containers:
    - name: redis
      image: redis:7.2-alpine
      ports:
        - containerPort: 6379
      livenessProbe:
        tcpSocket:
          port: 6379
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
        limits:
          memory: "128Mi"
          cpu: "100m"
```

```bash
# Apply TCP Probe
kubectl apply -f tcp-liveness-test.yaml

# ดู Probe Details
kubectl describe pod tcp-liveness-test -n probe-workshop

# ตรวจสอบ Redis ทำงาน
kubectl exec tcp-liveness-test -n probe-workshop -- redis-cli PING
```

### ขั้นตอนที่ 6: สร้าง Application พร้อม Health Endpoint ที่ดี

```yaml
# good-health-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: well-designed-health
  namespace: probe-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: well-designed-health
  template:
    metadata:
      labels:
        app: well-designed-health
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # State Management
              echo "alive" > /tmp/liveness
              echo "ready" > /tmp/readiness
              
              # Simulate application work
              (
                while true; do
                  echo "$(date) Processing request..." >> /tmp/app.log
                  sleep 5
                done
              ) &
              
              # Health Server
              while true; do
                # Liveness Check - ตรวจสอบแค่ว่า Process ยังทำงานอยู่
                LIVE_STATUS=$(cat /tmp/liveness 2>/dev/null)
                
                if [ "$LIVE_STATUS" = "alive" ]; then
                  printf 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n{"status":"alive","uptime":"'"$SECONDS"'"}\n' | \
                    nc -l -p 8080 -q 1 2>/dev/null
                else
                  printf 'HTTP/1.0 503 Service Unavailable\r\nContent-Type: application/json\r\n\r\n{"status":"dead"}\n' | \
                    nc -l -p 8080 -q 1 2>/dev/null
                fi
              done
          ports:
            - containerPort: 8080
          # Liveness: ตรวจสอบ Process
          livenessProbe:
            httpGet:
              path: /livez
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          # Readiness: ตรวจสอบว่าพร้อมรับ Traffic
          readinessProbe:
            exec:
              command:
                - cat
                - /tmp/readiness
            initialDelaySeconds: 5
            periodSeconds: 5
          # Startup: รอ Application Start
          startupProbe:
            exec:
              command:
                - cat
                - /tmp/liveness
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 12  # รอสูงสุด 60 วินาที
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

### ขั้นตอนที่ 7: ดู Liveness Probe Events

```bash
# Script สำหรับ Monitor Probe Events
watch_probe_events() {
    local namespace="${1:-probe-workshop}"
    
    echo "=== Watching Probe Events in $namespace ==="
    kubectl get events -n "$namespace" \
        --field-selector reason=Unhealthy \
        -w --sort-by='.lastTimestamp'
}

# รัน
watch_probe_events probe-workshop &

# ดู Restart Count ของ Pods
watch -n 5 'kubectl get pods -n probe-workshop -o wide'
```

### ขั้นตอนที่ 8: Probe Debugging

```bash
# Debug ว่าทำไม Liveness Probe Fail

# 1. ดู Events
kubectl describe pod <pod-name> -n probe-workshop | grep -A 20 Events

# 2. ดู Container Logs
kubectl logs <pod-name> -n probe-workshop --previous

# 3. ทดสอบ Probe ด้วยตนเอง (HTTP)
kubectl exec <pod-name> -n probe-workshop -- \
    wget -qO- http://localhost:8080/health

# 4. ทดสอบ Probe ด้วยตนเอง (TCP)
kubectl exec <pod-name> -n probe-workshop -- \
    sh -c "nc -z localhost 6379 && echo 'Port open' || echo 'Port closed'"

# 5. ดู Container Resource Usage
kubectl top pod -n probe-workshop

# 6. ดู Container Status
kubectl get pod <pod-name> -n probe-workshop \
    -o jsonpath='{.status.containerStatuses[*]}' | python3 -m json.tool
```

### ขั้นตอนที่ 9: จำลองสถานการณ์ที่ซับซ้อน

```yaml
# complex-liveness-scenario.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-health-demo
  namespace: probe-workshop
  annotations:
    description: "Application that gets stuck after processing too many requests"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: complex-health
  template:
    metadata:
      labels:
        app: complex-health
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              REQUEST_COUNT=0
              MAX_REQUESTS=30
              STUCK=false
              
              echo "Application starting..."
              
              # HTTP Server
              while true; do
                REQUEST_COUNT=$((REQUEST_COUNT + 1))
                
                # จำลอง Application ที่ Stuck หลัง Process Requests เยอะเกินไป
                if [ "$REQUEST_COUNT" -gt "$MAX_REQUESTS" ] && [ "$STUCK" = "false" ]; then
                  echo "Application is stuck after $MAX_REQUESTS requests!"
                  STUCK=true
                fi
                
                if [ "$STUCK" = "true" ]; then
                  # ไม่ตอบสนอง
                  sleep 30 &
                  wait
                else
                  # ตอบสนองปกติ
                  printf 'HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\n\r\nOK (requests: '"$REQUEST_COUNT"')\n' | \
                    nc -l -p 8080 -q 1 2>/dev/null
                fi
              done
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3    # จะ Timeout เมื่อ Application Stuck
            failureThreshold: 3  # Restart หลัง 3 ครั้ง (15 วินาที)
          resources:
            requests:
              memory: "16Mi"
              cpu: "5m"
            limits:
              memory: "32Mi"
              cpu: "20m"
```

```bash
# Apply และ Watch
kubectl apply -f complex-liveness-scenario.yaml

# Watch ว่า Kubernetes Restart Container เมื่อ Application Stuck
kubectl get pod -l app=complex-health -n probe-workshop -w

# ดู Restart Count เพิ่มขึ้น
kubectl get pod -l app=complex-health -n probe-workshop \
  -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}'
```

### ขั้นตอนที่ 10: Liveness Probe Resource Impact

```bash
# ดู Resource Usage ของ Probe
# Probe ทำให้เกิด HTTP Requests ไปยัง Container
# ควรเลือก periodSeconds ที่เหมาะสม

# Script วัด Overhead ของ Probe
cat > /tmp/measure-probe-overhead.sh << 'MEASURE_EOF'
#!/bin/bash
NAMESPACE="${1:-probe-workshop}"
POD="${2}"

if [ -z "$POD" ]; then
    POD=$(kubectl get pods -n "$NAMESPACE" -o jsonpath='{.items[0].metadata.name}')
fi

echo "Measuring probe overhead for pod: $POD"
echo ""

# ดู Probe Configuration
echo "=== Probe Configuration ==="
kubectl describe pod "$POD" -n "$NAMESPACE" | grep -A 15 "Liveness:"

echo ""
echo "=== Container Metrics ==="
kubectl top pod "$POD" -n "$NAMESPACE"
MEASURE_EOF

chmod +x /tmp/measure-probe-overhead.sh
```

### ขั้นตอนที่ 11: สร้าง Flask Application พร้อม Proper Health Checks

```yaml
# flask-app-with-health.yaml
# ใช้ python:3.11-slim Image พร้อม Inline Script
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-health-demo
  namespace: probe-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-health
  template:
    metadata:
      labels:
        app: flask-health
    spec:
      containers:
        - name: app
          image: python:3.11-slim
          command:
            - python3
            - -c
            - |
              from http.server import HTTPServer, BaseHTTPRequestHandler
              import json
              import time
              import threading
              
              start_time = time.time()
              is_healthy = True
              
              class HealthHandler(BaseHTTPRequestHandler):
                  def log_message(self, format, *args):
                      pass  # Suppress access logs
                  
                  def do_GET(self):
                      if self.path == '/healthz' or self.path == '/livez':
                          # Liveness Check
                          if is_healthy:
                              self.send_response(200)
                              self.send_header('Content-Type', 'application/json')
                              self.end_headers()
                              response = {
                                  'status': 'ok',
                                  'uptime': int(time.time() - start_time)
                              }
                              self.wfile.write(json.dumps(response).encode())
                          else:
                              self.send_response(503)
                              self.end_headers()
                              self.wfile.write(b'{"status":"unhealthy"}')
                      elif self.path == '/readyz':
                          # Readiness Check
                          self.send_response(200)
                          self.send_header('Content-Type', 'application/json')
                          self.end_headers()
                          self.wfile.write(b'{"status":"ready"}')
                      else:
                          self.send_response(200)
                          self.send_header('Content-Type', 'application/json')
                          self.end_headers()
                          self.wfile.write(b'{"service":"flask-health-demo"}')
              
              server = HTTPServer(('0.0.0.0', 8080), HealthHandler)
              print('Server running on port 8080')
              server.serve_forever()
          ports:
            - containerPort: 8080
              name: http
          # Liveness: ตรวจสอบ Process Alive
          livenessProbe:
            httpGet:
              path: /livez
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          # Readiness: ตรวจสอบพร้อมรับ Traffic
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: flask-health-demo
  namespace: probe-workshop
spec:
  selector:
    app: flask-health
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
# Apply Flask App
kubectl apply -f flask-app-with-health.yaml

# รอ Pods Ready
kubectl wait --for=condition=Ready pod -l app=flask-health \
  -n probe-workshop --timeout=60s

# ทดสอบ Health Endpoints
kubectl port-forward -n probe-workshop svc/flask-health-demo 8080:8080 &
sleep 2

curl http://localhost:8080/livez
curl http://localhost:8080/readyz

kill %% 2>/dev/null
```

### ขั้นตอนที่ 12: ทำความสะอาด

```bash
# ลบ Resources
kubectl delete namespace probe-workshop

# Reset Default Namespace
kubectl config set-context --current --namespace=default

echo "Cleanup เสร็จสิ้น"
```

---

## Troubleshooting Liveness Probe Issues

### Common Problems

```bash
# Problem 1: "Liveness probe failed: HTTP probe failed with statuscode: 503"
# สาเหตุ: Application Return Error Code
# แก้ไข: ดู Application Logs และแก้ Application หรือปรับ failureThreshold

# Problem 2: "Back-off restarting failed container"
# สาเหตุ: Container Restart Loop
# แก้ไข: เพิ่ม initialDelaySeconds

# Problem 3: "Liveness probe failed: timeout: dial tcp"
# สาเหตุ: Port ไม่ Open หรือ Application ช้า
# แก้ไข: เพิ่ม timeoutSeconds หรือตรวจสอบ Port

# Debug Commands:
kubectl describe pod <pod-name>  # ดู Events
kubectl logs <pod-name> --previous  # ดู Logs ก่อน Restart
kubectl get events --sort-by='.lastTimestamp'  # ดู Events ล่าสุด
```

---

## สรุป

Liveness Probe เป็นกลไกสำคัญที่ทำให้ Kubernetes สามารถ Self-heal ได้:

1. **HTTP Probe**: เหมาะสำหรับ Web Applications
2. **TCP Probe**: เหมาะสำหรับ Services ที่ใช้ TCP (Redis, MySQL)
3. **Exec Probe**: Flexible สำหรับ Custom Health Checks
4. **Configuration**: ตั้ง initialDelaySeconds, periodSeconds, failureThreshold ให้เหมาะสม
5. **Best Practices**: Probe ควร Lightweight และไม่ตรวจสอบ External Dependencies

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ Readiness Probe ซึ่งทำงานคล้ายกับ Liveness Probe แต่มีวัตถุประสงค์ที่ต่างกัน
