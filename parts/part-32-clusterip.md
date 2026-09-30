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
