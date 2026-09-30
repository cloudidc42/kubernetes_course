# Part 60: Image Security

## บทนำ

Container Image Security เป็นส่วนสำคัญของ Kubernetes security เพราะทุกอย่างที่รันใน cluster มาจาก container images หากมีช่องโหว่ใน image หรือใช้ image ที่ไม่น่าเชื่อถือ อาจเป็นประตูให้ผู้โจมตีเข้ามาได้

ในบทนี้เราจะเรียนรู้:
1. Container Image Security พื้นฐาน
2. Image Scanning ด้วย Trivy และ Clair
3. Private Registry
4. Image Pull Secrets
5. Admission Controller สำหรับ Image Policy

---

## 60.1 Container Image Security พื้นฐาน

### ความเสี่ยงของ Container Images

```
ความเสี่ยงหลัก:
1. Vulnerable packages - OS packages หรือ libraries ที่มี CVE
2. Malware - images ที่ถูก inject malicious code
3. Exposed secrets - credentials ถูก bake ใน image layers
4. Insecure configurations - root user, writable filesystem
5. Untrusted sources - pull images จาก public registry โดยไม่ verify
6. Image tampering - image ถูกแก้ไขหลัง scan
```

### Best Practices สำหรับ Dockerfile

```dockerfile
# ไม่ดี - image ใหญ่ มีหลาย vulnerabilities
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl wget git python3
COPY . /app
RUN pip3 install -r requirements.txt
CMD ["python3", "app.py"]

# ดีกว่า - ใช้ minimal base image
FROM python:3.11-slim-bookworm
WORKDIR /app

# สร้าง non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup -u 1001 appuser

# Copy requirements แยกเพื่อ leverage cache
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip check

# Copy application code
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER 1001

# Document port
EXPOSE 8080

# ใช้ exec form เพื่อ proper signal handling
CMD ["python3", "-m", "uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Multi-stage Build

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app -ldflags="-w -s" ./cmd/app

# Stage 2: Run (minimal image)
FROM scratch   # empty image - ปลอดภัยสูงสุด
# หรือ FROM alpine:3.18 ถ้าต้องการ shell
COPY --from=builder /build/app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 65534:65534   # nobody user
ENTRYPOINT ["/app"]
```

### Distroless Images

```dockerfile
# Google Distroless: ไม่มี shell, package manager, utilities
# ลด attack surface อย่างมาก

FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY . .
RUN go build -o app ./cmd/app

FROM gcr.io/distroless/static-debian11
COPY --from=builder /build/app /
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

---

## 60.2 Image Scanning ด้วย Trivy

Trivy เป็น comprehensive security scanner ที่สามารถ scan:
- Container images
- Filesystems
- Git repositories
- Kubernetes cluster
- Configuration files (IaC)

### ติดตั้ง Trivy

```bash
# macOS
brew install aquasecurity/trivy/trivy

# Linux
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.48.0

# Debian/Ubuntu
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy

# ตรวจสอบ
trivy --version
```

### Scan Container Images

```bash
# Scan image พื้นฐาน
trivy image nginx:1.25

# Scan และแสดงเฉพาะ HIGH และ CRITICAL
trivy image --severity HIGH,CRITICAL nginx:1.25

# Scan พร้อม output เป็น JSON
trivy image --format json --output nginx-scan.json nginx:1.25

# Scan และ exit code 1 ถ้ามี CRITICAL
trivy image --exit-code 1 --severity CRITICAL nginx:1.25
echo "Exit code: $?"  # 0=clean, 1=found vulnerabilities

# Scan image ที่ยังไม่ถูก push
docker build -t myapp:latest .
trivy image myapp:latest

# Scan โดยไม่ดึงจาก internet (offline mode)
trivy image --download-db-only
trivy image --skip-update myapp:latest
```

### Trivy Scan ตัวอย่างผลลัพธ์

```
nginx:1.25 (debian 12.4)
========================
Total: 89 (HIGH: 23, CRITICAL: 3)

┌──────────────────────┬────────────────┬──────────┬─────────────────────┬──────────────────────┐
│       Library        │  Vulnerability │ Severity │   Installed Version │     Fixed Version    │
├──────────────────────┼────────────────┼──────────┼─────────────────────┼──────────────────────┤
│ libssl3              │ CVE-2024-0727  │ HIGH     │ 3.0.11-1~deb12u2    │ 3.0.12-1~deb12u1     │
│ openssl              │ CVE-2023-5678  │ CRITICAL │ 3.0.11-1~deb12u2    │ 3.0.12-1~deb12u1     │
└──────────────────────┴────────────────┴──────────┴─────────────────────┴──────────────────────┘
```

### Scan Kubernetes Cluster

```bash
# Scan running cluster
trivy k8s --report summary cluster

# Scan specific namespace
trivy k8s --namespace production --report all cluster

# Scan deployments
trivy k8s --report summary deployment/nginx -n production

# ดู misconfigurations
trivy k8s --scanners misconfig --namespace production cluster
```

### Scan Kubernetes Manifests (IaC)

```bash
# Scan Kubernetes YAML files
trivy config ./kubernetes/

# Scan Helm charts
trivy config ./helm-chart/

# Scan Dockerfile
trivy config ./Dockerfile

# ตรวจสอบ misconfigurations
trivy config --severity HIGH,CRITICAL ./kubernetes/
```

---

## 60.3 Image Scanning ด้วย Clair

Clair เป็น open source vulnerability scanner สำหรับ container images โดย Quay.io

### ติดตั้ง Clair

```yaml
# clair-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clair-config
  namespace: clair
data:
  config.yaml: |
    clair:
      database:
        type: pgsql
        options:
          source: postgresql://clair:clair@postgres/clair?sslmode=disable
          cachesize: 16384
      api:
        healthPort: 6061
        port: 6060
        timeout: 900s
      updater:
        interval: 2h
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clair
  namespace: clair
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clair
  template:
    metadata:
      labels:
        app: clair
    spec:
      containers:
      - name: clair
        image: quay.io/projectquay/clair:4.7.0
        ports:
        - containerPort: 6060
          name: http
        - containerPort: 6061
          name: health
        volumeMounts:
        - name: config
          mountPath: /etc/clair
      volumes:
      - name: config
        configMap:
          name: clair-config
```

---

## 60.4 Private Registry

Private Registry ใช้เพื่อ:
- ควบคุม images ที่ใช้ใน organization
- ประหยัด bandwidth (pull จาก internal)
- Scan images ก่อน deployment
- Enforce image policies

### ประเภทของ Private Registry

```
Self-hosted:
- Harbor (แนะนำสำหรับ Kubernetes)
- Docker Registry
- Nexus Repository

Cloud-managed:
- AWS ECR (Elastic Container Registry)
- GCR (Google Container Registry)
- ACR (Azure Container Registry)
- Quay.io
```

### ติดตั้ง Harbor

```bash
# Harbor เป็น enterprise-grade container registry
# รองรับ: multi-tenancy, vulnerability scanning, image signing, RBAC

# ติดตั้งด้วย Helm
helm repo add harbor https://helm.getharbor.io
helm repo update

helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  --set expose.type=nodePort \
  --set expose.tls.enabled=false \
  --set externalURL=http://harbor.example.com \
  --set harborAdminPassword=Harbor12345 \
  --wait

kubectl get pods -n harbor
```

### Configure Harbor

```bash
# เข้า Harbor UI: http://harbor.example.com
# Login: admin / Harbor12345

# หรือใช้ API
HARBOR_URL="http://harbor.example.com"
HARBOR_USER="admin"
HARBOR_PASS="Harbor12345"

# สร้าง project
curl -X POST "$HARBOR_URL/api/v2.0/projects" \
  -u "$HARBOR_USER:$HARBOR_PASS" \
  -H "Content-Type: application/json" \
  -d '{
    "project_name": "production",
    "public": false,
    "metadata": {
      "enable_content_trust": "true",
      "prevent_vul": "true",
      "severity": "critical",
      "auto_scan": "true"
    }
  }'

# Enable Trivy scanner
curl -X POST "$HARBOR_URL/api/v2.0/projects/1/scanner" \
  -u "$HARBOR_USER:$HARBOR_PASS" \
  -H "Content-Type: application/json" \
  -d '{"name": "Trivy"}'
```

### Push Images ไปยัง Harbor

```bash
# Login ไป Harbor
docker login harbor.example.com -u admin -p Harbor12345

# Tag image
docker tag myapp:1.0.0 harbor.example.com/production/myapp:1.0.0

# Push
docker push harbor.example.com/production/myapp:1.0.0

# Pull (ต้อง authenticated)
docker pull harbor.example.com/production/myapp:1.0.0
```

---

## 60.5 Image Pull Secrets

Image Pull Secrets เก็บ credentials สำหรับ pull images จาก private registry

### สร้าง imagePullSecret

```bash
# สร้าง Secret สำหรับ Docker registry
kubectl create secret docker-registry my-registry-secret \
  --docker-server=harbor.example.com \
  --docker-username=myapp-user \
  --docker-password=mypassword \
  --docker-email=myapp@example.com \
  --namespace=production

# ดูข้อมูล
kubectl get secret my-registry-secret -n production -o yaml

# Decode config.json
kubectl get secret my-registry-secret -n production \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.'
```

### ใช้ imagePullSecrets ใน Pod

```yaml
# Method 1: ใน Pod spec
apiVersion: v1
kind: Pod
metadata:
  name: private-app
  namespace: production
spec:
  imagePullSecrets:
  - name: my-registry-secret   # ระบุ secret name
  containers:
  - name: app
    image: harbor.example.com/production/myapp:1.0.0
```

```yaml
# Method 2: ใน Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      imagePullSecrets:
      - name: my-registry-secret
      - name: gcr-secret        # หลาย registries
      containers:
      - name: app
        image: harbor.example.com/production/myapp:1.0.0
```

### Attach ไปยัง ServiceAccount

```bash
# แทนที่จะระบุใน ทุก Pod ให้ attach กับ ServiceAccount
kubectl patch serviceaccount default \
  -n production \
  -p '{"imagePullSecrets": [{"name": "my-registry-secret"}]}'

# ตรวจสอบ
kubectl get serviceaccount default -n production -o yaml
# จะเห็น imagePullSecrets ใน SA
```

```yaml
# หรือสร้าง ServiceAccount พร้อม imagePullSecrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
imagePullSecrets:
- name: my-registry-secret
---
# Pods ที่ใช้ SA นี้จะ inherit imagePullSecrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: app-sa  # ใช้ SA ที่มี imagePullSecrets
      containers:
      - name: app
        image: harbor.example.com/production/myapp:1.0.0
```

### ECR Image Pull Secret (AWS)

```bash
# AWS ECR token หมดอายุทุก 12 ชั่วโมง
# ต้อง refresh อัตโนมัติ

# Method 1: CronJob refresh
kubectl create secret docker-registry ecr-secret \
  --docker-server=123456789.dkr.ecr.ap-southeast-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-southeast-1) \
  -n production
```

```yaml
# CronJob สำหรับ refresh ECR token
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-token-refresh
  namespace: production
spec:
  schedule: "0 */11 * * *"   # ทุก 11 ชั่วโมง
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ecr-refresh-sa   # SA ที่มี IRSA สำหรับ ECR
          containers:
          - name: refresh
            image: amazon/aws-cli:latest
            command: ["/bin/sh", "-c"]
            args:
            - |
              # Get ECR token
              TOKEN=$(aws ecr get-login-password --region ap-southeast-1)
              
              # Update secret
              kubectl create secret docker-registry ecr-secret \
                --docker-server=123456789.dkr.ecr.ap-southeast-1.amazonaws.com \
                --docker-username=AWS \
                --docker-password="$TOKEN" \
                -n production \
                --dry-run=client -o yaml | \
              kubectl apply -f -
              
              echo "ECR token refreshed!"
          restartPolicy: OnFailure
```

---

## 60.6 Admission Controller สำหรับ Image Policy

Admission Controllers ตรวจสอบและ enforce policies เมื่อ resources ถูกสร้างหรืออัพเดต

### OPA Gatekeeper สำหรับ Image Policy

```bash
# ติดตั้ง OPA Gatekeeper
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --wait

kubectl get pods -n gatekeeper-system
```

### สร้าง Image Policy ด้วย Gatekeeper

```yaml
# ConstraintTemplate: กำหนด Policy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredregistry
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredRegistry
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredregistry
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not starts_with_allowed_registry(container.image)
        msg := sprintf("Container '%v' uses image '%v' from disallowed registry. Allowed: %v",
          [container.name, container.image, input.parameters.registries])
      }
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.initContainers[_]
        not starts_with_allowed_registry(container.image)
        msg := sprintf("InitContainer '%v' uses image '%v' from disallowed registry. Allowed: %v",
          [container.name, container.image, input.parameters.registries])
      }
      
      starts_with_allowed_registry(image) {
        allowed := input.parameters.registries[_]
        startswith(image, allowed)
      }
---
# Constraint: enforce policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredRegistry
metadata:
  name: require-approved-registries
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production", "staging"]
    excludedNamespaces: ["kube-system", "gatekeeper-system"]
  parameters:
    registries:
    - "harbor.example.com/production/"
    - "harbor.example.com/staging/"
    - "gcr.io/my-project/"
    # ไม่อนุญาต docker.io, ghcr.io ที่ไม่ได้ verified
```

### ทดสอบ Image Policy

```bash
# Deploy pod จาก disallowed registry
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: unauthorized-pod
  namespace: production
spec:
  containers:
  - name: app
    image: docker.io/nginx:latest   # ไม่อยู่ใน allowed registries
EOF
# ควรเห็น error!

# Deploy pod จาก allowed registry
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: authorized-pod
  namespace: production
spec:
  containers:
  - name: app
    image: harbor.example.com/production/nginx:1.25   # allowed
EOF
# ผ่าน
```

### Cosign: Image Signing และ Verification

```bash
# ติดตั้ง cosign
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key harbor.example.com/production/myapp:1.0.0

# Verify signature
cosign verify --key cosign.pub harbor.example.com/production/myapp:1.0.0

# Attach SBOM (Software Bill of Materials)
syft harbor.example.com/production/myapp:1.0.0 -o spdx-json > sbom.json
cosign attach sbom --sbom sbom.json harbor.example.com/production/myapp:1.0.0
```

### Policy สำหรับ Signed Images เท่านั้น

```yaml
# ConstraintTemplate สำหรับ verify cosign signatures
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sverifysignature
spec:
  crd:
    spec:
      names:
        kind: K8sVerifySignature
      validation:
        openAPIV3Schema:
          type: object
          properties:
            publicKey:
              type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sverifysignature
      
      # Note: ใน production จะต้องใช้ cosign library จริงๆ
      # นี่เป็นตัวอย่าง simplified
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not image_is_signed(container.image)
        msg := sprintf("Container image '%v' must be signed", [container.image])
      }
      
      image_is_signed(image) {
        # ตรวจสอบ annotation ว่ามีการ sign
        annotations := input.review.object.metadata.annotations
        annotations["cosign.sigstore.dev/verified"] == "true"
      }
```

---

## 60.7 Workshop: Secure Image Pipeline

### สถานการณ์

สร้าง secure image pipeline ที่:
1. Build image
2. Scan ด้วย Trivy
3. Sign ด้วย Cosign
4. Push ไป Harbor
5. Deploy พร้อม Policy enforcement

### Step 1: Setup

```bash
kubectl create namespace workshop-images
kubectl config set-context --current --namespace=workshop-images
```

### Step 2: สร้าง Image Pull Secret

```bash
# สำหรับ workshop ใช้ Docker Hub หรือ local registry

# สร้าง secret สำหรับ Docker Hub
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=docker.io \
  --docker-username=YOUR_DOCKERHUB_USERNAME \
  --docker-password=YOUR_DOCKERHUB_TOKEN \
  --docker-email=YOUR_EMAIL \
  -n workshop-images

# ตรวจสอบ
kubectl get secret dockerhub-secret -n workshop-images
kubectl get secret dockerhub-secret -n workshop-images -o yaml
```

### Step 3: สร้าง Deployment พร้อม Image Pull Secret

```yaml
# workshop-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: workshop-images
  labels:
    app: workshop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-app
  template:
    metadata:
      labels:
        app: workshop-app
    spec:
      # Image pull secret
      imagePullSecrets:
      - name: dockerhub-secret   # หรือ registry secret
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      
      automountServiceAccountToken: false
      
      containers:
      - name: app
        image: nginx:1.25-alpine   # official image (ใช้ public สำหรับ demo)
        ports:
        - containerPort: 8080
        
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
        
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
        
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
```

```bash
kubectl apply -f workshop-deployment.yaml
kubectl get pods -n workshop-images
```

### Step 4: Scan Images ที่ใช้ใน Cluster

```bash
# ดู images ที่รันอยู่
kubectl get pods -n workshop-images -o json | \
  jq -r '.items[].spec.containers[].image' | sort -u

# Scan แต่ละ image
IMAGES=$(kubectl get pods -n workshop-images -o json | \
  jq -r '.items[].spec.containers[].image' | sort -u)

for IMAGE in $IMAGES; do
  echo "=== Scanning: $IMAGE ==="
  trivy image --severity HIGH,CRITICAL --quiet "$IMAGE" 2>&1 || echo "Scan completed for $IMAGE"
  echo ""
done
```

### Step 5: สร้าง Image Security Policy

```yaml
# image-policy.yaml
# ใช้ ValidatingWebhookConfiguration หรือ OPA Gatekeeper

# ตัวอย่างด้วย Kyverno (ง่ายกว่า OPA)
---
# ติดตั้ง Kyverno ก่อน
# kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.11.0/install.yaml

# Policy: ห้ามใช้ image ที่ไม่มี tag หรือใช้ :latest
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/description: >
      ห้ามใช้ image tag ":latest" เพราะไม่ reproducible
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - workshop-images
    validate:
      message: "Image tag ':latest' is not allowed. Use a specific version tag."
      pattern:
        spec:
          containers:
          - image: "?*:?*"    # ต้องมี tag
            (image): "!*:latest"  # tag ห้ามเป็น latest
---
# Policy: ต้อง run as non-root
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-nonroot
  annotations:
    policies.kyverno.io/title: Require Non-Root
spec:
  validationFailureAction: enforce
  rules:
  - name: check-runasnonroot
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - workshop-images
    validate:
      message: "Containers must not run as root"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
          containers:
          - securityContext:
              runAsNonRoot: true
```

```bash
# ติดตั้ง Kyverno (ถ้ายังไม่มี)
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.11.0/install.yaml 2>/dev/null || true
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kyverno \
  -n kyverno --timeout=120s

kubectl apply -f image-policy.yaml
```

### Step 6: ทดสอบ Policy Enforcement

```bash
# ทดสอบ: ใช้ latest tag - ควร FAIL
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-latest-tag
  namespace: workshop-images
spec:
  containers:
  - name: app
    image: nginx:latest   # ควรถูก reject
    command: ["sleep", "3600"]
EOF
echo "Expected: Rejected by Kyverno policy"

# ทดสอบ: ใช้ specific tag - ควร PASS (ถ้า security context ok)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-specific-tag
  namespace: workshop-images
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.25-alpine   # specific tag OK
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
EOF
echo "Expected: Accepted"
```

### Step 7: Image Scanning ใน CI/CD

```yaml
# .github/workflows/build-scan-push.yml
# ตัวอย่าง GitHub Actions workflow

name: Build, Scan, and Push Image

on:
  push:
    branches: [main]

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build Image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Scan with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'    # fail ถ้ามี CRITICAL
    
    - name: Upload Trivy Results
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Login to Harbor
      if: success()   # push เฉพาะถ้า scan ผ่าน
      run: |
        echo ${{ secrets.HARBOR_PASSWORD }} | \
        docker login harbor.example.com \
          -u ${{ secrets.HARBOR_USERNAME }} \
          --password-stdin
    
    - name: Sign Image with Cosign
      if: success()
      env:
        COSIGN_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
      run: |
        docker tag myapp:${{ github.sha }} \
          harbor.example.com/production/myapp:${{ github.sha }}
        cosign sign --key env://COSIGN_KEY \
          harbor.example.com/production/myapp:${{ github.sha }}
    
    - name: Push Image
      if: success()
      run: |
        docker push harbor.example.com/production/myapp:${{ github.sha }}
```

### Step 8: Image Vulnerability Dashboard

```bash
# ดูสรุป vulnerabilities ทั้ง cluster
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[].spec.containers[].image' | \
  sort -u | \
  while read IMAGE; do
    COUNT=$(trivy image --quiet --format json "$IMAGE" 2>/dev/null | \
      jq '.Results[]?.Vulnerabilities[]? | select(.Severity == "CRITICAL")' | \
      wc -l)
    echo "$IMAGE: $COUNT CRITICAL vulnerabilities"
  done
```

### Step 9: Cleanup

```bash
kubectl delete namespace workshop-images

# ลบ Kyverno policies
kubectl delete clusterpolicy disallow-latest-tag require-run-as-nonroot 2>/dev/null || true

echo "Workshop cleanup complete!"
```

---

## 60.8 Image Security Best Practices

### 1. ใช้ Minimal Base Images

```dockerfile
# ลำดับความปลอดภัย (ปลอดภัยกว่าจากบนลงล่าง):
# scratch       - ไม่มีอะไรเลย
# distroless    - ไม่มี shell, package manager
# alpine        - เล็กมาก ~5MB
# debian-slim   - ใหญ่กว่าแต่ compatible กว่า
# ubuntu/debian - ใหญ่ มี vulnerabilities มากกว่า
```

### 2. อัพเดต Base Images สม่ำเสมอ

```bash
# ใช้ Dependabot หรือ Renovate สำหรับ auto-update
# .github/dependabot.yml
version: 2
updates:
- package-ecosystem: "docker"
  directory: "/"
  schedule:
    interval: "weekly"
  target-branch: "main"
```

### 3. ห้าม Store Secrets ใน Images

```dockerfile
# ไม่ดี - secret ใน build arg
ARG API_KEY
ENV API_KEY=$API_KEY

# ดี - inject ตอน runtime
# ใช้ kubernetes secrets แทน
```

### 4. Scan ทุก Image ก่อน Deploy

```bash
# Policy: image ต้องผ่าน scan ก่อน production deploy
# - ไม่มี CRITICAL vulnerabilities
# - HIGH ต้องน้อยกว่า threshold ที่กำหนด
# - Image ต้องมี signature
```

### 5. Image Immutability

```bash
# ห้ามใช้ mutable tags (:latest, :stable)
# ใช้ digest แทน tag
docker pull nginx@sha256:abc123...

# หรือใช้ tag ที่ immutable
image: harbor.example.com/production/myapp:1.0.0-20240115-abc1234
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Image Security Basics**: Dockerfile best practices, multi-stage builds, distroless
2. **Trivy**: ใช้ scan images, k8s clusters, IaC configs
3. **Harbor**: Private registry ที่มี built-in scanning
4. **Image Pull Secrets**: authenticate กับ private registries
5. **Admission Controllers**: enforce image policies อัตโนมัติ

**Key Takeaways:**
- Scan images ทุก image ก่อน production
- ใช้ minimal base images (distroless/alpine)
- ห้ามใช้ :latest tag ใน production
- Private registry + scanning = secure image pipeline
- Admission controllers enforce policies อัตโนมัติ

---

## สรุปส่วน Security (Parts 51-60)

เราได้เรียนรู้ security ครบถ้วนใน 10 บทนี้:

| Part | หัวข้อ | Key Learning |
|------|-------|--------------|
| 51 | Environment Variables | ConfigMap vs Secret, envFrom, Downward API |
| 52 | ConfigMap Advanced | Immutability, Volume mount, Hot-reload |
| 53 | Secrets Management | Sealed Secrets, External Secrets Operator |
| 54 | HashiCorp Vault | Dynamic secrets, Agent Injector |
| 55 | RBAC | Roles, ClusterRoles, Least Privilege |
| 56 | Service Accounts | Identity for pods, Projected tokens |
| 57 | Security Contexts | runAsUser, Capabilities, ReadOnly FS |
| 58 | Pod Security Standards | Privileged/Baseline/Restricted, PSA |
| 59 | Network Policies | Micro-segmentation, Egress control |
| 60 | Image Security | Scanning, Private registry, Admission control |

**Defense in Depth สำหรับ Kubernetes:**
```
Layer 1: Image Security (scan + sign + private registry)
Layer 2: Pod Security (security contexts, PSS)
Layer 3: RBAC (least privilege)
Layer 4: Network Policies (micro-segmentation)
Layer 5: Secrets Management (Vault/Sealed Secrets)
Layer 6: Service Accounts (limited identity)
Layer 7: Audit Logging (detect anomalies)
```

---

**ต่อไป**: Part 61 - Monitoring และ Observability
