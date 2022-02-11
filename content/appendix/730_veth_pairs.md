---
title: "VETH Pairs"
chapter: false
weight: 730
draft: false
---

## Purpose

Container networking, and therefore Kubernetes networking, can be confusing or daunting. One common point of confusion centers on the nature of network namespaces and virtual ethernet interfaces. The purpose of this lesson is to build a foundational understanding of how those two technologies work together. 

In the *next* lesson, we'll build on this and focus on how these relate to containers, pods, and nodes in Kubernetes environments. 

In *this* lesson, you will:
- look at the basics of virtual ethernet interfaces (**veths**) and network namespaces (**netns**).


## A tale of two **veths** -- a `veth` ***pair***

A virtual ethernet interface, or `veth`, is a software network interface [like a loopback interface]({{< ref "appendix/722_loopback" >}}).
[Like a pipe]({{< ref "appendix/724_pipe" >}}), a `veth` is created with two endpoints. Each endpoint is called a `veth` and the combination you create is called a `veth` ***pair***. Unlike the *unidirectional* nature of a pipe but very much like a loopback interface, each `veth` is *bidirectional*. Therefore, the `veth` *pair* is bidirectional too. The process using one end of the `veth` pair can both write and read, while the process on the other end of the `veth` pair can read and write correspondingly.

However, a `veth` pair has a fundamental difference from typical unidirectional pipes and a simple loopback interface. What you write to one `veth` pops out the other end--to the other `veth`; what is written to the second `veth` pops out the first. Thus, a `veth` pair is essentially two separate unidirectional channels. This diagram illustrates the two `veth` interfaces in a pair as `Tap 1` and `Tap 2`.

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

In some container hosting scenarios, an `veth` pair is created for each individual *container* to communicate with and through its host. In Kubernetes, a `veth` pair is created for each *pod*. Your pod communicates with its node through the `veth` pair. Your pod communicates *through* the node using the `veth` pair. This is the primary mechanism of Kubernetes pod networking. 

Here, Linux namespaces come into play. Kubernetes creates PID namespaces per container. Kubernetes creates UTS namespaces per pod. The UTS hostname and the network should be aligned. Therefore, Kubernetes creates a **network** namespace per pod. Recall that a `pipe` between two processes is created just before those processes are created/separated with `fork`. Kubernetes creates a `veth` pair between a pod and its node just before the node uses `clone`/`unshare` to create the pod. As with pipes, the `veth` pair plumbing is adjusted before the containers in the pod are started.

## Build a Veth Pair

You can build veth pairs like the techniques used in the [Linux Namespaces section]({{< ref "014_linux_namespaces" >}}). This exercise is intended to help you understand the mechanics and nature of `veth` interfaces, and `veth` pairs. Kubernetes manages `veth` pairs for you via the `kubelet` working with the container runtime (e.g. `containerd`) and container network interface (CNI) such as `kindnet`, `flannel`, or `aws-node`.

Let's start simple.

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
- you can use `ip link add` to add a `veth` *pair* - **DEVICE** and **PEER** ends of the pair
- you can configure quality of service and priority preferences on a `veth`

The [veth man page](https://man7.org/linux/man-pages/man4/veth.4.html) presents some useful concepts and details.

Here are some essential concepts:
- A ***veth pair*** can be thought of as a pipe or tunnel that connects two endpoints. 
- Each endpoint is a single `veth`--a virtual ethernet interface. 
- Once created, the two `veth` interfaces of the pair can be moved to different network namespaces.

{{< step >}}Check what network interfaces you have first.{{< /step >}}

```bash
ip link show lo
ip link show eth0
```

{{< output >}}
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 06:3e:65:63:1f:39 brd ff:ff:ff:ff:ff:ff
{{< /output >}}
  
{{% notice tip %}}
You could simply run `ip link` without further parameters to show *all* network interfaces (adapters, data links, etc.) on your system. Knowing that `ip link` lets you manage your loopback and ethernet interfaces like `ifconfig` or `ipconfig` do in other systems the the point. The rest of the interfaces aren't important right now.
{{% /notice %}}

{{< step >}}Create a `veth` pair.{{< /step >}}

```bash
sudo ip link add tap1 type veth peer name tap2
```

No news is good news. Congratulations! If you did not receive an error, you created a `veth` pair.

{{< mermaid >}}
graph TB
subgraph cloud9-ns[Cloud9 Instance Network Namespace]
  bash((PID 5038<br>bash<br>))
  lo((lo))
  eth0((eth0))
  tap1((tap1))
  tap2((tap2))
end
outside((outside))
tap1 ===|VETH Pair| tap2
lo --> lo
eth0 --> outside
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#6495ed,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9-ns orange;
class lo,eth0,tap1,tap2 cyan;
class bash,outside blue2;
{{< /mermaid >}}

{{< step >}}Confirm that your two `veth` interfaces in the pair were created.{{< /step >}}

```bash
for if in lo eth0 tap1 tap2; do ip link show $if; done
```

{{< output >}}
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 06:3e:65:63:1f:39 brd ff:ff:ff:ff:ff:ff
24: tap1@tap2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 86:ac:56:a5:fa:f0 brd ff:ff:ff:ff:ff:ff
23: tap2@tap1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 9e:c4:5e:93:5f:69 brd ff:ff:ff:ff:ff:ff
{{< /output >}}

What is the difference in the ***state*** of the `eth0` Ethernet interface and your two veths?

By default, both `veth` interfaces should be `DOWN`.

{{< step >}}Create two network namespaces.{{< /step >}}

```bash
sudo ip netns add orange                # create a new network namespace named "orange"
sudo ip netns add green                 # create a new network namespace named "green"
```

{{< step >}}Move each `veth` to one of the namespaces; `tap1` in `orange` and `tap2` into `green`.{{< /step >}}

```bash
sudo ip link set tap1 netns orange      # move the tap1 veth into the orange network namespace
sudo ip link set tap2 netns green       # move the tap2 veth into the green network namespace
```

{{< step >}}Check if either of those `veth` interfaces is visible in the default network namespace of your Cloud9 instance.{{< /step >}}

{{< columns >}}
{{% column %}}
```bash
ip link show tap1
ip link show tap2
```
{{% /column %}}

{{< column >}}
{{< output >}}
Device "tap1" does not exist.
Device "tap2" does not exist.
{{< /output >}}
{{< /column >}}
{{< /columns >}}

## Configure the `tap1` veth in the `orange` netns

{{< step >}}Check which network interfaces are in the `orange` namespace.{{< /step >}}

```bash
sudo ip netns exec orange ip address 
```

{{< output >}}
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
24: tap1@if23: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 86:ac:56:a5:fa:f0 brd ff:ff:ff:ff:ff:ff link-netnsid 1
{{< /output >}}

{{< step >}}Bring `tap1` up.{{< /step >}}

```bash
sudo ip netns exec orange ip link set tap1 up
```

{{< step >}}Assign an IPv4 address to `tap1`.{{< /step >}}

```bash
sudo ip netns exec orange ifconfig tap1 192.168.1.2/24
```

{{< step >}}Confirm the revised configuration of `tap1` with that address.{{< /step >}}

```bash
sudo ip netns exec orange ip link show tap1
```

{{< output >}}
24: tap1@if23: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether 86:ac:56:a5:fa:f0 brd ff:ff:ff:ff:ff:ff link-netnsid 1
{{< /output >}}

{{% notice note %}}
What is the **state** of the `tap1` interface now? Why? You had set it to up before you assigned an address. The other end of the `veth` pair is not yet up.
{{% /notice %}}

## Configure the `tap2` veth in the `green` netns

{{< step >}}Check which network interfaces are in the `green` namespace.{{< /step >}}

```bash
sudo ip netns exec green ip address  
```

{{< output >}}
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
23: tap2@if24: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 9e:c4:5e:93:5f:69 brd ff:ff:ff:ff:ff:ff link-netnsid 0
{{< /output >}}


{{< step >}}Bring `tap2` up.{{< /step >}}

```bash
sudo ip netns exec green ip link set tap2 up
```

{{< step >}}Assign an IPv4 address to `tap2`.{{< /step >}}

```bash
sudo ip netns exec green ifconfig tap2 192.168.1.3/24
```

{{< step >}}Confirm the revised configuration of `tap2` with that address.{{< /step >}}

```bash
sudo ip netns exec green ip link show tap2
```

{{< output >}}
23: tap2@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 9e:c4:5e:93:5f:69 brd ff:ff:ff:ff:ff:ff link-netnsid 0
{{< /output >}}

{{% notice note %}}
What is the **state** of the `tap2` interface? What is the lower-layer state? 
{{% /notice %}}

Now both `veth` interfaces are up and configured. You have successfully configured a `veth` pair!

## Test Your `veth` Pair

{{< step >}}Try to connect from the `orange` network namespace to the `green` network namespace.{{< /step >}}

```bash
sudo ip netns exec orange ping -c 5 192.168.1.3
```

{{< output >}}
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=255 time=0.020 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=255 time=0.023 ms
64 bytes from 192.168.1.3: icmp_seq=3 ttl=255 time=0.039 ms
64 bytes from 192.168.1.3: icmp_seq=4 ttl=255 time=0.026 ms
64 bytes from 192.168.1.3: icmp_seq=5 ttl=255 time=0.023 ms

--- 192.168.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4078ms
rtt min/avg/max/mdev = 0.020/0.026/0.039/0.007 ms
{{< /output >}}

{{% notice note %}}
The `-c 5` option on `ping` specifies to just run 5 requests. `c` is short for "count."
{{% /notice %}}

{{< step >}}Test it the other way -- from the `green` netns to the `orange` netns.{{< /step >}}

```bash
sudo ip netns exec green ping -c 5 192.168.1.2
```

{{< output >}}
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=255 time=0.040 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=255 time=0.029 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=255 time=0.032 ms
64 bytes from 192.168.1.2: icmp_seq=4 ttl=255 time=0.025 ms
64 bytes from 192.168.1.2: icmp_seq=5 ttl=255 time=0.023 ms

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4098ms
rtt min/avg/max/mdev = 0.023/0.029/0.040/0.009 ms
{{< /output >}}

This figure represents the `ping` request going over the `veth` pair in the first example. For the second example, it is simply the reverse.

{{< mermaid >}}
graph LR
subgraph orange-ns[Orange Network Namespace]
  ping((PID 9764<br>ping<br>192.168.1.3))
  tap1((tap1<br>192.168.1.2))
end
subgraph green-ns[Green Network Namespace]
  icmp[ICMP<br>echo]
  tap2((tap2<br>192.168.1.3))
end
ping -->tap1
tap1 ===|VETH Pair| tap2
tap2 -->icmp
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
class ping,icmp blue2;
{{< /mermaid >}}

## Optional: Clean Up Your Namespaces

If you do not plan to use this `veth` pair or the two `netns` you created, delete them now. Note: deleting the network namespaces deletes the network interfaces inside.

{{< step >}}Confirm the network namespaces you had created earlier.{{< /step >}}

```bash
ip netns list
```

{{< output >}}
green
orange (id: 4)
{{< /output >}}

{{< step >}}Delete both namespaces.{{< /step >}}

```bash
sudo ip netns delete green           
sudo ip netns delete orange
```

## Success

In this lesson, you:
- Created a `veth` pair.
- Creted two network namespaces (`netns`).
- Configured each `veth` to be in one of those namespaces, brought them up, assigned IP addresses, and tested connectivity.

