---
title: "Meet Kubernetes"
chapter: false
weight: 17
draft: false
---

Now you have now covered the basics it is time to get formally acquainted with **Kubernetes**.

Kubernetes is a container *orchestration* system which is a Cloud Native Computing Foundation (CNCF) **Graduated** project.
The [CNCF Landscape](https://landscape.cncf.io) shows many other projects that live in and around the ecosystem that Kubernetes is part of.
Kubernetes is often abbreviated "K8s" which is short for K, eight-letters ('ubernete'), s.

Kubernetes is certainly more complicated than Docker, which is warranted because it help you solve bigger kinds of problems than just running containers.
We'll guide you though more of the justifications of *why* we do Kubernetes along the journey through the workshop.
For now, you will just focus on getting Kubernetes installed and up-and-running.

## Available options

Whether you are using Kubernetes for production operations, app and microservices development, or experimentation and learning, there are hard ways, recommended ways, and easy ways to install and operate Kubernetes. 

1. Hard -- Kelsey Hightower famously provided a guide called [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).
While some may argue that some valuable soul-building benefits can be gained, that is not our focus now.
2. Recommended -- One of our main objectives is to get you familiar enough with Kubernetes so you can better appreciate and benefit from the value that the Amazon Elastic Kubernetes Service (EKS) provides.
We do recommend running real workloads in Amazon EKS. More on that later.
3. **Easy** -- Our focus now is to get you up-and-running on Kubernetes in a few minutes.
We have chosen to guide you through setting up "Kubernetes in Docker" -- [`kind`](https://kind.sigs.k8s.io/docs/design/initial/).

Besides `kind`, some other easy ways to run Kubernetes on a desktop, IoT device, or Cloud9 instance include:
- [`minikube`](https://minikube.sigs.k8s.io/docs/) provides a local Kubernetes cluster on macOS, Linux, and Windows.
- [`k3s`](https://k8s.io) is a Lightweight Kubernetes implementation from [Rancher](https://rancher.com/docs/k3s/latest/en/). This is designed for Edge and IoT applications. 

## Install Kubernetes in Docker (`kind`)

{{< step >}}Install `kind` in your Cloud9 environment.{{< /step >}}

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

{{% notice note %}}
For further details or to install `kind` on macOS, Windows, or other Linux systems, you can refer to the [`kind` Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/).
{{% /notice %}}

## Create your first cluster

Your next task is to create a multi-node Kubernetes cluster. 
Simply typing `kind create cluster` would create a one node cluster.
*Please do **not** do that now.*

Most Kubernetes clusters are composed of multiple compute resources.
These are referred to as [nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) and they come in a couple of varieties, as follows.
- **control plane** nodes -- a set of servers that provide container ***orchestration*** support features (like a queen bee of a hive or the government of a town/city)
- **data plane** nodes -- a set of servers that run the container workloads; these servers are container hosts, sometimes called **workers**, like the worker bees in a colony, or the populace of a town/city.
Each data plane node hosts an OCI compliant container runtime.
This could be the Docker runtime or any alternative (e.g. [containerd](https://containerd.io/)) capable of running OCI compliant containers.
From the abstracted perspective of a Kubernetes user, the choice of underlying container runtime is of no great significance.

The `kind` documentation provides a [suggested configuration manifest for multi-node clusters](https://kind.sigs.k8s.io/docs/user/quick-start/#multinode-clusters). You can use such a file to create your multi-node Kubernetes cluster based on `kind`. Your cluster can have multiple worker nodes that are separate from the control plane node(s).

{{< step >}}Create a Kubernetes cluster manifest file.{{< /step >}}

```bash
cat <<EOF >~/environment/103-four-node-cluster.yaml 
# four node (three workers) cluster config
kind: Cluster            # this file describes the Kubernetes infrastructure -- a "cluster" 
apiVersion: kind.x-k8s.io/v1alpha4
name: kind               # this is a default for Kubernetes-in-Docker (KinD), but be explicit
nodes:                   # the nodes are the host servers that implement your Kubernetes cluster
- role: control-plane    # one control plane node offers no redundancy for high availability
- role: worker           # each worker node can run your container workloads
- role: worker           # multiple workers is good practice and offers protection from node failures
- role: worker
EOF
```

{{< step >}}Create your four node cluster{{< /step >}}

```bash
kind create cluster --config ~/environment/103-four-node-cluster.yaml
```

This should result in a Kubernetes cluster (of the `kind` flavor) running in your Cloud9 environment.

{{< output >}}
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
{{< /output >}}

This operation will take a few minutes to complete after which you will have created your multi-node Kubernetes cluster.

Although we didn't provide many fancy options for `kind create cluster` beyond the configuration file, here are a few options that are supported on that command for when you want to customize the creation of the cluster. Use `kind create cluster --help` for further options.

- `--image` - container image to use for the nodes when booting the cluster
- `--kubeconfig` - sets kubeconfig path instead of `$KUBECONFIG` or `$HOME/.kube/config`
- `--name` - specify a cluster name, overrides `$KIND_CLUSTER_NAME`, config (default **kind**)

You have successfully installed the `kind` software in your Cloud9 instance and used it to create Kubernetes cluster based on `kind` - running the Kubernetes cluster components within containers on your Cloud9 instance. You can now use this Kubernetes cluster to create and manage other containers which run your app and microservice component workloads.

## Install the CLI tool

Among the [tools for Kubernetes](https://kubernetes.io/docs/tasks/tools/) there are some which manage the infrastructure and others which manage the **workloads** *inside*.
- `kind`, `k3s`, `minikube`, `kubeadm`, and `eksctl` are examples of infrastructure management tools which allow you to govern *how* Kubernetes is hosted.
- `kubectl` - the Kubernetes *control* tool provides you with a command-line tool for managing what's *inside* your Kubernetes clusters. You can use `kubectl` [interactively](https://kubernetes.io/docs/reference/kubectl/kubectl/) or embedded within your own automation scripts.

{{< step >}}Within your Cloud9 instance, [install `kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).{{< /step >}}

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" # download tool
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256" # download checksum
echo "$(<kubectl.sha256)  kubectl" | sha256sum --check # confirm checksum
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl # install CLI
rm kubectl* # dispose of installation files
kubectl version --client
```

## Some terminology

Now that you have `kubectl`, you can manage the operations within your Kubernetes cluster. Kubernetes installations are made of a number of components. Let's take a brief tour through some basic anatomy. 

- **cluster** - an installation of Kubernetes
- **nodes** - compute power that hosts your cluster
- **data plane** - workers in your cluster
- **control plane** - government of your cluster
- **components** - members of the government

## First contact

Take a look at the nodes in your Kubernetes `kind` cluster.

{{< step >}}Confirm the list of nodes in your cluster{{< /step >}}

```bash
kubectl get nodes
```

Example output:
{{< output >}}
NAME                 STATUS   ROLES                  AGE     VERSION
kind-control-plane   Ready    control-plane,master   2m31s   v1.21.1
kind-worker          Ready    <none>                 2m3s    v1.21.1
kind-worker2         Ready    <none>                 2m2s    v1.21.1
kind-worker3         Ready    <none>                 2m2s    v1.21.1
{{< /output >}}

The results will display the Kubernetes software version and the roles each node performs.

Graphically, this could be depicted as follows:

{{< mermaid >}}
graph TB
subgraph Kubernetes-cluster[Kubernetes Cluster]
  subgraph Control-plane[Control Plane]
    kind-control-plane
  end
  subgraph Data-plane[Data Plane]
    kind-worker1
    kind-worker2
    kind-worker3
  end
end

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class Kubernetes-cluster orange;
class Control-plane blue;
class Data-plane green;
{{< /mermaid >}}

So you created a local cluster and installed the CLI.
Then, as if by magic, you successfully established a line of communication between them.
How did the CLI know where the cluster endpoint (or [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)) was located?
How did the CLI authenticate and authorize with the Kubernetes API?

After `KinD` created your cluster it deposited a fully formed config file at `~/.kube/config`.
Take a quick look at that file and see if you can identify which port the Kubernetes API is occupying.
Try **carefully** renaming that file and see if `kubectl get nodes` still works.
This file is commonly referred to as `kubeconfig` and, as you can see, it is a requirement for `kubectl` to function correctly.

{{% notice note %}}
Make sure you replace your original `~/.kube/config` before moving on.
{{% /notice %}}

{{< step >}}Check what workloads are running in your cluster.{{< /step >}}

```bash
kubectl get pods
```

Example output:
{{< output >}}
No resources found in default namespace.
{{< /output >}}

{{% notice note %}}
A [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) in Kubernetes is a collection of one or more containers that get scheduled and run together, like cetaceans (whales, orca, porpoises) traveling together. More on that later.
{{% /notice %}}

If you just created your Kubernetes cluster with `kind`, but have not yet deployed any apps to run in it, the cluster will have no pods in the **default** namespace, like a ship with no passengers. But… there is still a crew!

```bash
kubectl get pods -A
```

{{< output >}}
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-558bd4d5db-ck5ft                     1/1     Running   0          13m
kube-system          coredns-558bd4d5db-l29q2                     1/1     Running   0          13m
kube-system          etcd-kind-control-plane                      1/1     Running   0          14m
kube-system          kindnet-bmxfq                                1/1     Running   0          13m
kube-system          kindnet-dvqbz                                1/1     Running   0          13m
kube-system          kindnet-nj56c                                1/1     Running   0          13m
kube-system          kindnet-w84f2                                1/1     Running   0          13m
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          14m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          14m
kube-system          kube-proxy-2wm8s                             1/1     Running   0          13m
kube-system          kube-proxy-gtmt5                             1/1     Running   0          13m
kube-system          kube-proxy-l8mkf                             1/1     Running   0          13m
kube-system          kube-proxy-xvbs6                             1/1     Running   0          13m
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          14m
local-path-storage   local-path-provisioner-547f784dff-dtnp2      1/1     Running   0          13m
{{< /output >}}

When you use the `-A` option with `kubectl get pods` you are presented with the list of **all** your pods alongside **all** the Kubernetes system pods. This is like getting a list of both the passengers and crew of a ship.

## Kubernetes namespaces

{{% notice note %}}
The term "namespace" is used in a variety of contexts.
For example, **Linux** namespaces, which we encountered in previous chapters, are a different concept to **Kubernetes** namespaces.
For the remainder of this course when you see the word namespace you must interpret this as **Kubernetes namespace** unless otherwise stated.
{{% /notice %}}

Consider a class in which two students named Elizabeth enrolled.
If we did not enquire as to their surnames it would be impossible, on paper, to tell them apart.
The purpose of namespaces in Kubernetes is to allow two resources with the same name to coexist, so long as they occupy a different namespace.
When you use a single cluster, a textbook use-case for namespaces is to separate your **Dev** environment from, say, your **Test** environment.

To see a list of the current namespaces

```bash
kubectl get namespaces
```

Example output:
{{< output >}}
NAME                 STATUS   AGE
default              Active   14h
kube-node-lease      Active   14h
kube-public          Active   14h
kube-system          Active   14h
local-path-storage   Active   14h
{{< /output >}}

Your `kind` cluster may have namespaces such as those shown above.

{{% notice note %}}
The term Namespaces can be abbreviated "ns" as in `kubectl get ns`. 
{{% /notice %}}

## Success

In this chapter you have successfully:
- Installed the Kubernetes software (KinD) and the `kubectl` CLI
- Created your four-node cluster
- Inspected the nodes in the cluster
- Glimpsed some control plane components
- Confirmed you have no pods running
- Noted that there are some system pods running in the `kube-system` namespace
- Obtained list of pre-configured namespaces

Now you are ready to create your own Kubernetes resources, such as:
- namespaces
- pods
- configmaps and more...
