# Part 19: ConfigMaps - จัดการ Configuration อย่างมีระบบ

## สารบัญ
1. [ConfigMap คืออะไร](#configmap-คืออะไร)
2. [สร้าง ConfigMap หลายวิธี](#สร้าง-configmap)
3. [ใช้ ConfigMap ใน Pod](#ใช้-configmap-ใน-pod)
4. [ConfigMap Best Practices](#best-practices)
5. [Workshop: Config แอปพลิเคชัน Production](#workshop)

---

## 1. ConfigMap คืออะไร

**ConfigMap** คือ API object ที่ใช้เก็บ non-sensitive configuration data เป็น key-value pairs แยกออกจาก container image เพื่อให้ application configurations เปลี่ยนแปลงได้โดยไม่ต้องสร้าง image ใหม่

### ปัญหาที่ ConfigMap แก้ไข

```
ไม่มี ConfigMap (hardcode in image):
┌────────────────────────────────┐
│ Docker Image: myapp:prod       │
│ ENV DB_HOST=prod-db.example    │
│ ENV APP_PORT=8080              │
│ ENV MAX_CONNECTIONS=100        │
└────────────────────────────────┘
ปัญหา: ต้อง build image ใหม่ทุกครั้งที่ config เปลี่ยน!

มี ConfigMap:
┌─────────────────────┐    ┌──────────────────────────┐
│ ConfigMap           │    │ Docker Image: myapp:latest │
│ DB_HOST=prod-db     │──► │ (ไม่มี hardcode config)    │
│ APP_PORT=8080       │    └──────────────────────────┘
│ MAX_CONNECTIONS=100 │
└─────────────────────┘
เปลี่ยน config ได้โดยไม่ต้อง rebuild!
```

### ConfigMap Data Types

ConfigMap รองรับข้อมูล 2 types:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mixed-config
data:
  # 1. ข้อมูลธรรมดา (key: value)
  APP_PORT: "8080"
  DB_HOST: "postgres.default.svc.cluster.local"
  
  # 2. ไฟล์ข้อความ (multi-line)
  nginx.conf: |
    server {
      listen 80;
      root /var/www/html;
    }
  
  app.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://postgres:5432/mydb
    logging.level.root=INFO

binaryData:
  # Binary data (base64 encoded) - ไม่ค่อยใช้
  certificate.p12: <base64-encoded-binary>
```

---

## 2. สร้าง ConfigMap หลายวิธี

### วิธีที่ 1: From Literals (--from-literal)

```bash
# สร้าง ConfigMap จาก literal values
kubectl create configmap app-config \
    --from-literal=APP_ENV=production \
    --from-literal=APP_PORT=8080 \
    --from-literal=DB_HOST=postgres.default.svc.cluster.local \
    --from-literal=MAX_CONNECTIONS=100 \
    --from-literal=LOG_LEVEL=info

# ดู ConfigMap
kubectl get configmap app-config
kubectl describe configmap app-config

# ดู YAML
kubectl get configmap app-config -o yaml
```

### วิธีที่ 2: From File (--from-file)

```bash
# สร้างไฟล์ config ก่อน
cat <<'EOF' > /tmp/app.conf
# Application Configuration
server.port=8080
server.host=0.0.0.0
database.host=postgres
database.port=5432
database.name=myapp
connection.pool.size=10
cache.ttl=3600
log.level=INFO
log.format=json
EOF

cat <<'EOF' > /tmp/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server localhost:8080;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
}
EOF

# สร้าง ConfigMap จากไฟล์
# key = ชื่อไฟล์, value = เนื้อหาไฟล์
kubectl create configmap nginx-config \
    --from-file=/tmp/nginx.conf

# สร้าง ConfigMap จากหลายไฟล์
kubectl create configmap app-files-config \
    --from-file=/tmp/app.conf \
    --from-file=/tmp/nginx.conf

# สร้าง ConfigMap จากไฟล์แต่กำหนด key ใหม่
kubectl create configmap named-config \
    --from-file=application.conf=/tmp/app.conf \
    --from-file=webserver.conf=/tmp/nginx.conf

# สร้าง ConfigMap จากทั้ง directory
mkdir -p /tmp/config-dir
cp /tmp/app.conf /tmp/config-dir/
cp /tmp/nginx.conf /tmp/config-dir/
echo "DATABASE_URL=postgres://localhost/mydb" > /tmp/config-dir/database.env

kubectl create configmap dir-config \
    --from-file=/tmp/config-dir/

# ดู ConfigMaps
kubectl get configmaps
kubectl describe configmap dir-config
```

### วิธีที่ 3: From env-file (--from-env-file)

```bash
# สร้าง .env file
cat <<'EOF' > /tmp/app.env
# Application Settings
APP_ENV=production
APP_PORT=8080
APP_VERSION=2.0
APP_DEBUG=false

# Database Settings
DB_HOST=postgres.default.svc.cluster.local
DB_PORT=5432
DB_NAME=myapp
DB_POOL_SIZE=10

# Redis Settings
REDIS_HOST=redis.default.svc.cluster.local
REDIS_PORT=6379
REDIS_DB=0

# Logging
LOG_LEVEL=info
LOG_FORMAT=json
EOF

# สร้าง ConfigMap จาก env file
# (แต่ละ key-value pair จะเป็น key แยกใน ConfigMap)
kubectl create configmap env-config \
    --from-env-file=/tmp/app.env

# ดู ConfigMap
kubectl get configmap env-config -o yaml
```

### วิธีที่ 4: จาก YAML Manifest

```yaml
# full-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: production
  labels:
    app: webapp
    env: production
  annotations:
    description: "Main configuration for webapp"
    managed-by: "platform-team"
data:
  # === Simple Key-Value ===
  APP_ENV: "production"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  
  # === Database Config ===
  DB_HOST: "postgres.production.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "webapp_prod"
  DB_POOL_SIZE: "20"
  DB_CONN_TIMEOUT: "30"
  
  # === Redis Config ===
  REDIS_HOST: "redis.production.svc.cluster.local"
  REDIS_PORT: "6379"
  REDIS_DB: "0"
  REDIS_POOL_SIZE: "10"
  
  # === Feature Flags ===
  FEATURE_DARK_MODE: "true"
  FEATURE_NEW_CHECKOUT: "false"
  FEATURE_ANALYTICS: "true"
  
  # === Config Files ===
  app.properties: |
    # Application Properties
    server.port=8080
    server.shutdown=graceful
    
    # Database
    spring.datasource.url=jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.hibernate.ddl-auto=validate
    
    # Redis
    spring.redis.host=${REDIS_HOST}
    spring.redis.port=${REDIS_PORT}
    
    # Logging
    logging.level.root=INFO
    logging.level.com.example=DEBUG
    logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
  
  nginx.conf: |
    worker_processes auto;
    
    events {
        worker_connections 1024;
    }
    
    http {
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent"';
        
        access_log /var/log/nginx/access.log main;
        error_log /var/log/nginx/error.log warn;
        
        upstream backend {
            server 127.0.0.1:8080;
            keepalive 32;
        }
        
        server {
            listen 80;
            
            location / {
                proxy_pass http://backend;
                proxy_http_version 1.1;
                proxy_set_header Connection "";
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_read_timeout 300s;
                proxy_connect_timeout 75s;
            }
            
            location /health {
                return 200 'OK';
                add_header Content-Type text/plain;
            }
            
            location /metrics {
                return 200 'ok';
            }
        }
    }
  
  log4j2.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <Configuration status="WARN">
        <Appenders>
            <Console name="Console" target="SYSTEM_OUT">
                <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
            </Console>
        </Appenders>
        <Loggers>
            <Root level="info">
                <AppenderRef ref="Console"/>
            </Root>
        </Loggers>
    </Configuration>
```

---

## 3. ใช้ ConfigMap ใน Pod

### วิธีที่ 1: Environment Variables (envFrom)

```yaml
# pod-env-from-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: myapp:latest
    # โหลด ConfigMap ทั้งหมดเป็น Environment Variables
    envFrom:
    - configMapRef:
        name: webapp-config       # ทุก key จาก ConfigMap กลายเป็น ENV
        optional: false           # fail ถ้าไม่มี ConfigMap
    - configMapRef:
        name: feature-flags       # โหลดจากหลาย ConfigMaps
        optional: true            # ไม่ fail ถ้าไม่มี
    
    # หรือเพิ่ม prefix
    envFrom:
    - prefix: "APP_"             # key กลายเป็น APP_DB_HOST, APP_LOG_LEVEL
      configMapRef:
        name: webapp-config
```

### วิธีที่ 2: Environment Variables (valueFrom)

```yaml
# pod-env-value-from-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-selective-config
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # เลือก specific keys จาก ConfigMap
    - name: DATABASE_HOST        # ENV name ใน container
      valueFrom:
        configMapKeyRef:
          name: webapp-config    # ชื่อ ConfigMap
          key: DB_HOST           # key ใน ConfigMap
    
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: webapp-config
          key: DB_PORT
    
    - name: APP_LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: webapp-config
          key: LOG_LEVEL
          optional: true         # ไม่ fail ถ้าไม่มี key นี้
    
    # ผสมกับ hardcoded ENV
    - name: APP_NAME
      value: "my-application"
    
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
```

### วิธีที่ 3: Volume Mount (ไฟล์)

```yaml
# pod-volume-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config-files
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    # Mount ทั้ง ConfigMap เป็น directory
    - name: nginx-config
      mountPath: /etc/nginx/conf.d    # directory ใน container
      readOnly: true
    
    # Mount ไฟล์เดียว
    - name: app-config
      mountPath: /app/config/app.properties  # path ไฟล์
      subPath: app.properties                # key ใน ConfigMap
      readOnly: true
    
    # Mount ไฟล์อีกตัว
    - name: app-config
      mountPath: /app/config/log4j2.xml
      subPath: log4j2.xml
      readOnly: true
  
  volumes:
  # Volume จาก ConfigMap
  - name: nginx-config
    configMap:
      name: webapp-config
      items:
      - key: nginx.conf          # key ใน ConfigMap
        path: default.conf       # ชื่อไฟล์ใน volume
      defaultMode: 0644          # file permissions
  
  - name: app-config
    configMap:
      name: webapp-config
      # ถ้าไม่ระบุ items จะ mount ทุก keys เป็นไฟล์
      defaultMode: 0644
```

### วิธีที่ 4: Command Arguments

```yaml
# pod-args-from-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-args
spec:
  containers:
  - name: app
    image: myapp:latest
    # ใช้ ENV จาก ConfigMap ใน command args
    env:
    - name: APP_PORT
      valueFrom:
        configMapKeyRef:
          name: webapp-config
          key: APP_PORT
    command: ["./myapp"]
    args:
    - "--port=$(APP_PORT)"      # ใช้ ENV variable ใน args
    - "--config=/etc/config/app.properties"
    volumeMounts:
    - name: config
      mountPath: /etc/config
  
  volumes:
  - name: config
    configMap:
      name: webapp-config
```

### Hot Reload: ConfigMap อัปเดตอัตโนมัติ

```yaml
# ถ้า mount ConfigMap เป็น Volume → Kubernetes จะ update ไฟล์อัตโนมัติ
# (ใช้เวลาประมาณ 1-2 นาที)
# แต่ ENV variables จาก envFrom → ไม่ update (ต้อง restart Pod)

# สำหรับ ENV → ใช้ Deployment + trigger restart
kubectl rollout restart deployment/webapp  # บังคับ reload config

# ดู ConfigMap changes propagation
kubectl exec my-pod -- watch -n 5 'cat /etc/config/app.properties'
```

---

## 4. ConfigMap Best Practices

### Organization Best Practices

```yaml
# ✓ ดี: แยก ConfigMap ตาม concern
# webapp-app-config.yaml - application settings
# webapp-db-config.yaml - database settings
# webapp-nginx-config.yaml - nginx configuration

# ✓ ดี: ตั้งชื่อที่สื่อความหมาย
# app-config (ทั่วไป)
# nginx-config (เฉพาะ)
# feature-flags (feature flags)

# ✓ ดี: ใช้ namespace เพื่อแยก environments
# ConfigMap ใน namespace: production
# ConfigMap ใน namespace: staging
# (ชื่อเดียวกัน แต่ค่าต่างกัน)

# ✓ ดี: Version ConfigMaps
# webapp-config-v1 → webapp-config-v2
# หรือใช้ immutable ConfigMaps
```

### Immutable ConfigMaps (Kubernetes 1.21+)

```yaml
# immutable-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
  namespace: production
data:
  APP_VERSION: "1.0"
  DB_HOST: "postgres-v1.prod.svc.cluster.local"
immutable: true    # ไม่สามารถแก้ไขได้!
                   # ต้องลบแล้วสร้างใหม่เท่านั้น
                   # ดีสำหรับ production safety
                   # ดีสำหรับ performance (kube-apiserver ไม่ต้อง watch)
```

### Size Limits

```bash
# ConfigMap มีขนาดสูงสุด 1MB
# ถ้าใหญ่กว่า → ใช้ External Config Store:
# - AWS Parameter Store
# - HashiCorp Vault
# - etcd (โดยตรง)
# - ConfigServer (Spring Cloud)

# ตรวจสอบขนาด ConfigMap
kubectl get configmap my-config -o yaml | wc -c
```

---

## 5. Workshop: Config แอปพลิเคชัน Production

### สถานการณ์

เรามี Web Application ที่ต้องการ:
1. Application configuration (port, log level, features)
2. Database configuration
3. Nginx reverse proxy configuration
4. Feature flags

### Workshop Setup

```bash
kubectl create namespace configmap-workshop
kubectl config set-context --current --namespace=configmap-workshop
```

### Lab 1: สร้าง ConfigMaps

```bash
# สร้าง ConfigMap 1: Application Config
cat <<'EOF' > /tmp/app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: configmap-workshop
  labels:
    app: webapp
    type: application-config
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  APP_HOST: "0.0.0.0"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  MAX_REQUEST_SIZE: "10mb"
  REQUEST_TIMEOUT: "30s"
  CORS_ORIGINS: "https://app.example.com,https://admin.example.com"
EOF

# สร้าง ConfigMap 2: Database Config
cat <<'EOF' > /tmp/db-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: configmap-workshop
  labels:
    app: webapp
    type: database-config
data:
  DB_HOST: "postgres.configmap-workshop.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "webapp_db"
  DB_POOL_MIN: "5"
  DB_POOL_MAX: "20"
  DB_IDLE_TIMEOUT: "600000"
  DB_CONNECTION_TIMEOUT: "30000"
  DB_SSL_MODE: "require"
EOF

# สร้าง ConfigMap 3: Feature Flags
cat <<'EOF' > /tmp/feature-flags-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
  namespace: configmap-workshop
  labels:
    app: webapp
    type: feature-flags
  annotations:
    description: "Feature flags configuration"
    owner: "product-team"
data:
  FEATURE_DARK_MODE: "true"
  FEATURE_NEW_CHECKOUT: "false"
  FEATURE_ANALYTICS: "true"
  FEATURE_NOTIFICATIONS: "true"
  FEATURE_BETA_API: "false"
  FEATURE_MAINTENANCE_MODE: "false"
  ROLLOUT_PERCENTAGE: "100"
EOF

# สร้าง ConfigMap 4: Nginx Config File
cat <<'EOF' > /tmp/nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: configmap-workshop
  labels:
    app: webapp
    type: nginx-config
data:
  nginx.conf: |
    worker_processes auto;
    worker_rlimit_nofile 65535;
    
    events {
        worker_connections 1024;
        use epoll;
        multi_accept on;
    }
    
    http {
        include mime.types;
        default_type application/octet-stream;
        
        # Logging
        log_format json_combined escape=json
            '{'
            '"time":"$time_iso8601",'
            '"remote_addr":"$remote_addr",'
            '"request":"$request",'
            '"status":"$status",'
            '"body_bytes_sent":"$body_bytes_sent",'
            '"request_time":"$request_time",'
            '"http_user_agent":"$http_user_agent"'
            '}';
        
        access_log /var/log/nginx/access.log json_combined;
        error_log /var/log/nginx/error.log warn;
        
        # Performance
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        
        # Gzip
        gzip on;
        gzip_types text/plain text/css application/json application/javascript;
        gzip_min_length 1000;
        
        # Security Headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        
        server {
            listen 80;
            
            # Health check endpoint
            location /health {
                return 200 'OK';
                add_header Content-Type text/plain;
            }
            
            # Main app
            location / {
                root /usr/share/nginx/html;
                index index.html index.htm;
                try_files $uri $uri/ /index.html;
            }
        }
    }
  
  default.conf: |
    server {
        listen 80 default_server;
        server_name _;
        
        root /usr/share/nginx/html;
        
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        location /health {
            return 200 '{"status":"OK"}';
            add_header Content-Type application/json;
        }
    }
EOF

# Apply ทั้งหมด
kubectl apply -f /tmp/app-configmap.yaml
kubectl apply -f /tmp/db-configmap.yaml
kubectl apply -f /tmp/feature-flags-configmap.yaml
kubectl apply -f /tmp/nginx-configmap.yaml

# ดู ConfigMaps
kubectl get configmaps
kubectl describe configmap app-config
```

### Lab 2: Deploy Application ที่ใช้ ConfigMaps

```bash
cat <<'EOF' > /tmp/webapp-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: configmap-workshop
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: http
        
        # Method 1: โหลด ConfigMap ทั้งหมดเป็น ENV
        envFrom:
        - configMapRef:
            name: app-config
        - configMapRef:
            name: feature-flags
        
        # Method 2: เลือก specific keys
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_NAME
        
        # Resources
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Liveness & Readiness
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
        
        # Method 3: Mount ConfigMap เป็น Volume
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
          readOnly: true
      
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
          defaultMode: 0644
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: configmap-workshop
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f /tmp/webapp-with-config.yaml

# รอ Deployment พร้อม
kubectl rollout status deployment/webapp

# ตรวจสอบว่า ENV variables ถูก inject
POD_NAME=$(kubectl get pods -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- env | grep -E "(APP_|DB_|FEATURE_|LOG_)"

# ตรวจสอบ nginx config files
kubectl exec $POD_NAME -- cat /etc/nginx/nginx.conf
kubectl exec $POD_NAME -- cat /etc/nginx/conf.d/default.conf

# Port forward และทดสอบ
kubectl port-forward service/webapp-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080/health
kill $PF_PID
```

### Lab 3: Update ConfigMap และ Observe Changes

```bash
# Step 1: ดู current config
POD_NAME=$(kubectl get pods -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- env | grep APP_ENV

# Step 2: อัปเดต ConfigMap
kubectl patch configmap app-config \
    -p '{"data":{"APP_ENV":"maintenance","LOG_LEVEL":"debug"}}'

# Step 3: Volume mount changes (อัตโนมัติ ~1-2 นาที)
# สำหรับ ENV variables ต้อง restart Pod

# Restart deployment เพื่อ reload ENV variables
kubectl rollout restart deployment/webapp
kubectl rollout status deployment/webapp

# ตรวจสอบ
POD_NAME=$(kubectl get pods -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- env | grep APP_ENV
# ควรเห็น APP_ENV=maintenance

# Restore
kubectl patch configmap app-config \
    -p '{"data":{"APP_ENV":"production","LOG_LEVEL":"info"}}'
kubectl rollout restart deployment/webapp
```

### Lab 4: สร้าง Immutable ConfigMap

```bash
cat <<'EOF' > /tmp/immutable-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-version-config
  namespace: configmap-workshop
data:
  APP_VERSION: "2.5.1"
  BUILD_DATE: "2024-01-15"
  COMMIT_SHA: "abc123def456"
immutable: true     # ไม่สามารถแก้ไขได้!
EOF

kubectl apply -f /tmp/immutable-config.yaml

# ลองแก้ไข (จะ fail!)
kubectl patch configmap app-version-config \
    -p '{"data":{"APP_VERSION":"3.0.0"}}' || echo "Cannot modify immutable ConfigMap!"

# ดู immutable flag
kubectl get configmap app-version-config -o yaml | grep immutable

# ต้องลบแล้วสร้างใหม่เท่านั้น
# kubectl delete configmap app-version-config
```

### Lab 5: ConfigMap จาก External File

```bash
# สถานการณ์: โหลด config จากไฟล์ JSON

cat <<'EOF' > /tmp/app-settings.json
{
  "server": {
    "port": 8080,
    "host": "0.0.0.0",
    "timeout": 30,
    "maxConnections": 1000
  },
  "database": {
    "host": "postgres.production.svc.cluster.local",
    "port": 5432,
    "name": "webapp_prod",
    "pool": {
      "min": 5,
      "max": 20
    }
  },
  "cache": {
    "host": "redis.production.svc.cluster.local",
    "port": 6379,
    "ttl": 3600
  },
  "features": {
    "darkMode": true,
    "newCheckout": false,
    "analytics": true
  }
}
EOF

# สร้าง ConfigMap จากไฟล์ JSON
kubectl create configmap json-config \
    --from-file=settings.json=/tmp/app-settings.json \
    --namespace=configmap-workshop

# ดู ConfigMap
kubectl describe configmap json-config

# ใช้ใน Pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: json-config-pod
  namespace: configmap-workshop
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'cat /config/settings.json && sleep 3600']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: json-config
EOF

kubectl wait --for=condition=Ready pod/json-config-pod --timeout=60s
kubectl logs json-config-pod
kubectl delete pod json-config-pod
```

### Cleanup Workshop

```bash
# ลบทุกอย่าง
kubectl delete namespace configmap-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/app-configmap.yaml /tmp/db-configmap.yaml \
    /tmp/feature-flags-configmap.yaml /tmp/nginx-configmap.yaml \
    /tmp/webapp-with-config.yaml /tmp/immutable-config.yaml \
    /tmp/app-settings.json /tmp/app.conf /tmp/nginx.conf \
    /tmp/app.env
```

### ConfigMap Command Reference

```bash
# สร้าง ConfigMap
kubectl create configmap NAME --from-literal=key=value
kubectl create configmap NAME --from-file=key=file.txt
kubectl create configmap NAME --from-env-file=.env
kubectl apply -f configmap.yaml

# ดู ConfigMap
kubectl get configmaps
kubectl get cm NAME -o yaml
kubectl describe cm NAME

# แก้ไข ConfigMap
kubectl edit configmap NAME
kubectl patch configmap NAME -p '{"data":{"key":"newvalue"}}'

# ลบ ConfigMap
kubectl delete configmap NAME

# บันทึกเป็น file
kubectl get configmap NAME -o yaml > configmap.yaml
```

---

## สรุป

ConfigMap เป็น object สำคัญสำหรับจัดการ configuration ใน Kubernetes:

1. **Decouple config จาก image**: เปลี่ยน config ได้โดยไม่ rebuild
2. **Multiple consumption methods**: ENV vars, Volume mounts, CLI args
3. **Namespace-scoped**: แยก config ต่อ environment ได้ง่าย
4. **Immutable option**: ป้องกัน accidental changes ใน production

**เมื่อไหร่ใช้ ConfigMap:**
- Application configuration
- Config files (nginx.conf, etc.)
- Feature flags
- Non-sensitive environment variables

**เมื่อไหร่ไม่ใช้ ConfigMap:**
- Passwords, API keys, certificates → ใช้ **Secrets** แทน

ในบทต่อไปเราจะเรียนรู้ **Secrets** ซึ่งใช้เก็บข้อมูล sensitive อย่างปลอดภัย
