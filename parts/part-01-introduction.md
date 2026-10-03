# Part 01: แนะนำ Kubernetes

## สารบัญ
- [Kubernetes คืออะไร](#kubernetes-คืออะไร)
- [ประวัติความเป็นมา](#ประวัติความเป็นมา)
- [ทำไมต้องใช้ Kubernetes](#ทำไมต้องใช้-kubernetes)
- [Traditional Deployment vs Container vs Kubernetes](#traditional-deployment-vs-container-vs-kubernetes)
- [Use Cases และ Real-world Examples](#use-cases-และ-real-world-examples)
- [Architecture Overview](#architecture-overview)
- [Workshop: ดู Kubernetes Dashboard](#workshop-ดู-kubernetes-dashboard)

---

## Kubernetes คืออะไร

**Kubernetes** (มักเรียกย่อว่า **K8s**) คือระบบ Open-source สำหรับ Automating deployment, scaling, และ management ของ containerized applications

ชื่อ "Kubernetes" มาจากภาษากรีก หมายถึง "นายท้าย" หรือ "คนถือพวงมาลัยเรือ" สะท้อนถึงบทบาทของมันในการ "นำทาง" container applications

### ความสามารถหลักของ Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes ทำอะไรได้บ้าง                 │
├─────────────────────────────────────────────────────────────┤
│  🚀 Service Discovery & Load Balancing                      │
│     - ค้นหา Service โดยอัตโนมัติ                           │
│     - กระจาย Traffic ไปยัง Container หลายตัว               │
│                                                             │
│  💾 Storage Orchestration                                   │
│     - Mount Storage systems อัตโนมัติ                      │
│     - รองรับ Local storage, Cloud storage                   │
│                                                             │
│  🔄 Automated Rollouts & Rollbacks                         │
│     - Deploy ใหม่โดยไม่ Downtime                           │
│     - Rollback กลับเมื่อมีปัญหา                            │
│                                                             │
│  📦 Automatic Bin Packing                                   │
│     - จัด Container ลงบน Node อย่างเหมาะสม                │
│     - ประหยัด Resource                                      │
│                                                             │
│  🔧 Self-healing                                            │
│     - Restart Container ที่ล้มเหลว                         │
│     - แทนที่ Container ที่ไม่ตอบสนอง                      │
│                                                             │
│  🔐 Secret & Configuration Management                       │
│     - จัดการ Password, OAuth tokens, SSH keys              │
│     - ไม่ต้อง Rebuild Image                                │
└─────────────────────────────────────────────────────────────┘
```

---

## ประวัติความเป็นมา

### Timeline ของ Kubernetes

```
2003-2004: Google สร้าง Borg (Internal Container Management System)
    │
    ▼
2013: Google สร้าง Omega (Successor ของ Borg)
    │
    ▼
2014 (มิถุนายน): Google ประกาศ Kubernetes เป็น Open Source
    │            บน GitHub โดย Joe Beda, Brendan Burns, Craig McLuckie
    ▼
2015 (กรกฎาคม): Kubernetes 1.0 Released
    │            Google ร่วมกับ Linux Foundation ก่อตั้ง CNCF
    │            (Cloud Native Computing Foundation)
    ▼
2016: Kubernetes กลายเป็น Flagship Project ของ CNCF
    │   Helm Package Manager เปิดตัว
    ▼
2017: Docker Enterprise รองรับ Kubernetes
    │   Azure Kubernetes Service (AKS) เปิดตัว
    ▼
2018: Amazon EKS เปิดตัว
    │   Kubernetes 1.10 - RBAC เป็น Stable
    ▼
2019: CNCF ประกาศ Kubernetes เป็น Graduated Project
    │
    ▼
2020: Kubernetes 1.18 - Topology Manager เป็น Beta
    │
    ▼
2021: Kubernetes 1.20 - Docker runtime deprecated
    │
    ▼
2022: Kubernetes 1.24 - Docker shim removed
    │
    ▼
2023-ปัจจุบัน: Kubernetes ยังคงพัฒนาต่อเนื่อง
               มี Release ใหม่ทุก ~4 เดือน
```

### ทีมสร้าง Kubernetes

Kubernetes ถูกสร้างโดยทีมวิศวกรของ Google ที่มีประสบการณ์กับ **Borg** ซึ่งเป็น Internal Container Management System ที่ Google ใช้งานมานานกว่า 10 ปี

- **Joe Beda** - Software Engineer ที่ Google
- **Brendan Burns** - ปัจจุบันอยู่ที่ Microsoft
- **Craig McLuckie** - Co-founder ของ Heptio (ถูก VMware ซื้อ)

### Google Borg vs Kubernetes

| Feature | Google Borg | Kubernetes |
|---------|------------|------------|
| การใช้งาน | Internal (Google เท่านั้น) | Open Source |
| API | Proprietary | RESTful API |
| Language | C++ | Go |
| Community | Internal | Global Community |
| Scale | Millions of jobs | ขึ้นอยู่กับ Cluster |

---

## ทำไมต้องใช้ Kubernetes

### ปัญหาของการ Deploy แบบเดิม

**ลองนึกภาพสถานการณ์นี้:**

คุณมี E-commerce Application ที่ประกอบด้วย:
- Frontend (React)
- Backend API (Node.js)
- Database (MySQL)
- Cache (Redis)
- Message Queue (RabbitMQ)

**วิธีเดิม (Manual Deployment):**

```bash
# ปัญหา 1: ต้อง Deploy ทีละ Service บน Server จริง
ssh user@server1
cd /var/www/frontend
git pull
npm run build
pm2 restart frontend

ssh user@server2
cd /var/www/backend
git pull
npm install
pm2 restart backend

# ปัญหา 2: ถ้า Traffic เพิ่ม 10x ต้องทำอะไร?
# - ซื้อ Server ใหม่
# - Config Network
# - Deploy Application
# - Update Load Balancer
# ... ใช้เวลาหลายชั่วโมง/วัน
```

**ปัญหาที่พบบ่อย:**
1. **"It works on my machine"** - Environment ไม่ตรงกัน
2. **Downtime ระหว่าง Deploy** - User ไม่สามารถใช้งานได้
3. **Scale ยาก** - ต้องทำด้วยมือ
4. **Resource ใช้ไม่เต็มที่** - Server รัน 1 Service แต่ CPU/RAM เหลือเยอะ
5. **Recovery ช้า** - ถ้า Service ล้ม ต้องมีคนมา Restart

### Kubernetes แก้ปัญหาเหล่านี้อย่างไร?

```yaml
# แค่ apply ไฟล์นี้ Kubernetes จะจัดการให้ทุกอย่าง
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3          # ต้องการ 3 instances
  selector:
    matchLabels:
      app: my-app
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:v2.0   # Deploy เวอร์ชันใหม่
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

**สิ่งที่ Kubernetes ทำให้:**
- ✅ Deploy 3 instances พร้อมกัน
- ✅ ถ้า Instance ล้ม จะ Restart อัตโนมัติ
- ✅ Deploy Version ใหม่แบบ Rolling (ไม่มี Downtime)
- ✅ Scale ได้ด้วยคำสั่งเดียว

```bash
# Scale จาก 3 เป็น 10 instances ทันที
kubectl scale deployment my-app --replicas=10
```

### สถิติที่น่าสนใจ

- **90%+** ของ Fortune 500 ใช้ Kubernetes
- **Spotify** รัน Kubernetes บน 150+ Nodes
- **Airbnb** ใช้ Kubernetes สำหรับ 1000+ Services
- **Pinterest** ลด Infrastructure Cost ได้ **40%** หลังใช้ Kubernetes

---

## Traditional Deployment vs Container vs Kubernetes

### วิวัฒนาการของการ Deploy

```
Era 1: Traditional Deployment (2000s)
┌────────────────────────────────────┐
│           Physical Server          │
│  ┌──────────────────────────────┐  │
│  │        Operating System      │  │
│  ├──────────┬───────────────────┤  │
│  │  App A   │      App B        │  │
│  │(Python2) │    (Python3)      │  │
│  │          │                   │  │
│  │ CONFLICT!│ Resource Waste!   │  │
│  └──────────┴───────────────────┘  │
└────────────────────────────────────┘

Era 2: Virtual Machine Deployment (2010s)
┌────────────────────────────────────┐
│           Physical Server          │
│  ┌──────────────────────────────┐  │
│  │           Hypervisor         │  │
│  ├──────────────┬───────────────┤  │
│  │     VM 1     │     VM 2      │  │
│  │  ┌────────┐  │  ┌────────┐   │  │
│  │  │  OS    │  │  │  OS    │   │  │
│  │  ├────────┤  │  ├────────┤   │  │
│  │  │ App A  │  │  │ App B  │   │  │
│  │  └────────┘  │  └────────┘   │  │
│  │ Isolated!    │ But Heavy!    │  │
│  └──────────────┴───────────────┘  │
└────────────────────────────────────┘

Era 3: Container Deployment (2015+)
┌────────────────────────────────────┐
│           Physical Server          │
│  ┌──────────────────────────────┐  │
│  │        Operating System      │  │
│  ├──────────────────────────────┤  │
│  │       Container Runtime      │  │
│  ├──────────┬────────┬──────────┤  │
│  │Container │Container│Container│  │
│  │  App A   │ App B  │ App C   │  │
│  │ Isolated!│ Light! │ Fast!   │  │
│  └──────────┴────────┴──────────┘  │
└────────────────────────────────────┘

Era 4: Kubernetes Deployment (2017+)
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│  ┌───────────────┐   ┌───────────────┐   ┌──────────┐  │
│  │    Node 1     │   │    Node 2     │   │  Node 3  │  │
│  │  ┌─────────┐  │   │  ┌─────────┐  │   │          │  │
│  │  │App A x2 │  │   │  │App A x1 │  │   │ App B x2 │  │
│  │  └─────────┘  │   │  └─────────┘  │   │          │  │
│  │  ┌─────────┐  │   │  ┌─────────┐  │   └──────────┘  │
│  │  │ App B   │  │   │  │ App C   │  │                  │
│  │  └─────────┘  │   │  └─────────┘  │   Auto-scale!   │
│  └───────────────┘   └───────────────┘   Self-healing! │
└─────────────────────────────────────────────────────────┘
```

### เปรียบเทียบแบบละเอียด

| หัวข้อ | Traditional | VM | Container | Kubernetes |
|--------|------------|-----|-----------|------------|
| **Startup Time** | ชั่วโมง/วัน | นาที | วินาที | วินาที |
| **Resource Usage** | 40-60% | 60-80% | 70-90% | 80-95% |
| **Isolation** | None | Strong | Good | Good |
| **Portability** | ต่ำ | ปานกลาง | สูง | สูงมาก |
| **Scaling** | Manual | Semi-Auto | Manual | Auto |
| **Self-healing** | ❌ | ❌ | ❌ | ✅ |
| **Rolling Deploy** | ยาก | ปานกลาง | ยาก | อัตโนมัติ |
| **Cost** | สูง | ปานกลาง | ต่ำ | ต่ำมาก |

---

## Use Cases และ Real-world Examples

### 1. Microservices Architecture

Kubernetes เหมาะมากสำหรับ Microservices เพราะ:

```
┌──────────────────────────────────────────────────────────────┐
│                    E-commerce Platform                        │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  User    │  │ Product  │  │  Order   │  │ Payment  │   │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │   │
│  │  x3      │  │  x5      │  │  x3      │  │  x2      │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│       │              │              │              │         │
│  ┌────▼──────────────▼──────────────▼──────────────▼────┐  │
│  │                API Gateway (Ingress)                   │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**ตัวอย่าง: Netflix**
- มี Microservices มากกว่า 700 services
- ใช้ Kubernetes จัดการ Container หลายพันตัว
- Deploy ได้หลายพันครั้งต่อวัน

### 2. CI/CD Pipeline

```yaml
# GitHub Actions + Kubernetes
name: Deploy to K8s
on:
  push:
    branches: [main]
jobs:
  deploy:
    steps:
    - name: Build Docker Image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Push to Registry
      run: docker push myapp:${{ github.sha }}
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/myapp \
          myapp=myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp
```

### 3. Machine Learning Workloads

```yaml
# รัน ML Training Job
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training
spec:
  template:
    spec:
      containers:
      - name: trainer
        image: tensorflow:latest
        resources:
          limits:
            nvidia.com/gpu: 4  # ขอ GPU 4 ตัว
        command: ["python", "train.py"]
      restartPolicy: Never
```

### 4. Auto-scaling สำหรับ Flash Sales

**ลองนึกภาพ: 11.11 Sale ของ Lazada**

```
เวลาปกติ:
User Traffic: 1,000 req/sec
Pods Running: 5

เวลา 00:00:00 (Flash Sale เริ่ม):
User Traffic: 100,000 req/sec (เพิ่มขึ้น 100x!)

Kubernetes HPA (Horizontal Pod Autoscaler) ทำงาน:
- ตรวจจับว่า CPU Usage สูง
- Scale จาก 5 เป็น 500 pods ภายใน 2 นาที
- Traffic ถูก Handle ได้ทั้งหมด
- ไม่มี Downtime!
```

```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 5
  maxReplicas: 500
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 5. Multi-cloud Strategy

```
┌─────────────────────────────────────────────────────────┐
│                   Global Kubernetes Cluster             │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   AWS       │  │   GCP       │  │   Azure     │   │
│  │   Region    │  │   Region    │  │   Region    │   │
│  │  US-East    │  │  Asia       │  │  Europe     │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
│                                                         │
│  เดียวกัน Application, Config เดียวกัน               │
│  ไม่ vendor lock-in!                                    │
└─────────────────────────────────────────────────────────┘
```

### Real-world Success Stories

**Spotify:**
```
ก่อน Kubernetes:
- Deployment ใช้เวลา 2-4 ชั่วโมง
- 150+ ทีม Deploy แยกกัน
- Infrastructure ซับซ้อน

หลัง Kubernetes:
- Deployment ใช้เวลา < 10 นาที
- Standardized Deployment Process
- ประหยัด Infrastructure Cost 40%
```

**Pinterest:**
```
Challenge: 
- 250 billion pins
- 100 million monthly active users
- Need to scale dynamically

Solution with Kubernetes:
- Managed 1000+ microservices
- Reduced deployment time by 75%
- Improved resource efficiency significantly
```

---

## Architecture Overview

### Big Picture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                         │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                     Control Plane                         │    │
│  │                                                           │    │
│  │  ┌─────────────┐  ┌──────────┐  ┌────────────────────┐  │    │
│  │  │ API Server  │  │  etcd   │  │    Scheduler       │  │    │
│  │  │             │  │         │  │                    │  │    │
│  │  └─────────────┘  └──────────┘  └────────────────────┘  │    │
│  │                                                           │    │
│  │  ┌─────────────────────────────────────────────────────┐ │    │
│  │  │           Controller Manager                        │ │    │
│  │  └─────────────────────────────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────────────┘    │
│                              │                                     │
│                    API Calls │ (kubectl)                           │
│                              │                                     │
│  ┌───────────────┐  ┌────────┴──────┐  ┌───────────────────┐    │
│  │   Worker      │  │   Worker      │  │   Worker          │    │
│  │   Node 1      │  │   Node 2      │  │   Node 3          │    │
│  │               │  │               │  │                   │    │
│  │  ┌─────────┐  │  │  ┌─────────┐  │  │  ┌───────────┐  │    │
│  │  │ kubelet │  │  │  │ kubelet │  │  │  │  kubelet  │  │    │
│  │  ├─────────┤  │  │  ├─────────┤  │  │  ├───────────┤  │    │
│  │  │kube-    │  │  │  │kube-    │  │  │  │kube-proxy │  │    │
│  │  │proxy    │  │  │  │proxy    │  │  │  │           │  │    │
│  │  ├─────────┤  │  │  ├─────────┤  │  │  ├───────────┤  │    │
│  │  │Container│  │  │  │Container│  │  │  │Container  │  │    │
│  │  │Runtime  │  │  │  │Runtime  │  │  │  │Runtime    │  │    │
│  │  └─────────┘  │  │  └─────────┘  │  │  └───────────┘  │    │
│  │               │  │               │  │                   │    │
│  │  [Pod][Pod]   │  │  [Pod][Pod]   │  │    [Pod][Pod]    │    │
│  └───────────────┘  └───────────────┘  └───────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### Kubernetes Objects ที่สำคัญ

```
Kubernetes Resources Hierarchy:
│
├── Pod (หน่วยเล็กที่สุด)
│   └── Container(s)
│
├── ReplicaSet (ดูแล Pod หลายตัว)
│   └── Pod x N
│
├── Deployment (จัดการ Rolling Update)
│   └── ReplicaSet
│       └── Pod x N
│
├── Service (Network Access)
│   ├── ClusterIP
│   ├── NodePort
│   └── LoadBalancer
│
├── Ingress (HTTP Routing)
│
├── ConfigMap (Configuration)
│
├── Secret (Sensitive Data)
│
├── PersistentVolume (Storage)
│
└── Namespace (Isolation)
```

### Communication Flow

```
User → kubectl → API Server → etcd (Store)
                            → Scheduler (Schedule Pod)
                            → Controller Manager (Reconcile)
                            → kubelet (Run Container)
```

---

## Workshop: ดู Kubernetes Dashboard

ใน Workshop นี้เราจะ:
1. ติดตั้ง Minikube
2. เปิด Kubernetes Dashboard
3. ดูและทำความเข้าใจ Components ต่างๆ

### Prerequisites

```bash
# ตรวจสอบว่ามี kubectl
kubectl version --client

# ตรวจสอบว่ามี minikube
minikube version
```

### ขั้นตอนที่ 1: เริ่ม Minikube Cluster

```bash
# เริ่ม Minikube
minikube start --driver=docker

# ตัวอย่าง output ที่ควรเห็น:
# 😄  minikube v1.31.0 on Ubuntu 22.04
# ✨  Using the docker driver based on user configuration
# 📌  Using Docker driver with root privileges
# 👍  Starting control plane node minikube in cluster minikube
# 🚜  Pulling base image ...
# 🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
# 🐳  Preparing Kubernetes v1.27.3 on Docker 24.0.4 ...
# 🔗  Configuring bridge CNI (Container Networking Interface) ...
# 🔎  Verifying Kubernetes components...
# 🌟  Enabled addons: storage-provisioner, default-storageclass
# 🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

# ตรวจสอบสถานะ
minikube status
```

### ขั้นตอนที่ 2: ดู Cluster Information

```bash
# ดู Cluster Info
kubectl cluster-info

# ตัวอย่าง output:
# Kubernetes control plane is running at https://127.0.0.1:51234
# CoreDNS is running at https://127.0.0.1:51234/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# ดู Nodes
kubectl get nodes

# ตัวอย่าง output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.27.3

# ดูรายละเอียด Node
kubectl describe node minikube
```

### ขั้นตอนที่ 3: ดู Pods ที่ระบบสร้างมาให้

```bash
# ดู Pods ใน kube-system namespace
kubectl get pods -n kube-system

# ตัวอย่าง output:
# NAME                               READY   STATUS    RESTARTS   AGE
# coredns-5d78c9869d-xyzab           1/1     Running   0          5m
# etcd-minikube                      1/1     Running   0          5m
# kube-apiserver-minikube            1/1     Running   0          5m
# kube-controller-manager-minikube   1/1     Running   0          5m
# kube-proxy-xyzab                   1/1     Running   0          5m
# kube-scheduler-minikube            1/1     Running   0          5m
# storage-provisioner                1/1     Running   0          5m
```

### ขั้นตอนที่ 4: เปิด Kubernetes Dashboard

```bash
# Enable Dashboard Addon
minikube addons enable dashboard

# เปิด Dashboard
minikube dashboard

# หรือถ้าต้องการ URL เท่านั้น
minikube dashboard --url
```

### ขั้นตอนที่ 5: Deploy Application แรก

```bash
# Deploy Nginx
kubectl create deployment nginx --image=nginx

# ตรวจสอบ Deployment
kubectl get deployments

# ตัวอย่าง output:
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# nginx   1/1     1            1           30s

# ดู Pods
kubectl get pods

# ตัวอย่าง output:
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-77b4fdf86c-abc123  1/1     Running   0          30s

# Expose เป็น Service
kubectl expose deployment nginx --type=NodePort --port=80

# ดู URL
minikube service nginx --url
```

### ขั้นตอนที่ 6: ดู Dashboard

เปิด Browser ไปที่ URL ที่ได้จาก `minikube dashboard`

**สิ่งที่ควรดูใน Dashboard:**

1. **Workloads**
   - Deployments: เห็น nginx ที่เราสร้าง
   - Pods: เห็น Pod ที่กำลัง Running
   - Replica Sets: เห็น ReplicaSet ที่ Deployment สร้าง

2. **Service Discovery & Load Balancing**
   - Services: เห็น nginx service ที่เราสร้าง

3. **Cluster**
   - Nodes: เห็น Node ของ Minikube

### ขั้นตอนที่ 7: ทดลอง Scale

```bash
# Scale Nginx จาก 1 เป็น 3
kubectl scale deployment nginx --replicas=3

# ดู Pods (จะเห็น 3 pods)
kubectl get pods

# ตัวอย่าง output:
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-77b4fdf86c-abc123  1/1     Running   0          2m
# nginx-77b4fdf86c-def456  1/1     Running   0          10s
# nginx-77b4fdf86c-ghi789  1/1     Running   0          10s
```

**Refresh Dashboard** แล้วดูว่า Pods เพิ่มขึ้นเป็น 3 ตัว

### ขั้นตอนที่ 8: ทดลอง Self-healing

```bash
# ลบ Pod หนึ่งตัว
kubectl delete pod nginx-77b4fdf86c-abc123

# ดู Pods ทันที - จะเห็น Pod ใหม่ถูกสร้างขึ้นมาแทน
kubectl get pods -w

# ตัวอย่าง output:
# NAME                     READY   STATUS              RESTARTS   AGE
# nginx-77b4fdf86c-def456  1/1     Running             0          2m
# nginx-77b4fdf86c-ghi789  1/1     Running             0          2m
# nginx-77b4fdf86c-jkl012  0/1     ContainerCreating   0          2s
# nginx-77b4fdf86c-jkl012  1/1     Running             0          5s
```

Kubernetes สร้าง Pod ใหม่ทดแทนทันที!

### ขั้นตอนที่ 9: ทำความสะอาด

```bash
# ลบ Service และ Deployment
kubectl delete service nginx
kubectl delete deployment nginx

# หรือหยุด Minikube
minikube stop

# ลบ Minikube Cluster (ถ้าต้องการ)
minikube delete
```

### Workshop Summary

ในการทำ Workshop นี้ เราได้เห็น:

| สิ่งที่ทดลอง | ผลที่ได้ |
|------------|---------|
| Deploy Application | Kubernetes สร้าง Pod อัตโนมัติ |
| Scale | เพิ่ม/ลด Pod ได้ทันที |
| Self-healing | Pod ใหม่ถูกสร้างแทนที่ Pod ที่ถูกลบ |
| Dashboard | เห็น Cluster State แบบ Real-time |

---

## สรุป

Kubernetes คือ Platform ที่ช่วยให้การ Deploy และ Manage Containerized Applications เป็นเรื่องง่าย โดยเฉพาะในสภาพแวดล้อมที่ต้องการ:

- **High Availability** - ไม่มี Downtime
- **Scalability** - Scale ได้ตามความต้องการ
- **Self-healing** - Recover อัตโนมัติ
- **Resource Efficiency** - ใช้ Resource ได้เต็มประสิทธิภาพ

ในบทต่อไป เราจะเรียนรู้เรื่อง **Container Orchestration** ในเชิงลึกว่า Kubernetes จัดการ Container อย่างไร

---

## แบบฝึกหัด

1. ทดลอง Deploy Application อื่น เช่น `httpd` หรือ `mysql`
2. ทดลอง Scale ขึ้นและลง
3. ลอง Delete Pod แล้วดูว่า Self-healing ทำงานอย่างไร
4. สำรวจ Dashboard ทุก Section ให้ครบ

## คำถามทบทวน

1. Kubernetes แตกต่างจาก Docker Compose อย่างไร?
2. Self-healing ใน Kubernetes ทำงานอย่างไร?
3. Control Plane คืออะไรและมีหน้าที่อะไรบ้าง?
4. ทำไม Kubernetes ถึงเป็น Standard ของ Industry?

---

*ต่อไป: [Part 02: Container Orchestration](./part-02-container-orchestration.md)*

---

## Kubernetes vs Traditional VMs - เปรียบเทียบเชิงลึก

### ตาราง Feature Comparison

| คุณสมบัติ | Traditional VMs | Docker Containers | Kubernetes |
|-----------|----------------|-------------------|------------|
| Startup Time | 1-5 นาที | 1-30 วินาที | 5-60 วินาที (Pod) |
| Size | GB ต่อ VM | MB ต่อ Image | MB (Container) |
| Isolation | Full OS Isolation | Process Isolation | Namespace Isolation |
| Resource Usage | สูง (Full OS) | ต่ำ (Shared Kernel) | ต่ำมาก (Optimized) |
| Portability | ต่ำ (Image ใหญ่) | สูง (Lightweight) | สูงมาก (Platform Independent) |
| Scaling | ช้า (ต้อง Clone VM) | เร็ว (ไม่กี่วินาที) | อัตโนมัติ (HPA) |
| Self-healing | Manual | Manual | อัตโนมัติ |
| Load Balancing | ต้องตั้งค่าเอง | ต้องตั้งค่าเอง | Built-in |
| Service Discovery | ต้องตั้งค่าเอง | ต้องตั้งค่าเอง | Built-in (DNS) |
| Rolling Updates | ยุ่งยาก | ปานกลาง | อัตโนมัติ Zero-downtime |
| Rollback | ยากมาก | ปานกลาง | ง่าย (kubectl rollout undo) |
| Storage Management | Manual | Volumes | Persistent Volumes |
| Secret Management | Manual | Environment Vars | Secrets Object |
| Config Management | Manual | Environment Vars | ConfigMaps |
| Network Policy | OS Firewall | Docker Networks | NetworkPolicy |
| Multi-tenant | VMs แยก | Namespaces พื้นฐาน | Namespaces + RBAC |
| HA & Failover | ต้องตั้งค่า | ต้องตั้งค่า | Built-in (Node HA) |
| Cost | สูง | ปานกลาง | ต่ำสุด (Resource Efficiency) |
| Learning Curve | ต่ำ | ปานกลาง | สูง |

### เปรียบเทียบ Resource Overhead

```
Traditional VM Stack:
┌─────────────────────────────────────────────────────┐
│  Application (100MB)                                 │
├─────────────────────────────────────────────────────┤
│  Runtime (200MB)                                     │
├─────────────────────────────────────────────────────┤
│  Guest OS (2,000MB)  ← ใช้ Memory มาก!             │
├─────────────────────────────────────────────────────┤
│  Virtual Hardware                                    │
├─────────────────────────────────────────────────────┤
│  Hypervisor (500MB)                                  │
├─────────────────────────────────────────────────────┤
│  Host OS (1,000MB)                                   │
├─────────────────────────────────────────────────────┤
│  Physical Hardware                                   │
└─────────────────────────────────────────────────────┘
Total Overhead per App: ~3.8 GB

Container Stack (Kubernetes):
┌─────────────────────────────────────────────────────┐
│  Application (100MB)                                 │
├─────────────────────────────────────────────────────┤
│  Runtime (200MB)                                     │
├─────────────────────────────────────────────────────┤
│  Container Image Layers (100MB shared)               │
├─────────────────────────────────────────────────────┤
│  Docker/containerd (50MB)                            │
├─────────────────────────────────────────────────────┤
│  Host OS (1,000MB) ← ใช้ร่วมกันทุก Container       │
├─────────────────────────────────────────────────────┤
│  Physical Hardware                                   │
└─────────────────────────────────────────────────────┘
Total Overhead per App: ~450MB (ประหยัดถึง 8x!)
```

### เปรียบเทียบการ Scale

```
Scenario: ต้องการเพิ่มจาก 2 → 10 instances

Traditional VM Scaling (10-20 นาที):
[VM1] [VM2]
  ↓ clone
[VM1] [VM2] [VM3..VM10] ← ต้องรอ Clone + Boot OS

Container Scaling (30 วินาที):
[Pod1] [Pod2]
  ↓ kubectl scale --replicas=10
[Pod1] [Pod2] [Pod3] [Pod4] [Pod5] [Pod6] [Pod7] [Pod8] [Pod9] [Pod10]
  ← แค่ Pull Image แล้ว Start Process!
```

### เปรียบเทียบ CI/CD Pipeline

| ขั้นตอน | Traditional VM | Kubernetes |
|---------|----------------|------------|
| Build | Build Application | Build Docker Image |
| Test | Deploy to Test Server | Deploy to Test Namespace |
| Package | Zip/Tar Binary | Push to Registry |
| Deploy | SSH + Copy Files | kubectl apply |
| Verify | Manual Check | Readiness Probe |
| Rollback | Restore Backup (ชั่วโมง) | kubectl rollout undo (วินาที) |

---

## Real-world Case Studies

### Netflix บน Kubernetes

Netflix เป็นหนึ่งในองค์กรแรกๆ ที่นำ Kubernetes มาใช้ในระดับ Production ขนาดใหญ่

**ข้อมูล Infrastructure ของ Netflix:**
- จำนวน Microservices: มากกว่า 500 services
- จำนวน Requests ต่อวัน: มากกว่า 1 พันล้าน requests
- จำนวน Kubernetes Clusters: หลายร้อย clusters
- จำนวน Containers ที่รันพร้อมกัน: หลักล้าน containers

**ปัญหาที่ Netflix เผชิญก่อนใช้ Kubernetes:**

```
Before Kubernetes (2013-2016):
┌─────────────────────────────────────────────────────────────────┐
│  ปัญหา:                                                          │
│  1. Monolithic Application ยากต่อการ Scale                      │
│  2. Deployment ใช้เวลานาน (hours → days)                        │
│  3. ทีมต่างๆ ต้อง Coordinate การ Deploy                         │
│  4. Resource Utilization ต่ำมาก (~30%)                          │
│  5. เมื่อ Traffic พุ่งสูงในช่วง Peak (เย็นวันศุกร์)            │
│     ระบบมักล่ม หรือตอบสนองช้า                                   │
└─────────────────────────────────────────────────────────────────┘

After Kubernetes (2017-ปัจจุบัน):
┌─────────────────────────────────────────────────────────────────┐
│  ผลลัพธ์:                                                        │
│  1. Deployment ลดจาก hours เหลือ minutes                        │
│  2. Resource Utilization เพิ่มขึ้นเป็น 70%+                    │
│  3. Auto-scaling รับมือ Traffic Spike อัตโนมัติ                │
│  4. Self-healing ลด Manual Intervention 90%                      │
│  5. Engineer สามารถ Deploy ได้ตลอด 24 ชั่วโมง โดยไม่กลัว      │
└─────────────────────────────────────────────────────────────────┘
```

**Netflix Titus - Custom Kubernetes Platform:**

Netflix พัฒนา Titus ซึ่งเป็น Container Management System ที่สร้างบน Kubernetes โดยมีคุณสมบัติพิเศษ:

```yaml
# ตัวอย่าง Job Specification ของ Netflix Titus
{
  "applicationName": "video-transcoding-job",
  "resources": {
    "cpu": 4,
    "memoryMB": 4096,
    "diskMB": 10000,
    "networkMbps": 128,
    "gpu": 1  # ← Netflix ใช้ GPU สำหรับ Video Processing
  },
  "container": {
    "image": {
      "name": "netflix/transcoder",
      "tag": "v2.3.1"
    },
    "env": {
      "QUALITY": "4K",
      "OUTPUT_FORMAT": "DASH"
    }
  },
  "batch": {
    "size": 1,
    "retries": 3
  }
}
```

**สิ่งที่ Netflix เรียนรู้:**
1. **Chaos Engineering**: ใช้ Chaos Monkey ทดสอบความทนทานของ Kubernetes Cluster
2. **Multi-region**: Deploy บน AWS หลาย Region เพื่อ HA
3. **Canary Deployment**: ทดสอบ Feature ใหม่กับ User กลุ่มเล็กก่อน

---

### Airbnb บน Kubernetes

Airbnb ย้ายระบบทั้งหมดมาบน Kubernetes เพื่อแก้ปัญหา Monolith ขนาดใหญ่

**จุดเริ่มต้น - Monolith ปัญหา:**

```
Airbnb Monolith (2014):
┌─────────────────────────────────────────────────────────────────┐
│                    Airbnb Monolith                               │
│                                                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │  Search  │ │ Booking  │ │ Payment  │ │   Messaging      │  │
│  │  Module  │ │  Module  │ │  Module  │ │   Module         │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
│                                                                   │
│  ปัญหา: ทีม 100+ คน ทำงานบน Codebase เดียว                    │
│  - Deploy ใหม่ต้อง Test ทั้งหมด                               │
│  - Bug ในส่วนหนึ่งทำให้ทั้งระบบล่ม                           │
│  - Scale ไม่ได้เลือกเฉพาะส่วนที่ต้องการ                      │
└─────────────────────────────────────────────────────────────────┘
```

**การ Migrate ไปยัง Kubernetes:**

```
Phase 1 (2016-2017): Strangler Fig Pattern
Monolith ──extract──► Search Service (Kubernetes)
Monolith ──extract──► Pricing Service (Kubernetes)
Monolith (ยังคงรัน)

Phase 2 (2018-2019): Microservices Migration
Monolith ──extract──► Booking Service (Kubernetes)
Monolith ──extract──► Payment Service (Kubernetes)
Monolith ──extract──► Review Service (Kubernetes)
Monolith (เหลือแค่บางส่วน)

Phase 3 (2020+): Full Kubernetes
ทุก Service รันบน Kubernetes
Monolith ถูก Decommission
```

**ผลลัพธ์ที่ Airbnb ได้รับ:**
- Engineer Velocity เพิ่มขึ้น 3x (Deploy ได้บ่อยขึ้น)
- Resource Cost ลดลง 40% จาก Better Resource Utilization
- Deployment Frequency เพิ่มจาก 2 ครั้ง/สัปดาห์ เป็น 50+ ครั้ง/วัน
- Mean Time to Recovery (MTTR) ลดจาก 2 ชั่วโมง เหลือ 10 นาที

---

### Spotify บน Kubernetes

Spotify ใช้ Kubernetes เพื่อจัดการ Machine Learning Pipelines และ Microservices

**Spotify Infrastructure ปัจจุบัน:**
- 500+ Microservices บน Kubernetes
- มากกว่า 300 Engineers ใช้ Kubernetes ทุกวัน
- Deploy มากกว่า 200 ครั้งต่อวัน

**Backstage - Developer Portal:**

Spotify สร้าง Backstage ซึ่งปัจจุบันเป็น CNCF Project เพื่อแก้ปัญหา "Service Discovery สำหรับ Engineers":

```
ปัญหา: มี 500+ Services แล้วใครรู้ว่า Service ไหนทำอะไร?

Backstage Solution:
┌─────────────────────────────────────────────────────────────────┐
│                     Backstage Portal                             │
│                                                                   │
│  Service Catalog:                                                 │
│  ┌────────────────────┐  ┌────────────────────┐                │
│  │ playlist-service   │  │ recommendation-svc  │                │
│  │ Owner: Team Music  │  │ Owner: Team ML      │                │
│  │ SLO: 99.9%         │  │ SLO: 99.5%          │                │
│  │ Docs: /wiki/...    │  │ Docs: /wiki/...      │                │
│  └────────────────────┘  └────────────────────┘                │
│                                                                   │
│  Tech Docs, API Docs, Ownership, On-call Info - ครบในที่เดียว  │
└─────────────────────────────────────────────────────────────────┘
```

**Kubernetes + ML Pipelines:**

```yaml
# ตัวอย่าง Spotify ML Training Job
apiVersion: batch/v1
kind: Job
metadata:
  name: recommendation-model-training
  namespace: ml-platform
spec:
  template:
    spec:
      containers:
      - name: trainer
        image: spotify/ml-trainer:v3.2
        resources:
          requests:
            memory: "16Gi"
            cpu: "8"
            nvidia.com/gpu: "2"
          limits:
            memory: "32Gi"
            cpu: "16"
            nvidia.com/gpu: "2"
        env:
        - name: TRAINING_DATA_PATH
          value: "gs://spotify-ml-data/training/2024"
        - name: MODEL_OUTPUT_PATH
          value: "gs://spotify-models/recommendation/latest"
        - name: EPOCHS
          value: "100"
      restartPolicy: OnFailure
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-tesla-a100
```

---

## Kubernetes Roadmap 2024-2025

### Kubernetes Release Cycle

```
Kubernetes Release Timeline:
                                                          
2024:
  K8s 1.29 (Jan 2024) ─────► Stable
  K8s 1.30 (Apr 2024) ─────► Stable  
  K8s 1.31 (Aug 2024) ─────► Stable
  K8s 1.32 (Dec 2024) ─────► Latest

2025:
  K8s 1.33 (Apr 2025) ─────► Planned
  K8s 1.34 (Aug 2025) ─────► Planned
  K8s 1.35 (Dec 2025) ─────► Planned

รูปแบบ: 3 Releases ต่อปี (ทุก 4 เดือน)
Support: แต่ละ Version ได้รับ Support 14 เดือน
```

### Features ที่น่าสนใจใน Kubernetes 1.29-1.32

**Kubernetes 1.29 (Mandala):**

```
Feature Highlights:
├── ReadWriteOncePod PV Access Mode - GA
├── KMS v2 Encryption - GA
├── Node Volume Expansion - GA  
├── Pod Scheduling Readiness - GA
└── Mixed Version Proxy - Alpha
```

**Kubernetes 1.30 (Uwubernetes):**

```
Feature Highlights:
├── Structured Authentication Configuration - Beta
├── Recursive Read-only Mounts - Beta
├── Speed up recursive SELinux label change - GA
├── Custom profiling in kubectl debug - Alpha
└── Traffic Distribution for Services - Alpha
```

**Kubernetes 1.31 (Elli):**

```
Feature Highlights:
├── AppArmor Support - GA
├── Persistent Volume Last Phase Transition Time - GA
├── Resource Health Status in Pod Status - Alpha
├── Fine-grained SupplementalGroups control - Alpha
└── Informer-based Watch Cache - Alpha
```

**Kubernetes 1.32 (Penelope):**

```
Feature Highlights:
├── Multiple Service CIDRs - GA
├── Asynchronous Preemption in Scheduler - Alpha
├── DRA Structured Parameters - Beta
├── Job API managed-by mechanism - GA
└── OOMKill Policy for Sidecar Containers - Alpha
```

### Kubernetes Future Roadmap

**Workload Management ปัจจุบันและอนาคต:**

```
Evolution of Kubernetes Workloads:

2014-2016: Basic Workloads
  ├── Pod (Basic unit)
  ├── ReplicationController
  └── Service

2016-2018: Advanced Workloads
  ├── Deployment
  ├── DaemonSet  
  ├── StatefulSet
  ├── Job/CronJob
  └── HPA (Horizontal Pod Autoscaler)

2018-2022: Extended Capabilities
  ├── Custom Resources (CRDs)
  ├── Operators
  ├── VPA (Vertical Pod Autoscaler)
  └── KEDA (Event-driven Autoscaling)

2022-2025: Next Generation
  ├── Dynamic Resource Allocation (DRA) ← GPU, FPGA
  ├── Sidecar Containers (Native)
  ├── Job Success Policy
  ├── Elastic Indexed Jobs
  └── AI/ML Workload Optimizations
```

**AI/ML Integration - อนาคตของ Kubernetes:**

```
Kubernetes + AI/ML (2024-2025):

1. GPU Resource Management
   - Dynamic GPU Allocation ผ่าน DRA
   - GPU Time-sharing
   - MIG (Multi-Instance GPU) Support

2. Large Model Training
   - Distributed Training Operators (Kubeflow)
   - Checkpointing Support
   - Node Anti-affinity สำหรับ Training Jobs

3. Inference Serving
   - KServe (Model Serving Platform)
   - Auto-scaling based on Model Latency
   - A/B Testing สำหรับ Models

ตัวอย่าง AI Workload บน Kubernetes:
┌─────────────────────────────────────────────────────────────────┐
│                     MLOps Pipeline on K8s                        │
│                                                                   │
│  Data Prep     Training        Evaluation       Serving          │
│  ┌────────┐   ┌────────────┐  ┌────────────┐  ┌────────────┐  │
│  │ Spark  │──►│  PyTorch   │─►│  Evaluate  │─►│  KServe    │  │
│  │  Job   │   │  Training  │  │   Model    │  │  Serving   │  │
│  │ (K8s)  │   │  (4x GPU)  │  │   (K8s)   │  │  (K8s)    │  │
│  └────────┘   └────────────┘  └────────────┘  └────────────┘  │
│                                                                   │
│  Argo Workflows - Orchestration Engine                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workshop ละเอียด: ติดตั้ง Minikube และ Deploy Node.js App

### ขั้นตอนที่ 1: ติดตั้ง Minikube

**สำหรับ Linux (Ubuntu/Debian):**

```bash
# 1. ดาวน์โหลด Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# 2. ติดตั้ง
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# 3. ตรวจสอบ Version
minikube version
# minikube version: v1.32.0

# 4. ดาวน์โหลด kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 5. ติดตั้ง kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 6. ตรวจสอบ kubectl
kubectl version --client
```

**สำหรับ macOS:**

```bash
# ใช้ Homebrew
brew install minikube kubectl

# หรือดาวน์โหลดโดยตรง
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

**สำหรับ Windows:**

```powershell
# ใช้ Chocolatey
choco install minikube kubernetes-cli

# หรือใช้ Winget
winget install minikube
winget install Kubernetes.kubectl
```

### ขั้นตอนที่ 2: เริ่มต้น Minikube Cluster

```bash
# เริ่ม Cluster พร้อมกำหนด Resources
minikube start \
  --cpus=4 \
  --memory=8192 \
  --disk-size=50g \
  --driver=docker \
  --kubernetes-version=v1.32.0

# ตรวจสอบ Status
minikube status
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured

# ดู Nodes
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.32.0

# เปิด Dashboard (Optional)
minikube dashboard
```

### ขั้นตอนที่ 3: สร้าง Node.js Application

```bash
# สร้าง Project Directory
mkdir my-first-k8s-app
cd my-first-k8s-app

# สร้าง Node.js App
cat > app.js << 'EOF'
const http = require('http');
const os = require('os');

const PORT = process.env.PORT || 3000;
const APP_VERSION = process.env.APP_VERSION || 'v1.0';

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'healthy', version: APP_VERSION }));
    return;
  }
  
  const hostname = os.hostname();
  const response = {
    message: `Hello from Kubernetes! 🎉`,
    hostname: hostname,
    version: APP_VERSION,
    timestamp: new Date().toISOString(),
    nodeVersion: process.version
  };
  
  console.log(`Request received on ${hostname} at ${new Date().toISOString()}`);
  
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(response, null, 2));
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Hostname: ${os.hostname()}`);
  console.log(`Version: ${APP_VERSION}`);
});
EOF

# สร้าง package.json
cat > package.json << 'EOF'
{
  "name": "my-k8s-app",
  "version": "1.0.0",
  "description": "My first Kubernetes App",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
EOF
```

### ขั้นตอนที่ 4: สร้าง Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Base Image
FROM node:18-alpine

# Set Working Directory
WORKDIR /app

# Copy package files first (สำหรับ Layer Caching)
COPY package*.json ./

# Install Dependencies
RUN npm install --production

# Copy Application Code
COPY app.js .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs

# Switch to non-root user
USER nodeuser

# Expose Port
EXPOSE 3000

# Health Check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start Application
CMD ["node", "app.js"]
EOF

# Build Image โดยใช้ Minikube's Docker daemon
eval $(minikube docker-env)

# Build Image
docker build -t my-k8s-app:v1.0 .

# ตรวจสอบ Image
docker images | grep my-k8s-app
```

### ขั้นตอนที่ 5: สร้าง Kubernetes Manifests

```bash
# สร้าง Directory สำหรับ K8s Files
mkdir k8s

# สร้าง Deployment
cat > k8s/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    version: v1.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0
    spec:
      containers:
      - name: my-app
        image: my-k8s-app:v1.0
        imagePullPolicy: Never  # ใช้ Local Image
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: PORT
          value: "3000"
        - name: APP_VERSION
          value: "v1.0"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
EOF

# สร้าง Service
cat > k8s/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: default
  labels:
    app: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 3000
    nodePort: 30080
EOF

# สร้าง HPA (Horizontal Pod Autoscaler)
cat > k8s/hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
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
EOF
```

### ขั้นตอนที่ 6: Deploy ไปยัง Kubernetes

```bash
# Apply Manifests
kubectl apply -f k8s/

# ตรวจสอบ Deployment
kubectl get deployments
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# my-app   3/3     3            3           30s

# ตรวจสอบ Pods
kubectl get pods -o wide
# NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE
# my-app-5d4b7f9c6-abc12    1/1     Running   0          30s   172.17.0.3   minikube
# my-app-5d4b7f9c6-def34    1/1     Running   0          30s   172.17.0.4   minikube
# my-app-5d4b7f9c6-ghi56    1/1     Running   0          30s   172.17.0.5   minikube

# ตรวจสอบ Service
kubectl get services
# NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# my-app-service   NodePort   10.109.12.34    <none>        80:30080/TCP   30s

# เข้าถึง Application
curl $(minikube ip):30080
# {
#   "message": "Hello from Kubernetes! 🎉",
#   "hostname": "my-app-5d4b7f9c6-abc12",
#   "version": "v1.0",
#   "timestamp": "2024-01-15T10:30:00.000Z",
#   "nodeVersion": "v18.19.0"
# }

# เรียกซ้ำๆ เพื่อดู Load Balancing
for i in {1..6}; do
  curl -s $(minikube ip):30080 | python3 -m json.tool | grep hostname
done
# "hostname": "my-app-5d4b7f9c6-abc12"
# "hostname": "my-app-5d4b7f9c6-def34"
# "hostname": "my-app-5d4b7f9c6-ghi56"
# "hostname": "my-app-5d4b7f9c6-abc12"
# "hostname": "my-app-5d4b7f9c6-def34"
# "hostname": "my-app-5d4b7f9c6-ghi56"
# ← Load Balancing ทำงาน! แต่ละ Request ไปคนละ Pod
```

---

## ทดสอบ Self-healing

### การทดสอบ Pod Self-healing

```bash
# ดู Pods ปัจจุบัน
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# my-app-5d4b7f9c6-abc12    1/1     Running   0          5m
# my-app-5d4b7f9c6-def34    1/1     Running   0          5m
# my-app-5d4b7f9c6-ghi56    1/1     Running   0          5m

# ลบ Pod หนึ่งตัว (จำลองว่า Pod ล้มเหลว)
kubectl delete pod my-app-5d4b7f9c6-abc12

# ดู Pods ใหม่ทันที
kubectl get pods
# NAME                      READY   STATUS              RESTARTS   AGE
# my-app-5d4b7f9c6-def34    1/1     Running             0          5m
# my-app-5d4b7f9c6-ghi56    1/1     Running             0          5m
# my-app-5d4b7f9c6-xyz99    0/1     ContainerCreating   0          2s  ← ใหม่!

# รอสักครู่
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# my-app-5d4b7f9c6-def34    1/1     Running   0          5m
# my-app-5d4b7f9c6-ghi56    1/1     Running   0          5m
# my-app-5d4b7f9c6-xyz99    1/1     Running   0          15s  ← Pod ใหม่พร้อมแล้ว!

# Kubernetes สร้าง Pod ใหม่โดยอัตโนมัติ เพราะ Desired State = 3 Replicas
```

### การทดสอบ Node Failure Simulation

```bash
# ดู Nodes
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   10m   v1.32.0

# ใน Production ถ้า Node ล้ม:
# 1. Controller Manager ตรวจจับว่า Node ไม่ตอบสนอง
# 2. Pods บน Node นั้นถูก Evict
# 3. Pods ถูก Reschedule ไปยัง Node อื่น

# จำลองด้วย Minikube (เพิ่ม Node ก่อน)
minikube node add
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# minikube       Ready    control-plane   15m   v1.32.0
# minikube-m02   Ready    <none>          1m    v1.32.0

# ลบ Node
minikube node delete minikube-m02

# ดู Pods ที่ถูก Reschedule
kubectl get pods -o wide
```

### การทดสอบ Liveness Probe

```bash
# Deploy App ที่มี Liveness Probe ที่จำลองความล้มเหลว
cat > k8s/test-liveness.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      touch /tmp/healthy
      sleep 30
      rm -f /tmp/healthy
      sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
EOF

kubectl apply -f k8s/test-liveness.yaml

# ดู Pod Status (หลัง 35 วินาที Probe จะ Fail)
kubectl get pod liveness-test --watch
# NAME             READY   STATUS    RESTARTS   AGE
# liveness-test    1/1     Running   0          0s
# liveness-test    1/1     Running   0          30s
# liveness-test    0/1     Running   1          65s   ← Restart!
# liveness-test    1/1     Running   1          67s   ← กลับมาแล้ว

# ดู Events
kubectl describe pod liveness-test | grep Events -A 20
# Events:
#   Type     Reason     Message
#   ----     ------     -------
#   Warning  Unhealthy  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
#   Normal   Killing    Container liveness-test failed liveness probe, will be restarted

# ลบ Pod ทดสอบ
kubectl delete pod liveness-test
```

---

## ทดสอบ Scaling

### Manual Scaling

```bash
# ดู Deployment ปัจจุบัน
kubectl get deployment my-app
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# my-app   3/3     3            3           10m

# Scale ขึ้นเป็น 6 Replicas
kubectl scale deployment my-app --replicas=6

# ดู Pods ใหม่
kubectl get pods
# NAME                      READY   STATUS              RESTARTS   AGE
# my-app-5d4b7f9c6-abc12    1/1     Running             0          10m
# my-app-5d4b7f9c6-def34    1/1     Running             0          10m
# my-app-5d4b7f9c6-ghi56    1/1     Running             0          10m
# my-app-5d4b7f9c6-jkl78    0/1     ContainerCreating   0          2s
# my-app-5d4b7f9c6-mno90    0/1     ContainerCreating   0          2s
# my-app-5d4b7f9c6-pqr12    0/1     ContainerCreating   0          2s

# รอสักครู่
kubectl get pods
# ทั้ง 6 Pods Running แล้ว

# Scale ลงเป็น 2 Replicas
kubectl scale deployment my-app --replicas=2

# ดู Pods ที่ถูกยกเลิก
kubectl get pods
# NAME                      READY   STATUS        RESTARTS   AGE
# my-app-5d4b7f9c6-abc12    1/1     Running       0          12m
# my-app-5d4b7f9c6-def34    1/1     Running       0          12m
# my-app-5d4b7f9c6-ghi56    1/1     Terminating   0          12m  ← กำลังหยุด
# my-app-5d4b7f9c6-jkl78    1/1     Terminating   0          2m   ← กำลังหยุด
# ...
```

### Auto Scaling ด้วย HPA

```bash
# Apply HPA
kubectl apply -f k8s/hpa.yaml

# ดู HPA Status
kubectl get hpa
# NAME         REFERENCE           TARGETS          MINPODS   MAXPODS   REPLICAS
# my-app-hpa   Deployment/my-app   2%/70%, 1%/80%   2         10        2

# จำลอง Load (ต้อง Enable Metrics Server ก่อน)
minikube addons enable metrics-server

# สร้าง Load Test
kubectl run load-test --image=busybox:1.35 --restart=Never -- \
  sh -c "while true; do wget -q -O- http://my-app-service/; done"

# ดู HPA ทำงาน
kubectl get hpa --watch
# NAME         REFERENCE           TARGETS           MINPODS   MAXPODS   REPLICAS
# my-app-hpa   Deployment/my-app   2%/70%            2         10        2
# my-app-hpa   Deployment/my-app   85%/70%           2         10        2
# my-app-hpa   Deployment/my-app   85%/70%           2         10        4   ← Scale Up!
# my-app-hpa   Deployment/my-app   70%/70%           2         10        4

# หยุด Load Test
kubectl delete pod load-test

# ดู Scale Down (ใช้เวลาสักครู่)
kubectl get hpa --watch
# Replicas จะลดลงกลับมาเป็น 2
```

---

## ทดสอบ Rolling Update

### Update Application Version

```bash
# สร้าง Version 2 ของ App
cat > app-v2.js << 'EOF'
const http = require('http');
const os = require('os');

const PORT = process.env.PORT || 3000;
const APP_VERSION = process.env.APP_VERSION || 'v2.0';

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'healthy', version: APP_VERSION }));
    return;
  }
  
  const hostname = os.hostname();
  const response = {
    message: `Hello from Kubernetes v2! 🚀 NEW FEATURE!`,
    hostname: hostname,
    version: APP_VERSION,
    timestamp: new Date().toISOString(),
    features: ['feature-1', 'feature-2', 'new-feature-3'],  // ← ฟีเจอร์ใหม่
    nodeVersion: process.version
  };
  
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(response, null, 2));
});

server.listen(PORT, () => {
  console.log(`Server v2 running on port ${PORT}`);
});
EOF

cp app-v2.js app.js

# Build Image Version 2
docker build -t my-k8s-app:v2.0 .

# ทำ Rolling Update
kubectl set image deployment/my-app my-app=my-k8s-app:v2.0

# ดู Rolling Update Process
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "my-app" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "my-app" rollout to finish: 1 old replicas are pending termination...
# deployment "my-app" successfully rolled out

# ดู Pods ระหว่าง Update (ใน Terminal อื่น)
kubectl get pods --watch
# NAME                      READY   STATUS              RESTARTS
# my-app-5d4b7f9c6-abc12    1/1     Running             0  ← v1
# my-app-5d4b7f9c6-def34    1/1     Running             0  ← v1  
# my-app-5d4b7f9c6-ghi56    1/1     Running             0  ← v1
# my-app-7e5c8g0d7-xxx11    0/1     ContainerCreating   0  ← v2 กำลังสร้าง
# my-app-7e5c8g0d7-xxx11    1/1     Running             0  ← v2 พร้อมแล้ว
# my-app-5d4b7f9c6-abc12    1/1     Terminating         0  ← v1 กำลังหยุด
# ...

# ดู Rollout History
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

### Rollback กลับ Version เดิม

```bash
# สมมติว่า v2 มี Bug - ต้อง Rollback
kubectl rollout undo deployment/my-app

# หรือ Rollback ไปยัง Revision ที่ระบุ
kubectl rollout undo deployment/my-app --to-revision=1

# ดู Status
kubectl rollout status deployment/my-app
# deployment "my-app" successfully rolled out

# ตรวจสอบว่า Version กลับมาแล้ว
curl $(minikube ip):30080 | python3 -m json.tool | grep version
# "version": "v1.0"
```

---

## แบบฝึกหัด 10 ข้อพร้อมเฉลย

### ข้อที่ 1: Deploy Nginx บน Kubernetes

**โจทย์**: Deploy Nginx Web Server โดยมี 3 Replicas และสร้าง NodePort Service

**เฉลย**:

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "25m"
          limits:
            memory: "64Mi"
            cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30090
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods,services
curl $(minikube ip):30090
```

---

### ข้อที่ 2: Scale Deployment

**โจทย์**: Scale Nginx Deployment ที่สร้างในข้อ 1 จาก 3 เป็น 5 Replicas

**เฉลย**:

```bash
# วิธีที่ 1: kubectl scale
kubectl scale deployment nginx-web --replicas=5

# วิธีที่ 2: แก้ไข YAML แล้ว apply ใหม่
kubectl edit deployment nginx-web
# แก้ replicas: 3 เป็น replicas: 5

# วิธีที่ 3: kubectl patch
kubectl patch deployment nginx-web -p '{"spec":{"replicas":5}}'

# ตรวจสอบ
kubectl get deployment nginx-web
# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-web   5/5     5            5           5m
```

---

### ข้อที่ 3: ดู Logs จาก Pod

**โจทย์**: แสดง Logs ล่าสุด 50 บรรทัดจาก Nginx Pod

**เฉลย**:

```bash
# ดู Logs จาก Pod หนึ่งตัว
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME --tail=50

# ดู Logs แบบ Stream
kubectl logs $POD_NAME -f

# ดู Logs จากทุก Pod ที่มี Label app=nginx
kubectl logs -l app=nginx --tail=50

# ดู Logs พร้อม Timestamp
kubectl logs $POD_NAME --timestamps=true --tail=50
```

---

### ข้อที่ 4: Debug Pod ที่ไม่ Start

**โจทย์**: Pod ไม่ Start ต้องหาสาเหตุอย่างไร

**เฉลย**:

```bash
# Step 1: ดู Pod Status
kubectl get pods
# NAME              READY   STATUS             RESTARTS   AGE
# bad-pod-xxx       0/1     ImagePullBackOff   0          2m

# Step 2: ดู Events
kubectl describe pod bad-pod-xxx
# Events:
#   Warning  Failed     Failed to pull image "wrong-image:latest":
#            rpc error: ...no such host

# Step 3: ดู Logs (ถ้า Container เคย Start)
kubectl logs bad-pod-xxx

# Step 4: ดู Logs จาก Previous Container
kubectl logs bad-pod-xxx --previous

# Step 5: Exec เข้า Container เพื่อ Debug
kubectl exec -it bad-pod-xxx -- sh

# Step 6: ตรวจสอบ YAML ที่ Apply
kubectl get pod bad-pod-xxx -o yaml
```

---

### ข้อที่ 5: สร้าง ConfigMap และใช้ใน Pod

**โจทย์**: สร้าง ConfigMap ที่มีค่า Config จากนั้นใช้ใน Pod

**เฉลย**:

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_PORT: "3000"
  MAX_CONNECTIONS: "100"
  LOG_LEVEL: "info"
  DATABASE_URL: "postgres://db:5432/myapp"

---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: nginx:alpine
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: SPECIFIC_KEY
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
```

```bash
kubectl apply -f configmap.yaml

# ตรวจสอบ Environment Variables ใน Pod
kubectl exec app-with-config -- env | grep APP
# APP_ENV=production
# APP_PORT=3000
```

---

### ข้อที่ 6: สร้าง Secret

**โจทย์**: สร้าง Secret สำหรับ Database Password และใช้ใน Pod

**เฉลย**:

```bash
# สร้าง Secret จาก Command Line
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=SuperSecret123! \
  --from-literal=DB_NAME=myapp

# หรือสร้างจาก YAML (ต้อง Base64 encode ก่อน)
echo -n "admin" | base64          # YWRtaW4=
echo -n "SuperSecret123!" | base64  # U3VwZXJTZWNyZXQxMjMh
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: YWRtaW4=
  DB_PASSWORD: U3VwZXJTZWNyZXQxMjMh
  DB_NAME: bXlhcHA=

---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret
spec:
  containers:
  - name: app
    image: nginx:alpine
    envFrom:
    - secretRef:
        name: db-secret
```

```bash
kubectl apply -f secret.yaml
kubectl exec app-with-secret -- env | grep DB
# DB_USER=admin
# DB_PASSWORD=SuperSecret123!
# DB_NAME=myapp
```

---

### ข้อที่ 7: Rolling Update และ Rollback

**โจทย์**: Update Nginx Deployment จาก version 1.25 เป็น 1.26 แล้ว Rollback กลับ

**เฉลย**:

```bash
# ดู Version ปัจจุบัน
kubectl get deployment nginx-web -o jsonpath='{.spec.template.spec.containers[0].image}'
# nginx:1.25-alpine

# Update Image พร้อม Annotation (สำหรับ History)
kubectl set image deployment/nginx-web nginx=nginx:1.26-alpine \
  --record  # deprecated แต่ยังใช้ได้

# หรือใช้ Annotate แทน
kubectl annotate deployment nginx-web \
  kubernetes.io/change-cause="Update nginx to 1.26-alpine"
kubectl set image deployment/nginx-web nginx=nginx:1.26-alpine

# ดู Rollout Status
kubectl rollout status deployment/nginx-web

# ดู History
kubectl rollout history deployment/nginx-web
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         Update nginx to 1.26-alpine

# Rollback
kubectl rollout undo deployment/nginx-web

# ดู Version หลัง Rollback
kubectl get deployment nginx-web -o jsonpath='{.spec.template.spec.containers[0].image}'
# nginx:1.25-alpine
```

---

### ข้อที่ 8: Resource Limits

**โจทย์**: สร้าง Pod ที่มีการกำหนด Resource Requests และ Limits

**เฉลย**:

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        memory: "64Mi"     # ขั้นต่ำที่ต้องการ
        cpu: "100m"        # 100 millicores = 0.1 CPU
      limits:
        memory: "128Mi"    # สูงสุดที่ใช้ได้
        cpu: "200m"        # 200 millicores = 0.2 CPU
```

```bash
kubectl apply -f resource-pod.yaml

# ดู Resource Usage
kubectl top pod resource-demo
# NAME            CPU(cores)   MEMORY(bytes)
# resource-demo   1m           3Mi

# ดู Resource Spec
kubectl describe pod resource-demo | grep -A 8 "Limits:"
# Limits:
#   cpu:     200m
#   memory:  128Mi
# Requests:
#   cpu:     100m
#   memory:  64Mi
```

---

### ข้อที่ 9: Namespace Isolation

**โจทย์**: สร้าง 2 Namespaces (dev, prod) และ Deploy App แยกกัน

**เฉลย**:

```bash
# สร้าง Namespaces
kubectl create namespace dev
kubectl create namespace prod

# Deploy ไปยัง Namespace dev
kubectl create deployment nginx-dev --image=nginx:1.25 \
  --replicas=1 --namespace=dev

# Deploy ไปยัง Namespace prod
kubectl create deployment nginx-prod --image=nginx:1.26 \
  --replicas=3 --namespace=prod

# ดู Resources ใน Namespace ต่างๆ
kubectl get all -n dev
kubectl get all -n prod

# ดู Resources ทุก Namespace
kubectl get pods --all-namespaces
# หรือ
kubectl get pods -A

# ลบ Namespace ทั้งหมด (ลบ Resources ใน Namespace ด้วย)
kubectl delete namespace dev
```

---

### ข้อที่ 10: Port Forward เพื่อ Debug

**โจทย์**: เข้าถึง Service ใน Kubernetes โดยไม่ต้องสร้าง NodePort

**เฉลย**:

```bash
# Port Forward จาก Pod ไปยัง Local
kubectl port-forward pod/nginx-web-xxx 8080:80
# Forwarding from 127.0.0.1:8080 -> 80

# ใน Terminal อื่น
curl http://localhost:8080
# <html>Welcome to nginx!</html>

# Port Forward จาก Service ไปยัง Local
kubectl port-forward service/nginx-service 8080:80

# Port Forward ไปยัง Namespace อื่น
kubectl port-forward -n prod service/nginx-service 8080:80

# Port Forward แบบ Background (ใช้ & หรือ nohup)
kubectl port-forward service/nginx-service 8080:80 &
PF_PID=$!

# ทดสอบ
curl http://localhost:8080

# หยุด Port Forward
kill $PF_PID
```

---

## สรุป Part 01

ใน Part แรกนี้เราได้เรียนรู้:

1. **Kubernetes คืออะไร** - ระบบ Orchestration สำหรับ Containerized Applications
2. **ทำไมต้องใช้ Kubernetes** - Self-healing, Scaling, Rolling Update, Service Discovery
3. **Kubernetes vs Traditional VMs** - ประหยัด Resource, Deploy เร็วกว่า, ยืดหยุ่นกว่า
4. **Real-world Case Studies** - Netflix, Airbnb, Spotify ใช้ Kubernetes แก้ปัญหาจริง
5. **Kubernetes Roadmap** - Features ใหม่ใน 2024-2025
6. **Workshop จริง** - ติดตั้ง Minikube, Deploy Node.js App, ทดสอบ Features ต่างๆ

### Checklist ก่อนไปต่อ

- [ ] ติดตั้ง Minikube และ kubectl ได้สำเร็จ
- [ ] Deploy App ได้สำเร็จและเข้าถึงได้
- [ ] ทดสอบ Self-healing (ลบ Pod แล้วเห็น Pod ใหม่)
- [ ] ทดสอบ Scaling (Manual Scale ขึ้นและลง)
- [ ] ทดสอบ Rolling Update และ Rollback
- [ ] ทำแบบฝึกหัดครบ 10 ข้อ

---

*ต่อไป: [Part 02: Container Orchestration](./part-02-container-orchestration.md)*
