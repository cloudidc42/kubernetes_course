# Part 35: Ingress Controllers

## สารบัญ
1. [Ingress Controller คืออะไร](#ingress-controller-overview)
2. [NGINX Ingress Controller](#nginx-ingress)
3. [Traefik Ingress Controller](#traefik)
4. [ติดตั้งและ Configure Ingress Controllers](#install-configure)
5. [Workshop: Setup Production Ingress](#workshop)

---

## 1. Ingress Controller คืออะไร {#ingress-controller-overview}

### ปัญหาที่ Ingress แก้ไข

```
ปัญหาก่อนมี Ingress:
┌──────────────────────────────────────────────────────┐
│                                                      │
│  service1: LoadBalancer ──► External IP 1 ($$$)    │
│  service2: LoadBalancer ──► External IP 2 ($$$)    │
│  service3: LoadBalancer ──► External IP 3 ($$$)    │
│  service4: NodePort ──► Node:30080                 │
│  service5: NodePort ──► Node:30081                 │
│                                                      │
│  ปัญหา:                                            │
│  - หลาย External IPs (แพง)                         │
│  - ไม่มี HTTP routing                              │
│  - ไม่มี SSL/TLS termination รวมศูนย์              │
│  - ยากในการจัดการ                                  │
└──────────────────────────────────────────────────────┘

แก้ด้วย Ingress:
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Single External IP ──► Ingress Controller          │
│                              │                      │
│                    ┌─────────┴────────┐             │
│                    │  L7 HTTP Router  │             │
│                    └────────┬─────────┘             │
│                             │                       │
│         ┌───────────────────┼───────────────┐      │
│         │                   │               │      │
│  /api ──► service1    /web ──► service2    /──► service3 │
│                                                      │
│  ✅ Single IP/Domain                               │
│  ✅ SSL/TLS termination                            │
│  ✅ Host-based routing                             │
│  ✅ Path-based routing                             │
│  ✅ Rate limiting, Auth, etc.                      │
└──────────────────────────────────────────────────────┘
```

### Ingress Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  Internet                                                    │
│     │                                                        │
│     ▼                                                        │
│  LoadBalancer Service                                        │
│  (External IP: 1.2.3.4)                                     │
│     │                                                        │
│     ▼                                                        │
│  ┌─────────────────────────────────────┐                   │
│  │        Ingress Controller           │                   │
│  │    (nginx/traefik/haproxy pods)     │                   │
│  │                                     │                   │
│  │  Reads Ingress Resources:           │                   │
│  │  - Routing rules                    │                   │
│  │  - SSL certificates                 │                   │
│  │  - Backend services                 │                   │
│  └──────────────┬──────────────────────┘                   │
│                 │                                            │
│  ┌──────────────┼──────────────────┐                       │
│  │              │                  │                        │
│  ▼              ▼                  ▼                        │
│ Service A    Service B          Service C                   │
│ (api-svc)  (frontend-svc)     (admin-svc)                  │
│    │             │                 │                        │
│  Pods          Pods             Pods                        │
└─────────────────────────────────────────────────────────────┘
```

### Ingress Components

```
┌───────────────────────────────────────────────────┐
│                                                   │
│  Ingress Class                                    │
│  └── ระบุว่าจะใช้ Controller อะไร               │
│                                                   │
│  IngressClass Resource                            │
│  └── nginx, traefik, alb, etc.                  │
│                                                   │
│  Ingress Resource                                 │
│  └── Routing rules (Host, Path → Service)        │
│                                                   │
│  Ingress Controller                               │
│  └── Implementation (Nginx, Traefik, etc.)       │
│       Watches Ingress resources                   │
│       Configures Reverse Proxy                   │
└───────────────────────────────────────────────────┘
```

### เปรียบเทียบ Ingress Controllers

| Feature | NGINX | Traefik | HAProxy | Contour | Istio |
|---------|-------|---------|---------|---------|-------|
| Popularity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| Performance | Very High | High | Very High | High | Medium |
| Auto SSL | ✅ | ✅ | ✅ | ✅ | ✅ |
| gRPC | ✅ | ✅ | ✅ | ✅ | ✅ |
| mTLS | ✅ | ✅ | ✅ | ✅ | ✅ |
| Dashboard | ❌ | ✅ | ✅ | ❌ | ✅ |
| Config | Annotations | CRDs | Annotations | CRDs | CRDs |
| Complexity | Low | Low | Medium | Medium | High |

---

## 2. NGINX Ingress Controller {#nginx-ingress}

### NGINX Ingress Architecture

```
┌──────────────────────────────────────────────────────────┐
│              NGINX Ingress Controller                     │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Controller Process                              │  │
│  │  ├── Watch Ingress Resources                    │  │
│  │  ├── Watch Services/Endpoints                   │  │
│  │  ├── Generate nginx.conf                        │  │
│  │  └── Reload NGINX                              │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  NGINX Process                                   │  │
│  │  ├── HTTP/HTTPS traffic                         │  │
│  │  ├── SSL termination                            │  │
│  │  ├── Proxy to backends                          │  │
│  │  └── Load balancing                             │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### ติดตั้ง NGINX Ingress Controller

```bash
# วิธีที่ 1: Helm (แนะนำ)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux

# วิธีที่ 2: kubectl apply
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# ตรวจสอบ
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx
```

### NGINX Ingress Controller Configuration

```yaml
# nginx-ingress-config.yaml
# ConfigMap สำหรับ Global NGINX settings
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # Global settings
  use-forwarded-headers: "true"
  compute-full-forwarded-for: "true"
  use-proxy-protocol: "false"
  
  # Timeouts
  proxy-connect-timeout: "10"
  proxy-read-timeout: "120"
  proxy-send-timeout: "120"
  
  # Buffer sizes
  proxy-buffer-size: "16k"
  proxy-buffers-number: "4"
  
  # Security headers
  add-headers: "ingress-nginx/custom-headers"
  
  # SSL settings
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
  ssl-session-cache: "shared:SSL:10m"
  
  # Logging
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] [$proxy_alternative_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id'
  
  # Rate limiting
  limit-req-status-code: "429"
  
  # Gzip
  use-gzip: "true"
  gzip-level: "5"
  gzip-types: "application/json text/plain text/css application/javascript"
---
# Custom Headers ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-headers
  namespace: ingress-nginx
data:
  X-Frame-Options: "SAMEORIGIN"
  X-Content-Type-Options: "nosniff"
  X-XSS-Protection: "1; mode=block"
  Strict-Transport-Security: "max-age=31536000; includeSubDomains"
```

### NGINX-specific Annotations

```yaml
# nginx-annotated-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-features-demo
  namespace: default
  annotations:
    # เลือก Ingress Class
    kubernetes.io/ingress.class: "nginx"
    
    # SSL Redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # Load balancing
    nginx.ingress.kubernetes.io/load-balance: "round_robin"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
    
    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # Backend Protocol
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"  # หรือ HTTP, GRPC, GRPCS
    
    # Whitelist
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,172.16.0.0/12"
    
    # Session Affinity
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"

spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## 3. Traefik Ingress Controller {#traefik}

### Traefik Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      Traefik                                  │
│                                                              │
│  EntryPoints           Routers              Services         │
│  ┌──────────┐         ┌──────────┐        ┌──────────┐     │
│  │ :80 (web)│──────►  │ Router 1 │──────► │ Service A│──►Pods│
│  └──────────┘         └──────────┘        └──────────┘     │
│  ┌──────────┐         ┌──────────┐        ┌──────────┐     │
│  │:443(tls) │──────►  │ Router 2 │──────► │ Service B│──►Pods│
│  └──────────┘         └──────────┘        └──────────┘     │
│                                                              │
│  Middlewares:                                                │
│  ├── Authentication                                         │
│  ├── Rate Limiting                                          │
│  ├── Headers                                                │
│  └── Circuit Breaker                                        │
│                                                              │
│  Dashboard: :8080/dashboard                                 │
└──────────────────────────────────────────────────────────────┘
```

### ติดตั้ง Traefik

```bash
# Helm installation
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.storageClass=standard \
  --set logs.general.level=INFO \
  --set logs.access.enabled=true \
  --set ports.web.redirectTo=websecure \
  --set providers.kubernetesCRD.enabled=true \
  --set providers.kubernetesIngress.enabled=true

# ดู pods
kubectl get pods -n traefik
kubectl get service -n traefik
```

### Traefik Values YAML (Full Configuration)

```yaml
# traefik-values.yaml
image:
  repository: traefik
  tag: v2.10

deployment:
  replicas: 2
  
ports:
  web:
    port: 80
    redirectTo: websecure
  websecure:
    port: 443
    tls:
      enabled: true
  traefik:
    port: 9000  # Dashboard
  metrics:
    port: 9100

providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: false
    allowExternalNameServices: false
  kubernetesIngress:
    enabled: true
    publishedService:
      enabled: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /data/acme.json
      httpChallenge:
        entryPoint: web

logs:
  general:
    level: INFO
  access:
    enabled: true
    format: json

metrics:
  prometheus:
    enabled: true
    addEntryPointsLabels: true
    addServicesLabels: true

ingressRoute:
  dashboard:
    enabled: true
    entryPoints:
      - traefik
    
persistence:
  enabled: true
  size: 128Mi
  storageClass: standard

securityContext:
  capabilities:
    drop: [ALL]
    add: [NET_BIND_SERVICE]
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 65532
```

### Traefik IngressRoute (CRD)

```yaml
# traefik-ingressroute.yaml
# HTTP Router
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: web-app
  namespace: default
spec:
  entryPoints:
  - web
  - websecure
  routes:
  - match: Host(`app.example.com`) && PathPrefix(`/api`)
    kind: Rule
    priority: 10
    services:
    - name: api-service
      port: 80
      weight: 1
      sticky:
        cookie:
          name: lb
          secure: true
          httpOnly: true
    middlewares:
    - name: rate-limit
    - name: auth-middleware
  - match: Host(`app.example.com`)
    kind: Rule
    priority: 1
    services:
    - name: frontend-service
      port: 80
  tls:
    certResolver: letsencrypt
    domains:
    - main: app.example.com
```

### Traefik Middlewares

```yaml
# traefik-middlewares.yaml
# Rate Limiting
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 100
    burst: 50
---
# Basic Auth
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: auth-middleware
  namespace: default
spec:
  basicAuth:
    secret: basic-auth-secret
---
# Headers
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
  namespace: default
spec:
  headers:
    frameDeny: true
    contentTypeNosniff: true
    browserXssFilter: true
    sslRedirect: true
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    customResponseHeaders:
      X-Custom-Header: "custom-value"
---
# Redirect
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: www-redirect
  namespace: default
spec:
  redirectRegex:
    regex: "^https://www\\.(.+)"
    replacement: "https://${1}"
    permanent: true
---
# Circuit Breaker
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: circuit-breaker
  namespace: default
spec:
  circuitBreaker:
    expression: "NetworkErrorRatio() > 0.5 || ResponseCodeRatio(500, 600, 0, 600) > 0.5"
---
# Compress
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: compress
  namespace: default
spec:
  compress: {}
```

---

## 4. ติดตั้งและ Configure Ingress Controllers {#install-configure}

### IngressClass Resource

```yaml
# ingressclass.yaml
# สร้าง IngressClass สำหรับ NGINX
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # Set as default
spec:
  controller: k8s.io/ingress-nginx
---
# IngressClass สำหรับ Traefik
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik
spec:
  controller: traefik.io/ingress-controller
```

### Multiple Ingress Controllers

```
ใน Cluster เดียวสามารถมีหลาย Ingress Controllers:

┌─────────────────────────────────────────────────────────┐
│                    Cluster                               │
│                                                          │
│  NGINX Ingress (public)        Traefik (internal)       │
│  ├── External LB               ├── Internal LB          │
│  ├── Port 80/443               ├── Port 80/443          │
│  └── internet-facing           └── intranet-facing      │
│                                                          │
│  Ingress Rule:                 Ingress Rule:            │
│  ingressClassName: nginx       ingressClassName: traefik │
└─────────────────────────────────────────────────────────┘
```

### Default Backend

```yaml
# default-backend.yaml
# หน้า 404 เมื่อไม่มี route ที่ match
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-backend
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-backend
  template:
    metadata:
      labels:
        app: default-backend
    spec:
      containers:
      - name: default-backend
        image: registry.k8s.io/defaultbackend-amd64:1.5
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 10m
            memory: 20Mi
          limits:
            cpu: 100m
            memory: 50Mi
---
apiVersion: v1
kind: Service
metadata:
  name: default-backend
  namespace: ingress-nginx
spec:
  selector:
    app: default-backend
  ports:
  - port: 80
    targetPort: 8080
```

---

## 5. Workshop: Setup Production Ingress {#workshop}

### Workshop 1: ติดตั้ง NGINX Ingress บน Local

```bash
# ใช้ kind สำหรับ workshop
cat > kind-ingress.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ingress-demo
nodes:
  - role: control-plane
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
  - role: worker
EOF

kind create cluster --config kind-ingress.yaml

# ติดตั้ง NGINX Ingress สำหรับ kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# รอ Ingress Controller พร้อม
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Workshop 2: Deploy Sample Applications

```yaml
# sample-apps.yaml
# App 1: API Backend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=API Response from $(HOSTNAME)"
        - "-listen=:3000"
        ports:
        - containerPort: 3000
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-backend
  ports:
  - port: 80
    targetPort: 3000
---
# App 2: Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: default
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
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
# App 3: Admin Panel
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: hashicorp/http-echo
        args:
        - "-text=Admin Panel - $(HOSTNAME)"
        - "-listen=:3000"
        ports:
        - containerPort: 3000
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  selector:
    app: admin
  ports:
  - port: 80
    targetPort: 3000
```

```bash
kubectl apply -f sample-apps.yaml

# รอ Deployments พร้อม
kubectl wait --for=condition=available deployment/api-backend --timeout=60s
kubectl wait --for=condition=available deployment/frontend --timeout=60s
kubectl wait --for=condition=available deployment/admin --timeout=60s
```

### Workshop 3: สร้าง Ingress Rules

```yaml
# production-ingress.yaml
# Path-based Routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /admin(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
      - path: /()(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```bash
kubectl apply -f production-ingress.yaml

# เพิ่ม hosts entry (local testing)
echo "127.0.0.1 app.local" >> /etc/hosts

# ทดสอบ
curl http://app.local/
curl http://app.local/api
curl http://app.local/admin

# ดู Ingress
kubectl describe ingress app-ingress
```

### Workshop 4: TLS/HTTPS

```bash
# สร้าง Self-signed Certificate สำหรับ test
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=app.local/O=MyOrg"

# สร้าง Secret
kubectl create secret tls app-tls \
  --key tls.key \
  --cert tls.crt
```

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.local
    secretName: app-tls  # Secret ที่เราสร้าง
  rules:
  - host: app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

```bash
kubectl apply -f tls-ingress.yaml

# ทดสอบ HTTPS (skip cert verification เนื่องจาก self-signed)
curl -k https://app.local/
curl -k https://app.local/api

# HTTP ควร redirect ไป HTTPS
curl -v http://app.local/
```

### Workshop 5: Rate Limiting

```yaml
# rate-limit-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"          # 5 requests per second
    nginx.ingress.kubernetes.io/limit-connections: "10"  # 10 concurrent connections
    nginx.ingress.kubernetes.io/limit-req-status-code: "429"
spec:
  ingressClassName: nginx
  rules:
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

```bash
echo "127.0.0.1 api.local" >> /etc/hosts
kubectl apply -f rate-limit-ingress.yaml

# ทดสอบ rate limiting
for i in {1..20}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://api.local/)
  echo "Request $i: HTTP $STATUS"
done
```

### Workshop 6: Basic Authentication

```bash
# สร้าง htpasswd file
htpasswd -c auth admin  # ใส่ password เมื่อถาม

# หรือสร้าง encoded password
echo "admin:$(openssl passwd -apr1 'password')" > auth

# สร้าง Secret
kubectl create secret generic basic-auth \
  --from-file=auth
```

```yaml
# auth-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - Admin Panel"
spec:
  ingressClassName: nginx
  rules:
  - host: admin.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

```bash
echo "127.0.0.1 admin.local" >> /etc/hosts
kubectl apply -f auth-ingress.yaml

# ทดสอบ (ควรได้ 401)
curl http://admin.local/

# ทดสอบด้วย credentials (ควรได้ 200)
curl -u admin:password http://admin.local/
```

### Workshop 7: Debug Ingress

```bash
# ดู Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller -f

# ดู Ingress resource
kubectl describe ingress app-ingress

# ดู nginx.conf ภายใน Controller
kubectl exec -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller -o name) \
  -- nginx -T | head -100

# ตรวจสอบ Backend Services
kubectl get service
kubectl get endpoints

# ทดสอบ DNS resolution
kubectl run dns-test --image=nicolaka/netshoot --rm -it -- nslookup api-service

# ดู NGINX status
kubectl exec -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller -o name) \
  -- curl localhost/nginx_status
```

### Cleanup

```bash
kubectl delete -f sample-apps.yaml
kubectl delete -f production-ingress.yaml
kubectl delete secret app-tls basic-auth
kind delete cluster --name ingress-demo
```

---

## สรุป

```
Ingress Controller Summary:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  NGINX Ingress:                                         │
│  ├── ง่าย, stable, widely used                        │
│  ├── Annotation-based configuration                    │
│  └── Best for: General purpose                        │
│                                                          │
│  Traefik:                                               │
│  ├── Auto-discovery, CRD-based                        │
│  ├── Built-in dashboard                               │
│  └── Best for: Dynamic environments                   │
│                                                          │
│  Common Features:                                       │
│  ├── SSL/TLS termination                               │
│  ├── Path & Host based routing                        │
│  ├── Load balancing                                    │
│  ├── Authentication                                    │
│  ├── Rate limiting                                     │
│  └── CORS, Headers                                    │
└──────────────────────────────────────────────────────────┘
```

---

*ก่อนหน้า: [Part 34 - LoadBalancer Service](./part-34-loadbalancer.md)*
*ต่อไป: [Part 36 - Ingress Resources](./part-36-ingress-resources.md)*
