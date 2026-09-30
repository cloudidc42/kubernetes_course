# Part 36: Ingress Resources

## สารบัญ
1. [Ingress Resource YAML](#ingress-yaml)
2. [Path-based Routing](#path-routing)
3. [Host-based Routing](#host-routing)
4. [TLS/HTTPS Configuration](#tls-https)
5. [Workshop: Multi-service Routing](#workshop)

---

## 1. Ingress Resource YAML {#ingress-yaml}

### Ingress Resource Structure

```yaml
# ingress-anatomy.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  labels:
    app: my-app
    environment: production
  annotations:
    # Ingress Controller เฉพาะ (nginx, traefik, etc.)
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
spec:
  # ระบุว่าจะใช้ Ingress Controller อะไร
  ingressClassName: nginx
  
  # TLS Configuration
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls-secret
  
  # Default Backend (ถ้าไม่มี rule match)
  defaultBackend:
    service:
      name: default-backend-service
      port:
        number: 80
  
  # Routing Rules
  rules:
  - host: example.com  # Host-based routing
    http:
      paths:
      - path: /api     # Path-based routing
        pathType: Prefix  # หรือ Exact, ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
              # หรือใช้ port name:
              # name: http
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### PathType Options

```
PathType ทั้ง 3 แบบ:

Prefix:
  /api ──► matches: /api, /api/, /api/users, /api/v1/users
  ไม่ match: /apitest, /apiv2

Exact:
  /api ──► matches: /api เท่านั้น
  ไม่ match: /api/, /api/users

ImplementationSpecific:
  ขึ้นอยู่กับ Ingress Controller
  (nginx ใช้ regex ได้)
```

```yaml
# pathtype-examples.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pathtype-demo
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      # Exact: /health เท่านั้น
      - path: /health
        pathType: Exact
        backend:
          service:
            name: health-service
            port:
              number: 80
      
      # Prefix: /api และทุก subpath
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      
      # ImplementationSpecific with regex (nginx)
      - path: /v[0-9]+/users
        pathType: ImplementationSpecific
        backend:
          service:
            name: users-service
            port:
              number: 80
```

### Priority และ Order ของ Paths

```
Ingress Controller จะ match paths ตาม order นี้:

1. Exact paths ก่อน
2. Longer prefix paths ก่อน
3. ImplementationSpecific

ตัวอย่าง:
paths:
  - /api/v2 (Prefix)  ──► match /api/v2/users ก่อน
  - /api (Prefix)     ──► match /api/orders
  - / (Prefix)        ──► match อื่นๆ ทั้งหมด
```

---

## 2. Path-based Routing {#path-routing}

### Simple Path-based Routing

```yaml
# path-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - http:  # ไม่ระบุ host = รับทุก host
      paths:
      # API Service
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      
      # Static Files
      - path: /static(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: static-service
            port:
              number: 80
      
      # Websocket
      - path: /ws(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 8080
      
      # Default Frontend
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Microservices Path Routing

```
Application Architecture:
┌─────────────────────────────────────────────────────┐
│                                                      │
│  app.example.com                                    │
│  ├── /auth  ──────────► auth-service:8080          │
│  ├── /users ──────────► user-service:8080          │
│  ├── /products ────────► product-service:8080      │
│  ├── /orders ──────────► order-service:8080        │
│  ├── /notifications ───► notification-service:8080  │
│  ├── /static ──────────► cdn-service:80            │
│  └── / ────────────────► frontend-service:3000     │
└─────────────────────────────────────────────────────┘
```

```yaml
# microservices-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 8080
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 8080
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /notifications
        pathType: Prefix
        backend:
          service:
            name: notification-service
            port:
              number: 8080
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: cdn-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

### Path Rewriting

```yaml
# path-rewrite-ingress.yaml
# ตัวอย่างการ rewrite paths

# Case 1: Strip prefix
# Request: /api/v1/users  →  Backend receives: /v1/users
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: strip-prefix
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
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
---
# Case 2: Add prefix
# Request: /users  →  Backend receives: /api/v1/users
# (ต้องทำด้วย nginx snippet หรือ server-snippet)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: add-prefix
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /api/v1$1
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /(users.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 3. Host-based Routing {#host-routing}

### Multiple Hosts on Single Ingress

```yaml
# host-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  # Host 1: API Domain
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  
  # Host 2: Frontend Domain
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
  
  # Host 3: Admin Domain
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
  
  # Host 4: Docs Domain
  - host: docs.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: docs-service
            port:
              number: 4000
```

### Multi-tenant Architecture

```yaml
# multi-tenant-ingress.yaml
# แต่ละ tenant มี subdomain ของตัวเอง
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tenant-a-ingress
  namespace: tenant-a
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - tenant-a.example.com
    secretName: tenant-a-tls
  rules:
  - host: tenant-a.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant-a-app
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tenant-b-ingress
  namespace: tenant-b
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - tenant-b.example.com
    secretName: tenant-b-tls
  rules:
  - host: tenant-b.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant-b-app
            port:
              number: 80
```

### Wildcard Host

```yaml
# wildcard-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      set $subdomain "";
      if ($host ~* "^([a-z0-9-]+)\.example\.com$") {
        set $subdomain $1;
      }
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - "*.example.com"
    secretName: wildcard-tls
  rules:
  - host: "*.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

---

## 4. TLS/HTTPS Configuration {#tls-https}

### TLS Certificate Management

```bash
# สร้าง Self-signed Certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout app.key \
  -out app.crt \
  -subj "/CN=app.example.com/O=MyOrg/C=TH"

# สร้าง Kubernetes Secret
kubectl create secret tls app-tls \
  --key app.key \
  --cert app.crt \
  --namespace default

# ดู Secret
kubectl get secret app-tls
kubectl describe secret app-tls
```

```yaml
# tls-secret.yaml
# สร้าง Secret แบบ YAML
apiVersion: v1
kind: Secret
metadata:
  name: app-tls
  namespace: default
type: kubernetes.io/tls
data:
  # base64 encoded
  tls.crt: LS0tLS1CRUdJTi... (base64 of cert)
  tls.key: LS0tLS1CRUdJTi... (base64 of key)
```

### Let's Encrypt กับ cert-manager

```bash
# ติดตั้ง cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# ตรวจสอบ
kubectl get pods -n cert-manager
```

```yaml
# cluster-issuer.yaml
# ClusterIssuer สำหรับ Let's Encrypt Production
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    # HTTP-01 challenge
    - http01:
        ingress:
          class: nginx
---
# ClusterIssuer สำหรับ Let's Encrypt Staging (ทดสอบ)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: admin@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```yaml
# auto-tls-ingress.yaml
# Ingress ที่ cert-manager จัดการ TLS ให้อัตโนมัติ
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-example-com-tls  # cert-manager จะสร้าง Secret นี้
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Wildcard Certificate

```yaml
# wildcard-cert.yaml
# ต้องใช้ DNS-01 challenge
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-cert
  namespace: default
spec:
  secretName: wildcard-example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: "*.example.com"
  dnsNames:
  - "*.example.com"
  - "example.com"
```

### TLS Configuration สำหรับ NGINX

```yaml
# nginx-tls-config.yaml
# Custom TLS settings per Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
    nginx.ingress.kubernetes.io/ssl-prefer-server-ciphers: "on"
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: secure-example-tls
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 443
```

---

## 5. Workshop: Multi-service Routing {#workshop}

### Setup Workshop Environment

```bash
# ติดตั้ง kind cluster พร้อม Ingress support
cat > kind-full.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: routing-demo
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
      hostPort: 8080
    - containerPort: 443
      hostPort: 8443
  - role: worker
EOF

kind create cluster --config kind-full.yaml

# ติดตั้ง NGINX Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Workshop 1: Full E-commerce Application

```yaml
# ecommerce-app.yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
---
# Frontend Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
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
        - name: config
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init
        image: busybox
        command:
        - sh
        - -c
        - echo "<h1>Frontend Store</h1>" > /html/index.html
        volumeMounts:
        - name: config
          mountPath: /html
      volumes:
      - name: config
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
# Products API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-api
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: products-api
  template:
    metadata:
      labels:
        app: products-api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=Products API v2"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: products-api
  namespace: ecommerce
spec:
  selector:
    app: products-api
  ports:
  - port: 80
    targetPort: 5678
---
# Orders API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=Orders API v1"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: orders-api
  namespace: ecommerce
spec:
  selector:
    app: orders-api
  ports:
  - port: 80
    targetPort: 5678
---
# Users Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-api
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: users-api
  template:
    metadata:
      labels:
        app: users-api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=Users API - Auth Required"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: users-api
  namespace: ecommerce
spec:
  selector:
    app: users-api
  ports:
  - port: 80
    targetPort: 5678
```

```bash
kubectl apply -f ecommerce-app.yaml

# รอ Deployments พร้อม
kubectl wait --for=condition=available deployment/frontend -n ecommerce --timeout=60s
kubectl wait --for=condition=available deployment/products-api -n ecommerce --timeout=60s
kubectl wait --for=condition=available deployment/orders-api -n ecommerce --timeout=60s
```

### Workshop 2: Path-based Routing สำหรับ E-commerce

```yaml
# ecommerce-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-App-Version "2.0";
      add_header X-Request-ID $request_id;
spec:
  ingressClassName: nginx
  rules:
  - host: shop.local
    http:
      paths:
      - path: /api/products(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: products-api
            port:
              number: 80
      - path: /api/orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: orders-api
            port:
              number: 80
      - path: /api/users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: users-api
            port:
              number: 80
      - path: /()(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

```bash
# เพิ่ม hosts
echo "127.0.0.1 shop.local" >> /etc/hosts

kubectl apply -f ecommerce-ingress.yaml

# ทดสอบ routing
curl http://shop.local:8080/
curl http://shop.local:8080/api/products
curl http://shop.local:8080/api/orders
curl http://shop.local:8080/api/users
```

### Workshop 3: Host-based Routing

```yaml
# host-based-ingress.yaml
# Frontend via shop.local
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: ecommerce
spec:
  ingressClassName: nginx
  rules:
  - host: shop.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
---
# API via api.local
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: ecommerce
spec:
  ingressClassName: nginx
  rules:
  - host: api.local
    http:
      paths:
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: products-api
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: orders-api
            port:
              number: 80
```

```bash
echo "127.0.0.1 api.local" >> /etc/hosts

kubectl apply -f host-based-ingress.yaml

# ทดสอบ
curl http://shop.local:8080/
curl http://api.local:8080/products
curl http://api.local:8080/orders
```

### Workshop 4: TLS สำหรับ Production

```bash
# สร้าง Certificate สำหรับหลาย domains
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout shop.key \
  -out shop.crt \
  -subj "/CN=shop.local" \
  -addext "subjectAltName=DNS:shop.local,DNS:api.local"

# สร้าง TLS Secrets
kubectl create secret tls shop-tls \
  --key shop.key \
  --cert shop.crt \
  -n ecommerce
```

```yaml
# tls-ecommerce-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ecommerce
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - shop.local
    - api.local
    secretName: shop-tls
  rules:
  - host: shop.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: products-api
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: orders-api
            port:
              number: 80
```

```bash
kubectl apply -f tls-ecommerce-ingress.yaml

# ทดสอบ HTTPS
curl -k https://shop.local:8443/
curl -k https://api.local:8443/products

# ทดสอบ HTTP redirect
curl -v http://shop.local:8080/
# ควรได้ 301 redirect ไป https
```

### Workshop 5: Canary Deployment ด้วย Ingress

```yaml
# canary-ingress.yaml
# Production (stable) version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-stable
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      version: stable
  template:
    metadata:
      labels:
        app: frontend
        version: stable
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init
        image: busybox
        command: ["sh", "-c", "echo '<h1>Store v1 (Stable)</h1>' > /html/index.html"]
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-stable
  namespace: ecommerce
spec:
  selector:
    app: frontend
    version: stable
  ports:
  - port: 80
---
# Canary version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-canary
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
      version: canary
  template:
    metadata:
      labels:
        app: frontend
        version: canary
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init
        image: busybox
        command: ["sh", "-c", "echo '<h1>Store v2 (Canary - New Features!)</h1>' > /html/index.html"]
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-canary
  namespace: ecommerce
spec:
  selector:
    app: frontend
    version: canary
  ports:
  - port: 80
---
# Stable Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: stable-ingress
  namespace: ecommerce
spec:
  ingressClassName: nginx
  rules:
  - host: shop.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-stable
            port:
              number: 80
---
# Canary Ingress (20% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"  # 20% traffic
    # หรือ header-based:
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    # nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: shop.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-canary
            port:
              number: 80
```

```bash
kubectl apply -f canary-ingress.yaml

# ทดสอบ canary (ประมาณ 20% จะได้ canary response)
for i in {1..20}; do
  echo -n "Request $i: "
  curl -s http://shop.local:8080/ | grep -o "v[12] (.*)" 
done
```

### Workshop 6: Ingress Monitoring

```bash
# ดู Ingress status
kubectl get ingress -n ecommerce

# ดู Events
kubectl describe ingress ecommerce-ingress -n ecommerce

# NGINX Ingress metrics (ถ้าเปิด Prometheus)
curl http://localhost:10254/metrics

# ดู access logs
kubectl logs -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller -o name) \
  | tail -50

# ดู error logs
kubectl logs -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller -o name) \
  | grep -i error

# ดู nginx.conf
kubectl exec -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller -o name) \
  -- cat /etc/nginx/nginx.conf | head -100
```

### Cleanup

```bash
kubectl delete namespace ecommerce
kubectl delete secret shop-tls -n default
kind delete cluster --name routing-demo
```

---

## Advanced Ingress Patterns

### Ingress กับ Websocket

```yaml
# websocket-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
spec:
  ingressClassName: nginx
  rules:
  - host: ws.example.com
    http:
      paths:
      - path: /socket
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 8080
```

### Ingress กับ gRPC

```yaml
# grpc-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    # หรือ GRPCS ถ้า backend ใช้ TLS
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - grpc.example.com
    secretName: grpc-tls
  rules:
  - host: grpc.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grpc-service
            port:
              number: 9090
```

---

## Cheat Sheet

```bash
# CRUD Ingress
kubectl create ingress my-ingress --rule="host.com/path=service:port"
kubectl get ingress
kubectl describe ingress my-ingress
kubectl delete ingress my-ingress

# ดู Ingress class
kubectl get ingressclass

# TLS Secret
kubectl create secret tls my-tls --key tls.key --cert tls.crt

# ดู cert-manager certificates
kubectl get certificate
kubectl describe certificate my-cert
kubectl get certificaterequest
kubectl get orders -n cert-manager

# ดู Ingress rules
kubectl get ingress -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.rules[*]}{.host}{"\n"}{range .http.paths[*]}{.path}{" -> "}{.backend.service.name}:{.backend.service.port.number}{"\n"}{end}{end}{end}'

# Debug NGINX
kubectl exec -n ingress-nginx <controller-pod> -- nginx -T
kubectl exec -n ingress-nginx <controller-pod> -- nginx -s reload
```

---

*ก่อนหน้า: [Part 35 - Ingress Controllers](./part-35-ingress-controllers.md)*
*ต่อไป: [Part 37 - Network Policies](./part-37-network-policies.md)*
