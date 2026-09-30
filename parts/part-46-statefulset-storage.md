# Part 46: StatefulSet Storage - VolumeClaimTemplates และ Database HA

## บทนำ

**StatefulSet** เป็น workload API object ที่ใช้จัดการ stateful applications เช่น databases, message queues หรือ applications ที่ต้องการ stable identity และ persistent storage

### ความแตกต่างระหว่าง Deployment และ StatefulSet

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod Identity | Random names | Ordinal names (pod-0, pod-1) |
| Storage | Shared PVC | Each pod gets own PVC |
| Deployment Order | Parallel (default) | Sequential (0, 1, 2...) |
| Scale Down Order | Random | Reverse order (N, N-1...) |
| Pod Naming | Random | Predictable (name-0, name-1) |
| DNS | Generic | Stable hostname per pod |

---

## 1. VolumeClaimTemplates

**VolumeClaimTemplates** คือ feature ที่ทำให้ StatefulSet สร้าง PVC แยกสำหรับแต่ละ Pod โดยอัตโนมัติ

### โครงสร้างพื้นฐาน

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
spec:
  replicas: 3
  serviceName: my-service   # Headless service
  selector:
    matchLabels:
      app: my-app
  
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: data          # ต้อง match กับ VolumeClaimTemplate name
          mountPath: /data
  
  volumeClaimTemplates:       # PVC template สำหรับแต่ละ Pod
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### PVCs ที่ถูกสร้าง

เมื่อ StatefulSet มี 3 replicas จะได้ PVCs:
- `data-my-statefulset-0` (สำหรับ pod `my-statefulset-0`)
- `data-my-statefulset-1` (สำหรับ pod `my-statefulset-1`)
- `data-my-statefulset-2` (สำหรับ pod `my-statefulset-2`)

Pattern: `<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`

---

## 2. StatefulSet + PVC - รูปแบบต่างๆ

### 2.1 StatefulSet พร้อม Multiple VolumeClaimTemplates

```yaml
# statefulset-multi-pvc.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-cluster
  namespace: databases
spec:
  replicas: 3
  serviceName: mysql-cluster
  selector:
    matchLabels:
      app: mysql-cluster
  
  template:
    metadata:
      labels:
        app: mysql-cluster
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        
        volumeMounts:
        - name: data       # MySQL data directory
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: logs       # MySQL logs
          mountPath: /var/log/mysql
        - name: conf       # MySQL config
          mountPath: /etc/mysql/conf.d
        
        ports:
        - containerPort: 3306
          name: mysql
        
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-u", "root", "-p$(MYSQL_ROOT_PASSWORD)"]
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          exec:
            command: ["mysql", "-u", "root", "-p$(MYSQL_ROOT_PASSWORD)", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 5
      
      - name: metrics
        image: prom/mysqld-exporter:v0.15.0
        ports:
        - containerPort: 9104
          name: metrics
        env:
        - name: DATA_SOURCE_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: exporter-dsn
  
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: mysql-cluster
        type: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
  
  - metadata:
      name: logs
      labels:
        app: mysql-cluster
        type: logs
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: standard
      resources:
        requests:
          storage: 20Gi
```

### 2.2 StatefulSet Headless Service (สำคัญมาก)

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-cluster
  namespace: databases
  labels:
    app: mysql-cluster
spec:
  selector:
    app: mysql-cluster
  ports:
  - port: 3306
    name: mysql
  clusterIP: None    # Headless service = ไม่มี ClusterIP
```

**Headless Service สร้าง DNS records:**
- `mysql-cluster-0.mysql-cluster.databases.svc.cluster.local`
- `mysql-cluster-1.mysql-cluster.databases.svc.cluster.local`
- `mysql-cluster-2.mysql-cluster.databases.svc.cluster.local`

---

## 3. Database Clustering Patterns

### 3.1 Master-Slave Replication (MySQL)

```yaml
# mysql-master-slave.yaml
---
# ConfigMap สำหรับ Master config
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: databases
data:
  master.cnf: |
    [mysqld]
    log-bin=mysql-bin
    server-id=1
    binlog-format=ROW
    sync-binlog=1
    innodb-flush-log-at-trx-commit=1
    
  slave.cnf: |
    [mysqld]
    super-read-only
    
  primary-init.sql: |
    CREATE USER IF NOT EXISTS 'replicator'@'%' IDENTIFIED BY 'replicator-password';
    GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
    FLUSH PRIVILEGES;
    
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: databases
type: Opaque
stringData:
  root-password: "RootPass2024!"
  replication-password: "ReplPass2024!"

---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: databases
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql

---
# Read Service (สำหรับ read from slaves)
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  namespace: databases
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql

---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: databases
spec:
  replicas: 3
  serviceName: mysql
  selector:
    matchLabels:
      app: mysql
  
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      # Init container 1: กำหนด server-id และ config
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # สร้าง server-id จาก ordinal index
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          
          # Copy master or slave config
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
      # Init container 2: Clone data from master
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip if data already exists
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          
          # Skip if ordinal == 0 (master)
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          
          # Clone data from previous peer
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          xtrabackup --prepare --target-dir=/var/lib/mysql
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      
      containers:
      - name: mysql
        image: mysql:8.0
        
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        
        ports:
        - containerPort: 3306
          name: mysql
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
        
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      
      # Sidecar: xtrabackup สำหรับ streaming data
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          
          # Backup stream server
          if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
            cp xtrabackup_slave_info change_master_to.sql.in
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready..."
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            
            echo "Initializing replication from clone position"
            mysql -h 127.0.0.1 \
                  -e "$(<change_master_to.sql.in), \
                          MASTER_HOST='mysql-0.mysql', \
                          MASTER_USER='root', \
                          MASTER_PASSWORD='$MYSQL_ROOT_PASSWORD', \
                          MASTER_CONNECT_RETRY=10; \
                        START SLAVE;" || exit 1
            mv change_master_to.sql.in change_master_to.sql.orig
          fi
          
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=$MYSQL_ROOT_PASSWORD"
        
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

---

## 4. Workshop: Deploy PostgreSQL HA Cluster

### Workshop Overview
สร้าง PostgreSQL High Availability cluster ด้วย Patroni บน Kubernetes

### Prerequisites
- Kubernetes cluster ที่มี dynamic provisioning
- Helm installed
- เวลาประมาณ 90-120 นาที

### Step 1: เตรียม Namespace

```bash
kubectl create namespace postgres-ha
kubectl config set-context --current --namespace=postgres-ha
```

### Step 2: ติดตั้ง CloudNativePG Operator (แนะนำ)

CloudNativePG เป็น Kubernetes operator สำหรับ PostgreSQL HA

```bash
# ติดตั้ง Operator
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml

# รอ Operator start
kubectl wait --for=condition=available deployment/cnpg-controller-manager \
  -n cnpg-system --timeout=120s

# ดู Operator
kubectl get pods -n cnpg-system
```

### Step 3: สร้าง PostgreSQL Cluster ด้วย CloudNativePG

```yaml
# postgres-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: postgres-ha
spec:
  instances: 3    # 1 Primary + 2 Standby
  
  imageName: ghcr.io/cloudnative-pg/postgresql:15.4
  
  # Primary update strategy
  primaryUpdateStrategy: unsupervised
  
  # PostgreSQL configuration
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "768MB"
      maintenance_work_mem: "64MB"
      checkpoint_completion_target: "0.9"
      wal_buffers: "16MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
      work_mem: "3276kB"
      min_wal_size: "1GB"
      max_wal_size: "4GB"
      max_worker_processes: "4"
      max_parallel_workers_per_gather: "2"
      max_parallel_workers: "4"
      max_parallel_maintenance_workers: "2"
    pg_hba:
    - "host all all 0.0.0.0/0 md5"
    - "host replication streaming_replica 0.0.0.0/0 md5"
  
  # Bootstrap configuration
  bootstrap:
    initdb:
      database: appdb
      owner: appuser
      secret:
        name: postgres-user-secret
  
  # Storage configuration
  storage:
    storageClass: local-path    # เปลี่ยนตาม cluster ของคุณ
    size: 10Gi
  
  # WAL storage (แยก volume สำหรับ performance)
  walStorage:
    storageClass: local-path
    size: 2Gi
  
  # Resources
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  
  # Monitoring
  monitoring:
    enablePodMonitor: false   # เปิดถ้ามี Prometheus
  
  # Backup configuration
  backup:
    retentionPolicy: 7d
    barmanObjectStore:
      destinationPath: "s3://my-cluster-backups/"  # เปลี่ยนตาม setup ของคุณ
      s3Credentials:
        accessKeyId:
          name: backup-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-creds
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
```

```yaml
# postgres-user-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-user-secret
  namespace: postgres-ha
type: kubernetes.io/basic-auth
stringData:
  username: appuser
  password: "AppUser2024!"
```

```bash
kubectl apply -f postgres-user-secret.yaml
kubectl apply -f postgres-cluster.yaml

# รอ Cluster start
kubectl get cluster postgres-cluster -w
```

### Step 4: ดู Cluster Status

```bash
# ดู Cluster
kubectl get cluster -n postgres-ha
kubectl describe cluster postgres-cluster -n postgres-ha

# ดู Pods
kubectl get pods -n postgres-ha -l cnpg.io/cluster=postgres-cluster

# ดู PVCs
kubectl get pvc -n postgres-ha

# ดู Services
kubectl get svc -n postgres-ha
```

### Step 5: เชื่อมต่อและทดสอบ

```bash
# ดู connection credentials
kubectl get secret postgres-cluster-app -n postgres-ha -o jsonpath='{.data.uri}' | base64 -d

# Port forward ไป Primary
kubectl port-forward svc/postgres-cluster-rw 5432:5432 -n postgres-ha &

# เชื่อมต่อ
PGPASSWORD="AppUser2024!" psql -h localhost -U appuser -d appdb

# สร้าง Table ทดสอบ
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer VARCHAR(100) NOT NULL,
    product VARCHAR(100) NOT NULL,
    quantity INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO orders (customer, product, quantity) VALUES
    ('Alice', 'Product A', 5),
    ('Bob', 'Product B', 3),
    ('Charlie', 'Product C', 10);

SELECT * FROM orders;
\q
```

### Step 6: ทดสอบ Failover

```bash
# ดู Primary Pod
kubectl get pods -n postgres-ha -l cnpg.io/instanceRole=primary

# ลบ Primary Pod เพื่อทดสอบ Failover
PRIMARY_POD=$(kubectl get pods -n postgres-ha -l cnpg.io/instanceRole=primary -o jsonpath='{.items[0].metadata.name}')
echo "Deleting primary: $PRIMARY_POD"
kubectl delete pod $PRIMARY_POD -n postgres-ha

# ดู Failover process
kubectl get pods -n postgres-ha -w

# รอ Failover เสร็จ
sleep 30

# ดู Primary ใหม่
kubectl get pods -n postgres-ha -l cnpg.io/instanceRole=primary

# ตรวจสอบว่า data ยังอยู่
kubectl port-forward svc/postgres-cluster-rw 5432:5432 -n postgres-ha &
sleep 3
PGPASSWORD="AppUser2024!" psql -h localhost -U appuser -d appdb -c "SELECT * FROM orders;"
```

### Step 7: ทดสอบ Read Scalability

```bash
# Read-only connections ไป Replica
kubectl port-forward svc/postgres-cluster-ro 5433:5432 -n postgres-ha &
sleep 3

# Read จาก Replica
PGPASSWORD="AppUser2024!" psql -h localhost -p 5433 -U appuser -d appdb -c "SELECT * FROM orders;"

# ลองเขียนบน Replica (ควร fail)
PGPASSWORD="AppUser2024!" psql -h localhost -p 5433 -U appuser -d appdb -c \
  "INSERT INTO orders (customer, product, quantity) VALUES ('Test', 'X', 1);"
# Expected: ERROR: cannot execute INSERT in a read-only transaction
```

### Step 8: ทำ Manual Switchover

```bash
# Switchover (graceful)
kubectl cnpg promote postgres-cluster-2 -n postgres-ha
# หรือ
kubectl patch cluster postgres-cluster -n postgres-ha \
  -p '{"spec":{"primaryUpdateStrategy":"supervised"}}' \
  --type=merge

# ดู status
kubectl get cluster postgres-cluster -n postgres-ha -o wide
```

### Step 9: ใช้ Manual Approach (ไม่ใช้ Operator)

ถ้าไม่ต้องการใช้ Operator สามารถใช้ StatefulSet ธรรมดาได้:

```yaml
# postgres-statefulset-manual.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: postgres-ha
data:
  POSTGRES_DB: appdb
  POSTGRES_USER: appuser
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            trust
    host    all             all             ::1/128                 trust
    host    all             all             0.0.0.0/0               md5
    host    replication     replicator      0.0.0.0/0               md5
  
  postgresql.conf: |
    listen_addresses = '*'
    wal_level = replica
    hot_standby = on
    max_wal_senders = 5
    wal_keep_size = 64MB
    synchronous_commit = on

---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: postgres-ha
type: Opaque
stringData:
  POSTGRES_PASSWORD: "AppPass2024!"
  REPLICATION_PASSWORD: "ReplPass2024!"

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres-ha
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    name: postgres

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: postgres-ha
spec:
  selector:
    app: postgres
    role: primary
  ports:
  - port: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres-ha
spec:
  replicas: 3
  serviceName: postgres
  selector:
    matchLabels:
      app: postgres
  
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
      - name: init-postgres
        image: postgres:15-alpine
        command:
        - /bin/sh
        - -c
        - |
          # กำหนด replica mode ตาม ordinal
          ORDINAL=$(echo $HOSTNAME | grep -oE '[0-9]+$')
          if [ "$ORDINAL" = "0" ]; then
            echo "PRIMARY" > /etc/postgres-role/role
          else
            echo "REPLICA" > /etc/postgres-role/role
          fi
          echo "Role: $(cat /etc/postgres-role/role)"
        volumeMounts:
        - name: role
          mountPath: /etc/postgres-role
      
      containers:
      - name: postgres
        image: postgres:15-alpine
        
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_PASSWORD
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        command:
        - /bin/sh
        - -c
        - |
          ROLE=$(cat /etc/postgres-role/role)
          ORDINAL=$(echo $HOSTNAME | grep -oE '[0-9]+$')
          
          if [ "$ROLE" = "PRIMARY" ]; then
            echo "Starting as PRIMARY"
            # Init database if needed
            if [ ! -d /var/lib/postgresql/data/pgdata ]; then
              initdb -D /var/lib/postgresql/data/pgdata
            fi
            postgres -D /var/lib/postgresql/data/pgdata \
              -c config_file=/etc/postgresql/postgresql.conf \
              -c hba_file=/etc/postgresql/pg_hba.conf
          else
            echo "Starting as REPLICA"
            # Wait for primary
            until pg_isready -h postgres-0.postgres -p 5432; do
              echo "Waiting for primary..."
              sleep 2
            done
            # Setup replica
            if [ ! -d /var/lib/postgresql/data/pgdata ]; then
              PGPASSWORD=$POSTGRES_PASSWORD pg_basebackup \
                -h postgres-0.postgres \
                -U $POSTGRES_USER \
                -D /var/lib/postgresql/data/pgdata \
                -P -Xs -R
            fi
            postgres -D /var/lib/postgresql/data/pgdata
          fi
        
        ports:
        - containerPort: 5432
          name: postgres
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: role
          mountPath: /etc/postgres-role
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        - name: postgres-config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser"]
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser"]
          initialDelaySeconds: 5
          periodSeconds: 5
      
      volumes:
      - name: role
        emptyDir: {}
      - name: postgres-config
        configMap:
          name: postgres-config
  
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: postgres
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: local-path
      resources:
        requests:
          storage: 5Gi
```

```bash
kubectl apply -f postgres-statefulset-manual.yaml
kubectl get pods -n postgres-ha -w
```

### Step 10: ตรวจสอบ PVCs สำหรับแต่ละ Pod

```bash
# ดู PVCs ทั้งหมด
kubectl get pvc -n postgres-ha

# Output:
# NAME                 STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-postgres-0      Bound    ...      5Gi        RWO            local-path     2m
# data-postgres-1      Bound    ...      5Gi        RWO            local-path     1m
# data-postgres-2      Bound    ...      5Gi        RWO            local-path     30s

# ดู PV ที่ถูกสร้าง
kubectl get pv | grep postgres

# ดูรายละเอียด PVC ของแต่ละ Pod
kubectl describe pvc data-postgres-0 -n postgres-ha
```

### Step 11: Scale StatefulSet

```bash
# Scale up เพิ่ม Replica
kubectl scale statefulset postgres -n postgres-ha --replicas=5

# ดู Pods ถูกสร้างทีละอัน (sequential)
kubectl get pods -n postgres-ha -w

# ดู PVCs ใหม่ถูกสร้าง
kubectl get pvc -n postgres-ha

# Scale down
kubectl scale statefulset postgres -n postgres-ha --replicas=3

# ดู Pods ถูกลบ reverse order
kubectl get pods -n postgres-ha -w

# หมายเหตุ: PVCs ไม่ถูกลบเมื่อ scale down!
kubectl get pvc -n postgres-ha    # ยังมี 5 PVCs
```

### Step 12: Rolling Update

```bash
# Update image
kubectl set image statefulset/postgres postgres=postgres:15.4-alpine -n postgres-ha

# ดู Rolling Update (ทีละ Pod)
kubectl rollout status statefulset/postgres -n postgres-ha

# หรือดู Pod ทีละอัน
kubectl get pods -n postgres-ha -w
```

### Step 13: ทำความสะอาด

```bash
# ลบ StatefulSet (แต่ PVCs ยังอยู่)
kubectl delete statefulset postgres -n postgres-ha

# ดู PVCs ยังอยู่
kubectl get pvc -n postgres-ha

# ลบ PVCs ด้วยตนเอง
kubectl delete pvc -l app=postgres -n postgres-ha

# ลบ namespace ทั้งหมด
kubectl delete namespace postgres-ha
```

---

## 5. StatefulSet Update Strategies

### RollingUpdate (default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2    # Update เฉพาะ pods ที่มี ordinal >= 2
```

**Canary Deployment ด้วย Partition:**
```bash
# Update image แบบ Canary
kubectl patch statefulset postgres -n postgres-ha \
  -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'

# Deploy new image
kubectl set image statefulset/postgres postgres=postgres:16 -n postgres-ha

# เฉพาะ Pod 2 จะถูก update ก่อน
# หลัง verify แล้ว ค่อยลด partition
kubectl patch statefulset postgres -n postgres-ha \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'
# แล้วลดไปเรื่อยๆ จนถึง 0
```

### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete    # Update เฉพาะเมื่อ Pod ถูกลบด้วยตนเอง
```

---

## 6. StatefulSet Ordering

### Pod Management Policy

```yaml
# Parallel: สร้าง/ลบทุก Pod พร้อมกัน (เหมาะสำหรับ independent pods)
spec:
  podManagementPolicy: Parallel
  
# OrderedReady (default): สร้าง/ลบตามลำดับ
spec:
  podManagementPolicy: OrderedReady
```

---

## 7. Best Practices สำหรับ Stateful Applications

### 7.1 ใช้ Operators สำหรับ Complex Stateful Apps

| Application | Operator |
|-------------|----------|
| PostgreSQL | CloudNativePG, Zalando Postgres Operator |
| MySQL | Percona XtraDB Operator, Oracle MySQL Operator |
| MongoDB | MongoDB Community Operator |
| Cassandra | K8ssandra Operator |
| Redis | Redis Enterprise Operator |
| Elasticsearch | ECK (Elastic Cloud on Kubernetes) |
| Kafka | Strimzi |

### 7.2 Storage Considerations

```yaml
# ใช้ storageClass ที่เหมาะกับ workload
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    storageClassName: fast-ssd   # Database ต้องการ fast storage
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 100Gi

# แยก WAL/logs ออกจาก data สำหรับ performance
- metadata:
    name: wal
  spec:
    storageClassName: fast-ssd
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 20Gi
```

### 7.3 Health Checks

```yaml
livenessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
  initialDelaySeconds: 30    # ให้เวลา DB start
  periodSeconds: 10
  failureThreshold: 5        # ให้ fail ได้หลายครั้งก่อน restart

readinessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

---

## สรุป

| Concept | คำอธิบาย |
|---------|----------|
| VolumeClaimTemplates | สร้าง PVC แยกสำหรับแต่ละ Pod อัตโนมัติ |
| Headless Service | ทำให้แต่ละ Pod มี DNS เฉพาะ |
| Stable Identity | Pod names: app-0, app-1, app-2 |
| Ordered Operations | Create/Delete เป็นลำดับ |
| Partition Update | Canary deployment strategy |
| PVC Retention | PVCs ไม่ถูกลบเมื่อ scale down |

### Key Takeaways:
1. **VolumeClaimTemplates** ทำให้แต่ละ Pod มี PVC เป็นของตัวเองโดยอัตโนมัติ
2. **Headless Service** จำเป็นสำหรับ stable DNS ของแต่ละ Pod
3. **PVCs ไม่ถูกลบ** เมื่อ StatefulSet scale down - ต้องลบเอง
4. **Operators** เป็นวิธีที่ดีที่สุดสำหรับ production databases บน Kubernetes
5. **subPath** สำคัญสำหรับ databases เช่น PostgreSQL, MySQL
6. ใช้ **Recreate strategy** ไม่ใช่ RollingUpdate สำหรับ single-instance databases

## แหล่งเรียนรู้เพิ่มเติม

- [StatefulSets Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [CloudNativePG](https://cloudnative-pg.io/)
- [Zalando Postgres Operator](https://github.com/zalando/postgres-operator)
- [Running Stateful Applications](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
