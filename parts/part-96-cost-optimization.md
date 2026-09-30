# Part 96: Kubernetes Cost Optimization

## บทนำ

การ Optimize ค่าใช้จ่ายใน Kubernetes เป็นสิ่งสำคัญที่ทุกองค์กรควรให้ความสำคัญ บทนี้ครอบคลุมกลยุทธ์และ Tools ที่ช่วยลดค่าใช้จ่าย Cloud โดยไม่กระทบต่อ Performance และ Reliability

## สารบัญ

1. Cost Visibility
2. Resource Right-sizing
3. Spot/Preemptible Instances
4. Cluster Autoscaler
5. Vertical Pod Autoscaler (VPA)
6. Namespace Resource Quotas
7. Cost Allocation
8. Workshop: Reduce Cloud Costs

---

## 1. Cost Visibility

### 1.1 Kubecost Installation

```bash
# ติดตั้ง Kubecost
helm repo add cost-analyzer https://kubecost.github.io/cost-analyzer/
helm repo update

helm install kubecost cost-analyzer/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="your-token" \
  --set prometheus.server.persistentVolume.storageClass=fast-ssd \
  --set prometheus.server.persistentVolume.size=50Gi

# เข้าถึง Dashboard
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090
```

### 1.2 ดู Cost Reports

```bash
# ดู Cost per Namespace
kubectl cost namespace \
  --show-all-resources \
  --historical \
  --window 1d

# ดู Cost per Deployment
kubectl cost deployment \
  --namespace production \
  --historical \
  --window 7d

# ดู Cost per Label
kubectl cost label \
  --historical \
  --window 30d \
  -l team=backend

# Export Cost Data
kubectl cost namespace --window 30d -o json > /tmp/cost-report.json
```

### 1.3 Prometheus Cost Metrics

```yaml
# cost-metrics-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-metrics
  namespace: monitoring
spec:
  groups:
  - name: cost
    interval: 1h
    rules:
    # CPU Cost per Pod per Hour
    - record: pod:cpu_cost_hourly
      expr: |
        sum(
          rate(container_cpu_usage_seconds_total{container!=""}[1h])
          * on(node) group_left()
          node_cpu_hourly_cost
        ) by (pod, namespace, node)

    # Memory Cost per Pod per Hour
    - record: pod:memory_cost_hourly
      expr: |
        sum(
          container_memory_usage_bytes{container!=""}
          * on(node) group_left()
          node_memory_hourly_cost / 1024 / 1024 / 1024
        ) by (pod, namespace, node)

    # Total Cost per Namespace per Day
    - record: namespace:total_cost_daily
      expr: |
        sum(
          pod:cpu_cost_hourly * 24 +
          pod:memory_cost_hourly * 24
        ) by (namespace)
```

---

## 2. Resource Right-sizing

### 2.1 VPA (Vertical Pod Autoscaler) Installation

```bash
# ติดตั้ง VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# ตรวจสอบ
kubectl get pods -n kube-system | grep vpa
```

### 2.2 VPA Configuration

```yaml
# vpa-examples.yaml
# Mode: Recommendation Only (ไม่แก้อัตโนมัติ)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"  # แค่ Recommendation
  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2000m"
        memory: "4Gi"
      controlledResources:
      - cpu
      - memory
---
# Mode: Auto (แก้อัตโนมัติ พร้อม restart)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: worker-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: worker
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: worker
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "4000m"
        memory: "8Gi"
---
# Mode: Initial (ตั้งค่าตอน Pod สร้างใหม่)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: batch-job-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: batch-job
  updatePolicy:
    updateMode: "Initial"
```

### 2.3 ดู VPA Recommendations

```bash
# ดู VPA Recommendations
kubectl get vpa -n production -o yaml | \
  yq '.status.recommendation.containerRecommendations'

# Script: แสดง Recommendations สำหรับทุก Deployments
#!/bin/bash
echo "=== VPA Recommendations ==="
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  for vpa in $(kubectl get vpa -n $ns -o name 2>/dev/null | cut -d/ -f2); do
    echo "Namespace: $ns, VPA: $vpa"
    kubectl get vpa $vpa -n $ns -o json | \
      jq -r '.status.recommendation.containerRecommendations[] | 
        "Container: \(.containerName)\n  Lower: cpu=\(.lowerBound.cpu), mem=\(.lowerBound.memory)\n  Target: cpu=\(.target.cpu), mem=\(.target.memory)\n  Upper: cpu=\(.upperBound.cpu), mem=\(.upperBound.memory)"'
    echo ""
  done
done
```

### 2.4 Resource Analysis Script

```bash
#!/bin/bash
# analyze-resources.sh - วิเคราะห์ Resource Usage

echo "=== Resource Usage Analysis ==="

echo "--- Pods with no Resource Requests ---"
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers[].resources.requests == null or .spec.containers[].resources.requests == {}) |
  "\(.metadata.namespace)/\(.metadata.name)"
'

echo ""
echo "--- Pods with CPU Request > Current Usage ---"
# ต้องการ metrics-server
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  kubectl top pods -n $ns --no-headers 2>/dev/null | while read name cpu mem; do
    REQUESTED=$(kubectl get pod $name -n $ns -o jsonpath='{.spec.containers[0].resources.requests.cpu}' 2>/dev/null)
    if [ -n "$REQUESTED" ]; then
      echo "Pod: $ns/$name | Requested: $REQUESTED | Using: ${cpu}"
    fi
  done
done

echo ""
echo "--- Namespaces without Resource Quotas ---"
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  if ! kubectl get resourcequota -n $ns --no-headers 2>/dev/null | grep -q .; then
    echo "$ns"
  fi
done

echo ""
echo "--- Unused PVCs ---"
kubectl get pvc -A --no-headers | while read ns name status vol capacity access class age; do
  if [ "$status" = "Bound" ]; then
    # ตรวจสอบว่ามี Pod ใช้ PVC นี้หรือไม่
    PODS=$(kubectl get pods -n $ns -o json | jq -r --arg PVC "$name" '
      .items[] |
      select(.spec.volumes[]?.persistentVolumeClaim.claimName == $PVC) |
      .metadata.name
    ')
    if [ -z "$PODS" ]; then
      echo "$ns/$name ($capacity) - No pods using this PVC!"
    fi
  fi
done
```

---

## 3. Spot/Preemptible Instances

### 3.1 AWS Spot Instances Configuration

```bash
# สร้าง Spot Node Group ใน EKS
eksctl create nodegroup \
  --cluster my-cluster \
  --name spot-workers \
  --node-type-filter m5.xlarge,m5.2xlarge,m4.xlarge,m4.2xlarge \
  --managed \
  --spot \
  --nodes-min 2 \
  --nodes-max 20 \
  --asg-access \
  --alb-ingress-access \
  --node-labels="workload-type=spot"

# หรือ ใช้ eksctl config file
cat <<'EOF' > spot-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: ap-southeast-1

managedNodeGroups:
- name: spot-workers
  instanceTypes:
  - m5.xlarge
  - m5.2xlarge
  - m4.xlarge
  - m4.2xlarge
  - m5a.xlarge
  - m5a.2xlarge
  spot: true
  minSize: 2
  maxSize: 20
  labels:
    workload-type: spot
    lifecycle: spot
  taints:
  - key: workload-type
    value: spot
    effect: NoSchedule
  tags:
    cost-center: "production"
    workload: "spot"
EOF
```

### 3.2 Spot Instance Handling

```yaml
# spot-tolerant-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spot-worker
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: spot-worker
  template:
    metadata:
      labels:
        app: spot-worker
    spec:
      # ยอมรับ Spot Nodes
      tolerations:
      - key: workload-type
        value: spot
        effect: NoSchedule
      # prefer spot แต่ไม่บังคับ
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: workload-type
                operator: In
                values: ["spot"]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["spot-worker"]
              topologyKey: kubernetes.io/hostname
      # Graceful termination (Spot interrupt warning = 2 minutes)
      terminationGracePeriodSeconds: 90
      containers:
      - name: worker
        image: worker:latest
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Save state before shutdown
                /app/save-state.sh
                # Wait for current task to complete (max 80 seconds)
                sleep 80
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
```

### 3.3 Spot Interrupt Handler

```bash
# ติดตั้ง AWS Node Termination Handler
helm repo add eks https://aws.github.io/eks-charts
helm install aws-node-termination-handler eks/aws-node-termination-handler \
  --namespace kube-system \
  --set enableSpotInterruptionDraining=true \
  --set enableScheduledEventDraining=true \
  --set enableRebalanceMonitoring=true \
  --set enableRebalanceDraining=true \
  --set podTerminationGracePeriod=120 \
  --set nodeTerminationGracePeriod=180 \
  --set emitKubernetesEvents=true \
  --set enableSqsTerminationDraining=true \
  --set queueURL="https://sqs.ap-southeast-1.amazonaws.com/123456789/my-node-termination-queue"
```

### 3.4 GCP Preemptible VMs

```bash
# สร้าง Preemptible Node Pool ใน GKE
gcloud container node-pools create preemptible-pool \
  --cluster my-cluster \
  --num-nodes 5 \
  --machine-type n1-standard-4 \
  --preemptible \
  --node-labels workload-type=preemptible \
  --node-taints workload-type=preemptible:NoSchedule \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 20

# หรือใช้ Spot VMs (ใหม่กว่า Preemptible)
gcloud container node-pools create spot-pool \
  --cluster my-cluster \
  --num-nodes 5 \
  --machine-type n2-standard-4 \
  --spot \
  --node-labels workload-type=spot \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 30
```

---

## 4. Cluster Autoscaler

### 4.1 AWS EKS Cluster Autoscaler

```bash
# สร้าง IAM Policy สำหรับ Cluster Autoscaler
cat <<'EOF' > cluster-autoscaler-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:DescribeTags",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeImages",
                "ec2:GetInstanceTypesFromInstanceRequirements",
                "eks:DescribeNodegroup"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EOF

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json

# ติดตั้ง Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=ap-southeast-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789:role/eks-cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.skip-nodes-with-local-storage=false \
  --set extraArgs.scale-down-utilization-threshold=0.5 \
  --set extraArgs.scale-down-delay-after-add=10m \
  --set extraArgs.scale-down-unneeded-time=10m \
  --set extraArgs.max-graceful-termination-sec=600
```

### 4.2 Cluster Autoscaler Tuning

```yaml
# cluster-autoscaler-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
data:
  nodes.min: "3"
  nodes.max: "50"
  scale-down-enabled: "true"
  scale-down-delay-after-add: "10m"
  scale-down-delay-after-delete: "0s"
  scale-down-delay-after-failure: "3m"
  scale-down-unneeded-time: "10m"
  scale-down-utilization-threshold: "0.5"
  max-graceful-termination-sec: "600"
  balance-similar-node-groups: "true"
  expander: "least-waste"  # หรือ priority, random
```

### 4.3 Custom Expander สำหรับ Cost Optimization

```yaml
# priority-expander-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |
    100:
    - .*spot.*
    - .*preemptible.*
    50:
    - .*on-demand-medium.*
    10:
    - .*on-demand-large.*
    1:
    - .*on-demand-xlarge.*
```

---

## 5. Resource Quotas และ LimitRange

### 5.1 Namespace Resource Quotas

```yaml
# namespace-quotas.yaml
# Production Namespace - Conservative limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"        # รวม CPU requests สูงสุด
    requests.memory: "200Gi"   # รวม Memory requests สูงสุด
    limits.cpu: "200"          # รวม CPU limits สูงสุด
    limits.memory: "400Gi"     # รวม Memory limits สูงสุด
    requests.storage: "2Ti"    # รวม Storage requests สูงสุด
    count/pods: "500"          # จำนวน Pods สูงสุด
    count/services: "100"      # จำนวน Services สูงสุด
    count/secrets: "500"       # จำนวน Secrets สูงสุด
    count/configmaps: "500"    # จำนวน ConfigMaps สูงสุด
    count/persistentvolumeclaims: "100"
    services.loadbalancers: "5"   # จำนวน LoadBalancers สูงสุด
    services.nodeports: "0"       # ห้ามใช้ NodePort
---
# Development Namespace - Limited resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    count/pods: "100"
    services.loadbalancers: "1"
---
# LimitRange สำหรับ Default Limits
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4000m"
      memory: "8Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "16"
      memory: "32Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "500Gi"
    min:
      storage: "1Gi"
```

---

## 6. Cost Allocation

### 6.1 Cost Tagging Strategy

```yaml
# cost-labels-template.yaml
# ทุก Resource ควรมี Labels เหล่านี้สำหรับ Cost Tracking
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    # Cost Allocation Labels
    team: "platform"           # ทีมที่รับผิดชอบ
    project: "ecommerce"       # ชื่อ Project
    cost-center: "CC-001"      # Cost Center
    environment: "production"  # Environment
    service: "api"             # ชื่อ Service
    version: "v2.1.0"         # Version
```

### 6.2 Chargeback Model

```bash
#!/bin/bash
# cost-chargeback.sh - คำนวณ Cost per Team

echo "=== Monthly Cost Chargeback Report ==="
echo "Month: $(date +%B %Y)"
echo ""

# ดึง Cost Data จาก Kubecost
KUBECOST_URL="http://kubecost-cost-analyzer.kubecost:9090"

# รับ Cost per Namespace
curl -s "${KUBECOST_URL}/model/aggregatedCostModel?window=month&aggregation=namespace" | \
  jq -r '.data | to_entries[] | "\(.key): $\(.value.totalCost | . * 100 | round / 100)"' | \
  sort -t'$' -k2 -rn

echo ""
echo "=== Cost per Team ==="
# จัดกลุ่มตาม Team Label
curl -s "${KUBECOST_URL}/model/aggregatedCostModel?window=month&aggregation=label&labelName=team" | \
  jq -r '.data | to_entries[] | "\(.key): $\(.value.totalCost | . * 100 | round / 100)"' | \
  sort -t'$' -k2 -rn

echo ""
echo "=== Top 10 Most Expensive Pods ==="
curl -s "${KUBECOST_URL}/model/topResults?window=month&n=10" | \
  jq -r '.data[] | "\(.namespace)/\(.pod): $\(.totalCost | . * 100 | round / 100)"'
```

---

## 7. Additional Cost Saving Strategies

### 7.1 Scheduled Scaling

```yaml
# scheduled-scaler.yaml
# ลด Replicas ในเวลากลางคืน
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-nights
  namespace: production
spec:
  schedule: "0 22 * * 1-5"  # 22:00 จันทร์-ศุกร์
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment api --replicas=2 -n production
              kubectl scale deployment worker --replicas=1 -n production
          restartPolicy: OnFailure
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-morning
  namespace: production
spec:
  schedule: "0 8 * * 1-5"  # 08:00 จันทร์-ศุกร์
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment api --replicas=10 -n production
              kubectl scale deployment worker --replicas=5 -n production
          restartPolicy: OnFailure
```

### 7.2 Node Consolidation

```bash
# ดู Node Utilization
kubectl top nodes --sort-by=cpu

# ปิด Unused Nodes โดยใช้ Cluster Autoscaler
# scale-down จะทำเองถ้าตั้งค่าถูกต้อง

# Manual Drain Node ก่อน Terminate
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=300
```

### 7.3 Cost Alerts

```yaml
# cost-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-alerts
  namespace: monitoring
spec:
  groups:
  - name: cost
    rules:
    - alert: HighCostNamespace
      expr: |
        sum(kubecost_cluster_management_cost) by (namespace) > 500
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Namespace {{ $labels.namespace }} cost is high"
        description: "Monthly projected cost exceeds $500"

    - alert: UnusedPVC
      expr: |
        kube_persistentvolumeclaim_info{phase="Bound"} 
        unless on(namespace, persistentvolumeclaim) 
        kube_pod_spec_volumes_persistentvolumeclaims_info
      for: 24h
      labels:
        severity: info
      annotations:
        summary: "Unused PVC detected"
        description: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} appears unused"
```

---

## 8. Workshop: Reduce Cloud Costs

### Workshop Overview

เป้าหมาย: ลดค่าใช้จ่าย Cloud 30-50% โดยไม่กระทบ Performance

### Step 1: Cost Audit

```bash
#!/bin/bash
# cost-audit.sh

echo "=== Kubernetes Cost Audit ==="
echo "Date: $(date)"
echo ""

echo "1. Resources without Requests/Limits:"
kubectl get pods -A -o json | jq -r '
  .items[] |
  .spec.containers[] as $c |
  select($c.resources.requests == null) |
  "\(.metadata.namespace)/\(.metadata.name)/\($c.name): No resource requests!"
'

echo ""
echo "2. Oversized Resources (using <10% of requests):"
# ต้องใช้ metrics
kubectl top pods -A --sort-by=cpu 2>/dev/null | awk 'NR>1 {
  print $1"/"$2, "CPU:", $3, "MEM:", $4
}'

echo ""
echo "3. Unused Services:"
kubectl get svc -A -o json | jq -r '
  .items[] |
  select(.spec.type == "LoadBalancer") |
  select(.status.loadBalancer.ingress == null) |
  "\(.metadata.namespace)/\(.metadata.name): LoadBalancer with no ingress!"
'

echo ""
echo "4. Old/Unused Images:"
kubectl get pods -A -o json | jq -r '
  [.items[].spec.containers[].image] | 
  group_by(.) | 
  map({image: .[0], count: length}) | 
  sort_by(.count) | 
  .[:10][] | 
  "\(.count)x \(.image)"
'

echo ""
echo "5. Namespaces without Quotas:"
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  if ! kubectl get resourcequota -n $ns --no-headers 2>/dev/null | grep -q .; then
    echo "No quota: $ns"
  fi
done
```

### Step 2: Implement VPA Recommendations

```bash
#!/bin/bash
# implement-vpa.sh

# ดู VPA Recommendations
echo "=== VPA Recommendations ==="

kubectl get vpa -A -o json | jq -r '
  .items[] |
  .metadata.name as $name |
  .metadata.namespace as $ns |
  .status.recommendation.containerRecommendations[]? |
  "VPA: \($ns)/\($name)\n  Container: \(.containerName)\n  Recommended CPU: \(.target.cpu)\n  Recommended Memory: \(.target.memory)\n"
'

# Apply Recommendations อัตโนมัติ (ระวัง!)
# สร้าง VPA สำหรับทุก Deployment
for ns in production staging; do
  for deployment in $(kubectl get deployments -n $ns -o name | cut -d/ -f2); do
    cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ${deployment}-vpa
  namespace: ${ns}
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: ${deployment}
  updatePolicy:
    updateMode: "Off"  # Recommendation only ก่อน
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
EOF
    echo "Created VPA for $ns/$deployment"
  done
done

echo "Wait 24 hours for VPA to collect data, then review recommendations"
```

### Step 3: Migrate to Spot Instances

```bash
#!/bin/bash
# migrate-to-spot.sh

echo "=== Migrating Non-critical Workloads to Spot ==="

# ระบุ Workloads ที่ Safe to run on Spot
SPOT_ELIGIBLE_LABELS="tier=batch,tier=worker,workload-type=background"

# เพิ่ม Spot Tolerations ให้ Deployments ที่เหมาะสม
for ns in production staging; do
  kubectl get deployments -n $ns -l "tier=batch" -o name | while read deploy; do
    DEPLOY_NAME=$(echo $deploy | cut -d/ -f2)
    
    echo "Adding spot tolerations to $ns/$DEPLOY_NAME"
    
    kubectl patch deployment $DEPLOY_NAME -n $ns --type=json -p='[
      {
        "op": "add",
        "path": "/spec/template/spec/tolerations",
        "value": [
          {
            "key": "workload-type",
            "value": "spot",
            "effect": "NoSchedule"
          }
        ]
      },
      {
        "op": "add",
        "path": "/spec/template/spec/affinity",
        "value": {
          "nodeAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [
              {
                "weight": 100,
                "preference": {
                  "matchExpressions": [
                    {
                      "key": "workload-type",
                      "operator": "In",
                      "values": ["spot"]
                    }
                  ]
                }
              }
            ]
          }
        }
      }
    ]'
  done
done

echo "Done! Monitor cost savings over next 24 hours."
```

### Step 4: Implement Scheduled Scaling

```bash
#!/bin/bash
# setup-scheduled-scaling.sh

# สร้าง ServiceAccount สำหรับ Scaler
kubectl create serviceaccount scaler-sa -n production
kubectl create clusterrolebinding scaler-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=production:scaler-sa

# Scale Down สำหรับ Non-Production Hours (เวลา 22:00 - 08:00)
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-evening
  namespace: production
spec:
  schedule: "0 22 * * *"  # 22:00 ทุกวัน (Bangkok time = UTC+7)
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
          - name: kubectl
            image: bitnami/kubectl:1.28
            command:
            - /bin/bash
            - -c
            - |
              set -e
              
              # บันทึก Replicas ปัจจุบัน
              kubectl get deployments -n production -o json | \
                jq '.items[] | {name: .metadata.name, replicas: .spec.replicas}' > /tmp/replicas-before.json
              
              # Scale Down
              echo "Scaling down non-critical services..."
              kubectl scale deployment \
                --selector=tier=worker \
                -n production \
                --replicas=1
              
              kubectl scale deployment \
                --selector=tier=batch \
                -n production \
                --replicas=0
              
              echo "Scale down complete at $(date)"
          restartPolicy: OnFailure
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-morning
  namespace: production
spec:
  schedule: "0 8 * * *"  # 08:00 ทุกวัน
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
          - name: kubectl
            image: bitnami/kubectl:1.28
            command:
            - /bin/bash
            - -c
            - |
              # Scale Up
              echo "Scaling up services..."
              kubectl scale deployment \
                --selector=tier=worker \
                -n production \
                --replicas=5
              
              kubectl scale deployment \
                --selector=tier=batch \
                -n production \
                --replicas=3
              
              echo "Scale up complete at $(date)"
          restartPolicy: OnFailure
EOF

echo "Scheduled scaling configured!"
```

### Step 5: Cost Savings Tracking

```bash
#!/bin/bash
# track-cost-savings.sh

echo "=== Cost Savings Report ==="
echo "Period: $(date -d '7 days ago' +%Y-%m-%d) to $(date +%Y-%m-%d)"
echo ""

# ดึง Cost จาก Kubecost
KUBECOST="http://kubecost-cost-analyzer.kubecost:9090"

# ก่อน Optimization (ใช้ Historical data)
BEFORE_COST=$(curl -s "${KUBECOST}/model/aggregatedCostModel?window=7d&offset=7d" | \
  jq -r '.data | to_entries[] | .value.totalCost' | \
  awk '{sum+=$1} END {print sum}')

# หลัง Optimization
AFTER_COST=$(curl -s "${KUBECOST}/model/aggregatedCostModel?window=7d" | \
  jq -r '.data | to_entries[] | .value.totalCost' | \
  awk '{sum+=$1} END {print sum}')

echo "Before Optimization: \$$BEFORE_COST"
echo "After Optimization: \$$AFTER_COST"

SAVINGS=$(echo "$BEFORE_COST $AFTER_COST" | awk '{printf "%.2f", $1 - $2}')
SAVINGS_PCT=$(echo "$BEFORE_COST $AFTER_COST" | awk '{printf "%.1f", (($1-$2)/$1)*100}')

echo "Weekly Savings: \$$SAVINGS ($SAVINGS_PCT%)"
echo "Monthly Projected Savings: \$$(echo $SAVINGS | awk '{printf "%.2f", $1*4.33}')"

echo ""
echo "=== Recommendations for Further Savings ==="
echo "1. Review VPA recommendations after 24h data collection"
echo "2. Consider Reserved Instances for stable workloads"
echo "3. Implement cross-team cost awareness training"
echo "4. Review and cleanup unused resources monthly"
```

### Workshop Summary

```
Cost Optimization Results (Expected):

Strategy              | Estimated Savings
----------------------|------------------
Spot Instances        | 60-70% on batch workloads
VPA Right-sizing      | 15-30% on all workloads
Scheduled Scaling     | 30-50% on non-prod hours
Unused Resource Cleanup | 5-15% overall
Total Expected        | 30-50% overall
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Cost Visibility** - Kubecost, Prometheus metrics
2. **Right-sizing** - VPA recommendations
3. **Spot Instances** - AWS Spot, GCP Preemptible
4. **Cluster Autoscaler** - Auto scale ตาม demand
5. **Resource Quotas** - จำกัด resource per namespace
6. **Cost Allocation** - Tagging, Chargeback
7. **Scheduled Scaling** - ลด cost ช่วง off-peak
8. **Workshop** - Complete cost reduction workflow

## แบบฝึกหัด

1. ติดตั้ง Kubecost และวิเคราะห์ Cost per Namespace
2. Enable VPA สำหรับ 3 Deployments และดู Recommendations หลัง 24 ชั่วโมง
3. สร้าง Spot Node Pool และ migrate batch workloads
4. ตั้งค่า Scheduled Scaling สำหรับ Development Environment
5. สร้าง Cost Alert เมื่อ Monthly Cost เกิน $1000

## References

- [Kubecost Documentation](https://docs.kubecost.com/)
- [Cluster Autoscaler Documentation](https://github.com/kubernetes/autoscaler)
- [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [AWS Spot Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)
