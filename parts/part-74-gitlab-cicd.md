# Part 74: GitLab CI/CD สำหรับ Kubernetes

## บทนำ

GitLab CI/CD เป็นระบบ CI/CD ที่รวมอยู่ใน GitLab Platform ให้ความสามารถครบครันตั้งแต่ Source Code Management, CI/CD, Container Registry, จนถึง Kubernetes Integration GitLab เป็นทางเลือกที่ดีสำหรับ Organizations ที่ต้องการ Self-hosted Solution ที่ครบวงจร

## 1. GitLab CI/CD Architecture

### 1.1 Components

```
┌─────────────────────────────────────────────────────────────┐
│                      GitLab Server                          │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────┐   │
│  │   Source     │   │  Container   │   │  Kubernetes   │   │
│  │   Code       │   │  Registry    │   │  Integration  │   │
│  │   (Git)      │   │  (Built-in)  │   │  (GitLab K8s) │   │
│  └──────────────┘   └──────────────┘   └───────────────┘   │
│          │                                                   │
│          ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 CI/CD Pipeline                       │   │
│  │                                                      │   │
│  │  .gitlab-ci.yml                                      │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │  Stages │ │  Jobs   │ │Artifacts│ │ Deploy  │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                    GitLab Runners                           │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────────────────┐   │
│  │  Shared Runners  │   │    Project Runners            │   │
│  │  (GitLab.com)    │   │    (Self-hosted on K8s)       │   │
│  └──────────────────┘   └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 .gitlab-ci.yml Structure

```yaml
# .gitlab-ci.yml - Complete Structure
image: alpine:latest              # Default image

variables:                        # Global variables
  DOCKER_DRIVER: overlay2
  REGISTRY: registry.gitlab.com

stages:                           # Pipeline stages
  - lint
  - test
  - build
  - scan
  - deploy-dev
  - deploy-staging
  - deploy-production

before_script:                    # Run before every job
  - echo "Starting pipeline"

after_script:                     # Run after every job
  - echo "Pipeline complete"

cache:                            # Shared cache
  paths:
    - .cache/

# Jobs
lint:
  stage: lint
  script:
    - golangci-lint run ./...

test:
  stage: test
  script:
    - go test ./...
```

## 2. GitLab Runner บน Kubernetes

### 2.1 ติดตั้ง GitLab Runner ด้วย Helm

```bash
# เพิ่ม GitLab Helm Repository
helm repo add gitlab https://charts.gitlab.io
helm repo update

# สร้าง values.yaml
cat > gitlab-runner-values.yaml << 'EOF'
# GitLab URL
gitlabUrl: https://gitlab.example.com

# Registration Token (จาก GitLab > Settings > CI/CD > Runners)
runnerToken: "your-runner-token"

# Runner configuration
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "gitlab-runner"
        image = "ubuntu:22.04"
        
        # CPU/Memory limits for build pods
        cpu_request = "100m"
        cpu_limit = "2"
        memory_request = "256Mi"
        memory_limit = "2Gi"
        
        # Service account
        service_account = "gitlab-runner"
        
        # Pull policy
        pull_policy = "if-not-present"
        
        # Volumes
        [[runners.kubernetes.volumes.empty_dir]]
          name = "docker-certs"
          mount_path = "/certs/client"
          medium = "Memory"
        
        # Pod labels
        [runners.kubernetes.pod_labels]
          "app" = "gitlab-runner"
        
        # Node selector
        [runners.kubernetes.node_selector]
          "kubernetes.io/os" = "linux"

# Resources for Runner Manager pod
resources:
  limits:
    memory: 256Mi
    cpu: 200m
  requests:
    memory: 128Mi
    cpu: 100m

# RBAC
rbac:
  create: true
  clusterWideAccess: false
  
# Metrics
metrics:
  enabled: true
  portName: metrics
  port: 9252
  serviceMonitor:
    enabled: true

# Replicas
replicas: 2
EOF

# ติดตั้ง
helm install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner \
  --create-namespace \
  -f gitlab-runner-values.yaml
```

### 2.2 RBAC สำหรับ GitLab Runner

```yaml
# gitlab-runner-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-runner
  namespace: gitlab-runner

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner
  namespace: gitlab-runner
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec", "pods/attach", "services", "secrets", "configmaps"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-runner
  namespace: gitlab-runner
subjects:
- kind: ServiceAccount
  name: gitlab-runner
  namespace: gitlab-runner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gitlab-runner
```

### 2.3 Runner Registration

```bash
# ดู Registration Token
# GitLab > Project > Settings > CI/CD > Runners

# ลงทะเบียน Runner แบบ Manual (ถ้าไม่ใช้ Helm)
gitlab-runner register \
  --url "https://gitlab.example.com" \
  --registration-token "your-token" \
  --executor "kubernetes" \
  --kubernetes-namespace "gitlab-runner" \
  --description "kubernetes-runner" \
  --tag-list "kubernetes,linux" \
  --non-interactive
```

## 3. .gitlab-ci.yml Templates

### 3.1 Basic Pipeline

```yaml
# .gitlab-ci.yml
image: golang:1.21-alpine

variables:
  GOPATH: $CI_PROJECT_DIR/.go
  CGO_ENABLED: "0"

stages:
  - lint
  - test
  - build
  - scan
  - deploy

# Cache Go modules
cache:
  paths:
    - .go/pkg/mod/

# Lint Stage
lint:
  stage: lint
  script:
    - go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
    - golangci-lint run ./...
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

# Test Stage
test:
  stage: test
  services:
    - name: postgres:15-alpine
      alias: postgres
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgres://test:test@postgres:5432/testdb?sslmode=disable
  script:
    - go test ./... -v -coverprofile=coverage.out
    - go tool cover -func=coverage.out | tee coverage-summary.txt
    - |
      COVERAGE=$(grep "total:" coverage-summary.txt | awk '{print $3}' | tr -d '%')
      echo "Coverage: $COVERAGE%"
      if (( $(echo "$COVERAGE < 80" | bc -l) )); then
        echo "Coverage below 80%!"
        exit 1
      fi
  coverage: '/^total:\s+\(statements\)\s+(\d+\.\d+)%/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.out
      - coverage-summary.txt
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

# Build Stage
.build-template: &build-template
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build-image:
  <<: *build-template
  script:
    - |
      IMAGE="${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
      docker build \
        --build-arg VERSION=${CI_COMMIT_SHORT_SHA} \
        --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
        -t $IMAGE \
        -t "${CI_REGISTRY_IMAGE}:latest" \
        .
      docker push $IMAGE
      docker push "${CI_REGISTRY_IMAGE}:latest"
      echo "IMAGE_TAG=${CI_COMMIT_SHORT_SHA}" >> build.env
  artifacts:
    reports:
      dotenv: build.env
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_TAG'

# Security Scan
trivy-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --severity HIGH,CRITICAL
        --format json
        --output trivy-report.json
        ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}
  artifacts:
    when: always
    paths:
      - trivy-report.json
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
  needs: ["build-image"]
  allow_failure: false
```

### 3.2 Deploy Jobs

```yaml
# Deploy Templates
.deploy-template: &deploy-template
  image: bitnami/kubectl:1.28
  before_script:
    - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig

# Deploy to Development
deploy-dev:
  <<: *deploy-template
  stage: deploy
  environment:
    name: development
    url: https://dev.example.com
    on_stop: stop-dev
  script:
    - kubectl set image deployment/myapp
        myapp=${CI_REGISTRY_IMAGE}:${IMAGE_TAG}
        -n development
    - kubectl rollout status deployment/myapp
        -n development
        --timeout=5m
    - kubectl get pods -n development -l app=myapp
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_DEV
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
  needs: ["trivy-scan"]

# Stop Development Environment
stop-dev:
  <<: *deploy-template
  stage: deploy
  environment:
    name: development
    action: stop
  script:
    - kubectl scale deployment/myapp --replicas=0 -n development
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_DEV
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      when: manual
  needs: []

# Deploy to Staging
deploy-staging:
  <<: *deploy-template
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/myapp
        myapp=${CI_REGISTRY_IMAGE}:${IMAGE_TAG}
        -n staging
    - kubectl rollout status deployment/myapp
        -n staging
        --timeout=5m
    - |
      # Run smoke tests
      STAGING_URL="https://staging.example.com"
      curl -f $STAGING_URL/health || exit 1
      echo "Staging smoke test passed"
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_STAGING
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  needs: ["trivy-scan"]

# Deploy to Production (Manual)
deploy-production:
  <<: *deploy-template
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - |
      echo "Deploying ${CI_REGISTRY_IMAGE}:${IMAGE_TAG} to production"
      kubectl set image deployment/myapp
        myapp=${CI_REGISTRY_IMAGE}:${IMAGE_TAG}
        -n production
      kubectl rollout status deployment/myapp
        -n production
        --timeout=10m
      
      # Verify deployment
      POD_COUNT=$(kubectl get pods -n production -l app=myapp --field-selector=status.phase=Running | grep -c Running)
      echo "Running pods: $POD_COUNT"
      
      # Health check
      curl -f https://example.com/health || exit 1
      echo "Production deployment successful!"
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_PROD
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
      allow_failure: false
  needs: ["deploy-staging"]
```

## 4. Advanced .gitlab-ci.yml Features

### 4.1 Include Templates

```yaml
# .gitlab-ci.yml
include:
  # GitLab built-in templates
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  
  # Local files
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/deploy.yml'
  
  # Remote files
  - remote: 'https://example.com/ci-templates/deploy.yml'
  
  # Another project
  - project: 'my-group/ci-templates'
    ref: main
    file: '/templates/go-pipeline.yml'
```

### 4.2 Dynamic Pipelines

```yaml
# Generate pipeline dynamically
generate-pipeline:
  stage: .pre
  script:
    - |
      cat > generated-pipeline.yml << YAML
      deploy-service-a:
        stage: deploy
        script:
          - echo "Deploy Service A"
      deploy-service-b:
        stage: deploy
        script:
          - echo "Deploy Service B"
      YAML
  artifacts:
    paths:
      - generated-pipeline.yml

dynamic-deploy:
  stage: deploy
  trigger:
    include:
      - artifact: generated-pipeline.yml
        job: generate-pipeline
    strategy: depend
```

### 4.3 Multi-project Pipelines

```yaml
# Trigger downstream pipeline
deploy-downstream:
  stage: deploy
  trigger:
    project: mygroup/deploy-repo
    branch: main
    strategy: depend
  variables:
    IMAGE_TAG: $CI_COMMIT_SHORT_SHA
    ENVIRONMENT: production
```

### 4.4 Parallel Matrix

```yaml
test:
  stage: test
  parallel:
    matrix:
      - RUNNER_TAG: [linux, windows]
        GO_VERSION: ["1.20", "1.21"]
  tags:
    - $RUNNER_TAG
  image: golang:${GO_VERSION}
  script:
    - go test ./...
```

### 4.5 DAG (Directed Acyclic Graph)

```yaml
stages:
  - build
  - test
  - deploy

build-frontend:
  stage: build
  script: npm run build

build-backend:
  stage: build
  script: go build ./...

test-frontend:
  stage: test
  needs: ["build-frontend"]  # DAG: รอแค่ build-frontend
  script: npm test

test-backend:
  stage: test
  needs: ["build-backend"]   # DAG: รอแค่ build-backend
  script: go test ./...

deploy:
  stage: deploy
  needs: ["test-frontend", "test-backend"]  # รอทั้งคู่
  script: kubectl apply -f k8s/
```

## 5. GitLab Registry Integration

### 5.1 ใช้ Built-in Registry

```yaml
# GitLab Container Registry
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    # Login to GitLab Registry อัตโนมัติ
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      IMAGE="${CI_REGISTRY_IMAGE}/${CI_PROJECT_NAME}:${CI_COMMIT_SHORT_SHA}"
      docker build -t $IMAGE .
      docker push $IMAGE
      
      # Tag latest บน main branch
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $IMAGE ${CI_REGISTRY_IMAGE}/${CI_PROJECT_NAME}:latest
        docker push ${CI_REGISTRY_IMAGE}/${CI_PROJECT_NAME}:latest
      fi
```

### 5.2 ใช้ Kaniko (ไม่ต้องการ Docker-in-Docker)

```yaml
build-with-kaniko:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - |
      mkdir -p /kaniko/.docker
      echo "{\"auths\":{\"${CI_REGISTRY}\":{\"auth\":\"$(echo -n ${CI_REGISTRY_USER}:${CI_REGISTRY_PASSWORD} | base64)\"}}}" > /kaniko/.docker/config.json
      
      /kaniko/executor \
        --context "${CI_PROJECT_DIR}" \
        --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
        --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}" \
        --destination "${CI_REGISTRY_IMAGE}:latest" \
        --cache=true \
        --cache-copy-layers
```

## 6. GitLab Environment Management

### 6.1 Environment Definition

```yaml
deploy-review:
  stage: deploy
  script:
    - echo "Deploy to review environment"
    - kubectl create namespace review-${CI_MERGE_REQUEST_IID} || true
    - kubectl apply -f k8s/ -n review-${CI_MERGE_REQUEST_IID}
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://review-$CI_MERGE_REQUEST_IID.example.com
    on_stop: stop-review
    auto_stop_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review:
  stage: deploy
  script:
    - kubectl delete namespace review-${CI_MERGE_REQUEST_IID} --ignore-not-found
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
```

### 6.2 Protected Environments

```yaml
# Protected Environment Variables
# ไปที่ Settings > CI/CD > Variables
# ตั้ง Variable: KUBECONFIG_PROD
# - Protected: Yes
# - Environment scope: production

deploy-production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - echo "$KUBECONFIG_PROD" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl apply -f k8s/production/
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

## 7. GitLab Security Features

### 7.1 SAST (Static Application Security Testing)

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml

variables:
  SAST_EXCLUDED_PATHS: "vendor, test"
  SAST_GOSEC_LEVEL: 2

gosec-sast:
  extends: .sast-analyzer
  variables:
    SAST_ANALYZER_IMAGE: registry.gitlab.com/security-products/gosec:4
```

### 7.2 Container Scanning

```yaml
include:
  - template: Security/Container-Scanning.gitlab-ci.yml

container_scanning:
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    CS_SEVERITY_THRESHOLD: HIGH
    CS_ANALYZER_IMAGE: registry.gitlab.com/security-products/container-scanning:6
```

### 7.3 Dependency Scanning

```yaml
include:
  - template: Security/Dependency-Scanning.gitlab-ci.yml

dependency_scanning:
  variables:
    DS_EXCLUDED_PATHS: "spec, test, tests, tmp"
```

## 8. Workshop: GitLab Pipeline สำหรับ Python Application

### Step 1: สร้าง Python Application

```bash
mkdir gitlab-pipeline-workshop
cd gitlab-pipeline-workshop

# สร้าง Flask Application
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
from datetime import datetime

app = Flask(__name__)

VERSION = os.getenv('APP_VERSION', 'development')
ENVIRONMENT = os.getenv('ENVIRONMENT', 'local')

@app.route('/')
def index():
    return jsonify({
        'message': 'Hello from GitLab CI/CD Workshop!',
        'version': VERSION,
        'environment': ENVIRONMENT,
        'timestamp': datetime.utcnow().isoformat()
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/ready')
def ready():
    return jsonify({'status': 'ready'})

if __name__ == '__main__':
    port = int(os.getenv('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
EOF

# สร้าง Tests
cat > test_app.py << 'EOF'
import pytest
import json
from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_index(client):
    response = client.get('/')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'message' in data
    assert 'GitLab CI/CD Workshop' in data['message']

def test_health(client):
    response = client.get('/health')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['status'] == 'healthy'

def test_ready(client):
    response = client.get('/ready')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['status'] == 'ready'
EOF

# สร้าง requirements.txt
cat > requirements.txt << 'EOF'
flask==3.0.0
EOF

cat > requirements-dev.txt << 'EOF'
-r requirements.txt
pytest==7.4.3
pytest-cov==4.1.0
flake8==6.1.0
bandit==1.7.5
EOF
```

### Step 2: สร้าง Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim AS base

# Security: Non-root user
RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -s /bin/bash appuser

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip install gunicorn

# Copy application
COPY --chown=appuser:appgroup app.py .

# Switch to non-root user
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:5000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

### Step 3: สร้าง Kubernetes Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab-workshop

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-workshop
  namespace: gitlab-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gitlab-workshop
  template:
    metadata:
      labels:
        app: gitlab-workshop
    spec:
      containers:
      - name: app
        image: registry.gitlab.com/mygroup/gitlab-workshop:latest
        ports:
        - containerPort: 5000
        env:
        - name: PORT
          value: "5000"
        - name: ENVIRONMENT
          value: "kubernetes"
        - name: APP_VERSION
          value: "PLACEHOLDER"
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
            port: 5000
          initialDelaySeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 5
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gitlab-workshop
  namespace: gitlab-workshop
spec:
  selector:
    app: gitlab-workshop
  ports:
  - port: 80
    targetPort: 5000

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab-workshop
  namespace: gitlab-workshop
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
            name: gitlab-workshop
            port:
              number: 80
```

### Step 4: สร้าง .gitlab-ci.yml

```yaml
# .gitlab-ci.yml

# ──────────────────────────────────────────
# Default Settings
# ──────────────────────────────────────────
default:
  image: python:3.11-slim
  
  # Retry on runner failure
  retry:
    max: 2
    when: runner_system_failure

# ──────────────────────────────────────────
# Variables
# ──────────────────────────────────────────
variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  IMAGE_LATEST: "$CI_REGISTRY_IMAGE:latest"

# ──────────────────────────────────────────
# Stages
# ──────────────────────────────────────────
stages:
  - validate
  - test
  - build
  - security
  - deploy-dev
  - integration-test
  - deploy-staging
  - deploy-production
  - notify

# ──────────────────────────────────────────
# Cache
# ──────────────────────────────────────────
cache:
  key: $CI_COMMIT_REF_SLUG
  paths:
    - .cache/pip

# ──────────────────────────────────────────
# VALIDATE Stage
# ──────────────────────────────────────────
lint:
  stage: validate
  script:
    - pip install flake8 --quiet
    - flake8 app.py --max-line-length=120 --ignore=E501
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

validate-yaml:
  stage: validate
  image: alpine:latest
  script:
    - apk add --no-cache python3 py3-yaml
    - |
      for f in k8s/*.yaml; do
        python3 -c "import yaml; yaml.safe_load_all(open('$f'))" && echo "$f: OK"
      done
  rules:
    - changes:
        - k8s/**/*.yaml

# ──────────────────────────────────────────
# TEST Stage
# ──────────────────────────────────────────
unit-test:
  stage: test
  script:
    - pip install -r requirements-dev.txt --quiet
    - pytest test_app.py -v \
        --cov=app \
        --cov-report=term-missing \
        --cov-report=xml:coverage.xml \
        --junit-xml=junit.xml
    - |
      COVERAGE=$(python3 -c "
      import xml.etree.ElementTree as ET
      tree = ET.parse('coverage.xml')
      root = tree.getroot()
      print(float(root.attrib['line-rate']) * 100)
      ")
      echo "Coverage: $COVERAGE%"
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: junit.xml
    paths:
      - coverage.xml
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# ──────────────────────────────────────────
# BUILD Stage
# ──────────────────────────────────────────
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --build-arg APP_VERSION="${CI_COMMIT_SHORT_SHA}" \
        --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --label "org.opencontainers.image.revision=${CI_COMMIT_SHA}" \
        --label "org.opencontainers.image.source=${CI_PROJECT_URL}" \
        -t $IMAGE \
        -t $IMAGE_LATEST \
        .
      
      docker push $IMAGE
      
      if [ "$CI_COMMIT_BRANCH" == "main" ] || [ "$CI_COMMIT_TAG" != "" ]; then
        docker push $IMAGE_LATEST
      fi
      
      echo "Built and pushed: $IMAGE"
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
    - if: '$CI_COMMIT_TAG'
  needs: ["unit-test"]

# ──────────────────────────────────────────
# SECURITY Stage
# ──────────────────────────────────────────
trivy:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  variables:
    TRIVY_NO_PROGRESS: "true"
    TRIVY_CACHE_DIR: ".trivycache/"
  cache:
    paths:
      - .trivycache/
  script:
    - trivy image
        --exit-code 0
        --format template
        --template "@/contrib/gitlab.tpl"
        -o gl-container-scanning-report.json
        $IMAGE
    - trivy image
        --exit-code 1
        --severity CRITICAL
        $IMAGE
  artifacts:
    when: always
    reports:
      container_scanning: gl-container-scanning-report.json
    paths:
      - gl-container-scanning-report.json
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
  needs: ["build-image"]

bandit-sast:
  stage: security
  script:
    - pip install bandit --quiet
    - bandit -r app.py -f json -o bandit-report.json || true
    - bandit -r app.py --severity-level high || exit 1
  artifacts:
    when: always
    paths:
      - bandit-report.json
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# ──────────────────────────────────────────
# DEPLOY-DEV Stage
# ──────────────────────────────────────────
.deploy-k8s: &deploy-k8s
  image: bitnami/kubectl:1.28
  before_script:
    - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl version --client

deploy-dev:
  <<: *deploy-k8s
  stage: deploy-dev
  environment:
    name: development
    url: https://dev.example.com
    on_stop: stop-dev
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_DEV
  script:
    - kubectl apply -f k8s/namespace.yaml
    - |
      # Update image
      sed -i "s|image: .*gitlab-workshop:.*|image: $IMAGE|g" k8s/deployment.yaml
      sed -i "s|value: \"PLACEHOLDER\"|value: \"$CI_COMMIT_SHORT_SHA\"|g" k8s/deployment.yaml
    - kubectl apply -f k8s/ -n gitlab-workshop
    - kubectl rollout status deployment/gitlab-workshop -n gitlab-workshop --timeout=5m
    - kubectl get pods -n gitlab-workshop -l app=gitlab-workshop
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
  needs: ["trivy"]

stop-dev:
  <<: *deploy-k8s
  stage: deploy-dev
  environment:
    name: development
    action: stop
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_DEV
  script:
    - kubectl delete namespace gitlab-workshop --ignore-not-found
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      when: manual
  needs: []

# ──────────────────────────────────────────
# INTEGRATION TEST Stage
# ──────────────────────────────────────────
integration-test:
  stage: integration-test
  script:
    - pip install requests --quiet
    - |
      python3 << 'PYTHON'
      import requests
      import sys
      
      base_url = "https://dev.example.com"
      
      # Test health
      r = requests.get(f"{base_url}/health")
      assert r.status_code == 200
      assert r.json()['status'] == 'healthy'
      print("Health check passed!")
      
      # Test main endpoint
      r = requests.get(f"{base_url}/")
      assert r.status_code == 200
      assert 'GitLab CI/CD Workshop' in r.json()['message']
      print("Main endpoint test passed!")
      
      print("All integration tests passed!")
      PYTHON
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
  needs: ["deploy-dev"]

# ──────────────────────────────────────────
# DEPLOY-STAGING Stage
# ──────────────────────────────────────────
deploy-staging:
  <<: *deploy-k8s
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.example.com
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_STAGING
  script:
    - |
      kubectl set image deployment/gitlab-workshop \
        app=$IMAGE \
        -n staging
      kubectl rollout status deployment/gitlab-workshop \
        -n staging \
        --timeout=5m
      
      # Smoke test
      sleep 10
      curl -f https://staging.example.com/health || exit 1
      echo "Staging deployment successful!"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  needs: ["trivy"]

# ──────────────────────────────────────────
# DEPLOY-PRODUCTION Stage
# ──────────────────────────────────────────
deploy-production:
  <<: *deploy-k8s
  stage: deploy-production
  environment:
    name: production
    url: https://example.com
  variables:
    KUBECONFIG_DATA: $KUBECONFIG_PROD
  script:
    - |
      echo "=== Production Deployment ==="
      echo "Image: $IMAGE"
      echo "Version: $CI_COMMIT_SHORT_SHA"
      echo "Deployed by: $GITLAB_USER_LOGIN"
      
      # Blue-Green Deployment
      kubectl set image deployment/gitlab-workshop-green \
        app=$IMAGE \
        -n production
      
      kubectl rollout status deployment/gitlab-workshop-green \
        -n production \
        --timeout=10m
      
      # Verify green deployment
      curl -f https://green.example.com/health || exit 1
      
      # Switch traffic
      kubectl patch service gitlab-workshop \
        -n production \
        -p '{"spec":{"selector":{"track":"green"}}}'
      
      # Verify production
      curl -f https://example.com/health || exit 1
      echo "Production deployment successful!"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
      allow_failure: false
  needs: ["deploy-staging"]

# ──────────────────────────────────────────
# NOTIFY Stage
# ──────────────────────────────────────────
notify-success:
  stage: notify
  image: alpine:latest
  when: on_success
  script:
    - apk add --no-cache curl
    - |
      curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-type: application/json' \
        --data "{
          \"text\": \"✅ *Deployment Successful*\",
          \"attachments\": [{
            \"color\": \"good\",
            \"fields\": [
              {\"title\": \"Project\", \"value\": \"$CI_PROJECT_NAME\", \"short\": true},
              {\"title\": \"Branch\", \"value\": \"$CI_COMMIT_BRANCH\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"$CI_COMMIT_SHORT_SHA\", \"short\": true},
              {\"title\": \"Author\", \"value\": \"$GITLAB_USER_LOGIN\", \"short\": true}
            ]
          }]
        }"
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

notify-failure:
  stage: notify
  image: alpine:latest
  when: on_failure
  script:
    - apk add --no-cache curl
    - |
      curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-type: application/json' \
        --data "{
          \"text\": \"❌ *Deployment Failed*\",
          \"attachments\": [{
            \"color\": \"danger\",
            \"fields\": [
              {\"title\": \"Project\", \"value\": \"$CI_PROJECT_NAME\", \"short\": true},
              {\"title\": \"Branch\", \"value\": \"$CI_COMMIT_BRANCH\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"$CI_COMMIT_SHORT_SHA\", \"short\": true},
              {\"title\": \"Pipeline\", \"value\": \"$CI_PIPELINE_URL\", \"short\": false}
            ]
          }]
        }"
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
```

### Step 5: ตั้งค่า GitLab Project

```bash
# 1. ตั้งค่า CI/CD Variables
# ไปที่ Settings > CI/CD > Variables

# KUBECONFIG_DEV = <base64-encoded kubeconfig>
# KUBECONFIG_STAGING = <base64-encoded kubeconfig>
# KUBECONFIG_PROD = <base64-encoded kubeconfig>
# SLACK_WEBHOOK = <slack webhook url>

# 2. Protected Variables สำหรับ Production
# KUBECONFIG_PROD: Protected, Masked

# 3. ตั้งค่า Protected Branches
# Settings > Repository > Protected Branches
# main: Allowed to merge = Developer + Maintainer
#       Allowed to push = No one (force push)

# 4. ตั้งค่า Protected Environments
# Settings > CI/CD > Protected environments
# production: Allowed to deploy = Maintainer

# 5. Push code
git init
git remote add origin https://gitlab.com/mygroup/gitlab-workshop.git
git add .
git commit -m "Initial commit: GitLab CI/CD Workshop"
git push -u origin main
```

### Step 6: ตรวจสอบ Pipeline

```bash
# ดู Pipeline Status ผ่าน GitLab CLI
glab ci view

# ดู Job Logs
glab ci view --job unit-test

# ดู Recent Pipelines
glab ci list

# Trigger Pipeline แบบ Manual
glab ci run --branch main

# ดู Environment Status
glab env list
```

## 9. GitLab Kubernetes Integration

### 9.1 GitLab Agent for Kubernetes (KAS)

```yaml
# ติดตั้ง GitLab Agent
# 1. สร้าง Agent ใน GitLab UI:
# Infrastructure > Kubernetes clusters > Connect a cluster

# 2. สร้าง Config ใน Repository
# .gitlab/agents/my-agent/config.yaml
gitops:
  manifest_projects:
  - id: mygroup/gitops-repo
    default_namespace: production
    paths:
    - glob: 'apps/myapp/**/*.yaml'
    reconcile_timeout: 3600s

ci_access:
  projects:
  - id: mygroup/myapp
  groups:
  - id: mygroup
    environments:
    - production
    - staging
```

```bash
# ติดตั้ง Agent ใน Cluster
helm repo add gitlab https://charts.gitlab.io
helm repo update

helm install gitlab-agent gitlab/gitlab-agent \
  --namespace gitlab-agent \
  --create-namespace \
  --set config.token=<agent-token> \
  --set config.kasAddress=wss://kas.gitlab.example.com
```

### 9.2 ใช้ Agent ใน CI/CD

```yaml
# ใช้ GitLab Agent เพื่อ Deploy
deploy:
  image: bitnami/kubectl:1.28
  script:
    - |
      # ใช้ kubectl context จาก GitLab Agent
      kubectl config use-context mygroup/myapp:my-agent
      kubectl apply -f k8s/
      kubectl rollout status deployment/myapp
```

## 10. GitLab CI/CD Best Practices

### 10.1 Optimization

```yaml
# ใช้ needs แทน stages เพื่อเร็วขึ้น
test-a:
  stage: test
  script: ./test-a.sh
  needs: ["lint"]  # รอแค่ lint ไม่ต้องรอ job อื่น

test-b:
  stage: test
  script: ./test-b.sh
  needs: ["lint"]  # รันพร้อมกับ test-a

deploy:
  stage: deploy
  script: ./deploy.sh
  needs: ["test-a", "test-b"]  # รอทั้งคู่
```

### 10.2 DRY (Don't Repeat Yourself)

```yaml
# ใช้ Extends
.base-deploy:
  image: bitnami/kubectl:1.28
  before_script:
    - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  script:
    - kubectl apply -f k8s/ -n $NAMESPACE
    - kubectl rollout status deployment/myapp -n $NAMESPACE --timeout=5m

deploy-dev:
  extends: .base-deploy
  variables:
    NAMESPACE: development
    KUBECONFIG_DATA: $KUBECONFIG_DEV

deploy-prod:
  extends: .base-deploy
  variables:
    NAMESPACE: production
    KUBECONFIG_DATA: $KUBECONFIG_PROD
```

## สรุป

GitLab CI/CD ให้:
1. **All-in-one Platform** - Code, CI/CD, Registry ในที่เดียว
2. **Powerful YAML** - Feature-rich .gitlab-ci.yml
3. **Built-in Security** - SAST, DAST, Container Scanning
4. **Kubernetes Native** - GitLab Agent for K8s
5. **Environments** - Built-in environment management

ในบทต่อไปจะเรียนรู้ ArgoCD ซึ่งเป็น GitOps Tool ยอดนิยม

## แบบฝึกหัด

1. ติดตั้ง GitLab Runner บน Kubernetes ด้วย Helm
2. สร้าง .gitlab-ci.yml ที่มีทุก Stage
3. ทดลองใช้ Review Environments
4. ตั้งค่า Protected Environments สำหรับ Production
5. สร้าง Multi-project Pipeline
