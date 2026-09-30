# Part 80: Kustomize - Multi-environment Configuration Management

## บทนำ

Kustomize เป็น Tool สำหรับจัดการ Kubernetes Configuration ที่รวมอยู่ใน kubectl (ตั้งแต่ v1.14) ต่างจาก Helm ที่ใช้ Template, Kustomize ใช้ Overlay Pattern โดยไม่แก้ไข Base YAML ทำให้ Configuration ง่ายต่อการเข้าใจและดูแลรักษา

## 1. Kustomize Philosophy

### 1.1 Template-free Configuration

```
Helm Approach (Templates):
  deployment.yaml (template)  ──> values.yaml ──> rendered.yaml
  
Kustomize Approach (Overlays):
  base/deployment.yaml ──────> kustomization.yaml ──> final.yaml
  overlays/prod/patch.yaml ──┘
```

**Kustomize Advantages:**
1. ไม่มี Template Language ที่ต้องเรียนรู้
2. Base YAML เป็น Valid Kubernetes YAML ทดสอบได้โดยตรง
3. Overlay ชัดเจน เห็นว่าเปลี่ยนอะไรในแต่ละ Environment
4. รวมอยู่ใน kubectl ไม่ต้องติดตั้งเพิ่ม
5. GitOps-friendly

### 1.2 Kustomize vs Helm

| ด้าน | Kustomize | Helm |
|------|-----------|------|
| Templating | ไม่มี (Patches) | Go Templates |
| Learning Curve | ต่ำ | ปานกลาง |
| Package Sharing | ยาก | ง่าย (Helm Registry) |
| Complexity | เหมาะสำหรับ Simple-Medium | เหมาะสำหรับ Complex |
| GitOps | ดีมาก | ดี |
| Hooks | ไม่มี Built-in | มี |
| Testing | ไม่มี Built-in | มี (helm test) |

## 2. kustomization.yaml

### 2.1 Structure ของ kustomization.yaml

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Metadata
namePrefix: dev-         # เพิ่ม prefix ทุก Resource name
nameSuffix: -v2          # เพิ่ม suffix
namespace: development   # กำหนด namespace

# Resources
resources:
- deployment.yaml
- service.yaml
- ../base              # Reference Base หรือ Overlay อื่น

# Images
images:
- name: myapp
  newName: registry.example.com/myapp
  newTag: v1.2.3

# Config Generator
configMapGenerator:
- name: app-config
  files:
  - config.yaml
  literals:
  - ENV=production
  - LOG_LEVEL=warn

# Secret Generator
secretGenerator:
- name: app-secret
  files:
  - secret.yaml
  literals:
  - password=mysecretpassword
  envs:
  - .env

# Patches
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
  target:
    kind: Deployment
    name: myapp

# Strategic Merge Patches
patchesStrategicMerge:
- patch-replicas.yaml

# JSON Patches
patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: myapp
  path: patch.yaml

# Common Labels
labels:
- pairs:
    environment: production
    team: platform
  includeSelectors: true

# Common Annotations
commonAnnotations:
  managed-by: kustomize
  version: v1.2.3

# Vars (deprecated ใน v5, ใช้ replacements แทน)
vars:
- name: SERVICE_NAME
  objref:
    kind: Service
    name: myapp
    apiVersion: v1

# Replacements (v5+)
replacements:
- source:
    kind: Service
    name: myapp
    fieldPath: metadata.name
  targets:
  - select:
      kind: Ingress
      name: myapp
    fieldPaths:
    - spec.rules.0.http.paths.0.backend.service.name

# Components
components:
- ../../components/monitoring

# Transformers
transformers:
- transformer.yaml
```

## 3. Base Configuration

### 3.1 สร้าง Base

```bash
# Directory Structure
app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        ├── kustomization.yaml
        └── patch-replicas.yaml
```

### 3.2 Base Files

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp  # Image name ที่ใช้อ้างอิงใน kustomization.yaml
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        envFrom:
        - configMapRef:
            name: app-config

---
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080

---
# base/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- ingress.yaml

configMapGenerator:
- name: app-config
  literals:
  - LOG_FORMAT=json

images:
- name: myapp
  newName: registry.example.com/myapp
  newTag: latest
```

## 4. Overlays

### 4.1 Development Overlay

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# อ้างอิง Base
resources:
- ../../base

# Override Namespace
namespace: development

# Override Image Tag
images:
- name: myapp
  newName: registry.example.com/myapp
  newTag: develop

# Override ConfigMap
configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=debug
  - ENVIRONMENT=development

# Patches
patches:
# ลด Resources สำหรับ Dev
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 1
      template:
        spec:
          containers:
          - name: myapp
            resources:
              requests:
                cpu: 50m
                memory: 64Mi
              limits:
                cpu: 200m
                memory: 128Mi

# Override Ingress Host
- patch: |-
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: myapp
    spec:
      rules:
      - host: dev.myapp.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80

# Common Labels
labels:
- pairs:
    environment: development
  includeSelectors: false
```

### 4.2 Staging Overlay

```yaml
# overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namespace: staging

images:
- name: myapp
  newName: registry.example.com/myapp
  newTag: v1.2.3-rc1

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=info
  - ENVIRONMENT=staging
  - FEATURE_FLAGS=new_ui:false,dark_mode:true

patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 2
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "200m"
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "256Mi"

- target:
    kind: Ingress
    name: myapp
  patch: |-
    - op: replace
      path: /spec/rules/0/host
      value: staging.myapp.example.com

labels:
- pairs:
    environment: staging
    tier: staging
```

### 4.3 Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base
- hpa.yaml
- pdb.yaml
- networkpolicy.yaml

namespace: production

images:
- name: myapp
  newName: registry.example.com/myapp
  newTag: v1.2.3

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=warn
  - ENVIRONMENT=production
  - FEATURE_FLAGS=new_ui:false,dark_mode:true

# Merge ด้วย Strategic Merge Patch
patchesStrategicMerge:
- patch-production.yaml

# JSON Patches
patches:
- target:
    kind: Ingress
    name: myapp
  patch: |-
    - op: replace
      path: /spec/rules/0/host
      value: myapp.example.com
    - op: add
      path: /metadata/annotations/cert-manager.io~1cluster-issuer
      value: letsencrypt-prod
    - op: add
      path: /spec/tls
      value:
        - hosts:
          - myapp.example.com
          secretName: myapp-tls

labels:
- pairs:
    environment: production
    tier: production
  includeSelectors: false

commonAnnotations:
  managed-by: kustomize
  team: platform
```

```yaml
# overlays/production/patch-production.yaml
# Strategic Merge Patch
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 1Gi
      # เพิ่ม Anti-affinity
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - myapp
            topologyKey: kubernetes.io/hostname
      # Rolling Update Strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```yaml
# overlays/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```yaml
# overlays/production/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: myapp

---
# overlays/production/networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - {}
```

## 5. Patches

### 5.1 Strategic Merge Patch

```yaml
# strategic-merge.yaml
# เหมือนกับ kubectl apply - merge แบบ Smart
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: myapp
        # เพิ่ม Environment Variable
        env:
        - name: NEW_VAR
          value: "new-value"
        # เพิ่ม Volume
      volumes:
      - name: new-volume
        emptyDir: {}
```

### 5.2 JSON Patch (RFC 6902)

```yaml
# json-patch.yaml
- op: replace
  path: /spec/replicas
  value: 5

- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: NEW_ENV
    value: "new-value"

- op: remove
  path: /spec/template/spec/containers/0/livenessProbe

- op: copy
  from: /spec/template/spec/containers/0/resources/requests
  path: /spec/template/spec/containers/0/resources/limits

- op: move
  from: /metadata/labels/old-label
  path: /metadata/labels/new-label

- op: test
  path: /spec/replicas
  value: 1  # ถ้าไม่ตรงจะ Error
```

### 5.3 Inline Patches

```yaml
# kustomization.yaml
patches:
# Inline JSON Patch
- target:
    kind: Deployment
    name: myapp
    namespace: production
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: add
      path: /spec/template/metadata/annotations/prometheus.io~1scrape
      value: "true"

# Inline Strategic Merge Patch
- target:
    kind: Service
    name: myapp
  patch: |-
    apiVersion: v1
    kind: Service
    metadata:
      name: myapp
    spec:
      type: LoadBalancer
      loadBalancerIP: 1.2.3.4
```

### 5.4 Patch ด้วย Regex

```yaml
# patch หลาย Resources พร้อมกัน
patches:
- target:
    kind: Deployment
    labelSelector: "app=myapp"
  patch: |-
    - op: add
      path: /metadata/annotations/checksum
      value: "abc123"

# patch โดยใช้ Annotation Selector
- target:
    kind: ConfigMap
    annotationSelector: "managed-by=kustomize"
  patch: |-
    - op: add
      path: /metadata/labels/updated
      value: "true"
```

## 6. Generators

### 6.1 ConfigMapGenerator

```yaml
# kustomization.yaml
configMapGenerator:
# สร้างจาก Literal Values
- name: app-config
  namespace: production
  literals:
  - LOG_LEVEL=warn
  - MAX_CONNECTIONS=100
  - TIMEOUT=30s

# สร้างจาก Files
- name: nginx-config
  files:
  - nginx.conf
  - configs/upstream.conf

# สร้างจาก .env File
- name: app-env
  envs:
  - .env
  - .env.production

# Behavior: create (default), replace, merge
- name: existing-config
  behavior: merge
  literals:
  - NEW_KEY=new-value

# Disable Hash Suffix
- name: config-no-hash
  literals:
  - key=value
  options:
    disableNameSuffixHash: true
    labels:
      managed-by: kustomize
```

### 6.2 SecretGenerator

```yaml
# kustomization.yaml
secretGenerator:
# จาก Literals
- name: db-credentials
  literals:
  - username=admin
  - password=mysecretpassword

# จาก Files
- name: tls-secret
  type: kubernetes.io/tls
  files:
  - tls.crt
  - tls.key

# จาก .env
- name: app-secrets
  envs:
  - secrets.env

# กำหนด Type
- name: docker-registry
  type: kubernetes.io/dockerconfigjson
  files:
  - .dockerconfigjson
```

### 6.3 Image Transformer

```yaml
# kustomization.yaml
images:
# เปลี่ยน Image Repository
- name: myapp
  newName: registry.example.com/myapp

# เปลี่ยน Image Tag
- name: myapp
  newTag: v1.2.3

# เปลี่ยนทั้ง Repository และ Tag
- name: nginx
  newName: bitnami/nginx
  newTag: "1.25"

# เปลี่ยน Digest
- name: myapp
  digest: sha256:abc123

# ใช้ Wildcard
- name: "*/myapp"  # Match ทุก Registry
  newTag: v1.2.3
```

## 7. Components

### 7.1 Kustomize Components

Components คือ Reusable Kustomize Configurations ที่สามารถนำไปใช้ใน Overlays ต่างๆ ได้

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- servicemonitor.yaml
- prometheusrule.yaml

configMapGenerator:
- name: prometheus-config
  literals:
  - scrape_interval=30s
```

```yaml
# components/tls/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

patches:
- target:
    kind: Ingress
  patch: |-
    - op: add
      path: /metadata/annotations/cert-manager.io~1cluster-issuer
      value: letsencrypt-prod
```

```yaml
# overlays/production/kustomization.yaml
resources:
- ../../base

# ใช้ Components
components:
- ../../components/monitoring
- ../../components/tls
- ../../components/networkpolicy
```

## 8. Replacements

### 8.1 Replacements (v5+)

```yaml
# แทนที่ค่าจาก Resource หนึ่งไปยังอีก Resource
replacements:
# เอา Service name ไปใส่ใน Ingress
- source:
    kind: Service
    name: myapp
    fieldPath: metadata.name
  targets:
  - select:
      kind: Ingress
      name: myapp
    fieldPaths:
    - spec.rules.[host=myapp.example.com].http.paths.[path=/].backend.service.name

# เอา ConfigMap data ไปใส่ใน Deployment env
- source:
    kind: ConfigMap
    name: app-config
    fieldPath: data.DATABASE_HOST
  targets:
  - select:
      kind: Deployment
      name: myapp
    fieldPaths:
    - spec.template.spec.containers.[name=myapp].env.[name=DATABASE_HOST].value
```

## 9. Kustomize CLI

### 9.1 Commands

```bash
# Build (Generate Final YAML)
kubectl kustomize ./overlays/production

# Apply Directly
kubectl apply -k ./overlays/production

# Build และ Save
kubectl kustomize ./overlays/production > production-manifests.yaml
kubectl apply -f production-manifests.yaml

# Diff
kubectl diff -k ./overlays/production

# ใช้ kustomize CLI โดยตรง
kustomize build ./overlays/production
kustomize build ./overlays/production | kubectl apply -f -

# Edit kustomization.yaml
kustomize edit set image myapp=registry.example.com/myapp:v1.2.3
kustomize edit add resource newresource.yaml
kustomize edit add configmap new-config --from-literal=key=value
```

### 9.2 ตรวจสอบ Output

```bash
# ดู Output ก่อน Apply
kubectl kustomize overlays/production

# ดู Diff
kubectl diff -k overlays/production

# Dry Run
kubectl apply -k overlays/production --dry-run=client

# Apply
kubectl apply -k overlays/production

# Delete
kubectl delete -k overlays/production
```

## 10. Workshop: Multi-environment Config

### Workshop Overview

สร้าง Configuration Management ด้วย Kustomize สำหรับ:
1. Web Application (React + Go API)
2. 3 Environments: dev, staging, production
3. Shared Components: Monitoring, NetworkPolicy, TLS

### Step 1: สร้าง Directory Structure

```bash
mkdir kustomize-workshop
cd kustomize-workshop

# สร้าง Structure
mkdir -p base/{frontend,backend,database}
mkdir -p overlays/{dev,staging,production}
mkdir -p components/{monitoring,tls,networkpolicy,autoscaling}

# ดู Structure
find . -type d
```

### Step 2: Base Manifests

```yaml
# base/backend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
    tier: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      serviceAccountName: backend
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: backend
        image: backend  # Reference name สำหรับ kustomize images
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: PORT
          value: "8080"
        envFrom:
        - configMapRef:
            name: backend-config
        - secretRef:
            name: backend-secret
            optional: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}

---
# base/backend/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080

---
# base/backend/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
automountServiceAccountToken: false

---
# base/backend/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- serviceaccount.yaml

configMapGenerator:
- name: backend-config
  literals:
  - LOG_FORMAT=json
  - LOG_LEVEL=info
  - ENVIRONMENT=local

images:
- name: backend
  newName: registry.example.com/backend
  newTag: latest
```

```yaml
# base/frontend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
    tier: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: frontend
        ports:
        - containerPort: 3000
          name: http
        envFrom:
        - configMapRef:
            name: frontend-config
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5

---
# base/frontend/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

configMapGenerator:
- name: frontend-config
  literals:
  - REACT_APP_API_URL=/api
  - REACT_APP_ENV=local

images:
- name: frontend
  newName: registry.example.com/frontend
  newTag: latest
```

```yaml
# base/kustomization.yaml (Root Base)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- frontend/
- backend/

commonAnnotations:
  managed-by: kustomize
```

### Step 3: Components

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- servicemonitor-backend.yaml
- servicemonitor-frontend.yaml

patches:
- target:
    kind: Deployment
  patch: |-
    - op: add
      path: /spec/template/metadata/annotations
      value: {}
    - op: add
      path: /spec/template/metadata/annotations/prometheus.io~1scrape
      value: "true"
    - op: add
      path: /spec/template/metadata/annotations/prometheus.io~1port
      value: "8080"
```

```yaml
# components/monitoring/servicemonitor-backend.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: backend
  labels:
    release: prometheus
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    port: http
  namespaceSelector:
    any: true
  selector:
    matchLabels:
      app: backend
```

```yaml
# components/tls/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- ingress.yaml

patches:
- target:
    kind: Ingress
  patch: |-
    - op: add
      path: /metadata/annotations/cert-manager.io~1cluster-issuer
      value: letsencrypt-prod
    - op: add
      path: /metadata/annotations/nginx.ingress.kubernetes.io~1ssl-redirect
      value: "true"
```

```yaml
# components/autoscaling/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
- hpa-backend.yaml
- hpa-frontend.yaml
```

```yaml
# components/autoscaling/hpa-backend.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Step 4: Development Overlay

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namespace: development

images:
- name: backend
  newName: registry.example.com/backend
  newTag: develop
- name: frontend
  newName: registry.example.com/frontend
  newTag: develop

configMapGenerator:
- name: backend-config
  behavior: merge
  literals:
  - LOG_LEVEL=debug
  - ENVIRONMENT=development
  - DEBUG=true

- name: frontend-config
  behavior: merge
  literals:
  - REACT_APP_API_URL=http://backend.development.svc.cluster.local
  - REACT_APP_ENV=development

patches:
# ลด Resources สำหรับ Dev
- target:
    kind: Deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "50m"
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "64Mi"

labels:
- pairs:
    environment: development
    cost-center: engineering
```

### Step 5: Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base
- ingress.yaml

namespace: production

# ใช้ Components
components:
- ../../components/monitoring
- ../../components/tls
- ../../components/autoscaling

images:
- name: backend
  newName: registry.example.com/backend
  newTag: v1.5.0
- name: frontend
  newName: registry.example.com/frontend
  newTag: v2.1.0

configMapGenerator:
- name: backend-config
  behavior: merge
  literals:
  - LOG_LEVEL=warn
  - ENVIRONMENT=production

- name: frontend-config
  behavior: merge
  literals:
  - REACT_APP_API_URL=https://api.example.com
  - REACT_APP_ENV=production

patchesStrategicMerge:
- patch-production-resources.yaml

labels:
- pairs:
    environment: production
    tier: production
  includeSelectors: false

commonAnnotations:
  deployment-environment: production
```

```yaml
# overlays/production/patch-production-resources.yaml
# Strategic Merge Patch สำหรับ Backend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: backend
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 1Gi
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - backend
            topologyKey: kubernetes.io/hostname
```

```yaml
# overlays/production/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
```

### Step 6: ทดสอบ Workshop

```bash
# Build Dev
kubectl kustomize overlays/dev

# Build Production
kubectl kustomize overlays/production

# Validate Output
kubectl kustomize overlays/production | kubectl apply --dry-run=client -f -

# Apply Dev
kubectl apply -k overlays/dev

# Apply Production (ใน Production Cluster)
kubectl apply -k overlays/production

# ดู Resources
kubectl get all -n development
kubectl get all -n production

# Update Image Version
cd overlays/production
kustomize edit set image backend=registry.example.com/backend:v1.6.0
git add kustomization.yaml
git commit -m "Update backend to v1.6.0"
git push

# Diff
kubectl diff -k overlays/production
```

### Step 7: Automated Image Updates ด้วย Script

```bash
#!/bin/bash
# update-image.sh

set -e

ENVIRONMENT=${1:-production}
SERVICE=${2:-backend}
VERSION=${3:-latest}

echo "Updating $SERVICE to $VERSION in $ENVIRONMENT"

cd overlays/$ENVIRONMENT

# Update image
kustomize edit set image $SERVICE=registry.example.com/$SERVICE:$VERSION

# Commit
git add kustomization.yaml
git commit -m "chore: update $SERVICE to $VERSION in $ENVIRONMENT"
git push

echo "Updated successfully!"
echo "ArgoCD/Flux will sync automatically"
```

```bash
# ใช้งาน
./update-image.sh production backend v1.6.0
./update-image.sh staging frontend v2.2.0-rc1
```

## 11. Kustomize กับ GitOps

### 11.1 GitOps Repository Structure

```
gitops-repo/
├── base/
│   └── myapp/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── components/
│   ├── monitoring/
│   ├── tls/
│   └── autoscaling/
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       ├── kustomization.yaml
│       └── patches/
└── clusters/
    ├── dev-cluster/
    │   └── apps/
    │       └── kustomization.yaml
    └── prod-cluster/
        └── apps/
            └── kustomization.yaml
```

### 11.2 ArgoCD กับ Kustomize

```yaml
# ArgoCD Application ที่ใช้ Kustomize
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: HEAD
    path: overlays/production
    
    # Kustomize Options
    kustomize:
      # Override images
      images:
      - myapp=registry.example.com/myapp:v1.2.3
      
      # Override namespace
      namePrefix: prod-
      
      # Version
      version: v5.0.0
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 12. Best Practices

### 12.1 ดี vs ไม่ดี

```yaml
# ✓ ดี: Base เป็น Valid Kubernetes YAML
# overlays ทำ Patch ไม่ได้ใส่ทุกอย่าง

# ✗ ไม่ดี: ใส่ Production-specific Config ใน Base
# base/deployment.yaml:
spec:
  replicas: 5  # Production specific ไม่ควรอยู่ใน base!

# ✓ ดี:
# base/deployment.yaml:
spec:
  replicas: 1  # Minimal default

# overlays/production/kustomization.yaml:
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

### 12.2 Secret Management

```bash
# ✗ ไม่ดี: เก็บ Secrets ใน Git แบบ Plaintext
secretGenerator:
- name: db-secret
  literals:
  - password=mysecretpassword  # อย่าทำ!

# ✓ ดี: ใช้ External Secret Management
# ใช้ ExternalSecrets Operator
# ใช้ Sealed Secrets
# ใช้ Vault + Vault Agent

# ✓ ดี: ใช้ SOPS สำหรับ Encrypt Files ก่อน Push
sops --encrypt --age age1xxx secrets.env > secrets.enc.env
# แล้วค่อย include ใน secretGenerator
```

### 12.3 Organization

```bash
# ✓ ดี: แยก Components ที่ Reusable
components/
├── monitoring/  # ServiceMonitor, Grafana Dashboards
├── security/    # NetworkPolicy, PodSecurityPolicy
└── scaling/     # HPA, VPA

# ✓ ดี: ใช้ Base ที่ Minimal
# Base ควรมีแค่ที่จำเป็น
# Overlays เพิ่มตามที่แต่ละ Environment ต้องการ
```

## สรุป

Kustomize ให้:
1. **Template-free** - ไม่ต้องเรียนรู้ Template Language
2. **Overlay Pattern** - Base + Patch แยกชัดเจน
3. **Native kubectl** - รวมอยู่ใน kubectl แล้ว
4. **GitOps-friendly** - Configuration เป็น Plain YAML
5. **Components** - Reusable Configuration Pieces
6. **Generators** - สร้าง ConfigMaps/Secrets อัตโนมัติ
7. **Multi-environment** - จัดการ Dev/Staging/Production ง่าย

ขอแสดงความยินดี! คุณได้เรียนรู้ CI/CD & GitOps สำหรับ Kubernetes ครบทั้ง 10 บทแล้ว ตั้งแต่ CI/CD Concepts, Jenkins, GitHub Actions, GitLab CI/CD, ArgoCD, Flux CD, Helm, และ Kustomize

## แบบฝึกหัด

1. สร้าง Kustomize Structure สำหรับ 3 Environments (dev/staging/production)
2. สร้าง Component สำหรับ Monitoring ที่ใช้ได้กับหลาย Applications
3. ตั้งค่า ArgoCD ให้ Deploy จาก Kustomize Overlays
4. เขียน Script สำหรับ Auto-update Image Version ใน kustomization.yaml
5. รวม Kustomize กับ Flux CD เพื่อทำ GitOps Pipeline
