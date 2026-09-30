# Part 30: Multi-Container Pods - Patterns การใช้หลาย Containers

## สารบัญ
1. [Multi-Container Pod คืออะไร](#multi-container-pod-คืออะไร)
2. [Shared Resources ระหว่าง Containers](#shared-resources-ระหว่าง-containers)
3. [Ambassador Pattern](#ambassador-pattern)
4. [Adapter Pattern](#adapter-pattern)
5. [Sidecar Pattern (Review)](#sidecar-pattern-review)
6. [เปรียบเทียบ Patterns](#เปรียบเทียบ-patterns)
7. [Workshop: Full-stack App ใน Single Pod](#workshop-full-stack-app-ใน-single-pod)
8. [Workshop: Database Proxy Ambassador](#workshop-database-proxy-ambassador)
9. [Workshop: Log Format Adapter](#workshop-log-format-adapter)
10. [Workshop: Redis Cache Sidecar](#workshop-redis-cache-sidecar)
11. [Container Communication](#container-communication)
12. [Best Practices และ Anti-patterns](#best-practices-และ-anti-patterns)

---

## Multi-Container Pod คืออะไร

**Multi-Container Pod** คือ Pod ที่มีมากกว่า 1 container ทำงานร่วมกัน:

### ทรัพยากรที่ Containers ใน Pod เดียวกัน Share กัน

```
Pod (shared context):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  Container A    │    │  Container B    │                │
│  │                 │    │                 │                │
│  │  localhost:8080 │◄──►│  localhost:9090 │ (shared network)│
│  └────────┬────────┘    └────────┬────────┘                │
│           │                     │                          │
│           └─────────┬───────────┘                          │
│                     │                                      │
│           ┌─────────▼───────────┐                          │
│           │   Shared Volume     │ (shared storage)         │
│           │   /shared-data      │                          │
│           └─────────────────────┘                          │
│                                                             │
│  Network: localhost, 127.0.0.1 (same network namespace)    │
│  Storage: emptyDir, PVC (mounted หลาย containers ได้)      │
│  Process: ดู processes ของกัน (ถ้า shareProcessNamespace)   │
└─────────────────────────────────────────────────────────────┘
```

### สิ่งที่แต่ละ Container **ไม่** Share กัน

- **Filesystem root**: แต่ละ container มี filesystem ของตัวเอง
- **Process space** (default): แต่ละ container มี PID namespace ของตัวเอง (ยกเว้นตั้ง `shareProcessNamespace: true`)
- **User/Group IDs**: แต่ละ container มี user ของตัวเอง

---

## Shared Resources ระหว่าง Containers

### 1. Shared Network

```yaml
spec:
  containers:
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80    # รับ traffic บน port 80
  
  - name: app-server
    image: my-app:latest
    ports:
    - containerPort: 8080  # ฟัง port 8080
  
  # nginx สามารถ proxy ไปยัง app ผ่าน localhost:8080
  # เพราะ share network namespace เดียวกัน
```

```bash
# ตัวอย่าง: container A ping container B ได้ผ่าน localhost
kubectl exec -it my-pod -c container-a -- curl http://localhost:8080
# ← ส่งไปยัง container B ที่ฟัง 8080
```

### 2. Shared Volumes

```yaml
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  
  containers:
  - name: writer
    image: busybox
    volumeMounts:
    - name: shared-data
      mountPath: /write
    command: ['sh', '-c', 'while true; do date > /write/time.txt; sleep 5; done']
  
  - name: reader
    image: busybox
    volumeMounts:
    - name: shared-data
      mountPath: /read
    command: ['sh', '-c', 'while true; do cat /read/time.txt; sleep 5; done']
  
  # writer เขียนไปยัง /write/time.txt
  # reader อ่านจาก /read/time.txt
  # ทั้งคู่ใช้ volume เดียวกัน (shared-data)
```

### 3. Share Process Namespace

```yaml
spec:
  shareProcessNamespace: true   # ← ให้ containers เห็น processes ของกัน
  
  containers:
  - name: app
    image: my-app:latest
  
  - name: debugger
    image: busybox
    command: ['sh', '-c', 'sleep infinity']
    securityContext:
      capabilities:
        add: ['SYS_PTRACE']  # สำหรับ debugging
```

```bash
# container debugger เห็น processes ของ app container
kubectl exec -it my-pod -c debugger -- ps aux
```

---

## Ambassador Pattern

**Ambassador** คือ container ที่ทำหน้าที่เป็น **proxy** สำหรับ external services:

```
┌──────────────────────────────────────────────────────────────┐
│                           Pod                               │
│                                                              │
│   ┌───────────────┐        ┌───────────────────┐            │
│   │  Main App     │        │  Ambassador       │            │
│   │               │◄──────►│  (Proxy)          │◄──External │
│   │  localhost:DB │        │  - Connection     │   Service  │
│   │               │        │    pooling        │            │
│   └───────────────┘        │  - Retry logic    │            │
│                             │  - Circuit break  │            │
│                             │  - Discovery      │            │
│                             └───────────────────┘            │
└──────────────────────────────────────────────────────────────┘

แอป: เชื่อมต่อ localhost:5432 (ง่ายๆ)
Ambassador: จัดการ connection ซับซ้อนกับ DB cluster จริง
```

### Use Cases ของ Ambassador

- **Database Proxy**: pg_bouncer, ProxySQL สำหรับ connection pooling
- **Service Discovery**: ช่วย main app ค้นหา services
- **Legacy App Adapter**: ช่วย legacy app เชื่อมต่อ modern infrastructure

### Ambassador Pattern YAML

```yaml
# ambassador-pattern.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-db-ambassador
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-with-db-ambassador
  template:
    metadata:
      labels:
        app: app-with-db-ambassador
    spec:
      containers:
      # ── Main Application ─────────────────────────────────
      # เชื่อมต่อ localhost:5432 (ง่ายๆ ไม่ต้องรู้ว่า DB อยู่ไหน)
      - name: app
        image: my-python-app:latest
        env:
        - name: DATABASE_URL
          value: "postgresql://postgres:password@localhost:5432/mydb"
          # ← ชี้ไปยัง localhost (ambassador)
        
        ports:
        - containerPort: 8080
        
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 5
      
      # ── Ambassador: PgBouncer Connection Pooler ──────────
      - name: pgbouncer
        image: pgbouncer/pgbouncer:latest
        
        env:
        - name: DATABASES_HOST
          value: "postgres-primary.production.svc.cluster.local"
          # ← ชี้ไปยัง actual PostgreSQL server
        - name: DATABASES_PORT
          value: "5432"
        - name: DATABASES_USER
          value: "postgres"
        - name: DATABASES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: DATABASES_DBNAME
          value: "mydb"
        - name: POOL_MODE
          value: "transaction"
        - name: MAX_CLIENT_CONN
          value: "100"
        - name: DEFAULT_POOL_SIZE
          value: "20"
        
        ports:
        - containerPort: 5432
          # ← Main app เชื่อมต่อ port นี้ ผ่าน localhost
        
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
        
        livenessProbe:
          tcpSocket:
            port: 5432
          periodSeconds: 10
```

### Ambassador สำหรับ Redis Sentinel

```yaml
# redis-ambassador.yaml
# Main app เชื่อมต่อ Redis แบบ standalone ผ่าน localhost
# Ambassador จัดการ Sentinel discovery

spec:
  containers:
  - name: app
    image: my-app:latest
    env:
    - name: REDIS_URL
      value: "redis://localhost:6379"  # ← localhost, ง่าย
  
  - name: redis-ambassador
    image: haproxy:latest
    volumeMounts:
    - name: haproxy-config
      mountPath: /usr/local/etc/haproxy/haproxy.cfg
      subPath: haproxy.cfg
    # Ambassador จัดการ Redis Sentinel routing ให้
```

---

## Adapter Pattern

**Adapter** คือ container ที่แปลง **output format** ของ main application ให้ตรงกับสิ่งที่ external system ต้องการ:

```
┌──────────────────────────────────────────────────────────────┐
│                           Pod                               │
│                                                              │
│   ┌───────────────┐        ┌───────────────────┐            │
│   │  Legacy App   │───────►│  Adapter          │───►External│
│   │               │        │  (Format          │   System   │
│   │  custom format│        │   converter)      │  (standard)│
│   └───────────────┘        └───────────────────┘            │
│                                                              │
└──────────────────────────────────────────────────────────────┘

App: output custom log format
Adapter: แปลงเป็น JSON format ที่ Elasticsearch ต้องการ
```

### Use Cases ของ Adapter

- **Metrics Format Converter**: แปลง StatsD → Prometheus format
- **Log Format Converter**: แปลง legacy log → structured JSON
- **Protocol Adapter**: แปลง AMQP → REST API
- **Schema Transformer**: แปลง data schema สำหรับ different versions

### Adapter Pattern YAML

```yaml
# adapter-pattern.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app-with-adapter
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
      annotations:
        # Prometheus จะ scrape adapter (ไม่ใช่ legacy app)
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      volumes:
      - name: shared-logs
        emptyDir: {}
      
      containers:
      # ── Legacy Application ────────────────────────────────
      # เขียน logs ในรูปแบบเก่า: "YYYY-MM-DD HH:MM:SS LEVEL message"
      - name: legacy-app
        image: legacy-java-app:latest
        
        volumeMounts:
        - name: shared-logs
          mountPath: /app/logs
        
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
      
      # ── Adapter: Log Format Converter ─────────────────────
      # แปลง legacy log format → JSON format
      - name: log-adapter
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import re
          import json
          import time
          import sys
          from datetime import datetime
          
          LOG_FILE = '/logs/app.log'
          OUTPUT_FILE = '/logs/app-formatted.json.log'
          
          # Legacy log pattern
          PATTERN = re.compile(
            r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) (\w+) (.*)'
          )
          
          def convert_log_line(line):
              line = line.strip()
              if not line:
                  return None
              
              match = PATTERN.match(line)
              if not match:
                  return json.dumps({
                      'timestamp': datetime.utcnow().isoformat(),
                      'level': 'UNKNOWN',
                      'message': line,
                      'source': 'legacy-app'
                  })
              
              timestamp, level, message = match.groups()
              return json.dumps({
                  'timestamp': timestamp + 'Z',
                  'level': level,
                  'message': message,
                  'source': 'legacy-app',
                  'app': 'legacy-java-app'
              })
          
          print("Log adapter started, watching:", LOG_FILE)
          
          # Tail log file และแปลง format
          try:
              with open(LOG_FILE, 'r') as infile, \
                   open(OUTPUT_FILE, 'a') as outfile:
                  
                  # ไปท้ายไฟล์
                  infile.seek(0, 2)
                  
                  while True:
                      line = infile.readline()
                      if line:
                          converted = convert_log_line(line)
                          if converted:
                              outfile.write(converted + '\n')
                              outfile.flush()
                      else:
                          time.sleep(0.1)
          except FileNotFoundError:
              print(f"Waiting for {LOG_FILE}...")
              time.sleep(5)
        
        volumeMounts:
        - name: shared-logs
          mountPath: /logs
        
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
      
      # ── Adapter: Metrics Format Converter ─────────────────
      # แปลง StatsD metrics → Prometheus format
      - name: metrics-adapter
        image: prom/statsd-exporter:latest
        args:
        - "--statsd.listen-udp=:9125"
        - "--web.listen-address=:9090"
        - "--statsd.mapping-config=/etc/statsd/mapping.yaml"
        
        ports:
        - containerPort: 9125
          protocol: UDP
          name: statsd
        - containerPort: 9090
          name: metrics
        
        # Legacy app ส่ง StatsD metrics มายัง localhost:9125
        # Adapter แปลงเป็น Prometheus format บน :9090
        
        resources:
          requests:
            cpu: "25m"
            memory: "32Mi"
          limits:
            cpu: "50m"
            memory: "64Mi"
```

---

## Sidecar Pattern (Review)

```yaml
# sidecar ที่รันตลอดเวลา (ต่างจาก init container)
spec:
  containers:
  - name: main-app
    image: my-app:latest
  
  - name: sidecar        # รันพร้อม main
    image: helper:latest
    # ทำงาน: logging, proxy, metrics, config reload
```

---

## เปรียบเทียบ Patterns

```
┌───────────────┬──────────────────┬────────────────────────────────┐
│ Pattern       │ Position         │ Purpose                        │
├───────────────┼──────────────────┼────────────────────────────────┤
│ Sidecar       │ ข้างๆ main        │ เพิ่ม functionality (logs,      │
│               │ รันพร้อมกัน       │ metrics, proxy)                │
├───────────────┼──────────────────┼────────────────────────────────┤
│ Ambassador    │ หน้า external    │ Proxy/abstraction ไปยัง        │
│               │ services          │ external services              │
│               │ รันพร้อมกัน       │ (DB pooling, service discovery)│
├───────────────┼──────────────────┼────────────────────────────────┤
│ Adapter       │ หลัง main output  │ แปลง output format ให้         │
│               │ รันพร้อมกัน       │ compatible กับ external systems │
├───────────────┼──────────────────┼────────────────────────────────┤
│ Init          │ ก่อน main        │ Setup, migration, wait for deps │
│ Container     │ รัน sequential    │                                │
└───────────────┴──────────────────┴────────────────────────────────┘
```

---

## Workshop: Full-stack App ใน Single Pod

เป้าหมาย: Deploy full-stack application ใน Single Pod (สำหรับ development)

**หมายเหตุ**: ใน production ควรแยก services ออกจากกัน การรวมใน single pod เหมาะสำหรับ dev/test

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Single Pod (fullstack-app)                  │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  React      │  │  FastAPI    │  │  Nginx      │            │
│  │  Frontend   │  │  Backend    │  │  Proxy      │            │
│  │  (port 3000)│  │  (port 8000)│  │  (port 80)  │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐                              │
│  │  Redis      │  │  Log        │                              │
│  │  Cache      │  │  Collector  │                              │
│  │  (port 6379)│  │  (Fluentd)  │                              │
│  └─────────────┘  └─────────────┘                              │
│                                                                  │
│  Shared Volume: /app/logs                                       │
└──────────────────────────────────────────────────────────────────┘
```

### 1. สร้าง ConfigMaps

```yaml
# fullstack-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: development
data:
  default.conf: |
    upstream frontend {
      server localhost:3000;
    }
    
    upstream backend {
      server localhost:8000;
    }
    
    server {
      listen 80;
      
      # Frontend
      location / {
        proxy_pass http://frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
      
      # Backend API
      location /api/ {
        proxy_pass http://backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
      
      # Health check
      location /health {
        return 200 "OK";
        add_header Content-Type text/plain;
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  REDIS_URL: "redis://localhost:6379"
  DATABASE_URL: "sqlite:///./test.db"
  DEBUG: "true"
  LOG_DIR: "/app/logs"
```

### 2. สร้าง Fullstack Deployment

```yaml
# fullstack-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fullstack-app
  namespace: development
  labels:
    app: fullstack-app
spec:
  replicas: 1   # Development: 1 replica เท่านั้น
  selector:
    matchLabels:
      app: fullstack-app
  template:
    metadata:
      labels:
        app: fullstack-app
    spec:
      volumes:
      - name: app-logs
        emptyDir: {}
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: app-data
        emptyDir: {}
      
      initContainers:
      # Init: สร้าง directories ที่จำเป็น
      - name: setup-dirs
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          mkdir -p /app/logs
          mkdir -p /app/data
          chmod 777 /app/logs /app/data
          echo "Directories created"
        volumeMounts:
        - name: app-logs
          mountPath: /app/logs
        - name: app-data
          mountPath: /app/data
      
      containers:
      # ── Container 1: React Frontend ──────────────────────
      - name: frontend
        image: node:18-alpine
        command:
        - sh
        - -c
        - |
          # สร้าง simple React-like app
          mkdir -p /app
          cat > /app/server.js << 'EOF'
          const http = require('http');
          const server = http.createServer((req, res) => {
            res.writeHead(200, {'Content-Type': 'text/html'});
            res.end(`
              <!DOCTYPE html>
              <html>
              <head><title>Fullstack Demo</title></head>
              <body>
                <h1>Kubernetes Fullstack Demo</h1>
                <p>Pod: ${process.env.POD_NAME || 'unknown'}</p>
                <div id="data"></div>
                <script>
                  fetch('/api/info')
                    .then(r => r.json())
                    .then(d => {
                      document.getElementById('data').innerHTML = 
                        '<pre>' + JSON.stringify(d, null, 2) + '</pre>';
                    });
                </script>
              </body>
              </html>
            `);
          });
          server.listen(3000, () => console.log('Frontend on :3000'));
          EOF
          node /app/server.js
        
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        
        ports:
        - containerPort: 3000
          name: frontend
        
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
      
      # ── Container 2: FastAPI Backend ──────────────────────
      - name: backend
        image: python:3.11-slim
        command:
        - sh
        - -c
        - |
          pip install fastapi uvicorn redis --quiet
          
          cat > /app/main.py << 'EOF'
          from fastapi import FastAPI
          import redis
          import os
          import json
          import logging
          from datetime import datetime
          
          app = FastAPI()
          
          log_dir = os.environ.get('LOG_DIR', '/app/logs')
          os.makedirs(log_dir, exist_ok=True)
          
          logging.basicConfig(
              filename=f'{log_dir}/backend.log',
              level=logging.INFO,
              format='%(asctime)s [%(levelname)s] %(message)s'
          )
          
          try:
              r = redis.from_url(os.environ.get('REDIS_URL', 'redis://localhost:6379'))
          except:
              r = None
          
          @app.get("/info")
          def get_info():
              cache_key = "app_info"
              
              # ลอง cache
              if r:
                  cached = r.get(cache_key)
                  if cached:
                      logging.info("Cache HIT")
                      return json.loads(cached)
              
              logging.info("Cache MISS - fetching fresh data")
              
              data = {
                  "status": "healthy",
                  "timestamp": datetime.utcnow().isoformat(),
                  "pod": os.environ.get("POD_NAME", "unknown"),
                  "version": "1.0.0",
                  "cache": "redis" if r else "none"
              }
              
              # บันทึกลง cache
              if r:
                  r.set(cache_key, json.dumps(data), ex=30)
              
              return data
          
          @app.get("/health")
          def health():
              return {"status": "ok"}
          EOF
          
          mkdir -p /app
          python -m uvicorn main:app --host 0.0.0.0 --port 8000 --app-dir /app
        
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_URL
        - name: LOG_DIR
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_DIR
        
        ports:
        - containerPort: 8000
          name: backend
        
        volumeMounts:
        - name: app-logs
          mountPath: /app/logs
        - name: app-data
          mountPath: /app/data
        
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "400m"
            memory: "512Mi"
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 5
      
      # ── Container 3: Redis Cache ──────────────────────────
      - name: redis
        image: redis:7.0-alpine
        command: ["redis-server", "--maxmemory", "64mb", "--maxmemory-policy", "allkeys-lru"]
        
        ports:
        - containerPort: 6379
          name: redis
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5
        
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 10
          periodSeconds: 10
      
      # ── Container 4: Nginx Proxy ──────────────────────────
      - name: nginx
        image: nginx:1.21-alpine
        
        ports:
        - containerPort: 80
          name: http
        
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
        
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          periodSeconds: 10
      
      # ── Container 5: Log Collector ────────────────────────
      - name: log-collector
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          # Simple log aggregator
          echo "Log collector started"
          
          while true; do
            for logfile in /app/logs/*.log; do
              if [ -f "$logfile" ]; then
                LINES=$(wc -l < "$logfile" 2>/dev/null || echo "0")
                echo "$(date): $logfile has $LINES lines"
              fi
            done
            sleep 60
          done
        
        volumeMounts:
        - name: app-logs
          mountPath: /app/logs
          readOnly: true
        
        resources:
          requests:
            cpu: "25m"
            memory: "16Mi"
          limits:
            cpu: "50m"
            memory: "32Mi"
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: fullstack-app
  namespace: development
spec:
  selector:
    app: fullstack-app
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

### 3. Deploy และทดสอบ

```bash
# สร้าง namespace
kubectl create namespace development

# Apply
kubectl apply -f fullstack-config.yaml
kubectl apply -f fullstack-deployment.yaml

# รอให้ pods พร้อม
kubectl rollout status deployment/fullstack-app -n development

# ดู pod status
kubectl get pods -n development

# ดู containers ทั้งหมดใน pod
kubectl describe pod -n development -l app=fullstack-app | grep "Container ID"

# ดู logs ของแต่ละ container
kubectl logs -n development -l app=fullstack-app -c frontend
kubectl logs -n development -l app=fullstack-app -c backend
kubectl logs -n development -l app=fullstack-app -c redis

# Port-forward เพื่อทดสอบ
kubectl port-forward svc/fullstack-app 8080:80 -n development &

# ทดสอบ
curl http://localhost:8080/          # Frontend
curl http://localhost:8080/api/info  # Backend API
curl http://localhost:8080/health    # Health check

# ดู metrics ของทุก container
kubectl top pod -n development -l app=fullstack-app --containers
```

---

## Workshop: Database Proxy Ambassador

```yaml
# db-ambassador.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-db-proxy
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-with-db-proxy
  template:
    metadata:
      labels:
        app: app-with-db-proxy
    spec:
      containers:
      # Main App: เชื่อมต่อ postgres ผ่าน localhost
      - name: app
        image: my-app:latest
        env:
        - name: DB_HOST
          value: "localhost"    # ← Ambassador listen บน localhost
        - name: DB_PORT
          value: "5432"
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
      
      # Ambassador: PgBouncer
      - name: pgbouncer-ambassador
        image: bitnami/pgbouncer:latest
        env:
        - name: POSTGRESQL_HOST
          value: "postgres-cluster.production.svc.cluster.local"
        - name: POSTGRESQL_PORT
          value: "5432"
        - name: POSTGRESQL_DATABASE
          value: "myapp"
        - name: POSTGRESQL_USERNAME
          value: "app_user"
        - name: POSTGRESQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: PGBOUNCER_POOL_MODE
          value: "transaction"
        - name: PGBOUNCER_MAX_CLIENT_CONN
          value: "100"
        - name: PGBOUNCER_DEFAULT_POOL_SIZE
          value: "20"
        - name: PGBOUNCER_BIND_ADDRESS
          value: "127.0.0.1"  # ← bind บน localhost เท่านั้น
        ports:
        - containerPort: 5432
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
```

---

## Workshop: Log Format Adapter

```yaml
# log-adapter.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-script
  namespace: production
data:
  adapt_logs.py: |
    #!/usr/bin/env python3
    """
    Log Format Adapter
    แปลง Apache Combined Log format → JSON format
    """
    import re
    import json
    import time
    import sys
    from datetime import datetime
    
    # Apache Combined Log Pattern
    APACHE_PATTERN = re.compile(
        r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
        r'"(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" '
        r'(?P<status>\d+) (?P<size>\S+) '
        r'"(?P<referer>[^"]*)" "(?P<agent>[^"]*)"'
    )
    
    def parse_apache_log(line):
        match = APACHE_PATTERN.match(line.strip())
        if not match:
            return None
        
        d = match.groupdict()
        return {
            "@timestamp": datetime.utcnow().isoformat() + "Z",
            "source_ip": d['ip'],
            "method": d['method'],
            "path": d['path'],
            "status_code": int(d['status']),
            "response_size": int(d['size']) if d['size'] != '-' else 0,
            "user_agent": d['agent'],
            "referer": d['referer'],
            "log_source": "apache"
        }
    
    # Process stdin
    for line in sys.stdin:
        parsed = parse_apache_log(line)
        if parsed:
            print(json.dumps(parsed))
        sys.stdout.flush()
```

---

## Workshop: Redis Cache Sidecar

```yaml
# redis-cache-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-cache
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-with-cache
  template:
    metadata:
      labels:
        app: app-with-cache
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: CACHE_HOST
          value: "localhost"    # ← Redis sidecar บน localhost
        - name: CACHE_PORT
          value: "6379"
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      
      # Redis Cache Sidecar
      - name: redis-cache
        image: redis:7.0-alpine
        args:
        - redis-server
        - --maxmemory
        - "128mb"
        - --maxmemory-policy
        - allkeys-lru
        - --bind
        - 127.0.0.1  # bind localhost เท่านั้น
        - --protected-mode
        - "no"
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: "50m"
            memory: "128Mi"
          limits:
            cpu: "100m"
            memory: "256Mi"
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          periodSeconds: 5
```

---

## Container Communication

### Communication ผ่าน localhost

```bash
# Containers ใน Pod เดียวกัน communicate ผ่าน localhost

# Container A → ส่ง request ไปยัง Container B (localhost:port)
kubectl exec -it my-pod -c container-a -- curl http://localhost:8080

# Container B → query Redis sidecar
kubectl exec -it my-pod -c backend -- redis-cli -h localhost ping
```

### Communication ผ่าน Shared Volume

```bash
# Container A เขียน
kubectl exec -it my-pod -c writer -- echo "Hello" > /shared/message.txt

# Container B อ่าน
kubectl exec -it my-pod -c reader -- cat /shared/message.txt
```

### Named Pipes (IPC)

```yaml
volumes:
- name: ipc-pipe
  emptyDir: {}

containers:
- name: producer
  command: ['sh', '-c', 'mkfifo /ipc/pipe; while true; do date > /ipc/pipe; done']
  volumeMounts:
  - name: ipc-pipe
    mountPath: /ipc

- name: consumer
  command: ['sh', '-c', 'while true; do cat /ipc/pipe; done']
  volumeMounts:
  - name: ipc-pipe
    mountPath: /ipc
```

---

## Best Practices และ Anti-patterns

### Best Practices

#### 1. แยก Concerns อย่างชัดเจน

```yaml
# ดี: แต่ละ container ทำหน้าที่เดียว
containers:
- name: app        # business logic
- name: proxy      # traffic handling
- name: logger     # log shipping
- name: metrics    # metrics export

# ไม่ดี: รวมทุกอย่างใน main container
containers:
- name: app        # business logic + logging + metrics + proxy
```

#### 2. Resource Isolation

```yaml
# กำหนด resources ให้ทุก container
containers:
- name: main
  resources:
    requests: {cpu: "200m", memory: "256Mi"}
    limits: {cpu: "500m", memory: "512Mi"}

- name: sidecar
  resources:
    requests: {cpu: "50m", memory: "64Mi"}   # สำคัญ!
    limits: {cpu: "100m", memory: "128Mi"}
```

#### 3. Proper Health Checks

```yaml
containers:
- name: nginx-proxy
  readinessProbe:
    httpGet:
      path: /health
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 5
  # readiness probe บน proxy (ไม่ใช่แค่ main app)
```

#### 4. Graceful Shutdown

```yaml
containers:
- name: main
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 5"]
  terminationGracePeriodSeconds: 30
```

### Anti-patterns ที่ควรหลีกเลี่ยง

#### Anti-pattern 1: รวม Services ที่ Scale แยกกัน

```yaml
# ไม่ดี: Frontend และ Backend ที่ต้องการ scale แยกกัน
# อยู่ใน Pod เดียวกัน

spec:
  containers:
  - name: frontend    # ต้องการ 10 replicas
  - name: backend     # ต้องการ 5 replicas
  # ← Scale พร้อมกัน ไม่ได้!

# ดี: แยกเป็น Deployments ต่างกัน
# Deployment frontend: replicas=10
# Deployment backend: replicas=5
```

#### Anti-pattern 2: Containers ที่มี Different Lifecycles

```yaml
# ไม่ดี: Database กับ Application ใน Pod เดียว
containers:
- name: postgres    # lifecycle ยาว, stateful
- name: app         # lifecycle สั้น, stateless
# ← ถ้า app crash → postgres ถูก restart ด้วย!
```

#### Anti-pattern 3: Heavy Sidecars

```yaml
# ไม่ดี: Sidecar ใช้ resources มากกว่า main app
- name: main
  resources:
    requests: {cpu: "100m", memory: "128Mi"}

- name: heavy-sidecar
  resources:
    requests: {cpu: "500m", memory: "1Gi"}  # ← มากเกินไป!
```

---

## สรุปหลักสูตร Parts 21-30

### สิ่งที่เรียนมาในหมวด Workloads:

| Part | หัวข้อ | สรุปสั้น |
|------|--------|---------|
| 21 | StatefulSets | สำหรับ stateful apps, stable identity, ordered ops |
| 22 | DaemonSets | รัน Pod บนทุก Node, log/monitoring agents |
| 23 | Jobs & CronJobs | Batch processing, scheduled tasks |
| 24 | HPA | Auto-scale จำนวน pods ตาม metrics |
| 25 | VPA | Auto-tune resource requests/limits |
| 26 | Resource Limits | Requests, limits, QoS, LimitRange, Quota |
| 27 | Pod Disruption Budget | ป้องกัน downtime ขณะ maintenance |
| 28 | Init Containers | Setup, migration ก่อน main container |
| 29 | Sidecar Containers | Log shipping, proxy, metrics ควบคู่ main |
| 30 | Multi-Container Pods | Ambassador, Adapter, Sidecar patterns |

### Key Takeaways:

1. **StatefulSet** = database cluster, message queues ที่ต้องการ identity
2. **DaemonSet** = เครื่องมือ per-node (logging, monitoring)
3. **Job/CronJob** = งานที่รันครั้งเดียวหรือตามเวลา
4. **HPA** = scale ออก/เข้าตาม load (stateless apps)
5. **VPA** = ปรับ CPU/Memory ให้พอดี (resource optimization)
6. **Resource Limits** = ป้องกัน resource starvation
7. **PDB** = รับประกัน availability ขณะ maintenance
8. **Init Containers** = ทำ setup ก่อน main container เสมอ
9. **Sidecar** = เพิ่ม functionality โดยไม่แก้ application code
10. **Multi-container Patterns** = Ambassador, Adapter, Sidecar

ยินดีด้วย! คุณได้เรียนรู้ **Kubernetes Workloads** ครบถ้วนแล้ว ซึ่งเป็นรากฐานสำคัญของการ deploy applications บน Kubernetes อย่างมืออาชีพ
