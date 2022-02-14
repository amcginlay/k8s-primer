---
title: "Kubernetes Architecture"
chapter: false
weight: 701
draft: false
---

## The Cluster is the Way

Kubernetes is designed to *orchestrate* management of large-scale *containerized compute* operations, however you can create and operate medium and small-scale configurations as you wish. 
You deploy Kubernetes as a **cluster**.
You can scale your Kubernetes cluster to meet your specific needs. 
You can divide your workloads into as few or as many Kubernetes clusters as you need.

Like peas in a pod, or whales in a pod, you provision your apps, app components, or microservices into Kubernetes as **pods** of compute containers. Each pod might be running one, two, three, or more containers based on your needs. You could measure your cluster by the number of pods you have running. Consider the following sketches of clusters with a few pods, hundreds of pods, and thousands of pods. How many app instances do you need running simultaneously: 
- during busy hour? 
- during your peak day of the season?
- in idle times?

Accordingly, the size of your cluster may vary over time.

{{< columns >}}
{{< column >}}
{{< mermaid >}}
graph LR
subgraph small[Small Cluster]
  pod0[Pod A]
  pod1[Pod B]
end
{{< /mermaid >}}
{{< /column >}}
{{< column >}}
{{< mermaid >}}
graph LR
subgraph medium[Medium Cluster]
  pod2[Pod A]
  pod3[Pod B]
  hundreds[... hundreds of pods ...]
  pod7[Pod ZY]
  pod8[Pod ZZ]
end 
{{< /mermaid >}}
{{< /column >}}
{{< column >}}
{{< mermaid >}}
graph LR
subgraph large[Large Cluster]
  pod9[Pod A]
  pod10[Pod B]
  thousands[... thousands of pods ...]
  pod23[Pod YYY]
  pod24[Pod YYZ]
end
{{< /mermaid >}}
{{< /column >}}
{{< /columns >}}

{{% notice note %}}
The following steps assume that you have a Kubernetes cluster running on `kind` and have `kubectl` installed to manage that cluster. If you have not yet done so, please visit the [Install Kubernetes]({{< ref "017_meet_kubernetes" >}}) section.
{{% /notice %}}

{{< step >}}Check how many pods you have running in your cluster.{{< /step >}}
```bash
kubectl get pods -A | grep -v NAME | wc -l
```

You might get a response such as 0, 1, or maybe even 26.

{{< output >}}
26
{{< /output >}}

{{% notice note %}}
The `-A` option of the `kubectl get pods` command gives you a list of the pods in *all* namespaces; namespaces are logical subdivisions you can create in your Kubernetes cluster.
The `grep -v` is used to remove the column headings from the list of pods.
The classic UNIX word count program (`wc`) can be used to count characters (or bytes), words, and lines, and the `-l` option yields just the number of lines of output, thus in this case the number of pods.
{{% /notice %}}

## Controlling your Kubernetes cluster

You can use the `kubectl` command-line tool to manage your cluster.
Your clients can connect into the cluster. Some of your pods may even be clients of other pods.

{{< mermaid >}}
graph TB
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
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class kubectl,pod0 blue2;
class client0,client1,pod3 green;
class pod2 orange;
{{< /mermaid >}}

## Division of Labor

Like a business made up of individual contributors and managers, Kubernetes clusters run workload pods and administrative pods in two separate planes:
- worker pods are in the **data plane**
- admin pods are in the **control plane**

{{< mermaid >}}
graph TB
kubectl([kubectl])
client0([Internet client]) 
client1([Mobile client]) 
subgraph cluster[Kubernetes cluster]
  subgraph control[Control Plane]
    pod0[Pod A]
  end
  subgraph data[Data Plane]
    pod2[Pod B]
    pod3[Pod C]
  end
end 
kubectl --> pod0
client0 --> pod2
client1 --> pod2
pod3 -->|internal client| pod2
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class kubectl,pod0 blue2;
class control,data yellow;
class client0,client1,pod3 green;
class pod2 orange;
{{< /mermaid >}}

Just as the government of a city is typically bigger than a mayor, the control plane of your Kubernetes cluster is not implemented with just one pod.

## Explore Control Plane Components

{{< step >}}Look at the components of the control plane in your Kubernetes cluster.{{< /step >}}
```bash
kubectl get pods -n kube-system | grep kind-control-plane
```

The output may be similar to the following:

{{< output >}}
etcd-kind-control-plane                      1/1     Running   0          5d21h
kube-apiserver-kind-control-plane            1/1     Running   0          5d21h
kube-controller-manager-kind-control-plane   1/1     Running   0          5d21h
kube-scheduler-kind-control-plane            1/1     Running   0          5d21h
{{< /output >}}

{{< mermaid >}}
graph TB
kubectl([kubectl])
client0([Internet client]) 
subgraph cluster[Kubernetes cluster]
  subgraph control[Control Plane]
    apiserver[K8s API Server<br>apiserver]
    etcd[(Cluster Config DB<br>etcd)]
    ctlmgr[Controller Manager<br>controller-manager]
    sched{Scheduler<br>scheduler}
  end
  subgraph data[Data Plane]
    pod2[Pod B]
    pod3[Pod C]
  end
end 
kubectl --> apiserver
client0 --> pod2
pod3 -->|internal client| pod2
apiserver <--> etcd
apiserver <--> ctlmgr
apiserver <--> sched
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class kubectl,apiserver,etcd,ctlmgr,sched blue2;
class control,data yellow;
class client0,pod3 green;
class pod2 orange;
{{< /mermaid >}}

The compute power is provided to both the control plane and data plane by **nodes**. Each node is simply a server with certain Kubernetes software installed and running. A node could be a server such as an:
- a single-board computer such as a Raspberry Pi,
- a physical server,
- a virtual machine,
- an Amazon EC2 instance,
- an AWS Fargate pod,
- or even a Docker container.

In some Kubernetes clusters, nodes might be designated exclusively as control plane nodes, data plane nodes (a.k.a worker nodes), or able to run both control plane and data plane components.

## Data Plane Components - Workers

{{< step >}}Find the name of a worker node.{{< /step >}}

```bash
docker ps
```

{{< output >}}
CONTAINER ID   IMAGE                  COMMAND                  CREATED      STATUS      PORTS                       NAMES
07042d75df60   kindest/node:v1.21.1   "/usr/local/bin/entr…"   6 days ago   Up 6 days                               kind-worker3
5fe4c4f33954   kindest/node:v1.21.1   "/usr/local/bin/entr…"   6 days ago   Up 6 days                               kind-worker
f2851e403d05   kindest/node:v1.21.1   "/usr/local/bin/entr…"   6 days ago   Up 6 days                               kind-worker2
5566dc7a7c12   kindest/node:v1.21.1   "/usr/local/bin/entr…"   6 days ago   Up 6 days   127.0.0.1:41101->6443/tcp   kind-control-plane
{{< /output >}}

The `kind-worker` listed in the NAMES column is as good a worker node as any other. Use that in the next step.

{{< step >}}Look at the components in the data plane as processes within a worker node.{{< /step >}}

```bash
docker exec kind-worker ps -ef | grep -v apache
```

{{< output >}}
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Feb01 ?        00:00:00 /sbin/init
root         165       1  0 Feb01 ?        00:00:00 /lib/systemd/systemd-journald
root         177       1  0 Feb01 ?        00:24:02 /usr/local/bin/containerd
root         240       1  1 Feb01 ?        01:55:23 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --fail-swap-on=false --node-ip=172.18.0.3 --node-labels= --pod-infra-container-image=k8s.gcr.io/pause:3.4.1 --provider-id=kind://docker/kind/kind-worker --fail-swap-on=false --cgroup-root=/kubelet
root         322       1  0 Feb01 ?        00:01:43 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 479176cfa170f2c260d4c7468e1b71c23ffff6b4a476b9fba5c8656f638cebbd -address /run/containerd/containerd.sock
65535        342     322  0 Feb01 ?        00:00:00 /pause
root         367       1  0 Feb01 ?        00:01:40 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 7d92b75761678a5d54eaa001d25d4ec71497c4730ce900d332c91d48dc0786ab -address /run/containerd/containerd.sock
65535        392     367  0 Feb01 ?        00:00:00 /pause
root         462     322  0 Feb01 ?        00:02:40 /bin/kindnetd
root         476     367  0 Feb01 ?        00:01:05 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=kind-worker
root       28827       1  0 Feb01 ?        00:01:34 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id ebf4f21de599c84d92cabbefcd91108371954feb45c94974c40b4c0d4cfa92b3 -address /run/containerd/containerd.sock
65535      28849   28827  0 Feb01 ?        00:00:00 /pause
root      146913       1  0 Feb02 ?        00:01:21 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id d37c7db158b030ef98f82cf77200c3f00b166d6779f45040273413f02719c715 -address /run/containerd/containerd.sock
65535     146934  146913  0 Feb02 ?        00:00:00 /pause
root      146976       1  0 Feb02 ?        00:01:22 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 12c23772eb9be2193cad2b1eecbbec6c4d978372f915161eea58a294567b0f24 -address /run/containerd/containerd.sock
65535     146997  146976  0 Feb02 ?        00:00:00 /pause
root     1024263       1  0 Feb06 ?        00:00:17 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id e7196a7d1a192f1ff3def5a357c5bf6d340e467b4fa6549c2b8a6edeb010f241 -address /run/containerd/containerd.sock
65535    1024285 1024263  0 Feb06 ?        00:00:00 /pause
1001     1024370 1024263  0 Feb06 ?        00:00:23 /dashboard --insecure-bind-address=0.0.0.0 --bind-address=0.0.0.0 --auto-generate-certificates --namespace=kubernetes-dashboard
root     1333030       0  0 17:52 ?        00:00:00 ps -ef
{{< /output >}}

Your results may vary. We pruned down the list of processes in the worker node using `grep -v apache`. Even so, there are many details. Rather than reduce redundancy, we have retained so detail in the example output above to support potential discussion. Ask yourself: what process hierarchy is represented here? Hint: follow the PID and PPID values. Are separate Linux namespaces and control groups (cgroups) at play here? We shall not answer those questions in this narrative.

Please focus on these three processes:
- `kubelet` (PID 240)
- `containerd` (PID 177)
- `containerd-shim-runc-v2` (PID 322)

{{< step >}}Look at the components in the data plane running as pods hosted on a worker node.{{< /step >}}

```bash
kubectl get pods -n kube-system | grep -v kind-control-plane
```

{{< output >}}
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-nzpkq                     1/1     Running   0          6d1h
coredns-558bd4d5db-pm4kq                     1/1     Running   0          6d1h
kindnet-6qjnv                                1/1     Running   0          6d1h
kindnet-h7fdw                                1/1     Running   0          6d1h
kindnet-qmqdx                                1/1     Running   0          6d1h
kindnet-wj4gn                                1/1     Running   0          6d1h
kube-proxy-7d8jx                             1/1     Running   0          6d1h
kube-proxy-jj88g                             1/1     Running   0          6d1h
kube-proxy-q9fzz                             1/1     Running   0          6d1h
kube-proxy-vf65b                             1/1     Running   0          6d1h
{{< /output >}}

Note that we did not filter this list of pods to show only those running on the one worker node `kind-worker`. Therefore, there are duplicate pods running on various nodes. Don't worry about the details of that at the moment. Let's simply pick out three key names here.
- `kube-proxy`
- `kindnet`
- `coredns`

Now, putting together the components running as processes on the node and these selected components running as pods, we have this picture, featuring *just the data plane*.

{{< mermaid >}}
graph TB
client0([Internet client]) 
subgraph cluster[Kubernetes cluster]
  subgraph data[Data Plane]
    kubelet[K8s Node Agent<br>kubelet]
    containerd[Container Runtime Engine<br>containerd]
    podrunner2[Pod Runner<br>shim-runc]
    podrunner3[Pod Runner<br>shim-runc]
    podrunner4[shim]
    podrunner5[shim]
    podrunner6[shim]
    kubeproxy[Network Proxy<br>kube-proxy]
    kindnet[Network Agent<br>kindnet]
    coredns[DNS Service<br>coredns]
    pod2[Pod B]
    pod3[Pod C]
  end
end 
client0 --> pod2
pod3 -->|internal client| pod2
kubelet <--> containerd
kubelet <--> kubeproxy
containerd -.-> podrunner2
containerd -.-> podrunner3
containerd -.-> podrunner4
containerd -.-> podrunner5
containerd -.-> podrunner6
podrunner2 -.-> pod2
podrunner3 -.-> pod3
podrunner4 -.-> kubeproxy
podrunner5 -.-> kindnet
podrunner6 -.-> coredns
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class kubelet,containerd,podrunner2,podrunner3,podrunner4,podrunner5,podrunner6 cyan;
class kubeproxy,kindnet,coredns lavender;
class data yellow;
class client0,pod3 green;
class pod2 orange;
{{< /mermaid >}}

The components in this diagram were composed from a subset of the components you revealed with:
- `docker exec kind-worker ps -ef` -- You used this command to show processes running on a worker node.
  - `kubelet` -- This is the Kubernetes Node Agent that accepts requests from the API Server in the control plane. In this way, the `kubelet` is the broker between what happens on each node and the orchestration of the whole cluster.
  - `containerd` -- This is the Container Runtime Engine that takes instructions from the `kubelet` and launches pods of containers. `containerd` is one example of a [container runtime](https://landscape.cncf.io/card-mode?category=container-runtime&grouping=category) that implements the Container Runtime Interface (CRI) so the `kubelet` can have a standardized interface which allows you choose. Other container runtime alternatives listed in the [CNCF Landscape](https://landscape.cncf.io/card-mode?category=container-runtime&grouping=category) are cri-o, Firecracker, rkt (rocket), and runc. These are all alternatives to using the Docker Daemon (dockerd) as a container runtime for Kubernetes. When Kubernetes transitioned to the flexible CRI, a "docker shim" was implemented to allow use of dockerd instead of containerd or the others.
  - `containerd-shim-runc-v2` -- The [runtime version 2 of containerd](https://github.com/containerd/containerd/blob/main/runtime/v2/README.md) provides an API for shims such as `containerd` to `runc`. [`runc`](https://github.com/opencontainers/runc) creates the proper Linux namespaces and control groups and the processes which implement the containers. Then `runc` terminates. This *shim* is a little adapter which persists to provide a watchdog to receive output of container processes and report error codes back up to the container runtime when containers exit. Note that in the node's PID namespace, the shim is the parent process of the containers (pause, etc.) running in a pod.
- `kubectl get pods -n kube-system` -- You used this command to show pods which implement essential services for networking with and within your cluster.
  - `kube-proxy` -- When the `kubelet` receives changes to Kubernetes *services* and the pods that are members of the *endpoints* behind those services, the `kubelet` asks the `kube-proxy` to update the node's operating system packet filters. Once these filters are updated, the actual data path for communications between your pod containers and their clients should be handled by the operating system network stack, and *not* go through this kube-proxy.
  - `kindnet` -- for Kubernetes clusters based on `kind`, this `kindnet` component provides assignment and mapping of IP addresses to each pod. In other Kubernetes environments, there will often be a [Container Network Interface (CNI)](https://github.com/containernetworking/cni) or other [cloud native network](https://landscape.cncf.io/card-mode?category=cloud-native-network&grouping=category) infrastructure. For example, in Amazon EKS, the `aws-node` pod in each node implements the [Amazon VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s).
  - `coredns` -- this provides the domain name system (DNS) [services for the cluster](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/). Pods and services each have DNS fully qualified domain names (FQDNs) assigned. Other components in the cluster can query CoreDNS to resolve these names into IP addresses.

Note that the dashed links between `containerd`, the shims, and the containers in the pods indicate container runtime management hierarchy, *not* process tree. Some example parent process IDs are shown in the process listing in the Data Plane Components section.

The solid lines (arrows) indicate data flow and management messaging.

## Big Picture

Certainly, there are many more details in both the control plane and data plane. For example, in the control plane, the Controller Manager manages many **controllers** which work with the API Server and Scheduler. High availability is not depicted in these simple figures of the Kubernetes architecture. 

With many moving parts, it is important to get a big picture of a Kubernetes cluster.

{{< mermaid >}}
graph TB
kubectl([kubectl])
client0([Internet client]) 
subgraph cluster[Kubernetes cluster]
  subgraph control[Control Plane]
    apiserver[K8s API Server<br>apiserver]
    etcd[(Cluster Config DB<br>etcd)]
    ctlmgr[Controller Manager<br>controller-manager]
    sched{Scheduler<br>scheduler}
  end
  subgraph data[Data Plane]
    kubelet[K8s Node Agent<br>kubelet]
    containerd[Container Runtime Engine<br>containerd]
    podrunner2[Pod Runner<br>shim-runc]
    podrunner3[Pod Runner<br>shim-runc]
    podrunner4[shim]
    podrunner5[shim]
    podrunner6[shim]
    kubeproxy[Network Proxy<br>kube-proxy]
    kindnet[Network Agent<br>kindnet]
    coredns[DNS Service<br>coredns]
    pod2[Pod B]
    pod3[Pod C]
  end
end 
kubectl --> apiserver
client0 --> pod2
pod3 -->|internal client| pod2
apiserver <--> etcd
apiserver <--> ctlmgr
apiserver <--> sched
apiserver <--> kubelet
kubelet <--> containerd
kubelet <--> kubeproxy
containerd -.-> podrunner2
containerd -.-> podrunner3
containerd -.-> podrunner4
containerd -.-> podrunner5
containerd -.-> podrunner6
podrunner2 -.-> pod2
podrunner3 -.-> pod3
podrunner4 -.-> kubeproxy
podrunner5 -.-> kindnet
podrunner6 -.-> coredns
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class kubectl,apiserver,etcd,ctlmgr,sched blue2;
class kubelet,containerd,podrunner2,podrunner3,podrunner4,podrunner5,podrunner6 cyan;
class kubeproxy,kindnet,coredns lavender;
class control,data yellow;
class client0,pod3 green;
class pod2 orange;
{{< /mermaid >}}

This figure shows only a single node. The API Server communicates with the `kubelet` on each node. Each node has most of the components described in this section and depicted in the diagram, with the exception of `coredns` which typically has only two replicas. Note that the pods in the data plane such as `coredns`, `kindnet`, and `kube-proxy` are implemented as `Deployment` and `DaemonSet` workloads.


## Success

In this lesson, you have:
- Identified Kubernetes control plane components such as `apiserver`, `etcd`, `controller-manager`, and `scheduler`.
- Investigated Kubernetes data plane components running as processes on worker nodes, including `kubelet`, the container runtime `containerd`, and possibly "shims" which help the container runtime orchestrate containers.
- Explored Kubernetes data plane components running as pods in worker nodes, including `kube-proxy`, `kindnet`, and `coredns`.
