# Part 15: Services - Networking และการเข้าถึง Applications

## สารบัญ
1. [Service คืออะไร](#service-คืออะไร)
2. [ClusterIP, NodePort, LoadBalancer, ExternalName](#service-types)
3. [Service YAML](#service-yaml)
4. [Endpoints](#endpoints)
5. [Workshop: Expose Deployment ด้วย Service](#workshop)

---

## 1. Service คืออะไร

**Service** คือ abstraction layer ที่ให้ stable network endpoint สำหรับเข้าถึง Pods เนื่องจาก Pods มี IP ที่เปลี่ยนแปลงได้ตลอดเวลา

### ปัญหาที่ Service แก้ไข

```
ปัญหา: Pod IP เปลี่ยนตลอด
Pod A: 10.244.1.5  → restart → Pod A: 10.244.1.8 (IP ใหม่!)
Pod B: 10.244.1.6  → restart → Pod B: 10.244.1.9 (IP ใหม่!)
Pod C: 10.244.1.7  → restart → Pod C: 10.244.1.10 (IP ใหม่!)

ถ้า Client ต้อง hardcode IP → พัง!

แก้ด้วย Service:
Client → Service (10.96.100.1 - ไม่เปลี่ยน!) → Pods
         Service ทำ load balancing ให้อัตโนมัติ
```

### Service ทำงานอย่างไร

```
1. Service มี Selector ที่ match Pod labels
2. kube-proxy บน Node แต่ละตัว update iptables/ipvs rules
3. Traffic ที่มาถึง Service IP/Port จะถูก forward ไปยัง Pod IPs

Service: webapp-service (10.96.100.1:80)
         ↓ kube-proxy/iptables rules
         ├── Pod 1 (10.244.1.5:8080)
         ├── Pod 2 (10.244.2.3:8080)  ← load balance
         └── Pod 3 (10.244.3.1:8080)
```

### Service Components

```yaml
# Service มีองค์ประกอบหลัก:
spec:
  selector:        # เลือก Pods ที่จะ forward traffic ไป
    app: webapp
  ports:
  - port: 80       # Port ของ Service (Client ใช้ port นี้)
    targetPort: 8080  # Port ของ Pod
    protocol: TCP
  type: ClusterIP  # Service type
  clusterIP: 10.96.100.1  # Virtual IP (assigned อัตโนมัติ)
```

---

## 2. Service Types

### 2.1 ClusterIP (Default)

**ClusterIP** ให้ Virtual IP ที่เข้าถึงได้จากภายใน Cluster เท่านั้น

```
External User → ❌ ไม่สามารถเข้าถึงได้
Internal Pod → ✓ เข้าถึงได้ผ่าน ClusterIP หรือ DNS

เมื่อ: frontend pod เรียก backend service
```

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: default
spec:
  type: ClusterIP           # Default type
  selector:
    app: backend
  ports:
  - name: http
    port: 80                # Service port
    targetPort: 8080        # Container port
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
```

```bash
# ใช้งาน ClusterIP
# จาก Pod ภายใน Cluster:
curl http://backend-service         # short DNS
curl http://backend-service.default  # ชื่อเต็ม
curl http://backend-service.default.svc.cluster.local  # FQDN
```

### 2.2 NodePort

**NodePort** เปิด Port บน ทุก Node เพื่อให้เข้าถึงจากภายนอก Cluster ได้

```
External User → Node IP:NodePort → Service → Pods

Port Range: 30000-32767 (default)

Node 1 (192.168.1.10:30080)
Node 2 (192.168.1.11:30080)  → Service → Pods
Node 3 (192.168.1.12:30080)
```

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
  namespace: default
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - name: http
    port: 80              # Service port (ClusterIP port)
    targetPort: 8080      # Container port
    nodePort: 30080       # Node port (30000-32767)
    # ถ้าไม่ระบุ nodePort Kubernetes จะ assign อัตโนมัติ
```

```bash
# เข้าถึง NodePort
curl http://<any-node-ip>:30080

# ดู NodePort ที่ assign
kubectl get service webapp-nodeport
# NAME              TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)        AGE
# webapp-nodeport   NodePort   10.96.100.1    <none>       80:30080/TCP   5m
```

### 2.3 LoadBalancer

**LoadBalancer** สร้าง External Load Balancer (Cloud Provider) ที่มี External IP สำหรับเข้าถึงจาก Internet

```
Internet → External LB (1.2.3.4:80) → NodePort → Service → Pods

ต้องใช้ใน Cloud Environment:
- AWS: Elastic Load Balancer (ELB)
- GCP: Cloud Load Balancing
- Azure: Azure Load Balancer
- On-premises: MetalLB
```

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-lb
  namespace: production
  annotations:
    # AWS-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    # GCP-specific annotations
    cloud.google.com/load-balancer-type: "External"
    # Azure-specific annotations
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  loadBalancerSourceRanges:   # Whitelist source IPs
  - "10.0.0.0/8"             # อนุญาตเฉพาะ internal IPs
  - "203.0.113.0/24"         # และ specific external IPs
  # externalTrafficPolicy: Local  # Preserve client IP (ไม่ SNAT)
```

```bash
# ดู External IP
kubectl get service webapp-lb
# NAME        TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
# webapp-lb   LoadBalancer   10.96.100.2    1.2.3.4        80:31234/TCP   5m

# เข้าถึง
curl http://1.2.3.4
```

### 2.4 ExternalName

**ExternalName** map Service ชื่อใน Kubernetes ไปยัง External DNS name

```
Pod → Service (database-service) → CNAME → external-database.example.com

ใช้เมื่อ:
- ต้องการ access external database โดยใช้ Kubernetes DNS
- Migration: ย้าย service ทีละน้อย
```

```yaml
# externalname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
  namespace: default
spec:
  type: ExternalName
  externalName: database.example.com    # External DNS name
  # ไม่มี selector!
  # ไม่มี ports!
```

```bash
# Pod สามารถ access ผ่าน:
# external-database.default.svc.cluster.local
# → CNAME → database.example.com

# ใช้งาน
kubectl exec my-pod -- nslookup external-database
# Server: 10.96.0.10
# Address: 10.96.0.10#53
# external-database.default.svc.cluster.local canonical name = database.example.com
```

### 2.5 Headless Service

**Headless Service** ไม่มี ClusterIP - Client ได้รับ IP ของ Pods โดยตรงจาก DNS

```
ปกติ: Service (ClusterIP) → Pods (load balanced)
Headless: DNS query → list of Pod IPs (client ทำ load balance เอง)

ใช้กับ:
- StatefulSets (Pods มีชื่อเฉพาะ)
- Custom service discovery
- Databases ที่ต้องการ direct connection
```

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: database
spec:
  clusterIP: None         # Headless = ClusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
# DNS query จะคืน Pod IPs โดยตรง
kubectl exec my-pod -- nslookup postgres-headless.database.svc.cluster.local
# Name: postgres-headless.database.svc.cluster.local
# Address: 10.244.1.5    # Pod 1 IP
# Address: 10.244.2.3    # Pod 2 IP
# Address: 10.244.3.1    # Pod 3 IP
```

### สรุปเปรียบเทียบ Service Types

| Type | Access | Use Case | External IP |
|------|--------|----------|-------------|
| ClusterIP | ภายใน Cluster | Internal services | ไม่มี |
| NodePort | NodeIP:Port | Dev/Test environments | Node IP |
| LoadBalancer | External LB IP | Production cloud | External IP |
| ExternalName | DNS CNAME | External services | ไม่มี |
| Headless | Pod IPs | StatefulSets, DBs | ไม่มี |

---

## 3. Service YAML

### Service ที่มีหลาย Ports

```yaml
# multi-port-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: multi-port-app
  ports:
  - name: http           # ต้องระบุชื่อเมื่อมีหลาย ports
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP
  - name: grpc
    port: 9000
    targetPort: 9000
    protocol: TCP
```

### Service ที่ใช้ Named Port

```yaml
# Pod ที่ตั้งชื่อ port
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - name: http-port      # ตั้งชื่อ port
      containerPort: 8080
    - name: metrics-port
      containerPort: 9090
---
# Service ที่ reference ชื่อ port
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: http-port    # reference ชื่อ port แทน number
  - name: metrics
    port: 9090
    targetPort: metrics-port
```

### Service กับ Session Affinity

```yaml
# session-affinity-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
spec:
  selector:
    app: stateful-app
  ports:
  - port: 80
    targetPort: 8080
  # Session Affinity: Client เดิมจะ route ไป Pod เดิมเสมอ
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600    # session อยู่ 1 ชั่วโมง
```

### Service ที่ไม่มี Selector (Manual Endpoints)

```yaml
# service-without-selector.yaml
# ใช้เมื่อต้องการ point ไปยัง external IP หรือ different service
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  # ไม่มี selector!
---
# กำหนด Endpoints ด้วยมือ
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service    # ต้องชื่อเดียวกับ Service
subsets:
- addresses:
  - ip: 192.168.100.10      # External server IP
  - ip: 192.168.100.11
  ports:
  - port: 80
```

---

## 4. Endpoints

**Endpoints** คือ list ของ IP:Port ที่ Service จะ forward traffic ไป

### ดู Endpoints

```bash
# ดู Endpoints ทั้งหมด
kubectl get endpoints
kubectl get ep   # short name

# ดู Endpoints ของ Service เฉพาะ
kubectl get endpoints my-service
kubectl describe endpoints my-service

# Output:
# NAME         ENDPOINTS                                    AGE
# my-service   10.244.1.5:8080,10.244.2.3:8080,10.244.3.1:8080   5m
```

### ทำความเข้าใจ Endpoints Lifecycle

```
1. สร้าง Service with selector
2. Endpoint controller ค้นหา Pods ที่ match selector
3. Endpoint controller สร้าง/อัปเดต Endpoints object
4. kube-proxy อ่าน Endpoints แล้วอัปเดต iptables/ipvs

เมื่อ Pod เปลี่ยน (restart, scale):
1. Pod IP เปลี่ยน
2. Endpoint controller อัปเดต Endpoints
3. kube-proxy อัปเดต rules
4. Service ยังทำงาน ✓
```

### Endpoints Slices (ใหม่กว่า)

```bash
# Endpoint Slices แบ่ง Endpoints เป็น Slices เล็กๆ สำหรับ large clusters
kubectl get endpointslices
kubectl get endpointslices -l kubernetes.io/service-name=my-service

# ดูรายละเอียด
kubectl describe endpointslice my-service-xxxxx
```

---

## 5. Workshop: Expose Deployment ด้วย Service

### Workshop Setup

```bash
# สร้าง namespace
kubectl create namespace service-workshop
kubectl config set-context --current --namespace=service-workshop
```

### Lab 1: ClusterIP Service

```bash
# Step 1: สร้าง Deployment
cat <<'EOF' > /tmp/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: service-workshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo:latest
        args:
        - "-text=Hello from Backend Pod: $(POD_NAME)"
        - "-listen=:5678"
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 5678
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
EOF

kubectl apply -f /tmp/backend-deployment.yaml

# Step 2: สร้าง ClusterIP Service
cat <<'EOF' > /tmp/backend-clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-clusterip
  namespace: service-workshop
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 5678
EOF

kubectl apply -f /tmp/backend-clusterip-service.yaml

# Step 3: ดู Service และ Endpoints
kubectl get service backend-clusterip
kubectl get endpoints backend-clusterip
kubectl describe service backend-clusterip

# Step 4: ทดสอบจาก Pod ภายใน Cluster
kubectl run test-pod --image=curlimages/curl:latest \
    --restart=Never \
    -- sleep 3600

kubectl wait --for=condition=Ready pod/test-pod --timeout=60s

# เรียก ClusterIP service
kubectl exec test-pod -- curl -s http://backend-clusterip

# รัน request หลายครั้งเพื่อดู load balancing
for i in {1..5}; do
    kubectl exec test-pod -- curl -s http://backend-clusterip
    echo ""
done
# จะเห็น responses จาก Pods ต่างกัน (load balancing!)

# ทดสอบ DNS
kubectl exec test-pod -- nslookup backend-clusterip
kubectl exec test-pod -- nslookup backend-clusterip.service-workshop.svc.cluster.local

# Cleanup Lab 1
kubectl delete pod test-pod
kubectl delete -f /tmp/backend-clusterip-service.yaml
```

### Lab 2: NodePort Service

```bash
# สร้าง NodePort Service
cat <<'EOF' > /tmp/webapp-nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
  namespace: service-workshop
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 5678
    nodePort: 30080    # เลือก port ที่ต้องการ
EOF

kubectl apply -f /tmp/webapp-nodeport-service.yaml

# ดู Service
kubectl get service webapp-nodeport
# NAME              TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)        AGE
# webapp-nodeport   NodePort   10.96.10.5    <none>       80:30080/TCP   30s

# ดู Node IPs
kubectl get nodes -o wide

# เข้าถึงจากภายนอก (ใช้ Node IP จาก output ข้างบน)
# curl http://<NODE_IP>:30080

# ทดสอบ via port-forward (ถ้าไม่มี external access)
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "NodePort URL: http://$NODE_IP:30080"

# Cleanup Lab 2
kubectl delete -f /tmp/webapp-nodeport-service.yaml
```

### Lab 3: Service Discovery และ DNS

```bash
# สร้าง Services ใน namespaces ต่างกัน
kubectl create namespace ns-a
kubectl create namespace ns-b

# สร้าง service ใน ns-a
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
  namespace: ns-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:latest
        args: ["-text=I am Service A", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: service-a
  namespace: ns-a
spec:
  selector:
    app: service-a
  ports:
  - port: 80
    targetPort: 5678
EOF

# สร้าง service ใน ns-b
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
  namespace: ns-b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:latest
        args: ["-text=I am Service B", "-listen=:5678"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
  namespace: ns-b
spec:
  selector:
    app: service-b
  ports:
  - port: 80
    targetPort: 5678
EOF

# สร้าง Pod สำหรับทดสอบ DNS
kubectl run dns-test --image=curlimages/curl:latest \
    --namespace=ns-a \
    --restart=Never \
    -- sleep 3600

kubectl wait --for=condition=Ready pod/dns-test -n ns-a --timeout=60s

# ทดสอบ DNS resolution ต่างๆ
echo "=== Short DNS (same namespace) ==="
kubectl exec -n ns-a dns-test -- curl -s http://service-a

echo "=== FQDN (different namespace) ==="
kubectl exec -n ns-a dns-test -- curl -s http://service-b.ns-b.svc.cluster.local

echo "=== DNS lookup ==="
kubectl exec -n ns-a dns-test -- nslookup service-a
kubectl exec -n ns-a dns-test -- nslookup service-b.ns-b.svc.cluster.local

# Cleanup Lab 3
kubectl delete namespace ns-a ns-b
```

### Lab 4: LoadBalancer Service (Simulation)

```bash
# ถ้าใช้ minikube ต้องรัน: minikube tunnel ก่อน
# ถ้าใช้ kind ต้องใช้ MetalLB

# สร้าง LoadBalancer Service
cat <<'EOF' > /tmp/webapp-lb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-loadbalancer
  namespace: service-workshop
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 5678
EOF

kubectl apply -f /tmp/webapp-lb-service.yaml

# ดู Service (External IP อาจแสดง <pending> ถ้าไม่มี cloud LB)
kubectl get service webapp-loadbalancer
kubectl describe service webapp-loadbalancer

# สำหรับ minikube:
# minikube service webapp-loadbalancer -n service-workshop --url

# Cleanup Lab 4
kubectl delete -f /tmp/webapp-lb-service.yaml
```

### Lab 5: Headless Service กับ StatefulSet

```bash
# สร้าง Headless Service + StatefulSet
cat <<'EOF' > /tmp/stateful-headless.yaml
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: service-workshop
  labels:
    app: postgres
spec:
  clusterIP: None          # Headless!
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres
---
# Regular Service สำหรับ read
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: service-workshop
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: service-workshop
spec:
  serviceName: postgres-headless    # ใช้ headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: busybox:1.36
        command: ['sh', '-c', 'echo "$(hostname)" > /data/hostname.txt && sleep 3600']
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

kubectl apply -f /tmp/stateful-headless.yaml

# รอ Pods พร้อม
kubectl get pods -n service-workshop -l app=postgres --watch &
WATCH_PID=$!
sleep 30
kill $WATCH_PID 2>/dev/null

# ดู Pods (StatefulSet ตั้งชื่อแบบ ordered: postgres-0, postgres-1, postgres-2)
kubectl get pods -n service-workshop -l app=postgres

# สร้าง Pod สำหรับทดสอบ DNS
kubectl run dns-tester \
    --image=busybox:1.36 \
    --namespace=service-workshop \
    --restart=Never \
    -- sleep 3600

kubectl wait --for=condition=Ready pod/dns-tester -n service-workshop --timeout=60s

# ทดสอบ DNS สำหรับ Headless Service
echo "=== Headless Service DNS ==="
kubectl exec -n service-workshop dns-tester -- nslookup postgres-headless.service-workshop.svc.cluster.local
# จะได้ IP ของ Pods ทุกตัว!

echo "=== Access individual Pod ==="
kubectl exec -n service-workshop dns-tester -- nslookup postgres-0.postgres-headless.service-workshop.svc.cluster.local
# เข้าถึง Pod ที่ระบุโดยตรง!

# Cleanup Lab 5
kubectl delete pod dns-tester -n service-workshop
kubectl delete -f /tmp/stateful-headless.yaml
# ลบ PVCs ที่สร้างโดย StatefulSet
kubectl delete pvc -l app=postgres -n service-workshop
```

### Lab 6: Service ที่ไม่มี Selector

```bash
# สถานการณ์: ต้องการ connect ไปยัง external database หรือ service นอก Cluster

# สร้าง Service ที่ไม่มี selector + Endpoints ด้วยมือ
cat <<'EOF' > /tmp/external-service.yaml
# Service ที่ไม่มี selector
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: service-workshop
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  # ไม่มี selector!
---
# กำหนด Endpoints ด้วยมือ
apiVersion: v1
kind: Endpoints
metadata:
  name: external-api    # ต้องชื่อเดียวกับ Service!
subsets:
- addresses:
  - ip: 8.8.8.8        # ตัวอย่าง: Google DNS (จริงๆ ใช้ database IP)
  ports:
  - port: 80
    protocol: TCP
EOF

kubectl apply -f /tmp/external-service.yaml

# ดู Service และ Endpoints
kubectl get service external-api -n service-workshop
kubectl get endpoints external-api -n service-workshop

# Cleanup Lab 6
kubectl delete -f /tmp/external-service.yaml
```

### Lab 7: สร้าง Full Application Stack

```bash
# สร้าง Full Stack: Frontend + Backend + Database

cat <<'EOF' > /tmp/full-stack.yaml
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: service-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args:
        - "-text=API Response from $(POD_NAME)"
        - "-listen=:5678"
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 5678
---
# Backend Service (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: service-workshop
spec:
  type: ClusterIP
  selector:
    app: api-backend
  ports:
  - port: 8080
    targetPort: 5678
---
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  namespace: service-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.25
        ports:
        - containerPort: 80
---
# Frontend Service (NodePort - external access)
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: service-workshop
spec:
  type: NodePort
  selector:
    app: web-frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30090
EOF

kubectl apply -f /tmp/full-stack.yaml

# ดู Services ทั้งหมด
kubectl get services -n service-workshop
kubectl get endpoints -n service-workshop

# ทดสอบ internal communication
kubectl run curl-test \
    --image=curlimages/curl:latest \
    --namespace=service-workshop \
    --restart=Never \
    -- sleep 3600

kubectl wait --for=condition=Ready pod/curl-test -n service-workshop --timeout=60s

# Frontend เรียก Backend ผ่าน ClusterIP
kubectl exec -n service-workshop curl-test -- curl -s http://api-service:8080

# Cleanup
kubectl delete pod curl-test -n service-workshop
kubectl delete -f /tmp/full-stack.yaml
```

### Cleanup Workshop

```bash
# ลบ namespace ทั้งหมด
kubectl delete namespace service-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/backend-*.yaml /tmp/webapp-*.yaml \
    /tmp/stateful-headless.yaml /tmp/external-service.yaml \
    /tmp/full-stack.yaml
```

### Service Troubleshooting

```bash
# 1. ตรวจสอบว่า Service มี Endpoints
kubectl get endpoints <service-name>
# ถ้าไม่มี Endpoints = selector ไม่ match Pod labels

# 2. ตรวจสอบ Pod labels
kubectl get pods --show-labels
kubectl get pods -l app=my-app    # ตรวจสอบ selector

# 3. ตรวจสอบ Port ใน Service vs Container
kubectl describe service my-service | grep -A5 "Port:"
kubectl describe pod my-pod | grep -A5 "Ports:"

# 4. ทดสอบ DNS
kubectl run dns-debug --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec dns-debug -- nslookup <service-name>
kubectl exec dns-debug -- cat /etc/resolv.conf

# 5. ทดสอบ connectivity
kubectl exec dns-debug -- wget -qO- http://<service-name>:<port>

# 6. ดู kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# 7. ตรวจสอบ iptables rules (บน Node)
# sudo iptables -t nat -L KUBE-SERVICES | grep <service-name>
```

---

## สรุป

Services เป็น abstraction ที่สำคัญมากใน Kubernetes ที่ทำให้:

1. **Stable Endpoint**: IP ไม่เปลี่ยนแม้ Pods จะ restart
2. **Load Balancing**: กระจาย traffic ไปยัง Pods หลายตัว
3. **Service Discovery**: ค้นหา Services ได้ผ่าน DNS
4. **Multiple Exposure Methods**: ClusterIP, NodePort, LoadBalancer ตามความต้องการ

**เลือก Service Type:**
- **ClusterIP**: สำหรับ Internal microservices
- **NodePort**: สำหรับ Dev/Testing
- **LoadBalancer**: สำหรับ Production บน Cloud
- **ExternalName**: สำหรับ external services
- **Headless**: สำหรับ StatefulSets หรือ custom service discovery

ในบทต่อไปเราจะเรียนรู้ **Namespaces** ซึ่งเป็นวิธีจัดระเบียบ Resources ใน Kubernetes Cluster
