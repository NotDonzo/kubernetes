# Kubernetes High Availability (HA) Cluster Setup Guide


## Prerequisites

Make sure you have the following ready before starting:

* At least **3 master nodes** (needed for HA)
* Minimum **2 worker nodes**
* A **load balancer** like HAProxy to balance traffic between API servers
* All nodes running **Ubuntu 22.04 or newer**
* **Root or sudo access** on every node
* All nodes should be able to communicate over the network

## Step 1: Load Balancer Setup (HAProxy)

We’ll install HAProxy to distribute the API requests evenly across all master nodes.

### Install HAProxy

```sh
sudo apt update && sudo apt install -y haproxy
```

### Configure HAProxy

Open the configuration file:

```sh
sudo vim /etc/haproxy/haproxy.cfg
```

Add these lines to define the frontend and backend:

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

Restart HAProxy and enable it to start on boot:

```sh
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

Check if HAProxy is running properly:

```sh
nc 192.168.2.10 6443 -v
```

## Step 2: Disable Swap and Configure Kernel Modules

Kubernetes requires swap to be off and some kernel modules enabled for networking.

### Turn off swap

```sh
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Enable necessary kernel settings

Create the sysctl config file:

```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

### Load kernel modules for containerd

```sh
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

## Step 3: Install Container Runtime (containerd)

We need containerd as the runtime for running containers.

```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
sudo apt-get install -y containerd
```

### Configure containerd

Set up the default config and modify it to use systemd for cgroups:

```sh
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

Change `SystemdCgroup` from `false` to `true`:

```ini
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
```

Restart containerd and enable it:

```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 4: Install Kubernetes Components

Now, install kubelet, kubeadm, and kubectl.

```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo tee /etc/apt/trusted.gpg.d/kubernetes.asc
echo "deb https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

## Step 5: Initialize Kubernetes Control Plane

On the main master node, start the control plane with this command:

```sh
sudo kubeadm init --control-plane-endpoint "192.168.2.10:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```

Make sure to save the join command output by kubeadm for adding other nodes later.

### Configure kubectl on the main master node

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 6: Add More Masters and Worker Nodes

### Adding more masters

Run this on each additional master node (replace tokens and cert keys accordingly):

```sh
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <cert-key>
```

### Adding worker nodes

Run this on each worker node:

```sh
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

## Step 7: Deploy Calico Network Plugin

Install Calico to handle networking inside your cluster:

```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

Check that all nodes are ready:

```sh
kubectl get nodes
```

If you find any issues, try restarting the kubelet:

```sh
sudo systemctl restart kubelet
```

## Step 8: Verify the ETCD Cluster

### Install etcd client

```sh
apt update && apt install -y etcd-client
```

### List etcd members

Run this command to check the health of your etcd cluster:

```sh
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## Conclusion

Your Kubernetes HA cluster is now ready, with multiple master and worker nodes managed by kubeadm, containerd, and Calico. Use the following command to check the status of your nodes:

```sh
kubectl get nodes
```

If any node shows as NotReady, restart kubelet and verify your network configuration.

---
