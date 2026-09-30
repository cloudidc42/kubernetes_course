# Part 34: LoadBalancer Service

## สารบัญ
1. [LoadBalancer Service คืออะไร](#loadbalancer-overview)
2. [Cloud Load Balancer Integration](#cloud-integration)
3. [MetalLB สำหรับ On-premise](#metallb)
4. [Workshop: Deploy LoadBalancer บน Cloud](#workshop)

---

## 1. LoadBalancer Service คืออะไร {#loadbalancer-overview}

### ภาพรวม

LoadBalancer Service เป็นส่วนขยายจาก NodePort ที่ provisioning Cloud Load Balancer โดยอัตโนมัติ

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Internet Users                                              │
│       │                                                      │
│       ▼                                                      │
│  ┌────────────────┐                                          │
│  │  Cloud Load    │  ◄── External IP: 34.123.45.67         │
│  │  Balancer      │                                          │
│  └───────┬────────┘                                          │
│          │                                                    │
│          ▼                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Kubernetes Cluster                      │    │
│  │                                                       │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │    │
│  │  │  Node 1  │  │  Node 2  │  │  Node 3  │          │    │
│  │  │:30080    │  │:30080    │  │:30080    │          │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘          │    │
│  │       └─────────────┼─────────────┘                │    │
│  │                     ▼                               │    │
│  │              ┌────────────┐                         │    │
│  │              │  Service   │ ClusterIP: 10.96.1.1   │    │
│  │              └─────┬──────┘                         │    │
│  │                    │                                │    │
│  │         ┌──────────┼──────────┐                    │    │
│  │         ▼          ▼          ▼                    │    │
│  │       Pod 1      Pod 2      Pod 3                  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### สร้าง LoadBalancer Service

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  namespace: default
  annotations:
    # Cloud-specific annotations (AWS example)
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  # externalTrafficPolicy: Local  # หรือ Cluster
  loadBalancerSourceRanges:
  - "10.0.0.0/8"       # จำกัด source IPs (optional)
  - "172.16.0.0/12"
```

```bash
# Apply service
kubectl apply -f loadbalancer-service.yaml

# รอ External IP
kubectl get service web-loadbalancer -w

# ผลลัพธ์:
# NAME               TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
# web-loadbalancer   LoadBalancer   10.96.100.1    34.123.45.67    80:30080/TCP,443:30443/TCP   2m
```

---

## 2. Cloud Load Balancer Integration {#cloud-integration}

### AWS Load Balancer (AWS LBC)

```bash
# ติดตั้ง AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### AWS NLB (Network Load Balancer)

```yaml
# aws-nlb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-nlb
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
```

#### AWS ALB (Application Load Balancer) - via IngressClass

```yaml
# aws-alb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-alb
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account-id:certificate/cert-id
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      idle_timeout.timeout_seconds=60,
      routing.http2.enabled=true,
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=my-alb-logs
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### GKE (Google Kubernetes Engine) Load Balancer

```yaml
# gke-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-glb
  namespace: production
  annotations:
    # Global Load Balancer
    cloud.google.com/load-balancer-type: "External"
    networking.gke.io/load-balancer-type: "External"
    cloud.google.com/neg: '{"ingress": true}'
    # Health check path
    cloud.google.com/app-protocols: '{"http":"HTTP"}'
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
```

#### GKE BackendConfig

```yaml
# gke-backend-config.yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: my-backend-config
  namespace: production
spec:
  healthCheck:
    checkIntervalSec: 15
    port: 8080
    type: HTTP
    requestPath: /health
  timeoutSec: 30
  connectionDraining:
    drainingTimeoutSec: 60
  sessionAffinity:
    affinityType: "GENERATED_COOKIE"
    affinityCookieTtlSec: 3600
  cdn:
    enabled: true
    cachePolicy:
      includeHost: true
      includeProtocol: true
      includeQueryString: false
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  annotations:
    cloud.google.com/backend-config: '{"default": "my-backend-config"}'
spec:
  type: ClusterIP  # ใช้ร่วมกับ Ingress
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

### Azure AKS Load Balancer

```yaml
# azure-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-lb
  namespace: production
  annotations:
    # Azure specific
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "my-rg"
    service.beta.kubernetes.io/azure-pip-name: "my-public-ip"
    service.beta.kubernetes.io/azure-allowed-ip-ranges: "1.2.3.4/32,5.6.7.8/32"
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-protocol: "http"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/health"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

### DigitalOcean Load Balancer

```yaml
# do-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-do-lb
  namespace: production
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-name: "my-lb"
    service.beta.kubernetes.io/do-loadbalancer-size-slug: "lb-small"
    service.beta.kubernetes.io/do-loadbalancer-algorithm: "round_robin"
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-protocol: "http"
    service.beta.kubernetes.io/do-loadbalancer-tls-ports: "443"
    service.beta.kubernetes.io/do-loadbalancer-certificate-id: "cert-id"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8080
```

---

## 3. MetalLB สำหรับ On-premise {#metallb}

### MetalLB คืออะไร?

MetalLB คือ Load Balancer implementation สำหรับ bare metal Kubernetes Clusters ที่ไม่มี Cloud Provider

```
MetalLB Modes:
┌─────────────────────────────────────────────────────┐
│                                                      │
│  Layer 2 Mode (ARP/NDP)                             │
│  ├── ง่ายในการ setup                               │
│  ├── ไม่ต้องการ BGP                                │
│  ├── Node เดียวที่ "own" IP ในแต่ละ service        │
│  └── Failover ผ่าน ARP announcement               │
│                                                      │
│  BGP Mode                                           │
│  ├── ต้องการ BGP Router                            │
│  ├── True load balancing ระหว่าง Nodes             │
│  ├── ECMP (Equal-Cost Multi-Path)                  │
│  └── Production-grade                               │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### ติดตั้ง MetalLB

```bash
# ติดตั้ง MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# รอ Pods พร้อม
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# ดู Pods
kubectl get pods -n metallb-system
```

### Configure MetalLB Layer 2 Mode

```yaml
# metallb-config.yaml
# IP Address Pool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250  # IP range ที่จะใช้
  # หรือ CIDR notation:
  # - 192.168.1.200/29
  autoAssign: true  # assign IPs อัตโนมัติ
---
# L2 Advertisement
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
  # ระบุ node ที่จะทำหน้าที่เป็น speaker (optional)
  nodeSelectors:
  - matchLabels:
      metallb.io/speaker: "true"
```

```bash
# Apply MetalLB config
kubectl apply -f metallb-config.yaml

# สร้าง Service ที่ใช้ MetalLB
kubectl apply -f loadbalancer-service.yaml

# ดู External IP ที่ได้รับ
kubectl get service web-loadbalancer
# NAME               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
# web-loadbalancer   LoadBalancer   10.96.100.1    192.168.1.200   80:30080/TCP
```

### Configure MetalLB BGP Mode

```yaml
# metallb-bgp-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.0/24  # BGP advertised range
---
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: sample
  namespace: metallb-system
spec:
  myASN: 64512      # Cluster AS Number
  peerASN: 64513    # Router AS Number
  peerAddress: 192.168.1.1  # BGP Router IP
  keepaliveTime: 30s
  holdTime: 90s
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
  aggregationLength: 32  # Advertise /32 routes
  localPref: 100
  communities:
  - 64512:1234
```

### MetalLB กับ Multiple IP Pools

```yaml
# multiple-pools.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: public-pool
  namespace: metallb-system
spec:
  addresses:
  - 203.0.113.0/28  # Public IPs
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: private-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.10.0.0/24  # Private IPs
---
# เลือก Pool สำหรับ Service
apiVersion: v1
kind: Service
metadata:
  name: public-service
  annotations:
    metallb.io/address-pool: public-pool  # ใช้ public pool
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: internal-service
  annotations:
    metallb.io/address-pool: private-pool  # ใช้ private pool
spec:
  type: LoadBalancer
  selector:
    app: internal
  ports:
  - port: 80
    targetPort: 8080
```

---

## 4. Workshop: Deploy LoadBalancer บน Cloud {#workshop}

### Workshop 1: Setup บน Local (MetalLB + kind)

```bash
# สร้าง kind cluster
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lb-demo
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
EOF

kind create cluster --config kind-config.yaml

# ติดตั้ง MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# ดู Docker network range สำหรับ kind
docker network inspect kind | grep -A5 "Subnet"
# สมมติว่าได้ 172.18.0.0/16

# Configure MetalLB ด้วย IP range ที่อยู่ใน docker network
cat > metallb-pool.yaml << 'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - local-pool
EOF

kubectl apply -f metallb-pool.yaml
```

### Workshop 2: Deploy Application พร้อม LoadBalancer

```yaml
# app-with-lb.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "<html><body>
          <h1>Demo App</h1>
          <p>Pod: $(hostname)</p>
          <p>Node: $NODE_NAME</p>
          </body></html>" > /html/index.html
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-lb
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: demo-app
  ports:
  - name: http
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f app-with-lb.yaml

# รอ External IP
kubectl get service demo-app-lb -w
# รอจน EXTERNAL-IP ไม่เป็น <pending>

EXTERNAL_IP=$(kubectl get service demo-app-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "External IP: $EXTERNAL_IP"

# ทดสอบ
curl http://$EXTERNAL_IP

# ทดสอบ Load Balancing
for i in {1..10}; do curl -s http://$EXTERNAL_IP | grep "Pod:"; done
```

### Workshop 3: Health Check Configuration

```yaml
# lb-with-health-check.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: health-app
  template:
    metadata:
      labels:
        app: health-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: health-app-lb
  annotations:
    # AWS: กำหนด health check
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-unhealthy-threshold: "3"
spec:
  type: LoadBalancer
  selector:
    app: health-app
  ports:
  - port: 80
    targetPort: 80
```

### Workshop 4: Static IP Assignment

```yaml
# static-ip-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: static-ip-service
  annotations:
    # MetalLB: กำหนด IP เฉพาะ
    metallb.io/loadBalancerIPs: "172.18.255.100"
    
    # AWS: ใช้ Elastic IP
    # service.beta.kubernetes.io/aws-load-balancer-eip-allocations: "eipalloc-xxx,eipalloc-yyy"
    
    # GCP: ใช้ Static IP
    # cloud.google.com/load-balancer-type: "External"
    # kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
    
    # Azure: ใช้ Public IP
    # service.beta.kubernetes.io/azure-pip-name: "myStaticIP"
spec:
  type: LoadBalancer
  loadBalancerIP: "172.18.255.100"  # deprecated แต่ยังใช้ได้
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 80
```

### Workshop 5: Internal Load Balancer

```yaml
# internal-lb.yaml
# สำหรับ traffic ภายใน VPC/Network เท่านั้น
apiVersion: v1
kind: Service
metadata:
  name: internal-lb-service
  annotations:
    # AWS: Internal NLB
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    
    # GKE: Internal Load Balancer
    # cloud.google.com/load-balancer-type: "Internal"
    # networking.gke.io/internal-load-balancer-allow-global-access: "true"
    
    # Azure: Internal Load Balancer
    # service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    # service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "my-subnet"
    
    # MetalLB: ใช้ Private Pool
    metallb.io/address-pool: private-pool
spec:
  type: LoadBalancer
  selector:
    app: internal-app
  ports:
  - port: 80
    targetPort: 8080
```

### Workshop 6: ทดสอบ Failover

```bash
# ดู Pods
kubectl get pods -o wide

# จำลอง Node failure (drain node)
NODE_NAME=$(kubectl get nodes -l node-role.kubernetes.io/worker='' -o name | head -1 | cut -d/ -f2)
kubectl cordon $NODE_NAME
kubectl drain $NODE_NAME --ignore-daemonsets --delete-emptydir-data

# ทดสอบว่า LoadBalancer ยังทำงาน
EXTERNAL_IP=$(kubectl get service demo-app-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
for i in {1..10}; do curl -s http://$EXTERNAL_IP; echo; done

# Restore node
kubectl uncordon $NODE_NAME
```

### Workshop 7: LoadBalancer Status Monitoring

```bash
# Monitor LoadBalancer status
kubectl get service -w

# ดู Events
kubectl describe service demo-app-lb

# ดู Cloud Provider events
kubectl get events | grep LoadBalancer

# ดู MetalLB logs
kubectl logs -n metallb-system -l app=metallb,component=controller
kubectl logs -n metallb-system -l app=metallb,component=speaker

# ดู MetalLB address assignments
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```

### Workshop 8: LoadBalancer กับ TLS Termination

```yaml
# tls-loadbalancer.yaml
# AWS Certificate Manager + NLB
apiVersion: v1
kind: Service
metadata:
  name: tls-lb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:region:account:certificate/cert-id"
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS13-1-2-2021-06"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8080  # TLS terminated at LB, forward HTTP to pods
```

### Cleanup

```bash
kubectl delete deployment demo-app health-app
kubectl delete service demo-app-lb health-app-lb static-ip-service internal-lb-service
kind delete cluster --name lb-demo
```

---

## เปรียบเทียบ Service Types

```
Service Comparison:
┌──────────────────────────────────────────────────────────┐
│ Feature         ClusterIP  NodePort   LoadBalancer       │
├──────────────────────────────────────────────────────────┤
│ Internal Access │   ✅      ✅         ✅               │
│ External Access │   ❌      ✅         ✅               │
│ Cloud LB        │   ❌      ❌         ✅ Auto          │
│ Static IP       │   ❌      ❌         ✅ (with config) │
│ SSL Termination │   ❌      ❌         ✅ (cloud)       │
│ L7 Routing      │   ❌      ❌         ❌ (need Ingress)│
│ Cost            │   Free   Free       💰 (cloud LB)    │
│ Port            │   Any    30000-32767 80/443 (typical) │
│ On-premise      │   ✅      ✅         ✅ (MetalLB)     │
└──────────────────────────────────────────────────────────┘
```

---

## Cheat Sheet

```bash
# สร้าง LoadBalancer Service
kubectl expose deployment my-app --type=LoadBalancer --port=80 --target-port=8080

# ดู External IP
kubectl get service my-service
kubectl get service my-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# รอ External IP
kubectl get service my-service -w --timeout=5m

# MetalLB commands
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
kubectl get bgppeers -n metallb-system
kubectl logs -n metallb-system -l component=controller -f

# ดู Cloud LB annotations
kubectl get service my-service -o yaml | grep annotation
```

---

*ก่อนหน้า: [Part 33 - NodePort Service](./part-33-nodeport.md)*
*ต่อไป: [Part 35 - Ingress Controllers](./part-35-ingress-controllers.md)*
