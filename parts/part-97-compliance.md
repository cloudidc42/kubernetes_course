# Part 97: Kubernetes Compliance

## บทนำ

Compliance ใน Kubernetes หมายถึงการปฏิบัติตามมาตรฐานและข้อกำหนดด้านความปลอดภัย เช่น CIS Benchmarks, PCI-DSS, HIPAA, SOC 2 บทนี้ครอบคลุมการตรวจสอบและ Implement Compliance Controls สำหรับ Kubernetes Cluster

## สารบัญ

1. Compliance Frameworks Overview
2. CIS Kubernetes Benchmark
3. Audit Logging
4. Policy as Code (OPA/Gatekeeper)
5. RBAC Compliance
6. Network Security Compliance
7. Container Security Compliance
8. Workshop: Compliance Audit

---

## 1. Compliance Frameworks Overview

### 1.1 Main Frameworks

```
Framework        | ใช้สำหรับ                    | สำคัญสำหรับ
-----------------|------------------------------|------------------
CIS Benchmarks   | General security hardening   | ทุกองค์กร
PCI-DSS         | Payment card industry         | Financial services
HIPAA           | Healthcare data protection    | Healthcare
SOC 2           | Service Organization Controls | SaaS providers
ISO 27001       | Information security          | Enterprise
NIST 800-53     | US Government                | Government/Defense
GDPR            | EU data protection            | EU businesses
```

### 1.2 Kubernetes-specific Compliance Requirements

```
Area               | Requirement
-------------------|--------------------------------------------------
API Security       | TLS encryption, Authentication, Authorization
etcd Security      | Encryption at rest, TLS client auth
Audit Logging      | Log all API server requests
Network Security   | Network policies, Encrypted traffic
Container Security | No privileged containers, read-only filesystem
RBAC               | Least privilege access
Secret Management  | Encrypted secrets, no hardcoded credentials
Image Security     | Signed images, vulnerability scanning
Node Security      | OS hardening, runtime security
```

---

## 2. CIS Kubernetes Benchmark

### 2.1 kube-bench (CIS Benchmark Scanner)

```bash
# ติดตั้งและรัน kube-bench
# สำหรับ Control Plane
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml

# สำหรับ Worker Node
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-node.yaml

# ดู Results
kubectl logs -f job/kube-bench-master
kubectl logs -f job/kube-bench-node

# รัน kube-bench โดยตรง
docker run --rm --pid=host \
  -v /etc/kubernetes:/etc/kubernetes \
  -v /var/lib/kubelet:/var/lib/kubelet \
  -v /etc/systemd:/etc/systemd \
  aquasec/kube-bench:latest --benchmark cis-1.8

# Export ผลเป็น JSON
docker run --rm --pid=host \
  -v /etc/kubernetes:/etc/kubernetes \
  aquasec/kube-bench:latest \
  --json > kube-bench-results.json
```

### 2.2 CIS Benchmark Results Analysis

```bash
#!/bin/bash
# analyze-cis.sh

RESULTS_FILE="kube-bench-results.json"

echo "=== CIS Kubernetes Benchmark Analysis ==="
echo ""

# สรุปผล
echo "Summary:"
jq -r '.Totals | "PASS: \(.total_pass), FAIL: \(.total_fail), WARN: \(.total_warn), INFO: \(.total_info)"' \
  $RESULTS_FILE

echo ""
echo "=== FAILED Checks ==="
jq -r '.Tests[].Results[] | 
  select(.status == "FAIL") | 
  "[\(.test_number)] \(.test_desc)\n  Remediation: \(.remediation)\n"' \
  $RESULTS_FILE

echo ""
echo "=== WARNING Checks ==="
jq -r '.Tests[].Results[] | 
  select(.status == "WARN") | 
  "[\(.test_number)] \(.test_desc)"' \
  $RESULTS_FILE
```

### 2.3 API Server Hardening (CIS Controls)

```yaml
# api-server-hardening.yaml
# เพิ่มใน /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    
    # CIS 1.2.1 - Ensure --anonymous-auth is set to false
    - --anonymous-auth=false
    
    # CIS 1.2.2 - Ensure --token-auth-file parameter is not set
    # (ไม่ใส่ --token-auth-file)
    
    # CIS 1.2.5 - Ensure --kubelet-client-certificate and --kubelet-client-key
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    
    # CIS 1.2.6 - Ensure --kubelet-certificate-authority
    - --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
    
    # CIS 1.2.7 - Ensure --authorization-mode includes Node,RBAC
    - --authorization-mode=Node,RBAC
    
    # CIS 1.2.9 - Ensure EventRateLimit
    - --enable-admission-plugins=NodeRestriction,EventRateLimit,PodSecurity
    - --admission-control-config-file=/etc/kubernetes/admission-config.yaml
    
    # CIS 1.2.10 - Ensure --disable-admission-plugins does not contain dangerous
    # (ไม่ใส่ AlwaysAdmit, AlwaysPullImages ถ้าไม่จำเป็น)
    
    # CIS 1.2.12 - Ensure AlwaysPullImages is set if using shared registry
    # - --enable-admission-plugins=...,AlwaysPullImages
    
    # CIS 1.2.16 - Ensure --audit-log-path is configured
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    
    # CIS 1.2.17 - Ensure request timeout is configured
    - --request-timeout=300s
    
    # CIS 1.2.18 - Ensure --service-account-lookup is true
    - --service-account-lookup=true
    
    # CIS 1.2.21 - Ensure TLS is configured
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    
    # CIS 1.2.24 - Ensure --service-account-signing-key-file
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-account-issuer=https://kubernetes.default.svc
    
    # CIS 1.2.29 - Ensure Encryption at Rest
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    
    # CIS 1.2.30 - Ensure --profiling is false
    - --profiling=false
```

### 2.4 etcd Hardening (CIS Controls)

```bash
# CIS 2.1 - Ensure TLS is enabled
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# CIS 2.2 - Ensure peer TLS
# ตรวจสอบ etcd config
grep -E "(peer|client)-cert-file|trusted-ca-file" /etc/kubernetes/manifests/etcd.yaml

# CIS 2.6 - Ensure etcd is not running as root
# (k8s managed etcd รันเป็น etcd user)
```

---

## 3. Audit Logging

### 3.1 Comprehensive Audit Policy

```yaml
# comprehensive-audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- "RequestReceived"

rules:
# ========== Sensitive Operations - Full Log ==========

# Log ALL Secret operations (full request/response)
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete", "deletecollection"]
  resources:
  - group: ""
    resources: ["secrets"]
  - group: "v1"
    resources: ["secrets"]

# Log SA token creation
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: "authentication.k8s.io"
    resources: ["tokenreviews"]
  - group: ""
    resources: ["serviceaccounts/token"]

# Log RBAC changes
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete", "deletecollection"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources:
    - "roles"
    - "rolebindings"
    - "clusterroles"
    - "clusterrolebindings"

# Log Pod Security changes
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "policy"
    resources: ["podsecuritypolicies"]
  - group: "networking.k8s.io"
    resources: ["networkpolicies"]

# ========== Exec/Attach - Full Log ==========
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]

# ========== Workload Changes - Log ==========
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["pods", "services", "configmaps"]
  - group: "apps"
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
  - group: "batch"
    resources: ["jobs", "cronjobs"]

# ========== Namespace Operations ==========
- level: RequestResponse
  verbs: ["create", "delete"]
  resources:
  - group: ""
    resources: ["namespaces"]

# ========== Node Operations ==========
- level: Request
  resources:
  - group: ""
    resources: ["nodes", "nodes/status"]
  verbs: ["create", "update", "patch", "delete"]

# ========== Metadata Only ==========
# API Discovery
- level: Metadata
  nonResourceURLs:
  - "/api"
  - "/api/*"
  - "/apis"
  - "/apis/*"
  verbs: ["get", "list", "watch"]

# ========== No Log ==========
# Skip noisy events
- level: None
  resources:
  - group: ""
    resources: ["events"]

# Skip system components
- level: None
  users:
  - "system:kube-scheduler"
  - "system:kube-controller-manager"
  
- level: None
  userGroups:
  - "system:nodes"

# Skip get/list/watch
- level: None
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources:
    - "configmaps"
    - "endpoints"
    - "services"
  - group: "apps"
    resources:
    - "deployments"
    - "replicasets"

# Default catch-all
- level: Metadata
```

### 3.2 Audit Log Processing

```bash
#!/bin/bash
# audit-log-analysis.sh

AUDIT_LOG="/var/log/kubernetes/audit.log"

echo "=== Audit Log Analysis ==="
echo "Log: $AUDIT_LOG"
echo ""

# Privileged Actions
echo "1. Privileged Container Creations:"
jq -r 'select(.verb == "create" and .objectRef.resource == "pods") |
  select(.requestObject.spec.containers[].securityContext.privileged == true) |
  "\(.requestReceivedTimestamp) - \(.user.username) created privileged pod \(.objectRef.namespace)/\(.objectRef.name)"' \
  $AUDIT_LOG

echo ""
echo "2. Secret Access (Last 100 events):"
jq -r 'select(.objectRef.resource == "secrets") |
  "\(.requestReceivedTimestamp) - \(.user.username): \(.verb) secret \(.objectRef.namespace)/\(.objectRef.name)"' \
  $AUDIT_LOG | tail -100

echo ""
echo "3. RBAC Changes:"
jq -r 'select(.objectRef.apiGroup == "rbac.authorization.k8s.io") |
  select(.verb | test("create|update|delete")) |
  "\(.requestReceivedTimestamp) - \(.user.username): \(.verb) \(.objectRef.resource) \(.objectRef.name)"' \
  $AUDIT_LOG

echo ""
echo "4. Failed Authentication (403/401):"
jq -r 'select(.responseStatus.code | test("401|403")) |
  "\(.requestReceivedTimestamp) - \(.user.username) - \(.verb) \(.objectRef.resource) - HTTP \(.responseStatus.code)"' \
  $AUDIT_LOG | tail -50

echo ""
echo "5. exec/attach/portforward:"
jq -r 'select(.objectRef.subresource | test("exec|attach|portforward")) |
  "\(.requestReceivedTimestamp) - \(.user.username): \(.objectRef.subresource) on \(.objectRef.namespace)/\(.objectRef.name)"' \
  $AUDIT_LOG
```

### 3.3 Falco สำหรับ Runtime Security Audit

```bash
# ติดตั้ง Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl=https://hooks.slack.com/services/xxx \
  --set tty=true
```

```yaml
# falco-rules.yaml
- rule: Terminal Shell in Container
  desc: A shell was spawned in a container with an attached terminal
  condition: >
    spawned_process
    and container
    and shell_procs
    and proc.tty != 0
    and not container.image.repository in (allowed_image_list)
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name %container.info shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: NOTICE
  tags: [container, shell, mitre_execution]

- rule: Read sensitive file untrusted
  desc: An attempt to read any sensitive file (e.g. files containing user/password/authentication information)
  condition: >
    sensitive_files and open_read
    and not proc.name in (user_mgmt_binaries, userexec_binaries, package_mgmt_procs)
    and not container.image.repository in (trusted_containers)
    and not user.name = "root"
  output: >
    Sensitive file opened for reading by non-trusted program
    (user=%user.name command=%proc.cmdline file=%fd.name %container.info)
  priority: WARNING
  tags: [filesystem, mitre_credential_access]

- rule: Write below binary dir
  desc: An attempt to write to any file below a set of binary directories
  condition: >
    bin_dir and evt.dir = < and open_write and not package_mgmt_procs
  output: >
    File below a known binary directory opened for writing
    (user=%user.name command=%proc.cmdline file=%fd.name %container.info)
  priority: ERROR
  tags: [filesystem, mitre_persistence]

- rule: K8s Secret Access
  desc: Attempt to access Kubernetes secrets in a pod
  condition: >
    ka.target.resource = "secrets" and
    not ka.user.name startswith "system:" and
    ka.verb in (get, list, watch)
  output: >
    Kubernetes secret access
    (user=%ka.user.name verb=%ka.verb secret=%ka.target.name ns=%ka.target.namespace)
  priority: WARNING
  source: k8s_audit
  tags: [k8s]
```

---

## 4. Policy as Code (OPA/Gatekeeper)

### 4.1 ติดตั้ง OPA Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml

# ตรวจสอบ
kubectl get pods -n gatekeeper-system
```

### 4.2 Constraint Templates

```yaml
# constraints.yaml

# 1. Require Team Label
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

      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("you must provide labels: %v", [missing])
      }
---
# Apply Constraint
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
      kinds: ["Deployment", "StatefulSet"]
  parameters:
    labels:
    - team
    - project
    - environment
---

# 2. Disallow Privileged Containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8spspprivilegedcontainer

      violation[{"msg": msg}] {
        c := input_containers[_]
        c.securityContext.privileged
        msg := sprintf("Privileged container is not allowed: %v", [c.name])
      }

      input_containers[c] {
        c := input.review.object.spec.containers[_]
      }

      input_containers[c] {
        c := input.review.object.spec.initContainers[_]
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
---

# 3. Require Resource Limits
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequireLimits
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequirelimits

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits.cpu
        msg := sprintf("Container '%v' must set cpu limits", [container.name])
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits.memory
        msg := sprintf("Container '%v' must set memory limits", [container.name])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireLimits
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "monitoring"]
---

# 4. Allowed Registry Policy
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
        not startswith_any(container.image, input.parameters.repos)
        msg := sprintf("Container image '%v' comes from disallowed registry", [container.image])
      }

      startswith_any(str, patterns) {
        startswith(str, patterns[_])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-image-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
  parameters:
    repos:
    - "mycompany.azurecr.io/"
    - "ghcr.io/mycompany/"
    - "registry.k8s.io/"
    - "gcr.io/google_containers/"
```

### 4.3 Kyverno Alternative

```bash
# ติดตั้ง Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set replicaCount=3
```

```yaml
# kyverno-policies.yaml
# Auto-add Labels Policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
  - name: add-labels
    match:
      any:
      - resources:
          kinds:
          - Deployment
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            managed-by: kyverno
            last-applied: "{{ time_truncate_unix('{{request.timestamp}}', '1') }}"
---
# Require Non-Root Policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-nonroot
spec:
  validationFailureAction: enforce
  rules:
  - name: check-containers
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Containers must run as non-root user"
      pattern:
        spec:
          containers:
          - (name): "*"
            securityContext:
              runAsNonRoot: "true"
---
# Generate NetworkPolicy Policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-networkpolicy
spec:
  rules:
  - name: deny-all-traffic
    match:
      any:
      - resources:
          kinds:
          - Namespace
    generate:
      kind: NetworkPolicy
      apiVersion: networking.k8s.io/v1
      name: deny-all
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      data:
        spec:
          podSelector: {}
          policyTypes:
          - Ingress
          - Egress
```

---

## 5. RBAC Compliance

### 5.1 RBAC Audit

```bash
#!/bin/bash
# rbac-audit.sh

echo "=== RBAC Compliance Audit ==="

echo "1. ClusterRoleBindings with cluster-admin:"
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  "Name: \(.metadata.name)\nSubjects: \(.subjects | map("\(.kind)/\(.name)") | join(", "))\n"
'

echo ""
echo "2. ServiceAccounts with cluster-admin:"
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  .subjects[]? |
  select(.kind == "ServiceAccount") |
  "\(.namespace)/\(.name)"
'

echo ""
echo "3. Roles with wildcard permissions (*):"
kubectl get clusterroles,roles -A -o json | jq -r '
  .items[] |
  select(.rules[]? | .verbs[]? == "*" or .resources[]? == "*") |
  "\(.metadata.namespace // "cluster")/\(.metadata.name)"
' | sort -u

echo ""
echo "4. Unused ServiceAccounts (no pods using them):"
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  for sa in $(kubectl get sa -n $ns -o name | cut -d/ -f2 | grep -v default); do
    POD_COUNT=$(kubectl get pods -n $ns -o json | \
      jq -r --arg SA "$sa" \
      '[.items[] | select(.spec.serviceAccountName == $SA)] | length')
    if [ "$POD_COUNT" = "0" ]; then
      echo "$ns/$sa (unused)"
    fi
  done
done

echo ""
echo "5. Default ServiceAccount Tokens (should be disabled):"
kubectl get serviceaccounts -A -o json | jq -r '
  .items[] |
  select(.metadata.name == "default") |
  select(.automountServiceAccountToken != false) |
  "\(.metadata.namespace)/default: automount is enabled!"
'
```

### 5.2 RBAC Hardening

```yaml
# rbac-hardening.yaml
# ปิด Default ServiceAccount Auto-mount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: production
automountServiceAccountToken: false
---
# Minimal Role สำหรับ Application
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
# อ่าน ConfigMaps ที่จำเป็น
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get", "watch"]
# อ่าน Secrets ที่จำเป็น
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secrets"]
  verbs: ["get"]
---
# ServiceAccount สำหรับ App
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
automountServiceAccountToken: true  # เปิดเฉพาะที่จำเป็น
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
```

---

## 6. Network Security Compliance

### 6.1 Network Policies Template

```yaml
# baseline-network-policies.yaml
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
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
---
# Allow Internal Communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
---
# Allow Monitoring (Prometheus)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9090
    - port: 8080
    - port: 9100
```

### 6.2 mTLS ด้วย Istio

```yaml
# mtls-policy.yaml
# Enable mTLS สำหรับทุก Communications
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  action: ALLOW
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

## 7. Container Security Compliance

### 7.1 Pod Security Standards

```yaml
# pod-security-standards.yaml
# Namespace Level Policy
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Compliant Pod Template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compliant-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: compliant-app
  template:
    metadata:
      labels:
        app: compliant-app
    spec:
      # Required for restricted pod security standard
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      # No service account token mount unless needed
      automountServiceAccountToken: false
      
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
        
        # Security Context - Required
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
          # Optional: add specific capabilities if needed
          # add:
          # - NET_BIND_SERVICE
        
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        
        # Read-only filesystem requires tmpfs for writable dirs
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/app
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

### 7.2 Image Signing ด้วย Cosign

```bash
# ติดตั้ง Cosign
curl -LO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign

# สร้าง Key Pair
cosign generate-key-pair

# Sign Image
cosign sign --key cosign.key myregistry.example.com/myapp:v1

# Verify Image
cosign verify \
  --key cosign.pub \
  myregistry.example.com/myapp:v1

# ติดตั้ง Sigstore Policy Controller
kubectl apply -f https://github.com/sigstore/policy-controller/releases/latest/download/policy-controller.yaml

# สร้าง ClusterImagePolicy
cat <<'EOF' | kubectl apply -f -
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
  - glob: "myregistry.example.com/**"
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        <your-cosign-public-key>
        -----END PUBLIC KEY-----
EOF
```

---

## 8. Workshop: Compliance Audit

### Workshop Overview

ทำ Full Compliance Audit สำหรับ Kubernetes Cluster

### Step 1: Run CIS Benchmark

```bash
#!/bin/bash
# run-compliance-audit.sh

echo "=== Starting Compliance Audit ==="
AUDIT_DIR="/tmp/compliance-audit-$(date +%Y%m%d_%H%M%S)"
mkdir -p $AUDIT_DIR

# 1. CIS Benchmark
echo "Running CIS Benchmark..."
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
sleep 60  # รอ Job เสร็จ
kubectl logs job/kube-bench-master > $AUDIT_DIR/cis-master.txt
kubectl delete job kube-bench-master

kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-node.yaml
sleep 60
kubectl logs job/kube-bench-node > $AUDIT_DIR/cis-node.txt
kubectl delete job kube-bench-node

echo "CIS Benchmark done: $AUDIT_DIR/cis-*.txt"
```

### Step 2: Run RBAC Audit

```bash
# 2. RBAC Audit
echo "Running RBAC Audit..."
cat > $AUDIT_DIR/rbac-audit.txt << 'EOF'
=== RBAC Audit ===
EOF

# Cluster Admin Bindings
echo "--- cluster-admin bindings ---" >> $AUDIT_DIR/rbac-audit.txt
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name == "cluster-admin") | 
  "Name: \(.metadata.name) | Subjects: \(.subjects | map(.name) | join(","))"' \
  >> $AUDIT_DIR/rbac-audit.txt

# Wildcard permissions
echo "--- Wildcard permissions ---" >> $AUDIT_DIR/rbac-audit.txt
kubectl get clusterroles -o json | \
  jq -r '.items[] | select(.rules[]?.verbs[]? == "*") | .metadata.name' \
  >> $AUDIT_DIR/rbac-audit.txt
```

### Step 3: Check Pod Security

```bash
# 3. Pod Security Check
echo "Checking Pod Security..."
cat > $AUDIT_DIR/pod-security.txt << 'EOF'
=== Pod Security Audit ===
EOF

# Privileged Containers
echo "--- Privileged Containers ---" >> $AUDIT_DIR/pod-security.txt
kubectl get pods -A -o json | \
  jq -r '.items[] | 
  select(.spec.containers[].securityContext.privileged == true) |
  "\(.metadata.namespace)/\(.metadata.name)"' \
  >> $AUDIT_DIR/pod-security.txt

# Root Containers
echo "--- Running as Root ---" >> $AUDIT_DIR/pod-security.txt
kubectl get pods -A -o json | \
  jq -r '.items[] |
  select(.spec.securityContext.runAsUser == 0 or
         .spec.securityContext.runAsNonRoot == false) |
  "\(.metadata.namespace)/\(.metadata.name)"' \
  >> $AUDIT_DIR/pod-security.txt

# No Resource Limits
echo "--- No Resource Limits ---" >> $AUDIT_DIR/pod-security.txt
kubectl get pods -A -o json | \
  jq -r '.items[] |
  select(.spec.containers[].resources.limits == null) |
  "\(.metadata.namespace)/\(.metadata.name)"' \
  >> $AUDIT_DIR/pod-security.txt
```

### Step 4: Generate Compliance Report

```bash
#!/bin/bash
# generate-report.sh

AUDIT_DIR="/tmp/compliance-audit-$(date +%Y%m%d)"

cat > $AUDIT_DIR/compliance-report.md << 'EOF'
# Kubernetes Compliance Audit Report

**Date:** $(date)
**Cluster:** $(kubectl config current-context)
**Auditor:** $(whoami)

## Executive Summary

| Category | Status | Score |
|----------|--------|-------|
| CIS Benchmark | ⚠️ Partial | 75/100 |
| RBAC Security | ✅ Pass | 90/100 |
| Pod Security | ⚠️ Partial | 80/100 |
| Network Security | ✅ Pass | 95/100 |
| Audit Logging | ✅ Pass | 100/100 |
| Image Security | ❌ Fail | 50/100 |

## Findings

### Critical (Fix Immediately)
1. **Privileged containers found** in namespace `default`
   - Impact: Container escape possible
   - Remediation: Remove `securityContext.privileged: true`

2. **No image signing policy** enforced
   - Impact: Untrusted images can be deployed
   - Remediation: Implement Cosign + Policy Controller

### High
1. **Default ServiceAccounts with auto-mount enabled**
   - 5 namespaces affected
   - Remediation: Set `automountServiceAccountToken: false`

2. **cluster-admin bindings for non-system users**
   - 2 users with cluster-admin
   - Remediation: Review and reduce to minimum required

### Medium
1. **Missing Resource Limits** in 15 pods
   - Remediation: Add resource limits to all containers

2. **Namespace without NetworkPolicy** - 3 namespaces
   - Remediation: Apply default deny policy

## Remediation Plan

| Finding | Priority | Owner | Due Date |
|---------|----------|-------|----------|
| Privileged containers | P0 | Platform Team | Week 1 |
| Image signing | P1 | DevOps Team | Week 2 |
| ServiceAccount tokens | P1 | Platform Team | Week 1 |
| Resource limits | P2 | Dev Teams | Week 3 |
EOF

echo "Report generated: $AUDIT_DIR/compliance-report.md"
cat $AUDIT_DIR/compliance-report.md
```

### Step 5: Apply Remediation

```bash
#!/bin/bash
# apply-remediation.sh

echo "=== Applying Security Remediations ==="

# 1. Disable Default SA auto-mount
for ns in $(kubectl get ns -o name | cut -d/ -f2 | grep -v kube-); do
  kubectl patch serviceaccount default -n $ns -p '{"automountServiceAccountToken": false}' 2>/dev/null || true
done
echo "1. Default SA auto-mount disabled"

# 2. Apply Network Policies
for ns in $(kubectl get ns -o name | cut -d/ -f2 | grep -v kube-system); do
  kubectl apply -n $ns -f - << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
done
echo "2. Default deny ingress applied to all namespaces"

# 3. Enable Pod Security Standards
for ns in production staging; do
  kubectl label namespace $ns \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/audit=restricted \
    pod-security.kubernetes.io/warn=restricted \
    --overwrite 2>/dev/null || true
done
echo "3. Pod Security Standards enabled"

# 4. Apply Gatekeeper Policies
kubectl apply -f constraints.yaml
echo "4. Gatekeeper Constraints applied"

echo "=== Remediation Complete ==="
echo "Run compliance audit again to verify fixes."
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Compliance Frameworks** - CIS, PCI-DSS, HIPAA, SOC 2
2. **CIS Benchmark** - kube-bench, analysis, hardening
3. **Audit Logging** - Comprehensive audit policy, log analysis
4. **Policy as Code** - OPA/Gatekeeper, Kyverno
5. **RBAC Compliance** - Audit, hardening
6. **Network Security** - Policies, mTLS
7. **Container Security** - Pod Security Standards, Image Signing
8. **Workshop** - Full Compliance Audit

## แบบฝึกหัด

1. รัน kube-bench และแก้ไข FAIL findings ทั้งหมด
2. ตั้งค่า Audit Logging ด้วย Policy ที่ครอบคลุม
3. Implement OPA/Gatekeeper เพื่อ enforce Image Registry Policy
4. ทำ RBAC Audit และลบ Cluster Admin ที่ไม่จำเป็น
5. เขียน Compliance Report สำหรับ Cluster ของคุณ

## References

- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [kube-bench](https://github.com/aquasecurity/kube-bench)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Falco](https://falco.org/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
