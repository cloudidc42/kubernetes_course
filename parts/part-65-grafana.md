# Part 65: Grafana - Visualization Dashboard

## Grafana คืออะไร

Grafana เป็น Open Source Analytics และ Visualization Platform ที่ช่วยให้เราสร้าง Dashboard แสดงข้อมูล Metrics, Logs, และ Traces จาก Data Sources หลายประเภท เช่น Prometheus, Elasticsearch, MySQL, PostgreSQL และอื่นๆ อีกมากมาย

### ทำไม Grafana ถึงสำคัญ

| ความสามารถ | รายละเอียด |
|-----------|-----------|
| **Multi-datasource** | รองรับ Data Sources มากกว่า 50 ประเภท |
| **Beautiful Dashboards** | Visualization ที่สวยงามและ Customizable |
| **Alerting** | ตั้ง Alert จาก Dashboard ได้โดยตรง |
| **Annotations** | เพิ่ม Events ลงใน Graph |
| **Variables** | สร้าง Dynamic Dashboards |
| **Sharing** | Share Dashboard กับทีมได้ |
| **Plugins** | ขยายความสามารถด้วย Plugins |

---

## ติดตั้ง Grafana

### ติดตั้งด้วย Helm (แนะนำ)

```bash
# เพิ่ม Helm Repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# ดู Values
helm show values grafana/grafana > grafana-values.yaml

# ติดตั้งแบบง่าย
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set adminPassword=admin123

# ดู Admin Password ที่ Generate
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# เข้าถึง Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80 &
echo "Grafana: http://localhost:3000 (admin/admin123)"
```

### Custom Values สำหรับ Production

```yaml
# grafana-values.yaml
replicas: 2

# Admin Credentials
adminUser: admin
adminPassword: "your-secure-password"  # ควรใช้ Secret แทน

# Persistence
persistence:
  enabled: true
  storageClassName: standard
  size: 10Gi

# Resource Limits
resources:
  limits:
    cpu: 500m
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 256Mi

# Service
service:
  type: ClusterIP
  port: 80

# Ingress
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - grafana.example.com
  tls:
    - secretName: grafana-tls
      hosts:
        - grafana.example.com

# Datasource Auto-configuration
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-kube-prometheus-prometheus:9090
        access: proxy
        isDefault: true
        jsonData:
          timeInterval: "30s"
      - name: Alertmanager
        type: alertmanager
        url: http://prometheus-kube-prometheus-alertmanager:9093
        access: proxy

# Dashboard Providers
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: 'default'
        orgId: 1
        folder: 'Kubernetes'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default

# Pre-configured Dashboards
dashboards:
  default:
    kubernetes-cluster:
      gnetId: 6417
      revision: 1
      datasource: Prometheus
    node-exporter:
      gnetId: 1860
      revision: 36
      datasource: Prometheus
    kubernetes-pods:
      gnetId: 6336
      revision: 1
      datasource: Prometheus

# Environment Variables
env:
  GF_SECURITY_ALLOW_EMBEDDING: "true"
  GF_AUTH_ANONYMOUS_ENABLED: "false"
  GF_LOG_LEVEL: "info"

# Grafana Configuration
grafana.ini:
  server:
    root_url: "https://grafana.example.com"
  security:
    cookie_secure: true
    cookie_samesite: lax
  auth:
    disable_login_form: false
  auth.ldap:
    enabled: false
  smtp:
    enabled: true
    host: smtp.gmail.com:587
    user: your-email@gmail.com
    password: your-app-password
    from_address: grafana@example.com
```

```bash
# ติดตั้งด้วย Custom Values
helm install grafana grafana/grafana \
  --namespace monitoring \
  -f grafana-values.yaml

# หรือ Upgrade
helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  -f grafana-values.yaml
```

### ติดตั้ง Grafana แบบ Manual

```yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
        - name: grafana
          image: grafana/grafana:10.2.0
          ports:
            - containerPort: 3000
              name: http-grafana
          env:
            - name: GF_SECURITY_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: grafana-credentials
                  key: username
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-credentials
                  key: password
            - name: GF_INSTALL_PLUGINS
              value: "grafana-clock-panel,grafana-piechart-panel"
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-storage
            - mountPath: /etc/grafana/provisioning/datasources
              name: grafana-datasources
            - mountPath: /etc/grafana/provisioning/dashboards
              name: grafana-dashboard-providers
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 60
            periodSeconds: 30
          resources:
            requests:
              memory: 256Mi
              cpu: 100m
            limits:
              memory: 1Gi
              cpu: 500m
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana-pvc
        - name: grafana-datasources
          configMap:
            defaultMode: 420
            name: grafana-datasources
        - name: grafana-dashboard-providers
          configMap:
            defaultMode: 420
            name: grafana-dashboard-providers
---
apiVersion: v1
kind: Secret
metadata:
  name: grafana-credentials
  namespace: monitoring
type: Opaque
stringData:
  username: admin
  password: "your-secure-password"
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-service:9090
        access: proxy
        isDefault: true
```

---

## Import Kubernetes Dashboards

### ยอดนิยมใน Grafana.com

```bash
# Kubernetes Cluster Dashboard (ID: 6417)
# ดู Node CPU, Memory, Network, Disk

# Node Exporter Dashboard (ID: 1860)
# รายละเอียด Node Metrics

# Kubernetes Pod Dashboard (ID: 6336)
# CPU, Memory สำหรับแต่ละ Pod

# Kubernetes Deployment Dashboard (ID: 8588)
# Status ของ Deployments

# Kubernetes Namespace (ID: 8686)
# Resources ตาม Namespace

# NGINX Ingress Controller (ID: 9614)
# Ingress Metrics

# MySQL Overview (ID: 7362)
# MySQL Metrics
```

### Import Dashboard ผ่าน API

```bash
# ดู URL ของ Grafana
GRAFANA_URL="http://localhost:3000"
GRAFANA_USER="admin"
GRAFANA_PASS="admin123"

# Import Dashboard จาก Grafana.com
curl -X POST "$GRAFANA_URL/api/dashboards/import" \
  -H "Content-Type: application/json" \
  -u "$GRAFANA_USER:$GRAFANA_PASS" \
  -d '{
    "dashboard": null,
    "folderId": 0,
    "inputs": [
      {
        "name": "DS_PROMETHEUS",
        "type": "datasource",
        "pluginId": "prometheus",
        "value": "Prometheus"
      }
    ],
    "overwrite": true,
    "path": "https://grafana.com/api/dashboards/1860/revisions/36/download"
  }'

# Function สำหรับ Import Dashboard by ID
import_dashboard() {
    local dashboard_id="$1"
    local datasource="${2:-Prometheus}"
    
    echo "Importing Dashboard ID: $dashboard_id"
    
    # ดาวน์โหลด Dashboard JSON
    DASHBOARD_JSON=$(curl -s "https://grafana.com/api/dashboards/${dashboard_id}/revisions/latest/download")
    
    # Import
    curl -s -X POST "$GRAFANA_URL/api/dashboards/import" \
        -H "Content-Type: application/json" \
        -u "$GRAFANA_USER:$GRAFANA_PASS" \
        -d "{
            \"dashboard\": $DASHBOARD_JSON,
            \"folderId\": 0,
            \"inputs\": [{
                \"name\": \"DS_PROMETHEUS\",
                \"type\": \"datasource\",
                \"pluginId\": \"prometheus\",
                \"value\": \"$datasource\"
            }],
            \"overwrite\": true
        }" | python3 -m json.tool
}

# Import Dashboards
import_dashboard 1860  # Node Exporter
import_dashboard 6417  # Kubernetes Cluster
import_dashboard 6336  # Kubernetes Pods
```

### Import Dashboard ผ่าน ConfigMap (Automated)

```yaml
# grafana-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubernetes-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"  # Label นี้ทำให้ Grafana Auto-import Dashboard
data:
  kubernetes-cluster.json: |
    {
      "__inputs": [
        {
          "name": "DS_PROMETHEUS",
          "label": "Prometheus",
          "description": "",
          "type": "datasource",
          "pluginId": "prometheus",
          "pluginName": "Prometheus"
        }
      ],
      "__requires": [
        {
          "type": "grafana",
          "id": "grafana",
          "name": "Grafana",
          "version": "6.0.0"
        }
      ],
      "annotations": {
        "list": []
      },
      "description": "Kubernetes Cluster Monitoring Dashboard",
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "panels": [],
      "refresh": "30s",
      "schemaVersion": 16,
      "style": "dark",
      "tags": ["kubernetes"],
      "title": "Kubernetes Cluster",
      "uid": "kubernetes-cluster",
      "version": 1
    }
```

---

## Custom Dashboards

### สร้าง Dashboard ผ่าน UI

1. คลิก **+** > **Dashboard**
2. คลิก **Add visualization**
3. เลือก Data Source
4. เขียน PromQL Query
5. เลือก Visualization Type
6. บันทึก Dashboard

### สร้าง Dashboard ผ่าน API

```bash
# สร้าง Dashboard ใหม่
GRAFANA_URL="http://localhost:3000"
curl -X POST "$GRAFANA_URL/api/dashboards/db" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d '{
    "dashboard": {
      "id": null,
      "title": "My Kubernetes Dashboard",
      "tags": ["kubernetes", "custom"],
      "timezone": "browser",
      "refresh": "30s",
      "panels": [
        {
          "id": 1,
          "title": "CPU Usage",
          "type": "graph",
          "targets": [
            {
              "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
              "legendFormat": "{{instance}}",
              "refId": "A"
            }
          ],
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 0
          }
        },
        {
          "id": 2,
          "title": "Memory Usage",
          "type": "graph",
          "targets": [
            {
              "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100",
              "legendFormat": "{{instance}}",
              "refId": "A"
            }
          ],
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 0
          }
        }
      ]
    },
    "folderId": 0,
    "overwrite": false
  }'
```

### Dashboard JSON Template

```json
{
  "title": "Kubernetes Production Dashboard",
  "uid": "k8s-prod",
  "tags": ["kubernetes", "production"],
  "timezone": "browser",
  "refresh": "30s",
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 2,
        "multi": true,
        "includeAll": true,
        "label": "Namespace"
      },
      {
        "name": "pod",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info{namespace=~\"$namespace\"}, pod)",
        "refresh": 2,
        "multi": true,
        "includeAll": true,
        "label": "Pod"
      }
    ]
  },
  "panels": [
    {
      "title": "Total Pods",
      "type": "stat",
      "id": 1,
      "targets": [
        {
          "expr": "count(kube_pod_info{namespace=~\"$namespace\"})",
          "instant": true
        }
      ],
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto"
      },
      "fieldConfig": {
        "defaults": {
          "color": {"mode": "thresholds"},
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 50},
              {"color": "red", "value": 100}
            ]
          }
        }
      },
      "gridPos": {"h": 4, "w": 4, "x": 0, "y": 0}
    }
  ]
}
```

---

## Workshop: Production Monitoring Dashboard

### เป้าหมาย

ในส่วนนี้เราจะสร้าง Production Monitoring Dashboard ที่ครอบคลุม:
1. Node Health Overview
2. Pod Status และ Resources
3. Application Performance Metrics
4. Error Rate และ Latency
5. Alerts Summary

### ขั้นตอนที่ 1: เตรียม Environment

```bash
# ตรวจสอบ Prometheus และ Grafana กำลังทำงาน
kubectl get pods -n monitoring

# Port Forward Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80 &
# หรือ
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &

echo "Grafana: http://localhost:3000"

# Port Forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
echo "Prometheus: http://localhost:9090"
```

### ขั้นตอนที่ 2: ตั้งค่า Datasource

```bash
# ตั้งค่า Prometheus Datasource ผ่าน API
curl -X POST "http://localhost:3000/api/datasources" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus-kube-prometheus-prometheus:9090",
    "access": "proxy",
    "isDefault": true,
    "jsonData": {
      "timeInterval": "30s",
      "queryTimeout": "60s",
      "httpMethod": "POST"
    }
  }'

# ทดสอบ Datasource
curl -X GET "http://localhost:3000/api/datasources/1/health" \
  -u admin:admin123
```

### ขั้นตอนที่ 3: สร้าง Node Overview Dashboard

```bash
# สร้าง Node Overview Dashboard
cat > /tmp/node-dashboard.json << 'DASHBOARD_EOF'
{
  "dashboard": {
    "title": "Kubernetes Node Overview",
    "uid": "k8s-nodes",
    "tags": ["kubernetes", "nodes"],
    "timezone": "browser",
    "refresh": "30s",
    "panels": [
      {
        "title": "Node CPU Usage %",
        "type": "timeseries",
        "id": 1,
        "targets": [
          {
            "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 70},
                {"color": "red", "value": 90}
              ]
            }
          }
        },
        "options": {
          "tooltip": {"mode": "multi"}
        },
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "title": "Node Memory Usage %",
        "type": "timeseries",
        "id": 2,
        "targets": [
          {
            "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100",
            "legendFormat": "{{instance}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      }
    ]
  },
  "folderId": 0,
  "overwrite": true
}
DASHBOARD_EOF

# Import Dashboard
curl -X POST "http://localhost:3000/api/dashboards/db" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d @/tmp/node-dashboard.json
```

### ขั้นตอนที่ 4: สร้าง Pod Status Dashboard

```yaml
# pod-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-status-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  pod-status.json: |
    {
      "title": "Kubernetes Pod Status",
      "uid": "k8s-pods",
      "tags": ["kubernetes", "pods"],
      "timezone": "browser",
      "refresh": "30s",
      "templating": {
        "list": [
          {
            "name": "namespace",
            "type": "query",
            "datasource": {"type": "prometheus", "uid": "prometheus"},
            "query": "label_values(kube_pod_info, namespace)",
            "refresh": 2,
            "includeAll": true,
            "multi": true,
            "label": "Namespace"
          }
        ]
      },
      "panels": [
        {
          "title": "Running Pods",
          "type": "stat",
          "id": 1,
          "targets": [
            {
              "expr": "count(kube_pod_status_phase{phase=\"Running\", namespace=~\"$namespace\"})",
              "instant": true,
              "refId": "A"
            }
          ],
          "options": {
            "colorMode": "value",
            "graphMode": "none",
            "justifyMode": "center",
            "textMode": "auto",
            "reduceOptions": {
              "calcs": ["lastNotNull"]
            }
          },
          "fieldConfig": {
            "defaults": {
              "color": {"mode": "fixed", "fixedColor": "green"},
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [{"color": "green", "value": null}]
              }
            }
          },
          "gridPos": {"h": 4, "w": 4, "x": 0, "y": 0}
        },
        {
          "title": "Pending Pods",
          "type": "stat",
          "id": 2,
          "targets": [
            {
              "expr": "count(kube_pod_status_phase{phase=\"Pending\", namespace=~\"$namespace\"}) or vector(0)",
              "instant": true,
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "color": {"mode": "fixed", "fixedColor": "yellow"}
            }
          },
          "gridPos": {"h": 4, "w": 4, "x": 4, "y": 0}
        },
        {
          "title": "Failed Pods",
          "type": "stat",
          "id": 3,
          "targets": [
            {
              "expr": "count(kube_pod_status_phase{phase=\"Failed\", namespace=~\"$namespace\"}) or vector(0)",
              "instant": true,
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "color": {"mode": "fixed", "fixedColor": "red"}
            }
          },
          "gridPos": {"h": 4, "w": 4, "x": 8, "y": 0}
        },
        {
          "title": "Pod Restarts (last 1h)",
          "type": "timeseries",
          "id": 4,
          "targets": [
            {
              "expr": "increase(kube_pod_container_status_restarts_total{namespace=~\"$namespace\"}[1h]) > 0",
              "legendFormat": "{{namespace}}/{{pod}}",
              "refId": "A"
            }
          ],
          "gridPos": {"h": 8, "w": 24, "x": 0, "y": 4}
        },
        {
          "title": "Pod CPU Usage",
          "type": "timeseries",
          "id": 5,
          "targets": [
            {
              "expr": "sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=\"\",container!=\"POD\",namespace=~\"$namespace\"}[5m]))",
              "legendFormat": "{{namespace}}/{{pod}}",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {"unit": "cores"}
          },
          "gridPos": {"h": 8, "w": 12, "x": 0, "y": 12}
        },
        {
          "title": "Pod Memory Usage",
          "type": "timeseries",
          "id": 6,
          "targets": [
            {
              "expr": "sum by (pod, namespace) (container_memory_working_set_bytes{container!=\"\",container!=\"POD\",namespace=~\"$namespace\"})",
              "legendFormat": "{{namespace}}/{{pod}}",
              "refId": "A"
            }
          ],
          "fieldConfig": {
            "defaults": {"unit": "bytes"}
          },
          "gridPos": {"h": 8, "w": 12, "x": 12, "y": 12}
        }
      ]
    }
```

```bash
# Apply Dashboard ConfigMap
kubectl apply -f pod-dashboard-configmap.yaml

# Grafana จะ Auto-import Dashboard นี้เนื่องจาก Label grafana_dashboard: "1"
```

### ขั้นตอนที่ 5: สร้าง Application Performance Dashboard

```bash
# สร้าง Application Performance Dashboard
cat > /tmp/app-performance.json << 'EOF'
{
  "dashboard": {
    "title": "Application Performance",
    "uid": "app-perf",
    "tags": ["application", "performance"],
    "timezone": "browser",
    "refresh": "30s",
    "panels": [
      {
        "title": "Request Rate (req/s)",
        "type": "timeseries",
        "id": 1,
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {"unit": "reqps"}
        },
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "title": "Error Rate (%)",
        "type": "timeseries",
        "id": 2,
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100",
            "legendFormat": "{{service}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "title": "Response Time P50/P95/P99",
        "type": "timeseries",
        "id": 3,
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P50",
            "refId": "A"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P95",
            "refId": "B"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P99",
            "refId": "C"
          }
        ],
        "fieldConfig": {
          "defaults": {"unit": "s"}
        },
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8}
      }
    ]
  },
  "folderId": 0,
  "overwrite": true
}
EOF

curl -X POST "http://localhost:3000/api/dashboards/db" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d @/tmp/app-performance.json
```

### ขั้นตอนที่ 6: ตั้งค่า Grafana Alerting

```bash
# สร้าง Contact Point (Slack)
curl -X POST "http://localhost:3000/api/v1/provisioning/contact-points" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d '{
    "name": "Slack Alerts",
    "type": "slack",
    "settings": {
      "url": "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK",
      "channel": "#alerts",
      "title": "Kubernetes Alert: {{ .CommonLabels.alertname }}"
    }
  }'

# สร้าง Alert Rule
curl -X POST "http://localhost:3000/api/v1/provisioning/alert-rules" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d '{
    "title": "High CPU Usage",
    "condition": "C",
    "data": [
      {
        "refId": "A",
        "queryType": "",
        "relativeTimeRange": {
          "from": 300,
          "to": 0
        },
        "datasourceUid": "prometheus",
        "model": {
          "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
          "intervalMs": 1000,
          "maxDataPoints": 43200,
          "refId": "A"
        }
      },
      {
        "refId": "C",
        "datasourceUid": "__expr__",
        "model": {
          "conditions": [
            {
              "evaluator": {
                "params": [90],
                "type": "gt"
              },
              "operator": {"type": "and"},
              "query": {"params": ["A"]},
              "reducer": {"type": "last"},
              "type": "query"
            }
          ],
          "refId": "C",
          "type": "classic_conditions"
        }
      }
    ],
    "noDataState": "NoData",
    "execErrState": "Error",
    "for": "5m",
    "annotations": {
      "description": "CPU usage is above 90% on {{ $labels.instance }}",
      "summary": "High CPU Usage Alert"
    },
    "labels": {
      "severity": "warning",
      "team": "platform"
    },
    "folderUID": "alerts",
    "ruleGroup": "kubernetes-alerts"
  }'
```

### ขั้นตอนที่ 7: Dashboard Variables

```json
// ตัวอย่าง Template Variables
{
  "templating": {
    "list": [
      {
        "name": "cluster",
        "type": "constant",
        "query": "production",
        "label": "Cluster"
      },
      {
        "name": "namespace",
        "type": "query",
        "datasource": {"type": "prometheus"},
        "query": "label_values(kube_namespace_labels, namespace)",
        "refresh": 2,
        "multi": true,
        "includeAll": true,
        "allValue": ".*",
        "label": "Namespace"
      },
      {
        "name": "pod",
        "type": "query",
        "datasource": {"type": "prometheus"},
        "query": "label_values(kube_pod_info{namespace=~\"$namespace\"}, pod)",
        "refresh": 2,
        "multi": true,
        "includeAll": true,
        "label": "Pod",
        "hide": 0
      },
      {
        "name": "interval",
        "type": "interval",
        "query": "1m,5m,10m,30m,1h,6h,12h,1d",
        "current": {"text": "5m", "value": "5m"},
        "label": "Interval"
      }
    ]
  }
}
```

### ขั้นตอนที่ 8: Export และ Import Dashboards

```bash
# Export Dashboard เป็น JSON
DASHBOARD_UID="k8s-nodes"
curl -s "http://localhost:3000/api/dashboards/uid/$DASHBOARD_UID" \
  -u admin:admin123 \
  | python3 -c "import sys, json; d=json.load(sys.stdin)['dashboard']; d.pop('id'); print(json.dumps(d, indent=2))" \
  > /tmp/$DASHBOARD_UID.json

echo "Dashboard exported to /tmp/$DASHBOARD_UID.json"

# Import Dashboard จาก JSON
curl -X POST "http://localhost:3000/api/dashboards/db" \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d "{
    \"dashboard\": $(cat /tmp/$DASHBOARD_UID.json),
    \"folderId\": 0,
    \"overwrite\": true
  }"
```

### ขั้นตอนที่ 9: Grafana Annotations

```bash
# สร้าง Annotation เมื่อ Deployment เกิดขึ้น
GRAFANA_URL="http://localhost:3000"

# เพิ่ม Annotation สำหรับ Deployment
add_deployment_annotation() {
    local service="$1"
    local version="$2"
    
    curl -X POST "$GRAFANA_URL/api/annotations" \
        -H "Content-Type: application/json" \
        -u admin:admin123 \
        -d "{
            \"text\": \"Deployed $service version $version\",
            \"tags\": [\"deployment\", \"$service\"],
            \"time\": $(date +%s000)
        }"
}

# ใช้งาน
add_deployment_annotation "my-app" "v1.2.3"
```

### ขั้นตอนที่ 10: Production Dashboard Checklist

```bash
# Script ตรวจสอบ Dashboard ก่อน Go-live
check_dashboard() {
    local dashboard_uid="$1"
    
    echo "=== Checking Dashboard: $dashboard_uid ==="
    
    # ตรวจสอบว่า Dashboard มีอยู่
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
        "http://localhost:3000/api/dashboards/uid/$dashboard_uid" \
        -u admin:admin123)
    
    if [ "$STATUS" = "200" ]; then
        echo "✓ Dashboard exists"
    else
        echo "✗ Dashboard not found (HTTP $STATUS)"
        return 1
    fi
    
    # ตรวจสอบว่า Datasource ทำงาน
    DS_STATUS=$(curl -s "http://localhost:3000/api/datasources/1/health" \
        -u admin:admin123 | python3 -c "import sys, json; print(json.load(sys.stdin).get('status', 'unknown'))")
    
    if [ "$DS_STATUS" = "OK" ]; then
        echo "✓ Datasource is healthy"
    else
        echo "✗ Datasource status: $DS_STATUS"
    fi
    
    echo "Dashboard check completed"
}

check_dashboard "k8s-nodes"
check_dashboard "k8s-pods"
```

---

## Tips และ Best Practices

### 1. Dashboard Organization

```
ควรแบ่ง Dashboard ตาม:
- Infrastructure (Nodes, Network, Storage)
- Application (API, Database, Cache)
- Business Metrics (Revenue, Users, Orders)
- SLO/SLI (Availability, Latency)
```

### 2. Variable Best Practices

```json
// ใช้ Variables เพื่อ Dynamic Dashboards
// อย่า Hardcode ค่าใน Queries
// ใช้ $variable ใน Queries

// ตัวอย่าง: ใช้ $namespace ใน Query
"expr": "kube_pod_info{namespace=~\"$namespace\"}"

// ตัวอย่าง: ใช้ $interval สำหรับ Rate
"expr": "rate(http_requests_total[$interval])"
```

### 3. Performance Tips

```
# Grafana Performance Tips:
1. ใช้ Recording Rules เพื่อ Pre-compute Queries ที่ซับซ้อน
2. ตั้ง Refresh Interval ให้เหมาะสม (ไม่ต้องใช้ 5s ถ้า 30s พอ)
3. ใช้ Time Range ที่สั้นกว่า เช่น 1h แทน 24h
4. Limit จำนวน Panels ใน Dashboard เดียว
5. ใช้ __interval Variable แทนค่า Fixed ใน rate()
```

### 4. สร้าง SLO Dashboard

```json
{
  "title": "SLO Dashboard",
  "panels": [
    {
      "title": "Availability SLO (99.9%)",
      "type": "gauge",
      "targets": [{
        "expr": "(1 - sum(rate(http_requests_total{status=~\"5..\"}[30d])) / sum(rate(http_requests_total[30d]))) * 100"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 99,
          "max": 100,
          "thresholds": {
            "steps": [
              {"color": "red", "value": 99},
              {"color": "yellow", "value": 99.5},
              {"color": "green", "value": 99.9}
            ]
          }
        }
      }
    }
  ]
}
```

---

## สรุป

Grafana เป็น Visualization Tool ที่ทรงพลังสำหรับ Kubernetes Monitoring:

1. **Installation**: ใช้ Helm สำหรับการติดตั้งที่ง่ายและ Reproducible
2. **Import Dashboards**: ใช้ Dashboards จาก Grafana.com เพื่อเริ่มต้นได้เร็ว
3. **Custom Dashboards**: สร้าง Dashboard ที่ตรงกับความต้องการของทีม
4. **Variables**: ใช้ Template Variables เพื่อ Dynamic Dashboards
5. **Alerting**: ตั้ง Alert Rules จาก Dashboard โดยตรง

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ AlertManager ซึ่งเป็นระบบ Alert Management ที่ทำงานร่วมกับ Prometheus
