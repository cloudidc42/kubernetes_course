# Part 06: Kubernetes Architecture

## สารบัญ
- [Kubernetes Architecture Overview](#kubernetes-architecture-overview)
- [Control Plane Components](#control-plane-components)
- [Worker Node Components](#worker-node-components)
- [Communication Flow](#communication-flow)
- [Workshop: สำรวจ Cluster Architecture](#workshop-สำรวจ-cluster-architecture)

---

## Kubernetes Architecture Overview

Kubernetes ใช้ Architecture แบบ **Master-Worker** (หรือ Control Plane - Worker Node)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                            │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                     Control Plane (Master)                       │  │
│  │                                                                   │  │
│  │  ┌────────────────┐   ┌────────┐   ┌─────────────────────────┐  │  │
│  │  │  kube-apiserver│   │  etcd  │   │  kube-controller-manager│  │  │
│  │  │                │   │        │   │                         │  │  │
│  │  │  REST API       │   │ State  │   │  ReplicationController  │  │  │
│  │  │  Authentication│   │ Store  │   │  NodeController         │  │  │
│  │  │  Authorization │   │        │   │  EndpointsController    │  │  │
│  │  └────────────────┘   └────────┘   └─────────────────────────┘  │  │
│  │                                                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                    kube-scheduler                            │  │  │
│  │  │  วาง Pods บน Nodes ที่เหมาะสม                               │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│              │ API Communication │                                     │
│              ▼                   ▼                                     │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐    │
│  │   Worker Node 1  │   │  Worker Node 2  │   │  Worker Node 3  │    │
│  │                  │   │                 │   │                 │    │
│  │  ┌────────────┐  │   │  ┌──────────┐  │   │  ┌──────────┐  │    │
│  │  │  kubelet   │  │   │  │ kubelet  │  │   │  │ kubelet  │  │    │
│  │  ├────────────┤  │   │  ├──────────┤  │   │  ├──────────┤  │    │
│  │  │ kube-proxy │  │   │  │kube-proxy│  │   │  │kube-proxy│  │    │
│  │  ├────────────┤  │   │  ├──────────┤  │   │  ├──────────┤  │    │
│  │  │  Container │  │   │  │Container │  │   │  │Container │  │    │
│  │  │  Runtime   │  │   │  │Runtime   │  │   │  │Runtime   │  │    │
│  │  └────────────┘  │   │  └──────────┘  │   │  └──────────┘  │    │
│  │                  │   │                 │   │                 │    │
│  │  [Pod1] [Pod2]   │   │ [Pod3] [Pod4]  │   │ [Pod5] [Pod6]  │    │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### หลักการทำงานของ Kubernetes

Kubernetes ใช้หลักการ **Desired State vs Actual State**:

```
Desired State (ที่ต้องการ):
  "ต้องการ 3 replicas ของ nginx"
  
Actual State (ที่มีจริง):
  "ตอนนี้มีแค่ 1 replica"
  
Controller Loop:
  1. อ่าน Desired State จาก etcd
  2. ดู Actual State ของ Cluster
  3. ถ้าต่างกัน → ทำให้เหมือนกัน (Reconcile)
  4. วนซ้ำไม่หยุด

ตัวอย่าง:
  Node ล้ม → Pod หาย → Controller สร้าง Pod ใหม่บน Node อื่น
```

---

## Control Plane Components

### 1. kube-apiserver

**หน้าที่:** Front Door ของ Kubernetes - จุดรับทุก Request

```
ทุก Interaction กับ Kubernetes ต้องผ่าน API Server:
                                                       
kubectl → REST API → kube-apiserver → etcd
                         ↓
                   Authentication
                   Authorization
                   Admission Control
                   Validation
                         ↓
                    Store/Execute
```

**ความสามารถ:**
- รับ REST API Requests
- Authentication (ตรวจสอบว่าเป็นใคร)
- Authorization (ตรวจสอบว่ามีสิทธิ์ทำอะไร)
- Admission Control (ตรวจสอบ/แก้ไข Request ก่อน Execute)
- Validation (ตรวจสอบ YAML ถูกต้องไหม)

```bash
# ตัวอย่างการใช้ API Server โดยตรง
kubectl get --raw /api/v1/pods

# หรือ Port-forward เพื่อเข้าถึง API
kubectl proxy --port=8001
curl http://localhost:8001/api/v1/pods
```

### 2. etcd

**หน้าที่:** Distributed Key-Value Store สำหรับเก็บ State ทั้งหมดของ Kubernetes

```
etcd คือ "Database" ของ Kubernetes:

┌──────────────────────────────────────────┐
│                  etcd                     │
│                                          │
│  /registry/pods/default/nginx-xxx         │
│  → {spec: {containers: [{image: nginx}]}} │
│                                          │
│  /registry/services/default/nginx        │
│  → {spec: {ports: [{port: 80}]}}         │
│                                          │
│  /registry/nodes/node1                   │
│  → {status: {capacity: {cpu: "4"}}}      │
│                                          │
│  ทุก Object ใน Kubernetes ถูกเก็บที่นี่!  │
└──────────────────────────────────────────┘

etcd ใช้ Raft Consensus Algorithm:
- ต้องการ Majority (Quorum) เพื่อ Write
- 3 nodes: ต้องการ 2 nodes
- 5 nodes: ต้องการ 3 nodes
- ทำให้ High Availability
```

**ข้อสำคัญ:**
- ถ้า etcd หาย → Kubernetes ทำงานไม่ได้
- ต้อง Backup etcd สม่ำเสมอ

```bash
# ดู etcd Status
kubectl get pods -n kube-system | grep etcd

# Backup etcd (ใน Production ทำสม่ำเสมอ)
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore backup.db
```

### 3. kube-scheduler

**หน้าที่:** ตัดสินใจว่า Pod จะรันบน Node ไหน

```
Scheduling Process:
                                              
New Pod (Pending) → Scheduler ดู:
                                              
1. Filtering (กรอง Nodes ที่ไม่เหมาะสมออก)
   - Node มี Resource พอไหม?
   - Node มี Label ตรงกับ nodeSelector ไหม?
   - Node มี Taint ที่ขัดกับ Pod tolerations ไหม?
   - Node อยู่ใน Zone ที่ต้องการไหม?
                                              
2. Scoring (ให้คะแนน Nodes ที่ผ่าน Filter)
   - Node ที่มี Resource เหลือมาก → คะแนนสูง
   - Node ที่ Pod ชนิดเดียวกันน้อย → คะแนนสูง
   - Node ที่ Image มีอยู่แล้ว → คะแนนสูง
                                              
3. Binding
   - เลือก Node ที่คะแนนสูงสุด
   - Bind Pod กับ Node นั้น
```

**Scheduling Constraints:**

```yaml
# nodeSelector - เลือก Node ตาม Label
spec:
  nodeSelector:
    disktype: ssd
    environment: production

# nodeAffinity - กฎที่ซับซ้อนกว่า
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values:
            - amd64
            - arm64

# podAntiAffinity - กระจาย Pods ไปหลาย Nodes
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: nginx
        topologyKey: kubernetes.io/hostname

# Taints and Tolerations
# Node Taint (ป้องกัน Pods ที่ไม่ Tolerate)
kubectl taint nodes node1 dedicated=gpu:NoSchedule

# Pod Toleration (ยอมรับ Taint)
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

### 4. kube-controller-manager

**หน้าที่:** รัน Controller หลายตัวที่คอย Reconcile Actual State ให้ตรงกับ Desired State

```
Controllers ที่รันใน kube-controller-manager:

1. Node Controller
   - ตรวจสอบ Node สถานะ
   - ถ้า Node ไม่ตอบสนอง → Mark as NotReady
   - ถ้า Node ไม่ตอบสนองนาน → Evict Pods

2. Replication Controller / ReplicaSet Controller
   - ดูแลให้มี Pods จำนวนที่ต้องการ
   - Pod ตาย → สร้างใหม่
   - Pod เกิน → ลบออก

3. Endpoints Controller
   - Update Endpoints สำหรับ Services
   - เมื่อ Pod เข้า/ออก Service

4. Service Account Controller
   - สร้าง Default Service Account ใน Namespace

5. Deployment Controller
   - จัดการ Rolling Update
   - สร้าง/ลบ ReplicaSets

6. StatefulSet Controller
   - จัดการ Stateful Applications

7. DaemonSet Controller
   - ดูแลให้ทุก Node มี Pod ของ DaemonSet

8. Job Controller / CronJob Controller
   - จัดการ Batch Jobs
```

**ตัวอย่าง Reconcile Loop:**

```
ReplicaSet Controller Loop:

1. อ่าน Desired State: "nginx: 3 replicas"
2. อ่าน Actual State: "nginx: 1 pod running"
3. ไม่ตรงกัน!
4. สร้าง 2 Pods เพิ่ม
5. รอ...
6. อ่าน Actual State: "nginx: 3 pods running"
7. ตรงกัน! ✅
8. วนกลับขั้นตอน 1...

(ทำซ้ำทุกๆ วินาที)
```

### 5. cloud-controller-manager (Optional)

**หน้าที่:** Integrate Kubernetes กับ Cloud Provider (AWS, GCP, Azure)

```
cloud-controller-manager จัดการ:

1. Node Controller (Cloud)
   - สร้าง/ลบ Node ตาม Cloud Instance

2. Route Controller
   - ตั้งค่า Routes ใน Cloud Network

3. Service Controller
   - สร้าง Cloud Load Balancer เมื่อมี Service Type LoadBalancer

4. Volume Controller
   - สร้าง Cloud Storage เมื่อมี PVC

ตัวอย่าง:
- สร้าง Service Type LoadBalancer บน AWS
  → cloud-controller-manager สร้าง ALB/NLB อัตโนมัติ

- สร้าง PVC บน GCP
  → cloud-controller-manager สร้าง GCE Persistent Disk อัตโนมัติ
```

---

## Worker Node Components

### 1. kubelet

**หน้าที่:** Agent ที่รันบนทุก Worker Node - สื่อสารกับ Control Plane และจัดการ Containers

```
kubelet ทำงานอย่างไร:

1. Register Node กับ API Server
2. Watch API Server สำหรับ Pods ที่ Assign มาให้ Node นี้
3. เมื่อมี Pod ใหม่ → สั่ง Container Runtime รัน Container
4. Monitor Container Status
5. Report สถานะกลับไปยัง API Server
6. Run Health Checks
```

**kubelet Configuration:**

```yaml
# kubelet config (ตัวอย่าง)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Maximum Pods ต่อ Node
maxPods: 110

# Node Heartbeat ทุก 10 วินาที
nodeStatusUpdateFrequency: 10s

# Health Check
healthzPort: 10248
healthzBindAddress: 127.0.0.1

# Container Runtime
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock

# Resource Eviction
evictionHard:
  memory.available: "200Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
```

**Probes ที่ kubelet ทำ:**

```yaml
# Liveness Probe - ตรวจว่า Container ยังทำงานอยู่ไหม
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 15  # รอ 15s ก่อน Probe แรก
  periodSeconds: 20         # Probe ทุก 20s
  timeoutSeconds: 5
  failureThreshold: 3       # Fail 3 ครั้ง → Restart

# Readiness Probe - ตรวจว่า Container พร้อมรับ Traffic ไหม
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3       # Fail 3 ครั้ง → ออกจาก Service

# Startup Probe - สำหรับ App ที่ Start ช้า
startupProbe:
  httpGet:
    path: /startup
    port: 3000
  failureThreshold: 30      # รอนาน (30 * 10s = 5 minutes)
  periodSeconds: 10
```

### 2. kube-proxy

**หน้าที่:** จัดการ Network Rules บน Node สำหรับ Service Load Balancing

```
kube-proxy ทำงานอย่างไร:

User Request → Service IP:Port
                    ↓
               kube-proxy
               (iptables/ipvs rules)
                    ↓
          เลือก Pod หนึ่งตัว (Load Balance)
                    ↓
              Pod IP:Port

Service 10.96.0.10:80 → Pod 192.168.1.5:8080
                       → Pod 192.168.1.6:8080
                       → Pod 192.168.1.7:8080
```

**Proxy Modes:**

```
1. iptables mode (default)
   - ใช้ Linux iptables rules
   - Random Load Balancing
   - ไม่ผ่าน User Space → เร็วกว่า userspace mode
   - ปัญหา: iptables rules เยอะขึ้นเรื่อยๆ ตาม Services

2. IPVS mode (แนะนำสำหรับ Large Cluster)
   - ใช้ Linux IPVS (IP Virtual Server)
   - Load Balancing Algorithms หลายแบบ (rr, lc, dh, sh, ...)
   - Performance ดีกว่า iptables มาก
   - เหมาะสำหรับ Cluster ที่มี Services เยอะ

3. userspace mode (legacy, ไม่แนะนำ)
   - ผ่าน User Space → ช้า
```

### 3. Container Runtime

**หน้าที่:** รัน Containers จริงๆ บน Node

```
Container Runtime Interface (CRI):

Kubernetes → CRI → Container Runtime → Container

                CRI Implementations:
                ├── containerd (default, recommended)
                ├── CRI-O (lightweight)
                └── Docker Engine (via cri-dockerd, deprecated)
```

**containerd:**
```
containerd Architecture:

Kubernetes
    │
    │ CRI (gRPC)
    ▼
containerd
    │
    ├── Image Management (Pull, Push)
    ├── Snapshot Management
    └── Container Lifecycle
             │
             │ OCI (Open Container Initiative)
             ▼
         runc (Low-level Container Runtime)
             │
             ▼
         Linux Namespace, Cgroups
         (Actual Container)
```

```bash
# ดู containerd version
containerd --version

# ดู Containers ผ่าน containerd
ctr containers list

# ดู Images ผ่าน containerd
ctr images list

# ใช้ crictl (CRI CLI Tool)
crictl ps                    # ดู Containers
crictl images                # ดู Images
crictl pods                  # ดู Pods
crictl logs <container-id>   # ดู Logs
crictl exec -it <id> sh      # เข้าไปใน Container
```

**CRI-O:**
```
CRI-O เป็น Lightweight Container Runtime ที่สร้างมาโดยเฉพาะสำหรับ Kubernetes:

- ไม่มี Docker dependency
- ขนาดเล็กกว่า containerd
- ใช้กับ OpenShift เป็นหลัก
- รองรับ OCI Container Images
```

---

## Communication Flow

### API Request Flow

```
Developer
    │
    │ kubectl apply -f deployment.yaml
    │
    ▼
kubectl CLI
    │
    │ 1. อ่าน ~/.kube/config
    │ 2. ส่ง HTTP Request ไปยัง API Server
    │
    ▼
kube-apiserver
    │
    │ 3. Authentication (ตรวจสอบ Credentials)
    │ 4. Authorization (ตรวจสอบ RBAC)
    │ 5. Admission Control (Validate/Mutate)
    │ 6. Write to etcd
    │
    ▼
etcd
    │
    │ 7. Notify Watchers
    │
    ▼
kube-scheduler (watching for Unscheduled Pods)
    │
    │ 8. Find suitable Node
    │ 9. Bind Pod to Node (Update etcd)
    │
    ▼
kubelet (watching for Pods assigned to its Node)
    │
    │ 10. Pull Container Image
    │ 11. Create Container (via CRI)
    │ 12. Update Pod Status
    │
    ▼
Container Runtime (containerd)
    │
    │ 13. Create Linux Container
    │     (Namespace, Cgroups)
    │
    ▼
Running Container!
```

### Node Communication

```
Control Plane ↔ Worker Nodes:

1. API Server → kubelet
   - Fetch Pod Logs
   - Execute Commands (kubectl exec)
   - Port Forwarding

2. kubelet → API Server
   - Register Node
   - Node Status Update
   - Pod Status Update

Security:
- TLS Certificates สำหรับทุก Connection
- Node Authorization
- Token-based Authentication
```

### Pod-to-Pod Communication

```
Pod Communication:

Pod A (192.168.1.5) → Pod B (192.168.2.5)

ในระดับ CNI (Container Network Interface):
- Pods ใน Cluster เดียวกัน communicate กันโดยตรง
- ไม่ต้องมี NAT
- ทุก Pod มี IP ที่ Unique

CNI Plugins (Container Network Interface):
├── Calico (Network Policy + BGP)
├── Flannel (Simple Overlay Network)
├── Cilium (eBPF-based, Advanced)
├── WeaveNet
└── AWS VPC CNI (EKS specific)
```

---

## Workshop: สำรวจ Cluster Architecture

### เป้าหมาย

ดู Architecture ของ Kubernetes Cluster จริงๆ:
1. ดู Control Plane Components
2. ดู Worker Node Components
3. สำรวจ API Resources
4. ดู Network Rules

### ขั้นตอน 1: เริ่ม Cluster

```bash
# เริ่ม Minikube
minikube start --driver=docker

# ดู Cluster Status
kubectl cluster-info
```

### ขั้นตอน 2: สำรวจ Control Plane

```bash
# ดู Pods ของ Control Plane
kubectl get pods -n kube-system
# NAME                               READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-xxx             1/1     Running   0          10m
# etcd-minikube                      1/1     Running   0          10m
# kube-apiserver-minikube            1/1     Running   0          10m
# kube-controller-manager-minikube   1/1     Running   0          10m
# kube-proxy-xxx                     1/1     Running   0          10m
# kube-scheduler-minikube            1/1     Running   0          10m
# storage-provisioner                1/1     Running   0          10m

# ดูรายละเอียด API Server
kubectl describe pod kube-apiserver-minikube -n kube-system

# ดู etcd Status
kubectl exec -n kube-system etcd-minikube -- \
  etcdctl --cacert=/var/lib/minikube/certs/etcd/ca.crt \
          --cert=/var/lib/minikube/certs/etcd/server.crt \
          --key=/var/lib/minikube/certs/etcd/server.key \
          member list

# ดู Scheduler Status
kubectl get componentstatuses
# NAME                 STATUS    MESSAGE   ERROR
# controller-manager   Healthy   ok
# scheduler            Healthy   ok
# etcd-0               Healthy   ok
```

### ขั้นตอน 3: สำรวจ Worker Nodes

```bash
# ดู Nodes
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   10m   v1.27.3

# ดูรายละเอียด Node
kubectl describe node minikube

# ดู Node Resource Usage
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# minikube   268m         13%    878Mi           22%

# ดู Labels ของ Node
kubectl get node minikube --show-labels

# ดู Events บน Node
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

### ขั้นตอน 4: ดู API Server

```bash
# ดู API Versions ที่รองรับ
kubectl api-versions

# ดู API Resources ทั้งหมด
kubectl api-resources

# ดู API Resources แบบ Namespaced
kubectl api-resources --namespaced=true

# ดู API Resources แบบ Cluster-level
kubectl api-resources --namespaced=false

# ดู API Docs
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.template.spec.containers.resources
```

### ขั้นตอน 5: สำรวจ etcd Data

```bash
# SSH เข้า Minikube Node
minikube ssh

# ดู etcd Data (ต้องอยู่ใน Node)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get /registry --prefix --keys-only | head -50

# ตัวอย่าง Output:
# /registry/apiregistration.k8s.io/apiservices/v1.
# /registry/configmaps/default/kube-root-ca.crt
# /registry/deployments/kube-system/coredns
# /registry/namespaces/default
# /registry/namespaces/kube-system
# /registry/nodes/minikube
# /registry/pods/kube-system/etcd-minikube

# ดูข้อมูลใน etcd (Pod)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get /registry/nodes/minikube | head -20

exit
```

### ขั้นตอน 6: ดู kubelet

```bash
# SSH เข้า Node
minikube ssh

# ดู kubelet Process
ps aux | grep kubelet

# ดู kubelet Config
sudo cat /var/lib/kubelet/config.yaml

# ดู kubelet Logs
sudo journalctl -u kubelet --no-pager | tail -30

# ดู Container Runtime (containerd)
sudo systemctl status containerd

# ดู Containers ผ่าน crictl
sudo crictl ps

exit
```

### ขั้นตอน 7: ดู Network Rules (kube-proxy)

```bash
# SSH เข้า Node
minikube ssh

# ดู iptables rules ที่ kube-proxy สร้าง
sudo iptables -t nat -L KUBE-SERVICES | head -20

# ดูจำนวน Rules ทั้งหมด
sudo iptables -t nat -L | wc -l

exit
```

### ขั้นตอน 8: Deploy Application และดู Flow

```bash
# Deploy Application
kubectl create deployment nginx --image=nginx --replicas=3

# ดู Pods ถูกสร้างอย่างไร
kubectl get pods -w

# ดู Events
kubectl describe deployment nginx

# ดู ReplicaSet ที่ Deployment สร้าง
kubectl get replicasets

# ดู Events ทั้งหมด
kubectl get events --sort-by='.lastTimestamp'
```

### ขั้นตอน 9: ทดสอบ Self-healing

```bash
# ดู Pods ปัจจุบัน
kubectl get pods -o wide

# ลบ Pod หนึ่งตัว
kubectl delete pod <pod-name>

# ดู Events ที่เกิดขึ้น
kubectl get events --sort-by='.lastTimestamp' | tail -10
# LAST SEEN   TYPE     REASON              OBJECT          MESSAGE
# ...
# 5s          Normal   Killing             pod/nginx-xxx   Stopping container nginx
# 3s          Normal   SuccessfulCreate    replicaset/...  Created pod: nginx-yyy
# 2s          Normal   Scheduled           pod/nginx-yyy   Successfully assigned default/nginx-yyy to minikube
# 1s          Normal   Pulled              pod/nginx-yyy   Container image "nginx:latest" already present on machine
# 0s          Normal   Created             pod/nginx-yyy   Created container nginx
# 0s          Normal   Started             pod/nginx-yyy   Started container nginx
```

### ขั้นตอน 10: สำรวจ Admission Controllers

```bash
# ดู Admission Controllers ที่ Enable อยู่
kubectl describe pod kube-apiserver-minikube -n kube-system | \
  grep enable-admission

# ทดสอบ Resource Quota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
  namespace: default
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
EOF

# ดู Quota
kubectl get resourcequota

# ทดลอง Deploy เกิน Quota
kubectl scale deployment nginx --replicas=10
# ถ้าเกิน Quota จะ Error
```

### ทำความสะอาด

```bash
kubectl delete deployment nginx
kubectl delete resourcequota test-quota
minikube stop
```

---

## สรุปสำคัญ

```
Control Plane:
┌──────────────────────────────────────────────────────┐
│ kube-apiserver  │ Front Door, REST API, Authn/Authz  │
│ etcd            │ Distributed State Store             │
│ kube-scheduler  │ Pod Placement                       │
│ kube-controller │ State Reconciliation                │
│ cloud-controller│ Cloud Integration (Optional)        │
└──────────────────────────────────────────────────────┘

Worker Node:
┌──────────────────────────────────────────────────────┐
│ kubelet         │ Node Agent, Container Management   │
│ kube-proxy      │ Network Rules, Service LB           │
│ Container RT    │ Run Containers (containerd/CRI-O)  │
└──────────────────────────────────────────────────────┘

Key Concepts:
- Desired State vs Actual State
- Reconciliation Loop
- Everything through API Server
- etcd = Source of Truth
```

---

## แบบฝึกหัด

1. ทดลองดู Logs ของแต่ละ Control Plane Component
2. สร้าง ResourceQuota และทดสอบ
3. ดู iptables rules ที่ kube-proxy สร้าง
4. ลอง Evict Pod และดู Scheduler ทำงาน

## คำถามทบทวน

1. ถ้า etcd ล้ม จะเกิดอะไรขึ้นกับ Kubernetes?
2. Scheduler ตัดสินใจวาง Pod บน Node ไหนโดยใช้เกณฑ์อะไร?
3. kubelet และ kube-proxy ต่างกันอย่างไร?
4. ทำไม Kubernetes ถึงใช้ Desired State Model?

---

*ต่อไป: [Part 07: Control Plane Deep Dive](./part-07-control-plane.md)*
