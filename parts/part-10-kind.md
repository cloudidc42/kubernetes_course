# Part 10: kind (Kubernetes in Docker)

## สารบัญ
- [kind คืออะไร](#kind-คืออะไร)
- [ติดตั้ง kind](#ติดตั้ง-kind)
- [สร้าง Cluster ด้วย kind](#สร้าง-cluster-ด้วย-kind)
- [สร้าง Multi-node Cluster](#สร้าง-multi-node-cluster)
- [kind vs Minikube](#kind-vs-minikube)
- [Workshop: สร้าง Multi-node Cluster ด้วย kind](#workshop)

---

## kind คืออะไร

**kind** (**K**ubernetes **In** **D**ocker) คือ Tool ที่รัน Kubernetes Cluster โดยใช้ Docker Containers เป็น Nodes

```
kind Architecture:

Host Machine
├── Docker Engine
│   ├── kind-control-plane (Container = K8s Control Plane)
│   ├── kind-worker-1      (Container = K8s Worker Node)
│   ├── kind-worker-2      (Container = K8s Worker Node)
│   └── kind-worker-3      (Container = K8s Worker Node)
```

### ทำไม kind ถึงเร็ว?

```
Traditional Local K8s:
  Host → VM/VirtualBox → Kubernetes
  (ช้า เพราะต้อง Boot VM)

kind:
  Host → Docker Container → Kubernetes
  (เร็ว เพราะ Container Start ได้ทันที)
```

### ประวัติและ Use Case

kind ถูกสร้างโดย Kubernetes SIG Testing ใช้สำหรับ:
- **Kubernetes CI/CD Testing** - ทดสอบ Kubernetes เอง
- **Local Development** - พัฒนา Applications
- **Testing K8s Features** - ทดสอบ Features ใหม่
- **Multi-node Testing** - ทดสอบ HA, Node Failure

```
kind เหมาะสำหรับ:
✅ CI/CD Pipelines (GitHub Actions, GitLab CI)
✅ ทดสอบ Multi-node behavior
✅ ทดสอบ HA Scenarios
✅ เร็วกว่า Minikube สำหรับ Automated Tests
✅ Resource น้อยกว่า VM-based solutions

kind ไม่เหมาะสำหรับ:
❌ Production Workloads
❌ ต้องการ LoadBalancer จริง (ต้องใช้ MetalLB เพิ่ม)
❌ ต้องการ Persistent Volumes แบบ Advanced
```

---

## ติดตั้ง kind

### Linux

```bash
# Method 1: Download Binary โดยตรง
[ $(uname -m) = x86_64 ] && curl -Lo ./kind \
  https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Method 2: ผ่าน Go (ถ้ามี Go ติดตั้งอยู่)
go install sigs.k8s.io/kind@v0.20.0

# ตรวจสอบ
kind version
# kind v0.20.0 go1.21.0 linux/amd64
```

### macOS

```bash
# ผ่าน Homebrew (แนะนำ)
brew install kind

# หรือ Download Binary
[ $(uname -m) = x86_64 ] && curl -Lo ./kind \
  https://kind.sigs.k8s.io/dl/v0.20.0/kind-darwin-amd64 || \
  curl -Lo ./kind \
  https://kind.sigs.k8s.io/dl/v0.20.0/kind-darwin-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# ตรวจสอบ
kind version
```

### Windows

```powershell
# ผ่าน Chocolatey
choco install kind

# หรือผ่าน Winget
winget install Kubernetes.kind

# หรือ Download Binary
curl.exe -Lo kind-windows-amd64.exe `
  https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64

Move-Item .\kind-windows-amd64.exe c:\windows\kind.exe

# ตรวจสอบ
kind version
```

### ข้อกำหนดเบื้องต้น

```bash
# ต้องมี Docker
docker version

# ต้องมี kubectl
kubectl version --client

# ตรวจสอบ Docker Memory
docker info | grep "Total Memory"
# ควรมีอย่างน้อย 4GB สำหรับ Multi-node Cluster
```

---

## สร้าง Cluster ด้วย kind

### Simple Cluster

```bash
# สร้าง Cluster แบบง่าย (1 node)
kind create cluster

# ตัวอย่าง Output:
# Creating cluster "kind" ...
#  ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
#  ✓ Preparing nodes 📦
#  ✓ Writing configuration 📜
#  ✓ Starting control-plane 🕹️
#  ✓ Installing CNI 🔌
#  ✓ Installing StorageClass 💾
# Set kubectl context to "kind-kind"
# You can now use your cluster with:
# kubectl cluster-info --context kind-kind

# ตรวจสอบ
kubectl cluster-info --context kind-kind
kubectl get nodes
```

### Cluster พร้อม Config

```bash
# สร้าง Config File
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster

# Cluster-level settings
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  disableDefaultCNI: false

nodes:
- role: control-plane
  image: kindest/node:v1.27.3
  # Extra Port Mappings (สำหรับ Ingress)
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  # Kubeadm Config
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
EOF

# สร้าง Cluster จาก Config
kind create cluster --config kind-config.yaml

# ตรวจสอบ
kubectl get nodes
```

### Named Cluster

```bash
# สร้าง Cluster พร้อมชื่อ
kind create cluster --name dev-cluster

# ดู Clusters ทั้งหมด
kind get clusters
# kind
# dev-cluster

# Switch Context
kubectl config use-context kind-dev-cluster

# ดู Docker Containers ที่ kind สร้าง
docker ps | grep kind
```

### ลบ Cluster

```bash
# ลบ Cluster ชื่อ default (kind)
kind delete cluster

# ลบ Cluster ชื่อ dev-cluster
kind delete cluster --name dev-cluster

# ลบทุก kind Clusters
kind delete clusters --all
```

---

## สร้าง Multi-node Cluster

### 3-node Cluster (1 Control Plane + 2 Workers)

```bash
cat <<EOF > multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: multinode

nodes:
- role: control-plane
  image: kindest/node:v1.27.3

- role: worker
  image: kindest/node:v1.27.3

- role: worker
  image: kindest/node:v1.27.3
EOF

kind create cluster --config multi-node.yaml

# ดู Nodes
kubectl get nodes
# NAME                      STATUS   ROLES           AGE   VERSION
# multinode-control-plane   Ready    control-plane   90s   v1.27.3
# multinode-worker          Ready    <none>          60s   v1.27.3
# multinode-worker2         Ready    <none>          60s   v1.27.3
```

### HA Control Plane (3 Control Planes + 3 Workers)

```yaml
# ha-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ha-cluster

nodes:
# 3 Control Plane nodes (HA)
- role: control-plane
  image: kindest/node:v1.27.3
- role: control-plane
  image: kindest/node:v1.27.3
- role: control-plane
  image: kindest/node:v1.27.3

# 3 Worker nodes
- role: worker
  image: kindest/node:v1.27.3
- role: worker
  image: kindest/node:v1.27.3
- role: worker
  image: kindest/node:v1.27.3
```

```bash
kind create cluster --config ha-cluster.yaml

kubectl get nodes
# NAME                     STATUS   ROLES           AGE   VERSION
# ha-cluster-control-plane   Ready    control-plane   2m    v1.27.3
# ha-cluster-control-plane2  Ready    control-plane   90s   v1.27.3
# ha-cluster-control-plane3  Ready    control-plane   60s   v1.27.3
# ha-cluster-worker          Ready    <none>          30s   v1.27.3
# ha-cluster-worker2         Ready    <none>          30s   v1.27.3
# ha-cluster-worker3         Ready    <none>          30s   v1.27.3
```

### Advanced Multi-node Config

```yaml
# advanced-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: advanced

# ตั้งค่า Feature Gates
featureGates:
  "GenericEphemeralVolume": true

# ตั้งค่า Runtime Config
runtimeConfig:
  "api/all": "true"

# Networking
networking:
  disableDefaultCNI: true  # ปิด Default CNI (จะใช้ Calico แทน)
  podSubnet: "192.168.0.0/16"  # สำหรับ Calico

nodes:
- role: control-plane
  image: kindest/node:v1.27.3
  # Label สำหรับ Ingress
  labels:
    node-type: "control-plane"
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
  - containerPort: 443
    hostPort: 443

- role: worker
  image: kindest/node:v1.27.3
  labels:
    node-type: "worker"
    workload: "app"
  extraMounts:
  - hostPath: /tmp/worker1-data
    containerPath: /data

- role: worker
  image: kindest/node:v1.27.3
  labels:
    node-type: "worker"
    workload: "database"
  extraMounts:
  - hostPath: /tmp/worker2-data
    containerPath: /data
```

### Load Image ลง kind Cluster

```bash
# Build Image
docker build -t my-app:v1 .

# Load Image เข้า kind (ไม่ต้อง Push ไป Registry!)
kind load docker-image my-app:v1

# Load เข้า Cluster เฉพาะ
kind load docker-image my-app:v1 --name my-cluster

# ตรวจสอบว่า Image มีใน Nodes
docker exec kind-control-plane crictl images | grep my-app

# Deploy โดยใช้ Image
kubectl run my-app \
  --image=my-app:v1 \
  --image-pull-policy=Never  # ← สำคัญ! ไม่ Pull จาก Registry
```

---

## kind vs Minikube

### เปรียบเทียบละเอียด

| Feature | kind | Minikube |
|---------|------|---------|
| **Primary Purpose** | Testing/CI | Learning/Dev |
| **Speed (Start)** | เร็วมาก (~30s) | ช้ากว่า (~2min) |
| **Multi-node** | ✅ Native | ✅ (v1.10+) |
| **HA Control Plane** | ✅ | ❌ |
| **LoadBalancer** | ❌ (ต้องใช้ MetalLB) | ✅ (minikube tunnel) |
| **Ingress** | ต้องตั้งเอง | ✅ addon |
| **Dashboard** | ❌ (ติดตั้งเอง) | ✅ addon |
| **Addons** | ❌ | ✅ หลายอย่าง |
| **Driver** | Docker only | หลายตัว (docker, VM) |
| **CI/CD** | ✅ เหมาะมาก | ❌ ช้า |
| **Resource Usage** | ต่ำ | ปานกลาง |
| **Load Image** | `kind load` | `docker-env` |
| **Community** | Kubernetes SIG | CNCF |

### เมื่อไหรควรใช้อันไหน

```
ใช้ kind เมื่อ:
✅ CI/CD Pipeline (GitHub Actions, Jenkins)
✅ ต้องการ Multi-node จริงๆ
✅ ต้องการ HA Control Plane
✅ Test Kubernetes features อย่างเร็ว
✅ Resource จำกัด
✅ ต้องการ Reproducible Environment

ใช้ Minikube เมื่อ:
✅ เรียน Kubernetes ครั้งแรก
✅ ต้องการ Dashboard พร้อมใช้
✅ ต้องการ Addons หลายอย่าง
✅ ต้องการ LoadBalancer ง่ายๆ
✅ ต้องการ Driver หลายแบบ (VM, Docker)
✅ MacOS/Windows แบบ GUI
```

### Performance Comparison

```
Benchmark: สร้าง Cluster + Deploy 10 Pods

kind:
  Cluster Create: ~30 seconds
  Pod Ready: ~15 seconds
  Total: ~45 seconds

Minikube (docker driver):
  Cluster Create: ~120 seconds
  Pod Ready: ~20 seconds
  Total: ~140 seconds

Minikube (VM driver):
  Cluster Create: ~300 seconds
  Pod Ready: ~25 seconds
  Total: ~325 seconds

Winner: kind (3x faster than Minikube docker, 7x faster than VM)
```

---

## Workshop: สร้าง Multi-node Cluster ด้วย kind

### เป้าหมาย

สร้าง Production-like Cluster ด้วย kind ที่ประกอบด้วย:
- 1 Control Plane Node
- 3 Worker Nodes
- Nginx Ingress Controller
- MetalLB Load Balancer
- Local Image Registry

### ขั้นตอน 1: สร้าง Cluster Configuration

```bash
# สร้าง Config
cat <<EOF > workshop-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: workshop

networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"

nodes:
# Control Plane
- role: control-plane
  image: kindest/node:v1.27.3
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP

# Worker Nodes
- role: worker
  image: kindest/node:v1.27.3
  labels:
    workload-type: "app"

- role: worker
  image: kindest/node:v1.27.3
  labels:
    workload-type: "app"

- role: worker
  image: kindest/node:v1.27.3
  labels:
    workload-type: "database"
EOF

# สร้าง Cluster
kind create cluster --config workshop-cluster.yaml

# ตรวจสอบ
kubectl get nodes
kubectl cluster-info
```

### ขั้นตอน 2: ติดตั้ง Nginx Ingress Controller

```bash
# ติดตั้ง Ingress Nginx สำหรับ kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# รอให้พร้อม
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# ตรวจสอบ
kubectl get pods -n ingress-nginx
```

### ขั้นตอน 3: ติดตั้ง MetalLB (LoadBalancer)

```bash
# ติดตั้ง MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# รอให้พร้อม
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=120s

# หา Docker Network Range
SUBNET=$(docker network inspect kind | \
  python3 -c "import sys,json; \
  data=json.load(sys.stdin); \
  print(data[0]['IPAM']['Config'][0]['Subnet'])")

echo "Kind Network Subnet: $SUBNET"

# กำหนด IP Range สำหรับ MetalLB
# (ใช้ส่วนท้ายของ Subnet)
# ตัวอย่าง: 172.18.0.0/16 → ใช้ 172.18.255.200-172.18.255.250

cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```

### ขั้นตอน 4: ติดตั้ง Local Registry

```bash
# สร้าง Local Registry Docker Container
REGISTRY_NAME="kind-registry"
REGISTRY_PORT="5001"

docker run -d \
  --restart=always \
  --name "$REGISTRY_NAME" \
  -p "127.0.0.1:${REGISTRY_PORT}:5000" \
  registry:2

# เชื่อมต่อ Registry กับ kind network
docker network connect "kind" "${REGISTRY_NAME}"

# Configure kind ให้รู้จัก Registry
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:${REGISTRY_PORT}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF

echo "Registry: localhost:${REGISTRY_PORT}"
```

### ขั้นตอน 5: Build และ Push App ไปยัง Local Registry

```bash
# สร้าง Simple App
mkdir -p /tmp/kind-demo
cat <<EOF > /tmp/kind-demo/Dockerfile
FROM node:18-alpine
WORKDIR /app
RUN echo 'const http = require("http"); \
const os = require("os"); \
const server = http.createServer((req, res) => { \
  const body = JSON.stringify({ \
    message: "Hello from kind Cluster!", \
    node: process.env.NODE_NAME || os.hostname(), \
    pod: process.env.POD_NAME || "unknown", \
    namespace: process.env.POD_NAMESPACE || "unknown" \
  }); \
  res.writeHead(200, {"Content-Type": "application/json"}); \
  res.end(body); \
}); \
server.listen(8080, () => console.log("Running on 8080"));' > server.js
EXPOSE 8080
CMD ["node", "server.js"]
EOF

# Build Image
docker build -t localhost:5001/demo-app:v1 /tmp/kind-demo

# Push ไปยัง Local Registry
docker push localhost:5001/demo-app:v1

# Load เข้า kind (alternative)
# kind load docker-image localhost:5001/demo-app:v1 --name workshop
```

### ขั้นตอน 6: Deploy Application

```bash
cat <<EOF | kubectl apply -f -
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo
spec:
  replicas: 6          # 6 replicas บน 3 worker nodes = 2 ต่อ node
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      # กระจาย Pods ไปทั่ว Nodes
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: demo-app
      
      containers:
      - name: app
        image: localhost:5001/demo-app:v1
        ports:
        - containerPort: 8080
        
        # ส่ง Pod/Node info เข้า Container
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 15
        
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: demo-app-svc
  namespace: demo
spec:
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-app-svc
            port:
              number: 80
EOF

# รอให้ Pods พร้อม
kubectl wait --for=condition=ready pod \
  -l app=demo-app -n demo --timeout=60s

# ดู Pods และ Node Distribution
kubectl get pods -n demo -o wide
```

### ขั้นตอน 7: ทดสอบ Load Balancing และ Node Distribution

```bash
# เพิ่ม /etc/hosts
echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts

# ทดสอบ Ingress (Loop ดู Pod และ Node ที่ต่างกัน)
for i in {1..12}; do
  RESPONSE=$(curl -s http://demo.local)
  NODE=$(echo $RESPONSE | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('node','?'))")
  POD=$(echo $RESPONSE | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('pod','?')[:20])")
  echo "Node: $NODE | Pod: $POD"
done

# ตัวอย่าง Output (Pods กระจายทั่ว Nodes):
# Node: workshop-worker  | Pod: demo-app-xxx-aaa
# Node: workshop-worker2 | Pod: demo-app-xxx-bbb
# Node: workshop-worker3 | Pod: demo-app-xxx-ccc
# Node: workshop-worker  | Pod: demo-app-xxx-ddd
# ...
```

### ขั้นตอน 8: ทดสอบ Node Failure

```bash
# ดู Pods ปัจจุบัน และ Node ที่อยู่
kubectl get pods -n demo -o wide

# Simulate Node Failure (หยุด Worker Node)
docker stop workshop-worker

# ดูว่า Kubernetes Handle Node Failure
kubectl get nodes
kubectl get pods -n demo -o wide

# รอประมาณ 1-2 นาที แล้วดู Pods ย้ายไปยัง Nodes อื่น
kubectl get pods -n demo -o wide -w

# Start Node กลับ
docker start workshop-worker

# รอ Node Ready
kubectl wait --for=condition=ready node/workshop-worker --timeout=120s

# ดู Pods กระจายกลับ
kubectl get pods -n demo -o wide
```

### ขั้นตอน 9: ทดสอบ LoadBalancer ด้วย MetalLB

```bash
# สร้าง Service Type LoadBalancer
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: demo-app-lb
  namespace: demo
spec:
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
EOF

# รอ External IP
kubectl get service demo-app-lb -n demo -w

# ตัวอย่าง Output:
# NAME          TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)
# demo-app-lb   LoadBalancer   10.96.0.100   172.18.255.200    80:PORT/TCP

# ทดสอบผ่าน External IP
EXTERNAL_IP=$(kubectl get svc demo-app-lb -n demo \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP
```

### ขั้นตอน 10: ดู Multi-node ใน Action

```bash
# Deploy แบบ DaemonSet (1 Pod ต่อ Node)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  namespace: demo
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: monitor
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo \"Running on: $(hostname)\"; sleep 30; done"]
        resources:
          requests:
            cpu: "10m"
            memory: "16Mi"
EOF

# ดู DaemonSet - ทุก Node มี 1 Pod
kubectl get pods -n demo -l app=node-monitor -o wide

# ตัวอย่าง Output:
# NAME              READY   STATUS    NODE
# node-monitor-xxx  1/1     Running   workshop-control-plane
# node-monitor-yyy  1/1     Running   workshop-worker
# node-monitor-zzz  1/1     Running   workshop-worker2
# node-monitor-aaa  1/1     Running   workshop-worker3
```

### ขั้นตอน 11: ทดสอบ Node Affinity

```bash
# Label Worker Nodes
kubectl label nodes workshop-worker app-tier=frontend
kubectl label nodes workshop-worker2 app-tier=backend
kubectl label nodes workshop-worker3 app-tier=database

# Deploy Frontend เฉพาะ frontend node
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: app-tier
                operator: In
                values:
                - frontend
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
EOF

# ตรวจสอบว่า Pods อยู่บน frontend node เท่านั้น
kubectl get pods -n demo -l app=frontend -o wide
# ทุก Pods อยู่บน workshop-worker
```

### ขั้นตอน 12: CI/CD Simulation กับ kind

```bash
# Script จำลอง CI/CD Pipeline
cat <<'EOF' > /tmp/ci-pipeline.sh
#!/bin/bash
set -e

echo "=== CI/CD Pipeline Simulation ==="

# Step 1: สร้าง Test Cluster
echo "📦 Creating test cluster..."
kind create cluster --name ci-test

# Step 2: Build Test Image
echo "🔨 Building application..."
docker build -t test-app:latest /tmp/kind-demo/
kind load docker-image test-app:latest --name ci-test

# Step 3: Deploy
echo "🚀 Deploying to test cluster..."
kubectl create deployment test-app \
  --image=test-app:latest \
  --context kind-ci-test

kubectl expose deployment test-app \
  --type=ClusterIP \
  --port=80 \
  --target-port=8080 \
  --context kind-ci-test

# Step 4: Run Tests
echo "🧪 Running tests..."
kubectl wait --for=condition=ready pod \
  -l app=test-app \
  --timeout=60s \
  --context kind-ci-test

# Simple Test
kubectl run test-client \
  --image=busybox \
  --restart=Never \
  --context kind-ci-test \
  -- wget -qO- http://test-app

echo "✅ Tests passed!"

# Step 5: Cleanup
echo "🧹 Cleaning up..."
kind delete cluster --name ci-test

echo "=== Pipeline Complete ==="
EOF

chmod +x /tmp/ci-pipeline.sh
bash /tmp/ci-pipeline.sh
```

### ทำความสะอาด

```bash
# ลบ Resources
kubectl delete namespace demo

# ลบ Local Registry
docker stop kind-registry
docker rm kind-registry

# ลบ Cluster
kind delete cluster --name workshop

# ลบ /etc/hosts entry
sudo sed -i '/demo.local/d' /etc/hosts
sudo sed -i '/hello-world.local/d' /etc/hosts
```

---

## GitHub Actions กับ kind

```yaml
# .github/workflows/test.yml
name: Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  integration-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up kind
      uses: helm/kind-action@v1.8.0
      with:
        version: v0.20.0
        cluster_name: test-cluster
        config: ./kind-config.yaml
    
    - name: Build Docker Image
      run: |
        docker build -t myapp:${{ github.sha }} .
        kind load docker-image myapp:${{ github.sha }} \
          --name test-cluster
    
    - name: Deploy to kind
      run: |
        kubectl apply -f k8s/
        kubectl set image deployment/myapp \
          myapp=myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp \
          --timeout=5m
    
    - name: Run Tests
      run: |
        # Port forward สำหรับ Testing
        kubectl port-forward svc/myapp 8080:80 &
        sleep 5
        
        # Run integration tests
        npm run test:integration
    
    - name: Collect Logs on Failure
      if: failure()
      run: |
        kubectl get all --all-namespaces
        kubectl describe pods --all-namespaces
```

---

## สรุป

```
kind Summary:

ติดตั้ง:
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
  chmod +x ./kind && sudo mv kind /usr/local/bin/kind

คำสั่งหลัก:
  kind create cluster              - สร้าง Cluster
  kind create cluster --config     - สร้างจาก Config
  kind delete cluster              - ลบ Cluster
  kind get clusters                - ดู Clusters
  kind load docker-image           - Load Image เข้า Cluster

จุดเด่น:
  ✅ เร็วมาก (30 วินาที)
  ✅ Multi-node Native
  ✅ HA Control Plane
  ✅ เหมาะสำหรับ CI/CD
  ✅ Resource น้อย

จุดด้อย:
  ❌ ไม่มี Dashboard Built-in
  ❌ LoadBalancer ต้องใช้ MetalLB
  ❌ Docker only

kind vs Minikube:
  CI/CD → kind
  Learning → Minikube
  Quick Start → kind
  Full Features → Minikube
```

---

## แบบฝึกหัด

1. สร้าง HA Cluster (3 Control Plane + 3 Worker) ด้วย kind
2. ทดสอบ Node Failure ใน Multi-node Cluster
3. ใช้ kind ใน GitHub Actions Pipeline
4. ติดตั้ง Calico CNI แทน Default CNI

## คำถามทบทวน

1. kind แตกต่างจาก Minikube อย่างไร?
2. ทำไม kind ถึงเร็วกว่า Minikube สำหรับ CI/CD?
3. `kind load docker-image` ทำอะไร?
4. เมื่อไหรควรใช้ kind และเมื่อไหรควรใช้ Minikube?

---

## บทส่งท้าย Parts 01-10

ยินดีด้วย! คุณได้เรียนรู้ Kubernetes Foundation ครบแล้ว:

```
สิ่งที่เรียนรู้แล้ว:
✅ Part 01: Kubernetes คืออะไร ทำไมต้องใช้
✅ Part 02: Container Orchestration
✅ Part 03: Docker Fundamentals 1
✅ Part 04: Docker Fundamentals 2
✅ Part 05: Docker Compose
✅ Part 06: Kubernetes Architecture
✅ Part 07: Control Plane Components
✅ Part 08: Worker Node Components
✅ Part 09: Minikube
✅ Part 10: kind

ต่อไปในหลักสูตร:
📚 Part 11-20: Kubernetes Objects (Pods, Deployments, Services)
📚 Part 21-30: Kubernetes Networking
📚 Part 31-40: Storage และ Persistence
📚 Part 41-50: Security และ RBAC
📚 Part 51-60: Production Best Practices
```

---

*ต่อไป: [Part 11: Pods](./part-11-pods.md)*
