# Part 40: Istio

## สารบัญ
1. [Istio Architecture](#istio-architecture)
2. [ติดตั้ง Istio](#install-istio)
3. [Virtual Services](#virtual-services)
4. [Destination Rules](#destination-rules)
5. [Traffic Management](#traffic-management)
6. [mTLS ใน Istio](#mtls)
7. [Workshop: Deploy Microservices ด้วย Istio](#workshop)

---

## 1. Istio Architecture {#istio-architecture}

### ภาพรวม Istio

```
┌───────────────────────────────────────────────────────────────┐
│                        Istio                                   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Control Plane (istiod)               │   │
│  │                                                       │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │   │
│  │  │   Pilot    │  │  Citadel   │  │   Galley   │     │   │
│  │  │ (Traffic   │  │ (Certificate│  │ (Config    │     │   │
│  │  │  Mgmt)     │  │  Auth)     │  │  Validation│     │   │
│  │  └────────────┘  └────────────┘  └────────────┘     │   │
│  │                                                       │   │
│  │  All merged into single istiod binary (v1.5+)        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Data Plane                         │   │
│  │                                                       │   │
│  │  ┌──────────────────┐    ┌──────────────────┐       │   │
│  │  │  Pod A           │    │  Pod B           │       │   │
│  │  │  ┌────────────┐  │    │  ┌────────────┐  │       │   │
│  │  │  │App (8080)  │  │    │  │App (8080)  │  │       │   │
│  │  │  └────────────┘  │    │  └────────────┘  │       │   │
│  │  │  ┌────────────┐  │    │  ┌────────────┐  │       │   │
│  │  │  │Envoy Sidecar│─┼────┼──│Envoy Sidecar│ │       │   │
│  │  │  │(istio-proxy)│ │    │  │(istio-proxy)│ │       │   │
│  │  │  └────────────┘  │    │  └────────────┘  │       │   │
│  │  └──────────────────┘    └──────────────────┘       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  Add-ons:                                                     │
│  ├── Kiali (Service Graph Dashboard)                        │
│  ├── Jaeger (Distributed Tracing)                           │
│  ├── Prometheus (Metrics)                                    │
│  └── Grafana (Dashboards)                                    │
└───────────────────────────────────────────────────────────────┘
```

### istiod Components

```
istiod = Pilot + Citadel + Galley

Pilot:
├── Service discovery
├── Traffic management (routing rules → xDS config)
└── Pushes config to Envoy proxies via xDS API

Citadel:
├── Certificate Authority (CA)
├── Issue and rotate certificates
└── Enable mTLS

Galley:
├── Validate Istio configurations
└── Process and distribute configuration
```

### Envoy Sidecar ใน Istio

```
Pod ด้วย Istio Sidecar:
┌─────────────────────────────────────────────────────────────┐
│                           Pod                                │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              init container (istio-init)            │    │
│  │  ตั้ง iptables rules เพื่อ redirect traffic         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌───────────────────┐    ┌───────────────────────────┐   │
│  │  Application      │    │  istio-proxy (Envoy)      │   │
│  │  Container        │    │                           │   │
│  │  Port: 8080       │    │  Inbound: :15006          │   │
│  │                   │    │  Outbound: :15001         │   │
│  │                   │    │  Admin: :15000            │   │
│  │                   │    │  Health: :15021           │   │
│  └───────────────────┘    └───────────────────────────┘   │
│                                                              │
│  iptables rules:                                            │
│  ALL inbound → port 15006 (istio-proxy)                    │
│  ALL outbound → port 15001 (istio-proxy)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. ติดตั้ง Istio {#install-istio}

### Prerequisites

```bash
# ต้องการ:
# - Kubernetes 1.22+
# - kubectl
# - 2+ CPU per Node
# - 4GB RAM per Node
# - เปิด privileged containers

# ตรวจสอบ cluster
kubectl get nodes
kubectl version
```

### ติดตั้ง istioctl

```bash
# ดาวน์โหลด istioctl
curl -L https://istio.io/downloadIstio | sh -
# หรือ version เฉพาะ:
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.19.0 sh -

# เพิ่ม istioctl ใน PATH
cd istio-1.19.0
export PATH=$PWD/bin:$PATH

# ตรวจสอบ
istioctl version
```

### ติดตั้ง Istio บน Cluster

```bash
# Profile options:
# default: Production use
# demo: Development/testing (with more features, more resources)
# minimal: Minimal components
# remote: Remote cluster in multi-cluster setup
# empty: No components (custom install)
# preview: Latest preview features

# Development/Testing
istioctl install --set profile=demo -y

# Production (resource optimized)
istioctl install --set profile=default -y

# ตรวจสอบ
kubectl get pods -n istio-system
kubectl get services -n istio-system
```

```
After installation:
$ kubectl get pods -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-egressgateway-xxx               1/1     Running   0          2m
istio-ingressgateway-xxx              1/1     Running   0          2m
istiod-xxx                            1/1     Running   0          2m
```

### IstioOperator Configuration

```yaml
# istio-config.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
  
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 2048Mi
        hpaSpec:
          maxReplicas: 5
          minReplicas: 1
    
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        service:
          type: LoadBalancer
          ports:
          - port: 80
            name: http2
            targetPort: 8080
          - port: 443
            name: https
            targetPort: 8443
    
    egressGateways:
    - name: istio-egressgateway
      enabled: true
  
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
          limits:
            cpu: 200m
            memory: 256Mi
      tracer:
        zipkin:
          address: zipkin.istio-system:9411
    
    pilot:
      autoscaleEnabled: true
      autoscaleMax: 5
      traceSampling: 1.0  # 100% sampling for dev
```

```bash
# ติดตั้งด้วย custom config
istioctl install -f istio-config.yaml -y

# Verify installation
istioctl verify-install
```

### Enable Sidecar Injection

```bash
# Inject สำหรับ namespace
kubectl label namespace default istio-injection=enabled

# ตรวจสอบ
kubectl get namespace default --show-labels

# Inject เฉพาะ Pod (ถ้าไม่ต้องการ inject ทุก namespace)
# เพิ่ม annotation ใน pod spec:
# sidecar.istio.io/inject: "true"
```

### ติดตั้ง Addons

```bash
# ติดตั้ง addons (Kiali, Jaeger, Prometheus, Grafana)
kubectl apply -f samples/addons/

# ตรวจสอบ
kubectl get pods -n istio-system

# Access dashboards
istioctl dashboard kiali       # Service mesh visualization
istioctl dashboard jaeger      # Distributed tracing
istioctl dashboard grafana     # Metrics dashboards
istioctl dashboard prometheus  # Metrics
```

---

## 3. Virtual Services {#virtual-services}

### Virtual Service คืออะไร?

Virtual Service กำหนด routing rules สำหรับ traffic ที่ไปยัง Service

```
VirtualService:
Request ──► VirtualService rules ──► Destination

เช่น:
- Host-based routing
- Path-based routing
- Header-based routing
- Weight-based routing (canary/A/B)
- Retry policy
- Timeout policy
- Fault injection
```

### Basic Virtual Service

```yaml
# basic-virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
  namespace: default
spec:
  hosts:
  - reviews  # Service name หรือ FQDN
  http:
  - match:
    - uri:
        prefix: "/reviews"
    route:
    - destination:
        host: reviews  # K8s Service name
        port:
          number: 9080
```

### Weight-based Routing (Canary)

```yaml
# canary-virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-canary
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1    # ต้องมี DestinationRule ที่กำหนด subset
      weight: 80      # 80% ไป v1
    - destination:
        host: reviews
        subset: v2
      weight: 20      # 20% ไป v2 (canary)
```

### Header-based Routing

```yaml
# header-routing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: header-based-routing
spec:
  hosts:
  - my-service
  http:
  # Routing ตาม header
  - match:
    - headers:
        x-user-type:
          exact: premium     # Premium users ไป v2
    route:
    - destination:
        host: my-service
        subset: v2
  - match:
    - headers:
        x-beta-user:
          exact: "true"      # Beta users ไป v3
    route:
    - destination:
        host: my-service
        subset: v3
  # Default route
  - route:
    - destination:
        host: my-service
        subset: v1
```

### Retry Configuration

```yaml
# retry-virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: retry-policy
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service
        port:
          number: 80
    retries:
      attempts: 3             # ลอง 3 ครั้ง
      perTryTimeout: 2s       # Timeout แต่ละครั้ง
      retryOn: "gateway-error,connect-failure,retriable-4xx"
      # Options: 5xx, gateway-error, reset, connect-failure,
      #          retriable-4xx, envoy-ratelimited
    timeout: 10s              # Overall timeout
```

### Fault Injection

```yaml
# fault-injection.yaml
# ใช้สำหรับ Chaos Engineering / Testing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fault-injection-test
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: testuser    # เฉพาะ user นี้จะได้รับ fault
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s       # Delay 7 วินาที
      abort:
        percentage:
          value: 10.0
        httpStatus: 503      # 10% จะได้ 503 error
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

### Advanced Routing

```yaml
# advanced-routing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: complex-routing
spec:
  hosts:
  - my-app
  http:
  # Match ด้วย URI, Method, Headers
  - match:
    - uri:
        prefix: "/api/v2"
      method:
        exact: GET
      headers:
        authorization:
          regex: "Bearer .*"
    route:
    - destination:
        host: api-v2-service
        port:
          number: 8080
    corsPolicy:
      allowOrigins:
      - exact: "https://app.example.com"
      allowMethods:
      - GET
      - POST
      - PUT
      allowHeaders:
      - authorization
      - content-type
      maxAge: "24h"
  
  # Traffic mirroring
  - match:
    - uri:
        prefix: "/api"
    route:
    - destination:
        host: api-v1-service
        port:
          number: 8080
    mirror:
      host: api-v2-service  # Mirror 100% traffic to v2 (for testing)
      port:
        number: 8080
    mirrorPercentage:
      value: 100.0
```

---

## 4. Destination Rules {#destination-rules}

### Destination Rule คืออะไร?

DestinationRule กำหนด policies สำหรับ traffic หลังจาก routing (Load balancing, connection pool, circuit breaker, TLS)

```
Flow:
Request ──► VirtualService (routing) ──► DestinationRule (policies) ──► Pod
```

### Subsets (Pod versions)

```yaml
# destination-rule-subsets.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-dr
  namespace: default
spec:
  host: reviews  # K8s Service name
  
  trafficPolicy:
    # Default policy สำหรับทุก subset
    loadBalancer:
      simple: RANDOM
  
  subsets:
  - name: v1          # ชื่อ subset
    labels:
      version: v1     # Pod label selector
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN  # Override default
  
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        consistentHash:
          httpHeaderName: x-user-id  # Sticky session
  
  - name: v3
    labels:
      version: v3
      canary: "true"
```

### Load Balancing Algorithms

```yaml
# load-balancing-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: lb-policy
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      # Simple algorithms:
      # ROUND_ROBIN (default)
      # LEAST_CONN - Least connections
      # RANDOM
      # PASSTHROUGH - Direct pass to destination
      simple: LEAST_CONN
      
      # OR: Consistent Hash (sticky sessions)
      # consistentHash:
      #   httpHeaderName: x-user-id
      #   OR: httpCookie:
      #         name: "user"
      #         ttl: 3600s
      #   OR: useSourceIp: true
```

### Circuit Breaker

```yaml
# circuit-breaker-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: circuit-breaker-policy
spec:
  host: my-backend-service
  trafficPolicy:
    outlierDetection:
      # Circuit Breaker / Outlier Detection
      consecutiveGatewayErrors: 5    # ผิดพลาด 5 ครั้งติดกัน
      consecutive5xxErrors: 5         # หรือ 5xx errors 5 ครั้ง
      interval: 30s                   # ตรวจสอบทุก 30 วิ
      baseEjectionTime: 30s           # ขับออก 30 วิ
      maxEjectionPercent: 50          # ขับออกสูงสุด 50%
      minHealthPercent: 50            # ต้องมี healthy endpoints อย่างน้อย 50%
    connectionPool:
      tcp:
        maxConnections: 100           # Max TCP connections
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 1000 # Max queued requests
        http2MaxRequests: 1000        # Max concurrent requests
        maxRequestsPerConnection: 10  # Max requests per connection
        maxRetries: 3                 # Max retries
        idleTimeout: 90s
        h2UpgradePolicy: UPGRADE      # Upgrade to HTTP/2
```

### TLS Configuration

```yaml
# tls-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: tls-policy
spec:
  host: my-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # Enable mTLS (default when PeerAuthentication set)
      # Modes:
      # DISABLE: No TLS
      # SIMPLE: TLS to destination
      # MUTUAL: Client certificate required
      # ISTIO_MUTUAL: Istio manages certificates
```

---

## 5. Traffic Management {#traffic-management}

### Ingress Gateway

```yaml
# istio-gateway.yaml
# Istio Gateway (replace Kubernetes Ingress)
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: app-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway  # ใช้ default ingress gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.example.com"
    tls:
      httpsRedirect: true  # HTTP → HTTPS redirect
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*.example.com"
    tls:
      mode: SIMPLE
      credentialName: app-tls  # K8s Secret
---
# VirtualService for Gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-vs
  namespace: default
spec:
  hosts:
  - "app.example.com"
  gateways:
  - istio-system/app-gateway  # ชี้ไป Gateway
  http:
  - match:
    - uri:
        prefix: "/api"
    route:
    - destination:
        host: api-service
        port:
          number: 8080
  - route:
    - destination:
        host: frontend-service
        port:
          number: 3000
```

### Egress Gateway

```yaml
# egress-gateway.yaml
# Control outbound traffic ผ่าน Egress Gateway
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - api.external-service.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-external-api
spec:
  hosts:
  - api.external-service.com
  tls:
  - match:
    - port: 443
      sniHosts:
      - api.external-service.com
    route:
    - destination:
        host: api.external-service.com
        port:
          number: 443
      weight: 100
```

### Service Entry (External Services)

```yaml
# service-entry.yaml
# Register external service ใน mesh
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-database
spec:
  hosts:
  - db.external.com
  ports:
  - number: 5432
    name: postgres
    protocol: TCP
  location: MESH_EXTERNAL
  resolution: DNS
---
# ServiceEntry สำหรับ IP range
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-ip-range
spec:
  hosts:
  - "*.legacy-system.com"
  ports:
  - number: 80
    name: http
    protocol: HTTP
  location: MESH_EXTERNAL
  resolution: NONE
  endpoints:
  - address: 10.1.2.3
    labels:
      version: legacy
```

---

## 6. mTLS ใน Istio {#mtls}

### PeerAuthentication

```yaml
# peer-authentication.yaml
# Enable mTLS ทั้ง namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS
    # PERMISSIVE: Accept both mTLS and plaintext (migration mode)
    # STRICT: Only mTLS
    # DISABLE: No mTLS
---
# Enable mTLS สำหรับ specific service
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: payment-service-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: STRICT
    9090:
      mode: PERMISSIVE  # Admin port permissive
```

### AuthorizationPolicy

```yaml
# authorization-policy.yaml
# กำหนดสิทธิ์การเข้าถึง Service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  # Allow จาก order-service เท่านั้น
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/order-service"
    to:
    - operation:
        methods: ["POST", "PUT"]
        paths: ["/api/payment*"]
  # Allow prometheus สำหรับ metrics
  - from:
    - source:
        namespaces: ["istio-system"]
    to:
    - operation:
        paths: ["/metrics"]
        methods: ["GET"]
---
# Deny all by default (zero trust)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  selector:
    matchLabels: {}
  # ไม่มี rules = deny all
```

### RequestAuthentication (JWT)

```yaml
# request-authentication.yaml
# JWT validation
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
    audiences:
    - "api.example.com"
    forwardOriginalToken: true
---
# AuthorizationPolicy ที่ใช้ JWT claims
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: jwt-based-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  rules:
  - when:
    - key: request.auth.claims[role]
      values: ["admin", "editor"]
    to:
    - operation:
        methods: ["POST", "PUT", "DELETE"]
  - when:
    - key: request.auth.claims[role]
      values: ["viewer", "admin", "editor"]
    to:
    - operation:
        methods: ["GET"]
```

---

## 7. Workshop: Deploy Microservices ด้วย Istio {#workshop}

### Workshop 1: ติดตั้ง Istio สำหรับ Workshop

```bash
# สร้าง kind cluster
cat > kind-istio.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: istio-demo
nodes:
  - role: control-plane
    extraPortMappings:
    - containerPort: 30080
      hostPort: 30080
  - role: worker
  - role: worker
networking:
  podSubnet: "10.244.0.0/16"
EOF

kind create cluster --config kind-istio.yaml

# ติดตั้ง Istio
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.19.0 sh -
cd istio-1.19.0
export PATH=$PWD/bin:$PATH

istioctl install --set profile=demo -y

# ติดตั้ง Addons
kubectl apply -f samples/addons/

# รอ Addons พร้อม
kubectl wait --for=condition=ready pod -l app=kiali -n istio-system --timeout=120s
kubectl wait --for=condition=ready pod -l app=prometheus -n istio-system --timeout=120s
kubectl wait --for=condition=ready pod -l app=grafana -n istio-system --timeout=120s
```

### Workshop 2: Deploy Bookinfo Application

```bash
# สร้าง namespace
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled

# Deploy Bookinfo (Istio sample app)
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo

# รอ Pods พร้อม
kubectl wait --for=condition=ready pod -l app=productpage -n bookinfo --timeout=120s
kubectl wait --for=condition=ready pod -l app=details -n bookinfo --timeout=120s
kubectl wait --for=condition=ready pod -l app=reviews -n bookinfo --timeout=120s
kubectl wait --for=condition=ready pod -l app=ratings -n bookinfo --timeout=120s

# ตรวจสอบ sidecar
kubectl get pods -n bookinfo
# ควรเห็น 2/2 READY
```

```
Bookinfo Architecture:
                         ┌────────────────────────────────┐
                         │           Istio Mesh           │
                         │                                │
User ──► Ingress ──► productpage ──► details             │
                              └──► reviews v1 (no stars) │
                                       reviews v2 (black) │
                                       reviews v3 (red)   │
                              └──► ratings                │
                         └────────────────────────────────┘
```

### Workshop 3: Setup Gateway

```bash
# สร้าง Gateway
kubectl apply -f - << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.local
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo-vs
  namespace: bookinfo
spec:
  hosts:
  - bookinfo.local
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
EOF

# ดู Ingress Gateway external IP
kubectl get service istio-ingressgateway -n istio-system

# เพิ่ม hosts
INGRESS_IP=$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$INGRESS_IP bookinfo.local" >> /etc/hosts

# ทดสอบ
curl http://bookinfo.local/productpage
```

### Workshop 4: Traffic Shifting (Canary)

```yaml
# bookinfo-traffic-shift.yaml
# DestinationRule สำหรับ reviews versions
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-dr
  namespace: bookinfo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
# VirtualService: All traffic ไป v1
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
```

```bash
kubectl apply -f bookinfo-traffic-shift.yaml

# ทดสอบ: ทุก request ไป v1 (no stars)
for i in {1..5}; do
  curl -s http://bookinfo.local/productpage | grep -o "reviews.*stars.*" | head -1
done

# Shift traffic: 50% ไป v1, 50% ไป v3
kubectl apply -f - << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
EOF

# ทดสอบ: mix ของ no stars และ red stars
for i in {1..10}; do
  STARS=$(curl -s http://bookinfo.local/productpage | grep -c "full")
  echo "Request $i: Stars=$STARS"
done
```

### Workshop 5: Fault Injection

```bash
# Inject 7 second delay สำหรับ user "jason"
kubectl apply -f - << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-delay
  namespace: bookinfo
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
EOF

# DestinationRule สำหรับ ratings
kubectl apply -f - << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings-dr
  namespace: bookinfo
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
EOF

# ทดสอบ: จำลอง request จาก user "jason" (ควรช้า 7 วิ)
time curl -s -H "end-user: jason" http://bookinfo.local/productpage | grep -o "Ratings service"

# Request ปกติ (ควรเร็ว)
time curl -s http://bookinfo.local/productpage | grep -o "Ratings service"
```

### Workshop 6: Circuit Breaker

```yaml
# circuit-breaker-workshop.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: httpbin-dr
  namespace: bookinfo
spec:
  host: httpbin
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 3
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
```

### Workshop 7: mTLS Verification

```bash
# ตรวจสอบ mTLS status
istioctl authn tls-check -n bookinfo reviews.bookinfo.svc.cluster.local

# ดู peer authentication
kubectl get peerauthentication -n bookinfo

# Enable STRICT mTLS
kubectl apply -f - << 'EOF'
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF

# ตรวจสอบ
istioctl authn tls-check reviews.bookinfo.svc.cluster.local

# ทดสอบว่า unencrypted connection ถูก reject
kubectl run test-pod --image=nicolaka/netshoot --rm -it -- bash
# curl http://reviews.bookinfo.svc.cluster.local:9080/reviews/1
# ควรถูก reject (ถ้าไม่มี sidecar)
```

### Workshop 8: Observability Dashboard

```bash
# เปิด Kiali dashboard
istioctl dashboard kiali &
echo "Kiali: http://localhost:20001/kiali"

# Generate traffic สำหรับ visualization
for i in {1..100}; do
  curl -s http://bookinfo.local/productpage > /dev/null
  sleep 0.1
done

# เปิด Jaeger สำหรับ distributed tracing
istioctl dashboard jaeger &
echo "Jaeger: http://localhost:16686"

# เปิด Grafana สำหรับ metrics
istioctl dashboard grafana &
echo "Grafana: http://localhost:3000"

# เปิด Prometheus
istioctl dashboard prometheus &
echo "Prometheus: http://localhost:9090"
```

### Workshop 9: Advanced Traffic Management

```yaml
# advanced-bookinfo.yaml
# จำลอง A/B testing based on user segment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-ab-test
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  # Premium users ไป v3 (red stars)
  - match:
    - headers:
        x-user-segment:
          exact: premium
    route:
    - destination:
        host: reviews
        subset: v3
  
  # Beta users ไป v2 (black stars)
  - match:
    - headers:
        x-user-segment:
          exact: beta
    route:
    - destination:
        host: reviews
        subset: v2
  
  # 20% random ไป v2 (gradual rollout)
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
```

```bash
kubectl apply -f advanced-bookinfo.yaml

# ทดสอบ A/B routing
curl -s -H "x-user-segment: premium" http://bookinfo.local/productpage | grep "full"
curl -s -H "x-user-segment: beta" http://bookinfo.local/productpage | grep "full"
curl -s http://bookinfo.local/productpage | grep "full"  # Random 80/20
```

### Workshop 10: Monitoring และ Debugging

```bash
# ตรวจสอบ proxy configuration
istioctl proxy-status

# ดู Envoy config สำหรับ specific Pod
PRODUCTPAGE_POD=$(kubectl get pod -n bookinfo -l app=productpage -o name | head -1)
istioctl proxy-config cluster -n bookinfo $PRODUCTPAGE_POD
istioctl proxy-config listener -n bookinfo $PRODUCTPAGE_POD
istioctl proxy-config route -n bookinfo $PRODUCTPAGE_POD
istioctl proxy-config endpoint -n bookinfo $PRODUCTPAGE_POD

# Analyze ปัญหา
istioctl analyze -n bookinfo

# ดู logs ของ sidecar
kubectl logs -n bookinfo $PRODUCTPAGE_POD -c istio-proxy

# ดู Istio metrics
kubectl exec -n bookinfo $PRODUCTPAGE_POD -c istio-proxy \
  -- pilot-agent request GET stats | head -30

# ตรวจสอบ mTLS
istioctl authn tls-check -n bookinfo
```

### Cleanup

```bash
# ลบ Bookinfo
kubectl delete namespace bookinfo

# Uninstall Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# ลบ cluster
kind delete cluster --name istio-demo
```

---

## Istio CRDs Summary

```
Istio Custom Resources:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Networking:                                            │
│  ├── Gateway          - Ingress/Egress gateway config   │
│  ├── VirtualService   - Traffic routing rules           │
│  ├── DestinationRule  - Traffic policies (LB, CB)       │
│  ├── ServiceEntry     - External service registration   │
│  └── Sidecar         - Per-service proxy config        │
│                                                          │
│  Security:                                              │
│  ├── PeerAuthentication  - mTLS settings               │
│  ├── RequestAuthentication - JWT validation            │
│  └── AuthorizationPolicy  - Access control             │
│                                                          │
│  Telemetry:                                             │
│  └── Telemetry          - Metrics/Tracing config       │
└──────────────────────────────────────────────────────────┘
```

---

## Cheat Sheet

```bash
# Istio installation
istioctl install --set profile=demo
istioctl uninstall --purge

# Namespace injection
kubectl label namespace <ns> istio-injection=enabled
kubectl label namespace <ns> istio-injection-  # Remove label

# Pod injection
kubectl annotate pod <pod> sidecar.istio.io/inject=true

# Verify
istioctl verify-install
istioctl analyze
istioctl proxy-status

# Debug
istioctl proxy-config cluster <pod>
istioctl proxy-config listener <pod>
istioctl proxy-config route <pod>
istioctl proxy-config endpoint <pod>
istioctl authn tls-check

# Dashboards
istioctl dashboard kiali
istioctl dashboard jaeger
istioctl dashboard grafana
istioctl dashboard prometheus
istioctl dashboard envoy <pod>

# Traffic management
kubectl get virtualservice
kubectl get destinationrule
kubectl get gateway
kubectl get serviceentry

# Security
kubectl get peerauthentication
kubectl get authorizationpolicy
kubectl get requestauthentication
```

---

## สรุป Part 31-40: Kubernetes Networking

```
Kubernetes Networking Journey:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Part 31: Networking Model & CNI                          │
│  └── How pods communicate + CNI plugins                  │
│                                                            │
│  Part 32: ClusterIP                                       │
│  └── Internal service discovery                          │
│                                                            │
│  Part 33: NodePort                                        │
│  └── External access via node ports                      │
│                                                            │
│  Part 34: LoadBalancer                                    │
│  └── Cloud/MetalLB load balancers                        │
│                                                            │
│  Part 35: Ingress Controllers                             │
│  └── NGINX, Traefik for L7 routing                       │
│                                                            │
│  Part 36: Ingress Resources                               │
│  └── Host/path routing, TLS, canary                      │
│                                                            │
│  Part 37: Network Policies                                │
│  └── Kubernetes firewall, zero-trust                     │
│                                                            │
│  Part 38: DNS                                             │
│  └── CoreDNS, service discovery                          │
│                                                            │
│  Part 39: Service Mesh Concepts                           │
│  └── Sidecar pattern, Linkerd intro                      │
│                                                            │
│  Part 40: Istio                                           │
│  └── Full service mesh: traffic, security, observe       │
└────────────────────────────────────────────────────────────┘
```

---

*ก่อนหน้า: [Part 39 - Service Mesh](./part-39-service-mesh.md)*
*ต่อไป: Part 41 - Storage (Coming Soon)*
