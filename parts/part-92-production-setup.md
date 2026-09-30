# Part 92: Production Cluster Setup

## บทนำ

การติดตั้ง Kubernetes Cluster สำหรับ Production Environment ต้องการการวางแผนและการตั้งค่าอย่างละเอียด บทนี้ครอบคลุมทุกขั้นตอนตั้งแต่การเตรียม Infrastructure จนถึงการติดตั้งและตรวจสอบ Cluster ที่พร้อมใช้งานใน Production

## สารบัญ

1. Production Cluster Requirements
2. Infrastructure Planning
3. OS Preparation
4. Container Runtime Setup
5. kubeadm Installation
6. etcd Best Practices
7. High Availability Setup
8. Post-installation Configuration
9. Workshop: Install Production Cluster

---

## 1. Production Cluster Requirements

### 1.1 Hardware Requirements

| Component | Minimum | Recommended | Production |
|-----------|---------|-------------|------------|
| Control Plane CPU | 2 cores | 4 cores | 8+ cores |
| Control Plane RAM | 2 GB | 8 GB | 16+ GB |
| Control Plane Storage | 30 GB | 100 GB | 200+ GB SSD |
| Worker Node CPU | 2 cores | 4 cores | 8-32 cores |
| Worker Node RAM | 4 GB | 16 GB | 32-256 GB |
| Worker Node Storage | 50 GB | 100 GB | 500+ GB |
| Network | 1 Gbps | 10 Gbps | 10-40 Gbps |

### 1.2 Software Requirements

```bash
# Kubernetes Version Support Matrix
# Control Plane ควร upgrade ก่อน Worker Nodes เสมอ
# Skew Policy: Worker Node version ห้าม exceed control plane

# OS ที่แนะนำ
# - Ubuntu 22.04 LTS
# - Rocky Linux 9
# - Red Hat Enterprise Linux 9

# Kernel Version
uname -r  # ต้องการ >= 4.15

# Required ports
# Control Plane:
# - 6443/TCP (Kubernetes API Server)
# - 2379-2380/TCP (etcd server client API)
# - 10250/TCP (Kubelet API)
# - 10259/TCP (kube-scheduler)
# - 10257/TCP (kube-controller-manager)

# Worker Nodes:
# - 10250/TCP (Kubelet API)
# - 30000-32767/TCP (NodePort Services)
```

### 1.3 Network Requirements

```bash
# ตัวอย่าง IP Planning สำหรับ Production
# Pod CIDR: 192.168.0.0/16 (65,534 IPs)
# Service CIDR: 10.96.0.0/12 (1,048,574 IPs)
# Node CIDR: 10.0.0.0/16

# Control Plane Nodes
# 10.0.0.10 - control-plane-1
# 10.0.0.11 - control-plane-2
# 10.0.0.12 - control-plane-3

# Load Balancer VIP (สำหรับ API Server)
# 10.0.0.100 - k8s-api-lb

# Worker Nodes
# 10.0.1.10 - worker-1
# 10.0.1.11 - worker-2
# 10.0.1.12 - worker-3
```

---

## 2. Infrastructure Planning

### 2.1 Server Preparation Script

```bash
#!/bin/bash
# prepare-node.sh - รันบน Node ทุกตัว

set -euo pipefail

# ตรวจสอบว่ารันเป็น root
if [[ $EUID -ne 0 ]]; then
   echo "This script must be run as root" 
   exit 1
fi

echo "=== Preparing Kubernetes Node ==="

# อัพเดท System
apt-get update && apt-get upgrade -y

# ติดตั้ง dependencies
apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  vim \
  wget \
  net-tools \
  htop \
  iotop \
  jq \
  git \
  unzip

# ปิด Swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# ยืนยัน Swap ปิด
echo "Swap status:"
free -h

# Load required kernel modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# ตั้งค่า sysctl
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv6.conf.all.forwarding        = 1
vm.swappiness                       = 0
vm.max_map_count                    = 262144
fs.inotify.max_user_watches         = 524288
fs.inotify.max_user_instances       = 512
fs.file-max                         = 100000
kernel.pid_max                      = 4194304
net.core.somaxconn                  = 32768
net.ipv4.tcp_max_syn_backlog        = 8096
net.ipv4.ip_local_port_range        = 1024 65535
EOF

sysctl --system

# ปิด SELinux (RHEL-based)
if command -v sestatus &> /dev/null; then
    setenforce 0
    sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
fi

# ตั้งค่า Firewall
if command -v ufw &> /dev/null; then
    ufw disable
fi

# ตั้งค่า Timezone
timedatectl set-timezone Asia/Bangkok

# ติดตั้ง chrony สำหรับ NTP
apt-get install -y chrony
systemctl enable chronyd
systemctl start chronyd

echo "=== Node preparation complete ==="
```

### 2.2 เพิ่ม Kernel Performance Parameters

```bash
# /etc/sysctl.d/99-kubernetes-performance.conf
cat <<EOF > /etc/sysctl.d/99-kubernetes-performance.conf
# Kubernetes Performance Tuning

# Memory
vm.swappiness = 0
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# Network
net.core.netdev_max_backlog = 16384
net.core.somaxconn = 32768
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_max_syn_backlog = 8096
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_slow_start_after_idle = 0

# File System
fs.aio-max-nr = 1048576
fs.file-max = 6815744
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512

# Kernel
kernel.pid_max = 4194304
kernel.threads-max = 4096000
EOF

sysctl -p /etc/sysctl.d/99-kubernetes-performance.conf
```

---

## 3. Container Runtime Setup

### 3.1 ติดตั้ง containerd

```bash
#!/bin/bash
# install-containerd.sh

CONTAINERD_VERSION="1.7.8"
RUNC_VERSION="1.1.9"
CNI_PLUGINS_VERSION="1.3.0"

# ติดตั้ง containerd
wget https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz

# สร้าง systemd service
cat <<EOF > /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

# ติดตั้ง runc
wget https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

# ติดตั้ง CNI Plugins
mkdir -p /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v${CNI_PLUGINS_VERSION}/cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz

# สร้าง containerd config
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# ตั้งค่า systemd cgroup driver
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# เปิด Sandbox Image ใน containerd config (ใช้ Public Mirror ถ้าต้องการ)
# sed -i 's/sandbox_image = .*/sandbox_image = "registry.k8s.io\/pause:3.9"/' /etc/containerd/config.toml

# Enable และ Start containerd
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd
```

### 3.2 ตั้งค่า containerd สำหรับ Private Registry

```toml
# /etc/containerd/config.toml (เพิ่มส่วน registry)
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://mirror.gcr.io"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
      endpoint = ["https://k8s-mirror.example.com"]
  
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."myregistry.example.com".auth]
      username = "admin"
      password = "password"
    [plugins."io.containerd.grpc.v1.cri".registry.configs."myregistry.example.com".tls]
      ca_file = "/etc/ssl/certs/ca.crt"
```

---

## 4. kubeadm Installation

### 4.1 ติดตั้ง kubeadm, kubelet, kubectl

```bash
#!/bin/bash
# install-kubernetes-tools.sh

K8S_VERSION="1.28.0"

# เพิ่ม Kubernetes APT Repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

# ติดตั้ง kubeadm, kubelet, kubectl
apt-get install -y \
  kubelet="${K8S_VERSION}-*" \
  kubeadm="${K8S_VERSION}-*" \
  kubectl="${K8S_VERSION}-*"

# Lock versions
apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
systemctl enable kubelet

echo "Kubernetes tools installed:"
kubeadm version
kubectl version --client
```

### 4.2 kubeadm Config สำหรับ Production

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.0.0.10"  # IP ของ Control Plane node แรก
  bindPort: 6443
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
nodeRegistration:
  name: control-plane-1
  criSocket: unix:///var/run/containerd/containerd.sock
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.28.0"
clusterName: "production-cluster"
controlPlaneEndpoint: "10.0.0.100:6443"  # Load Balancer VIP
certificatesDir: /etc/kubernetes/pki

# etcd Configuration
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      listen-metrics-urls: "http://0.0.0.0:2381"
      auto-compaction-retention: "1"
      quota-backend-bytes: "8589934592"  # 8GB
      snapshot-count: "10000"
      heartbeat-interval: "100"
      election-timeout: "1000"

# API Server Configuration
apiServer:
  certSANs:
  - "10.0.0.100"
  - "k8s-api.example.com"
  - "10.0.0.10"
  - "10.0.0.11"
  - "10.0.0.12"
  - "127.0.0.1"
  extraArgs:
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-log-maxage: "30"
    audit-log-maxbackup: "3"
    audit-log-maxsize: "100"
    audit-policy-file: "/etc/kubernetes/audit-policy.yaml"
    enable-admission-plugins: "NodeRestriction,PodSecurity,EventRateLimit"
    encryption-provider-config: "/etc/kubernetes/encryption-config.yaml"
    profiling: "false"
    request-timeout: "300s"
    service-account-lookup: "true"
    anonymous-auth: "false"
    authorization-mode: "Node,RBAC"
    oidc-issuer-url: "https://your-oidc-provider.example.com"
    oidc-client-id: "kubernetes"
    oidc-groups-claim: "groups"
    oidc-username-claim: "email"
  extraVolumes:
  - name: audit-config
    hostPath: /etc/kubernetes/audit-policy.yaml
    mountPath: /etc/kubernetes/audit-policy.yaml
    readOnly: true
    pathType: File
  - name: audit-log
    hostPath: /var/log/kubernetes
    mountPath: /var/log/kubernetes
    readOnly: false
    pathType: DirectoryOrCreate
  - name: encryption-config
    hostPath: /etc/kubernetes/encryption-config.yaml
    mountPath: /etc/kubernetes/encryption-config.yaml
    readOnly: true
    pathType: File

# Controller Manager
controllerManager:
  extraArgs:
    profiling: "false"
    terminated-pod-gc-threshold: "10"
    node-monitor-period: "5s"
    node-monitor-grace-period: "40s"
    pod-eviction-timeout: "5m0s"
    node-cidr-mask-size: "24"
    bind-address: "0.0.0.0"

# Scheduler
scheduler:
  extraArgs:
    profiling: "false"
    bind-address: "0.0.0.0"

# Networking
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "192.168.0.0/16"
  dnsDomain: "cluster.local"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
maxPods: 250
podsPerCore: 10
systemReserved:
  cpu: "500m"
  memory: "500Mi"
  ephemeral-storage: "1Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "1Gi"
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
evictionSoft:
  memory.available: "1Gi"
  nodefs.available: "15%"
  nodefs.inodesFree: "10%"
  imagefs.available: "20%"
evictionSoftGracePeriod:
  memory.available: "2m"
  nodefs.available: "5m"
  nodefs.inodesFree: "5m"
  imagefs.available: "5m"
evictionMinimumReclaim:
  memory.available: "500Mi"
  nodefs.available: "500Mi"
  imagefs.available: "500Mi"
tlsCipherSuites:
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
rotateCertificates: true
serverTLSBootstrap: true
```

---

## 5. etcd Best Practices

### 5.1 etcd Hardware Recommendations

```bash
# etcd ต้องการ Disk I/O สูง
# แนะนำให้ใช้ NVMe SSD

# ทดสอบ Disk Performance
fio --rw=write --ioengine=sync --fdatasync=1 \
    --directory=/var/lib/etcd \
    --size=22m --bs=2300 --name=mytest

# ผลที่ต้องการสำหรับ etcd:
# - 99th percentile fsync latency < 10ms
# - Sequential write throughput > 50 MB/s

# ตรวจสอบ Disk Latency
ioping -c 100 /var/lib/etcd
```

### 5.2 etcd Optimization

```yaml
# /etc/etcd/etcd.conf.yaml
name: etcd-1
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal

# Network
listen-peer-urls: https://10.0.0.10:2380
listen-client-urls: https://10.0.0.10:2379,https://127.0.0.1:2379
advertise-client-urls: https://10.0.0.10:2379
initial-advertise-peer-urls: https://10.0.0.10:2380

# Cluster Configuration
initial-cluster: etcd-1=https://10.0.0.10:2380,etcd-2=https://10.0.0.11:2380,etcd-3=https://10.0.0.12:2380
initial-cluster-state: new
initial-cluster-token: production-etcd-cluster

# TLS
client-transport-security:
  cert-file: /etc/kubernetes/pki/etcd/server.crt
  key-file: /etc/kubernetes/pki/etcd/server.key
  trusted-ca-file: /etc/kubernetes/pki/etcd/ca.crt
  client-cert-auth: true

peer-transport-security:
  cert-file: /etc/kubernetes/pki/etcd/peer.crt
  key-file: /etc/kubernetes/pki/etcd/peer.key
  trusted-ca-file: /etc/kubernetes/pki/etcd/ca.crt
  peer-client-cert-auth: true

# Performance
snapshot-count: 10000
heartbeat-interval: 100
election-timeout: 1000
max-snapshots: 5
max-wals: 5

# Quota
quota-backend-bytes: 8589934592  # 8GB

# Auto Compaction
auto-compaction-mode: periodic
auto-compaction-retention: "1h"

# Metrics
listen-metrics-urls: http://0.0.0.0:2381

# Logging
log-level: warn
log-outputs:
- /var/log/etcd/etcd.log
```

### 5.3 etcd Backup

```bash
#!/bin/bash
# etcd-backup.sh

BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/etcd-snapshot-${DATE}.db"
RETENTION_DAYS=7

# สร้าง Backup Directory
mkdir -p $BACKUP_DIR

# ทำ Snapshot
ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# ตรวจสอบ Snapshot
ETCDCTL_API=3 etcdctl snapshot status $BACKUP_FILE \
  --write-out=table

# Compress
gzip $BACKUP_FILE
BACKUP_FILE="${BACKUP_FILE}.gz"

# อัพโหลดไปที่ Remote Storage (ตัวอย่าง: S3)
aws s3 cp $BACKUP_FILE s3://my-etcd-backup-bucket/etcd/$(basename $BACKUP_FILE)

# ลบ Backup เก่า
find $BACKUP_DIR -name "*.gz" -mtime +${RETENTION_DAYS} -delete
find $BACKUP_DIR -name "*.db" -mtime +${RETENTION_DAYS} -delete

echo "Backup completed: $BACKUP_FILE"
echo "Backup size: $(du -sh $BACKUP_FILE | cut -f1)"

# Log
echo "$(date): Backup completed - $BACKUP_FILE" >> /var/log/etcd-backup.log
```

### 5.4 etcd Restore

```bash
#!/bin/bash
# etcd-restore.sh

BACKUP_FILE="$1"
RESTORE_DIR="/var/lib/etcd-restore"
ETCD_DATA_DIR="/var/lib/etcd"

if [[ -z "$BACKUP_FILE" ]]; then
    echo "Usage: $0 <backup-file.db.gz>"
    exit 1
fi

echo "=== Restoring etcd from: $BACKUP_FILE ==="

# ดาวน์โหลด Backup (ถ้าอยู่บน S3)
if [[ "$BACKUP_FILE" == s3://* ]]; then
    LOCAL_FILE="/tmp/etcd-restore.db.gz"
    aws s3 cp $BACKUP_FILE $LOCAL_FILE
    BACKUP_FILE=$LOCAL_FILE
fi

# Decompress
if [[ "$BACKUP_FILE" == *.gz ]]; then
    gunzip -c $BACKUP_FILE > /tmp/etcd-restore.db
    BACKUP_FILE="/tmp/etcd-restore.db"
fi

# หยุด Control Plane Pods
# (สำหรับ kubeadm cluster)
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
sleep 10  # รอให้ pods หยุด

# Restore
ETCDCTL_API=3 etcdctl snapshot restore $BACKUP_FILE \
  --name etcd-1 \
  --initial-cluster etcd-1=https://10.0.0.10:2380,etcd-2=https://10.0.0.11:2380,etcd-3=https://10.0.0.12:2380 \
  --initial-cluster-token production-etcd-cluster \
  --initial-advertise-peer-urls https://10.0.0.10:2380 \
  --data-dir $RESTORE_DIR

# ย้ายข้อมูลเก่า
mv $ETCD_DATA_DIR ${ETCD_DATA_DIR}.bak

# ใช้ข้อมูลที่ restore
mv $RESTORE_DIR $ETCD_DATA_DIR

# ตั้งค่า Permissions
chown -R etcd:etcd $ETCD_DATA_DIR

# เริ่ม Control Plane อีกครั้ง
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests

# รอให้ pods ขึ้นมา
echo "Waiting for control plane..."
sleep 60

# ตรวจสอบ
kubectl get nodes
kubectl get pods -A

echo "=== etcd restore completed ==="
```

---

## 6. Audit Policy

### 6.1 Production Audit Policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- "RequestReceived"

rules:
# Log secret, configmap, tokenreviews at Metadata level
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  - group: "authentication.k8s.io"
    resources: ["tokenreviews"]

# Log pod execution command
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]

# Log create, update, delete for most resources
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete", "deletecollection"]
  resources:
  - group: ""
    resources:
    - pods
    - services
    - endpoints
    - persistentvolumeclaims
    - persistentvolumes
    - configmaps
    - serviceaccounts
  - group: "apps"
    resources:
    - deployments
    - daemonsets
    - replicasets
    - statefulsets
  - group: "rbac.authorization.k8s.io"
    resources:
    - roles
    - rolebindings
    - clusterroles
    - clusterrolebindings

# Log node activities
- level: Request
  users: ["system:nodes"]
  verbs: ["update", "patch"]
  resources:
  - group: ""
    resources: ["nodes/status", "pods/status"]

# Log all authentication
- level: Metadata
  nonResourceURLs:
  - "/api*"
  - "/version"
  users: []
  verbs: ["get"]

# Default: don't log
- level: None
  resources:
  - group: ""
    resources: ["events"]

- level: None
  users:
  - "system:kube-scheduler"
  - "system:kube-proxy"
  - "system:apiserver"
  - "system:kube-controller-manager"

# Default catch-all
- level: Request
  omitStages:
  - "RequestReceived"
```

### 6.2 Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

```bash
# สร้าง Encryption Key
head -c 32 /dev/urandom | base64
```

---

## 7. Post-installation Configuration

### 7.1 ติดตั้ง Network Plugin (Calico)

```bash
# ติดตั้ง Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml

cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
  nodeMetricsPort: 9091
  typhaMetricsPort: 9093
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

# รอให้ Calico พร้อม
watch kubectl get tigerastatus
```

### 7.2 ติดตั้ง CoreDNS ที่ optimized

```yaml
# coredns-config.yaml
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
        cache 30 {
           success 9984 30
           denial 9984 5
        }
        loop
        reload
        loadbalance
    }
```

### 7.3 ตั้งค่า RBAC สำหรับ Production

```yaml
# production-rbac.yaml
# Developer Role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec", "services", "configmaps", "endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
# Read-only Role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
- apiGroups: ["", "apps", "autoscaling", "networking.k8s.io", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
# Ops Role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ops
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

### 7.4 Pod Disruption Budgets

```yaml
# pdb-templates.yaml
# PDB สำหรับ Critical Applications
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
  namespace: production
spec:
  minAvailable: 2  # ต้องมีอย่างน้อย 2 pods พร้อมใช้เสมอ
  selector:
    matchLabels:
      app: critical-app
---
# PDB สำหรับ Database
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: database-pdb
  namespace: production
spec:
  maxUnavailable: 1  # Upgrade ทีละ 1 pod
  selector:
    matchLabels:
      app: database
---
# PDB สำหรับ API Server
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  minAvailable: "75%"  # ต้องมีอย่างน้อย 75% พร้อมใช้เสมอ
  selector:
    matchLabels:
      role: api-server
```

---

## 8. Production Checklist

### 8.1 Security Checklist

```bash
#!/bin/bash
# security-check.sh - Production Security Checklist

echo "=== Kubernetes Production Security Checklist ==="

# 1. Anonymous Auth ปิดอยู่หรือเปล่า
echo "1. Checking Anonymous Auth..."
kubectl get configmap -n kube-system kube-apiserver-config -o yaml 2>/dev/null || \
  echo "Check API server flags: --anonymous-auth=false"

# 2. RBAC เปิดอยู่หรือเปล่า
echo "2. Checking RBAC..."
kubectl api-versions | grep rbac

# 3. Network Policies
echo "3. Checking Network Policies..."
kubectl get networkpolicies -A

# 4. Pod Security Standards
echo "4. Checking Pod Security..."
kubectl get namespace -o json | jq '.items[].metadata.labels | select(. != null)'

# 5. Secrets Encryption
echo "5. Checking Encryption at Rest..."
kubectl get pods -n kube-system -l component=kube-apiserver -o yaml | \
  grep encryption-provider-config

# 6. Audit Logging
echo "6. Checking Audit Logging..."
kubectl get pods -n kube-system -l component=kube-apiserver -o yaml | \
  grep audit-log-path

# 7. ETCD Security
echo "7. Checking etcd TLS..."
ps aux | grep etcd | grep -- --client-cert-auth

# 8. Kubelet Authorization
echo "8. Checking Kubelet Authorization..."
curl -sk https://localhost:10250/pods 2>&1 | head -1

echo "=== Security check complete ==="
```

### 8.2 Performance Checklist

```bash
#!/bin/bash
# performance-check.sh

echo "=== Kubernetes Performance Checklist ==="

# 1. Swap ปิดอยู่หรือเปล่า
echo "1. Swap Status:"
free -h | grep Swap

# 2. Transparent Huge Pages
echo "2. Transparent Huge Pages:"
cat /sys/kernel/mm/transparent_hugepage/enabled

# 3. CPU Governor
echo "3. CPU Governor:"
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null || echo "N/A"

# 4. Network Buffer Sizes
echo "4. Network Buffers:"
sysctl net.core.rmem_max net.core.wmem_max

# 5. File Limits
echo "5. File Limits:"
ulimit -n

# 6. Disk I/O Scheduler
echo "6. I/O Scheduler:"
cat /sys/block/sda/queue/scheduler 2>/dev/null || echo "N/A"

# 7. IRQ Balancing
echo "7. IRQ Balance:"
systemctl status irqbalance 2>/dev/null | head -3

echo "=== Performance check complete ==="
```

---

## 9. Workshop: Install Production Cluster

### Workshop Setup

ในบทนี้เราจะติดตั้ง Production Kubernetes Cluster ด้วย:
- 3 Control Plane Nodes (HA)
- 3 Worker Nodes
- HAProxy Load Balancer
- Calico Network Plugin
- etcd Backup Automation

### Prerequisites

```
Infrastructure:
- 1x HAProxy Load Balancer (2 CPU, 2 GB RAM): 10.0.0.100
- 3x Control Plane Nodes (4 CPU, 8 GB RAM): 10.0.0.10-12
- 3x Worker Nodes (8 CPU, 16 GB RAM): 10.0.1.10-12
- All nodes: Ubuntu 22.04 LTS
- All nodes: NVMe SSD สำหรับ etcd nodes
```

### Step 1: เตรียม Load Balancer

```bash
# บน HAProxy Node (10.0.0.100)
apt-get install -y haproxy

cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 50000

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend k8s-api-frontend
    bind *:6443
    mode tcp
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server control-plane-1 10.0.0.10:6443 check inter 3s fall 3 rise 2
    server control-plane-2 10.0.0.11:6443 check inter 3s fall 3 rise 2
    server control-plane-3 10.0.0.12:6443 check inter 3s fall 3 rise 2

listen stats
    bind *:8080
    mode http
    stats enable
    stats uri /haproxy-status
    stats auth admin:password
EOF

systemctl restart haproxy
systemctl enable haproxy

# ทดสอบ HAProxy
curl http://localhost:8080/haproxy-status
```

### Step 2: เตรียม Node ทุกตัว

```bash
# รันบน ALL NODES (Control Planes + Workers)

# 1. อัพเดท System
apt-get update && apt-get upgrade -y

# 2. ปิด Swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 3. Load Modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay && modprobe br_netfilter

# 4. ตั้งค่า sysctl
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 5. ติดตั้ง containerd
# (ใช้ script จากหัวข้อ 3.1)

# 6. ติดตั้ง kubeadm, kubelet, kubectl
# (ใช้ script จากหัวข้อ 4.1)
```

### Step 3: Init Control Plane แรก

```bash
# บน control-plane-1 (10.0.0.10)

# สร้าง Audit Policy
mkdir -p /etc/kubernetes
cat > /etc/kubernetes/audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "apps"
    resources: ["deployments", "statefulsets"]
- level: None
  resources:
  - group: ""
    resources: ["events"]
EOF

# สร้าง kubeadm config
cat > /tmp/kubeadm-init.yaml << 'EOF'
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.0.0.10"
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.28.0"
controlPlaneEndpoint: "10.0.0.100:6443"
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      auto-compaction-retention: "1"
      quota-backend-bytes: "8589934592"
apiServer:
  certSANs:
  - "10.0.0.100"
  - "10.0.0.10"
  - "10.0.0.11"
  - "10.0.0.12"
  - "k8s-api.example.com"
  - "127.0.0.1"
  extraArgs:
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-log-maxage: "30"
    audit-log-maxbackup: "3"
    audit-log-maxsize: "100"
    audit-policy-file: "/etc/kubernetes/audit-policy.yaml"
  extraVolumes:
  - name: audit-policy
    hostPath: /etc/kubernetes/audit-policy.yaml
    mountPath: /etc/kubernetes/audit-policy.yaml
    readOnly: true
    pathType: File
  - name: audit-log
    hostPath: /var/log/kubernetes
    mountPath: /var/log/kubernetes
    pathType: DirectoryOrCreate
networking:
  podSubnet: "192.168.0.0/16"
  serviceSubnet: "10.96.0.0/12"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
maxPods: 250
EOF

# สร้าง Log Directory
mkdir -p /var/log/kubernetes

# Init Cluster
kubeadm init --config /tmp/kubeadm-init.yaml --upload-certs

# ตั้งค่า kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 4: Join Control Planes

```bash
# บน control-plane-2 และ control-plane-3
# ใช้ join command จาก output ของ kubeadm init
# ตัวอย่าง:

kubeadm join 10.0.0.100:6443 \
  --token your-token \
  --discovery-token-ca-cert-hash sha256:your-hash \
  --control-plane \
  --certificate-key your-cert-key \
  --apiserver-advertise-address 10.0.0.11  # แก้ตาม node

# Setup kubectl บน control plane nodes
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

### Step 5: Join Worker Nodes

```bash
# บน worker nodes
kubeadm join 10.0.0.100:6443 \
  --token your-token \
  --discovery-token-ca-cert-hash sha256:your-hash

# ถ้า Token หมดอายุ ให้สร้างใหม่:
# (รันบน Control Plane)
kubeadm token create --print-join-command
```

### Step 6: ติดตั้ง Calico

```bash
# บน Control Plane แรก
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml

cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
EOF

# รอให้ Calico พร้อม
kubectl wait --for=condition=ready pod -n calico-system -l app.kubernetes.io/name=calico-node --timeout=300s

# ตรวจสอบ Nodes
kubectl get nodes
```

### Step 7: ตั้งค่า etcd Backup อัตโนมัติ

```bash
# บน Control Plane แรก
# สร้าง Backup Script
cat > /usr/local/bin/etcd-backup.sh << 'SCRIPT'
#!/bin/bash
BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/etcd-${DATE}.db"

mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

ETCDCTL_API=3 etcdctl snapshot status $BACKUP_FILE
gzip $BACKUP_FILE

# ลบ Backup เก่ากว่า 7 วัน
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

echo "$(date): Backup OK - ${BACKUP_FILE}.gz" >> /var/log/etcd-backup.log
SCRIPT

chmod +x /usr/local/bin/etcd-backup.sh

# ตั้งค่า Cron Job (ทุก 6 ชั่วโมง)
echo "0 */6 * * * root /usr/local/bin/etcd-backup.sh" > /etc/cron.d/etcd-backup
systemctl restart cron

# ทดสอบ Backup
/usr/local/bin/etcd-backup.sh
ls -lh /backup/etcd/
```

### Step 8: ตรวจสอบ Cluster

```bash
# Comprehensive Cluster Health Check

echo "=== Cluster Health Check ==="

# 1. Nodes
echo "--- Nodes ---"
kubectl get nodes -o wide

# 2. System Pods
echo "--- System Pods ---"
kubectl get pods -A

# 3. Component Status
echo "--- Component Status ---"
kubectl get componentstatuses

# 4. etcd Health
echo "--- etcd Health ---"
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# 5. etcd Member List
echo "--- etcd Members ---"
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://10.0.0.10:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -w table

# 6. Test Deployment
echo "--- Test Deployment ---"
kubectl create deployment test --image=nginx --replicas=3
kubectl wait --for=condition=available deployment/test --timeout=120s
kubectl get pods -l app=test -o wide
kubectl delete deployment test

echo "=== Cluster is Ready for Production ==="
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Requirements** - Hardware, Software, Network requirements สำหรับ Production
2. **OS Preparation** - การตั้งค่า OS อย่างถูกต้อง
3. **Container Runtime** - ติดตั้งและตั้งค่า containerd
4. **kubeadm** - Installation และ Production Config
5. **etcd** - Best Practices, Backup, Restore
6. **Audit Policy** - ตั้งค่า Audit Logging
7. **Encryption** - Encryption at Rest สำหรับ Secrets
8. **Workshop** - ติดตั้ง Production Cluster แบบ HA

## แบบฝึกหัด

1. ติดตั้ง Cluster ด้วย kubeadm แบบ Single Node สำหรับทดสอบ
2. ทำ etcd Backup และ Restore ให้สำเร็จ
3. ตั้งค่า Encryption at Rest สำหรับ Secrets
4. Implement Audit Policy ที่ Log operations ทั้งหมดสำหรับ Secrets
5. Upgrade Cluster จาก 1.27 ไปเป็น 1.28

## References

- [kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [etcd Documentation](https://etcd.io/docs/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
