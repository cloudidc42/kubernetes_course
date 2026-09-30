# Part 28: Init Containers - เตรียมความพร้อมก่อน Main Container

## สารบัญ
1. [Init Containers คืออะไร](#init-containers-คืออะไร)
2. [Use Cases](#use-cases)
3. [Init Container YAML ละเอียด](#init-container-yaml-ละเอียด)
4. [Sequencing และ Dependencies](#sequencing-และ-dependencies)
5. [Shared Volumes ระหว่าง Init และ Main](#shared-volumes-ระหว่าง-init-และ-main)
6. [Workshop: Database Migration](#workshop-database-migration)
7. [Workshop: Config File Generation](#workshop-config-file-generation)
8. [Workshop: Wait for Dependencies](#workshop-wait-for-dependencies)
9. [Debugging Init Containers](#debugging-init-containers)
10. [Best Practices](#best-practices)

---

## Init Containers คืออะไร

**Init Containers** คือ containers พิเศษที่รัน**ก่อน** main containers ใน Pod:

- รันตามลำดับ (sequential) - init container แต่ละตัวต้องเสร็จ (exit 0) ก่อนตัวถัดไปเริ่ม
- Main containers จะ**ไม่เริ่ม**จนกว่า init containers ทุกตัวสำเร็จ
- ถ้า init container ล้มเหลว → Pod จะ restart จนกว่าจะสำเร็จ (ตาม restartPolicy)

### Init Container vs Main Container

```
Pod Lifecycle:
─────────────────────────────────────────────────────────────────
  
  INIT PHASE                      │ MAIN PHASE
  ─────────────────────────────── │ ──────────────────────────────
  init-container-1                │
    → รันจนเสร็จ (exit 0)         │
    ↓                             │
  init-container-2                │
    → รันจนเสร็จ (exit 0)         │
    ↓                             │
  init-container-3                │
    → รันจนเสร็จ (exit 0)         │
                                  │ main-container-1 ────────────→
                                  │ main-container-2 ────────────→
                                  │ main-container-3 ────────────→
  
  Init containers รัน sequential  │ Main containers รัน parallel

ถ้า init-2 ล้มเหลว:
  init-1 → init-2 FAIL → restart → init-1 → init-2 (ลอง) → ...
```

### ความแตกต่างจาก Main Containers

| คุณสมบัติ | Init Container | Main Container |
|-----------|---------------|----------------|
| รันพร้อมกัน | ไม่ (sequential) | ใช่ (parallel) |
| Restart เมื่อล้มเหลว | ใช่ | ขึ้นกับ restartPolicy |
| Probes (liveness/readiness) | ไม่รองรับ | รองรับ |
| Resource ที่ใช้คำนวณ QoS | ค่าสูงสุดของ init container | ผลรวมของ main containers |
| เข้าถึง volumes ได้ | ใช่ | ใช่ |

---

## Use Cases

### 1. รอ Dependencies

```
Application ต้องการ Database แต่ DB อาจยังไม่พร้อม:
  Init Container: รอจนกว่า postgres:5432 เปิดอยู่
  Main Container: เริ่ม application
```

### 2. Database Migration

```
เมื่อ deploy version ใหม่:
  Init Container: รัน migration scripts (ALTER TABLE, etc.)
  Main Container: เริ่ม application ที่ใช้ schema ใหม่
```

### 3. Configuration Generation

```
Config ต้องการ secrets หรือค่าจาก external sources:
  Init Container: ดึง config จาก Vault, generate SSL cert
  Main Container: ใช้ config ที่ generate แล้ว
```

### 4. Permission Setup

```
Application ต้องการ directories พิเศษ:
  Init Container: สร้าง directories, set permissions
  Main Container: เขียน/อ่านไฟล์จาก directories นั้น
```

### 5. Clone Git Repository

```
สำหรับ GitOps หรือ config loading:
  Init Container: git clone repo → /shared-volume
  Main Container: ใช้ files จาก /shared-volume
```

---

## Init Container YAML ละเอียด

### Basic Init Container

```yaml
# basic-init-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  namespace: production
spec:
  # Init Containers (รันก่อน main containers)
  initContainers:
  - name: wait-for-service
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Waiting for database to be ready..."
      until nc -zv postgres.production.svc.cluster.local 5432; do
        echo "Database not ready, waiting..."
        sleep 2
      done
      echo "Database is ready!"
    
    # Resources สำหรับ init container
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
  
  # Main Containers (เริ่มหลัง init containers เสร็จ)
  containers:
  - name: app
    image: my-app:latest
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
```

### Init Container ที่ซับซ้อน

```yaml
# complex-init-containers.yaml
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
      # Volumes ที่ใช้ร่วมกัน
      volumes:
      - name: shared-config
        emptyDir: {}
      - name: shared-data
        emptyDir: {}
      - name: migration-scripts
        configMap:
          name: migration-scripts
      
      initContainers:
      # Init 1: รอ Database พร้อม
      - name: wait-db
        image: postgres:14
        command:
        - sh
        - -c
        - |
          echo "=== Waiting for database ==="
          until pg_isready -h $DB_HOST -U $DB_USER -p 5432; do
            echo "Database is not ready - sleeping"
            sleep 2
          done
          echo "=== Database is ready ==="
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db-host
        - name: DB_USER
          value: "postgres"
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
      
      # Init 2: รัน Database Migration
      - name: run-migration
        image: my-app-migration:latest
        command:
        - sh
        - -c
        - |
          echo "=== Running database migration ==="
          python manage.py migrate --no-input
          echo "=== Migration completed ==="
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        volumeMounts:
        - name: migration-scripts
          mountPath: /migrations
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
      
      # Init 3: Generate Configuration
      - name: generate-config
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import os
          import json
          
          # สร้าง config จาก environment variables
          config = {
            "database": {
              "host": os.environ.get("DB_HOST"),
              "port": int(os.environ.get("DB_PORT", "5432")),
              "name": os.environ.get("DB_NAME")
            },
            "redis": {
              "host": os.environ.get("REDIS_HOST"),
              "port": int(os.environ.get("REDIS_PORT", "6379"))
            },
            "features": {
              "debug": os.environ.get("DEBUG", "false").lower() == "true",
              "new_ui": os.environ.get("FEATURE_NEW_UI", "false").lower() == "true"
            }
          }
          
          # เขียน config ลง shared volume
          with open("/shared-config/app-config.json", "w") as f:
            json.dump(config, f, indent=2)
          
          print("Configuration generated successfully")
          print(f"Config: {json.dumps(config, indent=2)}")
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db-host
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db-name
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: redis-host
        - name: DEBUG
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: debug
        volumeMounts:
        - name: shared-config
          mountPath: /shared-config
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
      
      # Main Application Container
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080
        
        env:
        - name: CONFIG_FILE
          value: "/config/app-config.json"
        
        volumeMounts:
        - name: shared-config
          mountPath: /config
          readOnly: true
        
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 15
```

---

## Sequencing และ Dependencies

### หลาย Init Containers (Sequential)

```yaml
initContainers:
# ลำดับ 1: ตรวจสอบ network connectivity
- name: check-network
  image: busybox:1.35
  command: ['sh', '-c', 'ping -c 3 8.8.8.8 || exit 1']

# ลำดับ 2: รอ database (ทำหลังจาก check-network สำเร็จ)
- name: wait-database
  image: postgres:14
  command: ['sh', '-c', 'until pg_isready -h postgres -U postgres; do sleep 1; done']

# ลำดับ 3: รอ redis (ทำหลังจาก wait-database สำเร็จ)
- name: wait-redis
  image: redis:7.0
  command: ['sh', '-c', 'until redis-cli -h redis ping; do sleep 1; done']

# ลำดับ 4: รัน migration (ทำหลัง wait-redis สำเร็จ)
- name: migrate
  image: my-app:latest
  command: ['sh', '-c', './run_migrations.sh']
```

### Timeout สำหรับ Init Container

```yaml
initContainers:
- name: wait-with-timeout
  image: busybox:1.35
  command:
  - sh
  - -c
  - |
    # timeout หลัง 5 นาที ถ้า service ไม่พร้อม
    TIMEOUT=300
    ELAPSED=0
    
    while ! nc -z postgres 5432; do
      if [ $ELAPSED -ge $TIMEOUT ]; then
        echo "Timeout waiting for postgres after ${TIMEOUT}s"
        exit 1
      fi
      echo "Waiting for postgres... (${ELAPSED}s elapsed)"
      sleep 5
      ELAPSED=$((ELAPSED + 5))
    done
    
    echo "Postgres is ready!"
```

---

## Shared Volumes ระหว่าง Init และ Main

```yaml
spec:
  volumes:
  # emptyDir: temporary storage ที่ใช้ร่วมกันใน Pod
  - name: work-dir
    emptyDir: {}
  
  # configMap: share config files
  - name: app-scripts
    configMap:
      name: app-scripts
      defaultMode: 0755  # executable
  
  initContainers:
  - name: setup
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      # สร้างไฟล์ใน shared volume
      echo "Hello from init container" > /work/message.txt
      echo "Init complete at $(date)" >> /work/message.txt
      chmod 644 /work/message.txt
    volumeMounts:
    - name: work-dir
      mountPath: /work
  
  containers:
  - name: main
    image: nginx:1.21
    volumeMounts:
    - name: work-dir
      mountPath: /usr/share/nginx/html
      # Main container อ่านไฟล์ที่ init container สร้าง
```

---

## Workshop: Database Migration

เป้าหมาย: Deploy Django application พร้อม automatic database migration

### 1. สร้าง ConfigMap สำหรับ Migration Scripts

```yaml
# migration-scripts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: migration-scripts
  namespace: production
data:
  run_migrations.sh: |
    #!/bin/bash
    set -e
    
    echo "================================"
    echo "Database Migration Tool v1.0"
    echo "================================"
    echo "Target: ${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}"
    echo ""
    
    # ตรวจสอบ database connection
    echo "Step 1: Checking database connection..."
    python -c "
    import psycopg2
    import os
    import time
    
    max_retries = 30
    for i in range(max_retries):
        try:
            conn = psycopg2.connect(
                host=os.environ['DATABASE_HOST'],
                port=int(os.environ.get('DATABASE_PORT', '5432')),
                dbname=os.environ['DATABASE_NAME'],
                user=os.environ['DATABASE_USER'],
                password=os.environ['DATABASE_PASSWORD']
            )
            conn.close()
            print('Database connection successful!')
            break
        except Exception as e:
            print(f'Attempt {i+1}/{max_retries}: {e}')
            time.sleep(2)
    else:
        print('Failed to connect to database after all retries')
        exit(1)
    "
    
    echo ""
    echo "Step 2: Running migrations..."
    python manage.py migrate --no-input --verbosity 2
    
    echo ""
    echo "Step 3: Collecting static files..."
    python manage.py collectstatic --no-input
    
    echo ""
    echo "Step 4: Creating default superuser (if not exists)..."
    python manage.py shell -c "
    from django.contrib.auth.models import User
    import os
    
    username = os.environ.get('DJANGO_SUPERUSER_USERNAME', 'admin')
    email = os.environ.get('DJANGO_SUPERUSER_EMAIL', 'admin@example.com')
    password = os.environ.get('DJANGO_SUPERUSER_PASSWORD', 'changeme')
    
    if not User.objects.filter(username=username).exists():
        User.objects.create_superuser(username, email, password)
        print(f'Superuser {username} created')
    else:
        print(f'Superuser {username} already exists')
    "
    
    echo ""
    echo "================================"
    echo "Migration completed successfully!"
    echo "================================"
  
  check_db.sh: |
    #!/bin/bash
    # ตรวจสอบ database ว่าพร้อมหรือยัง
    
    DB_HOST=${DATABASE_HOST:-localhost}
    DB_PORT=${DATABASE_PORT:-5432}
    
    echo "Checking database at ${DB_HOST}:${DB_PORT}..."
    
    until pg_isready -h "$DB_HOST" -p "$DB_PORT" -U "$DATABASE_USER"; do
      echo "Database is not ready - waiting 2 seconds..."
      sleep 2
    done
    
    echo "Database is ready!"
```

### 2. สร้าง Secret

```bash
kubectl create secret generic app-secrets \
  --from-literal=database-password=supersecret \
  --from-literal=django-secret-key=your-secret-key-here \
  --from-literal=superuser-password=admin123 \
  -n production
```

### 3. สร้าง ConfigMap สำหรับ App Config

```yaml
# app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "myapp"
  DATABASE_USER: "postgres"
  REDIS_HOST: "redis.production.svc.cluster.local"
  DEBUG: "false"
```

### 4. สร้าง Deployment พร้อม Init Container

```yaml
# django-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
  namespace: production
  labels:
    app: django-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      volumes:
      - name: static-files
        emptyDir: {}
      - name: migration-scripts
        configMap:
          name: migration-scripts
          defaultMode: 0755
      
      initContainers:
      # Init 1: รอ Database
      - name: wait-for-database
        image: postgres:14
        command:
        - /bin/sh
        - /scripts/check_db.sh
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
        volumeMounts:
        - name: migration-scripts
          mountPath: /scripts
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
      
      # Init 2: รัน Migration
      - name: run-migration
        image: my-django-app:latest
        command:
        - /bin/bash
        - /scripts/run_migrations.sh
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
        - name: DJANGO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: django-secret-key
        - name: DJANGO_SUPERUSER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: superuser-password
        volumeMounts:
        - name: migration-scripts
          mountPath: /scripts
        - name: static-files
          mountPath: /static
        resources:
          requests:
            cpu: "300m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      
      # Main Container
      containers:
      - name: django
        image: my-django-app:latest
        command: ["gunicorn", "myapp.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
        ports:
        - containerPort: 8000
        
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
        - name: DJANGO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: django-secret-key
        
        volumeMounts:
        - name: static-files
          mountPath: /static
          readOnly: true
        
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        
        readinessProbe:
          httpGet:
            path: /health/
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        livenessProbe:
          httpGet:
            path: /health/
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 15
```

### 5. ทดสอบ Migration

```bash
# Apply
kubectl apply -f app-config.yaml
kubectl apply -f migration-scripts.yaml
kubectl apply -f django-app.yaml

# ดู init container progress
kubectl get pods -n production -l app=django-app

# ดู logs ของ init containers
kubectl logs -n production django-app-xxx -c wait-for-database
kubectl logs -n production django-app-xxx -c run-migration

# ดู migration output
kubectl logs -n production django-app-xxx -c run-migration | grep -E "Applying|Running|Migration"

# รอให้ pod พร้อม
kubectl rollout status deployment/django-app -n production
```

---

## Workshop: Config File Generation

### สร้าง Application ที่ Generate Config จาก Vault

```yaml
# vault-init-container.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      serviceAccountName: secure-app-sa
      
      volumes:
      - name: generated-config
        emptyDir:
          medium: Memory  # Store in memory, not disk
      
      initContainers:
      # ดึง secrets จาก Vault และสร้าง config file
      - name: vault-config-generator
        image: vault:latest
        command:
        - sh
        - -c
        - |
          set -e
          
          echo "Logging into Vault..."
          
          # ล็อกอิน Vault ด้วย Kubernetes auth
          VAULT_TOKEN=$(vault write -field=token \
            auth/kubernetes/login \
            role=secure-app \
            jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token))
          
          echo "Fetching secrets..."
          
          # ดึง database credentials
          DB_CREDS=$(vault read -format=json database/creds/secure-app-role)
          DB_USER=$(echo $DB_CREDS | jq -r '.data.username')
          DB_PASSWORD=$(echo $DB_CREDS | jq -r '.data.password')
          
          # ดึง API keys
          API_KEY=$(vault kv get -field=api_key secret/secure-app/api)
          
          # สร้าง config file
          cat > /generated-config/config.yaml << EOF
          database:
            host: ${DB_HOST}
            port: 5432
            username: ${DB_USER}
            password: ${DB_PASSWORD}
          
          api:
            key: ${API_KEY}
            endpoint: ${API_ENDPOINT}
          
          generated_at: $(date -u +%Y-%m-%dT%H:%M:%SZ)
          EOF
          
          chmod 400 /generated-config/config.yaml
          echo "Config generated successfully"
        
        env:
        - name: VAULT_ADDR
          value: "https://vault.production.svc.cluster.local:8200"
        - name: DB_HOST
          value: "postgres.production.svc.cluster.local"
        - name: API_ENDPOINT
          value: "https://api.external-service.com"
        
        volumeMounts:
        - name: generated-config
          mountPath: /generated-config
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
      
      containers:
      - name: app
        image: secure-app:latest
        command: ["./start.sh"]
        
        env:
        - name: CONFIG_FILE
          value: /config/config.yaml
        
        volumeMounts:
        - name: generated-config
          mountPath: /config
          readOnly: true
        
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

---

## Workshop: Wait for Dependencies

### Init Container ที่รอหลาย Services

```yaml
# wait-for-services.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-dependencies
  namespace: production
spec:
  initContainers:
  # รอทุก services พร้อม
  - name: wait-for-services
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      set -e
      
      wait_for_service() {
        local host=$1
        local port=$2
        local name=$3
        local timeout=${4:-300}
        local elapsed=0
        
        echo "Waiting for $name ($host:$port)..."
        
        while ! nc -zw1 "$host" "$port" 2>/dev/null; do
          elapsed=$((elapsed + 2))
          if [ $elapsed -ge $timeout ]; then
            echo "Timeout waiting for $name after ${elapsed}s"
            exit 1
          fi
          echo "  $name not ready (${elapsed}s elapsed)..."
          sleep 2
        done
        
        echo "$name is ready!"
      }
      
      # รอ services ตามลำดับ
      wait_for_service postgres 5432 "PostgreSQL"
      wait_for_service redis 6379 "Redis"
      wait_for_service elasticsearch 9200 "Elasticsearch"
      wait_for_service kafka 9092 "Kafka"
      
      echo ""
      echo "All dependencies are ready!"
    
    resources:
      requests:
        cpu: "25m"
        memory: "16Mi"
  
  containers:
  - name: app
    image: my-app:latest
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
```

### Init Container ที่ Clone Git Repository

```yaml
# git-clone-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-git-config
spec:
  volumes:
  - name: config-volume
    emptyDir: {}
  - name: git-credentials
    secret:
      secretName: git-ssh-key
  
  initContainers:
  - name: git-clone
    image: alpine/git:latest
    command:
    - sh
    - -c
    - |
      # Setup SSH
      mkdir -p /root/.ssh
      cp /ssh/id_rsa /root/.ssh/id_rsa
      chmod 600 /root/.ssh/id_rsa
      ssh-keyscan github.com >> /root/.ssh/known_hosts
      
      # Clone config repository
      git clone git@github.com:myorg/app-config.git /config
      
      # Checkout specific branch/tag
      cd /config
      git checkout v1.2.3
      
      echo "Config cloned successfully"
      ls -la /config
    
    volumeMounts:
    - name: config-volume
      mountPath: /config
    - name: git-credentials
      mountPath: /ssh
    
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
  
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
      readOnly: true
```

---

## Debugging Init Containers

### ดู Init Container Status

```bash
# ดู Pod phase
kubectl get pod my-pod -n production

# STATUS ที่เป็นไปได้:
# Init:0/3  → init container แรกกำลังรัน (0 จาก 3 เสร็จ)
# Init:1/3  → init container แรกเสร็จ ตัวที่ 2 กำลังรัน
# Init:2/3  → init container สองตัวเสร็จ ตัวที่ 3 กำลังรัน
# PodInitializing → init containers ทั้งหมดเสร็จ กำลัง start main
# Running  → main containers กำลังรัน
```

### ดู Logs ของ Init Container

```bash
# ดู logs ของ init container โดยระบุชื่อ
kubectl logs my-pod -c wait-for-database -n production

# ดู logs ของ init container ที่ crash แล้ว
kubectl logs my-pod -c run-migration -n production --previous

# ดู logs แบบ follow
kubectl logs my-pod -c wait-for-database -n production -f

# ดู logs ทุก container ใน pod
kubectl logs my-pod --all-containers -n production
```

### ดู Init Container Details

```bash
# ดู state ของ init containers
kubectl describe pod my-pod -n production | grep -A 20 "Init Containers:"

# Output ตัวอย่าง:
# Init Containers:
#   wait-for-database:
#     Container ID:  docker://abc123
#     Image:         postgres:14
#     State:          Running
#       Started:      Mon, 15 Jan 2024 10:00:00 +0700
#     Ready:          False
#   run-migration:
#     State:          Waiting
#       Reason:       PodInitializing

# ดู events ที่เกี่ยวข้อง
kubectl get events -n production --field-selector involvedObject.name=my-pod
```

### Debug Init Container ที่ล้มเหลว

```bash
# ดู exit code
kubectl describe pod my-pod -n production | grep -A 5 "Last State:"
# Last State: Terminated
#   Reason: Error
#   Exit Code: 1

# ดู logs ของ failed init container
kubectl logs my-pod -c failed-init -n production --previous

# Exec เข้า init container ที่รันอยู่ (ถ้ายังรัน)
kubectl exec -it my-pod -c wait-for-database -n production -- sh

# สร้าง temporary pod เพื่อ debug
kubectl run debug-init \
  --image=postgres:14 \
  -it --rm \
  --restart=Never \
  -n production \
  -- sh -c "pg_isready -h postgres.production.svc.cluster.local -U postgres"
```

### ปัญหา: Init Container ติด Infinite Loop

```bash
# ดูว่า init container รันนานแค่ไหน
kubectl get pod my-pod -n production
# Init:0/2  10m  ← รัน 10 นาทีแล้วยังไม่เสร็จ

# ดู logs
kubectl logs my-pod -c wait-for-database -n production -f

# ถ้าต้องการหยุด: ลบ pod (deployment จะสร้างใหม่)
kubectl delete pod my-pod -n production

# แก้ปัญหาก่อน: service ที่รอไม่ทำงาน?
kubectl get svc -n production | grep postgres
kubectl get pods -n production -l app=postgres
```

---

## Best Practices

### 1. ใช้ Init Container แทน Application Logic

```yaml
# ไม่ดี: ใช้ main container รอ dependencies
containers:
- name: app
  command:
  - sh
  - -c
  - |
    # รอ database ใน main container (ทำให้ container start ช้า)
    until nc -z postgres 5432; do sleep 1; done
    start_application.sh

# ดี: ใช้ init container
initContainers:
- name: wait-db
  command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 1; done']

containers:
- name: app
  command: ['start_application.sh']  # เริ่มได้ทันที
```

### 2. ตั้ง Timeout สำหรับ Init Containers

```yaml
initContainers:
- name: wait-for-service
  command:
  - sh
  - -c
  - |
    # ตั้ง timeout เพื่อป้องกัน infinite wait
    timeout 300 sh -c 'until nc -z $SERVICE_HOST $SERVICE_PORT; do sleep 2; done'
    if [ $? -ne 0 ]; then
      echo "Timeout waiting for service!"
      exit 1
    fi
```

### 3. ใช้ activeDeadlineSeconds สำหรับ Pod

```yaml
spec:
  # ถ้า init containers ไม่เสร็จใน 600 วินาที → fail pod
  activeDeadlineSeconds: 600
  initContainers:
  - name: wait-for-database
    ...
```

### 4. Idempotent Migration

```yaml
initContainers:
- name: run-migration
  command:
  - sh
  - -c
  - |
    # Migration ต้องรัน idempotent ได้
    # กรณี pod restart หรือ scale → migration อาจรันหลายครั้ง
    python manage.py migrate --no-input
    # Django's migrate command เป็น idempotent
    # (ไม่รัน migrations ที่ apply แล้ว)
```

### 5. Minimal Images สำหรับ Init Containers

```yaml
initContainers:
# ดี: ใช้ image ขนาดเล็ก
- name: wait-for-postgres
  image: postgres:14-alpine  # Alpine = เล็กกว่า
  command: ['sh', '-c', 'until pg_isready -h postgres; do sleep 1; done']

# หรือ busybox สำหรับ netcat
- name: wait-generic
  image: busybox:1.35
  command: ['sh', '-c', 'until nc -z $HOST $PORT; do sleep 1; done']

# ไม่ดี: ใช้ full image ใหญ่โดยไม่จำเป็น
- name: wait-for-postgres
  image: ubuntu:latest  # ใหญ่เกินไปสำหรับแค่รอ service
```

---

## สรุป

Init Containers เป็นเครื่องมือที่มีประโยชน์มากสำหรับ:

1. **Wait for Dependencies**: รอ database, message queue, external services
2. **Database Migration**: รัน schema migration ก่อน deploy
3. **Config Generation**: ดึง secrets, generate configuration files
4. **Setup Directories**: สร้าง directories, set permissions
5. **Clone Repositories**: ดึง code หรือ config จาก Git

ข้อสำคัญ:
- Init containers รัน **sequential** - ต้องเสร็จก่อน main container เริ่ม
- ใช้ **emptyDir volumes** เพื่อ share data ระหว่าง init และ main containers
- ตั้ง **timeout** เสมอเพื่อป้องกัน infinite loop
- Migration ต้องเป็น **idempotent** เพราะอาจรันหลายครั้ง

ในบทต่อไป เราจะเรียนรู้เรื่อง **Sidecar Containers** - containers เสริมที่รันควบคู่กับ main container
