# Part 41: Kubernetes Volumes - การจัดการ Storage ใน Pods

## บทนำ

ใน Kubernetes โดยปกติแล้ว Container จะมี filesystem ที่เป็น ephemeral (ชั่วคราว) นั่นคือเมื่อ Container ถูก restart หรือ Pod ถูกลบ ข้อมูลทั้งหมดที่อยู่ใน Container นั้นจะหายไปด้วย Volume ใน Kubernetes คือกลไกที่ช่วยให้เราสามารถ persist ข้อมูลไว้ได้และแชร์ข้อมูลระหว่าง Containers ใน Pod เดียวกันได้

### ความแตกต่างระหว่าง Docker Volume และ Kubernetes Volume

**Docker Volume:**
- มีชีวิตอยู่เกิน Container lifecycle
- ถูกจัดการโดย Docker daemon

**Kubernetes Volume:**
- ผูกติดกับ Pod lifecycle ไม่ใช่ Container lifecycle
- เมื่อ Pod ถูกลบ Volume จะถูกลบตามด้วย (ขึ้นอยู่กับ Volume type)
- รองรับ Volume types หลากหลายมาก
- Containers หลายตัวใน Pod เดียวกันสามารถแชร์ Volume ได้

---

## 1. Volume Types ทั้งหมดใน Kubernetes

Kubernetes รองรับ Volume types จำนวนมาก แบ่งตามประเภทได้ดังนี้:

### 1.1 Ephemeral Volumes (ชั่วคราว)
- `emptyDir` - สร้างใหม่เมื่อ Pod เริ่มต้น ลบเมื่อ Pod ถูกลบ
- `configMap` - Mount ConfigMap เป็น Volume
- `secret` - Mount Secret เป็น Volume
- `downwardAPI` - Mount ข้อมูลเกี่ยวกับ Pod/Container เป็นไฟล์
- `projected` - รวมหลาย Volume sources เข้าด้วยกัน
- `ephemeral` - Inline PVC (Kubernetes 1.19+)

### 1.2 Local Node Volumes
- `hostPath` - Mount directory จาก Node host
- `local` - Local storage บน Node (ต้องการ PersistentVolume)

### 1.3 Network File System
- `nfs` - NFS mount
- `cephfs` - CephFS filesystem
- `glusterfs` - GlusterFS volume
- `iscsi` - iSCSI volume
- `fc` - Fibre Channel volume

### 1.4 Cloud Provider Volumes
- `awsElasticBlockStore` - AWS EBS (deprecated ใช้ CSI แทน)
- `gcePersistentDisk` - GCE Persistent Disk (deprecated)
- `azureDisk` - Azure Disk (deprecated)
- `azureFile` - Azure File
- `vsphereVolume` - vSphere VMDK

### 1.5 Special Volumes
- `persistentVolumeClaim` - อ้างอิง PVC
- `csi` - Container Storage Interface driver
- `portworxVolume` - Portworx storage
- `rbd` - Rados Block Device (Ceph)

---

## 2. emptyDir Volume

`emptyDir` คือ Volume ที่ถูกสร้างเมื่อ Pod ถูก assign ไปยัง Node และจะมีชีวิตอยู่ตลอดที่ Pod ยังทำงานอยู่บน Node นั้น Containers ทุกตัวใน Pod สามารถ read/write ข้อมูลใน emptyDir ได้ แต่ Volume จะถูกลบเมื่อ Pod ถูกลบ

### Use Cases สำหรับ emptyDir:
- Scratch space สำหรับการคำนวณที่ต้องการพื้นที่ชั่วคราว
- Checkpointing ในกระบวนการที่ยาวนาน
- แชร์ข้อมูลระหว่าง Containers ใน Pod (sidecar pattern)
- Cache ที่ไม่จำเป็นต้อง persist

### ตัวอย่าง emptyDir พื้นฐาน

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
  namespace: default
spec:
  containers:
  - name: writer
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "$(date): Hello from writer container" >> /data/shared/messages.txt
        sleep 5
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data/shared
  
  - name: reader
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "--- Reading messages ---"
        cat /data/shared/messages.txt 2>/dev/null || echo "No messages yet"
        sleep 10
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data/shared
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

### emptyDir ใช้ Memory (RAM)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-memory-demo
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: cache
      mountPath: /cache
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
  
  volumes:
  - name: cache
    emptyDir:
      medium: Memory      # ใช้ tmpfs (RAM)
      sizeLimit: 128Mi    # จำกัดขนาด
```

> **หมายเหตุ:** เมื่อใช้ `medium: Memory` ข้อมูลจะถูกเก็บใน RAM ซึ่งเร็วกว่า disk มาก แต่ก็ถูกนับรวมกับ memory usage ของ Container ด้วย

### emptyDir พร้อม sizeLimit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-sized-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sleep", "3600"]
    volumeMounts:
    - name: temp-storage
      mountPath: /tmp/data
  
  volumes:
  - name: temp-storage
    emptyDir:
      sizeLimit: 500Mi   # จำกัดขนาด Volume ไม่เกิน 500MB
```

---

## 3. hostPath Volume

`hostPath` Volume จะ mount directory หรือไฟล์จาก Node's filesystem เข้าไปใน Pod ซึ่งมีประโยชน์ในบางกรณี แต่ก็มีความเสี่ยงด้านความปลอดภัยสูง

### hostPath Types

| Type | คำอธิบาย |
|------|-----------|
| `""` (ว่าง) | ไม่มีการ check ก่อน mount |
| `DirectoryOrCreate` | สร้าง directory ถ้าไม่มี |
| `Directory` | Directory ต้องมีอยู่แล้ว |
| `FileOrCreate` | สร้างไฟล์ถ้าไม่มี |
| `File` | ไฟล์ต้องมีอยู่แล้ว |
| `Socket` | Unix socket ต้องมีอยู่แล้ว |
| `CharDevice` | Character device ต้องมีอยู่แล้ว |
| `BlockDevice` | Block device ต้องมีอยู่แล้ว |

### ตัวอย่าง hostPath พื้นฐาน

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      ls -la /host-logs/
      echo "Pod started at: $(date)" >> /host-logs/pod-events.log
      sleep 3600
    volumeMounts:
    - name: host-logs
      mountPath: /host-logs
  
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log/pods-custom   # path บน Node
      type: DirectoryOrCreate      # สร้าง directory ถ้าไม่มี
```

### hostPath สำหรับ Docker Socket (DinD pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: docker-in-pod
spec:
  containers:
  - name: docker-client
    image: docker:24-cli
    command: ["sleep", "3600"]
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
```

> **คำเตือน:** การใช้ Docker socket มีความเสี่ยงด้านความปลอดภัยสูงมาก ควรใช้เฉพาะใน development environment เท่านั้น

### hostPath สำหรับ Node Monitoring

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-monitor
spec:
  containers:
  - name: monitor
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "=== Disk Usage ==="
        df -h /host-root
        echo "=== Memory ==="
        cat /host-proc/meminfo | head -5
        sleep 30
      done
    volumeMounts:
    - name: host-root
      mountPath: /host-root
      readOnly: true
    - name: host-proc
      mountPath: /host-proc
      readOnly: true
  
  hostNetwork: true
  
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
  - name: host-proc
    hostPath:
      path: /proc
      type: Directory
```

---

## 4. configMap Volume

ConfigMap สามารถ mount เป็น Volume เพื่อให้ Container อ่านข้อมูล configuration ได้ในรูปแบบไฟล์

### สร้าง ConfigMap ก่อน

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  app.properties: |
    server.port=8080
    server.host=0.0.0.0
    database.url=jdbc:postgresql://postgres:5432/mydb
    database.pool.size=10
    cache.ttl=3600
  
  logging.yaml: |
    level: INFO
    handlers:
      - type: console
        format: json
      - type: file
        path: /var/log/app/app.log
        maxSize: 100MB
        maxBackups: 5
  
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 'healthy';
            add_header Content-Type text/plain;
        }
    }
```

### Mount ConfigMap เป็น Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== app.properties ==="
      cat /config/app.properties
      echo ""
      echo "=== logging.yaml ==="
      cat /config/logging.yaml
      echo ""
      echo "=== nginx.conf ==="
      cat /config/nginx.conf
      sleep 3600
    volumeMounts:
    - name: config-volume
      mountPath: /config
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### Mount ConfigMap บางไฟล์เท่านั้น

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-selective
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx.conf          # mount เฉพาะไฟล์นี้
  
  volumes:
  - name: nginx-config
    configMap:
      name: app-config
      items:
      - key: nginx.conf            # key ใน ConfigMap
        path: nginx.conf           # ชื่อไฟล์ที่ mount
```

### ConfigMap Volume พร้อม defaultMode

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-permissions
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sleep", "3600"]
    volumeMounts:
    - name: scripts
      mountPath: /scripts
  
  volumes:
  - name: scripts
    configMap:
      name: app-scripts
      defaultMode: 0755    # ทำให้ไฟล์ executable (rwxr-xr-x)
```

---

## 5. Secret Volume

Secret Volume ใช้สำหรับ mount ข้อมูลที่ sensitive เช่น passwords, API keys, TLS certificates

### สร้าง Secret ก่อน

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: default
type: Opaque
data:
  # base64 encoded values
  db-password: cGFzc3dvcmQxMjM=      # password123
  api-key: bXlfc2VjcmV0X2FwaV9rZXk=  # my_secret_api_key
  
stringData:
  # Plain text (Kubernetes จะ encode ให้อัตโนมัติ)
  jwt-secret: "my-super-secret-jwt-key-2024"
  config.json: |
    {
      "database": {
        "host": "postgres.default.svc.cluster.local",
        "port": 5432,
        "name": "production"
      },
      "redis": {
        "host": "redis.default.svc.cluster.local",
        "port": 6379
      }
    }
```

### Mount Secret เป็น Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== Secrets mounted ==="
      ls -la /secrets/
      echo ""
      echo "DB Password file exists: $(test -f /secrets/db-password && echo yes || echo no)"
      echo "Config JSON:"
      cat /secrets/config.json
      sleep 3600
    volumeMounts:
    - name: secret-vol
      mountPath: /secrets
      readOnly: true       # Best practice: mount secrets as read-only
  
  volumes:
  - name: secret-vol
    secret:
      secretName: app-secrets
      defaultMode: 0400    # Owner read-only (ปลอดภัยกว่า)
```

### TLS Certificate Secret

```yaml
# สร้าง TLS secret จาก certificate files
# kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64 encoded cert
  tls.key: LS0tLS1CRUdJTi... # base64 encoded key

---
apiVersion: v1
kind: Pod
metadata:
  name: tls-server
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 443
    volumeMounts:
    - name: tls
      mountPath: /etc/nginx/ssl
      readOnly: true
  
  volumes:
  - name: tls
    secret:
      secretName: tls-secret
```

---

## 6. downwardAPI Volume

ใช้สำหรับส่งข้อมูล metadata ของ Pod เข้าไปใน Container ผ่านทาง filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-volume-demo
  labels:
    app: myapp
    version: "1.0"
  annotations:
    build-date: "2024-01-15"
    git-commit: "abc123"
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== Pod Information ==="
      echo "Pod Name: $(cat /etc/podinfo/pod-name)"
      echo "Namespace: $(cat /etc/podinfo/namespace)"
      echo "Pod IP: $(cat /etc/podinfo/pod-ip)"
      echo "Node Name: $(cat /etc/podinfo/node-name)"
      echo ""
      echo "=== Labels ==="
      cat /etc/podinfo/labels
      echo ""
      echo "=== Annotations ==="
      cat /etc/podinfo/annotations
      sleep 3600
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 128Mi
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "pod-name"
        fieldRef:
          fieldPath: metadata.name
      - path: "namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "pod-ip"
        fieldRef:
          fieldPath: status.podIP
      - path: "node-name"
        fieldRef:
          fieldPath: spec.nodeName
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "cpu-request"
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
      - path: "memory-limit"
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

---

## 7. projected Volume

`projected` Volume ช่วยให้เรา mount หลาย Volume sources เข้าไปใน directory เดียว

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "=== Config from ConfigMap ==="
      cat /projected/config/app.properties
      echo ""
      echo "=== Secret credentials ==="
      cat /projected/secret/db-password
      echo ""
      echo "=== Pod Info ==="
      cat /projected/podinfo/pod-name
      sleep 3600
    volumeMounts:
    - name: all-in-one
      mountPath: /projected
  
  volumes:
  - name: all-in-one
    projected:
      sources:
      - configMap:
          name: app-config
          items:
          - key: app.properties
            path: config/app.properties
      - secret:
          name: app-secrets
          items:
          - key: db-password
            path: secret/db-password
      - downwardAPI:
          items:
          - path: "podinfo/pod-name"
            fieldRef:
              fieldPath: metadata.name
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: "https://kubernetes.default.svc"
```

---

## 8. Volume Lifecycle

### 8.1 การทำงานของ Volume Lifecycle

```
Pod Created
    |
    v
Volumes Attached to Node
    |
    v
Volumes Mounted into Pod
    |
    v
Containers Start with Volumes
    |
    v
(Pod Running)
    |
    v
Container Crash → Container Restarts
    |          (Volume ยังอยู่ ข้อมูลไม่หาย)
    v
Pod Deleted
    |
    v
Containers Stop
    |
    v
Volumes Unmounted
    |
    v
Volumes Detached from Node
    |
    v
emptyDir Volumes Deleted (hostPath ยังอยู่บน Node)
```

### 8.2 emptyDir Lifecycle

```
Pod Created → emptyDir Volume Created (ว่างเปล่า)
Container writes data to emptyDir
Container Crashes → Pod restarts Container
Data still exists in emptyDir ✓
Pod Deleted → emptyDir Volume Deleted ✗
```

### 8.3 hostPath Lifecycle

```
Pod Created → hostPath mounts existing Node directory
Container reads/writes data
Pod Deleted → hostPath data REMAINS on Node ✓
New Pod starts → data still accessible via hostPath ✓
```

### 8.4 การ verify Volume lifecycle ด้วย example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: writer
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Container started at: $(date)"
      echo "$(date): Start" >> /data/lifecycle.log
      # จำลองการ crash
      sleep 30
      echo "Crashing now!" >&2
      exit 1
    restartPolicy: OnFailure
    volumeMounts:
    - name: data
      mountPath: /data
  
  volumes:
  - name: data
    emptyDir: {}
  
  restartPolicy: OnFailure
```

---

## 9. Volume subPath

`subPath` ช่วยให้ Mount เฉพาะ subdirectory หรือไฟล์จาก Volume แทนที่จะ mount ทั้ง Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-demo
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password123"
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
      subPath: mysql          # Mount เฉพาะ /data/mysql
    - name: data
      mountPath: /var/log/mysql
      subPath: mysql-logs     # Mount เฉพาะ /data/mysql-logs
  
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /var/www/html
      subPath: nginx-html     # Mount เฉพาะ /data/nginx-html
  
  volumes:
  - name: data
    emptyDir: {}
```

### subPathExpr สำหรับ Dynamic subPath

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-expr-demo
spec:
  containers:
  - name: app
    image: busybox:1.35
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    command: ["sleep", "3600"]
    volumeMounts:
    - name: logs
      mountPath: /logs
      subPathExpr: $(POD_NAME)  # Dynamic path ตาม Pod name
  
  volumes:
  - name: logs
    hostPath:
      path: /var/log/app-pods
      type: DirectoryOrCreate
```

---

## 10. Volume ReadOnly Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-volume-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
      readOnly: true         # Read-only mount
    - name: tls-certs
      mountPath: /etc/nginx/ssl
      readOnly: true         # Read-only TLS certs
    - name: cache
      mountPath: /var/cache/nginx
      readOnly: false        # Writable cache
  
  volumes:
  - name: config
    configMap:
      name: nginx-config
  - name: tls-certs
    secret:
      secretName: nginx-tls
  - name: cache
    emptyDir: {}
```

---

## 11. Init Container กับ Volumes

Init Containers มักใช้ร่วมกับ Volumes เพื่อ prepare ข้อมูลก่อน main container เริ่ม

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-volume-demo
spec:
  initContainers:
  - name: data-loader
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Loading initial data..."
      mkdir -p /data/db
      cat > /data/config/app.json << 'EOF'
      {
        "initialized": true,
        "timestamp": "$(date -Iseconds)",
        "version": "1.0.0"
      }
      EOF
      echo "Initial data loaded successfully"
    volumeMounts:
    - name: data
      mountPath: /data/db
      subPath: db
    - name: config
      mountPath: /data/config
  
  - name: permission-fixer
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Fixing permissions..."
      chown -R 1000:1000 /data
      chmod -R 755 /data
      echo "Permissions fixed"
    volumeMounts:
    - name: data
      mountPath: /data
  
  containers:
  - name: app
    image: busybox:1.35
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
    volumeMounts:
    - name: data
      mountPath: /data
    - name: config
      mountPath: /config
  
  volumes:
  - name: data
    emptyDir: {}
  - name: config
    emptyDir: {}
```

---

## 12. Multi-Container Pod with Shared Volumes (Sidecar Pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-logging-demo
  labels:
    app: web-with-sidecar
spec:
  containers:
  # Main application container
  - name: web-app
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
    - name: html
      mountPath: /usr/share/nginx/html
  
  # Sidecar: Log shipper
  - name: log-shipper
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        if [ -f /logs/access.log ]; then
          echo "=== Shipping logs at $(date) ==="
          tail -n 10 /logs/access.log
          echo "--- Shipped ---"
        fi
        sleep 15
      done
    volumeMounts:
    - name: logs
      mountPath: /logs
      readOnly: true
  
  # Sidecar: Content updater
  - name: content-updater
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        cat > /html/index.html << EOF
        <!DOCTYPE html>
        <html>
        <body>
          <h1>Updated at: $(date)</h1>
          <p>Pod: $HOSTNAME</p>
        </body>
        </html>
        EOF
        sleep 30
      done
    volumeMounts:
    - name: html
      mountPath: /html
  
  volumes:
  - name: logs
    emptyDir: {}
  - name: html
    emptyDir: {}
```

---

## 13. Workshop: ใช้ Volumes ใน Pods

### Workshop Overview
ในบทนี้เราจะสร้าง web application ที่ใช้ Volumes หลายประเภทร่วมกัน

### Prerequisites
- Kubernetes cluster ที่ทำงานได้ (Minikube หรือ kind)
- kubectl configured
- เวลาประมาณ 30-45 นาที

### Step 1: สร้าง Namespace

```bash
kubectl create namespace volumes-workshop
kubectl config set-context --current --namespace=volumes-workshop
```

### Step 2: สร้าง ConfigMap สำหรับ Nginx

```yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: volumes-workshop
data:
  nginx.conf: |
    worker_processes 1;
    
    events {
        worker_connections 1024;
    }
    
    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent"';
        
        access_log /var/log/nginx/access.log main;
        error_log /var/log/nginx/error.log warn;
        
        server {
            listen 80;
            server_name localhost;
            
            root /usr/share/nginx/html;
            index index.html;
            
            location / {
                try_files $uri $uri/ =404;
            }
            
            location /status {
                stub_status on;
                allow 127.0.0.1;
                deny all;
            }
        }
    }
  
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Kubernetes Volumes Workshop</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            .info { background: #f0f0f0; padding: 20px; border-radius: 5px; }
        </style>
    </head>
    <body>
        <h1>Kubernetes Volumes Workshop</h1>
        <div class="info">
            <h2>This page is served from a ConfigMap Volume!</h2>
            <p>The nginx.conf and this index.html are both from ConfigMap.</p>
        </div>
    </body>
    </html>
```

```bash
kubectl apply -f nginx-config.yaml
```

### Step 3: สร้าง Secret

```yaml
# app-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: volumes-workshop
type: Opaque
stringData:
  admin-password: "SuperSecret2024!"
  api-token: "eyJhbGciOiJIUzI1NiJ9.workshop"
  db-config.json: |
    {
      "host": "localhost",
      "port": 5432,
      "database": "workshop_db",
      "username": "app_user",
      "password": "SuperSecret2024!"
    }
```

```bash
kubectl apply -f app-secret.yaml
```

### Step 4: สร้าง Pod หลัก

```yaml
# workshop-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volumes-demo
  namespace: volumes-workshop
  labels:
    app: volumes-demo
spec:
  initContainers:
  - name: setup
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Setting up initial data..."
      mkdir -p /data/uploads
      mkdir -p /data/cache
      echo "Setup complete at: $(date)" > /data/init.log
      echo "Done"
    volumeMounts:
    - name: app-data
      mountPath: /data
  
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
      name: http
    
    volumeMounts:
    # ConfigMap Volume สำหรับ config
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    
    # ConfigMap Volume สำหรับ HTML content
    - name: nginx-html
      mountPath: /usr/share/nginx/html/index.html
      subPath: index.html
    
    # emptyDir สำหรับ logs
    - name: nginx-logs
      mountPath: /var/log/nginx
    
    # emptyDir สำหรับ app data
    - name: app-data
      mountPath: /data
    
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 128Mi
  
  - name: log-watcher
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Log watcher started"
      while true; do
        if [ -f /logs/access.log ]; then
          LINES=$(wc -l < /logs/access.log)
          echo "$(date): Access log has $LINES lines"
        fi
        if [ -f /logs/error.log ]; then
          ERRORS=$(wc -l < /logs/error.log)
          echo "$(date): Error log has $ERRORS lines"
        fi
        sleep 20
      done
    volumeMounts:
    - name: nginx-logs
      mountPath: /logs
      readOnly: true
  
  - name: secret-checker
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Secrets available:"
      ls /secrets/
      echo "Admin password length: $(cat /secrets/admin-password | wc -c)"
      echo "DB Config:"
      cat /secrets/db-config.json
      sleep 3600
    volumeMounts:
    - name: app-secrets
      mountPath: /secrets
      readOnly: true
  
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
  
  - name: nginx-html
    configMap:
      name: nginx-config
  
  - name: nginx-logs
    emptyDir: {}
  
  - name: app-data
    emptyDir:
      sizeLimit: 1Gi
  
  - name: app-secrets
    secret:
      secretName: app-secret
      defaultMode: 0400
```

```bash
kubectl apply -f workshop-pod.yaml
```

### Step 5: สร้าง Service สำหรับ access

```yaml
# workshop-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: volumes-demo-svc
  namespace: volumes-workshop
spec:
  selector:
    app: volumes-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

```bash
kubectl apply -f workshop-service.yaml
```

### Step 6: ทดสอบและ Verify

```bash
# ดู Pod status
kubectl get pods -n volumes-workshop -w

# ดู Volume mounts
kubectl describe pod volumes-demo -n volumes-workshop | grep -A 30 "Volumes:"

# ดู logs ของแต่ละ container
kubectl logs volumes-demo -c nginx -n volumes-workshop
kubectl logs volumes-demo -c log-watcher -n volumes-workshop
kubectl logs volumes-demo -c secret-checker -n volumes-workshop

# Exec เข้าไปดู files
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- ls -la /etc/nginx/
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- cat /etc/nginx/nginx.conf

# ดู files ใน emptyDir
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- ls -la /var/log/nginx/

# Access web server
kubectl port-forward pod/volumes-demo 8080:80 -n volumes-workshop &
curl http://localhost:8080
```

### Step 7: ทดสอบ Data Sharing ระหว่าง Containers

```bash
# เขียนข้อมูลจาก nginx container
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- \
  /bin/sh -c 'echo "Test data from nginx" > /data/test.txt'

# อ่านข้อมูลจาก log-watcher container (ถ้า mount ร่วมกัน)
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- \
  cat /data/test.txt
```

### Step 8: ทดสอบ Container Restart (emptyDir ยังคงอยู่)

```bash
# เขียนข้อมูลลงใน emptyDir
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- \
  /bin/sh -c 'echo "This survives container restart" > /data/persist-test.txt'

# ดู container จาก nginx
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- \
  cat /data/persist-test.txt

# (Optional) จำลอง container restart ด้วยการ kill process
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- \
  /bin/sh -c 'kill 1'

# รอสักครู่แล้ว check ว่า file ยังอยู่ไหม
sleep 5
kubectl exec -it volumes-demo -c nginx -n volumes-workshop -- \
  cat /data/persist-test.txt   # ควรยังอยู่
```

### Step 9: ทำความสะอาด

```bash
kubectl delete namespace volumes-workshop
```

---

## 14. Best Practices สำหรับ Volumes

### Security Best Practices

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-volume-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000             # Group สำหรับ Volume files
    fsGroupChangePolicy: "OnRootMismatch"  # เปลี่ยน permission เฉพาะเมื่อจำเป็น
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true   # Root filesystem read-only
    
    volumeMounts:
    - name: secrets
      mountPath: /secrets
      readOnly: true
    - name: config
      mountPath: /config
      readOnly: true
    - name: tmp
      mountPath: /tmp        # Writable temp directory
    - name: data
      mountPath: /data       # Writable data directory
  
  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
      defaultMode: 0400      # Read-only for owner only
  - name: config
    configMap:
      name: app-config
      defaultMode: 0444      # Read-only for all
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi
  - name: data
    emptyDir:
      sizeLimit: 10Gi
```

### Performance Best Practices

```yaml
# ใช้ Memory medium สำหรับ high-performance cache
volumes:
- name: fast-cache
  emptyDir:
    medium: Memory
    sizeLimit: 512Mi

# กำหนด sizeLimit เสมอเพื่อป้องกัน disk exhaustion
- name: logs
  emptyDir:
    sizeLimit: 5Gi
```

---

## 15. Troubleshooting Volumes

### ปัญหาที่พบบ่อย

**1. Volume ไม่ถูก mount**
```bash
# ดู Pod events
kubectl describe pod <pod-name> | grep -A 20 "Events:"

# ตรวจสอบ Volume mounts
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].volumeMounts}'
```

**2. Permission Denied errors**
```bash
# ดู fsGroup
kubectl get pod <pod-name> -o jsonpath='{.spec.securityContext}'

# Exec เข้าไปดู permissions
kubectl exec -it <pod-name> -- ls -la /path/to/volume
```

**3. ConfigMap ไม่ update อัตโนมัติ**
- ConfigMap Volume จะ update ภายใน ~1-2 นาที (sync period)
- ถ้าใช้ subPath จะไม่ update อัตโนมัติ ต้อง restart Pod

```bash
# Force reload ConfigMap
kubectl rollout restart deployment/<deployment-name>
```

**4. Secret ไม่ mount**
```bash
# ตรวจสอบ Secret มีอยู่จริง
kubectl get secret <secret-name>

# ตรวจสอบ namespace ถูกต้อง
kubectl get secret <secret-name> -n <namespace>
```

---

## สรุป

| Volume Type | Lifecycle | Use Case | Data Persistence |
|-------------|-----------|----------|-----------------|
| emptyDir | Pod lifecycle | Temp storage, sharing | ไม่ persist |
| hostPath | Node lifecycle | Monitoring, DinD | Persist on node |
| configMap | External | Configuration | External |
| secret | External | Credentials | External |
| downwardAPI | External | Pod metadata | External |
| projected | Combined | Multiple sources | Mixed |

### Key Takeaways:
1. **emptyDir** เหมาะสำหรับ temporary storage และการแชร์ข้อมูลระหว่าง containers ใน Pod
2. **hostPath** ใช้ระวัง เพราะผูกกับ Node เฉพาะ และมีความเสี่ยงด้านความปลอดภัย
3. **configMap/secret** ควรใช้ `readOnly: true` เสมอ
4. กำหนด **sizeLimit** สำหรับ emptyDir เพื่อป้องกัน disk exhaustion
5. ใช้ **subPath** เมื่อต้องการ mount เฉพาะไฟล์ หรือใช้ Volume เดียวกันหลาย mount points
6. **fsGroup** ใน securityContext ช่วยจัดการ file permissions สำหรับ Volumes

## แหล่งเรียนรู้เพิ่มเติม

- [Kubernetes Volumes Documentation](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Configure a Pod to Use a Volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
- [Volumes - Kubernetes Reference](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/volume/)
