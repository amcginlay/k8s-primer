---
title: "Take Control of Kubernetes"
chapter: false
weight: 015
draft: false
---

## Take Control of Kubernetes

Among the [tools for managing Kubernetes](https://kubernetes.io/docs/tasks/tools/) there are some tools which manage the infrastructure of Kubernetes hosting and others which manage the **workloads** *inside*.
- `kind`, `k3s`, `minikube`, `kubeadm`, and `eksctl` are infrastructure management tools which allow you to govern *how* Kubernetes is hosted.
- `kubectl` - the Kubernetes *control* tool provides you with a command-line tool for managing what's *inside* your Kubernetes clusters. You can use `kubectl` [interactively](https://kubernetes.io/docs/reference/kubectl/kubectl/) or certainly in your scripting and automation.

## Install `kubectl`

Within your Cloud9 instance, [install `kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" # download tool
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256" # download checksum
echo "$(<kubectl.sha256)  kubectl" | sha256sum --check # confirm checksum
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

## A pilot by any other name would sound as Greek

The word Kubernetes means to **govern**, or to **steer** or **pilot** in the nautical sense. The English word [Kubernetes](https://en.wiktionary.org/wiki/Kubernetes#English) and the modern Greek word [κυβερνήτης](https://en.wiktionary.org/wiki/κυβερνήτης#Greek) -- pronounced like kyvernítis -- are derived from the Ancient Greek [κυβερνήτης](https://en.wiktionary.org/wiki/κυβερνήτης#Ancient_Greek). The important aspect to remember is that Kubernetes is about governance and steering the ship. *Your* ship. *You* are the captain. Kubernetes is your pilot, helm, and rest of the crew in the engine room -- there to help you get where you need safely and efficiently. Your apps are your passengers and your data is your cargo. 

## Use `kubectl` to inspect your Kubernetes cluster

Now that you have `kubectl`, you can manage the operations within your Kubernetes cluster. Kubernetes installations are made of a number of components. Let's take a brief tour through some basic anatomy. 

- **cluster** - an installation of Kubernetes
- **nodes** - compute power that hosts your cluster
- **data plane** - workers in your cluster
- **control plane** - government of your cluster
- **components** - members of the government

Take a look at the nodes in your `kind` cluster.

```bash
kubectl get nodes
```

You should see results which note the Kubernetes software version and the roles each node performs.

{{< output >}}
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,master   14h   v1.21.1
{{< /output >}}

Within the control plane, there are Kubernetes components which each serve a purpose like the members of government.

```bash
kubectl get componentstatuses
```

{{< output >}}
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"}                                                                             
{{< /output >}}

{{% notice note %}}
The term ComponentStatuses can be abbreviated "cs" as in `kubectl get cs`. In general, there are singular ("kind"), plural, and short forms of the various types of Kubernetes resources.
{{% /notice %}}

Check what is running in your cluster.

```bash
kubectl get pods
```

{{< output >}}
No resources found in default namespace.
{{< /output >}}

{{% notice note %}}
A `Pod` in Kubernetes is a collection of one or more containers that get scheduled and run together, like cetaceans (whales, orca, porpoises) traveling together. More on that later.
{{% /notice %}}

If you just created your Kubernetes cluster with `kind`, but have not yet deployed any apps to run in it, the cluster will have no pods in the default namespace, like a ship with no passengers. But… there is still a crew!

```bash
kubectl get pods -A
```

{{< output >}}
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-558bd4d5db-qxdcd                     1/1     Running   0          14h
kube-system          coredns-558bd4d5db-vzdrc                     1/1     Running   0          14h
kube-system          etcd-kind-control-plane                      1/1     Running   0          14h
kube-system          kindnet-hmwbj                                1/1     Running   0          14h
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          14h
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          14h
kube-system          kube-proxy-sc7qw                             1/1     Running   0          14h
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          14h
local-path-storage   local-path-provisioner-547f784dff-ckbn8      1/1     Running   0          14h
{{< /output >}}

When you use the `-A` option with `kubectl get pods` you are presented with the list of your pods and the Kubernetes system pods. This is like getting a list of both the passengers and crew of a ship.

## Namespaces = Logical Subclusters

What was the distinction between the passengers and crew? Notice the leftmost column in the list of pods. The **namespace**. Your Kubernetes cluster has a `default` namespace and a `kube-system` namespace. But wait, there's more!

```bash
kubectl get namespaces
```

Your `kind` cluster may have namespaces such as these.

{{< output >}}
NAME                 STATUS   AGE
default              Active   14h
kube-node-lease      Active   14h
kube-public          Active   14h
kube-system          Active   14h
local-path-storage   Active   14h
{{< /output >}}

{{% notice note %}}
The term Namespaces can be abbreviated "ns" as in `kubectl get ns`. 
{{% /notice %}}

Kubernetes namespaces are logical subdivisions of your cluster. Please don't confuse these with Linux namespaces (used in containization by the way), DNS namespaces (used for public, private, and protected network naming), or other varieties of namespaces. Each app, microservice, or project you deploy could be in its own namespace. It is up to you.

## Success

You have successfully installed the `kubectl` software to manage your Kubernetes cluster. With this tool, you:
- inspected the node(s) in the cluster
- glimpsed some control plane components
- confirmed you have no pods running
- noted that there are some system pods running in the `kube-system` namespace
- got a list of namespaces

Now you are ready to create your own resources, such as your own Kubernetes:
- namespaces
- configmaps
- secrets
- pods
and more…
