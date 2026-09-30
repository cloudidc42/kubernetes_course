# Part 43: PersistentVolumeClaim (PVC) - การขอใช้ Storage

## บทนำ

**PersistentVolumeClaim (PVC)** คือ request สำหรับ storage โดย user หรือ application PVC เป็นสิ่งที่ Developer สร้างเพื่อขอใช้ storage โดยไม่ต้องสนใจรายละเอียดของ underlying storage infrastructure

### Analogy ที่เข้าใจง่าย

คิดว่า PV เหมือนห้องในอพาร์ตเมนต์ที่มีอยู่แล้ว และ PVC เหมือนสัญญาเช่าห้องนั้น:
- **PV** = ห้องที่มีคุณสมบัติ (ขนาด, สิ่งอำนวยความสะดวก)
- **PVC** = สัญญาเช่าที่ระบุความต้องการ (ต้องการห้องขนาดไหน, มีอะไรบ้าง)
- **Binding** = เมื่อ landlord (Kubernetes) จับคู่ห้องกับสัญญาเช่า
- **Pod** = ผู้เช่าที่ใช้ห้องนั้น

---

## 1. PVC คืออะไร

### คุณสมบัติ PVC:
- เป็น namespace-scoped resource
- ระบุความต้องการ storage (ขนาด, access mode, storage class)
- Kubernetes จะหา PV ที่ตรงกันให้อัตโนมัติ
- หลาย Pods สามารถใช้ PVC เดียวกันได้ (ขึ้นอยู่กับ access mode)

### ขั้นตอนการทำงาน

```
1. Admin สร้าง PV (หรือ StorageClass provision อัตโนมัติ)
2. Developer สร้าง PVC ระบุ:
   - storageClassName
   - accessModes  
   - storage size
   - (optional) selector labels
3. Kubernetes ทำ Binding:
   - หา PV ที่ capacity >= PVC request
   - Access modes ครอบคลุม PVC request
   - StorageClass ตรงกัน
4. Pod ใช้ PVC ผ่าน volumes section
```

---

## 2. PVC YAML - Manifests ต่างๆ

### 2.1 PVC พื้นฐาน

```yaml
# basic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
  labels:
    app: my-app
    environment: development
spec:
  # Access Mode ที่ต้องการ
  accessModes:
  - ReadWriteOnce
  
  # ขนาด Storage ที่ต้องการ
  resources:
    requests:
      storage: 5Gi
  
  # StorageClass ที่ต้องการ (ถ้าไม่ระบุจะใช้ default)
  storageClassName: standard
```

### 2.2 PVC พร้อม Selector

```yaml
# pvc-with-selector.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  
  resources:
    requests:
      storage: 100Gi
  
  storageClassName: ssd-storage
  
  # เลือก PV ที่มี labels ที่ต้องการ
  selector:
    matchLabels:
      storage-type: ssd
      environment: production
    matchExpressions:
    - key: tier
      operator: In
      values:
      - database
      - high-performance
```

### 2.3 PVC สำหรับ Database

```yaml
# postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: databases
  labels:
    app: postgresql
    component: data
  annotations:
    description: "PostgreSQL data directory"
    owner: "database-team"
    backup-policy: "daily"
spec:
  accessModes:
  - ReadWriteOnce    # Database ต้องการ RWO
  
  resources:
    requests:
      storage: 500Gi
  
  storageClassName: fast-ssd
  
  volumeMode: Filesystem
```

### 2.4 PVC สำหรับ Shared Storage (NFS)

```yaml
# shared-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-files-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany    # Shared storage ต้องการ RWX
  
  resources:
    requests:
      storage: 1Ti
  
  storageClassName: nfs-storage
```

### 2.5 PVC สำหรับ ReadOnly

```yaml
# readonly-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-assets-pvc
  namespace: web
spec:
  accessModes:
  - ReadOnlyMany    # Read-only shared
  
  resources:
    requests:
      storage: 50Gi
  
  storageClassName: slow-hdd
```

### 2.6 Block Volume PVC

```yaml
# block-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-block-pvc
  namespace: databases
spec:
  accessModes:
  - ReadWriteOnce
  
  volumeMode: Block    # Raw block device
  
  resources:
    requests:
      storage: 200Gi
  
  storageClassName: local-block
```

---

## 3. Binding Process ทำงานอย่างไร

### 3.1 Binding Algorithm

Kubernetes จะหา PV ที่เหมาะสมโดยพิจารณา:

1. **StorageClass ต้องตรงกัน** (หรือทั้งคู่ต้องไม่มี StorageClass)
2. **Access modes ของ PV ต้องครอบคลุม** access modes ที่ PVC ขอ
3. **Capacity ของ PV ต้องมากกว่าหรือเท่ากับ** ที่ PVC ขอ
4. **Selector (ถ้ามี) ต้อง match** กับ labels ของ PV
5. **Volume mode ต้องตรงกัน** (Filesystem หรือ Block)

### 3.2 Binding States

```
PVC Created
    |
    v
┌─────────────────────────────────────────┐
│         Kubernetes Control Plane        │
│                                         │
│  Find matching PV:                      │
│  - StorageClass match?           YES ─────────→ Bind PV to PVC
│  - Access modes OK?              NO ──────────→ PVC stays Pending
│  - Capacity sufficient?                 │         (retry when PV available)
│  - Selector match?                      │
└─────────────────────────────────────────┘
                                          |
                        PVC: Bound ←──────┘
                        PV: Bound
```

### 3.3 Binding Modes

**Immediate (Default):**
- PVC จะ bind ทันทีที่พบ PV ที่ตรงกัน
- ไม่รอว่า Pod จะถูก schedule ที่ Node ไหน
- ปัญหา: อาจ bind PV ที่ Node ไม่ใช่ Node ที่ Pod จะถูก schedule

**WaitForFirstConsumer:**
- PVC จะ Pending จนกว่า Pod ที่ใช้ PVC จะถูก schedule
- เมื่อ Pod ถูก schedule, Kubernetes จะ bind PV ที่ Node เดียวกัน
- เหมาะสำหรับ Local PV และ Topology-aware storage

---

## 4. ใช้ PVC ใน Pod

### 4.1 Pod ใช้ PVC พื้นฐาน

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: storage      # ต้อง match กับ volumes section
      mountPath: /data
  
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc   # ชื่อ PVC ที่สร้างไว้
```

### 4.2 ReadOnly PVC Mount

```yaml
# readonly-mount.yaml
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
spec:
  containers:
  - name: reader
    image: nginx:1.25
    volumeMounts:
    - name: shared-data
      mountPath: /data
      readOnly: true     # Mount as read-only
  
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: shared-files-pvc
      readOnly: true     # PVC mount mode
```

### 4.3 หลาย Containers ใช้ PVC เดียวกัน

```yaml
# multi-container-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-app
spec:
  containers:
  - name: writer
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      while true; do
        date >> /data/timestamps.log
        sleep 5
      done
    volumeMounts:
    - name: shared-storage
      mountPath: /data
  
  - name: reader
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      while true; do
        echo "Latest entries:"
        tail -5 /data/timestamps.log 2>/dev/null || echo "No data yet"
        sleep 10
      done
    volumeMounts:
    - name: shared-storage
      mountPath: /data
      readOnly: true   # Reader เข้าถึงแบบ read-only
  
  volumes:
  - name: shared-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### 4.4 Deployment ใช้ PVC

```yaml
# deployment-with-pvc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
spec:
  replicas: 1    # RWO PVC ต้องมี replicas=1 หรือใช้ RWX
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
        image: nginx:1.25
        volumeMounts:
        - name: web-data
          mountPath: /var/www/html
        - name: web-logs
          mountPath: /var/log/nginx
      
      volumes:
      - name: web-data
        persistentVolumeClaim:
          claimName: web-data-pvc
      - name: web-logs
        persistentVolumeClaim:
          claimName: web-logs-pvc
```

---

## 5. PVC Expansion (ขยาย Storage)

### 5.1 Enable Volume Expansion ใน StorageClass

```yaml
# expandable-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: docker.io/hostpath
allowVolumeExpansion: true    # Enable expansion
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 5.2 ขยาย PVC

```bash
# ดู PVC ปัจจุบัน
kubectl get pvc my-pvc

# แก้ไข PVC size
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# หรือแก้ไขด้วย edit
kubectl edit pvc my-pvc
# เปลี่ยน storage: 5Gi เป็น storage: 20Gi

# ดู progress การขยาย
kubectl get pvc my-pvc -w
kubectl describe pvc my-pvc | grep -A 5 "Conditions:"
```

### 5.3 Online vs Offline Expansion

**Online Expansion** (รองรับโดย storage)
- ขยายได้โดยไม่ต้อง restart Pod
- เช่น: AWS EBS gp2/gp3, GCE PD

**Offline Expansion** (บาง storage)
- ต้อง unmount Volume ก่อน (ลบ Pod)
- File system จะ resize เมื่อ Pod restart

---

## 6. PVC ใน Namespace ต่างๆ

PVC เป็น namespaced resource ดังนั้น Pod ต้องอยู่ใน Namespace เดียวกับ PVC

```bash
# สร้าง PVC ใน namespace production
kubectl create namespace production

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prod-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
EOF

# Pod ใน production namespace ใช้ PVC นี้ได้
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: prod-app
  namespace: production
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: prod-pvc   # ต้องอยู่ namespace เดียวกัน
EOF
```

---

## 7. PVC Lifecycle Management

### 7.1 PVC Protection

Kubernetes ป้องกัน PVC จากการลบในขณะที่ Pod ยังใช้งานอยู่

```bash
# ดู Finalizers บน PVC
kubectl get pvc my-pvc -o jsonpath='{.metadata.finalizers}'
# Output: ["kubernetes.io/pvc-protection"]

# พยายามลบ PVC ที่ Pod ยังใช้อยู่
kubectl delete pvc my-pvc
# PVC จะเป็น Terminating state จนกว่า Pod จะถูกลบก่อน

# ลบ Pod ก่อน แล้ว PVC จะถูกลบเอง
kubectl delete pod app-with-storage
kubectl get pvc my-pvc  # จะหายไปแล้ว
```

### 7.2 Orphaned PVCs

```bash
# หา PVCs ที่ไม่มี Pod ใช้งาน
kubectl get pvc -A -o json | jq -r '
  .items[] |
  select(.status.phase == "Bound") |
  "\(.metadata.namespace)/\(.metadata.name): \(.spec.resources.requests.storage)"
'

# ดู PVCs ที่ Pending นานเกินไป
kubectl get pvc -A --field-selector=status.phase=Pending
```

---

## 8. Workshop: Deploy Database ด้วย PVC

### Workshop Overview
Deploy PostgreSQL database บน Kubernetes พร้อม Persistent Storage

### Prerequisites
- Kubernetes cluster (Minikube/Kind)
- kubectl configured
- เวลาประมาณ 60-90 นาที

### Step 1: เตรียม Namespace

```bash
kubectl create namespace database-workshop
kubectl config set-context --current --namespace=database-workshop
```

### Step 2: สร้าง StorageClass (ถ้ายังไม่มี default)

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

```bash
kubectl apply -f storage-class.yaml
```

### Step 3: สร้าง PV บน Node

```bash
# สร้าง directory บน Minikube
minikube ssh "sudo mkdir -p /mnt/data/postgres && sudo chmod 777 /mnt/data/postgres"

# ดู Node name
kubectl get nodes -o jsonpath='{.items[0].metadata.name}'
```

```yaml
# postgres-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  labels:
    app: postgres
    type: local
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  local:
    path: /mnt/data/postgres
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube    # เปลี่ยนเป็น node name ของคุณ
```

```bash
kubectl apply -f postgres-pv.yaml
kubectl get pv postgres-pv
```

### Step 4: สร้าง PVC สำหรับ PostgreSQL

```yaml
# postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: database-workshop
  labels:
    app: postgres
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
  selector:
    matchLabels:
      app: postgres
      type: local
```

```bash
kubectl apply -f postgres-pvc.yaml
kubectl get pvc postgres-pvc

# PVC จะเป็น Pending เพราะ WaitForFirstConsumer
```

### Step 5: สร้าง Secret สำหรับ PostgreSQL

```yaml
# postgres-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: database-workshop
type: Opaque
stringData:
  POSTGRES_DB: "workshopdb"
  POSTGRES_USER: "dbuser"
  POSTGRES_PASSWORD: "Workshop2024!"
  POSTGRES_HOST_AUTH_METHOD: "md5"
```

```bash
kubectl apply -f postgres-secret.yaml
```

### Step 6: สร้าง PostgreSQL ConfigMap

```yaml
# postgres-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: database-workshop
data:
  postgresql.conf: |
    # Performance settings
    max_connections = 100
    shared_buffers = 128MB
    effective_cache_size = 512MB
    maintenance_work_mem = 64MB
    
    # Write-ahead logging
    wal_level = replica
    max_wal_senders = 3
    wal_keep_size = 64MB
    
    # Logging
    log_destination = 'stderr'
    logging_collector = on
    log_directory = '/var/log/postgresql'
    log_filename = 'postgresql-%Y-%m-%d.log'
    log_statement = 'all'
    log_duration = on
    
    # Connection settings
    listen_addresses = '*'
    port = 5432
  
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    host    all             all             0.0.0.0/0               md5
  
  init.sql: |
    -- สร้าง database และ tables เริ่มต้น
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        username VARCHAR(50) UNIQUE NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE TABLE IF NOT EXISTS products (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        price DECIMAL(10,2) NOT NULL,
        stock INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    -- Insert sample data
    INSERT INTO users (username, email) VALUES 
        ('admin', 'admin@example.com'),
        ('user1', 'user1@example.com')
    ON CONFLICT DO NOTHING;
    
    INSERT INTO products (name, price, stock) VALUES 
        ('Product A', 29.99, 100),
        ('Product B', 49.99, 50),
        ('Product C', 9.99, 200)
    ON CONFLICT DO NOTHING;
```

```bash
kubectl apply -f postgres-configmap.yaml
```

### Step 7: สร้าง PostgreSQL Deployment

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: database-workshop
  labels:
    app: postgres
spec:
  replicas: 1     # Database ต้องมี replicas=1 เมื่อใช้ RWO PVC
  selector:
    matchLabels:
      app: postgres
  
  strategy:
    type: Recreate    # Recreate strategy สำหรับ database (ไม่ใช่ RollingUpdate)
  
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        
        ports:
        - containerPort: 5432
          name: postgres
        
        # Environment จาก Secret
        envFrom:
        - secretRef:
            name: postgres-secret
        
        # Volume mounts
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata      # subPath สำคัญมากสำหรับ postgres
        
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        
        - name: postgres-config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        
        - name: init-scripts
          mountPath: /docker-entrypoint-initdb.d/init.sql
          subPath: init.sql
        
        - name: postgres-logs
          mountPath: /var/log/postgresql
        
        # Resource limits
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        
        # Liveness probe
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
      
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
      
      - name: postgres-config
        configMap:
          name: postgres-config
      
      - name: init-scripts
        configMap:
          name: postgres-config
      
      - name: postgres-logs
        emptyDir: {}
```

```bash
kubectl apply -f postgres-deployment.yaml
kubectl get pods -n database-workshop -w
```

### Step 8: สร้าง Service สำหรับ PostgreSQL

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: database-workshop
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
  type: ClusterIP
---
# NodePort Service สำหรับ external access ในระหว่าง workshop
apiVersion: v1
kind: Service
metadata:
  name: postgres-external
  namespace: database-workshop
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    nodePort: 30432
  type: NodePort
```

```bash
kubectl apply -f postgres-service.yaml
```

### Step 9: ตรวจสอบและทดสอบ

```bash
# ดู Pod, PVC, PV status
kubectl get all -n database-workshop
kubectl get pvc -n database-workshop
kubectl get pv

# ดู logs
kubectl logs -f deployment/postgres -n database-workshop

# Test connection
kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c '\dt'

# Query ข้อมูล
kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c 'SELECT * FROM users;'

kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c 'SELECT * FROM products;'

# Insert ข้อมูลทดสอบ
kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c \
  "INSERT INTO users (username, email) VALUES ('testuser', 'test@example.com');"

# Port forward เพื่อ access จาก local
kubectl port-forward service/postgres 5432:5432 -n database-workshop &
psql -h localhost -U dbuser -d workshopdb
```

### Step 10: ทดสอบ Data Persistence

```bash
# เพิ่มข้อมูลสำคัญ
kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c \
  "INSERT INTO products (name, price, stock) VALUES ('Persistent Product', 99.99, 10);"

# ตรวจสอบข้อมูล
kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c "SELECT * FROM products;"

# ลบ Pod (Deployment จะสร้าง Pod ใหม่)
kubectl delete pod -l app=postgres -n database-workshop

# รอ Pod ใหม่ start
kubectl get pod -l app=postgres -n database-workshop -w

# ตรวจสอบว่าข้อมูลยังอยู่
kubectl exec -it deployment/postgres -n database-workshop -- \
  psql -U dbuser -d workshopdb -c "SELECT * FROM products;"
```

**ผลลัพธ์ที่ควรได้:** ข้อมูลทั้งหมดยังอยู่ รวมถึง 'Persistent Product' ที่เพิ่งเพิ่ม

### Step 11: ทดสอบ PVC ยังคง Bound หลัง Pod restart

```bash
# ดู PVC status ตลอดเวลา
watch kubectl get pvc -n database-workshop

# ใน terminal อื่น: ลบ Pod
kubectl delete pod -l app=postgres -n database-workshop
```

### Step 12: Backup ข้อมูลจาก PVC

```bash
# สร้าง backup job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-backup
  namespace: database-workshop
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:15-alpine
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting backup at $(date)"
          pg_dump -h postgres -U dbuser workshopdb > /backup/workshopdb-$(date +%Y%m%d-%H%M%S).sql
          echo "Backup completed"
          ls -la /backup/
        volumeMounts:
        - name: backup-storage
          mountPath: /backup
      
      volumes:
      - name: backup-storage
        emptyDir: {}
      
      restartPolicy: Never
EOF

# ดู backup job
kubectl get job postgres-backup -n database-workshop
kubectl logs job/postgres-backup -n database-workshop
```

### Step 13: ทำความสะอาด

```bash
# ลบทุกอย่างใน namespace
kubectl delete namespace database-workshop

# ลบ PV (ข้อมูลยังอยู่บน Node)
kubectl delete pv postgres-pv

# ลบข้อมูลบน Node (optional)
minikube ssh "sudo rm -rf /mnt/data/postgres"
```

---

## 9. PVC Best Practices

### 9.1 Resource Management

```yaml
# กำหนด requests และ limits ที่เหมาะสม
spec:
  resources:
    requests:
      storage: 10Gi    # ขอ minimum
    limits:
      storage: 50Gi    # Maximum (ถ้า StorageClass รองรับ)
```

### 9.2 Labels และ Annotations

```yaml
metadata:
  labels:
    app: myapp
    component: database
    environment: production
  annotations:
    backup-policy: "daily"
    owner: "platform-team"
    cost-center: "engineering"
    created-by: "helm"
```

### 9.3 Naming Convention

```
<app>-<component>-<environment>-pvc
เช่น:
- postgres-data-prod-pvc
- redis-cache-staging-pvc
- webapp-uploads-dev-pvc
```

### 9.4 StorageClass Selection

```yaml
# ใช้ StorageClass ที่เหมาะสมกับ workload
spec:
  storageClassName: ssd-fast      # สำหรับ database
  storageClassName: hdd-standard  # สำหรับ logs/archives
  storageClassName: nfs-shared    # สำหรับ shared files
```

---

## 10. PVC Monitoring

```bash
# ดู PVC ทั้งหมดใน cluster
kubectl get pvc -A

# ดู PVC ที่ Pending
kubectl get pvc -A --field-selector=status.phase=Pending

# ดู PVC ที่ ไม่มี Pod ใช้งาน (potential orphans)
kubectl get pvc -A -o json | jq -r '
  .items[] | 
  select(.status.phase == "Bound") |
  "\(.metadata.namespace)/\(.metadata.name)"
'

# ดูขนาดรวมของ PVC ทั้งหมด
kubectl get pvc -A -o json | jq '[.items[].spec.resources.requests.storage] | join(", ")'

# Describe PVC ที่มีปัญหา
kubectl describe pvc <pvc-name> | grep -A 10 "Events:"
```

---

## 11. Troubleshooting PVC

### ปัญหาที่พบบ่อย

**1. PVC ค้างอยู่ที่ Pending**
```bash
kubectl describe pvc <name>
# ดูใน Events section

# สาเหตุที่เป็นไปได้:
# - ไม่มี PV ที่ match
# - StorageClass ไม่มีอยู่
# - Access modes ไม่ตรง
# - Capacity ไม่พอ
# - Node ไม่มีพื้นที่ว่าง (สำหรับ local PV)
```

**2. PVC ไม่สามารถ bind กับ PV ที่มี capacity พอ**
```bash
# ตรวจสอบ StorageClass
kubectl get pvc <name> -o jsonpath='{.spec.storageClassName}'
kubectl get pv -l app=<app> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.storageClassName}{"\n"}{end}'

# StorageClass ต้องตรงกัน!
```

**3. Permission denied เมื่อ Pod เขียน data**
```bash
# ตรวจสอบ fsGroup
kubectl get pod <name> -o jsonpath='{.spec.securityContext}'

# เพิ่ม fsGroup
spec:
  securityContext:
    fsGroup: 1000   # หรือ group ของ application
```

**4. PVC stuck ที่ Terminating**
```bash
# Remove finalizer ด้วยตนเอง (use with caution!)
kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}'
```

---

## สรุป

| Concept | คำอธิบาย |
|---------|----------|
| PVC | Request สำหรับ storage โดย user |
| Binding | กระบวนการจับคู่ PVC กับ PV |
| Access Modes | RWO (exclusive), ROX (shared read), RWX (shared write) |
| Storage Size | ต้องน้อยกว่าหรือเท่ากับ PV |
| StorageClass | กำหนด storage tier |
| Volume Expansion | ขยาย PVC ได้ถ้า StorageClass รองรับ |

### Key Takeaways:
1. **PVC** เป็น namespace-scoped resource - Pod และ PVC ต้องอยู่ namespace เดียวกัน
2. **subPath** สำคัญมากสำหรับ PostgreSQL - ป้องกัน data directory conflict
3. **Recreate strategy** เหมาะสำหรับ Database Deployment กับ RWO PVC
4. **WaitForFirstConsumer** binding mode จำเป็นสำหรับ topology-aware storage
5. **PVC Protection** finalizer ป้องกันการลบ PVC ที่ยังใช้งานอยู่
6. เสมอ **ตรวจสอบ PV availability** ก่อน deploy production workloads

## แหล่งเรียนรู้เพิ่มเติม

- [PersistentVolumeClaims Documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
- [Configure Pod to Use PersistentVolume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
- [Expanding Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)
