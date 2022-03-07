---
title: "Request Sequence"
chapter: false
weight: 760
draft: false
---

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


