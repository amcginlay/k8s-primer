---
title: "Running at Scale"
chapter: false
weight: 21
draft: false
---

## Purpose

Now it is time for you to unleash the next level of power in Kubernetes: ***Workloads***.
[Workloads](https://kubernetes.io/docs/concepts/workloads/) in Kubernetes are designed to support your operations for many ***kinds*** of enterprise app, microservice, and devops automation.
To paraphrase the Kubernetes docs, each ***workload*** manages a ***set*** of pods on your behalf.

In the previous section your were introduced to the most frequently used type of workload resource, the `Deployment`, but we barely kicked the tires.
This section briefly introduces the different workload types then provisions a fleet of pods using an updated `Deployment`.

{{< mermaid >}}
graph TB
deployment(Workload)-->|manages|podA[Pod A]
deployment-->|manages|podB[Pod B]
deployment-->|manages|podM[...]
deployment-->|manages|podZ[Pod Z]
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class podA,podB,podM,podZ yellow;
{{< /mermaid >}}

## Workload Resource Types

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

## Built-In Workloads

All Kubernetes clusters possess a `kube-system` namespace.
If you choose to consider Kubernetes to be an operating system, `kube-system` is the equivalent of `C:\Windows\System` and it comes equipped with a set of running workloads which keep everything ticking along.
Before you **scale-out** your own deployments, it may help to know at least one example already running in your Kubernetes cluster.

{{< step >}}Check for deployments in the `kube-system` namespace.{{< /step >}}
```bash
kubectl -n kube-system get deployments
```

{{< output >}}
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           9d
{{< /output >}}

{{< step >}}Use the `describe` command to take a closer look{{< /step >}}

```bash
kubectl -n kube-system describe deployment coredns
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
...
{{< /output >}}

Some of the more important configurable aspects of a `Deployment` (like this `coredns` one) are:
- The `Selector` which is a key value pair which can be used to identify components related to the deployment.
- The `Replicas` which represents the number of instances this deployment's associated `ReplicaSet` must maintain.
- A `StrategyType` of either `RollingUpdate` or `Recreate`, depending on whether the `Deployment` wishes to supports zero-downtime updates or not.

{{< step >}}Use the `Selector` to list just the main objects of this `Deployment`{{< /step >}}

```bash
kubectl -n kube-system get deployments,replicasets,pods -l k8s-app=kube-dns
```

Example output:
{{< output >}}
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   2/2     2            2           40m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-558bd4d5db   2         2         2       40m

NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-558bd4d5db-6xhk2   1/1     Running   0          40m
pod/coredns-558bd4d5db-pr28z   1/1     Running   0          40m
{{< /output >}}

{{% notice tip %}}
Repeat the previous command, adding the `-o wide` (or `--output wide`) switch to view more comprehensive details of the objects.
{{% /notice %}}

Take a close look at the output.
Can you see how the naming convention indicates that these objects are closely related?
These objects were **all** spawned from a single deployment manifest.

{{< mermaid >}}
graph TB
deployment(Deployment)-->|manages|replicaSetA(ReplicaSet)
replicaSetA-->|manages|podA[Pod A]
replicaSetA-->|manages|podB[...]
replicaSetA-->|manages|podM[Pod M]

  classDef green fill:#9f6,stroke:#333,stroke-width:4px;
  classDef orange fill:#f96,stroke:#333,stroke-width:4px;
  classDef blue fill:#69f,stroke:#333,stroke-width:4px;
  classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
  class deployment orange;
  class replicaSetA blue;
  class replicaSetB green;
  class podA,podB,podM,podN,podO,podZ yellow;
{{< /mermaid >}}

## Scaling Your Own Deployment

Now that you have seen a existing deployment running at scale, it is time to **scale out** the deployment you built for your app in the previous chapter.

{{< step >}}Update the deployment manifest and apply it.{{< /step >}}

```yaml
cat <<EOF | tee ~/environment/005-demo-deployment.yaml | kubectl -n dev apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 3                    # scale-out: https://12factor.net/concurrency
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: demo:1.0.0
        env:
        - name: GREETING
          value: Hi from
EOF
```

Example output:
{{< output >}}
deployment.apps/demo configured
{{< /output >}}

{{< step >}}Check the status of your deployment, its replicaset and its pods.{{< /step >}}

```bash
kubectl -n dev get deployments,replicasets,pods
```

Example output:
{{< output >}}
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo   3/3     3            3           3m58s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/demo-658bfb548   3         3         3       3m58s

NAME                       READY   STATUS    RESTARTS   AGE
pod/demo-658bfb548-8hjkx   1/1     Running   0          3m32s
pod/demo-658bfb548-nrhjv   1/1     Running   0          3m58s
pod/demo-658bfb548-rw7x6   1/1     Running   0          3m58s
{{< /output >}}

You have now successfully created a Kubernetes `Deployment` with three replicas.

{{< step >}}Now check you can still `kubectl exec` into one of the pods via its deployment.{{< /step >}}
```bash
kubectl -n dev exec deployment/demo -it -- curl http://localhost:80
```

Example output:
{{< output >}}
Hi from demo-658bfb548-rw7x6
{{< /output >}}

Repeat this command few times.
It appears that **only one** of the pods is used when we connect this way.
Did you expect that, or did you expect `kubectl exec` to distribute the load between the pods.

That is the subject you will tackle in the next chapter.

{{< mermaid >}}
graph TB
deployment(Deployment: demo)
replicaSetA(ReplicaSet: demo-658bfb548)
podA[Pod: demo-658bfb548-8hjkx]
podB[Pod: demo-658bfb548-nrhjv]
podC[Pod: demo-658bfb548-rw7x6]

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
- Looked at an existing `Deployment` in your cluster (`coredns`).
- **Scaled-out** your own `Deployment`.
