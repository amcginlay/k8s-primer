---
title: "Deploy a Fleet of Pods"
chapter: false
weight: 020
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
{{< /mermaid >}}

{{% notice tip %}}
Of these, the two most common and important are `DaemonSet` and `Deployment`!
{{% /notice %}}

With this in mind, to learn more about workloads, you can:
1. Inspect a `DaemonSet` in your Kubernetes cluster.
2. Look at an existing `Deployment` in your cluster.
3. Create your own `Deployment`.

## What Daemons are in Your Cluster?

TODO

## What Deployments Keep Things Humming?

TODO

## Deploy Your Own Deployment

TODO

## Success

In this training adventure, you have:
- Learned the purpose of Kubernetes workloads.
- Learned the different types of workloads.
- Inspected a `DaemonSet` in your Kubernetes cluster.
- Looked at an existing `Deployment` in your cluster.
- Created your own `Deployment`.
