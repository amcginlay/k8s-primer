---
title: "Deploy a Fleet of Pods"
chapter: false
weight: 21
draft: false
---

## Purpose

Let's review what you've done so far before we up the ante in this game.

1. Recall that a `Pod` is the smallest deployable unit of compute power in Kubernetes.
2. Kubernetes is a container orchestrator. It manages pod restarts and container health.
3. Therefore, even provisioning *individual pods* in Kubernetes is more appropriate than container-by-container administration with a mere container runtime like Docker.

Now it is time for you to unleash the next level of power in Kubernetes: ***Workloads***.
[Workloads](https://kubernetes.io/docs/concepts/workloads/) in Kubernetes are designed to support your operations for many ***kinds*** of enterprise app, microservice, and devops automation. To paraphrase the Kubernetes docs, each ***workload*** manages a ***set*** of pods on your behalf.

In this section, you will learn that although Kubernetes supports many `kind` of workload beyond the `Pod`, most enterprise use cases are satisfied by one `kind` in particular: `Deployment`. You will provision a fleet of pods using a `Deployment`.

{{< mermaid >}}
graph TB
deployment(Workload)-->|manages|podA[Pod A]
deployment-->|manages|podB[Pod B]
deployment-->|manages|podM[...]
deployment-->|manages|podZ[Pod Z]
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class podA,podB,podM,podZ yellow;
{{< /mermaid >}}

## Workloads Manage Pods

Each `kind` of workload manages a set of pods in a paricular way.  Kubernetes workloads can satisfy specific use cases:
- [`Job`](https://kubernetes.io/docs/concepts/workloads/controllers/job/) -- the lifetime of a `Job` workload is to manage a pod that runs once to completion. 
{{< expand "Read more" >}}
    <ul>
    <li>Alternatively, a <code>Job</code> can manage *multiple parallel* pods. Set <code>spec.parallelism</code> greater than 1.</li>
    <li>The <code>Job</code> tracks the number of pods that have completed their task.</li>
    <li>Use the <code>spec.completions</code> property to manage job success (not needed for one-pod-jobs).</li>
    <li>Suspending a <code>Job</code> will delete its active pods until you resume the <code>Job</code>.</li>
    <li>When you delete a <code>Job</code> it cleans up the pods created for the job.</li>
    </ul>
{{< /expand >}}
- [`CronJob`](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) -- you can schedule a pod or set of pods to run a certain time or on a repeating schedule using a `CronJob`.
{{< expand "Read more" >}}
    <ul>
    <li>You set the <code>spec.schedule</code> property of the <code>CronJob</code> to a <a href="https://pkg.go.dev/github.com/robfig/cron/v3#hdr-CRON_Expression_Format">cron expression</a> value.</li>
    <li>The <code>CronJob</code> manages a <code>Job</code> object at each repeated interval on the schedule you established.</li>
    <li>Those <code>Job</code> objects manage their pod(s) in turn.</li>
    </ul>
{{< /expand >}}
- [`StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) -- when you have **stateful** workloads, you could choose to manage those with a Kubernetes `StatefulSet` workload.
{{< expand "Read more" >}}
    <ul>
    <li>Quoting the K8s docs, <code>StatefulSet</code> "pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling." </li>
    <li>The key advantage of a <code>StatefulSet</code> is that it maintains stable persistence of <strong>unique pod naming</strong> and <strong>storage</strong> despite potential pod rescheduling.</li>
    <li>You must establish persistent volumes and specific pod network naming with a headless service to support that required persistence for your workload's state.</li>
    </ul>
{{< /expand >}}
- [`DaemonSet`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) -- you can use a `DaemonSet` to run a replica pod on each node in your Kubernetes cluster.
{{< expand "Read more" >}}
    <ul>
    <li>By default a replica <code>Pod</code> which matches your <code>DaemonSet</code> template is deployed to all nodes in your cluster.</li>
    <li>You can choose a subset of nodes rather than <em><strong>all</strong></em> nodes. Use the <code>spec.template.spec.nodeSelector</code> or <code>spec.template.spec.affinity</code> to control this.</li>
    <li>There are some default daemonsets in Kubernetes and more in Amazon EKS. Some daemonsets handle network plumbing, monitoring, and logging capabilities.</li>
    <li>Your clusters can have your own custom daemonets, core Kubernetes daemonsets, and daemonsets for extensions.</li>
    </ul>
{{< /expand >}}
- [`ReplicaSet`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) -- for your **stateless** workloads, you could choose a `ReplicaSet` workload, except that `Deployment` is a more common choice which manages `ReplicaSet` workloads for you.
{{< expand "Read more" >}}
    <ul>
    <li><code>ReplicaSet</code> objects possess three primary properties in their <code>spec</code>: <code>replicas</code>, <code>selector</code>, and <code>template</code>.</li>
    <li>You can specify the <code>spec.replicas</code> to establish your <em><strong>DESIRED</strong></em> number of `Pod` replicas in this workload.</li>
    <li>A <code>ReplicaSet</code> can <em><strong>acquire</strong></em> pods which match the <code>spec.selector</code> you specify.</li>
    <li>You provide a <code>spec.template</code> which is made of a <code>metadata</code> and <code>spec</code> as the recipe for allowing your <code>ReplicaSet</code> to <em><strong>create</strong></em> new pod replicas to meet your desired capacity.</li>
    </ul>
{{< /expand >}}
- [`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) -- you will most likely use Kubernetes `Deployment` definitions to provision and manage most of your K8s workloads.
{{< expand "Read more" >}}
    <ul>
    <li>You can create a <code>Deployment</code> which will create and manage one or more <code>ReplicaSet</code> objects.</li>
    <li>The key advantage of using a <code>Deployment</code> is that the deployment manages multiple versions of your workloads </li>
    <li>You specify the <code>spec.strategy.type</code> property to either <code>Recreate</code> or <code>RollingUpdate</code>. Note that <code>RollingUpdate</code> is the default.
    <li>You can manage the <code>Deployment</code> using <em>rollout</em> and <em>rollback</em> operations.</li>
    <li>When you want to control the rate of a rolling update, to prevent 200% of your desired capacity (100% from your old replica set, another 100% from your new replica set), configure <code>spec.strategy.rollingUpdate.maxSurge</code>. You can specify either an absolute number of pods or a percentage of pods.</li>
    </ul>
{{< /expand >}}

{{< mermaid >}}
graph TB
deployment(Deployment)-->|manages|replicaSetA(Old ReplicaSet)
replicaSetA-->|manages|podA[Pod A]
replicaSetA-->|manages|podB[...]
replicaSetA-->|manages|podM[Pod M]
deployment-->|manages|replicaSetB(New ReplicaSet)
replicaSetB-->|manages|podN[Pod N]
replicaSetB-->|manages|podO[...]
replicaSetB-->|manages|podZ[Pod Z]

  classDef green fill:#9f6,stroke:#333,stroke-width:4px;
  classDef orange fill:#f96,stroke:#333,stroke-width:4px;
  classDef blue fill:#69f,stroke:#333,stroke-width:4px;
  classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
  class deployment orange;
  class replicaSetA blue;
  class replicaSetB green;
  class podA,podB,podM,podN,podO,podZ yellow;
{{< /mermaid >}}

{{% notice tip %}}
Of these, the two most common and important are `DaemonSet` and `Deployment`!
{{% /notice %}}

With this in mind, to learn more about workloads, you can:
1. Inspect a `DaemonSet` in your Kubernetes cluster.
2. Look at an existing `Deployment` in your cluster.
3. Create your own `Deployment`.

## What Daemons are in Your Cluster?

`DaemonSet` workloads are typically used to provide useful background services that should be available on most nodes or every node in your Kubernetes clusters. 

{{< step >}}Check to see if you have any daemonsets running in your cluster:{{< /step >}}

```bash
kubectl get ds
```

{{< output >}}
No resources found in default namespace.
{{< /output >}}

If there are no daemonsets running in the **default** namespace of your cluster, which other namespace could they be in? 

{{< step >}}Check what namespaces you have:{{< /step >}}

```bash
kubectl get ns
```

Your list may look similar to this.

{{< output >}}
NAME                 STATUS   AGE
default              Active   9d
dev                  Active   2d
kube-node-lease      Active   9d
kube-public          Active   9d
kube-system          Active   9d
local-path-storage   Active   9d
staging              Active   30h
test                 Active   30h
{{< /output >}}

Which namespace contains some default `DaemonSet` objects?

{{% notice info %}}
It is the `kube-system` namespace that contains several built-in `DaemonSet` and `Deployment` workloads.
{{% /notice %}}

{{< step >}}Query for the list in `kube-system`:{{< /step >}}

```bash
kubectl get daemonsets -n kube-system
```

While other Kubernetes clusters may have a different set of daemonsets, your workshop cluster should look similar to this:

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kindnet      1         1         1       1            1           <none>                   9d
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   9d
{{< /output >}}

Workloads, like these daemonsets, ***managed*** pods. 

{{< step >}}Check what pods you have in the `kube-system` namespace.{{< /step >}}

```bash
kubectl get pods -n kube-system
```

{{< output >}}
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-qxdcd                     1/1     Running   1          9d
coredns-558bd4d5db-vzdrc                     1/1     Running   1          9d
etcd-kind-control-plane                      1/1     Running   1          9d
kindnet-hmwbj                                1/1     Running   1          9d
kube-apiserver-kind-control-plane            1/1     Running   1          9d
kube-controller-manager-kind-control-plane   1/1     Running   1          9d
kube-proxy-sc7qw                             1/1     Running   1          9d
kube-scheduler-kind-control-plane            1/1     Running   1          9d
{{< /output >}}

Some of these pods were created by the daemonsets, while others were created by deployments.

{{< step >}}Get the details of the `kube-proxy` pod:{{< /step >}}

```bash
kppod=$(kubectl get pods -n kube-system | grep kube-proxy | sed 's/ .*//')
echo $kppod
```

Notice the `Pod` name is derived from the `DaemonSet` name. The suffix after "kube-proxy" is automatically generated when the `kube-proxy` daemonset creates the pods to implement that on each node.

{{< output >}}
kube-proxy-sc7qw
{{< /output >}}

{{< step >}}Confirm again the basic properties of the pod.{{< /step >}}

```bash
kubectl get pods $kppod -n kube-system            
```

{{< output >}}
NAME               READY   STATUS    RESTARTS   AGE
kube-proxy-sc7qw   1/1     Running   1          9d
{{< /output >}}

As usual, the **READY** column shows the number of containers running within the pod.

{{< step >}}Now get further details of that pod:{{< /step >}}

```bash
kubectl get pods $kppod -n kube-system -o yaml
```

{{% notice note %}}
Many details have been elided from the following output. 
{{% /notice %}}

{{< output >}}
apiVersion: v1
kind: Pod
metadata:
  generateName: kube-proxy-
  name: kube-proxy-sc7qw
  namespace: kube-system
  ownerReferences:
  - apiVersion: apps/v1
    kind: DaemonSet
    name: kube-proxy
spec:
  containers:
  - command:
    - /usr/local/bin/kube-proxy
    - --config=/var/lib/kube-proxy/config.conf
    - --hostname-override=$(NODE_NAME)
    env:
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: spec.nodeName
    image: k8s.gcr.io/kube-proxy:v1.21.1
    name: kube-proxy
    volumeMounts:
      ...
  dnsPolicy: ClusterFirst
  ...
  volumes:
  - configMap:
      defaultMode: 420
      name: kube-proxy
    name: kube-proxy
  - ...
{{< /output >}}

Please focus first on the `metadata` section. 
- `generateName` specifies the prefix of the name given to the pod and directs Kubernetes to pseudorandomly *generate* a suffix to make the pod name *unique*.
- `ownerReferences` indicates that the `kube-proxy` `DaemonSet` ***owns*** this pod.

In the `spec` section, note the following:
- `env` is used to map an environment variable
- `volumeMounts` and `volumes` are used to bring in a `ConfigMap`
- a single container runs a program which refers to both the environment variable and `config.conf` data from the mounted volume as follows:
    - `/usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=$(NODE_NAME)` 

{{< step >}}Now, look back at the definition of the `DaemonSet`.{{< /step >}}

```bash
kubectl get daemonset kube-proxy -n kube-system   
```

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   9d
{{< /output >}}

Unlike the `Pod` **READY**, **STATUS**, and **RESTARTS** columns, the default `kubectl get` output for workloads such as `DaemonSet` and `Deployment` shows the following counts:
- **DESIRED** - the number of pod replicas you *want* in this workload
- **CURRENT** - how many replicas are *running*
- **READY** - how many replicas are *ready*, and available to users/clients; note that for a `Deployment`, `kubectl get` will show *ready*/*desired* values as a fraction in the **READY** column, and not show separate **DESIRED** and **CURRENT**.
- **UP-TO-DATE** - the number of pods which have been *updated* to the latest desired state. When you update the configuration of a workload, there will be some time period of transition from the prior state to the new desired state, thus this value can vary from **CURRENT** during updates.
- **AVAILABLE** - how many pods are not only updated, but available. Like current relative to up-to-date, the **READY** count may vary from the **AVAILABLE** count during updates. 

{{% notice tip %}}
The UP-TO-DATE and AVAILABLE values are shown in DaemonSet and Deployment workloads by default, but not in ReplicaSet state. Also, as noted above, a Deployment will show a fraction like 1/1 in the READY column for ready/desired replica counts instead of separate READY, CURRENT, and DESIRED columns.
{{% /notice %}}

{{< step >}}What does the definition of a `DaemonSet` look like? Check it out:{{< /step >}}

```bash
kubectl get daemonset kube-proxy -n kube-system -o yaml
```

{{% notice note %}}
As with the pod details earlier, many details have been elided from the following view of this daemonset. 
{{% /notice %}}

{{< output >}}
apiVersion: apps/v1
kind: DaemonSet
metadata:
  generation: 1
  name: kube-proxy
  namespace: kube-system
  resourceVersion: "503"
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    metadata: ...
    spec:
      containers:
        ... command, env, image, etc. ...
      ... other pod spec details ...
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
status:
  currentNumberScheduled: 1
  desiredNumberScheduled: 1
  numberAvailable: 1
  numberMisscheduled: 0
  numberReady: 1
  observedGeneration: 1
  updatedNumberScheduled: 1
{{< /output >}}

As you've already seen the kube-proxy-xyzzy pod (or whatever random name suffix) that this `DaemonSet` created, the `spec.template.metadata` and `spec.template.spec` details are not important right now. Focus on these four aspects:
- The top-level `metadata` has `generation` and `resourceVersion` properties managed by Kubernetes for this provisioned workload. We'll come back to this when you are creating your own `Deployment` later.
- `spec.revisionHistoryLimit` implies that the workload (e.g. daemonset or deployment) can manage rollout *history*.
- `spec.updateStrategy` specifies how the daemonset or deployment will manage updates. Think of canary, blue-green, and other update strategies.
- `status` is maintained for the workload, a subset of which we saw in the *summary* `kubectl get` output.

The primary take-aways about `DaemonSet` for now are:
1. Some of the background pods running your Kubernetes cluster are managed by `DaemonSet`.
2. `kube-proxy` is an example of a `DaemonSet`.
3. You can create your own daemonsets. Later.
4. Daemonsets and deployments share many characteristics. This was an episode in foreshadowing.

Let's move on to the *real* star of this show: deployments.

## What Deployments Keep Things Humming?

Before you create your own deployments, it may help to know at least one example of a deployment already running in your Kubernetes cluster. You didn't start it. It comes with the neighborhood (cluster).

Check if you have any deployments running.

```bash
kubectl get deployments
```

{{< output >}}
No resources found in default namespace.
{{< /output >}}

Is this a surprise? Did you want to have builtin deployments running in your **default** namespace? Which namespace would you expect them in?

{{< step >}}Check for deployments in the `kube-system` namespace.{{< /step >}}
```bash
kubectl get deployments -n kube-system
```

{{< output >}}
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           9d
{{< /output >}}

{{< step >}}Rather than using `kubectl get` with the `-o yaml` option, use `describe` instead:{{< /step >}}

```bash
kubectl describe deployment/coredns -n kube-system 
```

{{% notice note %}}
Many details have been elided from the following output. Please read through the output of *your* command.
{{% /notice %}}

{{< output >}}
Name:                   coredns
Namespace:              kube-system
Selector:               k8s-app=kube-dns
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
RollingUpdateStrategy:  1 max unavailable, 25% max surge
Pod Template:
  Labels:           k8s-app=kube-dns
  Containers:
   coredns:
    ... image, ports, args, mounts ...
  Volumes:
   ... configmap to mount ...
OldReplicaSets:  <none>
NewReplicaSet:   coredns-558bd4d5db (2/2 replicas created)
{{< /output >}}

Some of the most important aspects of a `Deployment` (like this `coredns` one) are:
- The `Deployment` manages a set of **replica** pods for you.
- You can specify the number of pod `replicas`. 
- You can choose an update **strategy** to either replace old pods or gracefully roll out new version pods and decommision old version pods.
- The `Deployment` manages **Old** and **New** replica sets (`kind: ReplicaSet`) according to your update strategy when you make changes to the deployment.
- Some changes do not require a new `ReplicaSet` but can be made in-place. Other changes require the deployment to upgrade to a new `ReplicaSet`.

{{< step >}}Take a look at the `ReplicaSet` the `coredns` deployment is managing:{{< /step >}}

```bash
kubectl get replicasets  -n kube-system
```

{{< output >}}
NAME                 DESIRED   CURRENT   READY   AGE
coredns-558bd4d5db   2         2         2       10d
{{< /output >}}

{{< step >}}Now look at the two `Pod`s that `ReplicaSet` manages:{{< /step >}}

```bash
kubectl get pods -n kube-system | grep -v kube | grep -v kind
```

{{< output >}}
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-qxdcd                     1/1     Running   1          9d
coredns-558bd4d5db-vzdrc                     1/1     Running   1          9d
{{< /output >}}

Note that `kubectl get pods -n kube-system | grep coredns` would show the same `coredns` pods, but without the header line. Certainly, `kubectl get pods -n kube-system` will suffice if you don't mind ignoring other pods.

{{< mermaid >}}
graph TB
deployment(Deployment: coredns)
replicaSetA(ReplicaSet: coredns-558bd4d5db)
podA[Pod: coredns-558bd4d5db-qxdcd]
podB[Pod: coredns-558bd4d5db-vzdrc]

deployment-->|manages|replicaSetA
replicaSetA-->|manages|podA
replicaSetA-->|manages|podB

  classDef green fill:#9f6,stroke:#333,stroke-width:4px;
  classDef orange fill:#f96,stroke:#333,stroke-width:4px;
  classDef blue fill:#69f,stroke:#333,stroke-width:4px;
  classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
  class deployment green;
  class replicaSetA orange;
  class podA,podB blue;
  %% class containerA,containerB yellow;
{{< /mermaid >}}

The primary take-aways about `Deployment` for now are:
1. Some of the background pods running your Kubernetes cluster are managed by `Deployment`.
2. `coredns` is an example of a `Deployment`.
3. You can create your own deployments. Next!

Now that you have looked at a pre-existing deployment, it is time to define your own.

## What's in a Manifest?

To create your own `Deployment`, you ned to add the proper `apiVersion` to your manifest.
Recall that for a `Pod` we have `apiVersion: v1`. This is not the same for the less fundamental kinds of workloads.

{{< step >}}Run the following sequence of commands to create a reference for the Kubernetes workload objects.{{< /step >}}

```bash
kubectl api-resources | head -1 >workloads.txt    
kubectl api-resources | grep batch >>workloads.txt
kubectl api-resources | grep apps >>workloads.txt 
cat workloads.txt 
```

{{< output >}}
NAME                SHORTNAMES   APIVERSION     NAMESPACED   KIND
cronjobs            cj           batch/v1       true         CronJob
jobs                             batch/v1       true         Job
controllerrevisions              apps/v1        true         ControllerRevision
daemonsets          ds           apps/v1        true         DaemonSet
deployments         deploy       apps/v1        true         Deployment
replicasets         rs           apps/v1        true         ReplicaSet
statefulsets        sts          apps/v1        true         StatefulSet
{{< /output >}}

As shown in the table, the API version for `Deployment` objects is `apps/v1`. A short command to check this is: `kubectl api-resources | grep Deployment`, but we used the sequence above to retain the column headings.

So you know the `apiVersion` and the `kind`, but what else goes in a manifest? Like `Pod` manifests, a `Deployment` manifest will also include `metadata` and `spec` sections. However, the details will be different. Here are three other ways to find the details should go in your `Deployment` manifest:
1. Read the Kubernetes [documentation for deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
2. Look at the manifest for an *existing* deployment. For example, you could Use `kubectl get deploy coredns -n kube-system -o yaml` or any other deployment.
3. You can use the ``--dry-run=client` option with a ***generator*** to show an example manifest.

{{< step >}}Use the third technique as follows:{{< /step >}}

```bash
kubectl create deployment demo --image demo:1.0.0 --dry-run=client -o yaml
```

{{< output >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: demo
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: demo
    spec:
      containers:
      - image: demo:1.0.0
        name: demo
        resources: {}
status: {}
{{< /output >}}

Here is a graphical representation of the components of that YAML `Deployment` manifest.

{{< mermaid >}}
graph TB
subgraph Deployment-manifest
  apiVersion(apiVersion: apps/v1)
  kind(kind: Deployment)
  subgraph Deployment-metadata[Deployment metadata]
    name(name)
    deploymentLabels[labels]
  end
  subgraph Deployment-spec[Deployment spec]
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

classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class Deployment-manifest orange;
class Deployment-metadata blue;
class Deployment-spec green;
class template yellow;
{{< /mermaid >}}

The critical ingredients in the deployment `spec` are:
- `replicas` - you specify the desired number of pod replicas
- `strategy` - choose the replacement or update strategy for new versions of your app
- `selector` - the selector allows the deployment to own pods with certain `Pod` labels
- `template` - the template allows the deployment to create new pods with specific metadata and `Pod` spec

## Write your Deployment manifest

Now that you know what goes in a `Deployment` manifest, it is time to create one. We'll skip the update `strategy`. For the `Pod` spec within, let's start simple, without the environment variables, volume mounts, and other fancy decorations.

{{< step >}}Create this deployment manifest and apply it.{{< /step >}}

```bash
cat <<EOF | tee ~/environment/101-demo-deployment.yaml | kubectl -n dev apply -f -
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

{{< step >}}Check the status of your deployment.{{< /step >}}

```bash
kubectl get deployment -n dev
```

{{< output >}}
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
demo   3/3     3            3           27s
{{< /output >}}

{{< step >}}Check the name of the `ReplicaSet` that your `Deployment` created.{{< /step >}}

```bash
kubectl get replicaset -n dev
```

{{< output >}}
NAME              DESIRED   CURRENT   READY   AGE
demo-7c9cd496db   3         3         3       35s
{{< /output >}}

How is the name of this replica set related to the name of your deployment?

{{< step >}}Look at the pods your deployment created (using that replica set):{{< /step >}}

```bash
kubectl get pod -n dev
```

{{< output >}}
NAME                    READY   STATUS    RESTARTS   AGE
demo                    1/1     Running   0          2d9h
demo-7c9cd496db-24kpc   1/1     Running   0          39s
demo-7c9cd496db-fhgcj   1/1     Running   0          39s
demo-7c9cd496db-zf5b5   1/1     Running   0          39s
{{< /output >}}

How are the pod names related to the names of the deployment and the replica set?

{{% notice note %}}
If you did not leave your "demo" `Pod` running from an earlier lesson, you may see *only* the newly created pods that are part of your deployment.
{{% /notice %}}

You have now successfully created a Kubernetes `Deployment` with three replicas!

{{< mermaid >}}
graph TB
deployment(Deployment: demo)
replicaSetA(ReplicaSet: demo-7c9cd496db)
podA[Pod: demo-7c9cd496db-24kpc]
podB[Pod: demo-7c9cd496db-fhgcj]
podC[Pod: demo-7c9cd496db-zf5b5]

deployment-->|manages|replicaSetA
replicaSetA-->|manages|podA
replicaSetA-->|manages|podB
replicaSetA-->|manages|podC

  classDef green fill:#9f6,stroke:#333,stroke-width:4px;
  classDef orange fill:#f96,stroke:#333,stroke-width:4px;
  classDef blue fill:#69f,stroke:#333,stroke-width:4px;
  classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
  class deployment green;
  class replicaSetA orange;
  class podA,podB,podC blue;
  %% class containerA,containerB yellow;
{{< /mermaid >}}

## Success

In this training adventure, you have:
- Learned the purpose of Kubernetes workloads.
- Learned the different types of workloads.
- Inspected a `DaemonSet` in your Kubernetes cluster (`kube-proxy`).
- Looked at an existing `Deployment` in your cluster (`coredns`).
- Created your own `Deployment`.
