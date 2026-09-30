# Part 85: Kubernetes API Extension

## บทนำ

Kubernetes API Extension ช่วยให้คุณสามารถขยาย Kubernetes API ได้เกินกว่าที่ CRDs ทำได้ ผ่านกลไก API Aggregation Layer ที่ช่วยให้คุณสร้าง Custom API Server ของตัวเองที่ทำงานร่วมกับ Kubernetes API Server

---

## 85.1 วิธีการขยาย Kubernetes API

### สามวิธีหลัก

```
1. Custom Resource Definitions (CRDs)
   - ง่ายที่สุด
   - เก็บข้อมูลใน etcd ของ Kubernetes
   - ใช้สำหรับ domain objects ทั่วไป
   
2. API Aggregation Layer
   - ซับซ้อนกว่า แต่ยืดหยุ่นกว่า
   - มี Custom API Server ของตัวเอง
   - สามารถใช้ storage backend อื่นได้ (ไม่ต้อง etcd)
   - รองรับ custom authentication/authorization
   - เหมาะสำหรับ metrics, logs, และ complex APIs
   
3. Webhooks (Admission Controllers)
   - ใช้สำหรับ validate/mutate ไม่ใช่ extend API
   - ดูบท 83-84
```

### เปรียบเทียบ CRD vs API Aggregation

```
CRDs:
✅ ง่ายต่อการสร้าง
✅ ใช้ etcd ของ Kubernetes
✅ kubectl ทำงานได้ทันที
✅ ไม่ต้องดูแล infrastructure เพิ่ม
❌ ไม่สามารถใช้ custom storage ได้
❌ validation ถูกจำกัด
❌ ไม่รองรับ streaming

API Aggregation:
✅ Custom storage backend
✅ Custom authentication
✅ รองรับ streaming, websockets
✅ Custom validation logic
✅ APIs ที่ซับซ้อนมาก (เช่น /logs, /exec)
❌ ซับซ้อนกว่ามาก
❌ ต้องดูแล API Server เพิ่ม
❌ ต้องจัดการ TLS ด้วยตัวเอง
```

---

## 85.2 API Aggregation Layer

### Architecture

```
kubectl → kube-apiserver → APIService (CRD) → Custom API Server
                       ↑
              (proxy based on APIService)
```

```yaml
# APIService resource
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.metrics.example.com
spec:
  group: metrics.example.com
  version: v1alpha1
  groupPriorityMinimum: 100
  versionPriority: 100
  service:
    name: custom-metrics-apiserver
    namespace: custom-metrics
    port: 443
  caBundle: <base64-encoded-ca-cert>
  insecureSkipTLSVerify: false
```

### ตัวอย่าง: Custom Metrics API Server

```bash
# ติดตั้ง metrics-server (ตัวอย่าง aggregated API)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# ตรวจสอบ
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods
```

---

## 85.3 สร้าง Custom API Server

### โปรเจกต์โครงสร้าง

```
custom-apiserver/
├── cmd/
│   └── server/
│       └── main.go
├── pkg/
│   ├── apis/
│   │   └── custom/
│   │       ├── v1alpha1/
│   │       │   ├── types.go
│   │       │   └── register.go
│   │       ├── install.go
│   │       └── register.go
│   ├── registry/
│   │   └── custom/
│   │       └── storage.go
│   └── apiserver/
│       └── server.go
├── deploy/
│   ├── apiservice.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── go.mod
```

### Types Definition

```go
// pkg/apis/custom/v1alpha1/types.go
package v1alpha1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// AppMetric represents custom application metrics
type AppMetric struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    // Spec defines the desired state
    Spec AppMetricSpec `json:"spec,omitempty"`
    
    // Status defines the observed state
    Status AppMetricStatus `json:"status,omitempty"`
}

type AppMetricSpec struct {
    // Application name to collect metrics for
    AppName string `json:"appName"`
    
    // Metrics to collect
    MetricNames []string `json:"metricNames,omitempty"`
    
    // Collection interval in seconds
    Interval int `json:"interval,omitempty"`
}

type AppMetricStatus struct {
    // Current metric values
    Values map[string]string `json:"values,omitempty"`
    
    // Last collection time
    LastCollected *metav1.Time `json:"lastCollected,omitempty"`
    
    // Collection status
    Phase string `json:"phase,omitempty"`
}

// AppMetricList contains a list of AppMetric
type AppMetricList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []AppMetric `json:"items"`
}
```

### API Server Implementation

```go
// pkg/apiserver/server.go
package apiserver

import (
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/apimachinery/pkg/runtime/serializer"
    "k8s.io/apiserver/pkg/registry/rest"
    genericapiserver "k8s.io/apiserver/pkg/server"
    
    customv1alpha1 "github.com/example/custom-apiserver/pkg/apis/custom/v1alpha1"
    customregistry "github.com/example/custom-apiserver/pkg/registry/custom"
)

// ExtraConfig contains extra configuration for the server
type ExtraConfig struct {
    // Add custom config here
}

// Config contains the configuration for the server
type Config struct {
    GenericConfig *genericapiserver.RecommendedConfig
    ExtraConfig   ExtraConfig
}

// CustomAPIServer contains state for the custom API server
type CustomAPIServer struct {
    GenericAPIServer *genericapiserver.GenericAPIServer
}

type completedConfig struct {
    GenericConfig genericapiserver.CompletedConfig
    ExtraConfig   *ExtraConfig
}

// Complete fills in defaults
func (cfg *Config) Complete() completedConfig {
    c := completedConfig{
        cfg.GenericConfig.Complete(),
        &cfg.ExtraConfig,
    }
    return c
}

// New creates a new CustomAPIServer from the configuration
func (c completedConfig) New() (*CustomAPIServer, error) {
    genericServer, err := c.GenericConfig.New("custom-apiserver", genericapiserver.NewEmptyDelegate())
    if err != nil {
        return nil, err
    }
    
    s := &CustomAPIServer{
        GenericAPIServer: genericServer,
    }
    
    // Install API groups
    apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(
        customv1alpha1.GroupName,
        Scheme,
        runtime.NewParameterCodec(Scheme),
        Codecs,
    )
    
    v1alpha1Storage := map[string]rest.Storage{}
    v1alpha1Storage["appmetrics"] = customregistry.NewStorage()
    v1alpha1Storage["appmetrics/status"] = customregistry.NewStatusStorage()
    
    apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1Storage
    
    if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
        return nil, err
    }
    
    return s, nil
}

var (
    Scheme = runtime.NewScheme()
    Codecs = serializer.NewCodecFactory(Scheme)
)

func init() {
    customv1alpha1.AddToScheme(Scheme)
}
```

### Storage Implementation

```go
// pkg/registry/custom/storage.go
package custom

import (
    "context"
    "fmt"
    "sync"
    
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/apimachinery/pkg/watch"
    "k8s.io/apiserver/pkg/registry/rest"
    
    customv1alpha1 "github.com/example/custom-apiserver/pkg/apis/custom/v1alpha1"
)

// AppMetricStorage provides in-memory storage for AppMetrics
type AppMetricStorage struct {
    mu      sync.RWMutex
    metrics map[string]*customv1alpha1.AppMetric
    
    // Watchers for streaming
    watchers map[chan watch.Event]struct{}
    watchMu  sync.Mutex
}

// NewStorage creates a new in-memory storage
func NewStorage() *AppMetricStorage {
    return &AppMetricStorage{
        metrics:  make(map[string]*customv1alpha1.AppMetric),
        watchers: make(map[chan watch.Event]struct{}),
    }
}

// Implement rest.Storage interface
func (s *AppMetricStorage) New() runtime.Object {
    return &customv1alpha1.AppMetric{}
}

func (s *AppMetricStorage) NewList() runtime.Object {
    return &customv1alpha1.AppMetricList{}
}

func (s *AppMetricStorage) NamespaceScoped() bool {
    return true
}

// Get retrieves an AppMetric by name
func (s *AppMetricStorage) Get(
    ctx context.Context,
    name string,
    options *metav1.GetOptions,
) (runtime.Object, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    key := namespacedKey(ctx, name)
    metric, ok := s.metrics[key]
    if !ok {
        return nil, errors.NewNotFound(schema.GroupResource{
            Group:    customv1alpha1.GroupName,
            Resource: "appmetrics",
        }, name)
    }
    
    return metric.DeepCopyObject(), nil
}

// List retrieves all AppMetrics
func (s *AppMetricStorage) List(
    ctx context.Context,
    options *metainternalversion.ListOptions,
) (runtime.Object, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    namespace := ""
    if ns, ok := request.NamespaceFrom(ctx); ok {
        namespace = ns
    }
    
    list := &customv1alpha1.AppMetricList{}
    for _, metric := range s.metrics {
        if namespace == "" || metric.Namespace == namespace {
            list.Items = append(list.Items, *metric.DeepCopy())
        }
    }
    
    return list, nil
}

// Create creates a new AppMetric
func (s *AppMetricStorage) Create(
    ctx context.Context,
    obj runtime.Object,
    createValidation rest.ValidateObjectFunc,
    options *metav1.CreateOptions,
) (runtime.Object, error) {
    metric, ok := obj.(*customv1alpha1.AppMetric)
    if !ok {
        return nil, fmt.Errorf("not an AppMetric object")
    }
    
    if err := createValidation(ctx, obj); err != nil {
        return nil, err
    }
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    key := namespacedKey(ctx, metric.Name)
    if _, exists := s.metrics[key]; exists {
        return nil, errors.NewAlreadyExists(schema.GroupResource{
            Group:    customv1alpha1.GroupName,
            Resource: "appmetrics",
        }, metric.Name)
    }
    
    metric.CreationTimestamp = metav1.Now()
    metric.ResourceVersion = "1"
    s.metrics[key] = metric.DeepCopy()
    
    // Notify watchers
    s.notifyWatchers(watch.Event{
        Type:   watch.Added,
        Object: metric.DeepCopyObject(),
    })
    
    return metric.DeepCopyObject(), nil
}

// Watch streams changes to AppMetrics
func (s *AppMetricStorage) Watch(
    ctx context.Context,
    options *metainternalversion.ListOptions,
) (watch.Interface, error) {
    ch := make(chan watch.Event, 100)
    
    s.watchMu.Lock()
    s.watchers[ch] = struct{}{}
    s.watchMu.Unlock()
    
    watcher := watch.NewStreamWatcher(nil, nil)
    
    go func() {
        defer func() {
            s.watchMu.Lock()
            delete(s.watchers, ch)
            s.watchMu.Unlock()
            close(ch)
        }()
        
        <-ctx.Done()
    }()
    
    return watcher, nil
}

func (s *AppMetricStorage) notifyWatchers(event watch.Event) {
    s.watchMu.Lock()
    defer s.watchMu.Unlock()
    
    for ch := range s.watchers {
        select {
        case ch <- event:
        default:
            // Channel full, skip
        }
    }
}

func namespacedKey(ctx context.Context, name string) string {
    ns, _ := request.NamespaceFrom(ctx)
    if ns == "" {
        return name
    }
    return ns + "/" + name
}
```

### Main Entry Point

```go
// cmd/server/main.go
package main

import (
    "flag"
    "os"
    
    genericapiserver "k8s.io/apiserver/pkg/server"
    genericoptions "k8s.io/apiserver/pkg/server/options"
    "k8s.io/component-base/cli"
    
    "github.com/example/custom-apiserver/pkg/apiserver"
)

type ServerOptions struct {
    RecommendedOptions *genericoptions.RecommendedOptions
}

func NewServerOptions() *ServerOptions {
    o := &ServerOptions{
        RecommendedOptions: genericoptions.NewRecommendedOptions(
            "/registry/custom.example.com",
            apiserver.Codecs.LegacyCodec(),
        ),
    }
    return o
}

func main() {
    o := NewServerOptions()
    
    flags := flag.NewFlagSet("custom-apiserver", flag.ExitOnError)
    o.RecommendedOptions.AddFlags(flags)
    flags.Parse(os.Args[1:])
    
    // สร้าง config
    config, err := o.RecommendedOptions.ApplyTo(
        &genericapiserver.RecommendedConfig{
            Config: *genericapiserver.NewConfig(apiserver.Codecs),
        },
    )
    if err != nil {
        panic(err)
    }
    
    serverConfig := &apiserver.Config{
        GenericConfig: config,
    }
    
    // สร้าง server
    server, err := serverConfig.Complete().New()
    if err != nil {
        panic(err)
    }
    
    // Start server
    if err := server.GenericAPIServer.PrepareRun().Run(genericapiserver.SetupSignalHandler()); err != nil {
        panic(err)
    }
}
```

---

## 85.4 ติดตั้งและ Deploy

```yaml
# deploy/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-apiserver
  namespace: custom-api-system
  labels:
    app: custom-apiserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-apiserver
  template:
    metadata:
      labels:
        app: custom-apiserver
    spec:
      serviceAccountName: custom-apiserver
      containers:
        - name: apiserver
          image: myregistry/custom-apiserver:latest
          args:
            - --secure-port=8443
            - --tls-cert-file=/etc/ssl/certs/tls.crt
            - --tls-private-key-file=/etc/ssl/private/tls.key
            - --client-ca-file=/etc/kubernetes/pki/ca.crt
          ports:
            - containerPort: 8443
              name: https
          volumeMounts:
            - name: tls
              mountPath: /etc/ssl
              readOnly: true
            - name: kubernetes-ca
              mountPath: /etc/kubernetes/pki
              readOnly: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8443
              scheme: HTTPS
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
      volumes:
        - name: tls
          secret:
            secretName: custom-apiserver-tls
        - name: kubernetes-ca
          hostPath:
            path: /etc/kubernetes/pki
```

```yaml
# deploy/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-apiserver
  namespace: custom-api-system
spec:
  selector:
    app: custom-apiserver
  ports:
    - name: https
      port: 443
      targetPort: 8443

---
# RBAC for the API server
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-apiserver
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["extension-apiserver-authentication"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["admissionregistration.k8s.io"]
    resources: ["mutatingwebhookconfigurations", "validatingwebhookconfigurations"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: custom-apiserver
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: custom-apiserver
subjects:
  - kind: ServiceAccount
    name: custom-apiserver
    namespace: custom-api-system

---
# Allow API server to delegate auth decisions to main API server
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: custom-apiserver-auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: custom-apiserver
    namespace: custom-api-system
```

```yaml
# deploy/apiservice.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.custom.example.com
spec:
  group: custom.example.com
  version: v1alpha1
  groupPriorityMinimum: 100
  versionPriority: 100
  service:
    name: custom-apiserver
    namespace: custom-api-system
    port: 443
  caBundle: <base64-encoded-ca-bundle>
  # หรือ
  insecureSkipTLSVerify: true  # สำหรับ dev only
```

```bash
# Deploy
kubectl apply -f deploy/

# ตรวจสอบ APIService
kubectl get apiservice v1alpha1.custom.example.com
kubectl describe apiservice v1alpha1.custom.example.com

# ทดสอบ API
kubectl get appmetrics
kubectl api-resources | grep custom.example.com

# ดู API discovery
kubectl get --raw /apis/custom.example.com
kubectl get --raw /apis/custom.example.com/v1alpha1
```

---

## 85.5 Workshop: Extend Kubernetes API

### เป้าหมาย
สร้าง custom API server สำหรับ Application Configuration Management

### ขั้นตอนที่ 1: สร้าง Types

```go
// pkg/apis/config/v1alpha1/types.go
package v1alpha1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// AppConfig represents application configuration
type AppConfig struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   AppConfigSpec   `json:"spec,omitempty"`
    Status AppConfigStatus `json:"status,omitempty"`
}

type AppConfigSpec struct {
    // Application this config belongs to
    App string `json:"app"`
    
    // Environment (dev, staging, production)
    Environment string `json:"environment"`
    
    // Configuration data
    Data map[string]string `json:"data,omitempty"`
    
    // Secret references
    SecretRefs []SecretRef `json:"secretRefs,omitempty"`
    
    // Validation rules
    ValidationRules []ValidationRule `json:"validationRules,omitempty"`
}

type SecretRef struct {
    Name string `json:"name"`
    Key  string `json:"key"`
    // Target key in data
    TargetKey string `json:"targetKey"`
}

type ValidationRule struct {
    // Key to validate
    Key string `json:"key"`
    // Regex pattern
    Pattern string `json:"pattern,omitempty"`
    // Required flag
    Required bool `json:"required,omitempty"`
}

type AppConfigStatus struct {
    // Current phase
    Phase string `json:"phase,omitempty"`
    // Validation result
    Valid bool `json:"valid,omitempty"`
    // Validation errors
    ValidationErrors []string `json:"validationErrors,omitempty"`
    // Hash of current config
    ConfigHash string `json:"configHash,omitempty"`
    // Last updated timestamp
    LastUpdated *metav1.Time `json:"lastUpdated,omitempty"`
    // Number of apps using this config
    ConsumerCount int `json:"consumerCount,omitempty"`
}

type AppConfigList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []AppConfig `json:"items"`
}
```

### ขั้นตอนที่ 2: Custom Storage with Validation

```go
// pkg/registry/config/storage.go
package config

import (
    "context"
    "crypto/sha256"
    "fmt"
    "regexp"
    "sync"
    "time"
    
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/watch"
    "k8s.io/apiserver/pkg/registry/rest"
    
    configv1alpha1 "github.com/example/custom-apiserver/pkg/apis/config/v1alpha1"
)

type AppConfigStorage struct {
    mu      sync.RWMutex
    configs map[string]*configv1alpha1.AppConfig
    watches []chan watch.Event
}

func NewStorage() *AppConfigStorage {
    return &AppConfigStorage{
        configs: make(map[string]*configv1alpha1.AppConfig),
    }
}

func (s *AppConfigStorage) Create(
    ctx context.Context,
    obj runtime.Object,
    createValidation rest.ValidateObjectFunc,
    options *metav1.CreateOptions,
) (runtime.Object, error) {
    config, ok := obj.(*configv1alpha1.AppConfig)
    if !ok {
        return nil, fmt.Errorf("not an AppConfig")
    }
    
    // Custom validation
    if err := s.validateConfig(config); err != nil {
        return nil, errors.NewInvalid(
            configv1alpha1.SchemeGroupVersion.WithKind("AppConfig").GroupKind(),
            config.Name,
            nil,
        )
    }
    
    // Calculate config hash
    config.Status.ConfigHash = calculateHash(config.Spec.Data)
    config.Status.Phase = "Active"
    config.Status.Valid = true
    config.Status.LastUpdated = &metav1.Time{Time: time.Now()}
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    key := fmt.Sprintf("%s/%s", config.Namespace, config.Name)
    s.configs[key] = config.DeepCopy()
    
    // Notify watches
    s.notifyWatches(watch.Event{Type: watch.Added, Object: config.DeepCopyObject()})
    
    return config.DeepCopyObject(), nil
}

func (s *AppConfigStorage) validateConfig(config *configv1alpha1.AppConfig) error {
    // Validate required fields
    if config.Spec.App == "" {
        return fmt.Errorf("app field is required")
    }
    
    validEnvs := map[string]bool{"development": true, "staging": true, "production": true}
    if !validEnvs[config.Spec.Environment] {
        return fmt.Errorf("invalid environment: %s", config.Spec.Environment)
    }
    
    // Validate data against validation rules
    for _, rule := range config.Spec.ValidationRules {
        value, exists := config.Spec.Data[rule.Key]
        
        if rule.Required && !exists {
            return fmt.Errorf("required key '%s' not found in data", rule.Key)
        }
        
        if exists && rule.Pattern != "" {
            matched, err := regexp.MatchString(rule.Pattern, value)
            if err != nil {
                return fmt.Errorf("invalid pattern for key '%s': %v", rule.Key, err)
            }
            if !matched {
                return fmt.Errorf("value for key '%s' does not match pattern '%s'", rule.Key, rule.Pattern)
            }
        }
    }
    
    return nil
}

func calculateHash(data map[string]string) string {
    h := sha256.New()
    for k, v := range data {
        h.Write([]byte(k + "=" + v + "\n"))
    }
    return fmt.Sprintf("%x", h.Sum(nil))[:16]
}

func (s *AppConfigStorage) notifyWatches(event watch.Event) {
    for _, ch := range s.watches {
        select {
        case ch <- event:
        default:
        }
    }
}
```

### ขั้นตอนที่ 3: Custom Endpoints

```go
// pkg/apiserver/custom_endpoints.go
package apiserver

import (
    "encoding/json"
    "net/http"
    
    "k8s.io/apiserver/pkg/endpoints/handlers/responsewriters"
    genericapiserver "k8s.io/apiserver/pkg/server"
)

// RegisterCustomRoutes เพิ่ม custom routes
func RegisterCustomRoutes(s *CustomAPIServer) {
    s.GenericAPIServer.Handler.GoRestfulContainer.Add(
        // Custom route สำหรับ validate config
        gorestful.WebService{}.
            Path("/apis/config.example.com/v1alpha1/validate").
            Route(gorestful.WebService{}.
                POST("/").
                To(validateConfigHandler),
            ),
    )
}

func validateConfigHandler(req *restful.Request, resp *restful.Response) {
    // Parse request body
    var config struct {
        Data map[string]string `json:"data"`
        Rules []struct {
            Key     string `json:"key"`
            Pattern string `json:"pattern"`
        } `json:"rules"`
    }
    
    if err := req.ReadEntity(&config); err != nil {
        resp.WriteError(http.StatusBadRequest, err)
        return
    }
    
    // Validate
    var errors []string
    for _, rule := range config.Rules {
        // ... validation logic
    }
    
    result := map[string]interface{}{
        "valid":  len(errors) == 0,
        "errors": errors,
    }
    
    resp.WriteEntity(result)
}
```

### ขั้นตอนที่ 4: ทดสอบ

```bash
# Deploy custom API server
kubectl apply -f deploy/

# ดู API availability
kubectl get apiservice v1alpha1.config.example.com

# สร้าง AppConfig
kubectl apply -f - <<EOF
apiVersion: config.example.com/v1alpha1
kind: AppConfig
metadata:
  name: my-app-config
  namespace: default
spec:
  app: my-application
  environment: staging
  data:
    DATABASE_HOST: "db.example.com"
    DATABASE_PORT: "5432"
    LOG_LEVEL: "info"
    MAX_CONNECTIONS: "100"
  validationRules:
    - key: DATABASE_PORT
      pattern: "^[0-9]{1,5}$"
      required: true
    - key: LOG_LEVEL
      pattern: "^(debug|info|warn|error)$"
      required: true
EOF

# ดู AppConfigs
kubectl get appconfigs
kubectl describe appconfig my-app-config

# ทดสอบ validation endpoint
curl -k -X POST https://<api-server>/apis/config.example.com/v1alpha1/validate \
  -H "Content-Type: application/json" \
  -d '{"data":{"key":"value"},"rules":[{"key":"key","pattern":"^value$"}]}'

# Watch changes
kubectl get appconfigs --watch
```

---

## 85.6 aggregated API Server vs CRDs Decision Matrix

```
เลือก CRDs เมื่อ:
✅ ต้องการเก็บ simple structured data
✅ ต้องการ CRUD operations เท่านั้น
✅ ไม่ต้องการ custom storage
✅ ทีมขาด Go experience
✅ ต้องการ time-to-market เร็ว

เลือก Aggregated API Server เมื่อ:
✅ ต้องการ custom storage (database, files)
✅ ต้องการ streaming APIs
✅ ต้องการ complex validation logic
✅ ต้องการ metrics APIs (เช่น metrics-server)
✅ ต้องการ custom authentication/authorization
✅ ต้องการ subresources ที่ซับซ้อน (เช่น /exec, /logs)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **API Extension Methods**: CRDs vs API Aggregation
2. **API Aggregation Layer**: Architecture และ APIService
3. **Custom API Server**: การสร้างด้วย Go
4. **Storage Implementation**: Custom storage backend
5. **Custom Endpoints**: เพิ่ม routes พิเศษ
6. **Workshop**: AppConfig Management API
7. **Decision Matrix**: เลือกวิธีที่เหมาะสม

บทถัดไปเราจะเรียนรู้เกี่ยวกับ Multi-cluster Architectures
