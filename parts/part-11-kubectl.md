# Part 11: kubectl - เครื่องมือสำคัญในการจัดการ Kubernetes

## สารบัญ
1. [kubectl คืออะไร](#kubectl-คืออะไร)
2. [การติดตั้ง kubectl](#การติดตั้ง-kubectl)
3. [kubectl Syntax และ Commands ทั้งหมด](#kubectl-syntax-และ-commands-ทั้งหมด)
4. [kubectl Config และ Contexts](#kubectl-config-และ-contexts)
5. [kubectl Plugins](#kubectl-plugins)
6. [Workshop: kubectl Cheatsheet แบบใช้งานจริง](#workshop-kubectl-cheatsheet)

---

## 1. kubectl คืออะไร

`kubectl` (อ่านว่า "kube-control" หรือ "kube-c-t-l") คือ Command-Line Interface (CLI) หลักที่ใช้ในการสื่อสารกับ Kubernetes Cluster โดยผ่าน Kubernetes API Server

### หน้าที่ของ kubectl

kubectl ทำหน้าที่เป็นตัวกลางระหว่าง Developer/Administrator กับ Kubernetes Cluster:

```
┌─────────────┐    HTTP/HTTPS    ┌────────────────┐    ┌──────────────────┐
│   Developer  │ ──────────────► │   API Server   │ ──► │  Kubernetes      │
│  (kubectl)   │                 │  (kube-apiserver)│    │  Components      │
└─────────────┘                  └────────────────┘    └──────────────────┘
```

### สิ่งที่ kubectl ทำได้

- **สร้าง** (Create): สร้าง Resources ใหม่ เช่น Pod, Deployment, Service
- **อ่าน** (Read): ดูข้อมูล Resources ที่มีอยู่
- **อัปเดต** (Update): แก้ไข Resources ที่มีอยู่
- **ลบ** (Delete): ลบ Resources ออกจาก Cluster
- **ดู Logs**: ดู logs จาก Pods
- **Execute**: รันคำสั่งภายใน Containers
- **Port Forwarding**: ส่งต่อ traffic จาก local ไปยัง Cluster
- **Copy Files**: คัดลอกไฟล์ระหว่าง local กับ Container

### kubectl Architecture

```
kubectl
├── Configuration (~/.kube/config)
│   ├── Clusters (เก็บข้อมูล cluster ต่างๆ)
│   ├── Users (credentials สำหรับ authentication)
│   └── Contexts (การจับคู่ cluster + user + namespace)
│
├── API Groups
│   ├── core (v1): pods, services, nodes, namespaces
│   ├── apps: deployments, replicasets, statefulsets
│   ├── batch: jobs, cronjobs
│   └── networking.k8s.io: ingress, networkpolicies
│
└── Output Formats
    ├── table (default)
    ├── yaml
    ├── json
    └── jsonpath
```

---

## 2. การติดตั้ง kubectl

### ติดตั้งบน Linux

#### วิธีที่ 1: ดาวน์โหลด Binary โดยตรง

```bash
# ดาวน์โหลด kubectl เวอร์ชันล่าสุด
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# ตรวจสอบ checksum (แนะนำ)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# ติดตั้ง
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# ตรวจสอบเวอร์ชัน
kubectl version --client
```

#### วิธีที่ 2: ติดตั้งผ่าน Package Manager (Ubuntu/Debian)

```bash
# อัปเดต apt และติดตั้ง dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# เพิ่ม Google Cloud public signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# เพิ่ม Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
    sudo tee /etc/apt/sources.list.d/kubernetes.list

# ติดตั้ง kubectl
sudo apt-get update
sudo apt-get install -y kubectl

# ตรวจสอบ
kubectl version --client
```

#### วิธีที่ 3: ติดตั้งผ่าน Package Manager (CentOS/RHEL/Fedora)

```bash
# สร้าง repo file
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF

# ติดตั้ง
sudo yum install -y kubectl

# หรือสำหรับ Fedora
sudo dnf install -y kubectl
```

### ติดตั้งบน macOS

```bash
# ใช้ Homebrew (แนะนำ)
brew install kubectl

# ตรวจสอบเวอร์ชัน
kubectl version --client

# อัปเดต kubectl
brew upgrade kubectl
```

### ติดตั้งบน Windows

```powershell
# ใช้ Chocolatey
choco install kubernetes-cli

# หรือใช้ Scoop
scoop install kubectl

# ตรวจสอบเวอร์ชัน
kubectl version --client
```

### ตั้งค่า Shell Completion

```bash
# สำหรับ Bash (Linux)
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc

# สำหรับ Zsh
echo 'source <(kubectl completion zsh)' >>~/.zshrc
source ~/.zshrc

# สำหรับ Fish
kubectl completion fish | source

# เพิ่ม alias สำหรับ kubectl
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

---

## 3. kubectl Syntax และ Commands ทั้งหมด

### Syntax พื้นฐาน

```
kubectl [command] [TYPE] [NAME] [flags]
```

- **command**: การกระทำที่ต้องการ (get, create, apply, delete, etc.)
- **TYPE**: ประเภทของ Resource (pod, service, deployment, etc.)
- **NAME**: ชื่อของ Resource (ถ้าไม่ระบุ จะแสดงทั้งหมด)
- **flags**: ตัวเลือกเพิ่มเติม (-n namespace, -o yaml, etc.)

### Commands หลัก

#### 3.1 Basic Commands (Beginner)

```bash
# create - สร้าง Resource จากไฟล์หรือ stdin
kubectl create -f pod.yaml
kubectl create deployment nginx --image=nginx
kubectl create namespace my-namespace
kubectl create service nodeport my-service --tcp=8080:80

# run - สร้าง Pod จาก Image
kubectl run nginx --image=nginx
kubectl run redis --image=redis --port=6379
kubectl run busybox --image=busybox -- sleep 3600

# expose - สร้าง Service จาก Resource
kubectl expose pod nginx --port=80
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment app --port=8080 --target-port=3000 --type=LoadBalancer

# delete - ลบ Resource
kubectl delete pod nginx
kubectl delete -f pod.yaml
kubectl delete pods --all
kubectl delete deployment,service app

# get - แสดงข้อมูล Resources
kubectl get pods
kubectl get pods -A               # ทุก namespaces
kubectl get pods -n kube-system   # namespace เฉพาะ
kubectl get pods -o wide          # แสดงข้อมูลเพิ่มเติม
kubectl get pods -o yaml          # แสดงเป็น YAML
kubectl get pods -o json          # แสดงเป็น JSON
kubectl get pods --watch          # watch การเปลี่ยนแปลง
kubectl get all                   # แสดงทุก Resources
kubectl get all -n default        # ทุก Resources ใน namespace
```

#### 3.2 Deploy Commands (Intermediate)

```bash
# rollout - จัดการ Deployment rollouts
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
kubectl rollout restart deployment/nginx

# scale - ปรับขนาด Replicas
kubectl scale deployment nginx --replicas=3
kubectl scale deployment nginx --replicas=0  # หยุดชั่วคราว
kubectl scale replicaset my-rs --replicas=5

# autoscale - ตั้งค่า Horizontal Pod Autoscaler
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# set - อัปเดต Resource
kubectl set image deployment/nginx nginx=nginx:1.24
kubectl set resources deployment nginx -c=nginx --limits=cpu=200m,memory=512Mi
kubectl set env deployment/app KEY=VALUE
kubectl set serviceaccount deployment nginx my-sa
```

#### 3.3 Cluster Management Commands

```bash
# cluster-info - แสดงข้อมูล Cluster
kubectl cluster-info
kubectl cluster-info dump  # dump ข้อมูลทั้งหมดสำหรับ debug

# top - แสดงการใช้ Resources (ต้องติดตั้ง metrics-server)
kubectl top pods
kubectl top pods -n kube-system
kubectl top nodes

# cordon - ป้องกันไม่ให้ schedule Pods ใหม่บน Node
kubectl cordon node-1

# uncordon - ยกเลิก cordon
kubectl uncordon node-1

# drain - ย้าย Pods ออกจาก Node (สำหรับ maintenance)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl drain node-1 --grace-period=300

# taint - เพิ่ม/ลบ Taint บน Node
kubectl taint nodes node-1 key=value:NoSchedule
kubectl taint nodes node-1 key=value:NoSchedule-  # ลบ taint

# label - จัดการ Labels
kubectl label nodes node-1 disktype=ssd
kubectl label pods my-pod env=production
kubectl label pods my-pod env-  # ลบ label

# annotate - จัดการ Annotations
kubectl annotate pods my-pod description="my application pod"
kubectl annotate pods my-pod description-  # ลบ annotation
```

#### 3.4 Troubleshooting and Debugging Commands

```bash
# describe - แสดงรายละเอียด Resource
kubectl describe pod nginx
kubectl describe node worker-1
kubectl describe service my-service
kubectl describe deployment my-app

# logs - แสดง logs
kubectl logs nginx
kubectl logs nginx -f                    # follow logs
kubectl logs nginx --previous            # logs ของ container ก่อนหน้า
kubectl logs nginx -c sidecar            # logs ของ specific container
kubectl logs nginx --tail=100            # แสดง 100 บรรทัดล่าสุด
kubectl logs nginx --since=1h            # logs ย้อนหลัง 1 ชั่วโมง
kubectl logs -l app=nginx                # logs จาก pods ที่มี label
kubectl logs nginx --timestamps          # แสดง timestamps

# exec - รันคำสั่งใน Container
kubectl exec nginx -- ls /
kubectl exec nginx -it -- /bin/bash      # interactive shell
kubectl exec nginx -c sidecar -- ps aux  # รันใน specific container
kubectl exec -it nginx -- env            # แสดง environment variables

# port-forward - Forward port จาก local ไปยัง Pod
kubectl port-forward pod/nginx 8080:80
kubectl port-forward service/nginx 8080:80
kubectl port-forward deployment/nginx 8080:80

# cp - คัดลอกไฟล์
kubectl cp /local/path nginx:/container/path
kubectl cp nginx:/container/path /local/path
kubectl cp /local/path nginx:/container/path -c sidecar

# attach - Attach ไปยัง Running Container
kubectl attach nginx -it

# proxy - เปิด proxy ไปยัง API server
kubectl proxy
kubectl proxy --port=8001
```

#### 3.5 Advanced Commands

```bash
# apply - Apply configuration (idempotent)
kubectl apply -f pod.yaml
kubectl apply -f ./directory/
kubectl apply -f https://url/manifest.yaml
kubectl apply -k ./kustomization/  # kustomize

# diff - แสดงความแตกต่างก่อน apply
kubectl diff -f pod.yaml

# patch - อัปเดต Resource บางส่วน
kubectl patch pod nginx -p '{"spec":{"containers":[{"name":"nginx","image":"nginx:1.24"}]}}'
kubectl patch deployment nginx --type=json \
    -p='[{"op": "replace", "path": "/spec/replicas", "value": 3}]'

# replace - แทนที่ Resource ทั้งหมด
kubectl replace -f pod.yaml
kubectl replace --force -f pod.yaml  # ลบแล้วสร้างใหม่

# wait - รอ Condition
kubectl wait --for=condition=Ready pod/nginx --timeout=60s
kubectl wait --for=condition=Available deployment/nginx --timeout=120s
kubectl wait --for=delete pod/nginx --timeout=30s

# auth - ตรวจสอบ Authorization
kubectl auth can-i create pods
kubectl auth can-i delete pods --as=john
kubectl auth can-i '*' '*' --all-namespaces
kubectl auth whoami

# api-resources - แสดง API Resources ทั้งหมด
kubectl api-resources
kubectl api-resources --namespaced=true
kubectl api-resources --api-group=apps

# api-versions - แสดง API Versions ทั้งหมด
kubectl api-versions

# explain - แสดงคำอธิบาย Resource fields
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.template.spec.containers.resources
```

#### 3.6 Output Formatting

```bash
# Output formats
kubectl get pods -o wide         # ข้อมูลเพิ่มเติม (IP, Node)
kubectl get pods -o yaml         # YAML format
kubectl get pods -o json         # JSON format
kubectl get pods -o name         # เฉพาะชื่อ
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# Sorting
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.phase

# Filtering
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=spec.nodeName=worker-1
kubectl get events --field-selector=type=Warning
```

---

## 4. kubectl Config และ Contexts

### kubeconfig ไฟล์

kubeconfig ไฟล์ (ปกติอยู่ที่ `~/.kube/config`) เก็บข้อมูลการเชื่อมต่อกับ Kubernetes Clusters

```yaml
# ตัวอย่าง ~/.kube/config
apiVersion: v1
kind: Config
preferences: {}

# รายการ Clusters ที่เชื่อมต่อได้
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...  # CA certificate (base64)
    server: https://10.0.0.10:6443                  # API Server URL
  name: production-cluster
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...
    server: https://10.0.0.20:6443
  name: staging-cluster
- cluster:
    server: https://127.0.0.1:6443
    certificate-authority: /path/to/ca.crt          # หรือระบุ path ไปยัง CA file
  name: local-cluster

# รายการ Users (credentials)
users:
- name: admin-user
  user:
    client-certificate-data: LS0tLS1CRUdJTi...      # Client cert (base64)
    client-key-data: LS0tLS1CRUdJTi...              # Client key (base64)
- name: developer-user
  user:
    token: eyJhbGciOiJSUzI1NiIs...                  # Service Account Token
- name: john
  user:
    username: john
    password: mypassword                             # Basic auth (deprecated)

# Contexts - จับคู่ cluster + user + namespace
contexts:
- context:
    cluster: production-cluster
    user: admin-user
    namespace: production
  name: prod-admin
- context:
    cluster: staging-cluster
    user: developer-user
    namespace: staging
  name: staging-dev
- context:
    cluster: local-cluster
    user: admin-user
    namespace: default
  name: local

# Context ที่ใช้งานอยู่ตอนนี้
current-context: local
```

### จัดการ Contexts

```bash
# ดู contexts ทั้งหมด
kubectl config get-contexts

# ดู current context
kubectl config current-context

# เปลี่ยน context
kubectl config use-context staging-dev
kubectl config use-context prod-admin

# ดู config ทั้งหมด
kubectl config view
kubectl config view --minify       # เฉพาะ current context
kubectl config view --flatten      # flatten embedded certs

# สร้าง context ใหม่
kubectl config set-context my-new-context \
    --cluster=my-cluster \
    --user=my-user \
    --namespace=my-namespace

# แก้ไข context ที่มีอยู่
kubectl config set-context prod-admin \
    --namespace=production-apps

# ลบ context
kubectl config delete-context old-context

# เปลี่ยน namespace ใน current context ชั่วคราว
kubectl config set-context --current --namespace=my-namespace

# ตั้งค่า Cluster
kubectl config set-cluster my-cluster \
    --server=https://10.0.0.10:6443 \
    --certificate-authority=/path/to/ca.crt

# ตั้งค่า User credentials
kubectl config set-credentials my-user \
    --client-certificate=/path/to/client.crt \
    --client-key=/path/to/client.key

# เพิ่ม token authentication
kubectl config set-credentials my-user \
    --token=eyJhbGciOiJSUzI1NiIs...
```

### จัดการหลาย kubeconfig ไฟล์

```bash
# ใช้ KUBECONFIG environment variable
export KUBECONFIG=~/.kube/config:~/.kube/config-staging:~/.kube/config-prod

# Merge kubeconfig ไฟล์
kubectl config view --flatten > ~/.kube/merged-config

# ใช้ config ไฟล์เฉพาะสำหรับคำสั่งนั้นๆ
kubectl --kubeconfig=/path/to/config get pods

# ใช้ context เฉพาะโดยไม่เปลี่ยน current context
kubectl --context=staging-dev get pods
```

### ตัวอย่าง: ตั้งค่า Context สำหรับ Multi-Cluster

```bash
# Scenario: มี 3 clusters (dev, staging, prod)

# เพิ่ม dev cluster
kubectl config set-cluster dev-cluster \
    --server=https://dev.k8s.example.com:6443 \
    --certificate-authority=/certs/dev-ca.crt

# เพิ่ม staging cluster  
kubectl config set-cluster staging-cluster \
    --server=https://staging.k8s.example.com:6443 \
    --certificate-authority=/certs/staging-ca.crt

# เพิ่ม prod cluster
kubectl config set-cluster prod-cluster \
    --server=https://prod.k8s.example.com:6443 \
    --certificate-authority=/certs/prod-ca.crt

# เพิ่ม credentials
kubectl config set-credentials dev-admin \
    --client-certificate=/certs/dev-admin.crt \
    --client-key=/certs/dev-admin.key

kubectl config set-credentials staging-developer \
    --token=staging-token-here

kubectl config set-credentials prod-readonly \
    --token=prod-readonly-token

# สร้าง contexts
kubectl config set-context dev \
    --cluster=dev-cluster \
    --user=dev-admin \
    --namespace=development

kubectl config set-context staging \
    --cluster=staging-cluster \
    --user=staging-developer \
    --namespace=staging

kubectl config set-context prod \
    --cluster=prod-cluster \
    --user=prod-readonly \
    --namespace=production

# ใช้งาน
kubectl config use-context dev
kubectl get pods  # ดู pods ใน dev cluster

kubectl config use-context prod
kubectl get pods  # ดู pods ใน prod cluster
```

---

## 5. kubectl Plugins

### kubectl Plugin คืออะไร

kubectl Plugin คือโปรแกรมภายนอกที่สามารถรันผ่าน `kubectl` ได้เหมือน sub-command ปกติ โดยต้องตั้งชื่อไฟล์ขึ้นต้นด้วย `kubectl-`

### Krew - Plugin Manager สำหรับ kubectl

```bash
# ติดตั้ง Krew (Linux/macOS)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# เพิ่ม PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# ตรวจสอบการติดตั้ง
kubectl krew version

# ค้นหา plugin
kubectl krew search
kubectl krew search context

# ติดตั้ง plugin
kubectl krew install ctx          # จัดการ contexts
kubectl krew install ns           # จัดการ namespaces
kubectl krew install neat         # แสดง YAML แบบสะอาด
kubectl krew install tree         # แสดง Resource tree
kubectl krew install stern        # Multi-pod log tailing
kubectl krew install node-shell   # เข้า Node shell
kubectl krew install resource-capacity  # ดู resource usage
kubectl krew install view-secret  # ดูค่า Secrets

# อัปเดต plugins
kubectl krew upgrade

# ลบ plugin
kubectl krew uninstall ctx
```

### Plugins ที่แนะนำ

#### kubectl-ctx (kubectx)

```bash
# ติดตั้ง
kubectl krew install ctx

# ใช้งาน
kubectl ctx              # แสดง contexts ทั้งหมด
kubectl ctx staging      # เปลี่ยนไป staging context
kubectl ctx -            # กลับ context ก่อนหน้า
kubectl ctx -d old-ctx   # ลบ context
```

#### kubectl-ns (kubens)

```bash
# ติดตั้ง
kubectl krew install ns

# ใช้งาน
kubectl ns               # แสดง namespaces ทั้งหมด
kubectl ns production    # เปลี่ยน namespace ปัจจุบัน
kubectl ns -             # กลับ namespace ก่อนหน้า
```

#### kubectl-neat

```bash
# ติดตั้ง
kubectl krew install neat

# ใช้งาน - แสดง YAML แบบสะอาด (ไม่มี managed fields ที่ไม่จำเป็น)
kubectl get pod nginx -o yaml | kubectl neat
kubectl neat get pod nginx
```

#### kubectl-tree

```bash
# ติดตั้ง
kubectl krew install tree

# ใช้งาน - แสดง Resource ownership tree
kubectl tree deployment nginx
kubectl tree ingress my-ingress
```

#### kubectl-stern (Log Tailing)

```bash
# ติดตั้ง stern (อาจต้องติดตั้งแยก)
brew install stern  # macOS

# หรือผ่าน Krew
kubectl krew install stern

# ใช้งาน
stern nginx              # tail logs จาก pods ที่ชื่อ nginx
stern -l app=nginx       # tail logs จาก pods ที่มี label
stern nginx -n production  # เฉพาะ namespace
stern nginx --since 1h   # logs ย้อนหลัง 1 ชั่วโมง
stern nginx -c sidecar   # เฉพาะ container
```

### สร้าง Custom kubectl Plugin

```bash
# สร้าง plugin ง่ายๆ
cat <<'EOF' > /usr/local/bin/kubectl-hello
#!/bin/bash
echo "Hello from kubectl plugin!"
echo "Args: $@"
kubectl get pods "$@"
EOF

chmod +x /usr/local/bin/kubectl-hello

# ใช้งาน
kubectl hello
kubectl hello -n kube-system

# สร้าง plugin ที่ซับซ้อนขึ้น
cat <<'EOF' > /usr/local/bin/kubectl-podip
#!/bin/bash
# แสดง Pod IP addresses
kubectl get pods -o custom-columns='NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName' "$@"
EOF

chmod +x /usr/local/bin/kubectl-podip

# ใช้งาน
kubectl podip
kubectl podip -n production
```

---

## 6. Workshop: kubectl Cheatsheet แบบใช้งานจริง

### สถานการณ์ที่ 1: Quick Start - Deploy Application

```bash
# สถานการณ์: Deploy nginx web server อย่างรวดเร็ว

# Step 1: สร้าง namespace สำหรับ workshop
kubectl create namespace workshop

# Step 2: Deploy nginx
kubectl create deployment nginx-web --image=nginx:1.25 \
    --replicas=3 \
    --namespace=workshop

# Step 3: Expose เป็น Service
kubectl expose deployment nginx-web \
    --port=80 \
    --type=NodePort \
    --namespace=workshop

# Step 4: ดูสถานะ
kubectl get all -n workshop

# Step 5: รับ NodePort
kubectl get service nginx-web -n workshop

# Step 6: ทดสอบ (ปรับ IP และ port ตาม environment)
# curl http://<node-ip>:<node-port>

# Cleanup
kubectl delete namespace workshop
```

### สถานการณ์ที่ 2: Debugging Pod ที่มีปัญหา

```bash
# สถานการณ์: Pod ไม่ start ให้หาสาเหตุ

# สร้าง Pod ที่มีปัญหา (image ไม่มีอยู่จริง)
kubectl run broken-pod --image=nginx:doesnotexist

# Step 1: ดูสถานะ Pod
kubectl get pods
# Output: broken-pod   0/1   ImagePullBackOff   0   2m

# Step 2: describe เพื่อดูรายละเอียด
kubectl describe pod broken-pod
# ดูที่ Events section:
# Warning  Failed     2m    kubelet  Failed to pull image "nginx:doesnotexist"
# Warning  Failed     2m    kubelet  Error: ErrImagePull

# Step 3: แก้ไขด้วยการเปลี่ยน image
kubectl set image pod/broken-pod broken-pod=nginx:latest
# หรือ
kubectl delete pod broken-pod
kubectl run broken-pod --image=nginx:latest

# Step 4: ตรวจสอบว่า Running
kubectl get pods --watch
# รอจนเห็น: broken-pod   1/1   Running

# Step 5: ดู logs เพื่อยืนยัน
kubectl logs broken-pod

# Cleanup
kubectl delete pod broken-pod
```

### สถานการณ์ที่ 3: Rolling Update และ Rollback

```bash
# สถานการณ์: Update application และต้อง rollback

# สร้าง deployment
kubectl create deployment webapp --image=nginx:1.20

# ดูสถานะ
kubectl get deployment webapp

# Update เป็น version ใหม่
kubectl set image deployment/webapp webapp=nginx:1.25 \
    --record  # deprecated แต่ยังใช้ได้

# ดู rollout status
kubectl rollout status deployment/webapp

# ดู history
kubectl rollout history deployment/webapp

# สมมติว่า version ใหม่มีปัญหา - ทำ rollback
kubectl rollout undo deployment/webapp

# Rollback ไปยัง revision เฉพาะ
kubectl rollout undo deployment/webapp --to-revision=1

# ยืนยัน rollback
kubectl get deployment webapp -o=jsonpath='{.spec.template.spec.containers[0].image}'

# Cleanup
kubectl delete deployment webapp
```

### สถานการณ์ที่ 4: Resource Investigation

```bash
# สถานการณ์: ตรวจสอบการใช้ Resources ใน Cluster

# ดู nodes และ resources
kubectl get nodes -o wide
kubectl describe nodes

# ดู resource usage (ต้องมี metrics-server)
kubectl top nodes
kubectl top pods --all-namespaces
kubectl top pods -A --sort-by=memory

# ดู Pods ที่ใช้ CPU มากที่สุด
kubectl top pods -A --sort-by=cpu

# ดู resource requests/limits
kubectl get pods -A -o custom-columns=\
'NAMESPACE:.metadata.namespace,'\
'NAME:.metadata.name,'\
'CPU_REQ:.spec.containers[*].resources.requests.cpu,'\
'CPU_LIM:.spec.containers[*].resources.limits.cpu,'\
'MEM_REQ:.spec.containers[*].resources.requests.memory,'\
'MEM_LIM:.spec.containers[*].resources.limits.memory'
```

### สถานการณ์ที่ 5: Exec และ File Operations

```bash
# สถานการณ์: Debug application ภายใน Container

# สร้าง pod สำหรับทดสอบ
kubectl run debug-pod --image=nginx:latest

# รอให้ Pod running
kubectl wait --for=condition=Ready pod/debug-pod --timeout=60s

# เข้า interactive shell
kubectl exec -it debug-pod -- /bin/bash

# รันคำสั่งโดยตรง
kubectl exec debug-pod -- ls /etc/nginx
kubectl exec debug-pod -- cat /etc/nginx/nginx.conf
kubectl exec debug-pod -- env | sort

# Copy file จาก Pod ไปยัง local
kubectl cp debug-pod:/etc/nginx/nginx.conf ./nginx.conf

# Copy file จาก local ไปยัง Pod
echo "test content" > test.txt
kubectl cp test.txt debug-pod:/tmp/test.txt

# ยืนยัน
kubectl exec debug-pod -- cat /tmp/test.txt

# Cleanup
kubectl delete pod debug-pod
rm -f test.txt nginx.conf
```

### สถานการณ์ที่ 6: Port Forwarding สำหรับ Local Development

```bash
# สถานการณ์: Access service ใน Cluster จาก local machine

# Deploy web app
kubectl create deployment my-webapp --image=nginx:latest
kubectl expose deployment my-webapp --port=80

# รอ deployment พร้อม
kubectl wait --for=condition=Available deployment/my-webapp --timeout=120s

# Port forward ไปยัง Pod
kubectl port-forward deployment/my-webapp 8080:80 &
PF_PID=$!

# ทดสอบ
curl http://localhost:8080

# หยุด port forwarding
kill $PF_PID

# Port forward ไปยัง Service (ดีกว่า - handle pod restarts)
kubectl port-forward service/my-webapp 8080:80 &
PF_PID=$!

curl http://localhost:8080

kill $PF_PID

# Cleanup
kubectl delete deployment my-webapp
kubectl delete service my-webapp
```

### สถานการณ์ที่ 7: Label และ Selector

```bash
# สถานการณ์: จัดการ Pods ด้วย Labels

# สร้าง Pods หลายตัวพร้อม labels
kubectl run pod-v1 --image=nginx:1.20 --labels="app=webapp,version=v1,env=prod"
kubectl run pod-v2 --image=nginx:1.25 --labels="app=webapp,version=v2,env=prod"
kubectl run pod-dev --image=nginx:latest --labels="app=webapp,version=v2,env=dev"

# ดู pods ทั้งหมดพร้อม labels
kubectl get pods --show-labels

# Filter ด้วย label selector
kubectl get pods -l app=webapp
kubectl get pods -l version=v2
kubectl get pods -l env=prod,version=v1  # AND condition
kubectl get pods -l 'version in (v1,v2)' # Set-based

# ดู pods ที่ไม่มี label env=dev
kubectl get pods -l 'env notin (dev)'

# เพิ่ม label
kubectl label pod pod-v1 tier=frontend

# แก้ไข label
kubectl label pod pod-v1 env=staging --overwrite

# ลบ label
kubectl label pod pod-v1 tier-

# Cleanup
kubectl delete pods pod-v1 pod-v2 pod-dev
```

### สถานการณ์ที่ 8: Namespace Operations

```bash
# สถานการณ์: จัดการ Multi-team Environment ด้วย Namespaces

# ดู namespaces ทั้งหมด
kubectl get namespaces

# สร้าง namespace สำหรับแต่ละทีม
kubectl create namespace team-frontend
kubectl create namespace team-backend
kubectl create namespace team-data

# Deploy apps ในแต่ละ namespace
kubectl create deployment frontend --image=nginx --namespace=team-frontend
kubectl create deployment backend --image=python:3.11 --namespace=team-backend
kubectl create deployment database --image=postgres:15 --namespace=team-data

# ดู pods ในแต่ละ namespace
kubectl get pods -n team-frontend
kubectl get pods -n team-backend

# ดู pods ทุก namespace
kubectl get pods -A

# ดู resources ทั้งหมดใน namespace
kubectl get all -n team-frontend

# เปลี่ยน default namespace ชั่วคราว
kubectl config set-context --current --namespace=team-frontend

# ตอนนี้ไม่ต้องระบุ -n
kubectl get pods

# กลับไป default namespace
kubectl config set-context --current --namespace=default

# ลบ namespace (จะลบ resources ทั้งหมดในนั้นด้วย)
kubectl delete namespace team-frontend team-backend team-data
```

### สถานการณ์ที่ 9: Imperative vs Declarative

```bash
# Imperative approach (ใช้สำหรับ quick tasks)
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80
kubectl scale deployment nginx --replicas=3

# Declarative approach (แนะนำสำหรับ production)
# สร้าง manifest
cat <<EOF > /tmp/webapp-manifest.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: default
  labels:
    app: webapp
spec:
  replicas: 3
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
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: default
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Apply manifest
kubectl apply -f /tmp/webapp-manifest.yaml

# ดูผลลัพธ์
kubectl get all -l app=webapp

# แก้ไข replicas ใน manifest แล้ว apply ใหม่
# (เปลี่ยน replicas: 3 เป็น replicas: 5)
sed -i 's/replicas: 3/replicas: 5/' /tmp/webapp-manifest.yaml
kubectl apply -f /tmp/webapp-manifest.yaml

# ดูการเปลี่ยนแปลง
kubectl get deployment webapp

# Cleanup
kubectl delete -f /tmp/webapp-manifest.yaml
```

### สถานการณ์ที่ 10: Generate YAML Template

```bash
# เทคนิค: ใช้ --dry-run=client -o yaml เพื่อ generate template

# Generate Pod YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml

# Generate Deployment YAML
kubectl create deployment nginx --image=nginx --replicas=3 \
    --dry-run=client -o yaml > deployment.yaml

# Generate Service YAML
kubectl expose deployment nginx --port=80 --type=NodePort \
    --dry-run=client -o yaml > service.yaml

# Generate ConfigMap YAML
kubectl create configmap my-config \
    --from-literal=key1=value1 \
    --from-literal=key2=value2 \
    --dry-run=client -o yaml > configmap.yaml

# Generate Secret YAML
kubectl create secret generic my-secret \
    --from-literal=username=admin \
    --from-literal=password=mypassword \
    --dry-run=client -o yaml > secret.yaml

# ดู generated files
cat deployment.yaml
cat service.yaml

# Cleanup
rm -f deployment.yaml service.yaml configmap.yaml secret.yaml
```

### kubectl Aliases ที่มีประโยชน์

```bash
# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgd='kubectl get deployments'
alias kgs='kubectl get services'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'
alias kds='kubectl describe service'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kns='kubectl config set-context --current --namespace'
alias kctx='kubectl config use-context'
alias kgctx='kubectl config get-contexts'

# Functions ที่มีประโยชน์
# ลบ pods ที่อยู่ใน Error/Completed state
kclean() {
    kubectl get pods -A | grep -E '(Error|Completed|Evicted)' | \
        awk '{print "kubectl delete pod " $2 " -n " $1}' | bash
}

# ดู logs ของ pod ล่าสุดใน deployment
klast() {
    local deployment=$1
    local namespace=${2:-default}
    local pod=$(kubectl get pods -n $namespace -l app=$deployment \
        --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
    kubectl logs -f $pod -n $namespace
}
```

### สรุป kubectl Commands สำคัญ

| Command | คำอธิบาย | ตัวอย่าง |
|---------|----------|---------|
| `get` | ดูข้อมูล Resources | `kubectl get pods -A` |
| `describe` | ดูรายละเอียด Resource | `kubectl describe pod nginx` |
| `apply` | Apply manifest (idempotent) | `kubectl apply -f app.yaml` |
| `create` | สร้าง Resource | `kubectl create deployment app --image=nginx` |
| `delete` | ลบ Resource | `kubectl delete pod nginx` |
| `logs` | ดู Container logs | `kubectl logs -f pod nginx` |
| `exec` | รันคำสั่งใน Container | `kubectl exec -it nginx -- bash` |
| `port-forward` | Forward ports | `kubectl port-forward pod/nginx 8080:80` |
| `scale` | ปรับ Replicas | `kubectl scale deployment app --replicas=5` |
| `rollout` | จัดการ Deployment rollout | `kubectl rollout undo deployment/app` |
| `top` | ดู Resource usage | `kubectl top pods` |
| `config` | จัดการ kubeconfig | `kubectl config use-context prod` |

---

## สรุป

`kubectl` เป็นเครื่องมือที่ขาดไม่ได้สำหรับการทำงานกับ Kubernetes ความเชี่ยวชาญใน kubectl จะช่วยให้:

1. **Troubleshoot** ปัญหาได้รวดเร็ว
2. **จัดการ Resources** ได้อย่างมีประสิทธิภาพ
3. **Automate** งาน routine ต่างๆ
4. **Understand** ว่า Kubernetes ทำงานอย่างไร

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **Pods** ซึ่งเป็น building block พื้นฐานที่สุดใน Kubernetes
