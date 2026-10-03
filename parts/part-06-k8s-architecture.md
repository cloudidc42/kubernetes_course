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

---

## Kubernetes Request Flow แบบ Step-by-step

### Overview: kubectl → API Server → etcd → Scheduler → kubelet

```
Full Request Flow เมื่อรัน: kubectl create deployment my-app --image=nginx

Step 1: kubectl
  ┌─────────────────────────────────────────────────────────────────┐
  │  User runs: kubectl create deployment my-app --image=nginx       │
  │                                                                   │
  │  kubectl:                                                         │
  │  1. อ่าน kubeconfig (~/.kube/config)                            │
  │  2. ดึง Credentials (Client Certificate / Token)                │
  │  3. Serialize request เป็น JSON                                   │
  │  4. ส่ง HTTP POST ไปยัง API Server                               │
  │                                                                   │
  │  HTTP Request:                                                    │
  │  POST /apis/apps/v1/namespaces/default/deployments               │
  │  Authorization: Bearer <token>                                   │
  │  Content-Type: application/json                                   │
  │  Body: {"apiVersion":"apps/v1","kind":"Deployment",...}          │
  └───────────────────────────┬─────────────────────────────────────┘
                               │ HTTPS
                               ▼
Step 2: API Server (kube-apiserver)
  ┌─────────────────────────────────────────────────────────────────┐
  │  kube-apiserver รับ Request:                                     │
  │                                                                   │
  │  Phase 1: Authentication (AuthN)                                 │
  │  ├── ตรวจสอบ Bearer Token / Certificate                         │
  │  ├── เรียก Token Review API หรือ Webhook                        │
  │  └── ผลลัพธ์: User Identity (system:serviceaccount:default:sa) │
  │                                                                   │
  │  Phase 2: Authorization (AuthZ - RBAC)                          │
  │  ├── ตรวจสอบว่า User มีสิทธิ์สร้าง Deployment หรือไม่           │
  │  ├── ดู ClusterRole / Role Bindings                              │
  │  └── ผลลัพธ์: Allowed หรือ Forbidden                           │
  │                                                                   │
  │  Phase 3: Admission Control                                      │
  │  ├── Mutating Webhooks (แก้ไข Object ก่อน)                     │
  │  │   - Inject Sidecar (Istio)                                    │
  │  │   - Add Default Labels                                        │
  │  │   - Set Default Resource Limits                               │
  │  ├── Object Validation                                           │
  │  │   - ตรวจสอบ Schema                                           │
  │  │   - ตรวจสอบ Field Values                                     │
  │  └── Validating Webhooks (ตรวจสอบสุดท้าย)                     │
  │                                                                   │
  │  Phase 4: Write to etcd                                         │
  │  └── บันทึก Deployment Object ลง etcd                          │
  └───────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
Step 3: etcd
  ┌─────────────────────────────────────────────────────────────────┐
  │  etcd เก็บ Object:                                               │
  │                                                                   │
  │  Key: /registry/apps/deployments/default/my-app                 │
  │  Value: (serialized Deployment object)                           │
  │                                                                   │
  │  etcd ส่ง Watch Event กลับ API Server:                         │
  │  Event Type: ADDED                                               │
  │  Object: Deployment/my-app                                       │
  └───────────────────────────┬─────────────────────────────────────┘
                               │ Watch Event
                               ▼
Step 4: Controller Manager
  ┌─────────────────────────────────────────────────────────────────┐
  │  Deployment Controller ได้รับ Watch Event:                      │
  │                                                                   │
  │  1. Reconcile Loop เริ่มทำงาน                                   │
  │  2. ตรวจสอบ Current State vs Desired State                     │
  │     Desired: 1 ReplicaSet, 1 Pod                                │
  │     Current: ไม่มีอะไร                                         │
  │  3. สร้าง ReplicaSet ผ่าน API Server                           │
  │  4. ReplicaSet Controller รับ Event ใหม่                       │
  │  5. สร้าง Pod ผ่าน API Server                                  │
  └───────────────────────────┬─────────────────────────────────────┘
                               │ Watch Event (Pod Created, Unscheduled)
                               ▼
Step 5: Scheduler (kube-scheduler)
  ┌─────────────────────────────────────────────────────────────────┐
  │  Scheduler เห็น Pod ที่ยังไม่ได้รับการ Schedule:               │
  │                                                                   │
  │  Filtering Phase:                                                │
  │  ├── PodFitsResources: Node มี CPU/Memory พอหรือไม่             │
  │  ├── PodFitsHostPorts: Port ไม่ชนกัน                            │
  │  ├── NoVolumeZoneConflict: Volume อยู่ใน Zone เดียวกับ Node     │
  │  ├── NodeAffinity: ตรงกับ Node Labels หรือไม่                   │
  │  ├── TaintToleration: Pod Tolerate Taint ของ Node หรือไม่       │
  │  └── ผลลัพธ์: List of Feasible Nodes                            │
  │                                                                   │
  │  Scoring Phase:                                                  │
  │  ├── LeastRequestedPriority: Node ที่ใช้ Resource น้อยสุด       │
  │  ├── BalancedResourceAllocation: CPU/Memory สมดุล               │
  │  ├── NodeAffinityPriority: ตรงกับ Preferred Node Affinity       │
  │  ├── InterPodAffinityPriority: ตามที่ Pod ต้องการ              │
  │  └── ผลลัพธ์: Node ที่มีคะแนนสูงสุด                           │
  │                                                                   │
  │  Binding:                                                        │
  │  └── Update Pod.spec.nodeName = "worker-node-1"                 │
  └───────────────────────────┬─────────────────────────────────────┘
                               │ Watch Event (Pod Scheduled)
                               ▼
Step 6: kubelet
  ┌─────────────────────────────────────────────────────────────────┐
  │  kubelet บน worker-node-1 เห็น Pod ที่ Assign มา:             │
  │                                                                   │
  │  1. ดึง Pod Spec จาก API Server                                 │
  │  2. ตรวจสอบ Image ว่ามีใน Local หรือไม่                        │
  │  3. Pull Image จาก Registry (ถ้าไม่มี)                          │
  │  4. เรียก Container Runtime (containerd/CRI-O) สร้าง Container │
  │  5. Setup Network (เรียก CNI Plugin)                             │
  │  6. Setup Volumes                                                │
  │  7. รัน Init Containers (ถ้ามี)                                  │
  │  8. รัน Main Containers                                          │
  │  9. รัน Liveness/Readiness Probes                               │
  │  10. Report Status กลับ API Server                              │
  └───────────────────────────┬─────────────────────────────────────┘
                               │ Status Update
                               ▼
Step 7: API Server → etcd → Status Update
  ┌─────────────────────────────────────────────────────────────────┐
  │  API Server รับ Status Update:                                   │
  │  Pod Status: Running                                             │
  │  Pod IP: 10.244.1.5                                             │
  │  Container Status: Running                                       │
  │                                                                   │
  │  บันทึกลง etcd                                                  │
  └─────────────────────────────────────────────────────────────────┘
```

### Detailed Authentication Flow

```
Authentication Methods ใน Kubernetes:

1. X.509 Client Certificates:
   ┌─────────────────────────────────────────────────────────────┐
   │  Client Certificate:                                         │
   │  - Subject: CN=admin,O=system:masters                       │
   │  - CN = Username                                             │
   │  - O = Group                                                 │
   │                                                              │
   │  API Server:                                                 │
   │  - ตรวจสอบ Certificate ด้วย CA Certificate                  │
   │  - Extract CN และ O สำหรับ RBAC                             │
   └─────────────────────────────────────────────────────────────┘

2. Bearer Tokens (ServiceAccount):
   ┌─────────────────────────────────────────────────────────────┐
   │  ServiceAccount Token (JWT):                                 │
   │  Header: {"alg":"RS256","typ":"JWT"}                        │
   │  Payload: {                                                  │
   │    "iss": "kubernetes/serviceaccount",                      │
   │    "kubernetes.io/serviceaccount/namespace": "default",     │
   │    "kubernetes.io/serviceaccount/service-account.name": "sa"│
   │  }                                                           │
   │  Signature: RSA256(base64(header)+"."+base64(payload), key) │
   └─────────────────────────────────────────────────────────────┘

3. OIDC (External Identity Provider):
   ┌─────────────────────────────────────────────────────────────┐
   │  User → OIDC Provider (Dex, Keycloak, etc.)                 │
   │       ← ID Token (JWT)                                       │
   │  User → API Server (Authorization: Bearer <id_token>)       │
   │  API Server → Validate Token Signature                       │
   │  API Server → Extract Claims (sub, groups, email)           │
   └─────────────────────────────────────────────────────────────┘
```

### Admission Control Pipeline

```
Admission Control ละเอียด:

Request
  │
  ▼
┌──────────────────────────────────────────────────────────────────┐
│              Mutating Admission Webhooks                          │
│                                                                    │
│  1. DefaultIngressClass → Set default IngressClass               │
│  2. MutatingAdmissionWebhook → Custom Webhooks                  │
│     - Istio: Inject sidecar containers                           │
│     - OPA/Gatekeeper: Add labels                                 │
│     - Cert-manager: Inject certificates                          │
│  3. PodPreset (deprecated) → Inject env vars                    │
└──────────────────────────────┬───────────────────────────────────┘
                                │ Modified Object
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│              Object Schema Validation                             │
│              (ตรวจสอบ Schema ตาม API Spec)                       │
└──────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│              Validating Admission Webhooks                        │
│                                                                    │
│  1. ValidatingAdmissionWebhook → Custom Validation              │
│     - OPA/Gatekeeper: Policy enforcement                         │
│       * ต้องมี Resource Limits                                   │
│       * ต้องไม่ใช้ latest tag                                    │
│       * ต้องมี specific Labels                                    │
│     - Falco: Security policies                                   │
│  2. PodSecurity → Check Pod Security Standards                  │
└──────────────────────────────┬───────────────────────────────────┘
                                │ Approved Object
                                ▼
                            etcd Store
```

---

## Object Relationships Diagram

### Core Object Hierarchy

```
Kubernetes Object Relationships:

Namespace
│
├── Deployment
│   │
│   └── ReplicaSet
│       │
│       └── Pod (1..N)
│           │
│           ├── Container (1..N)
│           │   ├── Image
│           │   ├── Resources (CPU/Memory)
│           │   ├── Env Vars
│           │   └── Volume Mounts
│           │
│           ├── Init Containers
│           ├── Ephemeral Containers
│           ├── Volumes
│           └── Service Account
│
├── StatefulSet
│   └── Pod (1..N) [stable network identity]
│       └── PersistentVolumeClaim (per-pod)
│
├── DaemonSet
│   └── Pod (1 per Node)
│
├── Job
│   └── Pod (1..N, runs to completion)
│
├── CronJob
│   └── Job (created on schedule)
│       └── Pod
│
├── Service
│   ├── ClusterIP
│   ├── NodePort
│   ├── LoadBalancer
│   └── ExternalName
│
├── Ingress
│   └── References Services
│
├── ConfigMap
│   └── Used by Pods (env, volume)
│
├── Secret
│   └── Used by Pods (env, volume)
│
├── ServiceAccount
│   ├── Secret (Token)
│   └── Used by Pods
│
└── PersistentVolumeClaim (PVC)
    └── Bound to PersistentVolume (PV)
```

### Network Object Relationships

```
Network Flow:

Internet → LoadBalancer Service → NodePort → ClusterIP → Pod

    ┌──────────┐
    │ Internet │
    └────┬─────┘
         │
    ┌────▼─────────────────────────────────────────────────────┐
    │                    Cloud Load Balancer                    │
    │              (AWS ALB / GCP LB / Azure LB)               │
    └────┬─────────────────────────────────────────────────────┘
         │ Port 80/443
    ┌────▼─────────────────────────────────────────────────────┐
    │                    Ingress Controller                     │
    │                  (nginx / traefik / etc.)                 │
    │                                                           │
    │  Rules:                                                   │
    │  myapp.com/api → backend-service:8080                    │
    │  myapp.com/    → frontend-service:80                     │
    └────┬──────────────────────────┬──────────────────────────┘
         │                          │
    ┌────▼──────────┐          ┌────▼──────────┐
    │ backend       │          │ frontend      │
    │ Service       │          │ Service       │
    │ (ClusterIP)   │          │ (ClusterIP)   │
    │ 10.96.1.100   │          │ 10.96.1.101   │
    └────┬──────────┘          └────┬──────────┘
         │                          │
    ┌────▼────────────────────┐   ┌────▼────────────────────┐
    │  Pod     Pod     Pod    │   │  Pod     Pod            │
    │ 10.244.0.10.244.0.10.244.0.│ 10.244.1.10.244.1.     │
    │  .5      .6      .7    │   │  .5      .6            │
    └─────────────────────────┘   └─────────────────────────┘
```

### Storage Object Relationships

```
Storage Hierarchy:

StorageClass (ผู้ดูแล Cloud Storage)
│
├── PersistentVolume (PV) ← Administrator สร้าง หรือ Dynamic Provisioning
│   ├── Capacity: 100Gi
│   ├── AccessModes: ReadWriteOnce
│   ├── ReclaimPolicy: Delete/Retain
│   └── VolumeSource: AWS EBS / GCP PD / NFS / etc.
│
└── PersistentVolumeClaim (PVC) ← Developer สร้าง
    ├── Requests: 10Gi
    ├── AccessModes: ReadWriteOnce
    └── Bound to: PV ที่ตรงกัน
        │
        └── Used by Pod
            └── Volume Mount: /data → PVC
```

---

## Kubernetes API Groups

### API Group Structure

```
Kubernetes API Groups:

REST Path: /apis/<group>/<version>/<resource>

Core API Group (ไม่มีชื่อ group):
  /api/v1/pods
  /api/v1/services
  /api/v1/configmaps
  /api/v1/secrets
  /api/v1/namespaces
  /api/v1/nodes
  /api/v1/persistentvolumes
  /api/v1/persistentvolumeclaims
  /api/v1/serviceaccounts
  /api/v1/events

Named API Groups:
  /apis/apps/v1/
    ├── deployments
    ├── replicasets
    ├── statefulsets
    ├── daemonsets
    └── controllerrevisions

  /apis/batch/v1/
    ├── jobs
    └── cronjobs

  /apis/networking.k8s.io/v1/
    ├── ingresses
    ├── networkpolicies
    └── ingressclasses

  /apis/rbac.authorization.k8s.io/v1/
    ├── clusterroles
    ├── clusterrolebindings
    ├── roles
    └── rolebindings

  /apis/storage.k8s.io/v1/
    ├── storageclasses
    ├── persistentvolumes
    └── volumeattachments

  /apis/autoscaling/v2/
    └── horizontalpodautoscalers

  /apis/policy/v1/
    └── poddisruptionbudgets

  /apis/apiextensions.k8s.io/v1/
    └── customresourcedefinitions (CRDs)
```

### API Versioning

```
Kubernetes API Stability Levels:

Alpha (v1alpha1, v1alpha2):
┌───────────────────────────────────────────────────────────────┐
│  - Feature ใหม่ ยังไม่ Stable                                 │
│  - อาจเปลี่ยน API ใน Minor Version                           │
│  - ไม่แนะนำใช้ใน Production                                   │
│  - ต้อง Enable ด้วย Feature Gate                              │
│  Example: v1alpha1, v2alpha1                                  │
└───────────────────────────────────────────────────────────────┘

Beta (v1beta1, v2beta1):
┌───────────────────────────────────────────────────────────────┐
│  - ใกล้ Stable แล้ว                                           │
│  - ได้รับการ Test แล้ว                                        │
│  - API อาจเปลี่ยนในอนาคต (backward compatible)               │
│  - ใช้ใน Production ได้แต่ระวัง                               │
│  Example: v1beta1, v2beta2                                    │
└───────────────────────────────────────────────────────────────┘

Stable/GA (v1, v2):
┌───────────────────────────────────────────────────────────────┐
│  - Production-ready                                            │
│  - API ไม่เปลี่ยนแบบ Breaking Change                         │
│  - ได้รับ Long-term Support                                   │
│  - แนะนำใช้ใน Production                                      │
│  Example: v1, v2                                              │
└───────────────────────────────────────────────────────────────┘

API Deprecation Policy:
- GA API: ไม่ Remove ก่อน 12 เดือน
- Beta API: ไม่ Remove ก่อน 9 เดือน / 3 releases
- Alpha API: Remove ได้เลยใน Minor Version
```

### Custom Resource Definitions (CRDs)

```yaml
# ตัวอย่าง CRD สำหรับ Application
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.mycompany.io
spec:
  group: mycompany.io
  names:
    kind: Application
    listKind: ApplicationList
    plural: applications
    singular: application
    shortNames:
    - app
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["image", "replicas"]
            properties:
              image:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 100
              port:
                type: integer
              env:
                type: array
                items:
                  type: object
                  properties:
                    name:
                      type: string
                    value:
                      type: string
    additionalPrinterColumns:
    - name: Image
      type: string
      jsonPath: .spec.image
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

```yaml
# ใช้ CRD ที่สร้างไว้
apiVersion: mycompany.io/v1
kind: Application
metadata:
  name: my-web-app
spec:
  image: nginx:1.25
  replicas: 3
  port: 80
  env:
  - name: ENV
    value: production
```

---

## etcd Data Model

### etcd Key Structure ใน Kubernetes

```
etcd เก็บข้อมูล Kubernetes ในรูปแบบ Key-Value:

Key Pattern: /registry/<group>/<resource>/<namespace>/<name>

Core Resources:
  /registry/pods/default/my-pod
  /registry/services/default/my-service
  /registry/configmaps/default/my-config
  /registry/secrets/default/my-secret
  /registry/namespaces/default
  /registry/nodes/worker-node-1

Apps Group:
  /registry/apps/deployments/default/my-deployment
  /registry/apps/replicasets/default/my-rs-xxx
  /registry/apps/statefulsets/default/my-stateful

Events:
  /registry/events/default/my-pod.xxxxx

RBAC:
  /registry/rbac.authorization.k8s.io/clusterroles/admin
  /registry/rbac.authorization.k8s.io/clusterrolebindings/cluster-admin

Leases (Leader Election):
  /registry/leases/kube-system/kube-controller-manager
  /registry/leases/kube-system/kube-scheduler
```

### etcd Watch Mechanism

```
Watch Mechanism (Heartbeat of Kubernetes):

1. API Server Watch etcd:
   ┌────────────────────────────────────────────────────────────┐
   │  API Server                                                 │
   │  ├── watch /registry/pods/...                              │
   │  ├── watch /registry/apps/deployments/...                  │
   │  └── watch /registry/nodes/...                             │
   └────────────────────────────────────────────────────────────┘

2. etcd ส่ง Watch Events:
   ┌────────────────────────────────────────────────────────────┐
   │  Event Types:                                               │
   │  ADDED   → Object ถูกสร้างใหม่                            │
   │  MODIFIED → Object ถูกแก้ไข                               │
   │  DELETED  → Object ถูกลบ                                  │
   └────────────────────────────────────────────────────────────┘

3. API Server กระจาย Events:
   ┌────────────────────────────────────────────────────────────┐
   │  API Server Watch Cache:                                    │
   │  - Informer ของ Controller Manager                        │
   │  - Informer ของ Scheduler                                  │
   │  - Informer ของ kubelet (Node polling)                     │
   └────────────────────────────────────────────────────────────┘
```

### etcd Cluster ใน Production

```
etcd HA Setup (3 หรือ 5 nodes):

Minimum HA: 3 nodes (quorum = 2)
Recommended: 5 nodes (quorum = 3)

┌──────────┐     ┌──────────┐     ┌──────────┐
│ etcd-1   │◄────►│ etcd-2   │◄────►│ etcd-3   │
│ (leader) │     │(follower)│     │(follower)│
│ 2380/tcp │     │ 2380/tcp │     │ 2380/tcp │
└──────────┘     └──────────┘     └──────────┘
      ▲                 ▲                ▲
      │                 │                │
      └─────────────────┴────────────────┘
                  API Server listens
                     2379/tcp

Quorum Formula:
  - 3 nodes: quorum = 2, รับ Node failure ได้ 1
  - 5 nodes: quorum = 3, รับ Node failure ได้ 2
  - 7 nodes: quorum = 4, รับ Node failure ได้ 3

etcd Backup (สำคัญมาก!):
  # ทำ Backup ทุกวัน
  ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Workshop: ดู Cluster State ผ่าน API Server โดยตรง

### Setup: เข้าถึง API Server

```bash
# วิธีที่ 1: kubectl proxy (ง่ายสุด)
kubectl proxy --port=8001 &

# ทดสอบ
curl http://localhost:8001/api/v1/pods

# วิธีที่ 2: ใช้ Service Account Token
TOKEN=$(kubectl create token default)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
CACERT=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.certificate-authority}')

# ถ้าใช้ minikube
APISERVER=$(minikube ip):8443

# API Call โดยตรง
curl -X GET \
  -H "Authorization: Bearer $TOKEN" \
  --cacert $CACERT \
  $APISERVER/api/v1/namespaces

# วิธีที่ 3: ใช้ kubectl --v flag (verbose)
kubectl get pods -v=8 2>&1 | grep "GET\|POST\|PATCH"
```

### สำรวจ API Groups

```bash
# ดู API Groups ทั้งหมด
curl http://localhost:8001/apis

# ดู Core API (v1)
curl http://localhost:8001/api/v1

# ดู apps/v1
curl http://localhost:8001/apis/apps/v1

# ดู Resources ใน apps/v1
curl http://localhost:8001/apis/apps/v1/namespaces/default/deployments

# ดู Resources ทั้งหมดที่ Kubernetes รองรับ
kubectl api-resources

# ดู API Versions
kubectl api-versions

# ดู Specific Resource
kubectl explain deployment
kubectl explain deployment.spec
kubectl explain deployment.spec.template.spec.containers
```

### ดู Pods ผ่าน API โดยตรง

```bash
# List all Pods
curl http://localhost:8001/api/v1/pods

# List Pods ใน Namespace
curl http://localhost:8001/api/v1/namespaces/default/pods

# ดู Specific Pod
curl http://localhost:8001/api/v1/namespaces/default/pods/my-pod

# Filter ด้วย Label Selector
curl "http://localhost:8001/api/v1/namespaces/default/pods?labelSelector=app=my-app"

# Field Selector
curl "http://localhost:8001/api/v1/namespaces/default/pods?fieldSelector=status.phase=Running"

# Watch Events
curl "http://localhost:8001/api/v1/namespaces/default/pods?watch=true"
```

### สร้าง Resource ผ่าน API โดยตรง

```bash
# สร้าง Pod ผ่าน API
curl -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8001/api/v1/namespaces/default/pods \
  -d '{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "test-pod",
      "labels": {
        "app": "test"
      }
    },
    "spec": {
      "containers": [
        {
          "name": "nginx",
          "image": "nginx:alpine",
          "ports": [{"containerPort": 80}]
        }
      ]
    }
  }'

# ดู Pod ที่สร้าง
curl http://localhost:8001/api/v1/namespaces/default/pods/test-pod

# Patch Pod (PATCH)
curl -X PATCH \
  -H "Content-Type: application/merge-patch+json" \
  http://localhost:8001/api/v1/namespaces/default/pods/test-pod \
  -d '{"metadata": {"labels": {"version": "v1"}}}'

# ลบ Pod
curl -X DELETE \
  http://localhost:8001/api/v1/namespaces/default/pods/test-pod
```

### ดู Cluster State ผ่าน API

```bash
# ดู Nodes
curl http://localhost:8001/api/v1/nodes

# ดู Node Status ละเอียด
curl http://localhost:8001/api/v1/nodes/minikube | \
  python3 -c "import sys,json; data=json.load(sys.stdin); \
  print('Conditions:', json.dumps(data['status']['conditions'], indent=2))"

# ดู Cluster Info
kubectl cluster-info

# ดู Component Status
kubectl get componentstatuses
# Warning: v1 ComponentStatus is deprecated
# NAME                 STATUS    MESSAGE   ERROR
# scheduler            Healthy   ok
# controller-manager   Healthy   ok
# etcd-0               Healthy   ok

# ดู Events ทั้งหมด
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Watch Events แบบ Real-time
kubectl get events --all-namespaces -w

# ดู API Server Metrics
curl http://localhost:8001/metrics | grep apiserver_request_total | head -20
```

### ดู etcd Data โดยตรง (บน minikube)

```bash
# SSH เข้า minikube
minikube ssh

# ดู etcd Data
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get /registry/pods/default/ --prefix --keys-only

# ดู Pod Data (ดิบ)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  get /registry/pods/default/test-pod | strings

# ดู etcd Cluster Health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  endpoint health

# etcd Stats
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key \
  endpoint status --write-out=table
```

### Audit Logging

```bash
# ดู Audit Log บน minikube
minikube ssh
sudo cat /var/log/kubernetes/audit.log | head -20 | python3 -m json.tool

# ตัวอย่าง Audit Log Entry:
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "abc123",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": {
    "username": "system:serviceaccount:default:default",
    "groups": ["system:serviceaccounts", "system:authenticated"]
  },
  "sourceIPs": ["127.0.0.1"],
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "test-pod",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "code": 201
  },
  "requestReceivedTimestamp": "2024-01-15T10:30:00.000000Z",
  "stageTimestamp": "2024-01-15T10:30:00.050000Z"
}
```

---

## แบบฝึกหัดพร้อมเฉลย

### ข้อที่ 1: ติดตาม Request Flow

**โจทย์**: อธิบาย Flow เมื่อรัน `kubectl delete pod my-pod`

**เฉลย**:

```
kubectl delete pod my-pod Flow:

1. kubectl:
   - อ่าน kubeconfig
   - ส่ง DELETE /api/v1/namespaces/default/pods/my-pod

2. API Server:
   - Authentication: ตรวจสอบ credentials
   - Authorization: ตรวจสอบ RBAC (ต้องมี delete pods permission)
   - Admission: ValidatingWebhooks (ถ้ามี)
   - Set deletionTimestamp บน Pod
   - บันทึกใน etcd

3. etcd:
   - บันทึก Pod with deletionTimestamp
   - ส่ง Watch Event (MODIFIED) ไป API Server

4. Controller Manager:
   - Endpoints Controller: Remove Pod จาก Endpoints
   - ReplicaSet Controller (ถ้า Pod ถูก Manage): สร้าง Pod ใหม่

5. kubelet (บน Node ที่ Pod รัน):
   - ได้รับ Watch Event ว่า Pod มี deletionTimestamp
   - ส่ง SIGTERM ไป Container
   - รอ terminationGracePeriodSeconds (default 30s)
   - ถ้ายังไม่หยุด: ส่ง SIGKILL
   - รายงาน Status ว่า Container Terminated

6. API Server:
   - รับ Status Update จาก kubelet
   - ลบ Pod Object จาก etcd (finalizers ทั้งหมด removed)
   - ส่ง Watch Event (DELETED)

7. kube-proxy:
   - ได้รับ Event ว่า Endpoints เปลี่ยน
   - Update iptables rules (ลบ Pod IP ออก)
```

### ข้อที่ 2: API Group และ Version

**โจทย์**: ระบุ API Group และ Version สำหรับ Resources ต่อไปนี้:
- Deployment
- CronJob
- Ingress
- ClusterRole
- HorizontalPodAutoscaler

**เฉลย**:

```bash
# ตรวจสอบด้วย kubectl api-resources
kubectl api-resources | grep -E "Deployment|CronJob|Ingress|ClusterRole|HorizontalPodAutoscaler"

# ผลลัพธ์:
NAME                   SHORTNAMES   APIVERSION              NAMESPACED   KIND
deployments            deploy       apps/v1                 true         Deployment
cronjobs               cj           batch/v1                true         CronJob
ingresses              ing          networking.k8s.io/v1    true         Ingress
clusterroles                        rbac.authorization.k8s.io/v1   false   ClusterRole
horizontalpodautoscalers  hpa       autoscaling/v2          true         HorizontalPodAutoscaler

# ดู REST API Path:
# Deployment:  /apis/apps/v1/namespaces/{ns}/deployments
# CronJob:     /apis/batch/v1/namespaces/{ns}/cronjobs
# Ingress:     /apis/networking.k8s.io/v1/namespaces/{ns}/ingresses
# ClusterRole: /apis/rbac.authorization.k8s.io/v1/clusterroles
# HPA:         /apis/autoscaling/v2/namespaces/{ns}/horizontalpodautoscalers
```

### ข้อที่ 3: Scheduler Decision

**โจทย์**: Pod ต้องการ CPU: 500m, Memory: 1Gi มี Nodes ดังนี้:
- Node-1: Available CPU: 200m, Memory: 2Gi
- Node-2: Available CPU: 600m, Memory: 500Mi
- Node-3: Available CPU: 700m, Memory: 2Gi

Scheduler จะเลือก Node ไหน?

**เฉลย**:

```
Scheduler Analysis:

Filtering Phase:
- Node-1: CPU 200m < 500m ← ไม่ผ่าน Filter!
- Node-2: Memory 500Mi < 1Gi ← ไม่ผ่าน Filter!
- Node-3: CPU 700m ≥ 500m, Memory 2Gi ≥ 1Gi ← ผ่าน!

Feasible Nodes: [Node-3]

Scoring Phase:
เนื่องจากมีแค่ Node-3 ที่ผ่าน Filtering
ผลลัพธ์: Pod ถูก Schedule ไปยัง Node-3

ถ้าไม่มี Node ที่ผ่าน Filtering:
- Pod จะอยู่ในสถานะ Pending
- Event: "0/3 nodes are available: 1 Insufficient cpu, 1 Insufficient memory, 1 node(s) had taints"

kubectl describe pod my-pod | grep Events -A 10
# Events:
#   Warning  FailedScheduling  0/3 nodes are available...
```

### ข้อที่ 4: etcd Key Path

**โจทย์**: ระบุ etcd Key Path สำหรับ Resources ต่อไปนี้:
1. Pod ชื่อ "api-server" ใน namespace "production"
2. Service ชื่อ "frontend" ใน namespace "staging"
3. Node ชื่อ "worker-1"
4. ClusterRole ชื่อ "view"

**เฉลย**:

```
etcd Key Paths:

1. Pod "api-server" ใน "production":
   Key: /registry/pods/production/api-server
   
   ตรวจสอบ:
   etcdctl get /registry/pods/production/api-server

2. Service "frontend" ใน "staging":
   Key: /registry/services/specs/staging/frontend
   หรือ: /registry/services/staging/frontend
   
   ตรวจสอบ:
   etcdctl get /registry/services/specs/staging/frontend

3. Node "worker-1":
   Key: /registry/minions/worker-1
   (Nodes ใช้ "minions" ใน etcd เพราะ Legacy)
   
   ตรวจสอบ:
   etcdctl get /registry/minions/worker-1

4. ClusterRole "view":
   Key: /registry/rbac.authorization.k8s.io/clusterroles/view
   
   ตรวจสอบ:
   etcdctl get /registry/rbac.authorization.k8s.io/clusterroles/view

ดู Keys ทั้งหมด:
etcdctl get / --prefix --keys-only | sort | head -50
```

---

## สรุป Part 06

ใน Part นี้เราได้เรียนรู้:

1. **Request Flow ละเอียด** - kubectl → AuthN → AuthZ → Admission → etcd → Controller → Scheduler → kubelet
2. **Object Relationships** - ความสัมพันธ์ระหว่าง Deployment, ReplicaSet, Pod, Service, PVC
3. **API Groups** - Core API, Named API Groups, API Versioning (Alpha/Beta/Stable)
4. **etcd Data Model** - Key Structure, Watch Mechanism, HA Setup
5. **Workshop** - ดู Cluster State ผ่าน API Server โดยตรง, ดู etcd Data

### Checklist ก่อนไปต่อ

- [ ] เข้าใจ Request Flow ตั้งแต่ kubectl ถึง Container
- [ ] รู้ API Groups และ Version ของ Resources หลักๆ
- [ ] เข้าถึง API Server ผ่าน kubectl proxy ได้
- [ ] ดู etcd Data โดยตรงได้ (บน minikube)
- [ ] เข้าใจว่า etcd Watch Mechanism ทำงานอย่างไร
- [ ] ทำแบบฝึกหัดครบ 4 ข้อ

---

*ต่อไป: [Part 07: Control Plane Deep Dive](./part-07-control-plane.md)*
