---
title: "Request Sequence"
chapter: false
weight: 760
draft: false
---

## Where are the APIs?

Kubernetes environments are based on web services.
There are many aspects to this:
- The containerized microservices that make up the workloads running *in* a Kubernetes cluster are usually a combination of one or more of the following:
    - Web-based application programming interfaces (APIs) based on JavaScript Object Notation (JSON), the eXtensible Markup Language (XML), or similar.
    - Web-based app components based on HTML, CSS, JavaScript, etc.
    - Networked data layer access using other non-web TCP-based protocols.
    - Networked messaging using UDP-based protocols.
Not only are the workloads *in* each Kubernetes cluster mostly web-based, but the operations and infrastructure *of* the Kubernetes cluster are based on web APIs.
This includes the Kubernetes administrative and management interfaces, including but not limited to:
- Kubernetes API into the cluster's control plane
- communications within the Kubernetes control plane
- Kubelet API from control plane to worker nodes
- Container Runtime Interface (CRI) from Kubelet to container runtime
Though much simpler than Kubernetes, even Docker operations are based on web APIs.
- The Docker API is similar to the Kubernetes CRI
In addition, most of the devops automation, monitoring, observability, proxy, and service mesh communications are based on web APIs. For example:
- Container readiness and liveness probes 
- Prometheus metrics scraping, collection, and aggregation
- Fluent Bit logging and forwarding

In summary, the workloads you run, the Kubernetes infrastructure itself, and your ability to monitor and measure any and all of that is based on web APIs. Therefore, you need to understand web APIs to work with Kubernetes beyond a conceptual level.

TODO: put together a good picture...

{{< mermaid >}}
graph 
kubectl([kubectl])
client0([Internet client]) 
client1([Mobile client]) 
subgraph cluster[Kubernetes cluster]
  pod0[Pod A]
  pod2[Pod B]
  pod3[Pod C]
end 
kubectl --> pod0
client0 --> pod2
client1 --> pod2
pod3 -->|internal client| pod2
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class kubectl,pod0 blue2;
class client0,client1,pod3 green;
class pod2 orange;
{{< /mermaid >}}

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph pod[demo Pod]
  subgraph container[demo Container]
    apache((PID 1<br>apache2<br>root))
    demo((PID 23<br>php<br>www-data))
    kill((PID 35<br>kill 1<br>root))
  end
end
cloud9 -->|kubectl exec| kill
kill -->|kill 1| apache
apache -.-|child| demo
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class pod yellow;
class container yelloworange;
class apache,demo,kill blue2;
{{< /mermaid >}}

## Docker API Request Sequence

Kubernetes is notably more complicated than Docker.
This complexity is justified by the higher level of service provided by Kubernetes as an orchestrator.

For example, Docker provides a simple container runtime.
You manage a single Docker container host at a time.
There is no persistence of containers beyond that host.

Here is a sketch of the interactions between components in a Docker environment.

{{< mermaid >}}
sequenceDiagram
    participant user
    participant docker
    participant containerd
    participant runc
    participant container
    user->>docker: docker run myimage
    note right of docker: uses Docker Engine API
    docker->>containerd: API call /containers/create myimage
    rect rgb(200, 150, 255)
        note right of containerd: Docker Engine<br/>container runtime
        containerd->>runc: create container
        rect rgb(255, 100, 255)
            runc->>container: load ReadOnly image myimage
            runc->>container: create ReadWrite container layer
        end
        runc-->>containerd: id = 7e4a358dc9f1
    end
    containerd-->>docker: created container id = 7e4a358dc9f1
    docker->>containerd: API call /containers/7e4a358dc9f1/start
    rect rgb(200, 150, 255)
        containerd->>runc: start 7e4a358dc9f1
        rect rgb(255, 100, 255)
            runc->>container: start container 7e4a358dc9f1
        end
        runc-->>containerd: started
    end
    containerd-->>docker: started id = 7e4a358dc9f1
    docker-->>user: 7e4a358dc9f1
{{< /mermaid >}}

[Develop with Docker Engine API](https://docs.docker.com/engine/api/) describes the Docker API.

## Kubernetes API Request Sequence

{{< mermaid >}}
sequenceDiagram
    participant kubectl
    participant apiserver
    participant etcd
    participant controllermanager
    participant scheduler
    participant kubelet
    kubectl->>apiserver: create Pod
    apiserver->>etcd: add Pod to cluster config
    etcd-->>apiserver: okay added Pod p
    apiserver-->>kubectl: created Pod p
    loop Scheduler
        Note right of scheduler: scheduler polls state
        scheduler-->>apiserver: anything to schedule?
        apiserver->>etcd: what's not scheduled?
        etcd-->>apiserver: here's a new Pod!
        apiserver-->>scheduler: here's a new Pod!
    end
    Note right of scheduler: scheduler picks node
    scheduler->>apiserver: ask Node n to run Pod p
    apiserver->>kubelet: hey Node n please run Pod p
    apiserver->>etcd: Pod p is scheduled on Node n
    Note right of kubelet: Node runs Pod
    kubelet-->>apiserver: okay it is running
    apiserver->>etcd: mark pod P as running on Node n
{{< /mermaid >}}

## Original example from Mermaid docs of sequence diagram

{{< mermaid >}}
sequenceDiagram
    participant Alice
    participant Bob
    participant John
    Alice->>John: Hello John, how are you?
    loop Healthcheck
        John->>John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail!
    John-->>Alice: Great!
    John->>Bob: How about you?
    Bob-->>John: Jolly good!
{{< /mermaid >}}


