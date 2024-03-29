# Index
- [Index](#index)
- [Kubernetes Cluster Setup](#kubernetes-cluster-setup)
  - [Playground Setup](#playground-setup)
  - [Set host name on the servers](#set-host-name-on-the-servers)
  - [Update host file](#update-host-file)
  - [Install containerd in each node](#install-containerd-in-each-node)
  - [Install kubernetes packages on each node](#install-kubernetes-packages-on-each-node)
  - [Initialize the control plane (cluster)](#initialize-the-control-plane-cluster)
  - [Join the worker nodes to the cluster](#join-the-worker-nodes-to-the-cluster)
- [Major Concepts in CKAD](#major-concepts-in-ckad)
- [Application Design and Build](#application-design-and-build)
  - [Building Container Images](#building-container-images)
    - [Build image from Dockerfile](#build-image-from-dockerfile)
    - [Run the image](#run-the-image)
    - [Stop the container](#stop-the-container)
    - [Export the image to a file](#export-the-image-to-a-file)
  - [Job](#job)
  - [CronJob](#cronjob)
  - [Init Container](#init-container)
  - [Volume](#volume)
    - [HostPath Volume](#hostpath-volume)
    - [EmptyDir Volume](#emptydir-volume)
    - [Persistent Volume](#persistent-volume)
- [Application Deployment](#application-deployment)
  - [Rolling Update](#rolling-update)
  - [Deployment Strategies](#deployment-strategies)
  - [Helm](#helm)
    - [Install helm](#install-helm)
    - [Add a Helm repository](#add-a-helm-repository)
    - [search for a package in the repo](#search-for-a-package-in-the-repo)
    - [Install a package](#install-a-package)
    - [Uninstall a package](#uninstall-a-package)
- [Application Observability and Maintenance](#application-observability-and-maintenance)
  - [Deprecation Policy](#deprecation-policy)
  - [Probes and Health Checks](#probes-and-health-checks)
    - [Liveness Probe](#liveness-probe)
    - [Readiness Probe](#readiness-probe)
    - [Startup Probe](#startup-probe)
- [Application Observability and Maintenance](#application-observability-and-maintenance-1)
  - [Monitoring](#monitoring)
  - [Container Logging](#container-logging)
- [Application Environment, Configuration, and Security](#application-environment-configuration-and-security)
  - [Custom Resource Definition (CRD)](#custom-resource-definition-crd)
  - [Service Account](#service-account)
  - [Roles](#roles)
  - [Role Binding](#role-binding)
  - [Admission Controllers](#admission-controllers)
  - [Managing Compute Resources for Containers](#managing-compute-resources-for-containers)
    - [Requests \& Limits](#requests--limits)
    - [Resource Quotas](#resource-quotas)
  - [ConfigMaps and Secrets](#configmaps-and-secrets)
  - [Security Context](#security-context)
    - [User ID and Group ID](#user-id-and-group-id)
    - [Allow Privilege Escalation](#allow-privilege-escalation)
    - [Read Only Root Filesystem](#read-only-root-filesystem)
- [Services and Networking](#services-and-networking)
  - [Controlling Network Access with NetworkPolicies - Part 1](#controlling-network-access-with-networkpolicies---part-1)
    - [Kubernetes Networking](#kubernetes-networking)
    - [Network Policies](#network-policies)
    - [Isolated vs Non-Isolated Pods](#isolated-vs-non-isolated-pods)
- [note](#note)
# Kubernetes Cluster Setup

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

## Initialize the control plane (cluster)

Run this only in the control plane node.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.24.0
```

> `--pod-network-cidr` is the CIDR used by the pod network. This is needed for the pod network to work.

Once the command is finished it will show a command to genrate kube config file used to interact with the cluster.

Install the networking plugin. This is needed for the pods to communicate with each other.

```bash
sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Join the worker nodes to the cluster

Run this command on control plane node to generate the command to join the worker nodes to the cluster. Copy the command that is generated and run it on each worker node.

```bash
sudo kubeadm token create --print-join-command
```

It will take a few minutes for the nodes to join the cluster.
[Index](#index)
# Major Concepts in CKAD

![picture 1](images/67b376a99f23c0ac141e6d4266f59b0e831c433dff7f0e2d3fe2ee6050c3e82f.png)
[Index](#index)
# Application Design and Build
![picture 2](images/e31948ec8a5588efaac37f62b9ebb696db16bd08957ba89fcdd34f5ebbbad2ca.png) 

## Building Container Images
Docker file from [here](./create_ngnx_docker_image/Dockerfile)

### Build image from Dockerfile
```bash
docker build -t my-wesite:0.0.1 .
```
```bash
docker build -t <image name>:<tag> <path to docker file>
```
### Run the image
```bash
docker run --rm --name my-website -d -p 8080:80 my-website:0.0.1
```
```bash
docker run --rm --name <container name> -d -p <host port>:<container port> <image name>:<tag>
```
now if you go to http://localhost:8080 you will see the website running.

### Stop the container
```bash
docker stop my-website
```
```bash
docker stop <container name>
```
### Export the image to a file
```bash
docker save -o my-website_0.0.1.tar my-website:0.0.1
```
```bash
docker save -o <file name> <image name>:<tag>
``` 
[Index](#index)
## Job
A job creates one or more pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the job tracks the successful completions. When a specified number of successful completions is reached, the job itself is complete. Deleting a Job will clean up the pods it created.

Here is the job definition file [job](./job/my-job.yml)

Apply the job definition file
```bash
kubectl apply -f my-job.yml
```
get jobs
```bash
kubectl get jobs
```
get pods
```bash
kubectl get pods
```
get logs of the pod
```bash
kubectl logs <pod name>
```
[Index](#index)
## CronJob
A Cron Job creates Jobs on a repeating schedule.
Here is the cron job definition file [cron job](./cronjob/my-cronjob.yml)
Apply the cron job definition file.
```bash
kubectl apply -f my-cronjob.yml
```
list cron jobs
```bash
kubectl get cronjobs
```
## Init Container
An init container is a container that runs before the main container. 

* The init container will *run to completion before the main container starts*. If the init container fails, the pod will not start.
* It uses the different image than the main container.
* It can be used to perform *initialization logic* such as creating directories, downloading files, etc. 
* This can be used to *delay the start of the main container* until certain conditions are met.
* Init Containers can perform *sensitive initialization* logic such as fetching secrets from a vault in isolation from the main container.

"Here is an example of an init container [init container](./init-container/init-container.yml). In this scenario, the main container will initiate once the init container completes its sleep cycle of one minute."

![picture 4](images/333155bf2b7990fa8026d6a032c9aa2eb2ffbe024f78a7999d5007b6c39c44b5.png)  

## Volume
A volume is a directory that is accessible to all containers in a pod. It is used to share data between containers in a pod.

### HostPath Volume
A hostPath volume mounts a file or directory from the host node's filesystem into your pod. This is not something that most Pods will need, but it offers a powerful escape hatch for some applications.

[Here](./volume-hostpath/hostpath-volume.yml) is an example of a hostPath volume. Before trying this example, create a file with the location /etc/hostname in the worker node. 

### EmptyDir Volume
EmptyDir volumes are created when a Pod is assigned to a Node, and they are deleted when the Pod is evicted from the Node for any reason.

[Here](./volume-emptydir/emptydir-volume.yml) is an example of an emptyDir volume. In this example, the init container creates a file in the emptyDir volume and the main container reads the file from the emptyDir volume.

### Persistent Volume
Persistent Volumes allows you to abstract volume storage details from the pod. It is a cluster-wide resource that you can use to store data in a way that is independent of any single pod. This treats storage like a consumable resource that can be requested by applications.
![picture 3](images/4d24a4c2afec7da70fcd5f596ca37b0b0c4424919c472123667fcd5f658e78ad.png)  

[Here](./volume-persistentVolume/pv-pod-test.yml) is the pod uses the persistent volume. 

# Application Deployment

[Here](./Deployment/simple-deployment.yml) is a simple deployment file.

Command to Sale the deployment up/down
```shell
kubectl scale deployment <deployment name> --replicas=<number of replicas>
```
You can directly edit the deployment file and apply it again. While editing you can update the replica count as well.
```shell
kubectl edit deployment <deployment name>
```

## Rolling Update
Rolling update is the default update strategy. It updates the pods one by one. It will create a new pod with the new image and then delete the old pod. This will ensure that the application is always available.

You can use rolling update to deploy a new version of the application.

Whenever you change the deployment file and apply it again, it will trigger a rolling update.

Command to update the image of the deployment
```shell
kubectl set image deployment/<deployment name> <container name>=<new image name>
```
Command to update image for all containers in a deployment
```shell
kubectl set image deployment/<deployment_name> *=<new_image>
```

Get rollout status
```shell
kubectl rollout status deployment/<deployment name>
```
Get rollout history
```shell
kubectl rollout history deployment/<deployment name>
```

Undo the rollout to previous version
```shell
kubectl rollout undo deployment/<deployment name>
```

## Deployment Strategies
A Deployment Strategy is a methhod of rolling out new code that is used to achieve some benefit, such as increasing reliability and minimizing risk.

## Helm
Helm is a package manager for Kubernetes. It is used to install applications on Kubernetes. It is like apt or yum for Kubernetes.

### Install helm
```shell
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
```
```shell
apt-get install helm
```

### Add a Helm repository
```shell
helm repo add <repo name> <repo url>
```
sample bitnami repo
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### search for a package in the repo
```shell
helm search repo <repo name> <package name>
```
```shell
helm search repo bitnami
```

### Install a package
```shell
helm install <package name> -n <namespace name> <name to the application> <repo name>/<package name>
```
```shell  
helm install --set persistence.enabled=false -n dokuwiki dokuwiki bitnami/dokuwiki
```

### Uninstall a package
```shell
helm uninstall -n <namespace name> <package name>
```
```shell
helm uninstall -n dokuwiki dokuwiki
```

# Application Observability and Maintenance

## Deprecation Policy
Kubernetes removes a deprecated feature that is GA (General Availability) in 12 months or next 3 releases, whichever is longer.

## Probes and Health Checks
Probes are used to check the health of the application. There are three types of probes.

### Liveness Probe
Liveness probe is used to check if the application is alive. If the liveness probe fails, the pod will be restarted.

[Here](./liveness-probe/liveness-probe.yml) is an example of a liveness probe.

`initialDelaySeconds` is the time to wait before starting the probe. This is needed because the application may take some time to start.

`periodSeconds` is the time between two probes.

### Readiness Probe
Readiness probe is used to check if the application is ready to receive traffic. If the readiness probe fails, the pod will be removed from the service.

[Here](./readiness-probe/readiness-probe-test.yml) is an example of a readiness probe.

`initialDelaySeconds` is the time to wait before starting the probe. This is needed because the application may take some time to start. Old pod will be removed from the service only after the new pod is ready to receive traffic.

`periodSeconds` is the time between two probes.

### Startup Probe
Startup probe is used to check if the application is ready to receive traffic. If the startup probe fails, the pod will be restarted. It is used in slow starting applications to check if container is healthy during the extended startup period.

# Application Observability and Maintenance

## Monitoring
Monitoring is the process of collecting metrics and events from the cluster and applications. It is used to detect and diagnose problems.

Command to view the metrics of the cluster
```shell
kubectl top node(s)/pod(s) -n <namespace name wich is optional>
```

Following manifest deploys a pod that consumes resources. Once applied you can run `kubectl top pods` to see the resource consumption.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-consumer-pod
spec:
  containers:
  - name: resource-consumer-container
    image: ubuntu:latest
    command:
    - "/bin/sh"
    - "-c"
    - "apt-get update && apt-get install -y stress && stress --cpu 1 --vm-bytes 128M --vm-keep"

```

## Container Logging
Kubernetes stores the stdout and stderr console output for each container in the container log. You can view the logs using the following command. You can use `-f` flag to view the logs in real time.

```shell
kubectl logs <pod name> -n <namespace name>
```

In multicountainer pods, you can specify the container name to view the logs of a specific container. If you have more than one container in a pod, you need to specify the container name else it will show an error.

```shell
kubectl logs <pod name> -n <namespace name> -c <container name>
```

All System and user defined pod logs will be at **`/var/log/pods/`** location of the control plane node. You can use `kubectl describe pod <pod name> -n <namespace name>` to get the location of the log file.

To view kubelet logs, use the following command.

```shell
sudo journalctl -u kubelet
```

To get pods from all namespaces, use the following command.

```shell
kubectl get pods --all-namespaces
```

# Application Environment, Configuration, and Security

## Custom Resource Definition (CRD)

Custom Resource Definitions (CRDs) are extensions of the Kubernetes API. Once created, they can be used like any other resource in the system. They can be created using the `kubectl apply` command.

[Here](./crd/pdf-crd.yml) is an example of a CRD.

Command to create CRD
```shell
kubectl apply -f pdf-crd.yml
```

Commd to get api resources
```shell
kubectl api-resources
```

[Here](./crd/pdfdocument.yaml) is an example of a custom resource usage.

## Service Account
Service account is used to authenticate the pods to the Kubernetes API server. It is used to control the permissions of the pods.

A **token** is mounted to the pod at **`/var/run/secrets/kubernetes.io/serviceaccount/token`**. This token is used to authenticate the pod to the Kubernetes API server.

![picture 5](images/6f86a0c43e72b136c2b341d64ff1240d2e25aab9b11a47ddc999a82c701567f1.png)  

[Here](./service-accounts/service-account.yml) is an example of a service account.
[Here](./service-accounts/sa-pod.yml) is an example of a pod using the service account.

## Roles
Roles are used to define the permissions list. Permission list is a list of verbs and resources. Verbs are the actions that can be performed on the resources. In Roles manifest permissions lists are called rules.

There are two types of roles Roles and ClusterRoles. Roles are used to define the permissions for a specific namespace. ClusterRoles are used to define the permissions for the entire cluster.

[Here](./service-accounts/role-pod-list.yml) is an example of a role that allows listing on pods.

## Role Binding
Role binding is used to bind the role to the users or service accounts. It is used to give the permissions to the user or service account.

There are two types of RoleBindings, RoleBinding and ClusterRoleBinding. RoleBinding is used to bind the role to the user or service account in a specific namespace. ClusterRoleBinding is used to bind the role to the user or service account in the entire cluster.

![picture 6](images/3e35b255a667ee265d2412921af2a6eb4b44ef061692450a6b7cb91f8637ad67.png)  

## Admission Controllers
Admission controllers intercept requests to the Kubernetes API server after authentication and authorization but before the object is persisted. They can be used to validate, deny or even modify the request.

Admission controllers comes with a set of admission controllers that are enabled or can be enabled. You can also create your own admission controllers.

![picture 7](images/e35e2d18497c6f0b64ba5c1cd4973dd8d0d526bfdc66efbbaf63eddd7ce2996a.png)  

[Here](./admission-controller/new-namespace-pod.yml) is the manifest that try to create a pod in the namespace that does not exist. It will be denied by the admission controller.

To make add the admission controller to the cluster, you need to add the following line to the kube-apiserver manifest file.

```yaml
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add add the following word.
![picture 8](images/1f2878e5b52acb26349c7f62d500bb267f3a9d1fed32e564c9ade2f9df9d0177.png)  

Once it is saved, the kube-apiserver will be restarted and the admission controller will be added to the cluster. This change will allow the pods creation in the non-existing namespace in which case it will create the namespace and then create the pod.

## Managing Compute Resources for Containers
Kubernetes provides a number of features to manage compute resources for containers. These features are used to manage the resources such as CPU and memory.

### Requests & Limits
| Resource | Limits |
| --- | --- |
|  Provides Kubernetes with an idea of how much resources the container is expected to use. | Provides an upper limit on how many resources a container is alloed to use. |
| The cluster uses this information to decide which node to place the pod on. | The cluster uses this information to **terminate the container process** if it attemps to use more than the allowed amount. |
| Requests are not hard limits. | Limits are hard limits. |

[Here](./resources/resources.yml) is an example of a pod with requests and limits.

### Resource Quotas
A ResourceQuota is a **kubernetes object** that sets limit on the resources userd **within a namespace**. If creating or modifying a resource would go beyond the quota, the request will be denied.

To be able to add resource quota, you need to add the following admission plugin to the kube-apiserver manifest file.

Once quota is created, **all the container resources should include the requests and limits**.

```yaml
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```
![picture 9](images/7a5bda00c690caf2fbad1f336a7fb9efcd1500e0ed09dd4db8c7043e46986f3b.png)  
Once saved and exited the kube api server will be restarted. Now you can create resource quota.

coammnd to list quotas with current usage
```shell
kubectl get resourcequota -n <namespace name>
```

[Here](./resources/quota-definition.yml) is an example of a resource quota definition.

If you try to apply [this](./resources/resources-exceeds.yml) it will be scheduled but not created. When you describe the replica set you can see the reason for the failure as `exceeded quota: limits.memory`.

## ConfigMaps and Secrets
ConfigMaps and Secrets are used to store configuration data and sensitive data respectively. They are used to decouple the configuration and secrets from the pods.

Kubernetes has options to encrypt the secrets.

![picture 10](images/87f6f8a5373d07dcea653a0d231c46f571245a68385ed125647d422703fb6baf.png)  

[Here](./configmaps-secrets/config-map-example.yml) is an example of a config map which is loaded to the environment variable and volume.

[Here](./configmaps-secrets/secret-example.yml) is an example of a secret which is loaded to the environment variable and volume.

## Security Context
A Security Context is used to control the security settings of a pod or container. It is used to control the security settings such as user id, group id, capabilities, etc.

Security context can be set at the pod level and at the container level.

### User ID and Group ID
You can customize the user and group id used by the container process. This is used to control the permissions of the container process.

### Allow Privilege Escalation
You can enable or disable privilege escalation for the container process.

### Read Only Root Filesystem
You can set the container's root filesystem to be read only.

[Here](./security-context/security-context-test.yml) is an example of a security context in container level.

Once this is applied, you can exec into the pod and check the user id and group id of the container process.

```shell
kubectl exec -it <pod name> -n <namespace name> -- /bin/sh
~$ id
```
or
```shell
kubectl exec <pod name> -n <namespace name> -- id
```

You can try create/edit the file. It will show an error because the root file system is read only.

```shell
~$ touch /test.txt
```
or
```shell
kubectl exec <pod name> -n <namespace name> -- touch /test.txt
```

# Services and Networking

## Controlling Network Access with NetworkPolicies - Part 1

### Kubernetes Networking

 The Kubernetes cluster uses a **Virtual Network** to allow Pods to communicate with one another seamlessly, even if they are on different nodes. 

![picture 11](images/167835f87cfab25a71789381b5ffeb6d44f45adedb654e8d21ebc2be6acacf49.png)  

### Network Policies

A NetworkPolicy is a **Kubernetes object** that allows you to **restrict network traffic** to and from Pods within the cluster network. It is used to control the traffic flow between the pods.

Use NetworkPolicies to **block** unnessary or unexpected network traffic and make your applications more secure.

### Isolated vs Non-Isolated Pods

![picture 12](images/fdd48e869b5062efa0bf560aab1adab877463fedcef0f639fd94b6cc1345ebc4.png)  

Network Policies use selector to select pods. The pods that are selected by atleast one network policy are called **isolated pods**. The pods that are not selected by any network policy are called **non-isolated pods**.


# note

Port forwarding command for service 
`kubectl port-forward svc/nginx 8080:80`

List pods with label
`kubectl get pods -l app=nginx`
