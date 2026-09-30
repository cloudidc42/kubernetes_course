# Part 47: NFS Storage - Shared Storage สำหรับ Kubernetes

## บทนำ

**NFS (Network File System)** เป็นโปรโตคอลสำหรับแชร์ filesystem ผ่านเครือข่าย ใน Kubernetes NFS ใช้เป็น shared storage ที่หลาย Pods สามารถ read/write พร้อมกันได้ (ReadWriteMany) ซึ่งเป็นสิ่งที่ local disk ไม่สามารถทำได้

### Use Cases สำหรับ NFS:
- Shared file storage (รูปภาพ, วิดีโอ, เอกสาร)
- Log aggregation
- Shared configuration
- Multi-pod write access

---

## 1. NFS คืออะไร

### NFS Architecture

```
  Kubernetes Cluster
  ┌──────────────────────────────────────┐
  │  Pod A          Pod B          Pod C  │
  │   |               |               |  │
  │   └───────────────┴───────────────┘  │
  │               NFS Mount              │
  │                   |                  │
  └───────────────────|──────────────────┘
                      |
              ┌───────────────┐
              │   NFS Server  │
              │  192.168.1.10 │
              │  /exports/k8s │
              └───────────────┘
```

### NFS Versions

| Version | คุณสมบัติ |
|---------|----------|
| NFSv3 | IPv4 only, stateless, UDP/TCP |
| NFSv4 | IPv4/IPv6, stateful, TCP only, ACLs, Kerberos |
| NFSv4.1 | Parallel NFS (pNFS), session-based |
| NFSv4.2 | Server-side copy, sparse files |

---

## 2. ติดตั้ง NFS Server

### 2.1 ติดตั้งบน Ubuntu/Debian

```bash
# ติดตั้ง NFS Server
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# สร้าง directories สำหรับ export
sudo mkdir -p /exports/kubernetes/storage1
sudo mkdir -p /exports/kubernetes/storage2
sudo mkdir -p /exports/kubernetes/shared

# กำหนด permissions
sudo chown nobody:nogroup /exports/kubernetes
sudo chmod 777 /exports/kubernetes

# กำหนด NFS exports
sudo vim /etc/exports
```

**เนื้อหาใน /etc/exports:**
```
# /etc/exports - NFS Export Configuration
# Format: path  clients(options)

# Kubernetes storage - allow all cluster nodes
/exports/kubernetes/storage1  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
/exports/kubernetes/storage2  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
/exports/kubernetes/shared    192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

# Read-only share
/exports/readonly             192.168.1.0/24(ro,sync,no_subtree_check)

# Allow specific node
/exports/node-specific        192.168.1.101(rw,sync,no_subtree_check,no_root_squash)
```

**NFS Export Options:**

| Option | คำอธิบาย |
|--------|---------|
| `rw` | Read-write access |
| `ro` | Read-only access |
| `sync` | Write sync (safe, slower) |
| `async` | Write async (faster, risk) |
| `no_subtree_check` | Disable subtree checking (ลด errors) |
| `no_root_squash` | Root บน client = Root บน server |
| `root_squash` | Root บน client = nobody บน server (ปลอดภัยกว่า) |
| `all_squash` | ทุก user = nobody |
| `anonuid=65534` | UID สำหรับ anonymous |
| `anongid=65534` | GID สำหรับ anonymous |

```bash
# Apply exports
sudo exportfs -ra

# ตรวจสอบ exports
sudo exportfs -v

# Start/Enable NFS server
sudo systemctl start nfs-kernel-server
sudo systemctl enable nfs-kernel-server
sudo systemctl status nfs-kernel-server

# ตรวจสอบ NFS Server
showmount -e localhost
showmount -e 192.168.1.10  # จาก client
```

### 2.2 ติดตั้งบน RHEL/CentOS

```bash
# ติดตั้ง NFS
sudo yum install -y nfs-utils

# หรือสำหรับ RHEL 8+
sudo dnf install -y nfs-utils

# สร้าง export directories
sudo mkdir -p /exports/kubernetes/{storage1,storage2,shared}
sudo chmod 777 /exports/kubernetes
sudo chown nfsnobody:nfsnobody /exports/kubernetes

# Configure exports
sudo cat > /etc/exports << 'EOF'
/exports/kubernetes/storage1  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
/exports/kubernetes/shared    192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

# Start services
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo exportfs -ra

# Firewall rules
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=nfs3
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
```

### 2.3 ตั้งค่า NFS ใน Kubernetes (สำหรับ Minikube Workshop)

```bash
# ถ้าใช้ Minikube สามารถ deploy NFS Server ใน cluster ได้

# Deploy NFS Server ใน Kubernetes
kubectl apply -f - << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: k8s.gcr.io/volume-nfs:0.8
        ports:
        - containerPort: 2049
          name: nfs
        - containerPort: 20048
          name: mountd
        - containerPort: 111
          name: rpcbind
        securityContext:
          privileged: true
        volumeMounts:
        - name: storage
          mountPath: /exports
      volumes:
      - name: storage
        hostPath:
          path: /mnt/nfs-data
          type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: nfs-server
spec:
  selector:
    app: nfs-server
  ports:
  - port: 2049
    name: nfs
  - port: 20048
    name: mountd
  - port: 111
    name: rpcbind
EOF

# ดู NFS Server IP
NFS_IP=$(kubectl get svc nfs-server -n nfs-server -o jsonpath='{.spec.clusterIP}')
echo "NFS Server IP: $NFS_IP"
```

---

## 3. NFS Client ใน Kubernetes

### 3.1 ติดตั้ง NFS Client บน Nodes

```bash
# Ubuntu/Debian
sudo apt-get install -y nfs-common

# RHEL/CentOS
sudo yum install -y nfs-utils

# ทดสอบ mount
sudo mount -t nfs 192.168.1.10:/exports/kubernetes/storage1 /mnt/test
ls /mnt/test
sudo umount /mnt/test
```

### 3.2 NFS Volume ใน Pod (ไม่ใช้ PV/PVC)

```yaml
# pod-with-nfs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-direct-pod
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "NFS Pod started"
      echo "$(date): Hello from NFS" >> /nfs/shared.log
      ls -la /nfs/
      sleep 3600
    volumeMounts:
    - name: nfs-storage
      mountPath: /nfs
  
  volumes:
  - name: nfs-storage
    nfs:
      server: 192.168.1.10    # NFS Server IP
      path: /exports/kubernetes/storage1
      readOnly: false
```

---

## 4. NFS Provisioner สำหรับ Kubernetes

### 4.1 NFS Subdir External Provisioner

**ติดตั้งด้วย Helm:**

```bash
# เพิ่ม Helm repo
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

# ติดตั้ง Provisioner
helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --create-namespace \
  --set nfs.server=192.168.1.10 \
  --set nfs.path=/exports/kubernetes \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false \
  --set storageClass.reclaimPolicy=Retain \
  --set storageClass.archiveOnDelete=true \
  --set storageClass.pathPattern='${.PVC.namespace}-${.PVC.name}'

# ตรวจสอบ
kubectl get pods -n nfs-provisioner
kubectl get storageclass nfs-client
```

**ติดตั้งด้วย YAML:**

```yaml
# nfs-provisioner-setup.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-provisioner
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
- kind: ServiceAccount
  name: nfs-client-provisioner
  namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
subjects:
- kind: ServiceAccount
  name: nfs-client-provisioner
  namespace: nfs-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
      - name: nfs-client-provisioner
        image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
        volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
        env:
        - name: PROVISIONER_NAME
          value: k8s-sigs.io/nfs-subdir-external-provisioner
        - name: NFS_SERVER
          value: "192.168.1.10"    # เปลี่ยนเป็น NFS Server IP
        - name: NFS_PATH
          value: "/exports/kubernetes"
      volumes:
      - name: nfs-client-root
        nfs:
          server: "192.168.1.10"   # เปลี่ยนเป็น NFS Server IP
          path: "/exports/kubernetes"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  server: "192.168.1.10"           # เปลี่ยนเป็น NFS Server IP
  path: "/exports/kubernetes"
  readOnly: "false"
  archiveOnDelete: "true"          # Archive แทนลบ
  pathPattern: "${.PVC.namespace}-${.PVC.name}"
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

```bash
kubectl apply -f nfs-provisioner-setup.yaml
kubectl get pods -n nfs-provisioner
kubectl get storageclass nfs-client
```

### 4.2 NFS CSI Driver

```bash
# ติดตั้ง NFS CSI Driver
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version v4.6.0

# ตรวจสอบ
kubectl get pods -n kube-system -l app=csi-nfs-controller
kubectl get pods -n kube-system -l app=csi-nfs-node
```

**StorageClass สำหรับ NFS CSI:**

```yaml
# nfs-csi-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.10
  share: /exports/kubernetes
  subDir: ""            # ว่างแปลว่าใช้ root
  mountPermissions: "0777"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- hard
- nfsvers=4.1
```

---

## 5. Workshop: Shared Storage ด้วย NFS

### Workshop Overview
Deploy web application ที่ใช้ NFS shared storage สำหรับแชร์ static files ระหว่าง Pods

### Prerequisites
- Kubernetes cluster (Minikube หรือ Kind)
- เวลาประมาณ 90-120 นาที

### Step 1: Setup NFS Server ใน Kubernetes (สำหรับ Lab)

```yaml
# nfs-server-setup.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-server
---
# NFS Server Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: itsthenetwork/nfs-server-alpine:12
        env:
        - name: SHARED_DIRECTORY
          value: "/nfsshare"
        volumeMounts:
        - name: nfs-storage
          mountPath: /nfsshare
        ports:
        - containerPort: 2049
          name: nfs
        securityContext:
          privileged: true
          capabilities:
            add:
            - SYS_ADMIN
      volumes:
      - name: nfs-storage
        hostPath:
          path: /tmp/nfs-data
          type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: nfs-server
spec:
  selector:
    app: nfs-server
  ports:
  - port: 2049
    name: nfs
    protocol: TCP
  clusterIP: None
```

```bash
kubectl apply -f nfs-server-setup.yaml

# รอ NFS Server start
kubectl wait --for=condition=available deployment/nfs-server -n nfs-server --timeout=120s
kubectl get pods -n nfs-server

# ดู NFS Server IP
NFS_SERVER_IP=$(kubectl get pods -n nfs-server -l app=nfs-server -o jsonpath='{.items[0].status.podIP}')
echo "NFS Server IP: $NFS_SERVER_IP"
```

### Step 2: สร้าง Namespace สำหรับ Workshop

```bash
kubectl create namespace nfs-workshop
kubectl config set-context --current --namespace=nfs-workshop
```

### Step 3: สร้าง PV สำหรับ NFS

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-shared-pv
  labels:
    storage: nfs
    app: web-shared
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany        # NFS รองรับ RWX
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  mountOptions:
  - hard
  nfs:
    server: <NFS_SERVER_IP>    # แทนด้วย IP จาก step 1
    path: /                     # Root ของ NFS share
    readOnly: false
```

```bash
# แทน NFS_SERVER_IP
sed -i "s/<NFS_SERVER_IP>/$NFS_SERVER_IP/" nfs-pv.yaml
kubectl apply -f nfs-pv.yaml
kubectl get pv nfs-shared-pv
```

### Step 4: สร้าง StorageClass สำหรับ NFS

```yaml
# nfs-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: kubernetes.io/no-provisioner  # Manual PV
volumeBindingMode: Immediate
reclaimPolicy: Retain
```

```bash
kubectl apply -f nfs-sc.yaml
```

### Step 5: สร้าง PVCs

```yaml
# nfs-pvcs.yaml
---
# PVC สำหรับ web static files (shared)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-static-pvc
  namespace: nfs-workshop
spec:
  accessModes:
  - ReadWriteMany    # หลาย Pods เขียนได้
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
  selector:
    matchLabels:
      storage: nfs
---
# PVC สำหรับ uploads (shared)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-uploads-pvc
  namespace: nfs-workshop
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
  storageClassName: nfs
```

```bash
kubectl apply -f nfs-pvcs.yaml
kubectl get pvc -n nfs-workshop
```

### Step 6: สร้าง Content Initializer

```yaml
# content-init-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: content-initializer
  namespace: nfs-workshop
spec:
  template:
    spec:
      containers:
      - name: init
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Initializing web content on NFS..."
          mkdir -p /static/{css,js,images}
          
          # สร้าง CSS file
          cat > /static/css/main.css << 'EOF'
          body {
              font-family: Arial, sans-serif;
              max-width: 800px;
              margin: 0 auto;
              padding: 20px;
              background-color: #f5f5f5;
          }
          .container {
              background: white;
              padding: 30px;
              border-radius: 10px;
              box-shadow: 0 2px 5px rgba(0,0,0,0.1);
          }
          .pod-info {
              background: #e8f4f8;
              padding: 15px;
              border-left: 4px solid #2196F3;
              margin: 10px 0;
          }
          EOF
          
          # สร้าง JavaScript file
          cat > /static/js/app.js << 'EOF'
          function updateTime() {
              document.getElementById('time').textContent = new Date().toLocaleString();
          }
          setInterval(updateTime, 1000);
          window.onload = updateTime;
          EOF
          
          # สร้าง index.html
          cat > /static/index.html << 'EOF'
          <!DOCTYPE html>
          <html>
          <head>
              <title>NFS Shared Storage Demo</title>
              <link rel="stylesheet" href="/css/main.css">
          </head>
          <body>
              <div class="container">
                  <h1>NFS Shared Storage Demo</h1>
                  <div class="pod-info">
                      <strong>Serving Pod:</strong> ${HOSTNAME}
                  </div>
                  <p>Current time: <span id="time"></span></p>
                  <p>This content is served from NFS shared storage!</p>
              </div>
              <script src="/js/app.js"></script>
          </body>
          </html>
          EOF
          
          echo "Content initialized!"
          ls -la /static/
        volumeMounts:
        - name: static-files
          mountPath: /static
      
      volumes:
      - name: static-files
        persistentVolumeClaim:
          claimName: web-static-pvc
      
      restartPolicy: OnFailure
```

```bash
kubectl apply -f content-init-job.yaml
kubectl logs job/content-initializer -n nfs-workshop
```

### Step 7: Deploy Web Application ที่ใช้ NFS

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: nfs-workshop
  labels:
    app: web-app
spec:
  replicas: 3    # 3 Pods ใช้ NFS ร่วมกัน
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        
        volumeMounts:
        - name: static-content
          mountPath: /usr/share/nginx/html
          readOnly: true         # Web server อ่านอย่างเดียว
        - name: upload-storage
          mountPath: /var/nginx/uploads
        - name: nginx-logs
          mountPath: /var/log/nginx
        
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      
      - name: log-watcher
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Log watcher started on $HOSTNAME"
          while true; do
            if [ -f /logs/access.log ]; then
              echo "$(date): Latest access on $HOSTNAME:"
              tail -3 /logs/access.log
            fi
            sleep 15
          done
        volumeMounts:
        - name: nginx-logs
          mountPath: /logs
          readOnly: true
      
      volumes:
      - name: static-content
        persistentVolumeClaim:
          claimName: web-static-pvc
      - name: upload-storage
        persistentVolumeClaim:
          claimName: web-uploads-pvc
      - name: nginx-logs
        emptyDir: {}
```

```bash
kubectl apply -f web-deployment.yaml
kubectl get pods -n nfs-workshop -w
```

### Step 8: สร้าง Services

```yaml
# web-services.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: nfs-workshop
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
  namespace: nfs-workshop
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

```bash
kubectl apply -f web-services.yaml
```

### Step 9: ทดสอบ NFS Shared Storage

```bash
# Port forward
kubectl port-forward service/web-app 8080:80 -n nfs-workshop &

# Access web app
curl http://localhost:8080

# ดูว่า requests ถูก load balance ไปยัง Pods ต่างๆ
for i in $(seq 1 9); do
  curl -s http://localhost:8080 | grep "Serving Pod"
done
```

### Step 10: ทดสอบ Shared Write

```bash
# เขียนไฟล์ใหม่ผ่าน Pod หนึ่ง
POD1=$(kubectl get pods -n nfs-workshop -l app=web-app -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -n nfs-workshop -l app=web-app -o jsonpath='{.items[1].metadata.name}')
POD3=$(kubectl get pods -n nfs-workshop -l app=web-app -o jsonpath='{.items[2].metadata.name}')

echo "Pod1: $POD1"
echo "Pod2: $POD2"
echo "Pod3: $POD3"

# เขียนไฟล์จาก Pod 1
kubectl exec -n nfs-workshop $POD1 -- \
  sh -c 'echo "Created by $HOSTNAME at $(date)" > /usr/share/nginx/html/shared-test.html'

# อ่านจาก Pod 2 (ควรเห็นไฟล์ที่สร้างจาก Pod 1)
kubectl exec -n nfs-workshop $POD2 -- \
  cat /usr/share/nginx/html/shared-test.html

# อ่านจาก Pod 3
kubectl exec -n nfs-workshop $POD3 -- \
  cat /usr/share/nginx/html/shared-test.html

# ทดสอบ HTTP request
curl http://localhost:8080/shared-test.html
```

### Step 11: ทดสอบ Data Persistence

```bash
# เขียนข้อมูลสำคัญ
kubectl exec -n nfs-workshop $POD1 -- \
  sh -c 'echo "IMPORTANT DATA: $(date)" > /usr/share/nginx/html/persistent.txt'

# ลบ Deployment ทั้งหมด
kubectl delete deployment web-app -n nfs-workshop

# รอ Pods ถูกลบ
kubectl get pods -n nfs-workshop -w

# Deploy ใหม่
kubectl apply -f web-deployment.yaml
kubectl get pods -n nfs-workshop -w

# ตรวจสอบว่าข้อมูลยังอยู่
NEW_POD=$(kubectl get pods -n nfs-workshop -l app=web-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n nfs-workshop $NEW_POD -- \
  cat /usr/share/nginx/html/persistent.txt
```

### Step 12: ทดสอบ Scale Up/Down

```bash
# Scale up
kubectl scale deployment web-app -n nfs-workshop --replicas=5

# รอ Pods start
kubectl get pods -n nfs-workshop -w

# ตรวจสอบ Pods ใหม่ก็เข้าถึง NFS ได้
NEW_PODS=$(kubectl get pods -n nfs-workshop -l app=web-app -o jsonpath='{.items[*].metadata.name}')
for pod in $NEW_PODS; do
  echo "=== $pod ==="
  kubectl exec -n nfs-workshop $pod -- ls /usr/share/nginx/html/
done

# Scale down
kubectl scale deployment web-app -n nfs-workshop --replicas=2
```

### Step 13: Monitor NFS Performance

```bash
# ทดสอบ write speed บน NFS
kubectl exec -n nfs-workshop $POD1 -- \
  dd if=/dev/urandom of=/usr/share/nginx/html/testfile bs=1M count=100 conv=fsync

# ทดสอบ read speed
kubectl exec -n nfs-workshop $POD1 -- \
  dd if=/usr/share/nginx/html/testfile of=/dev/null bs=1M

# ดู disk usage บน NFS
kubectl exec -n nfs-workshop $POD1 -- df -h /usr/share/nginx/html
```

### Step 14: ทำความสะอาด

```bash
kubectl delete namespace nfs-workshop
kubectl delete pv nfs-shared-pv
kubectl delete namespace nfs-server
```

---

## 6. NFS Best Practices

### 6.1 Performance Tuning

```yaml
# NFS mount options สำหรับ performance
mountOptions:
- hard
- nfsvers=4.2
- rsize=65536
- wsize=65536
- timeo=600
- retrans=2
- noatime          # ไม่ update access time (ลด I/O)
- nodiratime       # ไม่ update directory access time
```

### 6.2 Security

```bash
# /etc/exports ที่ปลอดภัยกว่า
/exports/kubernetes  192.168.1.0/24(rw,sync,no_subtree_check,root_squash,all_squash,anonuid=65534,anongid=65534)

# ใช้ Kerberos authentication (NFSv4)
/exports/secure  *(sec=krb5p,rw,no_subtree_check)
```

### 6.3 High Availability NFS

```bash
# ใช้ Pacemaker/Corosync สำหรับ HA NFS Server
# หรือใช้ cloud-based NFS เช่น:
# - AWS EFS (Elastic File System)
# - Azure Files
# - GCP Filestore
```

---

## 7. AWS EFS (Elastic File System) - NFS บน Cloud

```yaml
# efs-storageclass.yaml (EKS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap       # Access Points
  fileSystemId: fs-12345678      # EFS File System ID
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
reclaimPolicy: Delete
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
  - ReadWriteMany    # EFS รองรับ RWX
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

---

## 8. Troubleshooting NFS

### ปัญหาที่พบบ่อย

**1. Mount Timeout**
```bash
# ตรวจสอบ network connectivity
nc -zv 192.168.1.10 2049

# ตรวจสอบ NFS Server
showmount -e 192.168.1.10

# ดู NFS errors บน Node
journalctl -u nfs-client.target
dmesg | grep nfs
```

**2. Permission Denied**
```bash
# ตรวจสอบ NFS exports
cat /etc/exports

# ตรวจสอบ file permissions บน NFS server
ls -la /exports/kubernetes/

# ดู Pod securityContext
kubectl get pod <name> -o jsonpath='{.spec.securityContext}'
```

**3. Stale File Handle**
```bash
# Remount NFS
umount -l /mnt/nfs
mount -t nfs ...

# ใน Kubernetes: ลบ Pod เพื่อ remount
kubectl delete pod <pod-name>
```

**4. NFS Server ไม่ตอบสนอง**
```bash
# ตรวจสอบ Services บน NFS Server
systemctl status nfs-kernel-server nfs-mountd portmap rpcbind

# ตรวจสอบ RPC
rpcinfo -p 192.168.1.10
```

---

## สรุป

| Feature | คำอธิบาย |
|---------|----------|
| Access Mode | ReadWriteMany (RWX) - หลาย Pods เขียนพร้อมกัน |
| Performance | ขึ้นอยู่กับ network และ NFS Server |
| Use Case | Shared files, web content, uploads |
| HA | ต้องการ HA NFS Server หรือ Cloud NFS |
| Security | root_squash, Kerberos, Network segmentation |

### Key Takeaways:
1. **NFS** เหมาะสำหรับ shared storage ที่หลาย Pods ต้องการ access พร้อมกัน
2. **ReadWriteMany** access mode เป็นคุณสมบัติหลักของ NFS
3. **NFS Subdir External Provisioner** ช่วยทำ dynamic provisioning บน NFS
4. **mount options** สำคัญมากสำหรับ performance และ reliability
5. สำหรับ production ควรใช้ **Cloud NFS** (EFS, Azure Files, Filestore) เพื่อ HA
6. **Kerberos authentication** สำหรับ security ใน enterprise environments

## แหล่งเรียนรู้เพิ่มเติม

- [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs)
- [AWS EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
- [NFS Performance Tuning](https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfs-client.html)
