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

---

## 6. CNI Plugin เชิงลึก {#cni-deep-dive}

### 6.1 Calico CNI - BGP Configuration

Calico เป็น CNI ที่ใช้ BGP (Border Gateway Protocol) สำหรับ routing ระหว่าง Nodes โดยไม่ต้องใช้ overlay network

```
┌─────────────────────────────────────────────────────────────────┐
│                    Calico BGP Architecture                       │
│                                                                   │
│  Node 1 (10.0.0.1)          Node 2 (10.0.0.2)                   │
│  ┌─────────────────┐        ┌─────────────────┐                  │
│  │  Pod CIDR:      │  BGP   │  Pod CIDR:      │                  │
│  │  192.168.1.0/24 │◄──────►│  192.168.2.0/24 │                  │
│  │                 │ Peering│                  │                  │
│  │  Bird BGP       │        │  Bird BGP        │                  │
│  │  (calico-node)  │        │  (calico-node)   │                  │
│  └─────────────────┘        └─────────────────┘                  │
│           │                          │                            │
│           └──────────┬───────────────┘                           │
│                      ▼                                            │
│              Top of Rack Switch                                   │
│              (BGP Router)                                         │
└─────────────────────────────────────────────────────────────────┘
```

#### ติดตั้ง Calico

```bash
# ติดตั้ง Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# สร้าง Installation resource
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

#### Calico BGP Peer Configuration

```yaml
# bgppeer.yaml - กำหนด BGP Peer กับ External Router
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: external-router
spec:
  peerIP: 10.0.0.254        # IP ของ External BGP Router
  asNumber: 65000           # AS Number ของ External Router
  nodeSelector: all()       # Apply กับทุก Node

---
# bgpconfiguration.yaml - ตั้งค่า BGP Global
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true    # Full mesh BGP ระหว่าง nodes
  asNumber: 65001               # AS Number ของ Cluster
  serviceClusterIPs:
  - cidr: 10.96.0.0/12          # Advertise ClusterIP range
  serviceExternalIPs:
  - cidr: 203.0.113.0/24        # Advertise External IP range
```

#### Calico IP Pool Configuration

```yaml
# ippool.yaml - กำหนด IP Pool
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  blockSize: 26                 # /26 = 64 IPs per block
  ipipMode: Never               # ปิด IP-in-IP
  vxlanMode: CrossSubnet        # VXLAN เฉพาะ cross-subnet
  natOutgoing: true
  nodeSelector: all()
  disabled: false

---
# ippool-gpu-nodes.yaml - IP Pool แยกสำหรับ GPU Nodes
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: gpu-nodes-pool
spec:
  cidr: 172.16.0.0/24
  blockSize: 28                 # /28 = 16 IPs per block
  nodeSelector: gpu == "true"
  natOutgoing: true
```

#### Calico Network Policy (ละเอียดกว่า Kubernetes NetworkPolicy)

```yaml
# calico-network-policy.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
  namespace: production
spec:
  selector: app == "web"        # Apply กับ Pod ที่มี label app=web
  types:
  - Ingress
  - Egress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "frontend"
      namespaceSelector: tier == "frontend"
    destination:
      ports:
      - 8080
  - action: Allow
    protocol: TCP
    source:
      nets:
      - 10.0.0.0/8              # Allow จาก Internal Network
    destination:
      ports:
      - 443
  egress:
  - action: Allow
    protocol: TCP
    destination:
      selector: app == "database"
      ports:
      - 5432
  - action: Allow
    protocol: UDP
    destination:
      ports:
      - 53                      # DNS
  - action: Deny               # Block traffic อื่นๆ
    destination:
      nets:
      - 169.254.169.254/32      # Block AWS metadata

---
# calico-global-network-policy.yaml - Policy ระดับ Global
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-all
spec:
  selector: all()
  order: 1000                   # ลำดับความสำคัญ (ต่ำ = สูง)
  types:
  - Ingress
  - Egress
  egress:
  - action: Allow
    protocol: UDP
    destination:
      ports:
      - 53                      # Allow DNS
  - action: Allow
    destination:
      nets:
      - 10.96.0.0/12            # Allow to cluster services
```

#### Calico Commands

```bash
# ดู BGP Peers
calicoctl get bgppeers
calicoctl get bgpconfiguration

# ดู IP Pools
calicoctl get ippools -o wide
calicoctl get ipam nodes

# ดู IPAM allocations
calicoctl ipam show --show-blocks
calicoctl ipam show --ip=192.168.1.10

# Debug BGP
calicoctl node status
calicoctl node diags

# ดู Routes ที่ Bird advertise
kubectl exec -n calico-system -it $(kubectl get pod -n calico-system -l k8s-app=calico-node -o jsonpath='{.items[0].metadata.name}') -- birdcl show route

# ดู Felix logs
kubectl logs -n calico-system -l k8s-app=calico-node -c calico-node --tail=50
```

---

### 6.2 Flannel CNI - VXLAN Configuration

Flannel เป็น CNI ที่ง่ายที่สุด ใช้ VXLAN overlay network สำหรับ pod-to-pod communication

```
┌─────────────────────────────────────────────────────────────────┐
│                    Flannel VXLAN Architecture                    │
│                                                                   │
│  Node 1                           Node 2                         │
│  ┌──────────────────┐             ┌──────────────────┐           │
│  │  Pod: 10.244.1.2 │             │  Pod: 10.244.2.3 │           │
│  │       │          │             │       ▲          │           │
│  │       ▼          │             │       │          │           │
│  │    cni0 bridge   │             │    cni0 bridge   │           │
│  │  10.244.1.1/24   │             │  10.244.2.1/24   │           │
│  │       │          │             │       │          │           │
│  │       ▼          │             │       ▼          │           │
│  │    flannel.1     │   VXLAN     │    flannel.1     │           │
│  │  (VTEP device)   │◄───────────►│  (VTEP device)   │           │
│  │       │          │  UDP:8472   │       │          │           │
│  │       ▼          │             │       ▼          │           │
│  │    eth0          │             │    eth0          │           │
│  │  192.168.1.10    │             │  192.168.1.11    │           │
│  └──────────────────┘             └──────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

#### ติดตั้ง Flannel

```bash
# ติดตั้ง Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# หรือ custom CIDR
kubectl apply -f - <<EOF
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
        "VNI": 1,
        "Port": 8472,
        "DirectRouting": true
      }
    }
EOF
```

#### Flannel Backend Types

```yaml
# VXLAN Backend (default) - ปลอดภัย, overhead สูง
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "vxlan",
      "VNI": 1,
      "Port": 8472,
      "DirectRouting": false   # true = ใช้ direct route ถ้า nodes อยู่ subnet เดียว
    }
  }

# Host-GW Backend - Performance สูง, ต้องอยู่ L2 เดียวกัน
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "host-gw"
    }
  }

# WireGuard Backend - Encrypted
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "wireguard",
      "PSK": "pre-shared-key-here",
      "Port": 51820
    }
  }

# IPSec Backend - Encrypted
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "ipsec",
      "PSK": "pre-shared-key-here"
    }
  }
```

#### Debug Flannel

```bash
# ดู Flannel logs
kubectl logs -n kube-flannel -l app=flannel --tail=50

# ดู routes ที่ Flannel สร้าง
ip route show | grep flannel
ip route show | grep "10.244"

# ดู VXLAN interface
ip -d link show flannel.1
bridge fdb show dev flannel.1

# ดู ARP table
arp -an | grep flannel

# ตรวจสอบ VXLAN packets
tcpdump -i eth0 'udp port 8472' -XX -n
```

---

### 6.3 Cilium CNI - eBPF Architecture

Cilium ใช้ eBPF (extended Berkeley Packet Filter) แทน iptables สำหรับ networking, security, และ observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cilium eBPF Architecture                      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Linux Kernel                           │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │                  eBPF Programs                   │    │   │
│  │  │                                                   │    │   │
│  │  │  TC Ingress ──► Policy Enforcement ──► Forwarding│    │   │
│  │  │  TC Egress  ◄── Load Balancing   ◄── NAT        │    │   │
│  │  │                                                   │    │   │
│  │  │  eBPF Maps (Shared State):                       │    │   │
│  │  │  - Policy Map                                     │    │   │
│  │  │  - Connection Tracking Map                        │    │   │
│  │  │  - Service Map (replaces kube-proxy)              │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Userspace:                                                       │
│  - Cilium Agent (DaemonSet)                                       │
│  - Cilium Operator                                                │
│  - Hubble (Observability)                                         │
└─────────────────────────────────────────────────────────────────┘
```

#### ติดตั้ง Cilium

```bash
# ติดตั้ง Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

# ติดตั้ง Cilium ใน cluster
cilium install --version 1.14.0

# หรือใช้ Helm
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.14.0 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=10.0.0.1 \
  --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# ตรวจสอบ status
cilium status --wait
cilium connectivity test
```

#### Cilium CiliumNetworkPolicy

```yaml
# cilium-l7-policy.yaml - Layer 7 Policy (HTTP-aware)
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: api-gateway-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"       # Allow GET /api/v1/*
        - method: "POST"
          path: "/api/v1/users"    # Allow POST /api/v1/users
        - method: "PUT"
          path: "/api/v1/users/[0-9]+"  # Allow PUT /api/v1/users/{id}
  - fromEndpoints:
    - matchLabels:
        app: admin
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "DELETE"
          path: "/api/v1/.*"       # Admin can DELETE

---
# cilium-dns-policy.yaml - DNS-based Policy
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-dns
spec:
  endpointSelector:
    matchLabels:
      app: web
  egress:
  - toFQDNs:
    - matchName: "api.github.com"
    - matchPattern: "*.amazonaws.com"
    - matchPattern: "*.googleapis.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": kube-system
    toPorts:
    - ports:
      - port: "53"
        protocol: ANY
      rules:
        dns:
        - matchPattern: "*"
```

#### Hubble - Cilium Observability

```bash
# ติดตั้ง Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz
tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin

# Enable Hubble port-forward
cilium hubble port-forward &

# ดู traffic flows
hubble observe --namespace production --follow
hubble observe --pod frontend/web-xxx --protocol tcp
hubble observe --verdict DROPPED --follow

# ดู traffic ระหว่าง namespaces
hubble observe \
  --namespace production \
  --to-namespace database \
  --follow

# ดู HTTP requests
hubble observe \
  --protocol http \
  --http-url /api/v1/users \
  --follow

# ดู statistics
hubble observe summary
```

---

## 7. Pod IP Allocation {#pod-ip-allocation}

### 7.1 กระบวนการ Allocate IP ให้ Pod

```
1. User submits Pod spec → API Server
2. API Server stores Pod in etcd (no IP yet)
3. Scheduler assigns Pod to a Node
4. Kubelet on that Node receives the Pod assignment
5. Kubelet calls CRI (Container Runtime Interface)
6. CRI calls CNI plugin via /etc/cni/net.d/
7. CNI plugin allocates IP from IPAM
8. CNI creates veth pair (pod ↔ node bridge)
9. CNI assigns IP to pod's veth interface
10. CNI updates routing tables
11. Pod starts with allocated IP
```

```bash
# ดูขั้นตอนการ allocate IP
# 1. ดู IPAM configuration
cat /etc/cni/net.d/10-flannel.conflist
cat /etc/cni/net.d/10-calico.conflist

# 2. ดู IP ranges ที่แต่ละ Node ได้รับ
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# 3. ดู IPs ที่ถูกใช้งาน
kubectl get pods --all-namespaces -o wide | awk '{print $8}'

# 4. ดู IP allocation ใน Calico
calicoctl ipam show --show-blocks
```

### 7.2 IPAM (IP Address Management)

```yaml
# ตัวอย่าง CNI config พร้อม IPAM
{
  "cniVersion": "0.4.0",
  "name": "k8s-pod-network",
  "type": "calico",
  "log_level": "info",
  "datastore_type": "kubernetes",
  "nodename": "__KUBERNETES_NODE_NAME__",
  "mtu": "__CNI_MTU__",
  "ipam": {
    "type": "calico-ipam",
    "assign_ipv4": "true",
    "assign_ipv6": "false",
    "ipv4_pools": ["192.168.0.0/16"],
    "ipv6_pools": []
  },
  "policy": {
    "type": "k8s"
  },
  "kubernetes": {
    "kubeconfig": "__KUBECONFIG_FILEPATH__"
  }
}
```

### 7.3 Pod IP Lifecycle

```bash
# สร้าง Pod และดู IP allocation
kubectl run test-pod --image=nginx
kubectl get pod test-pod -o wide

# ดู IP ในรายละเอียด
kubectl get pod test-pod -o jsonpath='{.status.podIP}'
kubectl get pod test-pod -o jsonpath='{.status.podIPs}'

# ดู veth pair บน Node
# (ต้อง SSH เข้า Node)
ip link show type veth
ip addr show | grep -A2 "veth"

# ดู Pod network namespace
# หา PID ของ container
docker inspect <container-id> | grep Pid
# หรือ
crictl inspect <container-id> | grep pid

# เข้าดู network namespace
nsenter -t <pid> -n ip addr

# ลบ Pod - IP จะถูก return กลับ
kubectl delete pod test-pod
```

---

## 8. iptables Rules ใน Kubernetes {#iptables-k8s}

### 8.1 Overview ของ iptables Chains

```
┌─────────────────────────────────────────────────────────────────┐
│                    iptables Flow ใน Kubernetes                   │
│                                                                   │
│  Incoming Packet                                                  │
│       │                                                           │
│       ▼                                                           │
│  PREROUTING chain                                                 │
│  ├── KUBE-SERVICES (ClusterIP DNAT)                              │
│  │   ├── KUBE-SVC-XXXX → KUBE-SEP-YYYY (endpoint 1)             │
│  │   └── KUBE-SVC-XXXX → KUBE-SEP-ZZZZ (endpoint 2)             │
│  └── KUBE-NODEPORTS (NodePort DNAT)                              │
│       │                                                           │
│       ▼                                                           │
│  FORWARD chain                                                    │
│  └── KUBE-FORWARD (forward packets)                              │
│       │                                                           │
│       ▼                                                           │
│  POSTROUTING chain                                                │
│  └── KUBE-POSTROUTING (SNAT/Masquerade)                         │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 ดู iptables Rules ที่ kube-proxy สร้าง

```bash
# ดู NAT table ทั้งหมด
iptables -t nat -L -n --line-numbers

# ดู KUBE-SERVICES chain
iptables -t nat -L KUBE-SERVICES -n --line-numbers
# ตัวอย่าง output:
# num   target     prot  opt  source    destination
# 1     KUBE-SVC-ERIFXISMVVYYFZ3NK tcp  -- 0.0.0.0/0  10.96.0.10  /* kube-dns:dns-tcp */
# 2     KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  -- 0.0.0.0/0  10.96.0.1   /* default/kubernetes:https */

# ดู Service-specific chain
SERVICE_CHAIN=$(iptables -t nat -L KUBE-SERVICES -n | grep "my-service" | awk '{print $1}')
iptables -t nat -L $SERVICE_CHAIN -n --line-numbers

# ดู EndPoint chain (จริงๆ ว่า traffic ไปที่ไหน)
iptables -t nat -L KUBE-SEP-XXXXXXXXXXXXXXXX -n

# ดู KUBE-NODEPORTS chain
iptables -t nat -L KUBE-NODEPORTS -n

# ดู iptables stats (packet/byte counts)
iptables -t nat -L KUBE-SERVICES -n -v
```

### 8.3 Load Balancing ด้วย iptables

```
kube-proxy ใช้ statistic module ของ iptables สำหรับ load balancing:

Service มี 3 endpoints:
- Endpoint 1: probability 1/3 (33%)
- Endpoint 2: probability 1/2 ของที่เหลือ (33%)
- Endpoint 3: ที่เหลือทั้งหมด (33%)

Rules ที่สร้าง:
KUBE-SVC-XXXX
  → KUBE-SEP-1  statistic mode random probability 0.33333
  → KUBE-SEP-2  statistic mode random probability 0.50000
  → KUBE-SEP-3  (default - ที่เหลือ)
```

```bash
# ดู Load Balancing rules
iptables -t nat -L KUBE-SVC-ERIFXISMVVYYFZ3NK -n
# Output:
# KUBE-SEP-1  tcp  --  statistic mode random probability 0.33333333349
# KUBE-SEP-2  tcp  --  statistic mode random probability 0.50000000000
# KUBE-SEP-3  tcp  --

# นับ packets ที่ผ่านแต่ละ endpoint
watch -n 1 'iptables -t nat -L KUBE-SVC-XXXX -n -v'
```

### 8.4 IPVS Mode (แทน iptables)

```bash
# เปลี่ยน kube-proxy เป็น IPVS mode
kubectl edit configmap kube-proxy -n kube-system
# เปลี่ยน mode: "" เป็น mode: "ipvs"

# หรือ patch
kubectl patch configmap kube-proxy -n kube-system \
  --type merge \
  -p '{"data":{"config.conf":"...\nmode: ipvs\n..."}}'

# ดู IPVS rules
ipvsadm -l -n
# Output:
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port    Forward Weight ActiveConn InActConn
# TCP  10.96.100.1:80 rr
#   -> 192.168.1.2:8080      Masq    1      0          0
#   -> 192.168.2.3:8080      Masq    1      0          0

# ดู IPVS stats
ipvsadm -l -n --stats

# ดู IPVS connections
ipvsadm -l -n -c

# ดู timeout settings
ipvsadm -l --timeout
```

---

## 9. Network Troubleshooting Tools {#network-troubleshooting}

### 9.1 tcpdump

```bash
# Capture ทุก traffic บน interface
tcpdump -i eth0 -w capture.pcap

# Capture traffic เฉพาะ Pod IP
POD_IP=192.168.1.10
tcpdump -i any host $POD_IP -n

# Capture HTTP traffic
tcpdump -i eth0 'tcp port 80' -A

# Capture DNS queries
tcpdump -i any 'port 53' -n

# Capture traffic ระหว่าง 2 hosts
tcpdump -i eth0 'host 192.168.1.10 and host 192.168.2.20' -n

# Capture VXLAN traffic (Flannel)
tcpdump -i eth0 'udp port 8472' -n

# Run tcpdump inside Pod
kubectl exec -it my-pod -- tcpdump -i eth0 -n

# Run tcpdump ด้วย netshoot image
kubectl run tcpdump --rm -it --image=nicolaka/netshoot \
  --overrides='{"spec":{"nodeName":"node1","hostNetwork":true}}' \
  -- tcpdump -i any 'host 192.168.1.10' -n

# Filter และแสดง output ที่อ่านได้
tcpdump -i eth0 -A -s 0 'tcp port 80 and (tcp[13] & 8 != 0)'
```

### 9.2 netstat และ ss

```bash
# ดู listening ports ทั้งหมด
netstat -tlnp
ss -tlnp

# ดู established connections
netstat -tnp | grep ESTABLISHED
ss -tnp state established

# ดู connections ไป specific service
ss -tnp | grep ':8080'
netstat -tnp | grep ':8080'

# ดู socket statistics
ss -s
# Output:
# Total: 500 (kernel 510)
# TCP:   300 (estab 200, closed 50, orphaned 5, synrecv 0, timewait 45)

# ดู connections จาก Pod
kubectl exec -it my-pod -- netstat -tnp
kubectl exec -it my-pod -- ss -tnp

# ดู UDP sockets
ss -unp

# Monitor connections realtime
watch -n 1 'ss -s'

# ดู connection states
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# ดู connections ที่ TIME_WAIT (อาจมีปัญหา)
ss -tan state time-wait | wc -l
```

### 9.3 ping, traceroute, mtr

```bash
# ทดสอบ connectivity พื้นฐาน
kubectl exec -it pod-a -- ping 10.244.1.5 -c 4
kubectl exec -it pod-a -- ping my-service.default.svc.cluster.local -c 4

# traceroute - ดู path ของ packets
kubectl exec -it pod-a -- traceroute 10.244.2.10
kubectl exec -it pod-a -- tracepath google.com

# mtr - combine ping + traceroute
kubectl exec -it pod-a -- mtr --report google.com

# ดู latency ระหว่าง pods
kubectl exec -it pod-a -- ping pod-b.default.svc.cluster.local -c 100 | tail -2
```

### 9.4 nslookup และ dig

```bash
# ทดสอบ DNS resolution
kubectl exec -it my-pod -- nslookup kubernetes.default
kubectl exec -it my-pod -- nslookup my-service.production.svc.cluster.local

# dig - ละเอียดกว่า
kubectl exec -it my-pod -- dig my-service.default.svc.cluster.local
kubectl exec -it my-pod -- dig my-service.default.svc.cluster.local +short

# ดู DNS server ที่ Pod ใช้
kubectl exec -it my-pod -- cat /etc/resolv.conf

# ทดสอบ external DNS
kubectl exec -it my-pod -- dig google.com @8.8.8.8

# ตรวจสอบ reverse DNS
kubectl exec -it my-pod -- dig -x 10.96.100.1
```

### 9.5 curl และ wget สำหรับ HTTP Testing

```bash
# ทดสอบ HTTP service
kubectl exec -it my-pod -- curl http://my-service.default.svc.cluster.local

# ดู response headers
kubectl exec -it my-pod -- curl -I http://my-service

# ดู timing ละเอียด
kubectl exec -it my-pod -- curl -w "
dns_lookup:     %{time_namelookup}s
connect:        %{time_connect}s
appconnect:     %{time_appconnect}s
pretransfer:    %{time_pretransfer}s
redirect:       %{time_redirect}s
starttransfer:  %{time_starttransfer}s
total:          %{time_total}s
" -o /dev/null -s http://my-service

# ทดสอบ HTTPS
kubectl exec -it my-pod -- curl -k https://my-service

# Test LoadBalancer
curl -v http://<EXTERNAL-IP>:80

# wget
kubectl exec -it my-pod -- wget -qO- http://my-service/health
```

### 9.6 Network Bandwidth Testing

```bash
# ใช้ iperf3 ทดสอบ bandwidth
# Server pod
kubectl run iperf-server --image=networkstatic/iperf3 -- -s -p 5201

# Client pod
kubectl run iperf-client --rm -it --image=networkstatic/iperf3 \
  -- -c iperf-server -p 5201 -t 30 -P 4

# ทดสอบ UDP bandwidth
kubectl run iperf-client --rm -it --image=networkstatic/iperf3 \
  -- -c iperf-server -u -b 1G -t 30

# ดู network stats บน node
cat /proc/net/dev
ip -s link show eth0

# netperf (alternative)
kubectl run netperf-server --image=henrylv206/k8s-netperf -- netserver
```

---

## 10. Workshop: Debug Pod Networking {#workshop-debug}

### Workshop Objectives

ในเวิร์กช็อปนี้ จะฝึก debug ปัญหา network จริงๆ ที่พบบ่อยใน Kubernetes

### Scenario 1: Pod ไม่สามารถ reach อีก Pod ได้

```bash
# สร้าง environment สำหรับทดสอบ
kubectl create namespace debug-lab

# สร้าง pods
kubectl run server --image=nginx -n debug-lab
kubectl run client --image=nicolaka/netshoot -n debug-lab -- sleep 3600

# รอ pods ready
kubectl wait --for=condition=Ready pod/server pod/client -n debug-lab --timeout=60s

# Step 1: ตรวจสอบ Pod IPs
kubectl get pods -n debug-lab -o wide

# Step 2: ทดสอบ connectivity
SERVER_IP=$(kubectl get pod server -n debug-lab -o jsonpath='{.status.podIP}')
kubectl exec -it client -n debug-lab -- ping $SERVER_IP -c 4

# Step 3: ตรวจสอบ Service
kubectl expose pod server --port=80 -n debug-lab
kubectl exec -it client -n debug-lab -- curl http://server.debug-lab.svc.cluster.local

# Step 4: Debug DNS
kubectl exec -it client -n debug-lab -- nslookup server.debug-lab.svc.cluster.local

# Cleanup
kubectl delete namespace debug-lab
```

### Scenario 2: Service ไม่ได้รับ Traffic

```bash
# สร้าง deployment และ service ที่มีปัญหา (label mismatch)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web      # label บน Pod
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app    # ← label ไม่ตรงกับ Pod!
  ports:
  - port: 80
    targetPort: 80
EOF

# Debug: ตรวจสอบ endpoints
kubectl get endpoints web-service
# ถ้า Endpoints แสดง <none> แสดงว่า label ไม่ match

# วิธีแก้: ดู labels บน pods
kubectl get pods --show-labels | grep web

# แก้ service selector
kubectl patch service web-service -p '{"spec":{"selector":{"app":"web"}}}'

# ตรวจสอบอีกครั้ง
kubectl get endpoints web-service

# Cleanup
kubectl delete deployment web-app
kubectl delete service web-service
```

### Scenario 3: Network Policy Blocking Traffic

```bash
# สร้าง environment
kubectl create namespace policy-lab
kubectl label namespace policy-lab tier=production

# สร้าง pods
kubectl run backend --image=nginx -n policy-lab -l app=backend
kubectl run frontend --image=nicolaka/netshoot -n policy-lab -l app=frontend -- sleep 3600
kubectl run attacker --image=nicolaka/netshoot -n policy-lab -l app=attacker -- sleep 3600

# สร้าง service
kubectl expose pod backend --port=80 -n policy-lab

# ทดสอบ: ตอนนี้ทุกคน reach ได้
kubectl exec -it frontend -n policy-lab -- curl backend.policy-lab.svc.cluster.local
kubectl exec -it attacker -n policy-lab -- curl backend.policy-lab.svc.cluster.local

# สร้าง Network Policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: policy-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend   # อนุญาตเฉพาะ frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# ทดสอบ: frontend ยังใช้ได้
kubectl exec -it frontend -n policy-lab -- curl backend.policy-lab.svc.cluster.local

# ทดสอบ: attacker ถูกบล็อก
kubectl exec -it attacker -n policy-lab -- curl --connect-timeout 5 backend.policy-lab.svc.cluster.local
# ควรจะ timeout

# Debug: ดู Network Policies
kubectl get networkpolicies -n policy-lab
kubectl describe networkpolicy backend-policy -n policy-lab

# Cleanup
kubectl delete namespace policy-lab
```

### Scenario 4: DNS Resolution ล้มเหลว

```bash
# Debug DNS ทีละขั้นตอน

# 1. ตรวจสอบ CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=30

# 2. ตรวจสอบ CoreDNS service
kubectl get service kube-dns -n kube-system

# 3. สร้าง debug pod
kubectl run dns-debug --rm -it --image=nicolaka/netshoot -- bash

# ภายใน pod:
# 4. ดู resolv.conf
cat /etc/resolv.conf
# Expected: nameserver 10.96.0.10

# 5. ทดสอบ DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup kubernetes.default
nslookup kubernetes

# 6. ถ้า DNS ล้มเหลว ตรวจสอบ CoreDNS
kubectl describe configmap coredns -n kube-system

# 7. ทดสอบ DNS โดยตรง
dig @10.96.0.10 kubernetes.default.svc.cluster.local

# 8. ดู DNS latency
time nslookup google.com

# แก้ปัญหา CoreDNS Loop
kubectl edit configmap coredns -n kube-system
# เพิ่ม: loop (disable loop detection)

# หรือ restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system
```

---

## 11. แบบฝึกหัด Kubernetes Networking {#exercises}

### แบบฝึกหัดที่ 1: ตรวจสอบ CNI Configuration

**วัตถุประสงค์:** เข้าใจ CNI ที่ cluster ใช้งานอยู่

**คำสั่ง:**
```bash
# 1. หา CNI ที่ใช้งานอยู่
ls /etc/cni/net.d/

# 2. อ่าน CNI configuration
cat /etc/cni/net.d/*.conf* 2>/dev/null || cat /etc/cni/net.d/*.json 2>/dev/null

# 3. ดู Pod CIDR ของแต่ละ Node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# 4. ดู CNI binaries
ls /opt/cni/bin/

# 5. ดูว่า CNI DaemonSet ชื่ออะไร
kubectl get daemonsets --all-namespaces | grep -E 'calico|flannel|cilium|weave|canal'
```

**คำถาม:**
- Cluster ของคุณใช้ CNI อะไร?
- Pod CIDR range คืออะไร?
- CNI binary ไหนที่มีอยู่?

---

### แบบฝึกหัดที่ 2: ทดสอบ Pod-to-Pod Communication

**วัตถุประสงค์:** ยืนยันว่า Pods สามารถคุยกันได้ทั้งบน Node เดียวกันและต่าง Node

```bash
# สร้าง 2 pods บน nodes ต่างกัน
NODE1=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
NODE2=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')

kubectl run pod1 --image=nicolaka/netshoot \
  --overrides="{\"spec\":{\"nodeName\":\"$NODE1\"}}" \
  -- sleep 3600

kubectl run pod2 --image=nicolaka/netshoot \
  --overrides="{\"spec\":{\"nodeName\":\"$NODE2\"}}" \
  -- sleep 3600

# รอ pods ready
kubectl wait --for=condition=Ready pod/pod1 pod/pod2 --timeout=60s

# ดู IPs
kubectl get pods -o wide

# ทดสอบ connectivity
POD2_IP=$(kubectl get pod pod2 -o jsonpath='{.status.podIP}')
kubectl exec pod1 -- ping $POD2_IP -c 4

# ดู route ที่ใช้
kubectl exec pod1 -- traceroute $POD2_IP

# Cleanup
kubectl delete pod pod1 pod2
```

---

### แบบฝึกหัดที่ 3: วิเคราะห์ iptables Rules

**วัตถุประสงค์:** เข้าใจว่า kube-proxy สร้าง iptables rules อย่างไร

```bash
# สร้าง service
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80 --target-port=80

# ดู ClusterIP ของ service
CLUSTER_IP=$(kubectl get service web -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $CLUSTER_IP"

# ดู iptables rules สำหรับ service นี้
iptables -t nat -L KUBE-SERVICES -n | grep $CLUSTER_IP

# หา chain ของ service
SVC_CHAIN=$(iptables -t nat -L KUBE-SERVICES -n | grep $CLUSTER_IP | awk '{print $1}')
echo "Service Chain: $SVC_CHAIN"

# ดู endpoints ใน chain นั้น
iptables -t nat -L $SVC_CHAIN -n --line-numbers

# ดู endpoint chain (จะเห็น real pod IP)
EP_CHAIN=$(iptables -t nat -L $SVC_CHAIN -n | grep KUBE-SEP | head -1 | awk '{print $1}')
iptables -t nat -L $EP_CHAIN -n

# Cleanup
kubectl delete deployment web
kubectl delete service web
```

---

### แบบฝึกหัดที่ 4: สร้างและทดสอบ Network Policy

**วัตถุประสงค์:** ควบคุม traffic ด้วย Network Policy

```bash
# Setup
kubectl create namespace netpol-test

# สร้าง backend
kubectl create deployment backend --image=nginx -n netpol-test
kubectl expose deployment backend --port=80 -n netpol-test

# สร้าง frontend และ other
kubectl run frontend --image=nicolaka/netshoot -n netpol-test -l tier=frontend -- sleep 3600
kubectl run other --image=nicolaka/netshoot -n netpol-test -l tier=other -- sleep 3600

# รอทุก pod ready
kubectl wait --for=condition=Ready pods --all -n netpol-test --timeout=60s

# ทดสอบก่อนมี policy (ทุกคน reach ได้)
kubectl exec frontend -n netpol-test -- curl -s --max-time 5 backend.netpol-test.svc.cluster.local | head -5
kubectl exec other -n netpol-test -- curl -s --max-time 5 backend.netpol-test.svc.cluster.local | head -5

# สร้าง Network Policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# ทดสอบหลังมี policy
echo "Frontend (ควรสำเร็จ):"
kubectl exec frontend -n netpol-test -- curl -s --max-time 5 backend.netpol-test.svc.cluster.local | head -3

echo "Other (ควร timeout):"
kubectl exec other -n netpol-test -- curl -s --max-time 5 backend.netpol-test.svc.cluster.local || echo "Blocked!"

# Cleanup
kubectl delete namespace netpol-test
```

---

## 12. เฉลยแบบฝึกหัด {#answers}

### เฉลยแบบฝึกหัดที่ 1

```bash
# ตัวอย่าง output สำหรับ Flannel cluster:
# /etc/cni/net.d/10-flannel.conflist

# Pod CIDR ตัวอย่าง:
# node1   10.244.0.0/24
# node2   10.244.1.0/24
# node3   10.244.2.0/24

# CNI binaries ที่ควรมี:
# /opt/cni/bin/flannel
# /opt/cni/bin/bridge
# /opt/cni/bin/host-local
# /opt/cni/bin/loopback
# /opt/cni/bin/portmap

# ตัวอย่าง output สำหรับ Calico:
# /etc/cni/net.d/10-calico.conflist
# DaemonSet: calico-node (namespace: calico-system หรือ kube-system)
```

### เฉลยแบบฝึกหัดที่ 2

```
ผลที่คาดหวัง:
- ping จาก pod1 ไป pod2 ควรสำเร็จ (0% packet loss)
- PING 10.244.2.5 (10.244.2.5) 56(84) bytes of data.
  64 bytes from 10.244.2.5: icmp_seq=1 ttl=62 time=0.512 ms
  64 bytes from 10.244.2.5: icmp_seq=2 ttl=62 time=0.432 ms

- traceroute แสดง path ผ่าน Node IP:
  1  10.244.0.1 (gateway บน node1)  0.123 ms
  2  192.168.1.11 (node2 IP)         0.456 ms  ← ผ่าน Node
  3  10.244.2.5 (pod2)               0.789 ms

หมายเหตุ: TTL=62 แสดงว่า packet ผ่าน 2 hops (pod1 → node1 → node2 → pod2)
```

### เฉลยแบบฝึกหัดที่ 3

```bash
# ClusterIP: 10.96.150.200 (ตัวอย่าง)

# KUBE-SERVICES output ตัวอย่าง:
# KUBE-SVC-LOLE4ISW44CQNEXN  tcp  -- 0.0.0.0/0  10.96.150.200  /* default/web:80 */

# Service Chain (KUBE-SVC-XXXX) output:
# KUBE-SEP-AAA  tcp  -- statistic mode random probability 0.33333
# KUBE-SEP-BBB  tcp  -- statistic mode random probability 0.50000
# KUBE-SEP-CCC  tcp  --

# Endpoint Chain output:
# DNAT tcp -- 0.0.0.0/0 0.0.0.0/0  to:192.168.1.10:80

# สังเกต: DNAT ชี้ไปที่ Pod IP โดยตรง
# ทั้ง 3 endpoints มี probability เท่ากันสำหรับ 3 replicas
```

### เฉลยแบบฝึกหัดที่ 4

```
ผลที่คาดหวัง:

Frontend curl:
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

Other curl:
curl: (28) Connection timed out after 5000 milliseconds
Blocked!

ทำไม timeout ไม่ใช่ Connection Refused?
- Network Policy ทำงานที่ level iptables/eBPF
- Packets ถูก drop (ไม่มี RST packet ส่งกลับ)
- ดังนั้น client เห็นเป็น timeout แทนที่จะเป็น refused

Debug เพิ่มเติม:
kubectl get networkpolicies -n netpol-test
kubectl describe networkpolicy backend-policy -n netpol-test
kubectl get endpoints backend -n netpol-test
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [CNI Specification](https://github.com/containernetworking/cni)
- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [Flannel GitHub](https://github.com/flannel-io/flannel)
- [Cilium Documentation](https://docs.cilium.io/)
- [Hubble - Cilium Observability](https://docs.cilium.io/en/stable/overview/intro/)
- [iptables Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
- [Linux Networking Tools](https://linux.die.net/man/8/ss)

---

*ต่อไป: [Part 32 - ClusterIP Service](./part-32-clusterip.md)*
