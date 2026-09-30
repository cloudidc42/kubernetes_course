# Part 75: ArgoCD - GitOps สำหรับ Kubernetes

## บทนำ

ArgoCD เป็น Declarative GitOps Continuous Delivery Tool สำหรับ Kubernetes ที่ใช้หลักการ Pull-based Deployment โดยจะ Monitor Git Repository และทำให้ Cluster State ตรงกับสิ่งที่ระบุไว้ใน Git เสมอ ArgoCD เป็นหนึ่งใน CNCF Graduated Projects และเป็นที่นิยมมากในการทำ GitOps

## 1. ArgoCD Architecture

### 1.1 Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  ArgoCD Namespace                    │   │
│  │                                                      │   │
│  │  ┌─────────────┐   ┌─────────────┐                  │   │
│  │  │  API Server  │   │ Repository  │                  │   │
│  │  │             │   │  Server     │                  │   │
│  │  │  REST API   │   │             │                  │   │
│  │  │  WebUI      │   │  Git Sync   │                  │   │
│  │  │  gRPC       │   │  Helm       │                  │   │
│  │  └─────────────┘   │  Kustomize  │                  │   │
│  │          │          └─────────────┘                  │   │
│  │          │                  │                        │   │
│  │  ┌─────────────┐   ┌─────────────┐                  │   │
│  │  │ Application │   │    Dex      │                  │   │
│  │  │ Controller  │   │  (OIDC)     │                  │   │
│  │  │             │   │             │                  │   │
│  │  │ Reconcile   │   │  SSO/OAuth  │                  │   │
│  │  │ Loop        │   │             │                  │   │
│  │  └─────────────┘   └─────────────┘                  │   │
│  │          │                                           │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │              Redis Cache                    │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           Managed Applications                      │   │
│  │   namespace-1/deployment-a  namespace-2/service-b   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
   Git Repository        Helm Registry
   (Source of Truth)     (Charts)
```

### 1.2 ArgoCD Concepts

**Application:** CRD ที่บอกว่า Application ควรจะ Deploy อะไร จาก Source ไหน ไปยัง Destination ไหน

**Project:** กลุ่มของ Applications ที่มี Policy ร่วมกัน (Source Repos, Destinations, Cluster Permissions)

**Sync:** กระบวนการทำให้ Cluster State ตรงกับ Git State

**Health Status:**
- `Healthy` - Application รันปกติ
- `Degraded` - บางส่วนไม่ทำงาน
- `Progressing` - กำลัง Update
- `Suspended` - หยุดทำงาน
- `Missing` - Resource ไม่มีใน Cluster

**Sync Status:**
- `Synced` - ตรงกับ Git
- `OutOfSync` - ไม่ตรงกับ Git
- `Unknown` - ไม่ทราบ Status

## 2. ติดตั้ง ArgoCD

### 2.1 ติดตั้งด้วย Manifest

```bash
# สร้าง Namespace
kubectl create namespace argocd

# ติดตั้ง ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอ Pods พร้อม
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=120s

# ดู Pods
kubectl get pods -n argocd
```

### 2.2 ติดตั้งด้วย Helm

```bash
# เพิ่ม Helm Repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# สร้าง values.yaml
cat > argocd-values.yaml << 'EOF'
global:
  nodeSelector: {}

# ArgoCD Server
server:
  replicas: 2
  
  # Ingress
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    hosts:
    - argocd.example.com
    tls:
    - secretName: argocd-server-tls
      hosts:
      - argocd.example.com
  
  # Config
  config:
    # ปิด self-signed certificate warning
    url: https://argocd.example.com
    
    # Repositories
    repositories: |
      - url: https://github.com/myorg/gitops-repo
        name: gitops
  
  # RBAC Config
  rbacConfig:
    policy.default: role:readonly
    policy.csv: |
      p, role:admin, applications, *, */*, allow
      p, role:admin, clusters, *, *, allow
      p, role:admin, repositories, *, *, allow
      g, admins, role:admin
      g, developers, role:developer
    scopes: '[groups]'
  
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

# Application Controller
controller:
  replicas: 1
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

# Repo Server
repoServer:
  replicas: 2
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

# Redis
redis:
  resources:
    requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi

# ApplicationSet Controller
applicationSet:
  enabled: true
  replicas: 2

# Notifications
notifications:
  enabled: true

# Dex (SSO)
dex:
  enabled: false  # Enable เมื่อต้องการ SSO
EOF

# ติดตั้ง
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd-values.yaml
```

### 2.3 เข้าถึง ArgoCD UI

```bash
# วิธีที่ 1: Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# วิธีที่ 2: LoadBalancer
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# ดู Initial Password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login ด้วย CLI
argocd login argocd.example.com \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) \
  --insecure

# เปลี่ยน Password
argocd account update-password
```

## 3. Application CRD

### 3.1 Basic Application

```yaml
# myapp-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  # ลบ Application เมื่อลบ Resource นี้
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  # Project ที่ Application อยู่
  project: default
  
  # Source: Git Repository
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: HEAD  # Branch/Tag/SHA
    path: apps/myapp/production
  
  # Destination: Kubernetes Cluster & Namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # Sync Policy
  syncPolicy:
    automated:
      prune: true       # ลบ Resource ที่ไม่มีใน Git
      selfHeal: true    # แก้ไขเมื่อ Cluster State เปลี่ยน
      allowEmpty: false # ไม่อนุญาต Sync ถ้าไม่มี Resource
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Ignore Differences (ไม่ compare บาง fields)
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # ไม่ track replicas (HPA จัดการ)
  - group: ""
    kind: ConfigMap
    name: managed-by-runtime
    jsonPointers:
    - /data
```

### 3.2 Helm Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-helm
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://charts.example.com
    chart: myapp
    targetRevision: 1.2.3
    
    # Helm Values
    helm:
      releaseName: myapp
      valueFiles:
      - values.yaml
      - values-production.yaml
      
      # Override Values
      values: |
        replicaCount: 3
        image:
          tag: v1.2.3
        ingress:
          enabled: true
          host: myapp.example.com
      
      parameters:
      - name: image.tag
        value: v1.2.3
      - name: replicaCount
        value: "3"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 3.3 Kustomize Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-kustomize
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: apps/myapp
    
    # Kustomize Options
    kustomize:
      images:
      - myapp=registry.example.com/myapp:v1.2.3
      
      patches:
      - target:
          kind: Deployment
          name: myapp
        patch: |
          - op: replace
            path: /spec/replicas
            value: 3
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 4. ArgoCD Projects

### 4.1 Project สำหรับ Isolation

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  description: "Team A applications"
  
  # Source Repositories ที่อนุญาต
  sourceRepos:
  - 'https://github.com/myorg/team-a-*'
  - 'https://charts.example.com'
  
  # Destination Clusters และ Namespaces ที่อนุญาต
  destinations:
  - namespace: team-a-*
    server: https://kubernetes.default.svc
  - namespace: monitoring
    server: https://kubernetes.default.svc
  
  # Cluster-wide Resources ที่อนุญาต
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  
  # Namespace-level Resources ที่อนุญาต
  namespaceResourceWhitelist:
  - group: apps
    kind: Deployment
  - group: apps
    kind: StatefulSet
  - group: ''
    kind: Service
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Secret
  - group: networking.k8s.io
    kind: Ingress
  
  # Namespace-level Resources ที่ไม่อนุญาต
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange
  
  # RBAC
  roles:
  - name: developer
    description: "Team A developer"
    policies:
    - p, proj:team-a:developer, applications, get, team-a/*, allow
    - p, proj:team-a:developer, applications, sync, team-a/*, allow
    groups:
    - team-a-developers
  
  - name: admin
    description: "Team A admin"
    policies:
    - p, proj:team-a:admin, applications, *, team-a/*, allow
    groups:
    - team-a-admins
  
  # Orphaned Resources Policy
  orphanedResources:
    warn: true
  
  # Sync Windows (อนุญาตให้ Sync ได้เฉพาะบางช่วงเวลา)
  syncWindows:
  - kind: allow
    schedule: '10 1 * * *'  # ทุกวัน 01:10 UTC
    duration: 1h
    applications:
    - '*'
    namespaces:
    - production
    clusters:
    - in-cluster
  - kind: deny
    schedule: '0 22 * * 5'  # ทุกวันศุกร์ 22:00 UTC
    duration: 14h            # จนถึงวันเสาร์ 12:00 UTC
    namespaces:
    - production
```

## 5. Sync Policies

### 5.1 Manual Sync

```yaml
# ไม่มี syncPolicy.automated
spec:
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 3
```

```bash
# Sync แบบ Manual
argocd app sync myapp

# Sync เฉพาะบาง Resources
argocd app sync myapp --resource deployment:myapp

# Dry-run
argocd app sync myapp --dry-run
```

### 5.2 Automated Sync

```yaml
spec:
  syncPolicy:
    automated:
      prune: true       # ลบ Resource ที่หายไปจาก Git
      selfHeal: true    # แก้ไขเมื่อ Drift เกิดขึ้น
```

### 5.3 Sync Options

```yaml
spec:
  syncPolicy:
    syncOptions:
    # สร้าง Namespace อัตโนมัติ
    - CreateNamespace=true
    
    # ใช้ Server-side Apply แทน kubectl apply
    - ServerSideApply=true
    
    # Prune เฉพาะ Resource ที่ Sync ใหม่ล้มเหลว
    - PrunePropagationPolicy=foreground
    
    # Sync เฉพาะ Resource ที่ OutOfSync
    - ApplyOutOfSyncOnly=true
    
    # แทนที่ Resources ที่มีอยู่แล้ว (แทน merge)
    - Replace=true
    
    # ข้าม Schema Validation
    - Validate=false
    
    # Respect Ignore Differences ระหว่าง Sync
    - RespectIgnoreDifferences=true
```

## 6. Repository Credentials

### 6.1 HTTPS Repository

```yaml
# Secret สำหรับ Private Repository
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/myorg/private-repo.git
  password: github-token
  username: not-used
```

```bash
# เพิ่ม Repository ด้วย CLI
argocd repo add https://github.com/myorg/private-repo.git \
  --username git \
  --password <token>
```

### 6.2 SSH Repository

```bash
# สร้าง SSH Key
ssh-keygen -t ed25519 -C "argocd@example.com" -f argocd-key -N ""

# เพิ่ม Public Key ไปยัง GitHub/GitLab Deploy Keys

# เพิ่ม Private Key ไปยัง ArgoCD
argocd repo add git@github.com:myorg/private-repo.git \
  --ssh-private-key-path argocd-key
```

```yaml
# Secret สำหรับ SSH
apiVersion: v1
kind: Secret
metadata:
  name: repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:myorg/private-repo.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

### 6.3 Helm Repository

```bash
# เพิ่ม Helm Repository
argocd repo add https://charts.example.com \
  --type helm \
  --name my-charts

# OCI Registry
argocd repo add registry.example.com \
  --type helm \
  --name oci-registry \
  --enable-oci \
  --username user \
  --password password
```

## 7. ArgoCD CLI

### 7.1 Application Commands

```bash
# ดู Applications ทั้งหมด
argocd app list

# ดูรายละเอียด Application
argocd app get myapp

# Sync Application
argocd app sync myapp --prune

# รอ Sync เสร็จ
argocd app wait myapp --health

# History
argocd app history myapp

# Rollback
argocd app rollback myapp 3  # rollback ไป revision 3

# Diff ก่อน Sync
argocd app diff myapp

# Delete Application
argocd app delete myapp --cascade  # ลบ resources ด้วย

# สร้าง Application
argocd app create myapp \
  --repo https://github.com/myorg/gitops.git \
  --path apps/myapp/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 7.2 Cluster Commands

```bash
# ดู Clusters ที่ Register
argocd cluster list

# เพิ่ม External Cluster
argocd cluster add my-cluster --in-cluster  # ถ้า kubeconfig ชี้ไปที่ cluster นั้น

# ลบ Cluster
argocd cluster rm https://my-cluster.example.com
```

### 7.3 Project Commands

```bash
# ดู Projects
argocd proj list

# สร้าง Project
argocd proj create team-a \
  --description "Team A Project" \
  --dest https://kubernetes.default.svc,team-a-* \
  --src https://github.com/myorg/team-a-repo.git

# เพิ่ม Role ไปยัง Project
argocd proj role create team-a developer
argocd proj role add-policy team-a developer \
  -a get -o "team-a/*"
```

## 8. Notifications

### 8.1 ตั้งค่า Slack Notifications

```yaml
# argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  
  trigger.on-deployed: |
    - description: Application is synced and healthy
      oncePer: app.status.sync.revision
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  
  trigger.on-health-degraded: |
    - description: Application has degraded
      on-change: true
      send:
      - app-health-degraded
      when: app.status.health.status == 'Degraded'
  
  trigger.on-sync-failed: |
    - description: Application syncing has failed
      on-change: true
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']
  
  template.app-deployed: |
    email:
      subject: Application {{.app.metadata.name}} is now running new version.
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [{
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }, {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          }, {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }]
        }]
  
  template.app-health-degraded: |
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#f4c030",
          "fields": [{
            "title": "Health Status",
            "value": "{{.app.status.health.status}}",
            "short": true
          }, {
            "title": "Message",
            "value": "{{range .app.status.conditions}}{{.message}}{{end}}",
            "short": false
          }]
        }]
  
  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#E96D76",
          "fields": [{
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }, {
            "title": "Error",
            "value": "{{.app.status.operationState.message}}",
            "short": false
          }]
        }]
```

```yaml
# Secret สำหรับ Slack Token
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  slack-token: xoxb-your-slack-token
```

### 8.2 Subscribe Application ไปยัง Notification

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: channel-name
    notifications.argoproj.io/subscribe.on-health-degraded.slack: alerts-channel
    notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts-channel
```

## 9. Workshop: Deploy App ด้วย ArgoCD

### Workshop Overview

ในการ Workshop นี้เราจะ:
1. ติดตั้ง ArgoCD บน Kubernetes
2. สร้าง GitOps Repository
3. สร้าง ArgoCD Application
4. ทดสอบ Auto-Sync
5. ทดสอบ Self-Heal
6. ทดสอบ Rollback

### Step 1: เตรียม GitOps Repository

```bash
# สร้าง GitOps Repository Structure
mkdir gitops-argocd-workshop
cd gitops-argocd-workshop

# สร้าง Directory Structure
mkdir -p apps/guestbook/{base,dev,staging,production}

# Base Deployment
cat > apps/guestbook/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: guestbook
        image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
EOF

cat > apps/guestbook/base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: guestbook
spec:
  selector:
    app: guestbook
  ports:
  - port: 80
    targetPort: 80
EOF

cat > apps/guestbook/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

commonLabels:
  managed-by: argocd
EOF

# Dev Overlay
cat > apps/guestbook/dev/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: guestbook-dev

resources:
- ../base

patches:
- target:
    kind: Deployment
    name: guestbook
  patch: |
    - op: replace
      path: /spec/replicas
      value: 1
EOF

# Production Overlay
cat > apps/guestbook/production/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: guestbook-production

resources:
- ../base

patches:
- target:
    kind: Deployment
    name: guestbook
  patch: |
    - op: replace
      path: /spec/replicas
      value: 3
    - op: add
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "100m"
    - op: add
      path: /spec/template/spec/containers/0/resources/limits/cpu
      value: "500m"
EOF

# Commit
git init
git add .
git commit -m "Initial GitOps repository"
git remote add origin https://github.com/myorg/gitops-argocd-workshop.git
git push -u origin main
```

### Step 2: ติดตั้ง ArgoCD

```bash
# ติดตั้ง ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอ Pods
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# ดู Initial Password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo "ArgoCD Password: $ARGOCD_PASSWORD"

# Login
argocd login localhost:8080 \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure
```

### Step 3: เพิ่ม Repository

```bash
# เพิ่ม Public Repository
argocd repo add https://github.com/myorg/gitops-argocd-workshop.git

# ตรวจสอบ
argocd repo list
```

### Step 4: สร้าง Application

```bash
# สร้าง Dev Application
argocd app create guestbook-dev \
  --repo https://github.com/myorg/gitops-argocd-workshop.git \
  --path apps/guestbook/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook-dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true

# ดู Status
argocd app get guestbook-dev
```

```yaml
# หรือสร้างด้วย YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-argocd-workshop.git
    targetRevision: HEAD
    path: apps/guestbook/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Step 5: ทดสอบ Auto-Sync

```bash
# เปลี่ยน Image Version ใน Git
cd gitops-argocd-workshop

# แก้ไข base deployment
sed -i 's|ks-guestbook-demo:0.2|ks-guestbook-demo:0.3|g' \
  apps/guestbook/base/deployment.yaml

git add .
git commit -m "Update guestbook to v0.3"
git push

# ArgoCD จะ Detect การเปลี่ยนแปลงและ Sync อัตโนมัติ
# ดู Status
argocd app get guestbook-dev --watch

# ดู Pods
kubectl get pods -n guestbook-dev -w
```

### Step 6: ทดสอบ Self-Heal

```bash
# แก้ไข Deployment โดยตรง (ไม่ผ่าน Git)
kubectl scale deployment guestbook --replicas=5 -n guestbook-dev

# ArgoCD จะ Detect Drift และ Self-Heal กลับไปที่ 1 replica
# ดู Events
kubectl get events -n guestbook-dev --sort-by='.lastTimestamp'

# ดู Status ใน ArgoCD
argocd app get guestbook-dev
```

### Step 7: ทดสอบ Rollback

```bash
# ดู History
argocd app history guestbook-dev

# Rollback ไป Previous Version
argocd app rollback guestbook-dev 1  # revision number จาก history

# รอ Rollback เสร็จ
argocd app wait guestbook-dev --health

# ตรวจสอบ
kubectl get pods -n guestbook-dev
```

### Step 8: สร้าง Production Application

```bash
# สร้าง Production Application (Manual Sync)
argocd app create guestbook-production \
  --repo https://github.com/myorg/gitops-argocd-workshop.git \
  --path apps/guestbook/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook-production \
  --sync-option CreateNamespace=true

# Sync แบบ Manual
argocd app sync guestbook-production

# ดู Status
argocd app get guestbook-production
kubectl get pods -n guestbook-production
```

## 10. ArgoCD Metrics และ Monitoring

### 10.1 Prometheus Metrics

```yaml
# ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: monitoring
  labels:
    release: prometheus
spec:
  endpoints:
  - port: metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - argocd
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  namespace: monitoring
  labels:
    release: prometheus
spec:
  endpoints:
  - port: metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - argocd
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
```

### 10.2 Grafana Dashboard

```bash
# Import ArgoCD Dashboard
# Grafana Dashboard ID: 14584 (ArgoCD)
# หรือ Import จาก https://github.com/argoproj/argo-cd/blob/master/examples/dashboard.json
```

## 11. Security Considerations

### 11.1 RBAC Configuration

```yaml
# argocd-rbac-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  
  policy.csv: |
    # Admin Role
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    p, role:admin, projects, *, *, allow
    p, role:admin, accounts, *, *, allow
    p, role:admin, gpgkeys, *, *, allow
    
    # Developer Role
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:developer, applications, action/*, */*, allow
    
    # Read-only Role
    p, role:readonly, applications, get, */*, allow
    
    # Group Mappings
    g, team-admins, role:admin
    g, team-developers, role:developer
    g, team-viewers, role:readonly
```

### 11.2 Secret Management

```yaml
# ใช้ Sealed Secrets
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: production
spec:
  encryptedData:
    password: AgB2...  # Encrypted value
  template:
    metadata:
      name: my-secret
      namespace: production
```

## สรุป

ArgoCD ให้:
1. **GitOps** - Git เป็น Single Source of Truth
2. **Pull-based** - ปลอดภัยกว่า Push-based
3. **Auto-Sync** - Detect และ Sync การเปลี่ยนแปลงอัตโนมัติ
4. **Self-Heal** - แก้ไข Drift อัตโนมัติ
5. **Multi-cluster** - จัดการหลาย Cluster จากที่เดียว
6. **RBAC** - Access Control ละเอียด
7. **UI** - Web UI ที่ใช้งานง่าย

ในบทต่อไปจะเรียนรู้ ArgoCD Advanced Features ได้แก่ ApplicationSets, App of Apps Pattern และ Multi-cluster Deployment

## แบบฝึกหัด

1. ติดตั้ง ArgoCD บน Kubernetes ด้วย Helm
2. สร้าง GitOps Repository และ ArgoCD Application
3. ทดสอบ Auto-Sync โดยการ Push Code ไปยัง Git
4. ทดสอบ Self-Heal โดยการแก้ไข Deployment โดยตรง
5. ตั้งค่า Slack Notifications สำหรับ Deployment Events
