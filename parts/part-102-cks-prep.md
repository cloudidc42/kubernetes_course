# Part 102: CKS (Certified Kubernetes Security Specialist) Exam Guide

## บทนำ

Certified Kubernetes Security Specialist (CKS) คือ Certification ระดับสูงสุดของ CNCF ที่เน้น Security บน Kubernetes บทนี้ครอบคลุม Exam Domains ทั้งหมดพร้อม Practice Questions กว่า 200 ข้อ

## ข้อมูล Exam

```
Exam Format: Performance-based (Hands-on)
Duration: 2 hours
Pass Score: 67%
Cost: $395 USD
Validity: 2 years
Prerequisite: CKA (required)
Number of Questions: 15-20 tasks

Exam Domains (v1.28):
- Cluster Setup: 10%
- Cluster Hardening: 15%
- System Hardening: 15%
- Minimize Microservice Vulnerabilities: 20%
- Supply Chain Security: 20%
- Monitoring, Logging and Runtime Security: 20%
```

## สารบัญ

1. Cluster Setup (10%)
2. Cluster Hardening (15%)
3. System Hardening (15%)
4. Minimize Microservice Vulnerabilities (20%)
5. Supply Chain Security (20%)
6. Monitoring, Logging and Runtime Security (20%)
7. Exam Tips
8. Practice Labs (200+ Questions)

---

## 1. Cluster Setup (10%)

### 1.1 Network Policies

```yaml
# Default Deny All
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow DNS (ต้องมีเสมอ)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### 1.2 CIS Benchmarks

```bash
# ติดตั้ง kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# ดูผลลัพธ์
kubectl logs -l app=kube-bench -n kube-bench

# รัน kube-bench เฉพาะ Master
docker run --pid=host \
  -v /etc:/etc:ro \
  -v /var:/var:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  aquasec/kube-bench:latest master

# รัน kube-bench เฉพาะ Worker
docker run --pid=host \
  -v /etc:/etc:ro \
  -v /var:/var:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  aquasec/kube-bench:latest node
```

### 1.3 TLS Ingress

```yaml
# สร้าง Self-signed Certificate
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.example.com/O=myapp"

# สร้าง TLS Secret
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key

# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

---

## 2. Cluster Hardening (15%)

### 2.1 RBAC Hardening

```bash
# ค้นหา Overprivileged Bindings
# ClusterAdmin ทั้งหมด
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | "\(.metadata.name): \(.subjects[].name)"'

# Wildcard Verbs/Resources
kubectl get clusterroles -o json | \
  jq -r '.items[] | select(.rules[]?.verbs[]? == "*") | .metadata.name'

# ลบ Default ServiceAccount Token Auto-mount
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
automountServiceAccountToken: false
EOF
```

### 2.2 API Server Hardening

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # Disable Anonymous Auth
    - --anonymous-auth=false
    # Enable RBAC
    - --authorization-mode=Node,RBAC
    # Admission Controllers
    - --enable-admission-plugins=NodeRestriction,PodSecurity
    # Disable Insecure Port
    - --insecure-port=0
    # Audit Logging
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    # TLS
    - --tls-min-version=VersionTLS12
    - --tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    # Encryption at Rest
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### 2.3 Secrets Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

```bash
# สร้าง Encryption Key
head -c 32 /dev/urandom | base64

# หลังจากตั้งค่า, Re-encrypt Secrets ที่มีอยู่
kubectl get secrets -A -o json | \
  kubectl replace -f -
```

---

## 3. System Hardening (15%)

### 3.1 AppArmor

```bash
# ดู AppArmor Profiles ที่ Load
apparmor_status
aa-status

# สร้าง Profile
cat <<'EOF' > /etc/apparmor.d/k8s-nginx
#include <tunables/global>

profile k8s-nginx flags=(attach_disconnected) {
  #include <abstractions/base>
  
  network inet tcp,
  network inet udp,
  
  /usr/sbin/nginx mr,
  /var/log/nginx/** rw,
  /etc/nginx/** r,
  /usr/share/nginx/** r,
  /var/cache/nginx/** rw,
  /run/nginx.pid rw,
  
  deny @{HOME}/ w,
  deny /etc/shadow r,
}
EOF

# Load Profile
apparmor_parser -r /etc/apparmor.d/k8s-nginx

# ใช้ใน Pod
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/k8s-nginx
```

### 3.2 Seccomp

```bash
# ดู Seccomp Profiles
ls /var/lib/kubelet/seccomp/

# สร้าง Custom Seccomp Profile
mkdir -p /var/lib/kubelet/seccomp/profiles
cat <<'EOF' > /var/lib/kubelet/seccomp/profiles/deny-write.json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["write", "open", "creat"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
EOF
```

```yaml
# ใช้ Seccomp ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-demo
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/deny-write.json
  containers:
  - name: app
    image: nginx
    securityContext:
      seccompProfile:
        type: RuntimeDefault  # หรือ Localhost
```

### 3.3 Pod Security Standards

```yaml
# Restricted Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: restricted
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Baseline Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: baseline
  labels:
    pod-security.kubernetes.io/enforce: baseline
```

---

## 4. Minimize Microservice Vulnerabilities (20%)

### 4.1 OPA Gatekeeper

```yaml
# ConstraintTemplate: ห้าม Privileged
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoPrivileged
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivileged
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8snoprivileged
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged == true
        msg := sprintf("Privileged container not allowed: %v", [container.name])
      }
---
# Apply Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoPrivileged
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
```

### 4.2 Kyverno

```yaml
# Kyverno Policy: Require Non-Root
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-non-root
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Running as root is not allowed"
      pattern:
        spec:
          =(securityContext):
            =(runAsNonRoot): true
          containers:
          - =(securityContext):
              =(runAsUser): ">0"
---
# Auto-generate NetworkPolicy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-netpol
spec:
  rules:
  - name: gen-network-policy
    match:
      any:
      - resources:
          kinds: ["Namespace"]
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: default-deny
      namespace: "{{request.object.metadata.name}}"
      data:
        spec:
          podSelector: {}
          policyTypes:
          - Ingress
          - Egress
```

### 4.3 mTLS with Istio

```yaml
# PeerAuthentication: Require mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# AuthorizationPolicy: Allow only specific source
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend-sa"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

---

## 5. Supply Chain Security (20%)

### 5.1 Image Scanning with Trivy

```bash
# ติดตั้ง Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan Image
trivy image nginx:latest

# Scan เฉพาะ HIGH และ CRITICAL
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan และ Export JSON
trivy image --format json -o results.json nginx:latest

# Scan ใน CI/CD - fail ถ้ามี HIGH vulnerability
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest
```

### 5.2 Image Signing with Cosign

```bash
# ติดตั้ง Cosign
curl -sL https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-linux-amd64 -o /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign

# สร้าง Key Pair
cosign generate-key-pair

# Sign Image
cosign sign --key cosign.key registry.io/myapp:1.0

# Verify Image
cosign verify --key cosign.pub registry.io/myapp:1.0

# Keyless Signing (ด้วย Fulcio/Rekor)
COSIGN_EXPERIMENTAL=1 cosign sign registry.io/myapp:1.0
```

### 5.3 Policy Controller

```yaml
# ติดตั้ง Policy Controller
helm install policy-controller \
  sigstore/policy-controller \
  --namespace cosign-system \
  --create-namespace

# ClusterImagePolicy
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
  - glob: "registry.io/**"
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        <cosign-public-key>
        -----END PUBLIC KEY-----
```

### 5.4 Allowed Image Registries (OPA)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sAllowedRepos
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
        satisfied := [good | repo = input.parameters.repos[_];
          good = startswith(container.image, repo)]
        not any(satisfied)
        msg := sprintf("Container %v has invalid image %v", [container.name, container.image])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    repos:
    - "registry.company.com/"
    - "gcr.io/company/"
```

---

## 6. Monitoring, Logging and Runtime Security (20%)

### 6.1 Falco

```bash
# ติดตั้ง Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  -n falco \
  --create-namespace

# Custom Rules
cat > /etc/falco/rules.d/custom_rules.yaml << 'EOF'
- rule: Write to etc
  desc: Detect writing to /etc
  condition: >
    open_write and fd.directory = /etc
    and not proc.name in (package managers)
  output: >
    Write to /etc (user=%user.name proc=%proc.name
    file=%fd.name cmdline=%proc.cmdline)
  priority: WARNING

- rule: Shell in Container
  desc: Shell spawned in container
  condition: >
    spawned_process and container
    and shell_procs and not proc.pname in (known_parents)
  output: >
    Shell spawned in container
    (container=%container.name proc=%proc.name)
  priority: CRITICAL
EOF

# ดู Falco Alerts
kubectl logs -n falco -l app=falco -f
```

### 6.2 Audit Logging

```yaml
# Audit Policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log Secret access ทั้งหมด
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log Pod exec/attach
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach"]

# Log Auth failures
- level: Request
  verbs: ["get", "list", "watch"]
  users: ["system:anonymous"]

# ไม่ต้อง Log health checks
- level: None
  users: ["system:apiserver"]
  verbs: ["get"]
  resources:
  - group: ""
    resources: ["nodes/metrics", "nodes/stats"]

# Default
- level: Metadata
```

### 6.3 Container Runtime Security

```bash
# Detect กระบวนการ Suspicious ใน Container
kubectl exec -n default suspicious-pod -- ps aux

# ดู Network Connections ใน Container
kubectl exec -n default suspicious-pod -- netstat -tunapl

# ดู File Changes
kubectl exec -n default suspicious-pod -- find / -newer /tmp -type f 2>/dev/null

# ตรวจสอบ Capabilities
kubectl exec -n default suspicious-pod -- cat /proc/1/status | grep Cap
```

---

## 7. Exam Tips

```
CKS Exam Strategy:
1. ต้องมี CKA ก่อน
2. อ่าน Falco Rules ให้เข้าใจ
3. ฝึก Seccomp, AppArmor, OPA
4. เข้าใจ mTLS concept
5. ฝึก Image scanning กับ Trivy
6. จำ Pod Security Standards levels

Frequently Tested:
- Network Policies (default deny)
- RBAC - ลด Privilege
- SecurityContext
- Image scanning
- Audit logs
- Falco rules
- etcd encryption
- Admission Controllers
```

---

## 8. Practice Labs (200+ Questions)

### Section A: Network Security

**Q1: สร้าง NetworkPolicy Default Deny ใน namespace web**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: web
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Q2: อนุญาต traffic จาก monitoring namespace ไปยัง app pod**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    ports:
    - port: 9090
```

**Q3: อนุญาต DNS egress เท่านั้น**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-only
spec:
  podSelector:
    matchLabels:
      tier: restricted
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

**Q4: อนุญาต frontend → backend เท่านั้น**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      role: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

**Q5: ตรวจสอบ NetworkPolicy ที่ใช้งาน**
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy -n web
```

### Section B: RBAC Security

**Q6: สร้าง Role ที่อ่านได้เฉพาะ ConfigMaps**
```bash
kubectl create role configmap-reader \
  --verb=get,list,watch \
  --resource=configmaps \
  -n dev
```

**Q7: ตรวจสอบว่า ServiceAccount มีสิทธิ์อะไรบ้าง**
```bash
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa
```

**Q8: ลด Privilege ของ ClusterRole**
```bash
# ดูสิทธิ์ปัจจุบัน
kubectl get clusterrole system:controller:service-account-controller -o yaml

# สร้าง ClusterRole ที่จำกัดกว่า
kubectl create clusterrole limited-role \
  --verb=get,list \
  --resource=pods,services
```

**Q9: สร้าง ServiceAccount ที่ไม่ automount token**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-token-sa
automountServiceAccountToken: false
```

**Q10: Bind Role ให้ ServiceAccount**
```bash
kubectl create rolebinding limited-binding \
  --clusterrole=view \
  --serviceaccount=default:no-token-sa \
  -n default
```

### Section C: Cluster Hardening

**Q11: ตรวจสอบ API Server flags**
```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E "(anonymous|authorization|admission|audit)"

# หรือ
ps aux | grep kube-apiserver | tr ' ' '\n' | grep -E "^--"
```

**Q12: เปิด Encryption at Rest สำหรับ Secrets**
```bash
# สร้าง Encryption Key
KEY=$(head -c 32 /dev/urandom | base64)

# สร้าง Config
cat > /etc/kubernetes/encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: ${KEY}
  - identity: {}
EOF

# เพิ่มใน kube-apiserver.yaml
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# Re-encrypt existing secrets
kubectl get secrets -A -o json | kubectl replace -f -
```

**Q13: ตรวจสอบว่า Secret ถูก Encrypt**
```bash
# ต้องรัน etcdctl เพื่อดู raw value
ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  /registry/secrets/default/my-secret | hexdump -C | head
# ถ้า Encrypt จะเห็น k8s:enc:aescbc...
```

**Q14: ตั้งค่า Audit Policy**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: Request
  verbs: ["delete"]
  omitStages: ["RequestReceived"]
- level: None
  users: ["system:kube-proxy"]
```

**Q15: ค้นหา Privilege Escalation ใน Audit Logs**
```bash
grep '"verb":"create","resource":"pods/exec"' /var/log/kubernetes/audit.log | jq '.'

# หา Secret Access
grep '"resource":"secrets"' /var/log/kubernetes/audit.log | \
  jq -r '. | "\(.user.username) \(.verb) \(.objectRef.namespace)/\(.objectRef.name)"'
```

### Section D: Pod Security

**Q16: สร้าง Pod ที่ Compliant กับ Restricted Policy**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

**Q17: Apply Pod Security Standard ให้ Namespace**
```bash
kubectl label namespace prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.28 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

**Q18: ทดสอบ Pod Security Standard**
```bash
# พยายามสร้าง Privileged Pod ใน restricted namespace
cat <<'EOF' | kubectl apply -n prod -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
# ควรได้รับ Error
```

**Q19: ใช้ AppArmor Profile**
```bash
# สร้างและ Load Profile
cat > /etc/apparmor.d/deny-write << 'EOF'
#include <tunables/global>
profile deny-write flags=(attach_disconnected) {
  #include <abstractions/base>
  file,
  deny /etc/** w,
  deny /var/** w,
}
EOF

apparmor_parser -r /etc/apparmor.d/deny-write

# ใช้ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/deny-write
spec:
  containers:
  - name: app
    image: nginx
```

**Q20: ใช้ Seccomp RuntimeDefault**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
```

### Section E: Supply Chain Security

**Q21: Scan Image ด้วย Trivy**
```bash
# Scan Image
trivy image --severity CRITICAL myapp:latest

# Fail ถ้ามี CRITICAL
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Scan Filesystem
trivy fs --severity HIGH,CRITICAL .

# Scan Kubernetes Manifests
trivy config --severity HIGH,CRITICAL ./k8s/
```

**Q22: ตรวจสอบ Image Digest**
```bash
# ดู Digest
docker inspect nginx:latest | jq '.[0].RepoDigests'

# ใช้ Digest ใน Pod (Immutable)
spec:
  containers:
  - name: nginx
    image: nginx@sha256:abc123def456...
```

**Q23: สร้าง Gatekeeper Policy สำหรับ Allowed Repos**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
  parameters:
    repos:
    - "company-registry.io/"
    - "gcr.io/trusted/"
```

**Q24: ตรวจสอบ Image ก่อน Deploy**
```bash
#!/bin/bash
IMAGE=$1
THRESHOLD=0

echo "Scanning $IMAGE..."
VULNS=$(trivy image --format json --severity CRITICAL $IMAGE | \
  jq '.Results[].Vulnerabilities | length // 0' | paste -sd+ | bc)

if [ "$VULNS" -gt "$THRESHOLD" ]; then
  echo "FAIL: $VULNS critical vulnerabilities found"
  exit 1
fi
echo "PASS: No critical vulnerabilities"
```

**Q25: ดู Image Vulnerability ใน JSON Format**
```bash
trivy image --format json nginx:latest | \
  jq '.Results[] | select(.Vulnerabilities) | .Vulnerabilities[] | select(.Severity=="CRITICAL") | "\(.VulnerabilityID): \(.Title)"'
```

### Section F: Falco และ Runtime Security

**Q26: สร้าง Falco Rule สำหรับ Detect Shell ใน Container**
```yaml
- rule: Unauthorized Shell in Container
  desc: Shell spawned unexpectedly
  condition: >
    spawned_process and container
    and proc.name in (sh, bash, zsh, csh, ksh)
    and not proc.pname in (allowed_parents)
  output: >
    Unauthorized shell in container
    (user=%user.name container=%container.name
    image=%container.image.repository proc=%proc.name
    cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

**Q27: ดู Falco Events ที่เกิดขึ้น**
```bash
# ดู Logs
kubectl logs -n falco -l app.kubernetes.io/name=falco -f

# Filter Events
kubectl logs -n falco -l app.kubernetes.io/name=falco | \
  grep "WARNING\|CRITICAL\|EMERGENCY"
```

**Q28: สร้าง Falco Rule สำหรับ Detect Crypto Miner**
```yaml
- rule: Detect Crypto Miner
  desc: Detect known crypto mining tools
  condition: >
    spawned_process and
    proc.name in (minerd, xmrig, ccminer, cgminer, bfgminer)
  output: >
    Crypto miner detected!
    (user=%user.name proc=%proc.name
    cmdline=%proc.cmdline container=%container.name)
  priority: CRITICAL
```

**Q29: Detect File Access ใน /etc**
```yaml
- rule: Read Sensitive File
  desc: Read sensitive system files
  condition: >
    open_read and
    (fd.name startswith /etc/shadow or
     fd.name startswith /etc/sudoers or
     fd.name startswith /etc/master.passwd) and
    not proc.name in (allowed_readers)
  output: >
    Sensitive file access
    (user=%user.name proc=%proc.name file=%fd.name)
  priority: WARNING
```

**Q30: สร้าง Falco Rule สำหรับ Network Activity**
```yaml
- rule: Unexpected Outbound Connection
  desc: Process makes unexpected outbound connection
  condition: >
    outbound and
    container and
    fd.net != "localhost" and
    not proc.name in (allowed_processes) and
    not fd.dport in (allowed_ports)
  output: >
    Unexpected outbound connection
    (user=%user.name proc=%proc.name
    connection=%fd.name container=%container.name)
  priority: NOTICE
```

### Section G: Advanced Security

**Q31: ค้นหา Pod ที่มี Privilege**
```bash
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[].securityContext.privileged==true) | "\(.metadata.namespace)/\(.metadata.name)"'
```

**Q32: ค้นหา Pod ที่ Mount hostPath**
```bash
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.hostPath) | 
  "\(.metadata.namespace)/\(.metadata.name): \(.spec.volumes[].hostPath.path // empty)"'
```

**Q33: ค้นหา Container ที่รันเป็น Root**
```bash
kubectl get pods -A -o json | \
  jq -r '.items[] | . as $pod | 
  .spec.containers[] | 
  select(.securityContext.runAsUser == 0 or .securityContext.runAsUser == null) |
  "\($pod.metadata.namespace)/\($pod.metadata.name): \(.name)"'
```

**Q34: Rotate etcd Encryption Key**
```bash
# 1. เพิ่ม Key ใหม่ ต่อท้ายรายการ
# 2. Restart API Server
# 3. Re-encrypt Resources
kubectl get secrets -A -o json | kubectl replace -f -
# 4. ลบ Key เก่า
```

**Q35: ตรวจสอบ Certificates ที่หมดอายุ**
```bash
kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
```

---

## สรุป CKS Exam

```
Key Security Concepts:
1. Defense in Depth
2. Least Privilege
3. Zero Trust Networking
4. Supply Chain Security
5. Runtime Threat Detection

Must Know:
- Network Policies (ทำ default deny)
- RBAC (ลด Privilege)
- Pod Security (runAsNonRoot, readOnlyRootFilesystem)
- Image Scanning (Trivy)
- Audit Logging
- Falco Rules
- OPA/Gatekeeper
- Encryption at Rest
- AppArmor/Seccomp
```

## References

- [CKS Official](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/)
- [CKS Study Guide](https://github.com/walidshaari/Certified-Kubernetes-Security-Specialist)
- [Kubernetes Security](https://kubernetes.io/docs/concepts/security/)
- [Falco Documentation](https://falco.org/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
