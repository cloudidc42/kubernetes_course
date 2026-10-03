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

---

## ปัญหา "Works on My Machine" แบบ Detailed

### สาเหตุของปัญหา

ปัญหา "Works on My Machine" เกิดจากความแตกต่างระหว่าง Environment ต่างๆ ซึ่งมีหลายระดับ:

```
Level 1: OS Differences
┌────────────────────────────────────────────────────────────────┐
│  Developer Machine        Production Server                     │
│  ─────────────────        ────────────────                      │
│  macOS 14.2               Ubuntu 22.04 LTS                     │
│  arm64 (Apple M2)         x86_64                               │
│  Case-insensitive FS      Case-sensitive FS ← Bug!             │
│  /usr/local/bin/python3   /usr/bin/python3  ← Path ต่างกัน!   │
└────────────────────────────────────────────────────────────────┘

Level 2: Runtime Version Differences
┌────────────────────────────────────────────────────────────────┐
│  Developer Machine        Staging Server    Production Server  │
│  ─────────────────        ─────────────     ────────────────   │
│  Node.js 20.11.0          Node.js 18.19.0   Node.js 16.20.0  │
│  Python 3.12              Python 3.10       Python 3.8        │
│  Go 1.22                  Go 1.21           Go 1.19           │
│  Java 21                  Java 17           Java 11           │
└────────────────────────────────────────────────────────────────┘

Level 3: Dependency Version Conflicts
┌────────────────────────────────────────────────────────────────┐
│  Developer: npm install → ได้ lodash@4.17.21                  │
│  Staging:   npm install → ได้ lodash@4.17.20 (สัปดาห์ก่อน)  │
│  Production: npm install → ได้ lodash@4.17.19 (เดือนก่อน)   │
│                                                                 │
│  หรือ Transitive Dependencies:                                 │
│  App → lib-a@1.0 → dep-x@2.0                                  │
│  App → lib-b@2.0 → dep-x@3.0 ← Version Conflict!             │
└────────────────────────────────────────────────────────────────┘

Level 4: System Library Differences
┌────────────────────────────────────────────────────────────────┐
│  Developer: glibc 2.35 (Ubuntu 22.04)                         │
│  Production: glibc 2.17 (CentOS 7) ← ไม่รองรับ binary ใหม่! │
│                                                                 │
│  OpenSSL Version:                                              │
│  Developer: OpenSSL 3.0                                        │
│  Production: OpenSSL 1.1 ← TLS configuration ต่างกัน!        │
└────────────────────────────────────────────────────────────────┘

Level 5: Configuration Differences
┌────────────────────────────────────────────────────────────────┐
│  Developer .env:              Production env:                   │
│  DATABASE_URL=localhost:5432  DATABASE_URL=prod-db:5432       │
│  DEBUG=true                   DEBUG=false                      │
│  LOG_LEVEL=debug              LOG_LEVEL=error                  │
│  CACHE_TTL=0 ← Disable Cache  CACHE_TTL=3600 ← Cache 1 ชั่วโมง│
└────────────────────────────────────────────────────────────────┘
```

### ผลกระทบที่เกิดขึ้นจริง

```
Real-world Examples:

1. ปัญหา Case Sensitivity (macOS vs Linux)
   ────────────────────────────────────────
   // ใน code:
   import MyComponent from './mycomponent'  ← ใช้งานได้บน macOS
   // ไฟล์จริงชื่อ: MyComponent.tsx
   
   บน Linux: Error - Cannot find module './mycomponent'
   บน macOS: ทำงานปกติ (case-insensitive FS)

2. ปัญหา Line Endings (Windows vs Linux)
   ────────────────────────────────────────
   Shell script ใน Windows: บรรทัดจบด้วย \r\n (CRLF)
   บน Linux: bash: ./run.sh: /bin/bash^M: bad interpreter
   
   ^M คือ \r ที่ Linux ไม่เข้าใจ!

3. ปัญหา Timezone
   ────────────────
   Developer: TZ=Asia/Bangkok (UTC+7)
   Production: TZ=UTC
   
   Code: new Date().toLocaleDateString()
   Dev: "1/15/2024" (Bangkok time)
   Prod: "1/14/2024" (UTC time) ← ผิด 1 วัน!

4. ปัญหา CPU Architecture
   ────────────────────────
   Developer: Apple M2 (arm64)
   Production: AWS EC2 (x86_64)
   
   Binary ที่ Compile บน arm64 รันบน x86_64 ไม่ได้!
   Docker build --platform linux/amd64 ← ต้อง specify!
```

### แนวทางแก้ไขที่ดีที่สุด

```
ระดับความสมบูรณ์ของการแก้ไข:

Level 1 - Baseline (Docker):
┌─────────────────────────────────────────────────────────────┐
│  ใช้ Docker Container                                        │
│  ✓ OS และ Runtime เดียวกัน                                  │
│  ✗ ยังต้องจัดการ Config เอง                                │
│  ✗ ยังต้องจัดการ Dependencies ระหว่าง Services เอง          │
└─────────────────────────────────────────────────────────────┘

Level 2 - Better (Docker Compose):
┌─────────────────────────────────────────────────────────────┐
│  ใช้ Docker Compose                                          │
│  ✓ OS, Runtime, Multi-services เดียวกัน                    │
│  ✓ Config Management ด้วย env files                         │
│  ✗ ยังไม่มี Production-grade HA                             │
│  ✗ ไม่ Auto-scale                                           │
└─────────────────────────────────────────────────────────────┘

Level 3 - Best (Kubernetes):
┌─────────────────────────────────────────────────────────────┐
│  ใช้ Kubernetes                                              │
│  ✓ ทุกอย่างเหมือนกัน Dev → Staging → Production             │
│  ✓ ConfigMaps และ Secrets จัดการ Config                     │
│  ✓ Health Checks, Auto-healing                              │
│  ✓ Auto-scaling                                             │
│  ✓ Rolling Updates Zero-downtime                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Comparison Table ละเอียด: K8s vs Docker Swarm vs Nomad vs OpenShift

### Overview Comparison

| คุณสมบัติ | Kubernetes | Docker Swarm | HashiCorp Nomad | OpenShift |
|-----------|------------|--------------|-----------------|-----------|
| ผู้พัฒนา | Google/CNCF | Docker Inc. | HashiCorp | Red Hat |
| เปิดตัวปี | 2014 | 2015 | 2015 | 2011 |
| License | Apache 2.0 | Apache 2.0 | MPL 2.0 | Apache 2.0 + Enterprise |
| Market Share | ~80% | ~10% | ~5% | ~5% |
| GitHub Stars | 107k+ | 7k+ | 14k+ | 9k+ |
| Complexity | สูง | ต่ำ | ปานกลาง | สูงมาก |
| Learning Curve | ชัน | ราบ | ปานกลาง | ชันมาก |

### Technical Comparison

| คุณสมบัติ | Kubernetes | Docker Swarm | Nomad | OpenShift |
|-----------|------------|--------------|-------|-----------|
| **Scheduling** | | | | |
| Workload Types | Containers | Containers | Containers, VMs, Jobs | Containers |
| Custom Schedulers | ✓ | ✗ | ✓ | ✓ |
| Gang Scheduling | ✓ (DRA) | ✗ | ✓ | ✓ |
| GPU Support | ✓ | ✗ | ✓ | ✓ |
| **Networking** | | | | |
| Built-in DNS | ✓ | ✓ | ✓ | ✓ |
| Network Policies | ✓ | ✗ | ✓ | ✓ |
| Ingress Controller | ✓ | Limited | ✓ | ✓ (HAProxy/F5) |
| Service Mesh | Istio/Linkerd | ✗ | Consul Connect | Istio (built-in) |
| **Storage** | | | | |
| Persistent Volumes | ✓ | ✓ | ✓ | ✓ |
| Storage Classes | ✓ | ✗ | ✓ | ✓ |
| Volume Snapshots | ✓ | ✗ | ✗ | ✓ |
| CSI Support | ✓ | ✗ | ✓ | ✓ |
| **Security** | | | | |
| RBAC | ✓ | ✗ | ✓ | ✓ (Enhanced) |
| Pod Security | ✓ (PSA) | ✗ | ✓ | ✓ (SCC) |
| Secret Encryption | ✓ | ✓ | ✓ | ✓ |
| Image Scanning | ✗ (3rd party) | ✗ | ✗ | ✓ (Built-in) |
| **Operations** | | | | |
| Multi-cluster | ✓ | ✗ | ✓ | ✓ |
| Multi-cloud | ✓ | Limited | ✓ | ✓ |
| Auto-scaling (HPA) | ✓ | ✗ | ✓ | ✓ |
| Auto-scaling (VPA) | ✓ | ✗ | ✗ | ✓ |
| Cluster Autoscaler | ✓ | ✗ | ✓ | ✓ |
| **Developer Experience** | | | | |
| Web UI | Dashboard (Basic) | Built-in | Built-in | Console (Excellent) |
| CLI | kubectl | docker stack | nomad | oc |
| Helm Support | ✓ | ✓ | ✗ | ✓ |
| CI/CD Integration | ✓ (Argo, Tekton) | ✓ | ✓ | ✓ (Tekton built-in) |
| Local Dev | minikube/kind | docker compose | ✓ | CRC |

### Use Case Comparison

```
เลือก Kubernetes เมื่อ:
├── ต้องการ Feature ครบที่สุด
├── Team ขนาดกลาง-ใหญ่ (20+ คน)
├── Microservices Architecture
├── ต้องการ Auto-scaling ซับซ้อน
├── ต้องการ Multi-cluster Management
└── ต้องการ Community ขนาดใหญ่

เลือก Docker Swarm เมื่อ:
├── ต้องการเริ่มต้นง่ายที่สุด
├── Team เล็ก (< 10 คน)
├── Monolith หรือ Small Microservices
├── ไม่ต้องการ Complexity ของ K8s
└── ใช้ Docker อยู่แล้ว

เลือก HashiCorp Nomad เมื่อ:
├── ต้องการรัน Non-container Workloads ด้วย
├── ใช้ HashiCorp Stack (Vault, Consul)
├── ต้องการ Simplicity กว่า K8s แต่ Powerful กว่า Swarm
├── On-premise หรือ Bare Metal
└── Mixed Workloads (Windows + Linux)

เลือก OpenShift เมื่อ:
├── Enterprise ต้องการ Support จาก Red Hat
├── ต้องการ Security ระดับสูง (DoD, HIPAA)
├── ต้องการ Developer Experience ที่ดี
├── ใช้ Red Hat/RHEL อยู่แล้ว
└── ต้องการ Built-in CI/CD (Tekton)
```

### Resource Requirements Comparison

```
Minimum Resources สำหรับ Production Cluster:

Kubernetes:
├── Control Plane: 3 Nodes × (4 CPU + 8GB RAM)
├── Worker Nodes: 3+ Nodes × (2 CPU + 4GB RAM)
└── Total Minimum: ~28 CPU + 44GB RAM

Docker Swarm:
├── Manager: 3 Nodes × (2 CPU + 4GB RAM)
├── Worker Nodes: 3+ Nodes × (2 CPU + 2GB RAM)
└── Total Minimum: ~12 CPU + 18GB RAM

Nomad:
├── Server: 3 Nodes × (2 CPU + 4GB RAM)
├── Client Nodes: 3+ Nodes × (2 CPU + 4GB RAM)
└── Total Minimum: ~12 CPU + 24GB RAM

OpenShift:
├── Control Plane: 3 Nodes × (4 CPU + 16GB RAM)
├── Infra Nodes: 3 Nodes × (4 CPU + 16GB RAM)
├── Worker Nodes: 3+ Nodes × (4 CPU + 16GB RAM)
└── Total Minimum: ~36 CPU + 144GB RAM (หนักที่สุด!)
```

---

## Container Orchestration Use Case Matrix

### Matrix: ประเภท Application vs Platform

| ประเภท Application | Docker Swarm | Kubernetes | Nomad | OpenShift | คำแนะนำ |
|-------------------|--------------|------------|-------|-----------|----------|
| Static Website | ✓✓✓ | ✓✓ | ✓✓ | ✓✓ | Swarm (ง่ายสุด) |
| REST API Service | ✓✓ | ✓✓✓ | ✓✓ | ✓✓✓ | K8s (Feature ครบ) |
| Microservices (10+) | ✓ | ✓✓✓ | ✓✓ | ✓✓✓ | K8s หรือ OpenShift |
| ML Training Jobs | ✗ | ✓✓✓ | ✓✓ | ✓✓ | K8s (GPU Support) |
| Batch Processing | ✓ | ✓✓✓ | ✓✓✓ | ✓✓ | K8s หรือ Nomad |
| Database (StatefulSet) | ✓ | ✓✓✓ | ✓✓ | ✓✓✓ | K8s |
| Legacy Java App | ✗ | ✓✓ | ✓✓ | ✓✓✓ | OpenShift |
| Windows Containers | ✗ | ✓✓ | ✓✓✓ | ✓✓ | Nomad หรือ K8s |
| Edge Computing | ✗ | ✓✓ (k3s) | ✓✓✓ | ✓ (MicroShift) | Nomad หรือ k3s |
| Multi-cloud | ✗ | ✓✓✓ | ✓✓✓ | ✓✓ | K8s หรือ Nomad |

### Matrix: Company Size vs Platform

```
Company Size Matrix:

┌────────────────────────────────────────────────────────────────┐
│           │ Small (< 20) │ Medium (20-100) │ Large (100+)      │
├───────────┼──────────────┼─────────────────┼───────────────────┤
│ Startup   │ Docker Swarm │ Kubernetes      │ Kubernetes        │
│           │ หรือ Compose │ หรือ Swarm      │                   │
├───────────┼──────────────┼─────────────────┼───────────────────┤
│ Scaleup   │ Kubernetes   │ Kubernetes      │ Kubernetes        │
│           │ (ลงทุนเลย)  │                 │ หรือ OpenShift    │
├───────────┼──────────────┼─────────────────┼───────────────────┤
│ Enterprise│ OpenShift    │ OpenShift        │ OpenShift         │
│ (Regulated)│ หรือ K8s   │ หรือ K8s        │ หรือ Multi-K8s    │
├───────────┼──────────────┼─────────────────┼───────────────────┤
│ Tech-heavy│ Kubernetes   │ Kubernetes +    │ Custom K8s Stack  │
│           │              │ Nomad          │ + Service Mesh    │
└────────────────────────────────────────────────────────────────┘
```

### Use Case: E-commerce Platform

```
ตัวอย่าง E-commerce บน Kubernetes:

┌─────────────────────────────────────────────────────────────────┐
│                    E-commerce on Kubernetes                      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Ingress (nginx)                        │  │
│  └──────────┬──────────────┬──────────────┬─────────────────┘  │
│             │              │              │                       │
│         ┌──────┐      ┌──────┐      ┌──────┐                  │
│         │Web   │      │API   │      │CDN   │                  │
│         │Frontend    │Gateway      │Edge  │                  │
│         │(3 pods)    │(5 pods)     │(2 pods)│                │
│         └──────┘      └──┬───┘      └──────┘                  │
│                          │                                       │
│              ┌───────────┼───────────┐                          │
│              │           │           │                           │
│          ┌──────┐   ┌──────┐   ┌──────┐                       │
│          │Product│  │Order │  │User  │                       │
│          │Service│  │Service│  │Service│                      │
│          │(3 pods)│ │(5 pods)│ │(3 pods)│                    │
│          └──────┘   └──┬───┘   └──────┘                       │
│                        │                                         │
│              ┌─────────┴─────────┐                              │
│              │                   │                               │
│          ┌──────┐          ┌──────────┐                        │
│          │Payment│          │Notification│                     │
│          │Service│          │Service    │                      │
│          │(3 pods)│         │(2 pods)   │                     │
│          └──────┘          └──────────┘                        │
│                                                                   │
│  Databases: MySQL, Redis, MongoDB (StatefulSets)                │
│  Message Queue: Kafka, RabbitMQ                                  │
│  Monitoring: Prometheus + Grafana                                │
└─────────────────────────────────────────────────────────────────┘

HPA Settings:
- Web Frontend: scale 3-20 pods, CPU > 70%
- API Gateway: scale 5-50 pods, CPU > 60%
- Order Service: scale 5-30 pods, RPS > 1000
- Payment Service: scale 3-15 pods, CPU > 50%
```

---

## Workshop: เปรียบเทียบ Deploy แบบ Manual vs Orchestrated

### Scenario: Deploy 3-tier Application

เราจะ Deploy Application ที่ประกอบด้วย:
- Frontend: React App
- Backend: Node.js API
- Database: PostgreSQL

### วิธีที่ 1: Manual Deployment (แบบเก่า)

```bash
# ===== Manual Deployment (ปวดหัวมาก!) =====

# 1. SSH เข้า Server
ssh user@server1

# 2. Install Dependencies บน Server
sudo apt-get update
sudo apt-get install -y nodejs npm postgresql

# 3. Setup Database
sudo -u postgres psql
CREATE DATABASE myapp;
CREATE USER myuser WITH PASSWORD 'secret123';
GRANT ALL ON DATABASE myapp TO myuser;
\q

# 4. Deploy Backend
cd /var/www
git clone https://github.com/myorg/backend.git
cd backend
npm install
echo "DATABASE_URL=postgres://myuser:secret123@localhost:5432/myapp" > .env
echo "PORT=3000" >> .env
pm2 start app.js --name backend

# 5. Deploy Frontend
cd /var/www
git clone https://github.com/myorg/frontend.git
cd frontend
npm install
npm run build
sudo cp -r build/* /var/www/html/

# 6. Configure Nginx (Reverse Proxy)
sudo cat > /etc/nginx/sites-available/myapp << 'NGINX'
server {
    listen 80;
    server_name myapp.com;
    
    location / {
        root /var/www/html;
    }
    
    location /api {
        proxy_pass http://localhost:3000;
    }
}
NGINX
sudo nginx -t && sudo nginx -s reload

# ===== ปัญหาของ Manual Deployment: =====
# ❌ Server 2 ต้องทำซ้ำทั้งหมด
# ❌ ถ้า Server ล้มต้องทำเองทุกอย่าง
# ❌ Scale ต้องทำเอง + Config Load Balancer เอง
# ❌ Update ต้อง SSH เข้าทุก Server
# ❌ Rollback ยุ่งยากมาก
```

### วิธีที่ 2: Docker Compose (ดีขึ้น)

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    image: myapp-frontend:latest
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
    - "80:80"
    depends_on:
    - backend
    environment:
    - REACT_APP_API_URL=http://backend:3000

  backend:
    image: myapp-backend:latest
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
    - "3000:3000"
    depends_on:
    - postgres
    environment:
    - DATABASE_URL=postgres://myuser:secret123@postgres:5432/myapp
      
  postgres:
    image: postgres:15-alpine
    volumes:
    - postgres-data:/var/lib/postgresql/data
    environment:
    - POSTGRES_DB=myapp
    - POSTGRES_USER=myuser
    - POSTGRES_PASSWORD=secret123

volumes:
  postgres-data:
```

```bash
# Deploy ด้วย Docker Compose
docker compose up -d

# ปัญหาของ Docker Compose ใน Production:
# ❌ รันบน Machine เดียว (Single Point of Failure)
# ❌ ไม่มี Auto-healing
# ❌ Scale ต้องทำเอง
# ❌ Load Balancing ไม่มี Built-in ที่ดี
```

### วิธีที่ 3: Kubernetes (Production-ready)

```bash
# สร้าง Namespace
kubectl create namespace myapp

# สร้าง Secret สำหรับ Database
kubectl create secret generic db-secret \
  --from-literal=POSTGRES_DB=myapp \
  --from-literal=POSTGRES_USER=myuser \
  --from-literal=POSTGRES_PASSWORD=secret123 \
  -n myapp
```

```yaml
# k8s/postgres.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        envFrom:
        - secretRef:
            name: db-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: myapp
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None  # Headless Service สำหรับ StatefulSet
```

```yaml
# k8s/backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myapp-backend:v1.0
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          value: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres:5432/$(POSTGRES_DB)"
        envFrom:
        - secretRef:
            name: db-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myapp
spec:
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```yaml
# k8s/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
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
      containers:
      - name: frontend
        image: myapp-frontend:v1.0
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: myapp
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 3000
```

```bash
# Deploy ทั้งหมด
kubectl apply -f k8s/ -n myapp

# ดู Status
kubectl get all -n myapp

# ===== ประโยชน์ของ Kubernetes: =====
# ✓ Self-healing: Pod ล้มจะถูกสร้างใหม่อัตโนมัติ
# ✓ Auto-scaling: HPA scale pods ตาม CPU usage
# ✓ Rolling Update: Zero-downtime deployment
# ✓ Rollback: kubectl rollout undo ใน 1 คำสั่ง
# ✓ Load Balancing: Kubernetes Service กระจาย Traffic
# ✓ Config Management: ConfigMaps + Secrets
# ✓ Resource Limits: ควบคุม CPU/Memory ของแต่ละ Pod
```

### เปรียบเทียบ Deployment Time

```
Deployment Time Comparison:

Task                      Manual    Docker Compose    Kubernetes
──────────────────────    ──────    ──────────────    ──────────
Initial Setup             4-8h      30min             2-4h (ครั้งแรก)
Deploy New Version        30-60min  5min              2-5min
Rollback                  1-2h      10min             30sec
Scale x3                  1-2h      5min              30sec
Recovery from Failure     30-60min  Manual            Automatic
Add new Service           4-8h      30min             30min
```

---

## แบบฝึกหัดพร้อมเฉลย

### ข้อที่ 1: เปรียบเทียบ Container Orchestration

**โจทย์**: ระบุว่าองค์กรต่อไปนี้ควรใช้ Platform ใด พร้อมอธิบายเหตุผล

1. Startup ขายของออนไลน์ ทีม 5 คน ไม่มี DevOps Expert
2. ธนาคารใหญ่ ต้องการ Compliance สูง มี Red Hat Support

**เฉลย**:

```
1. Startup ทีม 5 คน:
   คำตอบ: Docker Swarm หรือ Docker Compose
   
   เหตุผล:
   - ทีมเล็ก ไม่มี Resource ในการเรียน K8s
   - ต้องการ Deploy เร็ว
   - Workload ยังไม่ซับซ้อน
   - สามารถ Migrate ไป K8s ทีหลังได้เมื่อโตขึ้น

2. ธนาคารใหญ่:
   คำตอบ: Red Hat OpenShift
   
   เหตุผล:
   - Enterprise Support จาก Red Hat
   - Security Compliance (PCI-DSS, SOC2)
   - RBAC ที่แข็งแกร่ง
   - Audit Logging ครบถ้วน
   - SCC (Security Context Constraints) เข้มกว่า K8s
```

### ข้อที่ 2: ออกแบบ Architecture

**โจทย์**: ออกแบบ Kubernetes Architecture สำหรับ E-commerce ที่มี:
- Frontend: React (Stateless)
- Backend: Node.js API (Stateless)
- Database: PostgreSQL (Stateful)
- Cache: Redis (Stateful)
- Queue: RabbitMQ (Stateful)

**เฉลย**:

```yaml
# Workload Types ที่ควรใช้:

# 1. Frontend - Deployment (Stateless)
# เหตุผล: ไม่มี State, Scale ได้อิสระ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  # HPA สำหรับ Auto-scaling

# 2. Backend API - Deployment (Stateless)
# เหตุผล: ไม่มี State, Scale ได้อิสระ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 5
  # HPA สำหรับ Auto-scaling

# 3. PostgreSQL - StatefulSet
# เหตุผล: มี State (data), ต้องการ Stable Network Identity
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 1  # หรือใช้ Patroni สำหรับ HA

# 4. Redis - StatefulSet
# เหตุผล: มี State (cache data), Stable Identity สำหรับ Cluster
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  replicas: 3  # Redis Cluster Mode

# 5. RabbitMQ - StatefulSet
# เหตุผล: มี Message Queue State, ต้องการ Stable Network Identity
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  replicas: 3  # RabbitMQ Cluster
```

### ข้อที่ 3: Kubernetes vs Docker Swarm Decision

**โจทย์**: Team กำลัง Debate ระหว่าง K8s กับ Swarm สำหรับ New Project:
- Application: 15 Microservices
- Team: 15 Engineers, 2 DevOps
- Traffic: 10,000 requests/minute peak
- Requirement: Auto-scaling, Zero-downtime deploy

**เฉลย**:

```
คำตอบ: Kubernetes

เหตุผล:
1. 15 Microservices → K8s จัดการได้ดีกว่า Swarm มาก
   - Namespace isolation ระหว่าง Services
   - Network Policies ระหว่าง Services
   
2. Auto-scaling requirement:
   - K8s: HPA (CPU, Memory, Custom Metrics) + VPA + KEDA
   - Swarm: ไม่มี Auto-scaling built-in
   
3. Zero-downtime deploy:
   - K8s: Built-in Rolling Update + Readiness Probes
   - Swarm: Limited Rolling Update
   
4. 15 Engineers, 2 DevOps:
   - K8s Learning Curve สูงกว่า แต่คุ้มค่า
   - ทีมขนาดนี้รองรับการเรียน K8s ได้
   
5. 10,000 req/min peak:
   - HPA จะ Scale Pod ขึ้นรับ Traffic
   - Swarm ต้องทำ Manual
   
สรุป: ลงทุนเวลาเรียน K8s 1-2 เดือน
คุ้มค่ากับ Feature และ Scalability ที่ได้
```

### ข้อที่ 4: แก้ไข Deployment ที่มีปัญหา

**โจทย์**: Deployment นี้มีปัญหาอะไรบ้าง?

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
```

**เฉลย**:

```yaml
# ปัญหาและการแก้ไข:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:           # ← เพิ่ม Labels
    app: my-app
spec:
  replicas: 3       # ← ปัญหา 1: replicas=1 ไม่มี HA
  selector:         # ← ปัญหา 2: ขาด selector
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:       # ← ปัญหา 3: ขาด Labels บน Pod
        app: my-app
    spec:
      containers:
      - name: app
        image: myapp:1.0.0  # ← ปัญหา 4: latest เปลี่ยนแปลงได้ ควรระบุ version
        ports:
        - containerPort: 8080
        resources:          # ← ปัญหา 5: ไม่มี Resource Limits
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:      # ← ปัญหา 6: ไม่มี Health Checks
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

# สรุปปัญหา:
# 1. replicas=1 → Single Point of Failure
# 2. ไม่มี selector → Deployment ไม่รู้ว่าจะ Manage Pods ไหน
# 3. ไม่มี Labels บน Pod Template → Service ไม่พบ Pods
# 4. image:latest → Deployment ไม่รู้ว่าต้อง Update หรือเปล่า
# 5. ไม่มี Resource Limits → Pod อาจกิน Resource ไม่จำกัด
# 6. ไม่มี Health Checks → Traffic ส่งไป Pod ที่ยังไม่พร้อม
```

---

## สรุป Part 02

ใน Part นี้เราได้เรียนรู้:

1. **ปัญหา "Works on My Machine"** - สาเหตุจาก OS, Runtime, Dependencies, Config ที่แตกต่างกัน
2. **Container Orchestration แก้ปัญหา** - ทำให้ Environment เหมือนกันทุก Stage
3. **K8s vs Swarm vs Nomad vs OpenShift** - แต่ละ Platform เหมาะกับ Use Case ต่างกัน
4. **Use Case Matrix** - เลือก Platform ตาม App Type และ Company Size
5. **Workshop** - Deploy App แบบ Manual, Compose, และ Kubernetes เปรียบเทียบ

### Checklist ก่อนไปต่อ

- [ ] เข้าใจว่า Container Orchestration แก้ปัญหาอะไร
- [ ] สามารถเปรียบเทียบ K8s กับ Swarm และ Nomad ได้
- [ ] เข้าใจเมื่อไรควรใช้ Platform ใด
- [ ] Deploy 3-tier App บน Kubernetes ได้
- [ ] ทำแบบฝึกหัดครบ 4 ข้อ

---

*ต่อไป: [Part 03: Docker Fundamentals 1](./part-03-docker-fundamentals-1.md)*
