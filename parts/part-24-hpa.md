# Part 24: Horizontal Pod Autoscaler (HPA) - Auto-scale ตาม Metrics

## สารบัญ
1. [HPA คืออะไร](#hpa-คืออะไร)
2. [การทำงานของ HPA](#การทำงานของ-hpa)
3. [Metrics Server](#metrics-server)
4. [HPA YAML ละเอียด](#hpa-yaml-ละเอียด)
5. [CPU-based Scaling](#cpu-based-scaling)
6. [Memory-based Scaling](#memory-based-scaling)
7. [Custom Metrics](#custom-metrics)
8. [External Metrics](#external-metrics)
9. [HPA Behavior (Scaling Policies)](#hpa-behavior-scaling-policies)
10. [Workshop: Auto-scale Web App](#workshop-auto-scale-web-app)
11. [Workshop: Scale ด้วย Custom Metrics](#workshop-scale-ด้วย-custom-metrics)
12. [Troubleshooting HPA](#troubleshooting-hpa)
13. [Best Practices](#best-practices)

---

## HPA คืออะไร

**Horizontal Pod Autoscaler (HPA)** คือ Kubernetes resource ที่ปรับจำนวน Pod ของ Deployment, StatefulSet, หรือ ReplicaSet **อัตโนมัติ** ตาม metrics:

- เพิ่ม Pod เมื่อ load สูง (Scale Out)
- ลด Pod เมื่อ load ต่ำ (Scale In)
- รองรับ CPU, Memory, Custom Metrics, External Metrics

### ทำไมต้องใช้ HPA

```
ปัญหา: Traffic ไม่แน่นอน
──────────────────────────────────────────────────────
  Traffic:    Low    →    SPIKE!    →    Low
              │           │              │
  Without HPA: ├── 3 pods ──┤── 3 pods ──┤── 3 pods ──┤
               (under-provision)  (overloaded!)  (over-provision)

  With HPA:   ├── 3 pods ──┤── 10 pods ─┤── 3 pods ──┤
               (efficient)   (handles load) (cost-effective)
```

### ความแตกต่าง HPA vs VPA

| | HPA | VPA |
|--|-----|-----|
| Scale | Horizontal (จำนวน Pods) | Vertical (CPU/Memory ต่อ Pod) |
| เหมาะกับ | Stateless apps | Stateful apps, ต้องการ fine-tune |
| ทำงานร่วมกัน | ได้ (แต่ต้องระวัง) | ได้ (แต่ต้องระวัง) |

---

## การทำงานของ HPA

### Control Loop

```
┌─────────────────────────────────────────────────────────────┐
│                    HPA Control Loop                         │
│                                                             │
│  1. Metrics Server / Custom Metrics API                     │
│     ↓ ดึง metrics ทุก 15 วินาที                             │
│  2. HPA Controller                                          │
│     ↓ คำนวณ desiredReplicas                                 │
│  3. Scale Deployment/StatefulSet                            │
│     ↓ เพิ่ม/ลด Pods                                        │
│  4. รอ cooldown period                                      │
│     ↓ scale up: 3 นาที, scale down: 5 นาที                  │
│  5. กลับไปขั้นตอน 1                                         │
└─────────────────────────────────────────────────────────────┘
```

### สูตรคำนวณ

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))

ตัวอย่าง CPU:
- currentReplicas = 3
- currentCPUUtilization = 80%
- targetCPUUtilization = 50%
- desiredReplicas = ceil(3 × (80/50)) = ceil(4.8) = 5

HPA จะ scale จาก 3 → 5 pods
```

---

## Metrics Server

HPA ต้องการ **Metrics Server** สำหรับ CPU/Memory metrics:

### ติดตั้ง Metrics Server

```bash
# ติดตั้งด้วย kubectl
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# สำหรับ development/minikube (disable TLS verification)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# แก้ไข deployment เพิ่ม args
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'

# หรือใช้ minikube addon
minikube addons enable metrics-server

# ตรวจสอบ
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods
```

### ทดสอบว่า Metrics ทำงาน

```bash
# ดู node metrics
kubectl top nodes

# ดู pod metrics
kubectl top pods -A

# ดู metrics ของ specific pod
kubectl top pod my-pod --containers

# ถ้า error "unable to fully scrape metrics" รอสักครู่แล้วลองใหม่
```

---

## HPA YAML ละเอียด

### Basic HPA (CPU)

```yaml
# basic-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  # Target workload
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  # Min และ Max replicas
  minReplicas: 2
  maxReplicas: 20
  
  # Metrics ที่ใช้ในการ scale
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # target 50% CPU
```

### HPA แบบสมบูรณ์

```yaml
# full-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
  labels:
    app: web-app
    team: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  minReplicas: 2
  maxReplicas: 50
  
  metrics:
  # CPU Utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory Utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # HPA Behavior - ควบคุม scaling speed
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # รอ 60 วิ ก่อน scale up อีกครั้ง
      policies:
      - type: Percent
        value: 100                      # เพิ่มได้สูงสุด 100% ต่อนาที
        periodSeconds: 60
      - type: Pods
        value: 4                        # เพิ่มได้สูงสุด 4 Pods ต่อนาที
        periodSeconds: 60
      selectPolicy: Max                  # ใช้ policy ที่ scale มากกว่า
    
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาที ก่อน scale down
      policies:
      - type: Percent
        value: 10                       # ลดได้สูงสุด 10% ต่อนาที
        periodSeconds: 60
      - type: Pods
        value: 2                        # ลดได้สูงสุด 2 Pods ต่อนาที
        periodSeconds: 60
      selectPolicy: Min                  # ใช้ policy ที่ scale น้อยกว่า (conservative)
```

---

## CPU-based Scaling

### Deployment ที่จะ Scale

```yaml
# web-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 2   # HPA จะควบคุมค่านี้
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80
        
        # Resource requests สำคัญมากสำหรับ HPA!
        # HPA คำนวณ CPU utilization จาก: actual_cpu / requested_cpu
        resources:
          requests:
            cpu: "200m"     # HPA ใช้ค่านี้เป็น 100%
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### HPA สำหรับ CPU

```yaml
# cpu-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-cpu-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### ทดสอบ CPU Scaling

```bash
# Deploy application
kubectl apply -f web-app-deployment.yaml
kubectl apply -f cpu-hpa.yaml

# ดู HPA status
kubectl get hpa -n production

# ดู HPA details
kubectl describe hpa web-app-cpu-hpa -n production

# สร้าง load เพื่อทดสอบ
kubectl run load-generator \
  --image=busybox:1.35 \
  --restart=Never \
  -n production \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://web-app.production.svc.cluster.local; done"

# ดู HPA scaling ใน real-time
kubectl get hpa web-app-cpu-hpa -n production -w

# ดู pods scaling
kubectl get pods -n production -l app=web-app -w

# หยุด load generator
kubectl delete pod load-generator -n production

# ดู scale down (อาจต้องรอ 5+ นาที)
kubectl get hpa web-app-cpu-hpa -n production -w
```

---

## Memory-based Scaling

```yaml
# memory-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-memory-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 200Mi   # ถ้า avg memory > 200Mi → scale up
```

---

## Custom Metrics

Custom metrics ต้องการ **Custom Metrics API** (เช่น Prometheus Adapter):

### ติดตั้ง Prometheus Adapter

```bash
# ติดตั้งด้วย Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://prometheus-server.monitoring.svc.cluster.local \
  --set prometheus.port=80
```

### ConfigMap สำหรับ Custom Metrics

```yaml
# prometheus-adapter-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    # HTTP requests per second
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"
      metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
    
    # Queue length from Redis
    - seriesQuery: 'redis_queue_length{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^redis_queue_length$"
        as: "queue_length"
      metricsQuery: 'avg(<<.Series>>{<<.LabelMatchers>>})'
    
    # Active connections
    - seriesQuery: 'nginx_connections_active{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^nginx_connections_active$"
        as: "active_connections"
      metricsQuery: 'avg(<<.Series>>{<<.LabelMatchers>>})'
```

### HPA ด้วย Custom Metrics

```yaml
# custom-metrics-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-custom-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  
  metrics:
  # CPU + Custom metrics รวมกัน
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Custom metric: requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"   # target 100 req/sec per pod
  
  # Custom metric: queue length
  - type: Pods
    pods:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "30"    # ถ้า queue length > 30 per pod → scale
```

---

## External Metrics

External metrics ใช้สำหรับ metrics จากภายนอก cluster:

```yaml
# external-metrics-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-processor-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-processor
  minReplicas: 1
  maxReplicas: 10
  
  metrics:
  # External metric: SQS queue depth
  - type: External
    external:
      metric:
        name: sqs_approximate_number_of_messages_visible
        selector:
          matchLabels:
            queue: "my-processing-queue"
      target:
        type: AverageValue
        averageValue: "30"   # 1 pod ต่อ 30 messages
```

---

## HPA Behavior (Scaling Policies)

### Scale-up Policies

```yaml
behavior:
  scaleUp:
    # ช่วงเวลาที่รอก่อน scale up (ดู metrics ให้ stable ก่อน)
    stabilizationWindowSeconds: 0   # scale up เร็ว (default: 0)
    
    policies:
    # Policy 1: เพิ่ม 4 pods ทุก 60 วินาที
    - type: Pods
      value: 4
      periodSeconds: 60
    
    # Policy 2: เพิ่ม 100% ทุก 60 วินาที (double)
    - type: Percent
      value: 100
      periodSeconds: 60
    
    # ใช้ policy ที่ scale มากกว่า (aggressive scale-up)
    selectPolicy: Max
```

### Scale-down Policies

```yaml
behavior:
  scaleDown:
    # รอ 5 นาที เพื่อให้ metric stable ก่อน scale down
    stabilizationWindowSeconds: 300
    
    policies:
    # Policy 1: ลด 2 pods ทุก 60 วินาที
    - type: Pods
      value: 2
      periodSeconds: 60
    
    # Policy 2: ลด 10% ทุก 60 วินาที
    - type: Percent
      value: 10
      periodSeconds: 60
    
    # ใช้ policy ที่ scale น้อยกว่า (conservative scale-down)
    selectPolicy: Min
```

### Disable Scale-down (Scale up only)

```yaml
behavior:
  scaleDown:
    selectPolicy: Disabled   # ไม่ scale down เลย
```

---

## Workshop: Auto-scale Web App

เป้าหมาย: Deploy web application ที่ auto-scale ตาม CPU load

### 1. สร้าง Namespace และ Application

```bash
kubectl create namespace hpa-demo
```

### 2. สร้าง Web Application ที่ทำงานหนัก

```yaml
# cpu-intensive-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-app
  namespace: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-app
  template:
    metadata:
      labels:
        app: cpu-app
    spec:
      containers:
      - name: app
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          from http.server import HTTPServer, BaseHTTPRequestHandler
          import time
          import math
          
          class CPUHandler(BaseHTTPRequestHandler):
              def do_GET(self):
                  if self.path == '/':
                      self.send_response(200)
                      self.end_headers()
                      self.wfile.write(b'OK')
                  elif self.path == '/heavy':
                      # Heavy CPU computation
                      start = time.time()
                      result = sum(math.sqrt(i) for i in range(1000000))
                      elapsed = time.time() - start
                      
                      self.send_response(200)
                      self.end_headers()
                      msg = f'Result: {result:.2f}, Time: {elapsed:.3f}s'
                      self.wfile.write(msg.encode())
                  else:
                      self.send_response(404)
                      self.end_headers()
              
              def log_message(self, format, *args):
                  pass  # Suppress logs
          
          print("Starting CPU-intensive server on :8080")
          server = HTTPServer(('', 8080), CPUHandler)
          server.serve_forever()
        
        ports:
        - containerPort: 8080
        
        # สำคัญ: ต้องตั้ง requests สำหรับ HPA
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-app
  namespace: hpa-demo
spec:
  selector:
    app: cpu-app
  ports:
  - port: 80
    targetPort: 8080
```

### 3. สร้าง HPA

```yaml
# hpa-demo.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-app-hpa
  namespace: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30  # ใช้ threshold ต่ำเพื่อดู scaling เร็ว
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 3
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 1
        periodSeconds: 15
```

### 4. Deploy และทดสอบ

```bash
# Deploy
kubectl apply -f cpu-intensive-app.yaml
kubectl apply -f hpa-demo.yaml

# รอให้ pod พร้อม
kubectl wait --for=condition=ready pod -l app=cpu-app -n hpa-demo --timeout=60s

# ดู initial HPA status
kubectl get hpa -n hpa-demo

# Terminal 1: Monitor HPA
watch -n 5 kubectl get hpa,pods -n hpa-demo

# Terminal 2: สร้าง load
kubectl run load-test \
  --image=busybox:1.35 \
  --restart=Never \
  -n hpa-demo \
  -- /bin/sh -c "while true; do wget -q -O- http://cpu-app.hpa-demo.svc.cluster.local/heavy; done"

# ดู CPU usage
kubectl top pods -n hpa-demo

# รอดู HPA scale up (ประมาณ 1-3 นาที)
kubectl describe hpa cpu-app-hpa -n hpa-demo

# หยุด load test
kubectl delete pod load-test -n hpa-demo

# ดู HPA scale down (ประมาณ 5 นาที)
kubectl get hpa -n hpa-demo -w
```

---

## Workshop: Scale ด้วย Custom Metrics

### 1. ติดตั้ง Prometheus และ Prometheus Adapter

```bash
# ติดตั้ง kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# ติดตั้ง prometheus-adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring \
  --set prometheus.url="http://prometheus-operated.monitoring.svc.cluster.local" \
  --set prometheus.port=9090
```

### 2. สร้าง App ที่ expose Custom Metrics

```yaml
# metrics-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          from http.server import HTTPServer, BaseHTTPRequestHandler
          import time
          import threading
          
          # Counters
          request_count = 0
          active_connections = 0
          lock = threading.Lock()
          
          class MetricsHandler(BaseHTTPRequestHandler):
              def do_GET(self):
                  global request_count, active_connections
                  
                  with lock:
                      active_connections += 1
                  
                  if self.path == '/metrics':
                      # Prometheus format metrics
                      with lock:
                          metrics = f"""# HELP http_requests_total Total HTTP requests
          # TYPE http_requests_total counter
          http_requests_total{{method="GET",status="200"}} {request_count}
          
          # HELP active_connections Current active connections
          # TYPE active_connections gauge
          active_connections {active_connections}
          """
                      self.send_response(200)
                      self.send_header('Content-Type', 'text/plain')
                      self.end_headers()
                      self.wfile.write(metrics.encode())
                  else:
                      with lock:
                          request_count += 1
                      time.sleep(0.1)  # Simulate work
                      self.send_response(200)
                      self.end_headers()
                      self.wfile.write(b'Hello World')
                  
                  with lock:
                      active_connections -= 1
              
              def log_message(self, format, *args):
                  pass
          
          print("Starting server with metrics on :8080")
          HTTPServer(('', 8080), MetricsHandler).serve_forever()
        
        ports:
        - containerPort: 8080
        
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "300m"
            memory: "128Mi"
```

### 3. HPA ด้วย Custom Metrics

```yaml
# custom-metrics-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: metrics-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: metrics-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  # Scale based on HTTP requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "50"
```

### 4. ทดสอบ Custom Metrics HPA

```bash
# ตรวจสอบ custom metrics available
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | python3 -m json.tool

# ดู specific metric
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/production/pods/*/http_requests_per_second"

# ดู HPA status
kubectl describe hpa metrics-app-hpa -n production

# สร้าง load
kubectl run load-test \
  --image=busybox:1.35 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://metrics-app.production.svc.cluster.local; done"

# Monitor scaling
kubectl get hpa metrics-app-hpa -n production -w
```

---

## Troubleshooting HPA

### ปัญหา: HPA แสดง "unknown" ใน TARGETS

```bash
# ดู HPA status
kubectl describe hpa my-hpa -n production

# สาเหตุที่พบบ่อย:
# 1. Metrics Server ไม่ทำงาน
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system deployment/metrics-server

# 2. Pod ไม่มี resource requests
kubectl get deployment my-app -n production -o yaml | grep resources -A 5

# 3. HPA ดู metrics ไม่ได้
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```

### ปัญหา: HPA ไม่ Scale Up

```bash
# ตรวจสอบ events
kubectl describe hpa my-hpa -n production

# ดู conditions
kubectl get hpa my-hpa -n production -o jsonpath='{.status.conditions}' | python3 -m json.tool

# ตรวจสอบว่าถึง maxReplicas หรือยัง
kubectl get hpa my-hpa -n production -o custom-columns=\
'NAME:.metadata.name,MIN:.spec.minReplicas,MAX:.spec.maxReplicas,CURRENT:.status.currentReplicas'

# ดู actual CPU usage
kubectl top pods -n production
```

### ปัญหา: HPA Scale Down เร็วเกินไป

```bash
# เพิ่ม stabilizationWindow
kubectl patch hpa my-hpa -n production -p '{
  "spec": {
    "behavior": {
      "scaleDown": {
        "stabilizationWindowSeconds": 600
      }
    }
  }
}'
```

### คำสั่ง HPA ที่ใช้บ่อย

```bash
# ดู HPA ทั้งหมด
kubectl get hpa -A

# ดู HPA รายละเอียด
kubectl describe hpa my-hpa -n production

# ดู HPA events
kubectl get events -n production --field-selector reason=SuccessfulRescale

# ดู scaling history
kubectl get events -n production --sort-by='.lastTimestamp' | grep hpa

# Manual scale (bypass HPA ชั่วคราว)
kubectl scale deployment my-app --replicas=5 -n production
# หมายเหตุ: HPA จะ override ค่านี้ใน sync period ถัดไป

# Pause HPA ชั่วคราว (ลบ HPA ออก แล้วสร้างใหม่)
kubectl delete hpa my-hpa -n production
# แก้ไข replicas manually
kubectl scale deployment my-app --replicas=3 -n production
# สร้าง HPA ใหม่เมื่อพร้อม
kubectl apply -f hpa.yaml
```

---

## Best Practices

### 1. ตั้งค่า Resource Requests เสมอ

```yaml
resources:
  requests:
    cpu: "200m"     # จำเป็น! HPA ใช้ค่านี้คำนวณ utilization
    memory: "128Mi"
```

HPA คำนวณ CPU utilization = actual CPU / requested CPU ถ้าไม่ตั้ง requests จะ error

### 2. ใช้ minReplicas >= 2 สำหรับ Production

```yaml
minReplicas: 2  # High Availability - ไม่ให้เหลือ pod เดียว
maxReplicas: 20
```

### 3. ตั้งค่า ScaleDown ให้ Conservative

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # รอ 5 นาที
    policies:
    - type: Percent
      value: 10          # ค่อยๆ ลด
      periodSeconds: 60
```

### 4. ใช้หลาย Metrics พร้อมกัน

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80
# HPA จะ scale เมื่อ metric ใดๆ เกิน threshold
```

### 5. ทดสอบ Scaling ก่อน Production

```bash
# ทดสอบด้วย load testing tool
kubectl run load-test --image=williamyeh/wrk \
  --restart=Never \
  -- wrk -t4 -c100 -d60s http://my-app/

# ดู HPA behavior
kubectl get hpa -w
```

---

## สรุป

HPA เป็นเครื่องมือสำคัญสำหรับ auto-scaling:

1. **ต้องการ Metrics Server** สำหรับ CPU/Memory metrics
2. **ต้องตั้ง Resource Requests** บน containers
3. **HPA คำนวณ** ตามสูตร: desiredReplicas = currentReplicas × (currentMetric/targetMetric)
4. **ใช้ Behavior** เพื่อควบคุม scaling speed
5. **Custom Metrics** ต้องการ Prometheus Adapter
6. **Scale Down Conservative** ป้องกัน oscillation

ในบทต่อไป เราจะเรียนรู้เรื่อง **Vertical Pod Autoscaler (VPA)** - การปรับ CPU/Memory ของ Pod อัตโนมัติ
