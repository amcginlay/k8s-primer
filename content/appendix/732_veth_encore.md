---
title: "VETH Encore"
chapter: false
weight: 732
draft: false
---

## Purpose

The purpose of this lesson is to build on the foundational understanding virtual ethernet interfaces and network namespaces covered in the [VETH Pairs lesson]({{< ref "appendix/730_veth_pairs" >}}). In particular this lesson focuses on how `veth` and `netns` relate to containers, pods, and nodes in Kubernetes environments. 

In this lesson, you will:
- delve into how virtual ethernet interfaces (**veths**) are used by the containers running in pods.

{{% notice note %}}
Pods using **veth** network interfaces is the foundation for facilitating communications pod-to-pod, pod-to-node, pod-to-control plane, pod-to-cloud, and pod-to-outside.
{{% /notice %}}

## Look at Existing `veth` Interfaces on a Node

In the previous lesson, you created network namespaces and a `veth` pair and plumbed them together to practice with how those building blocks operate and relate to one another. In that lesson you worked in your Cloud9 instance. Now the focus is working with `veth` and `netns` resources on your Kubernetes worker nodes.

{{< step >}}Look at any network namespaces--`netns`--using the `ip` command in one of your Kubernetes worker nodes.{{< /step >}}

```bash
docker exec kind-worker ip netns list
```

{{< output >}}
cni-f4fd112d-d27d-954c-253c-858ed398084f (id: 5)
cni-fee38a0c-3503-10d5-39c7-a5bf173e0caf (id: 4)
cni-c5b8b5eb-b4e7-1254-9712-68d51dd52926 (id: 1)
{{< /output >}}

{{% notice note %}}
The `list` verb is optional with `ip netns`.
{{% /notice %}}

What do the letters "cni" stand for at the beginning of those network namespace names?

The Container Network Interface (CNI) is a pluggable interface that allows you to include a variety of underlying network infrastructures beneath Kubernetes. [Visit the Cloud Native Network category](https://landscape.cncf.io/card-mode?category=cloud-native-network&grouping=category) in the CNCF Landscape. The CNCF is the [Cloud Native Computing Foundation](https://www.cncf.io). Quoting their website: "Cloud Native Computing Foundation (CNCF) serves as the vendor-neutral home for many of the fastest-growing open source projects, including Kubernetes, Prometheus, and Envoy."

{{< step >}}Show the list of network interfaces on that node.{{< /step >}}

```bash
docker exec kind-worker ip link | awk '/^[0-9]+:/ {sub(/:/, "", $2);print$2}' 
```

{{< output >}}
lo
veth09e8e743@if3
veth8a50c639@if3
vethca880bcd@if3
eth0@if16
{{< /output >}}

{{< step >}}Display the IP address (or address range) associated with each of these interfaces.{{< /step >}}

```bash
docker exec kind-worker ip address | grep "inet "
```

{{< output >}}
    inet 127.0.0.1/8 scope host lo
    inet 10.244.3.1/32 scope global veth09e8e743
    inet 10.244.3.1/32 scope global veth8a50c639
    inet 10.244.3.1/32 scope global vethca880bcd
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
{{< /output >}}


{{< step >}}Show a greater degree of detail, including the up/down state, and MAC address for each.{{< /step >}}

```bash
docker exec kind-worker ip address | grep -v inet6 | grep -v forever
```

{{< output >}}
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: veth09e8e743@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether fa:ab:66:67:11:24 brd ff:ff:ff:ff:ff:ff link-netns cni-c5b8b5eb-b4e7-1254-9712-68d51dd52926
    inet 10.244.3.1/32 scope global veth09e8e743
6: veth8a50c639@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 4e:8e:5d:9d:1a:9d brd ff:ff:ff:ff:ff:ff link-netns cni-fee38a0c-3503-10d5-39c7-a5bf173e0caf
    inet 10.244.3.1/32 scope global veth8a50c639
7: vethca880bcd@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 12:4d:cb:11:ad:4f brd ff:ff:ff:ff:ff:ff link-netns cni-f4fd112d-d27d-954c-253c-858ed398084f
    inet 10.244.3.1/32 scope global vethca880bcd
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
{{< /output >}}

Each `veth` on the Kubernetes worker node represents one half of a `veth` pair.
Where are the matching `veth` interfaces for each `veth` on the node?

## Check Your Pod IP Addresses

{{< step >}}Check the IP address of each pod in your default namespace.{{< /step >}}

```bash
kubectl get pod -o wide
```

{{< output >}}
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
jumpbox   1/1     Running   0          9d    10.244.1.6   kind-worker2   <none>           <none>
{{< /output >}}

Your scenario will vary in terms of IP address assigned to the pod, and on which node the pod was scheduled.
In the example depicted above, the pod IP address was `10.244.1.6` and the worker node was `kind-worker2`.

{{< step >}}Get the list of pods running on your first worker node `kind-worker`.{{< /step >}}

```bash
kubectl get pod -A -o wide | awk '$8 == "kind-worker"'
```

{{< output >}}
dev-system           demo-daemon-vq4t7                      1/1 Running   0    10d    10.244.3.4  kind-worker 
dev                  demo-58465f467c-w4n6j                  1/1 Running   0    3d9h   10.244.3.9  kind-worker 
kube-system          kindnet-h7fdw                          1/1 Running   0    10d    172.18.0.3  kind-worker 
kube-system          kube-proxy-q9fzz                       1/1 Running   0    10d    172.18.0.3  kind-worker 
kubernetes-dashboard kubernetes-dashboard-576cb95f94-kvz69  1/1 Running   0    5d7h   10.244.3.8  kind-worker 
{{< /output >}}

{{< step >}}Look in the list of pods, note each IP address, and find the corresponding network interface on the node from the `ip address` query you did earlier.{{< /step >}}

{{< step >}}Is every pod using a `veth`? For those that are, look on the properties of the `veth` in the `ip address` output. What is the value after the word **link-netns**? Make a note of those values.{{< /step >}}

{{< step >}}For each **link-netns** value, is there a corresponding `netns` value in your `ip netns list` output at the beginning of this lesson?{{< /step >}}

The relationship between the network namespaces, veths, and pods in each Kubernetes worker node can be depicted as follows:

{{< mermaid >}}
graph TB
subgraph node-ns[Node Network Namespace]
  tap1N((veth))
  tap2N((veth))
  tap3N((veth))
  lo((lo))
  eth0((eth0))
  bridge{{bridge}}
end
subgraph podA-ns[netns cni-f4fd112d-...]
  tap1A((veth))
  podA([demo-daemon-vq4t7<br>10.244.3.4])
end
subgraph podB-ns[netns cni-fee38a0c-...]
  tap2B((veth))
  podB([demo-58465f467c-w4n6j<br>10.244.3.9])
end
subgraph podC-ns[netns cni-c5b8b5eb-...]
  tap3C((veth))
  podC([kubernetes-dashboard-576cb95f94-kvz69<br>10.244.3.8])
end
tap1N --> bridge
tap2N --> bridge
tap3N --> bridge
tap1N ===|VETH Pair| tap1A
tap2N ===|VETH Pair| tap2B
tap3N ===|VETH Pair| tap3C
tap1A --> podA
tap2B --> podB
tap3C --> podC
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class node-ns orange;
class podA-ns green;
class podB-ns blue2;
class podC-ns lavender;
class lo,eth0,tap1N,tap2N,tap3N,tap1A,tap2B,tap3C cyan;
class bridge,podA,podB,podC yellow;
{{< /mermaid >}}

## Success

In this lesson, you:
- Delved into how virtual ethernet interfaces (**veths**) are used by the containers running in pods.

