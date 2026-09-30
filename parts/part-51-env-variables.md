# Part 51: Environment Variables ใน Kubernetes

## บทนำ

Environment Variables (ตัวแปรสภาพแวดล้อม) เป็นวิธีการส่งค่า configuration ให้กับ container ที่รันอยู่ใน Kubernetes Pod ซึ่งเป็นแนวทางที่ได้รับการยอมรับตาม The Twelve-Factor App methodology สำหรับการ configure แอปพลิเคชัน

ใน Kubernetes เราสามารถกำหนด environment variables ได้หลายวิธี:
1. กำหนดค่าตรงๆ (literal values)
2. อ้างอิงจาก ConfigMap
3. อ้างอิงจาก Secret
4. อ้างอิงจาก field ของ Pod (Downward API)
5. อ้างอิงจาก resource limits/requests

---

## 51.1 การกำหนด Environment Variables แบบพื้นฐาน

### Literal Values

วิธีที่ง่ายที่สุดคือการกำหนดค่าตรงๆ ใน manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
  namespace: default
spec:
  containers:
  - name: env-demo-container
    image: nginx:1.25
    env:
    - name: DEMO_GREETING
      value: "สวัสดี Kubernetes"
    - name: DEMO_FAREWELL
      value: "ลาก่อน"
    - name: APP_ENV
      value: "production"
    - name: APP_PORT
      value: "8080"
    - name: APP_DEBUG
      value: "false"
```

### การตรวจสอบ Environment Variables

```bash
# ดู environment variables ของ container
kubectl exec -it env-demo -- env

# ดูเฉพาะ variable ที่ต้องการ
kubectl exec -it env-demo -- printenv DEMO_GREETING

# ดู environment จาก describe
kubectl describe pod env-demo
```

### ข้อจำกัดของ Literal Values

- ค่าที่เป็น sensitive ข้อมูล (password, token) ไม่ควรเขียนตรงๆ
- การเปลี่ยนค่าต้องแก้ manifest และ restart pod
- ไม่สามารถ share ข้ามหลาย pod ได้โดยง่าย

---

## 51.2 valueFrom: configMapKeyRef

การอ้างอิงค่าจาก ConfigMap ทำให้เราสามารถแยก configuration ออกจาก application code ได้

### สร้าง ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
  labels:
    app: myapp
    environment: production
data:
  # ค่า string ทั่วไป
  APP_NAME: "MyApplication"
  APP_ENV: "production"
  APP_PORT: "8080"
  APP_LOG_LEVEL: "info"
  
  # URL และ endpoint
  DATABASE_HOST: "postgres-service.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "myapp_db"
  
  # Feature flags
  FEATURE_DARK_MODE: "true"
  FEATURE_BETA_UI: "false"
  
  # Cache configuration
  REDIS_HOST: "redis-service.default.svc.cluster.local"
  REDIS_PORT: "6379"
  CACHE_TTL: "3600"
```

```bash
# สร้าง ConfigMap
kubectl apply -f app-config.yaml

# ดูข้อมูล ConfigMap
kubectl get configmap app-config
kubectl describe configmap app-config

# ดูข้อมูลแบบ yaml
kubectl get configmap app-config -o yaml
```

### ใช้ configMapKeyRef

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        env:
        # อ้างอิงจาก ConfigMap แบบเฉพาะ key
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config      # ชื่อ ConfigMap
              key: APP_NAME         # key ใน ConfigMap
        
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_PORT
        
        # Optional: ถ้า ConfigMap หรือ key ไม่มี ก็ไม่ error
        - name: OPTIONAL_SETTING
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: OPTIONAL_SETTING
              optional: true
        
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### ConfigMap กับ Namespace

```yaml
# ConfigMap ต้องอยู่ใน namespace เดียวกับ Pod
# ถ้า Pod อยู่ใน namespace "production" ต้องสร้าง ConfigMap ใน "production" ด้วย

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production  # ต้องตรงกัน
data:
  APP_ENV: "production"
  DATABASE_HOST: "postgres-prod.production.svc.cluster.local"
```

---

## 51.3 valueFrom: secretKeyRef

Secret ใช้สำหรับเก็บข้อมูลที่เป็นความลับ เช่น passwords, tokens, certificates

### สร้าง Secret

```bash
# สร้าง Secret จาก command line (แนะนำ)
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD='mySecurePassword123' \
  --from-literal=API_KEY='abc123xyz789' \
  --from-literal=JWT_SECRET='myJWTSecret456' \
  --namespace=default

# ดู Secret (จะแสดง base64 encoded)
kubectl get secret app-secrets -o yaml
```

### Secret จาก YAML (ต้อง base64 encode ก่อน)

```bash
# encode ค่าเป็น base64
echo -n 'mySecurePassword123' | base64
# output: bXlTZWN1cmVQYXNzd29yZDEyMw==

echo -n 'abc123xyz789' | base64
# output: YWJjMTIzeHl6Nzg5

echo -n 'myJWTSecret456' | base64
# output: bXlKV1RTZWNyZXQ0NTY=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: default
  labels:
    app: myapp
    security: sensitive
type: Opaque
data:
  # ค่าต้อง base64 encoded
  DB_PASSWORD: bXlTZWN1cmVQYXNzd29yZDEyMw==
  API_KEY: YWJjMTIzeHl6Nzg5
  JWT_SECRET: bXlKV1RTZWNyZXQ0NTY=
```

### ใช้ secretKeyRef

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        env:
        # อ้างอิงจาก ConfigMap
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        
        # อ้างอิงจาก Secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets    # ชื่อ Secret
              key: DB_PASSWORD     # key ใน Secret
        
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: API_KEY
        
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: JWT_SECRET
        
        # Optional Secret
        - name: OPTIONAL_TOKEN
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: OPTIONAL_TOKEN
              optional: true
```

---

## 51.4 envFrom: การโหลด Environment Variables ทั้งหมดจาก ConfigMap/Secret

แทนที่จะระบุทีละ key เราสามารถโหลดทุก key จาก ConfigMap หรือ Secret ได้ด้วย `envFrom`

### envFrom ConfigMap

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        # โหลดทุก key จาก ConfigMap เป็น env vars
        envFrom:
        - configMapRef:
            name: app-config
        
        # โหลดจาก Secret ด้วย
        - secretRef:
            name: app-secrets
        
        # เพิ่ม prefix เพื่อป้องกัน key ชนกัน
        - configMapRef:
            name: database-config
            prefix: "DB_"         # ทุก key จะมี DB_ นำหน้า
        
        - configMapRef:
            name: cache-config
            prefix: "CACHE_"      # ทุก key จะมี CACHE_ นำหน้า
        
        # Optional: ถ้าไม่มี ConfigMap/Secret จะไม่ error
        - configMapRef:
            name: optional-config
            optional: true
```

### ข้อควรระวังกับ envFrom

```yaml
# ถ้า ConfigMap และ Secret มี key เดียวกัน
# key ที่ประกาศหลังจะ override key ก่อนหน้า

# ConfigMap A มี: DATABASE_URL=config-value
# Secret B มี: DATABASE_URL=secret-value

envFrom:
- configMapRef:
    name: config-a    # DATABASE_URL=config-value
- secretRef:
    name: secret-b    # DATABASE_URL=secret-value (override)

# ผลลัพธ์: DATABASE_URL=secret-value
```

### ผสม envFrom กับ env

```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    
    # โหลดทุกอย่างจาก ConfigMap ก่อน
    envFrom:
    - configMapRef:
        name: base-config
    
    # แล้ว override หรือเพิ่ม individual values
    env:
    - name: APP_OVERRIDE
      value: "custom-value"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD
    # individual env vars จะ override envFrom
```

---

## 51.5 Downward API: อ้างอิง Pod/Node Information

Downward API ช่วยให้ container สามารถรู้ข้อมูลเกี่ยวกับตัวเองและ cluster ได้

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo
  namespace: default
  labels:
    app: demo
    version: "1.0"
  annotations:
    build-date: "2024-01-15"
    git-commit: "abc123"
spec:
  containers:
  - name: main
    image: nginx:1.25
    env:
    # ข้อมูลของ Pod เอง
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    
    - name: MY_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    
    - name: MY_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    
    - name: MY_SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    
    # Labels และ Annotations
    - name: APP_LABEL
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['app']
    
    - name: BUILD_DATE
      valueFrom:
        fieldRef:
          fieldPath: metadata.annotations['build-date']
    
    # Resource limits/requests
    - name: MY_CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
    
    - name: MY_MEM_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1Mi   # แสดงเป็น MiB
    
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

---

## 51.6 Best Practices สำหรับ Environment Variables

### 1. อย่าเก็บ Sensitive Data ใน env literals

```yaml
# ไม่ดี - password เขียนตรงๆ
env:
- name: DB_PASSWORD
  value: "mypassword123"   # อย่าทำแบบนี้!

# ดี - อ้างอิงจาก Secret
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

### 2. ใช้ Namespace เพื่อแยก Environment

```bash
# สร้าง namespace สำหรับแต่ละ environment
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# สร้าง ConfigMap ในแต่ละ namespace
kubectl create configmap app-config \
  --from-literal=APP_ENV=development \
  --namespace=development

kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --namespace=production
```

### 3. Naming Convention

```yaml
# ใช้ชื่อ env var แบบ UPPERCASE_WITH_UNDERSCORES
env:
- name: DATABASE_HOST      # ดี
- name: databaseHost       # ไม่แนะนำ
- name: database-host      # ไม่ถูกต้อง (เป็น invalid env var name)
```

### 4. Group Related Variables

```yaml
# ใช้ prefix สำหรับ group ที่เกี่ยวข้องกัน
env:
- name: DB_HOST
- name: DB_PORT
- name: DB_NAME
- name: DB_USER
- name: DB_PASSWORD

- name: CACHE_HOST
- name: CACHE_PORT
- name: CACHE_TTL

- name: LOG_LEVEL
- name: LOG_FORMAT
- name: LOG_FILE
```

### 5. Document Environment Variables

```yaml
# ใช้ annotations เพื่อ document
metadata:
  annotations:
    env-vars/DB_HOST: "Hostname ของ PostgreSQL database"
    env-vars/DB_PORT: "Port ของ database (default: 5432)"
    env-vars/API_KEY: "API key สำหรับ external service (from Secret)"
```

---

## 51.7 Workshop: Configure App ด้วย Environment Variables

### สถานการณ์

เราจะสร้าง web application ที่มีการ configure ผ่าน environment variables โดยมี:
- ConfigMap สำหรับ non-sensitive configuration
- Secret สำหรับ database credentials
- Downward API สำหรับ pod information

### Step 1: สร้าง Namespace

```bash
kubectl create namespace workshop-env
kubectl config set-context --current --namespace=workshop-env
```

### Step 2: สร้าง ConfigMap

```yaml
# app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: workshop-env
  labels:
    app: webapp
    tier: frontend
data:
  # Application settings
  APP_NAME: "Workshop Web Application"
  APP_VERSION: "1.0.0"
  APP_ENV: "development"
  APP_PORT: "8080"
  APP_HOST: "0.0.0.0"
  
  # Logging
  LOG_LEVEL: "debug"
  LOG_FORMAT: "json"
  LOG_OUTPUT: "stdout"
  
  # Database connection (non-sensitive)
  DB_HOST: "postgres-service.workshop-env.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "workshop_db"
  DB_MAX_CONNECTIONS: "10"
  DB_CONNECT_TIMEOUT: "30"
  
  # Cache settings
  REDIS_HOST: "redis-service.workshop-env.svc.cluster.local"
  REDIS_PORT: "6379"
  CACHE_DEFAULT_TTL: "3600"
  
  # Feature flags
  FEATURE_USER_REGISTRATION: "true"
  FEATURE_EMAIL_NOTIFICATIONS: "false"
  FEATURE_DARK_MODE: "true"
```

```bash
kubectl apply -f app-configmap.yaml
kubectl get configmap webapp-config -n workshop-env
kubectl describe configmap webapp-config -n workshop-env
```

### Step 3: สร้าง Secret

```bash
# สร้าง Secret สำหรับ database credentials
kubectl create secret generic webapp-secrets \
  --from-literal=DB_PASSWORD='Workshop@2024Secure' \
  --from-literal=DB_USER='webapp_user' \
  --from-literal=API_SECRET_KEY='s3cr3t-k3y-for-api-auth' \
  --from-literal=JWT_SIGNING_KEY='jwt-signing-secret-2024' \
  --from-literal=SMTP_PASSWORD='smtp-password-123' \
  --namespace=workshop-env

# ตรวจสอบ Secret ถูกสร้าง
kubectl get secret webapp-secrets -n workshop-env
kubectl describe secret webapp-secrets -n workshop-env
```

### Step 4: สร้าง Deployment

```yaml
# webapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: workshop-env
  labels:
    app: webapp
    version: "1.0.0"
  annotations:
    description: "Workshop web application deployment"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: "1.0.0"
      annotations:
        git-commit: "abc123def456"
        deploy-date: "2024-01-15"
    spec:
      containers:
      - name: webapp
        image: nginx:1.25   # ใช้ nginx แทน เพื่อ demo
        ports:
        - containerPort: 80
          name: http
        
        # โหลดทุก key จาก ConfigMap
        envFrom:
        - configMapRef:
            name: webapp-config
        
        # เพิ่ม sensitive values จาก Secret
        env:
        # Database credentials จาก Secret
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: DB_USER
        
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: DB_PASSWORD
        
        - name: API_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: API_SECRET_KEY
        
        - name: JWT_SIGNING_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secrets
              key: JWT_SIGNING_KEY
        
        # Downward API - ข้อมูลของ Pod
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        # Resource information
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: webapp
              resource: limits.cpu
        
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: webapp
              resource: limits.memory
              divisor: 1Mi
        
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "250m"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

```bash
kubectl apply -f webapp-deployment.yaml
kubectl get pods -n workshop-env
```

### Step 5: ตรวจสอบ Environment Variables

```bash
# รอ pod ขึ้น
kubectl wait --for=condition=ready pod -l app=webapp -n workshop-env --timeout=60s

# ดู environment variables ทั้งหมด
POD_NAME=$(kubectl get pod -l app=webapp -n workshop-env -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -n workshop-env -- env | sort

# ดูเฉพาะ vars ที่เราสนใจ
kubectl exec -it $POD_NAME -n workshop-env -- sh -c '
echo "=== Application Config ==="
echo "APP_NAME: $APP_NAME"
echo "APP_ENV: $APP_ENV"
echo "APP_VERSION: $APP_VERSION"

echo ""
echo "=== Database Config ==="
echo "DB_HOST: $DB_HOST"
echo "DB_PORT: $DB_PORT"
echo "DB_NAME: $DB_NAME"
echo "DB_USER: $DB_USER"
echo "DB_PASSWORD: [HIDDEN - length: ${#DB_PASSWORD}]"

echo ""
echo "=== Pod Information (Downward API) ==="
echo "POD_NAME: $POD_NAME"
echo "POD_NAMESPACE: $POD_NAMESPACE"
echo "POD_IP: $POD_IP"
echo "NODE_NAME: $NODE_NAME"

echo ""
echo "=== Resource Limits ==="
echo "CPU_LIMIT: $CPU_LIMIT"
echo "MEMORY_LIMIT: ${MEMORY_LIMIT}Mi"
'
```

### Step 6: ทดสอบการอัพเดต ConfigMap

```bash
# แก้ไข ConfigMap
kubectl edit configmap webapp-config -n workshop-env
# เปลี่ยน LOG_LEVEL จาก "debug" เป็น "info"

# หรือใช้ patch
kubectl patch configmap webapp-config -n workshop-env \
  --type merge \
  -p '{"data":{"LOG_LEVEL":"info","APP_VERSION":"1.0.1"}}'

# ดูการเปลี่ยนแปลง
kubectl get configmap webapp-config -n workshop-env -o yaml

# Restart deployment เพื่อ pick up ค่าใหม่
# (env vars จาก ConfigMap ไม่ hot-reload อัตโนมัติ ต้อง restart)
kubectl rollout restart deployment/webapp -n workshop-env
kubectl rollout status deployment/webapp -n workshop-env

# ตรวจสอบค่าใหม่
POD_NAME=$(kubectl get pod -l app=webapp -n workshop-env -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -n workshop-env -- printenv LOG_LEVEL APP_VERSION
```

### Step 7: การ Debug Environment Variables

```bash
# ดู events ถ้ามีปัญหา
kubectl describe pod $POD_NAME -n workshop-env

# ถ้า pod ไม่ขึ้นเพราะหา ConfigMap/Secret ไม่เจอ จะเห็น error:
# Error: configmap "webapp-config" not found
# Error: secret "webapp-secrets" not found

# ตรวจสอบ ConfigMap และ Secret มีอยู่
kubectl get configmap,secret -n workshop-env

# ดู log ของ pod
kubectl logs $POD_NAME -n workshop-env
```

### Step 8: การใช้ Environment Variables แบบ Production-Ready

```yaml
# production-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-prod
  namespace: production
  labels:
    app: webapp
    environment: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      environment: production
  template:
    metadata:
      labels:
        app: webapp
        environment: production
    spec:
      # ใช้ ServiceAccount ที่จำกัดสิทธิ์
      serviceAccountName: webapp-sa
      
      # ไม่ mount service account token โดยอัตโนมัติ
      automountServiceAccountToken: false
      
      containers:
      - name: webapp
        image: myapp:1.0.0
        
        # โหลดจาก ConfigMap
        envFrom:
        - configMapRef:
            name: webapp-prod-config
            
        # Sensitive values จาก Secret เท่านั้น
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-prod-secrets
              key: DB_PASSWORD
        
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-prod-secrets
              key: API_KEY
        
        # Downward API
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        
        # Security Context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
```

### Step 9: Cleanup

```bash
# ลบทุกอย่างที่สร้างไว้ใน workshop
kubectl delete namespace workshop-env

# ตรวจสอบว่าลบแล้ว
kubectl get namespace workshop-env
```

---

## 51.8 การ Debug ปัญหาทั่วไป

### ปัญหา: Pod ไม่ขึ้น เพราะหา ConfigMap/Secret ไม่เจอ

```bash
# ดู error message
kubectl describe pod <pod-name>
# จะเห็น: Error creating: configmap "app-config" not found

# แก้ไข: สร้าง ConfigMap ก่อน deploy
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
```

### ปัญหา: Key ไม่มีใน ConfigMap

```bash
# ดู error
kubectl describe pod <pod-name>
# จะเห็น: Error: couldn't find key "MY_KEY" in ConfigMap "my-config"

# แก้ไข: ใช้ optional: true หรือเพิ่ม key ใน ConfigMap
```

### ปัญหา: Secret ถูก base64 decode ผิด

```bash
# ตรวจสอบค่าใน Secret
kubectl get secret my-secret -o jsonpath='{.data.MY_KEY}' | base64 -d
echo ""  # เพิ่ม newline

# ตรวจสอบว่าค่าถูกต้อง
kubectl exec -it <pod> -- printenv MY_KEY
```

### ปัญหา: Environment Variable ไม่ได้รับการ Update

```bash
# Environment Variables จาก ConfigMap/Secret จะไม่ update อัตโนมัติ
# ต้อง restart pod

# วิธีที่ 1: Rollout restart
kubectl rollout restart deployment <deployment-name>

# วิธีที่ 2: Delete pod (ถ้ามี ReplicaSet จะสร้างใหม่)
kubectl delete pod <pod-name>

# วิธีที่ 3: Scale down แล้ว scale up
kubectl scale deployment <name> --replicas=0
kubectl scale deployment <name> --replicas=3
```

---

## 51.9 Security Best Practices

### 1. ไม่ใส่ Secrets ใน ConfigMap

```yaml
# ไม่ดี - password ใน ConfigMap
apiVersion: v1
kind: ConfigMap
data:
  DB_PASSWORD: "secret123"  # อย่าทำแบบนี้!

# ดี - ใช้ Secret
apiVersion: v1
kind: Secret
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0MTIz   # base64 encoded
```

### 2. ใช้ RBAC จำกัดการเข้าถึง Secret

```yaml
# role สำหรับอ่าน Secret เฉพาะบาง namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["webapp-secrets"]  # ระบุชื่อ Secret ที่อนุญาต
  verbs: ["get"]
```

### 3. Enable Encryption at Rest สำหรับ Secrets

```yaml
# /etc/kubernetes/encryption-config.yaml (ใน kube-apiserver)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

### 4. ไม่ Log Environment Variables ที่เป็น Sensitive

```bash
# อย่าทำแบบนี้ใน container logs:
echo "Database password: $DB_PASSWORD"  # ห้ามเด็ดขาด!

# ทำแบบนี้แทน:
echo "Database connection: $DB_HOST:$DB_PORT/$DB_NAME"  # ไม่แสดง credentials
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Literal Values**: การกำหนดค่าตรงๆ ใน manifest
2. **configMapKeyRef**: การอ้างอิงค่าเฉพาะจาก ConfigMap
3. **secretKeyRef**: การอ้างอิงค่าจาก Secret (สำหรับข้อมูลลับ)
4. **envFrom**: การโหลดทุก key จาก ConfigMap หรือ Secret
5. **Downward API**: การอ้างอิงข้อมูลของ Pod/Node เอง

**Key Takeaways:**
- แยก configuration (ConfigMap) ออกจาก sensitive data (Secret)
- ใช้ envFrom สำหรับการโหลดจำนวนมาก แต่ระวัง key collision
- Environment variables ไม่ hot-reload ต้อง restart pod
- ใช้ RBAC จำกัดการเข้าถึง Secrets
- ไม่เคย log ค่า sensitive ใน application logs

---

**ต่อไป**: Part 52 - ConfigMap Advanced (Immutability, Volume, Hot-reload)
