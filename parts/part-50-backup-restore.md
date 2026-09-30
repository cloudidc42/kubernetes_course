# Part 50: Backup และ Restore - การปกป้องข้อมูลใน Kubernetes

## บทนำ

การ Backup และ Restore ใน Kubernetes เป็นสิ่งสำคัญอย่างยิ่งสำหรับ production environment เพราะ Kubernetes cluster มีหลาย components ที่ต้องปกป้อง ได้แก่ cluster state (etcd), application configurations, และ persistent data

### สิ่งที่ต้องทำ Backup ใน Kubernetes

```
Kubernetes Cluster
├── Cluster State (etcd)
│   ├── Pods, Deployments, Services
│   ├── ConfigMaps, Secrets
│   ├── RBAC Policies
│   └── Custom Resources
│
├── Application Configurations
│   ├── Helm releases
│   ├── Kustomize manifests
│   └── Operator configurations
│
└── Persistent Data (PVs)
    ├── Database data
    ├── Application files
    └── Logs
```

---

## 1. Backup Strategies

### 1.1 กลยุทธ์ Backup ต่างๆ

**Strategy 1: etcd Backup**
- Backup etcd database ซึ่งเก็บ cluster state ทั้งหมด
- Restore ได้ทั้ง cluster
- ไม่รวม Persistent Volume data

**Strategy 2: Namespace/Resource Backup**
- Export Kubernetes resources เป็น YAML
- ง่ายแต่ต้อง restore ทีละ resource

**Strategy 3: Volume Snapshots**
- ใช้ CSI Volume Snapshot
- รวดเร็วและ consistent สำหรับ volumes
- ต้องการ storage ที่รองรับ snapshot

**Strategy 4: Application-level Backup**
- Backup ด้วย application tools (pg_dump, mysqldump)
- Consistent backup สำหรับ databases
- ต้องจัดการเอง

**Strategy 5: Velero (แนะนำสำหรับ full backup)**
- Backup ทั้ง cluster resources + volumes
- Schedule backups
- Cross-cluster restore

### 1.2 Backup Strategy Matrix

| Strategy | Cluster State | App Config | Volume Data | Complexity | Recovery Time |
|----------|---------------|-----------|-------------|-----------|---------------|
| etcd only | ✓ | ✓ | - | Medium | Fast |
| YAML export | ✓ | ✓ | - | Low | Slow |
| Volume Snapshot | - | - | ✓ | Low | Fast |
| App-level | - | - | ✓ | Medium | Medium |
| Velero | ✓ | ✓ | ✓ | Medium | Fast |
| GitOps + Velero | ✓ | ✓ | ✓ | High | Fastest |

### 1.3 RPO และ RTO

```
RPO (Recovery Point Objective) = ข้อมูลสูญหายได้มากที่สุดเท่าไหร่?
RTO (Recovery Time Objective) = ระบบต้องกลับมาใช้งานได้ภายในเท่าไหร่?

ตัวอย่าง:
- Production Database: RPO < 5 นาที, RTO < 30 นาที
- Dev Environment: RPO < 1 วัน, RTO < 4 ชั่วโมง
- Archive Data: RPO < 1 วัน, RTO < 24 ชั่วโมง
```

---

## 2. etcd Backup

### 2.1 Manual etcd Backup

```bash
# etcd backup ด้วย etcdctl
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# ตรวจสอบ snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table

# Output:
# +---------+----------+------------+------------+
# |  HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +---------+----------+------------+------------+
# | abc1234 |   123456 |       1234 |    4.2 MB  |
# +---------+----------+------------+------------+
```

### 2.2 etcd Restore

```bash
# หยุด kube-apiserver ก่อน restore (ถ้า single node)
# หรือ pause ใน cluster

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Update etcd configuration ให้ชี้ data-dir ใหม่
# แล้ว restart etcd
```

### 2.3 Automated etcd Backup Job

```yaml
# etcd-backup-cronjob.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: etcd-backup
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: etcd-backup
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # ต้องการ access สูง
subjects:
- kind: ServiceAccount
  name: etcd-backup
  namespace: kube-system
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"    # ทุกวัน 2AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: etcd-backup
          hostNetwork: true
          
          containers:
          - name: backup
            image: bitnami/etcd:3.5
            
            env:
            - name: ETCDCTL_API
              value: "3"
            - name: ETCD_ENDPOINTS
              value: "https://127.0.0.1:2379"
            - name: BACKUP_DIR
              value: "/backup"
            
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db"
              
              echo "Starting etcd backup at $(date)"
              
              etcdctl snapshot save ${BACKUP_FILE} \
                --endpoints=${ETCD_ENDPOINTS} \
                --cacert=/etc/etcd/pki/ca.crt \
                --cert=/etc/etcd/pki/server.crt \
                --key=/etc/etcd/pki/server.key
              
              if [ $? -eq 0 ]; then
                echo "Backup successful: ${BACKUP_FILE}"
                # ลบ backup เก่ากว่า 7 วัน
                find ${BACKUP_DIR} -name "etcd-snapshot-*.db" -mtime +7 -delete
                echo "Old backups cleaned up"
                ls -la ${BACKUP_DIR}/
              else
                echo "Backup FAILED!"
                exit 1
              fi
            
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/etcd/pki
              readOnly: true
            - name: backup-storage
              mountPath: /backup
          
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-storage
            hostPath:
              path: /var/backup/etcd
              type: DirectoryOrCreate
          
          restartPolicy: OnFailure
          
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
            effect: NoSchedule
          
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
```

```bash
kubectl apply -f etcd-backup-cronjob.yaml
# ทดสอบ manual run
kubectl create job --from=cronjob/etcd-backup etcd-backup-manual -n kube-system
kubectl logs job/etcd-backup-manual -n kube-system
```

---

## 3. Velero สำหรับ Backup

### 3.1 Velero คืออะไร

**Velero** (เดิมชื่อ Heptio Ark) คือ open-source tool สำหรับ:
- Backup และ Restore Kubernetes cluster resources
- Disaster recovery
- Cluster migration
- Namespace migration

### 3.2 Velero Architecture

```
Velero Server (ใน Cluster)
    │
    ├── Object Store Plugin (S3, GCS, Azure Blob, MinIO)
    │   └── Backup/Restore cluster resources (YAML)
    │
    └── Volume Snapshot Plugin (CSI, Restic/Kopia)
        └── Backup/Restore Persistent Volumes
```

### 3.3 Velero Components

- **velero server**: Deployment ใน cluster
- **velero CLI**: Command-line tool
- **BackupStorageLocation**: ที่เก็บ backup files
- **VolumeSnapshotLocation**: ที่เก็บ volume snapshots
- **Backup**: Backup job
- **Restore**: Restore job
- **Schedule**: Scheduled backup
- **BackupRepository**: Repository สำหรับ Restic/Kopia

---

## 4. ติดตั้งและ Configure Velero

### 4.1 ติดตั้ง Velero CLI

```bash
# ดาวน์โหลด Velero CLI
VELERO_VERSION=v1.13.0

# Linux
wget https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz
tar -xvzf velero-${VELERO_VERSION}-linux-amd64.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/

# macOS
brew install velero

# ตรวจสอบ
velero version --client-only
```

### 4.2 ติดตั้ง Velero กับ MinIO (Local S3-compatible)

สำหรับ local/development environment ใช้ MinIO แทน cloud storage

**ติดตั้ง MinIO:**

```yaml
# minio-setup.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: velero
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:RELEASE.2024-01-01T00-00-00Z
        command:
        - /bin/bash
        - -c
        args:
        - minio server /data --console-address ":9001"
        env:
        - name: MINIO_ROOT_USER
          value: "minio"
        - name: MINIO_ROOT_PASSWORD
          value: "minio123"
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    app: minio
  ports:
  - port: 9000
    name: api
    nodePort: 30900
  - port: 9001
    name: console
    nodePort: 30901
  type: NodePort
```

```bash
kubectl apply -f minio-setup.yaml
kubectl wait --for=condition=available deployment/minio -n velero --timeout=120s

# Port forward MinIO
kubectl port-forward service/minio 9000:9000 9001:9001 -n velero &

# สร้าง bucket
# Install MinIO client
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

mc alias set velero-minio http://localhost:9000 minio minio123
mc mb velero-minio/velero-backups
mc ls velero-minio/
```

**ติดตั้ง Velero กับ MinIO:**

```bash
# สร้าง credentials file
cat > credentials-velero << 'EOF'
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF

# ติดตั้ง Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
  --namespace velero

# รอ Velero start
kubectl wait --for=condition=available deployment/velero -n velero --timeout=180s
kubectl get pods -n velero
```

### 4.3 ติดตั้ง Velero กับ AWS S3

```bash
# สร้าง S3 bucket
aws s3api create-bucket \
  --bucket velero-backup-my-cluster \
  --region us-east-1

# สร้าง IAM Policy
aws iam create-policy \
  --policy-name VeleroBackupPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ec2:DescribeVolumes",
          "ec2:DescribeSnapshots",
          "ec2:CreateTags",
          "ec2:CreateVolume",
          "ec2:CreateSnapshot",
          "ec2:DeleteSnapshot"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:DeleteObject",
          "s3:PutObject",
          "s3:AbortMultipartUpload",
          "s3:ListMultipartUploadParts"
        ],
        "Resource": ["arn:aws:s3:::velero-backup-my-cluster/*"]
      },
      {
        "Effect": "Allow",
        "Action": ["s3:ListBucket"],
        "Resource": ["arn:aws:s3:::velero-backup-my-cluster"]
      }
    ]
  }'

# ติดตั้ง Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backup-my-cluster \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=true

# ตรวจสอบ
velero backup-location get
velero snapshot-location get
```

### 4.4 ติดตั้ง Velero ด้วย Helm

```bash
# เพิ่ม Helm repo
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# ติดตั้งด้วย Helm
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=aws \
  --set-file credentials.secretContents.cloud=./credentials-velero \
  --set configuration.backupStorageLocation[0].name=default \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=velero-backups \
  --set configuration.backupStorageLocation[0].config.region=minio \
  --set configuration.backupStorageLocation[0].config.s3ForcePathStyle="true" \
  --set configuration.backupStorageLocation[0].config.s3Url=http://minio.velero.svc:9000 \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=velero/velero-plugin-for-aws:v1.9.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins

# ตรวจสอบ
kubectl get pods -n velero
velero version
```

---

## 5. Backup Procedures

### 5.1 สร้าง Backup ด้วยตนเอง

```bash
# Backup ทั้ง cluster
velero backup create full-cluster-backup \
  --include-namespaces '*' \
  --wait

# Backup เฉพาะ Namespace
velero backup create production-backup \
  --include-namespaces production,databases \
  --wait

# Backup เฉพาะ Resources บางประเภท
velero backup create configs-backup \
  --include-resources configmaps,secrets,deployments,services \
  --include-namespaces production \
  --wait

# Backup พร้อม Volume data
velero backup create full-with-volumes \
  --include-namespaces production \
  --default-volumes-to-restic \
  --wait

# ดู Backup status
velero backup get
velero backup describe full-cluster-backup
velero backup logs full-cluster-backup
```

### 5.2 ตั้ง Scheduled Backup

```bash
# Daily backup ทุก 2AM
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces '*' \
  --ttl 168h    # เก็บไว้ 7 วัน

# Hourly backup เฉพาะ databases
velero schedule create hourly-database-backup \
  --schedule="@every 1h" \
  --include-namespaces databases \
  --ttl 24h

# Weekly full backup
velero schedule create weekly-full-backup \
  --schedule="0 0 * * 0" \
  --include-namespaces '*' \
  --ttl 720h    # เก็บไว้ 30 วัน

# ดู Schedules
velero schedule get
velero schedule describe daily-backup

# Trigger manual run จาก schedule
velero backup create --from-schedule daily-backup
```

### 5.3 Backup ด้วย YAML

```yaml
# velero-backup.yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: production-full-backup
  namespace: velero
spec:
  includedNamespaces:
  - production
  - databases
  
  excludedNamespaces:
  - kube-system
  
  includedResources:
  - '*'   # ทุก resources
  
  excludedResources:
  - events.events.k8s.io
  
  labelSelector:
    matchLabels:
      backup: "true"     # เฉพาะ resources ที่มี label นี้
  
  storageLocation: default
  
  volumeSnapshotLocations:
  - default
  
  ttl: 720h    # 30 วัน
  
  defaultVolumesToRestic: true    # Backup volumes ด้วย Restic
  
  hooks:
    resources:
    - name: database-hook
      includedNamespaces: [databases]
      includedResources: [pods]
      labelSelector:
        matchLabels:
          app: postgres
      pre:
      - exec:
          container: postgres
          command:
          - /bin/bash
          - -c
          - PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -c "SELECT pg_start_backup('velero-backup', true);"
          onError: Fail
          timeout: 60s
      post:
      - exec:
          container: postgres
          command:
          - /bin/bash
          - -c
          - PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -c "SELECT pg_stop_backup();"
          onError: Fail
          timeout: 60s
```

```bash
kubectl apply -f velero-backup.yaml
velero backup get production-full-backup
velero backup describe production-full-backup --details
```

### 5.4 Scheduled Backup ด้วย YAML

```yaml
# velero-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-production-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"    # Daily at 2AM UTC
  useOwnerReferencesInBackup: false
  
  template:
    includedNamespaces:
    - production
    - databases
    - monitoring
    
    ttl: 168h    # 7 days
    
    storageLocation: default
    
    defaultVolumesToRestic: true
    
    labelSelector:
      matchExpressions:
      - key: "do-not-backup"
        operator: DoesNotExist
```

```bash
kubectl apply -f velero-schedule.yaml
velero schedule get
```

---

## 6. Restore Procedures

### 6.1 Restore ทั้ง Backup

```bash
# ดู Backups ที่มีอยู่
velero backup get

# Restore ทั้ง backup
velero restore create --from-backup production-full-backup \
  --wait

# ดู Restore status
velero restore get
velero restore describe <restore-name>
velero restore logs <restore-name>
```

### 6.2 Restore เฉพาะ Namespace

```bash
# Restore เฉพาะ namespace
velero restore create restore-databases \
  --from-backup full-cluster-backup \
  --include-namespaces databases \
  --wait

# Restore ไปยัง Namespace ใหม่
velero restore create restore-to-staging \
  --from-backup production-full-backup \
  --namespace-mappings production:staging \
  --wait
```

### 6.3 Restore เฉพาะ Resources

```bash
# Restore เฉพาะ Deployments
velero restore create restore-deployments \
  --from-backup production-backup \
  --include-resources deployments,configmaps,secrets \
  --wait

# Restore เฉพาะ Resource ที่มี Label
velero restore create selective-restore \
  --from-backup production-backup \
  --selector "app=myapp" \
  --wait
```

### 6.4 Restore ด้วย YAML

```yaml
# velero-restore.yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: production-restore-20240115
  namespace: velero
spec:
  backupName: production-full-backup
  
  includedNamespaces:
  - production
  
  excludedNamespaces: []
  
  includedResources: ['*']
  
  excludedResources:
  - nodes
  - events
  - events.events.k8s.io
  - backups.velero.io
  - restores.velero.io
  
  # Restore volumes ด้วย
  restorePVs: true
  
  # Overwrite existing resources
  existingResourcePolicy: update
  
  # Hooks (เช่น database restore)
  hooks:
    resources:
    - name: database-restore-hook
      includedNamespaces: [production]
      includedResources: [pods]
      labelSelector:
        matchLabels:
          app: postgres
      init:
        initContainers:
        - name: wait-for-postgres
          image: busybox:1.35
          command: ["sh", "-c", "until nc -z localhost 5432; do sleep 2; done"]
```

```bash
kubectl apply -f velero-restore.yaml
velero restore get
velero restore describe production-restore-20240115 --details
```

---

## 7. Workshop: Backup และ Restore Cluster State

### Workshop Overview
ติดตั้ง Velero พร้อม MinIO และทำ full backup/restore cycle

### Prerequisites
- Kubernetes cluster (Minikube)
- Helm installed
- เวลาประมาณ 120-150 นาที

### Step 1: เตรียม Workshop Application

```bash
# สร้าง workshop namespace
kubectl create namespace backup-workshop
kubectl config set-context --current --namespace=backup-workshop
```

```yaml
# workshop-app.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: backup-workshop
  labels:
    app: workshop
    backup: "true"
data:
  config.json: |
    {
      "version": "1.0.0",
      "environment": "workshop",
      "features": {
        "backup-demo": true,
        "restore-demo": true
      }
    }
  message: "Hello from Backup Workshop!"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: backup-workshop
  labels:
    app: workshop
type: Opaque
stringData:
  api-key: "workshop-api-key-2024"
  db-password: "workshop-db-pass"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: backup-workshop
  labels:
    app: workshop
    backup: "true"
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: backup-workshop
  labels:
    app: workshop
    backup: "true"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop
  template:
    metadata:
      labels:
        app: workshop
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          CONFIG=$(cat /config/config.json)
          echo "Workshop app started at $(date)"
          echo "Config: $CONFIG"
          
          # Write persistent data
          echo "Workshop data: $(date)" >> /data/workshop.log
          echo "Pod: $HOSTNAME" >> /data/pods.log
          
          while true; do
            echo "$(date): Heartbeat from $HOSTNAME" >> /data/heartbeat.log
            sleep 30
          done
        volumeMounts:
        - name: app-data
          mountPath: /data
        - name: app-config
          mountPath: /config
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
      volumes:
      - name: app-data
        persistentVolumeClaim:
          claimName: app-data-pvc
      - name: app-config
        configMap:
          name: app-config
---
apiVersion: v1
kind: Service
metadata:
  name: workshop-app
  namespace: backup-workshop
  labels:
    app: workshop
spec:
  selector:
    app: workshop
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f workshop-app.yaml
kubectl get all -n backup-workshop
kubectl get pvc -n backup-workshop
```

### Step 2: เพิ่มข้อมูลสำคัญ

```bash
# รอ Pod start
kubectl get pods -n backup-workshop -w

# เพิ่มข้อมูลสำคัญ
APP_POD=$(kubectl get pods -n backup-workshop -l app=workshop -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n backup-workshop $APP_POD -- \
  sh -c 'echo "CRITICAL DATA: Workshop backup test at $(date)" > /data/critical.txt'

kubectl exec -n backup-workshop $APP_POD -- \
  sh -c 'for i in 1 2 3 4 5; do echo "Record $i: $(date)" >> /data/records.db; done'

# ตรวจสอบข้อมูล
kubectl exec -n backup-workshop $APP_POD -- cat /data/critical.txt
kubectl exec -n backup-workshop $APP_POD -- cat /data/records.db
```

### Step 3: ติดตั้ง MinIO

```bash
# Deploy MinIO
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: velero
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        command: ["/bin/bash", "-c"]
        args: ["minio server /data --console-address :9001"]
        env:
        - name: MINIO_ROOT_USER
          value: "minioadmin"
        - name: MINIO_ROOT_PASSWORD
          value: "minioadmin"
        ports:
        - containerPort: 9000
        - containerPort: 9001
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    app: minio
  ports:
  - port: 9000
    name: api
  - port: 9001
    name: console
EOF

kubectl wait --for=condition=available deployment/minio -n velero --timeout=120s

# Port forward
kubectl port-forward service/minio -n velero 9000:9000 &

# สร้าง bucket ด้วย mc
curl -sO https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc

./mc alias set myminio http://localhost:9000 minioadmin minioadmin
./mc mb myminio/velero
./mc ls myminio/
```

### Step 4: ติดตั้ง Velero

```bash
# สร้าง credentials
cat > credentials-velero << 'EOF'
[default]
aws_access_key_id=minioadmin
aws_secret_access_key=minioadmin
EOF

# ติดตั้ง Velero CLI
VELERO_VERSION=v1.13.0
wget -q https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz
tar -xzf velero-${VELERO_VERSION}-linux-amd64.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
velero version --client-only

# ติดตั้ง Velero บน cluster
MINIO_IP=$(kubectl get svc minio -n velero -o jsonpath='{.spec.clusterIP}')

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://${MINIO_IP}:9000 \
  --namespace velero

# รอ Velero start
kubectl wait --for=condition=available deployment/velero -n velero --timeout=180s
kubectl get pods -n velero

# ตรวจสอบ backup location
velero backup-location get
```

### Step 5: สร้าง Backup

```bash
# Backup ทั้ง workshop namespace
velero backup create workshop-backup-v1 \
  --include-namespaces backup-workshop \
  --wait

# ดู status
velero backup get
velero backup describe workshop-backup-v1

# ดู backup contents
velero backup describe workshop-backup-v1 --details | grep "Included"
```

### Step 6: จำลองการ Disaster (ลบข้อมูล)

```bash
# จดบันทึก state ก่อน disaster
echo "=== State Before Disaster ==="
kubectl get all -n backup-workshop
kubectl get pvc -n backup-workshop
kubectl exec -n backup-workshop $APP_POD -- cat /data/critical.txt 2>/dev/null

# จำลอง disaster: ลบทั้ง namespace!
kubectl delete namespace backup-workshop

# รอ namespace ถูกลบ
kubectl get namespace backup-workshop

echo "=== DISASTER! Namespace deleted ==="
kubectl get all -n backup-workshop 2>&1
```

### Step 7: Restore จาก Backup

```bash
# ตรวจสอบ backup ยังอยู่
velero backup get

# Restore namespace
velero restore create workshop-restore-v1 \
  --from-backup workshop-backup-v1 \
  --wait

# ดู restore status
velero restore get
velero restore describe workshop-restore-v1
```

### Step 8: ตรวจสอบ Restore

```bash
# ดู Namespace
kubectl get namespace backup-workshop

# ดู Resources ทั้งหมด
kubectl get all -n backup-workshop

# ดู PVCs
kubectl get pvc -n backup-workshop

# รอ Pods start
kubectl get pods -n backup-workshop -w

# ตรวจสอบข้อมูล
RESTORED_POD=$(kubectl get pods -n backup-workshop -l app=workshop -o jsonpath='{.items[0].metadata.name}')

# ตรวจสอบ ConfigMap
kubectl get configmap app-config -n backup-workshop -o jsonpath='{.data.message}'

# ตรวจสอบ Secret
kubectl get secret app-secrets -n backup-workshop -o jsonpath='{.data.api-key}' | base64 -d

# ตรวจสอบข้อมูลใน PVC
# (อาจต้องรอสักครู่สำหรับ Restic restore)
kubectl exec -n backup-workshop $RESTORED_POD -- cat /data/critical.txt
kubectl exec -n backup-workshop $RESTORED_POD -- cat /data/records.db
```

### Step 9: ตั้ง Scheduled Backup

```bash
# Daily backup
velero schedule create daily-workshop-backup \
  --schedule="@every 24h" \
  --include-namespaces backup-workshop \
  --ttl 168h    # เก็บ 7 วัน

# ดู schedules
velero schedule get

# Manual trigger
velero backup create --from-schedule daily-workshop-backup
velero backup get
```

### Step 10: Test Namespace Migration

```bash
# Restore ไปยัง namespace ใหม่
velero restore create migrate-to-staging \
  --from-backup workshop-backup-v1 \
  --namespace-mappings backup-workshop:staging-workshop \
  --wait

# ดู Namespace ใหม่
kubectl get all -n staging-workshop
kubectl get pvc -n staging-workshop
```

### Step 11: ทำความสะอาด

```bash
# ลบ backup
velero backup delete workshop-backup-v1

# Uninstall Velero
velero uninstall

# ลบ namespaces
kubectl delete namespace backup-workshop staging-workshop velero
```

---

## 8. Application-Level Backup

### 8.1 PostgreSQL Backup

```yaml
# postgres-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: databases
spec:
  schedule: "0 */6 * * *"    # ทุก 6 ชั่วโมง
  concurrencyPolicy: Forbid
  
  jobTemplate:
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
            - name: PGHOST
              value: "postgres.databases.svc.cluster.local"
            - name: PGUSER
              value: "appuser"
            - name: PGDATABASE
              value: "appdb"
            
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="/backup/postgres-${PGDATABASE}-${TIMESTAMP}.sql.gz"
              
              echo "Starting PostgreSQL backup at $(date)"
              
              pg_dump -h ${PGHOST} -U ${PGUSER} ${PGDATABASE} | \
                gzip > ${BACKUP_FILE}
              
              if [ $? -eq 0 ]; then
                echo "Backup successful: ${BACKUP_FILE}"
                echo "Size: $(ls -lh ${BACKUP_FILE} | awk '{print $5}')"
                # ลบ backup เก่ากว่า 7 วัน
                find /backup -name "postgres-*.sql.gz" -mtime +7 -delete
              else
                echo "Backup FAILED!"
                exit 1
              fi
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          
          restartPolicy: OnFailure
```

### 8.2 MySQL Backup

```yaml
# mysql-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
  namespace: databases
spec:
  schedule: "30 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mysql:8.0
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_HOST
              value: "mysql.databases.svc.cluster.local"
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="/backup/mysql-all-${TIMESTAMP}.sql.gz"
              
              mysqldump -h ${MYSQL_HOST} -u root \
                --password="${MYSQL_ROOT_PASSWORD}" \
                --all-databases \
                --single-transaction \
                --routines \
                --triggers | gzip > ${BACKUP_FILE}
              
              if [ $? -eq 0 ]; then
                echo "MySQL backup successful: ${BACKUP_FILE}"
                find /backup -name "mysql-*.sql.gz" -mtime +7 -delete
              fi
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: mysql-backup-pvc
          restartPolicy: OnFailure
```

---

## 9. GitOps Backup Strategy

การใช้ GitOps (ArgoCD/Flux) ร่วมกับ Velero เป็น best practice ที่สุด

```
Git Repository (Source of Truth)
    │
    └── Kubernetes Manifests
    │   ├── Deployments
    │   ├── Services  
    │   └── ConfigMaps
    │
    └── ArgoCD/Flux sync to cluster
    
Velero
    └── Backup Persistent Volume Data
    │   ├── Database dumps
    │   └── Application files
    │
    └── Store in S3/GCS/Azure Blob

Disaster Recovery:
1. Restore git repo → Deploy new cluster
2. ArgoCD sync cluster state
3. Velero restore → Restore volume data
```

---

## 10. Backup Verification

```bash
# Script ตรวจสอบ backup
cat << 'EOF' > verify-backups.sh
#!/bin/bash

echo "=== Velero Backup Health Check ==="
echo "Time: $(date)"
echo ""

echo "--- Backup Locations ---"
velero backup-location get

echo ""
echo "--- Recent Backups ---"
velero backup get | head -20

echo ""
echo "--- Failed Backups ---"
velero backup get | grep -E "PartiallyFailed|Failed"

echo ""
echo "--- Schedules ---"
velero schedule get

echo ""
echo "--- Last 5 Backups Status ---"
velero backup get --output json | jq -r '.items[] | "\(.metadata.name): \(.status.phase) - \(.status.completionTimestamp)"' | head -5

EOF

chmod +x verify-backups.sh
./verify-backups.sh
```

---

## สรุป

| Component | Tool | Purpose |
|-----------|------|---------|
| Cluster State | etcd backup | Full cluster restore |
| App Config | Velero + Git | K8s resources |
| Volume Data | Velero (Restic/Kopia) | PV data |
| Database | pg_dump/mysqldump | App-level backup |
| Schedule | CronJob | Automated backup |
| Storage | S3/MinIO/GCS | Backup destination |

### Key Takeaways:
1. **Velero** เป็น all-in-one solution สำหรับ Kubernetes backup
2. **etcd backup** จำเป็นสำหรับ full cluster disaster recovery
3. **Application-level backup** (pg_dump) ให้ consistent backup สำหรับ databases
4. **3-2-1 Rule**: เก็บ backup 3 copies, 2 storage types, 1 off-site
5. **Test your backups regularly** - backup ที่ไม่ผ่านการทดสอบ restore คือ backup ที่ไม่น่าเชื่อถือ
6. **GitOps + Velero** เป็น best practice สำหรับ production environment

## แหล่งเรียนรู้เพิ่มเติม

- [Velero Documentation](https://velero.io/docs/)
- [etcd Backup and Restore](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Kubernetes Backup Best Practices](https://kubernetes.io/docs/concepts/cluster-administration/)
- [MinIO Documentation](https://min.io/docs/minio/kubernetes/upstream/)
