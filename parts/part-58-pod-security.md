# Part 58: Pod Security Standards

## บทนำ

Pod Security Standards (PSS) เป็นมาตรฐานความปลอดภัยสำหรับ Pods ที่ถูกนำมาแทนที่ Pod Security Policies (PSP) ที่ deprecated ตั้งแต่ Kubernetes 1.21 และถูกลบใน 1.25

PSS กำหนด 3 ระดับความปลอดภัยและ enforce ผ่าน Pod Security Admission (PSA)

---

## 58.1 Pod Security Standards: 3 ระดับ

### Privileged (ไม่มีข้อจำกัด)

```
ระดับ: Privileged
ข้อจำกัด: ไม่มีเลย
ใช้เมื่อ: Trusted system workloads (CNI, CSI, monitoring agents)

อนุญาต:
✓ Privileged containers
✓ Host namespaces
✓ hostPath volumes
✓ Root user
✓ ทุก capabilities
```

### Baseline (ข้อจำกัดขั้นต่ำ)

```
ระดับ: Baseline
ข้อจำกัด: ป้องกัน privilege escalation ที่รู้จักกัน
ใช้เมื่อ: General purpose workloads ส่วนใหญ่

ห้าม:
✗ Privileged containers
✗ Host namespaces (hostPID, hostIPC, hostNetwork)
✗ hostPath volumes
✗ Dangerous capabilities (NET_ADMIN, SYS_ADMIN, etc.)

อนุญาต:
✓ Non-root capabilities
✓ Root user (แต่ไม่แนะนำ)
✓ emptyDir, configMap, secret volumes
```

### Restricted (ข้อจำกัดสูง)

```
ระดับ: Restricted
ข้อจำกัด: ตาม best practices ปัจจุบัน
ใช้เมื่อ: Security-sensitive workloads

ต้องทำ:
✓ runAsNonRoot: true
✓ allowPrivilegeEscalation: false
✓ seccompProfile: RuntimeDefault หรือ Localhost
✓ capabilities.drop: ["ALL"]
✓ ห้ามใช้ capabilities ยกเว้น NET_BIND_SERVICE

ห้าม:
✗ ทุกอย่างที่ Baseline ห้าม + เพิ่มเติม
✗ Root user
✗ HostProcess containers
✗ HostPath volumes
✗ HostPort
✗ ทุก volume types ยกเว้นที่ปลอดภัย
```

---

## 58.2 Pod Security Admission (PSA)

PSA enforce PSS ผ่าน admission controller ที่ทำงานใน kube-apiserver

### Modes ของ PSA

```
enforce: Reject pods ที่ไม่ผ่าน (hard block)
audit:   Log pods ที่ไม่ผ่านใน audit log
warn:    แสดง warning แก่ user (ไม่ reject)

สามารถตั้งค่าแต่ละ mode ให้ต่าง level ได้:
- enforce=restricted
- audit=restricted  
- warn=restricted
```

### Namespace Labels สำหรับ PSA

```bash
# Enable PSA สำหรับ namespace ด้วย labels

# ระดับ Restricted
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# ระดับ Baseline
kubectl label namespace development \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

# ระดับ Privileged (ไม่มีข้อจำกัด)
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged
```

### ดู Labels

```bash
kubectl get namespace production -o yaml
# จะเห็น labels ที่เพิ่มไป

kubectl describe namespace production
# ดู labels
```

---

## 58.3 Privileged Level

```yaml
# Namespace ที่ใช้ privileged level
apiVersion: v1
kind: Namespace
metadata:
  name: privileged-ns
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
# Pod ที่ใช้ privileged features
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: privileged-ns
spec:
  hostPID: true          # เข้าถึง host PID namespace
  hostIPC: true          # เข้าถึง host IPC
  hostNetwork: true      # ใช้ host network
  
  containers:
  - name: privileged
    image: alpine:3.18
    securityContext:
      privileged: true   # สิทธิ์เต็มที่
    
    volumeMounts:
    - name: host-root
      mountPath: /host
  
  volumes:
  - name: host-root
    hostPath:
      path: /           # mount host filesystem
```

---

## 58.4 Baseline Level

```yaml
# Namespace ที่ใช้ baseline level
apiVersion: v1
kind: Namespace
metadata:
  name: baseline-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted   # warn ถ้าไม่ผ่าน restricted
---
# Pod ที่ผ่าน baseline (แต่ไม่ผ่าน restricted)
apiVersion: v1
kind: Pod
metadata:
  name: baseline-ok
  namespace: baseline-ns
spec:
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      runAsUser: 0    # root user - ผ่าน baseline แต่ fail restricted
      # ไม่มี privileged: ผ่าน baseline ✓
---
# Pod ที่ไม่ผ่าน baseline (reject)
apiVersion: v1
kind: Pod
metadata:
  name: baseline-fail
  namespace: baseline-ns
spec:
  containers:
  - name: app
    image: alpine:3.18
    securityContext:
      privileged: true    # fail baseline! จะถูก reject
```

---

## 58.5 Restricted Level

```yaml
# Namespace restricted
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
---
# Pod ที่ผ่าน restricted level
apiVersion: v1
kind: Pod
metadata:
  name: restricted-ok
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true        # required
    runAsUser: 1000
    seccompProfile:           # required
      type: RuntimeDefault
  
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      allowPrivilegeEscalation: false   # required
      capabilities:
        drop: ["ALL"]                   # required
      readOnlyRootFilesystem: true
      runAsNonRoot: true
    
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  
  volumes:
  - name: tmp
    emptyDir: {}
---
# Pod ที่ไม่ผ่าน restricted (ไม่มี seccomp)
apiVersion: v1
kind: Pod
metadata:
  name: restricted-fail
  namespace: restricted-ns
spec:
  containers:
  - name: app
    image: nginx:1.25-alpine
    # ไม่มี seccompProfile - fail restricted!
    # Warning จะปรากฏ
```

---

## 58.6 Workshop: Implement Pod Security Policies

### สถานการณ์

เราจะสร้าง 3 namespace พร้อม PSA levels ต่างกัน และทดสอบว่า pods ผ่านหรือไม่

### Step 1: สร้าง Namespaces พร้อม PSA

```bash
# สร้าง namespaces
kubectl create namespace pss-privileged
kubectl create namespace pss-baseline
kubectl create namespace pss-restricted

# Label namespaces
kubectl label namespace pss-privileged \
  pod-security.kubernetes.io/enforce=privileged

kubectl label namespace pss-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

kubectl label namespace pss-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# ตรวจสอบ
kubectl get namespace pss-privileged pss-baseline pss-restricted \
  --show-labels
```

### Step 2: ทดสอบ Privileged Namespace

```bash
# Test 1: Privileged pod ใน privileged namespace - ผ่าน
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: pss-privileged
spec:
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
EOF
kubectl get pod test-privileged -n pss-privileged
echo "Status: $(kubectl get pod test-privileged -n pss-privileged -o jsonpath='{.status.phase}')"
kubectl delete pod test-privileged -n pss-privileged
```

### Step 3: ทดสอบ Baseline Namespace

```bash
# Test 2: Regular pod ใน baseline namespace - ผ่าน (แต่มี warning)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-baseline-ok
  namespace: pss-baseline
spec:
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000   # non-root OK สำหรับ baseline
EOF
# จะเห็น warning เพราะไม่ผ่าน restricted (warn mode)
kubectl get pod test-baseline-ok -n pss-baseline
kubectl delete pod test-baseline-ok -n pss-baseline

# Test 3: Privileged pod ใน baseline namespace - FAIL
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-baseline-fail
  namespace: pss-baseline
spec:
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
    securityContext:
      privileged: true    # ห้ามใน baseline!
EOF
# ควรเห็น error: pods "test-baseline-fail" is forbidden
echo "Expected: Pod rejected!"
```

### Step 4: ทดสอบ Restricted Namespace

```bash
# Test 4: Pod ที่ตรงตาม restricted - ผ่าน
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-restricted-ok
  namespace: pss-restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
EOF
kubectl get pod test-restricted-ok -n pss-restricted
kubectl delete pod test-restricted-ok -n pss-restricted

# Test 5: Pod ที่ขาด seccompProfile - FAIL
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-restricted-fail
  namespace: pss-restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    # ขาด seccompProfile!
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
EOF
echo "Expected: Pod rejected (missing seccompProfile)!"

# Test 6: Pod ที่ขาด capabilities.drop - FAIL
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-restricted-fail2
  namespace: pss-restricted
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      # ขาด capabilities.drop!
EOF
echo "Expected: Pod rejected (missing capabilities.drop)!"
```

### Step 5: Deploy Real Application ใน Restricted Namespace

```yaml
# production-app.yaml
# Deployment ที่ compliant กับ restricted PSS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-webapp
  namespace: pss-restricted
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-webapp
  template:
    metadata:
      labels:
        app: secure-webapp
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      # Disable service account token mount
      automountServiceAccountToken: false
      
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.25-alpine
        # nginxinc/nginx-unprivileged ออกแบบมาให้รัน non-root
        ports:
        - containerPort: 8080
        
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 101  # nginx user ใน unprivileged image
        
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
        - name: tmp
          mountPath: /tmp
        - name: config
          mountPath: /etc/nginx/conf.d/
          readOnly: true
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
        
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
      - name: tmp
        emptyDir: {}
      - name: config
        configMap:
          name: webapp-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: pss-restricted
data:
  default.conf: |
    server {
        listen 8080;
        location / {
            return 200 'Secure App - PSS Restricted';
            add_header Content-Type text/plain;
        }
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: secure-webapp
  namespace: pss-restricted
spec:
  selector:
    app: secure-webapp
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f production-app.yaml
kubectl get pods -n pss-restricted -w
```

### Step 6: ตรวจสอบผลลัพธ์

```bash
# ตรวจสอบ pods
kubectl get pods -n pss-restricted

POD=$(kubectl get pod -l app=secure-webapp -n pss-restricted -o jsonpath='{.items[0].metadata.name}')

# ตรวจสอบ security context
kubectl describe pod $POD -n pss-restricted | grep -A 20 "Security Context"

# ทดสอบ
kubectl port-forward svc/secure-webapp 8080:80 -n pss-restricted &
curl http://localhost:8080
kill %1

# ดู audit events สำหรับ PSS violations (ถ้า audit enabled)
kubectl get events -n pss-restricted --field-selector=reason=FailedCreate
```

### Step 7: Dry-run PSA ก่อน Apply

```bash
# ตรวจสอบว่า namespace ใดที่ PSA จะ affect
kubectl label namespace pss-test-dry-run \
  pod-security.kubernetes.io/enforce=restricted \
  --dry-run=server 2>&1 || true

# ทดสอบว่า pod ผ่าน PSA หรือเปล่า
kubectl apply --dry-run=server -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: dry-run-test
  namespace: pss-restricted
spec:
  containers:
  - name: app
    image: alpine:3.18
    command: ["sleep", "3600"]
EOF
# ถ้า fail จะเห็น error ทันที
```

### Step 8: Migration จาก PSP ไป PSS

```bash
# หา PSP ที่มีอยู่ (ถ้ายังใช้ Kubernetes < 1.25)
# kubectl get psp

# แปลง PSP เป็น PSS equivalent
# Privileged PSP → privileged level
# Restricted PSP → restricted level

# Script ตรวจสอบ pods ที่จะ affected
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | 
    select(.spec.securityContext.runAsNonRoot != true) | 
    .metadata.namespace + "/" + .metadata.name + ": missing runAsNonRoot"'
```

### Step 9: Cleanup

```bash
kubectl delete namespace pss-privileged pss-baseline pss-restricted
echo "Workshop cleanup complete!"
```

---

## 58.7 PSA Configuration ใน kube-apiserver

### Cluster-wide Default

```yaml
# /etc/kubernetes/admission-configuration.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      # Default สำหรับทุก namespace ที่ไม่มี label
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      # Usernames ที่ exempt
      usernames: []
      # RuntimeClasses ที่ exempt
      runtimeClasses: []
      # Namespaces ที่ exempt
      namespaces:
      - kube-system
      - kube-public
```

```bash
# เพิ่ม flag ใน kube-apiserver
# --admission-control-config-file=/etc/kubernetes/admission-configuration.yaml
```

---

## 58.8 Best Practices

### 1. เริ่มจาก warn mode

```bash
# ก่อน enforce ให้ใช้ warn/audit ก่อน เพื่อดูว่า pods อะไรที่จะ fail
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# รอดู warnings แล้วค่อย enforce
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

### 2. ใช้ kube-bench สำหรับ audit

```bash
# ตรวจสอบ cluster security posture
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```

### 3. ตรวจสอบ pods ที่ไม่ผ่าน restricted

```bash
# Script หา pods ที่จะ fail restricted
kubectl get pods --all-namespaces -o json | python3 -c "
import json, sys

data = json.load(sys.stdin)
issues = []

for pod in data['items']:
    ns = pod['metadata']['namespace']
    name = pod['metadata']['name']
    spec = pod['spec']
    
    pod_sec = spec.get('securityContext', {})
    
    if not pod_sec.get('runAsNonRoot', False):
        issues.append(f'{ns}/{name}: missing pod.securityContext.runAsNonRoot')
    
    if not pod_sec.get('seccompProfile'):
        issues.append(f'{ns}/{name}: missing pod.securityContext.seccompProfile')
    
    for c in spec.get('containers', []):
        c_sec = c.get('securityContext', {})
        if c_sec.get('allowPrivilegeEscalation', True):
            issues.append(f'{ns}/{name}/{c[\"name\"]}: allowPrivilegeEscalation not false')
        
        caps = c_sec.get('capabilities', {})
        if 'ALL' not in caps.get('drop', []):
            issues.append(f'{ns}/{name}/{c[\"name\"]}: missing capabilities.drop ALL')

for issue in issues[:20]:
    print(issue)
if len(issues) > 20:
    print(f'... and {len(issues)-20} more issues')
"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Pod Security Standards**: Privileged, Baseline, Restricted
2. **Pod Security Admission**: enforce, audit, warn modes
3. **Namespace Labels**: กำหนด PSS ต่อ namespace
4. **Migration**: จาก PSP ไป PSS

**Key Takeaways:**
- Production namespaces ควรใช้ restricted level
- เริ่มจาก warn/audit mode ก่อน enforce
- ใช้ `nginxinc/nginx-unprivileged` แทน `nginx` official สำหรับ restricted
- Exempt kube-system จาก restricted enforcement

---

## 58.9 Pod Security Standards กับ Helm Charts

### ตรวจสอบ Helm Charts ก่อน Deploy

```bash
# ใช้ helm template แล้ว scan ด้วย kube-score
helm template my-release bitnami/nginx | kube-score score -

# หรือใช้ trivy config
helm template my-release bitnami/nginx > rendered.yaml
trivy config rendered.yaml --severity HIGH,CRITICAL

# ใช้ checkov
checkov -f rendered.yaml --check CKV_K8S_*
```

### แก้ไข Helm Chart Values สำหรับ PSS

```yaml
# values-restricted.yaml สำหรับ deploy ใน restricted namespace

# ตัวอย่าง bitnami/postgresql
primary:
  podSecurityContext:
    enabled: true
    fsGroup: 1001
    runAsUser: 1001
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  
  containerSecurityContext:
    enabled: true
    runAsUser: 1001
    runAsNonRoot: true
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: false  # postgres ต้องการ writable
    capabilities:
      drop: ["ALL"]

# ตัวอย่าง bitnami/nginx
containerSecurityContext:
  enabled: true
  runAsUser: 1001
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

podSecurityContext:
  enabled: true
  fsGroup: 1001
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

---

## 58.10 Integration กับ OPA Gatekeeper

### Constraint Template สำหรับ PSS Compliance

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spodbaseline
spec:
  crd:
    spec:
      names:
        kind: K8sPodBaseline
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8spodbaseline
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged == true
        msg := sprintf("Container '%v' is privileged. This violates baseline PSS.", [container.name])
      }
      
      violation[{"msg": msg}] {
        input.review.object.spec.hostPID == true
        msg := "hostPID is not allowed in baseline PSS"
      }
      
      violation[{"msg": msg}] {
        input.review.object.spec.hostIPC == true
        msg := "hostIPC is not allowed in baseline PSS"
      }
      
      violation[{"msg": msg}] {
        input.review.object.spec.hostNetwork == true
        msg := "hostNetwork is not allowed in baseline PSS"
      }
      
      violation[{"msg": msg}] {
        volume := input.review.object.spec.volumes[_]
        volume.hostPath
        msg := sprintf("hostPath volume '%v' is not allowed in baseline PSS", [volume.name])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPodBaseline
metadata:
  name: enforce-baseline-pss
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - kube-public
```

---

## 58.11 Kyverno Policies สำหรับ PSS

```yaml
# kyverno-pss-restricted.yaml

# Policy 1: Require non-root
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-nonroot
  annotations:
    policies.kyverno.io/title: Require Run As Non-Root User
    policies.kyverno.io/category: Pod Security - Restricted
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: check-containers
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - production
          - staging
    validate:
      message: >-
        Containers must not run as root user.
        Set runAsNonRoot: true and runAsUser > 0.
      pattern:
        spec:
          =(securityContext):
            =(runAsNonRoot): "true"
          containers:
          - =(securityContext):
              =(runAsNonRoot): "true"
---
# Policy 2: Drop all capabilities
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: drop-all-capabilities
  annotations:
    policies.kyverno.io/title: Drop All Capabilities
    policies.kyverno.io/category: Pod Security - Restricted
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: require-drop-all
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: >-
        Containers must drop ALL capabilities.
        Add 'capabilities.drop: ["ALL"]' to securityContext.
      foreach:
      - list: request.object.spec.containers
        deny:
          conditions:
            any:
            - key: "{{ element.securityContext.capabilities.drop }}"
              operator: NotIn
              value: ["ALL"]
---
# Policy 3: Require seccomp profile
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-seccomp-profile
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: check-seccomp
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - production
    validate:
      message: >-
        A seccomp profile is required. 
        Set seccompProfile.type to RuntimeDefault or Localhost.
      anyPattern:
      - spec:
          securityContext:
            seccompProfile:
              type: "RuntimeDefault | Localhost"
      - spec:
          containers:
          - securityContext:
              seccompProfile:
                type: "RuntimeDefault | Localhost"
```

---

## 58.12 Audit: ตรวจสอบ PSS Compliance ทั้ง Cluster

```bash
#!/bin/bash
# pss-audit.sh - ตรวจสอบ PSS compliance

echo "=== Pod Security Standards Audit ==="
echo ""

# Function ตรวจสอบ pod
check_pod_pss() {
  local NS=$1
  local POD=$2
  local ISSUES=()
  
  # ดึงข้อมูล pod
  POD_JSON=$(kubectl get pod $POD -n $NS -o json 2>/dev/null)
  
  # ตรวจสอบ runAsNonRoot
  RUN_AS_NONROOT=$(echo $POD_JSON | jq -r '.spec.securityContext.runAsNonRoot // false')
  if [ "$RUN_AS_NONROOT" != "true" ]; then
    ISSUES+=("Missing: pod.securityContext.runAsNonRoot")
  fi
  
  # ตรวจสอบ seccompProfile
  SECCOMP=$(echo $POD_JSON | jq -r '.spec.securityContext.seccompProfile.type // "none"')
  if [ "$SECCOMP" = "none" ]; then
    ISSUES+=("Missing: pod.securityContext.seccompProfile")
  fi
  
  # ตรวจสอบ privileged
  PRIVILEGED=$(echo $POD_JSON | jq -r '[.spec.containers[].securityContext.privileged] | any')
  if [ "$PRIVILEGED" = "true" ]; then
    ISSUES+=("VIOLATION: privileged container found")
  fi
  
  # แสดงผล
  if [ ${#ISSUES[@]} -eq 0 ]; then
    echo "  ✓ $NS/$POD - COMPLIANT"
  else
    echo "  ✗ $NS/$POD - NON-COMPLIANT:"
    for ISSUE in "${ISSUES[@]}"; do
      echo "    - $ISSUE"
    done
  fi
}

# ตรวจสอบทุก namespace
for NS in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  # ข้าม system namespaces
  if [[ "$NS" =~ ^(kube-system|kube-public|kube-node-lease)$ ]]; then
    continue
  fi
  
  PODS=$(kubectl get pods -n $NS -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)
  
  if [ -n "$PODS" ]; then
    echo "Namespace: $NS"
    for POD in $PODS; do
      check_pod_pss $NS $POD
    done
    echo ""
  fi
done

echo "Audit complete!"
```

```bash
chmod +x pss-audit.sh
./pss-audit.sh
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Pod Security Standards**: Privileged, Baseline, Restricted - 3 ระดับความปลอดภัย
2. **Pod Security Admission**: enforce, audit, warn modes ผ่าน namespace labels
3. **Migration**: จาก PSP ไป PSS อย่างปลอดภัย
4. **Helm Charts**: ปรับ values ให้ compliant กับ PSS
5. **Policy Tools**: Gatekeeper, Kyverno สำหรับ custom policies
6. **Audit**: ตรวจสอบ compliance ทั้ง cluster

**Key Takeaways:**
- Production namespaces ควรใช้ restricted level
- เริ่มจาก warn/audit mode ก่อน enforce
- ใช้ `nginxinc/nginx-unprivileged` แทน `nginx` official สำหรับ restricted
- Exempt kube-system จาก restricted enforcement
- ทำ regular PSS compliance audit

---

**ต่อไป**: Part 59 - Advanced Network Policies
