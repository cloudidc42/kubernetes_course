# Part 71: CI/CD สำหรับ Kubernetes และ GitOps Principles

## บทนำ

ในยุคของ Cloud Native Development การพัฒนาและ Deploy แอปพลิเคชันอย่างรวดเร็วและปลอดภัยเป็นสิ่งสำคัญมาก CI/CD (Continuous Integration/Continuous Delivery) และ GitOps เป็นแนวทางที่ช่วยให้ทีมพัฒนาสามารถ Deploy แอปพลิเคชันได้อย่างมีประสิทธิภาพและเชื่อถือได้

## 1. CI/CD คืออะไร

### 1.1 Continuous Integration (CI)

Continuous Integration คือแนวทางการพัฒนาซอฟต์แวร์ที่นักพัฒนาทุกคน Merge โค้ดเข้า Shared Repository บ่อยครั้ง โดยปกติวันละหลายครั้ง การ Merge แต่ละครั้งจะถูก Verify โดยอัตโนมัติผ่าน Build และ Test

```
Developer 1 ──┐
Developer 2 ──┤──> Git Repository ──> CI Pipeline ──> Test Results
Developer 3 ──┘        │
                        └──> Code Review ──> Merge
```

**หลักการของ CI:**
- นักพัฒนา Commit โค้ดบ่อยๆ (อย่างน้อยวันละครั้ง)
- Build อัตโนมัติทุกครั้งที่มีการ Commit
- Test อัตโนมัติ (Unit, Integration, E2E)
- Fast Feedback เมื่อมีปัญหา
- Maintain a Single Source of Truth

### 1.2 Continuous Delivery (CD)

Continuous Delivery คือการขยาย CI โดยทุก Code Change ที่ผ่าน CI จะสามารถ Deploy ไปยัง Production ได้อัตโนมัติ

```
CI Pipeline ──> Staging Deploy ──> Acceptance Tests ──> Manual Approval ──> Production Deploy
```

### 1.3 Continuous Deployment

Continuous Deployment ไปอีกขั้นคือ ไม่ต้องมี Manual Approval ทุกอย่าง Deploy อัตโนมัติตลอด

```
Code Push ──> CI ──> CD ──> Production (อัตโนมัติ 100%)
```

## 2. GitOps Principles

### 2.1 GitOps คืออะไร

GitOps คือ Operational Framework ที่ใช้ Git เป็น Single Source of Truth สำหรับ Infrastructure และ Application Deployment

**4 หลักการหลักของ GitOps:**

1. **Declarative** - System ทั้งหมดต้องถูก Describe Declaratively
2. **Versioned and Immutable** - State ที่ต้องการต้องถูก Store ใน Version Control
3. **Pulled Automatically** - Software Agents ดึง Desired State โดยอัตโนมัติ
4. **Continuously Reconciled** - Software Agents Reconcile Desired vs Actual State

```yaml
# GitOps Flow
Developer ──> Git Push ──> Git Repository
                               │
                               ▼
                          GitOps Agent (ArgoCD/Flux)
                               │
                               ▼
                    Compare: Desired vs Actual State
                               │
                    ┌──────────┴──────────┐
                    │                     │
                 Same               Different
                    │                     │
                 No Action          Reconcile (Deploy)
```

### 2.2 Push vs Pull Model

**Push Model (Traditional CI/CD):**
```
Git Repository ──> CI/CD Tool ──> kubectl apply ──> Kubernetes Cluster
```
- CI/CD Tool มี Access ไปยัง Kubernetes Cluster
- ง่ายต่อการ Setup
- มี Security Risk เพราะ Credentials ต้องอยู่ใน CI/CD Tool

**Pull Model (GitOps):**
```
Git Repository <── GitOps Agent ──> Kubernetes Cluster
                        (in-cluster)
```
- GitOps Agent รันอยู่ใน Cluster
- Agent ดึง Config จาก Git
- ไม่ต้องให้ External Tool มี Access ไปยัง Cluster
- Security ดีกว่า

### 2.3 Benefits of GitOps

1. **Auditability** - ทุก Change มี Git History
2. **Rollback** - ง่ายต่อการ Rollback (แค่ Revert Git Commit)
3. **Consistency** - สิ่งที่อยู่ใน Git = สิ่งที่รันใน Production
4. **Developer Experience** - ใช้ Git Workflow ที่คุ้นเคย
5. **Security** - Credentials ไม่ต้องอยู่ใน CI/CD Tool

## 3. CI/CD Pipeline Design สำหรับ Kubernetes

### 3.1 Pipeline Stages

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline                           │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│  Source  │  Build   │   Test   │  Package │    Deploy      │
│          │          │          │          │                │
│ Git Push │ Compile  │ Unit Test│ Docker   │ Dev -> Staging │
│ PR Merge │ Lint     │ Int Test │ Build    │ -> Production  │
│          │ Security │ E2E Test │ Helm Pkg │                │
└──────────┴──────────┴──────────┴──────────┴────────────────┘
```

### 3.2 Source Stage

```yaml
# .github/workflows/ci.yml (ตัวอย่าง)
name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

### 3.3 Build Stage

```dockerfile
# Multi-stage Dockerfile สำหรับ Go Application
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main ./cmd/main.go

FROM alpine:3.18
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
COPY --from=builder /app/main /app/main
USER appuser
EXPOSE 8080
CMD ["/app/main"]
```

```dockerfile
# Multi-stage Dockerfile สำหรับ Node.js Application
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

### 3.4 Test Stage

```yaml
# Unit Tests Configuration
test:
  stage: test
  script:
    - go test ./... -v -coverprofile=coverage.out
    - go tool cover -html=coverage.out -o coverage.html
  coverage: '/coverage: \d+\.\d+%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

### 3.5 Security Scanning

```yaml
# Trivy Security Scan
security-scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --format json --output trivy-report.json $IMAGE_NAME:$CI_COMMIT_SHA
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE_NAME:$CI_COMMIT_SHA
  artifacts:
    when: always
    paths:
      - trivy-report.json
```

### 3.6 Package Stage

```yaml
# Docker Build and Push
build-push:
  stage: package
  script:
    - docker build -t $REGISTRY/$IMAGE:$VERSION .
    - docker push $REGISTRY/$IMAGE:$VERSION
    - docker tag $REGISTRY/$IMAGE:$VERSION $REGISTRY/$IMAGE:latest
    - docker push $REGISTRY/$IMAGE:latest
```

### 3.7 Deploy Stage

```yaml
# Kubernetes Deploy
deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/myapp myapp=$REGISTRY/$IMAGE:$VERSION -n staging
    - kubectl rollout status deployment/myapp -n staging

deploy-production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  when: manual
  script:
    - kubectl set image deployment/myapp myapp=$REGISTRY/$IMAGE:$VERSION -n production
    - kubectl rollout status deployment/myapp -n production
```

## 4. Container Registry

### 4.1 ทำไมต้องมี Container Registry

Container Registry เป็นที่เก็บ Docker Images ซึ่งจำเป็นสำหรับ:
- เก็บ Images อย่างเป็นระบบ
- Version Control สำหรับ Images
- Access Control
- Vulnerability Scanning

### 4.2 ประเภท Container Registry

**Public Registries:**
- Docker Hub (hub.docker.com)
- GitHub Container Registry (ghcr.io)
- Quay.io

**Private/Self-hosted:**
- Harbor (ยอดนิยมใน Enterprise)
- Docker Registry V2
- Nexus Repository

**Cloud Managed:**
- AWS ECR
- Google Artifact Registry
- Azure Container Registry

### 4.3 Image Tagging Strategy

```bash
# Semantic Versioning
myapp:1.2.3
myapp:1.2
myapp:1
myapp:latest

# Git-based Tagging
myapp:abc1234          # Git SHA
myapp:main-abc1234     # Branch + SHA
myapp:v1.2.3-abc1234   # Tag + SHA

# Date-based
myapp:20240115-abc1234
```

### 4.4 Harbor Registry Setup

```yaml
# harbor-values.yaml
expose:
  type: ingress
  ingress:
    hosts:
      core: registry.example.com
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls

externalURL: https://registry.example.com

persistence:
  persistentVolumeClaim:
    registry:
      storageClass: fast-storage
      size: 100Gi
    chartmuseum:
      storageClass: standard
      size: 10Gi

trivy:
  enabled: true
  
notary:
  enabled: true
```

```bash
# ติดตั้ง Harbor
helm repo add harbor https://helm.goharbor.io
helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  -f harbor-values.yaml
```

## 5. Kubernetes Deployment Strategies

### 5.1 Rolling Update (Default)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # สร้าง Pod ใหม่ได้สูงสุด 1 เกิน
      maxUnavailable: 0  # ไม่ให้ Pod ไม่พร้อมใช้งาน
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

### 5.2 Blue-Green Deployment

```yaml
# Blue Deployment (Current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1

---
# Green Deployment (New)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2

---
# Service (สลับระหว่าง blue/green)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # เปลี่ยนเป็น green เพื่อ switch
  ports:
  - port: 80
    targetPort: 8080
```

### 5.3 Canary Deployment

```yaml
# Stable Deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9  # 90% ของ traffic
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: myapp
        image: myapp:v1

---
# Canary Deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # 10% ของ traffic
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: myapp
        image: myapp:v2

---
# Service (ชี้ไปที่ทั้งคู่)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # จะเลือกทั้ง stable และ canary
  ports:
  - port: 80
    targetPort: 8080
```

### 5.4 A/B Testing ด้วย Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
    # หรือ based on header
    nginx.ingress.kubernetes.io/canary-by-header: "X-Version"
    nginx.ingress.kubernetes.io/canary-by-header-value: "v2"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-v2
            port:
              number: 80
```

## 6. Secrets Management ใน CI/CD

### 6.1 ปัญหาของ Secrets ใน CI/CD

- อย่า Hardcode Secrets ใน Code
- อย่า Commit Secrets ลง Git
- ต้องมี Rotation Policy
- ต้องมี Audit Trail

### 6.2 Kubernetes Secrets

```yaml
# สร้าง Secret สำหรับ Docker Registry
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

```bash
# สร้าง Secret จาก kubectl
kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

### 6.3 External Secrets Operator

```yaml
# ClusterSecretStore (เชื่อมต่อกับ AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key

---
# ExternalSecret (ดึง Secret จาก AWS)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: ClusterSecretStore
  target:
    name: myapp-secrets
    creationPolicy: Owner
  data:
  - secretKey: db-password
    remoteRef:
      key: myapp/prod/db-password
  - secretKey: api-key
    remoteRef:
      key: myapp/prod/api-key
```

### 6.4 Vault Integration

```yaml
# Vault Agent Sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-config: "secret/myapp/config"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/myapp/config" -}}
          export DB_PASSWORD={{ .Data.data.db_password }}
          export API_KEY={{ .Data.data.api_key }}
          {{- end }}
    spec:
      serviceAccountName: myapp
      containers:
      - name: myapp
        image: myapp:latest
        command: ["sh", "-c", "source /vault/secrets/config && ./myapp"]
```

## 7. Quality Gates

### 7.1 Code Quality Gates

```yaml
# SonarQube Quality Gate
sonar:
  stage: test
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner
      -Dsonar.projectKey=myapp
      -Dsonar.sources=.
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
      -Dsonar.coverage.exclusions=**/*_test.go
      -Dsonar.go.coverage.reportPaths=coverage.out
  allow_failure: false
```

### 7.2 Test Coverage Gate

```yaml
# ต้องมี Coverage อย่างน้อย 80%
coverage-check:
  stage: test
  script:
    - go test ./... -coverprofile=coverage.out
    - |
      COVERAGE=$(go tool cover -func coverage.out | grep total | awk '{print $3}' | tr -d '%')
      if (( $(echo "$COVERAGE < 80" | bc -l) )); then
        echo "Coverage $COVERAGE% is below threshold 80%"
        exit 1
      fi
      echo "Coverage $COVERAGE% passes threshold"
```

### 7.3 Security Gates

```yaml
# Dependency Check
dependency-check:
  stage: test
  script:
    # Go
    - govulncheck ./...
    # Node.js
    - npm audit --audit-level=high
    # Python
    - safety check -r requirements.txt
```

## 8. Notification และ Monitoring

### 8.1 Slack Notifications

```yaml
notify-slack:
  stage: notify
  when: always
  script:
    - |
      if [ "$CI_JOB_STATUS" == "success" ]; then
        COLOR="good"
        STATUS="✅ Deployment Successful"
      else
        COLOR="danger"
        STATUS="❌ Deployment Failed"
      fi
      
      curl -X POST $SLACK_WEBHOOK \
        -H 'Content-type: application/json' \
        --data "{
          \"attachments\": [{
            \"color\": \"$COLOR\",
            \"title\": \"$STATUS\",
            \"fields\": [
              {\"title\": \"Project\", \"value\": \"$CI_PROJECT_NAME\", \"short\": true},
              {\"title\": \"Branch\", \"value\": \"$CI_COMMIT_BRANCH\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"$CI_COMMIT_SHORT_SHA\", \"short\": true},
              {\"title\": \"Author\", \"value\": \"$CI_COMMIT_AUTHOR\", \"short\": true}
            ]
          }]
        }"
```

### 8.2 Deployment Metrics

```yaml
# Prometheus Annotations สำหรับ Deployment Tracking
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubernetes.io/change-cause: "Update to v1.2.3"
```

## 9. Multi-Environment Pipeline

### 9.1 Environment Configuration

```
environments/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── dev/
│   ├── kustomization.yaml
│   └── patch-dev.yaml
├── staging/
│   ├── kustomization.yaml
│   └── patch-staging.yaml
└── production/
    ├── kustomization.yaml
    └── patch-production.yaml
```

### 9.2 Environment-specific Values

```yaml
# base/deployment.yaml
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
        image: myapp:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi

---
# production/patch-production.yaml
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
            cpu: 1000m
            memory: 1Gi
```

### 9.3 GitOps Repository Structure

```
gitops-repo/
├── apps/
│   ├── myapp/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   ├── dev/
│   │   │   └── kustomization.yaml
│   │   ├── staging/
│   │   │   └── kustomization.yaml
│   │   └── production/
│   │       └── kustomization.yaml
│   └── otherapp/
│       └── ...
├── infrastructure/
│   ├── monitoring/
│   ├── logging/
│   └── networking/
└── clusters/
    ├── dev-cluster/
    ├── staging-cluster/
    └── production-cluster/
```

## 10. Workshop: วางแผน CI/CD Pipeline

### Workshop Objective

สร้าง CI/CD Pipeline สำหรับ Web Application ที่มี:
- Frontend (React)
- Backend (Go API)
- Database (PostgreSQL)

### Step 1: วิเคราะห์ Requirements

```
Application Requirements:
1. Frontend: React SPA
   - Build: npm build
   - Test: Jest
   - Docker Image: nginx-based
   
2. Backend: Go REST API
   - Build: go build
   - Test: go test
   - Docker Image: scratch-based
   
3. Database: PostgreSQL
   - Migrations: golang-migrate
   - Test DB: Docker
```

### Step 2: ออกแบบ Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CI/CD Pipeline Design                           │
├─────────┬──────────┬──────────┬────────────┬──────────┬────────────┤
│ Stage 1 │ Stage 2  │ Stage 3  │  Stage 4   │ Stage 5  │  Stage 6   │
│ Source  │  Build   │   Test   │  Security  │  Push    │   Deploy   │
├─────────┼──────────┼──────────┼────────────┼──────────┼────────────┤
│ Trigger │ FE Build │ Unit     │ SAST       │ Registry │ Dev Auto   │
│ PR/Push │ BE Build │ Integrat │ DAST       │ Push     │ Staging    │
│         │ DB Migr  │ E2E      │ Container  │ Sign     │ Prod Manual│
└─────────┴──────────┴──────────┴────────────┴──────────┴────────────┘
```

### Step 3: กำหนด Environment Variables

```bash
# CI/CD Variables ที่ต้องการ
REGISTRY_URL=registry.example.com
REGISTRY_USER=ci-bot
REGISTRY_PASSWORD=<secret>

# Kubernetes
KUBECONFIG_DEV=<base64-encoded>
KUBECONFIG_STAGING=<base64-encoded>
KUBECONFIG_PROD=<base64-encoded>

# Notifications
SLACK_WEBHOOK=<secret>

# SonarQube
SONAR_HOST_URL=https://sonar.example.com
SONAR_TOKEN=<secret>
```

### Step 4: สร้าง Pipeline File

```yaml
# .github/workflows/pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: registry.example.com
  IMAGE_NAME: myapp

jobs:
  # Stage 1: Lint and Static Analysis
  lint:
    name: Lint & Static Analysis
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Run golangci-lint
      uses: golangci/golangci-lint-action@v3
      with:
        version: latest
    
    - name: Setup Node
      uses: actions/setup-node@v4
      with:
        node-version: '18'
    
    - name: Run ESLint
      working-directory: frontend
      run: |
        npm ci
        npm run lint

  # Stage 2: Test
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Go Tests
      env:
        DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable
      run: |
        go test ./... -v -coverprofile=coverage.out
        go tool cover -func=coverage.out
    
    - name: Check Coverage
      run: |
        COVERAGE=$(go tool cover -func coverage.out | grep total | awk '{print $3}' | tr -d '%')
        echo "Coverage: $COVERAGE%"
        if (( $(echo "$COVERAGE < 80" | bc -l) )); then
          echo "ERROR: Coverage below 80%"
          exit 1
        fi
    
    - name: Run Frontend Tests
      working-directory: frontend
      run: |
        npm ci
        npm test -- --coverage --watchAll=false

  # Stage 3: Security Scan
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        exit-code: '1'
        severity: 'HIGH,CRITICAL'
    
    - name: Run gosec
      uses: securego/gosec@master
      with:
        args: '-severity high ./...'

  # Stage 4: Build and Push
  build:
    name: Build and Push Images
    runs-on: ubuntu-latest
    needs: security
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USER }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,prefix=sha-
          type=semver,pattern={{version}}
    
    - name: Build and push Backend
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Build and push Frontend
      uses: docker/build-push-action@v5
      with:
        context: frontend
        file: frontend/Dockerfile
        push: true
        tags: ${{ env.REGISTRY }}/frontend:${{ github.sha }}

  # Stage 5: Deploy to Dev
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://dev.example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure kubeconfig
      run: |
        echo "${{ secrets.KUBECONFIG_DEV }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Deploy to Dev
      run: |
        kubectl set image deployment/myapp-backend backend=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} -n dev
        kubectl set image deployment/myapp-frontend frontend=${{ env.REGISTRY }}/frontend:${{ github.sha }} -n dev
        kubectl rollout status deployment/myapp-backend -n dev --timeout=5m
        kubectl rollout status deployment/myapp-frontend -n dev --timeout=5m

  # Stage 6: Deploy to Production
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to Production
      run: |
        echo "Deploying ${{ github.sha }} to production"
        # GitOps: Update image tag in GitOps repo
        git clone https://token:${{ secrets.GITOPS_TOKEN }}@github.com/org/gitops-repo
        cd gitops-repo
        sed -i "s|image: .*myapp:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}|g" apps/myapp/production/deployment.yaml
        git config user.email "ci@example.com"
        git config user.name "CI Bot"
        git add .
        git commit -m "Deploy myapp:${{ github.sha }} to production"
        git push
```

### Step 5: Testing the Pipeline

```bash
# สร้าง Feature Branch
git checkout -b feature/new-api

# ทำงาน...
git add .
git commit -m "feat: add new API endpoint"
git push origin feature/new-api

# สร้าง Pull Request
gh pr create --title "Add new API endpoint" --body "..."

# Pipeline จะรัน:
# 1. Lint & Static Analysis
# 2. Unit & Integration Tests
# 3. Security Scan

# Merge to develop
gh pr merge --squash

# Pipeline จะรัน:
# 1-3 เหมือนเดิม
# 4. Build & Push Images
# 5. Deploy to Dev

# Merge to main (release)
git checkout main
git merge develop
git push origin main

# Pipeline จะรัน:
# 1-4 เหมือนเดิม  
# 5. Deploy to Production (ต้องมีคนกด Approve)
```

## 11. Best Practices

### 11.1 Pipeline Best Practices

1. **Fast Feedback** - ทดสอบที่เร็วที่สุดก่อน
2. **Fail Fast** - หยุด Pipeline ทันทีเมื่อมีปัญหา
3. **Idempotent** - รัน Pipeline ซ้ำได้
4. **Immutable Artifacts** - Image ที่ Build แล้วไม่ควรเปลี่ยน
5. **Environment Parity** - Dev, Staging, Prod ใช้ Config เหมือนกัน

### 11.2 Security Best Practices

```yaml
# ใช้ Specific Version แทน latest
FROM golang:1.21.5-alpine3.18  # ✓ ดี
FROM golang:latest               # ✗ ไม่ดี

# Run as Non-root
RUN adduser -u 1001 -S appuser
USER appuser

# Read-only filesystem
securityContext:
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

### 11.3 GitOps Best Practices

1. **Separate App and Config Repos** - แยก Source Code จาก Deployment Config
2. **Environment Branches** - ใช้ Branches แยกต่อ Environment หรือ Directories
3. **Automated Sync** - ใช้ ArgoCD/Flux สำหรับ Automatic Sync
4. **PR Reviews** - ทุก Production Change ต้องผ่าน PR Review
5. **Secret Management** - ไม่เก็บ Secrets ใน Git

## สรุป

ในบทนี้เราได้เรียนรู้:
1. CI/CD Concepts และความแตกต่างระหว่าง CI, CD, และ Continuous Deployment
2. GitOps Principles ทั้ง 4 ข้อ
3. Pipeline Design สำหรับ Kubernetes
4. Container Registry และ Image Tagging Strategies
5. Deployment Strategies (Rolling, Blue-Green, Canary)
6. Secrets Management ใน CI/CD
7. Quality Gates สำหรับ Code Quality, Coverage, และ Security
8. Workshop: วางแผน CI/CD Pipeline แบบครบวงจร

ในบทต่อไปเราจะเจาะลึกเรื่อง Jenkins บน Kubernetes ซึ่งเป็นหนึ่งใน CI/CD Tools ที่ได้รับความนิยมมากที่สุด

## แบบฝึกหัด

1. วิเคราะห์ Application ที่คุณทำงานอยู่และออกแบบ CI/CD Pipeline ที่เหมาะสม
2. สร้าง Dockerfile แบบ Multi-stage สำหรับ Application ของคุณ
3. ออกแบบ Image Tagging Strategy ที่เหมาะกับ Team ของคุณ
4. วางแผน GitOps Repository Structure สำหรับ Project ของคุณ
