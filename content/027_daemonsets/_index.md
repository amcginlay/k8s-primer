---
title: "12. DaemonSets"
chapter: false
weight: 27
draft: false
---

## Purpose

You have deployed a set of pods using a `Deployment`. The other commonly used kind of Kubernetes workload is a `DaemonSet`. In this section, you will inspect some built in examples before deploying your own `DaemonSet` and investigating its behavior on a multi-node Kubernetes cluster.

## Why DaemonSets?

DaemonSets can improve cluster performance by deploying pods that perform maintenance tasks and support services to every node.
DaemonSets are particularly well suited for long-running services which can include:
- Log collection
- Metrics generation
- Distributed tracing
- Networking
- Troubleshooting

## Built in DaemonSets

Kubernetes normally has some deployments and daemonsets running by default.
It can be useful to understand the differences and similarities between those and the daemonset you will create.

{{< step >}}Start by reviewing the nodes in your cluster.{{< /step >}}

```bash
kubectl get nodes -o wide
```

Example output (abbreviated):
{{< output >}}
NAME                 STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   ...
kind-control-plane   Ready    control-plane,master   2m32s   v1.21.1   172.18.0.2    ...
kind-worker          Ready    <none>                 118s    v1.21.1   172.18.0.5    ...
kind-worker2         Ready    <none>                 119s    v1.21.1   172.18.0.4    ...
kind-worker3         Ready    <none>                 118s    v1.21.1   172.18.0.3    ...
{{< /output >}}

{{% notice tip %}}
Observe from this output that, much like pods, nodes are also assigned IP addresses.
This is standard Kubernetes behavior for all nodes and pods.
If you look very closely you may also notice that, in your cluster, these IP addresses appear to be from an alternate range as those assigned to your pods (`172.18.x.x` vs. `10.244.x.x`).
This behavior can differ between Kubernetes distributions.
It depends upon the chosen implementation of the [CNI Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni).
The default CNI for KinD is named `kindnet`.
{{% /notice %}}

As you can see from the previous output, there are four nodes in total, comprised of one `control-plane` node and three `worker` nodes.

{{< step >}}Check which built in daemonsets are currently running in the `kube-system` namespace.{{< /step >}}

```bash
kubectl -n kube-system get daemonsets -o wide
```

Example output (abbreviated):
{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            ... SELECTOR
kindnet      4         4         4       4            4           <none>                   ... app=kindnet
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   ... k8s-app=kube-proxy
{{< /output >}}

There are four pods apiece, running in both the `kindnet` and `kube-proxy` daemonsets.
Observe the values of `SELECTOR` in the above output and let's keep digging.

{{< step >}}Get a list of the pods that match that `SELECTOR` for `kube-proxy`.{{< /step >}}

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

Example output (abbreviated):
{{< output >}}
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE                 ...
kube-proxy-42dkd   1/1     Running   0          22h   172.18.0.3   kind-worker3         ...
kube-proxy-fh8w2   1/1     Running   0          22h   172.18.0.2   kind-control-plane   ...
kube-proxy-g86fg   1/1     Running   0          22h   172.18.0.5   kind-worker          ...
kube-proxy-kd2qd   1/1     Running   0          22h   172.18.0.4   kind-worker2         ...
{{< /output >}}

You will observe that **each node** hosts **exactly one** pod instance.
That's the standard purpose of daemonsets, although you will see later how you can influence that result.

In the next section you will create your own daemonset and see how it compares to the built in daemonsets.

## System Namespaces

Namespaces such as `kube-system` and `istio-system` are commonly used to identify components that are running behind the scenes like a cast of supporting characters or the crew of a show.
Daemonsets are often used to accommodate those types of non-functional or hidden system requirements so, whilst putting them in a dedicated namespace is not a hard requirement, it is also not an unreasonable thing to do.

{{< step >}}Write a Kubernetes manifest for a `dev-system` namespace.{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/012-dev-system-namespace.yaml | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: dev-system
EOF
```

Example output:
{{< output >}}
namespace/dev-system created
{{< /output >}}

## Deploy Your DaemonSet

A `DaemonSet` manifest shares **almost** all its DNA with the `Deployment` manifest.
The most notable absentee is the `replicas` attribute, and you can perhaps already understand why.
Start with something simple that feels familiar.

{{< step >}}Create the manifest file.{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/013-demo-daemonset.yaml | kubectl -n dev-system apply -f -
apiVersion: apps/v1
kind: DaemonSet              # the object schema Kubernetes uses to validate this manifest
metadata:
  labels:
    app: demo-daemon
  name: demo-daemon
spec:
                             # huh ... no "replicas" attribute?
  selector:
    matchLabels:
      app: demo-daemon       # pods in this namespace and with this label are members
  template:
    metadata:
      labels:
        app: demo-daemon
    spec:
      containers:
      - name: demo
        image: demo:1.0.0    # re-using your demo container image, because you can ...
EOF
```

Example output:
{{< output >}}
daemonset.apps/demo-daemon created
{{< /output >}}

Your daemonset has now been successfully created.

## Inspect Your DaemonSet

{{< step >}}Check the status of your daemonset.{{< /step >}}

```bash
kubectl -n dev-system get daemonset -o wide
```

Example output (abbreviated):
{{< output >}}
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   ... SELECTOR
demo-daemon   3         3         3       3            3           <none>          ... app=demo-daemon
{{< /output >}}

{{< step >}}Use the `SELECTOR` to reveal the pods running in the `DaemonSet` and **where they are running**.{{< /step >}}

```bash
kubectl -n dev-system get pods -l app=demo-daemon -o wide
```

Example output (abbreviated):
{{< output >}}
NAME                READY   STATUS    RESTARTS   AGE   IP           NODE           ...
demo-daemon-pw67k   1/1     Running   0          15m   10.244.3.3   kind-worker    ...
demo-daemon-s6vns   1/1     Running   0          15m   10.244.2.3   kind-worker3   ...
demo-daemon-x77qp   1/1     Running   0          15m   10.244.1.3   kind-worker2   ...
{{< /output >}}

You will see `kind-worker[n]` appearing under `NODE` which tells us that, unlike the built in daemonsets such as `kube-proxy`, your pods are running everywhere **except** the control-plane.
This is `taints` and `tolerations` at work and this behavior is by design.

## Skewing Fairness

By default, you deploy pods to a Kubernetes cluster and are not concerned with where and how the pods are scheduled and restarted.
However, if you have reasons to control which pods run on which nodes, Kubernetes provides you with some mechanisms to facilitate bias such as the following.

- **Taints**: Nodes can be configured so they do not run all workloads. ***taints*** are conditions which *repel* pods, except when pods meet those conditions.
- **Tolerations**: Pods can be configured to match the conditions of specific taints. ***tolerations*** negate ***taints***, providing an override mechanism.

There are three flavors/degrees of effects for taints and tolerations. unless, tolerated by one or more pods, these **effects** in a tainted node behave as follows:
- **NoExecute** -- this effect will *evict* currently running pods, and will not *schedule* new pods on the node.
- **NoSchedule** -- this effect will leave pre-existing pods running, but will not *schedule* new pods on the node.
- **PreferNoSchedule** -- this effect will leave pre-existing pods running, the scheduler will attempt to find other nodes on which to run new pods, but ultimately the node could accept newly scheduled pods despite the taint.

## Using Taints

{{< step >}}Let's create a **taint** on `kind-worker3` to see if we can dissuade our daemonset from targeting all the worker nodes as it currently does.{{< /step >}}

```bash
kubectl taint nodes kind-worker3 VIPOnly=true:NoExecute
```

Example output:
{{< output >}}
node/kind-worker3 tainted
{{< /output >}}

The key/value pairs used in taints (e.g. `VIPOnly=true`) have no significant meaning to Kubernetes.
They are just conditions placed upon specific nodes which must be met, like a secret passcode required to gain entry to a private party.

{{< step >}}Check the node to ensure the taint has been successfully applied.{{< /step >}}

```bash
kubectl get node kind-worker3 -o jsonpath-as-json={.spec.taints[*]}
```

Example output:
{{< output >}}
[
    {
        "effect": "NoExecute",
        "key": "VIPOnly",
        "value": "true"
    }
]
{{< /output >}}

{{% notice tip %}}
As you can see `kubectl` is not limited to displaying results as YAML.
[JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/) provides a powerful query tool which helps to pick out fine detail such as `taints`.
If you prefer, you can also use `kubectl describe node kind-worker3` to yield the same results, but you will need to scroll around a little to find what you seek.
{{% /notice %}}

{{< step >}}Now check again where the pods are running.{{< /step >}}

```bash
kubectl -n dev-system get pods -l app=demo-daemon -o wide
```

Example output (abbreviated):
{{< output >}}
NAME                READY   STATUS    RESTARTS   AGE   IP           NODE           ...
demo-daemon-x77qp   1/1     Running   0          23m   10.244.1.3   kind-worker2   ...
demo-daemon-pw67k   1/1     Running   0          23m   10.244.3.3   kind-worker    ...
{{< /output >}}

{{% notice note %}}
It may take a few seconds before pods settle into a `Running` state.
{{% /notice %}}

As you can see, the pod which was previously running on `kind-worker3` is gone. It was **evicted** as it was unable to tolerate the new taint.

{{% notice tip %}}
For future reference, **untainting** is achieved by appending a single hyphen symbol to the previous `taint` command, i.e. `kubectl taint nodes kind-worker3 VIPOnly=true:NoExecute-`.
{{% /notice %}}

## Using Tolerations

{{< step >}}Now let's add a matching toleration to the daemonset and see if its pods can override the taint.{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/014-demo-daemonset.yaml | kubectl -n dev-system apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: demo-daemon
  name: demo-daemon
spec:
  selector:
    matchLabels:
      app: demo-daemon
  template:
    metadata:
      labels:
        app: demo-daemon
    spec:
      containers:
      - name: demo
        image: demo:1.0.0
      tolerations:           # tolerations
      - key: "VIPOnly"       #
        operator: "Equal"    #
        value: "true"        #
        effect: "NoExecute"  #
EOF
```

Example output:
{{< output >}}
daemonset.apps/demo-daemon configured
{{< /output >}}

{{% notice note %}}
In the manifest above you will see `operator: "Equal"`.
The Kubernetes documentation on [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) specifies two `operator` values: `Exists` and `Equal`.
The default operator is `Equal`, which means you don't actually *need* to specify that in your YAML.
When you provide no value with `Exists` that matches all values for your key.
If you leave out the key, `Exists` will match all keys.
Therefore, if you used `operator: Exists` with no key or value, that would tolerate *any taint*.
{{% /notice %}}

{{< step >}}Check the daemonset to ensure the single `toleration` has been successfully applied.{{< /step >}}

```bash
kubectl -n dev-system get daemonset demo-daemon -o jsonpath-as-json={.spec.template.spec.tolerations[*]}
```

Example output:
{{< output >}}
[
    {
        "effect": "NoExecute",
        "key": "VIPOnly",
        "operator": "Equal",
        "value": "true"
    }
]
{{< /output >}}

{{< step >}}Now check once again to see where the pods are running.{{< /step >}}

```bash
kubectl -n dev-system get pods -l app=demo-daemon -o wide
```

Example output (abbreviated):
{{< output >}}
NAME                READY   STATUS    RESTARTS   AGE    IP           NODE           ...
demo-daemon-6lh59   1/1     Running   0          88s    10.244.1.5   kind-worker3   ...
demo-daemon-crcqx   1/1     Running   0          77s    10.244.3.5   kind-worker    ...
demo-daemon-nr4wm   1/1     Running   0          104s   10.244.2.5   kind-worker2   ...
{{< /output >}}

So now the pods know the "secret passcode" and are once again able to attend almost every party in town.
The taints applied to the `control-plane` are, as intended, still prohibiting your pods from entering there.
Bonus points for identifying the taint(s) applied there ... but those taints are there for good reason so you are best advised to *not* try to circumvent or override them!

## DaemonSet Quiz

Please take the following quiz to review your knowledge of `DaemonSet` objects.

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## Which nodes?

---
shuffle_answers: false
---

Which nodes does a `DaemonSet` deploy pods to?

> How is a daemonset different than a deployment?

- [ ] a specific list of nodes in your `DaemonSet` spec
- [ ] randomly scheduled to the number of `replicas` you specify
- [ ] all nodes, always, no exceptions
- [x] all nodes, except for nodes that repel the pods

## Repel pods

Which Kubernetes attribute **repels** certain pods?

> Soft Cell sang about what kind of love?

- [ ] Schedule
- [x] Taint
- [ ] Toleration
- [ ] MatchLabels

## Kubernetes manifests

What is the typical order we have used to write Kubernetes manifests?

> Though not a mandatory order, we have followed a pattern in this workshop.

1. apiVersion
2. kind
3. metadata
4. spec

## Tolerations

Where do you put tolerations in a `DaemonSet` manifest?

> What kind of Kubernetes object does the `DaemonSet` manage?

- [ ] `metadata.tolerations`
- [ ] `spec.tolerations`
- [ ] `spec.template.tolerations`
- [x] `spec.template.spec.tolerations`

{{< /quizdown >}}

## Success

In this training adventure, you have:
- Investigated some built in daemonsets.
- Deployed a `DaemonSet` in your cluster (`demo-daemon`).
- Investigated some key difference between *your* daemonset and the built in ones.
- Tainted a worker node in the cluster and observed the effects.
- Updated your `DaemonSet` to **tolerate** those taints.
