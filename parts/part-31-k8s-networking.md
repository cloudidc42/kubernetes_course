# Part 31: Kubernetes Networking Model

## สารบัญ
1. [Kubernetes Networking Model พื้นฐาน](#networking-model)
2. [CNI (Container Network Interface)](#cni)
3. [Pod-to-Pod Communication](#pod-communication)
4. [ติดตั้ง CNI Plugins](#install-cni)
5. [Workshop: ทดสอบ Pod Networking](#workshop)

---

## 1. Kubernetes Networking Model พื้นฐาน {#networking-model}

### ภาพรวม Networking ใน Kubernetes

Kubernetes มีโมเดล Networking ที่ออกแบบมาเพื่อแก้ปัญหาการสื่อสารระหว่าง Containers, Pods, และ Services

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  ┌─────────────────┐      ┌─────────────────┐              │
│  │    Node 1        │      │    Node 2        │              │
│  │                  │      │                  │              │
│  │  ┌────────────┐  │      │  ┌────────────┐  │              │
│  │  │   Pod A    │  │      │  │   Pod C    │  │              │
│  │  │ 10.244.1.2 │  │      │  │ 10.244.2.2 │  │              │
│  │  └────────────┘  │      │  └────────────┘  │              │
│  │  ┌────────────┐  │      │  ┌────────────┐  │              │
│  │  │   Pod B    │  │      │  │   Pod D    │  │              │
│  │  │ 10.244.1.3 │  │      │  │ 10.244.2.3 │  │              │
│  │  └────────────┘  │      │  └────────────┘  │              │
│  │                  │      │                  │              │
│  │  eth0: 192.168   │      │  eth0: 192.168   │              │
│  │  .1.10           │      │  .1.11           │              │
│  └─────────────────┘      └─────────────────┘              │
│                                                              │
│  Network: 10.244.0.0/16 (Pod Network)                      │
│  Network: 192.168.1.0/24 (Node Network)                    │
└─────────────────────────────────────────────────────────────┘
```

### กฎ 4 ข้อของ Kubernetes Networking

Kubernetes กำหนดกฎพื้นฐาน 4 ข้อสำหรับ Networking:

**กฎที่ 1: Pods สามารถสื่อสารกับ Pods อื่นได้โดยไม่ต้องใช้ NAT**
```
Pod A (10.244.1.2) ──────────► Pod B (10.244.1.3)
                    No NAT needed!
```

**กฎที่ 2: Nodes สามารถสื่อสารกับ Pods ได้โดยไม่ต้องใช้ NAT**
```
Node (192.168.1.10) ──────────► Pod (10.244.1.2)
                     No NAT needed!
```

**กฎที่ 3: IP ที่ Pod เห็นตัวเองต้องเป็น IP เดียวกับที่ Pods อื่นเห็น**
```
Pod A sees itself as: 10.244.1.2
Other pods see Pod A as: 10.244.1.2
(Same IP - No NAT)
```

**กฎที่ 4: Services มี IP เป็นของตัวเอง (ClusterIP)**
```
Service ──► ClusterIP (10.96.100.1)
           ↓
    Load Balance to Pods
```

### Network Layers ใน Kubernetes

```
Layer 7 (Application)  ──► Ingress Controllers, Service Mesh
Layer 4 (Transport)    ──► Services (TCP/UDP)
Layer 3 (Network)      ──► Pod IPs, Node IPs, CNI
Layer 2 (Data Link)    ──► veth pairs, bridges, overlays
```

### Pod Network Namespace

ทุก Pod มี Network Namespace ของตัวเอง:

```
┌─────────────── Pod ───────────────┐
│                                   │
│  ┌─────────────────────────────┐  │
│  │     Network Namespace       │  │
│  │                             │  │
│  │  lo (127.0.0.1)            │  │
│  │  eth0 (10.244.1.2)         │  │
│  │                             │  │
│  │  Routing Table:             │  │
│  │  default via 10.244.1.1    │  │
│  └─────────────────────────────┘  │
│                                   │
│  Container 1    Container 2       │
│  (shares namespace)               │
└───────────────────────────────────┘
```

---

## 2. CNI (Container Network Interface) {#cni}

### CNI คืออะไร?

CNI (Container Network Interface) คือ specification และ library สำหรับการจัดการ Network Interface ใน Linux Containers

```
┌──────────────────────────────────────────────┐
│              Kubernetes (kubelet)             │
│                                              │
│  Pod Created ──► Call CNI Plugin             │
│  Pod Deleted ──► Call CNI Plugin (cleanup)   │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│              CNI Plugin                       │
│                                              │
│  • Calico    • Flannel    • Weave           │
│  • Cilium    • Canal      • Antrea          │
│  • Multus    • AWS VPC CNI                  │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│           Network Operations                  │
│                                              │
│  1. Create veth pair                         │
│  2. Assign IP address                        │
│  3. Setup routing                            │
│  4. Configure iptables/ebpf                  │
└──────────────────────────────────────────────┘
```

### CNI Plugin Types

**Overlay Networks:**
- ห่อหุ้ม packets ใน packets อีกชั้นหนึ่ง
- ทำงานกับ Network Infrastructure ที่หลากหลาย
- มี overhead เพิ่มขึ้นเล็กน้อย

```
┌─── Original Packet ───┐
│ Src: 10.244.1.2       │
│ Dst: 10.244.2.3       │
└───────────────────────┘
          │
          ▼ Encapsulate (VXLAN/IPIP)
┌─── Outer Packet ──────────────────────────┐
│ Src: 192.168.1.10 (Node 1)               │
│ Dst: 192.168.1.11 (Node 2)               │
│ ┌─── Inner Packet ────────────────────┐  │
│ │ Src: 10.244.1.2                     │  │
│ │ Dst: 10.244.2.3                     │  │
│ └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

**Underlay Networks (BGP/Routing):**
- ใช้ Routing โดยตรง ไม่มี encapsulation
- Performance ดีกว่า
- ต้องการ Network Infrastructure ที่รองรับ BGP

```
Node 1 (192.168.1.10)
  └── Route: 10.244.1.0/24 via local
  
Node 2 (192.168.1.11)  
  └── Route: 10.244.2.0/24 via local

BGP Router:
  └── Route: 10.244.1.0/24 via 192.168.1.10
  └── Route: 10.244.2.0/24 via 192.168.1.11
```

### CNI Configuration File

```bash
# ตัวอย่าง CNI Configuration
cat /etc/cni/net.d/10-flannel.conflist
```

```json
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

### เปรียบเทียบ CNI Plugins

| Feature | Calico | Flannel | Weave | Cilium |
|---------|--------|---------|-------|--------|
| Network Policy | ✅ Advanced | ❌ | ✅ Basic | ✅ Advanced |
| Performance | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Overlay Mode | Optional | Required | Required | Optional |
| BGP Support | ✅ | ❌ | ❌ | ✅ |
| eBPF Support | ✅ | ❌ | ❌ | ✅ Native |
| Encryption | ✅ WireGuard | ❌ | ✅ | ✅ |
| Complexity | Medium | Low | Low | High |
| Use Case | Production | Simple | Simple | Advanced |

---

## 3. Pod-to-Pod Communication {#pod-communication}

### Communication ภายใน Node เดียวกัน

```
┌─────────────────── Node ──────────────────────┐
│                                               │
│  ┌─────────┐    ┌─────────┐                  │
│  │  Pod A  │    │  Pod B  │                  │
│  │eth0     │    │eth0     │                  │
│  │10.244.1.2│    │10.244.1.3│                  │
│  └────┬────┘    └────┬────┘                  │
│       │              │                        │
│  veth0│          veth1│                       │
│       │              │                        │
│  ┌────▼──────────────▼────┐                  │
│  │      cbr0 (bridge)     │                  │
│  │      10.244.1.1        │                  │
│  └────────────────────────┘                  │
│                                               │
└───────────────────────────────────────────────┘

Traffic Flow:
Pod A → veth0 → cbr0 bridge → veth1 → Pod B
```

### Communication ข้าม Nodes (Overlay)

```
Node 1 (192.168.1.10)         Node 2 (192.168.1.11)
┌──────────────────┐          ┌──────────────────┐
│                  │          │                  │
│  Pod A           │          │  Pod C           │
│  10.244.1.2      │          │  10.244.2.2      │
│       │          │          │       ▲          │
│  veth0│          │          │  veth0│          │
│       │          │          │       │          │
│  ┌────▼──────┐   │          │  ┌────┴──────┐   │
│  │  cbr0     │   │          │  │  cbr0     │   │
│  │10.244.1.1 │   │          │  │10.244.2.1 │   │
│  └────┬──────┘   │          │  └────┬──────┘   │
│       │          │          │       │          │
│  ┌────▼──────┐   │          │  ┌────┴──────┐   │
│  │  flannel0 │   │          │  │  flannel0 │   │
│  │ (VXLAN)   │   │          │  │ (VXLAN)   │   │
│  └────┬──────┘   │          │  └────┬──────┘   │
│       │          │          │       │          │
│  ┌────▼──────┐   │          │  ┌────┴──────┐   │
│  │  eth0     │───┼──UDP:8472┼──│  eth0     │   │
│  │192.168.1.10│   │          │  │192.168.1.11│   │
│  └───────────┘   │          │  └───────────┘   │
└──────────────────┘          └──────────────────┘
```

### สร้าง veth Pair ด้วยมือ (เพื่อทำความเข้าใจ)

```bash
# สร้าง network namespace จำลอง pod
ip netns add pod1
ip netns add pod2

# สร้าง veth pair
ip link add veth1 type veth peer name veth2

# ย้าย veth เข้า namespace
ip link set veth1 netns pod1
ip link set veth2 netns pod2

# กำหนด IP address
ip netns exec pod1 ip addr add 10.244.0.1/24 dev veth1
ip netns exec pod2 ip addr add 10.244.0.2/24 dev veth2

# เปิด interface
ip netns exec pod1 ip link set veth1 up
ip netns exec pod2 ip link set veth2 up

# ทดสอบ
ip netns exec pod1 ping 10.244.0.2
```

### ตรวจสอบ Network ของ Pod

```bash
# ดู IP ของ Pod
kubectl get pod <pod-name> -o wide

# เข้าไปใน Pod และดู network
kubectl exec -it <pod-name> -- ip addr
kubectl exec -it <pod-name> -- ip route
kubectl exec -it <pod-name> -- cat /etc/resolv.conf

# ดู veth pair บน Node
ip link show | grep veth

# ดู Bridge
ip link show type bridge
brctl show
```

---

## 4. ติดตั้ง CNI Plugins {#install-cni}

### ติดตั้ง Calico

Calico เป็น CNI ที่นิยมใช้ในระดับ Production เนื่องจากมี Network Policy ที่ครบครัน

```bash
# วิธีที่ 1: ติดตั้งด้วย kubectl apply
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# วิธีที่ 2: ติดตั้งด้วย Helm
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm install calico projectcalico/tigera-operator --namespace tigera-operator --create-namespace

# ตรวจสอบการติดตั้ง
kubectl get pods -n kube-system | grep calico
kubectl get nodes
```

#### Calico Manifest ที่สำคัญ

```yaml
# calico-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: calico-config
  namespace: kube-system
data:
  # CNI network config
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "nodename": "__KUBERNETES_NODE_NAME__",
          "mtu": __CNI_MTU__,
          "ipam": {
            "type": "calico-ipam"
          },
          "policy": {
            "type": "k8s"
          },
          "kubernetes": {
            "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }
```

#### Calico IPPool Configuration

```yaml
# calico-ippool.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  ipipMode: CrossSubnet   # IPIP สำหรับ cross-subnet, None สำหรับ same-subnet
  vxlanMode: Never
  natOutgoing: true       # NAT สำหรับ traffic ออกไปนอก cluster
  disabled: false
  nodeSelector: all()
```

```bash
# ตรวจสอบ IPPool
calicoctl get ippool -o wide

# ดู Calico nodes
calicoctl get node

# ดู BGP peers
calicoctl get bgppeer

# ดู routing table
calicoctl get bgpconfig
```

### ติดตั้ง Flannel

Flannel เป็น CNI ที่ง่ายและเหมาะสำหรับ development

```bash
# ติดตั้ง Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# ตรวจสอบ
kubectl get pods -n kube-flannel
kubectl get nodes
```

#### Flannel ConfigMap

```yaml
# kube-flannel-cfg.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "VNI": 1
      }
    }
```

### ติดตั้ง Weave Net

```bash
# ติดตั้ง Weave Net
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# ตรวจสอบ
kubectl get pods -n kube-system | grep weave

# ดู Weave status
kubectl exec -n kube-system weave-net-<pod-id> -- /home/weave/weave --local status
```

#### Weave Net DaemonSet

```yaml
# weave-net.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: weave-net
  namespace: kube-system
  labels:
    name: weave-net
spec:
  minReadySeconds: 5
  selector:
    matchLabels:
      name: weave-net
  template:
    metadata:
      labels:
        name: weave-net
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      hostPID: true
      tolerations:
        - effect: NoSchedule
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: weave-net
      containers:
        - name: weave
          command:
            - /home/weave/launch.sh
          env:
            - name: INIT_CONTAINER
              value: "true"
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
            - name: IPALLOC_RANGE
              value: "10.244.0.0/16"
          image: weaveworks/weave-kube:2.8.1
          ports:
            - name: metrics
              containerPort: 6782
              hostPort: 6782
          readinessProbe:
            httpGet:
              host: 127.0.0.1
              path: /status
              port: 6784
          resources:
            requests:
              cpu: 50m
          securityContext:
            privileged: true
          volumeMounts:
            - name: weavedb
              mountPath: /weavedb
            - name: cni-bin
              mountPath: /host/opt
            - name: cni-bin2
              mountPath: /host/home
            - name: cni-conf
              mountPath: /host/etc
            - name: dbus
              mountPath: /host/var/lib/dbus
            - name: lib-modules
              mountPath: /lib/modules
            - name: xtables-lock
              mountPath: /run/xtables.lock
              readOnly: false
      volumes:
        - name: weavedb
          hostPath:
            path: /var/lib/weave
        - name: cni-bin
          hostPath:
            path: /opt
        - name: cni-bin2
          hostPath:
            path: /home
        - name: cni-conf
          hostPath:
            path: /etc
        - name: dbus
          hostPath:
            path: /var/lib/dbus
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
```

### ติดตั้ง Cilium (Advanced)

Cilium ใช้ eBPF สำหรับ performance สูง

```bash
# ติดตั้ง Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
mv cilium /usr/local/bin

# ติดตั้ง Cilium ด้วย Helm
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.14.0 \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443

# ตรวจสอบ
cilium status
cilium connectivity test
```

---

## 5. Workshop: ทดสอบ Pod Networking {#workshop}

### สภาพแวดล้อม

สำหรับ Workshop นี้ เราจะใช้ kind (Kubernetes in Docker) หรือ minikube

```bash
# สร้าง kind cluster พร้อม Flannel
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: networking-lab
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true  # ปิด default CNI เพื่อติดตั้งเอง
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
EOF

kind create cluster --config kind-config.yaml

# ติดตั้ง Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Workshop 1: ทดสอบ Basic Pod Networking

```bash
# สร้าง Pod สองตัวบน Nodes ต่างกัน
cat > pod-network-test.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  labels:
    app: pod-a
spec:
  nodeName: networking-lab-worker  # บน Worker Node 1
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  labels:
    app: pod-b
spec:
  nodeName: networking-lab-worker2  # บน Worker Node 2
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl apply -f pod-network-test.yaml
kubectl get pods -o wide
```

```bash
# ดู IP ของแต่ละ Pod
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')

echo "Pod A IP: $POD_A_IP"
echo "Pod B IP: $POD_B_IP"

# ทดสอบ connectivity
kubectl exec pod-a -- ping -c 3 $POD_B_IP
kubectl exec pod-b -- ping -c 3 $POD_A_IP
```

### Workshop 2: ตรวจสอบ Network Path

```bash
# ดู Network Interface ใน Pod
kubectl exec pod-a -- ip addr show
kubectl exec pod-a -- ip route show

# ตรวจสอบ DNS resolution
kubectl exec pod-a -- cat /etc/resolv.conf

# ทดสอบ traceroute ระหว่าง Pods
kubectl exec pod-a -- traceroute $POD_B_IP

# ดู iptables rules (บน Node)
# ต้อง ssh เข้า Node ก่อน
docker exec networking-lab-worker iptables -t nat -L -n | grep -A5 "KUBE"
```

### Workshop 3: ทดสอบ Network Bandwidth

```bash
# ทดสอบ bandwidth ระหว่าง Pods ด้วย iperf3
# รัน iperf3 server ใน pod-b
kubectl exec pod-b -- iperf3 -s -D

# รัน iperf3 client ใน pod-a
kubectl exec pod-a -- iperf3 -c $POD_B_IP -t 10

# ทดสอบ UDP
kubectl exec pod-a -- iperf3 -c $POD_B_IP -u -b 1G
```

### Workshop 4: ตรวจสอบ CNI Plugin

```bash
# ดู CNI configuration files
docker exec networking-lab-worker cat /etc/cni/net.d/10-flannel.conflist

# ดู CNI binaries
docker exec networking-lab-worker ls /opt/cni/bin/

# ดู Flannel subnet allocation
docker exec networking-lab-worker cat /run/flannel/subnet.env

# ตรวจสอบ routing table
docker exec networking-lab-worker ip route

# ดู VXLAN tunnel interfaces
docker exec networking-lab-worker ip link show type vxlan
```

### Workshop 5: Debug Network Issues

```bash
# สร้าง Pod สำหรับ debugging
kubectl run debug-pod --image=nicolaka/netshoot --command -- sleep infinity

# เข้าไปใน Pod แบบ interactive
kubectl exec -it debug-pod -- bash

# คำสั่งที่ใช้ debug ภายใน Pod
# ตรวจสอบ DNS
nslookup kubernetes.default.svc.cluster.local

# ตรวจสอบ connectivity
curl -k https://kubernetes.default.svc.cluster.local

# ดู network statistics
ss -tlnp
netstat -rn

# Packet capture
tcpdump -i eth0 -n icmp

# ออกจาก Pod
exit
```

### Workshop 6: Multi-Container Pod Networking

```bash
# สาธิต Containers ใน Pod เดียวกันใช้ Network Namespace เดียวกัน
cat > multi-container-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
  - name: client
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl apply -f multi-container-pod.yaml

# Container 'client' สามารถเข้าถึง nginx ผ่าน localhost
kubectl exec multi-container -c client -- curl localhost:80

# ดู network interface จาก container 'client'
kubectl exec multi-container -c client -- ip addr

# ดู network interface จาก container 'nginx' - เหมือนกัน!
kubectl exec multi-container -c nginx -- ip addr
```

### Workshop 7: Host Network Pod

```bash
# Pod ที่ใช้ Network ของ Host
cat > host-network-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: host-network-pod
spec:
  hostNetwork: true  # ใช้ network namespace ของ host
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl apply -f host-network-pod.yaml

# ดู IP - จะเป็น IP ของ Node
kubectl exec host-network-pod -- ip addr

# เปรียบเทียบกับ Pod ปกติ
kubectl exec pod-a -- ip addr
```

### การ Cleanup

```bash
# ลบ Resources ทั้งหมด
kubectl delete pod pod-a pod-b debug-pod multi-container host-network-pod
kubectl delete -f pod-network-test.yaml

# ลบ kind cluster (ถ้าต้องการ)
kind delete cluster --name networking-lab
```

---

## สรุปสิ่งที่ได้เรียน

```
Kubernetes Networking
├── Flat Network Model
│   ├── Pod IPs routable cluster-wide
│   ├── No NAT between pods
│   └── Each pod has unique IP
│
├── CNI (Container Network Interface)
│   ├── Calico (BGP/IPIP, full NetworkPolicy)
│   ├── Flannel (VXLAN overlay, simple)
│   ├── Weave (encrypted overlay)
│   └── Cilium (eBPF, high performance)
│
├── Communication Paths
│   ├── Same Node: via Linux bridge
│   ├── Cross Node: via overlay or routing
│   └── Pod → Service → Pod: via kube-proxy
│
└── Tools
    ├── ip, bridge, iptables
    ├── tcpdump, wireshark
    └── iperf3, netshoot
```

### Cheat Sheet

```bash
# ดู Pod IPs
kubectl get pods -o wide
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# ดู Node Network
kubectl get nodes -o wide

# Debug network ด้วย netshoot
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash

# ตรวจสอบ CNI
ls /etc/cni/net.d/
ls /opt/cni/bin/

# ดู Network Policies
kubectl get networkpolicies --all-namespaces

# ดู Endpoints
kubectl get endpoints

# Force delete stuck pod
kubectl delete pod <pod-name> --grace-period=0 --force
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [CNI Specification](https://github.com/containernetworking/cni)
- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [Flannel GitHub](https://github.com/flannel-io/flannel)
- [Cilium Documentation](https://docs.cilium.io/)

---

*ต่อไป: [Part 32 - ClusterIP Service](./part-32-clusterip.md)*
