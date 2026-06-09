Since you want a reusable shell script, here's a complete node bootstrap script that can be used on **Ubuntu 26.04** for both master and worker nodes. It:

* Disables swap
* Configures kernel modules
* Configures sysctl
* Installs containerd
* Configures `SystemdCgroup=true`
* Installs Kubernetes 1.33 components (`kubeadm`, `kubelet`, `kubectl`)
* Holds package versions
* Configures `/etc/hosts` with your cluster IPs

Save as `k8s-node-setup.sh`.

#!/bin/bash

set -e

echo "========== Kubernetes Node Bootstrap =========="

#---------------------------------------------------

# Update OS

#---------------------------------------------------
apt-get update
apt-get upgrade -y

#---------------------------------------------------

# Host Entries

#---------------------------------------------------
cat > /etc/hosts <<EOF
127.0.0.1 localhost
127.0.1.1 ubuntu

::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.65.200 c2-k8s-master
192.168.65.201 c2-k8s-worker1
192.168.65.202 c2-k8s-worker2
EOF

#---------------------------------------------------

# Disable Swap

#---------------------------------------------------
swapoff -a

sed -i '/swap.img/s/^/#/' /etc/fstab

#---------------------------------------------------

# Kernel Modules

#---------------------------------------------------
cat <<EOF >/etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

#---------------------------------------------------

# Sysctl Parameters

#---------------------------------------------------
cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF

sysctl --system

#---------------------------------------------------

# Required Packages

#---------------------------------------------------
apt-get install -y 
curl 
wget 
gnupg2 
apt-transport-https 
ca-certificates 
software-properties-common

#---------------------------------------------------

# Containerd

#---------------------------------------------------
apt-get install -y containerd

mkdir -p /etc/containerd

containerd config default > /etc/containerd/config.toml

sed -i 
's/SystemdCgroup = false/SystemdCgroup = true/' 
/etc/containerd/config.toml

systemctl daemon-reload
systemctl enable containerd
systemctl restart containerd

#---------------------------------------------------

# Kubernetes Repository

#---------------------------------------------------
mkdir -p /etc/apt/keyrings

curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key) 
| gpg --dearmor 
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.33/deb/](https://pkgs.k8s.io/core:/stable:/v1.33/deb/) /" \

> /etc/apt/sources.list.d/kubernetes.list

apt-get update

#---------------------------------------------------

# Kubernetes Components

#---------------------------------------------------
apt-get install -y kubelet kubeadm kubectl

apt-mark hold kubelet kubeadm kubectl

systemctl enable kubelet

echo ""
echo "=============================================="
echo "Bootstrap completed successfully"
echo "=============================================="
echo ""
echo "Verify:"
echo "  swapoff --show"
echo "  systemctl status containerd"
echo "  systemctl status kubelet"
echo ""
echo "For master node:"
echo "  kubeadm init --apiserver-advertise-address=192.168.65.200 --pod-network-cidr=192.168.0.0/16"
echo ""
EOF

### Run on all nodes

```bash
chmod +x k8s-node-setup.sh
sudo ./k8s-node-setup.sh
```

### Initialize Control Plane

On `c2-k8s-master`:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.65.200 \
  --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
```

Generate worker join command:

```bash
kubeadm token create --print-join-command
```

Run the generated `kubeadm join ...` command on `c2-k8s-worker1` and `c2-k8s-worker2`.

Verify:

```bash
kubectl get nodes -o wide
```

Expected:

```text
NAME             STATUS   ROLES           VERSION
c2-k8s-master    Ready    control-plane   v1.33.x
c2-k8s-worker1   Ready    <none>          v1.33.x
c2-k8s-worker2   Ready    <none>          v1.33.x
```
