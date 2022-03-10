---
title: "Multicontainer Pods"
chapter: false
weight: 717
draft: false
---

## Multi-Container Pods

In the [Pod Infrastructure lesson]({{< ref "appendix/714_pod" >}}), you created a pod and invesigated what it is made of.

Hopefully you learned:
- each container has its own `pid` and `mnt` namespaces
- Kubernetes adds a `pause` container to each of your pods
- each pod has its own `net`, `uts`, and `ipc` namespaces

In the [Pause Containers lesson]({{< ref "appendix/715_pause" >}}), you created another pod and invesigated the purpose of the pause container in each pod.
- the `pause` container retains the consistent network configuration of the pod
- your individual app containers can start, restart, or terminate within the pod lifecycle
- `kubectl get pod` does not count the `pause` container in the running/desired container count
- `kubectl describe pod` does not list the `pause` container

In the [brief interlude on Init Containers]({{< ref "appendix/716_initcontainer" >}}), we pointed out that:
- init containers run before you app containers
- when we talk about ***multicontainer pods*** we are explicitly not counting pause and init containers in the "muliti-" aspect of that

When we say "multicontainer pods" we mean multiple simultaneous parallel app containers in the same pod.
In other words, multiple **app* containers.

## Purpose

Singleton pods which run one-container-per-pod (not counting `pause`) are *quite common* in Kubernetes.
In many people's experience, one-container-pods account for the *vast majority* of pods, if not nearly *all*.

However, there are several scenarios in which you may want to run more than one container per pod.

Imagine that the **main** app container is like the star of a movie. 
They're the one with their name on the poster, in lights on the marquee, and central to the plot.

The additional app containers are like the supporting cast. They help out the main character.

When Kubernetes was younger, circa 2015, some people [distinguished three varieties of helper containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/):
- [**Sidecar** containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/#example-1-sidecar-containers)
- [**Ambassador** containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/#example-2-ambassador-containers)
- [**Adapter** containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/#example-3-adapter-containers)

However, the Kubernetes community has largely come to treat these all as varieties of helper containers.
Nowadays, the term **sidecar container** is used to refer generically to any of those three puposes.
Some people still distinguish them, yet that dogma is dated.

In [current Kubernetes documentation on Pods](https://kubernetes.io/docs/concepts/workloads/pods/),
only **sidecar** containers are mentioned in the [Workload resources for managing pods](https://kubernetes.io/docs/concepts/workloads/pods/#workload-resources-for-managing-pods).

One particular use case of multicontainer pods is
[Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Another use case is using an [Envoy proxy with a service mesh or ingress](https://www.envoyproxy.io).

In this section, you will deploy multiple containers in a pod which do ***not*** fit the logging or proxy patterns. 
Those can be explored in other lessons.

## TODO

$ cat 902-demo-duo.yaml 
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demoduo             # a name for your POD
  labels:
    app: demo
    style: duo
    count: "2"
spec:
  containers:                # a pod CAN consist of multiple containers, this one has TWO
  - name: demox              # a name for your first CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
    command:
    - php                    # where to start the container
    args:
    - -S                     # change the server IP + port binding to listen on
    - 127.0.0.1:9081         # use a unique port per container in the pod
  - name: demoy              # second container name
    image: demo:1.0.0        # would usually be different for distinct role
    command:
    - php
    args:
    - -S
    - 127.0.0.1:9082

$ kubectl apply -n dev -f 902-demo-duo.yaml 
pod/demoduo created

$ cat 903-demo-trio.yaml 
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demotrio            # a name for your POD
  labels:
    app: demo
    style: trio
    count: "3"
spec:
  containers:                # a pod CAN consist of multiple containers, this one has THREE
  - name: demox              # a name for your first CONTAINER
    image: demo:1.0.0        # the tagged image we previously injected using "kind load"
    command:
    - php                    # where to start the container
    args:
    - -S                     # change the server IP + port binding to listen on
    - 127.0.0.1:9081         # use a unique port per container in the pod
  - name: demoy              # second container name
    image: demo:1.0.0        # would usually be different for distinct role
    command:
    - php
    args:
    - -S
    - 127.0.0.1:9082
  - name: demoz              # third container name
    image: demo:1.0.0        # would usually be different for distinct role
    command:
    - php
    args:
    - -S
    - 127.0.0.1:9083
$ 

$ kubectl apply -n dev -f 903-demo-trio.yaml                                                                           
pod/demotrio created

$ kubectl get pods -n dev
NAME       READY   STATUS    RESTARTS   AGE
demoduo    2/2     Running   0          2m42s
demosolo   1/1     Running   0          4h57m
demotrio   3/3     Running   0          7s

$ kubectl get pods -n dev -o wide
NAME       READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
demoduo    2/2     Running   0          3m34s   10.244.2.2   kind-worker2   <none>           <none>
demosolo   1/1     Running   0          4h58m   10.244.3.2   kind-worker3   <none>           <none>
demotrio   3/3     Running   0          59s     10.244.1.2   kind-worker    <none>           <none>

{{< mermaid >}}
graph
subgraph pod[demo Pod]
  subgraph pausecontainer[pause Container]
    pause((PID 1<br>pause))
    pausens{{pid,mnt ns}}
  end
  subgraph phpcontainer[demo Container]
    demo((PID 1<br>php<br>-S 127.0.0.1:9081))
    phpns{{pid,mnt ns}}
  end
  podns{{net,uts,ipc ns}}
end
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class pod yellow;
class phpcontainer,pausecontainer yelloworange;
class demo,pause blue2;
class pausens,phpns,podns lavender;
{{< /mermaid >}}
