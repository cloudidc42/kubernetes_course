# Part 09: Minikube

## สารบัญ
- [Minikube คืออะไร](#minikube-คืออะไร)
- [ติดตั้ง Minikube บน Linux/Mac/Windows](#ติดตั้ง-minikube)
- [minikube commands](#minikube-commands)
- [Minikube Addons](#minikube-addons)
- [Workshop: ติดตั้งและรัน Hello World บน Minikube](#workshop)

---

## Minikube คืออะไร

**Minikube** คือ Tool ที่ช่วยให้รัน Kubernetes Cluster แบบ Single-node บน Local Machine ได้

```
Minikube สร้าง:
                                            
┌───────────────────────────────────────────────┐
│            Local Machine                       │
│                                               │
│  ┌──────────────────────────────────────────┐ │
│  │         Minikube Node (VM/Container)      │ │
│  │                                          │ │
│  │  ┌────────────────────────────────────┐  │ │
│  │  │       Kubernetes Control Plane      │  │ │
│  │  │  apiserver + etcd + scheduler +    │  │ │
│  │  │  controller-manager               │  │ │
│  │  └────────────────────────────────────┘  │ │
│  │                                          │ │
│  │  ┌────────────────────────────────────┐  │ │
│  │  │       Kubernetes Worker Node       │  │ │
│  │  │  kubelet + kube-proxy +            │  │ │
│  │  │  containerd                        │  │ │
│  │  └────────────────────────────────────┘  │ │
│  │                                          │ │
│  │  [Your Application Pods]                │ │
│  └──────────────────────────────────────────┘ │
└───────────────────────────────────────────────┘
```

### ทำไมใช้ Minikube?

```
Use Cases:
1. Learning Kubernetes
   - ไม่ต้องจ่าย Cloud Services
   - ทดลองได้ทุกอย่าง

2. Local Development
   - Test แบบ Kubernetes Environment
   - Debug ก่อน Deploy Production

3. CI/CD Testing
   - Run Integration Tests
   - ทดสอบ Kubernetes Manifests

ข้อดี:
✅ ง่ายในการติดตั้ง
✅ ฟรี
✅ รองรับ Driver หลายตัว (docker, vm, etc.)
✅ มี Addons มากมาย
✅ รองรับ Multi-node (ทดลองได้)
✅ รองรับ Custom Resources

ข้อเสีย:
❌ Single-node จริง (Control Plane + Worker ใน Node เดียว)
❌ ไม่เหมาะสำหรับ Production
❌ Resource จำกัดตาม Local Machine
```

### Minikube Drivers

```
Drivers ที่รองรับ:

Docker (แนะนำสำหรับทุก OS):
  - รัน Kubernetes ใน Docker Container
  - เร็ว, ประหยัด Resource
  - ไม่ต้องการ Virtualization

VirtualBox:
  - รัน ใน VM ผ่าน VirtualBox
  - Isolation ดีกว่า Docker
  - ใช้ได้บน Linux, Mac, Windows

Hyperkit (macOS):
  - Native VM บน macOS
  - เร็วกว่า VirtualBox บน Mac

Hyper-V (Windows):
  - Native VM บน Windows
  - ต้องการ Windows 10/11 Pro

KVM (Linux):
  - Native VM บน Linux
  - Performance ดีมาก
```

---

## ติดตั้ง Minikube

### Linux

```bash
# ขั้นตอนที่ 1: ติดตั้ง kubectl
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Verify (optional)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client

# ขั้นตอนที่ 2: ติดตั้ง Docker (ถ้ายังไม่มี)
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# เพิ่ม user ไปยัง docker group
sudo usermod -aG docker $USER
newgrp docker

# ขั้นตอนที่ 3: ติดตั้ง Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify
minikube version

# Start Minikube
minikube start --driver=docker
```

### macOS

```bash
# ขั้นตอนที่ 1: ติดตั้ง Homebrew (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ขั้นตอนที่ 2: ติดตั้ง kubectl
brew install kubectl

# ตรวจสอบ
kubectl version --client

# ขั้นตอนที่ 3: ติดตั้ง Docker Desktop
# Download จาก https://www.docker.com/products/docker-desktop/
# หรือ
brew install --cask docker

# หรือใช้ OrbStack (เร็วกว่า Docker Desktop บน Mac)
brew install --cask orbstack

# ขั้นตอนที่ 4: ติดตั้ง Minikube
brew install minikube

# ตรวจสอบ
minikube version

# Start Minikube
minikube start --driver=docker

# หรือใช้ Hyperkit (Native Mac VM)
minikube start --driver=hyperkit
```

### Windows

```powershell
# ขั้นตอนที่ 1: ติดตั้ง Chocolatey (Package Manager)
# เปิด PowerShell as Administrator
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ขั้นตอนที่ 2: ติดตั้ง kubectl
choco install kubernetes-cli

# ตรวจสอบ
kubectl version --client

# ขั้นตอนที่ 3: ติดตั้ง Docker Desktop
# Download จาก https://www.docker.com/products/docker-desktop/

# ขั้นตอนที่ 4: ติดตั้ง Minikube
choco install minikube

# หรือ Download โดยตรง
# https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe

# ตรวจสอบ
minikube version

# Start Minikube
minikube start --driver=docker

# หรือใช้ Hyper-V (Windows 10/11 Pro)
minikube start --driver=hyperv
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบทุกอย่าง
minikube start

# ตัวอย่าง Output ที่ควรเห็น:
# 😄  minikube v1.31.0 on Ubuntu 22.04
# ✨  Using the docker driver
# 📌  Using Docker driver with root privileges
# 👍  Starting control plane node minikube
# 🚜  Pulling base image ...
# 🔥  Creating docker container ...
# 🐳  Preparing Kubernetes v1.27.3 on Docker 24.0.4 ...
#     ▪ Generating certificates and keys ...
#     ▪ Booting up control plane ...
#     ▪ Configuring RBAC rules ...
# 🔗  Configuring bridge CNI ...
# 🔎  Verifying Kubernetes components...
#     ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
# 🌟  Enabled addons: storage-provisioner, default-storageclass
# 🏄  Done! kubectl is now configured to use "minikube" cluster

# ดูสถานะ
minikube status
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured

kubectl cluster-info
# Kubernetes control plane is running at https://127.0.0.1:PORT
# CoreDNS is running at ...

kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   2m    v1.27.3
```

---

## minikube commands

### Cluster Management

```bash
# เริ่ม Cluster
minikube start

# เริ่มพร้อมตั้งค่าพิเศษ
minikube start \
  --driver=docker \
  --cpus=4 \
  --memory=8192 \
  --disk-size=50g \
  --kubernetes-version=v1.27.3 \
  --container-runtime=containerd \
  --extra-config=kubelet.max-pods=200

# หยุด Cluster (เก็บ State)
minikube stop

# ลบ Cluster
minikube delete

# ลบทุก Clusters
minikube delete --all

# ดูสถานะ
minikube status

# ดู Cluster Info
kubectl cluster-info

# Pause (หยุดชั่วคราว ประหยัด CPU)
minikube pause

# Unpause
minikube unpause
```

### Multiple Clusters

```bash
# สร้าง Cluster ที่สอง
minikube start -p cluster2

# ดู Clusters ทั้งหมด
minikube profile list

# ตัวอย่าง Output:
# |----------|-----------|---------|---------|------|---------|---------|-------|--------|
# | Profile  | VM Driver | Runtime | IP      | Port | Version | Status  | Nodes | Active |
# |----------|-----------|---------|---------|------|---------|---------|-------|--------|
# | minikube | docker    | docker  | 1.2.3.4 | PORT | v1.27.3 | Running | 1     | *      |
# | cluster2 | docker    | docker  | 1.2.3.5 | PORT | v1.27.3 | Running | 1     |        |
# |----------|-----------|---------|---------|------|---------|---------|-------|--------|

# Switch Cluster
minikube profile cluster2

# หรือใช้ kubectl context
kubectl config use-context minikube
kubectl config use-context cluster2

# ดู Current Context
kubectl config current-context

# ดู Contexts ทั้งหมด
kubectl config get-contexts
```

### Service Access

```bash
# เข้าถึง Service ผ่าน URL
minikube service my-service --url

# เปิด Browser อัตโนมัติ
minikube service my-service

# Tunnel สำหรับ LoadBalancer Services
minikube tunnel

# ตัวอย่าง:
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=LoadBalancer --port=80

# เปิด Tunnel (ใน Terminal แยก)
minikube tunnel

# ดู External IP
kubectl get service nginx
# NAME    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)
# nginx   LoadBalancer   10.96.0.100   127.0.0.1     80:PORT/TCP

# เข้าถึงได้ที่ http://127.0.0.1
```

### SSH และ Docker Environment

```bash
# SSH เข้า Minikube Node
minikube ssh

# Run Command โดยตรง
minikube ssh -- docker ps
minikube ssh -- ls /

# ใช้ Docker ของ Minikube (Build Image โดยตรง)
eval $(minikube docker-env)
# ตอนนี้ docker commands จะใช้ Docker ของ Minikube

docker build -t my-local-app:v1 .

# Deploy โดยไม่ต้อง Push ไปยัง Registry
kubectl run my-app --image=my-local-app:v1 --image-pull-policy=Never

# Reset docker env
eval $(minikube docker-env --unset)
```

### IP และ Network

```bash
# ดู IP ของ Minikube
minikube ip
# 192.168.49.2

# ดู Internal Port ของ API Server
minikube kubectl -- get pods

# Dashboard URL
minikube dashboard --url

# ดู Logs
minikube logs

# ดู Logs แบบ Follow
minikube logs --follow

# ดู Component Logs
minikube logs -n kube-system
```

### Configuration

```bash
# ดู Configuration
minikube config view

# ตั้ง Default Config
minikube config set driver docker
minikube config set cpus 4
minikube config set memory 8192
minikube config set disk-size 50g

# ลบ Config
minikube config unset cpus

# ดู Config ที่ตั้งไว้
cat ~/.minikube/config/config.json
```

---

## Minikube Addons

### Addons คืออะไร

Minikube Addons คือ Extensions ที่เพิ่ม Functionality ให้ Cluster

```bash
# ดู Addons ทั้งหมด
minikube addons list

# ตัวอย่าง Output:
# |-----------------------------|----------|--------------|
# |         ADDON NAME          | PROFILE  |    STATUS    |
# |-----------------------------|----------|--------------|
# | dashboard                   | minikube | enabled ✅   |
# | default-storageclass        | minikube | enabled ✅   |
# | dns-cache                   | minikube | disabled     |
# | efk                         | minikube | disabled     |
# | freshpod                    | minikube | disabled     |
# | gcp-auth                    | minikube | disabled     |
# | helm-tiller                 | minikube | disabled     |
# | ingress                     | minikube | disabled     |
# | ingress-dns                 | minikube | disabled     |
# | istio                       | minikube | disabled     |
# | logviewer                   | minikube | disabled     |
# | metrics-server              | minikube | disabled     |
# | metallb                     | minikube | disabled     |
# | monitoring                  | minikube | disabled     |
# | registry                    | minikube | disabled     |
# |-----------------------------|----------|--------------|
```

### Addons สำคัญ

#### 1. Dashboard

```bash
# Enable Dashboard
minikube addons enable dashboard

# เปิด Dashboard
minikube dashboard

# ดู URL เท่านั้น
minikube dashboard --url
```

#### 2. Metrics Server

```bash
# Enable Metrics Server
minikube addons enable metrics-server

# รอสักครู่...
kubectl top nodes
kubectl top pods --all-namespaces
```

#### 3. Ingress

```bash
# Enable Ingress (nginx)
minikube addons enable ingress

# ตรวจสอบ
kubectl get pods -n ingress-nginx

# ทดสอบ Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hello-world.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
EOF

# เพิ่ม Host ใน /etc/hosts
echo "$(minikube ip) hello-world.local" | sudo tee -a /etc/hosts

# ทดสอบ
curl hello-world.local
```

#### 4. Registry (Private Registry)

```bash
# Enable Registry
minikube addons enable registry

# Push Image ไปยัง Minikube Registry
# Port 5000 ของ Minikube
REGISTRY_PORT=$(kubectl get svc -n kube-system registry -o jsonpath='{.spec.ports[0].nodePort}')
REGISTRY_IP=$(minikube ip)

# Build และ Push
docker build -t localhost:$REGISTRY_PORT/myapp:v1 .
docker push localhost:$REGISTRY_PORT/myapp:v1

# Deploy จาก Registry
kubectl run myapp --image=localhost:$REGISTRY_PORT/myapp:v1
```

#### 5. Metallb (LoadBalancer สำหรับ Local)

```bash
# Enable Metallb
minikube addons enable metallb

# Configure IP Range
minikube addons configure metallb
# Enter Load Balancer Start IP: 192.168.49.100
# Enter Load Balancer End IP: 192.168.49.200

# ทดสอบ
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=LoadBalancer --port=80

kubectl get svc nginx
# NAME    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# nginx   LoadBalancer   10.96.0.100   192.168.49.100   80:PORT/TCP
```

#### 6. Storage Provisioner

```bash
# Default Storage Provisioner (มักเปิดอยู่แล้ว)
kubectl get storageclass
# NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   k8s.io/minikube-hostpath   Delete          Immediate

# ทดสอบ PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc test-pvc
# NAME       STATUS   VOLUME     CAPACITY   STORAGECLASS
# test-pvc   Bound    pvc-xxx    1Gi        standard
```

---

## Workshop: ติดตั้งและรัน Hello World บน Minikube

### เป้าหมาย

ติดตั้ง Minikube และ Deploy แอพลิเคชัน Hello World แบบสมบูรณ์พร้อม:
- Service
- Ingress
- Health Checks
- Resource Limits

### ขั้นตอน 1: ติดตั้งและเริ่ม Cluster

```bash
# ติดตั้ง (ถ้ายังไม่มี)
# ดูคำสั่งในส่วน "ติดตั้ง Minikube"

# เริ่ม Cluster
minikube start \
  --driver=docker \
  --cpus=2 \
  --memory=4096

# ตรวจสอบ
minikube status
kubectl cluster-info
kubectl get nodes
```

### ขั้นตอน 2: Enable Addons ที่ต้องการ

```bash
# Enable Dashboard
minikube addons enable dashboard

# Enable Metrics Server
minikube addons enable metrics-server

# Enable Ingress
minikube addons enable ingress

# ตรวจสอบ Addons
minikube addons list | grep enabled

# รอ Addons พร้อม
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=ingress-nginx \
  -n ingress-nginx \
  --timeout=120s
```

### ขั้นตอน 3: สร้าง Hello World Application

สร้างไฟล์ `hello-world-app.yaml`:

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: hello-world

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
  namespace: hello-world
data:
  APP_NAME: "Hello World from Kubernetes!"
  APP_VERSION: "1.0.0"
  ENVIRONMENT: "minikube"

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello-world
  labels:
    app: hello-world
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: hello-world
        version: "1.0"
    spec:
      containers:
      - name: hello-world
        image: nginxdemos/hello:latest
        ports:
        - containerPort: 80
          name: http
        
        # Environment from ConfigMap
        envFrom:
        - configMapRef:
            name: hello-config
        
        # Resource Limits
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: hello-world-svc
  namespace: hello-world
  labels:
    app: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: hello-world
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: hello-world.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world-svc
            port:
              number: 80

---
# HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello-world-hpa
  namespace: hello-world
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-world
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# Apply ทั้งหมด
kubectl apply -f hello-world-app.yaml

# ดู Status
kubectl get all -n hello-world
```

### ขั้นตอน 4: ตรวจสอบ Deployment

```bash
# ดู Pods
kubectl get pods -n hello-world
# NAME                           READY   STATUS    RESTARTS   AGE
# hello-world-xxx-yyy            1/1     Running   0          30s
# hello-world-xxx-zzz            1/1     Running   0          30s
# hello-world-xxx-aaa            1/1     Running   0          30s

# ดูรายละเอียด Deployment
kubectl describe deployment hello-world -n hello-world

# ดู Events
kubectl get events -n hello-world --sort-by='.lastTimestamp'

# ดู Logs
kubectl logs -l app=hello-world -n hello-world

# ดู Resource Usage
kubectl top pods -n hello-world
```

### ขั้นตอน 5: เข้าถึง Application

```bash
# วิธีที่ 1: kubectl port-forward
kubectl port-forward -n hello-world \
  service/hello-world-svc 8080:80 &

curl http://localhost:8080
# ✅ เห็น Nginx Demo Page

# วิธีที่ 2: minikube service
minikube service hello-world-svc -n hello-world --url

# วิธีที่ 3: ผ่าน Ingress
# เพิ่ม /etc/hosts
echo "$(minikube ip) hello-world.local" | sudo tee -a /etc/hosts

curl http://hello-world.local
# ✅ เห็นผ่าน Ingress

# Kill port-forward ถ้ายังรันอยู่
kill %1 2>/dev/null
```

### ขั้นตอน 6: ทดสอบ Rolling Update

```bash
# ดู Image ปัจจุบัน
kubectl get deployment hello-world -n hello-world \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Update Image (เป็น Version ใหม่)
kubectl set image deployment/hello-world \
  hello-world=nginx:1.25-alpine \
  -n hello-world

# ดูกระบวนการ Update
kubectl rollout status deployment/hello-world -n hello-world

# ดู ReplicaSets (ทั้งเก่าและใหม่)
kubectl get replicasets -n hello-world

# ถ้ามีปัญหา Rollback
kubectl rollout undo deployment/hello-world -n hello-world

# ดู History
kubectl rollout history deployment/hello-world -n hello-world
```

### ขั้นตอน 7: ทดสอบ Self-healing

```bash
# ดู Pods ปัจจุบัน
kubectl get pods -n hello-world

# ลบ Pod หนึ่งตัว
kubectl delete pod \
  $(kubectl get pods -n hello-world -o name | head -1) \
  -n hello-world

# ดู Self-healing
kubectl get pods -n hello-world -w

# Pod ใหม่จะถูกสร้างภายในไม่กี่วินาที!
```

### ขั้นตอน 8: ทดสอบ Auto-scaling

```bash
# ดู HPA
kubectl get hpa -n hello-world

# สร้าง Load เพื่อ Trigger Scaling
kubectl run -n hello-world load-test \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://hello-world-svc; done"

# ดู HPA ทำงาน (รอสักครู่)
kubectl get hpa -n hello-world -w

# ใน Terminal อื่น
kubectl top pods -n hello-world

# หยุด Load Test
kubectl delete pod load-test -n hello-world
```

### ขั้นตอน 9: ดู Dashboard

```bash
# เปิด Dashboard
minikube dashboard

# สำรวจ:
# 1. Workloads → Deployments → hello-world
#    - ดู Pods, ReplicaSets
# 2. Workloads → Pods
#    - ดู Logs แต่ละ Pod
#    - ดู Resource Usage
# 3. Service Discovery → Ingresses
#    - ดู hello-world-ingress
# 4. Config and Storage → ConfigMaps
#    - ดู hello-config
# 5. Cluster → Nodes
#    - ดู Resource Usage ของ Node
```

### ขั้นตอน 10: Cleanup

```bash
# ลบ Application
kubectl delete namespace hello-world

# หรือลบแค่บาง Resources
kubectl delete -f hello-world-app.yaml

# หยุด Minikube
minikube stop

# ลบ Cluster (ถ้าต้องการ)
minikube delete
```

---

## Advanced Minikube Features

### Multi-node Cluster

```bash
# สร้าง Multi-node Cluster
minikube start \
  --nodes=3 \
  --driver=docker \
  --cpus=2 \
  --memory=2048

# ดู Nodes
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# minikube       Ready    control-plane   2m    v1.27.3
# minikube-m02   Ready    <none>          90s   v1.27.3
# minikube-m03   Ready    <none>          60s   v1.27.3

# ทดสอบ Pod กระจายไปหลาย Nodes
kubectl create deployment nginx --image=nginx --replicas=6
kubectl get pods -o wide

# ดู Distribution
kubectl get pods -o wide | awk '{print $7}' | sort | uniq -c
```

### Custom Addons

```bash
# Enable addon จาก local
minikube addons enable --profile minikube \
  --addons-registry https://raw.githubusercontent.com/kubernetes/minikube/master/deploy/addons \
  my-custom-addon
```

### Mount Local Directory

```bash
# Mount directory จาก Host ไปยัง Minikube
minikube mount /host/path:/minikube/path

# หลังจาก Mount:
# สามารถ Reference ใน Pod ได้
# volumeMounts:
# - name: host-files
#   mountPath: /app/data
# volumes:
# - name: host-files
#   hostPath:
#     path: /minikube/path
```

### Minikube Tunnel

```bash
# Tunnel สำหรับ LoadBalancer Services
# ต้องรัน เป็น Root/Sudo
minikube tunnel

# ในอีก Terminal
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=LoadBalancer --port=80

kubectl get svc nginx
# NAME    TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)
# nginx   LoadBalancer   10.96.xxx    127.0.0.1     80:PORT/TCP

curl http://127.0.0.1
```

---

## สรุป

```
Minikube Summary:

ติดตั้ง:
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  sudo install minikube-linux-amd64 /usr/local/bin/minikube

คำสั่งหลัก:
  minikube start     - เริ่ม Cluster
  minikube stop      - หยุด Cluster
  minikube status    - ดูสถานะ
  minikube dashboard - เปิด Dashboard
  minikube delete    - ลบ Cluster

Addons ที่มีประโยชน์:
  dashboard          - Web UI
  metrics-server     - Resource Monitoring
  ingress            - HTTP Routing
  registry           - Private Image Registry
  metallb            - LoadBalancer

เหมาะสำหรับ:
  ✅ Learning
  ✅ Local Development
  ✅ Testing
  ❌ Production
```

---

## แบบฝึกหัด

1. ติดตั้ง Minikube และทดลอง Deploy WordPress + MySQL
2. ทดสอบ Multi-node Cluster และดู Pod Distribution
3. ใช้ Ingress Route Traffic ไปยัง 2 Services ที่ต่างกัน
4. ทดสอบ Auto-scaling ด้วย HPA

## คำถามทบทวน

1. Minikube Driver คืออะไร ใช้อันไหนดีที่สุด?
2. ทำไมต้องใช้ `eval $(minikube docker-env)` ก่อน Build Image?
3. Minikube Addons คืออะไร Addon ไหนสำคัญที่สุด?
4. Multi-node Minikube ทำงานอย่างไร?

---

*ต่อไป: [Part 10: kind (Kubernetes in Docker)](./part-10-kind.md)*
