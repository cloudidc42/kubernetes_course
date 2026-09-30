# Part 81: Custom Resource Definitions (CRDs)

## บทนำ

Custom Resource Definitions (CRDs) เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Kubernetes ที่ช่วยให้คุณสามารถขยาย Kubernetes API ได้โดยการสร้าง Resource ประเภทใหม่ที่ตรงกับความต้องการของแอปพลิเคชันของคุณ

CRDs ช่วยให้คุณสามารถ:
- กำหนด Resource ประเภทใหม่ที่ Kubernetes ไม่ได้มีมาให้
- จัดการ Custom Resources ด้วย kubectl เหมือน Built-in Resources
- รวมกับ Kubernetes RBAC, Admission Controllers และระบบอื่นๆ
- สร้าง Domain-Specific Languages (DSL) สำหรับ Infrastructure

---

## 81.1 ความเข้าใจพื้นฐาน CRDs

### Kubernetes API และ Resources

ก่อนที่จะเข้าใจ CRDs ต้องเข้าใจว่า Kubernetes API ทำงานอย่างไร:

```
Kubernetes API Server
├── /api/v1 (Core API Group)
│   ├── pods
│   ├── services
│   ├── configmaps
│   └── ...
├── /apis/apps/v1
│   ├── deployments
│   ├── statefulsets
│   └── ...
├── /apis/networking.k8s.io/v1
│   ├── ingresses
│   └── networkpolicies
└── /apis/<custom-group>/<version>  ← CRDs อยู่ที่นี่
    └── <custom-resources>
```

### โครงสร้าง CRD

CRD มีส่วนประกอบหลักดังนี้:

```yaml
# ตัวอย่างโครงสร้าง CRD พื้นฐาน
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: <plural>.<group>          # ชื่อต้องเป็น <plural>.<group>
spec:
  group: <group>                  # API Group
  versions:                       # รายการ versions ที่รองรับ
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema: {}       # Schema validation
  scope: Namespaced | Cluster    # ขอบเขต
  names:
    plural: <plural>              # ชื่อพหูพจน์
    singular: <singular>          # ชื่อเอกพจน์
    kind: <Kind>                  # ชื่อ Kind (PascalCase)
    shortNames: []                # ชื่อย่อ (optional)
```

---

## 81.2 สร้าง CRD แรกของคุณ

### ตัวอย่าง: Database CRD

สมมติว่าเราต้องการสร้าง Resource สำหรับจัดการ Database Instances:

```yaml
# database-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
  labels:
    app.kubernetes.io/name: database-operator
    app.kubernetes.io/version: "1.0"
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - engine
                - version
              properties:
                engine:
                  type: string
                  enum: ["mysql", "postgresql", "mongodb"]
                  description: "Database engine type"
                version:
                  type: string
                  description: "Database engine version"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
                  description: "Number of replicas"
                storage:
                  type: object
                  properties:
                    size:
                      type: string
                      pattern: '^[0-9]+[MGTP]i$'
                      default: "10Gi"
                    storageClass:
                      type: string
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Failed", "Terminating"]
                message:
                  type: string
                readyReplicas:
                  type: integer
                endpoint:
                  type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
                      reason:
                        type: string
                      message:
                        type: string
      subresources:
        status: {}              # เปิดใช้ Status Subresource
        scale:                  # เปิดใช้ Scale Subresource
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Version
          type: string
          jsonPath: .spec.version
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
    categories:
      - all
      - data
```

### ติดตั้ง CRD

```bash
# ติดตั้ง CRD
kubectl apply -f database-crd.yaml

# ตรวจสอบ CRD
kubectl get crd databases.example.com

# ดูรายละเอียด
kubectl describe crd databases.example.com

# ตรวจสอบ API Groups
kubectl api-resources | grep example.com

# ทดสอบ schema ของ CRD
kubectl explain database
kubectl explain database.spec
kubectl explain database.spec.engine
```

### สร้าง Custom Resource

```yaml
# my-database.yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-mysql-db
  namespace: production
  labels:
    app: myapp
    env: production
  annotations:
    description: "Production MySQL database"
spec:
  engine: mysql
  version: "8.0"
  replicas: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
```

```bash
# สร้าง Resource
kubectl apply -f my-database.yaml

# ดูรายการ
kubectl get databases
kubectl get db  # ใช้ shortname

# ดูรายละเอียด
kubectl describe database my-mysql-db

# แก้ไข
kubectl edit database my-mysql-db

# ลบ
kubectl delete database my-mysql-db
```

---

## 81.3 OpenAPI Schema Validation

### Schema Validation พื้นฐาน

Schema Validation ใช้ OpenAPI v3 Schema เพื่อ validate ข้อมูลก่อนที่จะถูก persist ใน etcd:

```yaml
# schema-validation-example.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.apps.example.com
spec:
  group: apps.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            spec:
              type: object
              required: ["image", "port"]
              properties:
                # String validation
                image:
                  type: string
                  minLength: 1
                  maxLength: 255
                  description: "Docker image for the webapp"
                
                # String with pattern
                version:
                  type: string
                  pattern: '^v[0-9]+\.[0-9]+\.[0-9]+$'
                  description: "Version in semver format (e.g., v1.2.3)"
                
                # Integer with range
                port:
                  type: integer
                  minimum: 1
                  maximum: 65535
                  description: "Port number"
                
                # Number
                timeout:
                  type: number
                  minimum: 0.1
                  maximum: 300.0
                  default: 30.0
                
                # Boolean
                enableTLS:
                  type: boolean
                  default: false
                
                # Enum
                environment:
                  type: string
                  enum: ["development", "staging", "production"]
                  default: "development"
                
                # Array with items validation
                allowedOrigins:
                  type: array
                  items:
                    type: string
                    format: uri
                  minItems: 0
                  maxItems: 20
                  uniqueItems: true
                
                # Object with nested properties
                healthCheck:
                  type: object
                  properties:
                    path:
                      type: string
                      default: "/health"
                    intervalSeconds:
                      type: integer
                      minimum: 5
                      maximum: 300
                      default: 30
                    timeoutSeconds:
                      type: integer
                      minimum: 1
                      maximum: 60
                      default: 5
                    failureThreshold:
                      type: integer
                      minimum: 1
                      maximum: 10
                      default: 3
                
                # Map/Dictionary
                labels:
                  type: object
                  additionalProperties:
                    type: string
                  maxProperties: 20
                
                # Nested object
                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                      pattern: '^[0-9]+m?$'
                    memory:
                      type: string
                      pattern: '^[0-9]+[KMGT]i?$'
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
```

### Advanced Schema: oneOf, anyOf, allOf

```yaml
# advanced-schema.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: storagebackends.storage.example.com
spec:
  group: storage.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                # ใช้ oneOf เพื่อกำหนดว่าต้องเลือกอันใดอันหนึ่ง
                backend:
                  type: object
                  oneOf:
                    - required: ["s3"]
                      properties:
                        type:
                          type: string
                          enum: ["s3"]
                        s3:
                          type: object
                          required: ["bucket", "region"]
                          properties:
                            bucket:
                              type: string
                            region:
                              type: string
                            endpoint:
                              type: string
                    - required: ["gcs"]
                      properties:
                        type:
                          type: string
                          enum: ["gcs"]
                        gcs:
                          type: object
                          required: ["bucket", "project"]
                          properties:
                            bucket:
                              type: string
                            project:
                              type: string
                    - required: ["azure"]
                      properties:
                        type:
                          type: string
                          enum: ["azure"]
                        azure:
                          type: object
                          required: ["container", "accountName"]
                          properties:
                            container:
                              type: string
                            accountName:
                              type: string
                
                # ใช้ allOf เพื่อรวม schemas หลายอัน
                config:
                  allOf:
                    - type: object
                      properties:
                        maxConnections:
                          type: integer
                          minimum: 1
                    - type: object
                      properties:
                        timeout:
                          type: integer
                          minimum: 0
  scope: Namespaced
  names:
    plural: storagebackends
    singular: storagebackend
    kind: StorageBackend
```

### X-Kubernetes Extensions

```yaml
# x-kubernetes-extensions.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.platform.example.com
spec:
  group: platform.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                # x-kubernetes-int-or-string: ยอมรับ integer หรือ string
                replicas:
                  x-kubernetes-int-or-string: true
                
                # x-kubernetes-preserve-unknown-fields: เก็บ fields ที่ไม่รู้จัก
                annotations:
                  type: object
                  x-kubernetes-preserve-unknown-fields: true
                
                # x-kubernetes-list-type: กำหนดประเภทของ list
                ports:
                  type: array
                  x-kubernetes-list-type: map
                  x-kubernetes-list-map-keys:
                    - port
                    - protocol
                  items:
                    type: object
                    required:
                      - port
                      - protocol
                    properties:
                      port:
                        type: integer
                      protocol:
                        type: string
                        enum: ["TCP", "UDP"]
                      name:
                        type: string
                
                # x-kubernetes-embedded-resource: เก็บ Kubernetes object
                podTemplate:
                  type: object
                  x-kubernetes-embedded-resource: true
                  x-kubernetes-preserve-unknown-fields: true
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
```

---

## 81.4 CRD Versioning

### การจัดการ Multiple Versions

เมื่อ API ของคุณพัฒนาขึ้น คุณจำเป็นต้องจัดการหลาย versions:

```yaml
# versioned-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.platform.example.com
spec:
  group: platform.example.com
  versions:
    # Version v1alpha1 - เวอร์ชันเก่า (ยังคงใช้งานได้)
    - name: v1alpha1
      served: true      # ยังคง serve API นี้
      storage: false    # ไม่ใช้ storage (v1 ใช้แทน)
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
      deprecated: true  # บอกว่า deprecated
      deprecationWarning: "v1alpha1 is deprecated, use v1 instead"
    
    # Version v1beta1
    - name: v1beta1
      served: true
      storage: false
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                    memory:
                      type: string
    
    # Version v1 - เวอร์ชันปัจจุบัน (stable)
    - name: v1
      served: true
      storage: true     # version นี้ใช้ใน storage
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            spec:
              type: object
              required: ["image"]
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  default: 1
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                strategy:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: ["Recreate", "RollingUpdate"]
                      default: "RollingUpdate"
                    rollingUpdate:
                      type: object
                      properties:
                        maxSurge:
                          x-kubernetes-int-or-string: true
                        maxUnavailable:
                          x-kubernetes-int-or-string: true
            status:
              type: object
              properties:
                phase:
                  type: string
                availableReplicas:
                  type: integer
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  
  conversion:
    strategy: Webhook    # ใช้ Webhook สำหรับ conversion
    webhook:
      clientConfig:
        service:
          name: application-conversion-webhook
          namespace: default
          path: /convert
      conversionReviewVersions: ["v1", "v1beta1"]
  
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
    shortNames:
      - app
```

### Conversion Webhook Server (Go)

```go
// main.go - Conversion Webhook Server
package main

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"
    
    "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
)

var scheme = runtime.NewScheme()
var codecs = serializer.NewCodecFactory(scheme)

func main() {
    http.HandleFunc("/convert", convertHandler)
    fmt.Println("Conversion webhook server listening on :8443")
    if err := http.ListenAndServeTLS(":8443", "/etc/ssl/certs/tls.crt", "/etc/ssl/private/tls.key", nil); err != nil {
        panic(err)
    }
}

func convertHandler(w http.ResponseWriter, r *http.Request) {
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        http.Error(w, fmt.Sprintf("Failed to read body: %v", err), http.StatusBadRequest)
        return
    }
    
    // Decode ConversionReview
    convertReview := v1.ConversionReview{}
    if err := json.Unmarshal(body, &convertReview); err != nil {
        http.Error(w, fmt.Sprintf("Failed to unmarshal: %v", err), http.StatusBadRequest)
        return
    }
    
    // Perform conversion
    convertReview.Response = convertObjects(convertReview.Request)
    convertReview.Response.UID = convertReview.Request.UID
    
    // Encode response
    responseBytes, err := json.Marshal(convertReview)
    if err != nil {
        http.Error(w, fmt.Sprintf("Failed to marshal response: %v", err), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.Write(responseBytes)
}

func convertObjects(req *v1.ConversionRequest) *v1.ConversionResponse {
    response := &v1.ConversionResponse{}
    convertedObjects := []runtime.RawExtension{}
    
    for _, obj := range req.Objects {
        // Parse the object
        var appObj map[string]interface{}
        if err := json.Unmarshal(obj.Raw, &appObj); err != nil {
            response.Result = metav1.Status{
                Status:  "Failure",
                Message: fmt.Sprintf("Failed to unmarshal object: %v", err),
            }
            return response
        }
        
        // Determine source and target versions
        sourceVersion := appObj["apiVersion"].(string)
        targetVersion := req.DesiredAPIVersion
        
        // Perform conversion
        convertedObj, err := convert(appObj, sourceVersion, targetVersion)
        if err != nil {
            response.Result = metav1.Status{
                Status:  "Failure",
                Message: fmt.Sprintf("Failed to convert: %v", err),
            }
            return response
        }
        
        convertedObj["apiVersion"] = targetVersion
        
        raw, err := json.Marshal(convertedObj)
        if err != nil {
            response.Result = metav1.Status{
                Status:  "Failure",
                Message: fmt.Sprintf("Failed to marshal converted object: %v", err),
            }
            return response
        }
        
        convertedObjects = append(convertedObjects, runtime.RawExtension{Raw: raw})
    }
    
    response.ConvertedObjects = convertedObjects
    response.Result = metav1.Status{Status: "Success"}
    return response
}

func convert(obj map[string]interface{}, from, to string) (map[string]interface{}, error) {
    spec, ok := obj["spec"].(map[string]interface{})
    if !ok {
        return obj, nil
    }
    
    switch {
    case from == "platform.example.com/v1alpha1" && to == "platform.example.com/v1":
        // Convert v1alpha1 -> v1
        newSpec := map[string]interface{}{
            "image":    spec["image"],
            "replicas": spec["replicas"],
            "resources": map[string]interface{}{
                "requests": map[string]interface{}{
                    "cpu":    "100m",
                    "memory": "128Mi",
                },
            },
            "strategy": map[string]interface{}{
                "type": "RollingUpdate",
            },
        }
        obj["spec"] = newSpec
        
    case from == "platform.example.com/v1" && to == "platform.example.com/v1alpha1":
        // Convert v1 -> v1alpha1
        newSpec := map[string]interface{}{
            "image":    spec["image"],
            "replicas": spec["replicas"],
        }
        obj["spec"] = newSpec
    }
    
    return obj, nil
}
```

---

## 81.5 Subresources

### Status Subresource

```yaml
# crd-with-status.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                desired:
                  type: integer
            status:
              type: object
              properties:
                current:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      message:
                        type: string
      subresources:
        status: {}    # เปิดใช้ status subresource
  scope: Namespaced
  names:
    plural: myresources
    singular: myresource
    kind: MyResource
```

```bash
# อัปเดต Status ผ่าน Status Subresource
kubectl patch myresource my-resource --subresource=status --type=merge \
  -p '{"status":{"current":3,"conditions":[{"type":"Ready","status":"True"}]}}'

# ดู Status
kubectl get myresource my-resource -o jsonpath='{.status}'
```

### Scale Subresource

```bash
# ใช้ kubectl scale
kubectl scale myresource my-resource --replicas=5

# ดู HPA ที่ใช้ custom resource
kubectl autoscale myresource my-resource --min=2 --max=10 --cpu-percent=80
```

---

## 81.6 Printer Columns

### Custom Printer Columns

```yaml
# printer-columns.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: services.platform.example.com
spec:
  group: platform.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                url:
                  type: string
                region:
                  type: string
                tier:
                  type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                health:
                  type: string
                lastUpdated:
                  type: string
                  format: date-time
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
          description: "Docker image"
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
          description: "Number of replicas"
        - name: URL
          type: string
          jsonPath: .spec.url
          description: "Service URL"
        - name: Region
          type: string
          jsonPath: .spec.region
          priority: 1           # แสดงเมื่อใช้ -o wide
          description: "Deployment region"
        - name: Tier
          type: string
          jsonPath: .spec.tier
          priority: 1
          description: "Service tier"
        - name: Phase
          type: string
          jsonPath: .status.phase
          description: "Current phase"
        - name: Health
          type: string
          jsonPath: .status.health
          description: "Health status"
        - name: Last Updated
          type: date
          jsonPath: .status.lastUpdated
          description: "Last update time"
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: services
    singular: service
    kind: Service
```

```bash
# ดูด้วย columns ปกติ
kubectl get services.platform.example.com

# ดูด้วย -o wide (แสดง priority: 1)
kubectl get services.platform.example.com -o wide
```

---

## 81.7 Finalizers และ Owner References

### Finalizers

Finalizers ช่วยให้คุณควบคุมการลบ Resource:

```yaml
# resource-with-finalizer.yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-database
  finalizers:
    - databases.example.com/cleanup      # Finalizer
    - databases.example.com/backup       # อาจมีหลาย finalizers
spec:
  engine: mysql
  version: "8.0"
```

```go
// controller ที่จัดการ finalizer
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &examplev1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // ตรวจสอบว่ากำลังถูกลบหรือไม่
    if !db.DeletionTimestamp.IsZero() {
        // Resource กำลังถูกลบ - ทำ cleanup
        if containsString(db.Finalizers, "databases.example.com/cleanup") {
            // ทำการ cleanup
            if err := r.cleanupDatabase(ctx, db); err != nil {
                return ctrl.Result{}, err
            }
            
            // ลบ finalizer เมื่อ cleanup เสร็จ
            db.Finalizers = removeString(db.Finalizers, "databases.example.com/cleanup")
            if err := r.Update(ctx, db); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }
    
    // ตรวจสอบว่ามี finalizer หรือยัง
    if !containsString(db.Finalizers, "databases.example.com/cleanup") {
        db.Finalizers = append(db.Finalizers, "databases.example.com/cleanup")
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // Logic หลักของ controller
    return r.reconcileDatabase(ctx, db)
}
```

### Owner References

```go
// สร้าง resource ที่มี owner reference
func (r *DatabaseReconciler) createService(ctx context.Context, db *examplev1.Database) error {
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name + "-svc",
            Namespace: db.Namespace,
            // ตั้ง owner reference ให้ service ถูกลบเมื่อ Database ถูกลบ
            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(db, examplev1.GroupVersion.WithKind("Database")),
            },
        },
        Spec: corev1.ServiceSpec{
            Selector: map[string]string{
                "app": db.Name,
            },
            Ports: []corev1.ServicePort{
                {
                    Port:       3306,
                    TargetPort: intstr.FromInt(3306),
                },
            },
        },
    }
    
    return r.Create(ctx, svc)
}
```

---

## 81.8 Workshop: สร้าง Custom Resource สำหรับ Application Platform

### เป้าหมาย
สร้าง CRD สำหรับ `WebApplication` resource ที่จัดการ web application deployment อย่างครบถ้วน

### ขั้นตอนที่ 1: สร้าง CRD

```yaml
# webapp-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapplications.apps.workshop.io
  annotations:
    description: "CRD for managing web applications"
spec:
  group: apps.workshop.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            metadata:
              type: object
            spec:
              type: object
              required: ["name", "image"]
              properties:
                name:
                  type: string
                  minLength: 1
                  maxLength: 63
                  pattern: '^[a-z0-9][a-z0-9-]*[a-z0-9]$'
                  description: "Application name (lowercase, alphanumeric, dashes)"
                
                image:
                  type: string
                  minLength: 1
                  description: "Container image"
                
                replicas:
                  type: integer
                  minimum: 0
                  maximum: 100
                  default: 1
                  description: "Number of replicas"
                
                port:
                  type: integer
                  minimum: 1
                  maximum: 65535
                  default: 8080
                  description: "Application port"
                
                env:
                  type: array
                  items:
                    type: object
                    required: ["name"]
                    properties:
                      name:
                        type: string
                      value:
                        type: string
                      valueFrom:
                        type: object
                        properties:
                          secretKeyRef:
                            type: object
                            required: ["name", "key"]
                            properties:
                              name:
                                type: string
                              key:
                                type: string
                          configMapKeyRef:
                            type: object
                            required: ["name", "key"]
                            properties:
                              name:
                                type: string
                              key:
                                type: string
                
                ingress:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    host:
                      type: string
                    path:
                      type: string
                      default: "/"
                    tls:
                      type: boolean
                      default: false
                    tlsSecret:
                      type: string
                
                autoscaling:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    minReplicas:
                      type: integer
                      minimum: 1
                      default: 1
                    maxReplicas:
                      type: integer
                      minimum: 1
                      default: 10
                    targetCPUUtilization:
                      type: integer
                      minimum: 1
                      maximum: 100
                      default: 70
                
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                          default: "100m"
                        memory:
                          type: string
                          default: "128Mi"
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                          default: "500m"
                        memory:
                          type: string
                          default: "512Mi"
                
                healthCheck:
                  type: object
                  properties:
                    path:
                      type: string
                      default: "/health"
                    initialDelaySeconds:
                      type: integer
                      default: 10
                    periodSeconds:
                      type: integer
                      default: 30
                
                strategy:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: ["Recreate", "RollingUpdate", "Canary", "BlueGreen"]
                      default: "RollingUpdate"
                    canary:
                      type: object
                      properties:
                        weight:
                          type: integer
                          minimum: 0
                          maximum: 100
                    blueGreen:
                      type: object
                      properties:
                        activeService:
                          type: string
                        previewService:
                          type: string
            
            status:
              type: object
              x-kubernetes-preserve-unknown-fields: true
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Deploying", "Running", "Degraded", "Failed"]
                message:
                  type: string
                availableReplicas:
                  type: integer
                desiredReplicas:
                  type: integer
                url:
                  type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
                      reason:
                        type: string
                      message:
                        type: string
                lastDeployedAt:
                  type: string
                  format: date-time
      
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.availableReplicas
      
      additionalPrinterColumns:
        - name: Application
          type: string
          jsonPath: .spec.name
        - name: Image
          type: string
          jsonPath: .spec.image
        - name: Replicas
          type: string
          jsonPath: .spec.replicas
        - name: Available
          type: integer
          jsonPath: .status.availableReplicas
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: URL
          type: string
          jsonPath: .status.url
          priority: 1
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  
  scope: Namespaced
  names:
    plural: webapplications
    singular: webapplication
    kind: WebApplication
    shortNames:
      - wapp
      - webapp
    categories:
      - all
      - apps
```

### ขั้นตอนที่ 2: ติดตั้งและทดสอบ CRD

```bash
# ติดตั้ง CRD
kubectl apply -f webapp-crd.yaml

# ตรวจสอบ
kubectl get crd webapplications.apps.workshop.io
kubectl describe crd webapplications.apps.workshop.io

# ดู API resource
kubectl api-resources | grep workshop

# ดู schema
kubectl explain webapplication
kubectl explain webapplication.spec
kubectl explain webapplication.spec.ingress
kubectl explain webapplication.spec.autoscaling
```

### ขั้นตอนที่ 3: สร้าง Custom Resources ตัวอย่าง

```yaml
# sample-webapp-simple.yaml
apiVersion: apps.workshop.io/v1
kind: WebApplication
metadata:
  name: my-simple-app
  namespace: default
  labels:
    team: backend
    env: development
spec:
  name: my-simple-app
  image: nginx:1.25
  replicas: 2
  port: 80
```

```yaml
# sample-webapp-full.yaml
apiVersion: apps.workshop.io/v1
kind: WebApplication
metadata:
  name: my-full-app
  namespace: production
  labels:
    team: frontend
    env: production
    version: "1.5.0"
  annotations:
    deployment.kubernetes.io/revision: "3"
spec:
  name: my-full-app
  image: myregistry.io/myapp:1.5.0
  replicas: 3
  port: 3000
  
  env:
    - name: NODE_ENV
      value: production
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-url
    - name: REDIS_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: redis-host
  
  ingress:
    enabled: true
    host: myapp.example.com
    path: /
    tls: true
    tlsSecret: myapp-tls-secret
  
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilization: 70
  
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  
  healthCheck:
    path: /api/health
    initialDelaySeconds: 20
    periodSeconds: 30
  
  strategy:
    type: RollingUpdate
```

```bash
# สร้าง resources
kubectl apply -f sample-webapp-simple.yaml
kubectl apply -f sample-webapp-full.yaml

# ดูรายการ
kubectl get webapplications
kubectl get webapp  # ใช้ shortname

# ดูรายละเอียด
kubectl describe webapp my-full-app -n production

# อัปเดต replica count
kubectl scale webapp my-simple-app --replicas=5

# อัปเดต status (ปกติ controller จะทำ)
kubectl patch webapp my-simple-app --subresource=status --type=merge \
  -p '{"status":{"phase":"Running","availableReplicas":5}}'

# ดู all category
kubectl get all | grep webapp

# ลบ
kubectl delete webapp my-simple-app
```

### ขั้นตอนที่ 4: ทดสอบ Schema Validation

```bash
# ทดสอบ validation ที่ผิด - ควร fail
kubectl apply -f - <<EOF
apiVersion: apps.workshop.io/v1
kind: WebApplication
metadata:
  name: invalid-app
spec:
  name: Invalid Name!   # ผิด pattern (มี space และ !)
  image: nginx
  replicas: 200         # เกิน maximum 100
EOF

# ทดสอบ validation ที่ถูก
kubectl apply -f - <<EOF
apiVersion: apps.workshop.io/v1
kind: WebApplication
metadata:
  name: valid-app
spec:
  name: valid-app
  image: nginx:latest
  replicas: 3
  port: 8080
EOF
```

### ขั้นตอนที่ 5: ทดสอบ RBAC กับ CRD

```yaml
# webapp-rbac.yaml
---
# Role สำหรับดู webapplications เท่านั้น
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: webapp-viewer
  namespace: default
rules:
  - apiGroups: ["apps.workshop.io"]
    resources: ["webapplications"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps.workshop.io"]
    resources: ["webapplications/status"]
    verbs: ["get"]

---
# Role สำหรับจัดการ webapplications
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: webapp-manager
  namespace: default
rules:
  - apiGroups: ["apps.workshop.io"]
    resources: ["webapplications"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps.workshop.io"]
    resources: ["webapplications/status"]
    verbs: ["get", "update", "patch"]
  - apiGroups: ["apps.workshop.io"]
    resources: ["webapplications/scale"]
    verbs: ["get", "update"]
```

```bash
kubectl apply -f webapp-rbac.yaml

# ดู permissions
kubectl auth can-i create webapplications --as=system:serviceaccount:default:default
```

---

## 81.9 CRD Best Practices

### 1. การตั้งชื่อ

```yaml
# ✅ ถูกต้อง
metadata:
  name: databases.example.com    # format: <plural>.<group>

spec:
  group: example.com
  names:
    plural: databases
    singular: database
    kind: Database                # PascalCase
    shortNames:
      - db                        # lowercase

# ❌ ผิด
metadata:
  name: Database.example.com    # ต้องเป็น lowercase plural
```

### 2. Versioning Strategy

```
v1alpha1 → v1alpha2 → v1beta1 → v1beta2 → v1
                                              ↑
                                          Stable/GA
```

### 3. Schema Best Practices

```yaml
# ✅ ดี - มี required fields, descriptions, constraints
spec:
  type: object
  required: ["image"]
  properties:
    image:
      type: string
      minLength: 1
      description: "Container image (required)"
    replicas:
      type: integer
      minimum: 1
      maximum: 100
      default: 1
      description: "Number of pod replicas"

# ❌ ไม่ดี - ไม่มี validation
spec:
  x-kubernetes-preserve-unknown-fields: true  # อย่าใช้ใน spec ถ้าไม่จำเป็น
```

### 4. Status Conditions Pattern

```go
// ใช้ standard conditions format
type Condition struct {
    Type               string      `json:"type"`
    Status             string      `json:"status"`
    LastTransitionTime metav1.Time `json:"lastTransitionTime"`
    Reason             string      `json:"reason"`
    Message            string      `json:"message"`
}

// Condition Types ควรใช้ชื่อมาตรฐาน
const (
    ConditionReady    = "Ready"
    ConditionDegraded = "Degraded"
    ConditionProgressing = "Progressing"
)
```

---

## 81.10 Debugging CRDs

### Commands สำหรับ Debug

```bash
# ตรวจสอบ CRD validation errors
kubectl get crd databases.example.com -o yaml | grep -A 50 "schema:"

# ดู events ที่เกี่ยวกับ CRD
kubectl get events --field-selector reason=FailedValidation

# ตรวจสอบ API server logs
kubectl logs -n kube-system kube-apiserver-$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') | grep "databases.example.com"

# ทดสอบว่า schema ถูกต้อง
kubectl apply --dry-run=server -f my-resource.yaml

# ดู CRD conditions
kubectl get crd databases.example.com -o jsonpath='{.status.conditions}' | python3 -m json.tool

# ดู established CRDs
kubectl get crd -o custom-columns='NAME:.metadata.name,ESTABLISHED:.status.conditions[?(@.type=="Established")].status'

# ดู stored versions
kubectl get crd databases.example.com -o jsonpath='{.status.storedVersions}'
```

### Common Issues และ Solutions

```bash
# ปัญหา: CRD ไม่ถูก establish
# แก้: ตรวจสอบ syntax ของ CRD
kubectl get crd myresource.example.com -o yaml | grep -A 5 "conditions:"

# ปัญหา: Validation ไม่ทำงาน
# แก้: ตรวจสอบว่าใช้ apiextensions.k8s.io/v1 ไม่ใช่ v1beta1
kubectl get crd myresource.example.com -o jsonpath='{.apiVersion}'

# ปัญหา: Cannot delete CRD เพราะมี resources อยู่
# แก้: ลบ resources ก่อน
kubectl get myresource -A  # ดูทุก namespace
kubectl delete myresource --all -A
kubectl delete crd myresource.example.com
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CRD พื้นฐาน**: โครงสร้าง, การสร้าง, และการใช้งาน
2. **OpenAPI Schema Validation**: การ validate ข้อมูลด้วย schema
3. **CRD Versioning**: การจัดการหลาย versions และ conversion
4. **Subresources**: Status และ Scale subresources
5. **Printer Columns**: การกำหนด columns สำหรับ kubectl output
6. **Finalizers & Owner References**: การควบคุม lifecycle
7. **Workshop**: สร้าง WebApplication CRD ที่ครบถ้วน

CRDs เป็นพื้นฐานสำคัญสำหรับการสร้าง Kubernetes Operators ซึ่งเราจะเรียนรู้ในบทถัดไป

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes CRD Documentation](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [OpenAPI v3 Schema Specification](https://swagger.io/specification/)
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
