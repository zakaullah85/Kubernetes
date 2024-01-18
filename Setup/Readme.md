# Kubernetes Multi-Node Cluster Setup using Kubeadm on x86/x64/arm64

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
## Step 2:
Next you can give some human readable names to your nodes. for example, you can name the master node as "master-node" and the worker nodes as "worker-node1" and "woker-node2". Run the following command with the desired name on each node of the cluster to assign.

```bash
sudo hostname-ctl set-hostname 'master-node'
exec bash
````
Execute identical commands on the remaining nodes within this Kubernetes cluster, adjusting the hostname as needed. As an illustration, on a worker node, execute the command sudo hostnamectl set-hostname "worker-node1".

## Step 3:
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

## Step 4:
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
These commands are used to set up and install Kubernetes components on a Linux system, particularly Ubuntu. Let's break down each step:

1. sudo apt-get update: Updates the local package database with the latest information about available packages.
2. sudo apt-get install -y apt-transport-https ca-certificates curl: Installs necessary packages for secure communication and downloading packages over HTTPS.
3. sudo mkdir /etc/apt/keyrings: Creates a directory to store keyring files for package verification.
4. curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg: Retrieves the GPG key for the Kubernetes packages, converts it to the keyring format, and saves it in the specified directory.
5. echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list: Adds the Kubernetes repository to the package manager's sources list.
6. sudo apt-get update: Updates the package database again to include the newly added Kubernetes repository.
7. sudo apt install -y kubelet kubeadm kubectl: Installs the Kubernetes components, namely kubelet, kubeadm, and kubectl.
8. sudo apt install docker.io: Installs the Docker container runtime.
9. sudo mkdir /etc/containerd: Creates a directory for the containerd
10. sudo sh -c "containerd config default > /etc/containerd/config.toml": Generates a default containerd configuration file.
11. sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml: Modifies the containerd configuration to enable systemd cgroups.
12. sudo systemctl restart containerd.service: Restarts the containerd service to apply the new configuration.
13. sudo systemctl restart kubelet.service: Restarts the kubelet service after installing Kubernetes components.
14. sudo systemctl enable kubelet.service: Enables the kubelet service to start automatically on system boot.

These commands collectively set up the required environment, install Kubernetes components, configure the container runtime, and ensure that the necessary services are running and configured properly for a Kubernetes cluster.

## Step 5:
Initiaze the kubernetes cluster on the master node only by running the following command. It will install several images neccessary to orchestrate a cluster.

```bash
sudo kubeadm config images pull
```
The command sudo kubeadm config images pull is used to download the container images required by kubeadm for a specific Kubernetes version. This command is commonly used in scenarios where you want to pre-pull the required container images before initializing a Kubernetes cluster.

Next initialize a cluster by running the following command.

```bash
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=172.16.50.142
```
This command initiates the process of creating a Kubernetes cluster, defining the pod network, and configuring the API server's advertised address. After running this command, the output will provide instructions on how to join worker nodes to the newly initialized control-plane node. Replace the --apisever-advertise-address with the ethernet IP of your master-node.

Futher, execute the following commands on master-node:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
these commands are part of the post-initialization steps for setting up Kubernetes on a single-node cluster after running sudo kubeadm init. They configure the user's environment to use the Kubernetes cluster, ensuring that the user has the necessary permissions and access to the cluster's configuration.

## Step 6:
Run the following commands on master-node to instsall calico operator:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/tigera-operator.yaml
```
Next download the resoure file for the calico operator:
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/custom-resources.yaml -O
```
Modify the CIDR in the downloaded resource file to match the pod network by executing the following command on master-node:
```bash
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml
```
By default the CIDR in the resource file is set to 192.168.0.0, whereas in this tutorial 10.10.0.0 is being used for pod network, therefore the above command replaces the default CIDR with the desired IP range.

Finally, Create the the resource definition using kubectl as given in the following command:
```bash
kubectl create -f custom-resources.yaml
``
It usually takes sometime to initialize the calico network pods. Run the following command to monitor the status of calico pods intialzation:
```bash
watch kubectl get pods -n calico-system
```
Once, all of the newtork pod are created and are in running status then the worker nodes can be joined to the control plane being setup.

## Step 7:
Now its time to join the worker nodes to the cluster. Every cluster being created with kubeadm requires a join command that needs to be executed on worker nodes in order to be part of the cluster. The joining command can be obtained by runnig the following command on the master-node:

```bash
sudo kubeadm token create --print-join-command
```
The command sudo kubeadm token create --print-join-command is used to generate and print the token and the corresponding join command that can be used to join additional nodes to a Kubernetes cluster.

Breaking down the command:

sudo: Runs the command with elevated privileges.
kubeadm token create: Generates a new token for authenticating nodes joining the cluster.
--print-join-command: Instructs kubeadm to print the full join command, including the token and other necessary parameters.
The output of this command typically looks something like:

> kubeadm join <Master-Node-IP>:<Master-Node-Port> --token <token> --discovery-token-ca-cert-hash <ca-cert-hash>

copy the command being displayed and run on each of the worker node in order to join it to the cluster.

## Step 8:
The status of the cluster can be varified usig the following command:
```bash
kubectl get nodes
```
So, when you run kubectl get no, it queries the Kubernetes cluster for information about the nodes, and it should output a list of the nodes along with their status, roles, and other relevant details. This command is useful for checking the health and status of the nodes in your Kubernetes cluster.

## Conclusion

This tutorial provided instructions on installing Kubernetes on Ubuntu 22.x.x and configuring a cluster. We discussed checks on system requirements, configuring hostnames, installing components, initiating the cluster, and validating the setup. By adhering to these procedures, you can establish a robust cluster. Kubernetes serves as a reliable platform for deploying microservices applications, making it a crucial asset for competitive businesses.
