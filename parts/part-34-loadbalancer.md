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

---

## 6. MetalLB Layer2 vs BGP Mode {#metallb-deep}

### 6.1 MetalLB Overview

MetalLB ทำให้ bare-metal cluster มี LoadBalancer Service ได้ โดยใช้ ARP (Layer2) หรือ BGP protocols

```
┌─────────────────────────────────────────────────────────────────┐
│                    MetalLB Architecture                          │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  MetalLB Components                       │   │
│  │                                                           │   │
│  │  Controller (Deployment) - IP allocation                  │   │
│  │  Speaker (DaemonSet) - Network advertisement              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Layer2 Mode:                    BGP Mode:                       │
│  ┌─────────────────────┐        ┌─────────────────────────┐     │
│  │ Speaker AdvertisesARP│        │ Speaker peers with router│     │
│  │ for assigned IP     │        │ via BGP protocol        │     │
│  │ → 1 node owns IP   │        │ → Multiple nodes own IP │     │
│  │ (Active/Passive)    │        │ (ECMP load balancing)   │     │
│  └─────────────────────┘        └─────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 ติดตั้ง MetalLB

```bash
# ติดตั้ง MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# รอ MetalLB ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# ตรวจสอบ
kubectl get pods -n metallb-system
kubectl get daemonset -n metallb-system
```

### 6.3 MetalLB Layer2 Mode - การตั้งค่าครบถ้วน

```yaml
# metallb-layer2-config.yaml

# 1. IP Address Pool - กำหนด IP range
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.100-192.168.10.200   # IP range
  - 192.168.20.0/24                  # CIDR notation
  autoAssign: true                    # Auto-assign จาก pool นี้
  avoidBuggyIPs: true                 # Skip .0 and .255

---
# IP Pool สำหรับ internal services
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: internal-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.100.0/24
  autoAssign: false    # ต้อง annotate service เพื่อใช้ pool นี้

---
# 2. L2Advertisement - Advertise IPs ผ่าน ARP/NDP
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool     # Advertise IPs จาก pool นี้
  nodeSelectors:        # เฉพาะ nodes เหล่านี้
  - matchLabels:
      metallb-speaker: "true"
  interfaces:           # เฉพาะ network interface นี้
  - eth0

---
# L2Advertisement สำหรับ internal pool
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert-internal
  namespace: metallb-system
spec:
  ipAddressPools:
  - internal-pool
  interfaces:
  - eth1    # Internal network interface
```

#### ใช้งาน Layer2 MetalLB

```yaml
# service-layer2.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  annotations:
    # เลือก IP Pool เฉพาะ
    metallb.universe.tf/address-pool: production-pool
    # หรือ specify IP เลย
    # metallb.universe.tf/loadBalancerIPs: 192.168.10.105
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443

---
# Internal service
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  namespace: production
  annotations:
    metallb.universe.tf/address-pool: internal-pool
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
# ดู IP ที่ได้รับ
kubectl get service web-service -n production
# NAME          TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# web-service   LoadBalancer   10.96.100.1    192.168.10.100   80:30080/TCP

# ดู MetalLB events
kubectl describe service web-service -n production

# ดู MetalLB IP allocations
kubectl get ipaddresspool -n metallb-system -o wide
kubectl get l2advertisements -n metallb-system

# Debug Layer2
kubectl logs -n metallb-system -l component=speaker --tail=50
kubectl logs -n metallb-system -l component=controller --tail=50

# ตรวจสอบ ARP (จาก network ภายนอก)
arp -a | grep 192.168.10.100
# หรือ
arping -I eth0 192.168.10.100
```

### 6.4 MetalLB BGP Mode - การตั้งค่าครบถ้วน

```yaml
# metallb-bgp-config.yaml

# 1. IP Pool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
  - 203.0.113.0/28    # Public IP range (14 IPs)
  autoAssign: true

---
# 2. BGP Peer - กำหนด router ที่จะ peer ด้วย
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router-01
  namespace: metallb-system
spec:
  peerAddress: 10.0.0.254     # Router IP
  peerASN: 65000              # Router AS Number
  myASN: 65001                # Cluster AS Number
  routerID: 10.0.0.1          # Router ID (usually node IP)
  holdTime: 90s               # BGP session timeout
  keepaliveTime: 30s
  password: "bgp-secret"      # Optional MD5 password
  # กำหนดว่า Node ไหนจะ peer กับ router นี้
  nodeSelectors:
  - matchLabels:
      kubernetes.io/os: linux

---
# BGP Peer สำรอง (สำหรับ HA)
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router-02
  namespace: metallb-system
spec:
  peerAddress: 10.0.0.253
  peerASN: 65000
  myASN: 65001
  routerID: 10.0.0.2

---
# 3. BGP Advertisement
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
  peers:
  - router-01
  - router-02
  aggregationLength: 32        # Advertise /32 per IP
  aggregationLengthV6: 128
  localPref: 100               # BGP Local Preference
  communities:
  - 65535:65281               # BGP Community (no-export)

---
# BGP Advertisement พร้อม ECMP
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-ecmp-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
  peers:
  - router-01
  - router-02
  # ทุก node advertise IP เดียวกัน → Router ทำ ECMP load balancing
```

#### ตรวจสอบ BGP Mode

```bash
# ดู BGP peer status
kubectl get bgppeers -n metallb-system
kubectl describe bgppeer router-01 -n metallb-system

# ดู BGP session logs
kubectl logs -n metallb-system -l component=speaker --tail=100 | grep -i bgp

# ตรวจสอบ BGP routes บน Router (Cisco/Juniper/VyOS)
# show ip bgp summary
# show ip bgp neighbors 10.0.0.1 routes
# show ip route 203.0.113.0/28

# ดู speaker events
kubectl events -n metallb-system --field-selector reason=sessionEstablished

# Test ECMP (BGP mode)
for i in $(seq 1 5); do
  traceroute 203.0.113.1 | head -5
done
# ควรเห็น different paths
```

### 6.5 MetalLB Troubleshooting

```bash
# ปัญหา 1: Service stuck ที่ <pending>
kubectl describe service my-service | grep Events
# หา error message

# ตรวจสอบว่า pool มี IP ว่าง
kubectl get ipaddresspool -n metallb-system
# ถ้า pool เต็ม → เพิ่ม range

# ปัญหา 2: IP ได้รับแล้วแต่ ping ไม่ติด (Layer2)
# ตรวจสอบว่า speaker running บน node ที่ถูกต้อง
kubectl get pods -n metallb-system -o wide

# ตรวจสอบ ARP table บน switch/router
arp -a | grep 192.168.10.100

# Gratuitous ARP - speaker ส่งแล้วหรือยัง?
kubectl logs -n metallb-system \
  $(kubectl get pod -n metallb-system -l component=speaker -o jsonpath='{.items[0].metadata.name}') \
  | grep -i arp

# ปัญหา 3: BGP session ไม่ขึ้น
kubectl logs -n metallb-system -l component=speaker | grep -E "BGP|peer|session"

# ตรวจสอบ firewall - BGP ใช้ TCP port 179
# iptables -A INPUT -p tcp --dport 179 -j ACCEPT

# ปัญหา 4: Failover ช้า (Layer2)
# Layer2 failover ขึ้นอยู่กับ ARP cache timeout (~30 seconds)
# แก้ด้วยการ tune ARP cache
sysctl -w net.ipv4.neigh.default.gc_staletime=10
```

---

## 7. AWS NLB vs ALB {#aws-nlb-vs-alb}

### 7.1 ความแตกต่างระหว่าง NLB และ ALB

```
┌──────────────────────────────────────────────────────────────┐
│                   NLB vs ALB Comparison                       │
│                                                               │
│  Feature              NLB                    ALB              │
│  ─────────────────────────────────────────────────────────── │
│  OSI Layer            L4 (TCP/UDP)           L7 (HTTP/HTTPS) │
│  Protocol             TCP, UDP, TLS          HTTP, HTTPS, gRPC│
│  Latency              Ultra-low (~100μs)     Low (~400μs)     │
│  Static IP            Yes                    No (DNS only)    │
│  Preserve Client IP   Yes (by default)       Via X-Forwarded │
│  Path-based routing   No                     Yes              │
│  Host-based routing   No                     Yes              │
│  WebSocket            Yes                    Yes              │
│  gRPC                 Yes                    Yes              │
│  Price                Per LCU                Per LCU          │
│  Use case             High-perf, non-HTTP    HTTP apps        │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 AWS NLB Configuration

```yaml
# nlb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nlb-service
  namespace: production
  annotations:
    # NLB specific
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    
    # Internal NLB
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    
    # Cross-zone load balancing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    
    # IP preservation
    service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: "preserve_client_ip.enabled=true"
    
    # Health check
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
    
    # Subnets
    service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-xxxx,subnet-yyyy"
    
    # Security groups (NLB supports SG from 2023)
    service.beta.kubernetes.io/aws-load-balancer-security-groups: "sg-xxxx"
    
    # EIP allocation (Static IP)
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: "eipalloc-xxxx,eipalloc-yyyy"
    
    # Connection draining
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: "60"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  externalTrafficPolicy: Local    # สำคัญสำหรับ NLB IP preservation
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  - name: grpc
    port: 9090
    targetPort: 9090
    protocol: TCP
```

### 7.3 AWS ALB Configuration (AWS Load Balancer Controller)

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

```yaml
# alb-ingress.yaml - ALB ใช้ผ่าน Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alb-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    
    # SSL
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-1:XXXX:certificate/YYYY"
    alb.ingress.kubernetes.io/ssl-policy: "ELBSecurityPolicy-TLS13-1-2-2021-06"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    
    # WAF
    alb.ingress.kubernetes.io/wafv2-acl-arn: "arn:aws:wafv2:us-east-1:XXXX:regional/webacl/YYYY"
    
    # Subnets
    alb.ingress.kubernetes.io/subnets: "subnet-xxxx,subnet-yyyy"
    
    # Security groups
    alb.ingress.kubernetes.io/security-groups: "sg-xxxx"
    
    # Target group attributes
    alb.ingress.kubernetes.io/target-group-attributes: |
      deregistration_delay.timeout_seconds=30,
      slow_start.duration_seconds=30,
      stickiness.enabled=true,
      stickiness.lb_cookie.duration_seconds=86400
    
    # Health check
    alb.ingress.kubernetes.io/healthcheck-path: "/health"
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/success-codes: "200,201"
    
    # HTTP to HTTPS redirect
    alb.ingress.kubernetes.io/actions.ssl-redirect: |
      {"type":"redirect","redirectConfig":{"protocol":"HTTPS","statusCode":"HTTP_301"}}
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 8080
      - path: /v1/orders
        pathType: Prefix
        backend:
          service:
            name: orders-service
            port:
              number: 8080
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## 8. LoadBalancer Annotations ต่าง Cloud {#cloud-annotations}

### 8.1 Google Cloud (GKE) Annotations

```yaml
# gke-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: gke-lb-service
  namespace: production
  annotations:
    # Internal Load Balancer
    networking.gke.io/load-balancer-type: "Internal"
    
    # Static IP
    kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
    
    # Backend config
    cloud.google.com/backend-config: '{"default":"my-backend-config"}'
    
    # NEG (Network Endpoint Group)
    cloud.google.com/neg: '{"ingress":true,"exposed_ports":{"80":{}}}'
    
    # Load Balancer class
    networking.gke.io/load-balancer-type: "External"
    
    # Subnet for ILB
    networking.gke.io/internal-load-balancer-subnet: "my-subnet"
    
    # Allow global access (ILB)
    networking.gke.io/internal-load-balancer-allow-global-access: "true"
    
    # Health check
    networking.gke.io/app-protocols: '{"http":"HTTP","https":"HTTPS"}'
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080

---
# BackendConfig สำหรับ GKE
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: my-backend-config
  namespace: production
spec:
  sessionAffinity:
    affinityType: "GENERATED_COOKIE"
    affinityCookieTtlSec: 50
  connectionDraining:
    drainingTimeoutSec: 60
  cdn:
    enabled: true
    cachePolicy:
      includeHost: true
      includeProtocol: true
  healthCheck:
    checkIntervalSec: 15
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 2
    type: HTTP
    requestPath: /health
  iap:
    enabled: false
```

### 8.2 Azure (AKS) Annotations

```yaml
# aks-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: aks-lb-service
  namespace: production
  annotations:
    # Internal Load Balancer
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "internal-subnet"
    
    # Static IP
    service.beta.kubernetes.io/azure-load-balancer-ipv4: "10.0.0.100"
    
    # DNS label
    service.beta.kubernetes.io/azure-dns-label-name: "myapp"
    
    # Resource group
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "my-rg"
    
    # Health probe
    service.beta.kubernetes.io/azure-load-balancer-health-probe-interval: "5"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-num-of-probe: "3"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/health"
    
    # Idle timeout
    service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout: "4"
    
    # Disable outbound SNAT
    service.beta.kubernetes.io/azure-disable-load-balancer-floating-ip: "false"
    
    # PIP (Public IP) prefix
    service.beta.kubernetes.io/azure-pip-prefix-id: "/subscriptions/XXXX/resourceGroups/my-rg/providers/Microsoft.Network/publicIPPrefixes/my-prefix"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

### 8.3 Comparison Table

```yaml
# ตาราง annotations เปรียบเทียบ

# Internal LB:
# AWS:    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
# GKE:    networking.gke.io/load-balancer-type: "Internal"
# AKS:    service.beta.kubernetes.io/azure-load-balancer-internal: "true"

# Static IP:
# AWS:    service.beta.kubernetes.io/aws-load-balancer-eip-allocations
# GKE:    kubernetes.io/ingress.global-static-ip-name
# AKS:    service.beta.kubernetes.io/azure-load-balancer-ipv4

# Health Check Path:
# AWS:    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path
# GKE:    ใช้ BackendConfig CRD
# AKS:    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path
```

---

## 9. Internal vs External LoadBalancer {#internal-vs-external}

### 9.1 External LoadBalancer

```
External LoadBalancer:
┌─────────────────────────────────────────────────────────┐
│  Internet                                                │
│      │                                                   │
│      ▼                                                   │
│  ┌─────────────────────────────────────────┐            │
│  │  External Load Balancer                  │            │
│  │  IP: 203.0.113.10 (Public IP)            │            │
│  │                                           │            │
│  │  - Accessible from internet               │            │
│  │  - Usually needs SSL/TLS                  │            │
│  │  - WAF, DDoS protection                   │            │
│  └─────────────────────────────────────────┘            │
│                      │                                   │
│                      ▼                                   │
│              Kubernetes Cluster                          │
│                  (Private)                               │
└─────────────────────────────────────────────────────────┘
```

```yaml
# external-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-web
  namespace: production
spec:
  type: LoadBalancer
  # ไม่มี annotation internal → External by default
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  externalTrafficPolicy: Cluster
  # ระบุ External IP ที่ต้องการ (cloud-specific)
  loadBalancerIP: "203.0.113.10"
```

### 9.2 Internal LoadBalancer

```
Internal LoadBalancer:
┌─────────────────────────────────────────────────────────┐
│  VPC / Private Network                                   │
│                                                          │
│  ┌─────────────┐        ┌──────────────────────────┐   │
│  │  On-prem    │───────►│  Internal Load Balancer  │   │
│  │  Network    │        │  IP: 10.0.0.100 (Private)│   │
│  └─────────────┘        └──────────────────────────┘   │
│                                       │                  │
│  ┌─────────────┐                      ▼                  │
│  │  Other      │              Kubernetes Cluster         │
│  │  Services   │───────────────────────────────────────► │
│  └─────────────┘                                         │
└─────────────────────────────────────────────────────────┘
```

```yaml
# internal-lb-aws.yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-db-proxy
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-private-1,subnet-private-2"
spec:
  type: LoadBalancer
  selector:
    app: db-proxy
  externalTrafficPolicy: Local
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
    protocol: TCP
  - name: read-replica
    port: 5433
    targetPort: 5433
    protocol: TCP

---
# internal-lb-gke.yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  namespace: backend
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
    networking.gke.io/internal-load-balancer-subnet: "backend-subnet"
    networking.gke.io/internal-load-balancer-allow-global-access: "true"
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
```

### 9.3 LoadBalancer Source Ranges

```yaml
# restricted-lb.yaml - จำกัด IP ที่เข้าถึง LB ได้
apiVersion: v1
kind: Service
metadata:
  name: restricted-service
  namespace: production
spec:
  type: LoadBalancer
  selector:
    app: admin
  ports:
  - port: 443
    targetPort: 8443
  # อนุญาตเฉพาะ IP ranges เหล่านี้
  loadBalancerSourceRanges:
  - "10.0.0.0/8"         # Internal network
  - "203.0.113.0/24"     # Office IP range
  - "198.51.100.10/32"   # Specific jump host
```

---

## 10. Workshop: Production LoadBalancer Setup {#workshop}

### Workshop Overview

สร้าง Production-grade LoadBalancer setup สำหรับ multi-tier application

### Step 1: สร้าง Application Stack

```bash
kubectl create namespace production

kubectl apply -n production -f - <<'EOF'
# Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
---
# API Backend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
        tier: backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - -text
        - '{"status":"ok","pod":"$(HOSTNAME)"}'
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 5678
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 5
          periodSeconds: 10
EOF

kubectl wait --for=condition=Ready pods --all -n production --timeout=120s
```

### Step 2: สร้าง LoadBalancer Services

```bash
kubectl apply -n production -f - <<'EOF'
# External LoadBalancer สำหรับ Frontend
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  annotations:
    # MetalLB (bare-metal)
    metallb.universe.tf/address-pool: production-pool
    
    # หรือ AWS NLB
    # service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    # service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: frontend
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
---
# Internal LoadBalancer สำหรับ API
apiVersion: v1
kind: Service
metadata:
  name: api-lb-internal
  annotations:
    # MetalLB internal pool
    metallb.universe.tf/address-pool: internal-pool
spec:
  type: LoadBalancer
  selector:
    app: api
  externalTrafficPolicy: Local
  ports:
  - name: api
    port: 8080
    targetPort: 5678
EOF
```

### Step 3: ตรวจสอบและทดสอบ

```bash
# ดู External IPs
kubectl get services -n production -o wide
# รอ External IP ปรากฏ
kubectl get services -n production -w

# ทดสอบ Frontend
FRONTEND_IP=$(kubectl get service frontend-lb -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Frontend IP: $FRONTEND_IP"
curl http://$FRONTEND_IP

# ทดสอบ API (Internal)
API_IP=$(kubectl get service api-lb-internal -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "API IP: $API_IP"
curl http://$API_IP:8080

# ทดสอบ load distribution
echo "=== Load Balancing Test ==="
for i in $(seq 1 10); do
  curl -s http://$API_IP:8080 | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('pod','unknown'))"
done

# ดู MetalLB allocation
kubectl get ipaddresspool -n metallb-system
kubectl describe l2advertisement -n metallb-system
```

### Step 4: Health Checks

```bash
# ตรวจสอบ endpoints
kubectl get endpoints -n production

# ดู readiness ของแต่ละ pod
kubectl get pods -n production -o wide

# จำลอง pod failure
kubectl delete pod $(kubectl get pod -n production -l app=api -o jsonpath='{.items[0].metadata.name}') -n production

# ดูว่า LB หยุดส่ง traffic ไปที่ pod ที่ down แล้ว
kubectl get endpoints api-lb-internal -n production -w

# ดู events
kubectl events -n production
```

### Step 5: Cleanup

```bash
kubectl delete namespace production
```

---

## 11. แบบฝึกหัด LoadBalancer {#exercises}

### แบบฝึกหัดที่ 1: ติดตั้งและตั้งค่า MetalLB

```bash
# ติดตั้ง MetalLB (ถ้ายังไม่มี)
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# สร้าง IP Pool (ปรับ range ตาม network ของคุณ)
kubectl apply -f - <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: test-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: test-l2advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - test-pool
EOF

# สร้าง test service
kubectl create deployment test-web --image=nginx --replicas=2
kubectl expose deployment test-web --type=LoadBalancer --port=80

# รอ External IP
kubectl get service test-web -w

# ทดสอบ
EXT_IP=$(kubectl get service test-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXT_IP

# Cleanup
kubectl delete deployment test-web
kubectl delete service test-web
```

---

### แบบฝึกหัดที่ 2: เปรียบเทียบ Internal vs External LB

```bash
kubectl create namespace lb-compare

kubectl create deployment web -n lb-compare --image=nginx --replicas=2

# External LB
kubectl expose deployment web -n lb-compare \
  --type=LoadBalancer \
  --port=80 \
  --name=web-external

# Internal LB (MetalLB)
kubectl apply -n lb-compare -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-internal
  annotations:
    metallb.universe.tf/address-pool: internal-pool  # ต้องมี pool นี้
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

# รอ IPs
kubectl get services -n lb-compare -w

# เปรียบเทียบ
kubectl get services -n lb-compare -o wide

# Cleanup
kubectl delete namespace lb-compare
```

---

## 12. เฉลยแบบฝึกหัด {#answers}

### เฉลยแบบฝึกหัดที่ 1

```
ผลที่คาดหวัง:

MetalLB pods:
NAME                          READY   STATUS    RESTARTS
metallb-controller-xxxx       1/1     Running   0
metallb-speaker-aaaa          1/1     Running   0
metallb-speaker-bbbb          1/1     Running   0
metallb-speaker-cccc          1/1     Running   0

Service ได้ External IP:
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
test-web   LoadBalancer   10.96.100.5    192.168.1.200    80:30xxx/TCP

curl ส่งกลับ nginx default page:
<!DOCTYPE html><html><body><h1>Welcome to nginx!</h1>...

ถ้า External IP stuck ที่ <pending>:
1. ตรวจสอบว่า MetalLB pods running
2. ตรวจสอบว่า IP pool ไม่หมด
3. ดู controller logs: kubectl logs -n metallb-system -l component=controller
4. ดู events: kubectl describe service test-web
```

### เฉลยแบบฝึกหัดที่ 2

```
ผลที่คาดหวัง:

NAME           TYPE           CLUSTER-IP    EXTERNAL-IP      
web-external   LoadBalancer   10.96.x.x     192.168.1.201   ← routable from outside
web-internal   LoadBalancer   10.96.y.y     10.0.100.5      ← internal network only

ข้อแตกต่าง:
- External: IP routable จาก internet (ผ่าน router/gateway)
- Internal: IP accessible เฉพาะใน private network

Use cases:
- External: Web apps ที่ user เข้าถึงจาก internet
- Internal: Microservices ที่คุยกันภายใน, DB proxies,
            services ที่ access จาก on-premise ผ่าน VPN
```

---

## แหล่งข้อมูลเพิ่มเติม

- [MetalLB Documentation](https://metallb.universe.tf/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [GKE Load Balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer)
- [AKS Load Balancer](https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard)
- [Kubernetes LoadBalancer Spec](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
- [MetalLB BGP Mode](https://metallb.universe.tf/concepts/bgp/)

---

*ก่อนหน้า: [Part 33 - NodePort Service](./part-33-nodeport.md)*
*ต่อไป: [Part 35 - Ingress Controllers](./part-35-ingress-controllers.md)*
