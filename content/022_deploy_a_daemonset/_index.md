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

You can configure a pod-spec (e.g. in a bare `Pod` object or in the `template` of a `Deployment` or `DaemonSet`) to be attracted to run on node(s) with specific criteria. For example, you may have a workload that requires graphics processing unit (GPU) capable compute power, thus could configure `nodeAffinity` for the pods in that workload to either *prefer* or *require* scheduling and execution on GPU-capable nodes. ***Affinity is attractive***--you can configure you pod specs to indicate the kinds of hardware and software environmets of the nodes they are attracted to.

***Taints***, on the other hand, are **repulsive**--you can apply a *taints* to *nodes*--they repel pods that do not *tolerate* their particular taints.

{{< step >}}Generate a manifest for a **taint**.{{< /step >}}

```bash
kubectl taint nodes four-node-kind-worker2 repeldaemons=true:NoSchedule --dry-run=client -o yaml >~/environment/104-taint-example.yaml
head -26 104-taint-example.yaml | tail -9
```

{{< output >}}
spec:
  podCIDR: 10.244.2.0/24
  podCIDRs:
  - 10.244.2.0/24
  providerID: kind://docker/four-node-kind/four-node-kind-worker2
  taints:
  - effect: NoSchedule
    key: repeldaemons
    value: "true"
{{< /output >}}

There are three parts to the `taint` specification. Let's look at the short form you used in the command and the long form in the manifest:
- command line form: the short notation `repeldaemons=true:NoSchedule` specifies a key, value, and effect for the taint.
- manifest form: the long form also specifies the key, value, and effect of the taint, typically as three separate lines in the manifest file:
    - `effect: NoSchedule` -- the *effect* allows you to specify the way the taint affects scheduling and execution of pods.
    - `key: repeldaemons` -- you can choose a *key* name that reminds you of the purpose of the taint.
    - `value: "true"` -- the *value* part of the taint helps you remember the sense of how the key is interpreted.

{{% notice tip %}}
You could choose a key-value pair of `daemons=false` or `repeldaemons=true` to play into the repulsive nature of the taint. Kubernetes does not understand the key and value names, it only compares them to pods' tolerations for equality or existence. The *meaning* of the words you put in the key and value of the taint are for *you*.
{{% /notice %}}

There are three effects for taints and tolerations. unless, tolerated by one or more pods, these *effects** in a tainted node behave as follows:
- **NoExecute** -- this effect will *evict* curently running pods, and will not *schedule* new pods on the node.
- **NoSchedule** -- this effect will leave pre-existing pods running, but will not *schedule* new pods on the node.
- **PreferNoSchedule** -- this effect will leave pre-existing pods running, the scheduler will attempt to find other nodes on which to run new pods, but ultimately the node could accept newly scheduled pods despite the taint.

{{< step >}}Check which daemonsets are running now.{{< /step >}}

```bash
kubectl get daemonsets -n kube-system
```

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kindnet      4         4         4       4            4           <none>                   59m
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   59m
{{< /output >}}

{{< step >}}Check which labels the pods in the `kube-proxy` daemonset have.{{< /step >}}

```bash
kubectl describe daemonset kube-proxy -n kube-system | grep Labels
```

{{< output >}}
Labels:         k8s-app=kube-proxy
  Labels:           k8s-app=kube-proxy
{{< /output >}}

{{< step >}}Get a list of the pods in that daemonset.{{< /step >}}

```bash
kubectl get pods -l k8s-app=kube-proxy -n kube-system
```
{{< output >}}
NAME               READY   STATUS    RESTARTS   AGE
kube-proxy-4cvhq   1/1     Running   0          62m
kube-proxy-8gk9k   1/1     Running   0          62m
kube-proxy-hq2d9   1/1     Running   0          62m
kube-proxy-wzssr   1/1     Running   0          62m
{{< /output >}}

{{< step >}}Add the `-o wide` option to show the node names on which those pods are running.{{< /step >}}

```bash
kubectl get pods -l k8s-app=kube-proxy -n kube-system -o wide
```

{{< output >}}
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE                           NOMINATED NODE   READINESS GATES
kube-proxy-4cvhq   1/1     Running   0          62m   172.18.0.3   four-node-kind-worker3         <none>           <none>
kube-proxy-8gk9k   1/1     Running   0          62m   172.18.0.6   four-node-kind-worker          <none>           <none>
kube-proxy-hq2d9   1/1     Running   0          62m   172.18.0.5   four-node-kind-worker2         <none>           <none>
kube-proxy-wzssr   1/1     Running   0          63m   172.18.0.4   four-node-kind-control-plane   <none>           <none>
{{< /output >}}


{{< step >}}Taint your second and third worker nodes.{{< /step >}}

```bash
kubectl taint nodes four-node-kind-worker2 repeldaemons=true:NoSchedule 
kubectl taint nodes four-node-kind-worker3 repeldaemons=true:NoSchedule 
```

{{< step >}}Check again to see the effects of the taints on those daemonsets.{{< /step >}}

```bash
kubectl get daemonset/kube-proxy -n kube-system
```

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   59m
{{< /output >}}

{{< step >}}Replace those taints (delete and add new ones), with `NoExecute` instead of `NoSchedule`.{{< /step >}}

```bash
kubectl taint nodes four-node-kind-worker2 repeldaemons=true:NoSchedule-
kubectl taint nodes four-node-kind-worker3 repeldaemons=true:NoSchedule-
kubectl taint nodes four-node-kind-worker2 repeldaemons=true:NoExecute
kubectl taint nodes four-node-kind-worker3 repeldaemons=true:NoExecute
```

{{% notice note %}}
The notation `key=value:effect` ***adds*** a taint, whereas the notation `key=value:effect-` *(with the minus sign on the end)* ***deletes*** a taint--i.e. **untaints** the node. Thus, this sequence untaints the two nodes of the `repeldaemons=true:NoSchedule` then *taints* them each with `repeldaemons=true:NoExecute`.
{{% /notice %}}

{{< step >}}Check again to see the effects of the taints on those daemonsets.{{< /step >}}

```bash
kubectl get daemonset/kube-proxy -n kube-system
```

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   59m
{{< /output >}}

Again, of the three effects--`PreferNoSchedule`, `NoSchedule`, and `NoExecute`--this `NoExecute` you just tained two of your worker nodes with is the **strongest taint effect** you can place on nodes. How is it possible that those nodes are running pods provisioned by this `DaemonSet`, such that there are still four ready and available pods?

{{< step >}}Check the `tolerations` on the pod-spec in the `kube-proxy` `DaemonSet` template.{{< /step >}}

```bash
kubectl get ds/kube-proxy -n kube-system -o yaml >~/environment/105-kube-proxy-ds.yaml
grep tolerations -A 3 ~/environment/105-kube-proxy-ds.yaml  # note: -A 3 shows three lines after the match
```

{{< output >}}
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - operator: Exists
{{< /output >}}

{{< step >}}Show those `tolerations` in JSON format using a JSON-**path** specification.{{< /step >}}

```bash
kubectl get ds/kube-proxy -n kube-system -o jsonpath-as-json={.spec.template.spec.tolerations}
```

{{< output >}}
[
    [
        {
            "key": "CriticalAddonsOnly",
            "operator": "Exists"
        },
        {
            "operator": "Exists"
        }
    ]
]
{{< /output >}}

What both the YAML and JSON notations show is that the pods created by the `kube-proxy` daemonset will have two tolerations:
1. Tolerate taints on nodes which have a key of "CriticalAddonsOnly" and ***any*** value. The operator `Exists` (as opposed to `Equals`) matches taints with that key and anyvalue.
2. Tolerate taints on nodes which have ***ANY TAINT***. Seriously. The notation in the second toleration with just `"operator": "Exists"` will match and tolerate any taints with ***any*** key **and** ***any*** value.

{{% notice note %}}
Configuring a pod (via a `DaemonSet` or `Deployment` or otherwise) to tolerate ***any and all*** taints like this means that it is immune and unaffected by taints. System-critical pods like `kube-proxy` are the only pods that should be configured this way. The `DaemonSet` you'll create next will not have such a strong toleration.
{{% /notice %}}

## Write a DaemonSet Manifest

Like a `Deployment`, a `DaemonSet` Kubernetes manifest includes 

Here is a graphical representation of the components of that YAML `Deployment` manifest.

{{< mermaid >}}
graph TB
subgraph Deployment-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: Deployment)
  subgraph Deployment-metadata
    name(name)
    deploymentLabels[labels]
  end
  subgraph Deployment-spec
    replicas(replicas)
    strategy
    selector
    subgraph template
      subgraph Pod-metadata
        podLabels[labels]
      end
      subgraph Pod-spec
        containers
      end
    end
  end
end
{{< /mermaid >}}

The critical ingredients in the deployment `spec` are:
- `strategy` - choose the replacement or update strategy for new versions of your app
- `selector` - the selector allows the deployment to own pods with certain `Pod` labels
- `template` - the template allows the deployment to create new pods with specific metadata and `Pod` spec

{{% notice note %}}
Unlike a `Deployment`, your `DaemonSet` will not specify a number of `replicas` each node will have one `Pod` for this `DaemonSet`. Well, each untained node; more on that later.
{{% /notice %}}

## Write your Deployment manifest

Now that you know what goes in a `Deployment` manifest, it is time to create one. We'll skip the update `strategy`. For the `Pod` spec within, let's start simple, without the environment variables, volume mounts, and other fancy decorations.

```bash
cat << EOF | tee ~/environment/101-demo-deployment.yaml | kubectl -n dev apply -f -
apiVersion: apps/v1          # remember to use apps/v1 instead of merely v1
kind: Deployment             # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demo                 # a name for your DEPLOYMENT
  labels:                    # labels allow you to tag your DEPLOYMENT with a set of key/value pairs
    app: demo                # it is customary to have an "app" key with a value to identify your deployment
spec:                        # the DEPLOYMENT specification begins here, note no indentation!
  replicas: 3                # the "spec.replicas" is one of the most important aspects of a Deployment
  selector:                  # you can enable the Deployment to acquire/own pods which match your "selector"
    matchLabels:             # the most common selector clause is to match labels on existing Pods
      app: demo              # you want pre-existing pods with a label "app: demo" in this namespace
  template:                  # the "spec.template" is where you specify the Pod "metadata" and "spec" to deploy
    metadata:
      labels:
        app: demo
    spec:                    # this is like a Pod spec, but indented two YAML level (four spaces customarily)
      containers:            # a pod CAN consist of multiple containers, but this one has only one
      - name: demo           # a name for your CONTAINER
        image: demo:1.0.0    # the tagged image we previously injected using "kind load"
EOF
```

Hopefully your deployment is successfully created.

{{< output >}}
deployment.apps/demo created
{{< /output >}}

Check the status of your deployment.

```bash
kubectl get deployment -n dev
```

{{< output >}}
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
demo   3/3     3            3           27s
{{< /output >}}
TODO

## Deploy Your DaemonSet

## Untaint Nodes

## DaemonSet Quiz

## Success



## Write a DaemonSet Manifest

## Deploy Your DaemonSet

## Untaint Nodes

## DaemonSet Quiz

## Success

