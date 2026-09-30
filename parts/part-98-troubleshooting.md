# Part 98: Kubernetes Troubleshooting Guide

## บทนำ

การ Troubleshoot ปัญหาใน Kubernetes ต้องการความเข้าใจในสถาปัตยกรรมและเครื่องมือที่หลากหลาย บทนี้เป็น Guide ที่ครอบคลุมปัญหาที่พบบ่อยและวิธีการแก้ไข

## สารบัญ

1. Troubleshooting Framework
2. Pod Troubleshooting
3. Network Debugging
4. Storage Issues
5. Cluster Issues
6. Performance Issues
7. Useful Commands Reference
8. Workshop: Troubleshooting Scenarios

---

## 1. Troubleshooting Framework

### 1.1 5-Step Troubleshooting Process

```
Step 1: Identify Symptoms
- อะไรที่ไม่ทำงาน?
- Error message คืออะไร?
- เมื่อไหร่ที่เริ่มเกิดปัญหา?

Step 2: Gather Information
- kubectl describe
- kubectl logs
- kubectl events
- Monitoring dashboards

Step 3: Hypothesize
- ปัญหาเป็นไปได้อะไรบ้าง?
- Layer ไหนที่น่าจะเป็นปัญหา?

Step 4: Test and Verify
- ทดสอบ Hypothesis ทีละตัว
- Isolate ปัญหา

Step 5: Fix and Document
- Apply Fix
- ทดสอบ Fix
- Document ปัญหาและวิธีแก้
```

### 1.2 Kubernetes Layers

```
Layer 7: Application
Layer 6: Pod/Container
Layer 5: Deployment/Service
Layer 4: Kubernetes Object
Layer 3: Node
Layer 2: Container Runtime
Layer 1: OS/Network/Storage
```

---

## 2. Pod Troubleshooting

### 2.1 Pod Status Reference

```
Pending      : Pod ยังไม่ได้ถูก Schedule หรือ Image ยังโหลดไม่เสร็จ
Running      : Pod กำลังทำงาน
Succeeded    : Pod เสร็จแล้ว (สำหรับ Job)
Failed       : Pod ทำงานล้มเหลว
Unknown      : ไม่สามารถติดต่อ Node ได้
Terminating  : Pod กำลังถูก Delete

Container States:
Waiting      : Container รอ (เหตุผลเช่น ImagePullBackOff)
Running      : Container ทำงานอยู่
Terminated   : Container หยุดทำงานแล้ว
```

### 2.2 Common Pod Issues

```bash
#!/bin/bash
# diagnose-pod.sh <namespace> <pod-name>

NS="${1:-default}"
POD="${2}"

if [ -z "$POD" ]; then
  echo "Usage: $0 <namespace> <pod-name>"
  exit 1
fi

echo "=== Diagnosing Pod: $NS/$POD ==="

# 1. Basic Info
echo ""
echo "--- Pod Status ---"
kubectl get pod $POD -n $NS -o wide

# 2. Events
echo ""
echo "--- Recent Events ---"
kubectl get events -n $NS --field-selector involvedObject.name=$POD \
  --sort-by='.lastTimestamp' | tail -20

# 3. Describe
echo ""
echo "--- Pod Description ---"
kubectl describe pod $POD -n $NS

# 4. Logs
echo ""
echo "--- Container Logs (last 100 lines) ---"
kubectl logs $POD -n $NS --all-containers=true --tail=100 2>/dev/null || \
  echo "Cannot get logs (pod may not be running)"

# 5. Previous Logs (if restarted)
echo ""
echo "--- Previous Container Logs ---"
kubectl logs $POD -n $NS --previous --tail=50 2>/dev/null || \
  echo "No previous logs"

# 6. Resource Usage
echo ""
echo "--- Resource Usage ---"
kubectl top pod $POD -n $NS 2>/dev/null || echo "Metrics not available"
```

### 2.3 ImagePullBackOff / ErrImagePull

```bash
# ตรวจสอบ Image ที่ระบุ
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'

# ตรวจสอบ ImagePullSecrets
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'

# ดู Events สำหรับ Error Details
kubectl describe pod <pod-name> | grep -A 5 "Warning"

# ทดสอบ Pull Image โดยตรง
kubectl run test-pull \
  --image=myregistry.example.com/myapp:v1 \
  --restart=Never \
  --dry-run=client

# สร้าง Image Pull Secret
kubectl create secret docker-registry myregistry-secret \
  --docker-server=myregistry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=admin@example.com

# เพิ่ม Secret ใน Pod
kubectl patch deployment <deployment> -p '
{
  "spec": {
    "template": {
      "spec": {
        "imagePullSecrets": [{"name": "myregistry-secret"}]
      }
    }
  }
}'
```

### 2.4 CrashLoopBackOff

```bash
# ดู Logs ของ Container ที่ Crash
kubectl logs <pod-name> --previous

# ดู Exit Code
kubectl describe pod <pod-name> | grep -E "Exit Code|Reason|Message"

# รัน Container ด้วย Debug Command
kubectl run debug-pod \
  --image=myapp:v1 \
  --restart=Never \
  --command -- /bin/sh -c "sleep 3600"

kubectl exec -it debug-pod -- /bin/sh

# ตรวจสอบ Config Files
kubectl exec -it debug-pod -- cat /etc/app/config.yaml

# Common Causes:
# 1. Application Error - ดู logs
# 2. OOMKilled - ดู reason: OOMKilled, เพิ่ม memory limits
# 3. Config Error - ดู config files
# 4. Missing Dependencies - ดู error messages
```

### 2.5 OOMKilled

```bash
# ตรวจสอบว่า OOMKilled หรือเปล่า
kubectl describe pod <pod-name> | grep -E "OOMKilled|Reason|Exit Code"

# ดู Memory Usage
kubectl top pod <pod-name>

# ดู Memory Requests/Limits
kubectl get pod <pod-name> -o json | \
  jq '.spec.containers[].resources'

# เพิ่ม Memory Limits
kubectl set resources deployment <deployment> \
  --containers=<container-name> \
  --limits=memory=2Gi \
  --requests=memory=512Mi

# ตรวจสอบ Memory Leak ด้วย Debug Container
kubectl debug -it <pod-name> --image=ubuntu --target=<container-name>
apt-get install -y htop
htop
```

### 2.6 Pending Pod

```bash
# ตรวจสอบว่า Schedule ไม่ได้เพราะอะไร
kubectl describe pod <pod-name> | grep -A 10 "Events"

# Common Causes:
# 1. Insufficient Resources
kubectl describe node | grep -E "Allocated|Requests|Limits"

# 2. Node Selector ไม่ match
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeSelector}'
kubectl get nodes --show-labels

# 3. Taints/Tolerations
kubectl describe nodes | grep Taints
kubectl get pod <pod-name> -o jsonpath='{.spec.tolerations}'

# 4. PVC ไม่พร้อม
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name>

# ดู Scheduler Events
kubectl get events --field-selector reason=FailedScheduling
```

---

## 3. Network Debugging

### 3.1 Service DNS Resolution

```bash
# ทดสอบ DNS จากใน Pod
kubectl run dns-debug \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -- nslookup kubernetes.default

# ทดสอบ Service DNS
kubectl run dns-debug \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -- nslookup my-service.my-namespace.svc.cluster.local

# ทดสอบด้วย dig
kubectl run dns-debug \
  --image=tutum/dnsutils \
  --restart=Never \
  --rm -it \
  -- dig my-service.my-namespace.svc.cluster.local

# ดู CoreDNS Logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# ตรวจสอบ CoreDNS Config
kubectl get configmap coredns -n kube-system -o yaml

# ตรวจสอบ /etc/resolv.conf ใน Pod
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

### 3.2 Service Connectivity

```bash
# ทดสอบ Service จาก Pod
kubectl run test-client \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -- wget -O- http://my-service:80/

# ตรวจสอบ Endpoints
kubectl get endpoints my-service -n my-namespace

# ถ้า Endpoints ว่าง = Selector ไม่ match กับ Pod
kubectl get pod -l app=my-app  # Selector ของ Service
kubectl describe service my-service | grep Selector

# ตรวจสอบ Port ที่ถูกต้อง
kubectl get service my-service -o yaml | grep -E "port|targetPort"
kubectl get pod -l app=my-app -o yaml | grep containerPort

# ทดสอบ Connection โดยตรงผ่าน Pod IP
POD_IP=$(kubectl get pod <pod-name> -o jsonpath='{.status.podIP}')
kubectl run test-client \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -- wget -O- http://${POD_IP}:8080/
```

### 3.3 Network Policy Debugging

```bash
# ตรวจสอบ Network Policies
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# ทดสอบ Connectivity ระหว่าง Pods
kubectl run test-a \
  --image=busybox \
  --labels="app=test-a" \
  --restart=Never

kubectl run test-b \
  --image=busybox \
  --labels="app=test-b" \
  --restart=Never

TEST_B_IP=$(kubectl get pod test-b -o jsonpath='{.status.podIP}')
kubectl exec test-a -- wget -O- --timeout=3 http://$TEST_B_IP/ && echo "CONNECTED" || echo "BLOCKED"

# Debug ด้วย Netshoot
kubectl run netshoot \
  --image=nicolaka/netshoot \
  --restart=Never -it \
  -- /bin/bash

# ใน netshoot:
# nmap -p 80 <target-ip>
# traceroute <target-ip>
# tcpdump -i eth0 host <target-ip>
# ss -tulpn  # ดู listening ports

# ตรวจสอบ iptables rules
kubectl get pod -n <namespace> -o wide
# SSH เข้า Node แล้ว:
# iptables -L -n | grep <pod-ip>
```

### 3.4 Ingress Debugging

```bash
# ตรวจสอบ Ingress
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# ตรวจสอบ Ingress Controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100

# ตรวจสอบ Ingress Config
kubectl exec -n ingress-nginx -it <ingress-pod> \
  -- cat /etc/nginx/nginx.conf | grep -A 20 "server_name"

# ทดสอบ Host Header
curl -v -H "Host: myapp.example.com" http://<ingress-ip>/

# ตรวจสอบ TLS Certificate
kubectl get secret <tls-secret> -n <namespace> -o yaml
openssl x509 -noout -text -in <(kubectl get secret <tls-secret> \
  -o jsonpath='{.data.tls\.crt}' | base64 -d)
```

---

## 4. Storage Issues

### 4.1 PVC Stuck in Pending

```bash
# ตรวจสอบ PVC Status
kubectl describe pvc <pvc-name> -n <namespace>

# Common Causes:
# 1. No Available PV
kubectl get pv
kubectl get storageclass

# 2. StorageClass ไม่มี
kubectl get storageclass
kubectl get pvc <pvc-name> -o jsonpath='{.spec.storageClassName}'

# 3. PV ไม่ match PVC requirements
kubectl describe pv <pv-name>

# สร้าง PV Manual (สำหรับ Static Provisioning)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
EOF

# แก้ PVC ให้ใช้ StorageClass ที่มี
kubectl patch pvc <pvc-name> -p '{"spec":{"storageClassName":"standard"}}'
```

### 4.2 Volume Mount ล้มเหลว

```bash
# ดู Error ใน Events
kubectl describe pod <pod-name> | grep -A 5 "Warning"

# Common Issues:
# 1. Permission denied
kubectl exec -it <pod-name> -- ls -la /mnt/data
kubectl exec -it <pod-name> -- id  # ดู User ID

# แก้ด้วย fsGroup
# (ใน Pod spec):
# securityContext:
#   fsGroup: 2000

# 2. Already Mounted (ReadWriteOnce ใช้ 2 nodes)
kubectl get pvc <pvc-name>
kubectl describe pv <pv-name>

# 3. CSI Driver Error
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system <csi-pod-name>

# ตรวจสอบ Mount
kubectl exec -it <pod-name> -- mount | grep /mnt
kubectl exec -it <pod-name> -- df -h
```

### 4.3 Data Persistence ปัญหา

```bash
# ตรวจสอบ PV Reclaim Policy
kubectl get pv -o custom-columns='NAME:.metadata.name,POLICY:.spec.persistentVolumeReclaimPolicy'

# ตรวจสอบว่า PV ถูก Delete ไปหรือเปล่า
kubectl get pv | grep Released

# Recover Released PV
# 1. ดู PV ที่ Released
kubectl get pv <pv-name> -o yaml | grep claimRef

# 2. ลบ claimRef เพื่อ Reclaim
kubectl patch pv <pv-name> -p '{"spec":{"claimRef":null}}'

# 3. สร้าง PVC ใหม่ที่ match PV นี้
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: recovered-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeName: <pv-name>  # ระบุ PV โดยตรง
EOF
```

---

## 5. Cluster Issues

### 5.1 Node NotReady

```bash
# ตรวจสอบ Node Status
kubectl get nodes
kubectl describe node <node-name>

# ดู Node Conditions
kubectl get node <node-name> -o json | \
  jq '.status.conditions[] | select(.type != null)'

# SSH เข้า Node แล้วตรวจสอบ:
# 1. Kubelet Status
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager

# 2. Container Runtime
systemctl status containerd
crictl ps

# 3. Network
ping <api-server-ip>
curl -k https://<api-server-ip>:6443/healthz

# 4. Disk Space
df -h
du -sh /var/lib/docker /var/lib/containerd /var/lib/kubelet

# 5. Memory
free -h
cat /proc/meminfo | grep -E "MemFree|MemAvailable"

# 6. System Logs
journalctl -xe -n 200
dmesg | tail -50
```

### 5.2 API Server ไม่ตอบสนอง

```bash
# ตรวจสอบ API Server Pod
kubectl -n kube-system get pod -l component=kube-apiserver

# ดู API Server Logs
kubectl -n kube-system logs -l component=kube-apiserver --tail=100

# SSH เข้า Control Plane:
# ตรวจสอบ Static Pod
ls /etc/kubernetes/manifests/

# ดู API Server Process
ps aux | grep kube-apiserver

# ตรวจสอบ Certificates
openssl x509 -noout -text -in /etc/kubernetes/pki/apiserver.crt | \
  grep -E "Not Before|Not After"

# ตรวจสอบ etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

### 5.3 Certificate Expiration

```bash
# ตรวจสอบ Certificate Expiration
kubeadm certs check-expiration

# Renew Certificates
kubeadm certs renew all

# ตรวจสอบแต่ละ Cert
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -noout -text -in $cert | grep -E "Not Before|Not After"
done

# ตั้งค่า Auto-renewal (ใน kubelet config)
# rotateCertificates: true
# serverTLSBootstrap: true
```

### 5.4 etcd Issues

```bash
# ตรวจสอบ etcd Health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# ดู etcd Member Status
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -w table

# ตรวจสอบ Disk Usage
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -w json | jq '.[].Status.dbSize'

# Compact etcd (ลด Size)
ETCDCTL_API=3 etcdctl compact \
  $(ETCDCTL_API=3 etcdctl endpoint status \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
    -w json | jq '.[].Status.header.revision')

# Defragment etcd
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

---

## 6. Performance Issues

### 6.1 High CPU Usage

```bash
# ดู Node ที่ใช้ CPU สูง
kubectl top nodes --sort-by=cpu

# ดู Pods ที่กิน CPU มาก
kubectl top pods -A --sort-by=cpu | head -20

# ดู CPU Throttling
kubectl exec -it <pod-name> -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled

# ตรวจสอบ CPU Limits
kubectl get pod <pod-name> -o json | \
  jq '.spec.containers[].resources.limits.cpu'

# เพิ่ม CPU Limits
kubectl set resources deployment <deployment> \
  --containers=<container> \
  --limits=cpu=2000m

# SSH เข้า Node แล้วดู Process
top -H  # ดู Threads
perf top  # ดู CPU Profile (ต้องติดตั้ง)
```

### 6.2 High Memory Usage

```bash
# ดู Memory Usage
kubectl top nodes --sort-by=memory
kubectl top pods -A --sort-by=memory | head -20

# ดู Memory Details
kubectl exec -it <pod-name> -- cat /proc/meminfo

# ตรวจสอบ Memory Limits และ Usage
kubectl get pod <pod-name> -o json | \
  jq '.spec.containers[].resources'

# ดู OOM Events
kubectl get events -A --field-selector reason=OOMKilling
dmesg | grep -i "oom killer"

# Heap Dump สำหรับ Java
kubectl exec -it <java-pod> -- jcmd 1 VM.heap_info
kubectl exec -it <java-pod> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <java-pod>:/tmp/heap.hprof ./heap.hprof
```

### 6.3 Slow Pod Startup

```bash
# วัด Pod Startup Time
start=$(date +%s%3N)
kubectl run test-startup --image=nginx --restart=Never
kubectl wait --for=condition=ready pod/test-startup --timeout=120s
end=$(date +%s%3N)
echo "Startup time: $((end - start))ms"
kubectl delete pod test-startup

# ตรวจสอบ Image Pull Time
kubectl get events --field-selector reason=Pulled

# Enable Image Pre-pull
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: image-prepuller
  template:
    metadata:
      labels:
        name: image-prepuller
    spec:
      initContainers:
      - name: pull-image
        image: myapp:v1
        command: ["sh", "-c", "echo Image pulled"]
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
EOF
```

---

## 7. Useful Commands Reference

### 7.1 Essential kubectl Commands

```bash
# ดู Cluster Info
kubectl cluster-info
kubectl get componentstatuses

# ดู All Resources
kubectl get all -A

# ดู Resource ทุกประเภท
kubectl api-resources

# ดู Events เรียงตามเวลา
kubectl get events -A --sort-by='.lastTimestamp' | tail -50

# ดู Logs หลาย Containers
kubectl logs <pod> --all-containers=true -f

# Exec ใน Running Container
kubectl exec -it <pod> -c <container> -- /bin/sh

# Copy Files
kubectl cp <pod>:/path/to/file ./local-file
kubectl cp ./local-file <pod>:/path/to/file

# Port Forward
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward service/<service-name> 8080:80

# Watch Resources
kubectl get pods -w
kubectl get events -w

# Get Resources in JSON/YAML
kubectl get pod <pod> -o json
kubectl get pod <pod> -o yaml

# jsonpath Query
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'

# Label Selector
kubectl get pods -l app=myapp,env=prod
kubectl get pods -l 'env in (prod,staging)'

# Force Delete (ระวัง!)
kubectl delete pod <pod> --force --grace-period=0

# Rollout Commands
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
```

### 7.2 Debug Commands

```bash
# Create Debug Container (ephemeral container)
kubectl debug -it <pod-name> \
  --image=busybox \
  --target=<container-name>

# Debug Node
kubectl debug node/<node-name> \
  -it \
  --image=ubuntu

# Debug ด้วย Copy ของ Pod
kubectl debug <pod-name> -it \
  --copy-to=debug-pod \
  --image=ubuntu \
  --share-processes

# Network Debug Container
kubectl run netshoot \
  --image=nicolaka/netshoot \
  --restart=Never -it \
  -- /bin/bash

# tcpdump บน Node
kubectl debug node/<node-name> -it --image=ubuntu -- \
  nsenter -t 1 -n tcpdump -i eth0 -w /tmp/capture.pcap

# strace Process
kubectl exec -it <pod> -- strace -p 1 2>&1 | head -50
```

### 7.3 Monitoring Commands

```bash
# ดู Resource Requests/Limits ทั้ง Cluster
kubectl get pods -A -o json | jq '
  [.items[] |
  {
    namespace: .metadata.namespace,
    pod: .metadata.name,
    containers: [.spec.containers[] | {
      name: .name,
      cpuRequest: .resources.requests.cpu,
      memRequest: .resources.requests.memory,
      cpuLimit: .resources.limits.cpu,
      memLimit: .resources.limits.memory
    }]
  }]' | head -100

# ดู Node Resource Capacity vs Allocatable
kubectl get nodes -o json | jq '
  .items[] |
  {
    name: .metadata.name,
    capacity: .status.capacity,
    allocatable: .status.allocatable
  }'

# ดู Image ที่ใช้อยู่ทั้งหมด
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | \
  sort | uniq -c | sort -rn

# ตรวจสอบ Cluster Capacity
kubectl describe nodes | \
  awk '/^  Resource/{resource=$2} /^  (Requests|Limits)/{printf "%s %s\n", $1, $2}' | \
  grep -E "(cpu|memory)"
```

---

## 8. Workshop: Troubleshooting Scenarios

### Scenario 1: Application ไม่ Start

```bash
# สร้าง Broken Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: nginx:broken-tag  # Wrong tag
        ports:
        - containerPort: 80
EOF

# ทดสอบ: วิเคราะห์ปัญหา
echo "=== Exercise: Debug ImagePullBackOff ==="
echo ""
echo "Steps:"
echo "1. kubectl get pods"
echo "2. kubectl describe pod <pod-name>"
echo "3. ดู Events: Warning  Failed..."
echo "4. แก้ image tag"
echo "5. kubectl set image deployment/broken-app app=nginx:latest"
echo ""

# เฉลย
kubectl get pods -l app=broken-app
kubectl describe pods -l app=broken-app | grep -A 3 "Warning"
# แก้ Image
kubectl set image deployment/broken-app app=nginx:latest
kubectl rollout status deployment/broken-app
```

### Scenario 2: Service ไม่ Accessible

```bash
# สร้าง Service ที่มีปัญหา Selector
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend        # Label: app=backend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: wrong-backend      # Bug: Selector ไม่ match
  ports:
  - port: 80
    targetPort: 80
EOF

echo "=== Exercise: Debug Service ==="
echo ""
echo "Steps:"
echo "1. kubectl get endpoints backend-svc"
echo "2. สังเกตว่า Endpoints ว่าง"
echo "3. kubectl describe service backend-svc | grep Selector"
echo "4. kubectl get pods --show-labels | grep backend"
echo "5. แก้ Selector ให้ตรงกัน"
echo ""

# เฉลย
echo "Current Endpoints:"
kubectl get endpoints backend-svc

echo ""
echo "Service Selector:"
kubectl describe service backend-svc | grep Selector

echo ""
echo "Pod Labels:"
kubectl get pods -l app=backend --show-labels

# แก้ Selector
kubectl patch service backend-svc -p '{"spec":{"selector":{"app":"backend"}}}'

echo ""
echo "After Fix - Endpoints:"
kubectl get endpoints backend-svc
```

### Scenario 3: Resource Exhaustion

```bash
# สร้าง Pod ที่กิน Resources มาก
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-hog
  namespace: default
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "8", "--vm", "2", "--vm-bytes", "512M"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"    # จงใจตั้งไว้น้อย
        memory: "128Mi"
EOF

echo "=== Exercise: Debug Resource Issues ==="
echo ""
echo "Steps:"
echo "1. kubectl top pod resource-hog"
echo "2. kubectl describe pod resource-hog | grep -E 'OOM|CPU|Memory'"
echo "3. kubectl get events --field-selector involvedObject.name=resource-hog"
echo "4. เพิ่ม Resource Limits"
echo ""

sleep 30

# ดู Resource Usage
kubectl top pod resource-hog 2>/dev/null || echo "Waiting for metrics..."
kubectl describe pod resource-hog | grep -E "OOMKilled|Reason|Exit"

# Cleanup
kubectl delete pod resource-hog
```

### Scenario 4: Network Policy Block

```bash
# สร้าง NetworkPolicy ที่ Block Traffic
cat <<'EOF' | kubectl apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: server-svc
  namespace: default
spec:
  selector:
    app: server
  ports:
  - port: 80
    targetPort: 80
---
# Network Policy ที่ Block Traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-traffic
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress: []  # Block all ingress
EOF

kubectl wait --for=condition=available deployment/client deployment/server --timeout=60s

echo "=== Exercise: Debug Network Policy ==="
echo ""
echo "Expected: client cannot reach server"
CLIENT_POD=$(kubectl get pod -l app=client -o name | head -1 | cut -d/ -f2)
kubectl exec $CLIENT_POD -- wget -O- --timeout=3 http://server-svc/ && echo "SUCCESS" || echo "BLOCKED (expected)"

echo ""
echo "Steps to fix:"
echo "1. kubectl get networkpolicies"
echo "2. kubectl describe networkpolicy block-traffic"
echo "3. แก้ Policy ให้ Client เข้าถึง Server ได้"
echo ""

# เฉลย - แก้ Policy
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-traffic
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - port: 80
EOF

sleep 5
echo "After Fix:"
kubectl exec $CLIENT_POD -- wget -O- --timeout=3 http://server-svc/ > /dev/null && echo "SUCCESS" || echo "STILL BLOCKED"

# Cleanup
kubectl delete deployment client server
kubectl delete service server-svc
kubectl delete networkpolicy block-traffic
```

### Scenario 5: PVC Stuck

```bash
# สร้าง PVC ที่ Stuck
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stuck-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: nonexistent-storage  # StorageClass ที่ไม่มี
  resources:
    requests:
      storage: 10Gi
EOF

echo "=== Exercise: Debug Stuck PVC ==="
echo ""
echo "kubectl get pvc stuck-pvc"
kubectl get pvc stuck-pvc

echo ""
echo "kubectl describe pvc stuck-pvc"
kubectl describe pvc stuck-pvc

echo ""
echo "Steps:"
echo "1. ดู StorageClasses ที่มี: kubectl get storageclass"
echo "2. แก้ PVC ให้ใช้ StorageClass ที่มีอยู่"
echo ""

kubectl get storageclass

# แก้ - ต้อง Delete แล้ว Create ใหม่ (ไม่สามารถแก้ spec ได้)
kubectl delete pvc stuck-pvc

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stuck-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: $(kubectl get storageclass -o jsonpath='{.items[0].metadata.name}')
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc stuck-pvc
kubectl delete pvc stuck-pvc
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Framework** - 5-Step Troubleshooting Process
2. **Pod Issues** - ImagePullBackOff, CrashLoopBackOff, OOMKilled, Pending
3. **Network Debugging** - DNS, Service, Network Policies, Ingress
4. **Storage Issues** - PVC Stuck, Volume Mount, Data Persistence
5. **Cluster Issues** - Node NotReady, API Server, Certificates, etcd
6. **Performance** - High CPU/Memory, Slow Startup
7. **Commands Reference** - Essential kubectl commands
8. **Workshop** - Real-world troubleshooting scenarios

## แบบฝึกหัด

1. สร้าง Pod ที่มีปัญหา ImagePullBackOff และแก้ให้ได้
2. Debug Service ที่ Endpoints ว่าง
3. แก้ CrashLoopBackOff ที่เกิดจาก Config Error
4. ทำ etcd Defragmentation
5. Trace Network Traffic ด้วย tcpdump

## Quick Reference

```bash
# Most Used Commands
kubectl get pods -A               # ดู Pods ทุก Namespace
kubectl describe pod <pod>        # รายละเอียด Pod
kubectl logs <pod> -f             # Logs แบบ Real-time
kubectl exec -it <pod> -- sh      # เข้า Container
kubectl get events --sort-by=.lastTimestamp  # Events เรียงตามเวลา
kubectl top pods -A               # Resource Usage
kubectl rollout undo deployment   # Rollback Deployment
kubectl describe node <node>      # Node Details
```

## References

- [Kubernetes Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Kubernetes Network Debugging](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubernetes Cluster Administration](https://kubernetes.io/docs/tasks/administer-cluster/)
