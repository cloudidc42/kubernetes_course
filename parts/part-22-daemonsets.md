# Part 22: DaemonSets - รัน Pod บนทุก Node

## สารบัญ
1. [DaemonSet คืออะไร](#daemonset-คืออะไร)
2. [Use Cases](#use-cases)
3. [DaemonSet YAML ละเอียด](#daemonset-yaml-ละเอียด)
4. [Node Selection และ Tolerations](#node-selection-และ-tolerations)
5. [Rolling Update DaemonSet](#rolling-update-daemonset)
6. [Workshop: Deploy Fluentd บนทุก Node](#workshop-deploy-fluentd-บนทุก-node)
7. [Workshop: Deploy Node Monitoring](#workshop-deploy-node-monitoring)
8. [Workshop: Deploy Log Collector](#workshop-deploy-log-collector)
9. [Troubleshooting DaemonSets](#troubleshooting-daemonsets)
10. [Best Practices](#best-practices)

---

## DaemonSet คืออะไร

**DaemonSet** เป็น Kubernetes workload resource ที่รับประกันว่า Pod จะรันบน**ทุก Node** (หรือ Node ที่เลือก) ในคลัสเตอร์:

- เมื่อ Node ใหม่ถูกเพิ่มเข้าคลัสเตอร์ → Pod จะถูกสร้างบน Node นั้นโดยอัตโนมัติ
- เมื่อ Node ถูกลบออก → Pod บน Node นั้นจะถูกลบด้วย
- ไม่ต้องกำหนด `replicas` เพราะ Kubernetes จัดการเอง

### ภาพรวมการทำงาน

```
Kubernetes Cluster:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Node-1          Node-2          Node-3          Node-4    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐  │
│  │ fluentd  │   │ fluentd  │   │ fluentd  │   │fluentd │  │
│  │  pod     │   │  pod     │   │  pod     │   │  pod   │  │
│  └──────────┘   └──────────┘   └──────────┘   └────────┘  │
│                                                             │
│  DaemonSet "fluentd": รัน 1 Pod บนทุก Node                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘

เพิ่ม Node-5 → DaemonSet สร้าง fluentd pod บน Node-5 อัตโนมัติ!
ลบ Node-2  → DaemonSet ลบ fluentd pod บน Node-2 อัตโนมัติ!
```

---

## Use Cases

### 1. Log Collection

```
Node → /var/log/*.log → Fluentd Pod → Elasticsearch/Loki/S3
```
- **Fluentd**: รวม logs จาก containers ทุกตัวบน Node
- **Filebeat (Elastic)**: ส่ง logs ไปยัง Elasticsearch
- **Promtail (Grafana)**: ส่ง logs ไปยัง Loki

### 2. Node Monitoring

```
Node Metrics (CPU/Memory/Disk) → Node Exporter Pod → Prometheus
```
- **Prometheus Node Exporter**: expose hardware/OS metrics
- **Datadog Agent**: monitoring และ APM
- **New Relic Agent**: monitoring

### 3. Network Plugins (CNI)

```
Calico, Flannel, Weave Net → รันเป็น DaemonSet บนทุก Node
```

### 4. Storage Plugins (CSI)

```
CSI Node Driver → รันเป็น DaemonSet
```

### 5. Security Agents

```
Falco (intrusion detection), Twistlock, Aqua Security → DaemonSet
```

### 6. Node Configuration

```
ตั้งค่า Node-level config: iptables rules, kernel parameters → DaemonSet
```

---

## DaemonSet YAML ละเอียด

### Basic DaemonSet

```yaml
# basic-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
  namespace: monitoring
  labels:
    app: node-logger
    version: v1
spec:
  selector:
    matchLabels:
      app: node-logger
  
  # Update Strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # อัปเดตทีละ 1 Pod
      maxSurge: 0  # ไม่สร้าง Pod เกิน (DaemonSet ใช้ maxUnavailable เป็นหลัก)
  
  # Pod template
  template:
    metadata:
      labels:
        app: node-logger
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      # ให้ Pod เข้าถึง Node filesystem
      hostNetwork: false  # ใช้ host network (สำหรับ network monitoring)
      hostPID: false      # ใช้ host PID namespace
      
      # ให้ Pod schedule บน master nodes ด้วย (ถ้าต้องการ)
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      containers:
      - name: node-logger
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          while true; do
            echo "$(date) - Node: $NODE_NAME - Checking disk usage..."
            df -h /host/
            sleep 60
          done
        
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        resources:
          requests:
            cpu: "50m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "100Mi"
        
        # Mount host filesystem (read-only)
        volumeMounts:
        - name: host-root
          mountPath: /host
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      
      # Termination grace period
      terminationGracePeriodSeconds: 30
      
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

---

## Node Selection และ Tolerations

### เลือก Node ด้วย nodeSelector

```yaml
spec:
  template:
    spec:
      # รันเฉพาะบน Node ที่มี label นี้
      nodeSelector:
        node-type: worker
        disktype: ssd
```

```bash
# เพิ่ม label ให้ Node
kubectl label node node-1 node-type=worker disktype=ssd
kubectl label node node-2 node-type=worker disktype=hdd

# DaemonSet จะรันเฉพาะบน node-1 (มี disktype=ssd)
```

### เลือก Node ด้วย nodeAffinity

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
              - key: node-role
                operator: In
                values:
                - worker
                - database
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
```

### Tolerations - รัน Pod บน Tainted Nodes

```bash
# ดู taints บน node
kubectl describe node node-1 | grep Taint

# เพิ่ม taint ให้ node
kubectl taint nodes node-1 app=gpu:NoSchedule
kubectl taint nodes master node-role.kubernetes.io/master:NoSchedule
```

```yaml
# DaemonSet ที่ต้องการรันบน GPU nodes
spec:
  template:
    spec:
      tolerations:
      # รันบน GPU node ที่มี taint
      - key: app
        value: gpu
        operator: Equal
        effect: NoSchedule
      
      # รันบน master node
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      # รันบน unschedulable node
      - key: node.kubernetes.io/unschedulable
        operator: Exists
        effect: NoSchedule
      
      # รันได้แม้ node memory pressure
      - key: node.kubernetes.io/memory-pressure
        operator: Exists
        effect: NoSchedule
      
      # รันได้แม้ node disk pressure
      - key: node.kubernetes.io/disk-pressure
        operator: Exists
        effect: NoSchedule
```

---

## Rolling Update DaemonSet

### RollingUpdate Strategy

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # จำนวน Pod ที่สามารถหยุดทำงานได้ขณะ update
      maxUnavailable: 1  # ตัวเลข หรือ เปอร์เซ็นต์ "10%"
```

### OnDelete Strategy (Manual Update)

```yaml
spec:
  updateStrategy:
    type: OnDelete
    # Pod จะอัปเดตเฉพาะเมื่อถูกลบและสร้างใหม่
```

### ดำเนินการ Update

```bash
# Update image ด้วย kubectl set image
kubectl set image daemonset/fluentd fluentd=fluent/fluentd:v1.15.0 -n logging

# Update ด้วย kubectl apply
kubectl apply -f fluentd-daemonset.yaml

# ดู rollout status
kubectl rollout status daemonset/fluentd -n logging

# ดู rollout history
kubectl rollout history daemonset/fluentd -n logging

# Rollback
kubectl rollout undo daemonset/fluentd -n logging

# Rollback ไปยัง version ที่ต้องการ
kubectl rollout undo daemonset/fluentd --to-revision=2 -n logging

# Pause rollout
kubectl rollout pause daemonset/fluentd -n logging

# Resume rollout
kubectl rollout resume daemonset/fluentd -n logging
```

---

## Workshop: Deploy Fluentd บนทุก Node

เป้าหมาย: Deploy Fluentd เพื่อรวบรวม logs จากทุก Node และส่งไปยัง Elasticsearch

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Log Collection Architecture               │
│                                                              │
│  Node-1           Node-2           Node-3                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │ Fluentd  │    │ Fluentd  │    │ Fluentd  │               │
│  │  DaemonSet    │  DaemonSet    │  DaemonSet                │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘               │
│       │               │               │                      │
│       └───────────────┴───────────────┘                      │
│                       │                                      │
│                       ▼                                      │
│              ┌─────────────────┐                            │
│              │  Elasticsearch  │                            │
│              └─────────────────┘                            │
│                       │                                      │
│                       ▼                                      │
│              ┌─────────────────┐                            │
│              │     Kibana      │                            │
│              └─────────────────┘                            │
└──────────────────────────────────────────────────────────────┘
```

### 1. สร้าง Namespace

```bash
kubectl create namespace logging
```

### 2. สร้าง Elasticsearch (simplified)

```yaml
# elasticsearch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: elasticsearch:8.5.0
        env:
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        - name: xpack.security.enabled
          value: "false"
        ports:
        - containerPort: 9200
        - containerPort: 9300
        resources:
          limits:
            memory: "1Gi"
            cpu: "1"
          requests:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
spec:
  selector:
    app: elasticsearch
  ports:
  - name: rest
    port: 9200
    targetPort: 9200
  - name: inter-node
    port: 9300
    targetPort: 9300
```

### 3. สร้าง ConfigMap สำหรับ Fluentd

```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # Input: อ่าน logs จาก container log files
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      
      <parse>
        @type multi_format
        
        <pattern>
          format json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>
    
    # Input: system logs
    <source>
      @type systemd
      tag host.systemd
      path /var/log/journal
      read_from_head true
      
      <storage>
        @type local
        persistent true
        path /var/log/fluentd-journald-cursor.json
      </storage>
      
      <entry>
        fields_strip_underscores true
        fields_lowercase true
      </entry>
    </source>
    
    # Filter: เพิ่ม Kubernetes metadata
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "https://#{ENV['KUBERNETES_SERVICE_HOST']}:#{ENV['KUBERNETES_SERVICE_PORT_HTTPS']}"
      verify_ssl true
      ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file /var/run/secrets/kubernetes.io/serviceaccount/token
      skip_labels false
      skip_container_metadata false
      skip_master_url false
      skip_namespace_metadata false
    </filter>
    
    # Filter: parse JSON logs
    <filter kubernetes.**>
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      
      <parse>
        @type multi_format
        
        <pattern>
          format json
        </pattern>
        
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>
    
    # Filter: เพิ่ม node name
    <filter **>
      @type record_transformer
      <record>
        node_name "#{ENV['NODE_NAME']}"
        cluster_name "my-k8s-cluster"
      </record>
    </filter>
    
    # Output: ส่งไปยัง Elasticsearch
    <match kubernetes.**>
      @type elasticsearch
      @id out_es
      @log_level info
      
      host elasticsearch.logging.svc.cluster.local
      port 9200
      
      # Index pattern
      index_name fluentd-kubernetes
      type_name _doc
      
      # Use logstash format สำหรับ time-based indices
      logstash_format true
      logstash_prefix kubernetes
      logstash_dateformat %Y.%m.%d
      
      # Reconnection
      reconnect_on_error true
      reload_on_failure true
      reload_connections false
      
      # Buffer configuration
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever false
        retry_max_interval 30
        chunk_limit_size 2M
        total_limit_size 500M
        overflow_action block
      </buffer>
    </match>
    
    # Output: ส่ง system logs ไปยัง Elasticsearch
    <match host.systemd.**>
      @type elasticsearch
      @id out_es_system
      
      host elasticsearch.logging.svc.cluster.local
      port 9200
      
      logstash_format true
      logstash_prefix systemd
      
      <buffer>
        @type file
        path /var/log/fluentd-buffers/systemd.buffer
        flush_interval 5s
      </buffer>
    </match>
    
    # Output: stdout สำหรับ debug
    <match **>
      @type stdout
    </match>
```

### 4. สร้าง ServiceAccount และ RBAC

```yaml
# fluentd-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: logging
```

### 5. สร้าง DaemonSet สำหรับ Fluentd

```yaml
# fluentd-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    k8s-app: fluentd-logging
    version: v1
spec:
  selector:
    matchLabels:
      name: fluentd
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  
  template:
    metadata:
      labels:
        name: fluentd
        k8s-app: fluentd-logging
        version: v1
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      
      # tolerations เพื่อให้รันบน master nodes ด้วย
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - operator: "Exists"
        effect: "NoExecute"
      - operator: "Exists"
        effect: "NoSchedule"
      
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.15-debian-elasticsearch8-1
        
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENT_UID
          value: "0"
        - name: FLUENTD_SYSTEMD_CONF
          value: "disable"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        
        resources:
          limits:
            memory: 200Mi
            cpu: 200m
          requests:
            cpu: 100m
            memory: 200Mi
        
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc/fluent.conf
          subPath: fluent.conf
        - name: fluentd-buffer
          mountPath: /var/log/fluentd-buffers
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /api/health.check
            port: 24220
          initialDelaySeconds: 60
          periodSeconds: 60
          timeoutSeconds: 10
          failureThreshold: 5
        
        ports:
        - containerPort: 24224
          name: forward
          protocol: TCP
        - containerPort: 24220
          name: http-input
          protocol: TCP
      
      terminationGracePeriodSeconds: 30
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
      - name: fluentd-buffer
        hostPath:
          path: /var/log/fluentd-buffers
          type: DirectoryOrCreate
```

### 6. Deploy และทดสอบ

```bash
# Apply ทั้งหมด
kubectl apply -f elasticsearch.yaml
kubectl apply -f fluentd-rbac.yaml
kubectl apply -f fluentd-config.yaml
kubectl apply -f fluentd-daemonset.yaml

# รอให้ Elasticsearch พร้อม
kubectl wait --for=condition=ready pod -l app=elasticsearch -n logging --timeout=120s

# ดู DaemonSet status
kubectl get daemonset -n logging

# ดู pods บนแต่ละ node
kubectl get pods -n logging -o wide

# ดู logs ของ fluentd
kubectl logs -l name=fluentd -n logging --tail=50

# ทดสอบ: สร้าง Pod ที่ generate logs
kubectl run log-generator \
  --image=busybox \
  -n default \
  -- sh -c 'while true; do echo "$(date) Test log message from $HOSTNAME"; sleep 5; done'

# ตรวจสอบว่า Elasticsearch ได้รับ logs
kubectl port-forward svc/elasticsearch 9200:9200 -n logging &
curl -s http://localhost:9200/_cat/indices?v | grep kubernetes
curl -s http://localhost:9200/kubernetes-*/_count
```

---

## Workshop: Deploy Node Monitoring (Prometheus Node Exporter)

### 1. สร้าง DaemonSet สำหรับ Node Exporter

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
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
        prometheus.io/path: "/metrics"
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.5.0
        
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+)($|/)
        - --collector.filesystem.ignored-fs-types=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
        - --web.listen-address=:9100
        
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: metrics
          protocol: TCP
        
        resources:
          limits:
            cpu: 250m
            memory: 180Mi
          requests:
            cpu: 102m
            memory: 180Mi
        
        volumeMounts:
        - mountPath: /host/proc
          name: proc
          readOnly: true
        - mountPath: /host/sys
          name: sys
          readOnly: true
        - mountPath: /host/root
          name: root
          readOnly: true
          mountPropagation: HostToContainer
        
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - SYS_TIME
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 65534
      
      volumes:
      - hostPath:
          path: /proc
        name: proc
      - hostPath:
          path: /sys
        name: sys
      - hostPath:
          path: /
        name: root
---
# Service สำหรับ scrape metrics
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9100"
spec:
  clusterIP: None
  selector:
    app: node-exporter
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
```

### 2. ทดสอบ Node Metrics

```bash
# Apply DaemonSet
kubectl apply -f node-exporter-daemonset.yaml

# ดู pods บนทุก node
kubectl get pods -n monitoring -o wide -l app=node-exporter

# port-forward เพื่อดู metrics
kubectl port-forward pod/node-exporter-<pod-id> 9100:9100 -n monitoring &

# ดู metrics
curl http://localhost:9100/metrics | head -50

# ดู CPU metrics
curl http://localhost:9100/metrics | grep node_cpu_seconds_total | head -10

# ดู Memory metrics
curl http://localhost:9100/metrics | grep node_memory_MemAvailable_bytes

# ดู Disk metrics
curl http://localhost:9100/metrics | grep node_filesystem_size_bytes
```

---

## Workshop: Deploy Log Collector (Promtail สำหรับ Loki)

```yaml
# promtail-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: logging
  labels:
    app: promtail
spec:
  selector:
    matchLabels:
      app: promtail
  
  updateStrategy:
    type: RollingUpdate
  
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      containers:
      - name: promtail
        image: grafana/promtail:2.8.0
        
        args:
        - -config.file=/etc/promtail/config.yml
        - -config.expand-env=true
        
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        ports:
        - containerPort: 3101
          name: http-metrics
        
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: run
          mountPath: /run/promtail
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: pods
          mountPath: /var/log/pods
          readOnly: true
        
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 128Mi
        
        readinessProbe:
          httpGet:
            path: /ready
            port: http-metrics
          failureThreshold: 5
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        
        livenessProbe:
          httpGet:
            path: /ready
            port: http-metrics
          failureThreshold: 5
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
      
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: run
        hostPath:
          path: /run/promtail
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
      - name: pods
        hostPath:
          path: /var/log/pods
```

### Promtail ConfigMap

```yaml
# promtail-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: logging
data:
  config.yml: |
    server:
      http_listen_port: 3101
      grpc_listen_port: 0
    
    positions:
      filename: /run/promtail/positions.yaml
    
    clients:
    - url: http://loki.logging.svc.cluster.local:3100/loki/api/v1/push
    
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      
      pipeline_stages:
      - docker: {}
      - cri: {}
      
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_controller_name
        regex: ([0-9a-z-.]+?)(-[0-9a-f]{8,10})?
        action: replace
        target_label: __tmp_controller_name
      
      - source_labels:
        - __meta_kubernetes_pod_label_app_kubernetes_io_name
        - __meta_kubernetes_pod_label_app
        - __tmp_controller_name
        - __meta_kubernetes_pod_name
        regex: ^;*([^;]+)(;.*)?$
        action: replace
        target_label: app
      
      - source_labels:
        - __meta_kubernetes_pod_node_name
        action: replace
        target_label: node_name
      
      - source_labels:
        - __meta_kubernetes_namespace
        action: replace
        target_label: namespace
      
      - source_labels:
        - __meta_kubernetes_pod_name
        action: replace
        target_label: pod
      
      - source_labels:
        - __meta_kubernetes_pod_container_name
        action: replace
        target_label: container
      
      - replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_uid
        - __meta_kubernetes_pod_container_name
        target_label: __path__
      
      - action: replace
        regex: true/(.*)
        replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_annotationpresent_kubernetes_io_config_hash
        - __meta_kubernetes_pod_annotation_kubernetes_io_config_hash
        - __meta_kubernetes_pod_container_name
        target_label: __path__
```

---

## Troubleshooting DaemonSets

### ปัญหา: Pod ไม่รันบาง Node

```bash
# ดู DaemonSet status
kubectl get daemonset -n logging
# NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# fluentd   3         2         2       2            2           <none>

# ดูว่า Pod รันบน Node ไหน
kubectl get pods -n logging -o wide -l name=fluentd

# ดู Node ที่ไม่มี Pod
kubectl describe node node-3 | grep Taints
# อาจมี taint ที่ DaemonSet ไม่ tolerate

# ตรวจสอบ nodeSelector
kubectl get ds fluentd -n logging -o yaml | grep -A 5 nodeSelector

# ดู events
kubectl describe ds fluentd -n logging
```

### ปัญหา: Pod Pending/CrashLoopBackOff

```bash
# ดู logs
kubectl logs daemonset/fluentd -n logging

# ดู logs ของ specific pod
kubectl logs fluentd-abc123 -n logging --previous

# ดู events ของ pod
kubectl describe pod fluentd-abc123 -n logging

# ตรวจสอบ resource limits
kubectl top pods -n logging
```

### คำสั่ง DaemonSet ที่ใช้บ่อย

```bash
# ดู DaemonSet ทั้งหมด
kubectl get daemonset -A

# ดู DaemonSet detail
kubectl describe daemonset fluentd -n logging

# ดู Pod ทั้งหมดของ DaemonSet
kubectl get pods -n logging -l name=fluentd -o wide

# ดู YAML ของ DaemonSet
kubectl get daemonset fluentd -n logging -o yaml

# Force delete Pod (จะถูกสร้างใหม่อัตโนมัติ)
kubectl delete pod fluentd-node1 -n logging

# Scale down DaemonSet (ทำไม่ได้โดยตรง, ต้องใช้ node label)
kubectl label node node-1 logging=disabled
# ตั้ง nodeSelector: { logging: enabled } ใน DaemonSet

# ลบ DaemonSet
kubectl delete daemonset fluentd -n logging
```

---

## Best Practices

### 1. ใช้ Resource Limits เสมอ

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "100Mi"
  limits:
    cpu: "200m"
    memory: "200Mi"
```

DaemonSet รันบนทุก Node ดังนั้น resource ที่ใช้จะ x จำนวน Nodes

### 2. ใช้ Tolerations เพื่อรันบน Master Nodes (ถ้าต้องการ)

```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
- key: node.kubernetes.io/unreachable
  operator: Exists
  effect: NoExecute
```

### 3. ใช้ Rolling Update อย่างระมัดระวัง

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 10%  # ไม่ให้ Node เกิน 10% ไม่มี log collector
```

### 4. Mount Host Paths อย่างระวัง

```yaml
# Mount แบบ read-only เมื่อทำได้
volumeMounts:
- name: varlog
  mountPath: /var/log
  readOnly: true  # ป้องกัน container เขียน host

# กำหนด type เพื่อป้องกัน error
volumes:
- name: varlog
  hostPath:
    path: /var/log
    type: Directory  # ต้องมี directory นี้อยู่แล้ว
    # type: DirectoryOrCreate  # สร้างถ้ายังไม่มี
    # type: File  # ต้องเป็น file
    # type: Socket  # Unix socket
```

### 5. ใช้ Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65534
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # เฉพาะที่จำเป็น
```

---

## สรุป

DaemonSet เป็นเครื่องมือสำคัญสำหรับ:

1. **Log Collection**: Fluentd, Filebeat, Promtail บนทุก Node
2. **Monitoring**: Node Exporter สำหรับ Prometheus
3. **Network**: CNI plugins (Calico, Flannel)
4. **Security**: Falco, security agents
5. **Node Configuration**: ตั้งค่า kernel params, network rules

ข้อสำคัญ:
- Pod ถูกสร้างอัตโนมัติบน Node ใหม่
- ไม่มี `replicas` field
- ใช้ `tolerations` สำหรับ tainted nodes
- ใช้ `nodeSelector/nodeAffinity` สำหรับเลือก Node เฉพาะ
- Resource usage = resource per pod × number of nodes

ในบทต่อไป เราจะเรียนรู้เรื่อง **Jobs และ CronJobs** สำหรับ batch processing
