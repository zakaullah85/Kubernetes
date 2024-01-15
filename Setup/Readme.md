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
## Setup 1
Disable swap on each node master and worker node.
```bash
sudo swapoff -a
```
