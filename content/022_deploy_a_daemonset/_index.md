---
title: "Deploy a DaemonSet"
chapter: false
weight: 22
draft: false
---

## Purpose

You have deployed a set of pods using a `Deployment`. The other commonly used kind of Kubernetes workload is a `DaemonSet`. In this section, you will deploy a `DaemonSet` and investigate its behavior on a multi-node Kubernetes cluster. As a prerequisite, you should first deploy a multi-node Kubernetes cluster.

## Examine Your Current Cluster

{{< step >}}Find how many nodes are in your Kubernetes cluster.{{< /step >}}

```bash
kubectl get nodes
```

{{< output >}}
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,master   16d   v1.21.1
{{< /output >}}

{{< step >}}Run the same command with the `-o wide` option to show more details.{{< /step >}}

```bash
kubectl get nodes -o wide
```

{{< output >}}
NAME                 STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION                  CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane,master   16d   v1.21.1   172.18.0.2    <none>        Ubuntu 21.04   4.14.256-197.484.amzn2.x86_64   containerd://1.5.2
{{< /output >}}

{{< step >}}Because your Kubernetes cluster is implemented with `kind`, you can list the nodes from the hosting perspective using the `kind` command as follows.{{< /step >}}

```bash
kind get nodes
```

{{< output >}}
kind-control-plane
{{< /output >}}

{{< step >}}Finally, get use the `kind` command to get the name of your existing cluster.{{< /step >}}

```bash
kind get clusters
```

{{< output >}}
kind
{{< /output >}}

{{% notice note %}}
Most Kubernetes cluster hosting scenarios fully support adding and removing nodes from the cluster.
In fact, this is an essential aspect of Kubernetes infrastructure management of the node hosting.
However, due to simplicity of `kind`, it does not support adding or removing nodes.
This is not considered a bug, as it is so incredibly easy and fast to create and delete whole `kind` clusters.
{{% /notice %}}

You have a single-node cluster. To explore the nature and power of `DaemonSet` and `Deployment` workloads, as well as the essence of `Service` objects, a multi-node cluster is essential. Therefore, create a multi-node cluster before proceeding.

## Create a Three-Node Cluster

The `kind` documentation provides a [suggested configuration manifest for multi-node clusters](https://kind.sigs.k8s.io/docs/user/quick-start/#multinode-clusters). You can use such a file to create your multi-node Kubernetes cluster based on `kind`.

{{< step >}}Create a new cluster manifest.{{< /step >}}

```bash
cat <<EOF >~/environment/102-three-node-cluster.yaml 
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

{{< step >}}Create another one with more nodes.{{< /step >}}

```bash
cat <<EOF >~/environment/103-four-node-cluster.yaml 
# four node (three workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: four-node-kind
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
```

{{< step >}}Create your four node cluster{{< /step >}}

```bash
kind create cluster --config ~/environment/103-four-node-cluster.yaml
```

{{< output >}}
Creating cluster "four-node-kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-four-node-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-four-node-kind

Have a nice day! 👋
{{< /output >}}

You have now created a multi-node Kubernetes cluster!

{{< step >}}Confirm the list of nodes in your cluster{{< /step >}}

```bash
kubectl get nodes
```

{{< output >}}
NAME                           STATUS   ROLES                  AGE     VERSION
four-node-kind-control-plane   Ready    control-plane,master   2m31s   v1.21.1
four-node-kind-worker          Ready    <none>                 2m3s    v1.21.1
four-node-kind-worker2         Ready    <none>                 2m2s    v1.21.1
four-node-kind-worker3         Ready    <none>                 2m2s    v1.21.1
{{< /output >}}

Now you can use this cluster to provision and test `DaemonSet`, `Deployment`, and `Service` configurations.

{{% notice tip %}}
You should now have connection information for two Kubernetes clusters listed in your `~/.kube/config` file.
If you had done `kind delete cluster` (or specifically `kind delete cluster --name kind`) prior to creating your second cluster, there would now be only the one four-node cluster. Your default context has been set to the most recently created/added cluster. Therefore, you need not specify which cluster when you want to use the recently added one, but would specify the cluster or switch contexts when you want to work with the original cluster.
{{% /notice %}}

## Taint the Nodes

## Write a DaemonSet Manifest

## Deploy Your DaemonSet

## Untaint Nodes

## DaemonSet Quiz

## Success

