# Part 64: Prometheus - Metrics Monitoring

## Prometheus Architecture

Prometheus เป็นระบบ Monitoring และ Alerting แบบ Open Source ที่ออกแบบมาสำหรับ Cloud Native Applications โดยเฉพาะ Kubernetes Prometheus ถูกพัฒนาโดย SoundCloud และปัจจุบันเป็น CNCF Graduated Project

### หลักการทำงานของ Prometheus

Prometheus ใช้ **Pull Model** หมายความว่า Prometheus จะไป Scrape Metrics จาก Targets ที่กำหนดไว้เป็นระยะๆ แทนที่จะรอให้ Targets ส่ง Metrics มาให้ (Push Model)

```
+------------------+     HTTP Scrape     +--------------------+
|   Target 1       | <------------------ |                    |
| (app:8080/metrics)|                    |                    |
+------------------+                    |    Prometheus      |
                                         |    Server          |
+------------------+     HTTP Scrape     |                    |
|   Target 2       | <------------------ |  - Scrape Engine   |
| (node:9100/metrics)|                   |  - TSDB Storage    |
+------------------+                    |  - PromQL Engine   |
                                         |  - HTTP API        |
+------------------+     HTTP Scrape     |  - Alerting        |
|   Kubernetes     | <------------------ |                    |
| (kube-state)     |                     +--------------------+
+------------------+                            |
                                                 | Query / Alert
                                         +-------v-------+
                                         |   Grafana     |
                                         | AlertManager  |
                                         +---------------+
```

### Components ของ Prometheus

| Component | หน้าที่ |
|-----------|---------|
| **Prometheus Server** | Core Engine ทำหน้าที่ Scrape, Store, และ Query Metrics |
| **TSDB (Time Series DB)** | Database สำหรับเก็บ Metrics แบบ Time Series |
| **PromQL** | Query Language สำหรับ Query Metrics |
| **Alertmanager** | จัดการ Alerts และส่ง Notifications |
| **Pushgateway** | รับ Metrics จาก Short-lived Jobs |
| **Exporters** | แปลง Metrics จากระบบต่างๆ ให้ Prometheus เข้าใจ |

### Data Model ของ Prometheus

```
# Format: metric_name{label1="value1", label2="value2"} value timestamp

# ตัวอย่าง Metrics
http_requests_total{method="GET", endpoint="/api", status="200"} 1234 1705300000000
http_requests_total{method="POST", endpoint="/api", status="500"} 5 1705300000000
node_memory_MemAvailable_bytes{instance="node-1:9100"} 4294967296
container_cpu_usage_seconds_total{pod="nginx-abc", container="nginx"} 0.5
```

### Metric Types

```
# 1. Counter - ค่าที่เพิ่มขึ้นเรื่อยๆ (ไม่ลดลง)
http_requests_total{method="GET"} 12345

# 2. Gauge - ค่าที่ขึ้นลงได้
node_memory_MemFree_bytes 4294967296
container_memory_usage_bytes 134217728

# 3. Histogram - กระจาย Distribution
http_request_duration_seconds_bucket{le="0.05"} 100
http_request_duration_seconds_bucket{le="0.1"} 250
http_request_duration_seconds_bucket{le="0.5"} 500
http_request_duration_seconds_sum 150.5
http_request_duration_seconds_count 500

# 4. Summary - คล้าย Histogram แต่คำนวณ Quantile ที่ Client
rpc_duration_seconds{quantile="0.5"} 0.05
rpc_duration_seconds{quantile="0.9"} 0.1
rpc_duration_seconds{quantile="0.99"} 0.2
rpc_duration_seconds_sum 5000
rpc_duration_seconds_count 100000
```

---

## ติดตั้ง Prometheus ด้วย Helm

### ติดตั้ง kube-prometheus-stack

`kube-prometheus-stack` เป็น Helm Chart ที่รวม Prometheus, Grafana, AlertManager และ Exporters ต่างๆ ไว้ในที่เดียว

```bash
# เพิ่ม Helm Repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# สร้าง Namespace
kubectl create namespace monitoring

# ดู Values ที่มี
helm show values prometheus-community/kube-prometheus-stack > prometheus-values.yaml

# ติดตั้งแบบ Quick (สำหรับ Development)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false

# ตรวจสอบ Status
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### Custom Values สำหรับ Production

```yaml
# prometheus-production-values.yaml
prometheus:
  prometheusSpec:
    # Retention Policy
    retention: 15d
    retentionSize: "50GB"
    
    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 50Gi
    
    # Resource Limits
    resources:
      requests:
        memory: 1Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2000m
    
    # Scrape Config
    scrapeInterval: 30s
    evaluationInterval: 30s
    
    # Additional Scrape Configs
    additionalScrapeConfigs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: standard
          resources:
            requests:
              storage: 10Gi
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 500m

grafana:
  adminPassword: "your-secure-password"
  persistence:
    enabled: true
    size: 10Gi
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 1Gi
      cpu: 500m

# Enable all Node Exporters
nodeExporter:
  enabled: true

# Enable kube-state-metrics
kubeStateMetrics:
  enabled: true

# Enable Kubernetes Component Monitoring
kubelet:
  enabled: true
  
kubeApiServer:
  enabled: true

kubeControllerManager:
  enabled: true

kubeScheduler:
  enabled: true

kubeEtcd:
  enabled: true
```

```bash
# ติดตั้งด้วย Custom Values
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus-production-values.yaml

# ดู Status ของ All Components
kubectl get pods -n monitoring -w

# เข้าถึง Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
echo "เปิด Prometheus ที่ http://localhost:9090"

# เข้าถึง Grafana UI
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &
echo "เปิด Grafana ที่ http://localhost:3000 (admin/prom-operator)"

# เข้าถึง AlertManager UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093 &
echo "เปิด AlertManager ที่ http://localhost:9093"
```

### ติดตั้ง Prometheus แบบ Manual (ไม่ใช้ Helm)

```yaml
# prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: 'my-kubernetes-cluster'
        env: 'production'

    rule_files:
      - /etc/prometheus/rules/*.yml

    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - alertmanager:9093

    scrape_configs:
      # Scrape Prometheus itself
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      # Scrape Kubernetes API Server
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

      # Scrape Kubernetes Nodes
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)

      # Scrape Pods
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: kubernetes_pod_name

      # Scrape Services
      - job_name: 'kubernetes-services'
        kubernetes_sd_configs:
          - role: service
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
```

---

## PromQL

PromQL (Prometheus Query Language) เป็น Query Language สำหรับ Query และ Aggregate Metrics จาก Prometheus

### PromQL Basics

```promql
# ดู Metric ทั้งหมด
http_requests_total

# ดู Metric พร้อม Label Filter
http_requests_total{method="GET"}
http_requests_total{status="200"}
http_requests_total{method="GET", status="200"}

# ใช้ Regex Matching
http_requests_total{status=~"2.."} # 2xx Status Codes
http_requests_total{status!~"2.."} # ไม่ใช่ 2xx Status Codes

# ดูเฉพาะ Label
{__name__=~"http_.*"} # Metrics ที่ขึ้นต้นด้วย http_
```

### Rate และ Increase

```promql
# Rate: อัตราการเปลี่ยนแปลงต่อวินาที (ใช้กับ Counter)
rate(http_requests_total[5m])

# irate: Instant Rate (ใช้ 2 Points ล่าสุด)
irate(http_requests_total[5m])

# Increase: จำนวนที่เพิ่มขึ้นใน Time Range
increase(http_requests_total[1h])

# ตัวอย่าง: Request Rate ใน 5 นาที
rate(http_requests_total[5m])

# ตัวอย่าง: Error Rate
rate(http_requests_total{status=~"5.."}[5m])
```

### Aggregation Functions

```promql
# sum: รวมทั้งหมด
sum(http_requests_total)

# sum by: รวมตาม Label
sum(http_requests_total) by (method)
sum(http_requests_total) by (method, status)

# avg: ค่าเฉลี่ย
avg(container_memory_usage_bytes)

# max: ค่าสูงสุด
max(container_memory_usage_bytes) by (pod)

# min: ค่าต่ำสุด
min(container_memory_usage_bytes) by (pod)

# count: นับจำนวน
count(http_requests_total)

# topk: Top K Values
topk(5, rate(http_requests_total[5m]))

# bottomk: Bottom K Values
bottomk(5, container_cpu_usage_seconds_total)
```

### Math Operations

```promql
# การคำนวณพื้นฐาน
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# เปอร์เซ็นต์
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# CPU Usage Percentage
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Useful PromQL Queries สำหรับ Kubernetes

```promql
# === Node Metrics ===

# CPU Usage ของแต่ละ Node (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Usage ของแต่ละ Node (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk Usage ของแต่ละ Node (%)
(1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100

# Network I/O
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# === Pod Metrics ===

# CPU Usage ของ Pod
rate(container_cpu_usage_seconds_total{container!=""}[5m])

# Memory Usage ของ Pod
container_memory_usage_bytes{container!=""}

# Memory Usage vs Limit
container_memory_usage_bytes / container_spec_memory_limit_bytes

# Pod Restart Count
kube_pod_container_status_restarts_total

# Pods ที่ไม่ Ready
kube_pod_status_ready{condition="false"}

# === Deployment Metrics ===

# จำนวน Replicas ที่ต้องการ vs จำนวนที่ Ready
kube_deployment_spec_replicas - kube_deployment_status_ready_replicas

# Pods ที่ Unavailable
kube_deployment_status_replicas_unavailable

# === API Server Metrics ===

# Request Rate ไปยัง API Server
rate(apiserver_request_total[5m])

# Request Latency
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))

# Error Rate
rate(apiserver_request_total{code=~"5.."}[5m])

# === Application Metrics ===

# HTTP Request Rate
rate(http_requests_total[5m])

# HTTP Error Rate (5xx)
rate(http_requests_total{status=~"5.."}[5m])

# HTTP Latency P99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Service Availability
1 - (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))
```

---

## ServiceMonitor

ServiceMonitor เป็น Custom Resource ที่ Prometheus Operator ใช้เพื่อกำหนดว่าจะ Scrape Metrics จาก Service ไหน

### ทำไมต้องใช้ ServiceMonitor

แทนที่จะแก้ไข Prometheus Config โดยตรง ServiceMonitor ให้เราสร้าง Configuration เป็น Kubernetes Resource

### ตัวอย่าง ServiceMonitor

```yaml
# app-service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: prometheus  # ต้อง Match กับ Prometheus Selector
spec:
  namespaceSelector:
    matchNames:
      - default
      - production
  selector:
    matchLabels:
      app: my-app
      metrics: "true"
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
      # เพิ่ม Labels จาก Service
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
      # แปลง Metrics Names
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: '(.*)'
          targetLabel: __name__
          replacement: '${1}'
```

### Application พร้อม Metrics Endpoint

```yaml
# metrics-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-app
  namespace: default
  labels:
    app: metrics-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: app
          image: prom/prometheus:v2.48.0  # ใช้ Prometheus เป็น Example App
          ports:
            - name: metrics
              containerPort: 9090
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
  name: metrics-app
  namespace: default
  labels:
    app: metrics-app
    metrics: "true"  # Label สำหรับ ServiceMonitor Selector
spec:
  selector:
    app: metrics-app
  ports:
    - name: metrics
      port: 9090
      targetPort: 9090
```

### Node Exporter สำหรับ Node Metrics

```yaml
# node-exporter-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --no-collector.wifi
            - --no-collector.hwmon
            - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)
            - --collector.netclass.ignored-devices=^(veth.*)$
          ports:
            - containerPort: 9100
              hostPort: 9100
              name: metrics
          volumeMounts:
            - name: sys
              mountPath: /host/sys
              mountPropagation: HostToContainer
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 100Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
      volumes:
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
```

```yaml
# node-exporter-service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    release: prometheus
spec:
  jobLabel: node-exporter
  selector:
    matchLabels:
      app: node-exporter
  endpoints:
    - port: metrics
      interval: 30s
      scheme: http
```

---

## Workshop: Monitor Kubernetes Cluster

### เป้าหมาย

ในส่วนนี้เราจะ:
1. ติดตั้ง Prometheus ด้วย Helm
2. Deploy Application พร้อม Metrics
3. สร้าง ServiceMonitor
4. เขียน PromQL Queries
5. ดู Metrics ใน Prometheus UI

### ขั้นตอนที่ 1: ติดตั้ง kube-prometheus-stack

```bash
# เพิ่ม Helm Repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# สร้าง Namespace
kubectl create namespace monitoring

# ติดตั้ง kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.retention=7d \
  --set grafana.adminPassword=admin123

# รอให้ Pods Ready
kubectl wait --for=condition=Ready pods --all -n monitoring --timeout=300s

# ดู Status
kubectl get pods -n monitoring
```

### ขั้นตอนที่ 2: Deploy Sample Application

```yaml
# sample-app-with-metrics.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-metrics-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-metrics-app
  template:
    metadata:
      labels:
        app: sample-metrics-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2112"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: app
          # ใช้ prom/statsd-exporter เป็น Example Application ที่มี Metrics
          image: prom/statsd-exporter:v0.26.0
          ports:
            - name: metrics
              containerPort: 9102
            - name: statsd-udp
              containerPort: 9125
              protocol: UDP
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: sample-metrics-app
  namespace: default
  labels:
    app: sample-metrics-app
    monitoring: "true"
spec:
  selector:
    app: sample-metrics-app
  ports:
    - name: metrics
      port: 9102
      targetPort: 9102
```

```bash
# Apply Application
kubectl apply -f sample-app-with-metrics.yaml

# ตรวจสอบ
kubectl get pods -l app=sample-metrics-app
kubectl get svc sample-metrics-app
```

### ขั้นตอนที่ 3: สร้าง ServiceMonitor

```yaml
# sample-app-service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-metrics-app
  namespace: monitoring
  labels:
    release: prometheus  # ต้อง Match กับ Prometheus Helm Release Name
spec:
  namespaceSelector:
    matchNames:
      - default
  selector:
    matchLabels:
      monitoring: "true"
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

```bash
# Apply ServiceMonitor
kubectl apply -f sample-app-service-monitor.yaml

# ตรวจสอบ ServiceMonitor
kubectl get servicemonitor -n monitoring

# รอ 30-60 วินาที แล้วตรวจสอบใน Prometheus UI
```

### ขั้นตอนที่ 4: เข้าถึง Prometheus UI

```bash
# Port Forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
echo "Prometheus: http://localhost:9090"

# ทดสอบ PromQL Queries
curl 'http://localhost:9090/api/v1/query?query=up' | python3 -m json.tool
```

### ขั้นตอนที่ 5: PromQL Workshop

```bash
# เปิด Prometheus UI ที่ http://localhost:9090/graph
# ลอง Queries ต่อไปนี้:

# 1. ดู Targets ที่กำลัง Scrape
# ไปที่: Status > Targets

# 2. Queries พื้นฐาน
up

# 3. Node CPU Usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 4. Node Memory Usage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 5. Pod CPU Usage
rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[5m])

# 6. Pod Memory Usage
container_memory_working_set_bytes{container!="",container!="POD"}

# 7. Kubernetes Pod Status
kube_pod_status_phase

# 8. Kubernetes Deployment Status
kube_deployment_status_replicas_available
kube_deployment_status_replicas_unavailable

# 9. API Server Request Rate
rate(apiserver_request_total[5m])

# 10. API Server Latency P99
histogram_quantile(0.99, 
  rate(apiserver_request_duration_seconds_bucket[5m]))
```

### ขั้นตอนที่ 6: สร้าง Recording Rules

Recording Rules ช่วยลด Query Time โดย Pre-compute Metrics ที่ใช้บ่อยๆ

```yaml
# prometheus-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-recording-rules
  namespace: monitoring
  labels:
    release: prometheus
    role: alert-rules
spec:
  groups:
    - name: kubernetes.recording_rules
      interval: 30s
      rules:
        # Pre-compute CPU Usage per Node
        - record: node:node_cpu_utilisation:avg1m
          expr: |
            1 - avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[1m])
            )

        # Pre-compute Memory Usage per Node
        - record: node:node_memory_utilisation:
          expr: |
            1 - (
              node_memory_MemAvailable_bytes /
              node_memory_MemTotal_bytes
            )

        # Pre-compute Pod CPU Usage
        - record: pod:container_cpu_usage:sum
          expr: |
            sum by (namespace, pod) (
              rate(container_cpu_usage_seconds_total{
                container!="",
                container!="POD"
              }[5m])
            )

        # Pre-compute Pod Memory Usage
        - record: pod:container_memory_usage:sum
          expr: |
            sum by (namespace, pod) (
              container_memory_working_set_bytes{
                container!="",
                container!="POD"
              }
            )

        # Pre-compute HTTP Error Rate
        - record: job:http_requests_errors:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
```

```bash
# Apply Recording Rules
kubectl apply -f prometheus-recording-rules.yaml

# ตรวจสอบ Rules ใน Prometheus UI
# ไปที่: Status > Rules
```

### ขั้นตอนที่ 7: ใช้ Recording Rules ใน Queries

```promql
# ใช้ Recording Rules ที่สร้างไว้
node:node_cpu_utilisation:avg1m
node:node_memory_utilisation:

pod:container_cpu_usage:sum
pod:container_memory_usage:sum
```

### ขั้นตอนที่ 8: Prometheus Federation

```yaml
# prometheus-federation-config.yaml
# สำหรับ Multi-cluster Setup
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'
    static_configs:
      - targets:
          - 'prometheus-cluster-1:9090'
          - 'prometheus-cluster-2:9090'
```

### ขั้นตอนที่ 9: Long-term Storage ด้วย Thanos

```yaml
# thanos-sidecar.yaml
# เพิ่ม Thanos Sidecar ให้กับ Prometheus
spec:
  containers:
    - name: thanos-sidecar
      image: quay.io/thanos/thanos:v0.33.0
      args:
        - sidecar
        - --tsdb.path=/prometheus
        - --prometheus.url=http://localhost:9090
        - --grpc-address=0.0.0.0:10901
        - --http-address=0.0.0.0:10902
        - --objstore.config-file=/etc/thanos/objstore.yaml
      volumeMounts:
        - name: prometheus-data
          mountPath: /prometheus
          readOnly: false
        - name: objstore-config
          mountPath: /etc/thanos
```

### ขั้นตอนที่ 10: ทำความสะอาด

```bash
# ลบ Sample Application
kubectl delete -f sample-app-with-metrics.yaml
kubectl delete -f sample-app-service-monitor.yaml
kubectl delete -f prometheus-recording-rules.yaml

# ลบ Prometheus Stack (ถ้าต้องการ)
# helm uninstall prometheus -n monitoring
# kubectl delete namespace monitoring

echo "Workshop เสร็จสิ้น!"
```

---

## Prometheus Best Practices

### 1. Naming Conventions

```
# Format: namespace_subsystem_name_unit
http_requests_total           # Counter (ต้อง _total suffix)
http_request_duration_seconds # Histogram
node_memory_free_bytes        # Gauge (bytes unit)
process_cpu_seconds_total     # Counter

# อย่าใช้:
my_metric                     # ไม่ชัดเจน
my_metric_count               # ใช้ _total แทน
```

### 2. Label Best Practices

```promql
# ใช้ Labels ที่มี Cardinality ต่ำ
http_requests_total{method="GET", status="200"}  # ดี
http_requests_total{user_id="12345"}             # ไม่ดี (Cardinality สูง)

# High Cardinality Labels ทำให้ Prometheus ใช้ Memory มาก
# หลีกเลี่ยง: user_id, email, transaction_id
```

### 3. Retention และ Storage

```bash
# ตั้งค่า Retention ที่เหมาะสม
--storage.tsdb.retention.time=15d    # 15 วัน
--storage.tsdb.retention.size=50GB   # 50 GB

# Monitor Storage Usage
node_filesystem_free_bytes{mountpoint="/prometheus"}
```

---

## สรุป

Prometheus เป็นระบบ Monitoring ที่ขาดไม่ได้สำหรับ Kubernetes:

1. **Pull Model**: Prometheus Scrape Metrics จาก Targets
2. **PromQL**: Query Language ที่ทรงพลัง
3. **ServiceMonitor**: Kubernetes-native way ในการกำหนด Scrape Targets
4. **kube-prometheus-stack**: Bundle ที่ครบถ้วนสำหรับ Kubernetes Monitoring
5. **Recording Rules**: Pre-compute Metrics เพื่อประสิทธิภาพที่ดีขึ้น

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ Grafana ซึ่งเป็น Visualization Tool ที่ใช้คู่กับ Prometheus เพื่อสร้าง Dashboard ที่สวยงามและมีประโยชน์
