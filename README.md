Exactly! Here's your entire Kubernetes HA guide with **all command blocks formatted like your example** using triple backticks and `sh` for shell commands:

---

# Kubernetes High Availability (HA) Cluster Setup Guide

## Introduction

This guide provides detailed instructions for setting up a **Kubernetes High Availability (HA) Cluster** using the **Stacked etcd** architecture. In this configuration, `etcd` runs on the same nodes as the control plane, which streamlines the deployment and minimizes latency. You’ll set up **both master and worker nodes**, install **containerd** as the container runtime, and use `kubeadm` to bootstrap the cluster. For the CNI (Container Network Interface), we'll use **Calico**.

---

## Prerequisites

Make sure the following conditions are met before proceeding:

* **At least 3 master nodes** (to ensure high availability)
* **Minimum of 2 worker nodes**
* **A load balancer** (e.g., HAProxy) to balance API server traffic
* **Ubuntu 22.04 or newer** installed on every node
* **Root or sudo access** on all machines
* **Network connectivity** established among all nodes

---

## Step 1: Load Balancer Setup (HAProxy)

We use **HAProxy** to distribute incoming API requests across the master nodes evenly.

### Install HAProxy

```sh
sudo apt update && sudo apt install -y haproxy
```

### Configure HAProxy

Open the HAProxy configuration file:

```sh
sudo vim /etc/haproxy/haproxy.cfg
```

Add the following configuration:

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

Restart and enable HAProxy to apply changes:

```sh
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

Check if HAProxy is listening on the correct port:

```sh
nc 192.168.2.10 6443 -v
```

---

## Step 2: Disable Swap and Configure Kernel Modules

### Disable Swap

Turning off swap is necessary because Kubernetes requires swap to be disabled.

```sh
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Configure Kernel Parameters

Set kernel parameters required for Kubernetes networking:

```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

### Load Required Kernel Modules

Make sure these kernel modules are loaded to support container networking:

```sh
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

---

## Step 3: Install Container Runtime (containerd)

Update your package lists and install containerd:

```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
sudo apt-get install -y containerd
```

### Configure containerd

Create the configuration directory and generate default config:

```sh
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

Within the config file, change:

```ini
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
```

Save and exit.

Restart and enable containerd:

```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Step 4: Install Kubernetes Components

Add Kubernetes repo keys, update, and install necessary packages:

```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo tee /etc/apt/trusted.gpg.d/kubernetes.asc
echo "deb https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

---

## Step 5: Initialize Kubernetes Control Plane

Run this on your primary master node to start the control plane, pointing to the load balancer IP:

```sh
sudo kubeadm init --control-plane-endpoint "192.168.2.10:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```

Save the output join command for adding more nodes later.

### Set up kubectl for the admin user

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 6: Join Additional Masters and Workers

### Join extra master nodes

On each additional master, run:

```sh
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <cert-key>
```

### Join worker nodes

On each worker node, run:

```sh
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Step 7: Deploy Network Plugin (Calico)

Apply Calico manifests to enable networking between pods:

```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

Check the status of all nodes:

```sh
kubectl get nodes
```

If any nodes are not ready, try restarting kubelet:

```sh
sudo systemctl restart kubelet
```

---

## Step 8: Verify ETCD Cluster

### Install etcd client tools

```sh
apt update && apt install -y etcd-client
```

### Check cluster membership

Run the following to see your etcd members:

```sh
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Conclusion

Your Kubernetes HA cluster is now operational with multiple master and worker nodes, managed by `kubeadm`, using `containerd` and networked via Calico. Confirm the cluster health by listing nodes:

```sh
kubectl get nodes
```

If any nodes are **NotReady**, restart the kubelet service and review your network settings.

---

If you want me to save this as a markdown file or add anything else, just say!
