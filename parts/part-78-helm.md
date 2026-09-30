# Part 78: Helm - Package Manager สำหรับ Kubernetes

## บทนำ

Helm เป็น Package Manager สำหรับ Kubernetes ที่ช่วยให้การ Deploy แอปพลิเคชันที่ซับซ้อนง่ายขึ้น เปรียบเหมือน apt/yum สำหรับ Linux หรือ npm สำหรับ Node.js Helm ใช้ "Charts" ซึ่งเป็น Package ที่มี Template และ Values สำหรับ Configure การ Deploy

## 1. Helm Concepts

### 1.1 Terms ที่สำคัญ

**Chart:** Package ที่มีทุกอย่างที่จำเป็นสำหรับ Deploy Application ประกอบด้วย:
- Templates (Kubernetes YAML files with Go templating)
- Values (Default configuration)
- Chart.yaml (Metadata)
- README.md (Documentation)

**Release:** Instance ของ Chart ที่ Deploy แล้ว บน Kubernetes

**Repository:** ที่เก็บ Charts (คล้าย Docker Registry แต่สำหรับ Charts)

```
Helm Chart (Package)
    │
    ├── Chart.yaml       (Metadata)
    ├── values.yaml      (Default Values)
    ├── charts/          (Dependencies)
    ├── templates/       (Kubernetes Templates)
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── ingress.yaml
    │   ├── _helpers.tpl (Template Helpers)
    │   └── NOTES.txt    (Post-install Notes)
    └── README.md
```

### 1.2 Helm v3 vs v2

Helm v3 ความแตกต่างหลักจาก v2:
- ไม่มี Tiller (Server component) แล้ว ทำงานโดยตรงกับ Kubernetes API
- Release Information เก็บเป็น Kubernetes Secrets (แทน ConfigMap ใน v2)
- Namespace-scoped Releases
- Chart เก็บใน OCI Registry ได้

## 2. ติดตั้ง Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm

# ตรวจสอบ
helm version

# เพิ่ม Stable Repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## 3. Helm Commands

### 3.1 Repository Commands

```bash
# เพิ่ม Repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add jetstack https://charts.jetstack.io

# อัปเดต Repositories
helm repo update

# ดู Repositories ที่มี
helm repo list

# ลบ Repository
helm repo remove bitnami

# ค้นหา Chart
helm search repo nginx
helm search repo nginx --versions  # ดูทุก Versions
helm search hub nginx              # ค้นใน Artifact Hub
```

### 3.2 Install Commands

```bash
# ติดตั้ง Chart
helm install release-name chart-name
helm install my-nginx ingress-nginx/ingress-nginx

# ติดตั้งใน Namespace เฉพาะ
helm install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# ติดตั้งพร้อม Custom Values
helm install my-nginx ingress-nginx/ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer

# ติดตั้งจาก Values File
helm install my-nginx ingress-nginx/ingress-nginx \
  -f my-values.yaml

# ติดตั้ง Specific Version
helm install my-nginx ingress-nginx/ingress-nginx \
  --version 4.8.0

# Dry Run
helm install my-nginx ingress-nginx/ingress-nginx \
  --dry-run --debug

# Install หรือ Upgrade
helm upgrade --install my-nginx ingress-nginx/ingress-nginx
```

### 3.3 Upgrade Commands

```bash
# Upgrade Release
helm upgrade my-nginx ingress-nginx/ingress-nginx \
  --set controller.replicaCount=3

# Upgrade จาก Values File
helm upgrade my-nginx ingress-nginx/ingress-nginx \
  -f values.yaml

# Upgrade และ Reset Values
helm upgrade my-nginx ingress-nginx/ingress-nginx \
  --reset-values \
  -f new-values.yaml

# Rollback ไป Previous Revision
helm rollback my-nginx 1

# ดู Revision History
helm history my-nginx
```

### 3.4 Status และ Inspection

```bash
# ดู Releases ที่ Deploy
helm list
helm list --all-namespaces
helm list -n production

# ดู Status ของ Release
helm status my-nginx

# ดู Values ที่ใช้
helm get values my-nginx
helm get values my-nginx --all  # รวม Default Values ด้วย

# ดู Manifests ที่ Generate
helm get manifest my-nginx

# ดู Notes
helm get notes my-nginx

# ดู Hooks
helm get hooks my-nginx
```

### 3.5 Uninstall

```bash
# ลบ Release
helm uninstall my-nginx
helm uninstall my-nginx -n ingress-nginx

# ลบแต่ Keep History
helm uninstall my-nginx --keep-history
```

## 4. Chart Structure

### 4.1 สร้าง Chart ใหม่

```bash
# สร้าง Chart
helm create myapp

# โครงสร้างที่สร้าง
myapp/
├── Chart.yaml
├── values.yaml
├── charts/         (empty)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

### 4.2 Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2             # Helm Chart API Version
name: myapp                # Chart Name
description: My Application Chart
type: application          # application หรือ library

# Version ของ Chart
version: 1.2.3             # Semantic Versioning

# Version ของ Application
appVersion: "2.0.0"

# Keywords สำหรับค้นหา
keywords:
- webapp
- microservice

# Maintainers
maintainers:
- name: Team Platform
  email: platform@example.com
  url: https://platform.example.com

# Home
home: https://example.com

# Icon
icon: https://example.com/icon.png

# Sources
sources:
- https://github.com/myorg/myapp

# Dependencies
dependencies:
- name: postgresql
  version: ">=12.0.0 <13.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled

- name: redis
  version: ">=17.0.0 <18.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled

# Annotations
annotations:
  category: WebApplication
  licenses: Apache-2.0
```

### 4.3 values.yaml

```yaml
# values.yaml - Default Values
# Override ด้วย -f values-prod.yaml หรือ --set key=value

# Replica Count
replicaCount: 1

# Image Configuration
image:
  repository: registry.example.com/myapp
  pullPolicy: IfNotPresent
  tag: ""  # Default ใช้ .Chart.AppVersion

# Image Pull Secrets
imagePullSecrets: []

# Service Account
serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

# Pod Annotations และ Labels
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"

podLabels: {}

# Security Context
podSecurityContext:
  fsGroup: 2000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

# Service Configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress Configuration
ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
  - host: myapp.example.com
    paths:
    - path: /
      pathType: Prefix
  tls: []

# Resources
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

# Liveness and Readiness Probes
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

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

# Node Selector, Tolerations, Affinity
nodeSelector: {}

tolerations: []

affinity: {}

# Application Environment Variables
env:
  ENVIRONMENT: production
  LOG_LEVEL: info

# Extra Environment Variables from Secrets
envFrom:
  - secretRef:
      name: myapp-secrets

# Volume Mounts
volumeMounts:
  - name: tmp
    mountPath: /tmp

volumes:
  - name: tmp
    emptyDir: {}

# PostgreSQL (Subchart)
postgresql:
  enabled: false
  auth:
    database: myapp
    username: myapp
    existingSecret: ""

# Redis (Subchart)
redis:
  enabled: false
  architecture: standalone
  auth:
    enabled: false

# Config
config:
  database:
    host: ""
    port: 5432
    name: myapp
  
  cache:
    host: ""
    port: 6379
    ttl: 3600
  
  features:
    enableNewUI: false
    enableDarkMode: true
```

## 5. Templates

### 5.1 _helpers.tpl

```gotemplate
{{/*
_helpers.tpl - Reusable Template Helpers
*/}}

{{/* Chart Name */}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/* Release Name */}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/* Chart Label */}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/* Common Labels */}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/* Selector Labels */}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/* Service Account Name */}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/* Image Name */}}
{{- define "myapp.image" -}}
{{- $tag := default .Chart.AppVersion .Values.image.tag -}}
{{- printf "%s:%s" .Values.image.repository $tag -}}
{{- end }}
```

### 5.2 deployment.yaml Template

```gotemplate
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "myapp.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      
      {{- if .Values.initContainers }}
      initContainers:
        {{- toYaml .Values.initContainers | nindent 8 }}
      {{- end }}
      
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: {{ include "myapp.image" . }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          
          {{- if .Values.env }}
          env:
          {{- range $key, $value := .Values.env }}
          - name: {{ $key }}
            value: {{ $value | quote }}
          {{- end }}
          {{- end }}
          
          {{- if .Values.envFrom }}
          envFrom:
            {{- toYaml .Values.envFrom | nindent 12 }}
          {{- end }}
          
          {{- if .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          {{- end }}
          
          {{- if .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          {{- end }}
          
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          
          {{- if .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml .Values.volumeMounts | nindent 12 }}
          {{- end }}
      
      {{- if .Values.volumes }}
      volumes:
        {{- toYaml .Values.volumes | nindent 8 }}
      {{- end }}
      
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 5.3 service.yaml Template

```gotemplate
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

### 5.4 ingress.yaml Template

```gotemplate
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### 5.5 hpa.yaml Template

```gotemplate
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "myapp.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### 5.6 NOTES.txt

```
# templates/NOTES.txt
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ (index $host.paths 0).path }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "myapp.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "myapp.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "myapp.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "myapp.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

## 6. Values Files สำหรับ Multi-environment

### 6.1 Production Values

```yaml
# values-production.yaml
replicaCount: 5

image:
  tag: v1.2.3

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
  - host: myapp.example.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - secretName: myapp-tls
    hosts:
    - myapp.example.com

env:
  ENVIRONMENT: production
  LOG_LEVEL: warn
  DATABASE_HOST: postgres.production.svc.cluster.local

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app.kubernetes.io/name
          operator: In
          values:
          - myapp
      topologyKey: kubernetes.io/hostname

postgresql:
  enabled: false  # ใช้ External Database ใน Production
```

### 6.2 Staging Values

```yaml
# values-staging.yaml
replicaCount: 2

image:
  tag: v1.2.3-rc1

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false

ingress:
  enabled: true
  className: nginx
  hosts:
  - host: staging.example.com
    paths:
    - path: /
      pathType: Prefix

env:
  ENVIRONMENT: staging
  LOG_LEVEL: info

postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: stagingpassword
```

### 6.3 Development Values

```yaml
# values-dev.yaml
replicaCount: 1

image:
  tag: develop
  pullPolicy: Always

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi

autoscaling:
  enabled: false

ingress:
  enabled: false

env:
  ENVIRONMENT: development
  LOG_LEVEL: debug
  DEBUG: "true"

postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: devpassword
```

## 7. Template Functions และ Techniques

### 7.1 Conditional Rendering

```gotemplate
# If/Else
{{- if .Values.ingress.enabled }}
# Ingress resources
{{- end }}

# If/Else if/Else
{{- if eq .Values.service.type "LoadBalancer" }}
# LoadBalancer config
{{- else if eq .Values.service.type "NodePort" }}
# NodePort config
{{- else }}
# ClusterIP config
{{- end }}

# Unless (opposite of if)
{{- unless .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

### 7.2 Loops

```gotemplate
# Range over slice
{{- range .Values.ingress.hosts }}
- host: {{ .host | quote }}
{{- end }}

# Range over map
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}

# Range with index
{{- range $i, $host := .Values.ingress.hosts }}
- {{ $i }}: {{ $host.host }}
{{- end }}
```

### 7.3 Named Templates

```gotemplate
# Define Template
{{- define "myapp.configmap" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}

# Use Template
{{ include "myapp.configmap" . }}
```

### 7.4 Default Values

```gotemplate
# ใช้ default เมื่อ Value ไม่มีหรือ Empty
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}

# ใช้ required เมื่อ Value บังคับต้องมี
host: {{ required "ingress.host is required" .Values.ingress.host }}

# ใช้ coalesce เพื่อเลือก Value แรกที่ไม่ว่าง
namespace: {{ coalesce .Values.namespace .Release.Namespace "default" }}
```

### 7.5 String Functions

```gotemplate
# Trim
{{- .Values.name | trunc 63 | trimSuffix "-" -}}

# Upper/Lower
{{- .Values.env | upper -}}
{{- .Values.env | lower -}}

# Replace
{{- .Chart.Name | replace "+" "_" -}}

# Contains
{{- if contains "prod" .Values.env -}}
# production code
{{- end -}}

# Join
{{- join "," .Values.hosts -}}

# Quote
{{ .Values.message | quote }}

# Indent
{{- toYaml .Values.resources | indent 4 }}

# nindent (indent + leading newline)
{{- toYaml .Values.resources | nindent 4 }}
```

### 7.6 Type Conversion

```gotemplate
# toString
{{ .Values.port | toString }}

# toJson
{{ .Values.config | toJson }}

# toYaml
{{ .Values.resources | toYaml }}

# int
{{ .Values.port | int }}

# float64
{{ .Values.cpuLimit | float64 }}
```

## 8. Helm Repository

### 8.1 สร้าง Chart Repository

```bash
# Package Chart
helm package myapp/

# สร้าง Repository Index
mkdir helm-repo
cp myapp-1.0.0.tgz helm-repo/
helm repo index helm-repo/ --url https://charts.example.com

# ดู index.yaml ที่สร้าง
cat helm-repo/index.yaml
```

### 8.2 ใช้ GitHub Pages เป็น Helm Repository

```bash
# สร้าง gh-pages branch
git checkout --orphan gh-pages
git rm -rf .
git checkout main -- charts/  # copy charts directory

# Package และ Index
mkdir -p docs
helm package charts/* --destination docs/
helm repo index docs/ --url https://myorg.github.io/helm-repo

# Push
git add docs/
git commit -m "Update Helm Repository"
git push origin gh-pages

# เพิ่ม Repository
helm repo add myorg https://myorg.github.io/helm-repo
helm search repo myorg/
```

### 8.3 ใช้ OCI Registry

```bash
# Login to Registry
helm registry login registry.example.com \
  --username user \
  --password password

# Push Chart to OCI Registry
helm push myapp-1.0.0.tgz oci://registry.example.com/charts

# Pull Chart จาก OCI Registry
helm pull oci://registry.example.com/charts/myapp --version 1.0.0

# Install จาก OCI Registry
helm install myapp \
  oci://registry.example.com/charts/myapp \
  --version 1.0.0
```

## 9. Workshop: สร้าง Helm Chart แรก

### Workshop Overview

สร้าง Helm Chart สำหรับ REST API Application ที่มี:
- Deployment
- Service
- Ingress
- HorizontalPodAutoscaler
- ConfigMap
- NOTES.txt

### Step 1: สร้าง Chart

```bash
# สร้าง Chart
helm create todo-api
cd todo-api

# ดู Structure
ls -la
```

### Step 2: ปรับ Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: todo-api
description: A REST API for managing TODO items
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
- todo
- rest-api
- microservice
maintainers:
- name: Platform Team
  email: platform@example.com
```

### Step 3: กำหนด Values

```yaml
# values.yaml
replicaCount: 1

image:
  repository: registry.example.com/todo-api
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
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
  - host: todo-api.example.com
    paths:
    - path: /
      pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Application Configuration
config:
  server:
    port: 8080
    timeout: 30s
  database:
    driver: sqlite
    path: /data/todos.db
  logging:
    level: info
    format: json
  features:
    enableMetrics: true
    enableHealthCheck: true

# Persistence (for SQLite)
persistence:
  enabled: false
  storageClass: ""
  size: 1Gi
  accessMode: ReadWriteOnce

livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 15
  periodSeconds: 20

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10

nodeSelector: {}
tolerations: []
affinity: {}
```

### Step 4: สร้าง Templates

```gotemplate
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "todo-api.fullname" . }}-config
  labels:
    {{- include "todo-api.labels" . | nindent 4 }}
data:
  config.yaml: |
    server:
      port: {{ .Values.config.server.port }}
      timeout: {{ .Values.config.server.timeout }}
    database:
      driver: {{ .Values.config.database.driver }}
      path: {{ .Values.config.database.path }}
    logging:
      level: {{ .Values.config.logging.level }}
      format: {{ .Values.config.logging.format }}
    features:
      enableMetrics: {{ .Values.config.features.enableMetrics }}
      enableHealthCheck: {{ .Values.config.features.enableHealthCheck }}
```

```gotemplate
# templates/deployment.yaml (simplified)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "todo-api.fullname" . }}
  labels:
    {{- include "todo-api.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "todo-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "todo-api.labels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "todo-api.serviceAccountName" . }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.config.server.port }}
        
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        {{- if .Values.persistence.enabled }}
        - name: data
          mountPath: /data
        {{- end }}
        
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 10 }}
        
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 10 }}
        
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
      
      volumes:
      - name: config
        configMap:
          name: {{ include "todo-api.fullname" . }}-config
      {{- if .Values.persistence.enabled }}
      - name: data
        persistentVolumeClaim:
          claimName: {{ include "todo-api.fullname" . }}-data
      {{- end }}
```

### Step 5: Test Chart

```bash
# ตรวจสอบ Syntax
helm lint todo-api/

# Preview Output
helm template todo-api todo-api/

# Preview พร้อม Values
helm template todo-api todo-api/ \
  -f todo-api/values.yaml \
  --set ingress.enabled=true

# Dry Run ใน Cluster
helm install todo-api todo-api/ \
  --dry-run --debug

# ติดตั้งจริง
helm install todo-api todo-api/ \
  --namespace workshop \
  --create-namespace

# ตรวจสอบ
helm list -n workshop
helm status todo-api -n workshop
kubectl get all -n workshop
```

### Step 6: Package และ Push Chart

```bash
# Package Chart
helm package todo-api/
# สร้าง: todo-api-0.1.0.tgz

# Push ไปยัง OCI Registry
helm registry login registry.example.com \
  --username user \
  --password password

helm push todo-api-0.1.0.tgz \
  oci://registry.example.com/charts

# ทดสอบ Pull และ Install
helm install todo-api \
  oci://registry.example.com/charts/todo-api \
  --version 0.1.0 \
  -n workshop
```

## 10. Debugging Tips

```bash
# ดู Generated YAML
helm template myapp myapp/ > output.yaml
kubectl apply --dry-run=client -f output.yaml

# ดู Template ที่มี Error
helm template myapp myapp/ 2>&1 | head -50

# ใช้ --debug
helm install myapp myapp/ --debug

# Check Values
helm inspect values myapp/

# ดู Chart Information
helm inspect chart myapp/
helm inspect readme myapp/
```

## สรุป

Helm ให้:
1. **Package Management** - จัดการ Application เหมือน Package Manager
2. **Templating** - Go Templates สำหรับ Kubernetes YAML
3. **Versioning** - Track Chart และ App Versions
4. **Release Management** - Install, Upgrade, Rollback
5. **Repository** - แชร์ Charts ใน Team/Community
6. **Environment Configuration** - Values Files สำหรับแต่ละ Environment

ในบทต่อไปจะเรียนรู้ Helm Advanced Features ได้แก่ Hooks, Tests, Dependencies และ Production Helm Chart

## แบบฝึกหัด

1. สร้าง Helm Chart สำหรับ Application ของคุณ
2. ตั้งค่า Values Files สำหรับ Dev, Staging, Production
3. เพิ่ม HPA ใน Chart
4. Publish Chart ไปยัง GitHub Pages
5. ทดลอง Rollback ด้วย helm rollback
