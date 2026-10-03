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

### Lab 6: Query Pods ด้วย Annotations (Advanced)

```bash
# สร้าง Pods พร้อม Annotations ต่างๆ
kubectl create namespace annotation-query

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: monitored-pod
  namespace: annotation-query
  labels:
    app: monitored
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    team: "frontend"
    priority: "high"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: unmonitored-pod
  namespace: annotation-query
  labels:
    app: unmonitored
  annotations:
    prometheus.io/scrape: "false"
    team: "backend"
    priority: "low"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
  namespace: annotation-query
  labels:
    app: critical
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    team: "platform"
    priority: "critical"
    pagerduty-key: "PD-SERVICE-123"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
EOF

kubectl wait --for=condition=Ready pods --all \
    -n annotation-query --timeout=120s

# ค้นหา Pods ที่ Prometheus จะ scrape
kubectl get pods -n annotation-query \
    -o jsonpath='{range .items[?(@.metadata.annotations.prometheus\.io/scrape=="true")]}{.metadata.name}{"\n"}{end}'

# แสดง annotation ทุก Pods
kubectl get pods -n annotation-query \
    -o custom-columns='NAME:.metadata.name,SCRAPE:.metadata.annotations.prometheus\.io/scrape,TEAM:.metadata.annotations.team,PRIORITY:.metadata.annotations.priority'

# ดู annotation เฉพาะ pod
kubectl get pod critical-pod -n annotation-query \
    -o jsonpath='{.metadata.annotations}' | python3 -m json.tool

# Cleanup
kubectl delete namespace annotation-query
```

### Lab 7: Annotation-driven Automation Script

```bash
# สถานการณ์: Script อัตโนมัติที่อ่าน Annotations เพื่อทำงาน

# สร้าง Deployment ที่มี annotation สำหรับ auto-scaling policy
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-policy-app
  namespace: annotation-workshop
  annotations:
    autoscaling.policy/min-replicas: "2"
    autoscaling.policy/max-replicas: "10"
    autoscaling.policy/cpu-threshold: "70"
    backup.policy/enabled: "true"
    backup.policy/schedule: "0 2 * * *"
    backup.policy/retention-days: "30"
    monitoring.policy/enabled: "true"
    monitoring.policy/alert-slack: "#ops-alerts"
    monitoring.policy/on-call: "ops-team"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auto-policy-app
  template:
    metadata:
      labels:
        app: auto-policy-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF

# Script ที่อ่าน Annotations แล้วตัดสินใจ
cat <<'SCRIPT' > /tmp/policy-enforcer.sh
#!/bin/bash
# อ่าน autoscaling policy จาก annotations

NAMESPACE="${1:-annotation-workshop}"
DEPLOYMENT="${2:-auto-policy-app}"

echo "=== Policy Enforcer ==="
echo "Reading policies from: $DEPLOYMENT/$NAMESPACE"
echo ""

# อ่าน autoscaling policy
MIN_REPLICAS=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.autoscaling\.policy/min-replicas}')
MAX_REPLICAS=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.autoscaling\.policy/max-replicas}')
CPU_THRESHOLD=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.autoscaling\.policy/cpu-threshold}')
CURRENT_REPLICAS=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.spec.replicas}')

echo "Auto-scaling Policy:"
echo "  Min Replicas: ${MIN_REPLICAS:-not set}"
echo "  Max Replicas: ${MAX_REPLICAS:-not set}"
echo "  CPU Threshold: ${CPU_THRESHOLD:-not set}%"
echo "  Current Replicas: $CURRENT_REPLICAS"
echo ""

# Validate policy
if [ -n "$MIN_REPLICAS" ] && [ "$CURRENT_REPLICAS" -lt "$MIN_REPLICAS" ]; then
    echo "WARNING: Current replicas ($CURRENT_REPLICAS) below minimum ($MIN_REPLICAS)"
    echo "Action: Scaling up to minimum..."
    kubectl scale deployment $DEPLOYMENT -n $NAMESPACE --replicas=$MIN_REPLICAS
fi

# อ่าน backup policy
BACKUP_ENABLED=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.backup\.policy/enabled}')
BACKUP_SCHEDULE=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.backup\.policy/schedule}')

echo "Backup Policy:"
echo "  Enabled: ${BACKUP_ENABLED:-false}"
echo "  Schedule: ${BACKUP_SCHEDULE:-not set}"
echo ""

# อ่าน monitoring policy
MONITORING_ENABLED=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.monitoring\.policy/enabled}')
ALERT_SLACK=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.metadata.annotations.monitoring\.policy/alert-slack}')

echo "Monitoring Policy:"
echo "  Enabled: ${MONITORING_ENABLED:-false}"
echo "  Alert Slack: ${ALERT_SLACK:-not set}"
SCRIPT

chmod +x /tmp/policy-enforcer.sh
/tmp/policy-enforcer.sh annotation-workshop auto-policy-app

# Cleanup
kubectl delete deployment auto-policy-app -n annotation-workshop
rm -f /tmp/policy-enforcer.sh
```

### คำสั่ง Annotation ที่ใช้บ่อย

```bash
# เพิ่ม Annotation
kubectl annotate pod my-pod key=value
kubectl annotate deployment my-deploy key=value
kubectl annotate node my-node key=value

# เพิ่มหลาย Annotations
kubectl annotate pod my-pod \
    key1=value1 \
    key2=value2 \
    key3=value3

# แก้ไข Annotation (ต้องใช้ --overwrite)
kubectl annotate pod my-pod key=newvalue --overwrite

# ลบ Annotation (ต่อท้ายด้วย -)
kubectl annotate pod my-pod key-

# ดู Annotations
kubectl describe pod my-pod | grep -A 20 "Annotations:"
kubectl get pod my-pod -o jsonpath='{.metadata.annotations}'

# ดู Annotation เฉพาะ (ต้อง escape dots ด้วย \.)
kubectl get pod my-pod \
    -o jsonpath='{.metadata.annotations.kubernetes\.io/change-cause}'

# แสดง Annotation เป็น column
kubectl get pods \
    -o custom-columns='NAME:.metadata.name,CHANGE-CAUSE:.metadata.annotations.kubernetes\.io/change-cause'

# ค้นหา Resources ที่มี Annotation เฉพาะ
kubectl get pods -A \
    -o jsonpath='{range .items[?(@.metadata.annotations.prometheus\.io/scrape=="true")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}'
```

### Annotations Standard ที่ Kubernetes ใช้

```bash
# Annotations ที่ Kubernetes เองใช้:

# 1. kubectl.kubernetes.io/last-applied-configuration
# เก็บ manifest ล่าสุดที่ apply ด้วย kubectl apply
kubectl get pod my-pod \
    -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'

# 2. kubernetes.io/change-cause
# เก็บ reason ของ Deployment rollout
kubectl annotate deployment my-deploy \
    kubernetes.io/change-cause="Fix critical bug CVE-2024-001"

# 3. deployment.kubernetes.io/revision
# เก็บ revision number ของ Deployment (auto-managed)
kubectl get deployment my-deploy \
    -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}'

# 4. autoscaling.alpha.kubernetes.io/
# ใช้โดย HPA

# 5. cluster-autoscaler.kubernetes.io/
# ใช้โดย Cluster Autoscaler
kubectl annotate node my-node \
    cluster-autoscaler.kubernetes.io/scale-down-disabled="true"
```

---

## สรุป

Annotations เป็นส่วนสำคัญของ Kubernetes metadata system:

1. **ไม่ใช้ select**: ต่างจาก Labels ตรงที่ไม่ใช้ filter Resources
2. **เก็บข้อมูลเพิ่มเติม**: build info, deployment info, documentation
3. **Tool Integration**: Ingress controllers, Prometheus, Service Mesh ใช้ Annotations สำหรับ configuration
4. **Human Readable**: เก็บข้อมูลสำหรับ operators อ่านและ reference
5. **Automation**: Scripts และ tools อ่าน Annotations เพื่อตัดสินใจอัตโนมัติ

**Best Practices:**
- ใช้ prefix สำหรับ organization-specific annotations (mycompany.io/key)
- เก็บ build/deployment info ทุกครั้ง
- อย่าเก็บ secrets ใน Annotations (ใช้ Secrets object แทน)
- ใช้ JSON/YAML สำหรับ structured data ที่ซับซ้อน
- Document ว่า annotation แต่ละตัวหมายความว่าอะไร

ในบทต่อไปเราจะเรียนรู้ **ConfigMaps** ซึ่งใช้เก็บ configuration data สำหรับ Applications

---

## Annotations ที่ใช้บ่อยใน Production

### 1. Build และ CI/CD Annotations

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  annotations:
    # Build information
    build.ci/pipeline-id: "pipeline-2024-001"
    build.ci/run-id: "run-45678"
    build.ci/commit-sha: "abc123def456"
    build.ci/branch: "main"
    build.ci/build-time: "2024-01-15T10:30:00Z"
    build.ci/builder: "jenkins/2.387"
    
    # Docker image info
    image.registry/digest: "sha256:abc123..."
    image.registry/pushed-at: "2024-01-15T10:25:00Z"
    
    # Deployment info
    deployment.info/deployed-by: "john.doe@company.com"
    deployment.info/deployed-at: "2024-01-15T10:30:00Z"
    deployment.info/ticket: "JIRA-1234"
    deployment.info/changelog: "Fix critical security vulnerability CVE-2024-001"
```

### 2. Monitoring และ Alerting Annotations

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  annotations:
    # Prometheus scraping
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
    prometheus.io/scheme: "http"
    
    # Alert routing
    alert.pagerduty.com/service-id: "PXXXXXX"
    alert.pagerduty.com/escalation-policy: "default"
    alert.opsgenie.com/team: "backend-team"
    
    # SLA information
    sla.company.io/tier: "tier-1"
    sla.company.io/availability: "99.9"
    sla.company.io/rto: "15m"
    sla.company.io/rpo: "1h"
```

### 3. Networking และ Ingress Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    # Nginx Ingress Controller
    nginx.ingress.kubernetes.io/rewrite-target: "/"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.company.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
    
    # Certificate management (cert-manager)
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    cert-manager.io/acme-challenge-type: "http01"
spec:
  tls:
  - hosts:
    - api.company.com
    secretName: api-tls-cert
  rules:
  - host: api.company.com
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

### 4. Service Mesh Annotations (Istio)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice
  annotations:
    # Istio sidecar injection
    sidecar.istio.io/inject: "true"
    sidecar.istio.io/proxyCPU: "100m"
    sidecar.istio.io/proxyMemory: "128Mi"
    sidecar.istio.io/proxyCPULimit: "500m"
    sidecar.istio.io/proxyMemoryLimit: "256Mi"
    
    # Traffic management
    traffic.sidecar.istio.io/excludeOutboundPorts: "3306"
    
    # Tracing
    sidecar.jaegertracing.io/inject: "true"
spec:
  template:
    metadata:
      annotations:
        # Per-pod Istio settings
        proxy.istio.io/config: |
          holdApplicationUntilProxyStarts: true
```

### 5. Cluster Autoscaler Annotations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
  annotations:
    # ห้าม Cluster Autoscaler scale down node ที่รัน pod นี้
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

```yaml
apiVersion: v1
kind: Node
metadata:
  name: worker-node-01
  annotations:
    # ปิด scale down สำหรับ node นี้
    cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"
    # กำหนด node group
    cluster-autoscaler.kubernetes.io/node-group: "high-memory-nodes"
```

---

## Tool-specific Annotations

### kubectl.kubernetes.io/last-applied-configuration

เมื่อใช้ `kubectl apply` Kubernetes จะเก็บ manifest ล่าสุดไว้ใน annotation นี้

```bash
# ดู last-applied-configuration
kubectl get deployment my-deploy \
    -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' \
    | jq .

# Annotation นี้ใช้สำหรับ:
# 1. Three-way merge เมื่อ apply ครั้งต่อไป
# 2. ตรวจสอบว่า field ใดถูก manage โดย kubectl
# 3. ลบ fields ที่หายไปจาก manifest

# ลบ annotation นี้ถ้าต้องการ (ไม่แนะนำ)
kubectl annotate deployment my-deploy \
    kubectl.kubernetes.io/last-applied-configuration-
```

### kubernetes.io/change-cause

ใช้สำหรับ deployment history และ rollback

```bash
# ตั้ง change-cause ก่อน deploy
kubectl annotate deployment my-deploy \
    kubernetes.io/change-cause="Deploy version 2.0: add payment gateway" \
    --overwrite

# หรือตั้งระหว่าง apply
kubectl apply -f deployment.yaml
kubectl annotate deployment my-deploy \
    kubernetes.io/change-cause="v2.0: new features" \
    --overwrite

# ดู rollout history พร้อม change-cause
kubectl rollout history deployment/my-deploy

# Output:
# REVISION  CHANGE-CAUSE
# 1         Deploy version 1.0: initial release
# 2         Deploy version 2.0: add payment gateway
# 3         v2.0: new features

# Rollback ไปเวอร์ชันเก่า
kubectl rollout undo deployment/my-deploy --to-revision=2
```

### deployment.kubernetes.io/revision

```bash
# Annotation นี้ manage อัตโนมัติโดย Kubernetes
# ดู revision ปัจจุบัน
kubectl get deployment my-deploy \
    -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}'

# ดูใน ReplicaSet ด้วย
kubectl get rs \
    -l app=my-deploy \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.deployment\.kubernetes\.io/revision}{"\n"}{end}'
```

### autoscaling Annotations

```yaml
# HPA v2 สามารถใช้ annotations สำหรับ custom behavior
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
  annotations:
    autoscaling.alpha.kubernetes.io/conditions: |
      [{"type":"AbleToScale","status":"True"}]
    autoscaling.alpha.kubernetes.io/metrics: |
      [{"type":"Resource","resource":{"name":"cpu","currentAverageUtilization":45}}]
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Custom Annotations สำหรับ Internal Tools

### 1. Annotation Schema สำหรับองค์กร

```yaml
# กำหนด standard annotations ขององค์กร
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  annotations:
    # === Organization Info ===
    company.io/team: "backend"
    company.io/owner: "john.doe@company.com"
    company.io/slack-channel: "#backend-alerts"
    company.io/runbook: "https://wiki.company.io/runbook/my-service"
    company.io/repo: "https://github.com/company/my-service"
    
    # === Compliance ===
    compliance.company.io/data-classification: "confidential"
    compliance.company.io/pii-data: "false"
    compliance.company.io/gdpr-relevant: "true"
    compliance.company.io/last-security-review: "2024-01-01"
    
    # === Operations ===
    ops.company.io/maintenance-window: "Sunday 02:00-04:00 UTC"
    ops.company.io/backup-schedule: "daily"
    ops.company.io/dr-tier: "tier-1"
    ops.company.io/critical: "true"
    
    # === Cost ===
    cost.company.io/center: "CC-1234"
    cost.company.io/project: "project-alpha"
    cost.company.io/budget-code: "BUDGET-2024-001"
```

### 2. Annotation สำหรับ Deployment Pipeline

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  annotations:
    # GitOps annotations (ArgoCD)
    argocd.argoproj.io/managed-by: "argocd"
    argocd.argoproj.io/app-name: "api-service"
    
    # Flux annotations
    fluxcd.io/automated: "true"
    fluxcd.io/tag.api: "semver:~2.0"
    filter.fluxcd.io/api: "semver:^2.0.0"
    
    # Helm annotations
    meta.helm.sh/release-name: "api-service"
    meta.helm.sh/release-namespace: "production"
    helm.sh/chart: "api-service-1.5.0"
```

### 3. Script จัดการ Annotations

```bash
#!/bin/bash
# annotate-resources.sh - เพิ่ม standard annotations ให้ resources ทั้งหมด

NAMESPACE="${1:-default}"
TEAM="${2:-default-team}"
OWNER="${3:-ops@company.com}"

echo "Adding standard annotations to resources in namespace: $NAMESPACE"

# เพิ่ม annotations ให้ Deployments
for deploy in $(kubectl get deployments -n "$NAMESPACE" -o name); do
  kubectl annotate "$deploy" \
      -n "$NAMESPACE" \
      company.io/team="$TEAM" \
      company.io/owner="$OWNER" \
      company.io/annotated-at="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      --overwrite
  echo "Annotated: $deploy"
done

# เพิ่ม annotations ให้ Services
for svc in $(kubectl get services -n "$NAMESPACE" -o name); do
  kubectl annotate "$svc" \
      -n "$NAMESPACE" \
      company.io/team="$TEAM" \
      company.io/owner="$OWNER" \
      --overwrite
  echo "Annotated: $svc"
done

echo "Done! Annotated resources in $NAMESPACE"
```

### 4. Validation Script

```bash
#!/bin/bash
# validate-annotations.sh - ตรวจสอบว่า resources มี required annotations

REQUIRED_ANNOTATIONS=(
  "company.io/team"
  "company.io/owner"
  "company.io/runbook"
)

NAMESPACE="${1:-production}"
FAILED=0

echo "Checking required annotations in namespace: $NAMESPACE"
echo "Required: ${REQUIRED_ANNOTATIONS[*]}"
echo ""

for deploy in $(kubectl get deployments -n "$NAMESPACE" -o name); do
  MISSING=()
  for ann in "${REQUIRED_ANNOTATIONS[@]}"; do
    value=$(kubectl get "$deploy" -n "$NAMESPACE" \
        -o jsonpath="{.metadata.annotations.$ann}" 2>/dev/null)
    if [ -z "$value" ]; then
      MISSING+=("$ann")
    fi
  done
  
  if [ ${#MISSING[@]} -gt 0 ]; then
    echo "FAIL: $deploy"
    echo "  Missing annotations: ${MISSING[*]}"
    FAILED=$((FAILED + 1))
  else
    echo "PASS: $deploy"
  fi
done

echo ""
if [ $FAILED -gt 0 ]; then
  echo "FAILED: $FAILED resources missing required annotations"
  exit 1
else
  echo "All resources have required annotations"
fi
```

---

## แบบฝึกหัด: Annotations

### แบบฝึกหัดที่ 1: เพิ่ม Build Annotations

**โจทย์**: เพิ่ม annotations เกี่ยวกับ build info ให้ Deployment

**เฉลย**:
```bash
# สร้าง Deployment ก่อน
kubectl create deployment my-app --image=nginx

# เพิ่ม annotations ด้วย kubectl annotate
kubectl annotate deployment my-app \
    build.ci/commit="abc123" \
    build.ci/branch="main" \
    build.ci/pipeline="pipeline-001" \
    deployment.info/deployed-by="john@company.com" \
    deployment.info/deployed-at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# ตรวจสอบ
kubectl describe deployment my-app | grep -A10 "Annotations:"
```

---

### แบบฝึกหัดที่ 2: Annotation สำหรับ Prometheus Scraping

**โจทย์**: ตั้ง annotations ให้ Pod เพื่อให้ Prometheus scrape metrics

**เฉลย**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: metrics-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: app
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
    - containerPort: 9090
      name: metrics
```

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: metrics-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: app
    image: nginx:1.21
EOF

# ตรวจสอบ annotations
kubectl get pod metrics-app \
    -o jsonpath='{.metadata.annotations}' | jq
```

---

### แบบฝึกหัดที่ 3: ใช้ change-cause สำหรับ Rollout History

**โจทย์**: Deploy application 3 versions พร้อม change-cause แล้วดู history

**เฉลย**:
```bash
# Deploy version 1
kubectl create deployment rollout-demo \
    --image=nginx:1.19
kubectl annotate deployment rollout-demo \
    kubernetes.io/change-cause="Initial deployment: nginx 1.19"

# Deploy version 2
kubectl set image deployment/rollout-demo \
    nginx=nginx:1.20
kubectl annotate deployment rollout-demo \
    kubernetes.io/change-cause="Upgrade: nginx 1.20 - security patch" \
    --overwrite

# Deploy version 3
kubectl set image deployment/rollout-demo \
    nginx=nginx:1.21
kubectl annotate deployment rollout-demo \
    kubernetes.io/change-cause="Upgrade: nginx 1.21 - latest stable" \
    --overwrite

# ดู history
kubectl rollout history deployment/rollout-demo
# REVISION  CHANGE-CAUSE
# 1         Initial deployment: nginx 1.19
# 2         Upgrade: nginx 1.20 - security patch
# 3         Upgrade: nginx 1.21 - latest stable

# Rollback ไป version 2
kubectl rollout undo deployment/rollout-demo --to-revision=2
```

---

### แบบฝึกหัดที่ 4: ลบ Annotation

**โจทย์**: ลบ annotations ที่ไม่ต้องการออกจาก Deployment

**เฉลย**:
```bash
# ดู annotations ที่มี
kubectl get deployment my-app \
    -o jsonpath='{.metadata.annotations}' | jq keys

# ลบ annotation เฉพาะตัว (ใส่ - ท้ายชื่อ)
kubectl annotate deployment my-app \
    build.ci/pipeline-

# ลบหลาย annotations พร้อมกัน
kubectl annotate deployment my-app \
    build.ci/commit- \
    build.ci/branch-

# ตรวจสอบ
kubectl get deployment my-app \
    -o jsonpath='{.metadata.annotations}' | jq
```

---

### แบบฝึกหัดที่ 5: Nginx Ingress Annotations

**โจทย์**: สร้าง Ingress ที่มี rate limiting และ SSL redirect ด้วย annotations

**เฉลย**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    # Nginx controller
    kubernetes.io/ingress.class: "nginx"
    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    # Timeout
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    # Body size
    nginx.ingress.kubernetes.io/proxy-body-size: "5m"
spec:
  rules:
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
```

---

## สรุปทบทวน Annotations

### Cheat Sheet

```bash
# เพิ่ม annotation
kubectl annotate <resource> <name> key=value

# เพิ่ม/อัพเดต (overwrite)
kubectl annotate <resource> <name> key=value --overwrite

# ลบ annotation
kubectl annotate <resource> <name> key-

# ดู annotations
kubectl describe <resource> <name> | grep -A20 Annotations:
kubectl get <resource> <name> -o jsonpath='{.metadata.annotations}'

# ดู annotation เฉพาะตัว
kubectl get <resource> <name> \
    -o jsonpath='{.metadata.annotations.KEY}'
```

### Annotations vs Labels - สรุปความแตกต่าง

| Feature | Labels | Annotations |
|---------|--------|-------------|
| ใช้เลือก Resources | ใช่ | ไม่ |
| ขนาด value | จำกัด | ใหญ่ได้ (structured data) |
| ใช้ใน Selector | ใช่ | ไม่ |
| ใช้เก็บ metadata | ได้ | ได้ (เหมาะกว่า) |
| Tool integration | บางส่วน | หลักๆ |

Annotations เป็น metadata layer ที่ช่วยให้ ecosystem tools เช่น Ingress controllers, monitoring systems, service meshes และ CI/CD pipelines สามารถ configure behavior ของ Kubernetes resources ได้อย่างยืดหยุ่น
