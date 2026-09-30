# Part 103: โปรเจค E-Commerce Platform บน Kubernetes

## บทนำ

บทนี้เป็นโปรเจคปฏิบัติจริงที่นำความรู้ทั้งหมดมารวมกัน โดยจะสร้าง E-Commerce Platform แบบ Production-Ready บน Kubernetes ครอบคลุม Frontend, Backend APIs, Database, Cache, และ Message Queue

## สถาปัตยกรรมระบบ

```
┌─────────────────────────────────────────────────────────┐
│                        Internet                          │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                   Load Balancer (Nginx)                  │
│                    myshop.example.com                    │
└───────────┬───────────────┬───────────────┬─────────────┘
            │               │               │
    ┌───────▼──────┐ ┌──────▼─────┐ ┌──────▼─────┐
    │   Frontend   │ │  API GW    │ │  Admin UI  │
    │  (React/Next)│ │  (Kong)    │ │  (React)   │
    └──────────────┘ └──────┬─────┘ └────────────┘
                            │
        ┌───────────────────┼───────────────────────┐
        │                   │                       │
┌───────▼──────┐  ┌─────────▼──────┐  ┌────────────▼───┐
│ Product Svc  │  │  Order Service  │  │  User Service  │
│  (Node.js)   │  │   (Go)          │  │  (Python)      │
└───────┬──────┘  └─────────┬──────┘  └────────────┬───┘
        │                   │                       │
┌───────▼──────┐  ┌─────────▼──────┐  ┌────────────▼───┐
│ Product DB   │  │   Order DB      │  │   User DB      │
│ (PostgreSQL) │  │  (PostgreSQL)   │  │  (PostgreSQL)  │
└──────────────┘  └─────────┬──────┘  └────────────────┘
                             │
                  ┌──────────▼──────────┐
                  │   Message Queue     │
                  │   (RabbitMQ/Kafka)  │
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │   Notification Svc  │
                  │   (Python)          │
                  └─────────────────────┘

Shared Infrastructure:
- Redis Cache Cluster
- Elasticsearch (Search)
- MinIO (Object Storage)
- Prometheus + Grafana (Monitoring)
```

## โครงสร้างโปรเจค

```
ecommerce-k8s/
├── namespaces/
│   ├── namespace-frontend.yaml
│   ├── namespace-backend.yaml
│   └── namespace-infrastructure.yaml
├── infrastructure/
│   ├── postgresql/
│   ├── redis/
│   ├── rabbitmq/
│   ├── elasticsearch/
│   └── minio/
├── services/
│   ├── product-service/
│   ├── order-service/
│   ├── user-service/
│   └── notification-service/
├── frontend/
│   ├── storefront/
│   └── admin/
├── api-gateway/
├── monitoring/
└── security/
```

---

## Step 1: ตั้งค่า Namespaces

```yaml
# namespaces/namespace-frontend.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: frontend
  labels:
    app: ecommerce
    tier: frontend
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
---
# namespaces/namespace-backend.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: backend
  labels:
    app: ecommerce
    tier: backend
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
---
# namespaces/namespace-infrastructure.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: infrastructure
  labels:
    app: ecommerce
    tier: infrastructure
```

```bash
kubectl apply -f namespaces/
```

---

## Step 2: ติดตั้ง Infrastructure

### 2.1 PostgreSQL (Bitnami Chart)

```bash
# เพิ่ม Bitnami Repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# ติดตั้ง PostgreSQL สำหรับ Product Service
helm install product-db bitnami/postgresql \
  -n infrastructure \
  --set auth.postgresPassword=pgpassword \
  --set auth.database=productdb \
  --set primary.persistence.size=20Gi \
  --set readReplicas.replicaCount=2 \
  --set metrics.enabled=true
```

```yaml
# infrastructure/postgresql/product-db-values.yaml
auth:
  postgresPassword: "pgpassword"
  username: "productuser"
  password: "productpass"
  database: "productdb"

primary:
  persistence:
    enabled: true
    size: 20Gi
    storageClass: "fast"
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 4Gi

readReplicas:
  replicaCount: 2
  persistence:
    enabled: true
    size: 20Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

backup:
  enabled: true
  cronjob:
    schedule: "0 2 * * *"
    storage:
      size: 50Gi
```

### 2.2 Redis Cluster

```yaml
# infrastructure/redis/redis-values.yaml
architecture: replication

auth:
  enabled: true
  password: "redispassword"

master:
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 2Gi
  persistence:
    enabled: true
    size: 10Gi

replica:
  replicaCount: 2
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 2Gi
  persistence:
    enabled: true
    size: 10Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

```bash
helm install redis bitnami/redis \
  -n infrastructure \
  -f infrastructure/redis/redis-values.yaml
```

### 2.3 RabbitMQ

```yaml
# infrastructure/rabbitmq/values.yaml
auth:
  username: admin
  password: rabbitmqpassword
  erlangCookie: secretcookie

clustering:
  enabled: true
  replicaCount: 3

persistence:
  enabled: true
  size: 10Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

extraPlugins: "rabbitmq_management rabbitmq_shovel rabbitmq_shovel_management"
```

```bash
helm install rabbitmq bitnami/rabbitmq \
  -n infrastructure \
  -f infrastructure/rabbitmq/values.yaml
```

### 2.4 Elasticsearch

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch elastic/elasticsearch \
  -n infrastructure \
  --set replicas=3 \
  --set minimumMasterNodes=2 \
  --set resources.requests.cpu=500m \
  --set resources.requests.memory=1Gi \
  --set persistence.size=30Gi
```

### 2.5 MinIO (Object Storage)

```bash
helm repo add minio https://charts.min.io/

helm install minio minio/minio \
  -n infrastructure \
  --set replicas=4 \
  --set persistence.size=100Gi \
  --set rootUser=minioadmin \
  --set rootPassword=miniopassword \
  --set mode=distributed \
  --set metrics.serviceMonitor.enabled=true
```

---

## Step 3: Services

### 3.1 Product Service

```yaml
# services/product-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: backend
  labels:
    app: product-service
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: product-service
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: product-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: product-service
        image: company-registry.io/product-service:1.0.0
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: NODE_ENV
          value: production
        - name: PORT
          value: "3000"
        - name: DB_HOST
          value: product-db-postgresql.infrastructure.svc.cluster.local
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: productdb
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: product-db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: product-db-credentials
              key: password
        - name: REDIS_HOST
          value: redis-master.infrastructure.svc.cluster.local
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: password
        - name: ELASTICSEARCH_URL
          value: http://elasticsearch-master.infrastructure.svc.cluster.local:9200
        - name: RABBITMQ_URL
          valueFrom:
            secretKeyRef:
              name: rabbitmq-credentials
              key: url
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: "1"
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: product-images
          mountPath: /uploads
      volumes:
      - name: tmp
        emptyDir: {}
      - name: product-images
        persistentVolumeClaim:
          claimName: product-images-pvc
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: product-service
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: backend
spec:
  selector:
    app: product-service
  ports:
  - name: http
    port: 80
    targetPort: 3000
  - name: metrics
    port: 9090
    targetPort: 9090
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: product-service-pdb
  namespace: backend
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: product-service
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 3
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

### 3.2 Order Service

```yaml
# services/order-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: order-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: order-service
        image: company-registry.io/order-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: DB_DSN
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: dsn
        - name: RABBITMQ_URL
          valueFrom:
            secretKeyRef:
              name: rabbitmq-credentials
              key: url
        - name: PRODUCT_SERVICE_URL
          value: http://product-service.backend.svc.cluster.local
        - name: USER_SERVICE_URL
          value: http://user-service.backend.svc.cluster.local
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: backend
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
```

### 3.3 User Service

```yaml
# services/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: company-registry.io/user-service:1.0.0
        ports:
        - containerPort: 5000
        env:
        - name: FLASK_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: user-db-credentials
              key: url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        - name: REDIS_URL
          value: redis://:$(REDIS_PASSWORD)@redis-master.infrastructure.svc.cluster.local:6379
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: password
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 20
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: backend
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 5000
```

### 3.4 Notification Service

```yaml
# services/notification-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
    spec:
      containers:
      - name: notification-service
        image: company-registry.io/notification-service:1.0.0
        env:
        - name: RABBITMQ_URL
          valueFrom:
            secretKeyRef:
              name: rabbitmq-credentials
              key: url
        - name: SMTP_HOST
          value: smtp.sendgrid.net
        - name: SMTP_PORT
          value: "587"
        - name: SMTP_USER
          valueFrom:
            secretKeyRef:
              name: smtp-credentials
              key: username
        - name: SMTP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: smtp-credentials
              key: password
        - name: FROM_EMAIL
          value: no-reply@myshop.com
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

---

## Step 4: Frontend

```yaml
# frontend/storefront/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storefront
  namespace: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: storefront
  template:
    metadata:
      labels:
        app: storefront
    spec:
      containers:
      - name: storefront
        image: company-registry.io/storefront:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NEXT_PUBLIC_API_URL
          value: https://api.myshop.example.com
        - name: NEXTAUTH_URL
          value: https://myshop.example.com
        - name: NEXTAUTH_SECRET
          valueFrom:
            secretKeyRef:
              name: nextauth-secret
              key: secret
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: "1"
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: storefront
  namespace: frontend
spec:
  selector:
    app: storefront
  ports:
  - port: 80
    targetPort: 3000
```

---

## Step 5: Ingress และ API Gateway

```yaml
# api-gateway/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: frontend
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myshop.example.com
    - api.myshop.example.com
    secretName: myshop-tls
  rules:
  - host: myshop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: storefront
            port:
              number: 80
  - host: api.myshop.example.com
    http:
      paths:
      - path: /v1/products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 80
      - path: /v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

---

## Step 6: Network Policies

```yaml
# security/network-policies.yaml

# Backend: Default Deny All
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
---
# Allow Frontend to Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: product-service
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - port: 3000
---
# Allow Inter-service Communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: backend
      podSelector:
        matchLabels:
          tier: backend
---
# Allow Backend to Infrastructure
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-infrastructure
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: infrastructure
    ports:
    - port: 5432  # PostgreSQL
    - port: 6379  # Redis
    - port: 5672  # RabbitMQ
    - port: 9200  # Elasticsearch
    - port: 9000  # MinIO
```

---

## Step 7: Secrets Management

```bash
# สร้าง Secrets สำหรับทุก Service
# Product DB
kubectl create secret generic product-db-credentials \
  -n backend \
  --from-literal=username=productuser \
  --from-literal=password=productpass

# Order DB
kubectl create secret generic order-db-credentials \
  -n backend \
  --from-literal=dsn="host=order-db-postgresql.infrastructure port=5432 user=orderuser password=orderpass dbname=orderdb sslmode=require"

# User DB
kubectl create secret generic user-db-credentials \
  -n backend \
  --from-literal=url="postgresql://userservice:userpass@user-db-postgresql.infrastructure:5432/userdb"

# Redis
kubectl create secret generic redis-credentials \
  -n backend \
  --from-literal=password=redispassword

# RabbitMQ
kubectl create secret generic rabbitmq-credentials \
  -n backend \
  --from-literal=url="amqp://admin:rabbitmqpassword@rabbitmq.infrastructure.svc.cluster.local:5672"

# JWT
kubectl create secret generic jwt-secret \
  -n backend \
  --from-literal=secret=$(openssl rand -base64 32)

# SMTP
kubectl create secret generic smtp-credentials \
  -n backend \
  --from-literal=username=apikey \
  --from-literal=password=SG.xxxxx

# NextAuth
kubectl create secret generic nextauth-secret \
  -n frontend \
  --from-literal=secret=$(openssl rand -base64 32)
```

---

## Step 8: Monitoring

```yaml
# monitoring/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ecommerce-alerts
  namespace: monitoring
spec:
  groups:
  - name: ecommerce.critical
    rules:
    - alert: ProductServiceDown
      expr: up{job="product-service"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Product Service is down"
        description: "Product Service has been down for more than 1 minute"

    - alert: OrderServiceHighLatency
      expr: |
        histogram_quantile(0.99, 
          rate(http_request_duration_seconds_bucket{job="order-service"}[5m])
        ) > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Order Service high latency"
        description: "P99 latency is {{ $value }}s for the last 5 minutes"

    - alert: DatabaseConnectionPoolExhausted
      expr: |
        pg_stat_database_numbackends / pg_settings_max_connections > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Database connection pool almost exhausted"

    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m]) /
        rate(http_requests_total[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }}"

    - alert: LowOrderConversionRate
      expr: |
        rate(orders_completed_total[1h]) /
        rate(checkout_started_total[1h]) < 0.5
      for: 30m
      labels:
        severity: warning
      annotations:
        summary: "Low order conversion rate"
```

```yaml
# monitoring/grafana-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerce-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  ecommerce.json: |
    {
      "title": "E-Commerce Overview",
      "panels": [
        {
          "title": "Request Rate",
          "type": "graph",
          "targets": [
            {
              "expr": "rate(http_requests_total[5m])",
              "legendFormat": "{{service}}"
            }
          ]
        },
        {
          "title": "Error Rate",
          "type": "graph",
          "targets": [
            {
              "expr": "rate(http_requests_total{status=~'5..'}[5m]) / rate(http_requests_total[5m])"
            }
          ]
        },
        {
          "title": "Active Orders",
          "type": "stat",
          "targets": [
            {
              "expr": "orders_active_total"
            }
          ]
        }
      ]
    }
```

---

## Step 9: CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: company-registry.io
  IMAGE_PREFIX: ecommerce

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run tests
      run: |
        cd services/product-service
        npm test

  security-scan:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build image
      run: docker build -t test-image ./services/product-service
    
    - name: Scan with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: test-image
        format: table
        exit-code: 1
        severity: CRITICAL

  build-and-push:
    needs: security-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Login to registry
      run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login $REGISTRY -u ${{ secrets.REGISTRY_USER }} --password-stdin
    
    - name: Build and push
      run: |
        VERSION=${{ github.sha }}
        docker build -t $REGISTRY/$IMAGE_PREFIX/product-service:$VERSION \
          ./services/product-service
        docker push $REGISTRY/$IMAGE_PREFIX/product-service:$VERSION

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Deploy
      run: |
        VERSION=${{ github.sha }}
        kubectl set image deployment/product-service \
          product-service=$REGISTRY/$IMAGE_PREFIX/product-service:$VERSION \
          -n backend
        kubectl rollout status deployment/product-service -n backend --timeout=5m
```

---

## Step 10: Load Testing

```javascript
// k6 Load Test Script
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Histogram } from 'k6/metrics';

const errorCounter = new Counter('errors');
const orderDuration = new Histogram('order_duration');

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Warm up
    { duration: '5m', target: 500 },   // Load test
    { duration: '2m', target: 1000 },  // Stress test
    { duration: '5m', target: 1000 },  // Sustained load
    { duration: '2m', target: 0 },     // Cool down
  ],
  thresholds: {
    http_req_duration: ['p(99)<2000'],  // 99% < 2s
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
    errors: ['count<100'],
  },
};

const BASE_URL = 'https://api.myshop.example.com';

export function setup() {
  // สร้าง Test User
  const res = http.post(`${BASE_URL}/v1/users/register`, JSON.stringify({
    email: `test-${Date.now()}@test.com`,
    password: 'password123',
    name: 'Test User'
  }), { headers: { 'Content-Type': 'application/json' } });
  
  const token = res.json('token');
  return { token };
}

export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`
  };

  // Browse Products
  const productsRes = http.get(`${BASE_URL}/v1/products`, { headers });
  check(productsRes, {
    'products status 200': (r) => r.status === 200,
    'products response time < 500ms': (r) => r.timings.duration < 500,
  }) || errorCounter.add(1);

  const products = productsRes.json('products');
  if (!products || products.length === 0) return;

  // View Product Detail
  const product = products[Math.floor(Math.random() * products.length)];
  const productRes = http.get(`${BASE_URL}/v1/products/${product.id}`, { headers });
  check(productRes, {
    'product detail status 200': (r) => r.status === 200,
  }) || errorCounter.add(1);

  sleep(1);

  // Create Order
  const startTime = Date.now();
  const orderRes = http.post(`${BASE_URL}/v1/orders`, JSON.stringify({
    items: [{ product_id: product.id, quantity: 1 }],
    shipping_address: {
      street: '123 Main St',
      city: 'Bangkok',
      country: 'TH',
      postal_code: '10110'
    }
  }), { headers });

  orderDuration.add(Date.now() - startTime);
  check(orderRes, {
    'order created': (r) => r.status === 201,
    'order has id': (r) => r.json('id') !== undefined,
  }) || errorCounter.add(1);

  sleep(2);
}
```

---

## Workshop

### การ Deploy E-Commerce Platform แบบ Step-by-Step

```bash
#!/bin/bash
set -e

echo "=== Deploying E-Commerce Platform ==="

# 1. สร้าง Namespaces
echo "Creating namespaces..."
kubectl apply -f namespaces/

# 2. ติดตั้ง Infrastructure
echo "Installing infrastructure..."
helm upgrade --install product-db bitnami/postgresql \
  -n infrastructure \
  -f infrastructure/postgresql/product-db-values.yaml \
  --wait

helm upgrade --install redis bitnami/redis \
  -n infrastructure \
  -f infrastructure/redis/redis-values.yaml \
  --wait

helm upgrade --install rabbitmq bitnami/rabbitmq \
  -n infrastructure \
  -f infrastructure/rabbitmq/values.yaml \
  --wait

# 3. สร้าง Secrets
echo "Creating secrets..."
./scripts/create-secrets.sh

# 4. Deploy Services
echo "Deploying backend services..."
kubectl apply -f services/

# 5. Deploy Frontend
echo "Deploying frontend..."
kubectl apply -f frontend/

# 6. สร้าง Ingress
echo "Creating ingress..."
kubectl apply -f api-gateway/

# 7. Apply Network Policies
echo "Applying network policies..."
kubectl apply -f security/

# 8. ตั้งค่า Monitoring
echo "Setting up monitoring..."
kubectl apply -f monitoring/

echo "=== Deployment Complete ==="

# ตรวจสอบ Status
kubectl get pods -n backend
kubectl get pods -n frontend
kubectl get pods -n infrastructure
```

### Smoke Test

```bash
#!/bin/bash
echo "Running Smoke Tests..."

API_URL="https://api.myshop.example.com"
SHOP_URL="https://myshop.example.com"

# Health Checks
for svc in product-service order-service user-service; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" $API_URL/v1/$svc/health)
  if [ "$STATUS" != "200" ]; then
    echo "FAIL: $svc health check returned $STATUS"
    exit 1
  fi
  echo "PASS: $svc health check"
done

# API Tests
PRODUCTS=$(curl -s "$API_URL/v1/products" | jq '.total')
echo "Products in catalog: $PRODUCTS"

# Storefront
STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SHOP_URL)
if [ "$STATUS" != "200" ]; then
  echo "FAIL: Storefront returned $STATUS"
  exit 1
fi
echo "PASS: Storefront is up"

echo "All smoke tests passed!"
```

---

## สรุป

โปรเจค E-Commerce Platform นี้ครอบคลุม:

1. **Microservices Architecture** - แยก Services ตาม Domain
2. **High Availability** - Replicas, PDB, Affinity
3. **Security** - Network Policies, SecurityContext, Secrets
4. **Observability** - Prometheus, Grafana, Alerts
5. **CI/CD** - GitHub Actions, Image Scanning
6. **Performance** - HPA, Caching, Load Testing

## References

- [Kubernetes Patterns](https://k8spatterns.io/)
- [Cloud Native Applications](https://www.cncf.io/blog/)
- [Microservices with Kubernetes](https://microservices.io/)
