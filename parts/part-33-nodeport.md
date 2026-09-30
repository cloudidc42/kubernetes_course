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
