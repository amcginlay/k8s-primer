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
deployment(Deployment)-->|manages|podA[PodA]
deployment-->|manages|podB[PodB]
deployment-->|manages|podM[...]
deployment-->|manages|podZ[PodZ]
{{< /mermaid >}}

## Provisioning Workloads

Each `kind` of workload manages a set of pods in a paricular way.  Kubernetes workloads can satisfy specific use cases:
- [`Job`](https://kubernetes.io/docs/concepts/workloads/controllers/job/) -- the lifetime of a `Job` workload is to manage a pod that runs once to completion. 
{{< expand "Read more" >}}
    - Alternatively, a `Job` can manage *multiple parallel* pods. Set `.spec.parallelism` greater than 1.
    - The `Job` tracks the number of pods that have completed their task.
    - Use the `.spec.completions` property to manage job success (not needed for one-pod-jobs).
    - Suspending a `Job` will delete its active pods until you resume the `Job`.
    - When you delete a `Job`, it cleans up the pods created for the job.
{{< /expand >}}
- [`CronJob`](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) -- you can schedule a pod or set of pods to run a certain time or on a repeating schedule.
{{% expand "Read more" %}}
    - You set the `.spec.schedule` property of the `CronJob` to a [cron expression](https://pkg.go.dev/github.com/robfig/cron/v3#hdr-CRON_Expression_Format) value.
    - The `CronJob` manages a `Job` object at each repeated interval on the schedule you established.
    - Those `Job` objects manage their pod(s) in turn.
{{% /expand %}}
- [`StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) -- when you have **stateful** workloads, you could choose to manage those with a Kubernetes `StatefulSet` workload.
{{< expand "Read more" >}}
    <ul>
    <li>Quoting the K8s docs, <code>StatefulSet</code> "pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling." </li>
    <li>The key advantage of a <code>StatefulSet</code> is that it maintains stable persistence of <strong>unique pod naming</strong> and <strong>storage</strong> despite potential pod rescheduling.</li>
    <li>You must establish persistent volumes and specific pod network naming with a headless service to support that required persistence for your workload's state.</li>
    </ul>
{{< /expand >}}
- [`DaemonSet`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) -- you can run a replica pod on each node in your Kubernetes cluster.
{{% expand "Read more" %}}
    - By default a replica `Pod` which matches your `DaemonSet` template is deployed to all nodes in your cluster.
    - You can choose a subset of nodes rather than ***all*** nodes. Use the `.spec.template.spec.nodeSelector` or `.spec.template.spec.affinity` to control this.
    - There are some default daemonsets in Kubernetes and more in Amazon EKS. Some daemonsets handle network plumbing, monitoring, and logging capabilities.
    - Your clusters can have your own custom daemonets, core Kubernetes daemonsets, and daemonsets for extensions.
{{% /expand %}}
- [`ReplicaSet`]()
{{% expand "Read more" %}}
{{% /expand %}}
- [`Deployment`]()
{{% expand "Read more" %}}
{{% /expand %}}

{{% notice tip %}}
Of these, the two most common and important are `DaemonSet` and `Deployment`!
{{% /notice %}}

## Deployment

TODO

## Success

TODO
