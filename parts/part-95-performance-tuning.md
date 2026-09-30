# Part 95: Kubernetes Performance Tuning

## บทนำ

Performance Tuning ใน Kubernetes เป็นกระบวนการต่อเนื่องที่ต้องการการวัดผล วิเคราะห์ และปรับแต่งอย่างสม่ำเสมอ บทนี้ครอบคลุมทุกด้านของการ Tune Performance ตั้งแต่ระดับ OS ไปจนถึง Application

## สารบัญ

1. Performance Baseline
2. Node-level Optimization
3. Kubernetes Scheduler Tuning
4. Network Performance
5. Storage Performance
6. Application Performance
7. Monitoring Performance
8. Workshop: Benchmark และ Tune

---

## 1. Performance Baseline

### 1.1 Cluster Performance Metrics

```bash
#!/bin/bash
# collect-baseline.sh - เก็บ Performance Baseline

echo "=== Kubernetes Performance Baseline ==="
DATE=$(date +%Y%m%d_%H%M%S)
REPORT_DIR="/tmp/baseline-${DATE}"
mkdir -p $REPORT_DIR

# Node Resources
echo "--- Node Resources ---"
kubectl top nodes --sort-by=cpu > $REPORT_DIR/node-resources.txt
kubectl describe nodes > $REPORT_DIR/node-describe.txt

# Pod Resources  
echo "--- Pod Resources ---"
kubectl top pods -A --sort-by=cpu > $REPORT_DIR/pod-resources-cpu.txt
kubectl top pods -A --sort-by=memory > $REPORT_DIR/pod-resources-memory.txt

# API Server Performance
echo "--- API Server Latency ---"
for i in {1..10}; do
  time kubectl get pods -A > /dev/null 2>&1
done 2>> $REPORT_DIR/api-latency.txt

# etcd Performance
echo "--- etcd Performance ---"
ETCDCTL_API=3 etcdctl check perf \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  2>&1 > $REPORT_DIR/etcd-perf.txt

# Scheduler Performance  
echo "--- Scheduler Metrics ---"
curl -sk http://localhost:10259/metrics | grep -E "scheduler_" > $REPORT_DIR/scheduler-metrics.txt

echo "Baseline saved to: $REPORT_DIR"
```

### 1.2 Kubernetes Benchmark Tools

```bash
# ติดตั้ง kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# ดู Results
kubectl logs -n kube-bench job/kube-bench

# ติดตั้ง k6 สำหรับ Load Test
kubectl apply -f https://raw.githubusercontent.com/grafana/k6-operator/main/bundle.yaml

# kube-burner สำหรับ Cluster Performance Test
curl -L https://github.com/cloud-bulldozer/kube-burner/releases/latest/download/kube-burner-V1.9.7-linux-x86_64.tar.gz | tar xz
mv kube-burner /usr/local/bin/
```

---

## 2. Node-level Optimization

### 2.1 CPU Optimization

```bash
#!/bin/bash
# cpu-optimization.sh

# 1. ตั้งค่า CPU Governor เป็น Performance
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo "performance" > $cpu
done

# ยืนยัน
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# 2. Disable CPU Frequency Scaling
systemctl disable ondemand
systemctl stop ondemand

# 3. ปิด Hyper-Threading (ถ้าต้องการ latency ต่ำ)
# echo off > /sys/devices/system/cpu/smt/control

# 4. ตั้งค่า NUMA Topology
apt-get install -y numactl
numactl --hardware

# 5. IRQ Balancing
systemctl enable irqbalance
systemctl start irqbalance

# 6. ปิด Transparent Huge Pages (THP) สำหรับ Database workloads
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# บันทึกถาวร
cat >> /etc/rc.local << 'EOF'
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF
```

### 2.2 Memory Optimization

```bash
#!/bin/bash
# memory-optimization.sh

# 1. ปรับ Swappiness
echo "vm.swappiness = 0" >> /etc/sysctl.d/99-kubernetes.conf
sysctl -w vm.swappiness=0

# 2. ปรับ Memory Overcommit
echo "vm.overcommit_memory = 1" >> /etc/sysctl.d/99-kubernetes.conf
sysctl -w vm.overcommit_memory=1

# 3. ปรับ OOM Settings
echo "vm.panic_on_oom = 0" >> /etc/sysctl.d/99-kubernetes.conf
sysctl -w vm.panic_on_oom=0

# 4. Memory Huge Pages (สำหรับ Database workloads)
# สร้าง 2048 huge pages (2MB each = 4GB total)
echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
echo "vm.nr_hugepages = 2048" >> /etc/sysctl.d/99-kubernetes.conf

# 5. Dirty Memory Settings
cat >> /etc/sysctl.d/99-kubernetes.conf << 'EOF'
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.dirty_expire_centisecs = 3000
vm.dirty_writeback_centisecs = 500
EOF

sysctl -p /etc/sysctl.d/99-kubernetes.conf
```

### 2.3 Disk I/O Optimization

```bash
#!/bin/bash
# disk-optimization.sh

DISK="sda"  # แก้ตาม disk ที่ใช้

# 1. ตั้งค่า I/O Scheduler
# สำหรับ NVMe SSD - ใช้ none หรือ mq-deadline
echo none > /sys/block/${DISK}/queue/scheduler

# สำหรับ SATA SSD - ใช้ deadline
# echo deadline > /sys/block/${DISK}/queue/scheduler

# 2. Read-ahead
blockdev --setra 4096 /dev/${DISK}

# 3. Queue Depth
echo 128 > /sys/block/${DISK}/queue/nr_requests
echo 128 > /sys/block/${DISK}/queue/read_ahead_kb

# 4. Filesystem Mount Options (ใน /etc/fstab)
# UUID=xxx /var/lib/etcd ext4 defaults,noatime,nodiratime,barrier=0,data=writeback 0 2
# UUID=xxx /var/lib/kubelet ext4 defaults,noatime,nodiratime 0 2

# ทดสอบ Disk Performance
apt-get install -y fio
fio --rw=randwrite --bs=4k --ioengine=libaio --direct=1 \
    --iodepth=64 --numjobs=4 --runtime=60 \
    --time_based --filename=/var/lib/etcd/fio-test \
    --name=etcd-write-test \
    --output-format=json
rm -f /var/lib/etcd/fio-test
```

### 2.4 Network Optimization

```bash
#!/bin/bash
# network-optimization.sh

cat >> /etc/sysctl.d/99-network.conf << 'EOF'
# Kubernetes Network Tuning

# TCP Stack
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 8096
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.ip_local_port_range = 1024 65535

# TCP Buffers
net.core.rmem_default = 262144
net.core.rmem_max = 134217728
net.core.wmem_default = 262144
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mem = 88560 118080 177120

# UDP Buffers
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384

# Connection Tracking
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 3600

# Kubernetes Networking
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sysctl -p /etc/sysctl.d/99-network.conf
```

---

## 3. Kubernetes Scheduler Tuning

### 3.1 Scheduler Configuration

```yaml
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: NodeAffinity
        weight: 2
      - name: InterPodAffinity
        weight: 2
      - name: PodTopologySpread
        weight: 2
      - name: TaintToleration
        weight: 3
      - name: ImageLocality
        weight: 1
      - name: NodeResourcesBalancedAllocation
        weight: 1
    filter:
      enabled:
      - name: NodeUnschedulable
      - name: NodeName
      - name: TaintToleration
      - name: NodeAffinity
      - name: NodePorts
      - name: NodeResourcesFit
      - name: InterPodAffinity
      - name: PodTopologySpread
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated  # หรือ LeastAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
  - name: PodTopologySpread
    args:
      defaultConstraints:
      - maxSkew: 3
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
      - maxSkew: 5
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
      defaultMinDomains: 1
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: kube-scheduler
clientConnection:
  qps: 100
  burst: 200
```

### 3.2 Priority Classes

```yaml
# priority-classes.yaml
# Critical System Components
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-system
value: 2000000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Critical system components that must not be preempted"
---
# High Priority Applications
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority applications"
---
# Normal Applications
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: normal
value: 100
globalDefault: true
description: "Default priority for applications"
---
# Batch/Background Jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 10
preemptionPolicy: Never
description: "Low priority batch jobs"
---
# Spot/Preemptible Workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: spot
value: 1
preemptionPolicy: Never
description: "Workloads that can run on spot instances"
```

### 3.3 Custom Scheduler Profile

```yaml
# bin-packing-profile.yaml
# Scheduler Profile สำหรับ Bin Packing (ประหยัด Nodes)
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: bin-packing-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
        - name: nvidia.com/gpu
          weight: 5  # GPU resources have higher weight
```

```yaml
# pod-with-custom-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  schedulerName: bin-packing-scheduler  # ใช้ Custom Scheduler
  containers:
  - name: worker
    image: worker:latest
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
```

---

## 4. Network Performance

### 4.1 CNI Plugin Comparison

```
Performance Comparison (approximate):
Plugin      | Throughput | Latency | CPU Usage | Features
------------|-----------|---------|-----------|----------
Calico      | High      | Low     | Medium    | NetworkPolicy, BGP
Cilium      | Very High | Very Low| Medium    | eBPF, Advanced Security
Flannel     | Medium    | Medium  | Low       | Simple, VXLAN
WeaveNet    | Medium    | Medium  | Medium    | Encryption
Canal       | High      | Low     | Medium    | Calico+Flannel

แนะนำสำหรับ Production: Calico หรือ Cilium
```

### 4.2 Cilium สำหรับ High Performance

```bash
# ติดตั้ง Cilium
helm repo add cilium https://helm.cilium.io/
helm repo update

# ติดตั้งแบบ eBPF Mode (High Performance)
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set tunnel=disabled \
  --set autoDirectNodeRoutes=true \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=$(kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}') \
  --set k8sServicePort=6443 \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set bpf.masquerade=true \
  --set ipam.mode=kubernetes \
  --set bandwidth.enabled=true \
  --set loadBalancer.algorithm=maglev

# ตรวจสอบ
cilium status
cilium connectivity test
```

### 4.3 Service Mesh Performance

```yaml
# istio-performance-config.yaml
# ปรับ Istio สำหรับ High Performance
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: "500m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        env:
        - name: PILOT_ENABLE_PROTOCOL_SNIFFING_FOR_OUTBOUND
          value: "false"
        - name: PILOT_PUSH_THROTTLE
          value: "100"
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_HTTP10: "1"
    accessLogFile: ""  # ปิด access log เพื่อประสิทธิภาพ
    enablePrometheusMerge: true
```

### 4.4 DNS Performance

```yaml
# coredns-performance.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
           prefer_udp
        }
        cache {
           success 9984 30    # Cache 9984 entries สำหรับ 30 วินาที
           denial 9984 5
           prefetch 10 1m 10% # Prefetch entries ที่ใกล้หมดอายุ
        }
        loop
        reload
        loadbalance
    }
---
# Scale CoreDNS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 5  # เพิ่มจาก 2 เป็น 5 สำหรับ Large Clusters
```

### 4.5 NodeLocal DNSCache

```yaml
# nodelocaldns.yaml
# ใช้ NodeLocal DNSCache เพื่อลด Latency
# https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: node-local-dns
  template:
    metadata:
      labels:
        k8s-app: node-local-dns
    spec:
      priorityClassName: system-node-critical
      hostNetwork: true
      dnsPolicy: Default
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      - effect: "NoExecute"
        operator: "Exists"
      - effect: "NoSchedule"
        operator: "Exists"
      containers:
      - name: node-cache
        image: registry.k8s.io/dns/k8s-dns-node-cache:1.22.28
        resources:
          requests:
            cpu: 25m
            memory: 5Mi
        args:
        - -localip
        - "169.254.20.10,10.96.0.10"
        - -conf
        - /etc/Corefile
        - -upstreamsvc
        - kube-dns
        securityContext:
          privileged: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9253
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            host: 169.254.20.10
            path: /health
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
```

---

## 5. Storage Performance

### 5.1 Storage Class Optimization

```yaml
# fast-storage-classes.yaml
# NVMe Local Storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: rancher.io/local-path
parameters:
  nodePath: /mnt/nvme
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: false
---
# Fast SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io2
  iopsPerGB: "50"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# High Throughput
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-throughput
provisioner: kubernetes.io/aws-ebs
parameters:
  type: st1
  fsType: xfs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 5.2 Database Performance ใน Kubernetes

```yaml
# postgres-performance.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-performance
  namespace: database
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Pin ไปยัง Node ที่มี Local SSD
      nodeSelector:
        storage-type: nvme
      # ใช้ Guaranteed QoS
      containers:
      - name: postgres
        image: postgres:14
        resources:
          requests:
            cpu: "4"
            memory: "16Gi"
          limits:
            cpu: "4"     # CPU Request = Limit เพื่อ Guaranteed QoS
            memory: "16Gi"
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        # PostgreSQL Performance Tuning
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        args:
        - postgres
        - -c
        - max_connections=200
        - -c
        - shared_buffers=4GB           # 25% ของ RAM
        - -c
        - effective_cache_size=12GB    # 75% ของ RAM
        - -c
        - maintenance_work_mem=1GB
        - -c
        - checkpoint_completion_target=0.9
        - -c
        - wal_buffers=64MB
        - -c
        - default_statistics_target=100
        - -c
        - random_page_cost=1.1         # SSD
        - -c
        - effective_io_concurrency=200
        - -c
        - min_wal_size=2GB
        - -c
        - max_wal_size=8GB
        - -c
        - max_worker_processes=8
        - -c
        - max_parallel_workers_per_gather=4
        - -c
        - max_parallel_workers=8
        - -c
        - wal_level=replica
        - -c
        - synchronous_commit=local
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: shm
          mountPath: /dev/shm           # สำหรับ Shared Memory
      volumes:
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 2Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-nvme
      resources:
        requests:
          storage: 500Gi
```

---

## 6. Application Performance

### 6.1 Resource Requests and Limits Tuning

```yaml
# resource-tuning.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tuned-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            cpu: "1000m"    # ต้องการ 1 CPU จริงๆ
            memory: "1Gi"
            # ephemeral-storage: "2Gi"
          limits:
            cpu: "2000m"    # Max 2 CPU
            memory: "2Gi"   # Max 2 GB RAM
        # Java Application Tuning
        env:
        - name: JAVA_OPTS
          value: >-
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
            -XX:InitialRAMPercentage=50.0
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
            -XX:+ParallelRefProcEnabled
            -XX:G1HeapRegionSize=32m
            -XX:+AlwaysPreTouch
            -XX:+OptimizeStringConcat
            -server
        # Go Application Tuning
        # - name: GOMAXPROCS
        #   value: "2"   # ตาม CPU limits
        # - name: GOGC
        #   value: "100"
        # Node.js Application Tuning
        # - name: NODE_OPTIONS
        #   value: "--max-old-space-size=1536"  # 75% ของ memory limit
```

### 6.2 JVM Container Performance

```dockerfile
# Dockerfile สำหรับ Java App
FROM eclipse-temurin:17-jre-alpine

# ปรับ JVM สำหรับ Container
ENV JAVA_OPTS="-XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled \
  -Djava.security.egd=file:/dev/./urandom \
  -Dspring.jmx.enabled=false"

COPY app.jar /app.jar
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app.jar"]
```

### 6.3 Connection Pool Optimization

```yaml
# pgbouncer-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pgbouncer-config
  namespace: production
data:
  pgbouncer.ini: |
    [databases]
    mydb = host=postgres port=5432 dbname=mydb
    
    [pgbouncer]
    pool_mode = transaction     ; transaction pooling สำหรับ high concurrency
    max_client_conn = 1000      ; max connections from apps
    default_pool_size = 25      ; connections ต่อ database
    min_pool_size = 5
    reserve_pool_size = 5
    reserve_pool_timeout = 3
    
    ; Performance
    listen_port = 5432
    listen_addr = 0.0.0.0
    auth_type = md5
    auth_file = /etc/pgbouncer/userlist.txt
    
    ; Timeouts
    server_idle_timeout = 600
    client_idle_timeout = 0
    connect_timeout = 15
    query_timeout = 0
    
    ; Logging
    log_connections = 0
    log_disconnections = 0
    log_pooler_errors = 1
    stats_period = 60
    
    ; Admin
    admin_users = postgres
    stats_users = postgres
```

---

## 7. Monitoring Performance

### 7.1 Prometheus Performance Tuning

```yaml
# prometheus-config.yaml
# ปรับ Prometheus สำหรับ High-volume Metrics
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: production-prometheus
  namespace: monitoring
spec:
  replicas: 2
  shards: 3  # Sharding สำหรับ large clusters
  replicaExternalLabelName: "__replica__"
  externalLabels:
    cluster: production
  retention: "15d"
  retentionSize: "100GB"
  resources:
    requests:
      cpu: "2"
      memory: "8Gi"
    limits:
      cpu: "4"
      memory: "16Gi"
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 200Gi
  query:
    maxConcurrency: 20
    maxSamples: 50000000
    timeout: "2m"
  scrapeInterval: "30s"
  evaluationInterval: "30s"
  walCompression: true  # ลด Disk I/O
  prometheusRulesExcludedFromEnforce: []
```

### 7.2 Grafana Dashboard สำหรับ Performance

```json
{
  "title": "Kubernetes Performance Dashboard",
  "panels": [
    {
      "title": "API Server Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(apiserver_request_total[5m])) by (verb, resource)",
          "legendFormat": "{{verb}} {{resource}}"
        }
      ]
    },
    {
      "title": "API Server Latency P99",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))",
          "legendFormat": "P99 {{verb}}"
        }
      ]
    },
    {
      "title": "Scheduler Binding Latency",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(scheduler_binding_duration_seconds_bucket[5m])) by (le))",
          "legendFormat": "P99 Binding Latency"
        }
      ]
    },
    {
      "title": "etcd Write Duration P99",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))",
          "legendFormat": "P99 etcd Write"
        }
      ]
    }
  ]
}
```

---

## 8. Workshop: Benchmark และ Tune

### Workshop Overview

ในบทนี้เราจะ:
1. วัด Baseline Performance
2. ทำ Load Test
3. ระบุ Bottlenecks
4. Apply Optimizations
5. วัดผลหลัง Optimization

### Step 1: ติดตั้ง Benchmark Tools

```bash
# ติดตั้ง k6 สำหรับ HTTP Load Testing
kubectl apply -f https://github.com/grafana/k6-operator/releases/latest/download/bundle.yaml

# ติดตั้ง wrk สำหรับ HTTP Benchmarking
apt-get install -y wrk

# ติดตั้ง sysbench สำหรับ System Benchmark
apt-get install -y sysbench

# ติดตั้ง iperf3 สำหรับ Network Benchmark
apt-get install -y iperf3
```

### Step 2: Deploy Test Application

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: benchmark-app
  namespace: default
spec:
  replicas: 5
  selector:
    matchLabels:
      app: benchmark-app
  template:
    metadata:
      labels:
        app: benchmark-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: benchmark-app
  namespace: default
spec:
  selector:
    app: benchmark-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

kubectl wait --for=condition=available deployment/benchmark-app --timeout=60s
```

### Step 3: Baseline Measurement

```bash
#!/bin/bash
# baseline-test.sh

echo "=== Baseline Performance Test ==="
DATE=$(date +%Y%m%d_%H%M%S)

# 1. API Server Latency
echo "1. API Server Latency Test"
for i in {1..20}; do
  start=$(date +%s%3N)
  kubectl get pods -A > /dev/null 2>&1
  end=$(date +%s%3N)
  echo "$((end - start)) ms"
done | awk '{sum += $1; n++} END {print "Average: " sum/n " ms"}'

# 2. Pod Startup Time
echo "2. Pod Startup Time Test"
start=$(date +%s%3N)
kubectl create deployment startup-test --image=nginx --replicas=1
kubectl wait --for=condition=available deployment/startup-test --timeout=120s
end=$(date +%s%3N)
echo "Pod startup time: $((end - start)) ms"
kubectl delete deployment startup-test

# 3. HTTP Performance
echo "3. HTTP Performance Test"
SERVICE_IP=$(kubectl get svc benchmark-app -o jsonpath='{.spec.clusterIP}')
kubectl run wrk-test --image=appropriate/curl \
  --restart=Never -- \
  /bin/sh -c "apt-get install -y wrk && wrk -t4 -c100 -d30s http://${SERVICE_IP}/"

kubectl wait --for=condition=ready pod/wrk-test --timeout=120s
kubectl logs wrk-test

# 4. Node Resource Usage
echo "4. Node Resource Baseline"
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -20

# Cleanup
kubectl delete pod wrk-test 2>/dev/null || true
echo "=== Baseline Complete ==="
```

### Step 4: Load Test ด้วย k6

```javascript
// k6-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const failRate = new Rate('failed_requests');

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Steady
    { duration: '2m', target: 500 },   // Spike
    { duration: '5m', target: 500 },   // High load
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],  // P99 ต้องน้อยกว่า 500ms
    failed_requests: ['rate<0.01'],    // Error rate ต้องน้อยกว่า 1%
  },
};

export default function() {
  const res = http.get('http://benchmark-app.default.svc.cluster.local/');
  
  const success = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  failRate.add(!success);
  sleep(0.1);
}
```

```yaml
# k6-test-job.yaml
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: benchmark-test
  namespace: default
spec:
  parallelism: 4
  script:
    configMap:
      name: benchmark-script
      file: k6-load-test.js
  runner:
    image: grafana/k6:latest
    resources:
      requests:
        cpu: "1"
        memory: "256Mi"
      limits:
        cpu: "2"
        memory: "512Mi"
```

### Step 5: Apply Optimizations

```bash
#!/bin/bash
# apply-optimizations.sh

echo "=== Applying Performance Optimizations ==="

# 1. Enable NodeLocal DNSCache
echo "1. Enabling NodeLocal DNSCache..."
# Apply NodeLocal DNS DaemonSet
kubectl apply -f nodelocaldns.yaml

# 2. Tune CoreDNS
echo "2. Tuning CoreDNS..."
kubectl apply -f coredns-performance.yaml

# 3. Scale CoreDNS
echo "3. Scaling CoreDNS..."
kubectl scale deployment coredns -n kube-system --replicas=5

# 4. Tune API Server
echo "4. Applying API Server tuning..."
# อัพเดท API Server flags ใน /etc/kubernetes/manifests/kube-apiserver.yaml
# --max-requests-inflight=3000
# --max-mutating-requests-inflight=1000

# 5. Tune kubelet
echo "5. Tuning kubelet..."
cat > /tmp/kubelet-perf.json << 'EOF'
{
  "maxPods": 250,
  "podsPerCore": 10,
  "serializeImagePulls": false,
  "imageGCHighThresholdPercent": 85,
  "imageGCLowThresholdPercent": 80,
  "containerLogMaxSize": "100Mi",
  "containerLogMaxFiles": 3
}
EOF
kubectl -n kube-system get configmap kubelet-config -o yaml | \
  # Apply changes
  kubectl apply -f -

echo "=== Optimizations Applied ==="
```

### Step 6: Post-optimization Benchmark

```bash
#!/bin/bash
# post-optimization-test.sh

echo "=== Post-Optimization Performance Test ==="

# รันเหมือน baseline แต่บันทึกผลเพื่อเปรียบเทียบ

echo "Comparing Results:"
echo "==========================="
echo "Metric               | Before | After"
echo "----------------------------+--------+------"
echo "API Server Latency   | XXX ms | YYY ms"
echo "Pod Startup Time     | XXX ms | YYY ms"
echo "HTTP RPS             | XXXXX  | YYYYY"
echo "P99 Latency          | XXX ms | YYY ms"
echo "Error Rate           | X.XX%  | Y.YY%"
echo "==========================="

# DNS Resolution Test
echo "DNS Performance:"
kubectl run dns-test --image=busybox --restart=Never -- \
  /bin/sh -c "for i in \$(seq 1 100); do time nslookup kubernetes.default; done"
kubectl wait --for=condition=ready pod/dns-test --timeout=60s
kubectl logs dns-test | grep real | awk '{print $2}' | \
  awk -F'm' '{sum += $2; n++} END {print "Average DNS: " sum/n "s"}'
kubectl delete pod dns-test
```

### Step 7: Continuous Performance Monitoring

```yaml
# performance-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: performance-alerts
  namespace: monitoring
spec:
  groups:
  - name: performance
    rules:
    - alert: HighAPIServerLatency
      expr: histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb!~"WATCH|CONNECT"}[5m])) by (le, verb)) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "API Server P99 latency > 1s for {{ $labels.verb }}"

    - alert: HighSchedulerLatency
      expr: histogram_quantile(0.99, sum(rate(scheduler_scheduling_attempt_duration_seconds_bucket[5m])) by (le)) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Scheduler P99 latency > 1s"

    - alert: HighetcdCommitDuration
      expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.25
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "etcd commit duration P99 > 250ms"

    - alert: NodeHighCPU
      expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node CPU usage > 90% on {{ $labels.instance }}"

    - alert: NodeHighMemory
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node memory usage > 90% on {{ $labels.instance }}"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Performance Baseline** - การวัด Baseline ก่อน Tune
2. **Node Optimization** - CPU, Memory, Disk, Network tuning
3. **Scheduler Tuning** - Custom profiles, Priority Classes
4. **Network Performance** - Cilium, DNS optimization
5. **Storage Performance** - Storage class tuning, Database optimization
6. **Application Performance** - Resource tuning, JVM optimization
7. **Monitoring** - Prometheus, Grafana, Alerts
8. **Workshop** - Complete benchmark and tune cycle

## แบบฝึกหัด

1. วัด Baseline Performance ของ Cluster ของคุณ
2. Implement NodeLocal DNSCache และวัดผล
3. ปรับ Scheduler เป็น MostAllocated สำหรับ Cost Saving
4. Tune PostgreSQL บน Kubernetes สำหรับ High Performance
5. สร้าง Performance Dashboard บน Grafana

## References

- [Kubernetes Performance Tuning](https://kubernetes.io/docs/setup/production-environment/)
- [Cilium Performance](https://cilium.io/blog/2021/05/11/cni-benchmark/)
- [etcd Performance](https://etcd.io/docs/v3.5/tuning/)
- [kube-bench](https://github.com/aquasecurity/kube-bench)
