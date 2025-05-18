Introductie
In deze gids leg ik uit hoe je een Kubernetes-cluster opzet met hoge beschikbaarheid (High Availability, ofwel HA). We doen dat met de Stacked etcd-topologie: de etcd-database draait dan gewoon op dezelfde machines als de control planes, wat het eenvoudiger maakt en minder netwerkverkeer oplevert. We gebruiken containerd als container runtime en Calico voor de netwerkverbindingen.

Meer achtergrondinfo vind je op de officiële docs:
🔗 Kubernetes HA Topology Docs

✅ Wat heb je nodig?
Zorg dat je dit klaar hebt staan:

Minimaal 3 master nodes

Tenminste 2 worker nodes

Een Load Balancer (bv. HAProxy)

Ubuntu 22.04 of hoger op alle machines

Root-/sudo-toegang tot alles

Alle nodes moeten elkaar over het netwerk kunnen bereiken

1️⃣ HAProxy instellen (Load Balancer)
We gebruiken HAProxy om API-verkeer te verdelen over de masters.

Installeren:
bash
Copy
Edit
sudo apt update && sudo apt install -y haproxy
Configureren:
Bewerk dit bestand:

bash
Copy
Edit
sudo vim /etc/haproxy/haproxy.cfg
En voeg het volgende toe:

h
Copy
Edit
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
Start HAProxy:
bash
Copy
Edit
sudo systemctl restart haproxy
sudo systemctl enable haproxy
Test of het werkt:
bash
Copy
Edit
nc 192.168.2.10 6443 -v
2️⃣ Swap uitschakelen & kernel voorbereiden
Swap uitzetten (nodig voor kubelet):
bash
Copy
Edit
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
Kernel-instellingen voor Kubernetes:
bash
Copy
Edit
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
Vereiste kernelmodules:
bash
Copy
Edit
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
3️⃣ Containerd installeren
bash
Copy
Edit
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
sudo apt-get install -y containerd
Configureren:
bash
Copy
Edit
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
Pas dit aan in de config:

toml
Copy
Edit
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
Herstarten:
bash
Copy
Edit
sudo systemctl restart containerd
sudo systemctl enable containerd
4️⃣ Kubernetes installeren (kubeadm, kubelet, kubectl)
bash
Copy
Edit
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo tee /etc/apt/trusted.gpg.d/kubernetes.asc
echo "deb https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet
5️⃣ Eerste master-node initialiseren
Voer op de eerste master dit uit:

bash
Copy
Edit
sudo kubeadm init --control-plane-endpoint "192.168.2.10:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
Bewaar de join-commando's die worden gegenereerd.

kubectl instellen:
bash
Copy
Edit
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
6️⃣ Andere masters & workers toevoegen
Extra master-nodes:
Gebruik dit commando op master 2 en 3 (gebruik de gegevens van stap 5):

bash
Copy
Edit
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <cert-key>
Worker-nodes:
bash
Copy
Edit
kubeadm join 192.168.2.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
7️⃣ Calico deployen (netwerk-plugin)
bash
Copy
Edit
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
Controleren of alle nodes “Ready” zijn:
bash
Copy
Edit
kubectl get nodes
Indien nodig:

bash
Copy
Edit
sudo systemctl restart kubelet
8️⃣ ETCD-cluster controleren
ETCD client installeren:
bash
Copy
Edit
sudo apt update && sudo apt install -y etcd-client
Status van leden checken:
bash
Copy
Edit
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
🎉 Klaar!
Je HA Kubernetes-cluster is nu live. Check of alle nodes er goed bij staan:

bash
Copy
Edit
kubectl get nodes
Als er iets niet klopt (bijv. een node is “NotReady”), check je netwerk en herstart eventueel kubelet.

