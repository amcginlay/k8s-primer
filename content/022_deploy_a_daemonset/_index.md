---
title: "Deploy a DaemonSet"
chapter: false
weight: 22
draft: false
---

## Purpose

You have deployed a set of pods using a `Deployment`. The other commonly used kind of Kubernetes workload is a `DaemonSet`. In this section, you will deploy a `DaemonSet` and investigate its behavior on a multi-node Kubernetes cluster. As a prerequisite, you should first deploy a multi-node Kubernetes cluster.

## Create a Separate Namespace for Your DaemonSet

You could use any namespace for your `DaemonSet`. Namespaces such as `kube-system` and `istio-system` are used to identify components that are running behind the scenes like a cast of supporting characters or the crew of a show. Note that `kube-system` in particular contains both `Deployment` and `DaemonSet` workloads, so our use of `dev-system` for this `DaemonSet` is a bit arbitrary. You could simply use the `dev` namespace or any other.

{{< step >}}Write a Kubernetes manifest for a `dev-system` namespace.{{< /step >}}

```bash
cat > ~/environment/106-dev-system-namespace.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  name: dev-system
EOF

kubectl apply -f ~/environment/106-dev-system-namespace.yaml
```

`kubectl` will respond as follows.
{{< output >}}
namespace/dev-system created
{{< /output >}}


## Write a DaemonSet Manifest

Like a `Deployment`, a `DaemonSet` Kubernetes manifest includes `metadata` and a `spec`.

Here is a graphical representation of the components of that YAML `DaemonSet` manifest.

{{< mermaid >}}
graph TB
subgraph DaemonSet-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: DaemonSet)
  subgraph DaemonSet-metadata
    name(name)
    deploymentLabels[labels]
  end
  subgraph DaemonSet-spec
    updateStrategy
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

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class DaemonSet-manifest orange;
class DaemonSet-metadata blue;
class DaemonSet-spec green;
class template yellow;
{{< /mermaid >}}

The critical ingredients in the daemonset `spec` are:
- `selector` - the selector allows the daemonset to own pods with certain `Pod` labels
- `template` - the template allows the daemonset to create new pods with specific metadata and `Pod` spec
- `updateStrategy` - (optional) choose the replacement or update strategy for new versions of your app

{{% notice note %}}
Unlike a `Deployment`, your `DaemonSet` will *not* specify a number of `replicas`. Each node will have one `Pod` for this `DaemonSet`. Well, each untained node; more on that later.
{{% /notice %}}

Now that you know what goes in a `DaemonSet` manifest, it is time to create one. We'll skip the `updateStrategy`. For the `Pod` spec within, let's start simple, without the environment variables, volume mounts, and other fancy decorations.

{{< step >}}Create the manifest file.{{< /step >}}

```bash
cat <<EOF >~/environment/107-demo-daemonset.yaml 
apiVersion: apps/v1          # remember to use apps/v1 instead of merely v1
kind: DaemonSet              # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demo-daemon          # a name for your DAEMONSET
  labels:                    # labels allow you to tag your DAEMONSET with a set of key/value pairs
    app: demo-daemon         # it is customary to have an "app" key with a value to identify your daemonset
spec:                        # the DAEMONSET specification begins here, note no indentation!
  selector:                  # you can enable the DaemonSet to acquire/own pods which match your "selector"
    matchLabels:             # the most common selector clause is to match labels on existing Pods
      app: demo-daemon       # you want pre-existing pods with a label "app: demo-daemon" in this namespace
  template:                  # the "spec.template" is where you specify the Pod "metadata" and "spec" to deploy
    metadata:
      labels:
        app: demo-daemon
    spec:                    # this is like a Pod spec, but indented two YAML level (four spaces customarily)
      containers:            # a pod CAN consist of multiple containers, but this one has only one
      - name: demo           # a name for your CONTAINER
        image: demo:1.0.0    # the tagged image we previously injected using "kind load"
EOF
```

## Deploy Your DaemonSet

{{< step >}}Deploy your daemonset.{{< /step >}}

```kubectl
kubectl -n dev-system apply -f ~/environment/107-demo-daemonset.yaml 
```

{{< output >}}
daemonset.apps/demo-daemon created
{{< /output >}}

Hopefully your daemonset is successfully created.

{{< step >}}Check the status of your daemonset.{{< /step >}}

```bash
kubectl get daemonset -n dev-system
```

{{< output >}}
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
demo-daemon   3         3         3       3            3           <none>          9s
{{< /output >}}

{{< step >}}Reveal the pods running in the `DaemonSet`.{{< /step >}}

```bash
kubectl get pods -l app=demo-daemon -n dev-system
```

{{% notice tip %}}
The `-l app=demo-daemon` selects the pods by label rather than name.
{{% /notice %}}

{{< output >}}
NAME                READY   STATUS    RESTARTS   AGE
demo-daemon-pw67k   1/1     Running   0          15m
demo-daemon-s6vns   1/1     Running   0          15m
demo-daemon-x77qp   1/1     Running   0          15m
{{< /output >}}

{{< step >}}Get more detail about where those pods are running.{{< /step >}}

```bash
kubectl get pods -l app=demo-daemon -n dev-system -o wide
```

{{% notice tip %}}
Using the `-o wide` option also shows IP, NODE, NOMINATED NODE, and READINESS GATES columns.
{{% /notice %}}

{{< output >}}
NAME                READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
demo-daemon-pw67k   1/1     Running   0          15m   10.244.3.3   kind-worker    <none>           <none>
demo-daemon-s6vns   1/1     Running   0          15m   10.244.2.3   kind-worker3   <none>           <none>
demo-daemon-x77qp   1/1     Running   0          15m   10.244.1.3   kind-worker2   <none>           <none>
{{< /output >}}

You have deployed a `DaemonSet` that is running a pod on every worker node.

## Investigate System Daemons

Kubernetes normally has some deployments and daemonsets running by default. It can be useful to understand the differences and similarities between those and the daemonset you created.

{{< step >}}Check which daemonsets are running now in the `kube-system` namespace.{{< /step >}}

```bash
kubectl get daemonsets -n kube-system
```

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kindnet      4         4         4       4            4           <none>                   59m
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   59m
{{< /output >}}

Why are there four copies of `kindnet` and `kube-proxy` running, but only three copies of your daemonset?

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
kube-proxy-7d8jx   1/1     Running   0          39m
kube-proxy-jj88g   1/1     Running   0          39m
kube-proxy-q9fzz   1/1     Running   0          39m
kube-proxy-vf65b   1/1     Running   0          39m
{{< /output >}}

{{< step >}}Add the `-o wide` option to show the node names on which those pods are running.{{< /step >}}

```bash
kubectl get pods -l k8s-app=kube-proxy -n kube-system -o wide
```

{{< output >}}
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
kube-proxy-7d8jx   1/1     Running   0          39m   172.18.0.2   kind-worker2         <none>           <none>
kube-proxy-jj88g   1/1     Running   0          39m   172.18.0.5   kind-worker3         <none>           <none>
kube-proxy-q9fzz   1/1     Running   0          39m   172.18.0.3   kind-worker          <none>           <none>
kube-proxy-vf65b   1/1     Running   0          40m   172.18.0.4   kind-control-plane   <none>           <none>
{{< /output >}}

How is it possible that the `kube-proxy` daemonset (and `kindnet` too) is running both on the worker nodes and on the control plane node? And why did your `demo-daemon` daemonset stay on the worker nodes but *not* run on the control plane?

Quick answer: ***taints*** and ***tolerations***.

## Skewing Fairness -- Taints and Tolerations

Two of the greatest Kubernetes' orchestration benefits are:
- automatic placement and restart of pods, and 
- replacement of pods, including but not limited to for scale in and scale out.

You can deploy pods to a Kubernetes cluster and not be concerned with where and how the pods are scheduled and restarted.

However... if you have reasons to control which pods run on which nodes, Kubernetes provides you with a number of control mechanisms to facilitate bias.

- Nodes can be *tainted* so they do not run all workloads. ***taints*** are *repulsive* - they *repel* certain pods.
- Pods can *tolerate* specific taints. ***tolerations*** negate the repulsion of taints - they not *not* cause the tolerant pods to be scheduled on certain nodes--they *allow* it.
- Pods could also specify *node affinity*. ***affinity*** is *attractive* - those pods would be attracted to run on certain matching nodes. 

Recall that your `demo-daemon` runs on three nodes, but `kube-proxy` is running on four. Why? How? Let's look at the question of how.

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
Configuring a pod (via a `DaemonSet` or `Deployment` or otherwise) to tolerate ***any and all*** taints like this means that it is immune and unaffected by taints. System-critical pods like `kube-proxy` are the only pods that should be configured this way. The `DaemonSet` created does not have such a strong toleration, in fact, you did not explicitly add any tolerations. Yet.
{{% /notice %}}

## Taint Your Nodes

You can configure a pod-spec (e.g. in a bare `Pod` object or in the `template` of a `Deployment` or `DaemonSet`) to be attracted to run on node(s) with specific criteria. For example, you may have a workload that requires graphics processing unit (GPU) capable compute power, thus could configure `nodeAffinity` for the pods in that workload to either *prefer* or *require* scheduling and execution on GPU-capable nodes. ***Affinity is attractive***--you can configure you pod specs to indicate the kinds of hardware and software environmets of the nodes they are attracted to.

***Taints***, on the other hand, are **repulsive**--you can apply a *taints* to *nodes*--they repel pods that do not *tolerate* their particular taints.

{{< step >}}Generate a manifest for a **taint**.{{< /step >}}

```bash
kubectl taint nodes kind-worker3 repeldaemons=true:NoSchedule --dry-run=client -o yaml >~/environment/104-taint-example.yaml
grep ^spec: -A 8 104-taint-example.yaml
```

<!-- replaced the obscure with the grep above. had: head -26 104-taint-example.yaml | tail -9 -->

{{< output >}}
spec:
  podCIDR: 10.244.2.0/24
  podCIDRs:
  - 10.244.2.0/24
  providerID: kind://docker/kind/kind-worker3
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
You could choose a key-value pair of `daemons=false` or `repeldaemons=true` to play into the repulsive nature of the taint. Kubernetes does not understand the key and value names, it only compares them to pods' tolerations for *equality* or *existence*. The *meaning* of the words you put in the key and value of the taint are for *you*.
{{% /notice %}}

There are three flavors/degrees of effects for taints and tolerations. unless, tolerated by one or more pods, these *effects** in a tainted node behave as follows:
- **NoExecute** -- this effect will *evict* curently running pods, and will not *schedule* new pods on the node.
- **NoSchedule** -- this effect will leave pre-existing pods running, but will not *schedule* new pods on the node.
- **PreferNoSchedule** -- this effect will leave pre-existing pods running, the scheduler will attempt to find other nodes on which to run new pods, but ultimately the node could accept newly scheduled pods despite the taint.

{{< step >}}Write a YAML snippet for this taint.{{< /step >}}

```bash
echo $'spec:\n taints:\n - effect: NoSchedule\n   key: repeldaemons\n   value: "true"'
```

{{< output >}}
spec:
 taints:
 - effect: NoSchedule
   key: repeldaemons
   value: "true"
{{< /output >}}

{{% notice note %}}
The `\n` notation is a customary shorthand for a **newline** character (ASCII 10, Unicode U+000A). The output of this snippet should be functionally the same as the relevant fragment of the `kubectl taint nodes ... --dry-run ...` command, however we used one space per level of indentation rather than two. This is a cheap way to generate a little YAML snippet. The `taints:` line and `- effect` line are indented 1 level, and 3 levels for both the `key` and `value`. Alternatively, you can use JSON notation for patches. Wait? Patches? Yes!
{{% /notice %}}

{{< step >}}Patch the node with the taint!{{< /step >}}

```bash
kubectl patch node kind-worker3 -p $'spec:\n taints:\n - effect: NoSchedule\n   key: repeldaemons\n   value: "true"'
```

{{< output >}}
node/kind-worker3 patched
{{< /output >}}

{{% notice tip %}}
A ***patch*** is a Kubernetes operation that is **general purpose**. Is it the simplest way to apply a taint? Perhaps not. You could simply have done `kubectl taint node kind-worker3 repeldaemons=true:NoSchedule` interactively or in a script. You could even use `kubectl edit node kind-worker3` and interactively hack the taint into the node spec in an editor. Like `kubectl taint`, `kubectl patch` is scriptable which is good for devops. But the `patch` operation is more general purpose--it can be used for nodes, pods, configmaps, deployments, daemonsets, and more--and *you should know it*.
{{% /notice %}}

{{< step >}}Check what happened to your `demo-daemon` daemonset.{{< /step >}}

```bash
kubectl get ds/demo-daemon -n dev-system
```

{{< output >}}
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
demo-daemon   2         2         2       2            2           <none>          4h47m
{{< /output >}}

By design, your daemonset runs on all *untainted* nodes. Untolerated taints can repel daemonset pods, deployment pods, or bare pods. 

{{< step >}}Taint another node.{{< /step >}}

```bash
kubectl patch node kind-worker2 -p $'spec:\n taints:\n - effect: NoSchedule\n   key: repeldaemons\n   value: "true"'
```

{{< output >}}
node/kind-worker3 patched
{{< /output >}}

{{< step >}}Check your `demo-daemon` again.{{< /step >}}

```bash
kubectl get ds/demo-daemon -n dev-system
```

{{< output >}}
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
demo-daemon   1         1         1       1            1           <none>          4h53m
{{< /output >}}

There is only one untainted node on which your `demo-daemon` can run. Again, taints can affect which nodes on which pods are able to run. What about the "system" pods?

{{< step >}}Check what happened to the `kube-proxy` daemonset.{{< /step >}}

```bash
kubectl get ds/kube-proxy -n kube-system
```

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   4h51m
{{< /output >}}

## Make Your DaemonSet Pods Tolerate Your Taint

You can add one or more tolerations to your pods. If you are deploying pods via a `Deployment` or `DaemonSet`, you would want to specify the tolerations in the `spec.template.spec.tolerations` section. You extracted such a tolerations section when investigating the `kube-proxy` earlier, however you may not have looked that that in-situ within the context of the `spec.template.spec` hierarchy for that pod spec. 

While you certainly *could* provide a whole new `DaemonSet` manifest to add tolerations, try applying a toleration as a patch to your existing `demo-daemon` daemonset.

{{< step >}}But first, for comparison sake, save a copy of the **taint** you had applied to two of your nodes earlier.{{< /step >}}

```bash
echo $'spec:\n taints:\n - effect: NoSchedule\n   key: repeldaemons\n   value: "true"' >~/environment/108-taint-patch-file.yaml
```

{{< step >}}Now create a similar patch file for your new **toleration**.{{< /step >}}

```bash
echo $'spec:\n template:\n  spec:\n   tolerations:\n   - key: repeldaemons\n     operator: Equal\n     value: "true"\n     effect: NoSchedule' | tee ~/environment/109-toleration-patch-file.yaml
```

{{< output >}}
spec:
 template:
  spec:
   tolerations:
   - key: repeldaemons
     operator: Equal
     value: "true"
     effect: NoSchedule
{{< /output >}}

{{% notice note %}}
The Kubernetes documentation on [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) specifies two `operator` values: `Exists` and `Equal`. The default operator is `Equal`, which means you don't actually *need* to specify that in your patch file in this case. When you provide no value with `Exists` that matches all values for your key. If you leave out the key, `Exists` will match all keys. Therefore, if you used `operator: Exists` with no key or value, that would tolerate *any taint*. That was what had been specified as one of the tolerations of `kube-proxy`.
{{% /notice %}}

{{< step >}}Apply your toleration as a patch to your daemonset.{{< /step >}}

```bash
kubectl patch daemonset demo-daemon -n dev-system --patch-file ~/environment/109-toleration-patch-file.yaml
```

{{< output >}}
daemonset.apps/demo-daemon patched
{{< /output >}}

{{< step >}}Check your `demo-daemon` again.{{< /step >}}

```bash
kubectl get ds/demo-daemon -n dev-system
```

{{< output >}}
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
demo-daemon   3         3         3       3            3           <none>          5h33m
{{< /output >}}

Now that the pods deployed in your daemonset *tolerate* the taints on some of your nodes, the daemonset is able to run pods on all worker nodes.

## Untaint Nodes

If you no longer need the taints on those nodes, you can remove them.

{{< step >}}Create a file with a list of nodes to patch.{{< /step >}}

```bash
cat <<EOF >~/environment/110-nodes-to-untaint.txt
kind-worker2
kind-worker3
EOF
```

{{< step >}}Create a patch file in JSON format.{{< /step >}}

```bash
cat <<EOF >~/environment/111-no-taints.yaml
spec: {
    taints: []
}
EOF
```

{{< step >}}Remove the taints from those nodes.{{< /step >}}

```bash
for onenode in `cat ~/environment/110-nodes-to-untaint.txt`
do
    kubectl patch node $onenode --patch-file ~/environment/111-no-taints.yaml
done
```

{{< output >}}
node/kind-worker2 patched
node/kind-worker3 patched
{{< /output >}}

You have successfully removed the taints from both those worker nodes.

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

Which Kubernetes property **repels** certain pods?

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
- Deployed a `DaemonSet` in your cluster (`demo-daemon`).
- Investigated a key difference between *your* daemonset and `kube-proxy`--*tolerations*.
- Tainted some nodes in the cluster and observed the effects.
- Patched your `DaemonSet` to **tolerate** those taints.
- Untainted the nodes.
