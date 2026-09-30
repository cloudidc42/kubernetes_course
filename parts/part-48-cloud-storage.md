# Part 48: Cloud Storage - AWS EBS, GCE PD, Azure Disk และ Object Storage

## บทนำ

การ deploy Kubernetes บน cloud มาพร้อมกับ storage solutions ที่ cloud providers เตรียมไว้ให้ ซึ่งมีข้อดีคือ managed service ที่ reliable, scalable และ integrate กับ cloud ecosystem ได้ดี

### ประเภทของ Cloud Storage สำหรับ Kubernetes

| ประเภท | ตัวอย่าง | Access Mode | Use Case |
|--------|---------|-------------|---------|
| Block Storage | AWS EBS, GCE PD, Azure Disk | RWO | Database, stateful apps |
| File Storage | AWS EFS, Azure Files, GCP Filestore | RWX | Shared files |
| Object Storage | S3, GCS, Azure Blob | - (via SDK) | Backups, static assets |

---

## 1. AWS EBS (Elastic Block Store)

### คุณสมบัติ AWS EBS

- **Block storage** สำหรับ EC2 instances และ EKS
- **Single-node attachment** (RWO เท่านั้น)
- **Multiple volume types:** gp2, gp3, io1, io2, st1, sc1
- **Snapshots** สำหรับ backup
- **Encryption** ด้วย AWS KMS

### EBS Volume Types

| Type | คำอธิบาย | Max IOPS | Max Throughput | Use Case |
|------|---------|----------|----------------|---------|
| gp3 | General Purpose SSD v3 | 16,000 | 1,000 MiB/s | General workloads |
| gp2 | General Purpose SSD v2 | 16,000 | 250 MiB/s | Legacy |
| io2 | Provisioned IOPS SSD | 64,000 | 1,000 MiB/s | Critical databases |
| io1 | Provisioned IOPS SSD | 64,000 | 1,000 MiB/s | Legacy |
| st1 | Throughput Optimized HDD | 500 | 500 MiB/s | Big data, logs |
| sc1 | Cold HDD | 250 | 250 MiB/s | Infrequent access |

### 1.1 ติดตั้ง AWS EBS CSI Driver บน EKS

```bash
# สร้าง IAM Policy สำหรับ EBS CSI Driver
aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ec2:CreateSnapshot",
          "ec2:AttachVolume",
          "ec2:DetachVolume",
          "ec2:ModifyVolume",
          "ec2:DescribeAvailabilityZones",
          "ec2:DescribeInstances",
          "ec2:DescribeSnapshots",
          "ec2:DescribeTags",
          "ec2:DescribeVolumes",
          "ec2:DescribeVolumesModifications"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "ec2:CreateTags"
        ],
        "Resource": ["arn:aws:ec2:*:*:volume/*", "arn:aws:ec2:*:*:snapshot/*"],
        "Condition": {
          "StringEquals": {
            "ec2:CreateAction": ["CreateVolume", "CreateSnapshot"]
          }
        }
      },
      {
        "Effect": "Allow",
        "Action": ["ec2:DeleteTags"],
        "Resource": ["arn:aws:ec2:*:*:volume/*", "arn:aws:ec2:*:*:snapshot/*"]
      },
      {
        "Effect": "Allow",
        "Action": ["ec2:CreateVolume"],
        "Resource": "*",
        "Condition": {
          "StringLike": {"aws:RequestTag/ebs.csi.aws.com/cluster": "true"}
        }
      },
      {
        "Effect": "Allow",
        "Action": ["ec2:DeleteVolume"],
        "Resource": "*",
        "Condition": {
          "StringLike": {"ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"}
        }
      }
    ]
  }'

# สร้าง IRSA (IAM Role for Service Account)
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::ACCOUNT_ID:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole

# ติดตั้ง EBS CSI Driver ด้วย Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole

# ตรวจสอบ
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
kubectl get csidrivers ebs.csi.aws.com
```

### 1.2 EBS StorageClasses

```yaml
# ebs-storage-classes.yaml
---
# GP3 (แนะนำ - cost effective และ performance)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  csi.storage.k8s.io/fstype: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# GP3 High Performance
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-high
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "16000"
  throughput: "1000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
---
# IO2 Database
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-io2-database
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "64"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456:key/prod-key"
  csi.storage.k8s.io/fstype: xfs
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### 1.3 EBS Volume Snapshots

```yaml
# ebs-snapshot.yaml
---
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  csi.storage.k8s.io/snapshotter-secret-name: aws-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: kube-system
---
# สร้าง Snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: database-snapshot-2024
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: database-pvc  # PVC ที่ต้องการ snapshot
---
# Restore จาก Snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc-restored
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: database-snapshot-2024
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## 2. GCE Persistent Disk

### คุณสมบัติ GCE Persistent Disk

- **Network-attached block storage** สำหรับ GKE
- **Regional PD** สำหรับ multi-zone HA
- **Volume types:** pd-standard, pd-balanced, pd-ssd, hyperdisk-extreme

### 2.1 ติดตั้ง GCP PD CSI Driver บน GKE

```bash
# GKE ติดตั้ง CSI Driver อัตโนมัติ
# ตรวจสอบ
kubectl get csidrivers pd.csi.storage.gke.io

# สำหรับ self-managed Kubernetes บน GCP
helm repo add pd-csi-driver https://kubernetes-sigs.github.io/gcp-compute-persistent-disk-csi-driver
helm install gcp-pd-csi-driver pd-csi-driver/pd-driver \
  --namespace gce-pd-csi-driver \
  --create-namespace \
  --set image.repository=registry.k8s.io/cloud-provider-gcp/gcp-compute-persistent-disk-csi-driver
```

### 2.2 GCE PD StorageClasses

```yaml
# gce-storage-classes.yaml
---
# SSD Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd-retain
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
  csi.storage.k8s.io/fstype: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
---
# Regional SSD (Multi-zone HA)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
allowedTopologies:
- matchLabelExpressions:
  - key: topology.gke.io/zone
    values:
    - us-central1-a
    - us-central1-b
---
# Hyperdisk Extreme (highest performance)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hyperdisk-extreme
provisioner: pd.csi.storage.gke.io
parameters:
  type: hyperdisk-extreme
  provisioned-iops-on-create: "100000"
volumeBindingMode: WaitForFirstConsumer
```

---

## 3. Azure Disk

### คุณสมบัติ Azure Disk

- **Managed Disk** สำหรับ AKS
- **Disk types:** Premium SSD, Standard SSD, Standard HDD, Ultra Disk
- **Zone-redundant storage** สำหรับ HA

### 3.1 Azure Disk CSI Driver

```bash
# AKS มี CSI Driver ติดตั้งมาแล้ว
kubectl get csidrivers disk.csi.azure.com

# สำหรับ self-managed
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
  --namespace kube-system \
  --version v1.29.0
```

### 3.2 Azure StorageClasses

```yaml
# azure-storage-classes.yaml
---
# Premium SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
  cachingmode: ReadOnly
  NetworkAccessPolicy: DenyAll
  PublicNetworkAccess: Disabled
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# Ultra Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-ultra-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: UltraSSD_LRS
  diskIOPSReadWrite: "4000"
  diskMBpsReadWrite: "1000"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
---
# Zone-Redundant Storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-zrs-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_ZRS    # Zone Redundant
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

---

## 4. Object Storage Integration

Object Storage ไม่ได้ mount เป็น Volume โดยตรง แต่เข้าถึงผ่าน SDK หรือ tools พิเศษ

### 4.1 AWS S3 - Mountpoint

**Mountpoint for Amazon S3 CSI Driver:**

```bash
# ติดตั้ง Mountpoint S3 CSI Driver
helm install mountpoint-s3-csi-driver \
  https://awslabs.github.io/mountpoint-s3-csi-driver/charts/aws-mountpoint-s3-csi-driver-1.5.0.tgz \
  --namespace kube-system

# ตรวจสอบ
kubectl get csidrivers s3.csi.aws.com
```

```yaml
# s3-pv-pvc.yaml
---
# Static PV สำหรับ S3 Bucket
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3-pv
spec:
  capacity:
    storage: 1200Gi    # ขนาดที่ประกาศ (ไม่ใช่ S3 limit จริงๆ)
  accessModes:
  - ReadWriteMany
  mountOptions:
  - allow-delete
  - region us-east-1
  csi:
    driver: s3.csi.aws.com
    volumeHandle: my-s3-bucket-prod    # unique ID
    volumeAttributes:
      bucketName: my-s3-bucket-prod
      region: us-east-1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3-pvc
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1200Gi
  volumeName: s3-pv
```

### 4.2 S3 ด้วย s3fs (FUSE)

```yaml
# s3fs-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-app
  template:
    metadata:
      labels:
        app: s3-app
    spec:
      initContainers:
      - name: s3-mount
        image: perma/s3fs:1.93
        securityContext:
          privileged: true
          capabilities:
            add: [SYS_ADMIN]
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: access-key
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: secret-key
        - name: S3_BUCKET
          value: "my-bucket"
        - name: MNT_POINT
          value: "/mnt/s3"
        command:
        - /bin/sh
        - -c
        - |
          echo "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" > /etc/passwd-s3fs
          chmod 600 /etc/passwd-s3fs
          s3fs $S3_BUCKET $MNT_POINT -o passwd_file=/etc/passwd-s3fs -o use_path_request_style
          echo "S3 mounted successfully"
        volumeMounts:
        - name: s3-fuse
          mountPath: /mnt/s3
          mountPropagation: Bidirectional
      
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: s3-fuse
          mountPath: /usr/share/nginx/html
          mountPropagation: HostToContainer
      
      volumes:
      - name: s3-fuse
        emptyDir: {}
```

### 4.3 Google Cloud Storage (GCS)

**GCS FUSE CSI Driver:**

```bash
# GKE มี GCS FUSE CSI มาให้แล้ว (Kubernetes 1.25+)
# Enable ใน GKE
gcloud container clusters update my-cluster \
  --update-addons GcsFuseCsiDriver=ENABLED \
  --zone us-central1-a
```

```yaml
# gcs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gcs-pod
  annotations:
    gke-gcsfuse/volumes: "true"    # Enable GCS FUSE
spec:
  serviceAccountName: gcs-sa    # SA ที่มี GCS access
  containers:
  - name: app
    image: busybox:1.35
    command: ["sleep", "3600"]
    volumeMounts:
    - name: gcs-fuse
      mountPath: /data
  volumes:
  - name: gcs-fuse
    csi:
      driver: gcsfuse.csi.storage.gke.io
      readOnly: false
      volumeAttributes:
        bucketName: my-gcs-bucket
        mountOptions: "implicit-dirs"
```

### 4.4 Azure Blob Storage

```bash
# AKS Azure Blob CSI Driver (ติดมาแล้วใน AKS)
kubectl get csidrivers blob.csi.azure.com
```

```yaml
# azure-blob-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-blob-nfs
provisioner: blob.csi.azure.com
parameters:
  protocol: nfs        # NFS สำหรับ RWX access
  allowBlobPublicAccess: "false"
  networkEndpointType: privateEndpoint
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- nconnect=4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-blob-pvc
spec:
  accessModes:
  - ReadWriteMany    # Blob NFS รองรับ RWX
  storageClassName: azure-blob-nfs
  resources:
    requests:
      storage: 100Gi
```

---

## 5. Workshop: Deploy App ด้วย Cloud Storage

### Workshop Overview
Deploy production-ready application บน Minikube ที่จำลอง Cloud Storage patterns

### Prerequisites
- Minikube cluster
- Helm installed
- เวลาประมาณ 90-120 นาที

### Step 1: เตรียม Environment

```bash
# สร้าง namespace
kubectl create namespace cloud-storage-workshop
kubectl config set-context --current --namespace=cloud-storage-workshop

# ดู StorageClass ที่มีอยู่
kubectl get storageclass

# ติดตั้ง Local Path Provisioner ถ้ายังไม่มี
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

### Step 2: สร้าง Storage Tiers จำลอง Cloud Storage

```yaml
# simulated-cloud-storage.yaml
---
# จำลอง AWS EBS GP3
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-sim
  labels:
    cloud: aws
    type: block
    tier: standard
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# จำลอง AWS EBS IO2
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-io2-sim
  labels:
    cloud: aws
    type: block
    tier: high-performance
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
# จำลอง NFS (EFS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sim
  labels:
    cloud: aws
    type: file
    tier: shared
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f simulated-cloud-storage.yaml
kubectl get storageclass -l cloud=aws
```

### Step 3: Deploy Multi-tier Application

```yaml
# multi-tier-app.yaml
---
# Tier 1: Database (จำลอง EBS IO2)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: cloud-storage-workshop
type: Opaque
stringData:
  POSTGRES_DB: "production"
  POSTGRES_USER: "appuser"
  POSTGRES_PASSWORD: "CloudApp2024!"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data-pvc
  namespace: cloud-storage-workshop
  labels:
    app: database
    tier: data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ebs-io2-sim
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: cloud-storage-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        envFrom:
        - secretRef:
            name: db-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
        resources:
          requests: { cpu: 250m, memory: 256Mi }
          limits: { cpu: 1000m, memory: 1Gi }
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser"]
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: db-data
        persistentVolumeClaim:
          claimName: db-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: cloud-storage-workshop
spec:
  selector:
    app: database
  ports:
  - port: 5432
---
# Tier 2: Application Server (จำลอง EBS GP3)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: cloud-storage-workshop
  labels:
    app: app-server
    tier: application
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ebs-gp3-sim
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-server
  namespace: cloud-storage-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-server
  template:
    metadata:
      labels:
        app: app-server
    spec:
      initContainers:
      - name: db-wait
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Waiting for database..."
          until nc -z database 5432; do
            echo "Database not ready, waiting..."
            sleep 3
          done
          echo "Database is ready!"
      containers:
      - name: app
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-data
          mountPath: /app/data
        - name: app-config
          mountPath: /etc/nginx/conf.d
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits: { cpu: 500m, memory: 256Mi }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: app-data
        persistentVolumeClaim:
          claimName: app-data-pvc
      - name: app-config
        configMap:
          name: app-nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-nginx-config
  namespace: cloud-storage-workshop
data:
  default.conf: |
    server {
        listen 80;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /health {
            return 200 '{"status":"healthy","pod":"$hostname"}';
            add_header Content-Type application/json;
        }
        
        location /data {
            alias /app/data;
            autoindex on;
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: app-server
  namespace: cloud-storage-workshop
spec:
  selector:
    app: app-server
  ports:
  - port: 80
    nodePort: 30081
  type: NodePort
---
# Tier 3: Shared Storage (จำลอง EFS)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-assets-pvc
  namespace: cloud-storage-workshop
  labels:
    app: shared
    tier: files
spec:
  accessModes: [ReadWriteOnce]   # Local path ไม่รองรับ RWX
  storageClassName: efs-sim
  resources:
    requests:
      storage: 20Gi
---
# Content Init Job
apiVersion: batch/v1
kind: Job
metadata:
  name: content-init
  namespace: cloud-storage-workshop
spec:
  template:
    spec:
      containers:
      - name: init
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          mkdir -p /assets/{images,videos,documents}
          echo "Cloud Storage Workshop Asset" > /assets/documents/welcome.txt
          echo "Initialized at: $(date)" >> /assets/documents/welcome.txt
          echo "Init complete: $(ls -la /assets/)"
        volumeMounts:
        - name: assets
          mountPath: /assets
      volumes:
      - name: assets
        persistentVolumeClaim:
          claimName: shared-assets-pvc
      restartPolicy: OnFailure
```

```bash
kubectl apply -f multi-tier-app.yaml
kubectl get all -n cloud-storage-workshop
kubectl get pvc -n cloud-storage-workshop
```

### Step 4: ตรวจสอบ Storage Provisioning

```bash
# ดู PVCs ที่ถูกสร้าง
kubectl get pvc -n cloud-storage-workshop

# ดู PVs ที่ถูก provision
kubectl get pv | grep cloud-storage

# ดู Details ของแต่ละ PVC
kubectl describe pvc db-data-pvc -n cloud-storage-workshop
kubectl describe pvc app-data-pvc -n cloud-storage-workshop
kubectl describe pvc shared-assets-pvc -n cloud-storage-workshop
```

### Step 5: ทดสอบ Application

```bash
# ดู Pod status
kubectl get pods -n cloud-storage-workshop -w

# Port forward
kubectl port-forward service/app-server 8081:80 -n cloud-storage-workshop &

# ทดสอบ health endpoint
curl http://localhost:8081/health

# ทดสอบ data endpoint
curl http://localhost:8081/data/

# เขียนข้อมูลลง storage
APP_POD=$(kubectl get pods -n cloud-storage-workshop -l app=app-server -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n cloud-storage-workshop $APP_POD -- \
  sh -c 'echo "App data at $(date)" > /app/data/app-$(date +%s).txt'

# ตรวจสอบ
curl http://localhost:8081/data/
```

### Step 6: ทดสอบ Database กับ Persistent Storage

```bash
# Connect to database
DB_POD=$(kubectl get pods -n cloud-storage-workshop -l app=database -o jsonpath='{.items[0].metadata.name}')

# สร้าง Table
kubectl exec -it $DB_POD -n cloud-storage-workshop -- \
  psql -U appuser -d production -c "
    CREATE TABLE IF NOT EXISTS cloud_storage_test (
      id SERIAL PRIMARY KEY,
      message TEXT,
      cloud_provider VARCHAR(50),
      storage_type VARCHAR(50),
      created_at TIMESTAMP DEFAULT NOW()
    );
  "

# Insert ข้อมูล
kubectl exec -it $DB_POD -n cloud-storage-workshop -- \
  psql -U appuser -d production -c "
    INSERT INTO cloud_storage_test (message, cloud_provider, storage_type) VALUES
      ('Data on EBS IO2', 'AWS', 'EBS'),
      ('Persistent database record', 'GCP', 'PD-SSD'),
      ('HA database entry', 'Azure', 'Premium-SSD');
  "

# Query
kubectl exec -it $DB_POD -n cloud-storage-workshop -- \
  psql -U appuser -d production -c "SELECT * FROM cloud_storage_test;"
```

### Step 7: ทดสอบ Volume Expansion

```bash
# ขยาย app-data-pvc
kubectl patch pvc app-data-pvc -n cloud-storage-workshop \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# ดู progress
kubectl get pvc app-data-pvc -n cloud-storage-workshop -w

# ตรวจสอบ ใน Pod
kubectl exec -n cloud-storage-workshop $APP_POD -- df -h /app/data
```

### Step 8: Simulate Storage Failure Recovery

```bash
# เขียน checkpoint data
kubectl exec -n cloud-storage-workshop $APP_POD -- \
  sh -c 'echo "CHECKPOINT: $(date)" > /app/data/checkpoint.txt'

# ลบ Pod
kubectl delete pod $APP_POD -n cloud-storage-workshop

# รอ Pod ใหม่
kubectl get pods -n cloud-storage-workshop -l app=app-server -w

# ตรวจสอบ checkpoint ยังอยู่
NEW_APP_POD=$(kubectl get pods -n cloud-storage-workshop -l app=app-server -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n cloud-storage-workshop $NEW_APP_POD -- cat /app/data/checkpoint.txt
```

### Step 9: Multi-AZ Deployment Simulation

```yaml
# multi-az-example.yaml (สำหรับ Cloud Cluster จริง)
# ตัวอย่าง: StatefulSet ที่กระจาย pods ต่าง AZ
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: multi-az-db
spec:
  replicas: 3
  serviceName: multi-az-db
  selector:
    matchLabels:
      app: multi-az-db
  template:
    metadata:
      labels:
        app: multi-az-db
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: multi-az-db
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: db
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: ebs-gp3-sim
      resources:
        requests:
          storage: 10Gi
```

### Step 10: ทำความสะอาด

```bash
kubectl delete namespace cloud-storage-workshop
kubectl delete storageclass ebs-gp3-sim ebs-io2-sim efs-sim
```

---

## 6. Storage Cost Optimization

### 6.1 AWS EBS Cost Tips

```bash
# ตรวจสอบ unattached EBS volumes (ค่าใช้จ่ายสูญเปล่า)
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' \
  --output table

# ตรวจสอบ EBS snapshots เก่า
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[?StartTime<=`2024-01-01`].[SnapshotId,StartTime,VolumeSize]'
```

### 6.2 PVC Monitoring

```bash
# ดู PVC ที่ไม่มี Pod ใช้ (อาจลบได้)
kubectl get pvc -A -o json | jq -r '
  .items[] | 
  select(.status.phase == "Bound") | 
  "\(.metadata.namespace)/\(.metadata.name): \(.spec.resources.requests.storage)"
'

# ดู PVC ที่ Released (รอ reclaim)
kubectl get pv | grep Released
```

### 6.3 StorageClass Selection Matrix

```
Workload          | Recommended StorageClass | Reason
------------------|--------------------------|--------
PostgreSQL        | EBS IO2 / Premium SSD    | High IOPS, consistent latency
MySQL             | EBS GP3 High IOPS        | Good performance, cost effective
Redis Cache       | EBS GP3                  | Fast, moderate IOPS
Elasticsearch     | EBS GP3 Large            | Large sequential reads
Kafka             | EBS ST1                  | Throughput-optimized
Web Assets        | S3/GCS/Blob              | Cost-effective, CDN integration
Logs Archive      | S3/GCS Cold              | Very low cost
Shared Config     | NFS/EFS/Azure Files      | Multi-pod access
```

---

## สรุป

| Cloud | Block Storage | File Storage | Object Storage |
|-------|--------------|--------------|----------------|
| AWS | EBS (gp3, io2) | EFS | S3 |
| GCP | Persistent Disk | Filestore | GCS |
| Azure | Managed Disk | Azure Files | Azure Blob |

### Key Takeaways:
1. **Block Storage** (EBS, PD, Azure Disk) เหมาะสำหรับ databases และ stateful apps (RWO)
2. **File Storage** (EFS, Filestore, Azure Files) เหมาะสำหรับ shared storage (RWX)
3. **Object Storage** ไม่ mount เป็น filesystem โดยตรง ต้องใช้ FUSE driver หรือ SDK
4. **CSI Drivers** เป็น modern approach แทน built-in provisioners ที่ deprecated
5. **Volume Snapshots** สำคัญสำหรับ disaster recovery
6. ใช้ **WaitForFirstConsumer** binding mode สำหรับ topology-aware storage

## แหล่งเรียนรู้เพิ่มเติม

- [AWS EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [GCP PD CSI Driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver)
- [Azure Disk CSI Driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver)
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
