---
title: "VETH Encore"
chapter: false
weight: 732
draft: false
---

## Purpose

Container networking, and therefore Kubernetes networking, can be confusing or daunting. One common point of confusion centers on the nature of network namespaces and virtual ethernet interfaces. The purpose of this lesson is to build a foundational understanding of how these relate to containers, pods, and nodes in Kubernetes environments. 

In this lesson, you will:
- delve into how virtual ethernet interfaces (**veths**) are used by the containers running in pods.

{{% notice note %}}
Pods using **veth** network interfaces is the foundation for facilitating communications pod-to-pod, pod-to-node, pod-to-control plane, pod-to-cloud, and pod-to-outside.
{{% /notice %}}

{{< mermaid >}}
graph LR
subgraph orange-ns[Orange Network Namespace]
  curl((PID 6042<br>curl<br>port 49152))
  tap1((Tap 1))
end
subgraph green-ns[Green Network Namespace]
  nginx((PID 157<br>nginx<br>port 80))
  tap2((Tap 2))
end
curl -->tap1
tap1 ===|VETH Pair| tap2
tap2 -->nginx
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class orange-ns orange;
class green-ns green;
class tap1,tap2 cyan;
class curl,nginx blue2;
{{< /mermaid >}}

## Look at Existing `veth` Pairs

A virtual ethernet interfaces, or `veth`, is a software network interface like a loopback interface.
Like a pipe, a `veth` is created with two endpoints. Each endpoint is called a `veth` and the combination you create is called a `veth` ***pair***. Unlike the *unidirectional* nature of a pipe but very much like a loopback interface, each `veth` is *bidirectional*. Therefore, the `veth` *pair* is bidirectional too. The process using one end of the `veth` pair can both write and read, while the process on the other end of the `veth` pair can read and write correspondingly.

In some container hosting scenarios, an `veth` pair is created for each individual container to communicate with and through its host. In Kubernetes, a `veth` pair is created for each *pod*. Your pod communicates with its node through the `veth` pair. Your pod communicates *through* the node using the `veth` pair. This is the primary mechanism of Kubernetes pod networking. 

Here, Linux namespaces come into play. Kubernetes creates PID namespaces per container. Kubernetes creates UTS namespaces per pod. Therefore, Kubernetes creates a **network** namespace per pod. Recall that a `pipe` between two processes is created just before those processes are created with `fork`. Kubernetes creates a `veth` pair between a pod and its node just before the node uses `clone`/`unshare` to create the pod. As with pipes, the `veth` pair plumbing is adjusted before the containers in the pod are started.

{{< step >}}Look at any network namespaces--`netns`--using the `ip` command in one of your worker nodes.{{< /step >}}

```bash
docker exec kind-worker ip netns listcni-f4fd112d-d27d-954c-253c-858ed398084f (id: 5)
```

{{< output >}}
cni-fee38a0c-3503-10d5-39c7-a5bf173e0caf (id: 4)
cni-c5b8b5eb-b4e7-1254-9712-68d51dd52926 (id: 1)
{{< /output >}}

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
    inet 10.244.3.1/32 scope global vethf8061033
    inet 10.244.3.1/32 scope global vethd0a8334f
    inet 10.244.3.1/32 scope global veth8a50c639
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
4: vethf8061033@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether e2:4b:5d:06:4c:47 brd ff:ff:ff:ff:ff:ff link-netns cni-3fc9f5de-1a16-a7b3-b868-f593505b4477
    inet 10.244.3.1/32 scope global vethf8061033
5: vethd0a8334f@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 0a:ae:57:83:1e:16 brd ff:ff:ff:ff:ff:ff link-netns cni-0faa7af3-d826-d30b-76f6-2cc8d39da0eb
    inet 10.244.3.1/32 scope global vethd0a8334f
6: veth8a50c639@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 4e:8e:5d:9d:1a:9d brd ff:ff:ff:ff:ff:ff link-netns cni-fee38a0c-3503-10d5-39c7-a5bf173e0caf
    inet 10.244.3.1/32 scope global veth8a50c639
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
{{< /output >}}


## Building Veth Pairs

You can build veth pairs like the techniques used in the [Linux Namespaces section]({{< ref "014_linux_namespaces" >}}). This exercise is intended to help you understand the mechanics and nature of `veth` interfaces, and `veth` pairs. Kubernetes manages `veth` pairs for you via the `kubelet` working with the container runtime (e.g. `containerd`) and container network interface (CNI) such as `kindnet`, `flannel`, or `aws-node`.

{{< step >}}Obtain some basic information about the `ip link` command.{{< /step >}}

```bash
man ip-link | grep -B 3 -A 3 veth
```

{{< output >}}
    ip link help [ TYPE ]

    TYPE := [ bridge | bond | can | dummy | hsr | ifb | ipoib |
            macvlan | macvtap | vcan | vxcan | veth | vlan |
            vxlan | ip6tnl | ipip | sit | gre | gretap | erspan
            | ip6gre | ip6gretap | ip6erspan | vti | nlmon |
            ipvlan | lowpan | geneve | vrf | macsec ]
--
        ...
        veth - Virtual ethernet interface
        ...
    and this "4" priority can be used in the
    egress qos mapping to set VLAN prio "5":

        ip link set veth0.10 type vlan egress 4:5
--
    For a link of types VETH/VXCAN the following additional arguments are supported:

        ip link add DEVICE type { veth | vxcan } [ peer name NAME ]
{{< /output >}}

Note in these excerpts that:
- a `veth` is a virtual ethernet interface
- you can use `ip link add` to add a `veth` *pair* - **DEVICE** and **PEER**  ends of the pair
- you can configure quality of service and priority preferences on a `veth`

The [veth man page](https://man7.org/linux/man-pages/man4/veth.4.html) presents some useful concepts and details.

Here are some essential concepts:
- A ***veth pair*** can be thought of as a pipe or tunnel that connects two endpoints. 
- Each endpoint is a single `veth`--a virtual ethernet interface. 
- Once created, the two `veth` interfaces of the pair can be moved to different network namespaces.

{{< mermaid >}}
graph LR
subgraph orange-ns[Orange Network Namespace]
  curl((PID 6042<br>curl<br>port 49152))
  tap1((Tap 1))
end
subgraph green-ns[Green Network Namespace]
  nginx((PID 157<br>nginx<br>port 80))
  tap2((Tap 2))
end
curl -->tap1
tap1 ===|VETH Pair| tap2
tap2 -->nginx
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class orange-ns orange;
class green-ns green;
class tap1,tap2 cyan;
class curl,nginx blue2;
{{< /mermaid >}}

[Introduction to Container Networking - September 10, 2019 | By: Rancher Admin](https://www.suse.com/c/rancher_blog/introduction-to-container-networking/)



My notes from earlier on that:
vt = 26:10 network namespace

Net namespace: in theory
- Processes within a given network namespace gettheir own private network stack, including:
  - network interfaces (including lo)
  - routing tables
  - iptables rules
  - sockets (ss, netstat)
- YOu can move a network interface across netns
  - ip link set dev eth0 netns PID

Net namespace: in practice
- Typical use-case:
  - use veth pairs (two virtual interfaces actsing as a cross-over cable)
  - eth0 in container network namespace
  - paired with vethXXX in host network namespace
  - all the vethXXX are bridged together
  - (Docker calls the bridge docker0)
- But also: the magic of --net container
  - shared localhost (and more!)

-- end notes from earlier
vt = 48:30
### outside
pidof unshare
CPID=6902
ip link add name h6902 type veth peer name c6902
ip link set c$CPID netns $CPID
ip link set h$CPID master docker0 up
### inside
ip link set lo up
ip link set c6902 name eth0 up
ip addr addr 172.17.42.3/16 dev eth0
ip route add default via 172.17.42.1
ping 4.2.2.1

alpine has no bash, only sh

-- repeating notes from yesterday


--


## Success

In this lesson, you:
- Delved into how virtual ethernet interfaces (**veths**) are used by the containers running in pods.

