# 3-Node Kubernetes Cluster on RHEL 10

A complete step-by-step guide to deploy a production-ready 3-node Kubernetes cluster on **RHEL 10** using **kubeadm**, **containerd**, **Calico**, **MetalLB**, and **NGINX Ingress Controller**.

---

## 🌟 Cluster Specification

| Role          | Hostname      | IP Address          |
|---------------|---------------|---------------------|
| Control Plane | k8s-master    | `<master-ip>`       |
| Worker Node   | k8s-node1     | `<node1-ip>`        |
| Worker Node   | k8s-node2     | `<node2-ip>`        |

**Technology Stack (Latest Stable - May 2026)**

- **Kubernetes**: v1.36
- **Container Runtime**: containerd
- **CNI**: Calico v3.32.0
- **Load Balancer**: MetalLB v0.15.3
- **Ingress**: NGINX Ingress Controller v1.15.1

---

## 1. 🔧 Hostname & Network Configuration (All Nodes)

### 1.1 Set Hostnames
```bash
# On Master
sudo hostnamectl set-hostname k8s-master

# On Node1
sudo hostnamectl set-hostname k8s-node1

# On Node2
sudo hostnamectl set-hostname k8s-node2

exec bash
```

### 1.2 Configure Static IP
```bash
nmcli device status                    # Note your interface name (e.g. ens160)
```

```bash
sudo nmcli connection modify <interface-name> \
  ipv4.method manual \
  ipv4.addresses <your-node-ip>/24 \
  ipv4.gateway <your-gateway-ip> \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  connection.autoconnect yes

sudo nmcli connection down <interface-name> && sudo nmcli connection up <interface-name>
```

### 1.3 Update /etc/hosts (All Nodes)
```bash
sudo tee -a /etc/hosts <<EOF

# Kubernetes Cluster Nodes
<master-ip>   k8s-master
<node1-ip>    k8s-node1
<node2-ip>    k8s-node2
EOF
```

**Reboot all nodes**:
```bash
sudo reboot
```

---

## 2. 🛠 System Preparation (All Nodes)

```bash
sudo dnf update -y

# Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kernel Modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl Parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

**Firewall (Recommended)**
```bash
sudo firewall-cmd --permanent --add-port={6443,2379-2380,10250,10259,10257}/tcp
sudo firewall-cmd --reload
```

---

## 3. 📦 Install containerd (All Nodes)

```bash
# Add Docker repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Install containerd
sudo dnf install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Enable systemd cgroup driver (Required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Start service
sudo systemctl enable --now containerd
sudo systemctl status containerd
```

---

## 4. ☸️ Install Kubernetes Components (All Nodes)

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## 5. 🚀 Initialize Control Plane (Only on k8s-master)

```bash
sudo kubeadm config images pull

sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<master-ip> \
  --control-plane-endpoint=k8s-master
```

**Configure kubeconfig**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> Save the `kubeadm join` command shown at the end.

---

## 6. 🔗 Join Worker Nodes (k8s-node1 & k8s-node2)

Execute the `kubeadm join` command on both worker nodes.

---

## 7. 🌐 Install Calico CNI (On Master)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml

watch kubectl get tigerastatus
```

---

## 8. ⚖️ Install MetalLB (On Master)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

**Create `metallb-config.yaml`**:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - <your-external-ip-range>     # Example: 10.0.0.200-10.0.0.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

```bash
kubectl apply -f metallb-config.yaml
```

---

## 9. 🚪 Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer
```

---

## 10. ✅ Verification & Testing

```bash
kubectl get nodes
kubectl get pods --all-namespaces

kubectl get pods -n calico-system
kubectl get pods -n metallb-system
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Test LoadBalancer**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/test/e2e/manifests/simple-service.yaml
kubectl get svc -w
```

---

## 📚 Official Documentation

- [Kubernetes + kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [containerd](https://containerd.io/)
- [Calico](https://docs.tigera.io/calico/latest/)
- [MetalLB](https://metallb.universe.tf/)
- [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx)

---

**🎉 Your Kubernetes cluster is now ready for use!**

You can start deploying applications using Deployments, Services, and Ingress resources.
