# Part 45: Dynamic Provisioning - การ Provision Storage อัตโนมัติ

## บทนำ

**Dynamic Provisioning** คือความสามารถของ Kubernetes ในการสร้าง PersistentVolume โดยอัตโนมัติเมื่อมีการสร้าง PVC โดยไม่จำเป็นต้องให้ Admin สร้าง PV ล่วงหน้า ซึ่งช่วยลดงานของ Admin และเพิ่ม agility ให้กับ development team

### การเปรียบเทียบ Static vs Dynamic Provisioning

**Static Provisioning:**
```
Admin: สร้าง PV ล่วงหน้า (10 x 10Gi, 5 x 100Gi...)
Developer: สร้าง PVC ขอ 10Gi
Kubernetes: หา PV ที่ match แล้ว bind
```

**Dynamic Provisioning:**
```
Admin: สร้าง StorageClass ครั้งเดียว
Developer: สร้าง PVC ขอ 10Gi พร้อม StorageClass
Kubernetes: ให้ Provisioner สร้าง PV 10Gi ใหม่อัตโนมัติ
```

---

## 1. Dynamic Provisioning คืออะไร

### Architecture ของ Dynamic Provisioning

```
Developer creates PVC
        |
        v
Kubernetes API Server
        |
        v
    ┌───────────────────────────────────────┐
    │   Kubernetes Controller Manager       │
    │                                       │
    │  PersistentVolume Controller:         │
    │  - Watch PVC events                   │
    │  - Find matching StorageClass         │
    │  - Call provisioner plugin            │
    └───────────────────────────────────────┘
        |
        v
    Provisioner Plugin
    (internal or external)
        |
        v
    Storage Backend
    (Cloud API, NFS, etc.)
        |
        v
    PV Created Automatically
        |
        v
    PVC Bound to PV
        |
        v
    Pod can use PVC
```

### เงื่อนไขสำหรับ Dynamic Provisioning

1. **StorageClass ต้องมี provisioner** ที่ไม่ใช่ `kubernetes.io/no-provisioner`
2. **PVC ต้องระบุ storageClassName** ที่ชี้ไปยัง StorageClass นั้น (หรือใช้ default)
3. **Provisioner ต้องทำงานอยู่** ใน cluster (สำหรับ external provisioners)
4. **Permissions ต้องถูกต้อง** สำหรับ provisioner ในการสร้าง storage

---

## 2. Cloud Provisioners

### 2.1 AWS EBS Dynamic Provisioning

**ติดตั้ง AWS EBS CSI Driver:**

```bash
# ติดตั้งด้วย Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole

# ตรวจสอบ
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

**StorageClass สำหรับ AWS EBS:**

```yaml
# aws-storage-classes.yaml
---
# GP3 (แนะนำสำหรับ production)
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
# IO2 (สำหรับ high-performance database)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-io2-database
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "64"
  encrypted: "true"
  csi.storage.k8s.io/fstype: xfs
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
---
# SC1 (Cold storage สำหรับ archives)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc1-cold
provisioner: ebs.csi.aws.com
parameters:
  type: sc1
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

**PVC ที่ใช้ AWS EBS Dynamic Provisioning:**

```yaml
# aws-pvc-example.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 100Gi
```

### 2.2 GCP Persistent Disk

**StorageClass สำหรับ GCP:**

```yaml
# gcp-storage-classes.yaml
---
# SSD Persistent Disk (default สำหรับ GKE)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
  csi.storage.k8s.io/fstype: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# Balanced Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: balanced-rwo
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# Regional Persistent Disk (HA)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
  csi.storage.k8s.io/fstype: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
allowedTopologies:
- matchLabelExpressions:
  - key: topology.gke.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

### 2.3 Azure Disk

```yaml
# azure-storage-classes.yaml
---
# Premium SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
  cachingmode: ReadOnly
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# Ultra Disk (สำหรับ database)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ultra-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: UltraSSD_LRS
  diskIOPSReadWrite: "4000"
  diskMBpsReadWrite: "1000"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

---

## 3. Local Dynamic Provisioning

### 3.1 Rancher Local Path Provisioner

Local Path Provisioner สร้าง directories บน Node filesystem ให้อัตโนมัติ

**ติดตั้ง:**

```bash
# ติดตั้ง Local Path Provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# ตรวจสอบ
kubectl get pods -n local-path-storage
kubectl get storageclass local-path
```

**ปรับแต่ง Config:**

```yaml
# local-path-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |
    {
      "nodePathMap": [
        {
          "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths": ["/opt/local-path-provisioner"]
        },
        {
          "node": "storage-node-01",
          "paths": ["/mnt/ssd/local-path"]
        },
        {
          "node": "storage-node-02",
          "paths": ["/mnt/hdd/local-path"]
        }
      ]
    }
  
  setup: |
    #!/bin/sh
    set -eu
    mkdir -m 0777 -p "$VOL_DIR"
  
  teardown: |
    #!/bin/sh
    set -eu
    rm -rf "$VOL_DIR"
  
  helperPod.yaml: |
    apiVersion: v1
    kind: Pod
    spec:
      priorityClassName: system-node-critical
      tolerations:
      - key: node.kubernetes.io/disk-pressure
        operator: Tolerate
        effect: NoSchedule
      containers:
      - name: helper
        image: busybox
        imagePullPolicy: IfNotPresent
```

```bash
kubectl apply -f local-path-config.yaml
kubectl rollout restart deployment -n local-path-storage
```

**StorageClass สำหรับ Local Path:**

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

### 3.2 OpenEBS

OpenEBS เป็น Container Native Storage solution ที่ใช้ Node storage

**ติดตั้ง OpenEBS:**

```bash
# ติดตั้งด้วย Helm
helm repo add openebs https://openebs.github.io/charts
helm repo update

helm install openebs openebs/openebs \
  --namespace openebs \
  --create-namespace \
  --set engines.replicated.mayastor.enabled=false

# ตรวจสอบ
kubectl get pods -n openebs
kubectl get storageclass
```

**OpenEBS StorageClasses:**

```yaml
# openebs-storage-classes.yaml
---
# Local HostPath (สำหรับ local dev)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-hostpath
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  storageType: hostpath
  basePath: /var/openebs/local
---
# Local Device (สำหรับ raw device)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-device
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  storageType: device
  blockDeviceSelectors:
    openebs.io/block-device-tag: database
```

### 3.3 Longhorn

Longhorn เป็น distributed block storage system สำหรับ Kubernetes

**ติดตั้ง Longhorn:**

```bash
# Check prerequisites
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/prerequisite/longhorn-iscsi-installation.yaml
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/prerequisite/longhorn-nfs-installation.yaml

# ติดตั้ง Longhorn
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml

# ตรวจสอบ
kubectl get pods -n longhorn-system
kubectl get storageclass
```

**Longhorn StorageClass:**

```yaml
# longhorn-storage-classes.yaml
---
# Default Longhorn StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"          # Replicas สำหรับ HA
  staleReplicaTimeout: "2880"    # นาที
  fromBackup: ""
  fsType: "ext4"
---
# High Performance Longhorn
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-high-performance
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  numberOfReplicas: "2"
  diskSelector: "ssd"            # เลือกเฉพาะ SSD disks
  nodeSelector: "storage"        # เลือกเฉพาะ storage nodes
  fsType: "xfs"
```

---

## 4. Workshop: Auto-provision Storage

### Workshop Overview
ทดลอง Dynamic Provisioning ด้วย Local Path Provisioner และทดสอบ auto-provision

### Prerequisites
- Minikube หรือ Kind cluster
- เวลาประมาณ 60-90 นาที

### Step 1: เตรียม Environment

```bash
# สร้าง namespace
kubectl create namespace dynamic-provisioning-workshop
kubectl config set-context --current --namespace=dynamic-provisioning-workshop

# ดู StorageClass ที่มีอยู่
kubectl get storageclass
```

### Step 2: ติดตั้ง Local Path Provisioner (ถ้ายังไม่มี)

```bash
# ตรวจสอบว่ามี local-path provisioner หรือยัง
kubectl get storageclass | grep local-path

# ถ้ายังไม่มี ติดตั้ง
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# รอ provisioner start
kubectl wait --for=condition=available deployment/local-path-provisioner \
  -n local-path-storage --timeout=120s

# ดู
kubectl get pods -n local-path-storage
kubectl get storageclass local-path
```

### Step 3: สร้าง Multiple StorageClasses

```yaml
# workshop-storage-classes.yaml
---
# Fast tier
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: workshop-fast
  labels:
    workshop: "true"
    tier: fast
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Standard tier (default)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: workshop-standard
  labels:
    workshop: "true"
    tier: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Retain tier (สำหรับ important data)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: workshop-retain
  labels:
    workshop: "true"
    tier: retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain           # ข้อมูลไม่ถูกลบเมื่อ PVC ลบ
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f workshop-storage-classes.yaml
kubectl get storageclass -l workshop=true
```

### Step 4: ทดสอบ Auto-provision ด้วย PVC

```yaml
# workshop-pvcs.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
  namespace: dynamic-provisioning-workshop
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: workshop-fast
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: standard-pvc
  namespace: dynamic-provisioning-workshop
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: workshop-standard
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: retain-pvc
  namespace: dynamic-provisioning-workshop
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: workshop-retain
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f workshop-pvcs.yaml

# ดู PVC status
kubectl get pvc -n dynamic-provisioning-workshop

# PVC จะเป็น Pending เพราะ WaitForFirstConsumer
# ต้องสร้าง Pod ก่อน
```

### Step 5: Deploy Applications ที่ใช้ PVC ต่างๆ

```yaml
# workshop-apps.yaml
---
# App ที่ใช้ fast storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fast-app
  namespace: dynamic-provisioning-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fast-app
  template:
    metadata:
      labels:
        app: fast-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Fast app started at $(date)"
          echo "Testing write speed..."
          dd if=/dev/urandom of=/data/speedtest.dat bs=1M count=50 oflag=dsync 2>&1
          echo "Write test done"
          while true; do
            date >> /data/heartbeat.log
            sleep 60
          done
        volumeMounts:
        - name: fast-storage
          mountPath: /data
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 128Mi
      volumes:
      - name: fast-storage
        persistentVolumeClaim:
          claimName: fast-pvc
---
# App ที่ใช้ standard storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: standard-app
  namespace: dynamic-provisioning-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: standard-app
  template:
    metadata:
      labels:
        app: standard-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Standard app started at $(date)"
          while true; do
            echo "$(date): writing data" >> /data/app.log
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
# App ที่ใช้ retain storage (important data)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: important-app
  namespace: dynamic-provisioning-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: important-app
  template:
    metadata:
      labels:
        app: important-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Important app started at $(date)"
          echo "This data is VERY IMPORTANT" > /data/critical.txt
          echo "Saved at: $(date)" >> /data/critical.txt
          while true; do
            echo "$(date): Heartbeat" >> /data/activity.log
            sleep 30
          done
        volumeMounts:
        - name: retain-storage
          mountPath: /data
      volumes:
      - name: retain-storage
        persistentVolumeClaim:
          claimName: retain-pvc
```

```bash
kubectl apply -f workshop-apps.yaml
kubectl get pods -n dynamic-provisioning-workshop -w
```

### Step 6: ตรวจสอบ Dynamic Provisioning

```bash
# ดู PVs ที่ถูกสร้างอัตโนมัติ
kubectl get pv

# ดู PVCs
kubectl get pvc -n dynamic-provisioning-workshop

# ดูรายละเอียด PV ที่ถูก provision
kubectl describe pv $(kubectl get pvc fast-pvc -n dynamic-provisioning-workshop -o jsonpath='{.spec.volumeName}')

# ดู Events ของ PVC (จะเห็นว่า Provisioner สร้าง PV)
kubectl describe pvc fast-pvc -n dynamic-provisioning-workshop | grep -A 20 "Events:"
```

**Output ที่ควรเห็น:**
```
Events:
  Normal  ExternalProvisioning  Waiting for a volume to be created, either by external provisioner "rancher.io/local-path" or manually created by system administrator
  Normal  Provisioning          External provisioner is provisioning volume for claim "dynamic-provisioning-workshop/fast-pvc"
  Normal  ProvisioningSucceeded Volume was successfully provisioned...
  Normal  Bound                 Successfully bound to PV
```

### Step 7: ดูข้อมูลที่ถูกเขียน

```bash
# ดู fast app logs
kubectl logs deployment/fast-app -n dynamic-provisioning-workshop

# ดูข้อมูลใน fast storage
kubectl exec -it deployment/fast-app -n dynamic-provisioning-workshop -- \
  ls -lh /data/

# ดู important data
kubectl exec -it deployment/important-app -n dynamic-provisioning-workshop -- \
  cat /data/critical.txt
```

### Step 8: ทดสอบ Data Persistence หลัง Pod restart

```bash
# บันทึก data
kubectl exec -it deployment/fast-app -n dynamic-provisioning-workshop -- \
  /bin/sh -c 'echo "PERSIST THIS: $(date)" > /data/must-persist.txt'

# ลบ Pod (Deployment จะสร้างใหม่)
kubectl rollout restart deployment/fast-app -n dynamic-provisioning-workshop

# รอ Pod ใหม่
kubectl get pods -n dynamic-provisioning-workshop -w

# ตรวจสอบข้อมูล
kubectl exec -it deployment/fast-app -n dynamic-provisioning-workshop -- \
  cat /data/must-persist.txt
```

### Step 9: ทดสอบ Reclaim Policy

```bash
# เพิ่มข้อมูลสำคัญใน retain storage
kubectl exec -it deployment/important-app -n dynamic-provisioning-workshop -- \
  /bin/sh -c 'echo "CRITICAL: This must not be deleted!" >> /data/critical.txt'

# จดบันทึก PV name
RETAIN_PV=$(kubectl get pvc retain-pvc -n dynamic-provisioning-workshop -o jsonpath='{.spec.volumeName}')
echo "Retain PV: $RETAIN_PV"

# ลบ Deployment และ PVC
kubectl delete deployment important-app -n dynamic-provisioning-workshop
kubectl delete pvc retain-pvc -n dynamic-provisioning-workshop

# ดู PV status (ควรเป็น Released ไม่ใช่ Deleted)
kubectl get pv $RETAIN_PV

# สำหรับ Fast/Standard PVC (Delete policy)
FAST_PV=$(kubectl get pvc fast-pvc -n dynamic-provisioning-workshop -o jsonpath='{.spec.volumeName}')
kubectl delete deployment fast-app -n dynamic-provisioning-workshop
kubectl delete pvc fast-pvc -n dynamic-provisioning-workshop

# ดู PV status (ควรถูกลบไปแล้ว)
kubectl get pv $FAST_PV 2>/dev/null || echo "PV was deleted (expected for Delete policy)"
```

### Step 10: Dynamic Provisioning Scale Test

```yaml
# scale-test.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: scale-test
  namespace: dynamic-provisioning-workshop
spec:
  serviceName: scale-test
  replicas: 3
  selector:
    matchLabels:
      app: scale-test
  template:
    metadata:
      labels:
        app: scale-test
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          POD_INDEX=${HOSTNAME##*-}
          echo "Pod $HOSTNAME started - Index: $POD_INDEX"
          echo "My data: Pod-$HOSTNAME-$(date)" > /data/pod-info.txt
          while true; do
            date >> /data/activity.log
            sleep 30
          done
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: workshop-standard
      resources:
        requests:
          storage: 500Mi
```

```bash
kubectl apply -f scale-test.yaml

# ดู Pods และ PVCs ถูกสร้างอัตโนมัติ
kubectl get pods,pvc -n dynamic-provisioning-workshop -w

# ดู PVs ที่ถูกสร้าง
kubectl get pv | grep scale-test

# ดู data ในแต่ละ Pod
for i in 0 1 2; do
  echo "=== scale-test-$i ==="
  kubectl exec scale-test-$i -n dynamic-provisioning-workshop -- cat /data/pod-info.txt
done

# Scale up
kubectl scale statefulset scale-test -n dynamic-provisioning-workshop --replicas=5
kubectl get pods,pvc -n dynamic-provisioning-workshop -w

# ดู PVs ใหม่ที่ถูกสร้าง
kubectl get pv | grep scale-test
```

### Step 11: Volume Expansion Test

```bash
# ดู PVC ปัจจุบัน
kubectl get pvc standard-pvc -n dynamic-provisioning-workshop

# ขยาย PVC
kubectl patch pvc standard-pvc -n dynamic-provisioning-workshop \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# ดู progress
kubectl get pvc standard-pvc -n dynamic-provisioning-workshop -w

# ดูใน Pod
kubectl exec -it deployment/standard-app -n dynamic-provisioning-workshop -- df -h /data
```

### Step 12: ทำความสะอาด

```bash
kubectl delete namespace dynamic-provisioning-workshop

# ลบ StorageClasses ที่สร้างใน workshop
kubectl delete storageclass workshop-fast workshop-standard workshop-retain

# ดู PVs ที่ยังเหลือ (จาก Retain policy)
kubectl get pv
# ลบ PVs ที่เหลือ (ถ้าต้องการ)
kubectl delete pv <retained-pv-name>
```

---

## 5. Dynamic Provisioning Best Practices

### 5.1 StorageClass Design

```yaml
# best-practice-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-standard
  labels:
    environment: production
    managed-by: platform-team
  annotations:
    description: "Standard production storage - gp3 EBS, 3000 IOPS"
    contact: "platform-team@company.com"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 5.2 Namespace-level Defaults

```yaml
# storage-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: my-app
spec:
  hard:
    requests.storage: "500Gi"
    persistentvolumeclaims: "20"
    production-standard.storageclass.storage.k8s.io/requests.storage: "500Gi"
    production-standard.storageclass.storage.k8s.io/persistentvolumeclaims: "20"
---
# LimitRange สำหรับ PVC default size
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: my-app
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 100Gi
    min:
      storage: 1Gi
```

### 5.3 Monitoring

```bash
# Script ตรวจสอบ Dynamic Provisioning health
cat << 'EOF' > check-dynamic-provisioning.sh
#!/bin/bash

echo "=== Dynamic Provisioning Health Check ==="
echo "Timestamp: $(date)"
echo ""

echo "--- StorageClasses ---"
kubectl get storageclass -o custom-columns='NAME:.metadata.name,PROVISIONER:.provisioner,DEFAULT:.metadata.annotations.storageclass\.kubernetes\.io/is-default-class'

echo ""
echo "--- Pending PVCs (may indicate provisioning issues) ---"
kubectl get pvc -A --field-selector=status.phase=Pending

echo ""
echo "--- Recent Events related to PVC provisioning ---"
kubectl get events -A --field-selector=reason=ProvisioningSucceeded --sort-by='.lastTimestamp' | tail -10
kubectl get events -A --field-selector=reason=ProvisioningFailed --sort-by='.lastTimestamp' | tail -5

echo ""
echo "--- PV Summary ---"
kubectl get pv -o custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase,CLAIM:.spec.claimRef.namespace,SC:.spec.storageClassName' | sort

EOF

chmod +x check-dynamic-provisioning.sh
./check-dynamic-provisioning.sh
```

---

## 6. Troubleshooting Dynamic Provisioning

### ปัญหาที่พบบ่อย

**1. PVC ค้างอยู่ที่ Pending ตลอดเวลา**

```bash
# ดู Events ของ PVC
kubectl describe pvc <name> -n <namespace>

# ตรวจสอบ Provisioner ยังทำงานอยู่
kubectl get pods -n <provisioner-namespace>

# ดู Provisioner logs
kubectl logs deployment/<provisioner-name> -n <provisioner-namespace>

# ตรวจสอบ StorageClass
kubectl get sc <storageclass-name>
kubectl describe sc <storageclass-name>
```

**2. Provisioner ไม่สร้าง PV**

```bash
# ดู Events ทั้ง cluster
kubectl get events -A --field-selector=reason=ProvisioningFailed

# ตรวจสอบ Permissions (สำหรับ cloud provisioners)
# AWS: ตรวจสอบ IAM role ของ EC2 nodes หรือ IRSA

# ดู Provisioner logs
kubectl logs -l app.kubernetes.io/name=aws-ebs-csi-driver -n kube-system
```

**3. Volume Expansion ไม่สำเร็จ**

```bash
# ตรวจสอบว่า StorageClass รองรับ expansion
kubectl get sc <name> -o jsonpath='{.allowVolumeExpansion}'

# ดู PVC conditions
kubectl describe pvc <name>

# ดู Events
kubectl get events --field-selector=involvedObject.name=<pvc-name>
```

**4. Wrong Storage Tier ถูก Provision**

```bash
# ตรวจสอบ storageClassName ใน PVC
kubectl get pvc <name> -o jsonpath='{.spec.storageClassName}'

# เปรียบเทียบกับ StorageClass ที่มี
kubectl get sc
```

---

## สรุป

| Feature | Dynamic Provisioning | Static Provisioning |
|---------|---------------------|---------------------|
| PV Creation | Automatic | Manual |
| Admin Work | ต่ำ (สร้าง StorageClass ครั้งเดียว) | สูง (สร้าง PV ทีละอัน) |
| Flexibility | สูง (ขนาดตามที่ขอ) | จำกัด (ตามขนาดที่มี) |
| Cloud Integration | ดีมาก | จำกัด |
| Local Storage | รองรับด้วย Provisioner | Native |
| Production Use | Cloud environments | On-premise, specific needs |

### Key Takeaways:
1. **Dynamic Provisioning** ลด admin overhead และเพิ่ม self-service capability
2. **External Provisioners** (เช่น local-path, nfs-subdir) จำเป็นสำหรับ storage types ที่ไม่ใช่ cloud
3. **WaitForFirstConsumer** เป็น best practice สำหรับ local storage
4. **Reclaim Policy** สำคัญมาก - Retain สำหรับ production, Delete สำหรับ ephemeral
5. **Resource Quotas** ควรกำหนดสำหรับแต่ละ namespace เพื่อป้องกัน storage exhaustion

## แหล่งเรียนรู้เพิ่มเติม

- [Dynamic Provisioning Documentation](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)
- [OpenEBS Documentation](https://openebs.io/docs)
- [Longhorn Documentation](https://longhorn.io/docs)
