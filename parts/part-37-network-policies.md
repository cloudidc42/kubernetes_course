# Part 37: Network Policies

## สารบัญ
1. [Network Policy คืออะไร](#network-policy-overview)
2. [Ingress/Egress Rules](#ingress-egress)
3. [Network Policy YAML](#network-policy-yaml)
4. [Calico Network Policies](#calico-policies)
5. [Workshop: Implement Zero-trust Networking](#workshop)

---

## 1. Network Policy คืออะไร {#network-policy-overview}

### ปัญหาที่ Network Policy แก้ไข

```
ปัญหา: Default behavior ใน Kubernetes
┌──────────────────────────────────────────────────┐
│                                                  │
│  ทุก Pod สามารถสื่อสารกับทุก Pod ได้!            │
│                                                  │
│  Pod A ──────────────────────────► Pod B        │
│  Pod A ──────────────────────────► Pod C        │
│  Pod A ──────────────────────────► Database     │
│                                                  │
│  ⚠️ Frontend สามารถเข้าถึง Database โดยตรง!    │
│  ⚠️ Compromised Pod สามารถ scan network ทั้งหมด │
└──────────────────────────────────────────────────┘

แก้ด้วย Network Policy (Zero-trust):
┌──────────────────────────────────────────────────┐
│                                                  │
│  Frontend ──► API ──► Database                  │
│  (blocked directly)    (only from API)           │
│                                                  │
│  ✅ Frontend เข้าถึงได้แค่ API                  │
│  ✅ API เข้าถึง Database ได้                    │
│  ✅ Database ไม่ถูกเข้าถึงจาก Pods อื่น         │
└──────────────────────────────────────────────────┘
```

### Network Policy คืออะไร?

Network Policy คือ Kubernetes resource ที่กำหนดกฎ Firewall สำหรับ Pods โดยใช้ Label Selectors

```
Network Policy Components:
┌─────────────────────────────────────────────────────┐
│                                                      │
│  podSelector       ──── เลือก Pods ที่จะ apply     │
│  policyTypes       ──── Ingress, Egress, หรือทั้งคู่│
│  ingress rules     ──── อนุญาต traffic เข้า         │
│  egress rules      ──── อนุญาต traffic ออก          │
│                                                      │
│  Selectors:                                          │
│  ├── podSelector   ──── เลือกด้วย Pod labels        │
│  ├── namespaceSelector ── เลือกด้วย Namespace labels│
│  └── ipBlock       ──── เลือกด้วย IP CIDR          │
└─────────────────────────────────────────────────────┘
```

### สิ่งสำคัญที่ต้องรู้

```
Rules ของ Network Policy:

1. Default: ถ้าไม่มี NetworkPolicy  → ทุกอย่าง Allow
2. ถ้ามี NetworkPolicy apply กับ Pod → เฉพาะที่ระบุในนั้น Allow
3. Multiple policies → Union (OR) ของทุก rule
4. NetworkPolicy ต้องการ CNI ที่รองรับ (Calico, Cilium, Weave)
5. Flannel ไม่รองรับ NetworkPolicy (ต้องใช้ Canal หรือ Calico แทน)
```

---

## 2. Ingress/Egress Rules {#ingress-egress}

### Ingress Rules (Traffic เข้า Pod)

```
Ingress Rule:
"อนุญาต traffic ที่เข้า Pod นี้ จากไหน"

ตัวอย่าง:
Pod database:
  Allow ingress from: Pods with label app=api
  Block ingress from: everything else
```

### Egress Rules (Traffic ออกจาก Pod)

```
Egress Rule:
"อนุญาต traffic ที่ออกจาก Pod นี้ ไปไหน"

ตัวอย่าง:
Pod frontend:
  Allow egress to: Pods with label app=api
  Allow egress to: 0.0.0.0/0 on port 443 (external HTTPS)
  Block egress to: everything else
```

### Policy Types

```yaml
# policyTypes ที่เป็นไปได้:

# กำหนดเฉพาะ Ingress
policyTypes:
- Ingress

# กำหนดเฉพาะ Egress  
policyTypes:
- Egress

# กำหนดทั้งคู่
policyTypes:
- Ingress
- Egress
```

### Selector Types ใน Rules

```yaml
# 1. podSelector - เลือก Pods ด้วย Labels
from:
- podSelector:
    matchLabels:
      app: api

# 2. namespaceSelector - เลือก Namespace ด้วย Labels
from:
- namespaceSelector:
    matchLabels:
      environment: production

# 3. Combination: Pods ในกำหนด Namespace
from:
- podSelector:
    matchLabels:
      app: api
  namespaceSelector:
    matchLabels:
      environment: production
  # AND: Pods ที่มี label app=api ใน namespace ที่มี label environment=production

# 4. ipBlock - เลือกด้วย IP Range
from:
- ipBlock:
    cidr: 10.0.0.0/8
    except:
    - 10.1.0.0/16  # ยกเว้น subnet นี้
```

---

## 3. Network Policy YAML {#network-policy-yaml}

### Basic Network Policy Examples

```yaml
# 1. Deny All Traffic (Default Deny)
# ปิด traffic ทุกทิศทางสำหรับ Pods ที่ match
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # {} = match ทุก Pod ใน namespace
  policyTypes:
  - Ingress
  - Egress
  # ไม่มี ingress/egress rules = deny ทั้งหมด
```

```yaml
# 2. Allow All Traffic (ยกเลิก deny all)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}  # {} = allow จากทุกที่
  egress:
  - {}  # {} = allow ไปทุกที่
```

```yaml
# 3. Deny All Ingress (แต่ allow egress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # ไม่ระบุ Egress ใน policyTypes = ไม่ affect egress
```

### Application-specific Policies

```yaml
# database-policy.yaml
# Database: อนุญาตเฉพาะ API tier เข้าถึง
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # อนุญาตจาก API tier
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
  # อนุญาตจาก monitoring namespace
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9187  # postgres_exporter
  egress:
  # อนุญาต DNS
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```yaml
# api-policy.yaml
# API tier: อนุญาตจาก frontend และออกไป database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # รับจาก frontend
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  # รับจาก load balancer / ingress
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # ไป database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # ไป cache (Redis)
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  # DNS
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # External APIs
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
```

```yaml
# frontend-policy.yaml
# Frontend: รับจาก ingress, ออกไป API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # รับจาก ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  egress:
  # ไป API
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8080
  # DNS
  - ports:
    - protocol: UDP
      port: 53
```

### Namespace-level Policies

```yaml
# namespace-isolation.yaml
# แยก Namespaces ออกจากกัน
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # อนุญาตเฉพาะใน namespace เดียวกัน
  - from:
    - podSelector: {}  # ทุก Pod ใน namespace เดียวกัน
  # อนุญาตจาก ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  # อนุญาตเฉพาะใน namespace เดียวกัน
  - to:
    - podSelector: {}
  # DNS
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Kubernetes API server
  - to:
    - ipBlock:
        cidr: 10.96.0.1/32  # kube-apiserver ClusterIP
    ports:
    - protocol: TCP
      port: 443
```

---

## 4. Calico Network Policies {#calico-policies}

### Calico GlobalNetworkPolicy

Calico มี CRD เพิ่มเติมสำหรับ Advanced policies ที่ span ข้าม namespaces

```yaml
# calico-global-policy.yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-deny-all
spec:
  # Apply ทุก endpoint ใน cluster
  selector: all()
  order: 1000  # ต่ำกว่า = priority สูงกว่า
  types:
  - Ingress
  - Egress
  # ไม่มี rules = deny all
---
# Allow DNS ทั่วทั้ง Cluster
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  selector: all()
  order: 100
  types:
  - Egress
  egress:
  - action: Allow
    protocol: UDP
    destination:
      ports:
      - 53
  - action: Allow
    protocol: TCP
    destination:
      ports:
      - 53
```

### Calico NetworkPolicy (Namespace-scoped)

```yaml
# calico-network-policy.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: advanced-api-policy
  namespace: production
spec:
  # เลือก Pods
  selector: tier == 'api'
  order: 10
  types:
  - Ingress
  - Egress
  ingress:
  # Allow จาก frontend tier
  - action: Allow
    protocol: TCP
    source:
      selector: tier == 'frontend'
    destination:
      ports:
      - 8080
  # Allow จาก specific namespace
  - action: Allow
    protocol: TCP
    source:
      namespaceSelector: environment == 'production'
      selector: role == 'ingress'
  # Log และ deny อื่นๆ
  - action: Log
    metadata:
      annotations:
        from: "unauthorized-access"
  - action: Deny
  egress:
  # Allow ไป database
  - action: Allow
    protocol: TCP
    destination:
      selector: tier == 'database'
      ports:
      - 5432
  # Allow ไป external APIs
  - action: Allow
    protocol: TCP
    destination:
      nets:
      - 0.0.0.0/0
      notNets:
      - 10.0.0.0/8
      ports:
      - 443
```

### Calico Host Endpoints

```yaml
# host-endpoint-policy.yaml
# ควบคุม traffic บน Host Network ด้วย
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: node1-eth0
  labels:
    node: node1
spec:
  interfaceName: eth0
  node: node1
  expectedIPs:
  - 192.168.1.10
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: host-endpoint-policy
spec:
  selector: node == 'node1'
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      nets:
      - 192.168.1.0/24
    destination:
      ports:
      - 22  # SSH
      - 6443  # API Server
  - action: Deny
```

---

## 5. Workshop: Implement Zero-trust Networking {#workshop}

### สภาพแวดล้อม Workshop

```bash
# ต้องใช้ CNI ที่รองรับ NetworkPolicy
# ติดตั้ง kind กับ Calico
cat > kind-calico.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: zero-trust
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true  # ปิด default CNI
  podSubnet: "10.244.0.0/16"
EOF

kind create cluster --config kind-calico.yaml

# ติดตั้ง Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# รอ Calico พร้อม
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=120s
kubectl wait --for=condition=ready pod -l k8s-app=calico-kube-controllers -n kube-system --timeout=120s

kubectl get pods -n kube-system | grep calico
kubectl get nodes
```

### Workshop 1: Deploy 3-tier Application

```yaml
# three-tier-app.yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: three-tier
  labels:
    purpose: demo
---
# Database Tier
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: three-tier
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: db
        image: hashicorp/http-echo
        args:
        - "-text=Database Response"
        - "-listen=:5432"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: three-tier
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
---
# API Tier
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      tier: api
  template:
    metadata:
      labels:
        app: api
        tier: api
    spec:
      containers:
      - name: api
        image: nicolaka/netshoot
        command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: three-tier
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
---
# Frontend Tier
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: nicolaka/netshoot
        command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: three-tier
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
# Attacker Pod (จำลองการโจมตี)
apiVersion: v1
kind: Pod
metadata:
  name: attacker
  namespace: three-tier
  labels:
    app: attacker
spec:
  containers:
  - name: attacker
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
```

```bash
kubectl apply -f three-tier-app.yaml

# ตรวจสอบ Pods
kubectl get pods -n three-tier -o wide

# รอ Pods พร้อม
kubectl wait --for=condition=ready pod -l app=database -n three-tier --timeout=60s
kubectl wait --for=condition=ready pod -l app=api -n three-tier --timeout=60s
kubectl wait --for=condition=ready pod -l app=frontend -n three-tier --timeout=60s
```

### Workshop 2: Test ก่อนใช้ Network Policy

```bash
# ทดสอบ connectivity ก่อนใช้ Network Policy
DB_IP=$(kubectl get service database -n three-tier -o jsonpath='{.spec.clusterIP}')
echo "Database IP: $DB_IP"

# Frontend เข้าถึง Database ได้ (ไม่ควรเกิดในระบบจริง!)
kubectl exec -n three-tier deploy/frontend -- nc -zv database 5432
# Expected: ได้ผล (connection succeed) → PROBLEM!

# Attacker เข้าถึงได้ทุกอย่าง
kubectl exec -n three-tier attacker -- nc -zv database 5432
# Expected: ได้ผล → PROBLEM!

kubectl exec -n three-tier attacker -- nc -zv api 8080
# Expected: ได้ผล → PROBLEM!
```

### Workshop 3: Implement Default Deny

```yaml
# default-deny.yaml
# Step 1: Default deny ทุกอย่างก่อน
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: three-tier
spec:
  podSelector: {}  # Apply กับทุก Pod
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f default-deny.yaml

# ทดสอบ - ควร fail ทั้งหมด
kubectl exec -n three-tier deploy/frontend -- nc -zv database 5432 --wait=3
# Expected: Connection refused/timeout → GOOD!

kubectl exec -n three-tier deploy/api -- nc -zv database 5432 --wait=3
# Expected: Connection refused/timeout → GOOD!

# แต่ DNS ก็ fail ด้วย!
kubectl exec -n three-tier deploy/frontend -- nslookup database
# Expected: fail → Need to add DNS exception
```

### Workshop 4: Allow DNS

```yaml
# allow-dns.yaml
# Allow DNS สำหรับทุก Pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: three-tier
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
kubectl apply -f allow-dns.yaml

# ทดสอบ DNS
kubectl exec -n three-tier deploy/frontend -- nslookup database
# Expected: DNS resolution works

# แต่ยังเชื่อมต่อ Database ไม่ได้
kubectl exec -n three-tier deploy/frontend -- nc -zv database 5432 --wait=3
# Expected: fail → GOOD!
```

### Workshop 5: Allow Specific Traffic

```yaml
# specific-policies.yaml
# Allow API เข้าถึง Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-allow-api
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 5432
---
# Allow Frontend เข้าถึง API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # ไป Database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # DNS
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
kubectl apply -f specific-policies.yaml

# ทดสอบ
echo "=== Testing Zero-trust Network ==="

echo "1. API → Database (should succeed):"
kubectl exec -n three-tier deploy/api -- nc -zv database 5432 --wait=5
echo ""

echo "2. Frontend → Database (should FAIL):"
kubectl exec -n three-tier deploy/frontend -- nc -zv database 5432 --wait=3 2>&1 || echo "BLOCKED (as expected)"
echo ""

echo "3. Attacker → Database (should FAIL):"
kubectl exec -n three-tier attacker -- nc -zv database 5432 --wait=3 2>&1 || echo "BLOCKED (as expected)"
echo ""

echo "4. Attacker → API (should FAIL):"
kubectl exec -n three-tier attacker -- nc -zv api 8080 --wait=3 2>&1 || echo "BLOCKED (as expected)"
echo ""
```

### Workshop 6: Namespace Isolation

```yaml
# namespace-isolation.yaml
# Namespace A
apiVersion: v1
kind: Namespace
metadata:
  name: ns-alpha
  labels:
    team: alpha
---
apiVersion: v1
kind: Namespace
metadata:
  name: ns-beta
  labels:
    team: beta
---
# Pod ใน ns-alpha
apiVersion: v1
kind: Pod
metadata:
  name: pod-alpha
  namespace: ns-alpha
  labels:
    app: pod-alpha
spec:
  containers:
  - name: app
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
# Pod ใน ns-beta
apiVersion: v1
kind: Pod
metadata:
  name: pod-beta
  namespace: ns-beta
  labels:
    app: pod-beta
spec:
  containers:
  - name: app
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
# Isolate ns-alpha จาก ns-beta
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: ns-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # อนุญาตเฉพาะ within namespace
  - from:
    - podSelector: {}
  egress:
  # อนุญาตเฉพาะ within namespace
  - to:
    - podSelector: {}
  # DNS
  - ports:
    - protocol: UDP
      port: 53
```

```bash
kubectl apply -f namespace-isolation.yaml

# ดู Pods IP
ALPHA_IP=$(kubectl get pod pod-alpha -n ns-alpha -o jsonpath='{.status.podIP}')
BETA_IP=$(kubectl get pod pod-beta -n ns-beta -o jsonpath='{.status.podIP}')

echo "Alpha Pod IP: $ALPHA_IP"
echo "Beta Pod IP: $BETA_IP"

# ทดสอบ isolation
echo "Alpha → Beta (should FAIL):"
kubectl exec -n ns-alpha pod-alpha -- ping -c2 -W2 $BETA_IP 2>&1 || echo "ISOLATED!"

echo "Beta → Alpha (should work by default, beta has no policy):"
kubectl exec -n ns-beta pod-beta -- ping -c2 -W2 $ALPHA_IP
```

### Workshop 7: Monitoring Network Policy

```bash
# ดู Network Policies
kubectl get networkpolicies --all-namespaces

# ดู Policy details
kubectl describe networkpolicy default-deny-all -n three-tier

# Calico: ดู Network Policy ทั้งหมด
kubectl get networkpolicies -n three-tier
kubectl get networkpolicies -n three-tier -o yaml

# Debug network connectivity
kubectl exec -n three-tier deploy/api -- \
  curl -v --connect-timeout 3 telnet://database:5432 2>&1

# ดู Calico logs
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50

# Calico diagnostics
# (ถ้าติดตั้ง calicoctl)
calicoctl get networkpolicies --all-namespaces
```

### Workshop 8: Network Policy Testing Tool

```bash
# ติดตั้ง netpol-tester
kubectl apply -f https://raw.githubusercontent.com/ahmetb/kubernetes-network-policy-recipes/master/hack/test-runner.yaml

# หรือทดสอบด้วย netshoot
cat > test-connectivity.sh << 'SCRIPT'
#!/bin/bash

NAMESPACE="three-tier"
RESULTS=""

check_connectivity() {
  local from_pod=$1
  local to_host=$2
  local to_port=$3
  local expected=$4
  
  result=$(kubectl exec -n $NAMESPACE $from_pod -- \
    nc -zv $to_host $to_port --wait=3 2>&1)
  
  if echo "$result" | grep -q "succeeded\|open"; then
    actual="ALLOWED"
  else
    actual="BLOCKED"
  fi
  
  status="❌"
  if [ "$actual" == "$expected" ]; then
    status="✅"
  fi
  
  echo "$status $from_pod → $to_host:$to_port (expected: $expected, actual: $actual)"
}

echo "=== Network Policy Test Results ==="
check_connectivity "deploy/api" "database" "5432" "ALLOWED"
check_connectivity "deploy/frontend" "database" "5432" "BLOCKED"
check_connectivity "attacker" "database" "5432" "BLOCKED"
check_connectivity "attacker" "api" "8080" "BLOCKED"
SCRIPT

chmod +x test-connectivity.sh
./test-connectivity.sh
```

### Cleanup

```bash
kubectl delete namespace three-tier ns-alpha ns-beta
kind delete cluster --name zero-trust
```

---

## Best Practices

```
Network Policy Best Practices:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  1. Start with Default Deny                             │
│     - Apply deny-all ก่อน                              │
│     - เพิ่ม allow rules ทีละอัน                       │
│                                                          │
│  2. Always Allow DNS                                    │
│     - Pod ต้องการ DNS เพื่อ resolve service names      │
│                                                          │
│  3. Label Pods อย่างสม่ำเสมอ                          │
│     - tier: frontend/api/database                      │
│     - environment: prod/staging/dev                    │
│                                                          │
│  4. Test ทุก Policy                                    │
│     - ทดสอบ allowed connections                        │
│     - ทดสอบ blocked connections                       │
│                                                          │
│  5. Document Policies                                   │
│     - ระบุ purpose ใน metadata.annotations              │
│                                                          │
│  6. ใช้ Namespace-scoped policies                      │
│     - แยก policies ตาม namespace                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Cheat Sheet

```bash
# ดู Network Policies
kubectl get networkpolicies
kubectl get netpol  # short form
kubectl get netpol -n production

# ดู policy details
kubectl describe netpol my-policy -n production

# Apply policy
kubectl apply -f network-policy.yaml

# Delete policy (CAREFUL! Will open traffic)
kubectl delete netpol my-policy

# ดู Pods labels (for writing policies)
kubectl get pods --show-labels

# Test connectivity
kubectl exec <pod> -- nc -zv <target> <port>
kubectl exec <pod> -- curl --connect-timeout 3 http://<target>:<port>

# Calico tools
calicoctl get networkpolicies
calicoctl get globalnetworkpolicies
```

---

*ก่อนหน้า: [Part 36 - Ingress Resources](./part-36-ingress-resources.md)*
*ต่อไป: [Part 38 - DNS](./part-38-dns.md)*
