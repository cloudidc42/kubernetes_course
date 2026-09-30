# Part 02: Container Orchestration

## สารบัญ
- [Container Orchestration คืออะไร](#container-orchestration-คืออะไร)
- [ปัญหาที่ Container Orchestration แก้ไข](#ปัญหาที่-container-orchestration-แก้ไข)
- [เปรียบเทียบ Kubernetes vs Docker Swarm vs Nomad](#เปรียบเทียบ-kubernetes-vs-docker-swarm-vs-nomad)
- [Kubernetes Ecosystem](#kubernetes-ecosystem)
- [Workshop: ทดลองรัน Container เดี่ยวๆ แล้วเปรียบเทียบกับ Orchestrated](#workshop)

---

## Container Orchestration คืออะไร

**Container Orchestration** คือกระบวนการ Automate การ Deploy, Manage, Scale, และ Network ของ Containers

ลองนึกภาพว่าคุณมี Orchestra (วงดนตรี):
- ไม่มี Conductor = นักดนตรีแต่ละคนเล่นตามใจตัวเอง = เสียงที่ได้คือ Chaos
- มี Conductor = ทุกคนเล่นประสานกัน = เสียงดนตรีที่สวยงาม

Container Orchestration คือ **Conductor** ของ Containers

```
Without Orchestration:
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│App v1.0│  │App v1.0│  │App v1.2│  │App v1.1│
│Manual  │  │ Dead!  │  │ ???    │  │ OK     │
│Deploy  │  │ Nobody │  │Config  │  │        │
│        │  │ Knows! │  │ Wrong  │  │        │
└────────┘  └────────┘  └────────┘  └────────┘
Server 1    Server 2    Server 3    Server 4
CHAOS!

With Orchestration (Kubernetes):
┌─────────────────────────────────────────────┐
│              Orchestrator                    │
│  "ต้องการ App v1.2 รัน 4 copies เสมอ"       │
└──────────┬──────────┬──────────┬────────────┘
           ▼          ▼          ▼          ▼
        ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
        │App   │  │App   │  │App   │  │App   │
        │v1.2  │  │v1.2  │  │v1.2  │  │v1.2  │
        │ OK   │  │ OK   │  │ OK   │  │ OK   │
        └──────┘  └──────┘  └──────┘  └──────┘
ORDER AND HARMONY!
```

### หน้าที่หลักของ Container Orchestration

```
┌──────────────────────────────────────────────────────────┐
│               Container Orchestration Functions           │
│                                                          │
│  1. SCHEDULING                                           │
│     "วาง Container ตัวไหนบน Node ไหน"                   │
│     - คำนึงถึง Resource (CPU, RAM)                       │
│     - คำนึงถึง Constraints (Node Labels, Taints)        │
│                                                          │
│  2. SERVICE DISCOVERY                                    │
│     "Container หากันเจออย่างไร"                          │
│     - Internal DNS                                       │
│     - Service Registry                                   │
│                                                          │
│  3. LOAD BALANCING                                       │
│     "กระจาย Traffic ให้สมดุล"                            │
│     - Round Robin                                        │
│     - Least Connection                                   │
│                                                          │
│  4. HEALTH MONITORING                                    │
│     "ตรวจสอบว่า Container ยังทำงานอยู่"                  │
│     - Liveness Probe                                     │
│     - Readiness Probe                                    │
│                                                          │
│  5. AUTO-SCALING                                         │
│     "เพิ่ม/ลด Container ตามความต้องการ"                  │
│     - Horizontal Pod Autoscaler                          │
│     - Vertical Pod Autoscaler                            │
│     - Cluster Autoscaler                                 │
│                                                          │
│  6. ROLLING UPDATES                                      │
│     "Update โดยไม่มี Downtime"                           │
│     - Deploy ทีละส่วน                                    │
│     - Rollback ได้ถ้ามีปัญหา                             │
│                                                          │
│  7. RESOURCE MANAGEMENT                                  │
│     "จัดสรร CPU/RAM ให้เหมาะสม"                          │
│     - Requests & Limits                                  │
│     - Quality of Service                                 │
└──────────────────────────────────────────────────────────┘
```

---

## ปัญหาที่ Container Orchestration แก้ไข

### Problem 1: Container ล้มแล้วไม่มีใครดู

**สถานการณ์:**
```bash
# ไม่มี Orchestration
docker run -d --name webapp nginx

# ถ้า Container ล้มเพราะ OOM หรือ Bug
# ไม่มีใคร Restart ให้อัตโนมัติ
# User จะเห็น Error จนกว่าคนจะมา Restart

# ต้องทำเอง:
docker start webapp  # Restart ด้วยมือ
```

**แก้ไขด้วย Kubernetes:**
```yaml
# Kubernetes จัดการให้อัตโนมัติ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: webapp
        image: nginx
        # Kubernetes จะ Restart Container ถ้า Crash
        # Kubernetes จะสร้าง Pod ใหม่ถ้า Pod ล้ม
```

### Problem 2: Scale ยากและช้า

**สถานการณ์:**
```bash
# ไม่มี Orchestration - ต้อง Scale ด้วยมือ
# ขั้นตอน 1: SSH เข้า Server ใหม่
ssh user@new-server

# ขั้นตอน 2: ติดตั้ง Docker
apt-get install docker.io

# ขั้นตอน 3: Pull Image
docker pull myapp:latest

# ขั้นตอน 4: Run Container
docker run -d myapp:latest

# ขั้นตอน 5: Update Load Balancer
# ... แก้ Config ด้วยมือ

# ทั้งหมดนี้ใช้เวลา 30-60 นาที!
```

**แก้ไขด้วย Kubernetes:**
```bash
# ขยาย Scale ด้วยคำสั่งเดียว
kubectl scale deployment myapp --replicas=10

# หรือให้ Auto Scale
kubectl autoscale deployment myapp \
  --cpu-percent=50 \
  --min=3 \
  --max=100

# ทั้งหมดนี้ใช้เวลา < 1 นาที!
```

### Problem 3: Zero-downtime Deployment ทำได้ยาก

**สถานการณ์:**
```bash
# วิธีเดิม - มี Downtime
# ขั้นตอน 1: หยุด Service เก่า
docker stop myapp-v1

# ขั้นตอน 2: รัน Service ใหม่
docker run -d myapp-v2

# ช่วงระหว่าง Stop และ Start = DOWNTIME!
```

**แก้ไขด้วย Kubernetes:**
```bash
# Rolling Update - ไม่มี Downtime
kubectl set image deployment/myapp myapp=myapp:v2

# Kubernetes จะ:
# 1. สร้าง Pod ใหม่ (v2) ทีละตัว
# 2. รอให้ Pod ใหม่พร้อม
# 3. ลบ Pod เก่า (v1)
# 4. ทำซ้ำจนครบทุก Pod
# - ตลอดเวลา มีอย่างน้อย 1 Pod ทำงานอยู่เสมอ
```

### Problem 4: Resource Waste

**สถานการณ์:**
```
ไม่มี Orchestration:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    Server 1     │  │    Server 2     │  │    Server 3     │
│  CPU: 80%       │  │  CPU: 15%       │  │  CPU: 5%        │
│  RAM: 70%       │  │  RAM: 20%       │  │  RAM: 10%       │
│  App A (Heavy)  │  │  App B (Light)  │  │  App C (Idle)   │
└─────────────────┘  └─────────────────┘  └─────────────────┘

ปัญหา: Server 2 และ 3 ใช้ Resource น้อยมาก แต่ยังต้องจ่ายค่า Server
```

**แก้ไขด้วย Kubernetes:**
```yaml
# Kubernetes จัดการ Resource อย่างมีประสิทธิภาพ
containers:
- name: app-a
  resources:
    requests:
      cpu: "500m"   # ขอ CPU 0.5 core
      memory: "512Mi"
    limits:
      cpu: "2"      # ใช้ได้สูงสุด 2 cores
      memory: "2Gi"

# Kubernetes จะวาง Container ลง Node ที่มี Resource เพียงพอ
# และ Pack Container หลายตัวเข้า Node เดียว (Bin Packing)
```

```
กับ Kubernetes Bin Packing:
┌─────────────────────────────────────────────┐
│                   Server 1                   │
│  CPU: 75%                                    │
│  App A + App B + App C + App D              │
│  ประหยัด Server 2 และ 3 ได้เลย!             │
└─────────────────────────────────────────────┘
```

### Problem 5: Service Discovery ซับซ้อน

**สถานการณ์:**
```bash
# ไม่มี Orchestration
# App B ต้องการติดต่อ App A
# ต้อง Hard-code IP Address

# config.json ของ App B
{
  "app_a_host": "192.168.1.10",
  "app_a_port": 8080
}

# ปัญหา: ถ้า App A ย้าย IP หรือ Scale ต้องแก้ Config ด้วยมือ!
```

**แก้ไขด้วย Kubernetes:**
```yaml
# Kubernetes สร้าง DNS อัตโนมัติ
apiVersion: v1
kind: Service
metadata:
  name: app-a   # Service name = DNS name
spec:
  selector:
    app: app-a
  ports:
  - port: 8080
```

```bash
# App B เข้าถึง App A ผ่าน DNS name ได้เลย
curl http://app-a:8080/api

# ไม่ต้องรู้ IP!
# ไม่ว่า App A จะ Scale เป็นกี่ตัว หรือ IP จะเปลี่ยน
# App B ก็เข้าถึงได้เสมอ
```

---

## เปรียบเทียบ Kubernetes vs Docker Swarm vs Nomad

### ภาพรวม

```
Container Orchestration Landscape:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Kubernetes (Google/CNCF)                                   │
│  ├── Most Popular (75%+ market share)                       │
│  ├── Most Feature-rich                                      │
│  └── Steepest Learning Curve                                │
│                                                             │
│  Docker Swarm (Docker Inc.)                                 │
│  ├── Easiest to Learn                                       │
│  ├── Integrated with Docker                                 │
│  └── Less Features than K8s                                 │
│                                                             │
│  HashiCorp Nomad (HashiCorp)                               │
│  ├── Not just containers                                    │
│  ├── Can run VMs, binaries too                              │
│  └── Simpler than K8s                                       │
│                                                             │
│  Others: Mesos/Marathon, OpenShift (K8s-based)             │
└─────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบแบบละเอียด

| Feature | Kubernetes | Docker Swarm | Nomad |
|---------|-----------|--------------|-------|
| **ความยากในการเรียนรู้** | สูง | ต่ำ | ปานกลาง |
| **ความสามารถ** | สูงมาก | ปานกลาง | สูง |
| **Auto-scaling** | ✅ Built-in | ❌ ไม่มี | ✅ Enterprise |
| **Rolling Updates** | ✅ Advanced | ✅ Basic | ✅ |
| **Self-healing** | ✅ | ✅ | ✅ |
| **Multi-cloud** | ✅ | ❌ | ✅ |
| **Storage** | ✅ Advanced | ✅ Basic | ✅ |
| **Networking** | ✅ Advanced | ✅ Basic | ✅ |
| **RBAC** | ✅ | ❌ | ✅ |
| **Community** | ขนาดใหญ่มาก | ปานกลาง | เล็ก |
| **Market Share** | ~75% | ~10% | ~5% |
| **Use Case** | Enterprise | Simple Apps | Hybrid |

### Docker Swarm - จุดแข็ง/อ่อน

**จุดแข็ง:**
```bash
# ง่ายมาก - เริ่มใช้ได้ใน 5 นาที
docker swarm init
docker service create --replicas 3 --name webapp nginx
docker service scale webapp=10
```

**จุดอ่อน:**
```
- ไม่มี Auto-scaling
- Network Policy จำกัด
- Persistent Volume จำกัด
- Community เล็กกว่า Kubernetes มาก
- Docker Inc. ลด Focus บน Swarm แล้ว
```

**เมื่อใดควรใช้ Docker Swarm:**
- Project เล็กๆ ที่ต้องการ Orchestration แบบง่าย
- ทีมที่คุ้นเคยกับ Docker อยู่แล้ว
- ไม่ต้องการ Features ซับซ้อน

### HashiCorp Nomad - จุดแข็ง/อ่อน

**จุดแข็ง:**
```hcl
# Nomad Job File
job "webapp" {
  datacenters = ["dc1"]
  
  group "web" {
    count = 3
    
    task "nginx" {
      driver = "docker"
      
      config {
        image = "nginx:latest"
        ports = ["http"]
      }
    }
    
    # Nomad สามารถรัน VMs, Binaries ได้ด้วย
    task "batch-job" {
      driver = "exec"  # รัน executable โดยตรง
      config {
        command = "/usr/local/bin/my-script.sh"
      }
    }
  }
}
```

**จุดแข็ง:**
- ไม่ใช่แค่ Container - รัน VMs, Binaries ได้
- ง่ายกว่า Kubernetes
- ทำงานกับ HashiCorp Stack ได้ดี (Consul, Vault)

**เมื่อใดควรใช้ Nomad:**
- ต้องการ Orchestrate ทั้ง Container และ Non-container workloads
- ใช้ HashiCorp Stack อยู่แล้ว
- ต้องการความง่ายกว่า Kubernetes

### เมื่อไหรควรใช้ Kubernetes?

```
ใช้ Kubernetes เมื่อ:
✅ Application มีความซับซ้อน (Microservices)
✅ ต้องการ Auto-scaling
✅ ต้องการ Multi-cloud หรือ Hybrid
✅ ทีมมีเวลาเรียนรู้
✅ Long-term ต้องการ Enterprise features
✅ ต้องการ Community support ขนาดใหญ่

ไม่จำเป็นต้องใช้ Kubernetes เมื่อ:
❌ Application เล็กมาก (Single service)
❌ ทีมเล็ก ไม่มีเวลาเรียน
❌ ไม่ต้องการ Scale
❌ Simple web app ที่ Traffic ไม่เยอะ
```

---

## Kubernetes Ecosystem

Kubernetes ไม่ได้อยู่ตัวเดียว มี Ecosystem ขนาดใหญ่รอบข้าง:

```
┌──────────────────────────────────────────────────────────────────┐
│                     CNCF Landscape (Kubernetes Ecosystem)        │
│                                                                    │
│  📦 PACKAGING & DEPLOYMENT                                         │
│  ├── Helm (Package Manager like apt/npm for K8s)                  │
│  ├── Kustomize (Configuration Customization)                       │
│  └── Skaffold (Dev & Deploy tool)                                  │
│                                                                    │
│  🔍 MONITORING & OBSERVABILITY                                      │
│  ├── Prometheus (Metrics Collection)                               │
│  ├── Grafana (Visualization)                                       │
│  ├── Jaeger (Distributed Tracing)                                  │
│  └── ELK Stack (Logs)                                              │
│                                                                    │
│  🌐 NETWORKING                                                      │
│  ├── Calico (Network Policy)                                       │
│  ├── Flannel (CNI Plugin)                                          │
│  ├── Istio (Service Mesh)                                          │
│  └── Linkerd (Lightweight Service Mesh)                            │
│                                                                    │
│  💾 STORAGE                                                         │
│  ├── Rook (Storage Orchestration)                                  │
│  ├── Longhorn (Cloud Native Storage)                               │
│  └── OpenEBS (Container Native Storage)                            │
│                                                                    │
│  🔐 SECURITY                                                        │
│  ├── OPA/Gatekeeper (Policy Engine)                                │
│  ├── Falco (Runtime Security)                                      │
│  └── cert-manager (Certificate Management)                         │
│                                                                    │
│  🚀 CI/CD                                                           │
│  ├── ArgoCD (GitOps CD)                                            │
│  ├── FluxCD (GitOps CD)                                            │
│  └── Tekton (Cloud Native CI/CD)                                   │
│                                                                    │
│  ☁️ MANAGED KUBERNETES                                              │
│  ├── GKE (Google Kubernetes Engine)                                │
│  ├── EKS (Amazon Elastic Kubernetes Service)                       │
│  ├── AKS (Azure Kubernetes Service)                                │
│  └── OKE (Oracle Container Engine)                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Helm - Package Manager ของ Kubernetes

Helm เหมือนกับ `apt` สำหรับ Ubuntu หรือ `npm` สำหรับ Node.js

```bash
# ติดตั้ง Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# เพิ่ม Repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# ติดตั้ง WordPress ด้วย Helm (ง่ายมาก!)
helm install my-wordpress bitnami/wordpress

# ดู Release ที่ติดตั้ง
helm list

# ถอนการติดตั้ง
helm uninstall my-wordpress
```

**ทำไม Helm ถึงสำคัญ:**
```
ไม่มี Helm:
- ต้องสร้าง YAML files เยอะมาก
- ต้องจัดการ Dependencies เอง
- อัพเดทยาก

มี Helm:
- ติดตั้ง Complex Application ด้วยคำสั่งเดียว
- Version control ง่าย
- Rollback ง่าย
```

### Prometheus + Grafana - Monitoring Stack

```yaml
# ติดตั้ง Prometheus Stack ด้วย Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# แล้วเข้าดู Grafana Dashboard
kubectl port-forward svc/prometheus-stack-grafana 3000:80 -n monitoring
# เปิด http://localhost:3000
```

**Dashboard ที่ได้:**
```
Grafana Dashboard ประกอบด้วย:
- Cluster Overview (CPU, RAM, Network)
- Node Overview (ใช้ Resource เท่าไหร่)
- Pod Overview (แต่ละ Pod ใช้ Resource เท่าไหร่)
- Alert Management
```

### Istio - Service Mesh

```
Service Mesh ช่วยเรื่อง:
┌────────────────────────────────────────────────┐
│  Service A ──────────────► Service B           │
│       │                         │              │
│    Istio Proxy               Istio Proxy      │
│       │                         │              │
│  - TLS Encryption          - Load Balancing   │
│  - Auth/Authz               - Circuit Break   │
│  - Traffic Management       - Retry Logic     │
│  - Observability            - Canary Deploy   │
└────────────────────────────────────────────────┘
```

### ArgoCD - GitOps Continuous Delivery

```
GitOps Flow:
Developer → Push Code → Git Repo
                            │
                        ArgoCD watches
                            │
                            ▼
                    Detect Difference
                    (Git vs K8s State)
                            │
                            ▼
                    Auto-deploy to K8s
                    (Sync Git State)
```

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: k8s/
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true    # ลบ Resources ที่ไม่อยู่ใน Git
      selfHeal: true # แก้ไข Manual Changes อัตโนมัติ
```

---

## Workshop: ทดลองรัน Container เดี่ยวๆ แล้วเปรียบเทียบกับ Orchestrated

### เป้าหมาย

ใน Workshop นี้เราจะ:
1. รัน Container แบบ Manual (ไม่มี Orchestration)
2. รัน Container ผ่าน Kubernetes (มี Orchestration)
3. เปรียบเทียบประสบการณ์และความสามารถ

### Part A: Container ไม่มี Orchestration

#### ขั้นตอน A1: รัน Nginx Container

```bash
# รัน Nginx Container
docker run -d \
  --name nginx-manual \
  -p 8080:80 \
  nginx:latest

# ตรวจสอบ
docker ps
curl http://localhost:8080
```

#### ขั้นตอน A2: ทดลอง Scale (ยากมาก!)

```bash
# ต้องรัน Container ใหม่ด้วยมือ
docker run -d --name nginx-manual-2 -p 8081:80 nginx:latest
docker run -d --name nginx-manual-3 -p 8082:80 nginx:latest

# ปัญหา: Port ชนกัน! ต้องใช้ Port ต่างกัน
# ต้องมี Load Balancer แยกต่างหาก
```

#### ขั้นตอน A3: ทดลอง Self-healing (ไม่มี!)

```bash
# หยุด Container
docker stop nginx-manual

# Container หยุดเลย - ไม่มีใคร Restart ให้
docker ps  # ไม่เห็น nginx-manual แล้ว

# ต้อง Restart เอง
docker start nginx-manual
```

#### ขั้นตอน A4: ทดลอง Update Version

```bash
# ต้อง Stop เก่า แล้ว Run ใหม่ = DOWNTIME!
docker stop nginx-manual
docker rm nginx-manual
docker run -d --name nginx-manual -p 8080:80 nginx:1.25

# ช่วงระหว่างนี้ Service ไม่พร้อมให้บริการ!
```

#### ขั้นตอน A5: ทำความสะอาด

```bash
docker stop nginx-manual nginx-manual-2 nginx-manual-3
docker rm nginx-manual nginx-manual-2 nginx-manual-3
```

### Part B: Container กับ Kubernetes (Orchestrated)

#### ขั้นตอน B1: เริ่ม Minikube

```bash
minikube start
kubectl cluster-info
```

#### ขั้นตอน B2: Deploy Nginx

```bash
# สร้าง Deployment
kubectl create deployment nginx-k8s --image=nginx:latest

# ตรวจสอบ
kubectl get deployments
kubectl get pods
```

#### ขั้นตอน B3: ทดลอง Scale (ง่ายมาก!)

```bash
# Scale เป็น 5 replicas ด้วยคำสั่งเดียว
kubectl scale deployment nginx-k8s --replicas=5

# ดู Pods เพิ่มขึ้น
kubectl get pods

# ตัวอย่าง output:
# NAME                         READY   STATUS    RESTARTS   AGE
# nginx-k8s-5d88d7d5b9-abc12   1/1     Running   0          2m
# nginx-k8s-5d88d7d5b9-def34   1/1     Running   0          10s
# nginx-k8s-5d88d7d5b9-ghi56   1/1     Running   0          10s
# nginx-k8s-5d88d7d5b9-jkl78   1/1     Running   0          10s
# nginx-k8s-5d88d7d5b9-mno90   1/1     Running   0          10s
```

#### ขั้นตอน B4: ทดลอง Self-healing

```bash
# ดู Pod name
kubectl get pods

# ลบ Pod หนึ่งตัว
kubectl delete pod nginx-k8s-5d88d7d5b9-abc12

# ดู Pods ทันที - มี Pod ใหม่ถูกสร้าง!
kubectl get pods -w
# จะเห็น Pod ใหม่ถูกสร้างขึ้นมาแทนที่

# Kubernetes รักษา 5 replicas ตลอดเวลา
```

#### ขั้นตอน B5: ทดลอง Rolling Update (ไม่มี Downtime!)

```bash
# Expose เป็น Service ก่อน
kubectl expose deployment nginx-k8s --type=NodePort --port=80

# ดู Service URL
minikube service nginx-k8s --url
# เปิด URL ใน Browser - Nginx กำลังทำงาน

# Update เป็น Version ใหม่ (Rolling Update)
kubectl set image deployment/nginx-k8s nginx=nginx:1.25

# ดูกระบวนการ Update
kubectl rollout status deployment/nginx-k8s
# Waiting for deployment "nginx-k8s" rollout to finish...
# Waiting for deployment spec update to be observed...
# Waiting for rollout to finish: 2 out of 5 new replicas have been updated...
# Waiting for rollout to finish: 3 out of 5 new replicas have been updated...
# Waiting for rollout to finish: 4 out of 5 new replicas have been updated...
# deployment "nginx-k8s" successfully rolled out

# ตลอดเวลา Service ยังใช้งานได้! (Refresh Browser)
```

#### ขั้นตอน B6: ทดลอง Rollback

```bash
# ถ้า Version ใหม่มีปัญหา - Rollback ง่ายมาก!
kubectl rollout undo deployment/nginx-k8s

# ตรวจสอบ History
kubectl rollout history deployment/nginx-k8s

# Rollback ไปยัง Version ที่ต้องการ
kubectl rollout undo deployment/nginx-k8s --to-revision=1
```

#### ขั้นตอน B7: ทดลอง Auto-scaling

```bash
# ติดตั้ง Metrics Server (ถ้ายังไม่มี)
minikube addons enable metrics-server

# สร้าง HPA
kubectl autoscale deployment nginx-k8s \
  --cpu-percent=50 \
  --min=2 \
  --max=10

# ดู HPA
kubectl get hpa

# สร้าง Load Test เพื่อกระตุ้น HPA
kubectl run -it --rm load-test \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://nginx-k8s; done"

# ในอีก Terminal ดู HPA ทำงาน
kubectl get hpa -w
# จะเห็น REPLICAS เพิ่มขึ้นอัตโนมัติ
```

### การเปรียบเทียบสรุป

```
┌────────────────────────────────────────────────────────────┐
│                    สรุปการเปรียบเทียบ                       │
├──────────────────┬─────────────────┬──────────────────────┤
│    Operation     │ Manual Docker   │   Kubernetes         │
├──────────────────┼─────────────────┼──────────────────────┤
│ Deploy App       │ docker run      │ kubectl create       │
│                  │ (ง่าย)          │ deployment           │
├──────────────────┼─────────────────┼──────────────────────┤
│ Scale x5         │ docker run x4   │ kubectl scale        │
│                  │ ต้องจัดการ Port │ --replicas=5         │
│                  │ (ยาก)           │ (ง่ายมาก!)           │
├──────────────────┼─────────────────┼──────────────────────┤
│ Self-healing     │ ❌ ไม่มี        │ ✅ อัตโนมัติ         │
│                  │ ต้อง Restart เอง│                      │
├──────────────────┼─────────────────┼──────────────────────┤
│ Zero-downtime    │ ❌ มี Downtime   │ ✅ Rolling Update    │
│ Update           │                 │ ไม่มี Downtime       │
├──────────────────┼─────────────────┼──────────────────────┤
│ Rollback         │ Manual (ยากมาก) │ kubectl rollout undo │
│                  │                 │ (ง่ายมาก!)           │
├──────────────────┼─────────────────┼──────────────────────┤
│ Auto-scale       │ ❌ ไม่มี        │ ✅ HPA               │
├──────────────────┼─────────────────┼──────────────────────┤
│ Load Balancing   │ ต้องตั้งค่าเอง  │ ✅ อัตโนมัติ         │
└──────────────────┴─────────────────┴──────────────────────┘
```

### ทำความสะอาด

```bash
# ลบ Resources ทั้งหมด
kubectl delete deployment nginx-k8s
kubectl delete service nginx-k8s
kubectl delete hpa nginx-k8s

# หยุด Minikube
minikube stop
```

---

## สรุป

Container Orchestration โดยเฉพาะ Kubernetes ช่วยแก้ปัญหาหลักๆ ของการ Manage Containers:

1. **Self-healing** - Container ล้มก็ Restart อัตโนมัติ
2. **Easy Scaling** - Scale ด้วยคำสั่งเดียว
3. **Zero-downtime Deploy** - Rolling Update ไม่กระทบ User
4. **Auto-scaling** - ขยายตาม Traffic อัตโนมัติ
5. **Service Discovery** - ค้นหา Service ผ่าน DNS

Kubernetes Ecosystem มีเครื่องมือครอบคลุมทุกด้าน ตั้งแต่ Monitoring, Networking, Security จนถึง CI/CD

---

## แบบฝึกหัด

1. ทดลอง Scale Deployment ขึ้นและลง ดูว่า Kubernetes จัดการอย่างไร
2. ทดลองทำ Rolling Update และ Rollback
3. ค้นหาเกี่ยวกับ Helm Chart ที่น่าสนใจ
4. ลองติดตั้ง Prometheus + Grafana ด้วย Helm

## คำถามทบทวน

1. Container Orchestration แก้ปัญหาอะไรบ้าง?
2. ทำไม Kubernetes ถึง Popular กว่า Docker Swarm?
3. CNCF คืออะไร ทำไมถึงสำคัญ?
4. Helm ช่วยอะไรใน Kubernetes Workflow?

---

*ต่อไป: [Part 03: Docker Fundamentals 1](./part-03-docker-fundamentals-1.md)*
