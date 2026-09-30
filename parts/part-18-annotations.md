# Part 18: Annotations - Metadata ที่ไม่ใช้ Select

## สารบัญ
1. [Annotations คืออะไร](#annotations-คืออะไร)
2. [ความแตกต่างจาก Labels](#ความแตกต่างจาก-labels)
3. [Use Cases](#use-cases)
4. [Workshop: ใช้ Annotations จริง](#workshop)

---

## 1. Annotations คืออะไร

**Annotations** คือ key-value pairs ที่ attach กับ Kubernetes Objects เพื่อเก็บ metadata ที่ไม่ได้ใช้สำหรับ selection/filtering

### ลักษณะของ Annotations

```yaml
metadata:
  annotations:
    key: value
    prefix/key: value
```

**ข้อแตกต่างจาก Labels:**
- ไม่มี restriction ขนาด value (ใหญ่ได้มาก)
- ไม่สามารถใช้กับ selectors
- ใช้เก็บข้อมูลที่ human/tools อ่านเพื่อ reference

### Annotation Syntax

```
Key format:
  [prefix/]name
  
  prefix (optional):
  - DNS subdomain
  - ไม่เกิน 253 characters
  
  name (required):
  - ตัวอักษร ตัวเลข - _ .
  - ขึ้นต้นและจบด้วย alphanumeric
  - ไม่เกิน 63 characters

Value:
  - String (ไม่ต้องเป็น alphanumeric)
  - ไม่จำกัดขนาด (practical limit ~256KB)
  - สามารถเป็น JSON, YAML, URL, etc.
```

---

## 2. ความแตกต่างจาก Labels

### เปรียบเทียบ Labels vs Annotations

| Feature | Labels | Annotations |
|---------|--------|-------------|
| **วัตถุประสงค์** | Identify/Group Objects | Store metadata |
| **Querying** | ✓ kubectl -l selector | ✗ ไม่รองรับ |
| **ขนาด value** | ≤ 63 chars | ไม่จำกัด |
| **Value format** | alphanumeric/- /_ | ใดก็ได้ |
| **Kubernetes ใช้** | Selectors, Scheduling | Reference, Config |
| **ตัวอย่าง** | `app=nginx`, `env=prod` | `description`, `build-url` |

### เมื่อไหร่ควรใช้อะไร

```
ใช้ LABELS เมื่อ:
✓ ต้องการ filter Pods ด้วย Service selector
✓ ต้องการ group Resources เพื่อ kubectl operations
✓ ต้องการ scheduling decisions (nodeSelector, affinity)
✓ ต้องการ monitoring/alerting filters

ใช้ ANNOTATIONS เมื่อ:
✓ เก็บ URL ของ build pipeline
✓ เก็บ contact information
✓ เก็บ change history
✓ ส่ง configuration ให้ tools (ingress-nginx, prometheus)
✓ เก็บ JSON data
✓ เก็บ ข้อมูลที่ labels ขนาดไม่พอ
```

---

## 3. Use Cases

### Use Case 1: Build และ Deployment Information

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    # Build Information
    build.company.io/build-number: "12345"
    build.company.io/build-url: "https://jenkins.company.io/job/webapp/12345"
    build.company.io/git-commit: "a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
    build.company.io/git-branch: "main"
    build.company.io/git-repo: "https://github.com/company/webapp"
    
    # Deployment Information
    deployment.company.io/deployed-by: "john.doe@company.com"
    deployment.company.io/deployed-at: "2024-01-15T14:30:00Z"
    deployment.company.io/change-ticket: "JIRA-1234"
    deployment.company.io/changelog: |
      - Fixed login bug (JIRA-1230)
      - Added dark mode (JIRA-1231)
      - Performance improvements
```

### Use Case 2: Tool Configuration (Ingress-Nginx)

```yaml
# ingress-with-annotations.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    # Ingress Controller
    kubernetes.io/ingress.class: "nginx"
    
    # SSL Redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-XSS-Protection: 1; mode=block";
    
    # Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # Whitelist
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
    
    # Cert-manager (auto SSL certificate)
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    cert-manager.io/acme-challenge-type: "http01"
spec:
  tls:
  - hosts:
    - webapp.example.com
    secretName: webapp-tls
  rules:
  - host: webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

### Use Case 3: Prometheus Monitoring

```yaml
# deployment-with-prometheus-annotations.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitored-app
spec:
  template:
    metadata:
      annotations:
        # Prometheus scraping config
        prometheus.io/scrape: "true"       # บอกให้ Prometheus scrape
        prometheus.io/path: "/metrics"     # path ของ metrics endpoint
        prometheus.io/port: "9090"         # port ของ metrics
        prometheus.io/scheme: "http"       # http หรือ https
        
        # Grafana annotations (optional)
        grafana.io/dashboard: "webapp-dashboard"
        grafana.io/panel-id: "1"
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
```

### Use Case 4: Kubernetes Change Cause

```yaml
# เก็บ history ของการเปลี่ยนแปลง
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    # Kubernetes จะใช้นี้เป็น rollout history message
    kubernetes.io/change-cause: "Update to v2.0: Added OAuth2 support (JIRA-1234)"
```

```bash
# ดู history พร้อม change cause
kubectl rollout history deployment/webapp
# REVISION  CHANGE-CAUSE
# 1         Initial deployment - v1.0
# 2         Update to v1.5: Bug fixes
# 3         Update to v2.0: Added OAuth2 support (JIRA-1234)
```

### Use Case 5: Pod Annotations สำหรับ Service Mesh

```yaml
# pod-with-istio-annotations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: service-mesh-app
  annotations:
    # Istio sidecar injection
    sidecar.istio.io/inject: "true"
    
    # Traffic management
    traffic.sidecar.istio.io/includeInboundPorts: "8080"
    traffic.sidecar.istio.io/excludeOutboundIPRanges: "10.0.0.0/8"
    
    # Proxy resource limits
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "500m"
    sidecar.istio.io/proxyMemoryLimit: "512Mi"
    
    # Linkerd annotations
    linkerd.io/inject: "enabled"
    config.linkerd.io/proxy-cpu-request: "100m"
    config.linkerd.io/proxy-memory-request: "128Mi"
```

### Use Case 6: Node Annotations

```yaml
# node-annotations.yaml
# Annotations บน Nodes สำหรับ cluster management
# (Apply ผ่าน kubectl annotate)
apiVersion: v1
kind: Node
metadata:
  name: worker-1
  annotations:
    # Hardware information
    node.company.io/cpu-model: "Intel Xeon E5-2690"
    node.company.io/memory-gb: "64"
    node.company.io/disk-type: "nvme-ssd"
    node.company.io/network-speed: "10gbps"
    
    # Location
    topology.kubernetes.io/zone: "us-east-1a"
    topology.kubernetes.io/region: "us-east-1"
    
    # Maintenance
    node.company.io/last-maintenance: "2024-01-10"
    node.company.io/next-maintenance: "2024-04-10"
    node.company.io/maintenance-window: "Sunday 02:00-06:00 UTC"
    
    # Cluster autoscaler annotations
    cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"
```

### Use Case 7: PodDisruptionBudget Annotations

```yaml
# annotations-in-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  annotations:
    description: "Ensure at least 2 webapp pods are available during disruptions"
    team: "frontend-team"
    contact: "frontend-team@example.com"
    runbook: "https://wiki.example.com/runbooks/webapp-pdb"
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: webapp
```

---

## 4. Workshop: ใช้ Annotations จริง

### Workshop Setup

```bash
# สร้าง namespace
kubectl create namespace annotation-workshop
kubectl config set-context --current --namespace=annotation-workshop
```

### Lab 1: เพิ่มและจัดการ Annotations

```bash
# สร้าง Pod ธรรมดา
kubectl run test-pod \
    --image=nginx:1.25 \
    --namespace=annotation-workshop

kubectl wait --for=condition=Ready pod/test-pod --timeout=60s

# เพิ่ม Annotation ด้วย kubectl annotate
kubectl annotate pod test-pod \
    description="Test pod for annotation workshop"

kubectl annotate pod test-pod \
    contact="team@example.com"

kubectl annotate pod test-pod \
    build-number="12345" \
    git-commit="abc123" \
    deployed-by="john"

# ดู Annotations
kubectl describe pod test-pod | grep -A20 "Annotations:"

# ดู Annotations ผ่าน jsonpath
kubectl get pod test-pod \
    -o jsonpath='{.metadata.annotations}' | python3 -m json.tool

# แก้ไข Annotation
kubectl annotate pod test-pod \
    description="Updated description" \
    --overwrite

# ลบ Annotation
kubectl annotate pod test-pod contact-    # ลบ contact annotation

# ดูอีกครั้ง
kubectl get pod test-pod \
    -o jsonpath='{.metadata.annotations}'
```

### Lab 2: Deploy Application พร้อม Complete Annotations

```bash
cat <<'EOF' > /tmp/annotated-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-webapp
  namespace: annotation-workshop
  labels:
    app: production-webapp
    env: production
    version: "2.5.1"
  annotations:
    # === Build Information ===
    build.io/build-number: "54321"
    build.io/build-url: "https://ci.example.com/builds/54321"
    build.io/git-commit: "f7e6d5c4b3a2918172636455443322110fedcba9"
    build.io/git-branch: "release/2.5.1"
    build.io/git-repo: "https://github.com/example/webapp"
    build.io/built-at: "2024-01-15T12:00:00Z"
    build.io/built-by: "ci-bot"
    
    # === Deployment Information ===
    deployment.io/deployed-at: "2024-01-15T14:30:00Z"
    deployment.io/deployed-by: "john.doe@example.com"
    deployment.io/approved-by: "jane.smith@example.com"
    deployment.io/change-ticket: "CHANGE-2024-001"
    
    # === Application Information ===
    app.io/description: "Production web application serving the main website"
    app.io/team: "Frontend Platform Team"
    app.io/contact: "frontend-platform@example.com"
    app.io/slack-channel: "#frontend-platform"
    app.io/pagerduty-service: "webapp-production"
    
    # === Runbooks and Documentation ===
    docs.io/runbook: "https://wiki.example.com/runbooks/webapp"
    docs.io/architecture: "https://wiki.example.com/architecture/webapp"
    docs.io/sla: "99.9% uptime, p99 < 200ms"
    
    # === Changelog (JSON array) ===
    changelog.io/last-changes: |
      [
        {"version": "2.5.1", "date": "2024-01-15", "changes": ["Fix login timeout", "Update dependencies"]},
        {"version": "2.5.0", "date": "2024-01-10", "changes": ["Add dark mode", "Improve search"]}
      ]
    
    # Kubernetes rollout change-cause
    kubernetes.io/change-cause: "Release 2.5.1: Fix login timeout and security updates"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-webapp
  template:
    metadata:
      labels:
        app: production-webapp
        env: production
        version: "2.5.1"
      annotations:
        # === Pod-level Annotations ===
        # Prometheus scraping
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "9090"
        
        # Pod scheduling annotation
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF

kubectl apply -f /tmp/annotated-deployment.yaml

# ดู annotations
kubectl describe deployment production-webapp | grep -A 30 "Annotations:"

# ดู specific annotation
kubectl get deployment production-webapp \
    -o jsonpath='{.metadata.annotations.kubernetes\.io/change-cause}'

# ดู changelog annotation
kubectl get deployment production-webapp \
    -o jsonpath='{.metadata.annotations.changelog\.io/last-changes}'

# ดู Pod annotations
kubectl get pods -l app=production-webapp \
    -o jsonpath='{.items[0].metadata.annotations}' | python3 -m json.tool
```

### Lab 3: Annotations สำหรับ Ingress Controller

```bash
# สร้าง Service ก่อน
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: annotation-workshop
spec:
  selector:
    app: production-webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# สร้าง Ingress พร้อม Annotations
cat <<'EOF' > /tmp/ingress-with-annotations.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: annotation-workshop
  annotations:
    # nginx ingress controller
    kubernetes.io/ingress.class: "nginx"
    
    # Redirect HTTP to HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Proxy timeouts (สำหรับ slow APIs)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    
    # Max upload size
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Enable CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rpm: "100"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    
    # Custom error pages
    nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
    nginx.ingress.kubernetes.io/default-backend: "error-page-service"
    
    # Description
    description: "Main webapp ingress with security headers and CORS"
spec:
  rules:
  - host: webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
EOF

kubectl apply -f /tmp/ingress-with-annotations.yaml

# ดู Ingress annotations
kubectl describe ingress webapp-ingress

# ลบ Ingress
kubectl delete ingress webapp-ingress
```

### Lab 4: Script ที่ Read Annotations

```bash
# สคริปต์ที่อ่าน Annotations เพื่อทำงานต่างๆ
cat <<'SCRIPT' > /tmp/check-annotations.sh
#!/bin/bash

NAMESPACE="${1:-default}"
DEPLOYMENT="${2:-production-webapp}"

echo "=== Deployment Annotations Report ==="
echo "Deployment: $DEPLOYMENT"
echo "Namespace: $NAMESPACE"
echo ""

# ดึง annotations
BUILD_NUMBER=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.build\.io/build-number}' 2>/dev/null)
BUILD_URL=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.build\.io/build-url}' 2>/dev/null)
GIT_COMMIT=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.build\.io/git-commit}' 2>/dev/null)
DEPLOYED_BY=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.deployment\.io/deployed-by}' 2>/dev/null)
DEPLOYED_AT=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.deployment\.io/deployed-at}' 2>/dev/null)
CHANGE_CAUSE=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.kubernetes\.io/change-cause}' 2>/dev/null)

echo "Build Information:"
echo "  Build #: ${BUILD_NUMBER:-N/A}"
echo "  Build URL: ${BUILD_URL:-N/A}"
echo "  Git Commit: ${GIT_COMMIT:0:8}${GIT_COMMIT:+...}"
echo ""
echo "Deployment Information:"
echo "  Deployed By: ${DEPLOYED_BY:-N/A}"
echo "  Deployed At: ${DEPLOYED_AT:-N/A}"
echo "  Change Cause: ${CHANGE_CAUSE:-N/A}"
SCRIPT

chmod +x /tmp/check-annotations.sh
/tmp/check-annotations.sh annotation-workshop production-webapp
```

### Lab 5: kubectl explain สำหรับ Annotations

```bash
# ดู documentation ของ annotations
kubectl explain pod.metadata.annotations
kubectl explain deployment.metadata.annotations

# ดู annotations ของ Pods ทั้งหมดใน namespace
kubectl get pods -n annotation-workshop \
    -o custom-columns='NAME:.metadata.name,ANNOTATIONS:.metadata.annotations'

# ค้นหา Pods ที่มี annotation เฉพาะ
kubectl get pods -n annotation-workshop \
    -o jsonpath='{range .items[?(@.metadata.annotations.prometheus\.io/scrape=="true")]}{.metadata.name}{"\n"}{end}'
```

### Cleanup Workshop

```bash
# ลบทุกอย่าง
kubectl delete namespace annotation-workshop

# กลับ default namespace
kubectl config set-context --current --namespace=default

# ลบ temp files
rm -f /tmp/annotated-deployment.yaml \
    /tmp/ingress-with-annotations.yaml \
    /tmp/check-annotations.sh
```

### Common Annotation Patterns

```bash
# Pattern 1: เพิ่ม annotation ผ่าน kubectl
kubectl annotate deployment webapp \
    kubernetes.io/change-cause="Deploy v2.0: New features"

# Pattern 2: เพิ่มหลาย annotations ในครั้งเดียว
kubectl annotate pod nginx \
    app.io/contact="team@example.com" \
    app.io/description="Production nginx" \
    monitoring.io/enabled="true"

# Pattern 3: ดู annotation ที่ specific
kubectl get pod nginx \
    -o jsonpath='{.metadata.annotations.app\.io/contact}'

# Pattern 4: ดู annotations ทั้งหมดแบบ formatted
kubectl get pod nginx -o yaml | \
    python3 -c "import sys,yaml; d=yaml.safe_load(sys.stdin); print(yaml.dump(d['metadata'].get('annotations', {})))"

# Pattern 5: ค้นหา Resources ด้วย annotation value
kubectl get pods --all-namespaces \
    -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.team}{"\n"}{end}' | \
    grep "frontend-team"
```

---

## สรุป

Annotations เป็นส่วนสำคัญของ Kubernetes metadata system:

1. **ไม่ใช้ select**: ต่างจาก Labels ตรงที่ไม่ใช้ filter Resources
2. **เก็บข้อมูลเพิ่มเติม**: build info, deployment info, documentation
3. **Tool Integration**: Ingress controllers, Prometheus, Service Mesh ใช้ Annotations สำหรับ configuration
4. **Human Readable**: เก็บข้อมูลสำหรับ operators อ่านและ reference

**Best Practices:**
- ใช้ prefix สำหรับ organization-specific annotations
- เก็บ build/deployment info ทุกครั้ง
- อย่าเก็บ secrets ใน Annotations
- ใช้ JSON/YAML สำหรับ structured data

ในบทต่อไปเราจะเรียนรู้ **ConfigMaps** ซึ่งใช้เก็บ configuration data สำหรับ Applications
