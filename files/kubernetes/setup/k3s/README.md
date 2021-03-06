[Back to Kubernetes Setup](../README.md)

# Setup a k3s Kubernetes Cluster

## Content
- [Install k3s](#k3s)
- [Install k3s on a Raspberry Pi](#k3s-rpi)
- [Setup an Ingress Controller](./ingress-controller.md)

<p id="k3s">

## Install k3s
Install k3s using the following command
```bash
curl -sfL https://get.k3s.io | sh -
```

Copy the cluster configuration file to your local `.kube` directory for `kubectl` to work properly
```bash
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

<p id="k3s-rpi">

## Install k3s on a Raspberry Pi
Append to `config.txt` in `/boot/` under the `[all]` section
```bash
arm_64bit=1
```
Append the following to `/boot/cmdline.txt`
```bash
cgroup_memory=1 cgroup_enable=memory
```

### Setup the Master
Download and enable iptables
```bash
$ sudo apt install iptables -y
$ sudo iptables -F
$ reboot
```
Install k3s on the master Raspberry Pi
```bash
$ sudo curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -s -
```
Get the masters' token to register new nodes
```bash
$ cat /var/lib/rancher/k3s/server/node-token
```

### Setup a Node
Download and enable iptables
```bash
$ sudo apt install iptables -y
$ sudo iptables -F
$ reboot
```
Use the master token and it's ip-address
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="K10516f26b65da18955c59d56f098fa0d1ea72efdd5dbb50738ad032f0e94dcc362::server:ef10b88cabf1a3aadeda2087b519faec" K3S_URL="https://192.168.3.151:6443" K3S_NODE_NAME="rpi-cluster-n02" sh -
```
curl -sfL https://get.k3s.io | K3S_TOKEN=K1047c4b2695e3c7e7ac9c7ce78c838c4a850273a432017d538bdf8c8cedcc95d95::server:b468e4fb70ec014c7e417f33d0b0fc3b K3S_URL="https://10.10.11.81:6443" K3S_NODE_NAME="worker-01-rpi4-4gb" sh -