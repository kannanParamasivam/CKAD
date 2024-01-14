# CKAD

Certified Kubernetes Application Developer

## Playground Setup

![picture 0](images/e746b09f7f45a8aa2860edd6cf3957e0e2d85f8605b4d1543ddb1f6242e8be25.png)

## Set host name on the servers

```bash
sudo hostnamectl set-hostname k8s-control-plane
```

```bash
sudo hostnamectl set-hostname k8s-worker-1
```

```bash
sudo hostnamectl set-hostname k8s-worker-2
```

## Update host file

Do this in each node

```bash
~$ sudo vim /etc/hosts
```

Add the following lines

```bash
<ip address> k8s-control-plane
<ip address> k8s-worker-1
<ip address> k8s-worker-2
```

## Install containerd in each node

Do this is each node.

Create a file containerd.conf in /etc/modules-load.d/ so that the **modules are loaded on boot**.

> **Note:** The modules added to the /etc/modules-load/ location are loaded on system boot.

```bash
vim /etc/modules-load.d/containerd.conf
```

Add the folloiwing lines.

```bash
overlay
br_netfilter
```

Then start the modules immdiately with the following commands (so that you don't have to reboot the system for the modules to be loaded).

```bash
~$ sudo modprobe overlay
~$ sudo modprobe br_netfilter
```

Add the following network settings to **/etc/sysctl.d/99-kubernetes-cri.conf**.

> **Note:** The configurations that are added to sysctl.d are loaded on boot.

````bash

```bash
cloud_user@k8s-control:~$ cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
> net.bridge.bridge-nf-call-iptables
> net.ipv4.ip_forward
> net.bridge.bridge-nf-call-ipotables = 1
> EOF
````

Run the following command to load the new settings immediately.

```bash
>sudo sysctl --system
```

Install containerd apt package.

```bash
sudo apt-get update && sudo apt-get install -y containerd
```

Create directory for containerd configuration.

```bash
sudo mkdir -p /etc/containerd
```

Generate Containerd configuration and pipe it to a file.

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Restart containerd.

```bash
sudo systemctl restart containerd
```

## Install kubernetes packages on each node

Do this in each node.

It is needed to disable swap.

```bash
sudo swapoff -a
```

Install https and curl.

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```

Download the Google Cloud public signing key and add it to apt.

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

Add the Kubernetes apt repository.

```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
> deb https://apt.kubernetes.io/ kubernetes-xenial main
> EOF
```

Install Kubernetes packages.

```bash
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
```

Disable automatic updates for the kubernetest packages.

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```
