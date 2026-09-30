# Part 76: ArgoCD Advanced - ApplicationSets, App of Apps, และ Multi-cluster

## บทนำ

ในบทที่แล้วเราเรียนรู้ ArgoCD พื้นฐาน ในบทนี้จะเจาะลึก Features ขั้นสูงที่จำเป็นสำหรับ Production Environment ได้แก่ ApplicationSets, App of Apps Pattern, ArgoCD Notifications แบบละเอียด และการจัดการ Multi-cluster Deployment

## 1. ApplicationSet Controller

### 1.1 ApplicationSet คืออะไร

ApplicationSet เป็น Controller ที่ Generate ArgoCD Applications อัตโนมัติจาก Template และ Generators ต่างๆ ช่วยจัดการ Applications จำนวนมากอย่างเป็นระบบ

```
ApplicationSet (Template)
         │
         ├── Generator 1: List
         ├── Generator 2: Git
         ├── Generator 3: Cluster
         └── Generator 4: Matrix
                  │
                  ▼
         สร้าง Applications อัตโนมัติ
         ├── app-dev
         ├── app-staging
         └── app-production
```

### 1.2 List Generator

```yaml
# ใช้ List Generator เพื่อสร้าง Application หลายตัว
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-environments
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: dev
        namespace: myapp-dev
        replicas: "1"
        url: https://dev.example.com
      - env: staging
        namespace: myapp-staging
        replicas: "2"
        url: https://staging.example.com
      - env: production
        namespace: myapp-production
        replicas: "5"
        url: https://example.com
  
  template:
    metadata:
      name: 'myapp-{{env}}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo.git
        targetRevision: HEAD
        path: apps/myapp/{{env}}
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### 1.3 Git Generator

```yaml
# สร้าง Application สำหรับทุก Directory ใน Git
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-dir-generator
  namespace: argocd
spec:
  generators:
  # Directory Generator: สร้าง App สำหรับทุก Directory
  - git:
      repoURL: https://github.com/myorg/gitops-repo.git
      revision: HEAD
      directories:
      - path: apps/*
      - path: apps/deprecated/*
        exclude: true  # ยกเว้น directory นี้
  
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: false  # ไม่ prune เพื่อป้องกัน accident
          selfHeal: true

---
# File Generator: สร้าง App จาก JSON/YAML files
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-file-generator
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/myorg/gitops-repo.git
      revision: HEAD
      files:
      - path: "apps/**/config.json"
  
  template:
    metadata:
      name: '{{appName}}'
    spec:
      project: '{{project}}'
      source:
        repoURL: '{{repoURL}}'
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: '{{destinationServer}}'
        namespace: '{{namespace}}'
```

```json
// apps/myapp/config.json
{
  "appName": "myapp",
  "project": "team-a",
  "repoURL": "https://github.com/myorg/myapp.git",
  "path": "deploy/production",
  "destinationServer": "https://kubernetes.default.svc",
  "namespace": "myapp-production"
}
```

### 1.4 Cluster Generator

```yaml
# สร้าง Application สำหรับทุก Cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
          region: ap-southeast-1
      values:
        revision: v1.2.3
  
  template:
    metadata:
      name: 'myapp-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo.git
        targetRevision: '{{values.revision}}'
        path: apps/myapp/production
      destination:
        server: '{{server}}'
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 1.5 Matrix Generator

```yaml
# ใช้ Matrix Generator เพื่อ Cross-product ของ Generators
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix-generator
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      # Generator 1: Services
      - list:
          elements:
          - service: frontend
            port: "3000"
          - service: backend
            port: "8080"
          - service: worker
            port: "0"
      
      # Generator 2: Environments
      - list:
          elements:
          - env: dev
            namespace: myapp-dev
          - env: production
            namespace: myapp-prod
  
  template:
    metadata:
      name: '{{service}}-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo.git
        targetRevision: HEAD
        path: 'apps/{{service}}/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
```

### 1.6 SCM Provider Generator

```yaml
# สร้าง Application สำหรับทุก Repository ใน GitHub Organization
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: github-org-apps
  namespace: argocd
spec:
  generators:
  - scmProvider:
      github:
        organization: myorg
        # ดึง Repository ที่มี file นี้เท่านั้น
        filters:
        - repositoryMatch: '^service-'
        - pathsExist:
          - deploy/production
      cloneProtocol: https
  
  template:
    metadata:
      name: '{{repository}}'
    spec:
      project: default
      source:
        repoURL: '{{url}}'
        targetRevision: '{{branch}}'
        path: deploy/production
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{repository}}'
```

## 2. App of Apps Pattern

### 2.1 ความคิด App of Apps

```
Root Application (app-of-apps)
         │
         ├── monitoring/
         │   ├── prometheus.yaml
         │   └── grafana.yaml
         │
         ├── logging/
         │   ├── elasticsearch.yaml
         │   └── kibana.yaml
         │
         └── apps/
             ├── frontend.yaml
             ├── backend.yaml
             └── worker.yaml
```

### 2.2 สร้าง App of Apps

```yaml
# bootstrap/root-app.yaml (Root Application)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: HEAD
    path: bootstrap
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# bootstrap/monitoring.yaml (Child Application: Monitoring)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: HEAD
    path: infrastructure/monitoring
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true

---
# bootstrap/apps.yaml (Child Application: Apps)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-apps
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: HEAD
    path: apps/production
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# apps/production/frontend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/frontend.git
    targetRevision: v2.1.0
    path: deploy/production
    helm:
      values: |
        image:
          tag: v2.1.0
        replicas: 3
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# apps/production/backend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/backend.git
    targetRevision: v1.5.2
    path: deploy/production
    helm:
      values: |
        image:
          tag: v1.5.2
        replicas: 5
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### 2.3 Directory Structure

```
gitops-repo/
├── bootstrap/              # Root Apps (App of Apps)
│   ├── root-app.yaml       # Root Application
│   ├── monitoring.yaml     # Monitoring Application
│   ├── logging.yaml        # Logging Application
│   └── apps.yaml           # Production Apps Application
│
├── infrastructure/
│   ├── monitoring/
│   │   ├── kustomization.yaml
│   │   ├── prometheus.yaml
│   │   └── grafana.yaml
│   └── logging/
│       ├── kustomization.yaml
│       ├── elasticsearch.yaml
│       └── kibana.yaml
│
└── apps/
    ├── production/         # Production App Manifests
    │   ├── frontend.yaml   # ArgoCD Application CRD
    │   ├── backend.yaml
    │   └── worker.yaml
    └── staging/
        ├── frontend.yaml
        ├── backend.yaml
        └── worker.yaml
```

## 3. Multi-cluster Deployment

### 3.1 Register External Cluster

```bash
# วิธีที่ 1: ใช้ kubeconfig
kubectl config use-context production-cluster
argocd cluster add production-cluster \
  --name production-cluster

# วิธีที่ 2: ใช้ Secret
kubectl create -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: production-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: production-cluster
  server: https://production-k8s.example.com
  config: |
    {
      "bearerToken": "<token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64-ca-data>"
      }
    }
EOF

# ดู Clusters
argocd cluster list
```

### 3.2 Multi-cluster Repository Structure

```
gitops-repo/
├── clusters/
│   ├── dev-cluster/
│   │   ├── apps/
│   │   │   ├── frontend.yaml
│   │   │   └── backend.yaml
│   │   └── infrastructure/
│   │       └── monitoring.yaml
│   ├── staging-cluster/
│   │   ├── apps/
│   │   └── infrastructure/
│   └── production-cluster/
│       ├── apps/
│       └── infrastructure/
│
└── base/
    ├── frontend/
    │   ├── base/
    │   ├── dev/
    │   ├── staging/
    │   └── production/
    └── backend/
        └── ...
```

### 3.3 ApplicationSet สำหรับ Multi-cluster

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-frontend
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          region: ap-southeast-1
  
  template:
    metadata:
      name: 'frontend-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo.git
        targetRevision: HEAD
        path: 'base/frontend/{{metadata.labels.environment}}'
      destination:
        server: '{{server}}'
        namespace: frontend
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### 3.4 Cluster Labels สำหรับ Multi-cluster

```bash
# เพิ่ม Labels ไปยัง Cluster
kubectl label secret production-cluster \
  -n argocd \
  environment=production \
  region=ap-southeast-1 \
  tier=prod

kubectl label secret staging-cluster \
  -n argocd \
  environment=staging \
  region=ap-southeast-1 \
  tier=staging

# ดู Clusters พร้อม Labels
argocd cluster list --output yaml | grep -A 10 labels
```

## 4. ArgoCD Advanced Notifications

### 4.1 Microsoft Teams Notification

```yaml
# argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.teams: |
    webhookUrls:
      general: $teams-webhook-url
  
  template.teams-notification: |
    teams:
      themeColor: |
        {{if eq .app.status.health.status "Healthy"}}00FF00
        {{else if eq .app.status.health.status "Degraded"}}FF0000
        {{else}}FFFF00{{end}}
      summary: "ArgoCD: {{.app.metadata.name}}"
      sections:
      - activityTitle: "Application {{.app.metadata.name}} {{.app.status.health.status}}"
        activitySubtitle: "{{.context.argocdUrl}}"
        facts:
        - name: Sync Status
          value: "{{.app.status.sync.status}}"
        - name: Health Status
          value: "{{.app.status.health.status}}"
        - name: Deployed Version
          value: "{{.app.status.sync.revision}}"
```

### 4.2 Email Notification

```yaml
  service.email.gmail: |
    host: smtp.gmail.com
    port: 465
    from: argocd@example.com
    username: argocd@example.com
    password: $email-password
  
  template.email-deployment: |
    email:
      subject: "ArgoCD: {{.app.metadata.name}} deployment"
      body: |
        Application: {{.app.metadata.name}}
        Project: {{.app.spec.project}}
        Sync Status: {{.app.status.sync.status}}
        Health Status: {{.app.status.health.status}}
        
        {{if eq .app.status.health.status "Healthy"}}
        Deployment successful!
        {{else}}
        Deployment has issues. Please investigate.
        {{end}}
        
        ArgoCD URL: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
```

### 4.3 Custom Notification Triggers

```yaml
  trigger.on-major-version-change: |
    - description: Application has been updated to a major version
      send:
      - major-version-notification
      when: |
        app.status.operationState.phase in ['Succeeded'] and
        app.status.health.status == 'Healthy' and
        app.spec.source.targetRevision =~ '^v[0-9]+\.0\.0$'
  
  trigger.on-deployment-to-production: |
    - description: Application deployed to production
      oncePer: app.status.sync.revision
      send:
      - production-deployment-notification
      when: |
        app.status.operationState.phase in ['Succeeded'] and
        app.status.health.status == 'Healthy' and
        app.spec.destination.namespace == 'production'
```

## 5. Image Updater

### 5.1 ติดตั้ง ArgoCD Image Updater

```bash
# ติดตั้ง
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# ตรวจสอบ
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater
```

### 5.2 ตั้งค่า Image Updater

```yaml
# Application ที่ใช้ Image Updater
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    # ตั้งค่า Image Updater
    argocd-image-updater.argoproj.io/image-list: myapp=registry.example.com/myapp
    
    # Update Strategy: latest, semver, digest, name
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    
    # Version Constraint
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    
    # Write Back (แก้ไข Git)
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    
    # Helm Parameter
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.tag
spec:
  # ...
```

### 5.3 Registry Credentials

```yaml
# Secret สำหรับ Registry Access
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: argocd
  labels:
    app.kubernetes.io/component: image-updater
    app.kubernetes.io/part-of: argocd-image-updater
type: Opaque
stringData:
  credentials: |
    credentials:
    - registry: registry.example.com
      username: myuser
      password: mypassword
      type: BasicAuthentication
```

## 6. Health Checks

### 6.1 Custom Health Check

```yaml
# argocd-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Custom Health Check สำหรับ CRD
  resource.customizations.health.certmanager.io_Certificate: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "False" then
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
          if condition.type == "Ready" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = condition.message
            return hs
          end
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for certificate"
    return hs
  
  # Custom Health Check สำหรับ HPA
  resource.customizations.health.autoscaling_HorizontalPodAutoscaler: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.currentReplicas ~= nil then
        if obj.status.currentReplicas == obj.status.desiredReplicas then
          hs.status = "Healthy"
          hs.message = string.format("%d/%d replicas ready", 
            obj.status.readyReplicas or 0, 
            obj.status.desiredReplicas or 0)
          return hs
        end
        hs.status = "Progressing"
        hs.message = string.format("Scaling from %d to %d", 
          obj.status.currentReplicas, 
          obj.status.desiredReplicas)
        return hs
      end
    end
    hs.status = "Unknown"
    return hs
```

## 7. Resource Hooks

### 7.1 Pre-sync Job

```yaml
# Pre-sync Hook: รัน Database Migration ก่อน Deploy
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: wait-for-db
        image: busybox
        command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 1; done']
      containers:
      - name: migrate
        image: registry.example.com/myapp:latest
        command: ['./migrate', 'up']
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

### 7.2 Post-sync Job

```yaml
# Post-sync Hook: ส่ง Notification หลัง Deploy
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-deployment
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: notify
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-type: application/json' \
            -d '{"text": "Deployment completed successfully!"}'
        env:
        - name: SLACK_WEBHOOK
          valueFrom:
            secretKeyRef:
              name: slack-secret
              key: webhook-url
```

### 7.3 Sync Fail Hook

```yaml
# SyncFail Hook: Alert เมื่อ Sync ล้มเหลว
apiVersion: batch/v1
kind: Job
metadata:
  name: alert-sync-failure
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookFailed
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: alert
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          curl -X POST $PAGERDUTY_WEBHOOK \
            -H 'Content-type: application/json' \
            -d "{\"routing_key\": \"$ROUTING_KEY\", \"event_action\": \"trigger\", \"payload\": {\"summary\": \"ArgoCD Sync Failed\", \"severity\": \"critical\"}}"
```

## 8. Workshop: Production GitOps Setup

### Workshop Overview

ในการ Workshop นี้เราจะสร้าง Production-ready GitOps Setup:
1. App of Apps Structure
2. Multi-environment Deployment
3. Image Updater
4. Notifications
5. Rollback Procedures

### Step 1: สร้าง GitOps Repository Structure

```bash
mkdir production-gitops
cd production-gitops

# สร้าง Directory Structure
mkdir -p {bootstrap,infrastructure,apps/{dev,staging,production},base/{frontend,backend}/{base,dev,staging,production}}

# Root Application
cat > bootstrap/root.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/production-gitops.git
    targetRevision: HEAD
    path: bootstrap
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false  # ระวัง! ไม่ prune Apps ที่ไม่มีใน Git
      selfHeal: true
EOF

# Infrastructure Applications
cat > bootstrap/infrastructure.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infrastructure
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/production-gitops.git
    targetRevision: HEAD
    path: infrastructure
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Dev Applications
cat > bootstrap/apps-dev.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-dev
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/production-gitops.git
    targetRevision: HEAD
    path: apps/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Production Applications
cat > bootstrap/apps-production.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-production
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/production-gitops.git
    targetRevision: HEAD
    path: apps/production
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false  # ระวัง Production!
      selfHeal: false
EOF
```

### Step 2: สร้าง Base Applications

```yaml
# base/frontend/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
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
        image: registry.example.com/frontend:latest
        ports:
        - containerPort: 3000
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
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
```

```yaml
# base/frontend/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../base

patches:
- target:
    kind: Deployment
    name: frontend
  patch: |
    - op: replace
      path: /spec/replicas
      value: 5
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "200m"
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/cpu
      value: "1000m"
    - op: add
      path: /spec/strategy
      value:
        type: RollingUpdate
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 0

images:
- name: registry.example.com/frontend
  newTag: "REPLACE_WITH_CI_SHA"
```

### Step 3: สร้าง ArgoCD Applications ใน apps/

```yaml
# apps/production/frontend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-production
  namespace: argocd
  annotations:
    # Image Updater Configuration
    argocd-image-updater.argoproj.io/image-list: frontend=registry.example.com/frontend
    argocd-image-updater.argoproj.io/frontend.update-strategy: semver
    argocd-image-updater.argoproj.io/frontend.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    
    # Notification
    notifications.argoproj.io/subscribe.on-deployed.slack: production-deployments
    notifications.argoproj.io/subscribe.on-health-degraded.slack: production-alerts
    notifications.argoproj.io/subscribe.on-sync-failed.slack: production-alerts
  
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/production-gitops.git
    targetRevision: HEAD
    path: base/frontend/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    # Production: Manual sync + selfHeal
    syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 5m
  
  # ไม่ track replicas (HPA จัดการ)
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

### Step 4: Deploy Bootstrap

```bash
# Install ArgoCD (ถ้ายังไม่มี)
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Push GitOps repo
git add .
git commit -m "Initial production GitOps setup"
git push origin main

# Apply Root Application
kubectl apply -f bootstrap/root.yaml

# ดู Status
kubectl get applications -n argocd
argocd app list
```

### Step 5: ตั้งค่า RBAC สำหรับ Production

```yaml
# AppProject สำหรับ Production
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: "Production Applications"
  
  sourceRepos:
  - 'https://github.com/myorg/*'
  - 'https://charts.example.com'
  
  destinations:
  - namespace: 'production'
    server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: 'apps'
    kind: StatefulSet
  - group: ''
    kind: Service
  - group: 'networking.k8s.io'
    kind: Ingress
  - group: 'autoscaling'
    kind: HorizontalPodAutoscaler
  
  roles:
  - name: production-admin
    policies:
    - p, proj:production:production-admin, applications, *, production/*, allow
    groups:
    - platform-team
  
  - name: production-readonly
    policies:
    - p, proj:production:production-readonly, applications, get, production/*, allow
    groups:
    - developers
  
  # Allow Sync เฉพาะ Business Hours (Mon-Fri 08:00-17:00 ICT = 01:00-10:00 UTC)
  syncWindows:
  - kind: allow
    schedule: '0 1 * * 1-5'
    duration: 9h
    applications:
    - '*'
    namespaces:
    - production
    manualSync: true
  
  # ปิด Sync วันศุกร์ 17:00 ถึงจันทร์ 08:00 (Maintenance Window)
  - kind: deny
    schedule: '0 10 * * 5'
    duration: 63h
    namespaces:
    - production
    manualSync: false  # ยังอนุญาต Manual sync
```

### Step 6: Production Rollback Procedure

```bash
# ดู History ของ Production Application
argocd app history frontend-production

# ดู Diff กับ Previous Version
argocd app diff frontend-production --revision 5

# Rollback to previous version
argocd app rollback frontend-production 5

# รอ Rollback เสร็จ
argocd app wait frontend-production --health

# ตรวจสอบ
kubectl get pods -n production -l app=frontend

# ถ้าต้องการ Rollback ผ่าน Git
# 1. Revert Git commit
git revert HEAD --no-edit
git push origin main

# 2. ArgoCD จะ Auto-sync (ถ้าเปิด automated)
# หรือ Manual sync
argocd app sync frontend-production
```

## 9. Disaster Recovery

### 9.1 Export Applications

```bash
# Export ทุก Applications
argocd app list -o yaml > all-apps-backup.yaml

# Export ทีละ App
for app in $(argocd app list -o name); do
  argocd app get $app -o yaml > backup/$app.yaml
done
```

### 9.2 Backup ArgoCD Configuration

```bash
# Backup ArgoCD Secrets และ ConfigMaps
kubectl get secret -n argocd -o yaml > argocd-secrets-backup.yaml
kubectl get configmap -n argocd -o yaml > argocd-configmaps-backup.yaml

# Backup Applications
kubectl get applications -n argocd -o yaml > argocd-applications-backup.yaml

# ใช้ Velero สำหรับ Full Backup
velero backup create argocd-backup \
  --include-namespaces argocd \
  --storage-location default
```

### 9.3 Restore Procedure

```bash
# Restore จาก Backup
kubectl apply -f argocd-secrets-backup.yaml
kubectl apply -f argocd-configmaps-backup.yaml
kubectl apply -f argocd-applications-backup.yaml

# Verify
argocd app list
argocd app sync --all  # Sync ทุก Apps
```

## สรุป

ArgoCD Advanced Features ให้:
1. **ApplicationSets** - จัดการ Applications จำนวนมากอัตโนมัติ
2. **App of Apps** - Hierarchical Application Management
3. **Multi-cluster** - Deploy ไปหลาย Cluster จากที่เดียว
4. **Image Updater** - Update Image Version อัตโนมัติ
5. **Notifications** - แจ้งเตือน Deployment Events
6. **Hooks** - Pre/Post Sync Actions
7. **RBAC** - Fine-grained Access Control
8. **Sync Windows** - ควบคุมเวลา Deployment

ในบทต่อไปจะเรียนรู้ Flux CD ซึ่งเป็น GitOps Tool อีกตัวที่ได้รับความนิยม

## แบบฝึกหัด

1. สร้าง ApplicationSet ที่ Deploy Application ไปยังทุก Environment
2. ออกแบบ App of Apps Structure สำหรับ Organization ของคุณ
3. ตั้งค่า Image Updater สำหรับ Auto-update Image Tag
4. ตั้งค่า Slack Notifications สำหรับ Production Events
5. สร้าง Sync Window สำหรับ Production (Business Hours only)
