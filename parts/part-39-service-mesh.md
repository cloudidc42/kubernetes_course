# Part 39: Service Mesh

## สารบัญ
1. [Service Mesh คืออะไร](#service-mesh-overview)
2. [ปัญหาที่ Service Mesh แก้ไข](#problems-solved)
3. [Istio vs Linkerd vs Consul](#comparison)
4. [Service Mesh Architecture](#architecture)
5. [Workshop: ทดลอง Service Mesh Concepts](#workshop)

---

## 1. Service Mesh คืออะไร {#service-mesh-overview}

### ภาพรวม

Service Mesh คือ infrastructure layer ที่จัดการ network traffic ระหว่าง microservices โดยไม่ต้องเปลี่ยน code ของ application

```
โลกก่อน Service Mesh:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Service A ──────────────────────────── Service B          │
│                                                              │
│  ทุก Service ต้องจัดการเอง:                                │
│  ├── Retry logic                                            │
│  ├── Circuit breaker                                        │
│  ├── TLS/mTLS                                               │
│  ├── Observability (metrics, logs, traces)                  │
│  ├── Load balancing                                         │
│  └── Authentication/Authorization                          │
│                                                              │
│  ปัญหา: Code เยอะ, ซ้ำซ้อน, ยากดูแล                      │
└─────────────────────────────────────────────────────────────┘

โลกหลัง Service Mesh (Sidecar Pattern):
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌──────────────────┐      ┌──────────────────┐           │
│  │  Service A       │      │  Service B       │           │
│  │  ┌────────────┐  │      │  ┌────────────┐  │           │
│  │  │ App Code   │  │      │  │ App Code   │  │           │
│  │  └────────────┘  │      │  └────────────┘  │           │
│  │  ┌────────────┐  │      │  ┌────────────┐  │           │
│  │  │  Sidecar   │◄─┼──────┼──│  Sidecar   │  │           │
│  │  │  (Proxy)   │  │      │  │  (Proxy)   │  │           │
│  │  └────────────┘  │      │  └────────────┘  │           │
│  └──────────────────┘      └──────────────────┘           │
│                                                              │
│  Sidecar จัดการ:                                           │
│  ✅ mTLS ระหว่าง services                                   │
│  ✅ Traffic control (retry, timeout, circuit break)         │
│  ✅ Observability (metrics, tracing, logging)               │
│  ✅ Access control                                          │
│  App Code ไม่ต้องรู้อะไรเลย!                               │
└─────────────────────────────────────────────────────────────┘
```

### Sidecar Proxy Pattern

```
Pod Structure ด้วย Service Mesh:
┌─────────────────────────────────────────────────────────────┐
│                          Pod                                 │
│                                                              │
│  ┌──────────────────┐    ┌──────────────────┐              │
│  │  Application     │    │  Sidecar Proxy   │              │
│  │  Container       │    │  (Envoy/Linkerd)  │              │
│  │                  │    │                  │              │
│  │  Port: 8080      │    │  Inbound: 15006  │◄── Traffic   │
│  │  (app listen)    │    │  Outbound: 15001 │──► Traffic   │
│  └──────────────────┘    └──────────────────┘              │
│                                                              │
│  iptables rules redirect ALL traffic through sidecar        │
│  App thinks it's talking directly to other services         │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. ปัญหาที่ Service Mesh แก้ไข {#problems-solved}

### ปัญหา 1: Observability

```
ก่อน Service Mesh:
Service A ──► Service B ──► Service C
                 (ล้มเหลว!)
                 
คำถาม:
- Request ใช้เวลาเท่าไหร่ในแต่ละ service?
- มี error rate เท่าไหร่?
- Request ผ่านเส้นทางไหน?

หลัง Service Mesh:
- Distributed Tracing (Jaeger/Zipkin)
- Metrics per service (request rate, latency, errors)
- Service dependency map
```

```
Distributed Trace Example:
Request ID: abc-123
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Frontend (50ms)                                      │
│  └── API Gateway (30ms)                              │
│       ├── User Service (10ms) ✅                     │
│       ├── Product Service (15ms) ✅                  │
│       └── Order Service (5ms → FAILED ❌)            │
│            └── Database (TIMEOUT)                    │
└────────────────────────────────────────────────────────┘
```

### ปัญหา 2: Traffic Management

```
Traffic Splitting (A/B Testing / Canary):

User Request
    │
    ▼
Service Mesh
    │
    ├── 90% ──► Service v1 (Stable)
    └── 10% ──► Service v2 (Canary)

ไม่ต้องแก้ code ใดๆ!
```

### ปัญหา 3: Security (mTLS)

```
ปัญหา: ไม่มี encryption ระหว่าง services
Service A ──────plain text──────► Service B
          (ถูก intercept ได้!)

Service Mesh Solution (mTLS):
Service A (cert A) ◄──── encrypted + auth ────► Service B (cert B)

Automatic certificate management:
- Issue certificates via CA
- Rotate certificates automatically
- Verify identity before allowing connection
```

### ปัญหา 4: Resilience

```
Circuit Breaker Pattern:
                              ┌─────────────────────┐
Service A ──► Sidecar ──────► │  Service B          │
                   │          │  (unhealthy!)        │
                   │          └─────────────────────┘
                   │
                   │ Circuit Breaker OPEN
                   │ (ไม่ส่ง request ไป Service B)
                   │
                   └──► Fast Fail / Fallback Response

ป้องกัน Cascade Failures!
```

---

## 3. Istio vs Linkerd vs Consul {#comparison}

### เปรียบเทียบ Service Meshes

```
┌──────────────────────────────────────────────────────────────────┐
│                    Service Mesh Comparison                        │
├────────────────┬──────────────┬──────────────┬───────────────────┤
│ Feature        │ Istio        │ Linkerd      │ Consul Connect    │
├────────────────┼──────────────┼──────────────┼───────────────────┤
│ Complexity     │ High         │ Low          │ Medium            │
│ Performance    │ Medium       │ High         │ Medium            │
│ Resource Use   │ High         │ Low          │ Medium            │
│ mTLS           │ ✅           │ ✅ Auto      │ ✅                │
│ Traffic Mgmt   │ ✅ Advanced  │ ✅ Basic     │ ✅ Medium         │
│ Observability  │ ✅ Full      │ ✅ Full      │ ✅ Full           │
│ Multi-cluster  │ ✅           │ ✅           │ ✅                │
│ Non-K8s        │ Limited      │ Limited      │ ✅ Native         │
│ Sidecar Proxy  │ Envoy        │ Rust Proxy   │ Envoy             │
│ CRDs           │ Many         │ Few          │ Fewer             │
│ Learning Curve │ Steep        │ Gentle       │ Medium            │
│ Community      │ Very Large   │ Large        │ Large             │
│ CNCF Status    │ Graduated    │ Graduated    │ N/A               │
└────────────────┴──────────────┴──────────────┴───────────────────┘
```

### เมื่อไหรควรใช้อะไร

```
Istio:
├── Complex traffic management ที่ต้องการ
├── Advanced security policies
├── Large organization ที่ต้องการ feature ครบ
└── ทีมที่มี expertise สูง

Linkerd:
├── ต้องการ simplicity และ performance
├── Small/Medium team
├── Rust-based proxy (safer memory)
└── Quick setup for basic features

Consul:
├── Multi-cloud / Hybrid environments
├── Non-Kubernetes workloads (VMs)
├── HashiCorp ecosystem
└── Service Catalog + Mesh
```

---

## 4. Service Mesh Architecture {#architecture}

### Control Plane vs Data Plane

```
┌──────────────────────────────────────────────────────────────┐
│                     Control Plane                            │
│                                                              │
│  Istio: istiod (Pilot, Citadel, Galley)                    │
│  Linkerd: control-plane pods                               │
│  Consul: consul servers                                    │
│                                                              │
│  จัดการ:                                                    │
│  ├── Certificate management                                │
│  ├── Configuration distribution                            │
│  ├── Service discovery                                     │
│  └── Policy enforcement                                    │
└──────────────────────┬───────────────────────────────────────┘
                       │ xDS API
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                      Data Plane                              │
│                                                              │
│  Sidecar Proxies (Envoy/Linkerd-proxy/Envoy)               │
│                                                              │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐       │
│  │ Pod A      │    │ Pod B      │    │ Pod C      │       │
│  │ ┌────────┐ │    │ ┌────────┐ │    │ ┌────────┐ │       │
│  │ │App     │ │    │ │App     │ │    │ │App     │ │       │
│  │ └────────┘ │    │ └────────┘ │    │ └────────┘ │       │
│  │ ┌────────┐ │    │ ┌────────┐ │    │ ┌────────┐ │       │
│  │ │Sidecar │◄┼────┼►│Sidecar │◄┼────┼►│Sidecar │ │       │
│  │ └────────┘ │    │ └────────┘ │    │ └────────┘ │       │
│  └────────────┘    └────────────┘    └────────────┘       │
│                                                              │
│  จัดการ:                                                    │
│  ├── Actual traffic forwarding                             │
│  ├── mTLS encryption/decryption                            │
│  ├── Metrics collection                                    │
│  └── Health checking                                       │
└──────────────────────────────────────────────────────────────┘
```

### Envoy Proxy (ใช้ใน Istio & Consul)

```
Envoy Architecture:
┌──────────────────────────────────────────────────────────────┐
│                        Envoy Proxy                           │
│                                                              │
│  Listeners (inbound)                                        │
│  ├── HTTP/HTTPS                                            │
│  ├── TCP                                                   │
│  └── gRPC                                                  │
│                ↓                                            │
│  Filters                                                    │
│  ├── HTTP Connection Manager                               │
│  ├── Router                                                │
│  ├── Rate Limit                                            │
│  └── Authorization                                        │
│                ↓                                            │
│  Clusters (outbound)                                       │
│  ├── Service A → [Pod IPs]                                │
│  ├── Service B → [Pod IPs]                                │
│  └── Service C → [Pod IPs]                                │
│                                                              │
│  xDS APIs (receive config from Control Plane):             │
│  ├── CDS - Cluster Discovery Service                       │
│  ├── EDS - Endpoint Discovery Service                      │
│  ├── LDS - Listener Discovery Service                      │
│  ├── RDS - Route Discovery Service                        │
│  └── SDS - Secret Discovery Service                       │
└──────────────────────────────────────────────────────────────┘
```

### Linkerd Proxy (Rust-based)

```
Linkerd Proxy Architecture:
┌──────────────────────────────────────────────────────────────┐
│                     linkerd-proxy (Rust)                      │
│                                                              │
│  Inbound:                                                    │
│  ├── Accept connections                                     │
│  ├── mTLS termination                                       │
│  ├── HTTP/1.1, HTTP/2, gRPC                                │
│  └── Metrics collection                                     │
│                                                              │
│  Outbound:                                                   │
│  ├── Load balancing (EWMA)                                  │
│  ├── Retries with budget                                    │
│  ├── Timeouts                                               │
│  ├── mTLS origination                                       │
│  └── Metrics collection                                     │
│                                                              │
│  ข้อดี: ใช้ memory น้อยกว่า Envoy, latency ต่ำกว่า         │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Workshop: ทดลอง Service Mesh Concepts {#workshop}

### Workshop 1: จำลอง Service Mesh ด้วย Manual Proxy

เพื่อเข้าใจ concept โดยไม่ติดตั้ง full service mesh

```bash
# สร้าง cluster สำหรับ workshop
kind create cluster --name mesh-concepts

# สร้าง test application
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: mesh-demo
---
# Service A (Client)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
  namespace: mesh-demo
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
      - name: app
        image: nicolaka/netshoot
        command: ["sleep", "infinity"]
---
# Service B (Server)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
  namespace: mesh-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Service B Response: OK"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
  namespace: mesh-demo
spec:
  selector:
    app: service-b
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl wait --for=condition=available deployment/service-a -n mesh-demo --timeout=60s
kubectl wait --for=condition=available deployment/service-b -n mesh-demo --timeout=60s
```

### Workshop 2: จำลอง Observability (Manual)

```bash
# ทดสอบ basic connectivity
SERVICE_A_POD=$(kubectl get pod -n mesh-demo -l app=service-a -o name | head -1)

# Simple request without observability
kubectl exec -n mesh-demo $SERVICE_A_POD -- curl -s http://service-b/

# จำลอง request tracking (ส่ง header ด้วยตัวเอง)
kubectl exec -n mesh-demo $SERVICE_A_POD -- \
  curl -s -H "X-Request-ID: req-001" \
  -H "X-Trace-ID: trace-abc" \
  http://service-b/

# Loop requests เพื่อดู patterns
for i in {1..10}; do
  kubectl exec -n mesh-demo $SERVICE_A_POD -- \
    curl -s -w "\nTime: %{time_total}s\nHTTP Status: %{http_code}\n" \
    http://service-b/ 2>&1
  echo "---"
done
```

### Workshop 3: จำลอง Circuit Breaker

```yaml
# circuit-breaker-demo.yaml
# Service ที่จำลองการ fail
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unstable-service
  namespace: mesh-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: unstable-service
  template:
    metadata:
      labels:
        app: unstable-service
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: config
        configMap:
          name: unstable-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: unstable-config
  namespace: mesh-demo
data:
  default.conf: |
    server {
      listen 80;
      location / {
        # 50% chance of error (จำลอง)
        if ($request_id ~* "[02468]$") {
          return 503 "Service Unavailable";
        }
        return 200 "OK";
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: unstable-service
  namespace: mesh-demo
spec:
  selector:
    app: unstable-service
  ports:
  - port: 80
```

```bash
kubectl apply -f circuit-breaker-demo.yaml
kubectl wait --for=condition=available deployment/unstable-service -n mesh-demo --timeout=60s

# จำลอง circuit breaker logic ด้วย shell script
kubectl exec -n mesh-demo $SERVICE_A_POD -- bash << 'SCRIPT'
#!/bin/bash

FAILURE_COUNT=0
CIRCUIT_OPEN=false
RESET_TIMEOUT=10
LAST_OPEN_TIME=0

for i in {1..30}; do
  echo "Request $i:"
  
  if $CIRCUIT_OPEN; then
    CURRENT_TIME=$(date +%s)
    if [ $((CURRENT_TIME - LAST_OPEN_TIME)) -gt $RESET_TIMEOUT ]; then
      echo "  Circuit HALF-OPEN: Testing..."
      CIRCUIT_OPEN=false
      FAILURE_COUNT=0
    else
      echo "  Circuit OPEN: Fast fail! (skip request)"
      sleep 0.5
      continue
    fi
  fi
  
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 2 http://unstable-service/ 2>/dev/null)
  
  if [ "$STATUS" == "200" ]; then
    echo "  SUCCESS (HTTP $STATUS)"
    FAILURE_COUNT=0
  else
    echo "  FAILURE (HTTP $STATUS)"
    FAILURE_COUNT=$((FAILURE_COUNT + 1))
    
    if [ $FAILURE_COUNT -ge 3 ]; then
      echo "  ⚡ CIRCUIT BREAKER OPENED! (3 consecutive failures)"
      CIRCUIT_OPEN=true
      LAST_OPEN_TIME=$(date +%s)
    fi
  fi
  
  sleep 0.5
done
SCRIPT
```

### Workshop 4: จำลอง mTLS

```bash
# สร้าง certificates สำหรับ services
mkdir -p /tmp/certs

# สร้าง CA
openssl genrsa -out /tmp/certs/ca.key 2048
openssl req -x509 -new -nodes -key /tmp/certs/ca.key \
  -subj "/CN=mesh-ca" -days 365 -out /tmp/certs/ca.crt

# Certificate สำหรับ Service A
openssl genrsa -out /tmp/certs/service-a.key 2048
openssl req -new -key /tmp/certs/service-a.key \
  -subj "/CN=service-a.mesh-demo.svc.cluster.local" \
  -out /tmp/certs/service-a.csr
openssl x509 -req -in /tmp/certs/service-a.csr \
  -CA /tmp/certs/ca.crt -CAkey /tmp/certs/ca.key \
  -CAcreateserial -out /tmp/certs/service-a.crt -days 365

# Certificate สำหรับ Service B
openssl genrsa -out /tmp/certs/service-b.key 2048
openssl req -new -key /tmp/certs/service-b.key \
  -subj "/CN=service-b.mesh-demo.svc.cluster.local" \
  -out /tmp/certs/service-b.csr
openssl x509 -req -in /tmp/certs/service-b.csr \
  -CA /tmp/certs/ca.crt -CAkey /tmp/certs/ca.key \
  -CAcreateserial -out /tmp/certs/service-b.crt -days 365

# สร้าง Kubernetes Secrets
kubectl create secret generic service-a-cert \
  -n mesh-demo \
  --from-file=tls.crt=/tmp/certs/service-a.crt \
  --from-file=tls.key=/tmp/certs/service-a.key \
  --from-file=ca.crt=/tmp/certs/ca.crt

kubectl create secret generic service-b-cert \
  -n mesh-demo \
  --from-file=tls.crt=/tmp/certs/service-b.crt \
  --from-file=tls.key=/tmp/certs/service-b.key \
  --from-file=ca.crt=/tmp/certs/ca.crt

echo "Certificates created!"
echo "ใน Service Mesh จริง: ทั้งหมดนี้ automated โดย control plane"
```

### Workshop 5: Linkerd Quick Demo

```bash
# ติดตั้ง Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
linkerd version

# ตรวจสอบ cluster compatibility
linkerd check --pre

# ติดตั้ง Linkerd CRDs
linkerd install --crds | kubectl apply -f -

# ติดตั้ง Linkerd control plane
linkerd install | kubectl apply -f -

# ตรวจสอบ
linkerd check

# ดู Control Plane
kubectl get pods -n linkerd
```

```bash
# Inject Linkerd sidecar ใน namespace
kubectl annotate namespace mesh-demo \
  linkerd.io/inject=enabled

# Restart deployments เพื่อ inject sidecar
kubectl rollout restart deployment -n mesh-demo

# ตรวจสอบ sidecar injection
kubectl get pods -n mesh-demo
# ควรเห็น 2/2 READY แทน 1/1

# ดู mesh status
linkerd -n mesh-demo stat deployments

# ดู traffic
linkerd -n mesh-demo tap deploy/service-a

# Dashboard
linkerd dashboard &
```

### Workshop 6: Traffic Monitoring กับ Linkerd

```bash
# สร้าง traffic
SERVICE_A_POD=$(kubectl get pod -n mesh-demo -l app=service-a -o name | head -1)

# Generate traffic
for i in {1..50}; do
  kubectl exec -n mesh-demo $SERVICE_A_POD -- curl -s http://service-b/ > /dev/null
  sleep 0.1
done

# ดู statistics
linkerd -n mesh-demo stat deployments
# OUTPUT:
# NAME        MESHED  SUCCESS     RPS  LATENCY_P50  LATENCY_P95  LATENCY_P99  TCP_CONN
# service-a   1/1    100.00%   1.5rps       1ms          2ms          3ms         1
# service-b   2/2    100.00%   1.5rps       1ms          3ms          5ms         2

# ดู per-route stats
linkerd -n mesh-demo stat deployments --from deploy/service-a

# Real-time tap
linkerd -n mesh-demo tap deploy/service-b --namespace=mesh-demo
```

### Workshop 7: ทดลอง Retry Policy

```yaml
# retry-policy-demo.yaml
# Linkerd ServiceProfile สำหรับ retry
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: service-b.mesh-demo.svc.cluster.local
  namespace: mesh-demo
spec:
  routes:
  - name: GET /
    condition:
      method: GET
      pathRegex: /.*
    responseClasses:
    - condition:
        status:
          min: 500
          max: 599
      isFailure: true
    retryBudget:
      retryRatio: 0.2      # 20% retry budget
      minRetriesPerSecond: 10
      ttl: 10s
    timeout: 500ms
```

```bash
kubectl apply -f retry-policy-demo.yaml

# ทดสอบ retry
kubectl exec -n mesh-demo $SERVICE_A_POD -- bash -c "
for i in {1..20}; do
  curl -s -w '%{http_code}\n' -o /dev/null http://service-b/
done"

# ดู retry statistics
linkerd -n mesh-demo stat routes -t service-b.mesh-demo.svc.cluster.local
```

### Workshop 8: Compare With/Without Service Mesh

```bash
# สร้าง summary comparison
echo "=== Without Service Mesh ==="
echo "Communication: Plain HTTP (unencrypted)"
echo "Observability: None (app must implement)"
echo "Retry: None (app must implement)"
echo "Circuit Breaker: None (app must implement)"
echo ""

echo "=== With Linkerd Service Mesh ==="
echo "Communication: mTLS (automatic)"

# ดู mTLS status
linkerd -n mesh-demo edges deployment

echo ""
echo "Metrics:"
linkerd -n mesh-demo stat deployments
```

### Cleanup

```bash
# Uninstall Linkerd
linkerd install --ignore-cluster | kubectl delete -f -
kubectl delete namespace mesh-demo
kind delete cluster --name mesh-concepts
```

---

## Service Mesh Trade-offs

```
Pros:
✅ Observability ทันที (metrics, traces, logs)
✅ mTLS automatic (zero code change)
✅ Traffic management (retry, timeout, circuit break)
✅ Gradual rollout (canary, A/B testing)
✅ Policy enforcement
✅ Service discovery integration

Cons:
❌ Added complexity
❌ Increased resource usage (sidecar overhead)
❌ Additional latency (microseconds to milliseconds)
❌ Debugging complexity (another layer to troubleshoot)
❌ Learning curve
❌ Cold start time (sidecar injection)
```

---

## เมื่อไหรควรใช้ Service Mesh?

```
ควรใช้เมื่อ:
├── มี 10+ microservices
├── ต้องการ mTLS ระหว่าง services
├── ต้องการ distributed tracing
├── ต้องการ fine-grained traffic control
├── Multi-team development
└── Compliance/Security requirements

ยังไม่ต้องใช้เมื่อ:
├── Monolith หรือ few services
├── Team ยังเล็ก
├── Resource จำกัด
├── Simple architecture
└── ยังไม่มี observability requirements
```

---

## Cheat Sheet

```bash
# Linkerd
linkerd check
linkerd install | kubectl apply -f -
linkerd inject - | kubectl apply -f -
kubectl annotate namespace default linkerd.io/inject=enabled
linkerd -n <namespace> stat deployments
linkerd -n <namespace> tap deploy/<name>
linkerd -n <namespace> edges deployment
linkerd dashboard

# Istio (ถัดไป)
istioctl install
kubectl label namespace default istio-injection=enabled
istioctl analyze
istioctl proxy-status
istioctl dashboard kiali

# ดู sidecar status
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{","}{end}{"\n"}{end}'
```

---

## Service Mesh Monitoring

### Key Metrics ที่ควร Monitor

```
Service Mesh สำคัญ ที่ควรดู (The Four Golden Signals):

1. Latency (ความเร็ว)
   ├── P50 (median) latency
   ├── P95 latency  
   ├── P99 latency
   └── P99.9 latency (tail latency)

2. Traffic (ปริมาณ)
   ├── Requests per second (RPS)
   ├── Bytes per second
   └── Connection count

3. Errors (ข้อผิดพลาด)
   ├── Error rate (%)
   ├── 5xx errors
   ├── 4xx errors
   └── TCP connection errors

4. Saturation (ความอิ่มตัว)
   ├── CPU usage
   ├── Memory usage
   └── Connection pool utilization
```

### Linkerd Dashboard

```bash
# เปิด Linkerd Dashboard
linkerd dashboard &

# URL: http://localhost:50750/namespaces/default

# Metrics ที่เห็นใน Dashboard:
# - Success Rate per deployment
# - RPS (Requests Per Second)
# - Latency (P50, P95, P99)
# - Service dependencies map
```

### Prometheus Queries สำหรับ Service Mesh

```promql
# Request rate
sum(rate(request_total{namespace="default"}[5m])) by (deployment)

# Error rate
sum(rate(request_total{namespace="default", status_code=~"5.."}[5m])) 
/ 
sum(rate(request_total{namespace="default"}[5m]))

# P99 latency
histogram_quantile(0.99, sum(rate(response_latency_ms_bucket{namespace="default"}[5m])) by (le, deployment))

# Active connections
sum(tcp_open_connections{namespace="default"}) by (deployment)
```

### Service Topology Visualization

```
Kiali Service Graph:
┌─────────────────────────────────────────────────────────┐
│                    Service Mesh Map                      │
│                                                          │
│  [user] ──► [gateway] ──► [frontend]                   │
│                                  │                       │
│                                  ├──► [api-service]      │
│                                  │         │             │
│                                  │         ├──► [db]     │
│                                  │         └──► [cache]  │
│                                  └──► [static-cdn]       │
│                                                          │
│  Color coding:                                          │
│  Green  = Healthy (>99% success)                        │
│  Yellow = Warning (95-99% success)                      │
│  Red    = Error (<95% success)                          │
│  Blue   = VirtualService applied                        │
└─────────────────────────────────────────────────────────┘
```

---

## Common Service Mesh Patterns

### Pattern 1: Blue-Green Deployment

```
Blue-Green ด้วย Service Mesh:

Traffic:
100% ──► Blue (v1)  ← current production
  0% ──► Green (v2) ← new version (testing)

ทดสอบ Green เสร็จ:
  0% ──► Blue (v1)
100% ──► Green (v2)  ← switch instantly

Rollback:
100% ──► Blue (v1)  ← instant rollback
  0% ──► Green (v2)
```

### Pattern 2: Progressive Delivery

```
Progressive Delivery (Canary):

Week 1:  95% → v1,  5% → v2  (observe metrics)
Week 2:  80% → v1, 20% → v2  (if metrics ok)
Week 3:  50% → v1, 50% → v2  (continuing good)
Week 4:   0% → v1,100% → v2  (full rollout)

หากมีปัญหา: rollback ทันที → 100% → v1
```

### Pattern 3: Dark Launch (Shadowing)

```
Traffic Mirroring:

User ──► Service v1 (ตอบ user)
              │
              └──► Service v2 (copy ของ request)
                   (ไม่ตอบ user, แค่ test)

ประโยชน์:
- ทดสอบ v2 ด้วย production traffic จริง
- ไม่กระทบ user
- วัด performance difference
```

---

*ก่อนหน้า: [Part 38 - DNS](./part-38-dns.md)*
*ต่อไป: [Part 40 - Istio](./part-40-istio.md)*

---

## 7. Service Mesh Data Plane vs Control Plane {#data-control-plane}

### 7.1 ภาพรวม Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                Service Mesh Architecture                         │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    CONTROL PLANE                          │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │   │
│  │  │   Istiod    │  │  Linkerd     │  │  Consul       │  │   │
│  │  │  (Pilot)    │  │  Control     │  │  Server       │  │   │
│  │  │             │  │  Plane       │  │               │  │   │
│  │  │ - Config    │  │ - destiny    │  │ - Service     │  │   │
│  │  │ - Discovery │  │ - identity   │  │   Registry    │  │   │
│  │  │ - TLS certs │  │ - proxy-     │  │ - Health      │  │   │
│  │  │ - Policies  │  │   injector   │  │   Checks      │  │   │
│  │  └─────────────┘  └──────────────┘  └───────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                   xDS API / gRPC push                            │
│                              │                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     DATA PLANE                            │   │
│  │                                                           │   │
│  │  Pod A                          Pod B                     │   │
│  │  ┌─────────────────────┐        ┌─────────────────────┐  │   │
│  │  │  App Container      │        │  App Container      │  │   │
│  │  │    :8080            │        │    :8080            │  │   │
│  │  ├─────────────────────┤        ├─────────────────────┤  │   │
│  │  │  Sidecar Proxy      │───────►│  Sidecar Proxy      │  │   │
│  │  │  (Envoy/linkerd2)   │        │  (Envoy/linkerd2)   │  │   │
│  │  │  :15001 (Outbound)  │        │  :15006 (Inbound)   │  │   │
│  │  └─────────────────────┘        └─────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Control Plane ทำงานอย่างไร

```
xDS API (Envoy Discovery Service) ที่ Istio Control Plane ใช้:

1. LDS (Listener Discovery Service)
   - กำหนด ports ที่ proxy จะ listen
   - เช่น: listen on :15001 สำหรับ outbound

2. RDS (Route Discovery Service)
   - กำหนด HTTP routing rules
   - path-based, header-based routing

3. CDS (Cluster Discovery Service)
   - กำหนด upstream clusters (destination services)
   - load balancing policy

4. EDS (Endpoint Discovery Service)
   - กำหนด healthy endpoints ใน cluster
   - ได้จาก Kubernetes endpoints/endpointslices

5. SDS (Secret Discovery Service)
   - distribute TLS certificates
   - certificate rotation
```

```bash
# ดู xDS configuration ที่ proxy ได้รับ
# สำหรับ Istio
istioctl proxy-config listener my-pod.default
istioctl proxy-config route my-pod.default
istioctl proxy-config cluster my-pod.default
istioctl proxy-config endpoint my-pod.default

# ดู proxy status
istioctl proxy-status

# Debug specific proxy
istioctl dashboard envoy my-pod.default
```

### 7.3 Data Plane - Sidecar Proxy

```bash
# Envoy Proxy Statistics
kubectl exec -it my-pod -c istio-proxy -- \
  curl -s http://localhost:15000/stats | grep -E "upstream_cx|rq_total"

# ดู Envoy configuration dump
kubectl exec -it my-pod -c istio-proxy -- \
  curl -s http://localhost:15000/config_dump | python3 -m json.tool | head -100

# ดู health ของ proxy
kubectl exec -it my-pod -c istio-proxy -- \
  curl -s http://localhost:15000/server_info

# ดู upstream connections
kubectl exec -it my-pod -c istio-proxy -- \
  curl -s http://localhost:15000/clusters | grep -E "cx_active|rq_active"

# Hot restart proxy (reload config without downtime)
kubectl exec -it my-pod -c istio-proxy -- \
  kill -USR1 $(pgrep envoy)
```

---

## 8. Sidecar Injection เชิงลึก {#sidecar-injection}

### 8.1 กลไก Automatic Injection

```
Sidecar Injection Flow:

1. User: kubectl apply pod.yaml
2. API Server: ได้รับ Pod spec
3. Mutating Webhook:
   - API Server เรียก istiod/linkerd-proxy-injector webhook
   - Webhook เพิ่ม sidecar container ใน Pod spec
   - Webhook เพิ่ม init container (traffic redirection)
4. API Server: เก็บ Pod spec ที่ modified แล้ว
5. Kubelet: สร้าง Pod พร้อม sidecar
```

### 8.2 Namespace Level Injection

```yaml
# enable-injection.yaml - เปิด injection สำหรับ namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled        # Istio
    linkerd.io/inject: enabled      # Linkerd

---
# ปิด injection สำหรับ namespace
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    istio-injection: disabled
```

### 8.3 Pod Level Injection Control

```yaml
# pod-with-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    sidecar.istio.io/inject: "true"      # Force inject (แม้ namespace ปิด)
    # หรือ
    sidecar.istio.io/inject: "false"     # Disable inject (แม้ namespace เปิด)
    
    # Linkerd
    linkerd.io/inject: enabled           # Force inject
    linkerd.io/inject: disabled          # Disable inject
    
    # Custom sidecar config
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "500m"
    sidecar.istio.io/proxyMemoryLimit: "256Mi"
    
    # Exclude ports from interception
    traffic.sidecar.istio.io/excludeOutboundPorts: "3306,6379"
    traffic.sidecar.istio.io/excludeInboundPorts: "9090"
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
```

### 8.4 Manual Sidecar Injection

```bash
# Inject sidecar ด้วย kubectl inject
kubectl get deployment my-app -o yaml | \
  istioctl kube-inject -f - | \
  kubectl apply -f -

# หรือ inject ไฟล์โดยตรง
istioctl kube-inject -f deployment.yaml | kubectl apply -f -

# ตรวจสอบว่า sidecar inject แล้ว
kubectl describe pod my-pod | grep -A3 "Init Containers"
kubectl describe pod my-pod | grep "istio-proxy"

# ดู containers ใน pod
kubectl get pod my-pod -o jsonpath='{.spec.containers[*].name}'
# Output: my-app istio-proxy

# ดู init containers
kubectl get pod my-pod -o jsonpath='{.spec.initContainers[*].name}'
# Output: istio-init
```

### 8.5 Sidecar Resource

```yaml
# sidecar-resource.yaml - ควบคุม traffic ที่ sidecar intercept
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
  namespace: production
spec:
  egress:
  - hosts:
    - "./*"                      # Services ใน namespace เดียวกัน
    - "istio-system/*"           # Istio system services
    - "production/*"             # Services ใน production namespace
    # จำกัดเฉพาะที่ต้องการ (ลด memory ของ proxy)
    # ไม่ต้องรู้ routes ทั้ง cluster
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY           # Block traffic ที่ไม่ได้ register

---
# Sidecar สำหรับ specific workload
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: api-sidecar
  namespace: production
spec:
  workloadSelector:
    labels:
      app: api-server
  egress:
  - hosts:
    - "production/database-service"
    - "production/cache-service"
    - "istio-system/kiali"
```

---

## 9. Linkerd - ติดตั้งและใช้งาน {#linkerd}

### 9.1 ทำไมต้อง Linkerd

```
Linkerd vs Istio เปรียบเทียบ:

Feature          Linkerd          Istio
─────────────────────────────────────────
Complexity       Low              High
Memory/Pod       ~50MB            ~250MB
Latency Added    ~0.4ms           ~1ms
mTLS             Auto (default)   Opt-in
Configuration    Minimal          Extensive
Learning Curve   Easy             Steep
Control Plane    4 components     Istiod
Data Plane       linkerd2-proxy   Envoy
Protocol         HTTP, gRPC       HTTP, gRPC, TCP
Language         Rust             C++
```

### 9.2 ติดตั้ง Linkerd

```bash
# Step 1: ติดตั้ง Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# หรือ download version เฉพาะ
LINKERD2_VERSION=stable-2.14.5
curl -sL "https://github.com/linkerd/linkerd2/releases/download/${LINKERD2_VERSION}/linkerd2-cli-${LINKERD2_VERSION}-linux-amd64" \
  -o /usr/local/bin/linkerd
chmod +x /usr/local/bin/linkerd

# Step 2: ตรวจสอบ cluster compatibility
linkerd check --pre

# Step 3: ติดตั้ง Linkerd CRDs
linkerd install --crds | kubectl apply -f -

# Step 4: ติดตั้ง Linkerd control plane
linkerd install | kubectl apply -f -

# Step 5: รอ control plane ready
linkerd check

# Step 6: ติดตั้ง Linkerd Viz extension (dashboard)
linkerd viz install | kubectl apply -f -
linkerd viz check
```

### 9.3 Linkerd Components

```bash
# ดู components ที่ติดตั้ง
kubectl get pods -n linkerd
# linkerd-destination-xxxx     (service discovery)
# linkerd-identity-xxxx        (mTLS certificates)
# linkerd-proxy-injector-xxxx  (sidecar injection webhook)

kubectl get pods -n linkerd-viz
# grafana-xxxx          (metrics dashboard)
# metrics-api-xxxx      (metrics aggregation)
# prometheus-xxxx       (metrics storage)
# tap-xxxx              (live traffic inspection)
# web-xxxx              (Linkerd dashboard)
```

### 9.4 Inject Linkerd sidecar

```bash
# เปิด injection สำหรับ namespace
kubectl annotate namespace production \
  linkerd.io/inject=enabled

# Inject ไปยัง existing deployment
kubectl get deployment my-app -n production -o yaml | \
  linkerd inject - | \
  kubectl apply -f -

# ตรวจสอบ injection
linkerd check --proxy -n production

# ดู pods ที่ inject แล้ว
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{" - containers:"}{range .spec.containers[*]}{.name}{","}{end}{"\n"}{end}'
```

### 9.5 Linkerd YAML Configuration

```yaml
# linkerd-inject-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  annotations:
    linkerd.io/inject: enabled          # Enable injection
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
      annotations:
        linkerd.io/inject: enabled
        # Config proxy resources
        config.linkerd.io/proxy-cpu-request: "10m"
        config.linkerd.io/proxy-memory-request: "20Mi"
        config.linkerd.io/proxy-cpu-limit: "1000m"
        config.linkerd.io/proxy-memory-limit: "250Mi"
        # Opaque ports (ไม่ intercept - ใช้สำหรับ non-HTTP)
        config.linkerd.io/opaque-ports: "3306,6379,27017"
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80

---
# HTTPRoute สำหรับ Traffic Management
apiVersion: policy.linkerd.io/v1beta2
kind: HTTPRoute
metadata:
  name: web-routes
  namespace: production
spec:
  parentRefs:
  - name: web-service
    kind: Service
    group: core
    port: 80
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
      weight: 100
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
      weight: 100

---
# TrafficSplit สำหรับ Canary Release
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: web-app-split
  namespace: production
spec:
  service: web-service
  backends:
  - service: web-app-stable
    weight: 90
  - service: web-app-canary
    weight: 10

---
# Server Policy (mTLS)
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: web-server
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  port: 80
  proxyProtocol: HTTP/1

---
# ServerAuthorization
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: web-authz
  namespace: production
spec:
  server:
    name: web-server
  client:
    meshTLS:
      serviceAccounts:
      - name: frontend-sa
        namespace: production
      - name: api-sa
        namespace: production
```

### 9.6 Linkerd CLI Commands

```bash
# ดู traffic stats
linkerd viz stat deployments -n production
linkerd viz stat pods -n production
linkerd viz stat services -n production

# ดู real-time traffic
linkerd viz tap deployment/web-app -n production
linkerd viz tap pod/web-app-xxxx -n production \
  --to service/api-service

# ดู top routes
linkerd viz top deployment/web-app -n production

# ดู edges (connections)
linkerd viz edges deployments -n production

# Check mTLS
linkerd viz edges pods -n production | grep -v SECURED

# Open Dashboard
linkerd viz dashboard &

# ดู Grafana
linkerd viz dashboard --show grafana

# Diagnose
linkerd diagnostics -n production
linkerd viz stat ns

# ดู certificate expiry
linkerd check --proxy

# Rotate certificates (ถ้า cert ใกล้หมด)
linkerd upgrade | kubectl apply -f -
```

### 9.7 Linkerd mTLS Verification

```bash
# ตรวจสอบว่า traffic ใช้ mTLS
linkerd viz tap deployment/frontend -n production \
  --to deployment/backend \
  --output json | \
  jq '.destinationTLSStatus'

# ดู certificate info
kubectl exec -n production deploy/web-app -c linkerd-proxy -- \
  openssl s_client -connect localhost:4143 -showcerts 2>/dev/null | \
  openssl x509 -noout -text | grep -E "Subject:|Validity"

# ดู proxy metrics
kubectl exec -n production deploy/web-app -c linkerd-proxy -- \
  curl -s http://localhost:4191/metrics | grep -E "request_total|response_total"
```

---

## 10. Consul Connect {#consul-connect}

### 10.1 Consul Connect Overview

```
Consul Connect Architecture:

┌──────────────────────────────────────────────────────────────┐
│                    Consul Server Cluster                      │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Consul Servers (3-5 nodes)                          │    │
│  │  - Service Registry                                  │    │
│  │  - Health Check                                      │    │
│  │  - KV Store                                          │    │
│  │  - Certificate Authority (Vault integration)         │    │
│  │  - Intentions (Authorization)                        │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
                           │
              Consul Client (DaemonSet)
                           │
┌──────────────────────────────────────────────────────────────┐
│  Kubernetes Node                                              │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Pod                                                  │   │
│  │  ┌────────────────┐  ┌───────────────────────────┐  │   │
│  │  │  App Container │  │  Consul Connect Proxy      │  │   │
│  │  │    :8080       │  │  (Envoy)                   │  │   │
│  │  │                │  │  - :20000 (Inbound)        │  │   │
│  │  │                │  │  - Dynamic (Outbound)      │  │   │
│  │  └────────────────┘  └───────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 10.2 ติดตั้ง Consul บน Kubernetes

```bash
# เพิ่ม HashiCorp Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# ติดตั้ง Consul
helm install consul hashicorp/consul \
  --namespace consul \
  --create-namespace \
  --values consul-values.yaml
```

```yaml
# consul-values.yaml
global:
  name: consul
  datacenter: dc1
  
  # TLS สำหรับ all communication
  tls:
    enabled: true
    verify: true
    httpsOnly: true
  
  # Consul Connect (Service Mesh)
  acls:
    manageSystemACLs: true
  
  # Metrics
  metrics:
    enabled: true
    
  # Image version
  image: hashicorp/consul:1.17.0

server:
  replicas: 3
  bootstrapExpect: 3
  storage: 10Gi
  
  # Resources
  resources:
    requests:
      memory: "100Mi"
      cpu: "100m"
    limits:
      memory: "100Mi"
      cpu: "2000m"

client:
  enabled: true
  grpc: true

connectInject:
  enabled: true
  default: false    # Opt-in per namespace/service
  
  # Resources สำหรับ sidecar
  sidecarProxy:
    resources:
      requests:
        memory: "50Mi"
        cpu: "50m"
      limits:
        memory: "100Mi"
        cpu: "500m"

ui:
  enabled: true
  service:
    type: LoadBalancer
```

### 10.3 Consul Service Registration

```yaml
# consul-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
      annotations:
        # Enable Consul sidecar injection
        consul.hashicorp.com/connect-inject: "true"
        
        # Service name ใน Consul
        consul.hashicorp.com/connect-service: "api-service"
        
        # Port mapping
        consul.hashicorp.com/connect-service-port: "8080"
        
        # Upstream services ที่ต้องการ access
        consul.hashicorp.com/connect-service-upstreams: "database:5432,cache:6379"
        
        # Resources
        consul.hashicorp.com/sidecar-proxy-cpu-request: "10m"
        consul.hashicorp.com/sidecar-proxy-memory-request: "25Mi"
    spec:
      containers:
      - name: api
        image: my-api:1.0
        ports:
        - containerPort: 8080
        env:
        # ใช้ localhost สำหรับ upstream (ผ่าน sidecar)
        - name: DATABASE_URL
          value: "postgres://localhost:5432/mydb"
        - name: CACHE_URL
          value: "redis://localhost:6379"

---
# Consul Intention - Access Control
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceIntentions
metadata:
  name: api-to-database
  namespace: production
spec:
  destination:
    name: database
  sources:
  - name: api-service
    action: allow
  - name: "*"             # Block everything else
    action: deny

---
# Service Default - Protocol configuration
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: api-service
  namespace: production
spec:
  protocol: http           # http, http2, grpc, tcp

---
# Service Router - Traffic routing
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceRouter
metadata:
  name: api-service
  namespace: production
spec:
  routes:
  - match:
      http:
        pathPrefix: /v2/
    destination:
      service: api-service-v2
      prefixRewrite: /
  - match:
      http:
        header:
        - name: x-canary
          exact: "true"
    destination:
      service: api-service-canary

---
# Service Splitter - Canary
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceSplitter
metadata:
  name: api-service
  namespace: production
spec:
  splits:
  - weight: 90
    service: api-service-stable
  - weight: 10
    service: api-service-canary
```

### 10.4 Consul Commands

```bash
# ดู services ที่ registered
kubectl exec -n consul consul-server-0 -- consul catalog services

# ดู health ของ services
kubectl exec -n consul consul-server-0 -- consul health service api-service

# ดู intentions
kubectl get serviceintentions -n production

# ดู service defaults
kubectl get servicedefaults -n production

# Consul UI
kubectl port-forward -n consul service/consul-ui 8080:80
# เข้า http://localhost:8080

# ดู Connect certificate
kubectl exec -n production deploy/api-service -c consul-connect-inject-data-plane -- \
  curl -s http://localhost:20000/certs | python3 -m json.tool

# Debug intentions
kubectl exec -n consul consul-server-0 -- \
  consul intention check api-service database
```

---

## 11. Workshop: Compare Istio vs Linkerd {#workshop-compare}

### Workshop Overview

เปรียบเทียบ Istio และ Linkerd ในสถานการณ์จริง: mTLS, Traffic Management, Observability

### Step 1: Deploy Sample Application (ใช้กับทั้ง 2 mesh)

```bash
# สร้าง namespace สำหรับทดสอบ
kubectl create namespace mesh-demo

# สร้าง application
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
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
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
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
        - -text
        - '{"service":"backend","version":"1.0"}'
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
  - port: 5678
EOF

kubectl wait --for=condition=Ready pods --all -n mesh-demo --timeout=60s
```

### Step 2: Test ก่อน Mesh (baseline)

```bash
# ทดสอบ plain communication
kubectl exec -n mesh-demo deploy/frontend -- \
  wget -qO- http://backend-svc.mesh-demo.svc.cluster.local:5678

# ดู traffic (ไม่มี encryption)
kubectl exec -n mesh-demo deploy/frontend -- \
  wget -S http://backend-svc.mesh-demo.svc.cluster.local:5678 2>&1 | head -20
```

### Step 3A: ทดสอบกับ Linkerd

```bash
# Inject Linkerd (ถ้ามี Linkerd installed)
kubectl annotate namespace mesh-demo linkerd.io/inject=enabled

# Restart pods เพื่อ inject sidecar
kubectl rollout restart deployment -n mesh-demo

# รอ pods ready
kubectl wait --for=condition=Ready pods --all -n mesh-demo --timeout=60s

# ดู pods (ควรมี 2 containers: app + linkerd-proxy)
kubectl get pods -n mesh-demo

# ทดสอบ traffic
kubectl exec -n mesh-demo deploy/frontend -c frontend -- \
  wget -qO- http://backend-svc.mesh-demo.svc.cluster.local:5678

# ดู metrics
linkerd viz stat deployments -n mesh-demo

# ดู mTLS
linkerd viz edges deployments -n mesh-demo
# SECURED shows mTLS

# Tap traffic
linkerd viz tap deploy/frontend -n mesh-demo --to svc/backend-svc
```

### Step 3B: ทดสอบกับ Istio

```bash
# Enable Istio injection
kubectl label namespace mesh-demo istio-injection=enabled

# Restart pods
kubectl rollout restart deployment -n mesh-demo

# รอ pods ready
kubectl wait --for=condition=Ready pods --all -n mesh-demo --timeout=60s

# ดู pods (ควรมี 2 containers: app + istio-proxy)
kubectl get pods -n mesh-demo

# ทดสอบ traffic
kubectl exec -n mesh-demo deploy/frontend -c frontend -- \
  wget -qO- http://backend-svc.mesh-demo.svc.cluster.local:5678

# ดู status
istioctl proxy-status -n mesh-demo

# ดู mTLS
istioctl x authz check my-pod.mesh-demo

# ดู traffic details
kubectl exec -n mesh-demo deploy/frontend -c istio-proxy -- \
  curl -s http://localhost:15000/stats | grep -E "upstream_rq|cx_active"
```

### Step 4: Canary Deployment

```bash
# สร้าง v2 ของ backend
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v2
  template:
    metadata:
      labels:
        app: backend
        version: v2
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args:
        - -text
        - '{"service":"backend","version":"2.0","new":"feature"}'
        ports:
        - containerPort: 5678
EOF

# สำหรับ Linkerd - ใช้ TrafficSplit
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: backend-split
spec:
  service: backend-svc
  backends:
  - service: backend-svc-v1
    weight: 80
  - service: backend-svc-v2
    weight: 20
EOF

# ทดสอบ canary
for i in $(seq 1 10); do
  kubectl exec -n mesh-demo deploy/frontend -c frontend -- \
    wget -qO- http://backend-svc.mesh-demo.svc.cluster.local:5678
  echo
done
# ควรเห็น v2 ~20% ของเวลา
```

### Step 5: เปรียบเทียบ Resource Usage

```bash
# ดู resource usage
kubectl top pods -n mesh-demo
kubectl top pods -n linkerd
kubectl top pods -n istio-system

# ดู memory ของ sidecars
kubectl get pods -n mesh-demo -o json | \
  python3 -c "
import sys, json
pods = json.load(sys.stdin)
for pod in pods['items']:
    for c in pod['spec']['containers']:
        if 'proxy' in c['name'] or 'istio' in c['name']:
            print(f'{pod[\"metadata\"][\"name\"]}: {c[\"name\"]} - {c.get(\"resources\",{}).get(\"limits\",{})}')
"

# Cleanup
kubectl delete namespace mesh-demo
```

---

## 12. แบบฝึกหัด Service Mesh {#exercises}

### แบบฝึกหัดที่ 1: Linkerd mTLS Verification

```bash
# ติดตั้ง Linkerd (ถ้ายังไม่มี)
# ดูขั้นตอนใน Section 9.2

kubectl create namespace mtls-test
kubectl annotate namespace mtls-test linkerd.io/inject=enabled

# สร้าง services
kubectl apply -n mtls-test -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
      - name: client
        image: nicolaka/netshoot
        command: ["sleep", "3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
    spec:
      containers:
      - name: server
        image: hashicorp/http-echo
        args: ["-text", "hello from server"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: server-svc
spec:
  selector:
    app: server
  ports:
  - port: 5678
EOF

kubectl wait --for=condition=Ready pods --all -n mtls-test --timeout=60s

# ดู pods (ควรมี 2 containers)
kubectl get pods -n mtls-test

# ทดสอบ traffic
kubectl exec -n mtls-test deploy/client -c client -- \
  curl -s http://server-svc.mtls-test.svc.cluster.local:5678

# ตรวจสอบ mTLS
linkerd viz edges deployments -n mtls-test
# SECURED ← ทุก connection ควรเป็น SECURED

# ดู tap
linkerd viz tap deploy/client -n mtls-test --to svc/server-svc &
kubectl exec -n mtls-test deploy/client -c client -- \
  sh -c 'for i in 1 2 3; do curl -s http://server-svc.mtls-test.svc.cluster.local:5678; done'

# Cleanup
kubectl delete namespace mtls-test
```

---

### แบบฝึกหัดที่ 2: Canary Deployment ด้วย Service Mesh

```bash
kubectl create namespace canary-test
kubectl annotate namespace canary-test linkerd.io/inject=enabled

# สร้าง v1
kubectl apply -n canary-test -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text", '{"version":"1.0","msg":"stable"}']
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-svc
spec:
  selector:
    app: myapp
  ports:
  - port: 5678
EOF

# สร้าง v2 (canary)
kubectl apply -n canary-test -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text", '{"version":"2.0","msg":"canary"}']
        ports:
        - containerPort: 5678
EOF

kubectl wait --for=condition=Ready pods --all -n canary-test --timeout=60s

# ทดสอบก่อน split (50/50 เพราะ 3 pods v1, 1 pod v2)
echo "Before TrafficSplit:"
for i in $(seq 1 8); do
  kubectl run test-$RANDOM --rm -n canary-test --image=curlimages/curl \
    --restart=Never --quiet -- \
    curl -s http://app-svc.canary-test.svc.cluster.local:5678 2>/dev/null &
done
wait
echo

# สร้าง SMI TrafficSplit
kubectl apply -n canary-test -f - <<'EOF'
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: app-split
spec:
  service: app-svc
  backends:
  - service: app-svc-v1
    weight: 90
  - service: app-svc-v2
    weight: 10
EOF

# ดู split
linkerd viz stat trafficsplit -n canary-test 2>/dev/null || \
  echo "SMI TrafficSplit requires Linkerd SMI extension"

# Cleanup
kubectl delete namespace canary-test
```

---

## 13. เฉลยแบบฝึกหัด {#answers}

### เฉลยแบบฝึกหัดที่ 1

```
ผลที่คาดหวัง:

Pods (แต่ละ pod มี 2 containers หลัง injection):
NAME                      READY   STATUS
client-xxxx               2/2     Running   ← 2/2 = app + linkerd-proxy
server-xxxx               2/2     Running
server-yyyy               2/2     Running

linkerd viz edges output:
SRC          DST         SECURED
client-xxxx  server-xxxx ✓        ← mTLS enabled!
client-xxxx  server-yyyy ✓

ถ้า edges แสดง UNSECURED:
1. ตรวจสอบว่า namespace มี annotation: linkerd.io/inject=enabled
2. ตรวจสอบว่า pods ถูก restart หลัง inject
3. รัน: linkerd check --proxy

ขนาด Linkerd proxy:
Memory: ~20-50 MB per proxy (เล็กกว่า Istio มาก)
```

### เฉลยแบบฝึกหัดที่ 2

```
ผลที่คาดหวัง:

ก่อน TrafficSplit (4 pods: 3xv1, 1xv2):
Traffic distribution ≈ 75% v1, 25% v2 (สัดส่วนตาม pods)

หลัง TrafficSplit (90/10):
Traffic distribution ≈ 90% v1, 10% v2

linkerd viz stat deployments output:
NAME     MESHED   SUCCESS   RPS   LATENCY_P50   LATENCY_P99
app-v1   3/3       100.00%  8.9   1ms           2ms
app-v2   1/1       100.00%  0.9   1ms           2ms

การ monitor canary:
- ถ้า success rate ของ v2 ต่ำ → rollback ทันที (weight: 0%)
- ถ้า v2 ดี → เพิ่ม weight ทีละน้อย (10% → 25% → 50% → 100%)

Progressive delivery:
Week 1: 10% → v2
Week 2: 25% → v2
Week 3: 50% → v2
Week 4: 100% → v2, delete v1
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/2/overview/)
- [Consul Connect](https://developer.hashicorp.com/consul/docs/connect)
- [Service Mesh Interface (SMI)](https://smi-spec.io/)
- [CNCF Service Mesh Landscape](https://landscape.cncf.io/?category=service-mesh&grouping=category)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/)
- [xDS API Specification](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

---

*ก่อนหน้า: [Part 38 - DNS](./part-38-dns.md)*
*ต่อไป: [Part 40 - Istio](./part-40-istio.md)*
