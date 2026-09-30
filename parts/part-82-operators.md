# Part 82: Kubernetes Operators

## บทนำ

Kubernetes Operators คือ pattern สำหรับการจัดการ stateful applications โดยใช้ domain-specific knowledge ที่ถูก encode ลงใน software แทนที่จะต้องพึ่งพา human operator

Operators ใช้ Custom Resources เพื่อจัดการ applications และ components เปรียบเสมือน "สมองของ SRE" ที่รู้วิธีการ deploy, scale, backup, และ recover applications โดยอัตโนมัติ

---

## 82.1 Operator Pattern

### ความเข้าใจ Controller Loop

Operators ทำงานโดยใช้ Control Loop (Reconciliation Loop):

```
Observe → Analyze → Act
   ↑                  |
   └──────────────────┘
```

```
ต้องการสถานะ (Desired State)
           ↓
    Reconciliation Loop
           ↓
   สถานะปัจจุบัน ≠ สถานะที่ต้องการ?
           ↓ Yes
    ทำให้เข้าสู่สถานะที่ต้องการ
           ↓
   สถานะปัจจุบัน = สถานะที่ต้องการ ✓
```

### Controller vs Operator

```
Controller:
- เป็น generic pattern
- จัดการ Built-in resources (Deployment, Service ฯลฯ)
- ไม่มี domain knowledge

Operator:
- เป็น Domain-specific Controller
- จัดการ Custom Resources
- มี domain knowledge เกี่ยวกับ application
- ตัวอย่าง: Prometheus Operator, MySQL Operator, Vault Operator
```

### Operator Maturity Model

```
Level 1: Basic Install
  - Automated application provisioning and configuration management

Level 2: Seamless Upgrades  
  - Patch and minor version upgrades supported

Level 3: Full Lifecycle
  - App lifecycle, storage lifecycle (backup, failure recovery)

Level 4: Deep Insights
  - Metrics, alerts, log processing, workload analysis

Level 5: Auto Pilot
  - Horizontal/vertical scaling, auto config tuning, abnormal detection
```

---

## 82.2 Operator Framework

### สิ่งที่ต้องติดตั้ง

```bash
# ติดตั้ง operator-sdk
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.33.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
chmod +x operator-sdk_${OS}_${ARCH}
sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk

# ตรวจสอบ
operator-sdk version

# ติดตั้ง kubebuilder (alternative)
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/

# ติดตั้ง controller-gen
go install sigs.k8s.io/controller-tools/cmd/controller-gen@latest

# ติดตั้ง kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

---

## 82.3 สร้าง Operator ด้วย Go

### สร้างโปรเจกต์

```bash
# สร้างไดเรกทอรี
mkdir database-operator && cd database-operator

# Initialize project
operator-sdk init \
  --domain=example.com \
  --repo=github.com/example/database-operator \
  --plugins=go/v4

# สร้าง API และ Controller
operator-sdk create api \
  --group=data \
  --version=v1 \
  --kind=Database \
  --resource=true \
  --controller=true
```

### โครงสร้างไฟล์ที่ได้

```
database-operator/
├── api/
│   └── v1/
│       ├── database_types.go      # Custom Resource Type
│       ├── groupversion_info.go   # API registration
│       └── zz_generated.deepcopy.go
├── config/
│   ├── crd/                       # CRD manifests
│   ├── default/                   # Default kustomize config
│   ├── manager/                   # Operator deployment
│   ├── rbac/                      # RBAC rules
│   └── samples/                   # CR samples
├── controllers/
│   └── database_controller.go     # Controller logic
├── main.go                        # Entry point
├── go.mod
└── Makefile
```

### Types (api/v1/database_types.go)

```go
// api/v1/database_types.go
package v1

import (
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DatabaseSpec defines the desired state of Database
type DatabaseSpec struct {
    // +kubebuilder:validation:Enum=mysql;postgresql;mongodb
    // +kubebuilder:default=mysql
    Engine string `json:"engine"`
    
    // +kubebuilder:validation:MinLength=1
    Version string `json:"version"`
    
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas,omitempty"`
    
    Storage DatabaseStorage `json:"storage,omitempty"`
    
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`
    
    // +optional
    BackupSchedule string `json:"backupSchedule,omitempty"`
    
    // +optional
    MaintenanceWindow *MaintenanceWindow `json:"maintenanceWindow,omitempty"`
}

type DatabaseStorage struct {
    // +kubebuilder:validation:Pattern=`^[0-9]+[MGTP]i$`
    // +kubebuilder:default="10Gi"
    Size string `json:"size,omitempty"`
    
    // +optional
    StorageClass string `json:"storageClass,omitempty"`
}

type MaintenanceWindow struct {
    // +kubebuilder:validation:Pattern=`^[0-9]{2}:[0-9]{2}$`
    StartTime string `json:"startTime"`
    
    // DurationMinutes specifies maintenance duration
    // +kubebuilder:validation:Minimum=30
    // +kubebuilder:validation:Maximum=240
    DurationMinutes int `json:"durationMinutes"`
    
    // +kubebuilder:validation:Enum=Monday;Tuesday;Wednesday;Thursday;Friday;Saturday;Sunday
    DayOfWeek string `json:"dayOfWeek"`
}

// DatabasePhase represents the current phase of a database
// +kubebuilder:validation:Enum=Pending;Initializing;Running;Updating;Failed;Terminating
type DatabasePhase string

const (
    DatabasePhasePending      DatabasePhase = "Pending"
    DatabasePhaseInitializing DatabasePhase = "Initializing"
    DatabasePhaseRunning      DatabasePhase = "Running"
    DatabasePhaseUpdating     DatabasePhase = "Updating"
    DatabasePhaseFailed       DatabasePhase = "Failed"
    DatabasePhaseTerminating  DatabasePhase = "Terminating"
)

// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
    // +optional
    Phase DatabasePhase `json:"phase,omitempty"`
    
    // +optional
    Message string `json:"message,omitempty"`
    
    // +optional
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
    
    // +optional
    Endpoint string `json:"endpoint,omitempty"`
    
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    
    // +optional
    LastBackupTime *metav1.Time `json:"lastBackupTime,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas
// +kubebuilder:printcolumn:name="Engine",type=string,JSONPath=`.spec.engine`
// +kubebuilder:printcolumn:name="Version",type=string,JSONPath=`.spec.version`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Endpoint",type=string,JSONPath=`.status.endpoint`,priority=1
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

// Database is the Schema for the databases API
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// DatabaseList contains a list of Database
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Database `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Database{}, &DatabaseList{})
}
```

### Controller (controllers/database_controller.go)

```go
// controllers/database_controller.go
package controllers

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    datav1 "github.com/example/database-operator/api/v1"
)

const (
    databaseFinalizer = "data.example.com/finalizer"
    requeueAfter      = 30 * time.Second
)

// DatabaseReconciler reconciles a Database object
type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=data.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=data.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=data.example.com,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=persistentvolumeclaims,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create;update;patch;delete

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Reconciling Database", "database", req.NamespacedName)

    // 1. ดึง Database resource
    db := &datav1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        if errors.IsNotFound(err) {
            logger.Info("Database resource not found. Ignoring since object must be deleted")
            return ctrl.Result{}, nil
        }
        logger.Error(err, "Failed to get Database")
        return ctrl.Result{}, err
    }

    // 2. จัดการ deletion
    if !db.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, db)
    }

    // 3. เพิ่ม finalizer ถ้ายังไม่มี
    if !controllerutil.ContainsFinalizer(db, databaseFinalizer) {
        controllerutil.AddFinalizer(db, databaseFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }

    // 4. Reconcile logic หลัก
    return r.reconcileDatabase(ctx, db)
}

func (r *DatabaseReconciler) reconcileDatabase(ctx context.Context, db *datav1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // อัปเดต phase เป็น Initializing ถ้ายังไม่มี phase
    if db.Status.Phase == "" {
        if err := r.updateStatus(ctx, db, datav1.DatabasePhaseInitializing, "Initializing database"); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Reconcile Service
    if err := r.reconcileService(ctx, db); err != nil {
        logger.Error(err, "Failed to reconcile Service")
        r.updateStatus(ctx, db, datav1.DatabasePhaseFailed, fmt.Sprintf("Failed to reconcile Service: %v", err))
        return ctrl.Result{}, err
    }

    // Reconcile ConfigMap
    if err := r.reconcileConfigMap(ctx, db); err != nil {
        logger.Error(err, "Failed to reconcile ConfigMap")
        return ctrl.Result{}, err
    }

    // Reconcile StatefulSet
    sts, err := r.reconcileStatefulSet(ctx, db)
    if err != nil {
        logger.Error(err, "Failed to reconcile StatefulSet")
        r.updateStatus(ctx, db, datav1.DatabasePhaseFailed, fmt.Sprintf("Failed to reconcile StatefulSet: %v", err))
        return ctrl.Result{}, err
    }

    // อัปเดต status ตาม StatefulSet
    if sts.Status.ReadyReplicas == db.Spec.Replicas {
        endpoint := fmt.Sprintf("%s.%s.svc.cluster.local:3306", db.Name, db.Namespace)
        if err := r.updateStatusWithEndpoint(ctx, db, datav1.DatabasePhaseRunning, "Database is running", endpoint); err != nil {
            return ctrl.Result{}, err
        }
    } else {
        if err := r.updateStatus(ctx, db, datav1.DatabasePhaseInitializing, 
            fmt.Sprintf("Waiting for replicas: %d/%d ready", sts.Status.ReadyReplicas, db.Spec.Replicas)); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{RequeueAfter: requeueAfter}, nil
    }

    logger.Info("Database reconciled successfully", "database", db.Name)
    return ctrl.Result{RequeueAfter: requeueAfter}, nil
}

func (r *DatabaseReconciler) reconcileDelete(ctx context.Context, db *datav1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    if controllerutil.ContainsFinalizer(db, databaseFinalizer) {
        // ทำ cleanup ก่อนลบ
        logger.Info("Performing cleanup for Database", "database", db.Name)
        
        // อัปเดต phase
        r.updateStatus(ctx, db, datav1.DatabasePhaseTerminating, "Terminating database")
        
        // ทำ backup ก่อนลบ (optional)
        if err := r.performFinalBackup(ctx, db); err != nil {
            logger.Error(err, "Failed to perform final backup")
            // ไม่ return error เพื่อให้ deletion ดำเนินต่อได้
        }
        
        // ลบ finalizer
        controllerutil.RemoveFinalizer(db, databaseFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *datav1.Database) (*appsv1.StatefulSet, error) {
    logger := log.FromContext(ctx)
    
    // กำหนด image ตาม engine
    image := r.getDatabaseImage(db.Spec.Engine, db.Spec.Version)
    
    // กำหนด labels
    labels := map[string]string{
        "app":                            db.Name,
        "app.kubernetes.io/name":         db.Name,
        "app.kubernetes.io/managed-by":   "database-operator",
        "data.example.com/database":      db.Name,
        "data.example.com/engine":        db.Spec.Engine,
    }
    
    // กำหนด StatefulSet ที่ต้องการ
    desiredSts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
            Labels:    labels,
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas:    &db.Spec.Replicas,
            ServiceName: db.Name,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "database",
                            Image: image,
                            Ports: []corev1.ContainerPort{
                                {
                                    ContainerPort: r.getDatabasePort(db.Spec.Engine),
                                    Name:          "db",
                                },
                            },
                            Resources: db.Spec.Resources,
                            Env:       r.getDatabaseEnv(db),
                            VolumeMounts: []corev1.VolumeMount{
                                {
                                    Name:      "data",
                                    MountPath: "/var/lib/database",
                                },
                            },
                            ReadinessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    Exec: &corev1.ExecAction{
                                        Command: r.getReadinessCommand(db.Spec.Engine),
                                    },
                                },
                                InitialDelaySeconds: 30,
                                PeriodSeconds:       10,
                                FailureThreshold:    3,
                            },
                        },
                    },
                },
            },
            VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
                {
                    ObjectMeta: metav1.ObjectMeta{
                        Name: "data",
                    },
                    Spec: corev1.PersistentVolumeClaimSpec{
                        AccessModes: []corev1.PersistentVolumeAccessMode{
                            corev1.ReadWriteOnce,
                        },
                        Resources: corev1.VolumeResourceRequirements{
                            Requests: corev1.ResourceList{
                                corev1.ResourceStorage: resource.MustParse(db.Spec.Storage.Size),
                            },
                        },
                    },
                },
            },
        },
    }
    
    // ตั้ง owner reference
    if err := controllerutil.SetControllerReference(db, desiredSts, r.Scheme); err != nil {
        return nil, err
    }
    
    // ตรวจสอบว่ามี StatefulSet อยู่แล้วหรือไม่
    existingSts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, existingSts)
    if err != nil {
        if errors.IsNotFound(err) {
            // สร้างใหม่
            logger.Info("Creating StatefulSet", "name", desiredSts.Name)
            if err := r.Create(ctx, desiredSts); err != nil {
                return nil, err
            }
            return desiredSts, nil
        }
        return nil, err
    }
    
    // อัปเดตถ้าจำเป็น
    if *existingSts.Spec.Replicas != db.Spec.Replicas || 
       existingSts.Spec.Template.Spec.Containers[0].Image != image {
        logger.Info("Updating StatefulSet", "name", existingSts.Name)
        existingSts.Spec.Replicas = &db.Spec.Replicas
        existingSts.Spec.Template.Spec.Containers[0].Image = image
        existingSts.Spec.Template.Spec.Containers[0].Resources = db.Spec.Resources
        if err := r.Update(ctx, existingSts); err != nil {
            return nil, err
        }
    }
    
    return existingSts, nil
}

func (r *DatabaseReconciler) reconcileService(ctx context.Context, db *datav1.Database) error {
    labels := map[string]string{
        "app": db.Name,
        "data.example.com/database": db.Name,
    }
    
    desiredSvc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
            Labels:    labels,
        },
        Spec: corev1.ServiceSpec{
            Selector:  labels,
            ClusterIP: "None",  // Headless service สำหรับ StatefulSet
            Ports: []corev1.ServicePort{
                {
                    Name: "db",
                    Port: r.getDatabasePort(db.Spec.Engine),
                },
            },
        },
    }
    
    if err := controllerutil.SetControllerReference(db, desiredSvc, r.Scheme); err != nil {
        return err
    }
    
    existingSvc := &corev1.Service{}
    err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, existingSvc)
    if err != nil {
        if errors.IsNotFound(err) {
            return r.Create(ctx, desiredSvc)
        }
        return err
    }
    
    return nil
}

func (r *DatabaseReconciler) reconcileConfigMap(ctx context.Context, db *datav1.Database) error {
    config := r.getDatabaseConfig(db)
    
    desiredCM := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name + "-config",
            Namespace: db.Namespace,
        },
        Data: config,
    }
    
    if err := controllerutil.SetControllerReference(db, desiredCM, r.Scheme); err != nil {
        return err
    }
    
    existingCM := &corev1.ConfigMap{}
    err := r.Get(ctx, types.NamespacedName{Name: desiredCM.Name, Namespace: db.Namespace}, existingCM)
    if err != nil {
        if errors.IsNotFound(err) {
            return r.Create(ctx, desiredCM)
        }
        return err
    }
    
    existingCM.Data = config
    return r.Update(ctx, existingCM)
}

func (r *DatabaseReconciler) updateStatus(ctx context.Context, db *datav1.Database, phase datav1.DatabasePhase, message string) error {
    db.Status.Phase = phase
    db.Status.Message = message
    
    // อัปเดต condition
    condition := metav1.Condition{
        Type:               "Ready",
        LastTransitionTime: metav1.Now(),
        Message:            message,
    }
    
    if phase == datav1.DatabasePhaseRunning {
        condition.Status = metav1.ConditionTrue
        condition.Reason = "DatabaseRunning"
    } else if phase == datav1.DatabasePhaseFailed {
        condition.Status = metav1.ConditionFalse
        condition.Reason = "DatabaseFailed"
    } else {
        condition.Status = metav1.ConditionFalse
        condition.Reason = string(phase)
    }
    
    meta.SetStatusCondition(&db.Status.Conditions, condition)
    
    return r.Status().Update(ctx, db)
}

func (r *DatabaseReconciler) updateStatusWithEndpoint(ctx context.Context, db *datav1.Database, phase datav1.DatabasePhase, message, endpoint string) error {
    db.Status.Endpoint = endpoint
    return r.updateStatus(ctx, db, phase, message)
}

func (r *DatabaseReconciler) performFinalBackup(ctx context.Context, db *datav1.Database) error {
    // TODO: Implement backup logic
    log.FromContext(ctx).Info("Performing final backup", "database", db.Name)
    return nil
}

// Helper methods
func (r *DatabaseReconciler) getDatabaseImage(engine, version string) string {
    switch engine {
    case "mysql":
        return fmt.Sprintf("mysql:%s", version)
    case "postgresql":
        return fmt.Sprintf("postgres:%s", version)
    case "mongodb":
        return fmt.Sprintf("mongo:%s", version)
    default:
        return fmt.Sprintf("mysql:%s", version)
    }
}

func (r *DatabaseReconciler) getDatabasePort(engine string) int32 {
    switch engine {
    case "mysql":
        return 3306
    case "postgresql":
        return 5432
    case "mongodb":
        return 27017
    default:
        return 3306
    }
}

func (r *DatabaseReconciler) getDatabaseEnv(db *datav1.Database) []corev1.EnvVar {
    switch db.Spec.Engine {
    case "mysql":
        return []corev1.EnvVar{
            {
                Name: "MYSQL_ROOT_PASSWORD",
                ValueFrom: &corev1.EnvVarSource{
                    SecretKeyRef: &corev1.SecretKeySelector{
                        LocalObjectReference: corev1.LocalObjectReference{
                            Name: db.Name + "-secret",
                        },
                        Key: "root-password",
                    },
                },
            },
        }
    case "postgresql":
        return []corev1.EnvVar{
            {
                Name: "POSTGRES_PASSWORD",
                ValueFrom: &corev1.EnvVarSource{
                    SecretKeyRef: &corev1.SecretKeySelector{
                        LocalObjectReference: corev1.LocalObjectReference{
                            Name: db.Name + "-secret",
                        },
                        Key: "password",
                    },
                },
            },
        }
    default:
        return nil
    }
}

func (r *DatabaseReconciler) getReadinessCommand(engine string) []string {
    switch engine {
    case "mysql":
        return []string{"mysqladmin", "ping", "-h", "localhost"}
    case "postgresql":
        return []string{"pg_isready", "-U", "postgres"}
    case "mongodb":
        return []string{"mongosh", "--eval", "db.adminCommand('ping')"}
    default:
        return []string{"echo", "ok"}
    }
}

func (r *DatabaseReconciler) getDatabaseConfig(db *datav1.Database) map[string]string {
    switch db.Spec.Engine {
    case "mysql":
        return map[string]string{
            "my.cnf": fmt.Sprintf(`[mysqld]
max_connections=200
innodb_buffer_pool_size=256M
default_authentication_plugin=mysql_native_password
`),
        }
    case "postgresql":
        return map[string]string{
            "postgresql.conf": fmt.Sprintf(`
max_connections=200
shared_buffers=256MB
effective_cache_size=768MB
`),
        }
    default:
        return map[string]string{}
    }
}

// SetupWithManager sets up the controller with the Manager.
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&datav1.Database{}).
        Owns(&appsv1.StatefulSet{}).  // Watch StatefulSets ที่ owned โดย Database
        Owns(&corev1.Service{}).      // Watch Services ที่ owned โดย Database
        Owns(&corev1.ConfigMap{}).    // Watch ConfigMaps ที่ owned โดย Database
        Complete(r)
}
```

### main.go

```go
// main.go
package main

import (
    "flag"
    "os"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    metricsserver "sigs.k8s.io/controller-runtime/pkg/metrics/server"

    datav1 "github.com/example/database-operator/api/v1"
    "github.com/example/database-operator/controllers"
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(datav1.AddToScheme(scheme))
}

func main() {
    var metricsAddr string
    var enableLeaderElection bool
    var probeAddr string
    
    flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false, "Enable leader election for controller manager.")
    
    opts := zap.Options{
        Development: true,
    }
    opts.BindFlags(flag.CommandLine)
    flag.Parse()
    
    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))
    
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
        Metrics: metricsserver.Options{
            BindAddress: metricsAddr,
        },
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "database-operator-leader",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
    
    if err = (&controllers.DatabaseReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Database")
        os.Exit(1)
    }
    
    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }
    
    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

### Build และ Deploy Operator

```bash
# Generate CRD manifests
make generate
make manifests

# Build Docker image
make docker-build IMG=myregistry/database-operator:v1.0.0

# Push
make docker-push IMG=myregistry/database-operator:v1.0.0

# Deploy to cluster
make deploy IMG=myregistry/database-operator:v1.0.0

# หรือ install CRDs แยก
make install

# Run locally (for development)
make run

# ดู logs
kubectl logs -n database-operator-system deploy/database-operator-controller-manager -f
```

---

## 82.4 สร้าง Operator ด้วย Python

### Kopf Framework

```bash
# ติดตั้ง kopf
pip install kopf kubernetes
```

```python
# operator.py - Python Operator ด้วย kopf
import kopf
import kubernetes
import logging
from kubernetes import client as k8s_client

# Load in-cluster config หรือ local config
try:
    kubernetes.config.load_incluster_config()
except:
    kubernetes.config.load_kube_config()

logger = logging.getLogger(__name__)

# ===== Handler สำหรับสร้าง Database =====
@kopf.on.create('example.com', 'v1', 'databases')
def create_database(spec, name, namespace, logger, **kwargs):
    """จัดการการสร้าง Database resource"""
    logger.info(f"Creating database: {name} in namespace: {namespace}")
    
    engine = spec.get('engine', 'mysql')
    version = spec.get('version', 'latest')
    replicas = spec.get('replicas', 1)
    storage_size = spec.get('storage', {}).get('size', '10Gi')
    
    apps_v1 = k8s_client.AppsV1Api()
    core_v1 = k8s_client.CoreV1Api()
    
    # สร้าง Service
    service = create_service(name, namespace, engine)
    try:
        core_v1.create_namespaced_service(namespace=namespace, body=service)
        logger.info(f"Created Service: {name}")
    except k8s_client.rest.ApiException as e:
        if e.status != 409:  # 409 = Already Exists
            raise kopf.PermanentError(f"Failed to create Service: {e}")
    
    # สร้าง StatefulSet
    statefulset = create_statefulset(name, namespace, engine, version, replicas, storage_size)
    try:
        apps_v1.create_namespaced_stateful_set(namespace=namespace, body=statefulset)
        logger.info(f"Created StatefulSet: {name}")
    except k8s_client.rest.ApiException as e:
        if e.status != 409:
            raise kopf.PermanentError(f"Failed to create StatefulSet: {e}")
    
    # Return status update
    return {
        'phase': 'Initializing',
        'message': f'Creating {engine} {version} database',
        'endpoint': f'{name}.{namespace}.svc.cluster.local:{get_port(engine)}'
    }


# ===== Handler สำหรับอัปเดต Database =====
@kopf.on.update('example.com', 'v1', 'databases')
def update_database(spec, name, namespace, old, new, diff, logger, **kwargs):
    """จัดการการอัปเดต Database resource"""
    logger.info(f"Updating database: {name}")
    
    apps_v1 = k8s_client.AppsV1Api()
    
    # ตรวจสอบว่า replicas เปลี่ยนหรือไม่
    new_replicas = new['spec'].get('replicas', 1)
    old_replicas = old['spec'].get('replicas', 1) if old else new_replicas
    
    if new_replicas != old_replicas:
        logger.info(f"Scaling from {old_replicas} to {new_replicas} replicas")
        patch_body = {'spec': {'replicas': new_replicas}}
        try:
            apps_v1.patch_namespaced_stateful_set(
                name=name,
                namespace=namespace,
                body=patch_body
            )
        except k8s_client.rest.ApiException as e:
            raise kopf.TemporaryError(f"Failed to scale StatefulSet: {e}", delay=30)
    
    return {'phase': 'Updating', 'message': 'Updating database configuration'}


# ===== Handler สำหรับลบ Database =====
@kopf.on.delete('example.com', 'v1', 'databases')
def delete_database(spec, name, namespace, logger, **kwargs):
    """จัดการการลบ Database resource"""
    logger.info(f"Deleting database: {name}")
    
    # ทำ backup ก่อนลบ
    perform_backup(name, namespace, logger)
    
    # Kubernetes จะลบ owned resources อัตโนมัติเพราะ owner references
    logger.info(f"Database {name} deletion initiated")


# ===== Timer สำหรับ health check =====
@kopf.timer('example.com', 'v1', 'databases', interval=60)
def check_health(spec, name, namespace, status, patch, logger, **kwargs):
    """ตรวจสอบ health ของ Database ทุก 60 วินาที"""
    apps_v1 = k8s_client.AppsV1Api()
    
    try:
        sts = apps_v1.read_namespaced_stateful_set(name=name, namespace=namespace)
        ready_replicas = sts.status.ready_replicas or 0
        desired_replicas = spec.get('replicas', 1)
        
        if ready_replicas == desired_replicas:
            patch.status['phase'] = 'Running'
            patch.status['readyReplicas'] = ready_replicas
            patch.status['message'] = f'All {ready_replicas} replicas are ready'
        else:
            patch.status['phase'] = 'Degraded'
            patch.status['readyReplicas'] = ready_replicas
            patch.status['message'] = f'{ready_replicas}/{desired_replicas} replicas ready'
            
    except k8s_client.rest.ApiException as e:
        logger.error(f"Failed to check health: {e}")
        patch.status['phase'] = 'Failed'
        patch.status['message'] = str(e)


# ===== Helper Functions =====
def create_service(name, namespace, engine):
    """สร้าง Service object"""
    return {
        'apiVersion': 'v1',
        'kind': 'Service',
        'metadata': {
            'name': name,
            'namespace': namespace,
            'labels': {'app': name, 'managed-by': 'database-operator'}
        },
        'spec': {
            'clusterIP': 'None',
            'selector': {'app': name},
            'ports': [{'name': 'db', 'port': get_port(engine)}]
        }
    }


def create_statefulset(name, namespace, engine, version, replicas, storage_size):
    """สร้าง StatefulSet object"""
    image = f"{engine}:{version}" if engine != 'postgresql' else f"postgres:{version}"
    
    return {
        'apiVersion': 'apps/v1',
        'kind': 'StatefulSet',
        'metadata': {
            'name': name,
            'namespace': namespace,
            'labels': {'app': name, 'managed-by': 'database-operator'}
        },
        'spec': {
            'replicas': replicas,
            'serviceName': name,
            'selector': {'matchLabels': {'app': name}},
            'template': {
                'metadata': {'labels': {'app': name}},
                'spec': {
                    'containers': [{
                        'name': 'database',
                        'image': image,
                        'ports': [{'containerPort': get_port(engine)}],
                        'volumeMounts': [{
                            'name': 'data',
                            'mountPath': '/var/lib/database'
                        }]
                    }]
                }
            },
            'volumeClaimTemplates': [{
                'metadata': {'name': 'data'},
                'spec': {
                    'accessModes': ['ReadWriteOnce'],
                    'resources': {
                        'requests': {'storage': storage_size}
                    }
                }
            }]
        }
    }


def get_port(engine):
    """ส่งคืน port ของ database engine"""
    ports = {'mysql': 3306, 'postgresql': 5432, 'mongodb': 27017}
    return ports.get(engine, 3306)


def perform_backup(name, namespace, logger):
    """ทำ backup ก่อนลบ"""
    logger.info(f"Performing backup for {name} before deletion")
    # TODO: Implement actual backup logic
    pass


if __name__ == '__main__':
    kopf.run()
```

```bash
# Run Python operator
python operator.py

# หรือ deploy เป็น deployment
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-operator
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database-operator
  template:
    metadata:
      labels:
        app: database-operator
    spec:
      serviceAccountName: database-operator
      containers:
        - name: operator
          image: myregistry/python-database-operator:latest
          command: ["python", "operator.py"]
EOF
```

---

## 82.5 Workshop: สร้าง Simple Blog Operator

### เป้าหมาย
สร้าง Operator สำหรับจัดการ Blog application แบบ end-to-end

### ขั้นตอนที่ 1: สร้าง Project

```bash
mkdir blog-operator && cd blog-operator
operator-sdk init --domain=workshop.io --repo=github.com/workshop/blog-operator
operator-sdk create api --group=apps --version=v1 --kind=Blog --resource --controller
```

### ขั้นตอนที่ 2: กำหนด Types

```go
// api/v1/blog_types.go
package v1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

type BlogSpec struct {
    // +kubebuilder:validation:MinLength=1
    Title string `json:"title"`
    
    // +kubebuilder:default="wordpress:latest"
    Image string `json:"image,omitempty"`
    
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas,omitempty"`
    
    // +optional
    Domain string `json:"domain,omitempty"`
    
    Database DatabaseConfig `json:"database"`
}

type DatabaseConfig struct {
    // +kubebuilder:default="mysql"
    Type string `json:"type,omitempty"`
    
    // +kubebuilder:default="10Gi"
    StorageSize string `json:"storageSize,omitempty"`
}

type BlogStatus struct {
    // +optional
    Phase string `json:"phase,omitempty"`
    // +optional
    URL string `json:"url,omitempty"`
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Title",type=string,JSONPath=`.spec.title`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="URL",type=string,JSONPath=`.status.url`

type Blog struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              BlogSpec   `json:"spec,omitempty"`
    Status            BlogStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type BlogList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Blog `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Blog{}, &BlogList{})
}
```

### ขั้นตอนที่ 3: Implement Controller

```go
// controllers/blog_controller.go
package controllers

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    networkingv1 "k8s.io/api/networking/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/apimachinery/pkg/util/intstr"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    appsworkshopv1 "github.com/workshop/blog-operator/api/v1"
)

type BlogReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=apps.workshop.io,resources=blogs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.workshop.io,resources=blogs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=networking.k8s.io,resources=ingresses,verbs=get;list;watch;create;update;patch;delete

func (r *BlogReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    blog := &appsworkshopv1.Blog{}
    if err := r.Get(ctx, req.NamespacedName, blog); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Reconcile Deployment
    if err := r.reconcileDeployment(ctx, blog); err != nil {
        logger.Error(err, "Failed to reconcile Deployment")
        return ctrl.Result{}, err
    }

    // Reconcile Service
    if err := r.reconcileService(ctx, blog); err != nil {
        logger.Error(err, "Failed to reconcile Service")
        return ctrl.Result{}, err
    }

    // Reconcile Ingress (ถ้ามี domain)
    if blog.Spec.Domain != "" {
        if err := r.reconcileIngress(ctx, blog); err != nil {
            logger.Error(err, "Failed to reconcile Ingress")
            return ctrl.Result{}, err
        }
    }

    // อัปเดต status
    blog.Status.Phase = "Running"
    if blog.Spec.Domain != "" {
        blog.Status.URL = fmt.Sprintf("https://%s", blog.Spec.Domain)
    }
    if err := r.Status().Update(ctx, blog); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

func (r *BlogReconciler) reconcileDeployment(ctx context.Context, blog *appsworkshopv1.Blog) error {
    labels := map[string]string{"app": blog.Name, "component": "blog"}
    
    deploy := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      blog.Name,
            Namespace: blog.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, deploy, func() error {
        deploy.Labels = labels
        deploy.Spec = appsv1.DeploymentSpec{
            Replicas: &blog.Spec.Replicas,
            Selector: &metav1.LabelSelector{MatchLabels: labels},
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{Labels: labels},
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "blog",
                            Image: blog.Spec.Image,
                            Ports: []corev1.ContainerPort{{ContainerPort: 80}},
                            Env: []corev1.EnvVar{
                                {Name: "BLOG_TITLE", Value: blog.Spec.Title},
                            },
                        },
                    },
                },
            },
        }
        return controllerutil.SetControllerReference(blog, deploy, r.Scheme)
    })
    return err
}

func (r *BlogReconciler) reconcileService(ctx context.Context, blog *appsworkshopv1.Blog) error {
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      blog.Name,
            Namespace: blog.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
        svc.Spec = corev1.ServiceSpec{
            Selector: map[string]string{"app": blog.Name},
            Ports: []corev1.ServicePort{
                {Port: 80, TargetPort: intstr.FromInt(80)},
            },
        }
        return controllerutil.SetControllerReference(blog, svc, r.Scheme)
    })
    return err
}

func (r *BlogReconciler) reconcileIngress(ctx context.Context, blog *appsworkshopv1.Blog) error {
    pathType := networkingv1.PathTypePrefix
    ingress := &networkingv1.Ingress{
        ObjectMeta: metav1.ObjectMeta{
            Name:      blog.Name,
            Namespace: blog.Namespace,
        },
    }
    
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, ingress, func() error {
        ingress.Spec = networkingv1.IngressSpec{
            Rules: []networkingv1.IngressRule{
                {
                    Host: blog.Spec.Domain,
                    IngressRuleValue: networkingv1.IngressRuleValue{
                        HTTP: &networkingv1.HTTPIngressRuleValue{
                            Paths: []networkingv1.HTTPIngressPath{
                                {
                                    Path:     "/",
                                    PathType: &pathType,
                                    Backend: networkingv1.IngressBackend{
                                        Service: &networkingv1.IngressServiceBackend{
                                            Name: blog.Name,
                                            Port: networkingv1.ServiceBackendPort{Number: 80},
                                        },
                                    },
                                },
                            },
                        },
                    },
                },
            },
        }
        return controllerutil.SetControllerReference(blog, ingress, r.Scheme)
    })
    return err
}

func (r *BlogReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appsworkshopv1.Blog{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Owns(&networkingv1.Ingress{}).
        Complete(r)
}
```

### ขั้นตอนที่ 4: Build และ Test

```bash
# Generate
make generate
make manifests

# Install CRDs
make install

# Run locally
make run &

# สร้าง Blog
kubectl apply -f - <<EOF
apiVersion: apps.workshop.io/v1
kind: Blog
metadata:
  name: my-blog
  namespace: default
spec:
  title: "My Awesome Blog"
  image: nginx:1.25
  replicas: 2
  domain: blog.example.com
  database:
    type: mysql
    storageSize: 20Gi
EOF

# ดู Blog
kubectl get blogs
kubectl describe blog my-blog

# ดู resources ที่ถูกสร้าง
kubectl get deploy,svc,ingress | grep my-blog

# Scale
kubectl patch blog my-blog --type=merge -p '{"spec":{"replicas":3}}'

# ลบ
kubectl delete blog my-blog
# ดูว่า Deployment, Service, Ingress ถูกลบตาม
```

---

## 82.6 Operator Testing

### Unit Testing ด้วย EnvTest

```go
// controllers/blog_controller_test.go
package controllers

import (
    "context"
    "time"
    
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    
    appsworkshopv1 "github.com/workshop/blog-operator/api/v1"
)

var _ = Describe("Blog Controller", func() {
    Context("When creating a Blog", func() {
        It("Should create a Deployment", func() {
            ctx := context.Background()
            
            blog := &appsworkshopv1.Blog{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-blog",
                    Namespace: "default",
                },
                Spec: appsworkshopv1.BlogSpec{
                    Title:    "Test Blog",
                    Image:    "nginx:latest",
                    Replicas: 1,
                    Database: appsworkshopv1.DatabaseConfig{
                        Type:        "mysql",
                        StorageSize: "1Gi",
                    },
                },
            }
            
            Expect(k8sClient.Create(ctx, blog)).Should(Succeed())
            
            // รอให้ Deployment ถูกสร้าง
            deploymentLookupKey := types.NamespacedName{Name: "test-blog", Namespace: "default"}
            createdDeployment := &appsv1.Deployment{}
            
            Eventually(func() bool {
                err := k8sClient.Get(ctx, deploymentLookupKey, createdDeployment)
                return err == nil
            }, time.Second*30, time.Millisecond*250).Should(BeTrue())
            
            Expect(*createdDeployment.Spec.Replicas).Should(Equal(int32(1)))
            
            // Cleanup
            Expect(k8sClient.Delete(ctx, blog)).Should(Succeed())
        })
    })
})
```

```bash
# Run tests
make test

# หรือ
go test ./... -v
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Operator Pattern**: Control Loop, Maturity Model
2. **Operator Framework**: operator-sdk, kubebuilder
3. **Go Operator**: Types, Controller, Reconciliation Logic
4. **Python Operator**: kopf framework
5. **Workshop**: Blog Operator แบบ End-to-End

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Admission Controllers ซึ่งช่วยให้ validate และ mutate resources ก่อนที่จะถูก persist
