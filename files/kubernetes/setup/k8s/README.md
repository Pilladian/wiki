[Back to Kubernetes Setup](../README.md)

# Setup a Kubernetes Cluster
> [Blog Post](https://github.com/srcmkr/kubernetes-on-hetzner)

```bash
# Turn off swap
swapoff -a; sed -i '/swap/d' /etc/fstab

# Networking
sysctl settings for Kubernetes networking
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

# Install Docker
apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io

# Install Kubernetes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt update && apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00

# Init Cluster (Master Only)
kubeadm init --apiserver-advertise-address=<public-IP> --pod-network-cidr=<private-network>  --ignore-preflight-errors=all

# Networking
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

# Create Command to join Cluster (Master Only)
kubeadm token create --print-join-command

# Configure kubectl
cd ~
cp /etc/kubernetes/admin.conf .kube/config
```