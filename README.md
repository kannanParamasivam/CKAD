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

### Control plane node

```bash
~$ sudo vim /etc/hosts
```

Add the following lines

```bash
<ip address> k8s-control-plane
<ip address> k8s-worker-1
<ip address> k8s-worker-2
```
