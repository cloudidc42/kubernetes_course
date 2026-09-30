# Part 79: Helm Advanced - Hooks, Tests, Dependencies และ Production Chart

## บทนำ

ในบทที่แล้วเราเรียนรู้ Helm พื้นฐาน ในบทนี้จะเจาะลึก Advanced Features ที่จำเป็นสำหรับ Production-grade Helm Charts ได้แก่ Hooks สำหรับ Pre/Post Install Actions, Tests สำหรับ Automated Testing, Chart Dependencies, Helm Plugin และการสร้าง Production-ready Helm Chart

## 1. Helm Hooks

### 1.1 Hook Types

Hooks คือ Kubernetes Resources ที่รันในช่วงเวลาเฉพาะระหว่าง Release Lifecycle

```
Release Lifecycle:
┌─────────────────────────────────────────────────────────────┐
│                  helm install                               │
│                                                             │
│  pre-install ──> resources created ──> post-install        │
│                                                             │
│                  helm upgrade                               │
│                                                             │
│  pre-upgrade ──> resources upgraded ──> post-upgrade       │
│                                                             │
│                  helm rollback                              │
│                                                             │
│  pre-rollback ──> resources rolled back ──> post-rollback  │
│                                                             │
│                  helm uninstall                             │
│                                                             │
│  pre-delete ──> resources deleted ──> post-delete          │
└─────────────────────────────────────────────────────────────┘
```

**Hook Types:**
- `pre-install` - รันก่อน Resources ถูกสร้าง
- `post-install` - รันหลัง Resources ถูกสร้าง
- `pre-upgrade` - รันก่อน Resources ถูก Upgrade
- `post-upgrade` - รันหลัง Resources ถูก Upgrade
- `pre-rollback` - รันก่อน Rollback
- `post-rollback` - รันหลัง Rollback
- `pre-delete` - รันก่อน Resources ถูกลบ
- `post-delete` - รันหลัง Resources ถูกลบ
- `test` - รันเมื่อใช้ helm test

### 1.2 Database Migration Hook

```gotemplate
# templates/hooks/pre-upgrade-migration.yaml
{{- if .Values.migrations.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    # Hook Annotations
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
        job-type: migration
    spec:
      restartPolicy: Never
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      
      initContainers:
      - name: wait-for-database
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          until nc -z {{ .Values.database.host }} {{ .Values.database.port }}; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"
      
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        command: ["./migrate", "up"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-db-secret
              key: url
        - name: MIGRATION_PATH
          value: "/app/migrations"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
{{- end }}
```

### 1.3 Notification Hook

```gotemplate
# templates/hooks/post-install-notify.yaml
{{- if .Values.notifications.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-notify
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-delete-policy": hook-succeeded,hook-failed
    "helm.sh/hook-weight": "10"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: notify
        image: curlimages/curl:8.4.0
        command:
        - sh
        - -c
        - |
          REVISION="{{ .Release.Revision }}"
          RELEASE="{{ .Release.Name }}"
          NAMESPACE="{{ .Release.Namespace }}"
          VERSION="{{ .Chart.AppVersion }}"
          
          curl -X POST "$SLACK_WEBHOOK" \
            -H "Content-type: application/json" \
            --data "{
              \"text\": \"Deployment Complete\",
              \"attachments\": [{
                \"color\": \"good\",
                \"fields\": [
                  {\"title\": \"Release\", \"value\": \"$RELEASE\", \"short\": true},
                  {\"title\": \"Version\", \"value\": \"$VERSION\", \"short\": true},
                  {\"title\": \"Namespace\", \"value\": \"$NAMESPACE\", \"short\": true},
                  {\"title\": \"Revision\", \"value\": \"$REVISION\", \"short\": true}
                ]
              }]
            }"
        env:
        - name: SLACK_WEBHOOK
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-slack-secret
              key: webhook-url
{{- end }}
```

### 1.4 Cleanup Hook

```gotemplate
# templates/hooks/pre-delete-cleanup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-cleanup
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: cleanup
        image: bitnami/kubectl:1.28
        command:
        - sh
        - -c
        - |
          # ล้าง PVCs ที่ไม่ได้ลบโดยอัตโนมัติ
          kubectl delete pvc -l app.kubernetes.io/instance={{ .Release.Name }} \
            -n {{ .Release.Namespace }} \
            --ignore-not-found
          
          # ล้าง Jobs เก่า
          kubectl delete jobs -l app.kubernetes.io/instance={{ .Release.Name }} \
            -n {{ .Release.Namespace }} \
            --ignore-not-found
          
          echo "Cleanup complete"
```

### 1.5 Hook Weights และ Delete Policies

```yaml
# Hook Weight: ลำดับการรัน Hooks (น้อยกว่า = รันก่อน)
annotations:
  "helm.sh/hook": pre-install
  "helm.sh/hook-weight": "-10"  # รันก่อนสุด
  "helm.sh/hook-weight": "0"    # default
  "helm.sh/hook-weight": "10"   # รันหลัง

# Delete Policy: เมื่อไหนจะลบ Hook Resource
annotations:
  "helm.sh/hook-delete-policy": "before-hook-creation"  # ลบก่อนสร้าง Hook ใหม่
  "helm.sh/hook-delete-policy": "hook-succeeded"         # ลบเมื่อสำเร็จ
  "helm.sh/hook-delete-policy": "hook-failed"            # ลบเมื่อล้มเหลว
  # สามารถใช้หลาย Policy
  "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
```

## 2. Helm Tests

### 2.1 Test Structure

```gotemplate
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test-connection
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
  - name: test-connection
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      # Test Service Connection
      if wget -q -O- http://{{ include "myapp.fullname" . }}/health; then
        echo "Connection test passed"
        exit 0
      else
        echo "Connection test failed"
        exit 1
      fi
```

### 2.2 Comprehensive Tests

```gotemplate
# templates/tests/test-api.yaml
{{- if .Values.tests.enabled }}
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test-api
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
  - name: test-api
    image: curlimages/curl:8.4.0
    command:
    - sh
    - -c
    - |
      BASE_URL="http://{{ include "myapp.fullname" . }}.{{ .Release.Namespace }}"
      PASS=0
      FAIL=0
      
      echo "=== Testing {{ .Release.Name }} API ==="
      
      # Test 1: Health Check
      echo ""
      echo "Test 1: Health Check"
      RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $BASE_URL/health)
      if [ "$RESPONSE" = "200" ]; then
        echo "  PASS: Health check returned 200"
        PASS=$((PASS+1))
      else
        echo "  FAIL: Health check returned $RESPONSE"
        FAIL=$((FAIL+1))
      fi
      
      # Test 2: Ready Check
      echo ""
      echo "Test 2: Ready Check"
      RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $BASE_URL/ready)
      if [ "$RESPONSE" = "200" ]; then
        echo "  PASS: Ready check returned 200"
        PASS=$((PASS+1))
      else
        echo "  FAIL: Ready check returned $RESPONSE"
        FAIL=$((FAIL+1))
      fi
      
      # Test 3: API Endpoint
      echo ""
      echo "Test 3: API Endpoint"
      RESPONSE=$(curl -s $BASE_URL/api/v1/version)
      if echo "$RESPONSE" | grep -q "version"; then
        echo "  PASS: API returns version"
        PASS=$((PASS+1))
      else
        echo "  FAIL: API did not return version: $RESPONSE"
        FAIL=$((FAIL+1))
      fi
      
      # Test 4: Metrics (if enabled)
      {{- if .Values.metrics.enabled }}
      echo ""
      echo "Test 4: Metrics Endpoint"
      RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $BASE_URL/metrics)
      if [ "$RESPONSE" = "200" ]; then
        echo "  PASS: Metrics endpoint returned 200"
        PASS=$((PASS+1))
      else
        echo "  FAIL: Metrics endpoint returned $RESPONSE"
        FAIL=$((FAIL+1))
      fi
      {{- end }}
      
      echo ""
      echo "=== Test Results: $PASS passed, $FAIL failed ==="
      
      if [ $FAIL -gt 0 ]; then
        exit 1
      fi
      exit 0
{{- end }}
```

### 2.3 รัน Tests

```bash
# รัน Helm Tests
helm test my-release

# รัน Tests พร้อมดู Logs
helm test my-release --logs

# ดู Test Results
kubectl get pods -l helm.sh/chart=myapp-1.0.0 -n production

# ดู Logs ของ Test Pod
kubectl logs myapp-test-api -n production
```

## 3. Chart Dependencies

### 3.1 กำหนด Dependencies ใน Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
# PostgreSQL Database
- name: postgresql
  version: ">=12.0.0 <13.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
  alias: db  # ใช้ .Values.db แทน .Values.postgresql

# Redis Cache
- name: redis
  version: ">=17.0.0 <18.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled
  tags:
  - cache

# Elasticsearch
- name: elasticsearch
  version: ">=19.0.0 <20.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: elasticsearch.enabled
  tags:
  - search

# Shared Library Chart
- name: common
  version: "2.x.x"
  repository: https://charts.bitnami.com/bitnami

# Local Chart (ใน charts/ directory)
- name: internal-service
  version: ">=1.0.0"
  repository: "file://charts/internal-service"
```

### 3.2 Update Dependencies

```bash
# Download Dependencies
helm dependency update myapp/

# หลัง Update จะมี:
# myapp/
# ├── Chart.lock       (lock file)
# └── charts/
#     ├── postgresql-12.x.x.tgz
#     └── redis-17.x.x.tgz

# Build Dependencies (จาก Chart.lock)
helm dependency build myapp/

# ดู Dependencies
helm dependency list myapp/
```

### 3.3 ตั้งค่า Dependency Values

```yaml
# values.yaml
# PostgreSQL Configuration (using 'db' alias)
postgresql:
  enabled: true
  
  # หรือ ถ้าใช้ alias 'db':
db:
  enabled: true
  auth:
    database: myapp
    username: myapp
    existingSecret: "myapp-db-secret"
  
  primary:
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi
    
    persistence:
      enabled: true
      size: 20Gi
      storageClass: fast-storage
    
    configuration: |
      max_connections = 200
      shared_buffers = 256MB
      effective_cache_size = 768MB

# Redis Configuration
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: "myapp-redis-secret"
    existingSecretPasswordKey: "password"
  
  master:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
    
    persistence:
      enabled: true
      size: 5Gi
```

### 3.4 Library Charts

```yaml
# charts/common/Chart.yaml
apiVersion: v2
name: common
description: Shared library chart
type: library    # <-- Library Chart!
version: 1.0.0
```

```gotemplate
# charts/common/templates/_deployment.tpl
{{- define "common.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "common.fullname" . }}
  labels: {{ include "common.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  selector:
    matchLabels: {{ include "common.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{ include "common.labels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ include "common.image" . }}
        ports:
        - containerPort: {{ .Values.service.targetPort | default 8080 }}
        resources: {{ toYaml .Values.resources | nindent 10 }}
{{- end -}}
```

## 4. Advanced Template Techniques

### 4.1 Schema Validation

```json
// values.schema.json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "title": "MyApp Values",
  "type": "object",
  "required": ["image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "default": 1
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "description": "Container image repository"
        },
        "tag": {
          "type": "string"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"],
          "default": "IfNotPresent"
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"],
          "default": "ClusterIP"
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    },
    "ingress": {
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean"
        }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "requests": {
          "type": "object",
          "properties": {
            "cpu": {"type": "string"},
            "memory": {"type": "string"}
          }
        },
        "limits": {
          "type": "object",
          "properties": {
            "cpu": {"type": "string"},
            "memory": {"type": "string"}
          }
        }
      }
    }
  }
}
```

### 4.2 Custom Validate Templates

```gotemplate
# templates/_validate.tpl
{{- define "myapp.validate" -}}
{{/* Validate required values */}}
{{- if not .Values.image.repository -}}
{{- fail "image.repository is required" -}}
{{- end -}}

{{/* Validate ingress TLS */}}
{{- if and .Values.ingress.enabled .Values.ingress.tls -}}
{{- range .Values.ingress.tls -}}
{{- if not .secretName -}}
{{- fail "ingress.tls.secretName is required when TLS is enabled" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{/* Validate resource requests <= limits */}}
{{- end -}}

# templates/deployment.yaml
{{- include "myapp.validate" . -}}
# ... rest of deployment
```

### 4.3 Sprig Functions

```gotemplate
# Date/Time
{{ now | date "2006-01-02" }}
{{ now | date "2006-01-02T15:04:05Z07:00" }}

# String Manipulation
{{ "hello world" | title }}    # Hello World
{{ "  hello  " | trim }}        # hello
{{ "hello" | repeat 3 }}        # hellohellohello
{{ list "a" "b" "c" | join "," }} # a,b,c

# Math
{{ add 1 2 }}     # 3
{{ mul 2 3 }}     # 6
{{ div 10 3 }}    # 3
{{ mod 10 3 }}    # 1
{{ max 1 2 3 }}   # 3
{{ min 1 2 3 }}   # 1

# Type Testing
{{ kindIs "string" .Values.name }}
{{ typeIs "string" .Values.name }}

# URL Encoding
{{ "hello world" | urlquery }}  # hello+world

# Base64
{{ "hello" | b64enc }}  # aGVsbG8=
{{ "aGVsbG8=" | b64dec }}  # hello

# SHA
{{ "hello" | sha256sum }}

# UUID
{{ uuidv4 }}
```

### 4.4 Template Partials

```gotemplate
# templates/_partials.tpl
{{/*
สร้าง Environment Variables สำหรับ Database Connection
*/}}
{{- define "myapp.dbEnvVars" -}}
- name: DATABASE_HOST
  value: {{ .Values.config.database.host | quote }}
- name: DATABASE_PORT
  value: {{ .Values.config.database.port | quote }}
- name: DATABASE_NAME
  value: {{ .Values.config.database.name | quote }}
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: {{ include "myapp.fullname" . }}-db-secret
      key: url
{{- end }}

{{/*
สร้าง Volume Mounts สำหรับ Config Files
*/}}
{{- define "myapp.configVolumeMounts" -}}
- name: config
  mountPath: /app/config
  readOnly: true
{{- if .Values.persistence.enabled }}
- name: data
  mountPath: /app/data
{{- end }}
{{- if .Values.tls.enabled }}
- name: tls-certs
  mountPath: /app/tls
  readOnly: true
{{- end }}
{{- end }}

{{/*
สร้าง Common Annotations
*/}}
{{- define "myapp.podAnnotations" -}}
prometheus.io/scrape: "true"
prometheus.io/port: {{ .Values.metrics.port | quote }}
prometheus.io/path: {{ .Values.metrics.path | quote }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- with .Values.podAnnotations }}
{{ toYaml . }}
{{- end }}
{{- end }}
```

## 5. Production Helm Chart

### 5.1 Production Chart Structure

```
production-app/
├── Chart.yaml
├── Chart.lock
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-production.yaml
├── values.schema.json
├── README.md
├── CHANGELOG.md
├── charts/
│   ├── postgresql-12.x.x.tgz
│   └── redis-17.x.x.tgz
└── templates/
    ├── _helpers.tpl
    ├── _validate.tpl
    ├── _partials.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── rbac.yaml
    ├── hpa.yaml
    ├── pdb.yaml           # PodDisruptionBudget
    ├── configmap.yaml
    ├── secret.yaml
    ├── networkpolicy.yaml
    ├── servicemonitor.yaml
    ├── NOTES.txt
    ├── hooks/
    │   ├── pre-install-migration.yaml
    │   ├── pre-upgrade-migration.yaml
    │   └── post-install-notify.yaml
    └── tests/
        ├── test-connection.yaml
        ├── test-api.yaml
        └── test-database.yaml
```

### 5.2 PodDisruptionBudget Template

```gotemplate
# templates/pdb.yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if .Values.pdb.minAvailable }}
  minAvailable: {{ .Values.pdb.minAvailable }}
  {{- end }}
  {{- if .Values.pdb.maxUnavailable }}
  maxUnavailable: {{ .Values.pdb.maxUnavailable }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
{{- end }}
```

### 5.3 NetworkPolicy Template

```gotemplate
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  {{- if .Values.networkPolicy.ingress.allowExternal }}
  - {}  # Allow all ingress
  {{- else }}
  # Allow from ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: {{ .Values.service.targetPort }}
  
  # Allow from monitoring
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: {{ .Values.metrics.port }}
  
  # Allow from same namespace
  - from:
    - podSelector: {}
  {{- end }}
  
  egress:
  - {}  # Allow all egress (ปรับตามความต้องการ)
{{- end }}
```

### 5.4 ServiceMonitor Template

```gotemplate
# templates/servicemonitor.yaml
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    {{- with .Values.serviceMonitor.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
  namespace: {{ .Values.serviceMonitor.namespace | default .Release.Namespace }}
spec:
  endpoints:
  - port: http
    path: {{ .Values.serviceMonitor.path | default "/metrics" }}
    interval: {{ .Values.serviceMonitor.interval | default "30s" }}
    scrapeTimeout: {{ .Values.serviceMonitor.scrapeTimeout | default "10s" }}
    {{- if .Values.serviceMonitor.relabelings }}
    relabelings:
      {{- toYaml .Values.serviceMonitor.relabelings | nindent 6 }}
    {{- end }}
  namespaceSelector:
    matchNames:
    - {{ .Release.Namespace }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
{{- end }}
```

## 6. Workshop: Production Helm Chart

### Workshop Overview

สร้าง Production-ready Helm Chart สำหรับ E-commerce API ที่มี:
- Pre-install/upgrade Database Migrations
- Post-install Notifications
- Automated Tests
- PostgreSQL Dependency
- Redis Dependency
- NetworkPolicy
- PodDisruptionBudget
- ServiceMonitor

### Step 1: สร้าง Chart Structure

```bash
# สร้าง Chart
helm create ecommerce-api
cd ecommerce-api

# เพิ่ม Files ที่ต้องการ
touch templates/pdb.yaml
touch templates/networkpolicy.yaml
touch templates/servicemonitor.yaml
mkdir -p templates/hooks
touch templates/hooks/pre-install-migration.yaml
touch templates/hooks/post-install-notify.yaml
mkdir -p templates/tests
touch templates/tests/test-api.yaml
touch templates/tests/test-database.yaml
touch values.schema.json
```

### Step 2: Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: ecommerce-api
description: E-commerce REST API
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
- ecommerce
- api
- rest
maintainers:
- name: Platform Team
  email: platform@example.com
dependencies:
- name: postgresql
  version: ">=12.0.0 <13.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
- name: redis
  version: ">=17.0.0 <18.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled
```

### Step 3: Production Values

```yaml
# values.yaml
replicaCount: 3

image:
  repository: registry.example.com/ecommerce-api
  pullPolicy: IfNotPresent
  tag: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
  hosts:
  - host: api.example.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - secretName: api-tls
    hosts:
    - api.example.com

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

pdb:
  enabled: true
  minAvailable: 2

networkPolicy:
  enabled: true
  ingress:
    allowExternal: false

serviceMonitor:
  enabled: true
  namespace: monitoring
  interval: 30s
  labels:
    release: prometheus

metrics:
  port: 8080
  path: /metrics

migrations:
  enabled: true

notifications:
  enabled: true

tests:
  enabled: true

config:
  server:
    port: 8080
  database:
    host: ""
    port: 5432
    name: ecommerce
  redis:
    host: ""
    port: 6379
  features:
    enableSwaggerUI: false
    enableRateLimiting: true
    rateLimitRPS: 100

postgresql:
  enabled: true
  auth:
    database: ecommerce
    username: ecommerce
    existingSecret: ecommerce-db-secret
  primary:
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
    persistence:
      enabled: true
      size: 20Gi

redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: ecommerce-redis-secret
  master:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
    persistence:
      enabled: true
      size: 5Gi

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - ecommerce-api
        topologyKey: kubernetes.io/hostname
```

### Step 4: Pre-upgrade Migration Hook

```gotemplate
# templates/hooks/pre-install-migration.yaml
{{- if .Values.migrations.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "ecommerce-api.fullname" . }}-migration-{{ .Release.Revision }}
  labels:
    {{- include "ecommerce-api.labels" . | nindent 4 }}
    job-type: migration
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    metadata:
      labels:
        {{- include "ecommerce-api.selectorLabels" . | nindent 8 }}
        job-type: migration
    spec:
      restartPolicy: Never
      serviceAccountName: {{ include "ecommerce-api.serviceAccountName" . }}
      
      initContainers:
      - name: wait-for-db
        image: postgres:15-alpine
        command:
        - sh
        - -c
        - |
          until pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do
            echo "Waiting for PostgreSQL..."
            sleep 2
          done
          echo "PostgreSQL is ready!"
        env:
        - name: DB_HOST
          value: {{ .Values.config.database.host | default (printf "%s-postgresql" (include "ecommerce-api.fullname" .)) }}
        - name: DB_PORT
          value: {{ .Values.config.database.port | quote }}
        - name: DB_USER
          value: {{ .Values.postgresql.auth.username }}
      
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        command: ["./migrate", "up", "--all"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ .Values.postgresql.auth.existingSecret }}
              key: connection-string
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
{{- end }}
```

### Step 5: ติดตั้งและทดสอบ

```bash
# Download Dependencies
helm dependency update ecommerce-api/

# Validate Chart
helm lint ecommerce-api/

# Preview Output
helm template ecommerce-api ecommerce-api/ \
  -f ecommerce-api/values.yaml \
  --set postgresql.auth.existingSecret=ecommerce-db-secret \
  --debug

# Dry Run
helm install ecommerce-api ecommerce-api/ \
  --namespace production \
  --create-namespace \
  --dry-run \
  --debug

# ติดตั้งจริง
helm install ecommerce-api ecommerce-api/ \
  --namespace production \
  --create-namespace \
  -f values-production.yaml

# รอ Deployment
kubectl rollout status deployment/ecommerce-api -n production

# รัน Tests
helm test ecommerce-api -n production --logs

# Upgrade
helm upgrade ecommerce-api ecommerce-api/ \
  --namespace production \
  -f values-production.yaml \
  --set image.tag=v1.1.0 \
  --wait \
  --timeout 10m

# ดู History
helm history ecommerce-api -n production

# Rollback ถ้าจำเป็น
helm rollback ecommerce-api 1 -n production --wait
```

## 7. Helm Plugin

### 7.1 ติดตั้ง Popular Plugins

```bash
# helm-diff: ดู Diff ก่อน Upgrade
helm plugin install https://github.com/databus23/helm-diff

# helm-secrets: จัดการ Secrets ด้วย SOPS
helm plugin install https://github.com/jkroepke/helm-secrets

# helm-unittest: Unit Testing สำหรับ Helm Charts
helm plugin install https://github.com/helm-unittest/helm-unittest

# helm-s3: ใช้ S3 เป็น Helm Repository
helm plugin install https://github.com/hypnoglow/helm-s3.git

# ดู Plugins ที่ติดตั้ง
helm plugin list
```

### 7.2 helm-diff

```bash
# ดู Diff ก่อน Upgrade
helm diff upgrade ecommerce-api ecommerce-api/ \
  -f values-production.yaml \
  --set image.tag=v1.1.0

# ดู Diff ระหว่าง Revisions
helm diff revision ecommerce-api 3 4 -n production
```

### 7.3 helm-secrets

```bash
# สร้าง Encrypted Values File
helm secrets enc values-production-secrets.yaml

# ใช้ Encrypted Secrets ใน Install
helm secrets install ecommerce-api ecommerce-api/ \
  -f values-production.yaml \
  -f values-production-secrets.yaml

# ใช้ Encrypted Secrets ใน Upgrade
helm secrets upgrade ecommerce-api ecommerce-api/ \
  -f values-production.yaml \
  -f values-production-secrets.yaml
```

### 7.4 helm-unittest

```yaml
# tests/deployment_test.yaml
suite: Deployment Tests

templates:
- deployment.yaml

tests:
- it: should render deployment
  set:
    image.repository: myimage
    image.tag: v1.0.0
  asserts:
  - isKind:
      of: Deployment
  - equal:
      path: spec.template.spec.containers[0].image
      value: myimage:v1.0.0

- it: should set replicas
  set:
    replicaCount: 5
  asserts:
  - equal:
      path: spec.replicas
      value: 5

- it: should not set replicas when HPA enabled
  set:
    autoscaling.enabled: true
  asserts:
  - notExists:
      path: spec.replicas
```

```bash
# รัน Unit Tests
helm unittest ecommerce-api/

# รัน พร้อม Coverage
helm unittest ecommerce-api/ --coverage
```

## 8. Helm Best Practices

### 8.1 Chart Design

```yaml
# ✓ ดี: ใช้ Conditional Rendering
{{- if .Values.ingress.enabled }}
# Ingress
{{- end }}

# ✓ ดี: กำหนด Default ที่ Sensible
replicaCount: 1

# ✓ ดี: ใช้ required สำหรับ Values ที่บังคับ
image:
  repository: "" # required
  
# ✓ ดี: Comment Values อธิบาย
# replicaCount: จำนวน Replicas ที่ต้องการ
# @default 1
# @minimum 1
```

### 8.2 Security

```yaml
# ✓ ดี: ไม่ใส่ Secrets ใน values.yaml
# ✗ ไม่ดี:
# database:
#   password: mysecretpassword

# ✓ ดี: Reference Secret จากภายนอก
database:
  existingSecret: ""
  existingSecretKey: ""

# ✓ ดี: Set Security Context
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop: ["ALL"]
```

### 8.3 Naming Conventions

```gotemplate
# ✓ ดี: ใช้ include "chart.fullname"
name: {{ include "myapp.fullname" . }}

# ✓ ดี: Labels Consistent
labels:
  {{- include "myapp.labels" . | nindent 4 }}

# ✓ ดี: Selector Labels เหมือนกันทุก Resource
selector:
  matchLabels:
    {{- include "myapp.selectorLabels" . | nindent 6 }}
```

## สรุป

Helm Advanced Features ให้:
1. **Hooks** - Pre/Post Actions สำหรับ DB Migrations, Notifications
2. **Tests** - Automated Testing หลัง Install/Upgrade
3. **Dependencies** - จัดการ Sub-charts (PostgreSQL, Redis)
4. **Schema Validation** - Validate Values ก่อน Install
5. **Library Charts** - Reusable Template Functions
6. **Plugins** - ขยาย Helm ด้วย helm-diff, helm-secrets
7. **Unit Tests** - Test Templates ด้วย helm-unittest

ในบทต่อไปจะเรียนรู้ Kustomize ซึ่งเป็นอีกวิธีในการจัดการ Kubernetes Configuration

## แบบฝึกหัด

1. เพิ่ม Database Migration Hook ในมี Chart
2. สร้าง Helm Tests ที่ Test API Endpoints ทั้งหมด
3. เพิ่ม PostgreSQL Dependency และตั้งค่า Values ให้ถูกต้อง
4. ติดตั้ง helm-diff และทดสอบดู Diff ก่อน Upgrade
5. สร้าง Unit Tests ด้วย helm-unittest สำหรับ Chart ของคุณ
