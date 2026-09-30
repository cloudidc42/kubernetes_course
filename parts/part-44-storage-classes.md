# Part 44: StorageClass - การจัดการ Storage Tiers

## บทนำ

**StorageClass** คือกลไกที่ให้ Administrator กำหนด "classes" หรือ "profiles" ของ storage ที่ Cluster รองรับ แต่ละ StorageClass มี provisioner, parameters และ reclaim policy ของตัวเอง ช่วยให้ Developer สามารถเลือก storage ที่เหมาะสมกับ workload โดยไม่ต้องรู้รายละเอียดของ infrastructure

### ทำไมต้องใช้ StorageClass

**ก่อนมี StorageClass:**
- Admin ต้องสร้าง PV ทีละอัน (manual work)
- Developer ต้องรู้ว่ามี PV อะไรบ้าง
- ไม่สามารถ scale storage ได้อย่างรวดเร็ว

**เมื่อมี StorageClass:**
- Admin กำหนด "class" ของ storage เช่น fast-ssd, slow-hdd
- Developer ขอ PVC โดยระบุ StorageClass
- Kubernetes (provisioner) สร้าง PV ให้อัตโนมัติ (Dynamic Provisioning)

---

## 1. StorageClass คืออะไร

### Components ของ StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-storage-class
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Default class

provisioner: kubernetes.io/aws-ebs   # ใครสร้าง PV

parameters:                          # Parameters สำหรับ provisioner
  type: gp3
  iops: "3000"

reclaimPolicy: Delete                # Delete หรือ Retain

allowVolumeExpansion: true           # อนุญาตให้ขยาย PVC

mountOptions:                        # Mount options
- debug

volumeBindingMode: WaitForFirstConsumer  # Immediate หรือ WaitForFirstConsumer

allowedTopologies:                   # จำกัด topology
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
```

---

## 2. StorageClass YAML - ตัวอย่างต่างๆ

### 2.1 No Provisioner (สำหรับ Manual PV)

```yaml
# no-provisioner-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

### 2.2 HostPath Provisioner (Development)

```yaml
# hostpath-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: docker.io/hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

### 2.3 Local Path Provisioner (Rancher)

```yaml
# local-path-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 2.4 NFS Provisioner

```yaml
# nfs-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  server: 192.168.1.100          # NFS Server IP
  path: /exports/kubernetes      # NFS export path
  readOnly: "false"
  
  # Archive ก่อนลบ (optional)
  archiveOnDelete: "true"
  
  # Path pattern สำหรับ subdirectory
  pathPattern: "${.PVC.namespace}/${.PVC.name}"
  
  # OnDelete/Delete
  onDelete: "archive"

reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

### 2.5 AWS EBS StorageClass

```yaml
# aws-ebs-sc.yaml
# GP2 (General Purpose SSD v2)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs  # หรือ ebs.csi.aws.com สำหรับ CSI
parameters:
  type: gp2
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# GP3 (General Purpose SSD v3) - ถูกกว่าและดีกว่า GP2
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# IO1 (Provisioned IOPS SSD) - สำหรับ high-performance database
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-io1-high-perf
provisioner: ebs.csi.aws.com
parameters:
  type: io1
  iopsPerGB: "50"
  encrypted: "true"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

### 2.6 GCP Persistent Disk

```yaml
# gcp-pd-sc.yaml
# Standard Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  fstype: ext4
  replication-type: none
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# SSD Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  fstype: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Extreme Persistent Disk (Kubernetes 1.26+)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-extreme
provisioner: pd.csi.storage.gke.io
parameters:
  type: hyperdisk-extreme
  provisioned-iops-on-create: "100000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 2.7 Azure Disk

```yaml
# azure-disk-sc.yaml
# Premium SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# Standard HDD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-standard
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: Managed
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Azure File (SMB)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-file
provisioner: kubernetes.io/azure-file
parameters:
  storageAccount: mystorageaccount
  location: eastus
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
- dir_mode=0777
- file_mode=0777
- uid=1000
- gid=1000
- mfsymlinks
- nobrl
```

### 2.8 Rook-Ceph StorageClass

```yaml
# ceph-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  
  # Ceph credentials
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/fstype: ext4

reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
- discard
---
# Ceph RBD Retain
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-retain
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
---
# CephFS (Shared Filesystem)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

---

## 3. Provisioners

### Built-in Provisioners (Legacy - deprecated)

| Provisioner | ใช้กับ |
|-------------|--------|
| `kubernetes.io/aws-ebs` | AWS EBS |
| `kubernetes.io/gce-pd` | GCE Persistent Disk |
| `kubernetes.io/azure-disk` | Azure Managed Disk |
| `kubernetes.io/azure-file` | Azure File |
| `kubernetes.io/no-provisioner` | ไม่มี provisioner (manual PV) |

### CSI Provisioners (Modern)

| Provisioner | ใช้กับ |
|-------------|--------|
| `ebs.csi.aws.com` | AWS EBS CSI Driver |
| `pd.csi.storage.gke.io` | GCP PD CSI Driver |
| `disk.csi.azure.com` | Azure Disk CSI Driver |
| `file.csi.azure.com` | Azure File CSI Driver |
| `rook-ceph.rbd.csi.ceph.com` | Ceph RBD |
| `rook-ceph.cephfs.csi.ceph.com` | CephFS |
| `nfs.csi.k8s.io` | NFS CSI Driver |
| `local.csi.k8s.io` | Local Volume |
| `driver.longhorn.io` | Longhorn |
| `csi.vsphere.volume` | vSphere |

### External Provisioners

| Provisioner | ใช้กับ |
|-------------|--------|
| `k8s-sigs.io/nfs-subdir-external-provisioner` | NFS Subdir |
| `rancher.io/local-path` | Local Path (Rancher) |
| `openebs.io/local` | OpenEBS Local |
| `microk8s.io/hostpath` | MicroK8s |

---

## 4. Default StorageClass

### กำหนด Default StorageClass

```yaml
# default-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # กำหนดเป็น default
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

### เปลี่ยน Default StorageClass

```bash
# ดู StorageClass ปัจจุบัน
kubectl get storageclass
# NAME                 PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
# standard (default)   rancher.io/local-path          Delete          WaitForFirstConsumer   false               1h

# เปลี่ยน Default: unset ของเก่า
kubectl patch storageclass standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# ตั้ง Default ใหม่
kubectl patch storageclass new-standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# ตรวจสอบ
kubectl get storageclass
```

### PVC ที่ไม่ระบุ StorageClass

```yaml
# PVC ที่ไม่ระบุ storageClassName จะใช้ default
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-class-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # ไม่ระบุ storageClassName → ใช้ default
---
# PVC ที่ force ไม่ใช้ StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: no-class-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ""   # Empty string = ไม่ใช้ StorageClass
```

---

## 5. Volume Binding Mode

### Immediate Mode

```yaml
# BindPVC ทันทีที่ PVC ถูกสร้าง
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: immediate-binding
provisioner: rancher.io/local-path
volumeBindingMode: Immediate    # Default สำหรับ most cloud storage
```

**ข้อเสียของ Immediate:**
- อาจ provision storage ที่ Node ที่ Pod จะไม่ถูก schedule
- ทำให้เกิด Pod scheduling failures

### WaitForFirstConsumer Mode

```yaml
# รอจนกว่า Pod ถูก schedule ก่อน
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: delayed-binding
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer    # Recommended สำหรับ local storage
```

**ข้อดีของ WaitForFirstConsumer:**
- Storage จะถูก provision ที่ Node เดียวกับ Pod
- เหมาะสำหรับ topology-aware storage

---

## 6. Mount Options

```yaml
# mount-options-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-with-options
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.100
  share: /data
mountOptions:
- hard              # Hard mount (ไม่ยอมแพ้ถ้า server ไม่ตอบ)
- nfsvers=4.2      # NFS version
- intr              # Allow interrupt
- rsize=8192       # Read buffer size
- wsize=8192       # Write buffer size
- timeo=600        # Timeout
- retrans=2        # Retransmit count
```

---

## 7. Allowed Topologies

ควบคุมว่า Volume จะถูกสร้างใน Zone/Region ไหน

```yaml
# topology-aware-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: us-east-zones
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
    - us-east-1c
```

---

## 8. Workshop: สร้าง Custom StorageClass

### Workshop Overview
สร้าง StorageClass หลายประเภทสำหรับ workload ต่างๆ

### Prerequisites
- Minikube หรือ Kind cluster
- Helm installed
- เวลาประมาณ 60-90 นาที

### Step 1: ดู StorageClass ที่มีอยู่

```bash
# ดู StorageClass ทั้งหมด
kubectl get storageclass
kubectl get sc    # short form

# ดูรายละเอียด
kubectl describe storageclass standard

# ดู default StorageClass
kubectl get sc -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{"\n"}{end}'
```

### Step 2: ติดตั้ง Local Path Provisioner (สำหรับ development)

```bash
# ติดตั้ง Rancher Local Path Provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# ดู Provisioner
kubectl get pods -n local-path-storage
kubectl get storageclass local-path

# ดู ConfigMap ของ Provisioner
kubectl get configmap local-path-config -n local-path-storage -o yaml
```

### Step 3: สร้าง StorageClass หลายระดับ

```yaml
# storage-classes.yaml
---
# Tier 1: Fast SSD (สำหรับ Database)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  labels:
    tier: "1"
    performance: "high"
  annotations:
    description: "High performance SSD storage for databases"
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  nodePath: /mnt/fast-ssd    # Local path provisioner ใช้ nodePath

---
# Tier 2: Standard (สำหรับ Application data)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  labels:
    tier: "2"
    performance: "medium"
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
    description: "Standard storage for application data"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# Tier 3: Slow HDD (สำหรับ Logs/Archives)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
  labels:
    tier: "3"
    performance: "low"
  annotations:
    description: "Low-cost HDD storage for logs and archives"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  nodePath: /mnt/slow-hdd
```

```bash
kubectl apply -f storage-classes.yaml
kubectl get storageclass
```

### Step 4: สร้าง PVC สำหรับแต่ละ Tier

```yaml
# test-pvcs.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
  labels:
    tier: fast
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: standard-pvc
  labels:
    tier: standard
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: slow-pvc
  labels:
    tier: slow
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: slow
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f test-pvcs.yaml

# PVCs จะเป็น Pending เพราะ WaitForFirstConsumer
kubectl get pvc
```

### Step 5: Deploy Pods เพื่อ Trigger Binding

```yaml
# test-pods.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: fast-app
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Fast storage pod started"
      dd if=/dev/urandom of=/data/test-write bs=1M count=100 conv=fsync
      echo "Write test completed: $(ls -lh /data/test-write)"
      while true; do
        date >> /data/heartbeat.log
        sleep 30
      done
    volumeMounts:
    - name: fast-storage
      mountPath: /data
  volumes:
  - name: fast-storage
    persistentVolumeClaim:
      claimName: fast-pvc
---
apiVersion: v1
kind: Pod
metadata:
  name: standard-app
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Standard storage pod started"
      while true; do
        date >> /data/heartbeat.log
        sleep 30
      done
    volumeMounts:
    - name: standard-storage
      mountPath: /data
  volumes:
  - name: standard-storage
    persistentVolumeClaim:
      claimName: standard-pvc
---
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Slow storage pod started"
      while true; do
        date >> /data/heartbeat.log
        sleep 30
      done
    volumeMounts:
    - name: slow-storage
      mountPath: /data
  volumes:
  - name: slow-storage
    persistentVolumeClaim:
      claimName: slow-pvc
```

```bash
kubectl apply -f test-pods.yaml
kubectl get pods -w
```

### Step 6: ตรวจสอบ Dynamic Provisioning

```bash
# ดู PVCs หลัง Pods start
kubectl get pvc
# ทุก PVC ควร Bound แล้ว

# ดู PVs ที่ถูกสร้างอัตโนมัติ
kubectl get pv
kubectl describe pv <pv-name>

# ดู Storage Classes
kubectl get sc

# ดู Events ของ PVC
kubectl describe pvc fast-pvc | tail -20
```

### Step 7: ทดสอบ Volume Expansion

```bash
# ขยาย standard-pvc จาก 5Gi เป็น 10Gi
kubectl patch pvc standard-pvc -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# ดู progress
kubectl get pvc standard-pvc -w
kubectl describe pvc standard-pvc | grep -A 5 "Conditions:"

# ตรวจสอบ Pod ยังทำงานปกติ
kubectl exec -it standard-app -- df -h /data
```

### Step 8: ดู StorageClass ที่ Detail

```bash
# สร้าง script ดู StorageClass summary
cat << 'EOF' > check-sc.sh
#!/bin/bash
echo "=== StorageClass Summary ==="
echo ""
kubectl get storageclass -o custom-columns=\
'NAME:.metadata.name,\
PROVISIONER:.provisioner,\
RECLAIM:.reclaimPolicy,\
BINDING:.volumeBindingMode,\
EXPAND:.allowVolumeExpansion,\
DEFAULT:.metadata.annotations.storageclass\.kubernetes\.io/is-default-class'

echo ""
echo "=== PVCs per StorageClass ==="
for sc in $(kubectl get sc -o jsonpath='{.items[*].metadata.name}'); do
  count=$(kubectl get pvc -A --field-selector=spec.storageClassName=$sc --no-headers 2>/dev/null | wc -l)
  echo "$sc: $count PVCs"
done
EOF

chmod +x check-sc.sh
./check-sc.sh
```

### Step 9: สร้าง StorageClass สำหรับ NFS (Optional)

```bash
# ติดตั้ง NFS Subdir External Provisioner ด้วย Helm
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

# ติดตั้ง (ต้องมี NFS Server ก่อน)
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.1.100 \
  --set nfs.path=/exports/kubernetes \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false \
  --set storageClass.reclaimPolicy=Retain \
  --namespace nfs-provisioner \
  --create-namespace

# ดู provisioner
kubectl get pods -n nfs-provisioner
kubectl get storageclass nfs-client
```

### Step 10: ทำความสะอาด

```bash
kubectl delete pod fast-app standard-app slow-app
kubectl delete pvc fast-pvc standard-pvc slow-pvc
kubectl delete storageclass fast slow
# ไม่ลบ standard เพราะเป็น default
```

---

## 9. StorageClass สำหรับ Production

### 9.1 Multiple Tiers สำหรับ Production

```yaml
# production-storage-classes.yaml
---
# Database tier (highest performance)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-storage
  labels:
    tier: database
    sla: gold
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "64"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456:key/prod-key"
  csi.storage.k8s.io/fstype: xfs
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-east-1a
    - us-east-1b
---
# Application tier (balanced)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: application-storage
  labels:
    tier: application
    sla: silver
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Archive tier (lowest cost)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: archive-storage
  labels:
    tier: archive
    sla: bronze
provisioner: ebs.csi.aws.com
parameters:
  type: st1    # Throughput optimized HDD
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 9.2 StorageClass พร้อม Resource Quotas

```yaml
# namespace-storage-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    # จำกัด requests storage รวม
    requests.storage: "1Ti"
    
    # จำกัด PVC counts
    persistentvolumeclaims: "50"
    
    # จำกัดต่อ StorageClass
    database-storage.storageclass.storage.k8s.io/requests.storage: "500Gi"
    database-storage.storageclass.storage.k8s.io/persistentvolumeclaims: "10"
    
    application-storage.storageclass.storage.k8s.io/requests.storage: "500Gi"
    application-storage.storageclass.storage.k8s.io/persistentvolumeclaims: "40"
```

---

## 10. Monitoring StorageClass Usage

```bash
# ดู StorageClass usage
kubectl get pvc -A -o json | jq '
  .items | 
  group_by(.spec.storageClassName) | 
  map({
    storageClass: .[0].spec.storageClassName,
    count: length,
    totalSize: [.[].spec.resources.requests.storage] | join(", ")
  })'

# ดู PVs ที่ถูก provision โดยแต่ละ StorageClass
kubectl get pv -o json | jq -r '
  .items[] | 
  "\(.spec.storageClassName) - \(.metadata.name) - \(.spec.capacity.storage) - \(.status.phase)"
' | sort
```

---

## 11. StorageClass ที่มีกับ Cluster ต่างๆ

### Minikube
```bash
kubectl get sc
# NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION
# standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false
```

### Kind
```bash
kubectl get sc
# NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false
```

### EKS (AWS)
```bash
kubectl get sc
# NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false
```

### GKE (Google)
```bash
kubectl get sc
# NAME                     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# premium-rwo              pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true
# standard                 kubernetes.io/gce-pd    Delete          Immediate              true
# standard-rwo (default)   pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true
```

### AKS (Azure)
```bash
kubectl get sc
# NAME                    PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# azureblob-fuse-premium  blob.csi.azure.com   Delete          Immediate              true
# azureblob-nfs-premium   blob.csi.azure.com   Delete          Immediate              true
# azurefile               file.csi.azure.com   Delete          Immediate              true
# azurefile-csi           file.csi.azure.com   Delete          Immediate              true
# azurefile-csi-premium   file.csi.azure.com   Delete          Immediate              true
# azurefile-premium       file.csi.azure.com   Delete          Immediate              true
# default (default)       disk.csi.azure.com   Delete          WaitForFirstConsumer   true
# managed                 disk.csi.azure.com   Delete          WaitForFirstConsumer   true
# managed-csi             disk.csi.azure.com   Delete          WaitForFirstConsumer   true
# managed-csi-premium     disk.csi.azure.com   Delete          WaitForFirstConsumer   true
# managed-premium         disk.csi.azure.com   Delete          WaitForFirstConsumer   true
# ultra                   disk.csi.azure.com   Delete          WaitForFirstConsumer   true
```

---

## สรุป

| Field | คำอธิบาย | ตัวอย่าง |
|-------|----------|----------|
| provisioner | ผู้สร้าง PV | kubernetes.io/no-provisioner, ebs.csi.aws.com |
| parameters | Config ของ provisioner | type: gp3, iops: "3000" |
| reclaimPolicy | นโยบายเมื่อ PVC ถูกลบ | Delete, Retain |
| allowVolumeExpansion | อนุญาตขยาย PVC | true, false |
| volumeBindingMode | เมื่อไหร่ bind PV | Immediate, WaitForFirstConsumer |
| mountOptions | NFS/filesystem mount options | hard, nfsvers=4.1 |
| allowedTopologies | จำกัด zone/region | topology.kubernetes.io/zone |

### Key Takeaways:
1. **StorageClass** เป็น abstraction layer ระหว่าง Developer และ Storage Infrastructure
2. **Default StorageClass** ถูกใช้เมื่อ PVC ไม่ระบุ storageClassName
3. **WaitForFirstConsumer** สำคัญมากสำหรับ local storage เพื่อ topology awareness
4. **allowVolumeExpansion: true** จำเป็นสำหรับ workloads ที่ต้องการขยาย storage
5. ใช้ **หลาย StorageClass** สำหรับ workloads ต่างๆ (database vs application vs archive)
6. **Retain policy** สำหรับ production data สำคัญ, **Delete policy** สำหรับ temporary storage

## แหล่งเรียนรู้เพิ่มเติม

- [Storage Classes Documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Storage Best Practices](https://kubernetes.io/docs/concepts/storage/storage-limits/)
