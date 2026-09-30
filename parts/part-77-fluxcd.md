# Part 77: Flux CD - GitOps ด้วย Flux

## บทนำ

Flux CD (หรือ Flux v2) เป็น GitOps Tool สำหรับ Kubernetes ที่พัฒนาโดย Weaveworks และเป็น CNCF Graduated Project เช่นเดียวกับ ArgoCD Flux ใช้แนวทาง GitOps แบบ Pull-based และมีความยืดหยุ่นสูงในการทำงานกับ Helm, Kustomize และ Git Repositories

## 1. Flux Architecture

### 1.1 Flux Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  flux-system Namespace               │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │            Source Controller                   │  │   │
│  │  │  - GitRepository                               │  │   │
│  │  │  - HelmRepository                              │  │   │
│  │  │  - HelmChart                                   │  │   │
│  │  │  - OCIRepository                               │  │   │
│  │  │  - Bucket                                      │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │          Kustomize Controller                  │  │   │
│  │  │  - Kustomization                               │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │            Helm Controller                     │  │   │
│  │  │  - HelmRelease                                 │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │       Image Automation Controller              │  │   │
│  │  │  - ImageRepository                             │  │   │
│  │  │  - ImagePolicy                                 │  │   │
│  │  │  - ImageUpdateAutomation                       │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │       Notification Controller                  │  │   │
│  │  │  - Provider                                    │  │   │
│  │  │  - Alert                                       │  │   │
│  │  │  - Receiver                                    │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Flux vs ArgoCD

| Feature | Flux | ArgoCD |
|---------|------|--------|
| UI | ไม่มี (ใช้ CLI/Grafana) | มี Web UI สวยงาม |
| Multi-cluster | ดีมาก | ดี |
| Helm Support | ดีมาก | ดี |
| Kustomize | ดีมาก | ดี |
| Image Automation | Built-in | ต้องติดตั้งเพิ่ม |
| RBAC | Kubernetes RBAC | ArgoCD RBAC |
| Notification | Alert CRD | Notifications Plugin |
| ความง่ายเริ่มต้น | ยากกว่าเล็กน้อย | ง่ายกว่า (มี UI) |

## 2. ติดตั้ง Flux

### 2.1 ติดตั้ง Flux CLI

```bash
# macOS/Linux
curl -s https://fluxcd.io/install.sh | sudo bash

# Homebrew
brew install fluxcd/tap/flux

# ตรวจสอบ
flux version

# ตรวจสอบ Prerequisites
flux check --pre
```

### 2.2 Bootstrap Flux

```bash
# เตรียม GitHub Token
export GITHUB_TOKEN=ghp_xxxxxxxxxxxx
export GITHUB_USER=myusername
export GITHUB_REPO=gitops-flux

# Bootstrap บน GitHub
flux bootstrap github \
  --token-auth \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# Bootstrap บน GitLab
flux bootstrap gitlab \
  --token-auth \
  --owner=mygroup \
  --repository=gitops-flux \
  --branch=main \
  --path=clusters/my-cluster

# Bootstrap ด้วย SSH Key
flux bootstrap github \
  --owner=myorg \
  --repository=gitops-repo \
  --branch=main \
  --path=clusters/production \
  --ssh-key-algorithm=ed25519
```

### 2.3 สิ่งที่ Bootstrap ทำ

```bash
# หลัง Bootstrap จะมี:
# 1. Repository สร้างขึ้น (ถ้าไม่มี)
# 2. Deploy Key เพิ่มเข้า Repository
# 3. Flux Components ติดตั้งใน cluster
kubectl get all -n flux-system

# Directory Structure ใน Repository
clusters/
└── my-cluster/
    └── flux-system/
        ├── gotk-components.yaml  # Flux CRDs and Controllers
        ├── gotk-sync.yaml        # GitRepository + Kustomization
        └── kustomization.yaml
```

## 3. GitRepository Source

### 3.1 GitRepository CRD

```yaml
# GitRepository: ระบุ Git Repository ที่ Flux จะ Monitor
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-repo
  namespace: flux-system
spec:
  # Git URL
  url: https://github.com/myorg/gitops-repo.git
  
  # Polling Interval
  interval: 1m
  
  # Branch/Tag/SHA
  ref:
    branch: main
    # tag: v1.2.3
    # semver: ">=1.0.0"
    # commit: abc123
  
  # ไม่ดึง Directory นี้
  ignore: |
    *.md
    docs/
    tests/
  
  # Verify GPG Signature
  # verification:
  #   mode: head
  #   secretRef:
  #     name: gpg-public-keys

---
# GitRepository สำหรับ SSH
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: private-repo
  namespace: flux-system
spec:
  url: ssh://git@github.com/myorg/private-repo.git
  interval: 5m
  ref:
    branch: main
  secretRef:
    name: ssh-credentials

---
# SSH Credentials
apiVersion: v1
kind: Secret
metadata:
  name: ssh-credentials
  namespace: flux-system
type: Opaque
stringData:
  identity: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
  known_hosts: |
    github.com ecdsa-sha2-nistp256 ...
```

### 3.2 ดู GitRepository Status

```bash
# ดู Status
kubectl get gitrepositories -n flux-system
flux get sources git

# ดูรายละเอียด
kubectl describe gitrepository gitops-repo -n flux-system

# ดู Revision
kubectl get gitrepository gitops-repo -n flux-system \
  -o jsonpath='{.status.artifact.revision}'
```

## 4. Kustomization CRD

### 4.1 Kustomization ของ Flux (ต่างจาก kustomization.yaml ของ Kustomize)

```yaml
# Flux Kustomization: Deploy ไปยัง Cluster
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  # Git Source
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  
  # Path ใน Repository
  path: ./apps/production
  
  # Polling Interval
  interval: 10m
  
  # Retry Interval เมื่อ Fail
  retryInterval: 1m
  
  # Prune Resources ที่ไม่มีใน Git
  prune: true
  
  # Timeout
  timeout: 2m
  
  # Target Namespace (override namespace ใน manifests)
  targetNamespace: production
  
  # สร้าง Namespace ถ้าไม่มี
  # ต้องใช้ serviceAccountName ที่มี permission สร้าง Namespace
  
  # Wait for Resources
  wait: true
  
  # Decrypt Secrets (SOPS)
  decryption:
    provider: sops
    secretRef:
      name: sops-age
  
  # Health Checks
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: frontend
    namespace: production
  - apiVersion: apps/v1
    kind: Deployment
    name: backend
    namespace: production
  
  # Force Apply (แก้ไข Immutable Fields)
  force: false
  
  # Substitution Variables
  postBuild:
    substitute:
      ENV: production
      REGISTRY: registry.example.com
    substituteFrom:
    - kind: ConfigMap
      name: cluster-vars
    - kind: Secret
      name: cluster-secrets
      optional: true
```

### 4.2 Dependencies ระหว่าง Kustomizations

```yaml
# Infrastructure ต้องติดตั้งก่อน Apps
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  path: ./infrastructure
  interval: 10m
  prune: true

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  path: ./apps
  interval: 10m
  prune: true
  # รอ infrastructure ก่อน
  dependsOn:
  - name: infrastructure
```

## 5. HelmRelease

### 5.1 HelmRepository

```yaml
# HelmRepository: ระบุ Helm Repository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  url: https://charts.bitnami.com/bitnami
  interval: 30m

---
# OCI Helm Repository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: oci-charts
  namespace: flux-system
spec:
  type: oci
  url: oci://registry.example.com/charts
  interval: 10m
  secretRef:
    name: oci-credentials
```

### 5.2 HelmRelease

```yaml
# HelmRelease: Deploy Helm Chart
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: redis
  namespace: production
spec:
  # Install/Upgrade Interval
  interval: 10m
  
  # Helm Chart Reference
  chart:
    spec:
      chart: redis
      version: ">=17.0.0 <18.0.0"  # Semver constraint
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 1m
  
  # Values
  values:
    architecture: standalone
    auth:
      enabled: true
      existingSecret: redis-secret
      existingSecretPasswordKey: password
    master:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
    metrics:
      enabled: true
      serviceMonitor:
        enabled: true
  
  # Values from ConfigMap/Secret
  valuesFrom:
  - kind: ConfigMap
    name: redis-values
    valuesKey: values.yaml
  - kind: Secret
    name: redis-secret-values
    optional: true
  
  # Install Timeout
  install:
    timeout: 10m
    remediation:
      retries: 3
  
  # Upgrade Configuration
  upgrade:
    timeout: 10m
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true
  
  # Rollback Configuration
  rollback:
    timeout: 5m
    cleanupOnFail: true
    force: false
    recreate: false
  
  # Uninstall Configuration
  uninstall:
    keepHistory: false
  
  # Dependency Check
  dependsOn:
  - name: cert-manager
    namespace: cert-manager
```

### 5.3 HelmRelease สำหรับ Application

```yaml
# HelmRelease สำหรับ Application ของเรา
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  
  chart:
    spec:
      chart: myapp
      version: ">=1.0.0 <2.0.0"
      sourceRef:
        kind: HelmRepository
        name: internal-charts
        namespace: flux-system
  
  values:
    image:
      repository: registry.example.com/myapp
      tag: v1.2.3
    
    replicaCount: 3
    
    ingress:
      enabled: true
      className: nginx
      host: myapp.example.com
      tls:
        enabled: true
        secretName: myapp-tls
    
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
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70
    
    serviceMonitor:
      enabled: true
      namespace: monitoring
  
  # Post-renderer: ใช้ Kustomize patches บน Helm output
  postRenderers:
  - kustomize:
      patches:
      - target:
          kind: Deployment
          name: myapp
        patch: |
          - op: add
            path: /spec/template/metadata/annotations/prometheus.io~1scrape
            value: "true"
```

## 6. Image Automation

### 6.1 ImageRepository

```yaml
# Monitor Docker Registry สำหรับ Image Updates
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: registry.example.com/myapp
  interval: 5m
  
  # Credentials
  secretRef:
    name: registry-credentials
  
  # ดึงเฉพาะ Tags ที่ match Pattern (เร็วกว่า)
  exclusionList:
  - "^.*-dev$"
  - "^.*-test$"
  - "^latest$"
```

### 6.2 ImagePolicy

```yaml
# กำหนด Policy สำหรับการเลือก Image Tag
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
    namespace: flux-system
  
  # Semver Policy: เลือก Latest Stable Version
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"

---
# Alphabetical Policy: เลือก Lexicographically Latest
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp-main
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    alphabetical:
      order: asc
  filterTags:
    pattern: '^main-[a-f0-9]+-(?P<ts>[0-9]+)$'
    extract: '$ts'

---
# Numerical Policy: เลือก Latest Build Number
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp-build
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    numerical:
      order: asc
  filterTags:
    pattern: '^build-([0-9]+)$'
    extract: '$1'
```

### 6.3 ImageUpdateAutomation

```yaml
# Automatically update image tags in Git
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  
  # Source Repository
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  
  # Git Configuration
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@example.com
        name: Flux
      messageTemplate: |
        [ci skip] Update image {{range .Updated.Images -}}{{println .}} {{- end}}
    push:
      branch: main
      # หรือ Push ไปยัง Branch ใหม่แทน
      # refspec: refs/heads/image-updates
  
  # Update Policy
  update:
    strategy: Setters
    path: ./apps  # Path ที่จะ Update
```

### 6.4 Marker ใน Deployment

```yaml
# deployment.yaml ที่มี Image Marker
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        # Flux Image Updater Marker
        image: registry.example.com/myapp:v1.0.0 # {"$imagepolicy": "flux-system:myapp"}
```

## 7. Notifications

### 7.1 Provider

```yaml
# Slack Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: deployments
  address: https://hooks.slack.com/services/xxx/yyy/zzz
  # หรือใช้ Secret
  secretRef:
    name: slack-webhook

---
# Microsoft Teams Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: teams
  namespace: flux-system
spec:
  type: msteams
  address: https://outlook.office.com/webhook/...

---
# GitHub Commit Status Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: github-status
  namespace: flux-system
spec:
  type: github
  address: https://github.com/myorg/myapp
  secretRef:
    name: github-token

---
# PagerDuty Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: pagerduty
  namespace: flux-system
spec:
  type: pagerduty
  address: https://events.pagerduty.com/v2/enqueue
  secretRef:
    name: pagerduty-key
```

### 7.2 Alert

```yaml
# Alert สำหรับ Application Failures
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-call
  namespace: flux-system
spec:
  providerRef:
    name: pagerduty
  
  # เฉพาะ Error Events
  eventSeverity: error
  
  # Resources ที่ Monitor
  eventSources:
  - kind: HelmRelease
    namespace: production
    name: '*'  # ทุก HelmRelease
  - kind: Kustomization
    namespace: flux-system
    name: apps
  
  # Suspend Alert ชั่วคราว
  suspend: false

---
# Alert สำหรับ Deployments ทั้งหมด
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: deployments
  namespace: flux-system
spec:
  providerRef:
    name: slack
  
  eventSeverity: info
  
  eventSources:
  - kind: HelmRelease
    namespace: '*'
  - kind: Kustomization
    namespace: flux-system
  
  inclusions:
  - 'HelmRelease/.*/.*'
  
  exclusions:
  - 'HelmRelease/.*/.*\[no changes\]'
```

### 7.3 Receiver (Webhook)

```yaml
# Receiver: รับ Webhook จาก GitHub/GitLab เพื่อ Trigger Reconciliation ทันที
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: github-receiver
  namespace: flux-system
spec:
  type: github
  events:
  - "ping"
  - "push"
  secretRef:
    name: webhook-token
  resources:
  - apiVersion: source.toolkit.fluxcd.io/v1
    kind: GitRepository
    name: gitops-repo
    namespace: flux-system
```

```bash
# ดู Receiver URL
kubectl get receiver github-receiver -n flux-system \
  -o jsonpath='{.status.webhookPath}'
# /hook/sha256-xxxx

# Receiver URL: https://flux.example.com/hook/sha256-xxxx
# เพิ่ม URL นี้ใน GitHub Webhook Settings
```

## 8. Multi-tenant Setup

### 8.1 Tenant Isolation

```yaml
# Platform Team Kustomization (รัน as cluster-admin)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tenants
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  path: ./tenants
  prune: true
  serviceAccountName: kustomize-controller  # cluster-admin

---
# Team A's Kustomization (รัน as team-a ServiceAccount)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-a-apps
  namespace: team-a
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: team-a-repo
    namespace: team-a
  path: ./apps
  prune: true
  serviceAccountName: flux-reconciler  # Limited permissions
  targetNamespace: team-a
```

```yaml
# RBAC สำหรับ Team A's Flux
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flux-reconciler
  namespace: team-a

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: flux-reconciler
  namespace: team-a
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flux-reconciler
  namespace: team-a
subjects:
- kind: ServiceAccount
  name: flux-reconciler
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: flux-reconciler
```

## 9. SOPS Secrets Encryption

### 9.1 Setup Age Key

```bash
# ติดตั้ง age
brew install age   # macOS
# หรือ
curl -Lo age.tar.gz https://github.com/FiloSottile/age/releases/latest/download/age-v1.1.1-linux-amd64.tar.gz
tar xf age.tar.gz && sudo mv age/age /usr/local/bin/

# สร้าง Age Key Pair
age-keygen -o age.agekey

# ดู Public Key
cat age.agekey | grep "# public key:"
# public key: age1xxxxxxxxxxxx

# เพิ่ม Private Key ไปยัง Flux
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey
```

### 9.2 ตั้งค่า SOPS

```yaml
# .sops.yaml (ในรากของ Repository)
creation_rules:
- path_regex: .*/secrets/.*\.yaml$
  age: age1xxxxxxxxxxxx

- path_regex: clusters/production/.*\.yaml$
  age: >-
    age1xxxxxxxxxxxx,
    age1yyyyyyyyyy
```

### 9.3 Encrypt Secrets

```bash
# ติดตั้ง SOPS
brew install sops  # macOS
# หรือ download binary

# สร้าง Secret ที่จะ Encrypt
cat > secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
stringData:
  db-password: mysecretpassword
  db-host: db.example.com
EOF

# Encrypt ด้วย SOPS
sops --encrypt \
  --age age1xxxxxxxxxxxx \
  --encrypted-regex '^(data|stringData)$' \
  secret.yaml > secret.enc.yaml

# Decrypt เพื่อดู (ต้องมี Private Key)
sops --decrypt secret.enc.yaml
```

### 9.4 Kustomization ที่ใช้ SOPS

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  path: ./apps
  interval: 10m
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age  # Age Private Key
```

## 10. Workshop: GitOps ด้วย Flux

### Workshop Overview

ในการ Workshop นี้เราจะ:
1. Bootstrap Flux บน Kubernetes
2. Deploy Application ด้วย Kustomization
3. Deploy Helm Chart ด้วย HelmRelease
4. ตั้งค่า Image Automation
5. ตั้งค่า Notifications

### Step 1: Bootstrap Flux

```bash
# ตั้งค่า Environment
export GITHUB_TOKEN=ghp_xxxxxxxxxxxx
export GITHUB_USER=myusername

# Bootstrap
flux bootstrap github \
  --token-auth \
  --owner=$GITHUB_USER \
  --repository=flux-workshop \
  --branch=main \
  --path=clusters/workshop \
  --personal

# ตรวจสอบ
kubectl get pods -n flux-system
flux check
```

### Step 2: Clone Repository และสร้าง Structure

```bash
# Clone repository ที่ bootstrap สร้าง
git clone https://github.com/$GITHUB_USER/flux-workshop
cd flux-workshop

# สร้าง Directory Structure
mkdir -p {apps,infrastructure,sources}

# สร้าง App Directory
mkdir -p apps/podinfo/{base,production}
```

### Step 3: สร้าง App Manifests

```yaml
# apps/podinfo/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:6.5.0
        ports:
        - containerPort: 9898
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
        env:
        - name: PODINFO_UI_COLOR
          value: "#34577c"
```

```yaml
# apps/podinfo/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
spec:
  selector:
    app: podinfo
  ports:
  - port: 80
    targetPort: 9898
```

```yaml
# apps/podinfo/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
```

```yaml
# apps/podinfo/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: workshop

resources:
- ../base

patches:
- target:
    kind: Deployment
    name: podinfo
  patch: |
    - op: replace
      path: /spec/replicas
      value: 2
    - op: replace
      path: /spec/template/spec/containers/0/env/0/value
      value: "#4CAF50"
```

### Step 4: สร้าง Flux Kustomization

```yaml
# clusters/workshop/podinfo-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/podinfo/production
  prune: true
  targetNamespace: workshop
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: podinfo
    namespace: workshop
  syncOptions:
    - CreateNamespace=true
```

### Step 5: Deploy Helm Release (Redis)

```yaml
# sources/bitnami.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  url: https://charts.bitnami.com/bitnami
  interval: 30m
```

```yaml
# apps/redis/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: redis
  namespace: workshop
spec:
  interval: 10m
  chart:
    spec:
      chart: redis
      version: ">=17.0.0 <18.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    architecture: standalone
    auth:
      enabled: false
    master:
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
```

```yaml
# clusters/workshop/redis-helmrelease.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: redis
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/redis
  prune: true
  targetNamespace: workshop
  syncOptions:
    - CreateNamespace=true
```

### Step 6: Push และตรวจสอบ

```bash
# Push changes
git add .
git commit -m "Add podinfo and redis"
git push origin main

# ดู Flux Status
flux get all -n flux-system
flux get kustomizations
flux get helmreleases -A
flux get sources git

# ดู Resources ที่ Deploy แล้ว
kubectl get all -n workshop

# Force Reconcile
flux reconcile kustomization podinfo
flux reconcile helmrelease redis -n workshop

# ดู Logs
flux logs --follow
```

### Step 7: ตั้งค่า Image Automation

```bash
# ติดตั้ง Image Automation Controllers (ถ้ายังไม่มี)
flux install \
  --components-extra=image-reflector-controller,image-automation-controller
```

```yaml
# clusters/workshop/image-automation.yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  image: ghcr.io/stefanprodan/podinfo
  interval: 5m

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: podinfo
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo
  policy:
    semver:
      range: ">=6.0.0"

---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@workshop.com
        name: Flux Workshop
      messageTemplate: "Update image {{ range .Updated.Images }}{{println .}} {{end}}"
    push:
      branch: main
  update:
    strategy: Setters
    path: ./apps
```

```yaml
# apps/podinfo/base/deployment.yaml (เพิ่ม Marker)
containers:
- name: podinfo
  image: ghcr.io/stefanprodan/podinfo:6.5.0 # {"$imagepolicy": "flux-system:podinfo"}
```

### Step 8: ตั้งค่า Notifications

```yaml
# infrastructure/notifications/slack.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: slack-webhook
  namespace: flux-system
stringData:
  address: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK

---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: workshop-deployments
  secretRef:
    name: slack-webhook

---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: all-events
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
  - kind: Kustomization
    namespace: flux-system
    name: '*'
  - kind: HelmRelease
    namespace: workshop
    name: '*'
```

## 11. Flux Upgrade

```bash
# อัปเดต Flux CLI
brew upgrade fluxcd/tap/flux  # macOS

# อัปเดต Flux ใน Cluster
flux install --export > clusters/workshop/flux-system/gotk-components.yaml
git add clusters/workshop/flux-system/gotk-components.yaml
git commit -m "Update Flux to latest"
git push
```

## 12. Troubleshooting

### 12.1 ดู Logs

```bash
# ดู Logs ของ Controller
flux logs --level=debug --all-namespaces

# ดู Logs เฉพาะ Source Controller
kubectl logs -n flux-system deploy/source-controller --tail=50

# ดู Logs เฉพาะ Kustomize Controller
kubectl logs -n flux-system deploy/kustomize-controller --tail=50
```

### 12.2 Force Reconciliation

```bash
# Force Reconcile GitRepository
flux reconcile source git gitops-repo -n flux-system

# Force Reconcile Kustomization
flux reconcile kustomization apps -n flux-system

# Force Reconcile HelmRelease
flux reconcile helmrelease myapp -n production

# Suspend และ Resume
flux suspend kustomization apps
flux resume kustomization apps
```

### 12.3 Debug Kustomize Build

```bash
# ตรวจสอบว่า Kustomize Build ถูกต้อง
flux diff kustomization apps --path ./apps

# Preview Resources
kubectl kustomize apps/production
```

## สรุป

Flux CD ให้:
1. **Multi-controller Architecture** - แต่ละ Controller ทำหน้าที่เฉพาะ
2. **Source Management** - จัดการ Git, Helm, OCI Sources
3. **Kustomize Native** - รองรับ Kustomize อย่างเต็มที่
4. **Helm Native** - รองรับ Helm แบบ First-class
5. **Image Automation** - Update Image Tags อัตโนมัติ
6. **SOPS Encryption** - Secret Encryption Built-in
7. **Multi-tenant** - แยก Tenant ด้วย RBAC

ในบทต่อไปจะเรียนรู้ Helm ซึ่งเป็น Package Manager สำหรับ Kubernetes

## แบบฝึกหัด

1. Bootstrap Flux ด้วย GitHub และ สร้าง Kustomization สำหรับ Application
2. Deploy Helm Chart ด้วย HelmRelease
3. ตั้งค่า Image Automation สำหรับ Auto-update Image
4. Encrypt Secrets ด้วย SOPS+Age
5. ตั้งค่า Slack Notifications
