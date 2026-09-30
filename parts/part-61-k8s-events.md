# Part 61: Kubernetes Events

## Kubernetes Events คืออะไร

Kubernetes Events เป็นกลไกสำคัญที่ช่วยให้เราเข้าใจว่าเกิดอะไรขึ้นภายใน Cluster ของเรา Events ถูกสร้างขึ้นโดย Kubernetes components ต่างๆ เช่น kubelet, kube-scheduler, และ kube-controller-manager เพื่อบันทึกสิ่งที่เกิดขึ้นกับ Resources ต่างๆ ใน Cluster

Events ใน Kubernetes เป็น API Objects ประเภทหนึ่ง ที่จัดเก็บไว้ใน etcd และสามารถ query ได้ผ่าน Kubernetes API ปกติแล้ว Events จะถูกเก็บไว้ประมาณ 1 ชั่วโมง (ค่า default) ก่อนที่จะถูกลบออกไปโดยอัตโนมัติ

### ทำไม Events ถึงสำคัญ

1. **Debugging และ Troubleshooting**: เมื่อ Pod ไม่สามารถ Start ได้ Events จะบอกว่าทำไม
2. **Monitoring**: ติดตามสุขภาพของ Cluster ได้แบบ Real-time
3. **Audit Trail**: บันทึกการเปลี่ยนแปลงที่เกิดขึ้นกับ Resources
4. **Alerting**: ตั้ง Alert เมื่อมี Events ที่ผิดปกติ
5. **Capacity Planning**: เข้าใจ Pattern การใช้งาน Resources

### โครงสร้างของ Event Object

```yaml
apiVersion: v1
kind: Event
metadata:
  name: my-pod.17a3b8c9d4e5f6a7
  namespace: default
  creationTimestamp: "2024-01-15T10:30:00Z"
involvedObject:
  apiVersion: v1
  kind: Pod
  name: my-pod
  namespace: default
  uid: abc123-def456-ghi789
reason: Scheduled
message: Successfully assigned default/my-pod to node-1
source:
  component: default-scheduler
  host: ""
firstTimestamp: "2024-01-15T10:30:00Z"
lastTimestamp: "2024-01-15T10:30:00Z"
count: 1
type: Normal
```

### ฟิลด์สำคัญของ Event

| ฟิลด์ | คำอธิบาย |
|-------|----------|
| `involvedObject` | Resource ที่เกี่ยวข้องกับ Event นี้ |
| `reason` | เหตุผลสั้นๆ ว่าทำไม Event นี้ถูกสร้าง |
| `message` | รายละเอียดเพิ่มเติมของ Event |
| `source` | Component ที่สร้าง Event |
| `type` | ประเภทของ Event (Normal หรือ Warning) |
| `count` | จำนวนครั้งที่ Event นี้เกิดขึ้น |
| `firstTimestamp` | เวลาที่ Event นี้เกิดขึ้นครั้งแรก |
| `lastTimestamp` | เวลาที่ Event นี้เกิดขึ้นครั้งล่าสุด |

---

## Event Types

Kubernetes แบ่ง Events ออกเป็น 2 ประเภทหลัก:

### 1. Normal Events

Events ประเภท Normal บ่งบอกถึงสิ่งที่เกิดขึ้นตามปกติใน Cluster เช่น:

- **Scheduled**: Pod ถูก Assign ให้กับ Node สำเร็จ
- **Pulling**: กำลัง Pull Container Image
- **Pulled**: Pull Container Image สำเร็จ
- **Created**: สร้าง Container สำเร็จ
- **Started**: เริ่ม Container สำเร็จ
- **Killing**: กำลังหยุด Container
- **ScalingReplicaSet**: กำลัง Scale ReplicaSet

### 2. Warning Events

Events ประเภท Warning บ่งบอกถึงปัญหาหรือสิ่งที่ผิดปกติ:

- **BackOff**: Container กำลัง Restart ซ้ำๆ (CrashLoopBackOff)
- **Failed**: การกระทำบางอย่างล้มเหลว
- **FailedScheduling**: ไม่สามารถ Schedule Pod ได้
- **FailedMount**: ไม่สามารถ Mount Volume ได้
- **Unhealthy**: Liveness หรือ Readiness Probe ล้มเหลว
- **OOMKilling**: Container ถูก Kill เนื่องจากใช้ Memory เกิน Limit
- **NodeNotReady**: Node ไม่พร้อมทำงาน

### Common Event Reasons

```
# Pod Lifecycle Events
Scheduled          - Pod ถูก Schedule บน Node
Pulled             - Container Image ถูก Pull สำเร็จ
Created            - Container ถูกสร้าง
Started            - Container เริ่มทำงาน
Killing            - Container กำลังถูกหยุด

# Warning Events
BackOff            - Container กำลัง Restart
Failed             - มีบางอย่างล้มเหลว
FailedScheduling   - ไม่สามารถ Schedule Pod ได้
FailedMount        - ไม่สามารถ Mount Volume ได้
Unhealthy          - Probe ล้มเหลว
OOMKilling         - Out of Memory
ImagePullBackOff   - ไม่สามารถ Pull Image ได้
ErrImagePull       - Error ในการ Pull Image

# Node Events
NodeReady          - Node พร้อมทำงาน
NodeNotReady       - Node ไม่พร้อมทำงาน
NodeMemoryPressure - Node มี Memory Pressure
NodeDiskPressure   - Node มี Disk Pressure
```

---

## kubectl get events

### คำสั่งพื้นฐาน

```bash
# ดู Events ทั้งหมดใน Namespace ปัจจุบัน
kubectl get events

# ดู Events ใน Namespace เฉพาะ
kubectl get events -n my-namespace

# ดู Events ทุก Namespace
kubectl get events --all-namespaces
kubectl get events -A

# ดู Events แบบ Wide (แสดงข้อมูลเพิ่มเติม)
kubectl get events -o wide

# ดู Events แบบ YAML
kubectl get events -o yaml

# ดู Events แบบ JSON
kubectl get events -o json
```

### เรียงลำดับ Events

```bash
# เรียงตาม Last Timestamp (ล่าสุดก่อน)
kubectl get events --sort-by='.lastTimestamp'

# เรียงตาม First Timestamp
kubectl get events --sort-by='.firstTimestamp'

# เรียงตาม Count (เกิดบ่อยที่สุดก่อน)
kubectl get events --sort-by='.count'
```

### Filter Events

```bash
# ดู Events ของ Resource เฉพาะ
kubectl get events --field-selector involvedObject.name=my-pod

# ดู Events ประเภท Warning เท่านั้น
kubectl get events --field-selector type=Warning

# ดู Events ของ Pod เฉพาะ
kubectl get events --field-selector involvedObject.kind=Pod,involvedObject.name=my-pod

# ดู Events จาก Component เฉพาะ
kubectl get events --field-selector source.component=kubelet

# รวม Multiple Filters
kubectl get events \
  --field-selector type=Warning,involvedObject.kind=Pod \
  --sort-by='.lastTimestamp'
```

### ดู Events ของ Resource

```bash
# ดู Events ที่เกี่ยวข้องกับ Pod
kubectl describe pod my-pod

# ดู Events ที่เกี่ยวข้องกับ Node
kubectl describe node my-node

# ดู Events ที่เกี่ยวข้องกับ Deployment
kubectl describe deployment my-deployment

# ดู Events ที่เกี่ยวข้องกับ Service
kubectl describe service my-service
```

### ตัวอย่าง Output ของ kubectl get events

```
NAMESPACE   LAST SEEN   TYPE      REASON              OBJECT                    MESSAGE
default     2m          Normal    Scheduled           Pod/nginx-abc123          Successfully assigned default/nginx-abc123 to node-1
default     2m          Normal    Pulling             Pod/nginx-abc123          Pulling image "nginx:latest"
default     1m          Normal    Pulled              Pod/nginx-abc123          Successfully pulled image "nginx:latest"
default     1m          Normal    Created             Pod/nginx-abc123          Created container nginx
default     1m          Normal    Started             Pod/nginx-abc123          Started container nginx
default     30s         Warning   BackOff             Pod/bad-pod-xyz           Back-off restarting failed container
default     30s         Warning   Failed              Pod/bad-pod-xyz           Error: container failed to start
```

### Custom Output Format

```bash
# แสดงเฉพาะฟิลด์ที่ต้องการ
kubectl get events \
  -o custom-columns=\
'NAME:.metadata.name,'\
'TYPE:.type,'\
'REASON:.reason,'\
'MESSAGE:.message,'\
'COUNT:.count'

# ใช้ JSONPath
kubectl get events \
  -o jsonpath='{range .items[*]}{.type}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'

# Watch Events แบบ Real-time
kubectl get events -w
kubectl get events --watch

# Watch Events พร้อม Sort
kubectl get events -w --sort-by='.lastTimestamp'
```

---

## Event Logging และ Alerting

### การตั้งค่า Event Retention

ค่า default ของ Kubernetes จะเก็บ Events ไว้ 1 ชั่วโมง เราสามารถปรับได้ใน kube-apiserver:

```bash
# ใน kube-apiserver manifest (/etc/kubernetes/manifests/kube-apiserver.yaml)
# เพิ่ม flag นี้:
--event-ttl=2h  # เก็บ Events ไว้ 2 ชั่วโมง
```

### Kubernetes Event Exporter

ใช้ Kubernetes Event Exporter เพื่อส่ง Events ไปยังระบบ Monitoring ภายนอก:

```yaml
# kubernetes-event-exporter-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-config
  namespace: monitoring
data:
  config.yaml: |
    logLevel: debug
    logFormat: json
    route:
      routes:
        - match:
            - receiver: "dump"
    receivers:
      - name: "dump"
        stdout: {}
```

```yaml
# kubernetes-event-exporter-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: event-exporter
  template:
    metadata:
      labels:
        app: event-exporter
    spec:
      serviceAccountName: event-exporter
      containers:
        - name: event-exporter
          image: ghcr.io/resmoio/kubernetes-event-exporter:latest
          args:
            - -conf=/data/config.yaml
          volumeMounts:
            - mountPath: /data
              name: cfg
      volumes:
        - name: cfg
          configMap:
            name: event-exporter-config
```

```yaml
# kubernetes-event-exporter-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: event-exporter
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: event-exporter
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: event-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: event-exporter
subjects:
  - kind: ServiceAccount
    name: event-exporter
    namespace: monitoring
```

### ส่ง Events ไปยัง Slack

```yaml
# event-exporter-slack-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-config
  namespace: monitoring
data:
  config.yaml: |
    logLevel: info
    logFormat: json
    route:
      routes:
        - match:
            - receiver: "slack-warnings"
          drop:
            - type: "Normal"
        - match:
            - receiver: "dump"
    receivers:
      - name: "dump"
        stdout: {}
      - name: "slack-warnings"
        slack:
          webhookUrl: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
          channel: "#kubernetes-alerts"
          color: "#ff0000"
          title: "Kubernetes Warning Event"
          text: >-
            *{{ .Type }}* - {{ .Reason }}
            Namespace: {{ .InvolvedObject.Namespace }}
            Resource: {{ .InvolvedObject.Kind }}/{{ .InvolvedObject.Name }}
            Message: {{ .Message }}
```

### Prometheus Metrics จาก Events

ติดตั้ง kube-state-metrics เพื่อ Export Event Metrics:

```yaml
# kube-state-metrics-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app.kubernetes.io/name: kube-state-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
        - name: kube-state-metrics
          image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.0
          ports:
            - name: http-metrics
              containerPort: 8080
            - name: telemetry
              containerPort: 8081
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 5
            timeoutSeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65534
            seccompProfile:
              type: RuntimeDefault
```

### Alert Rules สำหรับ Events

```yaml
# prometheus-event-alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-event-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    - name: kubernetes.events
      rules:
        # Alert เมื่อมี Pod ที่อยู่ในสถานะ CrashLoopBackOff
        - alert: KubePodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Pod กำลัง Crash Loop"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} กำลัง Restart บ่อยผิดปกติ"

        # Alert เมื่อมี Pod ที่ไม่สามารถ Schedule ได้
        - alert: KubePodNotScheduled
          expr: |
            kube_pod_status_phase{phase="Pending"} > 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod ไม่สามารถ Schedule ได้"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} ยังอยู่ในสถานะ Pending มานานกว่า 10 นาที"

        # Alert เมื่อ Node ไม่พร้อมทำงาน
        - alert: KubeNodeNotReady
          expr: |
            kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node ไม่พร้อมทำงาน"
            description: "Node {{ $labels.node }} ไม่พร้อมทำงานมานานกว่า 5 นาที"
```

---

## Workshop: Monitor Cluster Events

### เป้าหมาย

ในส่วนนี้เราจะ:
1. สร้าง Namespace และ Resources ต่างๆ
2. สังเกต Events ที่เกิดขึ้น
3. จำลองปัญหาและดู Warning Events
4. ติดตั้ง Event Exporter
5. ตั้ง Alert Rules

### ขั้นตอนที่ 1: เตรียม Namespace

```bash
# สร้าง Namespace สำหรับ Workshop
kubectl create namespace event-workshop

# Set default namespace
kubectl config set-context --current --namespace=event-workshop
```

### ขั้นตอนที่ 2: สร้าง Pod ปกติและสังเกต Events

```yaml
# normal-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: event-workshop
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "50m"
        limits:
          memory: "128Mi"
          cpu: "100m"
```

```bash
# Apply pod
kubectl apply -f normal-pod.yaml

# ดู Events ทันทีหลัง Apply
kubectl get events -n event-workshop -w &

# รอ Pod Start
kubectl wait --for=condition=Ready pod/nginx-pod -n event-workshop --timeout=60s

# ดู Events ทั้งหมด
kubectl get events -n event-workshop --sort-by='.lastTimestamp'
```

Expected Output:
```
LAST SEEN   TYPE     REASON      OBJECT          MESSAGE
5s          Normal   Scheduled   Pod/nginx-pod   Successfully assigned event-workshop/nginx-pod to node-1
4s          Normal   Pulling     Pod/nginx-pod   Pulling image "nginx:1.25"
2s          Normal   Pulled      Pod/nginx-pod   Successfully pulled image "nginx:1.25"
2s          Normal   Created     Pod/nginx-pod   Created container nginx
2s          Normal   Started     Pod/nginx-pod   Started container nginx
```

### ขั้นตอนที่ 3: จำลอง CrashLoopBackOff

```yaml
# crashing-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: crashing-pod
  namespace: event-workshop
spec:
  containers:
    - name: crashing-container
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Starting...'; sleep 5; exit 1"]
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
        limits:
          memory: "64Mi"
          cpu: "50m"
```

```bash
# Apply crashing pod
kubectl apply -f crashing-pod.yaml

# Watch Events
kubectl get events -n event-workshop --sort-by='.lastTimestamp' -w

# รอดูจนกระทั่ง CrashLoopBackOff
# ควรเห็น Warning Events เช่น:
# Warning BackOff crashing-pod Back-off restarting failed container
```

### ขั้นตอนที่ 4: จำลอง Image Pull Error

```yaml
# invalid-image-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: invalid-image-pod
  namespace: event-workshop
spec:
  containers:
    - name: invalid-container
      image: nonexistent-registry.example.com/nonexistent-image:latest
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
```

```bash
# Apply pod with invalid image
kubectl apply -f invalid-image-pod.yaml

# ดู Events
kubectl get events -n event-workshop \
  --field-selector involvedObject.name=invalid-image-pod \
  --sort-by='.lastTimestamp'

# ควรเห็น:
# Warning   Failed    Pod/invalid-image-pod   Failed to pull image
# Warning   Failed    Pod/invalid-image-pod   Error: ErrImagePull
# Warning   BackOff   Pod/invalid-image-pod   Back-off pulling image
```

### ขั้นตอนที่ 5: จำลอง Resource Constraint

```yaml
# resource-constrained-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-constrained-pod
  namespace: event-workshop
spec:
  containers:
    - name: memory-hog
      image: busybox:1.36
      command: ["sh", "-c", "while true; do cat /dev/urandom | head -c 10M; done"]
      resources:
        requests:
          memory: "32Mi"
          cpu: "10m"
        limits:
          memory: "64Mi"  # จำกัด Memory ให้น้อยกว่าที่ใช้จริง
          cpu: "50m"
```

```bash
# Apply resource constrained pod
kubectl apply -f resource-constrained-pod.yaml

# ดู Events - ควรเห็น OOMKilling Event
kubectl get events -n event-workshop \
  --field-selector involvedObject.name=resource-constrained-pod \
  --sort-by='.lastTimestamp'
```

### ขั้นตอนที่ 6: สร้าง Script สำหรับ Monitor Events

```bash
#!/bin/bash
# monitor-events.sh

NAMESPACE="${1:-default}"
INTERVAL="${2:-5}"
LOG_FILE="/tmp/k8s-events-$(date +%Y%m%d-%H%M%S).log"

echo "กำลัง Monitor Events ใน Namespace: $NAMESPACE"
echo "Log ถูกบันทึกที่: $LOG_FILE"
echo "กด Ctrl+C เพื่อหยุด"
echo "---"

# Function สำหรับ Format Event
format_event() {
    local type="$1"
    local reason="$2"
    local object="$3"
    local message="$4"
    local timestamp="$5"
    
    if [ "$type" = "Warning" ]; then
        echo -e "\033[31m[WARNING]\033[0m [$timestamp] $object - $reason: $message"
    else
        echo -e "\033[32m[NORMAL]\033[0m [$timestamp] $object - $reason: $message"
    fi
}

# Watch Events แบบ Real-time
kubectl get events -n "$NAMESPACE" -w \
    --sort-by='.lastTimestamp' \
    -o custom-columns=\
'TYPE:.type,'\
'REASON:.reason,'\
'OBJECT:.involvedObject.name,'\
'MESSAGE:.message,'\
'TIME:.lastTimestamp' | \
while IFS= read -r line; do
    echo "$line" | tee -a "$LOG_FILE"
done
```

```bash
# ทำให้ Script Executable
chmod +x monitor-events.sh

# รัน Script
./monitor-events.sh event-workshop
```

### ขั้นตอนที่ 7: ติดตั้ง Event Exporter

```bash
# สร้าง Namespace สำหรับ Monitoring
kubectl create namespace monitoring

# Apply RBAC
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: event-exporter
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: event-exporter
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: event-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: event-exporter
subjects:
  - kind: ServiceAccount
    name: event-exporter
    namespace: monitoring
EOF

# Apply ConfigMap
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-config
  namespace: monitoring
data:
  config.yaml: |
    logLevel: info
    logFormat: json
    route:
      routes:
        - match:
            - receiver: "stdout"
    receivers:
      - name: "stdout"
        stdout: {}
EOF

# Apply Deployment
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: event-exporter
  template:
    metadata:
      labels:
        app: event-exporter
    spec:
      serviceAccountName: event-exporter
      containers:
        - name: event-exporter
          image: ghcr.io/resmoio/kubernetes-event-exporter:latest
          args:
            - -conf=/data/config.yaml
          volumeMounts:
            - mountPath: /data
              name: cfg
      volumes:
        - name: cfg
          configMap:
            name: event-exporter-config
EOF
```

### ขั้นตอนที่ 8: ดู Event Exporter Logs

```bash
# ดู Logs ของ Event Exporter
kubectl logs -f deployment/event-exporter -n monitoring

# ควรเห็น Events ในรูปแบบ JSON:
# {"level":"info","ts":"2024-01-15T10:30:00Z","msg":"Event","type":"Normal","reason":"Started",...}
```

### ขั้นตอนที่ 9: สร้าง Custom Event ด้วย Script

```bash
# create-custom-event.sh
#!/bin/bash

# สร้าง Custom Event โดยใช้ kubectl
NAMESPACE="${1:-default}"
RESOURCE_NAME="${2:-my-resource}"
REASON="${3:-CustomReason}"
MESSAGE="${4:-This is a custom event}"
EVENT_TYPE="${5:-Normal}"

kubectl create event \
  --for=Pod/$RESOURCE_NAME \
  --reason="$REASON" \
  --message="$MESSAGE" \
  --type="$EVENT_TYPE" \
  -n "$NAMESPACE" \
  "${RESOURCE_NAME}-$(date +%s)"
```

### ขั้นตอนที่ 10: สร้าง Event Monitoring Dashboard Script

```bash
#!/bin/bash
# event-dashboard.sh

# แสดง Dashboard ของ Events

NAMESPACE="${1:---all-namespaces}"
FLAGS=""

if [ "$NAMESPACE" != "--all-namespaces" ]; then
    FLAGS="-n $NAMESPACE"
fi

clear
echo "============================================"
echo "        Kubernetes Events Dashboard         "
echo "============================================"
echo "Time: $(date)"
echo ""

echo "--- Warning Events (ล่าสุด 10 รายการ) ---"
kubectl get events $FLAGS \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp' \
  -o custom-columns=\
'LAST SEEN:.lastTimestamp,'\
'TYPE:.type,'\
'REASON:.reason,'\
'OBJECT:.involvedObject.name,'\
'COUNT:.count,'\
'MESSAGE:.message' \
  2>/dev/null | tail -11

echo ""
echo "--- Normal Events (ล่าสุด 5 รายการ) ---"
kubectl get events $FLAGS \
  --field-selector type=Normal \
  --sort-by='.lastTimestamp' \
  -o custom-columns=\
'LAST SEEN:.lastTimestamp,'\
'REASON:.reason,'\
'OBJECT:.involvedObject.name,'\
'MESSAGE:.message' \
  2>/dev/null | tail -6

echo ""
echo "--- Event Summary ---"
echo "Total Normal Events: $(kubectl get events $FLAGS --field-selector type=Normal 2>/dev/null | tail -n +2 | wc -l)"
echo "Total Warning Events: $(kubectl get events $FLAGS --field-selector type=Warning 2>/dev/null | tail -n +2 | wc -l)"

echo ""
echo "กด Ctrl+C เพื่อออก หรือรอ 10 วินาทีเพื่อ Refresh"
```

### ขั้นตอนที่ 11: ทำความสะอาด

```bash
# ลบ Resources ที่สร้างใน Workshop
kubectl delete namespace event-workshop
kubectl delete namespace monitoring

# Reset default namespace
kubectl config set-context --current --namespace=default
```

### สรุปสิ่งที่เรียนรู้

1. **Event Types**: Normal Events บอกสถานะปกติ, Warning Events บอกปัญหา
2. **kubectl get events**: คำสั่งหลักสำหรับดู Events
3. **Filtering**: ใช้ `--field-selector` เพื่อ Filter Events ที่ต้องการ
4. **Sorting**: ใช้ `--sort-by` เพื่อเรียงลำดับ Events
5. **Real-time Monitoring**: ใช้ `-w` เพื่อ Watch Events แบบ Real-time
6. **Event Exporter**: ส่ง Events ไปยังระบบภายนอกเพื่อ Persistent Storage
7. **Alerting**: ตั้ง Alert Rules บนพื้นฐานของ Events

### Tips และ Best Practices

```bash
# Tip 1: ใช้ watch เพื่อดู Events แบบ Real-time
watch -n 2 'kubectl get events --sort-by=".lastTimestamp" | tail -20'

# Tip 2: Filter เฉพาะ Warning Events ที่เกิดใน 5 นาทีที่แล้ว
kubectl get events --field-selector type=Warning \
  --sort-by='.lastTimestamp' | \
  awk -v d="$(date -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S)" \
  'NR==1 || $1 >= d'

# Tip 3: ส่ง Events ไปยัง File
kubectl get events --all-namespaces \
  --sort-by='.lastTimestamp' \
  -o json > /tmp/k8s-events-$(date +%Y%m%d).json

# Tip 4: ดู Events ที่มี Count สูง (เกิดซ้ำๆ)
kubectl get events --sort-by='.count' | tail -10

# Tip 5: Monitor Events ของ Deployment ใหม่
kubectl get events -w \
  --field-selector involvedObject.kind=ReplicaSet &
kubectl apply -f my-deployment.yaml
```

---

## สรุป

Kubernetes Events เป็นเครื่องมือที่ทรงพลังสำหรับการ Monitor และ Debug Cluster ของเรา โดยการเข้าใจ Event Types และการใช้ kubectl get events อย่างมีประสิทธิภาพ เราสามารถ:

- ตรวจจับปัญหาได้รวดเร็ว
- เข้าใจ Lifecycle ของ Resources
- ตั้ง Alert เมื่อมีสิ่งผิดปกติ
- Debug ปัญหาที่ซับซ้อน

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับการใช้ kubectl logs สำหรับการดู Logs ของ Containers ซึ่งเป็นอีกหนึ่งเครื่องมือสำคัญในการ Monitor Kubernetes Cluster
