---
title: "Loopback Networking"
chapter: false
weight: 722
draft: false
---

## Purpose

Container networking, and therefore Kubernetes networking, can be confusing or daunting. It can be helpful to review some fundamentals of networking to better appreciate many ways in which containerized network is the same as uncontainerized communications. Then we can better discuss the differences.

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
- investigate further detail of that **loopback** scenario,

In subsequent lessons, you could:
- [explore the nature of **pipes**]({{< ref "appendix/724_pipe" >}}) for communicating between processes, and
- [delve into how virtual ethernet interfaces (**veths**)]({{< ref "appendix/730_veth_pairs" >}}) are used by the containers running in pods.

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

## Success

In this lesson, you:
- Investigated a simple **loopback** scenario.

