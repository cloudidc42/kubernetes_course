# Part 21: StatefulSets - การจัดการ Stateful Applications ใน Kubernetes

## สารบัญ
1. [StatefulSet คืออะไร](#statefulset-คืออะไร)
2. [ความแตกต่างระหว่าง StatefulSet และ Deployment](#ความแตกต่างระหว่าง-statefulset-และ-deployment)
3. [Stable Network Identity](#stable-network-identity)
4. [Ordered Deployment และ Termination](#ordered-deployment-และ-termination)
5. [Persistent Storage ใน StatefulSet](#persistent-storage-ใน-statefulset)
6. [StatefulSet YAML ละเอียด](#statefulset-yaml-ละเอียด)
7. [Headless Service](#headless-service)
8. [Update Strategies](#update-strategies)
9. [Workshop: Deploy PostgreSQL Cluster](#workshop-deploy-postgresql-cluster)
10. [Workshop: Deploy MySQL Cluster](#workshop-deploy-mysql-cluster)
11. [Troubleshooting StatefulSets](#troubleshooting-statefulsets)
12. [Best Practices](#best-practices)

---

## StatefulSet คืออะไร

StatefulSet เป็น Kubernetes workload resource ที่ออกแบบมาเฉพาะสำหรับ **stateful applications** - แอปพลิเคชันที่ต้องการ:

- **Stable Network Identity**: Pod แต่ละตัวมีชื่อและ hostname ที่แน่นอน ไม่เปลี่ยนแปลง
- **Stable Storage**: เชื่อมต่อกับ PersistentVolume ที่เฉพาะเจาะจง
- **Ordered Deployment**: Pod ถูกสร้างและลบตามลำดับที่แน่นอน
- **Ordered Scaling**: Scale up/down เป็นลำดับ ไม่ใช่พร้อมกันทั้งหมด

### ตัวอย่าง Use Cases ของ StatefulSet

```
┌─────────────────────────────────────────────────────────┐
│                    StatefulSet Use Cases                │
├─────────────────────────────────────────────────────────┤
│  Databases:                                             │
│  - MySQL, PostgreSQL, MongoDB, Cassandra                │
│  - Redis Cluster, Elasticsearch                         │
│                                                         │
│  Message Queues:                                        │
│  - Apache Kafka, RabbitMQ                               │
│  - Apache Zookeeper                                     │
│                                                         │
│  Distributed Systems:                                   │
│  - etcd Cluster                                         │
│  - Consul                                               │
│                                                         │
│  Any app that requires:                                 │
│  - Persistent state per instance                        │
│  - Stable hostname/identity                             │
│  - Ordered operations                                   │
└─────────────────────────────────────────────────────────┘
```

---

## ความแตกต่างระหว่าง StatefulSet และ Deployment

| คุณสมบัติ | Deployment | StatefulSet |
|-----------|-----------|-------------|
| Pod Names | สุ่ม (e.g., `app-7d8f9-xk2p`) | มีลำดับ (e.g., `app-0`, `app-1`) |
| Storage | Shared หรือ Ephemeral | ต่อ Pod (PVC ของแต่ละ Pod) |
| Network Identity | เปลี่ยนได้ | คงที่, Stable |
| Scaling | พร้อมกันทั้งหมด | เป็นลำดับทีละ Pod |
| Deletion | ลบพร้อมกัน | ลบย้อนกลับตามลำดับ |
| Use Case | Stateless Apps | Stateful Apps |
| DNS | ไม่มี stable DNS per Pod | มี stable DNS per Pod |
| Rolling Update | ทั้งหมดพร้อมกัน | ทีละ Pod ตามลำดับ |

### ตัวอย่างภาพ: Deployment vs StatefulSet

```
Deployment (Stateless):
─────────────────────
  ┌──────────┐
  │  Service │ ──→ Load Balances ──→ [ Pod-abc ] [ Pod-xyz ] [ Pod-def ]
  └──────────┘                       (ชื่อสุ่ม, แต่ละตัวเหมือนกัน)


StatefulSet (Stateful):
──────────────────────
  ┌──────────────┐
  │ Headless Svc │
  └──────────────┘
        │
        ├──→ mysql-0 ←──→ PVC-0 (data ของ mysql-0)
        ├──→ mysql-1 ←──→ PVC-1 (data ของ mysql-1)
        └──→ mysql-2 ←──→ PVC-2 (data ของ mysql-2)
        
  mysql-0.mysql.default.svc.cluster.local  (DNS คงที่)
  mysql-1.mysql.default.svc.cluster.local  (DNS คงที่)
  mysql-2.mysql.default.svc.cluster.local  (DNS คงที่)
```

---

## Stable Network Identity

### DNS Pattern ของ StatefulSet

เมื่อสร้าง StatefulSet ชื่อ `mysql` ใน namespace `default`, แต่ละ Pod จะมี DNS ดังนี้:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

ตัวอย่าง:
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

### ทดสอบ DNS Resolution

```bash
# สร้าง debug pod เพื่อทดสอบ DNS
kubectl run -it --rm debug --image=busybox:1.28 --restart=Never -- sh

# ภายใน pod ทดสอบ DNS
nslookup mysql-0.mysql.default.svc.cluster.local
nslookup mysql-1.mysql.default.svc.cluster.local

# ทดสอบ TCP connection
nc -zv mysql-0.mysql.default.svc.cluster.local 3306
```

---

## Ordered Deployment และ Termination

### Ordered Deployment (Scale Up)

StatefulSet สร้าง Pod ตามลำดับ 0, 1, 2, ... และรอให้แต่ละ Pod Ready ก่อนสร้างตัวถัดไป:

```
สร้าง StatefulSet replicas=3:

Step 1: สร้าง pod-0 → รอจนกว่า pod-0 Ready
Step 2: สร้าง pod-1 → รอจนกว่า pod-1 Ready  
Step 3: สร้าง pod-2 → รอจนกว่า pod-2 Ready

ถ้า pod-1 ไม่ Ready → pod-2 จะไม่ถูกสร้าง!
```

### Ordered Termination (Scale Down)

ลบ Pod ในลำดับย้อนกลับ N-1, N-2, ..., 0:

```
ลด StatefulSet จาก 3 → 1 replicas:

Step 1: ลบ pod-2 → รอจนกว่า pod-2 Terminated
Step 2: ลบ pod-1 → รอจนกว่า pod-1 Terminated

เหลือ pod-0 ทำงานต่อ
```

### ดูการทำงานแบบ Ordered

```bash
# ดู events ขณะ scale up
kubectl get pods -w -l app=mysql

# ดู events ขณะ scale down
kubectl scale statefulset mysql --replicas=1
kubectl get pods -w -l app=mysql
```

---

## Persistent Storage ใน StatefulSet

### VolumeClaimTemplates

StatefulSet ใช้ `volumeClaimTemplates` เพื่อสร้าง PVC สำหรับแต่ละ Pod โดยอัตโนมัติ:

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: "standard"
    resources:
      requests:
        storage: 10Gi
```

PVC ที่ถูกสร้างจะมีชื่อ: `<template-name>-<statefulset-name>-<ordinal>`

```
data-mysql-0
data-mysql-1
data-mysql-2
```

### สำคัญ: PVC ไม่ถูกลบอัตโนมัติ

เมื่อ Pod ถูกลบหรือ StatefulSet ถูก Scale Down, **PVC จะยังคงอยู่**:

```bash
# ดู PVC ที่ยังอยู่หลัง scale down
kubectl get pvc

# ต้องลบ PVC เองถ้าต้องการ
kubectl delete pvc data-mysql-0
kubectl delete pvc data-mysql-1
kubectl delete pvc data-mysql-2
```

---

## StatefulSet YAML ละเอียด

### Basic StatefulSet

```yaml
# basic-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: default
  labels:
    app: web
spec:
  # ชื่อ Headless Service ที่ใช้สำหรับ Network Identity
  serviceName: "web"
  
  # จำนวน replicas
  replicas: 3
  
  # Selector ต้องตรงกับ template labels
  selector:
    matchLabels:
      app: web
  
  # Pod template
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        
        # Resource limits
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        
        # Volume mount
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
        
        # Readiness probe (สำคัญมากสำหรับ ordered deployment)
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
  
  # สร้าง PVC ให้แต่ละ Pod อัตโนมัติ
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi
  
  # Update strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  
  # Pod Management Policy
  podManagementPolicy: OrderedReady  # หรือ Parallel
```

---

## Headless Service

StatefulSet ต้องการ Headless Service เพื่อ manage network identity:

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
  labels:
    app: mysql
spec:
  # clusterIP: None = Headless Service
  clusterIP: None
  
  # selector ต้องตรงกับ StatefulSet pods
  selector:
    app: mysql
  
  ports:
  - port: 3306
    name: mysql
    targetPort: 3306
```

### ความแตกต่างของ Headless Service

```
Normal Service (ClusterIP):
  DNS: mysql.default.svc.cluster.local → 10.96.0.100 (Virtual IP)
  ── Load balances ──→ [ pod-0 ] [ pod-1 ] [ pod-2 ]

Headless Service (clusterIP: None):
  DNS: mysql.default.svc.cluster.local → [ 10.0.0.1, 10.0.0.2, 10.0.0.3 ] (Pod IPs)
  DNS: mysql-0.mysql → 10.0.0.1 (direct to pod-0)
  DNS: mysql-1.mysql → 10.0.0.2 (direct to pod-1)
  DNS: mysql-2.mysql → 10.0.0.3 (direct to pod-2)
```

---

## Update Strategies

### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # partition: อัปเดต Pod ที่มี ordinal >= partition เท่านั้น
      # ใช้สำหรับ Canary Updates
      partition: 0
```

### Canary Update ด้วย Partition

```bash
# มี StatefulSet 3 replicas (pod-0, pod-1, pod-2)
# ต้องการทดสอบ version ใหม่กับ pod-2 ก่อน

# Set partition = 2 → อัปเดตเฉพาะ pod-2
kubectl patch statefulset mysql -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# อัปเดต image
kubectl set image statefulset/mysql mysql=mysql:8.0.30

# ตรวจสอบ → pod-2 จะได้ version ใหม่, pod-0, pod-1 ยังเดิม
kubectl get pods -l app=mysql -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'

# ถ้าโอเค set partition = 0 เพื่ออัปเดตทั้งหมด
kubectl patch statefulset mysql -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

### OnDelete Strategy

```yaml
spec:
  updateStrategy:
    type: OnDelete
    # อัปเดตเฉพาะเมื่อ admin ลบ Pod เอง
```

---

## Workshop: Deploy PostgreSQL Cluster

เป้าหมาย: Deploy PostgreSQL primary-replica cluster ด้วย StatefulSet

### 1. สร้าง Namespace

```bash
kubectl create namespace postgres-cluster
```

### 2. สร้าง ConfigMap สำหรับ PostgreSQL Config

```yaml
# postgres-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: postgres-cluster
data:
  # Configuration สำหรับ Primary
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 200
    shared_buffers = 256MB
    effective_cache_size = 768MB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.7
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 1310kB
    min_wal_size = 1GB
    max_wal_size = 4GB
    max_worker_processes = 8
    max_parallel_workers_per_gather = 4
    max_parallel_workers = 8
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 1GB
    hot_standby = on
    
  # pg_hba.conf สำหรับ replication
  pg_hba.conf: |
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            trust
    host    all             all             ::1/128                 trust
    host    all             all             0.0.0.0/0               md5
    host    replication     replicator      0.0.0.0/0               md5
  
  # Script สำหรับ setup replication
  setup-replication.sh: |
    #!/bin/bash
    set -e
    
    # สร้าง replication user
    psql -U postgres -c "CREATE USER replicator REPLICATION LOGIN CONNECTION LIMIT -1 ENCRYPTED PASSWORD 'replicapass';"
    
    echo "Replication user created"
```

### 3. สร้าง Secret สำหรับ Password

```bash
# สร้าง secret สำหรับ postgres password
kubectl create secret generic postgres-secret \
  --from-literal=postgres-password=mysecretpassword \
  --from-literal=replication-password=replicapass \
  -n postgres-cluster
```

### 4. สร้าง Headless Service

```yaml
# postgres-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres-cluster
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - name: postgresql
    port: 5432
    targetPort: 5432
```

### 5. สร้าง Service สำหรับ Primary (Read-Write)

```yaml
# postgres-primary-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: postgres-cluster
  labels:
    app: postgres
    role: primary
spec:
  type: ClusterIP
  selector:
    app: postgres
    # เลือกเฉพาะ primary pod (pod-0)
    statefulset.kubernetes.io/pod-name: postgres-0
  ports:
  - name: postgresql
    port: 5432
    targetPort: 5432
```

### 6. สร้าง Service สำหรับ Read Replicas

```yaml
# postgres-replica-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
  namespace: postgres-cluster
  labels:
    app: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
    role: replica
  ports:
  - name: postgresql
    port: 5432
    targetPort: 5432
```

### 7. สร้าง StatefulSet สำหรับ PostgreSQL

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres-cluster
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Init Container สำหรับ setup primary/replica
      initContainers:
      - name: init-postgres
        image: postgres:14
        command:
        - bash
        - "-c"
        - |
          set -ex
          
          # ตรวจสอบว่าเป็น primary (ordinal = 0) หรือ replica
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          
          if [[ $ordinal -eq 0 ]]; then
            # Primary: ไม่ต้องทำอะไร initContainer จัดการ data directory
            echo "This is primary (postgres-0)"
            echo "primary" > /mnt/conf.d/role
          else
            # Replica: clone data จาก primary
            echo "This is replica (postgres-$ordinal)"
            echo "replica" > /mnt/conf.d/role
            
            # รอให้ primary พร้อม
            until pg_isready -h postgres-0.postgres.postgres-cluster.svc.cluster.local -U postgres; do
              echo "Waiting for primary..."
              sleep 2
            done
            
            # Clone data จาก primary ถ้า data directory ว่าง
            if [ -z "$(ls -A /var/lib/postgresql/data/pgdata)" ]; then
              PGPASSWORD=$REPLICATION_PASSWORD pg_basebackup \
                -h postgres-0.postgres.postgres-cluster.svc.cluster.local \
                -U replicator \
                -D /var/lib/postgresql/data/pgdata \
                -P -Xs -R
            fi
          fi
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: replication-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: conf
          mountPath: /mnt/conf.d
      
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        
        ports:
        - name: postgresql
          containerPort: 5432
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        - name: config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        
        # Readiness probe
        readinessProbe:
          exec:
            command:
            - bash
            - "-c"
            - "psql -U postgres -c 'SELECT 1'"
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Liveness probe
        livenessProbe:
          exec:
            command:
            - bash
            - "-c"
            - "psql -U postgres -c 'SELECT 1'"
          initialDelaySeconds: 45
          periodSeconds: 20
          timeoutSeconds: 5
          failureThreshold: 3
      
      volumes:
      - name: config
        configMap:
          name: postgres-config
      - name: conf
        emptyDir: {}
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### 8. Apply ทั้งหมด

```bash
# Apply ไฟล์ทั้งหมด
kubectl apply -f postgres-config.yaml
kubectl apply -f postgres-headless-svc.yaml
kubectl apply -f postgres-primary-svc.yaml
kubectl apply -f postgres-replica-svc.yaml
kubectl apply -f postgres-statefulset.yaml

# ดู progress
kubectl get pods -n postgres-cluster -w

# ดู StatefulSet status
kubectl get statefulset -n postgres-cluster

# ดู PVC ที่ถูกสร้าง
kubectl get pvc -n postgres-cluster
```

### 9. ทดสอบ PostgreSQL Cluster

```bash
# เชื่อมต่อกับ Primary
kubectl run -it --rm psql-client \
  --image=postgres:14 \
  --restart=Never \
  -n postgres-cluster \
  -- psql -h postgres-primary.postgres-cluster.svc.cluster.local \
          -U postgres \
          -c "SELECT version();"

# สร้าง database ทดสอบ
kubectl run -it --rm psql-client \
  --image=postgres:14 \
  --restart=Never \
  -n postgres-cluster \
  -- psql -h postgres-primary.postgres-cluster.svc.cluster.local \
          -U postgres \
          -c "CREATE DATABASE testdb;"

# สร้าง table และ insert data
kubectl exec -it postgres-0 -n postgres-cluster -- psql -U postgres -d testdb -c "
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO users (name, email) VALUES 
  ('Alice', 'alice@example.com'),
  ('Bob', 'bob@example.com'),
  ('Charlie', 'charlie@example.com');
"

# ตรวจสอบ replication บน replica
kubectl exec -it postgres-1 -n postgres-cluster -- psql -U postgres -d testdb -c "
SELECT * FROM users;
"

# ดู replication status
kubectl exec -it postgres-0 -n postgres-cluster -- psql -U postgres -c "
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn 
FROM pg_stat_replication;
"
```

---

## Workshop: Deploy MySQL Cluster

### 1. สร้าง ConfigMap

```yaml
# mysql-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql-cluster
data:
  # Primary config
  primary.cnf: |
    [mysqld]
    log-bin
    server-id = 1
    log_replica_updates = 1
    binlog_format = row
    sync-binlog = 1
    
  # Replica config template
  replica.cnf: |
    [mysqld]
    super-read-only
    server-id = auto
```

### 2. สร้าง StatefulSet สำหรับ MySQL

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql-cluster
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # สร้าง server-id จาก hostname
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # ordinal 0 คือ primary, ที่เหลือคือ replica
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # copy config ที่เหมาะสม
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # ข้าม clone ถ้ามี data อยู่แล้ว
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # ข้าม clone ถ้าเป็น primary (ordinal 0)
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone จาก pod ก่อนหน้า
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare backup
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
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
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
            cpu: "1"
            memory: 2Gi
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
      
      # Sidecar: xtrabackup สำหรับ clone data ไปยัง replica ใหม่
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
          
          # ตรวจสอบตำแหน่งที่ clone มา
          if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
            cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
            rm -f xtrabackup_slave_info xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm -f xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          
          # Setup replication ถ้าเป็น replica
          if [[ -f change_master_to.sql.in ]]; then
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.in),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
            mv change_master_to.sql.in change_master_to.sql.orig
          fi
          
          # รอ request สำหรับ backup
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
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
          limits:
            cpu: 500m
            memory: 500Mi
      
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
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 3. ทดสอบ MySQL Cluster

```bash
# สร้าง namespace
kubectl create namespace mysql-cluster

# Apply config และ statefulset
kubectl apply -f mysql-config.yaml -n mysql-cluster
kubectl apply -f mysql-statefulset.yaml -n mysql-cluster

# ดู pod status
kubectl get pods -n mysql-cluster -w

# เชื่อมต่อกับ Primary
kubectl run -it --rm mysql-client \
  --image=mysql:8.0 \
  --restart=Never \
  -n mysql-cluster \
  -- mysql -h mysql-0.mysql.mysql-cluster.svc.cluster.local -uroot

# สร้าง database และ data
mysql> CREATE DATABASE testdb;
mysql> USE testdb;
mysql> CREATE TABLE products (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100), price DECIMAL(10,2));
mysql> INSERT INTO products VALUES (NULL, 'Product A', 99.99), (NULL, 'Product B', 149.99);

# ตรวจสอบ replication บน replica
kubectl run -it --rm mysql-client \
  --image=mysql:8.0 \
  --restart=Never \
  -n mysql-cluster \
  -- mysql -h mysql-1.mysql.mysql-cluster.svc.cluster.local -uroot -e "SELECT * FROM testdb.products;"

# ดู replication status
kubectl exec -it mysql-0 -n mysql-cluster -c mysql -- mysql -uroot -e "SHOW MASTER STATUS\G"
kubectl exec -it mysql-1 -n mysql-cluster -c mysql -- mysql -uroot -e "SHOW SLAVE STATUS\G"
```

---

## Troubleshooting StatefulSets

### ปัญหา: Pod ค้างอยู่ใน Pending

```bash
# ดู events ของ pod
kubectl describe pod postgres-0 -n postgres-cluster

# ตรวจสอบ PVC status
kubectl get pvc -n postgres-cluster
kubectl describe pvc data-postgres-0 -n postgres-cluster

# ตรวจสอบ StorageClass
kubectl get storageclass
kubectl describe storageclass standard

# ปัญหาที่พบบ่อย: ไม่มี PersistentVolume ว่าง
kubectl get pv
```

### ปัญหา: StatefulSet ไม่อัปเดต

```bash
# ดู update status
kubectl rollout status statefulset/postgres -n postgres-cluster

# ดู history
kubectl rollout history statefulset/postgres -n postgres-cluster

# Rollback
kubectl rollout undo statefulset/postgres -n postgres-cluster

# Force update โดยลบ pod (ระวัง!)
kubectl delete pod postgres-2 -n postgres-cluster
```

### ปัญหา: Pod ไม่ Ready

```bash
# ดู logs ของ pod
kubectl logs postgres-0 -n postgres-cluster

# ดู logs ของ init container
kubectl logs postgres-0 -n postgres-cluster -c init-postgres

# exec เข้าไปตรวจสอบ
kubectl exec -it postgres-0 -n postgres-cluster -- bash

# ตรวจสอบ readiness probe
kubectl describe pod postgres-0 -n postgres-cluster | grep -A 10 "Readiness"
```

### ตรวจสอบ StatefulSet Health

```bash
# ดู StatefulSet overview
kubectl get statefulset -A

# ดู detailed status
kubectl describe statefulset postgres -n postgres-cluster

# ดู pod ทั้งหมดของ StatefulSet
kubectl get pods -l app=postgres -n postgres-cluster -o wide

# ดู PVC ทั้งหมด
kubectl get pvc -n postgres-cluster

# ดู endpoints ของ headless service
kubectl get endpoints postgres -n postgres-cluster
```

---

## Best Practices

### 1. ตั้งค่า Resource Requests และ Limits เสมอ

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "1000m"
```

### 2. ใช้ Readiness Probe ที่เหมาะสม

```yaml
# สำหรับ Database ใช้ command probe
readinessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
```

### 3. กำหนด PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: postgres-cluster
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
```

### 4. ใช้ Anti-Affinity สำหรับ High Availability

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - postgres
              topologyKey: kubernetes.io/hostname
```

### 5. Backup Strategy

```bash
# สร้าง CronJob สำหรับ backup PostgreSQL
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: postgres-cluster
spec:
  schedule: "0 2 * * *"  # ทุกวัน 2am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:14
            command:
            - bash
            - -c
            - |
              BACKUP_FILE="/backup/postgres-$(date +%Y%m%d-%H%M%S).sql"
              pg_dump -h postgres-primary -U postgres -d mydb > $BACKUP_FILE
              echo "Backup completed: $BACKUP_FILE"
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: postgres-password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
          restartPolicy: OnFailure
```

### 6. Monitoring และ Alerting

```bash
# ตรวจสอบ StatefulSet status แบบสม่ำเสมอ
kubectl get statefulset -A -o custom-columns=\
'NAME:.metadata.name,NAMESPACE:.metadata.namespace,READY:.status.readyReplicas,DESIRED:.spec.replicas'

# สร้าง PrometheusRule สำหรับ alert
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: statefulset-alerts
spec:
  groups:
  - name: statefulset
    rules:
    - alert: StatefulSetNotReady
      expr: kube_statefulset_status_replicas_ready / kube_statefulset_status_replicas_desired < 1
      for: 5m
      annotations:
        summary: "StatefulSet {{ $labels.statefulset }} not ready"
```

---

## สรุป

StatefulSet เป็นเครื่องมือสำคัญสำหรับการรัน stateful applications ใน Kubernetes:

1. **ใช้ StatefulSet เมื่อ**: แอปต้องการ stable identity, persistent storage, หรือ ordered operations
2. **Headless Service**: จำเป็นสำหรับ DNS-based service discovery
3. **VolumeClaimTemplates**: สร้าง PVC แยกสำหรับแต่ละ Pod
4. **Ordered Operations**: Pod ถูกสร้าง/ลบตามลำดับ
5. **Update Strategies**: ใช้ partition สำหรับ canary updates

ในบทต่อไป เราจะเรียนรู้เรื่อง **DaemonSets** - วิธีรัน Pod บนทุก Node ในคลัสเตอร์
