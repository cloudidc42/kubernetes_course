# Part 32: ClusterIP Service

## สารบัญ
1. [ClusterIP Service คืออะไร](#clusterip-overview)
2. [DNS-based Service Discovery](#dns-discovery)
3. [Endpoints และ EndpointSlices](#endpoints)
4. [iptables และ ipvs](#iptables-ipvs)
5. [Workshop: Internal Service Communication](#workshop)

---

## 1. ClusterIP Service คืออะไร {#clusterip-overview}

### ภาพรวม

ClusterIP คือ Service type เริ่มต้นใน Kubernetes ที่ให้ IP address แบบ Virtual ที่เข้าถึงได้จากภายใน Cluster เท่านั้น

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌──────────┐    ClusterIP: 10.96.100.1               │
│  │  Client  │───────────────────────┐                  │
│  │  Pod     │                       │                  │
│  └──────────┘                       ▼                  │
│                           ┌─────────────────┐          │
│                           │    Service      │          │
│                           │ my-service      │          │
│                           │ ClusterIP:      │          │
│                           │ 10.96.100.1:80  │          │
│                           └────────┬────────┘          │
│                                    │ Load Balance       │
│                       ┌────────────┼────────────┐      │
│                       ▼            ▼            ▼      │
│                 ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│                 │  Pod 1   │ │  Pod 2   │ │  Pod 3   │ │
│                 │10.244.1.2│ │10.244.1.3│ │10.244.2.2│ │
│                 └──────────┘ └──────────┘ └──────────┘ │
│                                                          │
│  ⚠️  ClusterIP ไม่สามารถเข้าถึงจากภายนอก Cluster ได้   │
└─────────────────────────────────────────────────────────┘
```

### สร้าง ClusterIP Service

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
  labels:
    app: my-app
  annotations:
    description: "ClusterIP Service สำหรับ my-app"
spec:
  type: ClusterIP  # Default type - ไม่จำเป็นต้องระบุ
  selector:
    app: my-app    # เลือก Pods ที่มี label app=my-app
  ports:
  - name: http
    protocol: TCP
    port: 80       # Port ที่ Service รับ
    targetPort: 8080  # Port ที่ Container รับ
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  # clusterIP: 10.96.100.1  # กำหนดเองได้ (ปกติไม่จำเป็น)
  sessionAffinity: None  # หรือ ClientIP
```

```bash
# Apply service
kubectl apply -f clusterip-service.yaml

# ดู service
kubectl get service my-service
kubectl describe service my-service
```

### ผลลัพธ์ที่ได้

```
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
my-service   ClusterIP   10.96.100.1   <none>        80/TCP,443/TCP   5s
```

### ClusterIP "None" (Headless Service)

Headless Service ใช้สำหรับ StatefulSets หรือเมื่อต้องการ DNS records สำหรับแต่ละ Pod โดยตรง

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # Headless - ไม่มี Virtual IP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

```
Headless Service DNS Resolution:
headless-service.default.svc.cluster.local
  ├── 10.244.1.2 (Pod 1)
  ├── 10.244.1.3 (Pod 2)
  └── 10.244.2.2 (Pod 3)
  
(Returns all Pod IPs - not a single VIP)

Regular ClusterIP DNS Resolution:
my-service.default.svc.cluster.local
  └── 10.96.100.1 (Single VIP)
```

---

## 2. DNS-based Service Discovery {#dns-discovery}

### DNS ใน Kubernetes

ทุก Service ที่สร้างใน Kubernetes จะได้รับ DNS record โดยอัตโนมัติผ่าน CoreDNS

```
DNS Format:
<service-name>.<namespace>.svc.<cluster-domain>

ตัวอย่าง:
my-service.default.svc.cluster.local
my-service.production.svc.cluster.local
```

```
DNS Resolution Flow:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Pod requests: my-service.default.svc.cluster.local     │
│                           │                             │
│                           ▼                             │
│              ┌────────────────────────┐                 │
│              │  /etc/resolv.conf      │                 │
│              │  nameserver 10.96.0.10 │                 │
│              │  search default.svc... │                 │
│              └────────────┬───────────┘                 │
│                           │                             │
│                           ▼                             │
│              ┌────────────────────────┐                 │
│              │  CoreDNS (10.96.0.10) │                 │
│              │  Pod: coredns-xxx      │                 │
│              └────────────┬───────────┘                 │
│                           │                             │
│                           ▼                             │
│              Returns: 10.96.100.1 (ClusterIP)           │
└──────────────────────────────────────────────────────────┘
```

### DNS Record Types

```bash
# A Record - ชี้ไปที่ ClusterIP
my-service.default.svc.cluster.local  A  10.96.100.1

# SRV Record - ระบุ port
_http._tcp.my-service.default.svc.cluster.local SRV 0 100 80 my-service.default.svc.cluster.local

# PTR Record - Reverse lookup
1.100.96.10.in-addr.arpa  PTR my-service.default.svc.cluster.local
```

### ทดสอบ DNS Resolution

```bash
# รัน Pod สำหรับทดสอบ DNS
kubectl run dns-test --image=nicolaka/netshoot --rm -it -- bash

# ภายใน Pod:
# ดู resolv.conf
cat /etc/resolv.conf

# ทดสอบ DNS แบบต่างๆ
nslookup my-service
nslookup my-service.default
nslookup my-service.default.svc.cluster.local

# ใช้ dig
dig my-service.default.svc.cluster.local

# ใช้ host
host my-service.default.svc.cluster.local

# SRV record
dig SRV _http._tcp.my-service.default.svc.cluster.local

# ออกจาก Pod
exit
```

### Service Discovery Patterns

```yaml
# Pattern 1: ใช้ Service name เดียวกันใน Namespace เดียว
# Frontend ใน default namespace เรียก backend ใน default namespace
env:
- name: BACKEND_URL
  value: "http://backend-service:8080"

# Pattern 2: Cross-namespace communication
# Frontend ใน frontend namespace เรียก backend ใน backend namespace
env:
- name: BACKEND_URL
  value: "http://backend-service.backend-ns.svc.cluster.local:8080"

# Pattern 3: ใช้ Environment Variables (deprecated แต่ยังใช้งานได้)
# Kubernetes inject env vars โดยอัตโนมัติ
# MY_SERVICE_SERVICE_HOST=10.96.100.1
# MY_SERVICE_SERVICE_PORT=80
```

### Environment Variables ที่ Kubernetes สร้างให้

```bash
# ดู env vars ที่ Kubernetes สร้างให้ใน Pod
kubectl exec <pod-name> -- env | grep -i service

# ผลลัพธ์ที่ได้:
# MY_SERVICE_PORT=tcp://10.96.100.1:80
# MY_SERVICE_PORT_80_TCP=tcp://10.96.100.1:80
# MY_SERVICE_PORT_80_TCP_PROTO=tcp
# MY_SERVICE_PORT_80_TCP_PORT=80
# MY_SERVICE_PORT_80_TCP_ADDR=10.96.100.1
# MY_SERVICE_SERVICE_HOST=10.96.100.1
# MY_SERVICE_SERVICE_PORT=80
```

---

## 3. Endpoints และ EndpointSlices {#endpoints}

### Endpoints คืออะไร?

Endpoints คือ resource ที่เก็บ list ของ IP:Port ของ Pods ที่ Service ชี้ไป

```
Service my-service ──► Endpoints my-service
                         ├── 10.244.1.2:8080
                         ├── 10.244.1.3:8080
                         └── 10.244.2.2:8080
```

### ดู Endpoints

```bash
# ดู endpoints
kubectl get endpoints
kubectl get endpoints my-service

# ดูรายละเอียด
kubectl describe endpoints my-service

# ผลลัพธ์:
# Name:         my-service
# Namespace:    default
# Labels:       app=my-app
# Subsets:
#   Addresses:          10.244.1.2,10.244.1.3,10.244.2.2
#   NotReadyAddresses:  <none>
#   Ports:
#     Name  Port  Protocol
#     ----  ----  --------
#     http  8080  TCP
```

### Endpoints Lifecycle

```
Pod Created & Ready ──► kube-controller-manager adds to Endpoints
Pod Not Ready        ──► Removed from Endpoints (goes to NotReadyAddresses)
Pod Deleted          ──► Removed from Endpoints
```

### สร้าง Endpoints แบบ Manual (External Service)

ใช้สำหรับชี้ Service ไปที่ External Services (เช่น Database นอก Cluster)

```yaml
# external-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  ports:
  - port: 5432
    targetPort: 5432
  # ไม่มี selector - จะ manage Endpoints เอง
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-database  # ต้องชื่อเดียวกับ Service
subsets:
- addresses:
  - ip: 192.168.100.10  # IP ของ External DB Server
  - ip: 192.168.100.11  # IP ของ DB Replica
  ports:
  - port: 5432
    protocol: TCP
```

```bash
# Apply
kubectl apply -f external-service.yaml

# ทดสอบ
kubectl run test-pod --image=postgres:alpine --rm -it -- bash
# ภายใน Pod:
psql -h external-database -U postgres
```

### EndpointSlices (Kubernetes 1.17+)

EndpointSlices แก้ปัญหา scalability ของ Endpoints (Endpoints ปกติมี size จำกัด)

```yaml
# endpoint-slice.yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc123
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
- name: http
  appProtocol: http
  protocol: TCP
  port: 8080
endpoints:
- addresses:
  - "10.244.1.2"
  conditions:
    ready: true
    serving: true
    terminating: false
  targetRef:
    kind: Pod
    name: pod-1
    namespace: default
- addresses:
  - "10.244.1.3"
  conditions:
    ready: true
  targetRef:
    kind: Pod
    name: pod-2
    namespace: default
```

```bash
# ดู EndpointSlices
kubectl get endpointslices
kubectl describe endpointslice my-service-abc123
```

### Readiness Probe กับ Endpoints

```yaml
# pod-with-readiness.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        # Pod จะถูก add เข้า Endpoints เมื่อ readiness probe ผ่าน
        # Pod จะถูกลบจาก Endpoints เมื่อ readiness probe fail
```

---

## 4. iptables และ ipvs {#iptables-ipvs}

### kube-proxy คืออะไร?

kube-proxy เป็น component ที่ implement Service routing บนทุก Node โดยใช้ iptables หรือ ipvs

```
Architecture:
┌──────────────────────────────────────────────┐
│                  Node                         │
│                                               │
│  ┌──────────────────────────────────────┐    │
│  │  kube-proxy                          │    │
│  │  Watches: Services, Endpoints        │    │
│  │  Updates: iptables / ipvs rules      │    │
│  └──────────────────────────────────────┘    │
│                                               │
│  iptables/ipvs rules:                        │
│  ClusterIP:80 ──► DNAT ──► Pod IP:8080      │
│                                               │
└──────────────────────────────────────────────┘
```

### iptables Mode

```
Packet Flow (iptables mode):
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Client Pod ──► iptables ──► PREROUTING chain       │
│                      │                              │
│                      ▼                              │
│             KUBE-SERVICES chain                     │
│                      │                              │
│                      ▼                              │
│             KUBE-SVC-XXXX chain (per service)       │
│                      │                              │
│           ┌──────────┼──────────┐                  │
│           ▼          ▼          ▼                  │
│      KUBE-SEP-1  KUBE-SEP-2  KUBE-SEP-3           │
│    (Pod 1 IP)  (Pod 2 IP)  (Pod 3 IP)              │
│        │           │           │                   │
│        └───────────┴───────────┘                   │
│                    │                               │
│              DNAT to Pod IP                        │
│                                                    │
└────────────────────────────────────────────────────┘
```

```bash
# ดู iptables rules ที่ kube-proxy สร้าง
iptables -t nat -L KUBE-SERVICES -n

# ดู rules สำหรับ service เฉพาะ
iptables -t nat -L -n | grep -A 10 "KUBE-SVC"

# ตัวอย่างผลลัพธ์:
# Chain KUBE-SVC-XPGD46QRK7WJZT7O (1 references)
# target     prot opt source               destination
# KUBE-SEP-... all  --  0.0.0.0/0            0.0.0.0/0   /* default/my-service */ statistic mode random probability 0.33333333349
# KUBE-SEP-... all  --  0.0.0.0/0            0.0.0.0/0   /* default/my-service */ statistic mode random probability 0.50000000000
# KUBE-SEP-... all  --  0.0.0.0/0            0.0.0.0/0   /* default/my-service */
```

### ipvs Mode

ipvs (IP Virtual Server) ให้ performance ดีกว่า iptables สำหรับ Cluster ขนาดใหญ่

```
ipvs vs iptables:

iptables:
- Rules processed linearly O(n)
- Performance degrades with many services
- Good for small clusters (< 1000 services)

ipvs:
- Hash table lookup O(1)
- Better performance for large clusters
- Multiple load balancing algorithms
- Requires kernel module: ip_vs
```

```bash
# ตรวจสอบ kube-proxy mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# เปลี่ยน kube-proxy เป็น ipvs mode
kubectl edit configmap kube-proxy -n kube-system
# เปลี่ยน mode: "" เป็น mode: "ipvs"

# Restart kube-proxy pods
kubectl rollout restart daemonset kube-proxy -n kube-system

# ตรวจสอบ ipvs rules
ipvsadm -l -n

# ดู ipvs rules สำหรับ service เฉพาะ
ipvsadm -l -n --timeout
```

### Load Balancing Algorithms ใน ipvs

```yaml
# ConfigMap สำหรับ ipvs scheduler
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |-
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: ipvs
    ipvs:
      scheduler: "rr"  # round-robin (default)
      # ตัวเลือก: rr, lc, dh, sh, sed, nq
      # rr = Round Robin
      # lc = Least Connection
      # dh = Destination Hashing
      # sh = Source Hashing
      # sed = Shortest Expected Delay
      # nq = Never Queue
```

### Session Affinity

```yaml
# Service with session affinity
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 hour
```

---

## 5. Workshop: Internal Service Communication {#workshop}

### Setup: สร้าง Application Stack

```yaml
# app-stack.yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: service-demo
---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: service-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
        version: v1
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args:
        - "-text=Backend Pod: $(HOSTNAME)"
        ports:
        - containerPort: 5678
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        readinessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 5
          periodSeconds: 3
---
# Backend Service
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: service-demo
  labels:
    app: backend
spec:
  type: ClusterIP
  selector:
    app: backend
    tier: api
  ports:
  - name: http
    port: 80
    targetPort: 5678
    protocol: TCP
---
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: service-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/
      volumes:
      - name: nginx-config
        configMap:
          name: frontend-config
---
# Frontend ConfigMap (nginx config)
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: service-demo
data:
  default.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend-service:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
---
# Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: service-demo
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - name: http
    port: 80
    targetPort: 80
```

```bash
# Deploy application stack
kubectl apply -f app-stack.yaml

# ตรวจสอบ Deployment
kubectl get all -n service-demo

# รอจน Pods พร้อม
kubectl wait --for=condition=available deployment/backend -n service-demo --timeout=60s
kubectl wait --for=condition=available deployment/frontend -n service-demo --timeout=60s
```

### Workshop 1: ทดสอบ Service Discovery

```bash
# รัน test pod ใน namespace เดียวกัน
kubectl run test-client \
  --image=nicolaka/netshoot \
  --namespace=service-demo \
  --rm -it -- bash

# ภายใน test pod:
# ทดสอบ DNS resolution
nslookup backend-service
nslookup backend-service.service-demo
nslookup backend-service.service-demo.svc.cluster.local

# ทดสอบ HTTP connection
curl backend-service:80
curl http://backend-service.service-demo.svc.cluster.local

# ทดสอบหลายๆ ครั้งเพื่อดู load balancing
for i in {1..10}; do curl -s backend-service:80; echo; done

# ออกจาก Pod
exit
```

### Workshop 2: ตรวจสอบ Endpoints

```bash
# ดู Endpoints ของ service
kubectl get endpoints -n service-demo
kubectl describe endpoints backend-service -n service-demo

# ดู Pod IPs
kubectl get pods -n service-demo -o wide

# ตรวจสอบว่า Endpoint IPs ตรงกับ Pod IPs
kubectl get pods -n service-demo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# ทดสอบ scale pods และดูว่า Endpoints update
kubectl scale deployment backend --replicas=5 -n service-demo
kubectl get endpoints backend-service -n service-demo -w  # watch

# Scale down
kubectl scale deployment backend --replicas=2 -n service-demo
kubectl get endpoints backend-service -n service-demo
```

### Workshop 3: Test Readiness & Endpoints

```bash
# ดู pods พร้อมกับ status
kubectl get pods -n service-demo -o wide

# จำลอง Pod ที่ไม่ Ready
# (ปรับ readiness probe ให้ fail)
kubectl patch deployment backend -n service-demo --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path",
    "value": "/not-exist"
  }
]'

# รอสักครู่แล้วดู Endpoints - Pod ที่ไม่ Ready จะถูกลบออก
sleep 30
kubectl get endpoints backend-service -n service-demo
kubectl get pods -n service-demo

# แก้ไขกลับ
kubectl patch deployment backend -n service-demo --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path",
    "value": "/"
  }
]'
```

### Workshop 4: Cross-Namespace Communication

```yaml
# cross-namespace-test.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: other-namespace
---
apiVersion: v1
kind: Pod
metadata:
  name: cross-ns-client
  namespace: other-namespace
spec:
  containers:
  - name: client
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
```

```bash
kubectl apply -f cross-namespace-test.yaml

# ทดสอบ cross-namespace communication
kubectl exec -n other-namespace cross-ns-client -- \
  curl http://backend-service.service-demo.svc.cluster.local

# สั้นกว่า (แต่ต้องใช้ full DNS name)
kubectl exec -n other-namespace cross-ns-client -- \
  nslookup backend-service.service-demo
```

### Workshop 5: External Service (Manual Endpoints)

```yaml
# external-service-demo.yaml
# สมมติว่ามี external service ที่ IP 8.8.8.8:53 (Google DNS)
apiVersion: v1
kind: Service
metadata:
  name: external-dns
  namespace: service-demo
spec:
  ports:
  - port: 53
    targetPort: 53
    protocol: UDP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-dns
  namespace: service-demo
subsets:
- addresses:
  - ip: 8.8.8.8
  ports:
  - port: 53
    protocol: UDP
```

```bash
kubectl apply -f external-service-demo.yaml

# ทดสอบ external service ผ่าน ClusterIP
kubectl run dns-test -n service-demo --image=nicolaka/netshoot --rm -it -- bash
# nslookup google.com external-dns
# exit
```

### Workshop 6: Service Debugging

```bash
# ตรวจสอบ Service ที่มีปัญหา
# 1. ตรวจสอบ Service มีอยู่
kubectl get service backend-service -n service-demo

# 2. ตรวจสอบ selector ถูกต้อง
kubectl describe service backend-service -n service-demo

# 3. ตรวจสอบ Endpoints ไม่ว่าง
kubectl get endpoints backend-service -n service-demo

# 4. ตรวจสอบ Pod labels ตรงกับ selector
kubectl get pods -n service-demo --show-labels

# 5. ทดสอบ direct Pod connection (ข้าม Service)
POD_IP=$(kubectl get pod -n service-demo -l app=backend -o jsonpath='{.items[0].status.podIP}')
kubectl run test-pod -n service-demo --image=nicolaka/netshoot --rm -it -- curl $POD_IP:5678

# 6. ตรวจสอบ Network Policy (ถ้ามี)
kubectl get networkpolicies -n service-demo
```

### Workshop 7: Service Topology (Advanced)

```yaml
# service-topology.yaml - Kubernetes 1.21+
apiVersion: v1
kind: Service
metadata:
  name: topology-service
  namespace: service-demo
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
  # Prefer local node, then local zone, then any
  topologyKeys:
  - "kubernetes.io/hostname"      # Same node
  - "topology.kubernetes.io/zone" # Same zone
  - "*"                           # Any
```

### Cleanup

```bash
# ลบ resources ทั้งหมด
kubectl delete namespace service-demo
kubectl delete namespace other-namespace
```

---

## สรุปความแตกต่าง Service Types

```
Service Types:
                                        
ClusterIP (default)                     
  ├── Internal only                     
  ├── Virtual IP (VIP)                  
  ├── Load balanced across Pods         
  └── Use: Internal services            
                                        
Headless (clusterIP: None)              
  ├── No VIP                            
  ├── DNS returns all Pod IPs           
  ├── Direct Pod access                 
  └── Use: StatefulSets, direct routing 
                                        
NodePort (Next Part)                    
  └── External access via Node IP:Port  
                                        
LoadBalancer (Later Part)               
  └── External access via Cloud LB      
                                        
ExternalName                            
  ├── CNAME to external DNS             
  └── Use: External services            
```

### ExternalName Service

```yaml
# external-name-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: default
spec:
  type: ExternalName
  externalName: api.example.com  # CNAME target
  # DNS resolution: external-api.default.svc.cluster.local → api.example.com
```

---

## Cheat Sheet

```bash
# CRUD Services
kubectl create service clusterip my-service --tcp=80:8080
kubectl get service
kubectl describe service my-service
kubectl delete service my-service

# ดู Endpoints
kubectl get endpoints
kubectl get ep my-service  # short form

# ดู EndpointSlices
kubectl get endpointslices

# Debug Service
kubectl run tmp --image=nicolaka/netshoot --rm -it -- bash
# nslookup <service-name>
# curl http://<service-name>:<port>

# Port Forward (สำหรับ test)
kubectl port-forward service/my-service 8080:80

# ดู kube-proxy mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# iptables rules
iptables -t nat -L KUBE-SERVICES -n --line-numbers
iptables -t nat -L KUBE-SVC-XXXX -n

# ipvs rules
ipvsadm -l -n
ipvsadm -l -n --stats
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes Service Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes DNS Documentation](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [kube-proxy Documentation](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

---

*ก่อนหน้า: [Part 31 - Kubernetes Networking](./part-31-k8s-networking.md)*
*ต่อไป: [Part 33 - NodePort Service](./part-33-nodeport.md)*

---

## 6. Headless Service {#headless-service}

### 6.1 Headless Service คืออะไร

Headless Service คือ ClusterIP Service ที่ตั้งค่า `clusterIP: None` ทำให้ไม่มี Virtual IP แต่ใช้ DNS resolution โดยตรงไปยัง Pod IPs

```
ClusterIP Service (ปกติ):                Headless Service:
                                          
Client ──► 10.96.100.1 (VIP)            Client ──► DNS Query
           │                                         │
           ▼ (load balanced)                         ▼ (returns all Pod IPs)
     Pod1, Pod2, Pod3                        [10.0.1.2, 10.0.1.3, 10.0.1.4]
                                              Client เลือก Pod เอง
```

### 6.2 เมื่อไหร่ควรใช้ Headless Service

1. **StatefulSets** - ต้องการเข้าถึง Pod แต่ละตัวโดยตรง
2. **Databases** - MySQL, MongoDB, Cassandra cluster
3. **Service Discovery แบบ custom** - Client ทำ load balancing เอง
4. **gRPC** - ต้องการ connection ระยะยาวกับแต่ละ pod

### 6.3 สร้าง Headless Service

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: database
spec:
  clusterIP: None           # ← นี่คือ Headless Service
  selector:
    app: mysql
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306

---
# StatefulSet ที่ใช้ Headless Service
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database
spec:
  serviceName: "mysql-headless"    # ← ชี้ไปที่ Headless Service
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
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
          name: mysql
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

### 6.4 DNS Records สำหรับ Headless Service

```bash
# ดู DNS records สำหรับ Headless Service
kubectl run dns-test --rm -it --image=nicolaka/netshoot -- bash

# ภายใน pod:
# Headless Service - returns ทุก Pod IP
nslookup mysql-headless.database.svc.cluster.local
# Output:
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# Name:      mysql-headless.database.svc.cluster.local
# Address 1: 10.244.1.5 mysql-0.mysql-headless.database.svc.cluster.local
# Address 2: 10.244.2.6 mysql-1.mysql-headless.database.svc.cluster.local
# Address 3: 10.244.3.7 mysql-2.mysql-headless.database.svc.cluster.local

# เข้าถึง Pod แต่ละตัวโดยตรง (StatefulSet)
nslookup mysql-0.mysql-headless.database.svc.cluster.local
# → 10.244.1.5

nslookup mysql-1.mysql-headless.database.svc.cluster.local
# → 10.244.2.6

# Pattern: <pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

### 6.5 Headless Service สำหรับ Kafka Cluster

```yaml
# kafka-headless.yaml - ตัวอย่าง Production-grade Kafka
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: messaging
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - name: kafka-internal
    port: 9092
    targetPort: 9092
  - name: kafka-controller
    port: 9093
    targetPort: 9093

---
# kafka-service.yaml - Regular service สำหรับ external access
apiVersion: v1
kind: Service
metadata:
  name: kafka
  namespace: messaging
spec:
  selector:
    app: kafka
  ports:
  - name: kafka
    port: 9092
    targetPort: 9092

---
# kafka-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: messaging
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        env:
        - name: KAFKA_NODE_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka-headless.messaging.svc.cluster.local:9092"
        - name: KAFKA_CONTROLLER_QUORUM_VOTERS
          value: "0@kafka-0.kafka-headless.messaging.svc.cluster.local:9093,1@kafka-1.kafka-headless.messaging.svc.cluster.local:9093,2@kafka-2.kafka-headless.messaging.svc.cluster.local:9093"
        ports:
        - containerPort: 9092
          name: kafka
        - containerPort: 9093
          name: controller
        volumeMounts:
        - name: data
          mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
```

---

## 7. Session Affinity {#session-affinity}

### 7.1 Session Affinity คืออะไร

Session Affinity (หรือ Sticky Sessions) ทำให้ requests จาก client เดิมถูกส่งไป Pod เดิมเสมอ

```
ปกติ (No Affinity):           Session Affinity:
                                
Client A ──► Pod 1             Client A ──► Pod 1 (เสมอ)
Client A ──► Pod 2             Client A ──► Pod 1 (เสมอ)
Client A ──► Pod 3             Client A ──► Pod 1 (เสมอ)
                                
Client B ──► Pod 1             Client B ──► Pod 2 (เสมอ)
Client B ──► Pod 3             Client B ──► Pod 2 (เสมอ)
```

### 7.2 ประเภทของ Session Affinity

Kubernetes รองรับ 2 ประเภท:
1. **None** (default) - ไม่มี affinity
2. **ClientIP** - hash จาก Client IP

```yaml
# session-affinity-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: my-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800    # 3 ชั่วโมง (default: 10800)
  ports:
  - port: 80
    targetPort: 8080

---
# ปิด Session Affinity
apiVersion: v1
kind: Service
metadata:
  name: no-sticky-service
spec:
  selector:
    app: my-app
  sessionAffinity: None        # default
  ports:
  - port: 80
    targetPort: 8080
```

### 7.3 ทดสอบ Session Affinity

```bash
# สร้าง deployment ที่แสดง hostname
kubectl create deployment web --image=nginx --replicas=3
# Patch เพื่อแสดง pod name
kubectl patch deployment web --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/command","value":["/bin/sh","-c"]},
  {"op":"add","path":"/spec/template/spec/containers/0/args","value":["echo \"Pod: $HOSTNAME\" > /usr/share/nginx/html/index.html && nginx -g \"daemon off;\""]}
]'

# สร้าง service ปกติ
kubectl expose deployment web --port=80 --name=web-no-affinity

# สร้าง service พร้อม affinity
kubectl expose deployment web --port=80 --name=web-affinity
kubectl patch service web-affinity -p '{"spec":{"sessionAffinity":"ClientIP"}}'

# ทดสอบ - ไม่มี affinity: ได้ pod ต่างกัน
for i in $(seq 1 10); do
  kubectl run test-$i --rm -it --image=curlimages/curl --restart=Never \
    -- curl -s web-no-affinity.default.svc.cluster.local 2>/dev/null
done

# ทดสอบ - มี affinity: ได้ pod เดิมเสมอ (จาก IP เดิม)
kubectl run sticky-test --rm -it --image=curlimages/curl --restart=Never \
  -- sh -c 'for i in 1 2 3 4 5; do curl -s web-affinity.default.svc.cluster.local; echo; done'
```

### 7.4 Session Affinity กับ kube-proxy

```bash
# kube-proxy mode iptables - Session Affinity ใช้ recent module
iptables -t nat -L KUBE-SVC-XXXX -n
# Output จะมี: recent --name KUBE-SVC-XXXX --rcheck --seconds 10800

# kube-proxy mode IPVS - Session Affinity ใช้ persistence
ipvsadm -l -n | grep -A5 "my-service"
# Output จะมี: persistent 10800

# ดู IPVS persistence
ipvsadm -l -n -p
```

---

## 8. EndpointSlices เชิงลึก {#endpointslices-deep}

### 8.1 ปัญหาของ Endpoints แบบเดิม

```
Cluster ใหญ่ (1000+ nodes, 10000+ pods):

Traditional Endpoints:
- 1 Service = 1 Endpoints object
- Object อาจมีขนาด 1MB+ (10000 IPs)
- ทุก kube-proxy ต้องรับ update ทั้งหมดเมื่อมี 1 Pod เปลี่ยน
- Network bandwidth ที่ใช้: 10000 * 1MB = 10GB ต่อ update!

EndpointSlices:
- 1 Service = หลาย EndpointSlice objects
- แต่ละ Slice มีสูงสุด 100 endpoints
- เมื่อ Pod เปลี่ยน → อัพเดทเฉพาะ Slice ที่เกี่ยวข้อง
- Network bandwidth ลดลงมาก
```

### 8.2 โครงสร้าง EndpointSlice

```yaml
# ตัวอย่าง EndpointSlice object
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-xxxx           # สร้างโดย controller อัตโนมัติ
  namespace: default
  labels:
    kubernetes.io/service-name: my-service
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
  ownerReferences:
  - apiVersion: v1
    kind: Service
    name: my-service
    uid: "12345-abcde"
addressType: IPv4                  # IPv4, IPv6, หรือ FQDN
endpoints:
- addresses:
  - "10.244.1.5"                   # Pod IP
  conditions:
    ready: true
    serving: true
    terminating: false
  hostname: web-pod-1
  nodeName: node1
  targetRef:
    kind: Pod
    name: web-7d6d7d9fbb-abc12
    namespace: default
    uid: "67890-fghij"
- addresses:
  - "10.244.2.6"
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: node2
  targetRef:
    kind: Pod
    name: web-7d6d7d9fbb-def34
    namespace: default
ports:
- name: http
  port: 8080
  protocol: TCP
- name: https
  port: 8443
  protocol: TCP
```

### 8.3 การทำงานของ EndpointSlices

```bash
# ดู EndpointSlices ทั้งหมด
kubectl get endpointslices --all-namespaces

# ดู EndpointSlices ของ service เฉพาะ
kubectl get endpointslices -l kubernetes.io/service-name=my-service

# ดู details
kubectl describe endpointslice my-service-xxxx

# ดู topology (node distribution)
kubectl get endpointslice -l kubernetes.io/service-name=my-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\t"}{.nodeName}{"\n"}{end}'

# นับ endpoints ทั้งหมด
kubectl get endpointslice -l kubernetes.io/service-name=my-service \
  -o jsonpath='{.items[*].endpoints}' | python3 -c "import sys,json; data=json.loads(sys.stdin.read()); print('Total endpoints:', len(data))"
```

### 8.4 Topology Aware Routing

```yaml
# service-topology.yaml - ส่ง traffic ไป pods ที่อยู่ใกล้
apiVersion: v1
kind: Service
metadata:
  name: topology-aware-service
  annotations:
    service.kubernetes.io/topology-mode: "Auto"    # Kubernetes 1.27+
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080

# EndpointSlice จะมี hints
# endpoints:
# - addresses: ["10.244.1.5"]
#   hints:
#     forZones:
#     - name: "us-east-1a"   # Route traffic จาก zone นี้ไปที่ endpoint นี้
#   nodeName: node1
#   zone: us-east-1a
```

```bash
# ตรวจสอบ topology hints
kubectl get endpointslice -l kubernetes.io/service-name=topology-aware-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\t"}{.hints}{"\n"}{end}'

# ดู zone ของ nodes
kubectl get nodes -L topology.kubernetes.io/zone
```

### 8.5 Dual-Stack EndpointSlices (IPv4 + IPv6)

```yaml
# dual-stack-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: dual-stack-service
spec:
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
  - IPv4
  - IPv6
  selector:
    app: dual-stack-app
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# ดู dual-stack endpoints
kubectl get endpointslices -l kubernetes.io/service-name=dual-stack-service
# จะมี 2 slices: addressType=IPv4 และ addressType=IPv6
```

---

## 9. แบบฝึกหัด ClusterIP Service {#exercises}

### แบบฝึกหัดที่ 1: สร้าง ClusterIP Service แบบต่างๆ

**วัตถุประสงค์:** เข้าใจความแตกต่างระหว่าง ClusterIP แบบปกติและ Headless

```bash
# Part A: ClusterIP ปกติ
kubectl create namespace svc-lab

kubectl create deployment web -n svc-lab --image=nginx --replicas=3
kubectl expose deployment web -n svc-lab --port=80 --name=web-clusterip

# ตรวจสอบ ClusterIP
kubectl get service web-clusterip -n svc-lab
kubectl get endpoints web-clusterip -n svc-lab

# ทดสอบ DNS
kubectl run test --rm -it -n svc-lab --image=nicolaka/netshoot -- \
  nslookup web-clusterip.svc-lab.svc.cluster.local

# Part B: Headless Service
kubectl expose deployment web -n svc-lab --port=80 --name=web-headless --cluster-ip=None

# ทดสอบ DNS (จะเห็น Pod IPs หลายตัว)
kubectl run test --rm -it -n svc-lab --image=nicolaka/netshoot -- \
  nslookup web-headless.svc-lab.svc.cluster.local

# เปรียบเทียบ
echo "=== ClusterIP (1 IP) ==="
kubectl get service web-clusterip -n svc-lab -o jsonpath='{.spec.clusterIP}'

echo "=== Headless (None) ==="
kubectl get service web-headless -n svc-lab -o jsonpath='{.spec.clusterIP}'

# Cleanup
kubectl delete namespace svc-lab
```

---

### แบบฝึกหัดที่ 2: ทดสอบ Session Affinity

**วัตถุประสงค์:** เห็นความแตกต่างระหว่าง service ที่มีและไม่มี affinity

```bash
kubectl create namespace affinity-lab

# สร้าง pods ที่แสดง hostname
kubectl apply -n affinity-lab -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args:
        - -text
        - "$(HOSTNAME)"
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 5678
EOF

# สร้าง 2 services
kubectl expose deployment echo-server -n affinity-lab \
  --port=5678 --name=echo-no-affinity

kubectl expose deployment echo-server -n affinity-lab \
  --port=5678 --name=echo-with-affinity

kubectl patch service echo-with-affinity -n affinity-lab \
  -p '{"spec":{"sessionAffinity":"ClientIP","sessionAffinityConfig":{"clientIP":{"timeoutSeconds":60}}}}'

# รอ pods ready
kubectl wait --for=condition=Ready pods --all -n affinity-lab --timeout=60s

# ทดสอบ no-affinity (ควรได้ pods ต่างกัน)
echo "=== No Affinity (ควรได้ pods ต่างกัน) ==="
for i in $(seq 1 6); do
  kubectl run curl-$i --rm -n affinity-lab --image=curlimages/curl \
    --restart=Never --quiet -- curl -s echo-no-affinity.affinity-lab.svc.cluster.local:5678 2>/dev/null &
done
wait

# ทดสอบ with-affinity (จาก pod เดิม ควรได้ pod เดิม)
echo "=== With Affinity (จาก client เดิม ควรได้ pod เดิม) ==="
kubectl run sticky-client --rm -it -n affinity-lab --image=curlimages/curl \
  --restart=Never -- \
  sh -c 'for i in 1 2 3 4 5; do curl -s echo-with-affinity.affinity-lab.svc.cluster.local:5678; echo; done'

# Cleanup
kubectl delete namespace affinity-lab
```

---

### แบบฝึกหัดที่ 3: สำรวจ EndpointSlices

**วัตถุประสงค์:** เข้าใจโครงสร้างและข้อมูลใน EndpointSlice

```bash
kubectl create namespace eps-lab

# สร้าง deployment ขนาดใหญ่
kubectl create deployment big-web -n eps-lab --image=nginx --replicas=10

kubectl expose deployment big-web -n eps-lab --port=80 --name=big-service

# รอ pods ready
kubectl wait --for=condition=Ready pods --all -n eps-lab --timeout=120s

# ตรวจสอบ endpoints แบบเดิม
kubectl get endpoints big-service -n eps-lab

# ตรวจสอบ EndpointSlices
kubectl get endpointslices -n eps-lab -l kubernetes.io/service-name=big-service

# ดู details ของ slice
SLICE=$(kubectl get endpointslices -n eps-lab -l kubernetes.io/service-name=big-service -o jsonpath='{.items[0].metadata.name}')
kubectl describe endpointslice $SLICE -n eps-lab

# ดู distribution across nodes
kubectl get endpointslice -n eps-lab -l kubernetes.io/service-name=big-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\t"}{.nodeName}{"\n"}{end}'

# เปรียบเทียบ size
echo "Endpoints object size:"
kubectl get endpoints big-service -n eps-lab -o json | wc -c

echo "EndpointSlice total size:"
kubectl get endpointslices -n eps-lab -l kubernetes.io/service-name=big-service -o json | wc -c

# Cleanup
kubectl delete namespace eps-lab
```

---

## 10. เฉลยแบบฝึกหัด {#answers}

### เฉลยแบบฝึกหัดที่ 1

```bash
# ผลที่คาดหวัง:

# ClusterIP ปกติ:
# NAME            TYPE        CLUSTER-IP      PORT(S)
# web-clusterip   ClusterIP   10.96.150.100   80/TCP

# DNS resolution → 1 IP (ClusterIP VIP):
# Name:    web-clusterip.svc-lab.svc.cluster.local
# Address: 10.96.150.100

# Headless Service:
# NAME          TYPE        CLUSTER-IP   PORT(S)
# web-headless  ClusterIP   None         80/TCP

# DNS resolution → หลาย IPs (Pod IPs โดยตรง):
# Name:      web-headless.svc-lab.svc.cluster.local
# Address 1: 10.244.1.5
# Address 2: 10.244.2.6
# Address 3: 10.244.3.7

# ข้อสรุป:
# - ClusterIP ปกติ: มี VIP เดียว, kube-proxy ทำ load balancing
# - Headless: ไม่มี VIP, DNS ให้ Pod IPs โดยตรง, client เลือก Pod เอง
```

### เฉลยแบบฝึกหัดที่ 2

```
ผลที่คาดหวัง:

No Affinity:
echo-server-abc12  (request 1)
echo-server-def34  (request 2) ← ต่างกัน
echo-server-ghi56  (request 3) ← ต่างกัน
echo-server-abc12  (request 4)
echo-server-def34  (request 5)

With Affinity (จาก client IP เดิม):
echo-server-abc12  (request 1)
echo-server-abc12  (request 2) ← เดิมเสมอ
echo-server-abc12  (request 3) ← เดิมเสมอ
echo-server-abc12  (request 4) ← เดิมเสมอ
echo-server-abc12  (request 5) ← เดิมเสมอ

หมายเหตุ: Session Affinity ใน Kubernetes ใช้ Client IP เป็น hash key
ถ้าต้องการ cookie-based affinity → ใช้ Ingress Controller
```

### เฉลยแบบฝึกหัดที่ 3

```
ผลที่คาดหวัง:

EndpointSlice listing (10 replicas):
NAME                     ADDRESSTYPE   PORTS   ENDPOINTS   AGE
big-service-xxxx         IPv4          80      5           1m
big-service-yyyy         IPv4          80      5           1m
(หรือ 1 slice ถ้า endpoints < 100)

แต่ละ slice มีสูงสุด 100 endpoints (default MaxEndpointsPerSlice)

Distribution ตัวอย่าง:
10.244.0.5    node1
10.244.0.6    node1
10.244.1.7    node2
10.244.1.8    node2
10.244.2.9    node3
...

ข้อสังเกต:
- Endpoints object เก็บทุก IP ใน 1 object (ขนาดใหญ่กว่า)
- EndpointSlices แบ่งเป็นหลาย objects (อัพเดทได้แม่นยำกว่า)
- สำหรับ 10 pods อาจใช้แค่ 1 slice เพราะ < 100 limit
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes Service Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes DNS Documentation](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [kube-proxy Documentation](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)

---

*ก่อนหน้า: [Part 31 - Kubernetes Networking](./part-31-k8s-networking.md)*
*ต่อไป: [Part 33 - NodePort Service](./part-33-nodeport.md)*
