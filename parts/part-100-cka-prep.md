# Part 100: CKA (Certified Kubernetes Administrator) Exam Guide

## บทนำ

Certified Kubernetes Administrator (CKA) คือ Certification จาก CNCF ที่ยืนยันทักษะการบริหาร Kubernetes Cluster บทนี้ครอบคลุมทุก Exam Domain พร้อม Practice Questions มากกว่า 300 ข้อ

## ข้อมูล Exam

```
Exam Format: Performance-based (Hands-on)
Duration: 2 hours
Pass Score: 66%
Cost: $395 USD
Validity: 3 years
Number of Questions: 15-20 tasks
Open Book: Yes (Kubernetes.io documentation)

Exam Domains (v1.28):
- Cluster Architecture, Installation & Configuration: 25%
- Workloads & Scheduling: 15%
- Services & Networking: 20%
- Storage: 10%
- Troubleshooting: 30%
```

## สารบัญ

1. Cluster Architecture, Installation & Configuration (25%)
2. Workloads & Scheduling (15%)
3. Services & Networking (20%)
4. Storage (10%)
5. Troubleshooting (30%)
6. Exam Tips & Tricks
7. Practice Labs (300+ คำถาม)

---

## 1. Cluster Architecture, Installation & Configuration (25%)

### 1.1 Cluster Setup

```bash
# kubeadm init
kubeadm init \
  --apiserver-advertise-address=10.0.0.10 \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version=1.28.0

# Setup kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Join Worker
kubeadm join 10.0.0.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Generate Join Token
kubeadm token create --print-join-command

# Upgrade Cluster
kubeadm upgrade plan
kubeadm upgrade apply v1.28.0

# Upgrade kubelet
apt-get install -y kubelet=1.28.0-*
systemctl restart kubelet
```

### 1.2 RBAC

```bash
# สร้าง Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n default

# สร้าง RoleBinding
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --user=john \
  -n default

# สร้าง ClusterRole
kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes

# สร้าง ClusterRoleBinding
kubectl create clusterrolebinding node-viewer-binding \
  --clusterrole=node-viewer \
  --user=john

# ตรวจสอบ Permission
kubectl auth can-i get pods --as john
kubectl auth can-i get nodes --as john --all-namespaces
```

### 1.3 Certificate Management

```bash
# สร้าง CSR สำหรับ User
openssl genrsa -out john.key 2048
openssl req -new -key john.key -subj "/CN=john/O=dev-team" -out john.csr

# Submit CSR
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  request: $(cat john.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF

# Approve CSR
kubectl certificate approve john

# รับ Certificate
kubectl get csr john -o jsonpath='{.status.certificate}' | base64 -d > john.crt

# สร้าง kubeconfig
kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key

kubectl config set-context john-context \
  --cluster=kubernetes \
  --user=john

kubectl config use-context john-context
```

### 1.4 etcd Backup & Restore

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify Backup
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir /var/lib/etcd-backup

# อัพเดท etcd Static Pod ให้ใช้ data-dir ใหม่
```

---

## 2. Workloads & Scheduling (15%)

### 2.1 Deployments

```bash
# สร้าง Deployment
kubectl create deployment nginx \
  --image=nginx:1.25 \
  --replicas=3 \
  --port=80

# Scale
kubectl scale deployment nginx --replicas=5

# Update Image
kubectl set image deployment/nginx nginx=nginx:1.26

# Rollout
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2

# Pause/Resume
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
```

### 2.2 DaemonSet, StatefulSet

```yaml
# DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: agent
        image: agent:latest
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 2.3 Scheduling

```yaml
# nodeSelector
spec:
  nodeSelector:
    disktype: ssd

# NodeAffinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["zone-a", "zone-b"]

# PodAffinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["cache"]
        topologyKey: kubernetes.io/hostname

# Taints and Tolerations
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 key=value:NoSchedule-  # Remove

spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

---

## 3. Services & Networking (20%)

### 3.1 Services

```bash
# สร้าง Service
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP \
  --name=nginx-svc

# NodePort
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  --type=NodePort

# LoadBalancer
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer

# ดู Service Details
kubectl get svc nginx-svc
kubectl describe svc nginx-svc
kubectl get endpoints nginx-svc
```

### 3.2 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
```

### 3.3 Network Policies

```yaml
# Allow only specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

---

## 4. Storage (10%)

### 4.1 PV and PVC

```yaml
# PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local
  hostPath:
    path: /mnt/data
---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local
  resources:
    requests:
      storage: 5Gi
---
# Pod using PVC
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### 4.2 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## 5. Troubleshooting (30%)

### 5.1 Common Troubleshooting Commands

```bash
# Node Troubleshooting
kubectl get nodes
kubectl describe node <node-name>
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node-name>

# Pod Troubleshooting
kubectl get pods -A
kubectl describe pod <pod-name>
kubectl logs <pod-name> -f
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- /bin/sh

# Cluster Components
kubectl get componentstatuses
kubectl get pods -n kube-system

# API Server
curl -k https://localhost:6443/healthz
curl -k https://localhost:6443/version

# kubelet
systemctl status kubelet
journalctl -u kubelet -n 50

# etcd
ETCDCTL_API=3 etcdctl endpoint health
```

---

## 6. Exam Tips & Tricks

### 6.1 Time Management

```
Total Time: 120 minutes (2 hours)
~15-20 Questions
Average: 6-8 minutes per question

Easy Questions (1 point): 2-3 minutes
Medium Questions (4 points): 5-8 minutes
Hard Questions (8 points): 10-15 minutes

Strategy:
1. Read all questions first (5 min)
2. Tackle easy questions first
3. Skip hard ones, come back later
4. Use kubectl --help frequently
5. Copy-paste from documentation
```

### 6.2 Essential kubectl Shortcuts

```bash
# Imperative Commands (เร็วกว่า YAML)
kubectl run nginx --image=nginx --restart=Never
kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > pod.yaml

kubectl create deployment nginx --image=nginx --replicas=3
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

kubectl create service clusterip my-svc --tcp=80:80
kubectl expose deployment nginx --port=80 --target-port=80

# Force Replace (อันตราย)
kubectl replace --force -f pod.yaml

# Useful Shortcuts
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--grace-period=0 --force"

# ใช้งาน:
k run nginx --image=nginx $do > pod.yaml
k delete pod nginx $now
```

### 6.3 Context Switching

```bash
# ตรวจสอบ Context ปัจจุบัน
kubectl config current-context

# Switch Context
kubectl config use-context <context-name>

# ดู Contexts ทั้งหมด
kubectl config get-contexts

# ตั้งค่า Namespace
kubectl config set-context --current --namespace=<namespace>

# IMPORTANT: ในข้อสอบต้องตรวจสอบ Context ก่อนทุกข้อ!
```

### 6.4 Documentation Navigation

```
Key Pages:
- https://kubernetes.io/docs/concepts/ (Architecture)
- https://kubernetes.io/docs/tasks/ (How-to)
- https://kubernetes.io/docs/reference/ (API Reference)

Most Used Pages:
- kubectl cheat sheet
- kubeadm init options
- Pod lifecycle
- RBAC
- Network Policies
- PersistentVolumes
- Ingress
```

---

## 7. Practice Labs (300+ Questions)

### Domain 1: Cluster Architecture (75 Questions)

#### Basic Questions

**Q1: สร้าง Pod ชื่อ nginx ด้วย image nginx:1.25**
```bash
kubectl run nginx --image=nginx:1.25
```

**Q2: ตรวจสอบ Pod nginx ว่า Running หรือไม่**
```bash
kubectl get pod nginx
kubectl describe pod nginx
```

**Q3: ดู Logs ของ nginx pod**
```bash
kubectl logs nginx
kubectl logs nginx -f  # follow
kubectl logs nginx --tail=100  # last 100 lines
```

**Q4: Execute shell ใน nginx pod**
```bash
kubectl exec -it nginx -- /bin/sh
kubectl exec -it nginx -- /bin/bash
```

**Q5: ลบ Pod nginx**
```bash
kubectl delete pod nginx
kubectl delete pod nginx --force --grace-period=0
```

**Q6: สร้าง namespace ชื่อ dev**
```bash
kubectl create namespace dev
```

**Q7: List Pods ใน namespace dev**
```bash
kubectl get pods -n dev
```

**Q8: สร้าง Pod ใน namespace dev**
```bash
kubectl run nginx -n dev --image=nginx
```

**Q9: ดู Nodes ทั้งหมด**
```bash
kubectl get nodes
kubectl get nodes -o wide
```

**Q10: Drain Node worker-1**
```bash
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
```

**Q11: Cordon Node worker-1 (ห้าม Schedule Pods)**
```bash
kubectl cordon worker-1
```

**Q12: Uncordon Node worker-1**
```bash
kubectl uncordon worker-1
```

**Q13: Upgrade kubelet บน worker node ไปเป็น v1.28.0**
```bash
# บน worker node:
apt-get update
apt-get install -y kubelet=1.28.0-*
systemctl daemon-reload
systemctl restart kubelet
```

**Q14: Backup etcd ไปที่ /opt/etcd-backup.db**
```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Q15: สร้าง User john ที่มีสิทธิ์ get, list pods ใน namespace default**
```bash
# สร้าง Role
kubectl create role pod-reader \
  --verb=get,list \
  --resource=pods \
  -n default

# สร้าง RoleBinding
kubectl create rolebinding john-pod-reader \
  --role=pod-reader \
  --user=john \
  -n default
```

**Q16: ตรวจสอบว่า john มีสิทธิ์ delete pods หรือไม่**
```bash
kubectl auth can-i delete pods --as john -n default
```

**Q17: สร้าง ServiceAccount ชื่อ my-sa ใน namespace production**
```bash
kubectl create serviceaccount my-sa -n production
```

**Q18: สร้าง ClusterRole ที่สามารถ get, list, watch nodes ได้**
```bash
kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes
```

**Q19: ตั้งค่า kubeconfig context ให้ default namespace เป็น production**
```bash
kubectl config set-context --current --namespace=production
```

**Q20: ดู Certificate Expiration**
```bash
kubeadm certs check-expiration
```

#### Advanced Questions

**Q21: สร้าง kubeconfig file สำหรับ User john**
```bash
# สร้าง Private Key
openssl genrsa -out john.key 2048

# สร้าง CSR
openssl req -new -key john.key \
  -subj "/CN=john/O=developers" \
  -out john.csr

# Submit CSR ไปยัง Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  request: $(cat john.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# Approve
kubectl certificate approve john

# รับ Certificate
kubectl get csr john -o jsonpath='{.status.certificate}' | base64 -d > john.crt

# สร้าง kubeconfig
CLUSTER_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
CLUSTER_CA=$(kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key

kubectl config set-context john@kubernetes \
  --cluster=kubernetes \
  --user=john
```

**Q22: Restore etcd จาก Backup**
```bash
# หยุด API Server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 10

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir /var/lib/etcd-restored

# ย้ายข้อมูล
mv /var/lib/etcd /var/lib/etcd-old
mv /var/lib/etcd-restored /var/lib/etcd

# เริ่ม API Server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

**Q23: Enable Audit Logging**
```bash
# สร้าง Audit Policy
cat > /etc/kubernetes/audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
- level: Request
  resources:
  - group: ""
    resources: ["pods"]
- level: None
  resources:
  - group: ""
    resources: ["events"]
EOF

# เพิ่มใน kube-apiserver.yaml
# --audit-log-path=/var/log/kubernetes/audit.log
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

### Domain 2: Workloads (45 Questions)

**Q24: สร้าง Deployment ชื่อ webapp ด้วย image nginx:1.25 จำนวน 3 replicas**
```bash
kubectl create deployment webapp \
  --image=nginx:1.25 \
  --replicas=3
```

**Q25: Scale Deployment webapp เป็น 5 replicas**
```bash
kubectl scale deployment webapp --replicas=5
```

**Q26: Update Image ของ Deployment webapp เป็น nginx:1.26**
```bash
kubectl set image deployment/webapp nginx=nginx:1.26
```

**Q27: Rollback Deployment webapp**
```bash
kubectl rollout undo deployment/webapp
```

**Q28: Rollback ไปยัง Revision 1**
```bash
kubectl rollout history deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=1
```

**Q29: สร้าง HPA สำหรับ webapp ที่ Scale 2-10 replicas เมื่อ CPU > 50%**
```bash
kubectl autoscale deployment webapp \
  --cpu-percent=50 \
  --min=2 \
  --max=10
```

**Q30: สร้าง CronJob ที่รัน echo "hello" ทุกชั่วโมง**
```bash
kubectl create cronjob hello \
  --image=busybox \
  --schedule="0 * * * *" \
  -- /bin/sh -c "echo hello"
```

**Q31: สร้าง Job ที่รัน pi calculation**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

**Q32: Schedule Pod บน Node ที่มี Label disktype=ssd**
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**Q33: Schedule Pod ไม่ให้อยู่บน Node เดียวกัน**
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
```

**Q34: Taint Node worker-1 และสร้าง Pod ที่ Tolerate Taint นั้น**
```bash
# Taint Node
kubectl taint nodes worker-1 env=prod:NoSchedule

# Pod with Toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerate-pod
spec:
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
EOF
```

**Q35: สร้าง Pod ที่มี InitContainer ที่รัน ก่อน Main Container**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: myapp:latest
```

**Q36: สร้าง Pod ที่มี Sidecar Container สำหรับ Logging**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-agent
    image: busybox
    command: ['sh', '-c', 'tail -f /logs/access.log']
    volumeMounts:
    - name: logs
      mountPath: /logs
  volumes:
  - name: logs
    emptyDir: {}
```

**Q37: สร้าง DaemonSet ที่รันบนทุก Node รวม Control Plane**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: agent
        image: monitoring:latest
```

### Domain 3: Services & Networking (60 Questions)

**Q38: สร้าง Service ClusterIP สำหรับ nginx deployment**
```bash
kubectl expose deployment nginx --port=80 --target-port=80
```

**Q39: สร้าง Service NodePort บน Port 30080**
```bash
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  --type=NodePort \
  --name=nginx-nodeport

# แก้ nodePort เป็น 30080
kubectl patch svc nginx-nodeport -p \
  '{"spec":{"ports":[{"port":80,"nodePort":30080}]}}'
```

**Q40: สร้าง NetworkPolicy ที่ block traffic ทั้งหมดไป Pod ชื่อ secure**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-all
spec:
  podSelector:
    matchLabels:
      app: secure
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
```

**Q41: สร้าง NetworkPolicy ที่อนุญาต traffic จาก frontend เท่านั้น**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Q42: Debug Service ที่ไม่สามารถเชื่อมต่อได้**
```bash
# ตรวจสอบ Service
kubectl get svc
kubectl describe svc <service-name>

# ตรวจสอบ Endpoints
kubectl get endpoints <service-name>

# ถ้า Endpoints ว่าง = Selector ไม่ match
kubectl get pods --show-labels
kubectl describe svc | grep Selector

# ทดสอบ Connectivity
kubectl run test --image=busybox --rm -it -- wget -O- http://<service>:80
```

**Q43: สร้าง Ingress สำหรับ 2 Services ด้วย path-based routing**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

**Q44: DNS Resolution - ค้นหา IP ของ Service จากใน Pod**
```bash
# Service DNS format: <service>.<namespace>.svc.cluster.local
kubectl run debug --image=busybox --rm -it -- nslookup kubernetes.default.svc.cluster.local
kubectl run debug --image=busybox --rm -it -- nslookup my-svc.my-ns.svc.cluster.local
```

### Domain 4: Storage (30 Questions)

**Q45: สร้าง PV ด้วย hostPath ขนาด 1Gi**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-host
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
  persistentVolumeReclaimPolicy: Retain
```

**Q46: สร้าง PVC ขนาด 500Mi**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**Q47: สร้าง Pod ที่ Mount PVC**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pvc
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

**Q48: ดู StorageClass ทั้งหมด**
```bash
kubectl get storageclass
kubectl get sc  # short form
```

**Q49: ดู PVs และ PVCs**
```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pvc my-pvc
```

**Q50: Expand PVC จาก 500Mi เป็น 2Gi**
```bash
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```

### Domain 5: Troubleshooting (90 Questions)

**Q51: Pod อยู่ในสถานะ CrashLoopBackOff - วินิจฉัยและแก้ไข**
```bash
# ดู Logs
kubectl logs <pod-name> --previous

# ดู Events
kubectl describe pod <pod-name>

# ดู Exit Code
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[0].lastState'

# แก้ไขตาม Error:
# - Config Error: แก้ ConfigMap/Secret
# - OOMKilled: เพิ่ม Memory Limit
# - Missing Dependency: ตรวจสอบ Environment Variables
```

**Q52: ทำไม Pod ถึง Pending?**
```bash
kubectl describe pod <pod-name> | grep -A 10 "Events"

# Common Causes:
# 1. Insufficient CPU/Memory
kubectl describe nodes | grep -A 10 "Allocated resources"

# 2. NodeSelector ไม่ match
kubectl get pod <pod> -o yaml | grep nodeSelector
kubectl get nodes --show-labels

# 3. Taints ไม่มี Tolerations
kubectl describe nodes | grep Taints

# 4. PVC ไม่ Bound
kubectl get pvc
```

**Q53: ทำไม Service ไม่ทำงาน?**
```bash
# 1. ดู Endpoints
kubectl get endpoints <service>

# 2. ตรวจสอบ Selector
kubectl describe svc <service>
kubectl get pods --show-labels

# 3. ทดสอบ Connection
kubectl run test --image=busybox --rm -it -- wget -O- http://<service>

# 4. ตรวจสอบ kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-xxxx
```

**Q54: Worker Node ไม่ Ready - แก้ไข**
```bash
# SSH เข้า Node
# ตรวจสอบ kubelet
systemctl status kubelet
journalctl -u kubelet -n 50

# Common Issues:
# 1. kubelet หยุดทำงาน
systemctl restart kubelet

# 2. Disk เต็ม
df -h
du -sh /var/lib/docker/*

# 3. Network ปัญหา
ping <api-server-ip>

# 4. Certificate หมดอายุ
openssl x509 -noout -in /var/lib/kubelet/pki/kubelet-client-current.pem -enddate
```

**Q55: API Server ไม่ตอบสนอง**
```bash
# ตรวจสอบ Static Pods
ls /etc/kubernetes/manifests/

# ดู API Server Process
ps aux | grep kube-apiserver

# ตรวจสอบ etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# ดู Logs
kubectl logs -n kube-system kube-apiserver-master
```

**Q56: แก้ RBAC ที่ทำให้ User john ไม่สามารถ get pods ได้**
```bash
# ตรวจสอบ Permission
kubectl auth can-i get pods --as john

# ดู RoleBindings
kubectl get rolebindings -A | grep john
kubectl get clusterrolebindings | grep john

# ตรวจสอบ Role
kubectl describe role <role-name>
kubectl describe rolebinding <binding-name>

# สร้าง RoleBinding ที่ถูกต้อง
kubectl create rolebinding john-pod-reader \
  --role=pod-reader \
  --user=john \
  -n default
```

**Q57-Q100: Additional Practice Questions**

```bash
# Q57: รัน Pod ชั่วคราวเพื่อ Debug
kubectl run debug --image=busybox --rm -it -- sh

# Q58: ดู Events ทั้งหมดที่เพิ่งเกิดขึ้น
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Q59: ค้นหา Pod ที่กิน CPU มากที่สุด
kubectl top pods -A --sort-by=cpu | head -5

# Q60: ดู Resource Requests/Limits ของทุก Pod
kubectl get pods -A -o custom-columns='NAME:.metadata.name,CPU-REQ:.spec.containers[0].resources.requests.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory'

# Q61: สร้าง ConfigMap จาก File
echo "key=value" > config.env
kubectl create configmap my-config --from-env-file=config.env

# Q62: สร้าง Secret
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Q63: Mount Secret เป็น Volume
# (ดู yaml ด้านล่าง)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secret
  volumes:
  - name: secret-vol
    secret:
      secretName: my-secret
EOF

# Q64: ดู Secret ที่ Decode แล้ว
kubectl get secret my-secret -o json | jq -r '.data | map_values(@base64d)'

# Q65: ตรวจสอบ etcd Member List
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Q66: สร้าง Pod Priority Class
cat <<'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority class"
EOF

# Q67: ดู API Resources ที่มี
kubectl api-resources

# Q68: ดู API Versions
kubectl api-versions

# Q69: ตรวจสอบ Cluster Info
kubectl cluster-info
kubectl cluster-info dump

# Q70: สร้าง Namespace ด้วย Label
kubectl create namespace staging
kubectl label namespace staging environment=staging

# Q71: ดู Pods ด้วย Custom Columns
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP,NODE:.spec.nodeName'

# Q72: สร้าง Pod ด้วย Multiple Containers
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  - name: redis
    image: redis
    ports:
    - containerPort: 6379
EOF

# Q73: Port-Forward ไปยัง Pod
kubectl port-forward pod/nginx 8080:80

# Q74: Copy File ไปยัง Pod
echo "Hello World" > test.txt
kubectl cp test.txt nginx:/tmp/test.txt

# Q75: Copy File จาก Pod
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
```

---

## สรุป CKA Exam

```
Key Points:
1. ฝึก imperative commands มากๆ
2. จำ kubectl alias: alias k=kubectl
3. ตรวจสอบ Context ก่อนทุกข้อ
4. ใช้ --dry-run=client -o yaml เพื่อสร้าง YAML Template
5. อ่าน docs เป็น แต่ไม่ต้องอ่านทั้งหมด
6. Time management: ทำง่ายก่อน ยากทีหลัง
7. ฝึกบน KillerCoda, Play with Kubernetes หรือ Local Cluster
```

## References

- [CKA Official](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [KillerCoda Practice](https://killercoda.com/cka)
- [CKA Study Guide GitHub](https://github.com/dgkanatsios/CKAD-exercises)
