# Part 33: NodePort Service

## สารบัญ
1. [NodePort Service คืออะไร](#nodeport-overview)
2. [Port Range และการกำหนดค่า](#port-range)
3. [NodePort Use Cases](#use-cases)
4. [ข้อดีข้อเสียของ NodePort](#pros-cons)
5. [Workshop: Expose Service ผ่าน NodePort](#workshop)

---

## 1. NodePort Service คืออะไร {#nodeport-overview}

### ภาพรวม

NodePort คือ Service type ที่เปิด port บนทุก Node ใน Cluster เพื่อให้ Traffic จากภายนอกสามารถเข้าถึงได้

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  Internet ──► Node1:30080 ──┐                              │
│              Node2:30080 ──┼──► Service ──► Pods           │
│              Node3:30080 ──┘                               │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Node 1 (192.168.1.10)                                │  │
│  │                                                       │  │
│  │  ┌──────────────────────────────────┐               │  │
│  │  │  kube-proxy                       │               │  │
│  │  │  0.0.0.0:30080 ──► DNAT ──► Pod  │               │  │
│  │  └──────────────────────────────────┘               │  │
│  │                                                       │  │
│  │  Pod A (10.244.1.2:8080)                             │  │
│  │  Pod B (10.244.1.3:8080)                             │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Node 2 (192.168.1.11)                                │  │
│  │  Pod C (10.244.2.2:8080)                             │  │
│  │  ⚠️ Port 30080 เปิดอยู่แม้ไม่มี Pod บน Node นี้     │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Traffic Flow ของ NodePort

```
Client (External)
       │
       │ HTTP Request to 192.168.1.10:30080
       ▼
┌─────────────────────────────────────────────────┐
│  Node 1 (192.168.1.10)                          │
│                                                   │
│  iptables PREROUTING                             │
│       │                                          │
│       ▼                                          │
│  KUBE-NODEPORTS chain                           │
│       │                                          │
│       ▼                                          │
│  KUBE-SVC-XXXX (random pod selection)           │
│       │                                          │
│  ┌────┼────┐                                    │
│  ▼    ▼    ▼                                    │
│ Pod1 Pod2 Pod3  (may be on other nodes)         │
│                                                   │
│  DNAT: 192.168.1.10:30080 → 10.244.X.X:8080   │
└─────────────────────────────────────────────────┘
```

### สร้าง NodePort Service

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
  namespace: default
  labels:
    app: web-app
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80          # ClusterIP port (internal)
    targetPort: 8080  # Container port
    nodePort: 30080   # External port on each Node (30000-32767)
                      # ถ้าไม่ระบุ Kubernetes จะเลือกอัตโนมัติ
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
    nodePort: 30443
```

```bash
# Apply service
kubectl apply -f nodeport-service.yaml

# ดู service
kubectl get service nodeport-service

# ผลลัพธ์:
# NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
# nodeport-service   NodePort   10.96.200.1    <none>        80:30080/TCP,443:30443/TCP   5s

# ทดสอบ access
curl http://<NODE_IP>:30080
curl http://192.168.1.10:30080  # ผ่าน Node 1
curl http://192.168.1.11:30080  # ผ่าน Node 2 (ได้ผลเหมือนกัน)
```

---

## 2. Port Range และการกำหนดค่า {#port-range}

### Default Port Range

```
NodePort Range: 30000 - 32767

เหตุผล:
- Port < 1024 ต้องการ root privileges
- Port 1024-29999 อาจชนกับ services อื่น
- 30000-32767 เป็น ephemeral range ที่ปลอดภัย
```

### เปลี่ยน NodePort Range

```bash
# แก้ไข kube-apiserver manifest
# บน Control Plane node
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# เพิ่ม/แก้ไข argument:
# --service-node-port-range=20000-30000

# ตัวอย่างใน yaml:
```

```yaml
# kube-apiserver.yaml (excerpt)
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.1.10
    - --allow-privileged=true
    - --service-cluster-ip-range=10.96.0.0/12
    - --service-node-port-range=30000-32767  # แก้ไขตรงนี้
    # ... other args
```

### Automatic vs Manual NodePort Assignment

```yaml
# Auto assignment (Kubernetes เลือกให้)
apiVersion: v1
kind: Service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    # nodePort ไม่ระบุ → Kubernetes จะเลือกให้อัตโนมัติ

---
# Manual assignment (กำหนดเอง)
apiVersion: v1
kind: Service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # กำหนดเอง ต้องอยู่ใน range
```

### externalTrafficPolicy

```yaml
# Policy ที่ส่งผลต่อการ route traffic
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  externalTrafficPolicy: Local  # หรือ Cluster (default)
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

```
externalTrafficPolicy: Cluster (Default)
┌────────────────────────────────────────────────┐
│                                                │
│  Client ──► Node1:30080                       │
│                    │                           │
│              kube-proxy                        │
│                    │ (may forward to other node)│
│         ┌──────────┼──────────┐               │
│         ▼          ▼          ▼               │
│       Pod1       Pod2        Pod3             │
│    (Node1)     (Node2)     (Node3)            │
│                                                │
│  ✅ Load balancing ทุก Pod                    │
│  ❌ ไม่รู้ Client IP จริง (SNAT)              │
│  ❌ Extra hop อาจเพิ่ม latency               │
└────────────────────────────────────────────────┘

externalTrafficPolicy: Local
┌────────────────────────────────────────────────┐
│                                                │
│  Client ──► Node1:30080                       │
│                    │                           │
│              kube-proxy                        │
│                    │ (only local pods)         │
│         ┌──────────┘                          │
│         ▼                                     │
│       Pod1 (Node1 only)                       │
│                                                │
│  ✅ รักษา Client IP จริง (no SNAT)            │
│  ✅ ไม่มี extra hop                          │
│  ❌ Load balancing ไม่สม่ำเสมอ               │
│  ❌ Node ที่ไม่มี Pod จะ drop traffic         │
└────────────────────────────────────────────────┘
```

### ดู NodePort ที่ใช้งาน

```bash
# ดู ports ทั้งหมดที่ใช้งาน
kubectl get services --all-namespaces -o go-template='{{range .items}}{{if eq .spec.type "NodePort"}}{{.metadata.namespace}}/{{.metadata.name}}: {{range .spec.ports}}{{.nodePort}} {{end}}\n{{end}}{{end}}'

# ดูด้วย jsonpath
kubectl get services -o jsonpath='{range .items[?(@.spec.type=="NodePort")]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.nodePort}{" "}{end}{"\n"}{end}'
```

---

## 3. NodePort Use Cases {#use-cases}

### Use Case 1: Development Environment

```
Development:
┌─────────────────────────────────┐
│  Developer Machine              │
│                                 │
│  kubectl port-forward ...       │
│  OR                             │
│  curl http://localhost:30080    │
│                └──────────────► minikube/kind node:30080
└─────────────────────────────────┘
```

```yaml
# dev-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: dev-web
  namespace: development
  annotations:
    environment: "development"
    team: "frontend"
spec:
  type: NodePort
  selector:
    app: web-dev
  ports:
  - name: http
    port: 80
    targetPort: 3000  # React/Node dev server
    nodePort: 30300
  - name: debug
    port: 9229
    targetPort: 9229  # Node.js debugger
    nodePort: 30229
```

### Use Case 2: Simple On-Premise Deployment

```yaml
# on-premise-service.yaml
# บน On-premise ไม่มี Cloud LoadBalancer
# ใช้ NodePort + HAProxy/Nginx ด้านหน้า

apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
  - port: 443
    targetPort: 8443
    nodePort: 30443
```

```
HAProxy Configuration (on-premise):
frontend k8s_http
  bind *:80
  default_backend k8s_nodes

backend k8s_nodes
  balance roundrobin
  server node1 192.168.1.10:30080 check
  server node2 192.168.1.11:30080 check
  server node3 192.168.1.12:30080 check
```

### Use Case 3: Testing และ CI/CD

```bash
# ใน CI/CD Pipeline
# สร้าง test environment
kubectl apply -f test-deployment.yaml
kubectl apply -f test-service.yaml

# รอ Service พร้อม
kubectl wait --for=condition=available deployment/test-app --timeout=120s

# รัน integration tests
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NODE_PORT=$(kubectl get service test-service -o jsonpath='{.spec.ports[0].nodePort}')

# Run tests
pytest integration_tests/ --base-url=http://${NODE_IP}:${NODE_PORT}

# Cleanup
kubectl delete -f test-deployment.yaml
kubectl delete -f test-service.yaml
```

### Use Case 4: Monitoring Tools

```yaml
# monitoring-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-external
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
  - name: web
    port: 9090
    targetPort: 9090
    nodePort: 30090

---
apiVersion: v1
kind: Service
metadata:
  name: grafana-external
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - name: web
    port: 3000
    targetPort: 3000
    nodePort: 30300
```

---

## 4. ข้อดีข้อเสียของ NodePort {#pros-cons}

```
ข้อดี:
✅ ง่ายในการ setup
✅ ทำงานได้ทุก environment (Cloud, On-premise, Local)
✅ ไม่ต้องการ Cloud Load Balancer
✅ เหมาะสำหรับ Development และ Testing
✅ Port เข้าถึงได้จากทุก Node

ข้อเสีย:
❌ Port Range จำกัด (30000-32767)
❌ ต้องรู้ Node IP (อาจเปลี่ยนแปลง)
❌ ไม่มี built-in Load Balancing สำหรับ Nodes
❌ Security concerns (ports เปิดบน ALL nodes)
❌ ไม่เหมาะ Production (บน Cloud)
❌ ต้องการ Firewall rules เพิ่มเติม
```

### เมื่อไหรควรใช้ NodePort?

```
ใช้ NodePort เมื่อ:
├── Development/Testing environment
├── On-premise ที่ไม่มี Cloud LB
├── ต้องการ external access แบบง่ายๆ
├── ใช้ร่วมกับ external Load Balancer (HAProxy/Nginx)
└── Monitoring tools ที่ต้องการเข้าถึงจากภายนอก

ไม่ควรใช้ NodePort เมื่อ:
├── Production environment บน Cloud (ใช้ LoadBalancer แทน)
├── ต้องการ SSL termination
├── ต้องการ path-based routing (ใช้ Ingress แทน)
└── ต้องการ Port < 30000
```

---

## 5. Workshop: Expose Service ผ่าน NodePort {#workshop}

### Workshop 1: Basic NodePort

```yaml
# workshop-nodeport.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nodeport-demo
---
# Simple Web Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: nodeport-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init-html
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "<h1>Hello from $(hostname)</h1><p>Pod IP: $MY_POD_IP</p><p>Node: $MY_NODE_NAME</p>" > /html/index.html
        env:
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
# NodePort Service
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: nodeport-demo
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30880
```

```bash
# Deploy
kubectl apply -f workshop-nodeport.yaml

# ตรวจสอบ
kubectl get all -n nodeport-demo

# ดู Node IPs
kubectl get nodes -o wide

# ดู Service
kubectl get service -n nodeport-demo

# ทดสอบ access
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl http://${NODE_IP}:30880

# ทดสอบหลายๆ ครั้ง - จะเห็น response จาก Pods ต่างๆ
for i in {1..9}; do curl -s http://${NODE_IP}:30880; echo; done
```

### Workshop 2: NodePort กับ externalTrafficPolicy

```yaml
# traffic-policy-demo.yaml
# Cluster Policy (default)
apiVersion: v1
kind: Service
metadata:
  name: cluster-policy
  namespace: nodeport-demo
  annotations:
    description: "Cluster traffic policy - load balance across all pods"
spec:
  type: NodePort
  externalTrafficPolicy: Cluster
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30881
---
# Local Policy
apiVersion: v1
kind: Service
metadata:
  name: local-policy
  namespace: nodeport-demo
  annotations:
    description: "Local traffic policy - only local pods, preserves client IP"
spec:
  type: NodePort
  externalTrafficPolicy: Local
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30882
```

```bash
kubectl apply -f traffic-policy-demo.yaml

# ทดสอบ Cluster policy
echo "Testing Cluster Policy:"
for i in {1..6}; do curl -s http://${NODE_IP}:30881; echo; done

# ทดสอบ Local policy
echo "Testing Local Policy:"
for i in {1..6}; do curl -s http://${NODE_IP}:30882; echo; done
# หมายเหตุ: ถ้า Node นั้นไม่มี Pod จะ drop connection
```

### Workshop 3: Multi-Port NodePort

```yaml
# multi-port-nodeport.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-port-app
  namespace: nodeport-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multi-port-app
  template:
    metadata:
      labels:
        app: multi-port-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
        - containerPort: 8080
          name: admin
---
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
  namespace: nodeport-demo
spec:
  type: NodePort
  selector:
    app: multi-port-app
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30800
  - name: https
    port: 443
    targetPort: 443
    nodePort: 30443
  - name: admin
    port: 8080
    targetPort: 8080
    nodePort: 30808
```

```bash
kubectl apply -f multi-port-nodeport.yaml

# ดู Service
kubectl describe service multi-port-service -n nodeport-demo

# ทดสอบแต่ละ Port
curl http://${NODE_IP}:30800
curl http://${NODE_IP}:30808
```

### Workshop 4: ใช้ NodePort ร่วมกับ LoadBalancer บน-premise

```bash
# ติดตั้ง HAProxy (สมมติ)
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    maxconn 50000

defaults
    log global
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend k8s_http_front
    bind *:80
    default_backend k8s_http_back

backend k8s_http_back
    balance roundrobin
    option httpchk GET /
    server k8s-node1 192.168.1.10:30880 check
    server k8s-node2 192.168.1.11:30880 check
    server k8s-node3 192.168.1.12:30880 check
EOF

systemctl restart haproxy
```

### Workshop 5: NodePort Security

```yaml
# secure-nodeport.yaml
# NetworkPolicy เพื่อ control traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nodeport-allow-specific
  namespace: nodeport-demo
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  # อนุญาต traffic จาก Load Balancer IP range เท่านั้น
  - from:
    - ipBlock:
        cidr: 192.168.1.0/24  # Internal network
    ports:
    - protocol: TCP
      port: 80
```

```bash
# ตรวจสอบ open ports บน Node
# (ต้อง ssh เข้า Node)
ss -tlnp | grep 30880
netstat -tlnp | grep 30880

# Firewall rules (ถ้าใช้ iptables)
iptables -I INPUT -p tcp --dport 30880 -j ACCEPT
iptables -I INPUT -p tcp --dport 30881 -j ACCEPT
```

### Workshop 6: ทดสอบ Failover

```bash
# ทดสอบว่าเมื่อ Pod ล้มเหลว Service ยังทำงานได้
# ดู Pods ก่อน
kubectl get pods -n nodeport-demo -o wide

# ลบ Pod หนึ่งตัว
kubectl delete pod -n nodeport-demo -l app=web-app --field-selector=spec.nodeName=$(kubectl get nodes -o name | head -1 | cut -d/ -f2)

# ทดสอบว่า Service ยังทำงาน
for i in {1..10}; do curl -s http://${NODE_IP}:30880; echo; done

# ดู Pods อีกครั้ง - ReplicaSet จะสร้าง Pod ใหม่
kubectl get pods -n nodeport-demo -w
```

### Workshop 7: Port Forwarding vs NodePort

```bash
# Port Forward (Development only)
kubectl port-forward -n nodeport-demo service/web-nodeport 8080:80 &
curl http://localhost:8080

# NodePort (สำหรับ access จากภายนอก)
curl http://${NODE_IP}:30880

# ข้อแตกต่าง:
# Port Forward: ทำงานผ่าน kubectl proxy, ปิดเมื่อ kubectl หยุด
# NodePort: เปิดอยู่ตลอด, เข้าถึงได้โดยตรง
```

### Workshop 8: Debug NodePort Issues

```bash
# ปัญหา: ไม่สามารถเข้าถึง NodePort ได้

# 1. ตรวจสอบ Service
kubectl get service web-nodeport -n nodeport-demo
kubectl describe service web-nodeport -n nodeport-demo

# 2. ตรวจสอบ Endpoints
kubectl get endpoints web-nodeport -n nodeport-demo

# 3. ตรวจสอบ Pods Running
kubectl get pods -n nodeport-demo

# 4. ตรวจสอบ Port บน Node
# (ssh เข้า node)
ss -tlnp | grep 30880

# 5. ตรวจสอบ iptables rules
iptables -t nat -L -n | grep 30880

# 6. ตรวจสอบ Firewall
ufw status
firewall-cmd --list-ports

# 7. ทดสอบจาก Node เอง
curl localhost:30880

# 8. ตรวจสอบ Network Policy ที่อาจ block traffic
kubectl get networkpolicies -n nodeport-demo
```

### Cleanup

```bash
kubectl delete namespace nodeport-demo
```

---

## สรุป NodePort

```
NodePort Summary:
┌──────────────────────────────────────────────────────┐
│                                                      │
│  External Access:                                    │
│  http://<ANY_NODE_IP>:<NODE_PORT>                   │
│                                                      │
│  Port Range: 30000-32767                            │
│                                                      │
│  Components:                                         │
│  ├── ClusterIP (internal access)                    │
│  └── NodePort (external access on each node)        │
│                                                      │
│  Traffic Policies:                                   │
│  ├── Cluster: All pods, no client IP preservation   │
│  └── Local: Node-local pods only, preserves client IP│
│                                                      │
│  Best Practices:                                     │
│  ├── Use for Development/Testing                    │
│  ├── Combine with external LB for production        │
│  ├── Consider NetworkPolicy for security            │
│  └── Document all used NodePorts                    │
└──────────────────────────────────────────────────────┘
```

---

## Cheat Sheet

```bash
# สร้าง NodePort Service
kubectl expose deployment my-app --type=NodePort --port=80 --target-port=8080

# ดู NodePort
kubectl get service my-service
kubectl describe service my-service | grep NodePort

# ดู Node IPs
kubectl get nodes -o wide

# ทดสอบ NodePort
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NODE_PORT=$(kubectl get service my-service -o jsonpath='{.spec.ports[0].nodePort}')
curl http://${NODE_IP}:${NODE_PORT}

# Patch NodePort
kubectl patch service my-service -p '{"spec":{"ports":[{"port":80,"nodePort":30090,"targetPort":8080}]}}'

# เปลี่ยน type
kubectl patch service my-service -p '{"spec":{"type":"ClusterIP"}}'
```

---

*ก่อนหน้า: [Part 32 - ClusterIP Service](./part-32-clusterip.md)*
*ต่อไป: [Part 34 - LoadBalancer Service](./part-34-loadbalancer.md)*

---

## 6. NodePort สำหรับ Development vs Production {#dev-vs-prod}

### 6.1 การใช้ NodePort ใน Development

NodePort เหมาะมากสำหรับ development environment เพราะง่าย รวดเร็ว ไม่ต้องการ external infrastructure

```yaml
# dev-service.yaml - สำหรับ development
apiVersion: v1
kind: Service
metadata:
  name: api-service-dev
  namespace: development
  annotations:
    description: "Development only - DO NOT use in production"
spec:
  type: NodePort
  selector:
    app: api-server
    env: development
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    nodePort: 30080    # Fixed port สำหรับง่ายต่อการทดสอบ
  - name: debug
    port: 5005
    targetPort: 5005
    nodePort: 30005    # Java debug port
  - name: metrics
    port: 9090
    targetPort: 9090
    nodePort: 30090    # Prometheus metrics
```

```bash
# Development workflow
# 1. สร้าง service
kubectl apply -f dev-service.yaml -n development

# 2. หา Node IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# 3. ทดสอบ API
curl http://$NODE_IP:30080/api/health

# 4. เปิด debug port (Java remote debug)
# IDE: Connect to $NODE_IP:30005

# 5. ดู metrics
curl http://$NODE_IP:30090/metrics

# Port-forward ทางเลือก (ไม่ต้องใช้ NodePort)
kubectl port-forward service/api-service-dev 8080:8080 -n development
```

### 6.2 ปัญหาของ NodePort ใน Production

```
❌ ปัญหาที่พบใน Production:

1. Port Range จำกัด (30000-32767)
   - เพียง 2768 ports สำหรับทั้ง cluster
   - Services จำนวนมากอาจชน

2. Port ที่ใช้ยากจำ
   - api.example.com:31234 ไม่ user-friendly

3. ไม่มี SSL/TLS termination
   - ต้อง handle TLS ใน application เอง

4. Load Balancing ไม่เท่ากัน
   - Traffic ไปที่ node ใดก็แล้วแต่ DNS resolve
   - ถ้า node มี pods ไม่เท่ากัน → imbalanced

5. ความปลอดภัย
   - ทุก port เปิดบนทุก Node
   - ต้องมี Firewall rules ซับซ้อน

✅ แนะนำสำหรับ Production:
- LoadBalancer Service
- Ingress Controller
- API Gateway
```

### 6.3 Pattern: NodePort สำหรับ HA Setup

```yaml
# ha-nodeport.yaml - High Availability setup
apiVersion: v1
kind: Service
metadata:
  name: ha-web-service
  annotations:
    # External LB (เช่น HAProxy/Nginx) ชี้มาที่ NodePorts เหล่านี้
    external-lb: "haproxy-01.company.com"
spec:
  type: NodePort
  selector:
    app: web-server
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
  - port: 443
    targetPort: 8443
    nodePort: 30443
  externalTrafficPolicy: Local    # ดีกว่าสำหรับ HA

---
# สร้าง Deployment พร้อม Anti-Affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web-server
            topologyKey: "kubernetes.io/hostname"    # 1 pod per node
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 8080
        - containerPort: 8443
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

---

## 7. NodePort + External Load Balancer {#nodeport-with-lb}

### 7.1 Architecture: NodePort + HAProxy

```
┌─────────────────────────────────────────────────────────────────┐
│                    NodePort + External LB                        │
│                                                                   │
│  Internet                                                         │
│      │                                                            │
│      ▼                                                            │
│  ┌──────────────────────────────────────┐                        │
│  │  HAProxy / Nginx (External LB)       │                        │
│  │  IP: 203.0.113.10 (Public IP)        │                        │
│  │                                       │                        │
│  │  frontend myapp                       │                        │
│  │    bind :80                           │                        │
│  │    server node1 10.0.0.1:30080 check │                        │
│  │    server node2 10.0.0.2:30080 check │                        │
│  │    server node3 10.0.0.3:30080 check │                        │
│  └──────────────────────────────────────┘                        │
│              │           │           │                            │
│              ▼           ▼           ▼                            │
│  ┌────────┐  ┌────────┐  ┌────────┐                             │
│  │ Node 1 │  │ Node 2 │  │ Node 3 │                             │
│  │ :30080 │  │ :30080 │  │ :30080 │                             │
│  └────────┘  └────────┘  └────────┘                             │
│       │            │           │                                  │
│       └────────────┼───────────┘                                 │
│                    ▼                                              │
│               Kubernetes Service → Pods                           │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 HAProxy Configuration

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    chroot /var/lib/haproxy
    maxconn 50000
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    retries 3

# HTTP frontend
frontend k8s-http
    bind *:80
    default_backend k8s-nodes-http
    
    # Health check endpoint
    acl is_health path /health
    use_backend health-check if is_health

backend k8s-nodes-http
    balance roundrobin
    option httpchk GET /healthz
    
    # Kubernetes nodes
    server node1 10.0.0.1:30080 check inter 2s fall 3 rise 2 weight 100
    server node2 10.0.0.2:30080 check inter 2s fall 3 rise 2 weight 100
    server node3 10.0.0.3:30080 check inter 2s fall 3 rise 2 weight 100

# HTTPS frontend
frontend k8s-https
    bind *:443 ssl crt /etc/ssl/certs/myapp.pem
    default_backend k8s-nodes-https
    http-request set-header X-Forwarded-Proto https

backend k8s-nodes-https
    balance roundrobin
    server node1 10.0.0.1:30443 check ssl verify none
    server node2 10.0.0.2:30443 check ssl verify none
    server node3 10.0.0.3:30443 check ssl verify none

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
```

### 7.3 Nginx Configuration

```nginx
# /etc/nginx/conf.d/k8s-upstream.conf

upstream k8s_nodes {
    least_conn;                          # Least connections algorithm
    keepalive 32;                        # Connection pooling
    
    server 10.0.0.1:30080 weight=100 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:30080 weight=100 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:30080 weight=100 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name myapp.example.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com;
    
    ssl_certificate /etc/nginx/ssl/myapp.crt;
    ssl_certificate_key /etc/nginx/ssl/myapp.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location / {
        proxy_pass http://k8s_nodes;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        
        # Health check
        proxy_next_upstream error timeout http_500 http_502 http_503;
    }
    
    location /health {
        access_log off;
        return 200 "OK";
    }
}
```

### 7.4 Keepalived สำหรับ HA External LB

```
# /etc/keepalived/keepalived.conf - Virtual IP failover

vrrp_script check_nginx {
    script "/usr/bin/pgrep nginx"
    interval 2
    weight 2
    rise 2
    fall 3
}

vrrp_instance LB_MASTER {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 110             # Master มี priority สูงกว่า
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass your-secret-password
    }
    
    virtual_ipaddress {
        203.0.113.10/24      # VIP - ที่ DNS ชี้มา
    }
    
    track_script {
        check_nginx
    }
}
```

---

## 8. externalTrafficPolicy: Local vs Cluster {#external-traffic-policy}

### 8.1 ความแตกต่างหลัก

```
externalTrafficPolicy: Cluster (default)
┌─────────────────────────────────────────────┐
│                                              │
│  Client → Node1:30080                        │
│                │                             │
│                ▼ (kube-proxy)                │
│         DNAT + SNAT                          │
│                │                             │
│      ┌─────────┴─────────┐                  │
│      ▼                   ▼                  │
│   Pod on Node1        Pod on Node2           │
│   (source IP = NodeIP)                       │
│                                              │
│  ข้อดี: load distributed ทุก pods            │
│  ข้อเสีย: ไม่รู้ client IP จริง              │
└─────────────────────────────────────────────┘

externalTrafficPolicy: Local
┌─────────────────────────────────────────────┐
│                                              │
│  Client → Node1:30080                        │
│                │                             │
│                ▼ (kube-proxy)                │
│         DNAT only (NO SNAT)                  │
│                │                             │
│                ▼                             │
│         Pod on Node1 ONLY                    │
│         (source IP = real client IP!)        │
│                                              │
│  ข้อดี: รู้ client IP จริง                  │
│  ข้อเสีย: uneven load distribution          │
└─────────────────────────────────────────────┘
```

### 8.2 YAML Configuration

```yaml
# cluster-policy.yaml - default
apiVersion: v1
kind: Service
metadata:
  name: web-cluster-policy
spec:
  type: NodePort
  selector:
    app: web
  externalTrafficPolicy: Cluster    # default
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080

---
# local-policy.yaml - preserves client IP
apiVersion: v1
kind: Service
metadata:
  name: web-local-policy
spec:
  type: NodePort
  selector:
    app: web
  externalTrafficPolicy: Local      # ← เปลี่ยนตรงนี้
  healthCheckNodePort: 32000        # Port สำหรับ health check
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

### 8.3 ทดสอบ externalTrafficPolicy

```bash
# สร้าง deployment ที่ log client IP
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-ip
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo-ip
  template:
    metadata:
      labels:
        app: echo-ip
    spec:
      containers:
      - name: echo
        image: k8s.gcr.io/echoserver:1.4
        ports:
        - containerPort: 8080
EOF

# Service Cluster mode
kubectl expose deployment echo-ip \
  --type=NodePort \
  --port=80 \
  --target-port=8080 \
  --name=echo-cluster

# Service Local mode
kubectl expose deployment echo-ip \
  --type=NodePort \
  --port=80 \
  --target-port=8080 \
  --name=echo-local

kubectl patch service echo-local \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'

# หา NodePorts
CLUSTER_PORT=$(kubectl get service echo-cluster -o jsonpath='{.spec.ports[0].nodePort}')
LOCAL_PORT=$(kubectl get service echo-local -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

echo "Cluster port: $CLUSTER_PORT"
echo "Local port: $LOCAL_PORT"

# ทดสอบจาก client ภายนอก
# curl http://$NODE_IP:$CLUSTER_PORT
# → client_address=10.0.0.1 (Node IP - SNAT'd)

# curl http://$NODE_IP:$LOCAL_PORT
# → client_address=203.0.113.5 (Real client IP!)
```

### 8.4 Health Check Node Port

เมื่อใช้ `externalTrafficPolicy: Local` จะมี Health Check Node Port ที่ External LB ใช้ตรวจสอบว่า Node นั้นมี Pod อยู่หรือเปล่า

```yaml
# service-with-healthcheck.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  externalTrafficPolicy: Local
  healthCheckNodePort: 31500      # External LB check port นี้
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

```bash
# ทดสอบ healthCheckNodePort
NODE_IP="10.0.0.1"
HEALTH_PORT=31500

# ถ้า node มี Pod → HTTP 200
curl http://$NODE_IP:$HEALTH_PORT
# {"localEndpoints": 1}

# ถ้า node ไม่มี Pod → HTTP 503
# {"localEndpoints": 0}

# HAProxy ใช้ healthCheckNodePort
# server node1 10.0.0.1:30080 check port 31500 inter 5s
```

---

## 9. Security Considerations {#security}

### 9.1 ความเสี่ยงด้านความปลอดภัยของ NodePort

```
ความเสี่ยงหลัก:

1. Wide Attack Surface
   - ทุก node เปิด port เดียวกัน
   - ถ้า 1 node ถูก compromise → attacker ใช้ NodePort เข้า cluster ได้

2. Port Scanning
   - ง่ายต่อการ scan ports 30000-32767
   - attacker รู้ว่ามี service อะไรบ้าง

3. DDoS
   - traffic ตรงมาที่ node IP โดยตรง
   - ไม่มี Firewall/WAF กรอง

4. No Authentication by Default
   - ใครก็เข้าถึงได้ถ้ารู้ IP:Port
```

### 9.2 Firewall Rules สำหรับ NodePort

```bash
# iptables - อนุญาตเฉพาะ External LB เข้า NodePort
iptables -I INPUT -p tcp --dport 30080 -s 10.0.0.0/24 -j ACCEPT    # Allow from internal
iptables -I INPUT -p tcp --dport 30080 -s 203.0.113.10 -j ACCEPT    # Allow from External LB
iptables -A INPUT -p tcp --dport 30080 -j DROP                       # Block others

# AWS Security Groups
# Inbound rules:
# TCP 30000-32767  Source: 10.0.0.0/24 (Internal only)
# TCP 30000-32767  Source: sg-XXXXXXXX (External LB security group only)

# GCP Firewall Rules
gcloud compute firewall-rules create k8s-nodeport-internal \
  --allow tcp:30000-32767 \
  --source-ranges 10.0.0.0/8 \
  --target-tags kubernetes-node

# Block external access ยกเว้น LB
gcloud compute firewall-rules create k8s-nodeport-lb-only \
  --allow tcp:30000-32767 \
  --source-tags external-lb \
  --target-tags kubernetes-node
```

### 9.3 Network Policy สำหรับ NodePort

```yaml
# protect-nodeport-service.yaml
# Note: NetworkPolicy ทำงานที่ Pod level ไม่ใช่ Node level
# แต่ช่วย restrict traffic หลังจาก เข้ามาใน cluster แล้ว

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-web-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  # อนุญาตจาก NodePort (traffic มาจาก node IP)
  - from:
    - ipBlock:
        cidr: 10.0.0.0/24      # Node IP range
    ports:
    - protocol: TCP
      port: 8080
  # อนุญาตจาก Internal services
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 9.4 Rate Limiting สำหรับ NodePort

```bash
# iptables rate limiting
# จำกัด 100 connections ต่อ minute จาก IP เดียว
iptables -A INPUT -p tcp --dport 30080 \
  -m state --state NEW \
  -m recent --set --name nodeport-ratelimit

iptables -A INPUT -p tcp --dport 30080 \
  -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 100 \
  --name nodeport-ratelimit \
  -j DROP

# หรือใช้ hashlimit module
iptables -A INPUT -p tcp --dport 30080 \
  -m hashlimit \
  --hashlimit-mode srcip \
  --hashlimit-upto 100/min \
  --hashlimit-burst 200 \
  --hashlimit-name nodeport-limit \
  -j ACCEPT

iptables -A INPUT -p tcp --dport 30080 -j DROP
```

---

## 10. Workshop: Expose Multiple Services {#workshop}

### Workshop Overview

ในเวิร์กช็อปนี้ จะสร้าง microservices หลายตัวและ expose ผ่าน NodePort พร้อมตั้งค่า External Load Balancer จำลอง

### Step 1: สร้าง Application Stack

```bash
# สร้าง namespace
kubectl create namespace workshop

# สร้าง backend API
kubectl apply -n workshop -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      tier: backend
  template:
    metadata:
      labels:
        app: api
        tier: backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - -text
        - '{"service":"api","version":"1.0","pod":"$(HOSTNAME)"}'
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-panel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
      tier: admin
  template:
    metadata:
      labels:
        app: admin
        tier: admin
    spec:
      containers:
      - name: admin
        image: hashicorp/http-echo
        args:
        - -text
        - '{"service":"admin","access":"restricted"}'
        ports:
        - containerPort: 5678
EOF

# รอทุก pod ready
kubectl wait --for=condition=Ready pods --all -n workshop --timeout=120s
```

### Step 2: สร้าง NodePort Services

```bash
kubectl apply -n workshop -f - <<'EOF'
# API Service
apiVersion: v1
kind: Service
metadata:
  name: api-service
  annotations:
    purpose: "REST API endpoint"
spec:
  type: NodePort
  selector:
    app: api
    tier: backend
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 8080
    targetPort: 5678
    nodePort: 30180
---
# Web Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    purpose: "Web frontend"
spec:
  type: NodePort
  selector:
    app: web
    tier: frontend
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30181
---
# Admin Panel - restricted
apiVersion: v1
kind: Service
metadata:
  name: admin-service
  annotations:
    purpose: "Admin panel - internal only"
spec:
  type: NodePort
  selector:
    app: admin
    tier: admin
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    port: 8080
    targetPort: 5678
    nodePort: 30182
EOF
```

### Step 3: ทดสอบ Services

```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

echo "=== Testing API Service (port 30180) ==="
curl http://$NODE_IP:30180

echo "=== Testing Web Service (port 30181) ==="
curl -s http://$NODE_IP:30181 | head -5

echo "=== Testing Admin Service (port 30182) ==="
curl http://$NODE_IP:30182

# ทดสอบ load balancing
echo "=== Load Balancing Test ==="
for i in $(seq 1 5); do
  curl -s http://$NODE_IP:30180
  echo
done

# ดู services ทั้งหมด
kubectl get services -n workshop -o wide
```

### Step 4: Network Policy - Restrict Admin

```bash
kubectl apply -n workshop -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-admin
spec:
  podSelector:
    matchLabels:
      tier: admin
  policyTypes:
  - Ingress
  ingress:
  # อนุญาตเฉพาะจาก nodes ภายใน
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8    # Internal network only
    ports:
    - protocol: TCP
      port: 5678
EOF

# ทดสอบว่า NetworkPolicy ทำงาน
kubectl run test-client --rm -it -n workshop --image=curlimages/curl \
  --restart=Never -- curl -s --max-time 5 admin-service.workshop.svc.cluster.local:8080
```

### Step 5: Monitoring NodePort Usage

```bash
# ดู connection counts
kubectl exec -n workshop $(kubectl get pod -n workshop -l app=api -o jsonpath='{.items[0].metadata.name}') \
  -- netstat -an | grep :5678 | wc -l

# ดู service endpoints
kubectl get endpoints -n workshop

# ดู traffic distribution
for service in api-service web-service admin-service; do
  echo "=== $service ==="
  kubectl get endpoints $service -n workshop -o jsonpath='{.subsets[*].addresses[*].ip}'
  echo
done

# iptables rules สำหรับ NodePort
NODEPORT=30180
iptables -t nat -L KUBE-NODEPORTS -n | grep $NODEPORT
```

### Step 6: Cleanup

```bash
kubectl delete namespace workshop
```

---

## 11. แบบฝึกหัด NodePort {#exercises}

### แบบฝึกหัดที่ 1: เปรียบเทียบ externalTrafficPolicy

```bash
# สร้าง environment
kubectl create namespace traffic-policy-test

# สร้าง deployment ที่ log client IP
kubectl apply -n traffic-policy-test -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ip-echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ip-echo
  template:
    metadata:
      labels:
        app: ip-echo
    spec:
      containers:
      - name: echo
        image: k8s.gcr.io/echoserver:1.4
        ports:
        - containerPort: 8080
EOF

# สร้าง 2 services
kubectl expose deployment ip-echo -n traffic-policy-test \
  --type=NodePort --port=80 --target-port=8080 \
  --name=echo-cluster-policy

kubectl expose deployment ip-echo -n traffic-policy-test \
  --type=NodePort --port=80 --target-port=8080 \
  --name=echo-local-policy

kubectl patch service echo-local-policy -n traffic-policy-test \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'

# รอ pods
kubectl wait --for=condition=Ready pods --all -n traffic-policy-test --timeout=60s

# ดู NodePorts
kubectl get services -n traffic-policy-test

# ทดสอบจาก Node เอง
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
CLUSTER_PORT=$(kubectl get svc echo-cluster-policy -n traffic-policy-test -o jsonpath='{.spec.ports[0].nodePort}')
LOCAL_PORT=$(kubectl get svc echo-local-policy -n traffic-policy-test -o jsonpath='{.spec.ports[0].nodePort}')

echo "Cluster policy response:"
curl -s http://$NODE_IP:$CLUSTER_PORT | grep client_address

echo "Local policy response:"
curl -s http://$NODE_IP:$LOCAL_PORT | grep client_address

# สังเกต: client_address ต่างกัน!
# Cluster: node IP (SNAT)
# Local: real client IP

# Cleanup
kubectl delete namespace traffic-policy-test
```

---

### แบบฝึกหัดที่ 2: ค้นหาและแก้ไข NodePort ที่ชน

```bash
# สร้างสถานการณ์ที่ port ซ้ำกัน
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: service-a
spec:
  type: NodePort
  selector:
    app: dummy
  ports:
  - port: 80
    nodePort: 30300
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
spec:
  type: NodePort
  selector:
    app: dummy
  ports:
  - port: 80
    nodePort: 30300    # ← ชนกับ service-a!
EOF
# คาดว่าจะ error: The Service "service-b" is invalid: spec.ports[0].nodePort: Invalid value

# สำรวจ NodePorts ที่ใช้งานอยู่
kubectl get services --all-namespaces -o go-template='
{{range .items}}
{{if eq .spec.type "NodePort"}}
{{.metadata.namespace}}/{{.metadata.name}}:
{{range .spec.ports}}  {{.nodePort}} ({{.name}})
{{end}}
{{end}}
{{end}}'

# หา NodePort ที่ว่าง
python3 -c "
used_ports = {30300}  # ports ที่ใช้แล้ว
all_ports = set(range(30000, 32768))
available = sorted(all_ports - used_ports)
print('Available ports (first 10):', available[:10])
"

# Cleanup
kubectl delete service service-a service-b 2>/dev/null || true
```

---

## 12. เฉลยแบบฝึกหัด {#answers}

### เฉลยแบบฝึกหัดที่ 1

```
ผลที่คาดหวัง:

Cluster policy client_address:
client_address=10.0.0.1  ← Node IP (source NAT'd)

Local policy client_address:
client_address=YOUR_MACHINE_IP  ← Real client IP!

เหตุผล:
- Cluster mode: kube-proxy ทำ SNAT เพื่อ load balance ไปทุก pods
  Source IP ถูกแทนที่ด้วย Node IP
  
- Local mode: kube-proxy ไม่ทำ SNAT
  Source IP ถูก preserve ไว้
  แต่ traffic ไปได้เฉพาะ Pods บน Node ที่รับ request

เมื่อไหรควรใช้อะไร:
- Local: Web servers ที่ต้องรู้ client IP (geo-blocking, rate limiting by IP)
- Cluster: Services ที่ต้องการ load distribution เท่ากัน
```

### เฉลยแบบฝึกหัดที่ 2

```
ผลที่คาดหวัง:

Error เมื่อสร้าง service-b:
The Service "service-b" is invalid: 
  spec.ports[0].nodePort: Invalid value: 30300: 
  provided port is already allocated

วิธีแก้:
1. ใช้ port อื่นที่ไม่ซ้ำ
2. ปล่อยให้ Kubernetes auto-assign (ไม่ระบุ nodePort)
3. ใช้ kubectl get services --all-namespaces ตรวจสอบก่อน

เครื่องมือตรวจสอบ:
kubectl get services --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {range .spec.ports[*]}{.nodePort} {end}{"\n"}{end}' \
  | grep -v "^.*: $"
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes NodePort Documentation](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [externalTrafficPolicy](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip)
- [HAProxy Documentation](https://www.haproxy.org/download/2.8/doc/configuration.txt)
- [Nginx Upstream Configuration](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [iptables Rate Limiting](https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO.html)

---

*ก่อนหน้า: [Part 32 - ClusterIP Service](./part-32-clusterip.md)*
*ต่อไป: [Part 34 - LoadBalancer Service](./part-34-loadbalancer.md)*
