# Part 49: CSI Drivers - Container Storage Interface

## บทนำ

**CSI (Container Storage Interface)** คือ standard API ที่ช่วยให้ Container Orchestration systems (Kubernetes, Mesos ฯลฯ) ทำงานกับ storage systems ต่างๆ ได้อย่างสม่ำเสมอ โดยไม่ต้องแก้ไข core code ของ Kubernetes

### ทำไมต้องมี CSI

**ก่อน CSI (In-tree plugins):**
- Storage driver code อยู่ใน Kubernetes core
- เพิ่ม driver ใหม่ต้องรอ Kubernetes release cycle
- Bug fix ต้องรอ Kubernetes upgrade
- Storage vendors ต้องเปิด code ต่อ community

**หลัง CSI (Out-of-tree plugins):**
- Storage driver เป็น external plugin
- Deploy/Update ได้อิสระจาก Kubernetes
- Storage vendors จัดการ code เอง
- รองรับ multiple Kubernetes versions

---

## 1. CSI Driver Architecture

### 1.1 Components หลัก

```
Kubernetes Cluster
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Control Plane                                              │
│  ┌─────────────────────────────────────────────────┐       │
│  │ Kubernetes API Server                           │       │
│  │  - PV/PVC management                           │       │
│  │  - StorageClass management                     │       │
│  │  - VolumeAttachment management                 │       │
│  └─────────────────────────────────────────────────┘       │
│            │           │           │                        │
│            ▼           ▼           ▼                        │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────────────┐ │
│  │  CSI External│  │ CSI       │  │ CSI External         │ │
│  │  Provisioner │  │ External  │  │ Snapshotter          │ │
│  │  (creates PV)│  │ Attacher  │  │ (snapshots)         │ │
│  └──────────────┘  └───────────┘  └──────────────────────┘ │
│            │           │                                    │
│            └─────┬─────┘                                   │
│                  │                                          │
│  CSI Driver Controller Pod (DaemonSet or Deployment)        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  CSI Plugin Container (Controller Service)           │  │
│  │  - CreateVolume / DeleteVolume                       │  │
│  │  - ControllerPublishVolume / ControllerUnpublish     │  │
│  │  - CreateSnapshot / DeleteSnapshot                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Worker Nodes (DaemonSet)                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Node CSI Plugin (Node Service)                      │  │
│  │  - NodeStageVolume / NodeUnstageVolume               │  │
│  │  - NodePublishVolume / NodeUnpublishVolume           │  │
│  │  - NodeGetInfo / NodeGetCapabilities                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 CSI Components

**Controller Components (Deployment):**
- **external-provisioner**: ดู PVC และสร้าง/ลบ volumes
- **external-attacher**: Attach/Detach volumes จาก nodes
- **external-snapshotter**: สร้าง/ลบ snapshots
- **external-resizer**: Resize volumes
- **liveness-probe**: Monitor CSI driver health

**Node Components (DaemonSet):**
- **node-driver-registrar**: Register CSI driver กับ kubelet
- **CSI Driver**: Mount/Unmount volumes บน nodes
- **liveness-probe**: Monitor node plugin health

---

## 2. CSI Driver Interface

### 2.1 gRPC Services

```protobuf
// Identity Service (Controller และ Node ต้องมี)
service Identity {
  rpc GetPluginInfo(GetPluginInfoRequest) returns (GetPluginInfoResponse);
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest) returns (GetPluginCapabilitiesResponse);
  rpc Probe(ProbeRequest) returns (ProbeResponse);
}

// Controller Service (สำหรับ cloud operations)
service Controller {
  rpc CreateVolume(CreateVolumeRequest) returns (CreateVolumeResponse);
  rpc DeleteVolume(DeleteVolumeRequest) returns (DeleteVolumeResponse);
  rpc ControllerPublishVolume(ControllerPublishVolumeRequest) returns (ControllerPublishVolumeResponse);
  rpc ControllerUnpublishVolume(ControllerUnpublishVolumeRequest) returns (ControllerUnpublishVolumeResponse);
  rpc ValidateVolumeCapabilities(ValidateVolumeCapabilitiesRequest) returns (ValidateVolumeCapabilitiesResponse);
  rpc ListVolumes(ListVolumesRequest) returns (ListVolumesResponse);
  rpc GetCapacity(GetCapacityRequest) returns (GetCapacityResponse);
  rpc ControllerGetCapabilities(ControllerGetCapabilitiesRequest) returns (ControllerGetCapabilitiesResponse);
  rpc CreateSnapshot(CreateSnapshotRequest) returns (CreateSnapshotResponse);
  rpc DeleteSnapshot(DeleteSnapshotRequest) returns (DeleteSnapshotResponse);
  rpc ListSnapshots(ListSnapshotsRequest) returns (ListSnapshotsResponse);
  rpc ControllerExpandVolume(ControllerExpandVolumeRequest) returns (ControllerExpandVolumeResponse);
  rpc ControllerGetVolume(ControllerGetVolumeRequest) returns (ControllerGetVolumeResponse);
  rpc ControllerModifyVolume(ControllerModifyVolumeRequest) returns (ControllerModifyVolumeResponse);
}

// Node Service (สำหรับ mount operations)
service Node {
  rpc NodeStageVolume(NodeStageVolumeRequest) returns (NodeStageVolumeResponse);
  rpc NodeUnstageVolume(NodeUnstageVolumeRequest) returns (NodeUnstageVolumeResponse);
  rpc NodePublishVolume(NodePublishVolumeRequest) returns (NodePublishVolumeResponse);
  rpc NodeUnpublishVolume(NodeUnpublishVolumeRequest) returns (NodeUnpublishVolumeResponse);
  rpc NodeGetVolumeStats(NodeGetVolumeStatsRequest) returns (NodeGetVolumeStatsResponse);
  rpc NodeExpandVolume(NodeExpandVolumeRequest) returns (NodeExpandVolumeResponse);
  rpc NodeGetCapabilities(NodeGetCapabilitiesRequest) returns (NodeGetCapabilitiesResponse);
  rpc NodeGetInfo(NodeGetInfoRequest) returns (NodeGetInfoResponse);
}
```

---

## 3. ติดตั้ง CSI Driver

### 3.1 ขั้นตอนการ Deploy CSI Driver

```yaml
# csi-driver-template.yaml
---
# 1. CSIDriver Object
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: example.csi.driver.io
spec:
  attachRequired: true          # ต้องการ ControllerPublish
  podInfoOnMount: true          # ส่ง Pod info ไปยัง NodePublish
  volumeLifecycleModes:
  - Persistent                  # Standard persistent volumes
  - Ephemeral                   # Inline ephemeral volumes (optional)
  storageCapacity: true         # Report storage capacity
  fsGroupPolicy: File           # None, File, ReadWriteOnceWithFSType
---
# 2. ServiceAccount, ClusterRole, ClusterRoleBinding สำหรับ Controller
apiVersion: v1
kind: ServiceAccount
metadata:
  name: csi-driver-controller
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-driver-controller-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete", "patch"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["list", "watch", "create", "update", "patch"]
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["get", "list"]
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshotcontents"]
  verbs: ["get", "list"]
- apiGroups: ["storage.k8s.io"]
  resources: ["csinodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["volumeattachments"]
  verbs: ["get", "list", "watch", "patch"]
---
# 3. Controller Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csi-driver-controller
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: csi-driver-controller
  template:
    metadata:
      labels:
        app: csi-driver-controller
    spec:
      serviceAccountName: csi-driver-controller
      containers:
      # CSI External Provisioner sidecar
      - name: external-provisioner
        image: registry.k8s.io/sig-storage/csi-provisioner:v3.6.0
        args:
        - "--csi-address=/csi/csi.sock"
        - "--v=2"
        - "--feature-gates=Topology=true"
        - "--leader-election"
        volumeMounts:
        - name: socket-dir
          mountPath: /csi
      
      # CSI External Attacher sidecar
      - name: external-attacher
        image: registry.k8s.io/sig-storage/csi-attacher:v4.4.0
        args:
        - "--csi-address=/csi/csi.sock"
        - "--v=2"
        - "--leader-election"
        volumeMounts:
        - name: socket-dir
          mountPath: /csi
      
      # CSI External Resizer sidecar
      - name: external-resizer
        image: registry.k8s.io/sig-storage/csi-resizer:v1.9.0
        args:
        - "--csi-address=/csi/csi.sock"
        - "--v=2"
        - "--leader-election"
        volumeMounts:
        - name: socket-dir
          mountPath: /csi
      
      # Actual CSI Driver Plugin
      - name: csi-driver
        image: my-csi-driver:v1.0.0
        args:
        - "--endpoint=unix:///csi/csi.sock"
        - "--node-id=$(NODE_NAME)"
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: socket-dir
          mountPath: /csi
      
      volumes:
      - name: socket-dir
        emptyDir: {}
---
# 4. Node DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-driver-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-driver-node
  template:
    metadata:
      labels:
        app: csi-driver-node
    spec:
      containers:
      # Node Driver Registrar sidecar
      - name: node-driver-registrar
        image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.9.0
        args:
        - "--csi-address=/csi/csi.sock"
        - "--kubelet-registration-path=/var/lib/kubelet/plugins/example.csi.driver.io/csi.sock"
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: registration-dir
          mountPath: /registration
      
      # CSI Driver Node Plugin
      - name: csi-driver
        image: my-csi-driver:v1.0.0
        args:
        - "--endpoint=unix:///csi/csi.sock"
        securityContext:
          privileged: true      # จำเป็นสำหรับ mount operations
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: kubelet-dir
          mountPath: /var/lib/kubelet
          mountPropagation: Bidirectional
        - name: dev-dir
          mountPath: /dev
      
      volumes:
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/plugins/example.csi.driver.io
          type: DirectoryOrCreate
      - name: registration-dir
        hostPath:
          path: /var/lib/kubelet/plugins_registry
          type: Directory
      - name: kubelet-dir
        hostPath:
          path: /var/lib/kubelet
          type: Directory
      - name: dev-dir
        hostPath:
          path: /dev
          type: Directory
```

---

## 4. CSI Drivers ยอดนิยม

### 4.1 AWS EBS CSI Driver

```bash
# ติดตั้งด้วย Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true

# ตรวจสอบ
kubectl get csidrivers ebs.csi.aws.com
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

### 4.2 NFS CSI Driver

```bash
# ติดตั้ง
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version v4.6.0 \
  --set controller.replicas=2 \
  --set controller.livenessProbe.healthPort=29652

# ตรวจสอบ
kubectl get csidrivers nfs.csi.k8s.io
kubectl get pods -n kube-system -l app=csi-nfs-controller
kubectl get pods -n kube-system -l app=csi-nfs-node
```

**NFS CSI StorageClass:**

```yaml
# nfs-csi-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi-dynamic
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.10
  share: /exports/kubernetes
  # path pattern สำหรับ subdirectory
  subDir: ""
  # permissions สำหรับ volume root directory
  mountPermissions: "0777"
  # onDelete: delete หรือ retain
  onDelete: archive
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- hard
- nfsvers=4.1
```

### 4.3 Local Volume CSI Driver

```bash
# ติดตั้ง Local Static Provisioner
helm repo add sig-storage-local-static-provisioner https://kubernetes-sigs.github.io/sig-storage-local-static-provisioner
helm repo update

helm install local-static-provisioner sig-storage-local-static-provisioner/local-static-provisioner \
  --namespace kube-system \
  --set classes[0].name=local-ssd \
  --set classes[0].hostDir=/mnt/ssd \
  --set classes[0].storageClass=true \
  --set classes[0].reclaimPolicy=Delete
```

### 4.4 Longhorn CSI Driver

```bash
# ติดตั้ง Longhorn
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml

# รอ Longhorn start
kubectl wait --for=condition=available deployment/longhorn-manager \
  -n longhorn-system --timeout=300s

# ดู CSI Driver
kubectl get csidrivers driver.longhorn.io
kubectl get pods -n longhorn-system

# Access Longhorn UI
kubectl port-forward -n longhorn-system service/longhorn-frontend 8080:80 &
open http://localhost:8080
```

**Longhorn StorageClasses:**

```yaml
# longhorn-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: ext4
  dataLocality: disabled
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ha
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  numberOfReplicas: "3"
  dataLocality: best-effort
  fsType: ext4
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-strict
provisioner: driver.longhorn.io
allowVolumeExpansion: false
reclaimPolicy: Retain
parameters:
  numberOfReplicas: "3"
  dataLocality: strict-local  # Data stays on same node
  fsType: ext4
```

---

## 5. CSI Volume Snapshot

**Volume Snapshot** ช่วยให้สร้าง point-in-time copy ของ Volume

### 5.1 ติดตั้ง Snapshot Controller

```bash
# ติดตั้ง External Snapshotter CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# ติดตั้ง Snapshot Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml

# ตรวจสอบ
kubectl get pods -n kube-system | grep snapshot
```

### 5.2 VolumeSnapshotClass

```yaml
# snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: longhorn-snapshot-class
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  type: snap
---
# สำหรับ NFS
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: nfs-snapshot-class
driver: nfs.csi.k8s.io
deletionPolicy: Retain
```

### 5.3 สร้าง Volume Snapshot

```yaml
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: database-snapshot-$(date +%Y%m%d)
  namespace: production
  labels:
    app: database
    backup-type: snapshot
spec:
  volumeSnapshotClassName: longhorn-snapshot-class
  source:
    persistentVolumeClaimName: database-pvc
```

```bash
# สร้าง Snapshot
kubectl apply -f volume-snapshot.yaml

# ดู Snapshots
kubectl get volumesnapshots -n production

# ดูรายละเอียด
kubectl describe volumesnapshot database-snapshot-20240115 -n production

# ดู Snapshot Content (underlying storage)
kubectl get volumesnapshotcontents
```

### 5.4 Restore จาก Snapshot

```yaml
# restore-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc-restored
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn-standard
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: database-snapshot-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```bash
kubectl apply -f restore-from-snapshot.yaml
kubectl get pvc database-pvc-restored -n production -w
```

---

## 6. CSI Ephemeral Volumes

**Ephemeral Volumes** ด้วย CSI ช่วยให้ใช้ CSI storage แบบ temporary (ไม่ต้องสร้าง PVC)

```yaml
# csi-ephemeral-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-ephemeral-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sleep", "3600"]
    volumeMounts:
    - name: ephemeral-storage
      mountPath: /tmp/csi-data
  
  volumes:
  - name: ephemeral-storage
    csi:
      driver: inline.storage.kubernetes.io
      readOnly: false
      volumeAttributes:
        foo: bar
```

---

## 7. Workshop: ใช้ CSI Driver

### Workshop Overview
ติดตั้ง Longhorn CSI Driver และใช้งาน Volume Snapshots

### Prerequisites
- Kubernetes cluster (Minikube พร้อม enough resources)
- Helm installed
- ต้องการ CPU 2+ cores, RAM 4GB+ สำหรับ Longhorn
- เวลาประมาณ 90-120 นาที

### Step 1: ตรวจสอบ Prerequisites

```bash
# ตรวจสอบ resources
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory'

# ตรวจสอบ disk space
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.ephemeral-storage}{"\n"}{end}'

# Minikube: เพิ่ม resources ถ้าจำเป็น
minikube stop
minikube start --cpus=4 --memory=8192 --disk-size=50g

# ตรวจสอบ existing CSI drivers
kubectl get csidrivers
```

### Step 2: ติดตั้ง NFS CSI Driver (เบากว่า Longhorn)

```bash
# ติดตั้ง NFS CSI Driver
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set controller.replicas=1

# ตรวจสอบ
kubectl get pods -n kube-system -l app=csi-nfs-controller
kubectl get pods -n kube-system -l app=csi-nfs-node

# ดู CSI Driver registration
kubectl get csidrivers nfs.csi.k8s.io
kubectl describe csidriver nfs.csi.k8s.io
```

### Step 3: ตั้งค่า NFS Server สำหรับ Lab

```yaml
# lab-nfs-server.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: csi-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: csi-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: itsthenetwork/nfs-server-alpine:12
        env:
        - name: SHARED_DIRECTORY
          value: "/exports"
        volumeMounts:
        - name: nfs-data
          mountPath: /exports
        securityContext:
          privileged: true
        ports:
        - containerPort: 2049
          protocol: TCP
      volumes:
      - name: nfs-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: csi-lab
spec:
  selector:
    app: nfs-server
  ports:
  - port: 2049
    protocol: TCP
```

```bash
kubectl apply -f lab-nfs-server.yaml
kubectl wait --for=condition=available deployment/nfs-server -n csi-lab --timeout=120s

# ดู NFS Server IP
NFS_IP=$(kubectl get svc nfs-server -n csi-lab -o jsonpath='{.spec.clusterIP}')
echo "NFS Server ClusterIP: $NFS_IP"
```

### Step 4: สร้าง StorageClass ใช้ NFS CSI Driver

```yaml
# nfs-csi-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: nfs.csi.k8s.io
parameters:
  server: <NFS_IP>        # แทนด้วย NFS Server IP
  share: /
  mountPermissions: "0777"
  onDelete: retain
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- hard
```

```bash
# แทน NFS_IP ใน StorageClass
sed -i "s/<NFS_IP>/$NFS_IP/" nfs-csi-sc.yaml
kubectl apply -f nfs-csi-sc.yaml
kubectl get storageclass nfs-csi
```

### Step 5: ทดสอบ Dynamic Provisioning ด้วย CSI

```yaml
# csi-pvc-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-nfs-pvc
  namespace: csi-lab
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-csi
```

```bash
kubectl apply -f csi-pvc-test.yaml
kubectl get pvc csi-nfs-pvc -n csi-lab -w
kubectl describe pvc csi-nfs-pvc -n csi-lab
```

### Step 6: Deploy Application ที่ใช้ CSI Volume

```yaml
# csi-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csi-demo-app
  namespace: csi-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: csi-demo
  template:
    metadata:
      labels:
        app: csi-demo
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          POD_NAME=$(hostname)
          echo "CSI Demo App: $POD_NAME"
          echo "$POD_NAME: Started at $(date)" >> /data/pods.log
          while true; do
            echo "$POD_NAME: $(date)" >> /data/activity.log
            sleep 15
          done
        volumeMounts:
        - name: csi-storage
          mountPath: /data
      volumes:
      - name: csi-storage
        persistentVolumeClaim:
          claimName: csi-nfs-pvc
```

```bash
kubectl apply -f csi-app.yaml
kubectl get pods -n csi-lab -w

# ดูว่า Pods ทั้งสองแชร์ storage ได้
sleep 30
kubectl exec -n csi-lab deployment/csi-demo-app -- cat /data/pods.log
kubectl exec -n csi-lab deployment/csi-demo-app -- cat /data/activity.log | head -20
```

### Step 7: ตรวจสอบ CSI Driver Status

```bash
# ดู CSI Driver
kubectl get csidrivers
kubectl describe csidriver nfs.csi.k8s.io

# ดู CSI Nodes
kubectl get csinodes
kubectl describe csinode $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# ดู Volume Attachments
kubectl get volumeattachments

# ดู CSI Plugin pods
kubectl get pods -n kube-system | grep csi
```

### Step 8: Monitor CSI Driver

```bash
# ดู logs ของ CSI Controller
kubectl logs -n kube-system -l app=csi-nfs-controller -c nfs -f

# ดู logs ของ CSI Node plugin
kubectl logs -n kube-system -l app=csi-nfs-node -c nfs

# ดู Events ที่เกี่ยวกับ CSI
kubectl get events -n csi-lab --field-selector=reason=Provisioning
kubectl get events -n csi-lab --field-selector=reason=VolumeResizing
```

### Step 9: ทดสอบ Volume ใน Multiple Pods (RWX)

```bash
# Pod 1 เขียน
POD1=$(kubectl get pods -n csi-lab -l app=csi-demo -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -n csi-lab -l app=csi-demo -o jsonpath='{.items[1].metadata.name}')

echo "Pod1: $POD1"
echo "Pod2: $POD2"

# เขียนจาก Pod1
kubectl exec -n csi-lab $POD1 -- \
  sh -c 'echo "Written by $HOSTNAME at $(date)" > /data/pod1-file.txt'

# อ่านจาก Pod2 (ควรเห็นไฟล์จาก Pod1)
kubectl exec -n csi-lab $POD2 -- cat /data/pod1-file.txt

# เขียนจาก Pod2
kubectl exec -n csi-lab $POD2 -- \
  sh -c 'echo "Written by $HOSTNAME at $(date)" > /data/pod2-file.txt'

# อ่านจาก Pod1
kubectl exec -n csi-lab $POD1 -- cat /data/pod2-file.txt

# ดูไฟล์ทั้งหมด
kubectl exec -n csi-lab $POD1 -- ls -la /data/
```

### Step 10: CSI Volume Metrics

```bash
# ดู Volume Stats (ถ้า CSI Driver รองรับ)
kubectl exec -n csi-lab $POD1 -- df -h /data

# ดู disk usage
kubectl exec -n csi-lab $POD1 -- du -sh /data/*

# สร้าง test data
kubectl exec -n csi-lab $POD1 -- \
  dd if=/dev/urandom of=/data/testfile bs=1M count=50 2>&1
  
kubectl exec -n csi-lab $POD1 -- df -h /data
```

### Step 11: Cleanup และ Verify Data Retention

```bash
# จด PVC name
echo "PVC: csi-nfs-pvc"

# ลบ Deployment
kubectl delete deployment csi-demo-app -n csi-lab

# ดู PVC ยังอยู่ (เพราะ Retain policy)
kubectl get pvc csi-nfs-pvc -n csi-lab

# สร้าง Pod ใหม่ที่ใช้ PVC เดิม
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: verify-pod
  namespace: csi-lab
spec:
  containers:
  - name: verify
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "=== Files in CSI Volume ==="
      ls -la /data/
      echo ""
      echo "=== Activity Log ==="
      tail -5 /data/activity.log 2>/dev/null || echo "No activity log"
      echo ""
      echo "=== Pod files ==="
      cat /data/pods.log 2>/dev/null || echo "No pods log"
      sleep 3600
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: csi-nfs-pvc
EOF

kubectl get pod verify-pod -n csi-lab -w
kubectl logs verify-pod -n csi-lab
```

### Step 12: ทำความสะอาด

```bash
kubectl delete namespace csi-lab

# Uninstall NFS CSI Driver
helm uninstall csi-driver-nfs -n kube-system
kubectl delete storageclass nfs-csi

# ลบ CRDs (optional)
kubectl delete csidrivers nfs.csi.k8s.io 2>/dev/null || true
```

---

## 8. CSI Driver Development Overview

### 8.1 สร้าง Simple CSI Driver

```go
// main.go - Skeleton ของ CSI Driver
package main

import (
    "context"
    "fmt"
    "os"
    
    "github.com/container-storage-interface/spec/lib/go/csi"
    "google.golang.org/grpc"
)

type myCSIDriver struct {
    name    string
    version string
}

// Identity Service
func (d *myCSIDriver) GetPluginInfo(ctx context.Context, req *csi.GetPluginInfoRequest) (*csi.GetPluginInfoResponse, error) {
    return &csi.GetPluginInfoResponse{
        Name:          d.name,
        VendorVersion: d.version,
    }, nil
}

func (d *myCSIDriver) GetPluginCapabilities(ctx context.Context, req *csi.GetPluginCapabilitiesRequest) (*csi.GetPluginCapabilitiesResponse, error) {
    return &csi.GetPluginCapabilitiesResponse{
        Capabilities: []*csi.PluginCapability{
            {
                Type: &csi.PluginCapability_Service_{
                    Service: &csi.PluginCapability_Service{
                        Type: csi.PluginCapability_Service_CONTROLLER_SERVICE,
                    },
                },
            },
        },
    }, nil
}

func (d *myCSIDriver) Probe(ctx context.Context, req *csi.ProbeRequest) (*csi.ProbeResponse, error) {
    return &csi.ProbeResponse{}, nil
}

func main() {
    driver := &myCSIDriver{
        name:    "my-csi.storage.example.com",
        version: "v1.0.0",
    }
    
    endpoint := os.Getenv("CSI_ENDPOINT")
    if endpoint == "" {
        endpoint = "unix:///csi/csi.sock"
    }
    
    fmt.Printf("Starting CSI Driver: %s@%s\n", driver.name, driver.version)
    
    server := grpc.NewServer()
    csi.RegisterIdentityServer(server, driver)
    
    // Start listening...
    fmt.Println("CSI Driver started on", endpoint)
}
```

---

## 9. CSI Driver Compatibility Matrix

### ฟีเจอร์ที่ CSI Drivers รองรับ

| Driver | Dynamic Provisioning | Snapshots | Resize | RWX | Raw Block |
|--------|---------------------|-----------|--------|-----|-----------|
| AWS EBS | ✓ | ✓ | ✓ | - | ✓ |
| GCE PD | ✓ | ✓ | ✓ | - | ✓ |
| Azure Disk | ✓ | ✓ | ✓ | - | ✓ |
| Azure File | ✓ | ✓ | ✓ | ✓ | - |
| AWS EFS | ✓ | - | - | ✓ | - |
| NFS CSI | ✓ | - | ✓ | ✓ | - |
| Longhorn | ✓ | ✓ | ✓ | ✓ | - |
| Rook-Ceph RBD | ✓ | ✓ | ✓ | - | ✓ |
| Rook-Ceph CephFS | ✓ | ✓ | ✓ | ✓ | - |

---

## 10. Troubleshooting CSI Drivers

### ปัญหาที่พบบ่อย

**1. Volume ไม่ถูก attach**

```bash
# ดู VolumeAttachments
kubectl get volumeattachments

# ดู events
kubectl describe volumeattachment <name>

# ดู CSI Controller logs
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system -l app=csi-controller -o jsonpath='{.items[0].metadata.name}') \
  -c external-attacher
```

**2. Volume ไม่ถูก mount**

```bash
# ดู Node CSI logs
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system -l app=csi-node -o jsonpath='{.items[0].metadata.name}') \
  -c csi-driver

# ดู kubelet logs บน Node
journalctl -u kubelet --since "10 minutes ago" | grep -i "csi"
```

**3. Snapshot creation fails**

```bash
# ดู VolumeSnapshotContent
kubectl get volumesnapshotcontents

# ดู Snapshot events
kubectl describe volumesnapshot <name>

# ดู Snapshot Controller logs
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system -l app=snapshot-controller -o jsonpath='{.items[0].metadata.name}')
```

---

## สรุป

| Concept | คำอธิบาย |
|---------|----------|
| CSI | Standard interface สำหรับ storage plugins |
| Controller Plugin | Manages volumes at cloud/storage level |
| Node Plugin | Mounts volumes บน worker nodes |
| External Sidecar | Helper containers สำหรับ Kubernetes integration |
| VolumeSnapshot | Point-in-time copy ของ volume |
| Ephemeral Volume | Temporary CSI volume ใน Pod spec |

### Key Takeaways:
1. **CSI** เป็น standard ที่ทุก modern storage solution ควรใช้ แทน in-tree plugins ที่ deprecated
2. **Controller Plugin** ทำงานบน control plane (หรือ Deployment), **Node Plugin** ทำงานบนทุก Node (DaemonSet)
3. **External Sidecar containers** เป็น bridge ระหว่าง Kubernetes API และ CSI Driver
4. **VolumeSnapshots** ต้องการ External Snapshotter CRDs และ Controller
5. **privileged: true** จำเป็นสำหรับ Node Plugin เพื่อทำ mount operations
6. ตรวจสอบ **CSI Driver compatibility** ก่อนเลือกใช้ งาน โดยเฉพาะ RWX support

## แหล่งเรียนรู้เพิ่มเติม

- [CSI Specification](https://github.com/container-storage-interface/spec)
- [Kubernetes CSI Documentation](https://kubernetes-csi.github.io/docs/)
- [CSI Driver List](https://kubernetes-csi.github.io/docs/drivers.html)
- [External Snapshotter](https://github.com/kubernetes-csi/external-snapshotter)
