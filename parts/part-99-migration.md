# Part 99: Migration Strategy สำหรับ Kubernetes

## บทนำ

การย้าย Application จาก Traditional Infrastructure ไปยัง Kubernetes เป็นกระบวนการที่ต้องการการวางแผนอย่างละเอียด บทนี้ครอบคลุมกลยุทธ์ต่างๆ ตั้งแต่ Lift-and-Shift ไปจนถึงการออกแบบ Cloud-native Architecture ใหม่

## สารบัญ

1. Migration Strategies
2. Assessment Framework
3. Containerization Best Practices
4. Database Migration
5. Service Discovery Migration
6. CI/CD Pipeline Migration
7. Testing Strategy
8. Workshop: Migrate Legacy App

---

## 1. Migration Strategies

### 1.1 The 6 R's of Migration

```
1. Rehost (Lift and Shift)
   - ย้ายโดยไม่เปลี่ยน Code
   - ใช้เวลาน้อยที่สุด
   - ประโยชน์จาก Kubernetes น้อยที่สุด
   - เหมาะสำหรับ: Legacy apps, time-constrained migrations

2. Replatform (Lift, Tinker, and Shift)
   - เปลี่ยนบางส่วนเพื่อใช้ประโยชน์จาก Cloud
   - เช่น: เปลี่ยน Database เป็น Managed Service
   - Balance ระหว่าง effort และ benefit

3. Repurchase (Drop and Shop)
   - เปลี่ยน Application เป็น SaaS/Commercial product
   - เช่น: เปลี่ยน Email Server เป็น Gmail

4. Refactor (Re-architect)
   - ออกแบบใหม่เป็น Microservices
   - ใช้เวลามากที่สุด
   - ได้ประโยชน์สูงสุด
   - เหมาะสำหรับ: Core business applications

5. Retire
   - ยกเลิก Application ที่ไม่จำเป็น
   - ลดความซับซ้อน

6. Retain
   - คงไว้ที่เดิม
   - เหมาะสำหรับ: Legacy systems ที่ยังต้องการ Compliance หรือ ยังไม่พร้อม migrate
```

### 1.2 Migration Phases

```
Phase 1: Discovery and Assessment (2-4 weeks)
- Inventory Application Portfolio
- Identify Dependencies
- Assess Containerization Readiness
- Define Migration Priority

Phase 2: Pilot Migration (4-8 weeks)
- เลือก 1-2 Non-critical Applications
- Containerize และ Deploy บน Kubernetes
- Learn and Adjust Process
- Document Lessons Learned

Phase 3: Wave 1 - Low Risk Apps (8-12 weeks)
- Migrate Stateless Applications
- Setup CI/CD Pipeline
- Establish Monitoring

Phase 4: Wave 2 - Medium Risk Apps (12-16 weeks)
- Migrate Stateful Applications
- Database Migration
- Advanced Networking

Phase 5: Wave 3 - High Risk / Core Apps (16+ weeks)
- Migrate Business-Critical Applications
- Complete DR Setup
- Performance Optimization

Phase 6: Cutover and Optimization
- Final Cutover
- Decommission Legacy
- Continuous Optimization
```

---

## 2. Assessment Framework

### 2.1 Application Assessment Script

```bash
#!/bin/bash
# assess-application.sh <app-name>

APP="${1:-myapp}"
echo "=== Application Assessment: $APP ==="
echo ""

# Manual Checklist
cat << 'EOF'
Please answer the following questions:

ARCHITECTURE:
[ ] Is the application stateless?
[ ] Does it have external state (DB, Cache, Queue)?
[ ] Does it support horizontal scaling?
[ ] Are all configurations externalized?
[ ] Does it use environment variables for config?

DEPENDENCIES:
[ ] What OS does it require?
[ ] What runtime/language/version?
[ ] What libraries/packages?
[ ] External services it connects to?
[ ] Port bindings?

OPERABILITY:
[ ] Does it have health check endpoints?
[ ] Does it have structured logging?
[ ] Does it emit metrics?
[ ] Does it handle SIGTERM gracefully?
[ ] What's the startup time?

SECURITY:
[ ] Does it require root privileges?
[ ] Does it use hardcoded credentials?
[ ] Does it require specific ports < 1024?
[ ] What secrets does it need?

SCORING:
- Each YES in ARCHITECTURE: +2 points
- Each YES in OPERABILITY: +1 point
- Each YES in SECURITY risks: -2 points

> 10 points: Easy to containerize
5-10 points: Medium effort required
< 5 points: Significant refactoring needed
EOF
```

### 2.2 12-Factor App Compliance Check

```bash
#!/bin/bash
# twelve-factor-check.sh

echo "=== 12-Factor App Compliance Checklist ==="

cat << 'EOF'
1. Codebase
   [ ] One codebase tracked in version control
   [ ] Multiple deploys from same codebase
   Score: /2

2. Dependencies
   [ ] All dependencies explicitly declared
   [ ] No system-wide packages required
   Score: /2

3. Config
   [ ] Stored in environment variables
   [ ] No hardcoded config in code
   [ ] Different config per environment
   Score: /3

4. Backing Services
   [ ] Database/Cache/Queue treated as attached resources
   [ ] Can swap backends without code change
   Score: /2

5. Build, Release, Run
   [ ] Separate build stage
   [ ] Release = build + config
   [ ] Run stage is simple process
   Score: /3

6. Processes
   [ ] App is stateless
   [ ] No sticky sessions
   [ ] Session data in external store
   Score: /3

7. Port Binding
   [ ] Self-contained HTTP service
   [ ] Exports services via port
   Score: /2

8. Concurrency
   [ ] Scale via process model
   [ ] Can run multiple instances
   Score: /2

9. Disposability
   [ ] Fast startup (< 30 seconds)
   [ ] Graceful shutdown
   [ ] Robust against sudden death
   Score: /3

10. Dev/Prod Parity
    [ ] Small gap between dev and prod
    [ ] Continuous deployment
    Score: /2

11. Logs
    [ ] Treat logs as event streams
    [ ] Write to stdout/stderr
    Score: /2

12. Admin Processes
    [ ] Run admin tasks as one-off processes
    [ ] Same code as long-running processes
    Score: /2

TOTAL: /28
EOF
```

---

## 3. Containerization Best Practices

### 3.1 Dockerfile Best Practices

```dockerfile
# EXAMPLE 1: Node.js Application

# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files first (better layer caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source
COPY . .

# Build if needed
RUN npm run build

# Stage 2: Production Image
FROM node:20-alpine AS production

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built artifacts from builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Use non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Start command
CMD ["node", "dist/index.js"]
```

```dockerfile
# EXAMPLE 2: Java Spring Boot Application

# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# Copy Maven wrapper and pom.xml
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies
RUN ./mvnw dependency:go-offline -B

# Copy source
COPY src src

# Build
RUN ./mvnw package -DskipTests

# Stage 2: Production
FROM eclipse-temurin:17-jre-alpine

RUN addgroup -g 1001 spring && \
    adduser -u 1001 -G spring -s /bin/sh -D spring

WORKDIR /app

COPY --from=builder --chown=spring:spring /app/target/*.jar app.jar

USER spring

EXPOSE 8080

ENV JAVA_OPTS="-XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -Djava.security.egd=file:/dev/./urandom"

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

```dockerfile
# EXAMPLE 3: Python Flask Application

FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Production
FROM python:3.11-slim

# Create non-root user
RUN groupadd -g 1001 python && \
    useradd -u 1001 -g python -s /bin/sh -m python

WORKDIR /app

COPY --from=builder --chown=python:python /root/.local /home/python/.local
COPY --chown=python:python . .

USER python

ENV PATH=/home/python/.local/bin:$PATH
ENV FLASK_APP=app.py
ENV PYTHONUNBUFFERED=1

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:5000/health || exit 1

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:app"]
```

### 3.2 Container Image Optimization

```bash
#!/bin/bash
# optimize-image.sh <image-name>

IMAGE=$1

echo "=== Analyzing Image: $IMAGE ==="

# ดู Image Size
docker image inspect $IMAGE --format='Size: {{.Size}} bytes'

# ดู Layers
docker history $IMAGE

# ใช้ dive สำหรับ Analysis
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest $IMAGE

# Scan Vulnerabilities ด้วย Trivy
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image $IMAGE

echo ""
echo "Optimization Tips:"
echo "1. Use multi-stage builds"
echo "2. Use slim/alpine base images"
echo "3. Minimize layers (.RUN, COPY, ADD)"
echo "4. Use .dockerignore"
echo "5. Remove build tools from production"
echo "6. Use non-root user"
```

---

## 4. Database Migration

### 4.1 Database Migration Strategies

```
Strategy 1: Lift and Shift Database
- เอา Database ขึ้น Kubernetes เลย
- ใช้ StatefulSet + PVC
- ง่ายแต่ไม่แนะนำสำหรับ Production

Strategy 2: Keep Database External
- Database อยู่ที่เดิม (RDS, GCP CloudSQL)
- Application ย้ายเข้า Kubernetes
- ปลอดภัยและง่ายกว่า

Strategy 3: Managed Database Service
- ย้าย Database ไป Managed Service
- เช่น: AWS RDS, Azure PostgreSQL, GCP CloudSQL
- แนะนำสำหรับ Production

Strategy 4: Database on Kubernetes
- ใช้ Database Operator (Postgres Operator, Percona Operator)
- เหมาะสำหรับ Development/Test
- Production ใช้ได้แต่ต้องมีความเชี่ยวชาญ
```

### 4.2 Database Migration Script

```bash
#!/bin/bash
# migrate-database.sh

SOURCE_DB_HOST="old-db.example.com"
SOURCE_DB_PORT="5432"
SOURCE_DB_NAME="myapp"
SOURCE_DB_USER="dbuser"

TARGET_DB_HOST="myapp-db.postgres.database.azure.com"
TARGET_DB_NAME="myapp"
TARGET_DB_USER="dbadmin"

echo "=== Database Migration: $SOURCE_DB_HOST -> $TARGET_DB_HOST ==="

# 1. ตรวจสอบ Source Database
echo "Checking source database..."
psql -h $SOURCE_DB_HOST -U $SOURCE_DB_USER -d $SOURCE_DB_NAME -c "\dt" | wc -l

# 2. ทำ Schema Migration ก่อน
echo "Migrating schema..."
pg_dump \
  -h $SOURCE_DB_HOST \
  -U $SOURCE_DB_USER \
  -d $SOURCE_DB_NAME \
  --schema-only \
  -f schema.sql

psql \
  -h $TARGET_DB_HOST \
  -U $TARGET_DB_USER \
  -d $TARGET_DB_NAME \
  -f schema.sql

# 3. ทำ Data Migration
echo "Migrating data..."
pg_dump \
  -h $SOURCE_DB_HOST \
  -U $SOURCE_DB_USER \
  -d $SOURCE_DB_NAME \
  --data-only \
  -F c \
  -f data.dump

pg_restore \
  -h $TARGET_DB_HOST \
  -U $TARGET_DB_USER \
  -d $TARGET_DB_NAME \
  --no-acl \
  --no-owner \
  data.dump

# 4. ตรวจสอบผล
echo "Verifying migration..."
SOURCE_COUNT=$(psql -h $SOURCE_DB_HOST -U $SOURCE_DB_USER -d $SOURCE_DB_NAME -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public'")
TARGET_COUNT=$(psql -h $TARGET_DB_HOST -U $TARGET_DB_USER -d $TARGET_DB_NAME -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public'")

echo "Source tables: $SOURCE_COUNT"
echo "Target tables: $TARGET_COUNT"

# 5. Validate Row Counts
echo "Validating row counts..."
psql -h $SOURCE_DB_HOST -U $SOURCE_DB_USER -d $SOURCE_DB_NAME -t << 'SQL' > /tmp/source_counts.txt
SELECT tablename, n_live_tup as count
FROM pg_stat_user_tables
ORDER BY tablename;
SQL

# Compare with target...
echo "Migration complete!"
```

### 4.3 Zero-Downtime Database Migration

```bash
#!/bin/bash
# zero-downtime-db-migration.sh

# Blue-Green Database Strategy

echo "=== Zero-Downtime Database Migration ==="

# Step 1: Setup Replication from Old to New DB
echo "1. Setting up replication..."
# (ใช้ pg_logical หรือ AWS DMS สำหรับ Continuous Replication)

# Step 2: ทำ Initial Sync
echo "2. Initial data sync..."
# pg_dump + pg_restore

# Step 3: ทำ Dual-write
echo "3. Implementing dual-write..."
# Application เขียน DB ทั้งสอง

# Step 4: ตรวจสอบความสม่ำเสมอ
echo "4. Verifying data consistency..."
# Compare row counts และ checksums

# Step 5: Switch Read Traffic
echo "5. Switching read traffic to new DB..."
# อัพเดท Connection String ใน ConfigMap

# Step 6: Switch Write Traffic
echo "6. Switching write traffic to new DB..."
# อัพเดท Write Connection String

# Step 7: ตรวจสอบ
echo "7. Final verification..."
# Run integration tests

# Step 8: Decommission Old DB
echo "8. Decommissioning old DB after 1 week monitoring..."
```

---

## 5. Kubernetes Manifests สำหรับ Migration

### 5.1 Complete Legacy App Migration

```yaml
# legacy-app-k8s.yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: migrated-app
  labels:
    environment: production
    app: legacy-migrated

---
# ConfigMap สำหรับ App Config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: migrated-app
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  DB_HOST: "myapp-db.postgres.database.azure.com"
  DB_PORT: "5432"
  DB_NAME: "myapp"
  REDIS_HOST: "myapp-redis.cache.windows.net"
  REDIS_PORT: "6380"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"

---
# Secret สำหรับ Credentials
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: migrated-app
type: Opaque
stringData:
  DB_PASSWORD: "your-db-password"
  REDIS_PASSWORD: "your-redis-password"
  JWT_SECRET: "your-jwt-secret"
  API_KEY: "your-api-key"

---
# Main Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: migrated-app
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      terminationGracePeriodSeconds: 60
      
      containers:
      - name: app
        image: myregistry.azurecr.io/myapp:v1.2.3
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        
        envFrom:
        - configMapRef:
            name: app-config
        
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: REDIS_PASSWORD
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: JWT_SECRET
        
        # Downward API
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
        
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 2
          successThreshold: 2
        
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /var/log/app
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["myapp"]
              topologyKey: kubernetes.io/hostname
      
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: myapp

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: app-svc
  namespace: migrated-app
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080

---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: migrated-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 3
  maxReplicas: 20
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

---
# PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
  namespace: migrated-app
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: migrated-app
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-svc
            port:
              number: 80
```

---

## 6. Workshop: Migrate Legacy App

### Workshop Overview

ในบทนี้เราจะ Migrate Legacy Python Flask Application ไปยัง Kubernetes:
1. Assessment
2. Containerize
3. Create Kubernetes Manifests
4. Deploy และ Test
5. Configure Autoscaling

### Step 1: Legacy Application

```python
# legacy-app/app.py - Legacy Flask Application

from flask import Flask, request, jsonify
import os
import logging
import psycopg2
from datetime import datetime

app = Flask(__name__)

# ปัญหา: Config Hardcoded
DB_HOST = os.environ.get('DB_HOST', 'localhost')
DB_NAME = os.environ.get('DB_NAME', 'myapp')
DB_USER = os.environ.get('DB_USER', 'postgres')
DB_PASSWORD = os.environ.get('DB_PASSWORD', 'password')

def get_db_connection():
    return psycopg2.connect(
        host=DB_HOST,
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD
    )

@app.route('/')
def index():
    return jsonify({'message': 'Hello World', 'timestamp': str(datetime.now())})

@app.route('/health/live')
def liveness():
    return jsonify({'status': 'alive'})

@app.route('/health/ready')
def readiness():
    try:
        conn = get_db_connection()
        conn.close()
        return jsonify({'status': 'ready'})
    except:
        return jsonify({'status': 'not ready'}), 503

@app.route('/users', methods=['GET'])
def get_users():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT id, name, email FROM users LIMIT 100")
        users = [{'id': r[0], 'name': r[1], 'email': r[2]} for r in cur.fetchall()]
        cur.close()
        conn.close()
        return jsonify(users)
    except Exception as e:
        logging.error(f"Error: {e}")
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    port = int(os.environ.get('APP_PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

```text
# requirements.txt
flask==3.0.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
prometheus-flask-exporter==0.23.0
```

### Step 2: Containerize

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Create non-root user
RUN groupadd -g 1001 appuser && \
    useradd -u 1001 -g appuser -s /bin/sh -m appuser

WORKDIR /app

# Copy requirements first
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appuser . .

USER appuser

ENV PYTHONUNBUFFERED=1
ENV FLASK_APP=app.py

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:5000/health/live || exit 1

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", \
     "--timeout", "30", "--keep-alive", "2", \
     "--log-level", "info", "--access-logfile", "-", \
     "app:app"]
```

```
# .dockerignore
__pycache__
*.pyc
*.pyo
.git
.env
.pytest_cache
tests/
docs/
*.md
```

### Step 3: Build และ Push Image

```bash
#!/bin/bash
# build-and-push.sh

APP_NAME="myapp"
REGISTRY="myregistry.azurecr.io"
VERSION=$(git rev-parse --short HEAD)
TAG="${REGISTRY}/${APP_NAME}:${VERSION}"
LATEST="${REGISTRY}/${APP_NAME}:latest"

echo "Building image: $TAG"

# Build
docker build -t $TAG -t $LATEST .

# Scan Vulnerabilities
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image $TAG

# Push
docker push $TAG
docker push $LATEST

echo "Image built and pushed: $TAG"
echo "VERSION=${VERSION}" > .image-version
```

### Step 4: Create Kubernetes Manifests

```bash
# สร้างไฟล์ Manifests ทั้งหมด
mkdir -p k8s/overlays/{dev,staging,production}

# Base Configuration
cat > k8s/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: app
        image: myregistry.azurecr.io/myapp:latest
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
EOF

cat > k8s/base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: legacy-app-svc
spec:
  selector:
    app: legacy-app
  ports:
  - port: 80
    targetPort: 5000
EOF

cat > k8s/base/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "db.example.com"
  DB_NAME: "myapp"
  DB_USER: "appuser"
  APP_PORT: "5000"
  LOG_LEVEL: "info"
EOF

cat > k8s/base/kustomization.yaml << 'EOF'
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
EOF
```

### Step 5: Deploy และ Test

```bash
#!/bin/bash
# deploy-and-test.sh

NAMESPACE="migrated-app"
ENV="${1:-staging}"

echo "=== Deploying to $ENV ==="

# สร้าง Namespace ถ้าไม่มี
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Deploy
kubectl apply -k k8s/overlays/$ENV -n $NAMESPACE

# รอ Deployment
kubectl wait --for=condition=available deployment/legacy-app \
  -n $NAMESPACE \
  --timeout=300s

echo "Deployment successful!"
echo ""

# Run Tests
echo "Running smoke tests..."

# ดู Pod Status
kubectl get pods -n $NAMESPACE -l app=legacy-app

# Get Service URL
if kubectl get ingress -n $NAMESPACE legacy-app-ingress &>/dev/null; then
  APP_URL="https://$(kubectl get ingress -n $NAMESPACE legacy-app-ingress \
    -o jsonpath='{.spec.rules[0].host}')"
else
  # Port Forward สำหรับทดสอบ
  kubectl port-forward -n $NAMESPACE svc/legacy-app-svc 8080:80 &
  PF_PID=$!
  APP_URL="http://localhost:8080"
  sleep 3
fi

# Test Endpoints
echo ""
echo "Testing /"
curl -f $APP_URL/ && echo "" || echo "FAILED"

echo ""
echo "Testing /health/live"
curl -f $APP_URL/health/live && echo "" || echo "FAILED"

echo ""
echo "Testing /health/ready"
curl -f $APP_URL/health/ready && echo "" || echo "FAILED"

echo ""
echo "Testing /users"
curl -f $APP_URL/users | head -100

# Cleanup port-forward
[ -n "$PF_PID" ] && kill $PF_PID 2>/dev/null

echo ""
echo "=== Deployment complete! ==="
```

### Step 6: Cutover Strategy

```bash
#!/bin/bash
# cutover.sh - Blue-Green Cutover

OLD_SERVICE="legacy-app-old"
NEW_SERVICE="legacy-app-new"
LB_RULE="myapp-lb-rule"

echo "=== Migration Cutover ==="
echo ""
echo "Pre-cutover checks:"

# 1. ตรวจสอบ New Service
READY_PODS=$(kubectl get deployment legacy-app -n migrated-app \
  -o jsonpath='{.status.readyReplicas}')
echo "New service ready pods: $READY_PODS"

# 2. ตรวจสอบ Error Rate
echo "Checking error rate (last 5 minutes)..."
# (ตรวจสอบจาก Prometheus/Grafana)

# 3. ตรวจสอบ Performance
echo "Checking response time..."

read -p "Ready to proceed with cutover? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
  echo "Cutover cancelled"
  exit 0
fi

echo ""
echo "Executing cutover..."

# Switch DNS/Load Balancer
# (Depends on your setup - Route53, Azure DNS, etc.)
echo "Update DNS/LB to point to new service..."

# Monitor
echo "Monitoring for 5 minutes..."
for i in {1..5}; do
  echo "Minute $i:"
  kubectl top pods -n migrated-app
  sleep 60
done

echo ""
echo "Cutover complete!"
echo "Monitor: kubectl get pods -n migrated-app -w"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Migration Strategies** - 6 R's ของ Migration
2. **Assessment** - วิเคราะห์ Application ก่อน Migrate
3. **Containerization** - Best practices สำหรับ Dockerfile
4. **Database Migration** - Strategies และ Zero-downtime approach
5. **Kubernetes Manifests** - Complete manifests สำหรับ Production
6. **Testing** - Smoke tests และ Validation
7. **Workshop** - End-to-end migration walkthrough

## แบบฝึกหัด

1. Assessment Legacy Application ของคุณด้วย 12-Factor Framework
2. Containerize Application โดยใช้ Multi-stage Build
3. สร้าง Kubernetes Manifests ที่ครบถ้วนสำหรับ Application
4. Deploy บน Kubernetes และ Run Smoke Tests
5. วางแผน Zero-downtime Cutover Strategy

## References

- [12-Factor App](https://12factor.net/)
- [Google Container Best Practices](https://cloud.google.com/solutions/best-practices-for-building-containers)
- [Kubernetes Migration Guide](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
