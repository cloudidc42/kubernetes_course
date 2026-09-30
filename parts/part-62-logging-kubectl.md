# Part 62: Logging ด้วย kubectl

## kubectl logs คำสั่ง

`kubectl logs` เป็นคำสั่งพื้นฐานที่สำคัญที่สุดสำหรับการดู Logs ของ Containers ใน Kubernetes คำสั่งนี้ช่วยให้เราเข้าถึง stdout และ stderr ของ Container ที่กำลังทำงานหรือที่หยุดทำงานแล้ว

### ทำไม Logging ถึงสำคัญใน Kubernetes

ใน Kubernetes Architecture ที่มีหลาย Nodes และหลาย Pods, การดู Logs จาก Container เดียวจำเป็นต้องรู้ว่า Container นั้นอยู่ที่ Node ไหน kubectl logs ช่วยให้เราไม่ต้อง SSH เข้า Node โดยตรง

### Syntax พื้นฐาน

```bash
# ดู Logs ของ Pod
kubectl logs <pod-name>

# ดู Logs ของ Pod ใน Namespace เฉพาะ
kubectl logs <pod-name> -n <namespace>

# ดู Logs ของ Container เฉพาะใน Pod
kubectl logs <pod-name> -c <container-name>

# ดู Logs ของ Pod ที่ Exit แล้ว (Previous)
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -p

# ดู Logs แบบ Follow (Stream)
kubectl logs <pod-name> -f
kubectl logs <pod-name> --follow

# ดู N บรรทัดสุดท้าย
kubectl logs <pod-name> --tail=100

# ดู Logs จาก N นาทีที่แล้ว
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=30m
kubectl logs <pod-name> --since=10s

# ดู Logs ตั้งแต่ Timestamp เฉพาะ
kubectl logs <pod-name> --since-time="2024-01-15T10:00:00Z"

# แสดง Timestamps ใน Log
kubectl logs <pod-name> --timestamps=true
kubectl logs <pod-name> --timestamps
```

### ตัวอย่างการใช้งานจริง

```bash
# ดู Logs ล่าสุด 50 บรรทัดพร้อม Timestamp
kubectl logs nginx-pod --tail=50 --timestamps

# ดู Logs ย้อนหลัง 1 ชั่วโมงแบบ Follow
kubectl logs nginx-pod --since=1h -f

# ดู Logs ของ Container เฉพาะใน Multi-container Pod
kubectl logs my-pod -c sidecar-container --tail=100

# ดู Logs ของ Pod ที่ Crash แล้ว
kubectl logs crashed-pod --previous --tail=200

# ดู Logs และ Save ลง File
kubectl logs nginx-pod --since=24h > /tmp/nginx-logs.txt

# ดู Logs แบบ Verbose พร้อม Timestamps
kubectl logs nginx-pod --timestamps=true --since=1h | less
```

---

## Log Streaming

Log Streaming ช่วยให้เราดู Logs แบบ Real-time คล้ายกับ `tail -f` ใน Linux

### การใช้ -f Flag

```bash
# Stream Logs
kubectl logs -f my-pod

# Stream Logs ของ Container เฉพาะ
kubectl logs -f my-pod -c my-container

# Stream Logs ของ Pod ใน Namespace เฉพาะ
kubectl logs -f my-pod -n production

# Stream และแสดง Timestamps
kubectl logs -f my-pod --timestamps=true

# Stream 50 บรรทัดสุดท้ายก่อน จากนั้น Follow
kubectl logs -f my-pod --tail=50
```

### Streaming Logs จาก Deployment

```bash
# Stream Logs จากทุก Pods ใน Deployment (ต้องใช้ Selector)
kubectl logs -f deployment/my-deployment

# หรือใช้ Label Selector
kubectl logs -f -l app=my-app

# Stream Logs จาก Pods ทั้งหมดที่มี Label
kubectl logs -f -l app=nginx --max-log-requests=10
```

### การใช้ Labels กับ kubectl logs

```bash
# ดู Logs จาก Pods ที่มี Label app=nginx
kubectl logs -l app=nginx

# Stream Logs จาก Pods ที่มี Label
kubectl logs -l app=nginx -f

# ดู Logs จาก Pods ที่มี Multiple Labels
kubectl logs -l "app=nginx,env=production"

# ดู Logs พร้อม Prefix ชื่อ Pod
kubectl logs -l app=nginx --prefix=true
```

---

## Multi-container Logs

เมื่อ Pod มีหลาย Containers เราต้องระบุ Container ที่ต้องการดู Logs

### ดู Containers ใน Pod

```bash
# ดูว่า Pod มี Containers อะไรบ้าง
kubectl describe pod my-pod | grep -A 20 "Containers:"

# หรือใช้ jsonpath
kubectl get pod my-pod -o jsonpath='{.spec.containers[*].name}'

# ดูทุก Containers รวมถึง Init Containers
kubectl get pod my-pod \
  -o jsonpath='{range .spec.initContainers[*]}{.name}{"\n"}{end}'\
  && kubectl get pod my-pod \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
```

### ตัวอย่าง Multi-container Pod

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  namespace: default
  labels:
    app: multi-container
spec:
  initContainers:
    - name: init-db
      image: busybox:1.36
      command: ['sh', '-c', 'echo "Init DB..."; sleep 2; echo "DB Ready"']
    - name: init-config
      image: busybox:1.36
      command: ['sh', '-c', 'echo "Init Config..."; sleep 1; echo "Config Ready"']
  containers:
    - name: web-server
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "50m"
        limits:
          memory: "128Mi"
          cpu: "100m"
    - name: log-collector
      image: busybox:1.36
      command: ['sh', '-c', 'while true; do echo "Log entry: $(date)"; sleep 5; done']
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
        limits:
          memory: "64Mi"
          cpu: "50m"
    - name: metrics-exporter
      image: busybox:1.36
      command: ['sh', '-c', 'while true; do echo "Metrics: cpu=50%, mem=70%"; sleep 10; done']
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
        limits:
          memory: "64Mi"
          cpu: "50m"
```

### ดู Logs จาก Multi-container Pod

```bash
# Apply Multi-container Pod
kubectl apply -f multi-container-pod.yaml

# ดู Logs ของ Init Container
kubectl logs multi-container-pod -c init-db
kubectl logs multi-container-pod -c init-config

# ดู Logs ของ Main Containers
kubectl logs multi-container-pod -c web-server
kubectl logs multi-container-pod -c log-collector
kubectl logs multi-container-pod -c metrics-exporter

# Stream Logs จาก log-collector
kubectl logs -f multi-container-pod -c log-collector

# ดู Logs ของ Previous Init Container
kubectl logs multi-container-pod -c init-db --previous 2>/dev/null || echo "No previous logs"
```

### Script สำหรับดู Logs จากทุก Containers

```bash
#!/bin/bash
# view-all-container-logs.sh

POD_NAME="${1}"
NAMESPACE="${2:-default}"
LINES="${3:-50}"

if [ -z "$POD_NAME" ]; then
    echo "Usage: $0 <pod-name> [namespace] [lines]"
    exit 1
fi

echo "=== Pod: $POD_NAME (Namespace: $NAMESPACE) ==="
echo ""

# ดู Init Containers
INIT_CONTAINERS=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .spec.initContainers[*]}{.name}{"\n"}{end}' 2>/dev/null)

if [ -n "$INIT_CONTAINERS" ]; then
    echo "--- Init Containers ---"
    while IFS= read -r container; do
        echo ""
        echo ">>> Container: $container (Init) <<<"
        kubectl logs "$POD_NAME" -c "$container" -n "$NAMESPACE" --tail="$LINES" 2>/dev/null || \
            echo "  (ไม่มี Logs หรือ Container ยังไม่ได้ Run)"
    done <<< "$INIT_CONTAINERS"
fi

# ดู Main Containers
CONTAINERS=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}' 2>/dev/null)

echo ""
echo "--- Main Containers ---"
while IFS= read -r container; do
    echo ""
    echo ">>> Container: $container <<<"
    kubectl logs "$POD_NAME" -c "$container" -n "$NAMESPACE" --tail="$LINES" 2>/dev/null || \
        echo "  (ไม่มี Logs)"
done <<< "$CONTAINERS"
```

---

## stern tool

`stern` เป็น Tool ที่ช่วยให้การดู Logs จากหลาย Pods และหลาย Containers ง่ายขึ้นมาก โดยสามารถ Stream Logs จากหลาย Pods พร้อมกันได้และแสดงในรูปแบบที่อ่านง่าย

### ติดตั้ง stern

```bash
# macOS
brew install stern

# Linux - ติดตั้งจาก GitHub Releases
STERN_VERSION=$(curl -s https://api.github.com/repos/stern/stern/releases/latest | \
  grep tag_name | cut -d'"' -f4)
curl -L "https://github.com/stern/stern/releases/download/${STERN_VERSION}/stern_linux_amd64.tar.gz" \
  -o /tmp/stern.tar.gz
tar xzvf /tmp/stern.tar.gz -C /tmp
sudo mv /tmp/stern /usr/local/bin/
stern --version

# หรือใช้ Homebrew บน Linux
brew install stern
```

### การใช้งาน stern

```bash
# ดู Logs จาก Pod ที่ชื่อขึ้นต้นด้วย "nginx"
stern nginx

# ดู Logs จาก Pods ที่มี Label app=nginx
stern -l app=nginx

# ดู Logs จาก Namespace เฉพาะ
stern -n production nginx

# ดู Logs จากทุก Namespaces
stern --all-namespaces nginx

# ดู Logs จาก Container เฉพาะ
stern nginx -c web-server

# ดู Logs ย้อนหลัง 1 ชั่วโมง
stern nginx --since 1h

# ดู Logs และ Exclude บาง Lines
stern nginx --exclude "health check"

# ดู Logs เฉพาะที่ Match Pattern
stern nginx --include "ERROR|WARN"

# ดู Logs แบบ JSON และ Format ด้วย jq
stern nginx --output json | jq .

# กำหนดสีของ Output
stern nginx --color always

# ดู Logs จาก Namespace หลายๆ อัน
stern nginx -n production -n staging

# ใช้ Regex Pattern
stern "nginx-.*"

# ดู Logs แบบ Template
stern nginx --template '{{.PodName}} {{.Message}}'

# Stern พร้อม Timestamps
stern nginx --timestamps
```

### stern Output Format

stern แสดง Output ในรูปแบบ:
```
[pod-name][container-name] log message
```

ตัวอย่าง:
```
nginx-abc123[web-server] 10.244.0.1 - - [15/Jan/2024:10:30:00 +0000] "GET / HTTP/1.1" 200
nginx-def456[web-server] 10.244.0.2 - - [15/Jan/2024:10:30:01 +0000] "GET /health HTTP/1.1" 200
nginx-abc123[web-server] 10.244.0.1 - - [15/Jan/2024:10:30:02 +0000] "POST /api HTTP/1.1" 201
```

### stern Configuration File

```yaml
# ~/.config/stern/config.yaml
color: auto
output: default
template: |
  {{color .PodColor .PodName}} {{color .ContainerColor .ContainerName}} {{.Message}}
```

---

## Workshop: Effective Log Analysis

### เป้าหมาย

ในส่วนนี้เราจะ:
1. Deploy Application ที่มี Logging รูปแบบต่างๆ
2. ใช้ kubectl logs อย่างมีประสิทธิภาพ
3. ใช้ stern สำหรับ Multi-pod Logging
4. วิเคราะห์ Logs เพื่อหาปัญหา
5. สร้าง Log Analysis Scripts

### ขั้นตอนที่ 1: เตรียม Application

```yaml
# log-workshop-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: log-workshop
---
# log-generator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: log-workshop
  labels:
    app: log-generator
spec:
  replicas: 3
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                LEVEL=$((RANDOM % 4))
                case $LEVEL in
                  0) echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) INFO  Starting request processing" ;;
                  1) echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) WARN  High memory usage detected: $((RANDOM % 30 + 70))%" ;;
                  2) echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) ERROR Failed to connect to database: connection timeout" ;;
                  3) echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) INFO  Request completed in $((RANDOM % 500 + 10))ms" ;;
                esac
                sleep $((RANDOM % 3 + 1))
              done
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
        - name: sidecar
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) SIDECAR metrics_collected=true"
                sleep 15
              done
          resources:
            requests:
              memory: "16Mi"
              cpu: "5m"
            limits:
              memory: "32Mi"
              cpu: "20m"
```

```bash
# Apply Namespace และ Deployment
kubectl apply -f log-workshop-namespace.yaml
kubectl apply -f log-generator-deployment.yaml

# ตรวจสอบว่า Pods กำลัง Run
kubectl get pods -n log-workshop -w
```

### ขั้นตอนที่ 2: Basic Log Analysis

```bash
# ดู Logs จาก Pod ตัวแรก
POD=$(kubectl get pods -n log-workshop -l app=log-generator \
  -o jsonpath='{.items[0].metadata.name}')

echo "ดู Logs จาก Pod: $POD"

# ดู Logs ล่าสุด 20 บรรทัด
kubectl logs $POD -n log-workshop --tail=20

# ดู Logs จาก Container app เท่านั้น
kubectl logs $POD -c app -n log-workshop --tail=20

# ดู Logs จาก Container sidecar
kubectl logs $POD -c sidecar -n log-workshop --tail=20
```

### ขั้นตอนที่ 3: Filter Logs ด้วย grep

```bash
# ดูเฉพาะ ERROR Logs
kubectl logs -l app=log-generator -n log-workshop | grep ERROR

# ดูเฉพาะ WARN และ ERROR Logs
kubectl logs -l app=log-generator -n log-workshop | grep -E "WARN|ERROR"

# นับ ERROR ใน 5 นาทีที่แล้ว
kubectl logs -l app=log-generator -n log-workshop --since=5m | grep -c ERROR

# ดู Context รอบๆ ERROR (5 บรรทัดก่อนและหลัง)
kubectl logs -l app=log-generator -n log-workshop | grep -C 5 ERROR

# Stream Logs และ Filter
kubectl logs -f -l app=log-generator -n log-workshop | grep --line-buffered ERROR
```

### ขั้นตอนที่ 4: Log Analysis Script

```bash
#!/bin/bash
# analyze-logs.sh

NAMESPACE="${1:-log-workshop}"
SINCE="${2:-1h}"
LABEL="${3:-app=log-generator}"

echo "============================================"
echo "        Log Analysis Report                 "
echo "============================================"
echo "Namespace: $NAMESPACE"
echo "Period: Last $SINCE"
echo "Selector: $LABEL"
echo "Time: $(date)"
echo ""

# รวบรวม Logs
LOGS=$(kubectl logs -l "$LABEL" -n "$NAMESPACE" --since="$SINCE" 2>/dev/null)

if [ -z "$LOGS" ]; then
    echo "ไม่พบ Logs"
    exit 0
fi

# นับจำนวน Log ตาม Level
echo "--- Log Level Summary ---"
echo "INFO:  $(echo "$LOGS" | grep -c " INFO " || echo 0)"
echo "WARN:  $(echo "$LOGS" | grep -c " WARN " || echo 0)"
echo "ERROR: $(echo "$LOGS" | grep -c " ERROR " || echo 0)"

echo ""
echo "--- Top Error Messages ---"
echo "$LOGS" | grep " ERROR " | sort | uniq -c | sort -rn | head -5

echo ""
echo "--- Recent Warnings ---"
echo "$LOGS" | grep " WARN " | tail -5

echo ""
echo "--- Recent Errors ---"
echo "$LOGS" | grep " ERROR " | tail -5
```

```bash
# ทำให้ Script Executable
chmod +x analyze-logs.sh

# รัน Script
./analyze-logs.sh log-workshop 30m app=log-generator
```

### ขั้นตอนที่ 5: ใช้ stern สำหรับ Multi-Pod Logging

```bash
# Install stern ถ้ายังไม่มี
# macOS: brew install stern
# Linux:
curl -L "https://github.com/stern/stern/releases/download/v1.28.0/stern_linux_amd64.tar.gz" \
  -o /tmp/stern.tar.gz && \
  tar xzf /tmp/stern.tar.gz -C /tmp && \
  sudo mv /tmp/stern /usr/local/bin/

# ดู Logs จากทุก Pods ใน log-workshop
stern -n log-workshop log-generator

# ดูเฉพาะ ERROR
stern -n log-workshop log-generator --include ERROR

# ดูเฉพาะ Container app
stern -n log-workshop log-generator -c app

# ดู Logs ย้อนหลัง 10 นาที
stern -n log-workshop log-generator --since 10m

# แสดงแบบ JSON
stern -n log-workshop log-generator --output json

# Exclude Health Check Logs
stern -n log-workshop log-generator --exclude "SIDECAR"
```

### ขั้นตอนที่ 6: Structured Logging Analysis

```yaml
# structured-log-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: structured-logger
  namespace: log-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: structured-logger
  template:
    metadata:
      labels:
        app: structured-logger
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                REQUEST_ID="req-$(cat /dev/urandom | head -c 4 | xxd -p)"
                USER_ID=$((RANDOM % 1000 + 1))
                STATUS=$((RANDOM % 3))
                
                case $STATUS in
                  0)
                    echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"info\",\"request_id\":\"$REQUEST_ID\",\"user_id\":$USER_ID,\"action\":\"login\",\"status\":\"success\"}"
                    ;;
                  1)
                    echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"warn\",\"request_id\":\"$REQUEST_ID\",\"user_id\":$USER_ID,\"action\":\"api_call\",\"latency_ms\":$((RANDOM % 1000 + 500)),\"message\":\"high latency\"}"
                    ;;
                  2)
                    echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"error\",\"request_id\":\"$REQUEST_ID\",\"user_id\":$USER_ID,\"action\":\"payment\",\"error\":\"insufficient_funds\"}"
                    ;;
                esac
                sleep 2
              done
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

```bash
# Apply Structured Logger
kubectl apply -f structured-log-app.yaml

# ดู JSON Logs
kubectl logs -l app=structured-logger -n log-workshop --tail=20

# Parse JSON Logs ด้วย jq
kubectl logs -l app=structured-logger -n log-workshop --tail=50 | \
  grep "^{" | \
  jq -r '[.timestamp, .level, .action, .user_id] | @tsv'

# Filter Error Logs ด้วย jq
kubectl logs -l app=structured-logger -n log-workshop --since=5m | \
  grep "^{" | \
  jq 'select(.level == "error")'

# หา User ที่มี Error มากที่สุด
kubectl logs -l app=structured-logger -n log-workshop --since=1h | \
  grep "^{" | \
  jq -r 'select(.level == "error") | .user_id' | \
  sort | uniq -c | sort -rn | head -5

# วิเคราะห์ Latency
kubectl logs -l app=structured-logger -n log-workshop --since=5m | \
  grep "^{" | \
  jq 'select(.latency_ms != null) | .latency_ms' | \
  awk '{sum+=$1; count++} END {print "Average Latency:", sum/count, "ms"}'
```

### ขั้นตอนที่ 7: Log Rotation และ Size Management

```yaml
# pod-with-log-rotation.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-rotation-demo
  namespace: log-workshop
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "50m"
        limits:
          memory: "128Mi"
          cpu: "100m"
      # Container Log settings ถูกควบคุมโดย Container Runtime
      # สามารถ Configure ได้ใน /etc/docker/daemon.json หรือ containerd config
```

```bash
# ดูขนาดของ Log Files บน Node
# (ต้องมีสิทธิ์เข้าถึง Node)
kubectl debug node/$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') \
  -it --image=busybox -- \
  sh -c "find /host/var/log/pods -name '*.log' -exec ls -lh {} \;"
```

### ขั้นตอนที่ 8: Advanced Log Queries

```bash
#!/bin/bash
# advanced-log-query.sh

# Function สำหรับ Query Logs จาก Multiple Pods
query_logs() {
    local namespace="$1"
    local selector="$2"
    local pattern="$3"
    local since="${4:-1h}"
    
    echo "=== Query: '$pattern' in '$selector' (last $since) ==="
    
    # รับรายชื่อ Pods
    PODS=$(kubectl get pods -n "$namespace" -l "$selector" \
      -o jsonpath='{.items[*].metadata.name}')
    
    for pod in $PODS; do
        echo ""
        echo "--- Pod: $pod ---"
        kubectl logs "$pod" -n "$namespace" --since="$since" 2>/dev/null | \
          grep "$pattern" | \
          head -20
    done
}

# ใช้งาน Function
query_logs "log-workshop" "app=log-generator" "ERROR" "30m"
query_logs "log-workshop" "app=structured-logger" "error" "30m"
```

### ขั้นตอนที่ 9: Real-time Log Monitoring Dashboard

```bash
#!/bin/bash
# log-dashboard.sh

NAMESPACE="${1:-log-workshop}"

# ล้าง Screen
clear

while true; do
    clear
    echo "============================================"
    echo "     Real-time Log Monitoring Dashboard     "
    echo "============================================"
    echo "Namespace: $NAMESPACE"
    echo "Time: $(date)"
    echo ""
    
    # แสดง Pod Status
    echo "--- Pod Status ---"
    kubectl get pods -n "$NAMESPACE" 2>/dev/null
    echo ""
    
    # แสดง Recent Logs
    echo "--- Recent Logs (last 30 seconds) ---"
    kubectl logs -l app=log-generator -n "$NAMESPACE" \
      --since=30s --timestamps 2>/dev/null | tail -20
    echo ""
    
    # แสดง Error Count
    echo "--- Error Count (last 5 min) ---"
    ERROR_COUNT=$(kubectl logs -l app=log-generator -n "$NAMESPACE" \
      --since=5m 2>/dev/null | grep -c ERROR)
    WARN_COUNT=$(kubectl logs -l app=log-generator -n "$NAMESPACE" \
      --since=5m 2>/dev/null | grep -c WARN)
    echo "Errors: $ERROR_COUNT | Warnings: $WARN_COUNT"
    
    sleep 10
done
```

### ขั้นตอนที่ 10: Log Collection Best Practices

```yaml
# best-practices-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: well-logged-app
  namespace: log-workshop
  annotations:
    log-format: "json"
    log-level: "info"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: well-logged-app
  template:
    metadata:
      labels:
        app: well-logged-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Best Practice: ใช้ Structured JSON Logging
              while true; do
                echo "{\"time\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\
\"level\":\"info\",\
\"service\":\"well-logged-app\",\
\"version\":\"1.0.0\",\
\"pod\":\"$HOSTNAME\",\
\"message\":\"Processing request\",\
\"duration_ms\":$((RANDOM % 100 + 10))}"
                sleep 3
              done
          env:
            - name: LOG_LEVEL
              value: "info"
            - name: LOG_FORMAT
              value: "json"
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
          # Best Practice: ใช้ readinessProbe เพื่อ Traffic Routing
          readinessProbe:
            exec:
              command: ["sh", "-c", "exit 0"]
            initialDelaySeconds: 5
            periodSeconds: 10
```

### ขั้นตอนที่ 11: Troubleshooting ด้วย Logs

```bash
# Scenario 1: Pod ที่ Crash
# สร้าง Pod ที่ Crash
kubectl run crashing-app -n log-workshop \
  --image=busybox:1.36 \
  --restart=Never \
  -- sh -c "echo 'Starting...'; sleep 2; echo 'ERROR: Fatal error occurred' >&2; exit 1"

# ดู Logs ของ Pod ที่ Crash
kubectl logs crashing-app -n log-workshop

# ดู Logs ของ Previous Container
kubectl logs crashing-app -n log-workshop --previous

# ดู Status ของ Pod
kubectl describe pod crashing-app -n log-workshop

# Scenario 2: Application ที่มี Log Volume สูง
kubectl run high-volume-logger -n log-workshop \
  --image=busybox:1.36 \
  --restart=Never \
  -- sh -c "while true; do for i in \$(seq 1 100); do echo \"Log line \$i\"; done; sleep 1; done"

# ดู Logs แบบ Rate Limited
kubectl logs high-volume-logger -n log-workshop --tail=100 | head -50

# Scenario 3: หา Error ใน Specific Time Range
kubectl logs -l app=log-generator -n log-workshop \
  --since-time="$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" | \
  grep ERROR
```

### ขั้นตอนที่ 12: stern Advanced Usage

```bash
# ดู Logs จากทุก Pods ที่ Match Pattern
stern -n log-workshop "log-generator|structured-logger" --since 5m

# Filter โดยใช้ Regex บน Log Content
stern -n log-workshop log-generator --include "ERROR.*database"

# แสดง Logs พร้อม Pod Name และ Container Name
stern -n log-workshop log-generator \
  --template '{{.PodName}}/{{.ContainerName}}: {{.Message}}'

# ดู Logs จาก Namespace ทั้งหมด
stern --all-namespaces log-generator

# Save Logs ลง File พร้อม Color Stripped
stern -n log-workshop log-generator --color never > /tmp/stern-output.txt

# ดู Logs ของ Recent Pods ที่ Error
stern -n log-workshop log-generator \
  --include ERROR \
  --since 1h \
  --color always
```

### ขั้นตอนที่ 13: ทำความสะอาด

```bash
# ลบ Resources ที่สร้างใน Workshop
kubectl delete namespace log-workshop

# ตรวจสอบว่าลบเรียบร้อย
kubectl get namespace log-workshop 2>/dev/null || echo "Namespace ถูกลบแล้ว"
```

---

## Tips และ Best Practices

### 1. Structured Logging

```
# ควรใช้ JSON Format สำหรับ Structured Logs
{"time":"2024-01-15T10:30:00Z","level":"info","message":"Request processed","user_id":123,"duration_ms":45}

# แทนที่จะใช้:
[2024-01-15 10:30:00] INFO Request processed for user 123 in 45ms
```

### 2. Log Levels

```bash
# ใช้ Log Levels อย่างเหมาะสม:
# DEBUG  - ข้อมูล Detailed สำหรับ Development
# INFO   - ข้อมูลทั่วไปเกี่ยวกับ Operation
# WARN   - สิ่งที่น่าสนใจแต่ไม่ใช่ Error
# ERROR  - Error ที่ต้องการ Attention
# FATAL  - Error ที่ทำให้ Application หยุดทำงาน
```

### 3. Performance Tips

```bash
# อย่า Stream Logs จาก Production หาก Log Volume สูงมาก
# ใช้ --tail เพื่อ Limit จำนวน Logs ที่ดึงมา
kubectl logs my-pod --tail=100

# ใช้ --since เพื่อดู Logs ใน Time Range เฉพาะ
kubectl logs my-pod --since=10m

# หลีกเลี่ยง kubectl logs -f ใน Production ระยะเวลานาน
# ใช้ Centralized Logging แทน (EFK, Loki)
```

### 4. Troubleshooting Guide

```bash
# ขั้นตอนในการ Debug Pod ที่มีปัญหา:

# Step 1: ดู Status ของ Pod
kubectl get pod my-pod -o wide

# Step 2: ดู Events
kubectl describe pod my-pod | grep -A 20 Events

# Step 3: ดู Current Logs
kubectl logs my-pod --tail=100

# Step 4: ดู Previous Logs (ถ้า Pod Restart)
kubectl logs my-pod --previous --tail=100

# Step 5: Exec เข้า Container
kubectl exec -it my-pod -- sh

# Step 6: ดู Init Container Logs
kubectl logs my-pod -c init-container
```

---

## สรุป

การใช้ `kubectl logs` อย่างมีประสิทธิภาพเป็นทักษะสำคัญสำหรับทุกคนที่ทำงานกับ Kubernetes:

1. **kubectl logs**: เครื่องมือพื้นฐานสำหรับดู Container Logs
2. **Log Streaming**: ใช้ `-f` flag สำหรับดู Logs แบบ Real-time
3. **Multi-container Logs**: ใช้ `-c` flag เพื่อระบุ Container ที่ต้องการ
4. **stern**: Tool ที่ทรงพลังสำหรับ Multi-pod Log Aggregation
5. **Log Analysis**: ใช้ grep, jq เพื่อวิเคราะห์ Logs

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ EFK Stack (Elasticsearch, Fluentd, Kibana) ซึ่งเป็นระบบ Centralized Logging ที่ใช้กันอย่างแพร่หลายใน Production
