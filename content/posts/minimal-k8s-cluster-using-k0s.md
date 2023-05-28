---
title: "Minimal K8s Cluster Using K0s"
date: 2022-04-07T23:05:27+04:30
draft: false
categories: [kubernetes]
tags: [kubernetes, k0s, container, container orchestration, minimal cluster, container, raspberry pi]
keywords: [kubernetes, k8s, container, container orchestration, k0s, minimal cluster, containerd, docker, containers, cluster, raspberry pi]
---

# Today I will show you the power of minimalism

* at first we will talk about Kubernetes a bit
* then we will talk about what is k0s and comparing it to it's alternatives
* next I will show you two simple example of making a cluster on a single node or using `k0sctl`

> Before reading this section you should have a bit knowledge about docker

## Let's talk about Kubernetes a bit
K8s is a container orchestrator it's like a conductor for containers to manage your cluster and for example decide witch container should go where.

in Kubernetes we are not interacting with containers directly we have lots of resources they have they own usage to fit in all of your scenarios. if you need a resource type that you need and it's not inside kubernetes by default we have `CustomResourceDefinitions`

in K8s our tiniest resource is `Pod`, this resource can contain one or more containers inside it.

but we are not going to create them most of the times we will apply `Deployment`, `StatefulSet`, `DaemonSet`, etc.

> I can't talk about all of them here but if you don't have a knowledge about them go and checkout [Kubernetes Website](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

## K0s
[k0s](https://k0sproject.io/) is a minimal kubernetes cluster they call it __The Zero Friction Kubernetes__ made by [mirantis](https://www.mirantis.com) the creator of [Lens kubernetes IDE](https://k8slens.dev/)

instead of `docker` k0s is using raw [`containerd`](https://containerd.io/) this will make it hard sometimes but images are fully compatible with each other because the manifest of images in all of the container runtimes are standard
![containerd architecture](https://containerd.io/img/architecture.png "containerd architecture")

I think it also give you an option to apply helm charts with it's own `CustomResourceDefinitions`

so using `containerd` makes it a lot smaller

### but why not using normal Kubernetes?
when you want a cluster to be easy to run and being small in size and work with low hardware usage a normal K8s cluster is not what we want they made K8s as minimal as possible by keeping it's stability

For example a Raspberry Pi Cluster

### what are the alternatives to K0s?
* [microk8s](https://microk8s.io/) by canonical
* [K3s](https://k3s.io/) by Rancher Team (creator or Rancher and RKE)

I have tried all of them but K0s was more easy and more productive

### Requirements?
It works in everywhere but for more details check out [this link](https://github.com/k0sproject/k0s/blob/main/docs/system-requirements.md)

## Let's install a sample cluster
* install any Linux-Based Operating System on your nodes
* then select one of the installation ways (Single Node, k0sctl, Raspberry Pi 4, Ansible, etc.)
Now I will show you `Single Node` and `k0sctl` since they are recommended
> the instructions below is from the official documents with a bit of my experience on top of that

* __Single Node__

> you should run all of the commands as root user or using `sudo`

at first we need to install k0s binary (it's the only thing that we will install on nodes in everyway of installation) by running command below on our node:


```bash
curl -sSLf https://get.k0s.sh | sudo sh
```
Drink a coffee until installation finishes

confirm the installation by running `k0s` command without any option it will show you help of the k0s command

`k0s install` command is used when installing a node
run the command below to install it as a single node cluster
```bash
k0s install controller --single
```

to start it run the command below
```bash
k0s start
```

command below will show you status of the cluster
```bash
k0s status
```

`k0s kubectl` is used for interacting with your cluster

if you want `kubeconfig` file it's in `/var/lib/k0s/pki/admin.conf`

* __k0sctl__

if you have production usage or automation on top of your installation you need this type of installation

the picture below will show you how `k0sctl` works

![how `k0sctl` works](https://docs.k0sproject.io/v1.23.5+k0s.0/img/k0sctl_deployment.png "how `k0sctl` works")

first we need to install k0sctl command; just go to [this link](https://github.com/k0sproject/k0sctl/releases/latest) and grab a binary that is compatible with your Operating System and put it in one of the directories that is listed inside your `$PATH`
> don't install it inside you target nodes

first we need access to root user of the nodes (putting your `ssh-key` should be enough) make sure you can access to all nodes without entering you ssh password.

next run the command below to create a sample file and then modify it
```bash
k0sctl init > k0sctl.yaml
```

open `k0sctl.yaml` file with the editor that you love :)

the file will be something like this:
```yaml
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
spec:
  hosts:
  - role: controller
    ssh:
      address: 10.0.0.1 # replace with the controller's IP address
      user: root
      keyPath: ~/.ssh/id_rsa
  - role: worker
    ssh:
      address: 10.0.0.2 # replace with the worker's IP address
      user: root
      keyPath: ~/.ssh/id_rsa
```
* `hosts` section is an array of your host add them as many as you want
* `keyPath` is the address of your ssh private key
* `role` can be one of:

    - `controller` - a controller host
    - `controller+worker` - a controller host that will also run workloads
    - `single` - a single-node cluster host, the configuration can only contain one host
    - `worker` - a worker host
    > sometimes `controller+worker` option will not make node to accept workloads I will show you how you can fix it later on

more info on the configuration file is [here](https://github.com/k0sproject/k0sctl#configuration-file-spec-fields)

when your configuration file is ready run the command below
```bash
k0sctl apply --config k0sctl.yaml
```

this will install and deploy your cluster

to access your cluster run the command below to get your `kubeconfig`
```bash
k0sctl kubeconfig > kubeconfig
```
this command will put it inside a file called `kubeconfig`

checkout your cluster status by running
```bash
kubectl --kubeconfig kubeconfig get all -A
```
and
```bash
kubectl --kubeconfig kubeconfig get nodes
```

if you specified `controller+worker` but workloads are pending run the command below
```bash
kubectl --kubeconfig kubeconfig taint nodes controlplane node-role.kubernetes.io/master:NoSchedule-
```

__some notes:__

---

* `k0sctl` can only add more nodes to the cluster. It cannot remove existing nodes.
* running `k0sctl apply` again command will just apply changes to the cluster.

---

### recommended post-installation steps
* installing `metallb` inside cluster
* installing an `ingress controller`
* deploy some great open-source projects and make your own cloud ;)