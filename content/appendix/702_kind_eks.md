---
title: "KinD and EKS"
chapter: false
weight: 702
draft: false
---

## KinD and Amazon EKS - both Kubernetes

KinD and Amazon EKS both provide you with Kubernetes clusters.
Both technologies have:
- a Kubernetes **control plane** with API Server, Scheduler, and Controller Manager
- a **data plane** with worker nodes, each with a Kubelet, `kube-proxy`, and your workload pods

As Kubernetes clusters, you can manage both KinD and Amazon EKS with `kubectl` and other tools.

{{< columns >}}
{{< column title="KinD = Kubernetes in Docker" >}}
{{< mermaid >}}
graph TB
subgraph cloud9[Cloud9 instance]
  kubectl[kubectl]
  subgraph docker[Docker Engine]
    subgraph k8s-cluster[Kubernetes cluster]
      subgraph control-plane[kind-control-plane in container]
        apiserver[API Server]
      end
      subgraph data-plane[data plane]
        worker1[Worker Node<br>in container]
        worker2[Worker Node<br>in container]
      end
    end
  end
end 
kubectl --> apiserver
apiserver --> worker1
apiserver --> worker2
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class kubectl,apiserver,worker1,worker2 lavender;
class control-plane blue2;
class data-plane green;
class k8s-cluster orange;
class docker cyan;
class cloud9 yelloworange;
{{< /mermaid >}}
{{< /column >}}
{{< column title="Amazon EKS = managed control plane" >}}
{{< mermaid >}}
graph TB
subgraph bastion[bastion host]
  kubectl[kubectl]
end
subgraph k8s-cluster[Kubernetes cluster]
  subgraph control-plane[eks-control-plane managed]
    apiserver[API Server]
  end
  subgraph data-plane[data plane]
    worker1[Worker Node<br>in EC2]
    worker2[Worker Node<br>in EC2]
  end
end
kubectl --> apiserver
apiserver --> worker1
apiserver --> worker2
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class kubectl,apiserver lavender;
class data-plane green;
class k8s-cluster orange;
class docker cyan;
class bastion,control-plane,worker1,worker2 yelloworange;
{{< /mermaid >}}
{{< /column >}}
{{< /columns >}}

## KinD

In your KindD cluster, the control plane and each worker node run as containers in Docker.

Which then in turn host pods of containers.

Some people have likened this kind of nested containerization to Christopher Nolan's *Inception*, a story of a dream within a dream.

## Amazon EKS

In an Amazon EKS cluster: 
- the control plane is managed by AWS on the customer's behalf
- and each worker node runs on either (according to customer choice):
  1. a customer-managed Amazon EC2 instance
  2. an Amazon EC2 instance in an Amazon EC2 autoscaling group (an EKS managed node group)
  3. an AWS Fargate Firecracker node (serverless, on-demand, one-per-pod)

Which then in turn host pods of containers.

## Summary

Whether you run Kubernetes on bare-metal, virtual machines, or containers, the Kubernetes components are the same, only the infrastructure you're using to host it is different.

