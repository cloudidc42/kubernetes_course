# Part 12: Pods - Building Block พื้นฐานของ Kubernetes

## สารบัญ
1. [Pod คืออะไร](#pod-คืออะไร)
2. [Pod YAML Manifest](#pod-yaml-manifest)
3. [Pod Lifecycle](#pod-lifecycle)
4. [Multi-container Pods](#multi-container-pods)
5. [Pod Networking](#pod-networking)
6. [Workshop: สร้างและจัดการ Pods](#workshop-สร้างและจัดการ-pods)

---

## 1. Pod คืออะไร

**Pod** คือหน่วยการ deploy ที่เล็กที่สุดใน Kubernetes Pod เป็นกลุ่มของ Container หนึ่งตัวหรือมากกว่า ที่:

- ทำงานอยู่บน Node เดียวกัน
- Share Network Namespace (IP address เดียวกัน)
- Share Storage Volumes
- ถูก schedule ขึ้นและลงพร้อมกัน

### ทำไมต้องมี Pod ไม่ใช่แค่ Container?

```
ไม่มี Pod (แค่ Container):          มี Pod (Container กลุ่ม):
┌──────────┐ ┌──────────┐           ┌─────────────────────────────┐
│Container1│ │Container2│           │  Pod                        │
│  IP:A    │ │  IP:B    │           │  ┌──────────┐ ┌──────────┐  │
└──────────┘ └──────────┘           │  │Container1│ │Container2│  │
                                    │  └──────────┘ └──────────┘  │
ปัญหา: Container ไม่รู้จักกัน       │  Share: IP, volumes, lifecycle│
                                    └─────────────────────────────┘
                                    ✓ Container สื่อสารกันผ่าน localhost
```

### Pod เปรียบเหมือนอะไร?

คิดว่า Pod เหมือน "เครื่องคอมพิวเตอร์เสมือน" ที่รัน Container หนึ่งตัวหรือมากกว่า โดย:
- Container ใน Pod เดียวกันสื่อสารกันผ่าน `localhost`
- แต่ละ Pod มี IP address เป็นของตัวเอง
- Pod ถ่ายโอน IP ใหม่ทุกครั้งที่ restart

### Pod vs Container vs Node

```
Node (Physical/Virtual Machine)
├── Pod A (IP: 10.244.1.1)
│   ├── Container: nginx (port 80)
│   └── Container: log-shipper
│
├── Pod B (IP: 10.244.1.2)
│   └── Container: redis (port 6379)
│
└── Pod C (IP: 10.244.1.3)
    ├── Container: app (port 8080)
    ├── Container: sidecar-proxy (port 8081)
    └── Volume: /data (shared between containers)
```

### ข้อสำคัญเกี่ยวกับ Pods

1. **Ephemeral (ชั่วคราว)**: Pods ไม่ควร assume ว่าจะมีชีวิตยาวนาน
2. **IP ไม่คงที่**: ทุกครั้งที่ Pod ถูกสร้างใหม่ จะได้ IP ใหม่
3. **ไม่ควรสร้าง Pod โดยตรง**: ควรใช้ Controllers (Deployment, StatefulSet, etc.)
4. **One Container Per Pod** เป็น best practice สำหรับ main process

---

## 2. Pod YAML Manifest

### โครงสร้าง YAML พื้นฐาน

```yaml
apiVersion: v1          # API version สำหรับ Pod
kind: Pod               # ประเภท Resource
metadata:               # ข้อมูล metadata
  name: my-pod          # ชื่อของ Pod (ต้องไม่ซ้ำกันใน namespace)
  namespace: default    # namespace (default ถ้าไม่ระบุ)
  labels:               # Labels สำหรับ select/group
    app: my-app
    env: production
  annotations:          # Annotations สำหรับข้อมูลเพิ่มเติม
    description: "My first pod"
spec:                   # ข้อกำหนดของ Pod
  containers:           # รายการ Containers
  - name: my-container  # ชื่อ Container
    image: nginx:1.25   # Docker Image
    ports:
    - containerPort: 80 # Port ที่ Container เปิด
```

### Pod YAML ที่สมบูรณ์

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-example-pod
  namespace: default
  labels:
    app: my-webapp
    version: "1.0"
    environment: production
    tier: frontend
  annotations:
    kubernetes.io/description: "Production web application pod"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    team: "web-team"
    contact: "webteam@example.com"
spec:
  # Container specification
  containers:
  - name: web-app
    image: nginx:1.25
    imagePullPolicy: IfNotPresent   # Always, Never, IfNotPresent
    
    # Ports
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    - name: metrics
      containerPort: 8080
      protocol: TCP
    
    # Environment Variables
    env:
    - name: APP_ENV
      value: "production"
    - name: APP_PORT
      value: "80"
    - name: SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: secret-key
    - name: DB_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database-url
    
    # Resource Requests and Limits
    resources:
      requests:
        cpu: 100m          # 100 millicores = 0.1 CPU core
        memory: 128Mi      # 128 Mebibytes
      limits:
        cpu: 500m          # 500 millicores = 0.5 CPU core
        memory: 512Mi      # 512 Mebibytes
    
    # Volume Mounts
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: data-volume
      mountPath: /var/www/html
    - name: logs-volume
      mountPath: /var/log/nginx
    
    # Liveness Probe - ตรวจสอบว่า Container ยังทำงานอยู่
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10   # รอ 10 วินาทีก่อนเริ่ม probe
      periodSeconds: 10         # probe ทุก 10 วินาที
      timeoutSeconds: 5         # timeout 5 วินาที
      failureThreshold: 3       # fail 3 ครั้งจึง restart
      successThreshold: 1       # success 1 ครั้งถือว่า live
    
    # Readiness Probe - ตรวจสอบว่า Container พร้อมรับ traffic
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 2       # ต้อง success 2 ครั้งติดกัน
    
    # Startup Probe - สำหรับ Application ที่ start ช้า
    startupProbe:
      httpGet:
        path: /started
        port: 80
      failureThreshold: 30
      periodSeconds: 10         # รอสูงสุด 300 วินาที (30 * 10)
    
    # Lifecycle Hooks
    lifecycle:
      postStart:                # รันหลังจาก Container start
        exec:
          command: ["/bin/sh", "-c", "echo Container started"]
      preStop:                  # รันก่อน Container stop
        exec:
          command: ["/usr/sbin/nginx", "-s", "quit"]
    
    # Security Context สำหรับ Container
    securityContext:
      runAsUser: 1000           # รันด้วย user ID 1000
      runAsGroup: 3000          # รันด้วย group ID 3000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL                   # ลบ capabilities ทั้งหมด
        add:
        - NET_BIND_SERVICE      # อนุญาตเฉพาะที่จำเป็น
  
  # Init Containers (รันก่อน main containers)
  initContainers:
  - name: init-db-check
    image: busybox:1.36
    command: ['sh', '-c', 
              'until nc -z postgres 5432; do echo waiting for postgres; sleep 2; done;']
  
  # Volumes ที่ใช้ใน Pod
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
  - name: data-volume
    emptyDir: {}               # Temporary directory
  - name: logs-volume
    emptyDir:
      medium: Memory           # RAM-backed volume (tmpfs)
      sizeLimit: 100Mi
  - name: persistent-data
    persistentVolumeClaim:
      claimName: my-pvc
  - name: host-data
    hostPath:
      path: /data
      type: DirectoryOrCreate
  
  # Pod-level Security Context
  securityContext:
    runAsNonRoot: true
    fsGroup: 2000              # Volume files owned by group 2000
    supplementalGroups: [1000]
    seccompProfile:
      type: RuntimeDefault
  
  # Scheduling
  nodeSelector:                # เลือก Node ตาม label
    disktype: ssd
    kubernetes.io/arch: amd64
  
  nodeName: worker-node-1      # หรือระบุ Node โดยตรง (ไม่แนะนำ)
  
  # Affinity Rules
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values:
            - amd64
            - arm64
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - my-webapp
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - my-webapp
          topologyKey: kubernetes.io/hostname
  
  # Tolerations - รัน Pod บน Tainted Nodes
  tolerations:
  - key: "special-node"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # ทน 300 วินาทีก่อน evict
  
  # Service Account
  serviceAccountName: my-service-account
  automountServiceAccountToken: false
  
  # Restart Policy
  restartPolicy: Always       # Always, OnFailure, Never
  
  # DNS Policy
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - my-namespace.svc.cluster.local
    options:
    - name: ndots
      value: "5"
  
  # Host Network (ใช้ Network ของ Node โดยตรง - ระวัง!)
  hostNetwork: false
  hostPID: false
  hostIPC: false
  
  # Termination Grace Period
  terminationGracePeriodSeconds: 30
  
  # Image Pull Secrets
  imagePullSecrets:
  - name: registry-credentials
  
  # Priority
  priorityClassName: high-priority
```

### Pod YAML แบบ Minimal (ใช้งานทั่วไป)

```yaml
# simple-nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-simple
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
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

---

## 3. Pod Lifecycle

### States ของ Pod

```
┌─────────┐     ┌─────────┐     ┌──────────┐     ┌─────────┐
│ Pending │────►│ Running │────►│Succeeded │     │ Failed  │
└─────────┘     └─────────┘     └──────────┘     └─────────┘
     │               │                                  ▲
     │               │                                  │
     ▼               ▼                                  │
┌─────────┐     ┌─────────┐                            │
│ Unknown │     │ Failed  │────────────────────────────┘
└─────────┘     └─────────┘
```

| Phase | คำอธิบาย |
|-------|----------|
| **Pending** | Pod ได้รับการ accept แต่ยังไม่มี Container ที่ running (กำลัง schedule หรือ download image) |
| **Running** | Pod ถูก bind กับ Node แล้ว และ Container อย่างน้อย 1 ตัวกำลัง running |
| **Succeeded** | Container ทั้งหมดใน Pod terminated ด้วย exit code 0 |
| **Failed** | Container อย่างน้อย 1 ตัว terminated ด้วย non-zero exit code |
| **Unknown** | ไม่สามารถ determine สถานะของ Pod ได้ (Node ปัญหา) |

### Container States

```yaml
# ดูด้วยคำสั่ง:
# kubectl describe pod <pod-name>

containerStatuses:
- name: nginx
  state:
    running:           # Container กำลัง running
      startedAt: "2024-01-15T10:00:00Z"
  
  # หรือ
  state:
    waiting:           # Container กำลังรอ
      reason: ContainerCreating
      message: "pulling image..."
  
  # หรือ
  state:
    terminated:        # Container หยุดทำงาน
      exitCode: 0
      reason: Completed
      finishedAt: "2024-01-15T11:00:00Z"
  
  ready: true
  restartCount: 0
```

### Pod Conditions

```bash
# ดู Pod Conditions
kubectl describe pod nginx | grep -A 10 "Conditions:"

# Conditions:
#   Type              Status
#   Initialized       True    # Init containers เสร็จสิ้น
#   Ready             True    # Container พร้อมรับ traffic
#   ContainersReady   True    # Container ทั้งหมดพร้อม
#   PodScheduled      True    # Pod ถูก schedule ไปยัง Node แล้ว
```

### Restart Policy

```yaml
# restartPolicy options:
spec:
  restartPolicy: Always       # restart เสมอ (default สำหรับ Deployment)
  # restartPolicy: OnFailure  # restart เฉพาะเมื่อ exit code ไม่ใช่ 0
  # restartPolicy: Never      # ไม่ restart เลย (ใช้กับ batch jobs)
```

### Probe Types

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    
    # 1. HTTP GET Probe
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: MyValue
    
    # 2. TCP Socket Probe
    readinessProbe:
      tcpSocket:
        port: 3306    # เหมาะสำหรับ databases
      initialDelaySeconds: 5
      periodSeconds: 10
    
    # 3. Exec Probe
    startupProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - pg_isready -U postgres    # รันคำสั่งใน container
      failureThreshold: 30
      periodSeconds: 10
    
    # 4. gRPC Probe (Kubernetes 1.24+)
    livenessProbe:
      grpc:
        port: 8080
        service: myapp.v1.HealthService
```

---

## 4. Multi-container Pods

### เมื่อไหร่ควรใช้ Multi-container Pod?

ใช้เมื่อ Container หลายตัวต้องการ:
- Share network (สื่อสารผ่าน localhost)
- Share storage volumes
- Start และ stop พร้อมกัน

### Patterns สำหรับ Multi-container Pod

#### Pattern 1: Sidecar Pattern

```yaml
# sidecar-pod.yaml
# Main container + helper container ที่ช่วยเสริม
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-sidecar
spec:
  containers:
  - name: webapp                    # Main container
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  - name: log-shipper              # Sidecar: ส่ง logs ไปยัง centralized system
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx     # อ่าน logs จาก main container
      readOnly: true
    - name: fluent-config
      mountPath: /fluent-bit/etc
  
  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: fluent-config
    configMap:
      name: fluent-bit-config
```

#### Pattern 2: Ambassador Pattern

```yaml
# ambassador-pod.yaml
# Proxy container ที่ทำหน้าที่ relay connections
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-ambassador
spec:
  containers:
  - name: webapp                    # Main container
    image: myapp:latest
    env:
    - name: DB_HOST
      value: localhost              # Connect ผ่าน localhost (ambassador)
    - name: DB_PORT
      value: "6379"
  
  - name: redis-ambassador          # Ambassador: proxy ไปยัง Redis cluster
    image: redis-ambassador:latest
    ports:
    - containerPort: 6379
    env:
    - name: REDIS_CLUSTER_URL
      value: redis-cluster.production.svc.cluster.local
```

#### Pattern 3: Adapter Pattern

```yaml
# adapter-pod.yaml
# Adapter container ที่แปลง output ของ main container
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-adapter
spec:
  containers:
  - name: webapp                    # Main container (output แปลกๆ)
    image: legacy-app:latest
    volumeMounts:
    - name: app-logs
      mountPath: /app/logs
  
  - name: metrics-adapter           # Adapter: แปลง custom metrics เป็น Prometheus format
    image: metrics-adapter:latest
    ports:
    - containerPort: 9090           # Prometheus scrape ที่นี่
    volumeMounts:
    - name: app-logs
      mountPath: /app/logs
      readOnly: true
  
  volumes:
  - name: app-logs
    emptyDir: {}
```

#### Pattern 4: Init Containers

```yaml
# init-container-pod.yaml
# Init containers รันก่อน main containers และต้องสำเร็จก่อน
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-init
spec:
  initContainers:
  - name: init-permissions            # ตั้งค่า permissions
    image: busybox:1.36
    command: ['sh', '-c', 'chmod 755 /data && chown 1000:1000 /data']
    volumeMounts:
    - name: app-data
      mountPath: /data
  
  - name: init-config                 # Download config จาก external source
    image: alpine:latest
    command: ['sh', '-c', 
      'wget -O /etc/app/config.json https://config.example.com/app.json']
    volumeMounts:
    - name: config-vol
      mountPath: /etc/app
  
  - name: init-wait-db               # รอ database พร้อม
    image: postgres:15
    command: ['sh', '-c', 
      'until pg_isready -h postgres -p 5432; do echo waiting; sleep 2; done']
  
  containers:
  - name: webapp
    image: myapp:latest
    volumeMounts:
    - name: app-data
      mountPath: /data
    - name: config-vol
      mountPath: /etc/app
  
  volumes:
  - name: app-data
    emptyDir: {}
  - name: config-vol
    emptyDir: {}
```

### Containers สื่อสารกันใน Pod

```yaml
# inter-container-communication.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-port-pod
spec:
  containers:
  - name: nginx-proxy               # Port 80: รับ traffic จากภายนอก
    image: nginx:1.25
    ports:
    - containerPort: 80
    # nginx config proxy_pass ไปยัง localhost:8080
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
  
  - name: backend-app               # Port 8080: รับ traffic จาก nginx
    image: myapp:latest
    ports:
    - containerPort: 8080
    # Backend สามารถ listen บน 0.0.0.0:8080
    # nginx สามารถ proxy_pass http://localhost:8080 ได้

  volumes:
  - name: nginx-config
    configMap:
      name: nginx-proxy-config
```

---

## 5. Pod Networking

### Pod Network Model

Kubernetes ใช้ **flat network model** ที่:
- ทุก Pod มี IP address ของตัวเองที่ unique ทั่วทั้ง Cluster
- Pod ทุกตัวสามารถ communicate กันโดยตรงโดยไม่ต้อง NAT
- Container ใน Pod เดียวกัน share network namespace

```
Cluster Network: 10.0.0.0/8
├── Node 1: 10.0.1.0/24
│   ├── Pod A: 10.0.1.1 (nginx)
│   └── Pod B: 10.0.1.2 (redis)
│
└── Node 2: 10.0.2.0/24
    ├── Pod C: 10.0.2.1 (webapp)
    └── Pod D: 10.0.2.2 (api)

Pod A (10.0.1.1) ──────────────────► Pod C (10.0.2.1)
ไม่ต้อง NAT! ไม่ต้องผ่าน Port Mapping!
```

### DNS ใน Kubernetes

```bash
# DNS Pattern:
# <service-name>.<namespace>.svc.cluster.local

# ตัวอย่าง:
# Service: my-service ใน namespace: production
# DNS: my-service.production.svc.cluster.local

# ถ้าอยู่ใน namespace เดียวกัน:
# DNS: my-service (short name)

# ตรวจสอบ DNS ใน Pod
kubectl exec my-pod -- nslookup kubernetes
kubectl exec my-pod -- cat /etc/resolv.conf
```

### ตัวอย่าง Pod ที่เข้าถึง Service

```yaml
# pods-dns-example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test-pod
spec:
  containers:
  - name: dns-test
    image: busybox:1.36
    command: ['sh', '-c', 'sleep 3600']
    env:
    - name: REDIS_URL
      value: redis-service.default.svc.cluster.local
    - name: POSTGRES_URL
      value: postgres-service.database.svc.cluster.local
```

### Host Networking

```yaml
# host-network-pod.yaml (ใช้เฉพาะกรณีพิเศษ)
apiVersion: v1
kind: Pod
metadata:
  name: host-network-pod
spec:
  hostNetwork: true          # ใช้ Network ของ Node โดยตรง
  hostPID: true              # Share PID namespace กับ Node
  containers:
  - name: network-debug
    image: nicolaka/netshoot
    command: ['sleep', '3600']
    securityContext:
      privileged: true
```

---

## 6. Workshop: สร้างและจัดการ Pods

### Workshop Setup

```bash
# สร้าง namespace สำหรับ workshop
kubectl create namespace pod-workshop

# ตั้ง default namespace
kubectl config set-context --current --namespace=pod-workshop
```

### Lab 1: สร้าง Pod แรกของคุณ

```bash
# Step 1: สร้าง simple pod จาก command line
kubectl run my-first-pod --image=nginx:1.25 --port=80

# Step 2: ดูสถานะ
kubectl get pod my-first-pod
kubectl get pod my-first-pod -o wide   # ดู IP และ Node

# Step 3: ดูรายละเอียด
kubectl describe pod my-first-pod

# Step 4: ดู logs
kubectl logs my-first-pod

# Step 5: เข้า interactive shell
kubectl exec -it my-first-pod -- /bin/bash
# ภายใน container:
# ls /etc/nginx
# cat /etc/nginx/nginx.conf
# curl localhost
# exit

# Step 6: ทดสอบ port-forward
kubectl port-forward pod/my-first-pod 8080:80 &
curl http://localhost:8080
kill %1   # หยุด port-forward

# Cleanup Lab 1
kubectl delete pod my-first-pod
```

### Lab 2: สร้าง Pod จาก YAML Manifest

```bash
# Step 1: สร้างไฟล์ manifest
cat <<'EOF' > /tmp/nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: pod-workshop
  labels:
    app: nginx
    version: "1.25"
    environment: workshop
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
      name: http
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    volumeMounts:
    - name: custom-html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: custom-html
    configMap:
      name: nginx-html-config
EOF

# Step 2: สร้าง ConfigMap สำหรับ custom HTML
cat <<'EOF' > /tmp/nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html-config
  namespace: pod-workshop
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>My Kubernetes Pod</title></head>
    <body>
      <h1>Hello from Kubernetes Pod!</h1>
      <p>This page is served by nginx running in a Kubernetes Pod</p>
    </body>
    </html>
EOF

# Step 3: Apply manifests
kubectl apply -f /tmp/nginx-configmap.yaml
kubectl apply -f /tmp/nginx-pod.yaml

# Step 4: รอ Pod พร้อม
kubectl wait --for=condition=Ready pod/nginx-pod --timeout=60s

# Step 5: ทดสอบ
kubectl port-forward pod/nginx-pod 8080:80 &
sleep 2
curl http://localhost:8080
kill %1

# Cleanup Lab 2
kubectl delete -f /tmp/nginx-pod.yaml
kubectl delete -f /tmp/nginx-configmap.yaml
```

### Lab 3: Pod Lifecycle Observation

```bash
# Step 1: สร้าง Pod ที่จะ complete
cat <<'EOF' > /tmp/job-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: job-pod
  namespace: pod-workshop
spec:
  containers:
  - name: job
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Starting job..."; sleep 10; echo "Job completed!"']
  restartPolicy: Never
EOF

kubectl apply -f /tmp/job-pod.yaml

# Step 2: Watch Pod lifecycle
kubectl get pod job-pod --watch &
WATCH_PID=$!

# รอ Pod เสร็จ
sleep 20
kill $WATCH_PID 2>/dev/null

# ดูสถานะสุดท้าย
kubectl get pod job-pod
kubectl logs job-pod

# Step 3: สร้าง Pod ที่จะ fail
cat <<'EOF' > /tmp/fail-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fail-pod
  namespace: pod-workshop
spec:
  containers:
  - name: fail
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Failing..."; exit 1']
  restartPolicy: OnFailure
EOF

kubectl apply -f /tmp/fail-pod.yaml

# ดู Pod restart
kubectl get pod fail-pod --watch &
WATCH_PID=$!
sleep 30
kill $WATCH_PID 2>/dev/null

# ดู restart count
kubectl get pod fail-pod

# Cleanup
kubectl delete pod job-pod fail-pod
```

### Lab 4: Multi-container Pod

```bash
# สร้าง Multi-container Pod
cat <<'EOF' > /tmp/multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  namespace: pod-workshop
spec:
  # Init container รันก่อน
  initContainers:
  - name: init-message
    image: busybox:1.36
    command: ['sh', '-c', 
      'echo "Initialized at $(date)" > /shared/init.txt']
    volumeMounts:
    - name: shared
      mountPath: /shared
  
  containers:
  # Main container: web server
  - name: web-server
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared
      mountPath: /usr/share/nginx/html
  
  # Sidecar container: file watcher
  - name: file-watcher
    image: busybox:1.36
    command: ['sh', '-c', 
      'while true; do echo "Files: $(ls /shared)"; sleep 5; done']
    volumeMounts:
    - name: shared
      mountPath: /shared
      readOnly: true
  
  volumes:
  - name: shared
    emptyDir: {}
EOF

kubectl apply -f /tmp/multi-container-pod.yaml

# รอ Pod พร้อม
kubectl wait --for=condition=Ready pod/multi-container-pod --timeout=60s

# ดูทั้งสอง containers
kubectl logs multi-container-pod -c web-server
kubectl logs multi-container-pod -c file-watcher

# เข้า specific container
kubectl exec -it multi-container-pod -c web-server -- ls /usr/share/nginx/html
kubectl exec -it multi-container-pod -c file-watcher -- cat /shared/init.txt

# Port forward ไป main container
kubectl port-forward pod/multi-container-pod 8080:80 &
sleep 2
curl http://localhost:8080
kill %1

# Cleanup Lab 4
kubectl delete pod multi-container-pod
```

### Lab 5: Pod Resource Management

```bash
# สร้าง Pod พร้อม Resource Limits
cat <<'EOF' > /tmp/resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo-pod
  namespace: pod-workshop
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
EOF

kubectl apply -f /tmp/resource-pod.yaml

# ดู resource usage (ต้องมี metrics-server)
kubectl top pod resource-demo-pod

# ดู resource requests/limits
kubectl get pod resource-demo-pod \
    -o jsonpath='{.spec.containers[0].resources}'

# Cleanup Lab 5
kubectl delete pod resource-demo-pod
```

### Lab 6: Pod Anti-Affinity (Spread Pods)

```bash
# สร้าง Pods กระจายไปยัง Nodes ต่างกัน
cat <<'EOF' > /tmp/spread-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-1
  namespace: pod-workshop
  labels:
    app: web
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: web-2
  namespace: pod-workshop
  labels:
    app: web
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx:1.25
EOF

kubectl apply -f /tmp/spread-pods.yaml

# ดูว่า Pods อยู่บน Node ไหน
kubectl get pods web-1 web-2 -o wide

# Cleanup Lab 6
kubectl delete pod web-1 web-2
```

### Cleanup Workshop

```bash
# ลบทุกอย่างใน namespace
kubectl delete all --all -n pod-workshop
kubectl delete configmap --all -n pod-workshop

# ลบ namespace
kubectl delete namespace pod-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/nginx-pod.yaml /tmp/nginx-configmap.yaml \
    /tmp/job-pod.yaml /tmp/fail-pod.yaml \
    /tmp/multi-container-pod.yaml /tmp/resource-pod.yaml \
    /tmp/spread-pods.yaml
```

### Tips และ Best Practices สำหรับ Pods

```bash
# 1. ตรวจสอบ Pod ที่มีปัญหา
kubectl get pods -A | grep -v Running | grep -v Completed

# 2. ดู Events ทั้งหมดที่เกี่ยวกับ Pod
kubectl get events --field-selector involvedObject.name=my-pod

# 3. ดู Pod IP
kubectl get pod my-pod -o jsonpath='{.status.podIP}'

# 4. ดู Node ที่ Pod รันอยู่
kubectl get pod my-pod -o jsonpath='{.spec.nodeName}'

# 5. Force delete Pod ที่ติดอยู่
kubectl delete pod my-pod --grace-period=0 --force

# 6. ดู Previous Container logs (หลัง restart)
kubectl logs my-pod --previous

# 7. Debug Pod ที่ crash ด้วย ephemeral container (Kubernetes 1.23+)
kubectl debug -it my-pod --image=busybox --target=my-container

# 8. Copy entire directory จาก Pod
kubectl cp my-pod:/var/log ./local-logs/
```

### Pod Design Best Practices

1. **One main process per container** - แต่ละ container ควรทำงานเดียว
2. **Use init containers** สำหรับ setup tasks (ไม่ใช่ main container)
3. **Always set resource requests and limits** เพื่อป้องกัน resource starvation
4. **Implement health probes** (liveness, readiness, startup)
5. **Use labels consistently** เพื่อให้ select ได้ง่าย
6. **Avoid storing state in Pods** - ใช้ external storage แทน
7. **Set terminationGracePeriodSeconds** ให้เหมาะสม
8. **Never create standalone Pods in production** - ใช้ Controllers

---

## สรุป

Pod คือ building block พื้นฐานของ Kubernetes ที่ต้องเข้าใจก่อนจะเรียนรู้ Concepts ขั้นสูงอื่นๆ:

- Pod เป็นกลุ่มของ Container ที่ share network และ storage
- Pod มี Lifecycle ที่ต้องเข้าใจ (Pending → Running → Succeeded/Failed)
- Multi-container Patterns (Sidecar, Ambassador, Adapter, Init)
- Pod Networking ใช้ flat network model
- ไม่ควรสร้าง Pod โดยตรงใน production - ใช้ Controllers

ในบทต่อไปเราจะเรียนรู้ **ReplicaSet** ซึ่งเป็น Controller ที่จัดการการรัน Pods ให้ครบตามจำนวนที่กำหนด
