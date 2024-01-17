# Kubernetes Multi-Note Setup using Kubeadm on x86/x64/arm64

This guide will walk you through setting up a multi-node Kubernetes cluster using kubeadm. The setup is suitable for x86, x64, or arm64 architectures.

Prerequisites

Before you begin, ensure that you have the following:

* Three or more machines running a compatible Linux distribution (Ubuntu 18.04/20.04, CentOS 7/8, etc.) with SSH access and the following specs:
  * Master at least 2 core CPU & 4GB RAM
  * Nodes at least 1 core CPU & 4GB RAM
* Firewalls configured to allow traffic between nodes.
* Root or sudo access to all machines.

This guide assumes to use Ubuntu Server 22.x.x. 
## Setup 1: 
Disable swap on each node master and worker node by running the following command in terminal.
```bash
sudo swapoff -a
```
Next, Open the /etc/fstab file with a text editor.
```bash
sudo nano /etc/fstab
```
Look for the line that references the swap file. It will usually look something like this:

> /swapfile none swap sw 0 0

Either comment the file by putting \# at the begining of the line or delete the entire line.
Finally, reboot your system to make the changes take place by running the following command:
```bash
sudo reboot
```
##Step 2:
Next you can give some human readable names to your nodes. for example, you can name the master node as "master-node" and the worker nodes as "worker-node1" and "woker-node2". Run the following command with the desired name on each node of the cluster to assign.

```bash
sudo hostname-ctl set-hostname 'master-node'
exec bash
````
Execute identical commands on the remaining nodes within this Kubernetes cluster, adjusting the hostname as needed. As an illustration, on a worker node, execute the command sudo hostnamectl set-hostname "worker-node1".

##Step 3:
Set up the IPV4 bridge on all nodes.To configure the IPV4 bridge on all nodes, run the subsequent command on each node.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl parameters needed for the setup must be configured, and these settings should persist through reboots.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# run sysctl params to avoid reboot
sudo sysctl --system
```

##Step 4:
Install kubelet, kubeadm, kubectl, docker on each node by running the subsequent commands:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir /etc/apt/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt install -y kubelet kubeadm kubectl
sudo apt install docker.io
sudo mkdir /etc/containerd
sudo sh -c "containerd config default > /etc/containerd/config.toml"
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd.service
sudo systemctl restart kubelet.service
sudo systemctl enable kubelet.service
```

##Step 5:
Initiaze the kubernetes cluster on the master node only by running the following command:

```bash
sudo kubeadm config images pull
```
