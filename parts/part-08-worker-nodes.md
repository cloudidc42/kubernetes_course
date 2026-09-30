# Part 08: Worker Nodes - เจาะลึก

## สารบัญ
- [kubelet](#kubelet)
- [kube-proxy](#kube-proxy)
- [Container Runtime](#container-runtime)
- [Node Lifecycle](#node-lifecycle)
- [Workshop: ดู Node Status และ Components](#workshop-ดู-node-status-และ-components)

---

## kubelet

### ภาพรวม

**kubelet** คือ Node Agent ที่รันบนทุก Worker Node หน้าที่หลักคือทำให้ Container ที่ระบุใน Pod Spec รันอยู่เสมอ

```
kubelet ทำอะไรบ้าง:

API Server ─────────────────► kubelet
(Assign Pod to Node)              │
                                  ├── Pull Container Images
                                  ├── Create/Start Containers
                                  ├── Monitor Container Health
                                  ├── Run Probes (Liveness/Readiness)
                                  ├── Manage Volumes
                                  ├── Report Node Status
                                  └── Report Pod Status
```

### kubelet Architecture

```
kubelet ทำงานอย่างไร:

┌──────────────────────────────────────────────────────────┐
│                       kubelet                             │
│                                                          │
│  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ Pod Watcher  │  │ Sync Loop   │  │ Status Reporter│  │
│  │              │  │             │  │                │  │
│  │ Watch API    │  │ Ensure pods │  │ Update node    │  │
│  │ for pod      │  │ match spec  │  │ status to API  │  │
│  │ assignments  │  │             │  │                │  │
│  └──────────────┘  └─────────────┘  └────────────────┘  │
│                                                          │
│  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ Probe Manager│  │Volume Manager│  │  Image Manager │  │
│  │              │  │             │  │                │  │
│  │ Liveness     │  │ Mount/Unmount│  │ Pull/Prune    │  │
│  │ Readiness    │  │ Volumes      │  │ Images         │  │
│  │ Startup      │  │             │  │                │  │
│  └──────────────┘  └─────────────┘  └────────────────┘  │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              CRI (Container Runtime Interface)       │  │
│  │  containerd / CRI-O                                  │  │
│  └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Pod Sync Loop

```go
// Pseudocode: kubelet sync loop

for {
    // 1. Get desired pods for this node
    desiredPods := watchAPIServer()
    
    // 2. Get actual pods running
    actualPods := getRunningPods()
    
    // 3. Reconcile
    for pod in desiredPods {
        if !isRunning(pod) {
            // Pull image if needed
            pullImage(pod.image)
            // Create and start container
            createContainer(pod)
        }
        
        // Check health
        runProbes(pod)
        
        // Update status
        updatePodStatus(pod)
    }
    
    // Remove pods that shouldn't be running
    for pod in actualPods {
        if !inDesiredPods(pod) {
            stopContainer(pod)
        }
    }
    
    // Report node status
    updateNodeStatus()
    
    sleep(syncFrequency)
}
```

### kubelet Configuration

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Health and Monitoring
healthzBindAddress: 127.0.0.1
healthzPort: 10248

# Pod Settings
maxPods: 110
podCIDR: "192.168.0.0/16"
resolvConf: /etc/resolv.conf

# Authentication
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 2m0s
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt

# Authorization  
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s

# Resource Management
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"

evictionSoft:
  memory.available: "500Mi"
  nodefs.available: "15%"

evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "1m30s"

# Container Runtime
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
imageServiceEndpoint: unix:///var/run/containerd/containerd.sock

# Garbage Collection
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
containerLogMaxSize: "10Mi"
containerLogMaxFiles: 5

# Node Status
nodeStatusUpdateFrequency: 10s
nodeStatusReportFrequency: 5m0s

# CPU/Memory Management
cpuManagerPolicy: static
memoryManagerPolicy: Static
topologyManagerPolicy: single-numa-node
```

### Health Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: nginx
    
    # Startup Probe - ตรวจสอบว่า App เริ่มต้นสำเร็จหรือไม่
    # ถ้า Fail → Container ถูก Restart
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30    # ทดสอบ 30 ครั้ง
      periodSeconds: 10       # ทุก 10 วินาที (รวม 5 นาที)
    
    # Liveness Probe - ตรวจว่า Container ยังทำงานอยู่ไหม
    # ถ้า Fail → Container ถูก Restart
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: health-check
      initialDelaySeconds: 10
      periodSeconds: 30
      timeoutSeconds: 5
      successThreshold: 1
      failureThreshold: 3
    
    # Readiness Probe - ตรวจว่า Container พร้อมรับ Traffic หรือไม่
    # ถ้า Fail → Pod ถูกเอาออกจาก Service Endpoints
    readinessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - "redis-cli ping | grep PONG"
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
    
    # หรือใช้ TCP Socket Probe
    livenessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 15
      periodSeconds: 20
```

**Probe Types:**
```
HTTP Probe:
  - ส่ง HTTP GET Request
  - HTTP Status 200-399 = Success

TCP Probe:
  - ตรวจว่า Port เปิดอยู่ไหม
  - Connection Success = Success

Exec Probe:
  - Run command ใน Container
  - Exit Code 0 = Success

gRPC Probe:
  - ส่ง gRPC Health Check Request
  - Status SERVING = Success
```

### Resource Management

```yaml
# Resource Requests and Limits
spec:
  containers:
  - name: app
    resources:
      # Requests: ขอ Resource เพื่อ Scheduling
      # Node ต้องมี Resource อย่างน้อยเท่านี้
      requests:
        cpu: "250m"       # 250 millicores = 0.25 CPU core
        memory: "256Mi"   # 256 Mebibytes
      
      # Limits: จำกัด Resource สูงสุด
      # Container ใช้เกินนี้ไม่ได้
      limits:
        cpu: "1000m"      # 1 CPU core
        memory: "512Mi"   # 512 Mebibytes
```

**QoS Classes:**
```
QoS (Quality of Service) Classes:

1. Guaranteed (ดีที่สุด)
   - requests == limits สำหรับทุก Resource
   - ถ้า Node ขาด Memory → Evict Pods อื่นก่อน
   
2. Burstable (ปานกลาง)
   - มี requests แต่ requests != limits
   - ถ้า Node ขาด Memory → Evict หลัง BestEffort
   
3. BestEffort (แย่ที่สุด)
   - ไม่มี requests/limits เลย
   - ถ้า Node ขาด Memory → Evict ก่อนเลย
```

```yaml
# Guaranteed QoS
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"    # เท่ากับ requests
    memory: "256Mi"  # เท่ากับ requests

# Burstable QoS
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"    # ต่างจาก requests
    memory: "256Mi"

# BestEffort QoS
resources: {}  # ไม่มี requests/limits
```

---

## kube-proxy

### ภาพรวม

**kube-proxy** รันเป็น DaemonSet บนทุก Node จัดการ Network Rules สำหรับ Service Load Balancing

### Service Types และ kube-proxy

```
kube-proxy จัดการ:

1. ClusterIP (Internal)
   ─────────────────────────
   ClusterIP: 10.96.0.1:80
   
   iptables rules:
   KUBE-SVC → KUBE-SEP-1 (Pod 1)
              KUBE-SEP-2 (Pod 2)
              KUBE-SEP-3 (Pod 3)

2. NodePort (External via Node)
   ─────────────────────────────
   NodeIP:30000 → Service → Pods
   
   iptables rules:
   :30000 → KUBE-SVC → Pods

3. LoadBalancer
   ─────────────────────────────
   External-IP:80 → NodePort:30000 → Service → Pods
```

### iptables Mode (Default)

```bash
# ดู iptables Rules
sudo iptables -t nat -L KUBE-SERVICES | head -20

# ดู Service Rules
sudo iptables -t nat -L | grep KUBE-SVC

# ตัวอย่าง Rules สำหรับ Service nginx:
# Chain KUBE-SERVICES
# -A KUBE-SERVICES -d 10.96.0.1/32 -p tcp --dport 80
#   -j KUBE-SVC-NGINX

# Chain KUBE-SVC-NGINX (Load Balance ระหว่าง 3 Pods)
# -A KUBE-SVC-NGINX -m statistic --mode random --probability 0.33
#   -j KUBE-SEP-POD1
# -A KUBE-SVC-NGINX -m statistic --mode random --probability 0.5
#   -j KUBE-SEP-POD2
# -A KUBE-SVC-NGINX -j KUBE-SEP-POD3

# Chain KUBE-SEP-POD1 (DNAT ไปยัง Pod IP)
# -A KUBE-SEP-POD1 -p tcp -j DNAT --to-destination 192.168.1.5:8080
```

### IPVS Mode (Better for Large Scale)

```bash
# Enable IPVS Mode ใน Minikube
minikube start --extra-config=kube-proxy.mode=ipvs

# ดู IPVS Rules
sudo ipvsadm -l --numeric

# ตัวอย่าง Output:
# IP Virtual Server version 1.2.1
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
# TCP  10.96.0.1:80 rr
#   -> 192.168.1.5:8080             Masq    1      0          0
#   -> 192.168.1.6:8080             Masq    1      0          0
#   -> 192.168.1.7:8080             Masq    1      0          0
```

**IPVS Scheduling Algorithms:**
```
rr  - Round Robin (default)
lc  - Least Connection
dh  - Destination Hashing
sh  - Source Hashing
sed - Shortest Expected Delay
nq  - Never Queue
```

### Service Discovery Flow

```
Pod A ต้องการคุยกับ Service B:

1. Pod A ส่ง Request ไปยัง service-b:8080
2. DNS Resolution: service-b → 10.96.0.100
3. Request ออกจาก Pod A ผ่าน veth
4. Packet ถึง Node network
5. iptables/IPVS interceptor ดักจับ
6. DNAT: 10.96.0.100:8080 → 192.168.1.5:8080 (Pod B IP)
7. Packet ถึง Pod B

ทั้งหมดนี้ทำงานใน Kernel Space (ไม่ผ่าน User Space)
= Performance สูง
```

---

## Container Runtime

### Container Runtime Interface (CRI)

```
Kubernetes ใช้ CRI เพื่อ Abstraction:

Kubernetes (kubelet)
       │
       │ gRPC via CRI
       │
       ▼
CRI Implementation
├── containerd (กว้างที่สุด, default ใน most distros)
├── CRI-O (lightweight, OpenShift default)
└── Docker Engine (via cri-dockerd, deprecated)
       │
       │ OCI (Open Container Initiative)
       │
       ▼
OCI Runtime
└── runc (de facto standard)
    └── Linux Kernel (Namespaces, Cgroups)
```

### containerd

```
containerd Architecture:

┌─────────────────────────────────────────────────────────┐
│                      containerd                          │
│                                                         │
│  ┌────────────────┐  ┌───────────────┐  ┌───────────┐  │
│  │  Image Service │  │Container Svc  │  │  Snapshots│  │
│  │                │  │               │  │           │  │
│  │  Pull/Push     │  │  Create/Start │  │  Overlay  │  │
│  │  Images        │  │  Stop/Remove  │  │  FS       │  │
│  └────────────────┘  └───────────────┘  └───────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Content Store                        │   │
│  │  Store OCI Images, Manifests, Configs            │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                   shim                            │   │
│  │  Process per container, survives containerd restart│  │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                   runc                            │   │
│  │  OCI Runtime - Actually creates Linux containers │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

```bash
# ตรวจสอบ containerd
systemctl status containerd

# ดู Version
containerd --version
ctr version

# ดู Namespaces
ctr namespaces list
# NAME    LABELS
# k8s.io  (Kubernetes ใช้ Namespace นี้)
# moby    (Docker ใช้ Namespace นี้)

# ดู Images (ใน k8s.io namespace)
ctr -n k8s.io images list

# ดู Containers
ctr -n k8s.io containers list

# ดู Tasks (running processes)
ctr -n k8s.io tasks list
```

### crictl (CRI CLI)

```bash
# ติดตั้ง crictl
VERSION="v1.28.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz | tar -xz
sudo mv crictl /usr/local/bin/

# Config
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# Commands
crictl ps                    # ดู Running Containers
crictl ps -a                 # ดูทุก Containers (รวม Stopped)
crictl pods                  # ดู Pods
crictl images                # ดู Images
crictl pull nginx:latest     # Pull Image
crictl rmi nginx:latest      # Remove Image

# Container Operations
crictl logs <container-id>   # ดู Logs
crictl exec -it <id> sh      # exec ใน Container
crictl inspect <id>          # ดูรายละเอียด

# Pod Operations
crictl inspectp <pod-id>     # ดูรายละเอียด Pod
crictl stopp <pod-id>        # Stop Pod
crictl rmp <pod-id>          # Remove Pod (ต้อง stop ก่อน)

# Stats
crictl stats                 # CPU/Memory usage
```

### Linux Namespaces ใน Container

```
Container = Process ใน Linux Namespaces:

┌──────────────────────────────────────────────────────┐
│                    Container                          │
│                                                      │
│  Namespace Types:                                    │
│                                                      │
│  PID Namespace:                                      │
│    Container เห็นแค่ processes ของตัวเอง             │
│    PID 1 ใน Container = main process                │
│                                                      │
│  Network Namespace:                                  │
│    Container มี Network interfaces ของตัวเอง         │
│    (eth0 ของตัวเอง, routing table ของตัวเอง)         │
│                                                      │
│  Mount Namespace:                                    │
│    Container มี Filesystem tree ของตัวเอง            │
│    ไม่เห็น Host filesystem                           │
│                                                      │
│  UTS Namespace:                                      │
│    Container มี Hostname ของตัวเอง                   │
│                                                      │
│  IPC Namespace:                                      │
│    Container มี Inter-process Communication ของตัวเอง│
│                                                      │
│  User Namespace:                                     │
│    Container มี User/Group mapping ของตัวเอง          │
│    UID 0 ใน Container ≠ UID 0 บน Host               │
└──────────────────────────────────────────────────────┘
```

### Linux Cgroups

```
Cgroups (Control Groups) จำกัด Resource:

┌──────────────────────────────────────────────────────┐
│                     Cgroups v2                        │
│                                                      │
│  /sys/fs/cgroup/                                     │
│  └── kubepods/                                       │
│      ├── besteffort/                                 │
│      │   └── pod-abc123/                             │
│      │       ├── cpu.max: 500000 1000000  (50% CPU)  │
│      │       └── memory.max: 268435456   (256MB)     │
│      │                                               │
│      ├── burstable/                                  │
│      │   └── pod-def456/                             │
│      │       ├── cpu.max: 2000000 1000000 (200% CPU) │
│      │       └── memory.max: 536870912   (512MB)     │
│      │                                               │
│      └── guaranteed/                                 │
│          └── pod-ghi789/                             │
│              ├── cpu.max: 1000000 1000000 (100% CPU) │
│              └── memory.max: 1073741824  (1GB)       │
└──────────────────────────────────────────────────────┘

ถ้า Container ใช้ Memory เกิน limit:
  → Linux OOM Killer จะ Kill Process
  → kubelet ตรวจพบ Container ล้ม
  → Restart Container (ถ้า restartPolicy = Always)
```

---

## Node Lifecycle

### Node States

```
Node Lifecycle:

New Node Join:
  1. kubelet เริ่มทำงาน
  2. kubelet Register Node กับ API Server
  3. API Server สร้าง Node Object
  4. Node Status = NotReady
  5. kubelet อัพเดท Node Status
  6. Node Status = Ready ✓

Node Conditions:
  Ready           = Node พร้อมรับ Pods
  MemoryPressure  = RAM น้อย
  DiskPressure    = Disk น้อย
  PIDPressure     = Process จำนวนมาก
  NetworkUnavailable = Network มีปัญหา

Node Lifecycle:
  Added → Healthy → 
    MemoryPressure → Evict BestEffort Pods
    DiskPressure   → Evict Pods, Clean Images
    NotReady       → Evict all Pods (after timeout)
    Terminating    → Remove from Cluster
```

### Node Taints

```bash
# Taint Node (ป้องกัน Pods มา Schedule)
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 key=value:NoExecute
kubectl taint nodes node1 key=value:PreferNoSchedule

# ลบ Taint
kubectl taint nodes node1 key=value:NoSchedule-

# ดู Taints
kubectl describe node node1 | grep Taint

# ตัวอย่าง Taints ที่ Kubernetes ใช้:
# node.kubernetes.io/not-ready:NoExecute
# node.kubernetes.io/unreachable:NoExecute
# node.kubernetes.io/memory-pressure:NoSchedule
# node.kubernetes.io/disk-pressure:NoSchedule
# node.kubernetes.io/pid-pressure:NoSchedule
# node.kubernetes.io/network-unavailable:NoSchedule
# node.kubernetes.io/unschedulable:NoSchedule
```

```yaml
# Pod Toleration
spec:
  tolerations:
  # Tolerate specific taint
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  # Tolerate any taint with this key
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300   # ทน 300 วินาที ก่อน Evict
```

### Node Maintenance (Cordon & Drain)

```bash
# Cordon: หยุดรับ Pods ใหม่ (Pods ที่มีอยู่ยังรัน)
kubectl cordon node1

# ดูสถานะ
kubectl get nodes
# NAME    STATUS                     ROLES   AGE
# node1   Ready,SchedulingDisabled   worker  10d

# Drain: ย้าย Pods ทั้งหมดออกจาก Node
kubectl drain node1 \
  --ignore-daemonsets \    # ไม่ Drain DaemonSet Pods
  --delete-emptydir-data \ # ลบ emptyDir volumes
  --force                  # Force Drain แม้มี Pods ที่ไม่มี Controller

# ทำ Maintenance...
# อัพเดท OS, Kernel, หรือ ทำ Hardware Maintenance

# Uncordon: เปิดรับ Pods ใหม่อีกครั้ง
kubectl uncordon node1
```

### Node Auto-provisioning (Cluster Autoscaler)

```
Cluster Autoscaler ทำงานอย่างไร:

1. Pod Pending เพราะไม่มี Node ที่มี Resource พอ
   
   ┌─────────────────────────────────────────┐
   │ Pod: Pending                            │
   │ Reason: Insufficient CPU               │
   └─────────────────────────────────────────┘

2. Cluster Autoscaler ตรวจพบ

3. Request เพิ่ม Node จาก Cloud Provider
   (AWS: เพิ่ม EC2 instance)
   (GCP: เพิ่ม GCE instance)

4. Node ใหม่ Join Cluster

5. Pod ถูก Schedule บน Node ใหม่

   ┌─────────────────────────────────────────┐
   │ Pod: Running                            │
   │ Node: new-node-1                        │
   └─────────────────────────────────────────┘

6. เมื่อ Load ลด → ลบ Node ที่ไม่ได้ใช้
   (ย้าย Pods ออก → ลบ Node)
```

---

## Workshop: ดู Node Status และ Components

### เป้าหมาย

เรียนรู้การ Monitor และ Debug Worker Node

### ขั้นตอน 1: เริ่ม Cluster

```bash
minikube start --driver=docker --cpus=2 --memory=4096
kubectl get nodes -o wide
```

### ขั้นตอน 2: ดู Node Details

```bash
# ดูรายละเอียด Node ทั้งหมด
kubectl describe node minikube

# Key Information:
# - Capacity: Total Resources
# - Allocatable: Available for Pods (after system reserved)
# - System Info: OS, Kernel, Container Runtime
# - Conditions: Ready, MemoryPressure, etc.
# - Addresses: IP Addresses
# - Non-terminated Pods: Pods ที่กำลังรัน

# ดูแค่ Resources
kubectl describe node minikube | grep -A8 "Capacity:"
kubectl describe node minikube | grep -A8 "Allocatable:"

# ดู Resource Usage
kubectl top node minikube
```

### ขั้นตอน 3: ดู kubelet

```bash
# SSH เข้า Node
minikube ssh

# ดู kubelet Process
ps aux | grep kubelet | grep -v grep

# ดู kubelet Flags
cat /var/lib/kubelet/config.yaml

# ดู kubelet Logs (systemd)
sudo journalctl -u kubelet --no-pager -n 50

# ดู kubelet Metrics
curl -s http://localhost:10255/metrics | head -30
# หรือ
curl -s http://localhost:10248/healthz
# ok

# ดู kubelet Pods
curl -s http://localhost:10255/pods | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for pod in data['items']:
    name = pod['metadata']['name']
    phase = pod['status']['phase']
    print(f'{name}: {phase}')"

exit
```

### ขั้นตอน 4: ดู Container Runtime

```bash
# SSH เข้า Node
minikube ssh

# ดู containerd Status
sudo systemctl status containerd

# ดู containerd Version
containerd --version

# ดู Images
sudo ctr -n k8s.io images list | head -10

# ดู Containers ทั้งหมด
sudo ctr -n k8s.io containers list

# ดู Snapshots (Filesystem layers)
sudo ctr -n k8s.io snapshots list | head -10

# ใช้ crictl (ง่ายกว่า)
sudo crictl ps
sudo crictl images | head -10

# ดู Container Details
CONTAINER_ID=$(sudo crictl ps | grep nginx | awk '{print $1}' | head -1)
if [ ! -z "$CONTAINER_ID" ]; then
  sudo crictl inspect $CONTAINER_ID | head -50
fi

exit
```

### ขั้นตอน 5: ดู kube-proxy

```bash
# ดู kube-proxy Pods
kubectl get pods -n kube-system | grep kube-proxy

# ดู kube-proxy Logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# SSH เข้า Node
minikube ssh

# ดู iptables Rules
sudo iptables -t nat -L KUBE-SERVICES | head -20
sudo iptables -t nat -L | grep KUBE | head -30

# นับจำนวน Rules
echo "Total iptables rules: $(sudo iptables -t nat -L | wc -l)"

exit
```

### ขั้นตอน 6: Deploy Application และ Monitor Resources

```bash
# Deploy Application ที่กำหนด Resources
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
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
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
EOF

# รอให้ Pods พร้อม
kubectl wait --for=condition=ready pod -l app=resource-demo --timeout=60s

# ดู Resource Usage
kubectl top pods -l app=resource-demo
kubectl top nodes
```

### ขั้นตอน 7: ทดสอบ Health Probes

```bash
# Deploy App ที่ Probe ล้มเหลว
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-fail
spec:
  containers:
  - name: app
    image: nginx
    livenessProbe:
      httpGet:
        path: /this-path-does-not-exist
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
EOF

# ดูว่า Pod ถูก Restart
kubectl get pod probe-fail -w
# RESTARTS จะเพิ่มขึ้น

# ดู Events
kubectl describe pod probe-fail | tail -20
# Events:
#   Warning  Unhealthy  ... Liveness probe failed: HTTP probe failed with statuscode: 404
#   Normal   Killing    ... Container probe-fail failed liveness or startup probe, will be restarted

# ทำความสะอาด
kubectl delete pod probe-fail
```

### ขั้นตอน 8: Node Conditions Monitor

```bash
# ดู Node Conditions ทั้งหมด
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
REASON:.status.conditions[-1].reason,\
MESSAGE:.status.conditions[-1].message

# ดู Detailed Conditions
kubectl get nodes -o json | python3 -c "
import sys, json
data = json.load(sys.stdin)
for node in data['items']:
    print(f'Node: {node[\"metadata\"][\"name\"]}')
    for cond in node['status']['conditions']:
        status = '✅' if cond['status'] == 'True' else '❌'
        print(f'  {status} {cond[\"type\"]}: {cond[\"reason\"]}')
    print()"
```

### ขั้นตอน 9: ทดสอบ Cordon/Drain

```bash
# ดู Node ปัจจุบัน
kubectl get nodes

# Cordon Node
kubectl cordon minikube
kubectl get nodes
# STATUS: Ready,SchedulingDisabled

# ลอง Deploy - Pod จะ Pending
kubectl run test-pod --image=nginx
kubectl get pod test-pod
# STATUS: Pending

# Describe Pod ดูเหตุผล
kubectl describe pod test-pod | grep "Events:" -A5
# Warning  FailedScheduling  0/1 nodes are available: 1 node(s) were unschedulable.

# Uncordon Node
kubectl uncordon minikube
kubectl get nodes
# STATUS: Ready

# Pod จะถูก Schedule แล้ว
kubectl get pod test-pod
# STATUS: Running

# ทำความสะอาด
kubectl delete pod test-pod
```

### ขั้นตอน 10: ดู Cgroups

```bash
# SSH เข้า Node
minikube ssh

# ดู Cgroups ของ Kubernetes
ls /sys/fs/cgroup/kubepods/

# ดู CPU Limits
for pod in $(ls /sys/fs/cgroup/kubepods/burstable/ 2>/dev/null | head -5); do
  echo "Pod: $pod"
  cat "/sys/fs/cgroup/kubepods/burstable/$pod/cpu.max" 2>/dev/null || echo "N/A"
done

# ดู Memory Limits
for pod in $(ls /sys/fs/cgroup/kubepods/guaranteed/ 2>/dev/null | head -5); do
  echo "Pod: $pod"
  cat "/sys/fs/cgroup/kubepods/guaranteed/$pod/memory.max" 2>/dev/null || echo "N/A"
done

exit
```

### ทำความสะอาด

```bash
kubectl delete deployment resource-demo
minikube stop
```

---

## สรุป

```
Worker Node Summary:

kubelet:
  - Node Agent
  - Pod Sync Loop
  - Health Probes (Liveness/Readiness/Startup)
  - Resource Management
  - CRI Integration

kube-proxy:
  - Network Rules
  - Service Load Balancing
  - iptables / IPVS
  - ClusterIP, NodePort, LoadBalancer

Container Runtime (containerd):
  - Container Lifecycle
  - Image Management
  - OCI Standard
  - runc (actual container creation)

Node Lifecycle:
  - Register → Ready → Draining → Removed
  - Cordon: หยุดรับ Pods ใหม่
  - Drain: ย้าย Pods ออก
  - Maintenance Window
```

---

## แบบฝึกหัด

1. สร้าง Pod ที่มี Liveness และ Readiness Probe ที่ใช้งานได้จริง
2. ทดลอง Cordon/Drain Node และดูว่า Pods ย้ายไปไหน
3. ดู iptables Rules ก่อนและหลังสร้าง Service
4. Monitor Resource Usage ด้วย `kubectl top`

## คำถามทบทวน

1. kubelet กับ kube-proxy ต่างกันอย่างไร?
2. Liveness Probe กับ Readiness Probe ต่างกันอย่างไร?
3. QoS Classes ส่งผลต่อการ Evict อย่างไร?
4. ทำไม IPVS ดีกว่า iptables สำหรับ Large Scale?

---

*ต่อไป: [Part 09: Minikube](./part-09-minikube.md)*
