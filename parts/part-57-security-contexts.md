# Part 57: Security Contexts

## บทนำ

Security Context กำหนด privilege และ access control settings สำหรับ Pod หรือ Container ซึ่งรวมถึง:
- ระบุ user/group ID ที่รัน process
- Linux capabilities
- SELinux labels
- AppArmor profiles
- seccomp profiles
- Read-only filesystem

---

## 57.1 SecurityContext ระดับ Pod vs Container

### Pod Level SecurityContext

```yaml
spec:
  securityContext:
    # ใช้กับทุก containers ใน pod
    runAsUser: 1000           # UID
    runAsGroup: 3000          # GID
    fsGroup: 2000             # Group ID สำหรับ volumes
    runAsNonRoot: true        # ห้าม run as root
    
    # Sysctls
    sysctls:
    - name: net.ipv4.tcp_syncookies
      value: "1"
    
    # SELinux
    seLinuxOptions:
      level: "s0:c123,c456"
    
    # seccomp
    seccompProfile:
      type: RuntimeDefault
```

### Container Level SecurityContext

```yaml
containers:
- name: myapp
  securityContext:
    # override pod-level settings
    runAsUser: 2000           # ต่างจาก pod level
    runAsGroup: 3000
    runAsNonRoot: true
    
    # Read-only root filesystem
    readOnlyRootFilesystem: true
    
    # ห้าม privilege escalation
    allowPrivilegeEscalation: false
    
    # Privileged container (ห้ามใช้ใน production)
    privileged: false
    
    # Linux capabilities
    capabilities:
      add:
      - NET_BIND_SERVICE     # เพิ่ม specific capability
      drop:
      - ALL                  # ลบทุก capability ก่อน
    
    # seccomp profile
    seccompProfile:
      type: RuntimeDefault
    
    # AppArmor (ระบุใน annotations)
    # ดูด้านล่าง
```

---

## 57.2 runAsUser และ runAsGroup

### ทำไมต้อง Run as Non-root?

```
ถ้า container รัน as root:
- Process มีสิทธิ์เต็มใน container filesystem
- Kernel exploits อาจ escalate สู่ host root
- ไม่เป็นไปตาม security standards

เป้าหมาย: รัน as unprivileged user
```

### runAsUser/Group

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  securityContext:
    runAsUser: 1000    # UID 1000
    runAsGroup: 1000   # GID 1000
    fsGroup: 1000      # กำหนด group ownership ของ volumes
  
  containers:
  - name: demo
    image: nginx:1.25-alpine
    
    securityContext:
      runAsNonRoot: true  # reject ถ้า image ใช้ root
      allowPrivilegeEscalation: false
    
    command: ["sh", "-c"]
    args:
    - |
      echo "Running as user: $(id)"
      echo "User: $(whoami)"
      sleep infinity
```

```bash
kubectl apply -f security-demo.yaml

# ตรวจสอบ user
kubectl exec -it security-demo -- id
# uid=1000 gid=1000 groups=1000

kubectl exec -it security-demo -- whoami
# สามารถหา username ถ้ามี /etc/passwd entry
```

### Volume Permissions กับ fsGroup

```yaml
spec:
  securityContext:
    fsGroup: 2000    # files ใน volumes จะ owned by group 2000
  
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
  
  volumes:
  - name: data
    emptyDir: {}
```

```bash
kubectl exec -it <pod> -- ls -la /data
# drwxrwsr-x  2 root  2000  ... (group 2000)
```

### fsGroupChangePolicy

```yaml
spec:
  securityContext:
    fsGroup: 2000
    # Always: เปลี่ยน ownership ทุกครั้งที่ mount (ช้า)
    # OnRootMismatch: เปลี่ยนเฉพาะถ้า root ไม่ match (เร็วกว่า)
    fsGroupChangePolicy: "OnRootMismatch"
```

---

## 57.3 Linux Capabilities

### Capabilities คืออะไร?

```
Traditional Unix: root = ทำได้ทุกอย่าง, non-root = จำกัด
Linux Capabilities: แบ่ง root privilege เป็น "capabilities" เล็กๆ

ตัวอย่าง capabilities:
CAP_NET_BIND_SERVICE - bind port < 1024
CAP_NET_ADMIN        - network interface administration
CAP_SYS_ADMIN        - สิทธิ์ system admin หลายอย่าง (อันตราย)
CAP_CHOWN            - เปลี่ยน file ownership
CAP_DAC_OVERRIDE     - bypass file permission checks
CAP_KILL             - ส่ง signal ไปยัง processes

Default capabilities ที่ container ได้รับ:
AUDIT_WRITE, CHOWN, DAC_OVERRIDE, FOWNER, FSETID,
KILL, MKNOD, NET_BIND_SERVICE, NET_RAW, SETFCAP,
SETGID, SETPCAP, SETUID, SYS_CHROOT
```

### Drop All และ Add เฉพาะที่จำเป็น

```yaml
containers:
- name: app
  securityContext:
    capabilities:
      drop:
      - ALL              # ลบทุก capability ก่อน
      add:
      - NET_BIND_SERVICE  # เพิ่มเฉพาะที่จำเป็น (bind port < 1024)
```

### ตัวอย่างตาม Use Case

```yaml
# Web server ที่ต้อง bind port 80
containers:
- name: nginx
  securityContext:
    capabilities:
      drop: ["ALL"]
      add: ["NET_BIND_SERVICE"]
    runAsUser: 1000
    readOnlyRootFilesystem: true

---
# Network debugging tool
containers:
- name: netdebug
  securityContext:
    capabilities:
      drop: ["ALL"]
      add: ["NET_ADMIN", "NET_RAW"]  # สำหรับ ping, tcpdump

---
# Minimal app - ไม่ต้องการ capabilities เพิ่มเติม
containers:
- name: backend
  securityContext:
    capabilities:
      drop: ["ALL"]   # drop ทุกอย่าง
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
```

---

## 57.4 readOnlyRootFilesystem

### ทำไมต้องใช้?

```
ถ้า container ถูก compromise:
- ผู้โจมตีไม่สามารถแก้ไข/เพิ่ม executable files
- ไม่สามารถเขียน malware
- ลดความเสียหายที่เกิดขึ้นได้
```

### การใช้งาน

```yaml
containers:
- name: myapp
  securityContext:
    readOnlyRootFilesystem: true   # root FS read-only
  
  # ต้องระบุ writable paths สำหรับ runtime data
  volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: var-run
    mountPath: /var/run
  - name: logs
    mountPath: /var/log/app

volumes:
- name: tmp
  emptyDir: {}
- name: var-run
  emptyDir: {}
- name: logs
  emptyDir: {}
```

### Nginx กับ readOnlyRootFilesystem

```yaml
# Nginx ต้องการ writable directories
containers:
- name: nginx
  image: nginx:1.25-alpine
  securityContext:
    readOnlyRootFilesystem: true
    runAsNonRoot: true
    runAsUser: 101  # nginx user
    capabilities:
      drop: ["ALL"]
      add: ["NET_BIND_SERVICE"]
  
  volumeMounts:
  # Nginx ต้องการ paths เหล่านี้ writable
  - name: nginx-cache
    mountPath: /var/cache/nginx
  - name: nginx-run
    mountPath: /var/run
  - name: nginx-log
    mountPath: /var/log/nginx

volumes:
- name: nginx-cache
  emptyDir: {}
- name: nginx-run
  emptyDir: {}
- name: nginx-log
  emptyDir: {}
```

---

## 57.5 Privileged Containers

### อย่าใช้ Privileged Containers

```yaml
# ห้ามทำแบบนี้ใน production!
containers:
- name: dangerous
  securityContext:
    privileged: true    # เข้าถึง host kernel เกือบทั้งหมด
                        # เหมือนรัน root บน host โดยตรง
```

### ข้อยกเว้นที่ยอมรับได้ (จำกัดมาก)

```yaml
# สำหรับ DaemonSet ที่ต้องการ access host
# เช่น network plugins, storage plugins, monitoring agents
# แต่ต้องใช้ด้วยความระมัดระวังสูง

containers:
- name: cni-plugin
  securityContext:
    privileged: true    # จำเป็นสำหรับ network plugin
    # ควรจำกัดด้วย NodeSelector
```

---

## 57.6 Seccomp Profiles

Seccomp (Secure Computing Mode) จำกัด system calls ที่ process สามารถเรียกใช้ได้

### RuntimeDefault

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault    # ใช้ default ของ container runtime (Docker, containerd)
                              # ปลอดภัยกว่า Unconfined
  
  containers:
  - name: app
    securityContext:
      seccompProfile:
        type: RuntimeDefault  # container-level override
```

### Localhost Profile

```yaml
# ใช้ custom seccomp profile
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: "profiles/my-profile.json"  # relative to /var/lib/kubelet/seccomp/
```

### Custom Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "accept4", "epoll_wait", "pselect6", "futex",
        "madvise", "epoll_ctl", "getsockname",
        "setsockopt", "vfork", "mmap", "read", "write",
        "close", "arch_prctl", "sched_getaffinity",
        "munmap", "brk", "rt_sigaction", "rt_sigprocmask",
        "gettid", "clone", "bind", "socket", "openat",
        "renameat2", "sigaltstack", "exit", "getpid",
        "nanosleep", "fstat", "mprotect", "uname"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

## 57.7 AppArmor

AppArmor เป็น Linux Security Module ที่จำกัดสิทธิ์ของ processes

```yaml
# AppArmor ระบุผ่าน annotations (ยังไม่ใช่ field ใน spec)
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/nginx-restricted

spec:
  containers:
  - name: nginx
    # AppArmor profile จะถูกใช้กับ nginx container
```

### AppArmor Profile ตัวอย่าง

```
#include <tunables/global>

profile nginx-restricted flags=(attach_disconnected) {
  #include <abstractions/base>
  
  # อนุญาต network
  network inet tcp,
  network inet udp,
  
  # อ่าน config
  /etc/nginx/** r,
  /usr/share/nginx/** r,
  
  # เขียน log
  /var/log/nginx/** w,
  /var/run/nginx.pid rw,
  
  # ห้ามเข้าถึง sensitive paths
  deny /etc/passwd r,
  deny /etc/shadow r,
  deny /proc/** r,
}
```

---

## 57.8 Workshop: Hardened Pod Security

### สถานการณ์

เราจะ harden Nginx deployment ให้มีความปลอดภัยสูงสุด:
- Run as non-root user
- Read-only filesystem
- Drop all capabilities
- RuntimeDefault seccomp
- Specific writable volumes

### Step 1: Setup

```bash
kubectl create namespace workshop-security
kubectl config set-context --current --namespace=workshop-security
```

### Step 2: ทดสอบ Insecure Container ก่อน

```yaml
# insecure-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-nginx
  namespace: workshop-security
spec:
  containers:
  - name: nginx
    image: nginx:1.25-alpine
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f insecure-nginx.yaml
kubectl wait --for=condition=ready pod/insecure-nginx -n workshop-security --timeout=30s

# ดูว่า nginx รันเป็น root
kubectl exec -it insecure-nginx -n workshop-security -- id
# uid=0(root) gid=0(root) groups=0(root)

# สามารถเขียน filesystem ได้
kubectl exec -it insecure-nginx -n workshop-security -- touch /test-file
kubectl exec -it insecure-nginx -n workshop-security -- ls /test-file

kubectl delete pod insecure-nginx -n workshop-security
```

### Step 3: สร้าง Hardened Nginx

```yaml
# hardened-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hardened-nginx
  namespace: workshop-security
  labels:
    app: hardened-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hardened-nginx
  template:
    metadata:
      labels:
        app: hardened-nginx
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true      # ห้าม run as root
        runAsUser: 101          # nginx user (UID 101 ใน alpine)
        runAsGroup: 101         # nginx group
        fsGroup: 101            # volume group ownership
        seccompProfile:
          type: RuntimeDefault  # enable seccomp
      
      # ไม่ mount service account token
      automountServiceAccountToken: false
      
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 8080  # ใช้ port > 1024 เพื่อไม่ต้องใช้ capability
        
        # Container-level security
        securityContext:
          allowPrivilegeEscalation: false   # ห้าม privilege escalation
          readOnlyRootFilesystem: true       # read-only root
          runAsNonRoot: true
          runAsUser: 101
          capabilities:
            drop:
            - ALL                            # drop ทุก capability
            # ไม่ add NET_BIND_SERVICE เพราะใช้ port 8080
        
        # Volume mounts สำหรับ writable paths
        volumeMounts:
        - name: nginx-cache
          mountPath: /var/cache/nginx
        - name: nginx-run
          mountPath: /var/run
        - name: nginx-logs
          mountPath: /var/log/nginx
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/
          readOnly: true
        
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
        
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
      
      volumes:
      # Writable emptyDir volumes
      - name: nginx-cache
        emptyDir: {}
      - name: nginx-run
        emptyDir: {}
      - name: nginx-logs
        emptyDir: {}
      # ConfigMap สำหรับ nginx config
      - name: nginx-config
        configMap:
          name: hardened-nginx-config
---
# ConfigMap สำหรับ Nginx
apiVersion: v1
kind: ConfigMap
metadata:
  name: hardened-nginx-config
  namespace: workshop-security
data:
  default.conf: |
    server {
        listen 8080;  # port > 1024
        server_name _;
        root /usr/share/nginx/html;
        
        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000" always;
        add_header Content-Security-Policy "default-src 'self'" always;
        
        # Hide server info
        server_tokens off;
        
        location / {
            return 200 'Hardened Nginx - Security Workshop';
            add_header Content-Type text/plain;
        }
        
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: hardened-nginx
  namespace: workshop-security
spec:
  selector:
    app: hardened-nginx
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f hardened-nginx.yaml
kubectl get pods -n workshop-security -w
```

### Step 4: ตรวจสอบ Security Settings

```bash
# รอ pods พร้อม
kubectl wait --for=condition=ready pod -l app=hardened-nginx \
  -n workshop-security --timeout=60s

POD=$(kubectl get pod -l app=hardened-nginx -n workshop-security -o jsonpath='{.items[0].metadata.name}')

# ตรวจสอบ user
kubectl exec -it $POD -n workshop-security -- id
# uid=101(nginx) gid=101(nginx) groups=101(nginx)

# ทดสอบ read-only filesystem
kubectl exec -it $POD -n workshop-security -- touch /test-file 2>&1
# touch: /test-file: Read-only file system (expected!)

# ทดสอบว่า writable dirs ทำงานได้
kubectl exec -it $POD -n workshop-security -- touch /var/cache/nginx/test-file
kubectl exec -it $POD -n workshop-security -- ls /var/cache/nginx/

# ทดสอบ no privilege escalation
kubectl exec -it $POD -n workshop-security -- su root 2>&1
# su: Permission denied

# ตรวจสอบ capabilities
kubectl exec -it $POD -n workshop-security -- cat /proc/1/status | grep CapEff
# CapEff: 0000000000000000 (ไม่มี capabilities!)
```

### Step 5: ทดสอบ Network

```bash
kubectl port-forward svc/hardened-nginx 8080:80 -n workshop-security &

curl http://localhost:8080/
# Hardened Nginx - Security Workshop

curl -I http://localhost:8080/
# ดู security headers

kill %1
```

### Step 6: Security Scanning

```bash
# ใช้ kube-score เพื่อ check security settings
# ติดตั้ง kube-score
curl -L https://github.com/zegl/kube-score/releases/latest/download/kube-score_linux_amd64.tar.gz | tar xz
sudo mv kube-score /usr/local/bin/

# Scan manifest
kube-score score hardened-nginx.yaml

# ใช้ trivy สำหรับ scan config
# trivy config hardened-nginx.yaml
```

### Step 7: Describe และ ดู Security Context

```bash
kubectl describe pod $POD -n workshop-security

# Output จะแสดง:
# Security Context:
#   Allow Privilege Escalation:  false
#   Capabilities:
#     Drop:    ALL
#   Read Only Root Filesystem:  true
#   Run As Non Root:            true
#   Run As User:                101
#   Seccomp Profile:            RuntimeDefault
```

### Step 8: Cleanup

```bash
kubectl delete namespace workshop-security
echo "Workshop cleanup complete!"
```

---

## 57.9 Security Context Checklist

### ✅ Minimum Security Baseline

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
```

### ✅ Enhanced Security

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  
  automountServiceAccountToken: false
  
  containers:
  - securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        # เพิ่มเฉพาะที่จำเป็น:
        # add: ["NET_BIND_SERVICE"]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      privileged: false
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SecurityContext**: ควบคุม privilege และ access ของ pods/containers
2. **runAsUser/Group**: รัน process ด้วย unprivileged user
3. **Capabilities**: จำกัด Linux capabilities ให้เฉพาะที่จำเป็น
4. **readOnlyRootFilesystem**: ป้องกัน filesystem modifications
5. **Seccomp/AppArmor**: จำกัด system calls

**Key Takeaways:**
- Drop all capabilities แล้วเพิ่มเฉพาะที่ต้องการ
- ใช้ readOnlyRootFilesystem เสมอ (เพิ่ม emptyDir สำหรับ writable paths)
- ห้ามรัน privileged containers
- เปิด RuntimeDefault seccomp

---

## 57.10 Security Context กับ Init Containers

### Init Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-security-demo
  namespace: default
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  
  initContainers:
  # Init container อาจต้องการ elevated privileges ชั่วคราว
  - name: init-setup
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Init complete" > /data/init.txt && chmod 644 /data/init.txt']
    securityContext:
      runAsUser: 0    # root สำหรับ init เท่านั้น
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: data
      mountPath: /data
  
  # Main container ใช้ non-root
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      runAsNonRoot: true
      runAsUser: 101
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: data
      mountPath: /data
      readOnly: true
    - name: tmp
      mountPath: /tmp
  
  volumes:
  - name: data
    emptyDir: {}
  - name: tmp
    emptyDir: {}
```

---

## 57.11 Sidecar Container Security

### Pattern: Security Proxy Sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-security
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  # Main application
  - name: app
    image: myapp:1.0.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
      runAsUser: 1000
    # App ไม่ expose ตรง แต่ผ่าน proxy sidecar
    ports: []
    volumeMounts:
    - name: shared-socket
      mountPath: /var/run/app
    - name: tmp
      mountPath: /tmp
  
  # Security proxy sidecar
  - name: envoy-proxy
    image: envoyproxy/envoy:v1.28.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
      readOnlyRootFilesystem: true
      runAsUser: 1337   # envoy user
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
      readOnly: true
    - name: tmp
      mountPath: /tmp
  
  volumes:
  - name: shared-socket
    emptyDir: {}
  - name: envoy-config
    configMap:
      name: envoy-config
  - name: tmp
    emptyDir: {}
```

---

## 57.12 Runtime Security Monitoring

### Falco: Runtime Security Detection

```yaml
# ติดตั้ง Falco สำหรับ runtime security monitoring
# helm repo add falcosecurity https://falcosecurity.github.io/charts
# helm install falco falcosecurity/falco --namespace falco --create-namespace

# Falco Rules ตัวอย่าง
- rule: Write below etc
  desc: Attempt to write below /etc
  condition: >
    open_write and evt.dir=< and fd.name startswith /etc
    and not etc_dir_open_binaries
    and not linux_bench_writing_etc
    and not write_etc_common
  output: >
    File below /etc opened for writing
    (user=%user.name user_loginuid=%user.loginuid command=%proc.cmdline
    pid=%proc.pid parent=%proc.pname pcmdline=%proc.pcmdline 
    file=%fd.name program=%proc.name gparent=%proc.aname[2])
  priority: ERROR
  tags: [filesystem, mitre_persistence]

- rule: Read sensitive file untrusted
  desc: Reads sensitive files by untrusted programs
  condition: >
    open_read and sensitive_files and proc_name_exists
    and not proc.name in (user_mgmt_binaries)
    and not proc.name in (known_binaries)
  output: >
    Sensitive file opened for reading
    (user=%user.name command=%proc.cmdline file=%fd.name)
  priority: WARNING

- rule: Container running as root
  desc: Alert when container process is running as root
  condition: >
    container.id != host
    and proc.uid = 0
    and container
    and not image_list_with_root
  output: >
    Container running as root
    (user=%user.name container=%container.name image=%container.image.repository
    cmd=%proc.cmdline)
  priority: WARNING
```

```bash
# ดู Falco alerts
kubectl logs -n falco -l app.kubernetes.io/name=falco -f

# ทดสอบ trigger alert
kubectl exec -it <pod> -- sh -c "cat /etc/shadow"
# Falco จะ alert ทันที
```

---

## 57.13 เปรียบเทียบ Security Profiles

### SecurityContext สำหรับ Use Cases ต่างกัน

```yaml
# Use Case 1: Static website
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-web
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.25
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
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
---
# Use Case 2: Node.js API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-api
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: api
        image: node:20-alpine
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: npm-cache
          mountPath: /root/.npm
      volumes:
      - name: tmp
        emptyDir: {}
      - name: npm-cache
        emptyDir: {}
---
# Use Case 3: Python ML workload (อาจต้องการ writable paths มากกว่า)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-worker
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: ml
        image: python:3.11-slim
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: model-cache
          mountPath: /tmp/models
        - name: output
          mountPath: /output
      volumes:
      - name: tmp
        emptyDir: {}
      - name: model-cache
        emptyDir:
          sizeLimit: 1Gi
      - name: output
        emptyDir:
          sizeLimit: 500Mi
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SecurityContext**: ควบคุม privilege และ access ของ pods/containers
2. **runAsUser/Group**: รัน process ด้วย unprivileged user
3. **Capabilities**: จำกัด Linux capabilities ให้เฉพาะที่จำเป็น
4. **readOnlyRootFilesystem**: ป้องกัน filesystem modifications
5. **Seccomp/AppArmor**: จำกัด system calls
6. **Init Containers**: อาจต้องการ elevated privileges ชั่วคราว
7. **Runtime Monitoring**: Falco สำหรับ detect security events

**Key Takeaways:**
- Drop all capabilities แล้วเพิ่มเฉพาะที่ต้องการ
- ใช้ readOnlyRootFilesystem เสมอ (เพิ่ม emptyDir สำหรับ writable paths)
- ห้ามรัน privileged containers
- เปิด RuntimeDefault seccomp
- ใช้ Falco สำหรับ runtime security monitoring

---

**ต่อไป**: Part 58 - Pod Security Standards
