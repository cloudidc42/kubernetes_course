# Part 84: Admission Webhooks

## บทนำ

Admission Webhooks คือกลไกที่ช่วยให้คุณ extend Kubernetes admission control โดยการส่ง HTTP requests ไปยัง external services ที่คุณสร้างขึ้นเอง มีสองประเภทหลัก:

- **Mutating Admission Webhooks**: แก้ไข objects ก่อนที่จะถูก store
- **Validating Admission Webhooks**: validate objects และ approve/reject

---

## 84.1 Webhook Architecture

### Flow การทำงาน

```
kubectl apply
      │
      ▼
API Server receives request
      │
      ▼
Authentication & Authorization
      │
      ▼
Mutating Admission Webhooks
  ├── Webhook Server 1 (mutate)
  ├── Webhook Server 2 (mutate)
  └── Response: patch JSON
      │
      ▼
Object Schema Validation
      │
      ▼
Validating Admission Webhooks
  ├── Webhook Server A (validate)  ─┐ parallel
  ├── Webhook Server B (validate)  ─┤
  └── Response: allow/deny         ─┘
      │
      ▼
Store in etcd (if all approved)
```

### AdmissionReview Format

```json
// Request ที่ API Server ส่งให้ Webhook
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind": {"group": "apps", "version": "v1", "resource": "deployments"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "subResource": "",
    "requestKind": {"group": "apps", "version": "v1", "resource": "deployments"},
    "requestResource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "name": "my-deployment",
    "namespace": "default",
    "operation": "CREATE",
    "userInfo": {
      "username": "admin",
      "groups": ["system:masters", "system:authenticated"]
    },
    "object": {
      // สมบูรณ์ของ object ที่ถูกส่งมา
    },
    "oldObject": null,  // สำหรับ UPDATE จะมี old object
    "dryRun": false
  }
}
```

```json
// Response จาก Webhook
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "allowed": true,
    // สำหรับ Mutating webhook
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiJhZGQiLCJwYXRoIjoiL21ldGFkYXRhL2xhYmVscy9lbnYiLCJ2YWx1ZSI6InRlc3QifV0=",
    // สำหรับ rejection
    "status": {
      "code": 403,
      "message": "Deployment violates policy"
    }
  }
}
```

---

## 84.2 สร้าง Webhook Server ด้วย Go

### โปรเจกต์โครงสร้าง

```
admission-webhook/
├── cmd/
│   └── webhook/
│       └── main.go
├── pkg/
│   ├── webhook/
│   │   ├── handler.go
│   │   ├── mutate.go
│   │   └── validate.go
│   └── certs/
│       └── certs.go
├── deploy/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── webhook-config.yaml
│   └── cert-manager.yaml
├── Dockerfile
└── go.mod
```

### main.go

```go
// cmd/webhook/main.go
package main

import (
    "flag"
    "fmt"
    "net/http"
    "os"
    
    "github.com/example/admission-webhook/pkg/webhook"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    ctrl "sigs.k8s.io/controller-runtime"
)

func main() {
    var (
        port     int
        certFile string
        keyFile  string
    )
    
    flag.IntVar(&port, "port", 8443, "Webhook server port")
    flag.StringVar(&certFile, "tls-cert-file", "/etc/ssl/certs/tls.crt", "TLS certificate file")
    flag.StringVar(&keyFile, "tls-private-key-file", "/etc/ssl/private/tls.key", "TLS key file")
    flag.Parse()
    
    ctrl.SetLogger(zap.New(zap.UseDevMode(true)))
    logger := ctrl.Log.WithName("webhook")
    
    // Kubernetes client
    config, err := rest.InClusterConfig()
    if err != nil {
        logger.Error(err, "Failed to get cluster config")
        os.Exit(1)
    }
    
    kubeClient, err := kubernetes.NewForConfig(config)
    if err != nil {
        logger.Error(err, "Failed to create Kubernetes client")
        os.Exit(1)
    }
    
    // สร้าง webhook handler
    handler := webhook.NewHandler(kubeClient, logger)
    
    mux := http.NewServeMux()
    mux.HandleFunc("/mutate/pods", handler.MutatePods)
    mux.HandleFunc("/validate/pods", handler.ValidatePods)
    mux.HandleFunc("/mutate/deployments", handler.MutateDeployments)
    mux.HandleFunc("/validate/deployments", handler.ValidateDeployments)
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "ok")
    })
    mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "ok")
    })
    
    server := &http.Server{
        Addr:    fmt.Sprintf(":%d", port),
        Handler: mux,
    }
    
    logger.Info("Starting webhook server", "port", port)
    if err := server.ListenAndServeTLS(certFile, keyFile); err != nil {
        logger.Error(err, "Failed to start server")
        os.Exit(1)
    }
}
```

### handler.go

```go
// pkg/webhook/handler.go
package webhook

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"
    
    "github.com/go-logr/logr"
    admissionv1 "k8s.io/api/admission/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

// Handler จัดการ webhook requests
type Handler struct {
    kubeClient kubernetes.Interface
    log        logr.Logger
}

// NewHandler สร้าง Handler ใหม่
func NewHandler(kubeClient kubernetes.Interface, log logr.Logger) *Handler {
    return &Handler{
        kubeClient: kubeClient,
        log:        log,
    }
}

// admitFunc ประเภทของ function ที่ handle admission
type admitFunc func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse

// serve จัดการ HTTP request
func (h *Handler) serve(w http.ResponseWriter, r *http.Request, admit admitFunc) {
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        h.log.Error(err, "Failed to read request body")
        http.Error(w, "Failed to read request", http.StatusBadRequest)
        return
    }
    
    if contentType := r.Header.Get("Content-Type"); contentType != "application/json" {
        http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
        return
    }
    
    // Decode AdmissionReview
    review := &admissionv1.AdmissionReview{}
    if err := json.Unmarshal(body, review); err != nil {
        h.log.Error(err, "Failed to unmarshal AdmissionReview")
        http.Error(w, fmt.Sprintf("Failed to decode: %v", err), http.StatusBadRequest)
        return
    }
    
    if review.Request == nil {
        http.Error(w, "AdmissionReview request is nil", http.StatusBadRequest)
        return
    }
    
    // Process admission
    response := admit(review)
    response.UID = review.Request.UID
    
    // Encode response
    review.Response = response
    responseBytes, err := json.Marshal(review)
    if err != nil {
        h.log.Error(err, "Failed to marshal response")
        http.Error(w, "Failed to encode response", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.Write(responseBytes)
}

// newAllowedResponse สร้าง allowed response
func newAllowedResponse() *admissionv1.AdmissionResponse {
    return &admissionv1.AdmissionResponse{
        Allowed: true,
    }
}

// newDeniedResponse สร้าง denied response
func newDeniedResponse(code int32, message string) *admissionv1.AdmissionResponse {
    return &admissionv1.AdmissionResponse{
        Allowed: false,
        Result: &metav1.Status{
            Code:    code,
            Message: message,
        },
    }
}

// newPatchResponse สร้าง patch response
func newPatchResponse(patches []interface{}) *admissionv1.AdmissionResponse {
    patchBytes, err := json.Marshal(patches)
    if err != nil {
        return newDeniedResponse(500, fmt.Sprintf("Failed to marshal patches: %v", err))
    }
    
    patchType := admissionv1.PatchTypeJSONPatch
    return &admissionv1.AdmissionResponse{
        Allowed:   true,
        Patch:     patchBytes,
        PatchType: &patchType,
    }
}
```

### mutate.go

```go
// pkg/webhook/mutate.go
package webhook

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "time"
    
    admissionv1 "k8s.io/api/admission/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MutatePods handles pod mutation webhook
func (h *Handler) MutatePods(w http.ResponseWriter, r *http.Request) {
    h.serve(w, r, func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
        h.log.Info("Mutating pod", "namespace", review.Request.Namespace, "name", review.Request.Name)
        
        var pod corev1.Pod
        if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
            return newDeniedResponse(400, fmt.Sprintf("Failed to parse pod: %v", err))
        }
        
        patches := h.mutatePodPatches(&pod, review.Request.Namespace)
        if len(patches) == 0 {
            return newAllowedResponse()
        }
        
        return newPatchResponse(patches)
    })
}

// mutatePodPatches สร้าง patches สำหรับ pod
func (h *Handler) mutatePodPatches(pod *corev1.Pod, namespace string) []interface{} {
    var patches []interface{}
    
    // 1. เพิ่ม default labels
    patches = append(patches, h.addDefaultLabels(pod)...)
    
    // 2. เพิ่ม default annotations
    patches = append(patches, h.addDefaultAnnotations(pod, namespace)...)
    
    // 3. กำหนด imagePullPolicy
    patches = append(patches, h.setImagePullPolicy(pod)...)
    
    // 4. เพิ่ม resource defaults
    patches = append(patches, h.setResourceDefaults(pod)...)
    
    // 5. เพิ่ม sidecar ถ้าจำเป็น
    patches = append(patches, h.injectSidecar(pod)...)
    
    // 6. กำหนด security context defaults
    patches = append(patches, h.setSecurityDefaults(pod)...)
    
    return patches
}

func (h *Handler) addDefaultLabels(pod *corev1.Pod) []interface{} {
    var patches []interface{}
    
    if pod.Labels == nil {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels",
            "value": map[string]string{},
        })
    }
    
    defaultLabels := map[string]string{
        "admission-controller/mutated": "true",
        "admission-controller/version": "v1",
    }
    
    for key, value := range defaultLabels {
        if _, ok := pod.Labels[key]; !ok {
            // Escape forward slashes in label key
            escapedKey := strings.ReplaceAll(key, "/", "~1")
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  fmt.Sprintf("/metadata/labels/%s", escapedKey),
                "value": value,
            })
        }
    }
    
    return patches
}

func (h *Handler) addDefaultAnnotations(pod *corev1.Pod, namespace string) []interface{} {
    var patches []interface{}
    
    if pod.Annotations == nil {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/annotations",
            "value": map[string]string{},
        })
    }
    
    annotations := map[string]string{
        "admission-controller/injected-at": time.Now().Format(time.RFC3339),
        "admission-controller/namespace":   namespace,
    }
    
    for key, value := range annotations {
        escapedKey := strings.ReplaceAll(key, "/", "~1")
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  fmt.Sprintf("/metadata/annotations/%s", escapedKey),
            "value": value,
        })
    }
    
    return patches
}

func (h *Handler) setImagePullPolicy(pod *corev1.Pod) []interface{} {
    var patches []interface{}
    
    for i, container := range pod.Spec.Containers {
        if container.ImagePullPolicy == "" {
            var policy corev1.PullPolicy
            if strings.Contains(container.Image, ":") && !strings.HasSuffix(container.Image, ":latest") {
                policy = corev1.PullIfNotPresent
            } else {
                policy = corev1.PullAlways
            }
            
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  fmt.Sprintf("/spec/containers/%d/imagePullPolicy", i),
                "value": string(policy),
            })
        }
    }
    
    return patches
}

func (h *Handler) setResourceDefaults(pod *corev1.Pod) []interface{} {
    var patches []interface{}
    
    defaultCPURequest := resource.MustParse("100m")
    defaultMemRequest := resource.MustParse("128Mi")
    defaultCPULimit := resource.MustParse("500m")
    defaultMemLimit := resource.MustParse("512Mi")
    
    for i, container := range pod.Spec.Containers {
        if container.Resources.Requests == nil {
            patches = append(patches, map[string]interface{}{
                "op":   "add",
                "path": fmt.Sprintf("/spec/containers/%d/resources/requests", i),
                "value": map[string]string{
                    "cpu":    defaultCPURequest.String(),
                    "memory": defaultMemRequest.String(),
                },
            })
        }
        
        if container.Resources.Limits == nil {
            patches = append(patches, map[string]interface{}{
                "op":   "add",
                "path": fmt.Sprintf("/spec/containers/%d/resources/limits", i),
                "value": map[string]string{
                    "cpu":    defaultCPULimit.String(),
                    "memory": defaultMemLimit.String(),
                },
            })
        }
    }
    
    return patches
}

func (h *Handler) injectSidecar(pod *corev1.Pod) []interface{} {
    var patches []interface{}
    
    // ตรวจสอบ annotation สำหรับ sidecar injection
    if pod.Annotations["sidecar-injector/inject"] != "true" {
        return patches
    }
    
    // ตรวจสอบว่ามี sidecar อยู่แล้วหรือไม่
    for _, c := range pod.Spec.Containers {
        if c.Name == "envoy-proxy" {
            return patches
        }
    }
    
    // Inject Envoy sidecar
    sidecar := corev1.Container{
        Name:  "envoy-proxy",
        Image: "envoyproxy/envoy:v1.28.0",
        Ports: []corev1.ContainerPort{
            {Name: "http", ContainerPort: 8080},
            {Name: "admin", ContainerPort: 9901},
        },
        Resources: corev1.ResourceRequirements{
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("50m"),
                corev1.ResourceMemory: resource.MustParse("64Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("200m"),
                corev1.ResourceMemory: resource.MustParse("256Mi"),
            },
        },
    }
    
    patches = append(patches, map[string]interface{}{
        "op":    "add",
        "path":  "/spec/containers/-",
        "value": sidecar,
    })
    
    return patches
}

func (h *Handler) setSecurityDefaults(pod *corev1.Pod) []interface{} {
    var patches []interface{}
    
    // กำหนด pod-level security context ถ้ายังไม่มี
    if pod.Spec.SecurityContext == nil {
        runAsNonRoot := true
        patches = append(patches, map[string]interface{}{
            "op":   "add",
            "path": "/spec/securityContext",
            "value": map[string]interface{}{
                "runAsNonRoot": runAsNonRoot,
            },
        })
    }
    
    return patches
}

// MutateDeployments handles deployment mutation
func (h *Handler) MutateDeployments(w http.ResponseWriter, r *http.Request) {
    h.serve(w, r, func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
        h.log.Info("Mutating deployment", "namespace", review.Request.Namespace, "name", review.Request.Name)
        
        var deploy appsv1.Deployment
        if err := json.Unmarshal(review.Request.Object.Raw, &deploy); err != nil {
            return newDeniedResponse(400, fmt.Sprintf("Failed to parse deployment: %v", err))
        }
        
        patches := h.mutateDeploymentPatches(&deploy)
        if len(patches) == 0 {
            return newAllowedResponse()
        }
        
        return newPatchResponse(patches)
    })
}

func (h *Handler) mutateDeploymentPatches(deploy *appsv1.Deployment) []interface{} {
    var patches []interface{}
    
    // เพิ่ม recommended labels ถ้ายังไม่มี
    recommendedLabels := map[string]string{
        "app.kubernetes.io/managed-by": "admission-webhook",
    }
    
    if deploy.Labels == nil {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels",
            "value": map[string]string{},
        })
    }
    
    for key, value := range recommendedLabels {
        if _, ok := deploy.Labels[key]; !ok {
            escapedKey := strings.ReplaceAll(key, "/", "~1")
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  fmt.Sprintf("/metadata/labels/%s", escapedKey),
                "value": value,
            })
        }
    }
    
    // กำหนด revisionHistoryLimit ถ้าไม่ได้ระบุ
    if deploy.Spec.RevisionHistoryLimit == nil {
        defaultHistory := int32(5)
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/spec/revisionHistoryLimit",
            "value": defaultHistory,
        })
    }
    
    return patches
}
```

### validate.go

```go
// pkg/webhook/validate.go
package webhook

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    
    admissionv1 "k8s.io/api/admission/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
)

// ValidationResult เก็บผลการ validate
type ValidationResult struct {
    Allowed  bool
    Warnings []string
    Errors   []string
}

// ValidatePods handles pod validation webhook
func (h *Handler) ValidatePods(w http.ResponseWriter, r *http.Request) {
    h.serve(w, r, func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
        h.log.Info("Validating pod", "namespace", review.Request.Namespace)
        
        // Skip validation for delete operations
        if review.Request.Operation == admissionv1.Delete {
            return newAllowedResponse()
        }
        
        var pod corev1.Pod
        if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
            return newDeniedResponse(400, fmt.Sprintf("Failed to parse pod: %v", err))
        }
        
        result := h.validatePod(&pod, review.Request.Namespace)
        
        if !result.Allowed {
            message := strings.Join(result.Errors, "; ")
            return newDeniedResponse(403, message)
        }
        
        // ส่ง warnings ถ้ามี
        response := newAllowedResponse()
        if len(result.Warnings) > 0 {
            response.Warnings = result.Warnings
        }
        
        return response
    })
}

func (h *Handler) validatePod(pod *corev1.Pod, namespace string) ValidationResult {
    result := ValidationResult{Allowed: true}
    
    // Rule 1: ต้องมี resource limits
    for _, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil || container.Resources.Limits.Cpu().IsZero() {
            result.Errors = append(result.Errors, 
                fmt.Sprintf("Container '%s' missing CPU limits", container.Name))
            result.Allowed = false
        }
        if container.Resources.Limits == nil || container.Resources.Limits.Memory().IsZero() {
            result.Errors = append(result.Errors, 
                fmt.Sprintf("Container '%s' missing memory limits", container.Name))
            result.Allowed = false
        }
    }
    
    // Rule 2: ห้ามใช้ :latest tag ใน production
    if namespace == "production" {
        for _, container := range pod.Spec.Containers {
            if strings.HasSuffix(container.Image, ":latest") {
                result.Errors = append(result.Errors,
                    fmt.Sprintf("Container '%s' using 'latest' tag is not allowed in production", container.Name))
                result.Allowed = false
            }
            if !strings.Contains(container.Image, ":") {
                result.Warnings = append(result.Warnings,
                    fmt.Sprintf("Container '%s' image has no tag specified", container.Name))
            }
        }
    }
    
    // Rule 3: ต้องมี required labels
    requiredLabels := []string{"app", "env"}
    for _, label := range requiredLabels {
        if _, ok := pod.Labels[label]; !ok {
            result.Errors = append(result.Errors,
                fmt.Sprintf("Missing required label: %s", label))
            result.Allowed = false
        }
    }
    
    // Rule 4: ห้าม privileged containers
    for _, container := range pod.Spec.Containers {
        if container.SecurityContext != nil && container.SecurityContext.Privileged != nil && 
           *container.SecurityContext.Privileged {
            result.Errors = append(result.Errors,
                fmt.Sprintf("Container '%s' must not be privileged", container.Name))
            result.Allowed = false
        }
    }
    
    // Rule 5: ต้องมี liveness/readiness probes
    for _, container := range pod.Spec.Containers {
        if container.LivenessProbe == nil {
            result.Warnings = append(result.Warnings,
                fmt.Sprintf("Container '%s' has no liveness probe", container.Name))
        }
        if container.ReadinessProbe == nil {
            result.Warnings = append(result.Warnings,
                fmt.Sprintf("Container '%s' has no readiness probe", container.Name))
        }
    }
    
    // Rule 6: ตรวจสอบ host namespaces
    if pod.Spec.HostNetwork {
        result.Errors = append(result.Errors, "Pod must not use host network")
        result.Allowed = false
    }
    if pod.Spec.HostPID {
        result.Errors = append(result.Errors, "Pod must not use host PID namespace")
        result.Allowed = false
    }
    if pod.Spec.HostIPC {
        result.Errors = append(result.Errors, "Pod must not use host IPC namespace")
        result.Allowed = false
    }
    
    return result
}

// ValidateDeployments handles deployment validation webhook
func (h *Handler) ValidateDeployments(w http.ResponseWriter, r *http.Request) {
    h.serve(w, r, func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
        h.log.Info("Validating deployment", "namespace", review.Request.Namespace, "name", review.Request.Name)
        
        if review.Request.Operation == admissionv1.Delete {
            return newAllowedResponse()
        }
        
        var deploy appsv1.Deployment
        if err := json.Unmarshal(review.Request.Object.Raw, &deploy); err != nil {
            return newDeniedResponse(400, fmt.Sprintf("Failed to parse deployment: %v", err))
        }
        
        result := h.validateDeployment(&deploy, review.Request.Namespace)
        
        if !result.Allowed {
            message := strings.Join(result.Errors, "; ")
            return newDeniedResponse(403, message)
        }
        
        response := newAllowedResponse()
        if len(result.Warnings) > 0 {
            response.Warnings = result.Warnings
        }
        
        return response
    })
}

func (h *Handler) validateDeployment(deploy *appsv1.Deployment, namespace string) ValidationResult {
    result := ValidationResult{Allowed: true}
    
    // Rule 1: ต้องมี replicas > 0
    if deploy.Spec.Replicas != nil && *deploy.Spec.Replicas == 0 {
        result.Warnings = append(result.Warnings, "Deployment has 0 replicas")
    }
    
    // Rule 2: ใน production ต้องมี replicas >= 2
    if namespace == "production" && deploy.Spec.Replicas != nil && *deploy.Spec.Replicas < 2 {
        result.Errors = append(result.Errors, "Production deployments must have at least 2 replicas")
        result.Allowed = false
    }
    
    // Rule 3: ต้องมี required labels
    requiredLabels := []string{"app", "env", "team"}
    for _, label := range requiredLabels {
        if _, ok := deploy.Labels[label]; !ok {
            result.Errors = append(result.Errors, fmt.Sprintf("Missing required label: %s", label))
            result.Allowed = false
        }
    }
    
    // Rule 4: ตรวจสอบ strategy
    if deploy.Spec.Strategy.Type == appsv1.RecreateDeploymentStrategyType && namespace == "production" {
        result.Warnings = append(result.Warnings, 
            "Recreate strategy causes downtime - consider using RollingUpdate in production")
    }
    
    return result
}
```

---

## 84.3 Certificate Management

### ใช้ cert-manager สำหรับ TLS

```yaml
# cert-manager.yaml
---
# Certificate สำหรับ webhook server
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: admission-webhook-cert
  namespace: default
spec:
  dnsNames:
    - admission-webhook.default.svc
    - admission-webhook.default.svc.cluster.local
  issuerRef:
    kind: ClusterIssuer
    name: selfsigned-cluster-issuer
  secretName: admission-webhook-tls

---
# ClusterIssuer สำหรับ self-signed certificates
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

### Self-signed Certificate Script

```bash
#!/bin/bash
# generate-certs.sh

WEBHOOK_NAME="admission-webhook"
NAMESPACE="default"
SERVICE="${WEBHOOK_NAME}.${NAMESPACE}.svc"

# สร้าง CA key และ certificate
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=Admission Webhook CA"

# สร้าง server key และ CSR
openssl genrsa -out server.key 2048
cat > server.ext <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${WEBHOOK_NAME}
DNS.2 = ${WEBHOOK_NAME}.${NAMESPACE}
DNS.3 = ${SERVICE}
DNS.4 = ${SERVICE}.cluster.local
EOF

openssl req -new -key server.key -out server.csr \
  -subj "/CN=${SERVICE}" \
  -config server.ext

# Sign certificate ด้วย CA
openssl x509 -req -days 3650 \
  -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt \
  -extensions v3_req \
  -extfile server.ext

# สร้าง Kubernetes secret
kubectl create secret tls admission-webhook-tls \
  --cert=server.crt \
  --key=server.key \
  -n ${NAMESPACE}

# เก็บ CA bundle สำหรับ webhook config
CA_BUNDLE=$(cat ca.crt | base64 | tr -d '\n')
echo "CA Bundle: ${CA_BUNDLE}"

echo "Done! Use the CA bundle in your webhook configuration"
```

---

## 84.4 Deploy Webhook Server

```yaml
# deploy/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
  namespace: default
  labels:
    app: admission-webhook
spec:
  replicas: 2
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      serviceAccountName: admission-webhook
      containers:
        - name: webhook
          image: myregistry/admission-webhook:latest
          args:
            - --port=8443
            - --tls-cert-file=/etc/ssl/certs/tls.crt
            - --tls-private-key-file=/etc/ssl/private/tls.key
          ports:
            - containerPort: 8443
              name: https
          volumeMounts:
            - name: tls
              mountPath: /etc/ssl/certs
              readOnly: true
            - name: tls
              mountPath: /etc/ssl/private
              readOnly: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
      volumes:
        - name: tls
          secret:
            secretName: admission-webhook-tls

---
apiVersion: v1
kind: Service
metadata:
  name: admission-webhook
  namespace: default
spec:
  selector:
    app: admission-webhook
  ports:
    - name: https
      port: 443
      targetPort: 8443

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admission-webhook
  namespace: default

---
# RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admission-webhook
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admission-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admission-webhook
subjects:
  - kind: ServiceAccount
    name: admission-webhook
    namespace: default
```

```yaml
# deploy/webhook-config.yaml
---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: admission-webhook-mutating
  annotations:
    cert-manager.io/inject-ca-from: "default/admission-webhook-cert"
webhooks:
  - name: mutate.pods.admission.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        scope: "Namespaced"
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: /mutate/pods
        port: 443
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Ignore
    timeoutSeconds: 10
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values:
            - kube-system
            - gatekeeper-system
    reinvocationPolicy: Never
  
  - name: mutate.deployments.admission.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["apps"]
        apiVersions: ["v1"]
        resources: ["deployments"]
        scope: "Namespaced"
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: /mutate/deployments
        port: 443
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Ignore
    timeoutSeconds: 10

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: admission-webhook-validating
  annotations:
    cert-manager.io/inject-ca-from: "default/admission-webhook-cert"
webhooks:
  - name: validate.pods.admission.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        scope: "Namespaced"
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: /validate/pods
        port: 443
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
    timeoutSeconds: 10
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values:
            - kube-system
            - gatekeeper-system
  
  - name: validate.deployments.admission.example.com
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["apps"]
        apiVersions: ["v1"]
        resources: ["deployments"]
        scope: "Namespaced"
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: /validate/deployments
        port: 443
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
    timeoutSeconds: 10
```

---

## 84.5 Workshop: Custom Admission Logic

### เป้าหมาย
สร้าง Webhook ที่บังคับใช้ company policies แบบ real-world

### Policy ที่จะ implement:
1. Auto-inject monitoring sidecar ใน specific namespaces
2. Require cost center label ใน production
3. ห้ามใช้ Docker Hub images ใน production
4. ตรวจสอบ resource request/limit ratio

```go
// pkg/webhook/company_policies.go
package webhook

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    
    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
)

// CompanyPoliciesHandler handles company-specific policies
type CompanyPoliciesHandler struct {
    *Handler
    config *PolicyConfig
}

type PolicyConfig struct {
    AllowedRegistries     []string
    MonitoringNamespaces  []string
    RequiredProductionLabels []string
    MaxCPULimitRatio      float64
    MaxMemoryLimitRatio   float64
}

// ValidateCompanyPolicies handles company policy validation
func (h *CompanyPoliciesHandler) ValidateCompanyPolicies(w http.ResponseWriter, r *http.Request) {
    h.serve(w, r, func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
        if review.Request.Operation == admissionv1.Delete {
            return newAllowedResponse()
        }
        
        var pod corev1.Pod
        if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
            return newDeniedResponse(400, fmt.Sprintf("Failed to parse pod: %v", err))
        }
        
        namespace := review.Request.Namespace
        var errors []string
        var warnings []string
        
        // Policy 1: ต้องไม่ใช้ Docker Hub ใน production
        if namespace == "production" {
            for _, container := range pod.Spec.Containers {
                if isDockerHubImage(container.Image) {
                    errors = append(errors, 
                        fmt.Sprintf("Container '%s' uses Docker Hub image. Production must use approved registries: %v",
                            container.Name, h.config.AllowedRegistries))
                }
            }
        }
        
        // Policy 2: Cost center label ใน production
        if namespace == "production" {
            for _, label := range h.config.RequiredProductionLabels {
                if _, ok := pod.Labels[label]; !ok {
                    errors = append(errors, fmt.Sprintf("Production pods must have label: %s", label))
                }
            }
        }
        
        // Policy 3: Resource limit ratio
        for _, container := range pod.Spec.Containers {
            if container.Resources.Requests.Cpu() != nil && container.Resources.Limits.Cpu() != nil {
                reqCPU := container.Resources.Requests.Cpu().AsApproximateFloat64()
                limCPU := container.Resources.Limits.Cpu().AsApproximateFloat64()
                if reqCPU > 0 && limCPU/reqCPU > h.config.MaxCPULimitRatio {
                    warnings = append(warnings,
                        fmt.Sprintf("Container '%s' CPU limit/request ratio (%.1f) exceeds recommended %.1f",
                            container.Name, limCPU/reqCPU, h.config.MaxCPULimitRatio))
                }
            }
        }
        
        if len(errors) > 0 {
            return newDeniedResponse(403, strings.Join(errors, "; "))
        }
        
        response := newAllowedResponse()
        if len(warnings) > 0 {
            response.Warnings = warnings
        }
        return response
    })
}

// MutateWithMonitoring injects monitoring sidecar
func (h *CompanyPoliciesHandler) MutateWithMonitoring(w http.ResponseWriter, r *http.Request) {
    h.serve(w, r, func(review *admissionv1.AdmissionReview) *admissionv1.AdmissionResponse {
        namespace := review.Request.Namespace
        
        // ตรวจสอบว่าต้อง inject ใน namespace นี้หรือไม่
        if !h.shouldInjectMonitoring(namespace) {
            return newAllowedResponse()
        }
        
        var pod corev1.Pod
        if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
            return newDeniedResponse(400, fmt.Sprintf("Failed to parse pod: %v", err))
        }
        
        var patches []interface{}
        
        // ตรวจสอบว่ามี monitoring sidecar อยู่แล้วหรือไม่
        if !hasContainer(&pod, "prometheus-exporter") {
            sidecar := createMonitoringSidecar()
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  "/spec/containers/-",
                "value": sidecar,
            })
        }
        
        // เพิ่ม Prometheus scrape annotations
        if pod.Annotations == nil {
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  "/metadata/annotations",
                "value": map[string]string{},
            })
        }
        
        prometheusAnnotations := map[string]string{
            "prometheus.io/scrape": "true",
            "prometheus.io/port":   "9090",
            "prometheus.io/path":   "/metrics",
        }
        
        for key, value := range prometheusAnnotations {
            if _, ok := pod.Annotations[key]; !ok {
                escapedKey := strings.ReplaceAll(key, "/", "~1")
                patches = append(patches, map[string]interface{}{
                    "op":    "add",
                    "path":  fmt.Sprintf("/metadata/annotations/%s", escapedKey),
                    "value": value,
                })
            }
        }
        
        if len(patches) == 0 {
            return newAllowedResponse()
        }
        
        return newPatchResponse(patches)
    })
}

func (h *CompanyPoliciesHandler) shouldInjectMonitoring(namespace string) bool {
    for _, ns := range h.config.MonitoringNamespaces {
        if ns == namespace || ns == "*" {
            return true
        }
    }
    return false
}

func isDockerHubImage(image string) bool {
    // Docker Hub images ไม่มี registry prefix หรือ มี docker.io/
    if strings.HasPrefix(image, "docker.io/") {
        return true
    }
    // Official images เช่น nginx, mysql
    parts := strings.SplitN(image, "/", 2)
    if len(parts) == 1 {
        return true
    }
    // User images เช่น myuser/myimage แต่ไม่มี . ใน prefix
    if !strings.Contains(parts[0], ".") && !strings.Contains(parts[0], ":") {
        return true
    }
    return false
}

func hasContainer(pod *corev1.Pod, name string) bool {
    for _, c := range pod.Spec.Containers {
        if c.Name == name {
            return true
        }
    }
    return false
}

func createMonitoringSidecar() corev1.Container {
    return corev1.Container{
        Name:  "prometheus-exporter",
        Image: "prom/node-exporter:v1.7.0",
        Ports: []corev1.ContainerPort{
            {Name: "metrics", ContainerPort: 9090},
        },
        Resources: corev1.ResourceRequirements{
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    *parseQuantity("25m"),
                corev1.ResourceMemory: *parseQuantity("32Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    *parseQuantity("100m"),
                corev1.ResourceMemory: *parseQuantity("128Mi"),
            },
        },
        SecurityContext: &corev1.SecurityContext{
            AllowPrivilegeEscalation: boolPtr(false),
            ReadOnlyRootFilesystem:   boolPtr(true),
            RunAsNonRoot:             boolPtr(true),
        },
    }
}

func boolPtr(b bool) *bool {
    return &b
}
```

### Dockerfile

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o webhook ./cmd/webhook/

FROM scratch
COPY --from=builder /build/webhook /webhook
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

ENTRYPOINT ["/webhook"]
```

### ทดสอบ Webhook

```bash
# Deploy webhook
kubectl apply -f deploy/

# ดู logs
kubectl logs -l app=admission-webhook -f

# ทดสอบ mutation
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
  labels:
    app: test
    env: dev
  annotations:
    sidecar-injector/inject: "true"
spec:
  containers:
    - name: nginx
      image: nginx:1.25
EOF

# ดู pod ที่ถูก mutate
kubectl get pod test-pod -o yaml
# ควรเห็น: managed-by label, imagePullPolicy, resource defaults, sidecar container

# ทดสอบ validation failure
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: production
spec:
  containers:
    - name: nginx
      image: nginx:latest  # ไม่อนุญาตใน production
EOF

# ดู webhook metrics
kubectl port-forward svc/admission-webhook 8080:8080
curl http://localhost:8080/metrics
```

---

## 84.6 Webhook Best Practices

### 1. Idempotency

```go
// Mutating webhooks ต้อง idempotent
// เช็คก่อนเสมอว่า patch นั้นจำเป็นหรือไม่

// ❌ ไม่ดี - เพิ่ม label ทุกครั้งแม้จะมีอยู่แล้ว
patches = append(patches, map[string]interface{}{
    "op": "add", "path": "/metadata/labels/foo", "value": "bar",
})

// ✅ ดี - ตรวจสอบก่อน
if _, ok := pod.Labels["foo"]; !ok {
    patches = append(patches, map[string]interface{}{
        "op": "add", "path": "/metadata/labels/foo", "value": "bar",
    })
}
```

### 2. Performance

```go
// กำหนด timeout ที่เหมาะสม
// timeoutSeconds: 5-10 วินาที (default 10, max 30)

// ใช้ namespaceSelector เพื่อลด unnecessary webhook calls
namespaceSelector:
  matchLabels:
    webhook-enabled: "true"

// ใช้ objectSelector สำหรับ selective admission
objectSelector:
  matchLabels:
    admission: "enabled"
```

### 3. High Availability

```yaml
# Deploy multiple replicas
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # ห้ามมี downtime
      maxSurge: 1

# กำหนด Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: admission-webhook-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: admission-webhook
```

### 4. Failure Policy

```yaml
# failurePolicy: Fail
# - ปลอดภัยกว่า แต่ถ้า webhook down จะ block operations
# - ใช้สำหรับ security-critical validations

# failurePolicy: Ignore
# - webhook down ไม่ block operations
# - ใช้สำหรับ best-effort mutations/validations

# Best practice: ใช้ Fail สำหรับ validation, Ignore สำหรับ mutation
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Webhook Architecture**: AdmissionReview format
2. **Mutating Webhooks**: การ patch objects ด้วย JSON Patch
3. **Validating Webhooks**: การ validate และ reject requests
4. **Certificate Management**: cert-manager และ self-signed certs
5. **Deployment**: การ deploy และ configure webhooks
6. **Workshop**: สร้าง company-specific policies
7. **Best Practices**: Idempotency, Performance, HA

บทถัดไปเราจะเรียนรู้เกี่ยวกับ Kubernetes API Extension
