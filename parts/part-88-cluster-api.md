# Part 88: Cluster API (CAPI)

## บทนำ

Cluster API (CAPI) คือ Kubernetes-style APIs สำหรับการสร้าง จัดการ และดูแล Kubernetes clusters ตัวอื่น โดยใช้ Kubernetes เป็น control plane ในการ declare cluster infrastructure

CAPI ช่วยให้คุณ:
- สร้าง clusters ด้วย Kubernetes YAML manifests
- จัดการ cluster lifecycle (create, upgrade, scale, delete)
- รองรับ infrastructure providers หลายประเภท (AWS, GCP, Azure, vSphere, etc.)
- ทำ cluster upgrades อย่าง automated
- Scale node pools ได้

---

## 88.1 Cluster API Architecture

### Components

```
Management Cluster (ที่ติดตั้ง CAPI)
├── Core CAPI Components
│   ├── Cluster Controller
│   ├── Machine Controller
│   └── MachineSet Controller
│
├── Bootstrap Provider
│   └── Kubeadm Bootstrap Provider (CABPK)
│
├── Control Plane Provider
│   └── Kubeadm Control Plane Provider (KCP)
│
└── Infrastructure Provider
    ├── AWS Provider (CAPA)
    ├── GCP Provider (CAPG)
    ├── Azure Provider (CAPZ)
    ├── vSphere Provider (CAPV)
    └── Docker Provider (CAPD) - สำหรับ testing

Workload Clusters (ที่ถูกสร้างโดย CAPI)
├── Cluster 1
├── Cluster 2
└── Cluster 3
```

### API Resources

```
Cluster (core)         - Cluster definition
Machine (core)         - Individual node
MachineSet (core)      - Group of Machines
MachineDeployment (core) - Managed group with rolling updates

KubeadmConfig (CABPK)  - Bootstrap configuration
KubeadmControlPlane (KCP) - Control plane management

AWSCluster (CAPA)      - AWS-specific cluster
AWSMachine (CAPA)      - AWS EC2 instance
AWSMachineTemplate (CAPA) - Template for AWS machines
```

---

## 88.2 ติดตั้ง CAPI

### Prerequisites

```bash
# ติดตั้ง clusterctl
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.6.0/clusterctl-linux-amd64 \
  -o clusterctl
chmod +x clusterctl
sudo mv clusterctl /usr/local/bin/

# ตรวจสอบ
clusterctl version

# ต้องมี management cluster อยู่ก่อน
kubectl cluster-info
```

### ติดตั้งด้วย Docker Provider (สำหรับ testing)

```bash
# สร้าง management cluster ด้วย kind
cat <<EOF | kind create cluster --name capi-management --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
EOF

# Initialize CAPI ด้วย Docker provider
clusterctl init --infrastructure docker

# ดู components ที่ถูก install
kubectl get pods -n capi-system
kubectl get pods -n capi-kubeadm-bootstrap-system
kubectl get pods -n capi-kubeadm-control-plane-system
kubectl get pods -n capd-system
```

### ติดตั้งด้วย AWS Provider

```bash
# ตั้งค่า AWS credentials
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
export AWS_SESSION_TOKEN=<your-session-token>  # ถ้าใช้ temporary credentials

# Encode credentials
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)

# Initialize CAPI
clusterctl init --infrastructure aws

# ดู components
kubectl get pods -n capa-system
```

---

## 88.3 สร้าง Cluster ด้วย Docker Provider

### Generate Cluster Configuration

```bash
# Generate configuration สำหรับ Docker cluster
clusterctl generate cluster capi-quickstart \
  --infrastructure docker \
  --kubernetes-version v1.28.0 \
  --control-plane-machine-count=1 \
  --worker-machine-count=3 \
  > capi-quickstart.yaml

# ดูไฟล์ที่ generate
cat capi-quickstart.yaml
```

### ตัวอย่าง Cluster Manifest

```yaml
# capi-docker-cluster.yaml
---
# Cluster definition
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.128.0.0/12"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: DockerCluster
    name: my-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-cluster-cp

---
# Docker Cluster (infrastructure-specific)
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: DockerCluster
metadata:
  name: my-cluster
  namespace: default

---
# Control Plane
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-cluster-cp
  namespace: default
spec:
  replicas: 1
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: DockerMachineTemplate
      name: my-cluster-cp
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          enable-admission-plugins: NodeRestriction
  version: v1.28.0

---
# Control Plane Machine Template
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: DockerMachineTemplate
metadata:
  name: my-cluster-cp
  namespace: default
spec:
  template:
    spec: {}

---
# Worker MachineDeployment
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: my-cluster-workers
  namespace: default
spec:
  clusterName: my-cluster
  replicas: 3
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: my-cluster
    spec:
      clusterName: my-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: my-cluster-workers
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: DockerMachineTemplate
        name: my-cluster-workers

---
# Worker Machine Template
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: DockerMachineTemplate
metadata:
  name: my-cluster-workers
  namespace: default
spec:
  template:
    spec: {}

---
# Bootstrap Config Template
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: my-cluster-workers
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            eviction-hard: "imagefs.available<0%,nodefs.available<0%"
            fail-swap-on: "false"
```

```bash
# Apply cluster
kubectl apply -f capi-docker-cluster.yaml

# ดู status
kubectl get cluster
kubectl get kubeadmcontrolplane
kubectl get machinedeployment
kubectl get machines
kubectl get nodes --context=my-cluster
```

---

## 88.4 สร้าง Cluster บน AWS

### AWS Cluster Configuration

```yaml
# aws-cluster.yaml
---
# AWS Cluster
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: my-aws-cluster
  namespace: default
spec:
  region: us-east-1
  sshKeyName: my-keypair
  network:
    vpc:
      cidrBlock: 10.0.0.0/16
    subnets:
      - cidrBlock: 10.0.1.0/24
        availabilityZone: us-east-1a
        isPublic: true
      - cidrBlock: 10.0.2.0/24
        availabilityZone: us-east-1b
        isPublic: true
      - cidrBlock: 10.0.10.0/24
        availabilityZone: us-east-1a
        isPublic: false
      - cidrBlock: 10.0.20.0/24
        availabilityZone: us-east-1b
        isPublic: false

---
# Cluster
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-aws-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: my-aws-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-aws-cluster-cp

---
# AWS Machine Template สำหรับ Control Plane
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachineTemplate
metadata:
  name: my-aws-cluster-cp
  namespace: default
spec:
  template:
    spec:
      instanceType: t3.large
      iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
      sshKeyName: my-keypair
      rootVolume:
        size: 50
        type: gp3
      ami:
        id: ami-0abcdef1234567890  # Amazon Linux 2

---
# Control Plane
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-aws-cluster-cp
  namespace: default
spec:
  replicas: 3
  version: v1.28.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: AWSMachineTemplate
      name: my-aws-cluster-cp
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        name: "{{ ds.meta_data.local_hostname }}"
        kubeletExtraArgs:
          cloud-provider: aws
    clusterConfiguration:
      apiServer:
        extraArgs:
          cloud-provider: aws
      controllerManager:
        extraArgs:
          cloud-provider: aws
    joinConfiguration:
      nodeRegistration:
        name: "{{ ds.meta_data.local_hostname }}"
        kubeletExtraArgs:
          cloud-provider: aws

---
# AWS Machine Template สำหรับ Workers
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachineTemplate
metadata:
  name: my-aws-cluster-workers
  namespace: default
spec:
  template:
    spec:
      instanceType: t3.xlarge
      iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
      sshKeyName: my-keypair
      rootVolume:
        size: 100
        type: gp3

---
# Worker KubeadmConfigTemplate
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: my-aws-cluster-workers
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          name: "{{ ds.meta_data.local_hostname }}"
          kubeletExtraArgs:
            cloud-provider: aws

---
# Worker MachineDeployment
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: my-aws-cluster-workers
  namespace: default
spec:
  clusterName: my-aws-cluster
  replicas: 3
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-aws-cluster
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: my-aws-cluster
    spec:
      clusterName: my-aws-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: my-aws-cluster-workers
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: my-aws-cluster-workers
```

---

## 88.5 Machine Pools (Node Groups)

```yaml
# machinepool.yaml - สำหรับ cloud-managed node groups
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachinePool
metadata:
  name: my-cluster-pool
  namespace: default
spec:
  clusterName: my-aws-cluster
  replicas: 3
  template:
    spec:
      clusterName: my-aws-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfig
          name: my-cluster-pool
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachinePool
        name: my-cluster-pool

---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachinePool
metadata:
  name: my-cluster-pool
  namespace: default
spec:
  minSize: 2
  maxSize: 10
  availabilityZones:
    - us-east-1a
    - us-east-1b
  awsLaunchTemplate:
    instanceType: t3.xlarge
    iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
    sshKeyName: my-keypair
```

---

## 88.6 Cluster Lifecycle Management

### Upgrade Kubernetes Version

```bash
# อัปเกรด control plane
kubectl patch kubeadmcontrolplane my-aws-cluster-cp \
  --type=merge \
  -p '{"spec":{"version":"v1.29.0"}}'

# ดู upgrade progress
kubectl get kubeadmcontrolplane my-aws-cluster-cp -w
kubectl get machines -w

# อัปเกรด workers
kubectl patch machinedeployment my-aws-cluster-workers \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"version":"v1.29.0"}}}}'

# ดู rolling update progress
kubectl get machinedeployment my-aws-cluster-workers -w
kubectl get machines -w
```

### Scale Cluster

```bash
# เพิ่ม workers
kubectl scale machinedeployment my-aws-cluster-workers --replicas=5

# เพิ่ม control plane (ต้องเป็น odd number: 1, 3, 5)
kubectl patch kubeadmcontrolplane my-aws-cluster-cp \
  --type=merge \
  -p '{"spec":{"replicas":3}}'

# ดู machines
kubectl get machines
```

### Delete Cluster

```bash
# ลบ cluster (และ infrastructure ทั้งหมด)
kubectl delete cluster my-aws-cluster

# ดู deletion progress
kubectl get cluster my-aws-cluster -w
kubectl get machines -w
# Machines จะถูกลบก่อน จากนั้น cluster infrastructure
```

---

## 88.7 Workshop: Cluster Lifecycle Management

### เป้าหมาย
Create, upgrade, scale, และ delete Kubernetes cluster ด้วย CAPI

### ขั้นตอนที่ 1: Setup Management Cluster

```bash
# สร้าง management cluster
kind create cluster --name capi-lab \
  --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
EOF

# Initialize CAPI
clusterctl init --infrastructure docker

# รอให้ Ready
kubectl wait --for=condition=ready pod -A -l cluster.x-k8s.io/provider=infrastructure-docker --timeout=300s
```

### ขั้นตอนที่ 2: สร้าง Workload Cluster

```bash
# Generate cluster config
export CLUSTER_NAME=workload-1
export KUBERNETES_VERSION=v1.28.0

clusterctl generate cluster ${CLUSTER_NAME} \
  --infrastructure docker \
  --kubernetes-version ${KUBERNETES_VERSION} \
  --control-plane-machine-count=1 \
  --worker-machine-count=2 \
  > workload-cluster.yaml

# Apply
kubectl apply -f workload-cluster.yaml

# Watch progress
kubectl get cluster ${CLUSTER_NAME} -w &
kubectl get kubeadmcontrolplane -w &
kubectl get machinedeployment -w &

# รอให้ Ready
kubectl wait --for=condition=ready cluster ${CLUSTER_NAME} --timeout=300s
```

### ขั้นตอนที่ 3: เข้าถึง Workload Cluster

```bash
# ดึง kubeconfig
clusterctl get kubeconfig ${CLUSTER_NAME} > workload-kubeconfig.yaml

# ใช้ kubeconfig
export KUBECONFIG=workload-kubeconfig.yaml

# ดู nodes
kubectl get nodes

# ติดตั้ง CNI (จำเป็นสำหรับ pod networking)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# รอให้ nodes ready
kubectl wait --for=condition=ready node --all --timeout=300s

# ดู nodes
kubectl get nodes
```

### ขั้นตอนที่ 4: Deploy Application ใน Workload Cluster

```bash
# Deploy test application
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
  type: NodePort
EOF

kubectl get deployment nginx
kubectl get pods
```

### ขั้นตอนที่ 5: Scale Cluster

```bash
# กลับไป management cluster
export KUBECONFIG=~/.kube/config

# Scale workers จาก 2 เป็น 4
kubectl scale machinedeployment ${CLUSTER_NAME}-md-0 --replicas=4

# ดู progress
kubectl get machinedeployment ${CLUSTER_NAME}-md-0 -w
kubectl get machines -w

# ดู nodes ใน workload cluster
export KUBECONFIG=workload-kubeconfig.yaml
kubectl get nodes -w
```

### ขั้นตอนที่ 6: Upgrade Kubernetes Version

```bash
# กลับไป management cluster
export KUBECONFIG=~/.kube/config

# อัปเกรด control plane
kubectl patch kubeadmcontrolplane ${CLUSTER_NAME}-control-plane \
  --type=merge \
  -p '{"spec":{"version":"v1.29.0"}}'

# ดู progress
kubectl get kubeadmcontrolplane ${CLUSTER_NAME}-control-plane -w

# อัปเกรด workers
kubectl patch machinedeployment ${CLUSTER_NAME}-md-0 \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"version":"v1.29.0"}}}}'

kubectl get machinedeployment ${CLUSTER_NAME}-md-0 -w

# ยืนยัน version
export KUBECONFIG=workload-kubeconfig.yaml
kubectl get nodes
```

### ขั้นตอนที่ 7: Cleanup

```bash
# กลับไป management cluster
export KUBECONFIG=~/.kube/config

# ลบ workload cluster
kubectl delete cluster ${CLUSTER_NAME}

# รอให้ลบเสร็จ
kubectl wait --for=delete cluster ${CLUSTER_NAME} --timeout=300s

# ลบ management cluster
kind delete cluster --name capi-lab
```

---

## 88.8 ClusterClass (Managed Topology)

ClusterClass คือ feature ใหม่ใน CAPI ที่ช่วยให้กำหนด topology ของ cluster เป็น template ที่ reuse ได้

```yaml
# clusterclass.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: quick-start
  namespace: default
spec:
  controlPlane:
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta1
      kind: KubeadmControlPlaneTemplate
      name: quick-start-control-plane
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: DockerMachineTemplate
        name: quick-start-control-plane
  infrastructure:
    ref:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: DockerClusterTemplate
      name: quick-start
  workers:
    machineDeployments:
      - class: default-worker
        template:
          bootstrap:
            ref:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
              kind: KubeadmConfigTemplate
              name: quick-start-worker
          infrastructure:
            ref:
              apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
              kind: DockerMachineTemplate
              name: quick-start-worker
  variables:
    - name: imageRepository
      required: true
      schema:
        openAPIV3Schema:
          type: string
          default: k8s.gcr.io
    - name: etcdImageTag
      required: true
      schema:
        openAPIV3Schema:
          type: string
          default: 3.5.6-0
  patches:
    - name: imageRepository
      description: "Sets the imageRepository used for the KubeadmControlPlane."
      enabledIf: '{{ ne .imageRepository "k8s.gcr.io" }}'
      definitions:
        - selector:
            apiVersion: controlplane.cluster.x-k8s.io/v1beta1
            kind: KubeadmControlPlaneTemplate
            matchResources:
              controlPlane: true
          jsonPatches:
            - op: add
              path: /spec/template/spec/kubeadmConfigSpec/clusterConfiguration/imageRepository
              valueFrom:
                variable: imageRepository

---
# ใช้ ClusterClass
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  topology:
    class: quick-start
    version: v1.28.0
    controlPlane:
      replicas: 3
    workers:
      machineDeployments:
        - name: worker
          class: default-worker
          replicas: 5
    variables:
      - name: imageRepository
        value: k8s.gcr.io
      - name: etcdImageTag
        value: 3.5.6-0
```

---

## 88.9 Monitoring CAPI

```bash
# ดู cluster status
clusterctl describe cluster my-cluster

# ดูรายละเอียด
kubectl get cluster my-cluster -o yaml | grep -A 20 status

# ดู machine status
kubectl get machines -o wide

# ดู events
kubectl get events --field-selector involvedObject.kind=Cluster

# ดู CAPI controller logs
kubectl logs -n capi-system deploy/capi-controller-manager -f
kubectl logs -n capi-kubeadm-control-plane-system deploy/capi-kubeadm-control-plane-controller-manager -f
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CAPI Architecture**: Management cluster, providers, workload clusters
2. **ติดตั้ง CAPI**: clusterctl, Docker provider, AWS provider
3. **สร้าง Clusters**: Docker และ AWS
4. **Machine Pools**: Cloud-managed node groups
5. **Cluster Lifecycle**: Upgrade, Scale, Delete
6. **ClusterClass**: Reusable cluster templates
7. **Workshop**: สร้าง, อัปเกรด, และ scale cluster
8. **Monitoring**: ตรวจสอบ cluster status

บทถัดไปเราจะเรียนรู้เกี่ยวกับ Amazon EKS
