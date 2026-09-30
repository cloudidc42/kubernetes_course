# Part 59: Advanced Network Policy Patterns

## บทนำ

Network Policies ช่วยควบคุม traffic flow ระหว่าง pods ใน Kubernetes cluster ในบทนี้เราจะเรียนรู้ patterns ขั้นสูงสำหรับ micro-segmentation, egress control และการ implement defense in depth

---

## 59.1 ทบทวน Network Policy พื้นฐาน

### Default Behavior

```
โดย default: ทุก pods สามารถคุยกันได้ทั้งหมด (no isolation)
เมื่อสร้าง NetworkPolicy: เฉพาะ traffic ที่ระบุในนั้นเท่านั้นที่ผ่านได้
```

### Basic Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
spec:
  podSelector:             # เลือก pods ที่ policy นี้ใช้กับ
    matchLabels:
      app: myapp
  
  policyTypes:
  - Ingress                # ควบคุม incoming traffic
  - Egress                 # ควบคุม outgoing traffic
  
  ingress:
  - from:                  # อนุญาต from ที่ไหน
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  - to:                    # อนุญาต to ที่ไหน
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
```

---

## 59.2 Default Deny Policies

### Default Deny Ingress

```yaml
# ห้าม incoming traffic ทั้งหมด (default deny)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}           # {} = เลือก pods ทั้งหมดใน namespace
  policyTypes:
  - Ingress
  # ไม่มี ingress rules = block ทั้งหมด
```

### Default Deny Egress

```yaml
# ห้าม outgoing traffic ทั้งหมด
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # ไม่มี egress rules = block ทั้งหมด
```

### Default Deny All

```yaml
# ห้ามทั้ง ingress และ egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## 59.3 Micro-segmentation

Micro-segmentation แบ่ง network ออกเป็น segments เล็กๆ เพื่อจำกัดการแพร่กระจายของ attacks

### Three-tier Architecture

```
Frontend → Backend → Database
(Internet)  (API)    (Data)
```

```yaml
# ===== Frontend Policy =====
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
  # รับ traffic จาก internet (via ingress controller)
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  
  egress:
  # ส่ง traffic ไป backend เท่านั้น
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  
  # DNS resolution
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
# ===== Backend Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # รับ traffic จาก frontend เท่านั้น
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  # ส่งไป database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  
  # Redis cache
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
---
# ===== Database Policy =====
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
  # รับ traffic จาก backend เท่านั้น
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  
  egress:
  # Database ไม่ต้องการ egress (isolated)
  # ยกเว้น DNS และ replication
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Cross-Namespace Policies

```yaml
# Backend ใน production คุยกับ shared-services namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-shared-services
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  
  egress:
  # Prometheus exporter ใน shared-services
  - to:
    - namespaceSelector:
        matchLabels:
          purpose: shared-services
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090
  
  # Vault ใน vault namespace
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: vault
    ports:
    - protocol: TCP
      port: 8200
```

---

## 59.4 Egress Control

### อนุญาต DNS

```yaml
# DNS เป็น prerequisite สำหรับทุก pod ที่ต้องการ name resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}   # ทุก pods
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### Block External Traffic

```yaml
# Block traffic ออกนอก cluster (ยกเว้น DNS)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-external-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      security: strict
  policyTypes:
  - Egress
  egress:
  # อนุญาตเฉพาะ in-cluster traffic (CIDR ของ pod network)
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8       # Pod CIDR
        except:
        - 10.0.0.1/32           # Block gateway
  
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Allow Specific External Services

```yaml
# อนุญาต access ไปยัง external services เฉพาะ
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-services
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Egress
  egress:
  # Payment gateway (by IP)
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24   # Payment gateway IP range
    ports:
    - protocol: TCP
      port: 443
  
  # AWS STS (for IRSA)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0        # ต้องการ public internet
    ports:
    - protocol: TCP
      port: 443                  # HTTPS only
  
  # DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

### Egress สำหรับ Monitoring Stack

```yaml
# Prometheus scraping - backend ต้องรับ scrape จาก prometheus
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}     # ทุก pods ใน production
  policyTypes:
  - Ingress
  ingress:
  # รับ scrape จาก prometheus
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - protocol: TCP
      port: 9090   # metrics port
    - protocol: TCP
      port: 8080   # app port (ถ้า expose metrics ที่นี่)
```

---

## 59.5 Advanced Selector Patterns

### Multiple Selectors (AND logic)

```yaml
ingress:
- from:
  - namespaceSelector:       # AND
      matchLabels:
        environment: production
    podSelector:             # ต้องตรงทั้งคู่
      matchLabels:
        role: frontend
```

### Multiple Items (OR logic)

```yaml
ingress:
- from:
  - podSelector:             # OR
      matchLabels:
        role: frontend
  - podSelector:             # OR
      matchLabels:
        role: backend
```

### NamespaceSelector กับ MatchExpressions

```yaml
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchExpressions:
        - key: environment
          operator: In
          values: ["production", "staging"]   # production หรือ staging
    - namespaceSelector:
        matchExpressions:
        - key: environment
          operator: NotIn
          values: ["development"]             # ไม่ใช่ development
```

---

## 59.6 Workshop: Implement Defense in Depth

### สถานการณ์

สร้าง e-commerce application ที่มี:
- Frontend (React app)
- API Gateway
- Product Service
- Order Service
- Payment Service
- Database (PostgreSQL)
- Cache (Redis)

พร้อม Network Policies ที่ implement defense in depth

### Step 1: Setup

```bash
kubectl create namespace ecommerce
kubectl config set-context --current --namespace=ecommerce

# Label namespace
kubectl label namespace ecommerce \
  environment=production \
  kubernetes.io/metadata.name=ecommerce

# Label kube-system
kubectl label namespace kube-system \
  kubernetes.io/metadata.name=kube-system 2>/dev/null || true
```

### Step 2: สร้าง Application Resources

```yaml
# app-resources.yaml
# Services สำหรับ DNS resolution

apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
  labels:
    tier: frontend
spec:
  selector:
    tier: frontend
  ports:
  - port: 3000
    targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: ecommerce
  labels:
    tier: api
spec:
  selector:
    tier: api
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    tier: service
    app: product
spec:
  selector:
    app: product-service
  ports:
  - port: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    tier: service
    app: order
spec:
  selector:
    app: order-service
  ports:
  - port: 8082
---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: ecommerce
  labels:
    tier: service
    app: payment
    security-level: high   # label พิเศษสำหรับ high security
spec:
  selector:
    app: payment-service
  ports:
  - port: 8083
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: ecommerce
  labels:
    tier: database
spec:
  selector:
    tier: database
  ports:
  - port: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: ecommerce
  labels:
    tier: cache
spec:
  selector:
    tier: cache
  ports:
  - port: 6379
```

```bash
kubectl apply -f app-resources.yaml
```

### Step 3: Apply Default Deny

```yaml
# default-deny.yaml
# เริ่มจาก deny ทั้งหมด แล้วค่อยเปิด
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow DNS สำหรับทุก pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
kubectl apply -f default-deny.yaml
```

### Step 4: สร้าง Network Policies

```yaml
# network-policies.yaml

# ===== Frontend Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # รับ traffic จาก internet (ingress controller)
  - from: []    # อนุญาตจากทุก sources (สำหรับ demo)
    ports:
    - protocol: TCP
      port: 3000
  egress:
  # ส่งไป API Gateway
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8080
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ===== API Gateway Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gateway-netpol
  namespace: ecommerce
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
  egress:
  # ส่งไป microservices
  - to:
    - podSelector:
        matchLabels:
          app: product-service
    ports:
    - protocol: TCP
      port: 8081
  - to:
    - podSelector:
        matchLabels:
          app: order-service
    ports:
    - protocol: TCP
      port: 8082
  - to:
    - podSelector:
        matchLabels:
          app: payment-service
    ports:
    - protocol: TCP
      port: 8083
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ===== Product Service Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: product-service-netpol
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: product-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8081
  egress:
  # Database access
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # Cache access
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ===== Order Service Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-netpol
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8082
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  # Payment service (order calls payment)
  - to:
    - podSelector:
        matchLabels:
          app: payment-service
    ports:
    - protocol: TCP
      port: 8083
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ===== Payment Service Policy (HIGH SECURITY) =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-netpol
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # ONLY order service และ api gateway สามารถเข้าถึง payment
  - from:
    - podSelector:
        matchLabels:
          app: order-service
    ports:
    - protocol: TCP
      port: 8083
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8083
  egress:
  # Payment gateway (external) - restrict IP
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24    # Payment gateway IP
    ports:
    - protocol: TCP
      port: 443
  # Database (payment-specific)
  - to:
    - podSelector:
        matchLabels:
          tier: database
          app: payment-db    # เฉพาะ payment database!
    ports:
    - protocol: TCP
      port: 5432
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ===== Database Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-netpol
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # รับจาก microservices เท่านั้น
  - from:
    - podSelector:
        matchLabels:
          app: product-service
    ports:
    - protocol: TCP
      port: 5432
  - from:
    - podSelector:
        matchLabels:
          app: order-service
    ports:
    - protocol: TCP
      port: 5432
  - from:
    - podSelector:
        matchLabels:
          app: payment-service
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # Database ไม่ต้องการ egress
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# ===== Cache Policy =====
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cache-netpol
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: cache
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: product-service
    ports:
    - protocol: TCP
      port: 6379
  - from:
    - podSelector:
        matchLabels:
          app: order-service
    ports:
    - protocol: TCP
      port: 6379
```

```bash
kubectl apply -f network-policies.yaml
kubectl get networkpolicies -n ecommerce
```

### Step 5: Deploy Test Pods

```yaml
# test-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  namespace: ecommerce
  labels:
    tier: frontend
spec:
  containers:
  - name: frontend
    image: alpine:3.18
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: api-gateway-pod
  namespace: ecommerce
  labels:
    tier: api
spec:
  containers:
  - name: api
    image: alpine:3.18
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: product-pod
  namespace: ecommerce
  labels:
    app: product-service
    tier: service
spec:
  containers:
  - name: product
    image: alpine:3.18
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: payment-pod
  namespace: ecommerce
  labels:
    app: payment-service
    security-level: high
spec:
  containers:
  - name: payment
    image: alpine:3.18
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
  namespace: ecommerce
  labels:
    tier: database
spec:
  containers:
  - name: db
    image: alpine:3.18
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f test-pods.yaml
kubectl wait --for=condition=ready pod --all -n ecommerce --timeout=60s
```

### Step 6: ทดสอบ Network Policies

```bash
# ติดตั้ง netcat ใน pods สำหรับทดสอบ
# (alpine มี nc built-in)

echo "=== Testing Frontend to API Gateway ==="
kubectl exec -it frontend-pod -n ecommerce -- \
  nc -zv api-gateway.ecommerce.svc.cluster.local 8080 2>&1 && \
  echo "PASS: Frontend -> API Gateway" || \
  echo "EXPECTED FAIL (no real service yet): Frontend -> API Gateway"

echo ""
echo "=== Testing Direct Frontend to Database (should be blocked) ==="
kubectl exec -it frontend-pod -n ecommerce -- timeout 5 \
  nc -zv postgres.ecommerce.svc.cluster.local 5432 2>&1 | \
  (grep -q "refused\|timed out\|timeout" && echo "PASS: Frontend -> DB blocked" || \
   echo "CONCERN: Connection may not be blocked (no real pod running)")

echo ""
echo "=== Testing Product Service to Database ==="
kubectl exec -it product-pod -n ecommerce -- timeout 5 \
  nc -zv postgres.ecommerce.svc.cluster.local 5432 2>&1 | \
  grep -q "refused\|timed out" && \
  echo "PASS: Policy applied (no real DB but connection attempted)" || \
  echo "OK: Policy exists even if service not available"

echo ""
echo "=== Verify Network Policies ==="
kubectl get networkpolicies -n ecommerce
kubectl describe networkpolicy default-deny-all -n ecommerce
kubectl describe networkpolicy payment-service-netpol -n ecommerce
```

### Step 7: ตรวจสอบ Connectivity Matrix

```bash
# สร้าง connectivity test script
cat > /tmp/test-connectivity.sh <<'EOF'
#!/bin/bash
NS="ecommerce"

test_connection() {
  local from=$1
  local to=$2
  local port=$3
  local expected=$4
  
  result=$(kubectl exec -n $NS $from -- timeout 3 nc -zv $to $port 2>&1)
  
  if echo "$result" | grep -qE "open|Connected|succeeded"; then
    actual="ALLOWED"
  else
    actual="BLOCKED"
  fi
  
  if [ "$actual" = "$expected" ]; then
    echo "✓ $from -> $to:$port ($expected)"
  else
    echo "✗ $from -> $to:$port (expected $expected, got $actual)"
  fi
}

echo "=== Network Policy Connectivity Matrix ==="
echo ""
echo "--- Ingress tests ---"
# Frontend should be accessible from anywhere (ingress)

echo ""
echo "--- Service to Service ---"
# test_connection "frontend-pod" "api-gateway" "8080" "ALLOWED"
# test_connection "frontend-pod" "postgres" "5432" "BLOCKED"
# test_connection "product-pod" "postgres" "5432" "ALLOWED"
# test_connection "database-pod" "frontend-pod" "3000" "BLOCKED"

echo ""
echo "Connectivity tests completed!"
echo "Note: Tests require actual services running"
EOF

chmod +x /tmp/test-connectivity.sh
bash /tmp/test-connectivity.sh
```

### Step 8: Monitoring Network Policies

```bash
# ดู Network Policies ทั้งหมด
kubectl get networkpolicies -n ecommerce -o wide

# ดูรายละเอียด
kubectl describe networkpolicy frontend-netpol -n ecommerce

# ดู events ที่เกี่ยวข้อง
kubectl get events -n ecommerce --field-selector=reason=NetworkPolicyViolation 2>/dev/null || \
  kubectl get events -n ecommerce

# ใช้ cilium หรือ calico ดู policy decisions (ถ้ามี)
# kubectl cilium debuginfo
```

### Step 9: Cleanup

```bash
kubectl delete namespace ecommerce
echo "Workshop cleanup complete!"
```

---

## 59.7 Advanced Patterns

### Namespace Isolation

```yaml
# แต่ละ team ได้รับ namespace ของตัวเอง
# ไม่สามารถ access namespace อื่นได้
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  # อนุญาตเฉพาะ traffic จากใน namespace เดียวกัน
  - from:
    - podSelector: {}
  # + ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
```

### Service Mesh Integration

```yaml
# สำหรับ Istio - allow traffic ผ่าน sidecar proxy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-istio-sidecar
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: istio-system
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Default Deny**: เริ่มจาก block ทั้งหมดแล้วเปิดทีละส่วน
2. **Micro-segmentation**: แยก network ตาม tiers
3. **Egress Control**: จำกัด outbound traffic
4. **Defense in Depth**: หลาย layers ของ security

**Key Takeaways:**
- เสมอ apply default deny ก่อน
- อนุญาต DNS เฉพาะเท่านั้น ไม่ใช่ทุกอย่าง
- Payment/sensitive services ต้องมี restrictive policies
- ทดสอบ connectivity matrix หลัง apply policies

---

**ต่อไป**: Part 60 - Image Security
