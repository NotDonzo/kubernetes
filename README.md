# Kubernetes High Availability (HA) Cluster Setup Guide

## Introduction
This guide provides a step-by-step process for setting up a **Kubernetes High Availability (HA) Cluster** using the **Stacked etcd topology**. In this topology, the `etcd` database runs on the same nodes as the control plane, reducing network overhead and simplifying the setup. We will configure both **master and worker nodes**, set up a **container runtime (containerd)**, and deploy the **Kubernetes control plane** using `kubeadm`. For networking, we will use **Calico**.

Overview:

![image](https://github.com/user-attachments/assets/cd8de135-0012-45bf-b5fb-5ab59eef7530)

More info read k8s docs: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/

## Prerequisites
Before we begin, ensure that you meet the following requirements:

- **Minimum of 3 Master nodes** (for HA)
- **At least 2 Worker nodes**
- **A Load Balancer** (e.g., HAProxy) for distributing traffic to API servers
- **Ubuntu 22.04+ installed** on all nodes
- **Root or sudo privileges** on all nodes
- **Network connectivity** between all nodes

## Step 1: Load Balancer Setup (HAProxy)
We will use **HAProxy** to distribute API requests across multiple master nodes.

### Install HAProxy
```sh
sudo apt update && sudo apt install -y haproxy
```

### Configure HAProxy
Edit the configuration file:
```sh
sudo vim /etc/haproxy/haproxy.cfg
```
Add the following content:
```ini
frontend kubernetes-frontend
    bind 192.168.2.10:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1 192.168.2.11:6443 check
    server master-2 192.168.2.12:6443 check
    server master-3 192.168.2.13:6443 check
```
Restart HAProxy:
```sh
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```
Verify HAProxy is running:
```sh
nc 192.168.2.10 6443 -v
```

## Step 2: Disable Swap and Configure Kernel Modules

### Disable Swap
```sh
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Enable Kernel Modules for Kubernetes
```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

### Load Required Kernel Modules
```sh
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

## Step 3: Install Container Runtime (containerd)
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
sudo apt-get install -y containerd
```

### Configure containerd
```sh
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```
Modify the following line from false to true:

![image](https://github.com/user-attachments/assets/b79fbae1-8201-4b8b-b739-fe0a6f742a00)

```ini
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
```

Restart and enable containerd:
```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 4: Install Kubernetes Components
```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo tee /etc/apt/trusted.gpg.d/kubernetes.asc
echo "deb https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

## Step 5: Initialize Kubernetes Control Plane
Run the following command on the **primary master node**:
```sh
sudo kubeadm init --control-plane-endpoint "192.168.2.10:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```
After successful initialization, save the **join command** that kubeadm outputs.

### Configure kubectl for the Primary Master Node
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 6: Join Additional Masters and Workers

### Join Additional Master Nodes
Run the following command on each **additional master** node:
```sh
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <cert-key>
```

### Join Worker Nodes
Run the following command on each **worker node**:
```sh
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

## Step 7: Deploy Network Plugin (Calico)
```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```
Verify nodes are ready:
```sh
kubectl get nodes
```
If needed, restart kubelet:
```sh
sudo systemctl restart kubelet
```

## Step 8: Verify ETCD Cluster

### Install etcd client
```sh
apt update && apt install -y etcd-client
```

### Check the ETCD Members
```sh
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## Conclusion
Your **Kubernetes HA cluster** is now set up with multiple master and worker nodes using `kubeadm`, `containerd`, and `Calico`. Verify that all nodes are in a `Ready` state using:
```sh
kubectl get nodes
```
If any nodes are **NotReady**, restart the `kubelet` service and check your networking setup.

### 🎉 Congratulations! Your Kubernetes cluster is now operational and highly available. Sudo Su Masterrr 🚀

