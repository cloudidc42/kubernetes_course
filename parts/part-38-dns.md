# Part 38: DNS ใน Kubernetes

## สารบัญ
1. [CoreDNS คืออะไร](#coredns-overview)
2. [Service Discovery ด้วย DNS](#dns-service-discovery)
3. [DNS Resolution Process](#dns-resolution)
4. [Custom DNS Configuration](#custom-dns)
5. [Workshop: DNS Debugging](#workshop)

---

## 1. CoreDNS คืออะไร {#coredns-overview}

### ภาพรวม CoreDNS

CoreDNS เป็น DNS Server ที่ใช้ใน Kubernetes แทน kube-dns (ตั้งแต่ Kubernetes 1.13)

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                      │
│                                                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │              kube-system namespace                │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │         CoreDNS Deployment              │    │   │
│  │  │                                         │    │   │
│  │  │  Pod 1: coredns-xxx  (10.244.1.2)      │    │   │
│  │  │  Pod 2: coredns-yyy  (10.244.2.3)      │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  Service: kube-dns (ClusterIP: 10.96.0.10)      │   │
│  └──────────────────────────────────────────────────┘   │
│                                                           │
│  Every Pod:                                              │
│  /etc/resolv.conf:                                      │
│  nameserver 10.96.0.10  (CoreDNS ClusterIP)             │
│  search default.svc.cluster.local svc.cluster.local ... │
└───────────────────────────────────────────────────────────┘
```

### CoreDNS Architecture

```
DNS Query Flow:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Pod (nslookup my-service) ──► /etc/resolv.conf         │
│                                      │                  │
│                              nameserver 10.96.0.10      │
│                                      │                  │
│                                      ▼                  │
│                              CoreDNS Service            │
│                              (10.96.0.10:53)            │
│                                      │                  │
│                              ┌───────┴───────┐          │
│                              │   CoreDNS Pod │          │
│                              │               │          │
│                              │  Corefile:    │          │
│                              │  .:53 {      │          │
│                              │    kubernetes │          │
│                              │    forward .  │          │
│                              │  }           │          │
│                              └───────┬───────┘          │
│                                      │                  │
│         ┌────────────────────────────┴────────────┐    │
│         │                            │            │    │
│  Internal Query              External Query       │    │
│  (cluster.local)             (google.com)         │    │
│         │                            │            │    │
│         ▼                            ▼            │    │
│  Kubernetes API           Upstream DNS Servers    │    │
│  (Services/Pods)          (8.8.8.8, 1.1.1.1)    │    │
└──────────────────────────────────────────────────────────┘
```

### Corefile Configuration

```yaml
# kubectl get configmap coredns -n kube-system -o yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors           # Log errors
        health {         # Health endpoint
           lameduck 5s
        }
        ready            # Readiness endpoint
        kubernetes cluster.local in-addr.arpa ip6.arpa {  # Handle cluster.local
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153  # Metrics
        forward . /etc/resolv.conf {  # Forward external queries
           max_concurrent 1000
        }
        cache 30         # Cache responses
        loop             # Detect loops
        reload           # Reload config on change
        loadbalance      # Random loadbalance
    }
```

### CoreDNS Plugins

```
CoreDNS Built-in Plugins:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  kubernetes    ──── Handle cluster.local queries        │
│  forward       ──── Forward non-cluster queries         │
│  cache         ──── Cache DNS responses                 │
│  errors        ──── Log errors                         │
│  health        ──── Health check endpoint               │
│  ready         ──── Readiness endpoint                 │
│  prometheus    ──── Metrics                            │
│  rewrite       ──── Rewrite DNS queries                │
│  hosts         ──── Serve from /etc/hosts              │
│  file          ──── Serve from zone files              │
│  loop          ──── Detect DNS loops                   │
│  reload        ──── Hot reload Corefile                │
│  log           ──── Log DNS queries                    │
│  dnssec        ──── DNSSEC signing                     │
│  autopath      ──── Auto search path for pods         │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Service Discovery ด้วย DNS {#dns-service-discovery}

### DNS Records สำหรับ Services

```
Service DNS Records:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ClusterIP Service:                                     │
│  <service>.<namespace>.svc.cluster.local               │
│  → A Record: ClusterIP                                  │
│                                                          │
│  Headless Service:                                      │
│  <service>.<namespace>.svc.cluster.local               │
│  → A Records: All Pod IPs                              │
│                                                          │
│  Named Port (SRV):                                      │
│  _<port-name>._<protocol>.<service>.<ns>.svc.cluster.local │
│  → SRV Record: weight priority port target             │
│                                                          │
│  External Name Service:                                 │
│  <service>.<namespace>.svc.cluster.local               │
│  → CNAME: externalName value                          │
└──────────────────────────────────────────────────────────┘
```

### DNS Patterns

```yaml
# dns-test-services.yaml
# Pattern 1: Same namespace - use short name
# From pod in 'default' namespace:
# → nslookup my-service (resolved via search path)

# Pattern 2: Cross namespace - use FQDN
# From pod in 'frontend' namespace to 'backend' namespace:
# → nslookup api-service.backend.svc.cluster.local

# Pattern 3: Headless service - get all Pod IPs
apiVersion: v1
kind: Service
metadata:
  name: stateful-headless
  namespace: default
spec:
  clusterIP: None
  selector:
    app: stateful
  ports:
  - port: 80
    targetPort: 8080
```

### DNS สำหรับ StatefulSets

StatefulSets ใช้ Headless Service เพื่อให้แต่ละ Pod มี DNS name เป็นของตัวเอง

```
StatefulSet DNS:
<pod-name>.<headless-service>.<namespace>.svc.cluster.local

ตัวอย่าง:
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

```yaml
# statefulset-dns.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
spec:
  clusterIP: None  # Headless
  selector:
    app: mysql
  ports:
  - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: default
spec:
  serviceName: mysql  # Headless service name
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
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
```

```bash
# ทดสอบ StatefulSet DNS
kubectl exec -it mysql-0 -- bash
# mysql -h mysql-0.mysql.default.svc.cluster.local  # Access self
# mysql -h mysql-1.mysql.default.svc.cluster.local  # Access pod 1
# exit
```

### DNS สำหรับ Pods

```
Pod DNS (เมื่อ Pod มี hostname/subdomain):
<hostname>.<subdomain>.<namespace>.svc.cluster.local

ตัวอย่าง:
web-pod.web-service.default.svc.cluster.local
```

```yaml
# pod-with-dns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
spec:
  hostname: web-pod       # DNS hostname
  subdomain: web-service  # DNS subdomain (ต้องมี Service ชื่อนี้)
  containers:
  - name: app
    image: nginx:alpine
---
# ต้องมี Headless Service ชื่อเดียวกับ subdomain
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
spec:
  clusterIP: None
  selector:
    app: my-pod
  ports:
  - port: 80
```

---

## 3. DNS Resolution Process {#dns-resolution}

### /etc/resolv.conf ใน Pod

```
# ตัวอย่าง /etc/resolv.conf ใน Pod ที่อยู่ใน namespace 'production'
nameserver 10.96.0.10
search production.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Search Domain Resolution

```
Query: nslookup my-service

Steps:
1. my-service  → ลองก่อน (ถ้ามี dot น้อยกว่า ndots=5)
2. my-service.production.svc.cluster.local  → search path 1
3. my-service.svc.cluster.local  → search path 2
4. my-service.cluster.local  → search path 3
5. my-service.  → absolute query

ถ้า ndots=5 และ hostname มีน้อยกว่า 5 dots:
→ ลอง search paths ก่อนแล้วค่อย absolute

ถ้า hostname มี 5+ dots:
→ ลอง absolute ก่อน
```

### ndots Configuration

```yaml
# pod-dns-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
    - name: ndots
      value: "2"   # ลด ndots เพื่อ performance
    - name: timeout
      value: "5"
    - name: attempts
      value: "3"
    searches:
    - extra-search.example.com
  containers:
  - name: app
    image: nginx:alpine
```

### DNS Policies

```yaml
# dns-policies.yaml
# Policy 1: ClusterFirst (default)
# ถาม CoreDNS ก่อน, ถ้าไม่ได้ถาม upstream
spec:
  dnsPolicy: ClusterFirst

---
# Policy 2: Default
# ใช้ DNS ของ Node (ไม่ใช้ CoreDNS)
spec:
  dnsPolicy: Default

---
# Policy 3: None  
# กำหนด DNS เองทั้งหมด
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - example.com
    options:
    - name: ndots
      value: "5"

---
# Policy 4: ClusterFirstWithHostNet
# สำหรับ Pods ที่ใช้ hostNetwork: true
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

---

## 4. Custom DNS Configuration {#custom-dns}

### เพิ่ม Custom DNS Entries

```yaml
# coredns-custom.yaml
# เพิ่ม custom hosts
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        hosts /etc/coredns/NodeHosts {
          192.168.1.100 legacy-server.internal
          192.168.1.101 old-database.internal
          fallthrough
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### Stub Zones (Forwarding ไป DNS เฉพาะ)

```yaml
# coredns-stub-zone.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    
    # Stub zone: corp.example.com → internal DNS
    corp.example.com:53 {
        errors
        cache 30
        forward . 10.10.0.53 10.10.1.53  # Corporate DNS servers
    }
    
    # Stub zone: legacy.internal → legacy DNS
    legacy.internal:53 {
        errors
        cache 30
        forward . 192.168.1.10
    }
```

### Custom DNS สำหรับ Namespace

```yaml
# namespace-dns.yaml
# เพิ่ม DNS entries เฉพาะสำหรับ namespace
# (ต้องใช้ CoreDNS plugin หรือ rewrite)
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        rewrite name substring .prod.svc.cluster.local .production.svc.cluster.local
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### DNS Rewrite Rules

```yaml
# coredns-rewrite.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        # Rewrite example:
        # api.example.com → api-service.default.svc.cluster.local
        rewrite name api.example.com api-service.default.svc.cluster.local
        
        # Rewrite with regex:
        # *.legacy.example.com → <name>.legacy-ns.svc.cluster.local
        rewrite name regex (.*)\.legacy\.example\.com {1}.legacy-ns.svc.cluster.local
        
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### NodeLocal DNSCache (Performance)

NodeLocal DNSCache รัน DNS cache บนทุก Node เพื่อลด latency

```yaml
# nodelocaldns.yaml
# ติดตั้ง NodeLocal DNSCache
# https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/

# หลังจากติดตั้ง Pods จะใช้ 169.254.20.10 แทน CoreDNS
# (link-local address บน node)

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: node-local-dns
  template:
    metadata:
      labels:
        k8s-app: node-local-dns
    spec:
      hostNetwork: true
      dnsPolicy: Default
      containers:
      - name: node-cache
        image: registry.k8s.io/dns/k8s-dns-node-cache:1.22.28
        resources:
          requests:
            cpu: 25m
            memory: 5Mi
        args:
        - -localip
        - "169.254.20.10,10.96.0.10"
        - -conf
        - /etc/Corefile
        - -upstreamsvc
        - kube-dns
```

---

## 5. Workshop: DNS Debugging {#workshop}

### Workshop 1: ตั้ง Test Environment

```bash
# ใช้ existing cluster หรือสร้างใหม่
kind create cluster --name dns-debug

# ดู CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl get service kube-dns -n kube-system

# ดู Corefile
kubectl get configmap coredns -n kube-system -o yaml
```

### Workshop 2: สร้าง Test Services

```yaml
# dns-test-setup.yaml
# Namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: app-frontend
---
apiVersion: v1
kind: Namespace
metadata:
  name: app-backend
---
# Backend Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: app-backend
spec:
  replicas: 2
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
        image: hashicorp/http-echo
        args:
        - "-text=Backend Service Response"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: app-backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
---
# ExternalName Service
apiVersion: v1
kind: Service
metadata:
  name: google-dns
  namespace: app-frontend
spec:
  type: ExternalName
  externalName: dns.google
  ports:
  - port: 443
---
# Headless Service
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
  namespace: app-backend
spec:
  serviceName: stateful-headless
  replicas: 3
  selector:
    matchLabels:
      app: stateful
  template:
    metadata:
      labels:
        app: stateful
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: stateful-headless
  namespace: app-backend
spec:
  clusterIP: None
  selector:
    app: stateful
  ports:
  - port: 80
```

```bash
kubectl apply -f dns-test-setup.yaml
kubectl wait --for=condition=available deployment/backend-app -n app-backend --timeout=60s
```

### Workshop 3: Basic DNS Testing

```bash
# สร้าง DNS test pod
kubectl run dns-debug \
  --image=nicolaka/netshoot \
  --namespace=app-frontend \
  --rm -it -- bash

# ภายใน Pod:

# ดู resolv.conf
cat /etc/resolv.conf

# Test 1: Same namespace service (short name)
echo "=== Testing same namespace service ==="
nslookup google-dns
dig google-dns

# Test 2: Cross-namespace (full name)
echo "=== Testing cross-namespace service ==="
nslookup backend-service.app-backend.svc.cluster.local
dig backend-service.app-backend.svc.cluster.local

# Test 3: External DNS
echo "=== Testing external DNS ==="
nslookup google.com
dig google.com

# Test 4: FQDN with trailing dot
echo "=== Testing FQDN ==="
nslookup backend-service.app-backend.svc.cluster.local.

# Test 5: SRV records
echo "=== Testing SRV records ==="
dig SRV _http._tcp.backend-service.app-backend.svc.cluster.local

# ออก
exit
```

### Workshop 4: Headless Service DNS

```bash
kubectl run dns-debug2 \
  --image=nicolaka/netshoot \
  --namespace=app-backend \
  --rm -it -- bash

# ภายใน Pod:

# Test headless service - should return all Pod IPs
echo "=== Headless Service DNS ==="
nslookup stateful-headless
dig stateful-headless.app-backend.svc.cluster.local

# Test individual StatefulSet Pods
echo "=== Individual Pod DNS ==="
nslookup stateful-app-0.stateful-headless
nslookup stateful-app-1.stateful-headless
nslookup stateful-app-2.stateful-headless

# Full FQDN
nslookup stateful-app-0.stateful-headless.app-backend.svc.cluster.local

exit
```

### Workshop 5: DNS Performance Testing

```bash
kubectl run dns-perf \
  --image=nicolaka/netshoot \
  --namespace=default \
  --rm -it -- bash

# ภายใน Pod:

# วัด DNS resolution time
echo "=== DNS Timing ==="
time nslookup backend-service.app-backend.svc.cluster.local
time nslookup google.com

# ใช้ dig สำหรับ detailed timing
dig backend-service.app-backend.svc.cluster.local | grep "Query time"

# ทดสอบหลายๆ ครั้ง
for i in {1..10}; do
  time nslookup backend-service.app-backend.svc.cluster.local 2>&1 | grep real
done

# ทดสอบ ndots impact
# Query ที่มี dots น้อยกว่า 5 จะถูก search paths ก่อน
time nslookup backend    # 1 dot - search paths ก่อน (ช้ากว่า)
time nslookup backend-service.app-backend.svc.cluster.local  # FQDN (เร็วกว่า)

exit
```

### Workshop 6: Customize CoreDNS

```bash
# เพิ่ม log สำหรับ debug
kubectl edit configmap coredns -n kube-system
# เพิ่ม 'log' plugin ใน Corefile:
```

```
# เพิ่ม log plugin ใน Corefile
.:53 {
    errors
    log  # <-- เพิ่มบรรทัดนี้
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

```bash
# Restart CoreDNS pods เพื่อ reload config
kubectl rollout restart deployment/coredns -n kube-system

# ดู DNS logs (จะเห็น queries)
kubectl logs -n kube-system -l k8s-app=kube-dns -f

# ในอีก terminal: สร้าง DNS queries
kubectl run test --image=busybox --rm -it -- nslookup google.com
```

### Workshop 7: เพิ่ม Stub Zone

```yaml
# add-stub-zone.yaml
# เพิ่ม zone สำหรับ company internal DNS
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    
    # Stub zone สำหรับ internal domain
    # ส่ง queries ไปที่ internal DNS server
    company.internal:53 {
        errors
        cache 30
        forward . 10.10.0.53 {
            prefer_udp
        }
    }
```

### Workshop 8: Debug DNS Issues

```bash
# ปัญหาทั่วไปและวิธีแก้

# 1. DNS resolution ช้า
# ตรวจสอบ CoreDNS metrics
kubectl port-forward -n kube-system service/kube-dns 9253:9153
curl localhost:9253/metrics | grep coredns_

# 2. DNS queries ล้มเหลว
# ตรวจสอบ CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# 3. ndots ปัญหา - query ช้าเพราะลอง search paths หลายรอบ
# แก้โดย config dnsConfig
cat > fix-ndots.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-ndots
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"  # ลดจาก default 5
  containers:
  - name: app
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl apply -f fix-ndots.yaml
kubectl exec test-ndots -- cat /etc/resolv.conf
# จะเห็น ndots:2

# 4. ตรวจสอบ kube-dns Service
kubectl get service kube-dns -n kube-system

# 5. Test DNS จาก debug pod
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  --restart=Never -it --rm -- nslookup kubernetes.default

# 6. ตรวจสอบ CoreDNS configmap
kubectl get configmap coredns -n kube-system -o yaml

# 7. ตรวจสอบ resource limits
kubectl describe pod -n kube-system -l k8s-app=kube-dns | grep -A5 "Limits\|Requests"
```

### Workshop 9: Advanced DNS Debugging

```bash
# ใช้ tcpdump จับ DNS traffic
# ต้อง access Node โดยตรง

# หรือใช้ pod ที่มี cap NET_ADMIN
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dns-tcpdump
spec:
  containers:
  - name: tcpdump
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
EOF

kubectl exec -it dns-tcpdump -- bash
# ภายใน pod:
# จับ DNS traffic
tcpdump -i any -n port 53

# ในอีก terminal:
kubectl exec dns-tcpdump -- nslookup google.com

# หยุด tcpdump และดูผล

exit
```

### Workshop 10: DNS-based Service Configuration

```yaml
# config-via-dns.yaml
# ใช้ DNS records เพื่อ config application
apiVersion: v1
kind: Service
metadata:
  name: db-primary
  namespace: default
spec:
  selector:
    role: primary
    app: mysql
  ports:
  - port: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: db-replica
  namespace: default
spec:
  selector:
    role: replica
    app: mysql
  ports:
  - port: 3306
---
# Application ใช้ DNS names แทน hardcoded IPs
# DB_PRIMARY=db-primary.default.svc.cluster.local
# DB_REPLICA=db-replica.default.svc.cluster.local
```

### Cleanup

```bash
kubectl delete namespace app-frontend app-backend
kubectl delete pod dns-debug dns-debug2 dns-perf test-ndots dns-tcpdump 2>/dev/null
kind delete cluster --name dns-debug
```

---

## CoreDNS Metrics

```bash
# ดู CoreDNS metrics
kubectl port-forward -n kube-system service/kube-dns 9253:9153

curl localhost:9253/metrics | grep -E "coredns_dns_requests_total|coredns_dns_responses_total|coredns_cache"

# Metrics ที่สำคัญ:
# coredns_dns_requests_total - จำนวน DNS requests
# coredns_dns_responses_total - จำนวน DNS responses
# coredns_cache_hits_total - Cache hits
# coredns_cache_misses_total - Cache misses
# coredns_forward_requests_total - Forwarded requests
# coredns_forward_request_duration_seconds - Forward latency
```

---

## Cheat Sheet

```bash
# ดู CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get service kube-dns -n kube-system
kubectl get configmap coredns -n kube-system

# Debug DNS จาก Pod
kubectl run dns-test --image=nicolaka/netshoot --rm -it -- bash
# nslookup <service-name>
# dig <service-name>.<namespace>.svc.cluster.local
# cat /etc/resolv.conf

# ดู DNS records สำหรับ service
kubectl exec <pod> -- nslookup <service-name>.<namespace>
kubectl exec <pod> -- dig SRV _<port-name>._tcp.<service>.<namespace>.svc.cluster.local

# Restart CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# ดู CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# ดู resolv.conf ใน Pod
kubectl exec <pod> -- cat /etc/resolv.conf

# Test DNS
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local 10.96.0.10
```

---

*ก่อนหน้า: [Part 37 - Network Policies](./part-37-network-policies.md)*
*ต่อไป: [Part 39 - Service Mesh](./part-39-service-mesh.md)*
