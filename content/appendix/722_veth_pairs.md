---
title: "VETH Pairs"
chapter: false
weight: 722
draft: false
---

## Purpose

Container networking, and therefore Kubernetes networking, can be confusing or daunting. 
One common point of confusion centers on the nature of network namespaces and virtual ethernet interfaces.
The purpose of this lesson is to build a foundational understanding of how these relate to containers, pods, and nodes in Kubernetes environments.

In the lesson to [Set Up a Service]({{< ref "023_set_up_a_service" >}}), we had included a simple example of a pod talking to itself using a loopback interface. However, as the focus then was `Service` objects, we did not dwell on the loopback scenario. `kubectl exec -it jumpbox -- curl http://localhost:80` was merely used to test the jumpbox pod.

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph k8s-cluster[Kubernetes cluster]
  jumpbox[jumpbox<br>nginx<br>Pod]
end
cloud9 -->|kubectl exec| jumpbox
jumpbox -->|curl| jumpbox
{{< /mermaid >}}

In this lesson, you will:
1. investigate further detail of that **loopback** scenario,
2. explore the nature of **pipes** for communicating between processes, and
3. delve into how virtual ethernet interfaces (**veths**) are used by the containers running in pods.

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

## Talk Within Your Pod - Loopback Interface

You can use loopback interfaces to communicate between containers running in the same pod. Before we get into that scenario, let's review the basics of loopback interfaces. Loopback interfaces can also be used to communicate between processes in the same container.

Linux, macOS, Windows, and other systems have supported software loopback network interfaces for many decades. These loopback interfaces do not depend on Ethernet, WiFi (802.11), or other device drivers. Loopback interfaces may be designated in the operating system as `lo`, `lo0`, `Loopback Psuedo-Interface 1`, or other names.

{{< step >}}Get a list of network interfaces in your Cloud9 instance.{{< /step >}}

```bash
ip link | awk '/^[0-9]+:/ {sub(/:/, "", $2);print$2}'
```

Your output may be similar to this:

{{< output >}}
lo
eth0
br-d55bfafde6bf
docker0
veth6dce049@if13
veth3208e0c@if15
vetha5ee672@if17
veth3d8414c@if19
{{< /output >}}

{{% notice note %}}
On macOS and other UNIX systems, you can use `ifconfig` to show a list of network interfaces and `ifconfig lo0` to show just the configuration information for the loopback interface `lo0`. On Windows, although `ipconfig` does not show the loopback interface, the classic `route print` command does. Using PowerShell, `Get-NetIPInterface -InterfaceIndex 1` will show the loopback interface (both IPv4 and IPv6 bindings) and `Get-NetIPAddress -InterfaceIndex 1` will show the associated IP addresses bound to that.
{{% /notice %}}

{{< step >}}Reach into one of your Kubernetes nodes and ask it to show its loopback interface `lo`.{{< /step >}}

```bash
docker exec kind-worker ip link show lo
```

{{< output >}}
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
{{< /output >}}

{{< step >}}Get the IP address information for that `lo` interface.{{< /step >}}

```bash
docker exec kind-worker ip address show lo
```

{{< output >}}
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
{{< /output >}}

{{< step >}}Look up the entry for localhost in the worker node's `hosts` database.{{< /step >}}

```bash
docker exec kind-worker getent hosts localhost
```

{{< output >}}
::1             localhost ip6-localhost ip6-loopback
{{< /output >}}

{{< step >}}Look at the whole `hosts` data file on `kind-worker`.{{< /step >}}

```bash
docker exec kind-worker cat /etc/hosts
```

{{< output >}}
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.18.0.3      kind-worker
fc00:f853:ccd:e793::3   kind-worker
{{< /output >}}

Because of these mappings, you can use the name `localhost` when referring to the IPv4 address `127.0.0.1` or the IPv6 address `::1`. 

```bash
kubectl exec -it jumpbox -- curl http://localhost:80 # test the NGINX welcome page
```

{{< output >}}
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...yada yada style yada yada...
<body>
<h1>Welcome to nginx!</h1>
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
{{< /output >}}

{{% notice note %}}
The `localhost` name or addresses `127.0.0.1` or `::1` can be used for a container to talk with itself.
For TCP or UDP over IP, it is the port numbers which distinguish sender and listener processes, thus target one process from another within the container. 
{{% /notice %}}

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph k8s-cluster[Kubernetes cluster]
  subgraph jumpbox[jumpbox Pod]
    jumpbox-c0[nginx<br>container]
    lo((loopback<br>IPv4: 127.0.0.0/8<br>IPv6: ::1/128))
  end
end
cloud9 -->|kubectl exec| jumpbox-c0
jumpbox-c0 -->|curl| lo
lo -->|response| jumpbox-c0
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class jumpbox yellow;
class lo cyan;
class jumpbox-c0 lavender;
{{< /mermaid >}}

Zooming in a little closer to see the processes are port numbers involved:

{{< mermaid >}}
graph TB
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph k8s-cluster[Kubernetes cluster]
  subgraph jumpbox[jumpbox Pod]
    subgraph jumpbox-c0[nginx container]
      curl((PID 6042<br>curl<br>port 49152))
      nginx((PID 157<br>nginx<br>port 80))
    end
    lo((loopback<br>IPv4: 127.0.0.0/8<br>IPv6: ::1/128))
  end
end
cloud9 -->|0. kubectl exec| curl
curl -->|1. curl localhost:80| lo
lo -->|2. HTTP GET to localhost:80| nginx
nginx -->|3. HTTP 200 OK to localhost:49152| lo
lo -->|4. HTTP 200 OK to localhost:49152| curl
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class jumpbox yellow;
class lo cyan;
class jumpbox-c0 lavender;
class curl,nginx blue2;
{{< /mermaid >}}

{{% notice note %}}
Because the UNIX Time-Sharing (`uts`) namespace and Network (`net`) namespace is *shared* between the containers in a pod, we can go one step further. Inter-container communications within the pod can also run over the loopback interface.
{{% /notice %}}

## Talk to Another Process - Through a Pipe

Another mechanism you can use to communicate between processes is a **pipe**.

{{< step >}}Pipe the output from `man` to be used as the input to `grep`.{{< /step >}}

```bash
man 2 pipe | grep EXAMPLE -A 58
```

{{% notice note %}}
The details of the `pipe` manual page is targeted to *developers*. As this workshop is *not* written for that audience, we have heavily editorialized the output. Feel free to skip ahead to the diagram and narrative after this example output.
{{% /notice %}}

{{< output >}}
EXAMPLE
... description of fork(2), pipe(2), and reader/writer roles ...
... developer "magic" ...
    char* mess = argv[1];    // example in man page just uses argv[1] below
    int pipefd[2];
    pid_t cpid;
    char buf;

    pipe(pipefd);            // request TWO pipe endpoint "file descriptors"
    cpid = fork();           // spawn a child process
    if (cpid == 0) {         // Child reads from pipe
        close(pipefd[1]);    // Close unused write end
        while (read(pipefd[0], &buf, 1) > 0) // this is essentially cat(1)
            write(STDOUT_FILENO, &buf, 1);
        write(STDOUT_FILENO, "\n", 1);
        close(pipefd[0]);
    } else {                 // Parent writes message to pipe
        close(pipefd[0]);    // Close unused read end
        write(pipefd[1], mess, strlen(mess));
        close(pipefd[1]);    // Reader will see EOF
        wait(NULL);          // Wait for child
    }
{{< /output >}}

What does this do? In this case, your shell launches the **parent** process to run this program.
Within this program, that process creates a **pair** of pipe endpoints. 
Then it uses the `fork` system call spawns a child process. 
At the `if`/`else` portion, execution diverges. 
The ***child*** process runs the `if` clause.
The ***parent*** process runs the `else` clause.
In this example, the parent process writes messages to one end of the pipe: the "writer" endpoint.
Similar to a loopback interface, the pipe relays data that is written in one end so it comes out the other.
In this example, the child process reads the message out the other end of the pipe: the "reader" endpoint.

{{< mermaid >}}
graph LR
subgraph cloud9[Cloud 9 dev instance in VPC]
  shell((PID 1582<br>bash))
  parent((PID 1701<br>parent))
  child((PID 1702<br>child))
  pipe1((pipe<br>relays<br>data))
end
shell -.-> parent
parent -.-> child
parent -->|writes| pipe1
pipe1 -->|reads| child
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class shell,parent,child blue2;
class pipe1 cyan;
{{< /mermaid >}}

One of the main jobs of a shell is to do redirection and pipe-fitting. The developer-level details of how shells such as `sh`, `bash`, and `zsh` implement pipes such as discussion of `exec` is beyond the scope of this lesson. However, the conceptual perspective can aid in understanding network communications interconnections. The concepts illustrated with the `parent`/`child` example based on `fork` and `pipe` are sufficient to build on. 

In the actual command line you ran in the previous step: `man 2 pipe | grep EXAMPLE -A 58`, what is the relationship between the shell (assume `bash`), and the processes running `man`, and `grep`? 

{{< mermaid >}}
graph LR
subgraph cloud9[Cloud 9 dev instance in VPC]
  shell((PID 1582<br>bash))
  man((PID 1657<br>man))
  grep((PID 1658<br>grep))
  pipe1((pipe<br>relays<br>data))
end
shell -.-> man
shell -.-> grep
man -->|writes| pipe1
pipe1 -->|reads| grep
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class shell,man,grep blue2;
class pipe1 cyan;
{{< /mermaid >}}

As shown in the diagram, the shell--`bash`--is the parent of both of the other commands. With longer command pipelines composed of three, four, or more commands, the shell is the parent of all the commands in the pipeline. For some number, N, commands, the shell will create one fewer--i.e. N-1--pipes and plumb the pipes together between the sequence of commands.

In the above example, with two commands, `man` and `grep`, N=2. N-1=1, therefore the shell simply creates one pipe as shown in the figure. This is indicated with the dashed lines. Data flow goes from the first command in the pipeline, `man`, into the pipe. The pipe relays the data to the second command, `grep`. This kind of communications has been common in most operating systems for the past several decades. 

The processes within each of your containers can communicate with one another using pipes. You can use other mechanisms for communicating between processes in different containers. Beyond containers, inter-pod, inter-node, and inter-cluster communications would also be based on non-pipe techniques. Nevertheless, similarities between pipes and those other techniques should help your confidence with any communications mechanisms.

A major distinction between the `loopback` and `pipe` mechanisms is that a pipe is unidirectional and dedicated to just two processes. One process writes to the pipe and the other process reads. As network interfaces, loopback interfaces can have many processes communicating back and forth across one loopback interface. Those processes rely on distinct tuples of IP addresses, protocol numbers, and port numbers to uniquely identify their partner endpoints.

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

veth

https://man7.org/linux/man-pages/man4/veth.4.html


Virtual Ethernet Pair
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


## Talking Within and Between Pods - Via Pod Addresses

Can use pod IP for communication.

```bash
demopod=$(kubectl get pods -l app=demo -n dev -o jsonpath={.items[0].metadata.name})                     
echo Going to use $demopod
```

{{< output >}}
Going to use demo-58465f467c-24bcq
{{< /output >}}

```bash
kubectl exec $demopod -n dev  -- grep -v ip6 /etc/hosts                                                  
```

{{< output >}}
# Kubernetes-managed hosts file.
127.0.0.1       localhost
10.244.2.7      demo-58465f467c-24bcq
{{< /output >}}




## Success

In this lesson, you:
- Investigated a simple **loopback** scenario.
- Explored the nature of **pipes** for communicating between processes.
- Delved into how virtual ethernet interfaces (**veths**) are used by the containers running in pods.

