# Part 86: Multi-cluster Architectures

## บทนำ

Multi-cluster Architecture คือการใช้ Kubernetes clusters หลายตัวร่วมกันเพื่อรองรับ workloads ที่ต้องการ high availability, geographic distribution, compliance, หรือ isolation

---

## 86.1 ทำไมต้องใช้ Multi-cluster

### Use Cases

```
1. High Availability / Disaster Recovery
   - หลาย clusters ในหลาย regions
   - Traffic failover อัตโนมัติ
   - ไม่มี single point of failure

2. Geographic Distribution
   - ลด latency สำหรับ users ในแต่ละ region
   - Data residency requirements
   - Edge computing

3. Isolation
   - แยก production จาก staging/development
   - แยก teams หรือ business units
   - Security isolation ระหว่าง workloads

4. Scalability
   - หนึ่ง cluster รองรับได้แค่ ~5000 nodes
   - แบ่ง workloads ข้ามหลาย clusters

5. Compliance
   - GDPR, HIPAA requirements
   - แยก data ตาม jurisdiction
```

### Multi-cluster Patterns

```
1. Standalone Clusters
   - แต่ละ cluster ทำงานอิสระ
   - ไม่มี cross-cluster communication
   - ง่ายที่สุด

2. Shared Services
   - Central services (CI/CD, monitoring, logging)
   - workload clusters ใช้ shared services

3. Federated
   - จัดการหลาย clusters ด้วย control plane เดียว
   - Synchronized resources ข้าม clusters
   - Complex แต่ unified management

4. Active-Active
   - workloads ทำงานพร้อมกันหลาย clusters
   - Load balancing ข้าม clusters
   - Global traffic management
```

---

## 86.2 Multi-cluster Networking

### Service Discovery ข้าม Clusters

```yaml
# Cluster A: Production East
# Cluster B: Production West

# ปัญหา: Service ใน Cluster A เรียก Service ใน Cluster B ไม่ได้โดยตรง
```

### ตัวเลือก Multi-cluster Networking

```
1. VPN/Peering
   - เชื่อม networks ของแต่ละ cluster
   - ง่าย แต่ไม่ scale ดี
   - เหมาะสำหรับ small setups

2. Service Mesh (Istio Multi-cluster, Linkerd)
   - mTLS ระหว่าง clusters
   - Service discovery อัตโนมัติ
   - Traffic management ที่ละเอียด

3. Submariner
   - Open-source project จาก CNCF
   - Network connectivity ข้าม clusters
   - Service discovery

4. Admiral
   - Multi-cluster service discovery
   - Traffic management ด้วย Istio
```

### Istio Multi-cluster Setup

```bash
# Setup Istio Multi-cluster (Primary-Remote Model)

# Cluster 1: Primary (East)
# Cluster 2: Remote (West)

# 1. ติดตั้ง Istio บน Primary cluster
kubectl config use-context cluster1

cat <<EOF | istioctl install --set values.pilot.env.EXTERNAL_ISTIOD=true -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  profile: default
  values:
    pilot:
      env:
        EXTERNAL_ISTIOD: "true"
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          service:
            type: LoadBalancer
EOF

# 2. ดึง East cluster's ingress gateway IP
EAST_GW_IP=$(kubectl get svc istio-ingressgateway -n istio-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "East Gateway IP: ${EAST_GW_IP}"

# 3. สร้าง kubeconfig secret สำหรับ Remote cluster
istioctl x create-remote-secret \
  --name=cluster2 \
  --context=cluster2 | kubectl apply -f - --context=cluster1

# 4. ติดตั้ง Istio บน Remote cluster
kubectl config use-context cluster2

cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  profile: remote
  values:
    istiodRemote:
      injectionURL: https://${EAST_GW_IP}:15017/inject
    pilot:
      configSource:
        subscribedResources: {}
EOF
```

### Submariner Setup

```bash
# ติดตั้ง subctl tool
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin

# Deploy broker ใน cluster1
kubectl config use-context cluster1
subctl deploy-broker

# Join cluster1
subctl join --kubeconfig ~/.kube/config --context cluster1 \
  broker-info.subm --clusterid cluster1

# Join cluster2
subctl join --kubeconfig ~/.kube/config --context cluster2 \
  broker-info.subm --clusterid cluster2

# ทดสอบ connectivity
subctl verify --kubeconfig ~/.kube/config \
  --context cluster1 \
  --tocontext cluster2 \
  --connectivity
```

---

## 86.3 Cross-cluster Service Discovery

### Kubernetes Multi-cluster Services API

```yaml
# ServiceExport - expose service ข้าม clusters
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: my-service
  namespace: default
# No spec needed - exports the Service with same name

---
# ServiceImport - import service จาก cluster อื่น
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: my-service
  namespace: default
spec:
  type: ClusterSetIP
  ports:
    - name: http
      protocol: TCP
      port: 80
```

```bash
# Install MCS API controller (Submariner จัดการให้)
# หรือใช้ Google Cloud's implementation

# Export service จาก cluster1
kubectl config use-context cluster1
kubectl apply -f - <<EOF
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: nginx-service
  namespace: default
EOF

# ตรวจสอบว่า service ถูก import ใน cluster2
kubectl config use-context cluster2
kubectl get serviceimport nginx-service

# Service ใน cluster2 สามารถเรียก service จาก cluster1 ได้ด้วยชื่อ:
# nginx-service.default.svc.clusterset.local
```

---

## 86.4 Multi-cluster Load Balancing

### Global Traffic Management

```yaml
# Istio VirtualService สำหรับ cross-cluster traffic
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service-global
spec:
  hosts:
    - my-service.default.global
  http:
    - route:
        # 50% traffic ไป cluster1
        - destination:
            host: my-service.cluster1.global
            port:
              number: 80
          weight: 50
        # 50% traffic ไป cluster2
        - destination:
            host: my-service.cluster2.global
            port:
              number: 80
          weight: 50
```

### Global Load Balancer ด้วย ExternalDNS

```yaml
# ExternalDNS สำหรับ multi-cluster DNS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
        - name: external-dns
          image: k8s.gcr.io/external-dns/external-dns:latest
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=example.com
            - --provider=aws
            - --aws-zone-type=public
            - --registry=txt
            - --txt-owner-id=my-cluster
```

---

## 86.5 Multi-cluster GitOps

### Argo CD สำหรับ Multi-cluster

```yaml
# ลงทะเบียน Cluster ใน Argo CD
# cluster-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-west
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: cluster-west
  server: https://west.example.com
  config: |
    {
      "bearerToken": "<token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64-ca-cert>"
      }
    }
```

```bash
# เพิ่ม cluster ใน Argo CD
argocd cluster add cluster-east --name east --in-cluster
argocd cluster add cluster-west --name west
argocd cluster list

# Deploy application ไปหลาย clusters
argocd app create my-app-east \
  --repo https://github.com/myorg/myapp \
  --path kubernetes \
  --dest-server https://east.example.com \
  --dest-namespace production

argocd app create my-app-west \
  --repo https://github.com/myorg/myapp \
  --path kubernetes \
  --dest-server https://west.example.com \
  --dest-namespace production
```

### ApplicationSet สำหรับ Multi-cluster

```yaml
# applicationset.yaml - deploy ไปทุก clusters อัตโนมัติ
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
    # List generator - สร้าง application สำหรับแต่ละ cluster
    - list:
        elements:
          - cluster: production-east
            url: https://east.example.com
            region: us-east-1
          - cluster: production-west
            url: https://west.example.com
            region: us-west-2
          - cluster: production-eu
            url: https://eu.example.com
            region: eu-central-1
  
  template:
    metadata:
      name: "guestbook-{{cluster}}"
      labels:
        cluster: "{{cluster}}"
        region: "{{region}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/guestbook
        targetRevision: HEAD
        path: kubernetes/overlays/{{cluster}}
      destination:
        server: "{{url}}"
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 86.6 Multi-cluster Monitoring

### Thanos สำหรับ Multi-cluster Metrics

```yaml
# thanos-sidecar.yaml - เพิ่มใน Prometheus
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  retention: 2h
  thanos:
    image: quay.io/thanos/thanos:v0.32.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objectstorage
  externalLabels:
    cluster: cluster-east
    region: us-east-1
```

```yaml
# thanos-query.yaml - Query จากหลาย clusters
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
        - name: thanos-query
          image: quay.io/thanos/thanos:v0.32.0
          args:
            - query
            - --log.level=info
            - --query.replica-label=prometheus_replica
            # Store endpoints จากแต่ละ cluster
            - --store=thanos-sidecar-east:10901
            - --store=thanos-sidecar-west:10901
            - --store=thanos-sidecar-eu:10901
          ports:
            - containerPort: 10902
              name: http
            - containerPort: 10901
              name: grpc
```

### Loki สำหรับ Multi-cluster Logs

```yaml
# loki-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
data:
  loki.yaml: |
    auth_enabled: false
    
    server:
      http_listen_port: 3100
    
    ingester:
      lifecycler:
        ring:
          kvstore:
            store: memberlist
    
    # Multi-cluster label support
    limits_config:
      ingestion_rate_mb: 16
      ingestion_burst_size_mb: 32
      max_label_names_per_series: 30
```

---

## 86.7 Workshop: Multi-cluster Setup

### เป้าหมาย
ตั้ง 2-cluster setup พร้อม cross-cluster service discovery และ monitoring

### ขั้นตอนที่ 1: สร้าง Clusters ด้วย kind

```bash
# สร้าง cluster1 (East)
cat <<EOF | kind create cluster --name cluster1 --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.10.0.0/16"
  serviceSubnet: "10.20.0.0/16"
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# สร้าง cluster2 (West)
cat <<EOF | kind create cluster --name cluster2 --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.30.0.0/16"
  serviceSubnet: "10.40.0.0/16"
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# ดู contexts
kubectl config get-contexts
```

### ขั้นตอนที่ 2: Deploy Workloads

```bash
# Deploy application ใน cluster1
kubectl config use-context kind-cluster1
kubectl create namespace production

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    cluster: east
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        cluster: east
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
          env:
            - name: CLUSTER
              value: east
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
  ports:
    - port: 80
EOF

# Deploy application ใน cluster2
kubectl config use-context kind-cluster2
kubectl create namespace production

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    cluster: west
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        cluster: west
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
          env:
            - name: CLUSTER
              value: west
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
  ports:
    - port: 80
EOF
```

### ขั้นตอนที่ 3: ตั้ง Multi-cluster Monitoring

```bash
# ติดตั้ง Prometheus ใน cluster1
kubectl config use-context kind-cluster1

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.externalLabels.cluster=east \
  --set prometheus.prometheusSpec.externalLabels.region=us-east

# ติดตั้ง Prometheus ใน cluster2
kubectl config use-context kind-cluster2

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.externalLabels.cluster=west \
  --set prometheus.prometheusSpec.externalLabels.region=us-west

# ตรวจสอบ
kubectl get pods -n monitoring
```

### ขั้นตอนที่ 4: Cross-cluster Dashboard

```bash
# ดู cluster metrics จาก Grafana
kubectl config use-context kind-cluster1
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &

# เข้า http://localhost:3000
# ดู cluster label ใน metrics เพื่อแยก cluster1 จาก cluster2
```

### ขั้นตอนที่ 5: Multi-cluster Health Check Script

```bash
#!/bin/bash
# health-check.sh - ตรวจสอบ health ของทุก clusters

CLUSTERS=("kind-cluster1" "kind-cluster2")

echo "=== Multi-cluster Health Check ==="
echo "Timestamp: $(date)"
echo ""

for cluster in "${CLUSTERS[@]}"; do
    echo "--- Cluster: ${cluster} ---"
    kubectl config use-context ${cluster} 2>/dev/null
    
    if [ $? -ne 0 ]; then
        echo "ERROR: Cannot connect to ${cluster}"
        continue
    fi
    
    # Node status
    echo "Nodes:"
    kubectl get nodes --no-headers | awk '{print "  " $1 " - " $2}'
    
    # Pod status
    total_pods=$(kubectl get pods -A --no-headers 2>/dev/null | wc -l)
    running_pods=$(kubectl get pods -A --no-headers 2>/dev/null | grep Running | wc -l)
    echo "Pods: ${running_pods}/${total_pods} Running"
    
    # Failing pods
    failing=$(kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded \
      --no-headers 2>/dev/null | grep -v Completed)
    if [ -n "${failing}" ]; then
        echo "WARNING: Non-running pods:"
        echo "${failing}" | awk '{print "  " $1 "/" $2}'
    fi
    
    echo ""
done

echo "=== Check Complete ==="
```

---

## 86.8 Multi-cluster Security

### Cross-cluster RBAC

```yaml
# สร้าง service account สำหรับ cross-cluster access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cross-cluster-reader
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cross-cluster-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cross-cluster-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cross-cluster-reader
subjects:
  - kind: ServiceAccount
    name: cross-cluster-reader
    namespace: default
```

### Network Policies ข้าม Clusters

```yaml
# ใน cluster1 - อนุญาตให้ cluster2 เรียก service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cross-cluster
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  ingress:
    - from:
        # CIDR ของ cluster2's pods
        - ipBlock:
            cidr: 10.30.0.0/16
      ports:
        - port: 80
```

---

## 86.9 Multi-cluster Cost Management

### กลยุทธ์การลดต้นทุนใน Multi-cluster

```
1. Spot/Preemptible Instances
   - ใช้สำหรับ non-critical workloads
   - ประหยัดได้ 60-90%

2. Cluster Consolidation
   - รวม workloads ที่เหมาะสมเข้า cluster เดียวกัน
   - ลดจำนวน control planes

3. Workload Placement Optimization
   - วาง workloads ใน region ที่ถูกกว่า
   - ใช้ Reserved Instances

4. Resource Right-sizing
   - ปรับ requests/limits ให้เหมาะสม
   - ใช้ VPA สำหรับ recommendations
```

### Kubecost สำหรับ Multi-cluster Cost Analysis

```bash
# ติดตั้ง Kubecost ใน primary cluster
helm install kubecost cost-analyzer \
  --repo https://kubecost.github.io/cost-analyzer/ \
  --namespace kubecost \
  --create-namespace \
  --set prometheus.server.global.external_labels.cluster_id=cluster1 \
  --set prometheus.server.global.external_labels.region=us-east \
  --set kubecostToken=<your-token>

# Configure cross-cluster federation
# ใน Kubecost config เพิ่ม remote clusters
kubectl edit configmap kubecost-federated -n kubecost
```

```yaml
# kubecost-allocation.yaml - query cross-cluster costs
# เข้าไปที่ Kubecost UI แล้ว filter by cluster label
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubecost-federated
  namespace: kubecost
data:
  federatedStorageConfig: |
    federatedCluster: true
    primaryCluster: cluster1
    remoteReadEnabled: true
    remoteReadClusters:
      - id: cluster2
        address: http://kubecost.cluster2:9090
      - id: cluster3
        address: http://kubecost.cluster3:9090
```

---

## 86.10 Multi-cluster Disaster Recovery

### Backup Strategy

```bash
# ใช้ Velero สำหรับ cluster backup และ restore
# ติดตั้ง Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket my-velero-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-volume-snapshots=true \
  --secret-file ./credentials-velero

# สร้าง backup schedule
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces=production,staging \
  --ttl 720h  # 30 days

# สร้าง backup ทันที
velero backup create manual-backup \
  --include-namespaces production \
  --wait

# ดู backups
velero backup get
velero backup describe manual-backup
```

### Restore ไปยัง Cluster อื่น

```bash
# Restore จาก backup ใน cluster ใหม่
# (Velero ต้องชี้ไปที่ S3 bucket เดียวกัน)

velero restore create \
  --from-backup manual-backup \
  --include-namespaces production \
  --namespace-mappings production:production-restored \
  --wait

# ดู restore status
velero restore get
velero restore describe manual-backup-20240101
```

---

## 86.11 Multi-cluster Observability Dashboard

### สร้าง Cross-cluster Grafana Dashboard

```yaml
# grafana-multi-cluster.yaml
# Grafana DataSource สำหรับ Thanos
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Thanos
        type: prometheus
        url: http://thanos-query:9090
        access: proxy
        isDefault: true
        jsonData:
          timeInterval: 1m
```

```bash
# ดู Grafana dashboards
kubectl port-forward -n monitoring svc/grafana 3000:80 &

# Dashboard IDs ที่มีประโยชน์:
# 315 - Kubernetes cluster monitoring (via Prometheus)
# 1860 - Node Exporter Full
# 13770 - Kubernetes All-in-one Cluster Monitoring KR
# 7249 - Kubernetes Cluster

# Import dashboard
curl -X POST http://admin:password@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"folderId":0,"overwrite":true,"dashboard":{"id":315}}'
```

---

## 86.12 Troubleshooting Multi-cluster Issues

### Common Issues และ Solutions

```bash
# Issue 1: Cross-cluster service ไม่ทำงาน
# Solution: ตรวจสอบ network connectivity

# ตรวจสอบ pod ใน cluster1 เรียก service ใน cluster2 ได้หรือไม่
kubectl exec -n production debug-pod -- \
  curl -v http://nginx-service.default.svc.clusterset.local

# ตรวจสอบ Submariner connectivity
subctl diagnose all

# Issue 2: Metric federation ไม่ทำงาน
# Solution: ตรวจสอบ Thanos sidecars

kubectl get pods -n monitoring -l app.kubernetes.io/name=thanos-sidecar
kubectl logs -n monitoring thanos-sidecar-prometheus-0 -c thanos-sidecar

# Issue 3: Cross-cluster DNS resolution ไม่ทำงาน
# Solution: ตรวจสอบ CoreDNS configuration

kubectl get configmap -n kube-system coredns -o yaml
# ตรวจสอบว่ามี clusterset.local zone

# Issue 4: Argo CD ไม่ sync ไป cluster
# Solution: ตรวจสอบ cluster secret

argocd cluster list
argocd cluster get https://cluster2-api:6443
kubectl get secret -n argocd \
  -l argocd.argoproj.io/secret-type=cluster \
  -o jsonpath='{.items[*].metadata.name}'
```

### Multi-cluster Health Check Dashboard Script

```bash
#!/bin/bash
# multi-cluster-dashboard.sh

CLUSTERS=("context-east" "context-west" "context-eu")
NAMESPACES=("production" "staging")

print_header() {
    echo "╔════════════════════════════════════════════════════╗"
    echo "║        Multi-Cluster Health Dashboard              ║"
    echo "║        $(date '+%Y-%m-%d %H:%M:%S')                   ║"
    echo "╚════════════════════════════════════════════════════╝"
}

check_cluster() {
    local ctx=$1
    echo ""
    echo "┌─── Cluster: ${ctx} ───────────────────────────"
    
    # Check connectivity
    if ! kubectl config use-context ${ctx} &>/dev/null; then
        echo "│ ❌ Cannot connect to cluster"
        return
    fi
    
    # Node status
    local total_nodes=$(kubectl get nodes --no-headers 2>/dev/null | wc -l)
    local ready_nodes=$(kubectl get nodes --no-headers 2>/dev/null | grep " Ready" | wc -l)
    local status_icon="✅"
    [[ $ready_nodes -lt $total_nodes ]] && status_icon="⚠️"
    echo "│ ${status_icon} Nodes: ${ready_nodes}/${total_nodes} Ready"
    
    # Pod status by namespace
    for ns in "${NAMESPACES[@]}"; do
        local total=$(kubectl get pods -n ${ns} --no-headers 2>/dev/null | wc -l)
        local running=$(kubectl get pods -n ${ns} --no-headers 2>/dev/null | grep "Running" | wc -l)
        local failed=$(kubectl get pods -n ${ns} --no-headers 2>/dev/null | grep -E "Error|CrashLoopBackOff|OOMKilled" | wc -l)
        
        local ns_icon="✅"
        [[ $failed -gt 0 ]] && ns_icon="❌"
        [[ $running -lt $total && $failed -eq 0 ]] && ns_icon="⚠️"
        
        echo "│ ${ns_icon} [${ns}] Pods: ${running}/${total} Running, ${failed} Failed"
    done
    
    # Resource usage
    local cpu_usage=$(kubectl top nodes --no-headers 2>/dev/null | awk '{sum+=$3} END {print sum"%"}')
    local mem_usage=$(kubectl top nodes --no-headers 2>/dev/null | awk '{sum+=$5} END {print sum"%"}')
    echo "│ 📊 Cluster CPU: ${cpu_usage}, Memory: ${mem_usage}"
    
    echo "└──────────────────────────────────────────────────"
}

print_header
for cluster in "${CLUSTERS[@]}"; do
    check_cluster ${cluster}
done

echo ""
echo "Check complete. Press Ctrl+C to exit or wait for refresh..."
```

```bash
# Run dashboard (refresh every 30 seconds)
watch -n 30 ./multi-cluster-dashboard.sh
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Multi-cluster Patterns**: Standalone, Shared Services, Federated, Active-Active
2. **Multi-cluster Networking**: Istio, Submariner
3. **Cross-cluster Service Discovery**: MCS API
4. **Multi-cluster Load Balancing**: Global traffic management
5. **Multi-cluster GitOps**: Argo CD, ApplicationSet
6. **Multi-cluster Monitoring**: Thanos, Loki, GMP
7. **Workshop**: 2-cluster setup พร้อม monitoring
8. **Security**: Cross-cluster RBAC และ Network Policies
9. **Cost Management**: Kubecost สำหรับ multi-cluster
10. **Disaster Recovery**: Velero backup และ restore
11. **Observability**: Cross-cluster dashboards
12. **Troubleshooting**: Common issues และ solutions

บทถัดไปเราจะเรียนรู้เกี่ยวกับ Kubernetes Federation อย่างละเอียด
