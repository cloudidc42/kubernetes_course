# Part 101: CKAD (Certified Kubernetes Application Developer) Exam Guide

## บทนำ

Certified Kubernetes Application Developer (CKAD) มุ่งเน้นทักษะการพัฒนาและ Deploy Applications บน Kubernetes บทนี้ครอบคลุมทุก Exam Domain พร้อม Practice Questions กว่า 200 ข้อ

## ข้อมูล Exam

```
Exam Format: Performance-based (Hands-on)
Duration: 2 hours
Pass Score: 66%
Cost: $395 USD
Validity: 3 years
Number of Questions: 15-20 tasks

Exam Domains (v1.28):
- Application Design and Build: 20%
- Application Deployment: 20%
- Application Observability and Maintenance: 15%
- Application Environment, Configuration and Security: 25%
- Services and Networking: 20%
```

## สารบัญ

1. Application Design and Build (20%)
2. Application Deployment (20%)
3. Application Observability and Maintenance (15%)
4. Application Environment, Configuration and Security (25%)
5. Services and Networking (20%)
6. Exam Tips
7. Practice Labs (200+ Questions)

---

## 1. Application Design and Build (20%)

### 1.1 Container Images

```dockerfile
# Multi-stage Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

```bash
# Build และ Tag Image
docker build -t myapp:1.0 .
docker tag myapp:1.0 registry.io/myapp:1.0
docker push registry.io/myapp:1.0

# ดู Image Layers
docker history myapp:1.0
```

### 1.2 Jobs และ CronJobs

```yaml
# Job - one-time task
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 5      # ต้องสำเร็จ 5 ครั้ง
  parallelism: 2      # รันพร้อมกัน 2 pod
  backoffLimit: 3     # retry ไม่เกิน 3 ครั้ง
  activeDeadlineSeconds: 100
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: processor
        image: busybox
        command: ['sh', '-c', 'echo Processing done']
---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # ทุกคืน 2am
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ['./backup.sh']
```

### 1.3 Multi-Container Pod Patterns

```yaml
# Ambassador Pattern
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-example
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_HOST
      value: "localhost"  # เชื่อมต่อ ambassador
  - name: ambassador
    image: db-proxy:latest
    env:
    - name: DB_HOST
      value: "prod-db.example.com"
---
# Adapter Pattern
apiVersion: v1
kind: Pod
metadata:
  name: adapter-example
spec:
  containers:
  - name: legacy-app
    image: legacy:1.0
    # Output format เก่า
  - name: log-adapter
    image: log-transformer:latest
    # แปลง format เป็น JSON
    volumeMounts:
    - name: logs
      mountPath: /logs
  volumes:
  - name: logs
    emptyDir: {}
```

---

## 2. Application Deployment (20%)

### 2.1 Deployment Strategies

```yaml
# Rolling Update (Default)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # เพิ่มได้สูงสุด 1 pod ระหว่าง update
      maxUnavailable: 0 # ห้ามมี downtime
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.25
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Recreate Strategy
spec:
  strategy:
    type: Recreate
```

```bash
# Blue-Green Deployment
# Blue Environment
kubectl create deployment webapp-blue --image=myapp:1.0
kubectl expose deployment webapp-blue --port=80 --name=webapp-blue

# Green Environment
kubectl create deployment webapp-green --image=myapp:2.0
kubectl expose deployment webapp-green --port=80 --name=webapp-green

# Switch Traffic
kubectl patch svc webapp -p '{"spec":{"selector":{"version":"green"}}}'
```

### 2.2 Helm Charts

```bash
# ติดตั้ง Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# เพิ่ม Repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# ค้นหา Chart
helm search repo mysql

# ติดตั้ง Chart
helm install my-mysql bitnami/mysql \
  --set auth.rootPassword=secretpassword \
  --set primary.persistence.size=10Gi

# ดู Release ที่ติดตั้ง
helm list
helm status my-mysql

# Upgrade
helm upgrade my-mysql bitnami/mysql --set auth.rootPassword=newpassword

# Rollback
helm rollback my-mysql 1

# ถอนการติดตั้ง
helm uninstall my-mysql
```

---

## 3. Application Observability and Maintenance (15%)

### 3.1 Liveness, Readiness, Startup Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-demo
spec:
  containers:
  - name: app
    image: nginx
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
      successThreshold: 1
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /startup
        port: 80
      failureThreshold: 30
      periodSeconds: 10
```

### 3.2 Logging

```bash
# ดู Logs
kubectl logs pod-name
kubectl logs pod-name -c container-name  # Multi-container pod
kubectl logs pod-name --previous          # Previous container
kubectl logs pod-name -f --tail=100       # Follow, last 100 lines

# ดู Logs หลาย Pods พร้อมกัน
kubectl logs -l app=nginx --all-containers=true

# Structured Logging - ตัวอย่าง App
# app.log
{
  "timestamp": "2025-01-01T10:00:00Z",
  "level": "info",
  "message": "Request processed",
  "request_id": "abc123",
  "duration_ms": 45
}
```

### 3.3 Debugging

```bash
# Ephemeral Debug Container (K8s 1.23+)
kubectl debug pod-name -it \
  --image=busybox \
  --target=app \
  -- sh

# Debug Crashed Pod (ไม่มี shell)
kubectl debug pod-name -it \
  --copy-to=debug-pod \
  --image=busybox \
  -- sh

# Debug Node
kubectl debug node/worker-1 \
  -it \
  --image=busybox
```

---

## 4. Application Environment, Configuration and Security (25%)

### 4.1 ConfigMaps

```bash
# สร้าง ConfigMap
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PORT=3306

kubectl create configmap nginx-config \
  --from-file=nginx.conf

kubectl create configmap env-config \
  --from-env-file=.env

# ดู ConfigMap
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

```yaml
# ใช้ ConfigMap ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: app
    image: myapp
    # Environment Variables จาก ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
    # Environment Variable เฉพาะ key
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    # Mount เป็น Volume
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: nginx-config
```

### 4.2 Secrets

```bash
# สร้าง Secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# Docker Registry Secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com

# TLS Secret
kubectl create secret tls my-tls \
  --cert=server.crt \
  --key=server.key
```

```yaml
# ใช้ Secret ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
  imagePullSecrets:
  - name: regcred
```

### 4.3 SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### 4.4 Resource Quotas และ LimitRange

```yaml
# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
---
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: "2"
      memory: 2Gi
    min:
      cpu: 100m
      memory: 64Mi
```

### 4.5 ServiceAccount

```bash
# สร้าง ServiceAccount
kubectl create serviceaccount my-app-sa -n default

# Bind Role
kubectl create role pod-reader \
  --verb=get,list \
  --resource=pods

kubectl create rolebinding my-app-pod-reader \
  --role=pod-reader \
  --serviceaccount=default:my-app-sa
```

```yaml
# ใช้ ServiceAccount ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: true
  containers:
  - name: app
    image: myapp
```

---

## 5. Services and Networking (20%)

### 5.1 Services

```yaml
# ClusterIP (Default)
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
---
# NodePort
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767
---
# Headless Service (StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: my-headless
spec:
  clusterIP: None  # Headless!
  selector:
    app: statefulapp
  ports:
  - port: 80
```

### 5.2 Ingress

```yaml
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
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
      - path: /api/v[0-9]+/
        pathType: ImplementationSpecific
        backend:
          service:
            name: api
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

### 5.3 Network Policies

```yaml
# Allow Specific Egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-db
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53  # DNS
```

---

## 6. Exam Tips

```
CKAD Quick Tips:
1. สร้าง YAML Templates ด้วย --dry-run=client -o yaml
2. ใช้ kubectl explain สำหรับ API Reference
3. ฝึก Port-forward สำหรับ Debug
4. จำ readinessProbe vs livenessProbe ต่างกันอย่างไร
5. เข้าใจ Rolling Update parameters
6. รู้จัก Helm commands

Important Shortcuts:
alias k=kubectl
export do="--dry-run=client -o yaml"
k run nginx --image=nginx $do > pod.yaml
```

---

## 7. Practice Labs (200+ Questions)

### Section A: Pods & Containers

**Q1: สร้าง Pod ชื่อ web-pod ด้วย image nginx และ label app=web**
```bash
kubectl run web-pod --image=nginx -l app=web
```

**Q2: สร้าง Pod ที่รัน Command แทน Default**
```bash
kubectl run cmd-pod \
  --image=busybox \
  --command -- /bin/sh -c "while true; do echo hello; sleep 10; done"
```

**Q3: สร้าง Pod YAML Template**
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

**Q4: สร้าง Pod ที่มี Resource Requests/Limits**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

**Q5: สร้าง Pod ที่อ่าน Environment Variables จาก ConfigMap**
```bash
# สร้าง ConfigMap
kubectl create configmap myconfig --from-literal=DB_HOST=mysql --from-literal=DB_PORT=3306

# Pod ที่ใช้ ConfigMap
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: nginx
    envFrom:
    - configMapRef:
        name: myconfig
EOF
```

**Q6: สร้าง Multi-Container Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  - name: logger
    image: busybox
    command: ['sh', '-c', 'while true; do date; sleep 5; done']
```

**Q7: สร้าง Pod ที่ Mount Volume ระหว่าง Containers**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'while true; do echo $(date) > /shared/time.txt; sleep 5; done']
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: reader
    image: busybox
    command: ['sh', '-c', 'while true; do cat /shared/time.txt; sleep 5; done']
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
```

**Q8: สร้าง Init Container**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-service
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do sleep 2; done']
  containers:
  - name: app
    image: nginx
```

**Q9: ดู Logs เฉพาะ Container ใน Multi-Container Pod**
```bash
kubectl logs multi-pod -c logger
```

**Q10: Exec Command ใน Container ที่เฉพาะเจาะจง**
```bash
kubectl exec -it multi-pod -c nginx -- nginx -t
```

### Section B: Deployments & Scaling

**Q11: สร้าง Deployment พร้อม Rolling Update Config**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: rolling
  template:
    metadata:
      labels:
        app: rolling
    spec:
      containers:
      - name: app
        image: nginx:1.25
```

**Q12: ทำ Canary Deployment**
```bash
# Stable version (90% traffic)
kubectl create deployment stable --image=myapp:1.0 --replicas=9

# Canary version (10% traffic)
kubectl create deployment canary --image=myapp:2.0 --replicas=1

# Service ที่ Select ทั้งสอง
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # ต้องมี label นี้บน Pods ทั้งสอง
  ports:
  - port: 80
EOF
```

**Q13: Pause Deployment และ Update หลายรายการ**
```bash
kubectl rollout pause deployment/webapp
kubectl set image deployment/webapp nginx=nginx:1.26
kubectl set env deployment/webapp ENV=production
kubectl rollout resume deployment/webapp
```

**Q14: สร้าง HPA**
```bash
kubectl autoscale deployment webapp \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

**Q15: ดู Rollout History**
```bash
kubectl rollout history deployment/webapp
kubectl rollout history deployment/webapp --revision=3
```

### Section C: Jobs & CronJobs

**Q16: สร้าง Job ที่ต้องทำให้สำเร็จ 3 ครั้ง**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-job
spec:
  completions: 3
  parallelism: 1
  backoffLimit: 6
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo Task $JOB_COMPLETION_INDEX done']
```

**Q17: สร้าง CronJob ทุก 5 นาที**
```bash
kubectl create cronjob heartbeat \
  --image=busybox \
  --schedule="*/5 * * * *" \
  -- /bin/sh -c "echo Heartbeat"
```

**Q18: ตรวจสอบ Job Status**
```bash
kubectl get jobs
kubectl describe job multi-job
kubectl logs -l job-name=multi-job
```

**Q19: Manually รัน CronJob ทันที**
```bash
kubectl create job --from=cronjob/heartbeat manual-run
```

### Section D: Configuration & Security

**Q20: สร้าง Pod ที่รันด้วย User ที่ไม่ใช่ Root**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-root-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
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

**Q21: สร้าง Secret จาก File**
```bash
echo -n "secretpassword" > password.txt
kubectl create secret generic file-secret --from-file=password=password.txt
```

**Q22: Mount Secret เป็น Environment Variable**
```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: file-secret
          key: password
```

**Q23: สร้าง ResourceQuota สำหรับ Namespace**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
EOF
```

**Q24: ตรวจสอบ Resource Quota ใช้ไปเท่าไหร่**
```bash
kubectl describe resourcequota team-quota -n team-a
```

**Q25: สร้าง LimitRange**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 200m
      memory: 256Mi
    max:
      cpu: "1"
      memory: 1Gi
```

### Section E: Observability

**Q26: เพิ่ม Liveness Probe ให้ Pod**
```yaml
spec:
  containers:
  - name: app
    image: nginx
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
```

**Q27: เพิ่ม Readiness Probe**
```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/ready
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Q28: สร้าง Pod ที่มี Startup Probe**
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30  # 30 * 10s = 5 minutes
  periodSeconds: 10
```

**Q29: Debug Pod ที่มี OOMKilled**
```bash
kubectl describe pod oom-pod | grep -A 10 "Last State"
kubectl get pod oom-pod -o json | jq '.status.containerStatuses[].lastState'

# แก้ไข: เพิ่ม Memory Limit
kubectl set resources deployment/webapp --limits=memory=512Mi
```

**Q30: ดู Top Pods และ Top Nodes**
```bash
kubectl top pods -A --sort-by=memory
kubectl top nodes
```

### Section F: Services & Networking

**Q31: สร้าง Service ExternalName**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.prod.example.com
```

**Q32: ทดสอบ DNS จากใน Pod**
```bash
kubectl run dns-test \
  --image=busybox \
  --rm \
  -it \
  -- nslookup my-service.default.svc.cluster.local
```

**Q33: สร้าง NetworkPolicy ที่ Allow Ingress จาก Namespace เฉพาะ**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: production
```

**Q34: ทำ Port Forward ไปยัง Service**
```bash
kubectl port-forward svc/my-service 8080:80
```

**Q35: สร้าง Service สำหรับ StatefulSet**
```yaml
# Headless Service สำหรับ StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306

# ใช้ DNS: mysql-0.mysql.default.svc.cluster.local
```

### Section G: Storage

**Q36: สร้าง Pod ที่ใช้ emptyDir**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
```

**Q37: สร้าง PVC และ Pod ที่ใช้ PVC**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-pvc
```

**Q38: อ่านและเขียนไฟล์ใน PVC**
```bash
# เขียนไฟล์
kubectl exec pvc-pod -- sh -c "echo 'test data' > /data/test.txt"

# อ่านไฟล์
kubectl exec pvc-pod -- cat /data/test.txt
```

**Q39: ตรวจสอบว่า PVC Bound แล้วหรือยัง**
```bash
kubectl get pvc app-pvc
# STATUS ต้องเป็น Bound
```

**Q40: Mount ConfigMap เป็น File ใน Pod**
```bash
# สร้าง ConfigMap จาก nginx config
cat <<'EOF' > nginx.conf
server {
  listen 80;
  location / {
    return 200 'Hello World!';
  }
}
EOF

kubectl create configmap nginx-config --from-file=nginx.conf

# Mount ใน Pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config
    configMap:
      name: nginx-config
EOF
```

### Section H: Advanced Topics

**Q41: สร้าง StatefulSet ด้วย Stable Network Identity**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

**Q42: Update StatefulSet Image**
```bash
kubectl set image statefulset/web nginx=nginx:1.26
kubectl rollout status statefulset/web
```

**Q43: สร้าง Pod ที่มี Node Affinity**
```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values: ["ssd"]
```

**Q44: ใช้ Projected Volume**
```yaml
spec:
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: my-secret
      - configMap:
          name: my-config
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
```

**Q45: สร้าง Pod ที่ใช้ hostPath Volume**
```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-logs
      mountPath: /var/log/app
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log/myapp
      type: DirectoryOrCreate
```

---

## สรุป CKAD Exam

```
Key Skills ที่ต้องมี:
1. สร้าง Pod/Deployment/Service ด้วย Imperative Commands
2. เข้าใจ ConfigMap, Secret, SecurityContext
3. Debug Pod Issues
4. เข้าใจ Networking/Ingress/NetworkPolicy
5. ใช้งาน PVC/PV
6. ทำ Rolling Update และ Rollback
7. สร้าง Job/CronJob

Practice Schedule:
Week 1: Pod Design, Multi-container, Configuration
Week 2: Deployments, Services, Networking
Week 3: Observability, Security
Week 4: Mock Exams (KillerCoda, Udemy)
```

## References

- [CKAD Official](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/)
- [CKAD Exercises GitHub](https://github.com/dgkanatsios/CKAD-exercises)
- [KillerCoda CKAD](https://killercoda.com/killer-shell-ckad)
