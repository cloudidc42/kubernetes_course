# Part 89: Amazon EKS (Elastic Kubernetes Service)

## บทนำ

Amazon EKS (Elastic Kubernetes Service) คือ managed Kubernetes service ของ AWS ที่ช่วยให้คุณ run Kubernetes ได้โดยไม่ต้องจัดการ control plane เอง AWS จัดการ availability, security patching, และ scaling ของ control plane ให้คุณ

### ทำไมต้องใช้ EKS?

```
✅ AWS จัดการ control plane (99.95% SLA)
✅ Integration กับ AWS services (IAM, VPC, ELB, EBS, etc.)
✅ Auto-scaling ที่ยืดหยุ่น
✅ Security ด้วย IAM, KMS encryption
✅ เหมาะสำหรับ enterprises ที่ใช้ AWS
✅ Fargate support (serverless pods)
✅ Managed Node Groups
```

---

## 89.1 EKS Architecture

### Components

```
AWS Account
└── EKS Cluster
    ├── Control Plane (managed by AWS)
    │   ├── kube-apiserver
    │   ├── etcd (multi-AZ)
    │   ├── kube-controller-manager
    │   └── kube-scheduler
    │
    └── Data Plane (managed by you)
        ├── Managed Node Groups (EC2)
        │   ├── Node Group 1 (t3.large)
        │   └── Node Group 2 (m5.xlarge)
        ├── Self-managed Node Groups (EC2)
        └── Fargate Profiles (serverless)
```

### EKS Networking

```
VPC
├── Public Subnets (per AZ)
│   └── Load Balancers
└── Private Subnets (per AZ)
    └── Worker Nodes
        ├── Pods (VPC CNI - native VPC IPs)
        └── Services (ClusterIP, NodePort)
```

---

## 89.2 ติดตั้งและใช้ eksctl

### ติดตั้ง eksctl

```bash
# Linux
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# ตรวจสอบ
eksctl version

# ติดตั้ง AWS CLI
pip install awscli
aws configure
# Enter: Access Key, Secret Key, Region, Output format
```

### สร้าง EKS Cluster อย่างง่าย

```bash
# สร้าง cluster แบบง่ายที่สุด
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 5

# ดู cluster
eksctl get cluster
kubectl get nodes
kubectl cluster-info
```

### สร้าง EKS Cluster ด้วย Config File

```yaml
# eks-cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: us-east-1
  version: "1.28"
  tags:
    Environment: production
    Team: platform
    CostCenter: engineering

# VPC Configuration
vpc:
  cidr: 10.0.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
  publicAccessCIDRs:
    - "203.0.113.0/24"  # Your office IP
  
# IAM Configuration
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    - metadata:
        name: external-dns
        namespace: kube-system
      wellKnownPolicies:
        externalDNS: true
    - metadata:
        name: cluster-autoscaler
        namespace: kube-system
      wellKnownPolicies:
        autoScaler: true
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true

# Node Groups
managedNodeGroups:
  # General purpose nodes
  - name: general
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 100
    volumeType: gp3
    labels:
      role: general
      nodegroup-type: managed
    tags:
      NodeGroup: general
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true
    availabilityZones:
      - us-east-1a
      - us-east-1b
      - us-east-1c
    amiFamily: AmazonLinux2
    updateConfig:
      maxUnavailablePercentage: 25
  
  # Compute-intensive nodes
  - name: compute
    instanceType: c5.2xlarge
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    labels:
      role: compute
      nodegroup-type: managed
    taints:
      - key: dedicated
        value: compute
        effect: NoSchedule
  
  # Memory-intensive nodes
  - name: memory
    instanceType: r5.2xlarge
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    labels:
      role: memory
      nodegroup-type: managed
    taints:
      - key: dedicated
        value: memory
        effect: NoSchedule

# Fargate Profiles
fargateProfiles:
  - name: serverless
    selectors:
      - namespace: serverless
        labels:
          workload-type: fargate
      - namespace: kube-system
        labels:
          k8s-app: kube-dns

# Add-ons
addons:
  - name: vpc-cni
    version: latest
    configurationValues: |-
      env:
        ENABLE_PREFIX_DELEGATION: "true"
        WARM_PREFIX_TARGET: "1"
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true

# CloudWatch Logging
cloudWatch:
  clusterLogging:
    enableTypes:
      - audit
      - authenticator
      - controllerManager
      - scheduler
      - api
    logRetentionInDays: 30
```

```bash
# สร้าง cluster จาก config
eksctl create cluster -f eks-cluster-config.yaml

# ใช้เวลาประมาณ 15-20 นาที

# ดู progress
eksctl utils describe-stacks --region=us-east-1 --cluster=production-cluster

# Update kubeconfig
aws eks update-kubeconfig --name production-cluster --region us-east-1

# ตรวจสอบ
kubectl get nodes
kubectl get nodes -o custom-columns='NAME:.metadata.name,TYPE:.metadata.labels.role,STATUS:.status.conditions[-1].type'
```

---

## 89.3 IAM Integration

### IAM Roles for Service Accounts (IRSA)

IRSA ช่วยให้ Pods ใช้ IAM roles โดยไม่ต้อง share credentials

```bash
# สร้าง IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster production-cluster \
  --approve

# ดู OIDC provider
aws iam list-open-id-connect-providers | grep -i $(aws eks describe-cluster --name production-cluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

```bash
# สร้าง IAM role สำหรับ service account
eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace default \
  --cluster production-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts

# ดู service account
kubectl get serviceaccount s3-reader -o yaml
# จะเห็น annotation: eks.amazonaws.com/role-arn
```

```yaml
# ใช้ IRSA ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: s3-test
spec:
  serviceAccountName: s3-reader  # ใช้ SA ที่มี IAM role
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      command: ["aws", "s3", "ls"]
      # Pod จะมี AWS credentials อัตโนมัติจาก IRSA
```

### AWS Load Balancer Controller

```bash
# ติดตั้ง AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts

# ต้องมี IAM ServiceAccount ก่อน (ทำใน eksctl config หรือ)
eksctl create iamserviceaccount \
  --cluster=production-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller
```

```yaml
# Ingress ที่ใช้ AWS ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789:certificate/xxx
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/subnets: subnet-xxx,subnet-yyy
    alb.ingress.kubernetes.io/security-groups: sg-xxx
    alb.ingress.kubernetes.io/tags: Environment=production,Team=platform
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## 89.4 EKS Storage

### EBS CSI Driver

```bash
# ติดตั้ง EBS CSI Driver (ถ้าไม่ได้ทำใน eksctl config)
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster production-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster production-cluster \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole
```

```yaml
# StorageClass สำหรับ EBS GP3
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/xxx
```

### EFS CSI Driver

```bash
# ติดตั้ง EFS CSI Driver
eksctl create iamserviceaccount \
  --name efs-csi-controller-sa \
  --namespace kube-system \
  --cluster production-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --approve

helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa
```

```yaml
# StorageClass สำหรับ EFS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxxxxx
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"

---
# PVC ที่ใช้ EFS (ReadWriteMany support!)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 10Gi
```

---

## 89.5 Cluster Autoscaler

```bash
# ติดตั้ง Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=production-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.scale-down-delay-after-add=5m

# ดู logs
kubectl logs -n kube-system -l app.kubernetes.io/name=cluster-autoscaler -f
```

---

## 89.6 EKS Fargate

### กำหนด Fargate Profile

```yaml
# fargate-profile.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
  region: us-east-1

fargateProfiles:
  - name: default
    selectors:
      - namespace: fargate-workloads
  - name: kube-system
    selectors:
      - namespace: kube-system
        labels:
          eks.amazonaws.com/component: coredns
```

```bash
# สร้าง Fargate profile
eksctl create fargateprofile \
  --cluster production-cluster \
  --name serverless-apps \
  --namespace serverless

# ดู Fargate profiles
eksctl get fargateprofile --cluster production-cluster
```

```yaml
# Pod บน Fargate
apiVersion: v1
kind: Pod
metadata:
  name: fargate-pod
  namespace: serverless  # ต้อง match Fargate profile selector
spec:
  containers:
    - name: app
      image: nginx:1.25
  # ไม่ต้องกำหนด nodeSelector - Fargate profile จะ handle ให้
```

---

## 89.7 Workshop: Production EKS Setup

### เป้าหมาย
ตั้ง production-ready EKS cluster พร้อม security, monitoring, และ autoscaling

### ขั้นตอนที่ 1: สร้าง Production Cluster

```bash
# Export variables
export CLUSTER_NAME=prod-workshop
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# สร้าง cluster
cat > prod-cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.28"
  tags:
    Environment: production
    Workshop: true

vpc:
  cidr: 10.0.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    - metadata:
        name: cluster-autoscaler
        namespace: kube-system
      wellKnownPolicies:
        autoScaler: true
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true

managedNodeGroups:
  - name: system
    instanceType: m5.large
    minSize: 2
    maxSize: 4
    desiredCapacity: 2
    labels:
      role: system
    taints:
      - key: CriticalAddonsOnly
        value: "true"
        effect: NoSchedule

  - name: application
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    labels:
      role: application

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest

cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
EOF

eksctl create cluster -f prod-cluster.yaml
```

### ขั้นตอนที่ 2: ติดตั้ง Core Add-ons

```bash
# อัปเดต kubeconfig
aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}

# ติดตั้ง AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=${AWS_REGION} \
  --set vpcId=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.resourcesVpcConfig.vpcId" --output text)

# ติดตั้ง Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=${CLUSTER_NAME} \
  --set awsRegion=${AWS_REGION} \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler

# ติดตั้ง Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# ตรวจสอบ
kubectl get pods -n kube-system
kubectl top nodes
```

### ขั้นตอนที่ 3: ตั้ง Monitoring

```bash
# ติดตั้ง Prometheus + Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.annotations."kubernetes\.io/ingress\.class"=alb \
  --set grafana.ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"=internet-facing \
  --set grafana.ingress.hosts[0]=grafana.${DOMAIN}

# ดู Grafana password
kubectl get secret -n monitoring prometheus-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

### ขั้นตอนที่ 4: Deploy Sample Application

```yaml
# production-app.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: "1.0.0"
    spec:
      serviceAccountName: web-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web-app

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app
  namespace: production

---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
  ports:
    - port: 80

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: ${ACM_CERT_ARN}
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
kubectl apply -f production-app.yaml

# ดู status
kubectl get pods,svc,ingress -n production
kubectl describe ingress web-app -n production
# ดู ALB DNS name ใน Address

# ทดสอบ
ALB_DNS=$(kubectl get ingress web-app -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -k https://${ALB_DNS}/
```

### ขั้นตอนที่ 5: ทดสอบ Auto-scaling

```bash
# สร้าง load test
kubectl run -n production load-test \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://web-app:80; done"

# ดู HPA scaling
kubectl get hpa -n production -w

# ดู cluster autoscaling (เพิ่ม nodes)
kubectl get nodes -w
kubectl logs -n kube-system -l app.kubernetes.io/name=cluster-autoscaler -f

# หยุด load test
kubectl delete pod load-test -n production
```

### ขั้นตอนที่ 6: Cleanup

```bash
# ลบ resources ใน cluster
kubectl delete -f production-app.yaml

# ลบ addons
helm uninstall prometheus -n monitoring
helm uninstall aws-load-balancer-controller -n kube-system
helm uninstall cluster-autoscaler -n kube-system

# ลบ cluster
eksctl delete cluster --name ${CLUSTER_NAME} --region ${AWS_REGION}
```

---

## 89.8 EKS Security Best Practices

```bash
# 1. Enable envelope encryption สำหรับ Secrets
aws eks create-cluster \
  --name my-cluster \
  --kubernetes-version 1.28 \
  --role-arn ${CLUSTER_ROLE_ARN} \
  --resources-vpc-config subnetIds=${SUBNET_IDS} \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"${KMS_KEY_ARN}"}}]'

# 2. Enable Private Cluster
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true

# 3. Scan images สำหรับ vulnerabilities
# ใช้ Amazon ECR image scanning
aws ecr put-image-scanning-configuration \
  --repository-name my-repo \
  --image-scanning-configuration scanOnPush=true

# 4. Enable GuardDuty EKS Protection
aws guardduty create-detector --enable \
  --features '[{"Name":"EKS_AUDIT_LOGS","Status":"ENABLED"},{"Name":"EKS_RUNTIME_MONITORING","Status":"ENABLED"}]'
```

```yaml
# Network Policy สำหรับ Micro-segmentation
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
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-app
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: aws-load-balancer-controller
      ports:
        - port: 80
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **EKS Architecture**: Control plane, data plane, networking
2. **eksctl**: สร้างและจัดการ EKS clusters
3. **IAM Integration**: IRSA สำหรับ pod-level IAM permissions
4. **Storage**: EBS CSI, EFS CSI drivers
5. **Load Balancing**: AWS Load Balancer Controller
6. **Fargate**: Serverless pods
7. **Cluster Autoscaler**: Auto-scale nodes
8. **Workshop**: Production EKS setup แบบ end-to-end
9. **Security**: Encryption, private endpoints, GuardDuty

บทถัดไปเราจะเรียนรู้เกี่ยวกับ Google Kubernetes Engine (GKE)
