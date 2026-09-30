# Part 83: Admission Controllers

## บทนำ

Admission Controllers คือ plugin ที่ทำงานในส่วน kube-apiserver เพื่อ intercept requests ที่เข้ามายัง Kubernetes API ก่อนที่ Object จะถูก persist ใน etcd แต่หลังจาก authentication และ authorization

Admission Controllers ช่วยให้คุณสามารถ:
- **Validate** requests เพื่อให้แน่ใจว่าตรงตาม policy
- **Mutate** requests เพื่อเพิ่มหรือแก้ไขข้อมูล
- บังคับ best practices ทั่วทั้ง cluster
- ป้องกัน misconfiguration ที่อาจเกิดขึ้น

---

## 83.1 ประเภทของ Admission Controllers

### Built-in Admission Controllers

```
Request Flow:
kubectl → API Server → Authentication → Authorization → Admission Controllers → etcd

Admission Controllers:
1. Mutating Admission    ← แก้ไข request ก่อน
2. Object Schema Validation
3. Validating Admission  ← validate request
```

### Built-in Controllers ที่สำคัญ

```
NamespaceLifecycle     - ป้องกันการสร้าง objects ใน namespace ที่กำลังถูกลบ
LimitRanger            - กำหนด default limits/requests สำหรับ pods
ResourceQuota          - บังคับ resource quotas
ServiceAccount         - เพิ่ม service account token โดยอัตโนมัติ
DefaultStorageClass    - กำหนด default storage class
DefaultTolerationSeconds - กำหนด default tolerations
PodSecurity            - บังคับ Pod Security Standards
NodeRestriction        - จำกัด node permissions
MutatingAdmissionWebhook  - เรียก external webhook สำหรับ mutation
ValidatingAdmissionWebhook - เรียก external webhook สำหรับ validation
```

### ดู Built-in Controllers

```bash
# ดู enabled admission plugins
kubectl exec -n kube-system kube-apiserver-$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') \
  -- kube-apiserver --help 2>&1 | grep -A 1 "enable-admission-plugins"

# หรือดูจาก kube-apiserver flags
ps aux | grep kube-apiserver | grep -o 'enable-admission-plugins=[^ ]*'
```

---

## 83.2 Mutating vs Validating Admission Controllers

### ความแตกต่าง

```
Mutating Admission Controllers:
- ทำงานก่อน Validating Controllers
- สามารถ เพิ่ม/แก้ไข/ลบ fields ใน object
- เช่น: เพิ่ม sidecar container, กำหนด default values, inject environment variables
- ต้อง idempotent (ผลลัพธ์เหมือนกันไม่ว่าจะเรียกกี่ครั้ง)

Validating Admission Controllers:
- ทำงานหลัง Mutating Controllers
- ไม่สามารถแก้ไข object ได้
- approve หรือ reject request เท่านั้น
- สามารถรัน parallel ได้
```

### ลำดับการทำงาน

```
Request
   │
   ▼
Authentication & Authorization
   │
   ▼
Mutating Admission Webhooks (ทำงาน serial)
   │ (Object ถูกแก้ไขได้)
   ▼
Object Schema Validation
   │
   ▼
Validating Admission Webhooks (ทำงาน parallel)
   │ (ถ้า reject → request ถูก reject)
   ▼
Stored in etcd
```

---

## 83.3 OPA/Gatekeeper

### Open Policy Agent (OPA)

OPA เป็น policy engine ที่ใช้ Rego language สำหรับเขียน policies

```bash
# ติดตั้ง OPA Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.15/deploy/gatekeeper.yaml

# ตรวจสอบ
kubectl get pods -n gatekeeper-system
kubectl get crd | grep gatekeeper
```

### Gatekeeper Architecture

```
Kubernetes API Server
        │
        │ Validating Webhook
        ▼
Gatekeeper Controller
        │
        ├── Constraint Templates (policy definitions using Rego)
        └── Constraints (policy instances with parameters)

Flow:
1. สร้าง ConstraintTemplate (Rego policy)
2. สร้าง Constraint (นำ policy มาใช้พร้อม parameters)
3. Gatekeeper validates requests ตาม constraints
```

### สร้าง ConstraintTemplate

```yaml
# require-labels-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  annotations:
    description: "Requires resources to have specific labels"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema for the parameters
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
              description: "List of required label keys"
            message:
              type: string
              description: "Custom error message"
  
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        # ตรวจสอบว่ามี required labels ครบหรือไม่
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          # ดึง labels ที่ต้องการจาก parameters
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          
          # สร้าง message
          default_msg := sprintf("Missing required labels: %v", [missing])
          msg := object.get(input.parameters, "message", default_msg)
        }
```

```yaml
# require-labels-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    labels:
      - "team"
      - "env"
    message: "Resources must have 'team' and 'env' labels"
```

### Rego Policy Examples

```yaml
# container-limits-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8scontainerlimits
spec:
  crd:
    spec:
      names:
        kind: K8sContainerLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            cpu:
              type: string
            memory:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8scontainerlimits
        
        import future.keywords.every
        
        # ตรวจสอบ cpu limits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' does not have CPU limit", [container.name])
        }
        
        # ตรวจสอบ memory limits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' does not have memory limit", [container.name])
        }
        
        # ตรวจสอบว่า cpu limit ไม่เกิน max
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.resources.limits.cpu
          max_cpu := input.parameters.cpu
          cpu_in_milli := to_number(trim_suffix(container.resources.limits.cpu, "m"))
          max_cpu_in_milli := to_number(trim_suffix(max_cpu, "m"))
          cpu_in_milli > max_cpu_in_milli
          msg := sprintf("Container '%v' CPU limit %v exceeds maximum %v", 
            [container.name, container.resources.limits.cpu, max_cpu])
        }
```

```yaml
# no-privileged-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoprivileged
        
        # ตรวจสอบ containers
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged
          msg := sprintf("Container '%v' must not run as privileged", [container.name])
        }
        
        # ตรวจสอบ init containers
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged
          msg := sprintf("Init container '%v' must not run as privileged", [container.name])
        }
        
        # ตรวจสอบ Pod-level securityContext
        violation[{"msg": msg}] {
          input.review.object.spec.securityContext.runAsUser == 0
          msg := "Pod must not run as root (UID 0)"
        }
```

```yaml
# allowed-repos-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [repo | repo := input.parameters.repos[_]; 
                        startswith(container.image, repo)]
          count(satisfied) == 0
          msg := sprintf("Container image '%v' is not from an allowed repository. Allowed: %v", 
                        [container.image, input.parameters.repos])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          satisfied := [repo | repo := input.parameters.repos[_]; 
                        startswith(container.image, repo)]
          count(satisfied) == 0
          msg := sprintf("Init container image '%v' is not from an allowed repository", [container.image])
        }
```

---

## 83.4 Workshop: Policy-as-Code

### เป้าหมาย
ติดตั้งและกำหนด comprehensive security policies สำหรับ production cluster

### ขั้นตอนที่ 1: ติดตั้ง Gatekeeper

```bash
# ติดตั้ง Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.15/deploy/gatekeeper.yaml

# รอให้ Ready
kubectl wait --for=condition=ready pod -l control-plane=controller-manager \
  -n gatekeeper-system --timeout=300s

# ตรวจสอบ
kubectl get pods -n gatekeeper-system
kubectl get crd | grep gatekeeper.sh
```

### ขั้นตอนที่ 2: สร้าง Policy Library

```yaml
# policy-library/templates/require-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

```yaml
# policy-library/templates/container-security.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8scontainersecurity
spec:
  crd:
    spec:
      names:
        kind: K8sContainerSecurity
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowPrivileged:
              type: boolean
              default: false
            requireReadOnlyRootFilesystem:
              type: boolean
              default: false
            allowPrivilegeEscalation:
              type: boolean
              default: false
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8scontainersecurity
        
        check_container(container) {
          not input.parameters.allowPrivileged
          container.securityContext.privileged
        }
        
        check_container(container) {
          input.parameters.requireReadOnlyRootFilesystem
          not container.securityContext.readOnlyRootFilesystem
        }
        
        check_container(container) {
          not input.parameters.allowPrivilegeEscalation
          container.securityContext.allowPrivilegeEscalation
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          check_container(container)
          msg := sprintf("Container '%v' violates security policy", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          check_container(container)
          msg := sprintf("Init container '%v' violates security policy", [container.name])
        }
```

```yaml
# policy-library/templates/resource-limits.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            requireLimits:
              type: boolean
              default: true
            requireRequests:
              type: boolean
              default: true
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sresourcelimits
        
        violation[{"msg": msg}] {
          input.parameters.requireLimits
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' must have CPU limits", [container.name])
        }
        
        violation[{"msg": msg}] {
          input.parameters.requireLimits
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must have memory limits", [container.name])
        }
        
        violation[{"msg": msg}] {
          input.parameters.requireRequests
          container := input.review.object.spec.containers[_]
          not container.resources.requests.cpu
          msg := sprintf("Container '%v' must have CPU requests", [container.name])
        }
        
        violation[{"msg": msg}] {
          input.parameters.requireRequests
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("Container '%v' must have memory requests", [container.name])
        }
```

### ขั้นตอนที่ 3: ติดตั้ง Templates

```bash
# ติดตั้ง templates
kubectl apply -f policy-library/templates/

# ตรวจสอบ
kubectl get constrainttemplate
kubectl describe constrainttemplate k8srequiredlabels
```

### ขั้นตอนที่ 4: สร้าง Constraints

```yaml
# constraints/production-labels.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: production-required-labels
spec:
  enforcementAction: deny  # deny หรือ warn หรือ dryrun
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
      - apiGroups: [""]
        kinds: ["Namespace"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - cert-manager
  parameters:
    labels:
      - "team"
      - "env"
      - "app.kubernetes.io/name"
```

```yaml
# constraints/container-security.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sContainerSecurity
metadata:
  name: container-security-policy
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet", "ReplicaSet"]
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    allowPrivileged: false
    requireReadOnlyRootFilesystem: false  # ตั้งเป็น true เมื่อพร้อม
    allowPrivilegeEscalation: false
```

```yaml
# constraints/resource-limits.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sResourceLimits
metadata:
  name: require-resource-limits
spec:
  enforcementAction: warn  # เริ่มต้นด้วย warn ก่อนเปลี่ยนเป็น deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    excludedNamespaces:
      - kube-system
  parameters:
    requireLimits: true
    requireRequests: true
```

```yaml
# constraints/allowed-registries.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedregistries
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRegistries
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
        package k8sallowedregistries
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not allowed_registry(container.image)
          msg := sprintf("Container image '%v' not from allowed registry", [container.image])
        }
        
        allowed_registry(image) {
          registry := input.parameters.registries[_]
          startswith(image, registry)
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRegistries
metadata:
  name: restrict-container-registries
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    registries:
      - "gcr.io/myproject/"
      - "myregistry.example.com/"
      - "docker.io/myorg/"
```

### ขั้นตอนที่ 5: ทดสอบ Policies

```bash
# ติดตั้ง constraints
kubectl apply -f constraints/

# ทดสอบ policy - ควร fail เพราะไม่มี labels
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy-no-labels
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: nginx
          image: nginx:latest
EOF

# ควรได้ error ประมาณนี้:
# Error from server: [...] Missing required labels: {"env", "team", "app.kubernetes.io/name"}

# ทดสอบที่ถูกต้อง
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy-correct
  labels:
    team: platform
    env: staging
    app.kubernetes.io/name: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          securityContext:
            allowPrivilegeEscalation: false
EOF

# ดู violations ที่เกิดขึ้น
kubectl get constraint
kubectl describe constraint production-required-labels

# ดู audit violations
kubectl get k8srequiredlabels.constraints.gatekeeper.sh production-required-labels \
  -o jsonpath='{.status.violations}' | python3 -m json.tool
```

### ขั้นตอนที่ 6: Gatekeeper Audit

```bash
# Gatekeeper จะ audit existing resources ทุก 60 วินาที
# ดู violations ของ existing resources

# ดู all violations
kubectl get constraint -o wide

# ดู specific constraint violations
kubectl describe k8srequiredlabels production-required-labels

# ดู Gatekeeper metrics
kubectl port-forward -n gatekeeper-system svc/gatekeeper-controller-manager-metrics-service 8888:8888 &
curl localhost:8888/metrics | grep gatekeeper
```

---

## 83.5 Pod Security Standards (PSS)

### ความเข้าใจ Pod Security Standards

```
Privileged:     ไม่มี restriction - สำหรับ privileged, system-level workloads
Baseline:       prevents escalation - สำหรับ typical workloads
Restricted:     hardened - สำหรับ security-critical workloads
```

```bash
# ติดตั้ง Pod Security Admission (built-in ตั้งแต่ Kubernetes 1.25)
# ใช้ labels บน namespace

# Namespace สำหรับ baseline
kubectl create namespace test-baseline
kubectl label namespace test-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# ทดสอบ
kubectl -n test-baseline apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        privileged: true  # ควรถูก reject ใน baseline namespace
EOF

# Namespace สำหรับ restricted
kubectl create namespace test-restricted
kubectl label namespace test-restricted \
  pod-security.kubernetes.io/enforce=restricted

# ทดสอบ
kubectl -n test-restricted apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: nginx
      image: nginx:latest
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
EOF
```

---

## 83.6 Custom Admission Controllers

### Validating Admission Controller ด้วย Go

```go
// cmd/admission/main.go
package main

import (
    "crypto/tls"
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"
    "strings"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
)

var (
    scheme  = runtime.NewScheme()
    codecs  = serializer.NewCodecFactory(scheme)
)

func main() {
    cert, err := tls.LoadX509KeyPair("/etc/ssl/certs/tls.crt", "/etc/ssl/private/tls.key")
    if err != nil {
        panic(err)
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/validate-pods", validatePods)
    mux.HandleFunc("/mutate-pods", mutatePods)
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    server := &http.Server{
        Addr: ":8443",
        TLSConfig: &tls.Config{
            Certificates: []tls.Certificate{cert},
        },
        Handler: mux,
    }

    fmt.Println("Starting admission webhook server on :8443")
    if err := server.ListenAndServeTLS("", ""); err != nil {
        panic(err)
    }
}

func validatePods(w http.ResponseWriter, r *http.Request) {
    review := readAdmissionReview(r)
    if review == nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    var pod corev1.Pod
    if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
        writeAdmissionResponse(w, review, false, fmt.Sprintf("Failed to parse pod: %v", err))
        return
    }

    // Validation rules
    var violations []string

    // Rule 1: ต้องมี resource limits
    for _, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil {
            violations = append(violations, fmt.Sprintf("Container '%s' missing resource limits", container.Name))
        }
    }

    // Rule 2: ต้องไม่ run as root
    if pod.Spec.SecurityContext != nil && pod.Spec.SecurityContext.RunAsUser != nil {
        if *pod.Spec.SecurityContext.RunAsUser == 0 {
            violations = append(violations, "Pod must not run as root (UID 0)")
        }
    }

    // Rule 3: ต้องมี labels
    requiredLabels := []string{"app", "env"}
    for _, label := range requiredLabels {
        if _, ok := pod.Labels[label]; !ok {
            violations = append(violations, fmt.Sprintf("Missing required label: %s", label))
        }
    }

    // Rule 4: ห้ามใช้ latest tag
    for _, container := range pod.Spec.Containers {
        if strings.HasSuffix(container.Image, ":latest") || !strings.Contains(container.Image, ":") {
            violations = append(violations, fmt.Sprintf("Container '%s' must not use 'latest' tag", container.Name))
        }
    }

    if len(violations) > 0 {
        writeAdmissionResponse(w, review, false, strings.Join(violations, "; "))
        return
    }

    writeAdmissionResponse(w, review, true, "")
}

func mutatePods(w http.ResponseWriter, r *http.Request) {
    review := readAdmissionReview(r)
    if review == nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    var pod corev1.Pod
    if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
        writeAdmissionResponse(w, review, false, fmt.Sprintf("Failed to parse pod: %v", err))
        return
    }

    var patches []map[string]interface{}

    // Mutation 1: เพิ่ม default labels
    if pod.Labels == nil {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels",
            "value": map[string]string{},
        })
    }
    if _, ok := pod.Labels["managed-by"]; !ok {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels/managed-by",
            "value": "admission-controller",
        })
    }

    // Mutation 2: เพิ่ม sidecar container (logging)
    sidecarExists := false
    for _, c := range pod.Spec.Containers {
        if c.Name == "log-collector" {
            sidecarExists = true
            break
        }
    }

    if !sidecarExists && pod.Annotations["inject-log-collector"] == "true" {
        sidecar := corev1.Container{
            Name:  "log-collector",
            Image: "fluent/fluentd:v1.16",
            Resources: corev1.ResourceRequirements{
                Requests: corev1.ResourceList{
                    corev1.ResourceCPU:    *parseQuantity("50m"),
                    corev1.ResourceMemory: *parseQuantity("64Mi"),
                },
                Limits: corev1.ResourceList{
                    corev1.ResourceCPU:    *parseQuantity("100m"),
                    corev1.ResourceMemory: *parseQuantity("128Mi"),
                },
            },
        }
        
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/spec/containers/-",
            "value": sidecar,
        })
    }

    // Mutation 3: กำหนด imagePullPolicy เป็น Always ถ้าไม่ได้ระบุ
    for i, container := range pod.Spec.Containers {
        if container.ImagePullPolicy == "" {
            patches = append(patches, map[string]interface{}{
                "op":    "replace",
                "path":  fmt.Sprintf("/spec/containers/%d/imagePullPolicy", i),
                "value": "IfNotPresent",
            })
        }
    }

    patchBytes, err := json.Marshal(patches)
    if err != nil {
        writeAdmissionResponse(w, review, false, fmt.Sprintf("Failed to marshal patches: %v", err))
        return
    }

    patchType := admissionv1.PatchTypeJSONPatch
    response := &admissionv1.AdmissionReview{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "admission.k8s.io/v1",
            Kind:       "AdmissionReview",
        },
        Response: &admissionv1.AdmissionResponse{
            UID:       review.Request.UID,
            Allowed:   true,
            Patch:     patchBytes,
            PatchType: &patchType,
        },
    }

    writeResponse(w, response)
}

func readAdmissionReview(r *http.Request) *admissionv1.AdmissionReview {
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        return nil
    }

    review := &admissionv1.AdmissionReview{}
    if err := json.Unmarshal(body, review); err != nil {
        return nil
    }

    return review
}

func writeAdmissionResponse(w http.ResponseWriter, review *admissionv1.AdmissionReview, allowed bool, message string) {
    response := &admissionv1.AdmissionReview{
        TypeMeta: metav1.TypeMeta{
            APIVersion: "admission.k8s.io/v1",
            Kind:       "AdmissionReview",
        },
        Response: &admissionv1.AdmissionResponse{
            UID:     review.Request.UID,
            Allowed: allowed,
        },
    }

    if !allowed && message != "" {
        response.Response.Result = &metav1.Status{
            Message: message,
        }
    }

    writeResponse(w, response)
}

func writeResponse(w http.ResponseWriter, response *admissionv1.AdmissionReview) {
    responseBytes, err := json.Marshal(response)
    if err != nil {
        http.Error(w, fmt.Sprintf("Failed to marshal response: %v", err), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(responseBytes)
}
```

### ลงทะเบียน Webhooks

```yaml
# webhook-config.yaml
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-validator
  annotations:
    cert-manager.io/inject-ca-from: "default/admission-webhook-cert"
webhooks:
  - name: validate-pods.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: /validate-pods
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail       # Fail หรือ Ignore
    namespaceSelector:
      matchLabels:
        admission-webhook: "enabled"

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-mutator
webhooks:
  - name: mutate-pods.example.com
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: /mutate-pods
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Ignore     # Ignore ถ้า webhook ไม่พร้อม
    reinvocationPolicy: Never  # Never หรือ IfNeeded
```

---

## 83.7 Monitoring Admission Controllers

```bash
# ดู webhook configurations
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# ดู webhook failures ใน events
kubectl get events --field-selector reason=WebhookRejection

# ดู audit logs
kubectl get events | grep -i "admission"

# ตรวจสอบ Gatekeeper violations
kubectl get constraint -o custom-columns='NAME:.metadata.name,VIOLATIONS:.status.totalViolations'

# ดู webhook metrics ใน Prometheus
# metric: apiserver_admission_webhook_admission_duration_seconds
# metric: apiserver_admission_webhook_rejection_count
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Admission Controllers**: ประเภทและการทำงาน
2. **Mutating vs Validating**: ความแตกต่างและ use cases
3. **OPA/Gatekeeper**: Policy-as-Code ด้วย Rego
4. **Pod Security Standards**: Built-in security policies
5. **Custom Admission Controllers**: สร้าง webhook เอง
6. **Workshop**: ติดตั้ง comprehensive security policies

บทถัดไปเราจะลงลึกเกี่ยวกับ Admission Webhooks โดยเฉพาะ
