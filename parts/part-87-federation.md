# Part 87: Kubernetes Federation (KubeFed)

## บทนำ

Kubernetes Federation (KubeFed) คือระบบที่ช่วยให้คุณจัดการหลาย Kubernetes clusters จาก control plane เดียว ช่วยให้คุณสามารถ deploy resources ไปยังหลาย clusters พร้อมกัน และจัดการ placement, overrides, และ status ของ federated resources

---

## 87.1 KubeFed Architecture

### Components

```
KubeFed Control Plane (Host Cluster)
├── KubeFed Controller Manager
├── Federated Resource Types (CRDs)
│   ├── FederatedDeployment
│   ├── FederatedService
│   ├── FederatedConfigMap
│   └── ...
├── KubeFedCluster (member clusters)
└── FederatedTypeConfig (type configuration)

Member Clusters:
├── Cluster 1 (East)
├── Cluster 2 (West)  
└── Cluster 3 (EU)
```

### KubeFed vs Raw Multi-cluster

```
Raw Multi-cluster (ที่เราเรียนใน Part 86):
- ต้อง deploy resources ไปแต่ละ cluster แยกกัน
- ไม่มี centralized state
- ต้องจัดการ divergence เอง

KubeFed:
- Deploy ครั้งเดียว กระจายไปทุก clusters
- Placement policy (เลือกว่า cluster ไหนได้รับ resource)
- Override policy (custom values ต่างกันในแต่ละ cluster)
- Status aggregation จากทุก clusters
```

---

## 87.2 ติดตั้ง KubeFed

### Requirements

```bash
# ต้องมี:
# - Kubernetes clusters อย่างน้อย 2 ตัว
# - kubectl กำหนดค่าสำหรับทุก clusters
# - Helm 3

# ตรวจสอบ contexts
kubectl config get-contexts
```

### ติดตั้งด้วย Helm

```bash
# เพิ่ม KubeFed Helm repo
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts

# ติดตั้ง KubeFed ใน host cluster
kubectl config use-context host-cluster

helm install kubefed kubefed-charts/kubefed \
  --namespace kube-federation-system \
  --create-namespace \
  --version=0.10.0 \
  --set controllermanager.replicaCount=2

# รอให้ Ready
kubectl wait --for=condition=ready pod \
  -l control-plane=controller-manager \
  -n kube-federation-system \
  --timeout=300s

# ตรวจสอบ
kubectl get pods -n kube-federation-system
kubectl get crd | grep kubefed
```

### ติดตั้ง kubefedctl

```bash
# ดาวน์โหลด kubefedctl
curl -LO https://github.com/kubernetes-sigs/kubefed/releases/download/v0.10.0/kubefedctl-0.10.0-linux-amd64.tgz
tar -xzf kubefedctl-0.10.0-linux-amd64.tgz
sudo mv kubefedctl /usr/local/bin/
chmod +x /usr/local/bin/kubefedctl

# ตรวจสอบ
kubefedctl version
```

---

## 87.3 Join Member Clusters

```bash
# Join cluster1
kubefedctl join cluster1 \
  --cluster-context=cluster1 \
  --host-cluster-context=host-cluster \
  --v=2

# Join cluster2
kubefedctl join cluster2 \
  --cluster-context=cluster2 \
  --host-cluster-context=host-cluster \
  --v=2

# Join cluster3
kubefedctl join cluster3 \
  --cluster-context=cluster3 \
  --host-cluster-context=host-cluster \
  --v=2

# ดูรายการ clusters
kubectl get kubefedcluster -n kube-federation-system
kubectl describe kubefedcluster cluster1 -n kube-federation-system

# ตรวจสอบ cluster health
kubectl get kubefedcluster -n kube-federation-system \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[0].status,REASON:.status.conditions[0].reason'
```

---

## 87.4 Federated Types

### เปิดใช้ Federated Types

```bash
# เปิดใช้ federation สำหรับ built-in types
kubefedctl enable deployments.apps --host-cluster-context=host-cluster
kubefedctl enable services --host-cluster-context=host-cluster
kubefedctl enable configmaps --host-cluster-context=host-cluster
kubefedctl enable namespaces --host-cluster-context=host-cluster
kubefedctl enable secrets --host-cluster-context=host-cluster
kubefedctl enable serviceaccounts --host-cluster-context=host-cluster

# ดู enabled types
kubectl get federatedtypeconfigs -n kube-federation-system
```

### FederatedTypeConfig

```yaml
# federatedtypeconfig-deployment.yaml (auto-generated)
apiVersion: core.kubefed.io/v1beta1
kind: FederatedTypeConfig
metadata:
  name: deployments.apps
  namespace: kube-federation-system
spec:
  federatedType:
    group: types.kubefed.io
    kind: FederatedDeployment
    pluralName: federateddeployments
    scope: Namespaced
    version: v1beta1
  propagation: Enabled
  statusCollection: Enabled
  targetType:
    group: apps
    kind: Deployment
    pluralName: deployments
    scope: Namespaced
    version: v1
```

---

## 87.5 Federated Resources

### FederatedNamespace

```yaml
# federated-namespace.yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: production
  namespace: production
spec:
  template:
    metadata:
      labels:
        env: production
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
      - name: cluster3
```

```bash
# สร้าง federated namespace
kubectl apply -f federated-namespace.yaml

# ตรวจสอบว่า namespace ถูกสร้างใน member clusters
kubectl get namespace production --context=cluster1
kubectl get namespace production --context=cluster2
kubectl get namespace production --context=cluster3
```

### FederatedDeployment

```yaml
# federated-deployment.yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: my-app
  namespace: production
spec:
  # Template: ใช้ทุก clusters เป็น default
  template:
    metadata:
      labels:
        app: my-app
        version: "1.5.0"
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
            - name: app
              image: myregistry/myapp:1.5.0
              ports:
                - containerPort: 8080
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
              env:
                - name: APP_ENV
                  value: production
  
  # Placement: เลือก clusters ที่จะ deploy
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
      - name: cluster3
  
  # Overrides: ค่าที่แตกต่างในแต่ละ cluster
  overrides:
    # cluster1 (East) - รับ traffic มากกว่า
    - clusterName: cluster1
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5
        - path: "/spec/template/spec/containers/0/env"
          value:
            - name: APP_ENV
              value: production
            - name: REGION
              value: us-east-1
    
    # cluster2 (West)
    - clusterName: cluster2
      clusterOverrides:
        - path: "/spec/replicas"
          value: 3
        - path: "/spec/template/spec/containers/0/env"
          value:
            - name: APP_ENV
              value: production
            - name: REGION
              value: us-west-2
    
    # cluster3 (EU)
    - clusterName: cluster3
      clusterOverrides:
        - path: "/spec/replicas"
          value: 2
        - path: "/spec/template/spec/containers/0/env"
          value:
            - name: APP_ENV
              value: production
            - name: REGION
              value: eu-central-1
```

```bash
# Apply federated deployment
kubectl apply -f federated-deployment.yaml

# ตรวจสอบ status
kubectl get federateddeployment my-app -n production -o yaml

# ดู deployment ใน member clusters
kubectl get deployment my-app -n production --context=cluster1
kubectl get deployment my-app -n production --context=cluster2
kubectl get deployment my-app -n production --context=cluster3
```

### FederatedService

```yaml
# federated-service.yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedService
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      selector:
        app: my-app
      ports:
        - name: http
          port: 80
          targetPort: 8080
      type: LoadBalancer
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
      - name: cluster3
```

### FederatedConfigMap

```yaml
# federated-configmap.yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedConfigMap
metadata:
  name: app-config
  namespace: production
spec:
  template:
    data:
      app.properties: |
        log.level=info
        max.connections=100
        timeout=30
  placement:
    clusters:
      - name: cluster1
      - name: cluster2
      - name: cluster3
  overrides:
    - clusterName: cluster1
      clusterOverrides:
        - path: "/data/app.properties"
          value: |
            log.level=info
            max.connections=200
            timeout=30
            region=us-east-1
    - clusterName: cluster3
      clusterOverrides:
        - path: "/data/app.properties"
          value: |
            log.level=warn
            max.connections=100
            timeout=30
            region=eu-central-1
            gdpr.mode=enabled
```

---

## 87.6 Placement Policies

### ClusterSelector

```yaml
# deploy ไปยัง clusters ที่มี specific labels
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    # ... deployment template
  placement:
    clusterSelector:
      matchLabels:
        region: us  # เฉพาะ clusters ที่มี label region=us
```

```bash
# ตั้ง labels บน clusters
kubectl label kubefedcluster cluster1 region=us -n kube-federation-system
kubectl label kubefedcluster cluster2 region=us -n kube-federation-system
kubectl label kubefedcluster cluster3 region=eu -n kube-federation-system
```

### ReplicaSchedulingPreference

```yaml
# กระจาย replicas ตาม preference
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: my-app
  namespace: production
spec:
  targetKind: FederatedDeployment
  totalReplicas: 10
  clusters:
    cluster1:
      weight: 5         # 50% = 5 replicas
      minReplicas: 2
      maxReplicas: 7
    cluster2:
      weight: 3         # 30% = 3 replicas
      minReplicas: 1
      maxReplicas: 5
    cluster3:
      weight: 2         # 20% = 2 replicas
      minReplicas: 1
      maxReplicas: 3
  rebalance: true        # auto-rebalance เมื่อ cluster ล้มเหลว
```

---

## 87.7 Status Aggregation

```bash
# ดู status รวมจากทุก clusters
kubectl get federateddeployment my-app -n production -o yaml

# Status จะแสดง:
# status:
#   clusters:
#     - name: cluster1
#       status:
#         availableReplicas: 5
#         readyReplicas: 5
#         replicas: 5
#     - name: cluster2
#       status:
#         availableReplicas: 3
#         readyReplicas: 3
#         replicas: 3
#     - name: cluster3
#       status:
#         availableReplicas: 2
#         readyReplicas: 2
#         replicas: 2
```

---

## 87.8 Workshop: Deploy Federated Application

### เป้าหมาย
Deploy สมบูรณ์ขั้นตอนสำหรับ E-commerce Application ไปยัง 3 clusters

### ขั้นตอนที่ 1: Setup Environment

```bash
# ใช้ kind สำหรับ workshop
# สร้าง host cluster
kind create cluster --name host

# สร้าง member clusters
kind create cluster --name east
kind create cluster --name west
kind create cluster --name eu

# ติดตั้ง KubeFed ใน host
kubectl config use-context kind-host

helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm install kubefed kubefed-charts/kubefed \
  --namespace kube-federation-system \
  --create-namespace

# Join member clusters
for cluster in east west eu; do
  kubefedctl join ${cluster} \
    --cluster-context=kind-${cluster} \
    --host-cluster-context=kind-host
done

# ดูรายการ clusters
kubectl get kubefedcluster -n kube-federation-system
```

### ขั้นตอนที่ 2: Enable Federated Types

```bash
# Enable types ที่ต้องการ
for type in namespaces deployments.apps services configmaps secrets; do
  kubefedctl enable ${type} --host-cluster-context=kind-host
done

# Label clusters
kubectl label kubefedcluster east \
  region=us-east \
  tier=production \
  -n kube-federation-system

kubectl label kubefedcluster west \
  region=us-west \
  tier=production \
  -n kube-federation-system

kubectl label kubefedcluster eu \
  region=eu-central \
  tier=production \
  gdpr=required \
  -n kube-federation-system
```

### ขั้นตอนที่ 3: Deploy E-commerce App

```yaml
# ecommerce-federated.yaml
---
# Federated Namespace
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: ecommerce
  namespace: ecommerce
spec:
  template:
    metadata:
      labels:
        app: ecommerce
        managed-by: kubefed
  placement:
    clusters:
      - name: east
      - name: west
      - name: eu

---
# Federated ConfigMap
apiVersion: types.kubefed.io/v1beta1
kind: FederatedConfigMap
metadata:
  name: ecommerce-config
  namespace: ecommerce
spec:
  template:
    data:
      config.json: |
        {
          "log_level": "info",
          "cache_ttl": 300,
          "max_connections": 100
        }
  placement:
    clusters:
      - name: east
      - name: west
      - name: eu
  overrides:
    - clusterName: eu
      clusterOverrides:
        - path: "/data/config.json"
          value: |
            {
              "log_level": "info",
              "cache_ttl": 300,
              "max_connections": 100,
              "gdpr_mode": true,
              "data_retention_days": 30
            }

---
# Federated Deployment - Frontend
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  template:
    metadata:
      labels:
        app: ecommerce
        component: frontend
    spec:
      replicas: 3
      selector:
        matchLabels:
          component: frontend
      template:
        metadata:
          labels:
            component: frontend
        spec:
          containers:
            - name: frontend
              image: myregistry/ecommerce-frontend:1.0.0
              ports:
                - containerPort: 3000
              env:
                - name: NODE_ENV
                  value: production
                - name: API_URL
                  value: http://backend:8080
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
  placement:
    clusters:
      - name: east
      - name: west
      - name: eu
  overrides:
    - clusterName: east
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5  # US East traffic มากกว่า

---
# Federated Deployment - Backend
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: backend
  namespace: ecommerce
spec:
  template:
    metadata:
      labels:
        app: ecommerce
        component: backend
    spec:
      replicas: 2
      selector:
        matchLabels:
          component: backend
      template:
        metadata:
          labels:
            component: backend
        spec:
          containers:
            - name: backend
              image: myregistry/ecommerce-backend:1.0.0
              ports:
                - containerPort: 8080
              envFrom:
                - configMapRef:
                    name: ecommerce-config
              resources:
                requests:
                  cpu: "200m"
                  memory: "256Mi"
                limits:
                  cpu: "1000m"
                  memory: "1Gi"
  placement:
    clusters:
      - name: east
      - name: west
      - name: eu

---
# Federated Services
apiVersion: types.kubefed.io/v1beta1
kind: FederatedService
metadata:
  name: frontend
  namespace: ecommerce
spec:
  template:
    spec:
      selector:
        component: frontend
      ports:
        - name: http
          port: 80
          targetPort: 3000
      type: LoadBalancer
  placement:
    clusters:
      - name: east
      - name: west
      - name: eu

---
apiVersion: types.kubefed.io/v1beta1
kind: FederatedService
metadata:
  name: backend
  namespace: ecommerce
spec:
  template:
    spec:
      selector:
        component: backend
      ports:
        - name: http
          port: 8080
  placement:
    clusters:
      - name: east
      - name: west
      - name: eu
```

```bash
# สร้าง namespace ก่อน
kubectl create namespace ecommerce

# Apply
kubectl apply -f ecommerce-federated.yaml

# ดู status
kubectl get federateddeployment -n ecommerce
kubectl get federatedservice -n ecommerce

# ดู deployments ใน member clusters
for cluster in east west eu; do
  echo "=== Cluster: ${cluster} ==="
  kubectl get deployment -n ecommerce --context=kind-${cluster}
done
```

### ขั้นตอนที่ 4: ทดสอบ Failover

```bash
# Simulate cluster failure - ลบ east cluster
kind delete cluster --name east

# ดูว่า KubeFed อัปเดต status
kubectl get kubefedcluster -n kube-federation-system

# ดู ReplicaSchedulingPreference rebalance
kubectl get federateddeployment frontend -n ecommerce -o yaml | grep -A 20 status

# สังเกตว่า traffic ถูก route ไป west และ eu แทน

# Restore east cluster
kind create cluster --name east
kubefedctl join east \
  --cluster-context=kind-east \
  --host-cluster-context=kind-host

# Reconcile
kubectl rollout restart deployment/kubefed-controller-manager -n kube-federation-system
```

### ขั้นตอนที่ 5: Update ทุก Clusters พร้อมกัน

```bash
# Update image ใน federated deployment
kubectl patch federateddeployment frontend -n ecommerce \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/template/spec/containers/0/image","value":"myregistry/ecommerce-frontend:1.1.0"}]'

# ดู rollout progress ใน ทุก clusters
for cluster in east west eu; do
  echo "=== Cluster: ${cluster} ==="
  kubectl rollout status deployment/frontend -n ecommerce --context=kind-${cluster}
done
```

---

## 87.9 KubeFed vs Alternatives

### เปรียบเทียบ

```
KubeFed:
✅ Centralized management
✅ Placement policies
✅ Override per cluster
✅ Status aggregation
❌ Complex setup
❌ Limited active development
❌ Not production-ready for all use cases

Argo CD ApplicationSet (ดูใน Part 86):
✅ GitOps native
✅ Better tooling
✅ Active development
✅ Easier to understand
❌ No automatic failover
❌ Manual placement management

Fleet (Rancher):
✅ Simplified multi-cluster
✅ GitOps based
✅ Good for Rancher users
❌ Rancher-specific

Crossplane:
✅ Infrastructure as Code
✅ Provider-agnostic
✅ Active development
❌ Steeper learning curve
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **KubeFed Architecture**: Host cluster, member clusters, federated types
2. **ติดตั้ง KubeFed**: Helm, kubefedctl
3. **Federated Resources**: Namespace, Deployment, Service, ConfigMap
4. **Placement Policies**: clusterSelector, labels
5. **ReplicaSchedulingPreference**: กระจาย replicas ตาม weight
6. **Workshop**: E-commerce app deployment ไปหลาย clusters
7. **Alternatives**: เปรียบเทียบกับ Argo CD, Fleet, Crossplane

บทถัดไปเราจะเรียนรู้เกี่ยวกับ Cluster API (CAPI)
