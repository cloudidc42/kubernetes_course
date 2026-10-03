# Part 25: Vertical Pod Autoscaler (VPA) - ปรับ Resources อัตโนมัติ

## สารบัญ
1. [VPA คืออะไร](#vpa-คืออะไร)
2. [VPA Architecture](#vpa-architecture)
3. [ติดตั้ง VPA](#ติดตั้ง-vpa)
4. [VPA YAML ละเอียด](#vpa-yaml-ละเอียด)
5. [VPA Modes](#vpa-modes)
6. [VPA vs HPA](#vpa-vs-hpa)
7. [Workshop: ใช้ VPA Optimize Resources](#workshop-ใช้-vpa-optimize-resources)
8. [Workshop: VPA ใน Off Mode (Recommendation Only)](#workshop-vpa-ใน-off-mode)
9. [Workshop: VPA ร่วมกับ HPA](#workshop-vpa-ร่วมกับ-hpa)
10. [Troubleshooting VPA](#troubleshooting-vpa)
11. [Best Practices](#best-practices)

---

## VPA คืออะไร

**Vertical Pod Autoscaler (VPA)** คือ Kubernetes component ที่ปรับ **CPU และ Memory requests/limits** ของ containers ใน Pod อัตโนมัติ:

- วิเคราะห์การใช้งาน resource จริง
- แนะนำค่า requests/limits ที่เหมาะสม
- สามารถปรับค่าโดยอัตโนมัติ (ต้อง restart Pod)

### HPA vs VPA

```
HPA (Horizontal): เพิ่ม/ลดจำนวน Pod
─────────────────────────────────────
  Load ↑  →  [ Pod ] [ Pod ] [ Pod ] [ Pod ]  (เพิ่ม Pod)
  Load ↓  →  [ Pod ] [ Pod ]                  (ลด Pod)

VPA (Vertical): เพิ่ม/ลด resources ของ Pod แต่ละตัว
───────────────────────────────────────────────────────
  Load ↑  →  [ Pod: CPU 200m→500m, Mem 128Mi→512Mi ]  (ขยาย resources)
  Load ↓  →  [ Pod: CPU 500m→100m, Mem 512Mi→128Mi ]  (ลด resources)
```

### Use Cases ของ VPA

```
เหมาะกับ:
✓ Stateful applications ที่ scale horizontal ยาก
✓ Applications ที่ resource usage เปลี่ยนแปลงตามเวลา
✓ ค้นหา initial resource values ที่เหมาะสม
✓ ลด waste จาก over-provisioning
✓ ป้องกัน OOMKill จาก under-provisioning

ไม่เหมาะกับ:
✗ Applications ที่ต้องการ continuous availability (VPA ต้อง restart Pod)
✗ ใช้ร่วมกับ HPA บน CPU/Memory metrics เดียวกัน
```

---

## VPA Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       VPA Components                           │
│                                                                 │
│  ┌─────────────────┐                                           │
│  │  VPA Recommender│ ← ดู metrics จาก Metrics Server          │
│  │                 │   คำนวณ recommended resources             │
│  └────────┬────────┘                                           │
│           │ recommendations                                     │
│  ┌────────▼────────┐                                           │
│  │   VPA Object    │ ← ผู้ใช้สร้าง VPA object                  │
│  │  (TargetRef,    │   กำหนด mode, min/max                      │
│  │   Mode, etc.)   │                                           │
│  └────────┬────────┘                                           │
│           │ (ถ้า mode != Off)                                   │
│  ┌────────▼────────┐                                           │
│  │  VPA Updater   │ → ลบ Pod ที่ต้อง resize                    │
│  └────────────────┘   Pod ใหม่จะมี resources ที่ถูกปรับ        │
│           │                                                     │
│  ┌────────▼────────┐                                           │
│  │ VPA Admission   │ → Mutate Pod spec เมื่อ Pod ถูกสร้าง      │
│  │  Webhook        │   ใส่ recommended resource requests        │
│  └────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## ติดตั้ง VPA

### วิธีที่ 1: ติดตั้งจาก Source

```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# ติดตั้ง VPA
./hack/vpa-install.sh

# ตรวจสอบการติดตั้ง
kubectl get pods -n kube-system | grep vpa
```

### วิธีที่ 2: ติดตั้งด้วย Helm

```bash
# เพิ่ม helm repo
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm repo update

# ติดตั้ง VPA
helm install vpa fairwinds-stable/vpa \
  --namespace kube-system \
  --set admissionController.enabled=true \
  --set recommender.enabled=true \
  --set updater.enabled=true

# ตรวจสอบ
kubectl get pods -n kube-system -l app=vpa
```

### วิธีที่ 3: ติดตั้ง Component ด้วยตนเอง

```bash
# สร้าง CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml

# สร้าง RBAC
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/vpa-rbac.yaml

# Deploy VPA components
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/recommender-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/updater-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/admission-controller-deployment.yaml

# ตรวจสอบ
kubectl get pods -n kube-system | grep vpa
kubectl get crds | grep verticalpodautoscaler
```

### ตรวจสอบการติดตั้ง

```bash
# ดู VPA CRDs
kubectl api-resources | grep autoscaling.k8s.io

# ดู VPA components
kubectl get deploy -n kube-system | grep vpa
# vpa-admission-controller
# vpa-recommender
# vpa-updater

# ดู status
kubectl get pods -n kube-system -l app=vpa-admission-controller
kubectl get pods -n kube-system -l app=vpa-recommender
kubectl get pods -n kube-system -l app=vpa-updater
```

---

## VPA YAML ละเอียด

### Basic VPA

```yaml
# basic-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: production
spec:
  # Target workload
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  
  # Update Policy
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
  
  # Resource Policy (optional)
  resourcePolicy:
    containerPolicies:
    - containerName: "*"    # ทุก container
      minAllowed:
        cpu: "50m"          # minimum
        memory: "50Mi"
      maxAllowed:
        cpu: "4"            # maximum
        memory: "4Gi"
      controlledResources:  # ควบคุม resource อะไรบ้าง
      - cpu
      - memory
```

### VPA แบบสมบูรณ์

```yaml
# full-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: production
  labels:
    app: web-app
    team: platform
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  updatePolicy:
    updateMode: "Auto"
    
    # Minimum replicas ที่ต้องรันขณะ update (ต้องการ VPA >= 0.8)
    minReplicas: 2
  
  resourcePolicy:
    containerPolicies:
    # Container: web
    - containerName: web
      mode: Auto           # Auto หรือ Off per container
      minAllowed:
        cpu: "100m"
        memory: "64Mi"
      maxAllowed:
        cpu: "2000m"       # ไม่เกิน 2 CPU
        memory: "2Gi"
      controlledResources:
      - cpu
      - memory
      controlledValues: RequestsAndLimits  # RequestsOnly หรือ RequestsAndLimits
    
    # Container: sidecar (ไม่ต้องการ VPA)
    - containerName: sidecar
      mode: "Off"          # ไม่ปรับ container นี้
```

---

## VPA Modes

### Mode: Off (Recommendation Only)

```yaml
updatePolicy:
  updateMode: "Off"
```

- VPA **คำนวณ recommendation เท่านั้น**
- ไม่ปรับ Pod จริง
- ผู้ใช้อ่าน recommendation แล้วปรับ manual

```bash
# ดู VPA recommendations
kubectl describe vpa my-app-vpa -n production

# หรือ
kubectl get vpa my-app-vpa -n production -o yaml | grep -A 30 recommendation
```

### Mode: Initial

```yaml
updatePolicy:
  updateMode: "Initial"
```

- VPA ปรับ resources เฉพาะเมื่อ **Pod ใหม่ถูกสร้าง** (ผ่าน admission webhook)
- ไม่ restart Pod ที่รันอยู่แล้ว
- เหมาะสำหรับ: initial resource calibration

### Mode: Recreate

```yaml
updatePolicy:
  updateMode: "Recreate"
```

- VPA ปรับ resources และ **restart Pod เมื่อจำเป็น**
- Pod ถูกลบและสร้างใหม่เมื่อ recommended resources เปลี่ยนมาก
- ไม่ทำ live update (ต้อง evict Pod)

### Mode: Auto (Default)

```yaml
updatePolicy:
  updateMode: "Auto"
```

- VPA เลือกใช้ Recreate (ปัจจุบัน) และอาจใช้ in-place update ในอนาคต
- เหมือน Recreate ในปัจจุบัน

### ตารางเปรียบเทียบ Modes

| Mode | สร้าง recommendation | ปรับ new Pods | Restart running Pods |
|------|---------------------|---------------|---------------------|
| Off | ✓ | ✗ | ✗ |
| Initial | ✓ | ✓ | ✗ |
| Recreate | ✓ | ✓ | ✓ (evict) |
| Auto | ✓ | ✓ | ✓ (evict) |

---

## VPA vs HPA

### เมื่อไรใช้ VPA

```
ใช้ VPA เมื่อ:
✓ Scale horizontal ยาก (เช่น single-threaded app)
✓ ต้องการหา resource values ที่เหมาะสม
✓ App ที่ memory requirement เปลี่ยนตามเวลา
✓ Database (StatefulSet) ที่ต้องการ dedicated resources
✓ ต้องการลด resource waste (over-provisioned apps)

ใช้ HPA เมื่อ:
✓ Stateless web applications
✓ Traffic spike ที่ต้องการ scale เร็ว
✓ Apps ที่ scale horizontal ได้ดี
✓ ต้องการ zero-downtime scaling

ใช้ทั้ง VPA + HPA เมื่อ:
✓ VPA: ควบคุม memory (ไม่ scale ด้วย memory)
✓ HPA: scale ด้วย CPU หรือ custom metrics
```

### ข้อห้าม: ไม่ควรใช้ VPA + HPA กับ metric เดียวกัน

```yaml
# ห้ามทำ! VPA + HPA ด้วย CPU
# VPA ปรับ CPU requests → HPA คำนวณ utilization ผิดพลาด
# เพราะ HPA utilization = actual/request และ VPA เปลี่ยน request ตลอด

# ทำแบบนี้แทน:
# VPA: ควบคุม memory
# HPA: scale ด้วย CPU หรือ custom metrics
```

### ตัวอย่าง VPA + HPA ที่ถูกต้อง

```yaml
# HPA: scale based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
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
      name: cpu              # HPA ใช้ CPU
      target:
        type: Utilization
        averageUtilization: 60

---
# VPA: ควบคุมเฉพาะ memory
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources:
      - memory             # VPA ควบคุม memory เท่านั้น
      # ไม่รวม cpu!
```

---

## Workshop: ใช้ VPA Optimize Resources

เป้าหมาย: Deploy application ที่ over-provisioned และให้ VPA ปรับค่าให้เหมาะสม

### 1. สร้าง Namespace

```bash
kubectl create namespace vpa-demo
```

### 2. Deploy Application ที่ Over-provisioned

```yaml
# over-provisioned-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: vpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.21
        
        # Over-provisioned! ใช้ resources น้อยกว่าที่ขอมาก
        resources:
          requests:
            cpu: "1000m"     # ใช้จริง ~10m
            memory: "1Gi"    # ใช้จริง ~20Mi
          limits:
            cpu: "2000m"
            memory: "2Gi"
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### 3. สร้าง VPA ใน Off Mode ก่อน

```yaml
# vpa-recommendation.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: vpa-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Off"    # แค่ดู recommendation ก่อน
  resourcePolicy:
    containerPolicies:
    - containerName: web
      minAllowed:
        cpu: "10m"
        memory: "10Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
```

### 4. Deploy และดู Recommendations

```bash
# Apply
kubectl apply -f over-provisioned-app.yaml
kubectl apply -f vpa-recommendation.yaml

# รอ VPA เก็บ metrics (อย่างน้อย 5-10 นาที)
kubectl get vpa -n vpa-demo

# ดู recommendations
kubectl describe vpa web-app-vpa -n vpa-demo

# หรือดูแบบ JSON
kubectl get vpa web-app-vpa -n vpa-demo -o json | \
  python3 -m json.tool | grep -A 30 recommendation
```

Output ตัวอย่าง:
```yaml
Status:
  Conditions:
  - Last Transition Time:  2024-01-15T10:00:00Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
    - Container Name:  web
      Lower Bound:
        Cpu:     11m        # ต่ำสุดที่แนะนำ
        Memory:  262144k
      Target:
        Cpu:     25m        # ค่าที่แนะนำ
        Memory:  52428k
      Uncapped Target:
        Cpu:     25m
        Memory:  52428k
      Upper Bound:
        Cpu:     101m       # สูงสุดที่แนะนำ
        Memory:  1Gi
```

### 5. เปลี่ยนเป็น Auto Mode

```yaml
# vpa-auto.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: vpa-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 1
  resourcePolicy:
    containerPolicies:
    - containerName: web
      minAllowed:
        cpu: "10m"
        memory: "10Mi"
      maxAllowed:
        cpu: "500m"
        memory: "512Mi"
```

```bash
# อัปเดต VPA เป็น Auto mode
kubectl apply -f vpa-auto.yaml

# ดู VPA status
kubectl get vpa -n vpa-demo

# ดู Pod ที่ถูก evict และ recreate
kubectl get pods -n vpa-demo -w

# ดู resources ของ Pod ใหม่
kubectl get pods -n vpa-demo -o yaml | grep -A 10 resources
```

---

## Workshop: VPA ใน Off Mode

เป้าหมาย: ใช้ VPA เป็นเครื่องมือ recommendation สำหรับ right-sizing

### 1. Setup Script สำหรับอ่าน VPA Recommendations

```bash
#!/bin/bash
# vpa-report.sh: สร้าง report ของ VPA recommendations

NAMESPACE=${1:-default}

echo "=== VPA Recommendations for namespace: $NAMESPACE ==="
echo ""

# loop through all VPAs
for vpa in $(kubectl get vpa -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  echo "VPA: $vpa"
  echo "---"
  
  # ดู recommendations
  kubectl get vpa $vpa -n $NAMESPACE -o jsonpath='{.status.recommendation.containerRecommendations[*]}' | \
    python3 -c "
import json, sys

data = sys.stdin.read()
if not data:
    print('  No recommendations yet')
    exit()

# Handle multiple containers
recs = json.loads('[' + data.replace('}{', '},{') + ']')
for rec in recs:
    name = rec.get('containerName', 'unknown')
    target = rec.get('target', {})
    lower = rec.get('lowerBound', {})
    upper = rec.get('upperBound', {})
    
    print(f'  Container: {name}')
    print(f'    Target CPU:    {target.get(\"cpu\", \"N/A\")}')
    print(f'    Target Memory: {target.get(\"memory\", \"N/A\")}')
    print(f'    Min CPU:       {lower.get(\"cpu\", \"N/A\")}')
    print(f'    Max CPU:       {upper.get(\"cpu\", \"N/A\")}')
    print()
"
  echo ""
done
```

```bash
chmod +x vpa-report.sh
./vpa-report.sh production
```

### 2. VPA สำหรับ Multiple Deployments

```yaml
# vpa-all-apps.yaml
# VPA สำหรับ web-frontend
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-frontend-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  updatePolicy:
    updateMode: "Off"
---
# VPA สำหรับ api-server
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-server-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Off"
---
# VPA สำหรับ worker
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: worker-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  updatePolicy:
    updateMode: "Off"
```

```bash
kubectl apply -f vpa-all-apps.yaml

# รอ 30 นาที - 1 ชั่วโมง เพื่อให้ VPA เก็บ data เพียงพอ

# ดู recommendations ทั้งหมด
kubectl get vpa -n production -o custom-columns=\
'NAME:.metadata.name,MODE:.spec.updatePolicy.updateMode,LAST_UPDATED:.status.conditions[0].lastTransitionTime'
```

---

## Workshop: VPA ร่วมกับ HPA

### Deploy Application

```yaml
# app-with-hpa-vpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scalable-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: scalable-app
  template:
    metadata:
      labels:
        app: scalable-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
---
# HPA: scale based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: scalable-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: scalable-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
---
# VPA: optimize memory only
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: scalable-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: scalable-app
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 2
  resourcePolicy:
    containerPolicies:
    - containerName: app
      controlledResources:
      - memory         # VPA ควบคุมเฉพาะ memory
      minAllowed:
        memory: "64Mi"
      maxAllowed:
        memory: "1Gi"
```

```bash
kubectl apply -f app-with-hpa-vpa.yaml

# Monitor ทั้ง HPA และ VPA
watch "kubectl get hpa,vpa -n production; echo '---'; kubectl top pods -n production"
```

---

## Troubleshooting VPA

### ปัญหา: VPA ไม่แสดง Recommendations

```bash
# ตรวจสอบว่า VPA components รันอยู่
kubectl get pods -n kube-system | grep vpa

# ดู recommender logs
kubectl logs -n kube-system deployment/vpa-recommender --tail=50

# ตรวจสอบว่ามี metrics พอ (ต้องรอ 5-10 นาที)
kubectl describe vpa my-app-vpa -n production

# ตรวจสอบว่า Metrics Server ทำงาน
kubectl top pods -n production
```

### ปัญหา: VPA ไม่ Evict Pods

```bash
# ตรวจสอบ updater logs
kubectl logs -n kube-system deployment/vpa-updater --tail=50

# ตรวจสอบว่า PodDisruptionBudget ไม่บล็อก
kubectl get pdb -n production

# ตรวจสอบว่า minReplicas ไม่มาก
kubectl get vpa my-app-vpa -n production -o yaml | grep minReplicas
```

### ปัญหา: Admission Webhook Error

```bash
# ตรวจสอบ admission controller
kubectl get pods -n kube-system -l app=vpa-admission-controller

# ดู logs
kubectl logs -n kube-system deployment/vpa-admission-controller

# ตรวจสอบ webhook configuration
kubectl get validatingwebhookconfigurations | grep vpa
kubectl get mutatingwebhookconfigurations | grep vpa
```

### คำสั่ง VPA ที่ใช้บ่อย

```bash
# ดู VPA ทั้งหมด
kubectl get vpa -A

# ดู VPA detail และ recommendations
kubectl describe vpa my-app-vpa -n production

# ดู recommendations แบบ JSON
kubectl get vpa my-app-vpa -n production -o json | python3 -m json.tool

# ดู VPA events
kubectl get events -n production | grep vpa

# Force evict Pod เพื่อ apply VPA recommendations
kubectl delete pod my-app-pod-xxx -n production

# ดู VerticalPodAutoscalerCheckpoint (history ของ recommendations)
kubectl get verticalpodautoscalercheckpoints -n production

# ลบ VPA
kubectl delete vpa my-app-vpa -n production
```

---

## Best Practices

### 1. เริ่มด้วย Off Mode เสมอ

```yaml
updatePolicy:
  updateMode: "Off"   # สังเกต recommendations ก่อน
```

รอ 24-72 ชั่วโมงเพื่อให้ VPA เก็บ data ครบ cycle แล้วค่อย switch เป็น Auto

### 2. ตั้ง minAllowed และ maxAllowed

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: "*"
    minAllowed:
      cpu: "50m"        # ป้องกัน VPA ลด resource ต่ำเกินไป
      memory: "64Mi"
    maxAllowed:
      cpu: "2"          # ป้องกัน VPA เพิ่ม resource มากเกินไป
      memory: "2Gi"
```

### 3. ใช้ controlledResources เพื่อจำกัดขอบเขต

```yaml
# ถ้าใช้ HPA ด้วย CPU - ไม่ให้ VPA ยุ่งกับ CPU
resourcePolicy:
  containerPolicies:
  - containerName: "*"
    controlledResources: ["memory"]  # ควบคุมเฉพาะ memory
```

### 4. ตั้ง minReplicas สำหรับ Auto Mode

```yaml
updatePolicy:
  updateMode: "Auto"
  minReplicas: 2    # ต้องมี pod อย่างน้อย 2 ตัวก่อน VPA จะ evict
```

### 5. Monitor ผลกระทบของ VPA

```bash
# Script ตรวจสอบว่า VPA เปลี่ยน resources อะไรบ้าง
kubectl get events -n production \
  --field-selector reason=EvictedByVPA \
  --sort-by='.lastTimestamp' \
  | tail -20

# ดู actual vs recommended resources
kubectl top pods -n production -l app=my-app
kubectl get vpa my-app-vpa -n production -o yaml | grep target
```

### 6. ทดสอบใน Staging ก่อน

```
Flow ที่ดี:
1. Deploy VPA ใน Off mode บน Production
2. เก็บ recommendations 1-2 สัปดาห์
3. Apply recommendations ใน Staging ด้วย Auto mode
4. ทดสอบว่า app ทำงานได้ดี
5. Apply ใน Production
```

---

## สรุป

VPA เป็นเครื่องมือที่ดีสำหรับ:

1. **Resource Right-sizing**: หาค่า requests/limits ที่เหมาะสม
2. **ลด Waste**: ลด over-provisioning
3. **ป้องกัน OOMKill**: ปรับ memory ก่อนที่จะเกิด problem
4. **ร่วมกับ HPA**: VPA ควบคุม memory, HPA scale ด้วย CPU

ข้อระวัง:
- VPA ต้อง restart Pod เพื่อปรับ resources (ยกเว้น in-place update ที่กำลังพัฒนา)
- ไม่ใช้ VPA และ HPA กับ metric เดียวกัน
- เริ่มด้วย Off mode เสมอ

ในบทต่อไป เราจะเรียนรู้เรื่อง **Resource Limits** - การจัดการทรัพยากรของ Pods อย่างละเอียด

---

## VPA Algorithm Deep Dive

### ภาพรวมของ VPA Algorithm

VPA Recommender ใช้ algorithm หลายขั้นตอนในการคำนวณ recommendations:

```
1. เก็บ Metrics จาก Prometheus/Metrics Server
   ↓
2. Histogram Bucketing (เก็บ distribution ของการใช้ resources)
   ↓
3. Percentile Calculation (คำนวณ P50, P90, P95, P99)
   ↓
4. Apply Safety Margin
   ↓
5. Bound Recommendations ด้วย Min/Max policies
   ↓
6. คำนวณ Lower Bound, Target, Upper Bound
```

### Histogram Decay

VPA ใช้ exponential decay histogram เพื่อให้ข้อมูลเก่ามีน้ำหนักน้อยกว่าข้อมูลใหม่

```
น้ำหนักของ sample ที่อายุ t ชั่วโมง = exp(-t * decay_factor)

ค่า default:
- CPU halflife: 24 ชั่วโมง (ข้อมูล 24h ที่แล้วมีน้ำหนักครึ่งหนึ่ง)
- Memory halflife: 24 ชั่วโมง
```

### Percentile ที่ VPA ใช้

```
Target (ค่าที่แนะนำ):
- CPU: Percentile 90 ของการใช้งานใน confidence window
- Memory: Percentile 90 ของ Working Set Size

Lower Bound (ค่าต่ำสุดที่ safe):
- CPU: Percentile 50 (ไม่ต่ำกว่า P50)
- Memory: Percentile 50

Upper Bound (ค่าสูงสุดที่จำเป็น):
- CPU: Percentile 95
- Memory: Percentile 95
```

### Confidence Window

VPA ต้องการข้อมูลอย่างน้อย X วันก่อนให้ recommendation

```yaml
# VPA Recommender configuration
# ค่า default:
# - confidenceMultiplier: 1.0 (ต้องการ data เพียงพอ)
# - backoffLimit: ลองใหม่ถ้า data ไม่พอ

# คำนวณ confidence:
# confidence = min(1.0, dataAge / minRequiredAge)
# minRequiredAge = 2 วัน (default)
```

---

## VPA Components Deep Dive

### 1. VPA Recommender

Recommender เป็น component ที่คำนวณ resource recommendations

```
Architecture:
┌─────────────────────────────────────────────┐
│                VPA Recommender               │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │  Metrics Fetch Loop (ทุก 1 นาที)     │   │
│  │  - Prometheus / Metrics Server        │   │
│  │  - เก็บ CPU/Memory usage history      │   │
│  └──────────────────────────────────────┘   │
│                ↓                            │
│  ┌──────────────────────────────────────┐   │
│  │  Histogram Update                    │   │
│  │  - Update histogram buckets           │   │
│  │  - Apply decay function              │   │
│  └──────────────────────────────────────┘   │
│                ↓                            │
│  ┌──────────────────────────────────────┐   │
│  │  Recommendation Calculation          │   │
│  │  - คำนวณ percentiles                  │   │
│  │  - Apply min/max bounds              │   │
│  │  - Apply safety margin               │   │
│  └──────────────────────────────────────┘   │
│                ↓                            │
│  ┌──────────────────────────────────────┐   │
│  │  Update VPA Status                   │   │
│  │  - เขียน recommendations ลง VPA object│   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

```bash
# ดู Recommender logs
kubectl logs -n kube-system \
    deployment/vpa-recommender \
    --tail=50

# ดู Recommender metrics
kubectl port-forward -n kube-system \
    deployment/vpa-recommender 8942:8942
curl http://localhost:8942/metrics | grep vpa_
```

### 2. VPA Admission Controller

Admission Controller intercepted Pod creation request และปรับ resources

```
Flow:
1. User/Controller สร้าง Pod
2. API Server ส่งไปที่ Admission Webhook
3. Admission Controller ดู VPA recommendation ที่ match
4. ปรับ resources.requests และ resources.limits
5. ส่ง Pod กลับไปให้ API Server
6. Pod ถูกสร้างด้วย resources ที่ถูกปรับแล้ว
```

```yaml
# ดู Admission Controller configuration
kubectl get mutatingwebhookconfigurations | grep vpa

# ดูรายละเอียด
kubectl describe mutatingwebhookconfiguration vpa-webhook-config
```

```bash
# ทดสอบว่า Admission Controller ทำงาน
# สร้าง Pod แล้วดูว่า resources ถูกปรับหรือไม่
cat > test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: vpa-test
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 1m
        memory: 1Mi
EOF

kubectl apply -f test-pod.yaml
kubectl get pod vpa-test \
    -o jsonpath='{.spec.containers[0].resources}' | jq
# ถ้า VPA Auto mode ทำงาน resources จะถูกปรับ
```

### 3. VPA Updater

Updater ตรวจสอบ Running Pods และ evict ถ้า resources ต่างจาก recommendation มาก

```
Updater Loop (ทุก 1 นาที):
1. List Pods ที่จัดการโดย VPA
2. เปรียบเทียบ current resources กับ recommendation
3. ถ้าต่างเกิน threshold → evict Pod
4. Pod ใหม่จะถูกสร้างโดย ReplicaSet/StatefulSet
5. Admission Controller ปรับ resources ตอนสร้างใหม่
```

```bash
# ดู Updater logs
kubectl logs -n kube-system \
    deployment/vpa-updater \
    --tail=50

# ดู eviction events
kubectl get events \
    --field-selector reason=EvictedByVPA \
    --all-namespaces
```

---

## VPA + HPA ใช้ร่วมกัน

### ปัญหาและข้อจำกัด

ไม่ควรใช้ VPA และ HPA กับ metric เดียวกัน เพราะ:
- HPA scale Pods ตาม CPU/Memory
- VPA ปรับ CPU/Memory requests
- ถ้าใช้ร่วมกันบน CPU: VPA เพิ่ม CPU request → HPA เห็น CPU utilization ลด → scale down → VPA เห็น load สูงขึ้น → เพิ่ม CPU อีก (loop)

### Pattern ที่แนะนำ: VPA สำหรับ Memory, HPA สำหรับ CPU

```yaml
# VPA: ปรับ Memory อัตโนมัติ
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    - containerName: my-container
      controlledResources:
      # VPA ควบคุมเฉพาะ memory
      - memory
      minAllowed:
        memory: 64Mi
      maxAllowed:
        memory: 4Gi
```

```yaml
# HPA: scale ตาม CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # ไม่ใช้ memory ใน HPA เพราะ VPA จัดการอยู่
```

```yaml
# Deployment ที่ใช้ร่วมกัน
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-app:1.0
        resources:
          requests:
            # CPU: HPA จะ scale ตาม metric นี้
            cpu: 200m
            # Memory: VPA จะปรับค่านี้
            memory: 256Mi
          limits:
            cpu: "1"
            memory: 1Gi
```

### Pattern 2: VPA Off + HPA custom metrics

```yaml
# VPA ใช้เฉพาะ recommend (ไม่ auto-update)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa-advisory
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"     # แค่ recommend ไม่แก้ไข
  resourcePolicy:
    containerPolicies:
    - containerName: my-container
      controlledResources: ["cpu", "memory"]
```

```yaml
# HPA ใช้ custom metrics (ไม่ใช่ CPU/Memory)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue: "my-queue"
      target:
        type: Value
        value: "100"
```

### ตรวจสอบ VPA + HPA ทำงานร่วมกัน

```bash
# ดู VPA recommendations
kubectl get vpa my-app-vpa -o yaml | \
    grep -A20 "recommendation:"

# ดู HPA status
kubectl get hpa my-app-hpa

# Monitor ทั้งสอง
watch -n 5 "
echo '=== VPA Recommendations ===';
kubectl get vpa my-app-vpa -o jsonpath='{.status.recommendation}' | jq;
echo '';
echo '=== HPA Status ===';
kubectl get hpa my-app-hpa;
echo '';
echo '=== Pod Resources ===';
kubectl get pods -l app=my-app \
    -o custom-columns='NAME:.metadata.name,CPU-REQ:.spec.containers[0].resources.requests.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory';
"
```

---

## Custom VPA Policies

### Policy ตาม Container

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: multi-container-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: multi-container-app
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    # Container หลัก: ปรับได้มาก
    - containerName: main-app
      controlledResources: ["cpu", "memory"]
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: "4"
        memory: 8Gi
    # Sidecar: จำกัดให้เล็ก
    - containerName: log-agent
      controlledResources: ["cpu", "memory"]
      minAllowed:
        cpu: 10m
        memory: 16Mi
      maxAllowed:
        cpu: 200m
        memory: 256Mi
    # Prometheus exporter: ไม่ให้ VPA ปรับ
    - containerName: metrics-exporter
      mode: "Off"
```

### Policy สำหรับ Critical Applications

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: critical-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-processor
  updatePolicy:
    updateMode: Auto
    # Eviction policy - เมื่อไหรจะ evict Pod
    evictionRequirements:
    # ต้องมี Pod อื่น Running อยู่อย่างน้อย 1 ตัวก่อน evict
    - resources: ["cpu", "memory"]
      changeRequirement: TargetHigherThanRequests
  resourcePolicy:
    containerPolicies:
    - containerName: payment
      controlledResources: ["cpu", "memory"]
      minAllowed:
        # กำหนด minimum ที่ safe สำหรับ production
        cpu: 500m
        memory: 512Mi
      maxAllowed:
        # จำกัด max เพื่อควบคุม cost
        cpu: "8"
        memory: 16Gi
```

### Policy แบบ Conservative (ปลอดภัยกว่า)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: conservative-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    # Initial: ปรับเฉพาะ Pod ใหม่ (ไม่ evict ที่รันอยู่)
    updateMode: Initial
  resourcePolicy:
    containerPolicies:
    - containerName: app
      controlledResources: ["memory"]   # เฉพาะ memory เท่านั้น
      minAllowed:
        memory: 128Mi
      maxAllowed:
        memory: 2Gi
```

---

## Workshop ละเอียด: ปรับ Resources อัตโนมัติ

### ภาพรวม Workshop

Workshop นี้จะ:
1. ติดตั้ง VPA ในสภาพแวดล้อม local
2. Deploy application ที่มี resources ไม่เหมาะสม
3. เฝ้าดู VPA recommendations
4. Enable Auto mode และสังเกตการปรับ
5. ใช้ร่วมกับ HPA

### ขั้นตอนที่ 1: ติดตั้ง VPA (Minikube/Kind)

```bash
# Clone VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# ติดตั้ง VPA components
./hack/vpa-install.sh

# ตรวจสอบการติดตั้ง
kubectl get pods -n kube-system | grep vpa
# ควรเห็น:
# vpa-admission-controller-xxx   1/1     Running
# vpa-recommender-xxx            1/1     Running
# vpa-updater-xxx                1/1     Running

# ตรวจสอบ CRDs
kubectl get crd | grep autoscaling.k8s.io
# verticalpodautoscalers.autoscaling.k8s.io
# verticalpodautoscalercheckpoints.autoscaling.k8s.io
```

### ขั้นตอนที่ 2: Deploy Application ที่มี Resources ไม่เหมาะสม

```bash
cat > workshop-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
      - name: demo
        image: polinux/stress
        # Resources ที่ over-provisioned มาก
        resources:
          requests:
            cpu: "2"        # ต้องการจริงๆ แค่ 200m
            memory: 2Gi     # ต้องการจริงๆ แค่ 256Mi
          limits:
            cpu: "4"
            memory: 4Gi
        command:
        - stress
        args:
        # CPU load: ~200m cores
        - "--cpu"
        - "1"
        - "--cpu-load"
        - "20"
        # Memory: ~200Mi
        - "--vm"
        - "1"
        - "--vm-bytes"
        - "200M"
        - "--vm-hang"
        - "1"
EOF

kubectl apply -f workshop-app.yaml
kubectl get pods
```

### ขั้นตอนที่ 3: ตั้ง VPA ใน Off Mode (สังเกต)

```bash
cat > workshop-vpa-off.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: resource-demo-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: resource-demo
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: demo
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: "4"
        memory: 8Gi
EOF

kubectl apply -f workshop-vpa-off.yaml

# รอ VPA เก็บข้อมูล (อย่างน้อย 5-10 นาที)
echo "Waiting for VPA to collect data..."
sleep 300  # รอ 5 นาที
```

### ขั้นตอนที่ 4: ดู Recommendations

```bash
# ดู VPA recommendations
kubectl describe vpa resource-demo-vpa

# ดูแบบ YAML ละเอียด
kubectl get vpa resource-demo-vpa -o yaml

# Script ดู recommendations อย่างชัดเจน
kubectl get vpa resource-demo-vpa -o json | jq '
  .status.recommendation.containerRecommendations[] |
  {
    container: .containerName,
    lowerBound: .lowerBound,
    target: .target,
    upperBound: .upperBound,
    uncappedTarget: .uncappedTarget
  }
'

# ผลลัพธ์ตัวอย่าง:
# {
#   "container": "demo",
#   "target": { "cpu": "202m", "memory": "262144k" },
#   "lowerBound": { "cpu": "100m", "memory": "131072k" },
#   "upperBound": { "cpu": "404m", "memory": "524288k" }
# }
# เห็นได้ชัดว่า ค่า request เดิม (2 CPU, 2Gi) นั้น over-provision มาก
```

### ขั้นตอนที่ 5: Enable Auto Mode

```bash
# อัพเดต VPA เป็น Auto mode
kubectl patch vpa resource-demo-vpa \
    --type=merge \
    -p '{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'

# สังเกต VPA Updater ทำงาน
kubectl get events \
    --sort-by='.lastTimestamp' | grep -i evict

# ดู Pods ถูก evict และสร้างใหม่
kubectl get pods -w

# ดู resources ของ Pod ใหม่
kubectl get pods -l app=resource-demo \
    -o custom-columns=\
"NAME:.metadata.name,\
CPU-REQ:.spec.containers[0].resources.requests.cpu,\
MEM-REQ:.spec.containers[0].resources.requests.memory"
# ควรเห็นค่าต่ำกว่าเดิมมาก
```

### ขั้นตอนที่ 6: เพิ่ม HPA สำหรับ Scale

```bash
cat > workshop-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: resource-demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: resource-demo
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
EOF

kubectl apply -f workshop-hpa.yaml

# แก้ไข VPA ให้ควบคุมเฉพาะ memory (ไม่ใช่ CPU ที่ HPA ใช้อยู่)
kubectl patch vpa resource-demo-vpa \
    --type=merge \
    -p '{
      "spec": {
        "resourcePolicy": {
          "containerPolicies": [{
            "containerName": "demo",
            "controlledResources": ["memory"]
          }]
        }
      }
    }'
```

### ขั้นตอนที่ 7: สรุปผลการ Workshop

```bash
# Dashboard สรุป
echo "=== Workshop Summary ==="
echo ""
echo "=== VPA Recommendations ==="
kubectl get vpa resource-demo-vpa -o jsonpath=\
'{.status.recommendation.containerRecommendations[0]}' | jq

echo ""
echo "=== HPA Status ==="
kubectl get hpa resource-demo-hpa

echo ""
echo "=== Current Pod Resources ==="
kubectl get pods -l app=resource-demo \
    -o custom-columns=\
"NAME:.metadata.name,\
REPLICAS:.metadata.name,\
CPU-REQ:.spec.containers[0].resources.requests.cpu,\
MEM-REQ:.spec.containers[0].resources.requests.memory,\
CPU-LIM:.spec.containers[0].resources.limits.cpu,\
MEM-LIM:.spec.containers[0].resources.limits.memory"

echo ""
echo "=== Resource Savings ==="
echo "Original CPU request: 2000m per pod"
echo "Optimized CPU request: ~200m per pod"
echo "CPU savings: ~90%"
echo ""
echo "Original Memory request: 2048Mi per pod"
echo "Optimized Memory request: ~256Mi per pod"
echo "Memory savings: ~87%"

# Cleanup
kubectl delete -f workshop-app.yaml
kubectl delete -f workshop-vpa-off.yaml
kubectl delete -f workshop-hpa.yaml
```

---

## แบบฝึกหัด: VPA

### แบบฝึกหัดที่ 1: VPA Off Mode

**โจทย์**: สร้าง VPA ใน Off mode สำหรับ nginx deployment แล้วดู recommendations

**เฉลย**:
```bash
# สร้าง nginx deployment
kubectl create deployment nginx-app \
    --image=nginx:1.21 \
    --replicas=2

# ตั้ง resource requests (จงใจ over-provision)
kubectl set resources deployment/nginx-app \
    --requests=cpu=500m,memory=512Mi \
    --limits=cpu=2,memory=2Gi

# สร้าง VPA
cat > nginx-vpa.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-app
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: 25m
        memory: 32Mi
      maxAllowed:
        cpu: "2"
        memory: 2Gi
EOF
kubectl apply -f nginx-vpa.yaml

# รอ 5-10 นาที แล้วดู recommendations
sleep 300
kubectl describe vpa nginx-vpa
```

---

### แบบฝึกหัดที่ 2: VPA Initial Mode

**โจทย์**: เปลี่ยน VPA เป็น Initial mode แล้วสังเกต: Pods เดิมไม่ถูก restart แต่ Pods ใหม่ได้ resources ที่ปรับแล้ว

**เฉลย**:
```bash
# อัพเดต VPA mode
kubectl patch vpa nginx-vpa \
    --type=merge \
    -p '{"spec":{"updatePolicy":{"updateMode":"Initial"}}}'

# Scale down แล้ว scale up เพื่อทดสอบ
kubectl scale deployment/nginx-app --replicas=0
kubectl scale deployment/nginx-app --replicas=2

# ดู resources ของ Pods ใหม่
kubectl get pods -l app=nginx-app \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'
# Pods ใหม่ควรมี resources ตาม VPA recommendation
```

---

### แบบฝึกหัดที่ 3: Container-specific Policy

**โจทย์**: สร้าง Deployment ที่มี 2 containers (main + sidecar) แล้วตั้ง VPA policy แตกต่างกัน

**เฉลย**:
```yaml
# Deployment ที่มี 2 containers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: two-container-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: two-container
  template:
    metadata:
      labels:
        app: two-container
    spec:
      containers:
      - name: main-app
        image: nginx:1.21
        resources:
          requests:
            cpu: 1000m
            memory: 1Gi
      - name: log-sidecar
        image: busybox
        command: ["sh", "-c", "while true; do sleep 30; done"]
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
```

```yaml
# VPA ที่มี policy แตกต่างกันต่อ container
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: two-container-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: two-container-app
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    # Main app: ปรับได้ทั้ง CPU และ Memory
    - containerName: main-app
      controlledResources: ["cpu", "memory"]
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: "4"
        memory: 8Gi
    # Sidecar: ปรับได้แค่ Memory
    - containerName: log-sidecar
      controlledResources: ["memory"]
      minAllowed:
        memory: 16Mi
      maxAllowed:
        memory: 256Mi
```

---

### แบบฝึกหัดที่ 4: VPA + HPA ร่วมกัน

**โจทย์**: ตั้ง VPA ควบคุม memory และ HPA scale ตาม CPU สำหรับ application เดียวกัน

**เฉลย**:
```bash
# สร้าง Deployment
kubectl create deployment combo-app \
    --image=nginx:1.21 \
    --replicas=2

kubectl set resources deployment/combo-app \
    --requests=cpu=200m,memory=256Mi \
    --limits=cpu=1,memory=1Gi

# VPA สำหรับ memory เท่านั้น
cat > combo-vpa.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: combo-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: combo-app
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      # ควบคุมเฉพาะ memory
      controlledResources: ["memory"]
      minAllowed:
        memory: 64Mi
      maxAllowed:
        memory: 2Gi
EOF
kubectl apply -f combo-vpa.yaml

# HPA สำหรับ CPU
kubectl autoscale deployment combo-app \
    --cpu-percent=70 \
    --min=2 \
    --max=10

# ตรวจสอบทั้งสอง
kubectl get vpa combo-vpa
kubectl get hpa combo-app
```

---

### แบบฝึกหัดที่ 5: Debugging VPA

**โจทย์**: แก้ปัญหา VPA ที่ไม่แสดง recommendations

```bash
# สร้าง VPA ที่มีปัญหา (target ไม่มีอยู่จริง)
cat > broken-vpa.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: broken-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nonexistent-deployment
  updatePolicy:
    updateMode: "Off"
EOF
kubectl apply -f broken-vpa.yaml
```

**เฉลย - การ Debug**:
```bash
# ขั้นที่ 1: ตรวจสอบ VPA status
kubectl describe vpa broken-vpa
# ดูใน Conditions - จะเห็น error ว่า target ไม่มีอยู่

# ขั้นที่ 2: ดู VPA Recommender logs
kubectl logs -n kube-system \
    deployment/vpa-recommender | grep broken-vpa

# ขั้นที่ 3: แก้ไข VPA ให้ชี้ไปที่ deployment ที่มีจริง
kubectl patch vpa broken-vpa \
    --type=merge \
    -p '{"spec":{"targetRef":{"name":"nginx-app"}}}'

# ขั้นที่ 4: ตรวจสอบอีกครั้ง
kubectl describe vpa broken-vpa
# ควรเห็น recommendations หลังจากรอสักครู่
```

---

## สรุปทบทวน VPA

### Cheat Sheet

```bash
# สร้าง VPA
kubectl apply -f vpa.yaml

# ดูรายการ VPA
kubectl get vpa
kubectl get vpa --all-namespaces

# ดู recommendations
kubectl describe vpa <name>
kubectl get vpa <name> -o yaml | grep -A20 recommendation

# เปลี่ยน mode
kubectl patch vpa <name> \
    --type=merge \
    -p '{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'

# ลบ VPA
kubectl delete vpa <name>
```

### สรุปเปรียบเทียบ VPA Modes

| Mode | ปรับ Pod ที่รัน | ปรับ Pod ใหม่ | แนะนำสำหรับ |
|------|----------------|--------------|------------|
| Off | ไม่ | ไม่ | สังเกตเท่านั้น |
| Initial | ไม่ | ใช่ | ระวัง disruption |
| Recreate | ใช่ (evict) | ใช่ | ยอมรับ restart |
| Auto | ใช่ (intelligent) | ใช่ | production ที่ stable |

VPA เป็นเครื่องมือที่ทรงพลังในการ optimize resource usage แต่ต้องเข้าใจ trade-off ระหว่างความแม่นยำของ recommendations และ disruption ที่เกิดจาก Pod eviction
