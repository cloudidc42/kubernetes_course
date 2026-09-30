# Part 66: AlertManager - Production Alerting

## AlertManager คืออะไร

AlertManager เป็น Component สำคัญของ Prometheus Ecosystem ที่ทำหน้าที่จัดการ Alerts ที่ส่งมาจาก Prometheus AlertManager ไม่ใช่แค่ Forward Alerts แต่ยัง:

- **Grouping**: รวม Alerts ที่เกี่ยวข้องกันเข้าด้วยกัน
- **Inhibition**: ระงับ Alerts บางอันเมื่อมี Alert อื่นที่ Critical กว่า
- **Silencing**: หยุด Notifications ชั่วคราว
- **Routing**: ส่ง Alerts ไปยัง Team หรือ Channel ที่ถูกต้อง

### AlertManager Architecture

```
+------------------+      Alert     +------------------+
|   Prometheus     | ------------> |  AlertManager    |
|   (Alert Rules)  |               |                  |
+------------------+               |  1. Receive      |
                                    |  2. Group        |
                                    |  3. Route        |
                                    |  4. Inhibit      |
                                    |  5. Silence      |
                                    |  6. Notify       |
                                    +--------+---------+
                                             |
                     +-----------------------+----------------------+
                     |                       |                      |
              +------v------+        +-------v------+      +-------v------+
              |    Slack    |        |  PagerDuty   |      |    Email     |
              +-------------+        +--------------+      +--------------+
```

### Alert Lifecycle

```
1. Alert Rule Firing  → Prometheus ตรวจพบ Condition
2. Pending           → Alert อยู่ในสถานะ "For" Duration
3. Firing            → Alert ถูกส่งไปยัง AlertManager
4. AlertManager      → Group, Route, Inhibit, Silence
5. Notification      → ส่งไปยัง Receivers (Slack, PagerDuty, etc.)
6. Resolved          → Alert กลับสู่สถานะปกติ
```

---

## Alert Rules (PromQL)

### การสร้าง Alert Rules

```yaml
# prometheus-alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alert-rules
  namespace: monitoring
  labels:
    release: prometheus
    role: alert-rules
spec:
  groups:
    # ===== NODE ALERTS =====
    - name: node.alerts
      rules:
        # Node Down
        - alert: NodeDown
          expr: up{job="node-exporter"} == 0
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node ไม่ตอบสนอง"
            description: "Node {{ $labels.instance }} ไม่ตอบสนองมานานกว่า 5 นาที"
            runbook_url: "https://wiki.example.com/runbooks/node-down"

        # High CPU Usage
        - alert: NodeHighCPUUsage
          expr: |
            100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "CPU Usage สูงผิดปกติ"
            description: "Node {{ $labels.instance }} มี CPU Usage {{ $value }}% มานานกว่า 15 นาที"

        # High Memory Usage
        - alert: NodeHighMemoryUsage
          expr: |
            (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Memory Usage สูงผิดปกติ"
            description: "Node {{ $labels.instance }} มี Memory Usage {{ $value | humanize }}% มานานกว่า 10 นาที"

        # Disk Space Low
        - alert: NodeLowDiskSpace
          expr: |
            (1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100 > 85
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Disk Space เหลือน้อย"
            description: "Node {{ $labels.instance }} มี Disk Usage {{ $value | humanize }}% บน {{ $labels.mountpoint }}"

        # Disk Space Critical
        - alert: NodeCriticalDiskSpace
          expr: |
            (1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100 > 95
          for: 1m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Disk Space วิกฤต!"
            description: "Node {{ $labels.instance }} มี Disk ใกล้เต็ม {{ $value | humanize }}%"

    # ===== POD ALERTS =====
    - name: pod.alerts
      rules:
        # Pod CrashLoopBackOff
        - alert: PodCrashLoopBackOff
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
          for: 5m
          labels:
            severity: warning
            team: app
          annotations:
            summary: "Pod กำลัง Crash"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} กำลัง Restart ซ้ำๆ"
            runbook_url: "https://wiki.example.com/runbooks/pod-crash"

        # Pod Not Ready
        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="false"} == 1
          for: 15m
          labels:
            severity: warning
            team: app
          annotations:
            summary: "Pod ไม่พร้อมทำงาน"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} ไม่ Ready มานานกว่า 15 นาที"

        # Pod OOMKilled
        - alert: PodOOMKilled
          expr: |
            kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          for: 0m
          labels:
            severity: warning
            team: app
          annotations:
            summary: "Pod ถูก OOMKilled"
            description: "Container {{ $labels.container }} ใน Pod {{ $labels.namespace }}/{{ $labels.pod }} ถูก Kill เนื่องจาก Memory เกิน Limit"

        # High Pod Restart Count
        - alert: HighPodRestartCount
          expr: |
            kube_pod_container_status_restarts_total > 10
          for: 1m
          labels:
            severity: warning
            team: app
          annotations:
            summary: "Pod Restart Count สูง"
            description: "Container {{ $labels.container }} ใน Pod {{ $labels.pod }} Restart ไปแล้ว {{ $value }} ครั้ง"

    # ===== DEPLOYMENT ALERTS =====
    - name: deployment.alerts
      rules:
        # Deployment Replicas Mismatch
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas != kube_deployment_status_available_replicas
          for: 10m
          labels:
            severity: warning
            team: app
          annotations:
            summary: "Deployment Replicas ไม่ตรงกัน"
            description: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} ต้องการ {{ $value }} Replicas แต่มีเพียง {{ query \"kube_deployment_status_available_replicas{deployment='\" $labels.deployment \"',namespace='\" $labels.namespace \"'}\" | first | value }} Replicas"

        # Deployment Not Available
        - alert: DeploymentNotAvailable
          expr: |
            kube_deployment_status_replicas_available == 0
          for: 5m
          labels:
            severity: critical
            team: app
          annotations:
            summary: "Deployment ไม่มี Replicas ที่พร้อมใช้"
            description: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} ไม่มี Replicas ที่ Available"

    # ===== APPLICATION ALERTS =====
    - name: application.alerts
      rules:
        # High Error Rate
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            / sum(rate(http_requests_total[5m])) by (service) > 0.05
          for: 5m
          labels:
            severity: critical
            team: app
          annotations:
            summary: "Error Rate สูงผิดปกติ"
            description: "Service {{ $labels.service }} มี Error Rate {{ $value | humanizePercentage }}"

        # High Latency
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1.0
          for: 10m
          labels:
            severity: warning
            team: app
          annotations:
            summary: "Latency สูงผิดปกติ"
            description: "Service {{ $labels.service }} มี P99 Latency {{ $value }}s"

    # ===== KUBERNETES ALERTS =====
    - name: kubernetes.alerts
      rules:
        # PersistentVolume Near Full
        - alert: PersistentVolumeNearFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100 > 85
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "PersistentVolume ใกล้เต็ม"
            description: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} ใช้ไป {{ $value | humanize }}%"

        # Kubernetes API Server Errors
        - alert: KubeAPIServerErrors
          expr: |
            sum(rate(apiserver_request_total{code=~"5.."}[5m])) by (verb, resource) > 0.1
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Kubernetes API Server มี Errors สูง"
            description: "API Server มี Error Rate สูงสำหรับ {{ $labels.verb }} {{ $labels.resource }}"

        # Node Not Ready
        - alert: KubeNodeNotReady
          expr: |
            kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node ไม่พร้อมทำงาน"
            description: "Node {{ $labels.node }} ไม่พร้อมทำงาน"
```

---

## Notification Channels

### AlertManager Configuration

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    # Global Configuration
    global:
      # Default Timeout สำหรับ HTTP Requests
      resolve_timeout: 5m
      
      # SMTP Configuration (สำหรับ Email)
      smtp_from: 'alertmanager@example.com'
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_auth_username: 'your-email@gmail.com'
      smtp_auth_password: 'your-app-password'
      smtp_require_tls: true
      
      # Slack Global Config
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    # Templates
    templates:
      - '/etc/alertmanager/templates/*.tmpl'
    
    # Routing Tree
    route:
      # Default Receiver
      receiver: 'default'
      
      # Group Alerts
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s      # รอ 30 วินาทีก่อนส่ง Group แรก
      group_interval: 5m   # ส่ง Updates ทุก 5 นาที
      repeat_interval: 4h  # Repeat Alert ทุก 4 ชั่วโมง
      
      # Sub-routes
      routes:
        # Critical Alerts -> PagerDuty
        - match:
            severity: critical
          receiver: 'pagerduty-critical'
          group_wait: 10s
          repeat_interval: 1h
          continue: true  # ส่งต่อไปยัง Parent Route ด้วย
        
        # Platform Team Alerts -> Platform Slack Channel
        - match:
            team: platform
          receiver: 'slack-platform'
          group_by: ['alertname', 'node']
          
        # App Team Alerts -> App Slack Channel
        - match:
            team: app
          receiver: 'slack-app'
          
        # All Warnings -> Email
        - match:
            severity: warning
          receiver: 'email-warnings'
    
    # Inhibit Rules - ระงับ Alert เมื่อมี Alert ที่ Critical กว่า
    inhibit_rules:
      # ถ้า NodeDown ก็ไม่ต้อง Alert สำหรับ Pod ที่อยู่บน Node นั้น
      - source_match:
          severity: critical
          alertname: NodeDown
        target_match:
          severity: warning
        equal: ['instance']
      
      # ถ้า Cluster Down ก็ไม่ต้อง Alert สำหรับ Components
      - source_match:
          alertname: KubeNodeNotReady
        target_match_re:
          alertname: "Pod.*|Deployment.*"
        equal: ['namespace']
    
    # Receivers
    receivers:
      # Default Receiver
      - name: 'default'
        slack_configs:
          - channel: '#alerts-default'
            title: '{{ .CommonAnnotations.summary }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      
      # PagerDuty สำหรับ Critical
      - name: 'pagerduty-critical'
        pagerduty_configs:
          - service_key: 'YOUR-PAGERDUTY-SERVICE-KEY'
            severity: '{{ .CommonLabels.severity }}'
            description: '{{ .CommonAnnotations.summary }}'
            details:
              firing: '{{ .Alerts.Firing | len }}'
              resolved: '{{ .Alerts.Resolved | len }}'
              runbook: '{{ .CommonAnnotations.runbook_url }}'
      
      # Slack สำหรับ Platform Team
      - name: 'slack-platform'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/PLATFORM/WEBHOOK'
            channel: '#platform-alerts'
            color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
            title: '{{ .CommonAnnotations.summary }}'
            title_link: '{{ template "slack.default.titlelink" . }}'
            text: |
              {{ range .Alerts }}
              *Alert:* {{ .Annotations.summary }}
              *Details:* {{ .Annotations.description }}
              *Severity:* {{ .Labels.severity }}
              *Runbook:* {{ .Annotations.runbook_url }}
              {{ end }}
            actions:
              - type: button
                text: 'View in Prometheus'
                url: '{{ template "slack.default.titlelink" . }}'
              - type: button
                text: 'Silence Alert'
                url: '{{ template "slack.default.actionsfunc" . }}'
      
      # Slack สำหรับ App Team
      - name: 'slack-app'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/APP/WEBHOOK'
            channel: '#app-alerts'
            color: '{{ if eq .Status "firing" }}{{ if eq .CommonLabels.severity "critical" }}danger{{ else }}warning{{ end }}{{ else }}good{{ end }}'
            title: '{{ .CommonLabels.alertname }}'
            text: |
              {{ range .Alerts }}
              {{ .Annotations.description }}
              {{ end }}
      
      # Email สำหรับ Warnings
      - name: 'email-warnings'
        email_configs:
          - to: 'team@example.com'
            from: 'alertmanager@example.com'
            smarthost: 'smtp.gmail.com:587'
            auth_username: 'your-email@gmail.com'
            auth_password: 'your-app-password'
            send_resolved: true
            headers:
              Subject: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
            html: |
              <h2>{{ .CommonAnnotations.summary }}</h2>
              {{ range .Alerts }}
              <p><strong>Description:</strong> {{ .Annotations.description }}</p>
              <p><strong>Severity:</strong> {{ .Labels.severity }}</p>
              <p><strong>Runbook:</strong> <a href="{{ .Annotations.runbook_url }}">Link</a></p>
              {{ end }}
```

### Slack Integration

```yaml
# slack-notification-template.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-templates
  namespace: monitoring
data:
  slack.tmpl: |
    {{ define "slack.custom.title" -}}
      [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
    {{- end }}
    
    {{ define "slack.custom.text" -}}
      {{ range .Alerts -}}
      *Alert:* {{ .Labels.alertname }}{{ if .Labels.severity }} - {{ .Labels.severity }}{{ end }}
      *Namespace:* {{ .Labels.namespace }}
      *Summary:* {{ .Annotations.summary }}
      *Description:* {{ .Annotations.description }}
      {{ if .Annotations.runbook_url -}}
      *Runbook:* {{ .Annotations.runbook_url }}
      {{- end }}
      *Started at:* {{ .StartsAt | since }}
      ---
      {{ end -}}
    {{- end }}
    
    {{ define "slack.custom.color" -}}
      {{ if eq .Status "firing" -}}
        {{ if eq .CommonLabels.severity "critical" -}}
          #FF0000
        {{- else if eq .CommonLabels.severity "warning" -}}
          #FFA500
        {{- else -}}
          #FFFF00
        {{- end }}
      {{- else -}}
        #00FF00
      {{- end }}
    {{- end }}
```

### PagerDuty Integration

```yaml
# pagerduty-config.yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: 'YOUR-PAGERDUTY-ROUTING-KEY'
        description: |
          {{ .CommonAnnotations.summary }}
        severity: |
          {{ if eq .CommonLabels.severity "critical" }}critical
          {{ else if eq .CommonLabels.severity "warning" }}warning
          {{ else }}info{{ end }}
        details:
          # Custom Details
          cluster: '{{ .CommonLabels.cluster }}'
          namespace: '{{ .CommonLabels.namespace }}'
          pod: '{{ .CommonLabels.pod }}'
          container: '{{ .CommonLabels.container }}'
          description: '{{ .CommonAnnotations.description }}'
          runbook: '{{ .CommonLabels.runbook_url }}'
        # Custom Fields
        links:
          - href: '{{ .CommonAnnotations.runbook_url }}'
            text: 'Runbook'
          - href: 'https://grafana.example.com/d/{{ .CommonLabels.dashboard_uid }}'
            text: 'Grafana Dashboard'
        images:
          - src: '{{ .CommonAnnotations.graph_url }}'
            alt: 'Graph'
            href: '{{ .CommonAnnotations.graph_url }}'
```

### Webhook Integration

```yaml
# webhook-receiver.yaml
receivers:
  - name: 'webhook-receiver'
    webhook_configs:
      - url: 'https://your-webhook-endpoint.example.com/alerts'
        send_resolved: true
        http_config:
          bearer_token: 'your-auth-token'
        # Custom Template สำหรับ Webhook Payload
        # AlertManager จะส่ง JSON ในรูปแบบ Default:
        # {
        #   "version": "4",
        #   "status": "firing|resolved",
        #   "alerts": [...],
        #   "commonLabels": {...},
        #   "commonAnnotations": {...}
        # }
```

---

## On-call Rotation

### การใช้ PagerDuty สำหรับ On-call

```yaml
# on-call-alertmanager-config.yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'service']
  routes:
    # Critical Alerts ส่งไปยัง On-call Engineer ผ่าน PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'
      group_wait: 0s  # ส่งทันที
      repeat_interval: 30m  # Escalate ทุก 30 นาที
      
    # Warning Alerts ส่งไปยัง Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_wait: 5m
      repeat_interval: 4h

receivers:
  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - routing_key: 'YOUR-ROUTING-KEY'
        severity: critical
        
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#k8s-warnings'
        send_resolved: true
```

### AlertManager Silence

```bash
# Silence Alert ผ่าน API
ALERTMANAGER_URL="http://localhost:9093"

# สร้าง Silence สำหรับ Maintenance Window
curl -X POST "$ALERTMANAGER_URL/api/v2/silences" \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "NodeHighCPUUsage",
        "isRegex": false
      },
      {
        "name": "instance",
        "value": "node-1",
        "isRegex": false
      }
    ],
    "startsAt": "2024-01-15T10:00:00Z",
    "endsAt": "2024-01-15T14:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance window for node-1"
  }'

# ดู Silences ที่มีอยู่
curl -s "$ALERTMANAGER_URL/api/v2/silences" | python3 -m json.tool

# ลบ Silence
SILENCE_ID="your-silence-id"
curl -X DELETE "$ALERTMANAGER_URL/api/v2/silences/$SILENCE_ID"
```

### Script สำหรับ Maintenance Window

```bash
#!/bin/bash
# create-maintenance-window.sh

ALERTMANAGER_URL="${1:-http://localhost:9093}"
START_TIME="${2:-$(date -u +%Y-%m-%dT%H:%M:%SZ)}"
DURATION_HOURS="${3:-2}"
CREATED_BY="${4:-admin}"
COMMENT="${5:-Scheduled maintenance}"

# คำนวณ End Time
END_TIME=$(date -u -d "$START_TIME + $DURATION_HOURS hours" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
  python3 -c "
from datetime import datetime, timedelta
start = datetime.fromisoformat('${START_TIME}'.replace('Z', '+00:00'))
end = start + timedelta(hours=$DURATION_HOURS)
print(end.strftime('%Y-%m-%dT%H:%M:%SZ'))
")

echo "Creating Maintenance Window:"
echo "Start: $START_TIME"
echo "End: $END_TIME"
echo "Comment: $COMMENT"

# สร้าง Silence สำหรับ All Alerts
curl -X POST "$ALERTMANAGER_URL/api/v2/silences" \
  -H "Content-Type: application/json" \
  -d "{
    \"matchers\": [
      {
        \"name\": \"severity\",
        \"value\": \".*\",
        \"isRegex\": true
      }
    ],
    \"startsAt\": \"$START_TIME\",
    \"endsAt\": \"$END_TIME\",
    \"createdBy\": \"$CREATED_BY\",
    \"comment\": \"$COMMENT\"
  }" | python3 -m json.tool

echo ""
echo "Maintenance window created successfully"
```

---

## Workshop: Setup Production Alerting

### เป้าหมาย

ในส่วนนี้เราจะ:
1. ติดตั้ง AlertManager
2. สร้าง Alert Rules สำหรับ Production
3. ตั้งค่า Slack Notifications
4. ทดสอบ Alert Flow
5. ตั้งค่า Silencing และ Inhibition

### ขั้นตอนที่ 1: Deploy AlertManager

```bash
# AlertManager มาพร้อมกับ kube-prometheus-stack
# ตรวจสอบว่า Running
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager

# เข้าถึง AlertManager UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093 &
echo "AlertManager: http://localhost:9093"
```

### ขั้นตอนที่ 2: ตั้งค่า AlertManager Configuration

```bash
# ดู Config ปัจจุบัน
kubectl get secret alertmanager-prometheus-kube-prometheus-alertmanager \
  -n monitoring \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 --decode

# สร้าง Custom Config
cat > /tmp/alertmanager-config.yaml << 'EOF'
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 10s
      repeat_interval: 1h
    
    - match:
        severity: warning
      receiver: 'warning-receiver'

receivers:
  - name: 'default-receiver'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK'
        channel: '#k8s-alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ "\n" }}{{ end }}'

  - name: 'critical-receiver'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK'
        channel: '#k8s-critical'
        color: 'danger'
        title: '[CRITICAL] {{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ "\n" }}{{ end }}'

  - name: 'warning-receiver'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK'
        channel: '#k8s-warnings'
        color: 'warning'
        title: '[WARNING] {{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ "\n" }}{{ end }}'

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'cluster', 'service']
EOF

# Update AlertManager Config
kubectl create secret generic alertmanager-prometheus-kube-prometheus-alertmanager \
  --from-file=alertmanager.yaml=/tmp/alertmanager-config.yaml \
  -n monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

### ขั้นตอนที่ 3: สร้าง Alert Rules

```bash
# สร้าง Alert Rules สำหรับ Workshop
kubectl apply -f - << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: workshop-alert-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: workshop.alerts
      rules:
        # Alert เมื่อ Pod ไม่ Ready (สำหรับ Workshop ตั้ง Threshold ต่ำ)
        - alert: PodNotReadyWorkshop
          expr: kube_pod_status_ready{condition="false"} == 1
          for: 1m
          labels:
            severity: warning
            team: workshop
          annotations:
            summary: "Pod ไม่ Ready (Workshop)"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} ไม่ Ready"
        
        # Alert เมื่อมี Pod ที่ Restart บ่อย
        - alert: PodRestartWorkshop
          expr: increase(kube_pod_container_status_restarts_total[5m]) > 0
          for: 0m
          labels:
            severity: info
            team: workshop
          annotations:
            summary: "Pod ทำการ Restart (Workshop)"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} ทำการ Restart"
EOF
```

### ขั้นตอนที่ 4: จำลอง Alert

```bash
# จำลอง Pod ที่ CrashLoop เพื่อทดสอบ Alert
kubectl create namespace alert-test

kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  namespace: alert-test
spec:
  containers:
    - name: crash-container
      image: busybox:1.36
      command: ["sh", "-c", "exit 1"]
      resources:
        requests:
          memory: "16Mi"
          cpu: "5m"
        limits:
          memory: "32Mi"
          cpu: "20m"
  restartPolicy: Always
EOF

# รอ และดู Alerts
sleep 30
kubectl get pods -n alert-test

# ดู Alerts ใน Prometheus
curl -s "http://localhost:9090/api/v1/alerts" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
alerts = data.get('data', {}).get('alerts', [])
for alert in alerts:
    print(f\"Alert: {alert['labels'].get('alertname')}\")
    print(f\"  State: {alert.get('state')}\")
    print(f\"  Summary: {alert.get('annotations', {}).get('summary')}\")
    print()
"
```

### ขั้นตอนที่ 5: ใช้งาน AlertManager API

```bash
ALERTMANAGER_URL="http://localhost:9093"

# ดู Status
curl -s "$ALERTMANAGER_URL/api/v2/status" | python3 -m json.tool

# ดู Alerts ที่กำลัง Firing
curl -s "$ALERTMANAGER_URL/api/v2/alerts" | python3 -m json.tool

# ดู Alert Groups
curl -s "$ALERTMANAGER_URL/api/v2/alerts/groups" | python3 -m json.tool

# Silence Alert สำหรับ 10 นาที
START_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u -d "+10 minutes" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
  python3 -c "from datetime import datetime, timedelta; print((datetime.utcnow() + timedelta(minutes=10)).strftime('%Y-%m-%dT%H:%M:%SZ'))")

curl -X POST "$ALERTMANAGER_URL/api/v2/silences" \
  -H "Content-Type: application/json" \
  -d "{
    \"matchers\": [
      {
        \"name\": \"namespace\",
        \"value\": \"alert-test\",
        \"isRegex\": false
      }
    ],
    \"startsAt\": \"$START_TIME\",
    \"endsAt\": \"$END_TIME\",
    \"createdBy\": \"workshop\",
    \"comment\": \"Workshop silence\"
  }"
```

### ขั้นตอนที่ 6: สร้าง Alertmanager Operator Configuration

```yaml
# alertmanager-operator-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: workshop-alertmanager-config
  namespace: monitoring
spec:
  route:
    receiver: 'workshop-receiver'
    matchers:
      - name: team
        value: workshop
    groupBy: ['alertname']
    groupWait: 10s
    groupInterval: 1m
    repeatInterval: 30m
    
  receivers:
    - name: 'workshop-receiver'
      slackConfigs:
        - apiURL:
            name: slack-webhook-secret
            key: webhook-url
          channel: '#k8s-workshop'
          sendResolved: true
          title: 'Workshop Alert: {{ .CommonAnnotations.summary }}'
          text: |
            {{ range .Alerts }}
            *Alert:* {{ .Labels.alertname }}
            *Details:* {{ .Annotations.description }}
            {{ end }}
```

### ขั้นตอนที่ 7: ทดสอบ Alert Routing

```bash
# สร้าง Alert Test Script
cat > /tmp/test-alert.sh << 'SCRIPT_EOF'
#!/bin/bash
ALERTMANAGER_URL="${1:-http://localhost:9093}"

echo "=== Testing Alert Routing ==="

# ส่ง Test Alert
curl -X POST "$ALERTMANAGER_URL/api/v2/alerts" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "warning",
        "team": "workshop",
        "namespace": "test",
        "pod": "test-pod"
      },
      "annotations": {
        "summary": "This is a test alert",
        "description": "Testing AlertManager routing"
      },
      "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
      "endsAt": "'$(date -u -d "+5 minutes" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || python3 -c "from datetime import datetime, timedelta; print((datetime.utcnow() + timedelta(minutes=5)).strftime('%Y-%m-%dT%H:%M:%SZ'))'")'",
      "generatorURL": "http://prometheus:9090"
    }
  ]'

echo ""
echo "Test alert sent! Check AlertManager UI at $ALERTMANAGER_URL"
SCRIPT_EOF

chmod +x /tmp/test-alert.sh
/tmp/test-alert.sh
```

### ขั้นตอนที่ 8: Monitor AlertManager Health

```bash
# ดู Metrics ของ AlertManager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093 &

# AlertManager Metrics
curl -s "http://localhost:9093/metrics" | grep -E "alertmanager_(alerts|notifications|notification)"

# PromQL สำหรับ Monitor AlertManager
# - จำนวน Alerts ที่กำลัง Firing
# alertmanager_alerts{state="firing"}

# - Notifications ที่ส่งสำเร็จ
# rate(alertmanager_notifications_total{status="success"}[5m])

# - Notifications ที่ Failed
# rate(alertmanager_notifications_failed_total[5m])
```

### ขั้นตอนที่ 9: สร้าง Runbook

```bash
# สร้าง Runbook Template สำหรับ Alert
cat > /tmp/runbook-template.md << 'RUNBOOK_EOF'
# Runbook: NodeHighCPUUsage

## Alert Summary
- **Alert Name**: NodeHighCPUUsage
- **Severity**: Warning
- **Team**: Platform

## เมื่อ Alert นี้ Firing

Alert นี้ Firing เมื่อ CPU Usage ของ Node สูงกว่า 90% มานานกว่า 15 นาที

## ขั้นตอนการแก้ไข

### 1. ตรวจสอบสถานะ
\`\`\`bash
# ดู Node ที่มีปัญหา
kubectl top nodes

# ดู Pods ที่ใช้ CPU มากที่สุด
kubectl top pods --all-namespaces --sort-by=cpu | head -20
\`\`\`

### 2. หา Root Cause
\`\`\`bash
# ดู Processes บน Node
kubectl debug node/${NODE} -it --image=busybox -- top

# ดู Events บน Node
kubectl describe node/${NODE}
\`\`\`

### 3. แก้ไข
- ถ้าเป็น Application ที่ใช้ CPU สูง: Scale Down/Kill Pod
- ถ้าเป็น System Process: Escalate ไปยัง Platform Team
- ถ้าเป็น Legitimate High Load: เพิ่ม Node หรือ Scale Cluster

## Escalation
- หากแก้ไขไม่ได้ใน 30 นาที: PagerDuty Page Oncall Engineer
- Contact: @platform-team ใน Slack
RUNBOOK_EOF

echo "Runbook created at /tmp/runbook-template.md"
```

### ขั้นตอนที่ 10: ทำความสะอาด

```bash
# ลบ Test Resources
kubectl delete namespace alert-test

# ลบ Workshop Alert Rules
kubectl delete prometheusrule workshop-alert-rules -n monitoring

# Stop Port Forwards
kill $(lsof -ti:9093) 2>/dev/null || true
kill $(lsof -ti:9090) 2>/dev/null || true

echo "Cleanup เสร็จสิ้น"
```

---

## Best Practices สำหรับ Alerting

### 1. Alert Rule Design

```yaml
# หลักการ Alert Rule ที่ดี:
# 1. Actionable - Alert ทุกอันต้องมีการ Action ที่ชัดเจน
# 2. Context - มีข้อมูลเพียงพอที่จะ Debug
# 3. Appropriate Severity - กำหนด Severity ให้ถูกต้อง
# 4. Not Too Noisy - อย่า Alert มากเกินไปจนทีม Ignore

# ตัวอย่าง Alert Rule ที่ดี:
- alert: ServiceHighLatency
  expr: |
    histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2.0
  for: 10m  # ต้อง Firing นาน 10 นาทีก่อนส่ง Alert
  labels:
    severity: warning
    team: app              # ระบุ Team ที่ Responsible
    service: my-api        # ระบุ Service
  annotations:
    summary: "Service {{ $labels.service }} Latency สูง"
    description: |
      Service {{ $labels.service }} มี P99 Latency {{ $value }}s
      ซึ่งสูงกว่า Threshold (2s)
    runbook_url: "https://wiki.example.com/runbooks/high-latency"
    dashboard_url: "https://grafana.example.com/d/app-perf"
```

### 2. Severity Levels

```
critical  - ส่งผลกระทบต่อ User โดยตรง ต้องแก้ทันที
warning   - อาจส่งผลกระทบถ้าไม่แก้ไข
info      - ข้อมูลสำหรับ Visibility ไม่ต้องการ Action
```

### 3. Alert Grouping

```yaml
# Group Alerts เพื่อลด Noise
route:
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 30s
  group_interval: 5m
```

---

## สรุป

AlertManager เป็นระบบที่ทรงพลังสำหรับจัดการ Alerts ใน Production:

1. **Alert Rules**: เขียน PromQL เพื่อกำหนด Conditions ที่ต้องการ Alert
2. **Routing**: ส่ง Alerts ไปยัง Team และ Channel ที่ถูกต้อง
3. **Inhibition**: ลด Alert Noise ด้วยการ Inhibit Alerts ที่ไม่จำเป็น
4. **Silencing**: หยุด Alerts ชั่วคราวระหว่าง Maintenance
5. **On-call**: ใช้ PagerDuty สำหรับ On-call Rotation

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ Jaeger สำหรับ Distributed Tracing
