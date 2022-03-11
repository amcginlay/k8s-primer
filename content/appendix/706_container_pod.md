---
title: "Containers and Pods"
chapter: false
weight: 706
draft: false
---

## Docker Containers and Kubernetes Pods

Containers are made of processes isolated with Linux namespaces and governed by control groups.

Exercises on each of those topics are included in the lessons:
- [Anatomy of a Process]({{< ref "013_anatomy_of_a_process" >}})
- [Linux Namespaces]({{< ref "014_linux_namespaces" >}})

Have you ever asked any of these questions?
- Is a container in Docker the same thing as a container in Kubernetes?
- What is a pod? 
- How is a pod different than a container? 
- How are pods and containers related?

Consider the following diagrams for comparison. 

{{% notice tip %}}
The abbreviation "ns" stands for *namespace* or *namespaces*,
which are the distinct Linux namespace(s) associated with the pod or container.
These are shown like badges the pods and containers wear.
In reality, these namespaces are the very essence of the pod or container's *boundary*.
For example, `pid,mnt ns` is shown on a Kubernetes pod. The pod is *made of* those namespaces.
Similarly, `pid,mnt ns` is shown on a Kubernetes container. The container is *made of* those namespaces.
{{% /notice %}}

{{< columns >}}
{{< column title="Docker container" >}}
Each container for itself.
{{< mermaid >}}
graph LR
subgraph phpcontainer[demo Container]
  demox((PID 1<br>php<br>-S 127.0.0.1:80))
  phpns{{pid,mnt,net,uts,ipc ns}}
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
{{< column title="Kubernetes singlet pod" >}}
One container per pod.
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
{{< column title="Kubernetes doublet pod" >}}
Two containers per pod.
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
{{< /columns >}}

## Docker container

A Docker container is made from `pid`, `mnt`, `net`, `uts`, and `ipc` namespaces.
These each provide the processes in the container with these attributes:

Related lesson:
- [Run Containers]({{< ref "016_run_containers" >}})

## Kubernetes singlet pod

A Kubernetes pod is made from `net`, `uts`, and `ipc` namespaces.
- Each container in the pod is made of `pid` and `mnt` namespaces.
- In addition to your app container, Kubernetes adds a `pause` container to your pod.
- The `pause` container prevents your pod's `net`, `uts`, and `ipc` namespaces from being abandoned and recycled if your app container is restarted.
- Each pod has its own unique IP address (`net` namespace).
- Each pod has its own unique hostname (`uts` namespace).

Related lessons:
- [Orchestrate Containers]({{< ref "019_orchestrate_containers" >}}) -- covers the basics of Kubernetes pods
- [Pod Infrastructure]({{< ref "appendix/714_pod" >}}) -- introduces this division of namespace responsibility between the pod and the containers inside it
- [Pause Containers]({{< ref "appendix/715_pause" >}}) -- illustrates the purpose of the pause container

## Kubernetes doublet pod

When you create multiple app containers within a pod, they are all scheduled together and run (execute) together.
- The app containers in the pod all run simultaneously.
- Each container shares the same network (`net`) namespace:
    - Containers in the pod share the same IP address
    - Containers in the pod share the same TCP and UDP port space
    - Containers can communicate with one another using loopback interfaces (e.g. `127.0.0.1` or `::1`)
    - Inbound communications to the pod's IP address is routed to the proper container by port number
    - Inbound management (e.g. `kubectl exec`) is dispatched to the proper container by name or default designation
- Each container shares the same hostname, as they are in the same `uts` namespace
- One use case is **sidecar** containers which help the **main** container, like supporting characters supporting a main character; this topic is introduced in the [Multicontainer Pods]({{< ref "appendix/717_multicontainer" >}}) lesson

Related lessons:
- [Init Containers]({{< ref "appendix/716_initcontainer" >}})
- [Multicontainer Pods]({{< ref "appendix/717_multicontainer" >}})
- [Pod Networking]({{< ref "appendix/720_pod_networking" >}}) -- introduces pod networking
- [Loopback Networking]({{< ref "appendix/722_loopback" >}}) -- an interlude into loopback interfaces
- [Virtual ethernet (veth) pairs]({{< ref "appendix/730_veth_pairs" >}}) -- introduces `veth` interfaces
- [Veth Encore]({{< ref "appendix/732_veth_encore" >}}) -- in which you explore Kubernetes-created veths

## Success

You have reviewed the high-level relationships between containers and pods.
Please explore the lessons linked herein for further details.
