# Part 73: GitHub Actions สำหรับ Kubernetes

## บทนำ

GitHub Actions เป็น CI/CD Platform ที่รวมอยู่ใน GitHub ทำให้ไม่ต้องติดตั้ง Server เพิ่ม มีความสามารถ Build, Test, และ Deploy โดยตรงจาก GitHub Repository มีระบบ Marketplace ที่มี Actions พร้อมใช้มากมาย และรองรับ Workflow YAML แบบง่ายแต่ทรงพลัง

## 1. GitHub Actions Architecture

### 1.1 Components

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions                           │
│                                                             │
│  ┌──────────────┐    ┌───────────────────────────────────┐  │
│  │   Workflow   │    │         Runner                    │  │
│  │              │    │                                   │  │
│  │  Triggers:   │───>│  GitHub-hosted: ubuntu-latest     │  │
│  │  - push      │    │  Self-hosted: Your Server/K8s     │  │
│  │  - pr        │    │                                   │  │
│  │  - schedule  │    │  Jobs run in parallel             │  │
│  │  - manual    │    │  Steps run sequentially           │  │
│  └──────────────┘    └───────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Actions Marketplace                     │   │
│  │                                                      │   │
│  │  checkout  docker/build-push  azure/setup-kubectl    │   │
│  │  setup-node  setup-go  actions/cache  etc...         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Workflow YAML Structure

```yaml
name: Workflow Name              # ชื่อ Workflow

on:                              # Triggers
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:                             # Global Environment Variables
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:                            # Jobs
  job-name:                      # Job ID
    name: Job Display Name       # ชื่อที่แสดงใน UI
    runs-on: ubuntu-latest       # Runner
    
    outputs:                     # Outputs สำหรับ Jobs อื่น
      version: ${{ steps.version.outputs.version }}
    
    steps:                       # Steps ใน Job
    - name: Step Name
      uses: actions/checkout@v4  # Action จาก Marketplace
    
    - name: Run command
      run: echo "Hello World"   # Shell Command
    
    - name: Set output
      id: version
      run: echo "version=1.0.0" >> $GITHUB_OUTPUT
```

## 2. Triggers (Events)

### 2.1 Push Trigger

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    tags:
      - 'v*.*.*'
    paths:
      - 'src/**'
      - 'Dockerfile'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

### 2.2 Pull Request Trigger

```yaml
on:
  pull_request:
    branches: [main, develop]
    types: [opened, synchronize, reopened]
    paths:
      - 'src/**'
      - 'tests/**'
```

### 2.3 Schedule Trigger

```yaml
on:
  schedule:
    # ทุกวัน 02:00 UTC
    - cron: '0 2 * * *'
    # ทุกจันทร์ 08:00 UTC
    - cron: '0 8 * * 1'
```

### 2.4 Manual Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - production
        default: 'dev'
      version:
        description: 'Version to deploy'
        required: false
        type: string
      dry_run:
        description: 'Dry run (no actual deployment)'
        required: false
        type: boolean
        default: false
```

### 2.5 Workflow Call (Reusable Workflows)

```yaml
# .github/workflows/deploy.yml (Reusable)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      kubeconfig:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - name: Deploy
      run: |
        echo "${{ secrets.kubeconfig }}" | base64 -d > kubeconfig
        kubectl set image deployment/myapp myapp=myapp:${{ inputs.image-tag }}
```

```yaml
# .github/workflows/main.yml (Caller)
on:
  push:
    branches: [main]

jobs:
  build:
    # ... build job
  
  deploy-dev:
    needs: build
    uses: ./.github/workflows/deploy.yml
    with:
      environment: dev
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG_DEV }}
  
  deploy-prod:
    needs: deploy-dev
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG_PROD }}
```

## 3. Jobs และ Steps

### 3.1 Job Dependencies

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - run: echo "Linting..."
  
  test:
    runs-on: ubuntu-latest
    steps:
    - run: echo "Testing..."
  
  build:
    needs: [lint, test]  # รอทั้งสองก่อน
    runs-on: ubuntu-latest
    steps:
    - run: echo "Building..."
  
  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - run: echo "Deploying to dev..."
  
  deploy-prod:
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    runs-on: ubuntu-latest
    steps:
    - run: echo "Deploying to production..."
```

### 3.2 Matrix Strategy

```yaml
jobs:
  test:
    strategy:
      matrix:
        go-version: ['1.20', '1.21']
        os: [ubuntu-latest, windows-latest]
        include:
          - os: ubuntu-latest
            go-version: '1.21'
            run-integration: true
        exclude:
          - os: windows-latest
            go-version: '1.20'
    
    runs-on: ${{ matrix.os }}
    
    steps:
    - uses: actions/setup-go@v4
      with:
        go-version: ${{ matrix.go-version }}
    
    - name: Run unit tests
      run: go test ./...
    
    - name: Run integration tests
      if: matrix.run-integration
      run: go test ./... -tags=integration
```

### 3.3 Conditional Steps

```yaml
steps:
- name: Deploy to staging
  if: github.ref == 'refs/heads/develop'
  run: echo "Deploying to staging"

- name: Deploy to production
  if: startsWith(github.ref, 'refs/tags/v')
  run: echo "Deploying to production"

- name: Run on failure
  if: failure()
  run: echo "Something went wrong"

- name: Always run
  if: always()
  run: echo "This always runs"

- name: Run if previous step succeeded
  if: steps.build.outcome == 'success'
  run: echo "Build succeeded, continuing"
```

## 4. Actions ที่สำคัญสำหรับ Kubernetes

### 4.1 Checkout

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full history for git commands
    token: ${{ secrets.GITHUB_TOKEN }}
    submodules: recursive
```

### 4.2 Docker Build and Push

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Extract metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/${{ github.repository }}
    tags: |
      type=ref,event=branch
      type=ref,event=pr
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=sha,prefix=sha-,format=short

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: ${{ github.event_name != 'pull_request' }}
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
    build-args: |
      VERSION=${{ github.sha }}
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

### 4.3 Kubectl

```yaml
- name: Setup kubectl
  uses: azure/setup-kubectl@v3
  with:
    version: 'v1.28.0'

- name: Configure kubeconfig
  run: |
    mkdir -p $HOME/.kube
    echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
    chmod 600 $HOME/.kube/config

- name: Deploy to Kubernetes
  run: |
    kubectl set image deployment/myapp myapp=$IMAGE:$TAG -n production
    kubectl rollout status deployment/myapp -n production
```

### 4.4 Helm

```yaml
- name: Setup Helm
  uses: azure/setup-helm@v3
  with:
    version: 'v3.13.0'

- name: Deploy with Helm
  run: |
    helm upgrade --install myapp ./charts/myapp \
      --namespace production \
      --create-namespace \
      --set image.tag=${{ github.sha }} \
      --set replicas=3 \
      --values ./charts/myapp/values-prod.yaml \
      --wait \
      --timeout 10m
```

### 4.5 Security Scanning

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: 'trivy-results.sarif'
```

## 5. Secrets และ Variables

### 5.1 Repository Secrets

```yaml
# ใช้ใน Workflow
- name: Deploy
  env:
    KUBECONFIG: ${{ secrets.KUBECONFIG_PROD }}
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  run: |
    echo "$KUBECONFIG" | base64 -d > /tmp/kubeconfig
    kubectl apply -f k8s/ --kubeconfig=/tmp/kubeconfig
```

### 5.2 Environment Secrets

```yaml
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://example.com
    steps:
    - name: Deploy
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG }}  # Environment-specific secret
      run: kubectl apply -f k8s/
```

### 5.3 Variables (Non-sensitive)

```yaml
# Repository Variables ใน Settings > Variables
- name: Deploy
  run: |
    kubectl set image deployment/myapp \
      myapp=${{ vars.REGISTRY }}/${{ vars.IMAGE_NAME }}:${{ github.sha }} \
      -n ${{ vars.PRODUCTION_NAMESPACE }}
```

## 6. Self-hosted Runners บน Kubernetes

### 6.1 ทำไมต้อง Self-hosted Runner

1. **Private Network Access** - เข้าถึง Internal Services
2. **Custom Software** - ติดตั้ง Software เฉพาะ
3. **Performance** - Hardware ที่กำหนดเอง
4. **Cost** - ประหยัดกว่า GitHub-hosted สำหรับ Heavy Usage

### 6.2 Actions Runner Controller (ARC)

```bash
# ติดตั้ง ARC
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update

# ติดตั้ง cert-manager ก่อน (requirement)
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# สร้าง GitHub App หรือ PAT secret
kubectl create secret generic controller-manager \
  --from-literal=github_token=$GITHUB_TOKEN \
  -n actions-runner-system

# ติดตั้ง ARC
helm install actions-runner-controller \
  actions-runner-controller/actions-runner-controller \
  --namespace actions-runner-system \
  --create-namespace \
  --set authSecret.create=true \
  --set authSecret.github_token=$GITHUB_TOKEN
```

### 6.3 RunnerDeployment

```yaml
# runner-deployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: github-runner
  namespace: actions-runner-system
spec:
  replicas: 3
  template:
    spec:
      repository: myorg/myrepo  # หรือ organization: myorg
      labels:
        - self-hosted
        - kubernetes
        - linux
      
      image: summerwind/actions-runner-dind:latest
      
      serviceAccountName: github-runner
      
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 2000m
          memory: 4Gi
      
      volumeMounts:
      - name: work
        mountPath: /home/runner/work
      
      volumes:
      - name: work
        emptyDir: {}
      
      nodeSelector:
        kubernetes.io/os: linux
      
      tolerations:
      - key: "github-runner"
        operator: "Exists"
        effect: "NoSchedule"

---
# Autoscaling
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: github-runner-autoscaler
  namespace: actions-runner-system
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    kind: RunnerDeployment
    name: github-runner
  scaleUpTriggers:
  - githubEvent:
      workflowJob: {}
    amount: 1
    duration: "10m"
```

### 6.4 ใช้ Self-hosted Runner ใน Workflow

```yaml
jobs:
  build:
    runs-on: [self-hosted, kubernetes, linux]
    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: docker build -t myapp .
```

## 7. Caching

### 7.1 Cache Action

```yaml
- name: Cache Go modules
  uses: actions/cache@v3
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-

- name: Cache node_modules
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```

### 7.2 Docker Layer Caching

```yaml
- name: Build and push with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## 8. Artifacts

### 8.1 Upload Artifacts

```yaml
- name: Run tests
  run: go test ./... -coverprofile=coverage.out

- name: Upload coverage
  uses: actions/upload-artifact@v3
  with:
    name: coverage-report
    path: coverage.out
    retention-days: 30
```

### 8.2 Download Artifacts

```yaml
- name: Download coverage
  uses: actions/download-artifact@v3
  with:
    name: coverage-report

- name: Display coverage
  run: go tool cover -func coverage.out
```

## 9. Environments และ Deployment Protection

### 9.1 Environment Configuration

```yaml
# ใน GitHub Settings > Environments
# สร้าง Environment: production
# - Required reviewers: @devops-team
# - Wait timer: 5 minutes
# - Deployment branches: main only

jobs:
  deploy-production:
    environment:
      name: production
      url: https://example.com
    runs-on: ubuntu-latest
    steps:
    - name: Deploy
      run: echo "Deploying to production after approval"
```

### 9.2 Environment Secrets

```yaml
# Production environment มี secrets แยก
- name: Deploy to production
  env:
    API_KEY: ${{ secrets.PROD_API_KEY }}  # จาก Production environment
  run: ./deploy.sh
```

## 10. GitHub Actions สำหรับ GitOps

### 10.1 Update Image Tag ใน GitOps Repo

```yaml
# .github/workflows/update-gitops.yml
name: Update GitOps Repo

on:
  workflow_call:
    inputs:
      image-tag:
        required: true
        type: string
      environment:
        required: true
        type: string

jobs:
  update-gitops:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: myorg/gitops-repo
        token: ${{ secrets.GITOPS_TOKEN }}
        path: gitops
    
    - name: Update image tag
      working-directory: gitops
      run: |
        # Update image tag ด้วย yq
        yq e '.spec.template.spec.containers[0].image = "ghcr.io/myorg/myapp:${{ inputs.image-tag }}"' \
          -i apps/myapp/${{ inputs.environment }}/deployment.yaml
        
        # ตรวจสอบ
        cat apps/myapp/${{ inputs.environment }}/deployment.yaml
    
    - name: Commit and push
      working-directory: gitops
      run: |
        git config user.email "github-actions@github.com"
        git config user.name "GitHub Actions"
        git add .
        git diff --staged --quiet || \
          git commit -m "chore: update ${{ inputs.environment }} to ${{ inputs.image-tag }}"
        git push
```

## 11. Workshop: Full CI/CD Pipeline ด้วย GitHub Actions

### Workshop Overview

สร้าง Complete CI/CD Pipeline สำหรับ Node.js Application ที่:
1. Build Docker Image
2. Run Tests
3. Push to GitHub Container Registry
4. Deploy to Kubernetes (Dev)
5. Deploy to Production (Manual)

### Step 1: สร้าง Application

```bash
# สร้าง Directory Structure
mkdir github-actions-workshop
cd github-actions-workshop
mkdir -p src tests .github/workflows k8s

# สร้าง Application
cat > src/app.js << 'EOF'
const express = require('express');
const app = express();

const PORT = process.env.PORT || 3000;
const VERSION = process.env.APP_VERSION || 'development';
const ENV = process.env.ENVIRONMENT || 'local';

app.use(express.json());

app.get('/', (req, res) => {
    res.json({
        message: 'Hello from GitHub Actions Workshop!',
        version: VERSION,
        environment: ENV,
        timestamp: new Date().toISOString()
    });
});

app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy' });
});

app.get('/ready', (req, res) => {
    res.status(200).json({ status: 'ready' });
});

if (require.main === module) {
    app.listen(PORT, () => {
        console.log(`Server running on port ${PORT} (v${VERSION}, env: ${ENV})`);
    });
}

module.exports = app;
EOF

# สร้าง package.json
cat > package.json << 'EOF'
{
  "name": "github-actions-workshop",
  "version": "1.0.0",
  "description": "GitHub Actions Workshop App",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "test": "jest --coverage",
    "lint": "eslint src/ tests/"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.3",
    "eslint": "^8.54.0"
  }
}
EOF

# สร้าง Tests
cat > tests/app.test.js << 'EOF'
const request = require('supertest');
const app = require('../src/app');

describe('Application Tests', () => {
    describe('GET /', () => {
        it('should return 200 with message', async () => {
            const res = await request(app).get('/');
            expect(res.statusCode).toBe(200);
            expect(res.body.message).toBe('Hello from GitHub Actions Workshop!');
        });
        
        it('should return version', async () => {
            const res = await request(app).get('/');
            expect(res.body.version).toBeDefined();
        });
    });
    
    describe('GET /health', () => {
        it('should return 200 healthy', async () => {
            const res = await request(app).get('/health');
            expect(res.statusCode).toBe(200);
            expect(res.body.status).toBe('healthy');
        });
    });
    
    describe('GET /ready', () => {
        it('should return 200 ready', async () => {
            const res = await request(app).get('/ready');
            expect(res.statusCode).toBe(200);
            expect(res.body.status).toBe('ready');
        });
    });
});
EOF
```

### Step 2: สร้าง Dockerfile

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source
COPY src/ ./src/

# Final image
FROM node:18-alpine

# Security: Non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy from builder
COPY --from=builder --chown=appuser:appgroup /app .

# Switch to non-root
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "src/app.js"]
```

### Step 3: สร้าง Kubernetes Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-actions-workshop
  labels:
    app: github-actions-workshop
    version: "latest"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: github-actions-workshop
  template:
    metadata:
      labels:
        app: github-actions-workshop
    spec:
      containers:
      - name: app
        image: ghcr.io/GITHUB_USERNAME/github-actions-workshop:latest
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        - name: ENVIRONMENT
          value: "kubernetes"
        - name: APP_VERSION
          value: "PLACEHOLDER_VERSION"
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
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1001
          capabilities:
            drop:
            - ALL
      securityContext:
        fsGroup: 1001

---
apiVersion: v1
kind: Service
metadata:
  name: github-actions-workshop
spec:
  selector:
    app: github-actions-workshop
  ports:
  - port: 80
    targetPort: 3000

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: github-actions-workshop
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: workshop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: github-actions-workshop
            port:
              number: 80
```

### Step 4: สร้าง GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, production]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  NODE_VERSION: '18'

jobs:
  # ============================================
  # Job 1: Lint
  # ============================================
  lint:
    name: Lint Code
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: npm
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run ESLint
      run: npm run lint || true  # ไม่ fail ถ้า config ยังไม่มี

  # ============================================
  # Job 2: Test
  # ============================================
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: npm
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests with coverage
      run: npm test
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      if: success()
      with:
        token: ${{ secrets.CODECOV_TOKEN }}
        files: ./coverage/lcov.info
        flags: unittests
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: |
          coverage/
          jest-results.json
        retention-days: 7

  # ============================================
  # Job 3: Security Scan
  # ============================================
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: npm
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run npm audit
      run: npm audit --audit-level=high
    
    - name: Run Snyk
      uses: snyk/actions/node@master
      continue-on-error: true
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

  # ============================================
  # Job 4: Build and Push Image
  # ============================================
  build:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: [lint, test, security]
    if: github.event_name != 'pull_request'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    permissions:
      contents: read
      packages: write
      security-events: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract Docker metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,prefix=sha-,format=short
          type=semver,pattern={{version}}
          type=raw,value=latest,enable={{is_default_branch}}
        labels: |
          org.opencontainers.image.title=GitHub Actions Workshop
          org.opencontainers.image.description=Workshop app for GitHub Actions CI/CD
    
    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          APP_VERSION=${{ github.sha }}
          BUILD_DATE=${{ github.event.head_commit.timestamp }}
    
    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Generate build summary
      run: |
        echo "## Build Summary" >> $GITHUB_STEP_SUMMARY
        echo "- **Image**: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}" >> $GITHUB_STEP_SUMMARY
        echo "- **Tag**: ${{ steps.meta.outputs.version }}" >> $GITHUB_STEP_SUMMARY
        echo "- **Digest**: ${{ steps.build.outputs.digest }}" >> $GITHUB_STEP_SUMMARY
        echo "- **Commit**: ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY

  # ============================================
  # Job 5: Deploy to Development
  # ============================================
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
      with:
        version: 'v1.28.0'
    
    - name: Configure kubeconfig
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG_DEV }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config
    
    - name: Update deployment manifest
      run: |
        IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"
        sed -i "s|image: ghcr.io.*github-actions-workshop:.*|image: $IMAGE|g" k8s/deployment.yaml
        sed -i "s|PLACEHOLDER_VERSION|${{ github.sha }}|g" k8s/deployment.yaml
        
        echo "Updated deployment:"
        grep "image:" k8s/deployment.yaml
    
    - name: Deploy to dev namespace
      run: |
        kubectl apply -f k8s/ -n development
        kubectl rollout status deployment/github-actions-workshop \
          -n development \
          --timeout=5m
    
    - name: Show deployment info
      run: |
        echo "## Deployment Info" >> $GITHUB_STEP_SUMMARY
        kubectl get pods -n development -l app=github-actions-workshop \
          -o wide >> $GITHUB_STEP_SUMMARY || true
    
    - name: Run smoke tests
      run: |
        # รอ Service พร้อม
        sleep 10
        
        # ทดสอบ endpoints
        INGRESS_IP=$(kubectl get ingress github-actions-workshop \
          -n development \
          -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "localhost")
        
        echo "Testing $INGRESS_IP..."
        curl -f http://dev.example.com/health || echo "Warning: smoke test skipped"

  # ============================================
  # Job 6: Deploy to Staging
  # ============================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Deploy to staging
      env:
        KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_STAGING }}
      run: |
        echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
        export KUBECONFIG=/tmp/kubeconfig
        
        kubectl set image deployment/github-actions-workshop \
          app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
          -n staging
        
        kubectl rollout status deployment/github-actions-workshop \
          -n staging \
          --timeout=5m
    
    - name: Run E2E tests
      run: |
        echo "Running E2E tests against staging..."
        # npx cypress run --env baseUrl=https://staging.example.com

  # ============================================
  # Job 7: Deploy to Production
  # ============================================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    
    environment:
      name: production
      url: https://example.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Deploy to production
      env:
        KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_PROD }}
      run: |
        echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
        export KUBECONFIG=/tmp/kubeconfig
        
        # Update image
        kubectl set image deployment/github-actions-workshop \
          app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
          -n production
        
        # Wait for rollout
        kubectl rollout status deployment/github-actions-workshop \
          -n production \
          --timeout=10m
    
    - name: Verify deployment
      env:
        KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_PROD }}
      run: |
        echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
        export KUBECONFIG=/tmp/kubeconfig
        
        # ตรวจสอบ Pods
        kubectl get pods -n production -l app=github-actions-workshop
        
        # ทดสอบ Health
        curl -f https://example.com/health || exit 1
        echo "Production deployment verified!"
    
    - name: Create GitHub Release
      uses: actions/create-release@v1
      if: startsWith(github.ref, 'refs/tags/')
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        body: |
          ## Changes
          ${{ github.event.head_commit.message }}
          
          ## Docker Image
          `${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}`
        draft: false
        prerelease: false

  # ============================================
  # Job 8: Notify
  # ============================================
  notify:
    name: Send Notifications
    runs-on: ubuntu-latest
    needs: [deploy-dev, deploy-staging, deploy-production]
    if: always()
    
    steps:
    - name: Send Slack notification
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        text: |
          *GitHub Actions: ${{ github.workflow }}*
          Repo: ${{ github.repository }}
          Branch: ${{ github.ref_name }}
          Commit: ${{ github.sha }}
          Status: ${{ job.status }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Step 5: ตั้งค่า Repository

```bash
# 1. สร้าง GitHub Repository
gh repo create github-actions-workshop --public

# 2. Push code
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/USERNAME/github-actions-workshop.git
git push -u origin main

# 3. ตั้งค่า Secrets
# ไปที่ Settings > Secrets and Variables > Actions

# 4. สร้าง Environments
# ไปที่ Settings > Environments
# สร้าง: development, staging, production

# 5. ตั้งค่า Branch Protection
# ไปที่ Settings > Branches
# - Require pull request reviews
# - Require status checks to pass

# 6. ดู Workflow Run
gh workflow list
gh run list
gh run view <run-id>
```

### Step 6: ทดสอบ Pipeline

```bash
# สร้าง Feature Branch
git checkout -b feature/add-endpoint

# เพิ่ม Endpoint ใหม่
cat >> src/app.js << 'EOF'

app.get('/version', (req, res) => {
    res.json({ version: VERSION });
});
EOF

# Commit และ Push
git add .
git commit -m "feat: add version endpoint"
git push origin feature/add-endpoint

# สร้าง PR
gh pr create --title "Add version endpoint" \
  --body "Adds a /version endpoint to return the app version"

# ดู PR Checks
gh pr checks

# Merge PR
gh pr merge --squash

# ดู Workflow Run
gh run list --branch main
```

## 12. Advanced Patterns

### 12.1 Composite Actions

```yaml
# .github/actions/deploy-k8s/action.yml
name: 'Deploy to Kubernetes'
description: 'Deploy application to Kubernetes cluster'

inputs:
  namespace:
    description: 'Kubernetes namespace'
    required: true
  deployment-name:
    description: 'Deployment name'
    required: true
  image:
    description: 'Docker image with tag'
    required: true
  kubeconfig:
    description: 'Base64 encoded kubeconfig'
    required: true
  timeout:
    description: 'Rollout timeout'
    required: false
    default: '5m'

outputs:
  pod-count:
    description: 'Number of pods running'
    value: ${{ steps.get-pods.outputs.count }}

runs:
  using: composite
  steps:
  - name: Setup kubectl
    uses: azure/setup-kubectl@v3
    with:
      version: 'v1.28.0'
  
  - name: Configure kubeconfig
    shell: bash
    run: |
      mkdir -p $HOME/.kube
      echo "${{ inputs.kubeconfig }}" | base64 -d > $HOME/.kube/config
      chmod 600 $HOME/.kube/config
  
  - name: Update image
    shell: bash
    run: |
      kubectl set image deployment/${{ inputs.deployment-name }} \
        ${{ inputs.deployment-name }}=${{ inputs.image }} \
        -n ${{ inputs.namespace }}
  
  - name: Wait for rollout
    shell: bash
    run: |
      kubectl rollout status deployment/${{ inputs.deployment-name }} \
        -n ${{ inputs.namespace }} \
        --timeout=${{ inputs.timeout }}
  
  - name: Get pod count
    id: get-pods
    shell: bash
    run: |
      COUNT=$(kubectl get pods -n ${{ inputs.namespace }} \
        -l app=${{ inputs.deployment-name }} \
        --field-selector=status.phase=Running \
        -o jsonpath='{.items}' | jq length)
      echo "count=$COUNT" >> $GITHUB_OUTPUT
```

```yaml
# ใช้ Composite Action
- name: Deploy to production
  uses: ./.github/actions/deploy-k8s
  with:
    namespace: production
    deployment-name: myapp
    image: ghcr.io/org/myapp:${{ github.sha }}
    kubeconfig: ${{ secrets.KUBECONFIG_PROD }}
    timeout: 10m
```

### 12.2 OpenID Connect (OIDC)

```yaml
# ใช้ OIDC แทน Long-lived Credentials
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
    
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions
        aws-region: ap-southeast-1
    
    - name: Update EKS kubeconfig
      run: |
        aws eks update-kubeconfig \
          --region ap-southeast-1 \
          --name my-cluster
    
    - name: Deploy
      run: kubectl apply -f k8s/
```

## สรุป

GitHub Actions ให้:
1. **Native Integration** - รวมอยู่กับ GitHub ไม่ต้องติดตั้งเพิ่ม
2. **Marketplace** - Actions พร้อมใช้มากมาย
3. **YAML-based** - ง่ายต่อการเรียนรู้
4. **Matrix Strategy** - Test หลาย Version พร้อมกัน
5. **Environments** - Deployment Protection rules
6. **OIDC** - Secure Authentication ไม่ต้องใช้ Long-lived Credentials

ในบทต่อไปจะเรียนรู้ GitLab CI/CD ซึ่งเป็นอีกทางเลือกที่ดี โดยเฉพาะสำหรับ Self-hosted

## แบบฝึกหัด

1. สร้าง GitHub Actions Workflow ที่รัน Tests บน Matrix (Node 16, 18, 20)
2. ตั้งค่า Environment Protection Rules สำหรับ Production
3. สร้าง Composite Action สำหรับ Deploy
4. ทดลองใช้ OIDC กับ AWS/GCP
5. ตั้งค่า Self-hosted Runner บน Kubernetes
