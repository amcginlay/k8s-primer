---
title: "Install Kubernetes"
chapter: false
weight: 17
draft: false
---

## Kubernetes the Quick and Easy Way

**Kubernetes** is a container *orchestration* system which is a Cloud Native Computing Foundation (CNCF) **Graduated** project. The [CNCF Landscape](https://landscape.cncf.io) shows many other projects that live in and around the ecosystem that Kubernetes is part of. Kubernetes is often abbreviated "K8s" which is short for K, eight-letters ('ubernete'), s.

Kubernetes is certainly more complicated than Docker, which is warranted because it help you solve bigger kinds of problems than just running containers. We'll guide you though more of the justifications of *why* we do Kubernetes along the journey through the workshop. For now, let's just focus on getting Kubernetes installed and up-and-running!

Whether you are using Kubernetes for production operations, app and microservices development, or experimentation and learning, there is are easy ways, recommended ways, and hard ways to install and operate Kubernetes. 

1. Hard -- Kelsey Hightower famously provided a guide called [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way). While some may argue that some valuable soul-building benefits can be gained, that is not our focus now.
2. Recommended -- One of our main objectives is to get you familiar enough with Kubernetes so you can better appreciate and benefit from the value that the Amazon Elastic Kubernetes Service (EKS) provides. We do recommend running real workloads in Amazon EKS. More on that later.
3. Easy -- Our focus now is to get you up-and-running on Kubernetes in a few minutes. We have chosen to guide you through setting up "Kubernetes in Docker" -- [`kind`](https://kind.sigs.k8s.io/docs/design/initial/).

{{% notice note %}}
Besides `kind`, some other easy ways to run Kubernetes on a desktop, IoT device, or Cloud9 instance include:
- [`minikube`](https://minikube.sigs.k8s.io/docs/) provides a local Kubernetes cluster on macOS, Linux, and Windows.
- [`k3s`](https://k8s.io) is a Lightweight Kubernetes implementation from [Rancher](https://rancher.com/docs/k3s/latest/en/). This is designed for Edge and IoT applications. 
{{% /notice %}}

## Install Kubernetes in Docker (`kind`)

<!--
Prior to installing `kind`, update the version of the Go programming language in your Cloud9 instance:
```bash
go version
curl https://go.dev/dl/go1.17.6.linux-amd64.tar.gz -o go1.17.6.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.6.linux-amd64.tar.gz
go version
```
-->

Install `kind` in your Cloud9 environment. 
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

<!-- GO111MODULE="on" go get sigs.k8s.io/kind@v0.11.1 -->
{{% notice note %}}
For further details or to install `kind` on macOS, Windows, or other Linux systems, you can refer to the [`kind` Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/).
{{% /notice %}}

## Create a Kubernetes cluster with `kind`

Create a `kind` cluster.
```bash
kind create cluster
```

This should result in a Kubernetes cluster (of the `kind` flavor) running in your Cloud9 environment.
{{< output >}}
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
{{< /output >}}

Although we didn't provide any fancy options for `kind create cluster` in this simple case, here are a few options that are supported on that command for when you want to customize the creation of the cluster. Use `kind create cluster --help` for further options.

- `--image` - container image to use for the nodes when booting the cluster
- `--kubeconfig` - sets kubeconfig path instead of `$KUBECONFIG` or `$HOME/.kube/config`
- `--name` - specify a cluster name, overrides `$KIND_CLUSTER_NAME`, config (default **kind**)

## Success

You have successfully installed the `kind` software in your Cloud9 instance and used it to create Kubernetes cluster based on `kind` - running the Kubernetes cluster components within containers on your Cloud9 instance. You can now use this Kubernetes cluster to create and manage other containers which run your app and microservice component workloads.

