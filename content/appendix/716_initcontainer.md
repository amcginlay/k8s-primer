---
title: "Init Containers"
chapter: false
weight: 716
draft: false
---

## Purpose

The main intention of this section is merely to convey some context on the relationship between pause, init, and app containers in Kubernetes. 
At this time, there is no tutorial or detailed examples provided.

The positioning of this init container interlude is quite intentionally after coverage of pod infrastructure and the purpose of pause containers, yet before multicontainer pods.

The primary purpose of providing this context is so that you can distinguish the meaning of multicontainer pods from pause and init containers.

## Init Containers

There are three kinds of containers in Kubernetes:
- The Pod's `pause` container that starts before all other containers in the pod.
    - you don't declare the pause container; Kubernetes manages it for you
- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) that must run before the main containers.
    - you can specify init containers in the `initContainers` list of your pod's spec
- App Containers, which are the the main containers of your pod. These are often called regular containers or simply containers.
    - you specify app containers in the `containers` list of your pod's spec

Here are some important aspects of `initContainers` you should know:
- Init containers must run before your app containers are started
- If you specify multiple init containers for a pod, they run in sequence, not in parallel
- Therefore, you can use init containers to set up prerequisites for your app containers
- You can populate a cache, a database, a query pipeline, and more using init containers
- You can plumb out network connectivity needed by your app containers

## Pod Lifecycle

Here is the sequence in which a pod's containers run:
- A pod starts off with just its `pause` container; that remains throughout the life of the pod.
- Then the init containers run.
- Then the app containers run.

{{< columns >}}
{{< column title="pause starts" >}}
{{< mermaid >}}
graph LR
subgraph pod[demo Pod]
  subgraph pausecontainer[pause Container]
    pause((PID 1<br>pause))
    pausens{{pid,mnt ns}}
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
class pausecontainer yelloworange;
class pause blue2;
class pausens,podns lavender;
{{< /mermaid >}}
{{< /column >}}
{{< column title="init containers run" >}}
{{< mermaid >}}
graph LR
subgraph pod[demo Pod]
  subgraph pausecontainer[pause Container]
    pause((PID 1<br>pause))
    pausens{{pid,mnt ns}}
  end
  subgraph initcontainer[init Container]
    setup((PID 1<br>setup))
    initns{{pid,mnt ns}}
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
class initcontainer,pausecontainer yelloworange;
class setup,pause blue2;
class pausens,initns,podns lavender;
{{< /mermaid >}}
{{< /column >}}
{{< column title="app containers run" >}}
{{< mermaid >}}
graph LR
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
{{< /column >}}
{{< /columns >}}

{{% notice note %}}
Note that the interaction between init containers and the subsequent app containers in the pod 
is most often through files created or configured by the init container(s) 
in a **volume** shared by all containers in the pod.
However, in this simple depiction, no such volumes are shown.
{{% /notice %}}

## Success

This section is merely informational. 
If you would like a tutorial or detailed lesson on init containers, please submit a request.
