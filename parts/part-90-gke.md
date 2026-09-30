# Part 90: Google Kubernetes Engine (GKE)

## บทนำ

Google Kubernetes Engine (GKE) คือ managed Kubernetes service ของ Google Cloud ที่เป็นหนึ่งใน most mature managed Kubernetes offerings เนื่องจาก Google เป็นผู้สร้าง Kubernetes เอง

GKE มีสองโหมดหลัก:
- **GKE Standard**: คุณจัดการ node pools เอง (similar to EKS)
- **GKE Autopilot**: Google จัดการ infrastructure ทั้งหมด คุณจ่ายตาม pods เท่านั้น

---

## 90.1 GKE Architecture

### Standard Mode

```
Google Cloud Project
└── GKE Cluster (Standard)
    ├── Control Plane (managed by Google)
    │   └── Multi-zone, highly available
    │
    └── Node Pools (managed by you)
        ├── Node Pool 1: e2-standard-4 (3 nodes)
        ├── Node Pool 2: n2-standard-8 (spot instances)
        └── Node Pool 3: GPU nodes
```

### Autopilot Mode

```
Google Cloud Project
└── GKE Cluster (Autopilot)
    ├── Control Plane (managed by Google)
    └── Nodes (managed by Google - you don't see them)
        └── เพิ่ม/ลด nodes อัตโนมัติ
            จ่ายตาม pod resources เท่านั้น
```

---

## 90.2 ติดตั้งและ Setup

### ติดตั้ง gcloud CLI

```bash
# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# macOS
brew install --cask google-cloud-sdk
gcloud init

# ตรวจสอบ
gcloud version
gcloud auth login
gcloud config set project my-project-id

# ติดตั้ง kubectl plugin
gcloud components install kubectl
gcloud components install gke-gcloud-auth-plugin
```

### สร้าง GKE Standard Cluster

```bash
# สร้าง cluster แบบง่าย
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --disk-size 100 \
  --disk-type pd-balanced

# อัปเดต kubeconfig
gcloud container clusters get-credentials my-cluster --zone us-central1-a

# ตรวจสอบ
kubectl get nodes
```

### สร้าง GKE Standard Cluster (Production-ready)

```bash
export PROJECT_ID=my-project-id
export CLUSTER_NAME=production-cluster
export REGION=us-central1

# สร้าง cluster แบบ production
gcloud container clusters create ${CLUSTER_NAME} \
  --project ${PROJECT_ID} \
  --region ${REGION} \
  --cluster-version latest \
  --release-channel regular \
  --enable-ip-alias \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr 172.16.0.0/28 \
  --enable-master-authorized-networks \
  --master-authorized-networks "203.0.113.0/32" \
  --network my-vpc \
  --subnetwork my-subnet \
  --enable-network-policy \
  --enable-shielded-nodes \
  --shielded-secure-boot \
  --shielded-integrity-monitoring \
  --enable-workload-identity \
  --workload-pool ${PROJECT_ID}.svc.id.goog \
  --enable-vertical-pod-autoscaling \
  --enable-dataplane-v2 \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM \
  --no-enable-basic-auth \
  --no-issue-client-certificate \
  --service-account gke-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --num-nodes 0 \  # ไม่สร้าง default pool
  --no-enable-intra-node-visibility

# เพิ่ม node pools หลังสร้าง cluster
```

---

## 90.3 Node Pools

### สร้าง Node Pools

```bash
# System node pool
gcloud container node-pools create system-pool \
  --cluster ${CLUSTER_NAME} \
  --region ${REGION} \
  --machine-type e2-standard-2 \
  --num-nodes 1 \
  --min-nodes 1 \
  --max-nodes 3 \
  --enable-autoscaling \
  --node-labels role=system \
  --node-taints dedicated=system:NoSchedule \
  --disk-type pd-balanced \
  --disk-size 50 \
  --image-type COS_CONTAINERD \
  --shielded-secure-boot \
  --shielded-integrity-monitoring \
  --enable-autoupgrade \
  --enable-autorepair

# Application node pool
gcloud container node-pools create application-pool \
  --cluster ${CLUSTER_NAME} \
  --region ${REGION} \
  --machine-type n2-standard-4 \
  --num-nodes 1 \
  --min-nodes 1 \
  --max-nodes 10 \
  --enable-autoscaling \
  --node-labels role=application \
  --disk-type pd-ssd \
  --disk-size 100 \
  --image-type COS_CONTAINERD \
  --enable-autoupgrade \
  --enable-autorepair

# Spot/Preemptible node pool (สำหรับ non-critical workloads)
gcloud container node-pools create spot-pool \
  --cluster ${CLUSTER_NAME} \
  --region ${REGION} \
  --machine-type n2-standard-8 \
  --num-nodes 0 \
  --min-nodes 0 \
  --max-nodes 20 \
  --enable-autoscaling \
  --spot \
  --node-labels role=spot \
  --node-taints spot=true:NoSchedule \
  --enable-autoupgrade \
  --enable-autorepair

# GPU node pool
gcloud container node-pools create gpu-pool \
  --cluster ${CLUSTER_NAME} \
  --region ${REGION} \
  --machine-type n1-standard-4 \
  --accelerator type=nvidia-tesla-t4,count=1 \
  --num-nodes 0 \
  --min-nodes 0 \
  --max-nodes 5 \
  --enable-autoscaling \
  --node-labels role=gpu \
  --node-taints nvidia.com/gpu=present:NoSchedule
```

### Node Pool Configuration ด้วย Terraform

```hcl
# main.tf
resource "google_container_cluster" "production" {
  name     = "production-cluster"
  location = "us-central1"
  project  = var.project_id
  
  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1
  
  # Networking
  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name
  
  ip_allocation_policy {
    cluster_ipv4_cidr_block  = "10.48.0.0/14"
    services_ipv4_cidr_block = "10.52.0.0/20"
  }
  
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
  
  # Security
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
    gcp_filestore_csi_driver_config {
      enabled = true
    }
    gcs_fuse_csi_driver_config {
      enabled = true
    }
  }
  
  release_channel {
    channel = "REGULAR"
  }
  
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }
  
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    managed_prometheus {
      enabled = true
    }
  }
}

resource "google_container_node_pool" "application" {
  name     = "application-pool"
  location = "us-central1"
  cluster  = google_container_cluster.production.name
  
  autoscaling {
    min_node_count  = 1
    max_node_count  = 10
    location_policy = "BALANCED"
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
  
  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
    strategy        = "SURGE"
  }
  
  node_config {
    preemptible  = false
    machine_type = "n2-standard-4"
    disk_type    = "pd-ssd"
    disk_size_gb = 100
    image_type   = "COS_CONTAINERD"
    
    labels = {
      role = "application"
    }
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

---

## 90.4 GKE Autopilot

### สร้าง Autopilot Cluster

```bash
# สร้าง Autopilot cluster (ง่ายกว่า Standard มาก)
gcloud container clusters create-auto ${CLUSTER_NAME}-autopilot \
  --project ${PROJECT_ID} \
  --region ${REGION} \
  --release-channel stable \
  --network my-vpc \
  --subnetwork my-subnet

# อัปเดต kubeconfig
gcloud container clusters get-credentials ${CLUSTER_NAME}-autopilot \
  --region ${REGION}

# ดู nodes (จะเห็น nodes ถูก auto-provision)
kubectl get nodes
```

### Autopilot vs Standard

```
Autopilot:
✅ ไม่ต้องจัดการ nodes
✅ จ่ายตาม pod requests เท่านั้น
✅ SLO ครอบคลุม nodes
✅ Security ดีกว่า (GKE managed security)
❌ ไม่สามารถ customize nodes ได้
❌ บาง features ไม่พร้อมใช้ (DaemonSets ที่ privileged)
❌ Cold start เมื่อ scale from zero

Standard:
✅ ควบคุม node ได้เต็มที่
✅ รองรับ DaemonSets, privileged containers
✅ เลือก machine type ได้ตามต้องการ
❌ ต้องจัดการ node lifecycle
❌ จ่ายตาม nodes ไม่ใช่ pods
```

### Autopilot Resource Classes

```yaml
# กำหนด compute class สำหรับ Autopilot
apiVersion: v1
kind: Pod
metadata:
  name: high-memory-pod
spec:
  nodeSelector:
    cloud.google.com/compute-class: "Balanced"  # หรือ "Scale-Out", "Performance"
  containers:
    - name: app
      image: myapp:latest
      resources:
        requests:
          cpu: "2"
          memory: "8Gi"
        limits:
          cpu: "2"
          memory: "8Gi"
```

---

## 90.5 GKE Workload Identity

### Setup Workload Identity

```bash
# Enable Workload Identity (ทำตอนสร้าง cluster)
gcloud container clusters update ${CLUSTER_NAME} \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --region ${REGION}

# สร้าง Google Service Account (GSA)
gcloud iam service-accounts create my-app-gsa \
  --project=${PROJECT_ID} \
  --display-name="My App Service Account"

# ให้ GSA มีสิทธิ์ใน Google Cloud
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:my-app-gsa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# สร้าง Kubernetes Service Account (KSA)
kubectl create serviceaccount my-app-ksa -n production

# Bind KSA กับ GSA
gcloud iam service-accounts add-iam-policy-binding \
  my-app-gsa@${PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${PROJECT_ID}.svc.id.goog[production/my-app-ksa]"

# Annotate KSA
kubectl annotate serviceaccount my-app-ksa \
  -n production \
  iam.gke.io/gcp-service-account=my-app-gsa@${PROJECT_ID}.iam.gserviceaccount.com
```

```yaml
# ใช้ Workload Identity ใน Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: my-app-ksa  # ใช้ KSA ที่ bind กับ GSA
      containers:
        - name: app
          image: myapp:latest
          # Application จะ automatically ได้ Google Cloud credentials
```

---

## 90.6 GKE Storage

### Persistent Disks

```yaml
# StorageClass สำหรับ SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-ssd
  replication-type: none  # หรือ regional-pd สำหรับ multi-zone
  disk-encryption-key: projects/${PROJECT}/locations/${REGION}/keyRings/${RING}/cryptoKeys/${KEY}

---
# Regional PD (สำหรับ high availability)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-ssd
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-ssd
  replication-type: regional-pd
  zones: us-central1-a,us-central1-b
```

### GCS FUSE (Cloud Storage)

```yaml
# Deployment ที่ mount GCS bucket
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-gcs
spec:
  template:
    spec:
      serviceAccountName: my-app-ksa  # ต้องมีสิทธิ์ใน GCS
      containers:
        - name: app
          image: myapp:latest
          volumeMounts:
            - name: gcs-bucket
              mountPath: /data
      volumes:
        - name: gcs-bucket
          csi:
            driver: gcsfuse.csi.storage.gke.io
            readOnly: false
            volumeAttributes:
              bucketName: my-bucket
              mountOptions: implicit-dirs
```

### Filestore (NFS)

```yaml
# StorageClass สำหรับ Filestore
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-rwx
provisioner: filestore.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  tier: standard
  network: my-vpc

---
# PVC ที่ใช้ Filestore (ReadWriteMany)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: filestore-rwx
  resources:
    requests:
      storage: 1Ti
```

---

## 90.7 GKE Ingress

### GKE Ingress (นใช้ Google Cloud Load Balancer)

```yaml
# กำหนด BackendConfig สำหรับ health check
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: my-app-backend
  namespace: production
spec:
  healthCheck:
    checkIntervalSec: 30
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /healthz
    port: 8080
  sessionAffinity:
    affinityType: CLIENT_IP
    affinityCookieTtlSec: 3600
  timeoutSec: 60
  cdn:
    enabled: true
    cachePolicy:
      includeHost: true
      includeProtocol: true
      includeQueryString: false
  logging:
    enable: true
    sampleRate: 1.0
  connectionDraining:
    drainingTimeoutSec: 30

---
# Service ที่ใช้ BackendConfig
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  annotations:
    cloud.google.com/backend-config: '{"default": "my-app-backend"}'
    cloud.google.com/neg: '{"ingress": true}'
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080

---
# FrontendConfig สำหรับ SSL redirect
apiVersion: networking.gke.io/v1beta1
kind: FrontendConfig
metadata:
  name: my-frontend
  namespace: production
spec:
  redirectToHttps:
    enabled: true
    responseCodeName: MOVED_PERMANENTLY_DEFAULT
  sslPolicy: my-ssl-policy

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: gce
    networking.gke.io/v1beta1.FrontendConfig: my-frontend
    kubernetes.io/ingress.global-static-ip-name: my-static-ip
    networking.gke.io/managed-certificates: my-ssl-cert
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### Managed SSL Certificates

```yaml
# Managed Certificate (Google จัดการ SSL ให้อัตโนมัติ)
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: my-ssl-cert
  namespace: production
spec:
  domains:
    - myapp.example.com
    - api.myapp.example.com
```

---

## 90.8 GKE Monitoring

### Google Cloud Managed Service for Prometheus (GMP)

```bash
# GMP จะถูก enable อัตโนมัติเมื่อสร้าง cluster
# ดู status
kubectl get pods -n gke-managed-prometheus
```

```yaml
# PodMonitoring resource (GMP-specific)
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

```yaml
# ClusterPodMonitoring สำหรับ cluster-wide monitoring
apiVersion: monitoring.googleapis.com/v1
kind: ClusterPodMonitoring
metadata:
  name: cluster-monitoring
spec:
  selector:
    matchLabels:
      monitoring.googleapis.com/enabled: "true"
  endpoints:
    - port: metrics
      interval: 60s
```

### Cloud Logging

```bash
# ดู logs ใน Cloud Logging
gcloud logging read "resource.type=k8s_container AND resource.labels.cluster_name=${CLUSTER_NAME}" \
  --limit 100 \
  --format json

# หรือใช้ kubectl logs (ส่งไป Cloud Logging อัตโนมัติ)
kubectl logs -n production deployment/my-app
```

---

## 90.9 Workshop: Production GKE Setup

### เป้าหมาย
ตั้ง production-ready GKE cluster แบบครบถ้วน

### ขั้นตอนที่ 1: สร้าง Infrastructure

```bash
# ตั้ง variables
export PROJECT_ID=my-gke-workshop
export REGION=us-central1
export CLUSTER_NAME=production-gke

# สร้าง project (ถ้าต้องการ)
gcloud projects create ${PROJECT_ID}
gcloud config set project ${PROJECT_ID}
gcloud services enable container.googleapis.com

# สร้าง VPC
gcloud compute networks create workshop-vpc \
  --subnet-mode custom \
  --bgp-routing-mode regional

gcloud compute networks subnets create workshop-subnet \
  --network workshop-vpc \
  --region ${REGION} \
  --range 10.0.0.0/20 \
  --secondary-range pods=10.48.0.0/14,services=10.52.0.0/20

# สร้าง GKE cluster
gcloud container clusters create ${CLUSTER_NAME} \
  --project ${PROJECT_ID} \
  --region ${REGION} \
  --release-channel regular \
  --enable-ip-alias \
  --network workshop-vpc \
  --subnetwork workshop-subnet \
  --cluster-secondary-range-name pods \
  --services-secondary-range-name services \
  --enable-workload-identity \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --enable-shielded-nodes \
  --enable-network-policy \
  --no-enable-basic-auth \
  --monitoring SYSTEM,WORKLOAD \
  --logging SYSTEM,WORKLOAD \
  --num-nodes 0

# เพิ่ม node pool
gcloud container node-pools create general \
  --cluster ${CLUSTER_NAME} \
  --region ${REGION} \
  --machine-type n2-standard-4 \
  --min-nodes 1 \
  --max-nodes 10 \
  --enable-autoscaling \
  --enable-autoupgrade \
  --enable-autorepair \
  --num-nodes 2

# อัปเดต kubeconfig
gcloud container clusters get-credentials ${CLUSTER_NAME} --region ${REGION}
```

### ขั้นตอนที่ 2: ติดตั้ง Core Add-ons

```bash
# ติดตั้ง cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# ติดตั้ง External DNS
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install external-dns bitnami/external-dns \
  --namespace kube-system \
  --set provider=google \
  --set google.project=${PROJECT_ID} \
  --set domainFilters[0]=example.com

# ตรวจสอบ
kubectl get pods -n cert-manager
kubectl get pods -n kube-system | grep external-dns
```

### ขั้นตอนที่ 3: Deploy Application

```yaml
# gke-production-app.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted

---
# Google Cloud Service Account สำหรับ app
# (สร้างด้วย gcloud ก่อน)

---
# Kubernetes Service Account พร้อม Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: web-app-gsa@${PROJECT_ID}.iam.gserviceaccount.com

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        monitoring.googleapis.com/enabled: "true"
    spec:
      serviceAccountName: web-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: us-central1-docker.pkg.dev/${PROJECT_ID}/myrepo/web-app:1.0.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          env:
            - name: PORT
              value: "8080"
            - name: GCP_PROJECT
              value: "${PROJECT_ID}"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web-app

---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  selector:
    app: web-app
  ports:
    - name: http
      port: 80
      targetPort: 8080

---
# Managed Certificate
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: web-app-cert
  namespace: production
spec:
  domains:
    - webapp.example.com

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  namespace: production
  annotations:
    kubernetes.io/ingress.class: gce
    networking.gke.io/managed-certificates: web-app-cert
    kubernetes.io/ingress.global-static-ip-name: web-app-ip
spec:
  rules:
    - host: webapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80

---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120

---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
```

```bash
# Reserve static IP
gcloud compute addresses create web-app-ip \
  --project ${PROJECT_ID} \
  --global

# Apply
kubectl apply -f gke-production-app.yaml

# ดู status
kubectl get ingress web-app -n production -w
kubectl get managedcertificate web-app-cert -n production
kubectl get pods -n production

# ดู load balancer IP
kubectl get ingress web-app -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### ขั้นตอนที่ 4: ตั้ง Monitoring ด้วย GMP

```bash
# กำหนด alerting rule
kubectl apply -f - <<EOF
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: web-app-alerts
  namespace: production
spec:
  groups:
    - name: web-app
      interval: 1m
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{job="web-app",status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="web-app"}[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate detected"
            description: "Error rate is {{ \$value | humanizePercentage }} for {{ \$labels.job }}"
        
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99, 
              sum(rate(http_request_duration_seconds_bucket{job="web-app"}[5m])) by (le)
            ) > 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High latency detected"
            description: "99th percentile latency is {{ \$value }}s"
EOF
```

### ขั้นตอนที่ 5: ทดสอบ Auto-scaling

```bash
# สร้าง load
kubectl run -n production load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://web-app:80; done"

# ดู HPA scaling
kubectl get hpa web-app -n production -w

# ดู node scaling
kubectl get nodes -w

# ดู metrics
kubectl top pods -n production
kubectl top nodes

# หยุด load generator
kubectl delete pod load-generator -n production
```

### ขั้นตอนที่ 6: Cluster Upgrade

```bash
# ดู available versions
gcloud container get-server-config --region ${REGION}

# อัปเกรด control plane
gcloud container clusters upgrade ${CLUSTER_NAME} \
  --master \
  --region ${REGION} \
  --cluster-version 1.29.0-gke.1000

# อัปเกรด node pools
gcloud container clusters upgrade ${CLUSTER_NAME} \
  --node-pool general \
  --region ${REGION}

# ดู upgrade status
gcloud container operations list --filter="targetLink ~ ${CLUSTER_NAME}"
```

### ขั้นตอนที่ 7: Cleanup

```bash
# ลบ resources
kubectl delete -f gke-production-app.yaml

# ลบ cluster
gcloud container clusters delete ${CLUSTER_NAME} \
  --region ${REGION} \
  --quiet

# ลบ VPC
gcloud compute networks subnets delete workshop-subnet --region ${REGION}
gcloud compute networks delete workshop-vpc
```

---

## 90.10 GKE Security Best Practices

```bash
# 1. Enable Binary Authorization
gcloud container clusters update ${CLUSTER_NAME} \
  --region ${REGION} \
  --binauthz-evaluation-mode PROJECT_SINGLETON_POLICY_ENFORCE

# สร้าง Binary Auth policy
cat > policy.yaml <<EOF
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/${PROJECT_ID}/attestors/production-attestor
EOF

gcloud container binauthz policy import policy.yaml

# 2. Enable Security Posture Dashboard
gcloud container clusters update ${CLUSTER_NAME} \
  --region ${REGION} \
  --enable-workload-vulnerability-scanning \
  --security-posture standard

# 3. Enable Config Connector (สำหรับ manage GCP resources ผ่าน Kubernetes)
gcloud container clusters update ${CLUSTER_NAME} \
  --region ${REGION} \
  --update-addons ConfigConnector=ENABLED
```

---

## 90.11 GKE vs EKS เปรียบเทียบ

```
GKE:
✅ Kubernetes expertise (Google สร้าง Kubernetes)
✅ Autopilot mode - serverless nodes
✅ Anthos/GKE Enterprise สำหรับ hybrid
✅ Google Cloud integration (BigQuery, Pub/Sub, etc.)
✅ Managed Prometheus (GMP) built-in
✅ Binary Authorization
✅ Config Connector
❌ Google Cloud lock-in
❌ Pricing อาจแพงกว่าในบาง scenarios

EKS:
✅ AWS ecosystem ที่ใหญ่มาก
✅ IAM integration (IRSA)
✅ Fargate สำหรับ serverless pods
✅ AWS service integration (S3, RDS, etc.)
✅ Excellent networking (VPC CNI)
❌ AWS lock-in
❌ Control plane ไม่ฟรี ($0.10/hour)

เลือก GKE เมื่อ:
- ใช้ Google Cloud ecosystem
- ต้องการ Autopilot (ไม่อยากจัดการ nodes)
- ต้องการ advanced Kubernetes features เร็วที่สุด

เลือก EKS เมื่อ:
- ใช้ AWS ecosystem
- ต้องการ integration กับ AWS services
- Organization ใช้ AWS เป็นหลัก
```

---

## สรุปบทที่ 90 และ Series Parts 81-90

### สิ่งที่เรียนรู้ใน Parts 81-90 (Advanced Kubernetes):

1. **Part 81 - CRDs**: Custom Resource Definitions, Schema Validation, Versioning
2. **Part 82 - Operators**: Operator Pattern, Go/Python Operators, Controller logic
3. **Part 83 - Admission Controllers**: OPA/Gatekeeper, Pod Security Standards
4. **Part 84 - Webhooks**: Mutating/Validating Webhooks, Custom admission logic
5. **Part 85 - API Extension**: API Aggregation Layer, Custom API Server
6. **Part 86 - Multi-cluster**: Architectures, Networking, GitOps
7. **Part 87 - Federation**: KubeFed, Federated Resources, Placement Policies
8. **Part 88 - Cluster API**: CAPI, Infrastructure Providers, Cluster Lifecycle
9. **Part 89 - EKS**: Amazon EKS, eksctl, IAM Integration, Production Setup
10. **Part 90 - GKE**: Google GKE, Autopilot, Workload Identity, Production Setup

### Next Steps

หลังจากเรียน Advanced Kubernetes แล้ว ขั้นตอนถัดไปที่แนะนำ:

```
- ลองสร้าง Operator สำหรับ use case จริง
- ทดสอบ CAPI กับ cloud provider
- ตั้ง Multi-cluster ด้วย Argo CD
- ศึกษา Service Mesh (Istio, Linkerd)
- ศึกษา Platform Engineering ด้วย Backstage
- ศึกษา GitOps ด้วย Flux หรือ Argo CD
```

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes Patterns Book](https://k8spatterns.io/)
