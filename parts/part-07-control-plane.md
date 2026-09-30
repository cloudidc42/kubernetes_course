# Part 07: Control Plane - เจาะลึก

## สารบัญ
- [kube-apiserver](#kube-apiserver)
- [etcd](#etcd)
- [kube-scheduler](#kube-scheduler)
- [kube-controller-manager](#kube-controller-manager)
- [cloud-controller-manager](#cloud-controller-manager)
- [Workshop: ดู Control Plane Status](#workshop-ดู-control-plane-status)

---

## kube-apiserver

### ภาพรวม

**kube-apiserver** เป็น Component ที่สำคัญที่สุดของ Kubernetes - ทุก Request ไม่ว่าจะมาจากที่ไหนต้องผ่าน API Server

```
Everything goes through API Server:

kubectl → API Server ← kubelet
                     ← kube-scheduler
                     ← kube-controller-manager
                     ← External Users
                     ← CI/CD Systems
                     ← Monitoring Tools
```

### API Server Request Lifecycle

```
Request Journey:
                                                          
HTTP Request
    │
    ▼
┌─────────────────────────────────────────────────┐
│                  Authentication                   │
│                                                   │
│  Who are you?                                     │
│  ├── X.509 Client Certificates                   │
│  ├── Bearer Tokens (ServiceAccount tokens)       │
│  ├── HTTP Basic Auth (deprecated)                │
│  ├── Bootstrap Tokens                            │
│  └── OpenID Connect Tokens (OAuth2)              │
│                                                   │
│  ถ้า Fail → 401 Unauthorized                     │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│                  Authorization                    │
│                                                   │
│  Do you have permission?                          │
│  ├── RBAC (Role-Based Access Control)            │
│  ├── ABAC (Attribute-Based Access Control)       │
│  ├── Node Authorization                          │
│  └── Webhook                                     │
│                                                   │
│  ถ้า Fail → 403 Forbidden                        │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              Admission Control                    │
│                                                   │
│  Should this request be allowed/modified?         │
│  ├── MutatingAdmissionWebhook (แก้ไข Request)   │
│  ├── ValidatingAdmissionWebhook (ตรวจสอบ)       │
│  ├── ResourceQuota (ตรวจ Quota)                 │
│  ├── LimitRanger (ตั้ง Default Limits)          │
│  ├── PodSecurity (ตรวจ Security)                │
│  └── NamespaceLifecycle                          │
│                                                   │
│  ถ้า Fail → 422 Unprocessable Entity             │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              Object Schema Validation             │
│                                                   │
│  Is the YAML/JSON valid?                          │
│  - Required fields?                              │
│  - Correct types?                                │
│  - Valid values?                                 │
│                                                   │
│  ถ้า Fail → 422 Unprocessable Entity             │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│                   etcd Storage                    │
│                                                   │
│  Persist the object                              │
│  Notify watchers                                 │
└─────────────────────────────────────────────────┘
```

### RBAC (Role-Based Access Control)

```yaml
# 1. สร้าง Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]          # "" หมายถึง core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
# 2. สร้าง ClusterRole (cluster-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]

---
# 3. RoleBinding - ผูก Role กับ User/ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: developer1
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-service-account
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# 4. ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: Group
  name: monitoring-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Admission Controllers ที่สำคัญ

```bash
# ดู Admission Controllers ที่ Enable
kubectl describe pod kube-apiserver-minikube -n kube-system | \
  grep -A5 "enable-admission"

# Default Admission Controllers:
# NamespaceLifecycle    - ป้องกันสร้าง Objects ใน Namespace ที่ Terminating
# LimitRanger           - ตั้ง Default Resource Limits
# ServiceAccount        - สร้าง Default ServiceAccount
# DefaultStorageClass   - ตั้ง Default StorageClass
# ResourceQuota         - บังคับ Resource Quotas
# MutatingAdmissionWebhook   - Custom Webhooks ที่แก้ไข Objects
# ValidatingAdmissionWebhook - Custom Webhooks ที่ Validate Objects
```

**Webhook ตัวอย่าง:**
```yaml
# MutatingAdmissionWebhook สำหรับ Inject Sidecar
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionWebhook
metadata:
  name: sidecar-injector
webhooks:
- name: inject.istio.io
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  clientConfig:
    service:
      name: istio-sidecar-injector
      namespace: istio-system
      path: "/inject"
  admissionReviewVersions: ["v1"]
  sideEffects: None
```

### API Server High Availability

```
HA Setup:
                    Load Balancer
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
       API Server  API Server  API Server
       (Active)    (Active)    (Active)
              │         │         │
              └─────────┼─────────┘
                        │
                      etcd
                  (3 or 5 nodes)
```

```bash
# ดู API Server Options
kubectl describe pod kube-apiserver-minikube -n kube-system | \
  grep "Command" -A 30
```

---

## etcd

### ภาพรวม

**etcd** เป็น Distributed Key-Value Store ที่ใช้ Raft Consensus Algorithm

```
etcd เก็บทุกอย่างใน Kubernetes:

/registry/
  ├── namespaces/
  │   ├── default
  │   ├── kube-system
  │   └── production
  │
  ├── pods/
  │   └── default/
  │       ├── nginx-abc123
  │       └── nginx-def456
  │
  ├── services/
  │   └── default/
  │       └── nginx-svc
  │
  ├── deployments/
  │   └── default/
  │       └── nginx
  │
  ├── secrets/
  │   └── default/
  │       └── my-secret
  │
  └── configmaps/
      └── default/
          └── app-config
```

### Raft Consensus Algorithm

```
Raft ทำงานอย่างไร:

Cluster มี 3 nodes: etcd-1, etcd-2, etcd-3

1. Election (เลือก Leader):
   - Node ที่ไม่ได้ยิน Heartbeat → เป็น Candidate
   - ส่ง Vote Request ไปยัง Nodes อื่น
   - ได้ Majority Votes → เป็น Leader

2. Log Replication:
   - Client เขียนข้อมูลไปยัง Leader
   - Leader เขียนลง Log
   - Leader ส่ง Log ไปยัง Followers
   - ได้ Majority Acknowledgment → Commit
   - ตอบ Client ว่า Success

3. Fault Tolerance:
   - 3 nodes: ทนได้ 1 node fail
   - 5 nodes: ทนได้ 2 nodes fail
   - n nodes: ทนได้ (n-1)/2 node fails

   Quorum Formula: floor(n/2) + 1
```

```
Timeline ตัวอย่าง:

Client: "Write data=hello"
        │
        ▼
Leader (etcd-1):
  Step 1: เขียนลง Log
  Step 2: ส่ง AppendEntries → etcd-2, etcd-3
        │
        ├─► etcd-2: ได้รับ, เขียน Log, ตอบ OK
        └─► etcd-3: ได้รับ, เขียน Log, ตอบ OK
        │
  Step 3: ได้ 2/3 OK (Majority) → Commit
  Step 4: ตอบ Client ว่า Success

etcd-2 ล้ม → etcd-1 ยังมี etcd-3 → ยังทำงานได้
etcd-2 และ etcd-3 ล้ม → เหลือแค่ etcd-1 → ไม่มี Quorum!
```

### etcd Operations

```bash
# ติดตั้ง etcdctl
apt-get install etcd-client

# ตั้ง Environment
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# ดู Member List
etcdctl member list

# ดู Health Status
etcdctl endpoint health
etcdctl endpoint status --write-out=table

# ดู Keys
etcdctl get /registry --prefix --keys-only | head -20

# ดู Value ของ Key
etcdctl get /registry/namespaces/default

# Put Key
etcdctl put /my/key "my-value"

# Get Key
etcdctl get /my/key

# Delete Key
etcdctl del /my/key

# Watch Key
etcdctl watch /my/key

# Backup (Snapshot)
etcdctl snapshot save /backup/snapshot.db

# Verify Backup
etcdctl snapshot status /backup/snapshot.db

# Restore from Backup
etcdctl snapshot restore /backup/snapshot.db \
  --data-dir=/var/lib/etcd-restore
```

### etcd Backup Strategy

```bash
#!/bin/bash
# Backup Script สำหรับ etcd

BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/snapshot_$DATE.db"

# สร้าง Backup
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save "$BACKUP_FILE"

echo "Backup saved to $BACKUP_FILE"

# ตรวจสอบ
ETCDCTL_API=3 etcdctl snapshot status "$BACKUP_FILE" \
  --write-out=table

# ลบ Backups เก่ากว่า 7 วัน
find "$BACKUP_DIR" -name "*.db" -mtime +7 -delete

echo "Old backups cleaned"
```

---

## kube-scheduler

### Scheduling Algorithm

```
Scheduling Process:

1. Queue
   Unscheduled Pods → Scheduling Queue
   (Priority Queue ตาม Pod Priority)

2. Filter Phase
   ┌─────────────────────────────────────────┐
   │  Predicates (Filter Plugins):            │
   │                                         │
   │  NodeUnschedulable - Node ถูก Cordon?   │
   │  NodeName         - nodeName ตรงไหม?   │
   │  NodeAffinity     - Affinity Rules?     │
   │  NodeResourcesFit - Resource พอไหม?    │
   │  NodePorts        - Port ว่างไหม?       │
   │  VolumeBinding    - Volume Available?   │
   │  TaintToleration  - Taint/Toleration?  │
   └─────────────────────────────────────────┘

3. Score Phase
   ┌─────────────────────────────────────────┐
   │  Priority Plugins (Score 0-100):         │
   │                                         │
   │  NodeResourcesBalancedAllocation        │
   │  - Balance CPU/Memory usage             │
   │                                         │
   │  LeastAllocated                         │
   │  - Prefer nodes with least allocated    │
   │                                         │
   │  ImageLocality                          │
   │  - Prefer nodes that have the image     │
   │                                         │
   │  InterPodAffinity                       │
   │  - Pod Affinity/Anti-affinity scores   │
   └─────────────────────────────────────────┘

4. Bind
   เลือก Node Score สูงสุด → Bind Pod
```

### Advanced Scheduling

```yaml
# Priority Class
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority for critical services"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
preemptionPolicy: Never
description: "Low priority batch jobs"

---
# Pod with Priority
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
```

**Pod Preemption:**
```
ถ้า Node ไม่มี Resource เพียงพอ:
- High Priority Pod รอใน Queue
- Scheduler ดูว่า Evict Low Priority Pod แล้วจะมีพอไหม
- ถ้าพอ → Evict Low Priority Pod → Schedule High Priority Pod
```

```yaml
# Topology Spread Constraints (กระจาย Pods ทั่ว Zones)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                    # ต่างกันได้สูงสุด 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: my-app
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: my-app
      containers:
      - name: app
        image: nginx
```

---

## kube-controller-manager

### Controllers ทั้งหมด

```
kube-controller-manager รัน Controllers เหล่านี้:

Workload Controllers:
├── DeploymentController
│   └── จัดการ Rolling Update, ReplicaSets
├── ReplicaSetController
│   └── รักษา Replicas count
├── StatefulSetController
│   └── Stateful Applications
├── DaemonSetController
│   └── Ensure Pod ทุก Node
├── JobController
│   └── Batch Jobs
└── CronJobController
    └── Scheduled Jobs

Service Controllers:
├── ServiceController
│   └── จัดการ Service Resources
├── EndpointController (deprecated)
│   └── Sync Endpoints กับ Pods
└── EndpointSliceController
    └── Modern Endpoint Management

Infrastructure Controllers:
├── NodeController
│   └── Monitor Node Health
├── NamespaceController
│   └── Cleanup Terminating Namespaces
├── ServiceAccountController
│   └── Create Default ServiceAccounts
├── TokenController
│   └── Manage ServiceAccount Tokens
└── ResourceQuotaController
    └── Enforce Resource Quotas

Persistent Volume Controllers:
├── PersistentVolumeController
│   └── Bind PV กับ PVC
└── PVCProtectionController
    └── Protect PVCs from deletion
```

### Reconciliation Loop Deep Dive

```go
// Pseudocode แสดง Controller Pattern

type ReplicaSetController struct {
    client kubernetes.Interface
    rsLister  // ReplicaSet Lister
    podLister // Pod Lister
    queue     // Work Queue
}

func (c *ReplicaSetController) Run() {
    // Watch for changes
    go c.watchReplicaSets()
    go c.watchPods()
    
    // Process queue
    for {
        key := c.queue.Get()
        c.syncReplicaSet(key)
    }
}

func (c *ReplicaSetController) syncReplicaSet(key string) error {
    // 1. Get current ReplicaSet
    rs := c.rsLister.Get(key)
    
    // 2. Get current Pods
    pods := c.podLister.ListByLabel(rs.Spec.Selector)
    
    // 3. Calculate diff
    desired := rs.Spec.Replicas
    actual  := len(pods)
    diff    := desired - actual
    
    // 4. Reconcile
    if diff > 0 {
        // Create more pods
        for i := 0; i < diff; i++ {
            c.createPod(rs)
        }
    } else if diff < 0 {
        // Delete excess pods
        for i := 0; i < -diff; i++ {
            c.deletePod(pods[i])
        }
    }
    
    return nil
}
```

### Node Controller

```
Node Controller ทำงานอย่างไร:

Every 5 seconds:
  Check Node Heartbeat (NodeStatus Update)
  
  If no heartbeat for 40s:
    Node Status → Unknown
    
  If no heartbeat for 5min:
    Node Status → NotReady
    Taint node: node.kubernetes.io/not-ready:NoExecute
    
  If no heartbeat for 5min + pod-eviction-timeout (300s):
    Evict all Pods from Node
    Pod Status → Unknown
    Reschedule Pods on other Nodes
```

```bash
# ดู Node Conditions
kubectl describe node minikube | grep -A20 "Conditions:"

# Node Conditions:
#   Type             Status  LastHeartbeatTime   Reason
#   MemoryPressure   False   ...                 KubeletHasSufficientMemory
#   DiskPressure     False   ...                 KubeletHasNoDiskPressure
#   PIDPressure      False   ...                 KubeletHasSufficientPID
#   Ready            True    ...                 KubeletReady
```

### Deployment Controller

```
Deployment Controller จัดการ Rolling Update:

Before Update:
ReplicaSet v1 (3 replicas): Pod-1, Pod-2, Pod-3

Update Triggered:
1. Create ReplicaSet v2
2. Scale up v2 (1 pod)
3. Scale down v1 (1 pod)
   v1: Pod-1, Pod-2    v2: Pod-A

4. Scale up v2 (1 pod)
5. Scale down v1 (1 pod)
   v1: Pod-1           v2: Pod-A, Pod-B

6. Scale up v2 (1 pod)
7. Scale down v1 (0 pods)
   v1: (empty)         v2: Pod-A, Pod-B, Pod-C

After Update:
ReplicaSet v2 (3 replicas): Pod-A, Pod-B, Pod-C
ReplicaSet v1 (0 replicas): (kept for rollback)
```

```yaml
# Deployment Strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # สร้างได้เกิน Desired เท่าไหร่
      maxUnavailable: 0 # ไม่ให้มี Unavailable Pod เลย (Zero downtime)
  
  # หรือ Recreate Strategy (มี Downtime)
  # strategy:
  #   type: Recreate
  
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
        image: my-app:v2
```

---

## cloud-controller-manager

### การ Integrate กับ Cloud Provider

```
cloud-controller-manager แยก Cloud-specific Logic ออกจาก Kubernetes Core:

Before (monolithic):                After (modular):
kube-controller-manager             kube-controller-manager
  + AWS logic                         (no cloud logic)
  + GCP logic                       cloud-controller-manager
  + Azure logic                       + AWS provider
                                      + GCP provider
                                      + Azure provider
```

### Load Balancer Provisioning

```yaml
# Service Type LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # AWS
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

```
เมื่อ Apply Service:

1. kube-apiserver รับ Request → บันทึกใน etcd
2. cloud-controller-manager เห็น Service ใหม่
3. เรียก AWS API → สร้าง ELB/NLB
4. Update Service Status ด้วย External IP
5. kubectl get svc → เห็น EXTERNAL-IP

BEFORE:
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)
my-service   LoadBalancer   10.0.0.100    <pending>     80:31234/TCP

AFTER:
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP         PORT(S)
my-service   LoadBalancer   10.0.0.100    1.2.3.4             80:31234/TCP
```

### Persistent Volume Provisioning

```yaml
# StorageClass สำหรับ AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer

---
# PVC ที่ใช้ StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```
เมื่อ Apply PVC:

1. PVC Status → Pending
2. cloud-controller-manager เห็น PVC ใหม่
3. เรียก AWS API → สร้าง EBS Volume
4. สร้าง PersistentVolume
5. Bind PV กับ PVC
6. PVC Status → Bound

kubectl get pvc
NAME     STATUS   VOLUME    CAPACITY   STORAGECLASS
my-pvc   Bound    pvc-xxx   10Gi       fast-ssd
```

---

## Workshop: ดู Control Plane Status

### เป้าหมาย

สำรวจและทำความเข้าใจ Control Plane Components อย่างละเอียด

### ขั้นตอน 1: เริ่ม Cluster

```bash
minikube start --driver=docker
```

### ขั้นตอน 2: ดู Control Plane Status

```bash
# ดู Component Status
kubectl get componentstatuses
# NAME                 STATUS    MESSAGE                         ERROR
# controller-manager   Healthy   ok
# scheduler            Healthy   ok
# etcd-0               Healthy   {"health":"true","reason":""}

# ดู Control Plane Pods
kubectl get pods -n kube-system -o wide
```

### ขั้นตอน 3: ดู API Server Details

```bash
# ดู API Server Configuration
kubectl describe pod kube-apiserver-minikube -n kube-system

# ดู API Resources
kubectl api-resources --sort-by=name

# ดู API Versions
kubectl api-versions | sort

# ทดสอบ API โดยตรง
kubectl proxy &
sleep 2

# ดู Cluster Info
curl http://localhost:8001/

# ดู API Groups
curl http://localhost:8001/apis | python3 -c "
import sys, json
data = json.load(sys.stdin)
for group in data['groups'][:10]:
    print(f\"  {group['name']}: {group['preferredVersion']['version']}\")"

# ดู Pods ผ่าน API
curl "http://localhost:8001/api/v1/namespaces/kube-system/pods" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for pod in data['items']:
    print(f\"  {pod['metadata']['name']}: {pod['status']['phase']}\")"

# หยุด proxy
kill %1
```

### ขั้นตอน 4: ดู etcd

```bash
# ดู etcd Pod
kubectl describe pod etcd-minikube -n kube-system | head -50

# เข้าไปใน etcd Pod
kubectl exec -it etcd-minikube -n kube-system -- sh

# ใน etcd Container:
# ดู Member List
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  member list --write-out=table

# ดู Keys ทั้งหมด
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get / --prefix --keys-only | grep -E "^/registry/(pods|deployments|services)" | head -20

exit
```

### ขั้นตอน 5: ดู Scheduler

```bash
# ดู Scheduler Config
kubectl describe pod kube-scheduler-minikube -n kube-system

# ดู Scheduler Logs
kubectl logs kube-scheduler-minikube -n kube-system | head -20

# ทดสอบ Scheduling ด้วย Resource Requests
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-test
  template:
    metadata:
      labels:
        app: resource-test
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
EOF

# ดู Pods ถูก Schedule ที่ไหน
kubectl get pods -o wide

# ดู Events การ Schedule
kubectl describe pod -l app=resource-test | grep -A5 "Events:"
```

### ขั้นตอน 6: ดู Controller Manager

```bash
# ดู Controller Manager Logs
kubectl logs kube-controller-manager-minikube -n kube-system | head -30

# ทดสอบ Reconciliation Loop
# สร้าง Deployment
kubectl create deployment test-reconcile --image=nginx --replicas=3

# ดู Pods
kubectl get pods -l app=test-reconcile

# ลบ Pod หนึ่งตัว
POD=$(kubectl get pods -l app=test-reconcile -o name | head -1)
kubectl delete $POD

# ดู Controller สร้าง Pod ใหม่
kubectl get pods -l app=test-reconcile -w &
WATCH_PID=$!
sleep 5
kill $WATCH_PID

# ดู Events
kubectl describe deployment test-reconcile
```

### ขั้นตอน 7: RBAC Workshop

```bash
# สร้าง ServiceAccount
kubectl create serviceaccount demo-sa

# สร้าง Role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

# สร้าง RoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-pod-reader
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# ทดสอบ Permission
kubectl auth can-i list pods --as=system:serviceaccount:default:demo-sa
# yes

kubectl auth can-i create pods --as=system:serviceaccount:default:demo-sa
# no

kubectl auth can-i delete deployments --as=system:serviceaccount:default:demo-sa
# no

# ดู Permissions ทั้งหมดของ SA
kubectl auth can-i --list --as=system:serviceaccount:default:demo-sa | head -20
```

### ขั้นตอน 8: ResourceQuota

```bash
# สร้าง ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    persistentvolumeclaims: "5"
    services.loadbalancers: "2"
EOF

# ดู Quota
kubectl describe resourcequota compute-quota

# สร้าง Pod ที่ไม่มี Resource Requests
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-resource-pod
spec:
  containers:
  - name: nginx
    image: nginx
EOF
# Error จาก Admission Controller!
# Error from server (Forbidden): pods "no-resource-pod" is forbidden:
# failed quota: compute-quota: must specify limits.cpu for: nginx;
# must specify limits.memory for: nginx

# สร้าง Pod ที่มี Resource Requests
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-resource-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
EOF

# ดู Quota Usage
kubectl describe resourcequota compute-quota
```

### ทำความสะอาด

```bash
kubectl delete deployment resource-test test-reconcile
kubectl delete pod with-resource-pod
kubectl delete serviceaccount demo-sa
kubectl delete role pod-reader
kubectl delete rolebinding demo-pod-reader
kubectl delete resourcequota compute-quota
minikube stop
```

---

## สรุป

```
Control Plane Summary:

kube-apiserver:
  - Front Door สำหรับทุก Request
  - Authentication + Authorization
  - Admission Control
  - REST API

etcd:
  - State Store
  - Raft Consensus
  - Source of Truth
  - BACKUP สม่ำเสมอ!

kube-scheduler:
  - Pod Placement
  - Filter + Score
  - Constraints (Affinity, Taints, Resources)

kube-controller-manager:
  - Reconciliation Loop
  - หลาย Controllers
  - Desired State = Actual State

cloud-controller-manager:
  - Cloud Integration
  - Load Balancer Provisioning
  - Volume Provisioning
```

---

## แบบฝึกหัด

1. สร้าง RBAC สำหรับ Developer ที่ Manage ได้แค่ Namespace ของตัวเอง
2. ทดลองสร้าง ResourceQuota ที่ Limit CPU/Memory
3. ดู Scheduler Events เมื่อ Node ไม่มี Resource พอ
4. ทดสอบ LimitRanger สำหรับตั้ง Default Resource

## คำถามทบทวน

1. อะไรเกิดขึ้นถ้า kube-scheduler ล้ม?
2. Admission Controller แตกต่างจาก Authorization อย่างไร?
3. etcd ต้องมีกี่ Nodes เพื่อ High Availability?
4. Controller Loop ทำงานอย่างไร?

---

*ต่อไป: [Part 08: Worker Nodes](./part-08-worker-nodes.md)*
