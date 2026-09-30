# Part 91: Azure Kubernetes Service (AKS)

## บทนำ

Azure Kubernetes Service (AKS) เป็น Managed Kubernetes Service จาก Microsoft Azure ที่ช่วยลดความซับซ้อนในการ Deploy และจัดการ Kubernetes Cluster บน Cloud AKS จัดการ Control Plane ให้โดยอัตโนมัติ ทำให้ทีมสามารถโฟกัสที่การพัฒนา Application ได้มากขึ้น

## สารบัญ

1. AKS Overview และ Architecture
2. การติดตั้ง AKS
3. Azure AD Integration
4. AKS-specific Features
5. Networking ใน AKS
6. Storage ใน AKS
7. Monitoring และ Logging
8. Security Best Practices
9. Workshop: Production AKS Setup

---

## 1. AKS Overview และ Architecture

### 1.1 AKS Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Subscription                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  AKS Cluster                          │   │
│  │  ┌─────────────┐    ┌──────────────────────────────┐│   │
│  │  │ Control Plane│    │       Node Pools             ││   │
│  │  │ (Azure Managed)│  │  ┌──────────┐ ┌──────────┐  ││   │
│  │  │ - API Server │    │  │System Pool│ │User Pools │  ││   │
│  │  │ - etcd       │    │  │(System)   │ │(Workload) │  ││   │
│  │  │ - Scheduler  │    │  └──────────┘ └──────────┘  ││   │
│  │  │ - Controller │    └──────────────────────────────┘│   │
│  │  └─────────────┘                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │ Azure AD    │  │  Azure CNI  │  │  Azure Monitor   │    │
│  │ Integration │  │  Networking │  │  (Log Analytics) │    │
│  └─────────────┘  └─────────────┘  └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 AKS Pricing Model

- **Control Plane**: ฟรีสำหรับ Standard Tier (มีค่าใช้จ่ายสำหรับ Uptime SLA)
- **Node Pools**: จ่ายตาม VM ที่ใช้
- **Managed Disks**: จ่ายตาม Storage ที่ใช้
- **Load Balancers**: จ่ายตาม Azure Load Balancer

### 1.3 AKS Tiers

| Feature | Free Tier | Standard Tier |
|---------|-----------|---------------|
| SLA | ไม่มี | 99.95% |
| Support | Community | Azure Support |
| Node Count | สูงสุด 10 | ไม่จำกัด |
| Uptime SLA | ไม่มี | มี |

---

## 2. การติดตั้ง AKS

### 2.1 Prerequisites

```bash
# ติดตั้ง Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login เข้า Azure
az login

# ตั้งค่า Subscription
az account set --subscription "your-subscription-id"

# ติดตั้ง kubectl
az aks install-cli

# ติดตั้ง Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 2.2 สร้าง Resource Group

```bash
# สร้าง Resource Group
az group create \
  --name myAKSResourceGroup \
  --location southeastasia

# ตรวจสอบ
az group show --name myAKSResourceGroup
```

### 2.3 สร้าง AKS Cluster แบบ Basic

```bash
# สร้าง AKS Cluster แบบง่าย
az aks create \
  --resource-group myAKSResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --generate-ssh-keys \
  --enable-managed-identity \
  --location southeastasia

# รับ Credentials
az aks get-credentials \
  --resource-group myAKSResourceGroup \
  --name myAKSCluster

# ทดสอบ
kubectl get nodes
```

### 2.4 สร้าง AKS Cluster แบบ Production

```bash
# สร้าง Virtual Network ก่อน
az network vnet create \
  --resource-group myAKSResourceGroup \
  --name myVNet \
  --address-prefix 10.0.0.0/8 \
  --subnet-name myAKSSubnet \
  --subnet-prefix 10.240.0.0/16

# รับ Subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group myAKSResourceGroup \
  --vnet-name myVNet \
  --name myAKSSubnet \
  --query id -o tsv)

# สร้าง AKS Cluster Production Grade
az aks create \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --min-count 3 \
  --max-count 10 \
  --enable-cluster-autoscaler \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids "your-admin-group-id" \
  --enable-azure-rbac \
  --uptime-sla \
  --zones 1 2 3 \
  --generate-ssh-keys \
  --tier standard \
  --enable-addons monitoring \
  --workspace-resource-id "/subscriptions/sub-id/resourceGroups/rg/providers/Microsoft.OperationalInsights/workspaces/workspace-name"
```

### 2.5 Node Pools

```bash
# เพิ่ม Node Pool สำหรับ Workload เฉพาะ
az aks nodepool add \
  --resource-group myAKSResourceGroup \
  --cluster-name myProductionAKS \
  --name gpupool \
  --node-count 2 \
  --node-vm-size Standard_NC6 \
  --node-taints sku=gpu:NoSchedule \
  --node-labels workload=gpu \
  --mode User

# เพิ่ม Spot Node Pool
az aks nodepool add \
  --resource-group myAKSResourceGroup \
  --cluster-name myProductionAKS \
  --name spotnodepool \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 20 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule

# อัพเดท Node Pool
az aks nodepool update \
  --resource-group myAKSResourceGroup \
  --cluster-name myProductionAKS \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 15

# ลบ Node Pool
az aks nodepool delete \
  --resource-group myAKSResourceGroup \
  --cluster-name myProductionAKS \
  --name gpupool
```

### 2.6 Upgrade AKS Cluster

```bash
# ดู Versions ที่มี
az aks get-upgrades \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --output table

# Upgrade Control Plane
az aks upgrade \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --kubernetes-version 1.28.0

# Upgrade Node Pool เฉพาะ
az aks nodepool upgrade \
  --resource-group myAKSResourceGroup \
  --cluster-name myProductionAKS \
  --name nodepool1 \
  --kubernetes-version 1.28.0
```

---

## 3. Azure AD Integration

### 3.1 AKS-managed Azure AD

```bash
# Enable Azure AD Integration ใหม่
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-aad \
  --aad-admin-group-object-ids "your-admin-group-id"

# สร้าง ClusterRoleBinding สำหรับ Azure AD Group
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: aks-cluster-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "your-azure-ad-group-id"  # Object ID ของ Azure AD Group
EOF
```

### 3.2 Azure RBAC สำหรับ Kubernetes

```bash
# Enable Azure RBAC
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-azure-rbac

# กำหนด Role ให้ User
az role assignment create \
  --role "Azure Kubernetes Service RBAC Admin" \
  --assignee "user@example.com" \
  --scope "/subscriptions/sub-id/resourceGroups/myAKSResourceGroup/providers/Microsoft.ContainerService/managedClusters/myProductionAKS"

# Role ที่มีให้ใช้:
# - Azure Kubernetes Service RBAC Admin
# - Azure Kubernetes Service RBAC Cluster Admin
# - Azure Kubernetes Service RBAC Reader
# - Azure Kubernetes Service RBAC Writer
```

### 3.3 Workload Identity (แนะนำสำหรับ Production)

```bash
# Enable OIDC Issuer
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-oidc-issuer

# Enable Workload Identity
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-workload-identity

# รับ OIDC Issuer URL
OIDC_ISSUER=$(az aks show \
  --name myProductionAKS \
  --resource-group myAKSResourceGroup \
  --query "oidcIssuerProfile.issuerUrl" \
  -otsv)

echo "OIDC Issuer: $OIDC_ISSUER"
```

### 3.4 สร้าง Managed Identity สำหรับ Application

```bash
# สร้าง User Assigned Managed Identity
az identity create \
  --name myAppIdentity \
  --resource-group myAKSResourceGroup \
  --location southeastasia

# รับ Client ID
IDENTITY_CLIENT_ID=$(az identity show \
  --name myAppIdentity \
  --resource-group myAKSResourceGroup \
  --query 'clientId' -o tsv)

# รับ Object ID
IDENTITY_OBJECT_ID=$(az identity show \
  --name myAppIdentity \
  --resource-group myAKSResourceGroup \
  --query 'principalId' -o tsv)

# กำหนด Role สำหรับ Identity
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee-object-id $IDENTITY_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/sub-id/resourceGroups/myDataRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount"

# สร้าง Kubernetes Service Account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: "$IDENTITY_CLIENT_ID"
EOF

# สร้าง Federated Identity Credential
az identity federated-credential create \
  --name myAppFederatedCred \
  --identity-name myAppIdentity \
  --resource-group myAKSResourceGroup \
  --issuer "$OIDC_ISSUER" \
  --subject "system:serviceaccount:default:my-app-sa" \
  --audiences api://AzureADTokenExchange
```

### 3.5 การใช้ Workload Identity ใน Pod

```yaml
# pod-with-workload-identity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    azure.workload.identity/use: "true"  # สำคัญมาก
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: AZURE_CLIENT_ID
      value: "your-client-id"
    - name: AZURE_TENANT_ID
      value: "your-tenant-id"
    - name: AZURE_FEDERATED_TOKEN_FILE
      value: "/var/run/secrets/azure/tokens/azure-identity-token"
    volumeMounts:
    - name: azure-identity-token
      mountPath: /var/run/secrets/azure/tokens
      readOnly: true
  volumes:
  - name: azure-identity-token
    projected:
      sources:
      - serviceAccountToken:
          audience: api://AzureADTokenExchange
          expirationSeconds: 3600
          path: azure-identity-token
```

---

## 4. AKS-specific Features

### 4.1 Azure Container Registry Integration

```bash
# สร้าง ACR
az acr create \
  --resource-group myAKSResourceGroup \
  --name myaksacr \
  --sku Standard

# Attach ACR ให้ AKS
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --attach-acr myaksacr

# Build และ Push Image
az acr build \
  --registry myaksacr \
  --image myapp:v1 \
  .

# ใช้ Image จาก ACR
kubectl create deployment myapp \
  --image=myaksacr.azurecr.io/myapp:v1
```

### 4.2 AKS Cluster Autoscaler

```bash
# Enable Cluster Autoscaler
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20

# ปรับ Autoscaler Profile
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --cluster-autoscaler-profile \
    scan-interval=30s \
    scale-down-delay-after-add=10m \
    scale-down-unneeded-time=10m \
    scale-down-utilization-threshold=0.5 \
    max-graceful-termination-sec=600
```

### 4.3 Virtual Nodes (ACI Integration)

```bash
# Enable Virtual Nodes
az aks enable-addons \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --addons virtual-node \
  --subnet-name myVirtualNodeSubnet

# Deploy Pod ไปที่ Virtual Node
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: virtual-node-demo
  namespace: default
spec:
  tolerations:
  - key: virtual-kubelet.io/provider
    operator: Exists
  nodeSelector:
    kubernetes.io/role: agent
    beta.kubernetes.io/os: linux
    type: virtual-kubelet
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF
```

### 4.4 Azure Policy สำหรับ AKS

```bash
# Enable Azure Policy Addon
az aks enable-addons \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --addons azure-policy

# ดู Policy Definitions ที่มี
az policy definition list \
  --query "[?contains(displayName, 'Kubernetes')]" \
  --output table
```

```yaml
# ตัวอย่าง Constraint Template สำหรับ Azure Policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels:
    - key: team
      allowedRegex: "^[a-zA-Z]+$"
```

### 4.5 KEDA (Kubernetes Event-driven Autoscaling)

```bash
# ติดตั้ง KEDA Addon
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-keda
```

```yaml
# keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-servicebus-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-app-deployment
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: my-queue
      namespace: my-servicebus-namespace
      messageCount: "5"
    authenticationRef:
      name: azure-servicebus-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-auth
  namespace: default
spec:
  podIdentity:
    provider: azure-workload
```

### 4.6 Azure Key Vault Provider for Secrets Store CSI Driver

```bash
# Enable Secrets Store CSI Driver
az aks enable-addons \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --addons azure-keyvault-secrets-provider

# ตรวจสอบ
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)'
```

```yaml
# keyvault-secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "your-managed-identity-client-id"
    keyvaultName: "myKeyVault"
    cloudName: ""
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: api-key
          objectType: secret
    tenantId: "your-tenant-id"
  secretObjects:
  - secretName: db-password
    type: Opaque
    data:
    - objectName: db-password
      key: password
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-secrets-store-inline
spec:
  containers:
  - name: busybox
    image: k8s.gcr.io/e2e-test-images/busybox:1.29-4
    command: ["sleep", "3600"]
    volumeMounts:
    - name: secrets-store01-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
  - name: secrets-store01-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-kvname"
```

---

## 5. Networking ใน AKS

### 5.1 Azure CNI

```bash
# สร้าง AKS ด้วย Azure CNI
az aks create \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10
```

### 5.2 Network Policies

```yaml
# ติดตั้ง Calico Network Policy
# สร้าง AKS ด้วย Network Policy
az aks create \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --network-plugin azure \
  --network-policy calico

# ตัวอย่าง Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 5.3 Ingress Controller บน AKS

```bash
# ติดตั้ง NGINX Ingress Controller ด้วย Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-nginx \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set controller.service.externalTrafficPolicy=Local

# ตรวจสอบ External IP
kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 5.4 Application Gateway Ingress Controller (AGIC)

```bash
# Enable AGIC Addon
az aks enable-addons \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --addons ingress-appgw \
  --appgw-name myApplicationGateway \
  --appgw-subnet-cidr "10.225.0.0/16"
```

```yaml
# agic-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: agic-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/connection-draining: "true"
    appgw.ingress.kubernetes.io/connection-draining-timeout: "30"
spec:
  tls:
  - secretName: my-tls-secret
    hosts:
    - myapp.example.com
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## 6. Storage ใน AKS

### 6.1 Azure Disk Storage Class

```yaml
# azure-disk-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
  kind: Managed
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# PVC สำหรับ Azure Disk
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-disk-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 10Gi
```

### 6.2 Azure Files Storage Class

```yaml
# azure-files-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-premium
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
  enableLargeFileShares: "true"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- dir_mode=0777
- file_mode=0777
- uid=0
- gid=0
- mfsymlinks
- cache=strict
- actimeo=30
---
# PVC สำหรับ Azure Files
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-files-pvc
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile-premium
  resources:
    requests:
      storage: 100Gi
```

### 6.3 Azure Blob Storage NFS

```yaml
# blob-nfs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azureblob-nfs-premium
provisioner: blob.csi.azure.com
parameters:
  protocol: nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- nconnect=4
```

---

## 7. Monitoring และ Logging

### 7.1 Container Insights

```bash
# Enable Container Insights
az aks enable-addons \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --addons monitoring \
  --workspace-resource-id "/subscriptions/sub-id/resourceGroups/rg/providers/Microsoft.OperationalInsights/workspaces/myWorkspace"

# ดู Logs ใน Log Analytics
# KQL Query ตัวอย่าง:
# ContainerLog
# | where TimeGenerated > ago(1h)
# | where LogEntry contains "error"
# | summarize count() by ContainerName, bin(TimeGenerated, 5m)
# | render timechart
```

### 7.2 Prometheus และ Grafana บน AKS

```bash
# ติดตั้ง Azure Managed Prometheus
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-azure-monitor-metrics

# ติดตั้ง Grafana
az grafana create \
  --resource-group myAKSResourceGroup \
  --name myGrafana

# เชื่อมต่อ Grafana กับ Prometheus
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id "/subscriptions/sub-id/resourceGroups/rg/providers/microsoft.monitor/accounts/myPrometheusWorkspace" \
  --grafana-resource-id "/subscriptions/sub-id/resourceGroups/rg/providers/Microsoft.Dashboard/grafana/myGrafana"
```

### 7.3 Alert Rules

```json
{
  "properties": {
    "description": "Alert when Pod is in CrashLoopBackOff",
    "severity": 2,
    "enabled": true,
    "evaluationFrequency": "PT1M",
    "windowSize": "PT5M",
    "criteria": {
      "odata.type": "Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria",
      "allOf": [
        {
          "name": "CrashLoopBackOff",
          "metricName": "kube_pod_container_status_waiting_reason",
          "dimensions": [
            {
              "name": "reason",
              "operator": "Include",
              "values": ["CrashLoopBackOff"]
            }
          ],
          "operator": "GreaterThan",
          "threshold": 0,
          "timeAggregation": "Average"
        }
      ]
    }
  }
}
```

---

## 8. Security Best Practices สำหรับ AKS

### 8.1 Microsoft Defender for Containers

```bash
# Enable Microsoft Defender for Containers
az security pricing create \
  --name Containers \
  --tier "Standard"

# ตรวจสอบ Security Recommendations
az security recommendation list \
  --query "[?contains(name, 'Kubernetes')]" \
  --output table
```

### 8.2 Pod Security Standards

```yaml
# namespace-security.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```yaml
# secure-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: myaksacr.azurecr.io/myapp:v1
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

### 8.3 Image Scanning

```bash
# Enable Image Scan ใน ACR
az acr update \
  --name myaksacr \
  --resource-group myAKSResourceGroup \
  --sku Premium

# ดู Vulnerability Scan Results
az acr repository show \
  --name myaksacr \
  --image myapp:v1

# AKS Image Cleaner
az aks update \
  --resource-group myAKSResourceGroup \
  --name myProductionAKS \
  --enable-image-cleaner \
  --image-cleaner-interval-hours 48
```

### 8.4 Private Cluster

```bash
# สร้าง Private AKS Cluster
az aks create \
  --resource-group myAKSResourceGroup \
  --name myPrivateAKS \
  --enable-private-cluster \
  --private-dns-zone system \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --enable-managed-identity

# เข้าถึง Private Cluster ผ่าน Azure Bastion หรือ VPN
az aks command invoke \
  --resource-group myAKSResourceGroup \
  --name myPrivateAKS \
  --command "kubectl get pods -A"
```

---

## 9. Workshop: Production AKS Setup

### Workshop Overview

ในบทนี้เราจะ Deploy Production-grade AKS Cluster พร้อม:
- Azure AD Integration
- Network Security
- Monitoring
- Auto-scaling
- Secure Image Storage

### Step 1: เตรียม Environment

```bash
# ตั้งค่า Variables
export RESOURCE_GROUP="myProductionRG"
export CLUSTER_NAME="myProductionAKS"
export LOCATION="southeastasia"
export ACR_NAME="myproductionacr"
export VNET_NAME="myProductionVNet"
export KEYVAULT_NAME="myProductionKV"

# Login
az login
az account set --subscription "your-subscription-id"

# สร้าง Resource Group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 2: สร้าง Infrastructure

```bash
# สร้าง Virtual Network
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --address-prefix 10.0.0.0/8 \
  --location $LOCATION

# สร้าง AKS Subnet
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name aksSubnet \
  --address-prefix 10.240.0.0/16

# สร้าง App Gateway Subnet
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name appGwSubnet \
  --address-prefix 10.225.0.0/16

# รับ Subnet ID
AKS_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name aksSubnet \
  --query id -o tsv)

# สร้าง ACR
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Premium \
  --location $LOCATION

# สร้าง Key Vault
az keyvault create \
  --resource-group $RESOURCE_GROUP \
  --name $KEYVAULT_NAME \
  --location $LOCATION \
  --enable-rbac-authorization

# สร้าง Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name myLogAnalytics \
  --location $LOCATION

WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name myLogAnalytics \
  --query id -o tsv)
```

### Step 3: สร้าง AKS Cluster

```bash
# สร้าง Azure AD Group สำหรับ Admin
az ad group create \
  --display-name "AKS-Admins" \
  --mail-nickname "AKS-Admins"

ADMIN_GROUP_ID=$(az ad group show \
  --group "AKS-Admins" \
  --query id -o tsv)

# สร้าง AKS Cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --min-count 3 \
  --max-count 10 \
  --enable-cluster-autoscaler \
  --network-plugin azure \
  --vnet-subnet-id $AKS_SUBNET_ID \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids $ADMIN_GROUP_ID \
  --enable-azure-rbac \
  --uptime-sla \
  --zones 1 2 3 \
  --generate-ssh-keys \
  --tier standard \
  --enable-addons monitoring,azure-policy,azure-keyvault-secrets-provider \
  --workspace-resource-id $WORKSPACE_ID \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --network-policy calico \
  --attach-acr $ACR_NAME

echo "AKS Cluster created successfully!"
```

### Step 4: ตั้งค่า Node Pools

```bash
# Spot Node Pool สำหรับ Non-critical Workloads
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name spotpool \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 10 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule \
  --node-labels workload-type=spot

# High Memory Node Pool
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name highmem \
  --node-count 1 \
  --node-vm-size Standard_E8s_v3 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 5 \
  --node-taints workload=highmem:NoSchedule \
  --node-labels workload-type=highmem
```

### Step 5: Deploy Application

```bash
# รับ Credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME

# ตรวจสอบ Nodes
kubectl get nodes -o wide

# สร้าง Namespace
kubectl create namespace production
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  environment=production

# Deploy Application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      serviceAccountName: default
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: ${ACR_NAME}.azurecr.io/myapp:v1
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["production-app"]
            topologyKey: kubernetes.io/hostname
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: workload-type
                operator: NotIn
                values: ["spot"]
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: production-app
EOF
```

### Step 6: ตั้งค่า Autoscaling

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: production-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-app
  minReplicas: 3
  maxReplicas: 30
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

### Step 7: ตรวจสอบและทดสอบ

```bash
# ตรวจสอบ Cluster Health
kubectl get nodes
kubectl get pods -A
kubectl top nodes
kubectl top pods -A

# ทดสอบ Autoscaling
kubectl run load-test --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://production-app-service.production.svc.cluster.local/; done"

# ดู HPA
kubectl get hpa -n production -w

# ทดสอบ Node Autoscaler
kubectl scale deployment production-app -n production --replicas=50
kubectl get nodes -w

# ทำความสะอาด
kubectl delete pod load-test
kubectl scale deployment production-app -n production --replicas=3
```

### Step 8: ตั้งค่า Monitoring Alerts

```bash
# สร้าง Alert Rule สำหรับ Node CPU
az monitor metrics alert create \
  --resource-group $RESOURCE_GROUP \
  --name "AKS-NodeCPUHigh" \
  --scopes "/subscriptions/sub-id/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "Node CPU usage is above 80%"

# สร้าง Action Group
az monitor action-group create \
  --resource-group $RESOURCE_GROUP \
  --name "AKS-Ops-Team" \
  --short-name "AKSOps" \
  --email-receivers name=OpsTeam email=ops@example.com

echo "Production AKS Setup Complete!"
```

### Workshop Verification Checklist

```bash
# Checklist ตรวจสอบ Production Setup
echo "=== AKS Production Verification ==="

# 1. Check Cluster
echo "1. Cluster Status:"
az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME \
  --query "{provisioningState:provisioningState, kubernetesVersion:kubernetesVersion}" \
  -o table

# 2. Check Nodes
echo "2. Node Status:"
kubectl get nodes -o wide

# 3. Check System Pods
echo "3. System Pods:"
kubectl get pods -n kube-system

# 4. Check Add-ons
echo "4. Add-ons:"
az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME \
  --query "addonProfiles" -o json

# 5. Check Network Policy
echo "5. Network Policies:"
kubectl get networkpolicies -A

# 6. Check RBAC
echo "6. RBAC Bindings:"
kubectl get clusterrolebindings | grep -E "aks|azure"

# 7. Check Autoscaler
echo "7. Cluster Autoscaler:"
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# 8. Check Monitoring
echo "8. Monitoring (Container Insights):"
kubectl get pods -n kube-system | grep omsagent
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **AKS Architecture** - การทำงานของ AKS และ Components ต่างๆ
2. **การติดตั้ง AKS** - ทั้งแบบ Basic และ Production Grade
3. **Azure AD Integration** - Workload Identity และ Azure RBAC
4. **AKS Features** - ACR, Autoscaler, Virtual Nodes, KEDA, Key Vault
5. **Networking** - Azure CNI, Network Policies, Ingress Controllers
6. **Storage** - Azure Disk, Files, Blob Storage
7. **Monitoring** - Container Insights, Prometheus, Grafana
8. **Security** - Pod Security Standards, Image Scanning, Private Cluster
9. **Workshop** - Production AKS Setup แบบครบถ้วน

## แบบฝึกหัด

1. สร้าง AKS Cluster ด้วย Availability Zones
2. ตั้งค่า Workload Identity สำหรับ Application ที่ต้องการเข้าถึง Azure Storage
3. Deploy Application ด้วย KEDA Autoscaling โดยใช้ Azure Service Bus Queue
4. ตั้งค่า Private AKS Cluster พร้อม Private Endpoint สำหรับ ACR
5. Implement Zero-trust Security ด้วย Network Policies

## References

- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- [AKS Security](https://docs.microsoft.com/en-us/azure/aks/concepts-security)
- [Azure RBAC for Kubernetes](https://docs.microsoft.com/en-us/azure/aks/manage-azure-rbac)
