# Part 93: High Availability Kubernetes

## บทนำ

High Availability (HA) ใน Kubernetes คือการออกแบบ Cluster ให้สามารถทนต่อความล้มเหลวของ Component ใดๆ ได้โดยไม่กระทบต่อ Service บทนี้ครอบคลุมการออกแบบและติดตั้ง HA Kubernetes Cluster

## สารบัญ

1. HA Architecture Overview
2. HA Control Plane
3. HA etcd Cluster
4. Load Balancer สำหรับ API Server
5. Multi-zone Deployment
6. Node Failure Handling
7. Pod High Availability
8. Network High Availability
9. Workshop: HA Cluster Testing

---

## 1. HA Architecture Overview

### 1.1 Stacked etcd Topology

```
                          ┌──────────────────────────────────────────┐
                          │              Load Balancer                │
                          │            10.0.0.100:6443               │
                          └────────────┬────────────────┬────────────┘
                                       │                │
                    ┌──────────────────┼────────────────┼──────────────────┐
                    │                  │                │                  │
          ┌─────────▼──────┐  ┌────────▼───────┐  ┌────▼────────────┐
          │ Control Plane 1│  │ Control Plane 2 │  │ Control Plane 3 │
          │ 10.0.0.10      │  │ 10.0.0.11       │  │ 10.0.0.12       │
          │                │  │                 │  │                 │
          │ API Server     │  │ API Server      │  │ API Server      │
          │ Scheduler      │  │ Scheduler       │  │ Scheduler       │
          │ Ctrl Manager   │  │ Ctrl Manager    │  │ Ctrl Manager    │
          │ etcd           │  │ etcd            │  │ etcd            │
          └────────────────┘  └─────────────────┘  └─────────────────┘
                    │                  │                  │
          ┌─────────▼──────────────────▼──────────────────▼─────────────┐
          │                      Worker Nodes                            │
          │     Worker-1 (10.0.1.10)  Worker-2 (10.0.1.11)  ...        │
          └──────────────────────────────────────────────────────────────┘
```

### 1.2 External etcd Topology

```
                          ┌──────────────────────────────────────────┐
                          │              Load Balancer                │
                          │            10.0.0.100:6443               │
                          └────────────┬────────────────┬────────────┘
                                       │                │
          ┌────────────────────────────┼────────────────┼──────────────────────┐
          │ Control Planes              │                │                       │
          │    ┌────────────┐   ┌──────▼─────┐   ┌─────▼──────┐              │
          │    │  CP-1      │   │   CP-2     │   │   CP-3     │              │
          │    │ API Server │   │ API Server │   │ API Server │              │
          │    │ Scheduler  │   │ Scheduler  │   │ Scheduler  │              │
          │    │ Ctrl Mgr   │   │ Ctrl Mgr   │   │ Ctrl Mgr   │              │
          │    └────────────┘   └────────────┘   └────────────┘              │
          └──────────────────────────────────────────────────────────────────┘
                                       │
          ┌────────────────────────────▼─────────────────────────────────────┐
          │ etcd Cluster                                                       │
          │    ┌────────────┐   ┌────────────┐   ┌────────────┐             │
          │    │  etcd-1    │   │   etcd-2   │   │   etcd-3   │             │
          │    │ 10.0.0.20  │   │ 10.0.0.21  │   │ 10.0.0.22  │             │
          │    └────────────┘   └────────────┘   └────────────┘             │
          └──────────────────────────────────────────────────────────────────┘
```

### 1.3 Quorum Requirements

```
จำนวน etcd nodes สำหรับ HA:
- 1 node: ไม่มี Fault Tolerance
- 3 nodes: ทนความล้มเหลวได้ 1 node
- 5 nodes: ทนความล้มเหลวได้ 2 nodes
- 7 nodes: ทนความล้มเหลวได้ 3 nodes

Quorum Formula: (n + 1) / 2
- 3 nodes: quorum = 2
- 5 nodes: quorum = 3
- 7 nodes: quorum = 4
```

---

## 2. HA Control Plane

### 2.1 kubeadm HA Setup

```bash
#!/bin/bash
# ha-init.sh - สร้าง HA Cluster ด้วย kubeadm

# กำหนด Variables
CP1_IP="10.0.0.10"
CP2_IP="10.0.0.11"
CP3_IP="10.0.0.12"
LB_IP="10.0.0.100"
LB_PORT="6443"
POD_CIDR="192.168.0.0/16"
SVC_CIDR="10.96.0.0/12"

# สร้าง kubeadm config
cat > /tmp/ha-kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "$CP1_IP"
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.28.0"
controlPlaneEndpoint: "$LB_IP:$LB_PORT"
etcd:
  local:
    dataDir: /var/lib/etcd
networking:
  podSubnet: "$POD_CIDR"
  serviceSubnet: "$SVC_CIDR"
apiServer:
  certSANs:
  - "$LB_IP"
  - "$CP1_IP"
  - "$CP2_IP"
  - "$CP3_IP"
  - "127.0.0.1"
  - "k8s-api.example.com"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
EOF

# Init Control Plane แรก
kubeadm init \
  --config /tmp/ha-kubeadm-config.yaml \
  --upload-certs \
  --v=5

# Setup kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

### 2.2 การตรวจสอบ HA Control Plane

```bash
#!/bin/bash
# check-ha-control-plane.sh

echo "=== HA Control Plane Health Check ==="

# ตรวจสอบ Nodes
echo "--- Nodes ---"
kubectl get nodes -l node-role.kubernetes.io/control-plane

# ตรวจสอบ API Server Pods
echo "--- API Server Pods ---"
kubectl get pods -n kube-system -l component=kube-apiserver -o wide

# ตรวจสอบ etcd
echo "--- etcd Pods ---"
kubectl get pods -n kube-system -l component=etcd -o wide

# ตรวจสอบ etcd Health
echo "--- etcd Health ---"
for NODE in 10.0.0.10 10.0.0.11 10.0.0.12; do
  echo "etcd on $NODE:"
  ETCDCTL_API=3 etcdctl endpoint health \
    --endpoints=https://${NODE}:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
done

# ตรวจสอบ Leader
echo "--- etcd Leader ---"
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -w table

echo "=== Done ==="
```

### 2.3 Simulating Control Plane Failure

```bash
#!/bin/bash
# test-cp-failure.sh - ทดสอบ Control Plane Failure

echo "=== Testing Control Plane Failure ==="

# บันทึก State ก่อน
kubectl get nodes -o wide > /tmp/before-failure.txt
kubectl get pods -A > /tmp/pods-before.txt

echo "1. Current State:"
kubectl get nodes

# หยุด API Server บน CP-1 (จำลอง Failure)
echo "2. Stopping kube-apiserver on CP-1..."
# (ทำบน control-plane-1)
# systemctl stop kubelet  # หรือ kill kube-apiserver pod

# ทดสอบว่ายังทำงานได้ผ่าน Load Balancer
echo "3. Testing cluster availability..."
kubectl get pods -A --server=https://10.0.0.100:6443
kubectl get nodes

# ตรวจสอบว่า API ยังตอบสนอง
for i in {1..10}; do
  curl -sk https://10.0.0.100:6443/healthz -o /dev/null -w "%{http_code}\n"
  sleep 1
done

echo "4. Cluster survived CP failure!"

# เปิด API Server กลับ
echo "5. Bringing CP-1 back online..."
# systemctl start kubelet

# ตรวจสอบหลัง Recovery
sleep 30
kubectl get nodes
```

---

## 3. HA etcd Cluster

### 3.1 External etcd Setup

```bash
#!/bin/bash
# setup-external-etcd.sh - ติดตั้ง External etcd Cluster

ETCD_VER="v3.5.9"
ETCD1_IP="10.0.0.20"
ETCD2_IP="10.0.0.21"
ETCD3_IP="10.0.0.22"
CURRENT_IP=$(hostname -I | awk '{print $1}')
CLUSTER_TOKEN="production-etcd-cluster"

# ติดตั้ง etcd
curl -L https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
tar xzvf /tmp/etcd.tar.gz -C /tmp
mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/

# สร้าง User
useradd -r -s /bin/false etcd
mkdir -p /var/lib/etcd /var/log/etcd /etc/etcd
chown etcd:etcd /var/lib/etcd /var/log/etcd /etc/etcd

# สร้าง TLS Certificates (ใช้ cfssl)
apt-get install -y golang-cfssl

mkdir -p /etc/etcd/ssl
cd /etc/etcd/ssl

# CA Config
cat > ca-config.json << 'EOF'
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
        "expiry": "87600h",
        "usages": ["signing", "key encipherment", "server auth", "client auth"]
      }
    }
  }
}
EOF

# CA CSR
cat > ca-csr.json << 'EOF'
{
  "CN": "etcd-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "TH",
      "L": "Bangkok",
      "O": "Kubernetes",
      "ST": "Thailand"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# etcd Server Certificate
cat > etcd-csr.json << EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "$ETCD1_IP",
    "$ETCD2_IP",
    "$ETCD3_IP"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  etcd-csr.json | cfssljson -bare etcd

chown -R etcd:etcd /etc/etcd/ssl

# สร้าง etcd systemd service
cat > /etc/systemd/system/etcd.service << EOF
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name etcd-$(hostname) \\
  --data-dir /var/lib/etcd \\
  --listen-client-urls https://${CURRENT_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${CURRENT_IP}:2379 \\
  --listen-peer-urls https://${CURRENT_IP}:2380 \\
  --initial-advertise-peer-urls https://${CURRENT_IP}:2380 \\
  --initial-cluster etcd-1=https://${ETCD1_IP}:2380,etcd-2=https://${ETCD2_IP}:2380,etcd-3=https://${ETCD3_IP}:2380 \\
  --initial-cluster-token ${CLUSTER_TOKEN} \\
  --initial-cluster-state new \\
  --client-cert-auth \\
  --trusted-ca-file /etc/etcd/ssl/ca.pem \\
  --cert-file /etc/etcd/ssl/etcd.pem \\
  --key-file /etc/etcd/ssl/etcd-key.pem \\
  --peer-client-cert-auth \\
  --peer-trusted-ca-file /etc/etcd/ssl/ca.pem \\
  --peer-cert-file /etc/etcd/ssl/etcd.pem \\
  --peer-key-file /etc/etcd/ssl/etcd-key.pem \\
  --auto-compaction-retention=1 \\
  --quota-backend-bytes=8589934592 \\
  --snapshot-count=10000 \\
  --heartbeat-interval=100 \\
  --election-timeout=1000 \\
  --log-outputs=/var/log/etcd/etcd.log \\
  --log-level=warn
Restart=always
RestartSec=5
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

# ตรวจสอบ
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://${CURRENT_IP}:2379 \
  --cacert=/etc/etcd/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  -w table
```

### 3.2 etcd Monitoring

```yaml
# etcd-prometheus-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd
  namespace: monitoring
  labels:
    app: etcd
spec:
  endpoints:
  - interval: 30s
    port: metrics
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.crt
      certFile: /etc/prometheus/secrets/etcd-certs/client.crt
      keyFile: /etc/prometheus/secrets/etcd-certs/client.key
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: etcd
```

```yaml
# etcd-alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-alerts
  namespace: monitoring
spec:
  groups:
  - name: etcd
    rules:
    - alert: EtcdMembersDown
      expr: max without (endpoint) (sum without (instance) (up{job="etcd"} == bool 0)) > 0
      for: 3m
      labels:
        severity: critical
      annotations:
        summary: "etcd cluster member down"
        description: "etcd cluster member(s) are down"

    - alert: EtcdInsufficientMembers
      expr: sum(up{job="etcd"} == bool 1) < ((count(up{job="etcd"}) + 1) / 2)
      for: 3m
      labels:
        severity: critical
      annotations:
        summary: "etcd cluster has insufficient quorum"

    - alert: EtcdHighCommitDuration
      expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.25
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "etcd commit duration is high"

    - alert: EtcdHighFsyncDuration
      expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "etcd fsync duration is high"

    - alert: EtcdHighDatabaseSize
      expr: etcd_mvcc_db_total_size_in_bytes > 7e+09
      for: 0m
      labels:
        severity: warning
      annotations:
        summary: "etcd database size is large (>7GB)"

    - alert: EtcdLeaderChanges
      expr: increase(etcd_server_leader_changes_seen_total[1h]) > 3
      for: 0m
      labels:
        severity: warning
      annotations:
        summary: "etcd leader changes too often"
```

---

## 4. Load Balancer สำหรับ API Server

### 4.1 HAProxy Configuration

```
# /etc/haproxy/haproxy.cfg
global
    log         /dev/log local0
    log         /dev/log local1 notice
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     50000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
    ssl-default-bind-ciphers ECDHE+AESGCM:ECDHE+CHACHA20:RSA+AESGCM:RSA+AES
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log                     global
    mode                    tcp
    option                  tcplog
    option                  dontlognull
    option                  tcp-smart-accept
    option                  tcp-smart-connect
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout check           10s
    maxconn                 50000

# Kubernetes API Server
frontend k8s-api-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance leastconn
    option tcp-check
    timeout connect 5s
    timeout server 60s
    server control-plane-1 10.0.0.10:6443 check inter 3s fall 3 rise 2 weight 1
    server control-plane-2 10.0.0.11:6443 check inter 3s fall 3 rise 2 weight 1
    server control-plane-3 10.0.0.12:6443 check inter 3s fall 3 rise 2 weight 1

# HAProxy Stats
listen stats
    bind 0.0.0.0:8080
    mode http
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:StrongPassword123!
    stats admin if TRUE
```

### 4.2 Keepalived สำหรับ VIP HA

```bash
# ติดตั้ง Keepalived บน Load Balancer Nodes
apt-get install -y keepalived

# /etc/keepalived/keepalived.conf สำหรับ Master
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id LVS_DEVEL
    vrrp_skip_check_adv_addr
    vrrp_strict
    vrrp_garp_interval 0
    vrrp_gna_interval 0
}

vrrp_script check_haproxy {
    script "kill -0 $(cat /var/run/haproxy.pid)"
    interval 2
    weight -20
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass StrongVRRPPass
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        check_haproxy
    }
}
EOF

# /etc/keepalived/keepalived.conf สำหรับ Backup
cat > /etc/keepalived/keepalived.conf << 'EOF'
# บน Load Balancer Backup Node
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90  # ต่ำกว่า Master
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass StrongVRRPPass
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
}
EOF

systemctl enable keepalived
systemctl start keepalived
```

### 4.3 kube-vip Alternative

```yaml
# kube-vip as Static Pod
# /etc/kubernetes/manifests/kube-vip.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - name: kube-vip
    image: ghcr.io/kube-vip/kube-vip:v0.6.3
    imagePullPolicy: Always
    args:
    - manager
    env:
    - name: vip_arp
      value: "true"
    - name: port
      value: "6443"
    - name: vip_interface
      value: eth0
    - name: vip_cidr
      value: "32"
    - name: cp_enable
      value: "true"
    - name: cp_namespace
      value: kube-system
    - name: vip_ddns
      value: "false"
    - name: vip_leaderelection
      value: "true"
    - name: vip_leaseduration
      value: "5"
    - name: vip_renewdeadline
      value: "3"
    - name: vip_retryperiod
      value: "1"
    - name: address
      value: "10.0.0.100"
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
    volumeMounts:
    - mountPath: /etc/kubernetes/admin.conf
      name: kubeconfig
  hostAliases:
  - hostnames:
    - kubernetes
    ip: 127.0.0.1
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/admin.conf
    name: kubeconfig
```

---

## 5. Multi-zone Deployment

### 5.1 Zone Labeling

```bash
# Label Nodes ด้วย Zone Information
kubectl label node worker-1 topology.kubernetes.io/zone=zone-a
kubectl label node worker-2 topology.kubernetes.io/zone=zone-a
kubectl label node worker-3 topology.kubernetes.io/zone=zone-b
kubectl label node worker-4 topology.kubernetes.io/zone=zone-b
kubectl label node worker-5 topology.kubernetes.io/zone=zone-c
kubectl label node worker-6 topology.kubernetes.io/zone=zone-c

# ตรวจสอบ
kubectl get nodes --show-labels | grep zone
```

### 5.2 Topology Spread Constraints

```yaml
# multi-zone-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
  namespace: production
spec:
  replicas: 9
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      # กระจาย Pods ตาม Zone
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ha-app
      # กระจาย Pods ตาม Node
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ha-app
      # ไม่ให้ Pod เดียวกัน อยู่บน Node เดียวกัน
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["ha-app"]
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:v1
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

### 5.3 Zone-aware Services

```yaml
# zone-aware-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ha-app-service
  namespace: production
  annotations:
    service.kubernetes.io/topology-mode: "Auto"
spec:
  selector:
    app: ha-app
  ports:
  - port: 80
    targetPort: 8080
  topologyKeys:
  - "topology.kubernetes.io/zone"  # ชอบ Local Zone ก่อน
  - "*"  # ถ้าไม่มีใน Local Zone ใช้ Global
```

### 5.4 StatefulSet Multi-zone

```yaml
# statefulset-multizone.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database-cluster
  namespace: production
spec:
  serviceName: database-headless
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: database
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["database"]
            topologyKey: kubernetes.io/hostname
      containers:
      - name: database
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi
```

---

## 6. Node Failure Handling

### 6.1 Node Taint และ Tolerations สำหรับ HA

```yaml
# critical-app-tolerations.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 5
  template:
    spec:
      tolerations:
      # รับ node ที่มี memory pressure
      - key: "node.kubernetes.io/memory-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      # รับ node ที่ไม่ ready ชั่วคราว (5 นาที)
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 300
      # รับ unreachable node (5 นาที)
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 300
```

### 6.2 Node Conditions Monitoring

```yaml
# node-problem-detector.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-problem-detector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-problem-detector
  template:
    metadata:
      labels:
        app: node-problem-detector
    spec:
      serviceAccountName: node-problem-detector
      containers:
      - name: node-problem-detector
        image: registry.k8s.io/node-problem-detector/node-problem-detector:v0.8.14
        command:
        - /node-problem-detector
        - --logtostderr
        - --config.system-log-monitor=/config/kernel-monitor.json
        - --config.custom-plugin-monitor=/config/network-problem-monitor.json
        - --prometheus-address=0.0.0.0
        - --prometheus-port=20257
        securityContext:
          privileged: true
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: log
          mountPath: /var/log
          readOnly: true
        - name: kmsg
          mountPath: /dev/kmsg
          readOnly: true
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
      volumes:
      - name: log
        hostPath:
          path: /var/log/
      - name: kmsg
        hostPath:
          path: /dev/kmsg
      - name: localtime
        hostPath:
          path: /etc/localtime
```

### 6.3 Graceful Node Shutdown

```yaml
# kubelet-config-graceful-shutdown.yaml
# เพิ่มใน KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
shutdownGracePeriod: 60s
shutdownGracePeriodCriticalPods: 20s
# แยก Priority Class:
shutdownGracePeriodByPodPriority:
- priority: 0
  shutdownGracePeriodSeconds: 30
- priority: 1000000
  shutdownGracePeriodSeconds: 60
```

---

## 7. Pod High Availability

### 7.1 Pod Disruption Budgets

```yaml
# pdb-examples.yaml
# ห้าม Evict ถ้าเหลือ Pods ไม่ถึง 75%
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  minAvailable: "75%"
  selector:
    matchLabels:
      app: api-server
---
# อนุญาตให้ Unavailable ได้ไม่เกิน 1 pod
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: worker
---
# ไม่อนุญาตให้ Evict เลย (สำหรับ Critical Services)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-pdb
  namespace: production
spec:
  minAvailable: "100%"
  selector:
    matchLabels:
      tier: critical
```

### 7.2 Liveness และ Readiness Probes ที่ดี

```yaml
# robust-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: robust-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          # รอ App start ก่อน check
          initialDelaySeconds: 30
          # ตรวจสอบทุก 15 วินาที
          periodSeconds: 15
          # ถ้าตอบช้ากว่า 5 วินาที = failed
          timeoutSeconds: 5
          # ต้อง fail 3 ครั้งติดกัน จึง restart
          failureThreshold: 3
          # ต้อง success 1 ครั้ง ถึงจะ live
          successThreshold: 1
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
          # ต้อง fail 2 ครั้งติดกัน จึง remove จาก endpoints
          failureThreshold: 2
          # ต้อง success 2 ครั้งติดกัน จึง add กลับมาใน endpoints
          successThreshold: 2
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          # รอนาน ถ้า App start ช้า (max 5 นาที)
          failureThreshold: 30
          periodSeconds: 10
          # หลังจาก startup probe ผ่าน จึงเริ่ม liveness/readiness
```

### 7.3 Rolling Update Strategy

```yaml
# rolling-update-strategy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Update ทีละ 2 pods
      maxSurge: 2
      # ไม่อนุญาตให้ unavailable ระหว่าง update
      maxUnavailable: 0
  minReadySeconds: 30  # รอ 30 วินาทีก่อนถือว่า Pod ready
  progressDeadlineSeconds: 600  # timeout 10 นาที
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: myapp:v2
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # รอให้ connections drain
```

---

## 8. Network High Availability

### 8.1 CoreDNS HA

```yaml
# coredns-ha.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 3  # เพิ่มจาก default 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: coredns
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["coredns"]
              topologyKey: kubernetes.io/hostname
```

### 8.2 Service Redundancy

```yaml
# redundant-service.yaml
# ใช้ Multiple Endpoints
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  # Session Affinity สำหรับ Stateful Connections
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
---
# ExternalTrafficPolicy สำหรับ External Load Balancing
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Preserve Source IP
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

---

## 9. Workshop: HA Cluster Testing

### Workshop Overview

ในบทนี้เราจะทดสอบ HA โดย:
1. ทดสอบ Control Plane Failure
2. ทดสอบ etcd Node Failure
3. ทดสอบ Worker Node Failure
4. ทดสอบ Network Partition
5. วัด Recovery Time

### Scenario 1: API Server Failure

```bash
#!/bin/bash
# test-apiserver-failure.sh

echo "=== Test: API Server Failure ==="

# Deploy test application ก่อน
kubectl create deployment test-app --image=nginx --replicas=5
kubectl expose deployment test-app --port=80 --type=NodePort

# บันทึก Port
NODEPORT=$(kubectl get svc test-app -o jsonpath='{.spec.ports[0].nodePort}')

# รัน continuous test ใน background
kubectl run tester --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://test-app.default.svc.cluster.local/ > /dev/null 2>&1 && echo 'OK' || echo 'FAIL'; sleep 1; done" &

echo "Starting API Server failure test..."
echo "Watch pod status in another terminal: kubectl get pods -A -w"

# จำลอง API Server failure บน Control Plane 1
# (รันบน control-plane-1)
# kill $(pgrep kube-apiserver)
echo "ACTION: Kill kube-apiserver on control-plane-1"
echo "Expected: Cluster still works via other control planes"
echo "Wait 60 seconds..."
sleep 60

# ตรวจสอบว่า Application ยังทำงาน
echo "Testing application availability..."
for i in {1..10}; do
  curl -sk http://$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}'):$NODEPORT/ > /dev/null && echo "Request $i: OK" || echo "Request $i: FAILED"
  sleep 1
done

# ตรวจสอบ Nodes
kubectl get nodes

# Cleanup
kubectl delete deployment test-app
kubectl delete svc test-app
kubectl delete pod tester --force
```

### Scenario 2: etcd Node Failure

```bash
#!/bin/bash
# test-etcd-failure.sh

echo "=== Test: etcd Node Failure ==="

# ตรวจสอบ etcd health ก่อน
echo "Before failure:"
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -w table

# สร้าง Secrets หลายๆ ตัวก่อน
for i in {1..20}; do
  kubectl create secret generic "test-secret-${i}" \
    --from-literal=key="value-${i}" 2>/dev/null
done

echo "Created 20 secrets"

# จำลอง etcd failure บน node หนึ่ง
echo "ACTION: Stop etcd on control-plane-2"
echo "Expected: Cluster works with 2/3 etcd members (quorum maintained)"

# ตรวจสอบหลัง failure
sleep 10
echo "After failure:"
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://10.0.0.10:2379,https://10.0.0.12:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -w table

# ทดสอบว่ายัง write ได้
kubectl create secret generic "test-secret-99" --from-literal=key=value
echo "Write test: $?"

# ทดสอบว่ายัง read ได้
kubectl get secret test-secret-99
echo "Read test: $?"

# Cleanup
for i in {1..20}; do
  kubectl delete secret "test-secret-${i}" 2>/dev/null
done
kubectl delete secret test-secret-99 2>/dev/null
```

### Scenario 3: Worker Node Failure

```bash
#!/bin/bash
# test-worker-failure.sh

echo "=== Test: Worker Node Failure ==="

# Deploy Application ที่กระจาย 5 pods
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: ha-test
  template:
    metadata:
      labels:
        app: ha-test
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ha-test
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

kubectl wait --for=condition=available deployment/ha-test --timeout=60s

echo "Initial pod distribution:"
kubectl get pods -l app=ha-test -o wide

# Drain Worker Node
TARGET_NODE=$(kubectl get pods -l app=ha-test -o wide | grep -v NAME | head -1 | awk '{print $7}')
echo "Draining node: $TARGET_NODE"

kubectl drain $TARGET_NODE \
  --delete-emptydir-data \
  --force \
  --ignore-daemonsets \
  --grace-period=30

echo "After drain:"
kubectl get nodes
kubectl get pods -l app=ha-test -o wide

echo "Waiting for pods to reschedule..."
kubectl wait --for=condition=available deployment/ha-test --timeout=120s

echo "Final pod distribution:"
kubectl get pods -l app=ha-test -o wide

# Uncordon
kubectl uncordon $TARGET_NODE

# Cleanup
kubectl delete deployment ha-test
```

### Scenario 4: วัด Recovery Time

```bash
#!/bin/bash
# measure-recovery-time.sh

echo "=== Measuring Recovery Time ==="

# Deploy Application
kubectl create deployment recovery-test --image=nginx --replicas=10
kubectl expose deployment recovery-test --port=80

# Wait for all pods
kubectl wait --for=condition=available deployment/recovery-test --timeout=120s

# บันทึก Start Time
START_TIME=$(date +%s)

# หยุด Node
TARGET_NODE=$(kubectl get pods -l app=recovery-test -o wide | grep -v NAME | head -1 | awk '{print $7}')
echo "Simulating failure on: $TARGET_NODE"
kubectl taint node $TARGET_NODE key=value:NoExecute

# รอให้ Pods Reschedule
until [ $(kubectl get pods -l app=recovery-test --field-selector=status.phase=Running | grep -c Running) -eq 10 ]; do
  echo "Waiting for recovery..."
  sleep 5
done

END_TIME=$(date +%s)
RECOVERY_TIME=$((END_TIME - START_TIME))
echo "Recovery Time: ${RECOVERY_TIME} seconds"

# ลบ Taint
kubectl taint node $TARGET_NODE key=value:NoExecute-

# Cleanup
kubectl delete deployment recovery-test
kubectl delete svc recovery-test
```

### ผลการทดสอบที่ควรได้

```
Scenario 1 - API Server Failure:
- Application availability: 100%
- Recovery time: < 30 seconds
- Data loss: None

Scenario 2 - etcd Node Failure (1/3):
- Cluster availability: 100%
- Write capability: Maintained
- Recovery time: Automatic

Scenario 3 - Worker Node Failure:
- Pod rescheduling: < 5 minutes
- Service availability: Maintained (via PDB)
- Data persistence: Maintained

Scenario 4 - Measured Recovery:
- Target: < 5 minutes for node failure
- Target: < 30 seconds for API server failover
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HA Architecture** - Stacked vs External etcd
2. **HA Control Plane** - Multi-master setup
3. **HA etcd** - External cluster, monitoring, alerts
4. **Load Balancer** - HAProxy + Keepalived, kube-vip
5. **Multi-zone** - Topology spread, zone-aware services
6. **Node Failure** - Handling, graceful shutdown
7. **Pod HA** - PDB, probes, rolling updates
8. **Network HA** - CoreDNS HA, service redundancy
9. **Workshop** - HA Testing scenarios

## แบบฝึกหัด

1. ติดตั้ง HA Cluster ด้วย 3 Control Planes และ HAProxy
2. ทดสอบ Control Plane Failure - ยืนยันว่า Cluster ยังทำงาน
3. ตั้งค่า etcd Backup Automation ที่ทำ Backup ทุก 6 ชั่วโมง
4. Deploy Application ที่มี PDB และทดสอบ Node Drain
5. Measure Recovery Time สำหรับ Worker Node Failure

## References

- [Kubernetes HA Setup](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [etcd Operations Guide](https://etcd.io/docs/v3.5/op-guide/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
