# Part 29: Sidecar Containers - Container เสริมที่รันควบคู่

## สารบัญ
1. [Sidecar Pattern คืออะไร](#sidecar-pattern-คืออะไร)
2. [Use Cases](#use-cases)
3. [Sidecar YAML ละเอียด](#sidecar-yaml-ละเอียด)
4. [Log Shipping Sidecar](#log-shipping-sidecar)
5. [Proxy Sidecar (Envoy/Nginx)](#proxy-sidecar)
6. [Auth Sidecar](#auth-sidecar)
7. [Metrics Collection Sidecar](#metrics-collection-sidecar)
8. [Workshop: Log Shipping ด้วย Sidecar](#workshop-log-shipping-ด้วย-sidecar)
9. [Workshop: Service Mesh (Envoy Sidecar)](#workshop-service-mesh)
10. [Workshop: SSL Termination Sidecar](#workshop-ssl-termination-sidecar)
11. [Sidecar vs Init Container](#sidecar-vs-init-container)
12. [Native Sidecar (Kubernetes 1.29+)](#native-sidecar)
13. [Best Practices](#best-practices)

---

## Sidecar Pattern คืออะไร

**Sidecar** คือ design pattern ที่เพิ่ม functionality ให้ main container โดยรัน **container เสริมใน Pod เดียวกัน**:

- Sidecar และ main container **share** network namespace, storage, process namespace
- ทำงาน**พร้อมกัน** (ไม่ใช่ sequential เหมือน init containers)
- เสริมความสามารถโดยไม่แก้ไข main application code

### แนวคิดของ Sidecar Pattern

```
Traditional Approach: รวมทุกอย่างใน single container
──────────────────────────────────────────────────────
  ┌───────────────────────────────────────┐
  │             Application               │
  │  ┌─────────────────────────────────┐  │
  │  │  Business Logic                 │  │
  │  │  + Logging                      │  │
  │  │  + Metrics                      │  │
  │  │  + Auth                         │  │
  │  │  + Proxy                        │  │
  │  └─────────────────────────────────┘  │
  └───────────────────────────────────────┘
  ปัญหา: ซับซ้อน, coupling สูง, update ยาก

Sidecar Pattern: แยก concerns
──────────────────────────────────────────────────────
  ┌───────────────────────────────────────────────────┐
  │                      Pod                         │
  │  ┌─────────────────┐  ┌─────────────────────┐   │
  │  │  Main Container │  │  Sidecar Container  │   │
  │  │  Business Logic │◄─┤  Logging/Metrics/   │   │
  │  │                 │  │  Auth/Proxy         │   │
  │  └─────────────────┘  └─────────────────────┘   │
  │         shared volumes, network, process          │
  └───────────────────────────────────────────────────┘
  ข้อดี: separation of concerns, reusable, update อิสระ
```

---

## Use Cases

### 1. Log Collection (Log Shipping)

```
Main App → writes logs to /var/log/app.log
Sidecar (Fluentd/Filebeat) → reads /var/log/app.log → sends to Elasticsearch
```

### 2. Proxy/Service Mesh

```
Client → Sidecar (Envoy/Nginx) → Main App
          ↑ handles: TLS, retries, circuit breaker, rate limiting
```

### 3. Authentication/Authorization

```
Request → Sidecar (Auth proxy) → Main App (authenticated)
           ↑ validates JWT, checks permissions
```

### 4. Metrics Collection

```
Main App → exposes metrics on non-standard format
Sidecar (Prometheus Exporter) → transforms to Prometheus format
```

### 5. Certificate Management

```
Sidecar (cert-manager agent) → renews SSL certificates
                              → writes to shared volume
Main App → reads certificate from shared volume
```

### 6. Configuration Hot-reload

```
Sidecar → watches ConfigMap changes → reloads config file
Main App → reads updated config without restart
```

---

## Sidecar YAML ละเอียด

### Basic Sidecar

```yaml
# basic-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
  namespace: production
spec:
  # Shared volumes ระหว่าง containers
  volumes:
  - name: shared-logs
    emptyDir: {}
  
  containers:
  # Main Application Container
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 8080
    
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
    
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
  
  # Sidecar Container
  - name: log-shipper
    image: fluent/fluentd:v1.15
    
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
      readOnly: true
    
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
```

### Sidecar แบบสมบูรณ์พร้อม Multiple Sidecars

```yaml
# full-sidecar-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      volumes:
      # Shared log directory
      - name: app-logs
        emptyDir: {}
      # Nginx config
      - name: nginx-config
        configMap:
          name: nginx-sidecar-config
      # SSL certificates
      - name: tls-certs
        secret:
          secretName: app-tls-cert
      
      containers:
      # ── Main Application ────────────────────────────────
      - name: app
        image: my-python-app:latest
        ports:
        - containerPort: 5000
          name: http
        
        env:
        - name: LOG_FILE
          value: "/var/log/app/app.log"
        - name: LOG_LEVEL
          value: "INFO"
        
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
        
        resources:
          requests:
            cpu: "300m"
            memory: "256Mi"
          limits:
            cpu: "600m"
            memory: "512Mi"
        
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 15
      
      # ── Sidecar 1: Nginx Proxy ────────────────────────────
      - name: nginx-proxy
        image: nginx:1.21
        ports:
        - containerPort: 443
          name: https
        - containerPort: 80
          name: http
        
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: tls-certs
          mountPath: /etc/nginx/ssl
          readOnly: true
        
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          periodSeconds: 10
      
      # ── Sidecar 2: Log Shipper ────────────────────────────
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.15
        
        env:
        - name: FLUENTD_CONF
          value: "fluent.conf"
        
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
      
      # ── Sidecar 3: Metrics Exporter ──────────────────────
      - name: metrics-exporter
        image: prom/statsd-exporter:latest
        ports:
        - containerPort: 9090
          name: metrics
        - containerPort: 9125
          name: statsd
          protocol: UDP
        
        resources:
          requests:
            cpu: "25m"
            memory: "32Mi"
          limits:
            cpu: "50m"
            memory: "64Mi"
```

---

## Log Shipping Sidecar

### Architecture

```
Application Container
  └── writes → /var/log/app/*.log (shared volume)

Fluentd Sidecar
  └── reads ← /var/log/app/*.log
  └── processes logs
  └── sends → Elasticsearch/Loki/S3
```

### ConfigMap สำหรับ Fluentd Sidecar

```yaml
# fluentd-sidecar-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-sidecar-config
  namespace: production
data:
  fluent.conf: |
    # Input: อ่าน log files จาก shared volume
    <source>
      @type tail
      path /var/log/app/*.log
      pos_file /tmp/fluentd.pos
      tag app.log
      read_from_head true
      
      <parse>
        @type multi_format
        
        <pattern>
          format json
        </pattern>
        
        <pattern>
          format /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) \[(?<level>[^\]]+)\] (?<message>.*)$/
          time_format %Y-%m-%d %H:%M:%S
        </pattern>
        
        <pattern>
          format none
        </pattern>
      </parse>
    </source>
    
    # Filter: เพิ่ม pod metadata
    <filter app.log>
      @type record_transformer
      <record>
        pod_name "#{ENV['POD_NAME']}"
        pod_namespace "#{ENV['POD_NAMESPACE']}"
        node_name "#{ENV['NODE_NAME']}"
        container_name "app"
        app_name "#{ENV['APP_NAME']}"
      </record>
    </filter>
    
    # Filter: แยก level
    <filter app.log>
      @type grep
      <regexp>
        key level
        pattern /ERROR|WARN|INFO|DEBUG/
      </regexp>
    </filter>
    
    # Output: Elasticsearch
    <match app.log>
      @type elasticsearch
      host "#{ENV['ELASTICSEARCH_HOST']}"
      port "#{ENV['ELASTICSEARCH_PORT']}"
      logstash_format true
      logstash_prefix app-logs
      
      <buffer>
        @type file
        path /tmp/fluentd-buffer
        flush_interval 5s
      </buffer>
    </match>
```

### Deployment พร้อม Log Shipping Sidecar

```yaml
# app-with-log-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      volumes:
      - name: app-logs
        emptyDir: {}
      - name: fluentd-config
        configMap:
          name: fluentd-sidecar-config
      
      containers:
      # Main Application
      - name: app
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import logging
          import time
          import os
          from http.server import HTTPServer, BaseHTTPRequestHandler
          
          # Setup logging ไปยัง file
          log_dir = os.environ.get('LOG_DIR', '/var/log/app')
          os.makedirs(log_dir, exist_ok=True)
          
          logging.basicConfig(
              level=logging.INFO,
              format='%(asctime)s [%(levelname)s] %(message)s',
              handlers=[
                  logging.FileHandler(f'{log_dir}/app.log'),
                  logging.StreamHandler()
              ]
          )
          logger = logging.getLogger(__name__)
          
          class AppHandler(BaseHTTPRequestHandler):
              def do_GET(self):
                  logger.info(f"Request: {self.path} from {self.client_address[0]}")
                  self.send_response(200)
                  self.end_headers()
                  self.wfile.write(b'Hello World')
              
              def log_message(self, format, *args):
                  pass
          
          logger.info("Application starting...")
          HTTPServer(('', 8080), AppHandler).serve_forever()
        
        ports:
        - containerPort: 8080
        
        env:
        - name: LOG_DIR
          value: "/var/log/app"
        
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
        
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "400m"
            memory: "256Mi"
      
      # Fluentd Sidecar
      - name: fluentd
        image: fluent/fluentd:v1.15
        
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: APP_NAME
          value: "web-app"
        - name: ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: ELASTICSEARCH_PORT
          value: "9200"
        
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc/fluent.conf
          subPath: fluent.conf
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
```

---

## Proxy Sidecar

### Nginx Reverse Proxy Sidecar

```yaml
# nginx-proxy-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-sidecar-config
  namespace: production
data:
  nginx.conf: |
    events {
      worker_connections 1024;
    }
    
    http {
      # Upstream: main application
      upstream app {
        server localhost:5000;
      }
      
      # HTTP → HTTPS redirect
      server {
        listen 80;
        return 301 https://$host$request_uri;
      }
      
      # HTTPS server
      server {
        listen 443 ssl http2;
        
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        
        # Security headers
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000";
        
        # Rate limiting
        limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
        
        # Proxy pass
        location / {
          limit_req zone=api burst=20 nodelay;
          
          proxy_pass http://app;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          
          # Timeout
          proxy_connect_timeout 60s;
          proxy_send_timeout 60s;
          proxy_read_timeout 60s;
        }
        
        # Health check endpoint
        location /health {
          proxy_pass http://app/health;
          access_log off;
        }
        
        # Metrics endpoint (internal only)
        location /metrics {
          allow 10.0.0.0/8;
          deny all;
          proxy_pass http://app/metrics;
        }
      }
    }
```

---

## Auth Sidecar

### OAuth2 Proxy Sidecar

```yaml
# oauth2-proxy-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: protected-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: protected-app
  template:
    metadata:
      labels:
        app: protected-app
    spec:
      containers:
      # Main Application (ไม่รู้เรื่อง auth)
      - name: app
        image: my-internal-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
      
      # Auth Sidecar: OAuth2 Proxy
      - name: oauth2-proxy
        image: quay.io/oauth2-proxy/oauth2-proxy:latest
        args:
        - --provider=google
        - --email-domain=company.com
        - --upstream=http://localhost:8080
        - --http-address=0.0.0.0:4180
        - --cookie-secure=true
        - --cookie-httponly=true
        - --redirect-url=https://app.example.com/oauth2/callback
        
        env:
        - name: OAUTH2_PROXY_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: oauth2-secrets
              key: client-id
        - name: OAUTH2_PROXY_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth2-secrets
              key: client-secret
        - name: OAUTH2_PROXY_COOKIE_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth2-secrets
              key: cookie-secret
        
        ports:
        - containerPort: 4180
          name: oauth2-proxy
        
        livenessProbe:
          httpGet:
            path: /ping
            port: 4180
          periodSeconds: 10
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
```

---

## Metrics Collection Sidecar

### Prometheus JMX Exporter สำหรับ Java Apps

```yaml
# java-app-with-metrics-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      volumes:
      - name: jmx-config
        configMap:
          name: jmx-exporter-config
      
      containers:
      # Java Application
      - name: java-app
        image: my-java-app:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9010
          name: jmx
        
        env:
        - name: JAVA_OPTS
          value: "-Dcom.sun.management.jmxremote.port=9010 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
        
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
      
      # JMX Exporter Sidecar
      - name: jmx-exporter
        image: bitnami/jmx-exporter:latest
        ports:
        - containerPort: 9090
          name: metrics
        
        args:
        - "9090"
        - /config/jmx-config.yaml
        
        volumeMounts:
        - name: jmx-config
          mountPath: /config
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
```

---

## Workshop: Log Shipping ด้วย Sidecar

เป้าหมาย: Deploy application ที่ใช้ Filebeat sidecar ส่ง logs ไปยัง Elasticsearch

### 1. สร้าง Namespace และ Elasticsearch

```bash
kubectl create namespace log-demo

# Deploy Elasticsearch
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: log-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: elasticsearch:8.5.0
        env:
        - name: discovery.type
          value: single-node
        - name: xpack.security.enabled
          value: "false"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        ports:
        - containerPort: 9200
        resources:
          limits:
            memory: "1Gi"
          requests:
            memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: log-demo
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
EOF
```

### 2. สร้าง Filebeat ConfigMap

```yaml
# filebeat-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: log-demo
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      enabled: true
      paths:
      - /var/log/app/*.log
      
      # JSON logging
      json.keys_under_root: true
      json.overwrite_keys: true
      json.add_error_key: true
      
      # Multiline support
      multiline.type: pattern
      multiline.pattern: '^\d{4}-\d{2}-\d{2}'
      multiline.negate: true
      multiline.match: after
      
      fields:
        app: ${APP_NAME}
        environment: ${ENVIRONMENT}
        pod_name: ${POD_NAME}
        pod_namespace: ${POD_NAMESPACE}
    
    processors:
    - add_host_metadata:
        when.not.contains.tags: forwarded
    - add_kubernetes_metadata: ~
    
    output.elasticsearch:
      hosts: ["${ELASTICSEARCH_HOST}:${ELASTICSEARCH_PORT}"]
      index: "app-logs-%{+yyyy.MM.dd}"
      
    setup.ilm.enabled: false
    setup.template.name: "app-logs"
    setup.template.pattern: "app-logs-*"
    
    logging.level: warning
    logging.to_files: false
```

### 3. สร้าง Application Deployment พร้อม Filebeat Sidecar

```yaml
# log-demo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-demo-app
  namespace: log-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: log-demo-app
  template:
    metadata:
      labels:
        app: log-demo-app
    spec:
      volumes:
      - name: app-logs
        emptyDir: {}
      - name: filebeat-config
        configMap:
          name: filebeat-config
      - name: filebeat-data
        emptyDir: {}
      
      containers:
      # ── Main App: สร้าง log entries ──────────────────────
      - name: app
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import logging
          import json
          import time
          import os
          import random
          from datetime import datetime
          
          # JSON formatter
          class JSONFormatter(logging.Formatter):
              def format(self, record):
                  return json.dumps({
                      'timestamp': datetime.utcnow().isoformat() + 'Z',
                      'level': record.levelname,
                      'logger': record.name,
                      'message': record.getMessage(),
                      'module': record.module,
                      'function': record.funcName,
                  })
          
          log_dir = '/var/log/app'
          os.makedirs(log_dir, exist_ok=True)
          
          handler = logging.FileHandler(f'{log_dir}/app.log')
          handler.setFormatter(JSONFormatter())
          
          logger = logging.getLogger('demo_app')
          logger.addHandler(handler)
          logger.setLevel(logging.DEBUG)
          
          events = [
              ('INFO', 'User login', {'user_id': lambda: random.randint(1, 100)}),
              ('INFO', 'API request', {'endpoint': lambda: random.choice(['/api/users', '/api/products', '/api/orders'])}),
              ('WARNING', 'High memory usage', {'percent': lambda: random.randint(70, 95)}),
              ('ERROR', 'Database timeout', {'query': lambda: 'SELECT * FROM orders', 'duration_ms': lambda: random.randint(5000, 10000)}),
              ('DEBUG', 'Cache hit', {'key': lambda: f'user:{random.randint(1,100)}'}),
          ]
          
          logger.info('Application started')
          
          while True:
              level, msg, extra_funcs = random.choice(events)
              extra = {k: v() for k, v in extra_funcs.items()}
              
              log_method = getattr(logger, level.lower())
              log_method(msg, extra=extra)
              
              time.sleep(random.uniform(0.5, 2.0))
        
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
        
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
      
      # ── Filebeat Sidecar ─────────────────────────────────
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.5.0
        args:
        - "-e"
        - "-strict.perms=false"
        
        env:
        - name: APP_NAME
          value: "log-demo-app"
        - name: ENVIRONMENT
          value: "production"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: ELASTICSEARCH_HOST
          value: "elasticsearch.log-demo.svc.cluster.local"
        - name: ELASTICSEARCH_PORT
          value: "9200"
        
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        - name: filebeat-config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
          readOnly: true
        - name: filebeat-data
          mountPath: /usr/share/filebeat/data
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        
        securityContext:
          runAsUser: 0
```

### 4. ทดสอบ Log Shipping

```bash
# Apply ทั้งหมด
kubectl apply -f filebeat-config.yaml
kubectl apply -f log-demo-deployment.yaml

# รอ pods พร้อม
kubectl get pods -n log-demo -w

# ดู logs ของ app container
kubectl logs -n log-demo -l app=log-demo-app -c app --tail=20

# ดู filebeat logs
kubectl logs -n log-demo -l app=log-demo-app -c filebeat --tail=20

# Port-forward ไปยัง Elasticsearch
kubectl port-forward svc/elasticsearch 9200:9200 -n log-demo &

# ตรวจสอบ indices
curl http://localhost:9200/_cat/indices?v | grep app-logs

# Query logs
curl -X GET "http://localhost:9200/app-logs-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match": {
        "level": "ERROR"
      }
    },
    "size": 5,
    "sort": [{"@timestamp": "desc"}]
  }'
```

---

## Workshop: Service Mesh

### Envoy Sidecar Proxy (Manual Injection)

```yaml
# envoy-sidecar-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
  namespace: production
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 15001
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: local_app
              http_filters:
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      
      clusters:
      - name: local_app
        connect_timeout: 0.25s
        type: STATIC
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: local_app
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 127.0.0.1
                    port_value: 8080
    
    admin:
      access_log_path: /tmp/admin_access.log
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9901
```

---

## Workshop: SSL Termination Sidecar

### Application ที่ใช้ Nginx สำหรับ SSL Termination

```yaml
# ssl-termination-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ssl-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ssl-app
  template:
    metadata:
      labels:
        app: ssl-app
    spec:
      volumes:
      - name: nginx-config
        configMap:
          name: ssl-nginx-config
      - name: tls-certs
        secret:
          secretName: app-tls
      
      containers:
      # Main app (HTTP only)
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
      
      # Nginx SSL Termination Sidecar
      - name: nginx-ssl
        image: nginx:1.21
        ports:
        - containerPort: 443
          name: https
        
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: tls-certs
          mountPath: /etc/nginx/ssl
          readOnly: true
        
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
        
        readinessProbe:
          tcpSocket:
            port: 443
          periodSeconds: 5
---
# SSL Service (HTTPS)
apiVersion: v1
kind: Service
metadata:
  name: ssl-app
  namespace: production
spec:
  selector:
    app: ssl-app
  ports:
  - name: https
    port: 443
    targetPort: 443
  - name: http
    port: 80
    targetPort: 8080
```

---

## Sidecar vs Init Container

### เมื่อไรใช้อะไร

| Use Case | Init Container | Sidecar |
|----------|---------------|---------|
| รอ service พร้อม | ✓ | ✗ |
| Database migration | ✓ | ✗ |
| Config generation | ✓ | ✗ |
| Log collection | ✗ | ✓ |
| Proxy/SSL termination | ✗ | ✓ |
| Auth/AuthZ | ✗ | ✓ |
| Metrics collection | ✗ | ✓ |
| Certificate renewal | ✗ | ✓ |

```
Init: รันก่อน main, ทำงานครั้งเดียว, รัน sequential
Sidecar: รันพร้อม main, ทำงานตลอดเวลา, รัน parallel
```

---

## Native Sidecar (Kubernetes 1.29+)

Kubernetes 1.29+ รองรับ **Native Sidecar Containers** ที่:
- ถูกกำหนดใน `initContainers` section แต่มี `restartPolicy: Always`
- เริ่มก่อน main containers แต่รันตลอดเวลา
- หยุดหลัง main containers หยุด (ต่างจาก traditional sidecars)

```yaml
# native-sidecar-k8s-1.29.yaml (ต้องการ feature gate SidecarContainers=true)
spec:
  initContainers:
  # Native Sidecar (ใช้ restartPolicy: Always)
  - name: log-agent
    image: fluent/fluentd:v1.15
    restartPolicy: Always   # ← นี่คือ native sidecar!
    
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
      readOnly: true
    
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
  
  containers:
  - name: app
    image: my-app:latest
    # App เริ่มหลัง log-agent ready
```

**ข้อดีของ Native Sidecar**:
- Sidecar เริ่มก่อน main → ไม่พลาด early logs
- Sidecar หยุดหลัง main → ไม่ตัด logs ก่อนเวลา
- Job pods: sidecar หยุดหลัง job container เสร็จ

---

## Best Practices

### 1. Resource Isolation

```yaml
# กำหนด resources สำหรับ sidecar แยกต่างหาก
# Sidecar ไม่ควรแย่ง resources จาก main app
- name: fluentd-sidecar
  resources:
    requests:
      cpu: "50m"      # น้อย
      memory: "64Mi"  # น้อย
    limits:
      cpu: "100m"
      memory: "128Mi"
```

### 2. Separate Concerns

```yaml
# ดี: แต่ละ sidecar ทำหน้าที่เดียว
- name: log-shipper      # แค่ log shipping
- name: metrics-exporter # แค่ export metrics
- name: cert-manager     # แค่ manage certs

# ไม่ดี: sidecar ทำหลายหน้าที่
- name: do-everything-sidecar  # log + metrics + cert + proxy
```

### 3. Use Volumes for Communication

```yaml
# ดี: ใช้ shared volume สำหรับ data exchange
volumes:
- name: shared-data
  emptyDir: {}

# Main writes to /shared, sidecar reads from /shared
```

### 4. Health Checks

```yaml
# ตั้ง readiness probe สำหรับ sidecar ด้วย
- name: nginx-proxy
  readinessProbe:
    httpGet:
      path: /health
      port: 80
    periodSeconds: 5
```

---

## สรุป

Sidecar Pattern เป็นเครื่องมือทรงพลังสำหรับ:

1. **Log Collection**: Fluentd, Filebeat ส่ง logs ออกจาก app
2. **Proxy**: Nginx, Envoy handle TLS, routing, retries
3. **Auth**: OAuth2 proxy ป้องกัน main app
4. **Metrics**: Prometheus exporters เปลี่ยน format metrics
5. **Config**: Vault agent ดึง secrets อัตโนมัติ

ข้อสำคัญ:
- Sidecar และ main container **share** network, storage
- ใช้ **emptyDir volume** สำหรับ data sharing
- กำหนด **resource limits** สำหรับ sidecar เสมอ
- ใช้ **Native Sidecar** (1.29+) สำหรับ lifecycle ที่ดีกว่า

ในบทต่อไป เราจะเรียนรู้เรื่อง **Multi-Container Pods** - patterns การใช้หลาย containers ใน Pod เดียวกัน
