---
title: "Request Sequence"
chapter: false
weight: 760
draft: false
---

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


