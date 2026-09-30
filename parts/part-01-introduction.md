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
