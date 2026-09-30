# Part 52: ConfigMap Advanced

## บทนำ

ใน Part 51 เราได้เรียนรู้การใช้ ConfigMap พื้นฐาน ในบทนี้เราจะเจาะลึกฟีเจอร์ขั้นสูงของ ConfigMap ได้แก่:
- ConfigMap Immutability (การล็อคไม่ให้แก้ไข)
- ConfigMap as Volume (mount เป็น file)
- Hot-reload ConfigMap (อัพเดตโดยไม่ restart pod)
- Dynamic Configuration Patterns

---

## 52.1 ConfigMap Immutability

ตั้งแต่ Kubernetes 1.21 เราสามารถ mark ConfigMap ให้เป็น immutable ได้ ซึ่งมีข้อดีหลายอย่าง

### ทำไมต้องใช้ Immutable ConfigMap?

1. **Performance**: kube-apiserver ไม่ต้อง watch changes ของ immutable ConfigMap ช่วยลด load
2. **Safety**: ป้องกันการแก้ไขโดยไม่ตั้งใจที่อาจทำให้ app พัง
3. **Versioning**: บังคับให้สร้าง ConfigMap ใหม่แทนการแก้ไขของเดิม

### สร้าง Immutable ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
  namespace: default
  labels:
    app: myapp
    version: "v1"
immutable: true     # ทำให้ ConfigMap นี้ immutable
data:
  APP_ENV: "production"
  APP_VERSION: "1.0.0"
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "info"
```

```bash
kubectl apply -f immutable-configmap.yaml

# พยายามแก้ไข - จะ error
kubectl patch configmap app-config-v1 \
  --type merge \
  -p '{"data":{"LOG_LEVEL":"debug"}}'
# Error: configmap "app-config-v1" is immutable
```

### Versioning Pattern กับ Immutable ConfigMap

```bash
# ไม่แก้ไข ConfigMap เดิม แต่สร้างเวอร์ชันใหม่

# Version 1 (immutable)
kubectl apply -f configmap-v1.yaml

# เมื่อต้องการเปลี่ยน config สร้าง version 2
# configmap-v2.yaml
```

```yaml
# configmap-v2.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2      # ชื่อใหม่
  namespace: default
  labels:
    app: myapp
    version: "v2"
immutable: true
data:
  APP_ENV: "production"
  APP_VERSION: "1.0.1"     # เปลี่ยนแล้ว
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "debug"       # เปลี่ยนแล้ว
```

```bash
kubectl apply -f configmap-v2.yaml

# อัพเดต Deployment ให้ใช้ ConfigMap เวอร์ชันใหม่
kubectl set env deployment/myapp --from=configmap/app-config-v2

# หรือแก้ไข manifest แล้ว apply ใหม่
kubectl rollout restart deployment/myapp

# ลบ ConfigMap เก่าเมื่อไม่ใช้แล้ว
kubectl delete configmap app-config-v1
```

### Canary Deployment ด้วย Immutable ConfigMap

```yaml
# Deployment สำหรับ stable version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  template:
    spec:
      containers:
      - name: myapp
        envFrom:
        - configMapRef:
            name: app-config-v1   # stable config

---
# Deployment สำหรับ canary version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        envFrom:
        - configMapRef:
            name: app-config-v2   # new config
```

---

## 52.2 ConfigMap as Volume

แทนที่จะใช้ ConfigMap เป็น environment variables เราสามารถ mount เป็น files ได้ ซึ่งมีประโยชน์สำหรับ:
- Configuration files (nginx.conf, application.properties)
- Scripts
- Certificate files (non-sensitive)

### ConfigMap สำหรับ Config Files

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  # แต่ละ key จะกลายเป็น filename
  nginx.conf: |
    user nginx;
    worker_processes auto;
    error_log /var/log/nginx/error.log warn;
    pid /var/run/nginx.pid;
    
    events {
        worker_connections 1024;
    }
    
    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
        
        access_log /var/log/nginx/access.log main;
        
        sendfile on;
        keepalive_timeout 65;
        
        server {
            listen 80;
            server_name localhost;
            root /usr/share/nginx/html;
            index index.html;
            
            location / {
                try_files $uri $uri/ /index.html;
            }
            
            location /health {
                return 200 'OK';
                add_header Content-Type text/plain;
            }
            
            location /metrics {
                stub_status on;
            }
        }
    }
  
  # Config อื่นๆ
  default.conf: |
    server {
        listen 8080;
        location /api {
            proxy_pass http://backend-service:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
  
  # Script file
  startup.sh: |
    #!/bin/bash
    set -e
    echo "Starting application..."
    echo "APP_ENV: $APP_ENV"
    exec nginx -g 'daemon off;'
```

### Mount ConfigMap as Volume

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        
        # Mount volumes
        volumeMounts:
        # Mount nginx.conf
        - name: nginx-config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf          # mount เฉพาะ file เดียว
          readOnly: true
        
        # Mount ทุก file จาก ConfigMap เป็น directory
        - name: nginx-extra-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        
        # Mount scripts
        - name: startup-scripts
          mountPath: /scripts
          readOnly: true
      
      volumes:
      # Volume จาก ConfigMap
      - name: nginx-config-volume
        configMap:
          name: nginx-config
          # defaultMode: 0644  # file permissions
      
      - name: nginx-extra-config
        configMap:
          name: nginx-config
          # เลือก items ที่จะ mount
          items:
          - key: default.conf
            path: default.conf         # filename ใน directory
          - key: startup.sh
            path: startup.sh
            mode: 0755                 # executable
      
      - name: startup-scripts
        configMap:
          name: nginx-config
          items:
          - key: startup.sh
            path: startup.sh
          defaultMode: 0755            # ทุก file เป็น executable
```

### Application Properties ด้วย ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-config
  namespace: default
data:
  application.properties: |
    # Spring Application Properties
    spring.application.name=my-spring-app
    spring.profiles.active=production
    
    # Server
    server.port=8080
    server.servlet.context-path=/api
    
    # Database
    spring.datasource.url=jdbc:postgresql://postgres:5432/mydb
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.hibernate.ddl-auto=validate
    spring.jpa.show-sql=false
    
    # Logging
    logging.level.root=INFO
    logging.level.com.mycompany=DEBUG
    logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
    
    # Actuator
    management.endpoints.web.exposure.include=health,info,metrics
    management.endpoint.health.show-details=when_authorized
    
  application-production.properties: |
    # Production overrides
    spring.jpa.show-sql=false
    logging.level.root=WARN
    
    # Connection pool
    spring.datasource.hikari.maximum-pool-size=20
    spring.datasource.hikari.minimum-idle=5
    spring.datasource.hikari.connection-timeout=30000
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  template:
    spec:
      containers:
      - name: spring-app
        image: my-spring-app:1.0.0
        volumeMounts:
        - name: app-config
          mountPath: /app/config
          readOnly: true
        env:
        - name: SPRING_CONFIG_LOCATION
          value: "file:/app/config/"
      
      volumes:
      - name: app-config
        configMap:
          name: spring-config
```

---

## 52.3 Hot-reload ConfigMap

เมื่อ mount ConfigMap เป็น Volume, Kubernetes จะอัพเดตไฟล์ใน container โดยอัตโนมัติเมื่อ ConfigMap เปลี่ยน (โดยไม่ต้อง restart pod) ซึ่งต่างจากการใช้เป็น environment variables

### วิธีการทำงานของ Hot-reload

```
ConfigMap เปลี่ยน
       ↓
kubelet detect การเปลี่ยนแปลง (ทุก ~1 นาที)
       ↓
อัพเดตไฟล์ใน volume mount ของ Pod
       ↓
Application อ่านไฟล์ใหม่ (ถ้า support hot-reload)
```

**ข้อสำคัญ**: Application ต้องรองรับการอ่าน config file ใหม่ด้วยตัวเอง (เช่น inotify, signal reload)

### Demo: Nginx Hot-reload

```yaml
# hot-reload-demo.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-hot-config
  namespace: default
data:
  default.conf: |
    server {
        listen 80;
        location / {
            return 200 "Version 1\n";
            add_header Content-Type text/plain;
        }
        location /health {
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hot
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-hot
  template:
    metadata:
      labels:
        app: nginx-hot
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        
        # Sidecar container สำหรับ watch และ reload
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "nginx -t && nginx -s reload || true"]
      
      # Sidecar สำหรับ config reload
      - name: config-reloader
        image: alpine:3.18
        command: ["/bin/sh", "-c"]
        args:
        - |
          apk add --no-cache inotify-tools
          while true; do
            inotifywait -e modify,create,delete /etc/nginx/conf.d/ 2>/dev/null
            echo "Config changed, reloading nginx..."
            # ส่ง SIGHUP ไปที่ nginx process
            kill -HUP $(pgrep nginx | head -1) 2>/dev/null || true
            sleep 2
          done
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: nginx-hot-config
```

### ทดสอบ Hot-reload

```bash
kubectl apply -f hot-reload-demo.yaml

# Port forward เพื่อทดสอบ
kubectl port-forward deployment/nginx-hot 8080:80 &

# ทดสอบ response แรก
curl http://localhost:8080
# ได้: Version 1

# อัพเดต ConfigMap
kubectl patch configmap nginx-hot-config \
  --type merge \
  -p '{
    "data": {
      "default.conf": "server {\n    listen 80;\n    location / {\n        return 200 \"Version 2\\n\";\n        add_header Content-Type text/plain;\n    }\n}"
    }
  }'

# รอสักครู่ (~1 นาที สำหรับ kubelet sync)
sleep 70

# ทดสอบอีกครั้ง
curl http://localhost:8080
# ได้: Version 2 (ไม่ต้อง restart pod!)

# ดู config ใน container
kubectl exec -it $(kubectl get pod -l app=nginx-hot -o jsonpath='{.items[0].metadata.name}') -- cat /etc/nginx/conf.d/default.conf
```

### Hot-reload กับ Java/Spring Boot

```yaml
# spring-app ที่รองรับ hot-reload
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-hot-config
data:
  application.properties: |
    spring.application.name=hot-app
    logging.level.root=INFO
    app.feature.enabled=true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-hot-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-spring-app:1.0.0
        env:
        - name: SPRING_CONFIG_LOCATION
          value: "file:/config/"
        # Spring Cloud Config Watcher สำหรับ hot reload
        - name: SPRING_CLOUD_KUBERNETES_CONFIG_RELOAD_ENABLED
          value: "true"
        - name: SPRING_CLOUD_KUBERNETES_CONFIG_RELOAD_STRATEGY
          value: "refresh"
        volumeMounts:
        - name: config-volume
          mountPath: /config
      
      volumes:
      - name: config-volume
        configMap:
          name: spring-hot-config
```

---

## 52.4 Advanced ConfigMap Patterns

### Pattern 1: Environment-Specific ConfigMaps

```yaml
# base-config.yaml (ใช้ร่วมกันทุก environment)
apiVersion: v1
kind: ConfigMap
metadata:
  name: base-config
data:
  APP_NAME: "MyApp"
  LOG_FORMAT: "json"
  TIMEZONE: "Asia/Bangkok"

---
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: development
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  DB_HOST: "postgres-dev.development.svc.cluster.local"
  REPLICAS: "1"

---
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "warn"
  DB_HOST: "postgres-prod.production.svc.cluster.local"
  REPLICAS: "5"
```

```yaml
# deployment.yaml (เหมือนกันทุก environment)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        envFrom:
        - configMapRef:
            name: base-config    # ค่าพื้นฐาน
        - configMapRef:
            name: env-config     # ค่าเฉพาะ environment (override)
```

### Pattern 2: ConfigMap จาก File

```bash
# สร้าง ConfigMap จากไฟล์จริง
kubectl create configmap nginx-config \
  --from-file=nginx.conf=/path/to/nginx.conf \
  --from-file=mime.types=/path/to/mime.types \
  --namespace=default

# สร้างจาก directory
kubectl create configmap app-configs \
  --from-file=/path/to/config/directory/ \
  --namespace=default

# สร้างจาก env file
kubectl create configmap app-env \
  --from-env-file=/path/to/.env \
  --namespace=default
```

### Pattern 3: JSON/YAML ใน ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: json-config
data:
  config.json: |
    {
      "database": {
        "host": "postgres",
        "port": 5432,
        "name": "mydb",
        "poolSize": 10
      },
      "cache": {
        "host": "redis",
        "port": 6379,
        "ttl": 3600
      },
      "features": {
        "darkMode": true,
        "betaFeatures": false
      }
    }
  
  config.yaml: |
    database:
      host: postgres
      port: 5432
      name: mydb
      poolSize: 10
    
    cache:
      host: redis
      port: 6379
      ttl: 3600
    
    features:
      darkMode: true
      betaFeatures: false
```

### Pattern 4: Binary Data ใน ConfigMap

```yaml
# บันทึก binary data ใน ConfigMap (ใช้ binaryData แทน data)
apiVersion: v1
kind: ConfigMap
metadata:
  name: binary-config
data:
  text-config: "some text value"
binaryData:
  # base64 encoded binary
  logo.png: iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==
```

---

## 52.5 Workshop: Dynamic Configuration

### สถานการณ์

เราจะสร้าง web application ที่:
1. ใช้ Nginx เป็น web server
2. Config ถูก mount จาก ConfigMap
3. รองรับ hot-reload เมื่อ config เปลี่ยน
4. มี sidecar container คอย watch และ reload

### Step 1: สร้าง Namespace

```bash
kubectl create namespace workshop-configmap
kubectl config set-context --current --namespace=workshop-configmap
```

### Step 2: สร้าง ConfigMaps

```yaml
# nginx-base-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-base
  namespace: workshop-configmap
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    error_log /var/log/nginx/error.log warn;
    pid /tmp/nginx.pid;
    
    events {
        worker_connections 1024;
    }
    
    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        log_format json_combined escape=json
          '{"time":"$time_local",'
          '"remote_addr":"$remote_addr",'
          '"request":"$request",'
          '"status":$status,'
          '"bytes":$body_bytes_sent}';
        
        access_log /var/log/nginx/access.log json_combined;
        
        sendfile on;
        keepalive_timeout 65;
        
        include /etc/nginx/conf.d/*.conf;
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-site-config
  namespace: workshop-configmap
  annotations:
    config-version: "v1"
    last-updated: "2024-01-15"
data:
  site.conf: |
    server {
        listen 8080;
        server_name _;
        root /usr/share/nginx/html;
        
        # Health check
        location /health {
            return 200 '{"status":"healthy","version":"1.0","config":"v1"}';
            add_header Content-Type application/json;
        }
        
        # API proxy (ตัวอย่าง)
        location /api/ {
            return 200 '{"message":"API endpoint - config v1"}';
            add_header Content-Type application/json;
        }
        
        # Static files
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
  
  # HTML ที่แสดงผล
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Workshop App</title></head>
    <body>
      <h1>Kubernetes ConfigMap Workshop</h1>
      <p>Configuration Version: v1</p>
      <p>This page is served from ConfigMap!</p>
    </body>
    </html>
```

```bash
kubectl apply -f nginx-base-config.yaml
kubectl get configmap -n workshop-configmap
```

### Step 3: สร้าง Deployment พร้อม Config Reloader

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dynamic
  namespace: workshop-configmap
  labels:
    app: nginx-dynamic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-dynamic
  template:
    metadata:
      labels:
        app: nginx-dynamic
    spec:
      # Shared volume สำหรับ nginx pid
      initContainers:
      - name: init-config
        image: busybox:1.36
        command: ['sh', '-c', 'cp /config-base/nginx.conf /etc/nginx-writable/nginx.conf && echo "Init done"']
        volumeMounts:
        - name: nginx-base-config
          mountPath: /config-base
        - name: nginx-writable-config
          mountPath: /etc/nginx-writable
      
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 8080
          name: http
        
        volumeMounts:
        # Mount nginx.conf หลัก
        - name: nginx-base-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
        
        # Mount site configs (hot-reloadable)
        - name: nginx-site-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        
        # Mount HTML files
        - name: nginx-site-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
          readOnly: true
        
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
      
      # Config Reloader Sidecar
      - name: config-reloader
        image: bitnami/nginx:1.25
        command: ["/bin/bash", "-c"]
        args:
        - |
          echo "Config reloader started"
          CHECKSUM=""
          while true; do
            NEW_CHECKSUM=$(md5sum /etc/nginx/conf.d/* 2>/dev/null | md5sum)
            if [ "$CHECKSUM" != "$NEW_CHECKSUM" ]; then
              echo "$(date): Config changed, testing nginx config..."
              if nginx -t 2>/dev/null; then
                echo "$(date): Config valid, reloading nginx..."
                nginx -s reload 2>/dev/null || true
                echo "$(date): Nginx reloaded"
              else
                echo "$(date): Config invalid! Not reloading."
              fi
              CHECKSUM="$NEW_CHECKSUM"
            fi
            sleep 5
          done
        volumeMounts:
        - name: nginx-site-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        
        resources:
          requests:
            memory: "16Mi"
            cpu: "10m"
          limits:
            memory: "32Mi"
            cpu: "50m"
      
      volumes:
      - name: nginx-base-config
        configMap:
          name: nginx-base
      
      - name: nginx-site-config
        configMap:
          name: nginx-site-config
      
      - name: nginx-writable-config
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-dynamic
  namespace: workshop-configmap
spec:
  selector:
    app: nginx-dynamic
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -n workshop-configmap -w
```

### Step 4: ทดสอบการทำงานเบื้องต้น

```bash
# รอ pods พร้อม
kubectl wait --for=condition=ready pod -l app=nginx-dynamic \
  -n workshop-configmap --timeout=60s

# Port forward
kubectl port-forward svc/nginx-dynamic 8080:80 -n workshop-configmap &

# ทดสอบ
curl http://localhost:8080/health
# ได้: {"status":"healthy","version":"1.0","config":"v1"}

curl http://localhost:8080/api/
# ได้: {"message":"API endpoint - config v1"}

curl http://localhost:8080/
# ได้: HTML page ที่ mount จาก ConfigMap
```

### Step 5: ทดสอบ Hot-reload

```bash
# อัพเดต ConfigMap
kubectl patch configmap nginx-site-config \
  -n workshop-configmap \
  --type merge \
  -p '{
    "data": {
      "site.conf": "server {\n    listen 8080;\n    server_name _;\n    root /usr/share/nginx/html;\n    \n    location /health {\n        return 200 '\''{\"status\":\"healthy\",\"version\":\"1.0\",\"config\":\"v2\"}'\''; \n        add_header Content-Type application/json;\n    }\n    \n    location /api/ {\n        return 200 '\''{\"message\":\"API endpoint - config v2 updated!\"}'\''; \n        add_header Content-Type application/json;\n    }\n    \n    location / {\n        try_files $uri $uri/ /index.html;\n    }\n}"
    }
  }'

echo "รอ kubelet sync (~60 วินาที)..."
sleep 70

# ทดสอบอีกครั้ง
curl http://localhost:8080/health
# ได้: config:"v2" (เปลี่ยนแล้ว!)

curl http://localhost:8080/api/
# ได้: config v2 updated!
```

### Step 6: ตรวจสอบ ConfigMap Version

```bash
# ดู ConfigMap version annotation
kubectl get configmap nginx-site-config -n workshop-configmap -o yaml

# ดู config file ใน container โดยตรง
POD=$(kubectl get pod -l app=nginx-dynamic -n workshop-configmap -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -n workshop-configmap -c nginx -- cat /etc/nginx/conf.d/site.conf

# ดู reloader logs
kubectl logs $POD -n workshop-configmap -c config-reloader --tail=20
```

### Step 7: Immutable ConfigMap Pattern ใน Workshop

```bash
# สร้าง immutable ConfigMap version 3
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-site-config-v3
  namespace: workshop-configmap
  labels:
    version: "v3"
immutable: true
data:
  site.conf: |
    server {
        listen 8080;
        server_name _;
        root /usr/share/nginx/html;
        
        location /health {
            return 200 '{"status":"healthy","version":"1.0","config":"v3-immutable"}';
            add_header Content-Type application/json;
        }
        
        location /api/ {
            return 200 '{"message":"API endpoint - immutable config v3"}';
            add_header Content-Type application/json;
        }
        
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Workshop App v3</title></head>
    <body>
      <h1>Kubernetes ConfigMap Workshop</h1>
      <p>Configuration Version: v3 (Immutable)</p>
    </body>
    </html>
EOF

# อัพเดต Deployment ให้ใช้ ConfigMap ใหม่
kubectl patch deployment nginx-dynamic \
  -n workshop-configmap \
  --type json \
  -p '[
    {
      "op": "replace",
      "path": "/spec/template/spec/volumes/1/configMap/name",
      "value": "nginx-site-config-v3"
    }
  ]'

kubectl rollout status deployment/nginx-dynamic -n workshop-configmap

# ทดสอบ
curl http://localhost:8080/health
# ได้: config:"v3-immutable"
```

### Step 8: Monitoring ConfigMap Changes

```bash
# Watch ConfigMap changes
kubectl get configmap -n workshop-configmap -w &

# ดู pod events เมื่อ config reload
kubectl get events -n workshop-configmap --sort-by='.lastTimestamp' --watch &

# ทดสอบการ rollback
kubectl rollout history deployment/nginx-dynamic -n workshop-configmap
kubectl rollout undo deployment/nginx-dynamic -n workshop-configmap
kubectl rollout status deployment/nginx-dynamic -n workshop-configmap
```

### Step 9: ConfigMap Size Limit

```bash
# ConfigMap มี limit 1MB
# ทดสอบดูขนาด
kubectl get configmap nginx-site-config-v3 -n workshop-configmap -o json | wc -c

# ถ้าต้องการ store ข้อมูลใหญ่ ใช้ volume mount จาก PVC แทน
```

### Step 10: Cleanup

```bash
kubectl delete namespace workshop-configmap
kill %1 %2 %3 2>/dev/null  # ยกเลิก port-forward และ watch

echo "Workshop cleanup complete!"
```

---

## 52.6 ConfigMap Best Practices

### 1. ตั้งชื่อให้สื่อความหมาย

```bash
# ดี
app-config
nginx-site-config
database-connection-config

# ไม่ดี
config1
my-config
cfg
```

### 2. ใช้ Labels และ Annotations

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app: myapp
    environment: production
    managed-by: helm
    version: "v2"
  annotations:
    description: "Application configuration for production"
    last-updated: "2024-01-15"
    updated-by: "platform-team"
```

### 3. จำกัด ConfigMap ให้อยู่ใน Namespace

```bash
# ใช้ namespace เพื่อแยก environments
kubectl create configmap app-config \
  --from-literal=ENV=production \
  --namespace=production

kubectl create configmap app-config \
  --from-literal=ENV=staging \
  --namespace=staging
```

### 4. ระวังขนาด ConfigMap

```yaml
# ConfigMap มี limit 1MB ทั้งหมด
# ถ้าข้อมูลใหญ่ ควรใช้ PVC หรือ Object Storage แทน

# สำหรับ config files ขนาดใหญ่
# ใช้ ConfigMap สำหรับ reference URLs แทน
data:
  config-url: "https://config-server.internal/app/config.json"
```

### 5. Version ConfigMap กับ Deployment

```bash
# ใช้ checksum annotation เพื่อ trigger rolling update
# เมื่อ ConfigMap เปลี่ยน

# ใน Helm chart:
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ConfigMap Immutability**: ป้องกันการแก้ไขและ improve performance
2. **ConfigMap as Volume**: mount เป็น files แทน env vars
3. **Hot-reload**: อัพเดต config โดยไม่ต้อง restart pod (สำหรับ volume mounts)
4. **Advanced Patterns**: versioning, environment-specific configs

**Key Takeaways:**
- Hot-reload ใช้ได้เฉพาะ volume-mounted configs (ไม่ใช่ env vars)
- Immutable ConfigMaps ช่วย performance และ safety
- ใช้ versioning pattern สำหรับ production deployments
- Application ต้องรองรับ hot-reload ด้วยตัวเอง

---

**ต่อไป**: Part 53 - Secrets Management (Sealed Secrets, External Secrets Operator)
