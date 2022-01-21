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

`DaemonSet` workloads are typically used to provide useful background services that should be available on most nodes or every node in your Kubernetes clusters. Check to see if you have any daemonsets running in your cluster:

```bash
kubectl get ds
```

{{< output >}}
No resources found in default namespace.
{{< /output >}}

If there are no daemonsets running in the **default** namespace of your cluster, which other namespace could they be in? Check what namespaces you have:

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

Query for the list in `kube-system`:

```bash
kubectl get daemonsets -n kube-system
```

While other Kubernetes clusters may have a different set of daemonsets, your workshop cluster should look similar to this:

{{< output >}}
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kindnet      1         1         1       1            1           <none>                   9d
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   9d
{{< /output >}}

Workloads, like these daemonsets, ***managed*** pods. Check what pods you have in the `kube-system` namespace.

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
Get the details of the `kube-proxy` pod:

```bash
kppod=$(kubectl get pods -n kube-system | grep kube-proxy | sed 's/ .*//')
echo $kppod
```

Notice the `Pod` name is derived from the `DaemonSet` name. The suffix after "kube-proxy" is automatically generated when the `kube-proxy` daemonset creates the pods to implement that on each node.

{{< output >}}
kube-proxy-sc7qw
{{< /output >}}

Confirm again the basic properties of the pod.

```bash
kubectl get pods $kppod -n kube-system            
```

{{< output >}}
NAME               READY   STATUS    RESTARTS   AGE
kube-proxy-sc7qw   1/1     Running   1          9d
{{< /output >}}

As usual, the **READY** column shows the number of containers running within the pod.

Now get further details of that pod:

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

Now, look back at the definition of the `DaemonSet`.

TODO... wrangle this...

DevAccountRole:~/environment $ kubectl get daemonset kube-proxy -n kube-system   
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   9d
DevAccountRole:~/environment $ kubectl get daemonset kube-proxy -n kube-system -o yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2022-01-11T01:02:18Z"
  generation: 1
  labels:
    k8s-app: kube-proxy
  name: kube-proxy
  namespace: kube-system
  resourceVersion: "503"
  uid: 61a669f1-9bf5-468f-af5e-bddbb70e0dd8
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kube-proxy
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
        imagePullPolicy: IfNotPresent
        name: kube-proxy
        resources: {}
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/kube-proxy
          name: kube-proxy
        - mountPath: /run/xtables.lock
          name: xtables-lock
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-node-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: kube-proxy
      serviceAccountName: kube-proxy
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - operator: Exists
      volumes:
      - configMap:
          defaultMode: 420
          name: kube-proxy
        name: kube-proxy
      - hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
        name: xtables-lock
      - hostPath:
          path: /lib/modules
          type: ""
        name: lib-modules
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

--

## What Deployments Keep Things Humming?

TODO

DevAccountRole:~/environment $ kubectl get deployments
No resources found in default namespace.
DevAccountRole:~/environment $ kubectl get deployments -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           9d
DevAccountRole:~/environment $ kubectl describe deployment/coredns -n kube-system 
Name:                   coredns
Namespace:              kube-system
CreationTimestamp:      Tue, 11 Jan 2022 01:02:18 +0000
Labels:                 k8s-app=kube-dns
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               k8s-app=kube-dns
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 25% max surge
Pod Template:
  Labels:           k8s-app=kube-dns
  Service Account:  coredns
  Containers:
   coredns:
    Image:       k8s.gcr.io/coredns/coredns:v1.8.0
    Ports:       53/UDP, 53/TCP, 9153/TCP
    Host Ports:  0/UDP, 0/TCP, 0/TCP
    Args:
      -conf
      /etc/coredns/Corefile
    Limits:
      memory:  170Mi
    Requests:
      cpu:        100m
      memory:     70Mi
    Liveness:     http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
    Readiness:    http-get http://:8181/ready delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /etc/coredns from config-volume (ro)
  Volumes:
   config-volume:
    Type:               ConfigMap (a volume populated by a ConfigMap)
    Name:               coredns
    Optional:           false
  Priority Class Name:  system-cluster-critical
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   coredns-558bd4d5db (2/2 replicas created)
Events:          <none>
DevAccountRole:~/environment $ kubectl get pods coredns -n kube-system
Error from server (NotFound): pods "coredns" not found
DevAccountRole:~/environment $ kubectl get pods -n kube-system | grep coredns
coredns-558bd4d5db-qxdcd                     1/1     Running   1          9d
coredns-558bd4d5db-vzdrc                     1/1     Running   1          9d


## Deploy Your Own Deployment

TODO

## Success

In this training adventure, you have:
- Learned the purpose of Kubernetes workloads.
- Learned the different types of workloads.
- Inspected a `DaemonSet` in your Kubernetes cluster.
- Looked at an existing `Deployment` in your cluster.
- Created your own `Deployment`.
