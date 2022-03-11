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

## A Tale of Two Containers

{{< step >}}Create a pod with two containers.{{< /step >}}

```bash
cat <<EOF | tee ~/environment/902-demo-duo.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demoduo              # a name for your POD
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
```

{{< output >}}
pod/demoduo created
{{< /output >}}

{{< step >}}Create a pod with three containers.{{< /step >}}

```bash
cat <<EOF | tee ~/environment/903-demo-trio.yaml | kubectl -n dev apply -f -
apiVersion: v1
kind: Pod                    # the object schema Kubernetes uses to validate this manifest
metadata:
  name: demotrio             # a name for your POD
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
``` 

{{< output >}}
pod/demotrio created
{{< /output >}}

{{< step >}}Check the running pods.{{< /step >}}

```bash
kubectl get pods -n dev
```

Example output:
{{< output >}}
NAME       READY   STATUS    RESTARTS   AGE
demoduo    2/2     Running   0          2m42s
demosolo   1/1     Running   0          4h57m
demotrio   3/3     Running   0          7s
{{< /output >}}

{{< step >}}Get more details of those pods, including their IP addresses and the nodes they are running on.{{< /step >}}

```bash
kubectl get pods -n dev -o wide
```

{{< output >}}
NAME       READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
demoduo    2/2     Running   0          3m34s   10.244.2.2   kind-worker2   <none>           <none>
demosolo   1/1     Running   0          4h58m   10.244.3.2   kind-worker3   <none>           <none>
demotrio   3/3     Running   0          59s     10.244.1.2   kind-worker    <none>           <none>
{{< /output >}}


{{< step >}}Capture the "duo" pod's node name in a shell variable `demonode`:{{< /step >}}

```bash
demonode=$(kubectl get pod demoduo -n dev -o jsonpath={.spec.nodeName})
```

{{< step >}}Get a list of processes running on that node, focused on the last pod.{{< /step >}}

```bash
docker exec $demonode ps -eo pid,ppid,cmd | head -1; docker exec $demonode ps -eo pid,ppid,cmd | tail -5
```

{{% notice note %}}
The use of `ps` piped to `head -1` should yield the header; the second use of `ps` piped to `tail -5` is intended to get the duo of `php` processes, their `pause` sibling, and hopefully their "runc shim" and along with the second instance of `ps`. If your output doesn't include that, simply use `docker exec $demonode ps -eo pid,ppid,cmd` instead. This adjustment may be necessary if you have a different mix of pods per node than our example.
{{% /notice %}}

Example output:
{{< output >}}
    PID    PPID CMD
  18318       1 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id fb355a21bc980cab173c1b29476b74c3cd7c269cd83089cad6be363e679ae870 -address /run/containerd/containerd.sock
  18340   18318 /pause
  18380   18318 php -S 127.0.0.1:9081
  18412   18318 php -S 127.0.0.1:9082
 182523       0 ps -eo pid,ppid,cmd
{{< /output >}}

The parent process ids of your two `php` processes and the `pause` process should all be the same--
the `runc` watcher shim.

{{< step >}}Get a list of namespaces on that node.{{< /step >}}

```bash
docker exec $demonode lsns | head -1; docker exec $demonode lsns | tail -9
```

{{% notice note %}}
Again, if your output is not similar to the example below, try simply doing `docker exec $demonode lsns` alone without doing it twice with head and tail shenanigans.
{{% /notice %}}

{{< output >}}
        NS TYPE   NPROCS   PID USER  COMMAND
4026532905 net         3 18340 65535 /pause
4026532981 mnt         1 18340 65535 /pause
4026532982 uts         3 18340 65535 /pause
4026532983 ipc         3 18340 65535 /pause
4026532984 pid         1 18340 65535 /pause
4026532985 mnt         1 18380 root  php -S 127.0.0.1:9081
4026532986 pid         1 18380 root  php -S 127.0.0.1:9081
4026532987 mnt         1 18412 root  php -S 127.0.0.1:9082
4026532988 pid         1 18412 root  php -S 127.0.0.1:9082
{{< /output >}}

{{< step >}}Correlate the process ids in the `lsns` output with those in the `ps` output.{{< /step >}}

Do the boundaries of the containers and their pod align as you would expect?

{{< step >}}Do the NPROCS values in the `lsns` output make sense?{{< /step >}}

How many containers share that `net`, `uts`, and `ipc` namespaces?

{{< step >}}Repeat the process for the "trio" pod with the three containers.{{< /step >}}

```bash
demonode=$(kubectl get pod demotrio -n dev -o jsonpath={.spec.nodeName})
docker exec $demonode ps -eo pid,ppid,cmd | head -1; docker exec $demonode ps -eo pid,ppid,cmd | tail -6
docker exec $demonode lsns | head -1; docker exec $demonode lsns | tail -11
```

Example output:
{{< output >}}
    PID    PPID CMD
  18463       1 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 50a6ea311acbf5d80315c310e76615faf02fbade73c0d1182514d874ec4eb7cf -address /run/containerd/containerd.sock
  18484   18463 /pause
  18519   18463 php -S 127.0.0.1:9081
  18551   18463 php -S 127.0.0.1:9082
  18582   18463 php -S 127.0.0.1:9083
 204421       0 ps -eo pid,ppid,cmd
        NS TYPE   NPROCS   PID USER  COMMAND
4026532990 net         4 18484 65535 /pause
4026533066 mnt         1 18484 65535 /pause
4026533067 uts         4 18484 65535 /pause
4026533068 ipc         4 18484 65535 /pause
4026533069 pid         1 18484 65535 /pause
4026533070 mnt         1 18519 root  php -S 127.0.0.1:9081
4026533071 pid         1 18519 root  php -S 127.0.0.1:9081
4026533072 mnt         1 18551 root  php -S 127.0.0.1:9082
4026533073 pid         1 18551 root  php -S 127.0.0.1:9082
4026533074 mnt         1 18582 root  php -S 127.0.0.1:9083
4026533075 pid         1 18582 root  php -S 127.0.0.1:9083
{{< /output >}}

As before with the duo pod, the pids and namespaces for this trio pod should correlate.

{{< step >}}For the trio pod, do the NPROCS values in the `lsns` output make sense?{{< /step >}}

How many containers share that `net`, `uts`, and `ipc` namespaces?

From *within* each pod's container, each `php` process should be running as PID 1.

{{< step >}}Check inside each pod's containers.{{< /step >}}

```bash
kubectl exec -n dev demoduo -c demox -- ps -ef
kubectl exec -n dev demoduo -c demoy -- ps -ef
kubectl exec -n dev demotrio -c demox -- ps -ef
kubectl exec -n dev demotrio -c demoy -- ps -ef
kubectl exec -n dev demotrio -c demoz -- ps -ef
```

{{< output >}}
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Mar09 ?        00:00:01 php -S 127.0.0.1:9081
root          15       0  0 02:00 ?        00:00:00 ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Mar09 ?        00:00:01 php -S 127.0.0.1:9082
root           9       0  0 02:00 ?        00:00:00 ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Mar09 ?        00:00:01 php -S 127.0.0.1:9081
root           8       0  0 02:00 ?        00:00:00 ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Mar09 ?        00:00:01 php -S 127.0.0.1:9082
root           9       0  0 02:00 ?        00:00:00 ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Mar09 ?        00:00:01 php -S 127.0.0.1:9083
root          10       0  0 02:00 ?        00:00:00 ps -ef
{{< /output >}}

Here is a graphical depiction.

{{< columns >}}
{{< column title="demosolo" >}}
{{< mermaid >}}
graph LR
subgraph pod[demosolo Pod]
  subgraph pausecontainer[pause Container]
    pause((PID 1<br>pause))
    pausens{{pid,mnt ns}}
  end
  subgraph phpcontainer[demox Container]
    demox((PID 1<br>php<br>-S 127.0.0.1:9081))
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
class demox,pause blue2;
class pausens,phpns,podns lavender;
{{< /mermaid >}}
{{< /column >}}
{{< column title="demoduo" >}}
{{< mermaid >}}
graph LR
subgraph pod[demoduo Pod]
  subgraph pausecontainer[pause Container]
    pause((PID 1<br>pause))
    pausens{{pid,mnt ns}}
  end
  subgraph phpcontainer[demox Container]
    demox((PID 1<br>php<br>-S 127.0.0.1:9081))
    phpns{{pid,mnt ns}}
  end
  subgraph phpcontainer2[demoy Container]
    demoy((PID 1<br>php<br>-S 127.0.0.1:9082))
    phpns2{{pid,mnt ns}}
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
class phpcontainer,phpcontainer2,pausecontainer yelloworange;
class demox,demoy,pause blue2;
class pausens,phpns,phpns2,podns lavender;
{{< /mermaid >}}
{{< /column >}}
{{< column title="demotrio" >}}
{{< mermaid >}}
graph LR
subgraph pod[demotrio Pod]
  subgraph pausecontainer[pause Container]
    pause((PID 1<br>pause))
    pausens{{pid,mnt ns}}
  end
  subgraph phpcontainer[demox Container]
    demox((PID 1<br>php<br>-S 127.0.0.1:9081))
    phpns{{pid,mnt ns}}
  end
  subgraph phpcontainer2[demoy Container]
    demoy((PID 1<br>php<br>-S 127.0.0.1:9082))
    phpns2{{pid,mnt ns}}
  end
  subgraph phpcontainer3[demoz Container]
    demoz((PID 1<br>php<br>-S 127.0.0.1:9083))
    phpns3{{pid,mnt ns}}
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
class phpcontainer,phpcontainer2,phpcontainer3,pausecontainer yelloworange;
class demox,demoy,demoz,pause blue2;
class pausens,phpns,phpns2,phpns3,podns lavender;
{{< /mermaid >}}
{{< /column >}}
{{< /columns >}}

## Success

In this lesson, you confirmed that:
- Multicontainer pods have a `pause` container
- Your app containers in your pod share the same network `net`, hostname `uts`, and interprocess communications `ipc` namespaces
- Each container has its own process `pid` and file system mount `mnt` namespaces

