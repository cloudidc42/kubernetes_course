# Part 42: PersistentVolume (PV) - การจัดการ Storage แบบถาวร

## บทนำ

ใน Part 41 เราได้เรียนรู้เกี่ยวกับ Volume พื้นฐานที่ผูกกับ Pod lifecycle ใน Part นี้เราจะเรียนรู้เกี่ยวกับ **PersistentVolume (PV)** ซึ่งเป็น storage resource ที่มีชีวิตอยู่เกิน Pod lifecycle ทำให้ข้อมูลไม่หายแม้ Pod จะถูกลบ

### ปัญหาที่ PersistentVolume แก้ไข

**โดยไม่มี PV:**
- ข้อมูลหายเมื่อ Pod ถูกลบ
- Developers ต้องรู้รายละเอียดของ storage infrastructure
- ไม่มีการแยก concerns ระหว่าง Dev และ Ops

**เมื่อมี PV:**
- ข้อมูล persist เกิน Pod lifecycle
- Developers ขอ storage ผ่าน PVC โดยไม่ต้องรู้รายละเอียด
- Ops/Admin จัดการ PV แยกต่างหาก

---

## 1. PersistentVolume คืออะไร

**PersistentVolume (PV)** คือ storage resource ใน Kubernetes cluster ที่ถูก provision ไว้ล่วงหน้าโดย Administrator หรือถูก provision แบบ dynamic โดย StorageClass

### คุณสมบัติของ PV:
- มีชีวิตอยู่เกิน Pod lifecycle
- เป็น cluster-level resource (ไม่ขึ้นกับ namespace)
- มี lifecycle เป็นของตัวเอง
- รองรับ storage หลายประเภท (local disk, NFS, cloud storage ฯลฯ)

### Relationship: PV, PVC, Pod

```
Administrator                Developer
     |                           |
     v                           v
Creates PV              Creates PVC (ขอ storage)
     |                           |
     +----------- Binding -------+
                     |
                     v
              Pod uses PVC
              (ผ่าน volumeMount)
```

---

## 2. PV Access Modes

Access Mode กำหนดว่า Volume สามารถ mount ได้กี่ Node และในลักษณะใด

### Access Modes ทั้งหมด

| Access Mode | ย่อ | คำอธิบาย |
|-------------|-----|-----------|
| `ReadWriteOnce` | RWO | Mount เป็น read-write ได้บน Node เดียว |
| `ReadOnlyMany` | ROX | Mount เป็น read-only ได้บนหลาย Nodes |
| `ReadWriteMany` | RWX | Mount เป็น read-write ได้บนหลาย Nodes |
| `ReadWriteOncePod` | RWOP | Mount เป็น read-write ได้โดย Pod เดียว (Kubernetes 1.22+) |

### ความสามารถของ Storage ต่างๆ

| Volume Plugin | RWO | ROX | RWX | RWOP |
|---------------|-----|-----|-----|------|
| Local | ✓ | - | - | ✓ |
| HostPath | ✓ | - | - | - |
| NFS | ✓ | ✓ | ✓ | - |
| AWS EBS | ✓ | - | - | ✓ |
| GCE PD | ✓ | ✓ | - | ✓ |
| Azure Disk | ✓ | - | - | ✓ |
| Azure File | ✓ | ✓ | ✓ | - |
| CephFS | ✓ | ✓ | ✓ | - |
| Ceph RBD | ✓ | ✓ | - | ✓ |

---

## 3. Reclaim Policies

Reclaim Policy กำหนดสิ่งที่จะเกิดขึ้นกับ PV หลังจาก PVC ที่ bind อยู่ถูกลบ

### Reclaim Policies ทั้งหมด

**1. Retain (เก็บไว้)**
- PV จะเปลี่ยน state เป็น `Released`
- ข้อมูลยังคงอยู่
- Admin ต้องมา reclaim PV ด้วยตนเอง
- เหมาะสำหรับ production data สำคัญ

**2. Delete (ลบ)**
- PV จะถูกลบ พร้อมกับข้อมูลใน underlying storage
- สำหรับ Cloud storage จะลบ EBS volume, GCE PD ฯลฯ ด้วย
- เหมาะสำหรับ dynamic provisioning

**3. Recycle (deprecated)**
- ทำ `rm -rf /thevolume/*` บน Volume
- Deprecated ใน Kubernetes 1.11+

```
PVC Deleted
     |
     +-- Retain Policy:
     |       PV state = Released
     |       Data preserved
     |       Admin must reclaim manually
     |
     +-- Delete Policy:
     |       PV deleted
     |       Underlying storage deleted
     |
     +-- Recycle Policy (deprecated):
             Data wiped
             PV available for new binding
```

---

## 4. PV Phases (States)

```
Available ──→ Bound ──→ Released ──→ Failed
                               │
                               └──→ (Admin recycles) ──→ Available
```

| Phase | ความหมาย |
|-------|----------|
| `Available` | ยังไม่ได้ bind กับ PVC ใด |
| `Bound` | Bind กับ PVC แล้ว |
| `Released` | PVC ถูกลบแล้ว แต่ resource ยังไม่ถูก reclaim |
| `Failed` | Automatic reclamation ล้มเหลว |

---

## 5. PV YAML - Manifests ต่างๆ

### 5.1 Local PV พื้นฐาน

```yaml
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-01
  labels:
    type: local
    environment: development
spec:
  capacity:
    storage: 10Gi            # ขนาด Storage
  
  volumeMode: Filesystem     # Filesystem หรือ Block
  
  accessModes:
  - ReadWriteOnce            # Access Mode
  
  persistentVolumeReclaimPolicy: Retain   # Reclaim Policy
  
  storageClassName: local-storage   # Storage Class
  
  local:
    path: /mnt/data/pv-01   # Path บน Node
  
  nodeAffinity:             # Local PV ต้องระบุ Node
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01           # Node name
```

### 5.2 NFS PV

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-01
  labels:
    type: nfs
    app: shared-storage
spec:
  capacity:
    storage: 50Gi
  
  volumeMode: Filesystem
  
  accessModes:
  - ReadWriteMany            # NFS รองรับ RWX
  - ReadOnlyMany
  
  persistentVolumeReclaimPolicy: Retain
  
  storageClassName: nfs
  
  mountOptions:             # NFS mount options
  - hard
  - nfsvers=4.1
  
  nfs:
    path: /exports/kubernetes    # NFS export path
    server: 192.168.1.100        # NFS server IP
    readOnly: false
```

### 5.3 HostPath PV (สำหรับ Development)

```yaml
# hostpath-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-01
  labels:
    type: hostpath
    environment: dev
spec:
  capacity:
    storage: 5Gi
  
  volumeMode: Filesystem
  
  accessModes:
  - ReadWriteOnce
  
  persistentVolumeReclaimPolicy: Retain
  
  storageClassName: hostpath-local
  
  hostPath:
    path: /var/kubernetes/pv-data/pv-01
    type: DirectoryOrCreate
```

### 5.4 AWS EBS PV (Legacy - ก่อน CSI)

```yaml
# aws-ebs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aws-ebs-pv
  labels:
    type: aws-ebs
spec:
  capacity:
    storage: 100Gi
  
  volumeMode: Filesystem
  
  accessModes:
  - ReadWriteOnce            # EBS รองรับเฉพาะ RWO
  
  persistentVolumeReclaimPolicy: Delete
  
  storageClassName: aws-ebs-gp3
  
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0   # EBS Volume ID
    fsType: ext4
    partition: 0
```

### 5.5 CSI PV (Modern approach)

```yaml
# csi-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: csi-pv-01
  labels:
    type: csi
spec:
  capacity:
    storage: 100Gi
  
  volumeMode: Filesystem
  
  accessModes:
  - ReadWriteOnce
  
  persistentVolumeReclaimPolicy: Delete
  
  storageClassName: ebs-sc
  
  csi:
    driver: ebs.csi.aws.com        # CSI Driver name
    volumeHandle: vol-0123456789   # Volume identifier
    fsType: ext4
    volumeAttributes:
      storage.kubernetes.io/csiProvisionerIdentity: "1234567890"
```

### 5.6 Block Volume (Raw Block)

```yaml
# block-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv-01
spec:
  capacity:
    storage: 50Gi
  
  volumeMode: Block            # Raw block device
  
  accessModes:
  - ReadWriteOnce
  
  persistentVolumeReclaimPolicy: Retain
  
  storageClassName: local-block
  
  local:
    path: /dev/sdb             # Raw block device
  
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - storage-node-01
```

---

## 6. PV Labels และ Selectors

การใช้ Labels ช่วยให้ PVC เลือก PV ที่เหมาะสมได้

```yaml
# pv-with-labels.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ssd-01
  labels:
    storage-type: ssd
    environment: production
    tier: database
    region: us-east-1a
spec:
  capacity:
    storage: 200Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  local:
    path: /mnt/ssd/pv-ssd-01
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: storage-type
          operator: In
          values:
          - ssd
```

---

## 7. Multiple PVs สำหรับ Production

```yaml
# production-pvs.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgres-data
  labels:
    app: postgres
    type: data
spec:
  capacity:
    storage: 500Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ssd-storage
  local:
    path: /mnt/ssd/postgres/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - db-node-01
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgres-wal
  labels:
    app: postgres
    type: wal
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ssd-storage
  local:
    path: /mnt/ssd/postgres/wal
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - db-node-01
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgres-backup
  labels:
    app: postgres
    type: backup
spec:
  capacity:
    storage: 2Ti
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: hdd-storage
  nfs:
    path: /backups/postgres
    server: 10.0.0.100
```

---

## 8. PV Capacity Planning

```yaml
# ตัวอย่าง PV Pool สำหรับ different workloads
---
# Small PVs สำหรับ development
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-small-01
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Delete
  storageClassName: standard
  hostPath:
    path: /mnt/pvs/small/01
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-small-02
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Delete
  storageClassName: standard
  hostPath:
    path: /mnt/pvs/small/02
---
# Medium PVs สำหรับ application data
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-medium-01
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/pvs/medium/01
---
# Large PVs สำหรับ databases
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-large-01
spec:
  capacity:
    storage: 100Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast
  local:
    path: /mnt/ssd/pvs/large/01
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - storage-node
```

---

## 9. StorageClass ใน PV

```yaml
# pv-with-storageclass.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-standard
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: "standard"    # ต้อง match กับ StorageClass name
  hostPath:
    path: /data/pv-standard
```

**หมายเหตุ:** ถ้า PV ไม่ได้ระบุ `storageClassName` หรือระบุเป็น `""` จะถือว่าเป็น PV ที่ไม่มี StorageClass และจะ bind เฉพาะกับ PVC ที่ไม่มี StorageClass ด้วย

---

## 10. Workshop: สร้าง Local PersistentVolume

### Workshop Overview
สร้าง Local PV และทดสอบการ bind กับ PVC บน Minikube หรือ Kind cluster

### Prerequisites
- Minikube หรือ Kind cluster ที่ทำงานได้
- kubectl configured
- เวลาประมาณ 45-60 นาที

### Step 1: เตรียม Cluster และ Directories

**สำหรับ Minikube:**
```bash
# SSH เข้า Minikube node
minikube ssh

# สร้าง directories สำหรับ PVs
sudo mkdir -p /mnt/data/pv-01
sudo mkdir -p /mnt/data/pv-02
sudo mkdir -p /mnt/data/pv-03
sudo chmod 777 /mnt/data/pv-01
sudo chmod 777 /mnt/data/pv-02
sudo chmod 777 /mnt/data/pv-03
ls -la /mnt/data/

# ออกจาก SSH
exit
```

**สำหรับ Kind:**
```bash
# ดู Node name
kubectl get nodes

# Exec เข้า Kind node container
docker exec -it kind-control-plane bash

# สร้าง directories
mkdir -p /mnt/data/pv-01 /mnt/data/pv-02 /mnt/data/pv-03
chmod 777 /mnt/data/pv-01 /mnt/data/pv-02 /mnt/data/pv-03
ls -la /mnt/data/

# ออกจาก container
exit
```

### Step 2: สร้าง StorageClass สำหรับ Local Storage

```yaml
# local-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer   # สำคัญสำหรับ local storage
reclaimPolicy: Retain
allowVolumeExpansion: false
```

```bash
kubectl apply -f local-storageclass.yaml
kubectl get storageclass
```

### Step 3: สร้าง PersistentVolumes

```yaml
# local-pvs.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-01
  labels:
    type: local
    size: small
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data/pv-01
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube   # หรือ kind-control-plane
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-02
  labels:
    type: local
    size: medium
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data/pv-02
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-03
  labels:
    type: local
    size: large
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/data/pv-03
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
```

```bash
# ดู Node hostname ก่อน
kubectl get nodes -o wide
kubectl get nodes --show-labels | grep hostname

# แก้ไข hostname ใน YAML ให้ถูกต้อง
# แล้ว apply
kubectl apply -f local-pvs.yaml
```

### Step 4: ตรวจสอบ PV Status

```bash
# ดู PV ทั้งหมด
kubectl get pv

# ดูรายละเอียด PV
kubectl describe pv local-pv-01
kubectl describe pv local-pv-02
kubectl describe pv local-pv-03

# ดู PV ในรูปแบบ wide
kubectl get pv -o wide

# ดู PV phases
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

**Output ที่ควรได้:**
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
local-pv-01   1Gi        RWO            Retain           Available           local-storage            1m
local-pv-02   5Gi        RWO            Retain           Available           local-storage            1m
local-pv-03   10Gi       RWO,ROX        Delete           Available           local-storage            1m
```

### Step 5: สร้าง PVC เพื่อทดสอบ Binding

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc-small
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi     # ขอ 500Mi จาก PV ที่มี 1Gi
  storageClassName: local-storage
```

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc

# PVC จะเป็น Pending เพราะ WaitForFirstConsumer
# ต้องสร้าง Pod ที่ใช้ PVC ก่อน
```

### Step 6: สร้าง Pod ที่ใช้ PVC

```yaml
# test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pv-pod
  namespace: default
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Pod started at: $(date)"
      echo "Writing to PV..."
      echo "Hello from Pod $(hostname)!" > /data/hello.txt
      echo "Timestamp: $(date)" >> /data/hello.txt
      
      while true; do
        echo "$(date): Alive" >> /data/heartbeat.log
        ls -la /data/
        sleep 30
      done
    volumeMounts:
    - name: storage
      mountPath: /data
  
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc-small
```

```bash
kubectl apply -f test-pod.yaml
kubectl get pod test-pv-pod -w
```

### Step 7: ตรวจสอบ Binding

```bash
# ดู PVC status (ควรเป็น Bound แล้ว)
kubectl get pvc
kubectl describe pvc test-pvc-small

# ดู PV status (ควรเป็น Bound แล้ว)
kubectl get pv local-pv-01
kubectl describe pv local-pv-01

# ดูข้อมูลใน PV จาก Pod
kubectl exec -it test-pv-pod -- cat /data/hello.txt
kubectl exec -it test-pv-pod -- ls -la /data/
```

### Step 8: ทดสอบ Data Persistence

```bash
# เขียนข้อมูลสำคัญ
kubectl exec -it test-pv-pod -- \
  /bin/sh -c 'echo "Important data that must persist!" > /data/important.txt'

# ลบ Pod
kubectl delete pod test-pv-pod

# สร้าง Pod ใหม่ที่ใช้ PVC เดิม
kubectl apply -f test-pod.yaml

# รอ Pod start
kubectl get pod test-pv-pod -w

# ตรวจสอบว่าข้อมูลยังอยู่
kubectl exec -it test-pv-pod -- cat /data/important.txt
kubectl exec -it test-pv-pod -- cat /data/hello.txt
```

**ผลลัพธ์ที่ควรได้:** ข้อมูลยังอยู่ครบ เพราะ PV มี Retain policy

### Step 9: ทดสอบ Reclaim Policy

```bash
# ลบ PVC (แต่ข้อมูลยังอยู่เพราะ Retain policy)
kubectl delete pod test-pv-pod
kubectl delete pvc test-pvc-small

# ดู PV status (ควรเป็น Released ไม่ใช่ Available)
kubectl get pv local-pv-01

# PV จะไม่สามารถ bind กับ PVC ใหม่ได้จนกว่า Admin จะ reclaim

# วิธี reclaim PV ด้วยตนเอง:
# 1. ลบ PV เก่า (ข้อมูลยังอยู่บน Node)
kubectl delete pv local-pv-01

# 2. สร้าง PV ใหม่ที่ชี้ไปที่ path เดิม (ข้อมูลยังอยู่)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-01
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data/pv-01
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
EOF

# PV ควรเป็น Available อีกครั้ง
kubectl get pv local-pv-01
```

### Step 10: ตรวจสอบข้อมูลโดยตรงบน Node

```bash
# สำหรับ Minikube
minikube ssh
ls -la /mnt/data/pv-01/
cat /mnt/data/pv-01/important.txt
cat /mnt/data/pv-01/hello.txt
exit

# สำหรับ Kind
docker exec kind-control-plane ls -la /mnt/data/pv-01/
docker exec kind-control-plane cat /mnt/data/pv-01/important.txt
```

### Step 11: ทำความสะอาด

```bash
# ลบ resources ทั้งหมด
kubectl delete pod test-pv-pod 2>/dev/null
kubectl delete pvc test-pvc-small 2>/dev/null
kubectl delete pv local-pv-01 local-pv-02 local-pv-03
kubectl delete storageclass local-storage
```

---

## 11. PV Monitoring และ Management

### ดู PV สถานะทั้งหมด

```bash
# ดู PV ทั้งหมดพร้อม status
kubectl get pv --sort-by=.spec.capacity.storage

# ดู PV ที่ Available
kubectl get pv -o json | jq '.items[] | select(.status.phase == "Available") | .metadata.name'

# ดู PV ที่ Released (รอ reclaim)
kubectl get pv -o json | jq '.items[] | select(.status.phase == "Released") | {name: .metadata.name, capacity: .spec.capacity.storage}'

# ดู PV usage summary
kubectl get pv -o custom-columns=\
'NAME:.metadata.name,\
CAPACITY:.spec.capacity.storage,\
ACCESS:.spec.accessModes[0],\
RECLAIM:.spec.persistentVolumeReclaimPolicy,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name'
```

### Script ตรวจสอบ PV Health

```bash
#!/bin/bash
# pv-health-check.sh

echo "=== PersistentVolume Health Report ==="
echo "Generated: $(date)"
echo ""

echo "--- Summary ---"
kubectl get pv --no-headers | awk '{print $5}' | sort | uniq -c

echo ""
echo "--- Available PVs ---"
kubectl get pv --no-headers | grep "Available"

echo ""
echo "--- Released PVs (need reclaim) ---"
kubectl get pv --no-headers | grep "Released"

echo ""
echo "--- Failed PVs ---"
kubectl get pv --no-headers | grep "Failed"

echo ""
echo "--- Bound PVs ---"
kubectl get pv --no-headers | grep "Bound"
```

```bash
chmod +x pv-health-check.sh
./pv-health-check.sh
```

---

## 12. PV Security Considerations

### การ Protect PV จากการลบโดยไม่ตั้งใจ

```bash
# เพิ่ม Finalizer ป้องกันการลบ
kubectl patch pv local-pv-01 -p '{"metadata":{"finalizers":["kubernetes.io/pv-protection"]}}'

# ดู Finalizers
kubectl get pv local-pv-01 -o jsonpath='{.metadata.finalizers}'
```

### PV Encryption

```yaml
# สำหรับ AWS EBS ที่เข้ารหัส
apiVersion: v1
kind: PersistentVolume
metadata:
  name: encrypted-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0encrypted123
    fsType: ext4
    volumeAttributes:
      encrypted: "true"
      kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/my-key"
```

---

## 13. Advanced PV Configurations

### VolumeAttributesClass (Kubernetes 1.29+)

```yaml
# ปรับแต่ง volume attributes
apiVersion: storage.k8s.io/v1alpha1
kind: VolumeAttributesClass
metadata:
  name: silver
driverName: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
```

### PV ที่มี Mount Options

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-with-options
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
  - hard
  - nfsvers=4.1
  - rsize=8192
  - wsize=8192
  - timeo=600
  - retrans=2
  nfs:
    server: 192.168.1.10
    path: /data/shared
```

---

## 14. ความแตกต่างระหว่าง Static และ Dynamic Provisioning

### Static Provisioning (ที่เรียนไปแล้ว)
```
Admin creates PV manually
    |
    v
Developer creates PVC
    |
    v
Kubernetes matches PV to PVC
    |
    v
Pod uses PVC
```

### Dynamic Provisioning (จะเรียนใน Part 45)
```
Admin creates StorageClass
    |
    v
Developer creates PVC with storageClassName
    |
    v
Kubernetes auto-creates PV via StorageClass provisioner
    |
    v
Pod uses PVC
```

---

## สรุป

| Feature | คำอธิบาย |
|---------|----------|
| PV | Cluster-level storage resource ที่ Admin สร้าง |
| Namespace | ไม่ขึ้นกับ namespace (cluster-wide) |
| Access Modes | RWO, ROX, RWX, RWOP |
| Reclaim Policy | Retain, Delete, Recycle (deprecated) |
| Phases | Available, Bound, Released, Failed |
| StorageClass | กำหนด class ของ storage |

### Key Takeaways:
1. **PV** มีชีวิตอยู่เกิน Pod lifecycle - เหมาะสำหรับ stateful applications
2. **Access Modes** ขึ้นอยู่กับ underlying storage technology
3. **Retain Policy** สำหรับ production data - ป้องกันการลบโดยไม่ตั้งใจ
4. **Delete Policy** สำหรับ dynamic provisioning - cloud storage
5. **Local PV** ต้องระบุ **nodeAffinity** เสมอ
6. **WaitForFirstConsumer** binding mode สำคัญสำหรับ local storage

## แหล่งเรียนรู้เพิ่มเติม

- [Kubernetes PersistentVolumes Documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Configure a Pod to Use a PersistentVolume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
