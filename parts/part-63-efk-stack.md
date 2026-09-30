# Part 63: EFK Stack - Centralized Logging

## EFK Stack คืออะไร

EFK Stack คือชุดของ Tools 3 ตัวที่ทำงานร่วมกันเพื่อสร้างระบบ Centralized Logging สำหรับ Kubernetes:

- **E** = **Elasticsearch**: Search Engine และ Data Store สำหรับ Logs
- **F** = **Fluentd**: Log Collector และ Processor
- **K** = **Kibana**: Visualization Dashboard

บางครั้งอาจเห็น **ELK Stack** ซึ่งใช้ **Logstash** แทน Fluentd หรือ **EFK with Fluent Bit** ซึ่งใช้ Fluent Bit (เบากว่า Fluentd) แทน

### ทำไมต้องใช้ EFK Stack

| ปัญหา | EFK แก้ได้อย่างไร |
|-------|-----------------|
| Logs กระจายอยู่ใน Pods หลายตัว | Fluentd รวบรวม Logs จากทุก Pods |
| Logs หายไปเมื่อ Pod ถูกลบ | Elasticsearch เก็บ Logs ถาวร |
| ค้นหา Logs ยาก | Kibana มี Search Interface |
| ไม่มี Visualization | Kibana สร้าง Dashboard และ Graphs |
| Logs ไม่มี Context | Fluentd เพิ่ม Metadata เช่น Pod Name, Namespace |

### Architecture ของ EFK Stack

```
+------------------+     +------------------+     +------------------+
|   Applications   |     |    Fluentd       |     |  Elasticsearch   |
|   (Pods/Nodes)   | --> |   (DaemonSet)    | --> |   (StatefulSet)  |
+------------------+     +------------------+     +------------------+
                                                          |
                                                   +------v-------+
                                                   |    Kibana    |
                                                   |  (Deployment)|
                                                   +--------------+
```

**Data Flow**:
1. Applications เขียน Logs ไปยัง stdout/stderr
2. Container Runtime บันทึก Logs ลงใน Node's filesystem
3. Fluentd DaemonSet อ่าน Log Files จาก Node
4. Fluentd เพิ่ม Metadata และส่งไปยัง Elasticsearch
5. Kibana Query Elasticsearch และแสดง Visualization

---

## Deploy EFK Stack บน Kubernetes

### ข้อกำหนดเบื้องต้น

```bash
# ตรวจสอบ Resources ที่มี
kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory'

# EFK Stack ต้องการ Resources ประมาณ:
# Elasticsearch: 2 CPU, 4GB RAM (minimum)
# Fluentd: 200m CPU, 200Mi RAM per node
# Kibana: 500m CPU, 1GB RAM
```

### ขั้นตอนที่ 1: สร้าง Namespace

```bash
# สร้าง Namespace สำหรับ Logging
kubectl create namespace logging

# Set labels สำหรับ Network Policy
kubectl label namespace logging \
  kubernetes.io/metadata.name=logging \
  app.kubernetes.io/component=logging
```

### ขั้นตอนที่ 2: Deploy Elasticsearch

```yaml
# elasticsearch-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - name: rest
      port: 9200
    - name: inter-node
      port: 9300
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-service
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
    - name: rest
      port: 9200
      targetPort: 9200
  type: ClusterIP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  serviceName: elasticsearch
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
          image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
          ports:
            - containerPort: 9200
              name: rest
            - containerPort: 9300
              name: inter-node
          env:
            - name: cluster.name
              value: kubernetes-logging
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: discovery.type
              value: single-node
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
            - name: xpack.security.enabled
              value: "false"
            - name: xpack.security.enrollment.enabled
              value: "false"
          resources:
            limits:
              cpu: 1000m
              memory: 2Gi
            requests:
              cpu: 100m
              memory: 512Mi
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          readinessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 5
      initContainers:
        - name: fix-permissions
          image: busybox:1.36
          command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          securityContext:
            privileged: true
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: busybox:1.36
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: busybox:1.36
          command: ["sh", "-c", "ulimit -n 65536"]
          securityContext:
            privileged: true
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: elasticsearch
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

```bash
# Apply Elasticsearch
kubectl apply -f elasticsearch-statefulset.yaml

# รอให้ Elasticsearch Ready
kubectl wait --for=condition=Ready pod/elasticsearch-0 \
  -n logging --timeout=300s

# ทดสอบ Elasticsearch
kubectl port-forward -n logging svc/elasticsearch-service 9200:9200 &
sleep 3
curl http://localhost:9200/_cluster/health?pretty
kill %1
```

### ขั้นตอนที่ 3: Deploy Kibana

```yaml
# kibana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:8.11.0
          ports:
            - containerPort: 5601
          env:
            - name: ELASTICSEARCH_HOSTS
              value: http://elasticsearch-service:9200
            - name: SERVER_NAME
              value: kibana
          resources:
            limits:
              cpu: 500m
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /api/status
              port: 5601
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /api/status
              port: 5601
            initialDelaySeconds: 60
            periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    app: kibana
  ports:
    - port: 5601
      targetPort: 5601
  type: ClusterIP
```

```bash
# Apply Kibana
kubectl apply -f kibana-deployment.yaml

# รอให้ Kibana Ready
kubectl wait --for=condition=Available deployment/kibana \
  -n logging --timeout=300s
```

### ขั้นตอนที่ 4: Deploy Fluentd

```yaml
# fluentd-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # Input: อ่าน Container Logs จาก Node
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

    # Filter: เพิ่ม Kubernetes Metadata
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "https://#{ENV['KUBERNETES_SERVICE_HOST']}:#{ENV['KUBERNETES_SERVICE_PORT']}/api"
      verify_ssl true
      ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      skip_labels false
      skip_container_metadata false
      skip_master_url false
      skip_namespace_metadata false
    </filter>

    # Filter: ลบ Logs ที่ไม่ต้องการ
    <filter kubernetes.**>
      @type grep
      <exclude>
        key $.kubernetes.namespace_name
        pattern /^kube-system$/
      </exclude>
    </filter>

    # Filter: Parse JSON Logs
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

    # Output: ส่ง Logs ไปยัง Elasticsearch
    <match kubernetes.**>
      @type elasticsearch
      @id out_es
      @log_level info
      include_tag_key true
      host elasticsearch-service
      port 9200
      scheme http
      ssl_verify false
      logstash_format true
      logstash_prefix kubernetes
      logstash_dateformat %Y.%m.%d
      include_timestamp true
      type_name fluentd
      tag_key @log_name
      reload_connections false
      reconnect_on_error true
      reload_on_failure true
      request_timeout 120s
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
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
```

```yaml
# fluentd-daemonset.yaml
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
  - apiGroups: [""]
    resources:
      - namespaces
      - pods
      - nodes
    verbs: ["get", "list", "watch"]
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
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch-service"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
            - name: FLUENT_ELASTICSEARCH_SSL_VERIFY
              value: "false"
            - name: FLUENT_ELASTICSEARCH_SSL_VERSION
              value: "TLSv1_2"
            - name: FLUENT_ELASTICSEARCH_USER
              value: ""
            - name: FLUENT_ELASTICSEARCH_PASSWORD
              value: ""
            - name: FLUENTD_SYSTEMD_CONF
              value: disable
            - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
              value: /var/log/containers/fluentd*
            - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
              value: /^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/
          resources:
            limits:
              memory: 512Mi
              cpu: 200m
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: dockercontainerlogdirectory
              mountPath: /var/log/pods
              readOnly: true
            - name: config-volume
              mountPath: /fluentd/etc/kubernetes/
        terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: dockercontainerlogdirectory
          hostPath:
            path: /var/log/pods
        - name: config-volume
          configMap:
            name: fluentd-config
```

```bash
# Apply Fluentd
kubectl apply -f fluentd-configmap.yaml
kubectl apply -f fluentd-daemonset.yaml

# รอให้ Fluentd DaemonSet Ready
kubectl rollout status daemonset/fluentd -n logging

# ดู Logs ของ Fluentd
kubectl logs -n logging daemonset/fluentd --tail=50
```

---

## Fluentd Configuration

### Input Plugins

```
# อ่าน Logs จาก File
<source>
  @type tail
  path /var/log/app.log
  pos_file /var/log/app.log.pos
  tag app.logs
  <parse>
    @type json
  </parse>
</source>

# รับ Logs ผ่าน HTTP
<source>
  @type http
  port 9880
  bind 0.0.0.0
  body_size_limit 32m
  keepalive_timeout 10s
</source>

# รับ Logs ผ่าน TCP
<source>
  @type tcp
  tag tcp.events
  port 5170
  <parse>
    @type json
  </parse>
</source>
```

### Filter Plugins

```
# เพิ่ม Kubernetes Metadata
<filter kubernetes.**>
  @type kubernetes_metadata
</filter>

# ลบ Fields ที่ไม่ต้องการ
<filter **>
  @type record_transformer
  remove_keys stream,docker
</filter>

# เพิ่ม Fields
<filter **>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    tag ${tag}
    environment "production"
  </record>
</filter>

# Parse JSON Log Message
<filter kubernetes.**>
  @type parser
  key_name log
  reserve_data true
  <parse>
    @type json
  </parse>
</filter>

# Grep Filter - เก็บเฉพาะ Logs ที่ Match
<filter app.**>
  @type grep
  <regexp>
    key level
    pattern /^(error|warn)$/
  </regexp>
</filter>
```

### Output Plugins

```
# ส่งไปยัง Elasticsearch
<match kubernetes.**>
  @type elasticsearch
  host elasticsearch-service
  port 9200
  logstash_format true
  logstash_prefix kubernetes
  <buffer>
    @type file
    path /var/log/fluentd-buffers/kubernetes.buffer
    flush_interval 5s
    chunk_limit_size 2M
    queue_limit_length 8
    overflow_action block
  </buffer>
</match>

# ส่งไปยัง Multiple Outputs
<match app.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch-service
    port 9200
    logstash_format true
  </store>
  <store>
    @type file
    path /var/log/app-backup
    time_format %Y%m%d
  </store>
</match>

# ส่งไปยัง S3
<match archive.**>
  @type s3
  aws_key_id YOUR_KEY
  aws_sec_key YOUR_SECRET
  s3_bucket your-bucket
  s3_region us-east-1
  path logs/
  <buffer>
    @type file
    path /var/log/fluentd-s3-buffer
    timekey 3600
    timekey_wait 10m
  </buffer>
</match>
```

### Fluent Bit (เบากว่า Fluentd)

```yaml
# fluent-bit-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        1
        Daemon       Off
        Log_Level    info
        Parsers_File parsers.conf
        HTTP_Server  On
        HTTP_Listen  0.0.0.0
        HTTP_Port    2020

    [INPUT]
        Name             tail
        Path             /var/log/containers/*.log
        multiline.parser docker, cri
        Tag              kube.*
        Mem_Buf_Limit    5MB
        Skip_Long_Lines  On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch-service
        Port            9200
        HTTP_User
        HTTP_Passwd
        Index           fluent-bit
        Suppress_Type_Name On
        
  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On

    [PARSER]
        Name        syslog
        Format      regex
        Regex       ^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key    time
        Time_Format %b %d %H:%M:%S
```

---

## Kibana Dashboards

### การเข้าถึง Kibana

```bash
# Port Forward ไปยัง Kibana
kubectl port-forward -n logging svc/kibana 5601:5601 &

# เปิด Browser ไปที่:
# http://localhost:5601
```

### การสร้าง Index Pattern

ขั้นตอนในการสร้าง Index Pattern ใน Kibana:
1. เปิด Kibana ที่ http://localhost:5601
2. ไปที่ **Stack Management** > **Index Patterns**
3. คลิก **Create index pattern**
4. ใส่ `kubernetes-*` ใน Index pattern
5. เลือก `@timestamp` เป็น Time field
6. คลิก **Create index pattern**

### Kibana Discover

```
# ค้นหา Logs ด้วย KQL (Kibana Query Language)

# ค้นหา Error Logs ทั้งหมด
level: "error"

# ค้นหา Logs จาก Namespace เฉพาะ
kubernetes.namespace_name: "production"

# ค้นหา Logs จาก Pod เฉพาะ
kubernetes.pod_name: "nginx-abc123"

# ค้นหา Logs ที่มีข้อความเฉพาะ
message: "database connection failed"

# ค้นหาด้วย Wildcard
kubernetes.container_name: nginx*

# Combine Queries
level: "error" AND kubernetes.namespace_name: "production"

# Range Query
http_status_code >= 500

# ค้นหาใน Time Range
@timestamp: [now-1h TO now]
```

### การสร้าง Visualization ใน Kibana

```json
// ตัวอย่าง Visualization สำหรับ Error Rate
{
  "type": "line",
  "title": "Error Rate Over Time",
  "params": {
    "type": "line",
    "grid": {"categoryLines": false},
    "categoryAxes": [{"scale": {"type": "linear"}}],
    "valueAxes": [{"scale": {"type": "linear"}, "labels": {"filter": true}}]
  },
  "aggs": [
    {
      "id": "1",
      "type": "count",
      "schema": "metric"
    },
    {
      "id": "2",
      "type": "date_histogram",
      "params": {"field": "@timestamp", "interval": "auto"},
      "schema": "segment"
    },
    {
      "id": "3",
      "type": "filters",
      "params": {
        "filters": [
          {"input": {"query": "level: error"}},
          {"input": {"query": "level: warn"}}
        ]
      },
      "schema": "group"
    }
  ]
}
```

### Kibana via API

```bash
# สร้าง Index Pattern ผ่าน API
curl -X POST "localhost:5601/api/saved_objects/index-pattern" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "attributes": {
      "title": "kubernetes-*",
      "timeFieldName": "@timestamp"
    }
  }'

# ค้นหา Logs ผ่าน Elasticsearch API
curl -X GET "localhost:9200/kubernetes-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [
          {"match": {"level": "error"}},
          {"range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }}
        ]
      }
    },
    "sort": [{"@timestamp": {"order": "desc"}}],
    "size": 20
  }'
```

---

## Workshop: Centralized Logging Setup

### เป้าหมาย

ในส่วนนี้เราจะ:
1. Deploy EFK Stack บน Kubernetes
2. Deploy Application ที่สร้าง Logs
3. ตั้งค่า Fluentd Configuration
4. สร้าง Kibana Dashboard
5. ทดสอบ Log Search และ Alerting

### ขั้นตอนที่ 1: Setup ด้วย Helm (แนะนำ)

```bash
# เพิ่ม Helm Repository
helm repo add elastic https://helm.elastic.co
helm repo update

# ดู Values ที่มี
helm show values elastic/elasticsearch > elasticsearch-values.yaml

# ติดตั้ง Elasticsearch ด้วย Helm
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=512M \
  --set resources.limits.cpu=1000m \
  --set resources.limits.memory=2Gi \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set esJavaOpts="-Xmx512m -Xms512m" \
  --set xpack.security.enabled=false

# ติดตั้ง Kibana ด้วย Helm
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearchHosts=http://elasticsearch-master:9200 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=256M \
  --set resources.limits.cpu=500m \
  --set resources.limits.memory=1Gi
```

### ขั้นตอนที่ 2: ติดตั้ง Fluent Bit ด้วย Helm

```bash
# เพิ่ม Helm Repository
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# ดู Values ที่มี
helm show values fluent/fluent-bit > fluent-bit-values.yaml

# สร้าง Custom Values
cat > fluent-bit-custom-values.yaml << 'EOF'
config:
  service: |
    [SERVICE]
        Flush         5
        Daemon        Off
        Log_Level     info
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

  inputs: |
    [INPUT]
        Name             tail
        Path             /var/log/containers/*.log
        multiline.parser docker, cri
        Tag              kube.*
        Mem_Buf_Limit    5MB
        Skip_Long_Lines  On

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

  outputs: |
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch-master
        Port            9200
        Index           fluent-bit
        Suppress_Type_Name On

tolerations:
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
EOF

# ติดตั้ง Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  -f fluent-bit-custom-values.yaml
```

### ขั้นตอนที่ 3: Deploy Test Application

```yaml
# test-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-test-app
  namespace: default
  labels:
    app: log-test-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: log-test-app
  template:
    metadata:
      labels:
        app: log-test-app
        version: "1.0"
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                RAND=$((RANDOM % 5))
                TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)
                USER_ID=$((RANDOM % 100 + 1))
                LATENCY=$((RANDOM % 500 + 10))
                
                case $RAND in
                  0)
                    echo "{\"timestamp\":\"$TIMESTAMP\",\"level\":\"info\",\"service\":\"api\",\"pod\":\"$HOSTNAME\",\"user_id\":$USER_ID,\"action\":\"GET /api/users\",\"status\":200,\"latency_ms\":$LATENCY}"
                    ;;
                  1)
                    echo "{\"timestamp\":\"$TIMESTAMP\",\"level\":\"info\",\"service\":\"api\",\"pod\":\"$HOSTNAME\",\"user_id\":$USER_ID,\"action\":\"POST /api/orders\",\"status\":201,\"latency_ms\":$LATENCY}"
                    ;;
                  2)
                    echo "{\"timestamp\":\"$TIMESTAMP\",\"level\":\"warn\",\"service\":\"api\",\"pod\":\"$HOSTNAME\",\"user_id\":$USER_ID,\"action\":\"GET /api/products\",\"status\":200,\"latency_ms\":$((LATENCY + 500)),\"message\":\"high latency detected\"}"
                    ;;
                  3)
                    echo "{\"timestamp\":\"$TIMESTAMP\",\"level\":\"error\",\"service\":\"api\",\"pod\":\"$HOSTNAME\",\"user_id\":$USER_ID,\"action\":\"DELETE /api/users/$USER_ID\",\"status\":500,\"error\":\"database connection timeout\"}"
                    ;;
                  4)
                    echo "{\"timestamp\":\"$TIMESTAMP\",\"level\":\"info\",\"service\":\"api\",\"pod\":\"$HOSTNAME\",\"action\":\"health check\",\"status\":200,\"latency_ms\":5}"
                    ;;
                esac
                sleep 2
              done
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "50m"
```

```bash
# Apply Test Application
kubectl apply -f test-app-deployment.yaml

# ดูว่า Application กำลัง Log
kubectl logs -l app=log-test-app --tail=10

# รอ 1-2 นาทีแล้วตรวจสอบว่า Logs เข้า Elasticsearch
```

### ขั้นตอนที่ 4: Verify Logs ใน Elasticsearch

```bash
# Port Forward Elasticsearch
kubectl port-forward -n logging svc/elasticsearch-master 9200:9200 &

# ตรวจสอบว่ามี Index ของ Fluent Bit
curl -s http://localhost:9200/_cat/indices?v | grep fluent

# ดู Documents ใน Index
curl -s http://localhost:9200/fluent-bit/_search?pretty&size=3 | \
  python3 -m json.tool | head -50

# ค้นหา Error Logs
curl -s -X GET "localhost:9200/fluent-bit/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": {"level": "error"}
    },
    "size": 5,
    "sort": [{"@timestamp": {"order": "desc"}}]
  }'

# นับ Logs ตาม Level
curl -s -X GET "localhost:9200/fluent-bit/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "by_level": {
        "terms": {"field": "level.keyword"}
      }
    }
  }'
```

### ขั้นตอนที่ 5: ตั้งค่า Kibana Dashboard

```bash
# Port Forward Kibana
kubectl port-forward -n logging svc/kibana-kibana 5601:5601 &

# เปิด Browser ไปที่ http://localhost:5601
echo "เปิด Kibana ที่ http://localhost:5601"
```

**การสร้าง Dashboard ใน Kibana:**

1. **สร้าง Index Pattern**:
   - ไปที่ Stack Management > Data Views
   - สร้าง Data View `fluent-bit*`
   - เลือก `@timestamp` เป็น time field

2. **Discover - ดู Logs**:
   - ไปที่ Discover
   - เลือก Data View ที่สร้างไว้
   - ค้นหา `level: "error"`
   - ค้นหา `kubernetes.namespace_name: "default"`

3. **สร้าง Visualization**:
   - ไปที่ Visualize Library > Create visualization
   - เลือก Lens
   - สร้าง Bar Chart แสดง Logs ตาม Level

4. **สร้าง Dashboard**:
   - ไปที่ Dashboard > Create dashboard
   - เพิ่ม Visualizations ที่สร้างไว้

### ขั้นตอนที่ 6: Advanced Fluentd Configuration

```yaml
# advanced-fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-advanced-config
  namespace: logging
data:
  fluent.conf: |
    # ===== INPUT =====
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format /^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<flags>[^ ]+) (?<message>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
      </parse>
    </source>

    # ===== FILTER =====
    
    # เพิ่ม Kubernetes Metadata
    <filter raw.kubernetes.**>
      @type kubernetes_metadata
    </filter>

    # Rename Tag
    <match raw.kubernetes.**>
      @type relabel
      @label @DISPATCH
    </match>

    # ===== DISPATCH =====
    <label @DISPATCH>
      # แยก Logs ตาม Namespace
      <match **>
        @type route
        <route **>
          remove_tag_prefix raw
          @label @OUTPUT
        </route>
      </match>
    </label>

    # ===== OUTPUT =====
    <label @OUTPUT>
      # ส่ง Logs ทั้งหมดไปยัง Elasticsearch
      <match kubernetes.**>
        @type elasticsearch
        host elasticsearch-service
        port 9200
        logstash_format true
        logstash_prefix kubernetes
        <buffer>
          @type file
          path /var/log/fluentd-buffers/kubernetes.buffer
          flush_interval 5s
          retry_max_times 5
        </buffer>
      </match>
    </label>
```

### ขั้นตอนที่ 7: Log Retention Policy

```bash
# ตั้งค่า Index Lifecycle Management (ILM) ใน Elasticsearch
curl -X PUT "localhost:9200/_ilm/policy/kubernetes-logs-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": {
              "max_primary_shard_size": "50gb",
              "max_age": "1d"
            },
            "set_priority": {"priority": 100}
          }
        },
        "warm": {
          "min_age": "7d",
          "actions": {
            "shrink": {"number_of_shards": 1},
            "forcemerge": {"max_num_segments": 1},
            "set_priority": {"priority": 50}
          }
        },
        "cold": {
          "min_age": "30d",
          "actions": {
            "set_priority": {"priority": 0},
            "freeze": {}
          }
        },
        "delete": {
          "min_age": "90d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

### ขั้นตอนที่ 8: Monitor EFK Stack

```bash
# ดู Status ของ EFK Components
echo "=== Elasticsearch Status ==="
kubectl get pods -n logging -l app=elasticsearch

echo ""
echo "=== Kibana Status ==="
kubectl get pods -n logging -l app=kibana

echo ""
echo "=== Fluent Bit Status ==="
kubectl get pods -n logging -l app.kubernetes.io/name=fluent-bit

# ดู Logs ของ Fluent Bit เพื่อตรวจสอบ Errors
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=50

# ตรวจสอบ Elasticsearch Cluster Health
kubectl exec -n logging elasticsearch-master-0 -- \
  curl -s http://localhost:9200/_cluster/health?pretty

# ดู Indices
kubectl exec -n logging elasticsearch-master-0 -- \
  curl -s http://localhost:9200/_cat/indices?v
```

### ขั้นตอนที่ 9: Troubleshooting EFK

```bash
# Problem 1: Fluentd ไม่ส่ง Logs ไปยัง Elasticsearch

# ตรวจสอบ Fluentd Logs
kubectl logs -n logging daemonset/fluentd | grep -i error

# ตรวจสอบว่า Fluentd เชื่อมต่อ Elasticsearch ได้
kubectl exec -n logging $(kubectl get pods -n logging -l app=fluentd -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s http://elasticsearch-service:9200/_cluster/health

# Problem 2: Elasticsearch ไม่พอ Disk Space
kubectl exec -n logging elasticsearch-master-0 -- \
  curl -s http://localhost:9200/_cat/allocation?v

# Problem 3: Kibana ไม่สามารถเชื่อมต่อ Elasticsearch
kubectl logs -n logging deployment/kibana | grep -i error

# ตรวจสอบ Service
kubectl get svc -n logging
kubectl describe svc elasticsearch-service -n logging
```

### ขั้นตอนที่ 10: ทำความสะอาด

```bash
# ลบ Test Application
kubectl delete deployment log-test-app -n default

# ลบ EFK Stack (ถ้าติดตั้งด้วย Helm)
helm uninstall elasticsearch -n logging
helm uninstall kibana -n logging
helm uninstall fluent-bit -n logging

# ลบ Namespace
kubectl delete namespace logging

# ตรวจสอบ
kubectl get namespace logging 2>/dev/null || echo "Namespace ถูกลบแล้ว"
```

---

## Tips และ Best Practices

### 1. Resource Management

```yaml
# กำหนด Resource Limits สำหรับ EFK Components
# Elasticsearch ต้องการ Memory มาก - ตั้ง Heap ที่ 50% ของ Container Memory
env:
  - name: ES_JAVA_OPTS
    value: "-Xms1g -Xmx1g"  # ถ้า Limit Memory = 2Gi

# Fluentd ต้องมี Buffer ที่เหมาะสม
<buffer>
  @type file
  flush_interval 5s    # ส่งทุก 5 วินาที
  chunk_limit_size 2M  # ขนาดสูงสุดของ Chunk
  queue_limit_length 8 # จำนวน Queue สูงสุด
  overflow_action block # Block เมื่อ Queue เต็ม
</buffer>
```

### 2. Security

```bash
# ใช้ Elasticsearch Authentication
# ตั้งค่าใน elasticsearch values:
xpack.security.enabled: true
xpack.security.authc.api_key.enabled: true

# สร้าง Secret สำหรับ Credentials
kubectl create secret generic elasticsearch-credentials \
  -n logging \
  --from-literal=username=elastic \
  --from-literal=password=my-secure-password
```

### 3. Performance Tuning

```yaml
# Elasticsearch Performance Settings
esConfig:
  elasticsearch.yml: |
    cluster.routing.allocation.disk.threshold_enabled: true
    cluster.routing.allocation.disk.watermark.low: "85%"
    cluster.routing.allocation.disk.watermark.high: "90%"
    cluster.routing.allocation.disk.watermark.flood_stage: "95%"
    indices.memory.index_buffer_size: 30%
    thread_pool.write.queue_size: 1000
```

---

## สรุป

EFK Stack เป็นระบบ Centralized Logging ที่ทรงพลังสำหรับ Kubernetes:

1. **Elasticsearch**: เก็บและค้นหา Logs ได้อย่างรวดเร็ว
2. **Fluentd/Fluent Bit**: รวบรวมและ Process Logs จากทุก Pods
3. **Kibana**: Visualize Logs ด้วย Dashboard ที่หลากหลาย

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ Prometheus ซึ่งเป็นระบบ Metrics Monitoring ที่ใช้คู่กับ EFK Stack เพื่อ Observability ที่สมบูรณ์
