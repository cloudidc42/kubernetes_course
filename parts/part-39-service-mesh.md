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
